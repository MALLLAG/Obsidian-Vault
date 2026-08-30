---
title: 표현식으로서의 제어 흐름 - if와 when
date: 2026-07-13
tags: [kotlin, control-flow, when, expression, exhaustiveness, 학습노트]
---

**이 장이 답하는 질문:**

- 코틀린엔 왜 삼항 연산자 `? :`가 없는가? 그 자리를 무엇이 메우는가?
- `if`는 문(statement)인가 표현식(expression)인가? 둘 다일 수 있는가?
- `when`은 `switch`인가? 왜 아닌가? (M12)
- `when`의 **완전성(exhaustiveness)** 검사는 언제 *강제*되고 언제 *권고*되는가?
- 분기(branch)는 위에서 아래로 평가되는가? 겹치는 조건에서 순서가 결과를 바꾸는가?
- `when`엔 왜 fall-through가 없는가? 그 부재가 어떤 종류의 버그를 원천 차단하는가?
- 대상 있는 `when(x)`와 대상 없는 `when { }`은 내부적으로 무엇이 다른가?
- `when`이 스마트 캐스트와 결합하면 분기 안에서 타입이 어떻게 좁혀지는가?
- Kotlin 2.1의 **가드 조건(`if`)** 은 어떤 표현력의 구멍을 메웠는가?
- `if`/`when` 표현식의 *타입*은 어떻게 계산되는가? 블록의 값은 무엇으로 정해지는가?

---

앞의 [[16 - 스코프 함수와 수신 객체 관용구]]에서 우리는 `let`·`run`·`with`·`apply`·`also`가 결국 "값을 반환하는 작은 스코프"라는 관점을 세웠다. 그 관점의 뿌리에는 코틀린을 관통하는 하나의 설계 결정이 있다. **거의 모든 것이 값을 낳는 표현식이다.** 스코프 함수가 표현식으로 유용했던 것은 언어 자체가 제어 흐름조차 표현식으로 취급하기 때문이다. 이 장은 그 뿌리 — `if`와 `when` — 을 명세와 바이트코드 수준까지 파고든다.

이 장이 다루는 것은 *분기(branching)* 로서의 제어 흐름이다. 즉 조건에 따라 한 갈래를 고르고, 그 갈래가 낳는 값을 전체 표현식의 값으로 삼는 기계. `if`와 `when`, 그리고 그것들이 표현식이기에 자연스럽게 따라오는 개념들 — 블록의 값, 분기 타입의 최소 상계(least upper bound), 완전성 검사, fall-through의 부재, 스마트 캐스트와의 결합, 2.1의 가드 조건 — 이 이 장의 영토다. 반면 *반복(looping)* 으로서의 제어 흐름, 즉 `for`·`while`·`in`·범위(`..`)와 `break`/`continue`의 라벨 규칙은 [[18 - 범위와 진행과 반복]]이 소유한다(이 장에선 `when` 안에서의 `break`/`continue`만 짧게 짚는다). `try`/`catch`가 표현식이라는 사실과 `throw`가 `Nothing` 타입이라는 사실은 [[41 - 예외와 Nothing]]과 [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]]이 소유한다. `when`이 완전성을 통해 만개하는 무대인 봉인 계층(sealed hierarchy)의 세부는 [[30 - sealed 클래스와 대수적 데이터 타입]]이, 열거형은 [[29 - enum 클래스]]가 소유한다.

이 장의 논증 궤적은 이렇다. 먼저 "문 대 표현식"이라는 낡은 이분법을 코틀린이 어떻게 재편했는지 본다. 그 위에서 `if`가 삼항의 부재를 어떻게 메우는지, 블록의 마지막 표현식이 어떻게 값이 되는지를 본다. 그다음 `when`을 두 얼굴(대상 있음/없음)로 해부하고, 그 분기 조건의 문법(상수·범위·`is`·임의 표현식)을 하나씩 판다. 그러고 나서 이 장의 심장인 **완전성**과 **fall-through 부재**를 다루며 M12("`when`은 `switch`다")를 정면으로 무너뜨린다. 마지막으로 스마트 캐스트 결합, 2.1 가드 조건, 그리고 이 모든 것이 JVM 백엔드에서 `tableswitch`/`lookupswitch`/`if-else` 사슬로 내려가는 과정을 본다.

---

## 1. 문과 표현식 — 코틀린이 다시 그은 경계선

### 1.1 두 세계의 정의

전통적인 명령형 언어(C, Java)는 문법을 **문(statement)** 과 **표현식(expression)** 으로 가른다. 표현식은 평가되어 *값을 낳는다*(`3 + 4`, `f(x)`, `a > b`). 문은 *효과를 낳지만 값을 낳지 않는다*(`if (c) { ... }`, `for (...) { ... }`, `return`). Java에서 `if`는 문이다. 그래서 아래는 Java에서 문법 오류다.

```java
// Java — 컴파일 에러: if는 표현식이 아니다
int max = if (a > b) a else b;
```

Java는 이 자리를 메우려고 별도의 **삼항 연산자(ternary operator)** `? :`를 둔다.

```java
// Java — 삼항 연산자로 우회
int max = (a > b) ? a : b;
```

코틀린은 이 이원 구조를 재편했다. 코틀린에도 문과 표현식의 구분은 있지만, **`if`와 `when`과 `try`가 표현식**이다. 즉 값을 낳는다. 그래서 코틀린엔 삼항 연산자가 아예 존재하지 않는다 — 필요가 없기 때문이다.

```kotlin
// Kotlin — if가 표현식이므로 그 값을 바로 대입
val max = if (a > b) a else b   // => a와 b 중 큰 값
```

> **역사 메모**
> 코틀린 설계자들은 삼항 연산자를 "읽기 어렵고 중첩하면 지옥이 되는 특수 문법"으로 보았고, `if`를 표현식으로 만들면 별도 삼항이 불필요하다고 판단했다. `? :`의 부재는 실수가 아니라 의도된 절약이다. 대가로 `if (a > b) a else b`는 삼항보다 몇 글자 길지만, 중첩 시 `if/else if/else` 사슬이 `?:`의 우선순위 함정보다 훨씬 명료하다. Scala·Rust·Swift(부분적)도 같은 길을 택했다.

### 1.2 "문으로 쓰이는 표현식"이라는 이중성

여기서 핵심은, `if`가 표현식이라고 해서 문처럼 못 쓴다는 뜻이 *아니라*는 점이다. 코틀린에서 `if`는 **문맥에 따라 표현식으로도 문으로도** 쓰인다.

```kotlin
// 문처럼 — 반환값을 버리고 부수효과만 취함
if (balance < 0) {
    println("잔액 부족")
}

// 표현식처럼 — 값을 대입에 사용
val grade = if (score >= 90) "A" else "B"
```

이 이중성의 규칙은 단순하다. **결과값이 실제로 쓰이면(대입·인자·반환 등) 표현식으로 취급되고, 그때만 완결성 조건(`if`엔 `else` 필수)이 강제된다.** 값을 버리는 문 위치에서는 `else`가 없어도 된다.

```kotlin
// 문 위치 — else 없어도 OK (값을 안 씀)
if (isReady) start()

// 표현식 위치 — else 없으면 컴파일 에러
val x = if (isReady) 1
// 컴파일 에러: 'if' must have both main and 'else' branches if used as an expression
```

