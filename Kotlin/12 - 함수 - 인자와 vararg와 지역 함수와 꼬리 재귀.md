---
title: 함수 - 인자와 vararg와 지역 함수와 꼬리 재귀
date: 2026-07-13
tags: [kotlin, function, vararg, tailrec, overload, 학습노트]
---

**이 장이 답하는 질문:**

- Kotlin의 함수는 JVM 백엔드에서 정확히 무엇으로 컴파일되는가? 파일에 자유롭게 놓은 최상위 함수는 어디로 사라지는가?
- 함수 파라미터가 `val`이라는 말은 무슨 뜻인가? Java처럼 파라미터에 재대입할 수 없는 이유는?
- `= expr`로 쓴 단일 표현식 함수는 왜 반환 타입을 생략할 수 있고, 블록 본문 `{ ... }`는 왜 못 하는가? 여기서 어떤 함정이 생기는가?
- 기본 인자(default argument)는 컴파일러가 어떤 마법을 부려 구현하는가? 왜 Java에서 부를 때는 `@JvmOverloads`가 필요한가?
- 이름 붙은 인자가 있으면 오버로딩이 왜 대부분 필요 없어지는가? 그런데도 Java 메서드엔 왜 이름으로 못 부르는가?
- `vararg`는 진짜 가변 인자인가, 아니면 배열의 위장인가? 스프레드 `*`는 무엇을 복사하는가?
- 오버로딩 해소(overload resolution)에서 기본 인자·vararg·정확한 시그니처는 어떤 순서로 경쟁하는가?
- 지역 함수는 왜 존재하는가, 그리고 `tailrec`은 스택을 어떻게 없애며 어디서 조용히 실패하는가?

---

[[11 - 변수와 초기화 - val과 const와 lateinit]]에서 우리는 값에 이름을 붙이는 일 — `val`, `var`, `const val`, `lateinit` — 을 기계 수준에서 해부했다. 변수는 값을 담는 그릇이었다. 이번 장은 그 값들을 *받아서 다른 값을 만들어내는 기계*, 즉 함수(function)를 같은 깊이로 파고든다. Kotlin에서 함수는 일급 시민(first-class citizen)이며, 언어의 가장 기본적인 구성 단위다. 클래스보다 먼저 함수가 온다. 파일 최상단에 클래스 없이 `fun main()`만 덜렁 놓을 수 있다는 사실 자체가 Kotlin의 설계 철학([[01 - 코틀린이란 무엇인가 - 설계 철학과 세 개의 타깃]])을 요약한다.

이 장이 다루는 것은 함수의 *선언과 호출* 그 자체다. 파라미터의 불변성, 기본 인자와 이름 붙은 인자가 만드는 유연한 호출 규약, `vararg`의 배열 정체, 단일 표현식 함수의 타입 추론, 지역 함수와 그것이 여는 클로저의 문, 그리고 재귀를 반복문으로 바꾸는 `tailrec`까지다. 반대로 이 장이 *다루지 않는* 것도 분명히 못박아 두자. 함수를 값으로 다루는 이야기 — 함수 타입 `(Int) -> Int`, 람다, 함수 참조 `::foo` — 는 [[13 - 함수 타입과 람다와 함수 참조]]가 소유한다. 함수가 바깥 변수를 붙잡는 캡처(capture)의 메모리·수명 문제는 [[14 - 클로저와 캡처]]로 미룬다. `inline`, `noinline`, `crossinline`, `reified`가 만드는 인라이닝 의미론은 [[15 - 인라인 함수와 reified]]가, 수신 객체에 붙는 확장 함수는 [[35 - 확장 함수와 프로퍼티]]가 가져간다. 이 장은 "평범한 함수"의 뼈와 살에 집중한다.

논증의 궤적은 이렇다. 먼저 함수가 무엇으로 컴파일되는지 그 물성을 확인하고(1절), 파라미터가 왜 불변인지(2절), 반환 타입 추론의 경계는 어디인지(3절)를 못박는다. 그다음 호출 규약의 세 기둥 — 기본 인자(4절), 이름 붙은 인자(5절), vararg(6절) — 를 컴파일된 형태까지 파고들고, 이 셋이 얽혀 만드는 오버로딩 해소 규칙(7절)을 정리한다. 마지막으로 함수 안의 함수, 즉 지역 함수(8절)와 재귀의 특수형인 꼬리 재귀(9절)로 마무리한다. 각 절은 "왜 이렇게 설계됐나 / 내부에서 무슨 일이 일어나나 / 어디서 사람들이 틀리나"를 끝까지 판다.

---

## 1. 함수의 물성 — 무엇으로 컴파일되는가

### 1.1 `fun`이라는 선언과 그 골격

Kotlin 함수의 기본 문법은 다섯 조각으로 이루어진다: `fun` 키워드, 이름, 괄호 안의 파라미터 목록, 콜론 뒤의 반환 타입, 그리고 본문이다.

```kotlin
fun area(width: Int, height: Int): Int {
    return width * height
}
```

파라미터는 `이름: 타입` 순서로 쓴다. Java의 `int width`와 정반대다. 이 "이름 먼저, 타입 나중" 순서는 우연이 아니라 Kotlin 전체를 관통하는 규칙이다. `val x: Int`, `fun f(): T`, 람다 `{ x: Int -> ... }` 모두 같은 방향이다. 타입은 생략 가능한 부가정보라는 사고방식(타입 추론이 있으니까)이 문법 순서에 각인돼 있다. 반환 타입도 콜론 뒤에 오며, 뒤에서 보듯 단일 표현식 함수에서는 생략된다.

함수를 정의할 수 있는 자리는 네 곳이다. 파일 최상위(top-level), 클래스·객체의 멤버(member), 다른 함수 안(local), 그리고 어떤 타입에 붙는 확장(extension). 이 장은 앞의 세 가지를 다룬다. 최상위 함수는 [[04 - 파일과 패키지와 선언 - 프로그램의 골격]]가 보여줬듯 클래스를 강제하지 않는 Kotlin의 특징이고, 확장 함수는 [[35 - 확장 함수와 프로퍼티]]의 주제다.

### 1.2 JVM 백엔드에서 함수는 정적 메서드가 된다

Kotlin은 멀티타깃 언어이므로 "함수가 무엇이 되는가"는 백엔드마다 다르다. 그러나 가장 많이 쓰이는 JVM 백엔드에서 최상위 함수는 클래스의 정적(static) 메서드로 컴파일된다. JVM 바이트코드에는 클래스 밖에 떠 있는 함수라는 개념이 없기 때문에, 컴파일러는 파일 이름을 기반으로 한 합성 클래스를 하나 만든다. `Geometry.kt`에 들어 있는 최상위 함수들은 `GeometryKt`라는 클래스의 `public static` 메서드가 된다.

```text
Geometry.kt
   fun area(w: Int, h: Int): Int { ... }
   fun perimeter(w: Int, h: Int): Int { ... }
        │  컴파일 (JVM 백엔드)
        ▼
public final class GeometryKt {
   public static int area(int w, int h) { ... }
   public static int perimeter(int w, int h) { ... }
}
```

이 이름은 `@file:JvmName("Geometry")` 애노테이션으로 바꿀 수 있고, Java에서는 `GeometryKt.area(3, 4)`처럼 호출한다. 여기서 눈여겨볼 점은 파라미터 타입이다. 소스의 `Int`가 바이트코드에서는 원시 타입 `int`로 내려간다. 이것이 M4의 "Kotlin엔 원시 타입이 없다"는 오해를 깨는 지점이다 — 소스 레벨에는 원시 타입이 없지만 JVM 백엔드는 가능한 곳에서 `int`, `long`, `double`로 컴파일한다. 자세한 것은 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]가 소유한다. 컴파일 파이프라인 전반은 [[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]]를 보라.

> **명세 정밀**: "정적 메서드로 컴파일된다"는 JVM 백엔드에 한정된 사실이다. Kotlin/Native에서는 LLVM IR을 거친 네이티브 함수가, Kotlin/JS에서는 JavaScript 함수가, Kotlin/Wasm에서는 Wasm 함수가 된다. 이 장의 바이트코드 서술은 모두 "JVM 백엔드에서"라는 조건이 붙는다. 다만 의미론(파라미터 불변, 기본 인자, vararg의 논리적 동작)은 모든 타깃에서 동일하다.

### 1.3 표현식 본문과 블록 본문

