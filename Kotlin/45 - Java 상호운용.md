---
title: Java 상호운용
date: 2026-07-13
tags: [kotlin, interop, java, jvm, platform-types, 학습노트]
---

[[44 - 애노테이션과 리플렉션]]에서는 `.class` 파일 안에 이름과 타입, 애노테이션이 어떻게 기록되는지, 그리고 Kotlin 메타데이터가 리플렉션으로 어떻게 복원되는지를 살펴보았다. 44장이 "컴파일된 산출물을 런타임에 다시 읽는" 이야기였다면, 이 장에서는 그 산출물이 처음부터 **다른 언어(Java)와 같은 바이트코드 위에서 만난다**는 사실을 다룬다.

Kotlin/JVM은 독자적인 기반 위에서 실행되지 않는다. 20년 넘게 Java를 위해 다듬어진 JVM 위에서 실행된다. Kotlin이 "실용주의"([[01 - 코틀린이란 무엇인가 - 설계 철학과 세 개의 타깃]])를 가장 중요한 가치로 내세운 이유의 절반은 이 JVM 생태계와 매끄럽게 어울리기 위해서였다.

그런데 두 언어가 같은 바이트코드로 컴파일된다고 해서 **타입 시스템까지 같은 것은 아니다**. Kotlin은 널 가능성을 타입에 담았지만([[06 - 널 안전성 - nullable와 스마트 캐스트와 플랫폼 타입]]), Java의 타입에는 그 정보가 없다.

Kotlin은 읽기 전용 `List`와 가변 `MutableList`를 구분하지만([[38 - 컬렉션 - 읽기 전용과 가변]]), Java의 `java.util.List`는 하나뿐이다. Kotlin에는 모듈 단위 가시성 `internal`이 있지만([[26 - 가시성 한정자]]), JVM 바이트코드에는 그런 접근 등급이 없다.

상호운용은 이런 **불일치를 메우는 기술**이다. 불일치를 메운 경계가 깨지는 곳에서 버그가 생기므로, 이 장에서는 그 경계를 하나씩 따라가 본다. 이 장에서 다루는 것은 **JVM 백엔드에서** Kotlin과 Java가 상호작용하는 규칙이다.

구체적으로는 두 타입 시스템이 만나는 가장 위험한 경계인 플랫폼 타입과 이를 다루는 널 애노테이션, Java 멤버가 Kotlin에서 보이는 방식과 프로퍼티 합성, Kotlin 선언을 Java에서 쓰기 좋게 만드는 `@Jvm*` 애노테이션, `internal`의 실제 모습과 이름 맹글링, 검사 예외가 없는 Kotlin과 `@Throws`([[41 - 예외와 Nothing]]), 타입 매핑을 차례로 살펴보고, 마지막으로 실전에서 자주 만나는 함정을 정리한다.

Kotlin/Native가 C, Objective-C, Swift와, Kotlin/JS가 JavaScript, TypeScript와 상호작용하는 방식은 [[46 - 멀티플랫폼과 진화와 어두운 구석]]에서 다룬다.

---

## 1. 두 언어가 만나는 곳: 상호운용이 왜 존재 이유인가

### 1.1 같은 바이트코드, 다른 언어

Kotlin/JVM 컴파일러는 `.kt` 소스를 받아 `.class` 파일을 만든다([[02 - 컴파일러의 해부 - K2와 IR 백엔드와 바이트코드]]). 이 `.class` 파일은 `javac`가 만든 `.class`와 **바이트코드 수준에서 구별되지 않는다**. JVM은 어느 언어가 만든 파일인지 모르는 채로 실행한다.

그래서 Kotlin 클래스는 Java의 클래스패스에 그대로 올라가고, Java 클래스도 Kotlin의 클래스패스에 그대로 올라간다. 한 프로젝트 안에서 `.kt`와 `.java`를 섞어 컴파일하는 일도 흔하다.

두 언어가 같은 바이트코드를 공유한다는 점이 상호운용의 물리적 기반이다. Kotlin에서 Java의 `ArrayList`를 쓰는 것은 외부 라이브러리를 호출하는 것이 아니라 **같은 런타임의 같은 클래스**를 참조하는 것이다.

Kotlin의 `String`은 실제로 `java.lang.String`이고, Kotlin의 `Int`는 문맥에 따라 JVM의 `int`가 되거나 `java.lang.Integer`가 된다([[07 - 수 1 - Int와 Long과 부동소수점]]). 이것이 8절에서 다룰 **매핑된 타입(mapped types)** 이다.

```text
   Kotlin 소스 (.kt)              Java 소스 (.java)
        │                              │
   kotlinc                          javac
        │                              │
        ▼                              ▼
   ┌──────────────── JVM 바이트코드 (.class) ────────────────┐
   │  Animal.class, AnimalKt.class, java/util/List.class ... │
   └───────────────────────────┬────────────────────────────┘
                               ▼
                       JVM (실행 시 구별 불가)
```

> [!info] 역사 메모
> JetBrains가 Kotlin을 만들 때 "완벽한 새 언어"가 아니라 "Java와 100% 상호운용되는 실용 언어"를 목표로 삼은 것은 전략적인 선택이었다. 기존 Java 코드베이스에 Kotlin을 **파일 단위로 점진적으로 도입**할 수 있어야 시장에 들어갈 수 있었기 때문이다. Scala, Ceylon 같은 JVM 대안 언어와의 경쟁에서 Kotlin이 이긴 결정적인 요인 중 하나도 이처럼 이질감 없는 상호운용이었다.

### 1.2 상호운용의 두 방향은 대칭이 아니다

상호운용에는 두 방향이 있지만, 방향마다 난이도가 다르다.

- **Java → Kotlin (Java에서 Kotlin 호출)**: Kotlin이 Java에서 쓰기 좋은 바이트코드를 만들도록 개발자가 **`@Jvm*` 애노테이션으로 표시**해야 한다. Java에는 프로퍼티, 기본 인자, 최상위 함수, companion 같은 개념이 없기 때문이다.
- **Kotlin → Java (Kotlin에서 Java 호출)**: 대체로 투명하게 동작한다. 하지만 Java의 타입에는 **널 가능성 정보가 없으므로**, 플랫폼 타입이라는 위험 지대가 생긴다.

이 비대칭을 기준으로 이 장의 전반부(2, 3, 4절: Kotlin이 Java를 볼 때)와 후반부(5, 6절: Java가 Kotlin을 볼 때)가 나뉜다.

### 1.3 불일치를 메운 경계가 곧 함정이다

두 타입 시스템의 표현력 차이를 표로 정리해 두면 이후 절을 읽을 때 지도로 쓸 수 있다.

