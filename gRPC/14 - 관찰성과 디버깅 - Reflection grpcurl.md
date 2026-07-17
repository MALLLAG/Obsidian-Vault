---
title: "관찰성과 디버깅 - Reflection grpcurl"
date: 2026-06-26
tags:
  - grpc
  - observability
  - debugging
  - server-reflection
  - grpcurl
  - grpcui
  - evans
  - channelz
  - opentelemetry
  - distributed-tracing
  - traceparent
  - wireshark
  - grpc-trace
  - 학습노트
---

**이 장이 답하는 질문**

- 바이너리 protobuf를 쓰는 gRPC 서버에 어떻게 `.proto` 파일 하나 없이 접속해서 메서드를 호출할 수 있는가? (서버 리플렉션의 원리)
- `grpcurl`, `grpcui`, `evans`, `grpc_cli`는 각각 언제 쓰고, 리플렉션이 꺼져 있으면 어떻게 디스크립터를 직접 먹이는가?
- "연결이 끊긴 것 같다"를 코드 수정 없이 런타임에서 들여다보려면(channelz), 그리고 더 깊이 C-core 트레이스를 켜려면(GRPC_TRACE) 무엇을 만지는가?
- RPC 한 건의 지연·상태·바이트를 메트릭으로 뽑고, 서비스 경계를 넘어 트레이스를 잇는(W3C traceparent) 표준은 무엇인가?
- `UNAVAILABLE`, `DEADLINE_EXCEEDED`, `UNIMPLEMENTED`가 떴을 때 무엇을 어떤 순서로 확인하는가?

---

## 14.0 들어가며: gRPC는 왜 디버깅이 "어렵다"고 느껴지는가

처음 gRPC를 만진 백엔드 엔지니어가 가장 먼저 부딪히는 벽은 익숙한 도구가 통하지 않는다는 점이다. REST/JSON 시절에는 문제가 생기면 반사적으로 손이 움직였다. `curl -v https://api.example.com/orders/42`를 치면 요청 라인, 헤더, 본문이 사람이 읽을 수 있는 텍스트로 죽 흘러나왔고, 브라우저 개발자 도구의 Network 탭에서 응답 JSON을 펼쳐 보면 끝이었다. 와이어 위를 흐르는 것이 곧 우리가 작성한 그 텍스트였기 때문이다.

gRPC에서는 그 직관이 두 번 배신당한다.

첫째, **본문이 바이너리다.** 메시지는 Protocol Buffers로 직렬화되어([[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]) 필드 이름조차 와이어에 실리지 않는다. `0a 05 41 6c 69 63 65` 같은 바이트 열을 봐도, 스키마(`.proto`)를 모르면 이게 `name = "Alice"`인지 `city = "Alice"`인지 알 길이 없다. 필드 번호 1이 무엇을 뜻하는지는 오직 `.proto`에만 적혀 있다.

둘째, **전송 계층이 HTTP/2다.** 요청은 헤더 압축(HPACK), 스트림 멀티플렉싱(multiplexing), 바이너리 프레이밍으로 잘게 쪼개진다([[04 - HTTP2 깊이 보기 - 전송 계층]]). `tcpdump`로 패킷을 떠도 HPACK으로 압축된 헤더는 사람이 바로 읽을 수 없고 하나의 TCP 연결 위에 여러 RPC가 인터리빙(interleaving)되어 있어 "이 바이트가 어느 RPC의 것인가"를 손으로 추적하기가 만만치 않다.

그래서 gRPC를 다루는 엔지니어에게 관찰성(observability)은 선택이 아니라 생존 도구다. 다행히 gRPC 생태계는 이 어려움을 정면으로 인정하고 REST가 갖지 못한 강력한 자기서술(self-description)·자기관찰(self-inspection) 메커니즘을 프로토콜 수준에서 표준화해 두었다. 이 장은 그 도구 상자를 다섯 개의 층으로 나눠 본다.

```text
 ┌───────────────────────────────────────────────────────────────┐
 │  Layer 5  와이어 레벨   Wireshark + SSLKEYLOGFILE (프레임/바이트)  │  ← 최후의 진실
 ├───────────────────────────────────────────────────────────────┤
 │  Layer 4  관측 신호     OpenTelemetry 메트릭·트레이스 (RPC 단위)   │  ← 평상시 모니터링
 ├───────────────────────────────────────────────────────────────┤
 │  Layer 3  런타임 내부   channelz + GRPC_TRACE/grpclog (연결 상태)  │  ← 라이브 디버깅
 ├───────────────────────────────────────────────────────────────┤
 │  Layer 2  대화형 호출   grpcurl / grpcui / evans (애드혹 콜)       │  ← 손으로 찔러보기
 ├───────────────────────────────────────────────────────────────┤
 │  Layer 1  스키마 발견   Server Reflection (서버가 .proto를 노출)   │  ← 모든 것의 토대
 └───────────────────────────────────────────────────────────────┘
```

아래로 갈수록 더 깊고 위로 갈수록 더 일상적이다. 우리는 가장 토대가 되는 1층, "서버가 스스로 자기 스키마를 말해주는" 서버 리플렉션부터 시작한다. 이것이 없으면 2층의 대화형 도구가 절반밖에 동작하지 않기 때문이다.

---

## 14.1 서버 리플렉션 (Server Reflection Protocol)

### 14.1.1 왜 필요한가: "스키마 없이 호출하기"라는 모순을 푸는 법

상황을 떠올려 보자. 운영 중인 서버에 문제가 생겼고 당신은 `OrderService.GetOrder`를 직접 한 번 찔러 보고 싶다. 그런데 손에는 컴파일된 서버 바이너리만 있고 `.proto` 파일은 다른 레포지토리에 있거나, 버전이 맞는지 확신이 없거나, 그냥 지금 당장 찾기 귀찮다. REST였다면 `curl`로 URL만 알면 끝났을 텐데, gRPC는 메시지를 직렬화하려면 스키마가 반드시 있어야 한다. 필드 번호와 타입을 모르면 바이트 한 줄도 만들 수 없다.

서버 리플렉션은 이 모순을 우아하게 푼다. 핵심 아이디어는 이렇다.

> **서버는 이미 자기 스키마를 알고 있다.** 컴파일될 때 모든 `.proto`가 `FileDescriptor`(파일 디스크립터)라는 형태로 바이너리 안에 박혀 들어가기 때문이다([[06 - 코드 생성과 protoc 툴체인]]). 그렇다면 서버가 이 디스크립터를 **하나의 gRPC 서비스로 외부에 노출**하면, 클라이언트는 `.proto` 없이도 서버에게 "너 무슨 서비스 있어? 그 메서드 입력 타입이 뭐야?"라고 물어볼 수 있다.

리플렉션은 **gRPC로 gRPC의 스키마를 질의하는** 메타 서비스다. 닭이 자기 알의 설계도를 들고 다니는 셈이다.

### 14.1.2 프로토콜 정의: `grpc.reflection.v1`

리플렉션 자체도 평범한 gRPC 서비스로 정의되어 있다. 서비스 이름은 `grpc.reflection.v1.ServerReflection`이며, 양방향 스트리밍(bidirectional streaming) 메서드 하나만 둔다.

```proto
// grpc/reflection/v1/reflection.proto (요지)
syntax = "proto3";
package grpc.reflection.v1;

service ServerReflection {
  // 양방향 스트림: 클라가 질의를 흘려보내면 서버가 응답을 흘려준다
  rpc ServerReflectionInfo(stream ServerReflectionRequest)
      returns (stream ServerReflectionResponse);
}

message ServerReflectionRequest {
  string host = 1;
  // 다섯 가지 질의 중 하나(oneof)
  oneof message_request {
    string file_by_filename = 3;                  // 파일명으로 디스크립터 요청
    string file_containing_symbol = 4;            // 심볼(서비스/메서드/메시지)을 담은 파일 요청
    ExtensionRequest file_containing_extension = 5;// 확장 필드를 담은 파일 요청
    string all_extension_numbers_of_type = 6;     // 특정 타입의 모든 확장 번호
    string list_services = 7;                      // 서비스 목록 요청
  }
}

message ServerReflectionResponse {
  string valid_host = 1;
  ServerReflectionRequest original_request = 2;
  oneof message_response {
    FileDescriptorResponse file_descriptor_response = 4;       // 직렬화된 FileDescriptorProto 묶음
    ExtensionNumberResponse all_extension_numbers_response = 5;
    ListServiceResponse list_services_response = 6;
    ErrorResponse error_response = 7;
  }
}

message FileDescriptorResponse {
  // 핵심: 직렬화된 google.protobuf.FileDescriptorProto 바이트들
  repeated bytes file_descriptor_proto = 1;
}
```

양방향 스트림을 쓰는 이유가 중요하다. 스키마는 **의존성 그래프**를 이룬다. `OrderService`가 정의된 `order.proto`는 `common.proto`의 `Money`를 import하고, `common.proto`는 다시 `google/protobuf/timestamp.proto`를 import한다. 클라이언트가 `file_containing_symbol`로 `OrderService`를 물으면 서버는 그 파일과 함께 **추이적 의존(transitive dependency)** 파일들을 모두 돌려줘야 클라이언트가 메시지를 온전히 조립할 수 있다. 한 번의 질의가 여러 응답을 부르고 클라이언트는 받은 디스크립터를 보고 "아직 이 의존은 안 받았네" 하며 추가 질의를 던진다. 이 다중 왕복을 한 연결에서 자연스럽게 처리하려고 양방향 스트림을 택했다.

```text
클라이언트                                      리플렉션 서버
   │                                                │
   │── list_services ──────────────────────────────▶│
   │◀──────────────── [OrderService, grpc.health…]──│
   │                                                │
   │── file_containing_symbol("OrderService") ─────▶│
   │◀── FileDescriptorResponse{ order.proto } ──────│
   │  (order.proto가 common.proto를 import한다고 적혀있음)
   │                                                │
   │── file_by_filename("common.proto") ───────────▶│
   │◀── FileDescriptorResponse{ common.proto } ─────│
   │  (common.proto가 timestamp.proto를 import)
   │                                                │
   │── file_by_filename("google/.../timestamp") ───▶│
   │◀── FileDescriptorResponse{ timestamp.proto } ──│
   │                                                │
   ▼  이제 클라이언트는 GetOrderRequest를 완전히 조립 가능
```

### 14.1.3 FileDescriptor란 무엇인가 — 스키마의 스키마

여기서 한 겹 더 들어가자. `FileDescriptorProto`는 무엇인가? 이것은 **"`.proto` 파일을 protobuf 메시지로 표현한 것"** 이다. 다시 말해 Protocol Buffers는 자기 자신의 문법을 protobuf로 기술한다. 이 자기참조 구조는 `descriptor.proto`라는 메타 파일에 정의되어 있다.

```proto
// google/protobuf/descriptor.proto (대폭 요약)
message FileDescriptorProto {
  optional string name = 1;            // "order.proto"
  optional string package = 2;         // "shop.v1"
  repeated string dependency = 3;      // ["common.proto", "google/protobuf/timestamp.proto"]
  repeated DescriptorProto message_type = 4;   // 메시지 정의들
  repeated ServiceDescriptorProto service = 6; // 서비스 정의들
  optional string syntax = 12;         // "proto3"
}

message ServiceDescriptorProto {
  optional string name = 1;            // "OrderService"
  repeated MethodDescriptorProto method = 2;
}

message MethodDescriptorProto {
  optional string name = 1;            // "GetOrder"
  optional string input_type = 2;      // ".shop.v1.GetOrderRequest"
  optional string output_type = 3;     // ".shop.v1.Order"
  optional bool client_streaming = 5;
  optional bool server_streaming = 6;
}
```

`protoc`가 `.proto`를 컴파일할 때, 코드 생성과 별개로 이 `FileDescriptorProto` 표현을 만들어 생성 코드 안에 바이트 상수로 박아 넣는다([[06 - 코드 생성과 protoc 툴체인]] 참고). 그래서 서버 프로세스는 자기 안에 모든 스키마를 디스크립터 형태로 들고 있고 리플렉션 서비스는 그저 그 바이트를 꺼내 돌려줄 뿐이다. 클라이언트(예: `grpcurl`)는 받은 `FileDescriptorProto` 묶음으로 메모리 안에 디스크립터 풀(descriptor pool)을 재구성하고 그것으로 JSON↔protobuf 변환을 수행한다.

이것이 "마법"의 정체다. 리플렉션은 새로운 정보를 만들어내지 않는다. 컴파일 타임에 이미 존재했지만 와이어에는 실리지 않던 스키마 정보를, **요청 시점에 별도 채널로 끌어오는** 것뿐이다.

### 14.1.4 v1 vs v1alpha — 버전 협상의 현실

역사적으로 리플렉션 서비스는 오랫동안 `grpc.reflection.v1alpha.ServerReflection`이라는 이름으로 제공됐다. 이후 안정 버전 `grpc.reflection.v1.ServerReflection`이 표준화됐지만, 메시지 구조는 두 버전이 사실상 동일하다(패키지 이름만 다르다). 문제는 호환성이다.

- 오래된 서버는 `v1alpha`만 노출한다.
- 최신 서버는 보통 `v1`과 `v1alpha`를 **둘 다** 등록한다(구형 클라이언트 호환을 위해).
- 최신 `grpcurl` 같은 도구는 먼저 `v1`로 시도하고, `Unimplemented`가 오면 자동으로 `v1alpha`로 폴백(fallback)한다.

그래서 실무에서 "리플렉션이 안 된다"는 증상의 절반은 버전 불일치다. 도구가 `v1`만 시도하는데 서버는 `v1alpha`만 켜져 있거나(혹은 그 반대), 폴백 로직이 없는 구형 도구를 쓰는 경우다. 의심되면 도구를 최신으로 올리거나, 서버에 양쪽을 모두 등록하면 대부분 해결된다.

### 14.1.5 서버에 리플렉션 켜기 (언어별)

리플렉션은 기본적으로 꺼져 있다. 명시적으로 등록해야 한다.

**Go**

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
)

