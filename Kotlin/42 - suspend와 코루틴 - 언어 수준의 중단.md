---
title: suspend와 코루틴 - 언어 수준의 중단
date: 2026-07-13
tags: [kotlin, coroutines, suspend, continuation, cps, 학습노트]
---

[[41 - 예외와 Nothing]]에서는 코틀린이 검사 예외(checked exception)를 없애고, `throw`를 `Nothing` 타입의 표현식으로 다루며, `try`를 문이 아닌 표현식으로 취급한다는 것을 보았다. 예외는 정상 흐름을 *위로* 강제로 되감는(unwind) 제어 흐름이다.

이 장에서는 그와 반대되는 제어 흐름, 즉 정상 흐름을 *멈췄다가 나중에 같은 자리에서 다시 시작하는* 기능인 **중단(suspension)** 을 다룬다. 예외는 스택을 무너뜨리며 위로 올라가지만, 중단은 스택의 한 지점을 통째로 보관해 두었다가 원래 자리로 정확히 되돌린다.

이 장에서는 언어와 컴파일러 수준의 중단 메커니즘을 다룬다. `suspend` 키워드가 타입 시스템에서 무엇을 뜻하는지, 컴파일러가 이를 어떤 코드(CPS 변환과 상태 기계)로 바꾸는지, `Continuation` 인터페이스가 왜 핵심인지, 표준 라이브러리가 보장하는 것과 `kotlinx.coroutines`가 그 위에 만든 것의 경계가 어디인지를 설명하고, 중단 가능이 비동기라는 오해와 `suspend`가 마법이라는 오해를 바로잡는다.

코루틴 빌더, `CoroutineScope`와 구조적 동시성, `Dispatchers`, `Job`, `Flow`, 취소는 Kotlin Coroutines 시리즈([[02 - suspend와 Continuation - CPS와 상태 기계]], [[06 - CoroutineScope와 구조적 동시성]], [[08 - 코루틴 빌더 - launch async runBlocking withContext]])에서, 메모리 모델과 스레드 안전은 [[43 - 동시성과 메모리 모델]]에서 다룬다.

---

## 1. 함수의 색깔: 블로킹과 콜백 사이의 문제

### 1.1 블로킹이라는 낭비

프로그램이 네트워크 응답, 디스크 읽기, 타이머, 다른 계산의 완료처럼 시간이 걸리는 일을 기다려야 할 때 가장 단순한 방법은 **블로킹(blocking)** 이다. 스레드가 그 자리에 멈춰서 결과가 올 때까지 아무것도 하지 않는다.

```kotlin
// 블로킹 방식 (개념 예시)
fun loadProfile(id: Int): Profile {
    val user = fetchUserBlocking(id)       // 이 줄에서 스레드가 멈춘다 (예: 200ms)
    val avatar = fetchAvatarBlocking(user) // 여기서 또 멈춘다 (예: 150ms)
    return Profile(user, avatar)
}
```

이 코드는 읽기 쉽다. 위에서 아래로 한 줄씩 자연스러운 순서로 실행된다. 문제는 하나뿐인데, 기다리는 동안 **스레드 하나가 통째로 놀고 있다**는 것이다. 스레드는 비싼 자원이다. JVM 백엔드에서 스레드 하나는 보통 수백 KB에서 1MB 정도의 스택 메모리를 예약하고, 운영체제 커널이 스케줄링하는 대상이다.

수만 개의 요청을 동시에 처리하려면 수만 개의 스레드가 필요하고, 그러면 메모리와 문맥 전환(context switch) 비용 때문에 금세 한계에 부딪힌다. 블로킹이 낭비인 이유는 여기에 있다. `fetchUserBlocking`이 200ms를 기다리는 동안 그 스레드는 CPU를 쓰지 않으면서도 자기 자원(스택과 커널 슬롯)을 계속 점유한다. 기다림은 CPU 작업이 아닌데도 스레드를 잡아 둔다.

> [!caution] 성능 주의
> "스레드를 더 만들면 되지 않나"라는 답은 어느 정도까지만 통한다. 스레드 수가 코어 수를 크게 넘어서면 실제 병렬성은 늘지 않고 문맥 전환 오버헤드만 커진다. I/O 대기가 많은 서버가 "요청당 스레드 하나" 모델에서 확장성의 한계에 부딪히는 이유가 이것이다.

### 1.2 콜백 지옥과 Future의 한계

블로킹을 피하는 고전적인 방법은 **콜백(callback)** 이다. "일이 끝나면 이 함수를 호출해 줘"라고 등록해 두고, 스레드는 바로 반환해 다른 일을 한다.

```kotlin
// 콜백 방식 (개념 예시)
fun loadProfile(id: Int, onResult: (Profile) -> Unit) {
    fetchUser(id) { user ->
        fetchAvatar(user) { avatar ->
            onResult(Profile(user, avatar))
        }
    }
}
```

이제 스레드는 낭비되지 않는다. `fetchUser`는 요청만 보내고 바로 반환하며, 응답이 오면 콜백이 (아마 다른 스레드에서) 실행된다. 하지만 치르는 대가가 크다. 순차적인 흐름이 **중첩된 콜백의 피라미드**로 바뀐다. 단계가 셋, 넷으로 늘어나면 코드는 계속 오른쪽으로 밀려난다("콜백 지옥").

더 심각한 문제는 **에러 처리와 제어 흐름이 깨진다**는 것이다. 콜백 안에서 던진 예외는 원래 호출자의 `try/catch`에 잡히지 않는다. 호출 스택이 이미 반환되었기 때문이다. 루프, 조건문, 조기 반환 같은 평범한 제어 구조를 콜백 위에서 쓰려면 코드가 매우 복잡해진다.

`Future`/`Promise`(그리고 자바의 `CompletableFuture`)는 콜백을 값으로 만들어 중첩을 줄인다. `.thenCompose{}.thenApply{}`처럼 연결해서 피라미드를 평평하게 편다.

하지만 여전히 **일반 제어 흐름을 쓸 수 없다**. `if`, `for`, `try/catch`를 Future 연결 안에서 자연스럽게 쓸 수 없고, 모든 것을 `thenCompose`/`handle` 같은 조합자(combinator)로 바꿔 써야 한다. 그 결과 코드에는 "무엇을 하는가"보다 "어떻게 연결하는가"가 더 많이 드러난다.

### 1.3 함수의 색깔: 코틀린의 답

이와 관련해 잘 알려진 관찰이 있다(밥 나이스트롬의 에세이 "What Color is Your Function"). 콜백이나 Future를 쓰는 환경에서 함수는 두 가지 "색"으로 나뉜다. 하나는 값을 그대로 반환하는 **동기 함수**이고, 다른 하나는 콜백을 받거나 Future를 반환하는 **비동기 함수**이다. 두 색은 잘 섞이지 않는다. 비동기 함수는 비동기 문맥에서만 호출할 수 있고, 동기 코드에서 비동기 결과를 쓰려면 블로킹하거나 다시 콜백을 써야 한다.

코틀린은 이 색을 **언어 문법으로 끌어올리되**, 색이 다른 함수를 *동기 함수와 똑같은 문법으로* 호출할 수 있게 만들었다. 그 색의 이름이 `suspend`이다.

```kotlin
// 코틀린의 답: 블로킹처럼 읽히지만 블로킹하지 않는다
suspend fun loadProfile(id: Int): Profile {
    val user = fetchUser(id)       // 여기서 "중단"될 수 있다 — 스레드는 멈추지 않는다
    val avatar = fetchAvatar(user) // 여기서도
    return Profile(user, avatar)
}
```

이 코드는 1.1절의 블로킹 코드와 **글자 그대로 똑같이** 생겼다. 위에서 아래로 실행되고, `if`, `for`, `try/catch`가 모두 자연스럽게 동작한다. 그런데도 `fetchUser`가 응답을 기다리는 동안 스레드는 멈추지 않고 풀려나 다른 일을 할 수 있다. 콜백의 효율성과 블로킹의 가독성을 함께 얻는 것, 이것이 `suspend`가 존재하는 이유이다.

$$
\text{suspend} \;=\; \underbrace{\text{콜백의 논블로킹}}_{\text{스레드를 놓아준다}} \;+\; \underbrace{\text{블로킹의 순차 가독성}}_{\text{일반 제어 흐름을 쓴다}}
$$

스레드를 멈추지 않으면서 함수를 "멈췄다가 다시 이어 가는" 일이 어떻게 가능한지가 이 장의 핵심이다. 하지만 그 전에 이 새로운 색이 정확히 무엇을 *의미하는지*부터 분명히 해야 한다. 코틀린을 배우는 사람이 가장 흔히 오해하는 부분이 바로 여기이기 때문이다.

