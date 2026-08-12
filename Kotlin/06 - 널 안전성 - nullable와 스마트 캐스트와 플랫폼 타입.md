---
title: 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입
date: 2026-07-13
tags: [kotlin, null-safety, smart-cast, platform-type, elvis, 학습노트]
---

**이 장이 답하는 질문:**

- 코틀린의 널 안전성은 *런타임 검사*인가, 아니면 대부분 *컴파일 타임*에 끝나는가? (M7)
- `String`과 `String?`은 문법 설탕의 차이인가, 아니면 서로 다른 *타입*인가? 둘 사이엔 어떤 부분타입 관계가 있는가?
- 안전 호출 `?.`은 내부에서 정확히 어떤 코드로 펼쳐지는가? 체인 `a?.b?.c`는 어디서 멈추는가?
- 엘비스 `?:`는 "널이면 기본값"만 하는 연산자인가, 아니면 *제어 흐름*을 표현하는가? (M32)
- `!!`은 무엇을 던지며, 왜 그것을 "죄"라고 부르는가? `as?`는 어떻게 예외 대신 널을 낳는가?
- 스마트 캐스트는 마법인가? 컴파일러는 무슨 근거로 `x`가 널이 아니라고 확신하는가?
- 왜 `var` 프로퍼티는, 방금 널 검사를 했는데도 스마트 캐스트가 *안 되는가*? (M13)
- Java에서 넘어온 `String!`(플랫폼 타입)은 안전한가? 왜 컴파일러가 침묵하는가? (M8)
- 널 검사를 함수 밖으로 "약속"하는 contract란 무엇인가? 표준 라이브러리는 그것을 어떻게 쓰는가?

---

앞의 [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]]에서 우리는 코틀린 타입 우주의 지형을 그렸다. 꼭대기엔 `Any?`가 있고, 그 바로 아래 `Any`가 있으며, 맨 바닥엔 모든 타입의 하위인 `Nothing`이 있었다. 그 지도에서 가장 눈에 띄는 균열이 하나 있었다. 같은 이름의 타입이 물음표 하나로 갈라진다는 것 — `Any`와 `Any?`, `String`과 `String?`. 그 물음표가 정확히 무엇을 의미하고, 컴파일러가 그것으로 무엇을 할 수 있는지를 05장은 "이 장에서 심화한다"며 미뤄 두었다. 이 장이 그 빚을 갚는다.

이 장이 다루는 영토는 **널 가능성(nullability)이 타입 시스템에 편입되었을 때 벌어지는 모든 일**이다. 널 가능 타입(`T?`)의 정체, 그 타입을 다루는 세 연산자(`?.`, `?:`, `!!`)와 안전 캐스트(`as?`), 컴파일러가 제어 흐름을 읽어 타입을 좁히는 **스마트 캐스트(smart cast)**, 그리고 Java 세계에서 널 정보 없이 넘어오는 **플랫폼 타입(platform type)** `T!`이 이 장의 다섯 기둥이다. 반면, 널이 아니라 *초기화*의 지연을 다루는 `lateinit`과 `by lazy`는 [[11 - 변수와 초기화 - val과 const와 lateinit]]이 소유하고(널 가능성과 자주 혼동되므로 이 장에서 경계만 못박는다), `==`가 널을 어떻게 안전하게 비교하는지의 세부는 [[10 - 불리언과 동등성과 동일성]]이, `Nothing`이 왜 바닥 타입인지의 이론은 05장과 [[41 - 예외와 Nothing]]이, Java 상호운용의 전모는 [[45 - Java 상호운용]]이 소유한다. 이 장은 그 경계들에 손을 걸치되, 널 안전성이라는 하나의 궤도만 곧게 따라간다.

이 장의 논증은 이렇게 쌓인다. 먼저 "널"이 왜 십억 달러짜리 실수였는지, 그리고 코틀린이 그것을 *라이브러리*가 아니라 *타입 시스템*에서 푼다는 것이 왜 근본적으로 다른지를 본다. 그 위에서 `T`와 `T?`의 부분타입 관계를 세우고, 세 연산자를 하나씩 바이트코드 수준까지 펼친다. 그다음 이 장의 심장인 스마트 캐스트를 — 그것이 *되는* 원리와 *안 되는* 조건(M13)을 짝지어 — 해부한다. 이어서 플랫폼 타입 `T!`이 어떻게 널 안전성의 벽에 난 유일한 합법적 구멍인지(M8)를 보고, 마지막으로 이 모든 검사가 **대부분 컴파일 타임에 완결된다**(M7)는 사실로 도입부의 첫 질문을 회수한다. 엘비스가 단순한 병합이 아니라 제어 흐름이라는 것(M32)은 그 흐름 속에서 자연스럽게 드러난다.

---

## 1. 널이라는 문제 — 타입 시스템으로 옮긴 대답

### 1.1 십억 달러의 실수

널 참조(null reference)를 발명한 사람은 토니 호어(Tony Hoare)다. 그는 1965년 ALGOL W를 설계하며 "값이 없음"을 표현하는 특별한 참조를 넣었다. 이유는 단순했다 — 컴파일러가 구현하기 쉬웠기 때문이다. 2009년 그는 그것을 공개적으로 후회하며 "십억 달러짜리 실수(the billion-dollar mistake)"라 불렀다. 이후 수십 년간 셀 수 없이 많은 널 역참조(null dereference)가 시스템을 무너뜨렸고, 그 하나하나가 디버깅·장애·데이터 손상으로 쌓여 갔다는 회한이었다.

