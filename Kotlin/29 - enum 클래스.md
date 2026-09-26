---
title: enum 클래스
date: 2026-07-13
tags: [kotlin, enum, entries, ordinal, exhaustiveness, 학습노트]
---

**이 장이 답하는 질문:**

- `enum class`의 각 상수는 "이름 붙은 정수"인가, 아니면 진짜 객체인가 — `Color.RED`를 로드하면 JVM 백엔드에서 메모리에 무엇이 태어나는가?
- enum 상수에 생성자 인자를 주고 메서드를 붙일 수 있다면, enum은 일반 클래스와 어디까지 같고 어디서부터 다른가?
- 상수마다 서로 다른 `override`를 주는 "상수별 본문(body)"은 컴파일러가 무엇으로 번역하는가 — 왜 `RED::class`와 enum 클래스 자체가 다를 수 있는가?
- `values()`는 왜 호출할 때마다 새 배열을 만들고, 2.x의 `entries`는 그 비용을 어떻게 없앴는가?
- `ordinal`에 의존하는 코드는 왜 시한폭탄인가 — 상수 하나를 알파벳순으로 정렬하겠다고 위로 옮기면 무슨 일이 벌어지는가?
- `when`이 enum을 완전(exhaustive)하게 덮었는지 컴파일러는 어떻게 알며, 여기에 `else`를 붙이는 순간 무엇을 잃는가? (M12)
- `enumValues<T>()`와 `enumValueOf<T>(name)`은 타입 소거의 세계에서 어떻게 런타임 타입을 손에 넣는가?
- 같은 "닫힌 집합"을 표현하는데 언제 `enum`을 쓰고 언제 `sealed`를 써야 하는가 — 무엇이 이 둘을 가르는가?

---

앞선 [[27 - 데이터 클래스]]에서 우리는 컴파일러가 값을 담는 클래스의 보일러플레이트를 대신 짜 주는 장면을 봤고, [[28 - object와 companion - 싱글턴과 동반 객체]]에서는 `object` 선언이 정적 클래스가 아니라 지연·스레드 안전하게 초기화되는 *진짜 인스턴스*라는 것(M24)을 못박았다. 이 장의 주인공인 `enum class`는 이 두 가닥이 만나는 지점에 서 있다. enum 상수 하나하나는 사실 그 enum 타입의 *싱글턴 인스턴스*이고, enum 클래스 전체는 컴파일러가 상당한 보일러플레이트 — 상수 배열, `name`, `ordinal`, `valueOf`, `Comparable` 구현 — 를 대신 생성해 주는 특수한 클래스다. enum은 "이름 붙은 상수의 집합"이라는 소박한 겉모습 아래 객체지향의 여러 장치가 응축된 구조물이다.

이 장이 다루는 것과 다루지 않는 것의 경계를 먼저 세우자. enum 상수의 정체(객체임), 생성자·프로퍼티·메서드, 상수별 익명 본문, 표준 멤버(`name`/`ordinal`/`values`/`entries`/`valueOf`), 인터페이스 구현, 그리고 reified 헬퍼가 이 장의 심장이다. 그러나 `when`이 *표현식*이라는 사실과 그 완전성 검사의 일반 이론은 [[17 - 표현식으로서의 제어 흐름 - if와 when]]이 정본으로 소유하며(M12), 여기서는 enum이 그 완전성을 *어떻게 충족시키는지와 어디서 사람을 배신하는지*만 본다. 제한된 계층·대수적 데이터 타입으로서의 봉인(sealed)과 enum의 근본적 차이는 [[30 - sealed 클래스와 대수적 데이터 타입]]이 소유하고(M23), 이 장은 마지막에 두 도구의 선택 기준만 대조한다. reified 타입 파라미터가 타입 소거를 우회하는 원리 자체는 [[15 - 인라인 함수와 reified]]와 [[34 - 제네릭 2 - 타입 소거와 reified와 바운드]]가 소유하고(M18, M19), 여기서는 `enumValues`/`enumValueOf`가 그 기계를 어떻게 활용하는지만 다룬다.

논증의 궤적은 이렇다. 먼저 enum이 해결하는 문제(마법의 상수와 타입 안전성)를 세우고, 각 상수가 진짜 객체임을 컴파일된 형태로 증명한다. 그다음 enum이 생성자·프로퍼티·메서드·추상 멤버를 가질 수 있다는 점에서 일반 클래스와 얼마나 같은지 보이고, 상수별 본문이 익명 서브클래스로 번역되는 기계를 파헤친다. 이어 표준 멤버들을 명세 수준에서 하나씩 해부하되 `ordinal` 의존의 위험과 `values()` vs `entries`의 성능 차이를 정면으로 다룬다. reified 헬퍼, `when` 완전성(M12 참조), 그리고 enum과 sealed의 선택 기준으로 마무리한다.

---

## 1. enum이란 무엇인가 — 닫힌 상수 집합이라는 문제

### 1.1 문제: 마법의 상수와 타입 없는 세계

프로그램에는 "가능한 값이 미리 정해진, 유한하고 고정된 집합"이 도처에 있다. 나침반의 네 방위, 카드의 네 무늬, 신호등의 세 색, 요일, 트럼프의 랭크. 이런 값을 표현하는 가장 원시적인 방법은 정수 상수다.

```kotlin
// 안티패턴: 마법의 정수로 방위를 표현
const val NORTH = 0
const val EAST = 1
const val SOUTH = 2
const val WEST = 3

fun rotateClockwise(dir: Int): Int = (dir + 1) % 4

fun main() {
    println(rotateClockwise(NORTH))  // => 1  (EAST를 뜻하지만 그냥 1)
    println(rotateClockwise(42))     // => 3  (42는 방위가 아닌데도 통과)
}
```

이 방식의 죄는 두 가지다. 첫째, **타입 안전성이 없다.** `rotateClockwise`의 파라미터 타입은 `Int`라서 방위가 아닌 아무 정수 `42`도 받아들이고, 컴파일러는 이를 막지 못한다. 반환값 `1`은 "EAST"라는 의미를 잃고 그냥 숫자다. 둘째, **자기 문서화가 안 된다.** 로그에 `dir=2`가 찍히면 그것이 SOUTH인지 알아내려면 상수 표를 뒤져야 한다. 문자열 상수(`"NORTH"`)로 바꿔도 오타(`"NROTH"`)를 컴파일러가 못 잡는다는 점에서 근본은 같다.

우리가 원하는 것은 "방위라는 새로운 *타입*을 만들고, 그 타입이 가질 수 있는 값은 정확히 이 넷뿐임을 컴파일러가 알고 강제하는 것"이다.

### 1.2 해법: enum class — 각 상수는 하나의 값이자 타입의 인스턴스

Kotlin의 `enum class`는 바로 이 요구를 언어 기능으로 흡수한다.

```kotlin
enum class Direction {
    NORTH, EAST, SOUTH, WEST
}

fun rotateClockwise(dir: Direction): Direction {
    val next = (dir.ordinal + 1) % Direction.entries.size
    return Direction.entries[next]
}

fun main() {
    println(rotateClockwise(Direction.NORTH))  // => EAST  (이름이 그대로 찍힌다)
    // rotateClockwise(42)                      // 컴파일 에러: Int는 Direction이 아님
    val d: Direction = Direction.SOUTH
    println(d)                                  // => SOUTH  (toString이 name을 반환)
}
```

`Direction`은 이제 하나의 타입이다. 그 타입의 값이 될 수 있는 것은 정확히 `NORTH`, `EAST`, `SOUTH`, `WEST` 넷뿐이며, 컴파일러가 이를 안다. `rotateClockwise(42)`는 컴파일조차 되지 않는다. `println(d)`는 `SOUTH`를 찍는다 — enum의 기본 `toString`은 상수의 `name`을 반환하기 때문이다.

여기서 핵심 통찰을 미리 못박자. **`Direction.NORTH`는 정수 `0`을 예쁘게 포장한 별명이 아니다. 그것은 `Direction` 타입의 실재하는 객체(인스턴스)다.** enum 상수는 값이자 동시에 그 enum 타입의 유일한 인스턴스(싱글턴)다 — 이 사실이 이 장의 나머지 전부를 떠받친다.

> **역사 메모** — enum이라는 개념은 C의 `enum`(정수 상수의 나열)에서 출발했지만, C의 enum은 그냥 정수라 타입 안전성이 없다. Java 5(2004)가 도입한 "typesafe enum"은 Joshua Bloch의 유명한 패턴("Effective Java"의 typesafe enum pattern)을 언어 기능으로 승격한 것으로, 각 상수를 진짜 객체로 만들었다. Kotlin의 enum은 이 Java 모델을 계승하되(JVM 백엔드에서 `java.lang.Enum`으로 매핑) 문법을 다듬고 `when` 완전성 검사·`entries` 같은 개선을 얹었다.

