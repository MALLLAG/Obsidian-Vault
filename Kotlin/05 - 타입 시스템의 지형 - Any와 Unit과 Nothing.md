---
title: 타입 시스템의 지형 - Any와 Unit과 Nothing
date: 2026-07-13
tags: [kotlin, type-system, any, unit, nothing, 학습노트]
---

**이 장이 답하는 질문:**

- Kotlin의 타입들은 하늘에 흩뿌려진 별이 아니라 하나의 지형을 이룬다. 그 지형의 정상과 밑바닥에는 무엇이 있는가?
- `Any`는 Java의 `Object`와 같은가? 그렇다면 왜 `Any`에는 `wait()`나 `getClass()`가 없는가?
- 진짜 최상위 타입은 `Any`인가 `Any?`인가? `null`은 대체 어느 타입의 값인가?
- `Unit`은 "값이 없음"을 뜻하는 `void`인가, 아니면 무언가 실체가 있는가? (M10)
- `Nothing`은 `null`의 사촌 같은 "빈 값"인가, 아니면 정반대의 무언가인가? (M11)
- 값을 절대 만들 수 없는 타입이 왜 언어에 필요한가? `throw`와 `return`에 타입을 붙여서 무슨 이득을 보는가?
- `if (조건) 1 else throw 예외`의 타입은 왜 정확히 `Int`인가? 컴파일러는 두 갈래의 타입을 어떻게 합치는가?
- Kotlin에는 왜 TypeScript 같은 합집합 타입(`String | Int`)이 없는데, `String?`은 있는가? 스마트 캐스트가 만드는 교집합 타입은 소스에 적을 수 있는가?

---

[[04 - 파일과 패키지와 선언 - 프로그램의 골격]]에서 우리는 프로그램의 골격 — 파일, 패키지, 최상위 선언, `main` — 을 세웠다. 골격은 이름들이 사는 공간을 정의했지만, 그 이름들이 *어떤 값을 가리킬 수 있는가*는 아직 열려 있었다. 이 장은 그 질문에 답하는 지도 한 장이다. Kotlin의 모든 타입은 서로 무관한 섬이 아니라, 하나의 부분타입(subtype) 관계로 엮인 격자(lattice) — 정상이 하나, 밑바닥이 하나인 지형 — 를 이룬다. 이 지형의 좌표계를 손에 넣으면 나머지 45개 장에서 마주칠 모든 타입 규칙이 "이 지형의 어디쯤인가"라는 한 가지 질문으로 환원된다.

이 장은 지형의 극점 세 개를 정밀 측량한다. 정상에 있는 `Any`(와 그보다 한 뼘 위의 `Any?`), 아무 값도 반환하지 않는 함수가 실은 반환하는 값 `Unit`, 그리고 값이 단 하나도 살지 않는 밑바닥 `Nothing`. 이 셋을 관통하는 주제는 "타입은 값의 집합이다"라는 관점이다. `Any`는 (거의) 모든 값을 담은 큰 집합, `Unit`은 원소가 딱 하나인 집합, `Nothing`은 공집합이다. 이 집합론적 시선이 왜 `Nothing`이 모든 타입의 하위타입인지, 왜 `Unit`이 `void`와 다른지를 단번에 설명한다.

경계도 못박아 두자. 이 장은 지형의 *극점과 골격*을 다룬다. `null` 가능성이 만드는 산맥의 세부 지질 — 안전 호출 `?.`, 엘비스 `?:`, 스마트 캐스트의 정확한 발동 조건 — 은 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]의 소유다. 원시 타입이 없다는 사실("모든 것은 객체")의 기계 수준 진상은 [[07 - 수 1 - Int와 Long과 부동소수점]]과 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]가, 예외가 왜 `Nothing`을 낳는지의 예외 계층은 [[41 - 예외와 Nothing]]이 이어받는다. 여기서는 그 세부로 내려가기 전에, 지도의 등고선을 먼저 그린다.

논증의 궤적은 이렇다. 먼저 "타입 = 값의 집합"과 부분타입 = 부분집합이라는 관점을 세우고(1절), 그 격자의 정상 `Any`/`Any?`를 측량하고(2절), 원소가 하나뿐인 특이점 `Unit`을 해부해 `void`라는 오해를 정면으로 부순다(3~4절). 그 다음 격자의 밑바닥 `Nothing`을 파고들어(5절) 그것이 "빈 값"이 아니라 "값이 없는 타입"임을 못박고(6절), `Nothing`이 제어 흐름과 타입 추론에서 하는 실제 노동을 본다(7절). 마지막으로 이 지형에 합집합/교집합이 어떻게 들어오는지 — nullable, 스마트 캐스트, 정의상 비-널 타입 — 를 살펴(8~9절), 정적·강타입이라는 규율로 갈무리한다.

---

## 1. 타입은 값의 집합이다 — 지형이라는 은유

### 1.1 왜 "지형"인가

프로그래밍 언어의 타입을 처음 배울 때 우리는 대개 타입을 "값에 붙는 라벨"로 여긴다. `42`에는 `Int` 라벨이, `"cat"`에는 `String` 라벨이 붙는다는 식이다. 이 라벨 관점은 틀리지 않지만 얕다. 더 깊고 생산적인 관점은 집합론적이다: **타입은 그 타입에 속하는 값들의 집합**이다. `Boolean`은 `{true, false}`라는 원소 두 개짜리 집합이고, `Int`는 $-2^{31}$부터 $2^{31}-1$까지의 정수 약 43억 개의 집합이며, `String`은 (메모리가 허락하는 한) 무한에 가까운 문자열들의 집합이다.

이 관점을 택하는 순간, 타입들 사이의 관계가 집합들 사이의 관계로 번역된다. 그 중 가장 중요한 것이 **부분타입 관계(subtyping)** 다. 타입 $S$가 타입 $T$의 부분타입이라는 것($S <: T$)은, 집합의 언어로는 $S$의 값 전부가 $T$의 값이기도 하다는 뜻 — 즉 $S \subseteq T$다. 개는 동물이므로 `Dog`의 인스턴스 전부가 `Animal`의 인스턴스이기도 하다: `Dog <: Animal`. 부분타입 관계가 곧 부분집합 관계라는 이 대응이, 이 장 전체의 뼈대다.

```kotlin
open class Animal(val name: String)
class Dog(name: String) : Animal(name)

val a: Animal = Dog("Rex")   // => OK. Dog의 값은 Animal의 값이기도 하다 (Dog <: Animal)
// val d: Dog = Animal("?")  // 컴파일 에러: Animal은 Dog가 아니다. 부분집합 방향이 반대
```

집합 관점이 강력한 이유는, 극단적인 집합 — 전체집합과 공집합 — 에 대응하는 타입이 무엇일지를 곧바로 묻게 만들기 때문이다. "모든 값을 담은 집합"에 해당하는 타입이 있는가? 있다 — 그게 이 장의 `Any`(정확히는 `Any?`)다. "원소가 하나도 없는 공집합"에 해당하는 타입이 있는가? 있다 — 그게 `Nothing`이다. 이 두 극점 사이에 나머지 모든 타입이 층층이 쌓여 지형을 이룬다.

### 1.2 부분타입 격자의 정상과 밑바닥

부분타입 관계는 단순한 순서가 아니라 격자(lattice)에 가까운 구조를 이룬다. 격자의 핵심 성질은 두 가지다. 임의의 두 타입에는 그 둘을 모두 포함하는 가장 작은 공통 상위타입(least upper bound, LUB — 합집합에 가까움)이 있고, 그 둘에 모두 포함되는 가장 큰 공통 하위타입(greatest lower bound, GLB — 교집합에 가까움)이 있다는 것이다. Kotlin 컴파일러가 `if/else`나 `when`의 여러 갈래를 하나의 타입으로 묶을 때, 그리고 스마트 캐스트로 타입을 좁힐 때 하는 일이 바로 이 LUB와 GLB 계산이다.

이 격자가 격자로서 "완성"되려면 정상과 밑바닥이 반드시 필요하다. 정상(top)은 모든 타입의 상위타입 — 전체집합 — 이고, 밑바닥(bottom)은 모든 타입의 하위타입 — 공집합 — 이다. 이것이 없으면 "공통 상위타입이 아예 없는 두 타입" 같은 구멍이 생겨 타입 추론이 막힌다. Kotlin은 이 두 극점을 명시적으로 언어에 박아 넣었다: 정상은 `Any?`, 밑바닥은 `Nothing`이다.

```text
                        Any?          ← 정상(top): 모든 타입의 상위타입 = 전체집합
                       /    \
                     Any    (모든 T?: String?, Int?, Animal? ...)
                    / | \        │
              String Int Animal  │  Nothing?  ← 값은 null 하나뿐
                    \ | /        │   │
                     (모든 비-널 타입 T)
                       \        /
                        Nothing            ← 밑바닥(bottom): 모든 타입의 하위타입 = 공집합
```

이 그림에서 당장 눈에 띄는 비대칭이 하나 있다. 밑바닥은 `Nothing` 하나로 깔끔한데, 정상은 `Any`가 아니라 `Any?`다. `Any`는 정상이 아니다. 왜냐하면 `null`이라는 값이 `Any`에는 속하지 않기 때문이다 — `Any`는 "널이 아닌 모든 값"의 집합이고, `null`까지 포함한 진짜 전체집합은 `Any?`다. 이 한 뼘의 차이가 Kotlin 타입 시스템의 심장인 널 안전성의 기하학적 표현이다. 2절과 8절에서 이 비대칭을 정밀하게 파낸다.

> **명세 정밀**: "격자"라는 말은 편의상의 은유다. 널 가능성과 플랫폼 타입(유연한 타입, flexible type)까지 포함하면 Kotlin의 부분타입 관계는 순수한 격자보다 복잡한 반순서(preorder)에 가깝다. 하지만 실무적으로 중요한 건 "정상 `Any?`, 밑바닥 `Nothing`, 그리고 둘을 잇는 부분타입 사슬"이라는 골격이며, 이 골격은 정확하다.

