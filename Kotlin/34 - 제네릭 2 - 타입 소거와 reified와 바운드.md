---
title: 제네릭 2 - 타입 소거와 reified와 바운드
date: 2026-07-13
tags: [kotlin, generics, erasure, reified, bounds, 학습노트]
---

[[33 - 제네릭 1 - 타입 파라미터와 변성]]에서는 제네릭을 *컴파일 타임의 계약*으로 다뤘다. `out`/`in`이 대입 가능성을 어떻게 좁히고 넓히는지, 왜 `List<Dog>`가 `List<Animal>`의 하위 타입이 될 수 있는지를 살펴봤는데, 그 논증은 모두 **컴파일러가 보는 범위** 안에서만 성립하는 이야기였다. 이 장에서는 컴파일이 끝나고 프로그램이 실행되는 런타임으로 내려가서 묻는다. 컴파일러가 공들여 추적한 타입 인자는 실행 파일 안에 남아 있는가?

JVM 백엔드에서 답은 분명하다. **대부분 사라진다.** `List<String>`의 `String`, `Map<Int, Dog>`의 `Int`와 `Dog`, 함수의 타입 파라미터 `T`는 컴파일이 끝나면 지워지고, 바이트코드에는 뼈대만 남는다. 이것을 **타입 소거(type erasure)** 라고 한다. 따라서 "Kotlin 제네릭은 런타임에 자기 타입을 안다"는 믿음은 틀렸다.

이 장에서는 소거가 *왜* 일어나는지, 소거 때문에 *어디에서* 막히는지, 그리고 Kotlin이 `reified`라는 우회로로 그 제약을 *어떻게 국소적으로* 푸는지 살펴본다. 또한 `reified`로도 "최상위 타입만" 되살릴 수 있다는 한계([[15 - 인라인 함수와 reified]]에서 자세히 다룬다)도 함께 본다.

변성의 안전성 규칙은 33장에서, `inline`의 의미론 전반(비지역 반환, `noinline`/`crossinline`, 코드 팽창 비용)은 15장에서 다루므로, 여기서는 `reified`를 *소거를 우회하는 메커니즘*으로만 다룬다.

배열의 기본 성질(별개 타입, 원시 배열)은 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]와 [[38 - 컬렉션 - 읽기 전용과 가변]]에서, 리플렉션 전반(`KClass`, `KType`, `kotlin-reflect`)은 [[44 - 애노테이션과 리플렉션]]에서 다루고, 이 장에서는 *배열의 공변성과 소거가 어떻게 맞물리는지*만 본다.

이 장은 소거가 무엇인지 밝히는 데서 시작한다(1절). 이어서 소거가 남기는 흔적과 제약을 살펴보고(2~3절), `reified`가 그 제약을 국소적으로 푸는 원리와 한계를 본다(4~5절). 그다음 타입 파라미터에 *계약*을 부여하는 상한(bound)을 다루고(6~8절), 마지막으로 소거의 문제가 가장 크게 드러나는 배열의 공변 함정(9절)과 소거를 전제로 쓰는 실전 도구(10절)를 정리한다.

---

## 1. 타입 소거란 무엇인가: 컴파일러가 지우는 것

### 1.1 "런타임에 타입을 안다"는 오해 바로잡기

먼저 흔한 오해부터 살펴보자. 많은 개발자가 "Kotlin은 타입 시스템이 강력하니까 실행 중에도 `List<String>`인지 `List<Int>`인지 알 것"이라고 생각한다. **이 생각은 틀렸다.** JVM 백엔드에서 컴파일된 코드는 이 둘을 전혀 구별하지 못한다. 런타임에는 둘 다 그냥 `List`이다.

```kotlin
fun main() {
    val strings: List<String> = listOf("dog", "cat")
    val ints: List<Int> = listOf(1, 2, 3)

    // 런타임 클래스를 물어보면 —
    println(strings.javaClass)  // => class java.util.Arrays$ArrayList
    println(ints.javaClass)     // => class java.util.Arrays$ArrayList

    println(strings.javaClass == ints.javaClass)  // => true
}
```

두 리스트의 런타임 클래스는 **같다.** `<String>`도 `<Int>`도 어디에도 남아 있지 않다. 컴파일러는 소스 코드에서 이 정보를 완벽하게 추적했지만, 바이트코드를 만들 때 지워 버렸다. 이것이 소거가 일어난다는 핵심 증거이다.

> [!note] 명세 기준
> 소거는 "Kotlin 언어의 성질"이 아니라 **타깃 플랫폼의 런타임 모델**에서 비롯된다. JVM은 (자바 5에서 제네릭을 도입한 이래) 하위 호환을 위해 제네릭을 소거 방식으로 구현했고, Kotlin이 JVM을 타깃으로 삼으면 그 모델을 따를 수밖에 없다. 그래서 이 장에서 설명하는 "소거"는 원칙적으로 **JVM 백엔드**가 기준이다.
>
> Kotlin/Native·JS·Wasm도 기본적으로는 런타임에 일반 제네릭 타입 인자를 보존하지 않지만, 구체적인 표현 방식과 예외는 백엔드마다 다르다. 확실한 것은 하나이다. 소스에서 `reified`가 아닌 타입 파라미터는 *어떤 백엔드에서도* 런타임에 신뢰할 수 있는 타입 정보를 남긴다고 가정해서는 안 된다.

### 1.2 소거의 정의: 무엇이 무엇으로 바뀌는가

타입 소거의 규칙은 개념적으로 두 가지이다.

**첫째, 타입 인자(type argument)는 지워진다.** `List<String>`, `Map<Int, Dog>`, `Box<Circle>`에서 꺾쇠 안의 내용은 모두 사라지고 **원시 타입(raw type)** 만 남는다. `List<String>`은 `List`가 되고, `Map<Int, Dog>`는 `Map`이 된다.

**둘째, 타입 파라미터(type parameter) `T` 자체는 자신의 상한(upper bound)으로 바뀐다.** 상한이 없으면 기본 상한은 `Any?`이고, JVM에서는 이것이 `java.lang.Object`가 된다.

```kotlin
class Box<T>(val value: T)                        // T의 상한: Any? → Object
class NumberBox<T : Number>(val value: T)         // T의 상한: Number
```

두 클래스의 필드 시그니처를 바이트코드 관점의 의사 코드로 보면 다음과 같다.

```text
class Box {
    Object value;         // T가 Object로 소거됨 (상한이 Any?)
}
class NumberBox {
    Number value;         // T가 Number로 소거됨 (상한이 Number)
}
```

여기서 중요한 점을 알 수 있다. **상한은 컴파일 타임에만 쓰이는 문서가 아니라, 소거한 뒤 실제로 남는 타입을 결정한다.** `<T : Number>`라고 쓰면 필드가 `Object`가 아니라 `Number`로 소거되므로, 바이트코드 수준에서 무검사 캐스트 없이 `Number`의 메서드(`toInt()`, `toDouble()` 등)를 바로 호출할 수 있다. 6절에서 다룰 상한의 두 가지 역할(성능 측면과 의미 측면)은 여기서 나온다.

