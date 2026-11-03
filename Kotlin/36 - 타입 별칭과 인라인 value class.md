---
title: 타입 별칭과 인라인 value class - 이름과 새 타입 사이
date: 2026-07-13
tags: [kotlin, typealias, value-class, inline-class, boxing, 학습노트]
---

[[35 - 확장 함수와 프로퍼티]]에서는 타입을 수정하지 않고 기능을 덧붙이는 정적 디스패치를 살펴봤다. 확장은 기존 타입이 제공하는 기능을 넓혔을 뿐, 새 타입을 만들지는 않았다.

이 장에서는 반대 방향의 질문을 다룬다. **기존 타입에 새 이름을 붙이거나, 새 타입을 만들면서도 런타임 비용은 들지 않게 하는** 두 가지 도구인 `typealias`와 `value class`이다. 둘은 겉으로 비슷해 보인다. 둘 다 이미 있는 타입을 감싸기 때문이다. 하지만 `typealias`는 컴파일 타임에만 존재하는 별명이고, `value class`는 타입 시스템에 실제로 존재하는 새 타입이다. 이 장의 핵심은 이 차이이다.

이 장에서 바로잡을 오해는 두 가지이다. 첫째는 **`typealias`가 새 타입을 만든다는 오해**이다. 별칭은 컴파일러가 곧바로 원래 타입으로 치환하므로 타입 검사에 아무 영향을 주지 않는다. `typealias UserId = Int`를 선언해도 컴파일러는 `UserId`와 `Int`를 완전히 같은 타입으로 보기 때문에, 두 타입을 자유롭게 섞어 쓸 수 있다.

둘째는 **`value class`가 항상 박싱을 없앤다는 오해**이다. value class는 가능한 경우에만 밑바탕 값으로 펼쳐진다. nullable 위치, 제네릭 인자, 상위 타입으로의 업캐스트, 컬렉션 원소 같은 곳에서는 다시 박싱된다.

박싱과 언박싱의 원리, 부호 없는 정수의 수치적 성질은 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]에서 다루고, 이 장에서는 부호 없는 정수를 value class로 구현한 대표 사례로만 살펴본다. `inline` **함수**와 `reified`는 [[15 - 인라인 함수와 reified]]에서 다룬다. 이름은 비슷하지만 value class의 "인라인"과는 전혀 다른 메커니즘이다. 인라인 함수는 호출하는 쪽에 코드를 펼치고, value class는 값의 표현을 펼친다.

자동 생성 멤버로 값 타입을 흉내 내는 [[27 - 데이터 클래스]]와, Java와의 이름 맹글링 상호작용을 다루는 [[45 - Java 상호운용]]도 함께 참고할 수 있다. 이 장에서는 먼저 `typealias`가 왜 이름에 불과한지 설명하고, 타입 안전성이 없다는 한계 때문에 `value class`가 필요해지는 과정을 보인다. 그다음 value class가 컴파일러 안에서 어떻게 펼쳐지고 어디서 다시 박싱되는지 자세히 살펴본다.

---

## 1. `typealias`: 타입에 붙이는 또 다른 이름

### 1.1 문법과 첫 직관

`typealias`는 **이미 존재하는 타입에 두 번째 이름을 붙이는** 선언이다. 파일 최상위(top-level)에만 선언할 수 있고([[04 - 파일과 패키지와 선언 - 프로그램의 골격]]), 함수나 클래스 안에는 선언할 수 없다.

```kotlin
typealias Name = String
typealias Age = Int
typealias Predicate<T> = (T) -> Boolean

fun greet(name: Name, age: Age) {
    println("$name is $age")
}

fun main() {
    greet("Bora", 30)        // => Bora is 30
    val s: String = "Bora"
    val n: Name = s          // OK: Name과 String은 같은 타입
    greet(s, 30)             // OK: 굳이 Name으로 감쌀 필요 없다
}
```

이 예제에서 핵심이 이미 드러난다. `Name` 자리에 아무 `String`이나 넣을 수 있고, `Name`을 `String`에 대입할 수도 있다. 별칭은 원래 타입과 **완전히 호환**된다. 컴파일러 입장에서 `Name`은 곧 `String`이기 때문이다. 별칭은 타입 검사가 시작되기 전에 이미 원래 타입으로 풀린다.

### 1.2 컴파일러 안에서 일어나는 일: 즉시 확장(expansion)

`typealias`의 실체를 가장 정확하게 이해하려면, 컴파일러가 별칭을 만나는 즉시 오른쪽 타입으로 치환한다고 생각하면 된다. 타입 리졸션(type resolution) 단계에서 `Name`이라는 이름은 곧바로 `String`으로 **확장(expand)** 되고, 그 뒤의 모든 단계(오버로드 해소, 스마트 캐스트, 변성 검사)는 `String`만 본다. 별칭 자체는 바이트코드를 비롯한 어떤 런타임 산출물에도 흔적을 남기지 않는다.

```text
소스에 등장한 이름          타입 리졸션 후 컴파일러가 다루는 실제 타입
─────────────────          ──────────────────────────────────────
Name                    →   String
Age                     →   Int
Predicate<Dog>          →   (Dog) -> Boolean   [즉 Function1<Dog, Boolean>]
```

별칭의 성질은 모두 이 즉시 확장으로 설명된다. 별칭은 원본 타입을 짧게 부르는 방법일 뿐이다. JVM 백엔드에서 컴파일한 `.class` 파일을 디컴파일해 보면 `Name`이라는 문자열은 어디에도 없다. 시그니처는 전부 `String`, `int`, `Function1`로 적혀 있다.

> [!note] 명세 기준
> Kotlin 언어 명세는 타입 별칭을 "이미 존재하는 타입에 대한 대체 이름(alternative name)"으로 정의하며, 별칭이 **자신이 가리키는 타입과 동등(equivalent)** 하다고 명시한다. 즉 `typealias A = B`를 선언하면 타입 시스템에서 `A`와 `B`는 서로 바꿔 쓸 수 있다. 별칭은 새로운 클래스 파일이나 타입 엔티티를 만들지 않는다.

### 1.3 별칭의 실제 용도: 긴 타입 다루기

별칭이 새 타입을 만들지 않는다면 어디에 쓸까? 답은 **가독성**이다. Kotlin의 타입은 함수 타입과 제네릭이 겹치면 특히 길고 읽기 어려워진다. 별칭을 쓰면 이런 복잡한 타입을 이름 하나로 줄일 수 있다.

```kotlin
// 별칭 없이: 시그니처가 눈에 안 들어온다
fun register(
    handler: (event: String, retries: Int) -> Map<String, List<Int>>
) { /* ... */ }

// 별칭으로: 의도가 드러난다
typealias EventHandler = (event: String, retries: Int) -> Map<String, List<Int>>

fun register(handler: EventHandler) { /* ... */ }
```