### 1.3 정적 타입과 강타입 — 지형은 컴파일 타임의 것

이 지형이 언제 쓰이는가를 못박아 두자. Kotlin은 **정적 타입(statically typed)** 언어다. 모든 식(expression)은 컴파일 시점에 하나의 정적 타입을 가지며, 부분타입 관계의 검사는 대부분 컴파일 타임에 끝난다. `val a: Animal = Dog(...)`가 허용되는지는 프로그램을 실행하기 전에 결정된다. 이것이 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]에서 다룰 널 안전성이 "런타임 검사가 아니라 컴파일 타임 검사"인 이유의 뿌리이기도 하다(M7 참조).

동시에 Kotlin은 **강타입(strongly typed)** 이다. 타입 사이의 암시적이고 무언(無言)의 변환이 거의 없다. Java·C에서 당연하던 `int`→`long` 자동 확대조차 Kotlin에는 없어서 `toLong()`을 명시해야 한다([[07 - 수 1 - Int와 Long과 부동소수점]]). 이 강타입 규율 때문에, 지형에서 위로 올라가는(상위타입으로 보는) 것은 자동이지만 — 개를 동물로 보는 것은 공짜다 — 아래로 내려가는(하위타입으로 좁히는) 것은 반드시 명시적 캐스트(`as`, `is`, 스마트 캐스트)를 요구한다. 지형의 등고선을 오르내리는 규칙이 곧 타입 검사의 규칙이다.

---

## 2. `Any` — 정상 바로 아래, 널이 아닌 모든 것의 뿌리

### 2.1 모든 클래스의 암시적 슈퍼클래스

명시적 슈퍼타입을 적지 않은 모든 Kotlin 클래스는 `Any`를 상속한다. `class Coordinate(val x: Int, val y: Int)`라고만 써도, 이 클래스는 암묵적으로 `Any`의 하위타입이다. 명시적으로 `: Any()`라고 적을 수도 있지만 아무 의미 없는 잉여다.

```kotlin
class Coordinate(val x: Int, val y: Int)          // 암묵적으로 : Any()
class Circle(val r: Double) : Any()               // 명시했지만 잉여 — 위와 동일

fun describe(value: Any) {                          // Any 파라미터: 널 아닌 무엇이든 받는다
    println(value.toString())
}

describe(Coordinate(1, 2))  // => OK
describe(42)                // => OK — Int도 Any의 하위타입
describe("cat")             // => OK
describe(true)              // => OK
// describe(null)           // 컴파일 에러: Null can not be a value of a non-null type Any
```

마지막 줄이 핵심이다. `Any`는 "널이 아닌 모든 값"의 집합이지 "모든 값"의 집합이 아니다. `null`은 `Any`의 값이 아니다. 이것이 2.4절과 8절에서 `Any`와 `Any?`를 가르는 결정적 사실이다.

`Any`가 모든 타입의 (널 아닌 쪽) 뿌리라는 것은 실용적으로 두 가지를 뜻한다. 첫째, 어떤 값이든 담을 수 있는 이질적 컨테이너의 원소 타입으로 `Any`를 쓸 수 있다(단 타입 안전성은 포기). 둘째, 모든 값에서 `Any`가 선언한 세 멤버 — `equals`, `hashCode`, `toString` — 를 호출할 수 있다. 정수든, 람다든, 좌표든 `.toString()`을 부를 수 있는 것은 이들이 모두 `Any`의 하위타입이기 때문이다.

### 2.2 `Any`가 선언하는 것은 정확히 세 개뿐

`Any`의 정의는 놀랄 만큼 작다. 표준 라이브러리에서 `Any`는 대략 이렇게 선언되어 있다.

```kotlin
public open class Any {
    public open operator fun equals(other: Any?): Boolean
    public open fun hashCode(): Int
    public open fun toString(): String
}
```

멤버가 셋뿐이다. `equals`, `hashCode`, `toString`. 이 세 계약(contract)의 정확한 의미 — `==`가 왜 `equals`로 번역되는지, `hashCode`와 `equals`의 일관성 계약, 참조 동일성 `===`와의 차이 — 는 [[10 - 불리언과 동등성과 동일성]]의 소유다(M6). 여기서 중요한 건 그 셋이 *모든 값에 보편적으로 존재하는 능력*이라는 사실이다. 어떤 값이든 문자열로 만들 수 있고, 다른 값과 같은지 물을 수 있고, 해시 코드를 뽑을 수 있다.

`equals`의 파라미터 타입이 `Any?`(널 허용)라는 점을 눈여겨보라. `x == null` 같은 비교가 타입 오류 없이 성립해야 하므로, `equals`의 상대는 널일 수 있어야 한다. 반면 `Any` 자체는 `null`을 값으로 갖지 못한다 — 타입으로서의 `Any`와 파라미터 타입 `Any?`의 이 차이가 정확히 널 가능성의 문법이다.

### 2.3 `Any`는 `java.lang.Object`가 아니다 — JVM 매핑의 진실

가장 흔한 오해 중 하나는 "Kotlin의 `Any`는 Java의 `Object`에 대한 새 이름일 뿐"이라는 것이다. JVM 백엔드에서 `Any`가 `java.lang.Object`로 매핑되는 것은 맞다 — 그래서 Kotlin과 Java가 상호운용될 때 `Any`와 `Object`가 서로 통한다. 하지만 *언어 수준에서* `Any`는 `Object`와 같지 않다. 두 가지 이유가 있다.

첫째, `Object`에는 `wait()`, `notify()`, `notifyAll()`, `getClass()` 같은 메서드가 있지만 **`Any`에는 없다**. Kotlin은 이 메서드들을 `Any`의 멤버에서 의도적으로 제거했다. `wait/notify`는 저수준 스레드 동기화의 낡고 오류를 부르는 API라 코루틴·고수준 동시성([[43 - 동시성과 메모리 모델]])을 지향하는 Kotlin이 기본 노출을 원치 않았고, `getClass()`는 Kotlin의 리플렉션 진입점 `::class`([[44 - 애노테이션과 리플렉션]])로 대체되었다.

```kotlin
val c = Coordinate(1, 2)
// c.wait()         // 컴파일 에러: unresolved reference — Any에는 wait가 없다
// c.getClass()     // 컴파일 에러: unresolved reference — 대신 c::class 를 쓴다

// JVM 백엔드에서 정말 Object의 메서드가 필요하면 명시적으로 Object로 캐스팅
(c as java.lang.Object).wait()   // JVM 한정. 이제 wait 가시
```

둘째, 방향은 반대지만 `Any`가 `Object`보다 *더* 넓다. Kotlin에서는 `Int`, `Boolean` 같은 값도 `Any`의 하위타입이다. Java에서 `int`는 `Object`의 하위타입이 아니지만(원시 타입은 클래스 계층 밖), Kotlin에서 `42`는 `Any`로 취급될 수 있다. 다만 그렇게 하려면 JVM 백엔드에서는 박싱이 일어난다 — 이 대목이 M4("원시 타입이 없다")·M5("Int?는 항상 박싱")와 이어지며, 정확한 기계 수준 진상은 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]가 소유한다. 여기서는 "언어 지형에서는 모든 값이 `Any` 아래에 있지만, 그것이 곧 원시 타입이 사라졌다는 뜻은 아니다"만 기억하면 된다.

> **흔한 오해**: "`Any`는 `Object`의 alias다." — JVM 백엔드의 *런타임 매핑*은 그렇지만, *컴파일 타임 타입*으로서 `Any`는 `wait/notify/getClass`가 없고 원시 값도 하위에 두는 별개의 타입이다. 또한 Native/JS/Wasm 백엔드에는 `java.lang.Object` 자체가 존재하지 않는다. `Any`는 언어의 개념이고 `Object`는 JVM 플랫폼의 구현이다.

### 2.4 `Any`와 `Any?` — 한 뼘의 차이가 전부다

`Any`는 지형의 정상이 아니다. `Any` 위에 `Any?`가 있다. `Any?`는 `Any`의 모든 값에 `null` 하나를 더한 집합이다. 집합으로 쓰면:

$$\text{Any?} = \text{Any} \cup \{\texttt{null}\}$$

`Any?`가 진짜 정상(모든 타입의 상위타입)인 이유는, 어떤 타입 `T`든 그 값은 `Any`에 속하거나 `null`이거나 둘 중 하나이기 때문이다 — 두 경우 다 `Any?`에 속한다. 심지어 `String?` 같은 널 허용 타입의 값(`null` 포함)도 `Any?`에 속한다.

```kotlin
val anything: Any? = null          // => OK — Any?는 null을 담는다
val everything: Any? = "cat"       // => OK
val stillOk: Any? = Coordinate(1,2)// => OK

// val notNull: Any = null         // 컴파일 에러: Any는 null을 담지 못한다

fun <T> asAnything(value: T): Any? = value   // 어떤 T든 Any?로 승격 가능 (T <: Any?)
```

마지막 함수가 시사적이다. 임의의 타입 파라미터 `T`의 값을 아무 제약 없이 `Any?`로 받을 수 있다. 이는 **타입 파라미터의 기본 상한(upper bound)이 `Any?`** 라는 사실의 표현이다(M45). `fun <T>`라고만 쓰면 `T`는 널 허용까지 포함하는 `T : Any?`를 뜻한다. 널을 배제하고 싶으면 `fun <T : Any>`로 상한을 낮춰야 한다. 이 규칙의 자세한 결과 — 제네릭에서 `null`이 불쑥 들어오는 함정 — 는 [[33 - 제네릭 1 - 타입 파라미터와 변성]]이 소유한다.

정리하면 지형의 정상 부근은 이렇게 생겼다.

