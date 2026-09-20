---
title: object와 companion - 싱글턴과 동반 객체
date: 2026-07-13
tags: [kotlin, object, companion, singleton, jvmstatic, 학습노트]
---

**이 장이 답하는 질문:**

- `object` 선언 한 줄은 정말 "정적 클래스"인가, 아니면 진짜 인스턴스인가 — 그 인스턴스는 언제, 어떤 스레드가, 몇 번 만드는가?
- `object`가 "지연 초기화·스레드 안전"이라는 말은 무엇에 근거하는가 — 이 보장은 어느 백엔드에서 무엇에 의존하며, 어디서 깨질 수 있는가?
- `companion object`의 멤버는 Java의 `static` 멤버와 같은가? Java에서 `Foo.bar()`로 부를 수 있는가, 아니면 `Foo.Companion.bar()`여야 하는가?
- 클래스 이름으로 `Point.origin()`을 부를 수 있는데 왜 이것이 "정적 메서드"가 아니라 "동반 객체 인스턴스의 메서드"라고 하는가?
- `object`는 인터페이스를 구현하고 클래스를 상속할 수 있는데, 그렇다면 싱글턴을 다형적으로 넘길 수 있다는 뜻인가?
- `object 표현식`(익명 객체)과 `object 선언`(싱글턴)은 왜 같은 키워드를 쓰면서 정반대의 수명을 갖는가?
- `companion object`에 인터페이스를 구현시키고 확장 함수를 붙이는 것은 왜 가능하며, 어떤 실전 패턴을 열어 주는가?
- 상수는 최상위(top-level)에 둬야 하는가 `companion object` 안에 둬야 하는가 — `const val`은 어느 클래스의 어느 필드로 컴파일되는가?

---

앞선 [[21 - 클래스와 생성자]]에서 우리는 클래스가 `new` 없이 어떻게 인스턴스화되고 `init` 블록·프로퍼티 초기화가 어떤 순서로 도는지를 기계 수준에서 봤다. [[24 - 상속과 오버라이딩과 초기화 순서]]는 기본이 `final`인 Kotlin에서 `open`·`override`가 어떻게 다형성을 여는지를, [[25 - 인터페이스]]는 상태 없는 계약과 기본 구현을 다뤘다. 이 장은 그 위에 하나의 특수한 클래스 형태를 얹는다. **인스턴스가 정확히 하나뿐인 클래스**, 즉 싱글턴을 언어가 직접 문법으로 제공하는 `object`와, 그것을 클래스에 못 박아 "정적 멤버처럼 보이지만 실은 인스턴스 멤버"인 자리를 만드는 `companion object`다.

이 장이 다루는 것과 다루지 않는 것의 경계를 먼저 못박자. `object 선언`(싱글턴의 정체·초기화·다형성)과 `companion object`(클래스당 하나의 동반 인스턴스·팩토리·JVM 표현)가 이 장의 심장이다. `object 표현식`(익명 객체)은 여기서 싱글턴과의 대비를 위해 필요한 만큼 다루되, 중첩·이너 클래스와 익명 객체의 캡처·다중 수신자 같은 세부는 [[31 - 중첩 클래스와 이너 클래스와 익명 객체]]가 정본으로 소유한다. `companion object`에 확장 함수를 붙일 수 있다는 사실은 여기서 보이되, 확장의 정적 디스패치 원리 자체는 [[35 - 확장 함수와 프로퍼티]]가 소유한다. `data object`는 [[27 - 데이터 클래스]]와 [[30 - sealed 클래스와 대수적 데이터 타입]]의 교차점이므로 이 장에서는 일반 `object`와의 차이만 짚는다. `const val`이 왜 컴파일 상수인지의 정본은 [[11 - 변수와 초기화 - val과 const와 lateinit]]이며, 여기서는 상수를 "어디에 두느냐"의 결정에서만 다시 만난다.

논증의 궤적은 이렇다. 먼저 `object`를 손으로 쓴 등가의 클래스로 탈설탕(desugar)해 컴파일러가 하는 일 — 하나의 클래스, 하나의 `INSTANCE` 필드, 그 필드를 채우는 정적 초기화 — 를 드러내고, 여기서 **object는 정적 클래스가 아니라 진짜 인스턴스다(M24)** 라는 첫 번째 오해를 정면으로 부순다. 그다음 그 초기화가 "지연·스레드 안전"이라는 주장의 근거(JVM 백엔드의 클래스 초기화 락)와 한계(다른 백엔드, 순환 참조, 클래스로더)를 판다. 이어 object가 인터페이스를 구현하고 상속할 수 있음을 보여 싱글턴을 다형적으로 다루는 길을 열고, `object 표현식`과의 수명 대비로 키워드의 이중성을 정리한다. 후반부는 `companion object`로 넘어가 **동반 객체 멤버는 static이 아니다(M25)** 를 JVM 바이트코드 수준에서 교정하고, `@JvmStatic`·`@JvmField`·`const`가 각각 무엇을 어느 클래스로 옮기는지, 그리고 companion에 인터페이스·확장·연산자를 얹는 실전 패턴으로 마무리한다.

---

## 1. object 선언 — 싱글턴을 언어가 흡수하다

### 1.1 문제: 손으로 짜는 싱글턴의 함정

프로그램에는 "단 하나만 존재해야 의미 있는" 것들이 있다. 애플리케이션 전역의 설정 레지스트리, 로그를 모으는 수집기, 수학 상수와 순수 함수의 묶음, 빈 컬렉션을 나타내는 유일한 원소 — 이들은 인스턴스가 여럿이면 오히려 버그다. 전통적으로 이 "인스턴스가 하나뿐임"을 보장하려면 이른바 싱글턴 패턴을 손으로 짜야 했다. Java에서 이 코드는 악명 높게 미묘하다.

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

이 코드의 진짜 문제는 "인스턴스가 하나임을 보장하는 로직"이 개발자의 손에 있다는 것이다. 동시성 아래에서 `if (instance == null)`은 두 스레드가 동시에 통과할 수 있고, 그러면 싱글턴이 둘 생긴다. 이를 막으려면 `synchronized`나 이중 검사 락(double-checked locking)에 `volatile`을 얹는 정교한 관용구가 필요하고, 그 관용구는 메모리 모델을 정확히 이해하지 못하면 미묘하게 틀린다. 이 이야기의 정본 배경은 [[43 - 동시성과 메모리 모델]]에 있다.

Kotlin은 이 반복되는 위험을 언어 기능으로 흡수했다. `class` 대신 `object` 키워드로 선언하면, 그 순간 "인스턴스가 정확히 하나"라는 보장이 **컴파일러와 런타임의 책임**으로 넘어간다.

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

`Registry`는 타입 이름인 동시에 그 타입의 유일한 인스턴스를 가리키는 값이다. `Registry.put(...)`은 "`Registry`라는 클래스의 정적 메서드"를 부르는 것처럼 보이지만, 이 겉모습이 바로 이 장이 부술 첫 오해다.

