---
title: Protocol Buffers 1 - 문법과 타입 시스템
date: 2026-06-26
tags: [grpc, protobuf, idl, serialization, proto3, schema, well-known-types, 학습노트]
---

이 장이 답하는 질문:

- Protobuf는 왜 JSON이나 XML이 아니라 **스키마를 먼저** 요구하는가? 그게 무슨 이득을 주는가?
- proto3는 왜 `required`를 없애고 기본값을 도입했으며, 그 결과 어떤 "presence(존재성) 문제"가 생겼고 `optional`로 어떻게 되돌렸는가?
- 필드 번호 `1`과 `16`은 무엇이 다르고, 왜 자주 쓰는 필드는 작은 번호여야 하는가?
- 필드를 지우면 왜 `reserved`를 남겨야 하고, 안 남기면 무슨 사고가 나는가?
- `oneof`, `map`, `enum`, `Any`, `Timestamp` 같은 것들은 내부적으로 어떻게 표현되는가?

---

## 0. 들어가며: 계약서를 먼저 쓰는 사람들

[[01 - gRPC란 무엇인가]]에서 우리는 gRPC가 "원격 함수 호출"이라는 추상을 어떻게 제공하는지 봤다. 그런데 함수를 호출하려면 인자(argument)와 반환값(return value)이 있어야 한다. 네트워크 너머의 프로세스로 인자를 보내고 반환값을 받으려면, 그 데이터를 **바이트의 나열**로 바꿔야 한다(직렬화, serialization). 받는 쪽은 그 바이트를 다시 구조체로 복원해야 한다(역직렬화, deserialization).

여기서 근본적인 질문이 생긴다. **보내는 쪽과 받는 쪽이 "이 바이트는 이런 모양의 데이터다"라는 합의를 어떻게 공유하는가?**

세상에는 두 가지 큰 철학이 있다.

```text
 [철학 A] 스키마-온-리드 (schema-on-read)  — JSON, XML, MessagePack
 ─────────────────────────────────────────────────────────────
   바이트 안에 "필드 이름"과 "구조"가 통째로 들어있다.
   읽는 쪽이 런타임에 그 구조를 해석한다.
   { "userId": 42, "name": "Alice", "active": true }
   → 자기설명적(self-describing). 스키마 없이도 읽힌다.
   → 대신 키 문자열이 매번 따라다녀 뚱뚱하고, 타입은 추측해야 한다.

 [철학 B] 스키마-온-라이트 (schema-on-write) — Protocol Buffers, Avro, Thrift
 ─────────────────────────────────────────────────────────────
   바이트에는 "값"과 "필드 번호"만 들어간다. 이름도, 타입 설명도 없다.
   쓰기 전에 양쪽이 동일한 스키마(.proto)를 공유해야 한다.
   0x08 0x2A 0x12 0x05 ...  ← 이게 무슨 뜻인지는 스키마를 봐야 안다.
   → 작고 빠르고 타입 안전. 대신 스키마라는 "사전 합의"가 반드시 필요.
```

Protocol Buffers(이하 Protobuf)는 철저히 **B 진영**이다. 데이터를 쓰기 전에 그 데이터의 모양을 `.proto`라는 파일에 **계약서**로 먼저 적는다. 이 계약서가 곧 IDL(Interface Definition Language, 인터페이스 정의 언어)이다. 같은 계약서를 직렬화 규칙으로도 그대로 쓴다. Protobuf는 **"IDL"과 "직렬화 포맷"이라는 두 모자를 동시에 쓴 하나의 물건**이다.

이 두 역할이 한 몸이라는 점이 Protobuf의 정체성을 만든다.

- **IDL로서의 Protobuf**: "주문(Order)은 id, 상품 목록, 합계금액을 가진다", "OrderService는 CreateOrder라는 메서드를 가진다" 같은 **인터페이스의 모양**을 언어 중립적으로 기술한다. 이걸 가지고 Go/Java/Python 등 각 언어의 코드를 자동 생성한다([[06 - 코드 생성과 protoc 툴체인]]).
- **직렬화 포맷으로서의 Protobuf**: 그 인터페이스를 통해 오가는 실제 값을 작고 빠른 바이트로 인코딩한다(이 인코딩의 비트 단위 디테일은 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 손으로 디코딩하며 본다).

이 장은 **계약서를 쓰는 문법**과 **타입 시스템**, 그리고 그 규칙들이 "왜" 그렇게 설계됐는지에 집중한다. 와이어(wire) 위의 바이트가 실제로 어떻게 생겼는지는 다음 장으로 위임하되, 문법의 의미를 이해하기 위해 꼭 필요한 만큼은 여기서도 슬쩍슬쩍 보여줄 것이다.

### 스키마-온-라이트가 주는 진짜 가치

"스키마를 먼저 쓰는 게 왜 좋냐"는 질문에 대한 답은 단순히 "빠르다"가 아니다. 더 깊은 가치가 셋 있다.

1. **타입 안전성(type safety)이 컴파일 타임으로 당겨진다.** JSON은 `user.age`가 숫자인지 문자열인지 런타임에 까봐야 안다. Protobuf는 `int32 age = 3;`이라고 못 박혀 있어 코드 생성 단계에서 타입이 결정된다. 오타나 타입 불일치가 빌드에서 걸린다.

2. **하위호환성(backward compatibility)을 구조적으로 강제할 수 있다.** 이름이 아니라 **번호**로 필드를 식별하기 때문에, 필드 이름을 바꿔도 와이어는 멀쩡하다. 옛 클라이언트와 새 서버가 같은 바이트를 서로 다른 버전의 스키마로 읽어도 무너지지 않는다. 이건 분산 시스템에서 무중단 배포의 토대가 된다([[17 - 실전 - proto 관리와 하위호환성]]).

3. **계약이 곧 문서가 된다.** `.proto` 파일 하나면 프론트엔드, 백엔드, 모바일이 같은 진실을 본다. "이 API의 응답에 그 필드가 있던가요?"라는 슬랙 질문이 사라진다.

이 세 가지를 머리에 넣고 문법으로 들어가자.

---

## 1. proto2 vs proto3: 역사가 만든 철학의 흉터

Protobuf 문법에는 `proto2`와 `proto3`라는 두 방언(dialect)이 있다. `.proto` 파일 맨 위의 한 줄이 그걸 가른다.

```proto
syntax = "proto3";   // 또는 "proto2"
```

이 한 줄이 의외로 많은 것을 바꾼다. 둘의 차이를 "옛날 게 나쁘고 새 게 좋다"로 이해하면 핵심을 놓친다. 실제로는 **"필드의 존재성(presence)을 어떻게 다룰 것인가"라는 한 가지 철학적 질문을 두고 두 번 마음을 바꾼 역사**다.

### proto2의 세계관: 모든 필드는 "있거나/없거나"

proto2는 각 필드에 `required`, `optional`, `repeated`라는 **존재성 레이블**을 붙이도록 강제했다.

```proto
syntax = "proto2";

message User {
  required int64  id    = 1;   // 반드시 있어야 함
  optional string name  = 2;   // 있을 수도, 없을 수도
  repeated string email = 3;   // 0개 이상
}
```

핵심은 proto2에서는 **모든 단일 필드가 "설정됨(set)"인지 "설정 안 됨(unset)"인지를 항상 구분**한다는 점이다. `name`을 빈 문자열 `""`로 설정한 것과, 아예 설정하지 않은 것은 **다르다**. 코드에는 `hasName()` 같은 메서드가 생겨서 그 구분을 물어볼 수 있다. 이걸 **명시적 존재성(explicit presence)**이라고 부른다. 내부적으로 각 필드마다 "이 필드가 설정됐는가"를 기록하는 비트(has-bit)가 따라다닌다.

그런데 `required`가 재앙이었다.

### `required`는 왜 "백만 달러짜리 실수"가 됐나

`required`는 "이 필드는 무조건 있어야 하고, 없으면 메시지 파싱 자체가 실패한다"는 뜻이다. 직관적으로는 안전해 보인다. 그런데 분산 시스템의 시간 축 위에서는 **함정**이다.