```text
   Any?   ← 진짜 정상. 모든 값 + null. 타입 파라미터의 기본 상한.
    │
    ├── Any           ← 널 아닌 모든 값. Object로 매핑되나 wait/getClass 없음.
    │
    └── Nothing?      ← null 하나만. (5절에서 자세히)
```

`Any?`와 `Any` 사이의 이 한 칸이 Kotlin 널 안전성의 기하학적 원점이다. "`?`를 붙인다"는 것은 지형에서 정확히 한 층 위로 — `null`을 원소로 추가한 더 큰 집합으로 — 올라가는 연산이다.

---

## 3. `Unit` — 원소가 하나뿐인 타입

### 3.1 "아무것도 반환하지 않는" 함수가 반환하는 것

의미 있는 값을 돌려주지 않는 함수를 생각해 보자. 좌표를 콘솔에 찍기만 하는 함수 말이다.

```kotlin
fun printCoordinate(c: Coordinate) {
    println("(${c.x}, ${c.y})")
}
```

반환 타입을 적지 않았다. 많은 언어라면 여기서 "이 함수는 값을 반환하지 않는다(void)"고 말할 것이다. 그러나 Kotlin에서 이 함수의 반환 타입은 `void`가 아니라 **`Unit`** 이다. 위 선언은 다음과 완전히 동일하다.

```kotlin
fun printCoordinate(c: Coordinate): Unit {   // : Unit 은 생략된 것뿐
    println("(${c.x}, ${c.y})")
    return Unit                               // 이 return도 암묵적으로 존재
}
```

`Unit`은 이름이며 동시에 값이다. 표준 라이브러리에서 `Unit`은 이렇게 선언되어 있다.

```kotlin
public object Unit {
    override fun toString(): String = "kotlin.Unit"
}
```

즉 `Unit`은 `object` 선언 — 싱글턴([[28 - object와 companion - 싱글턴과 동반 객체]]) — 이다. 프로그램 전체에서 `Unit` 타입의 값은 정확히 하나, 그 싱글턴 인스턴스뿐이다. 그래서 `Unit`은 **원소가 딱 하나인 집합**에 대응하는 타입이다. 집합으로 쓰면 $\text{Unit} = \{\,\texttt{Unit}\,\}$ — 크기 1의 집합. 타입 이론에서는 이런 타입을 단위 타입(unit type)이라 부르며, 그 자매 서술은 [[10 - 곱·합·단위·공집합]]에 있다.

값을 반환하지 않는 함수가 사실은 "정보가 0비트인 값" 하나를 반환한다는 이 발상은, 처음엔 궤변처럼 들린다. 하지만 이것이 `void`보다 훨씬 규칙적이고 유용한 설계임을 다음 절들에서 보게 된다.

### 3.2 원소가 하나면 정보량이 0이다

`Unit` 값이 담는 정보량을 정보 이론으로 재보자. 가능한 값이 $n$개인 타입이 담을 수 있는 정보량은 $\log_2 n$ 비트다. `Boolean`은 값이 2개($\log_2 2 = 1$비트), `Int`는 약 43억 개($\log_2 2^{32} = 32$비트). 그렇다면 `Unit`은?

$$\log_2 |\text{Unit}| = \log_2 1 = 0 \text{ 비트}$$

`Unit` 값은 정보를 0비트 담는다. 이것이 "값을 반환하지 않는다"라는 직관과 "값을 반환하되 그 값은 정보가 없다"라는 Kotlin의 형식화를 잇는 다리다. 실용적으로는 같은 이야기다 — 어차피 볼 게 없는 값이다. 하지만 형식적으로 "값이 있다"고 못박아 두면, 그 값을 다른 값들과 똑같이 취급할 수 있다: 변수에 담고, 제네릭 타입 인자로 넘기고, 함수 타입의 결과로 삼을 수 있다. `void`는 이런 취급이 불가능하다. 이 차이가 4절의 핵심이다.

```kotlin
val result: Unit = printCoordinate(Coordinate(1, 2))   // => Unit 값을 변수에 담을 수 있다
println(result)                                         // => kotlin.Unit

val u1 = Unit
val u2 = printCoordinate(Coordinate(3, 4))
println(u1 === u2)   // => true — Unit 값은 유일한 싱글턴이므로 항상 동일
```

### 3.3 람다와 `Unit` 강제 변환

`Unit`이 값이라는 사실은 람다에서 미묘하고 편리한 규칙을 낳는다. 함수 타입 `() -> Unit`을 기대하는 자리에 람다를 넘길 때, 그 람다의 마지막 식이 `Unit`이 아니어도 컴파일러가 자동으로 `Unit`으로 맞춰 준다. 이를 **Unit 강제 변환(Unit coercion)** 이라 한다.

```kotlin
fun runTwice(action: () -> Unit) {
    action(); action()
}

val log = mutableListOf<String>()
runTwice {
    log.add("tick")   // add는 Boolean을 반환하지만...
}                     // ...() -> Unit 문맥이라 Boolean 결과는 Unit으로 강제 변환된다
// 컴파일 OK. 만약 Unit 강제 변환이 없었다면 () -> Boolean 이라 타입 불일치였을 것
```

이 관용을 [[13 - 함수 타입과 람다와 함수 참조]]에서 다시 만난다. 여기서의 요점은, `Unit`을 "값 있는 타입"으로 형식화했기 때문에 이런 매끈한 변환 규칙이 성립한다는 것이다. `void`였다면 "값 없음을 값 있음으로 변환"이라는 형용모순을 다뤄야 했을 것이다.

---

## 4. `Unit`은 `void`가 아니다 (M10)

### 4.1 오해의 해부

이 장이 소유한 첫 번째 오개념을 정면으로 교정하자. **오해: `Unit`은 Java·C의 `void`에 대한 Kotlin식 이름일 뿐이다. (M10)** 이 오해는 표면적으로 그럴듯하다 — 둘 다 "쓸모 있는 반환값이 없는 함수"에 등장하니까. 하지만 `void`와 `Unit`은 타입 시스템에서 근본적으로 다른 지위를 갖는다.

`void`(Java·C)는 **타입이 아니거나, 값이 없는 특수 표식**이다. Java에서 `void`는 "이 메서드는 값을 반환하지 않는다"는 메서드 시그니처의 표식이며, 값으로 다룰 수 없다. `void` 타입의 변수를 만들 수 없고, `List<void>` 같은 제네릭 인자로 쓸 수 없고, `void` 값을 다른 함수에 넘길 수 없다. Java에는 별도로 `java.lang.Void`라는 클래스가 있지만, 이 클래스는 인스턴스를 만들 수 없어서 그 유일한 "값"은 `null`이다 — 즉 `Void` 타입 변수에는 `null`밖에 담을 수 없는 절름발이다.

`Unit`(Kotlin)은 **원소가 하나인 진짜 타입**이며, 그 유일한 값은 실재하는 싱글턴 객체다. 변수에 담을 수 있고(`val x: Unit`), 제네릭 인자로 쓸 수 있고(`List<Unit>`), 함수에 넘길 수 있다. `Void`처럼 `null`로 흉내 내는 게 아니라, `Unit`이라는 실체 있는 값이 거기 있다.

| 성질 | `void` (Java/C) | `java.lang.Void` | `Unit` (Kotlin) |
|------|-----------------|------------------|-----------------|
| 타입인가 | 표식(값 아님) | 타입(클래스) | 타입(object) |
| 값의 개수 | 없음 | `null` 하나뿐(인스턴스화 불가) | 싱글턴 하나 |
| 변수에 담기 | 불가 | 가능하나 `null`만 | 가능(`Unit`) |
| 제네릭 인자 | 불가 | 가능하나 `null`만 | 가능(진짜 값) |
| 정보량 | — | 0비트(그러나 null 오염) | 0비트(깔끔) |

### 4.2 왜 이 구분이 실제로 중요한가 — 제네릭 균일성

이 구분이 궤변이 아닌 이유는 제네릭에서 드러난다. "결과를 하나 돌려주는 작업"을 추상화하는 제네릭 타입 `Task<R>`를 생각하자.

```kotlin
class Task<R>(private val body: () -> R) {
    fun run(): R = body()
}

val computeArea: Task<Double> = Task { 3.14 * 2 * 2 }   // 결과 Double
val logMessage: Task<Unit>    = Task { println("done") } // 결과 Unit — 아무 특례 없이 성립!

val area: Double = computeArea.run()   // => 12.56
val nothing: Unit = logMessage.run()   // => Unit. 균일하게 동작
```

`Task<Unit>`이 아무 특례 없이 성립한다는 점이 핵심이다. "값을 반환하는 작업"과 "값을 반환하지 않는 작업"을 *같은 제네릭 틀*로 다룰 수 있다. 만약 Kotlin이 `void`를 썼다면, `Task<void>`는 불법이라 "결과 없는 작업"을 위한 별도의 타입(`Runnable` 같은)을 따로 만들어야 했을 것이다. Java가 실제로 그렇게 한다: 값을 돌려주는 `Callable<V>`와 돌려주지 않는 `Runnable`이 별개의 인터페이스로 갈라져 있고, `Future<Void>`처럼 `Void`+`null` 조합으로 어색하게 메꾼다.

`Unit`을 값 있는 타입으로 만든 덕분에 Kotlin은 이 갈라짐을 없앴다. 함수 타입도 마찬가지다 — `() -> Unit`은 특별한 "프로시저 타입"이 아니라 그냥 결과 타입이 `Unit`인 평범한 함수 타입이다. 이 균일성이 [[13 - 함수 타입과 람다와 함수 참조]]와 [[16 - 스코프 함수와 수신 객체 관용구]], 나아가 코루틴([[42 - suspend와 코루틴 - 언어 수준의 중단]])의 설계까지 깔끔하게 떠받친다.

### 4.3 JVM 바이트코드에서는 결국 `void`가 된다 — 그러나 그것은 구현이다