---

## 2. suspend 키워드의 의미

### 2.1 "중단 가능"의 정확한 뜻

`suspend`는 함수 선언 앞에 붙는 변경자(modifier)이다. 이 변경자가 함수에 주는 능력은 하나뿐이다. **실행 도중에 중단(suspend)했다가 나중에 그 지점부터 재개(resume)될 수 있다**는 것이다. 그 이상도 그 이하도 아니다.

좀 더 정확히 설명해 보자. 일반 함수는 한 번 시작하면 값을 반환하거나 예외를 던질 때까지 자기 스레드를 점유한 채 끝까지 실행된다. `suspend` 함수는 실행 도중에 **중단점(suspension point)** 을 가질 수 있다. 중단점에서 함수는 자신의 실행 상태(어디까지 실행했는지, 지역 변수 값이 무엇인지)를 객체에 저장하고, 제어를 호출자에게 반환하며, 자기 스레드를 놓아준다. 나중에 누군가 저장된 상태를 되살리면(재개하면) 함수는 중단했던 바로 그 줄부터 계속 실행된다.

핵심은 "**될 수 있다**"는 표현이다. `suspend fun`이라고 해서 반드시 중단하는 것은 아니다. 실제로 중단이 일어나는 곳은 **다른 `suspend` 함수를 호출하는 지점뿐**이고, 그마저도 그 호출이 실제로 중단을 선택했을 때만 일어난다. 아래 함수는 `suspend`이지만 중단점이 하나도 없으므로 절대 중단하지 않는다.

```kotlin
suspend fun addLabeled(a: Int, b: Int): Int {
    val sum = a + b   // 일반 계산 — 중단점 없음
    return sum        // 이 함수는 suspend지만 실제로는 절대 중단하지 않는다
}
```

이 함수에는 `suspend`를 붙일 이유가 사실상 없다(호출 규칙만 제약할 뿐이다). `suspend`가 의미를 갖는 것은 함수 본문 안에서 *다른* `suspend` 함수를 호출할 때이다.

### 2.2 중단은 비동기가 아니다

이제 첫 번째 오해를 바로잡아 보자.

> [!warning] 흔한 오해
> "`suspend` 함수는 비동기다. `suspend`를 붙이면 코드가 백그라운드에서 돌고, 새 스레드가 생기고, 호출자는 기다리지 않고 계속 진행한다."

**모두 틀렸다.** `suspend`는 비동기성을 뜻하지 않는다. `suspend` 함수를 호출하면 호출자는 그 함수가 **완료될 때까지 (중단을 포함해) 순차적으로 기다린다**. 반환값을 받아야 다음 줄로 넘어간다. 이것은 일반 함수 호출과 정확히 같은 의미이다.

```kotlin
suspend fun demo() {
    println("A")
    val user = fetchUser(1)   // 이 호출이 끝나야
    println("B: $user")       // 이 줄이 실행된다 — 순서 보장
    val avatar = fetchAvatar(user)
    println("C: $avatar")     // A → B → C 순서로만 출력된다
}
// 출력은 항상: A, B: ..., C: ...  (이 순서로)
```

`fetchUser`가 내부에서 네트워크를 기다리며 중단하더라도, `demo`의 입장에서는 그저 "값을 돌려주는 함수 호출"일 뿐이다. `println("B")`는 `fetchUser`가 완전히 끝난 뒤에만 실행된다. 중단은 이 순차성을 깨지 않는다. 기다리는 방식이 "스레드를 점유한 채"가 아니라 "스레드를 놓아준 채"라는 점만 다르다.

여러 일이 동시에 진행되는 비동기성을 원한다면, 그것은 `suspend`가 아니라 **코루틴 빌더가 명시적으로 제공하는 기능**이다. `kotlinx.coroutines`의 `launch`나 `async`로 별도의 코루틴을 시작해야 비로소 병렬 실행이 생긴다.

```kotlin
import kotlinx.coroutines.*

suspend fun loadBoth(a: Int, b: Int): Pair<User, User> = coroutineScope {
    val ua = async { fetchUser(a) }   // 명시적으로 새 코루틴 → 병렬 시작
    val ub = async { fetchUser(b) }   // 또 하나 → 병렬
    ua.await() to ub.await()          // 둘 다 기다림
}
```

여기서 두 `fetchUser`가 병렬로 실행되는 것은 `async`라는 *빌더* 때문이지 `suspend` 때문이 아니다. `async`를 빼고 `fetchUser(a); fetchUser(b)`라고 쓰면 두 호출은 순차적으로 실행된다. **`suspend`는 병렬 실행을 만들지 않는다. 병렬 실행은 항상 명시적으로 요청해야 한다.**

이 사실은 코루틴을 안전하게 쓰는 데 매우 중요하다. "숨은 병렬 실행"이 없다는 것은 코드를 위에서 아래로 읽은 순서가 곧 실행 순서라는 뜻이다.

### 2.3 스레드를 만들지 않는다: 스레드는 디스패처가 정한다

비슷한 오해가 하나 더 있다. "`suspend`를 붙이면 백그라운드 스레드가 뜬다"는 생각이다. 그렇지 않다. `suspend` 함수는 **자기가 실행될 스레드를 스스로 정하지 않는다**. 함수는 자기를 호출한 쪽(정확히는 자기를 *재개한* 쪽)의 스레드에서 실행될 뿐이다.

코루틴 코드가 어느 스레드에서 재개될지를 정하는 것은 `suspend` 키워드가 아니라 **디스패처(dispatcher)** 이다. 디스패처는 `kotlinx.coroutines`가 제공하는 라이브러리 개념으로, 코루틴 문맥(`CoroutineContext`)을 이루는 요소 중 하나이다.

`Dispatchers.Default`(CPU 작업용 스레드 풀), `Dispatchers.IO`(블로킹 I/O용 큰 풀), `Dispatchers.Main`(UI 스레드) 등이 있고, `withContext(Dispatchers.IO) { ... }`로 특정 블록을 특정 스레드 풀에서 실행하도록 지정할 수 있다.

```kotlin
suspend fun loadFromDisk(): ByteArray = withContext(Dispatchers.IO) {
    // 이 블록은 IO 디스패처의 스레드에서 실행된다
    readFileBlocking("data.bin")
}
```

요점은 **스레드 선택이 순전히 라이브러리 기능**이라는 것이다. 언어의 `suspend` 자체는 스레드에 대해 아무것도 모른다. `suspend fun`을 정의하고 디스패처를 전혀 지정하지 않으면, 그 함수는 자기를 재개한 스레드에서 그대로 실행된다. 새 스레드는 어디에서도 생기지 않는다.

```text
오해된 그림 (M33):
    suspend fun f()  ──▶  [새 스레드 생성]  ──▶  백그라운드 실행  (X)

실제 그림:
    suspend fun f()  ──▶  현재 코루틴의 디스패처가 정한 스레드에서 실행
                          디스패처가 없으면 재개한 쪽의 스레드 그대로
                          스레드 전환은 오직 withContext/디스패처가 명시할 때만
```

### 2.4 함수의 색깔로서의 suspend: 타입의 일부

`suspend`는 단순한 표시가 아니라 **함수 타입의 일부**이다. `suspend () -> Unit`과 `() -> Unit`은 서로 다른 타입이고, 서로 대입할 수 없다. 전자에서 후자로도, 후자에서 전자로도 자동으로 변환되지 않는다(자세한 내용은 6절에서 다룬다). 이 "색"이 타입에 들어 있으므로 컴파일러는 `suspend` 함수를 잘못된 문맥에서 호출하는 것을 컴파일 타임에 막을 수 있다(3절).

`suspend`를 함수의 색으로 보는 관점은 이 장 전체에 걸쳐 적용된다. 색은 "이 함수는 중단할 수 있다"는 정보를 타입에 담아 전달하고, 컴파일러는 이 정보를 근거로 CPS 변환(4절)을 적용하고 호출 규칙(3절)을 강제한다.

밥 나이스트롬이 지적한 "색이 전염된다"는 문제, 즉 비동기 함수를 호출하는 함수도 비동기가 되어야 한다는 문제는 코틀린에도 여전히 있다. `suspend` 함수를 호출하는 함수는 자신도 `suspend`여야 한다. 코틀린은 *같은 문법으로* 호출하게 해서 전염의 불편을 최소화했을 뿐, 전염 자체를 없애지는 못했다. 그것이 중단의 본질이므로 없앨 수도 없다.