```text
 시나리오: User 메시지에 required phone = 4 를 추가했다고 하자.

  [T0]  서버 v1: phone 없음. 옛 데이터 수억 건이 phone 없이 저장됨.
  [T1]  서버 v2: phone을 required로 추가.
  [T2]  v2가 T0 시절 데이터를 읽으려 한다 → "required phone 없음!" → 파싱 전체 실패.
        ↑ 단지 필드 하나 추가했을 뿐인데 기존 데이터가 통째로 못 읽힌다.
```

`required`는 **한 번 박으면 영원히 뺄 수 없는 못**이다. 빼는 순간, 그 필드를 채워서 보내던 옛 시스템과의 호환이 깨진다. 추가하는 순간, 그 필드가 없던 옛 데이터와의 호환이 깨진다. 즉 `required`는 스키마 진화(schema evolution)를 양방향으로 틀어막는다. 구글 내부에서 `required` 때문에 대규모 장애와 마이그레이션 지옥이 반복됐고, 결국 "required는 백만 달러짜리 실수"라는 격언이 남았다.

### proto3의 결단: required 제거, 기본값 도입

proto3는 이 문제를 칼같이 잘라냈다.

- **`required` 완전 제거.** 문법에서 사라졌다. 모든 필드는 본질적으로 선택적이다.
- **`optional` 레이블도 (초기엔) 제거.** "어차피 다 선택적인데 굳이 적을 필요 없다"는 논리.
- **대신 "기본값(default value)" 개념 도입.** 설정 안 된 스칼라 필드를 읽으면, 에러가 아니라 **타입별 0값**이 나온다.

```proto
syntax = "proto3";

message User {
  int64           id    = 1;   // 미설정이면 0
  string          name  = 2;   // 미설정이면 ""
  bool            active = 3;   // 미설정이면 false
  repeated string email = 4;   // 미설정이면 빈 리스트
}
```

기본값 표는 이렇다.

| 타입 부류 | proto3 기본값 |
|---|---|
| 숫자형(int32, int64, float, double, uint…) | `0` |
| bool | `false` |
| string | `""` (빈 문자열) |
| bytes | 빈 바이트열 |
| enum | 0번 값 |
| message(메시지 타입 필드) | "설정 안 됨"(null/None) — **기본값이 없음** |
| repeated | 빈 리스트 |

이 결정으로 `required` 지옥은 사라졌다. 어떤 필드가 빠져도 파싱은 절대 실패하지 않는다(받는 쪽은 그냥 기본값을 본다). 스키마 진화가 부드러워졌다.

그런데 이 "기본값"이라는 우아한 해결책이 **새로운 골칫거리**를 데려왔다. 바로 다음 절의 주제다.

---

## 2. presence 문제: "0"과 "설정 안 함"을 구분 못 하는 비극

proto3의 핵심 트릭은 이것이다(상세 인코딩은 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]):

> **스칼라 필드의 값이 기본값(0/""/false)과 같으면, 와이어에 아예 안 싣는다.**

왜? 작게 만들려고. 0인 필드를 굳이 바이트로 보낼 이유가 없으니, "없으면 0이다"라고 약속하고 생략한다. 이걸 **암묵적 존재성(implicit presence)**이라 부른다.

```text
 message Toggle { bool enabled = 1; }

  enabled = true  로 설정 → 와이어:  08 01       (필드1, 값 1)
  enabled = false 로 설정 → 와이어:  (아무것도 안 나감!)
  enabled 설정 안 함       → 와이어:  (아무것도 안 나감!)

  받는 쪽 입장: "08 01이 안 왔다" → enabled는 false다.
  그런데 보낸 쪽이 false로 명시한 건지, 깜빡한 건지 알 수가 없다.
```

여기서 비극이 발생한다. **"사용자가 명시적으로 0/false/빈문자열을 보낸 것"과 "그 필드를 아예 안 건드린 것"을 구분할 수 없다.** 둘 다 와이어에서 "필드 없음"으로 똑같이 보이기 때문이다.

이게 왜 실무에서 치명적인가? **부분 업데이트(partial update / PATCH)** 시나리오를 보자.

```text
 할일(Todo) 항목을 수정하는 UpdateTodo 요청.
   message Todo { string title = 1; bool done = 2; int32 priority = 3; }

 사용자가 "제목만 바꾸고 싶다"며 title만 채워 보냈다.
   → 와이어에는 title만 실린다. done과 priority는 기본값이라 생략.

 서버 입장: done=false, priority=0 이 왔다.
   "사용자가 done을 false로 바꾸고 priority를 0으로 바꾸라는 건가?
    아니면 그냥 안 건드린 건가?"  ← 구분 불가!

   순진하게 다 덮어쓰면 → 멀쩡하던 done=true가 false로 날아간다(데이터 손실).
```

이건 proto3 초기의 진짜 아픈 곳이었다. 회피책으로 wrapper 타입(아래 7절)이나 FieldMask를 썼지만 번거로웠다.

### `optional`의 부활: explicit presence를 되찾다

결국 protobuf 팀은 proto3에 `optional` 키워드를 **되살렸다**(실험적 지원은 2020년 protoc 3.12에서 `--experimental_allow_proto3_optional` 플래그로, 정식 기본 활성화는 2021년 protoc 3.15부터). 단어는 같지만 의미는 proto2 때와 통한다. "이 필드에 명시적 존재성을 다시 부여하라."

```proto
syntax = "proto3";

message Todo {
  string          title    = 1;   // 암묵적 존재성: ""과 미설정 구분 불가
  optional bool   done     = 2;   // 명시적 존재성: has_done()으로 구분 가능
  optional int32  priority = 3;   // 명시적 존재성
}
```

`optional`을 붙이면 생성 코드에 `has_done()` / `HasDone()` / `done IS NOT None` 류의 **존재성 질의 메서드**가 생긴다. 와이어에서도 `done = false`로 명시 설정하면 **기본값이어도 굳이 바이트로 실어 보낸다**(설정됐다는 사실 자체가 정보이므로).

```text
 message Todo { optional bool done = 2; }

  done = false 로 명시 설정 → 와이어:  10 00     (필드2, 값 0이지만 그래도 실림)
  done 설정 안 함           → 와이어:  (안 나감)  → has_done() == false
```

내부 구현 관점에서 `optional` 스칼라 필드는 사실 **멤버가 하나뿐인 익명 `oneof`로 컴파일**된다(6절의 oneof 참고). oneof는 "설정 여부"를 추적하므로, 그 메커니즘을 빌려 has-bit를 공짜로 얻는다. 똑똑한 재활용이다.

### 정리: presence 규칙의 큰 그림

| 필드 종류 | presence(존재성) | "미설정"과 "기본값" 구분 |
|---|---|---|
| proto3 일반 스칼라 (`int32 x = 1`) | 암묵적(implicit) | **불가능** |
| proto3 `optional` 스칼라 (`optional int32 x = 1`) | 명시적(explicit) | 가능 (`has_x()`) |
| proto2 `optional`/`required` 스칼라 | 명시적(explicit) | 가능 |
| **message 타입 필드** (`Address addr = 5`) | **항상 명시적** | **항상 가능** |
| `repeated` / `map` | presence 개념 없음 | "빈 컬렉션"만 존재 |
| `oneof` 멤버 | 명시적(어느 멤버가 셋인지 추적) | 가능 |

여기서 꼭 기억할 한 줄: **메시지 타입 필드는 언제나 존재성이 있다.** `Address addr = 5;`는 `optional`을 안 붙여도 "주소가 없음(null)"과 "비어있지만 존재하는 주소"를 구분한다. 왜냐하면 메시지에는 "기본값"이라는 게 없기 때문이다(빈 메시지 `{}`는 "있는 것"이지 "없는 것"이 아니다). 이 비대칭성 — 스칼라는 기본값이 있고 메시지는 없다 — 이 protobuf 타입 시스템의 가장 헷갈리면서도 중요한 포인트다.

> **설계 직관**: proto3는 "기본값으로 작게"를 택했다가 presence를 잃었고, `optional`로 presence를 부분적으로 되샀다. 이건 실패가 아니라 **트레이드오프의 재조정**이다. "대부분의 필드는 presence가 필요 없다(작게 가자), 일부 필드만 명시적으로 presence를 켜라"는 절충이 현재의 결론이다.

---

## 3. 스칼라 타입 전체 지도