### 1.3 왜 소거인가: 역사와 대안

> [!info] 역사 메모
> 자바가 제네릭을 도입한 2004년(자바 5)에 설계자들에게는 두 가지 선택지가 있었다. 하나는 C#/.NET처럼 런타임에 타입 인자를 보존하는 **실체화(reification)** 이고, 다른 하나는 컴파일 타임에만 검사하고 런타임에는 지우는 **소거**이다. 자바는 후자를 택했다.
>
> 이유는 단 하나, **하위 호환**이었다. 제네릭 이전에 작성된 방대한 `List`(원시 타입) 코드가 제네릭 도입 후에도 그대로 동작해야 했고, 이를 위해 `List<String>`을 런타임에 `List`와 똑같이 취급해야 했다.
>
> Kotlin은 JVM 위에서 자바와 상호운용하므로 이 결정을 물려받았다. C#은 CLR을 처음부터 실체화 제네릭으로 설계했기 때문에 `typeof(List<string>)`이 온전히 동작하지만, Kotlin/JVM에서는 그럴 수 없다. Kotlin의 `reified`는 이 부족함을 *inline이라는 국소적인 수단*으로 메우려는 영리한 타협이다(4절).

소거의 대가는 분명하다. 런타임 타입 정보를 잃는다. 그 대신 얻는 것은 다음과 같다.

