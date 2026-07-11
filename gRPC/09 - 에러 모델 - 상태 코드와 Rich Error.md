---
title: "09 - 에러 모델 - 상태 코드와 Rich Error"
date: 2026-06-26
tags:
  - grpc
  - error-handling
  - status-code
  - rich-error-model
  - google-rpc-status
  - trailers
  - retry
  - 학습노트
---

## 이 장이 답하는 질문

- gRPC에서 "에러"는 정확히 무엇으로 구성되며, 성공한 호출조차 왜 `grpc-status=0`을 갖는가?
- 16개 상태 코드는 각각 무엇을 의미하고, 헷갈리기 쉬운 코드들(INVALID_ARGUMENT vs FAILED_PRECONDITION vs OUT_OF_RANGE, ABORTED vs FAILED_PRECONDITION, PERMISSION_DENIED vs UNAUTHENTICATED)을 어떻게 구분하는가?
- 어떤 코드가 안전하게 재시도(retry) 가능하고 어떤 코드가 아닌가? 멱등성(idempotency)은 여기에 어떻게 얽히는가?
- `code` 하나로는 부족할 때, 구조화된 에러 정보(Rich Error Model, `google.rpc.Status`)를 어떻게 와이어에 실어 보내고 꺼내는가?
- 클라이언트가 "사람이 읽는 메시지 문자열 파싱" 없이 프로그램적으로 분기하려면 어떤 detail을 붙여야 하는가?

---

## 1. 왜 gRPC는 에러를 "반환값"으로 모델링했는가

대부분의 RPC 초심자는 에러를 "정상 흐름에서 벗어난 예외 상황"으로 생각한다. 그러나 분산 시스템에서 네트워크는 **항상 부분적으로 고장 나 있다**(partial failure). 패킷은 유실되고, 상대 서버는 재배포 중이며, 데드라인(deadline)은 만료되고, 인증 토큰은 갱신 직전이다. 이런 환경에서 "에러는 예외다"라는 관점은 위험하다. 에러야말로 **정상 흐름의 일부**이고, 모든 호출은 "성공했거나, 어떤 이유로 실패했거나" 둘 중 하나로 **반드시** 끝난다.

gRPC는 이 철학을 타입 시스템에 못 박았다. 모든 gRPC 호출의 종착점에는 예외 없이 하나의 **`Status`(상태)** 객체가 존재한다. 성공이란 별도의 개념이 아니라 "코드가 `OK`(0)인 `Status`"일 뿐이다. 즉 gRPC의 세계관에서는:

```text
모든 RPC 호출  →  정확히 하나의 Status로 종료
                 ├─ code == OK(0)        → 성공 (응답 메시지 유효)
                 └─ code != OK           → 실패 (응답 메시지 없음/무의미)
```

이 설계의 직관을 잡으려면 HTTP와 비교해 보면 된다. HTTP에서는 상태가 응답 **헤더의 맨 앞**(status line, 예: `200 OK`, `404 Not Found`)에 온다. 서버는 응답을 만들기 시작하는 순간 이미 성공/실패를 알고 있어야 한다. 하지만 스트리밍(streaming) 응답을 생각해 보자. 서버가 1만 개의 항목을 흘려보내다가 9,999번째에서 DB 커넥션이 끊기면? HTTP/1.1에서는 이미 `200 OK`를 보내버렸기 때문에 "사실 실패했다"고 정정할 방법이 마땅치 않다(chunked transfer의 trailer가 있지만 거의 안 쓰인다).

gRPC는 이 문제를 정면으로 해결한다. **최종 상태를 응답의 맨 뒤, 즉 트레일러(trailer)에 둔다.** 서버는 모든 메시지를 다 보낸 **후에** "이 호출은 결국 성공/실패였다"를 선언한다. 스트리밍에서 중간에 터져도 정직하게 실패 상태를 마지막에 붙일 수 있다. 이것이 gRPC가 상태 코드를 HTTP 상태 라인이 아니라 HTTP/2 트레일러로 운반하는 근본 이유다. 이 트레일러 메커니즘 자체는 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 다룬 HEADERS 프레임의 두 번째 등장(end-of-stream 플래그를 단)으로 구현된다.

### 에러 = 세 겹의 양파

gRPC `Status`는 세 개의 필드로 이루어진다. 이 세 겹을 명확히 구분하는 것이 이 장의 전부라고 해도 과언이 아니다.

```text
┌─────────────────────────────────────────────────────────────┐
│  gRPC Status                                                 │
│                                                              │
│  ① code     : enum (0~16)    ← 기계가 분기하는 거친 분류       │
│  ② message  : UTF-8 문자열    ← 사람(개발자)이 읽는 디버그 설명  │
│  ③ details  : Any[] (구조화)  ← 기계가 정밀 분기하는 부가 정보   │
└─────────────────────────────────────────────────────────────┘
```

- **① code (상태 코드)**: 17개(OK 포함) 중 하나의 열거형(enum). "무엇이 잘못됐는가"의 가장 거친 분류. 클라이언트 라이브러리, 로드밸런서(load balancer), 재시도 정책(retry policy)이 이 값만 보고 동작을 결정한다. **언어·도메인 중립적**이며 절대 임의로 늘릴 수 없는 닫힌 집합(closed set)이다.
- **② message (상태 메시지)**: 자유 형식 UTF-8 문자열. 주로 **로그와 디버깅**에 쓴다. 기계가 이 문자열을 파싱해서 분기하면 안 된다(문자열은 언제든 바뀐다). 여기에 민감 정보(스택 트레이스, 내부 호스트명, PII)를 담으면 클라이언트로 그대로 새어 나가므로 주의해야 한다.
- **③ details (상태 상세)**: `google.protobuf.Any`의 리스트. 여기에 **구조화된, 타입이 있는** 에러 정보를 담는다. 이것이 곧 **Rich Error Model**이며 이 장의 후반부 주제다.

이 셋의 역할 분담이 핵심이다. `code`는 "교통 경찰"(라이브러리/인프라가 보는 신호등), `message`는 "사고 현장 메모"(사람이 보는 글), `details`는 "보험 서류"(클라이언트 코드가 정밀하게 처리하는 데이터)라고 비유할 수 있다.

---

## 2. 와이어에서 실제로 무슨 일이 일어나는가

추상 모델을 봤으니 이제 바이트 레벨로 내려가자. gRPC over HTTP/2의 한 번의 Unary 호출이 끝나는 순간을 손으로 디코딩해 본다. (HTTP/2 프레임·HPACK 헤더 압축의 기초는 [[04 - HTTP2 깊이 보기 - 전송 계층]] 참조.)

### 2.1 성공 응답의 와이어 형태

서버가 정상 응답을 보낼 때, HTTP/2 스트림 위에는 보통 세 덩어리가 흐른다.

```text
[1] 응답 HEADERS 프레임 (initial metadata)
    :status: 200
    content-type: application/grpc+proto

[2] 응답 DATA 프레임 (메시지 본문, length-prefixed)
    00 00 00 00 0A  <10바이트의 protobuf 메시지>
    └┬┘ └────┬───┘
   압축0   길이=10 (big-endian uint32)

[3] 응답 트레일러 HEADERS 프레임 (END_STREAM 플래그 셋)
    grpc-status: 0
    grpc-message: (없음 또는 빈 값)
```

여기서 두 가지를 놓치기 쉽다.

첫째, **HTTP 상태는 거의 항상 `200 OK`다.** gRPC 호출이 `NOT_FOUND`로 실패해도, `PERMISSION_DENIED`로 실패해도 HTTP `:status`는 `200`이다. "gRPC 호출이 실제로 서버 핸들러에 도달해서 처리됐다"면, 그 처리 결과(성공이든 도메인 실패든)는 HTTP 레벨이 아니라 `grpc-status` 트레일러로만 표현된다. HTTP 상태가 `200`이 아닌 경우는 HTTP 레벨에서 문제가 생긴 경우(예: 프록시가 `502`, `503`을 반환)이고, 그때는 gRPC 라이브러리가 HTTP 상태를 적절한 gRPC 코드로 매핑한다(뒤의 표 참조).

둘째, **성공조차 `grpc-status: 0`이라는 트레일러를 명시적으로 갖는다.** "에러가 없으면 트레일러도 없다"가 아니다. 성공은 `code=OK`라는 상태의 한 종류이며, 이 트레일러가 도착해야 비로소 클라이언트는 "호출이 완전히, 정상적으로 끝났다"고 확신할 수 있다. 트레일러가 안 오고 연결이 끊기면 그건 성공이 아니라 `UNAVAILABLE` 같은 전송 에러로 귀결된다.

### 2.2 실패 응답과 Trailers-Only 응답

도메인 에러(예: `NOT_FOUND`)가 나면 보통 DATA 프레임 없이 트레일러만 온다. 그런데 더 흥미로운 케이스가 **Trailers-Only 응답**이다. 서버가 요청을 받자마자(메시지 본문을 처리하기도 전에) 실패를 확정할 수 있으면(예: 라우팅 실패, 인증 실패), 서버는 **헤더와 트레일러를 한 번의 HEADERS 프레임에 합쳐서** 보낸다.