`else`가 필수인 이유는 완전성이다. 표현식은 *반드시* 값을 낳아야 하는데, 조건이 거짓일 때 값이 없으면 표현식이 성립하지 않는다. 이 원리는 `when`의 완전성 검사(5절)로 곧장 확장된다.

### 1.3 문의 타입은 Unit

그렇다면 "문으로 쓰인 `if`"의 타입은 무엇인가? 코틀린엔 값 없는 자리가 없다. 문 위치의 `if`도 값을 낳긴 하는데, 그 값의 타입이 `Unit`이다. `Unit`은 값이 정확히 하나뿐인 실제 타입이며(자세히는 [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]], M10), 그 유일한 값은 `Unit`이라는 객체다. Java의 `void`("값 없음")와 달리 코틀린엔 "값 없음"이 없다 — 대신 "정보 없는 값"이 있다.

```kotlin
val nothing: Unit = if (flag) println("a") else println("b")
// println의 반환 타입이 Unit이므로 이 if 표현식의 타입도 Unit
// nothing == kotlin.Unit
```

`else`가 없는 문 위치 `if`의 타입도 `Unit`이다. 조건이 거짓이면 아무 일도 안 하고 `Unit`을 낳는 것으로 취급된다. 이 관점에서 보면 "`else` 없는 `if`"는 사실 암묵적으로 "`else { }`(빈 블록, 값은 `Unit`)"를 가진 셈이고, 그래서 표현식으로 못 쓰이는 것이다 — 두 분기의 타입이 각각 `T`와 `Unit`이면 전체 타입이 `Any`로 뭉개져 쓸모가 없어지기 때문. 실제로는 컴파일러가 표현식 위치에서 아예 `else`를 요구하는 쪽으로 못 박는다.

---

## 2. if 표현식의 해부

### 2.1 블록의 값 — 마지막 표현식 규칙

`if`의 각 분기는 단일 표현식일 수도, 중괄호 블록일 수도 있다. 블록일 때 **그 블록의 값은 마지막 표현식의 값**이다. 이것이 코틀린 표현식 지향의 심장이다.

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

블록 안에서 마지막이 아닌 표현식들의 값은 버려진다(부수효과만 취한다). 마지막 표현식만이 "블록의 값"으로 위로 전달된다. 만약 블록의 마지막이 표현식이 아니라 선언(`val`/`var`/`fun`)이거나 대입이거나 루프라면, 그 블록의 값은 `Unit`이다.

```kotlin
val r = if (c) {
    doWork()
    val z = 3      // 선언은 값이 아니다 → 블록의 값은 Unit
} else {
    Unit
}
// r: Unit
```

> **명세 정밀**
> 대입(`x = 5`)은 코틀린에서 **표현식이 아니라 문**이며 그 타입은 `Unit`이다. 이는 Java/C와 다른 지점이다. Java에선 `while ((line = read()) != null)`처럼 대입 결과를 조건에 쓸 수 있지만, 코틀린에선 `x = 5`가 값을 낳지 않으므로 `if (x = 5)` 같은 실수가 애초에 컴파일되지 않는다. C의 고전적 함정 `if (x = 0)`("`==`를 쓰려다 `=`을 씀")이 언어 차원에서 봉쇄된다.

### 2.2 분기 타입의 최소 상계(LUB)

`if`가 표현식이면 그 타입은 무엇인가? 두 분기가 서로 다른 타입을 낳을 때, 전체 표현식의 타입은 두 분기 타입의 **최소 상계(least upper bound)**, 즉 둘을 모두 포함하는 가장 좁은 공통 상위 타입이다.

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

이 LUB 계산은 곧 볼 `when`에도 동일하게 적용되며, 실은 코틀린 타입 추론 전반을 관통하는 규칙이다. 분기 중 하나가 `Nothing` 타입(예: `throw` 또는 무한 루프, [[41 - 예외와 Nothing]])이면, `Nothing`은 모든 타입의 하위 타입이므로 LUB 계산에서 사실상 무시된다. 이것이 아래 관용구가 성립하는 이유다.

```kotlin
val port: Int = if (rawPort != null) rawPort else throw IllegalArgumentException("포트 없음")
// throw는 Nothing 타입 → LUB(Int, Nothing) = Int → port: Int
```

`throw`가 값을 낳는 표현식이라는 이 사실(M11)은 코틀린의 널 처리 관용구 `?:`(엘비스, [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]])의 우변에 `throw`나 `return`을 쓸 수 있게 해 주는 바로 그 성질이다.

### 2.3 삼항의 부재가 강제하는 것

삼항 연산자가 없다는 사실은 스타일 문제로 보이지만, 실제로는 몇 가지를 강제한다. 첫째, 조건 분기는 항상 `if`/`else` 키워드로 명시되므로 `?:`(엘비스)와 시각적으로 구분된다 — 코틀린의 `?:`는 삼항이 아니라 널 병합(null-coalescing) 전용이다(M32). 둘째, 중첩 조건은 `else if` 사슬로 쓰게 되어 우선순위 괄호 지옥이 사라진다.

```kotlin
// 중첩 — else if 사슬이 삼항 중첩보다 읽기 쉽다
val grade = if (score >= 90) "A"
            else if (score >= 80) "B"
            else if (score >= 70) "C"
            else "F"
```

셋째, 이런 등급 분기는 곧 볼 대상 없는 `when`으로 더 깔끔히 쓸 수 있어, 실무에선 `if/else if` 사슬이 3개를 넘기면 `when`으로 넘어가는 게 관습이 된다. 즉 `if`와 `when`은 경쟁이 아니라 스펙트럼의 양 끝이다.

### 2.4 if와 when의 선택 기준

`if`와 `when`은 언제 무엇을 쓰는가? 언어가 강제하진 않지만, 관용적으로 다음 경계가 있다.

| 상황 | 권장 | 이유 |
|---|---|---|
| 조건이 하나(참/거짓 이분) | `if/else` | 가장 직접적, `when`은 과함 |
| 한 값에 대한 다지 분기 | `when(x)` | 대상 한 번 명시, 상수/범위/타입 통합 |
| 서로 무관한 여러 조건 사슬 | `when {}` | `if/else if` 사슬을 평평하게 |
| sealed/enum 처리 | `when(x)` (else 없이) | 완전성으로 미래 경우를 컴파일 강제 |
| 널 검사 후 대체값 | `?:`(엘비스) | 널 병합 전용, `if`보다 간결 |

핵심 직관은 이렇다. **분기의 축이 "하나의 대상"이면 `when(x)`, "여러 독립 술어"면 `when {}` 또는 `if`.** 대상이 sealed/enum이면 거의 항상 `when(x)`가 정답인데, 완전성이라는 무기가 거기서만 발동하기 때문이다. 이 무기의 정체가 5절이다.

---

## 3. when의 두 얼굴 — 대상 있음과 대상 없음

`when`은 코틀린 제어 흐름의 주력이다. 그리고 문법적으로 **두 형태**를 가진다. 이 둘을 혼동하면 `when`을 오해하게 된다.

### 3.1 대상 있는 when — `when(subject)`

괄호에 대상(subject)을 두면, 각 분기의 왼쪽은 그 대상과 *비교되는* 것으로 해석된다.

