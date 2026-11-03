---
title: 타입 안전 빌더와 DSL
date: 2026-07-13
tags: [kotlin, dsl, builder, dslmarker, receiver, 학습노트]
---

[[16 - 스코프 함수와 수신 객체 관용구]]에서는 `apply`/`run`/`with`가 모두 **수신 객체 지정 람다**(`T.() -> R`)라는 하나의 언어 기능을 바탕으로 만들어졌다는 것을 보았다. [[35 - 확장 함수와 프로퍼티]]에서는 확장이 정적으로 디스패치되는 "겉보기 멤버"라는 점을 살펴보았다.

[[13 - 함수 타입과 람다와 함수 참조]]에서는 함수가 일급 값이고 후행 람다(trailing lambda) 문법 덕분에 코드가 간결해진다는 것을, [[15 - 인라인 함수와 reified]]에서는 람다 호출을 인라이닝으로 없애는 방법을 다뤘다.

이 장에서는 이 기능들을 **하나로 합쳐 하나의 결과물**을 만든다. 코틀린이라는 범용 언어 안에서 특정 문제 영역(HTML, 빌드 설정, 상태 기계, 쿼리)에 맞춘 **작은 언어처럼 읽히는 API**를 만드는 기법, 즉 타입 안전 빌더(type-safe builder)와 내부 DSL(internal DSL)이다.

DSL이 어떤 언어 기능의 조합으로 만들어지는지, 그 조합이 왜 "타입 안전"한지, 그리고 어디서 허점이 생기는지를 차례로 다룬다. 연산자 관례 이름의 전체 목록은 [[19 - 연산자 오버로딩과 관례]]에서 설명한다.

---

## 1. DSL이란 무엇인가: 내부 DSL과 호스트 언어

### 1.1 도메인 특화 언어의 두 종류

**DSL(Domain-Specific Language)** 은 특정 문제 영역에 맞춰 표현력을 좁히는 대신, 그 영역에서의 간결함과 안전성을 최대한 높인 언어다. 범용 언어(GPL, general-purpose language)는 무엇이든 표현할 수 있지만 특정 영역에서는 코드가 장황해진다. DSL은 그 반대를 택한다. 정규표현식, SQL, CSS, HTML, `Makefile`, `.gitignore` 문법이 모두 DSL이다.

DSL은 구현 방식에 따라 크게 두 가지로 나뉜다.

```text
┌─────────────────────────────────────────────────────────────┐
│ 외부 DSL (external DSL)                                       │
│   독립된 문법 + 전용 파서/렉서/인터프리터                     │
│   예: SQL, 정규식, JSON, YAML, Gradle Groovy(부분)           │
│   장점: 문법을 완전히 자유 설계                               │
│   단점: 파서를 짜야 함, 호스트 언어와 단절(문자열로 오감),   │
│         에러가 "런타임 파싱 실패"로 늦게 드러남              │
├─────────────────────────────────────────────────────────────┤
│ 내부 DSL (internal / embedded DSL)                            │
│   호스트 언어의 "부분집합처럼 보이는" 정상적 코드             │
│   예: Kotlin html{}, Gradle Kotlin DSL, kotlinx 여러 빌더    │
│   장점: 파서 불필요(컴파일러가 이미 함), 타입검사·자동완성   │
│         ·리팩터링·디버깅이 전부 공짜로 따라옴                 │
│   단점: 호스트 언어 문법의 한계 안에서만 설계 가능           │
└─────────────────────────────────────────────────────────────┘
```

코틀린에서 만드는 것은 거의 항상 **내부 DSL**이다. `html { ... }`을 실행할 때 새로운 파서가 동작하는 것이 아니다. 이 코드는 완전히 평범한 코틀린 함수 호출이고, 여러 문법 설탕이 겹쳐서 *선언형 마크업 언어처럼 읽히도록* 배열되었을 뿐이다.

여기서 중요한 점은 **DSL은 언어 기능이 아니라 API 설계 기법이다.** 코틀린에는 "DSL 키워드"가 없다(`@DslMarker` 하나가 예외적인 보조 장치일 뿐이다). DSL은 함수, 람다, 수신자, 확장, 연산자처럼 이미 있는 기능을 특정한 방식으로 조합했을 때 *새롭게 나타나는 성질*이다.

> [!info] 역사 메모
> "타입 안전 빌더(type-safe builder)"라는 용어는 코틀린 공식 문서에서 소개하면서 널리 쓰이게 되었다. 이 발상은 Groovy의 빌더(예: `MarkupBuilder`)에서 왔다. Groovy 빌더는 `methodMissing` 같은 동적 디스패치로 임의의 태그 이름을 받아들였다. 이 방식은 깔끔하지만 오타가 런타임까지 발견되지 않는다(`bdoy { }`라고 써도 컴파일된다).
>
> 코틀린은 같은 겉모습을 **정적 타입**으로 구현했다. 존재하지 않는 태그는 컴파일되지 않는다. 이름도 여기서 나왔다. 같은 빌더 패턴에 타입 시스템의 보장을 더했다는 뜻이다.

### 1.2 코틀린이 내부 DSL에 특히 유리한 이유

내부 DSL의 품질은 호스트 언어가 "문법적인 잡음을 얼마나 줄일 수 있는가"에 달려 있다. 코틀린에는 이런 잡음을 줄이는 기능이 여러 개 있고, 이 기능들이 함께 작용해 DSL을 만들 수 있다.

| 언어 기능 | DSL에 기여하는 것 | 관련 장 |
|-----------|-------------------|---------|
| 후행 람다(trailing lambda) | `f({ ... })` → `f { ... }`. 괄호가 사라져 블록처럼 보임 | 13장 |
| 수신 객체 지정 람다 `T.() -> R` | 블록 안에서 `this`가 바뀌어 태그의 멤버를 이름만으로 호출 | 16장 |
| 확장 함수/프로퍼티 | 다른 타입에 어휘를 덧붙임(파일을 열지 않고 DSL 확장) | 35장 |
| 중위 호출 `infix` | `1 to "a"`, `state on Event` 같은 자연어에 가까운 이항 표현 | 이 장 §6 |
| 연산자 오버로딩 | `+"text"`(unaryPlus), `config["key"]`(get), `route(...)`( invoke) | 19장 |
| 세미콜론 추론 | 줄바꿈이 문장을 끝냄 → 마크업처럼 한 줄에 한 요소 | [[03 - 렉시컬 구조와 세미콜론 추론]] |
| 기본 인자·이름 붙은 인자 | 선택적 속성을 팩토리 함수를 늘리지 않고 자연스럽게 표현 | [[12 - 함수 - 인자와 vararg와 지역 함수와 꼬리 재귀]] |
| 람다 인라이닝 `inline` | 위 문법 설탕의 런타임 비용을 상당 부분 제거 | 15장 |

표의 각 기능은 하나씩 보면 대단하지 않다. 후행 람다 하나만으로는 DSL을 만들 수 없다. 그러나 이 기능들이 **겹치면** 결과가 질적으로 달라진다. 다음 한 줄을 보자.

```kotlin
button("저장") { onClick { save() } }
```

이 코드에서 문법 설탕을 하나씩 걷어 내고 원래 형태로 되돌리면 다음과 같다.

```kotlin
button("저장", block = { it: Button ->
    it.onClick(handler = { it2: ClickContext ->
        it2.save()
    })
})
```

