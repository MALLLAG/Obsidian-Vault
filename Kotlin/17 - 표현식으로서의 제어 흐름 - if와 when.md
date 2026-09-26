---
title: 표현식으로서의 제어 흐름 - if와 when
date: 2026-07-13
tags: [kotlin, control-flow, when, expression, exhaustiveness, 학습노트]
---

[[16 - 스코프 함수와 수신 객체 관용구]]에서는 `let`·`run`·`with`·`apply`·`also`를 "값을 반환하는 작은 스코프"로 보았다. 이런 관점이 가능한 이유는 코틀린 전체에 적용되는 설계 결정 하나 때문이다. **코틀린에서는 거의 모든 것이 값을 만드는 표현식이다.** 스코프 함수가 표현식으로서 유용했던 것도 언어가 제어 흐름까지 표현식으로 다루기 때문이다. 이 장에서는 그 출발점인 `if`와 `when`을 명세와 바이트코드 수준까지 살펴본다.

이 장에서 다루는 것은 *분기(branching)* 로서의 제어 흐름이다. 조건에 따라 한 갈래를 고르고, 그 갈래가 만든 값을 전체 표현식의 값으로 삼는 구조를 말한다. `if`와 `when`을 시작으로 블록의 값, 분기 타입의 최소 상계(least upper bound), 완전성 검사, fall-through가 없다는 점, 스마트 캐스트와의 결합, 2.1의 가드 조건을 차례로 다룬다.

마지막으로 JVM 백엔드가 `when`을 `tableswitch`/`lookupswitch`/`if-else` 사슬로 컴파일하는 과정을 본다. 반면 *반복(looping)* 으로서의 제어 흐름, 즉 `for`·`while`·`in`·범위(`..`)와 `break`/`continue`의 라벨 규칙은 [[18 - 범위와 진행과 반복]]에서 설명하고, 이 장에서는 `when` 안에서의 `break`/`continue`만 짧게 짚는다.

---

## 1. 문과 표현식: 코틀린이 새로 정한 경계

### 1.1 두 개념의 정의

전통적인 명령형 언어(C, Java)는 문법 요소를 **문**(statement)과 **표현식**(expression)으로 나눈다. 표현식은 평가되면 *값을 만든다*(`3 + 4`, `f(x)`, `a > b`). 문은 *효과를 만들지만 값은 만들지 않는다*(`if (c) { ... }`, `for (...) { ... }`, `return`). Java에서 `if`는 문이므로, 아래 코드는 Java에서 문법 오류이다.

```java
// Java — 컴파일 에러: if는 표현식이 아니다
int max = if (a > b) a else b;
```

Java는 이 빈자리를 채우려고 **삼항 연산자**(ternary operator) `? :`를 따로 둔다.

```java
// Java — 삼항 연산자로 우회
int max = (a > b) ? a : b;
```

코틀린은 이 구분을 다시 정리했다. 코틀린에도 문과 표현식의 구분은 있지만, **`if`와 `when`과 `try`는 표현식**이다. 즉 값을 만든다. 그래서 코틀린에는 삼항 연산자가 아예 없다. 필요가 없기 때문이다.

```kotlin
// Kotlin — if가 표현식이므로 그 값을 바로 대입
val max = if (a > b) a else b   // => a와 b 중 큰 값
```

> [!info] 역사 메모
> 코틀린 설계자들은 삼항 연산자를 "읽기 어렵고, 중첩하면 알아보기 힘들어지는 특수 문법"으로 보았다. 그리고 `if`를 표현식으로 만들면 삼항 연산자를 따로 둘 필요가 없다고 판단했다. `? :`가 없는 것은 실수가 아니라 의도적으로 문법을 줄인 결과이다.
>
> 그 대가로 `if (a > b) a else b`는 삼항 연산자보다 몇 글자 길다. 하지만 중첩할 때는 `if/else if/else` 사슬이 `?:`의 우선순위 함정보다 훨씬 명료하다. Scala와 Rust, 그리고 부분적으로 Swift도 같은 방식을 택했다.

### 1.2 문으로도 쓰이는 표현식

여기서 중요한 점은 `if`가 표현식이라고 해서 문처럼 쓸 수 없다는 뜻은 *아니라는* 것이다. 코틀린에서 `if`는 **문맥에 따라 표현식으로도, 문으로도** 쓰인다.

```kotlin
// 문처럼 — 반환값을 버리고 부수효과만 취함
if (balance < 0) {
    println("잔액 부족")
}

// 표현식처럼 — 값을 대입에 사용
val grade = if (score >= 90) "A" else "B"
```

둘을 구분하는 규칙은 단순하다. **결과값이 실제로 쓰이면(대입, 인자, 반환 등) 표현식으로 취급되고, 이때에만 완결성 조건(`if`에는 `else`가 필요하다)이 강제된다.** 값을 버리는 문 위치에서는 `else`가 없어도 된다.

```kotlin
// 문 위치 — else 없어도 OK (값을 안 씀)
if (isReady) start()

// 표현식 위치 — else 없으면 컴파일 에러
val x = if (isReady) 1
// 컴파일 에러: 'if' must have both main and 'else' branches if used as an expression
```

`else`가 필요한 이유는 완전성이다. 표현식은 *반드시* 값을 만들어야 하는데, 조건이 거짓일 때 값이 없으면 표현식이 성립하지 않는다. 이 원리는 `when`의 완전성 검사(5절)로 그대로 이어진다.

### 1.3 문의 타입은 Unit

그렇다면 문으로 쓰인 `if`의 타입은 무엇일까? 코틀린에는 값이 없는 자리가 없다. 문 위치의 `if`도 값을 만들며, 그 값의 타입은 `Unit`이다. `Unit`은 값이 정확히 하나뿐인 실제 타입이고(자세한 내용은 [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]]), 그 유일한 값은 `Unit`이라는 객체이다. Java의 `void`는 "값 없음"을 뜻하지만, 코틀린에는 "값 없음"이 없다. 그 대신 "정보가 없는 값"이 있다.

```kotlin
val nothing: Unit = if (flag) println("a") else println("b")
// println의 반환 타입이 Unit이므로 이 if 표현식의 타입도 Unit
// nothing == kotlin.Unit
```

`else`가 없는 문 위치 `if`의 타입도 `Unit`이다. 조건이 거짓이면 아무 일도 하지 않고 `Unit`을 만드는 것으로 취급된다. 이렇게 보면 `else` 없는 `if`는 사실상 "`else { }`(빈 블록, 값은 `Unit`)"를 암묵적으로 가진 셈이다.

그래서 이런 `if`는 표현식으로 쓸 수 없다. 두 분기의 타입이 각각 `T`와 `Unit`이면 전체 타입이 `Any`로 합쳐져 쓸모가 없어지기 때문이다. 실제로 컴파일러는 표현식 위치에서 아예 `else`를 요구하도록 규칙을 정해 두었다.

---

## 2. if 표현식의 구조

### 2.1 블록의 값: 마지막 표현식 규칙

`if`의 각 분기는 단일 표현식일 수도 있고, 중괄호 블록일 수도 있다. 분기가 블록이면 **그 블록의 값은 마지막 표현식의 값**이다. 이 규칙이 코틀린의 표현식 중심 설계에서 가장 중요한 부분이다.