이제 계약서에 적을 수 있는 **기본 재료**들을 본다. Protobuf의 스칼라 타입은 15가지. 처음 보면 "int가 왜 이렇게 많아?" 싶지만, 각각은 **인코딩 전략**이 다르고 그게 곧 용도를 가른다(비트 단위 디테일은 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]).

| Proto 타입 | 비트 | 부호 | 값 범위 | 인코딩 성향 | 언제 쓰나 |
|---|---|---|---|---|---|
| `double` | 64 | ○ | IEEE 754 배정밀도 | 고정 8바이트 | 일반 실수, 금액 외 |
| `float` | 32 | ○ | IEEE 754 단정밀도 | 고정 4바이트 | 정밀도 덜 중요한 실수 |
| `int32` | 32 | ○ | -2³¹ ~ 2³¹-1 | varint, **음수에 비효율** | 보통의 정수(음수 드묾) |
| `int64` | 64 | ○ | -2⁶³ ~ 2⁶³-1 | varint, **음수에 비효율** | 큰 정수, 타임스탬프(ms) |
| `uint32` | 32 | × | 0 ~ 2³²-1 | varint | 음수 없는 카운트/크기 |
| `uint64` | 64 | × | 0 ~ 2⁶⁴-1 | varint | 큰 음수 없는 수 |
| `sint32` | 32 | ○ | -2³¹ ~ 2³¹-1 | varint + **ZigZag**, 음수 효율적 | 음수가 흔한 32비트 정수 |
| `sint64` | 64 | ○ | -2⁶³ ~ 2⁶³-1 | varint + **ZigZag**, 음수 효율적 | 음수가 흔한 64비트 정수 |
| `fixed32` | 32 | × | 0 ~ 2³²-1 | 고정 4바이트 | 큰 값이 흔할 때(varint보다 작음) |
| `fixed64` | 64 | × | 0 ~ 2⁶⁴-1 | 고정 8바이트 | 해시, 큰 무부호 |
| `sfixed32` | 32 | ○ | -2³¹ ~ 2³¹-1 | 고정 4바이트 | 부호 있는 고정폭 |
| `sfixed64` | 64 | ○ | -2⁶³ ~ 2⁶³-1 | 고정 8바이트 | 부호 있는 고정폭 |
| `bool` | 1(논리) | — | true/false | varint 1바이트 | 플래그 |
| `string` | 가변 | — | **UTF-8** 텍스트 | length-prefixed | 사람이 읽는 텍스트 |
| `bytes` | 가변 | — | 임의 바이트열 | length-prefixed | 바이너리/암호화 데이터 |

이 표에서 **왜 int 계열이 셋(int/uint/sint)으로 갈라지는가**가 핵심 통찰이다. varint(가변 길이 정수) 인코딩은 작은 양수를 짧게 표현한다. 그런데 두 가지 함정이 있다.

```text
 [함정 1] int32/int64에 음수를 넣으면 varint가 비효율적이다.
   varint는 음수를 "2의 보수 64비트"로 보고 인코딩한다.
   그래서 -1 같은 작은 음수도 항상 10바이트(64비트 꽉 채움)를 먹는다.
   int32 x = -1;  → 10바이트. 끔찍하다.

 [함정 2] 그래서 ZigZag가 등장한다 → sint32/sint64.
   ZigZag는 음수를 작은 양수로 "지그재그" 매핑한다.
    0→0, -1→1, 1→2, -2→3, 2→4, ...
   덕분에 절댓값이 작은 음수가 짧게 인코딩된다.
   sint32 x = -1;  → 1바이트. 훌륭하다.
```

요약 규칙:
- **음수가 거의 안 나오는 정수** → `int32`/`int64` (가장 흔함).
- **음수가 자주 나오는 정수**(좌표, 온도 등) → `sint32`/`sint64`.
- **음수 자체가 불가능한 값**(개수, 크기, ID) → `uint32`/`uint64` (의미도 명확).
- **값이 거의 항상 크다**(예: 랜덤 ID, 해시) → `fixed32`/`fixed64` (varint가 오히려 손해니까 고정폭).

`string`과 `bytes`의 구분도 중요하다. 둘 다 와이어에서는 "길이 + 내용"으로 똑같이 인코딩되지만, **`string`은 반드시 유효한 UTF-8이어야 한다**(파서가 검증). 바이너리(이미지, 암호문, 압축데이터)를 `string`에 담으면 UTF-8 검증에서 깨질 수 있으니 반드시 `bytes`를 써라. "텍스트면 string, 그 외 전부 bytes"가 안전한 기준이다.

---

## 4. 필드 번호: 이름이 아니라 번호가 진짜 신분증

Protobuf 메시지의 각 필드에는 `= 1`, `= 2` 같은 **필드 번호(field number)**가 붙는다. 이게 단순한 순서표가 아니라 **와이어 위에서 필드를 식별하는 유일한 신분증**이다. 필드 이름은 와이어에 안 실린다. 오직 번호만 실린다.

```proto
message Product {
  string name  = 1;   // 와이어에서 이 필드는 "1번"
  int64  price = 2;   // 와이어에서 이 필드는 "2번"
}
```

이 사실에서 하위호환의 마법이 나온다.

```text
 name을 product_name으로 이름만 바꿔도 와이어는 그대로 "1번"이다.
 → 옛 클라이언트가 보낸 "1번 = Coffee" 바이트를, 새 서버가
   product_name으로 멀쩡히 읽는다. 이름 변경은 와이어 호환에 무해.

 반대로, 번호를 바꾸면(1→5) 같은 데이터가 다른 필드로 해석된다.
 → 절대 금지. 번호는 한번 정하면 영원히 고정.
```

### 번호의 범위와 예약 구역

필드 번호에는 사양이 정한 제약이 있다.

```text
 유효 범위:  1  ~  536,870,911  (= 2^29 - 1)
 금지 구역:  19,000 ~ 19,999    (protobuf 내부 구현 예약 — 쓰면 컴파일 에러)
 그리고 0은 사용 불가(필드 번호는 1부터).
```

19000~19999는 protobuf 라이브러리 자체가 내부용으로 찜해둔 구역이라 사용자가 못 쓴다. 실수로 적으면 protoc가 에러를 낸다.

### 왜 1~15가 "황금 번호"인가

필드 번호는 와이어에서 **태그(tag)**라는 1개 이상의 바이트로 인코딩된다. 태그 = `(필드번호 << 3) | 와이어타입`. 번호와 와이어 타입(3비트)을 한 정수에 합쳐 varint로 적는다(자세히는 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]).

varint는 7비트씩 끊어 담는다. 그래서:

```text
 태그 정수 = (필드번호 << 3) | 와이어타입
 한 varint 바이트는 값 비트 7개를 담는다.

 필드번호 1~15  → 태그 정수가 0~127 범위 → varint 1바이트!
 필드번호 16~2047 → 태그 정수가 더 큼 → varint 2바이트
 필드번호 2048~ ... → 3바이트 ...

 손으로 확인:  필드번호 15, 와이어타입 0(varint)
   (15 << 3) | 0 = 120 = 0x78  → 1바이트.  OK
 필드번호 16, 와이어타입 0
   (16 << 3) | 0 = 128 = 0x80  → 128은 7비트를 넘음 → 2바이트.
```

**1~15번 필드는 태그가 1바이트, 16번부터는 2바이트**다. 메시지가 초당 수백만 건 오가는 상황에서 모든 인스턴스마다 1바이트씩 차이가 나면 누적 트래픽이 크게 갈린다. 그래서 공식 가이드는 이렇게 권한다:

> **자주 등장하는 필드(특히 `repeated` 안에서 반복되는 필드, 핫패스의 필드)에는 1~15번을 배정하라. 16번 이상은 드물게 쓰는 필드에 양보하라. 그리고 미래 확장을 위해 1~15 중 몇 개는 비워둘 수도 있다.**

이건 "성능을 위해 번호를 아껴 쓰는" 흔치 않은 종류의 마이크로 최적화다. 작은 메시지를 대량으로 주고받을수록 효과가 크다.

---

## 5. reserved: 지운 필드의 무덤을 표시하라

분산 시스템에서는 한번 배포된 스키마가 어딘가에서 영원히 살아있다. 옛 데이터, 옛 클라이언트, 캐시된 메시지… 그래서 **필드를 지우는 행위**가 위험하다. Protobuf는 이 위험을 `reserved`라는 장치로 막는다.