> [!note] 명세 기준
> `suspend`는 함수, 람다, 함수 타입에 붙는 변경자이다. 프로퍼티 게터/세터에는 붙일 수 없고(코틀린 2.x 기준으로 일반 프로퍼티 접근자는 `suspend`가 될 수 없다), 생성자에도 붙일 수 없다. `main` 함수에는 붙일 수 있다(3.3절). 즉 `suspend`를 붙일 수 있는 곳은 제한되어 있으며, 이는 "중단은 함수 호출 경계에서만 일어난다"는 모델과 일치한다.

---

## 3. 어디서 호출할 수 있는가: 중단 문맥

### 3.1 suspend 함수는 suspend 문맥에서만 호출한다

`suspend` 함수 호출에는 엄격한 규칙이 있다. **`suspend` 함수는 다른 `suspend` 함수(또는 `suspend` 람다)의 본문에서만 호출할 수 있다.** 일반(비-suspend) 함수 안에서는 호출할 수 없다.

```kotlin
suspend fun fetchUser(id: Int): User { /* ... */ }

suspend fun caller1() {
    val u = fetchUser(1)   // OK — caller1도 suspend
}

fun caller2() {
    val u = fetchUser(1)   // 컴파일 에러: suspend 함수는 코루틴 또는 다른 suspend 함수에서만 호출 가능
}
```

이 규칙의 근거는 4절에서 자세히 드러나지만, 직관적으로는 이렇다. 중단하려면 "재개할 방법"이 필요하다. 즉 이어서 실행할 계산을 담은 객체(`Continuation`)를 넘겨받아야 한다. 그 객체는 `suspend` 문맥에만 존재한다. 일반 함수에는 그런 객체가 없으므로 중단할 수 없고, 따라서 중단할 수 있는 함수를 호출할 수도 없다.

이 "색의 전염"은 호출 체인의 위쪽으로 퍼진다. `fetchUser`를 호출하는 함수는 `suspend`여야 하고, 그 함수를 호출하는 함수도 `suspend`여야 한다. 그렇다면 이 사슬은 어디에서 끝날까? 어딘가에는 `suspend` 세계와 일반 세계의 경계가 있어야 한다. 그 경계가 **코루틴 빌더**이다.

### 3.2 코루틴 빌더: 일반 세계와 suspend 세계를 잇는 다리

**코루틴 빌더(coroutine builder)** 는 인자로 `suspend` 람다를 받는 일반 함수이다. 빌더는 일반 세계에서 `suspend` 세계로 건너가는 다리 역할을 한다. 빌더 자신은 `suspend`가 아니므로 어디서나 호출할 수 있고, 인자로 받은 `suspend` 람다를 실행할 코루틴을 만든다.

`kotlinx.coroutines`가 제공하는 가장 기본적인 빌더는 다음과 같다.

```kotlin
import kotlinx.coroutines.*

fun main() {                      // 일반 함수
    runBlocking {                 // 빌더: 여기 안은 suspend 세계
        val u = fetchUser(1)      // OK — runBlocking의 람다는 suspend
        println(u)
    }                             // runBlocking은 코루틴이 끝날 때까지 현재 스레드를 블로킹
}
```

| 빌더 | 제공처 | 반환 | 용도 |
|------|--------|------|------|
| `runBlocking { }` | kotlinx.coroutines | 람다 결과 | 일반 코드에서 코루틴을 시작하고 끝날 때까지 **블로킹** (main, 테스트용) |
| `launch { }` | kotlinx.coroutines | `Job` | 결과가 없는 코루틴을 시작만 하고 넘어감(fire-and-forget) |
| `async { }` | kotlinx.coroutines | `Deferred<T>` | 결과가 있는 코루틴, `.await()`로 결과를 받음 |
| `coroutineScope { }` | kotlinx.coroutines | 람다 결과 | 자신도 `suspend`이며, 자식 코루틴을 기다리는 스코프 |

여기서 언어와 라이브러리의 경계가 처음 드러난다. **빌더는 언어 기능이 아니라 라이브러리 함수이다.** `runBlocking`, `launch`, `async`는 모두 `kotlinx.coroutines`에 있다. 하지만 이들이 내부에서 하는 일, 즉 `suspend` 람다를 받아 실제로 실행을 시작하는 일은 언어와 표준 라이브러리가 제공하는 기본 요소(primitive)를 바탕으로 한다. 그 기본 요소가 `startCoroutine`이다(7.1절).

> [!info] 역사 메모
> 코루틴은 코틀린 1.1(2017)에서 실험 기능으로 등장했고 `kotlin.coroutines.experimental` 패키지에 있었다. 1.3(2018)에서 안정화되면서 `kotlin.coroutines`로 옮겨졌고, `suspend`의 CPS 변환과 `Continuation` 인터페이스가 언어와 표준 라이브러리에 확정되었다. 반면 빌더, 스코프, 디스패처, `Flow`는 처음부터 별도 라이브러리인 `kotlinx.coroutines`로 분리되었다.
>
> 이 분리는 의도적이다. 언어는 "중단"이라는 최소한의 메커니즘만 제공하고, 그 위의 동시성 정책은 라이브러리가 자유롭게 발전시키도록 한 것이다.

### 3.3 suspend main과 진입점

코틀린 1.3부터는 `main` 함수에 `suspend`를 붙일 수 있다.

```kotlin
suspend fun main() {
    val u = fetchUser(1)   // main이 suspend이므로 OK
    println(u)
}
```

이것은 별도의 빌더 없이 `suspend` 세계로 들어가게 해 주는 편의 기능이다. JVM 백엔드 기준으로, 컴파일러는 `suspend fun main`을 발견하면 실제 JVM 진입점인 `public static void main(String[])`를 합성한다. 그리고 그 안에서 `suspend main`을 코루틴으로 시작한 뒤, 완료될 때까지 이벤트 루프를 돌며 기다리는 코드를 생성한다.

즉 `suspend main`은 "언어가 자동으로 넣어 주는 빌더"라고 할 수 있다. 개념적으로는 `runBlocking { realMain() }`과 비슷하지만, 표준 라이브러리의 최소 실행기(runner)를 쓰며 `kotlinx.coroutines`에 의존하지 않는다.

### 3.4 suspend 람다와 인라인의 상호작용

`suspend` 문맥은 함수뿐 아니라 람다에도 적용된다. 코루틴 빌더가 받는 것이 바로 `suspend` 람다이다.

```kotlin
// launch의 시그니처(개념적으로):
// fun CoroutineScope.launch(block: suspend CoroutineScope.() -> Unit): Job

launch {
    // 이 람다는 suspend CoroutineScope.() -> Unit
    fetchUser(1)   // 여기서 suspend 호출 OK
}
```

여기에는 [[15 - 인라인 함수와 reified]]와 관련된 미묘한 상호작용이 하나 있다. **인라인 함수에 넘긴 람다는 인라인되므로, 그 람다 안에서 `suspend` 함수를 호출하려면 인라인 함수를 호출하는 문맥이 `suspend`여야 한다.** 예를 들어 `list.forEach { ... }`의 `forEach`는 인라인 함수이므로, 그 람다는 호출 지점에 펼쳐진다. 따라서 아래 코드는 문제없이 동작한다.

```kotlin
suspend fun processAll(ids: List<Int>) {
    ids.forEach { id ->          // forEach는 inline → 람다가 여기 펼쳐짐
        val u = fetchUser(id)    // OK — 펼쳐진 자리가 suspend 문맥
        println(u)
    }
}
```

`forEach`가 인라인 함수가 아니었다면 그 람다는 별도의 `(Int) -> Unit` 함수 객체가 되고, 그 안에서는 `suspend` 함수를 호출할 수 없었을 것이다(`(Int) -> Unit`은 `suspend`가 아니기 때문이다).

인라인 덕분에 표준 라이브러리의 수많은 고차 함수(`map`, `filter`, `let`, `run`, `repeat` 등, 대부분 인라인)를 `suspend` 코드에서 자연스럽게 쓸 수 있다. 반대로 인라인이 아닌 고차 함수 안에서 `suspend` 함수를 호출하려면, 그 고차 함수가 `suspend` 람다를 받도록 설계되어 있어야 한다.

```text
inline 고차 함수 (forEach, map, let, run, repeat, ...)
    └─ 람다가 호출 지점에 펼쳐짐 → 바깥이 suspend면 람다 안에서도 suspend 호출 OK

비-inline 고차 함수 (일반 (T) -> R 파라미터)
    └─ 람다가 별도 함수 객체 → suspend 호출 불가 (그 람다는 suspend 타입이 아님)
```

---

## 4. CPS 변환: Continuation이 핵심이다

### 4.1 suspend는 마법이 아니다

이제 두 번째 오해를 바로잡을 차례이다.