> **역사 메모**
> 호어의 후회의 핵심은 "널 그 자체"가 아니라 "널을 타입에 표시하지 않은 것"이었다. `String` 타입 변수가 문자열일 수도, `null`일 수도 있는데 타입은 그 둘을 구분하지 못했다. 그래서 "이 값은 널일 수 있는가?"라는 질문의 답이 *타입에 없고 개발자의 머릿속에만* 있었다. 널 안전성을 갖춘 언어들(코틀린, 스위프트, C# 8+의 nullable 참조 타입, 러스트의 `Option<T>`)의 공통점은 하나다 — **그 답을 타입으로 끌어올린 것**이다.

전통적인 Java에서 이 문제는 이렇게 나타난다.

```java
// Java — 타입은 널 가능성을 말하지 않는다
String name = findUser(id);   // null일 수 있는가? 시그니처만 봐선 모른다
int len = name.length();      // name이 null이면 여기서 NullPointerException
```

`findUser`가 널을 반환할 수 있는지 없는지는 `String`이라는 타입 어디에도 적혀 있지 않다. 문서를 읽거나, 소스를 열거나, 운영 중 NPE를 맞고 나서야 알게 된다. 검사는 전적으로 개발자의 규율에 맡겨진다.

### 1.2 라이브러리가 아니라 타입 시스템에서 푼다

여기서 흔한 오해 하나를 미리 짚자. 어떤 이는 코틀린의 널 안전성을 `Optional<T>` 같은 "래퍼 라이브러리"의 일종으로 여긴다. 그렇지 않다. Java 8의 `Optional<String>`은 *라이브러리 타입*이다 — 여전히 `Optional` 참조 자체가 널일 수 있고(`Optional<String> o = null;`), 값을 꺼낼 때 `.get()`을 잘못 부르면 `NoSuchElementException`이 튀며, 런타임에 객체 하나가 추가로 할당된다.

코틀린은 그 문제를 **타입 시스템(type system)** 안에서, 즉 컴파일러의 타입 검사 규칙 자체로 푼다. `String`과 `String?`은 이름은 비슷하지만 컴파일러에게는 *서로 다른 타입*이다. 그리고 결정적으로, `String` 타입 값에는 `null`을 대입하는 것 자체가 **컴파일 에러**다.

```kotlin
val a: String = "hello"
// val b: String = null   // 컴파일 에러: Null can not be a value of a non-null type String

val c: String? = "hello"  // OK — 널 가능 타입
val d: String? = null     // OK
```

이 한 줄의 차이가 전부를 바꾼다. 널일 수 있는 값은 `?`가 붙은 타입에만 담을 수 있고, `?`가 없는 타입은 컴파일러가 "절대 널이 아님"을 *보장*한다. 그리고 널 가능 타입의 값을 널 검사 없이 역참조하려 하면, 그것 역시 컴파일 에러다.

```kotlin
fun length(s: String?): Int {
    // return s.length   // 컴파일 에러: Only safe (?.) or non-null asserted (!!.)
                          //   calls are allowed on a nullable receiver of type String?
    return s?.length ?: 0 // 안전 호출 + 엘비스로 널을 명시적으로 처리해야 통과
}
```

즉 코틀린에서 "널일 수 있는 값을 그냥 점(`.`)으로 부르는 것"은 문법적으로 금지된다. 컴파일러가 널 처리를 *강제*한다. 이것이 라이브러리 접근과 근본적으로 다른 점이다 — 규율에 의존하지 않고, 타입 규칙으로 못박는다.

### 1.3 그래서 NPE는 어디서 오는가

여기서 미리 이 장 전체의 결론 하나를 심어 두자. 코틀린에서도 `NullPointerException`은 발생한다. 하지만 "순수한 코틀린 코드"에서 아무 우회 장치 없이 NPE가 나는 일은 *구조적으로 불가능*하다. NPE가 나는 경로는 몇 개로 한정된다 — `!!` 연산자, 플랫폼 타입(Java 상호운용), 리플렉션을 통한 우회, 초기화 전 `lateinit` 접근(이건 사실 다른 예외다), 생성자에서 미초기화 `this`를 흘리는 함정. 이 경로들은 이 장에서 하나씩 정면으로 다룬다. 핵심은, 이것들이 *예외적 경로*라는 점이다. 널 안전성의 기본값은 "컴파일 타임에 막힘"이다(M7). 이 명제는 이 장 9절에서 정본으로 회수한다.

---

## 2. `T`와 `T?` — 물음표의 타입 이론

### 2.1 널 가능 타입은 합집합이다

`String?`이라는 타입은 무엇인가. 개념적으로 그것은 "`String`인 모든 값 **또는** `null`"이다. 타입 이론의 언어로 말하면 합집합 타입(union type)에 가깝다.

$$\text{String?} \;=\; \text{String} \;\cup\; \{\texttt{null}\}$$

더 정확히, 코틀린의 `T?`는 `T`와 `Nothing?`의 최소 상계(least upper bound)로 볼 수 있다. `null` 리터럴 자체의 타입은 `Nothing?`이다 — 값이 하나(`null`)뿐인, 모든 널 가능 타입의 하위 타입.

```kotlin
val n = null           // 추론된 타입: Nothing?
val s: String? = null  // Nothing?의 값 null을 String?에 대입 — OK
```

이 그림에서 핵심은 **부분타입(subtype) 관계**다. `T`는 언제나 `T?`의 부분타입이다.

$$T <: T? \qquad \text{(모든 타입 } T \text{에 대해)}$$

직관적으로도 옳다 — `String`인 값은 전부 "String이거나 null"의 집합에 속하니까. 그래서 널이 아닌 값을 널 가능 타입 변수에 넣는 것은 언제나 합법이다(상향 변환, upcast). 반대 방향은 안 된다.

```kotlin
val notNull: String = "x"
val nullable: String? = notNull   // OK — String <: String? (상향 변환)

val back: String? = null
// val down: String = back         // 컴파일 에러 — String?는 String의 하위가 아니다
```

이 관계를 05장의 전체 지형과 이으면 이런 그림이 된다.

```text
                 Any?              ← 최상위 (모든 것을 담음, null 포함)
                /    \
             Any     String?·Int?·…    ← 널 가능 타입들 (Any? 아래; Any 아래는 아님)
            /   \              \
      String   Int  …        Nothing?  ← 널 가능 바닥 타입 (null 리터럴의 타입, 모든 T?의 하위)
            \    \           /
             \    \         /
              +--- Nothing ---+         ← 최하위 바닥 타입 (모든 T·모든 T?의 하위; throw, 무한루프)

수직 관계:  String  <: String? <: Any?
            Int     <: Int?    <: Any?
            Nothing  <: 모든 T,  모든 T?   ← 절대 바닥 (Nothing <: Nothing? 도 성립)
            Nothing? <: 모든 T?
```

`Any`는 널을 담을 수 없고(`Any`는 널 불가), `Any?`가 널까지 포함하는 진짜 최상위다. 그래서 "무엇이든 받는다"를 뜻하려면 `Any`가 아니라 `Any?`를 써야 한다. 이 미묘함은 05장이 소유한다.

### 2.2 `?`는 타입에 붙는다, 값에 붙지 않는다

초심자가 자주 헷갈리는 지점. `?`는 *타입 표기*의 일부지 값에 대한 연산이 아니다. `String?`은 하나의 타입 이름이다.

```kotlin
val list: List<String?> = listOf("a", null, "b")  // 원소가 널 가능한 리스트
val maybeList: List<String>? = null               // 리스트 자체가 널 가능
val both: List<String?>? = null                   // 둘 다 널 가능
```

세 타입은 전혀 다르다. `List<String?>`은 리스트는 반드시 존재하되 그 안의 원소가 널일 수 있고, `List<String>?`은 리스트 자체가 널일 수 있되 있으면 원소는 널이 아니다. `?`가 어디에 붙었는지가 의미를 결정한다. 컬렉션에서의 이 구분은 [[38 - 컬렉션 - 읽기 전용과 가변]]과도 이어진다.

### 2.3 제네릭과 널 가능성 — 기본 상한은 `Any?`

한 가지 함정. 제네릭 타입 파라미터 `T`는 *기본적으로 널 가능성을 이미 포함*한다. 타입 파라미터의 기본 상한(upper bound)이 `Any?`이기 때문이다(M45, 자세히는 [[33 - 제네릭 1 - 타입 파라미터와 변성]]).

```kotlin
fun <T> identity(x: T): T = x

identity("hello")          // T = String
identity(null)             // T = Nothing?  — null도 통과한다!

fun <T> firstOrNull(list: List<T>): T? = if (list.isEmpty()) null else list[0]
```

즉 `<T>`라고만 쓰면 `T`는 이미 널을 담을 수 있다. `T`에 널을 절대 넣지 못하게 하려면 상한을 명시해야 한다.

```kotlin
fun <T : Any> requireValue(x: T): T = x  // 이제 T는 널 불가
// requireValue(null)   // 컴파일 에러 — Nothing?는 Any의 하위가 아니다
```

이 `T : Any` 관용구는 "널이 아닌 아무 타입"을 뜻하며, 표준 라이브러리 곳곳(`requireNotNull`, `checkNotNull`, `T?.let` 대비)에 쓰인다. M45는 이 오해("`T`엔 널을 못 넣는다")를 33장이 정본으로 교정하지만, 널 안전성의 관점에서 여기서 미리 짚어 둔다.

---

## 3. 안전 호출 `?.` — 널을 전파하는 기계

### 3.1 의미론: 널이면 건너뛰고 널을 낳는다

안전 호출 연산자(safe call operator) `?.`의 규칙은 한 문장이다. **수신 객체가 널이 아니면 호출하고 그 결과를 낳으며, 널이면 호출을 건너뛰고 `null`을 낳는다.**

```kotlin
val name: String? = null
val len: Int? = name?.length   // name이 null이므로 호출 안 함 → len == null
```

여기서 결과 타입에 주목하자. `String.length`는 `Int`를 반환하지만, `name?.length`의 타입은 `Int?`다. 안전 호출은 언제나 결과를 널 가능하게 만든다 — 널일 가능성을 한 단계 전파하기 때문이다. 이것은 명세가 보장하는 성질이다.

`?.`은 개념적으로 아래 `if`와 정확히 같다(단, 수신 객체를 한 번만 평가한다는 점이 다르다 — 3.3 참고).

```kotlin
// name?.length 는 개념적으로:
val len: Int? = if (name != null) name.length else null
```

### 3.2 바이트코드 수준에서: 임시 변수와 분기

JVM 백엔드에서 `?.`이 어떻게 내려가는지를 보면 이해가 확실해진다. 컴파일러는 수신 객체를 임시 슬롯에 담고, 널이면 분기해서 `null`을 스택에 올린다.

```text
// name?.length 의 JVM 바이트코드 개념 (의사 코드)
ALOAD name           ; 수신 객체를 스택에
DUP                  ; 널 검사용 복제
IFNULL L_null        ; 널이면 L_null로 점프
  INVOKEVIRTUAL length()   ; 아니면 length() 호출
  (결과를 Integer로 박싱 — 결과 타입이 Int?이므로)
  GOTO L_end
L_null:
  POP                ; 복제한 null 정리
  ACONST_NULL        ; null을 결과로
L_end:
```

여기서 한 가지 성능 관련 사실이 드러난다. 결과 타입이 `Int?`이므로, JVM 백엔드에서 결과는 원시 `int`가 아니라 박싱된 `Integer`가 된다(널을 표현하려면 참조 타입이어야 하므로). 이것은 M5("`Int?`는 항상 박싱된 래퍼다")가 부분적으로만 맞는 이유와 이어진다 — 널을 실제로 표현해야 하는 문맥에서는 대개 박싱되지만, 컴파일러가 널일 수 없음을 아는 문맥에선 최적화될 수 있다. 정본 해부는 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]에 있다.

> **성능 주의**
> `?.` 자체는 널 검사 분기 하나에 불과해 비용이 사실상 없다. 진짜 비용은 결과가 널 가능 원시 타입일 때의 박싱이다. 뜨거운 루프에서 `Int?`를 반복 생성하면 박스 할당이 쌓인다. 다만 대개는 무시할 만하며, "성능 때문에 널 안전성을 포기"하는 것은 거의 언제나 잘못된 트레이드오프다. 측정 먼저.