문제 상황을 먼저 보자.

```text
 v1:  message User { string name = 1; string nickname = 2; int32 age = 3; }
 어느 날 nickname(2번)을 더 안 쓰기로 하고 그냥 지웠다.
 v2:  message User { string name = 1;                      int32 age = 3; }

 몇 달 뒤, 누군가 새 필드 country를 추가하며 무심코 번호 2를 재사용:
 v3:  message User { string name = 1; string country = 2;  int32 age = 3; }

 사고:  v1 시절 저장된 데이터에는 "2번 = 어떤닉네임" 바이트가 있다.
        v3가 그걸 읽으면 "2번 = country" 로 해석한다.
        → 옛 닉네임 문자열이 country 필드로 둔갑한다(데이터 오염).
```

번호 2를 재사용한 게 화근이다. 이걸 막으려면 **지운 번호(와 이름)를 묘비처럼 예약**해 둬야 한다.

```proto
message User {
  reserved 2;                 // 번호 2는 영원히 재사용 금지
  reserved "nickname";        // 이름 nickname도 재사용 금지
  // (범위도 가능)  reserved 2, 5 to 9, 100 to max;
  // (이름 여러 개) reserved "foo", "bar";

  string name = 1;
  int32  age  = 3;
}
```

이제 누군가 번호 2나 이름 nickname을 다시 쓰려 하면 **protoc가 컴파일 에러로 막아준다**. 사람의 기억력에 의존하지 않고 도구가 강제하는 안전장치다.

규칙 정리:
- **필드를 삭제할 때는 반드시 그 번호를 `reserved`로 남긴다.** 가능하면 이름도 함께.
- 번호와 이름은 **같은 `reserved` 문장에 섞어 쓸 수 없다.** 번호용/이름용을 따로 적는다(`reserved 2;`와 `reserved "nickname";`을 분리).
- `to max`로 "이 번호 이상 전부 예약"도 가능.
- enum 값에도 동일한 `reserved`가 있다(6절 참고).

> 이 규칙이 [[17 - 실전 - proto 관리와 하위호환성]]의 핵심 토대다. "필드는 지우되 번호는 무덤으로 남긴다"가 protobuf 진화의 제1계명이다.

---

## 6. 복합 타입: 스칼라를 넘어선 구조

스칼라만으로는 현실의 데이터를 못 담는다. Protobuf는 다섯 가지 복합 구조를 제공한다: `enum`, 중첩 메시지(nested message), `oneof`, `map`, `repeated`. 하나씩, 특히 **내부 표현**에 집중해서 본다.

### 6.1 enum: 0번 값은 신성하다

열거형은 정해진 후보 중 하나를 고르는 타입이다.

```proto
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;   // proto3에서 0번은 필수
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_PAID        = 2;
  ORDER_STATUS_SHIPPED     = 3;
  ORDER_STATUS_CANCELLED   = 4;
}
```

proto3 enum의 **절대 규칙**: **맨 처음 나열한 값(첫 번째 enumerator)이 반드시 `0`이어야 하고**, 그게 **기본값**이 된다(단순히 "어딘가에 0이 있으면 된다"가 아니라 *첫 줄*이 0이어야 한다 — 아니면 protoc가 컴파일 에러를 낸다). 왜 강제할까? 3절에서 봤듯 proto3 스칼라(enum 포함)는 기본값이 와이어에서 생략된다. enum 필드가 미설정이면 받는 쪽은 "0번 값"을 본다. 만약 0번이 정의돼 있지 않으면 "정의되지 않은 값"이 기본값이 되는 모순이 생긴다. 그래서 0번은 항상 존재해야 하고, 관례상 `_UNSPECIFIED`(또는 `_UNKNOWN`)라는 "아직 정해지지 않음" 의미를 부여한다.

이건 단순 규칙이 아니라 **하위호환 설계**다. "0 = 미지정"으로 두면, 옛 클라이언트가 enum을 안 채워 보내도 새 서버가 "아, 이 값은 모르겠다는 뜻이군"으로 안전하게 해석한다. `PENDING`을 0번으로 두면 "안 채운 것"과 "PENDING"이 섞여버린다.

**alias(별칭)**: 같은 숫자에 두 이름을 붙이고 싶으면 `allow_alias`를 켠다.

```proto
enum Direction {
  option allow_alias = true;
  DIRECTION_UNSPECIFIED = 0;
  DIRECTION_NORTH       = 1;
  DIRECTION_UP          = 1;   // NORTH의 별칭 (allow_alias 없으면 컴파일 에러)
}
```

**열린 enum vs 닫힌 enum (open/closed)**: 이 미지값 처리 방식이 proto2와 proto3의 또 하나의 갈림길이다. proto3 enum은 **열린(open) enum**이다 — 정의에 없는 숫자가 와도 거부하지 않고 그대로 받아들인다. 반면 proto2 enum은 **닫힌(closed) enum**이라, 모르는 숫자는 enum 필드가 아니라 "알 수 없는 필드(unknown fields)" 영역으로 흘려보낸다(필드를 다시 읽으면 기본값처럼 보인다). 이 차이 때문에 같은 숫자를 보내도 proto2 수신자와 proto3 수신자의 동작이 달라질 수 있다.

**미지의 enum 값 처리**: 새 서버가 `ORDER_STATUS_REFUNDED = 5`를 추가했는데 옛 클라이언트는 5를 모른다. proto3(열린 enum)에서는 옛 클라이언트가 그 값을 버리지 않고 **정수 5 그대로 보존**한다(언어별로 `UNRECOGNIZED`/raw int 등으로 표현). 그래서 메시지를 받아 다시 직렬화해 넘겨도 5가 살아남는다 — 중간 노드가 모르는 새 enum 값을 조용히 0으로 깎아먹지 않는다는 뜻이라, 분산 진화에 중요하다. 코드에서 미지값을 만났을 때 `default` 분기로 안전하게 처리하도록 `switch`/`when`을 작성해야 한다. enum도 메시지처럼 `reserved`로 지운 **값(번호)과 이름**을 따로 예약해 재사용 사고를 막을 수 있다.

### 6.2 nested message: 메시지 안의 메시지

메시지는 다른 메시지를 필드로 가질 수 있고, 메시지 정의 안에 메시지를 **중첩 선언**할 수도 있다.

```proto
message Order {
  message LineItem {              // Order 안에서만 의미 있는 타입
    string product_id = 1;
    int32  quantity   = 2;
    int64  unit_price = 3;
  }

  string            order_id = 1;
  repeated LineItem items    = 2;   // 중첩 타입을 반복 필드로
}
```

중첩은 **이름 공간(namespace)을 정리**하는 수단이다. `LineItem`이 `Order` 밖에서 쓰일 일이 없다면 안에 넣어 `Order.LineItem`으로 한정한다. 와이어 인코딩 관점에서 중첩이든 아니든 메시지 필드는 동일하게 "length-prefixed 바이트 블록"으로 들어간다 — **메시지를 필드에 담는다는 건, 그 메시지를 통째로 직렬화한 바이트를 길이와 함께 박는 것**이다(상세는 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]). 그래서 메시지 안에 메시지를 무한히 중첩해도 재귀적으로 같은 규칙이 적용된다.

### 6.3 oneof: "이것 중 하나만"의 메모리 절약과 의미 표현

`oneof`는 여러 필드 중 **동시에 최대 하나만** 설정될 수 있음을 표현한다.

```proto
message Notification {
  string title = 1;

  oneof payload {            // 아래 셋 중 하나만 설정됨
    string text_body  = 2;
    bytes  image_data = 3;
    string link_url   = 4;
  }
}
```

`oneof`의 두 얼굴:

1. **의미(semantics)**: "이메일이거나, 이미지이거나, 링크이거나 — 셋이 동시에 올 수 없다"는 **상호배타성**을 타입으로 못 박는다. C/C++의 union, Rust의 enum(태그드 유니언), Kotlin의 sealed class에 대응한다. 받는 쪽은 "어느 멤버가 설정됐는지"를 묻는 코드(`switch (payloadCase)` 류)를 얻는다.