```text
단일 HEADERS 프레임 (END_STREAM 플래그 셋) — Trailers-Only
    :status: 200
    content-type: application/grpc
    grpc-status: 5            (NOT_FOUND)
    grpc-message: user 42 not found
```

즉 DATA 프레임도 없고, "initial metadata HEADERS"와 "trailing metadata HEADERS"를 따로 보내지도 않는다. 하나의 HEADERS 프레임에 `:status`와 `grpc-status`가 함께 들어 있고 그게 곧 스트림의 끝이다. 이것을 **Trailers-Only**라고 부르며, 가벼운 에러 경로의 최적화다. gRPC 클라이언트 구현은 "initial HEADERS와 trailing HEADERS가 분리되어 오는 경우"와 "Trailers-Only로 합쳐져 오는 경우"를 모두 처리할 수 있어야 한다.

### 2.3 grpc-message의 퍼센트 인코딩

`grpc-message`는 HTTP/2 헤더 값에 들어가므로 ASCII만 안전하게 담을 수 있다. 그런데 메시지는 UTF-8 한글 등 임의 바이트를 담을 수 있어야 한다. gRPC는 이를 위해 **Percent-Encoding의 변형**을 쓴다. gRPC 와이어 사양이 정의하는 비인코딩 범위는 `0x20~0x24`와 `0x26~0x7E`, 즉 **출력 가능 ASCII(0x20~0x7E) 중 `%`(0x25) 하나만 제외한** 바이트다. 이 범위의 바이트는 그대로 두고, 나머지 — 0x20 미만의 제어 문자, `%` 자신, 그리고 0x7E를 넘는 모든 비ASCII 바이트 — 는 `%XX`(대문자 16진수)로 인코딩한다. 한 가지 흔히 오해하는 점: **공백(0x20)은 비인코딩 범위(0x20~0x24)에 속하므로 `%20`으로 바뀌지 않고 리터럴 빈칸 그대로 남는다.**

예를 들어 메시지가 "사용자 없음"이라면, UTF-8 바이트열이 그대로 헤더에 들어가지 못하고 다음처럼 퍼센트 인코딩된다.

```text
"사" = UTF-8 0xEC 0x82 0xAC  →  %EC%82%AC
"용" = UTF-8 0xEC 0x9A 0xA9  →  %EC%9A%A9
...
grpc-message: %EC%82%AC%EC%9A%A9%EC%9E%90 %EC%97%86%EC%9D%8C
                                       ↑ '자'와 '없' 사이의 공백(0x20)은
                                         비인코딩 범위라 리터럴 빈칸 그대로
```

(즉 비ASCII 한글 바이트만 `%XX`로 바뀌고, 그 사이의 공백 0x20은 사양상 인코딩 대상이 아니어서 그대로 빈칸으로 남는다.) 클라이언트 라이브러리가 받을 때 `%XX` 토큰들을 디코딩해서 원래 메시지를 복원한다. 이 때문에 메시지에 무엇을 담든 바이트 안전성은 보장되지만, 한 가지 함의가 있다. **메시지는 헤더 크기 제한(보통 8KB 안팎의 HPACK 동적 테이블/프레임 제한)을 공유**한다는 점이다. 그래서 긴 에러 정보는 message가 아니라 details(아래)에 담는 것이 옳다.

### 2.4 details는 어디로? — grpc-status-details-bin 트레일러

`code`와 `message`는 `grpc-status`/`grpc-message` 헤더로 충분히 전달된다. 그렇다면 구조화된 `details`(여러 개의 Any)는 어떻게 보낼까? 정답은 별도의 **바이너리 트레일러**다.

gRPC 메타데이터에서 키 이름이 `-bin`으로 끝나면 그 값은 텍스트가 아니라 **임의 바이너리(base64로 인코딩되어 전송)** 로 취급된다(이 규칙은 [[08 - 메타데이터와 인터셉터]]에서 다룬다). Rich Error는 이 규칙을 활용한다.

```text
grpc-status:              5
grpc-message:             user 42 not found
grpc-status-details-bin:  <base64( google.rpc.Status 직렬화 바이트 )>
```

여기서 결정적인 사실이 하나 있다. **`grpc-status-details-bin`에 담기는 것은 `google.rpc.Status` 메시지 전체다.** 즉 이 트레일러 안에는 `code`와 `message`가 **다시 한 번** 들어 있고, 거기에 더해 `details: repeated Any`가 들어 있다.

```protobuf
// google/rpc/status.proto
package google.rpc;

message Status {
  int32 code = 1;                              // grpc-status와 동일해야 함
  string message = 2;                          // grpc-message와 동일/유사
  repeated google.protobuf.Any details = 3;    // 구조화된 상세 (Rich!)
}
```

왜 중복일까? 일관성과 자기완결성 때문이다. `grpc-status`/`grpc-message`(텍스트 헤더)는 details를 모르는 단순 프록시·로드밸런서·구버전 클라이언트도 읽을 수 있는 "기본 채널"이고, `grpc-status-details-bin`은 Rich Error를 이해하는 클라이언트만 들여다보는 "확장 채널"이다. 서버는 양쪽의 `code`/`message`를 일치시켜야 한다. 만약 details를 모르는 클라이언트라면 `grpc-status` 트레일러만 읽고도 `code=5`라는 핵심 정보를 잃지 않는다 — 이것이 **하위 호환(backward compatibility)** 의 우아한 지점이다.

### 2.5 google.rpc.Status 바이트를 손으로 디코딩하기

직관을 확실히 하기 위해, `code=5(NOT_FOUND)`, `message="not found"`인 가장 단순한 `google.rpc.Status`를 protobuf로 직렬화한 바이트를 직접 디코딩해 보자. (protobuf 와이어 포맷의 기초는 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]] 참조.)

```text
원본 메시지: code=5, message="not found", details=[]

직렬화 바이트(hex):
08 05 12 09 6E 6F 74 20 66 6F 75 6E 64

손 디코딩:
08            → 태그: field #1, wire type 0 (varint)   (0x08 = (1<<3)|0)
05            → 값: 5                                    (code = NOT_FOUND)
12            → 태그: field #2, wire type 2 (length-delimited)  (0x12 = (2<<3)|2)
09            → 길이: 9 바이트
6E 6F 74 20 66 6F 75 6E 64  → "not found" (ASCII)        (message)
(field #3 details 없음)
```

이 13바이트를 base64로 감싸면 `CAUSCW5vdCBmb3VuZA==`가 되고, 그것이 `grpc-status-details-bin` 트레일러의 값으로 와이어를 탄다. 클라이언트는 이 값을 base64 디코딩 → protobuf 역직렬화 → `google.rpc.Status`로 복원 → `details`의 각 `Any`를 알려진 타입으로 언패킹(unpack)하는 순서로 처리한다.

`details`에 실제로 `Any`가 하나 들어가면 어떻게 보일까? `Any`는 두 필드를 가진 메시지다.

```protobuf
// google/protobuf/any.proto
message Any {
  string type_url = 1;   // 예: "type.googleapis.com/google.rpc.ErrorInfo"
  bytes  value    = 2;   // 위 타입을 직렬화한 바이트
}
```

즉 `details`의 한 원소는 "이게 무슨 타입인지 알려주는 URL(type_url)" + "그 타입을 직렬화한 바이트(value)"의 쌍이다. 클라이언트는 `type_url`의 끝부분(`google.rpc.ErrorInfo`)을 보고 자기가 아는 타입이면 `value`를 그 타입으로 다시 역직렬화한다. 모르는 타입이면 그냥 건너뛴다 — 이 역시 확장성과 하위 호환을 보장하는 설계다.

---

## 3. 16개 상태 코드 전부 — 의미·전형 상황·재시도

이제 닫힌 집합인 17개 코드(OK 포함)를 전부 본다. gRPC 코드는 원래 Google 내부의 canonical error codes에서 유래했고, 모든 gRPC 언어 구현이 정확히 같은 숫자·이름을 공유한다.

