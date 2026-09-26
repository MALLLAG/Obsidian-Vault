---
title: sealed 클래스와 대수적 데이터 타입
date: 2026-07-13
tags: [kotlin, sealed, adt, when, exhaustiveness, 학습노트]
---

**이 장이 답하는 질문:**

- `sealed`는 정확히 무엇을 "봉인"하는가 — 상속을 막는 `final`과 무엇이 다른가?
- sealed 하위 타입은 정말 "같은 파일"에만 둘 수 있는가, 아니면 다른 규칙이 있는가?
- `when`이 `else` 없이도 컴파일되는 마법의 근거는 무엇이며, 그 완전성은 어디까지 컴파일러가 보증하는가?
- `else`를 하나 추가하는 것이 왜 안전을 *깨뜨리는* 행위가 될 수 있는가?
- sealed와 enum은 둘 다 "닫힌 집합"인데, 언제 무엇을 골라야 하는가 — 둘의 근본 차이는 무엇인가?
- "대수적 데이터 타입"에서 *대수(algebra)*란 무슨 뜻인가 — 타입에 왜 덧셈과 곱셈이 있는가?
- 왜 `Nothing`이 제네릭 sealed 계층의 빈 케이스를 표현하는 열쇠가 되는가?
- 새 하위 타입을 라이브러리에 추가하는 것이 왜 하위 호환을 깨는 변경이 될 수 있는가?

---

앞선 [[29 - enum 클래스]]에서 우리는 "미리 정해진 유한한 상수들의 집합"을 언어가 어떻게 하나의 타입으로 못박는지 — 각 상수가 사실은 그 enum 타입의 단일 인스턴스(객체)이며, `when`이 그 유한성을 근거로 완전성 검사를 수행한다는 것 — 을 보았다. 그러나 enum은 강력한 만큼 경직돼 있다. 모든 상수가 *같은 타입*이고 *같은 프로퍼티 스키마*를 공유하며, 각 상수는 정확히 하나의 인스턴스뿐이다. "원 하나와 사각형 하나"처럼 케이스마다 담아야 할 데이터의 *모양*이 다르면 enum으로는 표현이 어색해진다. 이 장의 sealed 클래스는 정확히 그 빈자리를 채운다 — **케이스는 유한하되, 각 케이스가 자기만의 타입·생성자·상태를 갖는** 닫힌 계층이다.

이 장이 다루는 것과 다루지 않는 것의 경계를 먼저 못박자. sealed 클래스·인터페이스의 문법과 봉인 규칙(어디에 하위 타입을 둘 수 있는가), `when`의 완전성(exhaustiveness), 그리고 sealed가 곱타입인 [[27 - 데이터 클래스]]와 결합해 대수적 데이터 타입(ADT)을 이루는 큰 그림이 이 장의 심장이다. 그러나 `when` 표현식 자체의 문법·스마트캐스트·2.1의 가드 조건은 [[17 - 표현식으로서의 제어 흐름 - if와 when]]이 정본으로 소유하고, 여기서는 sealed가 그 완전성을 어떻게 *가능케 하는지*에 집중한다. 바닥 타입 `Nothing`의 위상은 [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]]과 [[41 - 예외와 Nothing]]이 소유하며, 여기서는 그것이 제네릭 ADT의 빈 케이스로 어떻게 쓰이는지만 본다. 변성(`out`/`in`)의 안전성 규칙은 [[33 - 제네릭 1 - 타입 파라미터와 변성]]이 소유하고, 여기서는 제네릭 sealed 계층이 왜 공변이어야 하는지만 짚는다. 데이터 클래스가 왜 `final`이라 상속을 못 하는지, 그래서 다형적 값 계층을 봉인 아래에 두어야 하는지의 배경은 [[27 - 데이터 클래스]]가 이미 깔아 두었다.

논증의 궤적은 이렇다. 먼저 sealed가 "제한된 상속"으로서 무엇을 봉인하는지 문법과 컴파일 형태로 밝히고, 이어 이 장의 소유 오개념인 **"sealed는 같은 파일에만 둘 수 있다"(M23)** 를 역사와 함께 정면으로 교정한다. 그다음 봉인의 유일한 실용적 보상인 `else` 없는 완전한 `when`을 해부하고, 역설적으로 `else`가 어떻게 그 안전을 되돌리는지 보인다. sealed와 enum을 대조해 선택 기준을 세우고, "대수"라는 말의 정체 — 타입의 개수(cardinality) 산술 — 를 통해 합타입과 곱타입이 왜 각각 덧셈·곱셈인지 증명한다. 마지막으로 재귀적 ADT(표현식 트리·상태 기계), 제네릭 sealed와 `Nothing`의 결합, 그리고 봉인의 컴파일·리플렉션·하위 호환이라는 어두운 구석으로 내려간다.

---

## 1. sealed란 무엇인가 — 봉인된 계층

### 1.1 문제: "완전성을 아는" 다형성

객체지향의 다형성은 근본적으로 *개방적*이다. `open class Animal`을 만들면, 오늘 내가 아는 하위 타입은 `Dog`·`Cat`뿐이어도, 내일 다른 누군가가 `Snake`를 추가할 수 있다. 이 개방성은 확장에는 좋지만, "이 타입의 모든 경우를 빠짐없이 처리했는가?"를 컴파일러가 검증할 수 없게 만든다. `when (animal)`에서 `Dog`와 `Cat`만 처리하고 `else`로 얼버무릴 수밖에 없고, 그러면 나중에 추가된 `Snake`가 조용히 `else`로 흘러들어가 논리 버그가 된다.

그러나 프로그램에는 개방적일 필요가 *없는* 계층이 아주 많다. 도형은 원·사각형·삼각형으로 충분하고, 결제의 결과는 성공·실패·대기뿐이며, 계산식은 숫자·덧셈·곱셈의 조합이다. 이런 계층은 "가능한 경우가 유한하고, 그 목록을 저자가 완전히 통제한다"는 성질을 갖는다. sealed는 이 성질을 언어에 선언하는 장치다 — **"이 타입의 직접 하위 타입은 지금 여기 있는 것들이 전부이며, 외부의 누구도 새로 추가할 수 없다"** 고 컴파일러에게 약속한다.

```kotlin
sealed interface Shape
data class Circle(val radius: Double) : Shape
data class Rectangle(val width: Double, val height: Double) : Shape
data class Triangle(val base: Double, val height: Double) : Shape

fun area(s: Shape): Double = when (s) {   // else가 없다!
    is Circle -> Math.PI * s.radius * s.radius
    is Rectangle -> s.width * s.height
    is Triangle -> 0.5 * s.base * s.height
}
```

`Shape`가 sealed이므로 컴파일러는 "직접 하위 타입은 정확히 `Circle`·`Rectangle`·`Triangle` 셋뿐"임을 안다. 그래서 `when`이 이 셋을 모두 처리하면 완전하다고 판정하고 `else`를 요구하지 않는다. 만약 `Triangle` 케이스를 빠뜨리면 컴파일 에러가 난다. 이 "빠뜨림을 컴파일 타임에 잡아 준다"는 것이 sealed가 존재하는 유일하고 결정적인 이유다.

### 1.2 문법: `sealed class`와 `sealed interface`

sealed는 `class`와 `interface` 앞에 붙는 수식어(modifier)다. 둘의 선택은 [[25 - 인터페이스]]와 [[24 - 상속과 오버라이딩과 초기화 순서]]가 다루는 클래스/인터페이스의 일반 차이를 따른다.

```kotlin
// sealed class: 공통 상태·생성자를 하위와 공유할 수 있다
sealed class Expr {
    abstract fun eval(): Double     // 추상 멤버 가능
}

// sealed interface: 상태 없는 순수 계약, 다중 상속 가능
sealed interface Event
```

`sealed interface`는 Kotlin 1.5에서 도입됐다(그전엔 sealed class만 있었다). 인터페이스라서 **한 클래스가 여러 sealed 인터페이스를 동시에 구현**할 수 있어, 하나의 클래스가 여러 닫힌 계층에 동시에 속하는 유연한 분류가 가능하다. 반면 sealed class는 상태와 공통 구현을 하위 타입에 물려줄 수 있다는 장점이 있다.

### 1.3 sealed class는 암묵적으로 abstract다

sealed class의 본질을 이해하는 핵심은, **sealed class 자체는 인스턴스화할 수 없다**는 것이다. sealed는 암묵적으로 `abstract`이며, 그 생성자는 직접 호출될 수 없다.

```kotlin
sealed class Expr

fun main() {
    val e = Expr()   // 컴파일 에러: Sealed types cannot be instantiated
}
```

이는 논리적으로 당연하다. sealed는 "이 타입의 값은 반드시 나열된 하위 타입 중 하나"라는 약속인데, `Expr` 자체의 인스턴스를 만들 수 있다면 그 약속이 깨진다 — 어느 하위 타입에도 속하지 않는 "그냥 Expr"이 존재하게 되기 때문이다. 그래서 sealed class는 언제나 추상이고, 값은 오직 구체 하위 타입을 통해서만 존재한다.