### 1.3 enum도 클래스다 — 이 장이 놀라운 이유

초심자는 흔히 enum을 "상수 목록을 적는 특수 문법"으로만 이해하고 멈춘다. 그러나 `enum class`의 `class`는 장식이 아니다. enum은 생성자를 가질 수 있고, 프로퍼티를 가질 수 있고, 메서드를 가질 수 있고, 추상 메서드를 선언하고 상수마다 다르게 구현할 수 있고, 인터페이스를 구현할 수 있고, 동반 객체(companion object)를 가질 수 있다. enum은 "인스턴스의 개수가 컴파일 타임에 고정된 클래스"에 가깝다.

```text
enum이 할 수 있는 것 / 할 수 없는 것

할 수 있는 것                          할 수 없는 것
─────────────────────────────         ─────────────────────────────
주 생성자 + 상수별 인자                 다른 클래스 상속 (이미 Enum 상속)
프로퍼티 (val/var)                     open / abstract enum (상수 개수 고정)
메서드 (일반/추상)                      enum을 상속해 상수 추가
상수별 익명 본문(override)              생성자를 외부에서 호출 (new 불가)
인터페이스 구현                         타입 파라미터 (enum은 제네릭 불가)
companion object, 중첩 클래스           상수의 재대입 (전부 사실상 final)
```

이 표의 왼쪽이 이 장에서 우리가 하나씩 풀어낼 능력들이고, 오른쪽은 enum의 본질("닫힌, 고정된 인스턴스 집합")에서 필연적으로 따라 나오는 제약들이다.

---

## 2. enum 상수는 객체다 — 컴파일된 형태를 뜯어보기

### 2.1 각 상수 = 그 타입의 싱글턴 인스턴스

앞 절에서 못박은 "상수는 객체"라는 명제를 이제 증명하자. enum 상수는 [[28 - object와 companion - 싱글턴과 동반 객체]]에서 본 `object`처럼 그 타입의 단 하나뿐인 인스턴스다(M24 참조). 그래서 `===`(참조 동일성, [[10 - 불리언과 동등성과 동일성]])로 비교해도 의미가 있고, 같은 상수는 항상 같은 객체다.

```kotlin
enum class Suit { HEARTS, DIAMONDS, CLUBS, SPADES }

fun main() {
    val a = Suit.HEARTS
    val b = Suit.HEARTS
    println(a === b)        // => true   (같은 싱글턴 인스턴스, 참조가 같다)
    println(a == b)         // => true   (equals도 당연히 true)
    println(a is Suit)      // => true   (Suit 타입의 인스턴스)
    println(a is Enum<*>)   // => true   (모든 enum 상수는 Enum의 하위)
}
```

`a === b`가 `true`라는 사실이 결정적이다. 만약 상수가 "값 복사되는 정수"였다면 참조 비교는 무의미했을 것이다. enum 상수가 진짜 인스턴스이기 때문에, enum 값 비교에서는 `==`와 `===`가 항상 같은 답을 준다 — enum은 `equals`를 참조 동일성으로 구현하므로(정확히는 `Enum.equals`가 `this === other`), 두 연산자가 일치한다. 이는 [[10 - 불리언과 동등성과 동일성]]에서 다룬 박싱 캐시의 예측 불가(M48)와 대조적이다. enum에서는 `===`가 언제나 안전하고 의미 있다.

### 2.2 JVM 백엔드에서의 컴파일 — final 클래스와 정적 필드

이제 컴파일러가 enum을 무엇으로 번역하는지 보자. 아래는 **JVM 백엔드에서** enum이 생성하는 구조의 개념적 등가물이다(실제 바이트코드는 더 복잡하지만 형태는 이렇다).

```java
// enum class Suit { HEARTS, DIAMONDS, CLUBS, SPADES } 의 JVM 개념 등가물
public final class Suit extends java.lang.Enum<Suit> {
    // 각 상수는 public static final 필드 = 그 클래스의 인스턴스
    public static final Suit HEARTS   = new Suit("HEARTS", 0);
    public static final Suit DIAMONDS = new Suit("DIAMONDS", 1);
    public static final Suit CLUBS    = new Suit("CLUBS", 2);
    public static final Suit SPADES   = new Suit("SPADES", 3);

    // 모든 상수를 담은 배열 (values()의 원본)
    private static final Suit[] $VALUES = { HEARTS, DIAMONDS, CLUBS, SPADES };

    // 생성자는 private — 외부에서 new Suit(...) 불가
    private Suit(String name, int ordinal) { super(name, ordinal); }

    public static Suit[] values() { return $VALUES.clone(); } // 방어적 복사!
    public static Suit valueOf(String name) { /* 선형/맵 탐색, 없으면 예외 */ }
}
```

세 가지를 주목하라. 첫째, **enum 클래스는 `final`이다.** 상속을 허용하면 상수 집합이 열려 버리므로("누군가 서브클래스로 다섯 번째 무늬를 추가") enum의 본질과 모순된다. 둘째, **각 상수는 `public static final` 필드**로, 클래스가 로드될 때 정적 초기화 블록에서 딱 한 번 생성된다. 셋째, **생성자는 private**라 `new Suit(...)`로 인스턴스를 더 만들 수 없다. 인스턴스의 개수와 정체가 컴파일 타임에 완전히 고정된다.

JVM 백엔드에서 Kotlin의 `Suit`은 `java.lang.Enum<Suit>`을 상속한다(Kotlin 소스에서는 `kotlin.Enum`이지만 JVM에서 이는 `java.lang.Enum`으로 매핑된다). 이 상속 관계가 `name`·`ordinal`·`compareTo`·`equals`·`hashCode`를 전부 상위 `Enum`에서 물려받게 하는 뿌리다.

> **명세 정밀** — "각 상수가 `static final` 필드로 컴파일된다"는 서술은 **JVM 백엔드**의 이야기다. Kotlin/Native나 Kotlin/JS는 JVM 바이트코드를 생성하지 않으므로 저수준 표현은 다르다. 그러나 언어 수준의 *의미* — 상수는 유일한 싱글턴 인스턴스이고, `entries`/`valueOf`/`name`/`ordinal`이 제공되며, 생성자를 외부에서 호출할 수 없다 — 는 모든 백엔드에서 동일하게 보장된다. 저수준 표현이 궁금하면 IntelliJ의 "Show Kotlin Bytecode → Decompile"로 JVM 산출물을 직접 확인할 수 있다.

### 2.3 초기화 순서 — 상수는 언제 태어나는가

enum 상수는 그 enum 클래스가 처음 로드/사용될 때 정적 초기화 단계에서 **선언 순서대로** 생성된다. 이 순서는 `ordinal` 값과 정확히 일치한다(첫 상수가 0). 상수에 생성자 인자나 본문이 있으면 그 초기화도 이때 함께 일어난다.

```kotlin
enum class Planet(val mass: Double) {
    MERCURY(3.30e23),
    VENUS(4.87e24),
    EARTH(5.97e24);   // 상수 목록 뒤에 멤버가 오면 세미콜론 필수

    init {
        println("초기화: $name (ordinal=$ordinal, mass=$mass)")
    }
}

fun main() {
    println("main 시작")
    val e = Planet.EARTH   // 이 접근이 Planet 클래스 로드를 촉발
    println("고른 행성: $e")
}
// => main 시작
// => 초기화: MERCURY (ordinal=0, mass=3.3E23)
// => 초기화: VENUS (ordinal=1, mass=4.87E24)
// => 초기화: EARTH (ordinal=2, mass=5.97E24)
// => 고른 행성: EARTH
```

주목할 점: `Planet.EARTH` 하나만 접근했는데 `MERCURY`, `VENUS`도 초기화됐다. enum은 **모든 상수가 클래스 로드 시 한꺼번에** 만들어진다(개별 지연 로딩이 아니다). 그리고 `init` 블록은 *각 상수가 생성될 때마다* 실행된다 — 위에서 `init`이 세 번 찍힌 이유다. 이는 [[21 - 클래스와 생성자]]의 초기화 순서 규칙(M46)이 enum 상수 각각에 적용되는 것이다: 각 상수는 주 생성자를 통해 태어나고, 프로퍼티 초기화와 `init` 블록이 그 안에서 순서대로 돈다.