func main() {
    s := grpc.NewServer()
    shoppb.RegisterOrderServiceServer(s, &orderServer{})

    // 이 한 줄이 grpc.reflection.v1 / v1alpha 서비스를 등록한다
    reflection.Register(s)

    lis, _ := net.Listen("tcp", ":50051")
    s.Serve(lis)
}
```

**Java**

```java
import io.grpc.protobuf.services.ProtoReflectionService;

Server server = ServerBuilder.forPort(50051)
    .addService(new OrderServiceImpl())
    .addService(ProtoReflectionService.newInstance()) // 리플렉션 등록
    .build()
    .start();
```

**Python**

```python
from grpc_reflection.v1alpha import reflection

SERVICE_NAMES = (
    shop_pb2.DESCRIPTOR.services_by_name['OrderService'].full_name,
    reflection.SERVICE_NAME,  # 리플렉션 자신도 목록에 포함
)
reflection.enable_server_reflection(SERVICE_NAMES, server)
```

### 14.1.6 보안: 프로덕션에서 리플렉션을 켜야 하는가

리플렉션은 양날의 검이다. 디버깅에는 천국이지만, 동시에 **서버의 전체 API 표면을 익명에게 광고**한다. 공격자 입장에서 리플렉션이 켜진 서버는 "여기 어떤 서비스·메서드·필드가 있는지 친절히 알려주는" 정찰(reconnaissance) 대상이다. 숨겨진 관리용 메서드, 내부 필드 이름, 타입 구조가 그대로 드러난다.

판단 기준을 정리하면 이렇다.

| 환경 | 권장 | 이유 |
|------|------|------|
| 로컬/개발 | 항상 켬 | 생산성. 외부 노출 없음 |
| 사내망 스테이징 | 켜되 네트워크 격리 | 팀 디버깅 편의 vs 내부망 신뢰 |
| 프로덕션(외부 노출) | **끔** 또는 인증 뒤에 | API 표면 은닉, 정찰 차단 |
| 프로덕션(내부 mesh) | 정책에 따라 켬 | mTLS([[11 - 보안 - TLS mTLS 인증]])로 보호된 내부 호출만 허용 시 디버깅 가치 큼 |

"리플렉션을 끈다"가 곧 보안이라는 뜻은 아니다. 리플렉션이 꺼져 있어도 공격자가 메서드 경로(`/shop.v1.OrderService/GetOrder`)를 알면 호출은 가능하다. 리플렉션은 인증·인가의 대체물이 아니라, 단지 **발견 가능성(discoverability)을 낮추는** 심층 방어(defense in depth)의 한 겹일 뿐이다. 진짜 방어는 인터셉터([[08 - 메타데이터와 인터셉터]])에서 하는 인증, mTLS, 그리고 인가 로직이다. 외부 노출 프로덕션이라면 리플렉션은 끄고 디버깅이 필요할 땐 디스크립터 셋(`.pb` 파일)을 도구에 직접 먹이는 방식(다음 절)을 쓰는 것이 정석이다.

---

## 14.2 도구 생태계: grpcurl, grpcui, evans, grpc_cli, Postman

리플렉션이라는 토대 위에서, 사람이 손으로 서버를 찔러보는 도구들이 자란다. 이들은 모두 "리플렉션(또는 디스크립터)으로 스키마를 얻고 → 사람이 친 JSON을 protobuf로 직렬화해 호출하고 → 응답 protobuf를 JSON으로 역직렬화해 보여준다"는 동일한 골격을 공유한다. 차이는 인터페이스다.

| 도구 | 형태 | 한 줄 정체성 | 강점 |
|------|------|-------------|------|
| `grpcurl` | CLI | gRPC계의 `curl` | 스크립트·일회성 호출, CI, 가장 보편적 |
| `grpc_cli` | CLI | gRPC 공식(C++) CLI | C-core 환경, 공식 배포에 포함 |
| `grpcui` | 웹 UI | gRPC계의 Postman(로컬 웹) | 폼 기반 클릭 호출, 비개발자 친화 |
| `evans` | REPL | 대화형 셸 | 탐색적 사용, 탭 자동완성, 패키지 전환 |
| Postman | GUI | API 플랫폼의 gRPC 모드 | 팀 협업, 컬렉션, 환경변수 관리 |

이 절은 가장 널리 쓰이는 `grpcurl`을 중심에 두고, 나머지를 그 변주로 설명한다.

### 14.2.1 grpcurl 기본 — 두 가지 스키마 소스

`grpcurl`의 모든 명령은 "스키마를 어디서 얻는가"라는 갈림길에서 시작한다.

```text
                    스키마를 어디서 얻나?
                          │
          ┌───────────────┴────────────────┐
          ▼                                 ▼
   (A) 서버 리플렉션                  (B) 로컬 .proto / 디스크립터 셋
   서버가 켜뒀을 때                   리플렉션이 꺼졌거나 prod일 때
   추가 플래그 없음                   -proto / -protoset / -import-path 필요
```

**(A) 리플렉션이 켜진 경우 — 가장 흔한 사용**

```bash
# 1) 서버가 노출하는 서비스 목록
grpcurl localhost:50051 list

# 출력 예:
#   grpc.reflection.v1.ServerReflection
#   grpc.health.v1.Health
#   shop.v1.OrderService

# 2) 특정 서비스의 메서드 목록
grpcurl localhost:50051 list shop.v1.OrderService

# 출력 예:
#   shop.v1.OrderService.GetOrder
#   shop.v1.OrderService.ListOrders
#   shop.v1.OrderService.CreateOrder

# 3) 메서드/메시지 스키마 상세
grpcurl localhost:50051 describe shop.v1.OrderService.GetOrder

# 출력 예:
#   shop.v1.OrderService.GetOrder is a method:
#   rpc GetOrder ( .shop.v1.GetOrderRequest ) returns ( .shop.v1.Order );

# 4) 입력 메시지 구조 보기
grpcurl localhost:50051 describe shop.v1.GetOrderRequest

