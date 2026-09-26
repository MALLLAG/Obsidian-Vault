---
title: 인라인 함수와 reified
date: 2026-07-13
tags: [kotlin, inline, reified, generics, jvm, 학습노트]
---

[[13 - 함수 타입과 람다와 함수 참조]]에서는 함수가 일급 값이고 JVM에서는 `FunctionN` 인터페이스의 인스턴스로 표현된다는 점을 보았다. [[14 - 클로저와 캡처]]에서는 람다가 자유 변수를 캡처하려고 객체로 할당된다는 점을 보았다. 두 장이 람다의 *의미*를 다뤘다면, 이 장은 람다가 *어디에 어떤 비용으로 놓이는지*, 그리고 컴파일러가 그 비용을 없애려고 소스 코드를 어떻게 재배치하는지를 다룬다.

`inline`은 흔히 성능 최적화용 어노테이션 정도로 오해된다. 실제로 `inline`은 컴파일러에게 "이 함수의 본문과 이 함수에 넘어온 람다의 본문을 호출 지점에 그대로 복사해 넣어라"라고 지시하는 코드 변형 명령이다. 이렇게 코드를 복사하면 세 가지 결과가 생긴다.

- 람다 객체 할당과 가상 호출이 사라진다(성능).
- 람다 안에서 바깥 함수를 종료하는 **비지역 반환**(non-local return)이 가능해진다.
- 타입 인자가 호출 지점에서 구체 타입으로 치환되므로, `reified`를 통해 런타임 타입 검사를 할 수 있다.

이 가운데 성능과 관련된 것은 첫째뿐이고, 둘째와 셋째는 언어 의미론에 속한다. 그래서 `inline`은 컴파일러에게 주는 힌트가 아니다.

이 장에서는 람다의 숨은 비용, `inline`이 그 비용을 없애는 방식, 비지역 반환과 이를 제어하는 `crossinline`·`noinline`, `reified`와 그 한계, `inline`의 비용과 제약, 표준 라이브러리의 활용 사례, 그리고 언제 `inline`을 쓸지에 대한 판단 기준을 다룬다.

제네릭의 변성은 [[33 - 제네릭 1 - 타입 파라미터와 변성]]에서, 타입 소거의 전체 그림은 [[34 - 제네릭 2 - 타입 소거와 reified와 바운드]]에서 설명한다. 이름이 비슷한 `inline value class`(예전 이름은 `inline class`)는 전혀 다른 기능이며 [[36 - 타입 별칭과 인라인 value class]]에서 다룬다. 이 장 끝에서는 두 기능의 차이만 짚는다.

---

## 1. 람다의 숨은 비용: 왜 인라인이 필요한가

### 1.1 함수 타입은 JVM에서 인터페이스다

Kotlin 소스에서 `(Int) -> Boolean`은 간결한 함수 타입이지만, JVM 백엔드에서는 `kotlin.jvm.functions.Function1<Int, Boolean>` 인터페이스로 컴파일된다(13장 함수 타입과 람다). 파라미터 개수에 따라 `Function0`, `Function1`, …, `Function22`가 있고, 각 인터페이스에는 추상 메서드 `invoke(...)`가 하나씩만 있다.

```kotlin
fun forEachInt(list: List<Int>, action: (Int) -> Unit) {
    for (x in list) action(x)   // action.invoke(x) 로 컴파일
}
```

파라미터 `action`을 호출하는 `action(x)`는 JVM에서 `action.invoke(x)`라는 **인터페이스 메서드 호출**(`invokeinterface`)이 된다. 인터페이스 호출은 런타임에 구현 클래스를 찾아야 하므로 가상 디스패치 비용이 든다. 원시 타입이 관여하면 박싱 비용도 더해진다.

`Function1<Int, Unit>`의 타입 인자 `Int`는 제네릭 타입 인자라서 참조 타입이어야 하므로, 호출할 때마다 원시 `int`가 `java.lang.Integer`로 박싱된다(박싱은 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]에서 다룬다).

### 1.2 람다를 넘기면 객체가 하나 생긴다

호출 지점에서 람다를 실제로 넘겨 보자.

```kotlin
val nums = listOf(1, 2, 3)
var sum = 0
forEachInt(nums) { x -> sum += x }
```

람다 `{ x -> sum += x }`는 바깥의 `sum`을 캡처한다. JVM 백엔드에서 이 람다는 대략 다음 Java 코드에 해당하는 익명 클래스 인스턴스가 된다.

```java
// 개념적 디컴파일 (JVM 백엔드)
final IntRef sum = new IntRef();   // 캡처된 var를 담는 래퍼
sum.element = 0;
forEachInt(nums, new Function1<Integer, Unit>() {
    public Unit invoke(Integer x) {
        sum.element += x.intValue();   // 언박싱
        return Unit.INSTANCE;
    }
});
```

여기에는 세 가지 낭비가 있다.

- `Function1` 구현 객체 하나가 **힙에 할당**된다.
- 캡처된 `var sum`을 담으려고 `IntRef` 래퍼 객체가 하나 더 할당된다(14장에서 본 참조 셀).
- 루프가 원소 하나를 처리할 때마다 `Integer` **박싱/언박싱**과 `invoke`의 **가상 호출**이 반복된다.

리스트의 원소가 100만 개라면 이 오버헤드도 100만 번 발생한다.

> [!caution] 성능 주의
> 문제는 람다를 한 번 쓰는 것이 아니라, **자주 실행되는 루프(hot loop)의 본문에서 가상 호출과 박싱이 반복되는 것**이다. JIT가 단형성(monomorphic) 호출 지점을 인라인해 줄 수도 있다. 하지만 같은 고차 함수가 여러 종류의 람다로 호출되면 호출 지점이 다형성(megamorphic)이 되어 JIT 인라이닝이 막힌다. 컴파일 타임 인라인은 JIT의 판단에 의존하지 않고 이 비용을 확실하게 없앤다.

### 1.3 비캡처 람다와 invokedynamic: 구현에 따라 달라지는 영역

캡처가 없는 람다, 즉 자유 변수를 캡처하지 않는 람다는 매번 새 객체를 만들 필요가 없다. 상태가 없으므로 인스턴스 하나를 재사용하면 된다.

```kotlin
forEachInt(nums) { x -> println(x) }   // 바깥 상태 캡처 없음
```

Kotlin 1.x의 JVM 백엔드는 이런 무상태 람다를 **싱글턴**으로 컴파일했다(생성된 클래스에 `INSTANCE` 필드가 하나 있다). Kotlin 2.0부터 JVM 백엔드는 기본적으로 **`invokedynamic` + `LambdaMetafactory`** 방식으로 람다를 생성한다(예전의 `-Xlambdas=indy` 옵션이 2.0에서 기본값이 되었다).

이 방식은 별도의 `.class` 파일을 미리 만들지 않고 런타임에 람다 구현을 생성하며, 무상태 람다의 인스턴스 캐싱도 런타임이 처리한다.

