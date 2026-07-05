---
title: "통신의 4가지 방식 - Unary와 Streaming"
date: 2026-06-26
tags:
  - grpc
  - 학습노트
  - unary
  - streaming
  - server-streaming
  - client-streaming
  - bidirectional
  - http2
  - rpc
  - backpressure
  - half-close
  - trailers
---
## 이 장이 답하는 질문

- 네 가지 RPC 유형은 무엇이 다른가, 그리고 proto의 `stream` 키워드 위치가 정확히 무엇을 바꾸는가?
- Unary조차 "스트림 위의 메시지 1개"라는 말은 무슨 뜻이며, HTTP/2 프레임 수준에서는 어떤 일이 벌어지는가?
- half-close(클라이언트가 "나는 더 안 보낸다"라고 알리는 것)와 trailer(응답의 최종 상태)는 각 유형에서 언제 발생하는가?
- 한 스트림 안에서 메시지 순서는 보장되는데, 왜 스트림 사이에서는 보장되지 않는가?
- bidirectional 스트림에서 보내기와 받기를 동시에 다룰 때 데드락은 왜 생기고 어떻게 피하는가?
- 언제 server streaming을, 언제 client streaming을, 언제 bidi를 골라야 하는가? 그리고 어떤 사용이 안티패턴인가?

---

## 들어가며 — "네 가지"가 아니라 "한 가지를 네 방향에서 본 것"

gRPC를 처음 배우는 사람들은 거의 항상 이 네 가지를 별개의 기능처럼 외운다. "Unary는 보통 함수 호출이고, server streaming은 서버가 여러 개 주는 거고, client streaming은 클라이언트가 여러 개 주는 거고, bidi는 둘 다 여러 개 주는 거다." 이렇게 외우면 시험은 통과하지만 실제로 스트림이 중간에 끊기거나 데드락이 걸리거나 부분 처리가 발생했을 때 무슨 일이 벌어지는지는 전혀 설명하지 못한다.

핵심을 먼저 못 박자. **gRPC의 모든 호출은 예외 없이 HTTP/2 스트림(stream) 하나 위에서 일어난다.** 그리고 그 스트림 위에서는 양방향 모두 "길이가 앞에 붙은 메시지(length-prefixed message)"를 0개 이상 흘려보낼 수 있다. 네 가지 RPC 유형이란, 이 일반적인 능력을 두고 **"클라이언트 쪽은 몇 개를 보낼 수 있는가"** 와 **"서버 쪽은 몇 개를 보낼 수 있는가"** 라는 두 개의 축에 제약을 거는 것에 지나지 않는다.

```text
                서버가 보내는 메시지 수
                ┌─────────────┬─────────────────────┐
                │   정확히 1개   │      0개 이상(stream)  │
   ┌────────────┼─────────────┼─────────────────────┤
 클 │ 정확히 1개   │   Unary      │   Server streaming   │
 라 │            │   (1 : 1)    │   (1 : N)            │
 이 ├────────────┼─────────────┼─────────────────────┤
 언 │ 0개 이상     │   Client     │   Bidirectional      │
 트 │  (stream)  │   streaming  │   streaming          │
   │            │   (N : 1)    │   (N : M)             │
   └────────────┴─────────────┴─────────────────────┘
```

이 2×2 표가 이 장 전체의 지도다. `stream` 키워드는 요청 메시지 타입 앞에 붙으면 "클라이언트가 여럿 보낼 수 있음"을, 응답 메시지 타입 앞에 붙으면 "서버가 여럿 보낼 수 있음"을 뜻한다. 그것이 전부다. 나머지는 전부 이 한 문장의 따름정리(corollary)다.

왜 이렇게 설계했을까? RPC라는 개념은 원래 "원격 함수 호출"이다. 함수는 인자 하나(또는 묶음)를 받고 값 하나를 돌려준다. 그것이 Unary다. 하지만 현실의 분산 시스템에서는 "결과가 너무 커서 한 번에 못 주는 경우"(대량 조회), "입력이 너무 커서 한 번에 못 보내는 경우"(파일 업로드), "끝이 정해지지 않은 대화"(채팅, 실시간 동기화)가 흔하다. gRPC를 만든 사람들은 이런 경우마다 별도의 프로토콜을 만드는 대신, HTTP/2가 이미 제공하는 **하나의 스트림 위에서 양방향으로 데이터를 흘려보내는 능력**을 그대로 노출하기로 했다. 그래서 네 가지가 "따로" 있는 게 아니라, **하나의 능력에 제약을 걸어 네 가지 모양을 만든** 것이다. HTTP/2 자체의 프레임·멀티플렉싱(multiplexing)·흐름 제어는 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 다뤘으니, 이 장은 "그 위에서 메시지가 어떻게 흐르는가"에 집중한다.

---

## 5.1 네 가지 유형을 proto 시그니처로 선언하기

추상적인 설명보다 먼저 코드를 보자. 채팅·로그·시세를 다루는 가상의 서비스 하나에 네 유형을 모두 담았다.

```proto
syntax = "proto3";

package demo.v1;
option go_package = "example.com/demo/v1;demov1";

// ── 메시지 타입들 ───────────────────────────────
message GetMessageRequest {
  string message_id = 1;
}

message Message {
  string message_id = 1;
  string room_id    = 2;
  string author     = 3;
  string text       = 4;
  int64  sent_at_ms = 5;
}

message SubscribeRequest {
  string room_id = 1;
}

message LogEntry {
  string service   = 1;
  string level     = 2;   // "INFO" | "WARN" | "ERROR"
  string body      = 3;
  int64  ts_ms     = 4;
}

message UploadSummary {
  int64 received_count = 1;
  int64 error_count    = 2;
  int64 first_ts_ms    = 3;
  int64 last_ts_ms     = 4;
}

message ChatMessage {
  string room_id = 1;
  string author  = 2;
  string text    = 3;
}

// ── 서비스 ─────────────────────────────────────
service DemoService {
  // (1) Unary  ─ 1:1
  //   요청 1개 → 응답 1개. 평범한 원격 함수 호출.
  rpc GetMessage(GetMessageRequest) returns (Message);

  // (2) Server streaming ─ 1:N
  //   요청 1개 → 응답 0개 이상. 'stream'이 응답 쪽에 붙는다.
  rpc SubscribeMessages(SubscribeRequest) returns (stream Message);

  // (3) Client streaming ─ N:1
  //   요청 0개 이상 → 응답 1개. 'stream'이 요청 쪽에 붙는다.
  rpc UploadLogs(stream LogEntry) returns (UploadSummary);

  // (4) Bidirectional streaming ─ N:M
  //   요청 0개 이상 ↔ 응답 0개 이상. 양쪽 모두 'stream'.
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

이 한 화면에 네 유형이 다 들어 있다. 시그니처만 비교하면 차이가 선명하다.

| 유형 | 시그니처 패턴 | 클라이언트 메시지 | 서버 메시지 |
|---|---|---|---|
| Unary | `rpc M(Req) returns (Res)` | 정확히 1 | 정확히 1 |
| Server streaming | `rpc M(Req) returns (stream Res)` | 정확히 1 | 0 이상 |
| Client streaming | `rpc M(stream Req) returns (Res)` | 0 이상 | 정확히 1 |
| Bidirectional | `rpc M(stream Req) returns (stream Res)` | 0 이상 | 0 이상 |

여기서 한 가지 강조할 점. **"0개 이상"이지 "1개 이상"이 아니다.** 빈 스트림이 합법이다. 예를 들어 client streaming에서 클라이언트가 한 건도 보내지 않고 곧장 스트림을 닫아버리면, 서버는 메시지를 0개 받은 채로 `UploadSummary{received_count: 0}`을 돌려주는 게 정상이다. server streaming에서 조건에 맞는 결과가 하나도 없으면 서버는 DATA를 한 번도 안 보내고 곧장 OK trailer를 보낼 수 있다. 이 "0개도 정상"이라는 사실은 나중에 빈 결과 처리, 헬스 체크, 타임아웃 설계에서 중요해진다.

`stream` 키워드는 **컴파일 타임에 메서드 스텁의 모양 자체를 바꾼다.** 같은 `protoc`이 같은 메서드를, Unary면 "값을 반환하는 함수"로, server streaming이면 "스트림 객체를 반환하는 함수"로 생성한다. 즉 네 유형의 차이는 런타임 플래그가 아니라 **생성된 API의 타입**으로 굳어진다. 코드 생성과 `protoc` 툴체인의 일반론은 [[06 - 코드 생성과 protoc 툴체인]]을 보라. 이 장에서는 "생성된 그 함수가 내부적으로 무슨 짓을 하는가"에 집중한다.

---

## 5.2 모든 RPC는 스트림 위의 "메시지 시퀀스"다

본격적으로 들어가기 전에, 네 유형을 관통하는 공통 와이어 구조를 손에 쥐어야 한다. 그래야 "Unary가 사실 스트림 1개짜리"라는 말이 비유가 아니라 사실로 다가온다.

### 5.2.1 HTTP/2 위에서의 한 통화(call)의 골격

하나의 gRPC 호출은 HTTP/2 스트림 하나를 새로 열어(클라이언트가 짝수가 아닌 홀수 stream id를 부여) 시작하고, 그 스트림이 닫히면 끝난다. 프레임(frame) 수준에서 보면 거의 항상 이런 순서다.

```text
클라이언트                                      서버
   │                                            │
   │  HEADERS (stream=3)                         │   요청 헤더
   │    :method = POST                           │   (의사 헤더 + gRPC 헤더)
   │    :scheme = https                          │
   │    :path   = /demo.v1.DemoService/GetMessage│
   │    content-type = application/grpc          │
   │    te = trailers                            │
   │ ─────────────────────────────────────────► │
   │                                            │
   │  DATA (stream=3)  [길이앞붙임 메시지 1개]       │   요청 본문
   │     END_STREAM ⬅ "나는 더 안 보낸다"(half-close)│
   │ ─────────────────────────────────────────► │
   │                                            │
   │                       HEADERS (stream=3)   │   응답 헤더
   │                         :status = 200       │   (END_STREAM 아님!)
   │                         content-type = ...  │
   │ ◄───────────────────────────────────────── │
   │                                            │
   │                       DATA (stream=3)       │   응답 본문
   │                         [길이앞붙임 메시지]     │
   │ ◄───────────────────────────────────────── │
   │                                            │
   │                       HEADERS (stream=3)   │   응답 트레일러(trailers)
   │                         grpc-status = 0     │   END_STREAM ⬅ 통화 종료
   │                         grpc-message = ...  │
   │ ◄───────────────────────────────────────── │
