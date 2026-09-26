---
title: object와 companion - 싱글턴과 동반 객체
date: 2026-07-13
tags: [kotlin, object, companion, singleton, jvmstatic, 학습노트]
---

[[21 - 클래스와 생성자]]에서는 클래스를 `new` 없이 인스턴스화하는 방법과 `init` 블록·프로퍼티 초기화가 실행되는 순서를 JVM 수준에서 살펴봤다. [[24 - 상속과 오버라이딩과 초기화 순서]]에서는 기본이 `final`인 Kotlin에서 `open`·`override`로 다형성을 허용하는 방법을, [[25 - 인터페이스]]에서는 상태 없는 계약과 기본 구현을 다뤘다. 이 장에서는 그 위에 특수한 형태의 클래스를 하나 더한다.

바로 **인스턴스가 정확히 하나뿐인 클래스**, 즉 싱글턴을 언어가 문법으로 직접 제공하는 `object`와, 그 싱글턴을 클래스에 붙여 "정적 멤버처럼 보이지만 실제로는 인스턴스 멤버"인 자리를 만드는 `companion object`이다.

이 장에서는 `object` 선언을 등가의 클래스로 풀어 써서 object가 정적 클래스가 아니라 실제 인스턴스라는 점을 보이고, 지연·스레드 안전 초기화의 근거와 한계, object의 다형성, `object` 표현식과의 차이를 다룬다. 이어서 `companion object` 멤버가 static이 아니라는 점을 바이트코드로 확인하고, `@JvmStatic`·`@JvmField`·`const`가 각각 무엇을 어느 클래스로 옮기는지와 companion의 실전 패턴을 정리한다.

익명 객체의 캡처 같은 세부는 [[31 - 중첩 클래스와 이너 클래스와 익명 객체]]에서, 확장의 정적 디스패치 원리는 [[35 - 확장 함수와 프로퍼티]]에서 다룬다. `data object`는 [[27 - 데이터 클래스]], [[30 - sealed 클래스와 대수적 데이터 타입]]과 관련이 있으므로 여기서는 일반 `object`와의 차이만 짚고, `const val`이 컴파일 상수인 이유는 [[11 - 변수와 초기화 - val과 const와 lateinit]]에서 설명한다.

---

## 1. object 선언: 싱글턴을 언어 기능으로 제공한다

### 1.1 문제: 직접 작성하는 싱글턴의 함정

프로그램에는 "단 하나만 있어야 의미가 있는" 것들이 있다. 애플리케이션 전역의 설정 레지스트리, 로그를 모으는 수집기, 수학 상수와 순수 함수의 묶음, 빈 컬렉션을 나타내는 유일한 원소가 그렇다. 이런 것은 인스턴스가 여러 개 생기면 오히려 버그가 된다. 전통적으로 인스턴스가 하나뿐임을 보장하려면 싱글턴 패턴을 직접 작성해야 했는데, Java에서는 이 코드가 까다롭기로 유명하다.

```java
// Java: 손으로 짠 싱글턴 — 스레드 안전을 얻으려면 신경 써야 할 것이 많다
public final class Registry {
    private static Registry instance;          // 가시성·재배열 문제의 씨앗
    private Registry() {}                       // 외부 생성 차단
    public static Registry getInstance() {      // 이중 검사 락(DCL)이 필요할 수도
        if (instance == null) {                 // 두 스레드가 동시에 통과하면?
            instance = new Registry();          // 두 개가 생길 수 있다
        }
        return instance;
    }
}
```

이 코드의 진짜 문제는 인스턴스가 하나임을 보장하는 로직을 개발자가 직접 작성해야 한다는 점이다. 동시에 실행되는 환경에서는 두 스레드가 `if (instance == null)`을 동시에 통과할 수 있고, 그러면 싱글턴이 두 개 생긴다. 이를 막으려면 `synchronized`를 쓰거나 이중 검사 락(double-checked locking)에 `volatile`을 더하는 정교한 관용구가 필요하다.

이 관용구는 메모리 모델을 정확히 이해하지 못하면 미묘하게 틀리기 쉽다. 이 배경은 [[43 - 동시성과 메모리 모델]]에서 자세히 다룬다.

Kotlin은 이렇게 반복되는 위험을 언어 기능으로 해결했다. `class` 대신 `object` 키워드로 선언하면, 인스턴스가 정확히 하나라는 보장은 **컴파일러와 런타임이 책임진다**.

```kotlin
object Registry {
    private val entries = mutableMapOf<String, String>()
    fun put(key: String, value: String) { entries[key] = value }
    fun get(key: String): String? = entries[key]
}

fun main() {
    Registry.put("host", "localhost")
    println(Registry.get("host"))   // => localhost
    // val r = Registry()           // 컴파일 에러: object는 생성자를 호출할 수 없다
    // val r = new Registry()       // Kotlin엔 애초에 new가 없다
}
```

`Registry`는 타입 이름이면서 동시에 그 타입의 유일한 인스턴스를 가리키는 값이다. `Registry.put(...)`은 `Registry` 클래스의 정적 메서드를 호출하는 것처럼 보인다. 하지만 이 겉모습이 이 장에서 가장 먼저 바로잡을 오해이다.

### 1.2 object에는 생성자가 없다

`object` 선언에는 주 생성자도 부 생성자도 없다. 인스턴스를 하나만 만들고, 그것도 런타임이 만들기 때문에 누가 어떤 인자로 만들지를 지정할 여지가 없다. 그래서 `object Registry(config: Config)` 같은 선언은 문법 오류이다. 초기 상태가 필요하면 프로퍼티 초기화식과 `init` 블록으로 채운다.

```kotlin
object Config {
    val version: String
    val maxConnections: Int
    init {
        // 인스턴스가 처음 접근될 때 딱 한 번 실행된다
        version = System.getProperty("app.version") ?: "dev"   // JVM 백엔드 예시
        maxConnections = 10
        println("Config 초기화됨")
    }
}
```

`init` 블록과 프로퍼티 초기화가 실행되는 순서는 일반 클래스와 완전히 같다(21장 클래스와 생성자). 위에서 아래로, 선언한 순서대로 실행된다. 차이는 하나뿐이다. 이 초기화는 생성자를 호출할 때가 아니라 그 object에 처음 접근할 때, 프로그램 전체에서 딱 한 번 일어난다. 이 동작은 2절에서 자세히 살펴본다.

### 1.3 탈설탕: 하나의 클래스와 하나의 INSTANCE

`object`가 무엇으로 컴파일되는지 개념적으로 그려 보면 오해가 쉽게 풀린다. JVM 백엔드에서 `object Registry`는 대략 다음과 같은 클래스로 컴파일된다.

```java
// object Registry가 JVM 백엔드에서 컴파일되는 형태(개념적)
public final class Registry {
    // 유일한 인스턴스를 담는 정적 필드
    public static final Registry INSTANCE;

    // 정적 초기화 블록: 클래스가 처음 로드·초기화될 때 JVM이 한 번 실행
    static {
        INSTANCE = new Registry();   // 여기서 딱 한 번 생성
    }

    // 생성자는 private — 외부에서 새로 만들 수 없다
    private Registry() {
        // init 블록·프로퍼티 초기화가 여기 들어간다
    }

    // 멤버 메서드는 INSTANCE에 대한 "인스턴스 메서드"다 (static이 아님!)
    public final void put(String key, String value) { /* ... */ }
    public final String get(String key) { /* ... */ }
}
```