```kotlin
val name = when (dayOfWeek) {
    1 -> "월"
    2 -> "화"
    3, 4, 5 -> "평일"       // 콤마 = OR
    6, 7 -> "주말"
    else -> "알 수 없음"
}
```

여기서 `1 ->`는 "`dayOfWeek == 1`이면"을 뜻한다. 즉 대상 있는 `when`의 분기 조건은 대상과의 **동등성 비교(`==`)** 를 기본으로 하되, 뒤에서 볼 `in`(범위/컬렉션 포함)과 `is`(타입 검사)로 확장된다. 콤마로 나열한 값들은 OR로 묶인다(`3, 4, 5 ->`는 "3이거나 4거나 5면").

대상은 그 자리에서 변수로 **바인딩**할 수도 있다. 이는 대상이 되는 계산 결과를 분기 안에서 재사용할 때 유용하다.

```kotlin
when (val response = fetchStatus()) {
    200 -> render(response)          // response를 분기에서 사용
    in 400..499 -> logClientError(response)
    else -> retry(response)
}
// response의 스코프는 이 when 블록으로 한정됨
```

`when (val x = ...)` 형태는 `x`의 스코프를 `when` 블록 안으로 정확히 가둔다. 밖에서 임시 변수가 새는 것을 막는 작은 캡슐화다.

### 3.2 대상 없는 when — `when { }`

괄호를 아예 생략하면, `when`은 완전히 다른 것이 된다. 이때 각 분기의 왼쪽은 **독립적인 Boolean 표현식**이며, `when`은 위에서 아래로 처음 `true`가 되는 분기를 고른다. 즉 대상 없는 `when`은 `if/else if/else` 사슬과 의미가 같다.

```kotlin
val grade = when {
    score >= 90 -> "A"
    score >= 80 -> "B"
    score >= 70 -> "C"
    else -> "F"
}
```

여기엔 대상이 없으므로 `score >= 90` 같은 각 조건은 그 자체로 `Boolean`이어야 한다. 대상 있는 `when`의 `1 ->`와 달리, 여기서 `90 ->` 같은 상수는 (Boolean이 아니므로) 컴파일 에러다. 이 두 형태의 차이는 아래 표로 못 박아 둔다.

| 구분 | 대상 있는 `when(x)` | 대상 없는 `when {}` |
|---|---|---|
| 분기 왼쪽의 의미 | `x`와 비교되는 값/범위/타입 | 독립 `Boolean` 표현식 |
| `42 ->` | `x == 42` | 컴파일 에러(Boolean 아님) |
| `in 1..9 ->` | `x in 1..9` | 컴파일 에러(Boolean 필요) → `x in 1..9 ->`로 명시 |
| `is String ->` | `x is String` (+ 스마트 캐스트) | `x is String ->`로 명시 |
| 등가 형태 | `switch`에 가까운 형태 | `if/else if` 사슬 |
| 스마트 캐스트 대상 | 대상 `x` 자동 | 조건에 등장한 변수 각각 |

> **흔한 오해**
> "`when`은 `switch`다"(M12)라는 오해는 대개 대상 있는 `when`의 상수 분기만 보고 생기는데, 대상 없는 `when`을 보면 이 오해가 즉시 깨진다. 대상 없는 `when`은 비교할 값이 없고 그냥 조건 사슬이다. `switch`엔 이런 형태가 존재하지 않는다. `when`은 `switch`의 상위집합조차 아니라, 아예 다른 구조물이다(5절에서 정면으로 다룬다).

### 3.3 when이 표현식일 때와 문일 때

`if`와 마찬가지로 `when`도 표현식으로도 문으로도 쓰인다.

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

표현식 위치에서 `when`은 반드시 값을 낳아야 하므로 **완전해야** 한다. 이 완전성 요구가 `when`을 `switch`와 갈라놓는 결정적 지점이며, 5절 전체가 이를 다룬다.

---

## 4. when의 분기 조건 — 상수·범위·타입·임의 표현식

대상 있는 `when`의 진짜 힘은 분기 왼쪽에 올 수 있는 것이 상수만이 아니라는 데 있다. Java `switch`가 (전통적으로) 상수·enum·문자열만 허용한 것과 대비된다.

### 4.1 상수 동등성과 콤마 OR

가장 기본은 동등성 비교다. 대상 있는 `when`의 `value ->`는 `subject == value`를 뜻하며, 여기서 `==`는 구조적 동등(`equals`)이다([[10 - 불리언과 동등성과 동일성]], M6). 따라서 상수뿐 아니라 임의의 값과도 비교된다.

```kotlin
val threshold = computeThreshold()
val over = when (amount) {
    0 -> "없음"
    threshold -> "임계값 정확히 도달"   // 상수 아닌 값과도 비교 가능
    else -> "그 외"
}
```

Java `switch`의 `case`는 컴파일 타임 상수여야 하지만, 코틀린 `when`의 분기 값은 **런타임 값**이어도 된다. `threshold ->`가 성립하는 것이 그 증거다. 이는 `when`이 컴파일러가 반드시 점프 테이블로 내리는 구조가 아니라, 필요하면 `if/else` 사슬로도 내려가는 유연한 구조이기 때문이다(9절).

콤마는 OR을 뜻한다:

```kotlin
when (c) {
    'a', 'e', 'i', 'o', 'u' -> "모음"
    else -> "자음"
}
```

### 4.2 범위 포함 — `in`

`in`을 쓰면 대상이 범위나 컬렉션에 속하는지 검사한다. 이는 `contains` 관례([[19 - 연산자 오버로딩과 관례]])로 위임되며, 범위([[18 - 범위와 진행과 반복]])와 결합해 매우 표현력이 높다.

```kotlin
val tier = when (score) {
    in 90..100 -> "S"
    in 70..<90 -> "A"        // ..< = rangeUntil, 90 제외
    in 50..<70 -> "B"
    !in 0..100 -> "범위 오류"  // 부정도 가능
    else -> "F"
}
```

`in`은 `IntRange`뿐 아니라 임의의 `Iterable`·`Set`·문자열 등 `contains`를 가진 무엇과도 동작한다.

```kotlin
val vowels = setOf('a', 'e', 'i', 'o', 'u')
when (ch) {
    in vowels -> "모음"
    in 'a'..'z' -> "그 외 소문자"
    else -> "비-소문자"
}
```

Java `switch`로는 표현할 수 없는 이 "범위 분기"가 코틀린 `when`의 실전 킬러 기능이다.

### 4.3 타입 검사 — `is` (그리고 스마트 캐스트)

`is`를 쓰면 대상의 런타임 타입을 검사한다. 그리고 그 분기 안에서 대상은 해당 타입으로 **스마트 캐스트**된다(7절에서 자세히).

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

`is Circle ->` 분기 안에서 `s`는 별도 캐스트 없이 `Circle`의 멤버 `r`에 접근한다. `is`는 부정형 `!is`도 지원한다.

### 4.4 임의 표현식 분기와 콤마 안의 이질성

대상 있는 `when`에서 각 분기의 콤마 목록은 서로 다른 종류(상수·범위·타입)를 섞을 수 있다.

```kotlin
when (x) {
    0, in 100..200, is String -> handle(x)  // 세 종류를 OR로 혼합
    else -> other()
}
```

