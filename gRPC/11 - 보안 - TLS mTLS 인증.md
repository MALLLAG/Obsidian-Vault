---
title: "11 - 보안 - TLS mTLS 인증"
date: 2026-06-26
tags:
  - grpc
  - security
  - tls
  - mtls
  - authentication
  - authorization
  - channel-credentials
  - call-credentials
  - alpn
  - sni
  - x509
  - pki
  - spiffe
  - svid
  - oauth2
  - jwt
  - 학습노트
---

## 0. 시작하기 전에: 보안을 두 개의 질문으로 나누기

분산 시스템에서 호출 하나가 안전하려면 성격이 완전히 다른 두 질문에 동시에 답해야 한다.

1. **이 파이프를 도청·변조·가로채기로부터 보호할 수 있는가?** (전송 보안, 즉 연결의 무결성과 기밀성)
2. **이 파이프 반대편에서 요청을 보낸 주체는 누구이며, 그 주장을 믿을 근거가 있는가?** (신원, 즉 인증)

두 질문은 서로 직교한다. 한쪽 답이 다른 쪽 답을 전제하지 않는다는 뜻이다. 파이프가 암호화되어 있어도(1번 해결) 신원 증명이 전혀 없는 요청이 그 안으로 들어올 수 있다. 반대로 강력한 토큰을 가지고 있어도(2번 해결) 그 토큰을 평문으로 보내면 중간자가 토큰을 훔쳐 그대로 흉내 낼 수 있다.

gRPC의 보안 모델이 처음에 복잡해 보이는 이유는 이 직교성을 **타입 시스템 수준에서 그대로 드러냈기** 때문이다. gRPC는 보안을 "보안 옵션" 하나로 뭉뚱그리지 않고, 두 종류의 자격증명 객체로 나눈다.

```
                gRPC 보안의 두 축 (직교)

  채널 자격증명 (Channel Credentials)          호출 자격증명 (Call Credentials)
  ───────────────────────────────            ─────────────────────────────
  "연결(파이프)을 어떻게 보호하는가"             "요청마다 누구인지 어떻게 증명하는가"

  · TLS / SSL                                 · OAuth2 / JWT (Bearer 토큰)
  · mTLS (상호 TLS)                            · Google ADC / 서비스 계정
  · insecure (평문)                            · 커스텀 per-RPC 메타데이터
  · ALTS (구글 환경)                            · API key
  · local

  → "연결 1회당" 적용                            → "RPC 호출 1건당" 적용
  → TLS 핸드셰이크에서 확립                       → metadata(authorization 헤더)로 운반
  → 채널 = 신뢰 경계의 "관(管)"                    → 토큰 = 관 속을 흐르는 "신분증"

         └──────────── 합성(composite) ────────────┘
              CompositeChannelCredentials
        "TLS로 보호된 관" + "관마다/요청마다 토큰"
```

이 그림을 기억해 두면 이 장의 나머지 내용은 모두 이 두 축을 자세히 설명하는 것으로 읽힌다. 채널 자격증명은 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 다룬 전송 계층 바로 위에 올라가는 TLS 레이어를 다룬다. 호출 자격증명은 [[08 - 메타데이터와 인터셉터]]에서 다룬 메타데이터(HTTP/2 헤더) 위에서 동작한다. 인가(authorization)는 여기서 한 번 더 분리되며, 보통 8장(메타데이터와 인터셉터)에서 설명한 서버 측 인터셉터가 정책으로 처리한다.

세 단계로 정리하면 다음과 같다.

| 단계    | 질문         | gRPC에서 담당              | 산출물                |
| ----- | ---------- | ---------------------- | ------------------ |
| 전송 보안 | 파이프가 안전한가  | 채널 자격증명 (TLS/mTLS)     | 암호화·무결성·(상호)인증된 채널 |
| 인증    | 누구인가       | 호출 자격증명 + (m)TLS 피어 신원 | 검증된 주체(principal)  |
| 인가    | 무엇을 해도 되는가 | 서버 인터셉터의 정책 검사         | 허용/거부 결정           |

이 장은 이 표를 위에서 아래로 따라간다. 먼저 파이프(TLS)를 만들고, 양쪽이 서로를 증명하게 한 다음(mTLS), 그 위에 신분증(토큰)을 올린다. 마지막으로 이 주체가 이 작업을 해도 되는지 판정하는 인가를 다룬다.

---

## 1. 전송 보안의 토대: gRPC가 사실상 TLS를 요구하는 이유

### 1.1 HTTP/2와 TLS의 관계

gRPC는 HTTP/2를 전송 계층으로 쓴다(4장 HTTP2 깊이 보기). 그런데 HTTP/2에는 연결을 여는 방식이 두 가지 있다.

- **`h2`**: TLS 위에서 동작하는 HTTP/2이며, ALPN으로 협상한다.
- **`h2c`**: cleartext(평문) HTTP/2이며, TLS 없이 TCP 위에서 바로 동작한다.

웹 브라우저는 `h2c`를 사실상 지원하지 않는다. 즉 브라우저가 쓰는 HTTP/2는 항상 TLS 위의 `h2`다. gRPC도 프로덕션에서는 거의 항상 `h2`(TLS)를 쓴다. `h2c`(평문)는 개발용으로 쓰거나, 사이드카나 서비스 메시 프록시가 이미 mTLS를 처리하고 있을 때 그 뒤의 로컬 홉(hop)에서만 쓴다.

RFC 7540(HTTP/2)은 `h2`의 보안 요구사항을 꽤 엄격하게 정해 두었다. 핵심만 추리면 다음과 같다.

- **TLS 1.2 이상**을 사용해야 한다(RFC 7540 §9.2). 오늘날 실무에서는 TLS 1.2 또는 TLS 1.3(RFC 8446)을 쓴다.
- **SNI 확장**을 반드시 보내야 한다.
- **TLS 수준 압축(TLS-level compression)을 끄고**, **재협상(renegotiation)을 금지**한다.
- 특정 약한 암호 스위트(cipher suite)의 **블랙리스트**가 있다(RFC 7540 Appendix A). 예를 들어 `TLS_RSA_WITH_AES_128_CBC_SHA` 같은 스위트는 `h2`에서 쓸 수 없다. 이를 어기면 `INADEQUATE_SECURITY`라는 HTTP/2 에러 코드(0xc)로 연결이 끊긴다.

여기서 중요한 점이 있다. **HTTP/2의 멀티플렉싱(multiplexing) 효율은 오래 유지되는 연결 하나를 전제로 한다.** TLS 연결 하나 위에서 수백 개의 스트림이 동시에 흐르므로(4장 HTTP2 깊이 보기), 그 연결이 끊기면 그 위의 모든 스트림이 영향을 받는다. 보안도 마찬가지여서, 이 관 하나의 보안 수준이 곧 그 위를 지나는 모든 RPC의 보안 수준이 된다. 그만큼 TLS를 제대로 구성하는 일의 효과가 크다.

### 1.2 ALPN: 핸드셰이크 안에서 h2 사용을 합의하기

`h2`를 쓰려면 클라이언트와 서버가 "이 TLS 연결 위에서 쓸 프로토콜은 HTTP/2다"라고 합의해야 한다. 이 합의를 **추가 왕복 없이** TLS 핸드셰이크 안에 끼워 넣는 메커니즘이 ALPN(Application-Layer Protocol Negotiation, RFC 7301)이다.

흐름은 다음과 같다.

```
 클라이언트                                              서버
   │                                                     │
   │  ClientHello                                        │
   │    + ALPN 확장: ["h2", "http/1.1"]  ───────────────▶│  (클라이언트가 말할 수 있는
   │      (내가 선호하는 순서대로 프로토콜 후보 나열)        │   프로토콜 후보를 제시)
   │                                                     │
   │                                       ServerHello   │
   │◀───────────────  + ALPN 확장: "h2"                  │  (서버가 그중 하나를 "선택")
   │                    (서버가 골라준 단 하나)             │
   │                                                     │
   │   이제 양쪽 다 "이 연결은 h2다"를 안다.                 │
   │   TLS 핸드셰이크가 끝나면 곧바로 HTTP/2 preface 전송     │
   ▼                                                     ▼
```

핵심은 **선택권이 서버에 있다는 점**이다. 클라이언트는 후보 목록을 선호 순서대로 제시할 뿐이고, 최종 선택은 서버가 한다. gRPC 서버는 ALPN으로 `h2`를 광고하고, gRPC 클라이언트도 `h2`를 후보에 넣는다. 서버가 `h2`를 고르면 gRPC 통신이 성립한다. 서버가 `h2`를 지원하지 않아 다른 프로토콜을 고르거나 ALPN 협상 자체가 이루어지지 않으면, 엄격한 gRPC 클라이언트는 연결을 실패로 처리한다.

