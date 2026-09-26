---
title: 타입 시스템의 지형 - Any와 Unit과 Nothing
date: 2026-07-13
tags: [kotlin, type-system, any, unit, nothing, 학습노트]
---

[[04 - 파일과 패키지와 선언 - 프로그램의 골격]]에서는 파일, 패키지, 최상위 선언, `main`으로 프로그램의 골격을 세웠다. 골격은 이름이 존재하는 공간을 정의했지만, 그 이름이 *어떤 값을 가리킬 수 있는지*는 아직 정하지 않았다. 이 장에서는 그 질문에 답한다.

Kotlin의 타입은 서로 무관하게 흩어져 있지 않다. 모든 타입은 부분타입(subtype) 관계로 연결된 하나의 격자(lattice)를 이루며, 이 격자에는 꼭대기가 하나, 바닥이 하나 있다. 이 구조를 이해하면 나머지 45개 장에서 만날 타입 규칙을 모두 "이 격자의 어디에 있는가"라는 질문 하나로 정리할 수 있다.

이 장에서는 격자의 끝에 있는 타입 세 개를 자세히 살펴본다. 꼭대기에 있는 `Any`(와 그보다 한 단계 위의 `Any?`), 아무 값도 반환하지 않는 것처럼 보이는 함수가 실제로 반환하는 값인 `Unit`, 그리고 값이 하나도 없는 바닥 타입 `Nothing`이다.

세 타입을 이해하는 공통 관점은 "타입은 값의 집합이다"라는 것이다. `Any`는 (거의) 모든 값을 담은 큰 집합이고, `Unit`은 원소가 하나뿐인 집합이며, `Nothing`은 공집합이다. 이 집합론적 관점으로 보면 `Nothing`이 모든 타입의 하위타입인 이유와 `Unit`이 `void`와 다른 이유를 한 번에 설명할 수 있다.

안전 호출 `?.`, 엘비스 `?:`, 스마트 캐스트가 적용되는 정확한 조건 같은 `null` 가능성의 세부 사항은 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]에서 다룬다.

원시 타입이 없다는 사실("모든 것은 객체")이 JVM 수준에서 어떻게 구현되는지는 [[07 - 수 1 - Int와 Long과 부동소수점]]과 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]에서, 예외가 왜 `Nothing` 타입을 갖는지와 예외 계층은 [[41 - 예외와 Nothing]]에서 설명한다. 이 장은 그 세부로 들어가기 전에 전체 구조를 먼저 그린다.

---

## 1. 타입은 값의 집합이다: 지형이라는 비유

### 1.1 왜 "지형"인가

프로그래밍 언어의 타입을 처음 배울 때는 보통 타입을 "값에 붙는 라벨"로 생각한다. `42`에는 `Int`라는 라벨이, `"cat"`에는 `String`이라는 라벨이 붙는다는 식이다. 이 관점이 틀리지는 않지만 깊이가 부족하다.

더 유용한 관점은 집합론에서 나온다. 즉 **타입은 그 타입에 속하는 값들의 집합**이다. `Boolean`은 `{true, false}`라는 원소 두 개짜리 집합이고, `Int`는 $-2^{31}$부터 $2^{31}-1$까지 약 43억 개의 정수로 이루어진 집합이며, `String`은 (메모리가 허락하는 한) 무한에 가까운 문자열의 집합이다.

이 관점을 택하면 타입 사이의 관계를 집합 사이의 관계로 옮겨 생각할 수 있다. 그중 가장 중요한 것이 **부분타입 관계**(subtyping)이다. 타입 $S$가 타입 $T$의 부분타입이라는 것($S <: T$)은 집합의 언어로 말하면 $S$의 값이 모두 $T$의 값이기도 하다는 뜻, 즉 $S \subseteq T$라는 뜻이다.

개는 동물이므로 `Dog`의 인스턴스는 모두 `Animal`의 인스턴스이기도 하다. 따라서 `Dog <: Animal`이다. 부분타입 관계가 부분집합 관계와 대응한다는 이 사실이 이 장 전체의 기본 틀이다.

```kotlin
open class Animal(val name: String)
class Dog(name: String) : Animal(name)

val a: Animal = Dog("Rex")   // => OK. Dog의 값은 Animal의 값이기도 하다 (Dog <: Animal)
// val d: Dog = Animal("?")  // 컴파일 에러: Animal은 Dog가 아니다. 부분집합 방향이 반대
```

집합 관점이 유용한 이유는 극단적인 집합, 즉 전체집합과 공집합에 대응하는 타입이 무엇인지 자연스럽게 묻게 되기 때문이다. "모든 값을 담은 집합"에 해당하는 타입이 있는가? 있다. 이 장에서 다루는 `Any`(정확히는 `Any?`)이다. "원소가 하나도 없는 공집합"에 해당하는 타입도 있는가? 있다. 바로 `Nothing`이다. 이 두 끝 사이에 나머지 모든 타입이 층층이 쌓여 지형을 이룬다.

### 1.2 부분타입 격자의 꼭대기와 바닥

부분타입 관계는 단순한 순서가 아니라 격자(lattice)에 가까운 구조를 이룬다. 격자의 핵심 성질은 두 가지다. 임의의 두 타입에는 둘을 모두 포함하는 가장 작은 공통 상위타입(least upper bound, LUB, 합집합에 가깝다)이 있고, 둘에 모두 포함되는 가장 큰 공통 하위타입(greatest lower bound, GLB, 교집합에 가깝다)이 있다.

Kotlin 컴파일러가 `if/else`나 `when`의 여러 분기를 하나의 타입으로 묶을 때, 그리고 스마트 캐스트로 타입을 좁힐 때 하는 일이 바로 이 LUB와 GLB 계산이다.

이 구조가 완전한 격자가 되려면 꼭대기와 바닥이 반드시 있어야 한다. 꼭대기(top)는 모든 타입의 상위타입, 즉 전체집합이고, 바닥(bottom)은 모든 타입의 하위타입, 즉 공집합이다. 이것이 없으면 "공통 상위타입이 아예 없는 두 타입" 같은 빈틈이 생겨 타입 추론이 막힌다. Kotlin은 이 두 끝을 언어에 명시적으로 정의했다. 꼭대기는 `Any?`이고, 바닥은 `Nothing`이다.

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

이 그림에서 바로 눈에 띄는 비대칭이 하나 있다. 바닥은 `Nothing` 하나로 깔끔한데, 꼭대기는 `Any`가 아니라 `Any?`이다. `Any`가 꼭대기가 아닌 이유는 `null`이라는 값이 `Any`에 속하지 않기 때문이다. `Any`는 "널이 아닌 모든 값"의 집합이고, `null`까지 포함한 진짜 전체집합은 `Any?`이다. 이 한 단계의 차이가 Kotlin 타입 시스템의 핵심인 널 안전성을 구조적으로 나타낸다. 2절과 8절에서 이 비대칭을 자세히 살펴본다.

> [!note] 명세 기준
> "격자"라는 말은 편의상 쓰는 비유이다. 널 가능성과 플랫폼 타입(유연한 타입, flexible type)까지 포함하면 Kotlin의 부분타입 관계는 순수한 격자보다 복잡한 준순서(preorder)에 가깝다. 하지만 실무에서 중요한 것은 "꼭대기 `Any?`, 바닥 `Nothing`, 그리고 둘을 잇는 부분타입 사슬"이라는 기본 구조이며, 이 구조는 정확하다.

### 1.3 정적 타입과 강타입: 지형은 컴파일 타임에 쓰인다

이 지형이 언제 쓰이는지부터 정리하자. Kotlin은 **정적 타입**(statically typed) 언어이다. 모든 식(expression)은 컴파일 시점에 하나의 정적 타입을 가지며, 부분타입 관계 검사는 대부분 컴파일 타임에 끝난다. `val a: Animal = Dog(...)`가 허용되는지는 프로그램을 실행하기 전에 결정된다. 6장(널 안전성)에서 다루는 널 안전성이 "런타임 검사가 아니라 컴파일 타임 검사"인 이유도 여기에 있다.