| Kotlin이 가진 것 | Java의 대응물 | 메운 결과 | 관련 절 |
|---|---|---|---|
| 널 가능성 타입(`T`, `T?`) | 없음(애노테이션은 선택) | 플랫폼 타입 `T!` | 2·3 |
| 읽기 전용/가변 컬렉션 구분 | `java.util.List` 하나 | 읽기 전용 보장이 Java에서 사라짐 | 8 |
| 프로퍼티(`val`/`var`) | 필드 + getter/setter 관례 | getter/setter ↔ 프로퍼티 합성 | 4 |
| 최상위 함수 | 없음(클래스만) | `FileKt` 파사드 클래스 | 5 |
| `companion object` | `static` 멤버 | `Companion` 필드 + `@JvmStatic` | 5 |
| 기본 인자 | 오버로딩 | `@JvmOverloads` | 5 |
| `internal` 가시성 | 없음(패키지·public만) | 이름 맹글링 | 6 |
| 검사 예외 없음 | 검사 예외 있음 | `@Throws` | 7 |
| `Unit`/`Nothing` 타입 | `void`/없음 | `void`/`Void` 매핑 | 8 |

표의 각 행은 잘 메워져 있으면 드러나지 않지만, 모르고 지나치면 큰 문제를 일으키는 지점이다.

---

## 2. 플랫폼 타입: 널 안전성의 경계

### 2.1 문제의 근원: Java 타입에는 널 정보가 없다

Kotlin의 널 안전성은 "타입에 널 가능 여부가 담겨 있다"는 전제를 바탕으로 한다(6장 널 안전성). `String`은 절대 null이 아니고, `String?`은 null일 수 있다. 컴파일러는 이 정보를 이용해 **컴파일 타임에** NPE를 차단한다.

문제는 Java다. Java의 `String`은 null일 수도 있고 아닐 수도 있어서, 타입만 봐서는 알 수 없다. 다음 Java 클래스를 생각해 보자.

```java
// Java
public class Shelter {
    public String findName(int id) {
        // 못 찾으면 null을 돌려줄 수도 있다 — 타입엔 안 나타난다
        return id == 0 ? "Rex" : null;
    }
}
```

Kotlin에서 이 `findName`을 호출할 때 반환 타입을 `String`으로 취급하면 null이 섞여 들어와 위험하다. 반대로 `String?`으로 취급하면 실제로 null이 아닌 경우에도 매번 널 검사를 해야 하므로, Java와의 상호운용이 무척 번거로워진다. Kotlin은 세 번째 방법인 **플랫폼 타입(platform type)** 을 선택했다.

### 2.2 플랫폼 타입 `T!`의 정체: 유연한 타입

Java에서 넘어온 타입에 널 애노테이션이 없으면, 컴파일러는 이를 **플랫폼 타입**으로 취급하고 내부적으로 `T!`로 표기한다. `T!`는 실제로는 `T`(비-널)와 `T?`(널 가능) 사이의 **유연한 타입(flexible type)** `(T..T?)`이다. 핵심은 다음과 같다.

> [!note] 명세 기준
> 플랫폼 타입 `T!`는 **개발자가 소스에 직접 쓸 수 없다**. `val x: String! = ...`는 문법 오류다(`컴파일 에러: unexpected token`). `!` 표기는 IDE의 타입 힌트나 컴파일러 진단 메시지에만 나타난다. 이 표기는 "컴파일러만 알고 있는, 널 검사를 유예한 상태"를 뜻하는 내부 표기다.

```kotlin
val shelter = Shelter()

// findName의 반환은 플랫폼 타입 String!
val a = shelter.findName(0)          // a의 추론 타입: String! (유연)
val b: String = shelter.findName(0)  // 비-널로 못박음: 대입 시점엔 검사 안 함
val c: String? = shelter.findName(1) // 널 가능으로 못박음: 안전

println(a.length)   // 컴파일 통과! 하지만 findName이 null이면 런타임 NPE
println(c?.length)  // 안전 호출 강제됨 — 컴파일러가 c를 String?로 알기 때문
```

`a`처럼 플랫폼 타입을 그대로 두면 컴파일러는 `a.length` 같은 **비-널 사용을 막지 않는다**. 널 검사의 책임을 개발자에게 넘긴 것이다. 바로 이 지점에서 "널 안전한 Kotlin"에서도 NPE가 다시 발생한다.

### 2.3 플랫폼 타입은 "안전"하지 않다

> [!warning] 흔한 오해
> "플랫폼 타입 `T!`는 컴파일러가 알아서 잘 처리하니 안전하다"는 생각은 **틀렸다.** `T!`는 안전한 타입이 아니라 **컴파일러가 널 검사를 유예(suspend)한** 타입이다. 유예한다는 것은 "지금은 검사하지 않지만, 비-널로 다루다가 null이 오면 예외가 발생한다"는 뜻이다. 안전성을 지켜 주는 컴파일 타임 검사가 이 경계에서만 빠진다.

개념은 6장에서 설명했지만, 그 위험이 **실제로 나타나는 곳은 이 장에서 다루는 상호운용**이다. 플랫폼 타입은 Java(또는 널 정보가 없는 다른 상호운용 경계)에서만 생기기 때문이다. 순수 Kotlin 코드에서는 플랫폼 타입이 생길 수 없다.

방어 원칙은 하나다. **Java에서 값을 받는 즉시 명시적인 타입으로 확정한다.**

```kotlin
// 나쁨: 플랫폼 타입이 코드 깊숙이 흘러다닌다
fun processBad(shelter: Shelter) {
    val name = shelter.findName(1)   // String! — 위험이 잠복
    // ... 수십 줄 뒤 ...
    println(name.uppercase())        // 여기서 NPE. 원인은 저 위 findName
}

// 좋음: 경계에서 즉시 널 가능성을 확정
fun processGood(shelter: Shelter) {
    val name: String? = shelter.findName(1)  // 못박음
    println(name?.uppercase() ?: "이름 없음") // 컴파일러가 강제
}
```

> [!caution] 성능 주의
> 플랫폼 타입을 비-널 타입으로 확정하면(`val b: String = javaCall()`) 컴파일러는 그 지점에 **인트린식 널 검사**(`Intrinsics.checkNotNullExpressionValue`)를 넣을 수 있다. 이 검사는 값이 실제로 null이면 사용 지점이 아니라 대입 지점에서 바로 `NullPointerException`을 던지므로, 예외가 발생하는 곳과 원인이 있는 곳이 가까워진다.
>
> 이런 조기 검사는 디버깅에 유리하지만 약간의 비용이 든다. `-Xno-call-assertions` 같은 플래그로 끌 수 있지만, 대개는 끄지 않는 것이 좋다.

### 2.4 플랫폼 타입의 전파와 제네릭

플랫폼 타입은 제네릭 인자에도 붙는다. Java의 `List<String>`을 반환하는 메서드는 Kotlin에서 `(Mutable)List<String!>!`로 보인다. 리스트 자체도 플랫폼 타입이라 null일 수 있고, 원소도 플랫폼 타입이라 null일 수 있다. 위험이 두 겹인 셈이다.

```kotlin
// Java: public List<String> allNames()
val names = shelter.allNames()   // (Mutable)List<String!>!
for (n in names) {               // names가 null이면 여기서 NPE
    println(n.length)            // n이 null이면 여기서 NPE
}
```

가장 안전한 방법은 받는 즉시 원하는 형태로 명시하는 것이다: `val names: List<String> = shelter.allNames()`. 이렇게 하면 리스트가 null이거나 원소에 null이 있을 때 문제가 일찍 드러난다(원소 검사는 접근 시점에 인트린식으로 이루어질 수 있다).