> **성능 주의** — enum 상수 목록이 매우 크고 각 상수의 생성자가 무거운 계산을 한다면(예: 정규식 컴파일, 큰 테이블 로드), 그 비용은 enum 클래스가 처음 접근되는 순간 전부 한꺼번에 지불된다. 대부분의 경우 무시할 만하지만, "상수 하나만 쓰는데 200개가 다 초기화된다"는 점은 기억할 가치가 있다.

또한 위 예제는 Kotlin에서 세미콜론이 **거의 유일하게 필수인 자리**를 보여준다: enum 상수 목록 뒤에 프로퍼티·메서드·`init` 등 다른 멤버가 이어지면, 상수 목록의 끝을 세미콜론으로 명시해야 한다([[03 - 렉시컬 구조와 세미콜론 추론]] 연결). 상수만 있고 다른 멤버가 없으면 세미콜론은 생략 가능하다.

---

## 3. 생성자·프로퍼티·메서드 — enum도 진짜 클래스다

### 3.1 주 생성자와 상수별 인자

enum 클래스는 주 생성자를 가질 수 있고, 각 상수는 그 생성자에 넘길 인자를 괄호로 지정한다. 이것이 enum을 "이름표 붙은 데이터 레코드의 고정 집합"으로 만든다.

```kotlin
enum class Coin(val cents: Int) {
    PENNY(1),
    NICKEL(5),
    DIME(10),
    QUARTER(25);

    fun valueInDollars(): Double = cents / 100.0
}

fun main() {
    println(Coin.QUARTER.cents)             // => 25
    println(Coin.DIME.valueInDollars())     // => 0.1
    val total = Coin.entries.sumOf { it.cents }
    println(total)                          // => 41
}
```

`Coin`의 주 생성자는 `(val cents: Int)`이고, 각 상수는 `PENNY(1)`처럼 자신의 `cents` 값을 넘긴다. `val cents`는 프로퍼티이므로 `Coin.QUARTER.cents`로 읽을 수 있다. enum 생성자는 항상 사실상 private다 — 오직 상수 선언만이 그것을 호출할 수 있다. 그래서 `Coin(99)`처럼 임의 인스턴스를 만들 수 없고, 이것이 "가능한 값은 이 넷뿐"이라는 불변식을 지킨다.

여러 인자, 기본 인자, 심지어 부 생성자도 가능하다. enum은 이 점에서 일반 클래스와 다르지 않다.

```kotlin
enum class HttpStatus(val code: Int, val message: String) {
    OK(200, "OK"),
    NOT_FOUND(404, "Not Found"),
    SERVER_ERROR(500, "Internal Server Error");

    val isError: Boolean get() = code >= 400   // 계산 프로퍼티 (백킹 필드 없음)

    override fun toString(): String = "$code $message"
}

fun main() {
    println(HttpStatus.NOT_FOUND)            // => 404 Not Found  (재정의한 toString)
    println(HttpStatus.OK.isError)           // => false
    println(HttpStatus.SERVER_ERROR.isError) // => true
}
```

`isError`는 백킹 필드 없는 계산 프로퍼티([[22 - 프로퍼티와 백킹 필드]], M35 참조)이고, `toString`은 자유롭게 재정의할 수 있다(기본은 `name`을 반환하지만 위처럼 바꿀 수 있다). enum이 얼마나 온전한 클래스인지 여기서 분명해진다.

### 3.2 프로퍼티 — 상수마다 딸린 데이터

enum의 프로퍼티는 두 방식으로 값을 얻는다. (1) 위처럼 **주 생성자 인자를 통해 상수별로 다른 값**을 받거나, (2) 모든 상수가 **공유하는 계산·기본 프로퍼티**로 정의하거나. `var` 프로퍼티도 문법적으로는 허용되지만, enum 상수는 싱글턴이므로 가변 상태를 두면 전역 가변 상태가 되어 스레드 안전성과 예측 가능성을 해친다.

```kotlin
enum class TrafficLight(val durationSeconds: Int) {
    RED(30), YELLOW(5), GREEN(25);

    var timesActivated: Int = 0   // 나쁜 생각: enum에 가변 상태 = 전역 가변 상태
}
```

`TrafficLight.RED.timesActivated++`는 프로그램 전역에서 공유되는 `RED` 인스턴스의 상태를 바꾼다. enum 상수는 사실상 전역 싱글턴이므로, 여기에 가변 프로퍼티를 두는 것은 전역 변수를 두는 것과 같다. enum의 프로퍼티는 상수를 특징짓는 *불변 데이터*(`val`)로 두는 것이 정석이다.

### 3.3 메서드와 추상 메서드 — 상수에 행위를 붙이기

enum은 일반 메서드는 물론 **추상 메서드**를 선언할 수 있다. 추상 메서드를 선언하면 *모든 상수가 그것을 반드시 구현*해야 한다 — 이것이 다음 절의 "상수별 본문"으로 이어진다.

```kotlin
enum class ArithmeticOp {
    PLUS {
        override fun apply(a: Int, b: Int) = a + b
    },
    MINUS {
        override fun apply(a: Int, b: Int) = a - b
    },
    TIMES {
        override fun apply(a: Int, b: Int) = a * b
    };

    abstract fun apply(a: Int, b: Int): Int   // 모든 상수가 구현해야 함
}

fun main() {
    for (op in ArithmeticOp.entries) {
        println("$op: ${op.apply(6, 4)}")
    }
}
// => PLUS: 10
// => MINUS: 2
// => TIMES: 24
```

`abstract fun apply`는 각 상수가 자신만의 방식으로 구현하도록 강제한다. `PLUS`는 덧셈을, `MINUS`는 뺄셈을 안다. 이것은 "다형성을 enum 상수 안에 접어 넣는" 강력한 패턴이다. 그러나 이 편리함 뒤에는 컴파일러가 조용히 벌이는 일 — 각 상수를 익명 서브클래스로 만드는 것 — 이 있고, 그것이 다음 절의 주제다.

---

## 4. 상수별 본문 — 익명 서브클래스로서의 상수

### 4.1 각 상수가 자기만의 override를 가질 때

상수 이름 뒤에 `{ ... }` 블록을 붙이면 그 상수는 자신만의 멤버를 가지거나 상속받은 멤버를 재정의할 수 있다. 앞 절의 `ArithmeticOp`가 그 예다. 이 "상수별 본문"은 추상 메서드 구현뿐 아니라 임의의 override에도 쓸 수 있다.

```kotlin
enum class Weekday {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY,
    FRIDAY {
        override fun isFun(): Boolean = true      // 금요일만 특별
    },
    SATURDAY {
        override fun isFun(): Boolean = true
    },
    SUNDAY {
        override fun isFun(): Boolean = true
    };

    open fun isFun(): Boolean = false             // 기본: 재미없음
}

fun main() {
    println(Weekday.MONDAY.isFun())   // => false  (기본 구현)
    println(Weekday.FRIDAY.isFun())   // => true   (상수별 재정의)
}
```

`isFun`은 `open` 기본 구현을 두고, `FRIDAY`/`SATURDAY`/`SUNDAY`만 재정의했다. 재정의하려는 멤버는 `open`이거나 `abstract`여야 한다(일반 클래스의 오버라이드 규칙 [[24 - 상속과 오버라이딩과 초기화 순서]]와 동일).

### 4.2 컴파일: 상수별 본문 = 익명 서브클래스

여기서 결정적인 내부 사실이 나온다. **본문을 가진 enum 상수는 컴파일러가 그 enum의 익명 서브클래스로 번역한다.** 즉 `FRIDAY`는 `Weekday` 자체의 인스턴스가 아니라, `Weekday`를 상속한 이름 없는 서브클래스의 인스턴스다.

```text
enum class Weekday { ... FRIDAY { override fun isFun() = true } ... }

컴파일 개념 (JVM 백엔드):

  Weekday (본문 없는 상수의 직접 타입, isFun의 기본 구현 보유)
    │
    ├── MONDAY   : Weekday 인스턴스 (본문 없음 → Weekday 직접)
    ├── TUESDAY  : Weekday 인스턴스
    ├── ...
    ├── FRIDAY   : Weekday$1 인스턴스  ← 익명 서브클래스! (isFun 재정의)
    ├── SATURDAY : Weekday$2 인스턴스  ← 또 다른 익명 서브클래스
    └── SUNDAY   : Weekday$3 인스턴스
```

이 사실은 관찰 가능한 결과를 낳는다. 본문 없는 상수의 런타임 클래스는 enum 클래스 자신이지만, 본문 있는 상수의 런타임 클래스는 익명 서브클래스다.