동시에 Kotlin은 **강타입**(strongly typed) 언어이다. 타입 사이에 암시적으로 조용히 일어나는 변환이 거의 없다. Java나 C에서 당연하게 여기는 `int`→`long` 자동 확대조차 Kotlin에는 없어서 `toLong()`을 명시해야 한다(7장 수 1).

이런 강타입 규칙 때문에 지형에서 위로 올라가는 것(상위타입으로 보는 것)은 자동으로 이루어지지만, 아래로 내려가는 것(하위타입으로 좁히는 것)은 반드시 명시적 캐스트(`as`, `is`, 스마트 캐스트)가 필요하다. 개를 동물로 보는 데에는 아무 비용이 들지 않지만, 그 반대는 그렇지 않다. 지형의 위아래를 오가는 규칙이 곧 타입 검사의 규칙이다.

---

## 2. `Any`: 꼭대기 바로 아래, 널이 아닌 모든 값의 뿌리

### 2.1 모든 클래스의 암시적 슈퍼클래스

슈퍼타입을 명시하지 않은 Kotlin 클래스는 모두 `Any`를 상속한다. `class Coordinate(val x: Int, val y: Int)`라고만 써도 이 클래스는 암묵적으로 `Any`의 하위타입이 된다. `: Any()`라고 명시할 수도 있지만 아무 의미 없는 중복이다.

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

핵심은 마지막 줄이다. `Any`는 "널이 아닌 모든 값"의 집합이지 "모든 값"의 집합이 아니다. `null`은 `Any`의 값이 아니다. 2.4절과 8절에서 `Any`와 `Any?`를 구분하는 근거가 바로 이 사실이다.

`Any`가 (널이 아닌 쪽) 모든 타입의 뿌리라는 것은 실용적으로 두 가지를 뜻한다. 첫째, 어떤 값이든 담을 수 있는 이질적인 컨테이너의 원소 타입으로 `Any`를 쓸 수 있다(단, 타입 안전성은 포기해야 한다). 둘째, 모든 값에서 `Any`가 선언한 세 멤버인 `equals`, `hashCode`, `toString`을 호출할 수 있다. 정수든 람다든 좌표든 `.toString()`을 호출할 수 있는 것은 모두 `Any`의 하위타입이기 때문이다.

### 2.2 `Any`가 선언하는 멤버는 세 개뿐이다

`Any`의 정의는 놀랄 만큼 작다. 표준 라이브러리에는 `Any`가 대략 다음과 같이 선언되어 있다.

```kotlin
public open class Any {
    public open operator fun equals(other: Any?): Boolean
    public open fun hashCode(): Int
    public open fun toString(): String
}
```

멤버는 `equals`, `hashCode`, `toString` 세 개뿐이다. 이 세 계약(contract)의 정확한 의미, 즉 `==`가 왜 `equals` 호출로 바뀌는지, `hashCode`와 `equals`가 지켜야 하는 일관성 계약, 참조 동일성 `===`와의 차이는 [[10 - 불리언과 동등성과 동일성]]에서 다룬다.

여기서 중요한 것은 이 세 멤버가 *모든 값이 공통으로 갖는 기능*이라는 점이다. 어떤 값이든 문자열로 바꿀 수 있고, 다른 값과 같은지 비교할 수 있고, 해시 코드를 구할 수 있다.

`equals`의 파라미터 타입이 `Any?`(널 허용)라는 점에 주목하자. `x == null` 같은 비교가 타입 오류 없이 성립해야 하므로 `equals`에 넘기는 상대 값은 널일 수 있어야 한다. 반면 `Any` 자체는 `null`을 값으로 가질 수 없다. 타입으로서의 `Any`와 파라미터 타입 `Any?`의 이 차이가 바로 널 가능성을 표현하는 문법이다.

### 2.3 `Any`는 `java.lang.Object`가 아니다: JVM 매핑의 실제

가장 흔한 오해 중 하나는 "Kotlin의 `Any`는 Java `Object`의 다른 이름일 뿐"이라는 생각이다. JVM 백엔드에서 `Any`가 `java.lang.Object`로 매핑되는 것은 사실이다. 그래서 Kotlin과 Java를 함께 쓸 때 `Any`와 `Object`가 서로 호환된다. 하지만 *언어 수준에서* `Any`는 `Object`와 같지 않다. 이유는 두 가지다.

첫째, `Object`에는 `wait()`, `notify()`, `notifyAll()`, `getClass()` 같은 메서드가 있지만 **`Any`에는 없다**. Kotlin은 이 메서드들을 `Any`의 멤버에서 의도적으로 뺐다.

`wait/notify`는 오래되고 실수하기 쉬운 저수준 스레드 동기화 API이므로, 코루틴과 고수준 동시성([[43 - 동시성과 메모리 모델]])을 지향하는 Kotlin은 이를 기본으로 노출하지 않기로 했다. `getClass()`는 Kotlin 리플렉션의 진입점인 `::class`([[44 - 애노테이션과 리플렉션]])로 대체되었다.

```kotlin
val c = Coordinate(1, 2)
// c.wait()         // 컴파일 에러: unresolved reference — Any에는 wait가 없다
// c.getClass()     // 컴파일 에러: unresolved reference — 대신 c::class 를 쓴다

// JVM 백엔드에서 정말 Object의 메서드가 필요하면 명시적으로 Object로 캐스팅
(c as java.lang.Object).wait()   // JVM 한정. 이제 wait 가시
```

둘째, 방향은 반대이지만 `Any`는 `Object`보다 *더* 넓다. Kotlin에서는 `Int`, `Boolean` 같은 값도 `Any`의 하위타입이다. Java에서 `int`는 `Object`의 하위타입이 아니지만(원시 타입은 클래스 계층 밖에 있다), Kotlin에서는 `42`를 `Any`로 다룰 수 있다. 다만 JVM 백엔드에서는 이때 박싱이 일어난다.

이 부분은 "원시 타입이 없다"는 오해, "`Int?`는 항상 박싱된다"는 오해와 이어지며, JVM 수준에서 실제로 어떻게 동작하는지는 8장(수 2)에서 다룬다. 여기서는 "언어의 지형에서는 모든 값이 `Any` 아래에 있지만, 그렇다고 원시 타입이 사라진 것은 아니다"라는 점만 기억하면 된다.

> [!warning] 흔한 오해
> "`Any`는 `Object`의 alias다"라는 생각은 틀렸다. JVM 백엔드의 *런타임 매핑*은 그렇지만, *컴파일 타임 타입*으로서 `Any`는 `wait/notify/getClass`가 없고 원시 값도 하위에 두는 별개의 타입이다. 또한 Native/JS/Wasm 백엔드에는 `java.lang.Object` 자체가 없다. `Any`는 언어의 개념이고, `Object`는 JVM 플랫폼의 구현이다.

### 2.4 `Any`와 `Any?`: 한 단계의 차이가 전부다

`Any`는 지형의 꼭대기가 아니다. `Any` 위에 `Any?`가 있다. `Any?`는 `Any`의 모든 값에 `null` 하나를 더한 집합이다. 집합으로 쓰면 다음과 같다.

$$\text{Any?} = \text{Any} \cup \{\texttt{null}\}$$

`Any?`가 진짜 꼭대기(모든 타입의 상위타입)인 이유는 어떤 타입 `T`의 값이든 `Any`에 속하거나 `null`이거나 둘 중 하나이기 때문이다. 두 경우 모두 `Any?`에 속한다. `String?` 같은 널 허용 타입의 값(`null` 포함)도 `Any?`에 속한다.

```kotlin
val anything: Any? = null          // => OK — Any?는 null을 담는다
val everything: Any? = "cat"       // => OK
val stillOk: Any? = Coordinate(1,2)// => OK

// val notNull: Any = null         // 컴파일 에러: Any는 null을 담지 못한다

fun <T> asAnything(value: T): Any? = value   // 어떤 T든 Any?로 승격 가능 (T <: Any?)
```