```

이 그림에서 끄집어내야 할 사실이 네 가지 있다.

1. **요청 헤더는 단 한 번** 보낸다. `:path`가 곧 호출할 메서드의 풀네임(`/패키지.서비스/메서드`)이다. gRPC에는 별도의 "메서드 이름 필드"가 없다. 메서드는 URL 경로다.
2. **응답에는 두 번의 HEADERS**가 있다. 첫 번째는 응답 헤더(initial metadata), 마지막은 트레일러(trailers, 흔히 trailing metadata). 응답 헤더의 `:status: 200`은 "HTTP 수준에서 요청이 도달했다"만 의미하고, **실제 gRPC 성공/실패는 마지막 트레일러의 `grpc-status`로 전달된다.** 이 분리(HTTP 200인데 gRPC는 실패)가 처음엔 혼란스럽지만 스트리밍을 생각하면 필연이다. 서버가 결과를 절반쯤 흘려보낸 뒤에야 에러가 났다면, 그때는 이미 `:status` 헤더를 보낸 뒤라 되돌릴 수 없다. 그래서 gRPC는 "최종 판정"을 맨 끝 트레일러로 미룬다. 이 에러 모델은 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에서 깊게 다룬다.
3. **END_STREAM 플래그가 "끝"의 신호**다. 클라이언트가 마지막 DATA(또는 HEADERS)에 END_STREAM을 달면 "내 방향은 끝"이라는 half-close다. 서버가 트레일러 HEADERS에 END_STREAM을 달면 "통화 전체 종료"다.
4. **메시지는 DATA 프레임에 실려 흐른다.** 그런데 DATA 프레임 경계와 메시지 경계는 일치하지 않는다. 메시지 하나가 여러 DATA 프레임에 쪼개질 수도 있고, DATA 프레임 하나에 메시지 여러 개가 담길 수도 있다. 그래서 메시지 경계를 알아내려면 별도의 길이 표시가 필요하다. 그게 다음 절이다.

### 5.2.2 길이 앞붙임 메시지(length-prefixed message) — gRPC의 진짜 단위

HTTP/2 DATA 프레임은 그냥 바이트 스트림이다. "여기까지가 메시지 1개"라는 경계를 DATA 프레임이 알려주지 않으므로, gRPC는 애플리케이션 페이로드 안에 자체적인 프레이밍(framing)을 둔다. 모든 메시지는 **정확히 5바이트의 접두부(prefix)** 를 앞에 달고 나간다.

```text
 ┌──────────┬───────────────────────────┬───────────────────────┐
 │ 1 byte   │  4 bytes (big-endian u32)  │   N bytes             │
 │ 압축 플래그 │  메시지 길이 = N             │   직렬화된 메시지 본문    │
 │ 0 또는 1  │                            │   (protobuf 바이트)    │
 └──────────┴───────────────────────────┴───────────────────────┘
   compressed-flag        length                message
```

- **압축 플래그(1바이트)**: `0x00`이면 압축 안 함, `0x01`이면 이 메시지가 (스트림에 협상된 압축기로) 압축됐음. 압축 협상은 `grpc-encoding` 헤더로 한다.
- **길이(4바이트, big-endian)**: 뒤따르는 메시지 본문의 바이트 수. 부호 없는 32비트라 메시지 하나는 이론상 4GiB까지 가능하지만 실제로는 수신 측 `maxReceiveMessageSize`(기본 4MiB)에 걸린다.
- **메시지 본문(N바이트)**: Protocol Buffers로 직렬화된 바이트. 이 인코딩 자체는 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 다룬다.

이 5바이트가 왜 결정적이냐면, **이것이 있어야 수신 측이 "메시지 1개가 다 왔는지"를 DATA 프레임과 무관하게 판단**할 수 있기 때문이다. 수신 측 디코더는 항상 이렇게 동작한다. "먼저 5바이트를 모은다 → 길이 N을 읽는다 → 본문 N바이트가 다 모일 때까지 기다린다 → 한 메시지를 디코드해서 애플리케이션에 전달한다 → 버퍼에 남은 바이트로 다음 5바이트를 다시 모은다." 이 루프가 스트리밍 전체를 떠받친다.

#### 손으로 디코딩해 보기

`Message`가 아니라 더 단순한 메시지로 직접 바이트를 따라가 보자. 아래 proto와 값으로 시작한다.

```proto
message Ping { string text = 2; }   // text = "hi"
```

Protocol Buffers 인코딩(자세한 규칙은 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]):

```text
필드 2, wire type 2(LEN):  tag = (2 << 3) | 2 = 0x12
문자열 길이 2:                            0x02
"hi" =                                  0x68 0x69
→ protobuf 본문 = 12 02 68 69  (총 4바이트)
```

이제 gRPC 프레이밍을 입힌다.

```text
압축 플래그(1B) :  00                  ← 압축 안 함
길이(4B)       :  00 00 00 04         ← 본문이 4바이트라는 뜻
본문(4B)       :  12 02 68 69
──────────────────────────────────────
와이어 바이트   :  00 00 00 00 04 12 02 68 69   (1 + 4 + 4 = 총 9바이트)
```

`bash`에서 grpcurl 등으로 실제 바이트를 들여다보면 정확히 이 구조가 보인다. 두 메시지를 연달아 보내면 이 9바이트 블록이 그냥 두 번 이어 붙는다.

```text
메시지1 ("hi") : 00 00 00 00 04 12 02 68 69          ← 5바이트 접두부 + 본문 4B = 9B
메시지2 ("bye"): 00 00 00 00 05 12 03 62 79 65       ← 5바이트 접두부 + 본문 5B = 10B
두 메시지 연속  : 00 00 00 00 04 12 02 68 69┃00 00 00 00 05 12 03 62 79 65
                 (┃ 표시가 메시지 경계 — 와이어에는 이런 구분자가 실제로 없다.
                  오직 길이 접두부가 "다음 5+N 바이트가 한 메시지"임을 말해줄 뿐)
```

핵심: **이 바이트 열에는 "메시지 경계"를 알려주는 구분자가 없다.** 오로지 길이 접두부가 "다음 5+N 바이트가 한 메시지"라고 말해줄 뿐이다. 그래서 DATA 프레임이 어디서 쪼개지든 상관없다. 수신 측은 길이만 보고 정확히 메시지를 잘라낸다. 이 단순한 규칙 하나로 Unary/스트리밍이 **모두 동일한 디코더**를 쓴다. 스트리밍이라고 별도의 마법이 있는 게 아니라, "길이 앞붙임 메시지를 1개만 읽고 끝내느냐, 계속 읽느냐"의 차이일 뿐이다.

이제 이 공통 구조를 손에 쥐고 네 유형을 하나씩 해부하자.

---

## 5.3 Unary — "스트림 위의 메시지 1개"

Unary는 가장 단순해 보이지만 "사실은 스트림 1개짜리"라는 진실을 가장 먼저 체화하기 좋은 유형이다.

### 5.3.1 와이어에서 실제로 벌어지는 일

`GetMessage(GetMessageRequest) returns (Message)`를 호출하면:

```text
클라이언트 → 서버
  1) HEADERS (stream=3, END_HEADERS)
        :method POST  :path /demo.v1.DemoService/GetMessage
        content-type application/grpc  te trailers  ...
  2) DATA   (stream=3, END_STREAM)
        00 00 00 00 0C  [GetMessageRequest 12바이트...]
        └ 접두부 5B(플래그 00 + 길이 00 00 00 0C) + 본문 12B
        └ END_STREAM = "요청 끝. 나는 더 안 보낸다" (half-close)