| # | 코드 | 한 줄 의미 | 전형적 상황 | 일반적 재시도 가능성 |
|---|------|-----------|-------------|----------------------|
| 0 | **OK** | 성공 | 정상 처리 완료 | — (재시도 불필요) |
| 1 | **CANCELLED** | 호출자가 취소함 | 클라이언트가 `cancel()` 호출, 컨텍스트 취소 | ✗ (의도된 취소) |
| 2 | **UNKNOWN** | 분류 불가/원인 미상 | 서버가 던진 미매핑 예외, 다른 주소 공간의 알 수 없는 에러 | △ (원인 불명, 보통 신중히) |
| 3 | **INVALID_ARGUMENT** | 인자 자체가 잘못됨 | 음수 page_size, 형식 안 맞는 이메일 | ✗ (그대로 재시도해도 또 실패) |
| 4 | **DEADLINE_EXCEEDED** | 데드라인 초과 | 응답 전 타임아웃 만료 | △ (멱등하면 ○, 부작용 모를 수 있음) |
| 5 | **NOT_FOUND** | 대상 리소스 없음 | user/order id가 존재 안 함 | ✗ (대상이 생기기 전엔 무의미) |
| 6 | **ALREADY_EXISTS** | 이미 존재 | 중복 생성, 유니크 제약 위반 | ✗ |
| 7 | **PERMISSION_DENIED** | 권한 없음(인증은 됨) | 인증된 사용자가 권한 부족 | ✗ (권한 바뀌기 전엔 무의미) |
| 8 | **RESOURCE_EXHAUSTED** | 자원 고갈 | 쿼터 초과, 레이트 리밋, 디스크/메모리 부족 | △ (RetryInfo/백오프 후 ○) |
| 9 | **FAILED_PRECONDITION** | 시스템 상태가 작업 전제 위반 | 비어있지 않은 디렉터리 rmdir, 잘못된 상태 전이 | ✗ (상태 고치기 전엔 재시도 무의미) |
| 10 | **ABORTED** | 동시성 충돌로 중단 | 트랜잭션 충돌, 낙관적 락(OCC) 실패, 시퀀서 충돌 | ○ (상위 레벨에서 재시도) |
| 11 | **OUT_OF_RANGE** | 유효 범위 밖 | 파일 끝 너머 seek/read, 음수 오프셋 범위 초과 | △ (상태 바뀌면 유효해질 수 있음) |
| 12 | **UNIMPLEMENTED** | 미구현/미지원 메서드 | 서버에 없는 RPC, 비활성 기능 | ✗ |
| 13 | **INTERNAL** | 심각한 내부 불변식 깨짐 | 직렬화 오류, 깨진 가정, 버그 | ✗ (보통; 버그 신호) |
| 14 | **UNAVAILABLE** | 일시적으로 사용 불가 | 서버 재배포/다운, 커넥션 끊김, 과부하 | ○ (가장 전형적인 재시도 대상) |
| 15 | **DATA_LOSS** | 복구 불가능한 데이터 손실/손상 | 체크섬 불일치, 영구 손상 | ✗ |
| 16 | **UNAUTHENTICATED** | 인증 자격 없음/무효 | 토큰 없음/만료/위조 | ✗ (자격 갱신 후엔 ○) |

표만 보면 외워야 할 것 같지만, 실제로는 **몇 개의 직관 축**으로 묶으면 자연스럽게 분류된다.

- **호출자 책임 vs 서버 책임**: `INVALID_ARGUMENT`(3), `NOT_FOUND`(5), `ALREADY_EXISTS`(6), `PERMISSION_DENIED`(7), `UNAUTHENTICATED`(16)는 "클라이언트가 뭔가 잘못 보냈다". `INTERNAL`(13), `DATA_LOSS`(15), `UNKNOWN`(2)은 "서버 쪽 사정".
- **지금 안 되지만 나중엔 될 수도**: `UNAVAILABLE`(14), `RESOURCE_EXHAUSTED`(8), `ABORTED`(10)는 "시간이 지나거나 재시도하면 풀릴 가능성".
- **상태/전제 문제**: `FAILED_PRECONDITION`(9), `OUT_OF_RANGE`(11)는 "시스템의 현재 상태와 작업이 안 맞는다".

### 3.1 HTTP 상태와의 매핑

직관을 보강하기 위해 흔히 쓰이는 HTTP 상태 ↔ gRPC 코드 매핑을 정리한다. gRPC 게이트웨이([[16 - gRPC-Web과 REST 게이트웨이]])나 프록시를 다룰 때 자주 마주친다. 아래는 "프록시가 HTTP 에러를 gRPC 코드로 변환할 때"의 표준 매핑이다.

| HTTP 상태 | → gRPC 코드 |
|-----------|-------------|
| 400 Bad Request | INTERNAL (※ 주의: 클라이언트가 보낸 HTTP가 깨진 것) |
| 401 Unauthorized | UNAUTHENTICATED |
| 403 Forbidden | PERMISSION_DENIED |
| 404 Not Found | UNIMPLEMENTED |
| 429 Too Many Requests | UNAVAILABLE |
| 502 Bad Gateway | UNAVAILABLE |
| 503 Service Unavailable | UNAVAILABLE |
| 504 Gateway Timeout | UNAVAILABLE |

여기서 두 줄이 직관에 반한다. HTTP `404`가 gRPC `NOT_FOUND`가 아니라 `UNIMPLEMENTED`로 매핑되는 이유는, 이 매핑이 "**프록시가 HTTP/2 경로 자체를 못 찾았다**"는 전송 레벨 상황을 다루기 때문이다. gRPC에서 경로(`/package.Service/Method`)를 못 찾는다는 건 "그 메서드가 구현 안 됐다"는 뜻이다. 도메인 리소스가 없는 `NOT_FOUND`(5)와는 층위가 완전히 다르다. 마찬가지로 `429`(레이트 리밋)가 `RESOURCE_EXHAUSTED`가 아니라 `UNAVAILABLE`로 가는 것도, 이 변환이 "프록시/인프라 레벨"의 일시 불가를 다루는 까닭이다(서버 핸들러가 의도적으로 던지는 쿼터 초과라면 핸들러가 직접 `RESOURCE_EXHAUSTED`를 쓴다).

---

## 4. 헷갈리는 코드들 — 깊이 있는 구분

코드를 "외우는" 것과 "고르는" 것은 다르다. API 설계의 품질은 이 미묘한 구분에서 갈린다. 잘못된 코드는 클라이언트의 재시도 정책을 망가뜨리고(예: 재시도하면 안 되는 걸 무한 재시도), 운영자의 디버깅을 오도한다.

### 4.1 INVALID_ARGUMENT vs FAILED_PRECONDITION vs OUT_OF_RANGE

이 셋은 공식 문서가 명시적으로 함께 묶어 설명할 만큼 헷갈린다. 핵심 판별 기준은 **"인자 자체가 문제인가, 시스템 상태가 문제인가, 그리고 시간이 지나면 같은 인자가 유효해질 수 있는가"** 다.

```text
질문 1: 요청 인자가 시스템의 현재 상태와 "무관하게" 그 자체로 틀렸는가?
        (예: page_size = -1, 이메일 형식 깨짐)
        → YES: INVALID_ARGUMENT (3)
                "이 인자는 어떤 우주에서도 틀렸다."

질문 2: 인자는 형식상 멀쩡한데, 시스템의 현재 상태가 작업을 거부하는가?
        (예: 비어있지 않은 버킷을 삭제하려 함)
        → YES: FAILED_PRECONDITION (9)
                "인자는 OK지만 지금 상태에선 안 된다.
                 너는 상태를 먼저 고쳐야 한다(버킷을 비워라)."

질문 3: 형식은 맞지만 '유효 범위'를 벗어났고, 그 범위가 상태에 따라 달라지는가?
        (예: 길이 100인 파일에서 offset 200을 read)
        → YES: OUT_OF_RANGE (11)
                "지금은 범위 밖이지만, 파일이 자라면 같은 offset이 유효해질 수 있다."
```

`INVALID_ARGUMENT`와 `OUT_OF_RANGE`의 결정적 차이가 바로 **상태 의존성**이다. 공식 가이드라인의 표현을 빌리면: 시스템 상태와 **무관하게** 틀린 값은 `INVALID_ARGUMENT`이고, **현재 상태**에서만 범위를 벗어난 값(나중엔 유효해질 수 있는)은 `OUT_OF_RANGE`다. 후자는 클라이언트에게 유리하다. 예컨대 순차적으로 파일을 읽는 클라이언트는 `INVALID_ARGUMENT`(음수 offset 등 자기 버그)와 `OUT_OF_RANGE`(파일 끝에 도달, 즉 EOF로 정상 종료)를 또렷이 구분해 처리한다.

`FAILED_PRECONDITION`과 `INVALID_ARGUMENT`의 차이도 같은 축이다. "page_size=-1"은 시스템에 뭐가 있든 틀렸으니 `INVALID_ARGUMENT`. "버킷을 지우려는데 안이 안 비었다"는 인자(버킷 이름)는 멀쩡하고 시스템 상태가 거부하는 것이니 `FAILED_PRECONDITION`. 후자에는 보통 "무엇을 고쳐야 하는가"(버킷을 비워라)를 `PreconditionFailure` detail로 첨부한다(6절).

### 4.2 NOT_FOUND vs UNAVAILABLE

둘 다 "원하는 걸 못 얻었다"지만 의미와 재시도 가능성이 정반대다.

```text
NOT_FOUND (5):     "리소스는 존재하지 않는다(또는 너에게 안 보인다)."
                   → 같은 요청을 재시도해도 영원히 NOT_FOUND.
                   → 재시도 의미 없음. 클라이언트가 id를 바꾸거나 포기해야 함.

UNAVAILABLE (14):  "서버/백엔드가 지금 응답할 수 없다."
                   → 잠시 후 재시도하면 성공할 가능성이 매우 높다.
                   → 백오프 후 재시도의 '가장 전형적인' 대상.
```