```text
Java 반환 타입          →   Kotlin 플랫폼 표기        →   권장 명시
─────────────────────────────────────────────────────────────
String                  →   String!                   →   String 또는 String?
List<String>            →   (Mutable)List<String!>!    →   List<String>
Map<String,Integer>     →   (Mutable)Map<String!,Int!>!→   Map<String, Int>
int[]                   →   IntArray!                  →   IntArray
String[]                →   Array<(out) String!>!      →   Array<String>
```

---

## 3. 널 애노테이션 존중: 경계에서 널 가능성 검사하기

### 3.1 애노테이션이 플랫폼 타입을 없앤다

플랫폼 타입은 정보가 없어서 생긴다. 그렇다면 Java 쪽에서 **정보를 제공**하면 해결될까? 실제로 그렇다. Java 코드에 널 애노테이션(`@Nullable`/`@NotNull`)이 붙어 있으면, Kotlin 컴파일러는 이를 읽고 플랫폼 타입 대신 **정확한 널 가능 타입이나 비-널 타입**을 부여한다.

```java
// Java (org.jetbrains.annotations 사용)
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

public class Zoo {
    @NotNull  public String officialName() { return "Seoul Zoo"; }
    @Nullable public String nickname()     { return null; }
}
```

```kotlin
val zoo = Zoo()
val on: String  = zoo.officialName()  // @NotNull → String (비-널) 그대로 대입 OK
val nn: String? = zoo.nickname()      // @Nullable → String? 강제

zoo.officialName().length             // OK — 비-널 보장
zoo.nickname().length                 // 컴파일 에러: only safe (?.) or non-null (!!) calls allowed
```

`@NotNull`이 붙으면 플랫폼 타입이 아니라 진짜 `String`이 되므로 널 검사가 필요 없다. `@Nullable`이 붙으면 진짜 `String?`이 되므로 컴파일러가 널 검사를 강제한다. 경계에 검문소가 생긴 셈이다.

### 3.2 컴파일러가 존중하는 애노테이션 체계

Kotlin은 특정 벤더에 묶이지 않고, 널리 쓰이는 여러 널 애노테이션 체계를 인식한다. 대표적인 체계는 다음과 같다.

| 체계 | 패키지(예) | 비고 |
|---|---|---|
| JetBrains | `org.jetbrains.annotations` | `@Nullable`/`@NotNull` |
| Android | `androidx.annotation`, `android.support.annotation` | `@Nullable`/`@NonNull` |
| JSR-305 | `javax.annotation` | `@Nonnull`, `@CheckForNull` 등 |
| FindBugs/SpotBugs | `edu.umd.cs.findbugs.annotations` | |
| Eclipse JDT | `org.eclipse.jdt.annotation` | |
| Checker Framework | `org.checkerframework.checker.nullness.qual` | |
| Lombok | `lombok.NonNull` | |
| **JSpecify** | `org.jspecify.annotations` | 업계 표준화 시도, 최신 Kotlin에서 지원 강화 |

> [!info] 역사 메모
> JVM 생태계에는 오랫동안 널 애노테이션이 **난립**하는 문제가 있었다. 라이브러리마다 서로 다른 `@Nullable`이 있어서 도구가 이를 통합해 처리하기 어려웠다. **JSpecify**는 Google, JetBrains, Uber 등이 참여해 이런 파편화를 끝내려는 표준화 프로젝트다.
>
> Kotlin은 JSpecify의 `@NullMarked`(패키지나 클래스 단위로 "기본값은 비-널"이라고 선언)와 `@Nullable`을 존중하며, 최신 버전(1.9+ 및 2.x)에서 지원이 성숙했다. 다만 JSpecify를 얼마나 존중하는지는 컴파일러 버전과 옵션에 따라 다를 수 있으므로 프로젝트에서 확인하는 것이 좋다.

### 3.3 `@NotNull` 위반을 런타임에 잡는 방법

Java 메서드가 `@NotNull`이라고 약속해 놓고 실제로 null을 반환하면 어떻게 될까? Kotlin은 그 값을 비-널 `String`으로 다루므로, 경계에 **인트린식 널 검사**를 넣어 계약 위반을 일찍 드러낸다.

```kotlin
// Java officialName()이 계약을 어기고 null을 반환한다면
val on: String = zoo.officialName()
// 바이트코드에 삽입된 개념:
//   String tmp = zoo.officialName();
//   Intrinsics.checkNotNullExpressionValue(tmp, "zoo.officialName()");
//   → tmp == null 이면 즉시 NPE (메시지에 표현식 이름 포함)
```

즉 애노테이션이 있으면 없을 때(플랫폼 타입)보다 **더 안전할 뿐 아니라 진단하기도 쉽다**. 반대로 Java가 Kotlin의 비-널 파라미터에 null을 넣어 호출하는 경우에도, 컴파일러가 함수 시작 부분에 `Intrinsics.checkNotNullParameter`를 넣어 방어한다(5절, 9절과 연결된다).

> [!note] 명세 기준
> 이 인트린식 검사는 "널 가능성 계약을 지켜야 하는 경계"에만 들어간다. 순수 Kotlin에서 Kotlin을 호출할 때는 타입 시스템이 이미 비-널을 보장하므로 대체로 필요하지 않다(단, public API 경계의 파라미터 검사는 리플렉션이나 Java 호출에 대비해 남아 있을 수 있다). 이 검사를 넣을지는 `-Xno-param-assertions`, `-Xno-call-assertions`, `-Xno-receiver-assertions` 플래그로 제어한다.

### 3.4 애노테이션의 한계와 `!!`를 쓰는 자리

애노테이션이 있다고 모든 문제가 해결되지는 않는다. Java 라이브러리 전체에 애노테이션이 붙어 있는 경우는 드물고, 제네릭 타입 인자의 널 가능성까지 정밀하게 표기하기는 어렵다. 그래서 상호운용 경계에서는 결국 개발자가 판단해야 하며, "이 값은 절대 null이 아니다"라고 확신할 때 `!!`(비-널 단언 연산자)를 쓰게 된다.

하지만 6장에서 경고했듯이 `!!`는 "null이면 NPE를 던지겠다고 명시적으로 포기하는 것"이다. 상호운용 코드에 `!!`가 많다면, 대개는 Java 쪽에 애노테이션을 붙이거나 경계에서 명시적인 타입으로 확정하는 편이 낫다.

---

## 4. Java 멤버가 Kotlin에서 보이는 모습

### 4.1 getter/setter가 프로퍼티로 합성된다

Java에는 프로퍼티가 없고, 대신 JavaBeans 관례(`getX()`/`setX()`/`isX()`)를 쓴다. Kotlin은 이 관례를 인식해 **합성 프로퍼티(synthetic property)** 로 노출한다. `getName()`/`setName(...)` 쌍이 있으면 `var name`처럼 보이고, `getName()`만 있으면 `val name`처럼 보인다.

```java
// Java
public class Animal {
    private String name;
    private boolean hungry;
    public String getName() { return name; }
    public void setName(String n) { this.name = n; }
    public boolean isHungry() { return hungry; }
    public void setHungry(boolean h) { this.hungry = h; }
    public int computeAge() { return 3; }  // get 접두사 없음 → 프로퍼티 아님
}
```