1. 하위 호환
2. 런타임 오버헤드가 없음(모든 `List<T>`가 같은 바이트코드를 공유하며, C#처럼 타입마다 특수화된 코드를 생성하지 않음)
3. 메모리 절약

Kotlin은 이 trade-off를 물려받았지만, 뒤에서 볼 `reified`와 `typeOf`로 필요한 곳에서만 실체화를 흉내 낸다.

---

## 2. 소거의 증거: 무엇을 잃고 무엇이 남는가

### 2.1 남는 것: 원시 타입과 상한

소거가 *모든* 정보를 지우지는 않는다. 무엇이 남는지 정확히 알아야 한다.

```kotlin
fun inspect(x: Any) {
    println(x is List<*>)      // OK — 원시 List 검사는 가능
    println(x is Map<*, *>)    // OK
    println(x is Collection<*>) // OK
}

fun main() {
    inspect(listOf(1, 2))          // => true, false, true
    inspect(mapOf("a" to 1))       // => false, true, false  (Map은 Collection이 아니다)
}
```

`is List<*>`는 컴파일된다. 이 검사는 **원시 `List`인지 여부만** 묻기 때문이다. `<*>`(스타 프로젝션)는 "타입 인자가 무엇이든 상관없다"는 뜻이고, 소거된 런타임에는 애초에 원시 `List`밖에 없으므로 이 검사는 온전히 의미가 있다. 런타임은 "이 객체가 `List` 인터페이스를 구현하는가?"라는 질문에 `instanceof`로 답할 수 있다.

### 2.2 잃는 것: 타입 인자 검사

반면 다음 코드는 **컴파일조차 되지 않는다.**

```kotlin
fun countStrings(x: Any): Boolean {
    return x is List<String>   // 컴파일 에러: Cannot check for instance of erased type: List<String>
}
```

컴파일러는 이 검사를 정직하게 거부한다. `List<String>` 검사는 런타임에 *불가능*하므로, 조용히 통과시켰다가 나중에 오작동하게 두지 않고 아예 막는 것이다. 런타임에는 `<String>`이 존재하지 않으니, `x`가 `List<Int>`이든 `List<Dog>`이든 이 검사는 늘 같은 질문(원시 `List`인가)에만 답할 수 있다. 그 답은 프로그래머가 의도한 "원소가 String인가?"와 다르다. Kotlin은 이런 잘못된 안심을 처음부터 허용하지 않는다.

> [!warning] 흔한 오해
> "`is List<String>`이 안 되는 것은 Kotlin이 미숙하기 때문"이라고 생각하기 쉽지만 그렇지 않다. 이것은 JVM 소거 모델에서 **원리적으로 판정할 수 없는** 검사이다. 자바의 `x instanceof List<String>`도 똑같이 컴파일 에러이다(자바는 `List<?>`만 허용한다). Kotlin은 오히려 더 명확한 에러 메시지("erased type")로 이유를 알려 준다. 이것은 결함이 아니라 소거를 정직하게 드러내는 것이다.

### 2.3 무검사 캐스트: 소거가 조용히 넘어가는 지점

가장 위험한 지점은 **캐스트**이다. 검사(`is`)는 막히지만 캐스트(`as`)는 *경고만 내고 통과*한다.

```kotlin
@Suppress("UNCHECKED_CAST")
fun trust(x: Any): List<String> {
    return x as List<String>   // 경고: Unchecked cast: Any to List<String>
}

fun main() {
    val ints: List<Int> = listOf(1, 2, 3)
    val strings = trust(ints)  // 런타임에 아무 예외 없음! (소거 때문)

    // 그러나 실제로 원소를 String으로 쓰는 순간 —
    val first: String = strings[0]  // ClassCastException: Integer cannot be cast to String
}
```

여기서 소거의 위험한 면이 드러난다. `x as List<String>`은 런타임에 **원시 `List`인지만 검사**한다. `ints`는 실제로 `List`이므로 캐스트는 성공하고, 원소가 정말 String인지는 검사하지 *않는다*. 그래서 `strings`는 겉으로는 `List<String>` 타입이지만 안에는 `Int`가 들어 있어, 언제든 예외를 일으킬 수 있는 상태이다.

예외는 원소를 실제로 꺼내 String으로 쓰는 순간, 즉 `strings[0]`이 컴파일러가 삽입한 `checkcast String`을 만나는 지점에서 발생한다. 컴파일러가 `UNCHECKED_CAST` 경고를 내는 이유가 바로 이것이다. "이 캐스트의 타입 인자 부분은 검증할 수 없으니, 프로그래머가 책임져야 한다"는 뜻이다.

> [!caution] 성능 주의
> 이런 무검사 캐스트가 항상 나쁜 것은 아니다. 예를 들어 `Any`로 들어온 값이 문맥상 반드시 `List<String>`이라고 프로그래머가 *외부 지식*으로 확신할 수 있을 때(직렬화 경계 등)는 `@Suppress("UNCHECKED_CAST")`로 명시적으로 책임을 지는 것이 정당한 패턴이다. 다만 이 확신이 틀리면 예외가 캐스트에서 멀리 떨어진 곳에서 발생해 디버깅이 어려워진다는 점을 기억해야 한다.

### 2.4 소거와 `javaClass` / `::class`의 차이

리플렉션 도구도 소거된 정보는 되살리지 못한다.

```kotlin
fun main() {
    val box = listOf("a", "b")
    val klass = box::class            // KClass<out List<String>> — 정적 타입은 안다
    println(klass)                     // => class java.util.Arrays$ArrayList — 원시 List 구현체 (타입 인자 없음)

    // KClass는 원시 클래스만 안다. 타입 인자 정보는 없음.
    println(klass.typeParameters)      // => 클래스가 선언한 타입 파라미터 목록(선언부), 실제 인자 아님
}
```

`someObject::class`는 객체의 **런타임 클래스**(원시 타입)를 돌려준다. 이 값은 소거의 영향을 받으므로, `listOf("a")::class`와 `listOf(1)::class`는 같은 `KClass`를 반환할 수 있다.

반면 뒤에서 볼 `typeOf<List<String>>()`은 **정적 타입 표현**을 돌려주며 타입 인자를 온전히 담는다(10절). "객체가 무엇인가"(소거됨)와 "타입 표현이 무엇인가"(컴파일 타임에 보존됨)를 혼동하는 것은 리플렉션을 처음 다룰 때 흔히 빠지는 함정이다.

---

## 3. 소거가 만드는 제약: 실전에서 부딪히는 곳

소거는 추상적인 이론에 그치지 않고, 구체적인 컴파일 에러로 개발자의 작업을 막는다. 그런 제약을 하나씩 살펴보자.

### 3.1 제약 1: 타입 파라미터에 대한 `is` / `as`

일반(비-reified) 함수 안에서는 타입 파라미터를 검사할 수 없다.

```kotlin
fun <T> firstOfType(items: List<Any>): T? {
    for (item in items) {
        if (item is T) return item   // 컴파일 에러: Cannot check for instance of erased type: T
    }
    return null
}
```

`T`는 런타임에 존재하지 않으므로 `item is T`는 의미가 없고, 컴파일러는 이 코드를 막는다. 이 제약을 넘는 유일한 방법이 4절의 `reified`이다. 미리 보면 다음과 같다.

```kotlin
inline fun <reified T> firstOfType(items: List<Any>): T? {
    for (item in items) {
        if (item is T) return item   // OK! reified라서 T가 실체화됨
    }
    return null
}

fun main() {
    val mixed: List<Any> = listOf(1, "two", 3.0, "four")
    println(firstOfType<String>(mixed))  // => two
}
```

### 3.2 제약 2: 제네릭 배열 생성

JVM에서는 타입 파라미터로 `Array<T>`를 직접 생성할 수 없다.

```kotlin
fun <T> makeArray(size: Int, value: T): Array<T> {
    return Array(size) { value }   // 컴파일 에러: Cannot use 'T' as reified type parameter. Use a class instead.
}
```

배열은 소거되지 않는 특이한 타입이다. JVM 배열은 런타임에 자신의 **원소 타입**을 안다(`Object[]`와 `String[]`은 런타임에 구별되며, 이 성질이 9절의 공변 함정을 만든다). 그래서 배열을 만들려면 원소의 실제 클래스를 알아야 하는데, 소거된 `T`로는 그 클래스를 알 수 없다. Kotlin의 `Array(size) { init }` 생성자가 `inline fun <reified T>`로 선언된 이유가 바로 이것이다.

우회하는 방법은 세 가지이다.

```kotlin
// 방법 1: reified inline
inline fun <reified T> makeArray(size: Int, value: T): Array<T> =
    Array(size) { value }   // OK — reified T로 실제 배열 타입 결정

// 방법 2: arrayOfNulls (nullable 원소, Object[] 기반이라 reified 불필요)
fun <T> makeNullableArray(size: Int): Array<T?> =
    arrayOfNulls<T>(size)   // OK — 런타임엔 Object[]를 만들고 캐스트

// 방법 3: 클래스 객체를 인자로 받아 java.lang.reflect.Array 사용 (구식)
```

> [!note] 명세 기준
> `arrayOfNulls<T>(size)`는 reified가 아닌데 왜 동작할까? 이 함수는 내부적으로 `Object[]`(즉 `Array<Any?>`)를 만들고 `Array<T?>`로 캐스트하기 때문이다. 원소 타입을 몰라도 `Object[]`는 만들 수 있으므로 소거의 제약에 걸리지 않는다. 대신 반환된 배열의 *런타임 원소 타입*은 `String[]`이 아니라 `Object[]`라는 미묘한 차이가 생긴다.
>
> 반면 `Array(n){}`나 `emptyArray<T>()`는 `reified`라서 실제 `String[]`을 만든다. 이 차이는 그 배열을 `String[]`을 기대하는 자바 API에 다시 넘길 때 드러난다.

### 3.3 제약 3: 플랫폼 선언 충돌 (시그니처 겹침)

가장 당황스러운 제약이다. 논리적으로 다른 두 함수가 소거 후에 같은 시그니처가 되어 충돌한다.

```kotlin
fun printAll(items: List<String>) { items.forEach(::println) }
fun printAll(items: List<Int>) { items.forEach(::println) }
// 컴파일 에러: Platform declaration clash:
//   The following declarations have the same JVM signature (printAll(Ljava/util/List;)V)
```

두 함수는 소스에서는 서로 다른 오버로드이지만, 소거 후에는 둘 다 `printAll(List): void`가 되어 JVM 수준에서 이름과 시그니처가 겹친다. JVM은 이 둘을 구별할 수 없으므로 컴파일러가 미리 막는다.

해결책은 `@JvmName`으로 바이트코드상의 이름을 다르게 지정하는 것이다.

```kotlin
fun printAll(items: List<String>) { items.forEach(::println) }

@JvmName("printAllInts")
fun printAll(items: List<Int>) { items.forEach(::println) }
// OK — 소스에선 같은 이름, 바이트코드에선 printAll / printAllInts
```

> [!warning] 흔한 오해
> "오버로드는 인자 타입만 다르면 되는 것 아닌가?"라고 생각하기 쉽지만, 컴파일된 선언은 **소거 후 JVM 시그니처**가 겹치면 충돌한다. `List<String>`과 `List<Int>`는 소스에서는 다르지만 JVM 시그니처(`(Ljava/util/List;)`)는 같다. 이것은 자바에도 있는 고전적인 함정이며, Kotlin은 `@JvmName`이라는 명시적인 해결 수단을 제공한다.
>
> 참고로 이 충돌은 *제네릭 타입 인자만* 다른 경우에만 발생한다. `fun f(x: String)`과 `fun f(x: Int)`는 소거해도 시그니처가 다르므로 문제가 없다.

### 3.4 제약 4: 제네릭 예외 타입

`catch` 블록은 런타임에 예외의 실제 타입을 검사(`instanceof`)해야 하므로, 소거되는 타입 파라미터로는 예외를 잡을 수 없다. Kotlin은 이 문제를 처음부터 막는다. **Throwable의 하위 클래스는 타입 파라미터를 가질 수 없다.**

```kotlin
class MyException<T>(val payload: T) : Exception()
// 컴파일 에러: Type parameter of a subclass of 'Throwable' can not be reified
//              (Generic subclass of Throwable is not allowed)
```

이것이 허용된다면 `catch (e: MyException<String>)` 같은 코드를 쓰고 싶어질 것이다. 하지만 소거 때문에 런타임에는 `<String>`을 검사할 수 없으므로, 이 코드는 늘 `catch (e: MyException<*>)`처럼만 동작하게 된다. Kotlin은 이런 잘못된 안심을 막기 위해 제네릭 Throwable 자체를 금지한다.

```text
소거의 벽 요약
┌─────────────────────────────┬──────────────────────────────┐
│ 하려는 것                   │ 결과                          │
├─────────────────────────────┼──────────────────────────────┤
│ x is List<String>           │ 컴파일 에러 (erased type)     │
│ x is List<*>                │ OK (원시 검사)                │
│ item is T (일반 함수)       │ 컴파일 에러                   │
│ item is T (reified)         │ OK                            │
│ Array<T>(n){...} (일반)     │ 컴파일 에러                   │
│ x as List<String>           │ 경고 후 통과 (무검사)         │
│ 오버로드 f(List<A>)/f(List<B>)│ 플랫폼 선언 충돌            │
│ class E<T> : Throwable       │ 컴파일 에러                   │
└─────────────────────────────┴──────────────────────────────┘
```

---

## 4. reified: inline이 열어 주는 우회로

### 4.1 원리: 실체화가 아니라 인라이닝이다

`reified`는 흔히 "런타임 타입을 얻는 마법"으로 오해받지만, 실제로는 훨씬 평범하고 영리한 메커니즘이다. 핵심은 다음과 같다. **`reified`는 런타임에 아무것도 되살리지 않는다. 컴파일 타임에 타입을 *복사해 넣을* 뿐이다.**

`reified`는 반드시 `inline` 함수에서만 쓸 수 있다. `inline`의 의미론 전반은 15장에서 다루므로 여기서는 핵심만 짚는다. inline 함수는 호출 지점에 **본문이 그대로 복사**된다. 그리고 복사하는 시점에 컴파일러는 타입 인자가 *구체적으로 무엇인지 알고 있다*. 그래서 본문의 `T`를 그 구체 타입으로 치환할 수 있다.

```kotlin
inline fun <reified T> Any?.isInstanceOf(): Boolean = this is T

fun main() {
    val x: Any = "hello"
    println(x.isInstanceOf<String>())  // => true
    println(x.isInstanceOf<Int>())     // => false
}
```

`x.isInstanceOf<String>()` 호출은 컴파일 타임에 다음처럼 **펼쳐진다(인라인된다)**.

```kotlin
// 컴파일러가 호출 지점에 실제로 넣는 코드(의사 코드):
val result = x is String   // T가 String으로 치환됨 — 진짜 String이 박힘!
```

즉 런타임에는 `is String`이라는 *구체적인 검사*만 남는다. `T`라는 추상 타입은 컴파일 타임에 사라지고, 각 호출 지점에 구체 타입이 들어간다. 이것이 소거를 "우회"하는 실제 방식이다. 소거를 되돌리는 것이 아니라, 소거가 일어나기 *전에* 타입을 코드에 고정해 두는 것이다.

```text
reified의 작동 흐름
─────────────────────────────────────────────
소스:   inline fun <reified T> f(): X = ... T ...
호출:   f<String>()   f<Dog>()
          │              │
          ▼  (인라인 + T 치환)
호출지점1: ... String ...      호출지점2: ... Dog ...
          │              │
          ▼  (컴파일)     ▼
바이트코드: String이 박힌 코드   Dog가 박힌 코드
─────────────────────────────────────────────
결론: T는 런타임에 존재하지 않는다. 각 호출지점에
      구체 타입이 이미 박혀 있을 뿐이다.
```

### 4.2 reified로 할 수 있는 것

인라인 치환 덕분에 본문에서 `T`를 구체 타입처럼 쓸 수 있다.

```kotlin
inline fun <reified T> demo() {
    // 1. is 검사
    val ok = "x" is Any && Any::class.isInstance("x")

    // 2. T::class — 클래스 리터럴
    val klass = T::class
    println(klass.simpleName)

    // 3. 캐스트도 (여전히 무검사 부분은 남지만 T 최상위는 검사됨)
    // 4. 배열 생성
    val arr = arrayOfNulls<T>(3)
}

inline fun <reified T> tag(): String = T::class.simpleName ?: "?"

fun main() {
    println(tag<Int>())     // => Int
    println(tag<String>())  // => String
    println(tag<List<*>>()) // => List
}
```

이렇게 하면 3절에서 본 소거의 제약(타입 파라미터에 대한 `is`, 배열 생성)과 `T::class`를 쓸 수 없다는 제약을 국소적으로 넘을 수 있다.

### 4.3 표준 라이브러리의 reified

Kotlin 표준 라이브러리는 곳곳에서 reified를 쓴다. 이런 함수들을 알아 두면 reified를 쓰는 관용적인 방식을 익힐 수 있다.

```kotlin
// filterIsInstance — 컬렉션에서 특정 타입만 골라내기
val mixed: List<Any> = listOf(1, "a", 2, "b", 3.0)
val onlyStrings: List<String> = mixed.filterIsInstance<String>()
println(onlyStrings)  // => [a, b]

// emptyArray, arrayOf 계열
val empty: Array<Dog> = emptyArray()   // emptyArray는 reified

// enumValues (29장 연결)
enum class Suit { HEART, SPADE, DIAMOND, CLUB }
inline fun <reified E : Enum<E>> allNames(): List<String> =
    enumValues<E>().map { it.name }
println(allNames<Suit>())  // => [HEART, SPADE, DIAMOND, CLUB]

class Dog
```

대표적인 예가 `filterIsInstance<String>()`이다. 이 함수는 내부적으로 각 원소에 `is String`(치환된 검사)을 적용해 필터링한다. reified가 없었다면 이 함수는 `filterIsInstance(klass: Class<T>)`처럼 클래스 객체를 인자로 받아야 했을 것이다. reified는 그런 보일러플레이트를 없애 준다.

> [!caution] 성능 주의
> reified는 공짜가 아니다. inline 함수이므로 **호출할 때마다 본문이 복사**된다. `T`를 여러 타입으로 바꿔 가며 자주 호출하면 그만큼 바이트코드가 늘어난다. 작은 유틸리티(`filterIsInstance`, `is` 한 줄)는 문제가 없지만, 본문이 큰 함수를 reified inline으로 만들어 수십 곳에서 다양한 타입으로 호출하면 코드 크기가 눈에 띄게 늘 수 있다.
>
> 따라서 reified가 *꼭 필요한 최소한의 본문*만 inline으로 두고 나머지는 일반 함수에 맡기는 것이 관용적인 방식이다(자세한 내용은 15장 인라인 함수와 reified).

---

## 5. reified의 한계: 최상위 타입만 실체화된다

### 5.1 얕은 실체화: 바깥 타입만 되살아난다

reified로 소거 문제를 완전히 해결한 것처럼 보이지만, 결정적인 한계가 있다. **reified는 타입의 *최상위(top-level)* 만 실체화한다. 그 안의 타입 인자는 여전히 소거된 상태이다.**

```kotlin
inline fun <reified T> describe(): String = T::class.simpleName ?: "?"

fun main() {
    println(describe<String>())         // => String
    println(describe<List<String>>())   // => List   (String은 어디에?)
    println(describe<List<Int>>())      // => List   (Int도 사라짐!)
    println(describe<Map<String, Dog>>()) // => Map   (String도 Dog도 없음)
}

class Dog
```

`describe<List<String>>()`가 실체화하는 것은 바깥의 `List`뿐이다. 그 안의 `String`은 여전히 소거되어 접근할 수 없다. `T::class`는 `KClass`(원시 클래스)를 돌려주는데, 원시 클래스는 정의상 타입 인자를 담지 않기 때문이다.

따라서 reified로 "완전한 제네릭 타입"을 얻을 수 있다는 생각은 오해이다. `reified T`에 `List<String>`이 들어가더라도, 코드가 실제로 얻는 것은 `List`의 클래스 객체뿐이다.

> [!note] 명세 기준
> 왜 최상위 타입만 실체화될까? 4절에서 봤듯이 reified는 "호출 지점에 타입을 넣는" 방식이다. `T::class` 같은 표현은 그 자리에 들어간 타입의 **런타임 클래스 객체**로 컴파일되는데, JVM의 클래스 객체(`Class`/`KClass`)는 원시 타입 하나만 가리킬 뿐 타입 인자를 담을 자리가 없다.
>
> `List<String>`을 넣어도 이를 표현할 런타임 객체는 `List.class`밖에 없으므로, 안쪽의 `String`은 표현할 곳이 없어 사라진다. 즉 최상위 타입만 실체화되는 한계는 reified의 결함이 아니라 *런타임 클래스 표현의 한계*이다.

### 5.2 `is List<String>`은 reified로도 안 된다

이 한계 때문에 reified 함수 안에서도 중첩된 타입 인자는 검사할 수 없다.

```kotlin
inline fun <reified T> check(x: Any): Boolean = x is T

fun main() {
    // T = List<String>로 호출하면?
    val ints: List<Int> = listOf(1, 2, 3)
    println(check<List<String>>(ints))  // => true (!!)  — 원소가 Int인데도 true
}
```

`check<List<String>>(ints)`를 인라인하면 `ints is List<String>`이 된다. 그런데 이 검사는 3절에서 컴파일 에러였다. 실제로 이 경우 컴파일러는 `ints is List`(원시 검사, `<*>`에 해당)로 번역하고 경고를 낸다. 결과적으로 원소 타입은 검사되지 않으므로, `Int` 리스트인데도 `true`가 나온다. reified를 쓰더라도 중첩된 타입 인자에 대한 소거의 제약은 그대로 남는다.

### 5.3 유일한 예외: `typeOf`와 `KType` (미리 보기)

그렇다면 `List<String>`의 `String`을 되살릴 방법은 정말 없을까? 있다. 단, `KClass`가 아니라 `KType`을 써야 한다. `typeOf<T>()`(reified inline)는 **컴파일 타임 타입 표현 전체**를 `KType` 객체로 만들어 준다.

```kotlin
import kotlin.reflect.typeOf

inline fun <reified T> fullType() = typeOf<T>()

fun main() {
    val t = fullType<List<String>>()
    println(t)                    // => kotlin.collections.List<kotlin.String>
    println(t.arguments)          // => [kotlin.String] — 타입 인자가 살아있다!
}
```

여기서는 `String`이 되살아났다. 비결은 원시 클래스를 나타내는 `KClass`와 달리 `KType`은 타입 인자를 담는 자료구조라는 데 있다. `typeOf`는 reified로 `List<String>` 전체를 호출 지점에 넣고, 그것을 `KType`이라는 *더 많은 정보를 담는 객체*로 재구성한다. 이것은 소거의 예외라기보다 **컴파일 타임 정보를 런타임 객체로 옮겨 담은 것**이다.

자세한 내용은 10절과 44장(애노테이션과 리플렉션)에서 다룬다. 여기서 기억할 것은 하나이다. reified의 최상위 타입 한계를 넘고 싶다면 `T::class`가 아니라 `typeOf<T>()`를 써야 한다.

---

## 6. 상한(Upper Bound): 타입 파라미터에 계약을 부여하기

### 6.1 기본 상한은 `Any?`이다

지금까지 `<T>`는 "아무 타입"을 뜻했다. 정확히 말하면 **상한이 `Any?`인 타입 파라미터**이다. 33장(제네릭 1)에서 본 것처럼, 상한을 명시하지 않으면 기본 상한은 `Any?`(nullable 포함!)이므로 `T`에 null을 넣을 수 있다.

```kotlin
fun <T> identity(x: T): T = x

fun main() {
    println(identity<String?>(null))  // => null  — T = String?, null 허용
    println(identity(42))             // => 42
}
```

`<T>`는 `<T : Any?>`의 축약이다. `T`가 null을 받지 못하게 하려면 상한을 `Any`(non-null)로 명시해야 한다.

```kotlin
fun <T : Any> requireNonNull(x: T): T = x

fun main() {
    requireNonNull("ok")            // OK
    // requireNonNull<String?>(null)  // 컴파일 에러: Type argument is not within bounds. Expected: Any
}
```

이 구분은 5절의 reified나 변성과 관계없이 **모든** 타입 파라미터에 적용되는 기본 규칙이다.

### 6.2 상한이 부여하는 계약: 멤버 접근

상한의 진짜 역할은 null 허용 여부보다 **본문에서 어떤 멤버를 쓸 수 있는지**를 결정하는 데 있다. 상한이 `Any?`이면 `T`에 대해 `Any?`의 멤버(`equals`, `hashCode`, `toString`, 그리고 nullable이므로 `?.`)밖에 쓸 수 없다. 상한을 좁히면 그 타입의 멤버를 쓸 수 있게 된다.

```kotlin
// 상한 없음: Number의 메서드를 못 씀
fun <T> badSum(a: T, b: T): Double =
    a.toDouble() + b.toDouble()  // 컴파일 에러: Unresolved reference: toDouble

// 상한 Number: toDouble() 사용 가능
fun <T : Number> goodSum(a: T, b: T): Double =
    a.toDouble() + b.toDouble()  // OK — T가 Number이므로 toDouble() 보장

fun main() {
    println(goodSum(3, 4))       // => 7.0
    println(goodSum(1.5, 2.5))   // => 4.0
}
```

여기서 1.2절에서 본 내용이 다시 등장한다. `<T : Number>`는 소거 후에도 `T`를 `Number`로 남기므로, 바이트코드에서 무검사 캐스트 없이 `toDouble()`을 직접 호출할 수 있다. 즉 상한은 두 가지 역할을 한다.

1. 컴파일러에게 "이 멤버를 써도 된다"고 허가한다.
2. 소거 후 남는 실제 타입을 결정한다.

```text
상한과 소거의 관계
─────────────────────────────────────
소스           소거 후 T   본문에서 쓸 수 있는 멤버
<T>            Object      Any?의 멤버 (equals 등)
<T : Any>      Object      Any의 멤버 (non-null)
<T : Number>   Number      Number의 멤버 (toInt 등)
<T : Dog>      Dog         Dog의 멤버
─────────────────────────────────────
```

### 6.3 상한과 nullable: `T`와 `T?`의 미묘한 차이

상한이 `Any`(non-null)이더라도 함수 안에서 `T?`를 쓰면 nullable을 표현할 수 있다.

```kotlin
fun <T : Any> firstOrNull(list: List<T>): T? =
    if (list.isEmpty()) null else list[0]

fun main() {
    val names = listOf("a", "b")
    val first: String? = firstOrNull(names)  // T = String, 반환은 String?
    println(first)  // => a
}
```

`T : Any`는 "타입 인자로 non-null 타입만 허용한다"는 뜻이지, "T를 nullable로 쓸 수 없다"는 뜻이 아니다. `T?`는 언제나 `T`에 null 가능성을 더한 타입이다. *타입 인자에 대한 제약*(상한)과 *사용하는 자리의 nullable 표기*(`?`)는 헷갈리기 쉬우니 분명히 구분해 두자. 널 안전성 전반은 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]에서 다룬다.