두 코드를 비교하면, DSL은 결국 **후행 람다 + 수신 객체(암시적 `this`) + 확장/멤버 호출**이 여러 겹으로 쌓이면서 괄호와 `it`, `->`가 보이지 않게 된 것임을 알 수 있다. 새로운 언어가 아니라, 같은 코드를 **시각적으로 더 압축한 형태**다.

### 1.3 "빌더"라는 이름의 의미

빌더 패턴(builder pattern)은 고전적인 OOP에서 나온 패턴이다. 인자가 많고 그중 상당수가 선택적인 객체를 생성자 하나로 만들지 않고, 단계적으로 조립한다. 자바에서는 보통 다음과 같이 작성한다.

```java
// Java: 전통적 빌더
Pizza p = new Pizza.Builder()
    .size(12)
    .addTopping("cheese")
    .addTopping("basil")
    .build();
```

코틀린에도 이런 fluent 체이닝 빌더가 있지만, 코틀린의 강점은 한 단계 더 나아간 **DSL식 빌더**다. 메서드 체인 대신 **중첩 블록**을 사용한다.

```kotlin
// Kotlin: DSL 빌더 — 중첩 블록이 구조를 그대로 그림
val p = pizza {
    size = 12
    topping("cheese")
    topping("basil")
}
```

두 방식의 근본적인 차이는 **코드의 모양이 데이터의 모양을 닮는다**는 점이다. 피자가 토핑을 포함하듯이, `pizza { ... }` 블록이 `topping(...)` 호출을 포함한다. HTML처럼 본래 트리 구조인 데이터에서는 이 장점이 가장 커진다. 체이닝은 평평하지만 블록은 중첩된다. 이 장에서는 후자를 "타입 안전 빌더"라고 부르며 주로 다룬다.

자바의 fluent 빌더와 코틀린 DSL 빌더의 근본적인 차이도 이 "중첩"에서 나온다. 자바 빌더는 `this`를 반환하는 세터를 이어 붙여 **평평한 체인**을 만든다. 계층이 깊어지면 계층마다 별도의 빌더 클래스와 `.and()`/`.endHead()` 같은 부모로 돌아가는 메서드가 필요하므로 코드가 금방 복잡해진다.

```java
// Java: 깊은 계층에서 fluent 빌더의 한계
new HtmlBuilder()
    .head()
        .title("동물원")
    .endHead()          // ← 수동으로 부모로 되돌아와야 함
    .body()
        .h1("환영")
    .endBody()
    .build();
```

코틀린 DSL에서는 `endHead()` 같은 되돌아가기 메서드가 필요 없다. 중괄호가 닫히면 곧바로 부모 스코프로 돌아가기 때문이다. **문법적인 블록 구조가 데이터의 트리 구조와 일대일로 대응**한다는 이 성질 덕분에, 코틀린 빌더는 자바의 fluent 빌더보다 한 단계 더 나은 표현력을 갖는다.

---

## 2. 수신 객체 지정 람다: DSL의 핵심

### 2.1 `T.() -> R`를 JVM 수준에서 다시 보기

DSL의 동작은 모두 하나의 타입에서 나온다. 바로 **수신 객체 지정 함수 타입**(function type with receiver), `T.() -> R`이다. 이 타입을 쓰는 관용구는 16장에서 설명했으므로, 여기서는 DSL을 만드는 데 필요한 **정확한 의미론**만 다시 정리한다.

일반 함수 타입과 수신 함수 타입을 나란히 비교해 보자.

```kotlin
val plain: (StringBuilder) -> Unit = { sb -> sb.append("x") }
val withRcv: StringBuilder.() -> Unit = { append("x") } // this가 StringBuilder

val sb = StringBuilder()
plain(sb)        // 명시적으로 sb를 인자로 넘김
sb.withRcv()     // sb가 '수신자'가 됨 (확장 함수처럼 호출)
withRcv(sb)      // ← 이것도 합법! 수신자를 첫 인자처럼 넘길 수 있다
```

이 코드에서 바로 알 수 있는 핵심 사실은 두 가지다.

**첫째, 수신 함수 타입은 첫 파라미터를 수신자로 바꾼 일반 함수 타입과 사실상 서로 변환된다.** 위에서 `withRcv(sb)`가 컴파일되는 것이 그 증거다. JVM 백엔드에서 `StringBuilder.() -> Unit`과 `(StringBuilder) -> Unit`은 **같은 런타임 표현**(`kotlin.Function1<StringBuilder, Unit>`)을 가진다. 둘의 차이는 **호출 문법과 본문 안에서 `this`가 무엇인가**라는, 컴파일 타임에만 존재하는 차이다.

**둘째, 수신 람다의 본문 안에서는 수신자 `T`의 멤버에 `this.` 접두어 없이 접근할 수 있다.** `{ append("x") }`에서 `append`는 `this.append`이고, `this`는 `StringBuilder`다. 이 "암시적 `this`"가 DSL의 본질이다. 블록 안에서 이름만 적으면 컴파일러는 그 이름을 현재 수신자의 멤버로 해석한다. 마치 그 블록이 해당 클래스의 메서드 본문인 것처럼 동작한다.

> [!note] 명세 기준
> 수신 람다 안에서 `this`는 **암시적 수신자(implicit receiver)** 다. 코틀린에서는 한 스코프에 암시적 수신자가 여러 개 겹칠 수 있다(중첩된 수신 람다, 확장 멤버 등). 이름을 해석할 때 컴파일러는 가장 안쪽 수신자부터 바깥쪽으로 차례로 찾는다. 이렇게 암시적 수신자가 여러 겹으로 쌓이는 구조가 5절에서 다룰 누수 문제의 원인이다.

### 2.2 build 함수의 표준 형태

이제 DSL 진입점의 표준 형태를 정리해 보자. 거의 모든 타입 안전 빌더는 다음 세 줄짜리 골격을 공유한다.

```kotlin
class HtmlTag {
    val children = mutableListOf<HtmlTag>()
    // ...
}

fun html(block: HtmlTag.() -> Unit): HtmlTag {
    val root = HtmlTag()   // 1. 수신자 객체 생성
    root.block()           // 2. 블록을 수신자 위에서 실행 (root가 this)
    return root            // 3. 완성된 객체 반환
}
```

이 패턴은 16장에서 본 `apply`와 **정확히 같은 골격**이다. 표준 라이브러리의 `apply`는 개념적으로 다음과 같이 정의되어 있다.

```kotlin
inline fun <T> T.apply(block: T.() -> Unit): T {
    block()   // this.block()
    return this
}
```

즉 `html` 함수는 "새 객체를 만들고 그 객체에 `apply`를 호출하는 것"과 같다. `fun html(block: HtmlTag.() -> Unit) = HtmlTag().apply(block)`처럼 한 줄로 쓸 수도 있다. **모든 타입 안전 빌더의 build 함수는 결국 `apply`를 특수화한 것이다.** 이 점을 이해하면 DSL이 훨씬 단순하게 보인다. 새 객체를 만들고, 그 객체 위에서 사용자 블록을 실행하고, 객체를 돌려준다. 그것이 전부다.

### 2.3 왜 확장이 아니라 파라미터인가

빌더 진입점을 `HtmlTag.() -> Unit`을 **파라미터**로 받는 최상위 함수로 만드는 것과 `apply`처럼 **확장 함수**로 만드는 것 사이에는 미묘한 설계 차이가 있다. DSL의 최상위 진입점(`html { }`)은 보통 최상위 함수여야 한다. 아직 수신자가 없기 때문이다.