### 1.2 object에는 생성자가 없다

`object` 선언에는 주 생성자도 부 생성자도 없다. 인스턴스를 하나만, 그것도 런타임이 만들기 때문에 "누가 어떤 인자로 만들지"를 지정할 여지가 없기 때문이다. 그래서 `object Registry(config: Config)` 같은 선언은 문법 오류다. 초기 상태가 필요하면 프로퍼티 초기화식과 `init` 블록으로 채운다.

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

`init` 블록과 프로퍼티 초기화의 실행 순서 규칙은 일반 클래스와 완전히 동일하다([[21 - 클래스와 생성자]]). 위에서 아래로, 선언 순서대로 실행된다. 차이는 단 하나 — 이 초기화가 "생성자 호출 시점"이 아니라 "그 object에 처음 접근하는 시점"에, 프로그램 전체에서 딱 한 번 일어난다는 것이다. 이것을 2절에서 파헤친다.

### 1.3 탈설탕: 하나의 클래스와 하나의 INSTANCE

`object`가 무엇으로 컴파일되는지 개념적으로 그려 보면 오해가 저절로 풀린다. JVM 백엔드에서 `object Registry`는 대략 이런 클래스로 태어난다.

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

여기서 결정적인 두 가지를 읽어야 한다. 첫째, `put`·`get`은 `static` 메서드가 **아니다**. 그것들은 `INSTANCE`라는 실제 객체에 대한 인스턴스 메서드다. Kotlin에서 `Registry.put(...)`이라고 쓴 것은 컴파일 후 `Registry.INSTANCE.put(...)`으로 번역된다. 둘째, 인스턴스는 정적 초기화 블록(`static { ... }`, JVM 용어로 `<clinit>`) 안에서 단 한 번 생성된다. 이 두 사실이 M24를 정면으로 반박한다.

> **흔한 오해 (M24)**: "object는 정적 클래스일 뿐이다." — 아니다. object는 진짜 인스턴스다. 그 증거는 세 가지다. (1) 멤버가 static이 아니라 `INSTANCE`에 대한 인스턴스 메서드로 컴파일된다. (2) object는 클래스를 상속하고 인터페이스를 구현할 수 있다 — static 멤버 묶음은 그럴 수 없다. (3) object 인스턴스를 다른 함수에 값으로 넘길 수 있고, `is` 검사·`as` 캐스팅의 대상이 된다. "정적 클래스"라는 은유는 겉모습(`Registry.put`)만 붙잡고 실체(`INSTANCE`에 대한 다형적 메서드 호출)를 놓친다.

### 1.4 object의 위치: 최상위·중첩은 되고 지역은 안 된다

`object` 선언은 파일 최상위(top-level)에 둘 수도 있고, 다른 클래스나 object 안에 중첩(nested)할 수도 있다. 그러나 **함수 본문 안에 지역(local) object 선언은 불가능하다.**

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

이유는 수명에 있다. 이름 있는 `object 선언`은 "프로그램 전체에서 유일하며 지연 초기화되는 싱글턴"이라는 의미인데, 함수가 호출될 때마다 새로 만들어지는 지역 스코프에 그런 전역 싱글턴을 두는 것은 개념이 충돌한다. 함수 안에서 일회용 익명 객체가 필요하면 그것은 `object 표현식`(4절)의 몫이다 — 같은 키워드지만 정반대의 물건이다. 중첩 `object`는 바깥 인스턴스를 참조하지 않는 정적 중첩(static nested)에 해당하며, 바깥 인스턴스를 참조하는 `inner`는 object에 붙일 수 없다(싱글턴이 특정 바깥 인스턴스에 매이면 유일성이 깨진다). 이 대비의 정본은 [[31 - 중첩 클래스와 이너 클래스와 익명 객체]]다.

---

## 2. object의 초기화 — 언제, 어떻게, 몇 번

### 2.1 지연 초기화: 처음 접근하는 순간

`object`의 초기화는 프로그램 시작 시점이 아니라, 그 object에 **처음 접근하는 순간** 일어난다. 이 "접근"이란 object의 멤버를 읽거나 부르는 것, 혹은 object 자체를 값으로 참조하는 것을 말한다.

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

JVM 백엔드에서 이 "지연"은 공짜로 얻어진다. Java 가상 머신 명세(JVMS §5.5)는 클래스의 초기화(`<clinit>` 실행)를 "그 클래스가 처음 능동적으로 사용될 때"까지 미루도록 규정한다. `object Lazy`의 인스턴스는 `Lazy` 클래스의 정적 초기화 블록에서 생성되므로, 결국 `Lazy` 클래스에 처음 접근할 때까지 인스턴스 생성이 미뤄진다. Kotlin이 별도의 지연 로직을 짜 넣는 게 아니라 JVM의 클래스 로딩 규칙에 얹혀 가는 것이다.

> **명세 정밀**: object의 "지연 초기화"는 JVM 백엔드에서는 JVM의 클래스 초기화 시맨틱에 의존한다. Kotlin/Native·Kotlin/JS·Wasm 백엔드에서는 초기화 시점 보장이 런타임 구현에 따라 다르며, 역사적으로 Kotlin/Native는 초기화 전략이 달랐다(과거 최상위/전역의 즉시 초기화, 이후 지연화로의 변화). 따라서 "object는 항상 첫 접근에 지연 초기화된다"는 정확히는 **JVM 백엔드에서** 성립하는 서술이고, 다른 타깃에서는 "구현/런타임에 따라"로 한정하는 것이 옳다.

### 2.2 스레드 안전: 무엇이 보장하는가

여러 스레드가 동시에 `Lazy.timestamp`에 처음 접근하면 어떻게 될까? JVM 백엔드에서는 인스턴스가 정확히 한 번만 생성됨이 보장된다. 그 근거는 JVM 명세 §5.5의 클래스 초기화 절차인데, 이 절차는 **클래스별 초기화 락**을 획득하고 진행한다. 한 스레드가 클래스 초기화 중이면 다른 스레드는 그 락에서 대기하고, 초기화가 끝난 뒤 완성된 상태를 본다.

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

이 메커니즘 덕분에 Kotlin의 `object`는 1.1절에서 본 이중 검사 락 관용구를 개발자가 손으로 짤 필요 없이 "초기화 온 디맨드 홀더(initialization-on-demand holder)" 관용구의 안전성을 공짜로 제공한다. 즉, `object`는 스레드 안전한 지연 싱글턴이다 — 단, 이 문장의 정확한 주어는 "JVM 클래스 초기화 시맨틱"이다.

