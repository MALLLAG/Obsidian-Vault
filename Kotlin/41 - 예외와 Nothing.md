---
title: 예외와 Nothing
date: 2026-07-13
tags: [kotlin, exceptions, nothing, result, control-flow, 학습노트]
---

앞의 [[40 - 시퀀스와 지연 평가]]에서는 "계산을 *언제* 실행할 것인가"를 다뤘다. 시퀀스는 원소 하나가 필요해질 때까지 연산을 미뤘고, 종단 연산이 파이프라인을 끌어당길 때 비로소 값이 흘렀다. 이것은 *정상 흐름*의 제어였다.

이 장은 반대쪽, 즉 계산이 **정상적으로 끝나지 못할 때** 언어가 무엇을 하는지를 다룬다. 함수가 값을 돌려주는 대신 스택을 되감으며 빠져나갈 때, 표현식이 결코 값을 만들지 못할 때, 제어가 코드의 물리적 순서를 벗어나 다른 곳으로 이동할 때가 모두 여기에 해당한다. 이런 비정상 종료의 문법과 의미론, 그리고 실제 구현 방식이 이 장의 주제다.

그 중심에는 낯선 개념이 두 개 있다. 하나는 타입 시스템의 가장 아래에 놓인 바닥 타입(bottom type) `Nothing`이고, 다른 하나는 이 바닥 타입을 자기 타입으로 갖는 `throw` 표현식이다. 두 개념은 [[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]]에서 처음 소개했고, 이 장에서는 `Nothing`이 *예외 흐름과 타입 추론을 어떻게 연결하는지*를 바이트코드 수준까지 살펴본다.

예를 들어 `val name = user.name ?: throw IllegalStateException()` 한 줄은 왜 타입 검사를 통과할까? `throw`의 타입이 `Nothing`이고, `Nothing`이 `String`의 하위 타입이기 때문이다. 이 장을 마치면 이 문장이 왜 당연히 통과하는지 이해하게 된다.

이 장에서는 `Nothing`, `throw`, `try`/`catch`/`finally`, `Result`, `use`처럼 *언어 코어가 직접 정의하는* 예외 장치를 다룬다.