# 출력 예:
#   shop.v1.GetOrderRequest is a message:
#   message GetOrderRequest {
#     string order_id = 1;
#   }
```

`list`와 `describe`는 정확히 14.1의 리플렉션 질의(`list_services`, `file_containing_symbol`)로 번역된다. `grpcurl`은 그 위에 사람이 읽기 좋은 텍스트 렌더링을 입혔다.

**실제 메서드 호출**은 `-d`(data)에 JSON을 준다. `grpcurl`이 이 JSON을 디스크립터 기반으로 protobuf로 직렬화한다(protobuf JSON 매핑 규칙을 따른다).

```bash
grpcurl -d '{"order_id": "ord-42"}' \
    localhost:50051 shop.v1.OrderService.GetOrder

# 응답(서버가 protobuf로 준 것을 grpcurl이 JSON으로 역직렬화):
# {
#   "orderId": "ord-42",
#   "status": "PAID",
#   "totalAmount": { "currencyCode": "KRW", "units": "15000" }
# }
```

> 주의: protobuf JSON 매핑은 필드 이름을 기본적으로 **lowerCamelCase**로 출력한다(`order_id` → `orderId`). 입력은 양쪽 다 받아준다. `int64`/`uint64`는 정밀도 보존을 위해 **문자열**로 직렬화된다(`"15000"`). 이건 JSON의 number가 double이라 53비트 초과 정수에서 정밀도를 잃는 문제를 피하려는 protobuf JSON 규약이다.

**메타데이터(헤더)** 는 `-H`로 싣는다. 인증 토큰이 대표적이다([[08 - 메타데이터와 인터셉터]]).

```bash
grpcurl \
    -H 'authorization: Bearer eyJhbGciOi...' \
    -H 'x-request-id: debug-001' \
    -d '{"order_id": "ord-42"}' \
    localhost:50051 shop.v1.OrderService.GetOrder
```

**스트리밍 호출**도 된다. 클라이언트 스트리밍이면 `-d`에 여러 JSON을 줄 수 있고(개행 구분 또는 `@`로 stdin), 서버 스트리밍이면 응답이 여러 개 순차 출력된다([[05 - 통신의 4가지 방식 - Unary와 Streaming]]).

```bash
# stdin에서 여러 메시지를 읽어 클라이언트 스트리밍
echo '{"order_id":"a"}
{"order_id":"b"}' | grpcurl -d @ localhost:50051 shop.v1.OrderService.BatchGet
```

### 14.2.2 TLS와 평문 — 가장 흔한 첫 실수

`grpcurl`은 **기본적으로 TLS를 가정한다.** 그런데 로컬 개발 서버는 평문(plaintext)인 경우가 많다. 그래서 초심자가 가장 먼저 만나는 에러가 이것이다.

```bash
grpcurl localhost:50051 list
# Failed to dial target host "localhost:50051": tls: first record
# does not look like a TLS handshake
```

이건 "평문 서버에 TLS로 노크했다"는 뜻이다. 평문 서버라면 `-plaintext`를 붙인다.

```bash
grpcurl -plaintext localhost:50051 list          # 평문(h2c)
grpcurl -insecure  myserver:443 list             # TLS지만 인증서 검증 생략(자가서명)
grpcurl -cacert ca.pem myserver:443 list         # 사설 CA로 검증
grpcurl -cert client.pem -key client.key \
        -cacert ca.pem myserver:443 list         # mTLS(클라 인증서 제출)
```

TLS 관련 세부는 [[11 - 보안 - TLS mTLS 인증]]에 양보한다. 여기서 기억할 것은 **`-plaintext`는 "TLS를 끈다", `-insecure`는 "TLS는 쓰되 인증서 검증만 끈다"** 로 전혀 다른 의미라는 점이다.

### 14.2.3 (B) 리플렉션 없이 — `.proto`나 디스크립터 셋 직접 먹이기

프로덕션처럼 리플렉션이 꺼진 서버를 호출하려면, 스키마를 도구에 직접 줘야 한다. 두 가지 방법이 있다.

**방법 1: `-proto`로 소스 `.proto`를 직접 파싱**

```bash
grpcurl \
    -import-path ./proto \
    -proto shop/v1/order.proto \
    -d '{"order_id":"ord-42"}' \
    -plaintext localhost:50051 shop.v1.OrderService.GetOrder
```

- `-proto`: 진입 `.proto` 파일.
- `-import-path`(별칭 `-I`): import를 해석할 루트 디렉터리. `.proto` 안의 `import "shop/v1/common.proto";`를 이 경로 기준으로 찾는다. protoc의 `-I`와 같은 개념([[06 - 코드 생성과 protoc 툴체인]]).

문제는 의존성이다. `order.proto`가 여러 파일을 import하면 그 경로를 모두 갖춰야 하고 `google/protobuf/timestamp.proto` 같은 표준 import도 찾을 수 있어야 한다(보통 도구가 well-known types는 내장한다).

**방법 2: `-protoset`으로 컴파일된 디스크립터 셋 먹이기 (권장)**

의존성 지옥을 피하는 정석은 `protoc`로 **모든 의존을 한 파일에 묶은** `FileDescriptorSet`을 미리 만드는 것이다.

```bash
# 디스크립터 셋 생성: --include_imports가 핵심(추이적 의존 전부 포함)
protoc \
    --include_imports \
    --descriptor_set_out=shop.protoset \
    -I ./proto \
    proto/shop/v1/order.proto

# 그 한 파일만으로 호출 (서버 리플렉션 불필요)
grpcurl \
    -protoset shop.protoset \
    -d '{"order_id":"ord-42"}' \
    -plaintext localhost:50051 shop.v1.OrderService.GetOrder
```

`.protoset` 파일은 그냥 `FileDescriptorSet` 메시지(= `repeated FileDescriptorProto`)를 직렬화한 바이너리다. **리플렉션이 와이어로 돌려주는 것과 본질적으로 같은 데이터**를 파일에 담아 들고 다닌다. 그래서 프로덕션 운영 패턴은 보통 이렇다.

> 리플렉션은 프로덕션에서 끈다 → 대신 CI가 `--include_imports`로 `.protoset`을 빌드 산출물로 발행한다 → 운영자는 그 `.protoset`을 `grpcurl -protoset`에 먹여 안전하게 호출한다.

이렇게 하면 API 표면을 외부에 광고하지 않으면서도, 인가된 운영자는 정확한 버전의 스키마로 디버깅할 수 있다. 하위호환성 관점의 스키마 관리는 [[17 - 실전 - proto 관리와 하위호환성]]에서 더 다룬다.

심지어 리플렉션 자체에서 디스크립터를 떠서 `.protoset`으로 저장할 수도 있다.

```bash
# 리플렉션 켜진 dev 서버에서 디스크립터를 떠서 파일로 저장
grpcurl -plaintext -protoset-out dev.protoset localhost:50051 describe
```

### 14.2.4 grpc_cli — 공식 C++ CLI

`grpc_cli`는 gRPC 본체(C++)에 포함된 도구다. `grpcurl`(Go 생태계)과 기능은 겹치지만, C-core 빌드 환경이나 공식 디스트리뷰션을 쓸 때 만난다.

```bash
# 서비스 목록
grpc_cli ls localhost:50051

# 메서드 상세
grpc_cli ls localhost:50051 shop.v1.OrderService -l

# 호출 (입력은 protobuf 텍스트 포맷! JSON 아님에 주의)
grpc_cli call localhost:50051 GetOrder "order_id: 'ord-42'"
```

가장 큰 함정은 입력 포맷이다. `grpcurl`이 **JSON**을 받는 반면, `grpc_cli`는 protobuf **텍스트 포맷**(text format, `field: value` 형태)을 받는다. 둘을 섞으면 파싱 에러가 난다. 또 `grpc_cli`는 리플렉션 의존도가 높아, 리플렉션이 꺼진 서버에는 `--proto_path`/`--protofiles`로 소스를 줘야 한다.

### 14.2.5 grpcui — 브라우저에서 클릭으로 호출

`grpcui`는 `grpcurl`과 같은 저자가 만든 웹 UI다. 로컬에서 작은 웹서버를 띄우고 리플렉션(또는 `-protoset`)으로 얻은 스키마를 **HTML 폼**으로 렌더링한다. 메서드를 드롭다운에서 고르고, 필드를 폼에 채운 뒤 버튼을 누르면 호출되고 응답이 표시된다.

```bash
grpcui -plaintext localhost:50051
# Starting gRPC Web UI on http://127.0.0.1:port
```

`.proto`를 읽을 줄 모르는 QA·기획자에게 "이 메서드 한 번 눌러봐 달라"고 부탁할 때, 또는 enum/oneof/중첩 메시지가 많아 JSON을 손으로 치기 번거로울 때 빛난다. 내부적으로는 브라우저↔grpcui 사이는 일반 HTTP, grpcui↔서버 사이는 진짜 gRPC다(그래서 grpcui가 일종의 프록시 역할).

### 14.2.6 evans — 대화형 REPL

`evans`는 셸처럼 동작하는 대화형 도구다. 서버에 붙어 패키지·서비스 사이를 `cd`처럼 돌아다니고 탭 자동완성으로 메서드를 찾으며, `call`로 호출하면 필드를 **하나씩 대화형으로 입력**받는다.

```bash
evans --host localhost --port 50051 -r repl   # -r = 리플렉션 사용