```kotlin
val label = if (temp >= 30) {
    val note = "무더위"           // 블록 내 지역 선언 — 값 아님
    log(note)                    // 부수효과 — 중간 표현식, 값 버려짐
    "덥다: $note"                // 마지막 표현식 — 이 값이 블록의 값
} else {
    "견딜 만함"
}
// label == "덥다: 무더위"
```

블록 안에서 마지막이 아닌 표현식의 값은 버려지고, 부수효과만 남는다. 마지막 표현식만 "블록의 값"으로 바깥에 전달된다. 블록의 마지막이 표현식이 아니라 선언(`val`/`var`/`fun`)이나 대입, 루프라면 그 블록의 값은 `Unit`이다.

```kotlin
val r = if (c) {
    doWork()
    val z = 3      // 선언은 값이 아니다 → 블록의 값은 Unit
} else {
    Unit
}
// r: Unit
```

> [!note] 명세 기준
> 코틀린에서 대입(`x = 5`)은 **표현식이 아니라 문**이고, 그 타입은 `Unit`이다. 이 점은 Java나 C와 다르다. Java에서는 `while ((line = read()) != null)`처럼 대입 결과를 조건에 쓸 수 있다. 하지만 코틀린에서는 `x = 5`가 값을 만들지 않으므로 `if (x = 5)` 같은 실수는 처음부터 컴파일되지 않는다. C의 고전적인 함정인 `if (x = 0)`("`==`를 쓰려다 `=`을 쓴 경우")을 언어 차원에서 막는 것이다.

### 2.2 분기 타입의 최소 상계(LUB)

`if`가 표현식이라면 그 타입은 무엇일까? 두 분기가 서로 다른 타입의 값을 만들면, 전체 표현식의 타입은 두 분기 타입의 **최소 상계**(least upper bound)가 된다. 즉 두 타입을 모두 포함하는 가장 좁은 공통 상위 타입이다.

$$
\text{typeof}(\texttt{if (c) } a \texttt{ else } b) \;=\; \mathrm{lub}\big(\text{typeof}(a),\ \text{typeof}(b)\big)
$$

```kotlin
val a: Int = 1
val b: Int = 2
val x = if (c) a else b        // x: Int

val s: String = "hi"
val y = if (c) a else s        // y: Any  (Int과 String의 LUB = Any)

open class Animal
class Dog : Animal()
class Cat : Animal()
val z = if (c) Dog() else Cat() // z: Animal  (Dog과 Cat의 LUB)
```

이 LUB 계산은 뒤에서 볼 `when`에도 똑같이 적용되며, 실제로는 코틀린 타입 추론 전반에 쓰이는 규칙이다. 분기 중 하나가 `Nothing` 타입이면(예: `throw`나 무한 루프, [[41 - 예외와 Nothing]]) LUB 계산에서 사실상 무시된다. `Nothing`은 모든 타입의 하위 타입이기 때문이다. 아래 관용구가 성립하는 것도 이 때문이다.

```kotlin
val port: Int = if (rawPort != null) rawPort else throw IllegalArgumentException("포트 없음")
// throw는 Nothing 타입 → LUB(Int, Nothing) = Int → port: Int
```

`throw`가 값을 만드는 표현식이라는 성질 덕분에, 코틀린의 널 처리 관용구인 `?:`(엘비스, [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]])의 우변에 `throw`나 `return`을 쓸 수 있다.

### 2.3 삼항 연산자가 없어서 생기는 효과

삼항 연산자가 없다는 것은 스타일 문제처럼 보이지만, 실제로는 몇 가지 결과를 낳는다.

- 조건 분기는 항상 `if`/`else` 키워드로 명시되므로 `?:`(엘비스)와 눈으로 구분된다. 코틀린의 `?:`는 삼항 연산자가 아니라 널 병합(null-coalescing) 전용 연산자이다.
- 중첩 조건은 `else if` 사슬로 쓰게 되므로, 우선순위를 맞추려고 괄호를 겹겹이 쓸 일이 없어진다.

```kotlin
// 중첩 — else if 사슬이 삼항 중첩보다 읽기 쉽다
val grade = if (score >= 90) "A"
            else if (score >= 80) "B"
            else if (score >= 70) "C"
            else "F"
```

- 이런 등급 분기는 뒤에서 볼 대상 없는 `when`으로 더 깔끔하게 쓸 수 있다. 그래서 실무에서는 `if/else if` 사슬이 3개를 넘으면 `when`으로 바꾸는 것이 관례이다. 즉 `if`와 `when`은 서로 경쟁하는 문법이 아니라, 같은 범위의 양 끝에 있는 도구이다.

### 2.4 if와 when의 선택 기준

`if`와 `when`은 각각 언제 쓸까? 언어가 강제하지는 않지만, 관용적으로 다음과 같이 구분한다.

| 상황 | 권장 | 이유 |
|---|---|---|
| 조건이 하나(참/거짓 이분) | `if/else` | 가장 직접적, `when`은 과함 |
| 한 값에 대한 다지 분기 | `when(x)` | 대상 한 번 명시, 상수/범위/타입 통합 |
| 서로 무관한 여러 조건 사슬 | `when {}` | `if/else if` 사슬을 평평하게 |
| sealed/enum 처리 | `when(x)` (else 없이) | 완전성으로 미래 경우를 컴파일 강제 |
| 널 검사 후 대체값 | `?:`(엘비스) | 널 병합 전용, `if`보다 간결 |

핵심은 이렇다. **분기의 기준이 "하나의 대상"이면 `when(x)`를 쓰고, "서로 독립된 여러 조건"이면 `when {}`이나 `if`를 쓴다.** 대상이 sealed나 enum이면 거의 항상 `when(x)`가 맞다. 완전성 검사가 그 경우에만 동작하기 때문이다. 완전성 검사는 5절에서 자세히 다룬다.

---

## 3. when의 두 가지 형태: 대상 있음과 대상 없음

`when`은 코틀린 제어 흐름에서 가장 많이 쓰이는 구문이다. 문법적으로는 **두 가지 형태**가 있으며, 이 둘을 혼동하면 `when`을 잘못 이해하게 된다.

### 3.1 대상 있는 when: `when(subject)`

괄호 안에 대상(subject)을 두면, 각 분기의 왼쪽은 그 대상과 *비교하는* 값으로 해석된다.

```kotlin
val name = when (dayOfWeek) {
    1 -> "월"
    2 -> "화"
    3, 4, 5 -> "평일"       // 콤마 = OR
    6, 7 -> "주말"
    else -> "알 수 없음"
}
```

여기서 `1 ->`는 "`dayOfWeek == 1`이면"이라는 뜻이다. 즉 대상 있는 `when`의 분기 조건은 대상과의 **동등성 비교**(`==`)가 기본이고, 뒤에서 볼 `in`(범위나 컬렉션 포함)과 `is`(타입 검사)로 확장된다. 콤마로 나열한 값은 OR로 묶인다. `3, 4, 5 ->`는 "3이거나 4이거나 5이면"이라는 뜻이다.

대상을 그 자리에서 변수에 **바인딩**할 수도 있다. 대상으로 쓴 계산 결과를 분기 안에서 다시 써야 할 때 유용하다.

```kotlin
when (val response = fetchStatus()) {
    200 -> render(response)          // response를 분기에서 사용
    in 400..499 -> logClientError(response)
    else -> retry(response)
}
// response의 스코프는 이 when 블록으로 한정됨
```

`when (val x = ...)` 형태는 `x`의 스코프를 `when` 블록 안으로 한정한다. 임시 변수가 바깥으로 새어 나가지 않게 하는 작은 캡슐화이다.

