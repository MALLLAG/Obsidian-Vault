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

## 들어가며: "네 가지"가 아니라 "한 가지를 네 방향에서 본 것"

gRPC를 처음 배우는 사람은 대부분 이 네 가지를 별개의 기능처럼 외운다. "Unary는 보통 함수 호출이고, server streaming은 서버가 여러 개를 보내는 것이고, client streaming은 클라이언트가 여러 개를 보내는 것이고, bidi는 양쪽 모두 여러 개를 보내는 것이다." 이렇게 외우면 시험은 통과할 수 있다. 하지만 실제로 스트림이 중간에 끊기거나, 데드락이 걸리거나, 일부만 처리된 상황에서 무슨 일이 벌어지는지는 전혀 설명하지 못한다.

핵심부터 짚고 넘어가자. **gRPC의 모든 호출은 예외 없이 HTTP/2 스트림(stream) 하나 위에서 일어난다.** 그리고 그 스트림 위에서는 양방향 모두 "길이가 앞에 붙은 메시지(length-prefixed message)"를 0개 이상 보낼 수 있다. 네 가지 RPC 유형은 이 일반적인 능력에 **"클라이언트 쪽은 몇 개를 보낼 수 있는가"** 와 **"서버 쪽은 몇 개를 보낼 수 있는가"** 라는 두 축의 제약을 건 것에 지나지 않는다.

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

이 2×2 표가 이 장 전체의 지도다. `stream` 키워드가 요청 메시지 타입 앞에 붙으면 "클라이언트가 여러 개를 보낼 수 있음"을, 응답 메시지 타입 앞에 붙으면 "서버가 여러 개를 보낼 수 있음"을 뜻한다. 규칙은 이것이 전부이고, 나머지는 모두 이 한 문장에서 따라 나오는 결과(corollary)다.

왜 이렇게 설계했을까? RPC는 원래 "원격 함수 호출"이라는 개념이다. 함수는 인자 하나(또는 인자 묶음)를 받아 값 하나를 돌려주는데, 이것이 Unary다. 하지만 실제 분산 시스템에서는 결과가 너무 커서 한 번에 줄 수 없는 경우(대량 조회), 입력이 너무 커서 한 번에 보낼 수 없는 경우(파일 업로드), 끝이 정해지지 않은 대화(채팅, 실시간 동기화)가 흔하다.

gRPC를 만든 사람들은 이런 경우마다 별도의 프로토콜을 만드는 대신, HTTP/2가 이미 제공하는 **하나의 스트림 위에서 양방향으로 데이터를 보내는 능력**을 그대로 노출하기로 했다. 그래서 네 가지가 "따로" 있는 것이 아니라, **하나의 능력에 제약을 걸어 네 가지 형태를 만든** 것이다.

HTTP/2 자체의 프레임, 멀티플렉싱(multiplexing), 흐름 제어는 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 다뤘으므로, 이 장에서는 그 위에서 메시지가 어떻게 흐르는지에 집중한다.

---

## 5.1 네 가지 유형을 proto 시그니처로 선언하기

추상적인 설명보다 코드를 먼저 보자. 채팅, 로그, 시세를 다루는 가상의 서비스 하나에 네 유형을 모두 담았다.

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

이 한 화면에 네 유형이 모두 들어 있다. 시그니처만 비교해도 차이가 분명하다.

| 유형 | 시그니처 패턴 | 클라이언트 메시지 | 서버 메시지 |
|---|---|---|---|
| Unary | `rpc M(Req) returns (Res)` | 정확히 1 | 정확히 1 |
| Server streaming | `rpc M(Req) returns (stream Res)` | 정확히 1 | 0 이상 |
| Client streaming | `rpc M(stream Req) returns (Res)` | 0 이상 | 정확히 1 |
| Bidirectional | `rpc M(stream Req) returns (stream Res)` | 0 이상 | 0 이상 |

여기서 강조할 점이 하나 있다. **"1개 이상"이 아니라 "0개 이상"이다.** 즉 빈 스트림도 올바른 호출이다. 예를 들어 client streaming에서 클라이언트가 한 건도 보내지 않고 곧바로 스트림을 닫으면, 서버는 메시지를 0개 받은 상태에서 `UploadSummary{received_count: 0}`을 돌려주는 것이 정상이다.

server streaming에서 조건에 맞는 결과가 하나도 없으면, 서버는 DATA를 한 번도 보내지 않고 곧바로 OK trailer를 보낼 수 있다. "0개도 정상"이라는 사실은 뒤에서 빈 결과 처리, 헬스 체크, 타임아웃을 설계할 때 중요해진다.

`stream` 키워드는 **컴파일 타임에 메서드 스텁의 형태 자체를 바꾼다.** 같은 `protoc`이 같은 메서드를 Unary라면 "값을 반환하는 함수"로, server streaming이라면 "스트림 객체를 반환하는 함수"로 생성한다. 즉 네 유형의 차이는 런타임 플래그가 아니라 **생성된 API의 타입**으로 고정된다.

코드 생성과 `protoc` 툴체인의 전반적인 내용은 [[06 - 코드 생성과 protoc 툴체인]]을 참고하고, 이 장에서는 생성된 함수가 내부에서 무엇을 하는지에 집중한다.

---

## 5.2 모든 RPC는 스트림 위의 "메시지 시퀀스"다

본격적으로 들어가기 전에 네 유형에 공통인 와이어 구조를 먼저 이해해야 한다. 그래야 "Unary는 사실 메시지가 1개인 스트림"이라는 말이 비유가 아니라 사실이라는 점이 분명해진다.

### 5.2.1 HTTP/2 위에서 한 번의 호출(call)이 이루어지는 구조

하나의 gRPC 호출은 HTTP/2 스트림 하나를 새로 열면서 시작하고(클라이언트가 짝수가 아닌 홀수 stream id를 부여한다), 그 스트림이 닫히면 끝난다. 프레임(frame) 수준에서 보면 거의 항상 다음 순서를 따른다.

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

이 그림에서 알아 두어야 할 사실은 네 가지다.