```kotlin
fun main() {
    // JVM 백엔드 기준 관찰
    println(Weekday.MONDAY.javaClass.name)   // => ...Weekday          (enum 클래스 자체)
    println(Weekday.FRIDAY.javaClass.name)   // => ...Weekday$1        (익명 서브클래스!)
    println(Weekday.MONDAY::class == Weekday.FRIDAY::class)  // => false

    // 하지만 enum 타입으로서의 정체성은 유지된다
    println(Weekday.FRIDAY is Weekday)       // => true
    println(Weekday.FRIDAY.name)             // => FRIDAY   (name/ordinal은 그대로)
    println(Weekday.FRIDAY.ordinal)          // => 4
}
```

`MONDAY.javaClass`와 `FRIDAY.javaClass`가 다르다(위 출력은 JVM 백엔드 기준). `FRIDAY`는 `Weekday$1`이라는 익명 서브클래스의 인스턴스이기 때문이다. 그럼에도 `FRIDAY is Weekday`는 `true`이고 `name`/`ordinal`은 정상 작동한다 — 서브클래스이므로 `is` 검사와 상속 멤버가 모두 유지된다.

> **흔한 오해** — "`RED::class`로 enum 클래스를 얻을 수 있다"는 생각. 본문 없는 상수라면 맞지만, 본문 있는 상수의 `::class`는 익명 서브클래스를 가리킨다. enum *타입* 자체가 필요하면 상수의 클래스가 아니라 `Weekday::class` 또는 `MONDAY.declaringJavaClass`(JVM)를 써야 한다. 이 미묘함은 리플렉션([[44 - 애노테이션과 리플렉션]])이나 직렬화 라이브러리에서 실제 버그를 낳는다.

### 4.3 함정: 상수별 본문과 완전성·비교

상수별 본문이 익명 서브클래스라는 사실은 두 가지 부수 효과를 낳는다. 첫째, 본문 있는 상수가 섞여 있으면 `values()`/`entries`의 원소들은 여러 서로 다른 런타임 타입을 섞는다(본문 없는 상수는 enum 타입 자체, 본문 있는 상수는 익명 서브클래스). 여기서 미묘한 점: enum 클래스가 *추상 멤버*를 선언하면 — 즉 모든 상수가 본문으로 그것을 구현해야 하면(`ArithmeticOp`처럼) — enum 클래스 자체가 추상으로 컴파일되지만, `open` 멤버를 *일부* 상수만 재정의하면(위 `Weekday`처럼) 본문 없는 상수가 enum 타입의 직접 인스턴스여야 하므로 enum 클래스는 여전히 구체(concrete) 클래스로 남는다. 둘째, 그럼에도 `compareTo`·`ordinal`·`name`·`when` 완전성은 전혀 영향받지 않는다 — 이들은 전부 상위 `Enum`의 계약에 기반하며 런타임 서브클래스와 무관하기 때문이다. 즉 상수별 본문은 "각 상수에 다른 행위를 주는" 도구일 뿐, enum의 정체성 규칙을 흔들지 않는다. 다만 "각 상수의 구현이 조금씩 다르다"는 점 때문에 코드가 커지면 [[30 - sealed 클래스와 대수적 데이터 타입]]의 sealed 계층이 더 나은 선택이 되는 임계점이 있다(9절에서 다룬다).

---

## 5. enum과 인터페이스 — 능력은 구현하되 상속은 못한다

### 5.1 인터페이스 구현

enum은 다른 클래스를 상속할 수 없다(이미 `Enum`을 상속하므로, JVM은 단일 상속). 그러나 **인터페이스는 얼마든지 구현할 수 있다.** 이는 enum에 공통 계약을 부여하는 정석적 방법이다.

```kotlin
interface Describable {
    fun describe(): String
}

enum class Direction(val dx: Int, val dy: Int) : Describable {
    NORTH(0, 1), EAST(1, 0), SOUTH(0, -1), WEST(-1, 0);

    override fun describe(): String = "$name: 이동 벡터 ($dx, $dy)"

    fun opposite(): Direction = when (this) {
        NORTH -> SOUTH
        SOUTH -> NORTH
        EAST  -> WEST
        WEST  -> EAST
    }
}

fun printAll(items: List<Describable>) {
    items.forEach { println(it.describe()) }
}

fun main() {
    printAll(Direction.entries)   // Direction은 Describable로 취급 가능
    println(Direction.NORTH.opposite())  // => SOUTH
}
// => NORTH: 이동 벡터 (0, 1)
// => EAST: 이동 벡터 (1, 0)
// => SOUTH: 이동 벡터 (0, -1)
// => WEST: 이동 벡터 (-1, 0)
```

`Direction`이 `Describable`을 구현하므로, `Direction` 값들을 `List<Describable>`로 다룰 수 있다. enum이 상위 타입([[25 - 인터페이스]])을 가질 수 있다는 점은 다형적 코드에서 enum을 일급 시민으로 만든다. 인터페이스 메서드는 enum 전체가 공통으로 구현할 수도, 상수별 본문에서 상수마다 다르게 구현할 수도 있다.

### 5.2 인터페이스 구현을 상수별로 나누기

추상 메서드와 상수별 본문을 결합하면, "인터페이스가 요구하는 메서드를 각 상수가 자기 방식으로 구현"하는 강력한 표현이 나온다.

```kotlin
interface Operation {
    fun apply(x: Double): Double
}

enum class UnaryFunction : Operation {
    SQUARE {
        override fun apply(x: Double) = x * x
    },
    NEGATE {
        override fun apply(x: Double) = -x
    },
    RECIPROCAL {
        override fun apply(x: Double) = 1.0 / x
    };
    // Operation.apply를 각 상수가 상수별 본문에서 구현
}

fun main() {
    println(UnaryFunction.SQUARE.apply(3.0))      // => 9.0
    println(UnaryFunction.RECIPROCAL.apply(4.0))  // => 0.25
}
```

여기서 `UnaryFunction`은 `Operation.apply`에 대한 자체 기본 구현이 없으므로, 인터페이스 메서드가 곧 추상 멤버 역할을 하여 각 상수가 반드시 구현하게 된다. 이 구조는 "함수 객체의 유한한 카탈로그"를 타입 안전하게 담는 관용구다.

> **명세 정밀** — enum이 `fun interface`(SAM 인터페이스, [[25 - 인터페이스]])를 구현하는 것도 가능하지만, enum 상수 자체를 람다로 SAM 변환할 수는 없다(enum 상수는 함수 값이 아니라 명명된 인스턴스다). enum과 인터페이스의 결합은 "명명된 유한 집합 + 공통 계약"이 필요할 때 가장 빛난다.

---

## 6. 표준 멤버 — name, ordinal, values, entries, valueOf

enum 클래스는 컴파일러와 표준 라이브러리가 제공하는 일련의 멤버를 자동으로 갖는다. 이들을 정확히 이해하는 것이 enum을 안전하게 쓰는 핵심이다.

### 6.1 name과 ordinal — 그리고 ordinal 의존의 위험

모든 enum 상수는 상위 `Enum`으로부터 두 프로퍼티를 물려받는다.

- **`name: String`** — 소스에 선언된 상수 이름 그대로("NORTH"). `toString`의 기본 반환값.
- **`ordinal: Int`** — 선언 순서(0부터). 첫 상수가 0, 다음이 1, ….

```kotlin
enum class Priority { LOW, MEDIUM, HIGH, CRITICAL }

fun main() {
    println(Priority.HIGH.name)      // => HIGH
    println(Priority.HIGH.ordinal)   // => 2
    println(Priority.LOW.ordinal)    // => 0
    // 비교는 ordinal 기반
    println(Priority.LOW < Priority.HIGH)   // => true  (0 < 2)
}
```

`name`과 `ordinal`은 `val`이며 재정의할 수 없다(final). `ordinal`은 편리하지만 **깨지기 쉬운 의존**이다. `ordinal` 값은 순전히 *선언 순서*의 부산물이지, 상수의 본질적 속성이 아니다. 누군가 가독성을 위해 상수 순서를 재배열하거나 중간에 새 상수를 끼워 넣으면 모든 `ordinal`이 조용히 바뀐다.

```kotlin
// 위험한 코드: ordinal을 영속 데이터로 저장
enum class Grade { F, D, C, B, A }   // A.ordinal == 4

// DB에 grade.ordinal(=4)로 A를 저장했다고 하자.
// 나중에 누군가 낙제 등급을 세분화한다:
enum class GradeV2 { F, F_MINUS, D, C, B, A }   // 이제 A.ordinal == 5!
// 예전에 저장한 "4"는 이제 B를 뜻한다 → 데이터 손상, 컴파일러는 침묵
```