마지막 함수를 눈여겨보자. 임의의 타입 파라미터 `T`의 값을 아무 제약 없이 `Any?`로 받을 수 있다. 이는 **타입 파라미터의 기본 상한(upper bound)이 `Any?`** 라는 사실을 보여 준다.

`fun <T>`라고만 쓰면 `T`는 널 허용까지 포함하는 `T : Any?`를 뜻한다. 널을 제외하려면 `fun <T : Any>`로 상한을 낮춰야 한다. 이 규칙 때문에 제네릭에 `null`이 예상치 못하게 들어오는 함정이 생기는데, 자세한 내용은 [[33 - 제네릭 1 - 타입 파라미터와 변성]]에서 다룬다.

정리하면 지형의 꼭대기 부근은 다음과 같다.

```text
   Any?   ← 진짜 정상. 모든 값 + null. 타입 파라미터의 기본 상한.
    │
    ├── Any           ← 널 아닌 모든 값. Object로 매핑되나 wait/getClass 없음.
    │
    └── Nothing?      ← null 하나만. (5절에서 자세히)
```

`Any?`와 `Any` 사이의 이 한 칸이 Kotlin 널 안전성의 출발점이다. "`?`를 붙인다"는 것은 지형에서 정확히 한 층 위로, 즉 `null`을 원소로 추가한 더 큰 집합으로 올라가는 연산이다.

---

## 3. `Unit`: 원소가 하나뿐인 타입

### 3.1 "아무것도 반환하지 않는" 함수가 반환하는 것

의미 있는 값을 돌려주지 않는 함수를 생각해 보자. 좌표를 콘솔에 출력하기만 하는 함수가 그런 예이다.

```kotlin
fun printCoordinate(c: Coordinate) {
    println("(${c.x}, ${c.y})")
}
```

반환 타입을 적지 않았다. 많은 언어에서는 이런 함수를 "값을 반환하지 않는다(void)"고 설명한다. 그러나 Kotlin에서 이 함수의 반환 타입은 `void`가 아니라 **`Unit`** 이다. 위 선언은 다음 선언과 완전히 같다.

```kotlin
fun printCoordinate(c: Coordinate): Unit {   // : Unit 은 생략된 것뿐
    println("(${c.x}, ${c.y})")
    return Unit                               // 이 return도 암묵적으로 존재
}
```

`Unit`은 타입의 이름이면서 동시에 값이다. 표준 라이브러리에는 `Unit`이 다음과 같이 선언되어 있다.

```kotlin
public object Unit {
    override fun toString(): String = "kotlin.Unit"
}
```

즉 `Unit`은 `object` 선언, 곧 싱글턴([[28 - object와 companion - 싱글턴과 동반 객체]])이다. 프로그램 전체에서 `Unit` 타입의 값은 그 싱글턴 인스턴스 하나뿐이다. 그래서 `Unit`은 **원소가 하나뿐인 집합**에 대응하는 타입이다.

집합으로 쓰면 $\text{Unit} = \{\,\texttt{Unit}\,\}$, 즉 크기가 1인 집합이다. 타입 이론에서는 이런 타입을 단위 타입(unit type)이라고 부르며, 관련 설명은 [[10 - 곱·합·단위·공집합]]에 있다.

값을 반환하지 않는 함수가 실제로는 "정보가 0비트인 값" 하나를 반환한다는 발상은 처음에는 억지처럼 들린다. 하지만 다음 절들에서 이것이 `void`보다 훨씬 규칙적이고 유용한 설계라는 것을 확인하게 된다.

### 3.2 원소가 하나면 정보량이 0이다

`Unit` 값이 담는 정보량을 정보 이론으로 계산해 보자. 가능한 값이 $n$개인 타입이 담을 수 있는 정보량은 $\log_2 n$ 비트이다. `Boolean`은 값이 2개이므로 $\log_2 2 = 1$비트이고, `Int`는 약 43억 개이므로 $\log_2 2^{32} = 32$비트이다. 그렇다면 `Unit`은 어떨까?

$$\log_2 |\text{Unit}| = \log_2 1 = 0 \text{ 비트}$$

`Unit` 값이 담는 정보는 0비트이다. 이 계산이 "값을 반환하지 않는다"는 직관과 "값을 반환하지만 그 값에는 정보가 없다"는 Kotlin의 형식화를 연결한다. 실용적으로는 같은 이야기이다. 어차피 확인할 내용이 없는 값이기 때문이다.

하지만 형식적으로 "값이 있다"고 정해 두면 그 값을 다른 값과 똑같이 다룰 수 있다. 변수에 담을 수 있고, 제네릭 타입 인자로 넘길 수 있고, 함수 타입의 결과 타입으로 쓸 수 있다. `void`로는 이렇게 다룰 수 없다. 이 차이가 4절의 핵심이다.

```kotlin
val result: Unit = printCoordinate(Coordinate(1, 2))   // => Unit 값을 변수에 담을 수 있다
println(result)                                         // => kotlin.Unit

val u1 = Unit
val u2 = printCoordinate(Coordinate(3, 4))
println(u1 === u2)   // => true — Unit 값은 유일한 싱글턴이므로 항상 동일
```

### 3.3 람다와 `Unit` 강제 변환

`Unit`이 값이라는 사실 덕분에 람다에서 미묘하지만 편리한 규칙이 생긴다. 함수 타입 `() -> Unit`을 기대하는 자리에 람다를 넘길 때, 람다의 마지막 식이 `Unit`이 아니어도 컴파일러가 자동으로 `Unit`으로 맞춰 준다. 이를 **Unit 강제 변환**(Unit coercion)이라고 한다.

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

이 규칙은 [[13 - 함수 타입과 람다와 함수 참조]]에서 다시 나온다. 여기서 중요한 점은 `Unit`을 "값이 있는 타입"으로 정의했기 때문에 이런 자연스러운 변환 규칙이 성립한다는 것이다. `void`였다면 "값이 없는 것을 값이 있는 것으로 변환한다"는 모순된 개념을 다뤄야 했을 것이다.

---

## 4. `Unit`은 `void`가 아니다

### 4.1 오해 살펴보기

첫 번째 오해부터 바로잡자. 흔히 **`Unit`은 Java나 C의 `void`를 Kotlin식으로 부르는 이름일 뿐**이라고 생각한다. 둘 다 "쓸모 있는 반환값이 없는 함수"에 등장하므로 겉으로는 그럴듯하다. 하지만 타입 시스템에서 `void`와 `Unit`이 차지하는 위치는 근본적으로 다르다.

Java와 C의 `void`는 **타입이 아니거나, 값이 없다는 특수한 표시**이다. Java에서 `void`는 "이 메서드는 값을 반환하지 않는다"는 것을 메서드 시그니처에 표시할 뿐이며, 값으로 다룰 수 없다. `void` 타입의 변수를 만들 수 없고, `List<void>` 같은 제네릭 인자로 쓸 수 없고, `void` 값을 다른 함수에 넘길 수도 없다.

Java에는 따로 `java.lang.Void`라는 클래스가 있지만, 인스턴스를 만들 수 없으므로 유일한 "값"은 `null`이다. 즉 `Void` 타입 변수에는 `null`밖에 담을 수 없는 불완전한 타입이다.

Kotlin의 `Unit`은 **원소가 하나인 진짜 타입**이며, 그 유일한 값은 실제로 존재하는 싱글턴 객체이다. 변수에 담을 수 있고(`val x: Unit`), 제네릭 인자로 쓸 수 있고(`List<Unit>`), 함수에 넘길 수 있다. `Void`처럼 `null`로 흉내 내는 것이 아니라, `Unit`이라는 실제 값이 존재한다.

| 성질 | `void` (Java/C) | `java.lang.Void` | `Unit` (Kotlin) |
|------|-----------------|------------------|-----------------|
| 타입인가 | 표식(값 아님) | 타입(클래스) | 타입(object) |
| 값의 개수 | 없음 | `null` 하나뿐(인스턴스화 불가) | 싱글턴 하나 |
| 변수에 담기 | 불가 | 가능하나 `null`만 | 가능(`Unit`) |
| 제네릭 인자 | 불가 | 가능하나 `null`만 | 가능(진짜 값) |
| 정보량 | - | 0비트(그러나 null 오염) | 0비트(깔끔) |