여기서 중요한 점은 두 가지이다. 첫째, `put`·`get`은 `static` 메서드가 **아니다**. 이 메서드들은 `INSTANCE`라는 실제 객체의 인스턴스 메서드이다. Kotlin에서 쓴 `Registry.put(...)`은 컴파일되면 `Registry.INSTANCE.put(...)`이 된다.

둘째, 인스턴스는 정적 초기화 블록(`static { ... }`, JVM 용어로 `<clinit>`) 안에서 딱 한 번 생성된다. 이 두 사실은 object가 정적 클래스라는 오해가 틀렸음을 보여 준다.

> [!warning] 흔한 오해
> "object는 정적 클래스일 뿐이다"라는 생각은 틀렸다. object는 실제 인스턴스이다. 근거는 세 가지이다.
>
> 1. 멤버가 static이 아니라 `INSTANCE`의 인스턴스 메서드로 컴파일된다.
> 2. object는 클래스를 상속하고 인터페이스를 구현할 수 있다. static 멤버 묶음은 그럴 수 없다.
> 3. object 인스턴스를 다른 함수에 값으로 넘길 수 있고, `is` 검사와 `as` 캐스팅의 대상이 된다.
>
> "정적 클래스"라는 비유는 겉모습(`Registry.put`)만 보고, 실제 동작(`INSTANCE`에 대한 다형적 메서드 호출)을 놓친다.

### 1.4 object의 위치: 최상위와 중첩은 되고 지역은 안 된다

`object` 선언은 파일 최상위(top-level)에 둘 수도 있고, 다른 클래스나 object 안에 중첩(nested)할 수도 있다. 그러나 **함수 본문 안에는 지역(local) object를 선언할 수 없다.**

```kotlin
object TopLevel { fun greet() = "hi" }        // OK: 최상위

class Outer {
    object Nested { fun greet() = "nested" }  // OK: 중첩 (static nested 클래스처럼)
}

fun demo() {
    // object Local { }   // 컴파일 에러: object 선언은 지역이 될 수 없다
    val anon = object { val x = 1 }            // OK: 이건 '선언'이 아니라 '표현식'
}
```

이유는 수명에 있다. 이름 있는 `object` 선언은 "프로그램 전체에서 유일하며 지연 초기화되는 싱글턴"을 뜻한다. 그런데 지역 스코프는 함수가 호출될 때마다 새로 만들어지므로, 그 안에 전역 싱글턴을 두면 개념이 충돌한다. 함수 안에서 한 번 쓰고 버릴 익명 객체가 필요하면 `object` 표현식(4절)을 쓴다. 같은 키워드이지만 성격은 정반대이다.

중첩 `object`는 바깥 인스턴스를 참조하지 않는 정적 중첩(static nested) 클래스에 해당한다. 바깥 인스턴스를 참조하는 `inner`는 object에 붙일 수 없다. 싱글턴이 특정 바깥 인스턴스에 묶이면 유일성이 깨지기 때문이다. 이 차이는 31장(중첩 클래스와 이너 클래스와 익명 객체)에서 자세히 비교한다.

---

## 2. object의 초기화: 언제, 어떻게, 몇 번

### 2.1 지연 초기화: 처음 접근하는 순간

`object`는 프로그램이 시작될 때가 아니라 그 object에 **처음 접근하는 순간** 초기화된다. 여기서 접근이란 object의 멤버를 읽거나 호출하는 것, 또는 object 자체를 값으로 참조하는 것을 말한다.

```kotlin
object Lazy {
    val timestamp = System.nanoTime()          // JVM 백엔드 예시
    init { println("Lazy 초기화: $timestamp") }
}

fun main() {
    println("main 시작")
    println("아직 Lazy 접근 안 함")
    val t = Lazy.timestamp                      // 여기서 처음 접근 → 초기화 발생
    println("접근 후: $t")
}
// 출력 순서:
// main 시작
// 아직 Lazy 접근 안 함
// Lazy 초기화: 12345...       ← 접근하는 그 순간 처음 실행
// 접근 후: 12345...
```

JVM 백엔드에서는 이 지연 초기화를 추가 비용 없이 얻는다. Java 가상 머신 명세(JVMS §5.5)는 클래스 초기화(`<clinit>` 실행)를 그 클래스가 처음 능동적으로 사용될 때까지 미루도록 규정한다. `object Lazy`의 인스턴스는 `Lazy` 클래스의 정적 초기화 블록에서 생성되므로, 인스턴스 생성도 `Lazy` 클래스에 처음 접근할 때까지 미뤄진다. Kotlin이 별도의 지연 로직을 넣는 것이 아니라 JVM의 클래스 로딩 규칙을 그대로 이용하는 것이다.

> [!note] 명세 기준
> object의 지연 초기화는 JVM 백엔드에서 JVM의 클래스 초기화 시맨틱에 의존한다. Kotlin/Native·Kotlin/JS·Wasm 백엔드에서는 초기화 시점 보장이 런타임 구현에 따라 다르며, 역사적으로 Kotlin/Native는 초기화 전략이 달랐다(과거에는 최상위/전역을 즉시 초기화했고, 이후 지연 초기화로 바뀌었다).
>
> 따라서 "object는 항상 첫 접근에 지연 초기화된다"는 정확히 말하면 **JVM 백엔드에서** 성립하는 설명이다. 다른 타깃에서는 "구현/런타임에 따라 다르다"고 한정하는 것이 옳다.

### 2.2 스레드 안전: 무엇이 보장하는가

여러 스레드가 동시에 `Lazy.timestamp`에 처음 접근하면 어떻게 될까? JVM 백엔드에서는 인스턴스가 정확히 한 번만 생성된다는 것이 보장된다. 근거는 JVM 명세 §5.5의 클래스 초기화 절차이다. 이 절차는 **클래스별 초기화 락**을 획득한 뒤 진행된다. 한 스레드가 클래스를 초기화하는 동안 다른 스레드는 그 락에서 대기하고, 초기화가 끝난 뒤에 완성된 상태를 본다.

```text
[스레드 A]  Lazy.timestamp 첫 접근
              │
              ├─ JVM: Lazy 클래스 초기화 락 획득
              │        <clinit> 실행: INSTANCE = new Lazy()
              │
[스레드 B]  Lazy.timestamp 첫 접근 (거의 동시)
              │
              └─ JVM: 락이 A에게 있음을 보고 대기(BLOCK)
                       A가 <clinit> 완료 후 락 해제
                       B는 완성된 INSTANCE를 관찰 → 재생성 없음
```

이 메커니즘 덕분에 Kotlin의 `object`를 쓰면 1.1절에서 본 이중 검사 락 관용구를 직접 작성하지 않고도 "초기화 온 디맨드 홀더(initialization-on-demand holder)" 관용구만큼 안전한 싱글턴을 얻는다. 즉 `object`는 스레드 안전한 지연 싱글턴이다. 단, 이 안전성을 보장하는 주체는 정확히 말해 JVM의 클래스 초기화 시맨틱이다.

