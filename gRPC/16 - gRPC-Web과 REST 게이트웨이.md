---
title: gRPC-Web과 REST 게이트웨이
date: 2026-06-26
tags:
  - grpc
  - grpc-web
  - rest-gateway
  - grpc-gateway
  - transcoding
  - connect-protocol
  - envoy
  - browser
  - cors
  - 학습노트
---

## 이 장이 답하는 질문

- 브라우저의 `fetch`/`XHR`는 왜 표준 gRPC를 못 쓰는가? 정확히 무엇이 막혀 있는가?
- gRPC-Web은 트레일러(trailer)를 어디에 어떻게 숨기는가? `grpc-web-text`의 base64는 왜 필요한가?
- Envoy의 `grpc_web` 필터와 in-process 프록시는 내부에서 무엇을 변환하는가? 스트리밍은 어디까지 되는가?
- 하나의 `.proto`에 `google.api.http` 어노테이션 하나 붙였을 뿐인데 어떻게 REST/JSON API가 생기는가? path/query/body 매핑 규칙은?
- gRPC-Web, gRPC-Gateway(REST), Connect 중 무엇을 언제 골라야 하는가? CORS·인증·메타데이터는 어떻게 흐르는가?

---

## 1. 브라우저라는 감옥: 왜 표준 gRPC가 안 되는가

gRPC를 처음 배운 사람이 가장 먼저 부딪히는 벽은 의외로 단순하다. "서버는 gRPC로 다 만들었는데, 왜 React 앱에서 직접 호출이 안 되지?" 이 질문의 답을 이해하려면 먼저 표준 gRPC가 HTTP/2 전송 계층의 무엇에 의존하는지를 정확히 짚어야 한다. 이 부분은 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 깊이 다뤘으니 여기서는 "브라우저가 못 하는 것"에만 집중한다.

표준 gRPC가 와이어 위에서 의존하는 HTTP/2의 기능은 크게 세 가지다.

1. **트레일러(trailer) 헤더**: gRPC는 응답의 최종 상태(`grpc-status`, `grpc-message`)를 본문(body)이 아니라 **본문 뒤에 오는 트레일링 HEADERS 프레임**에 담는다. 왜냐하면 서버 스트리밍에서 메시지를 다 보낸 뒤에야 "성공/실패"가 결정되는 경우가 많기 때문이다. 응답을 시작할 때(`:status: 200`) 이미 본문을 흘려보내기 시작했는데, 끝에 가서 "사실 실패했어"라고 말하려면 trailer가 반드시 필요하다.
2. **프레임 수준 제어**: HEADERS/DATA 프레임의 경계, `END_STREAM` 플래그, 멀티플렉싱(multiplexing) 스트림 ID 등을 다룰 수 있어야 한다.
3. **양방향 풀 듀플렉스(full-duplex)**: 클라이언트가 본문을 계속 보내는 동시에 서버 본문을 받는 동작. 양방향 스트리밍(bidirectional streaming)의 전제다.

문제는 브라우저의 JavaScript에 노출된 네트워크 API — `XMLHttpRequest`(XHR)와 `fetch` — 가 이 셋 모두를 충분히 다루지 못한다는 데 있다.

```text
브라우저 JS가 통제할 수 있는 것 vs 표준 gRPC가 요구하는 것

                        브라우저 fetch/XHR        표준 gRPC 필요
HTTP/2 사용 자체           가능(브라우저가 알아서)    필요
요청 trailer 보내기         불가                     (단방향이라 보통 불필요)
응답 trailer 읽기          불가 ← 결정적 장벽         필수 (grpc-status)
DATA 프레임 직접 조작       불가                     라이브러리 내부에서 필요
요청 본문 스트리밍          제한적(부분 지원)          client/bidi 스트리밍에 필요
풀 듀플렉스                사실상 불가               bidi 스트리밍에 필요
```

여기서 결정적인 한 줄은 **"응답 trailer를 읽을 수 없다"** 이다. `fetch`로 받은 `Response` 객체에는 `response.headers`는 있지만 `response.trailers`는 없다. HTTP/2 트레일러는 브라우저 네트워크 스택 내부에서 처리되고 JS 레이어로 올라오지 않는다. 표준 gRPC에서 `grpc-status`가 trailer에 들어가는데, 그걸 읽을 방법이 없으니 클라이언트는 "이 호출이 성공인지 실패인지"를 끝내 알 수 없다. 이게 바로 브라우저에서 표준 gRPC가 통째로 무너지는 지점이다.

비유하자면 이렇다. 표준 gRPC는 택배 상자(본문) 위에 운송장(헤더)을 붙이고 **상자를 다 받은 뒤에 따로 도착하는 "최종 정산서"(trailer)** 로 "정상 배송됨/파손됨"을 통보하는 시스템이다. 그런데 브라우저라는 수령인은 상자와 운송장은 받지만 그 뒤에 오는 정산서를 받을 우편함이 아예 없다. 그러니 물건은 받았는데 이게 정상인지 불량인지 영영 모르는 상황이 된다.

> **자주 하는 오해**: "HTTP/2를 쓰면 gRPC가 되는 거 아냐?" 아니다. HTTP/2를 쓰느냐는 문제가 아니라 **HTTP/2의 어떤 기능을 JS에서 통제할 수 있느냐**가 문제다. 브라우저는 HTTP/2 위에서 통신하지만, 그 프레임/트레일러를 응용 코드가 만질 수 없도록 추상화 뒤에 봉인해 둔다.

그래서 등장한 해법이 세 갈래다. 이 장 전체를 관통하는 지도부터 그려 두자.

```text
                  브라우저/외부에서 gRPC 백엔드를 부르는 3가지 길

  (A) gRPC-Web        : 표준 gRPC와 거의 같은 와이어이되, trailer를 "본문 끝"으로
                        옮긴 변형. 중간 프록시가 gRPC-Web ↔ gRPC를 변환.
                        → 프로토콜 의미(메서드/메시지)는 그대로 유지.

  (B) REST 게이트웨이  : 아예 REST/JSON으로 노출. .proto의 google.api.http
     (transcoding)      어노테이션으로 "POST /v1/orders" 같은 RESTful API를 생성,
                        리버스 프록시가 JSON↔proto를 번역해 gRPC 백엔드 호출.

  (C) Connect          : gRPC/gRPC-Web과 호환되면서 HTTP/1.1·단순 POST+JSON까지
                        지원하는 새 프로토콜. 같은 서버가 세 프로토콜을 동시 응대.
```

세 길은 배타적이지 않다. 한 백엔드가 내부용 gRPC, 브라우저용 gRPC-Web, 파트너용 REST를 동시에 노출하는 일이 흔하다. 이제 하나씩 해부한다.

---

## 2. gRPC-Web 프로토콜 해부

gRPC-Web의 설계 철학은 한 문장으로 요약된다. **"표준 gRPC 와이어 포맷을 최대한 그대로 두되, 브라우저가 못 읽는 trailer만 본문(body) 안으로 끌어내린다."** 메시지 프레이밍, 상태 코드 체계, 메타데이터 의미론은 모두 그대로 재사용한다. 새 프로토콜을 발명하기보다 표준 gRPC의 "최소 변형판"을 만든 셈이다.

### 2.1 콘텐츠 타입(content-type)

gRPC-Web은 네 가지 `content-type`을 정의한다. 두 축의 조합이다: **(직렬화 포맷: proto/그 외) × (전송 인코딩: binary/text)**.

| content-type | 메시지 직렬화 | 전송 인코딩 | 용도 |
|---|---|---|---|
| `application/grpc-web` | protobuf | binary | 기본. `+proto`와 동의 |
| `application/grpc-web+proto` | protobuf | binary | 명시적 proto |
| `application/grpc-web-text` | protobuf | base64 | binary를 못 다루는 환경 |
| `application/grpc-web-text+proto` | protobuf | base64 | 위와 동일, 명시 |

`+json` 변형(`application/grpc-web+json`)도 일부 구현에서 쓰지만 표준 핵심은 proto다. 여기서 `-text` 변형이 왜 따로 있는지가 중요하다. 표준 gRPC 본문은 임의의 바이너리 바이트다. 그런데 일부 브라우저/환경 — 특히 **`XHR`로 진행 중(progress) 이벤트를 받으며 부분 응답을 읽어야 하는 server-streaming** — 에서는 임의 바이너리를 안정적으로 다루기 까다롭다. `XHR.responseText`는 텍스트로 디코딩을 시도하는데, 바이너리 gRPC 프레임에는 텍스트로 깨지는 바이트가 섞여 있다. 그래서 **전체 본문을 base64로 인코딩한 텍스트 스트림**으로 바꾼 것이 `-text`다. base64는 항상 안전한 ASCII만 쓰므로 텍스트 채널을 타고도 안 깨진다. 대가는 약 33% 크기 증가다.

