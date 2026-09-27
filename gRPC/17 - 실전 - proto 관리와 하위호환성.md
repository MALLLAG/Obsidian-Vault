---
title: "17 - 실전 - proto 관리와 하위호환성"
date: 2026-06-26
tags:
  - grpc
  - protobuf
  - backward-compatibility
  - api-versioning
  - buf
  - schema-governance
  - api-design
  - aip
  - migration
  - 학습노트
---

## 17.0 들어가며: proto는 코드가 아니라 "조약(treaty)"이다

지금까지 gRPC를 부품 단위로 나누어 살펴봤다. [[02 - Protocol Buffers 1 - 문법과 타입 시스템]]에서는 메시지를 정의하는 방법을, [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서는 그 메시지가 바이트로 바뀌는 방식을, [[04 - HTTP2 깊이 보기 - 전송 계층]]에서는 그 바이트가 전달되는 방식을 봤다.

부품은 모두 익혔다. 마지막 장인 이 장에서는 **그 부품으로 만든 시스템을 몇 년 동안 운영하면서도 망가뜨리지 않는 방법**을 다룬다.

여기서는 사고방식을 바꿔야 한다. 평소에는 코드를 "내가 고치면 내가 책임지는 것"으로 생각한다. 함수 시그니처를 바꾸면 컴파일러가 호출하는 곳을 모두 찾아 주고, 한 번에 고쳐서 배포하면 끝난다. 그런데 `.proto`는 다르다.

proto는 코드라기보다 **두 당사자 사이의 조약(treaty)에 가깝다.** 한쪽은 서버 팀이고, 다른 쪽은 클라이언트 팀이다. 클라이언트는 외부 회사일 수도 있고, 6개월 전에 빌드되어 사용자 휴대폰에 설치된 앱일 수도 있다. 이 조약에서 가장 까다로운 특성은 다음과 같다.

> **당신은 상대방을 동시에 업데이트할 수 없다.**

모바일 앱은 사용자가 업데이트하지 않으면 1년 전 proto로 계속 호출한다. 마이크로서비스 환경에서도 수십 개의 서비스가 같은 proto에 의존하므로, 이들을 한 트랜잭션처럼 동시에 배포할 수는 없다. 롤링 배포(rolling deployment) 중에는 구버전 파드와 신버전 파드가 **같은 순간에** 트래픽을 받는다. 즉 어떤 proto 변경이든 반드시 "구버전과 신버전이 공존하는 시간"을 거쳐야 한다.

그래서 이 장의 규칙은 모두 다음 질문 하나로 정리된다.

> **"이 proto를 아는 코드(구버전)와, 저 proto를 아는 코드(신버전)가 같은 와이어 위에서 만났을 때, 둘 다 무사한가?"**

이 질문에 "예"라고 답할 수 있게 만드는 것이 하위호환성(backward compatibility)이다. 이를 자동으로 검증하는 것이 거버넌스이고, 변경을 안전하게 진행하는 절차가 마이그레이션 플레이북이다. 하나씩 살펴보자.

---

## 17.1 호환성의 세 방향: backward / forward / full

먼저 용어를 정확히 정하자. "하위호환"이라는 말을 너무 느슨하게 쓰면 사고가 난다.

```text
                  데이터를 "쓴" 쪽
                        │
        ┌───────────────┼───────────────┐
        │               │               │
    구버전 스키마      ...            신버전 스키마
        │                               │
        ▼                               ▼
   데이터를 "읽는" 쪽
```

- **Backward compatibility(하위호환)**: **새 코드가 옛 데이터를 읽을 수 있다.** 신버전 서버가 구버전 클라이언트의 요청을 처리할 수 있다는 뜻이다. 가장 흔히 신경 쓰는 방향이다.
- **Forward compatibility(상위호환)**: **옛 코드가 새 데이터를 읽을 수 있다.** 구버전 클라이언트가 신버전 서버의 응답을 (모르는 필드는 무시하고) 처리할 수 있다는 뜻이다.
- **Full compatibility(완전 호환)**: 위의 두 가지를 모두 만족한다.

Protocol Buffers의 뛰어난 점은 **와이어 포맷이 처음부터 forward와 backward를 함께 잘 지원하도록 설계되었다는 것**이다. 3장(Protocol Buffers 2 인코딩과 와이어 포맷)에서 봤듯이 모든 필드는 앞에 `(필드 번호, 와이어 타입)` 태그를 달고 있고, **디코더는 모르는 태그를 만나면 그 길이만큼 건너뛰도록(skip)** 되어 있기 때문이다. 와이어에는 이름이 실리지 않고 번호만 실린다. 이 설계 결정 하나가 이 장 전체의 토대다.

직관적으로 비유하면, proto 메시지는 **번호표가 붙은 사물함이 늘어선 복도**다. 디코더는 "3번 사물함, 7번 사물함, 12번 사물함"을 차례로 지나가면서 자기가 아는 번호만 열어 보고, 모르는 번호는 그냥 지나친다. 사물함에 이름표(필드 이름)도 붙어 있지만 그것은 사람이 보라고 붙인 것일 뿐이고, 복도를 지나가는 기계는 번호만 본다. 그래서 다음과 같은 결과가 나온다.

- **번호를 바꾸면** 기계가 엉뚱한 사물함을 연다. 재앙이다.
- **이름만 바꾸면** 기계는 신경 쓰지 않는다. 와이어에는 무해하다.
- **새 번호의 사물함을 추가하면** 옛 기계는 그냥 지나친다. 안전하다.
- **사물함을 없애면** 옛 기계가 빈자리를 찾다가 헷갈릴 수 있으므로, "이 번호는 폐쇄됨"이라고 표시(reserved)해 두어야 한다.

이제 이 직관을 실전 규칙으로 구체화하자.

---

## 17.2 실전 호환성 규칙 총정리

아래 규칙들이 바이트 수준에서 왜 그렇게 동작하는지는 3장(Protocol Buffers 2 인코딩과 와이어 포맷)에서 설명했다. 여기서는 *실전에서 어떻게 판단하는지*에 집중한다. 같은 규칙이 구글의 공개 가이드 AIP-180(Backwards compatibility)에도 "무엇이 호환 파괴인가"라는 목록으로 정리되어 있으므로, 팀 표준을 만들 때 함께 참고할 만하다. 먼저 전체를 표로 정리하면 다음과 같다.

| 변경 | 와이어 호환? | 소스 호환? | JSON 호환? | 비고 |
|---|---|---|---|---|
| 필드 추가 (새 번호) | ✅ 안전 | ✅ | ✅ | 가장 권장되는 진화 방식 |
| 필드 삭제 + `reserved` 처리 | ✅ 안전 | ⚠️ 호출처 깨짐 | ✅ | reserved 필수 |
| 필드 번호 변경 | ❌ 파괴 | ❌ | ❌ | 절대 금지 |
| 필드 이름만 변경 (번호 유지) | ✅ 안전 | ❌ 코드 깨짐 | ❌ JSON 키 바뀜 | 와이어만 무해 |
| 타입 변경 (호환군 내) | ✅ 안전(주의) | ⚠️ | ⚠️ | int32↔int64↔uint32 등 제한적 |
| 타입 변경 (호환군 밖) | ❌ 파괴 | ❌ | ❌ | 예: int32→string |
| `optional`→`repeated` 등 라벨 변경 | ⚠️ 케이스별 | ❌ | ⚠️ | 대개 위험 |
| enum 값 추가 | ✅ 안전 | ⚠️ | ✅ | 미지 값 처리 필요 |
| 필드를 `oneof`로 이동 | ❌/⚠️ | ❌ | ⚠️ | 단일 필드 이동도 위험 |
| 메시지/서비스 rename | ✅ 와이어 무해 | ❌ | ⚠️ | 패키지 경로 주의 |
| `required` 추가/제거 (proto2) | ❌ 파괴 | ❌ | - | required 자체가 안티패턴 |

각 항목이 *왜* 그런지 하나씩 살펴보자.

### 17.2.1 필드 번호는 절대 바꾸지 않는다

필드 번호는 와이어에 실제로 인코딩되는 유일한 식별자다. 태그 바이트는 `(field_number << 3) | wire_type` 공식으로 만들어진다. 예를 들어 필드 번호가 2이고 와이어 타입이 0(varint)이면 다음과 같다.

```text
tag = (2 << 3) | 0 = 0b00010000 = 0x10
```

어느 날 누군가 `user_id`의 번호를 2에서 3으로 바꿨다고 하자. 옛 클라이언트는 여전히 태그 `0x10`(번호 2)을 보내는데, 새 서버는 이것을 "내가 모르는 번호 2"로 보고 건너뛴다. 그러면 `user_id`는 비어 있게 되고, `user_id`로 해석되어야 할 데이터는 사라진다. 컴파일 에러도, 예외도 없다. **조용히 데이터가 사라진다.** 타입 시스템이 이런 문제를 잡지 못한다는 점이 proto 사고가 무서운 이유다.

규칙: **한 번 릴리스된 필드 번호의 의미는 영원히 고정한다.** 의미를 바꾸고 싶다면 새 번호로 새 필드를 만든다.

한 가지 더 알아 둘 점이 있다. 1~15번 필드는 태그가 **1바이트**이고, 16번부터는 **2바이트** 이상이다(varint 인코딩 때문이다). 그래서 자주 쓰이고 반복되는 필드일수록 1~15번에 두는 것이 인코딩 효율에 좋다. 진화 규칙은 아니지만, 처음 설계할 때 자리를 잘 잡아 두기 위한 실전 팁이다. 19000~19999는 protobuf 구현이 내부적으로 예약한 범위이므로 쓸 수 없다.

### 17.2.2 필드를 삭제할 때는 반드시 reserved로 예약한다

필드를 지우는 것 자체는 와이어 호환을 깨지 않는다. 옛 클라이언트가 보낸 그 번호를 새 서버가 무시할 뿐이다. 진짜 위험은 **미래에 누군가 그 번호를 재사용하는 것**이다. 6번 필드 `phone_number`(string)를 지웠는데, 1년 뒤 신입 개발자가 6번을 `age`(int32)로 재사용했다고 하자. 옛 proto로 빌드된 클라이언트가 여전히 6번에 전화번호 문자열을 보내면, 새 서버는 이를 int32 age로 디코딩하려다가 쓰레기 값을 얻거나 파싱 에러를 낸다.

그래서 protobuf는 `reserved`라는 묘비를 제공한다. 필드를 지울 때는 **번호와 이름을 모두 예약**한다.

```proto
message User {
  string id = 1;
  string email = 2;
  // string phone_number = 6;  ← 삭제

  reserved 6;                  // 번호 무덤
  reserved "phone_number";     // 이름 무덤 (JSON/텍스트 포맷·코드 재사용 방지)
}
```

이렇게 해 두면 누군가 6번이나 `phone_number`라는 이름을 다시 쓰려고 할 때 **컴파일러가 거부**한다. 묘비는 사람의 선의를 믿지 않고 도구가 규칙을 강제하게 만드는 장치다. `reserved 6, 9 to 11, 20;`처럼 범위로도 예약할 수 있다.

> [!note] 실전 팁
> 필드를 곧바로 삭제하기보다는 먼저 `deprecated = true` 옵션을 달아 사용을 줄이고(아래 17.5 참조), 트래픽이 충분히 빠진 뒤에 삭제하고 reserved로 예약하는 2단계 방식이 안전하다.

### 17.2.3 타입 변경: "호환군(compatibility group)"이라는 좁은 안전지대

타입 변경은 대부분 금지되지만, **와이어 타입이 같은 정수 계열 안에서는 제한적으로 호환**된다. 3장에서 본 와이어 타입 분류가 그대로 적용된다.

varint(와이어 타입 0)로 인코딩되는 타입끼리는 서로 바꿔도 와이어가 깨지지 않는다.

```text
int32, int64, uint32, uint64, bool, enum   ← 전부 varint(타입 0)
```

단, "와이어가 깨지지 않는다"와 "값이 보존된다"는 다른 이야기다. 곳곳에 함정이 있다.

- `int32 ↔ int64`: 와이어는 문제없다. 그런데 `int32`로 디코딩할 때 값이 32비트를 넘으면 잘린다(truncate). 음수를 `int32`로 보내면 항상 10바이트 varint로 인코딩된다는 점도 주의해야 한다.
- `int32 ↔ uint32`: 와이어는 문제없다. 그러나 음수의 비트 패턴이 양수로 재해석되어, `-1`(int32)이 `4294967295`(uint32)로 바뀐다.
- `sint32/sint64`: 이 경우는 **다르다.** ZigZag 인코딩을 쓰기 때문에 같은 varint 타입이어도 `int32`와 `sint32`는 **호환되지 않는다.** 비트 패턴 자체가 다르다.
- `fixed32 ↔ sfixed32`: 둘 다 와이어 타입 5(32비트 고정)이므로 호환된다. `fixed64 ↔ sfixed64`는 타입 1(64비트 고정)이며 역시 호환된다.
- `string ↔ bytes`: 둘 다 와이어 타입 2(length-delimited)이므로 호환되기는 한다. 하지만 string은 UTF-8 검증을 하므로, bytes에 UTF-8이 아닌 데이터가 있으면 string으로 디코딩할 때 실패한다. bytes→string은 위험하고, string→bytes는 비교적 안전하다.
- `int32 → string`: **완전히 파괴된다.** 와이어 타입이 0에서 2로 바뀌므로, 디코더가 길이 프리픽스를 기대하다가 varint를 만나 실패한다.

핵심 규칙은 **"같은 호환군이라도 값의 의미가 보존되는지 따로 검증하라."** 이다. 도구(`buf breaking`)는 와이어 호환군은 검사해 주지만, "비즈니스 로직상 -1이 4294967295가 되어도 괜찮은가"는 사람이 판단해야 한다.

### 17.2.4 enum: 값 추가는 안전하지만 "미지의 값" 처리가 관건이다

enum에 값을 추가해도 와이어 호환은 깨지지 않는다. 와이어에서 enum은 그냥 varint(정수)이기 때문이다. 문제는 **옛 코드가 모르는 enum 값을 받았을 때 어떤 일이 일어나느냐**이다. 이 부분에서 proto2와 proto3는 결정적으로 다르다.

- **proto3 (open enum)**: 모르는 enum 값도 **정수 그대로 보존**된다. 옛 클라이언트가 새 값 `STATUS_ARCHIVED = 5`를 받으면, 자기 enum에 5가 없어도 정수 5를 들고 있다가 그대로 다시 직렬화할 수 있다. 원시 정수를 꺼내는 방법은 언어마다 다르다. Go는 enum 자체가 `int32` 별칭이므로 그냥 정수로 쓰면 된다. Java/Kotlin에서는 미지 값이 `UNRECOGNIZED`(number = -1)로 들어오므로 `getNumber()`를 호출하면 예외가 나고, 대신 `getStatusValue()` 같은 `...Value` 접근자로 원시 정수를 읽어야 한다. switch 문에서는 보통 `UNRECOGNIZED`로 가거나 어느 케이스에도 매칭되지 않는다.
- **proto2 (closed enum)**: 모르는 값은 unknown field로 취급되어 알려진 필드 집합에서 빠진다. 더 엄격하다.

그래서 **모든 enum의 첫 값(0번)은 반드시 `XXX_UNSPECIFIED`로 두는 것**이 정석이다(AIP-126의 권고이기도 하다). 이유는 두 가지다. 첫째, proto3에서는 모든 스칼라의 기본값이 0이므로 "값을 보내지 않음"과 "0을 보냄"을 구분할 수 없다. 0을 의미 있는 상태로 쓰면 "설정되지 않음"을 표현할 수 없다. 둘째, 미래에 추가될 값을 받는 옛 코드가 안전하게 처리할 수 있는 기본 케이스가 있어야 한다.

```proto
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;  // 0은 항상 "미지정". 의미 없는 자리.
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_PAID        = 2;
  ORDER_STATUS_SHIPPED     = 3;
  // 나중에 추가: 옛 클라는 4를 모르지만 정수로 보존
  ORDER_STATUS_CANCELLED   = 4;
}
```

클라이언트 코드에는 **반드시 미지 값을 다루는 default 분기**를 두어야 한다. forward compatibility가 실제 코드에서는 이런 모양이 된다.

```go
switch order.GetStatus() {
case OrderStatus_ORDER_STATUS_PAID:
    handlePaid()
case OrderStatus_ORDER_STATUS_SHIPPED:
    handleShipped()
default:
    // ORDER_STATUS_UNSPECIFIED, 그리고 "내가 모르는 미래의 값" 둘 다 여기로.
    // 절대 panic 하지 말 것. "알 수 없는 상태"로 우아하게 처리.
    log.Warnf("unknown order status: %d", order.GetStatus())
    handleUnknownGracefully()
}
```

> [!warning] 안티패턴
> enum 값을 **삭제하거나 번호를 재배치**하는 것은 피해야 한다. 와이어에서는 정수가 살아남지만 의미가 틀어진다. 삭제하는 대신 `reserved`로 묘비를 세운다(enum도 `reserved`를 지원한다). 또한 enum 이름만 바꾸는 것도 JSON 직렬화에서는 호환을 깬다. JSON은 값의 이름 문자열을 쓸 수 있기 때문이다.

### 17.2.5 oneof와 required: 사고가 가장 잦은 두 지점

**oneof 변경**은 특히 위험하다. oneof는 "이 중 정확히 하나만 설정된다"는 제약을 표현하지만, 와이어에서는 같은 메시지 안의 평범한 필드들일 뿐이다. 그래서 **여러 필드가 동시에 와이어에 실릴 수도 있고**(그러면 마지막 것이 이긴다), 진화 규칙이 미묘하다.

- 기존 **단일 필드를 oneof 안으로 옮기는 것**: 와이어 번호가 같으면 바이트는 호환되지만, 생성 코드의 API가 완전히 바뀐다(접근자가 `oneof case` 기반으로 바뀐다). 소스 호환이 깨진다. 의미도 바뀐다. 이제 그 필드를 설정하면 같은 oneof의 다른 필드가 지워진다.
- **여러 기존 필드를 하나의 oneof로 묶는 것**: 과거에는 두 필드가 모두 설정된 메시지가 와이어에 존재할 수 있었지만, 이제 그런 메시지는 oneof 불변식을 위반한다. 파싱할 때 마지막 것만 남는 식으로 데이터가 손실된다.
- **oneof에 새 케이스 추가**: 비교적 안전하다(새 필드 추가와 비슷하다). 단, 클라이언트의 `switch (case)`가 새 케이스를 default로 처리해야 한다.

규칙: **oneof는 처음 설계할 때 신중하게 정하고, 이후에는 "케이스 추가"만 하라.** 평범한 필드를 oneof로 옮기거나 그 반대로 옮기는 것은 사실상 새 메시지를 만드는 수준의 변경이다.

**required(proto2)** 는 더 단순하다. **쓰지 마라.** proto3는 `required`를 아예 없앴는데, 이 사실 자체가 큰 교훈이다. required 필드는 스키마 진화를 가로막는다. 한 번 required로 만들면 영원히 뺄 수 없다(빼는 순간 옛 메시지를 읽지 못하거나 보내지 못한다). 새 required 필드를 추가하면 옛 클라이언트가 보낸 메시지(그 필드가 없는 메시지)가 "유효하지 않음"이 되어 파싱 단계에서 거부된다. proto2를 다뤄야 한다면 모든 신규 필드는 `optional`로 만든다.

### 17.2.6 rename: 와이어는 무사하지만 코드와 JSON은 깨진다

메시지·서비스·메서드·필드의 **이름**을 바꾸는 것은 와이어(binary) 수준에서는 무해하다. 앞에서 말했듯 와이어에는 번호만 실리기 때문이다. 하지만 다음 문제가 생긴다.

- **생성 코드가 깨진다(source-breaking).** 메시지 이름을 `User`에서 `Account`로 바꾸면 그 타입을 import하던 모든 코드가 컴파일에 실패한다. 이것은 "내 코드만의 문제"가 아니다. 그 생성 코드 라이브러리를 쓰는 모든 다운스트림이 깨진다.
- **JSON 표현이 깨진다(json-breaking).** protobuf의 canonical JSON 매핑은 필드 이름(정확히는 lowerCamelCase로 변환된 JSON 이름)을 키로 쓴다. 필드 이름을 바꾸면 JSON 키가 바뀌어 [[16 - gRPC-Web과 REST 게이트웨이]]에서 다룬 REST/gRPC-Web JSON 경로가 깨진다. (`json_name` 옵션으로 JSON 키를 고정하면 막을 수 있다.)
- **서비스·메서드 이름은 HTTP/2 `:path`에 들어간다.** gRPC 호출은 `POST /package.ServiceName/MethodName` 경로로 라우팅된다([[07 - 채널 스텁 커넥션 생명주기]] 참조). 서비스나 메서드 이름을 바꾸면 옛 클라이언트는 옛 경로로 호출하는데 서버에는 그 경로가 없으므로 `UNIMPLEMENTED`가 난다. **이는 와이어 파괴에 준한다.** 그래서 패키지·서비스·메서드 이름은 사실상 영구 계약으로 취급해야 한다.

정리하면, 필드 이름 변경은 와이어에는 문제가 없지만 코드와 JSON을 깨뜨리고, **서비스·메서드·패키지 이름 변경은 라우팅을 깨뜨린다**. 후자가 훨씬 위험하다.

---

## 17.3 깨짐의 세 종류: wire / source / json breaking

"breaking change"라는 한 단어 안에는 사실 서로 다른 세 가지 재앙이 섞여 있다. 이를 구분하지 못하면 "와이어는 안 깨졌으니 괜찮다"며 배포했다가 클라이언트 빌드가 전부 멈추는 사고가 난다.

```text
┌────────────────────────────────────────────────────────────────┐
│ WIRE-BREAKING (가장 치명적, 조용함)                              │
│  이미 디스크/네트워크에 존재하는 바이트의 해석이 바뀜            │
│  예: 필드 번호 변경, 타입 변경(호환군 밖), required 추가         │
│  증상: 컴파일 통과, 런타임에 데이터 증발/오해석. 탐지 어려움.    │
├────────────────────────────────────────────────────────────────┤
│ SOURCE-BREAKING (시끄러움, 빌드 타임에 잡힘)                     │
│  생성 코드 API가 바뀌어 의존 코드가 컴파일 실패                  │
│  예: 메시지/필드 rename, 필드 oneof로 이동, 필드 삭제            │
│  증상: 다운스트림 빌드 실패. 적어도 배포 전에 발견됨.            │
├────────────────────────────────────────────────────────────────┤
│ JSON-BREAKING (REST/gRPC-Web 경로에서만)                        │
│  canonical JSON 표현이 바뀜                                     │
│  예: 필드 이름 변경(json_name 미고정), enum 값 이름 변경         │
│  증상: REST 게이트웨이/브라우저 클라가 필드를 못 읽음.          │
└────────────────────────────────────────────────────────────────┘
```

우선순위는 분명하다. **wire-breaking이 압도적으로 위험**하다. source-breaking은 적어도 빌드가 멈추면서 문제를 알려 주지만, wire-breaking은 프로덕션에서 데이터가 조용히 사라질 때까지 아무도 알아채지 못한다. 그래서 거버넌스 도구는 wire-breaking을 가장 엄격하게 검출한다.

`buf`는 이 세 등급을 그대로 정책 카테고리로 제공한다.

- `WIRE`: 와이어 호환만 검사한다(가장 느슨하다). 바이너리만 주고받고 코드는 공유하지 않는 극단적인 경우에 쓴다.
- `WIRE_JSON`: 와이어와 JSON 호환을 검사한다. REST 게이트웨이처럼 JSON도 쓰는 경우에 해당한다.
- `FILE`(기본·권장): 위의 두 가지에 생성 코드 안정성(source)까지 검사한다. 같은 빌드에서 생성 코드를 공유하는 대부분의 경우에 해당한다.
- `PACKAGE`: `FILE`과 비슷하지만 파일 이동을 패키지 단위로 허용한다(파일을 옮겨도 같은 패키지 안이면 통과한다).

대부분의 팀은 `FILE`을 쓴다. "와이어와 코드를 둘 다 깨지 않게 하라"가 합리적인 기본값이기 때문이다.

---

## 17.4 breaking change 자동 검출: buf breaking을 CI 게이트로

사람은 실수한다. "필드 번호를 바꾸지 않았는가?"를 PR마다 사람이 눈으로 검사하는 방식은 오래 유지할 수 없다. 그래서 **과거 스키마를 "기준 이미지(baseline)"로 고정해 두고, 새 스키마를 자동으로 비교**하는 도구가 필요하다. 사실상 표준은 `buf breaking`이다. 도구 전반은 [[06 - 코드 생성과 protoc 툴체인]]에서 다뤘으므로, 여기서는 breaking 검사가 *어떻게 동작하는지*에 집중한다.

### 17.4.1 내부에서 일어나는 일

핵심 아이디어는 단순하다. **proto 스키마 자체를 데이터로 직렬화해서 두 시점을 비교한다.** protobuf 컴파일러는 모든 `.proto`를 파싱해 `FileDescriptorSet`이라는 메시지로 표현할 수 있다(이 메시지 역시 `descriptor.proto`에 protobuf로 정의되어 있다). 여기에는 모든 메시지·필드·번호·타입·라벨·enum·서비스가 구조화되어 들어 있다.

```text
과거 .proto ──parse──► FileDescriptorSet (baseline 이미지)
                                  │
                                  ▼
                            [구조적 비교]  ◄── 규칙 셋(FIELD_NO_DELETE,
                                  ▲                FIELD_SAME_TYPE, ...)
                                  │
현재 .proto ──parse──► FileDescriptorSet (현재)
                                  │
                                  ▼
                       위반 목록 (어떤 규칙을 어디서 어겼는지)
```

`buf`는 `buf build`로 현재 proto를 descriptor 이미지(`.binpb`)로 만들 수 있고, breaking 검사는 "기준 이미지와 현재 이미지"를 규칙별로 대조한다. 규칙의 예는 다음과 같다.

- `FIELD_NO_DELETE`: 기준에 있던 필드가 사라졌는데 reserved 처리도 되어 있지 않다.
- `FIELD_SAME_TYPE`: 같은 번호 필드의 타입이 호환되지 않게 바뀌었다.
- `FIELD_SAME_NUMBER`: 같은 이름 필드의 번호가 바뀌었다.
- `ENUM_VALUE_NO_DELETE`, `RPC_NO_DELETE`, `MESSAGE_NO_DELETE` 등

기준 이미지는 보통 **git의 특정 브랜치나 태그**로 정한다. "현재 PR의 proto와 main 브랜치의 proto"를 비교하는 설정이 가장 흔하다.

### 17.4.2 CI 게이트 구성

`buf.yaml`(모듈 설정)과 CI 단계는 대략 다음과 같다.

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
breaking:
  use:
    - FILE          # 와이어 + JSON + 소스 안정성 (권장 기본)
  except:
    - FIELD_SAME_DEFAULT   # 팀 정책상 예외를 둘 수도 있음
lint:
  use:
    - STANDARD
```

```bash
# CI: 현재 브랜치 proto를 main(원격) 기준과 비교
# main에 있는 proto를 baseline으로 삼아 breaking 여부 검사
buf breaking --against "https://github.com/acme/apis.git#branch=main,subdir=proto"

# 또는 로컬에서 직전 태그 대비
buf breaking --against ".git#tag=v1.4.0"

# 빌드된 이미지 파일 대비(릴리스마다 이미지를 아티팩트로 저장해두는 패턴)
buf build -o image.binpb
buf breaking --against image.binpb
```

이 명령이 **0이 아닌 종료 코드를 내면 CI를 실패**시킨다. 이것이 "게이트(gate)"의 의미다. **breaking change를 담은 PR은 사람이 명시적인 승인 절차를 거치지 않는 한 머지될 수 없다.** 이 CI 설정 한 줄이 proto 사고의 8할을 막는다.

### 17.4.3 위반 예시와 수정

**위반 사례 1: 무심코 타입 바꾸기**

```proto
// before (v1.4.0, baseline)
message Product {
  string id = 1;
  int32 price_cents = 2;
}

// after (이번 PR) — "가격이 21억을 넘을 수 있으니 int64로!"
message Product {
  string id = 1;
  int64 price_cents = 2;   // ← 와이어로는 통과하지만...
}
```

흥미롭게도 `int32 → int64`는 같은 varint 호환군이므로 `buf`의 와이어 정책에서는 **통과**한다. 그러나 생성 코드에서 타입이 `int`에서 `long`으로 바뀌므로 **source-breaking**이고, `FILE` 정책에서는 언어별 타입이 바뀌는 것으로 검출된다. "와이어는 괜찮지만 소스는 깨지는" 전형적인 사례다. 정말 64비트가 필요하다면 새 필드를 추가하는 편이 안전하다.

```proto
message Product {
  string id = 1;
  int32 price_cents = 2 [deprecated = true];  // 옛 필드 유지
  int64 price_cents_v2 = 3;                    // 새 필드
}
```

**위반 사례 2: 필드 삭제 후 reserved 누락**

```proto
// after
message User {
  string id = 1;
  // string nickname = 4;  ← 삭제만 하고 reserved 안 함
}
```

`buf breaking`은 `FIELD_NO_DELETE` 위반을 보고한다. 다음과 같이 수정한다.

```proto
message User {
  string id = 1;
  reserved 4;
  reserved "nickname";
}
```

이렇게 하면 "의도적으로 묘비를 세웠다"는 신호가 되어 검사를 통과한다.

> [!note] 핵심
> 도구는 "와이어/소스/JSON 호환"을 기계적으로 검사한다. 하지만 "의미가 보존되는가"(예: int32→uint32로 음수가 양수가 되어도 비즈니스에 문제가 없는가)는 여전히 사람이 리뷰로 판단해야 한다. 도구는 안전망일 뿐 판단을 대신하지 않는다.

---

## 17.5 API 버저닝 전략: 언제 메서드를 더하고, 언제 v2로 가는가

호환성 규칙을 모두 지켜도, 그 자체로 호환되지 않는 변경이 있다. 리소스의 핵심 구조가 바뀌거나, 메서드의 시맨틱이 근본적으로 달라지는 경우다. 이때 버저닝(versioning)이 필요하다.

### 17.5.1 비파괴적 진화가 우선이다: 버전을 올리는 것은 최후의 수단

버저닝의 제1원칙은 역설적이게도 **"가능하면 버전을 올리지 마라"** 이다. 새 메이저 버전을 만들면 운영 비용이 두 배가 된다. 두 버전을 모두 유지보수하고 모니터링해야 하며, 클라이언트의 마이그레이션도 유도해야 한다. 그래서 대부분의 변경은 버전을 올리지 않고 처리해야 하고, 이를 가능하게 하려고 앞의 호환 규칙이 존재한다.

비파괴적으로 할 수 있는 일은 다음과 같다.

- **필드 추가**: 요청이나 응답에 optional 필드를 추가한다. 옛 클라이언트는 무시하고, 새 클라이언트는 활용한다.
- **메서드 추가**: 서비스에 새 RPC를 추가한다. 옛 클라이언트는 호출하지 않을 뿐 영향을 받지 않는다. "버전을 올리지 않고 기능을 추가"하는 핵심 도구다.
- **enum 값 추가**(미지 값 처리가 전제다).

예를 들어 `GetUser`만 있던 서비스에 `BatchGetUsers`를 추가하는 것은 완전히 안전하다. 새 메서드는 새 `:path`를 만들 뿐 기존 경로를 건드리지 않는다.

### 17.5.2 메이저 버전은 패키지 경로에 넣는다

리소스 모델을 다시 설계하거나 필드 의미를 전면적으로 바꾸는 것처럼 정말로 호환이 불가능한 변경이 필요하다면, **패키지 이름에 버전을 넣어 완전히 분리된 새 API를 만든다.** AIP-215가 권하는 방식이기도 하다.

```proto
// v1
package acme.orders.v1;
service OrderService { rpc GetOrder(GetOrderRequest) returns (Order); }

// v2 — 완전히 별개의 패키지, 별개의 서비스
package acme.orders.v2;
service OrderService { rpc GetOrder(GetOrderRequest) returns (Order); }
```

이렇게 하면 다음과 같은 이점이 있다.

- 생성 코드의 타입이 `acme.orders.v1.Order`와 `acme.orders.v2.Order`로 **이름공간이 나뉘어** 충돌하지 않는다.
- HTTP/2 라우팅 경로가 `/acme.orders.v1.OrderService/GetOrder`와 `/acme.orders.v2.OrderService/GetOrder`로 나뉘어 **한 서버가 두 버전을 동시에 서빙**할 수 있다.
- 옛 클라이언트는 v1 경로로, 새 클라이언트는 v2 경로로 각자 호출하므로 공존할 수 있다.

`v1alpha1`, `v1beta1` 같은 안정성 채널 표기도 흔히 쓴다(AIP-185). alpha는 언제든 깨질 수 있고, beta는 비교적 안정적이며, 버전 접미사가 없는 stable은 호환을 보장한다.

AIP-185의 규칙을 정확히 옮기면, **stable 채널은 접미사를 붙이지 않고**(`v1`이지 `v1beta`가 아니다), **alpha·beta 채널은 안정성 접미사 뒤에 증가하는 릴리스 번호를 붙인다**(`v1beta1`, `v1alpha5`). 또한 마이너·패치 버전은 노출하지 않는다(`v1`이지 `v1.2`가 아니다). 호환 가능한 변경은 같은 메이저 버전 안에서 처리하기 때문이다. 외부에 공개하는 API라면 이 안정성 레벨을 명시해 주는 것이 사용자에게 친절하다.

### 17.5.3 deprecation 절차: 갑자기 끄지 않는다

버전이나 필드를 없앨 때는 "공지 → 유예 → 제거" 절차를 밟는다. 갑작스럽게 제거하면 옛 클라이언트가 동작하지 않게 된다.

1. **표시(mark)**: `deprecated = true` 옵션을 단다. 그러면 생성 코드에 deprecation 경고가 생겨 개발자에게 "곧 없어질 항목"임을 알린다.

```proto
message User {
  string full_name = 5 [deprecated = true];  // given_name + family_name으로 대체
  string given_name = 6;
  string family_name = 7;
}

service UserService {
  rpc GetUserLegacy(GetUserLegacyRequest) returns (User) {
    option deprecated = true;
  }
}
```

2. **측정(measure)**: deprecated된 필드와 메서드의 **실제 트래픽을 모니터링**한다([[14 - 관찰성과 디버깅 - Reflection grpcurl]] 참조). 메서드별 호출량과, 어떤 클라이언트(메타데이터의 user-agent나 버전)가 아직 쓰고 있는지를 추적한다. 트래픽이 0에 가까워질 때까지 기다린다.

3. **유도(migrate)**: 남은 클라이언트의 담당자에게 마이그레이션을 직접 요청한다. 외부 API라면 deprecation 정책(예: "deprecated 후 최소 12개월 유지")을 문서화하고 지킨다.

4. **제거(remove)**: 트래픽이 0이 되고 유예 기간이 끝나면 필드는 `reserved`로 예약하고, 메서드는 (필요하면) 제거한다. 메이저 버전 전체를 끌 때도 같은 절차를 따른다.

### 17.5.4 여러 버전을 동시에 운영하는 현실

v1과 v2를 동시에 운영하려면 비용이 든다. 흔한 패턴은 **v2를 단일 원천(source of truth)으로 삼고 v1을 얇은 어댑터(shim)로 구현**하는 것이다.

```text
        클라(옛)              클라(새)
          │ v1                 │ v2
          ▼                    ▼
   ┌──────────────┐    ┌──────────────┐
   │ v1 Service   │    │ v2 Service   │
   │ (adapter)    │───►│ (real impl)  │
   └──────────────┘    └──────────────┘
        v1 요청을 v2로 변환해 위임,
        v2 응답을 v1 형태로 되변환
```

이렇게 하면 비즈니스 로직은 v2 한 곳에서만 유지하고, v1은 형태 변환만 맡는다. 코드 중복과 로직 드리프트를 막을 수 있다. 단, v1과 v2 사이의 변환에서 표현할 수 없는 필드(v2에만 있는 새 개념)를 누락할지 기본값으로 처리할지는 명시적으로 정해야 한다.

---

## 17.6 리소스 지향 API 설계(구글 AIP 기반)

호환성은 "어떻게 깨뜨리지 않을 것인가"의 문제이고, 설계는 "처음부터 어떻게 잘 만들 것인가"의 문제다. 잘 설계된 API는 진화하기도 쉽다. 여기서는 구글이 공개한 API Improvement Proposals(AIP, aip.dev)의 핵심 패턴을 살펴본다. 이 패턴들은 특정 회사의 내부 규칙이 아니라 **공개된 산업 표준에 가까운 가이드**다.

### 17.6.1 리소스 지향: 명사 + 표준 동사

REST가 자원(URL)과 HTTP 메서드(GET/POST/...)로 세상을 모델링하듯, gRPC에서도 **리소스(명사)를 정의하고 표준 메서드(동사)를 붙이는** 방식이 유지보수에 유리하다. RPC를 임의로 마구 만드는 것보다 규칙적이고 예측하기 쉽다.

표준 메서드 5개(AIP-131~135)는 다음과 같다.

```proto
package acme.library.v1;

service LibraryService {
  rpc GetBook(GetBookRequest)       returns (Book);
  rpc ListBooks(ListBooksRequest)   returns (ListBooksResponse);
  rpc CreateBook(CreateBookRequest) returns (Book);
  rpc UpdateBook(UpdateBookRequest) returns (Book);
  rpc DeleteBook(DeleteBookRequest) returns (google.protobuf.Empty);
}

message Book {
  // 리소스 이름: 컬렉션/리소스 계층 경로. 전역 유일.
  string name = 1;          // 예: "shelves/fiction/books/1984"
  string title = 2;
  string author = 3;
  google.protobuf.Timestamp create_time = 4;
  google.protobuf.Timestamp update_time = 5;
}
```

여기서 눈여겨볼 관례는 다음과 같다.

- 리소스의 식별자 필드는 관례적으로 `name`이고, **계층적 리소스 경로**(`shelves/{shelf}/books/{book}`)를 담는다. 이 경로는 REST URL 경로와 자연스럽게 매핑되어 16장(gRPC-Web과 REST 게이트웨이)의 `google.api.http` 애너테이션과 잘 맞는다.
- `create_time`, `update_time`은 `google.protobuf.Timestamp` 타입이다(이런 잘 알려진 타입은 2장(Protocol Buffers 1 문법과 타입 시스템) 참조). 서버가 채우는 출력 전용(output only) 필드다.
- 각 메서드는 **자기 전용 Request 메시지**를 둔다. `GetBook(Book)`이 아니라 `GetBook(GetBookRequest)`이다. 나중에 Get에만 필요한 파라미터(예: `view`, `read_mask`)를 추가할 여지를 남기기 위해서다. 응답 타입으로 리소스를 직접 돌려주는 것은 괜찮지만, 요청에는 반드시 전용 래퍼를 쓴다. **이는 진화 가능성을 위한 가장 중요한 설계 습관 중 하나다.**

### 17.6.2 표준 List 페이지네이션: page_token / page_size

대량의 컬렉션을 한 번에 모두 돌려주면 메모리, 지연, [[15 - 성능과 Flow Control 백프레셔]] 측면에서 재앙이 된다. 표준 페이지네이션(AIP-158)은 **불투명 토큰(opaque token)** 방식을 쓴다.

```proto
message ListBooksRequest {
  string parent = 1;       // 어느 컬렉션? 예: "shelves/fiction"
  int32  page_size = 2;    // 한 페이지 최대 개수(서버가 상한을 둘 수 있음)
  string page_token = 3;   // 이전 응답의 next_page_token. 첫 호출엔 빈 값.
  string filter = 4;       // 17.6.5 참조
  string order_by = 5;     // 17.6.5 참조
}

message ListBooksResponse {
  repeated Book books = 1;
  string next_page_token = 2;   // 다음 페이지 토큰. 마지막 페이지면 빈 문자열.
  int32  total_size = 3;        // (선택) 전체 개수 추정
}
```

왜 offset(`page=3&limit=20`)이 아니라 **불투명 토큰**을 쓰는가? 여기에 설계의 핵심이 있다.

- **offset 페이지네이션은 데이터가 바뀌면 깨진다.** 1페이지를 보는 사이에 누군가 항목을 추가하거나 삭제하면 offset이 밀려서 항목이 중복되거나 누락된다.
- **불투명 토큰은 "다음 페이지의 커서 위치"를 서버가 내부적으로 인코딩**한다(마지막 항목의 키, 정렬 상태, 필터 등). 클라이언트는 이 토큰을 해석하려 해서는 안 되고, 다음 호출에 그대로 돌려줄 뿐이다. 그래서 서버는 페이지네이션 구현(키셋 커서든 다른 방식이든)을 **클라이언트를 깨뜨리지 않고 자유롭게 바꿀 수 있다.** 토큰이 불투명하다는 사실 자체가 진화 가능성을 보장한다.

클라이언트는 다음과 같이 사용한다.

```go
req := &pb.ListBooksRequest{Parent: "shelves/fiction", PageSize: 50}
for {
    resp, err := client.ListBooks(ctx, req)
    if err != nil { return err }
    for _, b := range resp.GetBooks() {
        process(b)
    }
    if resp.GetNextPageToken() == "" {
        break // 마지막 페이지
    }
    req.PageToken = resp.GetNextPageToken() // 커서 전진
}
```

> 토큰에 만료 시간과 서명을 넣어 변조를 막거나, 서버가 재시작해도 유효하도록 stateless하게(키셋 정보를 토큰에 직접 인코딩) 설계하는 것이 견고하다.

### 17.6.3 부분 업데이트와 FieldMask

Update에서 가장 흔한 사고는 **"PUT 시맨틱"으로 인한 의도치 않은 필드 초기화**다. 클라이언트가 `title`만 바꾸려고 `UpdateBook`을 호출하면서 `author`를 채우지 않으면, 단순하게 구현된 서버는 `author`를 빈 문자열로 덮어쓴다. proto3에서는 "보내지 않음"과 "빈 값"을 구분할 수 없으므로 더 위험하다.

해법은 **`google.protobuf.FieldMask`** 이다(AIP-134). "실제로 바꾸려는 필드 경로 목록"을 명시적으로 함께 보낸다.

```proto
import "google/protobuf/field_mask.proto";

message UpdateBookRequest {
  Book book = 1;                            // 변경할 값들이 담긴 리소스
  google.protobuf.FieldMask update_mask = 2; // 실제로 바꿀 필드 경로
}
```

```go
// 클라: title만 바꾼다고 명시
req := &pb.UpdateBookRequest{
    Book: &pb.Book{
        Name:  "shelves/fiction/books/1984",
        Title: "Nineteen Eighty-Four", // 이것만 바꿈
    },
    UpdateMask: &fieldmaskpb.FieldMask{
        Paths: []string{"title"}, // ← 마스크에 없는 필드는 서버가 건드리지 않음
    },
}
```

서버는 마스크를 순회하면서 해당 경로만 적용한다. `update_mask`가 비어 있을 때 "전체 교체"로 볼지 "아무것도 바꾸지 않음"으로 볼지는 정책으로 정해 문서화한다(AIP는 비어 있으면 전체 교체로 보는 관례를 제시한다). 마스크는 중첩 경로(`author.email`)도 표현할 수 있으므로 깊은 부분 업데이트도 가능하다.

같은 FieldMask를 **읽기 쪽 `read_mask`** 로도 쓴다. 큰 리소스에서 클라이언트가 원하는 필드만 받아 대역폭을 아끼는 용도다. FieldMask는 `google.protobuf`의 잘 알려진 타입이므로 모든 언어 런타임이 헬퍼를 제공한다.

### 17.6.4 Long-Running Operations(LRO): 오래 걸리는 작업의 표준

대용량 내보내기나 인덱스 재구축처럼 수 초에서 수 분이 걸리는 작업이 있다. 이런 작업을 동기 unary로 처리하면 클라이언트가 [[10 - Deadline 취소 타임아웃]]에 걸려 실패하거나, 연결을 오래 점유한다. 표준 해법은 **작업을 즉시 "Operation 핸들"로 반환하고, 완료 여부를 폴링/조회**하게 하는 것이다(AIP-151).

```proto
import "google/longrunning/operations.proto";

service ArchiveService {
  // 즉시 Operation을 반환. 실제 작업은 백그라운드.
  rpc ExportBooks(ExportBooksRequest) returns (google.longrunning.Operation);
}

// google.longrunning.Operation 의 골자:
// message Operation {
//   string name = 1;          // 이 작업의 핸들
//   google.protobuf.Any metadata = 2;  // 진행률 등
//   bool done = 3;
//   oneof result {
//     google.rpc.Status error = 4;     // 실패 시
//     google.protobuf.Any response = 5; // 성공 시 결과
//   }
// }
```

클라이언트는 `Operation.name`으로 `GetOperation`을 폴링하거나, 별도의 알림 채널로 완료 통지를 받는다. 결과는 `oneof { error, response }`로 들어오는데, 이는 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에서 본 `google.rpc.Status`를 그대로 재사용한다.

서버 스트리밍([[05 - 통신의 4가지 방식 - Unary와 Streaming]])으로 진행률을 푸시하는 변형도 있다. 하지만 LRO는 "연결이 끊겨도 작업이 계속 진행되고 나중에 다시 조회할 수 있다"는 점에서 오래 걸리는 작업에 더 견고하다.

### 17.6.5 필터·정렬

List에는 `filter`(문자열)와 `order_by`(문자열) 필드를 둔다(AIP-160, AIP-132). 왜 구조화된 메시지가 아니라 문자열일까? **표현력과 진화 때문**이다. 필터 문법(예: `author = "Orwell" AND create_time > "2020-01-01T00:00:00Z"`)을 문자열로 두면, 새 연산자나 필드를 추가해도 proto를 바꿀 필요가 없다.

단, 서버는 이 문자열을 안전하게 파싱해야 한다(인젝션 방지). `order_by`는 `"create_time desc, title"` 같은 형식이다.

### 17.6.6 멱등성과 요청 ID: 안전한 재시도의 토대

[[13 - 안정성 - Retry Health Check Keepalive]]에서 봤듯이 gRPC는 재시도를 한다. 그런데 `CreateBook`을 재시도하면 책이 두 권 생길 수 있다. 네트워크가 응답을 잃었을 뿐 서버는 이미 요청을 처리했을 수도 있기 때문이다. 해법은 **클라이언트가 요청마다 고유 ID를 부여**하고 서버가 그 ID로 중복을 제거하는 것이다(AIP-155).

```proto
message CreateBookRequest {
  string parent = 1;
  Book   book = 2;
  // 클라가 생성한 UUID. 같은 ID의 요청은 한 번만 실행됨이 보장.
  string request_id = 3;
}
```

서버는 `request_id`를 일정 기간 저장해 두고, 같은 ID가 다시 오면 **새로 실행하지 않고 이전 결과를 그대로 반환**한다. 이렇게 하면 Create 같은 비멱등(non-idempotent) 연산도 **재시도에 안전(idempotent하게)** 만들 수 있다. Get/List/Delete는 본래 멱등하지만, Create와 (값이 아니라 증분을 적용하는) Update에서는 request_id가 특히 중요하다.

이 패턴은 [[08 - 메타데이터와 인터셉터]]의 인터셉터에서 request_id를 자동으로 주입하거나, 메타데이터 헤더(`x-idempotency-key`)로 옮겨 횡단 관심사로 처리할 수도 있다.

---

## 17.7 에러 설계: 도메인 에러 → 상태 코드 + Rich Error의 일관 규약

API 설계의 절반은 "성공했을 때"이고 나머지 절반은 "실패했을 때"다. 에러 모델의 동작 방식은 9장(에러 모델 상태 코드와 Rich Error)에서 다뤘으므로, 여기서는 **진화와 거버넌스 관점의 규약**만 정리한다.

핵심은 **"도메인 에러를 표준 상태 코드에 일관되게 매핑하고, 세부 정보는 Rich Error로 구조화한다"** 는 팀 전체의 합의다. 같은 종류의 실패가 서비스마다 다른 코드로 나오면 클라이언트는 대응하기가 매우 어려워진다.

```text
도메인 상황                       → gRPC 상태 코드        → Rich Error detail
─────────────────────────────────────────────────────────────────────────
요청 필드가 형식 위반            → INVALID_ARGUMENT(3)    → BadRequest.FieldViolation
인증 안 됨                      → UNAUTHENTICATED(16)    → (선택) ErrorInfo
권한 없음                       → PERMISSION_DENIED(7)   → ErrorInfo(reason)
리소스 없음                     → NOT_FOUND(5)           → ResourceInfo
이미 존재(중복 생성)            → ALREADY_EXISTS(6)      → ResourceInfo
쿼터 초과                       → RESOURCE_EXHAUSTED(8)  → QuotaFailure
사전조건 위반(상태가 안 맞음)   → FAILED_PRECONDITION(9) → PreconditionFailure
재시도하면 될 일시적 실패       → UNAVAILABLE(14)        → RetryInfo
서버 버그/예상 못한 내부 오류   → INTERNAL(13)           → DebugInfo(로그용)
```

에러 규약에도 다음과 같은 진화 규칙이 적용된다.

- **새 에러 "상황"을 추가하는 것은 괜찮다.** 클라이언트는 모르는 상태 코드나 Rich Error 타입을 만나면 일반적인 실패로 처리할 수 있어야 한다(forward compatibility). default 분기가 필수다.
- **기존 에러의 상태 코드를 바꾸는 것은 사실상 breaking이다.** 클라이언트가 `NOT_FOUND`를 기준으로 분기 로직을 짜 두었는데 어느 날 `FAILED_PRECONDITION`으로 바뀌면 로직이 깨진다. 상태 코드 매핑은 계약의 일부로 취급하라.
- **Rich Error의 `ErrorInfo.reason`은 안정적인 enum 형태의 문자열**(예: `"BOOK_OUT_OF_PRINT"`)로 두고, 사람이 읽는 `message`와 분리한다. 클라이언트는 `reason`으로 분기하고 `message`는 화면 표시에만 쓴다. 그래야 `message` 문구를 바꿔도 클라이언트 로직이 깨지지 않는다.

또 다른 설계 방식으로, **응답 메시지 안에 `oneof { data, error }`를 두어 도메인 에러를 타입으로 표현**하는 방법도 있다. gRPC 네이티브 상태 코드는 결국 "정수 + 문자열"이므로 도메인 에러를 표현하면 stringly-typed가 되기 쉽다. 반면 에러를 proto 메시지로 정의하면 컴파일 타임 안전성(언어별 패턴 매칭)을 얻는다.

다만 이 경우에도 "DB 장애나 네트워크 오류 같은 진짜 내부 오류"는 여전히 gRPC 네이티브 status(INTERNAL, UNAVAILABLE)로 올려 보내고, `oneof error`에는 **호출자가 의미 있게 처리할 수 있는 도메인 에러만** 담는 것이 좋다. 어느 규약을 쓰든, 서비스마다 어떤 규약을 따르는지 명시하는 것이 거버넌스가 할 일이다.

---

## 17.8 proto 저장소 전략: 어디에 두고 누가 책임지는가

proto를 잘 작성하는 것만큼 중요한 것이 **proto를 어디에 두고 어떻게 배포하느냐**다. 크게 세 가지 방식이 있다.

### 17.8.1 세 가지 배치 모델

```text
[A] 서비스 코드 옆 (in-repo / monorepo)
   service-a/
     proto/        ← 이 서비스가 노출하는 proto
     src/

   장점: proto와 구현이 한 PR에서 같이 변함. 응집도↑
   단점: 다른 서비스가 이 proto를 쓰려면 cross-repo 의존. 다언어 공유 까다로움.

[B] 중앙 proto 저장소 (central proto repo)
   apis-repo/
     acme/orders/v1/*.proto
     acme/users/v1/*.proto

   장점: 모든 계약이 한 곳. 일관된 lint/breaking 게이트. 다언어 생성 일원화.
   단점: proto 변경과 구현 변경이 두 PR로 갈림. 동기화 부담.

[C] 스키마 레지스트리 (BSR 등)
   원격 레지스트리에 모듈로 게시. 버전 태깅, 의존성 관리, 생성 코드 배포까지.

   장점: 패키지 매니저처럼 proto를 의존성으로 다룸. breaking 검사·문서·SDK 생성 통합.
   단점: 외부 의존(또는 self-host 운영). 도구 락인 고려.
```

선택 기준은 조직의 규모와 사용하는 언어의 수다. 단일 언어에 서비스가 적다면 [A]가 단순하고 좋다. 여러 팀과 여러 언어가 같은 계약을 공유한다면 [B]나 [C]가 일관성을 준다. BSR(Buf Schema Registry) 같은 레지스트리를 쓰면 proto를 "패키지"처럼 의존성으로 가져올 수 있고(`buf.lock`으로 버전 고정), 게시 시점에 breaking 검사를 강제하며, 다언어 SDK 생성을 중앙에서 관리할 수 있다.

(워크스페이스가 여러 repo로 나뉜 환경에서는 git submodule로 interface proto를 공유하는 변형도 있다. 이 경우 main을 pull한 뒤 submodule update를 빠뜨리면 컴파일이 깨지는 함정이 있으므로 동기화 절차를 자동화해 두는 것이 좋다.)

### 17.8.2 생성 코드 배포: 라이브러리 게시 vs 빌드 시 생성

proto에서 만든 생성 코드를 소비자에게 전달하는 방식도 두 가지로 나뉜다.

| 방식 | 작동 | 장점 | 단점 |
|---|---|---|---|
| **라이브러리 게시(pre-generated)** | proto가 바뀌면 CI가 언어별 패키지(Go module, Maven artifact, npm, PyPI 등)를 빌드해 게시한다. 소비자는 그 패키지를 의존성으로 설치한다. | 소비자 빌드가 빠르고 단순하다. 버전 고정이 명확하다. protoc 툴체인이 필요 없다. | 게시 파이프라인을 유지해야 한다. 버전이 급격히 늘어날 수 있다. |
| **빌드 시 생성(generate-on-build)** | 소비자가 proto를 가져와 자기 빌드에서 `protoc`/`buf generate`를 실행한다. | 항상 최신이다. 중간 아티팩트가 없다. | 모든 소비자가 protoc 플러그인 버전을 맞춰야 한다. 빌드가 느리고 재현성 문제가 생길 수 있다. |

실전에서는 **다언어·다팀 환경일수록 라이브러리 게시가 유리**하다. 특히 외부에 SDK를 제공할 때는 소비자에게 protoc를 강요할 수 없으므로 게시가 사실상 필수다(생성 도구는 6장(코드 생성과 protoc 툴체인)에서 다뤘다). 핵심은 **생성 코드의 버전이 proto 버전과 결정적으로 묶이는 것**이다. 즉 같은 proto에서는 항상 같은 코드가 나와야 한다(재현 가능한 빌드, 플러그인 버전 고정).

### 17.8.3 다언어 동기화

같은 proto에서 Go·Java·Python·TypeScript 코드를 생성할 때는 모든 언어가 같은 시점의 같은 proto를 보게 해야 한다. 흔한 함정은 Go 서버는 새 필드를 알지만 Python 클라이언트 SDK는 아직 옛 버전이라 그 필드를 쓰지 못하는 경우다. 와이어 호환이므로 깨지지는 않지만, 기능이 반영되지 않은 것처럼 보인다.

해결책은 **모든 언어의 SDK를 한 proto 릴리스에 묶어 동시에 게시**하고, 버전 번호를 일치시키는 것이다. 레지스트리 모델([C])이 이 부분을 가장 잘 자동화한다.

### 17.8.4 소유권과 리뷰 프로세스

proto는 계약이므로 **소유자(owner)가 명확해야 한다.** 권장하는 방식은 다음과 같다.

- 각 proto 패키지나 디렉터리에 `CODEOWNERS`를 두어, 변경 시 해당 팀의 리뷰를 강제한다.
- breaking 검사 CI는 반드시 통과해야 하는 게이트로 둔다(17.4).
- **클라이언트 팀도 리뷰에 참여한다.** 계약은 양 당사자의 것이므로 서버 팀이 일방적으로 바꾸지 못하게 한다.
- 새 메서드나 새 리소스 같은 큰 변경은 proto만 따로 설계 리뷰를 거친 뒤 머지한다.

---

## 17.9 스키마 거버넌스: lint, 네이밍, 문서화, 민감 필드

거버넌스는 "수백 개의 proto가 한 사람이 쓴 것처럼 일관되게 보이도록" 만드는 일이다.

### 17.9.1 lint로 스타일 강제

`buf lint`(또는 protolint)는 네이밍과 구조 규칙을 자동으로 검사한다. 표준 규칙 셋이 강제하는 항목은 다음과 같다.

```text
- 패키지는 소문자 + 점, 끝에 버전 (acme.orders.v1)
- 메시지명: PascalCase (CreateBookRequest)
- 필드명: lower_snake_case (page_token)
- enum 값: UPPER_SNAKE_CASE, 0은 _UNSPECIFIED
- 서비스명은 Service로 끝남 (LibraryService)
- RPC 입력/출력은 전용 메시지 (XxxRequest / XxxResponse)
- 한 파일에 한 패키지
```

```yaml
# buf.yaml
lint:
  use:
    - STANDARD
  except:
    - PACKAGE_VERSION_SUFFIX   # 내부 전용이라 버전 안 붙이는 예외 등
  ignore:
    - vendor/                  # 외부에서 가져온 proto는 제외
```

lint도 breaking 검사와 마찬가지로 **CI 게이트**로 건다. 그래야 PR마다 사람이 스타일을 두고 논쟁하지 않는다.

### 17.9.2 문서화 주석

proto의 주석은 단순한 메모가 아니다. 많은 생성기가 **proto 주석을 생성 코드의 doc comment와 API 문서로 전파**한다. 그래서 주석은 "공개 API 문서를 쓴다"는 자세로 작성한다.

```proto
// A Book is a single published work in the library catalog.
//
// Books are immutable once created except for `title` and `tags`.
message Book {
  // The resource name of the book.
  // Format: shelves/{shelf}/books/{book}
  string name = 1;

  // The display title. May be updated via UpdateBook.
  string title = 2;

  // Output only. The time the book was first created.
  google.protobuf.Timestamp create_time = 3;
}
```

`Output only`, `Required`, `Immutable` 같은 **동작 명세를 주석 관례로** 적어 두면(AIP-203의 field behavior) 사람이 읽기 좋을 뿐 아니라, 일부 도구는 `google.api.field_behavior` 애너테이션을 통해 기계적으로 판독하기도 한다.

### 17.9.3 비공개/내부 필드와 보안 검토

내부와 외부가 같은 proto를 공유한다면 **내부 전용 필드가 외부로 새지 않도록** 주의해야 한다. 접근 방법은 두 가지다.

- **버전/패키지 분리**: 외부 공개용 proto와 내부 전용 proto를 아예 다른 패키지로 분리한다. 가장 안전하다.
- **커스텀 옵션으로 표시**: `(acme.internal) = true` 같은 커스텀 필드 옵션을 달고, 외부 SDK를 생성할 때 그 필드를 제외하는 필터를 건다.

민감 데이터(개인정보, 비밀키 등)의 보안 검토 관점에서는 다음을 확인한다.

- **민감 필드를 명시적으로 표시**하고(주석이나 커스텀 옵션), 로깅 인터셉터(8장 메타데이터와 인터셉터)가 그 필드를 자동으로 마스킹하게 한다. 예를 들어 proto에 `[(acme.pii) = true]` 같은 옵션을 달고, 로깅 미들웨어가 메시지를 직렬화할 때 해당 필드를 `***`로 치환하는 방식이다.
- **민감 데이터는 메타데이터 헤더보다 메시지 본문**에 두는 편이 일반적으로 낫다(헤더는 프록시 로그나 트레이싱에 남기 쉽다). 단, 본문도 로깅을 통해 샐 수 있으므로 마스킹이 필요하다.
- proto 리뷰 체크리스트에 "이 필드가 PII인가? 로깅에서 마스킹되는가? 외부 노출 대상인가?"를 포함한다.

---

## 17.10 테스트 전략

proto는 계약이므로, 테스트의 상당 부분은 "계약을 지키는가"를 검증하는 일이다.

### 17.10.1 in-process 서버(버퍼 기반 연결)로 빠른 통합 테스트

gRPC 통합 테스트에서 실제 TCP 포트를 열면 느리고 포트 충돌도 생긴다. 대신 **메모리 내 버퍼로 연결을 잇는** 트랜스포트를 쓴다(Go의 `bufconn`, Java의 in-process transport 등). 클라이언트와 서버는 같은 프로세스 안에서 실제 gRPC 스택(직렬화, 인터셉터, 스트리밍 전부)을 거치지만, 네트워크는 메모리 파이프로 대체된다.

```go
// Go: bufconn 기반 in-process 서버
func newTestServer(t *testing.T) pb.LibraryServiceClient {
    lis := bufconn.Listen(1024 * 1024) // 메모리 listener
    srv := grpc.NewServer()
    pb.RegisterLibraryServiceServer(srv, &libraryServer{ /* 실제 구현 */ })
    go func() { _ = srv.Serve(lis) }()
    t.Cleanup(srv.Stop)

    conn, err := grpc.NewClient(
        "passthrough:///bufnet",
        grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
            return lis.DialContext(ctx)
        }),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { _ = conn.Close() })
    return pb.NewLibraryServiceClient(conn)
}
```

이렇게 하면 실제 직렬화, 인터셉터, 에러 매핑까지 모두 거치면서도 테스트가 밀리초 단위로 끝난다. 빠지는 것은 4장(HTTP2 깊이 보기 전송 계층)에서 본 프레이밍, TLS 같은 실제 네트워크 동작뿐이다.

### 17.10.2 목/스텁

단위 테스트에서는 의존하는 다른 gRPC 서비스의 클라이언트 스텁을 **목(mock)으로 대체**한다. 생성된 클라이언트 인터페이스를 구현한 가짜 객체를 주입하는 방식이다. 단, 목은 "내가 상상한 서버 동작"을 검증할 뿐이어서 실제 계약과 어긋날 수 있다. 따라서 목만으로는 부족하고, 계약 테스트로 보완해야 한다.

### 17.10.3 계약 테스트(contract test)

목의 한계를 보완하는 것이 계약 테스트다. **클라이언트가 기대하는 요청/응답 형태**(consumer 측 계약)와 **서버가 실제로 제공하는 형태**(provider 측)를 같은 명세로 양쪽에서 검증한다. proto가 이미 구조에 대한 계약이긴 하지만, "이 필드 조합일 때 이렇게 동작한다"는 행위 계약은 proto만으로 표현할 수 없다. 계약 테스트가 그 간극을 메운다. provider 측 통합 테스트가 consumer가 의존하는 시나리오를 모두 커버하는지 확인하는 방식이 흔하다.

### 17.10.4 골든 와이어 테스트(golden wire test)

골든 와이어 테스트는 호환성 회귀를 잡는 강력한 도구다. **"옛 버전이 직렬화한 실제 바이트"를 파일로 저장(golden file)해 두고, 새 코드가 그 바이트를 그대로 디코딩할 수 있는지** 빌드마다 검증한다. 3장(Protocol Buffers 2 인코딩과 와이어 포맷)에서 본 와이어 포맷이 정말로 깨지지 않았는지를 바이트 수준에서 확인하는 것이다.

```go
func TestGoldenWireCompat(t *testing.T) {
    // v1.4.0 시절 서버가 만든 실제 바이트(hex로 박제)
    // Book{ name:"books/1", title:"1984" } 를 직렬화한 결과
    golden, _ := hex.DecodeString("0a07626f6f6b732f31120431393834")

    var b pb.Book
    if err := proto.Unmarshal(golden, &b); err != nil {
        t.Fatalf("new code cannot read old bytes: %v", err) // 호환 깨짐!
    }
    if b.GetName() != "books/1" || b.GetTitle() != "1984" {
        t.Fatalf("field drift: %+v", &b)
    }
}
```

위의 hex를 손으로 디코딩해 보면 와이어 포맷이 보인다.

```text
0a 07 62 6f 6f 6b 73 2f 31   12 04 31 39 38 34
└┬┘ └┬┘ └──────┬──────────┘  └┬┘ └┬┘ └────┬───┘
 │   │         │              │   │       │
 │   │         "books/1"      │   │       "1984"
 │   len=7                    │   len=4
 │                            tag: (2<<3)|2 = 0x12 → 필드2(title), wire type 2
 tag: (1<<3)|2 = 0x0a → 필드1(name), wire type 2(length-delimited)