### 3.2 대상 없는 when: `when { }`

괄호를 아예 생략하면 `when`은 전혀 다르게 동작한다. 이때 각 분기의 왼쪽은 **독립적인 Boolean 표현식**이고, `when`은 위에서부터 아래로 내려가며 처음으로 `true`가 되는 분기를 고른다. 즉 대상 없는 `when`은 `if/else if/else` 사슬과 의미가 같다.

```kotlin
val grade = when {
    score >= 90 -> "A"
    score >= 80 -> "B"
    score >= 70 -> "C"
    else -> "F"
}
```

여기에는 대상이 없으므로 `score >= 90` 같은 각 조건은 그 자체로 `Boolean`이어야 한다. 대상 있는 `when`의 `1 ->`와 달리, 여기서 `90 ->` 같은 상수는 Boolean이 아니므로 컴파일 에러가 난다. 두 형태의 차이를 표로 정리하면 다음과 같다.

| 구분 | 대상 있는 `when(x)` | 대상 없는 `when {}` |
|---|---|---|
| 분기 왼쪽의 의미 | `x`와 비교되는 값/범위/타입 | 독립 `Boolean` 표현식 |
| `42 ->` | `x == 42` | 컴파일 에러(Boolean 아님) |
| `in 1..9 ->` | `x in 1..9` | 컴파일 에러(Boolean 필요) → `x in 1..9 ->`로 명시 |
| `is String ->` | `x is String` (+ 스마트 캐스트) | `x is String ->`로 명시 |
| 등가 형태 | `switch`에 가까운 형태 | `if/else if` 사슬 |
| 스마트 캐스트 대상 | 대상 `x` 자동 | 조건에 등장한 변수 각각 |

> [!warning] 흔한 오해
> "`when`은 `switch`다"라는 오해는 대개 대상 있는 `when`의 상수 분기만 보고 생긴다. 대상 없는 `when`을 보면 이 오해는 바로 깨진다. 대상 없는 `when`에는 비교할 값이 없고, 그냥 조건 사슬일 뿐이다. `switch`에는 이런 형태가 없다. `when`은 `switch`의 상위 집합도 아니고, 아예 다른 구조이다(5절에서 자세히 다룬다).

### 3.3 when이 표현식일 때와 문일 때

`if`와 마찬가지로 `when`도 표현식으로도, 문으로도 쓰인다.

```kotlin
// 표현식 — 값을 대입 (else 또는 완전성 필요)
val msg = when (level) {
    0 -> "안전"
    1 -> "주의"
    else -> "위험"
}

// 문 — 부수효과만 (완전성 불필요)
when (event) {
    is Click -> handleClick(event)
    is Scroll -> handleScroll(event)
    // else 없어도 문 위치에선 컴파일은 됨(단, 봉인/enum이면 경고)
}
```

표현식 위치의 `when`은 반드시 값을 만들어야 하므로 **완전해야** 한다. 이 완전성 요구가 `when`과 `switch`를 가르는 결정적인 차이이며, 5절 전체에서 이 내용을 다룬다.

---

## 4. when의 분기 조건: 상수, 범위, 타입, 임의 표현식

대상 있는 `when`의 진짜 강점은 분기 왼쪽에 상수 말고도 여러 가지를 쓸 수 있다는 점이다. 전통적으로 상수와 enum, 문자열만 허용한 Java `switch`와 대비된다.

### 4.1 상수 동등성과 콤마 OR

가장 기본적인 조건은 동등성 비교이다. 대상 있는 `when`의 `value ->`는 `subject == value`를 뜻하고, 여기서 `==`는 구조적 동등성(`equals`)이다([[10 - 불리언과 동등성과 동일성]]). 따라서 상수뿐 아니라 임의의 값과도 비교할 수 있다.

```kotlin
val threshold = computeThreshold()
val over = when (amount) {
    0 -> "없음"
    threshold -> "임계값 정확히 도달"   // 상수 아닌 값과도 비교 가능
    else -> "그 외"
}
```

Java `switch`의 `case`에는 컴파일 타임 상수만 쓸 수 있지만, 코틀린 `when`의 분기 값은 **런타임 값**이어도 된다. `threshold ->`가 성립하는 것이 그 예이다. `when`은 컴파일러가 반드시 점프 테이블로 바꾸는 구조가 아니라, 필요하면 `if/else` 사슬로도 컴파일되는 유연한 구조이기 때문이다(9절).

콤마는 OR을 뜻한다.

```kotlin
when (c) {
    'a', 'e', 'i', 'o', 'u' -> "모음"
    else -> "자음"
}
```

### 4.2 범위 포함: `in`

`in`을 쓰면 대상이 범위나 컬렉션에 속하는지 검사한다. 이 검사는 `contains` 관례([[19 - 연산자 오버로딩과 관례]])를 통해 처리되며, 범위(18장 범위와 진행과 반복)와 함께 쓰면 표현력이 매우 높다.

```kotlin
val tier = when (score) {
    in 90..100 -> "S"
    in 70..<90 -> "A"        // ..< = rangeUntil, 90 제외
    in 50..<70 -> "B"
    !in 0..100 -> "범위 오류"  // 부정도 가능
    else -> "F"
}
```

`in`은 `IntRange`뿐 아니라 임의의 `Iterable`이나 `Set`, 문자열 등 `contains`를 가진 모든 것과 함께 동작한다.

```kotlin
val vowels = setOf('a', 'e', 'i', 'o', 'u')
when (ch) {
    in vowels -> "모음"
    in 'a'..'z' -> "그 외 소문자"
    else -> "비-소문자"
}
```

Java `switch`로는 표현할 수 없는 이런 범위 분기가 실무에서 코틀린 `when`이 가장 유용한 기능이다.

### 4.3 타입 검사: `is`와 스마트 캐스트

`is`를 쓰면 대상의 런타임 타입을 검사한다. 그리고 그 분기 안에서 대상은 해당 타입으로 **스마트 캐스트**된다(자세한 내용은 7절).

```kotlin
open class Shape
class Circle(val r: Double) : Shape()
class Rectangle(val w: Double, val h: Double) : Shape()

fun area(s: Shape): Double = when (s) {
    is Circle -> Math.PI * s.r * s.r        // s가 Circle로 스마트 캐스트
    is Rectangle -> s.w * s.h               // s가 Rectangle로
    else -> 0.0
}
```

`is Circle ->` 분기 안에서는 따로 캐스트하지 않아도 `s`로 `Circle`의 멤버 `r`에 접근할 수 있다. `is`는 부정형인 `!is`도 지원한다.

### 4.4 임의 표현식 분기와 종류가 섞인 콤마 목록

대상 있는 `when`에서는 한 분기의 콤마 목록에 서로 다른 종류의 조건(상수, 범위, 타입)을 섞을 수 있다.

```kotlin
when (x) {
    0, in 100..200, is String -> handle(x)  // 세 종류를 OR로 혼합
    else -> other()
}
```

다만 이렇게 섞으면 스마트 캐스트가 약해진다. `0, is String ->` 분기 안에서는 `x`가 0일 수도 있으므로 `String`이라고 확정할 수 없고, 따라서 `String`으로 스마트 캐스트되지 않는다. 이 문제는 뒤에서 볼 가드 조건(8절)이 도입된 배경 중 하나이다.

