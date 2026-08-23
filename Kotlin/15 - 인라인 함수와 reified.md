---
title: 인라인 함수와 reified
date: 2026-07-13
tags: [kotlin, inline, reified, generics, jvm, 학습노트]
---

**이 장이 답하는 질문:**

- `inline`은 정말 "속도를 위한 힌트"에 불과한가, 아니면 언어 의미론을 바꾸는 키워드인가?
- 람다를 함수에 넘길 때 JVM에서는 무슨 비용이 발생하며, `inline`은 그 비용을 어떻게 없애는가?
- 왜 `forEach {}` 안에서 쓴 `return`은 바깥 함수를 통째로 종료시키는가? 그리고 왜 모든 람다가 그럴 수는 없는가?
- `noinline`과 `crossinline`은 각각 무엇을 금지하고 무엇을 허용하는가?
- `reified`는 어떻게 "지워진(erased)" 제네릭 타입을 런타임에 되살리는가? 그 마법의 정확한 범위는 어디까지인가?
- `inline fun <reified T> check(x: Any) = x is T`에서 `T`를 `List<String>`으로 부르면 `String`까지 검사되는가?
- `public` 인라인 함수는 왜 `private` 멤버를 건드리지 못하는가?
- 언제 `inline`이 이득이고 언제 바이트코드만 부풀리는 자충수인가?

---

이 장은 [[13 - 함수 타입과 람다와 함수 참조]]가 세운 "함수는 일급 값이며 JVM에서는 `FunctionN` 인터페이스의 인스턴스로 표현된다"는 사실과, [[14 - 클로저와 캡처]]가 밝힌 "람다는 자유 변수를 캡처하기 위해 객체로 할당된다"는 사실 위에서 출발한다. 그 두 장은 람다의 *의미*를 다뤘다. 이 장은 그 람다가 *어디에 얼마의 비용으로 놓이는가*, 그리고 컴파일러가 그 비용을 없애기 위해 소스 코드의 형태를 어떻게 물리적으로 재배치하는가를 다룬다.

`inline`은 흔히 "성능 최적화 어노테이션쯤" 되는 것으로 오해된다(M17). 사실 `inline`은 컴파일러에게 "이 함수의 몸통과, 이 함수에 넘어온 람다의 몸통을, 호출 지점에 그대로 복사해 붙여 넣어라"라고 지시하는 코드 변형 명령이다. 이 물리적 복사가 세 가지 의미론적 결과를 낳는다. 첫째, 람다 객체 할당과 가상 호출이 사라진다(성능). 둘째, 람다 안에서 바깥 함수를 종료하는 **비지역 반환**(non-local return)이 가능해진다(M41). 셋째, 타입 인자가 호출 지점에서 구체 타입으로 치환되므로 `reified`를 통해 런타임 타입 검사가 가능해진다(M18). 셋 중 첫째만 성능이고, 둘째·셋째는 순수한 언어 의미론이다. 그래서 `inline`은 힌트가 아니다.

이 장이 다루지 않는 것도 분명히 해두자. 함수 타입 자체의 표현과 SAM 변환은 [[13 - 함수 타입과 람다와 함수 참조]]가, 캡처의 메모리 수명은 [[14 - 클로저와 캡처]]가 소유한다. 제네릭의 변성과 타입 파라미터 일반론은 [[33 - 제네릭 1 - 타입 파라미터와 변성]]이, 타입 소거의 전모와 `reified`의 제네릭 이론적 자리는 [[34 - 제네릭 2 - 타입 소거와 reified와 바운드]]가 소유한다(이 장은 `reified`를 `inline`의 부산물로서 실무 관점에서 파고, 소거의 큰 그림은 34장에 위임한다). 그리고 이름이 헷갈리는 `inline value class`(옛 `inline class`)는 완전히 다른 기능으로 [[36 - 타입 별칭과 인라인 value class]]가 소유한다 — 이 장 끝에서 그 구분만 못박는다.

논증의 궤적은 이렇다. 먼저 람다의 숨은 비용을 JVM 수준에서 해부하고(§1), `inline`이 그 비용을 어떻게 물리적 복사로 없애는지 디컴파일로 확인한다(§2). 그다음 그 복사가 낳는 첫 번째 의미론적 결과인 비지역 반환을 파고(§3), 그것을 통제하는 `crossinline`·`noinline`을 본다(§4). 이어 같은 복사가 낳는 두 번째 결과인 `reified`를 다루되(§5), 그 마법의 정확한 경계 — 최상위 타입만 실체화된다는 M18의 핵심 — 를 정면으로 해부한다(§6). 마지막으로 `inline`의 대가와 제약(§7), 표준 라이브러리 속의 응용(§8), 그리고 언제 쓰고 언제 삼갈지의 판단(§9)으로 닫는다.

---

## 1. 람다의 숨은 비용 — 왜 인라인이 필요한가

### 1.1 함수 타입은 JVM에서 인터페이스다

Kotlin 소스에서 `(Int) -> Boolean`은 우아한 함수 타입이지만, JVM 백엔드에서 이것은 `kotlin.jvm.functions.Function1<Int, Boolean>` 인터페이스로 컴파일된다([[13 - 함수 타입과 람다와 함수 참조]]). 파라미터 개수에 따라 `Function0`, `Function1`, …, `Function22`가 있고, 각각 딱 하나의 추상 메서드 `invoke(...)`를 가진다.

```kotlin
fun forEachInt(list: List<Int>, action: (Int) -> Unit) {
    for (x in list) action(x)   // action.invoke(x) 로 컴파일
}
```