> [!caution] 성능 주의
> JVM의 클래스 초기화 락은 초기화가 **끝난 뒤**에는 비용이 들지 않는다. 초기화가 끝난 클래스에 접근할 때 JIT는 락을 제거하고 정적 필드 읽기로 최적화한다. 따라서 매번 동기화하는 이중 검사 락과 달리, object 접근은 정상 경로에서 추가 비용이 사실상 없다. 신경 써야 할 부분은 락이 아니라 무거운 초기화 블록(2.4절)이다.

### 2.3 몇 번? 클래스로더당 하나라는 세부 조건

"인스턴스가 하나"라는 말에는 자주 생략되는 조건이 있다. JVM에서 클래스의 정체성은 클래스의 이진 이름과 그 클래스를 로드한 클래스로더의 쌍으로 결정된다. 같은 `object Registry`라도 서로 다른 두 클래스로더가 각각 로드하면, 클래스로더마다 별개의 `Registry` 클래스가 존재하고 그만큼 별개의 `INSTANCE`가 생긴다.

```text
ClassLoader A ─► Registry (클래스 A판) ─► INSTANCE_A
ClassLoader B ─► Registry (클래스 B판) ─► INSTANCE_B      // A와 다른 객체!
```

일반적인 애플리케이션에서는 코드가 하나의 애플리케이션 클래스로더 아래에서 실행되므로 이 차이가 드러나지 않는다. 그래서 "object = 프로세스당 싱글턴"으로 취급해도 괜찮다. 그러나 여러 클래스로더로 코드를 격리하는 컨테이너나 플러그인 아키텍처에서는 object가 유일하다는 전제가 깨질 수 있다.

이것은 Kotlin의 결함이 아니라 JVM 플랫폼의 근본 성질이며, Java의 `static` 필드도 정확히 같은 조건을 따른다. 요컨대 `object`의 유일성은 **하나의 클래스로더 안에서** 성립한다.

### 2.4 함정: 순환 참조와 무거운 초기화

`object`의 지연·단일 초기화는 강력하지만, 두 가지 경우에 문제가 생긴다.

첫째는 **순환 참조**이다. 두 object가 초기화할 때 서로를 참조하면, 한쪽이 아직 초기화를 마치지 못한 상태에서 다른 쪽이 그 값을 읽어 `null`(또는 기본값)을 보게 된다.

```kotlin
object A {
    val name = "A"
    val partnerName = B.name        // B를 초기화하려 시도
}
object B {
    val name = "B"
    val partnerName = A.name        // A는 아직 초기화 중 → A.name이 아직 미할당
}

fun main() {
    println(A.partnerName)          // 접근 순서에 따라 예측 불가한 결과
    println(B.partnerName)          // (플랫폼/JVM에서 한쪽은 null이 관찰될 수 있다)
}
```

JVM은 초기화 중인 클래스에 같은 스레드가 다시 진입하면 데드락을 일으키는 대신 락을 재귀적으로 통과시킨다. 그 결과 부분적으로 초기화된 상태가 노출된다. 위 코드에서 `A`에 먼저 접근하면 `A`의 초기화가 `B`를 초기화하려 하고, `B`의 초기화는 다시 `A.name`을 읽는다. 이 시점에 `A.name`은 아직 대입되지 않았을 수 있다. 결과는 초기화 순서에 따라 달라지는 미묘한 버그이다.

따라서 **object의 초기화 블록에서 다른 object를 참조하는 순환을 만들지 말아야 한다.** 상태끼리 의존해야 한다면 함수로 감싸 사용하는 시점에 읽거나, [[23 - 위임 프로퍼티]]의 `by lazy`로 실제 사용 시점까지 초기화를 미룬다.

둘째는 **무거운 초기화 비용**이다. object는 처음 접근한 스레드에서 동기적으로 초기화된다. 그래서 파일 읽기, 네트워크 호출, 큰 테이블 구축처럼 초기화가 무거우면 첫 접근이 그만큼 느려지고, 그동안 다른 스레드는 초기화 락에서 대기한다. object는 가볍고 안전한 지연 싱글턴에 적합하다. 초기화가 무겁고 실패할 수 있다면 명시적 팩토리(6~7절의 companion 팩토리)나 의존성 주입을 고려하는 편이 낫다.

---

## 3. object는 실제 인스턴스다: 상속·구현·다형성

### 3.1 object는 클래스를 상속하고 인터페이스를 구현한다

object가 정적 클래스가 아니라는 가장 강력한 증거는 `object`가 슈퍼타입을 가질 수 있다는 사실이다. 슈퍼타입은 인스턴스에 대한 개념이므로, 정적 멤버의 묶음은 인터페이스를 구현하거나 클래스를 상속할 수 없다. 그런데 object는 그렇게 할 수 있다.

```kotlin
interface Shape {
    val area: Double
    fun describe(): String
}

// 원점의 넓이 0짜리 특수 도형 — 유일하게 존재하는 '빈 도형' 싱글턴
object EmptyShape : Shape {
    override val area: Double = 0.0
    override fun describe(): String = "빈 도형"
}

fun printArea(shape: Shape) {           // Shape를 받는 함수
    println("${shape.describe()}: ${shape.area}")
}

fun main() {
    printArea(EmptyShape)               // => 빈 도형: 0.0
    val shapes: List<Shape> = listOf(EmptyShape)   // 컬렉션에 담긴다
    println(EmptyShape is Shape)        // => true  (is 검사의 대상)
}
```

`EmptyShape`는 `Shape` 타입의 인스턴스로서 `printArea`에 값으로 전달되고, `List<Shape>`에 담기며, `is Shape` 검사를 통과한다. "정적 클래스"라면 이 가운데 어느 것도 할 수 없다. 즉 object는 다형성에 온전히 참여하는 객체이다. 이 패턴은 뒤에서 볼 널 객체(Null Object) 패턴, 빈 컬렉션의 유일 원소(예: 표준 라이브러리는 빈 리스트를 하나의 object로 공유한다), 기본 전략(strategy) 객체 등에서 널리 쓰인다.

### 3.2 sealed 계층의 object: 상태 없는 변형

`object`가 슈퍼타입을 가질 수 있다는 점은 30장(sealed 클래스와 대수적 데이터 타입)에서 특히 유용하다. 봉인 계층의 어떤 변형이 상태를 갖지 않는 유일한 경우라면, 매번 새로 만드는 `class`보다 하나만 존재하는 `object`로 두는 것이 자연스럽다.

```kotlin
sealed interface LoadState {
    object Idle : LoadState                        // 상태 없음 → 싱글턴이 자연스럽다
    object Loading : LoadState                      // 마찬가지
    data class Success(val data: String) : LoadState  // 상태 있음 → 인스턴스 여럿
    data class Failure(val reason: String) : LoadState
}

fun render(state: LoadState): String = when (state) {
    LoadState.Idle -> "대기 중"
    LoadState.Loading -> "불러오는 중..."
    is LoadState.Success -> "완료: ${state.data}"
    is LoadState.Failure -> "실패: ${state.reason}"
    // else 불필요 — sealed + object/data class로 완전성 보장
}
```

`Idle`과 `Loading`은 아무 데이터도 담지 않으므로 인스턴스가 여러 개일 이유가 없다. object 하나면 충분하고, 메모리와 동등성 비교(참조 동일성 하나로 끝난다) 모두에서 이득이다. 반면 `Success`·`Failure`는 값을 담으므로 데이터 클래스여야 한다. "상태가 없으면 object, 상태가 있으면 data class"라는 기준은 enum과 sealed 중 하나를 고를 때([[29 - enum 클래스]], 30장 sealed 클래스)에도 반복해서 쓰인다.