함정은 구현이 둘을 뒤섞는 경우다. 예를 들어 백엔드 DB가 다운됐는데 "행을 못 찾았으니 NOT_FOUND"라고 응답하면, 클라이언트는 "리소스가 정말 없구나" 하고 재시도를 포기한다. 실제로는 DB만 복구되면 데이터가 멀쩡히 있는데도 말이다. 반대로 진짜 없는 리소스에 `UNAVAILABLE`을 주면 클라이언트가 영원히 재시도하며 자원을 태운다. **"데이터가 없는가"(NOT_FOUND)와 "확인할 수가 없는가"(UNAVAILABLE)를 절대 혼동하지 말 것.** 한편 보안 관점에서, 권한이 없는 리소스의 존재 여부를 숨기려고 의도적으로 `PERMISSION_DENIED` 대신 `NOT_FOUND`를 쓰는 패턴도 있다(존재 자체가 기밀일 때).

### 4.3 ABORTED vs FAILED_PRECONDITION — 재시도 가능성이 가른다

이 둘은 "동시성/시퀀싱" 맥락에서 자주 마주치고, 둘의 차이는 **클라이언트가 재시도해야 하는 층위**에 있다.

```text
FAILED_PRECONDITION (9):  재시도해도 "같은 결과". 상태를 고치기 전엔 무의미.
                          → 클라이언트는 시스템 상태를 먼저 바꿔야 한다.

ABORTED (10):             "동시성 충돌"로 중단됨(트랜잭션 충돌, OCC/CAS 실패 등).
                          → 클라이언트가 "read-modify-write 시퀀스 전체를
                             다시" 시도하면 성공할 수 있다.
```

공식 가이드라인의 결정 트리를 그대로 빌리면 명쾌하다.

```text
실패했고, 클라이언트가 무엇을 해야 하는가?

(a) 재시도하지 마라 (상태 고치기 전엔 절대 안 됨)
        → FAILED_PRECONDITION

(b) 더 높은 레벨에서 재시도하라 (예: read-modify-write 전체 재시도)
        → ABORTED

(c) 그냥 재시도하라 (이 호출만 다시)
        → UNAVAILABLE
```

예를 들어 "버전 7일 때만 업데이트해줘"라는 조건부 쓰기(compare-and-swap)를 보냈는데 그 사이 누가 버전을 8로 올렸다면 → `ABORTED`. 클라이언트는 최신 버전을 다시 읽고(read) 변경을 다시 계산해(modify) 다시 쓰면(write) 된다. 반면 "이 작업은 계정이 ACTIVE 상태여야만 가능"한데 계정이 SUSPENDED라면 → `FAILED_PRECONDITION`. 계정을 활성화하기 전에는 몇 번을 재시도해도 똑같이 실패한다.

### 4.4 PERMISSION_DENIED vs UNAUTHENTICATED

영어 단어 때문에 자주 거꾸로 쓰는 쌍이다. **"인증(authentication)"은 "너 누구냐", "인가(authorization)"는 "너 그거 해도 되냐"**임을 기억하자.

```text
UNAUTHENTICATED (16):  "너가 누구인지 모르겠다 / 자격 증명이 없거나 무효다."
                       → 토큰이 아예 없음, 만료됨, 서명 검증 실패.
                       → 클라이언트는 '로그인/토큰 갱신' 후 재시도하면 됨.
                       → HTTP 401에 대응.

PERMISSION_DENIED (7): "너가 누구인지는 안다. 그런데 이 작업 권한은 없다."
                       → 인증은 성공, 인가 실패(역할/스코프 부족).
                       → 클라이언트가 토큰을 갱신해봤자 소용없음(권한 자체가 없음).
                       → HTTP 403에 대응.
```

중요한 규칙 하나: `PERMISSION_DENIED`는 **요청이 거부된 이유가 자격 증명 고갈(인증 실패)이거나 자원 고갈(쿼터)일 때 쓰면 안 된다.** 자격 증명 문제는 `UNAUTHENTICATED`, 쿼터 문제는 `RESOURCE_EXHAUSTED`로 가야 한다. `PERMISSION_DENIED`는 순수하게 "신원은 확인됐으나 인가가 거부된" 상황만을 위한 코드다. 또 하나, `PERMISSION_DENIED`는 권한 검사가 작업의 다른 전제 조건보다 **먼저** 평가되어야 한다는 함의도 있다(권한 없는 자에게 NOT_FOUND/FAILED_PRECONDITION 등으로 시스템 내부를 추론할 단서를 주지 않기 위해).

### 4.5 UNKNOWN vs INTERNAL — "버그의 두 얼굴"

이 둘은 "서버 쪽이 뭔가 잘못됐다"는 점에서 비슷하지만 발생 경위가 다르다.

```text
UNKNOWN (2):    "에러가 났는데 그것을 분류할 수가 없다."
                → 대표적으로: 서버 핸들러가 gRPC Status가 아닌
                  '날것의 예외'(RuntimeException 등)를 던져서
                  프레임워크가 코드를 못 정함 → UNKNOWN으로 떨어짐.
                → 또는 다른 주소 공간에서 온 모르는 에러 코드를 수신.
                → 즉 '에러 모델을 제대로 안 따른' 신호일 때가 많다.

INTERNAL (13):  "심각한 내부 불변식(invariant)이 깨졌다."
                → 예: protobuf 역직렬화 실패, '있어선 안 되는' 상태,
                  명백한 자기 버그를 '의식적으로' 보고할 때.
                → 클라이언트가 할 수 있는 게 없는, '서버를 고쳐야 하는' 신호.
```

실무 함정: 많은 서버가 `try/catch`로 모든 예외를 잡아 무성의하게 `INTERNAL` 또는 `UNKNOWN`으로 던진다. 그러면 클라이언트는 "음수 인자를 보낸 자기 버그(INVALID_ARGUMENT여야 함)"와 "서버 DB 다운(UNAVAILABLE이어야 함)"을 구분할 수 없다. **`INTERNAL`/`UNKNOWN` 남용은 에러 모델의 가치를 통째로 무너뜨린다.** 서버는 가능한 한 도메인 예외를 적절한 코드로 명시적으로 매핑하고, 정말로 예상 못 한 것만 `INTERNAL`로 떨어뜨려야 한다. 운영 관점에서 `UNKNOWN` 비율이 높다면 그건 "코드 매핑이 빠진 핸들러가 있다"는 알람으로 읽어야 한다.

---

## 5. 재시도 안전성과 멱등성 — 코드만으로는 부족하다

재시도(retry)는 분산 시스템 복원력의 핵심이지만, **상태 코드만 보고 재시도하는 것은 위험하다.** 코드는 "재시도해도 되는 종류인가"의 *필요조건*이지 *충분조건*이 아니다. 충분조건은 **멱등성(idempotency)** 이다.

### 5.1 재시도 가능 코드 vs 그렇지 않은 코드

코드를 재시도 관점에서 다시 묶으면:

```text
[일반적으로 재시도 안전 — 서버가 "작업을 안 했다"가 분명한 경우]
  UNAVAILABLE (14)  : 거의 항상 재시도 1순위. 보통 연결도 못 맺음/즉시 거부.
  ABORTED    (10)  : 상위 레벨(read-modify-write)에서 재시도.
  RESOURCE_EXHAUSTED (8) : '반드시' 백오프 후. RetryInfo가 있으면 그 시간을 존중.

[조건부 — 멱등할 때만 재시도]
  DEADLINE_EXCEEDED (4) : 서버가 작업을 했는지 안 했는지 '모른다'.
                          멱등 연산만 재시도 안전.
  CANCELLED  (1)  : 보통 의도된 취소라 재시도 안 함. 그러나 전파된 취소면
                    상황에 따라 재시도 고려.
  UNKNOWN    (2)  : 원인 불명. 부작용 여부 모름 → 멱등할 때만.

[재시도 무의미/금지 — 같은 요청은 또 같은 실패]
  INVALID_ARGUMENT (3), NOT_FOUND (5), ALREADY_EXISTS (6),
  PERMISSION_DENIED (7), FAILED_PRECONDITION (9),
  UNIMPLEMENTED (12), UNAUTHENTICATED (16)*  도 보통 재시도 무의미.
  INTERNAL (13), DATA_LOSS (15)  도 보통 금지(버그/손상 신호).
  (* UNAUTHENTICATED은 '토큰 갱신 후'엔 재시도 의미 있음)
```

### 5.2 왜 멱등성이 결정적인가

핵심 시나리오는 `DEADLINE_EXCEEDED`다. 클라이언트가 "주문 생성"을 보냈는데 데드라인이 초과됐다. 이때 서버는 둘 중 하나다.

```text
시나리오 A: 요청이 서버에 도달도 못함 → 주문 생성 안 됨 → 재시도해야 함
시나리오 B: 서버가 주문을 '이미 생성'하고 응답만 늦음 → 재시도하면 '중복 주문'!
```

클라이언트는 A인지 B인지 **알 수 없다**. 응답을 못 받았으니까. 따라서:

- **멱등(idempotent) 연산**(같은 요청을 여러 번 해도 결과가 같음, 예: `GetUser`, `DeleteOrder(id)`, `SetStatus(x)`)은 안심하고 재시도할 수 있다. 중복으로 실행돼도 무해하다.
- **비멱등(non-idempotent) 연산**(예: 멱등키 없는 `CreateOrder`, `IncrementCounter`)은 재시도하면 중복 부작용을 일으킨다.