다만 이렇게 섞으면 스마트 캐스트가 약해진다. `0, is String ->` 분기 안에서는 `x`가 `String`이라고 확정할 수 없으므로(0일 수도 있으니) `String`으로 스마트 캐스트되지 않는다. 이것이 다음에 볼 가드 조건(8절)이 필요해진 배경 중 하나다.

> **성능 주의**
> 대상 있는 `when`에서 분기가 전부 `Int`/enum 상수이고 값이 조밀하면 JVM 백엔드는 `tableswitch`(O(1) 점프 테이블)로 컴파일한다. 그러나 하나라도 `in`·`is`·런타임 값·비-정수 상수가 끼면 그 분기부터는 `if/else` 순차 비교로 내려간다(9절). 성능이 극단적으로 중요한 뜨거운 경로에서 이 차이를 알고 싶다면, 분기 종류를 섞지 말고 상수 전용 `when`을 유지하는 편이 유리할 수 있다(단, 대부분의 코드에서 이 차이는 무의미하다).

---

## 5. 완전성(exhaustiveness)과 M12 — when은 switch가 아니다

이 절이 이 장의 심장이다. **`when`은 `switch`가 아니다(M12).** 표면적 유사성(값에 따라 갈라진다)에 속아 둘을 같다고 여기는 순간, 코틀린 타입 시스템이 `when`에 부여한 가장 강력한 안전장치를 놓치게 된다. 그 안전장치가 완전성(exhaustiveness)이다.

### 5.1 표현식 when은 완전해야 한다

`when`이 표현식으로 쓰이면(값이 대입·반환·인자로 실제 사용되면) 반드시 **모든 경우를 덮어야** 한다. 그러지 않으면 컴파일 에러다.

```kotlin
val label = when (n) {
    1 -> "하나"
    2 -> "둘"
}
// 컴파일 에러: 'when' expression must be exhaustive, add necessary 'else' branch
```

`n`이 `Int`이면 가능한 값이 무수히 많으므로, 표현식 `when`을 완전하게 만드는 유일한 방법은 `else`다. `else`는 "나머지 전부"를 덮어 완전성을 보장한다.

```kotlin
val label = when (n) {
    1 -> "하나"
    2 -> "둘"
    else -> "그 밖"     // 이제 완전 → 표현식으로 OK
}
```

이 요구는 2.2에서 본 "표현식 `if`엔 `else`가 필수"와 정확히 같은 원리다. 표현식은 값을 낳아야 하고, 어떤 입력에도 값이 정의돼야 하므로 모든 경우가 덮여야 한다.

### 5.2 enum과 sealed — else 없이 완전해지는 마법

`else`가 완전성의 *유일한* 길이라면 `when`은 그저 편한 `switch`에 그쳤을 것이다. 진짜 힘은 **대상의 타입이 유한한 경우의 집합을 가질 때** 나온다. 그럴 땐 모든 경우를 명시하면 `else` 없이도 완전해진다.

**enum:** 열거형([[29 - enum 클래스]])의 모든 상수를 덮으면 완전하다.

```kotlin
enum class Direction { NORTH, SOUTH, EAST, WEST }

fun dx(d: Direction): Int = when (d) {   // else 없음 — 그런데 완전!
    Direction.NORTH -> 0
    Direction.SOUTH -> 0
    Direction.EAST -> 1
    Direction.WEST -> -1
}
```

컴파일러는 `Direction`이 정확히 네 상수뿐임을 알고, 네 분기가 모두 있으니 완전하다고 판단한다. **`else`가 없다.** 여기가 핵심이다. 만약 나중에 `Direction`에 `UP`을 추가하면, 이 `when`은 **컴파일 에러**가 된다.

```kotlin
enum class Direction { NORTH, SOUTH, EAST, WEST, UP }  // 상수 추가
// dx()의 when: 컴파일 에러 — 'UP' 분기 누락, must be exhaustive
```

이것이 `switch`가 결코 줄 수 없는 것이다. Java `switch`에서 `default`를 빼고 enum 케이스를 나열해도, 새 상수가 추가되면 컴파일러는 경고조차 안 하고 런타임에 조용히 아무 케이스도 안 타는(혹은 잘못 타는) 버그를 낳는다. 코틀린은 `else`를 뺀 완전한 `when`을 통해, "새로운 경우가 생기면 이 분기 로직을 다시 검토하라"는 계약을 **컴파일 타임에** 강제한다.

**sealed:** 봉인 클래스/인터페이스([[30 - sealed 클래스와 대수적 데이터 타입]])의 모든 하위 타입을 덮으면 완전하다.

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

`sealed`는 하위 타입의 집합이 컴파일 타임에 **닫혀** 있으므로, 컴파일러가 "이게 전부"라고 확신할 수 있다. 이 sealed + 완전한 `when` 조합이 코틀린에서 대수적 데이터 타입(ADT)과 패턴 매칭을 흉내 내는 정석이다(자세히는 30장). 새 하위 타입을 추가하면 모든 완전한 `when`이 컴파일 에러로 잡히므로, "이 새 경우를 어디서 처리해야 하지?"를 컴파일러가 목록으로 알려준다.

**Boolean과 nullable:** `Boolean`은 `true`/`false` 둘뿐이라 둘 다 덮으면 완전하다. nullable enum/sealed는 `null` 분기까지 덮어야 완전하다.

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

### 5.3 문 위치 when의 완전성 — 경고와 그 역사

여기서 미묘한 지점이 있다. 지금까지의 완전성 강제는 **표현식 위치**의 이야기다. `when`을 값 버리는 **문 위치**로 쓰면 어떻게 될까?

```kotlin
fun handle(d: Direction) {
    when (d) {          // 문 위치 — 값 안 씀
        Direction.NORTH -> goNorth()
        Direction.SOUTH -> goSouth()
        // EAST, WEST 누락
    }
}
```

과거(Kotlin 1.x 초기)엔 이런 문 위치의 비완전 `when`이 아무 진단 없이 통과했다. 이는 위험했다 — enum에 상수를 추가해도 문 위치 `when`은 조용히 지나쳤기 때문이다. 그래서 언어는 진화했다.

> **역사 메모**
> Kotlin은 sealed/enum을 대상으로 하는 **문 위치**의 비완전 `when`에 대해, 표현식 위치와 동일하게 완전성을 요구하는 방향으로 점진적으로 이동해 왔다. 초기엔 진단이 없었고, 이후 경고(warning)로 격상되었으며, K2/2.x 세대에서는 enum·sealed·Boolean 같은 "닫힌" 대상에 대한 문 위치 비완전 `when`을 강한 경고로 잡는다(빌드 설정에 따라 에러로 승격 가능). 정확한 진단 강도(경고 대 에러)는 컴파일러 버전과 언어 버전 설정에 따라 다르므로, "완전한 `when`을 쓰라"는 규칙으로 통일하는 편이 안전하다. 표현식 위치에서는 처음부터 지금까지 항상 **에러**다.

실무 규칙은 단순하다: **sealed/enum을 `when`으로 처리할 땐 `else`를 넣지 말고 모든 경우를 명시하라.** 그래야 새 경우가 추가될 때 컴파일러가 잡아 준다. `else`를 넣으면 이 안전망이 사라진다 — 새 경우가 조용히 `else`로 흡수돼 버린다.

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