### 3.3 체인: 어디서 멈추는가

`?.`의 진가는 체인에서 드러난다. `a?.b?.c?.d`는 중간 어디서든 널이 나오면 **거기서 멈추고 전체가 `null`이 된다.**

```kotlin
class Country(val capital: City?)
class City(val mayor: Person?)
class Person(val name: String)

val country: Country? = Country(City(null))

val mayorName: String? = country?.capital?.mayor?.name
// country는 non-null → capital 접근
// capital은 non-null → mayor 접근
// mayor는 null → 여기서 멈춤, 전체 결과 null
// => mayorName == null
```

이것을 Java의 방어적 코드와 비교하면 절약이 극명하다.

```java
// Java — 같은 일을 하려면
String mayorName = null;
if (country != null) {
    City capital = country.getCapital();
    if (capital != null) {
        Person mayor = capital.getMayor();
        if (mayor != null) {
            mayorName = mayor.getName();
        }
    }
}
```

중첩 `if`의 피라미드가 `?.` 체인 한 줄로 접힌다. 중요한 명세적 성질 하나: 체인의 각 수신 객체는 **한 번만 평가된다.** 즉 `getCapital()`이 부수 효과가 있는 함수라도 `country?.capital?.mayor`에서 `capital`은 한 번만 호출된다. 3.1에서 "`if`와 개념적으로 같되 한 번만 평가"라 단서를 단 이유가 이것이다. `if (a.b != null) a.b.c`처럼 손으로 쓰면 `a.b`가 두 번 평가되지만, `a.b?.c`는 한 번이다.

### 3.4 `?.let` — 널이 아닐 때만 블록 실행

안전 호출과 스코프 함수 `let`([[16 - 스코프 함수와 수신 객체 관용구]])을 엮으면, "널이 아니면 이 블록을 실행"이라는 관용구가 된다.

```kotlin
val name: String? = readName()

name?.let {
    // 이 블록은 name이 non-null일 때만 실행되고,
    // 블록 안에서 it은 String (널 아님)으로 스마트 캐스트된다
    println("이름 길이: ${it.length}")
}
```

`name?.let { ... }`은 `name`이 널이면 `let`을 아예 부르지 않으므로 블록이 실행되지 않는다. 널이 아니면 `let`이 호출되고, 그 안의 `it`은 널이 아닌 `String`이다. 이 관용구는 명령형 `if (name != null) { ... }`을 표현식으로 바꾸는데, 둘 사이엔 미묘한 차이가 있다(9.3에서 스마트 캐스트 대비로 다룬다). 지금은 이것이 널 처리의 가장 흔한 관용구 중 하나라는 것만 기억하자.

### 3.5 인덱스·호출도 안전 버전이 있다

`?.`은 멤버 접근뿐 아니라, 안전 호출의 형태로 다른 관례와도 결합한다. 다만 인덱스 접근 `a?[i]` 같은 문법은 없다 — 대신 `a?.get(i)`를 쓴다.

```kotlin
val map: Map<String, Int>? = mapOf("a" to 1)
val v: Int? = map?.get("a")     // 안전 호출로 get
// val bad = map?["a"]          // 이런 문법은 없음

// 함수 타입 값도 안전 호출로 invoke
val callback: (() -> Unit)? = null
callback?.invoke()              // callback이 null이면 호출 안 함
// callback?()  ← 이런 문법도 없음
```

`callback?.invoke()`는 널 가능 함수 타입([[13 - 함수 타입과 람다와 함수 참조]])을 안전하게 부르는 표준 방식이다.

---

## 4. 엘비스 `?:` — 널 병합을 넘어 제어 흐름으로

### 4.1 기본: 좌변이 널이면 우변

엘비스 연산자(Elvis operator) `?:`의 이름은 기호 `?:`를 90도 돌리면 엘비스 프레슬리의 앞머리와 눈처럼 보인다는 데서 왔다. 규칙은 단순하다 — **좌변이 널이 아니면 좌변을, 널이면 우변을 낳는다.**

```kotlin
val name: String? = null
val display: String = name ?: "익명"   // name이 null이므로 → "익명"
```

결과 타입에 주목하자. `name`은 `String?`인데 `name ?: "익명"`의 타입은 `String`(널 불가)이다. 우변이 널이 아닌 값을 주면, 전체 표현식은 절대 널이 될 수 없기 때문이다. 이 성질 덕분에 엘비스는 "널 가능 타입을 널 불가 타입으로 승격시키는" 표준 도구가 된다.

`?.`과 `?:`를 함께 쓰는 관용구는 코틀린 코드 어디에나 있다.

```kotlin
fun greetingLength(name: String?): Int = name?.length ?: 0
// name이 null이면 name?.length가 null → ?: 0으로 0
// name이 non-null이면 length 그대로
```

### 4.2 M32 교정: 엘비스는 "병합"이 아니라 "제어 흐름"이다

여기서 이 장이 소유하는 오개념 M32를 정면으로 무너뜨린다. 흔히 엘비스를 "널 병합 연산자(null-coalescing operator)"라 부르고, C#의 `??`나 JavaScript의 `??`와 같은 것으로 여긴다. 그 이해는 절반만 맞다. 코틀린 엘비스의 진짜 힘은, **우변에 값뿐 아니라 `throw`나 `return`을 놓을 수 있다**는 데 있다.

```kotlin
fun process(input: String?) {
    val value: String = input ?: return          // input이 null이면 함수를 즉시 반환
    // 이 아래로 오면 value는 반드시 non-null String
    println(value.length)
}

fun mustHave(config: Config?): Config =
    config ?: throw IllegalStateException("설정이 없습니다")  // 널이면 예외
```

왜 이것이 가능한가? 답은 05장의 바닥 타입 `Nothing`에 있다. `return`과 `throw`는 값을 낳지 않고 제어 흐름을 끊는 표현식이며, 그 *타입이 `Nothing`* 이다(M11, [[41 - 예외와 Nothing]]). 그리고 `Nothing`은 모든 타입의 하위 타입이다. 엘비스 `a ?: b`의 결과 타입은 좌변의 널 불가 버전과 우변 타입의 최소 상계인데, 우변이 `Nothing`이면 최소 상계는 그냥 좌변의 널 불가 타입이 된다.

$$\text{lub}(\text{String},\ \text{Nothing}) = \text{String}$$

그래서 `input ?: return`의 타입은 `String`이 되고, 그 아래 코드에서 `value`는 널이 아닌 것으로 컴파일러가 확신한다. 이것은 단순한 값 대체가 아니라, **컴파일러의 제어 흐름 분석에 참여하는 문법**이다. "널이면 여기서 나가라, 아니면 계속하되 이제 널이 아님이 보장된다" — 이 패턴을 얼리 리턴(early return) 또는 가드 절(guard clause)이라 부르며, 코틀린에서 가장 널리 쓰이는 널 처리 관용구다.

> **명세 정밀**
> `?:`의 우변에 `Nothing` 타입 표현식(`throw`, `return`, `continue`, `break`, 또는 `Nothing`을 반환하는 함수 호출)을 놓을 수 있는 것은 특별 규칙이 아니라 *타입 시스템의 일반 규칙의 자연스러운 귀결*이다. `Nothing <: T`이므로 어디든 들어갈 수 있고, 최소 상계 계산이 알아서 좌변 타입으로 수렴한다. 이 우아함이 M32가 말하는 "엘비스는 병합 그 이상"의 정확한 의미다.

### 4.3 엘비스 체인과 우선순위

엘비스는 오른쪽 결합(right-associative)이다. 여러 개를 연달아 쓰면 왼쪽부터 첫 번째 널 아닌 값이 선택된다.

```kotlin
val result = a ?: b ?: c ?: "default"
// a가 non-null이면 a
// a가 null이고 b가 non-null이면 b
// … 모두 null이면 "default"
```

우선순위 함정 하나. 엘비스는 대부분의 이항 연산자보다 우선순위가 낮다. 그래서 산술이나 비교와 섞으면 의도와 달라질 수 있다.

```kotlin
val x: Int? = null
val y = x ?: 0 + 1
// ?: 가 + 보다 우선순위가 낮으므로 → x ?: (0 + 1) → x ?: 1
// => y == 1  (x가 null이므로)

// 의도가 (x ?: 0) + 1 이었다면 괄호가 필요
val z = (x ?: 0) + 1   // => 1
```

이 경우엔 우연히 결과가 같지만, `x`가 널이 아니었다면 달라진다. 애매하면 괄호로 명시하라.

### 4.4 `?:`로 예외 대신 기본값을 주는 안티패턴 주의

엘비스는 강력하지만, 모든 널을 기본값으로 삼키면 버그를 숨긴다. "설정 파일이 없으면 빈 문자열"과 "사용자가 없으면 즉시 실패"는 다른 결정이다. 널이 *정상적 부재*를 뜻하면 기본값(`?: emptyList()`)이 옳고, 널이 *있어선 안 되는 상태*를 뜻하면 `?: error("...")`로 시끄럽게 실패하는 편이 낫다. 엘비스의 우변에 무엇을 놓을지는 도메인 결정이다.