#### ClientHello를 직접 디코딩해 보기

ALPN은 단순한 설정값이 아니라 실제 바이트로 핸드셰이크에 들어간다. 이를 직접 확인해 보자. 아래는 TLS ClientHello 레코드의 앞부분을 바이트 단위로 풀어 본 것이다(값은 설명을 위한 대표적인 예시다).

```text
TLS 레코드 헤더
  16          ContentType = 22 (Handshake)
  03 01       (legacy) record version = TLS 1.0  ← 호환성 때문에 1.0으로 보냄
  02 00       record length = 0x0200 바이트

핸드셰이크 헤더
  01          HandshakeType = 1 (ClientHello)
  00 01 fc    handshake length = 0x0001fc
  03 03       client_version = TLS 1.2 (0x0303)   ← TLS 1.3도 호환성상 여기엔 1.2를 적음
  <32 bytes>  client_random (32바이트 난수)
  20          session_id length = 32
  <32 bytes>  session_id
  00 08       cipher_suites length = 8바이트 (= 4개 스위트)
  13 01       TLS_AES_128_GCM_SHA256       (TLS 1.3)
  13 02       TLS_AES_256_GCM_SHA384       (TLS 1.3)
  c0 2b       TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 (TLS 1.2)
  c0 2f       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256    (TLS 1.2)
  01          compression_methods length = 1
  00          compression = null  ← HTTP/2는 TLS 압축 금지

확장(extensions) 영역 ...
  -- SNI 확장 (server_name) --
  00 00       extension_type = 0  (server_name)
  00 14       extension_length = 20
  00 12       server_name_list length
  00          name_type = 0 (host_name)
  00 0f       host_name length = 15
  61 70 69 2e 65 78 61 6d 70 6c 65 2e 63 6f 6d   "api.example.com"

  -- ALPN 확장 (application_layer_protocol_negotiation) --
  00 10       extension_type = 16 (ALPN)
  00 0e       extension_length = 14   (= 리스트 길이 필드 2 + 프로토콜 목록 12)
  00 0c       ALPN protocol list length = 12  (= (1+2) + (1+8))
  02          string length = 2
  68 32       "h2"
  08          string length = 8
  68 74 74 70 2f 31 2e 31   "http/1.1"
```

위 디코딩에서 두 가지를 눈여겨보자.

- **확장 타입 `00 00`(SNI)** 에는 `"api.example.com"`이라는 호스트명이 **평문으로** 들어간다(TLS 1.2 기준). 이것이 SNI다. 서버가 IP 하나에서 여러 가상 호스트를 서비스할 때, 클라이언트는 핸드셰이크를 시작하면서 접속하려는 호스트를 이렇게 알려 준다. 서버는 이 값을 보고 어떤 인증서를 제시할지 고른다.
- **확장 타입 `00 10`(ALPN)** 에는 `"h2"`와 `"http/1.1"`이 선호 순서대로 들어간다.

TLS 1.3에서는 ServerHello 이후의 협상 결과(인증서, ALPN 선택 등)가 EncryptedExtensions로 암호화되어 보호된다. 하지만 ClientHello의 SNI는 여전히 평문이다. ECH(Encrypted Client Hello)라는 확장으로 이를 가리는 별도 기술이 있지만, 일반적인 gRPC 운영에서는 거의 쓰지 않는다.

### 1.3 핸드셰이크 → preface → 첫 HEADERS

`h2` TLS 핸드셰이크가 끝나면 곧바로 HTTP/2가 시작된다. 클라이언트는 가장 먼저 **연결 preface**라는 고정 바이트열을 보낸다(4장 HTTP2 깊이 보기).

```text
HTTP/2 connection preface (클라이언트가 보냄, 24바이트 고정)
  50 52 49 20 2a 20 48 54 54 50 2f 32 2e 30 0d 0a   "PRI * HTTP/2.0\r\n"
  0d 0a 53 4d 0d 0a 0d 0a                           "\r\nSM\r\n\r\n"
```

그다음 양쪽이 SETTINGS 프레임을 교환하고, 클라이언트가 gRPC 호출용 HEADERS 프레임을 보낸다. 이 HEADERS에는 `:method: POST`, `:path: /패키지.서비스/메서드`, `content-type: application/grpc`와 함께 **`authorization: Bearer ...` 같은 호출 자격증명 메타데이터**가 들어간다.

즉 토큰은 **TLS로 암호화된 HTTP/2 헤더 안에** 실린다. 이 구조를 기억해 두면, 뒤에서 평문 위의 토큰을 금지하는 이유가 자연스럽게 이해된다.

```
[TCP] ── [TLS 1.3 record layer: 전부 암호화] ──┐
                                               │
   HTTP/2 frames (HEADERS, DATA, ...)          │  ← 이 층 전체가 TLS 안쪽
     HEADERS:                                  │
       :method: POST                           │
       :path: /chat.ChatService/SendMessage    │
       content-type: application/grpc          │
       authorization: Bearer eyJhbGci...  ◀────┼── 토큰은 여기, TLS로 보호됨
     DATA: <protobuf 직렬화 바이트>              │
                                               │
───────────────────────────────────────────────┘
```

---

## 2. 인증서와 신뢰 체인: 서버를 믿는 근거

TLS가 하는 일은 본질적으로 두 가지다. 하나는 (1) 키 교환으로 대칭키를 합의해 이후 트래픽을 암호화하는 것이고, 다른 하나는 (2) **인증서**로 상대가 주장하는 신원을 검증하는 것이다. gRPC 보안 실무에서 사람을 가장 많이 괴롭히는 부분은 (2)의 검증 로직이다.

### 2.1 PKI와 신뢰 체인

X.509 인증서는 "이 공개키는 이 신원(주체, subject)의 것이다"라는 내용을 **인증 기관(CA, Certificate Authority)이 서명으로 보증**한 문서다. 클라이언트는 서버 인증서를 받으면 다음 항목을 검증한다.

```
        신뢰 체인 (chain of trust)

   [Root CA 인증서] (자기서명, self-signed)
        │  서명
        ▼
   [Intermediate CA 인증서]
        │  서명
        ▼
   [서버 leaf 인증서]  ← 서버가 핸드셰이크에서 제시
        - Subject / SAN: api.example.com
        - 유효기간: 2026-01-01 ~ 2026-04-01
        - 공개키

   클라이언트의 검증:
   1) leaf의 서명을 intermediate 공개키로 검증
   2) intermediate의 서명을 root 공개키로 검증
   3) root가 "내 신뢰 저장소(trust store)"에 있는가?
   4) 각 인증서가 유효기간 내인가? 폐기(revoke)되지 않았는가?
   5) leaf의 SAN이 내가 접속하려는 호스트명과 일치하는가?  ← 자주 빠지는 단계
```

다섯 단계 가운데 1~4는 이 인증서가 실제 CA가 발급한 유효한 인증서인지를 확인한다. 그러나 **5번 호스트네임 검증이 빠지면 보안 전체가 무너진다.** 공격자도 어떤 CA에서든 "evil.attacker.com"용으로 완벽하게 유효한 인증서를 발급받을 수 있기 때문이다. 1~4만 확인하면, 유효하지만 다른 호스트용인 그 인증서로 중간자 공격이 성립한다. 5번은 이 유효한 인증서가 **바로 내가 접속하려던 그 서버의 것인지**를 확인하는 마지막 단계다.

### 2.2 SAN과 호스트네임 검증 (CN은 더 이상 쓰지 않는다)

호스트네임 검증 규칙(RFC 6125, 2023년에 이를 대체한 RFC 9525, 그리고 RFC 2818의 갱신)은 명확하다.

- 검증 대상은 인증서의 **SAN(Subject Alternative Name)** 확장에 들어 있는 `dNSName` 항목이다.
- **Common Name(CN)은 더 이상 호스트네임 매칭에 쓰지 않는다.** 과거에는 CN을 fallback으로 확인했지만, 현대 클라이언트(브라우저, Go 1.15 이상 등)는 SAN이 없으면 그냥 실패로 처리한다. Go는 1.15부터 CN fallback을 완전히 제거했다.
- 와일드카드 `*.example.com`은 **레이블 하나**에만 매치된다. `a.example.com`은 매치되지만 `a.b.example.com`은 매치되지 않는다.

따라서 인증서를 만들 때는 반드시 SAN을 넣어야 한다. openssl로 SAN이 들어간 서버 인증서를 만드는 예시는 뒤의 mTLS 절에서 함께 다룬다. 여기서는 검증이 SAN을 확인한다는 사실만 기억해 두자.