> [!caution] 성능 주의
> 대상 있는 `when`에서 분기가 모두 `Int`나 enum 상수이고 값이 조밀하면, JVM 백엔드는 `tableswitch`(O(1) 점프 테이블)로 컴파일한다. 그러나 `in`이나 `is`, 런타임 값, 정수가 아닌 상수가 하나라도 섞이면 그 분기부터는 `if/else` 순차 비교로 컴파일된다(9절).
>
> 성능이 매우 중요한 자주 실행되는 경로(hot path)에서 이 차이를 신경 써야 한다면, 분기 종류를 섞지 않고 상수만 쓰는 `when`을 유지하는 편이 유리할 수 있다. 단, 대부분의 코드에서 이 차이는 의미가 없다.

---

## 5. 완전성(exhaustiveness): when은 switch가 아니다

이 절이 이 장에서 가장 중요한 부분이다. **`when`은 `switch`가 아니다.** 값에 따라 갈라진다는 겉모습만 보고 둘을 같다고 여기면, 코틀린 타입 시스템이 `when`에 부여한 가장 강력한 안전장치를 놓치게 된다. 그 안전장치가 바로 완전성(exhaustiveness)이다.

### 5.1 표현식 when은 완전해야 한다

`when`을 표현식으로 쓰면, 즉 그 값을 대입이나 반환, 인자로 실제로 사용하면 `when`은 반드시 **모든 경우를 다뤄야** 한다. 그렇지 않으면 컴파일 에러가 난다.

```kotlin
val label = when (n) {
    1 -> "하나"
    2 -> "둘"
}
// 컴파일 에러: 'when' expression must be exhaustive, add necessary 'else' branch
```

`n`이 `Int`이면 가능한 값이 무수히 많으므로, 표현식 `when`을 완전하게 만드는 방법은 `else`뿐이다. `else`는 "나머지 전부"를 다루므로 완전성을 보장한다.

```kotlin
val label = when (n) {
    1 -> "하나"
    2 -> "둘"
    else -> "그 밖"     // 이제 완전 → 표현식으로 OK
}
```

이 요구는 1.2에서 본 "표현식 `if`에는 `else`가 필요하다"는 규칙과 원리가 정확히 같다. 표현식은 값을 만들어야 하고, 어떤 입력에 대해서도 값이 정의되어야 하므로 모든 경우를 다뤄야 한다.

### 5.2 enum과 sealed: else 없이도 완전해지는 경우

`else`가 완전성을 얻는 *유일한* 방법이었다면 `when`은 조금 편한 `switch`에 그쳤을 것이다. `when`의 진짜 강점은 **대상 타입이 가질 수 있는 경우가 유한할 때** 드러난다. 이런 경우에는 모든 경우를 명시하면 `else` 없이도 완전해진다.

**enum**: 열거형([[29 - enum 클래스]])의 모든 상수를 다루면 완전하다.

```kotlin
enum class Direction { NORTH, SOUTH, EAST, WEST }

fun dx(d: Direction): Int = when (d) {   // else 없음 — 그런데 완전!
    Direction.NORTH -> 0
    Direction.SOUTH -> 0
    Direction.EAST -> 1
    Direction.WEST -> -1
}
```

컴파일러는 `Direction`의 상수가 정확히 네 개라는 것을 알고, 네 분기가 모두 있으므로 완전하다고 판단한다. 여기서 핵심은 **`else`가 없다**는 점이다. 나중에 `Direction`에 `UP`을 추가하면 이 `when`은 **컴파일 에러**가 된다.

```kotlin
enum class Direction { NORTH, SOUTH, EAST, WEST, UP }  // 상수 추가
// dx()의 when: 컴파일 에러 — 'UP' 분기 누락, must be exhaustive
```

이것은 `switch`로는 절대 얻을 수 없는 효과이다. Java `switch`에서 `default` 없이 enum 케이스를 나열해 두어도, 새 상수가 추가되면 컴파일러는 경고조차 하지 않는다. 그 결과 런타임에 아무 케이스에도 해당하지 않거나 엉뚱한 케이스로 가는 버그가 조용히 생긴다.

코틀린은 `else`를 뺀 완전한 `when`을 통해 "새로운 경우가 생기면 이 분기 로직을 다시 검토하라"는 약속을 **컴파일 타임에** 강제한다.

**sealed**: 봉인 클래스나 봉인 인터페이스([[30 - sealed 클래스와 대수적 데이터 타입]])의 모든 하위 타입을 다루면 완전하다.

```kotlin
sealed interface Json
data class JsonNumber(val value: Double) : Json
data class JsonString(val value: String) : Json
data object JsonNull : Json

fun render(j: Json): String = when (j) {   // else 없이 완전
    is JsonNumber -> j.value.toString()
    is JsonString -> "\"${j.value}\""
    JsonNull -> "null"
}
```

`sealed`는 하위 타입의 집합이 컴파일 타임에 **닫혀** 있으므로, 컴파일러는 "이것이 전부"라고 확신할 수 있다. sealed와 완전한 `when`의 조합은 코틀린에서 대수적 데이터 타입(ADT)과 패턴 매칭을 구현하는 표준적인 방법이다(자세한 내용은 30장).

새 하위 타입을 추가하면 완전한 `when`이 모두 컴파일 에러를 내므로, 새 경우를 어디서 처리해야 하는지 컴파일러가 목록으로 알려 준다.

**Boolean과 nullable**: `Boolean`은 `true`와 `false` 두 값뿐이므로 둘 다 다루면 완전하다. nullable enum이나 nullable sealed는 `null` 분기까지 다뤄야 완전하다.

```kotlin
fun toInt(b: Boolean): Int = when (b) {   // else 없이 완전
    true -> 1
    false -> 0
}

fun describe(d: Direction?): String = when (d) {
    Direction.NORTH -> "북"
    Direction.SOUTH -> "남"
    Direction.EAST -> "동"
    Direction.WEST -> "서"
    null -> "미정"                          // nullable이면 null도 덮어야
}
```

### 5.3 문 위치 when의 완전성: 경고와 그 변화 과정

여기에는 미묘한 부분이 있다. 지금까지 설명한 완전성 강제는 **표현식 위치**에 대한 이야기이다. 값을 버리는 **문 위치**에서 `when`을 쓰면 어떻게 될까?

```kotlin
fun handle(d: Direction) {
    when (d) {          // 문 위치 — 값 안 씀
        Direction.NORTH -> goNorth()
        Direction.SOUTH -> goSouth()
        // EAST, WEST 누락
    }
}
```

과거(Kotlin 1.x 초기)에는 문 위치의 불완전한 `when`이 아무 진단 없이 통과했다. enum에 상수를 추가해도 문 위치의 `when`은 그냥 지나쳤으므로 위험했다. 그래서 언어가 바뀌었다.

> [!info] 역사 메모
> Kotlin은 sealed나 enum을 대상으로 하는 **문 위치**의 불완전한 `when`에도 표현식 위치와 똑같이 완전성을 요구하는 방향으로 조금씩 바뀌어 왔다. 초기에는 진단이 없었고, 이후 경고(warning)가 추가되었다. K2/2.x 세대에서는 enum, sealed, Boolean처럼 "닫힌" 대상에 대한 문 위치의 불완전한 `when`을 강한 경고로 잡는다(빌드 설정에 따라 에러로 올릴 수 있다).
>
> 정확한 진단 수준(경고인지 에러인지)은 컴파일러 버전과 언어 버전 설정에 따라 다르다. 따라서 "완전한 `when`을 쓴다"는 규칙으로 통일하는 편이 안전하다. 표현식 위치에서는 처음부터 지금까지 항상 **에러**이다.