> [!warning] 흔한 오해
> "`suspend`는 컴파일러의 마법이다. 함수를 어떻게 멈췄다가 되살리는지는 블랙박스다. 스레드를 얼려 두는 특수한 런타임 트릭이 있을 것이다."

**마법이 아니다.** `suspend`는 순전히 **컴파일러가 수행하는 소스 변환**이다. 이 변환을 **CPS(Continuation-Passing Style, 연속 전달 방식) 변환**이라고 한다. 스레드를 얼려 두는 특수한 런타임 기능은 없다(적어도 프로젝트 룸 이전의 표준 코틀린 코루틴에는 없다. 9.4절의 성능 설명을 참고하라). 컴파일러가 하는 일은 두 가지뿐이다.

1. 모든 `suspend` 함수에 **숨은 파라미터 `Continuation`을 추가**한다.
2. `suspend` 함수의 본문을 중단점마다 상태를 저장하고 재개할 수 있는 **상태 기계(state machine)** 로 바꾼다(5절).

이 두 변환을 이해하면 코루틴의 "마법"은 완전히 사라진다. 남는 것은 평범한 객체와 평범한 메서드 호출, 평범한 `when` 분기뿐이다.

### 4.2 Continuation 인터페이스

CPS의 핵심은 표준 라이브러리 인터페이스인 `Continuation`이다. "연속(continuation)"은 "이 지점 이후에 이어질 계산 전체"를 하나의 객체로 구체화(reify)한 개념이다.

```kotlin
// kotlin.coroutines 패키지 (표준 라이브러리)
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```

각 멤버의 의미를 살펴보자.

- `context`: 이 연속이 속한 코루틴 문맥(`CoroutineContext`)이다. 디스패처, `Job`, 이름 등이 여기에 담긴다(자세한 내용은 Kotlin Coroutines 시리즈의 [[04 - CoroutineContext - 문맥의 대수]]에서 다룬다).
- `resumeWith(result: Result<T>)`: **"멈췄던 계산을 이 결과로 재개하라"** 는 명령이다. `Result<T>`는 성공(`T` 값)이거나 실패(예외)이다. 즉 값으로도 재개할 수 있고 예외로도 재개할 수 있다.

표준 라이브러리는 편의를 위해 확장 함수 두 개를 제공한다.

```kotlin
fun <T> Continuation<T>.resume(value: T)                    // = resumeWith(Result.success(value))
fun <T> Continuation<T>.resumeWithException(e: Throwable)   // = resumeWith(Result.failure(e))
```

`Continuation<T>`를 "타입 `T`를 받는 콜백"이라고 생각해도 된다. `resume(value)`는 "결과가 준비되었으니 이 값으로 계산을 이어 가라"는 콜백 호출이고, `resumeWithException(e)`는 "이어질 계산 자리에서 예외를 던져라"라는 뜻이다. 일반 콜백과 다른 점은, 이 콜백이 *중단되었던 함수의 나머지 전체*를 되살린다는 것이다. 결국 `Continuation`은 **함수에서 "이어서 실행할 나머지 부분"을 담은 콜백**이다.

### 4.3 컴파일러가 추가하는 숨은 파라미터

CPS 변환의 첫 번째 규칙은 모든 `suspend` 함수의 시그니처에 `Continuation` 파라미터가 하나 추가된다는 것이다. 그리고 **반환 타입이 `Any?`로 바뀐다**.

```kotlin
// 우리가 쓰는 것 (소스):
suspend fun fetchUser(id: Int): User

// 컴파일러가 만드는 것 (JVM 백엔드, 개념적 시그니처):
fun fetchUser(id: Int, completion: Continuation<User>): Any?
//                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  숨은 파라미터
//                                                 반환: Any? (User 또는 특수 마커)
```

```java
// Java에서 이 함수를 본다면 대략 이렇게 보인다:
Object fetchUser(int id, Continuation<? super User> completion);
```

이 변환으로 3.1절의 호출 규칙이 설명된다. `fetchUser`를 호출하려면 `Continuation<User>`를 넘겨야 한다. 그렇다면 그 `Continuation`은 어디서 얻을까? 바로 **자기를 호출한 `suspend` 함수의 숨은 `Continuation` 파라미터**에서 얻는다. 일반 함수에는 이 숨은 파라미터가 없으므로 넘겨줄 것이 없고, 따라서 `suspend` 함수를 호출할 수 없다. 색의 전염은 이 파라미터 전달 방식에서 필연적으로 생기는 결과이다.

### 4.4 반환 타입이 Any?가 되는 이유: COROUTINE_SUSPENDED

반환 타입이 `User`가 아니라 `Any?`가 되는 이유는, `suspend` 함수 호출이 **두 가지 결과** 중 하나를 낼 수 있기 때문이다.

1. **즉시 값을 반환**: 함수가 중단 없이 바로 끝났다면 실제 결과값(`User`)을 반환한다.
2. **중단**: 함수가 중단하기로 했다면 값 대신 특수 마커 객체인 **`COROUTINE_SUSPENDED`** 를 반환한다.

```kotlin
// kotlin.coroutines.intrinsics 패키지
public val COROUTINE_SUSPENDED: Any   // 유일무이한 마커 객체(싱글턴)
```

`User`와 `COROUTINE_SUSPENDED`를 하나의 반환 타입으로 표현하려면 두 타입의 공통 상위 타입이 필요하다. 그래서 반환 타입이 `Any?`가 된다.

호출하는 쪽 코드는 다음과 같이 동작한다.

```text
val result = fetchUser(id, myContinuation)
if (result === COROUTINE_SUSPENDED) {
    // 함수가 중단했다 → 나도 즉시 COROUTINE_SUSPENDED를 반환하고 손을 뗀다.
    // 나중에 fetchUser가 myContinuation.resume(user)를 부르면 내가 재개된다.
    return COROUTINE_SUSPENDED
} else {
    // 함수가 즉시 끝났다 → result는 진짜 User. 계속 진행.
    val user = result as User
    ...
}
```

이 `if (result === COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED` 검사가 **중단이 스택을 따라 위로 전파되는 메커니즘**이다. 중단이 일어나면 그 마커 값이 콜 스택을 한 단계씩 올라간다. 각 `suspend` 함수는 "아래에서 중단했으니 나도 중단하자"고 판단해 자기 `Continuation`을 남겨 두고 `COROUTINE_SUSPENDED`를 반환한다.

이렇게 스택 전체가 되감기고, 맨 아래의 스레드가 풀려나 다른 일을 하러 간다. 나중에 결과가 준비되면 저장된 `Continuation` 사슬을 `resume`으로 거슬러 올라가며 스택이 복원된다.

> [!note] 명세 기준
> `COROUTINE_SUSPENDED`와 비교할 때는 반드시 참조 동일성(`===`)을 쓴다([[10 - 불리언과 동등성과 동일성]]). 이 마커는 유일한 싱글턴 객체이므로 다른 어떤 정상 반환값과도 참조가 같을 수 없다. 그래서 "중단했는가"를 모호함 없이 판정할 수 있다.
>
> `COROUTINE_SUSPENDED` 자체는 `kotlin.coroutines.intrinsics`의 공개(public) 저수준 인트린식이라 참조할 수는 있다(내부 `enum`인 `CoroutineSingletons`의 싱글턴을 노출한다). 하지만 이 마커를 정상 결과로 내놓는 것은 컴파일러가 생성한 상태 기계 코드뿐이고, 일반 `suspend` 함수의 정상 반환 경로는 이 값을 결과로 돌려주지 않는다.
>
> 즉 안전성은 "접근할 수 없다"는 데서가 아니라 "유일한 싱글턴과 `===`로 비교하고, 생성된 코드만 이 마커를 다룬다"는 데서 나온다.

---

## 5. 상태 기계: 중단점을 어떻게 기억하는가

### 5.1 문제: 중단한 뒤 어디서 재개하는가

CPS 변환의 나머지 절반은 더 흥미롭다. `suspend` 함수가 중단했다가 재개될 때, 함수는 자신이 어디까지 실행했는지를 기억해야 한다. 지역 변수 `user`의 값과 다음에 실행할 줄의 위치 같은 정보가 스레드를 놓아준 동안에도 유지되어야 한다.

일반 함수라면 이 정보는 스레드의 스택 프레임에 들어 있다. 하지만 중단하면 스택을 되감아 스레드를 놓아주므로 스택 프레임이 사라진다. 그래서 컴파일러는 이 상태를 **힙에 있는 객체**에 저장한다. 그 객체가 바로 해당 함수의 `Continuation` 구현체이자 **상태 기계(state machine)** 이다.

컴파일러는 `suspend` 함수를 다음 요소를 가진 상태 기계로 바꾼다.