gRPC에서 흔히 겪는 함정이 있다. 인증서의 SAN은 `api.example.com`인데 클라이언트가 IP `10.0.0.5:443`으로 직접 연결하면 호스트네임 검증이 실패한다. 해결책은 두 가지다. (a) 인증서 SAN에 IP를 `iPAddress` 항목으로 추가하거나, (b) 클라이언트에서 이 연결의 검증용 권위(authority)를 `api.example.com`으로 명시적으로 오버라이드한다(Go의 `tls.Config.ServerName`, gRPC의 `WithAuthority` 등).

[[12 - 이름 해석과 로드밸런싱]]에서 다루는 이름 해석도 이 문제와 깊이 얽혀 있다. 이름 해석은 타깃 호스트명을 IP로 바꾸는 과정인데, 그 과정에서 검증용 권위를 어떻게 유지하느냐가 관건이기 때문이다.

### 2.3 gRPC의 보안 수준(security level) 개념

gRPC 내부(특히 C-core 기반 구현)는 채널의 보안 상태를 **security level**이라는 단계로 추상화한다.

| security level          | 의미                                |
| ----------------------- | --------------------------------- |
| `NONE`                  | 보안 없음(평문). insecure 채널.           |
| `INTEGRITY_ONLY`        | 변조 방지(무결성)는 되나 암호화(기밀성)는 안 됨. 드묾. |
| `PRIVACY_AND_INTEGRITY` | 기밀성 + 무결성 모두. 정상적인 TLS/mTLS/ALTS. |

이 개념이 중요한 이유는 **채널의 security level이 `PRIVACY_AND_INTEGRITY`일 때만 호출 자격증명(토큰)을 보낼 수 있도록** gRPC가 설계되어 있기 때문이다(자세한 내용은 5절). 즉 라이브러리가 이 관이 충분히 안전한지를 타입과 런타임 수준에서 따져서, 안전하지 않은 관으로 토큰이 새어 나가는 사고를 구조적으로 막는다.

---

## 3. 채널 자격증명의 종류: 언제 무엇을 쓰는가

채널 자격증명은 이 연결을 어떻게 보호할지에 대한 선택지다. gRPC는 몇 가지 표준 종류를 제공한다.

```
   채널 자격증명 선택 트리

   서버가 외부(인터넷)에 노출되는가?
     ├─ 예 → TLS (서버 인증서) 필수
     │        클라이언트도 인증해야 하나(제로 트러스트)?
     │          ├─ 예 → mTLS
     │          └─ 아니오 → 일반 TLS + (위에) 토큰 콜 자격증명
     │
     └─ 아니오 (클러스터 내부 서비스 간)
              ├─ 서비스 메시(Istio/Linkerd 등) 사이드카가 mTLS 처리?
              │     └─ 예 → 앱은 local/insecure, 메시가 mTLS 자동 적용
              ├─ 구글 클라우드 환경(GCE/GKE) 내부?
              │     └─ ALTS 고려
              └─ 직접 mTLS 운영 → mTLS
```

### 3.1 TLS 채널 자격증명

가장 기본적인 방식이다. 서버는 인증서와 개인키를 가지고, 클라이언트는 신뢰하는 CA 목록(trust roots)을 가진다. 서버를 검증하는 쪽은 클라이언트뿐이다(단방향). 인터넷에 노출되는 API의 기본 형태다.

### 3.2 mTLS (상호 TLS) 채널 자격증명

서버뿐 아니라 **클라이언트도 자기 인증서를 제시**한다. 서버는 클라이언트 인증서를 검증해서, 이 연결 반대편이 누구인지를 TLS 핸드셰이크 단계에서 확정한다. 서비스 간(service-to-service) 통신과 제로 트러스트 네트워크에서 표준으로 쓰인다. 자세한 내용은 4절에서 다룬다.

### 3.3 insecure (평문)

암호화도 인증도 없으며, security level은 `NONE`이다. **프로덕션에서 외부로 노출해서는 절대 안 된다.** 정당한 용도는 두 가지뿐이다.

- **로컬 개발과 테스트**: `localhost`에서 빠르게 실행해 볼 때 쓴다.
- **사이드카나 메시 뒤의 로컬 홉**: Istio 같은 서비스 메시에서 사이드카 프록시(Envoy)가 mTLS를 대신 처리하는 경우, 앱과 사이드카 사이의 `127.0.0.1` 루프백 구간은 외부로 나가지 않으므로 평문이어도 허용된다. 이때도 사이드카가 실제로 mTLS를 강제하는지(STRICT 모드)를 반드시 확인해야 한다.

### 3.4 ALTS (Application Layer Transport Security)

구글이 설계한 상호 인증·암호화 프로토콜로, 구글 인프라(GCE/GKE) 내부에서 쓴다. 목적은 TLS와 비슷하지만 구글 환경의 신원(서비스 계정)과 묶여서 동작하므로, 구글 클라우드 밖에서는 의미가 없다. 이런 것이 있다는 정도만 알아 두면 된다.

### 3.5 local

`local` 채널 자격증명은 루프백(`localhost`/UDS) 연결에 한해 인증을 생략하는 특수 자격증명이다. 사이드카 패턴에서 앱과 로컬 프록시 사이의 통신을 다룰 때 쓴다. insecure와 비슷하지만, 연결이 정말 로컬인지를 확인한다는 점에서 조금 더 안전하다.

### 비교표

| 종류 | 암호화 | 서버 인증 | 클라이언트 인증 | 주 용도 |
|------|:---:|:---:|:---:|------|
| TLS | O | O | X | 외부 노출 API |
| mTLS | O | O | O | 서비스 간, 제로 트러스트 |
| insecure | X | X | X | 로컬 개발, 메시 내부 홉 |
| ALTS | O | O | O | 구글 클라우드 내부 |
| local | (루프백) | (생략) | (생략) | 사이드카 로컬 홉 |

---

## 4. mTLS: 양쪽이 서로를 증명한다

### 4.1 일반 TLS와 다른 점

핸드셰이크 수준에서 보면 mTLS는 일반 TLS에 **CertificateRequest**와 **클라이언트 Certificate / CertificateVerify**를 더한 형태다.

```
  일반 TLS (단방향)                          mTLS (양방향)
  ─────────────────                         ─────────────
  C → ClientHello                            C → ClientHello
  S → ServerHello                            S → ServerHello
  S → Certificate (서버 인증서)               S → Certificate (서버 인증서)
                                             S → CertificateRequest  ★ "너도 인증서 내놔"
  S → ServerHelloDone/Finished               S → (Hello 완료 신호)
  C   서버 검증                                C   서버 검증
                                             C → Certificate (클라 인증서) ★
                                             C → CertificateVerify         ★ "이 키 내가 가졌음을 서명으로 증명"
  C → Finished                               C → Finished
  S → Finished                               S   클라 인증서 검증 후
                                             S → Finished
  ──────────────────────────────────────────────────────────────────
  결과: 클라가 서버를 안다                       결과: 서로가 서로를 안다
```

여기서 핵심은 `CertificateVerify`다. 클라이언트가 인증서(공개키)만 제시해서는 그 공개키에 대응하는 개인키를 실제로 가지고 있는지 증명되지 않는다. 인증서는 공개 정보라서 복사할 수 있기 때문이다.

`CertificateVerify`는 클라이언트가 지금까지 오간 핸드셰이크 메시지에 **자기 개인키로 서명**한 값이므로, 개인키를 가지고 있다는 사실을 증명한다. 서버는 이 서명을 클라이언트 인증서의 공개키로 검증해서, 상대가 인증서를 복사한 것이 아니라 실제 개인키 소유자임을 확인한다.

### 4.2 mTLS가 필요한 이유: 네트워크 위치는 신원이 아니다

전통적인 보안은 내부 네트워크 안에 있으면 믿는 방식이었다(성벽 모델). 그러나 클라우드와 컨테이너 환경에서는 같은 VPC, 같은 클러스터 안에서도 수백 개의 서비스가 실행되고, 파드 하나가 뚫리면 내부 전체가 위험해진다. **제로 트러스트(zero trust)** 의 핵심 명제는 "네트워크 위치는 신원이 아니다"이다. 호출이 IP 10.x 대역에서 왔다는 사실만으로는 그 호출자가 결제 서비스라는 증거가 되지 못한다.

mTLS는 이 문제를 직접 해결한다. 모든 서비스가 자기 인증서(즉 암호학적 신원)를 가지고, 연결할 때마다 서로 그 신원을 증명한다. 인증의 근거는 "10.0.3.7에서 왔다"가 아니라 "인증서 SAN이 `spiffe://cluster.local/ns/payments/sa/charger`인 워크로드가 왔다"가 된다.

### 4.3 인증서 한 벌 만들기 (openssl 실습)

mTLS를 직접 구성해 보려면 (1) CA, (2) SAN이 들어간 서버 인증서, (3) 클라이언트 인증서가 필요하다.