### 3.3 data object: 자기 이름을 출력하는 싱글턴

일반 `object`의 `toString()`은 기본적으로 `클래스명@해시` 형태(예: `Idle@1a2b3c`)를 출력한다. object도 `Any.toString`의 기본 구현을 물려받기 때문이다. 3.2절의 `when`에서는 문제가 없지만, 로그나 디버깅에서 object를 출력하면 사람이 읽기 어렵다. Kotlin 1.9에서 안정화되어 2.x에서 표준이 된 `data object`가 이 문제를 해결한다.

```kotlin
sealed interface LoadState {
    data object Idle : LoadState
    data object Loading : LoadState
    data class Success(val data: String) : LoadState
}

fun main() {
    println(LoadState.Idle)             // => Idle       (data object: 이름을 toString으로)
    // 일반 object였다면 => LoadState$Idle@6d06d69c 같은 출력
    println(LoadState.Idle == LoadState.Idle)  // => true (안정적 equals/hashCode)
}
```

`data object`를 쓰면 컴파일러가 이름을 그대로 반환하는 `toString()`과, 싱글턴에 맞는 일관된 `equals`/`hashCode`(모든 인스턴스가 곧 자기 자신이므로 참조 동일성과 일치한다)를 생성한다. 일반 object도 인스턴스가 하나뿐이므로 `==`가 사실상 참조 동일성으로 동작한다. `data object`는 그 의미를 명시적으로 정하고, 사람이 읽기 좋은 문자열 표현까지 제공한다.

`data object`와 일반 `data class`의 차이는 27장(데이터 클래스)에서 자세히 다루므로, 여기서는 "object에도 data를 붙여 toString/equals를 다듬을 수 있다"는 점만 기억해 두자.

> [!note] 명세 기준
> `data object`가 자동으로 생성하는 것은 `toString`(이름 반환)과 싱글턴 시맨틱에 맞춘 `equals`/`hashCode`이다. `copy`와 `componentN`은 생성하지 않는다. object에는 복사할 프로퍼티도, 구조 분해할 위치 인자도 없기 때문이다. 주 생성자 프로퍼티를 근거로 6개 멤버를 만드는 `data class`와 가장 크게 다른 점이 여기에 있다.

### 3.4 object의 정체성과 `===`

object 인스턴스는 하나뿐이므로, 같은 object를 두 번 참조하면 언제나 같은 객체를 얻는다. 이는 [[10 - 불리언과 동등성과 동일성]]에서 본 `===`(참조 동일성)로 확인할 수 있다.

```kotlin
object Sun
fun main() {
    val a = Sun
    val b = Sun
    println(a === b)     // => true  — 언제나 동일 인스턴스
    println(a == b)      // => true  — 기본 equals는 참조 동일성이므로 일치
}
```

이 성질 덕분에 object는 정체성으로 비교하는 토큰으로 쓰기에 알맞다. 예를 들어 맵의 특수한 마커 키, 알고리즘의 종료 신호(sentinel), 두 코드 경로가 같은 전역 상태를 보는지를 참조 동일성으로 판단하는 경우에 쓸 수 있다. 인스턴스가 하나뿐이라는 보장이 `===`의 의미를 확실하게 만든다. 박싱 캐시 범위에 따라 `===`의 결과를 예측할 수 없는 원시 값(10장 불리언과 동등성과 동일성)과 대조적이다.

---

## 4. object 표현식: 같은 키워드, 정반대의 수명

### 4.1 선언과 표현식의 차이

`object` 키워드는 완전히 다른 두 가지를 만든다. 지금까지 본 **object 선언**(`object Name { ... }`)은 이름 있는 전역 싱글턴이다. 반면 **object 표현식**(`object : Super { ... }` 또는 `object { ... }`)은 그 자리에서 바로 만드는 익명 객체이며, 코드가 그 위치를 실행할 때마다 새 인스턴스가 생긴다.

```kotlin
// 선언: 프로그램 전체에서 유일, 지연 초기화
object GlobalCounter { var count = 0 }

fun makeListener(): Runnable {
    // 표현식: 이 함수를 부를 때마다 새 익명 객체 생성
    return object : Runnable {
        override fun run() { println("실행") }
    }
}

fun main() {
    val a = makeListener()
    val b = makeListener()
    println(a === b)     // => false  — 매번 새 인스턴스 (선언과 정반대)
}
```

같은 키워드로 유일한 싱글턴과 한 번 쓰고 버릴 익명 객체를 모두 표현하므로 처음에는 혼란스럽지만, 규칙은 단순하다. **이름이 있으면 선언(싱글턴)이고, `:`나 `{`가 바로 이어지며 값 자리에 놓이면 표현식(익명)이다.** 익명 객체의 캡처, 다중 슈퍼타입, 중첩 클래스와 이너 클래스의 비교 같은 세부는 31장(중첩 클래스와 이너 클래스와 익명 객체)에서 다룬다. 여기서는 싱글턴과 수명을 비교하는 데 필요한 만큼만 살펴본다.

### 4.2 익명 객체는 여러 슈퍼타입을 가질 수 있다

object 표현식에는 슈퍼타입을 여러 개 나열할 수 있고, 슈퍼타입 없이 임시 프로퍼티만 묶은 객체를 만들 수도 있다.

```kotlin
interface Clickable { fun click() }
interface Hoverable { fun hover() }

val widget = object : Clickable, Hoverable {   // 두 인터페이스를 동시에 구현
    override fun click() = println("클릭")
    override fun hover() = println("호버")
}

fun main() {
    val point = object {                        // 슈퍼타입 없는 익명 객체 (지역이라 x·y 접근 가능)
        val x = 10
        val y = 20
    }
    println("${point.x}, ${point.y}")           // => 10, 20
}
```

슈퍼타입 없는 익명 객체(`object { val x = 10 }`)에서 주의할 점은 그 **타입**이다. 이 익명 타입에는 이름이 없으므로, 고유 멤버(`x`, `y`)에 접근하려면 익명 타입이 정적으로 보이는 스코프 안에 있어야 한다. Kotlin 규칙에 따르면 object 표현식을 반환하는 함수나 프로퍼티의 타입이 `public`/`protected`로 노출될 때는 익명 타입이 아니라 그 슈퍼타입(슈퍼타입이 없으면 `Any`)으로 좁혀진다.

```kotlin
class Scope {
    // private → 익명 타입이 이 스코프 안에서 그대로 보인다 → x 접근 가능
    private val local = object { val x = 42 }
    fun read() = local.x                        // => 42  OK

    // public → 반환 타입이 Any로 좁혀진다 → 고유 멤버 접근 불가
    fun leak() = object { val y = 99 }          // 추론 타입: Any
}
fun main() {
    // Scope().leak().y  // 컴파일 에러: Any에는 y가 없다
}
```

이 규칙 때문에 슈퍼타입 없는 익명 객체의 고유 멤버는 지역 문맥이나 private 문맥에서만 쓸모가 있다. 공개 API 경계를 넘겨야 한다면 이름 있는 타입(인터페이스나 클래스)을 슈퍼타입으로 지정해야 한다. 이 타입 좁힘 규칙도 31장에서 자세히 설명한다.