반면 트리 안에서 자식 태그를 여는 함수(`head { }`, `body { }`)는 부모 태그의 **멤버 함수 또는 확장 함수**여야 한다. 그래야 부모 스코프 안에서만 보이기 때문이다. "어디에 정의하느냐가 어디서 보이느냐를 결정한다"는 이 원리가 4절에서 다룰 타입 안전성으로 바로 이어진다.

```text
html { }          ← 자유 함수 (전역에서 보임, DSL의 문)
  head { }        ← HTML의 멤버/확장 (html 블록 안에서만 보임)
    title { }     ← Head의 멤버/확장 (head 블록 안에서만 보임)
      +"제목"     ← Title의 unaryPlus (title 블록 안에서만 보임)
```

각 함수가 "누구의 멤버 또는 확장인가"에 따라 "어느 중첩 깊이에서만 쓸 수 있는가"가 정해진다. 이 대응 관계가 DSL 구조의 뼈대다.

---

## 3. 빌더 만들기: HTML DSL을 처음부터

지금까지의 이론을 바탕으로 작은 HTML 빌더를 처음부터 하나씩 만들어 보자. 목표는 다음 코드가 컴파일되고 실행되어 문자열을 출력하는 것이다.

```kotlin
val page = html {
    head {
        title { +"동물원" }
    }
    body {
        h1 { +"환영합니다" }
        p {
            +"오늘의 동물은 "
            b { +"수달" }
            +" 입니다."
        }
    }
}
println(page.render())
```

### 3.1 태그 계층

먼저 모든 태그가 공통으로 상속할 기반 클래스를 만든다. 태그는 이름과 자식 목록을 가지고, 자신을 문자열로 출력하는 기능을 제공한다.

```kotlin
abstract class Tag(private val name: String) {
    val children = mutableListOf<Tag>()

    // 자식 태그를 만들어 초기화하고 목록에 등록하는 공통 헬퍼
    protected fun <T : Tag> initTag(child: T, init: T.() -> Unit): T {
        child.init()          // 자식 위에서 사용자 블록 실행
        children += child     // 부모의 자식 목록에 등록
        return child
    }

    open fun render(indent: String = ""): String {
        val sb = StringBuilder()
        sb.append("$indent<$name>\n")
        for (c in children) sb.append(c.render("$indent  "))
        sb.append("$indent</$name>\n")
        return sb.toString()
    }
}
```

`initTag`가 이 빌더의 핵심이다. §2.2에서 본 build 골격, 즉 "자식을 만들고, 그 위에서 블록을 실행하고, 등록한 뒤 반환한다"는 흐름을 그대로 담았다. 모든 자식 태그 함수가 이 헬퍼를 호출한다.

이제 텍스트만 담는 특수 태그와 자식을 가질 수 있는 태그를 구분한다.

```kotlin
class TextElement(private val text: String) : Tag("#text") {
    override fun render(indent: String) = "$indent$text\n"
}
```

### 3.2 구조 태그와 자식 함수

각 구조 태그는 자신이 포함할 수 있는 자식 태그를 여는 **멤버 함수**를 제공한다. 여기서 "어떤 태그가 어떤 함수를 가지는가"가 곧 문서의 문법이 된다.

```kotlin
class HTML : Tag("html") {
    fun head(init: Head.() -> Unit) = initTag(Head(), init)
    fun body(init: Body.() -> Unit) = initTag(Body(), init)
}

class Head : Tag("head") {
    fun title(init: Title.() -> Unit) = initTag(Title(), init)
}

class Title : Tag("title")

class Body : Tag("body") {
    fun h1(init: H1.() -> Unit) = initTag(H1(), init)
    fun p(init: P.() -> Unit) = initTag(P(), init)
}

class H1 : Tag("h1")
class P : Tag("p") {
    fun b(init: B.() -> Unit) = initTag(B(), init)
}
class B : Tag("b")
```

`HTML`에는 `head`와 `body`만 있고 `title`은 없다. 그래서 `html { title { } }`은 **컴파일되지 않는다**. `title`은 `Head`의 멤버이고 `HTML`의 멤버가 아니기 때문이다. 이것이 4절에서 설명할 타입 안전성의 실체다. 각 태그 클래스의 멤버 목록이 곧 그 위치에서 문법적으로 허용되는 요소의 목록이다.

### 3.3 텍스트를 넣는 unaryPlus

`+"제목"`처럼 텍스트를 넣으려면, 텍스트를 담을 수 있는 태그에 `unaryPlus` 연산자를 정의해야 한다. 여러 태그가 함께 쓰므로 중간 클래스로 분리하는 것이 자연스럽지만, 간단하게 보여 주기 위해 여기서는 `Tag`에 확장 연산자로 직접 정의한다.

```kotlin
operator fun Tag.unaryPlus() {
    children += TextElement(this@unaryPlus.let { "" } + this.toStringSafe())
}
```

이 코드는 어색하다. 실제로는 태그 스코프 안에서 `String`에 대한 연산자를 호출하는 형태가 자연스럽다. 표준적인 방법은 텍스트 컨테이너 태그의 멤버로 `String.unaryPlus`를 두는 것이다.

```kotlin
abstract class TagWithText(name: String) : Tag(name) {
    operator fun String.unaryPlus() {
        children += TextElement(this)  // this는 String, children은 바깥 TagWithText
    }
}
```

여기서 주목할 것은 **이중 수신자**다. `String.unaryPlus`의 본문 안에서 `this`는 연산자 대상인 `String`이지만, `children`은 이 함수를 감싸는 `TagWithText`의 멤버다. 즉 이 확장 멤버 함수는 디스패치 수신자(`TagWithText`)와 확장 수신자(`String`)라는 두 개의 암시적 수신자를 동시에 사용한다. 멤버 확장의 이중 수신자는 35장에서 자세히 다루며, DSL에서는 이 기능을 매우 자주 사용한다.

텍스트를 담는 태그(`Title`, `H1`, `P`, `B`)가 `TagWithText`를 상속하도록 고치면, `+"..."`는 그 태그의 블록 안에서만 쓸 수 있다. `head { +"직접텍스트" }`는 컴파일되지 않는다. `Head`는 `TagWithText`가 아니므로 그 스코프에 `String.unaryPlus`가 없기 때문이다. 문법이 타입에 의해 자동으로 강제되는 것이다.

### 3.4 디슈가링: 한 줄이 어떻게 펼쳐지는가

이제 사용자가 작성한 DSL 코드 한 조각을 완전히 펼쳐 보자. 다음 코드는

```kotlin
val page = html {
    body {
        p { +"수달" }
    }
}
```

컴파일러 입장에서는 다음과 같이 보인다(개념적인 디슈가링).

```kotlin
val page = html(block = { /* this: HTML */
    this.body(init = { /* this: Body */
        this.p(init = { /* this: P (TagWithText) */
            this.unaryPlus("수달")   // String.unaryPlus, this는 P, 대상은 "수달"
        })
    })
})
```

실행 시점의 제어 흐름은 다음과 같다.

```text
html(block)
 └ root = HTML()
 └ root.block()                      // this=root(HTML)
     └ body(init)
         └ child = Body()
         └ child.init()              // this=child(Body)
             └ p(init)
                 └ leaf = P()
                 └ leaf.init()       // this=leaf(P)
                     └ "수달".unaryPlus()   // P.children += TextElement("수달")
                 └ Body.children += leaf
         └ HTML.children += child
 └ return root
```