> [!note] 명세 기준
> "람다가 객체로 할당된다"는 설명은 **캡처하는 람다**에만 확실하게 들어맞는다. 무상태 람다가 실제로 어떻게 표현되는지(싱글턴 클래스인지 `invokedynamic` 부트스트랩인지, 인스턴스가 캐시되는지)는 **JVM 백엔드 버전과 컴파일 옵션에 따라 다르다**. Kotlin/Native와 Kotlin/JS 백엔드는 또 다른 표현을 쓴다. 이 장에서 "할당이 사라진다"고 말할 때는 캡처하는 람다를 기준으로 한다.

중요한 점은 싱글턴이든 indy든 이런 최적화로도 **가상 호출 자체는 없앨 수 없다**는 것이다. `invoke`는 여전히 인터페이스 메서드 호출로 남는다. 호출 자체를 없애는 것은 `inline`뿐이다. 람다의 본문이 호출 지점에 그대로 펼쳐지기 때문이다. 다음 절에서 이 과정을 살펴본다.

---

## 2. inline이 실제로 하는 일: 코드 복사

### 2.1 정의와 첫 관찰

함수 앞에 `inline`을 붙이면 컴파일러는 그 함수를 호출하는 코드를 호출로 남겨 두지 않고, **함수 본문 전체를 호출 지점에 복사해 넣는다**. 그 함수에 넘어온 람다 인자가 있으면 **람다의 본문까지** 호출 지점에 펼친다.

```kotlin
inline fun forEachInt(list: List<Int>, action: (Int) -> Unit) {
    for (x in list) action(x)
}

fun demo(nums: List<Int>) {
    var sum = 0
    forEachInt(nums) { x -> sum += x }
    println(sum)
}
```

`demo`를 컴파일하면 `forEachInt` 호출은 사라지고 그 자리에 루프가 직접 들어간다. `action(x)` 자리에는 람다 본문 `sum += x`가 그대로 들어간다. 개념적인 결과는 다음과 같다.

```java
// demo의 개념적 디컴파일 (JVM 백엔드, inline 적용 후)
public void demo(List<Integer> nums) {
    int sum = 0;                      // 이제 원시 int, 래퍼 객체 없음
    for (Integer x : nums) {          // forEachInt의 몸통이 여기 복사됨
        sum += x.intValue();          // action(x) 자리에 람다 몸통이 복사됨
    }
    System.out.println(sum);
}
```

`Function1` 객체도 없고, `IntRef` 래퍼도 없다. 캡처된 `sum`이 같은 스택 프레임의 지역 변수로 남기 때문에 참조 셀에 담을 필요가 없다. `invoke` 가상 호출도 없다. 남은 것은 루프뿐이며, 손으로 쓴 `for` 루프와 바이트코드가 사실상 같아진다. `inline`이 성능을 높이는 원리는 특별한 것이 아니라 소스 수준의 복사이다.

### 2.2 inline은 힌트가 아니다

C++의 `inline`이나 C#의 `[MethodImpl(AggressiveInlining)]`은 컴파일러나 JIT에 주는 *제안*이라서 무시될 수 있다. Kotlin의 `inline`은 다르다. **컴파일러는 반드시 인라인한다.** 정확히 말하면, 인라인할 수 없는 상황에서는 컴파일 에러를 내거나, 경고를 내고 특정 파라미터만 인라인하지 않는다(4절, 7절). 따라서 `inline`은 관찰할 수 있는 **의미론적 계약**이다.

- 넘긴 람다가 힙 객체로 존재하지 **않는다**. 그래서 람다에서 바깥 함수로 **비지역 반환**할 수 있다(3절).
- 타입 인자가 호출 지점에서 **구체 타입으로 치환**된다. 그래서 `reified`로 런타임 타입을 알 수 있다(5절).
- 본문이 호출 지점에 복사된다. 그래서 **디버깅할 때 스택 프레임이 보이지 않고**, 바이너리가 커진다(7절).

이 세 가지는 성능이 아니라 언어 규칙이다. 따라서 "`inline`을 떼도 동작은 같고 속도만 느려진다"는 생각은 틀렸다. 비지역 반환을 쓰는 코드에서 `inline`을 떼면 **컴파일 에러**가 난다. `reified`를 쓰는 코드에서 `inline`을 떼도 **컴파일 에러**가 난다.

> [!warning] 흔한 오해
> "작은 함수에 `inline`을 붙이면 모두 빨라진다"는 생각은 틀렸다. 람다 파라미터가 **없는** 함수에 `inline`을 붙이면 컴파일러가 오히려 경고한다.
>
> *"Expected performance impact from inlining is insignificant. Inlining works best for functions with parameters of functional types."* JIT가 이미 잘 인라인하는 평범한 함수를 소스 수준에서 강제로 복사하면 바이너리만 커진다. `inline`이 필요한 이유는 **람다 파라미터**와 `reified`이다.

### 2.3 디컴파일로 직접 확인하는 방법

지금까지의 설명은 직접 확인할 수 있다. IntelliJ IDEA에서 `Tools → Kotlin → Show Kotlin Bytecode → Decompile`을 누르면 컴파일된 `.class`를 Java로 역컴파일해 보여 준다. 인라인 함수의 호출 지점이 어떻게 펼쳐졌는지, `Function1` 객체가 사라졌는지를 직접 볼 수 있다.

이 장의 "개념적 디컴파일"은 모두 실제 도구로 검증할 수 있는 결과를 단순화한 것이다(컴파일 파이프라인 전반은 [[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]]에서 다룬다).

```text
소스                         컴파일러 인라인 단계                  바이트코드
────────────                ───────────────────                ──────────
inline fun f(g: ()->Unit)   f의 몸통을 호출 지점에 복사     ┌─ 호출 지점 A: f의 몸통 + 람다A 몸통
  { ...; g(); ... }         각 호출 지점의 람다 몸통을      ├─ 호출 지점 B: f의 몸통 + 람다B 몸통
                            g() 자리에 펼침                └─ 호출 지점 C: f의 몸통 + 람다C 몸통
f { 람다A }                                                  (f라는 메서드 호출은 어디에도 없음)
f { 람다B }
f { 람다C }
```

이 다이어그램은 7절에서 다룰 "코드 팽창"의 원인도 보여 준다. 호출 지점이 세 곳이면 본문이 세 번 복사된다. 그래서 `inline` 함수는 **작아야** 한다.

---

## 3. 비지역 반환: 코드 복사가 만드는 첫 번째 의미론

### 3.1 forEach 안의 return이 함수를 끝내는 이유

Kotlin 표준 라이브러리의 `forEach`는 `inline` 함수다. 그래서 다음 코드가 동작한다.

```kotlin
fun findFirstNegative(nums: List<Int>): Int? {
    nums.forEach { n ->
        if (n < 0) return n     // findFirstNegative 자체를 종료하고 n을 반환
    }
    return null
}
```