### 2.2 메시지 프레이밍: 표준 gRPC와 동일한 LPM

gRPC-Web 본문은 표준 gRPC와 동일한 **Length-Prefixed Message(LPM)** 프레이밍을 쓴다. 메시지 하나는 다음 5바이트 헤더로 시작한다.

```text
 1 byte    : Compressed-Flag  (0 = 비압축, 1 = 압축)  + 상위 비트로 프레임 종류 구분
 4 bytes   : Message-Length   (big-endian uint32, payload 바이트 수)
 N bytes   : payload          (직렬화된 protobuf 메시지)
```

여기까지는 [[05 - 통신의 4가지 방식 - Unary와 Streaming]]에서 본 표준 gRPC LPM과 똑같다. 핵심 차이는 **flag 바이트의 최상위 비트(MSB, 0x80)** 에 있다.

```text
flag 바이트의 비트 의미 (gRPC-Web)

 bit:  7 6 5 4 3 2 1 0
       │           │
       │           └─ bit 0: Compressed-Flag (1이면 이 프레임 payload가 압축됨)
       └──────────── bit 7 (0x80): Trailer-Flag
                      0x00 → 일반 데이터(메시지) 프레임
                      0x80 → 트레일러(trailer) 프레임 ← gRPC-Web만의 확장
```

즉 **flag 바이트의 0x80 비트가 켜져 있으면 그 프레임은 메시지가 아니라 trailer**다. 이게 gRPC-Web의 전부라 해도 과언이 아니다. 본문은 이런 형태가 된다.

```text
[ 0x00 | len | msg1 ] [ 0x00 | len | msg2 ] ... [ 0x80 | len | TRAILERS ]
   └── 데이터 프레임 (0개 이상) ──┘            └── 마지막에 trailer 프레임 ──┘
```

트레일러 프레임의 payload는 protobuf가 아니라 **HTTP/1.1 스타일의 헤더 텍스트**다. 즉 `key: value\r\n`을 이어붙인 형식이며, 여기에 `grpc-status`, `grpc-message`, 그리고 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에서 다루는 `grpc-status-details-bin`(Rich Error의 base64) 같은 트레일러 메타데이터가 들어간다.

### 2.3 손으로 디코딩하는 gRPC-Web 응답

말로만 하면 추상적이니, 실제 바이트를 손으로 뜯어보자. 다음 `.proto`가 있다고 하자.

```proto
syntax = "proto3";
package echo.v1;

message EchoResponse {
  string text = 1;
}
```

서버가 `text = "hi"`를 담아 성공(`grpc-status: 0`)으로 응답한다. 직렬화 과정은 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 배운 대로다.

```text
EchoResponse{ text="hi" } 의 protobuf 인코딩:
  field 1, wire type 2(LEN): tag = (1<<3)|2 = 0x0A
  length = 2                                = 0x02
  "hi"  = 0x68 0x69
  → 메시지 payload (4바이트): 0A 02 68 69
```

이제 이걸 gRPC-Web 본문으로 감싼다.

```text
① 데이터 프레임
   flag      : 00                (비압축, 데이터)
   length    : 00 00 00 04       (payload 4바이트, big-endian)
   payload   : 0A 02 68 69       (EchoResponse)
   → 00 00 00 00 04 0A 02 68 69

② 트레일러 프레임
   trailer 텍스트: "grpc-status:0\r\n"  (15바이트)
        g  r  p  c  -  s  t  a  t  u  s  :  0 \r \n
        67 72 70 63 2D 73 74 61 74 75 73 3A 30 0D 0A
   flag      : 80                (0x80 = trailer)
   length    : 00 00 00 0F       (15바이트)
   payload   : 67 72 70 63 2D 73 74 61 74 75 73 3A 30 0D 0A
   → 80 00 00 00 0F 67 72 70 63 2D 73 74 61 74 75 73 3A 30 0D 0A

전체 응답 본문 (29바이트):
  00 00 00 00 04 0A 02 68 69 80 00 00 00 0F 67 72 70 63 2D 73 74 61 74 75 73 3A 30 0D 0A
  └──── 데이터 프레임 ────┘ └──────────── 트레일러 프레임 ────────────┘
```

이 본문을 받은 브라우저 클라이언트 라이브러리는 이렇게 처리한다. (1) 첫 바이트 `0x00`을 보고 "데이터 프레임"으로 판단, 길이 4를 읽어 `0A 02 68 69`를 떼어 `EchoResponse`로 역직렬화한다. (2) 다음 바이트 `0x80`을 보고 "trailer 프레임"으로 판단, payload를 HTTP 헤더처럼 파싱해 `grpc-status:0`을 얻는다. (3) status 0 → 성공으로 콜백을 호출한다. **트레일러를 HTTP/2 trailer가 아니라 본문 안에서 읽었기 때문에**, `fetch`/`XHR`로도 상태 코드를 알 수 있게 됐다. 바로 1절에서 말한 "정산서 받을 우편함이 없다"는 문제를, 정산서를 상자 안 맨 밑에 같이 넣는 방식으로 우회한 셈이다.

이제 `grpc-web-text`라면? 위 29바이트 전체를 base64로 인코딩한다.

```text
원본 29바이트를 base64 →  AAAAAAQKAmhpgAAAAA9ncnBjLXN0YXR1czowDQo=

content-type: application/grpc-web-text+proto
응답 본문    : AAAAAAQKAmhpgAAAAA9ncnBjLXN0YXR1czowDQo=
```

base64를 디코딩하면 정확히 위 29바이트가 나온다. 텍스트 채널을 안전하게 통과하기 위한 포장일 뿐, 의미는 동일하다.

에러 응답은 trailer 텍스트만 바뀐다. "order not found"라는 NOT_FOUND(5) 에러라면:

```text
trailer 텍스트: "grpc-status:5\r\ngrpc-message:order not found\r\n"  (45바이트)
trailer 프레임 : 80 00 00 00 2D 67 72 70 63 2D 73 74 61 74 75 73 3A 35 0D 0A
                 67 72 70 63 2D 6D 65 73 73 61 67 65 3A 6F 72 64 65 72 20
                 6E 6F 74 20 66 6F 75 6E 64 0D 0A
```

흥미로운 점: gRPC-Web 에러 응답이라도 **HTTP 상태는 보통 200 OK**다. 진짜 상태는 trailer의 `grpc-status`에 있다. 이는 표준 gRPC와 동일한 철학이며 디버깅할 때 "HTTP 200인데 왜 실패?"로 헷갈리기 쉬운 지점이다. (예외: 프록시 자체가 만든 오류, 인증 실패 등은 진짜 HTTP 4xx/5xx로 나올 수 있다.)

한 가지 더 알아 둘 형태가 **trailers-only 응답**이다. 호출이 데이터 메시지를 한 개도 만들지 못하고 곧장 실패하면(예: 인가 거부, 메서드 없음), 응답 본문에 데이터 프레임(`0x00`) 없이 **트레일러 프레임(`0x80`) 하나만** 담겨 온다. 즉 `[ 0x80 | len | "grpc-status:7\r\n..." ]`만 있는 본문이다. 클라이언트 라이브러리는 "데이터가 0개여도 trailer만으로 상태를 판정"할 수 있어야 한다. 표준 gRPC에서는 이 경우 DATA 프레임 없이 trailing HEADERS만 오는 것에 대응되며, gRPC-Web 프록시는 그 trailer를 본문 프레임으로 재포장할 뿐이다.

### 2.4 요청(request) 방향

요청은 단순하다. 브라우저 → 서버 방향에는 trailer가 필요 없다(메시지를 다 보내고 나서 별도 상태를 통보할 일이 없으니). 그래서 요청 본문은 데이터 프레임만으로 구성되고, gRPC 메타데이터는 일반 HTTP 헤더로 보낸다.

```text
POST /echo.v1.EchoService/Echo HTTP/1.1
Host: api.example.com
content-type: application/grpc-web+proto
x-grpc-web: 1
accept: application/grpc-web+proto
authorization: Bearer eyJ...        ← 메타데이터는 HTTP 헤더로 ([[08 - 메타데이터와 인터셉터]])

(body) 00 00 00 00 04 0A 02 68 69   ← EchoRequest{text="hi"}를 LPM으로 감싼 것
```

요청 본문에 trailer 프레임이 없다는 점만 빼면 응답과 대칭이다.

### 2.5 압축은 어디서 일어나는가: 메시지 레벨 vs 전송 레벨