1. **요청 헤더는 한 번만** 보낸다. `:path`가 곧 호출할 메서드의 전체 이름(`/패키지.서비스/메서드`)이다. gRPC에는 별도의 "메서드 이름 필드"가 없고, 메서드는 URL 경로로 표현된다.
2. **응답에는 HEADERS가 두 번** 있다. 첫 번째는 응답 헤더(initial metadata)이고, 마지막은 트레일러(trailers, 흔히 trailing metadata라고 부른다)다. 응답 헤더의 `:status: 200`은 "HTTP 수준에서 요청이 도달했다"는 뜻일 뿐이고, **실제 gRPC 호출의 성공과 실패는 마지막 트레일러의 `grpc-status`로 전달된다.**

   HTTP는 200인데 gRPC는 실패하는 이 분리가 처음에는 혼란스럽지만, 스트리밍을 생각하면 피할 수 없는 설계다. 서버가 결과를 절반쯤 보낸 뒤에 에러가 났다면, 이미 `:status` 헤더를 보낸 뒤이므로 되돌릴 수 없다. 그래서 gRPC는 최종 판정을 맨 끝의 트레일러로 미룬다. 이 에러 모델은 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에서 자세히 다룬다.
3. **END_STREAM 플래그가 "끝"을 알리는 신호**다. 클라이언트가 마지막 DATA(또는 HEADERS)에 END_STREAM을 붙이면 "내 방향의 전송은 끝났다"는 half-close가 된다. 서버가 트레일러 HEADERS에 END_STREAM을 붙이면 호출 전체가 끝난다.
4. **메시지는 DATA 프레임에 실려 전달된다.** 그런데 DATA 프레임의 경계와 메시지의 경계는 일치하지 않는다. 메시지 하나가 여러 DATA 프레임으로 나뉠 수도 있고, DATA 프레임 하나에 메시지 여러 개가 담길 수도 있다. 그래서 메시지 경계를 알아내려면 별도의 길이 표시가 필요하다. 이것을 다음 절에서 설명한다.

### 5.2.2 길이 앞붙임 메시지(length-prefixed message): gRPC의 실제 전송 단위

HTTP/2 DATA 프레임은 단순한 바이트 스트림이다. DATA 프레임은 "여기까지가 메시지 1개"라는 경계를 알려 주지 않으므로, gRPC는 애플리케이션 페이로드 안에 자체적인 프레이밍(framing)을 둔다. 모든 메시지는 앞에 **정확히 5바이트의 접두부(prefix)** 를 붙여서 전송된다.

```text
 ┌──────────┬───────────────────────────┬───────────────────────┐
 │ 1 byte   │  4 bytes (big-endian u32)  │   N bytes             │
 │ 압축 플래그 │  메시지 길이 = N             │   직렬화된 메시지 본문    │
 │ 0 또는 1  │                            │   (protobuf 바이트)    │
 └──────────┴───────────────────────────┴───────────────────────┘
   compressed-flag        length                message
```

- **압축 플래그(1바이트)**: `0x00`이면 압축하지 않은 것이고, `0x01`이면 이 메시지가 (스트림에서 협상한 압축기로) 압축된 것이다. 압축 방식은 `grpc-encoding` 헤더로 협상한다.
- **길이(4바이트, big-endian)**: 뒤따르는 메시지 본문의 바이트 수다. 부호 없는 32비트이므로 메시지 하나는 이론상 4GiB까지 가능하지만, 실제로는 수신 측의 `maxReceiveMessageSize`(기본 4MiB) 제한에 걸린다.
- **메시지 본문(N바이트)**: Protocol Buffers로 직렬화된 바이트다. 이 인코딩 자체는 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 다룬다.

이 5바이트가 중요한 이유는, **이것이 있어야 수신 측이 DATA 프레임과 관계없이 메시지 1개가 모두 도착했는지 판단할 수 있기 때문이다.** 수신 측 디코더는 항상 다음과 같이 동작한다. "먼저 5바이트를 모은다 → 길이 N을 읽는다 → 본문 N바이트가 모두 모일 때까지 기다린다 → 메시지 하나를 디코드해 애플리케이션에 전달한다 → 버퍼에 남은 바이트로 다음 5바이트를 다시 모은다." 스트리밍 전체가 이 루프 위에서 동작한다.

#### 손으로 디코딩해 보기

`Message` 대신 더 단순한 메시지로 바이트를 직접 따라가 보자. 아래 proto와 값에서 시작한다.

```proto
message Ping { string text = 2; }   // text = "hi"
```

Protocol Buffers 인코딩은 다음과 같다(자세한 규칙은 3장 인코딩과 와이어 포맷 참고).

```text
필드 2, wire type 2(LEN):  tag = (2 << 3) | 2 = 0x12
문자열 길이 2:                            0x02
"hi" =                                  0x68 0x69
→ protobuf 본문 = 12 02 68 69  (총 4바이트)
```

여기에 gRPC 프레이밍을 적용한다.

```text
압축 플래그(1B) :  00                  ← 압축 안 함
길이(4B)       :  00 00 00 04         ← 본문이 4바이트라는 뜻
본문(4B)       :  12 02 68 69
──────────────────────────────────────
와이어 바이트   :  00 00 00 00 04 12 02 68 69   (1 + 4 + 4 = 총 9바이트)
```

`bash`에서 grpcurl 같은 도구로 실제 바이트를 들여다보면 정확히 이 구조가 보인다. 메시지 두 개를 연달아 보내면 이런 블록이 그대로 이어 붙는다.

```text
메시지1 ("hi") : 00 00 00 00 04 12 02 68 69          ← 5바이트 접두부 + 본문 4B = 9B
메시지2 ("bye"): 00 00 00 00 05 12 03 62 79 65       ← 5바이트 접두부 + 본문 5B = 10B
두 메시지 연속  : 00 00 00 00 04 12 02 68 69┃00 00 00 00 05 12 03 62 79 65
                 (┃ 표시가 메시지 경계 — 와이어에는 이런 구분자가 실제로 없다.
                  오직 길이 접두부가 "다음 5+N 바이트가 한 메시지"임을 말해줄 뿐)
```