### 4.3 두 object의 비교

| 축 | object 선언 | object 표현식 |
|---|---|---|
| 문법 | `object Name { }` | `object : Super { }` / `object { }` |
| 이름 | 있음 | 없음(익명) |
| 인스턴스 수 | 전 프로그램 하나(싱글턴) | 실행할 때마다 새로 |
| 초기화 시점 | 첫 접근에 지연(JVM 백엔드) | 표현식 평가 시 즉시 |
| 위치 | 최상위·중첩(지역 불가) | 어디서든(주로 지역) |
| 캡처 | 없음(전역이라 캡처할 스코프 없음) | 둘러싼 스코프의 변수 캡처 가능 |
| 대표 용도 | 전역 싱글턴, sealed의 상태 없는 변형 | 일회용 리스너·콜백·SAM 변환 |

특히 캡처 여부의 차이가 중요하다. 익명 객체는 [[14 - 클로저와 캡처]]에서 설명한 것처럼 둘러싼 함수의 지역 변수를 캡처해 보관한다. 반면 object 선언은 전역이어서 둘러싼 스코프가 없으므로 캡처할 것도 없다. 이 차이는 지역 object 선언을 금지하는 근본 이유(1.4절)와도 연결된다. 캡처가 필요한 상황은 정의상 지역이고, 지역이라면 표현식을 써야 하기 때문이다.

---

## 5. companion object: 클래스 안의 동반 인스턴스

### 5.1 문제: Kotlin에는 static이 없다

Kotlin에는 `static` 키워드가 없다. Java 개발자가 처음 부딪히는 문제가 바로 이것이다. 클래스에 속한 팩토리 메서드, 클래스 수준 상수, 인스턴스 없이 호출하는 유틸리티는 어디에 두어야 할까? Kotlin은 두 가지 방법을 제시한다.

특정 클래스에 개념적으로 속하지 않는 것은 파일 최상위(top-level) 함수·프로퍼티로 둔다([[04 - 파일과 패키지와 선언 - 프로그램의 골격]]). 특정 클래스에 속해야 하는 것, 특히 그 클래스의 `private` 생성자에 접근해야 하는 팩토리는 `companion object`에 둔다.

```kotlin
class User private constructor(val name: String, val id: Int) {
    // 클래스에 매인, private 생성자에 접근하는 팩토리 자리
    companion object {
        private var nextId = 1
        fun create(name: String): User = User(name, nextId++)   // private 생성자 호출 가능
    }
}

fun main() {
    val u = User.create("앨리스")       // 클래스 이름으로 호출 — static처럼 보인다
    println("${u.name} #${u.id}")       // => 앨리스 #1
    // val bad = User("밥", 2)          // 컴파일 에러: 생성자가 private
}
```

`User.create(...)`는 겉보기에도, 쓰는 방식도 Java의 `static` 팩토리 메서드와 같다. 이 겉모습 때문에 companion 멤버가 static이라는 오해가 생긴다.

### 5.2 companion 멤버는 static이 아니다

`companion object`는 이름 그대로 클래스의 "동반 객체"이다. 클래스마다 딱 하나 존재하는 **실제 object 인스턴스**이고, 그 안의 멤버는 이 인스턴스의 멤버이다. `User.create`가 static처럼 보이는 이유는 Kotlin이 클래스 이름으로 동반 객체 멤버에 접근하는 문법 설탕을 제공하기 때문이다. `create`가 static 메서드여서가 아니다.

JVM 백엔드에서 companion object가 컴파일되는 형태를 개념적으로 그려 보면 실제 구조가 드러난다.

```java
// class User { companion object { fun create(...) } } 의 컴파일 형태(개념적)
public final class User {
    private final String name;
    private final int id;
    private User(String name, int id) { this.name = name; this.id = id; }

    // 바깥 클래스에 동반 객체 인스턴스를 담는 static 필드가 하나 생긴다
    public static final Companion Companion = new Companion();

    // 동반 객체는 별도의 '중첩 클래스'로 컴파일된다
    public static final class Companion {
        private int nextId = 1;
        // create는 Companion 인스턴스의 '인스턴스 메서드'다 — static이 아니다!
        public final User create(String name) {
            return new User(name, this.nextId++);
        }
    }
}
```

여기서 볼 것은 두 가지이다. 첫째, `create`는 `Companion`이라는 중첩 클래스의 **인스턴스 메서드**이며 `static`이 아니다. 둘째, 바깥 클래스 `User`에는 `Companion`이라는 이름의 **static 필드**가 하나 생기고, 이 필드가 유일한 동반 객체 인스턴스를 담는다. Kotlin에서 쓴 `User.create(...)`는 컴파일되면 `User.Companion.create(...)`가 된다.

이 사실은 Java에서 이 코드를 호출할 때 분명하게 드러난다. Java에는 Kotlin의 문법 설탕이 없으므로 `User.create(...)`로 호출할 수 없고, 동반 객체 인스턴스를 명시적으로 거쳐야 한다.

```java
// Java에서 Kotlin의 companion object를 호출할 때
User u = User.Companion.create("밥");   // OK — Companion 인스턴스를 명시
// User u = User.create("밥");          // 컴파일 에러: create는 User의 static이 아니다
```

> [!warning] 흔한 오해
> "companion object의 멤버는 정적(static) 멤버다"라는 생각은 틀렸다. companion 멤버는 클래스마다 하나 존재하는 `Companion` 인스턴스의 인스턴스 멤버이다. Kotlin 소스에서 `클래스명.멤버`로 접근할 수 있는 것은 컴파일러가 제공하는 문법 설탕일 뿐이고, JVM 바이트코드 수준에서는 `클래스명.Companion.멤버`, 즉 인스턴스 메서드 호출이다. 실제 `static` 멤버로 만들려면 `@JvmStatic`(6절)이 필요하다.

### 5.3 클래스당 하나, 이름은 선택

한 클래스에는 companion object를 **하나만** 둘 수 있다. 이름을 지정하지 않으면 컴파일러가 `Companion`이라는 기본 이름을 붙이고, 원한다면 이름을 직접 지정할 수 있다.

```kotlin
class Circle private constructor(val radius: Double) {
    companion object Factory {              // 이름을 'Factory'로 지정
        fun of(radius: Double): Circle = Circle(radius)
    }
}

fun main() {
    val c1 = Circle.of(2.0)                 // 클래스명으로 직접 (권장)
    val c2 = Circle.Factory.of(3.0)         // 이름으로 명시해도 됨
    println(c1.radius + c2.radius)          // => 5.0
}
```

이름을 붙이면 Java 상호운용에서 `Circle.Factory.of(...)`처럼 그 이름으로 접근하게 되고, 확장 함수를 붙일 때도 이름이 쓸모 있다(7.3절). 이름 없는 companion은 자동으로 `Companion`이라는 이름을 얻으므로, Java에서는 언제나 `Circle.Companion.of(...)` 형태로 접근할 수 있다. 또한 companion에는 인스턴스가 아니라 **클래스 이름으로만** 접근할 수 있다는 점도 static과 미묘하게 다르다.

```kotlin
class Widget {
    companion object { fun make() = Widget() }
}
fun main() {
    Widget.make()               // OK — 클래스 이름으로
    val w = Widget()
    // w.make()                 // 컴파일 에러 — 인스턴스로 companion 멤버 접근 불가
}
```