`ordinal`(또는 정렬을 위한 `ordinal` 기반 비교)에 의존해 값을 **직렬화·저장·전송**하면, 상수 순서 변경이 곧 데이터 손상이 된다. 영속화가 필요하면 `ordinal` 대신 `name`(문자열)을 저장하거나, 명시적으로 안정적인 정수 코드를 프로퍼티로 부여하라.

```kotlin
enum class Grade(val dbCode: Int) {   // 명시적, 안정적인 코드
    F(0), D(1), C(2), B(3), A(4);

    companion object {
        private val byCode = entries.associateBy { it.dbCode }
        fun fromCode(code: Int): Grade =
            byCode[code] ?: error("알 수 없는 등급 코드: $code")
    }
}
```

이제 상수 순서를 바꿔도 `dbCode`는 그대로이므로 저장된 데이터가 안전하다. `name`의 안정성도 절대적이지는 않다(상수 이름을 리팩터링하면 깨진다) — 이 경우 명시적 코드 프로퍼티가 가장 견고하다.

> **흔한 오해** — "`ordinal`은 그 상수의 고유 ID다." 아니다. `ordinal`은 *선언 위치*일 뿐, 상수의 정체성과 무관한 우연한 값이다. `ordinal`을 로직·저장·비교의 근거로 삼는 것은 소스 코드의 줄 순서를 프로그램의 의미로 승격시키는 것이다. `ordinal`의 정당한 용도는 그 순서 자체가 의미인 경우(예: 요일·크기 등급의 자연 순서 정렬)에 한정된다.

### 6.2 values() vs entries — 방어적 복사와 그 종말

enum의 모든 상수를 순회하려면 전체 목록이 필요하다. 역사적으로 이는 컴파일러가 생성하는 정적 `values()` 메서드로 제공됐다.

```kotlin
enum class Season { SPRING, SUMMER, AUTUMN, WINTER }

fun main() {
    val arr: Array<Season> = Season.values()   // Array<Season> 반환
    for (s in arr) print("$s ")                 // => SPRING SUMMER AUTUMN WINTER
    println()
}
```

`values()`에는 미묘하지만 실질적인 문제가 있다. **반환 타입이 `Array<Season>`(가변 배열)이고, 호출할 때마다 새 배열을 복사해서 준다.** 배열이 가변이라 호출자가 원소를 바꿀 수 있으므로, enum의 내부 원본이 오염되지 않도록 매 호출마다 방어적 복사(`clone`)를 하는 것이다.

```kotlin
fun main() {
    val a = Season.values()
    val b = Season.values()
    println(a === b)   // => false  (매번 새 배열!)
    a[0] = Season.WINTER   // 배열은 가변이라 이런 짓이 가능(원본엔 영향 없음)
    println(Season.values()[0])  // => SPRING  (원본은 안전, 하지만 복사 비용 지불)
}
```

루프 안에서 `values()`를 반복 호출하면 매번 배열 복사 비용을 낸다. 이 두 문제(가변 반환·반복 복사)를 해결하기 위해 Kotlin 1.9에서 **`entries` 프로퍼티**가 도입되어 2.x에서 표준으로 자리 잡았다.

```kotlin
fun main() {
    val list: EnumEntries<Season> = Season.entries   // 불변 List (EnumEntries<E>)
    for (s in list) print("$s ")                      // => SPRING SUMMER AUTUMN WINTER
    println()

    println(Season.entries === Season.entries)  // => true  (캐시된 단일 인스턴스!)
    // Season.entries[0] = ...  // 컴파일 에러: List는 읽기 전용, set 없음
}
```

`entries`는 세 가지 면에서 `values()`보다 낫다.

| 특성 | `values()` | `entries` (1.9+) |
|------|-----------|------------------|
| 반환 타입 | `Array<E>` (가변) | `EnumEntries<E>` (읽기 전용 `List<E>`) |
| 호출당 할당 | 매번 새 배열 복사 | 캐시된 단일 인스턴스, 할당 없음 |
| 가변성 | 원소 재대입 가능 | 읽기 전용 (M26 참조) |
| 컬렉션 API | 배열 API | `List` 확장 전체 사용 가능 |

`entries`는 `List<E>`를 상속하므로 `map`/`filter`/`associateBy` 같은 컬렉션 연산([[39 - 컬렉션 연산과 함수형 파이프라인]])을 곧바로 쓸 수 있고, 매번 복사하지 않으므로 성능도 낫다. **2.x 코드에서는 `values()` 대신 `entries`를 쓰는 것이 권장 관용구다.** 단, `entries`가 반환하는 것은 "읽기 전용 뷰"이지 컬렉션이 내부적으로 불변 자료구조라는 뜻이 아니다 — 다만 enum 상수 집합 자체가 컴파일 타임 고정이므로 실질적으로 완전히 불변이다(읽기 전용 ≠ 불변 일반론은 M26, [[38 - 컬렉션 - 읽기 전용과 가변]]).

> **역사 메모** — `entries`는 Kotlin 1.8에서 실험적(`-language-version`/opt-in)으로 들어와 1.9에서 안정화됐다. 그 이전 십수 년간 `values()`가 유일한 순회 수단이었고, 성능에 민감한 코드는 흔히 `values()`의 결과를 `companion object`에 한 번 캐시해 두는 우회를 썼다. `entries`는 그 관용적 우회를 언어가 흡수한 결과다. Java의 enum에는 여전히 `values()`만 있고 `entries`는 Kotlin 고유다.

### 6.3 valueOf — 문자열에서 상수로, 그리고 그 예외

이름 문자열로부터 상수를 얻으려면 컴파일러가 생성하는 정적 `valueOf`를 쓴다.

```kotlin
enum class Color { RED, GREEN, BLUE }

fun main() {
    val c = Color.valueOf("GREEN")   // 정확히 일치해야 함(대소문자 구분)
    println(c)                        // => GREEN

    // 존재하지 않는 이름 → 예외
    try {
        Color.valueOf("green")        // 소문자 → 불일치
    } catch (e: IllegalArgumentException) {
        println("실패: ${e.message}")  // => 실패: No enum constant ...Color.green
    }
}
```

`valueOf`는 **대소문자를 정확히 구분**하며, 일치하는 상수가 없으면 `IllegalArgumentException`을 던진다([[41 - 예외와 Nothing]]). 사용자 입력이나 외부 데이터에서 온 문자열을 그대로 `valueOf`에 넘기면 예상치 못한 예외로 프로그램이 죽을 수 있다. 안전하게 하려면 예외를 잡거나, 표준 라이브러리의 `entries.find`로 널을 반환하게 만들거나, `runCatching`을 쓴다.

```kotlin
// 안전한 조회: 없으면 예외 대신 null
fun colorOrNull(name: String): Color? =
    Color.entries.find { it.name == name }

// 또는 대소문자 무시 매칭
fun colorIgnoreCase(name: String): Color? =
    Color.entries.find { it.name.equals(name, ignoreCase = true) }

fun main() {
    println(colorOrNull("green"))       // => null   (예외 없음)
    println(colorIgnoreCase("green"))   // => GREEN
}
```

수학적으로 `valueOf`는 부분 함수(partial function)다: $\text{valueOf}: \text{String} \rightharpoonup E$ 로, 정의역의 일부(정확한 상수 이름 문자열)에서만 값을 가지고 나머지에서는 예외로 발산한다. `entries.find { ... }`로 감싸면 이를 전 함수 $\text{String} \to E?$(널 가능 반환)로 바꿔 호출자가 실패를 값으로 다루게 할 수 있다.

### 6.4 Comparable과 compareTo — ordinal이 정하는 순서

enum은 `Comparable<E>`을 구현하며(상위 `Enum`이 구현), `compareTo`는 **`ordinal` 차이**로 정의된다. 따라서 enum 값의 `<`, `>`, 정렬은 전부 선언 순서를 따른다.

```kotlin
enum class Size { SMALL, MEDIUM, LARGE, XLARGE }

fun main() {
    println(Size.SMALL < Size.LARGE)     // => true   (0 < 2)
    val shuffled = listOf(Size.LARGE, Size.SMALL, Size.MEDIUM)
    println(shuffled.sorted())           // => [SMALL, MEDIUM, LARGE]  (ordinal 순)
    println(maxOf(Size.SMALL, Size.XLARGE))  // => XLARGE
}
```