> **성능 주의**: JVM의 클래스 초기화 락은 초기화가 **완료된 뒤**에는 비용이 없다. 초기화가 끝난 클래스에 접근할 때 JIT는 락을 제거하고 정적 필드 읽기로 최적화한다. 따라서 object 접근은 "매번 동기화하는" 이중 검사 락과 달리 정상 경로에서 추가 비용이 사실상 없다. 걱정할 지점은 락이 아니라 무거운 초기화 블록(2.4절)이다.

### 2.3 몇 번? 클래스로더당 하나라는 미세한 진실

"인스턴스가 하나"라는 말에는 종종 생략되는 조건이 있다. JVM에서 클래스의 정체성은 (클래스의 이진 이름 + 그것을 로드한 클래스로더)의 쌍으로 결정된다. 같은 `object Registry`라도 서로 다른 두 클래스로더가 각각 로드하면, 각 클래스로더의 세계에서 별개의 `Registry` 클래스가 존재하고 따라서 별개의 `INSTANCE`가 생긴다.

```text
ClassLoader A ─► Registry (클래스 A판) ─► INSTANCE_A
ClassLoader B ─► Registry (클래스 B판) ─► INSTANCE_B      // A와 다른 객체!
```

일상적인 애플리케이션에서는 단일 애플리케이션 클래스로더 아래 코드가 돌므로 이 구분이 드러나지 않고, "object = 프로세스당 싱글턴"으로 취급해도 무방하다. 그러나 여러 클래스로더로 격리하는 컨테이너·플러그인 아키텍처에서는 "object가 진짜 유일한가"가 무너질 수 있다. 이것은 Kotlin의 결함이 아니라 JVM 플랫폼의 근본 성질이며, Java의 `static` 필드도 정확히 같은 조건에 놓인다. 요컨대 `object`의 유일성은 **하나의 클래스로더 안에서** 성립한다.

### 2.4 함정: 순환 참조와 무거운 초기화

`object`의 지연·단일 초기화는 강력하지만, 두 가지 지점에서 발이 걸린다.

첫째는 **순환 참조**다. 두 object가 초기화 시점에 서로를 참조하면, 한쪽이 아직 초기화를 마치지 못한 상태에서 다른 쪽이 그것을 읽어 `null`(또는 기본값)을 보는 함정이 생긴다.

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

JVM은 초기화 중인 클래스에 같은 스레드가 재진입하면 락을 재귀적으로 통과시키므로(데드락 대신) 부분 초기화된 상태를 노출한다. 위에서 `A`를 먼저 접근하면 `A`의 초기화가 `B`를 초기화하려 하고, `B`의 초기화가 다시 `A.name`을 읽는데 이 시점에 `A.name`은 아직 대입되지 않았을 수 있다. 결과는 초기화 순서에 의존하는 미묘한 버그다. 교훈: **object의 초기화 블록에서 다른 object를 참조하는 사이클을 만들지 말라.** 상태 간 의존이 필요하면 지연 접근(함수로 감싸 사용 시점에 읽기)이나 [[23 - 위임 프로퍼티]]의 `by lazy`로 실제 사용 시점까지 미룬다.

둘째는 **무거운 초기화 비용**이다. object의 초기화는 첫 접근 스레드에서 동기적으로 일어나므로, 초기화가 무거우면(파일 읽기, 네트워크, 큰 테이블 구축) 첫 접근이 그만큼 느려지고 그동안 다른 스레드는 초기화 락에서 대기한다. object는 "가볍고 안전한 지연 싱글턴"에 적합하며, 초기화가 무겁고 실패 가능하다면 명시적 팩토리(6~7절의 companion 팩토리)나 의존성 주입을 고려하는 편이 낫다.

---

## 3. object는 진짜 인스턴스다 — 상속·구현·다형성

### 3.1 object는 클래스를 상속하고 인터페이스를 구현한다

M24를 부수는 가장 강력한 증거는, `object`가 슈퍼타입을 가질 수 있다는 사실이다. 정적 멤버의 묶음은 결코 인터페이스를 구현하거나 클래스를 상속할 수 없다 — 슈퍼타입이란 인스턴스에 대한 개념이기 때문이다. 그런데 object는 그것을 한다.

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

`EmptyShape`는 `Shape`라는 타입의 인스턴스로서 `printArea`에 값으로 전달되고, `List<Shape>`에 담기고, `is Shape` 검사를 통과한다. 이 모든 것이 "정적 클래스"라면 불가능하다. object가 다형성의 완전한 시민임을 보여 주는 것이다. 이 패턴은 뒤에서 볼 널 객체(Null Object) 패턴, 빈 컬렉션의 유일 원소(예: 표준 라이브러리가 빈 리스트를 하나의 object로 공유), 기본 전략(strategy) 객체 등에서 광범위하게 쓰인다.

### 3.2 sealed 계층에서의 object — 상태 없는 변형

`object`가 슈퍼타입을 가질 수 있다는 사실은 [[30 - sealed 클래스와 대수적 데이터 타입]]에서 특히 빛난다. 봉인 계층의 어떤 변형이 "상태를 갖지 않는 유일한 경우"라면, 그것을 매번 새로 만드는 `class`가 아니라 하나만 존재하는 `object`로 두는 것이 자연스럽다.

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

`Idle`과 `Loading`은 어떤 데이터도 담지 않으므로 인스턴스가 여럿일 이유가 없다 — object 하나면 충분하고, 그것이 메모리와 동등성 비교(참조 동일성 하나로 끝) 양쪽에서 이득이다. 반면 `Success`·`Failure`는 값을 담으므로 데이터 클래스여야 한다. 이 "상태 없으면 object, 상태 있으면 data class"라는 구분은 enum과 sealed의 선택([[29 - enum 클래스]], [[30 - sealed 클래스와 대수적 데이터 타입]])에서도 반복되는 핵심 감각이다.

### 3.3 data object — 이름을 말하는 싱글턴

일반 `object`의 `toString()`은 기본적으로 `클래스명@해시` 형태(예: `Idle@1a2b3c`)를 낸다 — object라도 `Any.toString`의 기본 구현을 물려받기 때문이다. 3.2절의 `when`에서는 문제없지만, 로그나 디버깅에서 object를 출력하면 사람이 읽기 나쁘다. Kotlin 1.9에서 안정화되어 2.x에서 표준이 된 `data object`가 이 틈을 메운다.

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

`data object`는 이름을 그대로 반환하는 `toString()`과, 싱글턴답게 일관된 `equals`/`hashCode`(모든 인스턴스가 곧 자기 자신이므로 참조 동일성과 일치)를 컴파일러가 생성해 준다. 일반 object도 `==`가 사실상 참조 동일성으로 동작하지만(인스턴스가 하나뿐이므로), `data object`는 그 의미를 명시적으로 못 박고 사람이 읽는 표현까지 얻는다. `data object`와 일반 `data class`의 차이의 정본은 [[27 - 데이터 클래스]]에 있으니, 여기서는 "object에도 data가 붙어 toString/equals를 다듬는다"는 교차점만 붙잡는다.