> **명세 정밀**: sealed class의 생성자는 기본적으로 `protected` 가시성을 가지며, `private`으로도 만들 수 있으나 `public`·`internal`로는 만들 수 없다. 왜냐하면 생성자가 외부에 공개되면 봉인 밖에서 인스턴스를 만들 여지가 생기기 때문이다. 컴파일러는 이를 문법 수준에서 강제한다. sealed interface는 생성자가 없으므로 이 제약이 무의미하다.

### 1.4 컴파일러가 직접 하위 타입을 전부 안다

sealed의 마법은 "컴파일러가 직접 하위 타입의 *완전한 목록*을 컴파일 타임에 안다"는 한 문장으로 요약된다. 일반 `open class`에서는 이 목록이 열려 있어(누가 어디서 상속할지 모름) 확정할 수 없지만, sealed에서는 봉인 규칙(2절) 덕분에 그 목록이 컴파일 시점에 확정된다.

```text
open class Animal                     sealed class Expr
      │                                     │
   ???  ← 목록이 열림                  ┌────┼────┐
  Dog  Cat  ...  (미지의 확장 가능)   Num  Add  Mul   ← 목록이 닫힘
                                       (컴파일러가 셋 전부를 안다)

  when(animal): else 필수              when(expr): else 불요
  (완전성 증명 불가)                   (완전성 증명 가능)
```

이 "닫힌 목록"이 곧 이후 모든 이야기의 토대다. 완전한 `when`(3절)도, 대수적 데이터 타입의 합타입 성질(5절)도, 리플렉션의 `sealedSubclasses`(7절)도 전부 이 한 가지 사실 — **직접 하위 타입의 집합이 컴파일 타임에 유한하고 확정적** — 에서 파생된다.

---

## 2. 봉인의 경계 — 어디에 하위 타입을 둘 수 있는가 (M23)

### 2.1 오개념의 정면 교정

**M23: "sealed의 하위 타입은 같은 파일에만 둘 수 있다."** 이것은 오래된 Kotlin 문서와 튜토리얼에 화석처럼 남은, 이제는 틀린 규칙이다. 진실은 이렇다: **현대 Kotlin(1.5 이상, 2.x 포함)에서 sealed 타입의 직접 하위 타입은 "같은 모듈(module) + 같은 패키지(package)"에 있으면 되며, 반드시 같은 파일일 필요는 없다.**

```kotlin
// ── 파일 Shape.kt ──
package geometry
sealed interface Shape

// ── 파일 Circle.kt (같은 패키지 geometry, 같은 모듈) ──
package geometry
data class Circle(val radius: Double) : Shape   // OK! 다른 파일이어도 됨

// ── 파일 Rectangle.kt (같은 패키지, 같은 모듈) ──
package geometry
data class Rectangle(val w: Double, val h: Double) : Shape   // OK!
```

세 하위 타입이 서로 다른 파일에 흩어져 있어도, 같은 패키지 `geometry`이고 같은 컴파일 모듈에 있으므로 봉인이 성립한다. "같은 파일" 신화를 믿고 모든 하위 타입을 한 파일에 욱여넣을 필요가 없다 — 큰 계층은 파일을 나눠 관리해도 된다.

### 2.2 봉인 규칙의 역사 — 왜 이 오개념이 생겼는가

M23이 한때 *참*이었기 때문에 이 오개념은 뿌리가 깊다. 규칙은 세 단계로 완화돼 왔다.

| 버전 | 직접 하위 타입의 위치 제약 |
|------|--------------------------|
| Kotlin 1.0 | sealed class 안에 **중첩(nested)** 되어야만 함 |
| Kotlin 1.1 | 같은 **파일**의 최상위(top-level)도 허용 |
| Kotlin 1.5+ (2.x) | 같은 **모듈 + 같은 패키지**면 됨(파일 무관), sealed interface 도입 |

> **역사 메모**: 초기 Kotlin(1.0)에서는 sealed 하위 타입을 반드시 부모 클래스 몸통 안에 중첩해야 했다 — `sealed class Expr { class Num : Expr(); class Add : Expr() }` 꼴이다. 이 시절 코드와 문서가 "sealed는 한 곳에 모여 있어야 한다"는 인상을 남겼고, 1.1이 "같은 파일"로 완화하자 M23의 형태로 굳었다. 1.5가 sealed interface를 들이며 "같은 모듈+패키지"로 다시 완화한 뒤에도, 옛 규칙의 잔상이 튜토리얼에 남아 오개념을 재생산한다. Kotlin 2.x 기준으로 정확한 규칙은 "같은 모듈 + 같은 패키지"다.

### 2.3 왜 하필 "모듈 + 패키지"인가

봉인의 목적은 완전성 검사의 *건전성(soundness)* 이다. 컴파일러가 "직접 하위 타입은 이게 전부"라고 단언하려면, 컴파일하는 시점에 그 전부를 *볼 수 있어야* 한다. 이 가시성의 경계가 바로 **모듈(컴파일 단위)** 이다([[26 - 가시성 한정자]]가 소유한 `internal`의 경계와 같은 개념).

- **모듈 경계가 필요한 이유**: 만약 다른 모듈이 sealed 타입을 상속할 수 있다면, 그 sealed 타입을 라이브러리로 배포한 뒤 사용자가 새 하위 타입을 추가할 수 있게 되고, 원래 모듈을 컴파일할 때는 그 하위 타입이 존재하지 않았으므로 완전성 검사가 거짓이 된다. 모듈 안으로 가두면 "이 모듈을 컴파일할 때 모든 직접 하위 타입이 함께 컴파일된다"가 보장된다.
- **패키지 경계가 필요한 이유**: 같은 모듈 안이라도 패키지까지 제한하면, 하위 타입들이 논리적으로 응집되고 컴파일러/도구가 하위 타입을 찾는 범위가 좁아진다. 이는 설계 규율이자 구현 편의다.

```text
        ┌──────────── 모듈 A (라이브러리) ────────────┐
        │  package geometry                            │
        │    sealed interface Shape                    │
        │    data class Circle    : Shape   ← 봉인 안  │
        │    data class Rectangle : Shape   ← 봉인 안  │
        └──────────────────────────────────────────────┘
                              ▲
                              │ 상속 시도
        ┌──────────── 모듈 B (사용자) ────────────┐
        │  class Hexagon : Shape   ← 컴파일 에러!   │
        │   "Inheritance of sealed types from      │
        │    other modules is prohibited"          │
        └───────────────────────────────────────────┘
```

이 그림이 sealed의 실용적 의미를 압축한다. **라이브러리 저자는 sealed로 계층을 "완결"시켜, 사용자가 새 케이스를 추가하지 못하게 봉인**한다. 그 대가로 저자는 모든 케이스를 자기가 통제하고, 사용자는 그 유한성에 기대어 완전한 `when`을 쓸 수 있다. 이것이 "확장에는 닫혀 있지만 소비에는 안전한" 계층의 설계다.

### 2.4 직접 하위 타입에만 걸리는 제약 — 간접 하위는 자유

봉인 규칙은 **직접(direct) 하위 타입**에만 적용된다. sealed 타입을 직접 상속한 하위 타입이 그 자신은 sealed가 아니고 `open`이라면, *그것의* 하위 타입(간접 하위)은 다른 모듈·패키지에 있어도 된다.

```kotlin
// 모듈 A
sealed class Node
open class Container : Node()      // 직접 하위 (같은 모듈+패키지 필수)

// 모듈 B
class SpecialContainer : Container()   // 간접 하위 — OK! Container가 open이므로
```

컴파일러의 완전성 검사는 `when`에서 `is Container`만 확인하면 되므로(간접 하위는 전부 `Container`의 하위이니 `is Container`에 걸린다), 간접 하위의 위치는 완전성 건전성에 영향을 주지 않는다. 봉인은 "첫 단계 분기"만 닫으면 충분하다.

### 2.5 하위 타입이 될 수 없는 것: 지역 클래스와 익명 객체

sealed 타입의 직접 하위 타입은 **이름 있는(named) 선언**이어야 한다. 함수 안의 지역 클래스나 이름 없는 익명 객체([[31 - 중첩 클래스와 이너 클래스와 익명 객체]])는 sealed의 하위가 될 수 없다.

```kotlin
sealed interface Shape

fun makeShape(): Shape {
    class LocalCircle : Shape   // 컴파일 에러: 지역 클래스는 sealed 하위 불가
    return object : Shape {}     // 컴파일 에러: 익명 객체는 sealed 하위 불가
}
```

이유는 명료하다. 완전성 검사가 성립하려면 컴파일러가 모든 하위 타입에 안정적인 이름으로 접근할 수 있어야 하는데, 지역 클래스·익명 객체는 특정 함수 실행 문맥에서만 존재하고 이름으로 참조할 수 없어 `when`의 분기 대상이 될 수 없다. 하위 타입은 반드시 최상위이거나 다른 이름 있는 선언(클래스·객체·인터페이스) 안에 중첩된 것이어야 한다.