- **`label` 필드**: 지금 몇 번째 중단점까지 왔는지를 나타내는 정수이다. `when(label)` 분기의 인덱스로 쓰인다.
- **스필된(spilled) 지역 변수 필드**: 중단점을 지나서도 유지되어야 하는 지역 변수는 스택이 아니라 이 객체의 필드로 옮겨진다.
- **`result` 필드**: 재개될 때 전달된 결과값을 임시로 담아 두는 자리이다.

### 5.2 디컴파일로 본 상태 기계

원본 함수는 다음과 같다.

```kotlin
suspend fun loadProfile(id: Int): Profile {
    val user = fetchUser(id)        // 중단점 1
    val avatar = fetchAvatar(user)  // 중단점 2
    return Profile(user, avatar)
}
```

컴파일러가 만드는 코드를 개념적인 코틀린 의사코드로 풀면 다음과 같다(실제 바이트코드는 이보다 복잡하지만 뼈대는 같다).

```kotlin
fun loadProfile(id: Int, completion: Continuation<Profile>): Any? {
    // 1) 상태 기계 객체 확보: 처음 진입이면 새로 만들고, 재개면 재사용
    val sm = completion as? LoadProfileStateMachine
        ?: LoadProfileStateMachine(id, completion)

    when (sm.label) {
        0 -> {                                    // ── 처음 진입
            checkResult(sm.result)                // (예외 재던지기 등)
            sm.label = 1                          // "다음엔 1번 상태로 재개하라"
            val r = fetchUser(sm.id, sm)          // fetchUser에 sm을 Continuation으로 넘김
            if (r === COROUTINE_SUSPENDED)
                return COROUTINE_SUSPENDED         // 중단 → 손을 뗌
            sm.result = r                          // 즉시 끝났으면 계속 (fall-through)
            // 아래 label==1 블록으로 흐름
        }
        1 -> {                                    // ── fetchUser가 끝나 재개된 지점
            checkResult(sm.result)
            sm.user = sm.result as User            // 스필된 변수 복원
            sm.label = 2
            val r = fetchAvatar(sm.user, sm)
            if (r === COROUTINE_SUSPENDED)
                return COROUTINE_SUSPENDED
            sm.result = r
            // 아래 label==2 블록으로 흐름
        }
        2 -> {                                    // ── fetchAvatar가 끝나 재개된 지점
            checkResult(sm.result)
            val avatar = sm.result as Avatar
            return Profile(sm.user, avatar)        // 최종 결과
        }
        else -> throw IllegalStateException()
    }
    // (실제 코드는 label 0/1이 다음 블록으로 이어지도록 짜인다)
}

// 상태 기계 = Continuation 구현체
class LoadProfileStateMachine(
    val id: Int,
    val completion: Continuation<Profile>
) : Continuation<Any?> {
    var label = 0
    var result: Any? = null
    var user: User? = null            // 중단점을 가로질러 살아남는 지역 변수

    override val context get() = completion.context

    override fun resumeWith(res: Result<Any?>) {
        this.result = res.getOrThrow()      // (개념적 단순화)
        // 자기 자신을 다시 호출 → when(label)이 저장된 지점부터 이어감
        val outcome = loadProfile(id, this)
        if (outcome !== COROUTINE_SUSPENDED) {
            // 함수가 완전히 끝났다 → 상위 completion에 결과 전달
            completion.resumeWith(Result.success(outcome as Profile))
        }
    }
}
```

코루틴의 동작은 이 코드가 전부이다. 핵심을 정리하면 다음과 같다.

- **중단점 하나가 `label` 하나이자 `when` 분기 하나이다.** 함수 본문이 중단점을 경계로 여러 조각으로 나뉘고, 각 조각이 `when`의 분기 하나가 된다.
- **중단하기 직전에 `label`을 다음 값으로 설정**하고 `Continuation`(상태 기계 자신)을 넘긴다. 그래야 재개될 때 올바른 조각으로 이동한다.
- **지역 변수는 필드로 스필된다.** `user`가 중단점 1과 2 사이에서 유지되려면 스택이 아니라 `sm.user` 필드에 있어야 한다.
- **재개는 `resumeWith`가 함수를 다시 호출하는 방식이다.** `resumeWith`는 결과를 `result`에 담고 `loadProfile`을 다시 호출한다. 그러면 `when(label)`이 저장된 지점으로 바로 분기한다.

```text
loadProfile의 상태 기계 (label의 여정):

  label=0 ──fetchUser 호출──▶ [중단] 스레드 놓아줌
                                  │
              (네트워크 응답 도착, resumeWith 호출)
                                  ▼
  label=1 ──user 복원──▶──fetchAvatar 호출──▶ [중단] 스레드 놓아줌
                                  │
              (응답 도착, resumeWith 호출)
                                  ▼
  label=2 ──avatar 복원──▶──Profile 조립──▶ completion.resumeWith(profile)  [완료]
```

### 5.3 재개 루프: 누가 resumeWith를 호출하는가

상태 기계는 스스로 재개하지 않는다. 누군가 `resumeWith`를 호출해 주어야 한다. 그 "누군가"는 **가장 아래에서 실제로 중단을 일으킨 비동기 소스**이다. 타이머, 네트워크 라이브러리의 콜백, 스레드 풀의 완료 통지 등이 여기에 해당한다.

여기서 코틀린 표준 라이브러리의 연결 함수인 `suspendCoroutine`이 등장한다(7.1절에서 자세히 다룬다). 이 함수는 콜백 기반 API를 `suspend` 함수로 감싸는 표준 도구이다.

```kotlin
// 콜백 기반 옛 API를 suspend로 감싸기
suspend fun fetchUser(id: Int): User = suspendCoroutine { cont ->
    // cont: Continuation<User>  ← 지금 이 지점의 "이어질 계산"
    legacyFetchUser(id, object : Callback {
        override fun onSuccess(u: User) = cont.resume(u)          // 성공 → 재개
        override fun onError(e: Throwable) = cont.resumeWithException(e)  // 실패 → 예외로 재개
    })
    // 이 람다가 끝나면 fetchUser는 COROUTINE_SUSPENDED를 반환하고 중단된다.
}
```

`suspendCoroutine`은 현재 지점의 `Continuation`을 꺼내 람다에 넘겨준다. 우리는 그 `cont`를 콜백에 저장해 두었다가, 나중에 결과가 오면 `cont.resume(...)`을 호출한다. 그 순간 상태 기계의 `resumeWith`가 실행되고, 중단되었던 `fetchUser`부터 그 위의 `loadProfile`까지 모두 다시 실행된다.

**콜백이 순차 코드로 바뀌는 지점이 정확히 여기이다.** 콜백 지옥의 가장 아래에서 `Continuation` 하나를 캡처해, 위로 이어진 `suspend` 호출 사슬 전체를 한 번에 연결한다.

> [!note] 명세 기준
> `cont.resume`은 **정확히 한 번** 호출해야 한다. 두 번 호출하면 하나의 코루틴을 두 번 재개하려는 것이므로 `IllegalStateException`이 발생한다. 한 번도 호출하지 않으면 그 코루틴은 영원히 중단된 채 다시 실행되지 않는다(누수). 따라서 `suspendCoroutine`으로 감쌀 때는 "콜백은 성공이든 실패든 반드시 정확히 한 번 호출된다"는 계약을 지켜야 한다.

### 5.4 꼬리 호출 최적화와 상태 기계가 없는 경우

모든 `suspend` 함수가 상태 기계로 바뀌는 것은 아니다. **중단점이 없는 `suspend` 함수**(2.1절의 `addLabeled` 같은 함수)에는 상태 기계가 필요 없다. `Continuation` 파라미터만 추가되고 본문은 그대로 컴파일된다.

또한 **마지막에 다른 `suspend` 함수를 한 번만 호출하고 그 결과를 그대로 반환하는** 경우, 컴파일러는 상태 기계를 만들지 않고 받은 `Continuation`을 그대로 넘길 수 있다(꼬리 호출 최적화의 일종이다).

```kotlin
suspend fun getUser(id: Int): User = fetchUser(id)   // 꼬리 호출 하나뿐
// → 상태 기계 없이 fetchUser(id, completion)로 곧장 위임 가능
```

중단점이 하나뿐이고 그 뒤에 아무 코드도 없으면 "재개한 뒤 이어서 할 일"이 없으므로 상태를 저장할 이유가 없다. 이 최적화는 얇은 래퍼 역할을 하는 `suspend` 함수의 오버헤드를 없앤다.