완전성을 조금 형식적으로 보면, 대상의 타입 $T$가 낳는 값의 집합을 $V(T)$라 할 때, `when`의 분기 조건들이 덮는 값 집합의 합집합이 $V(T)$ 전체를 포함해야 한다는 조건이다.

$$
\bigcup_{i} \text{cover}(\text{branch}_i) \;\supseteq\; V(T)
$$

- `T = Int`: $V(T)$가 사실상 무한 → `else` 없이는 덮을 수 없음.
- `T = Boolean`: $V(T) = \{\texttt{true}, \texttt{false}\}$ → 두 분기로 덮임.
- `T = Direction`(enum): $V(T) = \{\text{NORTH}, \text{SOUTH}, \text{EAST}, \text{WEST}\}$ → 네 분기로 덮임.
- `T = Json`(sealed): $V(T)$ = 세 하위 타입의 값들 → 세 `is`/`object` 분기로 덮임.

컴파일러의 완전성 검사는 이 포함 관계를 *구문적으로* 판정할 수 있는 경우(enum 전체 나열, sealed 전체 나열, Boolean 양쪽, `else`)에만 완전으로 인정한다. 예컨대 `in 0..100`과 `in 101..200`으로 `Int`를 나눠도 컴파일러는 "그럼 나머지 Int는?"이라고 물으며 완전으로 인정하지 않는다 — 범위 합집합이 전체 `Int`를 덮는지 판정하는 것은 일반적으로 하지 않기 때문이다.

### 5.5 switch와 when — 표로 못 박는 차이

M12를 한 표로 정리한다. `switch`(C/Java 전통형)와 코틀린 `when`은 표면만 닮았을 뿐 계약이 다르다.

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

Java 21의 스위치 표현식·패턴 매칭(`switch` expression, `case X x when ...`)은 이 표의 여러 칸에서 코틀린 `when`에 근접했다 — 표현식화, 패턴, 가드, 완전성이 Java에도 들어왔다. 그러나 코틀린 `when`은 이 능력을 언어 초기부터 통합된 형태로 제공했고, 대상 없는 조건 사슬(`when {}`)까지 하나의 키워드로 흡수한다는 점에서 여전히 더 넓은 구조물이다. 요컨대 "`when`은 `switch`다"라는 M12는 방향이 거꾸로다 — 오히려 최신 `switch`가 `when`을 닮아 가는 중이다.

---

## 6. fall-through의 부재와 분기 평가 모델

### 6.1 fall-through가 없다

C/Java `switch`의 악명 높은 함정은 **fall-through**다. `break`를 빠뜨리면 다음 `case`로 실행이 흘러내린다.

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

이 fall-through는 역사적 실수로 널리 인정되며, 무수한 버그의 근원이었다. 코틀린 `when`엔 **fall-through가 아예 없다.** 매칭된 분기 하나만 실행하고 `when`이 끝난다. 그래서 `break`가 필요 없고, 존재하지도 않는다.

```kotlin
when (day) {
    1 -> println("월")   // day==1이면 이것만 실행하고 when 종료
    2 -> println("화")
    else -> println("그 외")
}
// day==1 → "월"만 출력
```

"여러 경우가 같은 동작을 하길" 원하면 fall-through 대신 콤마 OR을 쓴다(`1, 2 -> ...`). 이 설계는 fall-through의 표현력(여러 case 공유)은 콤마로 보존하면서, fall-through의 위험(break 누락)은 원천 제거한다.

> **흔한 오해**
> "`when`에서 `break`로 빠져나온다"는 것은 오해다. `when`엔 나갈 `break`가 없다 — 매칭된 분기가 끝나면 자동으로 나간다. `when` *안에* 쓴 `break`/`continue`는 `when`이 아니라 이를 감싸는 **루프**를 제어한다(6.3).

### 6.2 위에서 아래로 — 첫 매칭 승리

`when`의 분기는 **작성된 순서대로 위에서 아래로** 평가되고, 처음으로 참이 되는 분기가 선택된다(first-match-wins). 이후 분기는 평가조차 되지 않는다. 조건이 서로 겹칠 때 순서가 결과를 바꾼다.

```kotlin
val category = when (n) {
    in 0..100 -> "작음"      // n=50이면 여기서 멈춤
    in 0..1000 -> "중간"     // 도달 못 함(위가 먼저 매칭)
    else -> "큼"
}
// n=50 → "작음"
```

만약 좁은 범위를 아래에 두면 도달 불가(unreachable) 분기가 생긴다. 위 예에서 `in 0..1000`은 `n`이 0..100이면 이미 위에서 잡히고, 101..1000이면 여기서 잡히므로 그 자체론 도달하지만, 논리적으로 순서를 뒤집으면 문제가 된다.

```kotlin
val category = when (n) {
    in 0..1000 -> "중간"     // 0..100도 여기서 다 잡힘
    in 0..100 -> "작음"      // 도달 불가 — 죽은 코드
    else -> "큼"
}
```

이런 "죽은 분기"는 논리 버그의 징후다. 대상 있는 `when`에서 상수만 쓰면 값이 겹칠 수 없어 안전하지만, `in`/`is`/조건을 섞으면 순서 의존이 생기니 좁은 조건을 위에, 넓은 조건을 아래에 두는 습관이 필요하다.

이 first-match 모델은 대상 없는 `when`에서 더 두드러진다 — 그건 본질적으로 `if/else if` 사슬이고, 조건들이 배타적이지 않을 수 있기 때문이다.

```kotlin
val sign = when {           // score가 여러 조건에 걸릴 수 있음
    score > 0 -> "양수"
    score < 0 -> "음수"
    else -> "영"
}
```

### 6.3 when 안의 break/continue

`when` 안에 `break`/`continue`를 쓰면 `when`이 아니라 이를 감싸는 루프를 제어한다.

```kotlin
for (x in items) {
    when (x.type) {
        Type.SKIP -> continue      // 다음 반복으로 (when이 아니라 for)
        Type.STOP -> break         // for 종료
        else -> process(x)
    }
}
```

> **역사 메모**
> Kotlin 초기(1.4 이전)엔 `when` 안에서 라벨 없는 `break`/`continue`가 금지되어, 루프를 제어하려면 명시적 라벨(`loop@ for ...`, `break@loop`)이 필요했다. `when`의 분기 종료를 뜻하는 것으로 오해될 여지를 없애려는 보수적 선택이었다. Kotlin 1.4부터 이 제약이 풀려, `when` 안의 라벨 없는 `break`/`continue`는 가장 가까운 감싸는 루프를 가리키게 되었다. 오늘날엔 위 코드가 자연스럽게 동작한다.

---

## 7. 스마트 캐스트와 when의 결합

### 7.1 대상 있는 when + is → 자동 스마트 캐스트

`is` 분기가 대상을 스마트 캐스트한다는 것을 4.3에서 예고했다. 이는 코틀린 널 안전성/타입 좁히기([[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]])의 흐름 분석(flow analysis)이 `when` 분기 안으로 스며드는 것이다.

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

`is KeyPress ->` 분기 안에서 `e`는 `KeyPress` 타입으로 확정되어 `key`에 직접 접근한다. sealed와 결합하면 이것이 **완전한 패턴 매칭**처럼 읽힌다 — 각 하위 타입을 나열하고, 그 안에서 구조에 접근하며, 새 하위 타입이 생기면 컴파일러가 잡아 준다(5.2).