이 `action` 파라미터를 호출하는 `action(x)`는 JVM에서 `action.invoke(x)`라는 **인터페이스 메서드 호출**(`invokeinterface`)이 된다. 인터페이스 호출은 구현 클래스를 런타임에 찾아야 하므로 가상 디스패치 비용이 든다. 게다가 원시 타입이 관여하면 박싱이 얹힌다: `Function1<Int, Unit>`의 타입 인자 `Int`는 제네릭이라 참조 타입이어야 하므로, 매 호출마다 원시 `int`가 `java.lang.Integer`로 박싱된다(박싱은 [[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]).

### 1.2 람다를 넘기면 객체가 하나 태어난다

호출 지점에서 람다를 실제로 넘겨보자.

```kotlin
val nums = listOf(1, 2, 3)
var sum = 0
forEachInt(nums) { x -> sum += x }
```

이 람다 `{ x -> sum += x }`는 바깥의 `sum`을 캡처한다. JVM 백엔드에서 이것은 대략 다음 Java에 해당하는 익명 클래스 인스턴스로 실체화된다:

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

여기서 낭비가 세 겹이다. 첫째, `Function1` 구현 객체 하나가 **힙에 할당**된다. 둘째, 캡처된 `var sum`을 담기 위해 `IntRef` 래퍼 객체가 하나 더 할당된다([[14 - 클로저와 캡처]]에서 본 참조 셀). 셋째, 루프가 도는 매 원소마다 `Integer` **박싱/언박싱**과 `invoke`의 **가상 호출**이 반복된다. 리스트가 100만 개면 이 오버헤드가 100만 번 곱해진다.

> **성능 주의**: 이 비용은 "람다를 한 번 쓰는 것"이 문제가 아니라, **뜨거운 루프의 몸통에서 반복되는 가상 호출과 박싱**이 문제다. JIT가 단형성(monomorphic) 호출 지점을 인라인해 줄 수도 있지만, 같은 고차 함수가 여러 종류의 람다로 불리면 다형성(megamorphic)이 되어 JIT 인라이닝이 막힌다. 컴파일 타임 인라인은 JIT의 변덕에 의존하지 않는 확정적 제거다.

### 1.3 비캡처 람다와 invokedynamic — 구현 재량의 영역

캡처가 없는 람다(자유 변수를 안 잡는 람다)는 매번 새 객체를 만들 필요가 없다. 상태가 없으니 하나의 인스턴스를 재사용하면 된다.

```kotlin
forEachInt(nums) { x -> println(x) }   // 바깥 상태 캡처 없음
```

Kotlin 1.x의 JVM 백엔드는 이런 무상태 람다를 **싱글턴**으로 컴파일했다(생성된 클래스에 `INSTANCE` 필드 하나). Kotlin 2.0부터 JVM 백엔드는 기본적으로 **`invokedynamic` + `LambdaMetafactory`** 방식으로 람다를 생성한다(과거 `-Xlambdas=indy` 옵션이 2.0에서 기본값이 됨). 이 방식은 별도 `.class` 파일을 미리 만들지 않고 런타임에 람다 구현을 생성하며, 무상태 람다의 인스턴스 캐싱도 런타임이 담당한다.

> **명세 정밀**: "람다가 객체로 할당된다"는 서술은 **캡처하는 람다**에 대해서만 확실하다. 무상태 람다의 실제 표현(싱글턴 클래스인지 `invokedynamic` 부트스트랩인지, 인스턴스가 캐시되는지)은 **JVM 백엔드 버전과 컴파일 옵션에 따라 다르다**. Kotlin/Native·Kotlin/JS 백엔드는 또 다른 표현을 쓴다. 이 장에서 "할당이 사라진다"고 말할 때의 기준선은 캡처 람다다.

여기서 핵심은, 이 모든 최적화(싱글턴이든 indy든)조차 **가상 호출 자체는 없애지 못한다**는 점이다. `invoke`는 여전히 인터페이스 메서드 호출로 남는다. `inline`만이 호출 자체를 사라지게 한다 — 람다의 몸통이 호출 지점에 그대로 펼쳐지기 때문이다. 그것이 다음 절의 주제다.

---

## 2. inline이 실제로 하는 일 — 물리적 복사 (M17)

### 2.1 정의와 첫 관찰

함수 앞에 `inline`을 붙이면, 컴파일러는 그 함수의 호출을 "호출"로 남겨두지 않고, **함수 몸통 전체를 호출 지점에 복사해 넣는다**. 그리고 그 함수에 넘어온 람다 인자가 있으면, 그 **람다의 몸통까지** 호출 지점에 펼친다.

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

`demo`를 컴파일하면, `forEachInt` 호출은 사라지고 그 자리에 루프가 직접 박히며, `action(x)` 자리에는 람다 몸통 `sum += x`가 그대로 들어간다. 개념적 결과는 다음과 같다:

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

`Function1` 객체가 없다. `IntRef` 래퍼가 없다(캡처된 `sum`이 같은 스택 프레임의 지역 변수로 남으므로 셀에 담을 필요가 없다). `invoke` 가상 호출이 없다. 남은 것은 순수한 루프뿐이다. 손으로 쓴 `for` 루프와 바이트코드가 사실상 동일해진다. 이것이 `inline`의 성능 효과의 정체다 — 마법이 아니라 소스 수준 복사다.

### 2.2 "힌트가 아니다" — M17의 정면 교정

C++의 `inline`이나 C#의 `[MethodImpl(AggressiveInlining)]`은 컴파일러/JIT에 주는 *제안*이며, 무시될 수 있다. Kotlin의 `inline`은 다르다. **컴파일러는 반드시 인라인한다**(정확히는, 인라인이 불가능한 상황이면 컴파일 에러를 내거나 경고와 함께 특정 파라미터만 인라인하지 않는다 — §4, §7). 그래서 `inline`은 관찰 가능한 **의미론적 계약**이다:

- 넘긴 람다가 힙 객체로 존재하지 **않는다** → 람다에서 바깥 함수로 **비지역 반환**할 수 있다(§3).
- 타입 인자가 호출 지점에서 **구체 타입으로 치환**된다 → `reified`로 런타임 타입을 알 수 있다(§5).
- 몸통이 호출 지점에 복사된다 → **디버깅 시 스택 프레임이 사라지고**, 바이너리가 커진다(§7).

이 셋은 성능이 아니라 언어 규칙이다. 그래서 `inline`을 "떼도 동작은 같고 속도만 느려진다"고 생각하면 틀린다(M17). 비지역 반환을 쓰는 코드에서 `inline`을 떼면 **컴파일 에러**가 난다. `reified`를 쓰는 코드에서 `inline`을 떼면 **컴파일 에러**가 난다.

> **흔한 오해**: "작은 함수에 `inline`을 붙이면 다 빨라진다." 아니다. 람다 파라미터가 **없는** 함수에 `inline`을 붙이면 컴파일러가 오히려 경고한다 — *"Expected performance impact from inlining is insignificant. Inlining works best for functions with parameters of functional types."* JIT가 이미 잘 인라인하는 평범한 함수를 소스 수준에서 강제 복사해봐야 바이너리만 커진다. `inline`의 존재 이유는 **람다 파라미터**(그리고 `reified`)다.

### 2.3 디컴파일로 직접 확인하는 법

이 모든 주장은 눈으로 확인할 수 있다. IntelliJ IDEA에서 `Tools → Kotlin → Show Kotlin Bytecode → Decompile`을 누르면, 컴파일된 `.class`를 Java로 역컴파일해 보여준다. 인라인 함수의 호출 지점이 어떻게 펼쳐졌는지, `Function1` 객체가 사라졌는지를 직접 볼 수 있다. 이 장의 모든 "개념적 디컴파일"은 실제 도구로 검증 가능한 형태를 단순화한 것이다([[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]]에서 파이프라인 전반을 다룬다).

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

이 다이어그램이 §7의 "코드 팽창"을 미리 설명한다. 호출 지점이 셋이면 몸통이 세 번 복사된다. 그래서 `inline` 함수는 **작아야** 한다.

---

## 3. 비지역 반환 — 복사가 낳는 첫 번째 의미론 (M41)

### 3.1 왜 forEach 안의 return이 함수를 끝내는가

Kotlin 표준 라이브러리의 `forEach`는 `inline` 함수다. 그래서 다음이 동작한다:

```kotlin
fun findFirstNegative(nums: List<Int>): Int? {
    nums.forEach { n ->
        if (n < 0) return n     // findFirstNegative 자체를 종료하고 n을 반환
    }
    return null
}
```

여기 `return n`은 **람다를 빠져나가는 게 아니라 바깥의 `findFirstNegative`를 통째로 종료**한다. 이것을 **비지역 반환**(non-local return)이라 한다. 처음 보면 놀랍다 — `return`이 왜 자기가 속한 람다가 아니라 두 단계 위의 함수를 끝내는가?

답은 §2에 있다. `forEach`가 `inline`이므로, 위 코드는 컴파일 시 이렇게 펼쳐진다:

```kotlin
// forEach 인라인 후의 개념적 형태
fun findFirstNegative(nums: List<Int>): Int? {
    for (n in nums) {           // forEach의 몸통
        if (n < 0) return n     // 이제 이 return은 findFirstNegative의 몸통 안에 있다
    }
    return null
}
```

람다의 몸통이 `findFirstNegative`의 몸통 안으로 물리적으로 복사되었으므로, 그 안의 `return`은 자연히 `findFirstNegative`를 종료한다. 별도의 함수 프레임이 존재하지 않으니, "람다를 빠져나간다"는 것 자체가 성립하지 않는다. 비지역 반환은 마법이 아니라 **복사의 논리적 귀결**이다.

### 3.2 M41의 정면 교정 — "람다의 return은 람다만 벗어난다"는 오해

여기서 M41을 못박자. "람다 안의 `return`은 람다만 빠져나간다"는 것은 **inline 람다에서는 거짓**이다. inline 람다에서 벌거벗은 `return`은 바깥 함수를 종료한다. 반대로 **비-inline 람다**에서는 벌거벗은 `return`이 아예 **금지**된다.

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

이유는 물리적이다. 비-inline 람다는 별도의 `Function1` 객체이고, 그 객체의 `invoke`가 실행되는 시점에 바깥 함수는 이미 반환했을 수도 있다(람다를 저장해뒀다가 나중에 부를 수 있으니까). 그런 상황에서 "바깥 함수를 종료하라"는 것은 물리적으로 불가능하다 — 종료할 프레임이 스택에 없을 수 있다. 그래서 컴파일러는 비-inline 람다의 비지역 `return`을 원천 봉쇄한다.

> **명세 정밀**: 벌거벗은 `return`이 비지역 반환이 되는 것은 **람다가 인라인되는 파라미터로 넘어갈 때**뿐이다. 지역 반환(자기 람다만 종료)은 항상 **라벨**로 표기한다: `return@forEach`, `return@label`. 이 라벨 반환은 inline이든 아니든 언제나 허용된다. 반대로 **익명 함수**(`fun(x: Int) { ... }`)는 벌거벗은 `return`이 항상 자기 자신을 종료한다 — 익명 함수는 람다와 달리 자체 `return` 스코프를 가진다([[13 - 함수 타입과 람다와 함수 참조]]).

### 3.3 라벨 반환 — 지역과 비지역의 선택

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

`return@forEach`는 "이번 람다 호출만 끝내라" — 즉 루프의 `continue`처럼 동작한다. `return`은 "`demo`를 끝내라". 이 두 가지를 정확히 구분하는 것이 실무에서 중요하다. 특히 `forEach`를 `continue`처럼 쓰려다 벌거벗은 `return`을 써서 함수 전체를 날려버리는 실수가 흔하다.

```text
inline 람다 안에서의 return 문법
──────────────────────────────
return          →  바깥 함수 종료   (비지역, inline일 때만 가능)
return@forEach  →  이 람다만 종료   (지역, 항상 가능)
return@myLabel  →  라벨 지점까지 종료 (지역, 명시 라벨)
break/continue  →  실제 루프에만 적용 (forEach는 함수 호출이므로 직접 못 씀)
```

### 3.4 비지역 반환의 실전 가치

비지역 반환 덕분에 inline 고차 함수는 **내장 제어 구조처럼** 자연스럽게 쓰인다. `let`, `run`, `with`, `apply`, `also`([[16 - 스코프 함수와 수신 객체 관용구]]), `repeat`, `use`([[41 - 예외와 Nothing]]) 같은 표준 함수들이 모두 inline이라, 그 안에서 `return`으로 바깥을 빠져나가도 손으로 쓴 `if`/`for`와 동일하게 동작한다.

```kotlin
fun readConfig(path: String): Config? {
    val text = readFileOrNull(path) ?: return null   // 엘비스 + 비지역 return
    return runCatching { parse(text) }.getOrElse {
        return null                                   // runCatching이 아니라 readConfig 종료
    }
}
```

`runCatching`도 inline이므로 그 람다 안의 `return null`이 `readConfig`를 끝낼 수 있다. 이런 매끄러움은 전적으로 inline 덕이다 — 만약 이 스코프 함수들이 비-inline이었다면 매 호출마다 람다 객체가 할당되고 비지역 반환은 불가능했을 것이다. Kotlin의 "함수 같은데 문법 구조처럼 느껴지는" DSL스러운 감각은 [[37 - 타입 안전 빌더와 DSL]]에서 절정에 이르지만, 그 뿌리는 여기, inline이 만드는 투명한 제어 흐름에 있다.

---

## 4. crossinline과 noinline — 인라이닝의 정밀 제어

### 4.1 문제: inline 람다를 "나중에" 부르고 싶을 때

inline 함수의 람다 파라미터는 기본적으로 인라인된다. 그런데 그 람다를 **호출 지점의 실행 흐름 밖에서** 부르려 하면 문제가 생긴다. 예를 들어 람다를 다른 객체 안에 담아 나중에 실행하려는 경우다.

```kotlin
inline fun runLater(action: () -> Unit) {
    val runnable = Runnable { action() }   // 컴파일 에러
    // Can't inline 'action' here: it may contain non-local returns.
    // Add 'crossinline' modifier to parameter declaration 'action'
    Thread(runnable).start()
}
```

왜 에러인가? `action`은 인라인되어야 하는데, `Runnable { action() }`이라는 **다른 실행 문맥**(익명 객체의 `run` 메서드) 안에 `action`의 몸통을 펼치면, 그 안의 비지역 `return`이 `runLater`를 종료하려 할 것이다. 하지만 `Runnable`은 다른 스레드에서 나중에 실행되므로 그때는 `runLater`의 프레임이 스택에 없다 — 비지역 반환이 물리적으로 불가능하다. 컴파일러는 이 위험을 미리 막는다.

### 4.2 crossinline — 인라인하되 비지역 반환은 금지

해결책은 `crossinline`이다. 이 한정자는 "이 람다는 여전히 인라인하되(성능 유지), 대신 **비지역 반환은 못 하게** 하라"는 뜻이다.

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

`crossinline`은 인라인의 성능 이득(객체 할당 없음)은 유지하면서, 위험한 비지역 반환만 잘라낸다. 정확히는 "람다가 인라인 함수의 몸통과는 다른 실행 문맥으로 넘어간다(cross)"는 상황을 위한 것이다. 비동기 콜백, 이벤트 핸들러, `Runnable`/`Comparator` 같은 SAM 어댑터 안에서 람다를 부를 때 필요하다.

> **명세 정밀**: `crossinline` 람다도 여전히 인라인된다. 즉 별도 `Function` 객체가 안 생긴다(구현이 인라인에 성공하는 한). 다만 컴파일러의 제어 흐름 분석에서 "이 지점 이후로 비지역 반환 금지" 표시가 붙는다. `crossinline`과 인라인의 관계를 "인라인을 끈다"로 오해하면 안 된다 — 끄는 것은 `noinline`이다.

### 4.3 noinline — 이 람다는 인라인하지 마라

때로는 람다 파라미터를 **객체로서 다뤄야** 한다. 예를 들어 그 람다를 변수에 저장하거나, 비-inline 함수에 다시 넘기거나, 반환값으로 돌려주려면, 람다가 실체 있는 `Function` 객체여야 한다. 이럴 때 `noinline`을 쓴다.

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

`onRollback`에 `noinline`이 없으면 `val handler = onRollback`에서 컴파일 에러가 난다 — 인라인될 람다는 실체가 없어 변수에 담을 수 없기 때문이다. `noinline`을 붙이면 그 파라미터만 평범한 `Function0` 객체로 남고, 저장·전달·반환이 자유로워진다. 물론 그 파라미터에 대해서는 인라인의 성능 이득이 없다.

### 4.4 세 한정자의 정리

```text
람다 파라미터의 세 가지 상태 (inline 함수 안에서)
─────────────────────────────────────────────
기본(수식어 없음)  │ 인라인 O │ 비지역 return O │ 저장/전달 X
crossinline        │ 인라인 O │ 비지역 return X │ 저장/전달 X (다른 문맥서 호출 O)
noinline           │ 인라인 X │ 비지역 return X │ 저장/전달 O (객체로 존재)
```

| 상황 | 필요한 한정자 |
|------|--------------|
| 람다를 그냥 호출만 함 | 기본(없음) |
| 람다를 익명 객체/다른 람다 안에서 호출 | `crossinline` |
| 람다를 변수에 저장·다른 함수에 전달·반환 | `noinline` |
| 람다를 nullable로 받기 (`(() -> Unit)?`) | 자동으로 `noinline`처럼 취급 |

마지막 행은 미묘하다. 인라인 함수의 람다 파라미터가 **nullable 함수 타입**이면, `null` 검사를 위해 실체 객체가 필요하므로 컴파일러가 사실상 인라인하지 못한다. 그래서 nullable 함수 파라미터는 기본적으로 인라인되지 않는다. 인라인 이득을 원하면 nullable을 피하고 오버로드나 기본 인자로 우회한다.

> **성능 주의**: inline 함수가 람다 파라미터 여럿을 받는데 그중 일부만 `noinline`이면, 인라인되는 파라미터는 할당이 사라지고 `noinline` 파라미터만 객체로 남는다. 즉 "부분 인라인"이 가능하다. 함수 전체가 all-or-nothing이 아니다.

---

## 5. reified — 지워진 타입을 되살리다 (M18, M19 참조)

### 5.1 타입 소거라는 배경

JVM 백엔드에서 제네릭 타입 인자는 **런타임에 지워진다**(type erasure, M19의 소유는 [[34 - 제네릭 2 - 타입 소거와 reified와 바운드]]). `List<String>`과 `List<Int>`는 런타임에 똑같이 그냥 `List`다. 그래서 일반 제네릭 함수 안에서는 타입 파라미터 `T`가 실제로 무엇인지 알 수 없다.

```kotlin
fun <T> isInstance(value: Any): Boolean {
    // return value is T     // 컴파일 에러:
    // Cannot check for instance of erased type: T
    TODO()
}
```

`value is T`가 금지되는 이유는, 런타임에 `T`가 무엇인지에 대한 정보가 어디에도 없기 때문이다. 함수 시그니처에는 `T`가 있지만, 컴파일된 바이트코드에서 `T`는 `Object`로 지워진다. `T::class`도 마찬가지로 불가능하다 — 지워진 타입의 클래스 리터럴을 만들 수 없다.

Java에서는 이 문제를 `Class<T>` 파라미터를 명시적으로 넘겨서 우회한다(`Class<T> clazz`를 인자로 받아 `clazz.isInstance(value)`). Kotlin도 그렇게 할 수 있지만, 더 우아한 길이 있다 — `reified`다.

### 5.2 reified의 기계학

`inline` 함수의 타입 파라미터 앞에 `reified`를 붙이면, 그 타입 파라미터는 호출 지점에서 **구체 타입으로 치환**된다. 함수가 인라인되어 몸통이 호출 지점에 복사될 때, 컴파일러는 `T`가 나타나는 모든 자리에 **실제 타입 인자**를 박아 넣는다. 그래서 `is T`, `as T`, `T::class`가 가능해진다.

```kotlin
inline fun <reified T> isInstance(value: Any): Boolean = value is T

fun demo() {
    println(isInstance<String>("hello"))   // => true
    println(isInstance<Int>("hello"))      // => false
}
```

컴파일 시 `isInstance<String>("hello")`는 인라인되어 `"hello" is String`으로, `isInstance<Int>(...)`는 `... is Int`로 각각 치환된다. 개념적 디컴파일:

```java
// demo의 개념적 형태 (reified 치환 후)
public void demo() {
    Object v1 = "hello";
    System.out.println(v1 instanceof String);          // isInstance<String>
    Object v2 = "hello";
    System.out.println(v2 instanceof Integer);         // isInstance<Int> → Integer instanceof
}
```

`T`가 각 호출 지점에서 사라지고 실제 타입(`String`, `Integer`)이 자리를 차지했다. **런타임에 타입을 아는 것이 아니라, 컴파일 타임에 각 호출 지점마다 타입을 미리 박아 넣는 것**이다. 이 구분이 결정적이다. `reified`는 "런타임 제네릭"이 아니라 "컴파일 타임 특수화"다.

> **역사 메모**: `reified`는 Kotlin 1.0부터 있었다. Java에는 이런 기능이 없어서, Java 개발자는 항상 `Class<T>` 토큰을 손으로 넘겨야 한다. Kotlin의 `reified`는 "inline이 타입 인자를 호출 지점에 특수화한다"는 성질을 영리하게 이용한 것으로, C++ 템플릿의 인스턴스화(각 타입마다 별도 코드 생성)와 개념적으로 가깝다 — 다만 Kotlin은 함수 하나에만, inline일 때만 적용한다.

### 5.3 reified로 할 수 있는 것들

`reified T`가 있으면 다음이 모두 가능하다:

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

실전에서 가장 흔한 응용은 **역직렬화 API**다. 타입 토큰을 손으로 넘기지 않고 타입 파라미터로 추론시킨다.

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

호출자가 `Class` 토큰을 반복해서 넘기지 않아도 되므로 API가 깔끔해진다. `enumValues<T>()`, `enumValueOf<T>(name)`([[29 - enum 클래스]])도 `reified`로 구현된 표준 함수다.

### 5.4 필터링 — filterIsInstance

표준 라이브러리의 `filterIsInstance`가 `reified`의 교과서적 사례다.

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

`element is R`가 성립하려면 `R`이 실체화되어야 하고, 그러려면 함수가 `inline`이어야 한다. 그래서 `filterIsInstance`는 반드시 `inline fun <reified R>`이다. 이 함수 하나가 "왜 `inline`과 `reified`가 항상 붙어 다니는가"를 요약한다.

---

## 6. reified의 경계 — 최상위 타입만 실체화된다 (M18 정면 교정)

### 6.1 핵심 착각: "reified면 완전한 제네릭 타입을 얻는다"

M18은 이렇게 말한다: "`reified`로 완전한 제네릭 타입을 얻는다"는 오해이며, 진실은 "**최상위 타입만** 실체화되고 그 안쪽 타입 인자는 여전히 소거된다"는 것이다. 이 절이 그 오해를 정면으로 해부한다.

`reified`는 인라인이 호출 지점에 타입을 박아 넣는 기법이다. 그런데 박아 넣을 수 있는 것은 결국 **JVM의 `instanceof` 검사가 표현할 수 있는 것**뿐이다. 그리고 `instanceof`는 소거된 타입만 검사한다. 그래서 `reified T`를 `List<String>`으로 인스턴스화하면, `is T`는 `is List<String>`이 아니라 **`is List` (원소 타입 무시)** 로 귀결된다.

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

두 번째 출력이 `true`인 것이 M18의 핵심이다. `checkType<List<String>>`은 사실상 `value is List<*>`밖에 검사하지 못한다. 내부의 `String`은 런타임에 지워져 있어 검사할 수단이 없다. `reified`는 **바깥 껍데기(`List`)만 실체화**하고, 안쪽 알맹이(`String`)는 여전히 소거 상태다.

> **흔한 오해**: "`reified`가 있으니 `List<String>`과 `List<Int>`를 런타임에 구별할 수 있겠지." 아니다. JVM 백엔드에서 둘은 런타임에 똑같은 `List`이며, `reified`로도 구별할 수 없다. `reified`가 되살리는 것은 지워진 타입 계층의 **한 겹**뿐이다. 원소 타입을 진짜로 알고 싶으면 원소를 실제로 꺼내 `is`로 검사하거나(예: `filterIsInstance`가 원소별로 검사하듯), 별도의 타입 토큰을 설계해야 한다.

### 6.2 컴파일러가 아예 막는 경우

사실 `value is List<String>`을 직접 쓰면 컴파일러가 막는다:

```kotlin
fun raw(value: Any) {
    // val ok = value is List<String>   // 컴파일 에러:
    // Cannot check for instance of erased type: List<String>
    val ok = value is List<*>           // OK: 스타 프로젝션으로 원소 무시 명시
    println(ok)
}
```

Kotlin은 정직하게도 "지워진 타입은 검사할 수 없다"고 직접 검사에서는 에러를 낸다. 그런데 `reified T`를 통하면 이 에러가 **우회**된다 — `checkType<List<String>>`은 컴파일되고, 내부적으로 `is List`(스타 프로젝션 상당)로 낮춰진다. 그래서 `reified`는 편의를 주는 대신, "완전한 타입 검사"라는 착각을 심을 위험이 있다. 정확히는 이렇게 이해해야 한다: **`reified T`에서 `is T`가 검사하는 것은 `T`의 소거된 상한(erased upper bound)까지다.**

### 6.3 예외: `typeOf<T>()`는 왜 제네릭 인자를 아는가

여기 미묘한 반례가 하나 있다. 표준 라이브러리의 `typeOf<T>()`([[44 - 애노테이션과 리플렉션]])는 `reified`로 구현되었지만, `List<String>`의 `String`까지 담은 완전한 `KType`을 돌려준다.

```kotlin
import kotlin.reflect.typeOf

fun demo() {
    val t = typeOf<List<String>>()
    println(t)   // => kotlin.collections.List<kotlin.String>  ← String까지 나온다!
}
```

`is T`는 `String`을 못 봤는데 `typeOf<T>()`는 어떻게 보는가? 모순처럼 보이지만 아니다. 핵심은 **정보의 출처가 다르다**는 점이다.

- `is T`는 **JVM의 런타임 `instanceof`** 에 의존한다 → 소거된 타입만 안다 → 껍데기만.
- `typeOf<T>()`는 컴파일러가 **호출 지점의 정적 타입 정보를 데이터 객체로 만들어 주입**한다 → `String` 인자까지 통째로 `KTypeProjection` 데이터로 박아 넣는다 → 완전한 타입.

즉 `typeOf`는 런타임 타입 검사를 하는 게 아니라, 컴파일 타임에 알던 타입 구조를 **직렬화한 메타데이터**를 런타임에 그대로 돌려주는 것이다. 이것은 M18에 대한 반례가 아니라 M18의 정확한 경계를 보여준다: **런타임 `is`/`as` 검사**는 최상위 타입까지만, **컴파일러가 주입하는 메타데이터**(`typeOf`, 리플렉션)는 완전한 타입을 담을 수 있다.

```text
reified T 로 얻는 것의 두 층위
──────────────────────────────
value is T        →  JVM instanceof  →  최상위 타입만  (List<String> → List)
value as T        →  JVM checkcast   →  최상위 타입만  (안쪽 미검사)
T::class          →  KClass          →  최상위 타입만  (List::class, 인자 없음)
typeOf<T>()       →  컴파일러 주입    →  완전한 KType  (List<String> 전체)
```

`T::class`가 `KClass<List<*>>`(즉 `List::class`)를 주는 것도 같은 원리다. `KClass`는 "클래스"를 대표하므로 타입 인자를 담지 않는다. 타입 인자까지 필요하면 `KType`(`typeOf`)을 써야 한다. 이 계층 구분이 [[44 - 애노테이션과 리플렉션]]의 `KClass` vs `KType` 논의로 이어진다.

### 6.4 reified로 인스턴스를 만들 수 있는가

흔한 욕심: "`reified T`가 있으니 `T()`로 인스턴스를 만들 수 있겠지?"

```kotlin
inline fun <reified T> create(): T {
    // return T()      // 컴파일 에러: Type parameter T cannot be called as function
    return T::class.java.getDeclaredConstructor().newInstance()  // 리플렉션으로는 가능(JVM)
}
```

`T()` 직접 호출은 안 된다 — `T`에 기본 생성자가 있는지 컴파일러가 보장할 수 없기 때문이다. 하지만 `reified`로 `T::class.java`를 얻을 수 있으니, 리플렉션을 통해 우회할 수 있다(JVM 백엔드, 기본 생성자가 있고 접근 가능할 때만; 없으면 런타임 예외). 이것은 `reified`가 "타입을 아는 것"과 "타입을 생성하는 것"이 별개임을 보여준다. `reified`는 앎을 주지, 생성을 주지 않는다.

---

## 7. inline의 대가와 제약

### 7.1 코드 팽창 — 복사의 필연적 비용

§2.3의 다이어그램이 예고했듯, inline 함수는 **호출될 때마다 몸통이 복사**된다. 호출 지점이 100군데면 몸통이 100번 복제되어 바이너리가 커진다. 이것은 성능상 함정이 될 수 있다 — 코드가 커지면 명령어 캐시(instruction cache) 압박이 늘고, 오히려 느려질 수 있다.

그래서 관례는 명확하다: **inline 함수는 작아야 한다.** 표준 라이브러리의 inline 함수들(`let`, `run`, `forEach`, `map` 등)은 대부분 몇 줄짜리다. 몸통이 큰 함수에 `inline`을 붙이면 각 호출 지점이 비대해진다.

> **성능 주의**: 큰 inline 함수를 여러 곳에서 부르면, 람다 할당을 아낀 이득보다 코드 팽창의 손해가 클 수 있다. 흔한 절충: **바깥 껍데기(람다 받는 부분)만 작게 inline하고, 무거운 실제 작업은 비-inline private 함수로 빼서 호출**한다. 그러면 각 호출 지점엔 작은 껍데기만 복사되고, 무거운 몸통은 한 곳에만 존재한다.

```kotlin
inline fun <T> measured(label: String, block: () -> T): T {
    val start = System.nanoTime()
    val result = block()             // 람다는 인라인
    logDuration(label, start)        // 무거운 로직은 비-inline 함수로 위임 → 복사 안 됨
    return result
}

fun logDuration(label: String, startNanos: Long) { /* 큰 몸통, 한 곳에만 존재 */ }
```

### 7.2 public inline 함수의 가시성 규칙 — private 접근 금지

이것이 실무에서 사람들을 가장 자주 당황시키는 제약이다. **`public`(또는 `protected`) inline 함수는 자기 클래스/파일의 `private` 멤버를 참조할 수 없다.**

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

이유는 §2의 물리적 복사에 있다. `public inline` 함수는 그 몸통이 **다른 모듈의 호출 지점에 복사**된다. 그 다른 모듈은 `Cache`의 `private store`에 접근 권한이 없다. 복사된 코드가 `store`를 참조하면, 접근 불가능한 멤버를 건드리게 되어 캡슐화가 깨진다. 그래서 컴파일러는 애초에 금지한다.

우회책은 `@PublishedApi`다. 이 어노테이션을 `internal` 멤버에 붙이면, "public inline 함수 안에서 쓰라고 공개한 내부 API"라고 표시하는 것이다.

```kotlin
class Cache {
    @PublishedApi
    internal val store = HashMap<String, Any>()   // internal + @PublishedApi

    inline fun <reified T> getOrNull(key: String): T? = store[key] as? T   // OK
}
```

`@PublishedApi internal`은 소스에서는 `internal`(모듈 내부, [[26 - 가시성 한정자]])이지만, inline 함수의 몸통에 복사되어 다른 모듈로 노출되어도 괜찮도록 컴파일러가 이름을 유지해 준다. 다만 이렇게 노출된 멤버는 **바이너리 호환성의 일부**가 되므로, 함부로 시그니처를 바꾸면 다른 모듈의 컴파일된 코드가 깨질 수 있다. `private`을 그냥 못 쓰는 게 아니라, "노출을 의식적으로 선언하라"는 것이다.

> **명세 정밀**: 이 제약은 함수의 가시성에 따라 달라진다. `private inline` 함수는 호출 지점이 같은 파일/클래스 안에만 있으므로 `private` 멤버를 마음껏 참조할 수 있다. `internal inline` 함수도 같은 모듈 안에서만 인라인되므로 `internal` 멤버를 참조할 수 있다. 문제는 오직 **모듈 경계를 넘어 복사되는 `public`/`protected` inline 함수**뿐이다.

### 7.3 재귀 불가

inline 함수는 **자기 자신을 (직접이든 간접이든) 호출할 수 없다**. 인라인은 호출을 몸통 복사로 바꾸는데, 재귀는 무한히 복사를 유발하기 때문이다.

```kotlin
inline fun factorial(n: Int): Int {
    // return if (n <= 1) 1 else n * factorial(n - 1)
    // 컴파일 에러: Inline function 'factorial' cannot be recursive
    TODO()
}
```

재귀가 필요하면 `inline`을 떼거나, 꼬리 재귀는 `tailrec`([[12 - 함수 - 인자와 vararg와 지역 함수와 꼬리 재귀]])로 별도 최적화한다. 이 둘은 완전히 다른 기법이다 — `inline`은 호출 지점 복사, `tailrec`은 재귀를 루프로 변환.

### 7.4 디버깅과 스택 트레이스

인라인된 함수는 자체 스택 프레임을 남기지 않는다. 예외가 나면 스택 트레이스에 inline 함수의 이름이 안 보이고, 호출자 위치가 뒤섞여 보일 수 있다(현대 컴파일러는 디버그 정보로 inline 위치를 표시하려 하지만 완벽하지 않다). 브레이크포인트를 inline 함수 안에 걸 때도 각 호출 지점마다 다르게 동작할 수 있다.

> **성능 주의**: inline이 스택 프레임을 없애는 건 성능엔 이득(프레임 push/pop 비용 제거)이지만, 관측 가능성엔 손해다. 아주 뜨거운 경로가 아니라면, 디버깅·프로파일링 편의를 위해 inline을 자제하는 게 나을 때도 있다.

### 7.5 인라인이 무의미하거나 경고를 부르는 경우

- **람다 파라미터가 없는 함수**: §2.2에서 봤듯 컴파일러가 "인라인 효과 미미" 경고를 낸다. 예외적으로 `reified`가 필요하면 람다 없이도 inline이 정당하다(그때는 경고가 안 난다 — reified가 인라인을 필수로 만드니까).
- **모든 람다 파라미터가 `noinline`**: 인라인할 람다가 하나도 없으니 이득이 없다.
- **거대한 함수**: 코드 팽창만 유발.

```kotlin
// 정당한 inline: 람다 파라미터가 있음
inline fun <T> withLock(lock: Lock, action: () -> T): T { /* ... */ TODO() }

// 정당한 inline: reified가 필요함 (람다 없어도 OK)
inline fun <reified T> typeName(): String = T::class.simpleName ?: "?"

// 의심스러운 inline: 람다도 reified도 없음 → 컴파일러 경고
inline fun add(a: Int, b: Int): Int = a + b   // 경고: 인라인 이득 미미
```

---

## 8. inline의 다른 얼굴들 — 프로퍼티, 표준 라이브러리, 그리고 이름 충돌

### 8.1 inline 프로퍼티 접근자

`inline`은 함수뿐 아니라 **프로퍼티 접근자**에도 붙는다. 백킹 필드([[22 - 프로퍼티와 백킹 필드]])가 없는 계산 프로퍼티의 getter/setter를 인라인할 수 있다.

```kotlin
val Int.isEven: Boolean
    inline get() = this % 2 == 0     // getter만 inline

var displayName: String
    inline get() = computeName()      // 두 접근자 각각 inline 가능
    inline set(value) { storeName(value) }
```

프로퍼티 전체에 `inline`을 붙이면(`inline val`/`inline var`) 접근자 전부가 인라인된다. 단, 백킹 필드가 있는 프로퍼티는 inline할 수 없다 — 필드 접근은 인라인의 대상이 아니기 때문이다. 프로퍼티는 타입 파라미터를 가질 수 없으므로 `reified`는 프로퍼티에 직접 쓸 수 없다(reified는 함수의 타입 파라미터 전용).

### 8.2 표준 라이브러리는 inline으로 지어졌다

Kotlin 표준 라이브러리의 상당 부분이 inline 함수다. 이 목록을 아는 것이 "왜 내 코드가 손으로 쓴 루프만큼 빠른가"를 설명한다.

| 함수 | inline인가 | 비지역 return | reified |
|------|-----------|--------------|---------|
| `let`, `run`, `with`, `apply`, `also` | O | O | — |
| `takeIf`, `takeUnless` | O | O | — |
| `repeat(n) { }` | O | O | — |
| `forEach`, `forEachIndexed`, `onEach` | O | O | — |
| `filter`, `map`, `flatMap` (Iterable) | O | O | — |
| `runCatching` | O | O | — |
| `use` (Closeable) | O | O | — |
| `synchronized`, `withLock` | O | O | — |
| `filterIsInstance<R>()` | O | — | O |
| `enumValues<T>()`, `enumValueOf<T>()` | O | — | O |
| `typeOf<T>()` | O | — | O (특수) |
| `arrayOf`, `emptyList` | — (비-inline) | — | — |

`map`, `filter` 같은 컬렉션 연산이 inline이라서, 함수형 파이프라인([[39 - 컬렉션 연산과 함수형 파이프라인]])이 람다 할당 없이 돌아간다. 다만 이들은 **즉시 평가**라 중간 리스트를 만든다(M28) — 그건 inline과 별개 문제로, 지연 평가는 `Sequence`([[40 - 시퀀스와 지연 평가]])에서 다룬다. inline은 "람다 호출 오버헤드"를 없앨 뿐, "중간 컬렉션 생성"을 없애진 않는다. 두 최적화를 혼동하면 안 된다.

### 8.3 스코프 함수와 DSL의 성능적 토대

[[16 - 스코프 함수와 수신 객체 관용구]]의 `let`/`apply` 등이 모두 inline이라, 아무리 중첩해 써도 람다 객체가 안 생긴다.

```kotlin
fun buildUser(name: String): User =
    User().apply {          // apply는 inline → 람다 할당 없음
        this.name = name
        this.createdAt = now()
    }.also {                // also도 inline
        log("created: ${it.name}")
    }
```

이 체이닝은 우아하면서도 손으로 필드를 세팅한 것과 동일한 바이트코드를 낸다. [[37 - 타입 안전 빌더와 DSL]]의 DSL이 성능 걱정 없이 깊게 중첩될 수 있는 것도, 그 바탕의 수신 객체 지정 람다들이 inline이기 때문이다. "우아함의 무료성(zero-cost abstraction)"은 inline이 떠받친다.

### 8.4 이름 충돌 정리 — inline function vs inline value class

마지막으로, 이 장 내내 미뤄둔 이름 혼동을 정리한다. `inline`이라는 단어가 Kotlin에 두 군데 등장한다.

```kotlin
// (1) 이 장의 주제: inline 함수 — 호출 지점 복사
inline fun <T> measure(block: () -> T): T { /* ... */ TODO() }

// (2) 완전히 다른 기능: inline value class — 래퍼 박싱 회피 (36장)
@JvmInline
value class UserId(val raw: Long)
```

둘은 이름만 비슷할 뿐 **아무 관련이 없다**.

| | inline **함수** (15장) | inline **value class** (36장) |
|---|---|---|
| 대상 | 함수/프로퍼티 접근자 | 단일 값을 감싸는 클래스 |
| 목적 | 람다 할당·호출 제거, reified | 래퍼 객체 할당 제거(값 자체를 그대로 사용) |
| 키워드 | `inline fun` | `@JvmInline value class` (옛 `inline class`) |
| 관련 M | M17, M18, M41 | M38, M39 |

역사적으로 값 클래스는 `inline class`라는 키워드로 실험 도입되었다가, 개념이 함수 인라인과 헷갈린다는 이유로 `value class`(+`@JvmInline`)로 이름이 바뀌었다. 그 잔재로 어노테이션에 `Inline`이 남았을 뿐이다. `value class`의 박싱 회피 조건(nullable·제네릭·인터페이스 상향변환 시 박싱됨, M39)은 [[36 - 타입 별칭과 인라인 value class]]가 소유한다. 이 장에서는 "둘은 다른 기능"이라는 것만 기억하면 된다.

---

## 9. 언제 inline을 쓰고, 언제 삼가는가

### 9.1 써야 하는 경우

`inline`은 다음 셋 중 하나라도 해당하면 정당하다:

1. **람다 파라미터를 받고, 뜨거운 경로에서 호출된다.** 람다 할당과 가상 호출을 없애는 것이 실측 이득이 되는 고차 함수. `forEach`, `withLock`, `use` 부류.
2. **`reified` 타입 파라미터가 필요하다.** 이건 선택이 아니라 필수 — `reified`는 `inline` 없이 존재할 수 없다. `filterIsInstance`, `parseJson`, `typeOf` 부류.
3. **비지역 반환을 제어 구조로 제공하고 싶다.** 사용자가 람다 안에서 바깥 함수를 종료할 수 있게 하려면 inline이어야 한다. 커스텀 제어 흐름 DSL.

### 9.2 삼가야 하는 경우

1. **람다도 reified도 없는 평범한 함수.** JIT가 알아서 인라인한다. 소스 인라인은 바이너리만 부풀린다.
2. **몸통이 큰 함수를 여러 곳에서 호출.** 코드 팽창이 이득을 삼킨다. 껍데기만 inline하고 몸통은 위임(§7.1).
3. **디버깅·프로파일링이 중요한 코드.** 스택 프레임 소실이 관측을 방해한다.
4. **public API인데 private 상태에 크게 의존.** `@PublishedApi`로 노출을 강제당하고, 바이너리 호환성 부담이 생긴다.

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

### 9.4 한 걸음 물러서서

`inline`의 세 얼굴 — 성능(할당 제거), 제어(비지역 반환), 타입(reified) — 은 모두 "몸통을 호출 지점에 복사한다"는 **하나의 물리적 사실**에서 파생된다. 이 통일된 관점을 잡으면, 겉보기에 무관해 보이는 규칙들(왜 재귀가 안 되는지, 왜 public inline이 private을 못 보는지, 왜 reified가 inline을 요구하는지, 왜 비지역 반환이 inline에서만 되는지)이 전부 같은 원리의 따름정리로 정리된다. `inline`은 어노테이션이 아니라 **컴파일러에게 코드의 물리적 형태를 재배치하라는 명령**이며, 그 재배치의 부작용이 곧 언어 의미론이다.

---

이 장은 람다의 숨은 비용에서 출발했다. JVM 백엔드에서 함수 타입은 인터페이스이고, 캡처하는 람다는 힙 객체이며, 뜨거운 루프에서 그 객체의 `invoke`를 반복 호출하는 것은 가상 디스패치와 박싱의 이중고였다(§1). `inline`은 이 비용을 "힌트"로 줄이는 게 아니라, 함수와 람다의 몸통을 호출 지점에 **물리적으로 복사**해 아예 없앤다(§2, M17). 그 복사가 세 가지 언어 규칙을 파생시킨다 — inline 람다 안의 `return`이 바깥 함수를 종료하는 비지역 반환(§3, M41), 그것을 통제하는 `crossinline`·`noinline`(§4), 그리고 타입 인자가 호출 지점에 특수화되어 가능해지는 `reified`(§5)다. 그러나 `reified`의 마법에는 정확한 경계가 있다: 되살아나는 것은 **최상위 소거 타입**뿐이고, `List<String>`의 `String`은 여전히 지워져 있다(§6, M18) — 단 `typeOf`처럼 컴파일러가 메타데이터를 주입하는 경로는 완전한 타입을 담을 수 있다는 반전과 함께. 이 모든 능력에는 대가가 따른다: 코드 팽창, public inline의 가시성 족쇄, 재귀 불가, 디버깅 저하(§7). 그래서 `inline`은 남발할 도구가 아니라, 람다·reified·비지역 반환이라는 명확한 이유가 있을 때 정확히 겨누는 도구다(§8, §9). 도입부의 질문 — "inline은 힌트인가"에 대한 답은 이제 분명하다. 아니다. inline은 코드의 물리적 형태를 바꾸는 명령이고, 그 형태 변화가 곧 관찰 가능한 의미론이다.

## 핵심 요약

- **`inline`은 성능 힌트가 아니라 의미론적 명령이다(M17).** 컴파일러는 함수 몸통과 넘어온 람다 몸통을 호출 지점에 물리적으로 복사한다. 이 복사가 성능(할당 제거)·제어(비지역 반환)·타입(reified)이라는 세 가지 관찰 가능한 규칙을 파생시킨다. inline을 떼면 속도만 느려지는 게 아니라, 비지역 반환·reified를 쓰는 코드는 아예 컴파일이 안 된다.
- **비-inline 람다는 JVM 백엔드에서 `FunctionN` 인터페이스 객체이고, 캡처하면 힙에 할당된다.** 원시 타입이 관여하면 박싱까지 얹힌다. 뜨거운 루프에서 이 가상 호출과 박싱이 반복되는 것이 inline이 없애려는 진짜 비용이다. (무상태 람다의 정확한 표현은 JVM 백엔드 버전·옵션에 따라 다르다 — 2.0부터 invokedynamic 기본.)
- **inline 람다 안의 벌거벗은 `return`은 바깥 함수를 종료한다(비지역 반환, M41).** 람다 몸통이 바깥 함수 몸통에 복사되기 때문이다. 비-inline 람다에서는 벌거벗은 `return`이 금지되고 `return@label`(지역 반환)만 허용된다. `forEach`를 `continue`처럼 쓰려면 `return@forEach`, 함수를 끝내려면 `return`.
- **`crossinline`은 인라인은 유지하되 비지역 반환만 금지한다.** 람다를 익명 객체나 다른 실행 문맥(예: `Runnable`) 안에서 호출할 때 필요하다. **`noinline`은 인라인 자체를 끈다** — 람다를 변수에 저장·전달·반환해야 할 때. nullable 함수 타입 파라미터는 자동으로 인라인되지 않는다.
- **`reified`는 런타임 제네릭이 아니라 컴파일 타임 특수화다.** inline 함수의 타입 파라미터가 각 호출 지점에서 구체 타입으로 치환되므로 `is T`, `as T`, `T::class`가 가능해진다. `reified`는 `inline` 없이는 존재할 수 없다. Java의 `Class<T>` 토큰 전달을 우아하게 대체한다.
- **`reified`가 되살리는 것은 최상위 소거 타입 한 겹뿐이다(M18).** `checkType<List<String>>(intList)`는 `true`를 반환한다 — 런타임 `is`는 `is List`까지만 검사하고 안쪽 `String`은 여전히 지워져 있기 때문이다. `List<String>`과 `List<Int>`는 reified로도 구별할 수 없다(M19 참조).
- **단, 컴파일러가 메타데이터를 주입하는 경로는 완전한 타입을 담는다.** `typeOf<List<String>>()`는 `String`까지 담은 `KType`을 돌려준다 — 런타임 `instanceof`가 아니라 컴파일 타임 정적 타입을 데이터로 직렬화하기 때문이다. `T::class`(KClass)는 최상위만, `typeOf`(KType)는 완전한 타입. 이 계층 구분이 리플렉션([[44 - 애노테이션과 리플렉션]])의 핵심이다.
- **`public`/`protected` inline 함수는 `private` 멤버에 접근할 수 없다.** 몸통이 다른 모듈의 호출 지점에 복사되는데, 그 모듈은 private에 접근 권한이 없기 때문이다. `@PublishedApi internal`로 의식적으로 노출해야 한다. `private inline`·`internal inline`은 이 제약이 없다.
- **inline 함수는 재귀할 수 없고, 호출마다 몸통이 복사되어 바이너리를 부풀린다.** 그래서 inline 함수는 작아야 한다. 큰 작업은 껍데기만 inline하고 무거운 몸통은 비-inline 함수로 위임한다. 스택 프레임이 사라져 디버깅·프로파일링이 어려워지는 것도 대가다.
- **표준 라이브러리의 스코프 함수·컬렉션 연산·`use`·`repeat`는 대부분 inline이다.** 그래서 함수형 파이프라인과 DSL이 람다 할당 없이 손으로 쓴 루프와 동일한 바이트코드를 낸다 — 무료 추상화의 정체다. 단 컬렉션 `map`/`filter`의 즉시 평가(중간 리스트 생성, M28)는 inline과 별개 문제로, 지연 평가는 `Sequence`가 담당한다.
- **`inline fun`(이 장)과 `@JvmInline value class`(옛 `inline class`, 36장)는 이름만 닮은 완전히 다른 기능이다.** 전자는 호출 지점 복사(람다·reified), 후자는 래퍼 박싱 회피(M39). 역사적 이름 혼동일 뿐 아무 관련이 없다.
- **inline은 붙일지 말지를 판단해서 겨누는 도구다.** reified가 필요하면 필수, 람다 받는 뜨거운 경로면 이득, 그 외 평범한 함수엔 붙이지 마라(JIT가 알아서 한다 — 컴파일러가 경고도 준다).

## 연결 노트

- [[13 - 함수 타입과 람다와 함수 참조]] — 이 장이 없애려는 `FunctionN` 인터페이스 표현과 SAM 변환, `it`(M42)의 출처. 람다의 "무엇"을 다룬 장이며, 이 장은 그 람다의 "비용과 배치"를 다룬다.
- [[14 - 클로저와 캡처]] — 캡처하는 람다가 왜 힙 객체(그리고 `var` 캡처를 위한 참조 셀)로 할당되는지의 원리. inline이 이 할당을 없애는 이유의 근거이자, 비지역 반환(M41)이 처음 언급된 장.
- [[34 - 제네릭 2 - 타입 소거와 reified와 바운드]] — 타입 소거(M19)의 전모와 reified의 제네릭 이론적 자리. 이 장이 실무 관점에서 다룬 reified의 소거 배경을 소유한다.
- [[33 - 제네릭 1 - 타입 파라미터와 변성]] — 타입 파라미터·변성·상한의 일반론. `reified T`도 결국 타입 파라미터이며, 그 기본 상한(`Any?`, M45)이 `is T` 검사의 범위를 정한다.
- [[36 - 타입 별칭과 인라인 value class]] — 이름이 헷갈리는 `@JvmInline value class`(옛 `inline class`)의 소유 장. 박싱 회피(M39)를 다루며, 이 장의 inline 함수와는 무관함을 §8.4에서 못박았다.
- [[16 - 스코프 함수와 수신 객체 관용구]] — `let`/`run`/`with`/`apply`/`also`가 모두 inline이라 비지역 반환과 무할당이 가능한 이유. 이 장의 inline 의미론이 스코프 함수의 매끄러움을 떠받친다.
- [[37 - 타입 안전 빌더와 DSL]] — 수신 객체 지정 람다 기반 DSL이 성능 걱정 없이 깊게 중첩되는 토대가 inline임을 §8.3에서 예고했다.
- [[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]] — inline·reified가 컴파일 파이프라인의 어느 단계에서 처리되는지, 디컴파일로 결과를 확인하는 법(§2.3)의 배경.
- [[44 - 애노테이션과 리플렉션]] — `T::class`(KClass) vs `typeOf<T>()`(KType)의 계층 구분(§6.3)이 리플렉션의 핵심으로 이어진다.
- [[41 - 예외와 Nothing]] — inline 함수 `use`·`runCatching`과 비지역 반환의 결합, 그리고 리소스 관리에서 inline이 쓰이는 방식.