함수 본문은 두 형태다. 중괄호로 감싼 블록 본문(block body)과 등호로 이은 표현식 본문(expression body)이다.

```kotlin
// 블록 본문 — return으로 값을 돌려준다
fun square(x: Int): Int {
    return x * x
}

// 표현식 본문 — 함수 = 하나의 표현식
fun squareExpr(x: Int): Int = x * x
```

이 둘은 같은 바이트코드로 컴파일된다. 표현식 본문은 단지 "이 함수는 하나의 표현식을 계산해 그 값을 반환한다"를 간결하게 쓰는 문법 설탕이다. Kotlin에서는 `if`, `when`, `try`조차 표현식이므로([[17 - 표현식으로서의 제어 흐름 - if와 when]]) 상당히 복잡한 함수도 표현식 본문으로 쓸 수 있다.

```kotlin
fun classify(n: Int): String = when {
    n < 0 -> "음수"
    n == 0 -> "영"
    else -> "양수"
}
```

두 본문 형태의 결정적 차이는 반환 타입 추론에 있다. 표현식 본문은 반환 타입을 생략할 수 있지만 블록 본문은 (Unit이 아닌 한) 못 한다. 이 차이의 논리와 함정은 3절에서 정면으로 다룬다.

---

## 2. 파라미터는 불변이다 — `val`의 그림자

### 2.1 재대입할 수 없는 파라미터

Kotlin의 함수 파라미터는 암묵적으로 `val`이다. 즉 함수 본문 안에서 파라미터에 새 값을 대입할 수 없다.

```kotlin
fun increment(count: Int): Int {
    count = count + 1   // 컴파일 에러: val cannot be reassigned
    return count
}
```

이것은 Java 개발자가 가장 먼저 부딪히는 벽이다. Java에서는 파라미터가 기본적으로 가변이라 `count = count + 1`이 합법이다(관용적으로 권장되지 않을 뿐이다). Kotlin은 아예 언어 차원에서 막는다.

```java
// Java — 파라미터 재대입이 합법
static int increment(int count) {
    count = count + 1;   // OK
    return count;
}
```

의도는 명확하다. 파라미터 재대입은 거의 항상 버그의 온상이다. 원래 넘어온 인자값을 잃어버리고, "이 변수는 호출자가 준 값인가 내가 바꾼 값인가"를 헷갈리게 만든다. 값을 바꾸고 싶으면 새 지역 변수(`var`)를 명시적으로 선언하라는 것이 Kotlin의 입장이다.

```kotlin
fun increment(count: Int): Int {
    var c = count   // 의도를 드러내는 새 변수
    c += 1
    return c
}
```

### 2.2 불변은 "재대입 불가"이지 "객체 불변"이 아니다

여기서 M3의 오해가 파라미터에도 그대로 재현된다. 파라미터가 `val`이라는 것은 *그 이름에 다른 값을 다시 붙일 수 없다*는 뜻이지, 그 이름이 가리키는 객체가 불변이라는 뜻이 아니다. 참조 대상이 가변 객체라면 그 내부 상태는 얼마든지 바꿀 수 있다.

```kotlin
fun fill(bucket: MutableList<Int>) {
    bucket = mutableListOf()   // 컴파일 에러: 파라미터 재대입 불가
    bucket.add(1)              // 완전히 합법 — 객체 내부 변경
    bucket.add(2)
}
```

`bucket`이라는 참조 자체는 못 바꾸지만, `bucket`이 가리키는 리스트에 원소를 추가하는 것은 아무 문제가 없다. 이 구별은 [[11 - 변수와 초기화 - val과 const와 lateinit]]가 소유한 오개념 M3의 핵심이며, 컬렉션의 "읽기 전용 ≠ 불변" 문제([[38 - 컬렉션 - 읽기 전용과 가변]])와도 연결된다. 파라미터로 가변 컬렉션을 받으면 함수는 호출자의 데이터를 조용히 변형시킬 수 있다 — 이것이 방어적 복사(defensive copy)를 논하는 이유다.

> **흔한 오해**: "Kotlin 파라미터가 val이니 함수는 부수효과가 없다"는 잘못된 안심이다. `val` 파라미터는 참조의 재대입만 막는다. 함수가 넘겨받은 가변 객체를 변형하거나, 전역 상태를 건드리거나, 콘솔에 출력하는 부수효과는 언어가 전혀 막지 않는다. 순수 함수(pure function)는 규율이지 언어 보장이 아니다.

### 2.3 컴파일된 형태와 `final`

JVM 백엔드에서 파라미터가 `val`이라는 사실은 실질적으로 바이트코드의 로컬 변수 슬롯을 재사용하지 않는다는 컴파일러 규율로 나타난다. 소스 수준에서 재대입을 금지하므로, 컴파일러는 파라미터가 함수 전체에서 한 값만 유지한다고 가정하고 최적화할 수 있다. 이것은 스마트 캐스트([[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]])와도 미묘하게 얽힌다. 파라미터는 `val`이라 재대입될 수 없으므로, 널 검사 후 스마트 캐스트가 안전하게 유지된다.

```kotlin
fun describe(name: String?): String {
    if (name == null) return "익명"
    // 여기서 name은 String으로 스마트 캐스트됨
    // 파라미터는 val이라 중간에 null로 바뀔 수 없음 → 안전
    return "이름 길이: ${name.length}"
}
```

만약 파라미터가 `var`였다면(그리고 어딘가에서 재대입 가능하다면) 컴파일러는 이 스마트 캐스트를 보장할 수 없었을 것이다. 파라미터의 불변성은 단순한 스타일 규칙이 아니라 타입 시스템의 안전성을 떠받치는 기둥이다.

---

## 3. 반환 타입 추론과 그 경계

### 3.1 표현식 본문만 반환 타입을 추론한다

Kotlin은 표현식 본문 함수의 반환 타입을 추론한다. 등호 오른쪽 표현식의 타입이 곧 반환 타입이기 때문이다.

```kotlin
fun double(x: Int) = x * 2          // 반환 타입 Int로 추론
fun greet(name: String) = "안녕, $name"  // 반환 타입 String으로 추론
fun midpoint(a: Double, b: Double) = (a + b) / 2  // Double
```

반면 블록 본문은 반환 타입을 추론하지 않는다. 블록 본문은 여러 개의 `return` 문을 가질 수 있고, 각 return이 다른 타입을 반환할 수도 있어 컴파일러가 단일 타입을 뽑아내기 어렵다. 그래서 블록 본문 함수는 반환 타입을 명시해야 한다. 단, 반환 타입이 `Unit`이면 생략 가능하다.

```kotlin
fun classify(n: Int) {   // 반환 타입 생략 → Unit으로 간주
    if (n > 0) return
    println("0 이하")
}

fun classify2(n: Int): String {  // 명시 필수
    if (n > 0) return "양수"
    return "0 이하"
}
```

만약 `classify2`에서 반환 타입을 생략하면, 컴파일러는 반환 타입을 `Unit`으로 간주하고 `return "양수"` 지점에서 "String을 반환하려는데 함수 반환 타입은 Unit"이라며 에러를 낸다. 블록 본문에서는 반환 타입이 곧 계약이고, 컴파일러는 그 계약을 추론해주지 않는다.

### 3.2 Unit 반환 — void가 아니다

값을 돌려주지 않는 함수는 `Unit`을 반환한다. 이것이 M10 — "Unit은 void다"라는 오해 — 의 현장이다. Unit은 값이 없음을 뜻하는 void가 아니라, 값이 정확히 하나뿐인 실제 타입이다. 그 유일한 값도 `Unit`이라는 이름의 싱글턴 객체다. 정본 해부는 [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]]가 소유하지만, 함수 관점에서 핵심만 짚자.

```kotlin
fun logIt(msg: String): Unit {   // 명시적 Unit
    println(msg)
}

fun logIt2(msg: String) {        // Unit 생략 — 위와 동일
    println(msg)
}

val result: Unit = logIt("hi")   // 반환값을 변수에 담을 수 있다
```

Unit이 진짜 타입이기 때문에 `logIt`의 반환값을 `Unit` 타입 변수에 담을 수 있고, 제네릭 타입 인자로 `Unit`을 넘길 수도 있다. `List<Unit>`도 유효한 타입이다. void는 이런 일을 못 한다. JVM 백엔드에서 Unit 반환 함수는 실제로 바이트코드상 `void` 메서드가 되지만(최적화), 언어 의미론상 Unit은 값이다. 이 미묘한 이중성 — 논리적으로는 값, 물리적으로는 void — 이 M10의 뿌리다.