여기서 정직해야 한다. **JVM 백엔드에서** `Unit`을 반환하는 Kotlin 함수는 바이트코드 수준에서 반환 타입이 `void`인 메서드로 컴파일된다. 매 호출마다 `Unit.INSTANCE`를 실제로 스택에 올려 반환하는 것은 낭비이므로, 컴파일러가 이를 최적화해 그냥 `void`로 만든다.

```text
Kotlin 소스:                  JVM 바이트코드(개념):
fun log(): Unit { ... }   →   public final void log() { ... }   // 반환 타입 void
```

그렇다면 "`Unit`은 `void`가 아니다"라는 이 절의 주장과 모순 아닌가? 아니다. 핵심은 **추상화 계층을 구분하는 것**이다. *언어 타입 시스템*에서 `Unit`은 값 있는 타입이고, 그래서 제네릭·함수 타입·변수 대입에서 일급으로 동작한다. *JVM 백엔드의 코드 생성*에서는 이 값이 정보 0비트임을 이용해 `void`로 눌러 담는 최적화를 한다. 값이 정말로 필요한 자리 — 예컨대 `Function0<Unit>`의 결과, `Task<Unit>`의 타입 인자 — 에서는 백엔드가 `Unit.INSTANCE`를 실제로 사용한다.

이 이중성은 다른 백엔드에서 또 달라진다. Kotlin/Native나 Kotlin/JS에는 JVM `void`가 없으므로 `Unit`은 그 플랫폼 나름의 표현을 갖는다. 그러니 "Unit은 void로 컴파일된다"는 서술은 **JVM 백엔드에 한정된 구현 사실**이지 언어의 정의가 아니다. 언어의 정의는 "Unit은 값이 하나인 타입"이고, 이것이 모든 타깃에서 참이다.

> **성능 주의**: `Unit` 반환 함수가 JVM에서 `void`로 컴파일되므로, 단순 프로시저 호출에는 `Unit` 때문에 생기는 런타임 비용이 사실상 없다. 다만 `List<Unit>`이나 `() -> Unit` 람다를 **박싱된 값으로** 실제 저장·전달하는 자리에서는 `Unit.INSTANCE`가 오간다. 이 인스턴스는 싱글턴이라 새 할당은 없지만, "언어상 값이 있다"는 사실이 완전히 공짜는 아님을 기억할 것.

> **역사 메모**: 값 하나짜리 단위 타입이라는 발상은 Kotlin의 발명이 아니다. ML 계열 언어의 `unit`(값 `()`), Scala의 `Unit`(값 `()`), Haskell의 `()`(유닛) 모두 같은 개념이다. Kotlin은 이 함수형 전통에서 `Unit`을 물려받아, `void`라는 명령형 전통의 절름발이를 대체했다. 이름을 `Unit`으로 택한 것부터가 "이것은 진짜 타입"이라는 선언이다.

---

## 5. `Nothing` — 값이 하나도 없는 밑바닥

### 5.1 공집합에 대응하는 타입

이제 지형의 반대편 극점, 밑바닥으로 내려간다. `Unit`이 원소 하나짜리 집합이라면, `Nothing`은 **원소가 하나도 없는 공집합**에 대응하는 타입이다. 표준 라이브러리에서 `Nothing`은 이렇게 선언되어 있다.

```kotlin
public class Nothing private constructor()
```

생성자가 `private`이고, 표준 라이브러리 어디서도 이 생성자를 호출하지 않는다. 따라서 `Nothing`의 인스턴스는 **결코 존재할 수 없다**. `Nothing` 타입의 값을 만드는 방법은 없다. 집합으로 쓰면 $\text{Nothing} = \varnothing$ — 공집합이다.

"값을 절대 만들 수 없는 타입이 대체 무슨 쓸모가 있는가?"가 자연스러운 반문이다. 값이 없는데 변수에 담을 수도, 함수에 넘길 수도 없다면 죽은 타입 아닌가? 아니다. `Nothing`의 쓸모는 값을 *담는* 데 있지 않고, **부분타입 관계에서의 위치**와 **"이 지점에는 값이 도달하지 않는다"는 정보**를 컴파일러에 전달하는 데 있다. 6~7절에서 이를 파낸다. 먼저 그 위치부터.

### 5.2 `Nothing`은 모든 타입의 하위타입이다

`Nothing`의 정의적 성질은 이것이다: **`Nothing`은 모든 타입의 하위타입이다.** 임의의 타입 `T`에 대해 $\text{Nothing} <: T$가 성립한다. `Nothing <: Int`, `Nothing <: String`, `Nothing <: Coordinate`, `Nothing <: Any?` … 예외 없이 전부.

왜 이것이 자연스러운가? 부분타입 = 부분집합의 관점으로 보면 즉시 풀린다. 공집합은 임의의 집합의 부분집합이다 — $\varnothing \subseteq S$는 모든 $S$에 대해 참이다(공집합의 원소 전부가 $S$에 속한다는 명제는, 원소가 없으니 공허하게 참). `Nothing`이 공집합에 대응하므로, `Nothing`은 모든 타입의 부분집합 = 하위타입이다.

이 성질을 코드로 확인해 보자. `Nothing` 타입의 값을 얻는 유일한 방법은 값을 *만드는* 게 아니라, 정상적으로 반환되지 않는 식 — `throw` — 에서 온다(6절). `throw` 식의 타입은 `Nothing`이다. 그리고 `Nothing`은 모든 타입의 하위타입이므로, `throw`를 어떤 타입이 기대되는 자리에든 놓을 수 있다.

```kotlin
fun bankBalance(accountId: String): Int {
    val account = findAccount(accountId)
        ?: throw NoSuchElementException("계좌 없음")   // throw는 Nothing 타입
        // Nothing <: Int 이므로, Int가 필요한 엘비스 우변에 throw를 놓을 수 있다
    return account.balance
}
```

여기서 엘비스 연산자 `?:`의 좌변 `findAccount(...)`가 널이면 우변이 평가되는데, 우변 `throw ...`의 타입이 `Nothing`이라 그것이 좌변의 비-널 타입 `Account`와 합쳐진다. `Nothing`이 모든 타입 아래에 있기에, 이 합침이 언제나 성립한다. 이 메커니즘의 널 처리 측면은 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]이 M32로 소유하지만, 그것을 떠받치는 타입 시스템의 뼈대 — `Nothing`이 밑바닥이라는 사실 — 는 이 장의 것이다.

### 5.3 지형 완성 — 정상과 밑바닥이 모두 채워졌다

이제 지형이 완성됐다. 정상에 `Any?`, 밑바닥에 `Nothing`이 있고, 나머지 모든 타입이 그 사이에 층층이 쌓인다.

```text
                    Any?              ← ⊤ 정상: 모든 타입의 상위타입, 전체집합
                   /    \
                Any      Nothing?     ← 널 층
               /|\         │
        String Int Coordinate ...     ← 널 아닌 구체 타입들
               \|/         │
             (모든 비-널 T) │
                   \       /
                    Nothing           ← ⊥ 밑바닥: 모든 타입의 하위타입, 공집합
```

부분타입 사슬을 하나 뽑아 보면 다음이 모두 성립한다.

$$\text{Nothing} <: \text{Int} <: \text{Any} <: \text{Any?}$$
$$\text{Nothing} <: \text{Nothing?} <: \text{Int?} <: \text{Any?}$$

밑바닥 `Nothing`은 `Int`뿐 아니라 `Nothing?`의 하위타입이기도 하다($T <: T?$는 항상 참이므로). 정상 `Any?`는 `Any`와 `Nothing?`을 모두 위에서 덮는다. 이 두 극점이 격자를 "닫아" 주어, 임의의 두 타입에 대한 LUB·GLB 계산이 언제나 답을 갖게 된다. `Nothing`이 없으면 예컨대 "아무 값도 도달하지 않는 갈래"의 타입을 표현할 수 없어 타입 추론에 구멍이 생긴다. 지형에 밑바닥이 필요한 이유가 바로 이것이다. 이 바닥 타입(bottom type)의 타입 이론적 배경은 [[16 - 부분 타입과 변성]]과 [[10 - 곱·합·단위·공집합]](거기서는 공집합/void 타입으로 부른다)이 다룬다.

---

## 6. `Nothing`은 `null`이 아니다 (M11)

### 6.1 정반대의 두 극점

이 장이 소유한 두 번째 오개념. **오해: `Nothing`은 `null`이나 "빈 값" 비슷한 무의미한 것이다. (M11)** 이 오해는 대개 이름에서 온다 — "Nothing = 아무것도 없음 = null?" 하지만 `Nothing`과 `null`은 지형에서 *정반대의 극점*에 산다.

`null`은 **값**이다. `Nothing?` 타입의 유일한 값이며, 널 허용 타입이 담을 수 있는 특별한 원소다. `null`은 실재하고, 변수에 담기고, 전달되고, 비교된다.

`Nothing`은 **값이 없는 타입**이다. 인스턴스가 하나도 없는 공집합이다. `Nothing` 타입의 변수에는 아무것도 담을 수 없다 — `null`조차도.

```kotlin
val a: Int? = null          // => OK — null은 Nothing? 타입의 값, Int?에 담긴다
// val b: Nothing = null    // 컴파일 에러: null은 Nothing?이지 Nothing이 아니다
// val c: Nothing = TODO()  // 대입 자체는 타입상 OK지만, TODO()가 예외를 던져 c에 결코 도달 못함
```

집합으로 대비하면 명료하다. `null`이 사는 타입 `Nothing?`은 원소가 하나($\{\texttt{null}\}$)인 집합이고, `Nothing`은 원소가 없는($\varnothing$) 집합이다. `Nothing?`은 `Nothing`에 `null` 하나를 더한 것 — 지형에서 `Nothing` 바로 한 칸 위다.