서버 → 클라이언트
  3) HEADERS (stream=3)   :status 200, content-type application/grpc
  4) DATA   (stream=3)    00 00 00 00 2A  [Message 42바이트...]  (접두부 5B + 본문 42B)
  5) HEADERS (stream=3, END_STREAM)   grpc-status 0   ← 통화 종료
```

여기서 두 가지를 보라.

- **요청 DATA에 곧장 END_STREAM이 달린다.** Unary는 클라이언트가 보낼 게 정확히 1개이므로, 그 1개를 보내자마자 "내 방향 끝"을 선언한다. 이게 half-close다. 메시지 송신과 half-close가 사실상 한 동작으로 합쳐진다.
- **서버는 DATA 1개 + 트레일러로 끝낸다.** `grpc-status: 0`이 OK다.

그러니까 Unary의 와이어 모습은 "5.2의 골격 그대로, 양쪽 메시지가 1개씩"이다. **Unary 전용 프로토콜 같은 건 없다.** 그래서 "Unary도 내부적으로는 스트림에 메시지 1개"라는 말이 정확히 맞다.

### 5.3.2 Trailers-Only 응답 — 에러 케이스의 지름길

한 가지 특수 케이스. 만약 서버가 메시지를 한 개도 보내지 않고 곧장 실패한다면(예: 인증 실패, 잘못된 인자), HTTP/2 차원에서 응답 헤더와 트레일러를 굳이 두 번 보낼 이유가 없다. 그래서 gRPC는 **Trailers-Only**라는 최적화를 허용한다. 서버가 처음이자 마지막인 단 하나의 HEADERS 프레임에 `:status: 200`, `grpc-status: 7`(PERMISSION_DENIED 등)을 함께 담고 END_STREAM을 단다. DATA 프레임은 없다. 클라이언트 구현은 "DATA 없이 트레일러만 온" 경우를 정상 처리해야 한다. 이 케이스는 server streaming에서 "결과 0개 + 에러"일 때도 그대로 쓰인다. 상세는 [[09 - 에러 모델 - 상태 코드와 Rich Error]].

### 5.3.3 언어별 API — "그냥 함수 호출"

Unary는 생성 코드도 가장 단순하다. 동기(blocking) 호출이면 정말 보통 함수처럼 보인다.

```go
// Go — 동기 Unary
func getOne(ctx context.Context, cli demov1.DemoServiceClient) error {
    // 요청 1개를 넣고, 응답 1개를 받는다. 끝.
    resp, err := cli.GetMessage(ctx, &demov1.GetMessageRequest{
        MessageId: "m-123",
    })
    if err != nil {
        // err에는 gRPC status가 들어 있다 (status.FromError로 코드 추출)
        return err
    }
    fmt.Println(resp.GetText())
    return nil
}
```

```java
// Java — blocking stub
DemoServiceGrpc.DemoServiceBlockingStub stub =
        DemoServiceGrpc.newBlockingStub(channel);

Message resp = stub.getMessage(
        GetMessageRequest.newBuilder().setMessageId("m-123").build());
System.out.println(resp.getText());   // 예외가 안 났으면 곧 응답
```

```python
# Python — 동기 stub
resp = stub.GetMessage(demo_pb2.GetMessageRequest(message_id="m-123"))
print(resp.text)
```

세 언어 모두 "인자 넣고 값 받는" 평범한 호출이다. 비동기가 필요하면 Java는 `FutureStub`(`ListenableFuture<Message>`)이나 `newStub`(콜백 기반 `StreamObserver`)을, Go는 별도 goroutine으로 감싸는 식으로 만든다. 하지만 와이어에서 벌어지는 일은 위 5.3.1과 동일하다. **동기/비동기는 클라이언트 코드의 모양일 뿐, 프로토콜은 같다.**

> 메모: Unary 호출에도 deadline/timeout, 메타데이터, 취소가 전부 적용된다. 이건 모든 유형 공통이라 별도 장으로 뺐다 — [[10 - Deadline 취소 타임아웃]], [[08 - 메타데이터와 인터셉터]].

---

## 5.4 Server streaming — 1:N, "한 번 묻고 여러 번 받는다"

`SubscribeMessages(SubscribeRequest) returns (stream Message)`. 클라이언트는 요청을 정확히 1개 보내고, 서버는 응답을 0개 이상 흘려보낸다. 대량 조회나 실시간 피드의 기본형이다.

### 5.4.1 와이어 흐름

```text
클라이언트 → 서버
  HEADERS (stream=5)  :path /demo.v1.DemoService/SubscribeMessages ...
  DATA    (stream=5, END_STREAM)  [SubscribeRequest 1개]
       └ 요청 1개 보내고 곧장 half-close. (클라는 더 보낼 게 없다)