---

## 5. `!!`의 죄와 `as?`의 자비

### 5.1 `!!` — 널 아님 단언

널 아님 단언 연산자(not-null assertion operator) `!!`은 컴파일러에게 "이 값은 널이 아니라고 내가 보증한다"고 말하는 문법이다. `a!!`은 `a`가 널이 아니면 `a`를 그 널 불가 타입으로 낳고, 널이면 예외를 던진다.

```kotlin
val name: String? = readName()
val len: Int = name!!.length
// name이 non-null이면 length
// name이 null이면 NullPointerException을 던진다
```

`!!`은 널 가능 타입 `T?`를 널 불가 `T`로 강제 변환한다. 즉 컴파일러의 널 검사를 *개발자가 책임지고 우회*하는 장치다. JVM 백엔드에서 `!!`은 `Intrinsics.checkNotNull` 계열 호출로 내려가 널이면 예외를 던진다.

> **명세 정밀**
> `!!`이 던지는 예외는 `NullPointerException`이다. 역사적으로 코틀린은 `KotlinNullPointerException`(`NullPointerException`의 하위 클래스)을 던졌으나, 이후 일반 `java.lang.NullPointerException`을 던지도록 정리되었다(JVM 백엔드 기준). 실무에서 이 구분은 거의 중요하지 않다 — `!!`이 터졌다는 사실 자체가 문제다.

### 5.2 왜 "죄"인가

`!!`을 "죄"라 부르는 이유는, 그것이 코틀린이 제공하는 널 안전성 보증을 *스스로 반납*하는 유일한 순수-코틀린 경로이기 때문이다. `!!`을 쓰는 순간, 그 값에 대한 널 안전성은 컴파일러가 아니라 개발자의 어깨로 넘어온다. 그리고 개발자의 판단이 틀리면 런타임 NPE가 난다 — 바로 코틀린이 없애려던 그것이.

```kotlin
// 나쁨 — 두 개의 !!이 한 줄에. NPE가 나면 어느 것 때문인지 스택 트레이스가 애매하다
val v = map[key]!!.transform()!!

// 나음 — 널을 명시적으로 처리
val v = map[key]?.transform() ?: error("key '$key'가 없거나 변환 실패")
```

> **흔한 오해**
> "IDE가 `!!`을 자동으로 넣어 줬으니 안전하다"는 생각은 틀렸다. IDE의 자동 삽입은 "여기서 널 처리가 필요하다"는 신호일 뿐, `!!`이 최선의 처리라는 보증이 아니다. 대부분의 `!!`은 `?.`, `?:`, `?.let`, 또는 스마트 캐스트로 대체할 수 있다. `!!`이 정당한 경우는 극히 드물다 — 예컨대 초기화 순서상 컴파일러는 모르지만 개발자는 확실히 아는 불변식이 있을 때. 그때조차 `checkNotNull`이 더 나은 메시지를 준다.

### 5.3 `!!`이 정당화되는 드문 경우

그럼에도 `!!`이 합리적인 경우가 있다. 컴파일러가 흐름을 따라오지 못하지만 개발자는 불변식을 아는 상황이다.

```kotlin
val list = mutableListOf(1, 2, 3)
if (list.isNotEmpty()) {
    // 컴파일러는 firstOrNull()의 반환이 널 아님을 추론하지 못한다
    val first = list.firstOrNull()!!   // 논리적으론 안전하나, 취약하다
}

// 더 나은 방법
if (list.isNotEmpty()) {
    val first = list.first()           // 예외 의미가 명확하고 !! 불필요
}
```

대부분의 경우, `!!`이 필요해 보이면 더 나은 API(`first()` vs `firstOrNull()`)나 구조(엘비스 가드)가 있다. `!!`은 최후의 수단이다.

### 5.4 `as?` — 자비로운 캐스트

안전 캐스트 연산자(safe cast operator) `as?`는 캐스트가 실패하면 예외 대신 `null`을 낳는다. 일반 캐스트 `as`는 실패 시 `ClassCastException`을 던진다.

```kotlin
val obj: Any = "hello"

val s1: String = obj as String     // OK — 실제로 String
// val n = obj as Int              // ClassCastException — 실패 시 예외

val s2: String? = obj as? String   // "hello" (성공)
val n2: Int? = obj as? Int         // null (실패, 예외 없음)
```

`as?`의 결과 타입은 언제나 널 가능(`T?`)이다 — 실패하면 널이 되므로. 이것은 엘비스와 짝을 이뤄 "캐스트 시도, 실패하면 기본 처리"라는 관용구를 만든다.

```kotlin
fun describe(x: Any): String {
    val n = x as? Int ?: return "정수가 아님"
    return "정수 ${n * 2}"
}
```

`as?`는 특히 `equals` 구현에서 유용하다([[10 - 불리언과 동등성과 동일성]]).

```kotlin
override fun equals(other: Any?): Boolean {
    val that = other as? Point ?: return false  // 타입 안 맞으면 즉시 false
    return this.x == that.x && this.y == that.y
}
```

`other`가 `Point`가 아니면(또는 널이면) `as?`가 널을 낳고 엘비스가 `false`를 반환한다. 한 줄에 널 검사·타입 검사·조기 반환이 모두 담긴다.

---

## 6. 스마트 캐스트 — 제어 흐름을 읽는 컴파일러

### 6.1 원리: 흐름 민감 타입 좁히기

이제 이 장의 심장, 스마트 캐스트(smart cast)로 간다. 스마트 캐스트란, 컴파일러가 **제어 흐름을 분석해서 어떤 지점에서 변수의 타입을 자동으로 더 구체적인 타입으로 좁혀 주는** 기능이다. 널 안전성의 맥락에선, 널 검사를 통과한 뒤 변수를 널 불가 타입처럼 쓸 수 있게 해 주는 것이다.

```kotlin
fun printLength(s: String?) {
    if (s != null) {
        // 이 블록 안에서 s의 타입은 String? 가 아니라 String
        // 컴파일러가 "s != null"을 근거로 좁혔다 — 스마트 캐스트
        println(s.length)   // ?. 나 !! 없이 그냥 . 으로 접근 가능
    }
}
```

`if (s != null)`을 통과한 블록 안에서, 컴파일러는 `s`가 널이 아님을 *안다*. 그래서 `s`를 마치 `String` 타입인 것처럼 다룰 수 있다. 이것이 스마트 캐스트다. 여기엔 별도 문법이 없다 — 컴파일러가 흐름을 따라가며 알아서 해 준다. 그래서 "스마트"다.

이 분석은 흐름 민감(flow-sensitive)하다. 즉 코드의 *위치*에 따라 같은 변수의 타입이 달라진다.

```kotlin
fun demo(x: Any?) {
    // 여기서 x: Any?
    if (x is String) {
        // 여기서 x: String  — is 검사로 좁혀짐
        println(x.length)
    }
    // 여기서 다시 x: Any?  — if 밖으로 나오면 좁힘이 풀림
}
```

### 6.2 좁힘을 만드는 조건들

스마트 캐스트는 널 검사(`!= null`)뿐 아니라 타입 검사(`is`), 그리고 이 둘의 논리 조합에서 발동한다.

```kotlin
fun show(x: Any?) {
    // is 검사
    if (x is Int) println(x + 1)         // x: Int

    // && 로 이어지면 오른쪽에서 이미 좁혀진 타입 사용 가능
    if (x is String && x.length > 0)     // && 오른쪽에서 x: String
        println(x[0])

    // || 와 조기 반환의 결합
    if (x !is String) return             // 여기서 반환
    // return을 통과했으므로 이 아래에서 x: String
    println(x.uppercase())
}
```

마지막 패턴이 특히 강력하다. `if (x !is String) return`은 "String이 아니면 나가라"이므로, 그 아래로 흐름이 도달했다는 것은 곧 `x`가 `String`이라는 뜻이다. 컴파일러는 `return`이 흐름을 끊는다는 것(그 타입이 `Nothing`)을 이용해, 그 뒤에서 `x`를 `String`으로 좁힌다. 4.2에서 본 엘비스 가드와 같은 원리다.

### 6.3 스마트 캐스트가 만드는 교집합 타입

여러 검사를 통과하면, 스마트 캐스트는 개념적으로 교집합 타입(intersection type)을 만든다.

```kotlin
interface Named { val name: String }
interface Aged { val age: Int }

fun describe(x: Any) {
    if (x is Named && x is Aged) {
        // 여기서 x는 개념적으로 Named & Aged
        // 두 인터페이스의 멤버를 모두 접근 가능
        println("${x.name}, ${x.age}")
    }
}
```

K2 컴파일러는 이 교집합 타입을 내부적으로 정교하게 다룬다. 스마트 캐스트가 겹치면 `x`는 두 타입 모두의 멤버를 노출한다.