이는 강력하지만 다시 **선언 순서에 의미를 싣는** 결정이다. `Size`처럼 순서 자체가 자연스러운 의미(작음→큼)를 가질 때는 정당하다. 그러나 순서에 의미가 없는 enum(예: `Color`)에서 `<` 비교나 `sorted()`를 쓰는 것은 우연한 선언 순서를 의미로 오독하는 것이다. `compareTo`를 재정의할 수는 없다(`Enum.compareTo`는 final). 다른 순서 기준이 필요하면 `sortedBy { it.someProperty }`처럼 명시적 키로 정렬하라.

> **명세 정밀** — enum의 `equals`·`hashCode`·`compareTo`는 모두 상위 `Enum`에서 오며 재정의 불가(final)다. `equals`는 참조 동일성(`===`)과 일치하고, `hashCode`는 각 상수마다 안정적이며(단, JVM에서 `Enum.hashCode`는 아이덴티티 해시 기반이라 **실행 세션 간에는 값이 다를 수 있다** — 그래서 enum의 `hashCode`를 영속 데이터로 저장하면 안 된다), `compareTo`는 `ordinal` 차이다. 이 불변식들 덕에 enum은 `HashSet`/`HashMap`/`TreeSet`의 키로 안전하게 쓸 수 있다.

JVM 백엔드에는 enum 전용 고성능 컬렉션인 `EnumSet`·`EnumMap`(비트벡터·배열 기반)이 있어, enum 키 컬렉션이 성능에 민감하면 이들을 Java 상호운용으로 활용할 수 있다. 이는 JVM 표준 라이브러리 기능이며 Kotlin 언어 자체의 것은 아니다.

---

## 7. reified 헬퍼 — enumValues와 enumValueOf

### 7.1 문제: 제네릭 함수 안에서 enum 상수 목록을 얻기

`Color.entries`나 `Color.valueOf(...)`는 enum 타입을 **정적으로 알 때만** 쓸 수 있다. 그런데 "임의의 enum 타입 `T`에 대해 동작하는" 제네릭 함수를 쓰고 싶을 때가 있다. 예컨대 "타입 `T`의 모든 상수를 출력하는 함수". 하지만 [[34 - 제네릭 2 - 타입 소거와 reified와 바운드]]에서 볼 타입 소거(M19) 때문에, 일반 제네릭 함수는 런타임에 `T`가 무엇인지 모른다.

```kotlin
// 이렇게는 안 된다 — T의 실제 타입을 런타임에 모름(타입 소거)
fun <T : Enum<T>> printAllNaive() {
    // T.entries  ← 불가: T가 소거되어 어떤 enum인지 모름
}
```

### 7.2 해법: reified inline 표준 함수

표준 라이브러리는 이 문제를 위해 두 개의 **reified inline** 함수를 제공한다([[15 - 인라인 함수와 reified]], M18 참조).

- **`enumValues<T>(): Array<T>`** — 타입 `T`의 모든 상수를 배열로.
- **`enumValueOf<T>(name: String): T`** — 이름으로 상수를 조회(없으면 예외).
- **`enumEntries<T>(): EnumEntries<T>`** (1.9+) — `entries`의 제네릭 버전(읽기 전용 List).

```kotlin
enum class Direction { NORTH, EAST, SOUTH, WEST }
enum class Color { RED, GREEN, BLUE }

// reified 덕분에 T의 실제 enum 타입을 컴파일 타임에 인라이닝
inline fun <reified T : Enum<T>> printAll() {
    for (constant in enumEntries<T>()) {
        println("${constant.ordinal}: ${constant.name}")
    }
}

inline fun <reified T : Enum<T>> parse(name: String): T = enumValueOf<T>(name)

fun main() {
    printAll<Direction>()
    // => 0: NORTH ... 3: WEST
    printAll<Color>()
    // => 0: RED ... 2: BLUE
    val d: Direction = parse("SOUTH")
    println(d)   // => SOUTH
}
```

핵심은 **`reified`와 `inline`의 결합**이다. `printAll`이 `inline`이고 `T`가 `reified`이므로, `printAll<Direction>()` 호출부에서 컴파일러가 `T`를 `Direction`으로 실체화해 인라이닝한다. 그 결과 함수 본문 안에서 `enumEntries<T>()`가 마치 `Direction.entries`인 것처럼 동작한다. 상한 `T : Enum<T>`(재귀 제네릭, [[34 - 제네릭 2 - 타입 소거와 reified와 바운드]])는 `T`가 반드시 enum 타입이도록 강제해, enum이 아닌 타입에 이 함수를 부르는 것을 컴파일 타임에 막는다.

이 헬퍼들은 "enum 종류를 파라미터로 받는" 일반화된 유틸리티 — 예를 들어 문자열 설정값을 임의 enum으로 파싱하는 범용 파서 — 를 타입 안전하게 짜는 열쇠다. 다만 `reified`는 최상위 타입만 실체화하며(M18), enum은 제네릭 인자를 갖지 않으므로 이 제약이 enum 헬퍼에서는 문제되지 않는다.

> **성능 주의** — `enumValues<T>()`는 내부적으로 결국 `values()` 계열을 부르므로 배열을 복사한다. 반복 호출이 잦다면 `enumEntries<T>()`(캐시된 읽기 전용 List)를 쓰는 편이 낫다. reified 함수는 인라인되므로 호출 자체의 오버헤드는 없지만, 호출 지점마다 함수 본문이 복제되어 코드 크기가 늘 수 있다는 inline의 일반적 비용(M17)은 여기에도 적용된다.

---

## 8. when과 완전성 — enum이 빛나는 자리 (M12 참조)

### 8.1 exhaustive when — else가 필요 없는 이유

enum의 가장 실용적인 강점은 `when`과의 결합이다. [[17 - 표현식으로서의 제어 흐름 - if와 when]]에서 못박았듯 `when`은 switch가 아니라 *표현식*이고 완전성(exhaustiveness) 검사를 받는다(M12). enum은 가능한 값의 집합이 컴파일 타임에 완전히 알려져 있으므로, 모든 상수를 다루는 `when`은 컴파일러가 "완전하다"고 판단하고 `else` 가지를 요구하지 않는다.

```kotlin
enum class Signal { RED, YELLOW, GREEN }

fun action(s: Signal): String = when (s) {   // 표현식으로 사용
    Signal.RED    -> "정지"
    Signal.YELLOW -> "주의"
    Signal.GREEN  -> "진행"
    // else 불필요 — 세 상수를 모두 덮었으므로 완전함
}

fun main() {
    println(action(Signal.YELLOW))   // => 주의
}
```

`when`을 **값을 만드는 표현식**으로 쓰면(위처럼 `= when`), 컴파일러는 완전성을 *강제*한다. 모든 상수를 덮으면 `else` 없이 통과하고, 하나라도 빠지면 컴파일 에러가 난다.

```kotlin
fun brokenAction(s: Signal): String = when (s) {
    Signal.RED    -> "정지"
    Signal.GREEN  -> "진행"
    // YELLOW를 빠뜨림!
}
// 컴파일 에러: 'when' expression must be exhaustive,
//              add necessary 'YELLOW' branch or 'else' branch instead
```

이 완전성 검사가 enum(그리고 [[30 - sealed 클래스와 대수적 데이터 타입]])을 상태 기계·명령 디스패치 같은 코드에서 강력하게 만든다.

### 8.2 새 상수를 추가하면 — 컴파일러가 빠진 곳을 알려준다

완전성 검사의 진짜 가치는 **enum이 변할 때** 드러난다. 새 상수를 추가하면, 그 상수를 아직 다루지 않는 모든 exhaustive `when`이 컴파일 에러(또는 최소한 경고)를 낸다. 컴파일러가 "여기, 여기, 여기서 새 상수를 처리하라"고 정확히 짚어 주는 것이다.

```kotlin
enum class Signal { RED, YELLOW, GREEN, FLASHING }   // FLASHING 추가!

fun action(s: Signal): String = when (s) {   // 표현식 when
    Signal.RED    -> "정지"
    Signal.YELLOW -> "주의"
    Signal.GREEN  -> "진행"
    // 컴파일 에러: FLASHING 가지가 없어 완전하지 않음
}
```

이것이 "잊힌 케이스"를 컴파일 타임에 잡는 안전망이다. `int` 상수 + `switch`였다면 이런 검사는 불가능했다. enum + exhaustive `when`은 "닫힌 집합의 모든 경우를 빠짐없이 처리했음"을 타입 시스템이 증명하게 만든다.

### 8.3 함정: else를 붙이면 안전망이 사라진다