### 4.2 이 구분이 실제로 중요한 이유: 제네릭의 균일성

이 구분이 말장난이 아니라는 것은 제네릭에서 드러난다. "결과를 하나 돌려주는 작업"을 추상화한 제네릭 타입 `Task<R>`를 생각해 보자.

```kotlin
class Task<R>(private val body: () -> R) {
    fun run(): R = body()
}

val computeArea: Task<Double> = Task { 3.14 * 2 * 2 }   // 결과 Double
val logMessage: Task<Unit>    = Task { println("done") } // 결과 Unit — 아무 특례 없이 성립!

val area: Double = computeArea.run()   // => 12.56
val nothing: Unit = logMessage.run()   // => Unit. 균일하게 동작
```

핵심은 `Task<Unit>`이 아무 예외 규칙 없이 성립한다는 점이다. "값을 반환하는 작업"과 "값을 반환하지 않는 작업"을 *같은 제네릭 틀*로 다룰 수 있다. 만약 Kotlin이 `void`를 썼다면 `Task<void>`는 허용되지 않으므로, "결과 없는 작업"을 위해 `Runnable` 같은 별도의 타입을 만들어야 했을 것이다.

Java가 실제로 그렇게 한다. 값을 돌려주는 `Callable<V>`와 돌려주지 않는 `Runnable`이 별개의 인터페이스로 나뉘어 있고, `Future<Void>`처럼 `Void`와 `null`을 조합해 어색하게 빈틈을 메운다.

Kotlin은 `Unit`을 값이 있는 타입으로 만들어 이런 분리를 없앴다. 함수 타입도 마찬가지다. `() -> Unit`은 특별한 "프로시저 타입"이 아니라 결과 타입이 `Unit`인 평범한 함수 타입이다. 이 균일성이 13장(함수 타입과 람다)과 [[16 - 스코프 함수와 수신 객체 관용구]], 나아가 코루틴([[42 - suspend와 코루틴 - 언어 수준의 중단]])의 설계까지 깔끔하게 뒷받침한다.

### 4.3 JVM 바이트코드에서는 결국 `void`가 된다: 그러나 그것은 구현이다

여기서 짚고 넘어갈 점이 있다. **JVM 백엔드에서는** `Unit`을 반환하는 Kotlin 함수가 바이트코드 수준에서 반환 타입이 `void`인 메서드로 컴파일된다. 호출할 때마다 `Unit.INSTANCE`를 실제로 스택에 올려 반환하는 것은 낭비이므로, 컴파일러가 이를 최적화해 `void`로 만든다.

```text
Kotlin 소스:                  JVM 바이트코드(개념):
fun log(): Unit { ... }   →   public final void log() { ... }   // 반환 타입 void
```

그렇다면 "`Unit`은 `void`가 아니다"라는 이 절의 주장과 모순되지 않는가? 그렇지 않다. 핵심은 **추상화 계층을 구분하는 것**이다.

*언어의 타입 시스템*에서 `Unit`은 값이 있는 타입이고, 그래서 제네릭, 함수 타입, 변수 대입에서 일급 값으로 동작한다. *JVM 백엔드의 코드 생성*에서는 이 값의 정보가 0비트라는 점을 이용해 `void`로 줄이는 최적화를 한다. 값이 실제로 필요한 자리, 예를 들어 `Function0<Unit>`의 결과나 `Task<Unit>`의 타입 인자에서는 백엔드가 `Unit.INSTANCE`를 실제로 사용한다.

이런 이중성은 백엔드마다 또 다르다. Kotlin/Native나 Kotlin/JS에는 JVM의 `void`가 없으므로 `Unit`은 각 플랫폼에 맞는 방식으로 표현된다. 따라서 "Unit은 void로 컴파일된다"는 설명은 **JVM 백엔드에 한정된 구현 사실**이지 언어의 정의가 아니다. 언어의 정의는 "Unit은 값이 하나인 타입"이며, 이 정의는 모든 타깃에서 성립한다.

> [!caution] 성능 주의
> `Unit`을 반환하는 함수는 JVM에서 `void`로 컴파일되므로, 단순한 프로시저 호출에서는 `Unit` 때문에 생기는 런타임 비용이 사실상 없다. 다만 `List<Unit>`이나 `() -> Unit` 람다처럼 **박싱된 값으로** 실제로 저장하거나 전달하는 자리에서는 `Unit.INSTANCE`가 오간다. 이 인스턴스는 싱글턴이라 새로 할당되지는 않지만, "언어상 값이 있다"는 사실이 완전히 공짜는 아니라는 점을 기억하자.

> [!info] 역사 메모
> 값이 하나뿐인 단위 타입은 Kotlin이 처음 만든 개념이 아니다. ML 계열 언어의 `unit`(값 `()`), Scala의 `Unit`(값 `()`), Haskell의 `()`(유닛)이 모두 같은 개념이다. Kotlin은 이 함수형 전통에서 `Unit`을 가져와, 명령형 전통의 불완전한 `void`를 대체했다. 이름을 `Unit`으로 정한 것 자체가 "이것은 진짜 타입"이라는 뜻을 드러낸다.

---

## 5. `Nothing`: 값이 하나도 없는 바닥

### 5.1 공집합에 대응하는 타입

이제 지형의 반대쪽 끝인 바닥으로 내려가자. `Unit`이 원소가 하나인 집합이라면, `Nothing`은 **원소가 하나도 없는 공집합**에 대응하는 타입이다. 표준 라이브러리에는 `Nothing`이 다음과 같이 선언되어 있다.

```kotlin
public class Nothing private constructor()
```

생성자가 `private`이고, 표준 라이브러리 어디에서도 이 생성자를 호출하지 않는다. 따라서 `Nothing`의 인스턴스는 **결코 존재할 수 없다**. `Nothing` 타입의 값을 만드는 방법은 없다. 집합으로 쓰면 $\text{Nothing} = \varnothing$, 즉 공집합이다.

여기서 "값을 절대 만들 수 없는 타입이 무슨 쓸모가 있는가?"라는 의문이 자연스럽게 생긴다. 값이 없어서 변수에 담을 수도, 함수에 넘길 수도 없다면 쓸모없는 타입이 아닌가? 그렇지 않다. `Nothing`의 쓸모는 값을 *담는* 데 있지 않고, **부분타입 관계에서 차지하는 위치**와 **"이 지점에는 값이 도달하지 않는다"는 정보**를 컴파일러에 전달하는 데 있다. 6~7절에서 이를 자세히 살펴보고, 여기서는 먼저 그 위치부터 확인한다.

### 5.2 `Nothing`은 모든 타입의 하위타입이다

`Nothing`을 정의하는 성질은 다음과 같다. **`Nothing`은 모든 타입의 하위타입이다.** 임의의 타입 `T`에 대해 $\text{Nothing} <: T$가 성립한다. `Nothing <: Int`, `Nothing <: String`, `Nothing <: Coordinate`, `Nothing <: Any?` 등 예외 없이 전부 성립한다.

이것이 자연스러운 이유는 부분타입을 부분집합으로 보면 바로 알 수 있다. 공집합은 모든 집합의 부분집합이다. $\varnothing \subseteq S$는 모든 $S$에 대해 참이다(공집합의 원소가 모두 $S$에 속한다는 명제는 원소가 없으므로 공허하게 참이다). `Nothing`은 공집합에 대응하므로 모든 타입의 부분집합, 즉 하위타입이다.

이 성질을 코드로 확인해 보자. `Nothing` 타입의 값은 값을 *만들어서* 얻는 것이 아니라, 정상적으로 반환되지 않는 식인 `throw`에서 나온다(6절). `throw` 식의 타입은 `Nothing`이다. 그리고 `Nothing`은 모든 타입의 하위타입이므로, 어떤 타입을 기대하는 자리에든 `throw`를 놓을 수 있다.