> **명세 정밀**: JVM 백엔드에서 반환 타입이 `Unit`인 함수는 바이트코드 시그니처가 `void`로 컴파일된다. 그러나 Kotlin 코드에서 그 반환값을 `Unit` 타입 값으로 쓰면, 컴파일러가 `Unit.INSTANCE` 싱글턴을 삽입한다. 즉 물리적 표현(void)과 논리적 타입(Unit 값)이 컴파일러 층에서 매개된다. 함수 타입 `() -> Unit`이 필요한 [[13 - 함수 타입과 람다와 함수 참조]]에서 이 이중성이 다시 중요해진다.

### 3.3 재귀 함수는 반환 타입을 명시해야 한다

표현식 본문이라도 재귀 함수는 반환 타입을 추론할 수 없다. 자기 자신을 부르는 표현식의 타입을 알려면 이미 반환 타입을 알아야 하는 순환이 생기기 때문이다.

```kotlin
fun factorial(n: Int) = if (n <= 1) 1 else n * factorial(n - 1)
// 컴파일 에러: 재귀 함수는 반환 타입을 명시해야 함

fun factorialOk(n: Int): Int =   // 명시하면 OK
    if (n <= 1) 1 else n * factorialOk(n - 1)
```

이것은 언어의 한계가 아니라 타입 추론의 논리적 필연이다. 컴파일러가 `factorial(n-1)`의 타입을 알려면 `factorial`의 반환 타입이 이미 확정돼 있어야 하는데, 바로 그것을 추론하려는 참이다. 순환을 끊으려면 개발자가 반환 타입을 못박아야 한다. 이 규칙은 뒤에서 다룰 `tailrec` 함수에도 그대로 적용된다.

---

## 4. 기본 인자 — 컴파일러의 마법

### 4.1 선언과 의미

Kotlin 파라미터는 기본값(default value)을 가질 수 있다. 호출자가 그 인자를 생략하면 기본값이 쓰인다.

```kotlin
fun connect(
    host: String,
    port: Int = 8080,
    timeout: Int = 3000,
    useTls: Boolean = false
) {
    println("$host:$port, timeout=$timeout, tls=$useTls")
}

connect("example.com")                        // => example.com:8080, timeout=3000, tls=false
connect("example.com", 443, useTls = true)    // => example.com:443, timeout=3000, tls=true
```

이 한 기능이 Java의 "망원경 생성자(telescoping constructor)" 안티패턴을 통째로 없앤다. Java에서는 파라미터 조합마다 오버로드를 여러 개 만들어야 했다 — `connect(host)`, `connect(host, port)`, `connect(host, port, timeout)`. Kotlin은 기본값 하나로 그 모든 조합을 표현한다. $n$개의 선택적 파라미터가 있다면 Java에서 최대 $2^n$개의 오버로드가 필요할 수 있는 것을, Kotlin은 함수 하나로 해결한다.

### 4.2 기본값은 언제 평가되는가

기본값 표현식은 호출 시점에, 그 인자가 생략됐을 때마다 평가된다. 상수가 아니라 함수 호출이나 표현식이어도 된다.

```kotlin
var counter = 0
fun nextId(): Int = ++counter

fun register(id: Int = nextId()) {
    println("등록: $id")
}

register()    // => 등록: 1  (nextId() 호출됨)
register()    // => 등록: 2  (또 호출됨)
register(99)  // => 등록: 99 (기본값 평가 안 됨)
```

기본값은 함수 정의 시점에 한 번 계산되는 것이 아니라(Python의 유명한 가변 기본 인자 함정과 다르다), 인자가 생략될 때마다 새로 평가된다. 그래서 위에서 `nextId()`가 호출마다 실행되어 다른 값이 나온다.

기본값은 앞서 선언된 파라미터를 참조할 수도 있다. 순서상 뒤의 파라미터가 앞 파라미터를 기본값 계산에 쓸 수 있다.

```kotlin
fun makeRange(start: Int, end: Int = start + 10) {
    println("$start..$end")
}
makeRange(5)      // => 5..15
makeRange(5, 20)  // => 5..20
```

역방향은 안 된다. `end`의 기본값이 뒤에 오는 파라미터를 참조할 수는 없다. 앞에서 뒤로만 흐른다.

### 4.3 내부 구현 — synthetic `$default` 메서드와 비트마스크

기본 인자는 어떻게 컴파일될까? JVM 백엔드는 원래 함수 외에 합성(synthetic) 메서드를 하나 더 생성한다. 이름은 원래 이름 뒤에 `$default`가 붙는다. 이 합성 메서드는 원래 파라미터 전부에 더해, 어떤 인자가 생략됐는지를 표시하는 정수 비트마스크(bitmask)와, 오버로드 구분용 마커 파라미터를 추가로 받는다.

```text
소스:
   fun connect(host: String, port: Int = 8080,
               timeout: Int = 3000, useTls: Boolean = false)

JVM 백엔드가 생성하는 것 (개념적):
   // 1) 모든 인자를 명시할 때 부르는 실제 메서드
   public static void connect(String host, int port, int timeout, boolean useTls)

   // 2) 기본 인자를 처리하는 합성 메서드
   public static void connect$default(
       String host, int port, int timeout, boolean useTls,
       int mask,        // 비트마스크: 어떤 파라미터가 생략됐나
       Object marker)   // 항상 null (오버로드 시그니처 구분용)
   {
       if ((mask & 2) != 0) port = 8080;      // 비트 1 세트 → port 생략됨 (비트 0은 host)
       if ((mask & 4) != 0) timeout = 3000;   // 비트 2 세트 → timeout 생략됨
       if ((mask & 8) != 0) useTls = false;   // 비트 3 세트 → useTls 생략됨
       connect(host, port, timeout, useTls);
   }
```

호출자가 `connect("example.com")`이라고 쓰면, 컴파일러는 `connect$default`를 호출하면서 마스크에 "port·timeout·useTls가 생략됨"을 뜻하는 비트($2|4|8 = 14$, 즉 비트 1·2·3)를 넘긴다. 생략된 자리에는 더미값(0, false, null)을 넣는다. `$default` 메서드가 마스크를 보고 실제 기본값을 채운다.

파라미터가 32개를 넘으면 마스크 하나로는 부족하므로 32개마다 정수 마스크가 하나씩 추가된다. $k$번째 파라미터의 생략 여부는 $\lfloor k/32 \rfloor$번째 마스크의 $k \bmod 32$번째 비트로 표현된다. 이 설계 덕분에 기본 인자는 별도의 오버로드 폭발 없이 정수 하나로 표현된다.

> **성능 주의**: 기본 인자 호출은 `$default` 메서드를 한 단계 더 거친다. 대부분의 코드에서 이 간접 비용은 무시할 만하지만, 극한의 뜨거운 루프(hot loop)에서 수억 번 호출된다면 JIT가 인라인하기 전까지 미세한 오버헤드가 있을 수 있다. 다만 모든 인자를 명시하면 컴파일러는 `$default`를 건너뛰고 원래 메서드를 직접 부른다. 성능이 정말 문제라면 프로파일로 확인하되, 대개는 기본 인자의 가독성 이득이 압도적이다.

### 4.4 Java 상호운용과 `@JvmOverloads`

기본 인자는 순수 Kotlin의 개념이다. Java에는 그런 문법이 없다. Java 코드가 위의 `connect`를 부르려 하면, 실제로 존재하는 시그니처는 전체 파라미터를 받는 `connect(String, int, int, boolean)` 하나뿐이다(`$default`는 합성 메서드라 Java에서 직접 쓰기 어색하다). Java에서는 모든 인자를 명시해야 한다.

이 불편을 없애려면 `@JvmOverloads`를 붙인다. 그러면 컴파일러가 Java용 오버로드들을 실제로 생성한다.

```kotlin
@JvmOverloads
fun connect(
    host: String,
    port: Int = 8080,
    timeout: Int = 3000,
    useTls: Boolean = false
) { /* ... */ }
```

`@JvmOverloads`는 뒤에서부터 파라미터를 하나씩 벗겨낸 오버로드들을 생성한다: `connect(host)`, `connect(host, port)`, `connect(host, port, timeout)`, 그리고 전체. Java에서 이제 이 각각을 직접 호출할 수 있다. 자세한 상호운용은 [[45 - Java 상호운용]]이 소유한다.