```

`0x0a` = `00001010` = `(1 << 3) | 2`이므로 필드 번호 1, 와이어 타입 2(length-delimited)다. 그 뒤에 길이 `0x07`(7바이트)이 오고, 이어서 `books/1`의 ASCII가 온다. 다음 `0x12` = `(2 << 3) | 2`는 필드 번호 2, 와이어 타입 2이며, 길이 4와 `1984`가 뒤따른다.

이렇게 저장해 둔 바이트가 미래에도 같은 필드로 디코딩된다면 와이어 호환은 유지되고 있는 것이다. 누군가 필드 번호를 바꿨다면 이 테스트가 즉시 실패한다.

### 17.10.5 로드 테스트

성능과 백프레셔는 15장(성능과 Flow Control 백프레셔)에서 자세히 다뤘으므로, 여기서는 한 가지만 짚는다. proto 변경은 (특히 메시지가 커지거나 스트리밍 패턴이 바뀔 때) **성능 회귀를 일으킬 수 있으므로**, 큰 변경 뒤에는 `ghz` 같은 도구로 처리량, 지연, 메모리를 측정해 회귀가 없는지 확인한다. 메시지에 큰 `repeated`/`bytes` 필드를 추가하는 변경은 와이어 호환에는 문제가 없더라도 페이로드가 급증하고 메모리 압박이 생길 수 있다.

---

## 17.11 마이그레이션 플레이북: 무사고로 진화시키기

규칙과 도구를 모두 갖췄더라도 **변경을 진행하는 순서**를 틀리면 사고가 난다. 모든 순서는 다음 원칙 하나로 결정된다.

> **"읽는 쪽(reader)이 먼저 새 형식을 이해할 수 있어야, 쓰는 쪽(writer)이 새 형식을 보낼 수 있다."**

### 17.11.1 필드 추가 → 이중 기록 → 읽기 전환 → 구필드 reserved

가장 흔한 시나리오를 보자. `User.full_name`(단일 문자열)을 `given_name`과 `family_name`으로 분리하고 싶다. 한 번에 바꾸면 공존 기간에 누군가는 깨진다. 그래서 **4단계 expand-and-contract(또는 parallel change)** 패턴을 따른다.

```text
단계 0 (시작): full_name 만 존재
   message User { string full_name = 5; }