2. **메모리(memory)**: 생성 코드에서 oneof 멤버들은 **같은 저장 공간을 공유**한다(union처럼). 셋 다 별도 필드로 두면 셋만큼 메모리를 먹지만, oneof는 하나 분량 + "지금 누가 설정됐나" 태그만 쓴다.

내부 동작의 핵심: oneof는 **존재성을 추적**한다. "지금 2번/3번/4번 중 누가 셋인가, 아무도 안 셋인가"를 항상 안다. 그래서 3절에서 말했듯 **proto3의 `optional` 스칼라는 멤버가 하나뿐인 oneof로 컴파일**된다. oneof의 "누가 셋인지 추적" 능력을 빌려 has-bit를 공짜로 구현한다.

주의: oneof 멤버에는 `repeated`를 쓸 수 없고, oneof 멤버는 `optional`을 또 붙일 수 없다. 또 새 멤버를 oneof에 **나중에 추가**하는 건 와이어 호환은 되지만, 옛 코드가 그 멤버를 모를 뿐이다.

### 6.4 map: 사실은 repeated 메시지의 가면

연관 배열(키-값 쌍)은 `map`으로 쓴다.

```proto
message Config {
  map<string, string> labels   = 1;   // 문자열→문자열
  map<int64, Order>   by_index = 2;   // int64→메시지
}
```

규칙:
- 키 타입은 **정수형/bool/string만** 가능(float/double/bytes/message/enum 불가). 값 타입은 거의 아무거나(다른 map 제외).
- map은 **순서가 보장되지 않는다**(직렬화 순서는 구현에 따라 다를 수 있음).

여기서 가장 중요한 통찰: **map은 와이어 포맷에 독립적으로 존재하지 않는다.** 컴파일러가 map을 다음과 같은 **repeated 메시지로 변환**한다.

```proto
// map<string, string> labels = 1;  은 내부적으로 아래와 동등하다:
message LabelsEntry {
  string key   = 1;
  string value = 2;
}
repeated LabelsEntry labels = 1;
```

map의 각 엔트리는 `{key, value}`를 담은 작은 메시지이고, 전체 map은 그 엔트리들의 반복이다. 이 사실은 실용적 결과를 낳는다:

```text
 - map은 "그냥 repeated 엔트리"이므로, 와이어에 같은 key가 두 번 와도
   파서는 거부하지 않는다. 보통 "마지막 값이 이긴다(last-one-wins)".
 - map 필드 자체는 presence가 없다. "빈 map"만 있고 "없는 map"은 같다.
 - map↔repeated 변환이 와이어 호환된다. map<K,V> = N 을
   repeated KVEntry = N 으로 바꿔도(엔트리 구조만 맞으면) 데이터가 안 깨진다.
```

이 "map = repeated entry message"라는 정체를 알면 map의 모든 제약(키 타입 제한, 순서 없음, presence 없음)이 자연스럽게 이해된다.

### 6.5 repeated: 0개 이상의 반복

리스트/배열은 `repeated`다.

```proto
message Playlist {
  repeated string song_ids = 1;   // 0개 이상의 문자열
  repeated int32  ratings  = 2;   // 0개 이상의 정수
}
```

`repeated`는 presence 개념이 없다 — "비어있음"만 있고 "없음"은 같은 뜻이다(빈 리스트). 와이어에서 같은 필드 번호가 여러 번 반복되는 형태로 표현되는데, **스칼라 숫자형 repeated는 기본적으로 "packed"로 인코딩**된다(proto3 기본). packed란 태그를 매번 반복하지 않고 값들을 한 블록에 몰아 담는 최적화다. 1만 개 정수를 보낼 때 태그 1만 번 반복 vs 태그 1번 + 값 블록 — 후자가 훨씬 작다. 이 packed의 바이트 디테일은 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 본다.

복합 타입을 한 장으로 정리하면:

| 구조 | 표현하는 것 | 내부 표현 / 특이점 |
|---|---|---|
| `enum` | 닫힌 후보 집합 | 0번 필수·기본값, 미지값은 정수 보존 |
| nested message | 구조의 중첩 | length-prefixed 바이트 블록(재귀적) |
| `oneof` | 상호배타 선택 | union 메모리, presence 추적, `optional`의 구현 기반 |
| `map<K,V>` | 키-값 매핑 | **repeated {key,value} 메시지로 컴파일** |
| `repeated` | 0개 이상 리스트 | 숫자형은 packed 인코딩, presence 없음 |

---

## 7. 패키지·임포트·파일 구조

`.proto`도 코드처럼 여러 파일로 쪼개고 서로 의존한다. 그 메커니즘이 `package`와 `import`다.

### package: 이름 충돌을 막는 네임스페이스

```proto
syntax = "proto3";

package shop.order.v1;   // 이 파일의 모든 타입은 이 네임스페이스에 속함

message Order { /* ... */ }   // 완전한 이름: shop.order.v1.Order
```

`package`는 메시지/enum/서비스의 **완전 한정 이름(fully-qualified name)**의 접두사가 된다. 서로 다른 팀이 둘 다 `Order`라는 메시지를 만들어도 패키지가 다르면 충돌하지 않는다. 관례로 도메인 역순 + 버전(`com.example.shop.v1` 또는 `shop.order.v1`)을 쓴다. 버전을 패키지에 넣는 것(`...v1`)은 나중에 호환 깨지는 대규모 변경이 필요할 때 `v2` 패키지를 통째로 새로 만들기 위한 탈출구다([[17 - 실전 - proto 관리와 하위호환성]]).

생성 코드의 언어별 네임스페이스/패키지는 별도 옵션으로 조정한다.

```proto
option go_package   = "github.com/example/shop/gen/orderv1;orderv1";
option java_package = "com.example.shop.order.v1";
option java_multiple_files = true;
```

이 옵션들은 [[06 - 코드 생성과 protoc 툴체인]]에서 자세히 다룬다.

### import: 다른 파일의 타입 끌어오기

```proto
// file: order.proto
syntax = "proto3";
package shop.order.v1;

import "shop/common/money.proto";          // 다른 파일의 타입 사용
import "google/protobuf/timestamp.proto";  // Well-Known Type

message Order {
  string                    order_id   = 1;
  shop.common.Money         total      = 2;   // money.proto의 타입
  google.protobuf.Timestamp created_at = 3;   // WKT
}
```

import 경로는 protoc의 import 경로(`-I`/`--proto_path`)를 기준으로 한 **상대 경로**다(파일시스템 절대경로가 아니다). 그래서 빌드 시 "루트 디렉터리들"을 protoc에 알려주고, 그 안에서의 경로로 import한다.

### public import: 의존성 재노출(re-export)

평범한 `import`는 그 파일에 직접 정의된 타입만 쓸 수 있게 한다. `import public`은 한 단계 더 나아가, **A가 B를 public import하면, A를 import한 C가 B의 타입까지 쓸 수 있게** 한다(전이적 재노출).

```text
 base.proto      :  message Money { ... }
 facade.proto    :  import public "base.proto";   // Money를 재노출
 app.proto       :  import "facade.proto";
                    → app.proto에서 Money를 직접 import 안 해도 쓸 수 있다.
```

용도: 파일을 리팩터링으로 쪼개거나 옮길 때, 옛 경로를 import하던 코드를 안 깨뜨리려고 옛 파일이 새 파일을 `import public`으로 가리키게 한다. 일종의 **하위호환 포워딩**이다. 남발하면 의존 관계가 흐려지니 꼭 필요할 때만 쓴다.

파일 구조의 일반 규칙: 한 파일에 한 주제(보통 한 서비스 + 그 메시지들, 혹은 공용 타입 묶음). 파일명은 `snake_case.proto`. import는 알파벳순 정렬이 관례다.

---

## 8. Well-Known Types: 구글이 미리 만들어 둔 표준 부품

날짜/시간, 동적 타입, 빈 메시지 같은 건 누구나 필요하다. 매번 각자 정의하면 호환이 안 된다. 그래서 protobuf는 `google.protobuf` 패키지 아래 **Well-Known Types(WKT)** 라는 표준 메시지들을 제공한다. import해서 그대로 쓴다. 핵심 멤버를 보자.

### Timestamp / Duration: 시간의 표준형

```proto
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";

message Event {
  google.protobuf.Timestamp occurred_at = 1;  // 절대 시각(UTC 기준)
  google.protobuf.Duration  ttl         = 2;  // 시간 간격
}
```