```kotlin
fun bankBalance(accountId: String): Int {
    val account = findAccount(accountId)
        ?: throw NoSuchElementException("계좌 없음")   // throw는 Nothing 타입
        // Nothing <: Int 이므로, Int가 필요한 엘비스 우변에 throw를 놓을 수 있다
    return account.balance
}
```

엘비스 연산자 `?:`는 좌변 `findAccount(...)`가 널이면 우변을 평가한다. 우변 `throw ...`의 타입은 `Nothing`이므로 좌변의 비-널 타입 `Account`와 합쳐진다. `Nothing`이 모든 타입 아래에 있기 때문에 이렇게 합치는 것은 항상 가능하다. 이 동작의 널 처리 측면은 6장(널 안전성)에서 다루지만, 그 바탕이 되는 타입 시스템의 기본 구조, 즉 `Nothing`이 바닥이라는 사실은 이 장에서 설명한다.

### 5.3 지형의 완성: 꼭대기와 바닥이 모두 채워졌다

이제 지형이 완성되었다. 꼭대기에 `Any?`, 바닥에 `Nothing`이 있고, 나머지 모든 타입이 그 사이에 층층이 쌓인다.

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

부분타입 사슬을 하나 골라 보면 다음 관계가 모두 성립한다.

$$\text{Nothing} <: \text{Int} <: \text{Any} <: \text{Any?}$$
$$\text{Nothing} <: \text{Nothing?} <: \text{Int?} <: \text{Any?}$$

바닥의 `Nothing`은 `Int`뿐 아니라 `Nothing?`의 하위타입이기도 하다($T <: T?$는 항상 참이기 때문이다). 꼭대기의 `Any?`는 `Any`와 `Nothing?`을 모두 위에서 덮는다. 이 두 끝이 격자를 "닫아" 주기 때문에 임의의 두 타입에 대해 LUB와 GLB를 계산하면 언제나 답이 나온다.

`Nothing`이 없다면 "아무 값도 도달하지 않는 분기"의 타입을 표현할 수 없으므로 타입 추론에 빈틈이 생긴다. 지형에 바닥이 필요한 이유가 바로 이것이다. 이런 바닥 타입(bottom type)의 타입 이론적 배경은 [[16 - 부분 타입과 변성]]과 Type Theory 시리즈 10장(곱·합·단위·공집합, 여기서는 공집합/void 타입이라고 부른다)에서 다룬다.

---

## 6. `Nothing`은 `null`이 아니다

### 6.1 정반대에 있는 두 개념

두 번째 오해는 **`Nothing`이 `null`이나 "빈 값"과 비슷한 무의미한 것**이라는 생각이다. 이 오해는 대개 이름 때문에 생긴다. "Nothing은 아무것도 없다는 뜻이니 null이 아닌가?"라고 생각하기 쉽다. 하지만 `Nothing`과 `null`은 지형에서 *정반대의 끝*에 있다.

`null`은 **값**이다. `Nothing?` 타입의 유일한 값이며, 널 허용 타입이 담을 수 있는 특별한 원소이다. `null`은 실제로 존재하고, 변수에 담기고, 전달되고, 비교된다.

`Nothing`은 **값이 없는 타입**이다. 인스턴스가 하나도 없는 공집합이다. `Nothing` 타입의 변수에는 아무것도 담을 수 없으며, `null`조차 담을 수 없다.

```kotlin
val a: Int? = null          // => OK — null은 Nothing? 타입의 값, Int?에 담긴다
// val b: Nothing = null    // 컴파일 에러: null은 Nothing?이지 Nothing이 아니다
// val c: Nothing = TODO()  // 대입 자체는 타입상 OK지만, TODO()가 예외를 던져 c에 결코 도달 못함
```

집합으로 비교하면 분명해진다. `null`이 속한 타입 `Nothing?`은 원소가 하나($\{\texttt{null}\}$)인 집합이고, `Nothing`은 원소가 없는($\varnothing$) 집합이다. `Nothing?`은 `Nothing`에 `null` 하나를 더한 것이므로, 지형에서 `Nothing` 바로 한 칸 위에 있다.

$$\text{Nothing?} = \text{Nothing} \cup \{\texttt{null}\} = \varnothing \cup \{\texttt{null}\} = \{\texttt{null}\}$$

그래서 `null` 리터럴 자체의 타입은 정확히 `Nothing?`이다. `val x = null`이라고만 쓰면 `x`의 추론 타입은 `Nothing?`이 된다.

```kotlin
val x = null           // x의 추론 타입: Nothing?
// x는 null 말고는 아무것도 담을 수 없다 — Nothing?의 유일한 값이 null이므로
```

### 6.2 `null`은 값의 부재를, `Nothing`은 계산의 부재를 나타낸다

두 개념의 역할을 한 문장으로 비교하면 다음과 같다. **`null`은 "값이 있어야 할 자리에 값이 없다"는 상태를 표현하는 값이고, `Nothing`은 "이 코드 지점에는 값이 애초에 도달하지 않는다"는 제어 흐름을 표현하는 타입이다.** 앞의 것은 데이터의 부재이고, 뒤의 것은 계산의 부재이다.

이 차이가 가장 잘 드러나는 곳이 반환 타입이다. `Nothing`을 반환한다고 선언한 함수는 **정상적으로 반환하는 일이 결코 없는** 함수이다. `Nothing` 값을 만들어 반환할 방법이 없으므로, 그 함수가 `return`으로 끝나는 것은 불가능하기 때문이다. `Nothing`을 반환하는 함수가 제어를 벗어나는 방법은 예외를 던지거나 영원히 끝나지 않는 것, 이 두 가지뿐이다.

```kotlin
fun fail(message: String): Nothing {
    throw IllegalStateException(message)   // 정상 반환 없음 — 반드시 예외로 이탈
}

fun loopForever(): Nothing {
    while (true) { /* ... */ }             // 정상 반환 없음 — 영원히 돌거나 예외
}

// fun broken(): Nothing { return }        // 컴파일 에러: Nothing 함수는 정상 반환 불가
```

"정상적으로 반환하지 않는다"는 이 정보가 `Nothing`의 핵심이다. 컴파일러는 `Nothing`을 반환하는 함수를 호출한 *뒤의* 코드를 "도달 불가능(unreachable)"으로 판정할 수 있다. 7절에서 이 도달 불가능 분석이 널 스마트 캐스트, 도달 불가 경고, 완전성 검사로 이어지는 과정을 살펴본다.

### 6.3 `Nothing?`: 널만 들어 있는 얇은 층

`Nothing?`은 실무에서 직접 쓸 일이 거의 없지만, 지형 전체가 일관되게 맞아떨어진다는 것을 이해하려면 알아 둘 만하다. `Nothing?`의 값은 `null` 하나뿐이므로, `Nothing?` 타입의 파라미터를 받는 함수는 사실상 `null`만 받는다. 이 성질은 "오직 널만 허용한다"는 것을 타입으로 표현하고 싶을 때처럼 드물게 유용하다. 하지만 대개는 `null` 리터럴의 타입으로 눈에 띄지 않게 등장할 뿐이다.

`Nothing?`이 `Any`의 하위타입이 *아니라는* 점도 지형의 비대칭을 다시 보여 준다. `Nothing? = {null}`인데 `null`은 `Any`의 값이 아니므로($\{\texttt{null}\} \not\subseteq \text{Any}$), `Nothing?`은 `Any`의 하위타입이 될 수 없다. `Nothing?`은 `Any?`의 하위타입이지만 `Any`는 거치지 않는다. 이처럼 지형에서 널 층은 비-널 층을 우회해 꼭대기 `Any?`로 바로 연결된다.

```text
   Any?
   /  \
 Any   \
  │      Nothing?      ← Any를 거치지 않고 Any?로 직결 (null은 Any의 값이 아니므로)
  │       │
Nothing ─┘             ← Nothing은 Any의 하위타입이자 Nothing?의 하위타입
```

---

## 7. `Nothing`의 실제 역할: 제어 흐름과 타입 추론

### 7.1 `throw`·`return`·`break`·`continue`는 모두 `Nothing` 타입이다