```bash
# 1) Root CA 키와 자기서명 인증서
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/CN=Example Internal Root CA" \
  -out ca.crt

# 2) 서버 키 + CSR
openssl genrsa -out server.key 2048
openssl req -new -key server.key \
  -subj "/CN=api.example.com" \
  -out server.csr

# 2-1) 서버 인증서에 SAN을 넣어 CA로 서명  (★ SAN 필수)
cat > server-ext.cnf <<'EOF'
subjectAltName = DNS:api.example.com, DNS:localhost, IP:127.0.0.1
extendedKeyUsage = serverAuth
EOF
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 365 -sha256 -extfile server-ext.cnf -out server.crt

# 3) 클라이언트 키 + CSR + 인증서 (clientAuth EKU가 핵심)
openssl genrsa -out client.key 2048
openssl req -new -key client.key \
  -subj "/CN=order-service" \
  -out client.csr
cat > client-ext.cnf <<'EOF'
subjectAltName = URI:spiffe://example.com/ns/orders/sa/order-service
extendedKeyUsage = clientAuth
EOF
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 365 -sha256 -extfile client-ext.cnf -out client.crt

# 검증: 인증서 내용 확인
openssl x509 -in server.crt -noout -text | grep -A1 "Subject Alternative Name"
```

여기서 EKU(Extended Key Usage)에 주목하자. 서버 인증서에는 `serverAuth`를, 클라이언트 인증서에는 `clientAuth`를 넣었다. mTLS에서는 이 구분이 중요하다. 클라이언트 인증서의 SAN에는 `URI:spiffe://...`를 넣었는데, 이것이 8절에서 다룰 SPIFFE 신원이다.

### 4.4 Go로 보는 단방향 TLS와 mTLS

먼저 일반 TLS 서버와 클라이언트를 보자.

```go
// ===== 서버: 일반 TLS =====
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
)

func newTLSServer() *grpc.Server {
    // 서버 인증서 + 개인키 로드
    creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
    if err != nil {
        log.Fatalf("load server cert: %v", err)
    }
    return grpc.NewServer(grpc.Creds(creds))
}

// ===== 클라이언트: 일반 TLS =====
func dialTLS(target string) (*grpc.ClientConn, error) {
    // 신뢰하는 CA(루트)로 서버를 검증
    creds, err := credentials.NewClientTLSFromFile("ca.crt", "api.example.com")
    //                                                          ^^^^^^^^^^^^^^^^
    //               두 번째 인자 = 검증에 사용할 서버 호스트명(SAN 매칭 대상)
    if err != nil {
        return nil, err
    }
    return grpc.NewClient(target, grpc.WithTransportCredentials(creds))
}
```

`NewClientTLSFromFile`의 두 번째 인자가 바로 2.2절에서 설명한 호스트네임 검증 대상을 지정하는 자리다. IP로 연결하더라도 이 인자를 `api.example.com`으로 주면 SAN을 그 이름으로 검증한다.

이제 mTLS를 보자. 차이는 두 가지다. (1) 서버가 클라이언트 인증서를 **요구**하고 검증하도록 `tls.Config`를 직접 구성하고, (2) 클라이언트가 자기 인증서를 함께 제시한다.

```go
// ===== 서버: mTLS =====
import (
    "crypto/tls"
    "crypto/x509"
    "os"
)

func newMTLSServer() (*grpc.Server, error) {
    // 서버 자신의 인증서/키
    serverCert, err := tls.LoadX509KeyPair("server.crt", "server.key")
    if err != nil {
        return nil, err
    }
    // 클라이언트 인증서를 검증할 CA 풀
    caPEM, err := os.ReadFile("ca.crt")
    if err != nil {
        return nil, err
    }
    clientCAs := x509.NewCertPool()
    if !clientCAs.AppendCertsFromPEM(caPEM) {
        return nil, fmt.Errorf("failed to add client CA")
    }

    tlsCfg := &tls.Config{
        Certificates: []tls.Certificate{serverCert},
        ClientCAs:    clientCAs,
        // ★ 핵심: 클라이언트 인증서를 "요구하고 검증"
        ClientAuth:   tls.RequireAndVerifyClientCert,
        MinVersion:   tls.VersionTLS12, // HTTP/2 h2 최소 요건
    }
    creds := credentials.NewTLS(tlsCfg)
    return grpc.NewServer(grpc.Creds(creds)), nil
}
```

mTLS의 강도는 `tls.ClientAuth` 값에 따라 달라진다.

| 값 | 동작 |
|----|------|
| `NoClientCert` | 클라 인증서 요청 안 함 (일반 TLS) |
| `RequestClientCert` | 요청은 하지만 없어도/검증 실패해도 통과 (위험) |
| `RequireAnyClientCert` | 인증서 요구하나 신뢰 체인 검증은 안 함 (위험) |
| `VerifyClientCertIfGiven` | 줬으면 검증, 안 줘도 통과 (점진 마이그레이션용) |
| `RequireAndVerifyClientCert` | **요구 + 검증. 진짜 mTLS** |

```go
// ===== 클라이언트: mTLS =====
func dialMTLS(target string) (*grpc.ClientConn, error) {
    clientCert, err := tls.LoadX509KeyPair("client.crt", "client.key")
    if err != nil {
        return nil, err
    }
    caPEM, _ := os.ReadFile("ca.crt")
    rootCAs := x509.NewCertPool()
    rootCAs.AppendCertsFromPEM(caPEM)

    tlsCfg := &tls.Config{
        Certificates: []tls.Certificate{clientCert}, // ★ 내 인증서 제시
        RootCAs:      rootCAs,                        // 서버 검증용 루트
        ServerName:   "api.example.com",              // ★ SAN 매칭 대상
        MinVersion:   tls.VersionTLS12,
    }
    creds := credentials.NewTLS(tlsCfg)
    return grpc.NewClient(target, grpc.WithTransportCredentials(creds))
}
```

핸드셰이크가 끝나면 서버는 RPC 핸들러에서 검증된 클라이언트 인증서의 정보(SAN, CN 등)를 꺼내 쓸 수 있다. 추가 작업 없이 mTLS가 제공하는 인증 정보다.

```go
import "google.golang.org/grpc/peer"

func peerIdentity(ctx context.Context) (string, error) {
    p, ok := peer.FromContext(ctx)
    if !ok {
        return "", fmt.Errorf("no peer info")
    }
    tlsInfo, ok := p.AuthInfo.(credentials.TLSInfo)
    if !ok {
        return "", fmt.Errorf("not a TLS connection")
    }
    chains := tlsInfo.State.VerifiedChains
    if len(chains) == 0 || len(chains[0]) == 0 {
        return "", fmt.Errorf("no verified client cert")
    }
    leaf := chains[0][0]
    // SAN의 URI(SPIFFE ID)나 DNS 이름, 또는 CN을 신원으로 사용
    if len(leaf.URIs) > 0 {
        return leaf.URIs[0].String(), nil // e.g. spiffe://example.com/ns/orders/sa/order-service
    }
    return leaf.Subject.CommonName, nil
}
```

`peerIdentity`가 반환하는 SPIFFE ID나 CN이 곧 이 호출자가 어떤 서비스인지에 대한 인증 결과다. 인가(authorization)는 이 값을 받아 정책을 검사한다(6절, 7절).

### 4.5 Java와 Python의 형태

언어마다 API는 다르지만 개념은 같다.

```java
// Java (Netty 기반): mTLS 서버
import io.grpc.netty.shaded.io.grpc.netty.GrpcSslContexts;
import io.grpc.netty.shaded.io.netty.handler.ssl.ClientAuth;
import io.grpc.netty.shaded.io.netty.handler.ssl.SslContext;

SslContext sslContext = GrpcSslContexts
    .forServer(new File("server.crt"), new File("server.key"))
    .trustManager(new File("ca.crt"))      // 클라 인증서 검증용 CA
    .clientAuth(ClientAuth.REQUIRE)        // ★ mTLS 강제
    .build();

Server server = NettyServerBuilder.forPort(8443)
    .sslContext(sslContext)
    .addService(new ChatServiceImpl())
    .build();
```

```python
# Python: mTLS 서버
import grpc

with open("server.key", "rb") as f: server_key = f.read()
with open("server.crt", "rb") as f: server_crt = f.read()
with open("ca.crt", "rb") as f:     ca_crt = f.read()

server_credentials = grpc.ssl_server_credentials(
    [(server_key, server_crt)],
    root_certificates=ca_crt,            # 클라 인증서 검증용
    require_client_auth=True,            # ★ mTLS 강제
)
server = grpc.server(futures.ThreadPoolExecutor())
server.add_secure_port("[::]:8443", server_credentials)
```