별칭을 가장 실용적으로 쓰는 곳은 **함수 타입**에 이름을 붙이는 경우이다. `(T) -> Boolean`을 `Predicate<T>`로, `(Throwable) -> Unit`을 `ErrorHandler`로 부르면 코드를 도메인 용어로 읽을 수 있다. 콜백이 많은 API에서는 별칭이 문서 역할도 한다.

```kotlin
typealias Comparator<T> = (T, T) -> Int
typealias Callback<T> = (T) -> Unit
typealias Reducer<S, A> = (state: S, action: A) -> S

// 파라미터 이름까지 담을 수 있어 자기 문서화가 된다
typealias Distance = (from: Point, to: Point) -> Double
```

### 1.4 제네릭 별칭과 중첩 타입·플랫폼 타입 이름 줄이기

별칭에는 타입 파라미터를 선언할 수 있다. 이때 별칭의 타입 파라미터는 오른쪽 타입에 그대로 전달된다.

```kotlin
typealias StringMap<V> = Map<String, V>
typealias Table<R, C, V> = Map<R, Map<C, V>>
typealias NodeSet = Set<Node>

val scores: StringMap<Int> = mapOf("a" to 1)   // Map<String, Int>
val grid: Table<Int, Int, Char> = mapOf()       // Map<Int, Map<Int, Char>>
```

별칭은 중첩 클래스의 긴 경로를 줄이는 데도 쓰인다. 예를 들어 어떤 라이브러리가 `Outer.Inner.Deep`처럼 깊이 중첩된 타입을 노출한다면([[31 - 중첩 클래스와 이너 클래스와 익명 객체]]), 별칭으로 한 번에 부를 수 있다.

```kotlin
class Graph {
    class Node {
        class Edge(val weight: Int)
    }
}

typealias Edge = Graph.Node.Edge   // 깊은 경로를 평평하게

fun main() {
    val e: Edge = Graph.Node.Edge(5)
    println(e.weight)   // => 5
}
```

> [!info] 역사 메모
> `typealias`는 Kotlin 1.1(2017)에서 정식으로 도입됐다. 그전에는 긴 타입을 짧게 부르는 수단이 `import ... as`뿐이었는데, 이 방식은 파일 단위의 지역 별칭이어서 다른 파일에서는 보이지 않았다. `typealias`는 선언 자체가 공개 API 요소가 되어 패키지 전체에서 참조할 수 있다는 점이 다르다(가시성 한정자도 붙일 수 있다).

---

## 2. `typealias`는 새 타입을 만들지 않는다

### 2.1 오해의 원인: 새 타입을 만든 것처럼 보인다

많은 개발자가 이렇게 생각한다. "`typealias UserId = Int`를 선언했으니 이제 `UserId` 타입이 생겼고, 함수가 `UserId`를 요구하는 자리에 아무 `Int`나 넘기면 컴파일러가 막아 주겠지." **바로 이 생각이 핵심 오해이다.** 별칭은 새 타입을 만들지 않으므로 그런 보호는 전혀 없다.

```kotlin
typealias UserId = Int
typealias OrderId = Int

fun cancelOrder(orderId: OrderId) {
    println("cancel order $orderId")
}

fun main() {
    val user: UserId = 42
    val order: OrderId = 7

    cancelOrder(order)   // 의도대로
    cancelOrder(user)    // => cancel order 42
    //          ^^^^ UserId를 OrderId 자리에 넣었는데도 컴파일 통과!
    cancelOrder(999)     // 그냥 Int 리터럴도 통과
}
```

컴파일러에게 `UserId`, `OrderId`, `Int`는 **말 그대로 같은 타입**이다. 세 이름은 모두 `Int`라는 한 타입을 가리키므로 서로 자유롭게 섞인다. 사람이 보기에 `cancelOrder(user)`는 버그이지만, 컴파일러가 보기에는 `Int` 자리에 `Int`를 넣은 정상적인 호출이다.

```text
typealias가 만드는 것:                value class가 만드는 것:
  Int 라는 타입 하나                    Int 와 UserId 라는 두 타입
  ├─ 별명 UserId                        ├─ Int
  ├─ 별명 OrderId          vs           └─ UserId (별개 타입, 섞이지 않음)
  └─ 이름 Int
  (모두 같은 타입, 뒤섞임)              (서로 대입 불가, 실수 차단)
```

### 2.2 증거: 오버로드는 별칭으로 구분되지 않는다

별칭이 새 타입이 아니라는 가장 확실한 증거는 함수 오버로드이다. 두 오버로드의 파라미터 타입이 별칭으로만 다르면, 컴파일러는 이를 **같은 시그니처의 중복 선언**으로 보고 거부한다.

```kotlin
typealias Celsius = Double
typealias Fahrenheit = Double

fun describe(t: Celsius) {}
fun describe(t: Fahrenheit) {}
// 컴파일 에러: Conflicting overloads: fun describe(t: Double), fun describe(t: Double)
```

별칭이 진짜 타입이었다면 두 함수는 서로 다른 오버로드가 됐을 것이다. 하지만 컴파일러는 두 시그니처를 모두 `describe(Double)`로 확장하고, 같은 시그니처가 두 개이므로 충돌 에러를 낸다. 에러 메시지에서 두 이름이 모두 `Double`로 풀려 나온 것이 별칭이 즉시 확장된다는 사실을 직접 보여 준다.

### 2.3 별칭으로 할 수 없는 것

별칭이 이름에 불과하다는 사실에서 여러 제약이 생긴다.

| 할 수 없는 것 | 이유 |
|---|---|
| 새 타입으로서 실수 차단 | 원본과 완전히 같은 타입 |
| 별칭에 새 멤버(메서드) 추가 | 별칭에는 선언 본문이 없다. 멤버는 확장 함수로만 추가할 수 있다(35장 확장 함수와 프로퍼티) |
| 원본 타입의 계약 제한/변경 | 별칭은 원본을 100% 그대로 노출 |
| 재귀적 정의 | `typealias T = List<T>`는 금지(순환) |
| 상한(bound)이나 변성 부여 | 별칭 자체는 타입 파라미터에 제약을 걸 수 없다 |

특히 확장 함수를 별칭 이름으로 정의해도 그 함수는 원본 타입 전체에 붙는다는 점에 유의하자. `fun UserId.foo()`는 사실 `fun Int.foo()`와 완전히 같으므로, 모든 `Int`에 붙는다.

```kotlin
typealias UserId = Int
fun UserId.label() = "user#$this"

fun main() {
    println(42.label())        // => user#42  (그냥 Int에도 붙는다)
    val order: Int = 7
    println(order.label())     // => user#7   (구분 없음)
}
```

### 2.4 `import as`와의 미묘한 차이

Kotlin에는 별칭과 비슷한 기능이 하나 더 있다. 바로 임포트 별칭 `import a.b.C as D`이다. 두 기능은 목적이 겹치지만 스코프와 표현력이 다르다.

```kotlin
import java.util.Date as JavaDate     // 파일 지역, 이름 충돌 해소용
// vs
typealias Handler = (Int) -> Unit     // 패키지 전역 선언, 함수/제네릭 타입도 이름 붙임
```