---

## 3. 완전한 when — 봉인이 주는 유일한 보상 (M12 참조)

### 3.1 else 없는 분기의 근거

sealed가 감내하는 모든 제약(인스턴스화 불가, 모듈 봉인, 상속 금지)의 대가로 얻는 단 하나의 보상이 **완전한 `when`** 이다. `when`의 대상 타입이 sealed이고 모든 직접 하위 타입을 분기가 덮으면, `else` 가지가 필요 없다.

```kotlin
sealed interface Json
data class JsonString(val value: String) : Json
data class JsonNumber(val value: Double) : Json
data class JsonBool(val value: Boolean) : Json
data object JsonNull : Json
data class JsonArray(val elements: List<Json>) : Json

fun render(j: Json): String = when (j) {   // 표현식 when — else 없음
    is JsonString -> "\"${j.value}\""
    is JsonNumber -> j.value.toString()
    is JsonBool -> j.value.toString()
    JsonNull -> "null"
    is JsonArray -> j.elements.joinToString(",", "[", "]") { render(it) }
}
```

`when`이 표현식([[17 - 표현식으로서의 제어 흐름 - if와 when]])으로 쓰일 때는 값을 반환해야 하므로 완전성이 *언제나* 요구된다. sealed 타입이면 컴파일러가 다섯 하위 타입을 전부 확인해 "완전함"을 증명하고 통과시킨다. 여기서 `JsonNull`은 `data object`라 인스턴스가 하나뿐이므로 `is` 없이 값 비교(`JsonNull ->`)로 처리하는 점, `JsonArray` 안에서 `render`를 재귀 호출하는 점을 눈여겨보라 — 재귀적 ADT(6절)의 전형이다.

이 완전성은 [[17 - 표현식으로서의 제어 흐름 - if와 when]]이 소유한 M12("when은 switch다")의 핵심 반례다. `switch`는 fall-through와 `default`로 대충 막는 문(statement)이지만, Kotlin의 `when`은 sealed·enum·Boolean 같은 유한 타입에 대해 완전성을 *증명하는* 표현식이다.

### 3.2 문(statement) when도 완전성이 강제된다

역사적으로 `when`을 문으로 쓸 때(값을 안 쓸 때)는 완전성이 강제되지 않아, 누락이 조용한 경고에 그쳤다. 그러나 **Kotlin 1.7부터 sealed·enum 대상의 문 `when`도 비완전(non-exhaustive)이면 컴파일 에러**가 된다.

```kotlin
sealed interface Signal
data object Red : Signal
data object Green : Signal
data object Yellow : Signal

fun handle(s: Signal) {
    when (s) {              // 문으로 사용 (반환값 없음)
        Red -> stop()
        Green -> go()
        // Yellow 누락!
    }
    // 컴파일 에러: 'when' expression must be exhaustive,
    //             add necessary 'Yellow' branch or 'else' branch
}
```

> **역사 메모**: Kotlin 1.6까지는 이 경우가 경고(warning)였고, 1.7에서 에러로 승격됐다. 그 전 코드베이스와의 호환을 위해 단계적으로 조인 것이다. 2.x에서는 완전한 에러이므로, sealed·enum을 대상으로 한 `when`은 문이든 표현식이든 모든 케이스를 덮거나 명시적 `else`를 두어야 한다. 이 승격은 "누락된 케이스가 조용히 무시되는" 버그 부류를 언어 차원에서 박멸하려는 결정이었다.

### 3.3 else의 역설 — 안전을 되돌리는 한 줄

완전한 `when`의 진짜 가치는 "지금 완전함"이 아니라 **"미래에 케이스가 늘면 컴파일러가 알려 준다"** 는 데 있다. 새 하위 타입을 추가하면, `else` 없는 모든 `when`이 동시에 컴파일 에러가 나서 "여기도 처리해야 한다"고 소리친다. 이것이 sealed의 가장 값진 안전망이다.

```kotlin
sealed interface Shape
data class Circle(val r: Double) : Shape
data class Rectangle(val w: Double, val h: Double) : Shape
data class Triangle(val b: Double, val h: Double) : Shape   // ← 새로 추가

// 기존에 else 없이 짠 when → Triangle 추가 순간 컴파일 에러!
fun area(s: Shape) = when (s) {
    is Circle -> Math.PI * s.r * s.r
    is Rectangle -> s.w * s.h
    // 컴파일 에러: 'when' must be exhaustive, add 'Triangle' branch
}
```

그런데 여기에 함정이 있다. 만약 원래 코드가 `else`를 두었다면, `Triangle`이 추가돼도 컴파일 에러가 나지 *않고* 조용히 `else`로 흘러들어간다.

```kotlin
fun area(s: Shape) = when (s) {
    is Circle -> Math.PI * s.r * s.r
    is Rectangle -> s.w * s.h
    else -> 0.0     // ← 이 한 줄이 안전망을 무력화한다
}
// Triangle 추가 후: 컴파일 성공, 그러나 area(triangle)은 조용히 0.0 반환 — 버그!
```

**`else` 한 줄이 sealed의 컴파일 타임 완전성 검사를 통째로 꺼 버린다.** `else`가 있으면 컴파일러는 "나머지는 저기서 처리되겠지" 하고 누락을 검사하지 않는다. 그래서 sealed를 다룰 때의 규율은 명확하다 — **정말로 "그 외 전부"를 동일하게 처리하는 의도가 아니라면, sealed의 `when`에는 `else`를 쓰지 말라.** `else`를 생략함으로써 우리는 "케이스가 늘면 컴파일러야, 나를 깨워라"라고 요청하는 것이다.

> **흔한 오해**: "안전을 위해 항상 `else`를 붙여 방어적으로 짜자"는 습관은 sealed에 관한 한 정반대의 결과를 낳는다. `else`는 "미래의 누락"을 감춰 버린다. 열린 계층(`open class`, 플랫폼 타입)에서는 `else`가 불가피하지만, sealed에서는 `else`의 부재가 곧 안전 장치다. 굳이 모든 케이스를 처리했음을 명시하고 싶다면, `else` 대신 반환 타입을 명시해 표현식으로 만들면 컴파일러가 완전성을 요구하게 된다.

### 3.4 완전성이 성립하지 않는 경우

완전성 검사는 컴파일러가 대상 타입의 하위 집합을 확정할 수 있을 때만 작동한다. 대상이 sealed의 하위 타입이 아니라 상위(예: `Any`)이거나, 널 가능([[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]])하거나, 스마트캐스트가 좁히지 못한 경우엔 완전성이 성립하지 않는다.

```kotlin
sealed interface Shape
data class Circle(val r: Double) : Shape
data class Rectangle(val w: Double, val h: Double) : Shape

fun describe(s: Shape?) = when (s) {   // Shape? — null이 하나의 케이스!
    is Circle -> "circle"
    is Rectangle -> "rect"
    null -> "none"                     // null도 명시해야 완전
}
// null 가지를 빼면 컴파일 에러 (Shape?의 완전성엔 null 케이스 포함)
```

`Shape?`는 `Shape`의 모든 케이스 *더하기* `null`이므로, 완전하려면 `null` 케이스까지 덮어야 한다. 이는 5절의 대수 관점으로 보면 자연스럽다 — `Shape?`의 카디널리티는 `Shape` 카디널리티에 1(널)을 더한 값이기 때문이다. nullable을 벗겨 `Shape`로 좁힌 뒤 분기하거나(`?.let`), `null` 가지를 명시적으로 두는 것이 정석이다.

---

## 4. sealed vs enum — 언제 무엇을 (29장 대조)

### 4.1 근본 차이: 인스턴스의 개수와 타입의 다양성

sealed와 [[29 - enum 클래스]]는 둘 다 "닫힌 유한 집합"을 표현하지만, 그 유한성의 *결이 다르다*. enum은 **정해진 개수의 단일 인스턴스**들의 집합이고, sealed는 **정해진 개수의 하위 타입**들의 집합이다.

```kotlin
// enum: 네 상수 = 네 개의 유일한 인스턴스, 전부 같은 타입 Direction
enum class Direction { NORTH, SOUTH, EAST, WEST }
// Direction.NORTH는 프로그램 전체에서 정확히 하나의 객체

// sealed: 세 하위 타입 = 각자 다른 타입, 각자 여러 인스턴스 가능
sealed interface Shape
data class Circle(val r: Double) : Shape        // Circle 인스턴스는 무수히 많다
data class Rectangle(val w: Double, val h: Double) : Shape
```