```python
# Python: mTLS 클라이언트
channel_credentials = grpc.ssl_channel_credentials(
    root_certificates=ca_crt,            # 서버 검증용 루트
    private_key=client_key,              # ★ 내 개인키
    certificate_chain=client_crt,        # ★ 내 인증서
)
channel = grpc.secure_channel("api.example.com:8443", channel_credentials)
```

---

## 5. 호출 자격증명: 요청마다 주체 정보를 싣기

채널 자격증명이 관을 보호한다면, 호출 자격증명(call credentials)은 관 속을 흐르는 **요청 단위 신분증**이다. mTLS는 요청이 어느 서비스(워크로드)에서 왔는지를 증명하는 데 강하고, 토큰은 보통 어느 사용자나 앱이 이 요청의 주체인지를 전달한다. 둘은 서로 보완한다.

### 5.1 토큰은 메타데이터로 간다

호출 자격증명의 실체는 단순하다. 각 RPC가 나갈 때 **메타데이터(HTTP/2 헤더)에 `authorization` 키를 추가**하는 훅(hook)이다(8장 메타데이터와 인터셉터). 가장 흔한 형식은 OAuth2 Bearer 토큰이다.

```text
HEADERS 프레임 안 (TLS로 암호화됨):
  :method: POST
  :path: /todo.TodoService/CreateTask
  content-type: application/grpc
  authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImFiYzEy...   ← JWT
```

JWT(JSON Web Token)는 `header.payload.signature` 세 부분을 각각 base64url로 인코딩해 점(.)으로 이은 것이다. payload에는 보통 `sub`(주체), `exp`(만료), `aud`(대상), `scope`/`scp`(권한 범위) 같은 클레임이 들어간다. 서버는 이 토큰의 서명을 발급자(IdP)의 공개키로 검증해서, 토큰이 위조되지 않았고 만료되지 않았으며 이 서버를 대상으로 발급된 것인지 확인한다.

### 5.2 Go: PerRPCCredentials 구현

gRPC는 호출 자격증명을 `PerRPCCredentials` 인터페이스로 추상화한다. 핵심 메서드는 두 개다.

```go
type PerRPCCredentials interface {
    // 각 RPC마다 호출되어, 메타데이터(헤더)에 넣을 키-값을 반환
    GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error)
    // 이 자격증명이 "전송 보안"을 요구하는가? true면 평문 채널에서 거부됨
    RequireTransportSecurity() bool
}
```

가장 단순한 형태는 정적(static) 토큰 자격증명이다.

```go
type bearerToken struct {
    token string
}

func (b bearerToken) GetRequestMetadata(ctx context.Context, _ ...string) (map[string]string, error) {
    return map[string]string{
        "authorization": "Bearer " + b.token,
    }, nil
}

// ★ true를 반환하면, 이 자격증명은 보안 채널에서만 동작.
//    insecure 채널에 붙이면 gRPC가 런타임 에러로 거부한다.
func (b bearerToken) RequireTransportSecurity() bool {
    return true
}

// 사용: 채널 전체에 적용
conn, _ := grpc.NewClient(
    "api.example.com:443",
    grpc.WithTransportCredentials(tlsCreds),          // 채널 자격증명 (TLS)
    grpc.WithPerRPCCredentials(bearerToken{token: t}),// 호출 자격증명 (토큰)
)
```

**특정 호출에만** 붙이고 싶다면 `CallOption`으로 넘긴다.

```go
resp, err := client.CreateTask(ctx, req,
    grpc.PerRPCCredentials(bearerToken{token: perCallToken}))
```

### 5.3 토큰 갱신(refresh): 자격증명을 계속 유효하게 유지해야 하는 이유

정적 토큰은 만료된다. 실제 운영에 쓰는 자격증명은 **만료를 감지하고 갱신**할 수 있어야 한다. `GetRequestMetadata`가 RPC마다 호출된다는 점을 이용해, 그 안에서 토큰이 만료되지 않았는지 관리한다.

```go
type oauthSource struct {
    mu        sync.Mutex
    token     string
    expiresAt time.Time
    fetch     func(ctx context.Context) (string, time.Time, error) // IdP에서 새 토큰 발급
}

func (s *oauthSource) GetRequestMetadata(ctx context.Context, _ ...string) (map[string]string, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    // 만료 60초 전이면 미리 갱신 (시계 오차/지연 대비 여유분)
    if time.Now().After(s.expiresAt.Add(-60 * time.Second)) {
        tok, exp, err := s.fetch(ctx)
        if err != nil {
            return nil, status.Errorf(codes.Unauthenticated, "token refresh failed: %v", err)
        }
        s.token, s.expiresAt = tok, exp
    }
    return map[string]string{"authorization": "Bearer " + s.token}, nil
}

func (s *oauthSource) RequireTransportSecurity() bool { return true }
```

여기서 **60초의 여유분(skew margin)** 이 실무에서 중요하다. 토큰의 `exp`가 가리키는 만료 시각에 맞춰 갱신을 시작하면, 발급 지연과 네트워크 왕복, 서버와의 시계 오차(clock skew) 때문에 이미 만료된 토큰을 보내게 되고 인증이 실패한다. 만료되기 전에 미리 갱신하면 이런 경계 상황을 피할 수 있다.

시계 오차는 9절의 함정에서 더 다룬다. 또한 IdP 호출에 `ctx`를 그대로 전달하므로 [[10 - Deadline 취소 타임아웃]]에서 다룬 deadline이 갱신 단계까지 전파되고, 토큰 발급이 무한정 멈춰 있지 않는다.

구글 환경에서는 ADC(Application Default Credentials)가 이 갱신 로직을 대신 처리한다. 환경 변수, 메타데이터 서버, 서비스 계정 키 파일에서 자격증명을 찾아 토큰을 발급하고 갱신한 뒤 메타데이터에 자동으로 붙인다. `fetch`를 직접 구현할 필요 없이 라이브러리가 처리한다.

### 5.4 평문 채널 위의 토큰을 금지하는 이유

이 장에서 가장 중요한 보안 원칙 가운데 하나다. 여기서는 `RequireTransportSecurity()`가 기본적으로 `true`인 이유와, insecure 채널에 토큰 자격증명을 붙이면 gRPC가 런타임에 거부하는 이유를 설명한다.

```
    평문(insecure) 채널 위에 Bearer 토큰을 보내면?

    클라이언트 ───[ TCP, 암호화 없음 ]──── 서버
                       │
                  중간자(스위치/프록시/와이파이/탭)
                       │
              authorization: Bearer eyJhbGci...  ← 그대로 읽힌다!
                       │
              공격자가 토큰을 복사 → 그대로 재사용(replay)
              → 토큰 만료 전까지 "그 사용자인 척" 모든 요청 가능
```

Bearer 토큰은 정의부터가 "이 토큰을 **소지한(bearer) 자**는 누구든 그 권한을 가진다"이다. 현금과 같은 성격이다. 그래서 평문으로 보내는 순간 도청자가 바로 그 신원을 도용할 수 있다.

mTLS의 클라이언트 인증서는 핸드셰이크마다 개인키 소유를 증명(CertificateVerify)해야 하므로 복사해도 쓸 수 없지만, Bearer 토큰은 복사하면 그대로 쓸 수 있다. 이 차이 때문에 토큰은 **반드시 암호화된 관(TLS, security level `PRIVACY_AND_INTEGRITY`) 위에서만** 보내야 한다.

gRPC는 이 규칙을 라이브러리에 내장해 두었다. 그래서 토큰 자격증명과 insecure 채널을 함께 쓰려고 하면 연결 시점이나 호출 시점에 에러를 낸다. 개발자가 실수로 토큰을 평문으로 보내지 못하게 막는 **구조적 안전장치**다. 테스트 목적으로 `RequireTransportSecurity()`가 `false`를 반환하게 만들 수는 있지만, 프로덕션에서 그렇게 하면 위 그림의 상황을 스스로 만드는 셈이다.

---

## 6. 자격증명 합성(composite): TLS 관 + 요청별 토큰

이제 두 축을 합쳐 보자. 표준 패턴은 mTLS나 TLS로 보호된 채널 위에 per-RPC 토큰을 올리는 것이다. gRPC에서는 이를 채널 자격증명과 호출 자격증명을 **합성(composite)** 한다고 표현한다.

```
   CompositeChannelCredentials
   ────────────────────────────
        TransportCredentials (TLS/mTLS)   ← 관을 보호
                   +
        CallCredentials (Bearer 토큰)     ← 요청마다 신원

   결과: 하나의 채널에
     - 연결은 TLS/mTLS로 암호화·(상호)인증
     - 모든 RPC에 authorization 헤더 자동 부착
```

Go에서는 연결 옵션 두 개를 함께 넘기는 것 자체가 합성이다.

```go
conn, err := grpc.NewClient(
    "api.example.com:443",
    grpc.WithTransportCredentials(tlsCreds),            // 채널 축
    grpc.WithPerRPCCredentials(&oauthSource{...}),      // 콜 축
)
```