핵심은 **이 바이트 열에 "메시지 경계"를 알려 주는 구분자가 없다**는 점이다. 길이 접두부만이 "다음 5+N 바이트가 메시지 하나"라는 사실을 알려 준다. 그래서 DATA 프레임이 어디서 나뉘든 상관없다. 수신 측은 길이만 보고 메시지를 정확히 잘라 낸다. 이 단순한 규칙 덕분에 Unary와 스트리밍은 **모두 같은 디코더**를 쓴다. 스트리밍에 별도의 특별한 장치가 있는 것이 아니라, 길이 앞붙임 메시지를 1개만 읽고 끝내는지, 계속 읽는지의 차이일 뿐이다.

이제 이 공통 구조를 바탕으로 네 유형을 하나씩 살펴보자.

---

## 5.3 Unary: "스트림 위의 메시지 1개"

Unary는 가장 단순해 보이지만, "사실은 메시지가 1개인 스트림"이라는 점을 가장 먼저 익히기에 좋은 유형이다.

### 5.3.1 와이어에서 실제로 벌어지는 일

`GetMessage(GetMessageRequest) returns (Message)`를 호출하면 다음과 같은 프레임이 오간다.

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

여기서 두 가지를 확인하자.

- **요청 DATA에 곧바로 END_STREAM이 붙는다.** Unary에서 클라이언트가 보낼 메시지는 정확히 1개이므로, 그 1개를 보내자마자 "내 방향의 전송은 끝났다"고 선언한다. 이것이 half-close다. 메시지 송신과 half-close가 사실상 하나의 동작으로 합쳐진다.
- **서버는 DATA 1개와 트레일러로 호출을 끝낸다.** `grpc-status: 0`이 OK를 뜻한다.

즉 Unary가 와이어에서 보이는 모습은 5.2의 구조에서 양쪽 메시지가 1개씩인 경우일 뿐이다. **Unary 전용 프로토콜 같은 것은 없다.** 그래서 "Unary도 내부적으로는 메시지가 1개인 스트림"이라는 말은 정확하다.

### 5.3.2 Trailers-Only 응답: 에러일 때의 지름길

특수한 경우가 하나 있다. 서버가 메시지를 하나도 보내지 않고 곧바로 실패한다면(예: 인증 실패, 잘못된 인자), HTTP/2 수준에서 응답 헤더와 트레일러를 굳이 두 번 보낼 이유가 없다. 그래서 gRPC는 **Trailers-Only**라는 최적화를 허용한다. 서버는 처음이자 마지막인 HEADERS 프레임 하나에 `:status: 200`과 `grpc-status: 7`(PERMISSION_DENIED 등)을 함께 담고 END_STREAM을 붙인다. DATA 프레임은 없다.

클라이언트 구현은 DATA 없이 트레일러만 도착한 경우를 정상적으로 처리해야 한다. 이 방식은 server streaming에서 결과가 0개이면서 에러가 난 경우에도 그대로 쓰인다. 자세한 내용은 9장(에러 모델)을 참고하자.

### 5.3.3 언어별 API: "그냥 함수 호출"

Unary는 생성된 코드도 가장 단순하다. 동기(blocking) 호출이라면 정말 보통 함수처럼 보인다.

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

세 언어 모두 인자를 넣고 값을 받는 평범한 호출이다. 비동기가 필요하면 Java에서는 `FutureStub`(`ListenableFuture<Message>`)이나 `newStub`(콜백 기반 `StreamObserver`)을 쓰고, Go에서는 별도의 goroutine으로 감싸는 식으로 만든다. 하지만 와이어에서 일어나는 일은 앞의 5.3.1과 같다. **동기와 비동기는 클라이언트 코드의 형태가 다를 뿐, 프로토콜은 같다.**

> [!note] 메모
> Unary 호출에도 deadline/timeout, 메타데이터, 취소가 모두 적용된다. 이 내용은 모든 유형에 공통이므로 별도의 장에서 다룬다. [[10 - Deadline 취소 타임아웃]]과 [[08 - 메타데이터와 인터셉터]]를 참고하자.

---

## 5.4 Server streaming: 1:N, "한 번 묻고 여러 번 받는다"

`SubscribeMessages(SubscribeRequest) returns (stream Message)`에서 클라이언트는 요청을 정확히 1개 보내고, 서버는 응답을 0개 이상 보낸다. 대량 조회나 실시간 피드의 기본 형태다.

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

요청 쪽은 Unary와 똑같다. 차이는 응답 쪽에 **DATA가 여러 개**라는 점뿐이다. 각 `Message`는 5.2.2의 길이 앞붙임 규칙에 따라 자기 5바이트 접두부를 붙여 전송된다. 서버는 모두 보낸 뒤 트레일러로 종료를 알린다.

여기서 직관적으로 알아 둘 점이 하나 있다. 서버 스트리밍의 DATA는 **시간 간격을 두고** 나갈 수 있다. 시세 피드라면 1초에 한 개씩 나갈 것이다. 그동안 스트림(stream id=5)은 계속 열린 상태로 유지되고, 같은 HTTP/2 연결의 다른 스트림과 멀티플렉싱되어 같은 TCP 연결을 공유한다. 긴 스트림 하나가 연결을 독점하지 않는다는 점이 HTTP/2의 강점이다(4장 HTTP2 깊이 보기).

### 5.4.2 언어별 API: 수신 루프(recv loop)

클라이언트 쪽 코드는 스트림을 받아 모두 읽을 때까지 도는 루프가 된다.

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

`io.EOF`가 핵심이다. **EOF는 에러가 아니라 서버가 정상적으로 종료했다는 신호**다. Go의 관용구에서 `Recv()`는 다음 메시지가 있으면 그 메시지를, 스트림이 정상적으로 닫혔으면 `io.EOF`를, 비정상적으로 끝났으면 실제 에러를 돌려준다. server streaming 클라이언트가 할 일은 이 세 경우를 구분하는 것이 전부다.

서버 쪽은 보내고 싶은 만큼 Send를 호출하고 함수를 반환하면 끝난다.

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

Python의 제너레이터(`yield`)는 server streaming을 가장 간결하게 표현한다. `yield`한 값 하나가 곧 DATA 메시지 하나이고, 제너레이터가 끝나면 스트림이 닫힌다.

세 언어는 표현이 달라도 **추상화는 같다.** 서버는 Send, onNext, yield를 여러 번 호출한 뒤 끝내고, 클라이언트는 EOF가 올 때까지 Recv 루프를 돈다. 이 대칭을 이해했다면 server streaming은 다 익힌 것이다.