바로 여기에 흔한 실수가 숨어 있다. "혹시 모르니" 습관적으로 `else` 가지를 붙이면, 완전성 검사가 무력화된다. `else`가 있으면 어떤 상수를 빠뜨려도 `else`가 받아 버리므로, 컴파일러는 "완전하다"고 판단하고 새 상수를 추가해도 침묵한다.

```kotlin
// 안티패턴: 방어적 else가 안전망을 없앤다
fun action(s: Signal): String = when (s) {
    Signal.RED    -> "정지"
    Signal.YELLOW -> "주의"
    Signal.GREEN  -> "진행"
    else          -> "알 수 없음"   // ← FLASHING 추가돼도 조용히 여기로 감
}
// FLASHING이 추가돼도 컴파일 에러 없음 → "알 수 없음"으로 잘못 처리, 아무도 모름
```

`FLASHING`을 추가해도 이 코드는 컴파일되고, `FLASHING`은 조용히 "알 수 없음"으로 처리된다. 버그가 런타임까지 숨는다. **원칙: enum(과 sealed)에 대한 `when`에서는 모든 상수를 명시적으로 나열하고 `else`를 피하라.** 정말로 "나머지 전부 동일 처리"가 의도라면 `else`를 쓰되, 그 순간 미래의 새 상수에 대한 컴파일러 안전망을 포기한다는 것을 의식적으로 선택하는 것이어야 한다.

> **명세 정밀** — `when`이 *표현식*(값을 만드는 위치)이면 완전성은 컴파일 에러로 강제된다 — 이는 모든 버전에서 확고하다(M12). `when`이 *문(statement)* 으로 쓰이면(값을 안 만드는 위치) 역사적으로 완전성 미달이 경고였으나, 최신 Kotlin은 enum/sealed 대상 `when` 문에 대해서도 완전성을 점점 강하게 요구하는 방향으로 조여 왔다(버전에 따라 경고→에러). 안전한 습관은 "문이든 표현식이든 enum `when`은 완전하게 나열하라"이다. 완전성을 명시적으로 강제하고 싶으면 문 대신 대입/반환처럼 표현식 위치로 만들면 확실하다.

### 8.4 when의 대상 없는 형태와 enum 조합

`when`은 대상 없이(subject 없이) 불리언 조건들의 분기로도 쓸 수 있고, enum 프로퍼티와 조합해 가독성 있는 분기를 만든다. 다만 대상 없는 `when`은 완전성 검사를 받지 않으므로(임의 조건이라 컴파일러가 완전성을 알 수 없다) 이 경우 `else`가 필요하다.

```kotlin
enum class Move { ROCK, PAPER, SCISSORS }

fun beats(a: Move, b: Move): Boolean = when {
    a == Move.ROCK     && b == Move.SCISSORS -> true
    a == Move.PAPER    && b == Move.ROCK     -> true
    a == Move.SCISSORS && b == Move.PAPER    -> true
    else -> false   // 대상 없는 when이라 else 필수
}
```

대상을 명시한 `when (a)` 형태라야 enum 완전성 검사의 혜택을 받는다. 두 스타일의 차이를 인식하고, enum 하나를 분해할 때는 대상 있는 `when`을 선택하는 것이 안전성 면에서 유리하다.

---

## 9. enum vs sealed — 닫힌 집합의 두 얼굴 (M23 참조)

### 9.1 근본적 차이: 인스턴스의 수와 모양

enum과 sealed 계층([[30 - sealed 클래스와 대수적 데이터 타입]])은 둘 다 "닫힌 집합"을 표현하고 둘 다 exhaustive `when`의 혜택을 받는다. 그래서 초심자는 언제 무엇을 써야 할지 헷갈린다. 결정적 차이는 하나다.

- **enum**: 각 경우가 **정확히 하나의 값(싱글턴 인스턴스)** 이고, 모든 경우가 **같은 모양(shape)** — 같은 프로퍼티 집합 — 을 공유한다. `Color.RED`는 딱 하나뿐이다.
- **sealed**: 각 경우가 **여러 인스턴스를 가질 수 있고**, 경우마다 **다른 모양(다른 프로퍼티)** 을 가질 수 있다. `Circle(radius=2.0)`과 `Circle(radius=3.0)`은 둘 다 `Circle`이지만 다른 값이다.

```kotlin
// enum: 각 경우 = 단일 값, 같은 모양
enum class CardSuit { HEARTS, DIAMONDS, CLUBS, SPADES }
// HEARTS는 온 세상에 하나. 무늬마다 데이터 모양이 같다.

// sealed: 각 경우 = 여러 인스턴스 가능, 다른 모양
sealed interface Shape {
    data class Circle(val radius: Double) : Shape       // 반지름 하나
    data class Rectangle(val w: Double, val h: Double) : Shape  // 폭·높이 둘
    data object Point : Shape                            // 데이터 없음
}
// Circle(2.0), Circle(3.0), Rectangle(1.0, 5.0) ... 무한히 많은 값
// 그러나 "가능한 종류"는 셋으로 닫혀 있다
```

`CardSuit`은 무늬가 넷이고 각 무늬는 유일하다 — enum이 완벽하다. `Shape`는 종류가 셋으로 닫혀 있지만 각 종류가 서로 다른 필드를 가지고 무한히 많은 인스턴스를 낳는다 — 이건 enum으로 표현할 수 없고 sealed가 답이다.

### 9.2 판단 기준을 표로

| 기준 | enum class | sealed class/interface |
|------|-----------|------------------------|
| 각 경우의 인스턴스 수 | 정확히 1 (싱글턴) | 여러 개 가능 |
| 경우별 데이터 모양 | 동일 (같은 프로퍼티) | 서로 다를 수 있음 |
| 경우별 상태(가변) | 부적절(전역 싱글턴) | 인스턴스마다 독립 상태 |
| 표준 멤버 | `name`/`ordinal`/`entries`/`valueOf` | 없음(직접 정의) |
| 순회 | `entries`로 전체 순회 가능 | 자동 순회 없음(직접 구현) |
| `when` 완전성 | ✓ | ✓ |
| 이름으로 조회 | `valueOf`/`entries.find` | 직접 구현 |
| 전형적 용도 | 방위·요일·상태 코드·플래그 | 표현식 트리·결과 타입·이벤트·상태 기계 |

### 9.3 경계 사례와 하이브리드

두 도구의 경계는 종종 미묘하다. "상태 기계"를 생각해 보자. 상태들이 데이터를 갖지 않고 순수한 라벨이라면 enum이 낫다. 그러나 어떤 상태가 데이터를 실어야 한다면(예: `Failed(reason: String)`, `Loaded(data: List<Item>)`) enum으로는 표현할 수 없고 sealed가 필요하다.

```kotlin
// 데이터 없는 순수 상태 → enum이 적합
enum class ConnectionState { DISCONNECTED, CONNECTING, CONNECTED }

// 상태마다 다른 데이터 → sealed가 필요
sealed interface LoadState<out T> {
    data object Loading : LoadState<Nothing>            // Nothing (M11, 05장 연결)
    data class Success<T>(val data: T) : LoadState<T>   // 데이터 실음
    data class Failure(val error: String) : LoadState<Nothing>
}
```

`LoadState.Failure`가 오류 메시지를 실어야 하는 순간 enum은 후보에서 탈락한다 — enum 상수는 컴파일 타임 고정 값이라 "실패마다 다른 메시지"를 담을 수 없기 때문이다(enum 상수에 `var`를 두는 것은 전역 가변 상태라 답이 아니다). 반대로 상태가 순수 라벨이고 순회·이름 조회가 필요하면 enum이 더 간결하고 `entries`·`valueOf` 같은 기본 도구를 공짜로 준다.

또한 sealed 계층의 "데이터 없는 여러 종류를 하나로 묶은 그룹"에는 `data object`(2.x)가 enum 상수처럼 쓰이기도 한다. 반대로 enum이 인터페이스를 구현하고 상수별 본문으로 다형성을 흉내 내면 sealed에 가까워 보인다. 실무의 결정 규칙은 이렇다: **"경우마다 붙는 데이터가 인스턴스별로 달라야 하는가?"** 그렇다면 sealed, 아니면 enum. 그리고 "전체 상수를 순회하거나 이름/코드로 조회할 일이 잦은가?"가 그렇다면 enum 쪽으로 저울이 기운다. 봉인의 "모듈+패키지 제한"이라는 규칙(M23)과 대수적 데이터 타입으로서의 깊은 이론은 [[30 - sealed 클래스와 대수적 데이터 타입]]이 마저 다룬다.