```kotlin
val a = Animal()
a.name = "Rex"          // setName("Rex") 호출
println(a.name)         // getName() 호출
a.isHungry = true       // setHungry(true) 호출 — is 게터의 프로퍼티 이름은 isHungry (하단 정밀 메모)
println(a.isHungry)     // isHungry() 게터 호출
println(a.computeAge()) // 접두사 없으므로 메서드 그대로
```

> [!note] 명세 기준
> `boolean` 게터의 프로퍼티 이름은 접두사에 따라 정해진다. `isHungry()`는 프로퍼티 이름이 `isHungry`가 되고(접두사 `is`를 떼지 않는다), `getHungry()`라면 프로퍼티 이름이 `hungry`가 된다. 세터는 `setHungry`처럼 게터의 프로퍼티 이름에 맞춰 짝으로 인식된다. 합성 프로퍼티는 **Java 멤버에만 적용되는 관례**이며, Kotlin이 만든 프로퍼티는 처음부터 진짜 프로퍼티다([[22 - 프로퍼티와 백킹 필드]]).

주의할 점이 있다. 합성 프로퍼티 문법(`a.name = ...`)을 쓸지 메서드 호출(`a.setName(...)`)을 쓸지는 취향이 아니라 **가독성을 일관되게 유지하는** 문제다. 또 Java 필드가 `public`으로 노출되어 있으면 Kotlin에서 필드에 **직접** 접근할 수도 있다. 게터/세터와 필드가 동시에 있으면 어느 쪽이 보이는지 헷갈릴 수 있다(대개 접근자가 우선한다).

### 4.2 배열: `int[]`는 `IntArray`, `String[]`은 `Array<String!>`

Java 배열과 Kotlin 배열의 매핑은 원시형인지에 따라 달라진다([[08 - 수 2 - 박싱과 오버플로와 비트와 부호 없는 정수]]).

| Java | Kotlin |
|---|---|
| `int[]` | `IntArray` |
| `long[]` | `LongArray` |
| `double[]` | `DoubleArray` |
| `boolean[]` | `BooleanArray` |
| `char[]` | `CharArray` |
| `String[]` | `Array<(out) String!>` |
| `T[]` (참조형) | `Array<(out) T!>` |

원시형 배열은 박싱 없는 전용 타입(`IntArray` 등)으로, 참조형 배열은 제네릭 `Array<T>`로 매핑된다. 여기서 **`Array<T>`는 `List<T>`가 아니다**. 둘은 별개의 타입이며 계약도 다르다. 자세한 내용은 38장(컬렉션)에서 설명한다.

`(out)` 프로젝션에 주목하자. Java 배열은 공변(covariant)이므로, `String[]`을 `Object[]`로 취급해 원소를 넣다가 런타임에 `ArrayStoreException`이 발생할 수 있다. Kotlin은 Java에서 온 배열 타입에 사용 지점 `out` 프로젝션을 씌워([[33 - 제네릭 1 - 타입 파라미터와 변성]]) 이 위험을 부분적으로 막는다.

```kotlin
// Java: public void feed(int... amounts)  (가변 인자)
fun useVararg(z: Feeder) {
    z.feed(1, 2, 3)              // 그냥 넘김
    val arr = intArrayOf(4, 5, 6)
    z.feed(*arr)                 // 스프레드 연산자 * 로 배열 펼치기
}
```

Java의 가변 인자(`int...`)는 Kotlin의 `vararg`처럼 호출할 수 있고, 이미 배열이 있으면 스프레드 연산자 `*`로 펼쳐서 넘긴다([[12 - 함수 - 인자와 vararg와 지역 함수와 꼬리 재귀]]).

### 4.3 SAM 변환: Java 함수형 인터페이스에 람다 넘기기

Java에는 단일 추상 메서드(SAM, Single Abstract Method) 인터페이스가 매우 많다(`Runnable`, `Comparator`, `Callable` 등). **Java의 SAM 인터페이스**를 기대하는 자리에 람다를 넘기면, Kotlin이 자동으로 인스턴스로 변환해 준다([[13 - 함수 타입과 람다와 함수 참조]]).

```kotlin
// Java: void schedule(Runnable r)
executor.schedule { println("실행됨") }   // 람다 → Runnable 인스턴스로 SAM 변환

// Java: void sort(List<T> l, Comparator<T> c)
Collections.sort(list, compareBy { it.length })

// 명시적 SAM 생성자
val r = Runnable { println("hi") }         // SAM 생성자
```

> [!warning] 흔한 오해
> "Kotlin 인터페이스에도 람다를 넘길 수 있다"고 생각하기 쉽지만, 역사적으로 SAM 변환은 **Java 인터페이스**에만 적용되었다. Kotlin에서 정의한 인터페이스에 람다를 쓰려면 `fun interface`(함수형 인터페이스, [[25 - 인터페이스]])로 명시적으로 선언해야 SAM 변환을 쓸 수 있다. Kotlin에는 이미 함수 타입(`(A) -> B`)이라는 일급 개념이 있으므로, 인터페이스의 SAM 변환을 무분별하게 허용하지 않으려고 이렇게 설계했다.

### 4.4 Java의 정적 멤버와 중첩 클래스 접근

Java의 `static` 멤버는 Kotlin에서 **클래스 이름을 통해** 그대로 접근한다.

```kotlin
Integer.parseInt("42")        // java.lang.Integer의 static 메서드
val max = Integer.MAX_VALUE   // static 필드
Math.max(1, 2)                // static 메서드

// 중첩 클래스
val entry: Map.Entry<String, Int> = ...
```

Java의 `Object` 멤버(`getClass()`, `wait()`, `notify()` 등)는 Kotlin에서 조금 다르게 노출된다. 예를 들어 `getClass()`는 Kotlin에서 `.javaClass` 확장으로 접근하고, `.class` 리터럴 대신 `::class`(KClass)를 쓴다(44장 애노테이션과 리플렉션).

`wait`/`notify`/`notifyAll`은 `Any`에 없으므로, 필요하면 `(obj as Object)`로 명시적으로 캐스팅해 접근해야 한다(권장하지 않는다).

---

## 5. Kotlin 선언을 Java에서 쓰기: `@Jvm*` 애노테이션

이제 방향을 바꿔 보자. Java에서 Kotlin 코드를 호출할 때, Kotlin 고유의 개념(최상위 함수, companion, 기본 인자, 프로퍼티)은 Java에 그대로 보이지 않는다. `@Jvm*` 애노테이션은 이 차이를 메우는 도구다.

### 5.1 최상위 선언과 파일 파사드 클래스

Java에는 클래스 밖에 있는 함수나 프로퍼티가 없다. 그래서 Kotlin의 최상위 함수([[04 - 파일과 패키지와 선언 - 프로그램의 골격]])는 컴파일할 때 **파일 이름에 `Kt` 접미사를 붙인** 클래스의 `static` 메서드가 된다.

```kotlin
// 파일: MathUtils.kt
package zoo

fun clampAge(x: Int): Int = x.coerceIn(0, 200)
const val MAX_AGE = 200
```

```java
// Java에서:
int a = MathUtilsKt.clampAge(30);   // 파일명 MathUtils.kt → MathUtilsKt 클래스
int m = MathUtilsKt.MAX_AGE;        // const val → public static final
```