실무 규칙은 단순하다. **sealed나 enum을 `when`으로 처리할 때는 `else`를 넣지 말고 모든 경우를 명시한다.** 그래야 새 경우가 추가될 때 컴파일러가 알려 준다. `else`를 넣으면 이 안전망이 사라진다. 새 경우가 아무 경고 없이 `else`로 처리되기 때문이다.

```kotlin
// 안티패턴: else가 미래의 새 상수를 삼켜 버림
fun dx(d: Direction) = when (d) {
    Direction.EAST -> 1
    Direction.WEST -> -1
    else -> 0            // NORTH/SOUTH, 그리고 미래의 UP까지 전부 여기로
}
// Direction에 UP 추가돼도 컴파일 에러 안 남 → 버그가 숨는다
```

### 5.4 완전성의 형식적 의미

완전성을 조금 형식적으로 표현해 보자. 대상 타입 $T$가 가질 수 있는 값의 집합을 $V(T)$라고 하면, `when`의 분기 조건들이 다루는 값 집합의 합집합이 $V(T)$ 전체를 포함해야 한다.

$$
\bigcup_{i} \text{cover}(\text{branch}_i) \;\supseteq\; V(T)
$$

- `T = Int`: $V(T)$가 사실상 무한하므로 `else` 없이는 다룰 수 없다.
- `T = Boolean`: $V(T) = \{\texttt{true}, \texttt{false}\}$이므로 두 분기로 다룰 수 있다.
- `T = Direction`(enum): $V(T) = \{\text{NORTH}, \text{SOUTH}, \text{EAST}, \text{WEST}\}$이므로 네 분기로 다룰 수 있다.
- `T = Json`(sealed): $V(T)$는 세 하위 타입의 값들이므로 세 개의 `is`/`object` 분기로 다룰 수 있다.

컴파일러의 완전성 검사는 이 포함 관계를 *구문만 보고* 판정할 수 있는 경우에만 완전하다고 인정한다. enum 전체 나열, sealed 전체 나열, Boolean의 두 값, `else`가 그런 경우이다.

예를 들어 `in 0..100`과 `in 101..200`으로 `Int`를 나누어도, 컴파일러는 "나머지 Int는 어떻게 하는가?"를 따지며 완전하다고 인정하지 않는다. 범위의 합집합이 `Int` 전체를 덮는지는 일반적으로 판정하지 않기 때문이다.

### 5.5 switch와 when의 차이 정리

`when`과 `switch`의 차이를 표 하나로 정리한다. `switch`(C/Java 전통형)와 코틀린 `when`은 겉모습만 닮았을 뿐, 동작 규칙이 다르다.

| 축 | Java `switch`(전통형) | 코틀린 `when` |
|---|---|---|
| 문/표현식 | 문(statement) | 표현식이자 문 |
| 분기 값 | 컴파일 타임 상수만 | 상수·런타임 값·범위·타입·임의 조건 |
| 대상 없는 형태 | 없음 | `when {}` (조건 사슬) |
| fall-through | 있음(`break` 필요) | 없음 |
| 여러 케이스 공유 | fall-through로 | 콤마 OR(`1, 2 ->`) |
| 완전성 검사 | 없음(default 선택) | 표현식이면 강제, sealed/enum은 else 없이 완전 |
| 새 enum 상수 추가 시 | 조용히 통과(버그) | 완전한 when이 컴파일 에러로 짚음 |
| 타입 좁히기 | 없음 | `is` 분기 스마트 캐스트 |
| 값 비교 방식 | 원시/참조/enum ordinal | `==`(구조적 동등, `equals`) |

Java 21의 스위치 표현식과 패턴 매칭(`switch` expression, `case X x when ...`)은 이 표의 여러 칸에서 코틀린 `when`에 가까워졌다. 표현식, 패턴, 가드, 완전성이 Java에도 들어온 것이다.

그러나 코틀린 `when`은 이 기능들을 언어 초기부터 통합된 형태로 제공했고, 대상 없는 조건 사슬(`when {}`)까지 하나의 키워드로 다룬다는 점에서 여전히 더 넓은 구조이다. 결국 "`when`은 `switch`다"라는 말은 방향이 거꾸로이다. 오히려 최신 `switch`가 `when`을 닮아 가고 있다.

---

## 6. fall-through가 없는 구조와 분기 평가 모델

### 6.1 fall-through가 없다

C/Java `switch`의 악명 높은 함정은 **fall-through**이다. `break`를 빠뜨리면 실행이 다음 `case`로 넘어간다.

```java
// Java — break를 빠뜨리면 fall-through
switch (day) {
    case 1: System.out.println("월");   // break 없음!
    case 2: System.out.println("화");   // 1일 때도 여기까지 실행됨
        break;
    default: System.out.println("그 외");
}
// day==1 → "월"과 "화" 둘 다 출력 (버그)
```

fall-through는 역사적인 실수로 널리 인정되며, 수많은 버그의 원인이 되었다. 코틀린 `when`에는 **fall-through가 아예 없다.** 매칭된 분기 하나만 실행하고 `when`이 끝난다. 그래서 `break`가 필요 없고, 이런 용도의 `break`는 존재하지도 않는다.

```kotlin
when (day) {
    1 -> println("월")   // day==1이면 이것만 실행하고 when 종료
    2 -> println("화")
    else -> println("그 외")
}
// day==1 → "월"만 출력
```

여러 경우가 같은 동작을 하도록 만들고 싶다면 fall-through 대신 콤마 OR을 쓴다(`1, 2 -> ...`). 이 설계는 fall-through의 표현력(여러 case가 동작을 공유하는 것)은 콤마로 유지하면서, fall-through의 위험(`break` 누락)은 처음부터 없앤다.

> [!warning] 흔한 오해
> "`when`에서 `break`로 빠져나온다"는 것은 오해이다. `when`에는 빠져나오기 위한 `break`가 없고, 매칭된 분기가 끝나면 자동으로 나간다. `when` *안에* 쓴 `break`/`continue`는 `when`이 아니라 `when`을 감싸는 **루프**를 제어한다(6.3).

### 6.2 위에서 아래로: 처음 매칭된 분기가 선택된다

`when`의 분기는 **작성된 순서대로 위에서 아래로** 평가되고, 처음으로 참이 되는 분기가 선택된다(first-match-wins). 그 뒤의 분기는 평가하지도 않는다. 따라서 조건이 서로 겹치면 순서에 따라 결과가 달라진다.

```kotlin
val category = when (n) {
    in 0..100 -> "작음"      // n=50이면 여기서 멈춤
    in 0..1000 -> "중간"     // 도달 못 함(위가 먼저 매칭)
    else -> "큼"
}
// n=50 → "작음"
```

좁은 범위를 아래에 두면 도달할 수 없는(unreachable) 분기가 생긴다. 위 예에서 `in 0..1000`은 `n`이 0..100이면 이미 위 분기에서 잡히지만, 101..1000이면 여기서 잡히므로 그 자체로는 도달할 수 있다. 하지만 두 분기의 순서를 뒤집으면 문제가 생긴다.

```kotlin
val category = when (n) {
    in 0..1000 -> "중간"     // 0..100도 여기서 다 잡힘
    in 0..100 -> "작음"      // 도달 불가 — 죽은 코드
    else -> "큼"
}
```