따라서 자동 재시도를 켤 때는 **메서드별로 멱등성을 표시**해야 한다. 실무에서는 (1) 본질적으로 멱등한 메서드만 재시도 대상으로 지정하거나, (2) 비멱등 메서드에 **멱등 키(idempotency key)** 를 도입해 서버가 중복을 흡수하게 만든다(같은 키의 재요청은 첫 결과를 그대로 반환). gRPC의 클라이언트 재시도 설정(서비스 config의 `retryPolicy`, `retryableStatusCodes`)과 그 메커니즘은 [[13 - 안정성 - Retry Health Check Keepalive]]에서 자세히 다룬다. 이 장에서 기억할 핵심은 단 하나: **"코드가 재시도 가능 종류다"와 "이 호출을 재시도해도 안전하다"는 별개이며, 후자는 멱등성이 보증한다.**

### 5.3 RetryInfo로 "언제 재시도할지"를 서버가 지시하다

`RESOURCE_EXHAUSTED`나 `UNAVAILABLE`을 줄 때, 서버는 종종 "얼마나 기다렸다 재시도하라"를 알고 있다(예: 쿼터 리셋까지 30초). 이걸 클라이언트가 추측(지수 백오프)하게 두지 말고 서버가 직접 알려주는 게 좋다. 그것이 다음 절에서 볼 `RetryInfo` detail이다. 잘 만든 클라이언트는 `RetryInfo.retry_delay`가 있으면 자체 백오프 대신 그 값을 존중한다.

---

## 6. Rich Error Model — code 하나로는 부족할 때

`code + message`만으로는 종종 부족하다. 예를 들어 "주문 생성"이 `INVALID_ARGUMENT`로 실패했다고 하자. 클라이언트(특히 사용자에게 폼을 보여주는 프론트엔드)는 **어느 필드가, 왜 틀렸는지**를 정확히 알아야 한다. 메시지 문자열을 정규식으로 파싱하라고? 그건 깨지기 쉽고(메시지는 언제든 바뀐다) 국제화(i18n)도 안 된다. 그래서 **Rich Error Model**이 필요하다.

Rich Error Model의 핵심은 이미 본 `google.rpc.Status`다. 그 `details` 필드에 **표준화된, 타입이 있는** detail 메시지들을 담는다. 이 표준 타입들은 `google/rpc/error_details.proto`에 정의돼 있고, 모든 언어 구현이 공유한다. 직접 임의의 메시지를 정의해 넣을 수도 있지만, 표준 타입을 쓰면 다른 팀·도구·라이브러리가 곧바로 이해한다.

### 6.1 표준 detail 타입 카탈로그

각 타입은 "언제 붙이는가"가 다르다. 아래 표가 의사결정의 출발점이다.

| detail 타입 | 무엇을 담나 | 언제 붙이나(전형 코드) |
|-------------|-------------|------------------------|
| **ErrorInfo** | `reason`(머신용 enum형 문자열), `domain`(서비스 식별자), `metadata`(맵) | 거의 모든 에러. **프로그램적 분기의 기준점.** |
| **RetryInfo** | `retry_delay`(Duration) | `RESOURCE_EXHAUSTED`, `UNAVAILABLE` — "이만큼 후 재시도" |
| **DebugInfo** | `stack_entries`, `detail` | 디버깅용 스택. **외부 노출 주의(보통 내부 전용).** |
| **QuotaFailure** | `violations[]`(subject, description) | `RESOURCE_EXHAUSTED` — 어떤 쿼터를 초과했나 |
| **PreconditionFailure** | `violations[]`(type, subject, description) | `FAILED_PRECONDITION` — 어떤 전제가 안 맞나 |
| **BadRequest** | `field_violations[]`(field, description) | `INVALID_ARGUMENT` — 어느 필드가 왜 틀렸나 |
| **RequestInfo** | `request_id`, `serving_data` | 임의 — 지원/추적용 요청 식별 |
| **ResourceInfo** | `resource_type`, `resource_name`, `owner`, `description` | `NOT_FOUND`, `ALREADY_EXISTS`, `PERMISSION_DENIED` — 어떤 리소스인가 |
| **Help** | `links[]`(description, url) | 임의 — 해결 방법 문서 링크 |
| **LocalizedMessage** | `locale`, `message` | 임의 — 사용자에게 보여줄 현지화된 메시지 |

이 타입들의 실제 proto 정의를 보면 구조가 명확해진다.

```protobuf
// google/rpc/error_details.proto (요지)

// 가장 중요한 타입. "기계가 분기하는 안정적 식별자".
message ErrorInfo {
  // 안정적이고 도메인 내에서 유일한 머신용 이유. UPPER_SNAKE_CASE 권장.
  // 예: "USER_NOT_FOUND", "QUOTA_EXCEEDED", "STOCK_INSUFFICIENT"
  string reason = 1;
  // 이 reason을 정의한 주체. 보통 서비스/도메인 이름.
  // 예: "orders.example.com"
  string domain = 2;
  // 추가 구조화 정보(키-값). 예: {"order_id":"42","sku":"ABC"}
  map<string, string> metadata = 3;
}

message RetryInfo {
  // 이 시간만큼 기다린 후 재시도하라.
  google.protobuf.Duration retry_delay = 1;
}

message BadRequest {
  message FieldViolation {
    string field = 1;        // 위반 필드 경로. 예: "items[0].quantity"
    string description = 2;  // 사람이 읽는 설명
    // (최신 정의엔 reason, localized_message도 포함될 수 있음)
  }
  repeated FieldViolation field_violations = 1;
}

message PreconditionFailure {
  message Violation {
    string type = 1;         // 전제 종류. 예: "TOS" / "ACCOUNT_STATE"
    string subject = 2;      // 대상. 예: "user:42"
    string description = 3;  // 설명. 예: "약관에 동의해야 합니다."
  }
  repeated Violation violations = 1;
}

message QuotaFailure {
  message Violation {
    string subject = 1;      // 예: "project:my-proj"
    string description = 2;  // 예: "일일 요청 쿼터 1000 초과"
  }
  repeated Violation violations = 1;
}

message ResourceInfo {
  string resource_type = 1;  // 예: "order"
  string resource_name = 2;  // 예: "orders/42"
  string owner = 3;
  string description = 4;
}

message LocalizedMessage {
  string locale = 1;         // BCP-47. 예: "ko-KR"
  string message = 2;        // 현지화된 사용자 노출 메시지
}
```

### 6.2 ErrorInfo가 왜 가장 중요한가

세 겹 모델을 다시 떠올리자. `code`는 너무 거칠다(16개뿐). `message`는 파싱하면 안 된다(불안정). 그 사이의 빈틈을 메우는 것이 **`ErrorInfo.reason`** 이다.

`reason`은 "도메인 안에서 안정적이고 유일한, 대문자 스네이크 케이스의 머신용 식별자"다. `code=INVALID_ARGUMENT` 하나로는 "이메일 형식 오류"인지 "수량 음수"인지 알 수 없지만, `reason="EMAIL_MALFORMED"` vs `reason="QUANTITY_NEGATIVE"`면 클라이언트가 **switch 문으로 분기**할 수 있다. 그리고 이 값은 (`message`와 달리) API 계약의 일부로서 함부로 바꾸지 않기로 약속한다. `domain`은 같은 `reason`이라도 어느 서비스가 낸 것인지 구분해 충돌을 막는다.

이것이 [[02 - Protocol Buffers 1 - 문법과 타입 시스템]]에서 본 "스키마로 계약을 박는다"는 철학과 정확히 같은 정신이다 — 문자열 메시지가 아니라 **타입과 안정적 식별자**로 계약한다.

---

## 7. 코드로 보기 — 붙이고, 꺼내기

이론을 실제 API로 옮긴다. 두 시나리오를 쓴다: **사용자 조회(NOT_FOUND + ResourceInfo)** 와 **주문 생성(INVALID_ARGUMENT + BadRequest, FAILED_PRECONDITION + PreconditionFailure)**.

### 7.1 proto 정의

```protobuf
syntax = "proto3";
package shop.v1;

import "google/protobuf/timestamp.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest  { int64 user_id = 1; }
message GetUserResponse { int64 user_id = 1; string name = 2; string email = 3; }

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
}

message CreateOrderRequest {
  int64 user_id = 1;
  repeated LineItem items = 2;
  string idempotency_key = 3;   // 비멱등 방지용
}
message LineItem { string sku = 1; int32 quantity = 2; }
message CreateOrderResponse {
  string order_id = 1;
  google.protobuf.Timestamp created_at = 2;
}
```

여기서 응답 메시지에 별도의 에러 필드가 **없다**는 점에 주목하자. gRPC에서는 에러를 응답 본문이 아니라 `Status`(트레일러)로 보내는 것이 정통이다. (응답 메시지 안에 `oneof { data; error }`를 두는 대안 패턴은 7.5절에서 일반론으로 비교한다.)

### 7.2 서버: Go — NOT_FOUND에 ResourceInfo 붙이기