2.2에서 flag 바이트의 bit 0이 "이 프레임 payload가 압축됨"을 뜻한다고 했다. 그런데 실무에서 gRPC-Web을 쓰면 이 비트는 거의 항상 0이다. 표준 gRPC와 gRPC-Web의 **압축 모델이 다르기 때문**이다.

- **표준 gRPC**: 메시지(프레임) 단위 압축을 지원한다. 각 LPM 프레임마다 압축 코덱(gzip 등)을 적용하고 flag bit 0으로 "이 프레임은 압축됨"을 표시한다. 클라/서버는 `grpc-encoding`/`grpc-accept-encoding` 메타데이터로 코덱을 협상한다. 프레임마다 켜고 끌 수 있어서 예컨대 작은 메시지는 비압축, 큰 메시지는 압축처럼 선택적 압축이 가능하다.
- **gRPC-Web**: 브라우저 JS가 프레임 단위로 임의 코덱을 푸는 일은 부담이 크다. 특히 server-streaming에서 부분 프레임을 받아가며 스트리밍 해제하는 것은 더 까다롭다. 그래서 gRPC-Web 클라이언트는 **메시지 레벨 압축을 사실상 쓰지 않고**, 대신 **HTTP 전송 레벨 압축(`Content-Encoding: gzip` 등)** 에 맡긴다. 전송 압축 해제는 브라우저 네트워크 스택이 알아서 처리하므로 JS 코드는 손댈 게 없다.

정리하면, gRPC-Web의 compressed-flag(bit 0)는 와이어 포맷 호환을 위해 자리만 잡아 두었을 뿐 보통 0이고 실제 압축은 한 단계 아래 HTTP 계층에서 일어난다. 변환 프록시(Envoy 등)는 두 압축 모델의 간극을 흡수한다. 백엔드가 메시지 레벨 압축을 켰다면 프록시가 그것을 풀어 브라우저로는 비압축(또는 전송 레벨 압축)으로 넘긴다. 그래서 "gRPC-Web 본문이 왜 안 풀리지?" 같은 디버깅을 할 때는, 프레임 flag의 압축 비트가 아니라 응답의 `Content-Encoding` 헤더부터 확인하는 편이 빠르다.

---

## 3. 변환 계층: gRPC-Web ↔ gRPC

gRPC-Web 와이어를 만들어 보냈다고 끝이 아니다. 백엔드 gRPC 서버는 표준 gRPC(트레일러는 HTTP/2 trailer로, 본문은 LPM만)를 말한다. 그러므로 누군가는 **gRPC-Web ↔ gRPC를 양방향 번역**해야 한다. 이 번역기는 보통 둘 중 하나다.

### 3.1 토폴로지

```text
방식 A: Envoy(또는 다른 프록시)의 grpc_web 필터

   ┌──────────┐  gRPC-Web/HTTP1.1·HTTP2   ┌──────────────┐  표준 gRPC/HTTP2  ┌────────────┐
   │ 브라우저 │ ───────────────────────▶ │ Envoy        │ ───────────────▶ │ gRPC 서버  │
   │ (JS app) │ ◀─────────────────────── │ grpc_web     │ ◀─────────────── │ (Go/Java/..)│
   └──────────┘   trailer = 본문 끝 프레임 │ + cors 필터  │  trailer = HTTP2  └────────────┘
                                          └──────────────┘   trailer 프레임

   Envoy가 하는 일:
   - 요청: gRPC-Web 본문(데이터 프레임)을 표준 gRPC 본문으로 패스스루 + content-type 교체
            grpc-web-text면 base64 디코딩
   - 응답: 백엔드의 HTTP/2 trailer(grpc-status 등)를 떼어 본문 끝 0x80 trailer 프레임으로 재포장
            grpc-web-text 요청이었으면 전체를 base64 인코딩
   - CORS preflight(OPTIONS) 처리, expose-headers 설정

방식 B: in-process 프록시 (예: Go의 grpcweb 래퍼, Connect 핸들러)

   ┌──────────┐  gRPC-Web         ┌─────────────────────────────────┐
   │ 브라우저 │ ───────────────▶  │ gRPC 서버 프로세스               │
   │ (JS app) │ ◀───────────────  │  ├─ HTTP 핸들러가 gRPC-Web 감지   │
   └──────────┘                   │  └─ 같은 프로세스 내 gRPC 디스패치 │
                                  └─────────────────────────────────┘
   별도 프록시 없이 한 프로세스가 두 프로토콜을 모두 응대. 운영 단순, 단일 바이너리.
```

방식 A(Envoy)는 인프라 계층에서 한 번에 처리하므로 백엔드 서버 코드를 손대지 않아도 된다는 큰 장점이 있다. CORS, 인증, 레이트 리밋, 관찰성을 게이트웨이에 모을 수 있다([[14 - 관찰성과 디버깅 - Reflection grpcurl]]). 단점은 Envoy라는 운영 컴포넌트가 추가된다는 점.

방식 B(in-process)는 작은 서비스나 데모, Connect 기반 서버에서 매력적이다. 별도 프록시 없이 단일 바이너리가 브라우저 요청을 직접 받는다. 단점은 인프라 공통 관심사(전역 CORS 정책, WAF 등)를 각 서버가 떠안게 된다는 점.

### 3.2 Envoy `grpc_web` 필터 설정 예

```yaml
# envoy.yaml (발췌) — http_connection_manager의 필터 체인
http_filters:
  - name: envoy.filters.http.grpc_web        # gRPC-Web ↔ gRPC 변환
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
  - name: envoy.filters.http.cors            # 브라우저 CORS 처리
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
  - name: envoy.filters.http.router
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

# 라우트에서 CORS 정책 (어떤 origin/header를 허용할지)
routes:
  - match: { prefix: "/" }
    route: { cluster: grpc_backend, timeout: 0s }   # 스트리밍 위해 timeout 0
    typed_per_filter_config:
      envoy.filters.http.cors:
        "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.CorsPolicy
        allow_origin_string_match:
          - prefix: "https://app.example.com"
        allow_methods: "GET, POST, OPTIONS"
        allow_headers: "content-type,x-grpc-web,authorization,x-user-agent"
        expose_headers: "grpc-status,grpc-message,grpc-status-details-bin"
        max_age: "86400"

clusters:
  - name: grpc_backend
    type: STRICT_DNS
    typed_extension_protocol_options:           # 백엔드로는 HTTP/2(h2c) 강제
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: grpc_backend
      endpoints:
        - lb_endpoints:
            - endpoint: { address: { socket_address: { address: backend, port_value: 50051 } } }
```

여기서 `expose_headers`에 `grpc-status`, `grpc-message`, `grpc-status-details-bin`을 노출해 두는 것이 자주 빠뜨리는 함정이다. 브라우저가 CORS 제약 때문에 노출되지 않은 응답 헤더를 못 읽기 때문이다(gRPC-Web에서는 상태가 trailer 프레임 안에 있어 시나리오에 따라 영향이 다르지만, preflight 통과와 일반 헤더 가시성을 위해 넣어 두는 게 안전하다).

### 3.3 스트리밍 제약: 여기가 가장 큰 한계

gRPC-Web의 가장 중요한 실무적 제약은 **스트리밍 지원 범위**다. [[05 - 통신의 4가지 방식 - Unary와 Streaming]]에서 본 4가지 통신 방식 중 브라우저에서 안정적으로 되는 것은 절반뿐이다.

```text
                       표준 gRPC      gRPC-Web (브라우저)
 Unary                   O              O
 Server streaming        O              O  (단, fetch/XHR로 부분 응답 읽기)
 Client streaming        O              X  (대부분 미지원)
 Bidirectional           O              X  (미지원)
```

이유는 1절의 "풀 듀플렉스 불가"로 돌아간다. 브라우저 `fetch`는 **요청 본문을 다 보내기 전에 응답 본문을 받기 시작하는** 진짜 양방향 동작을 일반적으로 지원하지 않는다(`fetch`의 request body streaming은 비교적 최근에 일부 브라우저에 들어왔고 HTTP/2 + 풀듀플렉스 제약이 많다). 그래서:

- **Server streaming**은 된다. 요청은 한 번에 다 보내고(unary 요청), 응답만 여러 메시지로 흘러오면 되는데, 이건 `XHR`의 `onprogress`나 `fetch`의 `ReadableStream`으로 받을 수 있다. `grpc-web-text`가 특히 server streaming에서 가치가 큰 이유가 이것이다 — 부분 응답을 base64 텍스트로 안전하게 누적 파싱할 수 있으니까.
- **Client streaming / Bidi**는 사실상 막혀 있다. 요청을 조금씩 보내면서 응답을 받는 동작 자체가 불가능에 가깝다.