---

## 7. 다중 상한과 `where` 절

### 7.1 꺾쇠 안에는 상한을 하나만 쓸 수 있다

타입 파라미터에 상한을 둘 이상 걸고 싶을 때가 있다. "T는 `Comparable`이면서 동시에 `Serializable`이어야 한다" 같은 요구가 그렇다. 그러나 꺾쇠 안의 문법(`<T : Bound>`)으로는 **상한을 하나만** 쓸 수 있다.

```kotlin
// 이건 안 됨 — 꺾쇠 안엔 상한 하나만
// fun <T : Comparable<T> : CharSequence> f(x: T)  // 문법 에러
```

상한이 둘 이상이면 `where` 절로 분리해 쓴다.

```kotlin
fun <T> copyWhenGreater(list: List<T>, threshold: T): List<String>
    where T : CharSequence,          // 상한 1: CharSequence
          T : Comparable<T> {        // 상한 2: Comparable<T>
    return list
        .filter { it > threshold }   // Comparable<T> 덕분에 > 사용 가능
        .map { it.toString() }        // CharSequence는 Any의 toString OK
}

fun main() {
    val words = listOf("apple", "banana", "cherry")
    println(copyWhenGreater(words, "b"))  // => [banana, cherry]
}
```

`where T : A, T : B`는 "T는 A이면서 B"라는 **교집합 제약**이다. 본문에서 `T`는 A의 멤버와 B의 멤버를 모두 쓸 수 있다. 위 예에서 `it > threshold`는 `Comparable`에서, 문자열 처리는 `CharSequence`에서 온다.