여기서 `return n`은 **람다를 빠져나가는 것이 아니라 바깥의 `findFirstNegative` 전체를 종료**한다. 이것을 **비지역 반환**(non-local return)이라고 한다. 처음 보면 이상하게 느껴진다. `return`이 자신이 속한 람다가 아니라 두 단계 위의 함수를 끝내기 때문이다.

그 이유는 2절에서 설명한 복사에 있다. `forEach`가 `inline`이므로 위 코드는 컴파일할 때 다음과 같이 펼쳐진다.

```kotlin
// forEach 인라인 후의 개념적 형태
fun findFirstNegative(nums: List<Int>): Int? {
    for (n in nums) {           // forEach의 몸통
        if (n < 0) return n     // 이제 이 return은 findFirstNegative의 몸통 안에 있다
    }
    return null
}
```

람다의 본문이 `findFirstNegative`의 본문 안으로 복사되었으므로, 그 안의 `return`은 당연히 `findFirstNegative`를 종료한다. 람다를 위한 별도의 함수 프레임이 없으니 "람다를 빠져나간다"는 개념 자체가 성립하지 않는다. 비지역 반환은 **코드 복사의 논리적 결과**이다.

### 3.2 "람다의 return은 람다만 벗어난다"는 오해

"람다 안의 `return`은 람다만 빠져나간다"는 설명은 **inline 람다에서는 틀렸다**. inline 람다에서 라벨 없는 `return`은 바깥 함수를 종료한다. 반대로 **비-inline 람다**에서는 라벨 없는 `return`을 아예 쓸 수 없다.

```kotlin
fun nonInlineDemo(nums: List<Int>) {
    // filter는 inline이 아니라고 가정 X — 실제로는 대부분 inline이지만,
    // 저장된 함수 타입 값(비-inline)으로 예를 보자:
    val action: (Int) -> Unit = { n ->
        // return          // 컴파일 에러: 'return' is not allowed here
        return@action      // 이것만 허용: 람다 자신만 빠져나감(지역 반환)
    }
    nums.forEach(action)   // action은 미리 만든 함수 값 — 비-inline 경로
}
```

이유는 실행 구조에 있다. 비-inline 람다는 별도의 `Function1` 객체이고, 이 객체의 `invoke`가 실행되는 시점에는 바깥 함수가 이미 반환했을 수도 있다. 람다를 저장해 두었다가 나중에 호출할 수 있기 때문이다. 이런 상황에서 바깥 함수를 종료하는 것은 불가능하다. 종료할 프레임이 스택에 없을 수도 있기 때문이다. 그래서 컴파일러는 비-inline 람다에서 비지역 `return`을 쓰지 못하게 막는다.

> [!note] 명세 기준
> 라벨 없는 `return`이 비지역 반환이 되는 것은 **람다가 인라인되는 파라미터로 전달될 때**뿐이다. 지역 반환(자신이 속한 람다만 종료)은 항상 `return@forEach`, `return@label`처럼 **라벨**을 붙여 쓴다. 라벨 반환은 inline 여부와 관계없이 언제나 허용된다.
>
> 반대로 **익명 함수**(`fun(x: Int) { ... }`)에서는 라벨 없는 `return`이 항상 익명 함수 자신을 종료한다. 익명 함수는 람다와 달리 자체 `return` 스코프를 가지기 때문이다(13장 함수 타입과 람다).

### 3.3 라벨 반환: 지역 반환과 비지역 반환 중 선택

같은 inline 람다 안에서도 두 종류의 반환을 골라 쓸 수 있다.

```kotlin
fun demo(nums: List<Int>) {
    nums.forEach { n ->
        if (n == 0) return@forEach   // 지역: 이 원소만 건너뛰고 다음 반복 (continue 효과)
        if (n < 0) return            // 비지역: demo 전체를 종료
        println(n)
    }
    println("끝")   // 위에서 음수를 만났다면 여기 도달하지 않음
}
```

`return@forEach`는 이번 람다 호출만 끝내므로 루프의 `continue`처럼 동작한다. `return`은 `demo`를 끝낸다. 실무에서는 이 둘을 정확히 구분해야 한다. 특히 `forEach`에서 `continue`처럼 동작시키려다 라벨 없는 `return`을 써서 함수 전체를 끝내 버리는 실수가 흔하다.

```text
inline 람다 안에서의 return 문법
──────────────────────────────
return          →  바깥 함수 종료   (비지역, inline일 때만 가능)
return@forEach  →  이 람다만 종료   (지역, 항상 가능)
return@myLabel  →  라벨 지점까지 종료 (지역, 명시 라벨)
break/continue  →  실제 루프에만 적용 (forEach는 함수 호출이므로 직접 못 씀)
```

### 3.4 비지역 반환의 실용적 가치

비지역 반환 덕분에 inline 고차 함수는 **내장 제어 구조처럼** 자연스럽게 쓸 수 있다. `let`, `run`, `with`, `apply`, `also`([[16 - 스코프 함수와 수신 객체 관용구]]), `repeat`, `use`([[41 - 예외와 Nothing]]) 같은 표준 함수는 모두 inline이다. 그래서 그 안에서 `return`으로 바깥 함수를 빠져나가도 손으로 쓴 `if`나 `for`와 똑같이 동작한다.

```kotlin
fun readConfig(path: String): Config? {
    val text = readFileOrNull(path) ?: return null   // 엘비스 + 비지역 return
    return runCatching { parse(text) }.getOrElse {
        return null                                   // runCatching이 아니라 readConfig 종료
    }
}
```

`getOrElse`도 inline이므로 그 람다 안의 `return null`이 `readConfig`를 끝낼 수 있다. 이렇게 자연스럽게 쓸 수 있는 것은 모두 inline 덕분이다. 이 스코프 함수들이 비-inline이었다면 호출할 때마다 람다 객체가 할당되고 비지역 반환도 불가능했을 것이다. Kotlin에서 함수 호출이 문법 구조처럼 느껴지는 DSL 같은 감각은 [[37 - 타입 안전 빌더와 DSL]]에서 가장 잘 드러나지만, 그 바탕에는 inline이 만드는 투명한 제어 흐름이 있다.

---

## 4. crossinline과 noinline: 인라이닝을 세밀하게 제어하기

### 4.1 문제: inline 람다를 나중에 호출하고 싶을 때

inline 함수의 람다 파라미터는 기본적으로 인라인된다. 그런데 그 람다를 **호출 지점의 실행 흐름 밖에서** 호출하려고 하면 문제가 생긴다. 람다를 다른 객체 안에 담아 두었다가 나중에 실행하려는 경우가 그 예이다.

```kotlin
inline fun runLater(action: () -> Unit) {
    val runnable = Runnable { action() }   // 컴파일 에러
    // Can't inline 'action' here: it may contain non-local returns.
    // Add 'crossinline' modifier to parameter declaration 'action'
    Thread(runnable).start()
}
```