설계 시사점: **브라우저 대상 API는 가능하면 unary로 설계하라.** "실시간 업데이트"가 필요하면 server streaming 또는 WebSocket/SSE 같은 별도 채널을 고려한다. 클라이언트가 데이터를 스트리밍으로 올려야 하는 시나리오(예: 대용량 업로드)는 gRPC-Web으로 풀려 하지 말고 별도 REST 업로드 엔드포인트로 빼는 게 현실적이다.

---

## 4. gRPC-Web 클라이언트 코드

브라우저 측 라이브러리는 크게 두 갈래다: 전통적인 `grpc-web`(Google)과 신세대 `Connect-Web`(connectrpc). 둘 다 `.proto`에서 코드 생성([[06 - 코드 생성과 protoc 툴체인]])해 타입 안전한 스텁을 만든다.

### 4.1 `.proto`와 코드 생성

```proto
syntax = "proto3";
package echo.v1;

service EchoService {
  rpc Echo(EchoRequest) returns (EchoResponse);
  rpc ServerStreamEcho(EchoRequest) returns (stream EchoResponse);
}

message EchoRequest  { string text = 1; }
message EchoResponse { string text = 1; }
```

```bash
# 전통 grpc-web (protoc 플러그인) — JS + TS 선언 생성
protoc -I . echo/v1/echo.proto \
  --js_out=import_style=commonjs:./gen \
  --grpc-web_out=import_style=typescript,mode=grpcwebtext:./gen

# 또는 buf + Connect-ES (요즘 권장 흐름)
# buf.gen.yaml에 plugin: connect-es, es 지정 후
buf generate
```

`mode=grpcwebtext`는 `application/grpc-web-text`(base64)를 쓰고 server streaming까지 지원한다. `mode=grpcweb`은 바이너리(`application/grpc-web+proto`)로 unary에 적합하다.

### 4.2 호출 스니펫 (전통 grpc-web, TypeScript)

```typescript
import { EchoServiceClient } from "./gen/echo/v1/EchoServiceClientPb";
import { EchoRequest } from "./gen/echo/v1/echo_pb";

// hostname은 Envoy(또는 변환 프록시)의 주소
const client = new EchoServiceClient("https://api.example.com");

const req = new EchoRequest();
req.setText("hi");

// unary — 메타데이터는 두 번째 인자(평범한 객체 = HTTP 헤더)
client.echo(req, { authorization: "Bearer eyJ..." }, (err, res) => {
  if (err) {
    // err.code === grpc status, err.message === grpc-message
    console.error("gRPC error", err.code, err.message);
    return;
  }
  console.log("echo:", res.getText());
});

// server streaming — 응답이 여러 번 흘러온다
const stream = client.serverStreamEcho(req, { authorization: "Bearer eyJ..." });
stream.on("data", (res) => console.log("chunk:", res.getText()));
stream.on("status", (s) => console.log("done, grpc-status =", s.code));  // trailer
stream.on("error", (e) => console.error(e.code, e.message));
```

`status` 이벤트가 바로 2.3절에서 손으로 디코딩한 **trailer 프레임**이 라이브러리 콜백으로 올라온 셈이다. 라이브러리가 본문 끝의 `0x80` 프레임을 파싱해 `code`로 변환해 준다.

### 4.3 호출 스니펫 (Connect-ES, 권장)

```typescript
import { createPromiseClient } from "@connectrpc/connect";
import { createGrpcWebTransport } from "@connectrpc/connect-web";
import { EchoService } from "./gen/echo/v1/echo_connect";

// gRPC-Web transport — Envoy 같은 프록시 뒤의 gRPC 백엔드 호출
const transport = createGrpcWebTransport({ baseUrl: "https://api.example.com" });
const client = createPromiseClient(EchoService, transport);

// unary — async/await, 메타데이터는 headers 옵션
const res = await client.echo({ text: "hi" }, {
  headers: { authorization: "Bearer eyJ..." },
});
console.log(res.text);

// server streaming — async iterator
for await (const msg of client.serverStreamEcho({ text: "hi" })) {
  console.log("chunk:", msg.text);
}
```

Connect-ES는 **transport만 바꾸면 같은 클라이언트 코드로 gRPC-Web 또는 Connect 프로토콜을 쓸 수 있어서** 매력적이다(`createConnectTransport`로 교체). 이는 7절에서 더 다룬다.

---

## 5. gRPC-Gateway: 한 .proto로 REST/JSON까지

gRPC-Web이 "gRPC의 의미를 유지한 채 브라우저용 포장만 바꾼" 접근이라면, **gRPC-Gateway**는 발상이 다르다. **"같은 `.proto`를 가지고, gRPC와는 별개로 진짜 RESTful HTTP/JSON API를 자동 생성"** 한다. 외부 파트너, 모바일, `curl`, 웹훅 수신처럼 "그냥 평범한 REST를 원하는" 클라이언트를 위한 길이다.

핵심 도구는 `protoc-gen-grpc-gateway`(Go 생태계)이며, 입력은 `.proto`에 붙인 **`google.api.http` 어노테이션**이다.

### 5.1 google.api.http 어노테이션

```proto
syntax = "proto3";
package order.v1;

import "google/api/annotations.proto";   // google.api.http 정의
import "google/protobuf/empty.proto";

service OrderService {
  // GET /v1/orders/{order_id}  → GetOrder
  rpc GetOrder(GetOrderRequest) returns (Order) {
    option (google.api.http) = {
      get: "/v1/orders/{order_id}"
    };
  }

  // POST /v1/orders  (요청 본문 전체가 Order)
  rpc CreateOrder(CreateOrderRequest) returns (Order) {
    option (google.api.http) = {
      post: "/v1/orders"
      body: "order"            // CreateOrderRequest.order 필드를 HTTP body로
    };
  }

  // PATCH /v1/orders/{order_id}  (부분 수정, body는 order 하위만)
  rpc UpdateOrder(UpdateOrderRequest) returns (Order) {
    option (google.api.http) = {
      patch: "/v1/orders/{order_id}"
      body: "order"
    };
  }

  // DELETE /v1/orders/{order_id}
  rpc DeleteOrder(DeleteOrderRequest) returns (google.protobuf.Empty) {
    option (google.api.http) = {
      delete: "/v1/orders/{order_id}"
    };
  }

  // GET /v1/orders  (query 파라미터로 필터/페이지네이션)
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse) {
    option (google.api.http) = {
      get: "/v1/orders"
    };
  }
}

message GetOrderRequest    { string order_id = 1; }
message DeleteOrderRequest { string order_id = 1; }

message CreateOrderRequest { Order order = 1; }
message UpdateOrderRequest {
  string order_id = 1;
  Order  order    = 2;
}

message ListOrdersRequest {
  int32  page_size  = 1;
  string page_token = 2;
  string status     = 3;   // query: ?status=PAID
}
message ListOrdersResponse {
  repeated Order orders          = 1;
  string         next_page_token = 2;
}

message Order {
  string order_id = 1;
  string customer = 2;
  int64  amount   = 3;
  string status   = 4;
}
```

이 한 파일로 두 세계가 동시에 생긴다.

- gRPC: `order.v1.OrderService/GetOrder` 등 — 내부 서비스가 쓰는 정통 gRPC.
- REST: `GET /v1/orders/{order_id}` 등 — gRPC-Gateway가 만든 리버스 프록시.

### 5.2 매핑 규칙: 무엇이 어디로 가는가

`google.api.http`의 매핑 규칙은 "요청 메시지의 각 필드가 HTTP 요청의 어디에서 채워지는가"를 결정한다. 우선순위 규칙은 명확하다.

```text
요청 메시지 필드의 출처 결정 규칙 (gRPC transcoding)

 1) path 템플릿에 {field} 로 등장한 필드   → URL path에서 채움
       예: get:"/v1/orders/{order_id}"  →  order_id ← path
 2) body 에 지정된 필드(또는 "*")          → HTTP request body(JSON)에서 채움
       body:"order"  →  order ← body
       body:"*"      →  메시지 전체 ← body
 3) 위 둘 다 아닌 나머지 필드               → query string에서 채움
       (GET/DELETE처럼 body 없는 메서드에서 특히)
       예: ?page_size=20&status=PAID
```

이 규칙이 우아한 이유: 한 메시지의 필드들이 **충돌 없이** path/body/query로 정확히 한 번씩 분배된다. `order_id`는 path에 있으니 query로 또 받지 않고, `order`는 body에 있으니 query로 안 받고, 나머지(`page_size`, `status`)는 자연히 query로 떨어진다.