### 7.2 대상 없는 when의 스마트 캐스트

대상 없는 `when`에서도 각 조건에 등장한 변수가 그 분기 안에서 좁혀진다.

```kotlin
fun classify(x: Any?): String = when {
    x == null -> "널"
    x is Int && x > 0 -> "양의 정수"     // x: Int로 좁혀짐 → x > 0 유효
    x is String && x.isNotEmpty() -> "비어있지 않은 문자열"  // x: String
    else -> "그 외"
}
```

`x is Int && x > 0`에서 `&&`의 오른쪽은 왼쪽이 참일 때만 평가되므로, 그 지점에서 `x`는 이미 `Int`로 스마트 캐스트되어 `x > 0`이 타입 검사를 통과한다. 이는 `if`의 스마트 캐스트와 완전히 동일한 흐름 분석이며, `when`이 특별해서가 아니라 언어 전역의 흐름 민감 타입 좁히기가 적용되는 것이다.

### 7.3 스마트 캐스트가 안 되는 경우

스마트 캐스트는 만능이 아니다(M13). `when` 대상이 `var`이고 커스텀 게터를 갖거나, 다른 모듈의 `open`/프로퍼티거나, 검사와 사용 사이에 값이 바뀔 수 있으면 스마트 캐스트가 거부된다.

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

이럴 땐 `when (val c = box.content)`로 지역 `val`에 바인딩하면 그 지역 변수는 안정적이라 스마트 캐스트된다. 이것이 3.1의 `when (val x = ...)` 바인딩이 실전에서 자주 쓰이는 이유다.

```kotlin
fun check(box: Box) {
    when (val c = box.content) {   // 지역 val에 고정
        is String -> println(c.length)  // c는 스마트 캐스트됨 → OK
        else -> {}
    }
}
```

---

## 8. 가드 조건 — Kotlin 2.1이 메운 구멍

### 8.1 표현력의 구멍

7절까지의 `when`엔 한 가지 답답한 지점이 있었다. 대상 있는 `when`에서 "타입이 X이면서 *동시에* 어떤 추가 조건을 만족"하는 분기를 자연스럽게 쓸 수 없었다. `is X` 분기 안에서 추가 조건으로 다시 분기하거나, 대상 없는 `when`으로 바꿔 스마트 캐스트를 수동으로 해야 했다.

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

Java 21의 패턴 매칭엔 `case Dog d when d.age < 1 ->` 같은 가드가 있는데, 코틀린엔 오랫동안 대응물이 없었다.

### 8.2 when 가드 조건 (2.1, 미리보기)

**Kotlin 2.1**에서 대상 있는 `when`에 **가드 조건(guard condition)** 이 미리보기(preview) 기능으로 도입되었다. 아직 정식(stable)이 아니라 옵트인(컴파일러 플래그 `-Xwhen-guards`)이 있어야 켜지는 실험적 기능이다. 분기 조건 뒤에 `if <불리언 표현식>`을 붙여, 대상 매칭과 추가 조건을 한 줄에 결합한다.

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

여기서 핵심은, `is Dog if a.age < 1` 안에서 `a`가 이미 `Dog`로 스마트 캐스트된 상태이므로 가드 `a.age`가 유효하다는 것이다. 가드는 대상 매칭(타입/값/범위)이 성립한 *뒤에* 평가되어, 둘 다 참일 때만 분기가 선택된다. 매칭은 됐지만 가드가 거짓이면 다음 분기로 넘어간다(위 예에서 `is Dog if a.age < 1`이 거짓이면 `is Dog ->`로).

> **명세 정밀**
> 가드 조건 `if`는 **대상 있는 `when`에서만** 쓸 수 있다(대상이 없으면 애초에 각 분기가 Boolean이라 가드가 불필요). 또한 한 분기에서 콤마 목록과 가드를 자유롭게 섞을 수는 없다 — 콤마로 여러 조건을 나열한 분기에 가드를 붙이려면 괄호로 묶는 등 명확히 해야 하며, 기본적으로 가드는 단일 조건 뒤에 붙는 형태를 상정한다. `else` 분기엔 가드를 붙이지 않는다(`else`는 무조건 매칭이므로). 정확한 문법 조합 규칙은 2.1 릴리스 노트/명세를 따른다.

### 8.3 가드가 완전성에 미치는 영향

가드 조건은 완전성 판정에 미묘한 영향을 준다. 컴파일러는 가드가 붙은 분기를 "이 경우를 *조건부로만* 덮는다"고 보므로, 가드 분기만으로는 완전성을 채울 수 없다. 위 `grade` 예에서 `is Dog if a.age < 1`만 있고 무가드 `is Dog ->`가 없다면, "나이 ≥ 1인 Dog"가 안 덮이므로 sealed 대상이라도 완전으로 인정되지 않는다. 따라서 가드 분기 뒤에는 대개 같은 타입의 무가드 분기나 `else`가 필요하다.

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

## 9. 컴파일 — when이 바이트코드로 내려가는 길

### 9.1 세 가지 하강 전략 (JVM 백엔드)

`when`은 단일한 바이트코드 형태로 컴파일되지 않는다. JVM 백엔드에서 컴파일러는 분기의 성격에 따라 **세 가지 전략** 중 하나를 고른다. (이하는 JVM 백엔드 기준이며, Native/JS/Wasm 백엔드는 각자의 저수준 형태로 내린다.)

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

`tableswitch`와 `lookupswitch`는 JVM 바이트코드 명령이다. 전자는 연속된 정수 키에 대한 배열 인덱싱 점프(상수 시간), 후자는 정렬된 (키, 오프셋) 쌍에 대한 이진 탐색이다. Java `switch`도 정확히 이 두 명령으로 컴파일되므로, **상수만 쓴 `when`은 `switch`와 동일한 기계어 효율**을 낸다. 차이는 표현력과 안전성(완전성)에 있지 성능에 있지 않다.

### 9.2 enum when의 실제 형태

enum 대상 `when`은 흥미로운 우회를 거친다. enum 상수는 컴파일 타임 정수 상수가 아니므로(런타임 객체다), 컴파일러는 `ordinal` 값이 아니라 별도의 **매핑 배열**(`$VALUES`에 기반한 `$EnumSwitchMapping$`)을 생성해 각 상수를 조밀한 정수로 대응시킨 뒤 `tableswitch`를 쓴다. 이는 Java `switch (enumValue)`가 하는 것과 같은 기법이며, enum에 상수를 추가·재배치해도 다른 클래스의 스위치가 깨지지 않도록 하는 이진 호환성 장치다.

```text
enum when (개념적 하강, JVM 백엔드)
  d: Direction
    ↓ mapping 배열로 ordinal → 조밀 정수
  when(d) { NORTH-> ...; SOUTH-> ...; ... }
    ↓
  tableswitch on mappingArray[d.ordinal()]
```

이 세부는 백엔드 구현 사항이며 코드 의미엔 영향이 없다. 다만 "enum `when`이 `ordinal` 순서에 의존한다"는 오해를 방지하는 데 도움이 된다 — 의미는 상수 동일성에 기반하지, ordinal 정수에 직접 기반하지 않는다.

### 9.3 String when — 해시 후 equals

문자열 대상 `when`은 `hashCode()`로 1차 `lookupswitch`를 한 뒤, 해시 충돌을 대비해 각 후보에 대해 `equals()`로 2차 확인한다. 이 역시 Java의 `switch (string)`과 동일한 컴파일 전략이다.