Kotlin은 표현식 지향 언어이다([[17 - 표현식으로서의 제어 흐름 - if와 when]]). 문(statement)처럼 보이는 제어 구조 중 상당수가 실제로는 값을 갖는 식(expression)이며, 그 값의 타입이 `Nothing`인 경우가 많다. 구체적으로 **`throw`, `return`, `break`, `continue`는 모두 `Nothing` 타입의 식**이다.

이 식들의 공통점은 "평가되는 순간 현재 지점의 정상적인 진행을 끝낸다"는 것이다. `throw`는 예외로 빠져나가고, `return`은 함수를 벗어나고, `break`/`continue`는 루프의 흐름을 바꾼다. 어느 것도 "값을 만들어 다음 계산으로 넘기는" 일을 하지 않는다. 다음으로 넘길 값이 없으므로 정확히 공집합, 즉 `Nothing`이다.

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

`throw` 분기의 타입이 `Nothing`이므로, `when`의 결과 타입을 계산할 때 이 분기는 다른 분기의 타입에 영향을 주지 않는다. `Nothing`은 모든 타입 아래에 있어서 LUB 계산에 흡수되기 때문이다. 그래서 나머지 분기가 모두 `String`이면 결과 타입은 `Nothing`이 아니라 `String`이다.

### 7.2 최소 상계(LUB)에서 `Nothing`은 항등원처럼 흡수된다

7.1의 계산을 일반화해 보자. `if/else`나 `when`에 분기가 여러 개 있으면, 전체 식의 타입은 분기 타입들의 최소 상계(least upper bound)이다. 여기서 `Nothing`은 특별한 역할을 한다. **어떤 타입 `T`와 `Nothing`의 LUB는 언제나 `T`** 이다.

$$\text{LUB}(T,\ \text{Nothing}) = T \quad (\text{모든 } T\text{에 대해})$$

$\text{Nothing} <: T$이므로 `Nothing`은 이미 `T` 아래에 있고, 둘을 모두 덮는 가장 작은 타입은 `T` 자신이기 때문이다. 집합으로 쓰면 $S \cup \varnothing = S$이다. 공집합과의 합집합은 자기 자신이다. 그래서 `Nothing`은 LUB 연산의 항등원(identity)처럼 동작한다. 분기 하나가 `throw`(=`Nothing`)라면 그 분기는 결과 타입에 영향을 주지 않고 흡수된다.

```kotlin
val radius: Double = if (input > 0) input.toDouble()
                     else throw IllegalArgumentException("반지름은 양수")
// LUB(Double, Nothing) = Double → radius는 Double (Double? 아님!)

val name: String = person.nickname ?: return   // return은 Nothing
// LUB(String, Nothing) = String → 엘비스 우변이 함수를 벗어나므로 name은 비-널 String
```

특히 두 번째 예가 중요하다. `person.nickname`이 `String?`(널 허용)일 때 `?: return`으로 널인 경우를 함수 탈출로 처리하면, 그 뒤로 `name`은 비-널 `String`으로 확정된다. `return`의 타입이 `Nothing`이라 엘비스 결과에서 널 가능성이 제거되기 때문이다.

`?: return`, `?: throw` 관용구의 널 안전성 측면은 6장(널 안전성)에서 다루지만, 이 관용구를 가능하게 하는 원리, 즉 `Nothing`이 LUB에 흡수된다는 사실은 이 장에서 설명하는 타입 지형에서 나온다.

### 7.3 도달 불가능 분석과 `TODO()`

`Nothing`이 컴파일러에 주는 가장 중요한 정보는 "이 지점 이후는 도달할 수 없다"는 것이다. `Nothing`을 반환하는 함수를 호출한 다음 줄부터는 실행이 절대 이르지 않으므로, 컴파일러는 그 코드를 도달 불가로 표시하고 이후의 타입 검사를 그에 맞게 조정한다.

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

`fail`이 `Nothing`을 반환하므로 `when`의 `else` 분기 이후로는 값이 넘어가지 않는다. 그래서 `rate`는 `Double`(널이 아니고 `Any`도 아니다)로 깔끔하게 추론된다. 만약 `fail`의 반환 타입이 `Unit`이었다면 `else` 분기가 `Unit`을 내놓아 `rate`의 타입이 `Any`로 넓어졌을 것이다. `Nothing`과 `Unit`의 이 차이가 실무에서 타입 추론의 품질을 좌우한다.

표준 라이브러리는 이 원리를 활용하는 도우미 함수를 여러 개 제공한다. 모두 반환 타입이 `Nothing`이다.

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

`TODO("삼각형 넓이 미구현")`을 `Double`이 필요한 자리에 놓을 수 있는 것은 `Nothing <: Double`이기 때문이다. 그리고 그 자리를 채워도 함수 전체가 여전히 `Double`을 반환한다고 컴파일러가 판단할 수 있는 것은 `Nothing`이 LUB에 흡수되기 때문이다.

아직 구현하지 않은 부분을 타입 검사를 통과시키면서 남겨 두는 이 관용구가 `Nothing`을 일상에서 가장 흔하게 쓰는 방법이다. 예외 계층, 검사 예외가 없다는 점, `runCatching`/`Result`처럼 예외와 `Nothing`이 더 깊이 연결되는 부분은 41장(예외와 Nothing)에서 다룬다.

### 7.4 공변 위치에서의 `Nothing`: 빈 컬렉션의 원소 타입

`Nothing`이 바닥이라는 성질은 제네릭 변성(33장 제네릭 1)과 만나 매우 실용적인 결과를 낳는다. 대표적인 예가 빈 컬렉션이다. 원소가 하나도 없는 리스트의 원소 타입은 무엇이어야 할까? 빈 `List<String>`으로도, 빈 `List<Coordinate>`로도 쓸 수 있어야 하므로 "무엇이든 될 수 있어야" 한다. 이 요구를 정확히 만족하는 원소 타입이 `Nothing`이다.

`List`는 원소 타입에 대해 공변(covariant, `out`)이다. 즉 `A <: B`이면 `List<A> <: List<B>`이다([[38 - 컬렉션 - 읽기 전용과 가변]]). `Nothing`은 모든 타입의 하위타입이므로 `List<Nothing>`은 *모든* `List<T>`의 하위타입이 된다.

$$\forall T:\ \text{Nothing} <: T \implies \text{List<Nothing>} <: \text{List<}T\text{>}$$

그래서 표준 라이브러리의 빈 리스트 싱글턴은 실제로 `List<Nothing>` 타입이다.

```kotlin
// 표준 라이브러리 내부(개념):
internal object EmptyList : List<Nothing> { /* size=0, get은 예외 ... */ }

public fun <T> emptyList(): List<T> = EmptyList   // List<Nothing>을 List<T>로 안전하게 반환

val strings: List<String>     = emptyList()   // => List<Nothing>이 List<String>으로 통용
val points:  List<Coordinate> = emptyList()   // => 같은 EmptyList 인스턴스를 재사용 가능
```

원소가 없으므로 원소 타입이 `Nothing`이어도 `get`으로 `Nothing` 값을 꺼낼 위험이 없다. 애초에 꺼낼 원소가 없기 때문이다. 이것이 "값이 없는 타입"이 "값이 없는 컬렉션"의 원소 타입으로 정확히 들어맞는 이유이다. 공집합의 원소 타입은 공집합 타입 `Nothing`이다. 같은 논리가 `emptySet()`, `emptyMap()`(값 쪽), 그리고 예외를 담지 않는 성공 결과 등에도 적용된다.

> [!note] 명세 기준
> `emptyList<T>()`가 내부적으로 `List<Nothing>`인 싱글턴을 `List<T>`로 반환하는 것은 안전하다. `List<T>`는 읽기 전용(생산자) 인터페이스라서 `T` 값을 *받는* 연산이 없고, `Nothing`에서 위쪽으로 향하는 공변 변환만 일어나기 때문이다.
>
> 만약 `MutableList<Nothing>`을 `MutableList<String>`으로 쓰려 했다면 `add("x")`가 `Nothing` 자리에 `String`을 넣는 셈이 되어 건전하지 않았을(unsound) 것이다. 그래서 이 방법은 공변인 읽기 전용 컬렉션에서만 성립한다.