> **명세 정밀**
> 스마트 캐스트를 뒷받침하는 제어 흐름 분석은 K2 컴파일러의 프론트엔드(FIR)가 수행한다([[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]]). K2는 K1(구 컴파일러)보다 스마트 캐스트를 여러 면에서 개선했다. 예컨대 `||` 오른쪽 가지에서의 좁힘, 조기 반환 이후의 좁힘, 그리고 안정적인 `val` 프로퍼티에 대한 좀 더 넓은 좁힘 등이 K2에서 더 매끄럽다. 다만 6.4가 다룰 "근본적 불가" 조건들은 K2에서도 여전히 유효하다 — 그것들은 컴파일러의 게으름이 아니라 *안전성의 요구*이기 때문이다.

### 6.4 왜 좁힘은 블록을 벗어나면 풀리는가

스마트 캐스트는 "그 검사가 참임이 보장되는 흐름 구간"에서만 유효하다. `if` 블록을 벗어나면 검사의 보장이 끝나므로 좁힘도 풀린다(6.1의 마지막 예). 이것은 제약이 아니라 *정확성*이다 — 블록 밖에서는 값이 바뀌었을 수도 있으니까. 이 "바뀌었을 수도 있음"이 다음 절(M13)의 핵심이다.

---

## 7. 스마트 캐스트가 *안 되는* 곳 — M13의 정면 교정

### 7.1 오개념: "널 검사만 하면 스마트 캐스트가 된다"

M13은 "스마트 캐스트는 항상 된다"는 오개념이다. 초심자는 아래 코드가 왜 컴파일 에러인지 이해하지 못한다.

```kotlin
class Session {
    var token: String? = null   // var 프로퍼티

    fun use() {
        if (token != null) {
            // 컴파일 에러: Smart cast to 'String' is impossible,
            //   because 'token' is a mutable property that could have been
            //   changed by this time
            // println(token.length)
            println(token?.length)  // 여전히 ?. 가 필요하다
        }
    }
}
```

방금 `token != null`을 확인했는데 왜 `token.length`가 안 되는가? 컴파일러가 멍청해서가 아니다. **정확하기 때문**이다.

### 7.2 근본 원인: 검사와 사용 사이에 값이 바뀔 수 있는가

스마트 캐스트가 안전하려면, 컴파일러가 이렇게 확신할 수 있어야 한다. "검사한 그 값과, 지금 사용하는 그 값이 *같은 값*이다." `var` 프로퍼티는 이 확신을 깨뜨린다. `token`은 가변 프로퍼티이므로, `if (token != null)`과 `token.length` 사이에 — 예컨대 다른 스레드가, 혹은 이 객체를 참조하는 다른 코드가, 혹은 심지어 프로퍼티의 커스텀 게터가 — `token`을 널로 바꿨을 수 있다. 그러면 검사는 통과했는데 사용 시점엔 널인, 정확히 NPE를 낳는 상황이 된다. 컴파일러는 그 가능성을 배제할 수 없으므로 스마트 캐스트를 거부한다.

이것이 M13의 핵심이다. **스마트 캐스트는 "값이 검사 이후 바뀌지 않았음을 컴파일러가 증명할 수 있을 때"만 된다.** 증명할 수 없으면 안 된다. 스마트 캐스트 불가는 컴파일러의 무능이 아니라, 널 안전성 보증을 지키기 위한 필연적 결과다.

### 7.3 스마트 캐스트가 불가능한 목록

정리하면, 다음의 경우 스마트 캐스트가 되지 않는다.

| 대상 | 스마트 캐스트 | 이유 |
|------|:---:|------|
| 지역 `val` | 가능 | 재대입 불가 → 검사 후 안 바뀜 |
| 지역 `var` | 조건부 가능 | 검사와 사용 사이 수정 없고, 수정하는 람다에 캡처 안 됐으면 |
| `val` 프로퍼티 (같은 모듈, 커스텀 게터 없음, non-open) | 가능 | 안정적(stable) |
| `var` 프로퍼티 | **불가** | 언제든 다른 코드가 바꿀 수 있음 |
| 커스텀 게터를 가진 프로퍼티 | **불가** | 게터가 매번 다른 값을 줄 수 있음 |
| `open val` 프로퍼티 | **불가** | 하위 클래스가 커스텀 게터로 오버라이드 가능 |
| 다른 모듈의 프로퍼티 | **불가** | 컴파일 경계 밖 — 게터 유무 등을 알 수 없음 |
| 위임 프로퍼티(`by`) | **불가** | `getValue`가 매번 다른 값을 줄 수 있음 |
| 수정하는 람다에 캡처된 지역 `var` | **불가** | 람다가 언제 값을 바꿀지 모름 |

각각을 예로 보자.

```kotlin
// (1) 지역 var — 검사와 사용 사이 수정이 없으면 스마트 캐스트 됨
fun localVar(s: String?) {
    var x = s
    if (x != null) {
        println(x.length)   // OK — x는 지역 var이고 중간에 안 바뀜
    }
}

// (2) 커스텀 게터 — 매 접근마다 다른 값 가능
class Weird {
    val random: String?
        get() = if (System.nanoTime() % 2 == 0L) "x" else null
    fun use() {
        if (random != null) {
            // println(random.length)  // 컴파일 에러 — 게터가 다음 호출에 null을 줄 수 있음
        }
    }
}

// (3) 캡처된 var — 람다가 바꿀 수 있음
fun captured(s: String?) {
    var x = s
    val mutate = { x = null }
    if (x != null) {
        mutate()
        // println(x.length)  // 만약 스마트 캐스트가 됐다면 여기서 NPE
        // 그래서 컴파일러는 캡처된 var의 스마트 캐스트를 거부한다
    }
}
```

(2)의 커스텀 게터 예가 특히 시사적이다. `random`을 두 번 접근하면 두 번 다른 값이 나올 수 있으므로, `if (random != null)`에서 확인한 값과 `random.length`에서 접근하는 값은 *다른 값*이다. 컴파일러가 이를 스마트 캐스트하면 NPE를 낳을 수 있다. 이것이 `val`이라 해도 커스텀 게터가 있으면 스마트 캐스트가 안 되는 이유다 — `val`은 재대입 불가일 뿐, 매 접근이 같은 값을 준다는 보장이 아니다(이 오해는 M3, [[11 - 변수와 초기화 - val과 const와 lateinit]]과 이어진다).

### 7.4 우회법: 지역 변수에 담아라

스마트 캐스트가 안 되는 프로퍼티를 다룰 때, 가장 흔한 우회는 **값을 지역 `val`에 복사**하는 것이다.

```kotlin
class Session {
    var token: String? = null

    fun use() {
        val t = token            // 스냅샷을 지역 val에
        if (t != null) {
            println(t.length)    // OK — t는 지역 val, 스마트 캐스트 됨
        }
    }
}
```

`token`을 지역 `val t`에 담으면, `t`는 재대입 불가이므로 검사 후 절대 바뀌지 않는다. 그래서 스마트 캐스트가 된다. 이 우회는 흔하고 관용적이며, 부수 효과로 "그 시점의 값을 원자적으로 캡처"하는 올바른 동시성 의미까지 준다.

또 다른 우회는 `?.let`이나 엘비스 가드다.

```kotlin
class Session {
    var token: String? = null

    fun useA() = token?.let { println(it.length) }   // it은 non-null
    fun useB() {
        val t = token ?: return
        println(t.length)                            // t는 non-null
    }
}
```

`token?.let { ... }`은 `token`을 한 번 평가해서 널이 아니면 그 값을 `it`으로 넘기므로, 안에서 `it`은 안전하다. 이것이 4.1의 `?.let`과 4.2의 엘비스 가드가 널 가능 *프로퍼티*를 다룰 때 특히 유용한 이유다.

> **흔한 오해**
> "K2가 나왔으니 이제 `var` 프로퍼티도 스마트 캐스트된다"는 말을 종종 듣는다. 틀렸다. K2는 스마트 캐스트의 *범위*를 넓혔지만(안정적 값에 대한 추론 개선), `var` 프로퍼티·커스텀 게터·다른 모듈 프로퍼티의 근본적 불가는 그대로다. 그것들은 컴파일러가 값의 불변을 *증명할 수 없는* 경우이고, 그건 어떤 컴파일러 개선으로도 바뀌지 않는다. 안전을 위한 원리적 한계다.

---

## 8. 플랫폼 타입 `T!` — 벽에 난 유일한 합법적 구멍

### 8.1 문제: Java엔 널 가능성 정보가 없다

지금까지의 모든 널 안전성은 "코틀린이 타입에 널 가능성을 안다"는 전제 위에 섰다. 그런데 코틀린은 Java와 완벽히 상호운용된다([[45 - Java 상호운용]]). 그리고 Java의 타입 `String`은 널 가능성 정보가 *없다*. Java 메서드가 `String`을 반환할 때, 그것이 널일 수 있는지 없는지 타입만으론 알 수 없다.