$$\text{Nothing?} = \text{Nothing} \cup \{\texttt{null}\} = \varnothing \cup \{\texttt{null}\} = \{\texttt{null}\}$$

그래서 `null` 리터럴 자체의 타입은 정확히 `Nothing?`이다. `val x = null`이라고만 쓰면 `x`의 추론 타입은 `Nothing?`이 된다.

```kotlin
val x = null           // x의 추론 타입: Nothing?
// x는 null 말고는 아무것도 담을 수 없다 — Nothing?의 유일한 값이 null이므로
```

### 6.2 `null`은 존재를, `Nothing`은 부재를 형식화한다

두 개념의 역할을 한 문장으로 대비하면 이렇다. **`null`은 "값이 있을 자리에 값이 없음"이라는 상태를 표현하는 값이고, `Nothing`은 "이 코드 지점에는 값이 애초에 도달하지 않음"이라는 제어 흐름을 표현하는 타입이다.** 전자는 데이터의 부재, 후자는 계산의 부재다.

이 대비를 극적으로 보여주는 것이 반환 타입이다. `Nothing`을 반환한다고 선언한 함수는 **정상적으로 반환하는 일이 결코 없는** 함수다. 왜냐하면 `Nothing` 값을 만들어 반환할 방법이 없으므로, 그 함수가 `return`으로 끝나는 것은 불가능하기 때문이다. `Nothing` 반환 함수가 제어권을 벗어나는 길은 오직 둘 — 예외를 던지거나, 영원히 끝나지 않거나 — 뿐이다.

```kotlin
fun fail(message: String): Nothing {
    throw IllegalStateException(message)   // 정상 반환 없음 — 반드시 예외로 이탈
}

fun loopForever(): Nothing {
    while (true) { /* ... */ }             // 정상 반환 없음 — 영원히 돌거나 예외
}

// fun broken(): Nothing { return }        // 컴파일 에러: Nothing 함수는 정상 반환 불가
```

이 "정상 반환하지 않음"이라는 정보야말로 `Nothing`의 알짜배기다. 컴파일러는 `Nothing` 반환 함수의 호출 *이후* 코드를 "도달 불가능(unreachable)"으로 판정할 수 있다. 이 도달 불가능 분석이 7절에서 널 스마트 캐스트, 도달 불가 경고, 완전성 검사로 이어진다.

### 6.3 `Nothing?` — 널만 사는 얇은 층

`Nothing?`은 실무에서 거의 직접 쓰지 않지만, 지형의 정합성을 위해 알아 둘 가치가 있다. `Nothing?`의 값은 `null` 하나뿐이므로, `Nothing?` 타입의 파라미터를 받는 함수는 사실상 `null`만 받는다. 이 성질이 드물게 유용한 경우가 있다 — 예컨대 "오직 널만 허용"을 타입으로 표현하고 싶을 때. 하지만 대개 `null` 리터럴의 타입으로 조용히 등장할 뿐이다.

`Nothing?`이 `Any`의 하위타입이 *아니라는* 점도 지형의 비대칭을 다시 확인해 준다. `Nothing? = {null}`인데 `null`은 `Any`의 값이 아니므로($\{\texttt{null}\} \not\subseteq \text{Any}$), `Nothing?`은 `Any`의 하위타입이 될 수 없다. `Nothing?`은 `Any?`의 하위타입이긴 하지만 `Any`는 건너뛴다. 지형에서 널 층은 이렇게 비-널 층을 우회해 정상 `Any?`로 곧장 연결된다.

```text
   Any?
   /  \
 Any   \
  │      Nothing?      ← Any를 거치지 않고 Any?로 직결 (null은 Any의 값이 아니므로)
  │       │
Nothing ─┘             ← Nothing은 Any의 하위타입이자 Nothing?의 하위타입
```

---

## 7. `Nothing`이 하는 실제 노동 — 제어 흐름과 타입 추론

### 7.1 `throw`·`return`·`break`·`continue`는 모두 `Nothing` 타입이다

Kotlin은 표현식 지향 언어다([[17 - 표현식으로서의 제어 흐름 - if와 when]]). 문(statement)처럼 보이는 제어 구조 상당수가 실은 값을 갖는 식(expression)이며, 그 값의 타입이 `Nothing`인 경우가 많다. 구체적으로 **`throw`, `return`, `break`, `continue`는 모두 `Nothing` 타입의 식**이다.

이들의 공통점은 "평가되면 현재 지점의 정상적 진행을 끝내 버린다"는 것이다. `throw`는 예외로 이탈하고, `return`은 함수를 벗어나고, `break`/`continue`는 루프 흐름을 바꾼다. 어느 것도 "값을 만들어 다음 계산으로 넘긴다"를 하지 않는다. 다음으로 흐를 값이 없다 — 정확히 공집합, 정확히 `Nothing`이다.

```kotlin
val category: String = when (val score = readScore()) {
    in 90..100 -> "A"
    in 80..89  -> "B"
    in 0..79   -> "C"
    else       -> throw IllegalArgumentException("점수 범위 밖: $score")  // 이 갈래는 Nothing
}
// when의 각 갈래 타입: String, String, String, Nothing
// LUB(String, String, String, Nothing) = String → category는 String으로 확정
```

`throw` 갈래가 `Nothing`이므로, `when`의 결과 타입을 계산할 때 그 갈래는 다른 갈래의 타입을 "끌어내리지" 않는다. `Nothing`은 모든 타입 아래에 있어서 LUB 계산에서 흡수되기 때문이다. 그래서 나머지 갈래가 전부 `String`이면 결과는 `Nothing`이 아니라 `String`이다.

### 7.2 최소 상계(LUB)에서 `Nothing`은 항등원처럼 흡수된다

7.1의 계산을 일반화하자. `if/else`나 `when`이 여러 갈래를 가질 때, 전체 식의 타입은 갈래 타입들의 최소 상계(least upper bound)다. 여기서 `Nothing`은 특별한 역할을 한다: **어떤 타입 `T`와 `Nothing`의 LUB는 언제나 `T`** 다.

$$\text{LUB}(T,\ \text{Nothing}) = T \quad (\text{모든 } T\text{에 대해})$$

왜냐하면 $\text{Nothing} <: T$이므로 `Nothing`은 이미 `T` 아래에 있고, 둘을 모두 덮는 가장 작은 타입은 그냥 `T`이기 때문이다. 집합으로는 $S \cup \varnothing = S$ — 공집합과의 합집합은 자기 자신. 그래서 `Nothing`은 LUB 연산의 항등원(identity)처럼 행동한다. 갈래 하나가 `throw`(=`Nothing`)라면, 그 갈래는 결과 타입에 영향을 주지 않고 조용히 흡수된다.

```kotlin
val radius: Double = if (input > 0) input.toDouble()
                     else throw IllegalArgumentException("반지름은 양수")
// LUB(Double, Nothing) = Double → radius는 Double (Double? 아님!)

val name: String = person.nickname ?: return   // return은 Nothing
// LUB(String, Nothing) = String → 엘비스 우변이 함수를 벗어나므로 name은 비-널 String
```

두 번째 예가 특히 중요하다. `person.nickname`이 `String?`(널 허용)일 때, `?: return`으로 널인 경우를 함수 탈출로 처리하면, 그 이후 `name`은 비-널 `String`으로 확정된다. `return`이 `Nothing`이라 엘비스 결과에서 널 가능성이 제거되기 때문이다. 이 관용구 `?: return`, `?: throw`의 널 안전성 측면은 M32로 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]가 소유하지만, 그것을 가능케 하는 원리 — `Nothing`이 LUB에서 흡수된다 — 는 이 장의 타입 지형에서 온다.

### 7.3 도달 불가능 분석과 `TODO()`

`Nothing`이 컴파일러에게 주는 결정적 정보는 "이 지점 이후는 도달 불가능"이다. `Nothing`을 반환하는 함수를 호출한 다음 줄부터는 실행이 결코 이르지 않으므로, 컴파일러는 그 코드를 도달 불가로 표시하고 뒤따르는 타입 검사를 조정한다.

```kotlin
fun computeDiscount(rank: String): Double {
    val rate = when (rank) {
        "gold"   -> 0.2
        "silver" -> 0.1
        else     -> fail("알 수 없는 등급: $rank")   // fail(): Nothing
    }
    return 100.0 * rate   // fail 갈래를 타면 여기 도달 안 함 → rate는 Double로 확정
}
```

`fail`이 `Nothing` 반환이라 `when`의 `else` 갈래 이후로는 값이 흐르지 않는다. 그래서 `rate`는 `Double`(널 아님, `Any` 아님)로 깔끔하게 추론된다. 만약 `fail`의 반환 타입이 `Unit`이었다면 `else` 갈래가 `Unit`을 내놓아 `rate`의 타입이 `Any`로 뭉개졌을 것이다. `Nothing`과 `Unit`의 이 차이가 실무에서 타입 추론의 품질을 가른다.

표준 라이브러리는 이 원리를 여러 도우미로 노출한다. 전부 반환 타입이 `Nothing`이다.

```kotlin
public inline fun TODO(): Nothing = throw NotImplementedError()
public inline fun TODO(reason: String): Nothing = throw NotImplementedError("...: $reason")
public inline fun error(message: Any): Nothing = throw IllegalStateException(message.toString())

// 사용례
fun area(shape: Shape): Double = when (shape) {
    is Circle -> Math.PI * shape.r * shape.r
    is Square -> shape.side * shape.side
    // else 없이 두면? sealed일 때 완전성 검사가 통과. 미구현 갈래는 TODO()로 채운다
    is Triangle -> TODO("삼각형 넓이 미구현")   // Nothing이라 반환 타입 Double을 깨지 않음
}
```

