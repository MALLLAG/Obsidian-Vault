---
title: 코드 생성과 protoc 툴체인
date: 2026-06-26
tags:
  - grpc
  - protoc
  - codegen
  - descriptor
  - plugin
  - protoc-gen-go
  - buf
  - bsr
  - bazel
  - gradle
  - toolchain
  - 학습노트
---

## 0. 들어가며: "코드 생성"이라는 단어가 가리는 것

[[02 - Protocol Buffers 1 - 문법과 타입 시스템]]에서는 `.proto`라는 계약서를 작성하는 방법을 배웠고, [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서는 그 계약에 따라 직렬화된 바이트가 어떤 모양인지 손으로 디코딩해 보았다.

그런데 그 사이의 한 단계는 "그렇게 된다"고만 하고 넘어갔다. **`.proto` 파일 하나에서 어떻게 Go의 struct, Java의 class, Python의 모듈, 그리고 gRPC 서버와 클라이언트의 골격까지 한꺼번에 만들어지는가?**

많은 사람이 이 단계를 블랙박스로 남겨 둔다. "`protoc`를 실행하면 코드가 나온다"는 설명은 틀리지 않다. 하지만 이 블랙박스를 열어 보지 않으면 다음과 같은 실무 질문에 답할 수 없다.

- 왜 회사마다 `protoc` 명령줄이 길고 쉽게 깨지며, 특정 사람의 로컬에서만 빌드되는 일이 생기는가?
- 생성된 `*.pb.go` 파일을 git에 커밋해야 하는가, 하지 말아야 하는가?
- gRPC reflection([[14 - 관찰성과 디버깅 - Reflection grpcurl]])을 쓰면 클라이언트가 `.proto` 파일을 갖고 있지 않아도 서버의 메서드를 알아낼 수 있는데, 이것은 어떻게 가능한가?
- 프론트엔드(TypeScript)와 백엔드(Go)가 같은 `.proto`를 참조할 때, 버전이 어긋나지 않게 하려면 어떻게 해야 하는가?

이 질문들의 답은 모두 한곳에서 나온다. **`protoc`는 "컴파일러"라기보다 "파서와 플러그인 디스패처"에 가깝다.** `protoc`는 `.proto` 텍스트를 읽어 **언어 중립적인 중간 표현(intermediate representation)** 으로 바꾸는 데까지만 담당하고, 실제 코드 생성은 외부 프로그램(플러그인)에 맡긴다. 그리고 그 중간 표현 자체도 **또 하나의 protobuf 메시지**이다.

이 장에서 따라갈 흐름을 그림으로 먼저 정리해 두자.

```text
   .proto 텍스트                                    생성된 소스 코드
  ┌───────────────┐                              ┌──────────────────┐
  │ syntax=proto3;│                              │ order.pb.go      │
  │ message Order │                              │ order_grpc.pb.go │
  │ service Orders│                              │ Order.java ...   │
  └───────┬───────┘                              └────────▲─────────┘
          │                                               │
          │  (1) 렉싱·파싱·타입검사                        │ (4) 텍스트로 직렬화
          ▼                                               │
  ┌────────────────────────────────────────────┐        │
  │            protoc 프런트엔드                  │        │
  │  .proto → FileDescriptorProto (메모리상)      │        │
  └───────┬──────────────────────────────────────┘       │
          │ (2) CodeGeneratorRequest 로 포장              │
          │     (= FileDescriptorSet + 옵션)              │
          ▼  stdin (직렬화된 바이트)                       │
  ┌────────────────────────────────────────────┐        │
  │     protoc-gen-go / -java / -python ...      │────────┘
  │  (외부 플러그인 프로세스)                       │ (3) CodeGeneratorResponse
  │  Request(프로토) → 소스 텍스트                  │     를 stdout 으로 반환
  └────────────────────────────────────────────┘
```

이 장에서는 네 박스 사이로 무엇이 전달되는지를 끝까지 따라간다. 특히 (1)에서 (2)로 넘어갈 때 텍스트가 어떤 데이터 구조로 바뀌는지, (2)에서 (3)으로 넘어갈 때 protoc와 플러그인이 어떤 규약으로 통신하는지를 중점적으로 살펴본다.

---

## 1. protoc 파이프라인: 컴파일러의 앞단과 뒷단

전통적인 컴파일러를 떠올려 보자. C 컴파일러는 대략 `소스 → 토큰(렉싱) → AST(파싱) → 의미 분석 → 중간 표현(IR) → 최적화 → 기계어`의 단계를 거친다. 앞쪽 절반, 즉 소스를 이해하는 부분을 **프런트엔드**라고 하고, 뒤쪽 절반, 즉 타깃 코드를 만드는 부분을 **백엔드**라고 한다.

LLVM이 큰 영향을 준 이유는 둘 사이를 **잘 정의된 IR**로 나누어, 프런트엔드(C, Rust, Swift...)와 백엔드(x86, ARM, WASM...)를 자유롭게 조합할 수 있게 했기 때문이다.

`protoc`도 같은 설계 원칙을 따른다. 다만 IR이 LLVM IR이 아니라 **`FileDescriptorProto`라는 protobuf 메시지**라는 점이 다르다.

```text
 [protoc 프런트엔드 — protoc 본체가 직접 한다]
   ┌──────────────────────────────────────────────────────────┐
   │ 1. 렉싱(lexing):  "message Order {" → 토큰 스트림           │
   │ 2. 파싱(parsing): 토큰 → 구문 트리                          │
   │ 3. 이름 해석:    import 따라가며 타입 심볼 테이블 구축       │
   │ 4. 타입 검사:    필드 번호 중복? 알 수 없는 타입? 옵션 정합? │
   │ 5. 정규화:       문법 설탕 제거, 기본값/패킹 규칙 확정       │
   │              ↓                                              │
   │        FileDescriptorProto (언어 중립 IR)                  │
   └──────────────────────────────────────────────────────────┘
                          │
                          │  여기가 프런트엔드/백엔드 경계
                          ▼
 [protoc 백엔드 — 외부 플러그인이 한다]
   ┌──────────────────────────────────────────────────────────┐
   │ 6. IR 을 받아 타깃 언어 소스 텍스트로 변환                   │
   │    protoc-gen-go, protoc-gen-java, protoc-gen-python ...    │
   └──────────────────────────────────────────────────────────┘
```

여기서 가장 중요한 점은 **protoc가 어떤 언어를 "몰라도" 그 언어의 코드를 생성할 수 있다**는 것이다. 물론 역사적인 이유로 protoc 바이너리에는 몇몇 언어의 생성기가 내장되어 있다. C++, Java, Python, C#, Kotlin, Objective-C, PHP, Ruby가 그렇다(protobuf를 가장 먼저 도입했고 구글 내부에서 많이 쓰던 언어들이다). 하지만 이 내장은 편의를 위한 것이지 구조상 필요한 것은 아니다.

Go, Dart, Rust, TypeScript처럼 나중에 합류한 언어들은 protoc를 한 줄도 고치지 않고 외부 플러그인만으로 지원된다. 어떤 언어의 생성기가 내장이냐 외부냐는 배포와 성능 편의의 문제일 뿐이고, protoc의 아키텍처는 모든 언어를 외부 플러그인으로 다룰 수 있도록 설계되어 있다. protoc는 "여기 검증된 IR이 있으니 각자 코드로 바꾸라"며 IR을 넘겨줄 뿐이다.

이 구조가 주는 이점은 LLVM이 준 이점과 같다.

1. **언어를 자유롭게 확장할 수 있다**: 새 언어를 지원하려고 protoc를 고칠 필요가 전혀 없다. `FileDescriptorProto`를 읽어 코드를 출력하는 플러그인을 하나 새로 만들면 된다. 실제로 protobuf 생태계에는 공식 언어 외에도 TypeScript, Rust, Dart, Swift, Kotlin, C# 등을 위한 서드파티 플러그인이 수십 개 있다. "문서를 생성하는 플러그인", "JSON Schema를 생성하는 플러그인", "gRPC 게이트웨이([[16 - gRPC-Web과 REST 게이트웨이]])를 생성하는 플러그인"도 같은 메커니즘으로 동작한다.
2. **파싱 로직이 한곳에만 있다**: 모든 언어가 protoc의 같은 파서와 타입 검사기를 공유한다. 그래서 "Go에서는 통과하는데 Java에서는 에러가 난다" 같은 일이 (적어도 파싱 단계에서는) 거의 생기지 않는다. IR을 만들기 전에 의미 검증이 끝나 있기 때문이다.

> 비유하자면 protoc는 **번역가들에게 회의록을 나눠 주는 속기사**이다. 속기사는 회의 내용을 정확한 표준 회의록(IR)으로 기록하는 데까지만 책임진다. 그 회의록을 영어로 옮길지, 일본어로 옮길지는 각 번역가(플러그인)가 정한다. 속기사는 영어도 일본어도 몰라도 된다.

---

## 2. descriptor.proto: "프로토를 표현하는 프로토"

이제 가장 헷갈리는 부분이다. protoc의 IR인 `FileDescriptorProto`는 **그 자체가 protobuf 메시지로 정의되어 있다.** 다시 말해 "`.proto` 파일의 구조를 기술하는 데이터 모델"이 또 다른 `.proto` 파일(`descriptor.proto`)로 작성되어 있다.

이것은 프로그래밍 언어에서 말하는 "메타순환 평가기(metacircular evaluator)"나 "리플렉션(reflection)"과 같은 종류의 자기 참조이다. Java의 `Class` 클래스가 그 자체로 Java 클래스이고, Python의 `type`이 그 자체로 객체인 것과 같은 구조이다.

`descriptor.proto`(protobuf 공식 저장소의 `google/protobuf/descriptor.proto`)의 핵심 골격을 실제 정의에 맞게 간추려 보면 다음과 같다.

```proto
syntax = "proto2";  // descriptor.proto 자체는 역사적으로 proto2다
package google.protobuf;

// 하나의 .proto 파일 전체를 표현한다
message FileDescriptorProto {
  optional string name    = 1;   // 파일 경로, 예: "order/v1/order.proto"
  optional string package = 2;   // package 선언, 예: "order.v1"

  repeated string dependency = 3;        // import 한 파일들의 name
  repeated int32  public_dependency = 10; // 그중 public import 인덱스

  repeated DescriptorProto      message_type = 4; // 이 파일의 message 들
  repeated EnumDescriptorProto  enum_type    = 5; // 이 파일의 enum 들
  repeated ServiceDescriptorProto service    = 6; // 이 파일의 service 들
  repeated FieldDescriptorProto extension     = 7; // 최상위 extension

  optional FileOptions options = 8;       // go_package 등 파일 옵션
  optional SourceCodeInfo source_code_info = 9; // 주석·위치 정보
  optional string syntax = 12;            // "proto2" / "proto3" / "editions"
  optional Edition edition = 14;          // 2023+ Editions (enum; 과거 13번/string은 폐기)
}

// 하나의 message 를 표현한다 (재귀적: 중첩 메시지 가능)
message DescriptorProto {
  optional string name = 1;                    // 메시지 이름
  repeated FieldDescriptorProto field = 2;      // 필드들
  repeated DescriptorProto nested_type = 3;     // 중첩 메시지 (재귀!)
  repeated EnumDescriptorProto enum_type = 4;   // 중첩 enum
  repeated OneofDescriptorProto oneof_decl = 8; // oneof 묶음
  repeated ReservedRange reserved_range = 9;    // reserved 번호 범위
  repeated string reserved_name = 10;           // reserved 이름
}

// 하나의 field 를 표현한다
message FieldDescriptorProto {
  optional string name   = 1;        // 필드 이름, 예: "order_id"
  optional int32  number = 3;        // 필드 번호, 예: 1
  optional Label  label  = 4;        // OPTIONAL / REQUIRED / REPEATED
  optional Type   type   = 5;        // TYPE_STRING, TYPE_INT64, TYPE_MESSAGE...
  optional string type_name = 6;     // 메시지/enum 타입이면 그 풀네임
  optional int32  oneof_index = 9;   // 어느 oneof 에 속하는지
  optional bool   proto3_optional = 17;

  enum Type {
    TYPE_DOUBLE = 1; TYPE_FLOAT = 2; TYPE_INT64 = 3; TYPE_UINT64 = 4;
    TYPE_INT32 = 5;  TYPE_BOOL = 8;  TYPE_STRING = 9; TYPE_MESSAGE = 11;
    TYPE_BYTES = 12; TYPE_ENUM = 14; TYPE_SINT32 = 17; TYPE_SINT64 = 18;
    // ... (전체 18종)
  }
  enum Label { LABEL_OPTIONAL = 1; LABEL_REQUIRED = 2; LABEL_REPEATED = 3; }
}

// 하나의 service 를 표현한다
message ServiceDescriptorProto {
  optional string name = 1;                 // 서비스 이름, 예: "OrderService"
  repeated MethodDescriptorProto method = 2; // rpc 들
}

// 하나의 rpc 메서드를 표현한다
message MethodDescriptorProto {
  optional string name = 1;             // 메서드 이름, 예: "GetOrder"
  optional string input_type  = 2;      // 요청 타입 풀네임, ".order.v1.GetOrderRequest"
  optional string output_type = 3;      // 응답 타입 풀네임
  optional bool client_streaming = 5;   // stream Req 인가?
  optional bool server_streaming = 6;   // stream Res 인가?
}

// 여러 FileDescriptorProto 를 한 봉투에 담은 것
message FileDescriptorSet {
  repeated FileDescriptorProto file = 1;
}
```

이 정의를 차근차근 살펴보면, 2장(Protocol Buffers 문법과 타입 시스템)에서 배운 `.proto` 문법의 **모든 요소가 빠짐없이 데이터 필드로 옮겨져** 있음을 알 수 있다.

`message`는 `DescriptorProto`, 필드는 `FieldDescriptorProto`, `service`는 `ServiceDescriptorProto`, `rpc`는 `MethodDescriptorProto`가 된다. [[05 - 통신의 4가지 방식 - Unary와 Streaming]]에서 강조했던 **`stream` 키워드의 위치**도 여기서는 `client_streaming`과 `server_streaming`이라는 평범한 `bool` 필드 두 개로 표현된다.

문법 요소와 descriptor 필드의 대응을 표로 정리하면 다음과 같다.

| `.proto` 문법 요소 | descriptor 표현 | 핵심 필드 |
|---|---|---|
| 파일 1개 | `FileDescriptorProto` | `name`, `package`, `syntax` |
| `import "x.proto"` | `FileDescriptorProto.dependency` | 의존 파일의 `name` 문자열 |
| `message Foo {}` | `DescriptorProto` | `name`, `field[]` |
| 중첩 메시지 | `DescriptorProto.nested_type` | 재귀적 `DescriptorProto` |
| `int64 order_id = 1;` | `FieldDescriptorProto` | `name`, `number=1`, `type=TYPE_INT64` |
| `repeated string tags = 4;` | `FieldDescriptorProto` | `label=LABEL_REPEATED` |
| `oneof payment {}` | `OneofDescriptorProto` + 필드의 `oneof_index` | |
| `enum Status {}` | `EnumDescriptorProto` | |
| `service OrderService {}` | `ServiceDescriptorProto` | `name`, `method[]` |
| `rpc GetOrder(Req) returns (Res);` | `MethodDescriptorProto` | `input_type`, `output_type` |
| `rpc Watch(Req) returns (stream Res);` | `MethodDescriptorProto` | `server_streaming=true` |
| `option go_package = "...";` | `FileOptions` | |
| `// 주석` | `SourceCodeInfo` | 위치(path)별 leading/trailing 주석 |

### 왜 이렇게까지 자기 참조적으로 만들었나

"파서가 만든 AST를 그냥 메모리 자료구조로 두면 될 텐데, 왜 굳이 protobuf 메시지로 표현했을까?" 이 질문의 답이 이 장 후반부에서 설명할 여러 기능이 가능한 이유이다.

1. **직렬화를 따로 만들 필요가 없다.** IR이 protobuf 메시지이므로, protoc가 IR을 바이트로 직렬화해 다른 프로세스(플러그인)에 넘기는 일은 자연스럽게 해결된다. AST를 언어 중립적으로 직렬화할 별도 포맷을 만들 필요가 없다. protobuf가 이미 그 일을 하기 때문이다(protobuf 자신을 표현하는 데 protobuf를 쓴다).
2. **모든 언어가 IR을 읽을 수 있다.** 플러그인을 Go로 작성했든 Rust로 작성했든, `descriptor.proto`로부터 생성한 자기 언어의 코드로 IR을 역직렬화해 다룬다. IR을 다루는 데 특별한 파서는 필요 없고, protobuf 런타임만 있으면 충분하다.
3. **런타임 리플렉션의 기반이 된다.** 이 `FileDescriptorProto`(또는 이를 묶은 `FileDescriptorSet`)를 **프로그램이 실행되는 동안에도** 가지고 있으면, 컴파일 시점에 알지 못했던 메시지의 구조를 런타임에 알아낼 수 있다. gRPC 서버 리플렉션(14장 관찰성과 디버깅)이 바로 이 원리로 동작한다. 서버는 자신이 제공하는 서비스들의 `FileDescriptorProto`를 메모리에 가지고 있다가, `grpcurl` 같은 클라이언트가 "어떤 메서드가 있는가?"라고 물으면 그 descriptor를 그대로 돌려준다. 이 연결은 9절에서 다시 자세히 살펴본다.

> 비유하자면 descriptor.proto는 **"사전을 설명하는 사전"** 이다. 보통 사전은 단어를 설명하지만, 이 사전은 "사전이 어떤 구조로 이루어져 있는가(표제어, 품사, 뜻풀이...)"를 설명한다. 그리고 이 메타 사전도 같은 사전 포맷으로 작성되어 있으므로, 사전을 읽는 도구로 메타 사전도 읽을 수 있다.

---

## 3. 플러그인 프로토콜: stdin도 프로토, stdout도 프로토

이제 프런트엔드(IR 생성)와 백엔드(코드 생성)가 어떻게 연결되는지, 둘 사이의 **계약(protocol)** 을 살펴보자. 핵심을 한 문장으로 요약하면 다음과 같다.

> **protoc는 플러그인을 자식 프로세스로 실행하고, `CodeGeneratorRequest`라는 protobuf 메시지를 직렬화해 그 프로세스의 stdin으로 보낸다. 플러그인은 `CodeGeneratorResponse`라는 protobuf 메시지를 직렬화해 stdout으로 돌려준다. 계약은 이것이 전부다.**

유닉스 파이프와 표준 입출력, 그리고 protobuf를 조합한 방식으로, 1970년대 유닉스 철학과 현대 직렬화 기술이 만나는 지점이다. 두 메시지의 정의도 `.proto`로 작성되어 있다(`google/protobuf/compiler/plugin.proto`).

```proto
syntax = "proto2";
package google.protobuf.compiler;
import "google/protobuf/descriptor.proto";

// protoc → 플러그인 (stdin 으로 흐름)
message CodeGeneratorRequest {
  // 사용자가 명령줄에 적은, 코드를 "직접" 생성해 달라고 한 파일들
  repeated string file_to_generate = 1;

  // --go_out=plugins=grpc:. 처럼 콜론 앞에 붙인 옵션 문자열
  optional string parameter = 2;

  // file_to_generate 와 그것들이 의존하는 "모든" 파일의 descriptor.
  // 위상정렬되어 있어, 의존 대상이 항상 먼저 온다.
  repeated FileDescriptorProto proto_file = 15;

  // 컴파일러 버전
  optional Version compiler_version = 3;
}

// 플러그인 → protoc (stdout 으로 흐름)
message CodeGeneratorResponse {
  optional string error = 1;   // 비어있지 않으면 protoc 가 실패로 간주

  // 플러그인이 지원하는 기능 비트마스크 (예: proto3 optional 지원)
  optional uint64 supported_features = 2;

  // 생성할 파일들 — 디스크에 쓰는 건 protoc 의 몫
  repeated File file = 15;

  message File {
    optional string name = 1;            // 출력 파일 경로 (상대)
    optional string insertion_point = 2; // 기존 파일의 특정 지점에 삽입
    optional string content = 15;        // 파일 내용 (소스 코드 텍스트!)
  }
}
```

여기에는 눈여겨볼 세부 사항이 많다.

**`file_to_generate`와 `proto_file`의 구분.** 미묘하지만 중요한 차이이다. `protoc order.proto`를 실행했는데 `order.proto`가 `common.proto`를 import하고 있다면, protoc는 **두 파일의 descriptor**를 모두 `proto_file`에 담아 보낸다. 플러그인이 타입 참조를 해석할 수 있어야 하기 때문이다. 하지만 `file_to_generate`에는 `order.proto`만 들어간다. 즉 "이 파일은 참조용으로만 주는 것이니 코드는 만들지 말라(common은 이미 다른 곳에서 생성되었을 것이다)"와 "이 파일은 실제로 코드를 만들라"를 구분하는 것이다. 이 구분을 무시하는 단순한 플러그인은 의존 파일의 코드까지 중복으로 생성해 충돌을 일으킨다.

**`content`는 단순한 문자열이다.** 플러그인은 추상 구문 트리를 돌려주지 않고, 완성된 소스 코드 **텍스트**를 돌려준다. protoc는 그 텍스트가 유효한 Go 코드인지 Java 코드인지 검사하지 않고, 받은 그대로 파일에 쓴다. 그래서 플러그인은 본질적으로 "FileDescriptorProto를 입력받아 문자열을 출력하는 순수 함수"에 가깝다. 구조가 이렇게 단순하기 때문에 플러그인을 직접 작성하는 일도 생각보다 쉽다.

**`insertion_point`라는 장치.** 한 플러그인이 만든 파일의 특정 지점에 다른 플러그인이 코드를 끼워 넣을 수 있게 하는 협업 메커니즘이다. 예를 들어 메시지 코드 생성기가 파일 안에 `// @@protoc_insertion_point(messages)` 같은 마커를 남겨 두면, 다른 플러그인이 같은 파일명과 `insertion_point`를 지정해 그 자리에 코드를 넣는다. 실무에서 자주 쓰지는 않지만, 생성된 코드를 확장하는 고급 기법이다.

### protoc-gen-\<name\> 이름 규약과 --\<name\>_out 매핑

protoc가 플러그인을 찾는 방법은 전적으로 **이름 규약(convention over configuration)** 에 달려 있다. 복잡해 보이지만 규칙은 한 줄로 정리된다.

```text
   명령줄 플래그              실행할 프로그램 이름
   ──────────────           ────────────────────
   --go_out          →      protoc-gen-go
   --go-grpc_out     →      protoc-gen-go-grpc
   --java_out        →      (protoc 내장 — 외부 프로그램 없음)
   --python_out      →      (protoc 내장)
   --grpc_python_out →      grpc_python_plugin (이름 규약 예외, 곧 설명)
   --dart_out        →      protoc-gen-dart
   --foo_out         →      protoc-gen-foo   ← 당신이 만든 플러그인도 OK
```

규칙은 다음과 같다. **protoc는 `--<name>_out` 플래그를 만나면 `PATH`에서 `protoc-gen-<name>`이라는 실행 파일을 찾아 자식 프로세스로 실행한다.** 이름만 맞으면 그 프로그램이 Go로 작성되었든, 셸 스크립트이든, 파이썬이든 상관없다.

그래서 `--go_out`은 `protoc-gen-go`를, `--connect-go_out`은 `protoc-gen-connect-go`를 호출한다. 새 플러그인을 지원하는 일이 결국 "`protoc-gen-X`라는 이름의 바이너리를 PATH에 두는 것"으로 끝나는 이유가 여기에 있다.

경로를 명시적으로 지정할 수도 있다. `--plugin=protoc-gen-foo=/path/to/my-binary`처럼 쓰면 이름 규약을 거치지 않고 임의의 실행 파일을 특정 플래그에 연결한다.

```text
$ protoc \
    --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    -I proto \
    proto/order/v1/order.proto

  내부에서 벌어지는 일:
  ┌─ protoc 가 order.proto 파싱 → FileDescriptorProto 생성
  ├─ --go_out 발견 → PATH 에서 `protoc-gen-go` 실행
  │    ├─ stdin: CodeGeneratorRequest (parameter="paths=source_relative")
  │    └─ stdout: CodeGeneratorResponse (file: order.pb.go)
  ├─ --go-grpc_out 발견 → PATH 에서 `protoc-gen-go-grpc` 실행
  │    ├─ stdin: 같은 CodeGeneratorRequest 형태 (parameter 만 다름)
  │    └─ stdout: CodeGeneratorResponse (file: order_grpc.pb.go)
  └─ 각 응답의 File 들을 디스크에 쓴다
```

`--go_opt`와 `--go-grpc_opt`로 지정한 값은 `CodeGeneratorRequest.parameter` 문자열로 직렬화되어 해당 플러그인에 전달된다. `paths=source_relative`는 "출력 파일 경로를 입력 .proto의 상대 경로와 같게 두라"는 옵션으로, Go 진영에서는 거의 항상 켠다. 이 옵션을 켜지 않으면 `go_package` 옵션의 전체 경로를 따라 디렉터리가 깊게 만들어진다.

---

## 4. 언어별 플러그인: 왜 어떤 것은 나뉘고 어떤 것은 합쳐졌나

이제 실제 언어 플러그인이 어떻게 구성되어 있는지, 그리고 그렇게 구성한 **설계 이유**를 살펴보자. 가장 흥미롭게 대비되는 것은 Go와 Java이다.

| 언어 | 메시지 코드 | 서비스/스텁 코드 | 형태 |
|---|---|---|---|
| **Go** | `protoc-gen-go` | `protoc-gen-go-grpc` (별도) | 두 플러그인 분리 |
| **Java** | `protoc --java_out` (내장) | `protoc-gen-grpc-java` (별도) | 메시지 내장 + gRPC 별도 |
| **Python** | `grpcio-tools` (`grpc_tools.protoc`) | 같은 도구가 `--grpc_python_out` | 한 패키지가 둘 다 |
| **C#** | `protoc --csharp_out` (내장) | `Grpc.Tools` (`--grpc_out`) | 메시지 내장 + gRPC 별도 |
| **TypeScript** | `protoc-gen-es` 등 | `protoc-gen-connect-es` 등 | 다양한 서드파티 조합 |

### 4-1. Go: 메시지(`protoc-gen-go`)와 서비스(`protoc-gen-go-grpc`)의 분리

Go에서는 완전히 별개인 플러그인 두 개가 두 종류의 파일을 만든다.

```text
   order.proto
       │
       ├──[protoc-gen-go]─────►  order.pb.go
       │                         (message Order, GetOrderRequest 등의
       │                          struct + Getter + Reset/String/ProtoReflect)
       │
       └──[protoc-gen-go-grpc]─►  order_grpc.pb.go
                                 (OrderServiceServer 인터페이스,
                                  OrderServiceClient 인터페이스,
                                  RegisterOrderServiceServer 등)
```

**왜 나누었을까?** 단순한 취향 문제가 아니라 의존성 위생(dependency hygiene)의 문제이다.

1. **관심사의 분리.** Protocol Buffers는 직렬화 포맷이고, gRPC는 그 위에 올린 RPC 프레임워크이다. 둘은 서로 독립적이다. 메시지만 쓰고 RPC는 쓰지 않는 사람(예: 메시지를 Kafka 페이로드로만 쓰는 경우)에게 gRPC 라이브러리 의존성을 강요해서는 안 된다. 플러그인을 나누면 메시지 코드는 `google.golang.org/protobuf`에만 의존하고, gRPC 코드는 `google.golang.org/grpc`에 의존하도록 깔끔하게 분리된다.
2. **역사적 배경: `--go_out=plugins=grpc`의 폐기.** 과거에는 `protoc-gen-go` 하나가 `plugins=grpc` 옵션을 받아 메시지 코드와 gRPC 코드를 한꺼번에 만들었다. 그러나 이 방식은 각자 독립적으로 발전해야 할 코드 생성기 두 개(protobuf 런타임 v2 마이그레이션과 gRPC API 변화)를 한 바이너리에 묶어 버려서, 한쪽을 고치면 다른 쪽이 깨지기 쉬웠다. 그래서 protobuf-go v2(`google.golang.org/protobuf`)로 넘어오면서 gRPC 생성기를 `protoc-gen-go-grpc`로 완전히 분리했다. 예전 `github.com/golang/protobuf`의 `protoc-gen-go`는 `plugins=grpc`를 받았지만, 지금의 v2 `protoc-gen-go`는 이 옵션을 아예 지원하지 않는다. 별도 플러그인인 `protoc-gen-go-grpc`를 쓰라는 뜻이다.
3. **버전 독립성.** 메시지 생성기와 서비스 생성기를 각자의 속도로 버전업할 수 있다. gRPC가 새 스텁 시그니처(예: 컨텍스트 전파 방식 변경)를 도입해도 메시지 코드는 바뀌지 않는다.

### 4-2. Java: 메시지는 내장, gRPC는 별도, 그리고 Builder 패턴

Java는 메시지 코드 생성기가 **protoc 바이너리에 내장**(`--java_out`)되어 있다. C++과 함께 역사적으로 가장 먼저 지원된 언어이기 때문이다. gRPC 서비스 코드는 `protoc-gen-grpc-java`라는 별도 플러그인(`--grpc-java_out`)이 만든다. 여기서도 메시지와 서비스를 분리하는 원칙은 같다.

Java가 특히 다른 점은 메시지를 만드는 방식이다. **메시지 클래스는 불변(immutable)이고 Builder 패턴**으로 생성한다. 5절에서 생성 코드를 분석할 때 자세히 살펴본다.

### 4-3. Python: grpcio-tools 하나로 처리

Python에서는 보통 `grpcio-tools` 패키지가 제공하는 `python -m grpc_tools.protoc`로 호출한다. 이 도구는 **protoc를 내부에 번들**하고 있어서 protoc를 따로 설치할 필요가 없고, `--python_out`(메시지 → `order_pb2.py`)과 `--grpc_python_out`(서비스 → `order_pb2_grpc.py`)을 한 번에 처리한다.

```bash
python -m grpc_tools.protoc \
    -I proto \
    --python_out=. \
    --grpc_python_out=. \
    proto/order/v1/order.proto
```

Python에는 악명 높은 함정이 하나 있다. 생성된 `order_pb2_grpc.py`는 `import order_pb2`처럼 **절대 import**를 쓰는데, 이것이 패키지 구조와 맞지 않아 `ModuleNotFoundError`가 자주 발생한다. `--python_out`의 경로 설정과 `sys.path`를 맞춰야 하며, buf나 명시적인 옵션으로 우회한다. Python 생성기에 오래전부터 남아 있는 불편한 부분이다.

### 4-4. "메시지 코드와 서비스 스텁 분리"의 일반 원칙

언어마다 패키징 방식은 달라도, 거의 모든 생태계가 다음 두 산출물을 **개념적으로 분리**한다.

```text
  ┌──────────────────────────┐     ┌────────────────────────────┐
  │  메시지 코드 (data)        │     │  서비스 스텁 코드 (RPC)       │
  │                          │     │                            │
  │  - 직렬화/역직렬화         │     │  - 서버 인터페이스           │
  │  - Getter/Setter/Builder │     │  - 클라이언트 스텁           │
  │  - 리플렉션 메타           │     │  - 스트림 타입               │
  │                          │     │  - 메서드 디스크립터          │
  │  protobuf 런타임에만 의존   │     │  gRPC 런타임에 의존          │
  └──────────────────────────┘     └────────────────────────────┘
       (proto 만 쓰는 사람용)            (RPC 쓰는 사람용 — 위에 의존)
```

이 분리를 기억해 두면, 다음 절에서 생성 코드를 읽을 때 "이것은 메시지 파일에 있겠고, 저것은 서비스 파일에 있겠다"고 자연스럽게 예측할 수 있다.

---

## 5. 생성된 코드 분석: 무엇이 자동이고 무엇을 직접 채워야 하는가

이론은 이 정도면 충분하다. 이제 실제로 생성된 코드를 한 줄씩 살펴보면서, **어디까지가 도구가 채워 준 부분이고 어디부터가 직접 채워야 할 빈칸인지** 명확히 구분해 보자. 다음 `.proto`를 기준으로 삼는다.

```proto
syntax = "proto3";
package order.v1;

option go_package = "example.com/gen/order/v1;orderv1";

message GetOrderRequest {
  string order_id = 1;
}

message Order {
  string order_id = 1;
  string customer = 2;
  int64  amount   = 3;
  Status status   = 4;

  enum Status {
    STATUS_UNSPECIFIED = 0;
    STATUS_PENDING     = 1;
    STATUS_PAID        = 2;
    STATUS_SHIPPED     = 3;
  }
}

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc WatchOrders(GetOrderRequest) returns (stream Order); // server streaming
}
```

### 5-1. Go 메시지 코드(`order.pb.go`) 분석

`protoc-gen-go`가 만드는 메시지 struct는 대략 다음과 같다(핵심만 발췌했으며, 실제 생성물은 이보다 훨씬 길다).

```go
package orderv1

import (
    protoreflect "google.golang.org/protobuf/reflect/protoreflect"
    protoimpl    "google.golang.org/protobuf/runtime/protoimpl"
)

type Order struct {
    state         protoimpl.MessageState  // 리플렉션·동시성 내부 상태
    sizeCache     protoimpl.SizeCache     // 직렬화 크기 캐시
    unknownFields protoimpl.UnknownFields // 모르는 필드 보존 ([[03 ...]] 참고)

    OrderId  string       `protobuf:"bytes,1,opt,name=order_id,json=orderId,proto3"`
    Customer string       `protobuf:"bytes,2,opt,name=customer,proto3"`
    Amount   int64        `protobuf:"varint,3,opt,name=amount,proto3"`
    Status   Order_Status `protobuf:"varint,4,opt,name=status,enum=order.v1.Order_Status,proto3"`
}

// 각 필드마다 nil-safe Getter 생성
func (x *Order) GetOrderId() string {
    if x != nil { return x.OrderId }
    return ""               // proto3 기본값
}
func (x *Order) GetAmount() int64 {
    if x != nil { return x.Amount }
    return 0
}
func (x *Order) GetStatus() Order_Status {
    if x != nil { return x.Status }
    return Order_STATUS_UNSPECIFIED
}

// protobuf 런타임이 요구하는 3종 세트
func (x *Order) Reset()         { *x = Order{} /* + 리플렉션 초기화 */ }
func (x *Order) String() string { return protoimpl.X.MessageStringOf(x) }
func (*Order) ProtoMessage()    {}

// v2 리플렉션 핵심 — descriptor 와 struct 를 잇는 다리
func (x *Order) ProtoReflect() protoreflect.Message { /* ... */ }

// 중첩 enum
type Order_Status int32
const (
    Order_STATUS_UNSPECIFIED Order_Status = 0
    Order_STATUS_PENDING     Order_Status = 1
    Order_STATUS_PAID        Order_Status = 2
    Order_STATUS_SHIPPED     Order_Status = 3
)
```

살펴볼 점은 다음과 같다.

- **struct 태그가 곧 와이어 포맷 명세이다.** `` `protobuf:"varint,3,opt,name=amount,proto3"` `` 한 줄에 3장(Protocol Buffers 인코딩과 와이어 포맷)에서 다룬 내용이 모두 들어 있다. 와이어 타입(`varint`), 필드 번호(`3`), presence(`opt`), JSON 이름, syntax가 그것이다. 런타임 직렬화기는 이 태그(더 정확히는 `ProtoReflect`)를 읽어 바이트를 만든다.
- **Getter는 항상 nil-safe이다.** `GetAmount()`는 리시버가 `nil`이어도 패닉하지 않고 기본값을 돌려준다. proto3의 "기본값" 원칙(2장 Protocol Buffers 문법과 타입 시스템)을 코드 수준에서 강제하는 장치이다. 그래서 Go gRPC 코드에서는 필드를 `o.Amount`로 직접 읽기보다 `o.GetAmount()`로 읽는 습관이 안전하다(중첩 메시지에서는 특히 그렇다).
- **`Reset/String/ProtoMessage/ProtoReflect`는 사람이 건드리는 영역이 아니다.** 사람이 손댈 일이 전혀 없다. 이 메서드들은 protobuf 런타임이 메시지를 일반적으로(generically) 다룰 수 있도록 인터페이스를 구현한 것이다.
- **숨은 필드 세 개 `state/sizeCache/unknownFields`.** 특히 `unknownFields`는 3장에서 본 "모르는 필드 보존"을 실제로 저장하는 곳이다. 신버전이 추가한 필드를 구버전 코드가 받더라도 버리지 않고 그대로 돌려보내는 전방 호환성([[17 - 실전 - proto 관리와 하위호환성]])이 여기에 기대고 있다.

### 5-2. Go 서비스 코드(`order_grpc.pb.go`) 분석: 빈칸은 어디인가

`protoc-gen-go-grpc`가 만드는 서비스 코드는 **서버 쪽 인터페이스**와 **클라이언트 쪽 스텁**으로 나뉜다.

```go
package orderv1

import (
    context "context"
    grpc    "google.golang.org/grpc"
)

// ── 서버 쪽: 내가 "구현"해야 할 인터페이스 ──────────────
type OrderServiceServer interface {
    GetOrder(context.Context, *GetOrderRequest) (*Order, error)              // unary
    WatchOrders(*GetOrderRequest, OrderService_WatchOrdersServer) error      // server stream
    mustEmbedUnimplementedOrderServiceServer()  // forward-compat 강제
}

// 새 메서드가 추가돼도 깨지지 않게 임베드하는 기본 구현
type UnimplementedOrderServiceServer struct{}
func (UnimplementedOrderServiceServer) GetOrder(context.Context, *GetOrderRequest) (*Order, error) {
    return nil, status.Errorf(codes.Unimplemented, "method GetOrder not implemented")
}
func (UnimplementedOrderServiceServer) WatchOrders(*GetOrderRequest, OrderService_WatchOrdersServer) error {
    return status.Errorf(codes.Unimplemented, "method WatchOrders not implemented")
}

// 내 구현체를 gRPC 서버에 등록하는 함수
func RegisterOrderServiceServer(s grpc.ServiceRegistrar, srv OrderServiceServer) { /* ... */ }

// 서버 streaming 의 송신 핸들
type OrderService_WatchOrdersServer interface {
    Send(*Order) error      // 서버가 클라에게 한 건씩 밀어내는 메서드
    grpc.ServerStream
}

// ── 클라이언트 쪽: 내가 "사용"하는 스텁 ──────────────
type OrderServiceClient interface {
    GetOrder(ctx context.Context, in *GetOrderRequest, opts ...grpc.CallOption) (*Order, error)
    WatchOrders(ctx context.Context, in *GetOrderRequest, opts ...grpc.CallOption) (OrderService_WatchOrdersClient, error)
}

// 채널(연결)을 받아 클라이언트 구현을 만든다 ([[07 ...]] 의 채널/스텁)
func NewOrderServiceClient(cc grpc.ClientConnInterface) OrderServiceClient { /* ... */ }

// 서버 streaming 의 수신 핸들
type OrderService_WatchOrdersClient interface {
    Recv() (*Order, error)  // 클라가 한 건씩 받는 메서드, EOF 로 종료
    grpc.ClientStream
}

// 메서드들의 메타데이터 — 디스패치 테이블
var OrderService_ServiceDesc = grpc.ServiceDesc{
    ServiceName: "order.v1.OrderService",
    HandlerType: (*OrderServiceServer)(nil),
    Methods:     []grpc.MethodDesc{ {MethodName: "GetOrder", Handler: _OrderService_GetOrder_Handler} },
    Streams:     []grpc.StreamDesc{ {StreamName: "WatchOrders", Handler: _OrderService_WatchOrders_Handler, ServerStreams: true} },
    Metadata:    "order/v1/order.proto",
}
```

**자동으로 만들어지는 부분과 직접 작성하는 부분의 경계를 분명히 그어 보자.**

```text
  기계가 만든 것 (절대 손대지 않음)          사람이 채우는 빈칸
  ─────────────────────────────────       ─────────────────────────────
  OrderServiceServer   (인터페이스)    ──►  type myServer struct {...}
  UnimplementedOrderServiceServer            func (s *myServer) GetOrder(...) {
  RegisterOrderServiceServer (함수)             // ★ 진짜 비즈니스 로직 ★
  OrderServiceClient   (인터페이스)             return &Order{...}, nil
  NewOrderServiceClient (함수)               }
  *_WatchOrdersServer/Client (스트림 타입)
  ServiceDesc (디스패치 메타)
```

서버를 만드는 사람이 할 일은 명확하다. **생성된 `OrderServiceServer` 인터페이스를 만족하는 struct를 정의하고, 각 메서드의 본문(비즈니스 로직)을 채우면 된다.** 할 일은 이것뿐이다. 네트워크 수신, HTTP/2 프레이밍([[04 - HTTP2 깊이 보기 - 전송 계층]]), protobuf 역직렬화, 디스패치는 모두 생성된 `ServiceDesc`와 런타임이 처리한다.

```go
// 사람이 작성하는 코드 — 이게 빈칸의 전부다
type orderServer struct {
    orderv1.UnimplementedOrderServiceServer  // forward-compat: 새 메서드 자동 stub
    db *sql.DB
}

func (s *orderServer) GetOrder(ctx context.Context, req *orderv1.GetOrderRequest) (*orderv1.Order, error) {
    o, err := s.lookup(ctx, req.GetOrderId())
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "order %s not found", req.GetOrderId())
    }
    return o, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer()
    orderv1.RegisterOrderServiceServer(s, &orderServer{db: openDB()}) // 등록
    s.Serve(lis)
}
```

> `UnimplementedOrderServiceServer`를 임베드하는 관례가 중요하다. 나중에 `.proto`에 새 rpc가 추가되어 `OrderServiceServer` 인터페이스의 메서드가 늘어나도, 직접 만든 struct는 임베드된 기본 구현 덕분에 **여전히 컴파일된다**(새 메서드는 `Unimplemented`를 반환한다).
>
> 17장(proto 관리와 하위호환성)에서 다루는 서버 측 전방 호환성을 코드 생성 수준에서 받쳐 주는 장치이다. 그래서 `protoc-gen-go-grpc`는 `mustEmbedUnimplemented...`라는 비공개 메서드를 인터페이스에 넣어 임베드를 사실상 강제한다.

### 5-3. Java 메시지 코드: 불변 객체와 Builder

Java 생성 코드는 Go와 성격이 꽤 다르다. 메시지는 **불변 객체**이고, 메시지를 만들려면 **Builder**를 거쳐야 한다.

```java
// 생성된 Order 는 불변. setter 가 없다.
Order order = Order.newBuilder()
        .setOrderId("A-1001")
        .setCustomer("alice")
        .setAmount(42_000L)
        .setStatus(Order.Status.STATUS_PAID)
        .build();                       // build() 시점에 불변 인스턴스 확정

long amount = order.getAmount();        // getter 만 존재
// order.setAmount(...);  ← 컴파일 에러: 그런 메서드 없음

Order modified = order.toBuilder()      // 바꾸려면 builder 로 되돌려 복제
        .setAmount(50_000L)
        .build();
```

**왜 불변 객체와 Builder를 쓸까?** Protocol Buffers 메시지는 여러 스레드가 공유하는 경우가 많고(예: 캐시된 응답), 네트워크로 직렬화되는 "값(value)"이다. 가변 객체를 공유하면 동시성 버그가 생기기 쉽다. 불변으로 만들면 안전하게 공유하고 캐시할 수 있고, Builder는 "여러 단계에 걸쳐 조립한 뒤 마지막에 확정한다"는 흔한 패턴을 깔끔하게 표현한다. Go는 가변 struct에 Getter를 붙이는 다른 선택을 했는데, 이는 언어 관용구의 차이이다.

### 5-4. Java gRPC 스텁 3종: blocking / async / future

`protoc-gen-grpc-java`는 **서비스 하나에 대해 세 종류의 클라이언트 스텁**을 만든다. Java 진영의 다양한 동시성 모델을 반영한 설계이다.

```java
// 같은 채널([[07 ...]]) 위에 세 가지 스텁
ManagedChannel ch = ManagedChannelBuilder.forAddress("localhost", 50051)
        .usePlaintext().build();

// 1) BlockingStub — 동기. 결과가 올 때까지 스레드를 막는다.
OrderServiceGrpc.OrderServiceBlockingStub blocking =
        OrderServiceGrpc.newBlockingStub(ch);
Order o = blocking.getOrder(GetOrderRequest.newBuilder().setOrderId("A-1001").build());
// server streaming 은 Iterator 로:
Iterator<Order> it = blocking.watchOrders(req);

// 2) Stub (async) — 비동기. StreamObserver 콜백으로 받는다.
OrderServiceGrpc.OrderServiceStub async = OrderServiceGrpc.newStub(ch);
async.getOrder(req, new StreamObserver<Order>() {
    public void onNext(Order value)     { /* 결과 수신 */ }
    public void onError(Throwable t)    { /* 에러 */ }
    public void onCompleted()           { /* 종료 */ }
});
// 모든 스트리밍 유형(client/bidi 포함)을 다룰 수 있는 유일한 스텁

// 3) FutureStub — ListenableFuture 반환. unary 전용.
OrderServiceGrpc.OrderServiceFutureStub future = OrderServiceGrpc.newFutureStub(ch);
ListenableFuture<Order> f = future.getOrder(req);
```

| 스텁 | 반환 형태 | unary | server stream | client/bidi stream | 언제 쓰나 |
|---|---|---|---|---|---|
| **BlockingStub** | 값 직접 / `Iterator` | O | O(Iterator) | X | 간단한 동기 코드, 스크립트 |
| **Stub (async)** | `StreamObserver` 콜백 | O | O | O | 진짜 비동기, 모든 스트리밍 |
| **FutureStub** | `ListenableFuture` | O | X | X | 동기를 피하되 future 합성이 필요할 때 |

왜 세 가지나 만들까? **하나의 RPC 의미를 세 가지 동시성 패러다임에 대응시킨** 것이다. 같은 `GetOrder`라도 어떤 코드는 결과를 기다리기만 하면 되고(blocking), 어떤 코드는 이벤트 루프에서 콜백으로 받아야 하며(async), 어떤 코드는 여러 future를 조합해야 한다(future).

스트리밍의 본질(5장 통신의 4가지 방식), 특히 client/bidi 스트리밍의 양방향 동시 송수신은 전통적으로 콜백 모델 없이는 표현하기 어려웠다. 그래서 세 스텁 중 async 스텁만 모든 유형을 지원한다. 코드 생성이 세 가지를 한 번에 만들어 주므로, 사용자는 상황에 맞는 스텁을 고르기만 하면 된다.

> [!note] 보강 메모
> 비교적 최근(grpc-java 1.66+)에는 양방향 스트리밍과 클라이언트 스트리밍까지 **동기적으로** 다룰 수 있는 `BlockingV2Stub`이 추가되어, "스트리밍은 async만 가능하다"는 위 구도가 일부 완화되었다. 다만 *동기 / 콜백 / future* 라는 세 패러다임의 개념적 골격은 그대로이며, 입문 단계에서는 위 3종으로 이해해도 충분하다.

### 5-5. 정리: 자동과 수동을 나누는 보편 법칙

언어와 관계없이 다음 법칙이 성립한다.

```text
  ┌─ 기계가 책임지는 것 (생성 코드) ────────────────────────┐
  │  • 메시지의 직렬화/역직렬화 (와이어 포맷)                  │
  │  • Getter/Builder/기본값/리플렉션 메타                    │
  │  • 서버 인터페이스 / 클라이언트 스텁의 "시그니처"          │
  │  • 메서드 디스패치 테이블, 스트림 타입                     │
  └────────────────────────────────────────────────────────┘
  ┌─ 사람이 책임지는 것 (수작업) ──────────────────────────┐
  │  • 서버 메서드의 "본문" = 비즈니스 로직                    │
  │  • 클라이언트에서 스텁을 "호출"하는 코드                   │
  │  • 채널/서버 셋업, 인터셉터([[08 ...]]), 에러 매핑([[09 ...]]) │
  └────────────────────────────────────────────────────────┘
```

**계약(.proto)이 곧 인터페이스이고, 코드 생성은 그 인터페이스의 타입 안전한 골격을 양쪽 언어로 동시에 만들어 내는 일이다.** 그래서 서버가 Java이고 클라이언트가 Go이더라도, 둘은 같은 `.proto`라는 단일 기준에서 생성된, 서로 짝이 맞는 코드를 쓴다. 6절에서 다룰 buf는 이 단일 기준이 흔들리지 않도록 지켜 준다.

---

## 6. buf: protoc 운영의 어려움을 구조로 해결하다

여기까지 보면 protoc가 강력하다는 것을 알 수 있다. 하지만 **protoc를 직접 운영해 본 사람이라면 그 과정이 꽤 고통스럽다는 것도 안다.** buf는 그 어려움이 어디서 오는지 정확히 진단하고 구조적으로 해결한 현대 툴체인이다. 먼저 어떤 어려움이 있는지부터 살펴보자.

### 6-1. protoc include path 문제

`.proto` 파일은 `import`로 서로를 참조한다(2장 Protocol Buffers 문법과 타입 시스템). 그런데 protoc는 import를 해석할 때 **`-I`(import path, 다른 이름으로 proto_path) 플래그로 지정한 디렉터리에서만** 파일을 찾는다. 의존 관계가 깊어지면 이 `-I` 목록이 감당하기 어려울 만큼 길어진다.

```bash
# 현실의 protoc 명령은 이렇게 생겼다 (그리고 누군가의 로컬에서만 돈다)
protoc \
  -I . \
  -I ./vendor \
  -I ./third_party \
  -I /usr/local/include \
  -I "$(go list -m -f '{{.Dir}}' github.com/grpc-ecosystem/grpc-gateway/v2)" \
  -I "$GOPATH/src" \
  --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  $(find proto -name '*.proto')
```

문제가 한둘이 아니다.

- **`-I` 순서에 따라 다른 파일이 선택될 수 있다**(같은 상대 경로가 여러 트리에 있을 때). 발견하기 어려운 "그림자(shadowing)" 버그이다.
- **공통 의존성(예: `google/protobuf/timestamp.proto`)을 어디서 가져와야 하는가?** protoc에 번들된 `well-known types`의 위치는 설치 방식마다 다르다(`/usr/local/include`, Homebrew 경로 등). 그래서 CI와 로컬 환경이 어긋난다.
- **서드파티 proto(grpc-gateway annotations, googleapis 등)를 vendoring** 하다 보면 third_party 디렉터리가 비대해진다.
- **protoc 버전과 플러그인 버전이 사람마다 다르다.** "내 로컬에서는 됐는데"라는 말이 나오는 전형적인 원인이다.

이 문제들은 모두 선언적인 설정이 없다는 데서 비롯된다. protoc는 명령줄 도구이므로 의존성과 설정이 셸 히스토리와 Makefile 곳곳에 흩어진다.

### 6-2. buf의 해법: 두 개의 설정 파일

buf는 protoc를 **감싸거나 대체하는** 데 그치지 않고, 의존성 해석과 설정을 **선언적인 파일**로 옮긴다. 핵심은 다음 두 파일이다.

```yaml
# buf.yaml — 모듈의 정체성과 규칙. proto 트리 루트에 둔다.
version: v2
modules:
  - path: proto                # .proto 들이 사는 디렉터리
deps:
  - buf.build/googleapis/googleapis     # BSR 의존성 (vendoring 불필요!)
  - buf.build/grpc-ecosystem/grpc-gateway
lint:
  use:
    - STANDARD                 # 표준 스타일 규칙 묶음
breaking:
  use:
    - FILE                     # 하위호환 검사 범주
```

```yaml
# buf.gen.yaml — 무엇을 어떻게 생성할지. protoc 의 --xxx_out 들을 대체.
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go      # 원격 플러그인! 로컬 설치 불필요
    out: gen
    opt: paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt: paths=source_relative
  # 로컬 플러그인을 쓰고 싶으면:
  # - local: protoc-gen-go
  #   out: gen
```

이 두 파일이 있으면 빌드 명령은 **인자 없는 한 줄**로 줄어든다.

```bash
buf generate     # buf.gen.yaml 을 읽어 알아서 생성. -I 지옥 없음.
buf lint         # buf.yaml 의 규칙으로 스타일 검사
buf breaking --against '.git#branch=main'   # main 대비 하위호환 깨짐 검사
```

**무엇이 달라졌고, 왜 중요한가.**

1. **import를 자동으로 해석한다.** buf에는 모듈이라는 개념이 있다. `buf.yaml`에서 모듈 루트를 알면 `import` 경로를 일관되게 해석한다. `-I` 플래그를 손으로 나열할 필요가 없어지고, 그림자 버그도 구조적으로 사라진다.
2. **의존성을 vendoring하지 않고 BSR에서 가져온다.** `deps`에 `buf.build/googleapis/googleapis`만 적으면, buf가 Buf Schema Registry(BSR, 6-4절)에서 그 모듈을 내려받아 import를 해결한다. `third_party`에 다른 팀의 proto를 복사해 둘 필요가 없다. `go.mod`가 의존성을 선언만 하고 `go mod download`가 실제로 가져오는 것과 같은 모델이다.
3. **원격 플러그인을 쓸 수 있다.** `remote: buf.build/protocolbuffers/go`라고 적으면, `protoc-gen-go`를 로컬에 설치하지 않아도 buf가 BSR에서 호스팅하는 플러그인으로 코드를 생성한다. 팀원마다 플러그인 버전이 다른 문제가 사라진다. 격리와 재현성이 중요하다면 `local:`로 버전을 고정한 로컬 바이너리를 쓸 수도 있다.

### 6-3. buf lint와 buf breaking: 계약을 지키는 두 가지 검사

buf가 단순한 빌드 래퍼를 넘어 "스키마 거버넌스 도구"가 되는 것은 lint와 breaking 덕분이다.

**`buf lint`** 는 스타일과 일관성 규칙을 강제한다. 예를 들어 STANDARD 규칙 묶음은 대략 다음과 같은 항목을 검사한다.

```text
  • 서비스 이름은 PascalCase + "Service" 접미사인가?
  • 필드 이름은 lower_snake_case 인가?
  • enum 의 0번 값이 ..._UNSPECIFIED 인가?  (proto3 기본값 함정 방지, [[02 ...]])
  • RPC 요청/응답 타입이 메서드 전용으로 분리돼 있는가?
  • 패키지에 버전 접미사(v1, v2)가 있는가?
  • import 가 실제로 쓰이는가?
```

이 규칙들은 단순히 보기 좋게 만들려는 것이 아니다. 대부분 **나중에 하위호환을 깨거나 스키마 발전을 가로막는 실수를 예방**하기 위한 것이다(0번 enum 함정, 요청/응답 타입 재사용 금지 등).

**`buf breaking`** 은 이 장과 짝을 이루는 17장(proto 관리와 하위호환성)의 핵심 도구이다. 이전 버전의 스키마(`--against`로 지정하며, git 브랜치, BSR 태그, 디렉터리 등을 쓸 수 있다)와 현재 스키마를 비교해 **호환성을 깨는 변경**을 컴파일 에러처럼 잡아낸다.

```bash
# main 브랜치 시점의 스키마 대비, 지금 변경이 호환성을 깨는가?
buf breaking --against '.git#branch=main'
```

```text
  잡아내는 대표 위반:
  ✗ 필드 번호 변경 (1 → 5)           ← 와이어 호환 파괴 ([[03 ...]])
  ✗ 필드 타입 변경 (int32 → string)  ← 디코딩 파괴
  ✗ 필드/메서드 삭제 (reserved 없이)  ← 구버전이 참조
  ✗ enum 값 번호 변경
  ✗ 메서드의 streaming 종류 변경      ← 스텁 시그니처 파괴 ([[05 ...]])
```

이 검사를 CI에 걸어 두면, 누군가 PR에서 무심코 필드 번호를 바꾸는 순간 빌드가 실패한다. **"이벤트는 append-only"** 같은 원칙(17장 proto 관리와 하위호환성)을 사람의 주의력이 아니라 도구로 강제하는 것이다. protoc만 쓰던 시절에는 이런 검사를 하려면 별도 스크립트나 리뷰어의 눈에 의존해야 했다.

### 6-4. Buf Schema Registry(BSR): proto의 패키지 매니저

BSR은 한마디로 **"`.proto`를 위한 npm/Maven Central"** 이다. 스키마 모듈을 버전 태그와 함께 게시(push)하면, 다른 프로젝트가 이를 의존성으로 가져다 쓴다.

```text
   팀 A (스키마 소유)                     팀 B / 프론트엔드 / 모바일
   ┌──────────────┐    buf push          ┌────────────────────────┐
   │ proto/       │ ───────────────────► │ buf.yaml:               │
   │ order/v1/*   │   buf.build/acme/    │   deps:                 │
   └──────────────┘   order  (모듈)       │   - buf.build/acme/order│
                          │               └────────────────────────┘
                          │ BSR 이 자동 생성·호스팅
                          ▼
   ┌───────────────────────────────────────────────────────────────┐
   │ 생성된 SDK 를 언어별로 바로 받아쓰기 (Generated SDKs):           │
   │   Go:     go get buf.build/gen/go/acme/order/...                │
   │   npm:    @buf/acme_order.community_...                         │
   │   Maven:  build.buf.gen:acme_order_...                          │
   └───────────────────────────────────────────────────────────────┘
```

BSR이 해결하는 문제는 다음과 같다.

- **여러 언어 간 동기화.** 백엔드(Go), 프론트엔드(TS), 모바일(Kotlin/Swift)이 같은 모듈의 같은 버전을 참조한다. 누구도 `.proto`를 복사해 가지고 다니지 않으므로, 단일 기준이 실제로 하나로 유지된다.
- **의존성 버전 관리.** `buf.lock` 파일이 의존 모듈의 정확한 커밋을 고정한다(`package-lock.json`과 같은 방식). 그래서 빌드를 재현할 수 있다.
- **Generated SDKs.** BSR이 코드 생성을 직접 호스팅하므로, "go get"이나 "npm install"로 생성된 클라이언트를 바로 받을 수 있다. 소비하는 쪽은 protoc도 buf도 실행할 필요가 없다.
- **문서와 탐색.** 스키마 브라우저에서 메시지와 서비스를 사람이 읽기 좋은 형태로 볼 수 있다.

BSR은 상용 호스팅 서비스이다. 자체 호스팅이나 BSR 없이 운영하는 것도 충분히 가능하며, 그럴 때는 `deps`를 로컬 모듈이나 vendoring으로 대신한다. 중요한 것은 "선언적 의존성"이라는 모델이다.

### 6-5. protoc와 buf 비교

| 측면 | protoc (raw) | buf |
|---|---|---|
| 설정 | 긴 명령줄 / Makefile | `buf.yaml` + `buf.gen.yaml` (선언적) |
| import 해석 | `-I` 수동 나열 (그림자 위험) | 모듈 기반 자동 |
| 의존성 | vendoring / third_party 복사 | BSR `deps` (패키지 매니저) |
| 플러그인 | PATH에 직접 설치 | 원격/로컬 선택, 버전 핀 |
| 스타일 검사 | 없음(별도 도구) | `buf lint` 내장 |
| 하위호환 검사 | 없음(수동/스크립트) | `buf breaking` 내장 |
| 재현성 | 환경 의존적 | `buf.lock` 으로 핀 |
| 빌드 명령 | 길고 깨지기 쉬움 | `buf generate` 한 줄 |

> 정리하면 protoc는 **저수준 컴파일러**이고, buf는 그 위에 있는 **빌드 시스템이자 패키지 매니저이자 린터**이다. C 컴파일러(`cc`)와 Cargo/Gradle의 관계와 같다. buf도 내부에서는 같은 플러그인 프로토콜(3절)로 코드를 생성한다. buf가 새로운 방식을 발명한 것이 아니라, protoc 환경에서 여기저기 흩어져 있던 운영 책임을 선언적인 구조로 모았을 뿐이다.

---

## 7. 빌드 시스템 통합과 "생성 코드를 어디에 둘까"

코드 생성을 손으로 한 번씩 실행하는 것은 학습용이고, 실무에서는 빌드 파이프라인에 연결한다. 통합 방식은 크게 세 가지이다.

### 7-1. Makefile / 셸: 가장 단순한 방식

```makefile
# 가장 흔한 출발점. buf 를 감싸 한 타깃으로.
.PHONY: generate
generate:
	buf generate

.PHONY: lint
lint:
	buf lint
	buf breaking --against '.git#branch=main'
```

작은 팀이나 단일 언어 프로젝트라면 이것으로 충분하다. 단점은 사람이 잊어버리고 실행하지 않을 수 있다는 점이다. 그래서 CI 검증(7-4)을 함께 둔다.

### 7-2. Gradle (Java/Kotlin): protobuf-gradle-plugin

JVM 진영에서는 보통 `com.google.protobuf` Gradle 플러그인으로 코드 생성을 빌드에 통합한다. 이 플러그인은 컴파일 전에 `.proto`로부터 코드를 자동으로 생성한다.

```kotlin
// build.gradle.kts (발췌)
plugins { id("com.google.protobuf") version "0.9.4" }

protobuf {
    protoc { artifact = "com.google.protobuf:protoc:3.25.x" }
    plugins {
        // gRPC 서비스 스텁 생성기 (메시지는 protoc 내장)
        create("grpc") { artifact = "io.grpc:protoc-gen-grpc-java:1.6x.x" }
    }
    generateProtoTasks {
        all().forEach { it.plugins { create("grpc") } }
    }
}
```

이 방식에서는 생성 코드를 **빌드 디렉터리(`build/generated`)** 에 두고 git에 커밋하지 않는 것이 관례이다. `./gradlew build`를 실행할 때마다 코드를 생성하고 곧바로 컴파일한다.

### 7-3. Bazel: rules_proto / rules_go, 그리고 정밀한 캐싱

대규모 모노레포에서는 Bazel이 강력하다. `.proto`를 라이브러리 타깃으로 선언하면 언어별 규칙이 그로부터 코드를 생성한다.

```python
# BUILD.bazel (개념적 발췌)
load("@rules_proto//proto:defs.bzl", "proto_library")
load("@io_bazel_rules_go//proto:def.bzl", "go_proto_library")

proto_library(
    name = "order_proto",
    srcs = ["order/v1/order.proto"],
    deps = ["@com_google_protobuf//:timestamp_proto"],
)

go_proto_library(
    name = "order_go_proto",
    proto = ":order_proto",
    compilers = ["@io_bazel_rules_go//proto:go_grpc"],  # 메시지+gRPC
    importpath = "example.com/gen/order/v1",
)
```

Bazel의 장점은 **세밀한 의존성 그래프와 캐싱**이다. `order.proto`가 바뀌지 않으면 생성과 컴파일을 건너뛰며, 원격 캐시를 공유할 수도 있다. 언어가 여러 개이고 빌드 대상이 수천 개인 모노레포에서는 이 점이 결정적이다. 대신 학습 곡선이 가파르고 설정 비용이 높다.

### 7-4. 핵심 트레이드오프: 생성 코드를 커밋할까, 빌드할 때 생성할까

거의 모든 팀이 한 번쯤 논쟁하는 주제이다. 정답은 상황에 따라 다르지만, 양쪽의 특성을 정확히 알고 있어야 한다.

```text
  (A) 생성 코드를 git 에 커밋한다 (check-in generated code)
  ──────────────────────────────────────────────────────────
   장점:                                단점:
   + 소비자가 protoc/buf 없이 빌드        - PR 디프가 생성물로 지저분
   + IDE 가 즉시 타입을 인식              - "수동 수정" 유혹 (절대 금지)
   + go get 등으로 바로 의존 가능         - 생성기 버전 바뀌면 대량 디프
   + CI 가 더 단순/빠름                   - 사람이 손으로 재생성 잊으면 표류

  (B) 빌드할 때마다 생성한다 (generate at build time)
  ──────────────────────────────────────────────────────────
   장점:                                단점:
   + 생성물이 늘 최신/일관               - 모든 빌드 환경에 툴체인 필요
   + 저장소가 깔끔                       - 빌드가 느려질 수 있음
   + 단일 진실(.proto)만 남음            - 소비자도 생성 환경 갖춰야
```

현실에서는 **두 방식을 절충해 함께 쓰는** 경우가 가장 흔하다.

- 생성 코드를 **커밋하되**, CI에서 **다시 생성한 뒤 차이가 없는지 검증**한다. 이렇게 하면 최신성(B의 장점)과 사용 편의(A의 장점)를 동시에 얻을 수 있다.

```bash
# CI: 생성코드가 .proto 와 일치하는지 검증하는 흔한 패턴
buf generate
if ! git diff --exit-code; then
  echo "ERROR: 생성 코드가 최신이 아닙니다. 'buf generate' 후 커밋하세요."
  exit 1
fi
```

- 특히 **Go 모듈로 배포하는 경우**에는 커밋하는 방식이 사실상 강제된다. `go get`은 소스를 받아 그 자리에서 컴파일할 뿐 protoc를 실행해 주지 않기 때문이다. 그래서 Go gRPC 라이브러리는 거의 모두 `*.pb.go`를 커밋한다.
- 반대로 **단일 서비스 내부에서만 쓰는** proto라면 (B)가 더 깔끔할 수 있다(Gradle이나 Bazel처럼).

> [!note] 한 줄 원칙
> **생성 코드는 절대 손으로 수정하지 않는다.** 커밋하든 하지 않든, 생성 코드는 `.proto`로부터 결정되는 결과물이다. 수정이 필요하면 `.proto`를 고치거나 생성기 옵션을 바꿔야 한다. 생성 파일 맨 위에 있는 `// Code generated by protoc-gen-go. DO NOT EDIT.` 주석은 농담이 아니다.

---

## 8. 버전과 배포 전략: 여러 언어 간 동기화와 semantic import versioning

코드 생성은 한 번 실행하면 끝나는 일이 아니라, `.proto`가 발전하는 동안 계속 따라다니는 문제이다. 배포 전략에서 답해야 할 핵심 질문은 세 가지이다.

### 8-1. 생성 코드를 어디에 두나

```text
  패턴 1: 모노레포 공유 디렉터리
    repo/
      proto/            ← .proto 단일 진실
      gen/go/...        ← 생성된 Go (모든 서비스가 import)
      gen/ts/...        ← 생성된 TS

  패턴 2: 전용 "스키마/SDK" 저장소
    schemas-repo/  →  buf generate  →  publish
      → go: 별도 모듈로 tag (example.com/schemas/gen/go)
      → npm: @org/schemas
      → maven: org.example:schemas

  패턴 3: BSR Generated SDKs
    → .proto 만 BSR 에 push, 소비자는 패키지 매니저로 SDK 설치
```

규칙은 다음과 같다. **생성 코드는 "여러 서비스가 공유하는 라이브러리"** 로 취급한다. 서비스마다 각자 생성하면 버전이 갈라진다. 한 곳에서 생성하고, 모든 서비스가 같은 산출물에 의존하게 해야 한다.

### 8-2. 여러 언어 간 동기화: 하나의 기준, 하나의 시점

백엔드(Go), 프론트엔드(TS), 모바일(Kotlin)이 같은 스키마를 쓸 때, **언제 누가 코드를 생성하느냐**가 버전이 어긋나는 근본 원인이다. 안전한 모델은 "커밋 하나가 모든 언어의 산출물을 동시에 결정"하게 하는 것이다. 모노레포에서 `buf generate`를 한 번에 실행하거나, BSR의 단일 버전 태그를 쓰는 방식이 여기에 해당한다. 그래야 "프론트엔드는 신버전 필드를 보는데 백엔드는 구버전"인 시간차가 생기지 않는다.

필드 단위의 호환성은 17장(proto 관리와 하위호환성)에서 다루고, 여기서는 배포 시점을 맞추는 데 초점을 둔다.

### 8-3. semantic import versioning: 패키지 경로에 버전을 넣는다

이것은 buf lint가 `v1` 같은 패키지 접미사를 권장하는 이유와 바로 연결된다. 핵심 아이디어는 **호환되지 않는 변경(major bump)을 새 타입이 아니라 새 "import 경로"로 표현한다**는 것이다.

```proto
// v1 과 v2 는 별개의 패키지 — 동시에 존재할 수 있다
package order.v1;   // proto/order/v1/order.proto
package order.v2;   // proto/order/v2/order.proto
```

```text
  order.v1.Order   →  example.com/gen/order/v1  (go import path)
  order.v2.Order   →  example.com/gen/order/v2

  → v1 소비자와 v2 소비자가 한 바이너리 안에 공존 가능.
  → 점진적 마이그레이션: 새 코드만 v2 를 import, 구 코드는 v1 유지.
```

왜 이렇게 할까? `Order` 메시지를 호환성을 깨면서 바꾸면(필드 의미를 바꾸는 등), 그 메시지를 import하는 모든 소비자가 한꺼번에 깨진다. 대신 `order.v2`라는 새 패키지로 호환성이 깨지는 변경을 격리하면, v1을 계속 쓰는 서비스는 영향을 받지 않고 새 서비스만 v2로 옮겨 간다. Go 모듈의 `/v2` 규칙(semantic import versioning)과 같은 원칙이다. **major 변경은 새 이름으로 한다**는 단순한 불변식이 호환성 문제를 구조적으로 막는다.

> 정리하면, 작은(호환되는) 변경은 같은 패키지 안에서 17장의 규칙(필드 추가만 허용, 번호 재사용 금지)으로 처리하고, 큰(호환성을 깨는) 변경은 패키지 버전을 올려(`v1`→`v2`) 격리한다.

---

## 9. descriptor와 reflection의 연결: 코드 생성과 짝을 이루는 기능

이제 2절에서 예고한 자기 참조 구조의 이점을 확인할 차례이다. `FileDescriptorProto`는 빌드할 때 IR로 쓰이고 버려지는 것이 아니다. descriptor를 **런타임까지 가지고 있으면, 컴파일 시점에 몰랐던 메시지도 동적으로 다룰 수 있다.**

코드 생성의 결과물은 두 가지 방식으로 쓰인다.

```text
  [정적 모드 — 우리가 5절에서 본 것]
   .proto → 생성 코드 (struct/스텁) → 컴파일타임 타입 안전.
   "나는 Order 타입을 안다. o.GetAmount() 호출 가능."

  [동적 모드 — reflection]
   .proto → FileDescriptorSet (런타임에 메모리/네트워크로 보유)
   → 컴파일타임에 Order 타입을 몰라도, descriptor 를 보고
     "이 메시지엔 amount 라는 int64 필드 3번이 있군" 을 런타임에 알아냄
   → 동적으로 메시지를 만들고/읽고/직렬화 가능.
```

gRPC 서버 리플렉션(14장 관찰성과 디버깅)은 동적 모드의 대표 사례이다. 서버는 빌드할 때 생성된 descriptor를 메모리에 등록해 두고, 별도의 reflection 서비스(`grpc.reflection.v1.ServerReflection`)를 노출한다. `grpcurl` 같은 도구는 `.proto` 파일을 하나도 갖고 있지 않지만, 이 reflection 서비스에 요청해 `FileDescriptorProto`를 받아 온다. 그러면 다음과 같은 일이 가능해진다.

```bash
# grpcurl 은 .proto 없이, 서버의 reflection 으로 스키마를 알아낸다
grpcurl -plaintext localhost:50051 list
# → order.v1.OrderService

grpcurl -plaintext localhost:50051 describe order.v1.OrderService
# → rpc GetOrder ( .order.v1.GetOrderRequest ) returns ( .order.v1.Order );
#   (이 정보는 서버가 들고 있던 ServiceDescriptorProto 그대로다!)

grpcurl -plaintext -d '{"order_id":"A-1001"}' \
    localhost:50051 order.v1.OrderService/GetOrder
# → grpcurl 이 descriptor 로 요청 JSON 을 protobuf 로 인코딩해 호출
```

이 과정을 그림으로 나타내면 다음과 같다.

```text
   grpcurl (스키마 없음)            서버 (descriptor 보유)
        │   1. ServerReflectionInfo 호출    │
        │ ─────────────────────────────────►│
        │   2. FileDescriptorProto 반환      │  ← 빌드 때의 IR 이
        │ ◄─────────────────────────────────│     런타임에 살아있다!
        │   3. descriptor 로 요청을 protobuf │
        │      로 인코딩 ([[03 ...]])         │
        │   4. 실제 RPC 호출                  │
        │ ─────────────────────────────────►│
        │   5. 응답을 descriptor 로 디코딩    │
        │ ◄─────────────────────────────────│
```

**2절에서 본 "프로토를 표현하는 프로토"라는 자기 참조 구조가 14장에서 다루는 동적 디버깅 도구 전체의 토대이다.** descriptor가 단순한 메모리 자료구조였다면 이런 일은 불가능했을 것이다. descriptor가 직렬화할 수 있는 protobuf 메시지이기 때문에 네트워크로 주고받고 런타임에 재구성할 수 있다.

동적 모드의 다른 응용도 같은 토대 위에 있다.

- **`buf breaking`**(6-3절)은 두 시점의 `FileDescriptorSet`을 비교한다. descriptor가 데이터이므로 이전 스키마와 현재 스키마의 차이를 알고리즘으로 계산할 수 있다.
- **JSON ↔ protobuf 트랜스코딩**(16장 gRPC-Web과 REST 게이트웨이)은 런타임에 descriptor를 보고 필드를 매핑한다.
- **범용 프록시와 관찰 도구**는 메시지 타입을 컴파일 시점에 몰라도 descriptor로 페이로드를 디코딩해 로그로 남긴다.

`buf`로 `FileDescriptorSet`을 파일로 뽑아 보면, 이 "데이터로서의 스키마"를 직접 다뤄 볼 수 있다.

```bash
# 모든 .proto 를 하나의 FileDescriptorSet (이미지) 으로 빌드
buf build -o descriptor.binpb
# 이 파일이 바로 reflection 이 돌려주는 것과 같은 종류의 데이터다.
# protoc 로도 가능:
protoc -I proto --include_imports \
    --descriptor_set_out=descriptor.binpb \
    proto/order/v1/order.proto
```

`--include_imports`는 의존 파일의 descriptor까지 포함하라는 옵션이다. 이 옵션을 빼면 이미지가 자체적으로 완결되지 않아서, reflection이나 트랜스코딩에서 타입을 해석하지 못한다.

---

## 10. 실습: 작은 proto로 전 과정을 직접 실행해 본다

이제 이 장에서 다룬 개념을 한 번의 실습으로 연결해 보자. 디렉터리를 만들고, proto를 작성하고, protoc와 buf 양쪽으로 코드를 생성하고, 중간 표현(descriptor)까지 직접 들여다본다.

### 10-1. 프로젝트 골격

```bash
mkdir -p grpc-codegen-lab/proto/order/v1
cd grpc-codegen-lab
```

```proto
# proto/order/v1/order.proto
syntax = "proto3";
package order.v1;

option go_package = "example.com/lab/gen/order/v1;orderv1";

message GetOrderRequest {
  string order_id = 1;
}

message Order {
  string order_id = 1;
  string customer = 2;
  int64  amount   = 3;
}

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc WatchOrders(GetOrderRequest) returns (stream Order);
}
```

### 10-2. raw protoc로 생성하기 (Go)

```bash
# 플러그인 설치 (한 번)
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# PATH 에 $(go env GOPATH)/bin 이 있어야 protoc 가 플러그인을 찾는다
export PATH="$PATH:$(go env GOPATH)/bin"

protoc \
  -I proto \
  --go_out=gen --go_opt=paths=source_relative \
  --go-grpc_out=gen --go-grpc_opt=paths=source_relative \
  proto/order/v1/order.proto

# 산출 트리
find gen -type f
```

예상되는 출력 트리는 다음과 같다.

```text
gen/order/v1/order.pb.go         ← protoc-gen-go      (메시지)
gen/order/v1/order_grpc.pb.go    ← protoc-gen-go-grpc (서비스 스텁)
```

`order.pb.go`에는 5-1절의 `Order` struct와 Getter가 들어 있고, `order_grpc.pb.go`에는 5-2절의 `OrderServiceServer`/`OrderServiceClient`/`ServiceDesc`가 들어 있다. 두 플러그인이 파일을 하나씩 만든 것으로, 4-1절에서 설명한 분리를 직접 확인할 수 있다.

### 10-3. 중간 표현(descriptor)을 직접 뽑아 보기

이 장의 핵심인 `FileDescriptorSet`을 파일로 뽑아, 사람이 읽을 수 있는 형태로 펼쳐 본다.

```bash
# 바이너리 descriptor set 생성 (의존까지 포함)
protoc -I proto --include_imports \
  --descriptor_set_out=order.binpb \
  proto/order/v1/order.proto

ls -l order.binpb     # 이게 2절의 IR 이 직렬화된 바이트다

# descriptor.proto 로 디코딩해 사람이 읽기 (protoc 의 --decode 사용)
protoc --decode=google.protobuf.FileDescriptorSet \
  $(brew --prefix protobuf 2>/dev/null)/include/google/protobuf/descriptor.proto \
  < order.binpb
```

출력되는 내용은 대략 다음과 같다(발췌). 우리가 작성한 `.proto`가 2절의 메시지 구조로 그대로 옮겨져 있는 것을 볼 수 있다.

```text
file {
  name: "order/v1/order.proto"
  package: "order.v1"
  message_type {
    name: "Order"
    field { name: "order_id" number: 1 label: LABEL_OPTIONAL type: TYPE_STRING }
    field { name: "customer" number: 2 label: LABEL_OPTIONAL type: TYPE_STRING }
    field { name: "amount"   number: 3 label: LABEL_OPTIONAL type: TYPE_INT64  }
  }
  service {
    name: "OrderService"
    method {
      name: "GetOrder"
      input_type:  ".order.v1.GetOrderRequest"
      output_type: ".order.v1.Order"
    }
    method {
      name: "WatchOrders"
      input_type:  ".order.v1.GetOrderRequest"
      output_type: ".order.v1.Order"
      server_streaming: true            # ← stream 키워드가 bool 로!
    }
  }
  syntax: "proto3"
}
```

`server_streaming: true` 한 줄을 보자. 5장(통신의 4가지 방식)에서 강조한 `stream` 키워드가 IR에서는 평범한 boolean 필드일 뿐이다. `protoc-gen-go-grpc`는 이 boolean 값을 보고 5-2절의 `OrderService_WatchOrdersClient`(Recv 루프) 타입을 만들지, 평범한 unary 시그니처를 만들지 결정한다. **코드 생성이란 결국 descriptor를 읽고 분기하는 일**이라는 점을 가장 분명하게 보여 주는 예이다.

### 10-4. buf로 같은 작업을 더 깔끔하게

```bash
# 설정 파일 두 장만 쓰면 된다
cat > buf.yaml <<'EOF'
version: v2
modules:
  - path: proto
lint:
  use: [STANDARD]
breaking:
  use: [FILE]
EOF

cat > buf.gen.yaml <<'EOF'
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt: paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt: paths=source_relative
EOF

buf lint                 # 스타일 검사
buf generate             # gen/ 아래에 10-2 와 동일한 산출물 — 단, -I 지옥 없이
buf build -o order.binpb # descriptor set (10-3 의 protoc 와 동일 개념)
```

`buf generate`는 10-2의 긴 protoc 명령과 **정확히 같은 파일**을 만들지만, `-I`도 플러그인 PATH 설정도 없이 한 줄로 끝난다. 이 차이를 직접 체감하는 것이 이 실습의 마지막 목표이다. buf는 새로운 방식이 아니라, 같은 플러그인 프로토콜(3절)을 선언적인 설정으로 감싼 것이다.

### 10-5. 생성된 스텁으로 서버 띄우기 (Go)

마지막으로 5-2절에서 말한 "빈칸 채우기"를 실제로 해 본다.

```go
// cmd/server/main.go
package main

import (
    "context"; "net"
    "google.golang.org/grpc"
    orderv1 "example.com/lab/gen/order/v1"
)

type server struct {
    orderv1.UnimplementedOrderServiceServer // forward-compat
}

func (s *server) GetOrder(ctx context.Context, req *orderv1.GetOrderRequest) (*orderv1.Order, error) {
    return &orderv1.Order{
        OrderId:  req.GetOrderId(),
        Customer: "alice",
        Amount:   42000,
    }, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer()
    orderv1.RegisterOrderServiceServer(s, &server{}) // 생성된 등록 함수
    _ = s.Serve(lis)
}
```

이 30줄 남짓한 코드에서 **사람이 작성한 의미 있는 로직은 `GetOrder`의 본문 하나**뿐이다. struct가 만족하는 인터페이스, 등록 함수, 메시지 타입, 직렬화는 모두 생성 코드와 런타임이 처리한다. 이것이 코드 생성이 주는 이점이다. **계약(.proto)만 올바르게 작성하면 양쪽 언어의 타입 안전한 골격이 저절로 만들어지고, 개발자는 빈칸만 채우면 된다.**