> **명세 정밀**: `data object`가 자동 생성하는 것은 `toString`(이름 반환)과, `equals`/`hashCode`(싱글턴 시맨틱에 맞게)다. `copy`나 `componentN`은 생성하지 않는다 — object에는 복사할 프로퍼티도, 구조 분해할 위치 인자도 없기 때문이다. 이 점이 주 생성자 프로퍼티를 근거로 6개 멤버를 만드는 `data class`와의 결정적 차이다.

### 3.4 object의 정체성과 `===`

object 인스턴스는 하나뿐이므로, 같은 object를 두 번 참조하면 언제나 같은 객체다. 이것은 [[10 - 불리언과 동등성과 동일성]]의 `===`(참조 동일성)로 확인된다.

```kotlin
object Sun
fun main() {
    val a = Sun
    val b = Sun
    println(a === b)     // => true  — 언제나 동일 인스턴스
    println(a == b)      // => true  — 기본 equals는 참조 동일성이므로 일치
}
```

이 성질은 object를 "정체성으로 비교하는 토큰"으로 쓰기에 알맞다. 예컨대 맵의 특수한 마커 키, 알고리즘의 종료 신호(sentinel), 두 코드 경로가 "같은 전역 상태를 보는가"를 참조 동일성으로 판단하는 자리 등이다. 인스턴스가 하나뿐이라는 보장이 `===`의 의미를 확실하게 만들어 준다 — 박싱 캐시 범위에 따라 `===`가 예측 불가해지는 원시 값(M48, [[10 - 불리언과 동등성과 동일성]])과 대조적이다.

---

## 4. object 표현식 — 같은 키워드, 정반대의 수명

### 4.1 선언과 표현식의 갈림길

`object` 키워드는 두 개의 완전히 다른 물건을 만든다. 지금까지 본 **object 선언**(`object Name { ... }`)은 이름 있는 전역 싱글턴이다. 반면 **object 표현식**(`object : Super { ... }` 또는 `object { ... }`)은 그 자리에서 즉석으로 만드는 익명 객체이며, 코드가 그 자리를 지날 때마다 새 인스턴스가 생긴다.

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

같은 키워드가 유일 싱글턴과 일회용 익명 객체를 동시에 표현한다는 점은 처음엔 혼란스럽지만, 규칙은 단순하다. **이름이 있으면 선언(싱글턴), `:` 또는 `{`로 곧장 값 자리에 놓이면 표현식(익명).** 익명 객체의 캡처·다중 슈퍼타입·중첩과 이너의 대비 같은 세부는 [[31 - 중첩 클래스와 이너 클래스와 익명 객체]]가 정본으로 다루므로, 여기서는 싱글턴과의 수명 대비에 필요한 만큼만 본다.

### 4.2 익명 객체는 여러 슈퍼타입을 가질 수 있다

object 표현식은 슈퍼타입을 여럿 나열할 수 있고, 슈퍼타입 없이 순수한 임시 프로퍼티 묶음을 만들 수도 있다.

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

슈퍼타입 없는 익명 객체(`object { val x = 10 }`)의 미묘함은 그 **타입**에 있다. 이 익명 타입은 이름이 없으므로, 그 고유 멤버(`x`, `y`)에 접근하려면 그 익명 타입이 정적으로 보이는 스코프 안이어야 한다. Kotlin 규칙상, object 표현식을 반환하는 함수나 프로퍼티의 타입이 `public`/`protected`로 노출되면 익명 타입이 아니라 그 슈퍼타입(슈퍼타입이 없으면 `Any`)으로 좁혀진다.

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

이 규칙 때문에 "슈퍼타입 없는 익명 객체의 고유 멤버"는 지역·private 문맥에서만 쓸모가 있다. 공개 API 경계를 넘겨야 한다면 이름 있는 타입(인터페이스나 클래스)을 슈퍼타입으로 두어야 한다. 이 좁힘 규칙의 정본 역시 [[31 - 중첩 클래스와 이너 클래스와 익명 객체]]다.

### 4.3 두 object의 대비 정리

| 축 | object 선언 | object 표현식 |
|---|---|---|
| 문법 | `object Name { }` | `object : Super { }` / `object { }` |
| 이름 | 있음 | 없음(익명) |
| 인스턴스 수 | 전 프로그램 하나(싱글턴) | 실행할 때마다 새로 |
| 초기화 시점 | 첫 접근에 지연(JVM 백엔드) | 표현식 평가 시 즉시 |
| 위치 | 최상위·중첩(지역 불가) | 어디서든(주로 지역) |
| 캡처 | 없음(전역이라 캡처할 스코프 없음) | 둘러싼 스코프의 변수 캡처 가능 |
| 대표 용도 | 전역 싱글턴, sealed의 상태 없는 변형 | 일회용 리스너·콜백·SAM 변환 |

캡처의 차이가 특히 본질적이다. 익명 객체는 [[14 - 클로저와 캡처]]에서 다루는 것처럼 둘러싼 함수의 지역 변수를 캡처해 자신 안에 붙든다. 반면 object 선언은 전역이라 "둘러싼 스코프"가 없으니 캡처할 것도 없다. 이 차이가 지역 object 선언을 금지하는 근본 이유(1.4절)와 맞물린다 — 캡처가 필요한 상황은 정의상 지역이고, 지역이면 표현식이어야 한다.

---

## 5. companion object — 클래스 안의 동반 인스턴스

### 5.1 문제: Kotlin에는 static이 없다

Kotlin에는 `static` 키워드가 없다. Java 개발자가 처음 만나는 벽이 이것이다. 클래스에 매인 팩토리 메서드, 클래스 수준 상수, 인스턴스 없이 부르는 유틸리티를 어디에 둘 것인가? Kotlin의 답은 두 갈래다. 특정 클래스에 개념적으로 매이지 않는 것은 파일 최상위(top-level) 함수·프로퍼티로 두고([[04 - 파일과 패키지와 선언 - 프로그램의 골격]]), 특정 클래스에 매여야 하는 것 — 특히 그 클래스의 `private` 생성자에 접근해야 하는 팩토리 — 은 `companion object`에 둔다.

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

`User.create(...)`는 Java의 `static` 팩토리 메서드처럼 보이고 쓰인다. 이 겉모습이 M25의 오해를 낳는다.

### 5.2 companion 멤버는 static이 아니다 (M25)

`companion object`는 이름 그대로 클래스의 "동반 객체"다 — 클래스마다 딱 하나 존재하는 **진짜 object 인스턴스**이고, 그 안의 멤버는 그 인스턴스의 멤버다. `User.create`가 static처럼 보이는 것은 Kotlin이 "클래스 이름으로 동반 객체 멤버에 접근"하는 문법 설탕을 제공하기 때문이지, `create`가 static 메서드여서가 아니다.

JVM 백엔드에서 companion object가 컴파일되는 형태를 개념적으로 그려 보면 진실이 드러난다.

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