---

## 5.5 Client streaming: N:1, "여러 번 보내고 한 번 받는다"

`UploadLogs(stream LogEntry) returns (UploadSummary)`는 server streaming을 거울에 비춘 형태다. 클라이언트가 메시지를 0개 이상 보내고, 서버는 모두 받은 뒤 응답 1개로 답한다. 업로드, 집계, 대량 적재(batch ingest)의 기본 형태다.

### 5.5.1 와이어 흐름과 half-close의 실제 의미

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

client streaming에서 가장 중요한 것은 **half-close**다. 클라이언트가 더 보낼 메시지가 없다는 사실(`END_STREAM`)을 명시적으로 알려야, 서버가 입력이 끝났음을 알고 집계를 마무리해 응답을 만들 수 있다. half-close가 없으면 서버는 다음 로그가 더 올 수도 있다고 보고 계속 기다린다.

여기서 흔한 오해를 하나 짚고 넘어가자. **half-close는 연결을 끊는 것이 아니다.** 자신의 *송신* 방향만 닫는다는 뜻이다. half-close 이후에도 서버→클라이언트 방향은 그대로 열려 있으므로, 서버는 그 방향으로 `UploadSummary`와 트레일러를 보낼 수 있다. "half"는 양방향 중 한쪽 절반만 닫는다는 의미다. 양쪽이 모두 닫혀야(서버가 트레일러에 END_STREAM을 붙여야) 스트림이 완전히 끝난다.

### 5.5.2 언어별 API: 송신 루프와 CloseAndRecv

클라이언트는 Send를 여러 번 호출한 뒤, CloseAndRecv로 스트림을 닫으면서 결과를 받는다.

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

`CloseAndRecv()`라는 이름이 동작을 그대로 설명한다. **Close**(더 보내지 않음, 즉 half-close) **And Recv**(서버가 주는 단 하나의 응답을 받음)다. 두 동작을 하나로 묶은 이유는 client streaming에서 응답이 정확히 1개이므로, 닫는 순간이 곧 응답을 받는 순간이기 때문이다.

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

Java의 이 API는 본질적으로 비동기이므로, 요청용과 응답용 `StreamObserver` 두 개가 짝을 이룬다. 요청 옵저버의 `onCompleted()`가 곧 half-close다.

```python
# Python — client streaming 클라이언트
def gen():
    for e in log_entries:
        yield e            # 보낼 메시지를 제너레이터로 흘린다
summary = stub.UploadLogs(gen())   # 제너레이터 소진 = half-close
print(summary.received_count)
```

서버 구현은 수신 루프를 돌며 메시지를 모두 받은 뒤, 마지막에 응답 1개를 돌려준다.

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

서버에서도 대칭이 보인다. 클라이언트의 `CloseAndRecv`와 짝을 이루는 것이 서버의 `SendAndClose`다. 클라이언트가 보낸 EOF를 받은 시점이 곧 응답을 만들어 보내고 스트림을 닫을 시점이다.

> [!note] 실전 주의
> client streaming에서 서버는 메시지를 *모두 받기 전에* 먼저 응답하고 끝낼 수도 있다(예: 첫 메시지가 잘못된 것을 보고 곧바로 `INVALID_ARGUMENT`로 종료). 그러면 클라이언트의 `Send`가 도중에 에러를 반환한다. 이때 클라이언트는 `Send` 에러 자체보다 `CloseAndRecv`(또는 응답 옵저버의 `onError`)가 전달하는 **실제 status를 신뢰**해야 한다. 부분 처리와 멱등성은 5.10에서 이어서 다룬다.

---

## 5.6 Bidirectional streaming: N:M, "동시에 말하고 동시에 듣는다"

`Chat(stream ChatMessage) returns (stream ChatMessage)`에서는 양쪽 모두 메시지를 0개 이상 보낸다. 가장 강력하면서도 가장 잘못 쓰기 쉬운 유형이다.

### 5.6.1 두 방향은 실제로 독립적이다

bidi의 가장 중요한 성질은 **송신 방향과 수신 방향이 완전히 독립적**이라는 점이다. 클라이언트가 메시지를 모두 보낸 뒤에야 서버가 응답을 시작할 수도 있고(그러면 사실상 client→server 다음에 server→client가 이어진다), 클라이언트가 보내는 도중에 서버가 응답할 수도 있다(실제 양방향 대화). 어떤 인터리빙(interleaving)을 쓸지는 **전적으로 애플리케이션이 정한다.** gRPC는 두 방향이 같은 스트림 위에서 독립적으로 흐를 수 있는 능력만 제공한다.

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

이 그림에서 가장 흥미로운 줄은 클라이언트가 half-close(`DATA END_STREAM`)를 한 *뒤에도* 서버가 `s4`를 보내는 부분이다. half-close는 **한쪽 방향만** 닫으므로, 클라이언트가 입력을 끝냈더라도 서버는 출력을 더 보낼 수 있다. 이 비대칭은 bidi 설계의 장점이면서 함정이기도 하다.

### 5.6.2 언어별 API: 동시 send/recv

bidi에서는 보내기와 받기를 **동시에** 처리해야 하므로, 거의 항상 별도의 실행 흐름(goroutine, thread, async task)이 필요하다.

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

여기서 `CloseSend()`는 client streaming의 half-close와 같은 역할을 한다. 다만 bidi에서는 그 뒤에도 수신을 계속하므로, **CloseSend 이후에도 Recv 루프는 EOF를 받을 때까지 계속 동작한다.** 송신 goroutine과 수신 goroutine을 따로 두는 이유가 바로 이것이다.

goroutine 하나에서 `Send`와 `Recv`를 번갈아 호출하면, 상대가 보내 주기를 기다리는 동안 내가 보내야 할 메시지를 보내지 못해 교착(deadlock)에 빠지기 쉽다(5.9에서 자세히 설명한다).

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

Java에서는 콜백(응답 `StreamObserver`)이 수신을 처리하므로, 두 실행 흐름이 명시적인 스레드가 아니라 콜백 구조 안에 숨어 있다. 하지만 개념은 같다. 보내는 코드와 받는 콜백이 서로 독립적으로 진행된다.