# REPL 안에서:
shop.v1@localhost:50051> show service
shop.v1@localhost:50051> service OrderService
shop.v1.OrderService@localhost:50051> call GetOrder
order_id (TYPE_STRING) => ord-42
# {
#   "orderId": "ord-42",
#   "status": "PAID"
# }
```

탐색적(exploratory) 디버깅, 즉 "이 서버에 뭐가 있는지 모르겠고 둘러보고 싶다"에 가장 잘 맞는다. JSON을 한 줄로 칠 필요 없이 필드를 하나씩 물어봐 주는 게 입력 실수를 줄여준다.

### 14.2.7 Postman의 gRPC 모드

GUI를 선호하는 팀, 특히 REST 컬렉션을 이미 Postman으로 관리하는 팀은 Postman의 gRPC 기능을 쓴다. 서버 URL을 넣고 리플렉션을 켜거나 `.proto`를 import하면 메서드 목록이 뜨고 메시지·메타데이터를 폼/JSON으로 채워 호출한다. 스트리밍 메서드도 메시지를 순차로 보내며 응답을 실시간으로 본다. 환경변수·컬렉션 공유 같은 협업 기능이 강점이다. 본질은 grpcui와 같다: 리플렉션/디스크립터 → 폼 → 직렬화 → 호출.

### 14.2.8 도구 선택 요약

```text
일회성/스크립트/CI            → grpcurl
리플렉션 꺼진 prod 안전 호출   → grpcurl -protoset (CI가 발행)
폼으로 클릭 호출/비개발자      → grpcui 또는 Postman
서버 둘러보며 탐색            → evans
공식 C++ 디스트리뷰션 환경    → grpc_cli
```

---

## 14.3 channelz — 런타임 내부 상태를 들여다보는 창

### 14.3.1 무엇을, 왜

리플렉션이 "스키마"를 노출한다면, **channelz**는 "런타임 상태"를 노출한다. 이것도 똑같이 gRPC 서비스(`grpc.channelz.v1.Channelz`)로 구현되어, 다음 질문에 답한다.

- 지금 이 프로세스가 가진 **채널(channel)** 과 **서브채널(subchannel)** 은 몇 개이고 각각 상태는 무엇인가? (`READY`/`CONNECTING`/`TRANSIENT_FAILURE`/`IDLE` 등 [[07 - 채널 스텁 커넥션 생명주기]])
- 각 채널이 시작/성공/실패시킨 **호출 수**는?
- 실제 TCP **소켓(socket)** 수준에서 송수신한 **바이트·프레임·스트림 수**, 로컬/원격 주소, keepalive 카운터는?
- 서버 쪽에서 현재 **수신 중인 스트림**은 몇 개인가?

이것이 왜 중요한가? 분산 시스템에서 "연결이 이상하다"는 증상은 코드만 봐서는 안 보인다. 로드밸런서가 죽은 백엔드로 계속 붙는지, 서브채널이 `TRANSIENT_FAILURE`에서 못 빠져나오는지, keepalive ping이 나가고는 있는지 — 이런 건 **프로세스 내부의 살아있는 상태**다. channelz는 이 상태를 재시작·코드수정 없이 라이브로 떠볼 수 있게 한다. `netstat`이 OS의 소켓을 보여준다면, channelz는 gRPC 라이브러리의 추상화(채널/서브채널/소켓)를 그 의미 그대로 보여준다.

### 14.3.2 객체 계층

```text
Server ──────────────┐
  │ (수신)            │
  ├── ServerSocket(s)  ← 클라가 붙은 소켓들, 스트림/바이트 카운터
  │
Channel  (논리적 연결 = 하나의 타깃 "dns:///orders.svc:50051")
  │
  ├── Subchannel  (해석된 개별 백엔드 주소 하나, 예: 10.0.0.7:50051)
  │     └── Socket  (실제 TCP/TLS 소켓, 프레임·바이트·keepalive 카운터)
  ├── Subchannel  (10.0.0.8:50051)
  │     └── Socket
  └── ...
```

하나의 `Channel`이 여러 `Subchannel`을 두는 것은 [[12 - 이름 해석과 로드밸런싱]]의 핵심이다. `round_robin` 정책이면 DNS가 돌려준 여러 IP마다 서브채널이 생기고 channelz로 보면 "어느 백엔드가 READY이고 어느 것이 죽었는지"가 한눈에 보인다.

### 14.3.3 켜기와 보기

서버에 channelz 서비스를 등록한다(Go 예).

```go
import "google.golang.org/grpc/channelz/service"

s := grpc.NewServer()
service.RegisterChannelzServiceToServer(s) // grpc.channelz.v1.Channelz 등록
```

조회는 두 가지 길이 있다. 하나는 `grpcdebug`라는 전용 CLI, 다른 하나는 `grpcurl`로 직접 channelz 메서드를 호출하는 것이다.

```bash
# grpcdebug: 사람이 읽기 좋은 요약
grpcdebug localhost:50051 channelz channels
grpcdebug localhost:50051 channelz channel 1   # 1번 채널 상세
grpcdebug localhost:50051 channelz sockets

# grpcurl로 직접 (리플렉션으로 channelz 스키마를 얻어서)
grpcurl -plaintext localhost:50051 grpc.channelz.v1.Channelz.GetTopChannels
grpcurl -plaintext -d '{"channel_id":"1"}' \
    localhost:50051 grpc.channelz.v1.Channelz.GetChannel
```

전형적인 channelz 출력(개념적 형태)은 이렇다.

```text
Channel ID:        1
Target:            dns:///orders.svc:50051
State:             TRANSIENT_FAILURE        ← 연결 실패 중
Calls Started:     1043
Calls Succeeded:   1001
Calls Failed:      42
Subchannels:
  [ID 2] 10.0.0.7:50051  READY              ← 정상
  [ID 3] 10.0.0.8:50051  TRANSIENT_FAILURE  ← 이 백엔드가 문제!
```

이 한 화면이 "전체 채널은 살아있지만 특정 백엔드(10.0.0.8)가 죽어서 일부 호출이 실패한다"는 결론을 코드 한 줄 안 보고 내리게 해준다. Retry/Health check([[13 - 안정성 - Retry Health Check Keepalive]])가 이 죽은 서브채널을 어떻게 처리하는지와 직접 연결된다.

> 보안 주의: channelz도 내부 상태(원격 주소, 트래픽 양)를 노출하므로 리플렉션과 마찬가지로 프로덕션 외부 노출은 신중해야 한다. 보통 별도 디버그 포트나 내부망 한정으로 연다.

---

## 14.4 내장 로깅·트레이싱 토글 — 코드 0줄로 켜는 X-ray

도구로도 안 잡히는 깊은 문제(핸드셰이크 실패, HPACK 오류, keepalive 타이밍)는 라이브러리 내부 로그를 켜서 본다. 이건 가장 저수준의, 그러나 가장 강력한 무료 도구다.

### 14.4.1 C-core 계열: GRPC_VERBOSITY와 GRPC_TRACE

C++, Python, Ruby, PHP, (구버전) Node.js 등은 공통 C 코어(C-core)를 공유한다. 이들은 두 환경변수로 내부 로그를 토글한다. **코드를 한 줄도 안 고치고** 실행 환경에 변수만 세팅하면 된다.

```bash
# 1) 로그 상세도: DEBUG로 올려야 트레이스가 보인다
export GRPC_VERBOSITY=DEBUG     # ERROR(기본) | INFO | DEBUG

# 2) 어떤 서브시스템을 추적할지(쉼표 구분, all 가능)
export GRPC_TRACE=http,call,channel

python my_grpc_client.py
```

`GRPC_TRACE`에 줄 수 있는 대표 트레이서와 그 의미:

| 트레이서 | 무엇을 보여주나 |
|----------|----------------|
| `http` | HTTP/2 프레임 단위 송수신(HEADERS, DATA, SETTINGS, WINDOW_UPDATE, GOAWAY) |
| `http2_stream_state` | 스트림 상태 전이 |
| `call` | RPC 호출의 시작/완료, 상태 코드 |
| `channel` | 채널 상태 전이(IDLE→CONNECTING→READY…) |
| `subchannel` | 서브채널 생성·연결·실패 |
| `connectivity_state` | 연결성 상태 변화 |
| `pick_first` / `round_robin` | LB 정책의 백엔드 선택 결정 |
| `dns_resolver` | DNS 이름 해석 과정 |
| `tcp` | 소켓 read/write 바이트 |
| `secure_endpoint` / `tsi` | TLS 핸드셰이크 세부 |
| `flowctl` | 흐름 제어 윈도우 변화([[15 - 성능과 Flow Control 백프레셔]]) |
| `all` | 전부(매우 시끄러움. 좁힌 뒤 쓸 것) |

`GRPC_TRACE=http`를 켜면 와이어에서 오가는 HTTP/2 프레임이 텍스트로 찍힌다. 예컨대 핸드셰이크 직후 SETTINGS 교환, HEADERS 프레임의 `:path`, GOAWAY가 왜 날아왔는지가 보인다. "서버가 GOAWAY를 보내서 끊겼다" 같은 진단은 보통 이 트레이스에서 결정된다([[04 - HTTP2 깊이 보기 - 전송 계층]]).

```text
# GRPC_TRACE=http,channel 출력 발췌(개념적):
I0626 ... channel_connectivity: IDLE -> CONNECTING
I0626 ... http2: [c0] OUTGOING SETTINGS: MAX_CONCURRENT_STREAMS=100 INITIAL_WINDOW_SIZE=65535
I0626 ... http2: [c0] INCOMING SETTINGS: ...
I0626 ... channel_connectivity: CONNECTING -> READY
I0626 ... http2: [c0][s1] OUTGOING HEADERS: :path=/shop.v1.OrderService/GetOrder
I0626 ... http2: [c0][s1] INCOMING HEADERS: grpc-status=0
```

### 14.4.2 Go: grpclog

Go gRPC는 C-core를 쓰지 않으므로 환경변수가 다르다. Go는 `grpclog` 패키지와 자체 환경변수를 쓴다.

```bash
# Go gRPC 상세 로그
export GRPC_GO_LOG_VERBOSITY_LEVEL=99
export GRPC_GO_LOG_SEVERITY_LEVEL=info

go run ./client
```

코드에서 커스텀 로거를 꽂을 수도 있다.

```go
import "google.golang.org/grpc/grpclog"