이런 "죽은 분기"는 논리 버그의 징후이다. 대상 있는 `when`에서 상수만 쓰면 값이 겹칠 수 없으므로 안전하다. 하지만 `in`이나 `is`, 조건을 섞으면 결과가 순서에 의존하게 되므로, 좁은 조건은 위에, 넓은 조건은 아래에 두는 습관이 필요하다.

처음 매칭된 분기를 고르는 이 모델은 대상 없는 `when`에서 더 두드러진다. 대상 없는 `when`은 본질적으로 `if/else if` 사슬이고, 조건들이 서로 배타적이지 않을 수 있기 때문이다.

```kotlin
val sign = when {           // score가 여러 조건에 걸릴 수 있음
    score > 0 -> "양수"
    score < 0 -> "음수"
    else -> "영"
}
```

### 6.3 when 안의 break/continue

`when` 안에서 `break`나 `continue`를 쓰면 `when`이 아니라 `when`을 감싸는 루프를 제어한다.

```kotlin
for (x in items) {
    when (x.type) {
        Type.SKIP -> continue      // 다음 반복으로 (when이 아니라 for)
        Type.STOP -> break         // for 종료
        else -> process(x)
    }
}
```

> [!info] 역사 메모
> Kotlin 초기(1.4 이전)에는 `when` 안에서 라벨 없는 `break`/`continue`를 쓸 수 없었다. 루프를 제어하려면 명시적인 라벨(`loop@ for ...`, `break@loop`)이 필요했다. 이것이 `when`의 분기 종료로 오해될 여지를 없애려는 보수적인 선택이었다.
>
> Kotlin 1.4부터 이 제약이 풀려서, `when` 안의 라벨 없는 `break`/`continue`는 가장 가까이에서 감싸는 루프를 가리키게 되었다. 오늘날에는 위 코드가 자연스럽게 동작한다.

---

## 7. 스마트 캐스트와 when의 결합

### 7.1 대상 있는 when과 is: 자동 스마트 캐스트

4.3에서 `is` 분기가 대상을 스마트 캐스트한다고 미리 언급했다. 이것은 코틀린의 널 안전성과 타입 좁히기(6장 널 안전성)에 쓰이는 흐름 분석(flow analysis)이 `when` 분기 안에도 적용되는 것이다.

```kotlin
sealed interface Event
data class KeyPress(val key: Char) : Event
data class MouseMove(val x: Int, val y: Int) : Event
data object Tick : Event

fun describe(e: Event): String = when (e) {
    is KeyPress -> "키 ${e.key}"          // e: KeyPress로 좁혀짐 → e.key
    is MouseMove -> "이동 (${e.x}, ${e.y})" // e: MouseMove → e.x, e.y
    Tick -> "틱"
}
```

`is KeyPress ->` 분기 안에서 `e`는 `KeyPress` 타입으로 확정되므로 `key`에 바로 접근할 수 있다. sealed와 함께 쓰면 이 코드는 **완전한 패턴 매칭**처럼 읽힌다. 각 하위 타입을 나열하고, 분기 안에서 그 구조에 접근하며, 새 하위 타입이 생기면 컴파일러가 알려 준다(5.2).

### 7.2 대상 없는 when의 스마트 캐스트

대상 없는 `when`에서도 각 조건에 등장한 변수는 그 분기 안에서 타입이 좁혀진다.

```kotlin
fun classify(x: Any?): String = when {
    x == null -> "널"
    x is Int && x > 0 -> "양의 정수"     // x: Int로 좁혀짐 → x > 0 유효
    x is String && x.isNotEmpty() -> "비어있지 않은 문자열"  // x: String
    else -> "그 외"
}
```

`x is Int && x > 0`에서 `&&`의 오른쪽은 왼쪽이 참일 때만 평가된다. 따라서 그 지점에서 `x`는 이미 `Int`로 스마트 캐스트되어 있고, `x > 0`은 타입 검사를 통과한다. 이것은 `if`의 스마트 캐스트와 똑같은 흐름 분석이다. `when`이 특별해서가 아니라, 언어 전체에 적용되는 흐름 기반 타입 좁히기가 여기에도 적용되는 것이다.

### 7.3 스마트 캐스트가 되지 않는 경우

스마트 캐스트가 항상 되는 것은 아니다. `when`의 대상이 `var`이면서 커스텀 게터를 가지거나, 다른 모듈의 `open` 프로퍼티이거나, 검사와 사용 사이에 값이 바뀔 수 있으면 스마트 캐스트가 거부된다.

```kotlin
class Box {
    var content: Any? = null    // var 프로퍼티
}

fun check(box: Box) {
    when (box.content) {
        is String -> {
            // box.content.length  // 스마트 캐스트 안 됨!
            // (var 프로퍼티라 when 검사 후 다른 스레드/코드가 바꿨을 수 있음)
        }
    }
}
```

이럴 때는 `when (val c = box.content)`로 지역 `val`에 바인딩하면 된다. 지역 변수는 값이 바뀌지 않으므로 스마트 캐스트된다. 3.1에서 본 `when (val x = ...)` 바인딩이 실무에서 자주 쓰이는 이유가 이것이다.

```kotlin
fun check(box: Box) {
    when (val c = box.content) {   // 지역 val에 고정
        is String -> println(c.length)  // c는 스마트 캐스트됨 → OK
        else -> {}
    }
}
```

---

## 8. 가드 조건: Kotlin 2.1이 보완한 부분

### 8.1 표현력의 빈틈

7절까지 본 `when`에는 한 가지 불편한 점이 있었다. 대상 있는 `when`에서 "타입이 X이면서 *동시에* 어떤 추가 조건을 만족하는" 분기를 자연스럽게 쓸 수 없었다. `is X` 분기 안에서 추가 조건으로 다시 분기하거나, 대상 없는 `when`으로 바꾸고 스마트 캐스트를 직접 처리해야 했다.

```kotlin
// 2.1 이전 — 중첩 when으로 우회 (장황함)
fun grade(a: Animal): String = when (a) {
    is Dog -> when {
        a.age < 1 -> "강아지"
        else -> "성견"
    }
    is Cat -> "고양이"
    else -> "기타"
}
```

Java 21의 패턴 매칭에는 `case Dog d when d.age < 1 ->` 같은 가드가 있지만, 코틀린에는 오랫동안 이에 해당하는 문법이 없었다.

### 8.2 when 가드 조건(2.1 미리보기, 2.2 정식)

**Kotlin 2.1**에서 대상 있는 `when`에 **가드 조건**(guard condition)이 미리보기(preview) 기능으로 도입되었다. 2.1에서는 옵트인(컴파일러 플래그 `-Xwhen-guards`)해야 켜지는 실험적 기능이었고, **Kotlin 2.2**에서 정식(stable) 기능이 되었다. 분기 조건 뒤에 `if <불리언 표현식>`을 붙여서 대상 매칭과 추가 조건을 한 줄로 결합한다.

```kotlin
// 2.1+ — 가드 조건 (if)
fun grade(a: Animal): String = when (a) {
    is Dog if a.age < 1 -> "강아지"       // Dog이면서 나이 < 1
    is Dog -> "성견"                       // 그 외 Dog
    is Cat if a.name == "나비" -> "특별한 고양이"
    is Cat -> "고양이"
    else -> "기타"
}
```