C++를 비롯한 일부 언어에는 `CompositeChannelCredentials(channelCreds, callCreds)`라는 명시적인 합성 API가 있으며, 의미는 같다. 합성할 때 gRPC는 호출 자격증명이 요구하는 security level(보통 `PRIVACY_AND_INTEGRITY`)을 채널 자격증명이 만족하는지 검사한다. mTLS와 TLS는 이 조건을 만족하므로 통과하고, insecure는 만족하지 못하므로 거부된다(5.4절).

이 합성 패턴이 실무의 표준이 된 이유는 두 자격증명이 서로 다른 정보를 보장하기 때문이다.

- **mTLS**는 이 호출을 보낸 워크로드(서비스 A의 파드)가 실제로 서비스 A인지를 보증한다.
- **토큰**은 이 요청의 최종 주체(최종 사용자 `alice`, 또는 권한을 위임받은 앱)가 누구인지를 전달한다.

서비스 A가 사용자 alice의 요청을 받아 서비스 B를 호출한다고 하자. B는 mTLS로 A가 호출했다는 사실을 알고, 토큰으로 원래 주체가 alice라는 사실을 안다. 두 정보를 합쳐야 A가 alice를 대신해 정당하게 B를 호출한다는 사실 전체가 확인된다.

---

## 7. 인증(authn)과 인가(authz)의 분리

지금까지 다룬 내용은 모두 **인증(authentication, "누구인가")** 이었다. TLS/mTLS로 워크로드 신원을 확인하고, 토큰으로 주체 신원을 확인했다. 그러나 누구인지 아는 것과 그 작업을 해도 되는지는 별개의 문제다. 후자가 **인가(authorization, "무엇을 해도 되는가")** 이며, gRPC에서는 보통 **서버 측 인터셉터**가 정책으로 처리한다(8장 메타데이터와 인터셉터).

```
   한 RPC가 서버에 도착해서 핸들러까지 가는 길

   ┌──────────────────────────────────────────────────────────┐
   │ TLS/mTLS 종료: 채널 신원 확립 (peer 인증서 → 워크로드 신원)    │  ← 인증 1
   └──────────────────────────────────────────────────────────┘
                          │
   ┌──────────────────────────────────────────────────────────┐
   │ 인증 인터셉터: authorization 헤더의 토큰 검증                  │  ← 인증 2
   │   - 서명 검증, exp/aud/iss 검사 → principal + scopes 추출     │
   │   - 실패 시 codes.Unauthenticated (16)                       │
   └──────────────────────────────────────────────────────────┘
                          │  ctx에 principal/scopes 주입
   ┌──────────────────────────────────────────────────────────┐
   │ 인가 인터셉터: "이 principal이 이 메서드를 호출해도 되는가"     │  ← 인가
   │   - 메서드별 필요한 scope/role 매핑 검사 (RBAC)              │
   │   - 실패 시 codes.PermissionDenied (7)                      │
   └──────────────────────────────────────────────────────────┘
                          │
                    [ 비즈니스 핸들러 ]
```

이때 어떤 상태 코드를 쓰는지가 중요하다([[09 - 에러 모델 - 상태 코드와 Rich Error]]).

- **`UNAUTHENTICATED` (16)**: 호출자가 누구인지 알 수 없거나, 토큰이 없거나 위조되었거나 만료된 경우다. 신원 확인에 실패했다는 뜻이다.
- **`PERMISSION_DENIED` (7)**: 호출자가 누구인지는 알지만 이 작업을 할 권한이 없는 경우다. 신원은 확인되었고 권한이 부족하다는 뜻이다.

두 코드를 섞어 쓰면 클라이언트는 토큰을 갱신해야 하는지(HTTP 401에 해당), 권한을 요청해야 하는지(HTTP 403에 해당) 구분할 수 없다.

### 7.1 Go: 인증 + 인가 인터셉터

```go
import (
    "context"
    "strings"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/status"
)

// 검증된 주체를 ctx에 싣기 위한 키
type principalKey struct{}

type Principal struct {
    Subject string   // 예: "user:alice"
    Scopes  []string // 예: ["task:read", "task:write"]
}

// ── 인증 인터셉터: 토큰 검증 → Principal 추출 ──
func authInterceptor(verify func(token string) (*Principal, error)) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (any, error) {

        md, ok := metadata.FromIncomingContext(ctx)
        if !ok {
            return nil, status.Error(codes.Unauthenticated, "missing metadata")
        }
        auth := md.Get("authorization")
        if len(auth) == 0 {
            return nil, status.Error(codes.Unauthenticated, "missing authorization header")
        }
        // "Bearer xxx"에서 토큰만 추출
        const prefix = "Bearer "
        if !strings.HasPrefix(auth[0], prefix) {
            return nil, status.Error(codes.Unauthenticated, "invalid authorization scheme")
        }
        token := strings.TrimPrefix(auth[0], prefix)

        p, err := verify(token) // 서명/exp/aud/iss 검증은 verify 안에서
        if err != nil {
            // ★ 토큰 값 자체는 절대 로그/에러 메시지에 넣지 않는다 (9절 함정 참고)
            return nil, status.Error(codes.Unauthenticated, "invalid token")
        }
        ctx = context.WithValue(ctx, principalKey{}, p)
        return handler(ctx, req)
    }
}

// ── 인가 인터셉터: 메서드별 필요한 스코프 검사 (RBAC) ──
var methodScopes = map[string]string{
    "/todo.TodoService/CreateTask": "task:write",
    "/todo.TodoService/ListTasks":  "task:read",
    "/todo.TodoService/DeleteTask": "task:admin",
}

func authzInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {

    required, guarded := methodScopes[info.FullMethod]
    if !guarded {
        return handler(ctx, req) // 보호 대상이 아닌 메서드는 통과
    }
    p, ok := ctx.Value(principalKey{}).(*Principal)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "no principal")
    }
    if !hasScope(p.Scopes, required) {
        return nil, status.Errorf(codes.PermissionDenied,
            "missing required scope %q", required)
    }
    return handler(ctx, req)
}

func hasScope(scopes []string, want string) bool {
    for _, s := range scopes {
        if s == want {
            return true
        }
    }
    return false
}

// ── 서버 조립: 인증 → 인가 순서로 체이닝 ──
server := grpc.NewServer(
    grpc.Creds(mtlsCreds),
    grpc.ChainUnaryInterceptor(
        authInterceptor(verifyJWT), // 먼저 "누구인가"
        authzInterceptor,           // 그 다음 "해도 되는가"
    ),
)
```

순서에는 의미가 있다. 인증이 먼저 실행되어 `Principal`을 ctx에 넣고, 인가가 그 값을 읽어 정책을 적용한다. 인터셉터 체이닝의 동작 방식은 8장(메타데이터와 인터셉터)에서 다룬다.

### 7.2 mTLS 신원과 토큰 신원을 함께 확인하는 인가

더 정교한 인가는 **두 신원을 교차 검증**한다. 예를 들어 서비스 A만 이 내부 메서드를 호출할 수 있게 하고(워크로드 인가, mTLS 기반), 그 안에서 토큰 주체는 자기 리소스만 다룰 수 있게 하는(주체 인가, 토큰 기반) 식이다.

```go
func adminOnlyByWorkload(ctx context.Context, req any, info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {

    // mTLS 피어 신원 (4.4의 peerIdentity 재사용)
    callerSPIFFE, err := peerIdentity(ctx)
    if err != nil {
        return nil, status.Error(codes.Unauthenticated, "no workload identity")
    }
    // 이 내부 메서드는 'order-service' 워크로드만 허용
    if callerSPIFFE != "spiffe://example.com/ns/orders/sa/order-service" {
        return nil, status.Errorf(codes.PermissionDenied,
            "workload %s not allowed", callerSPIFFE)
    }
    return handler(ctx, req)
}
```

이처럼 **채널 축(mTLS 워크로드)** 과 **호출 축(토큰 주체)** 의 인가를 서로 다른 인터셉터로 나누면 정책을 깔끔하게 계층화할 수 있다.

---

## 8. 서비스 메시, SPIFFE/SVID, 그리고 mTLS의 자동화

지금까지 다룬 mTLS는 인증서를 직접 만들고 배포하고 갱신한다는 전제였다. 서비스가 5개라면 감당할 만하지만 500개라면 감당하기 어렵다. 인증서 발급과 배포, 회전을 수동으로 하면 반드시 어딘가에서 만료 사고가 난다. 그래서 등장한 것이 **서비스 메시(service mesh)** 와 **SPIFFE/SVID** 같은 워크로드 신원 표준이다.

### 8.1 SPIFFE: 워크로드에 보편적인 신원 부여하기