단계 1 (확장 expand): 새 필드 추가. 둘 다 존재. 아무도 안 깨짐.
   message User {
     string full_name  = 5;   // 아직 진실의 원천
     string given_name = 6;
     string family_name = 7;
   }
   → 서버 먼저 배포. 옛 클라는 6,7 무시. 새 클라는 채워보낼 수 있음.

단계 2 (이중 기록 dual-write): 서버가 두 표현을 항상 동기화.
   - Create/Update 시 full_name 받으면 → given/family 파싱해서도 저장
   - given/family 받으면 → full_name 합성해서도 저장
   → DB에도 양쪽 칼럼을 채워 백필(backfill) 마이그레이션 수행.

단계 3 (읽기 전환 read-switch): 모든 reader가 given/family를 읽도록 전환.
   - 서버 내부 로직, 그리고 클라들이 새 필드를 우선 사용.
   - full_name은 여전히 채워두되 read 의존을 제거.

단계 4 (수축 contract): full_name 트래픽 0 확인 후 묘비.
   message User {
     reserved 5;
     reserved "full_name";
     string given_name = 6;
     string family_name = 7;
   }
```

각 단계 사이에는 모든 클라이언트가 따라올 수 있을 만큼 **충분한 시간**을 둔다. 확장은 빠르게, 수축은 느리게 하는 것이 expand-and-contract의 핵심이다.

### 17.11.2 메서드 교체

`GetUser`의 시맨틱을 바꾸고 싶다면 기존 메서드를 바꾸지 말고 **새 메서드를 추가**한다.

```text
1. GetUserV2 (또는 더 나은 이름) 추가. 서버 배포.
2. 클라들을 GetUserV2로 점진 이전.
3. GetUser 트래픽 모니터링([[14 - 관찰성과 디버깅 - Reflection grpcurl]]).
4. 트래픽 0 → GetUser에 option deprecated=true → 충분한 유예 → 제거.
```

메서드 이름과 경로는 라우팅 계약이므로 절대 바꾸지 않는다는 17.2.6의 규칙이 여기서 적용된다.

### 17.11.3 롤아웃 순서: 서버 먼저, 호환 확장으로

분산 배포에서는 누구를 먼저 배포하느냐가 결과를 좌우한다. 규칙은 다음과 같다.

```text
새 필드/메서드를 "추가"하는 변경  →  서버(reader/handler) 먼저
   이유: 새 클라가 새 필드를 보내기 전에, 서버가 그걸 이해할 준비가 되어야 함.
   서버가 먼저 새 필드를 알면, 옛 클라(안 보냄)도 새 클라(보냄)도 모두 안전.