> [!caution] 성능 주의
> 상태 기계 객체는 힙에 할당된다. 중단점이 여러 개인 `suspend` 함수는 호출될 때마다 상태 기계 객체를 하나 만든다(5.2절 의사코드처럼 첫 진입에서 생성한다). 중단점이 없거나 마지막 호출만 `suspend`인 함수는 상태 기계가 필요 없어 이 할당이 없다. 이 비용은 대부분 무시할 수 있지만, 자주 실행되는 루프(hot loop)에서 잠깐씩 중단하는 함수를 수백만 번 호출한다면 GC 부담이 될 수 있다. 이 문제는 9.4절에서 다시 살펴본다.

---

## 6. suspend가 타입에 나타나는 방식

### 6.1 suspend 함수 타입

2.4절에서 말했듯이 `suspend`는 함수 타입의 일부이다. 함수 타입 `(A) -> B` 앞에 `suspend`를 붙이면 `suspend (A) -> B`라는 별개의 타입이 된다.

```kotlin
val plain: (Int) -> Int = { it * 2 }
val susp: suspend (Int) -> Int = { delay(10); it * 2 }   // delay는 suspend

// 서로 다른 타입 — 값끼리는 어느 방향으로도 대입 불가:
// val a: (Int) -> Int = susp          // 컴파일 에러: suspend를 non-suspend에 대입 불가
// val b: suspend (Int) -> Int = plain // 컴파일 에러: non-suspend 값도 suspend 타입에 대입 불가
```

일반 함수 타입 `(A) -> B`와 `suspend (A) -> B` 사이에는 **어느 방향으로도 서브타입 관계가 없다.** 이미 `(Int) -> Int` 타입을 가진 *값*을 `suspend (Int) -> Int`가 필요한 자리에 넘길 수 없고, 그 반대도 마찬가지이다. `suspend` 함수를 일반 함수 자리에 넘길 수도 없다. `suspend` 함수는 `Continuation`을 요구하므로 일반 함수 타입으로 사용할 수 없기 때문이다.

다만 예외처럼 보이는 경우가 하나 있다. **람다 리터럴**은 기대 타입에 맞춰 컴파일되므로, `suspend (Int) -> Int`가 기대되는 자리에 쓴 `{ ... }`는 (본문에 중단점이 없더라도) 처음부터 `suspend` 람다로 만들어진다. 위의 `val susp` 초기화가 바로 이 경우이다. 이것은 *기존 값을 변환*하는 문제가 아니라 *리터럴을 어느 타입으로 만들지*의 문제이므로, "값끼리는 대입할 수 없다"는 원칙과 모순되지 않는다.

```text
람다 리터럴 { ... }   ──(기대 타입에 맞춰 컴파일)──▶  suspend (A)->B  또는  (A)->B
(A)->B 값            ──(불가: 서브타입 아님)──▶  suspend (A)->B  가 기대되는 자리
suspend (A)->B 값     ──(불가: 서브타입 아님)──▶  (A)->B  가 기대되는 자리
```

두 타입 사이에 서브타입 관계가 없는 근본적인 이유는 **런타임 표현이 다르기 때문**이다(6.2절). `suspend` 함수는 `Continuation`을 받는 다른 시그니처로 컴파일되므로, 한쪽 타입의 함수 객체를 다른 쪽 타입으로 그대로 취급할 수 없다. 반면 람다 리터럴은 아직 객체로 만들어지기 전이므로, 컴파일러가 어떤 표현으로 만들지 자유롭게 정할 수 있다.

### 6.2 런타임 표현: SuspendFunctionN과 FunctionN

`suspend` 람다는 JVM 백엔드에서 어떻게 표현될까? 일반 람다 `(A) -> B`가 `Function1<A, B>` 인터페이스를 구현하는 객체가 되는 것처럼([[13 - 함수 타입과 람다와 함수 참조]]), `suspend` 람다도 함수 객체가 된다. 다만 CPS 변환 때문에 `invoke`의 시그니처에 `Continuation`이 추가된다.

개념적으로 `suspend (A) -> B` 람다는 `Continuation` 파라미터가 하나 더 붙은 `FunctionN`을 구현한다.

`suspend (A) -> B`는 `invoke(a: A, completion: Continuation<B>): Any?` 형태의 `Function2<A, Continuation<B>, Any?>`로 구현되며, 여기에 "이것은 suspend 함수이다"를 표시하는 마커가 함께 붙는다(세부 사항은 구현과 백엔드에 따라 다르므로 정확한 클래스 이름에 의존하지 않아야 한다).

또한 실제 `suspend` 람다는 상태 기계여야 하므로, JVM 백엔드에서는 표준 라이브러리의 `SuspendLambda`(→ `ContinuationImpl` → `BaseContinuationImpl`) 계열을 상속한 객체로 컴파일된다.

```text
일반 람다  (A) -> B         →  Function1<A, B> 구현 객체
suspend 람다 suspend (A)->B  →  Function2<A, Continuation<B>, Any?> 구현
                                + SuspendFunction 마커
                                + 상태 기계(SuspendLambda 계열) 상속
```

요점은 두 가지이다. 첫째, **`suspend` 람다도 결국 평범한 힙 객체**이다. 특별한 런타임 요소가 아니며, 4.1절에서 본 것처럼 마법은 없다. 둘째, `invoke`의 실제 시그니처에 `Continuation`이 들어 있으므로 `suspend` 함수 타입과 일반 함수 타입은 런타임 표현부터 다르다. 이 점이 6.1절에서 본 타입 비호환의 근거가 된다.

### 6.3 함수 참조와 리플렉션

`suspend` 함수에 대한 함수 참조 `::fetchUser`는 `suspend (Int) -> User` 타입을 갖는다. 리플렉션([[44 - 애노테이션과 리플렉션]])에서는 `KFunction`의 `isSuspend` 프로퍼티로 이 색을 확인할 수 있다.

```kotlin
import kotlin.reflect.full.*

val ref = ::fetchUser
println(ref)               // KFunction 참조
// reflection API에서 (ref as KFunction<*>).isSuspend == true
```

즉 `suspend`는 소스, 타입, 바이트코드, 리플렉션의 모든 층위에서 일관되게 추적할 수 있는 실제 속성이다. "마법"이나 단순한 표시가 아니라, 함수의 정체성에 포함된 정보이다.

---

## 7. 언어와 라이브러리의 경계

이 장의 제목이 "언어 수준의 중단"인 이유가 여기에 있다. 사람들은 흔히 "코루틴"이라는 말에 `launch`, `Dispatchers`, `Flow`까지 한데 묶어 부르지만, 그중 언어와 표준 라이브러리가 보장하는 것은 놀라울 만큼 적다. 경계를 정확히 나누어 보자.

### 7.1 표준 라이브러리(kotlin.coroutines)가 제공하는 것

컴파일러와 표준 라이브러리(`kotlin-stdlib`, `kotlin.coroutines` 패키지)가 언어 차원에서 보장하는 것은 다음이 전부이다.

| 요소 | 정체 |
|------|------|
| `suspend` 변경자 | 언어 키워드. CPS 변환과 상태 기계 생성을 일으킨다 |
| `Continuation<T>` | 인터페이스. `context` + `resumeWith` |
| `resume`/`resumeWithException` | `Continuation` 확장 함수 |
| `CoroutineContext` / `EmptyCoroutineContext` | 문맥을 나타내는 대수적 자료구조(요소들의 집합) |
| `ContinuationInterceptor` | 재개 지점을 가로채는 문맥 요소(디스패처의 상위 개념) |
| `COROUTINE_SUSPENDED` | 중단을 표시하는 마커(intrinsics) |
| `suspendCoroutine { }` | 콜백을 suspend로 감싸는 표준 도구 |
| `startCoroutine` / `createCoroutine` | `suspend` 람다의 실행을 실제로 시작하는 기본 요소 |
| `@RestrictsSuspension` | 제한된 중단 스코프를 만드는 애노테이션(7.3절) |

언어가 아는 코루틴은 이것이 전부이다. 여기에는 스레드도, 병렬성도, 스코프도, 취소도, `Flow`도 없다. "중단하고 재개하는 능력"과 이를 위한 최소한의 구성 요소만 있다.

예를 들어 `(suspend () -> T).startCoroutine(completion)`은 `suspend` 람다를 받아 실행을 시작하고, 실행이 끝나면 넘겨받은 `completion` `Continuation`을 재개한다. 모든 코루틴 빌더는 결국 이 함수를 바탕으로 만들어진다.

```kotlin
// 라이브러리 없이, 표준 라이브러리 원시만으로 코루틴을 굴리는 최소 예
import kotlin.coroutines.*

fun runToy(block: suspend () -> Unit) {
    block.startCoroutine(object : Continuation<Unit> {
        override val context = EmptyCoroutineContext
        override fun resumeWith(result: Result<Unit>) {
            result.onFailure { it.printStackTrace() }   // 완료 시 콜백
        }
    })
}
```