| 구분 | `import ... as` | `typealias` |
|---|---|---|
| 스코프 | 그 파일 안에서만 | 선언된 곳의 가시성 범위 전역 |
| 대상 | 임포트 가능한 선언(클래스/함수 등) | 임의의 타입(함수 타입·제네릭 조합 포함) |
| 함수 타입 이름 짓기 | 불가 | 가능 |
| 새 타입 생성 | 아니오 | 아니오 (둘 다 순수 별명) |

둘 다 새 타입을 만들지 않는다는 점은 같다. 임포트 별칭은 주로 같은 이름의 클래스 두 개를 함께 쓸 때처럼 **이름 충돌**을 해결할 때 쓰고, `typealias`는 **복잡한 타입에 도메인 이름을 붙일 때** 쓴다.

> [!warning] 흔한 오해
> "별칭에 `@JvmInline`을 붙이면 값 타입이 되지 않을까?" 그렇지 않다. `@JvmInline`은 `value class` 선언에만 붙이는 애노테이션이고, `typealias`에는 문법적으로 붙일 수 없다. 별칭에는 애초에 인스턴스라는 개념이 없다. 감쌀 값도, 펼칠 표현도 없고 이름만 있을 뿐이다.

---

## 3. primitive obsession: 왜 새 타입이 필요한가

### 3.1 원시 타입 집착이라는 문제

2절에서 본 `UserId`/`OrderId` 혼동은 큰 코드베이스에서 실제로 버그를 일으키는 고전적인 문제이다. 사용자 식별자, 주문 식별자, 금액, 각도, 밀리초처럼 도메인에서 서로 다른 개념을 모두 `Int`나 `Long`, `Double`로 표현하면 타입 시스템은 이들을 구별하지 못한다. 이런 습관을 "**원시 타입 집착(primitive obsession)**"이라고 부른다.

```kotlin
// 모두 Double이라, 인자 순서를 바꿔도 컴파일이 통과한다
fun transfer(amount: Double, feeRate: Double, accountId: Double) { /* ... */ }

fun main() {
    // 잘못된 순서 — 금액과 계좌번호가 뒤바뀜. 컴파일러는 침묵.
    transfer(12345.0, 100.0, 0.03)   // 재앙
}
```

`typealias`로는 이 문제를 **해결할 수 없다**. `typealias AccountId = Double`을 만들어도 `Double` 자리에 그대로 섞여 들어가기 때문이다. 필요한 것은 이름만 다른 같은 타입이 아니라 **서로 대입할 수 없는, 실제로 다른 타입**이다.

### 3.2 클래스로 감싸기와 그 비용

실제로 다른 타입을 만드는 전통적인 방법은 필드가 하나뿐인 래퍼 클래스를 쓰는 것이다.

```kotlin
class AccountId(val value: Long)
class Amount(val cents: Long)

fun transfer(amount: Amount, to: AccountId) { /* ... */ }

fun main() {
    transfer(AccountId(7), Amount(100))
    // 컴파일 에러: type mismatch. Amount가 필요한데 AccountId를 줌.
}
```

이제 타입 시스템이 실수를 잡아낸다. `AccountId`를 `Amount` 자리에 넣으면 컴파일이 실패한다. 하지만 이 안전성에는 **런타임 비용**이 따른다. `Amount(100)`은 힙에 객체를 하나 할당한다. `Long` 하나를 담으려고 객체 헤더와 필드에 메모리를 쓰고, GC 부담이 늘고, 접근할 때마다 포인터를 따라가는 간접 참조가 일어난다. 금액 수백만 개를 다루는 루프라면 이 오버헤드는 무시할 수 없다.

```text
class Amount(val cents: Long)  의 JVM 힙 레이아웃 (개념도)
┌──────────────────────────────────┐
│ 객체 헤더 (mark word + klass ptr) │  ~12-16 bytes
│ cents: long                       │  8 bytes
│ (정렬 패딩)                        │  ~0-4 bytes
└──────────────────────────────────┘
  실제 데이터 8바이트를 담으려 24바이트 안팎 + 힙 할당 + 간접 참조
```

원하는 것은 **두 방식의 장점만** 취하는 것이다. 컴파일 타임에는 `AccountId`와 `Amount`가 별개 타입이어서 실수를 막고, 런타임에는 `long` 하나로 펼쳐져 래퍼 비용이 사라져야 한다. 이것이 `value class`가 존재하는 이유이다.

### 3.3 value class의 등장

```kotlin
@JvmInline
value class AccountId(val value: Long)

@JvmInline
value class Amount(val cents: Long)

fun transfer(amount: Amount, to: AccountId) { /* ... */ }

fun main() {
    transfer(Amount(100), AccountId(7))   // 올바름
    // transfer(AccountId(7), Amount(100)) // 컴파일 에러: 타입 안전!
}
```

이제 `AccountId`와 `Amount`는 서로 대입할 수 없는 별개의 타입이다. `typealias`로는 할 수 없던 일이다. 그러면서도 JVM 백엔드는 대부분의 상황에서 이 타입들을 **래퍼 없이 밑바탕 `long`으로 컴파일**한다. `transfer(Amount(100), AccountId(7))`의 바이트코드는 개념적으로 `long` 두 개를 넘기는 것에 가깝고, 힙 할당이 없다. 다음 절부터 이 동작의 내부와 예외를 자세히 살펴본다.

---

## 4. `value class`의 문법과 제약

### 4.1 기본 형태와 `@JvmInline`

value class의 최소 형태는 주 생성자에 **정확히 하나의 `val` 프로퍼티**를 두는 것이다. JVM 백엔드를 대상으로 할 때는 `@JvmInline` 애노테이션이 필수이다.

```kotlin
@JvmInline
value class Celsius(val degrees: Double)

@JvmInline
value class Email(val raw: String)
```

`value`는 소프트 키워드이다([[03 - 렉시컬 구조와 세미콜론 추론]]). 식별자로도 쓸 수 있고, 클래스 선언 앞에 올 때만 한정자로 해석된다. `@JvmInline`은 JVM 백엔드에 이 value class를 밑바탕 타입으로 펼치라고 지시한다.

> [!note] 명세 기준
> `value class`는 언어 차원의 개념이고, `@JvmInline`은 **JVM 백엔드 전용 애노테이션**이다. Kotlin/Native나 다른 백엔드에서는 value class를 표현하는 방식이 각 백엔드의 정책을 따른다.
>
> JVM에서는 현재 `@JvmInline`을 명시해야 인라인(펼친) 표현을 얻을 수 있다. 이렇게 둘이 나뉘어 있는 것 자체가 Kotlin이 멀티타깃 언어라는 사실을 반영한다([[46 - 멀티플랫폼과 진화와 어두운 구석]]). 이 장의 저수준 설명은 따로 언급하지 않으면 JVM 백엔드를 기준으로 한다.

### 4.2 왜 프로퍼티가 하나뿐인가