필드/메서드를 "제거"하는 변경    →  클라(writer/caller) 먼저
   이유: 클라가 그 필드/메서드를 더 이상 안 쓰게 된 뒤에야 서버에서 치울 수 있음.
```

한 문장으로 줄이면 **"확장은 서버부터, 수축은 클라이언트부터."** 이다. 그리고 롤링 배포 중에 구버전 파드와 신버전 파드가 공존하는 몇 분이 진짜 시험대이므로, 모든 변경은 그 공존 상태에서 양쪽이 무사한지를 기준으로 설계한다. 카나리(canary) 배포로 신버전을 소량의 트래픽에 먼저 노출해 검증하면 더 안전하다.

### 17.11.4 페이지네이션·FieldMask 적용 마이그레이션 예시

`repeated Book books`를 통째로 반환하던 기존 `ListBooks`에 페이지네이션을 "나중에" 붙이는 경우를 보자. 다행히 이 변경은 호환성을 유지하면서 할 수 있다.

```proto
// before
message ListBooksRequest  { string parent = 1; }
message ListBooksResponse { repeated Book books = 1; }

// after — 필드 추가만으로 페이지네이션 도입(와이어 호환)
message ListBooksRequest {
  string parent     = 1;
  int32  page_size  = 2;   // 추가
  string page_token = 3;   // 추가
}
message ListBooksResponse {
  repeated Book books     = 1;
  string next_page_token = 2;  // 추가
}
```

옛 클라이언트는 `page_size`와 `page_token`을 보내지 않고 `next_page_token`을 무시한다. 그래서 **서버는 토큰을 보내지 않는 옛 클라이언트에게 합리적인 기본 동작**(예: 기본 페이지 크기로 첫 페이지만 주거나, 호환을 위해 충분히 큰 기본값을 쓰는 것)을 제공해야 한다.

이때 "옛 클라이언트가 전체를 받던 동작"을 갑자기 잘라 버리면 행위 호환이 깨질 수 있으므로, 기본 page_size를 신중하게 고르거나 옛 동작을 한동안 유지한다. "와이어 호환에 문제가 없어도 행위 호환은 따로 챙겨야 한다"는 점을 보여 주는 또 다른 예다.

---

## 17.12 종합: 안전한 진화 before/after 한눈에

마지막으로 지금까지의 규칙을 하나의 diff로 정리해 보자. 같은 변경 의도를 **위험한 방식**과 **안전한 방식**으로 나란히 놓았다.

```proto
// ===== 위험한 변경 (절대 이렇게 하지 말 것) =====
message Order {
  string id = 1;
  int64  amount = 2;        // int32였던 걸 int64로 (source-breaking)
  string status = 3;        // enum이던 걸 string으로 (wire-breaking!)
  // string coupon = 4;     // 삭제하고 reserved 안 함 (재사용 위험)
  string customer = 4;      // 4번 재사용! (coupon 데이터가 customer로 오해석)
}
```

```proto
// ===== 안전한 변경 (권장) =====
import "google/protobuf/timestamp.proto";