### 7.2 소거 관점: 첫 번째 상한이 남는다

> [!note] 명세 기준 (JVM 백엔드)
> 다중 상한 `where T : A, T : B`에서 `T`는 **첫 번째로 나열된 클래스/인터페이스 상한**으로 소거된다(일반적으로 첫 번째 상한). 나머지 상한은 컴파일 타임 검사와, 필요할 때 삽입되는 무검사 캐스트/브리지로만 반영된다. 그래서 다중 상한 중 *실제로 자주 호출할 멤버가 있는 타입*을 첫 번째 상한으로 두면 브리지 메서드를 줄일 수 있다.
>
> 다만 이것은 최적화 세부 사항이고, 정확한 동작은 컴파일러 구현에 따라 다를 수 있으므로 성능이 결정적으로 중요할 때만 신경 쓰면 된다.

### 7.3 다중 상한과 스마트 캐스트의 교집합 타입

다중 상한이 만드는 "A이면서 B"라는 개념은 [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]]과 6장(널 안전성)에서 본 **교집합 타입(intersection type)** 과 같은 계열이다. 스마트 캐스트가 `x is A && x is B` 이후 `x`를 "A와 B의 교집합"으로 보는 것과, `where T : A, T : B`가 T를 그 교집합으로 제약하는 것은 같은 논리이다.