읽어야 할 것은 두 가지다. 첫째, `create`는 `Companion`이라는 중첩 클래스의 **인스턴스 메서드**다. `static`이 아니다. 둘째, 바깥 클래스 `User`에는 `Companion`이라는 이름의 **static 필드**가 하나 생기고, 그것이 유일한 동반 객체 인스턴스를 담는다. Kotlin에서 `User.create(...)`라고 쓴 것은 컴파일 후 `User.Companion.create(...)`로 번역된다.

이 사실은 Java에서 이 코드를 부를 때 적나라하게 드러난다. Java에는 Kotlin의 문법 설탕이 없으므로, `User.create(...)`가 아니라 명시적으로 동반 객체 인스턴스를 거쳐야 한다.

```java
// Java에서 Kotlin의 companion object를 호출할 때
User u = User.Companion.create("밥");   // OK — Companion 인스턴스를 명시
// User u = User.create("밥");          // 컴파일 에러: create는 User의 static이 아니다
```

> **흔한 오해 (M25)**: "companion object의 멤버는 정적(static) 멤버다." — 아니다. companion 멤버는 클래스마다 하나 존재하는 `Companion` 인스턴스에 대한 인스턴스 멤버다. Kotlin 소스에서 `클래스명.멤버`로 접근되는 것은 컴파일러가 제공하는 설탕일 뿐이고, JVM 바이트코드 수준에서는 `클래스명.Companion.멤버` 즉 인스턴스 메서드 호출이다. 진짜 `static` 멤버로 만들려면 `@JvmStatic`(6절)이 필요하다.

### 5.3 클래스당 하나, 이름은 선택

한 클래스에는 companion object를 **하나만** 둘 수 있다. 이름을 주지 않으면 컴파일러가 `Companion`이라는 기본 이름을 부여하고, 원한다면 이름을 명시할 수 있다.

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

이름을 붙이면 Java 상호운용에서 `Circle.Factory.of(...)`처럼 그 이름으로 접근하게 되고, 확장 함수를 붙일 때도 이름이 쓸모 있다(7.3절). 이름 없는 companion은 자동으로 `Companion`이라는 이름을 얻으므로, Java에서는 언제나 `Circle.Companion.of(...)` 형태가 존재한다. companion은 인스턴스를 통해서가 아니라 **클래스 이름을 통해서만** 접근한다는 점도 static과의 미묘한 차이다.

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

Java에서는 인스턴스 참조로도 static 멤버에 접근할 수 있지만(권장되지 않는 관례), Kotlin의 companion은 그것을 허용하지 않는다. 이것은 오히려 명료함이다 — companion 멤버는 인스턴스가 아니라 클래스(타입)에 매인 것임을 문법이 강제한다.

### 5.4 companion object의 초기화 시점

companion object 역시 하나의 object이므로 지연·단일 초기화된다. 다만 그 시점은 바깥 클래스와 얽혀 있다. JVM 백엔드에서 동반 객체 인스턴스는 바깥 클래스의 정적 초기화 블록에서 생성되므로, **바깥 클래스가 초기화될 때 함께 초기화**된다. 바깥 클래스는 그 클래스의 인스턴스를 만들거나, 그 클래스의 다른 static 멤버(companion 멤버 포함)에 접근할 때 초기화된다.

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

단, `const val`(6.3절)은 컴파일 시점 상수라 접근해도 클래스 초기화를 유발하지 않는다 — 상수 값이 사용처에 인라인되기 때문이다. 이 미묘함은 [[11 - 변수와 초기화 - val과 const와 lateinit]]의 `const` 논의와 이어진다.

---

## 6. companion의 JVM 표현과 @Jvm 애노테이션

### 6.1 진짜 static을 원할 때: @JvmStatic

5.2절에서 봤듯 companion 멤버는 기본적으로 `Companion` 인스턴스의 메서드다. Java에서 `User.Companion.create(...)`로 부르는 것은 번거롭다. 순수 Kotlin 코드끼리라면 이 구분이 보이지 않으므로 신경 쓸 필요 없지만, **Java에서 진짜 static처럼 부르고 싶다면** `@JvmStatic`을 붙인다. 그러면 컴파일러가 바깥 클래스에 진짜 static 전달(forwarding) 메서드를 하나 더 생성한다.

```kotlin
class User private constructor(val name: String) {
    companion object {
        @JvmStatic
        fun create(name: String): User = User(name)
    }
}
```

이제 JVM 바이트코드에는 두 개의 진입점이 생긴다. `User.Companion.create(...)`(원래의 인스턴스 메서드)와, 그것을 위임 호출하는 진짜 static `User.create(...)`. Java에서는 이제 자연스럽게 쓸 수 있다.

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

핵심은 `@JvmStatic`이 **Java 상호운용을 위한 표현 계층의 장치**라는 것이다. Kotlin 코드 안에서는 `@JvmStatic이 있든 없든` `User.create(...)`로 똑같이 부른다. 이 애노테이션의 전체 그림은 [[45 - Java 상호운용]]에 있다.

### 6.2 필드로서의 상태: @JvmField

companion object의 프로퍼티는 기본적으로 접근자(getter/setter)를 통한다. 그런데 JVM 백엔드에서 companion 프로퍼티의 **백킹 필드가 어디에 놓이는지**는 미묘하다 — 백킹 필드는 대개 바깥 클래스의 static 필드로 생성되고, 접근자는 `Companion` 인스턴스의 메서드다. Java에서 그 값을 필드로 직접 읽으려면 `@JvmField`를 붙여 접근자를 없애고 공개 static 필드로 노출한다.

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

`@JvmField`는 값이 불변인 단순 상수(원시가 아닌 값, 예컨대 미리 만든 객체 인스턴스)를 Java에 필드로 깔끔히 노출할 때 유용하다. 다만 접근자가 사라지므로 나중에 계산 로직을 끼워 넣을 유연성을 잃는다. 프로퍼티가 접근자 추상이라는 큰 그림은 [[22 - 프로퍼티와 백킹 필드]]가 소유한다.

### 6.3 컴파일 상수: const val은 어느 클래스로 가는가

companion 안에 `const val`을 두면 이야기가 또 달라진다. `const val`은 컴파일 시점에 값이 확정되는 상수(원시 타입 또는 `String`, 리터럴로 초기화)이며, JVM 백엔드에서 **바깥 클래스의 `public static final` 필드**로 컴파일되고 그 값은 사용처에 인라인된다.

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

`const val`, `@JvmField val`, 그냥 `val`이 각각 어떻게 다른 위치·형태로 컴파일되는지 정리하면 다음과 같다.