message Order {
  string id = 1;
  int64  amount = 2 [deprecated = true]; // 만약 정말 바꿔야 하면 새 필드로
  int64  amount_minor_units = 7;          // 새 번호로 추가

  OrderStatus status = 3;                  // enum 유지(확장은 값 추가로)

  reserved 4;                              // 옛 coupon 자리에 묘비
  reserved "coupon";

  string customer_name = 5;                // 새 필드는 새 번호
  google.protobuf.Timestamp create_time = 6;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_PAID        = 2;
  ORDER_STATUS_CANCELLED   = 3;   // 값 추가 = 안전
}
```

이 diff를 CI에 올리면 `buf breaking`은 위험한 쪽에서 위반을 줄줄이 보고하고, 안전한 쪽은 통과시킨다. 거버넌스가 매일 하는 일이 바로 이것이다.

---

## 17.13 실전 체크리스트

proto PR을 올리기 전에 스스로 확인할 질문은 다음과 같다.

```text
[호환성]
☐ 기존 필드 번호를 바꾸지 않았는가?
☐ 삭제한 필드/enum값을 reserved 처리했는가?
☐ 타입을 바꿨다면 같은 호환군이고 값 의미가 보존되는가?
☐ enum에 0 = _UNSPECIFIED 가 있고, 클라가 미지 값을 default 처리하는가?
☐ oneof/required를 위험하게 건드리지 않았는가?
☐ 서비스/메서드/패키지 이름을 바꾸지 않았는가(라우팅 파괴)?
☐ buf breaking(FILE 정책)이 통과하는가?

[설계]
☐ 각 메서드가 전용 XxxRequest를 갖는가?
☐ List에 page_size/page_token/next_page_token이 있는가?
☐ Update에 FieldMask가 있는가?
☐ Create(및 증분 변경)에 request_id(멱등성)가 있는가?
☐ 오래 걸리는 작업은 LRO로 모델링했는가?
☐ 에러를 표준 상태 코드 + Rich Error 규약에 맞춰 매핑했는가?

[거버넌스]
☐ buf lint 통과(네이밍/스타일)?
☐ 공개 API 수준의 문서 주석을 달았는가?
☐ 민감 필드를 표시/마스킹 대상으로 처리했는가?
☐ CODEOWNERS 리뷰(서버+클라 양쪽)를 받았는가?

[롤아웃]
☐ "확장은 서버 먼저, 수축은 클라 먼저" 순서를 지키는가?
☐ 골든 와이어 테스트가 있는가?
☐ deprecated 트래픽을 모니터링하고 0 확인 후 제거하는가?
```