왜 에러가 날까? `action`은 인라인되어야 하는데, `Runnable { action() }`이라는 **다른 실행 문맥**(익명 객체의 `run` 메서드) 안에 `action`의 본문을 펼치면, 그 안의 비지역 `return`이 `runLater`를 종료하려고 한다. 하지만 `Runnable`은 나중에 다른 스레드에서 실행되므로, 그때는 `runLater`의 프레임이 스택에 없다. 따라서 비지역 반환은 불가능하다. 컴파일러는 이 위험을 미리 막는다.

### 4.2 crossinline: 인라인하되 비지역 반환은 금지한다

해결책은 `crossinline`이다. 이 한정자는 "이 람다는 그대로 인라인하되(성능 유지), **비지역 반환은 하지 못하게** 하라"는 뜻이다.

```kotlin
inline fun runLater(crossinline action: () -> Unit) {
    val runnable = Runnable { action() }   // OK: action은 인라인되지만 비지역 return 불가
    Thread(runnable).start()
}

fun demo() {
    runLater {
        // return       // 컴파일 에러: 'return' is prohibited here (crossinline)
        println("나중에 실행")   // OK
        return@runLater          // 지역 반환은 여전히 허용
    }
}
```

`crossinline`은 인라인의 성능 이점(객체 할당 없음)은 유지하고, 위험한 비지역 반환만 막는다. 정확히 말하면 람다가 인라인 함수의 본문과는 다른 실행 문맥으로 넘어가는(cross) 상황을 위한 한정자이다. 비동기 콜백, 이벤트 핸들러, `Runnable`/`Comparator` 같은 SAM 어댑터 안에서 람다를 호출할 때 필요하다.

> [!note] 명세 기준
> `crossinline` 람다도 인라인된다. 즉 구현이 인라인에 성공하는 한 별도의 `Function` 객체가 생기지 않는다. 다만 컴파일러의 제어 흐름 분석에서 "이 지점 이후로는 비지역 반환 금지"라는 표시가 붙는다. `crossinline`을 "인라인을 끄는 것"으로 오해하면 안 된다. 인라인을 끄는 것은 `noinline`이다.

### 4.3 noinline: 이 람다는 인라인하지 않는다

람다 파라미터를 **객체로 다뤄야** 할 때도 있다. 람다를 변수에 저장하거나, 비-inline 함수에 다시 넘기거나, 반환값으로 돌려주려면 람다가 실제로 존재하는 `Function` 객체여야 한다. 이럴 때 `noinline`을 쓴다.

```kotlin
inline fun transaction(
    body: () -> Unit,               // 인라인됨
    noinline onRollback: () -> Unit // 인라인 안 됨 — 객체로 보관
) {
    val handler: () -> Unit = onRollback   // noinline이라 변수에 저장 가능
    try {
        body()
    } catch (e: Exception) {
        registerHandler(handler)    // 다른 곳에 넘겨 나중에 실행
    }
}
```

`onRollback`에 `noinline`이 없으면 `val handler = onRollback`에서 컴파일 에러가 난다. 인라인될 람다는 실체가 없으므로 변수에 담을 수 없기 때문이다. `noinline`을 붙이면 그 파라미터만 평범한 `Function0` 객체로 남으므로 자유롭게 저장하고 전달하고 반환할 수 있다. 물론 그 파라미터에서는 인라인의 성능 이점을 얻지 못한다.

### 4.4 세 가지 경우 정리

```text
람다 파라미터의 세 가지 상태 (inline 함수 안에서)
─────────────────────────────────────────────
기본(수식어 없음)  │ 인라인 O │ 비지역 return O │ 저장/전달 X
crossinline        │ 인라인 O │ 비지역 return X │ 저장/전달 X (다른 문맥서 호출 O)
noinline           │ 인라인 X │ 비지역 return X │ 저장/전달 O (객체로 존재)
```

| 상황 | 필요한 한정자 |
|------|--------------|
| 람다를 그냥 호출하기만 함 | 기본(없음) |
| 람다를 익명 객체나 다른 람다 안에서 호출 | `crossinline` |
| 람다를 변수에 저장하거나 다른 함수에 전달하거나 반환 | `noinline` |
| 람다를 nullable로 받음 (`(() -> Unit)?`) | 자동으로 `noinline`처럼 취급 |

마지막 행은 주의해서 봐야 한다. 인라인 함수의 람다 파라미터가 **nullable 함수 타입**이면 `null` 검사에 실제 객체가 필요하므로, 컴파일러는 사실상 인라인하지 못한다. 그래서 nullable 함수 파라미터는 기본적으로 인라인되지 않는다. 인라인의 이점을 얻으려면 nullable 타입을 피하고 오버로드나 기본 인자로 우회한다.

> [!caution] 성능 주의
> inline 함수가 람다 파라미터를 여러 개 받고 그중 일부만 `noinline`이면, 인라인되는 파라미터는 할당이 사라지고 `noinline` 파라미터만 객체로 남는다. 즉 "부분 인라인"이 가능하며, 함수 전체를 인라인하거나 전혀 하지 않는 둘 중 하나만 있는 것이 아니다.

---

## 5. reified: 지워진 타입을 되살린다

### 5.1 배경: 타입 소거

JVM 백엔드에서 제네릭 타입 인자는 **런타임에 지워진다**(type erasure, 자세한 내용은 34장 타입 소거와 reified에서 다룬다). `List<String>`과 `List<Int>`는 런타임에 똑같은 `List`이다. 그래서 일반 제네릭 함수 안에서는 타입 파라미터 `T`가 실제로 무엇인지 알 수 없다.

```kotlin
fun <T> isInstance(value: Any): Boolean {
    // return value is T     // 컴파일 에러:
    // Cannot check for instance of erased type: T
    TODO()
}
```

`value is T`를 쓸 수 없는 이유는 런타임에 `T`가 무엇인지 알려 주는 정보가 어디에도 없기 때문이다. 함수 시그니처에는 `T`가 있지만, 컴파일된 바이트코드에서 `T`는 `Object`로 지워진다. `T::class`도 같은 이유로 쓸 수 없다. 지워진 타입의 클래스 리터럴은 만들 수 없기 때문이다.

Java에서는 `Class<T>` 파라미터를 명시적으로 넘겨서 이 문제를 우회한다(`Class<T> clazz`를 인자로 받아 `clazz.isInstance(value)`를 호출한다). Kotlin에서도 같은 방법을 쓸 수 있지만, 더 간결한 방법이 있다. 바로 `reified`이다.

### 5.2 reified의 동작 원리

`inline` 함수의 타입 파라미터 앞에 `reified`를 붙이면, 그 타입 파라미터는 호출 지점에서 **구체 타입으로 치환**된다. 함수가 인라인되어 본문이 호출 지점에 복사될 때, 컴파일러는 `T`가 나오는 모든 자리에 **실제 타입 인자**를 넣는다. 그래서 `is T`, `as T`, `T::class`를 쓸 수 있다.