`Direction.NORTH`는 유일한 싱글턴이지만, `Circle(1.0)`과 `Circle(2.0)`은 서로 다른 두 인스턴스다. enum 상수는 상태를 가질 수 있으나(생성자 프로퍼티) **모든 상수가 같은 프로퍼티 스키마**를 공유해야 하는 반면, sealed 하위 타입은 **케이스마다 완전히 다른 프로퍼티**를 가질 수 있다.

### 4.2 비교표

| 기준 | `enum class` | `sealed class`/`interface` |
|------|-------------|---------------------------|
| 각 케이스의 정체 | 단일 인스턴스(싱글턴) | 하나의 타입 (여러 인스턴스 가능) |
| 케이스별 데이터 | 같은 스키마 공유(같은 생성자) | 케이스마다 다른 프로퍼티 |
| 케이스별 타입 | 모두 같은 enum 타입 | 각자 독립 타입 (`is`로 구분) |
| 인스턴스 수 | 상수당 정확히 1개 | 하위 타입당 임의 개수 |
| 열거 API | `entries`/`values()`/`valueOf` | 없음(리플렉션 `sealedSubclasses`) |
| `when` 완전성 | 지원 | 지원 |
| 순회 가능성 | 자연스러움(`entries`로 반복) | 어색함(인스턴스가 무한) |
| 적합한 경우 | 고정된 이름표·상태 없는 선택지 | 모양이 다른 케이스·데이터 담은 변형 |

### 4.3 선택 기준: 데이터의 유무

실무의 판단 기준은 간단하다. **각 케이스가 담을 데이터의 모양이 같으면(혹은 없으면) enum, 케이스마다 다르면 sealed.**

```kotlin
// enum이 맞는 경우: 요일은 이름표일 뿐, 추가 데이터 없음
enum class Weekday { MON, TUE, WED, THU, FRI }

// sealed가 맞는 경우: 각 결과가 담는 것이 다르다
sealed interface FetchResult
data class Success(val body: String) : FetchResult       // 본문 문자열
data class Failure(val code: Int, val message: String) : FetchResult  // 코드+메시지
data object Loading : FetchResult                        // 아무 데이터 없음
```

`FetchResult`를 enum으로 표현하려면 모든 상수가 `body`·`code`·`message`를 다 갖고 안 쓰는 필드는 null로 두는 어색한 스키마가 되어 버린다. 케이스마다 데이터가 다르면 sealed가 자연스럽다. 반대로 `Weekday`를 sealed로 만들면 다섯 개의 `data object`를 선언하는 과잉이 된다 — 데이터 없는 이름표엔 enum이 간결하다.

### 4.4 하이브리드: enum이 sealed interface를 구현한다

둘은 배타적이지 않다. **enum 클래스는 sealed 인터페이스를 구현할 수 있다**(1.5+). 이를 통해 "데이터 없는 여러 상수"와 "데이터 있는 변형"을 하나의 닫힌 계층으로 묶을 수 있다.

```kotlin
sealed interface Command
enum class Move : Command { UP, DOWN, LEFT, RIGHT }   // enum이 sealed 구현
data class Jump(val height: Int) : Command            // 데이터 있는 케이스
data object Quit : Command

fun exec(c: Command): String = when (c) {
    Move.UP -> "up"; Move.DOWN -> "down"
    Move.LEFT -> "left"; Move.RIGHT -> "right"
    is Jump -> "jump ${c.height}"
    Quit -> "quit"
}
```

여기서 `Move`의 네 상수는 데이터 없는 이름표라 enum이 적절하고, `Jump`는 높이 데이터를 담으니 data class가 적절하며, 이 이질적인 것들이 `Command`라는 sealed 인터페이스 아래에서 하나의 완전성 있는 계층을 이룬다. `when`은 enum 상수(`Move.UP`)와 sealed 하위 타입(`is Jump`)을 한 곳에서 섞어 처리하고, 여전히 `else` 없이 완전하다. 이 조합은 enum의 간결함과 sealed의 유연함을 함께 취하는 강력한 관용구다.

> **명세 정밀**: enum이 sealed interface를 구현하는 경우, 완전성 검사는 그 enum의 *모든 상수*를 개별 케이스로 취급한다. 위 예에서 `Move` 상수 하나라도 빠뜨리면 비완전이 된다. 다만 `is Move ->` 하나로 enum 전체를 뭉뚱그려 처리하는 것도 허용되며, 그 경우 `Move`의 개별 상수는 검사에서 만족된 것으로 본다.

---

## 5. 합타입과 곱타입 — "대수"의 정체

### 5.1 왜 "대수적" 데이터 타입인가

"대수적 데이터 타입(Algebraic Data Type, ADT)"이라는 이름은 은유가 아니라 문자 그대로의 수학이다. 여기서 "대수"란 **타입이 담을 수 있는 서로 다른 값의 개수 — 카디널리티(cardinality) — 에 대해 덧셈과 곱셈이 성립한다**는 뜻이다. 타입을 "그 타입의 가능한 값의 집합의 크기"로 환원하면, 복합 타입의 크기가 구성 요소 크기의 산술로 정확히 계산된다.

기본 타입들의 카디널리티를 세어 보자.

$$|\text{Nothing}| = 0, \quad |\text{Unit}| = 1, \quad |\text{Boolean}| = 2$$

`Nothing`([[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]])은 값이 하나도 없는 바닥 타입이라 크기가 0이다. `Unit`은 값이 정확히 하나(`Unit` 그 자체)라 크기가 1이다. `Boolean`은 `true`/`false` 둘이라 크기가 2다. 이 세 수 — 0, 1, 2 — 가 타입 대수의 상수다.

### 5.2 곱타입(product type): data class = 곱셈

[[27 - 데이터 클래스]]가 소유한 data class는 여러 값을 *동시에* 담는다. 두 필드를 가진 data class의 가능한 값의 개수는 각 필드가 가질 수 있는 값의 개수의 **곱**이다.

$$|A \times B| = |A| \times |B|$$

```kotlin
data class Pair2(val a: Boolean, val b: Boolean)
// 가능한 값: (F,F) (F,T) (T,F) (T,T) → 4개 = |Boolean| × |Boolean| = 2 × 2
```

`Boolean` 두 개를 곱하면 $2 \times 2 = 4$가지다. 필드가 하나라도 `Nothing`이면($|N|=0$) 전체 곱이 0이 되어 그 data class는 인스턴스를 만들 수 없다 — `Nothing` 필드를 채울 값이 없으니 당연하다. 필드가 `Unit`이면 곱의 그 항이 1이라 전체 개수에 영향을 주지 않는다. 그래서 data class를 **곱타입**이라 부른다.

### 5.3 합타입(sum type): sealed = 덧셈

sealed 계층은 여러 케이스 *중 하나*를 담는다. sealed 타입의 가능한 값의 개수는 각 하위 타입이 가질 수 있는 값의 개수의 **합**이다.

$$|A + B| = |A| + |B|$$

```kotlin
sealed interface Toggle
data class OnLevel(val level: Boolean) : Toggle   // |OnLevel| = |Boolean| = 2
data object Off : Toggle                          // |Off| = 1 (싱글턴)
// 가능한 값: OnLevel(false), OnLevel(true), Off → 3개 = 2 + 1
```

`Toggle`은 `OnLevel`(2가지) *또는* `Off`(1가지)이므로 $2 + 1 = 3$가지다. sealed는 "이것 아니면 저것"의 선택이라 카디널리티가 더해진다. 그래서 sealed를 **합타입**이라 부른다. enum도 사실 각 상수가 크기 1인 케이스들의 합이므로, `enum class Direction { NORTH, SOUTH, EAST, WEST }`의 카디널리티는 $1+1+1+1 = 4$다.

### 5.4 ADT = 합과 곱의 조합

진짜 대수적 데이터 타입은 합(sealed)과 곱(data class)을 자유롭게 중첩한 것이다. 이 조합이 임의의 복잡한 데이터 구조의 정확한 카디널리티를 산술로 표현하게 해 준다.

```kotlin
sealed interface Shape
data class Circle(val r: Double) : Shape                   // |Double|
data class Rect(val w: Double, val h: Double) : Shape      // |Double| × |Double|
data object Point : Shape                                  // 1

// |Shape| = |Double| + |Double|² + 1
```

`Shape`의 카디널리티는 $|Double| + |Double|^2 + 1$이다. 원은 반지름 하나(합의 한 항), 사각형은 너비×높이(곱), 점은 데이터 없는 싱글턴(1). 이렇게 sealed(합) 안에 data class(곱)를 넣어 복합 구조를 짓는 것이 ADT의 본질이다.

> **명세 정밀**: 함수 타입 `(A) -> B`([[13 - 함수 타입과 람다와 함수 참조]])는 대수적으로 **거듭제곱(지수)** 에 대응한다 — $|A \to B| = |B|^{|A|}$다. `(Boolean) -> Boolean`은 입력 2가지 각각에 출력 2가지를 배정하므로 $2^2 = 4$가지 서로 다른 함수(항등, 부정, 상수 true, 상수 false)가 존재한다. sealed(합)·data class(곱)·함수(지수)가 갖춰지면 타입 시스템은 초등 대수의 세 연산을 모두 갖는다. 이것이 "대수적"이라는 말의 완전한 의미다.