둘 다 여러 계약을 동시에 만족하는 타입을 뜻한다. 이런 개념적 공통점은 Type Theory 시리즈 [[16 - 부분 타입과 변성]]의 교집합 타입 논의로 이어진다.

---

## 8. 재귀 제네릭: 자기 자신을 상한에 두기

### 8.1 `T : Comparable<T>`: F-바운드 다형성

가장 우아하지만 처음에는 어리둥절한 패턴이 **재귀 제네릭(recursive generic bound)**, 또는 **F-바운드 다형성(F-bounded polymorphism)** 이다. 이 패턴에서는 타입 파라미터가 자기 자신을 언급하는 상한을 가진다.

```kotlin
fun <T : Comparable<T>> max(a: T, b: T): T =
    if (a >= b) a else b

fun main() {
    println(max(3, 7))          // => 7
    println(max("apple", "kiwi")) // => kiwi
}
```

`<T : Comparable<T>>`를 풀어 읽으면 "T는 *자기 자신과* 비교할 수 있는 타입이어야 한다"는 뜻이다. `Int : Comparable<Int>`, `String : Comparable<String>`이므로 둘 다 통과한다. 이 제약이 없으면 `a >= b`(내부적으로 `a.compareTo(b)`)를 쓸 수 없다. `compareTo`가 *어떤 타입과* 비교하는지 컴파일러가 모르기 때문이다.

`T : Comparable<T>`는 그 "어떤 타입"을 `T` 자신으로 고정해, `a.compareTo(b)`에서 `b`가 반드시 `T`임을 보장한다. `<T : Comparable<*>>`처럼 느슨하게 쓰면 `a`와 `b`를 서로 비교할 수 있다는 보장이 사라진다.

### 8.2 왜 필요한가: 자기 참조 타입 계약

재귀 제네릭이 진가를 발휘하는 곳은 "메서드가 자기 타입을 반환하거나 받아야 하는" 계약이다. 빌더 패턴이나 자기 비교가 대표적이다.