```python
# Python — bidi 클라이언트
def outgoing_gen():
    for text in messages:
        yield demo_pb2.ChatMessage(room_id="room-42", author="me", text=text)

# 보내기는 제너레이터, 받기는 응답 이터레이터
for reply in stub.Chat(outgoing_gen()):
    print(reply.author, reply.text)
```

서버 구현은 받으면서 동시에 보내는 형태다.

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

이 서버는 받자마자 응답하는 단순한 에코 서버다. 실제 채팅 서버라면 받은 메시지를 다른 클라이언트의 스트림으로 팬아웃(fan-out)하고, 이 스트림으로는 *다른* 사용자가 보낸 메시지를 보낼 것이다. 즉 Recv와 Send가 서로 다른 소스와 싱크에 연결된다.

---

## 5.7 메시지 순서: 한 스트림 안에서는 FIFO, 스트림 사이에는 보장 없음

이 절은 짧지만, 분산 시스템에서 버그가 자주 생기는 원인을 정확히 다룬다.

### 5.7.1 한 스트림 내부: FIFO 보장

**하나의 RPC 호출(즉 하나의 HTTP/2 스트림) 안에서 한 방향의 메시지는 보낸 순서 그대로 도착한다.** 서버가 `m1, m2, m3` 순서로 Send했다면 클라이언트는 정확히 `m1, m2, m3` 순서로 Recv한다. 나중에 보낸 메시지가 앞지르거나 순서가 바뀌는 일은 없다.

이 순서가 보장되는 이유는 세 계층의 순서 보장이 합쳐졌기 때문이다.

1. TCP가 바이트 스트림의 순서를 보장한다.
2. HTTP/2의 한 스트림에 속한 DATA 프레임은 그 TCP 바이트 스트림 위에 순서대로 배치된다.
3. gRPC의 길이 앞붙임 프레이밍은 그 바이트 순서대로 메시지를 잘라 낸다.

어느 계층도 같은 스트림 내부의 순서를 바꾸지 않으므로, 결과적으로 메시지 단위의 FIFO가 성립한다.

이 보장은 **방향별로** 적용된다. 클라이언트→서버 방향의 순서와 서버→클라이언트 방향의 순서는 각각 보장되지만, 두 방향 사이의 상대적인 타이밍은 보장되지 않는다(bidi에서 클라이언트의 c2와 서버의 s1 중 무엇이 먼저인지는 정해지지 않는다).

### 5.7.2 스트림 사이: 순서 보장 없음

서로 다른 RPC 호출(서로 다른 스트림 id) 사이에는 **순서가 전혀 보장되지 않는다.** 같은 채널 위에서 멀티플렉싱되더라도, 호출 A를 먼저 시작했다고 해서 A의 응답이 B의 응답보다 먼저 온다는 보장은 없다. HTTP/2 스트림은 각자 독립적으로 진행되고, 서버의 처리 시간도 제각각이며, 흐름 제어 윈도우(flow-control window)도 스트림마다 따로 관리된다.

```text
잘못된 가정:  "주문 생성 RPC를 먼저, 결제 RPC를 나중에 호출했으니
              서버에서도 주문이 먼저 처리되겠지"  ← 틀림

올바른 모델:  순서가 중요하면
              (a) 같은 스트림 안에 순서대로 흘려보내거나
              (b) 명시적 시퀀스 번호/버전을 메시지에 담거나
              (c) 한 RPC가 끝난 뒤 다음을 호출(직렬화)
```

실무에서 얻을 수 있는 결론은 이렇다. 순서가 의미를 가지는 일련의 작업을 **하나의 스트리밍 RPC** 안에 담으면 별도 비용 없이 FIFO를 얻는다. 반대로 여러 Unary 호출을 병렬로 보내면서 순서를 기대하면 순서가 깨진다. 이 점은 bidi로 상태 동기화를 설계할 때 특히 중요하다. 변경 이벤트를 순서대로 보내려면 한 스트림에 실어야 한다.

---

## 5.8 흐름 제어와 백프레셔: 스트리밍에서 실제로 체감된다

Unary에서는 메시지가 1개뿐이라 흐름 제어를 거의 의식하지 않는다. 하지만 스트리밍에서는 생산 속도가 소비 속도보다 빠르면 곧바로 문제가 된다. 빠른 생산자가 느린 소비자에게 메시지를 한없이 밀어 넣으면 메모리가 부족해진다. 이를 막는 것이 **백프레셔(backpressure, 역압)** 이고, gRPC는 이를 HTTP/2 흐름 제어 위에서 구현한다.

먼저 직관부터 잡아 두자. HTTP/2는 스트림마다, 그리고 연결 전체에도 **수신 윈도우(receive window)** 를 둔다. 수신자가 아직 처리하지 못한 데이터 때문에 윈도우가 0이 되면, 송신자는 `WINDOW_UPDATE` 프레임으로 윈도우가 다시 열릴 때까지 **DATA를 더 보낼 수 없다.**

gRPC 라이브러리는 이 신호를 애플리케이션 API까지 전달한다. 예를 들어 Go에서 `stream.Send()`는 흐름 제어 윈도우가 막혀 있으면 그 자리에서 블록된다(비동기 구현에서는 "준비 안 됨" 신호로 나타난다). 즉 **느린 소비자가 자동으로 빠른 생산자의 속도를 늦춘다.**

```text
빠른 서버(생산)                    느린 클라(소비)
  Send(m1) ─DATA─►  [윈도우 차감]     수신 버퍼에 쌓임, 아직 Recv 안 함
  Send(m2) ─DATA─►  [윈도우 차감]
  Send(m3) ─DATA─►  [윈도우=0]        ← 더 못 보냄!
  Send(m4) ......블록(대기)
                       ◄─WINDOW_UPDATE─ 클라가 Recv로 소비 → 윈도우 회복
  Send(m4) ─DATA─►  [재개]
```

핵심은 **streaming이 큐를 한없이 채우는 채널이 아니라는 점**이다. streaming은 흐름 제어가 적용되는 유한한 파이프다. 그래서 server streaming에서 1억 건을 stream.Send로 쏟아 내는 코드도, 클라이언트가 천천히 읽는 한 메모리가 폭발하지 않고 자연스럽게 속도가 맞춰진다. 반대로 흐름 제어를 의식하지 않고 보내는 쪽이 모두 보낼 때까지 받지 않는 구조를 만들면, 흐름 제어가 막히면서 데드락처럼 보이는 정지가 생긴다(5.9).