### 5.5 카디널리티가 알려 주는 설계 지혜

이 산술은 장난이 아니라 실전 설계 도구다. **불가능한 상태를 표현 불가능하게(make illegal states unrepresentable)** 만드는 원리가 여기서 나온다. 나쁜 설계는 카디널리티를 필요 이상으로 부풀려 "말이 안 되는 값의 조합"을 타입이 허용하게 만든다.

```kotlin
// 나쁨: nullable 필드의 곱 — 카디널리티가 과잉이고 모순 상태가 표현된다
data class BadResult(
    val isSuccess: Boolean,
    val data: String?,      // 성공일 때만 의미
    val error: String?,     // 실패일 때만 의미
)
// isSuccess=true인데 data=null, error="oops" 같은 모순 값이 타입상 가능하다

// 좋음: sealed 합타입 — 정확히 유효한 조합만 표현된다
sealed interface GoodResult
data class Ok(val data: String) : GoodResult      // 성공엔 반드시 data
data class Err(val error: String) : GoodResult    // 실패엔 반드시 error
// Ok에는 error 필드가 아예 없다 → 모순 상태를 만들 수 없다
```

`BadResult`는 `Boolean × String? × String?`의 곱이라 카디널리티가 거대하고, 그중 대부분이 "성공인데 에러 메시지가 있는" 모순 조합이다. `GoodResult`는 sealed 합이라 유효한 조합만 정확히 담는다. sealed로 "합"을 표현하는 것은 곧 **불필요한 곱을 없애 불가능한 상태를 타입 수준에서 제거**하는 일이다. 대수를 아는 설계자는 카디널리티를 줄여 버그의 여지를 줄인다.

---

## 6. 재귀적 ADT — 표현식 트리와 상태 기계

### 6.1 재귀적 sealed: 자기 자신을 담는 케이스

sealed 하위 타입이 다시 그 sealed 타입을 프로퍼티로 담으면 **재귀적 ADT**가 된다. 이는 트리, 리스트, 표현식 같은 재귀 구조를 정확하고 안전하게 모델링하는 정통 기법이다.

```kotlin
sealed interface Expr
data class Num(val value: Double) : Expr
data class Add(val left: Expr, val right: Expr) : Expr   // Expr을 다시 담는다
data class Mul(val left: Expr, val right: Expr) : Expr
data class Neg(val operand: Expr) : Expr

// (2 + 3) * -4 를 트리로
val tree: Expr = Mul(Add(Num(2.0), Num(3.0)), Neg(Num(4.0)))

fun eval(e: Expr): Double = when (e) {          // 완전 — else 없음
    is Num -> e.value
    is Add -> eval(e.left) + eval(e.right)      // 재귀
    is Mul -> eval(e.left) * eval(e.right)
    is Neg -> -eval(e.operand)
}

fun main() {
    println(eval(tree))   // => -20.0   ((2+3) * -4)
}
```

`Expr`은 sealed 합이고, `Add`/`Mul`/`Neg`는 자기 안에 다시 `Expr`을 담는 곱이다. `eval`은 sealed의 완전성 덕분에 `else` 없이 네 케이스를 모두 덮으며, `is Add ->`에서 스마트캐스트([[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]])로 `e`가 `Add`임이 확정돼 `e.left`·`e.right`에 안전하게 접근한다. 이것이 컴파일러가 검증하는 인터프리터 패턴 — 타입이 곧 문법이고, `when`이 곧 해석기다.

### 6.2 확장의 안전성

재귀 ADT의 진가는 확장할 때 드러난다. `Expr`에 뺄셈 `Sub`를 추가한다고 하자.

```kotlin
data class Sub(val left: Expr, val right: Expr) : Expr   // 새 케이스 추가

// eval, print, simplify ... Expr을 다루는 모든 when이 동시에 컴파일 에러!
// 컴파일러가 "Sub도 처리하라"고 전부 짚어 준다
```

`else`를 쓰지 않은 모든 `when`이 즉시 컴파일 에러를 내며 "여기, 여기, 여기도 `Sub`를 처리하라"고 알려 준다. 표현식 언어에 새 노드를 추가하는 위험한 리팩터링이, 컴파일러가 빠짐없이 안내하는 안전한 작업으로 바뀐다. 이 "확장 시 컴파일러의 전수 안내"가 sealed ADT를 프로덕션 파서·인터프리터·직렬화기의 뼈대로 만드는 이유다.

### 6.3 상태 기계: 상태를 타입으로

sealed는 **상태 기계(state machine)** 를 모델링하는 데 이상적이다. 각 상태를 하위 타입으로, 상태별 데이터를 그 하위 타입의 프로퍼티로, 전이를 함수로 표현하면, "그 상태에서만 유효한 데이터"가 타입 수준에서 보장된다.

```kotlin
// 문(door)의 상태 기계
sealed interface DoorState
data object Open : DoorState
data object Closed : DoorState
data class Locked(val keyCode: Int) : DoorState   // 잠긴 상태에만 키코드가 있다

// 전이 함수: 유효한 전이만 정의된다
fun DoorState.close(): DoorState = when (this) {
    Open -> Closed
    Closed -> Closed          // 이미 닫힘 (멱등)
    is Locked -> this         // 잠긴 문은 그냥 닫는 걸로 안 열림
}

fun DoorState.unlock(code: Int): DoorState = when (this) {
    is Locked -> if (code == keyCode) Closed else this  // 코드 맞으면 열림(닫힘 상태로)
    else -> this              // Open/Closed는 잠겨 있지 않으니 그대로
}
```

`Locked`만이 `keyCode`를 갖는다는 점이 핵심이다 — `Open` 상태에서 실수로 키코드에 접근하는 코드는 애초에 컴파일되지 않는다(스마트캐스트로 `is Locked` 안에서만 `keyCode`가 보인다). 상태가 담는 데이터가 상태마다 다르다는 성질을 sealed가 정확히 포착한다. enum으로 이를 흉내 내려면 모든 상수에 nullable `keyCode`를 두어야 하고, 그러면 5.5의 "불가능한 상태가 표현 가능해지는" 문제로 되돌아간다.

```text
상태 전이 다이어그램 (문)

        close()          unlock(code✓)
  Open ────────► Closed ◄──────────── Locked(keyCode)
                    │                      ▲
                    │  (lock 전이 생략)     │ unlock(code✗) → 자기 자신
                    └──────────────────────┘

각 상태 = sealed 하위 타입 / 전이 = when 기반 함수
Locked만 keyCode 보유 → 타입이 "그 상태의 데이터"를 강제
```

### 6.4 재귀 자료구조: 연결 리스트

교과서적 재귀 ADT의 극단은 연결 리스트다. 리스트는 "빈 리스트" 아니면 "머리 원소 + 꼬리 리스트"라는 두 케이스의 합이다.

```kotlin
sealed interface LinkedList<out T>
data object Nil : LinkedList<Nothing>                        // 빈 리스트
data class Cons<T>(val head: T, val tail: LinkedList<T>) : LinkedList<T>

fun <T> LinkedList<T>.size(): Int = when (this) {
    Nil -> 0
    is Cons -> 1 + tail.size()      // 재귀
}

fun main() {
    val list: LinkedList<Int> = Cons(1, Cons(2, Cons(3, Nil)))
    println(list.size())   // => 3
}
```

여기서 `Nil`이 `LinkedList<Nothing>`을 구현한 것과 `out T` 변성이 결정적인데, 이 마법은 다음 절에서 해부한다. 요점은 sealed 두 케이스(`Nil` + `Cons`)의 합으로 임의 길이의 재귀 구조가 정확히 표현되고, 그 위의 모든 연산(`size`·`map`·`fold`)이 완전한 `when`으로 안전하게 짜인다는 것이다.

---

## 7. 제네릭 sealed와 Nothing — 빈 케이스의 대수

### 7.1 제네릭 sealed 계층

sealed 타입도 제네릭일 수 있다. 함수형 프로그래밍의 두 간판 ADT — "값이 있거나 없거나"인 `Option`과 "둘 중 하나"인 `Either` — 가 대표적이다.

```kotlin
sealed interface Option<out T>
data class Some<T>(val value: T) : Option<T>
data object None : Option<Nothing>          // 값 없음

sealed interface Either<out L, out R>
data class Left<L>(val value: L) : Either<L, Nothing>
data class Right<R>(val value: R) : Either<Nothing, R>
```

`Option<T>`는 "값이 있는 `Some<T>` 또는 값이 없는 `None`"의 합이다. 대수적으로 $|Option\langle T\rangle| = |T| + 1$이다 — `T`의 모든 값에 "없음"이라는 한 값을 더한 것. 이것이 nullable `T?`의 카디널리티($|T|+1$)와 정확히 같다는 점이 흥미롭다. Kotlin은 언어 차원의 nullable([[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]])이 있어 `Option`을 표준으로 두진 않지만, 대수적으로 둘은 같은 합타입이다.