```go
import (
    "context"

    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    errdetails "google.golang.org/genproto/googleapis/rpc/errdetails"
)

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    // 1) 인자 자체 검증 → INVALID_ARGUMENT (상태와 무관하게 틀린 값)
    if req.GetUserId() <= 0 {
        st := status.New(codes.InvalidArgument, "user_id must be positive")
        st, _ = st.WithDetails(&errdetails.BadRequest{
            FieldViolations: []*errdetails.BadRequest_FieldViolation{{
                Field:       "user_id",
                Description: "must be > 0",
            }},
        })
        return nil, st.Err()
    }

    u, err := s.repo.Find(ctx, req.GetUserId())
    if errors.Is(err, ErrNotFound) {
        // 2) 리소스 부재 → NOT_FOUND + ResourceInfo + ErrorInfo
        st := status.New(codes.NotFound, "user not found")
        st, _ = st.WithDetails(
            &errdetails.ResourceInfo{
                ResourceType: "user",
                ResourceName: fmt.Sprintf("users/%d", req.GetUserId()),
                Description:  "no such user",
            },
            &errdetails.ErrorInfo{
                Reason:   "USER_NOT_FOUND",          // 머신용 안정 식별자
                Domain:   "users.shop.example.com",
                Metadata: map[string]string{"user_id": fmt.Sprint(req.GetUserId())},
            },
        )
        return nil, st.Err()
    }
    if err != nil {
        // 3) 예상 못 한 인프라 오류 → 정직하게 분류. DB 다운이면 UNAVAILABLE.
        //    절대 NOT_FOUND로 위장하지 말 것(클라가 재시도를 포기함).
        return nil, status.Error(codes.Unavailable, "user store temporarily unavailable")
    }

    return &pb.GetUserResponse{UserId: u.ID, Name: u.Name, Email: u.Email}, nil
}
```

`status.New(code, msg)`로 기본 `Status`를 만들고 `WithDetails(...)`로 Rich detail을 *얹는다*. 내부적으로 `WithDetails`는 각 detail을 `Any`로 패킹해 `google.rpc.Status.details`에 추가한다. 마지막에 `st.Err()`가 `error` 인터페이스를 구현하는 값을 돌려주고, gRPC 런타임이 이를 `grpc-status`/`grpc-message`/`grpc-status-details-bin` 트레일러로 직렬화한다.

### 7.3 서버: 주문 생성 — INVALID_ARGUMENT와 FAILED_PRECONDITION의 분기

```go
func (s *orderServer) CreateOrder(ctx context.Context, req *pb.CreateOrderRequest) (*pb.CreateOrderResponse, error) {
    // (A) 인자 검증: 여러 필드 위반을 모아서 한 번에 BadRequest로
    var fv []*errdetails.BadRequest_FieldViolation
    if len(req.GetItems()) == 0 {
        fv = append(fv, &errdetails.BadRequest_FieldViolation{
            Field: "items", Description: "at least one item is required"})
    }
    for i, it := range req.GetItems() {
        if it.GetQuantity() <= 0 {
            fv = append(fv, &errdetails.BadRequest_FieldViolation{
                Field:       fmt.Sprintf("items[%d].quantity", i),
                Description: "must be > 0",
            })
        }
    }
    if len(fv) > 0 {
        st := status.New(codes.InvalidArgument, "invalid CreateOrder request")
        st, _ = st.WithDetails(&errdetails.BadRequest{FieldViolations: fv})
        return nil, st.Err()
    }

    // (B) 상태 전제: 계정이 정지면 인자는 멀쩡해도 작업 불가 → FAILED_PRECONDITION
    acct, _ := s.accounts.Get(ctx, req.GetUserId())
    if acct.Status == Suspended {
        st := status.New(codes.FailedPrecondition, "account is suspended")
        st, _ = st.WithDetails(&errdetails.PreconditionFailure{
            Violations: []*errdetails.PreconditionFailure_Violation{{
                Type:        "ACCOUNT_STATE",
                Subject:     fmt.Sprintf("users/%d", req.GetUserId()),
                Description:  "account must be ACTIVE to place orders",
            }},
        })
        return nil, st.Err()
    }

    // (C) 동시성/재고: 재고 차감 트랜잭션이 충돌하면 → ABORTED(상위 재시도 권장)
    order, err := s.tryReserveAndCreate(ctx, req)
    if errors.Is(err, ErrConcurrencyConflict) {
        return nil, status.Error(codes.Aborted, "inventory changed during reservation; retry")
    }
    // (D) 재고 부족: 인자/상태는 OK지만 비즈니스 제약 위반 → FAILED_PRECONDITION
    if errors.Is(err, ErrOutOfStock) {
        st := status.New(codes.FailedPrecondition, "insufficient stock")
        st, _ = st.WithDetails(&errdetails.ErrorInfo{
            Reason: "STOCK_INSUFFICIENT", Domain: "orders.shop.example.com",
            Metadata: map[string]string{"sku": err.(*StockError).SKU},
        })
        return nil, st.Err()
    }
    if err != nil {
        return nil, status.Error(codes.Internal, "failed to create order")
    }
    return &pb.CreateOrderResponse{OrderId: order.ID, CreatedAt: timestamppb.New(order.CreatedAt)}, nil
}
```

이 핸들러 하나가 4.1~4.3절의 구분을 그대로 코드로 보여준다. 같은 "주문 생성 실패"라도 (A)는 호출자 인자 문제(`INVALID_ARGUMENT`), (B)는 시스템 상태 문제(`FAILED_PRECONDITION`), (C)는 동시성 충돌(`ABORTED`, 재시도 권장), (D)는 비즈니스 제약(`FAILED_PRECONDITION`)으로 정밀하게 갈린다.

### 7.4 클라이언트에서 꺼내기 — Go / Java / Python

서버가 정성껏 붙인 details를 클라이언트가 못 꺼내면 무용지물이다. 언어별 추출 API를 본다.

**Go** — `status.FromError`로 `Status`를 복원하고 `Details()`로 detail 슬라이스를 순회한다.

```go
resp, err := client.CreateOrder(ctx, req)
if err != nil {
    st, ok := status.FromError(err)
    if !ok {
        // gRPC Status가 아닌 에러(컨텍스트 취소 등). 별도 처리.
        return
    }
    switch st.Code() {
    case codes.InvalidArgument:
        for _, d := range st.Details() {
            switch v := d.(type) {
            case *errdetails.BadRequest:
                for _, fvio := range v.GetFieldViolations() {
                    log.Printf("field %s: %s", fvio.GetField(), fvio.GetDescription())
                    // → 프론트엔드 폼에서 해당 필드에 에러 표시
                }
            }
        }
    case codes.FailedPrecondition:
        // ErrorInfo.reason으로 프로그램적 분기
        for _, d := range st.Details() {
            if ei, ok := d.(*errdetails.ErrorInfo); ok && ei.GetReason() == "STOCK_INSUFFICIENT" {
                sku := ei.GetMetadata()["sku"]
                notifyRestock(sku)
            }
        }
    case codes.Unavailable, codes.Aborted:
        // 재시도 후보 (멱등성/idempotency_key 확인 후)
    }
}
```

**Java** — gRPC 자바에서 details는 `com.google.rpc.Status`(protobuf 메시지)로 다뤄진다. `StatusProto.fromThrowable`로 꺼내고, 각 `Any`를 `unpack`한다.

```java
import com.google.rpc.Status;          // protobuf 메시지 (google.rpc.Status)
import com.google.rpc.BadRequest;
import com.google.rpc.ErrorInfo;
import io.grpc.protobuf.StatusProto;    // gRPC <-> google.rpc.Status 브리지

try {
    CreateOrderResponse resp = blockingStub.createOrder(req);
} catch (StatusRuntimeException e) {
    Status status = StatusProto.fromThrowable(e);   // google.rpc.Status
    if (status != null) {
        for (com.google.protobuf.Any any : status.getDetailsList()) {
            if (any.is(BadRequest.class)) {
                BadRequest br = any.unpack(BadRequest.class);
                for (BadRequest.FieldViolation fv : br.getFieldViolationsList()) {
                    System.out.printf("%s: %s%n", fv.getField(), fv.getDescription());
                }
            } else if (any.is(ErrorInfo.class)) {
                ErrorInfo ei = any.unpack(ErrorInfo.class);
                System.out.println("reason=" + ei.getReason());
            }
        }
    }
}
```

서버 쪽 자바에서 details를 붙일 때도 같은 브리지를 쓴다.

```java
com.google.rpc.Status rpcStatus = com.google.rpc.Status.newBuilder()
    .setCode(io.grpc.Status.Code.NOT_FOUND.value())   // 5
    .setMessage("user not found")
    .addDetails(com.google.protobuf.Any.pack(
        ErrorInfo.newBuilder()
            .setReason("USER_NOT_FOUND")
            .setDomain("users.shop.example.com")
            .putMetadata("user_id", String.valueOf(id))
            .build()))
    .build();
responseObserver.onError(StatusProto.toStatusRuntimeException(rpcStatus));
```

`StatusProto`가 내부적으로 `google.rpc.Status`를 직렬화해 `grpc-status-details-bin` 메타데이터 키에 넣는다. 메타데이터 키를 직접 다룰 일은 드물지만, 키 이름이 `grpc-status-details-bin`이고 값이 바이너리(`-bin` 규칙)라는 것을 알면 [[14 - 관찰성과 디버깅 - Reflection grpcurl]]에서 trailer를 직접 들여다볼 때 도움이 된다.