value class의 핵심 아이디어는 래퍼를 없애고 밑바탕 값 하나로 대체하는 것이다. 그러려면 감쌀 값이 **딱 하나**여야 한다. 프로퍼티가 둘이면 무엇으로 펼칠지가 정해지지 않는다.

```kotlin
@JvmInline
value class Point(val x: Int, val y: Int)
// 컴파일 에러: value class must have exactly one primary constructor parameter
```

> [!info] 역사 메모
> 여러 필드를 담는 **다중 필드 value class(multi-field value classes, MFVC)** 는 오래전부터 논의돼 온 미래 기능이다. Kotlin 2.x 시점에 안정 기능으로 확정된 것은 아니므로, 이 장은 안정 기능을 기준으로 단일 프로퍼티를 전제한다. 최신 상태는 버전별 릴리스 노트에서 확인해야 한다. 지금 여러 값을 값 타입처럼 다루려면 27장(데이터 클래스)을 쓰되, 힙 할당이 따른다는 점을 감수해야 한다.

### 4.3 프로퍼티는 반드시 `val`, 백킹 필드 없음

value class의 유일한 프로퍼티는 `val`이어야 하고, `var`는 금지된다. 밑바탕이 값 하나로 펼쳐지므로 그 값을 제자리에서 바꾼다는 개념 자체가 성립하지 않는다. 값 타입은 불변인 것이 자연스럽다([[11 - 변수와 초기화 - val과 const와 lateinit]]).

```kotlin
@JvmInline
value class Meter(var length: Double)   // 컴파일 에러: value class primary constructor parameter must be val
```

또한 value class는 **추가 상태를 가질 수 없다**. 주 생성자 프로퍼티 외에는 백킹 필드가 있는 프로퍼티를 둘 수 없고, `lateinit`도 쓸 수 없다. 백킹 필드([[22 - 프로퍼티와 백킹 필드]])가 하나라도 더 생기면 값 하나로 펼친다는 전제가 무너지기 때문이다.

```kotlin
@JvmInline
value class Temperature(val celsius: Double) {
    // 계산 프로퍼티(백킹 필드 없음)는 OK
    val fahrenheit: Double get() = celsius * 9 / 5 + 32

    // 백킹 필드가 필요한 프로퍼티는 금지
    // val cached: Double = celsius * 2   // 컴파일 에러: delegate/backing field not allowed
}
```

### 4.4 가질 수 있는 것: 메서드·계산 프로퍼티·`init`

제약이 많다고 해서 value class가 빈 껍데기인 것은 아니다. 함수, 계산 프로퍼티(백킹 필드 없는 getter), `init` 블록을 가질 수 있다. 이 덕분에 value class는 단순한 래퍼가 아니라 실제 도메인 타입이 된다.

```kotlin
@JvmInline
value class Percentage(val value: Int) {
    init {
        require(value in 0..100) { "percentage out of range: $value" }
    }

    val asFraction: Double get() = value / 100.0

    fun applyTo(base: Double): Double = base * asFraction

    operator fun plus(other: Percentage): Percentage =
        Percentage((value + other.value).coerceAtMost(100))
}

fun main() {
    val p = Percentage(30)
    println(p.applyTo(200.0))          // => 60.0
    println((p + Percentage(80)).value) // => 100 (coerce)
    // Percentage(150)                  // => IllegalArgumentException at runtime
}
```

`init` 블록을 쓸 수 있다는 점은 중요하다. value class는 **불변식(invariant)을 강제**할 수 있다. `Percentage`는 항상 0~100 사이의 값임이 보장된다. `Email`이라면 형식을, `NonEmptyString`이라면 비어 있지 않음을 생성 시점에 검증할 수 있다. 원시 타입 집착을 넘어서는 실제 가치는 여기에 있다. 이름만 다른 타입이 아니라 **검증된 도메인 타입**이 되는 것이다.

> [!info] 역사 메모
> value class의 `init` 블록은 Kotlin 1.4.30에서 허용됐다. 초기의 인라인 클래스(1.3의 `inline class`)에는 `init`이 없어서 생성자에서 검증을 할 수 없었다. `init`이 추가되면서 value class는 펼쳐지는 검증 래퍼로서 완성된 형태를 갖추게 됐다.

### 4.5 상속 제약: 인터페이스는 가능, 클래스는 불가

value class는 다른 클래스를 상속할 수 없고(상위 타입은 `Any`뿐이다), 자신도 `open`이 될 수 없다(암묵적으로 `final`이다). 상속 계층에 들어갈 수 없는 이유는 밑바탕 값으로 펼쳐지면 다형적 디스패치에 필요한 클래스 정체성을 유지할 수 없기 때문이다. 다만 **인터페이스는 구현할 수 있다**([[25 - 인터페이스]]).

```kotlin
interface Named {
    val displayName: String
}

@JvmInline
value class Sku(val code: String) : Named {
    override val displayName: String get() = "SKU-$code"
}

fun main() {
    val s = Sku("A100")
    println(s.displayName)   // => SKU-A100
}
```

그런데 인터페이스를 구현한 value class를 그 인터페이스 타입으로 다루면 박싱이 발생한다. 이 내용은 다음 절들의 핵심 주제로 이어진다. 인터페이스 구현은 가능하지만, 인터페이스 타입으로 쓰는 순간 언박싱의 이점을 잃는다.

| value class가 **가질 수 있는 것** | value class가 **가질 수 없는 것** |
|---|---|
| 정확히 하나의 `val` 프로퍼티 | 둘 이상의 프로퍼티 |
| 함수(메서드) | `var` 프로퍼티 |
| 계산 프로퍼티(백킹 필드 없는 get) | 백킹 필드 있는 추가 프로퍼티 |
| `init` 블록(불변식 검증) | `lateinit` |
| 인터페이스 구현 | 클래스 상속 / `open` |
| `companion object` | `inner` 클래스 |
| 연산자·확장 함수 | Java에서 `equals` 기반 identity 가정 |

---

## 5. 컴파일: value class는 어떻게 펼쳐지는가 (JVM 백엔드)

### 5.1 밑바탕 타입으로 대체하기

JVM 백엔드에서 `@JvmInline value class Amount(val cents: Long)`을 컴파일하면, 컴파일러는 가능한 모든 곳에서 `Amount`를 밑바탕 타입 `long`으로 **대체**한다. 지역 변수나 함수의 파라미터·반환 타입이 value class이면, 실제 바이트코드에서는 밑바탕 원시 타입이나 참조 타입이 전달된다.

```kotlin
@JvmInline
value class Amount(val cents: Long)

fun addFee(a: Amount): Amount = Amount(a.cents + 100)
```

JVM 바이트코드 수준에서 이 함수는 개념적으로 다음과 같이 컴파일된다(의사 코드).

```text
// 개념적 디컴파일 (실제 시그니처는 맹글링됨 — 5.2 참조)
public static long addFee(long a) {
    return a + 100L;
}
```