```kotlin
val kind = when (mime) {
    "image/png", "image/jpeg" -> "이미지"
    "text/plain" -> "텍스트"
    else -> "기타"
}
```

이 코드는 `mime.hashCode()`로 후보를 좁히고 `equals`로 확정하는 형태로 내려간다. 이는 `String`의 `==`(구조적 동등, `equals`)와 일관된다 — 문자열 `when`은 참조가 아니라 내용으로 비교한다.

### 9.4 if-else 사슬로 내려가는 경우

`in`·`is`·런타임 값·비교가 섞인 `when`은 점프 테이블로 만들 수 없으니 순차 `if-else` 사슬이 된다. 이때 6.2의 first-match 순서 의존성이 바이트코드에 그대로 반영된다 — 위 분기부터 차례로 조건을 검사하고, 처음 참인 지점에서 점프한다.

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

이 관점에서 대상 없는 `when`은 처음부터 순수한 `if-else` 사슬이므로, 항상 순차 비교로 내려간다. 성능이 극도로 중요한 뜨거운 루프에서 조밀한 정수 분기를 원한다면, `in`/`is`를 피하고 상수 대상 `when`을 유지하면 `tableswitch`의 O(1)을 얻을 수 있다. 그러나 반복하건대, 절대다수의 코드에서 이 차이는 측정되지 않을 만큼 작고, 가독성과 안전성(완전성)이 우선이다.

> **성능 주의**
> `when`이 `tableswitch`가 되느냐 `if-else` 사슬이 되느냐를 육안으로 확인하려면, IntelliJ IDEA의 "Show Kotlin Bytecode → Decompile"로 생성 바이트코드를 보면 된다([[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]] 참고). "라이브러리·컴파일러 문서를 믿지 말고 표준·산출물을 직접 확인하라"는 원칙에 따라, 성능이 진짜 문제라면 추측 대신 디컴파일과 벤치마크로 검증하라.

---

## 10. 관용구·함정·안티패턴

### 10.1 값을 반환하는 when — 함수 본문으로

`when`이 표현식이므로, 단일 표현식 함수([[12 - 함수 - 인자와 vararg와 지역 함수와 꼬리 재귀]])의 본문으로 곧장 쓰는 것이 가장 흔한 관용구다.

```kotlin
fun httpMessage(code: Int): String = when (code) {
    in 200..299 -> "성공"
    in 300..399 -> "리다이렉트"
    in 400..499 -> "클라이언트 오류"
    in 500..599 -> "서버 오류"
    else -> "알 수 없음"
}
```

`= when { ... }`로 함수 전체가 하나의 표현식이 되어, 중간 `return`이 사라지고 완전성이 강제된다(표현식 위치이므로 `else`나 완전한 분기가 필수).

### 10.2 when 대상 바인딩으로 계산 결과 재사용

3.1과 7.3에서 본 `when (val x = ...)`는 계산을 한 번만 하고 분기와 분기 본문에서 재사용하며 스코프를 가두는 삼중 이득을 준다.

```kotlin
fun handle(raw: String): Result = when (val parsed = parse(raw)) {
    is Ok -> use(parsed.value)      // parse 한 번, 여기서 재사용
    is Err -> report(parsed.reason)
}
```

### 10.3 함정 1 — else가 미래를 삼킨다

5.3에서 강조한 안티패턴을 다시 못 박는다. sealed/enum `when`에 습관적으로 `else`를 붙이면, 새 경우가 추가돼도 컴파일러가 경고하지 않고 조용히 `else`로 흡수한다. **sealed/enum엔 `else`를 피하고 모든 경우를 명시하라.** `else`는 대상이 `Int`·`String` 등 열린 타입일 때만 쓴다.

### 10.4 함정 2 — 순서 의존을 잊는다

6.2의 first-match 규칙 때문에, 겹치는 조건에서 넓은 것을 위에 두면 아래가 죽는다. 이는 컴파일 에러가 아니라 논리 버그다.

```kotlin
// 버그: 넓은 조건이 위 → 아래 도달 불가
when {
    n >= 0 -> "음이 아님"
    n > 100 -> "큼"        // 도달 불가 — n>100이면 이미 n>=0에서 잡힘
    else -> "음수"
}
```

### 10.5 함정 3 — 표현식/문 혼동으로 인한 타입 뭉개짐

표현식 `when`의 어떤 분기가 값을, 다른 분기가 `Unit`을 낳으면 전체 타입이 `Any`로 뭉개져 의도치 않은 결과가 된다.

```kotlin
val x: Any = when (n) {
    1 -> "하나"          // String
    2 -> println("둘")   // Unit! (println 반환값)
    else -> "그 외"
}
// x의 타입: Any (String과 Unit의 LUB)
```

분기 중 하나에서 실수로 `println`(반환 `Unit`)이 마지막 표현식이 되면 이런 일이 생긴다. 표현식 `when`의 모든 분기가 의도한 타입을 낳는지 확인해야 한다.

### 10.6 함정 4 — when(true)/when(false) 오용

간혹 대상 없는 `when` 대신 `when (true) { cond -> ... }`를 쓰는 코드를 본다. 이는 대상 없는 `when { cond -> ... }`와 동작이 같지만 불필요하게 장황하고 의도를 흐린다. 조건 사슬엔 대상 없는 `when {}`를 쓰는 게 정석이다.

```kotlin
// 안티패턴
when (true) { score >= 90 -> "A"; else -> "F" }
// 정석
when { score >= 90 -> "A"; else -> "F" }
```

### 10.7 관용구 — sealed + 완전 when으로 상태 기계

sealed 계층 + 완전한 `when`은 상태 기계나 표현식 트리 평가의 정석이다(30장에서 심화). 각 상태/노드를 하위 타입으로 두고 `when`으로 전이·평가를 표현하면, 새 상태 추가 시 모든 처리 지점을 컴파일러가 짚어 준다.

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

이것이 `if`와 `when`이 단순한 문법 편의를 넘어, 코틀린에서 안전한 프로그램 설계의 뼈대가 되는 지점이다.

### 10.8 관용구 — when 분기 안의 블록과 조기 종료

각 `when` 분기의 오른쪽은 단일 표현식이거나 블록이다. 블록이면 2.1의 "마지막 표현식이 값" 규칙이 그대로 적용되어, 분기 안에서 지역 계산을 하고 마지막 값을 분기 값으로 낳을 수 있다.

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

분기 안에서 `return`으로 함수 자체를 조기 종료할 수도 있다. `return`은 `Nothing` 타입이라 이 분기가 값을 낳지 않아도 표현식 `when`의 타입 계산을 깨지 않는다(2.2의 LUB 논리).

```kotlin
fun price(tier: String, amount: Int): Int = when (tier) {
    "vip" -> amount / 2
    "banned" -> return 0            // 함수 조기 종료 — Nothing 타입
    else -> amount
}
```

### 10.9 함정 5 — 대상 있는 when에서 == 오해

대상 있는 `when`의 `value ->`는 `subject == value`(구조적 동등)이지 `===`(참조 동일)가 아니다(M6, [[10 - 불리언과 동등성과 동일성]]). 따라서 값 객체(예: 같은 내용의 문자열·`data class`)는 참조가 달라도 매칭된다. 반대로 참조 동일성이 필요한 드문 경우엔 `when`으로 표현할 수 없고 대상 없는 `when {}`에서 `===`를 직접 써야 한다.

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