**Python** — `grpcio-status` 패키지의 `rpc_status.from_call`을 쓴다.

```python
from grpc_status import rpc_status
from google.rpc import error_details_pb2
import grpc

try:
    resp = stub.CreateOrder(req)
except grpc.RpcError as rpc_error:
    status = rpc_status.from_call(rpc_error)   # google.rpc.Status
    if status is not None:
        for detail in status.details:
            if detail.Is(error_details_pb2.BadRequest.DESCRIPTOR):
                br = error_details_pb2.BadRequest()
                detail.Unpack(br)
                for v in br.field_violations:
                    print(f"{v.field}: {v.description}")
            elif detail.Is(error_details_pb2.ErrorInfo.DESCRIPTOR):
                ei = error_details_pb2.ErrorInfo()
                detail.Unpack(ei)
                print("reason:", ei.reason)
```

세 언어 모두 패턴이 같다: **`error` → `google.rpc.Status` 복원 → `details`의 각 `Any`를 `Is/unpack`으로 알려진 타입으로 풀기.** 이 대칭성이 Rich Error Model의 언어 중립성을 증명한다.

### 7.5 oneof { data; error } 패턴과의 비교 — 일반론

응답 메시지 안에 에러를 직접 모델링하는 대안도 있다.

```protobuf
message CreateOrderResponse {
  oneof result {
    Data  data  = 1;
    Error error = 2;
  }
  message Data  { string order_id = 1; }
  message Error {
    oneof kind {
      OutOfStock       out_of_stock       = 1;
      AccountSuspended account_suspended  = 2;
    }
  }
}
```

이 패턴의 장점은 **컴파일 타임 타입 안전성**이다. 가능한 도메인 에러가 proto에 열거되므로 클라이언트가 `when/switch`로 **빠짐없이(exhaustive)** 처리할 수 있고, 새 에러를 추가하면 컴파일러가 미처리 분기를 잡아준다. 함수형 언어(Scala의 `Either`, Kotlin의 sealed class)와 궁합이 좋다.

표준 `Status`/Rich Error와의 트레이드오프를 일반론으로 정리하면:

```text
                     gRPC Status + Rich Error       응답 oneof { data; error }
호출 결과 위치        트레일러(전송 계층 표준)         응답 본문(애플리케이션 계층)
인프라 친화성          ★ 로드밸런서/프록시/재시도가      △ 인프라는 grpc-status=OK로 보임
                       grpc-status로 즉시 인식          (도메인 실패를 인프라가 모름)
타입 안전/exhaustive   △ details는 런타임 언패킹         ★ 컴파일 타임 강제
재시도 자동화 연동      ★ retryableStatusCodes와 직결     △ 본문을 까봐야 판단 가능
언어 간 표준성          ★ 모든 gRPC 구현 공통             △ 팀이 직접 설계/공유
```

핵심 판단 기준: **"이 실패를 인프라(재시도/로드밸런서)가 알아야 하는가, 아니면 호출자 코드만 알면 되는가."** `UNAVAILABLE`처럼 인프라가 재시도·페일오버를 결정해야 하는 전송/가용성 실패는 반드시 `Status` 코드로 표현해야 한다(본문 oneof로 숨기면 인프라가 못 본다). 반면 "재고 부족" 같이 호출자만 분기하면 되고 무조건 성공 트랜잭션으로 끝나는(서버는 정상 동작한) 도메인 결과는 oneof로 모델링하는 것도 합리적이다. 많은 팀이 **둘을 섞어** 쓴다 — 가용성/인프라/일반 도메인 실패는 표준 `Status`(+ErrorInfo), 풍부하게 분기해야 하는 핵심 도메인 결과는 응답 oneof. 정답은 도메인마다 다르며, 이 책은 어느 한쪽을 강요하지 않는다.

---

## 8. HTTP/2 스트림 에러와 gRPC Status는 다른 층위다

여기서 초보자가 가장 많이 헷갈리는 지점을 짚는다. **HTTP/2 자체의 에러(RST_STREAM, GOAWAY)와 gRPC의 `Status`는 완전히 다른 계층이다.** ([[04 - HTTP2 깊이 보기 - 전송 계층]]의 프레임 모델을 전제로 한다.)

```text
계층 그림:

  ┌───────────────────────────────────────────────┐
  │  gRPC 애플리케이션 계층                          │
  │   - Status(code/message/details)                │
  │   - grpc-status / grpc-message / -details-bin    │  ← "도메인/호출 결과"
  └───────────────────────────────────────────────┘
  ┌───────────────────────────────────────────────┐
  │  HTTP/2 전송 계층                                 │
  │   - HEADERS / DATA / RST_STREAM / GOAWAY 프레임    │  ← "전송 자체의 문제"
  │   - HTTP/2 error code (NO_ERROR, CANCEL,         │
  │     REFUSED_STREAM, INTERNAL_ERROR, ...)         │
  └───────────────────────────────────────────────┘
```

정상적인 gRPC 호출은 전송 계층 에러 없이 끝난다. 서버가 `grpc-status: 7`(PERMISSION_DENIED) 트레일러를 보내고 스트림을 `END_STREAM`으로 정상 종료한다 — HTTP/2 레벨에서는 아무 에러도 아니다(에러는 gRPC 트레일러 안에만 있다).

반대로 HTTP/2 전송 계층에서 문제가 생기면 `Status` 트레일러가 **아예 오지 않을 수 있다**. 대표 사례:

- 클라이언트가 호출을 취소하면 클라이언트는 해당 스트림에 **`RST_STREAM`** 프레임(에러 코드 `CANCEL = 0x8`)을 보낸다. 서버 입장에서 이 호출은 `CANCELLED`(1)로 끝난다.
- 서버가 새 스트림을 받자마자 거부하면 **`RST_STREAM`(`REFUSED_STREAM = 0x7`)**. gRPC는 이를 보통 `UNAVAILABLE`(14)로 매핑한다. `REFUSED_STREAM`은 "서버가 이 요청을 *처리 시작도 안 했다*"는 강한 보장을 주므로, 비멱등 연산이라도 **안전하게 재시도** 가능한 특별한 경우다.
- 서버가 `GOAWAY` 프레임을 보내며 연결을 닫으면(graceful shutdown/재배포), 진행 중이던 스트림 중 `GOAWAY`의 last-stream-id를 초과한 것들은 처리 안 된 것이 보장되어 재시도 안전하다.
- 트레일러가 도착하기 전에 TCP 연결이 끊기면 → `UNAVAILABLE`.

요약하면 **두 갈래의 실패 경로**가 있다.

```text
경로 A (정상 종료 + 도메인 실패):
   서버 핸들러 도달 → 처리 → grpc-status != 0 트레일러로 결과 전달
   → "서버가 의도적으로 분류한" 에러. code가 도메인 의미를 가짐.

경로 B (전송 계층 실패):
   핸들러 도달 전/도중에 전송이 깨짐(RST_STREAM/GOAWAY/연결 끊김/타임아웃)
   → gRPC 라이브러리가 '클라이언트 측에서' 적절한 code를 합성
   → 보통 UNAVAILABLE / CANCELLED / DEADLINE_EXCEEDED
   → 이 code들은 "서버가 보낸 것"이 아니라 "클라이언트가 추론한 것"
```

이 구분이 실무에서 왜 중요한가? 운영 중 `UNAVAILABLE`을 봤을 때, 그것이 (1) 서버 핸들러가 의식적으로 던진 것인지, (2) 전송이 끊겨 클라이언트가 합성한 것인지에 따라 디버깅 방향이 완전히 달라지기 때문이다. 후자라면 서버 로그에는 해당 호출 자체가 없을 수도 있다. 또한 재시도 안전성 판단에도 직결된다 — `REFUSED_STREAM`/`GOAWAY` 기반의 `UNAVAILABLE`은 "처리 안 됨"이 보장되어 비멱등 연산도 재시도 가능한 반면, 일반적인 연결 끊김 기반 `UNAVAILABLE`은 "처리됐는지 모름"이라 멱등성을 따져야 한다.

---

## 9. 모범 사례 — 좋은 에러 API를 설계하는 법

지금까지의 내용을 실천 가능한 규칙으로 압축한다.

### 9.1 코드 선택은 "클라이언트가 무엇을 해야 하는가"로 결정하라

에러 코드는 서버의 내부 사정을 묘사하는 게 아니라 **클라이언트의 다음 행동을 안내**하는 신호다. 그래서 4.3절의 결정 트리("재시도 금지 → FAILED_PRECONDITION / 상위 재시도 → ABORTED / 그냥 재시도 → UNAVAILABLE")가 강력하다. 코드를 고를 때 항상 자문하라: "이 코드를 받은 클라이언트가 *올바르게* 행동할 수 있는가?"

### 9.2 INTERNAL/UNKNOWN을 남용하지 말라

`catch (Exception e) { return INTERNAL; }`는 에러 모델을 죽인다. 도메인 예외를 빠짐없이 적절한 코드로 매핑하고, 정말 예상 못 한 것만 `INTERNAL`로 보내라. `UNKNOWN` 비율이 높으면 "코드 매핑이 빠진 핸들러가 있다"는 운영 알람으로 취급하라.