`Amount` 객체는 **어디에서도 생성되지 않는다**. `Amount(a.cents + 100)` 생성자 호출조차 `long` 산술로 바뀐다. 프로퍼티 접근 `a.cents`도 필드를 읽는 것이 아니라 그 `long` 값 자체이다. 이것이 value class의 "인라인(펼침)"이다.

15장(인라인 함수와 reified)의 `inline`은 호출하는 쪽에 함수 본문을 펼치는 것으로, 이름만 같을 뿐이다. value class에서 펼쳐지는 것은 **값의 표현(representation)** 이다.

```text
class Amount           value class Amount
(래퍼)                 (@JvmInline)
                       
Amount 인스턴스        long 값 하나
= 힙 객체              = 스택/레지스터의 원시 값
= 할당 + GC 부담       = 할당 없음
= 포인터 간접참조      = 직접 값
```

### 5.2 이름 맹글링(name mangling): 함수 이름이 바뀌는 이유

여기에는 미묘한 문제가 있다. `addFee(a: Amount)`가 바이트코드에서 `addFee(long)`이 된다면, `fun addFee(a: Long)`이라는 **다른 함수**도 `addFee(long)`이 되어 JVM 시그니처가 충돌한다. 이를 막으려고 컴파일러는 value class를 파라미터로 받는 함수의 이름 뒤에 **해시 접미사를 붙인다**. 이것이 이름 맹글링이다.

```kotlin
@JvmInline
value class Amount(val cents: Long)

fun charge(a: Amount) {}
fun charge(raw: Long) {}   // 이제 충돌하지 않는다
```

JVM 바이트코드에서 두 함수의 실제 이름은 개념적으로 다음과 같다.

```text
charge(a: Amount)  →  charge-<hash>(long)     // 예: charge-mSVTQE0()
charge(raw: Long)  →  charge(long)
```

접미사에 들어간 하이픈(`-`)은 Java 식별자에 쓸 수 없는 문자이며, 이는 **의도한 것**이다. Kotlin 컴파일러는 이 함수가 value class를 다룬다는 사실을 시그니처에 인코딩하는 동시에, Java 쪽에서 이 함수를 실수로 호출하지 못하게 막는다(Java에서는 하이픈이 든 메서드 이름을 호출할 수 없다). Kotlin 코드에서는 컴파일러가 맹글링된 이름으로 알아서 호출하므로 개발자가 이 이름을 볼 일은 거의 없다. 스택 트레이스나 디컴파일 결과에서만 보게 된다.

> [!note] 명세 기준
> 이름 맹글링에는 두 가지 목적이 있다. (1) value class 언박싱 때문에 생기는 시그니처 충돌을 막고, (2) Kotlin 전용 API임을 표시한다. 개념적으로는 [[26 - 가시성 한정자]]의 `internal` 멤버가 Java에 노출될 때 이름이 맹글링되는 것과 비슷한 기법이지만, 접미사 규칙과 목적은 다르다. 정확한 맹글링 알고리즘은 컴파일러 구현 세부 사항이고 버전에 따라 달라질 수 있으므로, **맹글링된 이름을 하드코딩해 의존하면 안 된다**.

### 5.3 박스 타입의 존재: 언박싱할 수 없을 때를 위한 대비

value class를 항상 펼칠 수 있다면 좋겠지만, 그럴 수 없는 상황(6절)이 있다. 그래서 컴파일러는 각 value class마다 **실제 클래스**(박스 표현)도 하나 생성해 둔다. `Amount`라는 JVM 클래스가 실제로 존재하며, 그 안에 `long cents` 필드가 있다. 이 박스는 밑바탕 값으로 펼칠 수 없는 곳에서만 쓰인다.

```text
value class Amount 를 위해 컴파일러가 만드는 것 (JVM):
  1. 밑바탕 표현: long          ← 펼칠 수 있는 곳에서 사용
  2. 박스 클래스: Amount        ← 펼칠 수 없는 곳에서 사용 (nullable, 제네릭 등)
  + box-impl / unbox-impl 정적 헬퍼: 두 표현 사이 변환
```

컴파일러는 상황에 따라 `box-impl`(long → Amount 객체)과 `unbox-impl`(Amount 객체 → long)을 삽입한다. 즉 value class는 언박싱된 표현과 박싱된 표현을 오가며 쓰인다. 목표는 언박싱된 표현을 유지하는 것이고, 주의할 함정은 자신도 모르게 박싱된 표현으로 넘어가는 것이다.

### 5.4 특화된 `equals`와 `hashCode`

value class에도 밑바탕 값을 기준으로 `equals`/`hashCode`가 정의된다. 컴파일러는 박싱을 피하려고 **같은 value class끼리 비교하는 특화된(typed) `equals`** 를 생성할 수 있다. 이 최적화 덕분에 `Amount == Amount` 비교는 박싱 없이 `long` 비교로 이뤄진다. 반면 컬렉션 내부처럼 `Any`의 `equals(Any?)`를 거치는 비교에서는 박싱이 일어난다.

```kotlin
@JvmInline
value class Amount(val cents: Long)

fun main() {
    val a = Amount(100)
    val b = Amount(100)
    println(a == b)     // => true  (구조적 동등, 밑바탕 long 비교 — 박싱 회피 가능)
    // === 는 value class에 쓸 수 없다: 참조 동일성 개념이 안 맞음
}
```

`===`(참조 동일성, [[10 - 불리언과 동등성과 동일성]])는 value class 인스턴스에 **적용할 수 없다**(컴파일 에러). 언박싱된 표현에는 참조라는 것이 없고, 박싱할 때마다 새 객체가 만들어질 수 있어서 참조 동일성이 의미를 잃기 때문이다. `===`가 모든 값에서 의미가 있다고 생각하기 쉽지만, 이 금지가 보여 주듯 펼쳐지는 값에 참조 동일성을 묻는 것은 의미가 없다.

---

## 6. value class가 항상 언박싱되는 것은 아니다

### 6.1 오해의 원인: 펼쳐지려면 조건이 필요하다

이제 이 장의 두 번째 핵심 주제를 살펴본다. **`@JvmInline`을 붙였으니 이 타입은 힙에 할당될 일이 절대 없다고 생각하는 것은 오해이다.** value class는 밑바탕 타입을 **직접** 쓸 수 있는 문맥에서만 펼쳐진다. 밑바탕 타입이 들어갈 수 없는 문맥에 value class가 놓이면, 컴파일러는 박스 클래스(5.3)로 감쌀 수밖에 없다. 그런 문맥은 생각보다 많다.

박싱이 일어나는 대표적인 상황을 먼저 표로 정리하고, 하나씩 살펴보자.