- **`Timestamp`**: `seconds`(Unix epoch 1970-01-01 UTC 기준 초)와 `nanos`(나노초, 0~999,999,999)로 **절대 시점**을 표현. "언제"를 나타낼 때.
- **`Duration`**: 같은 `seconds`+`nanos` 구조지만 **시간의 길이**(부호 있음, 음수 가능). "얼마 동안"을 나타낼 때.

왜 그냥 `int64 millis`를 안 쓰고 이걸 쓰나? (1) 의미가 명확해지고(타입만 봐도 "이건 시각/간격이구나"), (2) 나노초 정밀도를 표준 방식으로 보장하며, (3) JSON 매핑이 RFC 3339 문자열(`"2026-06-26T07:30:00Z"`)로 깔끔하게 떨어지고, (4) 모든 언어의 생성 코드가 네이티브 시간 타입과의 변환 헬퍼를 제공한다.

### Any: 타입을 런타임에 담는 봉투

`Any`는 **임의의 메시지를 타입 정보와 함께 통째로 담는** 컨테이너다.

```proto
import "google/protobuf/any.proto";

message Envelope {
  string                event_id = 1;
  google.protobuf.Any   payload  = 2;   // 어떤 메시지든 담을 수 있음
}
```

`Any`의 정체는 단 두 필드다.

```proto
message Any {
  string type_url = 1;   // 예: "type.googleapis.com/shop.order.v1.Order"
  bytes  value    = 2;   // 그 타입으로 직렬화된 바이트
}
```

즉 `Any`는 "이 바이트(`value`)는 `type_url`이 가리키는 타입이다"라는 **자기설명적 봉투**다. 보낼 때 메시지를 직렬화해 `value`에 넣고 타입 식별자를 `type_url`에 적는다. 받는 쪽은 `type_url`을 보고 "아, 이건 Order구나" 하고 `value`를 그 타입으로 unpack한다. 코드에는 `pack`/`unpack`(또는 `Is<T>()`/`UnpackTo`) 헬퍼가 생긴다.

용도: 이벤트 버스의 다형적 페이로드, 플러그인 시스템, "타입이 런타임에 결정되는" 경우. 대가: 컴파일 타임 타입 안전성을 일부 포기한다(받는 쪽이 type_url을 보고 분기해야 함). 가능하면 `oneof`로 후보가 닫혀 있게 하고, 정말 열린 확장이 필요할 때만 `Any`를 쓴다. (참고: gRPC의 풍부한 에러 모델도 `Any`로 디테일을 싣는다 — [[09 - 에러 모델 - 상태 코드와 Rich Error]].)

### Struct / Value / ListValue: protobuf 안의 JSON

스키마 없는 동적 데이터(임의 JSON)를 담아야 할 때 쓴다.

```proto
import "google/protobuf/struct.proto";

message Webhook {
  google.protobuf.Struct extra = 1;   // 임의의 JSON 객체
}
```

- **`Value`**: null/숫자/문자열/bool/Struct/ListValue 중 하나를 담는 oneof. JSON의 "어떤 값"에 대응.
- **`Struct`**: `map<string, Value>` — JSON 객체.
- **`ListValue`**: `repeated Value` — JSON 배열.

이건 "protobuf로 JSON을 표현하는 표준 방법"이다. 스키마가 정말로 미정인 영역(서드파티 메타데이터 등)에만 쓰고, 알 수 있는 구조는 정식 메시지로 정의하는 게 낫다.

### FieldMask: 부분 업데이트의 정석

2절의 presence 문제를 정공법으로 푸는 도구다.

```proto
import "google/protobuf/field_mask.proto";

message UpdateUserRequest {
  User                       user        = 1;
  google.protobuf.FieldMask  update_mask = 2;   // 어떤 필드를 바꿀지 명시
}
```

`FieldMask`는 `repeated string paths` 하나뿐이다. `paths = ["display_name", "email"]`처럼 **"이 필드들만 갱신하라"**고 명시한다. 서버는 마스크에 든 경로만 덮어쓴다. "값이 기본값인지 미설정인지" 구분에 의존하지 않고, "무엇을 바꿀지"를 별도 채널로 받는 깔끔한 해법이다. PATCH류 API의 표준 패턴.

### Empty: 진짜 아무것도 없음

```proto
import "google/protobuf/empty.proto";

service PingService {
  rpc Ping(google.protobuf.Empty) returns (google.protobuf.Empty);
}
```

`Empty`는 필드가 0개인 메시지다. "인자/반환이 없는 RPC"를 표현한다. 다만 실무에서는 처음부터 `Empty` 대신 빈 커스텀 메시지(`message PingResponse {}`)를 두길 권하기도 한다 — 나중에 필드를 추가할 여지를 남기려고. (`Empty`는 영원히 빈 채로 못 박힌다.)

### Wrapper 타입: presence를 위한 박스

`Int32Value`, `Int64Value`, `UInt32Value`, `BoolValue`, `StringValue`, `DoubleValue`, `FloatValue`, `BytesValue` 등. 이들은 **스칼라 하나를 감싼 메시지**다.

```proto
import "google/protobuf/wrappers.proto";

message Patch {
  google.protobuf.Int32Value  retry_count = 1;   // "있음/없음" 구분 가능
  google.protobuf.BoolValue   enabled     = 2;
}
```

존재 이유는 명확하다. 2절의 presence 문제. **스칼라는 기본값과 미설정을 구분 못 하지만, 메시지는 항상 presence가 있다.** `Int32Value`는 int32를 메시지로 감싸서, "값이 0인 메시지"와 "메시지 자체가 없음(null)"을 구분하게 만든다.

```text
 int32 x = 1;                      → 0과 미설정 구분 불가
 google.protobuf.Int32Value x = 1; → "x={value:0}"(0임)과 "x=null"(미설정) 구분 가능
```

이건 proto3에 `optional`이 부활하기 전의 주된 회피책이었다. 이제는 `optional int32 x = 1;`이 더 가볍고 자연스러운 정답이라 wrapper의 필요성이 줄었지만, 여전히 JSON 연동이나 일부 API 스타일에서 보인다. **신규 코드라면 wrapper보다 `optional`을 우선 고려하라.** (wrapper는 메시지라서 더 무겁다.)

WKT 한눈 정리:

| WKT | 한 줄 정의 | 주 용도 |
|---|---|---|
| `Timestamp` | 절대 시각(epoch초+나노) | "언제 발생했나" |
| `Duration` | 시간 간격(부호 있음) | "얼마 동안/타임아웃" |
| `Any` | 타입URL+직렬바이트 | 다형 페이로드 |
| `Struct`/`Value`/`ListValue` | protobuf 속의 JSON | 스키마리스 데이터 |
| `FieldMask` | 필드 경로 목록 | 부분 업데이트(PATCH) |
| `Empty` | 빈 메시지 | 인자/반환 없는 RPC |
| `Int32Value` 등 wrapper | 스칼라를 감싼 박스 | presence 확보(레거시) |

---

## 9. service와 rpc: 계약서에 "동작"을 적다

여기까지가 "데이터의 모양(메시지)"이었다면, `service`는 "그 데이터로 무엇을 할 수 있나(메서드)"를 적는다. gRPC의 IDL 다움이 가장 잘 드러나는 부분이다.