func init() {
    // 표준 에러로 info 이상, verbosity 2
    grpclog.SetLoggerV2(grpclog.NewLoggerV2WithVerbosity(
        os.Stderr, os.Stderr, os.Stderr, 2))
}
```

Go에서 보이는 정보는 C-core의 `GRPC_TRACE`만큼 프레임 단위로 세밀하진 않지만, 이름 해석(resolver), LB 정책, 연결성 상태 전이, transport 에러 같은 핵심 이벤트는 충분히 드러난다. 더 깊은 와이어 분석이 필요하면 Go에서도 결국 14.7의 Wireshark로 내려간다.

### 14.4.3 언제 무엇을 켜나 (요령)

- **연결이 안 됨** → `GRPC_TRACE=tcp,secure_endpoint,subchannel,dns_resolver`. 어디까지 가다 끊기는지(DNS? TCP? TLS?)를 좁힌다.
- **잘못된 백엔드로 감** → `pick_first`/`round_robin`/`dns_resolver`. LB가 어떤 주소를 골랐는지 본다.
- **간헐적 끊김** → `http`(GOAWAY/RST_STREAM), `flowctl`(윈도우 고갈).
- **느림** → `flowctl`로 흐름 제어 정체 여부([[15 - 성능과 Flow Control 백프레셔]]).

주의: 이 트레이스들은 매우 시끄럽고 약간의 성능 비용이 있다. 프로덕션에서는 좁은 범위로, 짧게 켜고 끄는 것이 원칙이다.

---

## 14.5 메트릭·트레이싱 통합 — OpenTelemetry로 RPC를 신호화하기

지금까지는 "문제가 생긴 뒤 손으로 들여다보기"였다. 운영에서는 그 전에, 항상, 자동으로 RPC를 측정해야 한다. 이 자리를 채우는 표준이 **OpenTelemetry(OTel)** 다.

### 14.5.1 역사적 맥락: OpenCensus → OpenTelemetry

gRPC에는 처음부터 관측용 확장점이 있었다. 바로 **stats handler**(통계 핸들러)와 인터셉터([[08 - 메타데이터와 인터셉터]])다. 초기에는 Google의 **OpenCensus**가 이 확장점에 붙어 메트릭·트레이스를 수집했다. 이후 OpenCensus와 OpenTracing이 합쳐져 **OpenTelemetry**가 업계 표준이 됐고 오늘날 gRPC 공식 권장은 OTel이다(OpenCensus는 사실상 레거시).

핵심 메커니즘은 동일하다. gRPC는 RPC의 생애주기 이벤트(시작, 헤더 송수신, 메시지 송수신, 종료)를 **stats handler**에 콜백으로 흘려보낸다. OTel instrumentation은 이 콜백을 받아 메트릭을 적산하고 스팬(span)을 만든다. 인터셉터 기반 구현도 있지만, 정확한 바이트 카운트·메시지 카운트는 transport에 더 가까운 stats handler가 더 정밀하다.

```text
       RPC 한 건의 생애
  start → outHeader → outPayload(s) → inHeader → inPayload(s) → end
    │        │            │             │            │           │
    └────────┴────────────┴─────────────┴────────────┴───────────┘
                          ▼ (콜백)
                   gRPC stats.Handler
                          ▼
              OTel: 메트릭 적산 + 스팬 생성/종료
                          ▼
              OTLP exporter → Collector → Prometheus/Jaeger/...
```

### 14.5.2 표준 메트릭 — 무엇을 재는가

OTel의 gRPC 시맨틱 컨벤션은 클라이언트/서버 양쪽에 표준화된 메트릭 이름과 속성(attribute)을 정의한다. 큰 줄기는 세 가지다.

| 범주 | 의미 | 대표 지표 |
|------|------|-----------|
| 호출 수(count) | 시작/완료된 RPC 수 | 메서드별·상태코드별 카운터 |
| 지연(duration) | RPC 소요 시간 | 히스토그램(p50/p90/p99) |
| 메시지 크기(size) | 요청/응답 바이트 | 송수신 바이트 히스토그램 |

핵심 속성으로는 `rpc.system=grpc`, `rpc.service`(예: `shop.v1.OrderService`), `rpc.method`(예: `GetOrder`), `rpc.grpc.status_code`(숫자 상태 코드 [[09 - 에러 모델 - 상태 코드와 Rich Error]])가 붙는다. 그래서 대시보드에서 "`OrderService.GetOrder`의 p99 지연"이나 "`UNAVAILABLE`(코드 14) 비율"을 메서드 단위로 슬라이스할 수 있다.

스트리밍의 경우 한 RPC가 여러 메시지를 주고받으므로, 메시지 단위 카운트와 RPC 단위 지연이 분리되어 측정된다([[05 - 통신의 4가지 방식 - Unary와 Streaming]]).

### 14.5.3 등록 스니펫 (Go OTel)

gRPC-Go는 `google.golang.org/grpc/stats/opentelemetry`로 공식 OTel 통합을 제공한다(인터셉터를 직접 짜는 것보다 권장).

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/stats/opentelemetry"
    "go.opentelemetry.io/otel/sdk/metric"
)

func main() {
    // 1) OTel MeterProvider 구성 (OTLP exporter 등은 생략)
    mp := metric.NewMeterProvider(/* reader/exporter ... */)

    // 2) 서버에 stats handler로 OTel을 등록
    srv := grpc.NewServer(
        opentelemetry.ServerOption(opentelemetry.Options{
            MetricsOptions: opentelemetry.MetricsOptions{MeterProvider: mp},
        }),
    )

    // 3) 클라이언트에도 동일하게 DialOption으로
    cc, _ := grpc.NewClient("dns:///orders.svc:50051",
        opentelemetry.DialOption(opentelemetry.Options{
            MetricsOptions: opentelemetry.MetricsOptions{MeterProvider: mp},
        }),
        grpc.WithTransportCredentials(creds),
    )
    _ = srv; _ = cc
}
```

전통적 인터셉터 방식(여전히 유효, 특히 트레이싱)은 다음과 같다.

```go
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

srv := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
cc, _ := grpc.NewClient(target,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    grpc.WithTransportCredentials(creds),
)
```

**Java**는 `io.grpc:grpc-opentelemetry` 모듈로 `GrpcOpenTelemetry`를 빌더에 붙이고, **Python**은 `opentelemetry-instrumentation-grpc`의 `GrpcInstrumentorServer/Client`로 자동 계측한다. 언어는 달라도 "stats handler/인터셉터에 OTel을 꽂는다"는 골격은 같다.

### 14.5.4 인터셉터 vs stats handler — 어느 층에 붙는가

[[08 - 메타데이터와 인터셉터]]에서 인터셉터를 자세히 다뤘으니 여기선 관측 관점만 짚는다.

- **인터셉터**는 애플리케이션 경계에 있다. 요청/응답 메시지 객체에 접근할 수 있어, "요청의 user_id를 스팬 속성으로 넣기" 같은 도메인 친화 계측에 좋다. 그러나 transport에서 실제로 몇 바이트가 나갔는지는 모른다(직렬화 후 압축 전후 바이트는 더 아래 층 일).
- **stats handler**는 transport 가까이에 있다. 헤더/페이로드 송수신, 압축 후 바이트 수 같은 저수준 사실을 정확히 안다. OTel 표준 메트릭은 그래서 stats handler 기반이 정확하다.

실무에선 둘을 함께 쓴다: 표준 메트릭은 stats handler에, 도메인 태그·커스텀 스팬 속성은 인터셉터에.

---

## 14.6 분산 트레이싱 컨텍스트 전파 — W3C traceparent

### 14.6.1 문제: 트레이스는 서비스 경계에서 끊긴다

마이크로서비스에서 하나의 사용자 요청은 여러 서비스를 타고 흐른다. `Gateway → OrderService → InventoryService → NotificationService`. 분산 트레이싱이 가치 있으려면, 이 모든 호출이 **하나의 트레이스(trace)** 로 엮여 폭포(waterfall) 차트로 보여야 한다. 그러려면 각 서비스가 "지금 이 호출은 어느 트레이스의 일부이고, 내 부모 스팬은 무엇인가"를 알아야 한다.

문제는 이 정보가 프로세스 메모리 안의 컨텍스트(Go의 `context.Context`, Java의 `Context`)에만 있다는 점이다. 네트워크를 건너가면 사라진다. 그래서 **trace 컨텍스트를 요청과 함께 와이어에 실어 보내야** 한다.

### 14.6.2 어디에 싣나: gRPC 메타데이터 = HTTP/2 헤더

gRPC에서 "요청에 부가 정보를 싣는 곳"은 메타데이터(metadata)다([[08 - 메타데이터와 인터셉터]]). 메타데이터는 와이어 위에서 HTTP/2 HEADERS 프레임의 헤더 필드로 직렬화된다([[04 - HTTP2 깊이 보기 - 전송 계층]]). trace 컨텍스트는 결국 HTTP/2 헤더 한두 줄로 실린다. 표준 형식이 **W3C Trace Context**다.

### 14.6.3 traceparent 헤더의 해부

W3C Trace Context는 `traceparent`라는 단일 헤더에 트레이스 식별 정보를 담는다. 형식은 엄격히 고정되어 있다.

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ╰┬╯ ╰──────────────┬──────────────╯ ╰───────┬──────╯ ╰┬╯
           version    trace-id (16바이트=32hex)   parent-id(8B=16hex) flags