이 `runToy`는 `kotlinx.coroutines` 없이 표준 라이브러리만으로 만든 아주 단순한 빌더이다. `launch`나 `runBlocking`이 하는 일의 뼈대가 이것이다(물론 실제 빌더는 `Job`, 디스패처, 예외 전파, 취소를 모두 갖추고 있다).

### 7.2 kotlinx.coroutines가 제공하는 것

코루틴을 실제로 쓸 때 다루는 거의 모든 것은 **별도 라이브러리**인 `org.jetbrains.kotlinx:kotlinx-coroutines-core`에 있다. 이것은 표준 라이브러리가 아니므로 의존성을 따로 추가해야 한다.

| 요소 | 정체 (전부 kotlinx.coroutines) |
|------|------|
| `CoroutineScope` | 구조적 동시성의 단위 |
| `launch` / `async` / `runBlocking` / `coroutineScope` / `withContext` | 코루틴 빌더 |
| `Job` / `Deferred` | 코루틴 핸들과 생명주기 |
| `Dispatchers.Default/IO/Main/Unconfined` | 스레드 정책(`ContinuationInterceptor` 구현) |
| `delay` / `withTimeout` | 논블로킹 시간 지연과 타임아웃 |
| `CancellationException` / 협조적 취소 | 취소 프로토콜 |
| `Flow` / `Channel` / `StateFlow` | 비동기 스트림과 통신 |
| `suspendCancellableCoroutine` | 취소를 지원하는 `suspendCoroutine` |

이 경계는 실용적인 이유로 중요하다. `delay`를 import할 수 없어 컴파일이 안 된다면, 그것은 언어 버그가 아니라 `kotlinx.coroutines` 의존성을 추가하지 않았기 때문이다. `suspend fun`은 표준 라이브러리만으로 정의하고 컴파일할 수 있지만, 그 안에서 `delay`나 `withContext`를 쓰려면 라이브러리가 필요하다.

```text
┌─────────────────────────────────────────────────────────────┐
│ 언어 + 표준 라이브러리 (kotlin.coroutines)                    │
│   suspend, Continuation, CoroutineContext,                   │
│   COROUTINE_SUSPENDED, suspendCoroutine, startCoroutine,     │
│   @RestrictsSuspension                                        │
│   → "중단하고 재개하는 능력"만 제공                            │
└───────────────────────────┬─────────────────────────────────┘
                            │ 이 위에 세워짐
┌───────────────────────────▼─────────────────────────────────┐
│ kotlinx.coroutines (별도 라이브러리 의존성)                   │
│   CoroutineScope, launch/async, Dispatchers, Job,            │
│   delay, withContext, 취소, Flow, Channel ...                │
│   → "구조적 동시성·스레드 정책·스트림" 정책 제공              │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 제한된 중단: sequence 빌더의 원리

`suspend`가 `kotlinx.coroutines` 없이도 쓸모 있다는 것을 가장 잘 보여 주는 예가 **`sequence { }` 빌더**이다([[40 - 시퀀스와 지연 평가]]). 이 빌더는 표준 라이브러리 함수이고 내부에서 `suspend`를 쓰지만, 코루틴 라이브러리나 스레드, 비동기와는 관계가 없다.

```kotlin
val fibs = sequence {
    var a = 0; var b = 1
    while (true) {
        yield(a)             // yield는 suspend 함수!
        val next = a + b
        a = b; b = next
    }
}
println(fibs.take(6).toList())  // => [0, 1, 1, 2, 3, 5]
```

`yield`는 `suspend` 함수이다. `sequence` 블록은 `yield(a)`에서 중단해 값을 소비자에게 넘기고, 소비자가 다음 값을 요청하면 그 자리에서 재개한다. 지연 시퀀스(제너레이터)는 이런 방식으로 만들어진다. 즉 **중단과 재개라는 언어 메커니즘을 비동기가 아니라 "값을 하나씩 지연 생성"하는 데 쓰는 것이다.**

여기에는 스레드도, 디스패처도, `kotlinx.coroutines`도 전혀 없다. `suspend`가 동시성과 독립된 근본적인 기능이라는 점을 이보다 잘 보여 주는 예는 없다.

그런데 `sequence` 블록 안에서 `delay` 같은 실제 비동기 `suspend` 함수를 호출하면 안 된다. 시퀀스는 동기적인 제너레이터이지 비동기 코루틴이 아니기 때문이다. 이를 컴파일 타임에 막는 장치가 **`@RestrictsSuspension`** 이다.

```kotlin
// 표준 라이브러리:
@RestrictsSuspension
public abstract class SequenceScope<in T> {
    public abstract suspend fun yield(value: T)
    public abstract suspend fun yieldAll(iterator: Iterator<T>)
}
```

`@RestrictsSuspension`이 붙은 수신 타입(`SequenceScope`)의 스코프 안에서는 **그 수신 타입의 멤버나 확장인 `suspend` 함수만** 호출할 수 있고, 임의의 `suspend` 함수(`delay` 등)는 호출할 수 없다.

```kotlin
val bad = sequence {
    yield(1)
    // delay(100)   // 컴파일 에러: 제한된 중단 스코프에서는 임의 suspend 함수 호출 불가
}
```

이 애노테이션은 "이런 종류의 중단은 순수하게 동기적인 제너레이터를 위한 것이니 비동기 세계와 섞지 말라"는 제약을 타입 시스템으로 강제한다. `suspend`라는 하나의 메커니즘을 서로 다른 두 목적(비동기 중단과 동기적 값 생성)에 쓰면서도, 두 세계가 섞이지 않도록 언어가 경계를 정해 주는 것이다.

> [!info] 역사 메모
> `Iterator`를 만드는 `iterator { }` 빌더도 같은 `SequenceScope`/`@RestrictsSuspension` 메커니즘을 쓴다. 코루틴이 안정화되기 전(1.2)에도 시퀀스 빌더는 실험 코루틴 위에서 동작했다. 이는 코틀린 설계자들이 처음부터 `suspend`를 "비동기 전용"이 아니라 "일반화된 중단"으로 보았다는 증거이다.

---

## 8. 중단, 예외, 취소

### 8.1 예외는 resumeWith(failure)로 전달된다

41장(예외와 Nothing)에서 본 예외 모델은 `suspend` 세계에서도 그대로 성립한다. 다만 전달 경로가 하나 더 있다. `suspend` 함수 안에서 던진 예외는 일반 함수처럼 콜 스택(정확히는 `Continuation` 사슬)을 따라 위로 전파된다. 이 전파를 실제로 수행하는 것이 `resumeWith(Result.failure(e))`이다.

```kotlin
suspend fun risky(): Int {
    val x = fetchNumber()   // 이 안에서 예외가 나면
    return x + 1
}