```proto
service OrderService {
  // Unary: 요청 1 → 응답 1
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);

  // Server streaming: 요청 1 → 응답 N(스트림)
  rpc WatchOrder(WatchOrderRequest) returns (stream OrderEvent);

  // Client streaming: 요청 N(스트림) → 응답 1
  rpc UploadItems(stream LineItem) returns (UploadSummary);

  // Bidirectional streaming: 요청 N ↔ 응답 N
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

`rpc` 정의의 문법은 단순하다: `rpc 메서드명(요청타입) returns (응답타입);`. 요청/응답 앞에 `stream` 키워드가 붙느냐 안 붙느냐로 **4가지 통신 방식**이 정의된다.

| 방식 | 시그니처 형태 | 요청 | 응답 |
|---|---|---|---|
| Unary | `rpc M(Req) returns (Res)` | 1 | 1 |
| Server streaming | `rpc M(Req) returns (stream Res)` | 1 | N |
| Client streaming | `rpc M(stream Req) returns (Res)` | N | 1 |
| Bidirectional streaming | `rpc M(stream Req) returns (stream Res)` | N | N |

여기서 IDL 차원의 중요한 규칙 하나: **요청과 응답은 반드시 (단일) 메시지 타입이어야 한다.** 스칼라를 직접 받거나 반환할 수 없다. `rpc GetCount() returns (int32)` 같은 건 불가능하다. 항상 메시지로 감싼다. 왜? 그래야 나중에 필드를 추가하며 진화할 수 있기 때문이다. "지금은 카운트 하나만 필요해도, 메시지로 감싸 두면 내일 필드를 더할 수 있다." 이건 protobuf의 일관된 철학 — **모든 경계는 확장 가능한 메시지로** — 의 연장이다.

각 stream 조합이 와이어/HTTP2 위에서 실제로 어떻게 프레임을 주고받는지(반이중/전이중, half-close 등)는 [[05 - 통신의 4가지 방식 - Unary와 Streaming]]에서 다룬다. 여기서는 "문법으로 어떻게 선언하는가"까지가 범위다. 메서드 호출을 가로채는 인터셉터나 메타데이터는 [[08 - 메타데이터와 인터셉터]]에서 본다.

---

## 10. JSON 매핑: protobuf의 또 다른 얼굴

Protobuf는 바이너리가 본체지만, **표준 JSON 표현(canonical JSON)**도 사양으로 정의돼 있다. gRPC-Web, REST 게이트웨이, 디버깅, 로깅에서 이 매핑이 쓰인다([[16 - gRPC-Web과 REST 게이트웨이]]). 규칙을 알아두면 "왜 JSON 응답의 필드명이 내가 적은 거랑 다르지?" 같은 혼란을 피한다.

핵심 규칙:

1. **필드명은 lowerCamelCase로 변환된다.** proto에서 `snake_case`로 적은 필드(`order_id`)는 JSON에서 `orderId`가 된다. (원래 proto 이름도 받아주긴 하지만 출력은 camelCase가 기본.)

2. **64비트 정수(int64/uint64/fixed64/…)는 JSON에서 문자열로 표현된다.** `"price": "9007199254740993"`. 왜? JavaScript의 `number`(IEEE 754 double)는 2⁵³ 너머 정수를 정확히 표현 못 한다. 큰 64비트 값을 숫자로 내보내면 조용히 정밀도가 깨진다. 그래서 **사양이 문자열로 강제**한다. 이건 자주 사람을 놀라게 하는 규칙이니 꼭 기억하자.

3. **enum은 이름(문자열)으로 표현된다.** `"status": "ORDER_STATUS_PAID"`. 숫자도 받아들이지만 출력 기본은 이름.

4. **`bytes`는 표준 base64 문자열**로 인코딩된다.

5. **WKT는 특별 매핑**을 가진다. `Timestamp` → RFC 3339 문자열(`"2026-06-26T07:30:00Z"`), `Duration` → `"3.5s"`, `FieldMask` → 콤마로 이은 camelCase 경로(`"displayName,email"`), wrapper 타입 → 감싼 값 그대로, `Struct`/`Value` → 그에 대응하는 JSON, `Any` → `{"@type": "...", ...}` 형태.

6. **기본값 필드는 기본적으로 생략**된다(JSON 출력에서). 옵션으로 "기본값도 포함" 모드를 켤 수 있다. proto3 `optional` 필드는 설정됐다면 기본값이어도 출력된다.

```text
 proto:
   message Order { string order_id = 1; int64 total = 2;
                   OrderStatus status = 3; }
 값:  order_id="A-100", total=9007199254740993, status=PAID

 canonical JSON:
   {
     "orderId": "A-100",
     "total":   "9007199254740993",        ← 64비트라 문자열!
     "status":  "ORDER_STATUS_PAID"        ← enum은 이름
   }
```

이 매핑 덕에 같은 `.proto` 하나로 바이너리 gRPC와 JSON/HTTP를 동시에 서비스할 수 있다.

---

## 11. 공식 스타일 가이드: 읽기 좋은 계약서의 규칙

Protobuf 공식 스타일 가이드는 짧지만 일관성을 위해 지킬 가치가 있다. 핵심만 모은다.

```text
 [파일]
   - 파일명: lower_snake_case.proto  (예: order_service.proto)
   - 한 파일 = 한 주제. 줄당 80자 권장. 들여쓰기 2칸.
   - import는 알파벳 정렬. 맨 위에 syntax, 그다음 package, 그다음 import.

 [메시지/enum/서비스 이름]  → PascalCase(=UpperCamelCase)
   message Order  /  enum OrderStatus  /  service OrderService

 [필드 이름]  → lower_snake_case
   string order_id = 1;   repeated LineItem line_items = 2;
   (repeated 필드는 복수형 이름: line_items, song_ids)

 [enum 값 이름]  → UPPER_SNAKE_CASE, 보통 enum명을 접두사로
   enum OrderStatus {
     ORDER_STATUS_UNSPECIFIED = 0;   ← 0번은 _UNSPECIFIED
     ORDER_STATUS_PENDING     = 1;
   }
   (접두사를 붙이는 이유: 일부 언어에서 enum 값이 같은 스코프에 풀려
    이름 충돌이 나는 걸 막기 위함)

 [RPC 메서드]  → PascalCase, 동사+명사
   rpc CreateOrder(...)  rpc ListOrders(...)  rpc GetOrder(...)

 [주석]
   - // 또는 /* */. 필드/메시지 위에 의도(why)를 적는다.
   - 생성 코드의 문서주석으로 전파되므로 "무엇"보다 "왜/제약"을 적어라.
```

설계 원칙으로 한 가지 더: **한 메시지는 하나의 책임**을 가져야 한다. 거대한 "God message"에 모든 걸 욱여넣지 말고, 응집도 높은 단위로 쪼개라. `CreateOrderRequest`/`CreateOrderResponse`처럼 메서드별 요청·응답 메시지를 따로 두는 패턴(공유 메시지 재사용 자제)이 진화에 유리하다 — 한 메서드의 요청에 필드를 더해도 다른 메서드가 안 흔들리니까.

---

## 12. 전체를 엮는 실전 예시: 주문 서비스

지금까지의 모든 요소를 하나의 일관된 `.proto`로 엮어 보자. 작은 주문(Order) 도메인이다. 각 줄에 왜 그렇게 썼는지 주석으로 달았다.

```proto
syntax = "proto3";                          // proto3 방언 선언(필수, 맨 위)

package shop.order.v1;                       // 네임스페이스 + 버전(v1)

import "google/protobuf/timestamp.proto";    // WKT: 시각
import "google/protobuf/field_mask.proto";   // WKT: 부분 업데이트

option go_package = "github.com/example/shop/gen/orderv1;orderv1";
option java_package = "com.example.shop.order.v1";
option java_multiple_files = true;

// 주문 상태. 0번은 항상 _UNSPECIFIED(미지정=기본값).
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_PAID        = 2;
  ORDER_STATUS_SHIPPED     = 3;
  ORDER_STATUS_CANCELLED   = 4;
  // 5번은 예전에 REFUNDED였다가 정책 변경으로 삭제 → 무덤 표시.
  reserved 5;
  reserved "ORDER_STATUS_REFUNDED";
}

// 금액(통화 분리). 소수점 오차를 피하려 정수 minor unit 사용.
message Money {
  string currency_code = 1;   // ISO 4217, 예: "USD","KRW"
  int64  amount_minor  = 2;   // 최소 단위(센트/원). 음수 거의 없음 → int64
}

// 주문 한 줄. Order 안에서만 쓰이므로 중첩 정의.
message Order {
  message LineItem {
    string product_id = 1;     // 자주 등장 → 1~15 황금번호
    int32  quantity   = 2;
    Money  unit_price = 3;     // 메시지 필드: 항상 presence 있음
  }

  string                    order_id   = 1;   // 자주 읽힘 → 1번
  string                    buyer_id   = 2;
  repeated LineItem         items      = 3;   // 0개 이상(복수형 이름)
  Money                     total      = 4;
  OrderStatus               status     = 5;
  google.protobuf.Timestamp created_at = 6;   // 절대 시각
  optional string           memo       = 7;   // explicit presence:
                                              // "메모 없음"과 "빈 메모"를 구분
}