트리가 **깊이 우선으로, 잎에서부터 위로** 조립된다는 것을 알 수 있다. 각 `init` 블록이 끝까지 실행되어 자식이 완성된 다음에야 부모의 `children`에 등록된다(`initTag`에서 `child.init()`이 `children += child`보다 먼저 실행된다). 이 순서 덕분에 트리는 항상 아래에서부터 완성된 상태로 쌓인다.

타입 안전 빌더의 구조는 이것이 전부다. 특별한 장치는 없다. **수신 람다 + 자식 팩토리 함수 + `apply`식 조립**을 재귀적으로 중첩했을 뿐이다. `render()`를 호출하면 완성된 트리를 순회해 문자열을 만든다. DSL은 **트리를 만드는 문법**이고, 그 트리로 무엇을 할지(렌더링, 검증, 실행)는 완전히 별개의 문제다.

> [!warning] 흔한 오해
> "DSL 블록은 선언형이라 순서가 없다"는 생각은 틀렸다. `html { body{}; head{} }`처럼 순서를 바꾸면 자식 목록의 순서가 바뀌고 렌더링 결과도 달라진다. DSL 블록은 **위에서 아래로 순서대로 실행되는 명령형 코드**다. 선언형처럼 *보일* 뿐이고, 각 줄은 부수 효과(자식 등록)를 일으키는 함수 호출이다.

---

## 4. 왜 "타입 안전"한가: 컴파일 타임 보장

### 4.1 스코프가 쓸 수 있는 어휘를 제한한다

"타입 안전 빌더"의 "타입 안전"은 홍보용 표현이 아니라 정확한 기술적 주장이다. 그 내용은 다음과 같다. **각 중첩 깊이에서 호출할 수 있는 함수는 그 위치의 수신자 타입이 제공하는 멤버(와 스코프 안의 확장)로 정확히 제한되고, 그 밖의 함수를 호출하면 컴파일 에러가 난다.**

3절에서 만든 계층으로 이를 확인해 보자.

```kotlin
html {
    head {
        title { +"ok" }   // Title은 TagWithText → unaryPlus 있음 → OK
        h1 { }            // 컴파일 에러: 'h1'은 Head의 멤버가 아님
    }
    p { }                 // 컴파일 에러: 'p'는 HTML의 멤버가 아님(Body의 멤버)
    +"직접"               // 컴파일 에러: HTML은 TagWithText가 아님
}
```

세 가지 오류가 모두 **컴파일 타임에** 발견된다. `h1`은 `Body`에만 있으므로 `Head` 블록(`this: Head`)에서 호출할 수 없다. `p`도 `Body`에만 있으므로 `HTML` 블록에서 호출할 수 없다. `+"..."`는 `TagWithText`에만 있으므로 텍스트를 담지 않는 `HTML`에서는 쓸 수 없다.

**검사 코드를 따로 작성하지 않았는데도** 컴파일러가 이 모든 규칙을 추가 비용 없이 강제한다. 규칙이 클래스 계층과 멤버 배치에 *구조적으로 인코딩*되어 있기 때문이다.

### 4.2 외부 DSL과 비교: 에러가 언제 드러나는가

이 장점이 얼마나 큰지는 문자열 기반의 외부 DSL과 비교하면 분명해진다. 같은 문서를 문자열 템플릿으로 만든다고 해 보자.

```kotlin
val html = """
    <html>
      <head>
        <h1>제목</h1>   <!-- 잘못된 중첩: h1은 head에 오면 안 됨 -->
      </head>
    </html>
""".trimIndent()
```

이 문자열은 **아무 문제 없이 컴파일된다.** 닫히지 않은 태그, 잘못된 중첩, `<haed>` 같은 오타는 이 문자열을 *실제로 파싱하거나 렌더링하는 런타임 시점*에야 드러난다. 끝까지 드러나지 않고 깨진 HTML이 출력될 수도 있다. 내부 DSL은 이런 오류를 **코드를 작성하는 시점**, 즉 컴파일 타임(그리고 IDE의 실시간 밑줄 표시)으로 앞당긴다.

```text
오류가 드러나는 시점의 사다리 (빠를수록 좋다)
─────────────────────────────────────────────
타이핑 중  ← 내부 DSL (IDE 실시간 진단)
컴파일     ← 내부 DSL (타입 검사)
─────────────────────────────────────────────
테스트     ← 외부 DSL (파서를 태우는 테스트가 있다면)
런타임     ← 외부 DSL (파싱/렌더 시점)
프로덕션   ← 외부 DSL (아무도 그 경로를 안 밟았다면)
```

### 4.3 자동완성과 리팩터링도 같은 정보에서 나온다

타입 안전성에는 강력한 부수 효과가 있다. `head { ` 안에서 IDE 자동완성을 호출하면, IDE는 "여기서 `this`는 `Head`"라는 것을 알고 있으므로 `Head`의 멤버(`title`)만 제안한다. 이는 별도의 IDE 플러그인이 하는 일이 아니라, **IDE가 같은 타입 정보를 재사용**하는 것이다. 마찬가지로 `Title` 클래스의 이름을 바꾸면 리팩터링이 DSL을 사용하는 곳까지 안전하게 반영된다.

외부 DSL의 문자열 안에서는 이 중 어느 것도 할 수 없다. **내부 DSL의 진짜 가치는 "이미 있는 컴파일러와 IDE 인프라를 추가 비용 없이 그대로 활용하는 것"** 이다. 파서를 작성하지 않아도 파서가 제공하는 모든 기능을 얻는다.

> [!caution] 성능 주의
> 타입 안전성은 컴파일 타임에만 존재하는 개념이므로 **런타임 비용이 0**이다. 타입 검사는 컴파일이 끝나면 사라진다. DSL의 런타임 비용은 객체 할당과 람다 호출에서만 생기며(8절), 타입 안전성 자체에는 비용이 없다. 이 점에서 "정적 타입 내부 DSL"은 동적 언어의 동적 DSL보다 낫다. 안전성을 얻으면서도 성능은 잃지 않는다.

---

## 5. `@DslMarker`: 암시적 수신자 누수 막기

### 5.1 드러나지 않는 버그 재현하기

지금까지의 설계에는 심각한 허점이 있다. §2.1에서 설명했듯이 코틀린은 **여러 겹의 암시적 수신자**를 허용한다. 중첩된 DSL 블록 안에서는 안쪽 수신자뿐 아니라 **바깥쪽 수신자도 여전히 스코프에 남아 있다.** 이 때문에 에러 없이 잘못 동작하는 문제가 생긴다.

`HTML`에도 `body`가 있지만, 그 안의 `Body`에서 또 `body`를 호출할 이유는 없다. 그러나 계층이 조금만 복잡해지면 이 문제가 실제로 나타난다. 좌표 그리드 DSL로 재현해 보자.

```kotlin
class Grid {
    val rows = mutableListOf<Row>()
    fun row(init: Row.() -> Unit) { rows += Row().apply(init) }
}
class Row {
    val cells = mutableListOf<String>()
    fun cell(text: String) { cells += text }
}

fun grid(init: Grid.() -> Unit) = Grid().apply(init)
```

이제 일부러 실수를 해 보자.