코틀린은 이 딜레마 앞에서 두 극단을 모두 피한다. 만약 Java의 모든 `String`을 `String?`로 취급하면, Java API를 쓸 때마다 널 처리가 폭발해 상호운용이 고통스러워진다. 반대로 모두 `String`(널 불가)으로 취급하면, 실제로 널이 올 때 널 안전성 보증이 거짓말이 된다. 코틀린의 선택은 제3의 길 — **플랫폼 타입(platform type)** 이다.

### 8.2 `T!` — 널 검사를 유예한 타입

Java에서 넘어온, 널 가능성이 표기되지 않은 타입을 코틀린은 **플랫폼 타입** `T!`로 표현한다. `String!`은 "널일 수도, 아닐 수도 있는데 컴파일러가 검사를 *유예*하는 String"이다.

```java
// Java 코드
public class UserRepository {
    public String findName(int id) {
        return id == 0 ? null : "Alice";   // 널을 반환할 수도 있다
    }
}
```

```kotlin
// Kotlin에서 호출
val repo = UserRepository()
val name = repo.findName(1)   // name의 타입: String!  (플랫폼 타입)

// 플랫폼 타입은 널 불가처럼도, 널 가능처럼도 쓸 수 있다 — 컴파일러가 침묵한다
val len1 = name.length        // 컴파일 통과 (널 검사 안 함) — 위험!
val len2 = name?.length       // 이것도 통과 (널 가능처럼 다룸) — 안전
```

핵심은 `name.length`가 **컴파일 에러가 아니라는** 것이다. 컴파일러는 플랫폼 타입에 대해 널 검사를 요구하지 않는다. 검사를 개발자에게 유예한다. `findName(1)`이 실제로 널을 반환했다면, `name.length`에서 런타임 NPE가 난다.

> **명세 정밀**
> `T!`은 소스 코드에 직접 쓸 수 없는 표기다 — 컴파일러 내부와 IDE의 타입 힌트, 에러 메시지에만 나타난다. 이것은 "널 불가 `T`와 널 가능 `T?`를 모두 부분타입으로 갖는" 유연 타입(flexible type)이다. 개념적으로 `T!`은 구간 $[T, T?]$을 나타낸다 — 그 구간의 어느 타입으로든 쓰일 수 있다. 그래서 널 불가 문맥에도, 널 가능 문맥에도 대입된다.

### 8.3 M8 교정: `T!`은 안전하지 않다

M8은 "플랫폼 타입 `T!`는 안전하다"는 오개념이다. 정반대다. **플랫폼 타입은 코틀린 널 안전성의 벽에 난 유일한 합법적 구멍이며, 순수 코틀린 코드에서 NPE가 나는 가장 흔한 실제 경로다.** 컴파일러가 침묵하는 것은 "안전해서"가 아니라 "정보가 없어서 검사를 유예한 것"이다. 그 유예된 검사의 책임은 개발자에게 넘어온다 — `!!`과 똑같이.

위험이 특히 교활한 이유는, 플랫폼 타입이 *전파*되기 때문이다.

```kotlin
val name = repo.findName(1)     // String!
val upper = name.uppercase()    // 이것도 String! (플랫폼 타입 전파)
// 어디서도 컴파일러가 경고하지 않지만, findName이 null을 반환하면
// name.uppercase()에서 NPE
```

### 8.4 방어: 경계에서 명시적 타입으로 못박아라

플랫폼 타입을 다루는 올바른 방법은, Java 세계와 코틀린 세계의 *경계에서* 타입을 명시적으로 선언하는 것이다. 명시적 타입을 붙이면, 그 지점에서 컴파일러가 널 검사를 삽입한다.

```kotlin
// 나쁨 — 플랫폼 타입을 그대로 흘려보냄
val name = repo.findName(1)      // String! — 검사 유예, 위험 전파

// 좋음 (A) — 널 가능으로 못박기: 이제 코틀린 널 안전성이 다시 작동
val nameNullable: String? = repo.findName(1)
// nameNullable.length          // 컴파일 에러 — 널 검사를 강제받는다
println(nameNullable?.length)   // 안전

// 좋음 (B) — 널 불가로 못박기: 경계에서 즉시 검사, 널이면 여기서 NPE
val nameNotNull: String = repo.findName(1)
// ↑ findName이 null을 반환하면 이 대입 지점에서 NPE (Intrinsics.checkNotNull)
println(nameNotNull.length)     // 여기 도달했다면 안전이 보장됨
```

(B)가 미묘하지만 중요하다. `val nameNotNull: String = repo.findName(1)`에서, 컴파일러는 플랫폼 타입 값을 널 불가 타입에 대입하므로 **그 대입 지점에 널 검사 코드를 삽입**한다. 그래서 널이면 나중에 엉뚱한 곳이 아니라 바로 그 대입 줄에서 NPE가 난다. 이것이 "경계에서 못박기"의 이점이다 — NPE가 나더라도 발생 지점이 널의 근원에 가깝다.

### 8.5 널 애노테이션을 존중한다

Java 코드가 널 가능성 애노테이션(`@Nullable`, `@NotNull` 등 — JetBrains, JSR-305, Android, javax 등 다양한 표준)을 달고 있으면, 코틀린은 그것을 존중해서 플랫폼 타입 대신 정확한 널 가능성을 부여한다.

```java
// Java — 애노테이션을 단 경우
import org.jetbrains.annotations.Nullable;
import org.jetbrains.annotations.NotNull;

public class AnnotatedRepo {
    @Nullable public String findName(int id) { ... }   // 코틀린에서 String?
    @NotNull  public String getVersion()     { ... }   // 코틀린에서 String
}
```

```kotlin
val n = AnnotatedRepo().findName(1)   // 타입: String?  (플랫폼 타입 아님!)
// n.length                           // 컴파일 에러 — 정상적으로 널 검사 강제
val v = AnnotatedRepo().getVersion()  // 타입: String  (널 불가로 신뢰)
```

그래서 잘 애노테이션된 Java 라이브러리는 코틀린에서 마치 코틀린 API처럼 안전하게 쓰인다. 플랫폼 타입의 위험은 *애노테이션이 없는* Java 코드에서만 발생한다. 이 상호운용의 전모(애노테이션 표준, 컬렉션 매핑, `@Jsr305` 컴파일러 옵션 등)는 45장이 소유한다.

---

## 9. 컴파일 타임 vs 런타임 — M7의 정면 교정

### 9.1 오개념: "널 안전성은 런타임 검사다"

M7은 "널 안전성은 런타임 검사다"라는 오개념이다. Java 배경 개발자는 널 안전성을 "곳곳에 자동으로 `if (x == null) throw ...`가 삽입되는 런타임 방어"로 상상하곤 한다. 이 그림은 틀렸다.

진실은 이렇다. **코틀린 널 안전성의 절대 다수는 컴파일 타임에 완결된다.** `?.`, `?:`, 스마트 캐스트, 널 가능 타입에 대한 역참조 금지 — 이 모두는 컴파일러가 *컴파일 시점에* 검사하고 강제하는 규칙이다. `s.length`가 `s: String?`에서 컴파일 에러인 것은 런타임에 뭔가가 터지는 게 아니라, *애초에 컴파일이 안 되는 것*이다. 잘못된 코드는 실행 파일이 되지 못한다.

### 9.2 런타임 널 검사는 "경계"에만 있다

그럼 런타임 널 검사는 언제 삽입되는가? 코틀린이 컴파일 타임에 *보장할 수 없는* 경계에만 삽입된다. 그 경계 목록은 이 장 전체를 관통해 온 것과 정확히 일치한다.

```text
런타임에 NPE가 날 수 있는 경로 (순수 코틀린 관점)
────────────────────────────────────────────────
1. !!            → 개발자가 명시적으로 검사를 우회 (5절)
2. 플랫폼 타입 T!  → Java 경계에서 유예된 검사 (8절)
   - 널 불가 타입에 대입할 때 컴파일러가 checkNotNull 삽입
   - 널 불가 파라미터로 넘길 때도 삽입
3. 리플렉션/제네릭 소거 우회 → 타입 검사를 런타임에 속임 (44장)
4. 미초기화 lateinit 접근 → UninitializedPropertyAccessException (엄밀히는 NPE 아님, 11장)
5. 생성자에서 미초기화 this 누출 → open 멤버 함정 (24장)
```

이 목록의 공통점은 하나다 — **모두 컴파일러가 "이 값이 널이 아님"을 증명할 수 없는 경계**라는 것. 그 경계에서만 컴파일러는 런타임 검사(`Intrinsics.checkNotNull` 등)를 삽입하거나, 개발자에게 책임을 넘긴다. 경계 안쪽의 순수 코틀린 코드에서는 널 검사가 애초에 필요 없다 — 타입 시스템이 이미 컴파일 타임에 널을 배제했으므로.