```

| 필드 | 길이 | 의미 |
|------|------|------|
| version | 1바이트(2 hex) | 포맷 버전. 현재 `00` |
| trace-id | 16바이트(32 hex) | 전체 트레이스의 고유 ID. 모든 서비스에서 동일하게 유지 |
| parent-id (span-id) | 8바이트(16 hex) | 이 호출을 만든 부모 스팬 ID |
| trace-flags | 1바이트(2 hex) | 비트 플래그. 최하위 비트=sampled(`01`이면 이 트레이스를 샘플링/기록) |

추가로 `tracestate` 헤더가 벤더별 확장 정보를 키-값으로 나른다(예: `tracestate: vendorA=xyz,vendorB=123`).

### 14.6.4 전파의 실제 흐름 (inject/extract)

OTel은 이 전파를 자동화한다. 핵심 동작은 두 가지 동사다: **inject**(나가는 호출의 메타데이터에 trace 컨텍스트를 써넣기)와 **extract**(들어오는 호출의 메타데이터에서 trace 컨텍스트를 읽기).

```text
 Gateway                  OrderService              InventoryService
   │                          │                           │
 새 trace 시작                 │                           │
 trace-id=4bf9...             │                           │
 span A 생성                   │                           │
   │                          │                           │
   │ gRPC 호출(GetOrder)       │                           │
   │  inject ↓ 메타데이터에     │                           │
   │  traceparent:            │                           │
   │  00-4bf9...-AAAA-01 ─────▶│ extract ↑                 │
   │                          │ "trace-id=4bf9, parent=AAAA"
   │                          │ span B 생성(부모=AAAA)      │
   │                          │                           │
   │                          │ gRPC 호출(CheckStock)      │
   │                          │  inject ↓                  │
   │                          │  00-4bf9...-BBBB-01 ──────▶│ extract ↑
   │                          │                           │ span C(부모=BBBB)
   ▼                          ▼                           ▼
  세 서비스의 스팬 A,B,C가 같은 trace-id 4bf9로 묶여 하나의 폭포가 된다
```

trace-id는 처음부터 끝까지 변하지 않는다. 반면 각 호출마다 **부모-자식 관계**를 잇기 위해 parent-id(부모 span-id)는 매 hop에서 갱신된다. 이렇게 해서 Jaeger/Tempo 같은 백엔드가 스팬들을 트리로 재조립한다.

OTel을 stats handler/인터셉터로 등록(14.5)해 두면, inject/extract와 traceparent 직렬화는 **자동**이다. 개발자가 손으로 헤더를 만질 일은 보통 없다. 다만 메커니즘을 알면, "왜 트레이스가 끊겼지?"(= 어딘가 propagator 미설정으로 traceparent가 안 실렸다)를 진단할 수 있다.

직접 헤더를 확인하고 싶다면 grpcurl로 traceparent를 흉내내 보낼 수도 있다.

```bash
grpcurl \
  -H 'traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01' \
  -d '{"order_id":"ord-42"}' \
  -plaintext localhost:50051 shop.v1.OrderService.GetOrder
```

서버가 OTel 추출을 설정해뒀다면, 이 호출은 trace-id `4bf9...`에 묶인 스팬으로 기록된다.

### 14.6.5 W3C 표준을 직접 보는 이유

외부 라이브러리를 무조건 믿지 말고 표준 명세를 먼저 보라는 원칙이 여기서 그대로 적용된다. 트레이스가 끊기거나 헤더가 깨졌을 때, 라이브러리 문서 대신 W3C Trace Context 명세(헤더 문법, version/flags 규칙)를 보면 "traceparent 길이가 한 글자 모자라서 파서가 통째로 버렸다" 같은 근본 원인을 빠르게 짚을 수 있다. 형식이 엄격히 고정되어 있어 hex 길이만 세도 유효성 절반은 판별된다.

---

## 14.7 와이어 레벨 디버깅 — Wireshark로 프레임 보기

도구도, 트레이스도 답을 못 줄 때, 마지막 진실은 와이어에 있다. "라이브러리가 진짜로 무슨 바이트를 보냈는가"를 직접 확인하는 단계다. 이때 Wireshark가 등판한다.

### 14.7.1 Wireshark의 HTTP/2·gRPC 디섹터

Wireshark는 HTTP/2 디섹터(dissector)를 내장하고 그 위에 gRPC 디섹터까지 둔다. 캡처한 패킷을 단순 바이트가 아니라 **HTTP/2 프레임(HEADERS/DATA/SETTINGS/…)으로 파싱**하고 DATA 프레임 안의 gRPC 메시지(길이-접두 프레이밍)까지 인식한다. 여기에 `.proto`나 디스크립터 셋을 등록하면 **protobuf 필드까지 디코딩**해서 사람이 읽을 수 있게 보여준다.

```text
Wireshark가 보여주는 레이어 (한 RPC 요청):
  Frame → Ethernet → IP → TCP
    └ HTTP/2
        ├ Stream: 1
        ├ HEADERS frame
        │    :method: POST
        │    :path: /shop.v1.OrderService/GetOrder
        │    content-type: application/grpc
        │    te: trailers
        ├ DATA frame
        │    gRPC Message (5바이트 헤더 + protobuf)
        │      Compressed-Flag: 0
        │      Message-Length: 8
        │      [protobuf] order_id (1) = "ord-42"   ← .proto 등록 시
        └ (응답 스트림) HEADERS(grpc-status=0 trailer)
```

gRPC 메시지 프레이밍을 기억하자([[05 - 통신의 4가지 방식 - Unary와 Streaming]], [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]). DATA 프레임 페이로드의 각 메시지는 **5바이트 접두**로 시작한다: 1바이트 압축 플래그 + 4바이트 빅엔디언 길이.

```text
gRPC 길이-접두 프레이밍 (Length-Prefixed Message):

  ┌──────┬───────────────────┬─────────────────────────────┐
  │ 0x00 │ 00 00 00 08        │ 0a 06 6f 72 64 2d 34 32      │
  └──────┴───────────────────┴─────────────────────────────┘
   압축X   length = 8 (빅엔디언)   protobuf payload (8바이트)

  길이 접두 8 = 직렬화된 메시지 전체 바이트 수(tag+len+값). 압축 플래그/4바이트
  길이 자신은 이 수에 포함되지 않는다.

  손으로 디코딩한 protobuf 8바이트:
    0a            → tag: 필드번호 1, wire type 2(LEN)   (1<<3)|2 = 0x0a
    06            → length = 6 (이어지는 문자열 길이)
    6f 72 64 2d 34 32 → "ord-42"  (o r d - 4 2)
  ⇒ GetOrderRequest{ order_id: "ord-42" }   (= 1 + 1 + 6 = 8바이트)
```

이 손 디코딩이 가능한 이유가 바로 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 다룬 태그-와이어타입 규칙이다. 압축 플래그가 `0x01`이면 이어지는 바이트는 압축된 것이라 그대로는 못 읽고 압축 알고리즘(gzip 등)을 역으로 풀어야 한다.

### 14.7.2 TLS 복호화: SSLKEYLOGFILE

여기서 결정적 장벽이 있다. 프로덕션 트래픽은 TLS로 암호화되어 있어([[11 - 보안 - TLS mTLS 인증]]), Wireshark로 떠도 TLS Application Data로만 보이고 안의 HTTP/2 프레임은 안 보인다. 이걸 푸는 표준 방법이 **TLS 키 로그**다.

TLS 1.2/1.3은 세션마다 임시 키를 협상한다. 많은 TLS 라이브러리는 환경변수 `SSLKEYLOGFILE`이 설정되면, 협상한 세션 시크릿(마스터 시크릿 또는 TLS 1.3의 traffic secret)을 그 파일에 NSS Key Log Format으로 기록한다. Wireshark에 이 파일을 등록하면, 캡처한 암호문을 **사후에 복호화**해서 평문 HTTP/2를 볼 수 있다.

```bash
# 1) 키 로그를 남기도록 클라이언트(또는 서버) 실행
export SSLKEYLOGFILE=/tmp/grpc-keys.log

# (지원하는 런타임에서) 클라이언트 실행 — 협상된 세션 키가 파일에 누적됨
go run ./client     # 또는 curl, 브라우저 등 NSS keylog 지원 런타임

# 2) Wireshark 설정:
#    Preferences → Protocols → TLS → (Pre)-Master-Secret log filename
#       = /tmp/grpc-keys.log
# 이제 캡처한 TLS 세션이 복호화되어 HTTP/2/gRPC 디섹터가 동작
```

```text
키 로그 파일 한 줄 예 (개념):
  CLIENT_RANDOM <client_random_hex> <master_secret_hex>      ← TLS 1.2
  CLIENT_TRAFFIC_SECRET_0 <client_random> <secret>           ← TLS 1.3
                                ▲ Wireshark가 이걸로 세션을 복호화
```

주의: 이 키 로그는 그 세션의 트래픽을 누구나 복호화할 수 있게 하는 **민감 정보**다. 디버깅 후 반드시 삭제하고, 프로덕션 비밀이 흐르는 세션엔 함부로 쓰지 않는다.

### 14.7.3 평문 h2c 캡처

로컬 개발에서 TLS 없이(`-plaintext`) 도는 서버라면 키 로그조차 필요 없다. 평문 HTTP/2(=h2c, HTTP/2 over cleartext)는 Wireshark가 곧바로 디섹팅한다. 단, gRPC는 보통 사전협상(prior knowledge)으로 h2c를 시작하므로, Wireshark가 포트를 자동으로 HTTP/2로 인식 못 하면 "Decode As… → HTTP2"로 해당 포트를 수동 지정하면 된다.

```bash
# 로컬 평문 gRPC 포트 캡처
sudo tcpdump -i lo0 -w /tmp/grpc.pcap port 50051
# 이후 Wireshark에서 /tmp/grpc.pcap 열고, 필요시 50051 포트를 HTTP2로 Decode As
```

### 14.7.4 언제 와이어까지 내려가나

Wireshark는 비용이 큰 도구다(캡처 설정, 복호화, 프레임 해석). 보통 다음일 때만 내려간다.

- 라이브러리 버그가 의심될 때("우리가 보낸 헤더가 정말 이거 맞나?").
- 다른 모든 층(channelz, GRPC_TRACE)이 모순된 신호를 줄 때.
- 프로토콜 비호환(중간 프록시가 HTTP/2 프레임을 변형, GOAWAY 주입, trailer 제거 등)을 의심할 때.
- 압축/프레이밍 경계 문제처럼 바이트 정합성을 직접 봐야 할 때.

대부분의 문제는 와이어까지 안 가고 위쪽 층에서 끝난다. 그래서 다음 절의 "증상별 플레이북"이 실전에서는 더 자주 쓰인다.

---

## 14.8 증상별 진단 플레이북

운영 디버깅은 직관이 아니라 체크리스트다. 가장 자주 만나는 네 가지 상태 코드와 증상을 두고, "무엇을 어떤 순서로 확인하는가"를 정리한다. 상태 코드 자체의 의미는 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에 있고 여기선 진단 절차에 집중한다.

### 14.8.1 UNAVAILABLE (코드 14) — 가장 흔하고 가장 모호한 에러

`UNAVAILABLE`은 "지금은 못 한다, 재시도하면 될 수도"라는 뜻이라 원인이 광범위하다. 연결 자체가 안 되는 모든 경우가 여기로 수렴한다. 그래서 **아래에서 위로**, 네트워크→TLS→서버 순으로 좁힌다.

```text
UNAVAILABLE 진단 사다리 (아래→위로 한 칸씩 확인)

  ⑤ 서버 애플리케이션이 죽었나/과부하/GOAWAY 보냈나
       └ channelz로 서브채널 상태, GRPC_TRACE=http로 GOAWAY 확인
  ④ TLS 핸드셰이크 실패 (인증서 만료/SNI/CA 불일치)
       └ grpcurl -v 로 핸드셰이크 단계, openssl s_client로 인증서
  ③ 이름 해석 실패 (DNS가 빈 결과/잘못된 IP)
       └ GRPC_TRACE=dns_resolver, nslookup/dig
  ② 방화벽/보안그룹이 포트 차단
       └ nc -vz host port 로 TCP 도달성
  ① 호스트/포트 자체가 틀림
       └ 타깃 문자열, 포트 번호 재확인