### 5.3 실제 REST 호출 (curl) ↔ gRPC 대응

```bash
# ① GetOrder : path 파라미터
curl https://api.example.com/v1/orders/ord_123
# → gRPC: GetOrder(GetOrderRequest{order_id:"ord_123"})
# ← 응답 JSON: {"orderId":"ord_123","customer":"alice","amount":"5000","status":"PAID"}

# ② CreateOrder : body 매핑 (body:"order" 이므로 본문 = order 객체)
curl -X POST https://api.example.com/v1/orders \
  -H 'content-type: application/json' \
  -d '{"customer":"bob","amount":"3000"}'
# → gRPC: CreateOrder(CreateOrderRequest{order: Order{customer:"bob", amount:3000}})

# ③ UpdateOrder : path + body 동시
curl -X PATCH https://api.example.com/v1/orders/ord_123 \
  -H 'content-type: application/json' \
  -d '{"status":"CANCELLED"}'
# → gRPC: UpdateOrder(UpdateOrderRequest{order_id:"ord_123", order: Order{status:"CANCELLED"}})

# ④ ListOrders : query 파라미터
curl 'https://api.example.com/v1/orders?page_size=20&status=PAID'
# → gRPC: ListOrders(ListOrdersRequest{page_size:20, status:"PAID"})

# ⑤ DeleteOrder
curl -X DELETE https://api.example.com/v1/orders/ord_123
# → gRPC: DeleteOrder(DeleteOrderRequest{order_id:"ord_123"})
```

여기서 ②의 `"amount":"3000"`이 문자열인 점을 주목하라. proto3의 `int64`는 JSON canonical 매핑에서 **문자열**로 표현된다(6.2절에서 이유 설명). 정수로 보내도 보통 허용되지만 응답은 문자열로 온다.

### 5.4 response body 매핑

기본적으로 gRPC 응답 메시지 전체가 JSON 본문이 된다. 하지만 응답의 특정 하위 필드만 본문으로 내보내고 싶을 때 `response_body`를 쓴다.

```proto
rpc GetOrder(GetOrderRequest) returns (GetOrderResponse) {
  option (google.api.http) = {
    get: "/v1/orders/{order_id}"
    response_body: "order"     // 응답 전체가 아니라 GetOrderResponse.order만 본문으로
  };
}
message GetOrderResponse {
  Order  order = 1;
  string trace_id = 2;   // 이건 본문에 안 나감 (response_body:"order"라서)
}
```

이러면 `GET /v1/orders/ord_123`의 JSON 본문이 `GetOrderResponse` 래퍼가 아니라 `Order` 객체 그 자체가 된다. RESTful "리소스를 그대로 반환" 관례에 맞추기 위한 장치다.

### 5.5 additional_bindings: 한 RPC에 여러 경로

같은 RPC를 여러 URL로 노출하려면 `additional_bindings`를 쓴다. 하위 호환을 위해 구/신 경로를 동시에 살리거나, 다른 path 형태를 지원할 때 유용하다.

```proto
rpc GetOrder(GetOrderRequest) returns (Order) {
  option (google.api.http) = {
    get: "/v1/orders/{order_id}"
    additional_bindings { get: "/v1/customers/{customer}/orders/{order_id}" }
    additional_bindings { get: "/v2/orders/{order_id}" }
  };
}
```

### 5.6 gRPC-Gateway 런타임 배치

생성된 게이트웨이 코드는 보통 별도 프로세스(또는 같은 프로세스의 다른 포트)로 뜨는 **리버스 프록시**다.

```text
   ┌──────────┐   REST/JSON (HTTP/1.1)   ┌─────────────────────┐   gRPC (HTTP/2)   ┌────────────┐
   │ curl/    │ ───────────────────────▶ │ gRPC-Gateway        │ ───────────────▶ │ gRPC 서버  │
   │ 파트너   │ ◀─────────────────────── │ (생성된 리버스 프록시) │ ◀─────────────── │ OrderSvc   │
   └──────────┘     JSON 응답            │  JSON↔proto 변환     │    proto 메시지   └────────────┘
                                         └─────────────────────┘
```

```go
// gateway 진입점 (발췌) — 생성된 RegisterOrderServiceHandlerFromEndpoint 사용
func main() {
    ctx := context.Background()
    mux := runtime.NewServeMux(                 // grpc-gateway 런타임 mux
        runtime.WithIncomingHeaderMatcher(customHeaderMatcher), // 8절: 헤더→메타데이터
    )
    // 백엔드 gRPC 서버(:50051)로 dial하는 프록시 핸들러 등록
    err := orderv1.RegisterOrderServiceHandlerFromEndpoint(
        ctx, mux, "localhost:50051",
        []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())},
    )
    if err != nil { log.Fatal(err) }
    // :8080에서 REST 수신
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

이 프록시 자체는 일반 gRPC 클라이언트로서 백엔드에 붙는다. 즉 [[07 - 채널 스텁 커넥션 생명주기]]의 채널/스텁이 게이트웨이 안에서 동작하고, 게이트웨이는 들어온 JSON을 proto로 마샬링해 그 스텁으로 호출한 뒤, 돌아온 proto를 JSON으로 되돌린다.

### 5.7 OpenAPI(Swagger) 자동 생성

같은 어노테이션으로 OpenAPI v2(Swagger) 스펙도 생성할 수 있다. 외부 개발자에게 문서/SDK를 제공할 때 핵심이다.

```bash
# buf 또는 protoc로 openapiv2 생성
protoc -I . order/v1/order.proto \
  --openapiv2_out=./docs \
  --openapiv2_opt=logtostderr=true
# → docs/order/v1/order.swagger.json 생성
#   paths: /v1/orders/{order_id} (get/patch/delete), /v1/orders (get/post) ...
#   definitions: v1Order, v1ListOrdersResponse ...
```

이로써 **단일 진실 공급원(single source of truth)** 이 `.proto` 하나가 된다: gRPC 스텁, REST 프록시, OpenAPI 문서, 그리고 그 문서에서 파생되는 클라이언트 SDK까지 전부 한 파일에서 흘러나온다. [[17 - 실전 - proto 관리와 하위호환성]]에서 다룰 "proto를 API 계약의 중심에 두는" 전략의 정점이 바로 이 transcoding 패턴이다.

---

## 6. Transcoding의 내부 동작: JSON ↔ proto

gRPC-Gateway든 Envoy gRPC-JSON transcoder든, 핵심은 **JSON과 protobuf 사이의 canonical(정준) 매핑**이다. 이 매핑은 Protocol Buffers 공식 사양에 정의돼 있고, [[02 - Protocol Buffers 1 - 문법과 타입 시스템]]의 타입 시스템과 직결된다. 몇 가지 비직관적인 규칙을 짚어야 디버깅이 편하다.

### 6.1 핵심 매핑 규칙 표

| proto 타입 | JSON 표현 | 비고 |
|---|---|---|
| `int32`, `uint32`, `float`, `double`, `bool` | number / true·false | 자연스러움 |
| `int64`, `uint64`, `fixed64`, `sint64` | **문자열** `"123"` | JS Number 정밀도(2^53) 한계 회피 |
| `string` | string | |
| `bytes` | **base64 문자열** | 표준 base64(패딩 포함) |
| `enum` | 이름 문자열 `"PAID"` (또는 정수) | 기본은 이름 |
| `message` | object | 중첩 |
| `repeated T` | array | |
| `map<k,v>` | object | 키는 문자열화 |
| `google.protobuf.Timestamp` | RFC 3339 문자열 `"2026-06-26T01:02:03Z"` | well-known type |
| `google.protobuf.Duration` | `"3.5s"` | |
| `google.protobuf.FieldMask` | 콤마 구분 경로 문자열 | |
| `google.protobuf.Struct`/`Value` | 임의 JSON | |
| field 이름 | **lowerCamelCase** (`order_id`→`orderId`) | 기본 동작 |

### 6.2 왜 int64가 문자열인가

가장 자주 놀라는 지점. proto3 → JSON에서 `int64`/`uint64` 계열은 **문자열로 직렬화**된다. 이유는 JavaScript의 `Number`가 IEEE 754 double이라 안전한 정수 범위가 2^53−1까지이기 때문이다. 64비트 정수(최대 ~9.2×10^18)를 number로 보내면 큰 값이 조용히 깨진다. 그래서 정밀도 손실을 막으려고 문자열로 보낸다. 위 5.3절의 `"amount":"5000"`이 이래서 문자열이었다.

```text
order.amount = 9007199254740993 (2^53 + 1)
  JSON number로 보내면 → 9007199254740992 로 깨짐(!)  ← JS가 가장 가까운 double로 반올림
  JSON string "9007199254740993" 로 보내면 → 안전