`MathUtilsKt`라는 이름이 마음에 들지 않으면 `@JvmName`으로 클래스 이름을 바꾼다.

```kotlin
@file:JvmName("Ages")   // 파일 최상단, package 위
package zoo
fun clampAge(x: Int): Int = ...
```

```java
int a = Ages.clampAge(30);   // 이제 Ages
```

여러 파일의 최상위 선언을 **하나의 Java 파사드 클래스**로 합치려면 각 파일에 `@file:JvmName("Ages")`와 `@file:JvmMultifileClass`를 함께 붙인다. 그러면 `Ages.class` 하나에 여러 `.kt` 파일의 함수가 모인다.

### 5.2 `@JvmStatic`: companion과 object 멤버를 진짜 static으로

`companion object`의 멤버는 기본적으로 **동반 객체 인스턴스**의 멤버이며, Java의 `static`이 아니다([[28 - object와 companion - 싱글턴과 동반 객체]]). 그래서 Java에서는 `Companion`을 거쳐야 한다.

```kotlin
class Circle(val r: Double) {
    companion object {
        fun unit(): Circle = Circle(1.0)
        @JvmStatic fun zero(): Circle = Circle(0.0)
    }
}
```

```java
Circle u = Circle.Companion.unit();  // @JvmStatic 없으면 Companion 경유
Circle z = Circle.zero();            // @JvmStatic 붙이면 진짜 static처럼
```

`@JvmStatic`을 붙이면 컴파일러가 바깥 클래스(`Circle`)에 **진짜 정적 메서드**를 추가로 만들어, Java에서 `Circle.zero()`로 호출할 수 있게 한다. `object` 선언(싱글턴)의 멤버에도 똑같이 쓸 수 있다. `@JvmStatic`이 없으면 `MyObject.INSTANCE.foo()`로 호출해야 한다.

### 5.3 `@JvmField`: 프로퍼티를 필드로 노출

Kotlin 프로퍼티는 Java에서 getter/setter로 보인다. 단순한 데이터라서 접근자 없이 **필드로 직접** 노출하고 싶다면 `@JvmField`를 쓴다.

```kotlin
class Point(@JvmField val x: Int, @JvmField val y: Int)
```

```java
Point p = new Point(1, 2);
int sum = p.x + p.y;   // getX()/getY() 대신 필드 직접 접근
```

`@JvmField`에는 제약이 있다. `open`, `override`, `const`, `lateinit`, 위임 프로퍼티에는 쓸 수 없고, 인터페이스 프로퍼티에도 쓸 수 없다.

`companion object`의 프로퍼티에 `@JvmField`를 붙이면 바깥 클래스의 static 필드로 노출된다(Java에서 상수를 `MyClass.CONST`로 쓰고 싶을 때 유용하다). 참고로 `const val`은 처음부터 `public static final` 필드로 컴파일되므로 별도의 애노테이션이 필요 없다([[11 - 변수와 초기화 - val과 const와 lateinit]]).

| 선언 | Java에서 보이는 형태 |
|---|---|
| `val x: Int` (클래스 프로퍼티) | `getX()` |
| `var x: Int` | `getX()` / `setX(int)` |
| `@JvmField val x: Int` | `public final int x;` |
| `const val X = 1` (top-level/companion) | `public static final int X = 1;` (인라인됨) |
| companion `val x` (기본) | `Companion.getX()` |
| companion `@JvmStatic val x` | `getX()` static |
| companion `@JvmField val x` | static 필드 |

### 5.4 `@JvmOverloads`: 기본 인자를 Java 오버로드로

Java에는 기본 인자가 없다. Kotlin 함수의 기본값은 Java에서 보이지 않으므로, Java에서는 **모든 인자를 넘겨야** 한다. `@JvmOverloads`를 붙이면 컴파일러가 기본 인자 조합에 맞는 **오버로드 메서드**를 만든다.

```kotlin
class Greeter {
    @JvmOverloads
    fun greet(name: String, greeting: String = "Hello", loud: Boolean = false): String =
        (if (loud) "$greeting, $name!".uppercase() else "$greeting, $name")
}
```

```java
Greeter g = new Greeter();
g.greet("Rex");                      // greet(String)
g.greet("Rex", "Hi");                // greet(String, String)
g.greet("Rex", "Hi", true);          // greet(String, String, boolean)
```

`@JvmOverloads`가 없으면 인자 3개짜리 버전만 존재하므로 Java에서 `greet("Rex")`를 호출할 수 없다. 생성자에도 붙일 수 있다.

> [!caution] 성능 주의
> `@JvmOverloads`는 오버로드를 만들어 메서드 수를 늘린다. 기본 인자가 많으면 조합이 늘어날 수 있다(정확히는 인자 개수에 비례하는 선형 개수의 오버로드가 뒤에서부터 생성된다). Java에서 절대 호출하지 않을 함수라면 붙이지 않는 편이 좋다. 순수 Kotlin에서는 기본 인자가 바이트코드에 `default` 마스크 방식으로 담기므로 오버로드가 폭발적으로 늘어나지 않는다.

### 5.5 `@JvmName`: 시그니처 충돌 해결과 이름 제어

`@JvmName`은 함수나 프로퍼티 접근자의 **JVM상 이름**을 바꾼다. 대표적인 용도는 **타입 소거 때문에 생기는 시그니처 충돌**을 해결하는 것이다([[34 - 제네릭 2 - 타입 소거와 reified와 바운드]]).

```kotlin
// 둘 다 소거 후 filterValid(List)로 같은 시그니처 → 충돌
@JvmName("filterValidInts")
fun List<Int>.filterValid(): List<Int> = filter { it > 0 }

@JvmName("filterValidStrings")
fun List<String>.filterValid(): List<String> = filter { it.isNotEmpty() }
```

두 확장 함수는 각각 `List<Int>`와 `List<String>`을 받지만, 제네릭은 런타임에 소거되므로 JVM 시그니처가 모두 `filterValid(List)`가 되어 **충돌로 컴파일 에러**가 난다. `@JvmName`으로 서로 다른 JVM 이름을 주면 해결된다(Kotlin 소스에서는 여전히 `filterValid`로 호출한다).

프로퍼티 접근자의 이름만 바꾸려면 사용 지점 타깃과 함께 쓴다: `@get:JvmName("getSize")`(44장에서 다룬 사용 지점 타깃).

---

## 6. 이름 맹글링과 가시성: `internal`의 정체

### 6.1 JVM에는 `internal`이라는 접근 등급이 없다

Kotlin의 `internal`은 **모듈(module) 단위 가시성**이다. 같은 컴파일 모듈 안에서는 보이고, 모듈 밖에서는 보이지 않는다(26장 가시성 한정자). 그런데 JVM 바이트코드의 접근 제어자는 `public`/`protected`/`package-private`/`private`뿐이고, "모듈"이라는 개념 자체가 없다.

Kotlin은 이 차이를 어떻게 메울까? `internal` 멤버를 바이트코드에서 **`public`으로 컴파일하되 이름을 맹글링(mangling)해서**, Java에서 실수로 호출하기 어렵게 만든다.