```kotlin
grid {              // this: Grid  (암시적 수신자 #1)
    row {           // this: Row   (암시적 수신자 #2)
        cell("A")   // Row.cell — OK
        row {       // ← 사고! Grid.row가 여기서도 보인다
            cell("B")
        }
    }
}
```

안쪽 `row { }` 블록 안에서 `this`는 `Row`이지만, 바깥의 `Grid`도 여전히 암시적 수신자로 스코프에 남아 있다. `Row`에는 `row` 메서드가 없으므로 컴파일러는 바깥 수신자인 `Grid`의 `row`를 찾아 **아무 경고 없이 연결한다.** 그 결과 컴파일은 성공하지만, 실행하면 `Grid` 안에 행이 하나 더 생기는 엉뚱한 트리가 만들어진다. 오류는 아니지만 의도와 다르게 동작하는, 가장 찾기 어려운 종류의 버그다.

```text
암시적 수신자 스택 (안쪽 row 블록 지점)
────────────────────────────────
[가장 안쪽]  this: Row    ← cell 여기
             this: Grid   ← row 가 '여기서' 잘못 해소됨 (누수!)
[가장 바깥]  (파일 최상위)
```

### 5.2 `@DslMarker`의 규칙

코틀린은 이 누수를 막으려고 `@DslMarker` 메타 애노테이션을 도입했다. 사용 방법은 두 단계다.

**1단계**: `@DslMarker`를 붙인 애노테이션을 직접 정의한다.

```kotlin
@DslMarker
annotation class GridDsl
```

**2단계**: DSL의 모든 수신자 클래스(또는 수신 함수 타입)에 그 애노테이션을 붙인다.

```kotlin
@GridDsl class Grid { /* ... */ }
@GridDsl class Row  { /* ... */ }
```

이렇게 하면 규칙이 바뀐다. 명세의 규칙은 다음과 같다. **같은 `@DslMarker` 애노테이션이 붙은 암시적 수신자가 스코프에 두 개 이상 있으면, 그중 가장 안쪽 수신자만 암시적으로 접근할 수 있고 나머지는 암시적 호출 대상에서 제외된다.** 따라서 앞의 잘못된 코드는 이제 컴파일 에러가 된다.

```kotlin
grid {
    row {
        row {   // 컴파일 에러:
                // 'fun row(...)' can't be called in this context
                // by implicit receiver. Use the explicit one if necessary.
            cell("B")
        }
    }
}
```

컴파일러는 `Grid`와 `Row`에 같은 마커(`@GridDsl`)가 붙어 있다는 것을 알기 때문에, 안쪽 `Row` 블록에서 바깥 `Grid`의 멤버를 **암시적으로** 호출하는 것을 금지한다. 이렇게 누수가 차단된다.

### 5.3 그래도 바깥 수신자를 호출하려면: 명시적 한정

`@DslMarker`가 막는 것은 **암시적** 접근뿐이다. 정말로 바깥 수신자를 호출하려는 의도라면 라벨로 명시하면 된다.

```kotlin
grid {                      // this@grid: Grid
    row {                   // this@row: Row
        this@grid.row {     // 명시적: "나는 정말로 Grid의 row를 원한다"
            cell("의도적")
        }
    }
}
```

`this@grid`는 16장과 [[14 - 클로저와 캡처]]에서 본 라벨 붙은 `this`다. `@DslMarker`의 원칙은 "위험한 것을 금지한다"가 아니라 **"암시적일 때만 위험하므로 명시하도록 강제한다"** 이다. 실수는 막고, 의도한 호출은 허용한다. 기본 동작은 안전하게 하고 예외는 명시적으로 쓰게 하는 것이 코틀린의 전형적인 설계 방식이다.

> [!note] 명세 기준
> `@DslMarker`의 제외 규칙은 **같은 마커 애노테이션을 공유하는** 수신자들 사이에서만 적용된다. 서로 다른 두 DSL을 중첩하면(예: `html { }` 안에서 다른 마커를 가진 `css { }`) 두 DSL의 수신자는 서로를 가리지 않는다.
>
> 또한 마커는 수신자의 **타입**에 붙는다. 마커는 상속을 통해 전파되므로, 공통 기반 클래스(예: `Tag`)에 한 번 붙이면 모든 하위 태그가 같은 마커를 물려받아 계층 전체가 한꺼번에 보호된다. 3절의 HTML 빌더라면 `abstract class Tag`에 `@HtmlDsl`을 한 번 붙이는 것으로 충분하다.

### 5.4 마커를 함수 타입에 붙이기

수신자 클래스를 직접 수정할 수 없을 때(라이브러리의 타입일 때)를 위해, 마커는 **수신 함수 타입 자체**에도 붙일 수 있다.

```kotlin
fun grid(init: @GridDsl Grid.() -> Unit) = Grid().apply(init)
```

`@GridDsl Grid.() -> Unit`처럼 함수 타입의 수신자 위치에 애노테이션을 붙이면, 클래스 선언을 바꾸지 못해도 그 진입점으로 들어온 수신자에 마커 효과가 적용된다. 실무에서는 일관성 때문에 클래스에 붙이는 방식을 더 많이 쓰지만, 이 방법을 알아 두면 다른 라이브러리의 타입으로 DSL을 만들 때 유용하다.

---

## 6. DSL 도구: infix, operator, 확장, 기본 인자

타입 안전 빌더의 기본 구조는 3~5절에서 완성되었다. 이제 DSL을 **읽기 좋게** 만드는 부가 도구들을 살펴본다. 이 도구들은 반드시 필요한 것은 아니고 표현력을 높이기 위한 선택 사항이다. 잘 쓰면 코드가 자연어에 가까워지지만, 잘못 쓰면 기호가 넘쳐 읽기 어려운 코드가 된다.

### 6.1 중위 함수 `infix`

**중위 표기(infix notation)** 는 `a.f(b)`를 `a f b`로 쓸 수 있게 해 준다. 점과 괄호가 사라지므로 이항 연산처럼 읽힌다. 표준 라이브러리의 `to`(`1 to "a"` → `Pair`)가 대표적인 예다.

```kotlin
infix fun Int.upTo(end: Int): List<Int> = (this..end).toList()

val r = 3 upTo 7   // => [3, 4, 5, 6, 7]
```

함수를 `infix`로 선언하려면 다음 세 조건을 모두 만족해야 한다.

| 조건 | 설명 |
|------|------|
| 멤버 또는 확장 함수 | 최상위 자유 함수는 infix 불가 |
| 파라미터 정확히 하나 | 0개·2개 이상 불가 |
| 그 파라미터가 vararg 아니고 기본값 없음 | `infix fun f(x: Int = 0)` 불가 |

중위 호출의 우선순위는 언어가 고정해 두었으며, 산술 연산자(곱셈·덧셈·범위)보다 낮고 비교·동등·`&&`·`||` 연산자보다 높다. 그래서 괄호가 필요한 경우가 생긴다.

```kotlin
// 1 to "a" 를 리스트에 넣기
val ok = listOf(1 to "a", 2 to "b")

// 산술과 섞이면 우선순위 주의
infix fun Int.plusInfix(o: Int) = this + o
val x = 1 plusInfix 2 * 3   // => 1 plusInfix (2*3) = 7, (*가 먼저)
```

DSL에서 infix는 상태 기계나 규칙을 정의할 때 잘 맞는다.

