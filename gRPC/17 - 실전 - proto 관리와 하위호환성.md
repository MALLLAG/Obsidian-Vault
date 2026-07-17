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

이 장이 답하는 질문들:

- 필드를 더하고, 빼고, 바꿀 때 "와이어 호환(wire-compatible)"이 깨지는 경계는 정확히 어디인가? 왜 그 경계가 거기 있는가?
- `buf breaking` 같은 도구는 내부에서 무엇을 비교하길래 사람이 놓치는 사고를 잡아내는가? wire-breaking·source-breaking·json-breaking은 어떻게 다른가?
- 언제 메서드를 추가하면 되고, 언제 `v2` 패키지로 가야 하는가? 멀티 버전을 동시에 굴리는 비용은 무엇인가?
- 구글이 공개한 API 설계 가이드(AIP)가 권하는 리소스 지향 메서드·페이지네이션·FieldMask·LRO는 왜 그렇게 생겼는가?
- proto를 어디에 두고(monorepo vs 중앙 repo vs BSR), 누가 리뷰하고, 어떻게 마이그레이션을 무사고로 굴리는가?

---

## 17.0 들어가며: proto는 코드가 아니라 "조약(treaty)"이다

지금까지 우리는 gRPC를 부품 단위로 뜯어봤다. [[02 - Protocol Buffers 1 - 문법과 타입 시스템]]에서 메시지를 정의하는 법을, [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 그 메시지가 바이트로 어떻게 변하는지를, [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 그 바이트가 어떻게 흐르는지를 봤다. 부품은 모두 익혔다. 이제 마지막 장에서는 **그 부품들로 만든 시스템을 몇 년 동안 운영하면서도 망가뜨리지 않는 법**을 다룬다.

여기서 사고방식의 전환이 필요하다. 평소 우리는 코드를 "내가 고치면 내가 책임지는 것"으로 생각한다. 함수 시그니처를 바꾸면 컴파일러가 호출처를 다 잡아주고, 한 번에 고치고 배포하면 끝이다. 그런데 `.proto`는 다르다. proto는 코드라기보다 **두 당사자 사이의 조약(treaty)에 가깝다.** 서버 팀이 한쪽, 클라이언트 팀(어쩌면 외부 회사, 어쩌면 6개월 전에 빌드되어 사용자 휴대폰에 깔린 앱)이 다른 쪽이다. 그리고 이 조약의 가장 잔인한 특성은 이것이다:

> **당신은 상대방을 동시에 업데이트할 수 없다.**

모바일 앱은 사용자가 업데이트하지 않으면 1년 전 proto로 계속 호출한다. 마이크로서비스 환경에서도 수십 개의 서비스가 같은 proto에 의존하는데, 이들을 한 트랜잭션처럼 동시에 배포하는 것은 불가능하다. 롤링 배포(rolling deployment) 중에는 구버전 파드와 신버전 파드가 **같은 순간에** 트래픽을 받는다. 즉, 어떤 proto 변경이든 반드시 "구버전과 신버전이 공존하는 시간"을 통과해야 한다.

그래서 이 장의 모든 규칙은 단 하나의 질문으로 환원된다:

> **"이 proto를 아는 코드(구버전)와, 저 proto를 아는 코드(신버전)가 같은 와이어 위에서 만났을 때, 둘 다 무사한가?"**

이 질문에 "예"라고 답할 수 있게 만드는 것이 하위호환성(backward compatibility)이고, 그걸 자동으로 검증하는 것이 거버넌스이며, 그걸 안전하게 굴리는 절차가 마이그레이션 플레이북이다. 하나씩 보자.

---

## 17.1 호환성의 세 방향: backward / forward / full

먼저 용어를 정확히 못 박자. 사람들이 "하위호환"이라는 말을 너무 느슨하게 써서 사고가 난다.

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

- **Backward compatibility(하위호환)**: **새 코드가 옛 데이터를 읽을 수 있다.** 신버전 서버가 구버전 클라이언트의 요청을 처리할 수 있다는 뜻. 가장 흔히 신경 쓰는 방향.
- **Forward compatibility(상위호환)**: **옛 코드가 새 데이터를 읽을 수 있다.** 구버전 클라이언트가 신버전 서버의 응답을 (모르는 필드는 무시하고) 처리할 수 있다는 뜻.
- **Full compatibility(완전 호환)**: 위 둘을 모두 만족.

Protocol Buffers의 위대한 점은 **설계상 forward·backward를 동시에 잘 지원하도록 와이어 포맷이 만들어졌다는 것**이다. 그 비밀은 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 봤듯 모든 필드가 `(필드 번호, 와이어 타입)` 태그를 앞에 달고 다니고, **디코더가 모르는 태그를 만나면 길이만큼 건너뛰도록(skip)** 되어 있기 때문이다. 이름은 와이어에 실리지 않는다. 오직 번호만 실린다. 이 단 하나의 설계 결정이 이 장 전체의 토대다.

직관적 비유: proto 메시지는 **번호표가 붙은 사물함이 늘어선 복도**다. 디코더는 "3번 사물함, 7번 사물함, 12번 사물함"을 차례로 지나가며 자기가 아는 번호만 열어보고, 모르는 번호는 그냥 지나친다. 사물함에 이름표(필드 이름)가 붙어 있지만 그건 사람이 보라고 붙인 것일 뿐, 복도를 걷는 기계는 번호만 본다. 그래서:

- **번호를 바꾸면** 기계가 엉뚱한 사물함을 연다 → 재앙.
- **이름만 바꾸면** 기계는 신경도 안 쓴다 → 와이어엔 무해.
- **새 번호 사물함을 추가하면** 옛 기계는 그냥 지나친다 → 안전.
- **사물함을 없애면** 옛 기계가 빈 자리를 찾다가 헷갈릴 수 있으니, "이 번호는 폐쇄됨"이라고 표시(reserved)해 둬야 한다.

이제 이 직관을 실전 규칙으로 정밀화하자.

---

## 17.2 실전 호환성 규칙 총정리

아래 규칙들의 *원리*(왜 바이트 레벨에서 그런가)는 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에 있다. 여기서는 *실전 판단*에 집중한다. 같은 규칙들이 구글의 공개 가이드 AIP-180(Backwards compatibility)에도 "무엇이 호환 파괴인가"의 목록으로 정리돼 있으니, 팀 표준을 만들 때 교차 참고할 만하다. 먼저 한눈에 보는 표.

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

하나씩 *왜*를 파고들자.

### 17.2.1 필드 번호는 신성불가침이다

필드 번호는 와이어에 실제로 인코딩되는 유일한 식별자다. 태그 바이트는 `(field_number << 3) | wire_type` 공식으로 만들어진다. 예를 들어 필드 번호 2, 와이어 타입 0(varint)이면:

```text
tag = (2 << 3) | 0 = 0b00010000 = 0x10
```

만약 어느 날 누군가 `user_id`의 번호를 2에서 3으로 바꾸면, 옛 클라이언트는 여전히 태그 `0x10`(번호 2)을 보내는데 새 서버는 그걸 "내가 모르는 번호 2"로 보고 건너뛴다. `user_id`는 비어 있게 되고, 정작 `user_id`로 해석돼야 할 데이터가 사라진다. 컴파일 에러도, 예외도 없다. **조용히 데이터가 증발한다.** 이게 proto 사고가 무서운 이유다 — 타입 시스템이 못 잡는다.

규칙: **한 번 릴리스된 필드 번호의 의미는 영원히 고정.** 의미를 바꾸고 싶으면 새 번호의 새 필드를 만들어라.

또 하나 알아둘 것: 1~15번 필드는 태그가 **1바이트**, 16번부터는 **2바이트** 이상이다(varint 인코딩 때문). 그래서 자주 쓰이고 반복되는 필드일수록 1~15번에 배치하는 게 인코딩 효율에 좋다. 이건 진화 규칙은 아니지만 처음 설계할 때 자리를 잘 잡아두는 실전 팁이다. 19000~19999는 protobuf 구현이 내부적으로 예약한 범위라 못 쓴다.

### 17.2.2 필드를 삭제할 땐 반드시 reserved로 무덤을 파라

필드를 지우는 것 자체는 와이어 호환을 깨지 않는다(옛 클라가 보낸 그 번호를 새 서버가 무시할 뿐). 진짜 위험은 **미래에 누군가 그 번호를 재사용하는 것**이다. 6번 필드 `phone_number`(string)를 지웠는데, 1년 뒤 신입이 6번을 `age`(int32)로 재사용했다고 하자. 여전히 옛 proto로 빌드된 클라이언트가 6번에 전화번호 문자열을 보내면, 새 서버는 그걸 int32 age로 디코딩하려다 쓰레기 값을 얻거나 파싱 에러를 낸다.

그래서 protobuf는 `reserved`라는 묘비를 제공한다. 필드를 지울 때 **번호와 이름을 둘 다 예약**하라.

```proto
message User {
  string id = 1;
  string email = 2;
  // string phone_number = 6;  ← 삭제

  reserved 6;                  // 번호 무덤
  reserved "phone_number";     // 이름 무덤 (JSON/텍스트 포맷·코드 재사용 방지)
}
```

이렇게 하면 누가 6번이나 `phone_number`라는 이름을 다시 쓰려는 순간 **컴파일러가 거부**한다. 묘비는 사람의 선의를 믿지 않고 도구가 강제하게 만드는 장치다. 범위로도 예약할 수 있다: `reserved 6, 9 to 11, 20;`.

> 실전 팁: 필드를 곧장 삭제하기보다 먼저 `deprecated = true` 옵션을 달아 사용을 줄이고(아래 17.5 참조), 충분히 트래픽이 빠진 뒤 삭제+reserved로 가는 2단계가 안전하다.

### 17.2.3 타입 변경 — "호환군(compatibility group)"이라는 좁은 안전지대

타입 변경은 대부분 금지지만, **와이어 타입이 같은 정수 계열 안에서는 제한적으로 호환**된다. [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 본 와이어 타입 분류가 그대로 적용된다.

varint(와이어 타입 0)로 인코딩되는 그룹끼리는 서로 바꿔도 와이어가 깨지지 않는다:

```text
int32, int64, uint32, uint64, bool, enum   ← 전부 varint(타입 0)
```

단, "와이어가 안 깨진다"와 "값이 보존된다"는 다르다. 함정이 곳곳에 있다:

- `int32 ↔ int64`: 와이어 OK. 그런데 `int32`로 디코딩하면서 값이 32비트를 넘으면 잘린다(truncate). 음수를 `int32`로 보내면 항상 10바이트 varint로 인코딩되는 것도 주의.
- `int32 ↔ uint32`: 와이어 OK. 그러나 음수 비트 패턴이 양수로 재해석된다. `-1`(int32)이 `4294967295`(uint32)로 둔갑.
- `sint32/sint64`: 이건 **다르다.** ZigZag 인코딩을 쓰기 때문에 같은 varint 타입이어도 `int32`와 `sint32`는 **호환되지 않는다.** 비트 패턴 자체가 다르다.
- `fixed32 ↔ sfixed32`: 둘 다 와이어 타입 5(32비트 고정). 호환. `fixed64 ↔ sfixed64`는 타입 1(64비트 고정). 호환.
- `string ↔ bytes`: 둘 다 와이어 타입 2(length-delimited). 호환되긴 하나, string은 UTF-8 검증을 하므로 bytes에 비-UTF8 데이터가 있으면 string 디코딩이 깨진다. bytes→string은 위험, string→bytes는 비교적 안전.
- `int32 → string`: **완전 파괴.** 와이어 타입 0 → 2. 디코더가 길이 프리픽스를 기대하다가 varint를 만나 폭발.

규칙의 핵심: **"같은 호환군이라도 값 의미가 보존되는지 따로 검증하라."** 도구(`buf breaking`)는 와이어 호환군은 검사해주지만, "비즈니스 로직상 -1이 4294967295가 되어도 괜찮은가"는 사람이 판단해야 한다.

### 17.2.4 enum: 값 추가는 안전, 단 "미지의 값" 처리가 관건

enum에 값을 추가하는 것은 와이어 호환을 깨지 않는다. enum은 와이어에서 그냥 varint(정수)이기 때문이다. 문제는 **옛 코드가 모르는 enum 값을 받았을 때 무슨 일이 일어나느냐**다. 이건 proto2와 proto3가 결정적으로 다르다.

- **proto3 (open enum)**: 모르는 enum 값도 **그대로 정수로 보존**된다. 옛 클라가 새 값 `STATUS_ARCHIVED = 5`를 받으면, 자기 enum에 5가 없어도 정수 5를 들고 있다가 그대로 다시 직렬화할 수 있다. 원시 정수를 꺼내는 방법은 언어마다 다르다 — Go는 enum 자체가 `int32` 별칭이라 그냥 정수로 쓰면 되고 Java/Kotlin은 미지 값이 `UNRECOGNIZED`(number = -1)로 들어오므로 `getNumber()`를 부르면 예외가 나고 대신 `getStatusValue()` 같은 `...Value` 접근자로 원시 정수를 읽어야 한다. switch 문에서는 보통 `UNRECOGNIZED` 또는 매칭 안 됨으로 떨어진다.
- **proto2 (closed enum)**: 모르는 값은 unknown field로 취급되어 알려진 필드 집합에서 빠진다. 더 엄격.

그래서 **모든 enum의 첫 값(0번)은 반드시 `XXX_UNSPECIFIED`로 두는 것**이 정석이다(이건 AIP-126 권고이기도 하다). 이유는 두 가지다. 첫째, proto3에서 모든 스칼라는 0이 기본값이라 "값을 안 보냄"과 "0을 보냄"이 구분 안 된다. 0을 의미 있는 상태로 쓰면 "설정 안 됨"을 표현할 수 없다. 둘째, 미래에 추가될 값을 받는 옛 코드가 안전하게 떨어질 기본 케이스가 있어야 한다.

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

클라이언트 코드는 **반드시 미지 값을 다루는 default 분기**를 둬야 한다. 이게 forward compatibility의 실제 코드 모양이다.

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

> 안티패턴: enum 값을 **삭제하거나 번호를 재배치**하는 것. 와이어상으론 정수가 살아남지만 의미가 틀어진다. 삭제 대신 `reserved`로 묘비를 세운다(enum도 `reserved` 지원). 또한 enum 이름만 바꾸는 것도 JSON 직렬화에선 깨진다(JSON은 값 이름 문자열을 쓸 수 있으므로).

### 17.2.5 oneof와 required — 가장 사고가 잦은 두 지뢰

**oneof 변경**은 특히 위험하다. oneof는 "이 중 정확히 하나만 설정됨"이라는 제약을 표현하는데, 와이어상으로는 그냥 같은 메시지 안의 평범한 필드들이다. 그래서 **여러 개가 동시에 와이어에 실릴 수도 있고**(그러면 마지막 것이 이긴다) 진화 규칙이 미묘하다.

- 기존 **단일 필드를 oneof 안으로 옮기는 것**: 와이어 번호가 같으면 바이트는 호환되지만 생성 코드의 API가 완전히 바뀐다(접근자가 `oneof case` 기반으로). 소스 파괴. 또한 의미도 바뀐다 — 이제 그 필드를 세팅하면 같은 oneof의 다른 필드가 클리어된다.
- **여러 기존 필드를 하나의 oneof로 묶는 것**: 과거에는 둘 다 세팅된 메시지가 와이어에 존재할 수 있었는데, 이제 그건 oneof 불변식을 위반한다. 파싱 시 마지막 것만 남는 등 데이터 손실.
- **oneof에 새 케이스 추가**: 비교적 안전(새 필드 추가와 유사). 단 클라이언트의 `switch (case)`가 새 케이스를 default로 떨어뜨려야 함.

규칙: **oneof는 처음 설계할 때 신중히, 이후엔 "케이스 추가"만 하라.** 평범한 필드를 oneof로, 또는 그 반대로 옮기는 건 사실상 새 메시지를 만드는 수준의 변경이다.

**required(proto2)**는 더 단순하다. **쓰지 마라.** proto3는 아예 `required`를 없앴는데, 그 자체가 거대한 교훈이다. required 필드는 진화의 독이다. 한 번 required로 만들면 영원히 빼지 못한다(빼는 순간 옛 메시지를 못 읽거나 못 보냄). 새 required 필드를 추가하면 옛 클라가 보낸 메시지(그 필드가 없는)가 "유효하지 않음"이 되어 파싱 단계에서 거부된다. proto2를 다뤄야 한다면 모든 신규 필드는 `optional`로.

### 17.2.6 rename — 와이어는 웃지만 코드와 JSON은 운다

메시지·서비스·메서드·필드의 **이름**을 바꾸는 것은 와이어(binary) 레벨에서는 무해하다. 앞서 말했듯 와이어엔 번호만 실리니까. 하지만:

- **생성 코드가 깨진다(source-breaking).** `User` → `Account`로 메시지 이름을 바꾸면 그 타입을 import하던 모든 코드가 컴파일 실패. 이건 "내 코드만의 문제"가 아니다. 그 생성 코드 라이브러리를 쓰는 모든 다운스트림이 깨진다.
- **JSON 표현이 깨진다(json-breaking).** protobuf의 canonical JSON 매핑은 필드 이름(정확히는 lowerCamelCase 변환된 JSON 이름)을 키로 쓴다. 필드 이름을 바꾸면 JSON 키가 바뀌어 [[16 - gRPC-Web과 REST 게이트웨이]]의 REST/gRPC-Web JSON 경로가 깨진다. (`json_name` 옵션으로 JSON 키를 고정해 막을 수는 있다.)
- **서비스·메서드 이름은 HTTP/2 `:path`에 들어간다.** gRPC 호출은 `POST /package.ServiceName/MethodName` 경로로 라우팅된다([[07 - 채널 스텁 커넥션 생명주기]] 참조). 서비스나 메서드 이름을 바꾸면 옛 클라가 옛 경로로 호출하는데 서버엔 그 경로가 없어 `UNIMPLEMENTED`가 난다. **이건 와이어 파괴에 준한다.** 그래서 패키지/서비스/메서드 이름은 사실상 영구 계약으로 취급해야 한다.

정리하면: 필드 이름 변경은 와이어 OK·코드/JSON 파괴, **서비스/메서드/패키지 이름 변경은 라우팅 파괴**. 후자가 훨씬 위험하다.

---

## 17.3 깨짐의 세 종류: wire / source / json breaking

"breaking change"라는 한 단어 안에 사실 세 가지 다른 재앙이 섞여 있다. 이걸 구분하지 못하면 "와이어는 안 깨졌으니 괜찮아" 하고 배포했다가 클라이언트 빌드가 전부 멈추는 사고가 난다.

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

우선순위가 명확하다. **wire-breaking이 압도적으로 위험**하다. source-breaking은 적어도 빌드가 멈춰서 알려주지만, wire-breaking은 프로덕션에서 데이터가 조용히 사라질 때까지 아무도 모른다. 그래서 거버넌스 도구는 wire-breaking 검출을 가장 엄격하게 한다.

`buf`는 이 세 등급을 그대로 정책 카테고리로 제공한다:

- `WIRE`: 와이어 호환만 검사(가장 느슨). 바이너리만 주고받고 코드 공유 안 하는 극단적 경우.
- `WIRE_JSON`: 와이어 + JSON 호환. JSON도 쓰는 경우(REST 게이트웨이 등).
- `FILE`(기본·권장): 위 둘 + 생성 코드 안정성(source). 같은 빌드에서 생성 코드를 공유하는 대부분의 경우.
- `PACKAGE`: `FILE`과 유사하나 파일 이동을 패키지 단위로 허용(파일을 옮겨도 같은 패키지면 OK).

대부분 팀은 `FILE`을 쓴다. "와이어도 코드도 둘 다 안 깨지게 하라"가 합리적 기본값이기 때문이다.

---

## 17.4 breaking change 자동 검출: buf breaking을 CI 게이트로

사람은 실수한다. "필드 번호 안 바꿨지?"를 매 PR마다 사람이 눈으로 검사하는 건 지속 불가능하다. 그래서 **과거 스키마를 "기준 이미지(baseline)"로 박제해두고, 새 스키마를 자동 비교**하는 도구가 필요하다. `buf breaking`이 사실상 표준이다. 도구 자체에 대한 전반은 [[06 - 코드 생성과 protoc 툴체인]]에서 다뤘으니, 여기선 breaking 검사의 *작동 원리*에 집중한다.

### 17.4.1 내부에서 무슨 일이 일어나는가

핵심 아이디어는 단순하다: **proto 스키마 자체를 데이터로 직렬화해서 두 시점을 비교한다.** protobuf 컴파일러는 모든 `.proto`를 파싱하면 `FileDescriptorSet`이라는 메시지(역시 protobuf로 정의됨, `descriptor.proto`)로 표현할 수 있다. 여기엔 모든 메시지·필드·번호·타입·라벨·enum·서비스가 구조화되어 들어 있다.

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

`buf`는 `buf build`로 현재 proto를 descriptor 이미지(`.binpb`)로 만들 수 있고, breaking 검사는 "기준 이미지 vs 현재"를 규칙별로 대조한다. 규칙은 예컨대:

- `FIELD_NO_DELETE`: 기준에 있던 필드가 사라졌는데 reserved 처리도 안 됨
- `FIELD_SAME_TYPE`: 같은 번호 필드의 타입이 호환 안 되게 바뀜
- `FIELD_SAME_NUMBER`: 같은 이름 필드의 번호가 바뀜
- `ENUM_VALUE_NO_DELETE`, `RPC_NO_DELETE`, `MESSAGE_NO_DELETE` 등

기준 이미지는 보통 **git의 특정 브랜치/태그**로 잡는다. "현재 PR의 proto vs main 브랜치의 proto"를 비교하는 것이 가장 흔한 설정이다.

### 17.4.2 CI 게이트 구성

`buf.yaml`(모듈 설정)과 CI 단계는 대략 이렇게 생긴다.

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

이 명령이 **0이 아닌 종료 코드를 내면 CI를 실패**시킨다. 이것이 "게이트(gate)"의 의미다. **breaking change를 담은 PR은 사람이 명시적으로 승인 절차를 밟지 않는 한 머지될 수 없다.** 이 한 줄의 CI 설정이 proto 사고의 8할을 막는다.

### 17.4.3 위반 예시와 수정

**위반 사례 1 — 무심코 타입 바꾸기**

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

흥미롭게도 `int32 → int64`는 같은 varint 호환군이라 `buf`의 와이어 정책에서는 **통과**한다. 그러나 생성 코드에서 타입이 `int` → `long`으로 바뀌므로 **source-breaking**이고, `FILE` 정책에서는 언어별 타입이 바뀌는 것으로 잡힌다. 이게 "와이어는 OK인데 소스는 깨짐"의 교과서적 사례다. 정말 64비트가 필요하면 새 필드를 추가하는 게 안전하다:

```proto
message Product {
  string id = 1;
  int32 price_cents = 2 [deprecated = true];  // 옛 필드 유지
  int64 price_cents_v2 = 3;                    // 새 필드
}
```

**위반 사례 2 — 필드 삭제 후 reserved 누락**

```proto
// after
message User {
  string id = 1;
  // string nickname = 4;  ← 삭제만 하고 reserved 안 함
}
```

`buf breaking`은 `FIELD_NO_DELETE` 위반을 보고한다. 수정:

```proto
message User {
  string id = 1;
  reserved 4;
  reserved "nickname";
}
```

이러면 "의도적으로 묘비를 세웠다"는 신호가 되어 통과한다.

> 핵심: 도구는 "와이어/소스/JSON 호환"을 기계적으로 검사한다. 하지만 "의미가 보존되는가"(예: int32→uint32로 음수가 양수가 되어도 비즈니스가 괜찮은가)는 여전히 사람의 리뷰 몫이다. 도구는 안전망이지 판단의 대체가 아니다.

---

## 17.5 API 버저닝 전략: 언제 메서드를 더하고, 언제 v2로 가는가

호환성 규칙을 다 지켜도, 어떤 변경은 그 자체로 호환되지 않는다. 리소스의 핵심 구조가 바뀌거나, 메서드의 시맨틱이 근본적으로 달라질 때다. 이때 versioning이 등장한다.

### 17.5.1 비파괴적 진화가 우선 — 버전을 올리는 건 최후의 수단

버저닝의 제1원칙은 역설적이게도 **"가능하면 버전을 올리지 마라"**다. 새 메이저 버전은 두 배의 운영 비용을 의미한다(둘 다 유지보수, 둘 다 모니터링, 클라 마이그레이션 유도). 그래서 대부분의 변경은 버전을 올리지 않고 소화해야 하고, 그게 가능하도록 위의 호환 규칙이 존재한다.

비파괴적으로 할 수 있는 일들:

- **필드 추가**: 요청·응답에 optional 필드 추가. 옛 클라는 무시, 새 클라는 활용.
- **메서드 추가**: 서비스에 새 RPC 추가. 옛 클라는 호출 안 할 뿐 영향 없음. 이게 "버전 안 올리고 기능 추가"의 핵심 도구다.
- **enum 값 추가**(미지 값 처리 전제).

예: `GetUser`만 있던 서비스에 `BatchGetUsers`를 추가하는 것은 완전히 안전하다. 새 메서드는 새 `:path`를 만들 뿐 기존 경로를 건드리지 않는다.

### 17.5.2 메이저 버전은 패키지 경로에 박는다

진짜로 호환 불가능한 변경(리소스 모델 재설계, 필드 의미 전면 변경)이 필요하면, **패키지 이름에 버전을 넣어 완전히 분리된 새 API를 만든다.** AIP-215가 권하는 방식이기도 하다.

```proto
// v1
package acme.orders.v1;
service OrderService { rpc GetOrder(GetOrderRequest) returns (Order); }

// v2 — 완전히 별개의 패키지, 별개의 서비스
package acme.orders.v2;
service OrderService { rpc GetOrder(GetOrderRequest) returns (Order); }
```

이렇게 하면:

- 생성 코드의 타입이 `acme.orders.v1.Order`와 `acme.orders.v2.Order`로 **이름공간이 갈려** 충돌 없음.
- HTTP/2 라우팅 경로가 `/acme.orders.v1.OrderService/GetOrder` vs `/acme.orders.v2.OrderService/GetOrder`로 갈려 **한 서버가 두 버전을 동시에 서빙** 가능.
- 옛 클라는 v1 경로로, 새 클라는 v2 경로로 각자 호출. 공존 가능.

`v1alpha1`, `v1beta1` 같은 안정성 채널 표기도 흔하다(AIP-185). alpha는 언제든 깨질 수 있음, beta는 비교적 안정, 버전 접미사 없는 stable은 호환 보장. AIP-185의 규칙을 정확히 옮기면 **stable 채널은 접미사를 붙이지 않고**(`v1`이지 `v1beta`가 아님), **alpha·beta 채널은 안정성 접미사 뒤에 증가하는 릴리스 번호를 붙인다**(`v1beta1`, `v1alpha5`). 또한 마이너·패치 버전을 노출하지 않는다(`v1`이지 `v1.2`가 아님) — 호환 가능한 변경은 같은 메이저 버전 안에서 소화하기 때문이다. 외부에 공개하는 API라면 이 안정성 레벨을 명시하는 게 사용자에게 친절하다.

### 17.5.3 deprecation 절차 — 갑자기 끄지 않는다

버전이나 필드를 없앨 때는 "공지 → 유예 → 제거"의 절차를 밟는다. 갑작스러운 제거는 옛 클라를 죽인다.

1. **표시(mark)**: `deprecated = true` 옵션을 단다. 이건 생성 코드에 deprecation 경고를 만들어 개발자에게 "이거 곧 없어짐"을 알린다.

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

2. **측정(measure)**: deprecated된 필드·메서드의 **실제 트래픽을 모니터링**한다([[14 - 관찰성과 디버깅 - Reflection grpcurl]] 참조). 메서드 단위 호출량, 어떤 클라(메타데이터의 user-agent/버전)가 아직 쓰는지를 추적. 트래픽이 0에 수렴할 때까지 기다린다.

3. **유도(migrate)**: 남은 클라 소유자에게 마이그레이션을 직접 요청. 외부 API라면 deprecation 정책(예: "deprecated 후 최소 12개월 유지")을 문서화하고 지킨다.

4. **제거(remove)**: 트래픽이 0이 되고 유예가 끝나면 필드는 `reserved`로, 메서드는 (필요시) 제거. 메이저 버전 전체를 끌 때도 같은 절차.

### 17.5.4 멀티 버전 동시 운영의 현실

v1과 v2를 동시에 굴리는 건 비용이 든다. 흔한 패턴은 **v2를 진실의 원천으로 삼고 v1을 얇은 어댑터(shim)로 구현**하는 것이다.

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

이러면 비즈니스 로직은 v2 한 군데만 유지하고, v1은 형태 변환만 담당한다. 코드 중복과 로직 드리프트를 막는다. 단, v1↔v2 변환에서 표현 불가능한 필드(v2에만 있는 새 개념)는 어떻게 누락/기본값 처리할지 명시적으로 정해야 한다.

---

## 17.6 리소스 지향 API 설계(구글 AIP 기반)

호환성은 "어떻게 안 깨뜨리느냐"의 문제고, 설계는 "애초에 어떻게 잘 만드느냐"의 문제다. 잘 설계된 API는 진화도 쉽다. 여기서는 구글이 공개한 API Improvement Proposals(AIP, aip.dev)의 핵심 패턴을 본다. 이것들은 특정 회사 내부 규칙이 아니라 **공개된 산업 표준에 가까운 가이드**다.

### 17.6.1 리소스 지향: 명사 + 표준 동사

REST가 자원(URL)과 HTTP 메서드(GET/POST/...)로 세상을 모델링하듯, gRPC에서도 **리소스(명사)를 정의하고 표준 메서드(동사)를 붙이는** 방식이 유지보수에 강하다. 임의의 RPC를 마구 만드는 것보다 규칙적이고 예측 가능하다.

표준 5메서드(AIP-131~135):

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

여기서 눈여겨볼 관례:

- 리소스의 식별자 필드는 관례적으로 `name`이고, **계층적 리소스 경로**(`shelves/{shelf}/books/{book}`)를 담는다. 이게 REST URL 경로와 자연스럽게 매핑되어 [[16 - gRPC-Web과 REST 게이트웨이]]의 `google.api.http` 애너테이션과 잘 맞는다.
- `create_time`, `update_time`은 `google.protobuf.Timestamp`(이런 잘 알려진 타입들은 [[02 - Protocol Buffers 1 - 문법과 타입 시스템]] 참조). 서버가 채우는 출력 전용(output only) 필드.
- 각 메서드는 **자기 전용 Request 메시지**를 둔다. `GetBook(Book)`이 아니라 `GetBook(GetBookRequest)`. 왜? 나중에 Get에만 필요한 파라미터(예: `view`, `read_mask`)를 추가할 여지를 남기기 위해서다. 응답 타입으로 리소스를 직접 주는 건 괜찮지만, 요청은 반드시 전용 래퍼를 쓴다. **이건 진화 가능성을 위한 가장 중요한 설계 습관 중 하나다.**

### 17.6.2 표준 List 페이지네이션: page_token / page_size

대량 컬렉션을 한 번에 다 주는 건 메모리·지연·[[15 - 성능과 Flow Control 백프레셔]] 측면에서 재앙이다. 표준 페이지네이션(AIP-158)은 **불투명 토큰(opaque token)** 방식을 쓴다.

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

왜 offset(`page=3&limit=20`)이 아니라 **불투명 토큰**인가? 이게 설계의 핵심 통찰이다:

- **offset 페이지네이션은 데이터가 변하면 깨진다.** 1페이지를 본 사이에 누가 항목을 추가/삭제하면 offset이 밀려 항목이 중복되거나 건너뛰어진다.
- **불투명 토큰은 "다음 페이지의 커서 위치"를 서버가 내부적으로 인코딩**한다(마지막 항목의 키, 정렬 상태, 필터 등). 클라는 이 토큰을 해석하려 들면 안 된다(그냥 다음 호출에 그대로 돌려줄 뿐). 그래서 서버는 페이지네이션 구현(키셋 커서든 다른 무엇이든)을 **클라를 깨지 않고 자유롭게 바꿀 수 있다.** 토큰이 불투명하다는 것 자체가 진화 가능성을 보장한다.

클라이언트 사용 패턴:

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

> 토큰에 만료·서명을 넣어 변조를 막거나, 서버 재시작에도 유효하도록 stateless하게(키셋 정보를 토큰에 직접 인코딩) 설계하는 게 견고하다.

### 17.6.3 부분 업데이트와 FieldMask

Update에서 가장 흔한 사고는 **"PUT 시맨틱"으로 인한 의도치 않은 필드 초기화**다. 클라가 `title`만 바꾸려고 `UpdateBook`을 호출하면서 `author`를 안 채우면, 순진한 서버는 `author`를 빈 문자열로 덮어쓴다. proto3에선 "안 보냄"과 "빈 값"이 구분 안 되니 더 위험하다.

해법이 **`google.protobuf.FieldMask`**다(AIP-134). "내가 진짜로 바꾸려는 필드 경로 목록"을 명시적으로 같이 보낸다.

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

서버는 마스크를 순회하며 해당 경로만 적용한다. `update_mask`가 비어 있으면 "전체 교체"로 볼지 "아무것도 안 바꿈"으로 볼지 정책을 정해 문서화한다(AIP는 비어 있으면 전체 교체로 보는 관례를 든다). 마스크는 중첩 경로(`author.email`)도 표현할 수 있어 깊은 부분 업데이트가 가능하다.

같은 FieldMask를 **읽기 쪽 `read_mask`**로도 쓴다 — 큰 리소스에서 클라가 원하는 필드만 받아 대역폭을 아끼는 용도. FieldMask는 `google.protobuf`의 잘 알려진 타입이라 모든 언어 런타임이 헬퍼를 제공한다.

### 17.6.4 Long-Running Operations(LRO): 오래 걸리는 작업의 표준

어떤 작업은 수 초~수 분 걸린다(대용량 내보내기, 인덱스 재구축 등). 이걸 동기 unary로 두면 클라가 [[10 - Deadline 취소 타임아웃]]에 걸려 죽거나, 연결을 오래 붙잡는다. 표준 해법은 **작업을 즉시 "Operation 핸들"로 반환하고, 완료를 폴링/조회**하게 하는 것이다(AIP-151).

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

클라는 `Operation.name`으로 `GetOperation`을 폴링하거나, 별도 알림 채널로 완료를 받는다. 결과는 `oneof { error, response }`로 들어오는데, 이는 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에서 본 `google.rpc.Status`를 그대로 재사용한다. 서버 스트리밍([[05 - 통신의 4가지 방식 - Unary와 Streaming]])으로 진행률을 푸시하는 변형도 있지만, LRO는 "연결이 끊겨도 작업이 살아남고 나중에 다시 조회 가능"하다는 점에서 장수 작업에 더 견고하다.

### 17.6.5 필터·정렬

List에 `filter`(문자열)와 `order_by`(문자열) 필드를 둔다(AIP-160, AIP-132). 왜 구조화된 메시지가 아니라 문자열인가? **표현력과 진화 때문**이다. 필터 문법(예: `author = "Orwell" AND create_time > "2020-01-01T00:00:00Z"`)을 문자열로 두면, 새 연산자·필드를 추가해도 proto를 안 바꿔도 된다. 단, 서버는 이 문자열을 안전하게 파싱(인젝션 방지)해야 한다. `order_by`는 `"create_time desc, title"` 같은 형식.

### 17.6.6 멱등성과 요청 ID: 안전한 재시도의 토대

[[13 - 안정성 - Retry Health Check Keepalive]]에서 봤듯 gRPC는 재시도를 한다. 그런데 `CreateBook`을 재시도하면 책이 두 권 생길 수 있다. 네트워크가 응답을 잃었을 뿐 서버는 이미 처리했을 수도 있으니까. 해법은 **클라가 요청마다 고유 ID를 부여**하고 서버가 그걸로 중복을 제거하는 것이다(AIP-155).

```proto
message CreateBookRequest {
  string parent = 1;
  Book   book = 2;
  // 클라가 생성한 UUID. 같은 ID의 요청은 한 번만 실행됨이 보장.
  string request_id = 3;
}
```

서버는 `request_id`를 일정 기간 저장해두고, 같은 ID가 다시 오면 **새로 실행하지 않고 이전 결과를 그대로 반환**한다. 이러면 Create 같은 비멱등(non-idempotent) 연산도 **재시도에 안전(idempotent하게)** 만들 수 있다. Get/List/Delete는 본래 멱등하지만, Create와 (값이 아닌 증분) Update에 request_id가 특히 중요하다.

이 패턴은 [[08 - 메타데이터와 인터셉터]]의 인터셉터에서 request_id를 자동 주입하거나, 메타데이터 헤더(`x-idempotency-key`)로 옮겨 횡단 관심사로 처리할 수도 있다.

---

## 17.7 에러 설계: 도메인 에러 → 상태 코드 + Rich Error의 일관 규약

API 설계의 절반은 "성공했을 때"고 나머지 절반은 "실패했을 때"다. 에러 모델의 메커니즘은 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에서 다뤘으니, 여기서는 **진화·거버넌스 관점의 규약**만 정리한다.

핵심은 **"도메인 에러를 표준 상태 코드에 일관되게 매핑하고, 디테일은 Rich Error로 구조화한다"**는 팀 전체의 합의다. 같은 종류의 실패가 서비스마다 다른 코드로 나오면 클라가 미쳐버린다.

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

진화 규칙으로서의 에러 규약:

- **새 에러 "상황"을 추가하는 것은 OK.** 클라는 모르는 상태 코드/Rich Error 타입을 만나면 일반적인 실패로 처리할 수 있어야 한다(forward compatibility). default 분기 필수.
- **기존 에러의 상태 코드를 바꾸는 것은 사실상 breaking.** 클라가 `NOT_FOUND`를 보고 분기 로직을 짜뒀는데 어느 날 `FAILED_PRECONDITION`으로 바뀌면 로직이 깨진다. 상태 코드 매핑은 계약의 일부로 취급하라.
- **Rich Error의 `ErrorInfo.reason`은 안정적인 enum-like 문자열**(예: `"BOOK_OUT_OF_PRINT"`)로 두고, 사람이 읽는 `message`와 분리한다. 클라는 `reason`으로 분기하고 `message`는 화면에만 쓴다. `message` 문구를 바꿔도 클라 로직이 안 깨지게.

또 하나의 설계 분파로 **응답 메시지 안에 `oneof { data, error }`를 두어 도메인 에러를 타입으로 표현**하는 방식이 있다. gRPC 네이티브 상태 코드는 결국 "정수 + 문자열"이라 도메인 에러를 표현하면 stringly-typed가 되기 쉬운데, 에러를 proto 메시지로 정의하면 컴파일 타임 안전성(언어별 패턴 매칭)을 얻는다. 다만 이 경우 "DB 장애·네트워크 오류 같은 진짜 내부 오류"는 여전히 gRPC 네이티브 status(INTERNAL, UNAVAILABLE)로 올려보내고, `oneof error`에는 **호출자가 의미 있게 처리할 수 있는 도메인 에러만** 담는 게 좋다. 어느 규약을 쓰든, 서비스마다 어떤 규약을 따르는지 명시하는 것이 거버넌스의 일이다.

---

## 17.8 proto 저장소 전략: 어디에 두고 누가 책임지는가

proto를 잘 짜는 것만큼 중요한 게 **proto를 어디에 두고 어떻게 배포하느냐**다. 세 가지 큰 흐름이 있다.

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

선택 기준은 조직 규모와 다언어 정도다. 단일 언어·소수 서비스면 [A]가 단순하고 좋다. 여러 팀·여러 언어가 같은 계약을 공유하면 [B]나 [C]가 일관성을 준다. BSR(Buf Schema Registry) 같은 레지스트리는 proto를 "패키지"처럼 의존하게 해주고(`buf.lock`으로 버전 고정), breaking 검사를 게시 시점에 강제하며, 다언어 SDK 생성을 중앙화한다. (워크스페이스가 여러 repo로 나뉜 환경에서 git submodule로 interface proto를 공유하는 변형도 있는데, 이 경우 main pull 후 submodule update를 빠뜨리면 컴파일이 깨지는 함정이 있으니 동기화 절차를 자동화해두는 게 좋다.)

### 17.8.2 생성 코드 배포: 라이브러리 게시 vs 빌드시 생성

proto에서 만든 생성 코드를 소비자에게 전달하는 방식도 둘로 갈린다.

| 방식 | 작동 | 장점 | 단점 |
|---|---|---|---|
| **라이브러리 게시(pre-generated)** | proto 변경 시 CI가 언어별 패키지(Go module, Maven artifact, npm, PyPI 등)를 빌드해 게시. 소비자는 그걸 의존성으로 설치. | 소비자 빌드가 빠르고 단순. 버전 고정 명확. protoc 툴체인 불필요. | 게시 파이프라인 유지. 버전 폭증 가능. |
| **빌드시 생성(generate-on-build)** | 소비자가 proto를 가져와 자기 빌드에서 `protoc`/`buf generate` 실행. | 항상 최신. 중간 아티팩트 없음. | 모든 소비자가 protoc 플러그인 버전을 맞춰야 함. 빌드 느림·재현성 이슈. |

실전에선 **라이브러리 게시가 다언어·다팀 환경에서 우세**하다. 특히 외부에 SDK를 줄 때는 소비자에게 protoc를 강요할 수 없으니 게시가 사실상 필수다([[06 - 코드 생성과 protoc 툴체인]]에서 생성 도구를 다뤘다). 핵심은 **생성 코드의 버전이 proto 버전과 결정적으로 묶이는 것**이다 — 같은 proto에서 항상 같은 코드가 나오도록(재현 가능한 빌드, 플러그인 버전 고정).

### 17.8.3 다언어 동기화

같은 proto에서 Go·Java·Python·TypeScript 코드를 뽑을 때, 모두가 같은 시점의 같은 proto를 보도록 해야 한다. 흔한 함정: Go 서버는 새 필드를 알지만 Python 클라 SDK는 아직 옛 버전이라 그 필드를 못 쓰는 경우. 이건 와이어 호환이라 안 깨지긴 하지만 기능이 반영 안 된 것처럼 보인다. 해결은 **모든 언어 SDK를 한 proto 릴리스에 묶어 동시 게시**하고, 버전 번호를 일치시키는 것. 레지스트리 모델([C])이 이 부분을 가장 잘 자동화한다.

### 17.8.4 소유권과 리뷰 프로세스

proto는 계약이므로 **소유자(owner)가 명확해야 한다.** 권장:

- 각 proto 패키지/디렉터리에 `CODEOWNERS`를 둬서 변경 시 해당 팀 리뷰를 강제.
- breaking 검사 CI는 필수 통과 게이트(17.4).
- **클라이언트 팀도 리뷰에 참여.** 계약은 양 당사자의 것이니, 서버 팀이 일방적으로 바꾸지 못하게.
- 큰 변경(새 메서드, 새 리소스)은 proto만의 design review를 거친 뒤 머지.

---

## 17.9 스키마 거버넌스: lint, 네이밍, 문서화, 민감 필드

거버넌스는 "수백 개 proto가 한 사람이 쓴 것처럼 일관되게 보이도록" 만드는 일이다.

### 17.9.1 lint로 스타일 강제

`buf lint`(또는 protolint)는 네이밍·구조 규칙을 자동 검사한다. 표준 규칙 셋이 강제하는 것들:

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

lint도 breaking과 마찬가지로 **CI 게이트**로 건다. 스타일 논쟁을 PR마다 사람이 하지 않게.

### 17.9.2 문서화 주석

proto의 주석은 단순 메모가 아니다. 많은 생성기가 **proto 주석을 생성 코드의 doc comment로, 또 API 문서로 전파**한다. 그래서 주석을 "공개 API 문서를 쓴다"는 자세로 작성한다.

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

`Output only`, `Required`, `Immutable` 같은 **동작 명세를 주석 관례로** 박아두면(AIP-203의 field behavior), 사람이 읽을 뿐 아니라 일부 도구는 `google.api.field_behavior` 애너테이션으로 기계 판독도 한다.

### 17.9.3 비공개/내부 필드와 보안 검토

같은 proto를 내부와 외부가 공유하면 **내부 전용 필드가 외부로 새지 않도록** 주의해야 한다. 두 접근:

- **버전/패키지 분리**: 외부 공개용 proto와 내부 전용 proto를 아예 다른 패키지로 분리. 가장 안전.
- **커스텀 옵션으로 표시**: `(acme.internal) = true` 같은 커스텀 필드 옵션을 달고, 외부 SDK 생성 시 그 필드를 제외하는 필터를 건다.

민감 데이터(개인정보, 비밀키 등) 보안 검토 관점:

- **민감 필드를 명시적으로 표시**하고(주석 또는 커스텀 옵션), 로깅 인터셉터([[08 - 메타데이터와 인터셉터]])가 그 필드를 자동 마스킹하게 한다. proto에 `[(acme.pii) = true]` 같은 옵션을 달고, 로깅 미들웨어가 메시지를 직렬화할 때 해당 필드를 `***`로 치환하는 식.
- **민감 데이터는 메타데이터 헤더가 아니라 메시지 본문**에 두는 게 일반적으로 낫다(헤더는 프록시 로그·트레이싱에 남기 쉬움). 단 본문도 로깅에서 샐 수 있으니 마스킹 필요.
- proto 리뷰 체크리스트에 "이 필드가 PII인가? 로깅에서 마스킹되는가? 외부 노출 대상인가?"를 포함.

---

## 17.10 테스트 전략

proto가 계약이므로, 테스트의 상당 부분은 "계약을 지키는가"를 검증하는 일이다.

### 17.10.1 in-process 서버(버퍼 기반 연결)로 빠른 통합 테스트

gRPC 통합 테스트에서 진짜 TCP 포트를 열면 느리고 포트 충돌이 난다. 대신 **메모리 내 버퍼로 연결을 잇는** 트랜스포트를 쓴다(Go의 `bufconn`, Java의 in-process transport 등). 클라와 서버가 같은 프로세스 안에서 실제 gRPC 스택(직렬화·인터셉터·스트리밍 전부)을 통과하되, 네트워크는 메모리 파이프로 대체된다.

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

이러면 실제 직렬화·인터셉터·에러 매핑까지 다 거치면서도 테스트가 밀리초 단위로 끝난다. 진짜 네트워크 동작([[04 - HTTP2 깊이 보기 - 전송 계층]]의 프레이밍, TLS 등)만 빠진다.

### 17.10.2 목/스텁

단위 테스트에서는 의존하는 다른 gRPC 서비스의 클라 스텁을 **목(mock)으로 대체**한다. 생성된 클라 인터페이스를 구현한 가짜 객체를 주입. 단, 목은 "내가 상상한 서버 동작"을 검증할 뿐 실제 계약과 어긋날 수 있으니, 이것만으로는 부족하다 → 계약 테스트로 보완.

### 17.10.3 계약 테스트(contract test)

목의 한계를 메우는 게 계약 테스트다. **클라가 기대하는 요청/응답 형태**(consumer 측 계약)와 **서버가 실제로 제공하는 형태**(provider 측)를 같은 명세로 양쪽에서 검증한다. proto가 이미 구조 계약이지만, "이 필드 조합일 때 이런 동작"이라는 행위 계약은 proto만으론 표현 안 된다. 계약 테스트는 그 간극을 메운다. provider 측 통합 테스트가 consumer가 의존하는 시나리오를 모두 커버하는지 확인하는 방식이 흔하다.

### 17.10.4 골든 와이어 테스트(golden wire test)

이게 호환성 회귀를 잡는 강력한 무기다. **"옛 버전이 직렬화한 실제 바이트"를 파일로 박제(golden file)해두고, 새 코드가 그걸 그대로 디코딩할 수 있는지** 매 빌드마다 검증한다. [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]의 와이어 포맷이 정말로 안 깨졌는지를 바이트 레벨에서 확인하는 것.

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

위 hex를 손으로 디코딩해보면 와이어 포맷이 보인다:

```text
0a 07 62 6f 6f 6b 73 2f 31   12 04 31 39 38 34
└┬┘ └┬┘ └──────┬──────────┘  └┬┘ └┬┘ └────┬───┘
 │   │         │              │   │       │
 │   │         "books/1"      │   │       "1984"
 │   len=7                    │   len=4
 │                            tag: (2<<3)|2 = 0x12 → 필드2(title), wire type 2
 tag: (1<<3)|2 = 0x0a → 필드1(name), wire type 2(length-delimited)
```

`0x0a` = `00001010` = `(1 << 3) | 2` → 필드 번호 1, 와이어 타입 2(length-delimited). 그 뒤 길이 `0x07`(7바이트), 이어 `books/1`의 ASCII. 다음 `0x12` = `(2 << 3) | 2` → 필드 번호 2, 와이어 타입 2, 길이 4, `1984`. 이렇게 박제된 바이트가 미래에도 같은 필드로 디코딩되면 와이어 호환은 살아 있다. 만약 누가 필드 번호를 바꿨다면 이 테스트가 즉시 빨갛게 된다.

### 17.10.5 로드 테스트

성능·백프레셔는 [[15 - 성능과 Flow Control 백프레셔]]에서 깊게 다뤘다. 여기선 한 가지만: proto 변경(특히 메시지가 커지거나 스트리밍 패턴이 바뀔 때)은 **성능 회귀를 부를 수 있으므로**, 큰 변경 후엔 `ghz` 같은 도구로 처리량·지연·메모리를 측정해 회귀를 확인한다. 메시지에 큰 `repeated`/`bytes` 필드를 추가하는 변경은 와이어 호환은 OK여도 페이로드 폭증·메모리 압박을 부를 수 있다.

---

## 17.11 마이그레이션 플레이북: 무사고로 진화시키기

규칙과 도구를 다 갖췄어도, **변경을 굴리는 순서**를 틀리면 사고가 난다. 핵심 통찰 하나가 모든 순서를 지배한다:

> **"읽는 쪽(reader)이 먼저 새 형식을 이해할 수 있어야, 쓰는 쪽(writer)이 새 형식을 보낼 수 있다."**

### 17.11.1 필드 추가 → 이중 기록 → 읽기 전환 → 구필드 reserved

가장 흔한 시나리오: `User.full_name`(단일 문자열)을 `given_name` + `family_name`(분리)으로 바꾸고 싶다. 한 번에 바꾸면 공존 기간에 누군가 깨진다. 그래서 **4단계 expand-and-contract(또는 parallel change)** 패턴으로 간다.

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

각 단계 사이에 **충분한 시간**(모든 클라가 따라잡을 만큼)을 둔다. 이게 expand-and-contract의 본질이다 — 확장은 빠르게, 수축은 느리게.

### 17.11.2 메서드 교체

`GetUser`의 시맨틱을 바꾸고 싶다면, 기존 메서드를 바꾸지 말고 **새 메서드를 추가**한다.

```text
1. GetUserV2 (또는 더 나은 이름) 추가. 서버 배포.
2. 클라들을 GetUserV2로 점진 이전.
3. GetUser 트래픽 모니터링([[14 - 관찰성과 디버깅 - Reflection grpcurl]]).
4. 트래픽 0 → GetUser에 option deprecated=true → 충분한 유예 → 제거.
```

메서드 이름·경로는 라우팅 계약이라 절대 바꾸지 않는다는 17.2.6의 규칙이 여기서 작동한다.

### 17.11.3 롤아웃 순서: 서버 먼저, 호환 확장으로

분산 배포에서 누구를 먼저 배포하느냐가 생사를 가른다. 규칙:

```text
새 필드/메서드를 "추가"하는 변경  →  서버(reader/handler) 먼저
   이유: 새 클라가 새 필드를 보내기 전에, 서버가 그걸 이해할 준비가 되어야 함.
   서버가 먼저 새 필드를 알면, 옛 클라(안 보냄)도 새 클라(보냄)도 모두 안전.

필드/메서드를 "제거"하는 변경    →  클라(writer/caller) 먼저
   이유: 클라가 그 필드/메서드를 더 이상 안 쓰게 된 뒤에야 서버에서 치울 수 있음.
```

한 문장으로: **"확장은 서버부터, 수축은 클라부터."** 그리고 롤링 배포 중 "구버전 파드 + 신버전 파드"가 공존하는 그 몇 분이 진짜 시험대이므로, 모든 변경은 그 공존 상태에서 양쪽이 무사한지를 기준으로 설계한다. 카나리(canary) 배포로 신버전을 소량 트래픽에 먼저 노출해 검증하면 더 안전하다.

### 17.11.4 페이지네이션·FieldMask 적용 마이그레이션 예시

이미 `repeated Book books`를 통째로 반환하던 `ListBooks`에 페이지네이션을 "나중에" 붙이는 경우를 보자. 다행히 이건 호환적으로 가능하다.

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

옛 클라는 `page_size`/`page_token`을 안 보내고 `next_page_token`을 무시한다. 그래서 **서버는 옛 클라(토큰 안 보냄)에게는 합리적 기본 동작**(예: 기본 페이지 크기로 첫 페이지만, 또는 호환을 위해 충분히 큰 기본값)을 줘야 한다. 이때 "옛 클라가 전체를 받던 동작"을 갑자기 잘라버리면 행위 호환이 깨질 수 있으니, 기본 page_size를 신중히 고르거나 옛 동작을 한동안 유지한다. 이게 "와이어 호환은 OK여도 행위 호환은 따로 챙겨야 한다"의 또 다른 예다.

---

## 17.12 종합: 안전한 진화 before/after 한눈에

마지막으로 지금까지의 규칙을 하나의 diff로 응축해보자. 같은 변경 의도를 **위험한 방식**과 **안전한 방식**으로 나란히 둔다.

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

이 diff를 CI에 올리면 `buf breaking`이 위험한 쪽은 줄줄이 빨간불, 안전한 쪽은 통과시킨다. 그게 거버넌스가 매일 하는 일이다.

---

## 17.13 실전 체크리스트

proto PR을 올리기 전 스스로에게 묻는 질문들:

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

---

## 핵심 요약

- **proto는 코드가 아니라 영구 조약이다.** 구버전과 신버전이 같은 와이어 위에서 공존하는 순간을 항상 통과해야 하며, 모든 호환 규칙은 "그때 양쪽이 무사한가"라는 단 하나의 질문으로 환원된다.
- **와이어에 실리는 건 이름이 아니라 번호다.** 그래서 필드 번호는 신성불가침, 이름 변경은 와이어엔 무해(코드/JSON엔 파괴), 삭제는 반드시 `reserved` 묘비, 타입 변경은 같은 호환군 안에서만 제한적으로 안전하다.
- **breaking에는 세 종류가 있다.** wire-breaking(조용하고 치명적), source-breaking(빌드 타임에 잡힘), json-breaking(REST/gRPC-Web 경로). 우선순위는 wire ≫ source > json이며, `buf breaking`을 CI 게이트로 걸어 사람의 실수를 기계가 막는다.
- **버전은 최후의 수단이다.** 필드·메서드·enum 값 추가로 대부분의 진화를 비파괴적으로 소화하고, 정말 호환 불가능할 때만 패키지 경로에 `v2`를 박아 멀티 버전을 공존시킨다. 제거는 deprecation → 트래픽 0 확인 → reserved 절차로.
- **리소스 지향 설계(AIP)가 진화를 쉽게 만든다.** 전용 Request 메시지, 불투명 토큰 페이지네이션, FieldMask 부분 업데이트, LRO, request_id 멱등성은 모두 "나중에 안 깨고 확장할 여지"를 미리 남겨두는 장치다.
- **마이그레이션의 황금률: 확장은 서버부터, 수축은 클라부터.** expand-and-contract(추가 → 이중 기록 → 읽기 전환 → reserved) 패턴으로 공존 기간을 안전하게 통과한다.
- **거버넌스는 시스템이다.** lint·breaking 게이트·골든 와이어 테스트·CODEOWNERS·민감 필드 마스킹을 CI/프로세스에 박아 두면, proto 사고의 대부분은 사람의 선의가 아니라 도구가 막아준다.

---

## 연결 노트

- [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]] — 호환 규칙이 왜 성립하는지의 바이트 레벨 원리(태그·와이어 타입·unknown field skip)
- [[02 - Protocol Buffers 1 - 문법과 타입 시스템]] — FieldMask/Timestamp 등 잘 알려진 타입, enum·oneof 문법
- [[06 - 코드 생성과 protoc 툴체인]] — buf/protoc 툴체인, 생성 코드 파이프라인
- [[09 - 에러 모델 - 상태 코드와 Rich Error]] — 도메인 에러 → 상태 코드 + Rich Error 매핑의 메커니즘
- [[13 - 안정성 - Retry Health Check Keepalive]] — 재시도와 멱등성·request_id가 맞물리는 지점
- [[08 - 메타데이터와 인터셉터]] — request_id 주입·민감 필드 마스킹 등 횡단 관심사
- [[15 - 성능과 Flow Control 백프레셔]] — 스키마 변경의 성능 영향과 로드 테스트
- [[16 - gRPC-Web과 REST 게이트웨이]] — json-breaking이 실제로 문제되는 REST/브라우저 경로
- [[14 - 관찰성과 디버깅 - Reflection grpcurl]] — deprecated 트래픽 모니터링과 디버깅
- [[07 - 채널 스텁 커넥션 생명주기]] — 서비스/메서드 이름이 `:path`로 라우팅되는 원리
- [[10 - Deadline 취소 타임아웃]] — LRO가 deadline 문제를 어떻게 우회하는가
- [[05 - 통신의 4가지 방식 - Unary와 Streaming]] — LRO 대안으로서의 서버 스트리밍 진행률 푸시
- [[04 - HTTP2 깊이 보기 - 전송 계층]] — in-process 테스트가 생략하는 실제 프레이밍 계층