---

## 8. 합집합과 교집합: 지형에 들어오는 두 연산

### 8.1 Kotlin에는 일반 합집합 타입이 없다

TypeScript를 아는 독자라면 `String | Int`처럼 "이것 아니면 저것"을 뜻하는 합집합 타입(union type)에 익숙할 것이다. Scala 3에도 `String | Int`가 있다. 하지만 **Kotlin에는 이런 일반적인 합집합 타입이 없다.** `String`이거나 `Int`인 값을 담는 타입을 직접 적을 수 없다.

이는 의도한 설계 결정이다. 임의의 합집합은 타입 추론과 오버로드 해소를 복잡하게 만들기 때문에, Kotlin은 그 복잡성을 감수하는 대신 다른 도구로 같은 필요를 채운다.

그렇다면 "여러 타입 중 하나"는 어떻게 표현할까? Kotlin은 세 가지 대안을 제공한다.

- **공통 상위타입으로 올린다.** `String`과 `Int`가 모두 필요하면 둘의 공통 상위타입인 `Any`(또는 `Comparable<*>` 등)로 받는다. 대신 정밀함을 잃는다. `Any`로 받으면 다시 `is`로 좁혀야 한다.
- **봉인 클래스(sealed class/interface)로 닫힌 합을 만든다.** "이것 아니면 저것"에 해당하는 경우가 유한하고 미리 알려져 있다면, `sealed`로 대수적 합타입(sum type)을 명시적으로 만든다. Kotlin이 권장하는 정석적인 방법이며, 자세한 내용은 [[30 - sealed 클래스와 대수적 데이터 타입]]에서 다룬다.
- **널 가능성을 쓴다.** 이것만은 언어에 내장된 특수한 합집합이다(아래 설명).

```kotlin
sealed interface Shape
data class Circle(val r: Double) : Shape
data class Rectangle(val w: Double, val h: Double) : Shape
// Shape는 "Circle | Rectangle"의 닫힌 합을 타입 하나로 표현한다. when 완전성 검사도 붙는다
```

`String?`은 사실상 `String | Null`이라는 합집합이다. Kotlin은 일반 합집합은 허용하지 않으면서도, "타입 `T` 또는 널"이라는 *특수한 합집합 하나*만은 `?` 문법으로 특별하게 다룬다. 8.2에서 이 관점을 더 정확하게 설명한다.

### 8.2 `T?`는 특수한 합집합 타입이다

지형의 관점에서 널 가능성을 다시 보자. `String?`의 값 집합은 `String`의 값들에 `null` 하나를 더한 것이다.

$$\text{String?} = \text{String} \cup \{\texttt{null}\} = \text{String} \cup \text{Nothing?}$$

즉 `String?`은 `String`과 `Nothing?`의 합집합이다. Kotlin이 일반 합집합은 없으면서도 이 합집합만은 지원하는 이유는 두 가지다. 널 가능성은 실무에서 압도적으로 흔하고, 이 한 형태(`T` 또는 널)로 한정하면 타입 추론이 감당할 수 있기 때문이다. `?`는 지형에서 `null`이라는 원소 하나를 추가해 한 칸 위의 타입으로 올라가는, 잘 통제된 합집합 연산자이다.

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

`when`으로 `null`인 경우를 분리하면 `else` 분기에서 `maybeName`은 합집합의 `String` 부분으로 좁혀진다. 이것이 스마트 캐스트이다(8.3). 합집합을 나누어 각 부분으로 좁히는 이 흐름이 널 안전성이 동작하는 원리이며, 자세한 내용은 6장(널 안전성)에서 다룬다. 이 장에서는 "`?`가 지형에서 하는 일은 특수한 합집합을 만드는 것"이라는 위치만 확인한다.

### 8.3 교집합 타입은 있다: 스마트 캐스트가 만든다

합집합의 쌍대(dual)는 교집합(intersection)이다. `A & B`는 "`A`이면서 동시에 `B`인 값"의 타입이다. Kotlin에는 일반 합집합이 없지만, **교집합 타입은 스마트 캐스트의 결과로 실제로 존재한다.** 다만 대부분은 소스 코드에 직접 적을 수 없는 비-표기(non-denotable) 타입으로만 나타난다.

값 하나가 두 개의 `is` 검사를 모두 통과하면 컴파일러는 그 값을 두 타입의 교집합으로 좁힌다.

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

`x is Named && x is Aged` 이후의 블록에서 `x`는 `Named`이면서 `Aged`인 값이다. 컴파일러는 `x`의 타입을 교집합 `Named & Aged`로 두어, `name`과 `age`를 모두 별도의 캐스트 없이 호출할 수 있게 한다. 이 `Named & Aged`는 변수 선언에 `val y: Named & Aged`처럼 적을 수 없는 타입이지만(2.x 기준 일반 교집합은 표기할 수 없다), 타입 검사 내부에서는 분명히 존재하며 K2 컴파일러가 정밀하게 추적한다.

집합 관점에서 보면 당연하다. `Named & Aged`는 `Named`의 값 집합과 `Aged`의 값 집합의 교집합이다. 두 조건을 모두 통과한 `x`는 정확히 그 교집합에 속한다. 합집합이 지형에서 "위로 넓히기"라면 교집합은 "아래로 좁히기"이다. 스마트 캐스트는 지형을 아래로 좁히는 연산이고, 그 이론적 배경이 바로 이 교집합 타입이다.

### 8.4 정의상 비-널 타입 `T & Any`: 소스에 적을 수 있는 유일한 교집합

일반 교집합은 적을 수 없지만, Kotlin 1.7 이후(2.x 포함)에는 *딱 한 형태*의 교집합을 소스에 적을 수 있다. 바로 **정의상 비-널 타입**(definitely non-nullable type) `T & Any`이다. 이 타입은 "타입 파라미터 `T`이면서 동시에 `Any`(널 아님)인 값", 즉 `T`에서 널 가능성을 제거한 교집합이다.

이 표기가 필요한 이유는 제네릭에 있다. 타입 파라미터 `T`의 기본 상한이 `Any?`이므로 `T`는 널을 포함할 수 있다. 그런데 "`T`가 무엇이든 이 자리에는 널이 아닌 `T`가 와야 한다"는 것을 표현하고 싶을 때가 있다. 상한을 `T : Any`로 고정하면 호출자가 널 허용 타입 인자를 쓸 수 없으므로 제약이 너무 강하다. 대신 `T & Any`를 쓰면 `T`는 널 허용이어도 되면서, 이 특정 자리만 비-널로 좁힐 수 있다.

```kotlin
// T는 널 허용까지 허용하되, 반환값만은 반드시 비-널로 좁히고 싶다
fun <T> elvisLike(value: T, fallback: T & Any): T & Any =
    value ?: fallback
    // value: T (널일 수도), fallback: T & Any (비-널), 결과: T & Any (비-널)

val r1: String = elvisLike<String?>(null, "default")   // => "default" (비-null String)
val r2: Int    = elvisLike(null, 0)                    // => 0
```

`T & Any`는 지형에서 "`T`가 걸쳐 있던 널 층을 잘라내고 `Any` 아래로 좁힌" 교집합이다. Java 상호운용에서 플랫폼 타입([[45 - Java 상호운용]])의 널 가능성을 다룰 때 특히 유용하다. 이것이 Kotlin 소스에 명시적으로 적을 수 있는 유일한 교집합 타입이며, 일반 교집합(8.3)은 여전히 스마트 캐스트 내부에만 존재한다.

> [!note] 명세 기준
> `T & Any`에서 `Any`는 "널이 아님"을 뜻하는 상한 역할을 한다. `T`가 이미 비-널(`T : Any`)이면 `T & Any`는 `T`와 같다. `T`가 널 허용이면 `T & Any`는 그 널 가능성을 제거한 타입이다. 이 문법은 K2가 자리 잡기 전인 1.7에서 안정화되었고, 2.x에서 표준으로 쓰인다. 일반적인 두 클래스의 교집합(`Named & Aged` 같은)을 소스에 직접 적는 것은 별개의 문제이며, 2.x 기준으로 언어 문법에서 지원하지 않는다.