```kotlin
// 순수 코틀린 — 런타임 널 검사가 필요 없다. 컴파일 타임에 이미 안전
fun pureKotlin(s: String): Int = s.length   // s는 절대 널일 수 없음 (타입이 보장)
// 이 함수 본문에는 널 검사 바이트코드가 없다

// 단, public 함수의 파라미터는 방어적 검사가 삽입될 수 있다
// (Java에서 널을 넘길 수 있으므로 — 경계 방어)
```

> **명세 정밀**
> JVM 백엔드에서, `public`/`internal` 함수의 널 불가 파라미터에는 컴파일러가 진입부에 `Intrinsics.checkNotNullParameter` 호출을 삽입한다. 왜냐하면 그 함수는 Java에서도 호출될 수 있고, Java는 널 불가 계약을 어길 수 있기 때문이다. 이것은 "경계 방어"의 일종이다 — 코틀린 세계 내부는 컴파일 타임에 안전하지만, 외부(Java)와의 접점에는 런타임 방어벽을 세운다. `private` 함수엔 대개 이 검사가 없다(외부에서 못 부르므로). 이 동작은 컴파일러 옵션으로 조정 가능하다.

### 9.3 `if` 스마트 캐스트 vs `?.let` — 미묘한 차이

3.4에서 미룬 비교를 여기서 회수하자. "널이 아니면 실행"을 표현하는 두 방식은 미묘하게 다르다.

```kotlin
// 방식 A — if + 스마트 캐스트
val name: String? = getName()
if (name != null) {
    println(name.length)   // name이 스마트 캐스트로 String
}

// 방식 B — ?.let
getName()?.let {
    println(it.length)     // it이 String
}
```

방식 A는 지역 `val name`에 대해 스마트 캐스트가 작동한다. 하지만 `name`이 스마트 캐스트 불가 대상(예: `var` 프로퍼티)이면 A는 컴파일 에러가 나고, 그때 B(`?.let`)가 유용하다 — `?.let`은 값을 `it`으로 캡처하므로 프로퍼티의 가변성과 무관하게 안전하기 때문이다. 반대로, 성능이 극도로 민감한 코드에서 B는 람다 객체 할당을 유발할 수 있다(인라인되면 없어지지만 — `let`은 인라인 함수이므로 대개 할당이 없다, [[15 - 인라인 함수와 reified]]). 대부분의 경우 둘은 취향이지만, "스마트 캐스트가 안 되는 프로퍼티"라는 구체적 상황에선 B가 정답이 된다.

### 9.4 contract — 널 검사를 함수 밖으로 실어 나르기

스마트 캐스트는 컴파일러가 *직접 보는* 흐름에만 작동한다. 그런데 널 검사를 *함수로 추출*하면 컴파일러가 그 안을 못 보므로 스마트 캐스트가 끊긴다.

```kotlin
fun isNotNull(x: Any?): Boolean = x != null

fun use(s: String?) {
    if (isNotNull(s)) {
        // println(s.length)  // 컴파일 에러 — 컴파일러는 isNotNull이 무엇을
                              //   보장하는지 모른다. 스마트 캐스트 안 됨
    }
}
```

이 간극을 메우는 것이 **contract**다. contract는 함수가 "내가 참을 반환하면 이 인자는 널이 아니다" 같은 약속을 컴파일러에게 선언하는 실험적 기능이다.

```kotlin
import kotlin.contracts.ExperimentalContracts
import kotlin.contracts.contract

@OptIn(ExperimentalContracts::class)
fun isNotNull(x: Any?): Boolean {
    contract {
        returns(true) implies (x != null)   // true를 반환하면 x는 널이 아님을 약속
    }
    return x != null
}

fun use(s: String?) {
    if (isNotNull(s)) {
        println(s.length)   // 이제 OK — contract 덕분에 스마트 캐스트 작동
    }
}
```

이 기능이 실험적인 것은 사용자 정의 contract API의 문법이 아직 안정화 단계이기 때문이다. 그러나 표준 라이브러리는 이미 contract를 광범위하게 쓴다 — 그래서 우리는 매일 그 혜택을 받는다.

```kotlin
// 표준 라이브러리 함수들은 contract로 스마트 캐스트를 흘려보낸다
val s: String? = getName()

if (s.isNullOrEmpty()) return
println(s.length)   // OK — isNullOrEmpty()의 contract가
                    //   "false를 반환하면 s는 널 아님"을 약속

requireNotNull(s)   // 널이면 예외, 통과하면 s를 String으로 스마트 캐스트
checkNotNull(s)     // 마찬가지 (다른 예외 타입)

// requireNotNull/checkNotNull은 값도 반환하므로 이렇게도 쓴다
val notNull: String = requireNotNull(s) { "이름이 필요합니다" }
```

`s.isNullOrEmpty()`가 `false`를 반환하면 그 뒤에서 `s`가 널이 아니라고 컴파일러가 아는 것은, `isNullOrEmpty`의 시그니처에 `returns(false) implies (this@isNullOrEmpty != null)` contract가 붙어 있기 때문이다. contract는 "스마트 캐스트를 함수 경계 너머로 실어 나르는" 다리다.

> **역사 메모**
> contract는 코틀린 1.3(2018)에서 실험적으로 도입됐다. 목적은 정확히 이것 — 표준 라이브러리의 `require`/`check`/`isNullOrEmpty`/`isNullOrBlank` 같은 함수가 스마트 캐스트에 참여하게 만드는 것. 사용자 정의 contract는 여전히 opt-in 실험 상태지만, 표준 라이브러리 내부의 contract는 안정적으로 작동하며 매일 쓰인다. contract는 스마트 캐스트(6절)의 사정거리를 함수 호출 너머로 확장한 것으로 이해하면 정확하다.

---

## 10. 널 가능성이 흘러 다니는 문맥들

### 10.1 널 가능 불리언 — `if`에 바로 못 넣는다

`Boolean?`은 미묘한 함정이다. `if`의 조건은 `Boolean`이어야 하므로, `Boolean?`을 바로 넣을 수 없다.

```kotlin
val flag: Boolean? = null
// if (flag) { ... }        // 컴파일 에러 — Boolean?는 조건이 될 수 없다

if (flag == true) { ... }   // null이나 false면 실행 안 함
if (flag != false) { ... }  // null이나 true면 실행 (null 처리에 주의!)
if (flag == true) { ... } else { ... }  // 명시적
```

`flag == true`는 널을 안전하게 처리한다 — `flag`가 널이면 `null == true`는 `false`다([[10 - 불리언과 동등성과 동일성]]). "널이면 어느 쪽으로 취급할지"를 `== true`/`!= false`로 명시하는 것이 관용구다.

### 10.2 널 병합과 컬렉션

컬렉션의 널 가능 원소를 걸러내는 표준 관용구.

```kotlin
val items: List<String?> = listOf("a", null, "b", null)

val nonNull: List<String> = items.filterNotNull()  // [a, b] — 타입도 String으로 좁혀짐
val count = items.count { it != null }              // 2

// 맵에서 값 꺼내기 — 없는 키는 null
val map = mapOf("x" to 1)
val v: Int = map["x"] ?: 0        // 없으면 0
val w: Int = map.getOrDefault("y", -1)  // 없으면 -1
```

`filterNotNull()`은 널을 제거하면서 원소 타입을 `String?`에서 `String`으로 좁힌다 — 널 가능성이 컬렉션 연산과 결합하는 대표 예다([[39 - 컬렉션 연산과 함수형 파이프라인]]).

### 10.3 `lateinit`은 널이 아니다 — 혼동 금지

이 장을 닫기 전에, 자주 혼동되는 경계 하나를 못박자. `lateinit`은 **널 가능성과 무관**하다.

```kotlin
class Service {
    lateinit var repo: Repository   // 널 가능이 아니다! Repository (널 불가)

    fun init() { repo = Repository() }
    fun use() {
        // repo가 초기화 전이면:
        // UninitializedPropertyAccessException (NPE가 아니다!)
        repo.query()
    }
}
```

`lateinit var repo: Repository`는 "널일 수 있는 프로퍼티"가 아니라 "초기화가 늦은, 그러나 널일 수 없는 프로퍼티"다(M9). 접근 전에 초기화되지 않았으면 `NullPointerException`이 아니라 `UninitializedPropertyAccessException`이 난다. 즉 "값이 널"과 "값이 아직 없음"은 코틀린에서 다른 개념이고, 다른 예외를 낳는다. `lateinit`의 정본 해부와 `::repo.isInitialized` 검사는 [[11 - 변수와 초기화 - val과 const와 lateinit]]이 소유한다. 여기서는 "`lateinit`을 널 안전성의 우회로 착각하지 말라"는 경계만 세운다.

### 10.4 `!!`, 플랫폼 타입, lateinit — NPE의 세 얼굴

이 장이 다룬 "런타임에 터질 수 있는" 세 경로를 나란히 놓으면 성격이 뚜렷해진다.