### 7.2 왜 `None`이 `Option<Nothing>`인가 — 변성과 바닥 타입의 협연

`None`이 값을 담지 않으므로 타입 파라미터가 필요 없는데, 그렇다면 `None`은 `Option<Int>`인가 `Option<String>`인가? 답은 **`Option<Nothing>`이면서 동시에 모든 `Option<T>`** 다. 이 우아함은 두 요소의 협연에서 나온다.

1. **`Nothing`은 바닥 타입**([[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]])이라 모든 타입의 하위 타입이다: 모든 `T`에 대해 `Nothing <: T`.
2. **`Option`이 `out T`로 공변**([[33 - 제네릭 1 - 타입 파라미터와 변성]])이라, `Nothing <: T`이면 `Option<Nothing> <: Option<T>`.

두 사실을 합치면, `None: Option<Nothing>`은 모든 `T`에 대해 `Option<T>`의 하위 타입이 된다.

```kotlin
val a: Option<Int> = None       // OK: Option<Nothing> <: Option<Int>
val b: Option<String> = None    // OK: Option<Nothing> <: Option<String>
// 하나의 None 싱글턴이 모든 Option<T>의 자리에 들어간다
```

$$\text{Nothing} <: T \;\land\; \text{Option is covariant} \;\Rightarrow\; \text{Option}\langle\text{Nothing}\rangle <: \text{Option}\langle T\rangle$$

`None`이라는 단 하나의 싱글턴이 `Option<Int>`·`Option<String>`·`Option<Any>` 어디에나 값으로 들어갈 수 있는 것은 바닥 타입과 공변성의 이 조합 덕분이다. 6.4의 `Nil: LinkedList<Nothing>`도, `Either`의 `Left<L>: Either<L, Nothing>`도 전부 같은 원리다 — **"비어 있는 쪽"의 타입 인자를 `Nothing`으로 두고 공변으로 선언하면, 하나의 값이 모든 인스턴스화에 재사용된다.**

> **명세 정밀**: 만약 `Option`이 `out` 없이 무공변(invariant)이었다면, `Option<Nothing>`은 `Option<Int>`의 하위 타입이 아니게 되어 `val a: Option<Int> = None`이 컴파일되지 않는다. 그 경우 `None`을 매번 `None as Option<Int>`로 캐스팅하거나 제네릭 함수로 감싸야 한다. `out`이 이 불편을 없앤다. 반대로 담는 데이터가 소비 위치([[33 - 제네릭 1 - 타입 파라미터와 변성]]의 `in`)에 있으면 공변으로 만들 수 없어 이 기법을 못 쓴다. ADT의 케이스가 데이터를 *생산*(읽기)만 하고 소비(쓰기)하지 않는 불변 설계일 때 이 우아함이 성립한다.

### 7.3 표준 라이브러리 대조: `Result`는 sealed가 아니다

흥미롭게도 Kotlin 표준 라이브러리의 `kotlin.Result<T>`는 "성공 또는 실패"라는 전형적 합타입인데도 **sealed 클래스가 아니라 `value class`([[36 - 타입 별칭과 인라인 value class]])** 로 구현돼 있다.

```kotlin
// 개념적으로 Result는 대략 이렇게 생겼다 (실제 구현 단순화)
@JvmInline
value class Result<out T> internal constructor(val value: Any?) {
    // 성공이면 value에 T, 실패면 value에 Failure(예외 래퍼)를 담는다
}
```

왜 sealed가 아니라 value class인가? **박싱 회피** 때문이다([[36 - 타입 별칭과 인라인 value class]], M39). sealed로 만들면 성공값을 담을 때마다 `Success` 객체를 힙에 할당해야 하지만, value class로 하면 성공 경로에서 값을 감싸는 래퍼 객체 할당을 대개 피할 수 있다. `runCatching { ... }` 같은 흔한 연산의 성공 경로를 저렴하게 만들려는 성능 선택이다. 이는 "합타입이라고 무조건 sealed는 아니다"라는 교훈을 준다 — 케이스가 둘뿐이고 성능이 중요하면 value class가, 케이스가 많고 다형적 분기가 중심이면 sealed가 맞다. `Result`가 sealed였다면 `when`으로 분기할 수 있었겠지만, Kotlin은 성능을 위해 `getOrNull()`·`exceptionOrNull()`·`fold()` 같은 메서드 API를 대신 제공한다.

---

## 8. 봉인의 컴파일과 런타임 — 내부에서 무슨 일이

### 8.1 바이트코드로 본 sealed

sealed class는 JVM 백엔드에서 특별한 종류의 것이 아니라, **생성자가 봉인된 추상 클래스(abstract class)** 로 컴파일된다. sealed의 "봉인" 정보 — 즉 직접 하위 타입의 목록 — 은 Kotlin의 `@Metadata` 애노테이션에 기록되어, 컴파일러가 다른 모듈에서 이 타입을 볼 때 완전성 검사에 사용한다.

```text
sealed class Expr { abstract fun eval(): Double }
        │
        │  K2 프론트엔드(FIR): sealed 수식어 인지
        │  → 직접 하위 타입 집합 수집 (모듈+패키지 스캔)
        │  → 생성자 가시성을 protected/private로 제한
        │
        ▼  JVM 백엔드
   ┌──────────────────────────────────────────────┐
   │ Expr.class:                                    │
   │   - abstract class (직접 인스턴스화 불가)       │
   │   - 생성자: 공개 아님                            │
   │   - @Metadata: sealedSubclasses = [Num,Add,..] │
   └──────────────────────────────────────────────┘
```

> **명세 정밀**: 완전성 검사의 근거는 Kotlin 컴파일러가 아는 하위 타입 집합(같은 모듈의 소스 + 다른 모듈이면 그 `@Metadata`)이다. JVM 타깃·버전에 따라 컴파일러가 JVM 17의 네이티브 `sealed`/`PermittedSubclasses` 클래스 파일 속성을 함께 낼 수도 있으나, **코틀린 자체의 완전성 검사와 상속 금지는 언어 수준 정보(메타데이터·모듈 봉인)에 근거하며 JVM 속성에 의존하지 않는다.** 즉 sealed의 의미는 Kotlin 컴파일러가 보증하는 것이지, 밑바닥 플랫폼 기능의 파생물이 아니다. Kotlin/Native·JS·Wasm 타깃에서도 sealed는 동일한 언어 의미로 동작하며, 구현 방식은 백엔드에 따라 다르다.

### 8.2 리플렉션: `sealedSubclasses`

sealed의 하위 타입 목록은 런타임에도 리플렉션([[44 - 애노테이션과 리플렉션]])으로 조회할 수 있다. `KClass.sealedSubclasses`가 직접 하위 타입의 `KClass` 목록을 돌려준다.

```kotlin
sealed interface Shape
data class Circle(val r: Double) : Shape
data class Rectangle(val w: Double, val h: Double) : Shape
data object Point : Shape

fun main() {
    val subs = Shape::class.sealedSubclasses
    println(subs.map { it.simpleName })
    // => [Circle, Rectangle, Point]   (직접 하위 타입만, 순서는 보장 안 됨)
}
```

이 API는 "모든 케이스를 자동으로 순회"하는 코드(직렬화기, 테스트 생성기)를 짤 때 유용하다. 다만 두 가지 주의가 있다. 첫째, **직접 하위 타입만** 반환한다(간접 하위는 각 하위의 `sealedSubclasses`를 재귀로 따라가야 한다). 둘째, JVM에서 이 리플렉션은 `kotlin-reflect` 의존성을 요구하며 비용이 있다. `entries`로 O(1) 순회가 되는 enum과 달리, sealed는 리플렉션 없이는 하위 타입을 열거할 수 없다(하위마다 인스턴스가 무한이라 "값의 열거"라는 개념 자체가 성립하지 않는다).

### 8.3 스마트캐스트와 sealed의 결합

sealed의 `when`이 강력한 것은 완전성뿐 아니라 **각 분기 안에서 스마트캐스트**([[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]])가 작동하기 때문이다. `is Circle ->` 가지 안에서는 대상이 `Circle`로 자동 캐스트되어 `radius`에 바로 접근할 수 있다.

```kotlin
sealed interface Shape
data class Circle(val r: Double) : Shape
data class Rectangle(val w: Double, val h: Double) : Shape

fun perimeter(s: Shape): Double = when (s) {
    is Circle -> 2 * Math.PI * s.r        // s는 여기서 Circle (스마트캐스트)
    is Rectangle -> 2 * (s.w + s.h)       // s는 여기서 Rectangle
}
```