흐름 제어의 윈도우 계산, BDP 기반 자동 튜닝, `WINDOW_UPDATE`의 정확한 동작은 [[15 - 성능과 Flow Control 백프레셔]]에서 자세히 다룬다. 여기서는 스트리밍을 쓰면 백프레셔가 자동으로 작동한다는 점만 기억하면 충분하다.

---

## 5.9 bidi의 동시성 모델과 데드락 회피

bidi가 강력한 만큼 위험하기도 한 이유는 **양쪽이 send와 recv를 동시에 처리해야 하는데, 이 둘이 흐름 제어로 서로 묶일 수 있기** 때문이다. 가장 흔한 사고는 교착(deadlock)이다.

### 5.9.1 교착이 생기는 전형적인 패턴

다음 시나리오를 보자. 클라이언트를 메시지를 모두 보낸 다음에야 응답을 받기 시작하도록 작성했다고 하자(단일 스레드, 순차 실행).

```text
클라이언트 로직:                      서버 로직:
  for m in 1..1_000_000:               for in 1..:
      stream.Send(m)   ← 여기서 막힘       msg = stream.Recv()
  // 다 보낸 뒤에야:                          stream.Send(reply) ← 여기서 막힘
  for { stream.Recv() }                  // 클라가 Recv를 안 해서 윈도우 0
```

무슨 일이 벌어지는지 순서대로 따라가 보자.

1. 클라이언트가 계속 `Send`한다. 서버는 메시지를 받자마자 `reply`를 `Send`하려고 한다.
2. 하지만 클라이언트는 모두 보낼 때까지 `Recv`를 하지 않는다. 그래서 **서버→클라이언트 방향의 흐름 제어 윈도우가 0**이 된다.
3. 윈도우가 0이므로 서버의 `Send(reply)`가 블록된다. 서버는 블록된 상태라 다음 `Recv`도 하지 못한다.
4. 서버가 `Recv`를 멈췄으므로 이번에는 **클라이언트→서버 방향의 윈도우**도 0이 된다. 그래서 클라이언트의 `Send`도 블록된다.
5. 양쪽 모두 상대가 읽어 주기를 기다리며 영원히 멈춘다. 이것이 **교착**이다.

이 버그는 예외도 발생하지 않고 그냥 멈추기 때문에 진단하기가 가장 까다로운 종류다. CPU 사용률은 0%이고 로그도 조용한데 진행이 되지 않는다.

### 5.9.2 회피 원칙: send와 recv를 분리한다

규칙은 단순하다. **bidi에서는 보내는 흐름과 받는 흐름을 독립적으로 진행시켜야 한다.** 한 흐름이 막히더라도 다른 흐름은 계속 소비하거나 생산할 수 있어야 한다.

- **Go**: 5.6.2처럼 `Recv` 루프를 별도의 goroutine에 두고, 메인 goroutine은 `Send`만 한다. 그러면 서버가 보내는 응답을 그 goroutine이 꾸준히 소비하므로 윈도우가 닫히지 않는다.
- **Java**: 응답 `StreamObserver`(콜백)가 별도의 실행기 스레드에서 호출되므로 구조적으로 분리되어 있다. 단, 콜백 안에서 다시 블로킹 `Send`를 호출하면 같은 함정에 빠질 수 있으므로 주의해야 한다.
- **Python**: 동기 API에서 send 제너레이터와 응답 이터레이터를 같은 스레드에서 순차적으로 다루면 위험하다. 보내기와 받기를 각각 별도의 스레드나 태스크로 분리하거나, asyncio API(`grpc.aio`)로 `await send`와 `await recv`를 동시에 처리한다.

추가 원칙은 다음과 같다.

1. **상대가 보낸 메시지를 항상 꾸준히 소비한다.** 받은 메시지를 당장 쓰지 않더라도 일단 Recv해서 윈도우를 열어 줘야 상대가 막히지 않는다. 보내기만 하면 된다고 생각해 Recv를 소홀히 하면 윈도우가 닫혀 양쪽이 멈춘다.
2. **half-close 순서를 미리 약속한다.** 보통 "클라이언트가 보낼 메시지를 모두 보내고 CloseSend → 서버가 남은 응답을 모두 보내고 종료"와 같은 프로토콜을 메시지 의미로 정해 둔다. 누가 먼저 끝내는지 모호하면 양쪽 모두 상대를 기다리다 멈춘다.
3. **끝이 없는 bidi에는 반드시 종료 조건이나 취소를 둔다.** deadline이나 context 취소가 없으면 한쪽이 죽었을 때 다른 쪽이 영원히 기다린다(10장 Deadline 취소 타임아웃).

> [!note] 직관
> bidi는 전화 통화와 같다. 두 사람이 동시에 말하고 동시에 들을 수 있다. 하지만 둘 다 상대가 말을 끝낼 때까지 듣지 않겠다고 고집하면 통화가 멈춘다. 듣기 담당과 말하기 담당을 분리하는 것이 해결책이다.

---

## 5.10 스트리밍 도중의 에러와 취소: 부분 처리와 멱등성

스트리밍에서 가장 까다로운 사실은 **스트림이 절반쯤 진행된 뒤에도 에러가 발생할 수 있다는 점**이다. Unary에서는 성공 아니면 실패로 깔끔하게 나뉘지만, 스트리밍에서는 100건 중 60건을 처리한 뒤 실패하는 일이 생길 수 있다. 이 점은 설계에 큰 영향을 준다.

### 5.10.1 최종 상태는 언제나 trailer로 도달한다

5.2.1에서 봤듯이 gRPC의 최종 판정(`grpc-status`)은 **맨 끝의 트레일러**로 전달된다. 이 설계는 스트리밍에서 특히 유용하다. 서버가 `m1..m60`을 DATA로 보낸 뒤 DB 장애로 실패하면, `:status 200`과 60개의 DATA는 이미 전송된 상태다. 서버는 그 뒤에 `grpc-status: 14`(UNAVAILABLE) 트레일러를 보내, 지금까지 보낸 60개는 유효하지만 호출은 실패로 끝났다고 알린다. 클라이언트의 Recv 루프는 정상 메시지를 60번 받은 뒤, 61번째 Recv에서 EOF가 아니라 **에러**를 받는다.

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