Java에서는 인스턴스 참조로도 static 멤버에 접근할 수 있지만(권장하지 않는 관례이다), Kotlin의 companion은 이를 허용하지 않는다. 오히려 이 편이 더 명료하다. companion 멤버가 인스턴스가 아니라 클래스(타입)에 속한다는 점을 문법이 강제하기 때문이다.

### 5.4 companion object의 초기화 시점

companion object도 하나의 object이므로 지연·단일 초기화된다. 다만 초기화 시점은 바깥 클래스와 연결되어 있다. JVM 백엔드에서 동반 객체 인스턴스는 바깥 클래스의 정적 초기화 블록에서 생성되므로, **바깥 클래스가 초기화될 때 함께 초기화**된다. 바깥 클래스는 그 클래스의 인스턴스를 만들거나, 그 클래스의 다른 static 멤버(companion 멤버 포함)에 접근할 때 초기화된다.

```kotlin
class Machine {
    companion object {
        init { println("Machine.Companion 초기화") }
        val serialPrefix = "MX"
    }
    init { println("Machine 인스턴스 생성") }
}

fun main() {
    println("시작")
    println(Machine.serialPrefix)   // 여기서 companion 접근 → Machine 클래스 초기화
    println(Machine())              // 이미 초기화됨 → 인스턴스 생성만
}
// 시작
// Machine.Companion 초기화       ← companion 접근 시점
// MX
// Machine 인스턴스 생성
```

단, `const val`(6.3절)은 컴파일 시점 상수이므로 접근해도 클래스 초기화가 일어나지 않는다. 상수 값이 사용하는 곳에 인라인되기 때문이다. 이 내용은 11장(변수와 초기화)의 `const` 설명과 이어진다.

---

## 6. companion의 JVM 표현과 @Jvm 애노테이션

### 6.1 실제 static이 필요할 때: @JvmStatic

5.2절에서 봤듯이 companion 멤버는 기본적으로 `Companion` 인스턴스의 메서드이다. 그래서 Java에서는 `User.Companion.create(...)`로 호출해야 하므로 번거롭다. Kotlin 코드끼리만 호출한다면 이 차이가 드러나지 않으므로 신경 쓸 필요가 없다. 하지만 **Java에서 실제 static처럼 호출하고 싶다면** `@JvmStatic`을 붙인다. 그러면 컴파일러가 바깥 클래스에 실제 static 전달(forwarding) 메서드를 하나 더 생성한다.

```kotlin
class User private constructor(val name: String) {
    companion object {
        @JvmStatic
        fun create(name: String): User = User(name)
    }
}
```

이제 JVM 바이트코드에는 진입점이 두 개 생긴다. 원래의 인스턴스 메서드인 `User.Companion.create(...)`와, 이 메서드를 호출해 주는 실제 static 메서드 `User.create(...)`이다. 이제 Java에서도 자연스럽게 호출할 수 있다.

```java
User u = User.create("캐럴");   // OK — @JvmStatic 덕분에 진짜 static
```

```text
@JvmStatic 없음:
   User.class
     └─ static field Companion
   User$Companion.class
     └─ instance method create()          // Java: User.Companion.create()

@JvmStatic 있음:
   User.class
     ├─ static field Companion
     └─ static method create() ───┐        // Java: User.create()  ← 추가됨
   User$Companion.class           │
     └─ instance method create() ◄┘ 위임
```

핵심은 `@JvmStatic`이 **Java 상호운용을 위해 JVM 표현만 바꾸는 장치**라는 점이다. Kotlin 코드 안에서는 `@JvmStatic이 있든 없든` 똑같이 `User.create(...)`로 호출한다. 이 애노테이션의 전체 내용은 [[45 - Java 상호운용]]에서 다룬다.

### 6.2 필드로 노출하기: @JvmField

companion object의 프로퍼티는 기본적으로 접근자(getter/setter)를 거친다. 그런데 JVM 백엔드에서 companion 프로퍼티의 **백킹 필드가 어디에 생기는지**는 조금 복잡하다. 백킹 필드는 대개 바깥 클래스의 static 필드로 생성되고, 접근자는 `Companion` 인스턴스의 메서드가 된다. Java에서 이 값을 필드로 직접 읽으려면 `@JvmField`를 붙여 접근자를 없애고 공개 static 필드로 노출한다.

```kotlin
class Palette {
    companion object {
        @JvmField val DEFAULT_COLOR = "gray"    // Java: Palette.DEFAULT_COLOR (필드)
        val ACCENT_COLOR = "blue"                // Java: Palette.Companion.getACCENT_COLOR()
    }
}
```

```java
String d = Palette.DEFAULT_COLOR;                 // @JvmField → 직접 필드 접근
String a = Palette.Companion.getACCENT_COLOR();   // 기본 → 게터 경유
```

`@JvmField`는 값이 바뀌지 않는 단순한 상수(원시 타입이 아닌 값, 예를 들어 미리 만든 객체 인스턴스)를 Java에 필드로 깔끔하게 노출할 때 유용하다. 다만 접근자가 사라지므로, 나중에 계산 로직을 끼워 넣을 수 있는 유연성은 잃는다. 프로퍼티가 접근자를 추상화한 것이라는 큰 그림은 [[22 - 프로퍼티와 백킹 필드]]에서 설명한다.

### 6.3 컴파일 상수: const val은 어느 클래스로 가는가

companion 안에 `const val`을 두면 또 다르게 동작한다. `const val`은 컴파일 시점에 값이 확정되는 상수(원시 타입 또는 `String`이고 리터럴로 초기화)이다. JVM 백엔드에서는 **바깥 클래스의 `public static final` 필드**로 컴파일되고, 그 값은 사용하는 곳에 인라인된다.

```kotlin
class Physics {
    companion object {
        const val SPEED_OF_LIGHT = 299_792_458      // 컴파일 상수
        val computedPi = 3.14159                     // 런타임 초기화 (const 아님)
    }
}
```

```java
long c = Physics.SPEED_OF_LIGHT;                    // Java: 직접 static final, 값 인라인
double pi = Physics.Companion.getComputedPi();      // 게터 경유
```

`const val`, `@JvmField val`, 일반 `val`이 각각 어느 위치에 어떤 형태로 컴파일되는지 정리하면 다음과 같다.

| 선언(companion 안) | 백킹 위치(JVM 백엔드) | Java 접근 | 값 인라인 | 클래스 초기화 유발 |
|---|---|---|---|---|
| `const val X = 42` | 바깥 클래스 `public static final` | `C.X` | 예(사용처에 박힘) | 아니오 |
| `@JvmField val X = obj` | 바깥 클래스 `public static` 필드 | `C.X` | 아니오 | 예 |
| `val X = compute()` | 필드 + 접근자(게터는 Companion) | `C.Companion.getX()` | 아니오 | 예 |

`const val`의 값 인라인에는 대가가 있다. 상수를 참조하는 다른 모듈은 그 값을 자신의 바이트코드에 직접 넣는다. 그래서 상수 값을 바꾸고 이 클래스만 다시 컴파일하면, 이미 값을 넣어 둔 소비자 모듈은 **옛 값을 계속 쓴다**. 값이 바뀔 수 있다면 `const` 대신 일반 `val`을 쓰는 편이 안전하다. 상수의 컴파일 시맨틱과 "const val과 val은 같다"는 오해는 11장(변수와 초기화)에서 자세히 다룬다.