```

확인 명령 순서:

```bash
# ① TCP 레벨 도달성부터 (gRPC 이전 문제 배제)
nc -vz orders.svc 50051

# ③ 이름 해석
dig +short orders.svc
GRPC_TRACE=dns_resolver GRPC_VERBOSITY=DEBUG ./client   # C-core

# ④ TLS 핸드셰이크/인증서
openssl s_client -connect orders.svc:443 -servername orders.svc </dev/null \
  | openssl x509 -noout -dates -subject

# ⑤ 서버가 살아있고 메서드가 뜨나 (리플렉션 켜졌으면)
grpcurl -plaintext orders.svc:50051 list

# 라이브 상태: 어느 서브채널이 죽었나
grpcdebug orders.svc:50051 channelz channels
```

가장 흔한 함정: **`-plaintext`/TLS 미스매치**가 UNAVAILABLE 또는 "first record does not look like TLS"로 위장하는 경우(14.2.2). 그리고 클라이언트가 keepalive ping을 너무 공격적으로 보내 서버가 `ENHANCE_YOUR_CALM` GOAWAY로 끊는 경우([[13 - 안정성 - Retry Health Check Keepalive]])도 UNAVAILABLE로 나타난다 — 이건 `GRPC_TRACE=http`로 GOAWAY와 그 사유를 봐야 잡힌다.

### 14.8.2 DEADLINE_EXCEEDED (코드 4) — 시간이 모자랐다

데드라인은 클라이언트가 "이 시간 안에 못 끝내면 포기"라고 건 약속이다([[10 - Deadline 취소 타임아웃]]). 초과 시 이 코드가 난다. 진단의 첫 갈래는 "정말 서버가 느린가, 아니면 데드라인이 너무 짧은가, 아니면 중간 어딘가에서 막혔나"다.

```text
DEADLINE_EXCEEDED 분기

  서버 스팬(트레이싱)이 있나?
   ├─ 있고, 서버 처리 시간이 데드라인보다 길다  → 서버가 느림(다운스트림 DB/RPC 추적)
   ├─ 있고, 서버는 빨리 끝났는데 클라가 초과     → 네트워크 지연/클라측 큐잉/연결 수립 지연
   └─ 서버 스팬이 아예 없다                      → 요청이 서버에 도달 못함(연결/LB/이름해석)
```

확인 포인트:

- **데드라인 전파(propagation)**: A→B→C 호출에서 데드라인은 메타데이터로 전파된다(`grpc-timeout` 헤더). A의 데드라인이 1초인데 B가 C를 0.9초 남기고 호출하면 C는 거의 시간이 없다. 트레이스 폭포에서 각 hop에 남은 시간을 본다([[10 - Deadline 취소 타임아웃]]).
- **느림의 위치**: OTel 지연 히스토그램(14.5)에서 어느 메서드의 p99가 튀는지. 그 메서드의 서버 스팬에서 다운스트림(DB 쿼리, 외부 RPC) 자식 스팬을 본다.
- **재현**: `grpcurl`에 짧은/긴 데드라인을 줘서 경계를 찾는다.

```bash
# grpcurl로 명시적 데드라인을 걸어 재현 (-max-time 초)
grpcurl -max-time 0.5 -d '{"order_id":"ord-42"}' \
    -plaintext localhost:50051 shop.v1.OrderService.GetOrder
# 0.5초로 실패하고 5초로 성공하면 → 서버가 0.5~5초 사이로 느린 것
```

### 14.8.3 UNIMPLEMENTED (코드 12) — 경로가 틀렸다

`UNIMPLEMENTED`는 "서버가 그런 메서드를 모른다"는 뜻이다. gRPC 메서드 경로는 `/<package>.<Service>/<Method>` 형태의 HTTP/2 `:path`로 라우팅된다([[04 - HTTP2 깊이 보기 - 전송 계층]]). 이 경로가 한 글자라도 어긋나면 UNIMPLEMENTED다. 원인은 거의 항상 **불일치(mismatch)**다.

| 원인 | 구체 예 |
|------|---------|
| 서비스에 메서드 미등록 | 서버 코드에서 `RegisterXxxServer`를 빠뜨림 |
| 패키지/서비스 이름 오타·버전 차이 | 클라 `shop.v1.OrderService` vs 서버 `shop.v2.OrderService` |
| 프록시가 다른 백엔드로 라우팅 | 게이트웨이가 엉뚱한 upstream으로 보냄 |
| 리플렉션 자체가 UNIMPLEMENTED | 서버가 리플렉션 서비스를 등록 안 함(도구가 "Server does not support reflection") |

진단의 결정타는 리플렉션이다. 서버가 실제로 어떤 경로를 들고 있는지 직접 물어본다.

```bash
# 서버가 실제로 가진 서비스/메서드 경로 확인
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext localhost:50051 list shop.v1.OrderService
# 여기 안 보이는 경로를 호출하면 UNIMPLEMENTED가 정상
```

만약 `list`조차 "server does not support the reflection API"라면, 그건 리플렉션 미등록이지 우리 메서드의 문제가 아니다. 이땐 14.2.3처럼 `-protoset`으로 스키마를 직접 줘서 호출해 본다. 그래도 UNIMPLEMENTED면 진짜로 그 메서드가 그 서버에 없다(버전/배포 불일치 의심). 버전 스큐가 잦다면 [[17 - 실전 - proto 관리와 하위호환성]]의 스키마 관리 규율이 근본 처방이다.

### 14.8.4 헤더/Trailer 누락 — gRPC만의 함정

gRPC는 응답 상태를 **trailer**(후행 헤더)로 보낸다. 정상 응답조차 `grpc-status: 0`을 HTTP/2 trailers로 전달한다([[09 - 에러 모델 - 상태 코드와 Rich Error]], [[04 - HTTP2 깊이 보기 - 전송 계층]]). 이 구조 때문에 **HTTP/2 trailer를 제대로 다루지 못하는 중간 장비**가 끼면 gRPC가 깨진다.

대표 증상과 원인:

```text
증상                                   의심 원인
─────────────────────────────────────  ──────────────────────────────
"missing grpc-status in trailer"       프록시/LB가 HTTP/2 trailer를 떨굼
"unexpected HTTP status code 404/502"  요청이 HTTP/1.1 처리 경로로 잘못 감
                                       (gRPC는 HTTP/2 전제; 다운그레이드 금물)
content-type이 application/grpc 아님    응답을 보낸 게 gRPC 서버가 아님
                                       (에러 페이지, 프록시 응답 등)
te: trailers 누락으로 거부              일부 서버는 이 헤더 없으면 거절
```

이 부류는 거의 항상 **L7 프록시/로드밸런서/서비스 메시 설정** 문제다. HTTP/1.1로 다운그레이드하거나 trailer를 보존하지 않는 장비가 경로에 있는지 본다. 결정적 진단법은 둘이다. 14.7의 Wireshark로 "trailer가 와이어에 실제로 실렸는지"를 직접 확인하거나, `GRPC_TRACE=http`로 HEADERS(EOS) 프레임을 보면 된다. gRPC-Web/REST 게이트웨이를 통과할 때의 trailer 변환은 [[16 - gRPC-Web과 REST 게이트웨이]]에서 더 자세히 다룬다.

```bash
# trailer까지 포함한 응답 상세를 보려면 -v
grpcurl -v -plaintext localhost:50051 shop.v1.OrderService.GetOrder \
    -d '{"order_id":"ord-42"}'
# 출력에 Response trailers received: grpc-status, grpc-message 가 보여야 정상
```

### 14.8.5 진단 도구 빠른 매핑

```text
증상                  1차 도구              2차(깊이)
───────────────────  ───────────────────  ──────────────────────
UNAVAILABLE          nc/dig, grpcurl list  channelz, GRPC_TRACE=http(GOAWAY)
DEADLINE_EXCEEDED    OTel 지연 히스토그램    트레이스 폭포(hop별 남은 시간)
UNIMPLEMENTED        grpcurl list/describe  -protoset로 직접 호출, 배포 버전 확인
trailer 누락         grpcurl -v             Wireshark(프레임), GRPC_TRACE=http
느림/정체            OTel 지연 + flowctl     Wireshark WINDOW_UPDATE([[15 - 성능과 Flow Control 백프레셔]])
잘못된 백엔드          channelz 서브채널       GRPC_TRACE=round_robin/dns_resolver
```

---

## 14.9 종합 예시 — 리플렉션 켜진 서버를 끝에서 끝까지 디버깅

지금까지의 층을 하나의 시나리오로 엮어 보자. 가상의 `shop.v1.OrderService`가 dev에서 평문으로 돌고 리플렉션과 channelz, OTel이 모두 켜져 있다고 하자.

**서버 (Go, 관측 3종 세트 등록)**

```go
package main