### 8.5 유연한 타입: 지형의 불확실한 구간

지형에는 양 끝(꼭대기와 바닥)과 분명한 층 말고도 "불확실한 구간"이 하나 있다. 유연한 타입(flexible type), 흔히 플랫폼 타입이라고 부르는 타입이다. Java 코드에서 넘어온 값은 Kotlin 컴파일러가 널 가능성을 확정할 수 없으므로 `String!`처럼 표기되는 유연한 타입 `(String..String?)`으로 다룬다. 이 타입은 "`String`처럼 써도 되고 `String?`처럼 써도 되지만, 컴파일러가 널 검사를 강제하지 않는" 위험한 구간이다.

유연한 타입은 이 장에서 자세히 다루지 않는다. 정확한 규칙과 함정(NPE가 컴파일 타임 검사를 통과해 새어 나오는 지점)은 45장(Java 상호운용)과 6장(널 안전성)에서 설명한다. 여기서는 지도 위에 이 구간의 위치만 표시해 둔다. 플랫폼 타입은 지형에서 `T`와 `T?` 사이 어딘가에 걸쳐 있으며, 컴파일러가 경계의 확정을 미뤄 둔 구간이다. Kotlin 코드 안에서 명시적 타입으로 `T`나 `T?`를 지정하는 순간 불확실성이 사라지고 위치가 확정된다.

---

## 9. 지형 위를 오가는 규칙: 정적 타입과 강타입의 실제

### 9.1 위로는 자동, 아래로는 명시: 두 방향의 비대칭

지형의 위아래를 오가는 규칙을 정리해 보자. **하위타입에서 상위타입으로(위로) 올라가는 것은 자동이고 비용도 없다.** 개를 동물로, 정수를 `Any`로, 무엇이든 `Any?`로 보는 것은 별도의 문법 없이 대입만으로 된다. 이를 상향 변환(upcast)이라고 하며, 부분집합의 원소는 언제나 상위집합의 원소이므로 항상 안전하다.

**반대로 상위타입에서 하위타입으로(아래로) 내려가려면 명시적인 검사나 캐스트가 필요하다.** `Any`로 받은 값을 `Coordinate`로 보려면 `as`(단정 캐스트), `is`(검사 후 스마트 캐스트), `as?`(안전 캐스트) 중 하나를 명시해야 한다. 상위집합의 원소가 반드시 하위집합에 속하는 것은 아니므로, 하향 변환(downcast)은 런타임에 실패할 수 있기 때문이다.

```kotlin
val any: Any = Coordinate(1, 2)      // 상향: Coordinate → Any, 자동

// val c: Coordinate = any            // 컴파일 에러: Any는 Coordinate가 아니다 (하향은 자동 불가)
val c1 = any as Coordinate            // 단정 캐스트: 틀리면 런타임 ClassCastException
val c2 = any as? Coordinate           // 안전 캐스트: 틀리면 null (c2: Coordinate?)
if (any is Coordinate) {
    println(any.x)                    // 스마트 캐스트: is 검사 후 블록 안에서 Coordinate로 좁혀짐
}
```

위로는 표시 없이, 아래로는 명시적으로 이동한다는 이 비대칭이 정적 타입과 강타입 규칙의 핵심이다. 컴파일러는 "안전한 방향(위)"만 조용히 허용하고, "위험한 방향(아래)"은 개발자가 책임지고 표기하도록 강제한다. `Nothing`이 바닥이고 `Any?`가 꼭대기라는 것은 이렇게 오르내리는 범위의 두 끝을 정하는 기준점이다.

### 9.2 타입 추론은 지형에서 위치를 찾는 일이다

Kotlin에는 타입을 일일이 적지 않아도 되는 강력한 지역 타입 추론(local type inference)이 있다. `val x = "cat"`에서 컴파일러는 `x`가 `String`이라는 것을 안다. 이 추론의 상당 부분이 이 장에서 살펴본 지형 위의 계산, 특히 LUB 계산이다.

```kotlin
val items = listOf(Circle(1.0), Rectangle(2.0, 3.0))   // items의 추론 타입은?
// 원소 타입 = LUB(Circle, Rectangle). 둘의 공통 상위타입이 sealed Shape라면 List<Shape>
// (Circle, Rectangle이 무관하면 LUB는 Any 근처까지 올라간다)

val values = listOf(1, 2.0, "three")   // LUB(Int, Double, String) = Comparable<*> & Serializable 근처
// 실제 추론: List<Comparable<*>> 또는 그 비슷한 공통 상위타입 — 정밀 타입은 무관 타입들의 LUB
```

서로 관계없는 타입을 섞으면 LUB가 `Any`나 `Comparable<*>` 같은 넓은 타입까지 올라가므로, 원소를 다시 쓰려면 `is`로 좁혀야 한다. 반대로 `sealed` 계층 안의 타입만 섞으면 LUB가 그 봉인된 상위타입에서 멈추므로 유용한 타입이 나온다. "추론된 타입이 왜 이렇게 넓지?" 하고 당황하는 경우는 대개 분기들의 LUB가 예상보다 위로 올라갔기 때문이다. 지형을 알면 이런 결과도 계산해서 예측할 수 있다.

### 9.3 `Any` 남용이라는 안티패턴: 지형을 포기하는 코드

지형이 주는 안전성을 스스로 포기하는 흔한 실수가 `Any`(또는 `Any?`) 남용이다. "무엇이든 담을 수 있어서 편하다"며 파라미터나 필드의 타입을 `Any`로 두면, 컴파일러의 정적 검사를 통째로 끄는 셈이 된다. `Any`로 받은 값은 다시 `is`/`as`로 좁혀야만 쓸 수 있고, 좁히는 과정이 틀리면 런타임에 오류가 난다.

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

`Any`는 리플렉션이나 직렬화 경계처럼 정말로 이질적인 값을 다뤄야 할 때 쓰는 최후의 수단이지, 설계의 기본값이 아니다. 지형의 꼭대기는 "모든 것을 받는다"는 편의를 주지만, 그 대가로 "아무것도 보장하지 않는다"는 위험도 함께 가져온다.

좋은 Kotlin 코드는 값이 속하는 정확한 층을 타입으로 지정한다. 그것이 제네릭(33장 제네릭 1)이든, 봉인 계층(30장 sealed 클래스)이든, 인터페이스([[25 - 인터페이스]])든 마찬가지다. 타입을 좁게 잡을수록 컴파일러가 더 많은 것을 검사해 준다.

### 9.4 멀티플랫폼에서도 지형은 그대로다

마지막으로, 이 지형은 어느 타깃에서나 같다. `Any`/`Any?`/`Unit`/`Nothing`은 [[01 - 코틀린이란 무엇인가 - 설계 철학과 세 개의 타깃]]에서 본 네 타깃인 JVM, Native, JS, Wasm 모두에서 같은 부분타입 규칙을 따른다. `Nothing`은 모든 타깃에서 바닥이고, `Unit`은 모든 타깃에서 값이 하나뿐인 타입이다.

다만 *물리적 구현*은 타깃마다 다르다. JVM에서 `Any`는 `java.lang.Object`로, `Unit`을 반환하는 함수는 `void` 메서드로 컴파일된다. Native, JS, Wasm에는 `Object`도 JVM의 `void`도 없으므로, 각 백엔드가 자기 방식으로 같은 의미를 구현한다. 그래서 "`Any`는 `Object`다", "`Unit`은 `void`로 컴파일된다"는 설명은 언제나 **JVM 백엔드에 한정된 구현 사실**로 읽어야 한다.

꼭대기는 `Any?`이고, 바닥은 `Nothing`이며, `Unit`은 값이 하나이고, `Nothing`은 값이 없다는 언어의 정의는 구현과 무관하게 성립한다. 이 장에서 그린 지도가 담고 있는 내용이 바로 이 정의이다.