```kotlin
class Transition(val from: String, val on: String) { var to: String = "" }
infix fun String.on(event: String) = Transition(this, event)
infix fun Transition.goto(target: String) { this.to = target /* 등록... */ }

// "잠김" on "동전" goto "열림"   ← 자연어에 가까운 규칙
```

> [!warning] 흔한 오해
> `infix`가 새로운 연산자를 만든다고 오해하기 쉽지만 그렇지 않다. `infix`는 **일반 함수를 중위 표기로 호출할 수 있게** 할 뿐이고, 우선순위와 결합 방향은 언어가 정한 하나의 등급으로 고정되어 있다. 코틀린에서는 `<*>` 같은 임의의 기호 연산자를 정의할 수 없다. 실제 연산자 오버로딩은 정해진 관례 이름(`plus`, `times`…)으로만 할 수 있다(19장 참고).

### 6.2 연산자로 어휘 넓히기

DSL에서 특히 자주 쓰이는 연산자 관례가 몇 가지 있다.

**`unaryPlus`/`unaryMinus`**: 3절에서 `+"text"`로 이미 사용했다. 접두 `+`/`-`가 부수 효과가 있는 등록 동작으로 해석된다. 텍스트 삽입이나 요소 추가에 자주 쓴다.

```kotlin
class Menu {
    val items = mutableListOf<String>()
    operator fun String.unaryPlus() { items += this }
}
fun menu(init: Menu.() -> Unit) = Menu().apply(init)

menu {
    +"커피"
    +"홍차"
}   // items == ["커피", "홍차"]
```

**`invoke`**: 객체를 함수처럼 호출한다. 이름 없이 DSL 진입점을 여는 데 쓴다.

```kotlin
class Router {
    operator fun invoke(init: Router.() -> Unit) = apply(init)
    fun get(path: String) { /* ... */ }
}
val router = Router()
router {            // router.invoke { ... }
    get("/animals")
}
```

**`get`/`set`**: `config["timeout"] = 30` 같은 인덱스 접근으로 키-값 설정을 자연스럽게 표현한다.

```kotlin
class Config {
    private val map = mutableMapOf<String, Any?>()
    operator fun get(key: String) = map[key]
    operator fun set(key: String, value: Any?) { map[key] = value }
}
val c = Config()
c["port"] = 8080     // c.set("port", 8080)
c["host"] = "local"
```

이런 관례 이름의 전체 목록과 우선순위 규칙은 19장에서 다룬다. DSL은 그 목록에서 몇 가지를 골라 어휘를 만든다.

### 6.3 확장으로 DSL을 나중에 넓히기

내부 DSL의 큰 장점 중 하나는 **원본 코드를 수정하지 않고 어휘를 늘릴 수 있다**는 점이다. 라이브러리가 제공한 태그 타입에 확장 함수를 추가하면 새로운 자식 함수가 생긴다.

```kotlin
// 라이브러리가 Body를 줬고 우리는 못 고친다고 하자.
// 확장으로 새 자식 태그 함수를 추가:
fun Body.figure(init: Figure.() -> Unit) = initTagExt(Figure(), init)
```

물론 확장은 **정적 디스패치**되고 원본 클래스의 `private` 멤버에 접근할 수 없다(35장 참고). 그래서 `children` 같은 내부 상태가 `protected`나 `internal`로 열려 있어야 확장에서 자식을 등록할 수 있다. DSL 라이브러리가 확장을 통한 사용자 정의를 허용하려면 등록 지점을 적절한 가시성으로 공개해야 한다. 확장 가능성과 캡슐화 사이에서 어느 쪽을 택할지는 DSL을 설계할 때 실제로 내려야 하는 결정이다.

### 6.4 기본 인자와 이름 붙은 인자

DSL 함수의 속성을 다룰 때 **기본 인자**와 **이름 붙은 인자**를 쓰면 팩토리 함수가 지나치게 늘어나는 것을 막을 수 있다. 자바 빌더에서는 `.size(12).color("red")`처럼 세터를 이어 붙여야 하지만, 코틀린에서는 함수 호출 하나로 끝낼 수 있다.

```kotlin
fun circle(
    radius: Double,
    fill: String = "none",
    stroke: String = "black",
    strokeWidth: Double = 1.0,
) = Circle(radius, fill, stroke, strokeWidth)

// 필요한 것만, 이름으로:
val c = circle(radius = 5.0, fill = "red")
```

이 두 기능은 DSL에만 있는 것은 아니지만(12장에서 설명), DSL에서 세터 없이도 "속성 설정"을 선언적으로 표현할 수 있게 해 준다. 속성은 이름 붙은 인자로, 자식은 중첩 블록으로 표현하는 것이 잘 설계된 빌더의 일반적인 형태다.

### 6.5 도구 결합하기: 작은 그래프 DSL

여러 도구를 함께 쓰는 완성된 예제로 방향 그래프 DSL을 만들어 보자.

```kotlin
@DslMarker annotation class GraphDsl

@GraphDsl
class Graph {
    val edges = mutableListOf<Triple<String, String, Int>>()

    // 중위 + 연산자 결합: "a" - "b" weight 3
    inner class Edge(val from: String, val to: String)
    operator fun String.minus(other: String) = Edge(this, other)
    infix fun Edge.weight(w: Int) { edges += Triple(from, to, w) }
}

fun graph(init: Graph.() -> Unit) = Graph().apply(init)

val g = graph {
    "서울" - "부산" weight 400
    "서울" - "대전" weight 140
    "대전" - "부산" weight 260
}
// g.edges == [(서울,부산,400), (서울,대전,140), (대전,부산,260)]
```

`-`(minus 연산자)로 두 도시를 이어 `Edge`를 만들고, `weight`(infix)로 가중치를 붙여 등록한다. 연산자, 중위 함수, 수신 람다라는 세 도구가 함께 작동해서 `"서울" - "부산" weight 400`이 자연어처럼 읽힌다. 이 코드는 완전히 타입 안전하다. `weight`는 `Edge`에만 있고, `-`는 `String`에만(그것도 `Graph` 스코프 안에서만) 있으며, `@GraphDsl` 덕분에 중첩해도 누수가 생기지 않는다.

---

## 7. 실전 패턴: 설정 DSL, 타입 상태, 실제 사례

### 7.1 설정(config) DSL

실무에서 가장 흔한 용도는 **설정 객체 조립**이다. 옵션이 많고 계층 구조를 가질 때, DSL은 JSON이나 XML 설정 파일을 타입 안전하게 대체할 수 있다. 은행 계좌 설정을 예로 들어 보자.

```kotlin
@DslMarker annotation class AccountDsl

@AccountDsl
class AccountConfig {
    var owner: String = ""
    var currency: String = "KRW"
    private var _limit: Limit? = null

    fun limit(init: Limit.() -> Unit) { _limit = Limit().apply(init) }

    fun build() = Account(owner, currency, _limit)
}

@AccountDsl
class Limit {
    var daily: Long = Long.MAX_VALUE
    var perTransaction: Long = Long.MAX_VALUE
}

data class Account(val owner: String, val currency: String, val limit: Limit?)

fun account(init: AccountConfig.() -> Unit) = AccountConfig().apply(init).build()

// 사용:
val acc = account {
    owner = "홍길동"
    currency = "USD"
    limit {
        daily = 1_000_000
        perTransaction = 200_000
    }
}
```