---

지금까지 우리는 `if`가 삼항의 부재를 어떻게 표현식성으로 메우는지(1~2절), `when`이 대상 있음/없음의 두 얼굴로 `switch`와 조건 사슬을 하나의 구조에 통합하는지(3~4절), 그리고 완전성과 fall-through 부재가 어떻게 `when`을 `switch`와 근본적으로 갈라놓는지(5~6절)를 보았다. 스마트 캐스트 결합(7절)과 2.1 가드 조건(8절)은 `when`을 사실상 패턴 매칭으로 끌어올렸고, 바이트코드 하강(9절)은 그 모든 표현력이 상수 분기에서 `switch`와 동일한 효율로 내려감을 보여 주었다. 처음의 질문들 — 왜 삼항이 없는가, `if`는 문인가 표현식인가, `when`은 `switch`인가, 완전성은 언제 강제되는가 — 은 이제 하나의 답으로 수렴한다. **코틀린은 제어 흐름을 값을 낳는 표현식으로 재정의했고, 그 결과 `if`와 `when`은 분기를 고르는 문법이자 값을 조립하는 문법이며, 타입 시스템(LUB·완전성·스마트 캐스트)과 한 몸으로 맞물려 컴파일 타임에 버그를 잡는 안전장치가 되었다.** 이 표현식성의 물줄기는 다음 장에서 반복과 범위로, 그리고 이후 sealed·예외·패턴으로 계속 흐른다.

## 핵심 요약

- **코틀린엔 삼항 연산자 `? :`가 없다 — `if`가 표현식이기 때문이다.** `val max = if (a > b) a else b`가 그 자리를 메우며, 표현식 위치에선 `else`가 필수다(값을 반드시 낳아야 하므로).
- **`if`와 `when`은 문맥에 따라 표현식으로도 문으로도 쓰인다.** 값이 실제 쓰이면 표현식(완결성 강제), 버려지면 문(타입은 `Unit`, M10 참조).
- **블록의 값은 마지막 표현식의 값이다.** 선언·대입·루프가 마지막이면 블록의 값은 `Unit`. 대입은 코틀린에서 표현식이 아니라 `Unit` 타입의 문이라, C의 `if (x = 0)` 함정이 원천 봉쇄된다.
- **`if`/`when` 표현식의 타입은 분기 타입들의 최소 상계(LUB)다.** `throw`·무한 루프는 `Nothing` 타입(M11)이라 LUB에서 무시되어, `?:`/`when` 우변에 `throw`/`return`을 놓는 관용구를 성립시킨다.
- **`when`은 `switch`가 아니다(M12).** 대상 없는 `when {}`는 조건 사슬이고, 대상 있는 `when(x)`는 상수뿐 아니라 범위(`in`)·타입(`is`)·런타임 값과도 비교하며, fall-through가 없고 완전성 검사가 있다 — `switch`엔 이 중 어느 것도 없다.
- **표현식 `when`은 반드시 완전(exhaustive)해야 한다.** `Int`는 `else`로만 완전해지지만, enum·sealed·Boolean·nullable은 모든 경우를 명시하면 `else` 없이 완전해진다.
- **sealed/enum `when`엔 `else`를 피하고 모든 경우를 명시하라.** 그래야 새 경우가 추가될 때 컴파일러가 컴파일 에러로 짚어 준다. `else`를 붙이면 미래의 새 경우가 조용히 흡수되어 버그가 숨는다. 문 위치의 비완전 `when`도 K2에서 강한 경고로 잡힌다(버전·설정에 따라 에러 승격).
- **`when`엔 fall-through가 없다.** 매칭된 한 분기만 실행하고 종료하므로 `break`가 불필요하고 존재하지 않는다. 여러 경우 공유는 콤마 OR(`1, 2 ->`)로 표현한다. `when` 안의 `break`/`continue`는 감싸는 루프를 제어한다(1.4+).
- **분기는 위에서 아래로 평가되고 첫 매칭이 이긴다(first-match-wins).** 겹치는 조건에서 넓은 것을 위에 두면 아래가 도달 불가한 죽은 코드가 된다 — 컴파일 에러가 아닌 논리 버그다.
- **`when`은 스마트 캐스트와 결합한다(M13 참조).** `is` 분기 안에서 대상이 자동으로 좁혀지고, 대상 없는 `when`에선 `&&`로 연결된 조건이 흐름 분석으로 좁혀진다. `var`·커스텀 게터 등은 스마트 캐스트 불가라 `when (val x = ...)` 바인딩이 정석 해법이다.
- **Kotlin 2.1의 가드 조건(`if`)이 표현력의 구멍을 메웠다.** `is Dog if a.age < 1 ->`처럼 타입 매칭과 추가 조건을 한 분기에 결합하며(대상 있는 `when` 전용), 가드 분기만으론 완전성을 채우지 못해 뒤에 무가드 분기나 `else`가 필요하다.
- **JVM 백엔드에서 상수 분기 `when`은 `switch`와 동일하게 `tableswitch`/`lookupswitch`로 내려간다.** `in`/`is`/런타임 값이 섞이면 `if-else` 사슬(O(n))이 된다. 표현력과 안전성이 `switch`보다 우위이며, 순수 상수일 땐 효율도 동등하다.

## 연결 노트

- [[16 - 스코프 함수와 수신 객체 관용구]] — 스코프 함수가 "값을 낳는 스코프"로 유용한 뿌리가 바로 이 장의 표현식 지향이다.
- [[18 - 범위와 진행과 반복]] — 반복으로서의 제어 흐름(`for`/`while`/`in`/범위)을 소유. 이 장의 `when in 1..10`이 쓰는 범위·`contains`의 정본.
- [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]] — 문 위치 `if`/`when`의 타입 `Unit`(M10)과, 분기의 `throw`가 낳는 `Nothing`(M11), LUB 계산의 바닥 타입 근거.
- [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]] — `when`의 스마트 캐스트가 기대는 흐름 분석과 그 한계(M13), 엘비스 `?:`의 제어 흐름(M32).
- [[30 - sealed 클래스와 대수적 데이터 타입]] — 완전한 `when`이 만개하는 무대. sealed + `when`으로 ADT/상태 기계/패턴 매칭을 짓는 정본(M23).
- [[29 - enum 클래스]] — enum 상수 나열로 완전해지는 `when`, `entries`/`values`와 `when` 완전성의 상호작용.
- [[41 - 예외와 Nothing]] — `try`도 표현식(M31)이며, `throw`가 `Nothing` 타입(M11)이라 `when` 분기에서 제어 흐름으로 쓰인다.
- [[10 - 불리언과 동등성과 동일성]] — 대상 있는 `when`의 `value ->`가 쓰는 `==`(구조적 동등, M6), String `when`이 `equals`로 비교하는 근거.
- [[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]] — `when`이 `tableswitch`/`lookupswitch`/`if-else`로 내려가는 형태를 디컴파일로 확인하는 법.
- [[19 - 연산자 오버로딩과 관례]] — `when in`이 위임하는 `contains` 관례, `..`이 위임하는 `rangeTo` 관례.