```kotlin
// 파일: bank.kt (모듈 이름이 예: "app")
class Account {
    internal fun applyInterest(rate: Double) { /* ... */ }
}
```

이 `applyInterest`는 바이트코드에서 대략 `applyInterest$app(double)` 같은 **맹글링된 이름의 public 메서드**가 된다(접미사는 모듈 이름을 바탕으로 만든다). Java에서 `account.applyInterest(0.05)`라고 쓰면 그런 이름의 메서드가 없으므로 컴파일되지 않는다. `account.applyInterest$app(0.05)`처럼 맹글링된 이름을 알아야만 호출할 수 있다.

### 6.2 `internal`은 package-private가 아니다

> [!warning] 흔한 오해
> "Kotlin의 `internal`은 Java의 package-private(패키지 전용)와 같다"는 생각은 **틀렸다.** package-private는 **같은 패키지**를 경계로 삼지만, `internal`은 **같은 모듈**(하나의 컴파일 단위: Gradle 소스셋, Maven 모듈, IntelliJ 모듈 등)을 경계로 삼는다. 패키지가 달라도 같은 모듈이면 `internal`이 보이고, 같은 패키지라도 모듈이 다르면 보이지 않는다. 둘은 서로 다른 기준이다.

`internal`의 개념은 26장에서 설명했지만, `internal`이 **왜 바이트코드에서 public과 맹글링으로 나타나는지**는 이 장에서 다루는 상호운용 문제다. 모듈 경계는 컴파일러가 아는 정보이고 JVM은 모르는 정보다. 그래서 Kotlin 컴파일러는 (a) 모듈 안에서는 정상적인 이름으로 해석하고, (b) 모듈 밖에서는 맹글링된 이름 때문에 사실상 접근할 수 없게 만든다.

이 방식이 완벽하게 숨겨 주지는 않는다. **맹글링은 강제 수단이 아니라 억지력**이다. Java에서 맹글링된 이름을 알고 호출하면 접근할 수 있다.

```text
Kotlin 가시성    →  JVM 접근자        →  Java에서의 실질
────────────────────────────────────────────────────────
public           →  public            →  자유롭게 접근
internal         →  public (맹글링)   →  이름 몰라 사실상 불가 (억지력)
protected        →  protected         →  하위 클래스만 (단, 규칙 차이 아래)
private          →  private*          →  접근 불가
```

> [!note] 명세 기준
> `private` 최상위 선언과 최상위 멤버 등은 파일 클래스의 `private`로, 클래스 멤버 `private`는 클래스의 `private`로 컴파일된다. 다만 같은 클래스 안의 `private` 멤버에 컴패니언이나 이너 클래스에서 접근하는 경우 등에는 컴파일러가 접근자(synthetic accessor)를 만들기도 한다.
>
> 또한 `internal` **클래스** 자체는 멤버와 달리 맹글링되지 않고 public으로 노출되므로, Java에서 `internal` 클래스를 참조하는 것은 막을 수 없다. 은닉은 주로 멤버 수준에서 작동하는 억지력이다.

### 6.3 `protected`의 미묘한 차이

Kotlin의 `protected`는 "**선언한 클래스와 그 하위 클래스**"에서만 보인다. Java의 `protected`는 여기에 더해 "**같은 패키지**"에서도 보인다. 두 언어의 `protected`는 의미가 완전히 같지 않으므로 상호운용할 때 주의해야 한다. Java에서 Kotlin의 `protected` 멤버에 접근할 때, 그 클래스를 상속하지 않고 단지 같은 패키지에 있을 뿐인 클래스에서 접근하면 예상과 다르게 동작할 수 있다.

### 6.4 세터 가시성과 `@JvmName`으로 우회하기

`internal` 멤버를 Java에서 명시적으로 지정한 이름으로 노출하고 싶다면, `@JvmName`으로 맹글링을 무력화할 수도 있다. 하지만 이는 숨기려던 의도를 스스로 깨는 것이므로, "테스트에서만 접근" 같은 특수한 상황이 아니면 권하지 않는다. 세터만 `private`로 좁힌 프로퍼티(`var x: Int = 0; private set`)는 Java에서 세터가 `private`가 되므로 접근할 수 없다.

---

## 7. 예외: 검사 예외가 없는 Kotlin과 `@Throws`

### 7.1 Kotlin의 모든 예외는 비검사(unchecked) 예외다

Java에는 **검사 예외(checked exception)** 가 있다. `throws IOException`을 선언한 메서드를 호출하면, 호출하는 쪽에서 `catch`로 처리하거나 다시 `throws`로 선언해야 컴파일된다. Kotlin은 이 개념을 **완전히 없앴다**. Kotlin의 모든 예외는 비검사 예외이며, `throws` 선언을 강제하지 않는다(41장 예외와 Nothing).

이 설계는 순수 Kotlin에서는 편리하지만, **Java에서 Kotlin을 호출할 때** 미묘한 문제를 만든다.

### 7.2 함정: Java에서 Kotlin 예외를 `catch`하려다 컴파일 실패

```kotlin
// Kotlin
fun readConfig(path: String): String {
    if (path.isEmpty()) throw java.io.IOException("빈 경로")
    return "..."
}
```

```java
// Java — 이 코드는 컴파일 에러!
try {
    String c = ConfigKt.readConfig("");
} catch (IOException e) {   // 컴파일 에러: exception IOException is never thrown in body of ...
    // ...
}
```

Java 컴파일러는 바이트코드에 "`readConfig`가 `IOException`을 던진다"는 선언(`throws`)이 없다고 본다. 그래서 `catch (IOException)`을 "절대 도달할 수 없는 검사 예외 처리"로 간주하고 **컴파일을 거부**한다. Kotlin은 실제로 그 예외를 던지는데도, 시그니처에 `throws`가 없으므로 Java는 그 사실을 알 수 없다.

### 7.3 `@Throws`가 `throws` 절을 되살린다

해결책은 `@Throws`다. 이 애노테이션은 컴파일된 메서드 시그니처에 **`throws` 절을 추가**해서, Java가 그 검사 예외를 인식하게 한다.

```kotlin
@Throws(java.io.IOException::class)
fun readConfig(path: String): String {
    if (path.isEmpty()) throw java.io.IOException("빈 경로")
    return "..."
}
```

이제 Java의 `catch (IOException e)`가 컴파일된다. 여기서 중요한 것은 **효과가 미치는 방향**이다. `@Throws`는 Kotlin 쪽에 예외 처리를 강제하지 않는다(Kotlin에는 여전히 검사 예외가 없다). `@Throws`는 **Java 상호운용을 위해 바이트코드 시그니처에 덧붙이는 정보**일 뿐이다.

```text
              Kotlin 관점            Java 관점
─────────────────────────────────────────────────
@Throws 없음   예외 그냥 던짐        catch 시 "never thrown" 컴파일 에러
@Throws 있음   변화 없음(여전히      throws 절이 생겨 catch 가능
              강제 안 함)
```