| 상황 | 박싱? | 이유 |
|---|---|---|
| 지역 변수 / 비-널 파라미터·반환 (같은 value class 타입) | 아니오 | 밑바탕 타입으로 펼침 |
| **nullable 위치** `Amount?` | 대개 예 | null과 값을 구별하려면 참조 표현 필요 |
| **제네릭 타입 인자** `List<Amount>`, `T = Amount` | 예 | 타입 소거·제네릭은 참조 타입만 |
| **상위 타입으로 업캐스트** (`Any`, 구현한 인터페이스) | 예 | 다형성엔 실제 객체 필요 |
| **컬렉션·배열 원소** `Array<Amount>` | 예 | 원소가 제네릭/참조 슬롯 |
| **vararg** `vararg xs: Amount` | 예 | 내부적으로 `Array<Amount>` |
| 리플렉션을 통한 접근 | 예 | 리플렉션은 박스 객체를 다룸 |

핵심 원리는 하나이다. **밑바탕 원시 값이 그대로 들어갈 수 있으면 언박싱되고, 참조(객체)가 필요하면 박싱된다.** 아래에서 각 경우를 확인한다.

### 6.2 nullable: null을 표현하려면 박스가 필요하다

밑바탕 타입이 원시 `long`이라면, `long`에는 값이 없음(null)을 담을 자리가 없다. `Amount?`가 null이 될 수 있으려면 참조로 표현해야 하므로, nullable value class는 박싱된다.

```kotlin
@JvmInline
value class Amount(val cents: Long)

fun nonNull(a: Amount): Long = a.cents        // a는 long으로 펼쳐짐 (박싱 없음)

fun nullable(a: Amount?): Long =              // a는 Amount 박스 객체 (또는 null)
    a?.cents ?: 0L                            // ← 박싱 발생
```

`Amount`는 `long`으로 펼쳐지지만, `Amount?`는 박스 객체 `Amount`(nullable 참조)로 표현된다. 함수를 작성할 때 반환 타입에 무심코 `?`를 붙이면 언박싱의 이점이 소리 없이 사라진다.

> [!caution] 성능 주의
> 밑바탕 타입이 **이미 참조 타입**인 경우(예: `value class Email(val raw: String)`)는 사정이 조금 다르다. `String`은 그 자체로 null을 표현할 수 있으므로, `Email?`이 항상 별도의 박싱을 요구하는 것은 아니다. 다만 밑바탕 값이 null일 수 있는지, value class 자체의 null과 밑바탕 String의 null을 구별해야 하는지에 따라 컴파일러의 처리가 달라진다.
>
> 이런 경계 사례의 정확한 표현은 **컴파일러 구현과 버전에 따라** 다르므로, 성능이 정말 중요하다면 디컴파일해서 실제 시그니처를 확인하는 편이 안전하다. 확실한 것은 밑바탕이 **원시 타입**일 때 nullable은 박싱된다는 점이다.

### 6.3 제네릭 인자: 소거된 제네릭은 참조만 다룬다

value class를 제네릭 타입 인자로 쓰면 박싱된다. JVM에서 제네릭은 타입 소거([[34 - 제네릭 2 - 타입 소거와 reified와 바운드]])로 컴파일되고, 소거된 제네릭 슬롯에는 참조 타입(`Object`)만 담을 수 있기 때문이다. `long` 같은 원시 표현은 제네릭 슬롯에 직접 들어갈 수 없다.

```kotlin
@JvmInline
value class Amount(val cents: Long)

fun main() {
    val one: Amount = Amount(100)      // long (언박싱)
    val list: List<Amount> = listOf(Amount(100), Amount(200))
    //        ^^^^^^^^^^^^ 각 원소는 Amount 박스 객체 — 박싱됨
    val set: Set<Amount> = setOf(one)  // 역시 박싱
}

fun <T> identity(x: T): T = x
val boxed = identity(Amount(100))      // T로 넘길 때 박싱
```

실무에서 가장 흔히 빠지는 함정이 바로 이것이다. 개발자는 금액을 `value class`로 감쌌으니 가벼워졌다고 믿지만, 그 금액을 `List`나 `Map`에 넣는 순간 원소마다 박스 객체가 생긴다. value class 값을 대량으로 저장해야 하고 언박싱이 정말 중요하다면, 밑바탕 원시 타입의 **원시 배열**(`LongArray` 등, 8장 박싱과 오버플로와 비트와 부호 없는 정수)을 직접 쓰는 편이 나을 수 있다.

### 6.4 상위 타입 업캐스트: 다형성에는 객체가 필요하다

value class를 `Any`나 구현한 인터페이스 타입으로 다루면(업캐스트) 박싱된다. 다형적 디스패치에는 실제 클래스 정보를 가진 객체가 필요한데, 펼쳐진 `long`에는 클래스 정보가 없기 때문이다.

```kotlin
interface Priced { val cents: Long }

@JvmInline
value class Amount(val cents: Long) : Priced

fun main() {
    val a = Amount(100)          // long
    val any: Any = a             // Any로 업캐스트 → 박싱
    val priced: Priced = a       // 인터페이스로 업캐스트 → 박싱
    println(priced.cents)        // => 100 (박스 객체를 통해 접근)
}
```

4.5에서 봤듯이 value class는 인터페이스를 **구현할 수는** 있지만, 그 인터페이스 타입의 변수나 파라미터로 쓰는 순간 언박싱의 이점을 잃는다. 인터페이스를 구현할 수 있다는 것과 인터페이스 타입으로 써도 언박싱이 유지된다는 것은 다른 이야기이다.

### 6.5 그 밖의 박싱 지점: vararg, 배열, 리플렉션

`vararg`는 내부적으로 배열이고, 배열 원소는 제네릭/참조 슬롯이므로 박싱된다(단, 밑바탕이 원시 타입일 때 value class 배열을 최적화할지는 별개의 논의이다). 리플렉션([[44 - 애노테이션과 리플렉션]])은 언제나 박스 객체를 다룬다.

```kotlin
@JvmInline
value class Tag(val id: Int)

fun collect(vararg tags: Tag) {       // tags: Array<Tag> — 원소 박싱
    for (t in tags) println(t.id)
}

fun main() {
    collect(Tag(1), Tag(2))           // 각 인자 박싱
    val arr: Array<Tag> = arrayOf(Tag(1))   // 박싱된 원소
}
```

### 6.6 언박싱이 보장되는 경우

지금까지의 내용을 정리해 보자. value class가 밑바탕 값으로 **펼쳐지는(언박싱)** 것이 보장되는 경우는 대략 다음과 같다.

```text
언박싱 유지 (박싱 없음):
  • 비-널 value class 타입의 지역 변수
  • 비-널 value class 타입의 함수 파라미터·반환값
  • 같은 value class 타입끼리의 직접 연산 (특화 equals 등)

박싱 발생:
  • nullable (밑바탕이 원시일 때)
  • 제네릭 타입 인자 (List<VC>, T=VC, Map의 키/값 …)
  • 상위 타입 업캐스트 (Any, 구현 인터페이스)
  • 컬렉션·배열·vararg 원소
  • 리플렉션 접근
```

value class를 박싱이 절대 없는 타입으로 오해하면 실망하기 쉽다. 정확히 말하면 **value class는 박싱을 피할 수 있는 곳에서는 피하는 타입이고, 실제 hot path에는 그런 곳이 상당히 많다.** 게다가 박싱되더라도 **타입 안전성**은 100% 유지된다. `typealias`로는 얻을 수 없던 성질이다.