핵심 관용구는 **`var` 프로퍼티 = 속성, 중첩 함수 = 하위 구성**이다. 속성은 `x = v`로 직접 대입하고(수신자가 `this`이므로 `this.x = v`와 같다), 하위 구성은 또 다른 수신 람다로 연다. 마지막에 `build()`를 호출해 가변 빌더를 불변 결과(`data class Account`)로 바꾸는 것도 흔한 패턴이다. 빌더는 가변이어야 쓰기 편하지만, 결과는 불변이어야 안전하기 때문이다.

> [!caution] 성능 주의
> 위처럼 `var`를 공개하는 빌더는 편리하지만, `build()`를 호출하기 전에 필수 필드(`owner`)가 설정되었는지 컴파일러가 강제하지 못한다. `owner`를 설정하지 않으면 빈 문자열인 채로 아무 경고 없이 통과한다. 필수 항목과 선택 항목을 컴파일 타임에 구분해 강제하려면 다음에 설명할 타입 상태 기법이 필요하다.

### 7.2 타입 상태로 필수 단계 강제하기 (팬텀 타입)

고급 DSL은 **타입 파라미터로 빌더의 "완성도"를 추적**해서, 필수 단계를 빠뜨리면 컴파일되지 않도록 만들 수 있다. 이를 팬텀 타입(phantom type) 또는 타입 상태(typestate) 패턴이라고 한다. HTTP 요청 빌더에 URL이 반드시 있어야 한다고 해 보자.

```kotlin
// 상태를 표현하는 빈 마커 타입 (인스턴스는 안 만듦)
sealed interface UrlState
class NoUrl : UrlState
class HasUrl : UrlState

class RequestBuilder<S : UrlState> {
    var url: String? = null
    var method: String = "GET"
}

// url 설정은 상태를 NoUrl -> HasUrl 로 '전이'시킨다
fun RequestBuilder<NoUrl>.url(u: String): RequestBuilder<HasUrl> {
    this.url = u
    @Suppress("UNCHECKED_CAST")
    return this as RequestBuilder<HasUrl>
}

// send는 오직 HasUrl 상태에서만 호출 가능
fun RequestBuilder<HasUrl>.send() { /* url!! 안전 */ }
```

이제 `url`을 호출하지 않고 `send`를 호출하면, 타입이 `RequestBuilder<NoUrl>`이므로 `send`(`HasUrl`만 받는 확장)가 **컴파일되지 않는다.** [[34 - 제네릭 2 - 타입 소거와 reified와 바운드]]에서 다룬 팬텀 타입 개념이 여기서는 DSL의 "빠진 필수 단계"를 컴파일 에러로 바꿔 준다. 이 기법은 강력하지만 복잡하므로, 필수 제약이 정말 중요할 때만 쓴다. 대부분의 DSL은 §7.1의 단순한 `var` 빌더로 충분하다.

> [!note] 명세 기준
> 위 예에서 `sealed interface UrlState`와 마커 클래스들의 인스턴스는 **런타임에 만들어지지 않는다**. 타입 파라미터 `S`는 34장에서 본 것처럼 JVM에서 소거되므로 런타임 표현에 흔적이 남지 않는다. `S`는 컴파일러가 상태를 추적하는 데만 쓰는 "유령" 타입이다. 그래서 `as RequestBuilder<HasUrl>` 캐스트는 런타임에 아무 일도 하지 않는, 비용이 없는 연산이다.

### 7.3 실제 사례: Gradle Kotlin DSL, 라우팅, 직렬화

내부 DSL은 코틀린 생태계 곳곳에서 실제로 쓰인다. 여기서는 널리 알려진 **오픈 소스 도구의 공개된 개념**만 소개한다.

**Gradle Kotlin DSL**: 빌드 스크립트 `build.gradle.kts`는 전체가 코틀린 DSL이다. `dependencies { implementation(...) }`에서 `dependencies`는 수신 람다를 받는 함수이고, `implementation`은 그 수신자의 함수다. `plugins { }`, `repositories { }`도 같은 구조다. Groovy DSL(`build.gradle`)은 동적이라 오타가 런타임에야 드러나지만, Kotlin DSL은 이 장에서 설명한 타입 안전성 덕분에 IDE 자동완성과 컴파일 검증을 받을 수 있다. §4에서 설명한 장점이 그대로 적용된다.

**HTML/마크업 빌더**: `kotlinx.html`은 3절에서 축소판으로 만든 빌더의 완성형이다. 실제 HTML의 모든 태그와 속성을 타입으로 인코딩해서, 잘못된 중첩이 컴파일 에러가 된다.

**라우팅·설정 DSL**: 여러 웹 프레임워크가 `routing { get("/x") { } }` 형태의 DSL로 경로를 선언한다. 이것도 수신 람다를 중첩한 구조다.

이 사례들의 공통 구조는 항상 같다. **수신 람다 진입점 + 중첩 자식 함수 + `@DslMarker` 보호**. 겉모습이 아무리 달라도 기본 구조는 3~5절에서 본 것과 같다. DSL을 읽을 때 이 구조를 염두에 두면, 처음 보는 DSL도 "`X { }`는 `X`를 수신자로 하는 람다이고, 그 안의 함수들은 `X`의 멤버구나" 하고 바로 이해할 수 있다.

### 7.4 DSL을 쓰지 말아야 할 때

DSL은 수단이지 목적이 아니다. 다음과 같은 경우에는 평범한 함수나 생성자가 낫다.

- **구조가 평평할 때**: 중첩이 없으면 트리 모양을 코드에 반영한다는 DSL의 장점이 사라진다. `data class`의 이름 붙은 인자 생성자로 충분하다.
- **일회성이거나 단순한 설정**: `Config(port = 8080, host = "x")` 한 줄이면 되는 것을 DSL로 감싸면 간접 계층만 늘어난다.
- **조합과 재사용이 중요할 때**: DSL 블록은 값이 아니라 부수 효과이므로, 일부를 변수에 담아 다시 조합하기 어렵다(8.4).
- **팀이 DSL에 익숙하지 않을 때**: DSL에는 학습 비용이 있다. 처음 보는 사람은 "이 중괄호가 뭐지?"라고 생각하게 된다.

**좋은 DSL의 기준**은 "이 API가 없었다면 사용자가 절차적으로 길게 작성했을 트리 형태의 구성을, 데이터 모양 그대로 선언할 수 있게 해 주는가"이다. 이 조건을 만족하지 않으면 DSL은 불필요한 기술 부채가 된다.

---

## 8. 비용과 함정: DSL은 공짜가 아니다

깔끔한 문법에는 대가가 따른다. 이 절에서는 DSL에 숨어 있는 런타임 비용과 설계상의 함정을 살펴본다.

### 8.1 어디에서 객체가 할당되는가

`html { body { p { +"x" } } }` 한 조각을 실행하면 힙에 무엇이 할당될까?

```text
할당 목록 (개념적, JVM 백엔드)
─────────────────────────────
HTML 객체            1개  ← html() 안에서 new
Body 객체            1개
P 객체               1개
TextElement 객체     1개
각 노드의 children ArrayList  4개
그리고 람다 객체들:
  { body{...} }      1개  ← HTML.() -> Unit
  { p{...} }         1개
  { +"x" }           1개
```

즉 태그 하나마다 노드 객체, 컬렉션, 그 태그를 여는 람다까지 여러 번의 할당이 쌓인다. 작은 문서라면 무시해도 되지만, 루프 안에서 수천 번 실행되는 DSL이나 성능에 민감한 경로에서는 이 할당이 GC 부담이 된다. **DSL의 선언형 겉모습 아래에서는 명령형 객체 생성이 빽빽하게 일어난다.**