`data object`로 만든 하위 타입은 `is` 대신 값 비교(참조 동일성, [[10 - 불리언과 동등성과 동일성]])로 분기하는데, 싱글턴이라 인스턴스가 하나뿐이므로 `Point ->`처럼 쓴다. 컴파일러는 이 값 비교 분기도 완전성 계산에 포함한다. sealed + `when` + 스마트캐스트의 삼위일체가 "타입으로 분기하고 데이터에 안전하게 접근하는" ADT 소비의 표준 형태다.

### 8.4 sealed 하위 타입의 다양한 형태

sealed의 직접 하위 타입은 `data class`만 되는 게 아니다 — 일반 `class`, `object`, `data object`, 다른 `sealed class`(중첩 계층), 심지어 `enum class`(sealed interface의 경우)까지 될 수 있다.

```kotlin
sealed interface Tree
data class Leaf(val value: Int) : Tree              // data class
class RawNode(val children: List<Tree>) : Tree      // 일반 class
data object EmptyTree : Tree                         // data object 싱글턴

sealed interface Branch : Tree                       // 중첩 sealed! Tree의 하위이자 자신도 sealed
data class BinaryBranch(val l: Tree, val r: Tree) : Branch
data class UnaryBranch(val child: Tree) : Branch
```

여기서 `Branch`는 `Tree`의 직접 하위이면서 *자신도 sealed*라, 2단계 봉인 계층이 만들어진다. `when (tree)`는 `is Branch`로 한 번에 뭉뚱그리거나, `is BinaryBranch`/`is UnaryBranch`로 세분할 수 있고, 어느 쪽이든 완전성이 계산된다. 이런 중첩 봉인은 큰 도메인을 계층적으로 분류할 때 유용하다. 하위 타입의 형태가 자유롭다는 것이 enum(모든 상수가 동일 형태)에 비해 sealed가 갖는 유연함의 핵심이다.

---

## 9. 어두운 구석과 실무 규율

### 9.1 sealed 하위 타입 추가는 하위 호환을 깰 수 있다

sealed의 완전성 안전망은 양날의 검이다. 라이브러리가 sealed 타입에 **새 하위 타입을 추가하면, 그 타입을 `else` 없이 소비하던 모든 다운스트림 코드가 컴파일 에러**가 난다. 내 모듈 안에서는 이것이 축복이지만(6.2), 공개 라이브러리에서는 **소스 하위 호환을 깨는 변경(breaking change)** 이다.

```kotlin
// 라이브러리 v1
sealed interface ApiError
data class NotFound(val id: String) : ApiError
data class Timeout(val ms: Long) : ApiError

// 사용자 코드 (else 없이 완전하게 짬)
fun message(e: ApiError) = when (e) {
    is NotFound -> "not found: ${e.id}"
    is Timeout -> "timeout ${e.ms}ms"
}

// 라이브러리 v2에서 하위 타입 추가
data class RateLimited(val retryAfter: Long) : ApiError
// → 사용자 코드가 v2로 올리는 순간 컴파일 에러! (RateLimited 미처리)
```

그래서 공개 API에서 sealed는 신중히 써야 한다. "케이스가 앞으로 늘 수 있다"면 sealed로 봉인하는 것이 사용자에게 반복적 마이그레이션을 강요한다. 반대로 "케이스는 이게 전부이고 늘지 않는다"는 확신이 있을 때 sealed가 빛난다. Kotlin은 이 트레이드오프를 위해 실험적으로 특정 하위 타입만 공개하고 나머지를 감추는 장치(추상 상위 노출)를 두기도 하지만, 근본은 설계 판단이다 — **닫을 것인가(sealed), 열 것인가(일반 인터페이스).**

> **성능 주의**: sealed 자체엔 런타임 성능 비용이 없다(그냥 추상 클래스다). 비용은 오히려 설계·유지보수 차원이다. 하위 타입이 수십 개로 불어나면 `when` 분기가 비대해지고, 새 케이스마다 모든 `when`을 손봐야 한다. 이는 "동작을 타입에 붙이는" 다형성(각 하위가 메서드를 오버라이드)과 "동작을 when에 모으는" ADT 스타일 사이의 고전적 트레이드오프다. 케이스가 고정이고 연산이 자주 늘면 ADT(when 중심)가, 케이스가 자주 늘고 연산이 고정이면 다형성(오버라이드 중심)이 유리하다 — 이른바 표현 문제(expression problem)다.

### 9.2 sealed vs open 인터페이스: 개방-폐쇄의 두 얼굴

sealed는 [[24 - 상속과 오버라이딩과 초기화 순서]]가 다루는 개방-폐쇄 원칙(OCP)을 *뒤집는다*. 전통적 OCP는 "확장에 열려 있으라"고 하지만, sealed는 의도적으로 "확장에 닫는다". 이 상충은 무엇을 우선하느냐의 문제다.

| | `open` 인터페이스 (열림) | `sealed` (닫힘) |
|---|------------------------|----------------|
| 케이스 추가 | 누구나 어디서나 | 저자만, 같은 모듈+패키지 |
| 연산 추가 | 각 케이스에 메서드 추가(전부 수정) | `when` 함수 하나 추가(케이스 무수정) |
| 완전성 검사 | 불가(`else` 필수) | 가능(`else` 불요) |
| 적합 | 플러그인·확장점 | 폐쇄된 도메인 모델 |

즉 sealed는 "케이스는 고정하되 연산을 자유롭게 늘리고 싶을 때", 열린 인터페이스는 "연산은 고정하되 케이스(구현)를 자유롭게 늘리고 싶을 때" 맞다. 결제 상태나 파싱 노드처럼 "가능한 경우가 도메인상 확정된" 모델은 sealed가, 로그 핸들러나 렌더러처럼 "구현이 계속 추가되는" 확장점은 열린 인터페이스가 맞다.

### 9.3 완전성이 착각을 부르는 경우: 상위 타입으로의 우회

sealed의 완전성은 대상의 정적 타입이 sealed일 때만 작동한다. 대상을 `Any`나 다른 상위 타입으로 올리면 완전성이 사라지고 `else`가 강제된다. 또한 Java에서 정의된 계층([[45 - Java 상호운용]])이나 플랫폼 타입은 봉인 정보가 없어 완전성 검사가 안 된다.

```kotlin
sealed interface Shape
data class Circle(val r: Double) : Shape

fun f(x: Any) = when (x) {     // 대상이 Any → 완전성 없음
    is Circle -> "circle"
    else -> "other"            // else 필수 (Any는 무한히 열림)
}
```

이는 착각이 아니라 올바른 동작이다 — `Any`는 sealed가 아니므로 컴파일러가 케이스를 확정할 수 없다. sealed의 이점을 누리려면 대상을 sealed 타입으로 유지해야 한다.

### 9.4 sealed와 data class의 결합이 정석인 이유

[[27 - 데이터 클래스]]에서 보았듯 data class는 `final`이라 상속할 수 없다. 따라서 "값이면서 다형적인 계층"은 data class를 상속시켜 만들 수 없고, **sealed 계층 아래에 각 케이스를 data class로 두는 것**이 유일하게 안전한 길이다.

```kotlin
sealed interface Shape                        // 합타입(sum) — 다형성
data class Circle(val r: Double) : Shape       // 곱타입(product) — 값 의미론
data class Rectangle(val w: Double, val h: Double) : Shape
```

이 조합에서 각 data class는 자기 안에서 final이라 `equals` 대칭성([[27 - 데이터 클래스]]의 M21·상속 논증)이 보전되고, 서로 다른 타입 간 동등성은 항상 `false`라 문제가 없으며, sealed가 완전성을 준다. sealed(다형성)와 data class(값 의미론)는 서로의 약점을 정확히 메운다 — data class는 상속을 못 하지만 봉인 아래에서 다형적 케이스가 되고, sealed는 값 비교를 자동 생성하지 못하지만 각 케이스를 data class로 두어 무료 `equals`/`copy`/`toString`을 얻는다. 이 결합이 Kotlin에서 ADT를 표현하는 관용적 정석이다.

### 9.5 실무 체크리스트

sealed를 쓸지 말지의 실무 판단을 정리하면 이렇다.

- **케이스가 유한하고 저자가 통제하는가?** → 그렇다면 sealed로 완전성을 얻어라.
- **케이스마다 데이터 모양이 다른가?** → 그렇다면 enum이 아니라 sealed다(같으면/없으면 enum).
- **공개 라이브러리 API인가, 케이스가 늘 수 있는가?** → 그렇다면 sealed는 사용자에게 마이그레이션을 강요하니 신중히.
- **`when`에 `else`를 쓰고 있는가?** → sealed라면 대개 빼라. `else`는 완전성 안전망을 끈다.
- **케이스가 둘뿐이고 성능이 중요한가?** → sealed 대신 value class(`Result` 스타일)를 고려하라.
- **상태별로 다른 데이터를 강제하고 싶은가?** → sealed 상태 기계로 불가능한 상태를 표현 불가능하게 만들어라.

---