여기서 핵심은 `is Dog if a.age < 1`에서 `a`가 이미 `Dog`로 스마트 캐스트된 상태이므로 가드의 `a.age`가 유효하다는 점이다. 가드는 대상 매칭(타입, 값, 범위)이 성립한 *뒤에* 평가되고, 둘 다 참일 때만 분기가 선택된다. 매칭은 됐지만 가드가 거짓이면 다음 분기로 넘어간다. 위 예에서는 `is Dog if a.age < 1`이 거짓이면 `is Dog ->`로 넘어간다.

> [!note] 명세 기준
> 가드 조건 `if`는 **대상 있는 `when`에서만** 쓸 수 있다. 대상이 없으면 각 분기가 이미 Boolean이므로 가드가 필요 없다. 또한 콤마로 여러 조건을 나열한 분기에는 가드를 붙일 수 없다. 가드는 단일 조건 뒤에만 붙는다.
>
> `else` 분기에는 가드를 붙이지 않는다. `else`는 무조건 매칭되기 때문이다. 정확한 문법 조합 규칙은 2.1 릴리스 노트와 명세를 따른다.

### 8.3 가드가 완전성에 미치는 영향

가드 조건은 완전성 판정에 미묘한 영향을 준다. 컴파일러는 가드가 붙은 분기를 "이 경우를 *조건부로만* 다룬다"고 보기 때문에, 가드 분기만으로는 완전성을 채울 수 없다.

위 `grade` 예에서 `is Dog if a.age < 1`만 있고 가드 없는 `is Dog ->`가 없다면, "나이가 1 이상인 Dog"를 다루지 않으므로 sealed 대상이라도 완전하다고 인정되지 않는다. 따라서 가드 분기 뒤에는 대개 같은 타입의 가드 없는 분기나 `else`가 필요하다.

```kotlin
sealed interface Cmd
data class Move(val steps: Int) : Cmd
data object Halt : Cmd

fun run(c: Cmd): String = when (c) {
    is Move if c.steps > 0 -> "전진 ${c.steps}"
    is Move -> "정지(0 이하)"     // 무가드 — Move를 완전히 덮음
    Halt -> "멈춤"
}   // 이제 완전
```

---

## 9. 컴파일: when이 바이트코드로 바뀌는 과정

### 9.1 세 가지 변환 전략(JVM 백엔드)

`when`은 한 가지 바이트코드 형태로만 컴파일되지 않는다. JVM 백엔드의 컴파일러는 분기의 성격에 따라 **세 가지 전략** 중 하나를 고른다. 아래 내용은 JVM 백엔드 기준이며, Native/JS/Wasm 백엔드는 각자의 저수준 형태로 변환한다.

```text
대상 있는 when의 하강 (JVM 백엔드)
┌─────────────────────────────────────────────────────────────┐
│ 분기가 전부 Int/enum 상수 + 값이 조밀(dense)               │
│   → tableswitch (점프 테이블, O(1))                          │
│                                                              │
│ 분기가 전부 Int/enum 상수 + 값이 희소(sparse)              │
│   → lookupswitch (정렬된 키 이진탐색, O(log n))             │
│                                                              │
│ String 상수 분기                                             │
│   → hashCode로 lookupswitch + equals 확인                   │
│                                                              │
│ in / is / 런타임 값 / 비교 등이 섞임                        │
│   → if-else 사슬 (순차 비교, O(n))                          │
└─────────────────────────────────────────────────────────────┘
```

`tableswitch`와 `lookupswitch`는 JVM 바이트코드 명령이다. `tableswitch`는 연속된 정수 키로 배열을 인덱싱해 점프하므로 상수 시간이 걸리고, `lookupswitch`는 정렬된 (키, 오프셋) 쌍을 이진 탐색한다. Java `switch`도 정확히 이 두 명령으로 컴파일되므로, **상수만 쓴 `when`은 `switch`와 기계어 효율이 같다.** 둘의 차이는 성능이 아니라 표현력과 안전성(완전성)에 있다.

### 9.2 enum when의 실제 형태

enum을 대상으로 하는 `when`은 흥미로운 우회 과정을 거친다. enum 상수는 컴파일 타임 정수 상수가 아니라 런타임 객체이다. 그래서 컴파일러는 `ordinal` 값을 바로 쓰지 않고, 별도의 **매핑 배열**(`$VALUES`에 기반한 `$EnumSwitchMapping$`)을 만들어 각 상수를 조밀한 정수에 대응시킨 뒤 `tableswitch`를 쓴다.

이것은 Java의 `switch (enumValue)`가 쓰는 기법과 같다. enum에 상수를 추가하거나 순서를 바꿔도 다른 클래스의 스위치가 깨지지 않도록 하는, 이진 호환성을 위한 장치이다.

```text
enum when (개념적 하강, JVM 백엔드)
  d: Direction
    ↓ mapping 배열로 ordinal → 조밀 정수
  when(d) { NORTH-> ...; SOUTH-> ...; ... }
    ↓
  tableswitch on mappingArray[d.ordinal()]
```

이 내용은 백엔드 구현 세부이며 코드의 의미에는 영향을 주지 않는다. 다만 "enum `when`은 `ordinal` 순서에 의존한다"는 오해를 막는 데 도움이 된다. `when`의 의미는 상수의 동일성에 기반하며, ordinal 정수에 직접 기반하지 않는다.

### 9.3 String when: 해시 비교 후 equals

문자열을 대상으로 하는 `when`은 먼저 `hashCode()`로 `lookupswitch`를 수행하고, 해시 충돌에 대비해 각 후보를 `equals()`로 한 번 더 확인한다. 이것 역시 Java의 `switch (string)`과 같은 컴파일 전략이다.

```kotlin
val kind = when (mime) {
    "image/png", "image/jpeg" -> "이미지"
    "text/plain" -> "텍스트"
    else -> "기타"
}
```

이 코드는 `mime.hashCode()`로 후보를 좁히고 `equals`로 확정하는 형태로 컴파일된다. 이 방식은 `String`의 `==`(구조적 동등성, `equals`)와 일관된다. 문자열 `when`은 참조가 아니라 내용으로 비교한다.

### 9.4 if-else 사슬로 컴파일되는 경우

`in`이나 `is`, 런타임 값, 비교가 섞인 `when`은 점프 테이블로 만들 수 없으므로 순차적인 `if-else` 사슬이 된다. 이때 6.2에서 본 처음 매칭 우선의 순서 의존성이 바이트코드에 그대로 반영된다. 위 분기부터 차례로 조건을 검사하고, 처음 참이 되는 지점으로 점프한다.

```text
when (x) {
    is Circle -> A
    in 1..10  -> B
    else      -> C
}
   ↓ (JVM 백엔드, 개념)
   if (x instanceof Circle) goto A
   if (1 <= x && x <= 10)   goto B   // in → contains → 범위 비교
   goto C
```

이렇게 보면 대상 없는 `when`은 처음부터 순수한 `if-else` 사슬이므로 항상 순차 비교로 컴파일된다. 성능이 매우 중요한 자주 실행되는 루프(hot loop)에서 조밀한 정수 분기가 필요하다면, `in`/`is`를 피하고 상수 대상 `when`을 유지해야 `tableswitch`의 O(1)을 얻을 수 있다.

그러나 다시 강조하면, 대부분의 코드에서 이 차이는 측정되지 않을 만큼 작으며 가독성과 안전성(완전성)이 더 중요하다.