> [!info] 역사 메모
> 검사 예외를 없앤 것은 Kotlin의 의도적인 결정이다. C#, Scala, Ruby 등도 검사 예외를 채택하지 않았고, Java 커뮤니티 안에서도 검사 예외가 "예외를 삼키는 catch"와 보일러플레이트를 부추긴다는 비판이 오래전부터 있었다. Kotlin은 예외를 **개발자가 설계로 책임질 문제**로 보고, 그 대신 `runCatching`/`Result`나 봉인된 결과 타입([[30 - sealed 클래스와 대수적 데이터 타입]])으로 오류를 값으로 다루는 스타일을 권장한다(41장).

### 7.4 `Nothing` 타입의 Java 표현

`throw`의 타입은 `Nothing`이고([[05 - 타입 시스템의 지형 - Any와 Unit과 Nothing]]), `Nothing`은 값이 없는 바닥 타입이다. Java에는 바닥 타입이 없으므로, **JVM 백엔드에서** `Nothing`을 반환하는 Kotlin 함수는 바이트코드 시그니처에 대개 `java.lang.Void`(대문자로 시작하는 참조형)로 나타난다.

`Void`는 인스턴스를 만들 수 없는 참조형이어서 "값이 없음"을 표현하는 데 쓰인다. 실제로 그 함수는 정상적으로 반환하지 않으므로(항상 예외를 던지거나 무한 루프를 돈다), Java에서 호출해도 반환값을 받을 일은 없다.

```kotlin
fun fail(msg: String): Nothing = throw IllegalStateException(msg)
```

```java
// Java 관점 (개념): public static Void fail(String msg)  — 반환값은 오지 않음
FailKt.fail("nope");  // 예외가 전파됨
```

---

## 8. 타입 매핑의 미묘함: 매핑된 타입, 컬렉션, 와일드카드

### 8.1 매핑된 타입: Kotlin 타입은 Java 타입의 별명일 때가 많다

Kotlin의 여러 내장 타입은 컴파일할 때 대응하는 Java 타입(또는 JVM 원시 타입)으로 **매핑**된다. Kotlin 소스에서는 다른 이름을 쓰지만, 바이트코드에서는 같은 타입인 경우가 많다.

| Kotlin | JVM 표현 |
|---|---|
| `kotlin.Any` | `java.lang.Object` |
| `kotlin.String` | `java.lang.String` |
| `kotlin.CharSequence` | `java.lang.CharSequence` |
| `kotlin.Throwable` | `java.lang.Throwable` |
| `kotlin.Int` | `int` (원시) / `java.lang.Integer` (박싱, 8장 박싱과 오버플로) |
| `kotlin.Int?` | `java.lang.Integer` |
| `kotlin.collections.List<T>` | `java.util.List<T>` |
| `kotlin.collections.MutableList<T>` | `java.util.List<T>` |
| `kotlin.Unit` | `void` (반환 위치) / `kotlin.Unit` (제네릭 인자) |
| `kotlin.Nothing` | `java.lang.Void`(대략) |
| `kotlin.Annotation` | `java.lang.annotation.Annotation` |

`Int`가 `int`로 컴파일될지 `Integer`로 컴파일될지는 널 가능성과 위치에 따라 정해진다. 비-널 `Int`는 가능한 곳에서 원시 `int`로 컴파일되고(7장 수 1), `Int?`는 null을 표현해야 하므로 박싱된 `Integer`로 컴파일된다. 이 매핑 덕분에 Java의 `Integer.valueOf`가 반환한 값을 Kotlin의 `Int?`로 자연스럽게 받을 수 있다.

### 8.2 `Unit`은 `void`로 컴파일되지만 `void`가 아니다

Kotlin의 `Unit`은 값이 하나뿐인 진짜 타입이다(5장 타입 시스템의 지형). 반면 Java의 `void`는 "값이 없음"을 뜻한다. 그래서 매핑은 상황에 따라 달라진다.

- 함수의 **반환 위치**에서 `Unit`은 JVM `void`로 컴파일된다. `fun log(m: String): Unit`은 Java에서 `void log(String m)`으로 보인다.
- 그러나 `Unit`이 **제네릭 타입 인자**로 쓰이면(예: `Function0<Unit>`, `Callable<Unit>`) 실제 `kotlin.Unit` 타입이 필요하고, 유일한 인스턴스인 `Unit.INSTANCE`가 값으로 전달된다.

```kotlin
val supplier: () -> Unit = { println("side effect") }
// JVM: Function0<Unit>. invoke()는 Unit.INSTANCE를 반환 (void가 아님)
```

Java에서 이 함수 타입을 다루면 `kotlin.Unit`을 만나게 된다. `Unit`을 `void`와 같다고 생각하면 제네릭 문맥에서 혼란이 생긴다.

### 8.3 읽기 전용 컬렉션의 보장은 Java에서 사라진다

가장 위험한 상호운용 함정 중 하나다. Kotlin의 `List<T>`(읽기 전용)와 `MutableList<T>`(가변)는 **둘 다 `java.util.List<T>`로 매핑**된다(38장 컬렉션). Java에는 이런 구분이 없기 때문이다.

그 결과, Kotlin이 읽기 전용 `List`를 Java에 넘기면 Java는 그것이 읽기 전용인지 모르고 `add`/`remove`를 **호출할 수 있다**. 컴파일러도 이를 막지 못한다.

```kotlin
// Kotlin
fun animals(): List<String> = listOf("cat", "dog")   // 읽기 전용 반환
```

```java
// Java — 컴파일러가 막지 않는다
List<String> list = ZooKt.animals();
list.add("bird");   // 컴파일 OK. 런타임엔? 기반 구현에 따라 UnsupportedOperationException 가능
```

이 예제에서 읽기 전용 컬렉션의 실체가 드러난다. `listOf`가 만든 것은 "불변(immutable) 컬렉션"이 아니라, **읽기 전용 인터페이스로 노출된 컬렉션**일 뿐이다. Java에서는 인터페이스가 막아 주던 제약이 사라지므로, 기반 구현이 변경을 지원하는지에 따라 변경이 성공하거나 `UnsupportedOperationException`이 발생한다.

읽기 전용과 불변의 차이는 38장에서 설명했지만, **읽기 전용 보장이 언어 경계에서 얼마나 약한지**를 가장 분명하게 보여 주는 곳이 바로 Java 상호운용이다.

> [!caution] 성능 주의
> Java에서 온 컬렉션은 플랫폼 타입이면서 `Mutable`로 취급될 수 있다(`(Mutable)List<String!>!`). Kotlin 코드에서 이를 `List`로 받으면 읽기 전용처럼 보이지만, 실제 인스턴스는 여전히 가변일 수 있으므로 다른 경로로 변경될 위험이 있다. 어느 경계에서 방어적 복사(`.toList()`)가 필요한지 판단해야 한다.

### 8.4 제네릭 변성과 Java 와일드카드

Kotlin의 **선언 지점 변성**(`out`/`in`, 33장 제네릭 1)은 Java에 없다(Java에는 사용 지점 와일드카드 `? extends`/`? super`만 있다). 그래서 Kotlin 컴파일러는 선언 지점 변성을 Java 시그니처에서 **사용 지점 와일드카드**로 바꿔 표현한다.

```kotlin
// Kotlin: 반환 타입에 out 파라미터가 있으면
fun sources(): List<out Number> = ...   // 개념
// Java 시그니처엔 List<? extends Number>가 나타난다
```