```kotlin
inline fun <reified T> isInstance(value: Any): Boolean = value is T

fun demo() {
    println(isInstance<String>("hello"))   // => true
    println(isInstance<Int>("hello"))      // => false
}
```

컴파일할 때 `isInstance<String>("hello")`는 인라인되어 `"hello" is String`으로 바뀌고, `isInstance<Int>(...)`는 `... is Int`로 바뀐다. 개념적으로 디컴파일하면 다음과 같다.

```java
// demo의 개념적 형태 (reified 치환 후)
public void demo() {
    Object v1 = "hello";
    System.out.println(v1 instanceof String);          // isInstance<String>
    Object v2 = "hello";
    System.out.println(v2 instanceof Integer);         // isInstance<Int> → Integer instanceof
}
```

각 호출 지점에서 `T`가 사라지고 실제 타입(`String`, `Integer`)이 그 자리에 들어갔다. 즉 **런타임에 타입을 알아내는 것이 아니라, 컴파일 타임에 호출 지점마다 타입을 미리 넣어 두는 것**이다. 이 차이가 중요하다. `reified`는 "런타임 제네릭"이 아니라 "컴파일 타임 특수화"이다.

> [!info] 역사 메모
> `reified`는 Kotlin 1.0부터 있었다. Java에는 이런 기능이 없어서 Java 개발자는 늘 `Class<T>` 토큰을 직접 넘겨야 한다. Kotlin의 `reified`는 inline이 타입 인자를 호출 지점에 맞게 특수화한다는 성질을 활용한 것이다. 개념적으로는 C++ 템플릿의 인스턴스화(타입마다 별도 코드 생성)와 비슷하지만, Kotlin은 함수 하나에만, 그리고 inline일 때만 이를 적용한다.

### 5.3 reified로 할 수 있는 것

`reified T`가 있으면 다음 작업을 모두 할 수 있다.

```kotlin
inline fun <reified T> reifiedPlayground(value: Any?) {
    val checkIs   = value is T              // 타입 검사
    val checkNot  = value !is T             // 부정 검사
    val casted    = value as? T             // 안전 캐스트
    val kClass    = T::class                // KClass<T> 리터럴
    val jClass    = T::class.java           // java.lang.Class<T> (JVM 백엔드)
    val typeName  = T::class.simpleName     // 타입 이름 문자열
    println("$checkIs $checkNot $casted $kClass $typeName")
}
```

실무에서 가장 흔한 활용 사례는 **역직렬화 API**이다. 타입 토큰을 직접 넘기지 않고 타입 파라미터로 추론하게 한다.

```kotlin
// 개념 예시: JSON 문자열을 원하는 타입으로 파싱
inline fun <reified T> parseJson(json: String): T {
    val clazz: Class<T> = T::class.java     // reified 덕에 Class 토큰을 안에서 얻음
    return someJsonLibrary.fromJson(json, clazz)
}

data class Point(val x: Int, val y: Int)

fun demo() {
    val p: Point = parseJson("""{"x":1,"y":2}""")   // 타입 인자 추론
    val p2 = parseJson<Point>("""{"x":3,"y":4}""")  // 명시도 가능
    println(p)   // => Point(x=1, y=2)
}
```

호출하는 쪽에서 `Class` 토큰을 매번 넘기지 않아도 되므로 API가 깔끔해진다. `enumValues<T>()`와 `enumValueOf<T>(name)`([[29 - enum 클래스]])도 `reified`로 구현된 표준 함수이다.

### 5.4 필터링: filterIsInstance

표준 라이브러리의 `filterIsInstance`는 `reified`의 대표적인 사례이다.

```kotlin
// 표준 라이브러리 구현의 골자
inline fun <reified R> Iterable<*>.filterIsInstance(): List<R> {
    val result = ArrayList<R>()
    for (element in this) if (element is R) result.add(element)
    return result
}

fun demo() {
    val mixed: List<Any> = listOf(1, "a", 2, "b", 3.0)
    val strings = mixed.filterIsInstance<String>()  // => [a, b]
    val ints = mixed.filterIsInstance<Int>()        // => [1, 2]
    println(strings)
}
```

`element is R`를 쓰려면 `R`이 실체화되어야 하고, 그러려면 함수가 `inline`이어야 한다. 그래서 `filterIsInstance`는 반드시 `inline fun <reified R>`로 선언된다. 이 함수 하나를 보면 `inline`과 `reified`가 왜 항상 함께 쓰이는지 알 수 있다.

---

## 6. reified의 한계: 최상위 타입만 실체화된다

### 6.1 흔한 착각: reified면 완전한 제네릭 타입을 얻는다

흔히 "`reified`를 쓰면 완전한 제네릭 타입을 얻는다"고 생각하지만, 이는 오해이다. 실제로는 **최상위 타입만** 실체화되고, 그 안쪽의 타입 인자는 여전히 소거된다. 이 절에서는 이 점을 자세히 살펴본다.

`reified`는 인라인을 이용해 호출 지점에 타입을 넣는 기법이다. 그런데 넣을 수 있는 것은 결국 **JVM의 `instanceof` 검사가 표현할 수 있는 것**뿐이고, `instanceof`는 소거된 타입만 검사한다. 그래서 `reified T`를 `List<String>`으로 인스턴스화하면, `is T`는 `is List<String>`이 아니라 **`is List`(원소 타입 무시)** 가 된다.

```kotlin
inline fun <reified T> checkType(value: Any): Boolean = value is T

fun demo() {
    val listOfStrings: Any = listOf("a", "b")
    val listOfInts: Any = listOf(1, 2, 3)

    // T = List<String> 로 불러도, 런타임 검사는 'is List' 수준
    println(checkType<List<String>>(listOfStrings))  // => true
    println(checkType<List<String>>(listOfInts))     // => true  ← 함정!
    // Int 리스트인데도 List<String> 검사를 통과한다.
    // 원소가 String인지는 검사되지 않았기 때문.
}
```

핵심은 두 번째 출력이 `true`라는 점이다. `checkType<List<String>>`은 사실상 `value is List<*>`까지만 검사한다. 안쪽의 `String`은 런타임에 지워져 있으므로 검사할 방법이 없다. `reified`는 **바깥 타입(`List`)만 실체화**하고, 안쪽 타입(`String`)은 여전히 소거된 상태로 남는다.

> [!warning] 흔한 오해
> "`reified`가 있으니 `List<String>`과 `List<Int>`를 런타임에 구별할 수 있다"는 생각은 틀렸다. JVM 백엔드에서 둘은 런타임에 똑같은 `List`이며, `reified`로도 구별할 수 없다. `reified`가 되살리는 것은 지워진 타입 계층의 **한 겹**뿐이다. 원소 타입을 정말로 알아야 한다면 `filterIsInstance`처럼 원소를 실제로 꺼내 `is`로 검사하거나, 별도의 타입 토큰을 설계해야 한다.