sealed 클래스는 결국 하나의 약속의 언어적 표현이다. "이 타입의 경우는 여기 나열한 것이 전부이며, 아무도 더 추가할 수 없다"는 약속. 이 장이 던진 질문들은 모두 이 한 약속의 파생이다. `when`이 `else` 없이 완전한 것도, 새 케이스를 추가하면 컴파일러가 모든 소비처를 짚어 주는 것도, 리플렉션이 하위 타입을 열거할 수 있는 것도 전부 "직접 하위 타입의 집합이 컴파일 타임에 확정된다"는 봉인의 귀결이다. 그리고 그 봉인의 경계는 사람들이 오래 믿어 온 "같은 파일"이 아니라 **"같은 모듈 + 같은 패키지"** 다(M23) — 옛 규칙의 화석일 뿐, 현대 Kotlin은 파일을 나눠도 봉인이 성립한다. sealed는 합타입(sum type)으로서 곱타입인 data class와 결합해 대수적 데이터 타입을 이루며, 그 "대수"는 은유가 아니라 타입의 카디널리티에 대한 실제 덧셈과 곱셈이다. 이 산술을 아는 설계자는 sealed로 합을 표현해 불가능한 상태를 타입 수준에서 지워 버린다. 표현식 트리와 상태 기계, `Option`과 `Either`, 재귀 리스트가 전부 이 합과 곱의 조합 위에 안전하게 선다. 그리고 그 안전의 정점에서 `Nothing`이라는 바닥 타입과 공변성이 만나, 하나의 `None`이 모든 `Option<T>`의 자리를 채운다. 다음 [[31 - 중첩 클래스와 이너 클래스와 익명 객체]]에서 sealed 하위 타입이 될 수 없었던 익명 객체·이너 클래스의 세계로 내려가고, [[33 - 제네릭 1 - 타입 파라미터와 변성]]에서 이 장이 예고만 한 `out` 변성이 왜 안전한지를 규칙 수준에서 완성한다.

## 핵심 요약

- **sealed는 `final`과 다르다 — 상속을 완전히 막는 것이 아니라 "직접 하위 타입을 같은 모듈+패키지로 제한"하는 봉인이다.** sealed class는 암묵적으로 `abstract`라 직접 인스턴스화할 수 없고, 값은 오직 나열된 하위 타입을 통해서만 존재한다. 생성자는 `protected`/`private`만 가능하다.
- **sealed 하위 타입은 "같은 파일"이 아니라 "같은 모듈 + 같은 패키지"에 있으면 된다 (M23).** 이 오개념은 Kotlin 1.0(중첩만)·1.1(같은 파일)의 옛 규칙에서 온 화석이며, 1.5 이후 2.x는 파일을 나눠도 봉인이 성립한다. 봉인이 모듈로 제한되는 것은 완전성 검사의 건전성 때문이다.
- **봉인의 유일하고 결정적인 보상은 `else` 없는 완전한 `when`이다 (M12 참조).** 컴파일러가 직접 하위 타입의 완전한 목록을 컴파일 타임에 알기에, 모든 케이스를 덮으면 `else`가 불요하다. Kotlin 1.7부터 문(statement) `when`도 비완전이면 에러다.
- **`else` 한 줄이 완전성 안전망을 통째로 끈다.** sealed에 새 케이스가 추가될 때 `else` 없는 `when`은 컴파일 에러로 누락을 짚어 주지만, `else`가 있으면 조용히 그리로 흘러 버그가 된다. sealed의 `when`에는 원칙적으로 `else`를 쓰지 말라.
- **enum과 sealed는 유한성의 결이 다르다.** enum은 "정해진 개수의 단일 인스턴스"(모두 같은 타입·스키마), sealed는 "정해진 개수의 하위 타입"(각자 다른 타입·데이터·여러 인스턴스). 케이스별 데이터 모양이 같으면/없으면 enum, 다르면 sealed다. enum이 sealed interface를 구현하는 하이브리드도 가능하다.
- **"대수적"은 은유가 아니라 카디널리티 산술이다.** sealed는 합타입($|A+B|=|A|+|B|$), data class는 곱타입($|A\times B|=|A|\times|B|$), 함수 타입은 지수($|B|^{|A|}$)에 대응한다. `Nothing`은 크기 0, `Unit`은 1, `Boolean`은 2다.
- **sealed 합타입은 불가능한 상태를 표현 불가능하게 만든다.** nullable 필드들의 곱은 "성공인데 에러가 있는" 모순 조합을 허용하지만, sealed 합은 유효한 조합만 정확히 담아 카디널리티를 줄이고 버그의 여지를 없앤다.
- **재귀적 sealed는 트리·상태 기계·리스트를 안전하게 모델링한다.** 하위 타입이 다시 sealed 타입을 담으면 재귀 ADT가 되고, `when`이 곧 재귀 해석기가 된다. 상태 기계에서는 "그 상태에만 있는 데이터"를 하위 타입 프로퍼티로 두어 타입 수준에서 강제한다.
- **`Nothing`과 공변성이 만나 하나의 빈 케이스가 모든 인스턴스화를 채운다.** `None: Option<Nothing>`은 `Nothing <: T`(바닥 타입)와 `out T`(공변)의 결합으로 모든 `Option<T>`의 하위 타입이 된다. 이것이 제네릭 ADT의 빈 케이스를 싱글턴 하나로 표현하는 원리다(M45·M11 참조).
- **합타입이라고 반드시 sealed는 아니다.** 표준 `Result<T>`는 "성공 또는 실패"의 합이지만 박싱 회피를 위해 sealed가 아니라 `value class`로 구현됐다(M39). 케이스가 둘뿐이고 성능이 중요하면 value class, 케이스가 많고 다형적 분기가 중심이면 sealed다.
- **공개 API에서 sealed 하위 타입 추가는 소스 하위 호환을 깨는 변경이다.** `else` 없이 소비하던 다운스트림 코드가 새 케이스 앞에서 컴파일 에러를 낸다. 모듈 내부에서는 축복(안전망)이지만 라이브러리 경계에서는 사용자에게 마이그레이션을 강요하는 트레이드오프다.
- **sealed(다형성) + data class(값 의미론)의 결합이 ADT의 정석이다.** data class는 `final`이라 상속으로 다형적 계층을 못 만들지만, sealed 아래 각 케이스를 data class로 두면 `equals` 대칭성을 지키면서 무료 `equals`/`copy`/`toString`과 완전성을 함께 얻는다.

## 연결 노트

- [[29 - enum 클래스]] — sealed의 자매. "단일 인스턴스의 유한 집합"(enum) 대 "하위 타입의 유한 집합"(sealed)의 대조, `entries` 순회, enum이 sealed interface를 구현하는 하이브리드의 정본.
- [[27 - 데이터 클래스]] — sealed와 결합해 곱타입(product type)을 이루는 값 케이스. data class가 왜 `final`이라 봉인 아래에 두어야 하는지, `equals` 대칭성 논증(M21)을 소유한다.
- [[17 - 표현식으로서의 제어 흐름 - if와 when]] — `when`의 완전성(exhaustiveness) 검사와 표현식 의미론, 스마트캐스트 결합, 2.1 가드 조건의 정본(M12). 이 장은 sealed가 그 완전성을 어떻게 가능케 하는지를 다룬다.
- [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]] — 바닥 타입 `Nothing`(크기 0)이 제네릭 ADT의 빈 케이스로 쓰이는 배경, `Unit`(크기 1)·`Any`의 카디널리티를 소유(M10, M11).
- [[33 - 제네릭 1 - 타입 파라미터와 변성]] — `out T` 공변성이 `None: Option<Nothing>`을 모든 `Option<T>`의 하위 타입으로 만드는 규칙의 정본. 제네릭 sealed 계층이 왜 공변이어야 하는지를 완성한다(M20, M45).
- [[36 - 타입 별칭과 인라인 value class]] — 합타입 `Result`가 sealed가 아니라 value class로 구현된 이유(박싱 회피 M39). "합타입이라고 무조건 sealed는 아니다"의 배경.
- [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]] — `when` 각 분기의 스마트캐스트, `T?`의 카디널리티가 $|T|+1$이라 `null` 케이스까지 덮어야 완전해지는 원리(M13, M7).
- [[31 - 중첩 클래스와 이너 클래스와 익명 객체]] — sealed 하위 타입이 될 수 *없는* 지역 클래스·익명 객체의 세계. 왜 이름 있는 선언만 봉인 하위가 되는지의 배경.
- [[44 - 애노테이션과 리플렉션]] — `KClass.sealedSubclasses`로 직접 하위 타입을 런타임에 열거하는 리플렉션 API와 그 비용, `@Metadata`에 봉인 정보가 기록되는 방식.
- [[25 - 인터페이스]] — sealed interface의 토대. 인터페이스가 상태 없이 계약만 제공하며 다중 구현이 가능하다는 성질이 sealed interface의 유연함(한 클래스가 여러 봉인 계층에 속함)으로 이어진다.