import (
    "net"
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
    "google.golang.org/grpc/channelz/service"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
    shoppb "example.com/shop/v1"
)

func main() {
    srv := grpc.NewServer(
        grpc.StatsHandler(otelgrpc.NewServerHandler()), // 14.5 메트릭+트레이싱
    )
    shoppb.RegisterOrderServiceServer(srv, &orderServer{})

    reflection.Register(srv)                               // 14.1 리플렉션
    service.RegisterChannelzServiceToServer(srv)           // 14.3 channelz

    lis, _ := net.Listen("tcp", ":50051")
    srv.Serve(lis)
}
```

**1단계 — 발견(리플렉션)**

```bash
$ grpcurl -plaintext localhost:50051 list
grpc.channelz.v1.Channelz
grpc.reflection.v1.ServerReflection
shop.v1.OrderService

$ grpcurl -plaintext localhost:50051 describe shop.v1.OrderService.GetOrder
shop.v1.OrderService.GetOrder is a method:
rpc GetOrder ( .shop.v1.GetOrderRequest ) returns ( .shop.v1.Order );
```

**2단계 — 호출(애드혹)**

```bash
$ grpcurl -plaintext -d '{"order_id":"ord-42"}' \
    localhost:50051 shop.v1.OrderService.GetOrder
{
  "orderId": "ord-42",
  "status": "PAID"
}
```

**3단계 — 실패 재현과 진단**. 존재하지 않는 메서드를 부르면:

```bash
$ grpcurl -plaintext -d '{}' localhost:50051 shop.v1.OrderService.GetOrderV2
ERROR:
  Code: Unimplemented
  Message: unknown method GetOrderV2 ...
# → 14.8.3: list로 실제 메서드 확인하니 GetOrder만 있음. 클라가 잘못된 경로 사용.
```

데드라인을 짧게 걸어 느린 의존을 노출:

```bash
$ grpcurl -max-time 0.2 -plaintext -d '{"order_id":"slow"}' \
    localhost:50051 shop.v1.OrderService.GetOrder
ERROR:
  Code: DeadlineExceeded
# → 14.8.2: OTel 트레이스 폭포에서 GetOrder의 자식 스팬(재고 조회 RPC)이 0.18s 소요됨을 확인.
```

**4단계 — 연결 상태(channelz)**. 클라이언트가 업스트림으로 여러 백엔드를 가질 때:

```bash
$ grpcdebug localhost:50051 channelz channels
# Channel(1) dns:///inventory.svc:50051  TRANSIENT_FAILURE
#   Subchannel(2) 10.0.0.8:50051  TRANSIENT_FAILURE  ← 죽은 백엔드 특정
```

**5단계 — 와이어 확인(필요 시)**. trailer 누락이 의심되면:

```bash
$ grpcurl -v -plaintext -d '{"order_id":"ord-42"}' \
    localhost:50051 shop.v1.OrderService.GetOrder
# Response trailers received:
#   grpc-status: 0
#   grpc-message:
# → trailer 정상. 프록시 문제 아님.
```

이 흐름이 보여주듯, 다섯 개 층은 별개가 아니라 **점점 깊이 들어가는 하나의 깔때기**다. 대부분의 문제는 1~2단계(리플렉션·grpcurl)에서 끝나고 연결이 의심되면 channelz, 라이브러리/프로토콜이 의심되면 와이어로 내려간다.

---

## 14.10 관찰성 설계 체크리스트

마지막으로, 서비스를 만들 때 관찰성을 어떻게 끼워 넣을지 체크리스트로 정리한다.

| 항목 | 권장 | 근거 절 |
|------|------|---------|
| 리플렉션 | dev/stg 켬, 외부 prod 끔(+`-protoset` 배포) | 14.1.6 |
| channelz | 디버그 포트/내부망에 켬 | 14.3 |
| OTel stats handler | 클라·서버 모두 기본 등록 | 14.5 |
| 표준 메트릭 | 메서드·상태코드별 지연/카운트/바이트 대시보드 | 14.5.2 |
| 트레이스 전파 | W3C propagator 전역 설정(끊김 방지) | 14.6 |
| 인터셉터 태깅 | request-id, user-id 등 도메인 속성 | 14.5.4 |
| 트레이스 토글 | GRPC_TRACE 사용법 런북에 명시(짧게 켜기) | 14.4 |
| 키 로그 절차 | SSLKEYLOGFILE 사용·삭제 절차 문서화 | 14.7.2 |
| 증상 런북 | UNAVAILABLE/DEADLINE/UNIMPLEMENTED 플레이북 비치 | 14.8 |

핵심 철학은 하나다. **관찰성은 나중에 끼워 넣는 게 아니라, 서비스를 세울 때 transport에 함께 박아 둔다.** 리플렉션·channelz·OTel은 모두 "이미 프로세스 안에 있는 사실(스키마·연결상태·RPC 이벤트)"을 노출하는 얇은 어댑터일 뿐이라 비용이 작고 그 대가로 사고 한 번을 몇 시간 단축한다.

---

## 핵심 요약

- **gRPC는 바이너리라 `curl`이 안 통한다.** 대신 다섯 겹의 관찰 레이어가 있다: 리플렉션(스키마)→대화형 도구→channelz(런타임 상태)→OTel(신호)→Wireshark(와이어). 아래로 갈수록 깊고 위로 갈수록 일상적이다.
- **서버 리플렉션**(`grpc.reflection.v1`)은 컴파일 타임에 박힌 `FileDescriptorProto`를 양방향 스트림으로 노출해, 클라가 `.proto` 없이 서비스·메서드·메시지를 질의하게 한다. 새 정보를 만드는 게 아니라 이미 있던 스키마를 별도 채널로 끌어올 뿐이다.
- 리플렉션은 **API 표면을 광고**하므로 외부 prod에선 끄고 대신 `protoc --include_imports`로 만든 `.protoset`을 `grpcurl -protoset`에 먹이는 패턴을 쓴다.
- **grpcurl/grpcui/evans/grpc_cli/Postman**은 모두 "스키마 획득→JSON↔protobuf 변환→호출"이라는 골격을 공유한다. `-plaintext`(TLS 끔)와 `-insecure`(검증만 끔)를 혼동하지 말 것.
- **channelz**는 채널/서브채널/소켓의 런타임 상태(연결성, 호출 수, 바이트)를 라이브로 노출해, 코드 수정 없이 "어느 백엔드가 죽었나"를 짚게 한다.
- **GRPC_VERBOSITY/GRPC_TRACE**(C-core)와 **grpclog**(Go)는 코드 0줄로 HTTP/2 프레임·연결성·LB 결정·TLS 핸드셰이크 내부 로그를 켠다.
- **OpenTelemetry**는 gRPC의 stats handler/인터셉터에 붙어 메서드·상태코드별 지연·카운트·바이트를 표준 메트릭으로 수집하고 **W3C traceparent**를 메타데이터로 inject/extract해 서비스 경계를 넘어 트레이스를 잇는다.
- **Wireshark + SSLKEYLOGFILE**은 최후의 진실이다. gRPC 디섹터가 HTTP/2 프레임과 길이-접두(5바이트) gRPC 메시지를 파싱하고 키 로그로 TLS를 복호화해 protobuf 바이트까지 손으로 디코딩할 수 있다.
- **증상별 플레이북**: UNAVAILABLE은 네트워크→DNS→TLS→서버 순으로 사다리 진단, DEADLINE_EXCEEDED는 서버 스팬 유무로 분기, UNIMPLEMENTED는 리플렉션 `list`로 실제 경로 대조, trailer 누락은 L7 프록시의 HTTP/2 trailer 보존 문제를 의심한다.

---

## 연결 노트

- [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]] — FileDescriptor의 protobuf 표현, 와이어 태그/길이-접두 디코딩의 기반
- [[04 - HTTP2 깊이 보기 - 전송 계층]] — 메타데이터=HTTP/2 헤더, trailer로 전달되는 grpc-status, GOAWAY/프레임
- [[05 - 통신의 4가지 방식 - Unary와 Streaming]] — grpcurl 스트리밍 호출, 메시지 프레이밍
- [[06 - 코드 생성과 protoc 툴체인]] — descriptor set 생성(`--include_imports`), 디스크립터가 코드에 박히는 원리
- [[07 - 채널 스텁 커넥션 생명주기]] — channelz가 보여주는 채널/서브채널 상태(READY/CONNECTING/…)
- [[08 - 메타데이터와 인터셉터]] — 인증 헤더(-H), 트레이스 전파 지점, stats handler vs 인터셉터
- [[09 - 에러 모델 - 상태 코드와 Rich Error]] — UNAVAILABLE/DEADLINE_EXCEEDED/UNIMPLEMENTED 상태 코드의 의미
- [[10 - Deadline 취소 타임아웃]] — 데드라인 전파, DEADLINE_EXCEEDED 진단
- [[11 - 보안 - TLS mTLS 인증]] — `-plaintext`/`-insecure`/mTLS, SSLKEYLOGFILE로 TLS 복호화
- [[12 - 이름 해석과 로드밸런싱]] — channelz 서브채널과 LB 정책(round_robin), DNS resolver 트레이스
- [[13 - 안정성 - Retry Health Check Keepalive]] — keepalive ENHANCE_YOUR_CALM, health check와 죽은 서브채널
- [[15 - 성능과 Flow Control 백프레셔]] — flowctl 트레이스, WINDOW_UPDATE로 보는 흐름 제어 정체
- [[16 - gRPC-Web과 REST 게이트웨이]] — 게이트웨이에서의 trailer 변환과 디버깅
- [[17 - 실전 - proto 관리와 하위호환성]] — 버전 스큐로 인한 UNIMPLEMENTED의 근본 처방, descriptor set 운영