### 6.2 컴파일러가 아예 막는 경우

`value is List<String>`을 직접 쓰면 컴파일러가 막는다.

```kotlin
fun raw(value: Any) {
    // val ok = value is List<String>   // 컴파일 에러:
    // Cannot check for instance of erased type: List<String>
    val ok = value is List<*>           // OK: 스타 프로젝션으로 원소 무시 명시
    println(ok)
}
```

Kotlin은 직접 검사하는 코드에서는 "지워진 타입은 검사할 수 없다"는 에러를 정확하게 낸다. 그런데 `reified T`를 거치면 이 에러가 **우회**된다. `checkType<List<String>>`은 컴파일되고, 내부적으로는 `is List`(스타 프로젝션과 같은 수준)로 낮춰진다.

그래서 `reified`는 편리한 대신 "완전한 타입 검사"를 한다는 착각을 줄 위험이 있다. 정확히는 **`reified T`에서 `is T`가 검사하는 범위는 `T`의 소거된 상한(erased upper bound)까지**라고 이해해야 한다.

### 6.3 예외: typeOf<T>()가 제네릭 인자를 아는 이유

이 규칙에 어긋나 보이는 사례가 하나 있다. 표준 라이브러리의 `typeOf<T>()`([[44 - 애노테이션과 리플렉션]])는 `reified`로 구현되었지만, `List<String>`의 `String`까지 담은 완전한 `KType`을 반환한다.

```kotlin
import kotlin.reflect.typeOf

fun demo() {
    val t = typeOf<List<String>>()
    println(t)   // => kotlin.collections.List<kotlin.String>  ← String까지 나온다!
}
```

`is T`는 `String`을 보지 못하는데, `typeOf<T>()`는 어떻게 볼 수 있을까? 모순처럼 보이지만 그렇지 않다. 두 기능은 **정보를 얻는 출처가 다르다**.

- `is T`는 **JVM의 런타임 `instanceof`** 에 의존한다. 따라서 소거된 타입만 알 수 있고, 바깥 타입만 검사한다.
- `typeOf<T>()`는 컴파일러가 **호출 지점의 정적 타입 정보를 데이터 객체로 만들어 주입**한다. `String` 인자까지 모두 `KTypeProjection` 데이터로 넣으므로 완전한 타입을 얻는다.

즉 `typeOf`는 런타임에 타입을 검사하는 것이 아니라, 컴파일 타임에 알고 있던 타입 구조를 **직렬화한 메타데이터**를 런타임에 그대로 반환한다. 따라서 이 사례는 앞의 규칙에 대한 반례가 아니라, 규칙이 적용되는 경계를 정확히 보여 준다. **런타임 `is`/`as` 검사**는 최상위 타입까지만 다루고, **컴파일러가 주입하는 메타데이터**(`typeOf`, 리플렉션)는 완전한 타입을 담을 수 있다.

```text
reified T 로 얻는 것의 두 층위
──────────────────────────────
value is T        →  JVM instanceof  →  최상위 타입만  (List<String> → List)
value as T        →  JVM checkcast   →  최상위 타입만  (안쪽 미검사)
T::class          →  KClass          →  최상위 타입만  (List::class, 인자 없음)
typeOf<T>()       →  컴파일러 주입    →  완전한 KType  (List<String> 전체)
```

`T::class`가 `KClass<List<*>>`(즉 `List::class`)를 반환하는 것도 같은 원리이다. `KClass`는 클래스를 나타내므로 타입 인자를 담지 않는다. 타입 인자까지 필요하면 `KType`(`typeOf`)을 써야 한다. 이 구분은 44장(애노테이션과 리플렉션)의 `KClass`와 `KType` 비교로 이어진다.

### 6.4 reified로 인스턴스를 만들 수 있는가

"`reified T`가 있으니 `T()`로 인스턴스를 만들 수 있지 않을까?" 하고 기대하기 쉽다.

```kotlin
inline fun <reified T> create(): T {
    // return T()      // 컴파일 에러: Type parameter T cannot be called as function
    return T::class.java.getDeclaredConstructor().newInstance()  // 리플렉션으로는 가능(JVM)
}
```

`T()`처럼 직접 호출할 수는 없다. `T`에 기본 생성자가 있는지 컴파일러가 보장할 수 없기 때문이다. 하지만 `reified`로 `T::class.java`를 얻을 수 있으므로 리플렉션으로 우회할 수 있다. 이 방법은 JVM 백엔드에서, 기본 생성자가 있고 접근할 수 있을 때만 동작하며, 그렇지 않으면 런타임 예외가 발생한다. 이 예는 타입을 아는 것과 타입의 인스턴스를 생성하는 것이 서로 다른 일임을 보여 준다. `reified`는 타입 정보를 제공할 뿐, 생성 능력을 주지는 않는다.

---

## 7. inline의 비용과 제약

### 7.1 코드 팽창: 복사에 따르는 비용

2.3절의 다이어그램에서 보았듯이, inline 함수는 **호출될 때마다 본문이 복사**된다. 호출 지점이 100곳이면 본문이 100번 복제되어 바이너리가 커진다. 이 점은 성능 면에서 함정이 될 수 있다. 코드가 커지면 명령어 캐시(instruction cache)에 주는 부담이 늘어나 오히려 느려질 수 있기 때문이다.

그래서 관례는 분명하다. **inline 함수는 작아야 한다.** 표준 라이브러리의 inline 함수(`let`, `run`, `forEach`, `map` 등)는 대부분 몇 줄짜리이다. 본문이 큰 함수에 `inline`을 붙이면 각 호출 지점의 코드가 커진다.

> [!caution] 성능 주의
> 큰 inline 함수를 여러 곳에서 호출하면, 람다 할당을 줄여서 얻는 이점보다 코드 팽창으로 잃는 것이 클 수 있다. 흔히 쓰는 절충안은 **람다를 받는 바깥 부분만 작게 inline하고, 무거운 실제 작업은 비-inline private 함수로 분리해 호출**하는 것이다. 이렇게 하면 각 호출 지점에는 작은 바깥 부분만 복사되고, 무거운 본문은 한 곳에만 존재한다.

```kotlin
inline fun <T> measured(label: String, block: () -> T): T {
    val start = System.nanoTime()
    val result = block()             // 람다는 인라인
    logDuration(label, start)        // 무거운 로직은 비-inline 함수로 위임 → 복사 안 됨
    return result
}

fun logDuration(label: String, startNanos: Long) { /* 큰 몸통, 한 곳에만 존재 */ }
```

### 7.2 public inline 함수의 가시성 규칙: private 멤버 접근 금지

실무에서 가장 자주 당황하게 되는 제약이다. **`public`(또는 `protected`) inline 함수는 자신이 속한 클래스나 파일의 `private` 멤버를 참조할 수 없다.**