즉 최악의 경우에도 value class는 필드 하나짜리 래퍼 클래스만큼은 안전하고, 운이 좋은(그리고 흔한) 경우에는 래퍼 비용까지 사라진다. "안전성은 항상, 언박싱은 가능할 때"라고 기억하면 된다.

---

## 7. 표준 라이브러리의 value class

### 7.1 부호 없는 정수: value class로 만든 수 타입

Kotlin의 부호 없는 정수 `UInt`, `ULong`, `UByte`, `UShort`는 모두 **value class로 구현**돼 있다(자세한 내용은 8장 박싱과 오버플로와 비트와 부호 없는 정수에서 다룬다). 개념적으로 `UInt`는 밑바탕 `Int`를 감싸되, 연산할 때는 부호 없는 값으로 해석하는 value class이다.

```kotlin
// 표준 라이브러리의 개념적 정의 (실제와 유사)
@JvmInline
value class UInt internal constructor(internal val data: Int) : Comparable<UInt> {
    // 부호 없는 해석으로 재정의된 연산들 …
}

fun main() {
    val u: UInt = 3_000_000_000u    // Int 범위를 넘지만 UInt엔 담긴다
    println(u)                       // => 3000000000
    // 밑바탕은 Int지만 부호 없이 해석
}
```

`UInt`가 value class라는 사실은 value class가 항상 언박싱되는 것이 아니라는 점이 왜 중요한지 다시 보여 준다. `UInt` 값 하나는 밑바탕 `int`로 펼쳐지지만, `List<UInt>`나 `UInt?`는 박싱된다. 부호 없는 정수를 대량으로 쓸 때를 위해 `UIntArray` 같은 전용 배열이 있는 것도 박싱을 피하기 위해서이다.

> [!info] 역사 메모
> 부호 없는 정수는 인라인 클래스(현재의 value class)와 함께 안정화됐다. 인라인 클래스가 1.3에서 실험적으로 도입될 때 부호 없는 정수도 함께 실험 기능이었고, value class로 이름이 정리되고 `@JvmInline`이 자리 잡으면서(1.5, 2021) 부호 없는 정수도 안정 기능이 됐다. 즉 부호 없는 정수는 value class의 유용성을 표준 라이브러리에서 처음으로 크게 입증한 사례이다.

### 7.2 `Result`: 예외 없이 성공과 실패를 전달하는 value class

`kotlin.Result<T>`도 value class이다([[41 - 예외와 Nothing]]). 성공 값이나 실패(예외)를 담는 이 타입은 밑바탕에 `Any?`를 둔다. 성공이면 값을, 실패면 내부 실패 래퍼를 담는다. value class이므로 반환할 때 대부분 언박싱되어, 예외 없이 성공과 실패를 전달하는 비용이 낮다.

```kotlin
fun parsePositive(s: String): Result<Int> = runCatching {
    s.toInt().also { require(it > 0) }
}

fun main() {
    val r = parsePositive("42")
    println(r.getOrNull())          // => 42
    println(parsePositive("-1").isFailure)  // => true
}
```

다만 `Result<T>`도 제네릭 인자로 쓰이거나 컬렉션에 담기면 박싱된다. 언박싱의 이점을 살리려면 `Result`를 함수 반환 타입으로 직접 쓰는 것이 좋다.

### 7.3 `Duration`: 도메인 값 타입의 모범

`kotlin.time.Duration`도 value class이다. 밑바탕에 `Long`(내부적으로 나노초/밀리초를 부호로 인코딩)을 감싸고, `2.seconds`, `d.inWholeMilliseconds` 같은 풍부한 API를 제공한다. value class를 검증되고 연산이 풍부한 도메인 값 타입으로 쓰는 이상적인 예이다. 밑바탕은 원시 `Long` 하나이지만, 사용하는 쪽에서 보면 완전한 시간 도메인 타입이다.

```kotlin
import kotlin.time.Duration
import kotlin.time.Duration.Companion.seconds

fun main() {
    val d: Duration = 90.seconds
    println(d.inWholeMinutes)     // => 1
    println((d + 30.seconds))     // => 2m
}
```

세 사례(`UInt`, `Result`, `Duration`)의 공통점은 분명하다. **개념적으로는 원시 값 하나이지만 그 위에 도메인 의미와 연산, 불변식을 더하고 싶을 때 value class를 쓴다.** typealias는 별명일 뿐이라 의미를 더할 수 없고, 일반 클래스는 할당 비용이 든다. value class는 그 사이의 빈자리를 채운다.

---

## 8. 주의할 점과 확인하는 방법

### 8.1 언박싱을 직접 확인하기: 디컴파일

value class의 박싱 함정이 까다로운 이유는 박싱이 일어났는지를 소스만 보고는 알 수 없기 때문이다. 확실한 방법은 [[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]]에서 다루는 도구로 바이트코드를 직접 살펴보는 것이다. JVM 백엔드에서 IDE의 "Show Kotlin Bytecode → Decompile" 기능을 쓰면, value class가 어디서 밑바탕 값으로 펼쳐지고 어디서 `box-impl`/`unbox-impl`이 삽입되는지 직접 확인할 수 있다.

[[01 - 코틀린이란 무엇인가 - 설계 철학과 세 개의 타깃]]에서 강조한 "라이브러리나 직관을 믿지 말고 실제 동작을 검증하라"는 태도가 여기서 쓸모를 발휘한다.

```kotlin
@JvmInline
value class Cents(val v: Long)

fun add(a: Cents, b: Cents): Cents = Cents(a.v + b.v)   // 펼쳐짐: (long, long) -> long
fun sumAll(xs: List<Cents>): Cents {                     // xs 원소는 박싱된 Cents
    var acc = 0L
    for (c in xs) acc += c.v                              // 반복마다 unbox
    return Cents(acc)
}
```

디컴파일해 보면 `add`는 밑바탕 `long`만 주고받는 반면, `sumAll`에서 `List<Cents>`를 순회하는 부분에는 각 원소를 박스에서 꺼내는(`unbox-impl`) 코드가 보인다. 성능이 정말 중요한 코드라면 추측보다 이렇게 확인하는 편이 낫다.

### 8.2 스택 트레이스에 나타나는 맹글링된 이름

5.2에서 본 이름 맹글링은 평소에는 보이지 않다가 **예외 스택 트레이스**에서 드러난다. value class를 받는 함수 안에서 예외가 발생하면, 스택 프레임의 메서드 이름이 `charge-mSVTQE0` 같은 형태로 찍힌다. 처음 보면 당황스럽지만, value class를 파라미터로 받는 함수라고 읽으면 된다.

```text
Exception in thread "main" java.lang.IllegalArgumentException: ...
    at ExampleKt.charge-mSVTQE0(Example.kt:12)   ← 맹글링된 이름
    at ExampleKt.main(Example.kt:20)
```