```

### 6.3 field 이름 케이스와 enum

기본적으로 proto의 `snake_case` 필드는 JSON에서 `lowerCamelCase`가 된다(`order_id` → `orderId`). 다만 transcoder/마샬러 옵션으로 원본 이름(`order_id`)을 유지하게 할 수 있다(grpc-gateway의 `UseProtoNames`, Go protojson의 `UseProtoNames` 등). enum은 기본이 **이름 문자열**(`"PAID"`)이고 정수도 허용한다. 파트너 API라면 이름 문자열이 가독성·하위호환에 유리하다(번호가 바뀌어도 이름은 안정적).

### 6.4 기본값 생략과 명시적 포함

proto3 JSON 매핑의 또 하나 함정: **기본값(0, "", false, 빈 배열) 필드는 기본적으로 출력에서 생략**될 수 있다. `amount`가 0이면 JSON에 아예 안 나올 수 있다. 클라이언트가 "필드가 없네?"라고 헷갈리지 않게, 보통 마샬러 옵션 `EmitDefaultValues`/`emitUnpopulated`를 켜서 기본값도 항상 내보내도록 한다. 반대로 들어오는 JSON에 모르는 필드가 있으면 기본은 무시(또는 옵션에 따라 거부)다.

### 6.5 Envoy gRPC-JSON Transcoder: 게이트웨이의 대안

gRPC-Gateway가 "생성된 Go 프록시"라면, **Envoy의 gRPC-JSON transcoder 필터**는 "프록시 인프라에서 직접 transcoding"이다. 별도 게이트웨이 프로세스 없이 Envoy가 같은 일을 한다.

```yaml
# Envoy http_filters (발췌) — descriptor set 기반 transcoding
- name: envoy.filters.http.grpc_json_transcoder
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_json_transcoder.v3.GrpcJsonTranscoder
    proto_descriptor: "/etc/envoy/order_descriptor.pb"   # protoc --descriptor_set_out
    services: ["order.v1.OrderService"]
    print_options:
      always_print_primitive_fields: true     # 6.4의 기본값 포함
      preserve_proto_field_names: false        # 6.3의 케이스 변환
```

```bash
# transcoder가 먹을 descriptor set 생성 (어노테이션 포함 모든 의존성 컴파일)
protoc -I . -I third_party \
  --include_imports --include_source_info \
  --descriptor_set_out=order_descriptor.pb \
  order/v1/order.proto
```

| 구분 | gRPC-Gateway | Envoy gRPC-JSON transcoder |
|---|---|---|
| 형태 | 생성된 Go 코드(프록시) | Envoy 필터(설정) |
| 입력 | `.proto` + 플러그인 | descriptor set(`.pb`) |
| 배포 | 별도 서비스/프로세스 | 기존 Envoy에 필터 추가 |
| 커스터마이즈 | Go 미들웨어로 자유롭게 | Envoy 설정 범위 내 |
| 적합 | Go 스택, 세밀한 제어 | 이미 Envoy 쓰는 메시 |

선택 기준은 단순하다. **이미 Envoy/서비스 메시를 쓰고 있다면** transcoder 필터가 인프라 일관성에서 유리하고, **Go 중심 스택에서 미들웨어로 세밀하게 손대고 싶다면** gRPC-Gateway가 유연하다.

---

## 7. Connect 프로토콜: 제3의 길

gRPC-Web과 REST 게이트웨이는 각각 약점이 있다. gRPC-Web은 변환 프록시(Envoy 등)가 사실상 필수고 `curl`로 손쉽게 못 찌른다. REST 게이트웨이는 별도 transcoding 계층과 `.proto` 어노테이션 관리가 필요하다. **Connect(connectrpc)** 는 이 둘의 불편을 한 번에 줄이려는 비교적 새로운 프로토콜 패밀리다.

### 7.1 Connect의 핵심 아이디어

Connect 서버는 **한 엔드포인트에서 세 가지 프로토콜을 동시에 응대**한다.

```text
        ┌──────────────────────── Connect 서버(단일 핸들러) ────────────────────────┐
        │                                                                            │
 gRPC 클라이언트 ───── application/grpc        (HTTP/2, 표준 gRPC) ────────────────▶ │
 gRPC-Web 클라 ─────── application/grpc-web    (HTTP/1.1 or 2) ────────────────────▶ │
 Connect 클라/curl ── application/json 또는 +proto (HTTP/1.1, 단순 POST) ──────────▶ │
        │                                                                            │
        └────────────────────────────────────────────────────────────────────────┘
```

즉 같은 서버가 (1) 정통 gRPC, (2) gRPC-Web, (3) Connect 자체 프로토콜을 전부 말한다. Connect 자체 프로토콜의 매력은 분명하다. **HTTP/1.1에서 동작하고 unary는 그냥 평범한 `POST` + JSON 본문**이다.

### 7.2 Connect unary를 curl로 직접 호출

```bash
# Connect unary — 그냥 평범한 POST + JSON. 프록시도, 특수 프레이밍도 없음!
curl -X POST https://api.example.com/echo.v1.EchoService/Echo \
  -H 'content-type: application/json' \
  -d '{"text":"hi"}'
# ← {"text":"hi"}

# 헤더로 메타데이터, 에러는 HTTP status + JSON body로
curl -X POST https://api.example.com/order.v1.OrderService/GetOrder \
  -H 'content-type: application/json' \
  -H 'authorization: Bearer eyJ...' \
  -d '{"orderId":"nope"}'
# ← HTTP/1.1 404 Not Found
#   {"code":"not_found","message":"order not found"}
```

이 차이가 결정적이다. gRPC-Web 응답을 `curl`로 받으면 2.3절의 `0x80` 프레임이 섞인 바이너리라 사람이 못 읽는다. 반면 **Connect unary는 본문이 그냥 JSON**이라 `curl`/브라우저 devtools/로그에서 그대로 읽힌다. URL 경로는 gRPC와 동일하게 `/<package>.<Service>/<Method>` 형태를 유지하므로 라우팅 규칙도 단순하다.

에러 모델도 브라우저 친화적이다. Connect는 [[09 - 에러 모델 - 상태 코드와 Rich Error]]의 gRPC status를 그대로 쓰되, unary에서는 **HTTP status code로도 매핑**(예: `not_found`→404, `invalid_argument`→400)하고 본문에 `{"code","message","details"}` JSON을 둔다. trailer가 필요 없으니 브라우저가 그냥 읽는다.

### 7.3 Connect 스트리밍과 호환성

Connect도 스트리밍을 정의하지만(서버 스트리밍은 본문에 enveloped 메시지를 이어 보내는 방식), **브라우저 한계는 동일**하다(client/bidi는 제약). "스트리밍을 마법처럼 풀어 준다"가 아니라 "unary와 server-streaming을 더 단순하고 프록시 없이" 제공하는 게 강점이다.

호환성 관점에서 Connect 서버는 표준 gRPC 클라이언트와도 통신한다. 그래서 마이그레이션 전략이 깔끔하다: **서버를 Connect로 구현 → 내부는 기존 gRPC 클라이언트가 그대로 호출, 브라우저는 Connect/gRPC-Web으로 호출**. Envoy 같은 변환 프록시를 (gRPC-Web만 쓸 거라면) 생략할 수 있다.

```go
// Connect 서버 (Go) — net/http에 바로 붙는다. 별도 프록시 불필요
package main

import (
	"context"; "net/http"
	connect "connectrpc.com/connect"
	"golang.org/x/net/http2"; "golang.org/x/net/http2/h2c"
	echov1 "example/gen/echo/v1"
	"example/gen/echo/v1/echov1connect"
)

type echoServer struct{}
func (s *echoServer) Echo(ctx context.Context, req *connect.Request[echov1.EchoRequest]) (*connect.Response[echov1.EchoResponse], error) {
	return connect.NewResponse(&echov1.EchoResponse{Text: req.Msg.Text}), nil
}