```kotlin
class Cache {
    private val store = HashMap<String, Any>()   // private

    inline fun <reified T> getOrNull(key: String): T? {
        // return store[key] as? T     // 컴파일 에러:
        // Public-API inline function cannot access non-public-API 'private val store'
        TODO()
    }
}
```

이유는 2절에서 본 코드 복사에 있다. `public inline` 함수의 본문은 **다른 모듈의 호출 지점에 복사**된다. 그 모듈은 `Cache`의 `private store`에 접근할 권한이 없다. 복사된 코드가 `store`를 참조하면 접근할 수 없는 멤버를 건드리게 되어 캡슐화가 깨진다. 그래서 컴파일러는 처음부터 이를 금지한다.

우회 방법은 `@PublishedApi`이다. 이 어노테이션을 `internal` 멤버에 붙이면 "public inline 함수 안에서 쓰도록 공개한 내부 API"라고 표시하게 된다.

```kotlin
class Cache {
    @PublishedApi
    internal val store = HashMap<String, Any>()   // internal + @PublishedApi

    inline fun <reified T> getOrNull(key: String): T? = store[key] as? T   // OK
}
```

`@PublishedApi internal` 멤버는 소스에서는 `internal`(모듈 내부, [[26 - 가시성 한정자]])이지만, inline 함수의 본문에 복사되어 다른 모듈에 노출되어도 문제가 없도록 컴파일러가 이름을 유지해 준다. 다만 이렇게 노출된 멤버는 **바이너리 호환성의 일부**가 된다. 시그니처를 함부로 바꾸면 다른 모듈의 컴파일된 코드가 깨질 수 있다. 즉 이 규칙은 `private`을 쓰지 못하게 하려는 것이 아니라, 노출을 의식적으로 선언하게 하려는 것이다.

> [!note] 명세 기준
> 이 제약은 함수의 가시성에 따라 달라진다. `private inline` 함수는 호출 지점이 같은 파일이나 클래스 안에만 있으므로 `private` 멤버를 자유롭게 참조할 수 있다. `internal inline` 함수도 같은 모듈 안에서만 인라인되므로 `internal` 멤버를 참조할 수 있다. 문제가 되는 것은 **모듈 경계를 넘어 복사되는 `public`/`protected` inline 함수**뿐이다.

### 7.3 재귀 불가

inline 함수는 **자기 자신을 직접으로든 간접으로든 호출할 수 없다**. 인라인은 호출을 본문 복사로 바꾸는데, 재귀 호출이 있으면 복사가 끝없이 이어지기 때문이다.

```kotlin
inline fun factorial(n: Int): Int {
    // return if (n <= 1) 1 else n * factorial(n - 1)
    // 컴파일 에러: Inline function 'factorial' cannot be recursive
    TODO()
}
```

재귀가 필요하면 `inline`을 떼거나, 꼬리 재귀라면 `tailrec`([[12 - 함수 - 인자와 vararg와 지역 함수와 꼬리 재귀]])으로 따로 최적화한다. 두 기법은 전혀 다르다. `inline`은 호출 지점에 코드를 복사하고, `tailrec`은 재귀를 루프로 변환한다.

### 7.4 디버깅과 스택 트레이스

인라인된 함수는 자체 스택 프레임을 남기지 않는다. 그래서 예외가 발생하면 스택 트레이스에 inline 함수의 이름이 보이지 않고, 호출자 위치가 뒤섞여 보일 수 있다(최신 컴파일러는 디버그 정보로 inline 위치를 표시하려고 하지만 완벽하지는 않다). inline 함수 안에 브레이크포인트를 걸 때도 호출 지점마다 다르게 동작할 수 있다.

> [!caution] 성능 주의
> inline이 스택 프레임을 없애면 프레임 push/pop 비용이 사라지므로 성능에는 이득이지만, 관측 가능성은 떨어진다. 매우 자주 실행되는 경로가 아니라면 디버깅과 프로파일링의 편의를 위해 inline을 삼가는 편이 나을 때도 있다.

### 7.5 인라인이 의미가 없거나 경고가 나는 경우

- **람다 파라미터가 없는 함수**: 2.2절에서 보았듯이 컴파일러가 "인라인 효과가 미미하다"는 경고를 낸다. 다만 `reified`가 필요하면 람다가 없어도 inline을 쓰는 것이 타당하다. 이때는 `reified` 때문에 인라인이 필수이므로 경고가 나지 않는다.
- **모든 람다 파라미터가 `noinline`인 함수**: 인라인할 람다가 하나도 없으므로 이점이 없다.
- **거대한 함수**: 코드 팽창만 일으킨다.

```kotlin
// 정당한 inline: 람다 파라미터가 있음
inline fun <T> withLock(lock: Lock, action: () -> T): T { /* ... */ TODO() }

// 정당한 inline: reified가 필요함 (람다 없어도 OK)
inline fun <reified T> typeName(): String = T::class.simpleName ?: "?"

// 의심스러운 inline: 람다도 reified도 없음 → 컴파일러 경고
inline fun add(a: Int, b: Int): Int = a + b   // 경고: 인라인 이득 미미
```

---

## 8. inline의 다른 쓰임: 프로퍼티, 표준 라이브러리, 이름 충돌

### 8.1 inline 프로퍼티 접근자

`inline`은 함수뿐 아니라 **프로퍼티 접근자**에도 붙일 수 있다. 백킹 필드([[22 - 프로퍼티와 백킹 필드]])가 없는 계산 프로퍼티의 getter와 setter를 인라인할 수 있다.

```kotlin
val Int.isEven: Boolean
    inline get() = this % 2 == 0     // getter만 inline

var displayName: String
    inline get() = computeName()      // 두 접근자 각각 inline 가능
    inline set(value) { storeName(value) }
```

프로퍼티 전체에 `inline`을 붙이면(`inline val`/`inline var`) 모든 접근자가 인라인된다. 단, 백킹 필드가 있는 프로퍼티는 inline할 수 없다. 필드 접근은 인라인 대상이 아니기 때문이다. 또한 프로퍼티는 타입 파라미터를 가질 수 없으므로 프로퍼티에 `reified`를 직접 쓸 수는 없다(`reified`는 함수의 타입 파라미터에만 쓸 수 있다).

### 8.2 inline으로 만들어진 표준 라이브러리

Kotlin 표준 라이브러리의 상당 부분은 inline 함수이다. 어떤 함수가 inline인지 알면, 표준 라이브러리를 쓴 코드가 왜 손으로 쓴 루프만큼 빠른지 이해할 수 있다.