| 선언(companion 안) | 백킹 위치(JVM 백엔드) | Java 접근 | 값 인라인 | 클래스 초기화 유발 |
|---|---|---|---|---|
| `const val X = 42` | 바깥 클래스 `public static final` | `C.X` | 예(사용처에 박힘) | 아니오 |
| `@JvmField val X = obj` | 바깥 클래스 `public static` 필드 | `C.X` | 아니오 | 예 |
| `val X = compute()` | 필드 + 접근자(게터는 Companion) | `C.Companion.getX()` | 아니오 | 예 |

`const val`의 "값 인라인"에는 대가가 있다. 상수를 참조하는 다른 모듈이 그 값을 자신의 바이트코드에 박아 넣으므로, 상수 값을 바꾸고 이 클래스만 재컴파일하면 이미 값을 박아 둔 소비자 모듈은 **옛 값을 계속 쓴다**. 값을 바꿀 여지가 있다면 `const`가 아니라 일반 `val`이 안전하다. 이 상수의 컴파일 시맨틱과 M36("const val과 val은 같다"는 오해)의 정본은 [[11 - 변수와 초기화 - val과 const와 lateinit]]에 있다.

### 6.4 상수는 어디에 둘 것인가

`const val`은 최상위(top-level)에도, companion 안에도 둘 수 있다. 선택 기준은 "그 상수가 어떤 타입에 개념적으로 매이는가"다.

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

최상위 `const val`은 파일-클래스(예: `MyFileKt`)의 static 필드로, companion `const val`은 그 클래스의 static 필드로 컴파일된다. 순수 Kotlin에서는 어느 쪽이든 성능 차이가 없으니 순전히 이름공간(namespace) 설계의 문제다 — `HttpClient.DEFAULT_TIMEOUT_MS`가 `DEFAULT_TIMEOUT_MS`보다 소속을 분명히 드러낸다. 반대로, 타입 하나에 상수를 담기 위해서만 companion을 만드는 것은 과할 수 있다(companion object는 클래스 초기화 시 인스턴스가 생기지만, `const val`만 있다면 그 인스턴스가 실제로는 필요 없다 — 값이 인라인되므로). 상수만이라면 최상위가 더 가볍다는 관점도 합리적이다.

---

## 7. companion의 진짜 힘 — 인터페이스·팩토리·확장·연산자

### 7.1 companion object는 인터페이스를 구현한다

companion object는 그냥 object이므로, 3절에서 본 것처럼 **인터페이스를 구현하고 클래스를 상속할 수 있다.** 이것이 "static 멤버 묶음"이라면 절대 불가능한 일이며, companion을 단순한 static 대체품 이상으로 만드는 결정적 능력이다.

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

`Robot.Companion`이 `Factory<Robot>` 타입의 값으로 `buildTwo`에 전달된다. 즉 "클래스에 매인 팩토리"를 일급 값으로 추상화해 다형적으로 다룰 수 있다. 이 패턴은 여러 타입이 공통 팩토리 인터페이스를 구현하게 하여, 타입별 생성 로직을 통일된 방식으로 주입·교체하는 설계를 연다.

### 7.2 팩토리 패턴과 이름 있는 생성

companion의 가장 흔한 실전 용도는 **이름 있는 팩토리 메서드**다. 생성자는 클래스 이름 하나만 쓸 수 있어서, 서로 다른 의미의 생성 방법을 이름으로 구분할 수 없다. companion 팩토리는 이 한계를 넘고, private 생성자와 결합해 "생성 경로를 통제"한다.

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

세 개의 생성 경로 `fromCelsius`/`fromFahrenheit`/`fromKelvin`은 생성자로는 구분 불가능한 의미(같은 `Double` 인자, 다른 해석)를 이름으로 명료히 나눈다. private 생성자는 "반드시 이 팩토리들을 통해서만 만들라"는 계약을 강제한다. companion은 이 팩토리들이 `Temperature`의 private 생성자에 접근할 수 있게 해 주는데, 이는 companion이 그 클래스의 **내부**이기 때문이다.

### 7.3 companion에 확장 함수를 붙이다

companion object에는 확장 함수·프로퍼티를 붙일 수 있다. 이렇게 하면 마치 "클래스에 새로운 정적 팩토리를 나중에 추가"하는 것 같은 효과가 난다. 확장 함수의 정적 디스패치 원리 자체는 [[35 - 확장 함수와 프로퍼티]]가 소유하므로, 여기서는 companion을 수신 타입으로 쓰는 형태에 집중한다.

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

주목할 것은 확장을 붙이려면 companion object가 **존재해야** 한다는 점이다 — 빈 `companion object`라도 선언되어 있어야 `Money.Companion`이라는 확장 수신 타입이 성립한다. 이 기법은 라이브러리 사용자가 라이브러리 타입의 companion에 자신만의 팩토리를 덧붙일 수 있게 하는 확장점(extension point)으로 쓰인다. companion에 이름을 줬다면(`companion object Factory`) 확장은 `fun Money.Factory.xxx()`로 쓴다.

### 7.4 companion과 연산자·invoke

companion object에 `operator fun invoke`를 정의하면, 마치 생성자를 부르듯 클래스 이름을 함수처럼 호출하는 관용구를 만들 수 있다. 연산자 관례 `invoke`의 정본은 [[19 - 연산자 오버로딩과 관례]]이며, 여기서는 companion과 결합하는 모습만 본다.

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

`Logger("APP")`은 얼핏 생성자 호출로 보이지만, 실제로는 companion object의 `invoke` 연산자를 부르는 것이다. 이 기법은 팩토리를 "생성자처럼" 자연스럽게 노출하고 싶을 때 쓰이지만, 진짜 생성자와 혼동을 부를 수 있어 남용하면 코드를 읽기 어렵게 만든다. 표준 라이브러리도 이 패턴을 신중히 쓴다(예: 일부 타입의 팩토리).

### 7.5 companion object는 어디까지 담아야 하나

companion에 무엇이든 넣을 수 있다고 해서 넣어야 하는 것은 아니다. companion object는 다음에 잘 맞는다.

- private 생성자에 접근하는 **팩토리 메서드**(7.2절)
- 그 타입에 개념적으로 매인 **상수**(6.4절)
- 그 타입의 팩토리를 **다형적으로 추상화**하는 인터페이스 구현(7.1절)
- 확장으로 나중에 팩토리를 덧붙이기 위한 **확장점**(7.3절)

반대로, 특정 클래스와 무관한 순수 유틸리티 함수는 companion이 아니라 **최상위 함수**가 낫다 — 불필요한 클래스 소속을 강요하지 않고, import·이름공간이 깔끔하다. Java의 "모든 것을 클래스 안에" 습관 때문에 유틸리티를 무심코 companion에 몰아넣는 것은 Kotlin다운 설계가 아니다. "이 함수가 정말 이 타입에 매이는가, 아니면 그냥 static이라서 넣는가"를 물어야 한다.

---

## 8. object·companion의 함정과 안티패턴 종합

### 8.1 싱글턴이라는 이름의 전역 가변 상태