> **흔한 오해**: `@JvmOverloads`가 만드는 오버로드는 "끝에서부터 벗겨낸" 조합뿐이다. `connect(host, useTls)`처럼 중간을 건너뛴 조합은 생성되지 않는다 — 그런 조합은 Kotlin에서 이름 붙은 인자로만 표현되기 때문이다. Java에는 이름 붙은 인자가 없으므로 그 유연성은 Java로 넘어가지 못한다.

### 4.5 오버라이드와 기본 인자

오버라이딩 함수는 기본값을 다시 지정할 수 없다. 기본값은 베이스 선언에서 상속된다.

```kotlin
open class Shape {
    open fun draw(scale: Int = 1) { println("scale=$scale") }
}

class Circle : Shape() {
    override fun draw(scale: Int) {   // 여기서 = 1을 다시 쓰면 컴파일 에러
        println("원, scale=$scale")
    }
}
```

이유는 모호성이다. 만약 오버라이드가 다른 기본값을 지정할 수 있다면, `Shape` 타입 참조로 `draw()`를 부를 때 어느 기본값이 쓰이는지 정적으로 결정할 수 없다. 기본값은 정적 타입에 묶이고 실제 본문은 동적 디스패치로 결정되는 이 규칙은 [[24 - 상속과 오버라이딩과 초기화 순서]]의 세계와 맞닿아 있다. Kotlin은 이 모호성을 아예 원천 차단한다 — 오버라이드는 기본값을 못 적는다.

---

## 5. 이름 붙은 인자 — 위치의 폭정에서 벗어나기

### 5.1 순서로부터의 해방

이름 붙은 인자(named argument)는 호출 시 파라미터 이름을 명시하는 문법이다.

```kotlin
fun formatDate(year: Int, month: Int, day: Int, separator: String = "-") =
    "$year$separator$month$separator$day"

formatDate(2026, 7, 13)                              // 위치 기반
formatDate(year = 2026, month = 7, day = 13)         // 이름 기반
formatDate(day = 13, month = 7, year = 2026)         // 순서 뒤바꿔도 OK
formatDate(2026, 7, 13, separator = "/")             // 혼합
```

이름을 쓰면 인자의 순서를 자유롭게 바꿀 수 있고, 무엇보다 호출 지점이 자기 설명적(self-documenting)이 된다. `booleanFlag(true, false, true)`가 무슨 뜻인지는 아무도 모르지만, `booleanFlag(caseSensitive = true, wholeWord = false, useRegex = true)`는 즉시 읽힌다. 같은 타입의 파라미터가 여럿 있을 때 위치를 잘못 넣는 버그를 이름이 막아준다.

### 5.2 위치 인자와 이름 인자의 혼합 규칙

이름 붙은 인자와 위치 인자를 섞을 수 있다. Kotlin 1.4 이후로는 이름 인자 뒤에 위치 인자가 올 수도 있는데, 단 그 위치 인자들이 올바른 위치에 있을 때만 허용된다.

```kotlin
fun box(width: Int, height: Int, depth: Int) = width * height * depth

box(1, height = 2, depth = 3)     // OK: 위치 다음 이름
box(width = 1, 2, 3)              // OK (1.4+): 이름 다음 위치가 올바른 자리
box(1, 2, depth = 3)             // OK
box(height = 2, 1, 3)            // 컴파일 에러: 이름 뒤 위치가 자리를 어긋남
```

`box(height = 2, 1, 3)`가 거부되는 이유는, `height`를 이름으로 지정한 뒤 위치 인자 `1`이 어느 파라미터에 대응하는지 모호해지기 때문이다. 규칙의 본질은 "위치 인자는 언제나 자기 위치 그대로 해석된다"는 것이다.

> **역사 메모**: Kotlin 1.3까지는 이름 붙은 인자 다음에 위치 인자가 오는 것이 아예 금지였다. 1.4에서 "위치가 유지되면 허용"으로 완화됐다. 이 완화는 스프레드 연산자와 함께 vararg를 쓰는 흔한 패턴을 편하게 만들기 위한 실용적 결정이었다.

### 5.3 이름 붙은 인자는 Java 메서드에 못 쓴다

이름 붙은 인자의 가장 중요한 제약은 Java 메서드 호출에 쓸 수 없다는 것이다.

```kotlin
// Java의 StringBuilder.insert(int offset, String str)를 부를 때
val sb = StringBuilder("hello")
sb.insert(offset = 0, str = "> ")   // 컴파일 에러: Java 메서드엔 이름 인자 불가
sb.insert(0, "> ")                  // 위치로만 호출
```

이유는 바이트코드에 있다. 자바 클래스 파일은 파라미터 이름을 항상 보존하지 않는다. `-parameters` 플래그로 컴파일하지 않은 자바 라이브러리(대부분의 기존 라이브러리)의 바이트코드에는 파라미터 이름 대신 `arg0`, `arg1` 같은 합성 이름만 있거나 이름이 아예 없다. Kotlin 컴파일러는 그런 이름을 신뢰할 수 없으므로, Java 메서드 호출에서는 이름 인자를 원천 금지한다. 이름 붙은 인자는 Kotlin이 자기 함수의 파라미터 이름을 메타데이터로 온전히 알고 있기에 가능한 기능이다.

### 5.4 이름 인자가 오버로딩을 대체하는 방식

기본 인자와 이름 붙은 인자를 결합하면, 전통적으로 오버로딩으로 풀던 문제 대부분이 사라진다. Java의 오버로드 지옥은 "선택적 파라미터의 여러 조합"을 표현하려는 시도였는데, Kotlin은 그것을 기본값 + 이름으로 직접 표현한다.

```kotlin
fun createUser(
    name: String,
    age: Int = 0,
    email: String? = null,
    isAdmin: Boolean = false,
    isActive: Boolean = true
) { /* ... */ }

// 어떤 조합이든 이름으로 골라 부른다 — 오버로드 필요 없음
createUser("앨리스")
createUser("밥", age = 30)
createUser("캐럴", email = "carol@example.com", isAdmin = true)
createUser("데이브", isActive = false)
```

Java였다면 이 조합을 표현하려고 여러 개의 생성자·팩토리 메서드나 빌더 패턴을 만들어야 했다. Kotlin에서는 함수 하나가 그 모든 조합을 표현한다. 이 결합은 [[21 - 클래스와 생성자]]의 생성자 설계에도 그대로 적용되어, Kotlin에서 빌더 패턴의 필요성을 크게 줄인다.

---

## 6. vararg — 배열의 위장

### 6.1 가변 개수 인자

`vararg` 한정자를 붙이면 그 파라미터는 임의 개수의 인자를 받는다.

```kotlin
fun sumAll(vararg numbers: Int): Int {
    var total = 0
    for (n in numbers) total += n
    return total
}

sumAll()              // => 0
sumAll(1)             // => 1
sumAll(1, 2, 3, 4)    // => 10
```

호출자는 인자를 콤마로 나열하면 되고, 함수 안에서 `numbers`는 배열로 취급된다. 표준 라이브러리의 `listOf`, `setOf`, `arrayOf`, `println`의 여러 형태가 모두 vararg로 지어져 있다.

### 6.2 vararg의 정체는 배열이다

vararg 파라미터는 함수 본문 안에서 배열이다. 참조 타입이면 `Array<out T>`, 원시 타입이면 대응하는 원시 배열(`IntArray`, `LongArray`, `DoubleArray` 등)이 된다.

```kotlin
fun inspect(vararg items: String) {
    // items의 타입은 Array<out String>
    println(items.size)
    println(items[0])
    for (s in items) println(s)
}

fun inspectInts(vararg nums: Int) {
    // nums의 타입은 IntArray (박싱 없는 원시 배열)
    println(nums.sum())
}
```

여기서 `vararg items: Int`가 `Array<Int>`가 아니라 `IntArray`가 된다는 점이 중요하다. `Array<Int>`는 박싱된 `Integer[]`이지만 `IntArray`는 원시 `int[]`다. vararg가 원시 배열로 컴파일되므로, 정수 vararg를 넘길 때 불필요한 박싱이 일어나지 않는다. 이 원시 배열과 박싱 배열의 구별은 M43("Kotlin 배열은 List다"라는 오해)과 직접 연결되며 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]와 [[38 - 컬렉션 - 읽기 전용과 가변]]가 다룬다.