이 이름은 컴파일러 구현 세부 사항이어서 버전에 따라 달라질 수 있다. 따라서 로그 파싱이나 테스트에서 이 문자열을 하드코딩하면 쉽게 깨진다. 스택 트레이스를 읽을 때는 유용하지만, 프로그램이 의존할 값은 아니다.

### 8.3 `init` 검증은 박싱 여부와 관계없이 항상 실행된다

value class의 `init` 블록(불변식 검증)은 언박싱되든 박싱되든 밑바탕 값을 만들 때마다 실행된다. 안전성 면에서는 좋지만, `require`가 무거우면 대량으로 생성할 때 비용이 된다는 점을 알아 두어야 한다.

```kotlin
@JvmInline
value class Ratio(val v: Double) {
    init { require(v in 0.0..1.0) }   // 생성마다 검사
}

fun main() {
    val many = DoubleArray(1_000_000) { it / 1_000_000.0 }
    // many.map { Ratio(it) } 는 백만 번의 require 검사 + (List라서) 백만 번 박싱
}
```

검증 비용과 박싱 비용은 별개이다. `Ratio(x)` 하나를 지역 변수로 쓰면 박싱은 없지만 `require`는 실행된다. `List<Ratio>`에 담으면 박싱 비용까지 더해진다. 두 비용을 구분해서 생각해야 최적화할 지점을 정확히 찾을 수 있다.

### 8.4 value class와 `when`·스마트 캐스트

value class는 `is` 검사와 `when`에서 정상적으로 동작하지만([[17 - 표현식으로서의 제어 흐름 - if와 when]]), `is Cents` 같은 타입 검사에는 내부적으로 박스 표현이 필요할 수 있다. 밑바탕이 원시 `long`인 값에 대해 `Cents`인지 물으려면 클래스 정체성이 필요한데, 펼쳐진 `long`에는 그 정체성이 없기 때문이다.

실무에서는 value class를 `sealed` 계층의 대안이 아니라 **단일 값의 도메인 래퍼**로 쓰는 것이 알맞다. 여러 변형(variant)을 구별해야 한다면 [[30 - sealed 클래스와 대수적 데이터 타입]]을 쓰는 것이 맞다.

### 8.5 부생성자와 그 제약

value class도 부생성자(secondary constructor)를 가질 수 있지만, 결국 주 생성자에 위임([[21 - 클래스와 생성자]])해야 하고 본문에서 추가 상태를 만들 수 없다. 부생성자는 다른 입력으로 그 하나의 값을 계산해 주 생성자에 넘기는 편의 수단으로만 쓴다.

```kotlin
@JvmInline
value class Hex(val value: Int) {
    constructor(text: String) : this(text.toInt(16))   // 주 생성자로 위임
}

fun main() {
    println(Hex("ff").value)   // => 255
}
```

> [!warning] 흔한 오해
> "value class는 성능을 위한 것이니 검증이나 메서드는 넣지 말아야 한다." 그렇지 않다. value class의 큰 가치 중 하나가 바로 **언박싱된 값에 검증과 도메인 연산을 붙이는 것**이다(`Duration`, `Percentage`가 그 예이다). 성능은 기능을 비워서가 아니라, 박싱을 피할 수 있는 곳에서 피하기 때문에 얻어진다.

## 9. `typealias`와 `value class`: 선택 기준

### 9.1 두 도구 비교

| 관점 | `typealias` | `value class` |
|---|---|---|
| 새 타입 생성 | **아니오** (별명) | **예** (별개 타입, 실수 차단) |
| 원본과 대입 호환 | 예(완전 호환) | 아니오(명시 변환 필요) |
| 런타임 표현 | 원본 그대로 | 밑바탕 값(대개 언박싱) / 박스 |
| 힙 할당 | 없음(그냥 이름) | 대개 없음, 박싱 시 있음 |
| 멤버/메서드 추가 | 불가(확장뿐) | 가능(함수·계산 프로퍼티) |
| 불변식 검증(`init`) | 불가 | 가능 |
| 인터페이스 구현 | 불가 | 가능(단 그 타입으로 쓰면 박싱) |
| 오버로드 구분 | 불가(같은 시그니처) | 가능(별개 타입) |
| 주 용도 | 긴 타입 축약·가독성 | 타입 안전 도메인 값·박싱 회피 |

한 문장으로 정리하면 **`typealias`는 타이핑을 줄이는 도구이고, `value class`는 타입을 늘리는 도구이다.** `typealias`는 기존 타입을 부르는 이름을 하나 더할 뿐이고, `value class`는 타입 시스템에 실제로 존재하는 새 타입을 추가한다.

### 9.2 판단 흐름

```text
"이 개념에 이름을 주고 싶다"
        │
        ├─ 이 이름을 다른 것과 섞여도 상관없나? (그냥 짧게 부르고 싶을 뿐)
        │        └─ 예 → typealias
        │
        └─ 이 이름을 다른 타입과 섞이면 안 되나? (실수를 막고 싶다)
                 │
                 ├─ 값이 하나이고 런타임 비용을 아끼고 싶다 → value class
                 │
                 └─ 값이 여럿이거나 상속/복잡한 상태가 필요 → data class([[27 - 데이터 클래스]])
                        (안전은 얻되 할당 비용은 감수)
```

### 9.3 함께 쓰기: 별칭으로 value class 부르기

두 도구는 함께 쓸 수 있다. value class를 만들고, 그 value class가 들어간 긴 제네릭 조합을 별칭으로 부를 수 있다.

```kotlin
@JvmInline
value class UserId(val value: Long)

// value class를 원소로 하는 긴 타입을 별칭으로 축약
typealias UserIndex = Map<UserId, List<UserId>>

fun buildIndex(): UserIndex = emptyMap()
```

여기서 `UserId`는 실제로 새로운 타입이어서 안전성을 주고, `UserIndex`는 그 복합 타입을 짧게 부르는 이름이어서 가독성을 준다. 두 도구가 각자 잘하는 일을 맡는 셈이다.

### 9.4 KMP의 `actual typealias`: 별칭의 또 다른 쓰임

멀티플랫폼(46장 멀티플랫폼과 진화와 어두운 구석)에서 `typealias`는 특별한 역할을 하나 더 맡는다. `expect`로 선언한 타입을 특정 플랫폼에서 이미 존재하는 플랫폼 타입에 `actual typealias`로 연결할 수 있다.

```kotlin
// common
expect class Uuid

// jvmMain — JVM의 기존 타입에 연결
actual typealias Uuid = java.util.UUID
```

이는 별칭의 즉시 확장 성질을 KMP의 `expect`/`actual` 계약을 이행하는 데 활용하는 것이다. JVM에서 `Uuid`는 곧 `java.util.UUID`이다. 새 타입이 아니라는 성질이 여기서는 오히려 장점이 된다. 플랫폼에 이미 있는 타입을 그대로 가져다 쓸 수 있기 때문이다.