### 8.2 `inline`으로 람다 할당 없애기

람다 객체 할당은 15장에서 다룬 `inline`으로 상당 부분 없앨 수 있다. build 함수와 자식 함수를 `inline`으로 선언하면 전달한 람다가 호출 지점에 펼쳐지므로 **`Function` 객체가 생성되지 않는다.**

```kotlin
inline fun html(block: HTML.() -> Unit): HTML = HTML().apply(block)
```

표준 라이브러리의 `apply`가 이미 `inline`인 것도 이 때문이다. 다만 다음 사항에 주의해야 한다.

- **노드 객체(HTML, Body…) 할당은 인라인해도 사라지지 않는다.** 인라인은 람다만 없앤다. 트리 노드 자체는 여전히 할당된다.
- **`inline` 남용은 코드 팽창**을 일으킨다. 자식 함수가 많고 모두 inline이면 호출 지점마다 본문이 복제된다. 그래서 DSL의 진입점 한두 개만 인라인하고 깊은 내부 함수는 그대로 두는 절충안을 많이 쓴다.
- **비지역 반환의 함정**: inline 람다 안의 `return`은 바깥 함수에서 빠져나간다(14장 클로저와 캡처). DSL 블록 안에서 `return`을 쓰면 의도하지 않게 감싸고 있는 함수에서 빠져나갈 수 있다. `return@html`처럼 라벨을 쓰거나, `crossinline`으로 비지역 반환을 금지한다.

```kotlin
fun makePage(): HTML {
    html {
        body {
            return HTML()   // ← inline이면 makePage에서 return! (의도?)
        }
    }
    error("unreachable if inline")
}
```

### 8.3 스마트 캐스트, null과의 상호작용

DSL 빌더의 `var` 프로퍼티는 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]에서 본 스마트 캐스트의 한계에 부딪힌다. 빌더 안에서 nullable 프로퍼티를 검사한 뒤 사용하더라도, 그 프로퍼티가 `var`이면 스마트 캐스트되지 않는다.

```kotlin
class Cfg { var name: String? = null }
fun cfg(init: Cfg.() -> Unit) = Cfg().apply(init)

cfg {
    if (name != null) {
        println(name.length)   // 컴파일 에러: name은 var 프로퍼티 → 스마트캐스트 불가
        println(name!!.length) // 이렇게 !! 필요
    }
}
```

DSL에서는 클로저를 통해 여러 곳에서 프로퍼티를 바꿀 수 있으므로, 컴파일러가 값이 바뀌지 않는다고 보장할 수 없기 때문이다. DSL 내부 로직에서 이 문제가 자주 생긴다면, 지역 `val`에 값을 복사해 두고 스마트 캐스트를 적용받는 관용구를 쓴다(`val n = name; if (n != null) n.length`).

### 8.4 조합의 어려움: 블록은 값이 아니다

DSL의 근본적인 한계는 **블록이 값이 아니라 부수 효과**라는 점이다. 함수형 파이프라인([[39 - 컬렉션 연산과 함수형 파이프라인]])은 값을 반환하므로 조합하고 재사용할 수 있다. 반면 DSL 블록은 수신자에 등록하는 부수 효과만 일으키므로, 블록 자체를 변수에 담아 다시 조합하기 어렵다.

```kotlin
// 이렇게 '부분 DSL'을 담아 재사용하려면?
val commonHead: Head.() -> Unit = {
    title { +"공통 제목" }
}

html {
    head(commonHead)   // 수신 람다를 그대로 넘기면 재사용 가능
    body { /* ... */ }
}
```

수신 함수 타입을 값으로 분리해 재사용하는 이 방법이 유일한 해결책이다. `head { }` 자리에 `head(commonHead)`처럼 미리 만든 `Head.() -> Unit`을 넘긴다(§2.1에서 본 "수신자를 인자처럼 넘기는" 원리). 다만 이렇게 하면 DSL 특유의 읽기 좋은 모양이 사라지고 일반 함수 조합에 가까워진다. **DSL은 "만들기"에 강하고 "조합하기"에 약하다.** 이 트레이드오프를 이해하고 도구를 선택해야 한다.

### 8.5 디버깅과 스택 트레이스

DSL이 인라인되고 람다가 여러 겹으로 쌓이면 스택 트레이스를 읽기 어려워진다. 깊이 중첩된 블록 안에서 예외가 발생하면 스택에 `invoke`와 인라인 프레임이 쌓여, 실제로 잘못된 DSL 줄을 찾기 힘들다. 특히 `inline`은 프레임을 없애 버리므로 디버거의 스텝 실행이 예상과 다르게 건너뛴다. 복잡한 DSL을 디버깅할 때는 문제가 되는 구간만 임시로 non-inline 함수로 분리해 프레임이 보이게 하는 것이 실무에서 쓸 만한 방법이다.

### 8.6 실행 순서의 함정: 속성과 자식이 섞일 때

DSL 블록이 명령형이라는 사실(§3.4) 때문에 미묘한 순서 버그가 생긴다. 한 블록 안에 속성 설정과 자식 등록이 섞여 있으면, 자식이 등록되는 시점에 부모의 속성이 아직 설정되지 않았을 수 있다.

```kotlin
@DslMarker annotation class ReportDsl

@ReportDsl
class Report {
    var title: String = "무제"
    val lines = mutableListOf<String>()
    // 자식이 부모의 현재 title을 '즉시' 읽는다면?
    fun line(text: String) { lines += "[$title] $text" }
}

fun report(init: Report.() -> Unit) = Report().apply(init)

val r = report {
    line("첫 줄")        // 이 시점 title은 아직 "무제"
    title = "월간 보고"   // 나중에 바뀌어도
    line("둘째 줄")      // 이건 "월간 보고"로 찍힘
}
// r.lines == ["[무제] 첫 줄", "[월간 보고] 둘째 줄"]
```

`line`은 호출되는 **그 순간의** `title`을 읽는다. 따라서 속성을 자식보다 나중에 설정하면 먼저 등록된 자식은 이전 값을 보게 된다. 선언형처럼 *보이는* DSL이라서 "어차피 모두 설정되면 반영되겠지"라고 착각하기 쉽지만, 각 줄은 그 순간 실행되는 명령이다.

**DSL을 설계할 때는 속성을 먼저, 자식을 나중에 쓰도록 유도하거나(문서화·컨벤션), 자식이 부모 속성을 등록 시점이 아니라 `build()`/`render()` 시점에 지연 조회하도록** 만드는 것이 안전하다. 이 함정은 [[24 - 상속과 오버라이딩과 초기화 순서]]에서 본 "초기화 순서" 문제가 DSL에서 나타난 형태라고 할 수 있다.

> [!warning] 흔한 오해
> "DSL은 코드가 짧으니 빠르다"는 생각은 착각이다. 코드의 길이와 런타임 비용은 관계가 없다. `html { }` 세 줄이 내부적으로 수십 개의 객체를 할당할 수도 있다(§8.1). DSL을 선택하는 기준은 **성능이 아니라 표현력과 안전성**이다. 성능이 결정적인 경로라면 DSL의 편리함을 포기하고 직접 최적화한 절차적 코드를 쓰는 것이 맞다. 다행히 DSL의 타입 안전성 자체에는 비용이 없으므로(§4.3), 비용은 할당에서만 생긴다.