func main() {
	mux := http.NewServeMux()
	path, handler := echov1connect.NewEchoServiceHandler(&echoServer{})
	mux.Handle(path, handler)
	// h2c로 HTTP/2(cleartext)도 받아 gRPC 클라이언트까지 동시 지원
	http.ListenAndServe(":8080", h2c.NewHandler(mux, &http2.Server{}))
}
```

이 한 서버에 (a) `grpcurl`/Go gRPC 스텁(application/grpc), (b) Connect-Web 브라우저 클라(grpc-web 또는 connect), (c) `curl`(JSON)이 전부 붙는다. "프록시 한 대 줄이기"가 Connect의 실질적 운영 이득이다.

---

## 8. 보안·CORS·인증·메타데이터 매핑

브라우저/외부 노출은 곧 공격면(attack surface) 확대다. 게이트웨이/프록시 계층에서 처리해야 할 횡단 관심사를 정리한다. TLS/mTLS 기본은 [[11 - 보안 - TLS mTLS 인증]], 메타데이터/인터셉터 흐름은 [[08 - 메타데이터와 인터셉터]]를 전제로 한다.

### 8.1 CORS: 브라우저만의 관문

브라우저에서 cross-origin 요청을 보낼 때 CORS(Cross-Origin Resource Sharing)가 개입한다. gRPC-Web/Connect 요청은 보통 "단순 요청"이 아니라 커스텀 헤더(`x-grpc-web`, `content-type: application/grpc-web+proto`)를 달고 가므로 **preflight(OPTIONS)** 가 먼저 날아간다. 게이트웨이는 이에 응답해야 한다.

```text
브라우저                              Envoy/게이트웨이
   │  OPTIONS /echo.v1.EchoService/Echo   (preflight)
   │  Origin: https://app.example.com
   │  Access-Control-Request-Method: POST
   │  Access-Control-Request-Headers: content-type,x-grpc-web,authorization
   ├────────────────────────────────────▶
   │  204 No Content
   │  Access-Control-Allow-Origin: https://app.example.com
   │  Access-Control-Allow-Methods: POST, OPTIONS
   │  Access-Control-Allow-Headers: content-type,x-grpc-web,authorization
   │  Access-Control-Expose-Headers: grpc-status,grpc-message,grpc-status-details-bin
   │  Access-Control-Max-Age: 86400
   ◀────────────────────────────────────┤
   │  (이제 진짜) POST ... 본문
```

흔한 함정 셋: (1) `Access-Control-Allow-Headers`에 `x-grpc-web`/`authorization`/커스텀 메타데이터 헤더를 빠뜨려 preflight 실패, (2) `Access-Control-Expose-Headers`에 `grpc-status`류를 안 넣어 클라가 상태/에러 못 읽음, (3) 자격증명(쿠키)을 쓰면서 `Allow-Origin: *`를 써서(credentials와 와일드카드 동시 사용 불가) 브라우저가 거부.

### 8.2 인증 토큰의 흐름

표준 gRPC에서 인증 토큰은 metadata(`authorization` 키)로 보낸다([[08 - 메타데이터와 인터셉터]]). 브라우저 변형에서도 동일하게 **HTTP 헤더 `authorization`** 으로 보내면 변환 계층이 이를 백엔드 gRPC metadata로 전달한다.

```text
 브라우저: HTTP 헤더 authorization: Bearer <jwt>
    │  (gRPC-Web 요청 헤더)
    ▼
 Envoy/Gateway: 헤더 → gRPC metadata 키 "authorization" 매핑
    │
    ▼
 gRPC 서버 인터셉터: metadata에서 토큰 추출 → 검증 → context에 사용자 주입
```

게이트웨이 계층에서 JWT 검증을 끝내고(예: Envoy의 `jwt_authn` 필터), 검증 결과를 내부 헤더로 백엔드에 전달하는 패턴이 흔하다. 그러면 백엔드는 "이미 인증된 요청"으로 신뢰하고 인가(authorization)에 집중할 수 있다. 외부 경계(브라우저↔게이트웨이)는 TLS로 암호화하고 내부(게이트웨이↔서비스)는 mTLS로 상호 인증하는 이중 구조가 정석이다([[11 - 보안 - TLS mTLS 인증]]).

### 8.3 헤더 ↔ 메타데이터 매핑 제어

REST 게이트웨이는 모든 HTTP 헤더를 무작정 gRPC metadata로 넘기지 않는다. 화이트리스트/접두어 규칙으로 제어한다.

```go
// grpc-gateway: 어떤 들어온 HTTP 헤더를 metadata로 넘길지 결정
func customHeaderMatcher(key string) (string, bool) {
	switch strings.ToLower(key) {
	case "authorization", "x-request-id":
		return key, true              // 통과 (metadata로 전달)
	default:
		// 기본 규칙: "X-" 접두어 등은 grpcgateway- 접두어로 통과
		return runtime.DefaultHeaderMatcher(key)
	}
}
// 반대 방향: gRPC 응답 metadata/trailer → HTTP 응답 헤더 매핑은
// WithOutgoingHeaderMatcher / ServerMetadata로 제어
```

여기서 명심할 점: **민감한 내부 메타데이터(예: 내부 추적 토큰, 디버그 플래그)가 외부로 새지 않도록** 매처를 보수적으로 설정해야 한다. 기본 동작이 "관대"하면 의도치 않은 헤더 누출이 생긴다. 외부 경계 게이트웨이는 "명시적으로 허용한 것만 통과"라는 화이트리스트 원칙이 안전하다.

### 8.4 페이로드 검증과 한계

REST/JSON로 노출하는 순간 외부에서 임의 JSON이 들어온다. transcoder는 타입 매핑은 해 주지만 **비즈니스 검증은 안 한다**. 큰 본문 거부(요청 크기 제한), 알 수 없는 필드 처리 정책, 숫자/문자열 강제 변환 시 엣지케이스(6.2의 int64 문자열) 등을 게이트웨이/백엔드에서 명시적으로 다뤄야 한다. 특히 [[09 - 에러 모델 - 상태 코드와 Rich Error]]의 `INVALID_ARGUMENT`를 REST에서 400으로 잘 매핑해 클라이언트가 원인을 알게 하는 것이 중요하다.

### 8.5 데드라인·취소의 전파

브라우저 변형이라고 해서 데드라인([[10 - Deadline 취소 타임아웃]])이 사라지는 것은 아니다. 표준 gRPC는 데드라인을 `grpc-timeout` 헤더에 실어 보낸다(형식은 `숫자+단위`, 예: `grpc-timeout: 5S`는 5초, `100m`은 100밀리초). 게이트웨이/프록시 계층은 이 흐름을 그대로 이어 줘야 한다.

- **gRPC-Web**: 클라이언트가 호출에 데드라인을 걸면 라이브러리가 `grpc-timeout` 헤더를 붙이고, 변환 프록시가 이를 백엔드 gRPC 데드라인으로 전달한다. 사용자가 탭을 닫거나 `AbortController`로 취소하면 HTTP 연결이 끊기고 프록시는 이를 감지해 백엔드 호출을 취소(`CANCELLED`)한다.
- **REST 게이트웨이**: HTTP 요청의 수명(연결 종료·서버 타임아웃)이 게이트웨이가 백엔드로 거는 gRPC 호출의 `context.Context` 취소로 이어진다. 즉 클라이언트가 끊으면 게이트웨이→백엔드 호출도 함께 취소되어 자원 낭비를 막는다.

핵심은 데드라인이 경계마다 초기화되는 게 아니라 **"남은 시간(budget)"이 전파**되도록 설계하는 것이다. 게이트웨이가 자기 타임아웃을 백엔드 데드라인보다 짧게 잡으면, 백엔드는 아직 일하는 중인데 게이트웨이가 먼저 끊어 클라이언트에 혼란스러운 504(또는 `DEADLINE_EXCEEDED`)가 가는 상황이 생긴다. 반대로 Envoy 라우트 `timeout: 0s`(3.2절)처럼 무한 타임아웃을 두는 것은 server-streaming을 위해서일 뿐, unary 경로에는 적절한 상한을 두는 게 안전하다. 이 데드라인 버짓 전파 논의는 [[10 - Deadline 취소 타임아웃]]을 그대로 따른다.

---

## 9. 선택 가이드: 무엇을 언제 쓰는가

세 갈래(+내부 gRPC)를 한 표로 정리한다. 절대적 정답은 없고, **클라이언트가 누구냐**가 1차 기준이다.

| 기준 | 내부 gRPC | gRPC-Web | REST 게이트웨이 | Connect |
|---|---|---|---|---|
| 주 대상 | 서비스 간 | 브라우저 | 외부/파트너/`curl` | 브라우저 + 외부 + 내부 |
| 전송 | HTTP/2 | HTTP/1.1·2 | HTTP/1.1 | HTTP/1.1·2 |
| 본문 | proto(binary) | proto/base64 | JSON | JSON 또는 proto |
| 변환 프록시 | 불필요 | 보통 필요(Envoy 등) | 게이트웨이/transcoder | 불필요(서버 내장) |
| `curl` 친화 | 낮음(grpcurl 필요) | 낮음 | 높음 | 높음(JSON) |
| unary | O | O | O | O |
| server streaming | O | O(제약 적음) | 제한적(SSE류로) | O |
| client/bidi streaming | O | X | X | X(브라우저) |
| OpenAPI 문서 | 별도 | 별도 | 자동 생성 | 도구로 가능 |
| 운영 비용 | 낮음 | 프록시 추가 | 게이트웨이 추가 | 낮음 |
| 타입 안전 클라 | 강함 | 강함 | 약함(수기 or 생성) | 강함 |

실무 의사결정 트리.

```text
누가 호출하는가?
├─ 같은 클러스터의 다른 서비스        → 내부 gRPC (변환 없음, 최고 성능)  (07장 참고)
├─ 자체 브라우저 앱(우리가 클라 작성)
│    ├─ 새로 시작 / 프록시 줄이고 싶다  → Connect (curl·JS 친화, 서버 내장)
│    └─ 이미 Envoy 메시가 있다         → gRPC-Web + Envoy grpc_web 필터
├─ 외부 파트너/서드파티(우리가 클라 못 정함)
│    └─ 표준 REST/JSON + OpenAPI 기대  → REST 게이트웨이(transcoding)
└─ 둘 다(브라우저 + 파트너 + 내부)      → Connect 단일 서버로 3프로토콜 동시 응대
```

핵심 통찰 하나: 이 선택들은 **하나의 `.proto`를 공유한 채** 표현 계층만 다르게 가는 것이다([[02 - Protocol Buffers 1 - 문법과 타입 시스템]]). 내부는 gRPC, 외부는 gRPC-Web/REST/Connect — 같은 메시지·같은 서비스 정의에서 파생된다. 그래서 "API 계약은 proto, 노출 방식은 정책"이라는 분리가 가능하고, 이게 [[17 - 실전 - proto 관리와 하위호환성]]에서 다룰 하위호환 전략과 맞물린다.

또 하나: 어느 방식이든 **client/bidi 스트리밍은 브라우저에서 막힌다**는 사실은 변하지 않는다. 이건 프로토콜의 한계가 아니라 1절에서 본 브라우저 네트워크 API의 한계라서 어떤 영리한 프로토콜로도 마법처럼 풀리지 않는다. 진짜 양방향이 필요하면 WebSocket을 별도로 쓰거나, 문제를 server-streaming + 별도 업로드 엔드포인트로 재설계하는 게 정공법이다.

---

## 10. 전체 그림: 한 백엔드, 여러 얼굴

지금까지의 조각을 하나의 운영 토폴로지로 합치면 이렇게 된다.

```text
                                   ┌──────────────────────────────────────┐
   브라우저 앱 ── gRPC-Web/Connect ▶│                                      │
   (React/Vue)                     │   Edge/게이트웨이 계층                 │
                                   │   - CORS / TLS 종단                    │
   파트너 시스템 ── REST/JSON ─────▶│   - JWT 인증 (jwt_authn)              │── 표준 gRPC ──┐
   (curl/SDK)                      │   - gRPC-Web 필터 / JSON transcoder    │  (HTTP/2,mTLS) │
                                   │   - rate limit / 관찰성               │                │
   모바일 앱 ── gRPC 또는 REST ────▶│                                      │                │
                                   └──────────────────────────────────────┘                ▼
                                                                                  ┌────────────────┐
   내부 서비스 ───────── 표준 gRPC (HTTP/2, mTLS) ──────────────────────────────▶│  gRPC 백엔드    │
   (다른 마이크로서비스)                                                          │  OrderService   │
                                                                                  │  (단일 .proto)  │
                                                                                  └────────────────┘