### 9.3 message에 민감 정보를 넣지 말라

`grpc-message`와 details는 그대로 클라이언트로 간다. 스택 트레이스, 내부 호스트명/IP, SQL 쿼리, PII, 다른 사용자의 데이터를 넣지 말라. 디버깅용 스택은 `DebugInfo`에 담되, **외부 클라이언트에는 제거**하고 내부 호출(혹은 로그)에만 남기는 인터셉터([[08 - 메타데이터와 인터셉터]])를 두는 것이 정석이다. 보안 측면에서, 권한 없는 리소스의 존재 여부를 누설하지 않으려 `PERMISSION_DENIED` 대신 `NOT_FOUND`를 의도적으로 쓰는 선택도 있다.

### 9.4 프로그램적 분기는 ErrorInfo.reason으로

클라이언트가 메시지 문자열을 파싱하게 만들지 말라. `code`로 거친 분기, **`ErrorInfo.reason`(+`domain`)으로 정밀 분기**, `BadRequest`/`PreconditionFailure`로 필드/전제 단위 정보, `LocalizedMessage`로 사용자 표시 문구를 제공하라. `reason` 값은 API 계약의 일부이므로 한번 정하면 안정적으로 유지하라(추가는 OK, 변경/삭제는 [[17 - 실전 - proto 관리와 하위호환성]]의 호환성 원칙을 따른다).

### 9.5 재시도/백오프 힌트는 RetryInfo로

`RESOURCE_EXHAUSTED`/`UNAVAILABLE`에서 서버가 회복 시점을 알면 `RetryInfo.retry_delay`로 명시하라. 클라이언트는 자체 지수 백오프보다 이 값을 우선 존중한다. 이것은 thundering herd(동시 재시도 폭주)를 줄이는 실용적 장치다.

### 9.6 detail 타입과 코드의 자연스러운 짝

마지막으로 "어떤 코드엔 어떤 detail이 어울리나"를 정리한다.

```text
INVALID_ARGUMENT     → BadRequest(FieldViolation[])  (+ ErrorInfo)
FAILED_PRECONDITION  → PreconditionFailure(Violation[])  (+ ErrorInfo)
RESOURCE_EXHAUSTED   → QuotaFailure(Violation[]) + RetryInfo  (+ ErrorInfo)
UNAVAILABLE          → RetryInfo
NOT_FOUND            → ResourceInfo  (+ ErrorInfo)
ALREADY_EXISTS       → ResourceInfo  (+ ErrorInfo)
PERMISSION_DENIED    → ErrorInfo  (가능하면 노출 최소화)
(모든 에러 공통)      → ErrorInfo(reason/domain), RequestInfo, Help, LocalizedMessage
```

`ErrorInfo`는 거의 모든 에러에 붙일 가치가 있는 "기본 장착" detail이고, 나머지는 코드의 성격에 맞춰 선택적으로 얹는다.

---

## 10. 정리 — 세 겹과 두 채널, 그리고 한 가지 질문

이 장의 모델을 한 그림으로 압축한다.

```text
                     gRPC 호출 1회의 종료

   [세 겹의 Status]                  [두 개의 와이어 채널]
   ① code (enum)        ──────────►  grpc-status        (텍스트 트레일러)
   ② message (string)   ──────────►  grpc-message       (퍼센트 인코딩)
   ③ details (Any[])    ──────────►  grpc-status-details-bin
                                       └ base64( google.rpc.Status )
                                          └ code/message(중복) + details(Any[])

   읽는 주체
   - 인프라(LB/프록시/재시도) :  ① grpc-status 만으로 충분
   - 사람(로그/디버깅)        :  ②
   - 클라이언트 코드(분기)    :  ① + ③(ErrorInfo.reason / BadRequest / ...)
```

그리고 모든 코드 선택을 관통하는 한 가지 질문으로 끝맺는다.

> **"이 에러를 받은 클라이언트가, 이 `code`와 `details`만 보고 올바른 다음 행동을 결정할 수 있는가?"**

이 질문에 "예"라고 답할 수 있으면 좋은 에러 API다. 코드는 클라이언트의 행동을 안내하고, ErrorInfo는 정밀 분기를 가능하게 하고, BadRequest/PreconditionFailure는 무엇을 고쳐야 하는지 알려주며, RetryInfo는 언제 다시 시도할지 말해 준다. 에러를 예외가 아니라 **일급 반환값**으로 설계하는 것 — 그것이 견고한 분산 시스템 API의 출발점이다.

---

## 핵심 요약

- gRPC의 모든 호출은 예외 없이 하나의 **`Status = code(enum) + message(string) + details(Any[])`**로 끝난다. 성공조차 `code=OK(0)`인 Status이며, 와이어에서 `grpc-status: 0` 트레일러로 명시된다.
- 최종 상태를 **HTTP/2 트레일러**에 두는 설계 덕분에, 스트리밍 도중 실패도 정직하게 마지막에 선언할 수 있다. 빠른 실패는 헤더+트레일러를 합친 **Trailers-Only** 응답으로 최적화된다. 도메인 실패여도 HTTP `:status`는 보통 `200`이다.
- `message`는 ASCII 안전을 위해 **퍼센트 인코딩**되고, 구조화된 `details`는 **`grpc-status-details-bin` 바이너리 트레일러**에 `google.rpc.Status`(code/message/Any[])를 직렬화·base64로 실어 보낸다. 이 채널을 모르는 클라이언트도 `grpc-status`만으로 핵심 code를 잃지 않는다(하위 호환).
- **16개 코드(OK 포함 17개)**는 닫힌 집합이며 모든 언어가 공유한다. 헷갈리는 쌍의 판별 축은 "인자 자체 문제(INVALID_ARGUMENT) vs 상태 전제 문제(FAILED_PRECONDITION) vs 상태 의존 범위(OUT_OF_RANGE)", "데이터 없음(NOT_FOUND) vs 확인 불가(UNAVAILABLE)", "상태 고쳐야 함(FAILED_PRECONDITION) vs 동시성 충돌로 상위 재시도(ABORTED)", "인증 실패(UNAUTHENTICATED, 401) vs 인가 실패(PERMISSION_DENIED, 403)"다.
- 재시도는 **코드(필요조건) + 멱등성(충분조건)**의 곱이다. `UNAVAILABLE`/`ABORTED`는 전형적 재시도 대상이지만, `DEADLINE_EXCEEDED`처럼 부작용 여부를 모르는 경우는 멱등 연산(또는 멱등 키)만 안전하다. `REFUSED_STREAM`/`GOAWAY` 기반 실패는 "미처리"가 보장되어 비멱등도 재시도 가능하다.
- **Rich Error Model**은 `google.rpc.Status.details`에 표준 타입(ErrorInfo, RetryInfo, BadRequest, PreconditionFailure, QuotaFailure, ResourceInfo, DebugInfo, RequestInfo, Help, LocalizedMessage)을 담는다. 그중 **ErrorInfo(reason/domain/metadata)**가 code와 message 사이의 빈틈을 메우는 "프로그램적 분기의 안정적 식별자"로 가장 중요하다.
- **HTTP/2 전송 에러(RST_STREAM/GOAWAY)와 gRPC Status는 다른 층위**다. 서버가 의도적으로 분류한 code(경로 A)와 전송이 깨져 클라이언트가 합성한 code(경로 B)를 구분해야 디버깅과 재시도 판단이 정확해진다.
- 모범 사례: 코드는 "클라이언트가 무엇을 해야 하는가"로 고르고, INTERNAL/UNKNOWN을 남용하지 말며, message/details에 민감정보를 넣지 말고, 프로그램적 분기는 ErrorInfo.reason으로 제공하라.

---

## 연결 노트

- [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]] — `google.rpc.Status`/`Any`의 바이트 직렬화(varint·length-delimited 태그) 디코딩의 기초.
- [[04 - HTTP2 깊이 보기 - 전송 계층]] — 트레일러(HEADERS+END_STREAM), Trailers-Only, RST_STREAM/GOAWAY 등 에러가 실제로 흐르는 전송 메커니즘.
- [[02 - Protocol Buffers 1 - 문법과 타입 시스템]] — `oneof`, `Any`, enum 등 에러 모델링에 쓰는 타입 도구.
- [[08 - 메타데이터와 인터셉터]] — `-bin` 메타데이터 규칙, 에러 변환/민감정보 제거 인터셉터.
- [[10 - Deadline 취소 타임아웃]] — `DEADLINE_EXCEEDED`/`CANCELLED`가 생성되는 메커니즘과 전파.
- [[13 - 안정성 - Retry Health Check Keepalive]] — `retryPolicy`/`retryableStatusCodes` 설정과 멱등성 기반 자동 재시도.
- [[14 - 관찰성과 디버깅 - Reflection grpcurl]] — grpcurl로 trailer와 status-details-bin을 직접 들여다보기.
- [[16 - gRPC-Web과 REST 게이트웨이]] — gRPC 코드 ↔ HTTP 상태 매핑이 게이트웨이에서 실제로 적용되는 곳.
- [[17 - 실전 - proto 관리와 하위호환성]] — ErrorInfo.reason과 detail 타입을 계약으로 유지·진화시키는 호환성 원칙.