`object`가 손쉽게 만드는 것은 싱글턴이지만, 싱글턴은 곧 **전역 상태**가 될 수 있고 전역 가변 상태는 테스트·동시성·추론을 어렵게 하는 고전적 함정이다.

```kotlin
object GlobalCache {
    val store = mutableMapOf<String, Int>()   // 전역 가변 상태 — 위험 신호
}
```

`GlobalCache.store`는 프로그램 어디서든 읽고 쓸 수 있어, 두 스레드가 동시에 수정하면 경쟁 조건에 빠진다(가변 컬렉션은 스레드 안전하지 않다 — [[38 - 컬렉션 - 읽기 전용과 가변]], [[43 - 동시성과 메모리 모델]]). 게다가 테스트 사이에 상태가 새어(leak) 한 테스트가 다른 테스트의 결과를 오염시킨다. **object의 지연·스레드 안전 초기화는 인스턴스 생성에 대한 보장이지, 그 안의 가변 상태에 대한 동기화가 아니다.** 이 구분을 놓치면 "object니까 스레드 안전하겠지"라는 착각에 빠진다. object 안에 가변 상태를 둔다면 그 상태 자체의 동기화는 별도로 책임져야 한다.

### 8.2 무거운·실패 가능한 초기화를 object에 담기

2.4절에서 봤듯 object 초기화는 첫 접근 스레드에서 동기적으로 일어난다. 그런데 초기화가 예외를 던지면 어떻게 될까? JVM 백엔드에서 `<clinit>`이 예외를 던지면 그 클래스는 `ExceptionInInitializerError`로 표시되고, 이후 그 클래스에 접근하려는 모든 시도가 `NoClassDefFoundError`로 실패한다 — 회복 불가능한 상태다.

```kotlin
object Bootstrap {
    val config = loadConfigOrThrow()   // 여기서 예외가 나면 Bootstrap은 영구히 사용 불가
}
```

즉 object는 "실패하지 않는 가벼운 초기화"를 전제로 한다. 초기화가 외부 자원(파일·네트워크)에 의존해 실패할 수 있다면, 그 자원 획득을 object 초기화가 아니라 명시적 함수·팩토리로 옮겨 실패를 정상 흐름([[41 - 예외와 Nothing]]의 `Result` 등)으로 다루는 것이 옳다.

### 8.3 companion의 상태 공유 오해

companion object의 프로퍼티는 클래스의 **모든 인스턴스가 공유하는** 상태다(인스턴스별 상태가 아니라 클래스 수준 상태다 — Java의 static 필드와 같은 공유 성질). 이것을 인스턴스별로 착각하면 버그가 된다.

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

여기서 `total`이 공유 상태라는 것은 의도된 설계지만(인스턴스에 순번을 매기려는 것), 무심코 companion에 둔 가변 프로퍼티는 "왜 여러 인스턴스가 같은 값을 보나"라는 혼란을 부른다. companion 상태는 언제나 클래스 전역이다 — 이것을 명확히 의식하고 써야 한다.

### 8.4 object 대신 top-level, companion 대신 top-level

Kotlin다운 설계에서 반복되는 질문은 "이건 object/companion이어야 하는가, 아니면 최상위면 되는가"다. 순수 함수의 묶음(수학 유틸리티 등)을 담기 위해 만든 object는 대개 불필요하다 — 최상위 함수면 충분하고 더 가볍다.

```kotlin
// 안티패턴: 순수 함수를 담기 위한 object (Java의 유틸 클래스 습관)
object MathUtils {
    fun square(x: Int) = x * x
}

// Kotlin다운 방식: 최상위 함수
fun square(x: Int) = x * x
```

`MathUtils.square(3)`은 `MathUtils`라는 싱글턴 인스턴스를 초기화하고 그 인스턴스 메서드를 부르는 반면, 최상위 `square(3)`은 파일-클래스의 static 메서드 하나를 부를 뿐이다(JVM 백엔드). 후자가 더 단순하고, import도 함수 단위로 깔끔하다. object는 "상태를 담거나, 슈퍼타입을 구현하거나, 인터페이스로 넘겨야 할 때" 값을 하며, 그런 이유 없이 순수 함수만 담는다면 최상위가 낫다. 같은 논리가 companion에도 적용된다(7.5절) — 클래스에 매이지 않는 유틸리티라면 최상위로.

### 8.5 object vs enum vs sealed의 갈림

"상태 없는 유일 값"을 표현하는 세 도구가 겹쳐 보인다. 결정 감각을 정리하면 이렇다.

```text
값이 하나뿐이고 다형성/상속이 필요? ─► object (또는 sealed 안의 object)
값이 유한 개의 열거이고 서로 순번·이름이 필요? ─► enum ([[29 - enum 클래스]])
값이 유한 개이되 각각 다른 데이터를 담아야? ─► sealed + data class/object ([[30 - sealed 클래스와 대수적 데이터 타입]])
```

`object`는 "단 하나"에, `enum`은 "정해진 여럿"에, `sealed`는 "정해진 여럿이되 각자 모양이 다를 수 있음"에 대응한다. 이 세 도구의 정본은 각각 이 장, [[29 - enum 클래스]], [[30 - sealed 클래스와 대수적 데이터 타입]]이며, 실전에서는 sealed 계층 안에 상태 없는 변형을 `object`로, 상태 있는 변형을 `data class`로 섞는 조합(3.2절)이 가장 자주 등장한다.

---

지금까지의 여정을 한 줄기로 회수하자. 도입에서 던진 질문 — object는 정적 클래스인가 진짜 인스턴스인가 — 의 답은 이제 분명하다. object는 하나의 클래스와 하나의 `INSTANCE` 필드로 컴파일되는 **진짜 인스턴스**이며, 그 유일성과 지연·스레드 안전 초기화는 JVM 백엔드에서 클래스 초기화 시맨틱에 얹혀 공짜로 얻어진다(M24). companion object는 "정적 멤버 자리"처럼 보이지만 실은 클래스마다 하나 존재하는 동반 객체 인스턴스이고, `클래스명.멤버`는 Kotlin이 제공하는 설탕일 뿐 바이트코드로는 `클래스명.Companion.멤버`라는 인스턴스 호출이다(M25). 진짜 static이 필요하면 `@JvmStatic`, 필드 노출은 `@JvmField`, 컴파일 상수는 `const val`이 각각 다른 위치·형태로 그 뜻을 실현한다. 그리고 object가 인스턴스이기에 인터페이스를 구현하고 상속하며 다형적 값으로 넘어갈 수 있다는 사실이, companion을 단순한 static 대체가 아니라 팩토리·전략·확장점의 무대로 끌어올린다. 같은 `object` 키워드가 전역 싱글턴(선언)과 일회용 익명 객체(표현식)라는 정반대 수명의 두 물건을 만든다는 이중성까지 잡으면, object와 companion의 지형은 온전히 손에 들어온다. 남은 것은 이 인스턴스 하나가 어떤 함정 — 전역 가변 상태, 실패 가능한 초기화, 순환 참조 — 을 품는지를 늘 의식하는 규율이다.