그래서 클라이언트는 **이미 받아서 처리한 부분이 있다는 사실과 호출은 실패했다는 사실을 함께 고려해** 다음 동작을 결정해야 한다. 이것이 부분 처리(partial processing) 문제다.

### 5.10.2 취소(cancellation)도 양방향으로 전파된다

클라이언트가 도중에 취소하면(context cancel, deadline 초과, 또는 명시적인 `cancel()`), HTTP/2 `RST_STREAM` 프레임이 전송되어 그 스트림을 즉시 끊는다. 서버 쪽 핸들러는 다음 `Send`나 `Recv`에서 취소 에러를 받거나, context가 Done 상태가 된 것을 보고 작업을 중단할 수 있어야 한다. 반대로 서버가 먼저 끝내면 클라이언트에서 진행 중인 Send가 실패한다.

핵심은 **한쪽에서 취소가 일어나면 다른 쪽도 (다음 I/O 시점에) 그 사실을 알게 된다**는 점이다. deadline과 취소의 전파 방식은 10장(Deadline 취소 타임아웃)에서 자세히 다룬다.

### 5.10.3 그래서 멱등성(idempotency)을 설계에 넣어야 한다

부분 처리와 취소 가능성을 함께 고려하면 결론은 하나다. **스트리밍 작업은 재시도해도 안전한지를 처음부터 따져 봐야 한다.**

- **client streaming 업로드**: 클라이언트가 80건을 보낸 뒤 연결이 끊겼다. 재시도하면 그 80건이 다시 들어갈 수 있다. 해결 방법은 각 항목에 고유 ID(예: `LogEntry`의 `dedup_key`)를 넣어 서버가 중복을 무시하게 하거나, 서버가 "여기까지 받았다"는 커밋 지점을 응답으로 알려 클라이언트가 그 뒤부터 재개하게 하는 것이다.
- **server streaming 조회**: 60건을 받은 뒤 끊겼다. 재시도하면 처음부터 다시 받는다. 해결 방법은 커서나 오프셋(예: `SubscribeRequest`의 `resume_from`)을 두어 마지막으로 받은 지점 이후만 다시 받는 것이다.
- **bidi 동기화**: 양쪽 모두 일부만 진행된 상태라 더 복잡하다. 해결 방법은 시퀀스 번호와 ACK 프로토콜을 메시지 수준에서 직접 설계하는 것이다.

정리하면, **스트리밍은 "전부 아니면 전무(all-or-nothing)"를 저절로 보장하지 않는다.** 트랜잭션 경계는 애플리케이션이 메시지 의미(ID, 오프셋, ACK, 커밋 마커)로 직접 정해야 한다. 자동 재시도 정책과의 관계는 [[13 - 안정성 - Retry Health Check Keepalive]]에서 이어서 다룬다(특히 멱등하지 않은 스트리밍은 자동 재시도 대상에서 빼야 한다는 점).

---

## 5.11 설계 가이드: 언제 무엇을 고를까

네 유형을 모두 살펴봤으니, 실제 API를 설계할 때의 선택 기준을 정리하자. 핵심 질문은 두 개뿐이다. **"보낼 것이 여러 개인가?"** 와 **"받을 것이 여러 개인가?"** 이다.

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

### 5.11.2 각 유형이 잘 맞는 상황

| 유형 | 쓰기 좋은 상황 | 대표 예시 |
|---|---|---|
| **Unary** | 단건 조회나 명령처럼 요청과 응답이 한 덩어리로 충분한 경우 | 사용자 정보 조회, 주문 생성, 잔액 확인 |
| **Server streaming** | 결과가 많거나(페이지네이션 대체), 시간에 걸쳐 발생하는 피드 | 대량 검색 결과, 실시간 시세/날씨 피드, 로그 tail, 서버 푸시 알림 |
| **Client streaming** | 크거나 점진적으로 생기는 입력을 모아 하나의 결과로 만드는 경우 | 파일/로그 업로드, 메트릭 배치 적재, 대량 import 후 요약 |
| **Bidirectional** | 양쪽이 독립적으로 계속 주고받는 대화나 동기화 | 채팅, 협업 편집, 실시간 양방향 동기화, 게임 상태 교환 |

#### Unary를 기본값으로 삼는다

대부분의 API는 Unary로 충분하고, Unary가 가장 단순하다(재시도, 캐싱, 로드밸런싱이 모두 쉽다). **스트리밍은 Unary로는 정말 안 되는 이유가 있을 때만** 선택해야 한다. 결과가 조금 많은 정도라면 보통은 Unary와 페이지네이션(offset/cursor)을 함께 쓰는 편이 운영하기 쉽다.

스트리밍은 연결을 오래 유지하고, 로드밸런서나 프록시와 잘 맞지 않는 문제가 있으며, 부분 처리와 멱등성 부담도 생긴다. 이름 해석과 로드밸런싱이 스트림 수명과 충돌하는 지점은 [[12 - 이름 해석과 로드밸런싱]]에서 더 살펴본다.

#### Server streaming은 "밀어내기"가 본질일 때

조회 결과를 단순히 나눠 주는 용도라면 페이지네이션과 비교해 어느 쪽이 나은지 따져 봐야 한다. server streaming의 실제 강점은 **서버가 능동적으로, 시간에 걸쳐 데이터를 밀어내는(push)** 경우에 있다. 시세 틱, 알림, 진행률 업데이트, 로그 follow처럼 다음 데이터가 언제 생길지 클라이언트가 모르는 상황이 여기에 해당한다. 이때 Unary 폴링(polling)을 server streaming으로 바꾸면 지연과 부하가 함께 줄어든다.

#### Client streaming은 "모아서 한 번에 결론"을 낼 때

업로드처럼 입력이 점진적으로 생기고, 마지막에 **요약이나 확인 하나만** 필요할 때 잘 맞는다. 흐름 제어가 자동으로 작동하므로 거대한 단일 메시지보다 메모리를 적게 쓴다(5.11.4).

#### Bidirectional은 "실제 대화"일 때만