SPIFFE(Secure Production Identity Framework For Everyone)는 모든 워크로드에 암호학적으로 검증할 수 있는 보편적인 신원을 부여하자는 표준이다. 핵심 개념은 두 가지다.

- **SPIFFE ID**: `spiffe://<trust-domain>/<path>` 형식의 URI다. 예를 들어 `spiffe://example.com/ns/orders/sa/order-service`처럼 쓰며, 이것이 워크로드의 이름이 된다.
- **SVID(SPIFFE Verifiable Identity Document)**: SPIFFE ID를 담은 검증 가능한 문서다. **X.509-SVID**는 X.509 인증서의 SAN URI 필드에 SPIFFE ID를 넣은 것이다(4.3의 openssl 예시에서 `URI:spiffe://...`를 넣은 것이 바로 이것이다). JWT-SVID도 있다.

```
   X.509-SVID = 그냥 X.509 인증서인데
                SAN의 URI 항목 = SPIFFE ID

   Subject Alternative Name:
       URI:spiffe://example.com/ns/orders/sa/order-service
                    └── 이게 곧 워크로드의 신원 ──┘
```

mTLS 핸드셰이크에서 클라이언트가 X.509-SVID를 제시하면, 서버는 SAN의 SPIFFE ID를 보고 어떤 워크로드인지 바로 안다. 4.4절의 `peerIdentity`가 `leaf.URIs[0]`에서 꺼낸 값이 바로 이 SPIFFE ID다. 7.2절의 인가는 이 SPIFFE ID를 기준으로 워크로드 단위 정책을 적용한다.

### 8.2 사이드카가 mTLS를 대신 처리한다

Istio나 Linkerd 같은 메시에서는 파드마다 **사이드카 프록시(보통 Envoy)** 가 붙는다. mTLS에서 부담이 큰 작업(인증서 발급, 핸드셰이크, 회전)을 이 프록시가 대신 처리한다.

```
   서비스 A 파드                          서비스 B 파드
   ┌──────────────────┐                  ┌──────────────────┐
   │  앱(gRPC client)  │                  │  앱(gRPC server)  │
   │      │ 평문(127.0.0.1)│              │      ▲ 평문(127.0.0.1)│
   │      ▼            │                  │      │            │
   │  Envoy 사이드카   │ ══ mTLS(h2) ════▶│  Envoy 사이드카   │
   └──────────────────┘   암호화·상호인증  └──────────────────┘
        │                                       │
        └── SVID 인증서를 메시 CA로부터 자동 발급/회전 ──┘
            (보통 SPIRE 또는 메시의 control plane)
```

여기서 앱과 사이드카 사이는 루프백 평문(3.3, 3.5절에서 설명한 insecure/local의 정당한 용도)이고, **파드와 파드 사이가 mTLS**다. 앱 개발자는 TLS 코드를 한 줄도 작성하지 않고 mTLS를 적용할 수 있다. 단, 이 구성이 안전하려면 메시가 **STRICT mTLS 모드**(평문 fallback 금지)로 설정되어 있어야 한다. PERMISSIVE 모드는 평문도 받아 주므로, 마이그레이션 중이 아니라면 함정이 된다(9절).

### 8.3 xDS와 SDS: 설정을 동적으로 내려받기

메시의 control plane은 사이드카(또는 프록시리스 gRPC)에 설정을 동적으로 내려보낸다. 이 프로토콜 묶음이 **xDS**(LDS/RDS/CDS/EDS/SDS 등)다. 그중 보안과 직접 관련된 것이 **SDS(Secret Discovery Service)** 이며, 인증서와 키 같은 비밀(secret)을 동적으로 배포하고 회전한다.

```
   control plane (예: Istiod / SPIRE)
        │ xDS (gRPC 스트림)
        ▼
   ┌──────────────── 데이터 플레인 ────────────────┐
   │  CDS: 어떤 업스트림 클러스터가 있나                │
   │  EDS: 그 클러스터의 엔드포인트(IP) 목록           │  ← [[12 - 이름 해석과 로드밸런싱]]
   │  SDS: mTLS용 인증서/키/신뢰루트 (★ 자동 회전)     │
   └────────────────────────────────────────────┘
```

gRPC는 **프록시리스(proxyless) xDS**도 지원한다. 사이드카 없이 gRPC 라이브러리가 직접 xDS control plane에 연결해 클러스터, 엔드포인트, 보안 설정을 받아 온다. 이때 mTLS 인증서도 SDS로 받아 자동으로 회전한다. 이름 해석과 로드밸런싱이 xDS로 통합되는 전체 구조는 12장(이름 해석과 로드밸런싱)에서 다룬다. 여기서 핵심은 메시와 xDS가 mTLS 인증서의 발급, 배포, 회전을 자동화해서 8.4절의 만료 장애를 구조적으로 줄인다는 점이다.

### 8.4 인증서 회전(rotation)과 만료: 가장 흔한 대형 장애

mTLS를 도입한 조직이 가장 많이 겪는 장애는 거의 항상 **인증서 만료**다. 이 장애가 특히 위험한 이유는 다음과 같다.

```
   인증서 만료 장애의 폭발 반경

   - 인증서는 "특정 시각에 동시에" 만료된다 (유효기간 끝)
   - mTLS는 "모든 연결"이 인증서에 의존
   → 만료되는 순간 그 인증서를 쓰는 모든 서비스 간 연결이
     한꺼번에 실패 (cliff edge, 절벽형 장애)
   - 평소엔 멀쩡하다가 정해진 시각에 전면 다운
   - 재시도([[13 - 안정성 - Retry Health Check Keepalive]])로도 못 푼다
     (인증서가 유효해지지 않는 한 계속 실패)
```

이를 막는 운영 원칙은 다음과 같다.

| 원칙                       | 이유                                                                                                                |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| **수명을 짧게, 회전을 잦게**       | SPIFFE/SPIRE는 SVID 수명을 시간 단위로 짧게 잡고 자주 회전한다. 수명이 짧을수록 회전 파이프라인이 제대로 동작하는지 상시 검증된다. 수명이 길면 회전 코드가 1년에 한 번 실행되고, 그때 처음 버그가 드러난다. |
| **만료 전 선제 회전**           | 유효기간의 절반~2/3 지점에서 미리 새 인증서로 교체한다. 토큰의 60초 여유분(5.3)과 같은 발상이다.                                                             |
| **회전 시 무중단(hot reload)** | 프로세스를 재시작하지 않고 새 인증서를 메모리에 다시 로드한다. Go라면 `tls.Config.GetCertificate` 콜백으로 핸드셰이크마다 최신 인증서를 고르게 한다.                    |
| **만료 임박 모니터링/알람**        | "D-14, D-7, D-1" 만료 알람을 둔다. CA 인증서(루트/중간)는 leaf보다 수명이 훨씬 길지만 만료되면 더 치명적이므로 별도로 추적한다.                                               |
| **클럭 동기화(NTP)**          | 시계가 어긋나면 아직 유효한 인증서를 만료된 것으로 보거나, 그 반대의 일이 생긴다. 8.5절에서 다룬다.                                                                          |

Go에서 무중단 회전을 구현하는 패턴은 다음과 같다.

```go
// 매 핸드셰이크마다 디스크의 최신 인증서를 다시 로드해 고른다.
// (cert-manager/SPIRE가 파일을 갱신하면 재시작 없이 반영됨)
type rotatingCert struct {
    mu   sync.RWMutex
    cert *tls.Certificate
}

func (r *rotatingCert) getCertificate(*tls.ClientHelloInfo) (*tls.Certificate, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    return r.cert, nil
}

func (r *rotatingCert) reloadLoop(certFile, keyFile string) {
    for range time.Tick(1 * time.Minute) { // 주기적으로 디스크 확인
        c, err := tls.LoadX509KeyPair(certFile, keyFile)
        if err != nil {
            log.Printf("cert reload failed (keeping old): %v", err) // ★ 실패해도 기존 유지
            continue
        }
        r.mu.Lock()
        r.cert = &c
        r.mu.Unlock()
    }
}

tlsCfg := &tls.Config{
    GetCertificate: rc.getCertificate,            // 서버: 매번 최신 인증서
    ClientAuth:     tls.RequireAndVerifyClientCert,
    ClientCAs:      clientCAs,
    MinVersion:     tls.VersionTLS12,
}
```

### 8.5 시계 오차(clock skew)

인증서와 토큰은 모두 시간(유효기간, `exp`·`nbf`)에 의존한다. 서버와 클라이언트의 시계가 어긋나면 다음과 같은 일이 생긴다.

- 클라이언트가 방금 발급받은 토큰의 `nbf`(not before)가 서버 시계로는 아직 미래라서 거부될 수 있다.
- 인증서의 `NotBefore`가 검증자 시계로 미래이면 "아직 유효하지 않음"으로 검증이 실패한다.