```kotlin
// 자기 자신 타입을 반환하는 추상 빌더
abstract class Shape<SELF : Shape<SELF>> {
    protected abstract fun self(): SELF
    var name: String = ""

    fun named(n: String): SELF {
        name = n
        return self()   // SELF로 반환 → 하위 타입 체이닝 유지
    }
}

class Circle : Shape<Circle>() {
    var radius: Double = 0.0
    override fun self() = this
    fun withRadius(r: Double): Circle { radius = r; return this }
}

fun main() {
    val c = Circle()
        .named("wheel")      // 반환 타입이 Circle (SELF) — 다음에 Circle 메서드 가능!
        .withRadius(5.0)     // Circle의 메서드 계속 체이닝
    println("${c.name}: ${c.radius}")  // => wheel: 5.0
}
```

`named`는 `SELF`를 반환하므로, `Circle` 인스턴스에서 호출하면 `Circle`이 반환되어 `withRadius`로 체이닝을 이어 갈 수 있다. 재귀 제네릭이 없다면 `named`는 `Shape`를 반환하고, 그 뒤에는 `Circle` 전용 메서드를 부를 수 없다. 이것이 F-바운드 다형성을 실전에서 쓰는 이유이다.

> [!warning] 흔한 오해
> `<SELF : Shape<SELF>>`를 보면 "무한 재귀가 아닌가?"라고 걱정하게 된다. 그렇지 않다. 이것은 값의 재귀가 아니라 **타입 제약의 재귀**일 뿐이다. 컴파일러는 `Circle : Shape<Circle>`이 성립하는지 한 번 검사하고 끝내므로 무한 루프는 없다.
>
> 다만 이 패턴은 잘못 다루면 `Shape<WrongType>` 같은 실수를 허용하므로(하위 클래스가 자기 자신이 아닌 타입을 넣을 수 있음), Kotlin에서는 sealed 계층이나 다른 설계로 대체하는 편이 더 안전할 때가 많다.

### 8.3 다른 시리즈와의 연결: 변성과 재귀 바운드

재귀 제네릭은 변성(33장 제네릭 1)과 만나면 더 복잡해진다. `Comparable`이 `in T`(반공변)로 선언된 것이 왜 `T : Comparable<T>` 패턴을 안전하게 만드는지는 변성 이론에서 다룰 주제이다. 타입 이론에서 F-바운드 다형성의 일반론은 Type Theory 시리즈 16장(부분 타입과 변성)에서 더 깊이 다룬다.

---

## 9. 배열의 공변 함정: Java와 Kotlin 비교

### 9.1 배열은 소거되지 않는다: 그래서 위험하다

이 장 전체에 걸친 소거의 논리에서 **배열은 예외**이다. 3.2절에서 봤듯이 JVM 배열은 런타임에 자기 원소 타입을 안다(`String[]`과 `Integer[]`는 런타임에 구별된다). 이렇게 "실체화된 배열"이 제네릭의 소거와 어긋나면서 유명한 함정이 생긴다. 바로 **배열의 공변성(covariance)** 이다.

먼저 자바의 함정을 보자.

```java
// Java
Object[] objs = new String[3];   // 컴파일 통과! 배열은 공변 (String[] <: Object[])
objs[0] = 42;                    // 컴파일 통과! Object[]에 Integer 넣기
// 런타임: ArrayStoreException — 실제 배열은 String[]인데 Integer 저장 시도
```

자바는 배열을 **공변(covariant)** 으로 만들었다. 즉 `String[]`이 `Object[]`의 하위 타입이다. 그래서 `String[]`을 `Object[]`로 받아 `Integer`를 넣는 코드도 *컴파일은 통과*한다. 하지만 배열은 런타임에 실제 타입(`String[]`)을 알고 있으므로, 저장하는 시점에 `ArrayStoreException`이 발생한다. **컴파일 타임 안전을 포기하고 런타임 검사로 대신한 설계**이다.

### 9.2 Kotlin은 배열을 무공변으로 만들었다

Kotlin은 이 함정을 언어 차원에서 막는다. **`Array<T>`는 무공변(invariant)** 이다.

```kotlin
fun main() {
    val strings: Array<String> = arrayOf("a", "b", "c")
    // val objs: Array<Any> = strings   // 컴파일 에러: Type mismatch. Array<String> is not Array<Any>
}
```

`Array<String>`은 `Array<Any>`에 대입할 수 없다. Kotlin은 배열을 다른 제네릭 클래스와 마찬가지로 무공변으로 취급한다(33장에서 본 기본 무공변 원칙). 그래서 자바의 `ArrayStoreException` 같은 런타임 오류를 **컴파일 타임에** 걸러 낸다. Kotlin이 자바의 오래된 설계 실수를 의식적으로 바로잡은 대표적인 사례이다.

```text
배열 공변성 — Java vs Kotlin
──────────────────────────────────────────────
              Java              Kotlin
공변?         예 (String[]<:Object[])  아니오 (무공변)
Object[]=String[]  컴파일 OK        컴파일 에러
잘못된 저장   런타임 ArrayStoreException  애초에 대입 불가
안전성        런타임 검사           컴파일 타임 차단
──────────────────────────────────────────────
```

### 9.3 읽기 전용이 필요하면 `out`으로 명시

배열에서 *읽기만* 한다면(공변이 안전한 상황), 사용 지점 변성으로 명시할 수 있다.

```kotlin
fun printAll(items: Array<out Any>) {   // out — 읽기 전용 공변 프로젝션
    for (item in items) println(item)
}

fun main() {
    val strings: Array<String> = arrayOf("x", "y")
    printAll(strings)  // OK — Array<out Any>는 Array<String>을 받는다
}
```

`Array<out Any>`는 "Any의 하위 타입 배열이면 무엇이든 받되, 읽기만 한다"는 뜻이다(사용 지점 변성은 33장에서 다룬다). `out`이므로 이 배열에 값을 *쓰는* 연산(`items[0] = ...`)은 컴파일러가 막고, 따라서 자바의 `ArrayStoreException`이 원리적으로 발생할 수 없다. Kotlin은 공변이 안전한 경우(읽기)와 위험한 경우(쓰기)를 타입으로 분리해서, 자바가 구분하지 않은 부분을 정밀하게 다룬다.

### 9.4 원시 배열은 아예 다른 타입이다

한 가지 더 있다. `Array<Int>`(박싱된 `Integer[]`)와 `IntArray`(원시 `int[]`)는 **완전히 다른 타입**이다(자세한 내용은 38장 컬렉션과 8장 수 2에서 다룬다). 이 구분은 소거와 직접 관련은 없지만, 배열을 다룰 때 함께 기억해야 할 함정이다. `IntArray`는 제네릭이 아니라 전용 타입이므로 소거나 박싱 문제가 생기지 않는다.

---

## 10. 소거를 전제로 쓰는 실전 도구: class 리터럴, `KType`, 스타 프로젝션