양쪽이 서로의 메시지에 반응하면서 계속 주고받아야 할 때만 쓴다. bidi는 비용이 가장 크고(동시성, 교착, 취소 관리) 디버깅하기도 가장 어렵다.

### 5.11.3 안티패턴: 흔한 오용 사례

1. **끝없는 bidi를 메시지 브로커나 pub-sub 대신 쓰는 것**: 모든 클라이언트가 bidi 스트림을 하나씩 열어 두고 모든 이벤트를 그 위로 받게 하자는 유혹이다. gRPC 스트림은 *연결에 묶인* 1:1 채널이다. Kafka, NATS, Redis 같은 실제 브로커의 영속성, 팬아웃, 재전송, 컨슈머 그룹, 오프셋 관리를 흉내 내려 하면 금방 한계에 부딪힌다. 게다가 로드밸런서가 스트림을 한 백엔드에 고정(sticky)하므로 수평 확장과도 충돌한다.

   **pub-sub이 필요하면 pub-sub 미들웨어를 써야 한다.** server streaming으로 "구독"을 노출하고, 그 뒤에 실제 브로커를 두는 것이 적절한 절충안이다.
2. **거대한 단일 메시지와 청크 스트리밍 사이의 잘못된 선택**: 100MB 파일을 `bytes` 필드 하나에 통째로 담아 Unary로 보내는 경우다. gRPC의 기본 수신 한도(4MiB)에 막히고, 한도를 올리더라도 100MB 전체가 메모리에 올라간다(직렬화할 때는 두 배). 큰 데이터는 **client streaming으로 청크를 나눠 보내야 한다.** 그러면 흐름 제어가 메모리 사용을 제한해 준다.

   반대로 작은 데이터를 굳이 잘게 쪼개 수천 개의 메시지로 스트리밍하면 프레이밍 오버헤드(메시지당 5바이트와 HTTP/2 프레임 헤더)와 왕복 시간이 낭비된다. **청크 크기는 보통 16KB~64KB 정도**로 타협한다.
3. **여러 Unary 병렬 호출에 순서를 기대하는 것**(5.7.2): 순서가 의미를 가진다면 한 스트림에 담거나 호출을 직렬화해야 한다.
4. **"그냥 빠를 것 같아서" 스트리밍을 선택하는 것**: 단건 요청과 응답이라면 Unary가 거의 항상 더 빠르고 단순하다. 스트림을 수립하는 데에도 비용이 든다.
5. **bidi에서 스레드 하나로 Send와 Recv를 순차 처리하는 것**(5.9): 교착으로 가는 지름길이다.

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

## 5.12 구체적인 예시로 정리하기

마지막으로 세 가지 유형을 현실적인 시나리오로 한 번 더 살펴본다.

### 5.12.1 Server streaming: 날씨/시세 피드

한 번 구독하면 서버가 새 데이터를 계속 밀어 주는 방식의 전형적인 예다.

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

핵심은 이 스트림이 **끝나지 않을 수도 있다**는 점이다(클라이언트가 끊거나 deadline이 될 때까지 계속된다). `stream.Context().Done()`을 항상 확인해야 클라이언트가 떠났을 때 리소스를 해제할 수 있다. 폴링(매초 Unary GET)을 이 방식으로 바꾸면 변화가 있을 때만 데이터가 흐르므로 지연과 부하가 함께 줄어든다.

### 5.12.2 Client streaming: 로그 업로드/집계

점진적으로 생기는 입력을 모아 하나의 요약으로 만드는 경우다. 5.5에서 본 `UploadLogs`가 정확히 여기에 해당한다. 이번에는 멱등성을 더해서 다시 살펴보자.

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

`dedup_key` 덕분에 클라이언트가 연결이 끊긴 뒤 처음부터 다시 보내도, 서버는 이미 받은 항목을 건너뛴다. 5.10.3에서 말한 "트랜잭션 경계를 메시지 의미로 정하기"가 구체적으로 이런 모습이다.

### 5.12.3 Bidirectional: 채팅방

양쪽이 독립적으로 메시지를 계속 주고받는 전형적인 예다.

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

여기서 Recv(이 사용자가 보내는 말)와 Send(다른 사용자가 보낸 말)는 **완전히 다른 소스와 싱크**에 연결되어 동시에 진행된다. 5.9의 분리 원칙이 그대로 적용되어, 받기 루프와 보내기 goroutine을 따로 둔다.

그리고 이 패턴을 모든 이벤트를 나르는 만능 버스로 확장하려는 순간 5.11.3의 안티패턴(브로커 대용)에 빠진다는 점을 기억하자. 이 패턴은 채팅처럼 *연결 수명에 묶인 세션 대화*에는 이상적이지만, *영속성, 재전송, 대규모 팬아웃*이 필요하다면 그 뒤에 실제 브로커를 두어야 한다.

---

## 5.13 네 유형 한 장 정리표

| 항목 | Unary | Server streaming | Client streaming | Bidirectional |
|---|---|---|---|---|
| proto 시그니처 | `(Req) → (Res)` | `(Req) → (stream Res)` | `(stream Req) → (Res)` | `(stream Req) → (stream Res)` |
| 클라 메시지 수 | 1 | 1 | 0..N | 0..N |
| 서버 메시지 수 | 1 | 0..N | 1 | 0..M |
| 클라 half-close 시점 | 요청 보내자마자 | 요청 보내자마자 | 다 보낸 뒤(CloseAndRecv) | 다 보낸 뒤(CloseSend), 그 후에도 수신 계속 |
| 응답 trailer 시점 | DATA 1개 뒤 | 모든 DATA 뒤 | DATA 1개 뒤 | 모든 DATA 뒤 |
| 클라 API 형태 | 함수 호출 | recv 루프(EOF까지) | send 루프 + CloseAndRecv | 동시 send/recv(분리) |
| 순서 보장 | N/A(1개) | 응답 FIFO | 요청 FIFO | 각 방향 FIFO(교차는 X) |
| 백프레셔 체감 | 거의 없음 | 큼(서버→클라) | 큼(클라→서버) | 양방향 모두 |
| 교착 위험 | 없음 | 낮음 | 낮음 | 높음(주의) |
| 대표 용도 | 조회/명령 | 피드/대량결과 | 업로드/집계 | 채팅/동기화 |