`out` 프로젝션이 붙는 이유는 변성 안전성 때문이다. `Array<out String>`은 공변 위치라 읽기만 안전하다 — vararg 배열은 함수가 받아서 읽는 용도이므로 공변으로 프로젝션된다. 변성의 논리는 [[33 - 제네릭 1 - 타입 파라미터와 변성]]가 소유한다.

### 6.3 스프레드 연산자 `*` — 배열을 펼치기

이미 배열이 있는데 그것을 vararg 함수에 넘기고 싶으면, 그냥 넘기면 안 된다. 배열 하나를 원소 하나로 볼지 여러 원소로 펼칠지 모호하기 때문이다. 스프레드 연산자(spread operator) `*`를 배열 앞에 붙여 "이 배열을 개별 인자로 펼쳐라"라고 명시한다.

```kotlin
fun sumAll(vararg numbers: Int): Int = numbers.sum()

val arr = intArrayOf(1, 2, 3, 4)
sumAll(arr)      // 컴파일 에러: IntArray를 Int 자리에 넘길 수 없음
sumAll(*arr)     // => 10  (배열을 펼쳐서 전달)

// 스프레드와 개별 인자를 섞을 수도 있다
sumAll(0, *arr, 5)   // => 0 + 1+2+3+4 + 5 = 15
```

스프레드는 개별 인자와 자유롭게 섞을 수 있다. `sumAll(0, *arr, 5)`는 앞뒤에 낱개 인자를 두고 가운데에 배열을 펼친다.

내부적으로 스프레드는 JVM 백엔드에서 배열의 (얕은) 복사본을 만들어 전달하는 것이 일반적이다. 정확히 언제 복사가 일어나는지는 백엔드·상황에 따라 다르지만, 최소한 스프레드와 낱개 인자를 섞을 때는 새 배열을 조립해야 하므로 복사가 발생한다. 이 복사 때문에, 원본 배열을 함수 안에서 변형해도 호출자의 배열에는 영향이 없는 경우가 많다 — 다만 이는 구현 세부이므로 의존하지 말아야 한다.

```text
sumAll(0, *arr, 5) 의 개념적 처리 (JVM 백엔드):
    ┌─────────────────────────────────────┐
    │ 새 IntArray를 조립:                  │
    │   [0] ++ copy(arr) ++ [5]            │
    │   = [0, 1, 2, 3, 4, 5]               │
    └─────────────────────────────────────┘
              │ 이 배열을 numbers로 전달
              ▼
    sumAll(numbers = [0,1,2,3,4,5])
```

> **성능 주의**: 스프레드는 배열 복사를 유발할 수 있다. 아주 큰 배열을 뜨거운 경로에서 반복적으로 스프레드하면 복사 비용이 쌓인다. 이런 경우에는 vararg 함수 대신 배열이나 리스트를 직접 받는 오버로드를 두는 편이 낫다. 표준 라이브러리도 종종 `vararg` 버전과 `Collection`/`Array` 버전을 함께 제공한다.

### 6.4 vararg의 위치 제약과 이름 붙은 인자

한 함수에 vararg 파라미터는 하나만 올 수 있다. 두 개 이상은 어디까지가 첫 vararg인지 경계가 모호해 금지된다.

vararg가 마지막 파라미터가 아니어도 되지만, vararg 뒤에 오는 파라미터들은 반드시 이름 붙은 인자로 전달해야 한다(또는 vararg를 스프레드로 넘겨야 한다).

```kotlin
fun tag(vararg classes: String, id: String) {
    println("id=$id, classes=${classes.joinToString()}")
}

tag("btn", "primary", id = "submit")   // OK: id를 이름으로
tag("btn", "primary", "submit")        // 컴파일 에러: "submit"이 vararg에 흡수됨
```

`tag`에서 `id`가 vararg `classes` 뒤에 있으므로, `id`는 반드시 이름으로 넘겨야 한다. 그러지 않으면 컴파일러는 모든 문자열을 vararg에 흡수시키고 `id`에 넘길 인자가 없다고 판단한다. 이 패턴 — vararg 뒤에 필수 파라미터를 두고 이름으로 강제하기 — 은 DSL 설계([[37 - 타입 안전 빌더와 DSL]])에서 유용하다.

### 6.5 제네릭 vararg와 표준 라이브러리

vararg는 제네릭과 결합해 강력해진다. 표준 라이브러리의 `listOf`가 그 예다.

```kotlin
fun <T> myListOf(vararg items: T): List<T> {
    val result = ArrayList<T>()
    for (item in items) result.add(item)
    return result
}

val nums = myListOf(1, 2, 3)          // List<Int>
val names = myListOf("a", "b")        // List<String>
val mixed = myListOf(1, "a", true)    // List<Any> — 공통 상위 타입으로 추론
```

`myListOf(1, "a", true)`처럼 서로 다른 타입을 섞으면, 타입 인자 `T`는 그들의 최소 공통 상위 타입(여기서는 `Any` 또는 `Comparable<*>` 등 컴파일러가 계산한 상한)으로 추론된다. 제네릭 vararg 함수를 다른 제네릭 vararg 함수에 넘길 때는 스프레드가 필요하다.

```kotlin
fun <T> wrap(vararg items: T): List<T> = myListOf(*items)  // 스프레드로 전달
```

`items`는 `Array<out T>`이고, 이것을 다시 vararg에 넘기려면 `*items`로 펼쳐야 한다.

---

## 7. 오버로딩 해소 규칙

### 7.1 오버로딩이란

같은 이름의 함수를 파라미터가 다르게 여러 개 선언할 수 있다. 이것이 오버로딩(overloading)이다. 반환 타입만 다른 오버로드는 불가능하다 — 호출 지점에서 반환 타입만으로는 어느 것을 부를지 결정할 수 없기 때문이다.

```kotlin
fun render(x: Int) = "정수: $x"
fun render(x: String) = "문자열: $x"
fun render(x: Int, y: Int) = "좌표: ($x, $y)"

render(42)        // => 정수: 42
render("hi")      // => 문자열: hi
render(1, 2)      // => 좌표: (1, 2)

// fun render(x: Int): Int = x     // 위 render(Int)와 충돌 — 컴파일 에러
```

### 7.2 해소의 우선순위

여러 오버로드가 호출과 맞을 때, 컴파일러는 "가장 구체적인(most specific)" 후보를 고른다. 대략적인 우선순위는 다음과 같다.

```text
오버로딩 해소 우선순위 (개념적, 높은 것부터):
  1. 타입 변환·기본 인자·vararg 없이 그대로 맞는 후보
  2. 기본 인자를 채워야 맞는 후보
  3. vararg로 흡수해야 맞는 후보
  4. (이 안에서) 더 구체적인 타입을 받는 후보가 덜 구체적인 것을 이긴다
```

핵심 원칙 두 가지: (a) vararg를 쓰는 후보는 고정 인자(fixed-arity) 후보보다 우선순위가 낮다. (b) 더 구체적인 타입이 이긴다.

```kotlin
fun f(x: Int) = "고정 Int"
fun f(vararg xs: Int) = "vararg"

f(1)   // => 고정 Int   (vararg보다 고정 인자 우선)
f(1, 2) // => vararg     (고정 인자로는 못 맞음)
f()     // => vararg     (0개도 vararg가 흡수)
```

`f(1)`에서 두 후보 모두 맞지만, 고정 인자 `f(Int)`가 vararg `f(vararg Int)`를 이긴다. 이 규칙 덕분에 vararg 오버로드를 추가해도 기존의 단일 인자 호출이 조용히 vararg로 넘어가지 않는다.

### 7.3 더 구체적인 타입이 이긴다

타입 계층에서 더 아래에 있는(더 구체적인) 타입을 받는 후보가 우선한다.

```kotlin
open class Animal
class Dog : Animal()

fun feed(a: Animal) = "동물 급식"
fun feed(d: Dog) = "강아지 급식"

val dog = Dog()
feed(dog)   // => 강아지 급식  (Dog가 Animal보다 구체적)

val animal: Animal = Dog()
feed(animal)  // => 동물 급식  (정적 타입이 Animal이므로)
```

`feed(dog)`에서 `dog`의 정적 타입이 `Dog`라 더 구체적인 `feed(Dog)`가 선택된다. 그러나 `feed(animal)`에서는 `animal`의 정적 타입이 `Animal`이므로 `feed(Animal)`이 선택된다 — 오버로딩 해소는 정적 타입 기반이고 컴파일 타임에 결정된다. 이것이 오버라이딩(런타임 동적 디스패치, [[24 - 상속과 오버라이딩과 초기화 순서]])과 결정적으로 다른 점이다. 오버로딩은 컴파일러가 고르고, 오버라이딩은 JVM이 런타임에 고른다.