// --- 요청/응답 메시지: 메서드마다 따로 둔다 ---

message CreateOrderRequest {
  string             buyer_id = 1;
  repeated Order.LineItem items = 2;          // 중첩 타입 참조
}
message CreateOrderResponse {
  Order order = 1;                            // 생성된 주문 전체 반환
}

message GetOrderRequest  { string order_id = 1; }
message GetOrderResponse { Order order = 1; }

// 부분 업데이트: FieldMask로 "무엇을 바꿀지" 명시.
message UpdateOrderRequest {
  Order                     order       = 1;
  google.protobuf.FieldMask update_mask = 2;  // 예: paths=["status","memo"]
}
message UpdateOrderResponse { Order order = 1; }

message WatchOrderRequest { string order_id = 1; }

// 주문에서 일어나는 이벤트. 종류가 상호배타 → oneof.
message OrderEvent {
  google.protobuf.Timestamp at = 1;
  oneof event {
    OrderStatus status_changed = 2;   // 상태 바뀜
    Order.LineItem item_added  = 3;   // 품목 추가
    string         note_added  = 4;   // 메모 추가
  }
}

// 서비스: 데이터로 무엇을 할 수 있는가.
service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest)       returns (GetOrderResponse);
  rpc UpdateOrder(UpdateOrderRequest) returns (UpdateOrderResponse);

  // 서버 스트리밍: 주문 변화를 실시간 구독.
  rpc WatchOrder(WatchOrderRequest)   returns (stream OrderEvent);
}
```

이 한 파일에 이 장의 개념이 거의 다 들어있다: proto3 선언, package/버전, import와 WKT, enum의 0번·reserved, 중첩 메시지, 메시지 필드의 presence, `optional`, `repeated`, `oneof`, FieldMask 패턴, 황금 번호, 메서드별 요청/응답 분리, 그리고 unary/server-streaming 두 가지 rpc.

### 생성 코드를 쓰는 모습(맛보기)

이 `.proto`를 `protoc`로 컴파일하면([[06 - 코드 생성과 protoc 툴체인]]) 언어별 타입과 클라이언트/서버 스텁이 나온다. presence와 enum이 코드에서 어떻게 드러나는지 두 언어로 잠깐 본다.

```go
// Go: 생성된 타입 사용
o := &orderv1.Order{
    OrderId: "A-100",
    Status:  orderv1.OrderStatus_ORDER_STATUS_PAID,
    Total:   &orderv1.Money{CurrencyCode: "KRW", AmountMinor: 12000},
}
// optional string memo → *string 포인터로 생성됨(presence 표현)
if o.Memo == nil {
    // 메모를 "아예 설정 안 함"
} else {
    _ = *o.Memo  // 설정된 값(빈 문자열일 수도)
}
```

```python
# Python: 생성된 타입 사용
from shop.order.v1 import order_pb2

o = order_pb2.Order(order_id="A-100",
                    status=order_pb2.ORDER_STATUS_PAID)
o.total.currency_code = "KRW"      # 메시지 필드는 접근하면 자동 생성
o.total.amount_minor = 12000

# optional 필드는 HasField로 presence 질의
if o.HasField("memo"):
    print(o.memo)
else:
    print("memo 미설정")
```

두 코드 모두 핵심은 같다: `optional`/메시지 필드는 "설정됐는지"를 물어볼 수 있고(`!= nil`, `HasField`), 일반 스칼라는 그냥 기본값을 본다. 이게 이 장 내내 강조한 **presence**가 실제 코드에 드러나는 방식이다.

---

## 13. 한눈에 보는 의사결정 가이드

실무에서 `.proto`를 쓸 때 자주 부딪히는 갈림길을 표로 정리한다.

| 상황 | 선택 | 이유 |
|---|---|---|
| 양수 정수(개수/크기) | `int32`/`int64` 또는 `uint32/64` | 음수 없으면 uint가 의미 명확 |
| 음수가 흔한 정수 | `sint32`/`sint64` | ZigZag로 음수도 짧게 |
| 항상 큰 값/해시 | `fixed64`/`sfixed64` | varint보다 고정폭이 작음 |
| 텍스트 | `string` | UTF-8 검증 |
| 바이너리/암호문 | `bytes` | UTF-8 제약 없음 |
| "0과 미설정"을 구분해야 함 | `optional` 스칼라 | explicit presence |
| 시각/간격 | `Timestamp`/`Duration` | 표준·정밀·JSON 친화 |
| 부분 업데이트 | `FieldMask` | "무엇을 바꿀지" 명시 |
| 상호배타 선택 | `oneof` | union·presence |
| 키-값 매핑 | `map` | repeated entry로 표현 |
| 타입이 런타임 결정 | `Any`(닫혀있으면 `oneof`) | 다형 페이로드 |
| 필드 삭제 | 삭제 + `reserved` | 번호 재사용 사고 방지 |
| 자주 쓰는 필드 | 번호 1~15 | 1바이트 태그 |

---

## 핵심 요약

- **Protobuf는 IDL이자 직렬화 포맷이다.** 데이터의 모양을 코드보다 먼저 `.proto` 계약서에 못 박는 "스키마-온-라이트"로, 타입 안전·작은 크기·하위호환을 동시에 얻는다.
- **proto2→proto3는 presence를 둘러싼 철학의 재조정**이다. proto3는 `required`(백만 달러짜리 실수)를 없애고 기본값을 도입했으나, 그 대가로 "0과 미설정을 구분 못 하는" presence 문제를 낳았고, `optional`의 부활로 명시적 존재성을 부분적으로 되찾았다.
- **스칼라가 기본값이면 와이어에서 생략**된다(암묵적 존재성). 구분이 필요하면 `optional`을 붙여라. **메시지 타입 필드는 언제나 presence가 있다** — 이 비대칭이 타입 시스템의 핵심.
- **int 계열이 셋(int/uint/sint)인 이유는 varint 인코딩 전략** 때문이다. 음수 빈도와 값 크기에 따라 int/sint/fixed를 골라야 크기가 최적이 된다.
- **필드 번호가 진짜 신분증**이다. 이름은 와이어에 없다. 1~15는 1바이트 태그라 핫 필드에 배정하고, 삭제한 번호는 `reserved`로 무덤을 남겨 재사용 사고를 막아라.
- **복합 타입은 결국 기본 규칙으로 환원된다.** `map`은 repeated {key,value} 메시지로, `optional` 스칼라는 단일 멤버 oneof로, 중첩 메시지는 length-prefixed 블록으로 컴파일된다.
- **Well-Known Types는 표준 부품**이다. Timestamp/Duration/Any/Struct/FieldMask/Empty/wrapper를 직접 정의하지 말고 가져다 써라. 특히 presence가 필요하면 wrapper보다 `optional`을 우선.
- **service/rpc는 4가지 stream 조합**을 선언하고, 요청·응답은 반드시 (확장 가능한) 메시지여야 한다. JSON 매핑은 필드명 camelCase, 64비트는 문자열, enum은 이름이라는 규칙을 기억하라.

---

## 연결 노트

- [[01 - gRPC란 무엇인가]] — 이 계약서가 왜 필요한지, RPC라는 큰 그림.
- [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]] — 여기서 정의한 타입·필드 번호·packed·태그가 실제 바이트로 어떻게 변하는지 hex 단위로 디코딩.
- [[05 - 통신의 4가지 방식 - Unary와 Streaming]] — service에서 선언한 `stream` 조합이 HTTP/2 위에서 실제로 어떻게 동작하는지.
- [[06 - 코드 생성과 protoc 툴체인]] — `.proto`에서 언어별 코드를 뽑는 protoc/플러그인, go_package 등 옵션.
- [[08 - 메타데이터와 인터셉터]] — rpc 호출에 부가 정보를 싣고 가로채기.
- [[09 - 에러 모델 - 상태 코드와 Rich Error]] — 에러 디테일을 `Any`로 싣는 풍부한 에러 모델.
- [[16 - gRPC-Web과 REST 게이트웨이]] — JSON 매핑을 활용한 REST 변환.
- [[17 - 실전 - proto 관리와 하위호환성]] — reserved·필드 번호·패키지 버전을 실제 진화 전략으로 엮기.