> [!caution] 성능 주의
> `when`이 `tableswitch`로 컴파일되는지 `if-else` 사슬로 컴파일되는지 직접 확인하려면, IntelliJ IDEA의 "Show Kotlin Bytecode → Decompile"로 생성된 바이트코드를 보면 된다([[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]] 참고). "라이브러리와 컴파일러 문서를 그대로 믿지 말고 표준과 산출물을 직접 확인한다"는 원칙에 따라, 성능이 정말 문제라면 추측하지 말고 디컴파일과 벤치마크로 검증한다.

---

## 10. 관용구, 함정, 안티패턴

### 10.1 값을 반환하는 when: 함수 본문으로 쓰기

`when`은 표현식이므로 단일 표현식 함수([[12 - 함수 - 인자와 vararg와 지역 함수와 꼬리 재귀]])의 본문으로 바로 쓸 수 있다. 가장 흔한 관용구이다.

```kotlin
fun httpMessage(code: Int): String = when (code) {
    in 200..299 -> "성공"
    in 300..399 -> "리다이렉트"
    in 400..499 -> "클라이언트 오류"
    in 500..599 -> "서버 오류"
    else -> "알 수 없음"
}
```

`= when { ... }`으로 쓰면 함수 전체가 하나의 표현식이 된다. 그래서 중간의 `return`이 사라지고, 완전성이 강제된다. 표현식 위치이므로 `else`나 완전한 분기가 반드시 필요하다.

### 10.2 when 대상 바인딩으로 계산 결과 재사용하기

3.1과 7.3에서 본 `when (val x = ...)`에는 세 가지 이점이 있다. 계산을 한 번만 하고, 그 결과를 분기 조건과 분기 본문에서 다시 쓰며, 변수의 스코프를 `when` 안으로 한정한다.

```kotlin
fun handle(raw: String): Result = when (val parsed = parse(raw)) {
    is Ok -> use(parsed.value)      // parse 한 번, 여기서 재사용
    is Err -> report(parsed.reason)
}
```

### 10.3 함정 1: else가 미래의 경우를 삼킨다

5.3에서 강조한 안티패턴을 다시 정리한다. sealed나 enum `when`에 습관적으로 `else`를 붙이면, 새 경우가 추가되어도 컴파일러가 경고하지 않고 그 경우를 `else`로 처리해 버린다. **sealed와 enum에는 `else`를 쓰지 말고 모든 경우를 명시한다.** `else`는 대상이 `Int`나 `String`처럼 열린 타입일 때만 쓴다.

### 10.4 함정 2: 순서 의존성을 잊는다

6.2에서 본 처음 매칭 우선 규칙 때문에, 겹치는 조건 중 넓은 조건을 위에 두면 아래 분기에는 도달할 수 없다. 이것은 컴파일 에러가 아니라 논리 버그이다.

```kotlin
// 버그: 넓은 조건이 위 → 아래 도달 불가
when {
    n >= 0 -> "음이 아님"
    n > 100 -> "큼"        // 도달 불가 — n>100이면 이미 n>=0에서 잡힘
    else -> "음수"
}
```

### 10.5 함정 3: 표현식과 문을 혼동해 타입이 뭉개진다

표현식 `when`에서 어떤 분기는 값을 만들고 다른 분기는 `Unit`을 만들면, 전체 타입이 `Any`로 합쳐져 의도하지 않은 결과가 나온다.

```kotlin
val x: Any = when (n) {
    1 -> "하나"          // String
    2 -> println("둘")   // Unit! (println 반환값)
    else -> "그 외"
}
// x의 타입: Any (String과 Unit의 LUB)
```

분기 하나에서 실수로 `println`(반환 타입 `Unit`)이 마지막 표현식이 되면 이런 일이 생긴다. 표현식 `when`의 모든 분기가 의도한 타입의 값을 만드는지 확인해야 한다.

### 10.6 함정 4: when(true)/when(false) 오용

대상 없는 `when` 대신 `when (true) { cond -> ... }`를 쓰는 코드를 가끔 볼 수 있다. 대상 없는 `when { cond -> ... }`와 동작은 같지만, 불필요하게 장황하고 의도를 흐린다. 조건 사슬에는 대상 없는 `when {}`를 쓰는 것이 표준적인 방법이다.

```kotlin
// 안티패턴
when (true) { score >= 90 -> "A"; else -> "F" }
// 정석
when { score >= 90 -> "A"; else -> "F" }
```

### 10.7 관용구: sealed와 완전한 when으로 만드는 상태 기계

sealed 계층과 완전한 `when`의 조합은 상태 기계나 표현식 트리를 평가할 때 쓰는 표준적인 방법이다(30장에서 더 자세히 다룬다). 각 상태나 노드를 하위 타입으로 정의하고 `when`으로 전이와 평가를 표현하면, 새 상태를 추가할 때 처리해야 할 지점을 컴파일러가 모두 알려 준다.

```kotlin
sealed interface Expr
data class Num(val v: Int) : Expr
data class Add(val l: Expr, val r: Expr) : Expr
data class Mul(val l: Expr, val r: Expr) : Expr

fun eval(e: Expr): Int = when (e) {   // else 없이 완전
    is Num -> e.v
    is Add -> eval(e.l) + eval(e.r)
    is Mul -> eval(e.l) * eval(e.r)
}
// Expr에 Sub를 추가하면 eval의 when이 컴파일 에러 → 처리 강제
```

이 지점에서 `if`와 `when`은 단순한 문법 편의를 넘어, 코틀린에서 안전한 프로그램을 설계하는 기본 구조가 된다.

### 10.8 관용구: when 분기 안의 블록과 조기 종료

`when` 분기의 오른쪽은 단일 표현식이거나 블록이다. 블록이면 2.1에서 본 "마지막 표현식이 블록의 값" 규칙이 그대로 적용된다. 따라서 분기 안에서 지역 계산을 하고, 마지막 값을 분기의 값으로 만들 수 있다.

```kotlin
val discount = when (tier) {
    "gold" -> {
        val base = 0.2
        val bonus = if (isBirthday) 0.05 else 0.0
        base + bonus                  // 이 값이 gold 분기의 값
    }
    "silver" -> 0.1
    else -> 0.0
}
```

분기 안에서 `return`으로 함수 자체를 조기 종료할 수도 있다. `return`은 `Nothing` 타입이므로, 이 분기가 값을 만들지 않아도 표현식 `when`의 타입 계산은 깨지지 않는다(2.2의 LUB 규칙).

```kotlin
fun price(tier: String, amount: Int): Int = when (tier) {
    "vip" -> amount / 2
    "banned" -> return 0            // 함수 조기 종료 — Nothing 타입
    else -> amount
}
```

### 10.9 함정 5: 대상 있는 when에서 ==를 오해한다

대상 있는 `when`의 `value ->`는 `subject == value`(구조적 동등성)이며, `===`(참조 동일성)가 아니다(10장 불리언과 동등성과 동일성). 따라서 값 객체(예: 내용이 같은 문자열이나 `data class`)는 참조가 달라도 매칭된다. 반대로 참조 동일성이 필요한 드문 경우에는 대상 있는 `when`으로 표현할 수 없으므로, 대상 없는 `when {}`에서 `===`를 직접 써야 한다.

```kotlin
val a = "12".plus("3")   // 런타임 생성 String
when (a) {
    "123" -> println("매칭!")   // == (equals) → 내용 비교 → 매칭됨
}
// 만약 참조 동일성이 필요하면:
when {
    a === canonical -> println("같은 객체")
    else -> println("다른 객체")
}
```