널 안전성 연산자와 스마트 캐스트는 [[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]에서, `sealed` 계층으로 결과를 모델링하는 대수적 데이터 타입(ADT)은 [[30 - sealed 클래스와 대수적 데이터 타입]]에서, 코루틴 취소와 구조적 동시성의 예외 전파는 [[42 - suspend와 코루틴 - 언어 수준의 중단]]에서 다룬다.

순서는 `Nothing`(1절), `throw`와 예외 계층(2~3절), 검사 예외의 폐기(4절), `try` 표현식과 `finally`(5~6절), `use`와 `Result`(7~8절), 예외와 결과 타입의 설계 판단(9절), 코루틴 취소(10절)이다.

---

## 1. Nothing: 바닥 타입의 정체

### 1.1 타입 계층의 위와 아래

Kotlin의 타입 계층에는 양 끝이 있다. 가장 위에는 모든 것을 담는 `Any?`가 있고, 가장 아래에는 *아무것도* 담지 못하는 `Nothing`이 있다. 이 대칭 구조를 먼저 그림으로 확인하자.

```text
                    Any?          ← 최상위 타입 (nullable Any). 모든 타입의 상위타입.
                   /    \
                 Any    (null)    ← Any 는 non-null 최상위
                /  \
           Number  String  …      ← 사용자·표준 타입들
              |       |
             Int     …
              \      /
               \    /
              Nothing            ← 바닥 타입 (bottom). 모든 타입의 하위타입.
                                    인스턴스가 존재하지 않는다.
```

위쪽 끝의 `Any`는 "무엇이든 이 자리에 넣을 수 있다"는 뜻의 상한(*upper bound*)이다. 어떤 타입 `T`든 `T <: Any?`이고, non-null이면 `T <: Any`이다. 이 내용은 5장(타입 시스템의 지형)에서 다룬다.

아래쪽 끝의 `Nothing`은 그 반대다. 형식적으로 쓰면 다음과 같다.

$$\forall T.\quad \mathrm{Nothing} <: T$$

즉 `Nothing`은 **모든 타입의 하위 타입**이다. `Nothing`은 `Int`의 하위 타입이고, `String`의 하위 타입이며, `List<Animal>`의 하위 타입이고, 방금 정의한 클래스의 하위 타입이기도 하다.

타입 이론에서는 이런 타입을 바닥 타입(bottom type)이라 부르고 기호로 $\bot$을 쓴다. 자매 시리즈인 Type Theory의 [[01 - 타입이란 무엇인가]]에서 설명한 대로 부분 타입 순서(subtyping order)를 격자(lattice)로 보면, `Nothing`은 그 격자의 최소원(least element)이다.

> [!note] 명세 기준
> `Nothing`이 모든 타입의 하위 타입이라는 명제는 리스코프 치환 원칙(LSP)의 관점에서 자명하게 참이다. 하위 타입은 "상위 타입이 기대되는 모든 자리에 대신 놓을 수 있다"는 뜻인데, `Nothing`은 값이 하나도 없으므로 그런 자리에 실제로 놓일 값 자체가 존재하지 않는다. 치환해야 할 값이 없으니 규칙을 위반할 값도 없고, 따라서 공허하게(vacuously) 모든 제약을 만족한다. 바닥 타입을 *어느 타입 자리에도* 대입할 수 있는 논리적 근거가 여기에 있다.

### 1.2 인스턴스가 없다는 것의 의미

핵심 사실은 **`Nothing` 타입의 값은 존재할 수 없다**는 것이다. `Any`의 인스턴스는 무한히 많지만, `Nothing`의 인스턴스는 정확히 0개다. 생성자도 리터럴도 없고, `null`조차 `Nothing`이 아니다(`null`은 `Nothing?`에 속한다. 1.4절에서 설명한다).

```kotlin
val impossible: Nothing = TODO()   // 우변이 정상적으로 값을 돌려주는 순간은 오지 않는다
```

이 선언은 컴파일된다. 하지만 `impossible`에 실제로 값이 대입되는 순간은 오지 않는다. `TODO()`가 예외를 던지며 빠져나가기 때문이다. `Nothing` 타입의 지역 변수를 선언하는 것은 "이 지점 이후에는 도달할 수 없다"고 컴파일러에게 약속하는 것과 같다.

여기서 흔한 오해 하나를 바로잡자. **"`Nothing`은 `null`이거나 무의미한 무엇이다"** 라는 생각은 틀렸다. `Nothing`은 무의미하기는커녕 타입 시스템에서 가장 정밀한 도구 중 하나다. `Nothing`은 "값이 없음"을 나타내는 것이 아니다(그것은 `Unit`도 아니고 `void`도 아니다). `Nothing`은 **"이 표현식은 결코 정상적으로 값을 만들어 내지 못한다"는 사실 자체를 타입으로 인코딩**한다.

`Unit`이 "값을 하나 돌려주기는 하지만 그 값에 정보가 없다"는 뜻이라면, `Nothing`은 "돌려줄 값이 아예 없다. 실행이 이 지점을 정상적으로 통과하지 못한다"는 뜻이다. 이 차이를 표로 정리하면 다음과 같다.

| 타입 | 값의 개수 | 의미 | 예 |
|------|-----------|------|-----|
| `Unit` | 정확히 1개 (`Unit` 객체) | "값은 있으나 정보가 없음. 정상 복귀함." | `println()`의 반환 |
| `Nothing` | 0개 | "정상 복귀 자체가 없음." | `throw`, `TODO()`, 무한 루프 |
| `Nothing?` | 1개 (`null`) | "값은 `null` 하나뿐." | 항상 `null`인 변수의 타입 |
| Java `void` | (값 아님) | "돌려줄 것이 없는 반환 위치 표식." | Kotlin `Unit`이 이를 대체 |

### 1.3 Nothing의 쓰임새: 타입 추론을 이어 주는 역할

`Nothing`의 진짜 힘은 타입 추론에서 드러난다. Kotlin의 타입 추론은 여러 분기의 타입을 하나로 합칠 때 **최소 공통 상위 타입(least upper bound, LUB)** 을 찾는다. 어느 분기가 `Nothing`이면 그 분기는 LUB 계산에 아무 제약도 걸지 않는다. `Nothing`은 이미 상대편 타입의 하위 타입이므로, 합친 결과의 상한은 전적으로 반대편 분기가 결정한다.

$$\mathrm{LUB}(T,\ \mathrm{Nothing}) = T$$

이 성질은 엘비스 연산자와 `throw`를 함께 쓸 때 효과를 발휘한다.

```kotlin
class User(val name: String?)

fun greet(user: User): String {
    val name: String = user.name ?: throw IllegalArgumentException("이름 없음")
    //                 ^^^^^^^^^    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //                 String?      타입은 Nothing
    return "안녕, $name"
}
```

`user.name`의 타입은 `String?`이고, `?:`의 우변 `throw ...`의 타입은 `Nothing`이다. 엘비스 표현식 전체의 타입은 두 분기의 LUB인데, `String?`의 non-null 부분과 `Nothing`을 합치면 `String`이 된다(좌변의 값은 널이 아닐 때만 이 자리에 오므로 널 가능성이 사라진다). 결과적으로 `name`의 타입은 `String`으로 좁혀진다.

`throw`가 `Nothing`을 반환하므로 우변은 값을 만들지 않고, 그래서 좌변의 non-null 타입이 곧 전체 타입이 된다. 엘비스와 제어 흐름을 결합하는 이 관용구 자체는 6장(널 안전성)에서 설명했지만, *왜* 타입 검사를 통과하는지의 근거는 바로 `Nothing`의 LUB 성질이다.

표준 라이브러리의 `TODO`와 `error`도 이 성질을 그대로 이용한다.

```kotlin
public inline fun TODO(): Nothing = throw NotImplementedError()
public inline fun TODO(reason: String): Nothing = throw NotImplementedError("An operation is not implemented: $reason")
public inline fun error(message: Any): Nothing = throw IllegalStateException(message.toString())
```

세 함수 모두 반환 타입이 `Nothing`이다. 그래서 어떤 타입이 기대되는 자리에든 넣을 수 있다.

```kotlin
fun area(shape: Shape): Double = when (shape) {
    is Circle -> Math.PI * shape.r * shape.r
    is Rectangle -> shape.w * shape.h
    else -> error("알 수 없는 도형: $shape")   // Nothing 이므로 Double 자리에 OK
}

val config: Config = loadConfig() ?: TODO("설정 로더 미구현")  // Nothing 이 Config 자리에
```

`when`의 각 분기는 `Double`을 기대하는데, 마지막 분기의 `error(...)`는 `Nothing`을 돌려준다. `Nothing <: Double`이므로 문제가 없다. 만약 `error`가 `Unit`을 반환했다면 이 코드는 컴파일되지 않았을 것이다. `Unit`은 `Double`의 하위 타입이 아니기 때문이다. **`Nothing`은 "여기서 나가는 값이 없으니 어떤 타입 요구든 만족한 것으로 봐도 안전하다"는 컴파일러와의 계약이다.**

### 1.4 Nothing?와 null, 그리고 빈 컬렉션

`Nothing?`은 `Nothing`에 널 가능성을 더한 타입이다. `Nothing`의 인스턴스는 0개이고, 여기에 널 하나를 더하면 인스턴스는 정확히 1개, 즉 `null`뿐이다. 그래서 **`Nothing?`은 `null` 리터럴의 타입**이다.

```kotlin
val x = null            // 추론된 타입: Nothing?
val y: String? = null   // null 은 Nothing? 이고, Nothing? <: String? 이므로 대입 가능
```

이 사실은 표준 라이브러리 곳곳에서 실용적으로 쓰인다. 인자 없는 `emptyList()`가 대표적인 예다.

```kotlin
public fun <T> emptyList(): List<T> = EmptyList

internal object EmptyList : List<Nothing>, ... {
    override fun get(index: Int): Nothing = throw IndexOutOfBoundsException(...)
    // ...
}
```

빈 리스트의 원소 타입은 `Nothing`이다. 왜 그럴 수 있을까? 리스트가 비어 있으니 원소가 하나도 없고, 원소의 타입은 "결코 존재하지 않는 원소들의 타입", 즉 `Nothing`이면 충분하다. 그리고 `List<T>`는 원소 타입에 대해 공변(covariant, `List<out E>`)이므로 `List<Nothing>`은 임의의 `List<T>`의 하위 타입이다(변성은 [[33 - 제네릭 1 - 타입 파라미터와 변성]] 참조).

그래서 싱글턴 하나인 `EmptyList`를 `List<String>`으로도, `List<Animal>`으로도, `List<Int>`로도 안전하게 쓸 수 있다. `Nothing`이 없었다면 원소 타입마다 별도의 빈 리스트가 필요했을 것이다.

```kotlin
val strings: List<String> = emptyList()   // EmptyList : List<Nothing> <: List<String>
val animals: List<Animal> = emptyList()   // 같은 싱글턴 인스턴스를 재사용
```

> [!warning] 흔한 오해
> "빈 리스트니까 원소 타입은 무엇이든 상관없고 `Any?`로 두면 되지 않나?"라고 생각하기 쉽지만, 그렇게 할 수 없다. `List<Any?>`는 `List<String>`의 상위 타입이라 대입되지 않는다(공변이므로 방향이 반대다). 빈 컬렉션의 원소 타입은 *상한*이 아니라 *하한*이어야 하고, 그 하한이 바로 `Nothing`이다. 이렇게 "어디에나 대입할 수 있는 빈 값"을 표현하는 올바른 도구는 바닥 타입뿐이다.

### 1.5 Nothing과 도달 불가능성 분석

`Nothing` 타입의 표현식은 컴파일러의 제어 흐름 분석(control-flow analysis)에 "이 지점 이후에는 도달하지 않는다"고 알린다. 그래서 `Nothing`을 돌려주는 호출 뒤의 코드는 죽은 코드(dead code)로 취급되고, 이 사실이 여러 진단 결과를 바꾼다.

```kotlin
fun pick(flag: Boolean): Int {
    val result: Int
    if (flag) {
        result = 10
    } else {
        error("flag 는 항상 true 여야 함")   // Nothing
        // 여기 이후는 도달 불가 → result 미할당 경로가 아님
    }
    return result   // 컴파일러: 모든 경로에서 result 가 할당됨 (else 는 탈출하므로)
}
```

`error(...)`가 `Nothing`이므로 `else` 분기는 정상적으로 끝나지 않는다. 그래서 컴파일러는 `result`가 "도달 가능한 모든 경로에서 정확히 한 번 초기화된다"는 것을 증명할 수 있고, `val`의 초기화 강제 규칙([[11 - 변수와 초기화 - val과 const와 lateinit]])도 통과시킨다.

만약 `error`가 `Unit`을 돌려줬다면 컴파일러는 "`else`에서 `result`가 초기화되지 않은 채 `return`에 도달할 수 있다"고 판단해 에러를 냈을 것이다.

이 도달 불가능성 신호는 `throw`뿐 아니라 `return`, `break`, `continue`에서도 나온다. 놀랍게도 이들의 타입도 `Nothing`이다. 2.1절에서 자세히 살펴본다. 이처럼 `Nothing`은 예외 흐름과 널 처리, 타입 추론, 도달 가능성 분석을 하나의 개념으로 묶는, Kotlin 타입 시스템의 조용한 중심축이다.

---

## 2. throw와 예외를 던지는 방식

### 2.1 throw는 표현식이고, 그 타입은 Nothing이다

Java에서 `throw`는 문(statement)이다. Kotlin에서는 **표현식(expression)** 이며, 그 타입은 `Nothing`이다. 사소해 보이는 차이지만 앞 절의 내용 전체가 이 사실 위에서 성립한다.

```kotlin
val port: Int = System.getenv("PORT")?.toIntOrNull() ?: throw IllegalStateException("PORT 미설정")
//                                                       ^^^^^ throw 가 표현식이라 ?: 우변에 올 수 있다
```

`throw`가 표현식이므로 엘비스 우변이나 `when` 분기, 심지어 함수 인자 자리에도 들어갈 수 있다. 그리고 타입이 `Nothing`이므로 어떤 타입 요구도 만족한다(1.3절).

더 놀라운 사실은 `return`, `break`, `continue`도 표현식이고 타입이 `Nothing`이라는 점이다. 이들도 "이 지점에서 현재 문맥을 빠져나가므로 뒤의 코드에 값을 넘기지 않는다"는 점에서 `throw`와 성질이 같다.

```kotlin
fun firstPositive(numbers: List<Int>): Int {
    for (n in numbers) {
        val checked: Int = if (n > 0) n else continue   // continue 의 타입은 Nothing
        return checked
    }
    return -1
}
```

`if (n > 0) n else continue`에서 `then` 분기는 `Int`이고, `else` 분기는 `continue`(타입 `Nothing`)이다. 따라서 LUB는 `Int`이고, `checked`도 `Int`로 추론된다. `continue`가 값을 만들지 않으므로 전체 타입은 `then` 분기가 결정한다.

Kotlin은 별도의 삼항 연산자(`?:`처럼 생긴 것이 아니라 Java의 `cond ? a : b`)를 두지 않고도 `if`를 표현식으로 쓸 수 있는데, 이 성질은 그런 문법적 통일성의 일부다([[17 - 표현식으로서의 제어 흐름 - if와 when]]).

> [!note] 명세 기준
> `throw e`에서 `e`가 표현식이라는 것과 `throw` 자체가 표현식이라는 것은 서로 다른 층위의 이야기다. `throw e`가 표현식이라는 말은 "`throw e`를 값이 기대되는 문맥에 놓을 수 있다"는 뜻이고, 그 값의 정적 타입은 `Nothing`이다. 물론 실행할 때 그 "값"이 실제로 계산되어 어딘가에 저장되는 일은 없다. 스택이 되감기기 때문이다. `Nothing` 타입은 정확히 "실제로 값이 만들어지지 않는 표현식"을 위한 정적 표식이다.

### 2.2 던질 수 있는 것은 Throwable뿐이다

`throw` 뒤에 오는 표현식은 **`kotlin.Throwable`의 하위 타입**이어야 한다. 임의의 객체를 던질 수는 없다.

```kotlin
throw IllegalArgumentException("bad")   // OK — Throwable 의 하위타입
throw "그냥 문자열"                        // 컴파일 에러: Throwable 이 아님
throw 42                                 // 컴파일 에러
```

JVM 백엔드에서 `kotlin.Throwable`은 `java.lang.Throwable`로 매핑된다. 즉 JVM 백엔드에서 Kotlin의 예외 계층은 JVM의 예외 장치를 그대로 사용한다. Kotlin/Native와 Kotlin/JS에서도 `Throwable`이 예외의 루트이지만, 그 아래의 구현(스택 되감기, 스택 트레이스 캡처 방식)은 백엔드마다 다르다. 이 장의 성능과 표현에 관한 논의는 대부분 "JVM 백엔드에서"라는 전제로 읽어야 한다.

### 2.3 예외 객체 생성의 실제 비용: 스택 트레이스

예외를 던지는 것 자체는 비용이 적다. 비용이 큰 것은 예외 **객체를 생성**하는 일, 정확히는 그 생성자가 호출하는 스택 트레이스 캡처다. JVM 백엔드에서 `Throwable`의 생성자는 `fillInStackTrace()`를 호출해 *현재 스레드의 전체 호출 스택을 거슬러 올라가며* 스냅샷을 만든다. 스택이 깊을수록 이 비용은 커진다.

```kotlin
// 예외를 제어 흐름으로 남용하면(안티패턴) 이 비용이 반복 부과된다
fun findIndex(list: List<Int>, target: Int): Int {
    list.forEachIndexed { i, v ->
        if (v == target) throw ReturnValue(i)   // 매번 스택 트레이스를 캡처 — 낭비
    }
    throw NotFound()
}
```

> [!caution] 성능 주의
> JVM 백엔드에서 예외를 "빠른 탈출"용 제어 흐름으로 쓰는 것은 대개 나쁜 선택이다. `fillInStackTrace`가 예외마다 스택을 훑기 때문이다. 스택 트레이스가 필요 없는 제어용 예외를 꼭 써야 한다면(라이브러리 내부에서 드물게 정당화된다) 생성자에서 호출되는 `fillInStackTrace`를 오버라이드해 캡처를 건너뛸 수 있다. 코루틴 라이브러리의 일부 내부 예외가 이 기법을 쓴다.
>
> 하지만 애플리케이션 코드에서는 거의 항상 잘못된 도구다. 제어 흐름에는 `return`이나 `Nothing`, `sealed` 결과 타입(9절)을 써야 한다.

```kotlin
// 스택 트레이스 캡처를 생략하는 제어용 예외(고급, 드물게만)
class SilentSignal : Exception() {
    override fun fillInStackTrace(): Throwable = this   // 스택 훑기를 건너뜀
}
```

### 2.4 원인 체인과 다시 던지기

예외를 잡아 다른 예외로 감싸서 다시 던질 때 원래 예외를 `cause`로 연결하면 원인 체인(causal chain)이 보존된다. JVM 백엔드에서는 이 체인이 스택 트레이스에 `Caused by:` 절로 출력된다.

```kotlin
fun loadUser(id: Long): User =
    try {
        parseUserRow(fetchRow(id))
    } catch (e: SQLException) {
        throw UserLoadException("사용자 $id 로드 실패", cause = e)   // 원인 보존
    }

class UserLoadException(message: String, cause: Throwable) : RuntimeException(message, cause)
```

같은 예외를 그대로 다시 던지려면 `throw e`를 쓴다. 이때 스택 트레이스는 원래 캡처한 것이 유지되고 다시 캡처되지 않는다(예외 객체는 생성 시점에 이미 트레이스를 고정했다).

예외를 잡아서 로깅한 뒤 그대로 올려보내는 흔한 패턴에서는 원인을 삼키지 않도록 주의해야 한다. `catch (e: X) { log(e); throw e }`처럼 반드시 원본을 다시 던지거나 `cause`로 감싸야 한다. `catch (e: X) { throw MyException("실패") }`처럼 `cause`를 빠뜨리면 원래 스택 트레이스가 통째로 사라져 디버깅이 매우 어려워진다.

---

## 3. 예외 계층: Throwable, Error, Exception

### 3.1 계층 다이어그램

JVM 백엔드에서 Kotlin의 예외 계층은 `java.lang`의 계층에 매핑되므로 두 계층은 같다. 루트는 `Throwable`이며, 그 아래가 두 갈래로 나뉜다.

```text
                     Throwable                 ← throw 가능한 모든 것의 루트
                    /         \
                Error          Exception        ← 두 대분류
               /    \          /        \
   OutOfMemoryError  …  IOException    RuntimeException
   StackOverflowError                  /    |    \
   (복구 불가 시스템 오류)     (I/O 등    │    │     \
                            "외부" 실패) │    │      IllegalArgumentException
                          NullPointerException │      IllegalStateException
                          IndexOutOfBoundsExc. │      (스택 트레이스만 있으면 되는
                          ArithmeticException  │       프로그래밍 계약 위반들)
                          ConcurrentModExc.  ClassCastException
```

- **`Throwable`**: `throw`와 `catch`의 대상이 되는 최상위 타입이다. `message`, `cause`, 스택 트레이스, `suppressed` 목록을 가진다.
- **`Error`**: JVM이나 런타임 자체의 심각한 오류다. `OutOfMemoryError`, `StackOverflowError` 등이 있다. 애플리케이션이 의미 있게 복구할 수 없으므로 일반적으로 **잡지 말아야 한다**.
- **`Exception`**: 애플리케이션이 (적어도 이론상) 다룰 수 있는 예외다.
- **`RuntimeException`**: `Exception`의 하위 타입이다. Java에서 유일하게 "비검사(unchecked)"인 갈래이지만, Kotlin에서는 이 구분이 언어 차원에서 사라진다(4절).

### 3.2 Error를 잡지 말라는 관례

`Error`는 문법적으로 `catch (e: Error)`처럼 잡을 수 있지만, 잡지 않는 것이 관례다. `OutOfMemoryError`를 잡아서 무엇을 할 수 있을까? 메모리가 부족해서 던져진 예외를 처리하는 코드 자체가 메모리를 필요로 한다. `StackOverflowError`도 마찬가지다.

그래서 무차별적으로 `catch (e: Throwable)`을 쓰는 것은 위험하다. `Error`까지 삼켜 시스템을 제대로 동작하지 않는 좀비 상태로 만들 수 있기 때문이다. 넓게 잡아야 한다면 `catch (e: Exception)`을 상한으로 두는 것이 관례다.

```kotlin
try {
    riskyOperation()
} catch (e: Exception) {     // Error 는 통과시킨다(잡지 않음)
    log.error("작업 실패", e)
    fallback()
}
```

> [!warning] 흔한 오해
> "모든 것을 잡으려면 `catch (e: Throwable)`이 안전하다"고 생각하기 쉽지만, 사실은 정반대다. `Throwable`을 잡으면 복구할 수 없는 `Error`와 뒤에서 볼 코루틴의 `CancellationException`(정상 취소 신호)까지 삼킨다. 넓게 잡을수록 위험하다. 무엇을 잡을지는 무엇을 복구할 수 있는지에서 거꾸로 정해야 한다.

### 3.3 표준 예외

자주 쓰는 표준 예외와 그 의미는 다음과 같다.

| 예외 | 의미 | 던지는 표준 함수 |
|------|------|------------------|
| `IllegalArgumentException` | 함수에 부적절한 인자가 들어옴 | `require(...)` |
| `IllegalStateException` | 객체가 이 연산을 하기에 부적절한 상태 | `check(...)`, `error(...)` |
| `IndexOutOfBoundsException` | 인덱스 범위 초과 | 컬렉션 `get` 등 |
| `NoSuchElementException` | 원소 없음 | `first()`, 빈 이터레이터의 `next()` |
| `NullPointerException` | 널 역참조 | `!!`(널일 때, 6장 널 안전성) |
| `ClassCastException` | 잘못된 캐스트 | `as`(실패 시) |
| `ArithmeticException` | 산술 오류(정수 0 나눗셈 등) | `1 / 0` |
| `NumberFormatException` | 문자열→수 변환 실패 | `"x".toInt()` |
| `ConcurrentModificationException` | 순회 중 컬렉션 구조 변경 | 이터레이터 |
| `UnsupportedOperationException` | 지원하지 않는 연산 | 읽기 전용 뷰의 변경 시도 |

### 3.4 계약 함수: require, check, error, assert

Kotlin 표준 라이브러리는 예외를 직접 `throw`하는 대신 *의도를 드러내는* 계약(contract) 함수를 제공한다. 이 함수들은 조건을 검사하고, 실패하면 적절한 예외를 던진다. 무엇을 검증하느냐에 따라 예외 종류가 달라진다.

```kotlin
fun withdraw(account: Account, amount: Long) {
    require(amount > 0) { "출금액은 양수여야 함: $amount" }        // 실패 → IllegalArgumentException
    check(!account.frozen) { "동결 계좌: ${account.id}" }         // 실패 → IllegalStateException
    require(amount <= account.balance) { "잔액 부족" }            // 인자 검증 → IllegalArgumentException
    account.balance -= amount
}
```

- **`require(condition) { lazyMessage }`**: 인자(argument)를 검증한다. 실패하면 `IllegalArgumentException`을 던진다. "호출자가 잘못했다"는 뜻이다.
- **`check(condition) { lazyMessage }`**: 상태(state)를 검증한다. 실패하면 `IllegalStateException`을 던진다. "객체가 잘못된 상태다"라는 뜻이다.
- **`error(message): Nothing`**: 무조건 `IllegalStateException`을 던진다. "도달해서는 안 되는 곳"을 나타낸다.
- **`requireNotNull(x) { ... }` / `checkNotNull(x) { ... }`**: 널을 검사하고 스마트 캐스트를 적용한다(널이 아니면 non-null 타입을 반환한다).

메시지 인자가 람다 `{ ... }`라는 점이 중요하다. 조건이 참이면 이 람다는 *평가되지 않고*, 실패했을 때만 메시지 문자열을 만든다. 성공 경로에서 문자열 보간 비용을 치르지 않기 위한 설계다. 이 함수들은 `inline`이므로([[15 - 인라인 함수와 reified]]) 람다 호출 오버헤드도 없다.

`assert`는 성격이 다르다.

```kotlin
fun midpoint(a: Int, b: Int): Int {
    val mid = (a + b) / 2
    assert(mid in a..b) { "중점이 범위 밖: $mid" }   // 실패 → AssertionError (단, 활성화됐을 때만)
    return mid
}
```

> [!note] 명세 기준
> JVM 백엔드에서 `assert`는 **JVM의 어서션 활성화 플래그(`-ea`)가 켜져 있을 때만** 조건 위반 시 `AssertionError`를 던진다. 플래그가 꺼진 기본 운영 환경에서는 조건이 거짓이어도 아무 예외도 던지지 않는다.
>
> 다만 Java의 `assert` *문*(statement)과 달리 Kotlin의 `assert`는 일반 함수(인라인) 호출이므로, `-ea`가 꺼져 있어도 **조건식 인자 자체는 평가된다**. 생략되는 것은 예외를 던지는 동작뿐이다. 그래서 조건식에 부작용이나 비용이 큰 계산을 넣으면 운영 환경에서도 그 비용을 치른다.
>
> 따라서 `assert`는 개발과 테스트 중에 계약 위반을 잡는 도구이며, 운영에서 반드시 강제해야 하는 검증에 써서는 안 된다. 운영에서도 항상 검사해야 하는 조건에는 `require`나 `check`를 써야 한다. 이 활성화 동작은 JVM 백엔드에 고유한 것이며, 다른 백엔드에서 `assert`가 어떻게 동작하는지는 플랫폼 구현에 따른다.

`require`, `check`, `error`가 던지는 예외 객체는 스택 트레이스를 캡처하므로(2.3절), 정상 경로에서는 절대 실행되지 않도록 조건을 배치하는 것이 좋다. 이 함수들은 `contract`(6장 널 안전성 참조)로 "이 함수가 정상적으로 복귀했다면 조건이 참이다"라는 사실을 컴파일러에 알려, 이후 코드의 스마트 캐스트를 돕기도 한다.

---

## 4. 검사 예외가 없다

### 4.1 Java의 검사 예외

Java는 예외를 두 부류로 나눈다. `RuntimeException`과 `Error`의 하위 타입은 **비검사(unchecked)** 예외이고, 나머지 `Exception`의 하위 타입(`IOException`, `SQLException` 등)은 **검사(checked)** 예외다. 검사 예외를 던질 수 있는 메서드는 그 사실을 시그니처에 `throws`로 선언해야 하고, 호출자는 그 예외를 `catch`하거나 자신도 `throws`로 다시 선언해야 한다. 이 규칙은 컴파일러가 강제한다.

```java
// Java
public String readFirstLine(String path) throws IOException {   // 반드시 선언
    try (BufferedReader r = new BufferedReader(new FileReader(path))) {
        return r.readLine();
    }
}

// 호출자도 처리를 강제당함
void caller() {
    try {
        readFirstLine("a.txt");
    } catch (IOException e) { /* 강제 */ }
}
```

의도는 좋았다. "이 함수는 실패할 수 있으니 처리하라"는 사실을 컴파일러가 문서화하고 강제한다. 하지만 실무에서 검사 예외는 악명 높은 마찰을 낳았다.

### 4.2 Kotlin이 검사 예외를 버린 이유

Kotlin에는 검사 예외라는 개념이 **없다**. 모든 예외를 비검사 예외처럼 취급한다. 어떤 함수도 `throws`를 선언할 필요가 없고, 호출자도 어떤 예외든 잡을 의무가 없다.

```kotlin
fun readFirstLine(path: String): String? =
    File(path).bufferedReader().use { it.readLine() }   // IOException 을 던질 수 있지만 선언 불필요

fun caller() {
    val line = readFirstLine("a.txt")   // catch 강제 없음
}
```

Kotlin이 검사 예외를 버린 이유는 언어 설계자들이 관찰한 검사 예외의 실패 사례에 있다.

1. **예외를 삼키게 만든다(swallowing).** 처리를 강제당한 개발자는 종종 `catch (e: IOException) {}`처럼 빈 블록으로 예외를 삼킨다. 강제가 오히려 침묵을 낳는다.
2. **전파 경로를 오염시킨다(propagation).** 저수준 `IOException`이 계층을 타고 올라가면, 중간의 모든 인터페이스가 그 저수준 예외를 시그니처에 노출하거나 감싸야 한다. 추상화 경계가 구현 세부 사항에 오염된다.
3. **람다·제네릭과 잘 맞지 않는다.** `map { it.parse() }`에서 `parse`가 검사 예외를 던지면, `map`의 함수 타입 시그니처로는 그 사실을 표현할 수 없다. Java의 함수형 인터페이스는 검사 예외를 제네릭하게 전파하지 못하므로, 스트림과 람다에서는 검사 예외를 거의 항상 감싸야 했다.
4. **버전 진화를 방해한다.** 라이브러리가 새로운 실패 모드를 추가하려면 시그니처의 `throws`를 바꿔야 하고, 이는 모든 호출자를 깨뜨리는 소스 비호환 변경이다.

> [!info] 역사 메모
> 검사 예외에 회의적인 언어는 Kotlin이 처음이 아니다. C#은 애초에 검사 예외를 도입하지 않았고, 설계자 Anders Hejlsberg는 "검사 예외는 버전 관리와 확장성 문제를 낳으며, 실제로는 코드 품질을 떨어뜨린다"고 공개적으로 비판했다. Java 내부에서도 스트림 API(자바 8)를 도입할 때 검사 예외와 람다가 잘 맞지 않는다는 문제가 두드러졌다. Kotlin은 이 오랜 논쟁에서 "강제하지 않는다" 쪽을 택했다.

### 4.3 편리함의 대가: 누가 치르는가

여기서 흔한 오해를 바로잡자. **"검사 예외가 없으니 Kotlin에서는 예외를 신경 쓰지 않아도 된다"** 는 생각은 완전히 틀렸다. 검사 예외가 없다는 것은 **예외가 사라졌다는 뜻이 아니라, 예외를 다룰 책임이 컴파일러에서 개발자에게 옮겨졌다**는 뜻이다.

`readFirstLine`은 여전히 `IOException`을 던진다. 이 예외를 잡지 않으면 스택을 타고 올라가 `main`을 벗어나고 프로그램을 종료시킨다. 컴파일러가 경고하지 않을 뿐, 실패는 그대로 존재한다.

```kotlin
fun main() {
    val line = readFirstLine("존재하지-않는-파일.txt")
    // 파일이 없으면 FileNotFoundException 이 던져지고, 아무도 잡지 않으면
    // 이 예외가 main 을 뚫고 나가 프로세스가 비정상 종료된다. 컴파일러는 침묵했다.
    println(line)
}
```

그래서 Kotlin에서 예외 처리는 *규율*(discipline)의 문제가 된다.

- **문서화 책임.** 함수가 어떤 예외를 던질 수 있는지 KDoc의 `@throws`/`@exception` 태그로 문서화하는 일은 개발자가 해야 한다. 컴파일러가 대신 해 주지 않는다.
- **의도적 처리.** "이 실패를 여기서 복구할 것인가, 위로 올려보낼 것인가"를 매번 판단해야 한다.
- **결과 타입으로의 전환.** 예상할 수 있는 실패는 예외 대신 `sealed` 결과 타입이나 `Result`로 *타입에 담아*, 컴파일러가 처리를 강제하게 만드는 것이 Kotlin다운 접근이다(9절). 즉 Kotlin은 "실패를 예외로 던지되 컴파일러가 강제하지 않는" 방식과 "실패를 값으로 반환해 컴파일러가 `when` 완전성으로 강제하는" 방식 가운데 후자를 권장하는 언어다.

### 4.4 @Throws와 Java 상호 운용

Kotlin에 검사 예외가 없다는 사실은 Java와 함께 쓸 때 마찰을 낳는다. JVM 백엔드에서 Kotlin 함수를 컴파일하면 기본적으로 `throws` 절이 붙지 않는다. 그런데 Java 코드가 그 Kotlin 함수를 호출하면서 검사 예외를 잡으려 하면, Java 컴파일러는 "그 예외는 던져지지 않는다(unreachable catch)"며 컴파일 에러를 낸다.

```kotlin
// Kotlin — 그냥 두면 JVM 시그니처에 throws 가 없다
@Throws(IOException::class)   // 이 애노테이션이 JVM 시그니처에 throws IOException 을 심는다
fun writeLog(msg: String) {
    File("log.txt").appendText(msg)   // IOException 가능
}
```

```java
// Java 호출자 — @Throws 덕분에 이 catch 가 유효하다
try {
    KotlinFileKt.writeLog("hello");
} catch (IOException e) {   // @Throws 없으면 "unreachable catch" 컴파일 에러
    e.printStackTrace();
}
```

`@Throws`는 Kotlin 쪽 의미를 바꾸지 않는다(Kotlin 호출자는 여전히 처리를 강제받지 않는다). 이 애노테이션은 **JVM 바이트코드 메서드 시그니처에 `throws` 절을 추가**해 Java와의 상호 운용을 매끄럽게 할 뿐이다. 이 애노테이션과 상호 운용 전반은 [[45 - Java 상호운용]]에서 자세히 다룬다.

---

## 5. try는 표현식이다

### 5.1 try/catch/finally 문법

기본 문법부터 보자. `try` 블록에서 예외가 발생하면 타입이 맞는 `catch`로 제어가 넘어가고, `finally`는 성공과 실패에 관계없이 실행된다.

```kotlin
fun parseAge(s: String): Int {
    try {
        return s.trim().toInt()
    } catch (e: NumberFormatException) {
        return -1
    } finally {
        println("parseAge 종료")   // 성공하든 예외든 항상 실행
    }
}
```

`catch`는 0개 이상, `finally`는 0~1개 올 수 있다. 단 `try`에는 `catch`나 `finally` 중 적어도 하나가 있어야 한다(`try { }`만 쓰는 것은 의미가 없으므로 문법 오류다).

### 5.2 표현식으로서의 try: 값을 만든다

또 하나의 흔한 오해는 **"`try`는 문(statement)일 뿐이다"** 라는 생각이다. Kotlin에서 `try`/`catch`는 **표현식**이며 값을 만든다. 17장(표현식으로서의 제어 흐름)에서 본 `if`나 `when`처럼, `try`도 값이 기대되는 자리에 놓을 수 있다.

```kotlin
fun parseAge(s: String): Int {
    val age: Int = try {
        s.trim().toInt()          // try 블록의 마지막 표현식이 값이 된다 (성공 시)
    } catch (e: NumberFormatException) {
        -1                        // catch 블록의 마지막 표현식이 값이 된다 (예외 시)
    }
    return age
}
```

`try` 표현식의 값은 다음과 같이 정해진다.

- `try` 블록이 정상적으로 끝나면 `try` 블록의 마지막 표현식 값이 된다.
- 예외가 발생해 어떤 `catch`가 처리하면 그 `catch` 블록의 마지막 표현식 값이 된다.

두 경로가 값의 타입을 결정하고, 전체 표현식의 타입은 두 타입의 LUB다. 위 예에서 `try` 분기와 `catch` 분기가 모두 `Int`이므로 전체도 `Int`다. 이렇게 값을 기반으로 쓰는 스타일은 가변 변수를 없애고 결과를 `val`에 담을 수 있게 해(11장 변수와 초기화) 코드를 더 함수형으로 만든다.

```kotlin
// 흔한 관용구: 파싱 실패 시 기본값
val port: Int = try { config["port"]?.toInt() ?: 8080 } catch (e: NumberFormatException) { 8080 }
```

### 5.3 finally는 값에 기여하지 않는다

`finally` 블록은 실행되지만 `try` 표현식의 **값에는 기여하지 않는다**. 값은 오직 `try` 분기나 `catch` 분기에서 나온다.

```kotlin
val result: Int = try {
    10
} finally {
    99   // 이 값은 버려진다 — result 는 10 이다
}
// => result == 10  (finally 의 99 는 무시)
```

`finally`의 역할은 값을 만드는 것이 아니라 파일 닫기, 락 해제, 로그 남기기 같은 **정리(cleanup)** 작업이다. 그래서 `finally`의 마지막 표현식은 값으로 쓰이지 않는다. 만약 `finally`가 값에 기여한다면 "정상적으로 끝났는데 정리 코드가 결과를 덮어쓰는" 혼란이 생길 것이다.

> [!note] 명세 기준
> `try`를 표현식으로 쓰려면 값이 필요한 문맥이어야 하고, 이때 값은 `try` 분기나 `catch` 분기에서만 온다. 반대로 `try` 블록이나 `catch` 블록이 `return`으로 끝나면(5.1절의 예처럼) `try`는 표현식이 아니라 문으로 쓰인 것이다. `return`의 타입은 `Nothing`이라 값을 넘기지 않기 때문이다. 같은 `try` 문법이 문맥에 따라 표현식도 되고 문도 될 수 있다는 점에서 Kotlin의 "표현식 지향" 설계가 일관되게 드러난다.

### 5.4 다중 catch와 순서, 그리고 Kotlin에 없는 것

여러 종류의 예외를 서로 다르게 처리하려면 `catch`를 여러 개 나열한다. 위에서 아래로 검사해 **처음으로 타입이 맞는** `catch`가 예외를 처리한다.

```kotlin
try {
    risky()
} catch (e: NumberFormatException) {   // 더 구체적인 타입을 먼저
    handleFormat(e)
} catch (e: IllegalArgumentException) { // NumberFormatException 은 이것의 하위타입
    handleArg(e)
} catch (e: Exception) {                // 가장 일반적인 것을 마지막에
    handleGeneric(e)
}
```

`NumberFormatException`은 `IllegalArgumentException`의 하위 타입이므로, 순서를 뒤집으면 `NumberFormatException`용 `catch`에는 절대 도달하지 못한다. 더 구체적인 타입을 위에 두는 것은 개발자의 책임이다.

> [!warning] 흔한 오해
> "Kotlin도 Java처럼 `catch (e: A | B)` 형태의 다중 타입 캐치를 쓸 수 있다"고 생각하기 쉽지만, Kotlin 2.x 기준으로는 **지원하지 않는다.** Java 7의 유니온 캐치 문법 `catch (IOException | SQLException e)`는 Kotlin에 없다. 여러 타입을 같은 방식으로 처리하려면 (a) 타입마다 `catch`를 쓰거나, (b) 공통 상위 타입으로 한 번 잡은 뒤 `when (e)`로 분기한다. 방법 (b)는 다음과 같다.

```kotlin
try {
    risky()
} catch (e: Exception) {
    when (e) {
        is IOException, is TimeoutException -> retry()
        is IllegalArgumentException -> reject(e)
        else -> throw e   // 처리 못 하는 것은 다시 던진다 — 삼키지 말 것
    }
}
```

여기서 `else -> throw e`가 중요하다. `when`으로 분기하더라도 처리할 수 없는 예외는 반드시 다시 던져 위로 올려보내야 한다. 그러지 않으면 `catch (e: Exception)`이 예상하지 못한 모든 예외를 조용히 삼켜 버린다. 4.2절에서 본 검사 예외의 실패를 스스로 되풀이하는 셈이다.

---

## 6. finally의 동작 원리: 정말 항상 실행되는가?

### 6.1 return·break·continue·throw 앞에서도 실행된다

`finally`가 보장하는 핵심은 "`try`/`catch` 블록을 *어떻게* 벗어나든, 벗어나는 동작이 완료되기 전에 실행된다"는 것이다. 정상적으로 끝나든, 예외가 발생하든, `return`이나 `break`/`continue`로 빠져나가든 마찬가지다.

```kotlin
fun demo(): String {
    try {
        return "try 에서 return"   // return 값을 계산한 뒤, finally 를 실행하고, 그 다음 실제로 복귀
    } finally {
        println("finally 실행됨")   // return 이 완료되기 *전에* 출력된다
    }
}
// 실행 결과:
// finally 실행됨
// => 반환값 "try 에서 return"
```

`return "..."`은 반환할 값을 계산하지만, 실제 스택 복귀는 `finally`가 끝난 뒤에 일어난다. 그래서 `finally`의 `println`이 먼저 실행되고, 그다음에 함수가 값을 가지고 복귀한다. 루프 안이라면 `break`/`continue`도 똑같다. 루프를 빠져나가기 전에 `finally`가 실행된다.

### 6.2 finally에서 return하는 함정

`finally` 블록 안에서 `return`을 하면 `try`/`catch`에서 진행 중이던 반환이나 **예외 전파를 통째로 덮어쓴다**. 이는 거의 항상 버그다.

```kotlin
fun trap(): Int {
    try {
        throw RuntimeException("원래 예외")
    } finally {
        return 42   // 예외를 삼키고 42 를 반환해버린다 — 예외가 사라진다!
    }
}
// => trap() 은 예외를 던지지 않고 42 를 반환한다. "원래 예외" 는 흔적도 없이 사라졌다.
```

`try`에서 예외가 던져졌지만, `finally`의 `return 42`가 실행되면서 그 예외 전파가 취소되고 함수는 `42`를 정상적으로 반환한다. 던져졌어야 할 예외가 조용히 사라진 것이다. 같은 이유로 `finally`에서 예외를 던지면 `try`에서 진행 중이던 예외를 가려 버린다(masking). Kotlin 컴파일러는 `finally`에서 `return`을 하면 "finally 블록이 흐름 제어를 덮어쓴다"는 취지의 경고를 낸다.

> [!caution] 성능 주의 / 함정
> `finally`에서는 닫기나 해제 같은 정리 작업만 하고, 절대로 `return`하거나 새 예외를 던지지 않는 것이 철칙이다. 정리 중에 발생할 수 있는 예외도 원래 예외를 가리지 않도록 다뤄야 한다. 정리 중에 난 예외가 원래 예외를 가리는 이 문제를 표준 라이브러리가 어떻게 해결하는지는 7절의 `use`와 억압된 예외(suppressed exception)에서 살펴본다.

### 6.3 finally가 실행되지 않는 경우

"`finally`는 항상 실행된다"는 말에도 예외가 있다. `finally`도 결국 코드이므로, 그 코드에 도달하기 전에 스레드나 프로세스, 기계 자체가 멈추면 실행될 수 없다.

```text
finally 가 실행되지 않는 경우:
  1. try 블록에서 System.exit(...) 호출     → JVM 즉시 종료, finally 건너뜀
  2. try 블록에서 무한 루프 / 영원한 블로킹   → finally 에 도달 못 함
  3. 데몬 스레드였는데 JVM 이 종료됨          → 스레드가 중단됨
  4. StackOverflowError / OutOfMemoryError 로 finally 실행 자체가 불가할 때
  5. 프로세스 강제 종료(kill -9), 정전, 하드웨어 고장
  6. Kotlin/Native 등에서 런타임이 프로세스를 중단시키는 경우
```

```kotlin
fun willFinallyRun() {
    try {
        println("try 시작")
        System.exit(0)      // JVM 을 즉시 종료 — 아래 finally 는 실행되지 않는다
    } finally {
        println("이 줄은 출력되지 않는다")
    }
}
```

> [!note] 명세 기준
> `finally`가 보장하는 것은 "제어가 `try`/`catch`를 벗어나는 *정상적인 언어 수준 경로*에서는 반드시 실행된다"는 것이지, "물리적으로 무슨 일이 있어도 실행된다"는 것이 아니다. `System.exit()`은 언어 수준의 제어 흐름을 우회해 런타임을 직접 끝내므로 `finally`를 건너뛴다.
>
> 따라서 "`finally`는 반드시 실행된다"는 가정에 트랜잭션 커밋처럼 *정확성에 필수적인* 로직을 맡겨서는 안 된다. 그런 로직에는 별도의 복구 설계와 멱등성 설계가 필요하다. `finally`는 "리소스 누수를 줄이기 위한 최선의 노력(best effort)"으로 이해하는 것이 정확하다.

### 6.4 JVM 백엔드에서의 컴파일: 예외 테이블

JVM 백엔드에서 `try`/`catch`/`finally`는 어떤 바이트코드가 될까? `catch`는 메서드의 **예외 테이블(exception table)** 로 컴파일된다. 예외 테이블은 "바이트코드 오프셋 [start, end) 구간에서 타입 X의 예외가 던져지면 핸들러 오프셋 H로 점프하라"는 엔트리의 목록이다. 예외가 던져지면 JVM은 이 테이블에서 타입이 맞는 핸들러를 찾고, 없으면 스택을 한 프레임 되감아 상위 메서드의 테이블을 다시 조회한다.

```text
[개념적 바이트코드 — try { A } catch (e: X) { B }]

  Exception table:
    from  to  target  type
      L0  L1    L2     X          ← L0~L1 구간의 X 예외 → L2 로

  L0: <A 의 코드>
  L1: goto L3              ← 정상 종료 시 catch 를 건너뜀
  L2: astore e             ← 던져진 예외를 지역변수 e 에 저장
      <B 의 코드>
  L3: <이후 코드>
```

`finally`는 더 까다롭다. `finally` 블록은 "이 영역을 빠져나가는 *모든* 경로"에 들어가야 하기 때문이다. 정상 종료 경로와 각 `catch` 경로, 그리고 타입이 맞는 `catch`가 없어 예외가 계속 전파되는 경로까지, 컴파일러는 각 경로의 끝에서 `finally` 코드가 실행되도록 배치한다.

특히 타입이 맞는 `catch` 없이 전파되는 예외의 경로를 위해, 컴파일러는 해당 구간에 **어떤 예외든 잡는(catch-all) 핸들러**를 넣고, `finally`를 실행한 뒤 그 예외를 다시 던지는(rethrow) 코드를 생성한다. 예외가 빠져나가더라도 `finally`가 실행되는 이유가 여기에 있다.

옛 javac는 `finally`를 `jsr`/`ret` 서브루틴으로 한 번만 두고 각 이탈 경로에서 호출했다. 이 방식이 나중에 폐기되면서, 오늘날의 컴파일러(Kotlin JVM 백엔드 포함)는 대체로 `finally` 코드를 정상 이탈 경로와 각 `catch` 이탈 경로에 복제하고, 타입이 맞는 `catch` 없이 전파되는 경로에는 catch-all 핸들러를 넣는다. 구체적인 코드 생성 방식은 컴파일러 버전과 백엔드에 따라 다르지만, 관찰되는 의미는 같다.

---

## 7. 리소스 관리: use와 억압된 예외

### 7.1 문제: 정리 코드와 예외의 충돌

파일, 소켓, 락 같은 리소스는 다 쓰고 나면 닫아야 한다. `finally`로 닫는 것이 정석이지만, 앞 절에서 봤듯이 `finally`에서 예외가 나면 `try`의 원래 예외를 가려 버린다. 직접 작성하면 다음처럼 지저분한 코드가 된다.

```kotlin
// 손으로 짠 리소스 관리 — 미묘한 버그의 온상
fun readManual(path: String): String {
    val reader = File(path).bufferedReader()
    try {
        return reader.readText()   // 여기서 예외 A 발생 가능
    } finally {
        reader.close()             // 여기서 예외 B 발생 가능 → A 를 가림!
    }
}
```

`readText()`가 예외 A를 던지고, 스택을 되감는 도중에 `close()`가 예외 B를 던지면, 호출자는 근본 원인인 A를 보지 못하고 B만 받는다. Java 7은 이 문제를 `try`-with-resources 문법으로 해결했고, Kotlin은 문법 대신 **`use`라는 표준 라이브러리 함수**로 해결한다.

### 7.2 use: 인라인 확장 함수

`use`는 `Closeable`(공통 표준 라이브러리에서는 `AutoCloseable`)의 확장 함수다. 블록을 실행하고, 성공과 실패에 관계없이 리소스를 닫으며, 예외가 가려지는 문제를 올바르게 처리한다.

```kotlin
fun readWithUse(path: String): String =
    File(path).bufferedReader().use { reader ->
        reader.readText()          // 이 블록이 끝나면(정상이든 예외든) reader.close() 가 호출된다
    }
```

`use`는 `inline` 함수이므로(15장 인라인 함수와 reified) 람다 호출 오버헤드가 없고, 람다 안에서 비지역 `return`도 자연스럽게 쓸 수 있다. 표준 라이브러리의 구현을 개념적으로 요약하면 다음과 같다.

```kotlin
// kotlin-stdlib 의 use (개념 요약)
public inline fun <T : Closeable?, R> T.use(block: (T) -> R): R {
    var exception: Throwable? = null
    try {
        return block(this)
    } catch (e: Throwable) {
        exception = e          // 블록에서 던져진 예외를 기억
        throw e
    } finally {
        // 정상 종료면 그냥 close(). 예외 중이면 close 의 예외를 "억압"으로 붙임.
        when {
            this == null -> {}
            exception == null -> this.close()
            else -> try { this.close() } catch (closeEx: Throwable) {
                exception.addSuppressed(closeEx)   // 원래 예외를 가리지 않고 부착
            }
        }
    }
}
```

### 7.3 억압된 예외: 원인을 잃지 않는다

핵심은 `else` 분기의 `addSuppressed`다. 블록에서 예외 A가 진행 중일 때 `close()`가 예외 B를 던지면, `use`는 B를 A의 **억압된 예외(suppressed exception)** 목록에 붙인다. 그러면 호출자는 근본 원인인 A를 받고, B는 A의 스택 트레이스에 `Suppressed:` 절로 함께 출력된다. 어느 예외도 잃지 않는다.

```text
Exception in thread "main" java.io.IOException: 읽기 실패        ← 원래 예외 A (근본 원인)
    at ...readText...
    Suppressed: java.io.IOException: 닫기 실패                   ← close 의 예외 B (억압됨)
        at ...close...
```

> [!note] 명세 기준
> `addSuppressed`와 억압된 예외 목록은 JVM 백엔드에서 `java.lang.Throwable`이 제공하는 메커니즘(Java 7+)이며, Kotlin `Throwable`은 이를 그대로 노출한다(`e.suppressedExceptions`로 목록에 접근한다).
>
> "원래 예외를 우선하고 정리 중 예외는 억압한다"는 이 규칙 덕분에, `use`는 6.2절에서 본 "`finally`가 예외를 가리는 함정"을 라이브러리 차원에서 올바르게 피한다. `finally { close() }`를 직접 작성하면 이 억압 처리를 빠뜨리기 쉽다. 그래서 리소스는 항상 `use`로 다루는 것이 정석이다.

### 7.4 여러 리소스와 Java 비교

리소스가 여러 개면 `use`를 중첩한다. 리소스는 안쪽부터 바깥쪽 순서로 닫힌다.

```kotlin
File("in.txt").bufferedReader().use { input ->
    File("out.txt").bufferedWriter().use { output ->
        input.forEachLine { output.appendLine(it.uppercase()) }
    }   // output 이 먼저 닫힘
}       // input 이 그다음 닫힘
```

```java
// Java 는 문법(try-with-resources)으로 같은 일을 한다
try (var input = Files.newBufferedReader(Path.of("in.txt"));
     var output = Files.newBufferedWriter(Path.of("out.txt"))) {
    // ...
}   // 선언 역순으로 자동 close, 억압 예외도 자동 처리
```

Java는 이 기능을 *문법*으로 내장했고, Kotlin은 *함수*로 제공한다. Kotlin의 이 선택은 "특수 문법을 늘리기보다 인라인 고차 함수로 표현한다"는 언어 철학을 일관되게 보여 준다. 스코프 함수([[16 - 스코프 함수와 수신 객체 관용구]])나 `repeat`, `runCatching`도 모두 같은 계열이다. 문법을 늘리지 않고도 라이브러리로 같은 표현력을 얻을 수 있다는 점이 인라인 함수의 강점이다.

---

## 8. runCatching과 Result

### 8.1 Result<T>: 성공 또는 실패를 담는 값

예외를 던져 스택을 되감는 대신, 성공값이나 실패(예외)를 하나의 값에 담아 반환하고 싶을 때가 있다. 이때 쓰는 것이 Kotlin 표준 라이브러리의 `Result<T>`다. `Result<T>`는 내부에 성공값(`T`)이나 `Failure(Throwable)`를 담는다.

```kotlin
val ok: Result<Int> = Result.success(42)
val fail: Result<Int> = Result.failure(IllegalStateException("nope"))

println(ok.isSuccess)      // => true
println(fail.isFailure)    // => true
println(ok.getOrNull())    // => 42
println(fail.getOrNull())  // => null
println(fail.exceptionOrNull())  // => java.lang.IllegalStateException: nope
```

> [!note] 명세 기준
> `Result<T>`는 `@JvmInline value class`다([[36 - 타입 별칭과 인라인 value class]]). 내부에 `Any?` 하나만 담아서, 성공이면 값 자체를, 실패면 `Failure`라는 내부 래퍼를 넣는다.
>
> 값 클래스이므로 JVM 백엔드에서는 많은 경우 별도의 객체 할당 없이 표현되지만, nullable로 쓰이거나 제네릭 타입 인자로 전달되거나 컬렉션에 담기면 박싱된다. 즉 `Result`는 "가벼운 성공/실패 봉투"를 목표로 설계됐지만, 항상 할당이 없는 것은 아니다.

### 8.2 runCatching과 변환 연산

`runCatching`은 블록을 실행하고 그 결과를 `Result`로 감싼다. 성공하면 `Result.success(값)`을 만들고, 예외가 던져지면 그 예외를 잡아 `Result.failure(예외)`를 만든다.

```kotlin
fun parsePort(s: String): Result<Int> = runCatching { s.trim().toInt() }

val r = parsePort("8080")     // Result.success(8080)
val bad = parsePort("xyz")    // Result.failure(NumberFormatException)
```

`Result`는 함수형 변환 연산을 제공하므로, 예외 처리를 `try`/`catch`로 흩어 놓지 않고 파이프라인으로 다룰 수 있다.

```kotlin
val port: Int = runCatching { config["port"]!!.toInt() }
    .map { it.coerceIn(1, 65535) }          // 성공값 변환 (실패면 건너뜀)
    .recover { 8080 }                        // 실패를 기본값으로 복구
    .getOrThrow()                            // 최종적으로 값 꺼내기 (실패였다면 예외 재던짐)

// 자주 쓰는 꺼내기 연산들:
runCatching { risky() }
    .getOrNull()        // 실패면 null
    .let { /* ... */ }
runCatching { risky() }.getOrDefault(defaultValue)   // 실패면 기본값
runCatching { risky() }.getOrElse { e -> handle(e) } // 실패면 예외로 계산
runCatching { risky() }.fold(
    onSuccess = { v -> "성공: $v" },
    onFailure = { e -> "실패: ${e.message}" }
)   // 두 경로를 한 번에 처리
```

그중에서도 `fold`가 특히 유용하다. 성공과 실패 두 경로를 각각 람다로 처리해 하나의 값으로 합친다. `when` 없이도 두 경우를 빠짐없이 처리하도록 강제하는 셈이다.

### 8.3 runCatching의 위험: 무엇을 삼키는가

이 장에서 가장 중요한 실무 함정이 여기에 있다. **`runCatching`은 `Throwable`을 잡는다.** `Exception`이 아니라 `Throwable`이므로, `Error`도 코루틴의 `CancellationException`도 잡는다.

```kotlin
// runCatching 의 개념적 구현
public inline fun <T, R> T.runCatching(block: T.() -> R): Result<R> =
    try {
        Result.success(block())
    } catch (e: Throwable) {     // ← Throwable! Exception 이 아니다
        Result.failure(e)
    }
```

이 때문에 두 가지 위험이 생긴다.

첫째, **`Error`를 삼킨다.** `OutOfMemoryError`나 `StackOverflowError`가 `runCatching` 안에서 발생하면, 복구할 수 없는 시스템 오류가 `Result.failure`로 포장되어 정상 흐름처럼 취급될 수 있다(3.2절에서 본 위험이 그대로 재현된다).

둘째, 더 치명적인 것은 **`CancellationException`을 삼킨다**는 점이다. 코루틴에서 취소는 `CancellationException`을 던지는 방식으로 구현된다(42장 suspend와 코루틴).

코루틴 안에서 `runCatching { suspendingCall() }`을 쓰면, 취소 신호로 던져진 `CancellationException`이 `Result.failure`에 갇혀 코루틴 밖으로 전파되지 못한다. 그러면 취소된 코루틴이 취소를 알아차리지 못하고 계속 실행되거나, 구조적 동시성이 깨진다.

```kotlin
// 코루틴 안에서 — 위험한 패턴
suspend fun fetch(): Result<Data> = runCatching {
    apiCall()   // 여기서 취소되면 CancellationException 이 던져지는데
}               // runCatching 이 그것을 삼켜 Result.failure(CancellationException) 로 만든다!
```

> [!caution] 성능 주의 / 함정
> 코루틴 컨텍스트에서 `runCatching`(또는 `catch (e: Throwable)`, `catch (e: Exception)`)을 쓸 때는 **`CancellationException`을 반드시 다시 던져야** 한다. 관용적인 해법은 예외를 잡은 뒤 취소 예외인지 검사해서 다시 던지는 것이다.

```kotlin
suspend fun fetchSafe(): Result<Data> =
    runCatching { apiCall() }
        .onFailure { if (it is CancellationException) throw it }   // 취소는 재던짐

// 또는 직접:
suspend fun fetchSafe2(): Data? =
    try {
        apiCall()
    } catch (e: CancellationException) {
        throw e            // 취소는 반드시 전파
    } catch (e: Exception) {
        log.warn("실패", e)
        null
    }
```

취소 예외 처리의 전체 내용과 구조적 동시성에서의 예외 전파는 42장(suspend와 코루틴)에서 다룬다. 이 장에서는 "`runCatching`이 `Throwable`을 잡으므로 취소 신호까지 삼킨다"는 사실만 기억해 두자.

### 8.4 Result를 반환하거나 저장하는 경우

`Result`는 원래 지역적인 예외 처리를 표현하기 위해 설계됐다. 초기 버전에서는 `Result`를 함수의 **반환 타입으로 직접 쓰는 것을 컴파일러가 제한**했다. 이 제약은 이후(대략 Kotlin 1.5 즈음) 완화되어 반환 타입으로 쓸 수 있게 됐지만, 설계 의도의 뉘앙스는 남아 있다.

> [!info] 역사 메모
> `kotlin.Result`를 처음 도입할 때 언어팀은 이것이 "함수 경계를 넘어 실패를 전파하는 범용 결과 타입"으로 쓰이는 것을 경계했다. 공개 API의 반환 타입으로 `Result<T>`를 쓰면 "이 함수가 어떤 실패를 담을 수 있는가"가 타입에 드러나지 않는다. `Result<Data>`에는 어떤 예외든 들어갈 수 있으므로, 검사 예외가 주던 문서화 이점(4.1절)을 되살리지 못한다.
>
> 도메인 실패를 API 경계에서 표현하고 싶다면 `Result`보다 **의미가 드러나는 `sealed` 결과 타입**이 낫다는 것이 권장 방향이다(9절). `Result`와 `runCatching`은 "여러 예외를 지역적으로 한데 모아 다루는" 용도에 가장 잘 맞는다.

정리하면 `Result`의 쓰임은 다음과 같다.

- **좋은 쓰임**: 지역적으로 "이 블록이 던질 수 있는 여러 예외를 한 번에 잡아 성공/실패로 분기"할 때.
- **피할 쓰임**: 공개 API의 반환 타입으로 도메인 실패를 표현할 때(`sealed` 결과 타입을 권장한다), 코루틴에서 취소 예외를 고려하지 않을 때.

---

## 9. 예외인가 결과 타입인가: 설계 판단

### 9.1 예외가 맞는 경우: "예외적"인 상황

예외는 이름 그대로 **예외적(exceptional)** 상황을 위한 도구다. 즉 정상 흐름에서 일어나서는 안 되는 프로그래머의 실수나, 회복할 수 없는 환경 오류에 쓴다.

- **프로그래밍 계약 위반**: `require`/`check`가 던지는 `IllegalArgumentException`/`IllegalStateException`이 여기에 해당한다. 이는 버그이므로 잡아서 복구할 것이 아니라 고쳐야 한다.
- **회복할 수 없는 시스템 오류**: `OutOfMemoryError` 등이다. 잡지 않는다.
- **매우 드물고, 호출자가 대개 복구할 수 없는 실패**: 던져서 위로 올려보낸다.

예외는 정상 흐름에서 벗어나 스택을 되감는다는 특징이 있다. 자주 일어나는 일에 스택 트레이스 캡처 같은 무거운 장치를 쓰는 것은 성능과 가독성 모두에서 손해다.

### 9.2 결과 타입이 맞는 경우: "예상된" 실패

반대로 **정상적으로 예상되는 실패**는 예외가 아니라 **값으로 반환**하는 것이 Kotlin다운 접근이다. "사용자 입력 파싱 실패", "찾는 항목 없음", "검증 규칙 위반" 같은 실패는 예외적이지 않다. 흔히 일어나며, 호출자가 반드시 다뤄야 한다. 이런 실패는 `sealed` 계층으로 모델링해 타입에 담는다.

```kotlin
sealed interface ParseResult {
    data class Success(val value: Int) : ParseResult
    data class Invalid(val input: String, val reason: String) : ParseResult
    data object Empty : ParseResult
}

fun parse(s: String): ParseResult = when {
    s.isBlank() -> ParseResult.Empty
    s.toIntOrNull() == null -> ParseResult.Invalid(s, "정수 아님")
    else -> ParseResult.Success(s.trim().toInt())
}

// 호출자는 when 완전성 검사로 모든 경우 처리를 컴파일러에게 강제당한다
val message = when (val r = parse(userInput)) {
    is ParseResult.Success -> "값: ${r.value}"
    is ParseResult.Invalid -> "오류(${r.reason}): ${r.input}"
    ParseResult.Empty -> "비어 있음"
    // else 불필요 — sealed 라서 컴파일러가 완전성을 안다
}
```

이것이 4.3절에서 말한, 검사 예외의 이점을 다른 방식으로 되살리는 접근이다. Java의 검사 예외는 "실패를 처리하라"는 요구를 시그니처의 `throws`로 강제했지만, 예외를 삼키는 습관을 낳았다.

Kotlin의 `sealed` 결과 타입은 같은 요구를 **`when` 완전성 검사**로 강제하면서, 각 실패가 무엇인지를 타입으로 명확히 드러낸다. 컴파일러가 "이 경우를 빠뜨렸다"고 알려 주므로 실패를 삼킬 수 없다. `sealed`와 ADT 설계는 30장(sealed 클래스와 대수적 데이터 타입)에서 자세히 다룬다.

### 9.3 비교 표

| 축 | 예외(throw/catch) | 결과 타입(sealed) | `Result`/runCatching |
|----|-------------------|--------------------|----------------------|
| 실패의 성격 | 예외적·드묾·버그 | 예상됨·흔함·도메인 | 지역적으로 여러 예외 취합 |
| 처리 강제 | 컴파일러 강제 없음 | `when` 완전성으로 강제 | `fold` 등으로 명시적 |
| 실패 정보 | 예외 타입·메시지 | 각 케이스가 타입으로 | Throwable 하나 |
| 제어 흐름 | 스택 되감기(비지역) | 정상 반환(지역) | 정상 반환(값) |
| 성능(JVM) | 스택 트레이스 캡처 비용 | 무겁지 않음 | 값 클래스, 대개 가벼움 |
| 가독성 | 흐름이 흩어짐 | 흐름이 선형 | 파이프라인 |

### 9.4 판단 지침

실무 지침은 세 가지 질문으로 정리할 수 있다. **"호출자가 이 실패를 정상적으로 예상하고 다뤄야 하는가?"** 그렇다면 결과 타입으로 반환한다. **"이 실패는 버그이거나 회복할 수 없는 환경 오류인가?"** 그렇다면 예외를 던진다. **"여러 종류의 예외를 한곳에서 한꺼번에 잡아 값으로 다루고 싶은가?"** 그렇다면 `runCatching`을 쓰되, 코루틴에서는 취소 예외를 다시 던진다(8.3절).

이 판단은 언어가 대신해 주지 않는다. Kotlin은 검사 예외를 없애면서(4절) 이 결정을 온전히 개발자에게 맡겼다. 그 자유의 대가는 규율이다.

---

## 10. 코루틴 취소와 예외: 다음 장으로 이어지는 지점

### 10.1 취소는 예외로 구현된다

이 장을 마무리하면서 다음 장인 42장(suspend와 코루틴)과 이어지는 내용을 하나 짚어 둔다. 코루틴에서 **취소(cancellation)는 `CancellationException`이라는 특별한 예외를 던지는 방식으로 구현된다.** 코루틴이 취소되면 그 코루틴의 중단 지점(suspension point)에서 `CancellationException`이 던져진다. 이 예외가 스택을 되감으며 (리소스 정리를 위해) `finally` 블록들을 실행하고 코루틴을 정리한다.

이렇게 설계했기 때문에 취소는 예외 장치를 그대로 재사용한다. `try`/`finally`가 정리 코드를 실행하고, 억압 규칙과 전파 규칙도 그대로 적용된다. 하지만 `CancellationException`은 진짜 오류가 아니라 **"정상적인 협조적 취소 신호"** 라는 점에서 다른 예외와 근본적으로 다르다.

### 10.2 취소 예외를 잡으면 안 되는 이유

바로 그 때문에 넓은 `catch`로 `CancellationException`을 잡아 삼키면 취소 메커니즘이 깨진다.

```kotlin
// 안티패턴 — 취소를 삼켜 코루틴이 죽지 않는다
suspend fun broken() {
    try {
        doSuspendingWork()
    } catch (e: Exception) {   // CancellationException 은 Exception 의 하위타입! 삼켜진다.
        log.error("작업 실패", e)   // 취소 신호를 오류로 오인해 로깅하고 삼킴
    }
    // 취소됐어야 할 코루틴이 여기로 계속 진행한다 — 구조적 동시성 붕괴
}
```

`CancellationException`은 `IllegalStateException`을 거쳐 `Exception`의 하위 타입이 된다. 그래서 `catch (e: Exception)`이나 `runCatching`(8.3절)이 이 예외를 잡아 버린다. 올바른 코드는 취소 예외를 식별해 반드시 다시 던진다.

```kotlin
suspend fun fixed() {
    try {
        doSuspendingWork()
    } catch (e: CancellationException) {
        throw e                    // 취소는 무조건 전파
    } catch (e: Exception) {
        log.error("작업 실패", e)   // 진짜 오류만 여기서 처리
    }
}
```

### 10.3 이 장에서 다루지 않는 것

협조적 취소(cooperative cancellation) 메커니즘, `Job` 계층에서의 예외 전파, `CoroutineExceptionHandler`, `SupervisorJob`의 격리, `withContext(NonCancellable)`로 정리 코드를 보호하는 방법은 42장(suspend와 코루틴)과 [[43 - 동시성과 메모리 모델]], 그리고 Kotlin Coroutines 시리즈의 [[06 - CoroutineScope와 구조적 동시성]]에서 다룬다.

이 장에서 다진 것은 그 논의의 토대다. **취소가 예외로 구현되므로, 이 장에서 배운 예외 처리 규율(무엇을 잡을지, 무엇을 다시 던질지, `finally`가 언제 실행되는지)은 코루틴에도 그대로, 그리고 더 엄격하게 적용된다.**