이 와일드카드는 Java 호출자에게 불편할 때가 있다. 이를 제어하는 애노테이션은 다음과 같다.

- `@JvmSuppressWildcards`: 와일드카드를 **없애고** `List<Number>`처럼 노출한다.
- `@JvmWildcard`: 특정 자리에 와일드카드를 **강제로** 넣는다.

```kotlin
fun exactList(): List<@JvmSuppressWildcards Number> = ...
// Java: List<Number> (와일드카드 없음)
```

> [!note] 명세 기준
> 어느 자리에 와일드카드를 붙일지는 컴파일러가 변성 규칙과 위치(파라미터인지 반환인지)에 따라 결정한다. 일반적으로 `out`(공변) 타입 파라미터는 반환 위치에서 `? extends`로, `in`(반공변)은 `? super`로 나타난다. 세부 규칙이 복잡하므로, Java에서 Kotlin 제네릭 API를 쓸 때 시그니처가 예상과 다르면 `@JvmSuppressWildcards`/`@JvmWildcard`로 조정한다.

### 8.5 인터페이스의 기본 메서드와 `-Xjvm-default`

Kotlin 인터페이스는 기본 구현을 가질 수 있다(25장 인터페이스). 이 기본 구현이 Java에서 어떻게 보이는지는 컴파일러 옵션에 따라 달라진다.

- 전통적으로 Kotlin은 기본 구현을 별도의 `DefaultImpls` 정적 클래스에 담고, 인터페이스에는 추상 메서드만 두었다. 이 경우 **Java에서 그 인터페이스를 구현**하면 기본 구현을 물려받지 못하므로 직접 구현해야 했다.
- `-Xjvm-default=all`(또는 `all-compatibility`) 옵션을 쓰면 Kotlin이 **진짜 Java `default` 메서드**로 컴파일하므로, Java 구현체가 기본 구현을 물려받는다.

Kotlin 2.x로 오면서 이 기본값과 플래그 이름이 바뀌어 왔다. 그러므로 Java와 인터페이스를 깊이 공유하는 프로젝트라면, 사용하는 컴파일러 버전의 기본 동작을 확인해야 한다(구현과 버전에 따라 다르다).

---

## 9. 함정 모음: 실전 체크리스트

앞 절에서 본 경계를 "문제가 터지는 순간"을 중심으로 다시 정리한다. 이 절은 상호운용 코드를 리뷰할 때 체크리스트로 쓸 수 있다.

### 9.1 플랫폼 타입 NPE: 가장 흔한 원인

Java에서 온 값을 비-널로 다루다가 실제로 null이 오는 경우다(2절). 이 문제를 진단하기 어려운 이유는 NPE가 **원인(Java 호출)에서 멀리 떨어진 사용 지점**에서 발생하기 때문이다.

```kotlin
// 나쁨
val name = javaObj.findName()         // String! — null일 수 있음
saveToDb(name.trim())                 // 여기서 NPE — 원인은 위

// 좋음 — 경계에서 못박고 처리
val name: String = javaObj.findName() ?: return  // 조기 확정
```

**체크**: Java 호출 결과를 변수에 담을 때 항상 명시적인 타입(`: String` 또는 `: String?`)을 붙였는가? 가능하면 Java 쪽에 널 애노테이션을 붙였는가?

### 9.2 배열 공변과 `ArrayStoreException`

Java 배열은 공변이므로 타입 안전성 검사가 런타임까지 미뤄진다(34장 제네릭 2). Kotlin이 `(out)` 프로젝션으로 방어하지만, Java 코드와 배열을 주고받을 때는 여전히 조심해야 한다. `Array<T>`와 `List<T>`를 혼동하지 않도록 주의한다.

### 9.3 읽기 전용 컬렉션이 Java에서 변경됨

8.3절에서 보았듯이 Kotlin의 `List`를 Java에 넘기면 변경될 수 있다. **정말로 불변이어야 한다면** `java.util.Collections.unmodifiableList(...)`로 감싸거나, 방어적으로 복사하거나, 진짜 영속 컬렉션(kotlinx.collections.immutable 같은 별도 라이브러리)을 써야 한다. `listOf`가 불변을 보장한다고 믿어서는 안 된다.

### 9.4 static 접근 실수

companion 멤버에 `@JvmStatic`이 없으면 Java에서 `Companion.`을 거쳐야 하고(5.2절), object 멤버는 `INSTANCE.`를 거쳐야 한다. Java에서 "왜 이 함수가 static이 아니지?"라는 의문이 들면 `@JvmStatic`을 확인한다. 상수는 `const val`이나 `@JvmField`로 노출한다(5.3절).

### 9.5 검사 예외 catch 실패

Java에서 Kotlin 함수의 검사 예외를 `catch`하려는데 "never thrown" 컴파일 에러가 난다면, Kotlin 함수에 `@Throws`가 빠진 것이다(7절). 라이브러리를 Java에 노출한다면, 검사 예외를 던지는 함수에 `@Throws`를 붙이는 것을 잊지 말아야 한다.

### 9.6 시그니처 충돌

제네릭 확장 함수나 오버로드가 타입 소거 때문에 같은 JVM 시그니처가 되면 `@JvmName`으로 분리한다(5.5절). "platform declaration clash" 진단이 나오면 이 문제다.

### 9.7 `internal`을 은닉 수단으로 과신

`internal`은 맹글링으로 **억지력**을 제공할 뿐, 진짜 은닉 수단이 아니다(6절). 보안 경계로 삼아서는 안 된다. Java에서 맹글링된 이름을 알면 접근할 수 있다.

### 9.8 Java가 Kotlin 파라미터에 null을 넘김

Kotlin의 비-널 파라미터 `fun feed(name: String)`에 Java가 null을 넘기면, 함수 시작 부분의 `Intrinsics.checkNotNullParameter` 인트린식이 즉시 NPE를 던진다(3.3절). 이는 방어 장치이지만, Java 호출자 입장에서는 "왜 여기서 NPE가 나지?"라는 의문이 들 수 있다. 널 가능성을 계약으로 문서화해 두는 것이 좋다.

### 9.9 종합 표

| 증상 | 원인 | 대응 | 절 |
|---|---|---|---|
| 널 안전 Kotlin에서 NPE | 플랫폼 타입 방치 | 경계에서 명시 타입으로 확정 | 2 |
| Java에서 함수가 안 보임 | 맹글링된 `internal` | 가시성 조정 | 6 |
| Java에서 companion 함수 안 보임 | `@JvmStatic` 없음 | `@JvmStatic` | 5.2 |
| Java에서 상수 못 씀 | 접근자만 노출 | `const`/`@JvmField` | 5.3 |
| Java 오버로드 없음 | 기본 인자 안 보임 | `@JvmOverloads` | 5.4 |
| catch "never thrown" | `throws` 절 없음 | `@Throws` | 7 |
| platform declaration clash | 소거 후 시그니처 충돌 | `@JvmName` | 5.5 |
| 읽기 전용 List가 변경됨 | 매핑된 `java.util.List` | 방어적 복사 | 8.3 |
| Java에 이상한 와일드카드 | 선언 지점 변성 번역 | `@JvmSuppressWildcards` | 8.4 |