suspend fun caller() {
    try {
        val n = risky()
    } catch (e: IOException) {
        println("잡았다: $e")   // 일반 try/catch로 잡힌다 — 콜백과 결정적 차이
    }
}
```

콜백 방식에서는 콜백 안에서 던진 예외가 원래 호출자의 `try/catch`에 잡히지 않았다(1.2절). `suspend`는 이 문제를 해결한다. 중단과 재개를 거치면서도 논리적인 콜 스택이 보존되므로, `risky` 안에서 던진 예외는 `caller`의 `try/catch`에 정확히 도착한다.

동작 과정은 이렇다. `fetchNumber`를 감싼 콜백이 `cont.resumeWithException(e)`를 호출하면, 그 예외가 `Result.failure`로 상태 기계에 들어오고 재개될 때 다시 `throw`된다. 이 예외는 `when(label)` 분기를 거슬러 올라가며 `caller`의 `try/catch`까지 전파된다. **순차 코드의 예외 모델이 그대로 복원**되는 것이다. 이것은 콜백이나 Future에 비해 `suspend`가 주는 가장 큰 실용적 이점 중 하나이다.

### 8.2 CancellationException과 협조적 취소 (미리 보기)

`kotlinx.coroutines`의 코루틴은 **취소할 수 있다**. 취소는 특수한 예외인 `CancellationException`을 중단점에서 던지는 방식으로 구현되며, 이를 **협조적 취소(cooperative cancellation)** 라고 한다. 취소된 코루틴이 다음 중단점(`delay`, `yield`, `withContext` 등 취소 여부를 확인하는 지점)에 도달하면 `CancellationException`이 던져지고, 코루틴은 정리된 뒤 종료된다.

여기서 41장(예외와 Nothing)과 관련된 함정이 하나 있다. `try/catch (e: Exception)`로 모든 예외를 삼키면 `CancellationException`까지 삼키게 되어 취소가 동작하지 않는다.

```kotlin
// 안티패턴 (kotlinx.coroutines 문맥):
suspend fun loop() {
    while (true) {
        try {
            work()          // 내부에 중단점 있음
        } catch (e: Exception) {
            // CancellationException까지 잡아 삼킴 → 이 코루틴은 취소되지 않는다!
        }
    }
}
```

`CancellationException`은 잡더라도 다시 던져야 한다. 취소가 어느 시점에 확인되는지, `NonCancellable`, `ensureActive`, `isActive`가 무엇인지 같은 자세한 동작은 Kotlin Coroutines 시리즈의 [[09 - 취소 - 협조적 취소의 기계학]]에서 다룬다. 여기서는 "중단점이 곧 취소 확인 지점이고, 취소는 예외로 전달된다"는 언어와 라이브러리의 접점만 기억하면 된다.

### 8.3 구조적 동시성 미리 보기

2.2절에서 병렬 실행은 `async`/`launch`로 명시적으로 만든다고 했다. 그렇게 시작한 자식 코루틴이 부모보다 오래 살아 있거나, 하나가 실패했는데 형제 코루틴이 계속 실행되거나, 부모가 끝났는데 자식이 남아 있는 문제가 생길 수 있다. 이런 문제를 모두 규제하는 것이 **구조적 동시성(structured concurrency)** 이다. 핵심 규칙은 "부모 코루틴이나 스코프는 모든 자식이 끝날 때까지 완료되지 않는다"는 것이다. `coroutineScope { }`가 이 규칙을 적용하는 최소 단위이다.

이것은 순전히 `kotlinx.coroutines`(그리고 `CoroutineScope`/`Job`의 부모-자식 계층)가 제공하는 정책이며, 언어의 `suspend`와는 별개이다. 언어는 "중단"만 제공하고, "누가 누구를 기다리며 실패가 어떻게 전파되는가"라는 규칙은 라이브러리가 그 위에 더한다. 자세한 내용은 Kotlin Coroutines 시리즈 06장(CoroutineScope와 구조적 동시성)에서 다룬다.

여기서 분명히 해 둘 것은 하나뿐이다. **구조적 동시성은 언어 기능이 아니라 라이브러리가 정한 규칙이다.** `suspend`만으로는 어떤 구조도 강제되지 않는다.

---

## 9. 자주 틀리는 부분: 함정 모음

### 9.1 suspend를 붙였는데 여전히 블로킹된다

가장 흔한 착각은 "`suspend`를 붙였으니 이 함수는 논블로킹이다"라는 생각이다. 그렇지 않다. `suspend`는 논블로킹을 *보장하지 않는다*. `suspend` 함수 본문 안에서 블로킹 호출(`Thread.sleep`, 블로킹 I/O, `Thread.join` 등)을 하면 그 함수는 여전히 스레드를 점유한다.

```kotlin
suspend fun stillBlocking() {
    Thread.sleep(1000)   // suspend 함수인데도 스레드를 1초간 블로킹한다!
}
```

`Thread.sleep`은 중단점이 아니다. 중단은 *다른 `suspend` 함수를 호출할 때만* 일어날 수 있다(2.1절). `Thread.sleep`은 `suspend`가 아니므로 그냥 스레드를 재운다. 논블로킹으로 기다리려면 `delay(1000)`(kotlinx.coroutines, `suspend`)을 써야 한다. `suspend` 함수는 "논블로킹으로 만들 *기회*"를 줄 뿐이고, 본문을 논블로킹으로 작성하는 것은 개발자의 책임이다.

| | 블로킹 | 논블로킹(중단) |
|--|--------|----------------|
| 대기 | `Thread.sleep(n)` | `delay(n)` |
| I/O | 블로킹 스트림 읽기 | `suspend`로 감싼 논블로킹 I/O |
| 스레드 | 점유함 | 놓아줌 |
| suspend 함수 안? | 가능하지만 이득 없음 | 이것이 목적 |

### 9.2 순차 실행을 병렬 실행으로 착각한다

2.2절에서 보았듯이 두 `suspend` 호출을 나란히 쓰면 순차적으로 실행된다. 이것을 병렬 실행으로 착각해 "왜 빨라지지 않지?"라고 묻는 경우가 많다.

```kotlin
// 순차 — 총 시간 = fetchUser + fetchOrders (합)
suspend fun sequential(id: Int): Pair<User, Orders> {
    val user = fetchUser(id)       // 이게 끝나야
    val orders = fetchOrders(id)   // 이게 시작
    return user to orders
}

// 병렬 — 총 시간 ≈ max(fetchUser, fetchOrders)
suspend fun parallel(id: Int): Pair<User, Orders> = coroutineScope {
    val user = async { fetchUser(id) }     // 즉시 시작
    val orders = async { fetchOrders(id) } // 즉시 시작(위와 동시에)
    user.await() to orders.await()
}
```

두 호출이 서로 독립적이어서 병렬로 실행해 시간을 줄이고 싶다면 `async`로 명시해야 한다. 병렬 실행은 절대 저절로 생기지 않는다.

### 9.3 GlobalScope와 콜백 연결에서 생기는 누수

`suspendCoroutine`으로 콜백을 감쌀 때(5.3절) 콜백이 성공과 실패 중 어느 쪽도 호출하지 않으면, 그 코루틴은 영원히 중단된 채 남는다. `Continuation`이 힙에 계속 참조된 채 GC되지 않는 누수이다. 콜백 API가 "정확히 한 번 호출한다"는 계약을 어기면 이런 누수가 생긴다.

그래서 실무에서는 취소까지 고려한 `suspendCancellableCoroutine`(kotlinx.coroutines)을 쓰고, `cont.invokeOnCancellation { ... }`으로 취소될 때 자원을 해제한다.

또한 `GlobalScope.launch { ... }`처럼 구조적 동시성 밖에서 코루틴을 시작하면, 그 코루틴은 아무도 기다리지 않고 아무도 취소하지 않는 상태로 남는다(8.3절). 이것은 언어의 문제가 아니라 라이브러리를 사용하는 규칙의 문제이며, Kotlin Coroutines 시리즈 06장(CoroutineScope와 구조적 동시성)에서 자세히 다룬다.

### 9.4 성능: continuation 할당과 스택 없는 코루틴

`suspend`의 비용을 정확히 알아 두자. 중단점이 있는 함수는 호출될 때마다 상태 기계 객체가 힙에 할당된다(5.4절). 중단점이 여러 개라면 그 함수의 상태 기계 하나와, 스택을 되감으면서 각 프레임의 상태를 담는 객체들이 관여한다.

대부분의 애플리케이션에서 이 비용은 네트워크나 디스크 지연에 비하면 무시할 수 있다. 애초에 `suspend`를 쓰는 이유가 그런 느린 대기를 효율적으로 처리하기 위해서이다. 하지만 자주 실행되는 경로에서 잠깐씩 중단하는 함수를 초당 수백만 번 호출한다면 GC 부담이 될 수 있다.

알아 둘 점이 하나 더 있다. 표준 코틀린 코루틴은 **스택 없는(stackless) 코루틴**이다. 즉 각 `suspend` 함수를 상태 기계로 변환할 뿐, 임의 깊이의 네이티브 콜 스택을 통째로 얼려 두지 않는다. 그래서 중단은 반드시 함수 호출 경계에서만 일어나며, 일반 함수 중간에서 스택을 얼려 중단할 수는 없다. 이것이 2.1절에서 "중단은 다른 suspend 함수를 호출할 때만 일어난다"고 한 근본적인 이유이다.

이는 자바의 프로젝트 룸(가상 스레드, 스택풀 코루틴)과 대비되는 설계 선택이다. 룸에서는 런타임이 스택을 얼리지만, 코틀린 코루틴에서는 컴파일러가 CPS로 변환한다. 이 차이는 43장(동시성과 메모리 모델)과 Kotlin Coroutines 시리즈의 [[22 - 지형도 - Loom Go JS Reactive 멀티플랫폼 미래]]에서 더 깊이 다룬다.

> [!caution] 성능 주의
> "코루틴은 경량 스레드"라는 흔한 비유는 유용하지만 오해를 부를 수 있다. 코루틴은 스레드가 아니라, 스레드 위에서 실행되는 중단 가능한 계산의 상태이다. 스레드 하나가 중단된 코루틴 수천 개를 번갈아 재개할 수 있다. 코루틴이 "경량"인 이유는 스레드 스택이 아니라 작은 힙 객체로 자기 상태를 표현하기 때문이다.