### 10.1 `T::class`와 `typeOf<T>()`: 무엇을 되살리나

이 장의 결론을 실전 도구로 정리해 보자. 타입이 소거되는 환경에서 타입 정보를 다루는 도구는 크게 두 가지이다.

| 도구 | 얻는 것 | 타입 인자 보존 | 사용 조건 |
|------|---------|:---:|-----------|
| `obj::class` | 객체의 런타임 원시 클래스 (`KClass`) | 아니오 (소거됨) | 아무 객체 |
| `SomeType::class` | 지정 타입의 원시 클래스 (`KClass`) | 아니오 (최상위만) | 구체 타입 or reified T |
| `typeOf<T>()` | 컴파일 타임 타입 표현 (`KType`) | **예** (중첩까지) | reified inline 안 |

```kotlin
import kotlin.reflect.typeOf

inline fun <reified T> compareTools() {
    val fromClass = T::class          // KClass — 최상위만
    val fromTypeOf = typeOf<T>()      // KType — 전체
    println("class:  ${fromClass.simpleName}")
    println("typeOf: $fromTypeOf")
    println("args:   ${fromTypeOf.arguments}")
}

fun main() {
    compareTools<Map<String, List<Int>>>()
    // => class:  Map
    // => typeOf: kotlin.collections.Map<kotlin.String, kotlin.collections.List<kotlin.Int>>
    // => args:   [kotlin.String, kotlin.collections.List<kotlin.Int>]
}
```

`T::class`는 바깥의 `Map`만 돌려주지만, `typeOf<T>()`는 중첩된 `List<Int>`의 `Int`까지 포함해 `Map<String, List<Int>>` 전체를 되살린다. 다시 말하지만 이것은 소거의 예외가 아니라 **컴파일 타임 정보를 런타임으로 옮긴 것**이다. `typeOf`는 reified로 최상위 타입을 호출 지점에 넣고, 컴파일러가 알고 있던 나머지 인자들을 `KType` 그래프로 재구성해 담는다.

### 10.2 스타 프로젝션 `*`: 소거를 명시적으로 인정하기

`List<*>`의 `*`는 33장에서 스타 프로젝션으로 소개했다. 소거의 관점에서 다시 보면, `*`는 **"타입 인자가 무엇인지 모른다, 또는 상관없다"는 사실을 타입으로 명시**하는 장치이다. 그래서 런타임 검사와 잘 맞는다.

```kotlin
fun sizeOf(x: Any): Int = when (x) {
    is List<*> -> x.size       // OK — List<*>는 런타임에 검사 가능
    is Map<*, *> -> x.size
    is Set<*> -> x.size
    else -> -1
}

fun main() {
    println(sizeOf(listOf(1, 2, 3)))       // => 3
    println(sizeOf(mapOf("a" to 1)))       // => 1
    println(sizeOf("string"))              // => -1
}
```

`List<*>`에서 원소를 *읽으면* 무엇이 들어 있는지 모르므로 타입은 `Any?`가 된다. 원소를 *쓰는* 것은 안전한 타입을 특정할 수 없으므로 (null 외에는) 금지된다. 즉 스타 프로젝션은 소거된 런타임과 정확히 대응하는 컴파일 타임 개념이다. `is List<String>`은 안 되지만 `is List<*>`는 되는 이유가 여기에 있다.

### 10.3 `@UnsafeVariance`: 변성 검사를 예외적으로 끄는 장치

마지막으로 고급 도구를 하나 살펴보자. 표준 라이브러리의 `List`는 `List<out E>`(공변)로 선언되어 있다. 그런데 `contains(element: E)`는 `E`를 *인자*(in 위치)로 받는다. 공변 타입 파라미터를 in 위치에 쓰는 것은 원칙적으로 금지되어 있는데(33장의 변성 규칙), 어떻게 가능할까?

```kotlin
public interface List<out E> : Collection<E> {
    // 표준 라이브러리 실제 시그니처(개념):
    public operator fun contains(element: @UnsafeVariance E): Boolean
}
```

`@UnsafeVariance` 애노테이션은 컴파일러에게 "이 위치의 변성 검사는 내가 책임지고 끄겠다"고 선언한다. `contains`는 인자를 컬렉션에 *저장*하지 않고 *비교만* 하므로 실제로는 안전하다. 하지만 컴파일러의 기계적인 규칙은 이 사실을 알지 못하므로, 라이브러리 작성자가 직접 안전성을 보증하고 검사를 면제하는 것이다.

> [!warning] 성능 주의 / 흔한 오해
> `@UnsafeVariance`는 애플리케이션 코드에서 쓸 일이 거의 없고, 대부분의 경우 써서도 안 된다. 이것은 표준 라이브러리처럼 "변성 규칙이 지나치게 보수적이지만, 실제로 안전하다는 것을 작성자가 증명할 수 있는" 특수한 상황을 위한 예외 장치이다. 이름 그대로 "Unsafe"이므로, 잘못 쓰면 변성이 보장하던 타입 안전이 깨진다.
>
> 이 장에서 이 도구를 소개하는 이유는, 소거와 변성의 규칙이 "절대 법칙"이 아니라 "안전을 위한 보수적 근사"이고 작성자가 명시적으로 책임지는 예외 경로가 있다는 것을 알아야 언어를 깊이 이해할 수 있기 때문이다.

### 10.4 소거를 우회하는 실전 관용구 정리

소거된 환경에서 타입 정보가 필요할 때 쓸 수 있는 방법을 모으면 다음과 같다.

```kotlin
// 1. 런타임 타입 검사가 필요하면 → reified
inline fun <reified T> List<*>.onlyType(): List<T> = filterIsInstance<T>()

// 2. 클래스 객체가 필요하면 → reified + ::class
inline fun <reified T : Any> classOf(): kotlin.reflect.KClass<T> = T::class

// 3. 중첩 타입 인자까지 필요하면 → reified + typeOf
inline fun <reified T> typeToken() = kotlin.reflect.typeOf<T>()

// 4. 타입 인자를 몰라도 되면 → 스타 프로젝션
fun logSize(c: Collection<*>) = println(c.size)

// 5. 캐스트가 불가피하면 → @Suppress로 책임 명시
@Suppress("UNCHECKED_CAST")
fun <T> forceCast(x: Any?): T = x as T
```

이 다섯 가지가 소거를 전제로 할 때의 표준 도구이다. 핵심 원칙은 하나이다. **런타임 타입이 필요한 곳에는 컴파일 타임에 타입을 "넣어 두는" reified를 쓰고, 최상위 타입을 넘어 중첩 인자까지 필요하면 `typeOf`를, 타입 인자를 몰라도 되면 스타 프로젝션을 쓴다.** 그리고 어쩔 수 없이 무검사 캐스트를 할 때는 반드시 명시적으로 책임(`@Suppress`)을 표시한다.