### 7.4 수 타입과 오버로딩의 함정

정수 리터럴과 수 타입 오버로드가 만나면 미묘한 함정이 있다.

```kotlin
fun g(x: Int) = "Int"
fun g(x: Long) = "Long"

g(1)    // => Int   (정수 리터럴 1은 기본적으로 Int)
g(1L)   // => Long  (L 접미사)

fun h(x: Long) = "Long"
h(1)    // 컴파일 에러: Int를 Long에 자동 확대하지 않음
h(1L)   // => Long
```

Kotlin은 암시적 확대 변환(implicit widening)을 하지 않는다([[07 - 수 1 - Int와 Long과 부동소수점]]). Java라면 `h(1)`에서 `int` 1이 자동으로 `long`으로 확대되지만, Kotlin은 이를 거부한다. `Long`을 원하면 `1L`이라 명시하거나 `toLong()`을 호출해야 한다. 이 무-암시적-변환 규칙은 오버로딩 해소를 더 예측 가능하게 만든다 — "어느 오버로드로 갈지"가 리터럴 형태로 명확해진다.

### 7.5 기본 인자와 오버로드의 충돌

기본 인자와 오버로드를 함께 쓰면 모호성이 생길 수 있다.

```kotlin
fun p(x: Int, y: Int = 0) = "두 인자"
fun p(x: Int) = "한 인자"

p(5)   // => 한 인자  (기본 인자 없이 맞는 후보가 우선)
```

`p(5)`는 두 후보 모두 맞는다 — `p(Int)`는 그대로, `p(Int, Int=0)`은 `y`에 기본값을 채워서. 규칙에 따라 기본 인자를 채우지 않아도 되는 후보가 이긴다. 하지만 이런 설계는 읽는 사람을 혼란스럽게 하므로 대개 피하는 게 좋다. 기본 인자를 쓰는 함수와 오버로드를 동시에 두는 것은 안티패턴에 가깝다.

> **흔한 오해**: "오버로딩 해소는 인자의 런타임 타입을 본다"는 착각이다. 오버로딩은 전적으로 컴파일 타임에, 인자 표현식의 *정적 타입*으로 결정된다. `Animal`로 선언된 변수에 `Dog`가 들어 있어도 `feed(Animal)`이 불린다. 런타임 타입에 따라 다른 동작을 원하면 오버로딩이 아니라 오버라이딩(가상 함수)이나 `when (x) { is Dog -> ... }` 같은 타입 검사([[17 - 표현식으로서의 제어 흐름 - if와 when]])를 써야 한다.

---

## 8. 지역 함수와 클로저의 문턱

### 8.1 함수 안의 함수

함수 본문 안에 또 다른 함수를 선언할 수 있다. 이것이 지역 함수(local function)다.

```kotlin
fun printTriangle(rows: Int) {
    fun line(n: Int): String = "*".repeat(n)   // 지역 함수

    for (i in 1..rows) {
        println(line(i))
    }
}

printTriangle(3)
// => *
// => **
// => ***
```

`line`은 `printTriangle` 안에서만 보인다. 밖에서는 존재조차 모른다. 지역 함수의 가치는 두 가지다. 첫째, 오직 이 함수 안에서만 쓰이는 보조 로직을 캡슐화해 파일 네임스페이스를 깨끗하게 유지한다. 둘째 — 더 중요하게 — 바깥 함수의 파라미터와 지역 변수에 접근할 수 있다.

### 8.2 바깥 스코프를 붙잡는다 — 클로저의 시작

지역 함수는 자신을 감싼 함수의 파라미터·지역 변수를 클로저(closure)로 붙잡는다.

```kotlin
fun formatReport(title: String, items: List<String>): String {
    val sb = StringBuilder()

    fun appendLine(text: String) {
        sb.append(title).append(": ").append(text).append('\n')
        // title(파라미터)과 sb(지역 변수)를 캡처
    }

    for (item in items) appendLine(item)
    return sb.toString()
}

formatReport("항목", listOf("사과", "배"))
// => 항목: 사과
//    항목: 배
```

`appendLine`은 `title`과 `sb`를 인자로 받지 않았는데도 쓴다. 바깥 함수의 스코프를 그대로 붙잡았기 때문이다. 이것이 클로저의 본질 — 함수가 정의된 환경(변수들)을 함께 기억하는 것 — 이다.

주목할 점은, Kotlin 클로저는 *변수를 캡처*하지 값을 복사하지 않는다는 것이다. `sb`에 append하는 것은 바깥의 실제 `sb` 객체를 건드린다. 심지어 `var` 지역 변수를 캡처해서 수정할 수도 있다 — Java의 "effectively final" 제약이 없다.

```kotlin
fun countdown(from: Int) {
    var current = from
    fun tick() {
        println(current)
        current -= 1   // 캡처한 var를 수정 (Java에선 불가)
    }
    repeat(from) { tick() }
}
```

이 캡처의 메모리 표현(값이 아니라 `Ref` 래퍼로 감싸지는 것), 캡처의 수명, 루프 변수 캡처의 미묘함은 [[14 - 클로저와 캡처]]가 정본으로 소유한다. 여기서는 지역 함수가 클로저의 가장 단순한 형태라는 것만 확인한다. 람다도 클로저를 만들지만([[13 - 함수 타입과 람다와 함수 참조]]), 지역 함수는 이름이 있고 재귀 호출이 가능하다는 점이 다르다.

### 8.3 지역 함수의 컴파일 형태

JVM 백엔드에서 지역 함수는 어떻게 컴파일될까? 캡처 여부에 따라 다르다. 바깥 변수를 하나도 캡처하지 않는 지역 함수는 대개 바깥 클래스의 `private static` 합성 메서드로 컴파일될 수 있다. 반면 바깥 변수를 캡처하는 지역 함수는 캡처한 변수들을 어떤 형태로든(추가 파라미터나 합성 클래스의 필드로) 전달받아야 한다. 정확한 전략은 컴파일러 구현과 캡처 패턴에 따라 다르다.

```text
비캡처 지역 함수:
    fun outer() { fun helper() = 42; ... }
    → helper는 캡처가 없으니 독립적 합성 메서드로 내려갈 수 있다

캡처 지역 함수:
    fun outer(x: Int) { fun helper() = x + 1; ... }
    → helper는 x를 알아야 하므로, x를 넘겨받거나
      캡처를 담은 합성 구조를 통해 접근한다
```

핵심 교훈: 지역 함수는 공짜가 아닐 수 있다. 캡처가 있으면 객체 할당이나 추가 인자 전달이 생긴다. 대부분의 코드에서 이 비용은 무시할 만하지만, 극한 성능이 필요한 곳에서는 인라인 람다([[15 - 인라인 함수와 reified]])가 이 할당을 없애는 대안이 된다.

### 8.4 지역 함수와 비지역 반환

지역 함수 안의 `return`은 그 지역 함수만 벗어난다. 바깥 함수까지 벗어나지 않는다.

```kotlin
fun process(data: List<Int>): Int {
    fun validate(x: Int): Boolean {
        if (x < 0) return false   // validate만 벗어남
        return true
    }

    var sum = 0
    for (d in data) {
        if (!validate(d)) continue
        sum += d
    }
    return sum   // process를 벗어남
}
```

이것은 람다의 비지역 반환(non-local return)과 대조된다. 인라인 람다 안의 `return`은 바깥 함수를 벗어날 수 있다(M41). 지역 함수는 그렇지 않다 — 지역 함수의 `return`은 항상 그 지역 함수 자신을 벗어난다. M41("람다의 return은 람다만 벗어난다"는 오해)의 정본 교정은 [[14 - 클로저와 캡처]]가, 인라인의 비지역 반환 메커니즘은 [[15 - 인라인 함수와 reified]]가 소유한다. 여기서 새겨둘 것은 지역 함수와 람다가 `return` 의미론에서 다르다는 사실이다.

### 8.5 상호 재귀와 전방 참조

지역 함수는 서로를 부를 수 있지만, 전방 참조(선언 전 사용)에는 제약이 있다. 지역 함수는 선언된 이후에만 보인다.