| 경로 | 예외 | 성격 | 예방 |
|------|------|------|------|
| `!!` | `NullPointerException` | 개발자가 명시적으로 검사 우회 | `?.`, `?:`, `?.let`로 대체 |
| 플랫폼 타입 `T!` | `NullPointerException` | Java 경계의 유예된 검사 | 경계에서 명시적 타입, 애노테이션 |
| 미초기화 `lateinit` | `UninitializedPropertyAccessException` | 초기화 누락(널과 무관) | `isInitialized`, 생성자 초기화 |

셋 다 "코틀린이 컴파일 타임에 보장할 수 없는 경계"라는 점에서 M7의 예외 목록에 속한다. 그리고 셋 다 *예외적 경로*라는 점에서, 널 안전성의 기본값이 여전히 "컴파일 타임에 막힘"임을 재확인시킨다.

---

지금까지의 궤적을 한 줄기로 회수하자. 널은 타입에 표시되지 않은 채 수십 년간 시스템을 무너뜨린 십억 달러의 실수였고, 코틀린은 그것을 라이브러리가 아니라 *타입 시스템*에서 풀었다. `String`과 `String?`은 다른 타입이며, 그 사이엔 `String <: String?`이라는 부분타입 관계가 있다. 널 가능 값을 다루는 `?.`은 널을 전파하고, `?:`은 단순 병합이 아니라 `Nothing` 타입 덕분에 `throw`/`return`을 우변에 실어 제어 흐름을 표현하며(M32), `!!`은 그 모든 보증을 개발자가 반납하는 최후의 수단이다. 스마트 캐스트는 컴파일러가 흐름을 읽어 타입을 좁히는 기계지만, `var` 프로퍼티·커스텀 게터·다른 모듈처럼 "값이 검사 후 바뀌지 않았음을 증명할 수 없는" 곳에선 작동하지 않으며(M13), 그 불가는 무능이 아니라 안전의 필연이다. Java 세계에서 널 정보 없이 넘어오는 플랫폼 타입 `T!`은 벽에 난 유일한 합법적 구멍이고, 안전한 것이 아니라 검사가 유예된 것이므로 경계에서 명시적 타입으로 못박아야 한다(M8). 그리고 이 모든 검사의 절대 다수는 — 도입부의 첫 질문에 답하자면 — **런타임이 아니라 컴파일 타임에 완결된다**(M7). 런타임 NPE는 `!!`·플랫폼 타입·리플렉션이라는 좁은 경계에만 산다. 널 안전성이란 결국, "이 값이 널일 수 있는가?"라는 질문의 답을 개발자의 머릿속에서 컴파일러의 타입 검사로 옮긴 것이다.

## 핵심 요약

- **코틀린은 널 안전성을 라이브러리(`Optional`)가 아니라 타입 시스템으로 푼다.** `String`과 `String?`은 서로 다른 타입이고, 널 불가 타입에 `null`을 대입하거나 널 가능 타입을 검사 없이 역참조하는 것은 *컴파일 에러*다.
- **널 안전성의 절대 다수는 컴파일 타임에 완결된다 (M7).** 런타임 NPE는 `!!`, 플랫폼 타입(Java 상호운용), 리플렉션 우회라는 좁은 경계에만 존재한다. 순수 코틀린 코드에서 아무 우회 없이 NPE가 나는 것은 구조적으로 불가능하다.
- **`T`는 언제나 `T?`의 부분타입이다 ($T <: T?$).** `null` 리터럴의 타입은 `Nothing?`이며, `Any`는 널을 담지 못하고 `Any?`만이 널까지 포함하는 진짜 최상위다.
- **안전 호출 `?.`은 널을 전파한다.** 수신 객체가 널이면 호출을 건너뛰고 `null`을 낳으며, 결과 타입은 언제나 널 가능해진다. 체인은 첫 널에서 멈추고, 각 수신 객체는 한 번만 평가된다.
- **엘비스 `?:`은 널 병합이 아니라 제어 흐름이다 (M32).** 우변에 `throw`/`return`을 놓을 수 있는 것은 그것들의 타입이 바닥 타입 `Nothing`(M11)이어서, 최소 상계가 좌변의 널 불가 타입으로 수렴하기 때문이다.
- **`!!`은 널 안전성 보증을 개발자가 반납하는 최후의 수단이다.** 널이면 `NullPointerException`을 던지며, 대부분 `?.`/`?:`/`?.let`으로 대체할 수 있다. `as?`는 캐스트 실패 시 예외 대신 널을 낳아 널 검사와 타입 검사를 한 번에 처리한다.
- **스마트 캐스트는 컴파일러가 제어 흐름을 읽어 타입을 좁히는 것이다.** `if (x != null)`이나 `if (x is T)` 이후, 그 흐름 구간에서 `x`는 좁혀진 타입으로 쓰인다. K2는 이 분석을 K1보다 넓혔다.
- **`var` 프로퍼티·커스텀 게터·다른 모듈·위임 프로퍼티는 스마트 캐스트가 안 된다 (M13).** 검사와 사용 사이에 값이 바뀌지 않았음을 컴파일러가 *증명할 수 없기* 때문이다. 불가는 무능이 아니라 안전의 필연이며, 지역 `val`에 복사하거나 `?.let`으로 우회한다.
- **플랫폼 타입 `T!`은 안전하지 않다 (M8).** Java에서 널 정보 없이 넘어온 타입으로, 컴파일러가 검사를 *유예*할 뿐이다. 순수 코틀린에서 NPE가 나는 가장 흔한 실제 경로이며, 경계에서 명시적 타입(`String?`/`String`)으로 못박아야 한다. 널 애노테이션이 있으면 정확한 타입으로 매핑된다.
- **contract는 스마트 캐스트를 함수 경계 너머로 실어 나른다.** `requireNotNull`, `checkNotNull`, `isNullOrEmpty` 같은 표준 라이브러리 함수는 contract로 스마트 캐스트에 참여한다. 사용자 정의 contract는 아직 실험적이다.
- **`lateinit`은 널 가능성과 무관하다 (M9 참조).** 초기화가 늦은 널 *불가* 프로퍼티이며, 미초기화 접근은 NPE가 아니라 `UninitializedPropertyAccessException`을 낳는다. "값이 널"과 "값이 아직 없음"은 다른 개념이다.
- **제네릭 타입 파라미터 `T`의 기본 상한은 `Any?`라 이미 널을 담는다 (M45 참조).** 널을 배제하려면 `T : Any`로 상한을 명시해야 한다.

## 연결 노트

- [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]] — 이 장의 토대. `T?`의 부분타입 위상, `Nothing`이 왜 바닥 타입인지, `Any` vs `Any?`의 구분을 소유한다. 엘비스 우변의 `Nothing`(M32)이 여기서 온다.
- [[41 - 예외와 Nothing]] — `throw`가 `Nothing` 타입이라는 사실(M11)의 정본. 엘비스 가드 `?: throw`와 조기 반환 스마트 캐스트가 이 성질에 기댄다.
- [[10 - 불리언과 동등성과 동일성]] — `==`가 널을 안전하게 비교하는 규칙(M6), `Boolean?`을 `== true`로 다루는 관용구, `equals`에서의 `as?` 패턴을 소유한다.
- [[11 - 변수와 초기화 - val과 const와 lateinit]] — `lateinit`(M9)과 `by lazy`의 정본. `val`이 불변이 아니라 재대입 불가일 뿐(M3)이라는 점이 커스텀 게터 스마트 캐스트 불가와 이어진다.
- [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]] — `Int?`가 왜 대개 박싱되는지(M5). `?.`의 널 가능 원시 타입 결과가 JVM 백엔드에서 박싱되는 이유가 여기 있다.
- [[45 - Java 상호운용]] — 플랫폼 타입 `T!`(M8)의 전모: 널 애노테이션 표준, 컬렉션 매핑, 경계 방어 검사, `!!`이 흔해지는 상호운용의 현실을 소유한다.
- [[33 - 제네릭 1 - 타입 파라미터와 변성]] — 타입 파라미터의 기본 상한이 `Any?`라 `T`가 이미 널 가능하다는 것(M45)의 정본. 널 배제는 `T : Any`.
- [[16 - 스코프 함수와 수신 객체 관용구]] — `?.let`이 널 처리 관용구로 쓰이는 원리. 널 가능 프로퍼티를 스마트 캐스트 없이 안전하게 다루는 우회로.
- [[17 - 표현식으로서의 제어 흐름 - if와 when]] — 스마트 캐스트가 `when(x)`의 `is` 분기와 결합해 타입을 좁히는 방식. 엘비스의 제어 흐름 성격이 표현식 관점과 이어진다.
- [[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]] — 스마트 캐스트를 뒷받침하는 제어 흐름 분석을 K2 프론트엔드(FIR)가 수행한다. `?.`/`!!`이 바이트코드로 내려가는 과정의 배경.