`TODO("삼각형 넓이 미구현")`이 `Double`이 필요한 자리에 놓일 수 있는 것은 `Nothing <: Double` 덕분이고, 그 자리를 채워도 함수 전체가 여전히 `Double`을 반환한다고 컴파일러가 믿을 수 있는 것은 `Nothing`이 LUB에서 흡수되기 때문이다. 미구현 뼈대를 타입 검사를 통과시키면서 남겨 두는 이 관용구가 `Nothing`의 가장 흔한 일상적 쓸모다. 예외와 `Nothing`의 더 깊은 연결 — 예외 계층, 검사 예외의 부재, `runCatching`/`Result` — 은 [[41 - 예외와 Nothing]]이 소유한다.

### 7.4 공변 위치에서의 `Nothing` — 빈 컬렉션의 원소 타입

`Nothing`이 밑바닥이라는 성질은 제네릭 변성([[33 - 제네릭 1 - 타입 파라미터와 변성]])과 만나 빼어나게 실용적인 결과를 낳는다. 대표적인 것이 빈 컬렉션이다. 원소가 하나도 없는 리스트의 원소 타입은 무엇이어야 할까? "무엇이든 될 수 있어야 한다" — 빈 `List<String>`으로도, 빈 `List<Coordinate>`로도 쓸 수 있어야 한다. 이 요구를 정확히 만족하는 원소 타입이 `Nothing`이다.

`List`는 원소 타입에 대해 공변(covariant, `out`)이다. 즉 `A <: B`이면 `List<A> <: List<B>`다([[38 - 컬렉션 - 읽기 전용과 가변]]). `Nothing`은 모든 타입의 하위타입이므로, `List<Nothing>`은 *모든* `List<T>`의 하위타입이 된다.

$$\forall T:\ \text{Nothing} <: T \implies \text{List<Nothing>} <: \text{List<}T\text{>}$$

그래서 표준 라이브러리의 빈 리스트 싱글턴은 실제로 `List<Nothing>` 타입이다.

```kotlin
// 표준 라이브러리 내부(개념):
internal object EmptyList : List<Nothing> { /* size=0, get은 예외 ... */ }

public fun <T> emptyList(): List<T> = EmptyList   // List<Nothing>을 List<T>로 안전하게 반환

val strings: List<String>     = emptyList()   // => List<Nothing>이 List<String>으로 통용
val points:  List<Coordinate> = emptyList()   // => 같은 EmptyList 인스턴스를 재사용 가능
```

원소가 없으니 원소 타입이 `Nothing`이어도 `get`으로 `Nothing` 값을 꺼낼 위험이 없다 — 애초에 꺼낼 원소가 없으니까. 이것이 "값 없는 타입"이 "값 없는 컬렉션"의 원소 타입으로 완벽히 들어맞는 이유다. 공집합의 원소 타입은 공집합 타입 `Nothing`이다. 같은 논리가 `emptySet()`, `emptyMap()`(값 쪽), 그리고 예외를 담지 않는 성공 결과 등에도 적용된다.

> **명세 정밀**: `emptyList<T>()`가 내부적으로 `List<Nothing>`인 싱글턴을 `List<T>`로 반환하는 것은 안전하다. `List<T>`는 읽기 전용(생산자) 인터페이스라 `T` 값을 *받는* 연산이 없고, `Nothing`으로부터의 공변 상향만 일어나기 때문이다. 만약 `MutableList<Nothing>`을 `MutableList<String>`으로 통용시키려 했다면 `add("x")`가 `Nothing` 자리에 `String`을 넣는 셈이 되어 불건전(unsound)했을 것이다. 그래서 이 트릭은 공변 읽기 전용 컬렉션에서만 성립한다.

---

## 8. 합집합과 교집합 — 지형에 들어오는 두 연산

### 8.1 Kotlin에는 일반 합집합 타입이 없다

TypeScript를 아는 독자라면 `String | Int`처럼 "이것 아니면 저것"을 뜻하는 합집합 타입(union type)에 익숙할 것이다. Scala 3에도 `String | Int`가 있다. **Kotlin에는 이런 일반적 합집합 타입이 없다.** `String`이거나 `Int`인 값을 담는 타입을 직접 적을 수 없다. 이는 의도된 설계 결정이다 — 임의 합집합은 타입 추론과 오버로드 해소를 복잡하게 만들고, Kotlin은 그 복잡성 대신 다른 도구들로 같은 필요를 메운다.

그렇다면 "여러 타입 중 하나"를 어떻게 표현하는가? Kotlin은 세 가지 우회로를 제공한다.

**첫째, 공통 상위타입으로 올린다.** `String`과 `Int`가 모두 필요하면 둘의 공통 상위타입인 `Any`(또는 `Comparable<*>` 등)로 받는다. 대가는 정밀함의 상실 — `Any`로 받으면 다시 `is`로 좁혀야 한다.

**둘째, 봉인 클래스(sealed class/interface)로 닫힌 합을 만든다.** "이것 아니면 저것"의 경우가 유한하고 미리 알려져 있다면, `sealed`로 대수적 합타입(sum type)을 명시적으로 만든다. 이것이 Kotlin이 권장하는 정공법이며 [[30 - sealed 클래스와 대수적 데이터 타입]]이 소유한다.

```kotlin
sealed interface Shape
data class Circle(val r: Double) : Shape
data class Rectangle(val w: Double, val h: Double) : Shape
// Shape는 "Circle | Rectangle"의 닫힌 합을 타입 하나로 표현한다. when 완전성 검사도 붙는다
```

**셋째, 널 가능성 — 이것만은 언어에 내장된 특수한 합집합이다.** `String?`은 사실상 `String | Null`이라는 합집합이다. Kotlin은 일반 합집합은 거부하면서 "타입 `T` 또는 널"이라는 *한 가지 특수한 합집합*만은 문법(`?`)으로 특별 대우한다. 8.2에서 이 관점을 정밀화한다.

### 8.2 `T?`는 특수한 합집합 타입이다

지형의 관점에서 널 가능성을 다시 보자. `String?`의 값 집합은 `String`의 값들에 `null` 하나를 더한 것이다.

$$\text{String?} = \text{String} \cup \{\texttt{null}\} = \text{String} \cup \text{Nothing?}$$

즉 `String?`은 `String`과 `Nothing?`의 합집합이다. Kotlin이 일반 합집합을 안 가지면서도 이 합집합만은 지원하는 이유는, 널 가능성이 실무에서 압도적으로 흔하고, 딱 이 한 형태(`T` 또는 널)로 한정하면 타입 추론이 다룰 만하기 때문이다. `?`는 "지형에서 `null`이라는 원소 하나를 추가한 한 칸 위 타입"으로 올라가는, 잘 통제된 합집합 연산자다.

```kotlin
val maybeName: String? = if (hasName) "Rex" else null
// maybeName의 타입 String?은 String ∪ {null}
// - "Rex"는 String 부분에서 옴
// - null은 Nothing? 부분에서 옴 (null의 타입이 Nothing?이므로)

when (maybeName) {
    null -> println("이름 없음")   // Nothing? 부분을 소진
    else -> println(maybeName.length)  // 스마트 캐스트로 String 부분으로 좁혀짐
}
```

`when`으로 `null` 경우를 갈라내면 `else` 갈래에서 `maybeName`은 합집합의 `String` 부분으로 좁혀진다 — 이것이 스마트 캐스트다(8.4). 합집합을 갈라 각 부분으로 좁히는 이 흐름이 널 안전성의 작동 원리이고, 그 상세는 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]가 소유한다(M7, M13). 이 장에서는 "`?`가 지형에서 하는 일 = 특수 합집합"이라는 좌표만 잡는다.

### 8.3 교집합 타입은 있다 — 스마트 캐스트가 만든다

합집합의 쌍대(dual)는 교집합(intersection)이다. `A & B`는 "`A`이면서 동시에 `B`인 값"의 타입이다. Kotlin은 일반 합집합은 없지만, **교집합 타입은 스마트 캐스트의 결과로 실제로 존재한다.** 다만 대부분은 소스 코드에 직접 적을 수 없는 비-표기(non-denotable) 타입으로만 나타난다.

값 하나가 두 개의 `is` 검사를 동시에 통과하면, 컴파일러는 그 값을 두 타입의 교집합으로 좁힌다.

```kotlin
interface Named { val name: String }
interface Aged  { val age: Int }

fun greet(x: Any) {
    if (x is Named && x is Aged) {
        // 이 블록에서 x의 정적 타입: Named & Aged  (교집합 — 소스에는 못 적지만 컴파일러는 안다)
        println("${x.name}, ${x.age}세")   // 두 인터페이스의 멤버 모두 접근 가능
    }
}
```

`x is Named && x is Aged` 이후 블록에서 `x`는 `Named`이면서 `Aged`인 값이다. 컴파일러는 `x`의 타입을 교집합 `Named & Aged`로 두어, `name`과 `age`를 모두 스마트 캐스트 없이 부를 수 있게 한다. 이 `Named & Aged`는 우리가 변수 선언에 `val y: Named & Aged`처럼 적을 수 없는(2.x 기준 일반 교집합은 비표기) 타입이지만, 타입 검사 내부에서는 엄연히 존재하며 K2 컴파일러가 정밀하게 추적한다.

집합 관점으로는 자명하다 — `Named & Aged`는 `Named`의 값 집합과 `Aged`의 값 집합의 교집합이다. 두 조건을 모두 통과한 `x`는 정확히 그 교집합에 있다. 합집합이 지형에서 "위로 넓히기"라면 교집합은 "아래로 좁히기"다. 스마트 캐스트는 지형을 아래로 좁히는 연산이고, 그 원리적 배경이 이 교집합 타입이다.

### 8.4 정의상 비-널 타입 `T & Any` — 소스에 적을 수 있는 유일한 교집합

일반 교집합은 못 적지만, Kotlin 1.7 이후(2.x 포함) *딱 한 형태*의 교집합은 소스에 적을 수 있다: **정의상 비-널 타입(definitely non-nullable type)** `T & Any`다. 이것은 "타입 파라미터 `T`이면서 동시에 `Any`(널 아님)인 값" — 즉 `T`에서 널 가능성을 제거한 교집합이다.