| 함수 | inline인가 | 비지역 return | reified |
|------|-----------|--------------|---------|
| `let`, `run`, `with`, `apply`, `also` | O | O | - |
| `takeIf`, `takeUnless` | O | O | - |
| `repeat(n) { }` | O | O | - |
| `forEach`, `forEachIndexed`, `onEach` | O | O | - |
| `filter`, `map`, `flatMap` (Iterable) | O | O | - |
| `runCatching` | O | O | - |
| `use` (Closeable) | O | O | - |
| `synchronized`, `withLock` | O | O | - |
| `filterIsInstance<R>()` | O | - | O |
| `enumValues<T>()`, `enumValueOf<T>()` | O | - | O |
| `typeOf<T>()` | O | - | O (특수) |
| `arrayOf`, `emptyList` | - (비-inline) | - | - |

`map`, `filter` 같은 컬렉션 연산이 inline이므로, 함수형 파이프라인([[39 - 컬렉션 연산과 함수형 파이프라인]])은 람다를 할당하지 않고 실행된다. 다만 이 연산들은 **즉시 평가**되므로 중간 리스트를 만든다. 이것은 inline과는 별개의 문제이며, 지연 평가는 `Sequence`([[40 - 시퀀스와 지연 평가]])에서 다룬다. inline은 람다 호출 오버헤드를 없앨 뿐, 중간 컬렉션 생성을 없애지는 않는다. 두 최적화를 혼동하면 안 된다.

### 8.3 스코프 함수와 DSL의 성능 기반

16장(스코프 함수와 수신 객체 관용구)의 `let`, `apply` 등은 모두 inline이므로, 아무리 중첩해 써도 람다 객체가 생기지 않는다.

```kotlin
fun buildUser(name: String): User =
    User().apply {          // apply는 inline → 람다 할당 없음
        this.name = name
        this.createdAt = now()
    }.also {                // also도 inline
        log("created: ${it.name}")
    }
```

이 체이닝은 읽기 좋으면서도, 손으로 필드를 설정한 코드와 같은 바이트코드를 만든다. 37장(타입 안전 빌더와 DSL)의 DSL을 성능 걱정 없이 깊게 중첩할 수 있는 것도, 그 바탕에 있는 수신 객체 지정 람다들이 inline이기 때문이다. 추상화를 써도 비용이 들지 않는 것(zero-cost abstraction)은 inline 덕분이다.

### 8.4 이름 충돌 정리: inline 함수와 inline value class

마지막으로 이 장 내내 미뤄 둔 이름 혼동을 정리한다. Kotlin에는 `inline`이라는 단어가 두 곳에서 쓰인다.

```kotlin
// (1) 이 장의 주제: inline 함수 — 호출 지점 복사
inline fun <T> measure(block: () -> T): T { /* ... */ TODO() }

// (2) 완전히 다른 기능: inline value class — 래퍼 박싱 회피 (36장)
@JvmInline
value class UserId(val raw: Long)
```

둘은 이름만 비슷할 뿐 **서로 관련이 없다**.

| | inline **함수** (15장) | inline **value class** (36장) |
|---|---|---|
| 대상 | 함수/프로퍼티 접근자 | 단일 값을 감싸는 클래스 |
| 목적 | 람다 할당·호출 제거, reified | 래퍼 객체 할당 제거(값 자체를 그대로 사용) |
| 키워드 | `inline fun` | `@JvmInline value class` (옛 `inline class`) |

역사적으로 값 클래스는 `inline class`라는 키워드로 실험적으로 도입되었다. 그런데 함수 인라인과 헷갈린다는 이유로 `value class`(+`@JvmInline`)로 이름이 바뀌었고, 그 흔적으로 어노테이션 이름에 `Inline`이 남았다. `value class`가 박싱을 피하는 조건(nullable, 제네릭, 인터페이스로 상향 변환할 때는 박싱된다)은 36장(타입 별칭과 인라인 value class)에서 다룬다. 이 장에서는 두 기능이 서로 다르다는 점만 기억하면 된다.

---

## 9. 언제 inline을 쓰고, 언제 삼가는가

### 9.1 써야 하는 경우

다음 세 가지 중 하나라도 해당하면 `inline`을 쓰는 것이 타당하다.

1. **람다 파라미터를 받고, 자주 실행되는 경로에서 호출된다.** 람다 할당과 가상 호출을 없애서 실제로 측정 가능한 이득이 생기는 고차 함수가 여기에 해당한다. `forEach`, `withLock`, `use` 같은 함수이다.
2. **`reified` 타입 파라미터가 필요하다.** 이 경우 `inline`은 선택이 아니라 필수이다. `reified`는 `inline` 없이 쓸 수 없기 때문이다. `filterIsInstance`, `parseJson`, `typeOf` 같은 함수이다.
3. **비지역 반환을 제어 구조로 제공하고 싶다.** 사용자가 람다 안에서 바깥 함수를 종료할 수 있게 하려면 함수가 inline이어야 한다. 커스텀 제어 흐름 DSL이 여기에 해당한다.

### 9.2 삼가야 하는 경우

1. **람다도 reified도 없는 평범한 함수.** JIT가 알아서 인라인하므로, 소스 수준의 인라인은 바이너리만 키운다.
2. **본문이 큰 함수를 여러 곳에서 호출하는 경우.** 코드 팽창 때문에 이득이 사라진다. 바깥 부분만 inline하고 본문은 다른 함수로 분리한다(7.1절).
3. **디버깅과 프로파일링이 중요한 코드.** 스택 프레임이 사라져 관측하기 어려워진다.
4. **private 상태에 크게 의존하는 public API.** `@PublishedApi`로 내부 멤버를 노출할 수밖에 없고, 바이너리 호환성을 유지해야 하는 부담이 생긴다.

### 9.3 판단 흐름도

```text
inline을 붙일까?
│
├─ reified 타입 파라미터가 필요한가? ──── 예 ──→ inline 필수 (선택 아님)
│                                        아니오
│                                          │
├─ 람다 파라미터를 받는가? ──── 아니오 ──→ inline 붙이지 마라 (경고 대상)
│                              예
│                               │
├─ 뜨거운 경로 / 반복 호출인가? ── 아니오 ──→ 대개 불필요 (JIT에 맡겨라)
│                                예
│                                 │
├─ 함수 몸통이 작은가? ──── 아니오 ──→ 껍데기만 inline, 몸통은 비-inline 위임
│                          예
│                           │
└──────────────────────────→ inline 적합 ✓
```

### 9.4 전체를 다시 보기

`inline`의 세 가지 효과인 성능(할당 제거), 제어(비지역 반환), 타입(reified)은 모두 "본문을 호출 지점에 복사한다"는 **하나의 사실**에서 나온다. 이 관점으로 보면 서로 무관해 보이는 규칙들이 모두 같은 원리에서 나온 결과로 정리된다. 재귀가 안 되는 이유, public inline 함수가 private 멤버에 접근하지 못하는 이유, reified가 inline을 요구하는 이유, 비지역 반환이 inline에서만 되는 이유가 모두 그렇다.

`inline`은 단순한 어노테이션이 아니라 **컴파일러에게 코드의 형태를 재배치하라고 지시하는 명령**이며, 그 재배치의 결과가 곧 언어 의미론이 된다.