```kotlin
fun classifyNumber(n: Int): String {
    fun isEvenHelper(x: Int): Boolean =
        if (x == 0) true else isOddHelper(x - 1)   // 컴파일 에러: isOddHelper 아직 미선언

    fun isOddHelper(x: Int): Boolean =
        if (x == 0) false else isEvenHelper(x - 1)

    return if (isEvenHelper(n)) "짝수" else "홀수"
}
```

최상위·멤버 함수는 선언 순서와 무관하게 서로를 참조할 수 있지만([[04 - 파일과 패키지와 선언 - 프로그램의 골격]]), 지역 함수는 위에서 아래로 선언 순서를 따른다. 상호 재귀가 필요하면 지역 함수 대신 최상위나 멤버 함수로 빼는 것이 자연스럽다.

---

## 9. 꼬리 재귀 — 스택 없는 재귀

### 9.1 재귀의 스택 문제

재귀 함수는 우아하지만 위험하다. 각 재귀 호출은 스택 프레임을 하나 쌓고, 재귀 깊이가 깊어지면 스택이 넘친다.

```kotlin
fun sumTo(n: Long): Long =
    if (n == 0L) 0L else n + sumTo(n - 1)

sumTo(100)       // => 5050
sumTo(1_000_000) // => StackOverflowError (JVM 백엔드에서 스택 초과)
```

`sumTo(1_000_000)`은 백만 개의 스택 프레임을 쌓으려다 죽는다. 문제의 핵심은 `n + sumTo(n - 1)`에서 재귀 호출이 *마지막 연산이 아니라는* 점이다. `sumTo(n-1)`이 값을 돌려준 뒤에 `n +`을 해야 하므로, 각 프레임은 `n`을 기억한 채 재귀 결과를 기다려야 한다.

### 9.2 꼬리 위치와 `tailrec`

만약 재귀 호출이 함수의 *마지막 연산*이라면 — 즉 꼬리 위치(tail position)에 있다면 — 스택 프레임을 재사용할 수 있다. 재귀 결과에 아무것도 더 할 게 없으니, 현재 프레임을 버리고 새 호출로 점프하면 된다. 이것이 꼬리 호출 최적화(tail call optimization)다.

JVM은 자동으로 꼬리 호출 최적화를 하지 않는다. 그래서 Kotlin은 `tailrec` 한정자를 제공한다. 이걸 붙이면 컴파일러가 꼬리 재귀를 반복문(loop)으로 변환한다.

```kotlin
tailrec fun sumTo(n: Long, acc: Long = 0L): Long =
    if (n == 0L) acc else sumTo(n - 1, acc + n)

sumTo(1_000_000)   // => 500000500000  (스택 오버플로 없음)
```

바뀐 점을 보라. 누산기(accumulator) `acc`를 추가해서, 재귀 호출 `sumTo(n - 1, acc + n)`이 함수의 마지막 연산이 되게 했다. 재귀가 돌려준 값에 더 이상 아무것도 하지 않는다. 이제 `tailrec` 컴파일러가 이 재귀를 아래의 반복문과 동등한 바이트코드로 바꾼다.

```text
tailrec fun sumTo(n, acc) = if (n==0) acc else sumTo(n-1, acc+n)
        │  컴파일러가 변환 (개념적)
        ▼
fun sumTo(n, acc) {
    while (true) {
        if (n == 0L) return acc
        val newN = n - 1
        val newAcc = acc + n
        n = newN       // 파라미터를 갱신하고
        acc = newAcc
        // 루프 top으로 점프 (새 스택 프레임 없음)
    }
}
```

스택 프레임을 새로 쌓는 대신, 파라미터 값을 갱신하고 루프 처음으로 돌아간다. 재귀의 우아함과 반복문의 스택 안전성을 동시에 얻는다.

### 9.3 꼬리 위치가 아니면 조용히 실패한다 — 경고

`tailrec`을 붙였다고 아무 재귀나 최적화되지는 않는다. 재귀 호출이 진짜로 꼬리 위치에 있어야 한다. 그렇지 않으면 컴파일러는 최적화를 하지 못하고 경고를 낸다.

```kotlin
tailrec fun sumBad(n: Long): Long =
    if (n == 0L) 0L else n + sumBad(n - 1)
// 경고: A function is marked as tail-recursive but no tail calls are found

// n + sumBad(n-1)에서 재귀 호출 뒤에 '+'가 남아 있으므로 꼬리 위치가 아님
```

여기서 `n + sumBad(n - 1)`은 재귀 호출 결과에 `n +`을 더해야 하므로 꼬리 위치가 아니다. 컴파일러는 최적화하지 못하고 경고만 낸다. 무서운 점은 이것이 *경고*라는 것이다 — 컴파일은 성공하고, 함수는 여전히 일반 재귀로 동작해 깊은 입력에서 스택 오버플로를 낸다. `tailrec`을 붙였으니 안전하다고 믿었다가 프로덕션에서 터지는 함정이다. 반드시 경고를 확인해야 한다.

> **성능 주의**: `tailrec`의 경고를 절대 무시하지 마라. "no tail calls are found" 경고는 최적화가 적용되지 않았다는 뜻이고, 함수는 여전히 스택을 쓴다. IDE의 경고를 에러로 승격시키거나, 컴파일러 옵션으로 경고를 강제하는 것이 안전하다. 꼬리 재귀를 의도했다면 재귀 호출을 반드시 꼬리 위치로 옮겨야 하며, 대개 누산기 파라미터를 추가해서 해결한다.

### 9.4 `tailrec`의 조건과 한계

컴파일러가 `tailrec`을 적용하려면 여러 조건이 맞아야 한다.

| 조건 | 설명 |
|------|------|
| 재귀 호출이 꼬리 위치 | 재귀 호출이 함수의 마지막 연산이어야 함 |
| 자기 자신을 직접 호출 | 상호 재귀(A가 B를, B가 A를)는 불가 |
| `try`/`catch`/`finally` 안 불가 | 예외 핸들러 안의 재귀 호출은 최적화 안 됨 |
| `open` 함수 불가 | 오버라이드 가능한 함수는 꼬리 재귀 불가 |

몇 가지를 풀어보자. 첫째, 상호 재귀는 지원하지 않는다. `isEven`이 `isOdd`를 부르고 `isOdd`가 `isEven`을 부르는 식은 각각이 *자기 자신*을 부르는 게 아니므로 `tailrec`이 최적화하지 못한다. 둘째, `try` 블록 안에서는 안 된다 — 예외가 발생하면 스택을 되감아야 하므로 프레임을 함부로 버릴 수 없기 때문이다.

```kotlin
tailrec fun risky(n: Int): Int =
    try {
        if (n == 0) 0 else risky(n - 1)   // try 안이라 꼬리 위치 최적화 안 됨
    } catch (e: Exception) {
        -1
    }
// 경고: 꼬리 호출 최적화 불가
```

셋째, `open` 함수는 `tailrec`이 될 수 없다. 오버라이드가 가능하면 재귀 호출이 실제로 어느 구현으로 갈지 정적으로 알 수 없어, 꼬리 호출로 안전하게 변환할 수 없다.

### 9.5 실용적 꼬리 재귀 예제

꼬리 재귀의 전형적 패턴은 "누산기를 들고 다니며 마지막에 반환"이다. 몇 가지 예를 보자.

```kotlin
// 리스트 뒤집기 (꼬리 재귀)
tailrec fun <T> reverse(list: List<T>, acc: List<T> = emptyList()): List<T> =
    if (list.isEmpty()) acc
    else reverse(list.drop(1), listOf(list.first()) + acc)

reverse(listOf(1, 2, 3))   // => [3, 2, 1]

// 최대공약수 (유클리드 호제법 — 이미 꼬리 재귀 형태)
tailrec fun gcd(a: Long, b: Long): Long =
    if (b == 0L) a else gcd(b, a % b)

gcd(48, 36)   // => 12

// 고정점 찾기 (수렴할 때까지 반복)
tailrec fun fixedPoint(f: (Double) -> Double, x: Double, eps: Double = 1e-9): Double {
    val next = f(x)
    return if (kotlin.math.abs(next - x) < eps) next
    else fixedPoint(f, next, eps)
}
```

`gcd`는 원래부터 꼬리 재귀 형태다 — 재귀 호출 `gcd(b, a % b)`가 마지막 연산이라 `tailrec`을 붙이는 것만으로 최적화된다. 유클리드 호제법이 아름다운 이유 중 하나다. 재귀의 수학적 우아함을 지키면서 반복문의 효율을 얻는다.