이 표기가 필요한 이유는 제네릭에 있다. 타입 파라미터 `T`의 기본 상한이 `Any?`(M45)라, `T`는 널을 포함할 수 있다. 그런데 "`T`가 무엇이든, 이 자리에는 널이 아닌 `T`를 원한다"를 표현하고 싶을 때가 있다. 상한을 `T : Any`로 못박으면 호출자가 널 허용 타입 인자를 쓸 수 없어 너무 강하다. 대신 `T & Any`를 쓰면, `T`는 널 허용이어도 되면서 이 특정 자리만 비-널로 좁힐 수 있다.

```kotlin
// T는 널 허용까지 허용하되, 반환값만은 반드시 비-널로 좁히고 싶다
fun <T> elvisLike(value: T, fallback: T & Any): T & Any =
    value ?: fallback
    // value: T (널일 수도), fallback: T & Any (비-널), 결과: T & Any (비-널)

val r1: String = elvisLike<String?>(null, "default")   // => "default" (비-null String)
val r2: Int    = elvisLike(null, 0)                    // => 0
```

`T & Any`는 지형에서 "`T`가 걸쳐 있던 널 층을 잘라내고 `Any` 아래로 좁힌" 교집합이다. Java 상호운용에서 플랫폼 타입([[45 - Java 상호운용]], M8)의 널 가능성을 다룰 때 특히 요긴하다. 이것이 Kotlin에서 소스에 명시적으로 적을 수 있는 유일한 교집합 타입이며, 일반 교집합(8.3)은 여전히 스마트 캐스트 내부에만 산다.

> **명세 정밀**: `T & Any`에서 `Any`는 "널이 아님"을 뜻하는 상한 역할이다. `T`가 이미 비-널(`T : Any`)이면 `T & Any`는 그냥 `T`와 같다. `T`가 널 허용이면 `T & Any`는 그 널을 벗겨낸 것이다. 이 문법은 K2가 정착하기 전 1.7에서 안정화됐고 2.x에서 표준으로 쓰인다. 일반적인 두 클래스의 교집합(`Named & Aged` 같은)을 소스에 직접 적는 것은 별개 문제로, 2.x 기준 언어 문법으로 지원되지 않는다.

### 8.5 유연한 타입 — 지형의 안개 지대

지형에는 극점(정상·밑바닥)과 명료한 층 말고도 "안개 낀 지대"가 하나 있다: 유연한 타입(flexible type), 통칭 플랫폼 타입이다. Java 코드에서 넘어온 값은 Kotlin 컴파일러가 널 가능성을 확정할 수 없어, `String!`처럼 표기되는 유연한 타입 `(String..String?)`으로 다룬다. 이것은 "`String`처럼 써도 되고 `String?`처럼 써도 되지만, 널 검사를 컴파일러가 강제하지 않는" 위험 지대다.

유연한 타입은 이 장의 소유가 아니다 — 그 정밀한 규칙과 함정(NPE가 컴파일 타임 검사를 뚫고 새는 지점)은 M8로 [[45 - Java 상호운용]]과 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]가 소유한다. 여기서는 지형 지도에 이 안개 지대의 위치만 표시해 둔다: 플랫폼 타입은 지형에서 `T`와 `T?` 사이 어딘가에 걸친, 컴파일러가 경계를 확정하길 유예한 구간이다. Kotlin 코드 안에서 명시적 타입으로 `T`나 `T?`로 못박는 순간 안개가 걷히고 확정된 좌표를 얻는다.

---

## 9. 지형 위를 걷는 규칙 — 정적·강타입의 실전

### 9.1 위로는 자동, 아래로는 명시 — 두 방향의 비대칭

지형의 등고선을 오르내리는 규칙을 정리하자. **하위타입에서 상위타입으로(위로) 올라가는 것은 자동이고 공짜다.** 개를 동물로, 정수를 `Any`로, 무엇이든 `Any?`로 보는 것은 별도 문법 없이 대입만으로 된다. 이를 상향 변환(upcast)이라 하며, 항상 안전하다 — 부분집합의 원소는 언제나 상위집합의 원소이므로.

**반대로 상위타입에서 하위타입으로(아래로) 내려가는 것은 명시적 검사·캐스트를 요구한다.** `Any`로 받은 값을 `Coordinate`로 보려면 `as`(단정 캐스트), `is`(검사 후 스마트 캐스트), `as?`(안전 캐스트) 중 하나를 명시해야 한다. 하향 변환(downcast)은 런타임에 실패할 수 있기 때문이다 — 상위집합의 원소가 반드시 하위집합에 있는 것은 아니다.

```kotlin
val any: Any = Coordinate(1, 2)      // 상향: Coordinate → Any, 자동

// val c: Coordinate = any            // 컴파일 에러: Any는 Coordinate가 아니다 (하향은 자동 불가)
val c1 = any as Coordinate            // 단정 캐스트: 틀리면 런타임 ClassCastException
val c2 = any as? Coordinate           // 안전 캐스트: 틀리면 null (c2: Coordinate?)
if (any is Coordinate) {
    println(any.x)                    // 스마트 캐스트: is 검사 후 블록 안에서 Coordinate로 좁혀짐
}
```

이 비대칭 — 위로는 무언, 아래로는 명시 — 이 정적·강타입 규율의 알맹이다. 컴파일러는 "안전한 방향(위)"만 조용히 허락하고 "위험한 방향(아래)"은 개발자가 책임지고 표기하게 강제한다. `Nothing`이 밑바닥인 것과 `Any?`가 정상인 것은, 이 오르내림의 두 끝점을 정의하는 말뚝이다.

### 9.2 타입 추론은 지형에서 좌표를 찾는 일이다

Kotlin은 타입을 일일이 적지 않아도 되는 강력한 지역 타입 추론(local type inference)을 갖는다. `val x = "cat"`에서 `x`가 `String`임을 컴파일러가 안다. 이 추론의 상당 부분이 이 장에서 본 지형 위의 계산 — 특히 LUB 계산 — 이다.

```kotlin
val items = listOf(Circle(1.0), Rectangle(2.0, 3.0))   // items의 추론 타입은?
// 원소 타입 = LUB(Circle, Rectangle). 둘의 공통 상위타입이 sealed Shape라면 List<Shape>
// (Circle, Rectangle이 무관하면 LUB는 Any 근처까지 올라간다)

val values = listOf(1, 2.0, "three")   // LUB(Int, Double, String) = Comparable<*> & Serializable 근처
// 실제 추론: List<Comparable<*>> 또는 그 비슷한 공통 상위타입 — 정밀 타입은 무관 타입들의 LUB
```

무관한 타입들을 섞으면 LUB가 `Any`나 `Comparable<*>` 같은 넓은 타입까지 올라가, 원소를 다시 쓰려면 `is`로 좁혀야 한다. 반대로 `sealed` 계층 안의 타입들만 섞으면 LUB가 그 봉인된 상위타입에서 멈춰 유용한 타입이 나온다. "추론된 타입이 왜 이렇게 넓지?"라는 흔한 당혹은, 대개 갈래들의 LUB가 예상보다 위로 올라갔기 때문이다. 지형을 알면 이 당혹이 계산 가능한 결과로 바뀐다.

### 9.3 `Any` 남용이라는 안티패턴 — 지형을 포기하는 코드

지형이 주는 안전을 스스로 반납하는 흔한 실수가 `Any`(또는 `Any?`) 남용이다. "무엇이든 담을 수 있으니 편하다"며 파라미터·필드 타입을 `Any`로 두면, 컴파일러의 정적 검사를 통째로 꺼 버리는 셈이 된다. `Any`로 받은 값은 다시 `is`/`as`로 좁혀야만 쓸 수 있고, 그 좁히기가 틀리면 런타임에 터진다.

```kotlin
// 안티패턴: 지형 정상으로 도망친 코드
fun process(data: Any): Any {
    return when (data) {
        is Int -> data * 2
        is String -> data.uppercase()
        else -> throw IllegalArgumentException("지원 안 함")
    }
}
// 호출자는 반환 Any를 또 좁혀야 한다. 컴파일러가 지켜 줄 게 없다.

// 나은 설계: 지형의 적절한 층 — 제네릭이나 sealed로 정밀하게
fun <T : Number> doubled(data: T): Double = data.toDouble() * 2
// 또는 유한한 경우면 sealed 계층으로(30장). 타입이 곧 문서이자 검사다.
```

`Any`는 정말로 이질적 컨테이너를 다뤄야 할 때(리플렉션, 직렬화 경계 등)의 최후 수단이지, 설계의 기본값이 아니다. 지형의 정상은 "모든 것을 받는다"는 편의를 주지만, 그 대가로 "아무것도 보장하지 않는다"는 위험을 함께 준다. 좋은 Kotlin은 값이 사는 정확한 층을 타입으로 지목한다 — 그것이 제네릭([[33 - 제네릭 1 - 타입 파라미터와 변성]])이든, 봉인 계층([[30 - sealed 클래스와 대수적 데이터 타입]])이든, 인터페이스([[25 - 인터페이스]])든. 타입을 좁게 잡을수록 컴파일러가 더 많이 지켜 준다.

### 9.4 멀티플랫폼에서도 지형은 그대로다

마지막으로 이 지형이 어느 타깃에서나 동일하다는 점을 못박자. `Any`/`Any?`/`Unit`/`Nothing`은 [[01 - 코틀린이란 무엇인가 - 설계 철학과 세 개의 타깃]]에서 본 네 타깃 — JVM·Native·JS·Wasm — 전부에서 같은 부분타입 규칙을 따른다. `Nothing`은 모든 타깃에서 밑바닥이고, `Unit`은 모든 타깃에서 값 하나짜리 타입이다.