## 핵심 요약

- **`object`는 정적 클래스가 아니라 진짜 인스턴스다 (M24).** JVM 백엔드에서 하나의 클래스와 하나의 `INSTANCE` static 필드로 컴파일되며, 멤버는 static이 아니라 그 `INSTANCE`에 대한 인스턴스 메서드다. `Registry.put(...)`은 `Registry.INSTANCE.put(...)`으로 번역된다.
- **object의 유일성·지연·스레드 안전은 JVM 백엔드에서 클래스 초기화 시맨틱에 근거한다.** JVM은 클래스 초기화를 첫 능동 사용까지 미루고 클래스별 락으로 한 번만 수행하므로, 개발자가 이중 검사 락을 손으로 짤 필요가 없다. 다른 백엔드(Native/JS/Wasm)에서는 초기화 시점 보장이 런타임 구현에 따라 다르다.
- **object의 "유일성"은 하나의 클래스로더 안에서만 성립한다.** 서로 다른 클래스로더가 같은 object를 로드하면 별개의 인스턴스가 생긴다 — 이는 JVM 플랫폼의 성질이며 Java의 static과 동일한 조건이다.
- **object는 슈퍼타입을 가질 수 있다.** 클래스를 상속하고 인터페이스를 구현하며, `is`/`as`의 대상이 되고 값으로 전달된다. 이것이 "진짜 인스턴스"의 결정적 증거이자, sealed 계층의 상태 없는 변형·널 객체 패턴·다형적 싱글턴의 토대다.
- **같은 `object` 키워드가 정반대 수명의 두 물건을 만든다.** 이름 있는 **선언**은 전역 싱글턴(지역 불가), 값 자리의 **표현식**은 실행할 때마다 새로 생기는 익명 객체(캡처 가능)다.
- **`companion object`의 멤버는 static이 아니다 (M25).** 클래스마다 하나 존재하는 `Companion` 인스턴스의 멤버이며, `클래스명.멤버` 접근은 Kotlin의 문법 설탕이다. Java에서는 `클래스명.Companion.멤버`로 접근해야 하고, 진짜 static을 원하면 `@JvmStatic`을 붙여 바깥 클래스에 전달 메서드를 생성해야 한다.
- **`const val`·`@JvmField val`·일반 `val`은 companion 안에서 서로 다르게 컴파일된다.** `const val`은 바깥 클래스의 `static final` 필드로 값이 사용처에 인라인되고, `@JvmField`는 접근자 없는 static 필드로, 일반 `val`은 필드+접근자(게터는 Companion 인스턴스)로 컴파일된다 (M36은 [[11 - 변수와 초기화 - val과 const와 lateinit]]).
- **companion은 인터페이스를 구현하고 확장·연산자를 받을 수 있어 단순 static 대체를 넘어선다.** private 생성자에 접근하는 이름 있는 팩토리, `Factory<T>` 같은 인터페이스의 다형적 구현, 나중에 팩토리를 덧붙이는 확장점(빈 companion이라도 선언 필요), `invoke`로 생성자처럼 보이는 호출을 연다.
- **object의 지연·스레드 안전 보장은 인스턴스 생성에 대한 것이지 내부 가변 상태의 동기화가 아니다.** object 안의 가변 컬렉션·프로퍼티는 여전히 경쟁 조건에 노출되며, 그 동기화는 별도로 책임져야 한다.
- **object 초기화가 예외를 던지면 그 object는 영구히 사용 불가가 된다** (JVM 백엔드의 `ExceptionInInitializerError`). object는 가볍고 실패하지 않는 초기화를 전제로 하며, 실패 가능한 자원 획득은 명시적 팩토리로 옮겨야 한다.
- **`data object`(2.x)는 이름을 반환하는 `toString`과 싱글턴 시맨틱의 `equals`/`hashCode`를 생성한다** — `copy`·`componentN`은 없다. sealed 계층의 상태 없는 변형을 로그·디버깅 친화적으로 만든다.
- **순수 함수·타입 무관 유틸리티·상수는 object/companion보다 최상위 선언이 Kotlin답다.** object/companion은 상태를 담거나, 슈퍼타입을 구현하거나, 클래스의 private에 접근하거나, 클래스에 개념적으로 매일 때 값을 한다.

## 연결 노트

- [[21 - 클래스와 생성자]] — object·companion의 `init` 블록과 프로퍼티 초기화 순서는 일반 클래스의 규칙을 그대로 따른다. 이 장은 "인스턴스가 하나뿐인 클래스"라는 특수화다.
- [[27 - 데이터 클래스]] — `data object`가 두 장의 교차점이다. data가 생성하는 멤버(equals/hashCode/toString) 중 object가 취하는 것과 버리는 것(copy/componentN 없음)의 차이를 여기서 본다.
- [[30 - sealed 클래스와 대수적 데이터 타입]] — 상태 없는 변형을 `object`로, 상태 있는 변형을 `data class`로 두는 조합의 정본. object가 슈퍼타입을 가질 수 있다는 이 장의 사실이 그곳에서 완전한 when을 떠받친다.
- [[29 - enum 클래스]] — "정해진 여럿"의 열거. "단 하나"의 object와 대조하며 세 도구(object/enum/sealed)의 선택 감각을 세운다.
- [[31 - 중첩 클래스와 이너 클래스와 익명 객체]] — object 표현식(익명 객체)의 캡처·다중 슈퍼타입·타입 좁힘 규칙의 정본. 이 장은 싱글턴과의 수명 대비에 필요한 만큼만 다뤘다.
- [[35 - 확장 함수와 프로퍼티]] — companion object에 확장을 붙이는 기법의 배경. 확장의 정적 디스패치 원리(M14)가 그곳에 있다.
- [[45 - Java 상호운용]] — `@JvmStatic`·`@JvmField`·`@JvmName`의 전체 그림. companion 멤버가 Java에서 왜 `Companion`을 거치는지, 어떻게 진짜 static으로 노출하는지의 정본.
- [[11 - 변수와 초기화 - val과 const와 lateinit]] — `const val`이 컴파일 상수로서 어느 필드로 가고 왜 값이 인라인되는지(M36). 상수를 companion에 둘지 최상위에 둘지의 판단이 이어진다.
- [[10 - 불리언과 동등성과 동일성]] — object가 하나뿐이라 `===`(참조 동일성)의 의미가 확실해지는 것과, 박싱 캐시로 `===`가 흔들리는 원시 값(M48)의 대조.
- [[43 - 동시성과 메모리 모델]] — object의 스레드 안전 초기화가 기대는 클래스 초기화 락과, object 내부 가변 상태의 경쟁 조건을 다루는 정본.