### 6.4 상수는 어디에 둘 것인가

`const val`은 최상위(top-level)에도, companion 안에도 둘 수 있다. 선택 기준은 그 상수가 개념적으로 어떤 타입에 속하는지이다.

```kotlin
// 특정 타입과 무관한 전역 상수 → 최상위 (파일 수준)
const val MAX_RETRIES = 3

class HttpClient {
    companion object {
        // HttpClient에 개념적으로 매인 상수 → companion
        const val DEFAULT_TIMEOUT_MS = 5_000
    }
}
```

최상위 `const val`은 파일 클래스(예: `MyFileKt`)의 static 필드로, companion의 `const val`은 그 클래스의 static 필드로 컴파일된다. 순수 Kotlin에서는 어느 쪽이든 성능 차이가 없으므로, 결국 이름공간(namespace) 설계의 문제이다. `HttpClient.DEFAULT_TIMEOUT_MS`는 `DEFAULT_TIMEOUT_MS`보다 어디에 속한 상수인지 분명하게 드러낸다.

반대로 상수 하나를 담으려고 companion을 만드는 것은 과할 수 있다. companion object는 클래스가 초기화될 때 인스턴스가 생기는데, `const val`만 있다면 값이 인라인되므로 그 인스턴스는 실제로 필요하지 않다. 상수만 둔다면 최상위가 더 가볍다는 관점도 합리적이다.

---

## 7. companion 활용: 인터페이스·팩토리·확장·연산자

### 7.1 companion object는 인터페이스를 구현한다

companion object도 object이므로, 3절에서 본 것처럼 **인터페이스를 구현하고 클래스를 상속할 수 있다.** "static 멤버 묶음"이라면 절대 할 수 없는 일이며, 이 능력 덕분에 companion은 단순한 static 대체품 이상의 역할을 한다.

```kotlin
interface Factory<T> {
    fun create(): T
}

class Robot private constructor(val id: Int) {
    companion object : Factory<Robot> {        // 동반 객체가 Factory를 구현
        private var counter = 0
        override fun create(): Robot = Robot(++counter)
    }
}

fun <T> buildTwo(factory: Factory<T>): List<T> = listOf(factory.create(), factory.create())

fun main() {
    // Robot의 companion을 Factory<Robot> 값으로 넘긴다 — static이라면 불가능
    val robots = buildTwo(Robot.Companion)     // 또는 Robot 자체를 companion으로 참조하는 문맥
    println(robots.map { it.id })              // => [1, 2]
}
```

`Robot.Companion`은 `Factory<Robot>` 타입의 값으로 `buildTwo`에 전달된다. 즉 클래스에 속한 팩토리를 일급 값으로 추상화해 다형적으로 다룰 수 있다. 이 패턴을 쓰면 여러 타입이 공통 팩토리 인터페이스를 구현하게 해서, 타입별 생성 로직을 같은 방식으로 주입하거나 교체하도록 설계할 수 있다.

### 7.2 팩토리 패턴과 이름 있는 생성

companion을 실전에서 가장 흔하게 쓰는 곳은 **이름 있는 팩토리 메서드**이다. 생성자는 클래스 이름 하나로만 호출하므로, 의미가 다른 생성 방법을 이름으로 구분할 수 없다. companion 팩토리는 이 한계를 넘고, private 생성자와 함께 쓰면 객체를 만드는 경로를 통제할 수 있다.

```kotlin
class Temperature private constructor(val celsius: Double) {
    companion object {
        fun fromCelsius(c: Double) = Temperature(c)
        fun fromFahrenheit(f: Double) = Temperature((f - 32) * 5 / 9)
        fun fromKelvin(k: Double) = Temperature(k - 273.15)
        val ABSOLUTE_ZERO = Temperature(-273.15)   // 미리 만든 특수 인스턴스
    }
}

fun main() {
    val t1 = Temperature.fromFahrenheit(212.0)
    println(t1.celsius)                            // => 100.0
    println(Temperature.ABSOLUTE_ZERO.celsius)     // => -273.15
}
```

세 가지 생성 경로 `fromCelsius`/`fromFahrenheit`/`fromKelvin`은 생성자로는 구분할 수 없는 의미(같은 `Double` 인자, 다른 해석)를 이름으로 명확하게 나눈다. private 생성자는 반드시 이 팩토리들을 거쳐서만 만들라는 계약을 강제한다. companion이 `Temperature`의 private 생성자에 접근할 수 있는 이유는 companion이 그 클래스의 **내부**이기 때문이다.

### 7.3 companion에 확장 함수 붙이기

companion object에는 확장 함수와 확장 프로퍼티를 붙일 수 있다. 이렇게 하면 클래스에 새로운 정적 팩토리를 나중에 추가한 것과 같은 효과가 난다. 확장 함수의 정적 디스패치 원리는 35장(확장 함수와 프로퍼티)에서 설명하므로, 여기서는 companion을 수신 타입으로 쓰는 형태만 살펴본다.

```kotlin
class Money(val cents: Long) {
    companion object            // 비어 있어도 선언은 필요하다
}

// companion object를 수신 타입으로 하는 확장 함수
fun Money.Companion.fromDollars(dollars: Double): Money = Money((dollars * 100).toLong())

fun main() {
    val m = Money.fromDollars(9.99)    // 확장이지만 클래스명으로 호출되어 팩토리처럼 보인다
    println(m.cents)                   // => 999
}
```

주의할 점은 확장을 붙이려면 companion object가 **존재해야** 한다는 것이다. 비어 있더라도 `companion object`가 선언되어 있어야 `Money.Companion`이라는 확장 수신 타입을 쓸 수 있다.

이 기법은 라이브러리 사용자가 라이브러리 타입의 companion에 자신만의 팩토리를 덧붙일 수 있게 하는 확장점(extension point)으로 쓰인다. companion에 이름을 붙였다면(`companion object Factory`) 확장은 `fun Money.Factory.xxx()`로 쓴다.

### 7.4 companion과 연산자 invoke

companion object에 `operator fun invoke`를 정의하면, 생성자를 호출하듯 클래스 이름을 함수처럼 호출하는 관용구를 만들 수 있다. 연산자 관례 `invoke`는 [[19 - 연산자 오버로딩과 관례]]에서 자세히 다루며, 여기서는 companion과 함께 쓰는 모습만 본다.

```kotlin
class Logger private constructor(val tag: String) {
    companion object {
        // invoke 연산자를 companion에 정의
        operator fun invoke(tag: String): Logger = Logger(tag)
    }
    fun log(msg: String) = println("[$tag] $msg")
}

fun main() {
    val log = Logger("APP")        // Logger.Companion.invoke("APP") — 생성자처럼 보인다
    log.log("시작")                // => [APP] 시작
}
```

`Logger("APP")`은 얼핏 생성자 호출처럼 보이지만, 실제로는 companion object의 `invoke` 연산자를 호출한다. 이 기법은 팩토리를 생성자처럼 자연스럽게 노출하고 싶을 때 쓰인다. 하지만 실제 생성자와 헷갈릴 수 있으므로, 남용하면 코드를 읽기 어려워진다. 표준 라이브러리도 이 패턴을 신중하게 쓴다(예: 일부 타입의 팩토리).