그래서 다음 세 가지를 지킨다.

- 모든 노드에 NTP를 강제한다.
- 검증자는 작은 허용 오차(leeway, 보통 30초~몇 분)를 둔다.
- 토큰은 5.3처럼 만료 전에 여유를 두고 갱신한다.

분산 시스템에서는 노드마다 시계가 다르다는 사실이 보안 문제로 가장 뚜렷하게 드러나는 지점이 바로 여기다.

---

## 9. 흔한 함정과 보안 주의사항

실제 사고에서 배우는 것이 이론보다 많다. gRPC 보안에서 반복해서 일어나는 함정을 정리한다.

### 9.1 인증서 검증 비활성화 (`InsecureSkipVerify`)

가장 치명적이면서 가장 흔한 함정이다. 개발 중에 인증서 검증이 번거롭다는 이유로 꺼 두었다가 그대로 프로덕션에 배포하는 경우다.

```go
// 절대 프로덕션 금지
tlsCfg := &tls.Config{
    InsecureSkipVerify: true, // 서버 인증서를 전혀 검증 안 함
}
```

`InsecureSkipVerify: true`를 설정하면 TLS 핸드셰이크는 하지만 **상대가 누구인지 확인하지 않는다.** 암호화는 되지만, 암호화된 상태로 공격자와 통신하게 될 수 있다. 중간자가 자기 인증서로 끼어들어도 통과하므로 mTLS는커녕 단방향 인증조차 무력화된다. **암호화와 인증은 다르다**는 사실을 가장 비싼 대가를 치르고 배우게 되는 코드다. 검증을 끄는 대신 사설 CA를 trust roots에 정식으로 등록하자(4.4의 `RootCAs`).

### 9.2 평문 fallback / PERMISSIVE 모드

TLS 연결이 안 되면 평문으로라도 연결하는 fallback은 다운그레이드 공격(downgrade attack)의 통로가 된다. 공격자는 TLS 핸드셰이크를 일부러 방해해 연결을 평문으로 떨어뜨린 뒤 도청한다. 서비스 메시의 PERMISSIVE mTLS 모드도 평문을 받아 주므로 같은 위험이 있다. 마이그레이션이 끝나면 반드시 STRICT로 전환해야 한다. fallback은 가용성을 얻으려고 보안을 포기하는 선택인데, 대개 얻는 것보다 잃는 것이 크다.

### 9.3 토큰/인증서를 로그에 노출

`authorization` 헤더 값(토큰)이나 개인키가 로그, 트레이스, 에러 메시지에 남으면 로그 수집 파이프라인 전체가 자격증명 유출 경로가 된다.

```go
// 나쁨: 토큰이 그대로 로그에 남는다
log.Printf("incoming md: %v", md) // md에 authorization 헤더가 들어있다!

// 좋음: 민감 헤더는 마스킹
func redactMD(md metadata.MD) metadata.MD {
    out := md.Copy()
    for _, k := range []string{"authorization", "cookie", "x-api-key"} {
        if len(out.Get(k)) > 0 {
            out.Set(k, "***REDACTED***")
        }
    }
    return out
}
```

7.1절의 인증 인터셉터가 검증에 실패했을 때 "invalid token"만 남기고 토큰 값은 절대 기록하지 않은 것도 같은 이유다. 관찰성([[14 - 관찰성과 디버깅 - Reflection grpcurl]])을 높이려고 메타데이터를 통째로 덤프하다가 자격증명이 새는 사고가 흔하다.

### 9.4 grpcurl/reflection이 주는 착시

로컬에서 `grpcurl -plaintext`로 호출이 잘 되면 보안도 문제없다고 착각하기 쉽다. 하지만 `-plaintext`는 TLS를 끈 상태이므로 프로덕션 보안과는 관계가 없다. 14장(관찰성과 디버깅)에서 다루듯이, 리플렉션 서비스를 외부에 노출하면 공격자에게 API 구조를 알려 주는 셈이다. 따라서 프로덕션에서는 리플렉션을 인가 뒤에 두거나 끈다.

### 9.5 호스트네임 검증 누락 / SAN 없는 인증서

2.2절에서 본 것처럼, 유효한 인증서라도 SAN이 타깃 호스트와 맞지 않으면 검증을 통과시켜서는 안 된다. 그런데 일부 코드는 검증 에러가 나면 9.1의 `InsecureSkipVerify`로 "해결"한다. 올바른 해결책은 (a) SAN을 제대로 넣어 인증서를 재발급하거나, (b) 클라이언트의 검증용 권위(ServerName/authority)를 올바른 호스트명으로 지정하는 것이다.

### 9.6 토큰 만료·시계 오차

5.3절과 8.5절에서 다룬 내용이다. 갱신 여유분이 0이면 만료 경계에서 `UNAUTHENTICATED`가 산발적으로 발생한다. 재현이 어려워서 디버깅하기가 매우 힘들다. 인증 실패가 간헐적으로 일어나고 특정 노드에 몰려 있다면 시계 오차를 의심하자.

### 9.7 mTLS 인증서 대량 만료

8.4절에서 다룬 절벽형 장애다. 회전 파이프라인을 평소에 자주 실행해 상시 검증하는 것이 유일하게 믿을 만한 예방책이다. 유효기간이 1년인 인증서를 쓰면 회전 코드가 1년 동안 실행되지 않으므로, 그 코드에 버그가 있어도 아무도 모른다. 역설적으로 수명을 짧게 하고 자주 회전하는 편이 더 안전하다.

### 함정 요약표

| 함정 | 증상 | 올바른 처방 |
|------|------|------------|
| `InsecureSkipVerify` | 중간자에 무방비 | 사설 CA를 trust roots에 등록 |
| 평문 fallback / PERMISSIVE | 다운그레이드 공격 | STRICT, fallback 금지 |
| 토큰을 로그에 노출 | 로그가 유출 경로 | 민감 헤더 마스킹 |
| `-plaintext` 착시 | 보안 검증 오인 | 프로덕션은 TLS 경로로 검증 |
| SAN 누락/호스트 불일치 | 검증 실패 → 임시로 검증 끔 | SAN 재발급/authority 지정 |
| 토큰 만료·시계 오차 | 간헐적 UNAUTHENTICATED | 갱신 여유분 + NTP + leeway |
| mTLS 대량 만료 | 정해진 시각 전면 다운 | 짧은 수명 + 잦은 자동 회전 + 알람 |

---

## 10. 전체 그림: RPC 하나가 안전하게 전달되는 과정

마지막으로 지금까지 다룬 내용을 하나의 흐름으로 연결해 보자. 사용자 alice가 모바일 앱에서 "할 일 생성"을 누르면 다음과 같은 일이 일어난다.

```
1. [앱 → 게이트웨이]  TLS (서버 인증서) + Bearer 토큰
   - ALPN으로 h2 협상, SNI=api.example.com
   - 서버 인증서 검증(SAN 매칭) → 관 확보 (PRIVACY_AND_INTEGRITY)
   - authorization: Bearer <alice의 JWT>  (TLS 안쪽이므로 안전)

2. [게이트웨이 내부]  인증 인터셉터
   - JWT 서명/exp/aud 검증 → principal=user:alice, scopes=[task:write]
   - 인가 인터셉터: CreateTask는 task:write 필요 → 통과

3. [게이트웨이 → 할일서비스]  mTLS + (전달된) 토큰  ← 자격증명 합성
   - 양쪽 SVID 교환: 게이트웨이=spiffe://.../gateway, 할일서비스 검증
   - 할일서비스: peer 인증서 SAN으로 "게이트웨이가 호출"임을 확인
   - 토큰(또는 새로 발급한 내부 토큰)으로 "원 주체는 alice"임을 전달

4. [할일서비스]  워크로드 인가 + 주체 인가
   - 워크로드 인가: 호출자가 gateway SPIFFE ID인가? (mTLS 기반)
   - 주체 인가: alice가 자기 리소스만 건드리는가? (토큰 기반)
   - 통과 → 비즈니스 로직 실행 → 응답

   ※ 어디선가 토큰 만료/인증서 만료/검증 실패면:
     - 신원 모름 → UNAUTHENTICATED(16)
     - 신원은 알지만 권한 없음 → PERMISSION_DENIED(7)
```

이 과정에서 **채널 축(TLS/mTLS)** 과 **호출 축(토큰)**, 그리고 **인가(인터셉터)** 는 각자 다른 일을 하면서 여러 겹으로 쌓여 심층 방어(defense in depth)를 이룬다. 어느 한 층이 뚫려도 다른 층이 남는다. gRPC 보안 설계가 자격증명을 두 종류로 나눈 결과로 얻는 이점이 바로 이것이다.