```

왼쪽의 다양한 클라이언트가 각자 편한 포맷으로 들어오고, 게이트웨이 계층이 횡단 관심사를 흡수하며, 오른쪽 백엔드는 **오직 정통 gRPC 하나**만 신경 쓴다. 백엔드 개발자 입장에선 "브라우저용/파트너용/내부용"을 따로 구현하지 않는다. 표현 계층의 다양성이 인프라/게이트웨이로 격리되기 때문이다. 이 격리야말로 gRPC-Web·게이트웨이·Connect가 존재하는 근본 이유다.

---

## 핵심 요약

- 브라우저 `fetch`/`XHR`는 **HTTP/2 응답 trailer를 읽을 수 없다.** 표준 gRPC가 `grpc-status`를 trailer에 두기 때문에, 브라우저에서 표준 gRPC는 그대로 못 쓴다([[04 - HTTP2 깊이 보기 - 전송 계층]]).
- **gRPC-Web**은 표준 gRPC 와이어를 거의 유지하되 trailer를 본문 끝의 특수 프레임으로 옮긴다. **flag 바이트의 0x80 비트**가 켜지면 그 프레임은 메시지가 아니라 trailer(`grpc-status:0\r\n` 같은 HTTP 헤더 텍스트)다. `application/grpc-web-text`는 본문 전체를 base64로 감싸 텍스트 채널 안전성을 얻는다(크기 +33%).
- gRPC-Web은 **변환 계층(Envoy `grpc_web` 필터 또는 in-process 프록시)** 으로 gRPC-Web↔gRPC를 번역한다. 브라우저에서는 **unary와 server-streaming만** 안정적이고, **client/bidi 스트리밍은 불가**하다.
- **gRPC-Gateway**는 `.proto`의 `google.api.http` 어노테이션으로 REST/JSON 리버스 프록시를 생성한다. 요청 필드는 **path 템플릿 → body → 나머지는 query** 순으로 분배되고, `response_body`/`additional_bindings`로 세밀 제어하며, **OpenAPI(swagger) 문서까지 자동 생성**된다.
- transcoding의 심장은 **JSON↔proto canonical 매핑**이다. `int64`는 JS 정밀도 때문에 **문자열**로, `bytes`는 base64로, 필드명은 lowerCamelCase로, Timestamp는 RFC 3339로 매핑된다([[02 - Protocol Buffers 1 - 문법과 타입 시스템]]). **Envoy gRPC-JSON transcoder**는 같은 일을 descriptor set 기반 필터로 한다(게이트웨이의 대안).
- **Connect**는 한 서버가 gRPC·gRPC-Web·Connect 세 프로토콜을 동시 응대하며, unary는 **그냥 POST + JSON**이라 `curl`/브라우저에 가장 친화적이고 변환 프록시를 줄여 준다. 단, 브라우저 스트리밍 한계는 동일하다.
- 선택은 **클라이언트가 누구냐**로 갈린다: 내부=gRPC, 자체 브라우저앱=Connect/gRPC-Web, 외부 파트너=REST 게이트웨이. 모두 **하나의 `.proto`에서 파생**되며, 표현 계층은 게이트웨이가, 계약은 proto가 책임진다.
- 외부 노출에는 **CORS preflight 응답, `expose-headers`에 `grpc-status`류 노출, `authorization` 헤더→metadata 매핑, 보수적 헤더 화이트리스트**가 필수다([[08 - 메타데이터와 인터셉터]], [[11 - 보안 - TLS mTLS 인증]]).

---

## 연결 노트

- [[04 - HTTP2 깊이 보기 - 전송 계층]] — trailer/프레임/멀티플렉싱 등 브라우저가 못 만지는 HTTP/2 기능의 원리
- [[05 - 통신의 4가지 방식 - Unary와 Streaming]] — 4가지 통신 방식과 브라우저 스트리밍 제약의 근거
- [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]] — LPM 프레이밍과 메시지 직렬화(본문 hex 디코딩의 기반)
- [[02 - Protocol Buffers 1 - 문법과 타입 시스템]] — JSON↔proto canonical 매핑이 의존하는 타입 시스템
- [[06 - 코드 생성과 protoc 툴체인]] — grpc-web/Connect 클라이언트 스텁과 게이트웨이/OpenAPI 코드 생성
- [[07 - 채널 스텁 커넥션 생명주기]] — 게이트웨이가 백엔드로 거는 채널/스텁
- [[08 - 메타데이터와 인터셉터]] — 헤더↔메타데이터 매핑, 인증 토큰 전달
- [[09 - 에러 모델 - 상태 코드와 Rich Error]] — trailer의 grpc-status / Connect 에러 JSON / REST 상태 코드 매핑
- [[10 - Deadline 취소 타임아웃]] — grpc-timeout 헤더와 데드라인 버짓이 게이트웨이/프록시를 거쳐 전파되는 방식
- [[11 - 보안 - TLS mTLS 인증]] — 외부 TLS 종단 + 내부 mTLS 이중 구조, CORS와 인증
- [[14 - 관찰성과 디버깅 - Reflection grpcurl]] — 게이트웨이 계층에서의 관찰성/디버깅
- [[17 - 실전 - proto 관리와 하위호환성]] — 단일 .proto에서 여러 표현을 파생시키는 계약 관리 전략