수학적으로, 꼬리 재귀는 재귀 관계식 $f(n) = g(f(n-1))$을 $g$가 항등에 가까운 형태 — 즉 $f(n, acc) = f(n-1, h(acc, n))$ — 로 변형해 스택을 누산기로 대체한다. 재귀 깊이 $O(n)$의 스택 공간을 $O(1)$로 줄이는 변환이다. 이 변환은 함수형 프로그래밍의 오래된 관용구이며, `tailrec`은 그것을 언어가 안전하게 보증해주는 장치다.

> **역사 메모**: 많은 함수형 언어(Scheme, Scala, Haskell 등)는 꼬리 호출 최적화를 자동으로 하거나 강하게 보장한다. JVM은 바이트코드 레벨에서 꼬리 호출을 자동 제거하지 않으므로, JVM 위에 얹힌 언어들은 각자 방법을 마련해야 했다. Scala는 `@tailrec` 애노테이션을, Kotlin은 `tailrec` 한정자를 선택했다. 둘 다 "이건 꼬리 재귀여야 한다"는 개발자의 의도를 명시하게 하고, 아니면 경고/에러로 알려주는 방식이다. Kotlin이 애노테이션이 아니라 소프트 키워드(soft keyword) `tailrec`을 택한 것은 이 최적화가 함수의 계약이 아니라 구현 세부라는 관점을 반영한다.

---

이 장을 열며 던진 질문들로 돌아가자. Kotlin 함수는 JVM 백엔드에서 파일 이름 기반 클래스의 정적 메서드가 되고, 최상위에 자유롭게 놓인 함수도 그렇게 물성을 얻는다. 파라미터가 `val`이라는 것은 재대입 불가일 뿐 객체 불변이 아니며(M3의 그림자), 그 불변성이 스마트 캐스트를 떠받친다. 표현식 본문만 반환 타입을 추론하고 블록 본문은 계약을 명시해야 하며, 값이 없어 보이는 함수도 실은 Unit이라는 값을 반환한다(M10). 기본 인자는 `$default` 합성 메서드와 비트마스크로 구현되어 오버로드 폭발을 없애고, 이름 붙은 인자는 위치의 폭정에서 호출 지점을 해방하되 Java 메서드엔 닿지 못한다. vararg는 가변 인자의 탈을 쓴 배열이며 스프레드 `*`로 배열을 펼친다. 오버로딩 해소는 정적 타입 기반의 컴파일 타임 게임으로, 고정 인자가 vararg를, 구체적 타입이 일반 타입을 이긴다. 지역 함수는 클로저의 문턱을 열어 바깥 변수를 붙잡고, `tailrec`은 꼬리 위치의 재귀를 반복문으로 바꿔 스택을 지운다 — 다만 꼬리 위치가 아니면 조용한 경고와 함께 실패한다. 이 모든 것이 "평범한 함수"의 이야기다. 다음 장은 이 함수 자체를 값으로 다루기 시작한다 — 함수 타입, 람다, 그리고 함수 참조가 여는 일급 함수의 세계다.

---

## 핵심 요약

- **함수는 JVM 백엔드에서 파일 이름 기반 클래스(`FileNameKt`)의 정적 메서드로 컴파일된다.** 클래스 없이 최상위에 놓인 함수도 이렇게 물성을 얻으며, 이 서술은 "JVM 백엔드에서"라는 조건이 붙는다(다른 타깃에선 각 타깃의 함수가 된다).
- **함수 파라미터는 암묵적으로 `val`이라 재대입할 수 없지만, 이는 참조의 재대입 불가일 뿐 객체 불변을 뜻하지 않는다.** 가변 객체를 파라미터로 받으면 함수는 그 내부를 변형할 수 있다(M3 참조).
- **표현식 본문 `= expr`만 반환 타입을 추론하고, 블록 본문 `{ ... }`는 (Unit이 아닌 한) 반환 타입을 명시해야 한다.** 재귀 함수는 표현식 본문이라도 반환 타입을 명시해야 한다(순환 추론 불가).
- **값을 돌려주지 않는 함수는 void가 아니라 Unit을 반환한다.** Unit은 값이 하나뿐인 실제 타입이며, JVM 바이트코드상 void로 최적화되지만 논리적으로는 값이다(M10 참조).
- **기본 인자는 `$default` 합성 메서드와 정수 비트마스크로 컴파일되며, 32개 파라미터마다 마스크가 하나씩 늘어난다.** 이 설계가 Java의 망원경 생성자·오버로드 폭발을 없앤다. 기본값은 인자가 생략될 때마다 새로 평가된다.
- **`@JvmOverloads`는 뒤에서부터 파라미터를 벗겨낸 오버로드들을 Java용으로 생성하지만, 중간을 건너뛴 조합은 만들지 않는다.** 그 유연성은 이름 붙은 인자에서 오며 Java로 넘어가지 못한다.
- **이름 붙은 인자는 호출 순서를 자유롭게 하고 코드를 자기 설명적으로 만들지만, 파라미터 이름이 보존되지 않는 Java 메서드에는 쓸 수 없다.** 기본 인자와 결합하면 오버로딩 대부분이 불필요해진다.
- **`vararg`는 가변 인자의 탈을 쓴 배열이다** — 참조 타입은 `Array<out T>`, 원시 타입은 `IntArray` 등 원시 배열로 컴파일된다. 이미 있는 배열은 스프레드 `*`로 펼쳐야 하며, 스프레드는 대개 배열을 복사한다(M43 참조).
- **오버로딩 해소는 인자의 정적 타입 기반으로 컴파일 타임에 결정된다** — 고정 인자가 vararg를, 더 구체적인 타입이 덜 구체적인 타입을 이긴다. 런타임 타입에 따라 다르게 동작하려면 오버라이딩이나 타입 검사를 써야 한다.
- **지역 함수는 바깥 함수의 파라미터·지역 변수를 클로저로 붙잡으며, `var`도 캡처해 수정할 수 있다**(Java의 effectively final 제약 없음). 지역 함수의 `return`은 그 지역 함수만 벗어나며, 이는 인라인 람다의 비지역 반환과 다르다(M41 참조).
- **`tailrec`은 꼬리 위치의 자기 재귀를 반복문으로 변환해 스택 오버플로를 없앤다.** 누산기 파라미터로 재귀 호출을 마지막 연산으로 만드는 것이 관용구다.
- **`tailrec`이 꼬리 위치를 못 찾으면 에러가 아니라 경고만 내고, 함수는 여전히 스택을 쓰는 일반 재귀로 동작한다.** 상호 재귀·`try` 블록 내부·`open` 함수는 꼬리 재귀 최적화가 불가능하다.

## 연결 노트

- [[11 - 변수와 초기화 - val과 const와 lateinit]] — 파라미터의 `val` 성질과 M3(재대입 불가 ≠ 객체 불변)의 정본이 여기 있다. 이 장의 파라미터 불변성 논의의 뿌리.
- [[13 - 함수 타입과 람다와 함수 참조]] — 이 장이 "함수 선언"을 다뤘다면, 다음 장은 함수를 값으로 다룬다. `it`(M42), 함수 타입, 람다, `::foo` 참조.
- [[14 - 클로저와 캡처]] — 지역 함수가 열어젖힌 캡처의 메모리·수명·`Ref` 래핑, 비지역 반환(M41)의 정본.
- [[15 - 인라인 함수와 reified]] — 지역 함수·람다의 할당 비용을 없애는 인라이닝, 비지역 반환의 메커니즘, `crossinline`.
- [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]] — Unit 반환(M10)과 Nothing(M11)의 정본. Unit이 void가 아닌 이유.
- [[07 - 수 1 - Int와 Long과 부동소수점]] — 암시적 확대 변환 없음이 오버로딩 해소를 예측 가능하게 만드는 근거.
- [[24 - 상속과 오버라이딩과 초기화 순서]] — 오버로딩(정적, 컴파일 타임)과 오버라이딩(동적, 런타임)의 대조, 기본 인자와 오버라이드 규칙.
- [[21 - 클래스와 생성자]] — 기본 인자·이름 붙은 인자가 생성자에 적용되어 빌더 패턴을 대체하는 방식.
- [[38 - 컬렉션 - 읽기 전용과 가변]] — vararg의 원시 배열 정체와 `Array` vs `List`의 구별(M43), 가변 파라미터의 방어적 복사.
- [[45 - Java 상호운용]] — `@JvmOverloads`, `@JvmName`, 이름 붙은 인자가 Java에 닿지 못하는 이유의 정본.