> **흔한 오해** — "enum은 sealed의 하위 호환 버전이다." 아니다. 둘은 표현력이 겹치지만 서로를 포함하지 않는다. enum은 "각 경우가 유일한 명명된 값"에 특화되어 순회·조회·자연 순서 같은 배터리를 기본 탑재하고, sealed는 "각 경우가 다른 모양의 여러 값"에 특화되어 대수적 데이터 타입을 이룬다. 잘 설계된 코드는 둘을 상황에 맞게 갈아 끼운다.

---

이 장이 답하기로 한 질문들을 한 줄기로 회수하자. enum 상수는 이름 붙은 정수가 아니라 그 enum 타입의 유일한 싱글턴 인스턴스이며(JVM 백엔드에서는 `static final` 필드로 태어나 `java.lang.Enum`을 상속한다), 그렇기에 `===`가 항상 안전하고 생성자·프로퍼티·메서드·인터페이스를 온전히 가질 수 있는 진짜 클래스다. 상수별 본문은 컴파일러가 익명 서브클래스로 번역하기에 본문 있는 상수의 `::class`는 enum 타입 자체와 다를 수 있고, `name`·`ordinal`은 상위 `Enum`이 주지만 `ordinal`은 선언 순서의 우연한 부산물이라 저장·비교의 근거로 삼으면 시한폭탄이 된다. `values()`는 매번 방어적 복사를 하는 가변 배열을 주고, 2.x의 `entries`는 캐시된 읽기 전용 List로 그 비용과 가변성을 함께 없앴다. `enumValues`/`enumValueOf`/`enumEntries`는 reified inline으로 타입 소거를 우회해 제네릭 컨텍스트에서 enum 상수를 손에 넣게 하고, exhaustive `when`은 enum의 닫힌 집합성을 활용해 "모든 경우를 처리했음"을 컴파일 타임에 증명하되 `else`를 붙이는 순간 그 안전망을 스스로 걷어낸다(M12). 그리고 "각 경우가 유일한 명명된 값이냐, 아니면 다른 모양의 여러 값이냐"라는 물음이 enum과 sealed(M23, 30장)를 가른다. enum은 "고정된, 이름 있는, 유한한 상수 집합"이라는 소박한 문제에 대한, 그러나 객체지향의 여러 장치가 정밀하게 응축된 답이다.

## 핵심 요약

- **enum 상수는 이름 붙은 정수가 아니라 그 enum 타입의 유일한 싱글턴 인스턴스다.** 그래서 `==`와 `===`가 항상 일치하며, 참조 동일성 비교가 언제나 안전하고 의미 있다(M24, M48 대조).
- **enum은 진짜 클래스다.** 주 생성자와 상수별 인자, 프로퍼티, 일반·추상 메서드, 인터페이스 구현, companion object를 가질 수 있다. 단 다른 클래스를 상속할 수 없고(이미 `Enum` 상속), `final`이라 확장 불가하며 제네릭 타입 파라미터를 못 갖는다.
- **상수별 본문은 컴파일러가 익명 서브클래스로 번역한다.** JVM 백엔드에서 본문 있는 상수의 런타임 클래스는 `Enum$1` 같은 익명 서브클래스라, 그 상수의 `::class`는 enum 타입 자체와 다를 수 있다 — 리플렉션·직렬화에서 함정.
- **enum 상수는 클래스 로드 시 선언 순서대로 한꺼번에 초기화된다.** 상수 하나만 접근해도 모든 상수의 생성자와 `init` 블록이 그때 전부 실행된다(M46, 21장). 상수 목록 뒤에 멤버가 오면 세미콜론이 필수다(Kotlin에서 세미콜론이 거의 유일하게 필수인 자리).
- **`ordinal`은 상수의 정체성이 아니라 선언 위치의 우연한 부산물이다.** `ordinal`이나 `hashCode`를 저장·전송·비교의 근거로 삼으면 상수 순서 변경이 곧 데이터 손상이 된다. 영속화가 필요하면 명시적 코드 프로퍼티나 `name`을 써라.
- **`values()`는 호출마다 방어적 복사를 하는 가변 배열을 주고, 2.x의 `entries`는 캐시된 읽기 전용 `List`를 준다.** `entries`는 할당이 없고 컬렉션 API 전체를 쓸 수 있어 2.x의 권장 관용구다. `entries`가 읽기 전용이라는 것은 컬렉션 뷰의 성질이지만 enum 상수 집합 자체가 컴파일 타임 고정이라 실질적으로 완전 불변이다(M26 대조).
- **`valueOf`는 대소문자를 정확히 구분하는 부분 함수이고, 불일치 시 `IllegalArgumentException`을 던진다.** 외부 입력을 안전히 다루려면 `entries.find { ... }`로 감싸 널 반환으로 전 함수화하라.
- **enum의 `equals`·`hashCode`·`compareTo`는 상위 `Enum`이 주며 재정의 불가(final)이고, `compareTo`는 `ordinal` 차이로 정의된다.** 정렬·`<` 비교는 선언 순서를 따르므로, 순서에 의미가 없는 enum에서 비교·정렬을 쓰면 우연한 순서를 의미로 오독하는 것이다.
- **`enumValues<T>()`/`enumValueOf<T>()`/`enumEntries<T>()`는 reified inline으로 타입 소거를 우회한다.** 상한 `T : Enum<T>`가 enum이 아닌 타입 사용을 컴파일 타임에 막아, 임의 enum에 대해 동작하는 타입 안전한 제네릭 유틸리티를 가능하게 한다(M18, M19 참조).
- **enum + exhaustive `when`은 "모든 경우를 처리했음"을 컴파일 타임에 증명한다(M12).** 상수를 추가하면 그것을 아직 안 다룬 표현식 `when`이 컴파일 에러를 낸다. 그러나 `else`를 붙이면 이 안전망이 사라지고 새 상수가 조용히 오처리되므로, enum `when`에서는 `else`를 피하고 모든 상수를 명시하라.
- **enum과 sealed의 경계는 "각 경우가 유일한 명명된 값이냐, 다른 모양의 여러 값이냐"다(M23).** 경우마다 실을 데이터가 인스턴스별로 달라야 하면 sealed, 순수 명명 상수이고 순회·조회가 잦으면 enum. 둘은 표현력이 겹치되 서로를 포함하지 않는 상보적 도구다.

## 연결 노트

- [[27 - 데이터 클래스]] — enum과 함께 "값을 담는 클래스"의 한 축. sealed의 곱타입 case로 자주 결합되고, `data object`는 enum 상수와 유사한 자리를 차지한다.
- [[28 - object와 companion - 싱글턴과 동반 객체]] — enum 상수가 싱글턴 인스턴스라는 이 장의 토대(M24). enum의 companion object로 팩토리·안정 코드 매핑을 구현한다.
- [[17 - 표현식으로서의 제어 흐름 - if와 when]] — `when`의 완전성 검사(M12)를 정본으로 소유. enum이 그 완전성을 어떻게 충족·활용하는지를 이 장이 이어받는다.
- [[30 - sealed 클래스와 대수적 데이터 타입]] — enum의 형제. 닫힌 집합을 "다른 모양의 여러 값"으로 표현하는 sealed의 규칙(M23)과 대수적 데이터 타입 이론이 이 장의 9절을 이어간다.
- [[15 - 인라인 함수와 reified]] — `enumValues`/`enumValueOf`가 딛고 선 reified inline의 원리(M18)를 소유. 타입 소거 우회의 기계.
- [[34 - 제네릭 2 - 타입 소거와 reified와 바운드]] — `T : Enum<T>` 재귀 제네릭 상한과 타입 소거(M19)의 정본. enum 헬퍼가 왜 inline이어야 하는지의 근거.
- [[10 - 불리언과 동등성과 동일성]] — enum에서 `==`와 `===`가 일치하는 이유(참조 동일성 기반 `equals`). 박싱 캐시의 예측 불가(M48)와 대조된다.
- [[21 - 클래스와 생성자]] — enum 상수 각각이 주 생성자·`init` 블록을 통해 초기화되는 순서 규칙(M46)이 여기서 상수별로 적용된다.
- [[25 - 인터페이스]] — enum이 상속 대신 인터페이스로 능력을 얻는 방식. `fun interface`·다형성과의 결합.
- [[38 - 컬렉션 - 읽기 전용과 가변]] — `entries`가 반환하는 읽기 전용 `List`의 성질(M26). 읽기 전용과 불변의 구분을 enum의 실질적 불변성과 대조.