다만 *물리적 구현*은 타깃마다 다르다. JVM에서 `Any`는 `java.lang.Object`로, `Unit` 반환은 `void` 메서드로 내려간다. Native·JS·Wasm에는 `Object`도 JVM `void`도 없으니, 각 백엔드가 나름의 표현으로 같은 의미론을 구현한다. 그래서 "`Any`는 `Object`다", "`Unit`은 `void`로 컴파일된다"는 서술은 언제나 **JVM 백엔드에 한정된 구현 사실**로 읽어야 한다(M2 참조). 언어의 정의 — 정상은 `Any?`, 밑바닥은 `Nothing`, `Unit`은 값 하나, `Nothing`은 값 없음 — 는 구현 위에 있고, 그것이 이 장이 그린 지도의 진짜 내용이다.

---

지형을 한 바퀴 돌았다. 처음 던진 질문들이 이제 한 줄기로 회수된다. Kotlin의 타입들은 "타입 = 값의 집합"이라는 관점 아래 하나의 부분타입 격자를 이루며, 그 정상은 `Any`가 아니라 `Any?`(모든 값 + `null`)고, 밑바닥은 `Nothing`(공집합)이다. `Any`는 JVM에서 `Object`로 매핑되지만 `wait`/`getClass`가 없는 더 작고 다른 타입이며, 원시 값까지 아래에 두는 더 넓은 개념이다. `Unit`은 `void`가 아니라 원소가 하나인 진짜 타입 — 그래서 제네릭·함수 타입에서 일급으로 균일하게 동작하고, JVM에서 `void`로 눌리는 것은 언어가 아닌 백엔드의 최적화다(M10). `Nothing`은 `null`의 사촌이 아니라 정반대 극점 — `null`이 "값의 부재를 표현하는 값"이라면 `Nothing`은 "계산의 부재를 표현하는, 값 없는 타입"이다(M11). 값이 하나도 없는 이 타입이 언어에 필요한 이유는, 모든 타입의 하위타입으로서 `throw`·`return`에 타입을 부여하고, LUB에서 흡수되어 `if (c) 1 else throw`를 정확히 `Int`로 만들고, 빈 컬렉션의 원소 타입과 미구현 뼈대(`TODO()`)를 떠받치기 때문이다. Kotlin에 일반 합집합은 없지만 `T?`라는 특수 합집합과 `T & Any`라는 표기 가능한 교집합이 있고, 스마트 캐스트는 지형을 아래로 좁히는 교집합 연산이다. 이 지도를 손에 넣었으니, 이제 그 위의 산맥 하나하나 — 널 안전성, 수, 문자열, 동등성 — 를 정밀 답사할 차례다.

---

## 핵심 요약

- **타입은 값의 집합이고, 부분타입 관계는 부분집합 관계다.** `S <: T`는 곧 `S`의 값 전부가 `T`의 값이라는 뜻이며, 이 대응이 지형의 정상(`Any?`=전체집합)과 밑바닥(`Nothing`=공집합)을 자연스럽게 규정한다.
- **진짜 최상위 타입은 `Any`가 아니라 `Any?`다.** `Any`는 "널 아닌 모든 값", `Any? = Any ∪ {null}`이 모든 타입의 상위타입이다. 이 한 칸 차이가 널 안전성의 기하학적 원점이며, 타입 파라미터의 기본 상한도 `Any?`다(M45 참조).
- **`Any`는 `java.lang.Object`가 아니다.** JVM 백엔드에서 `Object`로 매핑되지만, `Any`에는 `wait`/`notify`/`getClass`가 없고(각각 코루틴·`::class`로 대체) 원시 값까지 하위에 둔다. "`Any`=`Object`"는 런타임 매핑일 뿐 언어 수준의 동일성이 아니다.
- **`Unit`은 `void`가 아니라 원소가 하나뿐인 진짜 타입이다. (M10)** `object Unit`이라는 싱글턴이 그 유일한 값이며, 정보량은 0비트지만 값으로서 변수·제네릭 인자·함수 결과에 일급으로 쓰인다. 이 덕분에 "값 반환"과 "값 없는 프로시저"를 하나의 제네릭·함수 타입 틀로 균일하게 다룬다.
- **`Unit`이 JVM에서 `void`로 컴파일되는 것은 언어가 아니라 백엔드 최적화다. (M10)** 언어 정의상 `Unit`은 값 있는 타입이고, JVM 코드 생성이 정보 0비트를 이용해 `void`로 누른다. 값이 실제로 필요한 자리에서는 `Unit.INSTANCE`가 쓰이며, Native/JS/Wasm에는 JVM `void` 자체가 없다.
- **`Nothing`은 `null`이 아니라 값이 하나도 없는 밑바닥 타입이다. (M11)** `null`은 `Nothing?`의 유일한 값(존재)이고, `Nothing`은 인스턴스가 없는 공집합(부재)이다. `null` 리터럴 자체의 타입은 `Nothing?`이며, `Nothing`과 `null`은 지형의 정반대 극점에 산다.
- **`Nothing`은 모든 타입의 하위타입이다.** 공집합은 임의 집합의 부분집합이므로 `Nothing <: T`가 모든 `T`에 성립한다. 그래서 `throw`·`return`·`break`·`continue`(모두 `Nothing` 타입)를 어떤 타입이 요구되는 자리에든 놓을 수 있다.
- **`Nothing`은 LUB 계산에서 항등원처럼 흡수된다.** `LUB(T, Nothing) = T`이므로 `if (c) 1 else throw e`는 정확히 `Int`, `x ?: return`은 비-널 타입으로 확정된다. `Nothing` 반환 함수(`fail()`, `TODO()`, `error()`)는 정상 반환하지 않아 이후 코드를 도달 불가로 만들고 타입 추론을 정밀하게 유지한다(엘비스 제어 흐름은 M32로 06장).
- **빈 컬렉션의 원소 타입은 `Nothing`이다.** `List`가 공변이고 `Nothing`이 밑바닥이라 `List<Nothing> <: List<T>`가 모든 `T`에 성립한다. 그래서 하나의 빈 리스트 싱글턴(`List<Nothing>`)을 `List<String>`이든 무엇이든으로 안전하게 재사용한다(변성은 33장).
- **Kotlin에 일반 합집합 타입은 없지만, `T?`는 특수 합집합이다.** `String? = String ∪ {null} = String ∪ Nothing?`. 임의 합집합(`String | Int`)은 봉인 클래스나 공통 상위타입으로 우회하고(30장), 널 가능성만 언어가 `?` 문법으로 특별 대우한다.
- **교집합 타입은 스마트 캐스트가 만든다.** `x is A && x is B` 후 `x`는 비표기 교집합 `A & B`로 좁혀진다. 소스에 적을 수 있는 유일한 교집합은 정의상 비-널 타입 `T & Any`(널 벗기기)뿐이며, 이는 제네릭·플랫폼 타입 처리에 쓰인다(스마트캐스트 상세는 M13/06장).
- **지형은 위로는 자동, 아래로는 명시다.** 상향 변환(하위→상위)은 대입만으로 공짜이고 안전하지만, 하향 변환(상위→하위)은 `as`/`is`/`as?`로 명시해야 하며 실패할 수 있다. `Any` 남용은 이 안전을 반납하는 안티패턴이고, 좋은 설계는 값이 사는 정확한 층을 타입으로 지목한다.

## 연결 노트

- [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]] — 이 장이 그린 `T`/`T?`의 한 칸 차이와 `Nothing`의 흡수가 실제 안전 호출·엘비스·스마트 캐스트로 전개되는 곳(M7, M8, M13, M32). 지형의 널 산맥을 정밀 답사.
- [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]] — "모든 것은 객체"이면서 원시 타입이 사라지지 않는 진상과, `Int`를 `Any`/`Any?`로 올릴 때 일어나는 박싱의 실체(M4, M5). 지형을 오를 때의 런타임 비용.
- [[10 - 불리언과 동등성과 동일성]] — `Any`가 선언한 `equals`/`hashCode`의 계약과 구조적 동등 `==` 대 참조 동일 `===`(M6, M48). 이 장이 `Any`의 세 멤버로 예고한 것.
- [[17 - 표현식으로서의 제어 흐름 - if와 when]] — `if`/`when`이 표현식이라 갈래들의 LUB로 타입이 정해지고, `Nothing` 갈래가 흡수되는 원리를 전면 전개(M12).
- [[41 - 예외와 Nothing]] — `throw`가 왜 `Nothing`인지, 검사 예외의 부재와 예외 계층, `try`가 표현식인 점(M30, M31). 이 장의 `Nothing`을 예외의 관점에서 심화.
- [[30 - sealed 클래스와 대수적 데이터 타입]] — 일반 합집합의 부재를 메우는 정공법. 닫힌 합타입과 `when` 완전성으로 "이것 아니면 저것"을 타입으로 표현(M23).
- [[33 - 제네릭 1 - 타입 파라미터와 변성]] — 타입 파라미터 기본 상한 `Any?`(M45), 공변으로 `List<Nothing> <: List<T>`가 성립하는 원리(M20). 지형과 변성의 결합.
- [[10 - 곱·합·단위·공집합]] — (Type Theory 자매 시리즈) 단위 타입(Unit)과 공집합/공백 타입(Nothing에 대응)의 이론적 정본. 이 장의 집합론적 은유의 뿌리.
- [[16 - 부분 타입과 변성]] — (Type Theory 자매 시리즈) 바닥 타입(bottom)·정상 타입(top)과 부분타입 규칙의 형식적 근거. `Nothing <: T`의 이론.
- [[05 - 값과 타입의 지형]] — (JavaScript 자매 시리즈) 같은 "지형" 주제의 대조군. `undefined`/`null` 두 종류와 동적 타입의 지형을, Kotlin의 정적 지형과 견줘 볼 것.