### 7.5 companion object에 무엇을 담아야 하나

companion에 무엇이든 넣을 수 있다고 해서 무엇이든 넣어야 하는 것은 아니다. companion object에는 다음이 잘 맞는다.

- private 생성자에 접근하는 **팩토리 메서드**(7.2절)
- 그 타입에 개념적으로 속한 **상수**(6.4절)
- 그 타입의 팩토리를 **다형적으로 추상화**하는 인터페이스 구현(7.1절)
- 나중에 확장으로 팩토리를 덧붙이기 위한 **확장점**(7.3절)

반대로 특정 클래스와 관계없는 순수 유틸리티 함수는 companion보다 **최상위 함수**로 두는 편이 낫다. 불필요하게 클래스에 소속시키지 않아도 되고, import와 이름공간도 깔끔하다. Java의 "모든 것을 클래스 안에" 두는 습관 때문에 유틸리티를 무심코 companion에 몰아넣는 것은 Kotlin다운 설계가 아니다. 이 함수가 정말 이 타입에 속하는지, 아니면 그저 static이 필요해서 넣는지를 따져 봐야 한다.

---

## 8. object·companion의 함정과 안티패턴

### 8.1 싱글턴이라는 이름의 전역 가변 상태

`object`로 쉽게 만들 수 있는 것은 싱글턴이다. 하지만 싱글턴은 곧 **전역 상태**가 될 수 있고, 전역 가변 상태는 테스트와 동시성, 코드 추론을 어렵게 만드는 대표적인 함정이다.

```kotlin
object GlobalCache {
    val store = mutableMapOf<String, Int>()   // 전역 가변 상태 — 위험 신호
}
```

`GlobalCache.store`는 프로그램 어디서든 읽고 쓸 수 있으므로, 두 스레드가 동시에 수정하면 경쟁 조건이 생긴다(가변 컬렉션은 스레드 안전하지 않다. [[38 - 컬렉션 - 읽기 전용과 가변]], 43장 동시성과 메모리 모델 참고). 또한 테스트 사이에 상태가 새어(leak) 나가, 한 테스트가 다른 테스트의 결과를 오염시킨다.

**object의 지연·스레드 안전 초기화는 인스턴스 생성을 보장할 뿐, 그 안의 가변 상태를 동기화하지는 않는다.** 이 차이를 놓치면 "object니까 스레드 안전하겠지"라고 착각하게 된다. object 안에 가변 상태를 둔다면 그 상태의 동기화는 따로 처리해야 한다.

### 8.2 무겁거나 실패할 수 있는 초기화를 object에 두기

2.4절에서 봤듯이 object 초기화는 처음 접근한 스레드에서 동기적으로 일어난다. 그런데 초기화 중에 예외가 발생하면 어떻게 될까? JVM 백엔드에서 `<clinit>`이 예외를 던지면 그 클래스는 `ExceptionInInitializerError`로 표시되고, 이후 그 클래스에 접근하려는 모든 시도가 `NoClassDefFoundError`로 실패한다. 회복할 수 없는 상태가 되는 것이다.

```kotlin
object Bootstrap {
    val config = loadConfigOrThrow()   // 여기서 예외가 나면 Bootstrap은 영구히 사용 불가
}
```

즉 object는 실패하지 않는 가벼운 초기화를 전제로 한다. 초기화가 파일이나 네트워크 같은 외부 자원에 의존해 실패할 수 있다면, 자원을 얻는 코드를 object 초기화에서 명시적 함수나 팩토리로 옮겨야 한다. 그래야 실패를 정상 흐름([[41 - 예외와 Nothing]]의 `Result` 등)으로 다룰 수 있다.

### 8.3 companion의 상태 공유 오해

companion object의 프로퍼티는 클래스의 **모든 인스턴스가 공유하는** 상태이다. 인스턴스별 상태가 아니라 클래스 수준 상태이며, Java의 static 필드처럼 공유된다. 이것을 인스턴스별 상태로 착각하면 버그가 생긴다.

```kotlin
class Counter {
    companion object { var total = 0 }   // 모든 인스턴스가 공유
    val id: Int
    init { id = ++total }                // companion의 total을 증가시켜 id 부여
}

fun main() {
    val a = Counter(); val b = Counter(); val c = Counter()
    println("${a.id} ${b.id} ${c.id}")   // => 1 2 3  (total은 공유되어 누적)
    println(Counter.total)               // => 3
}
```

이 예에서 `total`을 공유하는 것은 인스턴스에 순번을 매기려는 의도된 설계이다. 하지만 무심코 companion에 둔 가변 프로퍼티는 "왜 여러 인스턴스가 같은 값을 보는가"라는 혼란을 일으킨다. companion의 상태는 언제나 클래스 전역이라는 점을 분명히 의식하고 써야 한다.

### 8.4 object와 companion 대신 최상위 선언

Kotlin다운 설계를 할 때 반복되는 질문은 "이것이 object/companion이어야 하는가, 아니면 최상위 선언이면 되는가"이다. 순수 함수의 묶음(수학 유틸리티 등)을 담으려고 만든 object는 대개 불필요하다. 최상위 함수면 충분하고 더 가볍다.

```kotlin
// 안티패턴: 순수 함수를 담기 위한 object (Java의 유틸 클래스 습관)
object MathUtils {
    fun square(x: Int) = x * x
}

// Kotlin다운 방식: 최상위 함수
fun square(x: Int) = x * x
```

JVM 백엔드에서 `MathUtils.square(3)`은 `MathUtils`라는 싱글턴 인스턴스를 초기화하고 그 인스턴스 메서드를 호출한다. 반면 최상위 `square(3)`은 파일 클래스의 static 메서드 하나를 호출할 뿐이다. 후자가 더 단순하고, import도 함수 단위로 깔끔하다.

object는 상태를 담거나, 슈퍼타입을 구현하거나, 인터페이스 타입의 값으로 넘겨야 할 때 쓸모가 있다. 그런 이유 없이 순수 함수만 담는다면 최상위 선언이 낫다. companion에도 같은 논리가 적용된다(7.5절). 클래스에 속하지 않는 유틸리티라면 최상위에 둔다.

### 8.5 object, enum, sealed 중 무엇을 고를까

"상태 없는 유일 값"을 표현하는 세 도구는 역할이 겹쳐 보인다. 선택 기준을 정리하면 다음과 같다.

```text
값이 하나뿐이고 다형성/상속이 필요? ─► object (또는 sealed 안의 object)
값이 유한 개의 열거이고 서로 순번·이름이 필요? ─► enum ([[29 - enum 클래스]])
값이 유한 개이되 각각 다른 데이터를 담아야? ─► sealed + data class/object ([[30 - sealed 클래스와 대수적 데이터 타입]])
```

`object`는 "단 하나"에, `enum`은 "정해진 여러 개"에, `sealed`는 "정해진 여러 개이지만 각각 모양이 다를 수 있음"에 대응한다. 세 도구는 각각 이 장과 29장(enum 클래스), 30장(sealed 클래스와 대수적 데이터 타입)에서 자세히 다룬다. 실전에서는 sealed 계층 안에서 상태 없는 변형은 `object`로, 상태 있는 변형은 `data class`로 두는 조합(3.2절)이 가장 자주 쓰인다.