서버 → 클라이언트
  HEADERS (stream=5)  :status 200 ...
  DATA    (stream=5)  [Message #1]
  DATA    (stream=5)  [Message #2]
  DATA    (stream=5)  [Message #3]
        ...           (서버가 원하는 만큼, 시간 간격을 두고)
  HEADERS (stream=5, END_STREAM)  grpc-status 0   ← 스트림 종료
```

요청 쪽은 Unary와 똑같다. 차이는 응답 쪽이 **DATA 여러 개**라는 것뿐이다. 각 `Message`는 5.2.2의 길이 앞붙임 규칙대로 자기 5바이트 접두부를 달고 나간다. 서버가 다 보내면 트레일러로 종료를 알린다.

여기서 직관 하나. 서버 스트리밍의 DATA들은 **시간적으로 흩어져** 나갈 수 있다. 시세 피드라면 1초마다 한 개씩 나갈 것이다. 그동안 스트림(stream id=5)은 계속 "열린" 상태로 살아 있고, HTTP/2 연결의 다른 스트림들과 멀티플렉싱되어 같은 TCP 위를 공유한다. 이게 "긴 스트림 하나가 연결을 독점하지 않는" HTTP/2의 강점이다([[04 - HTTP2 깊이 보기 - 전송 계층]]).

### 5.4.2 언어별 API — 수신 루프(recv loop)

클라이언트 쪽은 "스트림을 받아서 다 읽을 때까지 도는 루프"가 된다.

```go
// Go — server streaming 클라이언트
stream, err := cli.SubscribeMessages(ctx, &demov1.SubscribeRequest{
    RoomId: "room-42",
})
if err != nil {
    return err // 스트림 '시작' 자체가 실패한 경우
}
for {
    msg, err := stream.Recv()
    if err == io.EOF {
        break // 서버가 정상적으로 스트림을 닫음 (grpc-status 0)
    }
    if err != nil {
        return err // 도중에 에러 상태(trailer)가 도착
    }
    fmt.Printf("[%s] %s\n", msg.GetAuthor(), msg.GetText())
}
```

`io.EOF`가 핵심이다. **EOF는 "에러"가 아니라 "서버가 정상 종료했다"는 신호**다. Go의 관용구에서 `Recv()`는 다음 메시지가 있으면 그것을, 스트림이 정상으로 닫혔으면 `io.EOF`를, 비정상으로 끝났으면 진짜 에러를 돌려준다. 이 세 갈래를 구분하는 게 server streaming 클라이언트의 전부다.

서버 쪽은 "보내고 싶은 만큼 Send를 호출하고, 함수를 반환하면 끝"이다.

```go
// Go — server streaming 서버 구현
func (s *server) SubscribeMessages(
    req *demov1.SubscribeRequest,
    stream demov1.DemoService_SubscribeMessagesServer,
) error {
    for _, m := range s.recentMessages(req.GetRoomId()) {
        if err := stream.Send(m); err != nil {
            return err // 클라가 끊었거나 네트워크 문제
        }
    }
    // 여기서 nil을 반환하면 grpc-status 0(OK) 트레일러가 나간다.
    // 에러를 반환하면 그 에러가 status로 변환되어 트레일러로 나간다.
    return nil
}
```

Java는 같은 일을 `StreamObserver`로 한다.

```java
// Java — server streaming 서버 구현
@Override
public void subscribeMessages(SubscribeRequest req,
                              StreamObserver<Message> obs) {
    for (Message m : recentMessages(req.getRoomId())) {
        obs.onNext(m);      // DATA 한 개
    }
    obs.onCompleted();      // grpc-status 0 트레일러
    // 에러로 끝내려면: obs.onError(Status.NOT_FOUND.asRuntimeException());
}
```

```python
# Python — server streaming 서버 (제너레이터로 yield)
def SubscribeMessages(self, request, context):
    for m in self.recent_messages(request.room_id):
        yield m        # yield 하나가 메시지 하나
    # 함수가 끝나면 자동으로 OK 트레일러
```

Python의 제너레이터(`yield`)는 server streaming을 가장 우아하게 표현한다. `yield`한 값 하나가 곧 DATA 메시지 하나이고, 제너레이터가 소진되면 스트림이 닫힌다.

세 언어가 표현은 달라도 **추상은 동일**하다. 서버: "여러 번 Send/onNext/yield → 끝." 클라이언트: "EOF가 올 때까지 Recv 루프." 이 대칭이 보이면 server streaming은 끝난 것이다.

---

## 5.5 Client streaming — N:1, "여러 번 보내고 한 번 받는다"

`UploadLogs(stream LogEntry) returns (UploadSummary)`. 거울에 비친 server streaming이다. 클라이언트가 0개 이상을 흘려보내고, 서버는 다 받은 뒤 응답 1개로 답한다. 업로드·집계·대량 적재(batch ingest)의 기본형이다.

### 5.5.1 와이어 흐름과 half-close의 진짜 의미

```text
클라이언트 → 서버
  HEADERS (stream=7)  :path /demo.v1.DemoService/UploadLogs ...
  DATA    (stream=7)  [LogEntry #1]
  DATA    (stream=7)  [LogEntry #2]
  DATA    (stream=7)  [LogEntry #3]
        ...
  DATA    (stream=7, END_STREAM)  [LogEntry #N]
       └ 마지막 DATA에 END_STREAM = "업로드 끝. 이제 집계해서 줘" (half-close)

서버 → 클라이언트
  HEADERS (stream=7)  :status 200 ...
  DATA    (stream=7)  [UploadSummary 1개]
  HEADERS (stream=7, END_STREAM)  grpc-status 0
```

여기서 **half-close가 client streaming의 심장**이다. 클라이언트가 보낼 게 더 없다는 사실(`END_STREAM`)을 명시적으로 알려야, 서버가 비로소 "입력이 끝났구나" 하고 집계를 마무리해 응답을 만들 수 있다. half-close가 없으면 서버는 영원히 "다음 로그가 더 올지도 몰라" 하며 기다린다.

흔한 오해 하나를 짚자. **half-close는 연결을 끊는 게 아니다.** "내 *송신* 방향만 닫는다"는 뜻이다. half-close 이후에도 서버→클라이언트 방향은 멀쩡히 열려 있어서 서버가 그 위로 `UploadSummary`와 트레일러를 보낼 수 있다. "half"는 양방향 중 한쪽 절반만 닫는다는 의미다. 양쪽이 다 닫혀야(서버가 트레일러에 END_STREAM) 스트림이 완전히 끝난다.

### 5.5.2 언어별 API — 송신 루프 + CloseAndRecv

클라이언트는 "여러 번 Send → CloseAndRecv로 닫고 동시에 결과 받기"다.

```go
// Go — client streaming 클라이언트
stream, err := cli.UploadLogs(ctx)
if err != nil {
    return err
}
for _, e := range logEntries {
    if err := stream.Send(e); err != nil {
        // Send 도중 에러 → 보통 CloseAndRecv로 진짜 status를 받아 본다
        break
    }
}
// CloseAndRecv: half-close(END_STREAM) + 서버 응답 1개 수신을 한 번에
summary, err := stream.CloseAndRecv()
if err != nil {
    return err
}
fmt.Printf("received=%d errors=%d\n",
    summary.GetReceivedCount(), summary.GetErrorCount())
```

`CloseAndRecv()`라는 이름이 동작을 그대로 말해준다. **Close**(나는 더 안 보냄 = half-close) **And Recv**(서버가 주는 단 하나의 응답을 받음). 이 두 동작이 하나로 묶인 이유는, client streaming에서 응답은 정확히 1개라서 "닫는 순간이 곧 받는 순간"이기 때문이다.

```java
// Java — client streaming 클라이언트 (async stub)
DemoServiceGrpc.DemoServiceStub stub = DemoServiceGrpc.newStub(channel);

final CountDownLatch done = new CountDownLatch(1);
StreamObserver<UploadSummary> respObs = new StreamObserver<>() {
    @Override public void onNext(UploadSummary s) {
        System.out.println("received=" + s.getReceivedCount());
    }
    @Override public void onError(Throwable t) { done.countDown(); }
    @Override public void onCompleted() { done.countDown(); }
};

StreamObserver<LogEntry> reqObs = stub.uploadLogs(respObs);
for (LogEntry e : logEntries) {
    reqObs.onNext(e);          // DATA 한 개씩
}
reqObs.onCompleted();          // half-close
done.await();                  // 응답이 올 때까지
```

Java는 본질적으로 비동기라 `StreamObserver` 두 개(요청용, 응답용)가 짝을 이룬다. 요청 옵저버의 `onCompleted()`가 곧 half-close다.

```python
# Python — client streaming 클라이언트
def gen():
    for e in log_entries:
        yield e            # 보낼 메시지를 제너레이터로 흘린다
summary = stub.UploadLogs(gen())   # 제너레이터 소진 = half-close
print(summary.received_count)
```

서버 구현은 "수신 루프 돌려 다 받고, 마지막에 응답 1개 돌려주기"다.

```go
// Go — client streaming 서버 구현
func (s *server) UploadLogs(
    stream demov1.DemoService_UploadLogsServer,
) error {
    var n, errs, first, last int64
    for {
        e, err := stream.Recv()
        if err == io.EOF {
            // 클라가 half-close 함 → 이제 집계 결과를 돌려준다
            return stream.SendAndClose(&demov1.UploadSummary{
                ReceivedCount: n, ErrorCount: errs,
                FirstTsMs: first, LastTsMs: last,
            })
        }
        if err != nil {
            return err
        }
        n++
        if e.GetLevel() == "ERROR" { errs++ }
        if first == 0 { first = e.GetTsMs() }
        last = e.GetTsMs()
    }
}
```

서버에서도 대칭이 보인다. 클라이언트의 `CloseAndRecv`에 짝을 이루는 게 서버의 `SendAndClose`다. 클라이언트가 보낸 EOF를 받은 시점이 곧 "응답을 만들어 보내고 닫을" 시점이다.

> 실전 주의: client streaming에서 서버가 *전부 받기 전에* 일찍 응답하고 끝내버릴 수도 있다(예: 잘못된 첫 메시지를 보고 곧장 `INVALID_ARGUMENT`로 종료). 그러면 클라이언트의 `Send`가 도중에 에러로 떨어진다. 이때 클라이언트는 `Send` 에러 자체보다 `CloseAndRecv`(또는 응답 옵저버의 `onError`)가 주는 **실제 status를 신뢰**해야 한다. 부분 처리·멱등성 이야기는 5.10에서 마저 다룬다.

---

## 5.6 Bidirectional streaming — N:M, "동시에 말하고 동시에 듣는다"

`Chat(stream ChatMessage) returns (stream ChatMessage)`. 양쪽 모두 0개 이상을 흘려보낸다. 가장 강력하고 가장 잘못 쓰기 쉬운 유형이다.

### 5.6.1 두 방향이 진짜로 독립적이다

bidi의 결정적 성질은 **송신 방향과 수신 방향이 완전히 독립**이라는 것이다. 클라이언트가 메시지를 다 보낸 뒤에야 서버가 응답하기 시작할 수도 있고(그러면 사실상 client→server 다음 server→client), 클라이언트가 보내는 도중에 서버가 끼어들어 응답할 수도 있다(진짜 양방향 대화). 어떤 인터리빙(interleaving)을 쓸지는 **순전히 애플리케이션이 정한다.** gRPC는 단지 "두 방향이 같은 스트림 위에서 독립적으로 흐를 수 있다"는 능력만 제공한다.

```text
클라이언트 → 서버                         서버 → 클라이언트
  HEADERS (stream=9)
  DATA [ChatMessage c1] ───────►
                          ◄─────── HEADERS (:status 200)
                          ◄─────── DATA [ChatMessage s1]
  DATA [ChatMessage c2] ───────►
                          ◄─────── DATA [ChatMessage s2]
  DATA [ChatMessage c3] ───────►
                          ◄─────── DATA [ChatMessage s3]
  DATA(END_STREAM)      ───────►   (클라 half-close: 더 안 보냄)
                          ◄─────── DATA [ChatMessage s4]  ← 여전히 보낼 수 있다!
                          ◄─────── HEADERS(END_STREAM) grpc-status 0
```

이 그림에서 가장 흥미로운 줄은 클라이언트가 half-close(`DATA END_STREAM`) 한 *뒤에도* 서버가 `s4`를 보내는 부분이다. half-close는 **한쪽 방향만** 닫으므로, 클라이언트가 입력을 끝냈어도 서버는 아직 출력을 더 흘려보낼 수 있다. 이 비대칭이 bidi 설계의 묘미이자 함정이다.

### 5.6.2 언어별 API — 동시 send/recv

bidi에서는 보내기와 받기를 **동시에** 다뤄야 하므로, 거의 항상 별도의 실행 흐름(goroutine, thread, async task)이 등장한다.

```go
// Go — bidi 클라이언트. 보내기와 받기를 동시에.
stream, err := cli.Chat(ctx)
if err != nil {
    return err
}

// (A) 수신 전용 goroutine
recvDone := make(chan error, 1)
go func() {
    for {
        in, err := stream.Recv()
        if err == io.EOF {
            recvDone <- nil // 서버가 자기 방향을 닫음
            return
        }
        if err != nil {
            recvDone <- err
            return
        }
        fmt.Printf("[%s] %s\n", in.GetAuthor(), in.GetText())
    }
}()

// (B) 메인 goroutine은 보내기 담당
for _, text := range outgoing {
    if err := stream.Send(&demov1.ChatMessage{
        RoomId: "room-42", Author: "me", Text: text,
    }); err != nil {
        break
    }
}
stream.CloseSend()      // 내 송신 방향 half-close

if err := <-recvDone; err != nil { // 수신 goroutine이 끝나길 기다림
    return err
}
```

여기서 `CloseSend()`가 client streaming의 half-close와 같은 역할이다. 단지 bidi에서는 그 뒤로도 수신을 계속하므로, **CloseSend 이후에도 Recv 루프는 EOF를 받을 때까지 살아 있다.** 송신 goroutine과 수신 goroutine이 따로 도는 이유가 바로 이것이다. 한 goroutine으로 `Send`와 `Recv`를 번갈아 호출하면, 상대가 보내주기를 기다리는 동안 내가 보내야 할 것을 못 보내 교착(deadlock)에 빠지기 쉽다(5.9에서 자세히).

```java
// Java — bidi 클라이언트
StreamObserver<ChatMessage> reqObs = stub.chat(new StreamObserver<>() {
    @Override public void onNext(ChatMessage in) {
        System.out.println(in.getAuthor() + ": " + in.getText());
    }
    @Override public void onError(Throwable t) { /* ... */ }
    @Override public void onCompleted() { /* 서버 종료 */ }
});

for (String text : outgoing) {
    reqObs.onNext(ChatMessage.newBuilder()
        .setRoomId("room-42").setAuthor("me").setText(text).build());
}
reqObs.onCompleted();   // 송신 half-close
```

Java에서는 콜백(응답 `StreamObserver`)이 수신을 처리하므로 "두 흐름"이 명시적 스레드가 아니라 콜백 구조로 숨어 있다. 하지만 개념은 같다. 보내는 코드와 받는 콜백이 독립적으로 진행된다.

```python
# Python — bidi 클라이언트
def outgoing_gen():
    for text in messages:
        yield demo_pb2.ChatMessage(room_id="room-42", author="me", text=text)

# 보내기는 제너레이터, 받기는 응답 이터레이터
for reply in stub.Chat(outgoing_gen()):
    print(reply.author, reply.text)
```

서버 구현은 "받으면서 동시에 보내는" 형태다.

```go
// Go — bidi 서버: 받은 메시지를 같은 방에 브로드캐스트하고, 받은 것도 에코
func (s *server) Chat(stream demov1.DemoService_ChatServer) error {
    for {
        in, err := stream.Recv()
        if err == io.EOF {
            return nil // 클라가 다 보냄 → 서버도 정상 종료
        }
        if err != nil {
            return err
        }
        // 받은 즉시 응답을 보낼 수 있다 (인터리빙은 자유)
        out := &demov1.ChatMessage{
            RoomId: in.GetRoomId(),
            Author: "server",
            Text:   "echo: " + in.GetText(),
        }
        if err := stream.Send(out); err != nil {
            return err
        }
    }
}
```

이 서버는 "받자마자 응답"하는 단순 에코지만 실전 채팅이라면 받은 메시지를 다른 클라이언트들의 스트림으로 팬아웃(fan-out)하고, 이 스트림으로는 *다른* 사용자가 보낸 메시지를 흘려보낼 것이다. 즉 Recv와 Send가 서로 다른 소스/싱크에 묶인다.

---

## 5.7 메시지 순서 — 한 스트림 안에서는 FIFO, 스트림 사이에는 보장 없음

이 절은 짧지만 분산 시스템 버그의 단골 원인을 정확히 짚는다.

### 5.7.1 한 스트림 내부: FIFO 보장

**하나의 RPC 호출(=하나의 HTTP/2 스트림) 안에서, 한 방향의 메시지들은 보낸 순서 그대로 도착한다.** 서버가 `m1, m2, m3` 순으로 Send 했다면 클라이언트는 정확히 `m1, m2, m3` 순으로 Recv 한다. 추월도, 재정렬도 없다.

왜 보장되는가? 두 층의 순서가 합쳐진 결과다. (1) TCP가 바이트 스트림의 순서를 보장한다. (2) HTTP/2의 한 스트림에 속한 DATA 프레임들은 그 TCP 바이트 스트림 위에서 순서대로 배치된다. (3) gRPC의 길이 앞붙임 프레이밍은 그 바이트 순서대로 메시지를 잘라낸다. 어느 층도 같은 스트림 내부의 순서를 흐트러뜨리지 않으므로, 결과적으로 메시지 단위 FIFO가 성립한다.

이 보장은 **방향별**이다. 클라이언트→서버의 순서와 서버→클라이언트의 순서는 각각 보장되지만 두 방향 사이의 상대적 타이밍은 보장되지 않는다(bidi에서 "클라 c2와 서버 s1 중 누가 먼저냐"는 정해지지 않음).

### 5.7.2 스트림 사이: 순서 보장 없음

서로 다른 RPC 호출(서로 다른 스트림 id)들 사이에는 **아무 순서 보장이 없다.** 같은 채널 위에서 멀티플렉싱되더라도, 호출 A를 먼저 시작했다고 A의 응답이 B보다 먼저 온다는 보장은 전혀 없다. HTTP/2 스트림들은 독립적으로 진행되고 서버 처리 시간도 제각각이며 흐름 제어 윈도우(flow-control window)도 스트림마다 따로 논다.

```text
잘못된 가정:  "주문 생성 RPC를 먼저, 결제 RPC를 나중에 호출했으니
              서버에서도 주문이 먼저 처리되겠지"  ← 틀림

올바른 모델:  순서가 중요하면
              (a) 같은 스트림 안에 순서대로 흘려보내거나
              (b) 명시적 시퀀스 번호/버전을 메시지에 담거나
              (c) 한 RPC가 끝난 뒤 다음을 호출(직렬화)
```

실무적 함의: "순서가 의미를 가지는 일련의 작업"은 **하나의 스트리밍 RPC** 안에 담으면 FIFO를 공짜로 얻는다. 반대로 여러 Unary 호출을 병렬로 던지면서 순서를 기대하면 깨진다. 이 점은 bidi로 상태 동기화를 설계할 때 특히 중요하다 — "변경 이벤트를 순서대로" 보내려면 한 스트림에 실어야 한다.

---

## 5.8 흐름 제어와 백프레셔 — 스트리밍에서만 진짜로 체감된다

Unary에서는 메시지가 1개라 흐름 제어를 거의 의식하지 못한다. 하지만 스트리밍에서는 "생산 속도 > 소비 속도"가 곧장 문제가 된다. 빠른 생산자가 느린 소비자에게 메시지를 무한정 밀어넣으면 메모리가 터진다. 이를 막는 게 **백프레셔(backpressure, 역압)** 이고, gRPC는 이를 HTTP/2 흐름 제어 위에 얹어 구현한다.

직관만 미리 잡자. HTTP/2는 스트림마다, 그리고 연결 전체에도 **수신 윈도우(receive window)** 를 둔다. 수신자가 아직 처리하지 못한 데이터로 윈도우가 0이 되면, 송신자는 `WINDOW_UPDATE` 프레임으로 윈도우가 다시 열릴 때까지 **DATA를 더 보낼 수 없다.** gRPC 라이브러리는 이 신호를 애플리케이션 API로 끌어올린다. 예컨대 Go에서 `stream.Send()`는 흐름 제어 윈도우가 막혀 있으면 그 자리에서 블록(또는 비동기 구현에서 "준비 안 됨" 신호)된다. 즉 **느린 소비자가 자동으로 빠른 생산자를 늦춘다.**

```text
빠른 서버(생산)                    느린 클라(소비)
  Send(m1) ─DATA─►  [윈도우 차감]     수신 버퍼에 쌓임, 아직 Recv 안 함
  Send(m2) ─DATA─►  [윈도우 차감]
  Send(m3) ─DATA─►  [윈도우=0]        ← 더 못 보냄!
  Send(m4) ......블록(대기)
                       ◄─WINDOW_UPDATE─ 클라가 Recv로 소비 → 윈도우 회복
  Send(m4) ─DATA─►  [재개]
```

핵심 직관: **streaming은 "큐를 무한정 채우는 채널"이 아니다.** 흐름 제어가 켜진 유한한 파이프다. 그래서 server streaming에서 "1억 건을 stream.Send로 쏟아붓는" 코드는 클라가 천천히 읽는 한 자연스럽게 속도가 맞춰진다 — 메모리 폭발 없이. 반대로 흐름 제어를 의식하지 못하고 "보내는 쪽에서 다 보낼 때까지 안 받는" 구조를 짜면, 흐름 제어가 막히면서 데드락처럼 보이는 정지가 생긴다(5.9). 흐름 제어의 윈도우 계산, BDP 기반 자동 튜닝, `WINDOW_UPDATE`의 정확한 동작은 [[15 - 성능과 Flow Control 백프레셔]]에서 깊게 판다. 여기서는 "스트리밍을 쓰면 백프레셔가 자동으로 작동한다"는 직관만 챙기면 충분하다.

---

## 5.9 bidi의 동시성 모델과 데드락 회피

bidi가 강력한 만큼 위험한 이유는, **양쪽이 동시에 send/recv를 다뤄야 하는데 그 둘이 흐름 제어로 서로 묶일 수 있기** 때문이다. 가장 흔한 사고가 "교착(deadlock)"이다.

### 5.9.1 교착이 생기는 전형적 패턴

다음 시나리오를 보자. 클라이언트가 "메시지를 전부 보낸 다음에 응답을 받기 시작"하도록 짰다고 하자(단일 스레드, 순차).

```text
클라이언트 로직:                      서버 로직:
  for m in 1..1_000_000:               for in 1..:
      stream.Send(m)   ← 여기서 막힘       msg = stream.Recv()
  // 다 보낸 뒤에야:                          stream.Send(reply) ← 여기서 막힘
  for { stream.Recv() }                  // 클라가 Recv를 안 해서 윈도우 0
```

무슨 일이 벌어지나:

1. 클라이언트가 계속 `Send`. 서버는 받자마자 `reply`를 `Send`하려 한다.
2. 하지만 클라이언트는 "다 보낼 때까지" `Recv`를 안 한다. 그래서 **서버→클라 방향 흐름 제어 윈도우가 0**이 된다.
3. 윈도우가 0이라 서버의 `Send(reply)`가 블록된다. 서버는 블록된 채라 다음 `Recv`도 못 한다.
4. 서버가 `Recv`를 멈췄으니 이번엔 **클라→서버 방향 윈도우**도 0이 된다. 그래서 클라이언트의 `Send`도 블록된다.
5. 양쪽 다 상대가 읽어주기를 기다리며 영원히 멈춘다. **교착.**

이것은 버그지만 "예외도 안 나고 그냥 멈추는" 가장 진단하기 까다로운 종류다. CPU는 0%, 로그는 조용한데 진행이 안 된다.

### 5.9.2 회피 원칙 — send와 recv를 분리하라

규칙은 단순하다. **bidi에서는 보내는 흐름과 받는 흐름을 독립적으로 진행시켜라.** 한 흐름이 막혀도 다른 흐름이 계속 소비/생산하게 하라.

- **Go**: 5.6.2처럼 `Recv` 루프를 별도 goroutine에 두고, 메인은 `Send`만 한다. 그러면 서버가 보내는 응답을 그 goroutine이 꾸준히 소비해 윈도우가 닫히지 않는다.
- **Java**: 응답 `StreamObserver`(콜백)가 별도 실행기 스레드에서 호출되므로 구조상 분리돼 있다. 단, 콜백 안에서 다시 블로킹 `Send`를 호출하면 같은 함정에 빠질 수 있으니 주의.
- **Python**: 동기 API에서 send 제너레이터와 응답 이터레이터를 같은 스레드에서 직렬로 다루면 위험. 보내기와 받기를 각각 스레드/태스크로 분리하거나, asyncio API(`grpc.aio`)로 동시에 `await send`/`await recv` 한다.

추가 원칙들:

1. **항상 상대 메시지를 꾸준히 소비하라.** 받은 걸 당장 안 써도 일단 Recv해서 윈도우를 열어줘야 상대가 막히지 않는다. "나는 보내기만 하면 돼" 하고 Recv를 게을리하면 윈도우가 닫혀 양쪽이 멈춘다.
2. **half-close 순서를 약속하라.** 보통 "클라가 보낼 걸 다 보내고 CloseSend → 서버가 남은 응답을 다 보내고 종료" 같은 프로토콜을 메시지 의미로 정해둔다. 누가 먼저 끝내는지 모호하면 둘 다 상대를 기다리다 멈춘다.
3. **무한 bidi에는 반드시 종료 조건/취소를 둬라.** deadline이나 context 취소가 없으면 한쪽이 죽었을 때 다른 쪽이 영원히 매달린다([[10 - Deadline 취소 타임아웃]]).

> 직관: bidi는 "전화 통화"다. 두 사람이 동시에 말하고 동시에 들을 수 있다. 하지만 둘 다 "상대가 말 끝낼 때까지 나는 안 듣겠다"고 고집하면 통화가 멈춘다. 분리(듣기 전담 + 말하기 전담)가 답이다.

---

## 5.10 스트리밍 도중의 에러·취소 — 부분 처리와 멱등성

스트리밍의 가장 까다로운 진실: **에러는 스트림이 절반쯤 진행된 뒤에도 발생할 수 있다.** Unary에서는 "성공 아니면 실패"가 깔끔하지만 스트리밍에서는 "100건 중 60건을 처리한 뒤 실패"가 가능하다. 이게 설계에 깊은 함의를 남긴다.

### 5.10.1 최종 상태는 언제나 trailer로 도달한다

5.2.1에서 봤듯, gRPC의 최종 판정(`grpc-status`)은 **맨 끝 트레일러**로 전달된다. 스트리밍에서 이 설계가 빛난다. 서버가 `m1..m60`을 DATA로 흘려보낸 뒤 DB 장애로 실패하면, 이미 `:status 200`과 60개의 DATA는 나간 상태다. 서버는 그 뒤에 `grpc-status: 14`(UNAVAILABLE) 트레일러를 보내 "여기까지 60개는 진짜지만 통화는 실패로 끝난다"고 알린다. 클라이언트의 Recv 루프는 60번 정상 메시지를 받은 뒤 61번째 Recv에서 EOF가 아니라 **에러**를 받는다.

```go
for {
    msg, err := stream.Recv()
    if err == io.EOF { break }       // 정상 종료 (grpc-status 0)
    if err != nil {
        // 여기 도달 = 도중 실패. 이미 받은 msg들은 '진짜'다.
        st, _ := status.FromError(err)
        log.Printf("stream failed after partial data: %v", st.Code())
        return err
    }
    process(msg)                     // 받은 건 실제로 처리했을 수 있다
}
```

그래서 클라이언트는 **"이미 받아서 처리한 부분"과 "통화는 실패했다"를 동시에 안고** 다음을 결정해야 한다. 이게 부분 처리(partial processing) 문제다.

### 5.10.2 취소(cancellation)도 양방향으로 전파된다

클라이언트가 도중에 취소하면(context cancel, deadline 초과, 또는 명시적 `cancel()`), HTTP/2 `RST_STREAM` 프레임이 나가 그 스트림을 즉시 끊는다. 서버 쪽 핸들러는 다음 `Send`/`Recv`에서 취소된 에러를 받거나, context가 Done 되어 작업을 중단할 수 있어야 한다. 반대로 서버가 먼저 끝내면 클라이언트의 진행 중인 Send가 깨진다. 핵심은 **취소가 한쪽에서 일어나면 다른 쪽도 그 사실을 (다음 I/O 시점에) 알게 된다**는 것. deadline·취소의 전파 메커니즘은 [[10 - Deadline 취소 타임아웃]]에서 정밀하게 다룬다.

### 5.10.3 그래서 멱등성(idempotency)을 설계에 넣어야 한다

부분 처리 + 취소 가능성이 합쳐지면 결론은 하나다. **스트리밍 작업은 "재시도해도 안전한가"를 처음부터 따져야 한다.**

- **client streaming 업로드**: 클라가 80건을 보낸 뒤 연결이 끊겼다. 재시도하면 그 80건이 또 들어갈 수 있다. 해법: 각 항목에 고유 ID(예: `LogEntry`에 `dedup_key`)를 넣어 서버가 중복을 무시하게 하거나, 서버가 "여기까지 받았다"는 커밋 지점을 응답으로 알려 클라가 그 뒤부터 재개하게 한다.
- **server streaming 조회**: 60건 받고 끊겼다. 재시도하면 처음부터 다시 받는다. 해법: 커서/오프셋(예: `SubscribeRequest`에 `resume_from`)을 둬 "마지막으로 받은 지점 이후"만 다시 받는다.
- **bidi 동기화**: 양쪽 모두 부분 진행이라 더 복잡하다. 해법: 시퀀스 번호 + ACK 프로토콜을 메시지 레벨에서 직접 설계한다.

요약하면, **스트리밍은 "전부 아니면 전무(all-or-nothing)"를 공짜로 주지 않는다.** 트랜잭션 경계는 애플리케이션이 메시지 의미(ID, 오프셋, ACK, 커밋 마커)로 직접 그어야 한다. 자동 재시도 정책과의 상호작용은 [[13 - 안정성 - Retry Health Check Keepalive]]에서 이어진다(특히 비멱등 스트리밍은 자동 재시도 대상에서 제외해야 한다는 점).

---

## 5.11 설계 가이드 — 언제 무엇을 고를까

네 유형을 다 이해했으니, 실제 API를 설계할 때의 선택 기준을 정리하자. 핵심 질문은 단 두 개다. **"보낼 게 여럿인가?"** 와 **"받을 게 여럿인가?"**

### 5.11.1 선택 결정 트리

```text
요청이 여러 개로 흐르나?
├─ 아니오(1개) ──── 응답이 여러 개로 흐르나?
│                  ├─ 아니오 → Unary           (요청1, 응답1)
│                  └─ 예    → Server streaming (요청1, 응답N)
└─ 예(N개) ──────── 응답이 여러 개로 흐르나?
                   ├─ 아니오 → Client streaming (요청N, 응답1)
                   └─ 예    → Bidirectional     (요청N, 응답M)
```

### 5.11.2 각 유형의 제격인 자리

| 유형 | 쓰기 좋은 상황 | 대표 예시 |
|---|---|---|
| **Unary** | 단건 조회/명령, 요청·응답이 한 덩어리로 충분 | 사용자 정보 조회, 주문 생성, 잔액 확인 |
| **Server streaming** | 결과가 많거나(페이지네이션 대체), 시간에 걸쳐 발생하는 피드 | 대량 검색 결과, 실시간 시세/날씨 피드, 로그 tail, 서버 푸시 알림 |
| **Client streaming** | 입력이 크거나 점진적으로 생기는 것을 모아서 한 결과로 | 파일/로그 업로드, 메트릭 배치 적재, 대량 import 후 요약 |
| **Bidirectional** | 양쪽이 독립적으로 계속 주고받는 대화/동기화 | 채팅, 협업 편집, 실시간 양방향 동기화, 게임 상태 교환 |

#### Unary를 기본값으로 삼아라

대부분의 API는 Unary로 충분하고, Unary가 가장 단순하다(재시도·캐싱·로드밸런싱이 다 쉽다). **스트리밍은 "Unary로는 정말 안 되는 이유"가 있을 때만** 꺼내는 카드다. "결과가 좀 많아서" 정도면 Unary + 페이지네이션(offset/cursor)이 보통 더 운영하기 쉽다. 스트리밍은 연결을 오래 잡고 로드밸런서·프록시와 궁합 문제가 있으며 부분 처리/멱등성 부담을 진다. 이름 해석과 로드밸런싱이 스트림 수명과 충돌하는 지점은 [[12 - 이름 해석과 로드밸런싱]]에서 더 본다.

#### Server streaming은 "밀어내기"가 본질일 때

조회 결과를 그냥 나눠 주는 용도라면 페이지네이션과 우열을 가려야 한다. server streaming의 진짜 강점은 **서버가 능동적으로, 시간에 걸쳐 밀어내는(push)** 경우다 — 시세 틱, 알림, 진행률 업데이트, 로그 follow처럼 "다음 데이터가 언제 생길지 클라가 모르는" 상황. 이때 Unary 폴링(polling)을 server streaming으로 바꾸면 지연과 부하가 동시에 준다.

#### Client streaming은 "모아서 한 번에 결론"일 때

업로드처럼 입력이 점진적으로 생기고, 마지막에 **단 하나의 요약/확인**이 필요할 때 딱 맞는다. 흐름 제어가 자동으로 작동하므로 거대한 단일 메시지보다 메모리 친화적이다(5.11.4).

#### Bidirectional은 "진짜 대화"일 때만

양쪽이 서로의 메시지에 반응하며 계속 주고받아야 할 때만 쓴다. bidi는 가장 비싸고(동시성·교착·취소 관리) 가장 디버깅하기 어렵다.

### 5.11.3 안티패턴 — 흔한 오남용

1. **무한 bidi를 메시지 브로커/pub-sub 대용으로**: "모든 클라가 bidi 스트림 하나씩 열어두고 모든 이벤트를 그 위로 받게 하자"는 유혹. gRPC 스트림은 *연결에 묶인* 1:1 채널이다. Kafka/NATS/Redis 같은 진짜 브로커의 영속성, 팬아웃, 재전송, 컨슈머 그룹, 오프셋 관리를 흉내 내려 하면 금세 한계에 부딪힌다. 게다가 로드밸런서가 스트림을 한 백엔드에 고정(sticky)하므로 수평 확장과 충돌한다. **pub-sub이 필요하면 pub-sub 미들웨어를 써라.** server streaming으로 "구독"을 노출하되, 그 뒤에 진짜 브로커를 두는 게 건강한 절충이다.
2. **거대한 단일 메시지 vs 청크 스트리밍의 오판**: 100MB 파일을 `bytes` 필드 하나에 통째로 담아 Unary로 보내는 것. gRPC 기본 수신 한도(4MiB)에 막히고, 한도를 올려도 그 100MB가 통째로 메모리에 올라간다(직렬화 시 두 배). 큰 데이터는 **client streaming으로 청크를 흘려보내라.** 흐름 제어가 메모리를 지켜준다. 반대로, 작은 데이터를 굳이 잘게 쪼개 수천 개 메시지로 스트리밍하면 프레이밍 오버헤드(메시지당 5바이트 + HTTP/2 프레임 헤더)와 라운드트립이 낭비다. **청크 크기는 보통 16KB~64KB 선**에서 타협한다.
3. **순서를 여러 Unary 병렬 호출에 기대기**(5.7.2): 순서가 의미 있으면 한 스트림에 담거나 직렬화하라.
4. **스트리밍을 "그냥 빠를 것 같아서" 선택**: 단건 요청-응답이면 Unary가 거의 항상 더 빠르고 단순하다. 스트림 수립 자체에도 비용이 있다.
5. **bidi에서 한 스레드로 Send/Recv 순차 처리**(5.9): 교착의 지름길.

### 5.11.4 "거대한 메시지"와 "스트리밍 청크"를 수치로 비교

```text
방식 A) Unary + 거대한 단일 메시지 (100MB)
  - 송신: 100MB를 한 번에 직렬화 → 메모리 ~200MB 순간 점유
  - 수신: maxReceiveMessageSize 한도(기본 4MiB)에 막힘 → 한도 상향 필요
  - 실패 시: 처음부터 전부 재전송 (부분 진행 없음)
  - 흐름 제어: 사실상 무의미 (한 덩어리)

방식 B) Client streaming + 32KB 청크 (~3,125개)
  - 송신: 32KB씩만 메모리에 → 일정한 작은 메모리
  - 수신: 청크마다 한도 검사 통과 (각 32KB ≪ 4MiB)
  - 실패 시: 마지막 커밋 청크부터 재개 가능 (오프셋 설계 시)
  - 흐름 제어: 느린 디스크/소비자에 맞춰 자동 감속
  → 큰 페이로드는 거의 항상 B가 우월
```

---

## 5.12 구체 예시로 굳히기

마지막으로 세 가지 유형을 현실적인 시나리오로 한 번 더 새긴다.

### 5.12.1 Server streaming — 날씨/시세 피드

"한 번 구독하면 서버가 새 데이터를 계속 밀어준다"의 교과서적 예.

```proto
service WeatherService {
  // 한 도시를 구독하면, 관측이 갱신될 때마다 서버가 push
  rpc Watch(WatchRequest) returns (stream WeatherUpdate);
}
message WatchRequest  { string city = 1; }
message WeatherUpdate {
  string city        = 1;
  double temp_c      = 2;
  int32  humidity    = 3;
  int64  observed_ms = 4;
}
```

```go
// 서버: 관측이 갱신될 때마다 Send. 클라가 끊으면 ctx가 Done.
func (s *weatherServer) Watch(req *WatchRequest,
    stream WeatherService_WatchServer) error {
    sub := s.subscribe(req.GetCity())
    defer s.unsubscribe(sub)
    for {
        select {
        case <-stream.Context().Done():
            return stream.Context().Err() // 클라 취소/끊김
        case upd := <-sub.updates:
            if err := stream.Send(upd); err != nil {
                return err
            }
        }
    }
}
```

핵심 포인트: 이 스트림은 **끝이 없을 수 있다**(클라가 끊거나 deadline이 올 때까지). `stream.Context().Done()`을 항상 살펴야 클라가 떠났을 때 리소스를 푼다. 폴링(매초 Unary GET)을 이걸로 바꾸면, "변화가 있을 때만" 데이터가 흐르므로 지연·부하가 함께 준다.

### 5.12.2 Client streaming — 로그 업로드/집계

"점진적으로 생기는 입력을 모아 하나의 요약으로." 5.5에서 본 `UploadLogs`가 정확히 이 자리다. 한 번 더, 멱등성을 곁들여서.

```proto
service LogIngest {
  rpc Upload(stream LogEntry) returns (UploadSummary);
}
message LogEntry {
  string dedup_key = 1; // 멱등성 키 (재시도 안전)
  string level     = 2;
  string body      = 3;
  int64  ts_ms     = 4;
}
```

```go
// 서버: dedup_key로 중복을 거르며 집계, half-close 시 요약 반환
func (s *logServer) Upload(stream LogIngest_UploadServer) error {
    seen := map[string]bool{}
    var n, errs int64
    for {
        e, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&UploadSummary{
                ReceivedCount: n, ErrorCount: errs,
            })
        }
        if err != nil { return err }
        if seen[e.GetDedupKey()] { continue } // 재시도로 인한 중복 무시
        seen[e.GetDedupKey()] = true
        n++
        if e.GetLevel() == "ERROR" { errs++ }
    }
}
```

`dedup_key` 덕분에 클라이언트가 연결 끊김 후 처음부터 다시 보내도 서버가 이미 본 항목을 건너뛴다. 이것이 5.10.3에서 말한 "트랜잭션 경계를 메시지 의미로 긋기"의 구체적 모습이다.

### 5.12.3 Bidirectional — 채팅방

양쪽이 독립적으로 계속 주고받는 전형.

```proto
service ChatService {
  rpc Join(stream Outbound) returns (stream Inbound);
}
message Outbound { string room = 1; string text = 2; } // 내가 보내는 말
message Inbound  { string room = 1; string author = 2; string text = 3; } // 받는 말
```

```go
// 서버: 받은 말은 방에 브로드캐스트, 방의 다른 말은 이 클라에게 흘려보냄
func (s *chatServer) Join(stream ChatService_JoinServer) error {
    // (1) 이 클라에게 보낼 메시지를 흘려보내는 goroutine
    out := make(chan *Inbound, 16)
    go func() {
        for msg := range out {
            if err := stream.Send(msg); err != nil { return }
        }
    }()
    // (2) 이 클라가 보내는 말을 받아 방에 뿌리는 루프
    for {
        in, err := stream.Recv()
        if err == io.EOF { return nil } // 클라가 나감
        if err != nil { return err }
        s.broadcast(in.GetRoom(), &Inbound{
            Room: in.GetRoom(), Author: s.who(stream), Text: in.GetText(),
        }, out) // 같은 방의 다른 참가자 out 채널들로 팬아웃
    }
}
```

여기서 Recv(이 사용자가 보내는 말)와 Send(다른 사용자가 보낸 말)가 **완전히 다른 소스/싱크**에 묶여 동시에 진행된다. 5.9의 분리 원칙이 그대로 적용된다 — 받기 루프와 보내기 goroutine을 따로 둔다. 그리고 이 패턴을 "모든 이벤트의 만능 버스"로 확장하려는 순간 5.11.3의 안티패턴(브로커 대용)에 빠진다는 점을 기억하라. 채팅처럼 *연결 수명에 묶인 세션 대화*에는 이상적이지만, *영속·재전송·대규모 팬아웃*이 필요하면 뒤에 진짜 브로커를 둬야 한다.

---

## 5.13 네 유형 한 장 정리표

| 항목 | Unary | Server streaming | Client streaming | Bidirectional |
|---|---|---|---|---|
| proto 시그니처 | `(Req) → (Res)` | `(Req) → (stream Res)` | `(stream Req) → (Res)` | `(stream Req) → (stream Res)` |
| 클라 메시지 수 | 1 | 1 | 0..N | 0..N |
| 서버 메시지 수 | 1 | 0..N | 1 | 0..M |
| 클라 half-close 시점 | 요청 보내자마자 | 요청 보내자마자 | 다 보낸 뒤(CloseAndRecv) | 다 보낸 뒤(CloseSend) — 그 후 수신 계속 |
| 응답 trailer 시점 | DATA 1개 뒤 | 모든 DATA 뒤 | DATA 1개 뒤 | 모든 DATA 뒤 |
| 클라 API 형태 | 함수 호출 | recv 루프(EOF까지) | send 루프 + CloseAndRecv | 동시 send/recv(분리) |
| 순서 보장 | N/A(1개) | 응답 FIFO | 요청 FIFO | 각 방향 FIFO(교차는 X) |
| 백프레셔 체감 | 거의 없음 | 큼(서버→클라) | 큼(클라→서버) | 양방향 모두 |
| 교착 위험 | 없음 | 낮음 | 낮음 | 높음(주의) |
| 대표 용도 | 조회/명령 | 피드/대량결과 | 업로드/집계 | 채팅/동기화 |

---

## 핵심 요약

- gRPC의 네 유형은 별개 기능이 아니라 **하나의 추상화("HTTP/2 스트림 위에서 양쪽이 0개 이상의 길이 앞붙임 메시지를 흘린다")** 에 `stream` 키워드로 제약을 건 네 가지 모양이다. `stream`이 요청 쪽이면 클라 N개, 응답 쪽이면 서버 N개.
- **모든 메시지는 5바이트 접두부(압축 플래그 1B + big-endian 길이 4B)** 를 달고 DATA 프레임에 실린다. 이 길이 표시가 DATA 프레임 경계와 무관하게 메시지 경계를 정의하므로, Unary와 스트리밍이 **같은 디코더**를 공유한다. Unary는 "스트림에 메시지 1개"일 뿐이다.
- **half-close(END_STREAM)는 "내 송신 방향만" 닫는다.** 연결을 끊는 게 아니다. client streaming에선 half-close가 "이제 집계해서 응답하라"는 신호이고, bidi에선 half-close 뒤에도 서버가 계속 보낼 수 있다.
- **최종 상태(`grpc-status`)는 항상 맨 끝 trailer로 도달한다.** 응답 헤더의 `:status 200`은 HTTP 도달일 뿐, gRPC 성패가 아니다. 스트리밍에서 "절반 보낸 뒤 실패"가 가능하기에 이 분리가 필연이다.
- **순서는 한 스트림 안에서 방향별 FIFO로 보장**되지만 **서로 다른 스트림(호출) 사이에는 보장이 없다.** 순서가 중요하면 한 스트림에 담거나 시퀀스 번호를 직접 넣어라.
- **스트리밍은 흐름 제어로 백프레셔가 자동 작동**한다(무한 큐가 아니다). 그래서 빠른 생산자가 느린 소비자에 맞춰 감속된다 — 단, **bidi에서 send/recv를 한 흐름으로 순차 처리하면 흐름 제어가 양쪽을 묶어 교착**한다. 받기/보내기를 분리하라.
- **스트리밍은 "전부 아니면 전무"를 공짜로 주지 않는다.** 부분 처리·재시도를 견디려면 dedup 키·오프셋·ACK 같은 멱등성/재개 장치를 메시지 의미로 직접 설계해야 한다.
- 선택의 기본값은 **Unary**다. server streaming은 "서버가 능동적으로 밀어내는 피드", client streaming은 "모아서 한 번에 결론짓는 업로드", bidi는 "진짜 양방향 대화"일 때만. **무한 bidi를 pub-sub 대용으로, 거대한 단일 메시지를 청크 스트리밍 대신 쓰는 것**이 대표 안티패턴이다.

---

## 연결 노트

- [[04 - HTTP2 깊이 보기 - 전송 계층]] — 프레임(DATA/HEADERS/RST_STREAM/WINDOW_UPDATE), 스트림 멀티플렉싱, END_STREAM 플래그의 정확한 의미.
- [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]] — 길이 앞붙임 메시지 안에 들어가는 protobuf 바이트의 인코딩 규칙.
- [[02 - Protocol Buffers 1 - 문법과 타입 시스템]] — `service`/`rpc`/`stream` 문법과 메시지 타입 정의.
- [[06 - 코드 생성과 protoc 툴체인]] — `stream` 키워드가 생성 스텁의 타입을 어떻게 바꾸는지.
- [[07 - 채널 스텁 커넥션 생명주기]] — 스트림이 올라타는 채널/커넥션의 수명과 상태.
- [[08 - 메타데이터와 인터셉터]] — 요청 헤더·응답 trailer로 실리는 메타데이터, 스트리밍 인터셉터.
- [[09 - 에러 모델 - 상태 코드와 Rich Error]] — trailer의 `grpc-status`, Trailers-Only, 스트리밍 도중 에러.
- [[10 - Deadline 취소 타임아웃]] — 스트리밍 취소·deadline의 양방향 전파.
- [[12 - 이름 해석과 로드밸런싱]] — 긴 스트림이 로드밸런싱·연결 고정과 충돌하는 지점.
- [[13 - 안정성 - Retry Health Check Keepalive]] — 스트리밍과 자동 재시도의 상호작용, 비멱등 스트림 제외.
- [[15 - 성능과 Flow Control 백프레셔]] — HTTP/2 흐름 제어 윈도우와 백프레셔의 정밀한 동작.
- [[16 - gRPC-Web과 REST 게이트웨이]] — 브라우저/REST 환경에서 스트리밍 유형이 어떻게 제한·매핑되는지.
