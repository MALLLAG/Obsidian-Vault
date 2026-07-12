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
## 이 장이 답하는 질문

- gRPC는 왜 "전송 보안(TLS)"과 "요청 인증(토큰)"을 한 덩어리로 묶지 않고 **두 종류의 자격증명**으로 쪼갰을까? 그 분리가 주는 실익은 무엇인가?
- HTTP/2가 사실상 TLS를 요구한다는 말의 정확한 의미는 무엇이며, ALPN과 SNI는 핸드셰이크 안에서 **실제로 어떤 바이트로** 협상되는가?
- mTLS는 일반 TLS와 핸드셰이크 수준에서 무엇이 다른가? 서비스 메시의 SPIFFE/SVID, xDS는 mTLS를 어떻게 자동화하는가?
- 왜 "평문 채널 위에 Bearer 토큰을 실어 보내는 것"을 gRPC 라이브러리가 **기본적으로 거부**하는가?
- 인증(authentication)과 인가(authorization)는 코드의 어디에서 갈라지며, 인증서/토큰 만료·시계 오차·`InsecureSkipVerify` 같은 함정은 어떤 장애로 현실에 나타나는가?

---

## 0. 시작하기 전에: 보안을 "두 개의 질문"으로 쪼개기

분산 시스템에서 한 호출이 안전하려면 사실 두 개의 완전히 다른 질문에 동시에 답해야 한다.

1. **이 파이프를 도청·변조·가로채기로부터 보호할 수 있는가?** (전송 보안 / 연결의 무결성과 기밀성)
2. **이 파이프 반대편에서 요청을 보낸 주체는 누구이며, 그 주장을 믿을 근거가 있는가?** (신원 / 인증)

이 두 질문은 직교한다. 무슨 말이냐면, 둘은 서로의 답을 가정하지 않는다. 암호화된 파이프(1번 해결)라도 그 안으로 아무 신원 증명 없이 요청이 들어올 수 있고, 반대로 강력한 토큰(2번 해결)을 들고 와도 그것을 평문으로 실어 보내면 중간자가 토큰을 훔쳐 그대로 흉내 낼 수 있다.

gRPC의 보안 모델이 처음 보면 복잡해 보이는 이유는, 바로 이 직교성을 **타입 시스템 수준에서 그대로 드러냈기** 때문이다. gRPC는 보안을 하나의 "보안 옵션" 덩어리로 뭉뚱그리지 않고, 두 종류의 자격증명 객체로 분리한다.

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

이 그림을 머릿속에 박아두면 이 장의 나머지는 전부 "이 두 축의 디테일"로 읽힌다. 채널 자격증명은 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 다룬 전송 계층 바로 위에 얹히는 TLS 레이어를 다루고, 호출 자격증명은 [[08 - 메타데이터와 인터셉터]]에서 다룬 메타데이터(HTTP/2 헤더) 위에서 동작한다. 인가(authorization)는 또 한 번 분리되어 보통 [[08 - 메타데이터와 인터셉터]]의 서버 측 인터셉터에서 정책으로 처리된다.

세 단계로 요약하면 이렇다.

| 단계    | 질문         | gRPC에서 담당              | 산출물                |
| ----- | ---------- | ---------------------- | ------------------ |
| 전송 보안 | 파이프가 안전한가  | 채널 자격증명 (TLS/mTLS)     | 암호화·무결성·(상호)인증된 채널 |
| 인증    | 누구인가       | 호출 자격증명 + (m)TLS 피어 신원 | 검증된 주체(principal)  |
| 인가    | 무엇을 해도 되는가 | 서버 인터셉터의 정책 검사         | 허용/거부 결정           |

이 장은 위에서 아래로 내려간다. 먼저 파이프(TLS)를 깐 다음, 양쪽이 서로를 증명하게 만들고(mTLS), 그 위에 신분증(토큰)을 얹고, 마지막으로 "그래서 이 사람이 이걸 해도 되는가"를 판정하는 인가까지 간다.

---

## 1. 전송 보안의 토대: 왜 gRPC는 사실상 TLS를 요구하는가

### 1.1 HTTP/2와 TLS의 관계

gRPC는 HTTP/2를 전송으로 쓴다([[04 - HTTP2 깊이 보기 - 전송 계층]]). 그런데 HTTP/2에는 두 가지 "출입 방식"이 있다.

- **`h2`**: TLS 위에서 동작하는 HTTP/2. ALPN으로 협상한다.
- **`h2c`**: cleartext(평문) HTTP/2. TLS 없이 TCP 위에서 바로 동작한다.

웹 브라우저 세계에서는 `h2c`를 사실상 지원하지 않는다. 즉 "브라우저가 쓰는 HTTP/2"는 항상 TLS 위의 `h2`다. gRPC도 마찬가지로 프로덕션에서는 거의 항상 `h2`(TLS)를 쓰고, `h2c`(평문)는 개발용이나 "사이드카/서비스 메시 프록시 뒤에서 이미 mTLS가 처리되는 경우"의 로컬 홉(hop)에서만 쓴다.

RFC 7540(HTTP/2)은 `h2`에 보안 요구사항을 꽤 엄격하게 못 박았다. 핵심만 추리면 이렇다.

- **TLS 1.2 이상**을 사용해야 한다(RFC 7540 §9.2). 오늘날 실무에서는 TLS 1.2 또는 TLS 1.3(RFC 8446)을 쓴다.
- **SNI 확장**을 반드시 보내야 한다.
- **압축(TLS-level compression)을 끄고**, **재협상(renegotiation)을 금지**한다.
- 특정 약한 암호 스위트(cipher suite)들의 **블랙리스트**가 있다(RFC 7540 Appendix A). 예컨대 `TLS_RSA_WITH_AES_128_CBC_SHA` 같은 것들은 `h2`에서 사용이 금지된다. 이를 어기면 `INADEQUATE_SECURITY`라는 HTTP/2 에러 코드(0xc)로 연결이 끊긴다.

여기서 중요한 직관: **HTTP/2의 멀티플렉싱(multiplexing) 효율성은 "하나의 오래 사는 연결"을 전제로 한다.** 하나의 TLS 연결 위에서 수백 개의 스트림이 동시에 흐른다([[04 - HTTP2 깊이 보기 - 전송 계층]]). 그래서 그 단 하나의 연결이 깨지면 그 위의 모든 스트림이 영향을 받는다. 보안 측면에서도 마찬가지여서 이 "하나의 관"의 보안 수준이 곧 그 위 모든 RPC의 보안 수준이 된다. TLS를 제대로 까는 일이 그만큼 레버리지가 크다.

### 1.2 ALPN: "이 TLS 위에서 h2를 쓰자"를 핸드셰이크 안에서 합의하기

`h2`를 쓰려면 클라이언트와 서버가 "이 TLS 연결 위에서 우리가 말할 프로토콜은 HTTP/2다"라고 합의해야 한다. 이 합의를 TLS 핸드셰이크 안에서, **추가 왕복 없이** 끼워 넣는 메커니즘이 ALPN(Application-Layer Protocol Negotiation, RFC 7301)이다.

흐름은 이렇다.

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

핵심은 **선택의 주도권이 서버에 있다는 것**이다. 클라이언트는 후보 목록을 "선호 순서대로" 제시할 뿐이고, 최종 선택은 서버가 한다. gRPC 서버는 ALPN으로 `h2`를 광고하고, gRPC 클라이언트도 `h2`를 후보에 넣는다. 서버가 `h2`를 골라주면 gRPC 통신이 성립한다. 만약 서버가 `h2`를 지원하지 않아 다른 걸 고르거나 ALPN 자체를 협상하지 못하면, 엄격한 gRPC 클라이언트는 연결을 실패시킨다.

#### ClientHello를 손으로 디코딩해보기

ALPN이 "그냥 설정"이 아니라 실제 바이트로 핸드셰이크 안에 들어간다는 걸 눈으로 보자. 아래는 TLS ClientHello 레코드의 앞부분을 바이트로 풀어 본 것이다(값은 설명을 위한 대표적 예시).

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

- **확장 타입 `00 00`(SNI)** 안에 `"api.example.com"`이라는 호스트명이 **평문으로** 들어간다(TLS 1.2 기준). 이게 SNI다. 서버가 여러 가상 호스트를 한 IP에서 서빙할 때, "내가 접속하려는 건 이 호스트야"를 핸드셰이크 시작 시점에 알려준다. 서버는 이걸 보고 어떤 인증서를 제시할지 고른다.
- **확장 타입 `00 10`(ALPN)** 안에 `"h2"`와 `"http/1.1"`이 선호 순으로 들어간다.

TLS 1.3에서는 ServerHello 이후의 협상 결과(인증서, ALPN 선택 등)가 EncryptedExtensions로 암호화되어 보호되지만, ClientHello의 SNI는 여전히 평문이다(ECH/Encrypted Client Hello라는 확장으로 이걸 가리는 별도 기술이 있으나 일반 gRPC 운영에서는 거의 안 쓴다).

### 1.3 핸드셰이크 → preface → 첫 HEADERS

`h2` TLS 핸드셰이크가 끝나면 곧바로 HTTP/2가 시작된다. 가장 먼저 클라이언트는 **연결 preface**라는 고정 바이트열을 보낸다([[04 - HTTP2 깊이 보기 - 전송 계층]]).

```text
HTTP/2 connection preface (클라이언트가 보냄, 24바이트 고정)
  50 52 49 20 2a 20 48 54 54 50 2f 32 2e 30 0d 0a   "PRI * HTTP/2.0\r\n"
  0d 0a 53 4d 0d 0a 0d 0a                           "\r\nSM\r\n\r\n"
```

그 다음 양쪽이 SETTINGS 프레임을 교환하고, 클라이언트가 gRPC 호출용 HEADERS 프레임을 보낸다. 이 HEADERS 안에 `:method: POST`, `:path: /패키지.서비스/메서드`, `content-type: application/grpc`, 그리고 **`authorization: Bearer ...` 같은 호출 자격증명 메타데이터**가 들어간다. 즉 토큰은 **TLS로 암호화된 HTTP/2 헤더 안에** 실린다. 이 그림을 기억해두면 나중에 "왜 평문 위 토큰을 금지하는가"가 자명해진다.

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

## 2. 인증서와 신뢰 체인: 서버를 어떻게 "믿는가"

TLS의 본질은 두 가지다. (1) 키 교환으로 대칭키를 합의해 이후 트래픽을 암호화하고, (2) **인증서**로 상대가 주장하는 신원을 검증한다. gRPC 보안에서 실무적으로 사람을 가장 많이 괴롭히는 것은 (2)의 검증 로직이다.

### 2.1 PKI와 신뢰 체인

X.509 인증서는 "이 공개키는 이 신원(주체, subject)의 것이다"를 **인증 기관(CA, Certificate Authority)이 서명으로 보증**한 문서다. 클라이언트는 서버 인증서를 받으면 다음을 검증한다.

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

이 다섯 단계 중 1~4는 "이 인증서가 진짜 CA가 발급한 유효한 것인가"를 본다. 그러나 **5번, 호스트네임 검증이 없으면 보안이 통째로 무너진다.** 왜냐하면 공격자도 어떤 CA로부터 "evil.attacker.com"용 완벽히 유효한 인증서를 발급받을 수 있기 때문이다. 1~4만 통과시키면 그 유효한(그러나 다른 호스트용) 인증서로 중간자 공격이 성립한다. 5번이 "이 유효한 인증서가 **바로 내가 접속하려던 그 서버의 것인가**"를 묶어주는 마지막 못이다.

### 2.2 SAN과 호스트네임 검증 (CN은 죽었다)

호스트네임 검증의 규칙(RFC 6125, 이를 2023년에 대체한 RFC 9525, 그리고 RFC 2818의 갱신)은 명확하다.

- 검증 대상은 인증서의 **SAN(Subject Alternative Name)** 확장 안의 `dNSName` 항목이다.
- **Common Name(CN)은 더 이상 호스트네임 매칭에 쓰지 않는다.** 과거에는 CN을 fallback으로 봤지만, 현대 클라이언트(브라우저, Go 1.15+ 등)는 SAN이 없으면 그냥 실패시킨다. Go는 1.15부터 CN fallback을 완전히 제거했다.
- 와일드카드 `*.example.com`은 **단일 레이블**만 매치한다. `a.example.com`은 매치하지만 `a.b.example.com`은 매치하지 않는다.

그래서 인증서를 만들 때는 반드시 SAN을 넣어야 한다. openssl로 SAN 있는 서버 인증서를 만드는 예시는 아래 mTLS 절에서 한꺼번에 다룬다. 여기서는 "검증이 SAN을 본다"는 사실만 못 박는다.

gRPC에서 흔히 겪는 함정: 인증서의 SAN은 `api.example.com`인데 클라이언트가 IP `10.0.0.5:443`로 직접 다이얼하면 호스트네임 검증이 실패한다. 해결책은 두 가지다. (a) 인증서 SAN에 IP를 `iPAddress` 항목으로 추가하거나, (b) 클라이언트에서 "이 연결의 검증용 권위(authority)"를 명시적으로 `api.example.com`으로 오버라이드한다(Go의 `tls.Config.ServerName`, gRPC의 `WithAuthority` 등). [[12 - 이름 해석과 로드밸런싱]]에서 다루는 이름 해석은 사실 이 "타깃 호스트명"을 어떻게 IP로 풀고, 그 과정에서 검증용 권위를 어떻게 유지하느냐와 깊게 얽힌다.

### 2.3 gRPC의 보안 수준(security level) 개념

gRPC 내부(특히 C-core 기반 구현)는 채널의 보안 상태를 **security level**이라는 단계로 추상화한다.

| security level          | 의미                                |
| ----------------------- | --------------------------------- |
| `NONE`                  | 보안 없음(평문). insecure 채널.           |
| `INTEGRITY_ONLY`        | 변조 방지(무결성)는 되나 암호화(기밀성)는 안 됨. 드묾. |
| `PRIVACY_AND_INTEGRITY` | 기밀성 + 무결성 모두. 정상적인 TLS/mTLS/ALTS. |

이 개념이 중요한 이유는, **호출 자격증명(토큰)은 채널의 security level이 `PRIVACY_AND_INTEGRITY`일 때만 전송이 허용되도록** gRPC가 설계되어 있기 때문이다(상세는 5절). 즉 "이 관이 충분히 안전한가"를 라이브러리가 타입/런타임 수준에서 따져서 안전하지 않은 관으로 토큰이 새어 나가는 사고를 구조적으로 막는다.

---

## 3. 채널 자격증명의 종류: 언제 무엇을 쓰는가

채널 자격증명은 "이 연결을 어떻게 보호할 것인가"의 선택지다. gRPC는 몇 가지 표준 종류를 제공한다.

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

가장 기본. 서버는 인증서+개인키를 갖고, 클라이언트는 "신뢰하는 CA 목록(trust roots)"을 갖는다. 클라이언트만 서버를 검증한다(단방향). 인터넷에 노출되는 API의 기본형이다.

### 3.2 mTLS (상호 TLS) 채널 자격증명

서버뿐 아니라 **클라이언트도 자기 인증서를 제시**한다. 서버는 클라이언트 인증서를 검증해 "이 연결 반대편이 누구인지"를 TLS 핸드셰이크 단계에서 확립한다. 서비스 간(service-to-service) 통신, 제로 트러스트 네트워크의 표준. (4절에서 깊이 다룬다.)

### 3.3 insecure (평문)

암호화도 인증도 없다. security level은 `NONE`. **프로덕션에서 외부로 노출하면 절대 안 된다.** 정당한 용도는 둘뿐이다.

- **로컬 개발/테스트**: `localhost`에서 빠르게 돌려볼 때.
- **사이드카/메시 뒤의 로컬 홉**: Istio 같은 서비스 메시에서 사이드카 프록시(Envoy)가 mTLS를 대신 처리하는 경우, 앱 ↔ 사이드카 사이의 `127.0.0.1` 루프백 구간은 평문이어도 외부로 안 나가므로 허용된다. 이 경우에도 "사이드카가 실제로 mTLS를 강제하는가(STRICT 모드)"를 반드시 확인해야 한다.

### 3.4 ALTS (Application Layer Transport Security)

구글 인프라(GCE/GKE) 내부에서 쓰는, 구글이 설계한 상호 인증·암호화 프로토콜. TLS와 목적은 비슷하나 구글 환경의 신원(서비스 계정)에 묶여 동작한다. 구글 클라우드 밖에서는 의미가 없다. "그런 게 있다" 정도로 알아두면 된다.

### 3.5 local

`local` 채널 자격증명은 루프백(`localhost`/UDS) 연결에 한해 인증을 생략하는 특수 자격증명이다. 사이드카 패턴에서 "앱 ↔ 로컬 프록시" 통신을 다룰 때 쓴다. insecure와 비슷하지만 "정말 로컬인지"를 확인한다는 점에서 조금 더 안전하다.

### 비교표

| 종류 | 암호화 | 서버 인증 | 클라이언트 인증 | 주 용도 |
|------|:---:|:---:|:---:|------|
| TLS | O | O | X | 외부 노출 API |
| mTLS | O | O | O | 서비스 간, 제로 트러스트 |
| insecure | X | X | X | 로컬 개발, 메시 내부 홉 |
| ALTS | O | O | O | 구글 클라우드 내부 |
| local | (루프백) | (생략) | (생략) | 사이드카 로컬 홉 |

---

## 4. mTLS: 양쪽이 서로를 증명하다

### 4.1 일반 TLS와 무엇이 다른가

핸드셰이크 수준에서 mTLS는 일반 TLS에 **CertificateRequest**와 **클라이언트 Certificate / CertificateVerify**가 더해진 형태다.

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

여기서 `CertificateVerify`가 핵심이다. 클라이언트가 인증서(공개키)만 제시하면 "그 공개키에 대응하는 개인키를 실제로 갖고 있는지"는 증명되지 않는다(인증서는 공개 정보라 복사할 수 있다). `CertificateVerify`는 클라이언트가 지금까지의 핸드셰이크 메시지들에 **자기 개인키로 서명**한 것이라, 개인키 소유를 증명한다. 서버는 이 서명을 클라 인증서의 공개키로 검증해 "복사한 인증서가 아니라 진짜 개인키 소유자"임을 확인한다.

### 4.2 왜 mTLS인가: 네트워크 위치 ≠ 신원

전통적 보안은 "내부 네트워크 안에 있으면 믿는다"였다(성벽 모델). 그러나 클라우드/컨테이너 시대에는 같은 VPC, 같은 클러스터 안에서도 수백 개 서비스가 돌고, 한 파드가 뚫리면 내부 전체가 위험해진다. **제로 트러스트(zero trust)** 의 핵심 명제는 "네트워크 위치는 신원이 아니다"이다. IP 10.x 대역에서 왔다는 사실이 그 호출자가 결제 서비스라는 증거가 되지 못한다.

mTLS는 이 문제를 정면으로 푼다. 모든 서비스가 자기 인증서(= 암호학적 신원)를 갖고, 연결할 때마다 서로 그 신원을 증명한다. "10.0.3.7에서 왔다"가 아니라 "인증서 SAN이 `spiffe://cluster.local/ns/payments/sa/charger`인 워크로드가 왔다"가 인증의 근거가 된다.

### 4.3 인증서 일습 만들기 (openssl 실전)

mTLS를 직접 해보려면 (1) CA, (2) SAN 있는 서버 인증서, (3) 클라이언트 인증서가 필요하다.

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

여기서 EKU(Extended Key Usage)를 주목하자. 서버 인증서는 `serverAuth`, 클라이언트 인증서는 `clientAuth`. mTLS에서 이 구분이 중요하다. 클라이언트 인증서 SAN에 `URI:spiffe://...`를 넣은 게 보이는데, 이게 다음 절의 SPIFFE 신원이다.

### 4.4 Go로 보는 단방향 TLS → mTLS

먼저 일반 TLS 서버/클라이언트.

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

`NewClientTLSFromFile`의 두 번째 인자가 바로 2.2절의 호스트네임 검증 대상을 명시하는 곳이다. IP로 다이얼하더라도 이 인자를 `api.example.com`으로 주면 SAN을 그 이름으로 검증한다.

이제 mTLS. 차이는 (1) 서버가 클라이언트 인증서를 **요구**하고 검증하도록 `tls.Config`를 직접 구성하고, (2) 클라이언트가 자기 인증서를 들고 가는 것이다.

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

`tls.ClientAuth`의 값이 mTLS 강도를 좌우한다.

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

핸드셰이크가 끝나면 서버는 검증된 클라이언트 인증서의 정보(SAN, CN 등)를 RPC 핸들러에서 꺼내 쓸 수 있다. 이게 mTLS가 주는 "공짜 인증 정보"다.

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

이 `peerIdentity`가 반환하는 SPIFFE ID 또는 CN이 곧 "이 호출자는 어떤 서비스인가"의 인증 결과다. 인가(authorization)는 이 값을 받아 정책을 검사한다(6절, 7절).

### 4.5 Java / Python에서의 모양

언어별 API는 다르지만 개념은 동일하다.

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

## 5. 호출 자격증명: 요청마다 "누구"를 운반하기

채널 자격증명이 "관"을 보호한다면, 호출 자격증명(call credentials)은 관 속을 흐르는 **요청 단위 신분증**이다. mTLS가 "어느 서비스(워크로드)에서 왔는가"를 증명하는 데 강하다면, 토큰은 보통 "어느 사용자/앱이 이 요청의 주체인가"를 운반한다. 둘은 상호 보완적이다.

### 5.1 토큰은 메타데이터로 간다

호출 자격증명의 정체는 단순하다. 각 RPC가 나갈 때 **메타데이터(HTTP/2 헤더)에 `authorization` 키를 추가**하는 훅(hook)이다([[08 - 메타데이터와 인터셉터]]). 가장 흔한 형식은 OAuth2 Bearer 토큰이다.

```text
HEADERS 프레임 안 (TLS로 암호화됨):
  :method: POST
  :path: /todo.TodoService/CreateTask
  content-type: application/grpc
  authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImFiYzEy...   ← JWT
```

JWT(JSON Web Token)는 `header.payload.signature` 세 부분을 base64url로 인코딩해 점(.)으로 이은 것이다. payload에는 보통 `sub`(주체), `exp`(만료), `aud`(대상), `scope`/`scp`(권한 범위) 같은 클레임이 들어간다. 서버는 이 토큰의 서명을 발급자(IdP)의 공개키로 검증해 "위조되지 않았고 만료되지 않았으며 우리를 향한 토큰"임을 확인한다.

### 5.2 Go: PerRPCCredentials 구현

gRPC는 호출 자격증명을 `PerRPCCredentials` 인터페이스로 추상화한다. 핵심 메서드 두 개다.

```go
type PerRPCCredentials interface {
    // 각 RPC마다 호출되어, 메타데이터(헤더)에 넣을 키-값을 반환
    GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error)
    // 이 자격증명이 "전송 보안"을 요구하는가? true면 평문 채널에서 거부됨
    RequireTransportSecurity() bool
}
```

가장 단순한 정적(static) 토큰 자격증명:

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

또는 **호출 단위로만** 붙이고 싶으면 `CallOption`으로 준다.

```go
resp, err := client.CreateTask(ctx, req,
    grpc.PerRPCCredentials(bearerToken{token: perCallToken}))
```

### 5.3 토큰 갱신(refresh): 자격증명이 "살아있어야" 하는 이유

정적 토큰은 만료된다. 진짜 운영용 자격증명은 **만료를 감지하고 갱신**할 수 있어야 한다. `GetRequestMetadata`가 매 RPC마다 호출된다는 점을 이용해, 그 안에서 토큰 신선도를 관리한다.

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

여기서 **60초의 여유분(skew margin)** 이 실무적으로 중요하다. 토큰의 `exp`가 정확히 만료되는 순간에 갱신을 시작하면, 발급 지연·네트워크 왕복·서버와의 시계 오차(clock skew) 때문에 "이미 만료된 토큰"을 보내 인증 실패가 난다. 만료 직전에 선제적으로 갱신하면 이 경계 케이스를 피한다. 시계 오차는 8절의 함정에서 더 다룬다. IdP 호출에 `ctx`를 그대로 전달하므로 [[10 - Deadline 취소 타임아웃]]의 deadline이 갱신 단계까지 전파되어 토큰 발급이 무한정 매달리지 않는다.

구글 환경에서는 ADC(Application Default Credentials)가 이 갱신 로직을 알아서 처리한다. 환경 변수/메타데이터 서버/서비스 계정 키 파일에서 자격증명을 찾아 토큰을 발급·갱신해 자동으로 메타데이터에 붙인다. 직접 `fetch`를 구현할 필요 없이 라이브러리가 처리한다.

### 5.4 왜 "평문 채널 위 토큰"을 금지하는가

이게 이 장에서 가장 중요한 보안 직관 중 하나다. `RequireTransportSecurity()`가 기본적으로 `true`인 이유, 그리고 gRPC가 insecure 채널에 토큰 자격증명을 붙이면 런타임에서 거부하는 이유.

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

Bearer 토큰의 정의 자체가 "이 토큰을 **소지한(bearer) 자**는 누구든 그 권한을 가진다"이다. 현금과 같다. 그래서 평문으로 보내는 순간 도청자가 곧바로 그 신원을 도용할 수 있다. mTLS의 클라이언트 인증서는 개인키 소유를 매 핸드셰이크마다 증명(CertificateVerify)하므로 복사해도 못 쓰지만, Bearer 토큰은 복사하면 그냥 쓰인다. 이 비대칭 때문에 토큰은 **반드시 암호화된 관(TLS, security level `PRIVACY_AND_INTEGRITY`) 위에서만** 보내야 한다.

gRPC는 이 규칙을 라이브러리에 박아 넣어 "토큰 자격증명 + insecure 채널" 조합을 시도하면 다이얼/호출 시점에 에러를 던진다. 개발자가 실수로라도 토큰을 평문으로 흘리지 못하게 막는 **구조적 안전장치**다. (테스트 목적으로 `RequireTransportSecurity()`를 `false`로 만들 수는 있으나, 프로덕션에서 그렇게 하는 건 위 그림을 자초하는 셈이다.)

---

## 6. 자격증명 합성(composite): TLS 관 + 요청별 토큰

이제 두 축을 합친다. 표준 패턴은 "mTLS 또는 TLS로 보호된 채널" 위에 "per-RPC 토큰"을 얹는 것이다. 이를 gRPC에서는 채널 자격증명과 호출 자격증명을 **합성(composite)** 한다고 한다.

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

Go에서는 다이얼 옵션 두 개를 같이 주는 것 자체가 합성이다.

```go
conn, err := grpc.NewClient(
    "api.example.com:443",
    grpc.WithTransportCredentials(tlsCreds),            // 채널 축
    grpc.WithPerRPCCredentials(&oauthSource{...}),      // 콜 축
)
```

C++/일부 언어에서는 `CompositeChannelCredentials(channelCreds, callCreds)`라는 명시적 합성 API가 있다. 의미는 같다. 합성 시 gRPC는 "콜 자격증명이 요구하는 security level(보통 `PRIVACY_AND_INTEGRITY`)을 채널 자격증명이 만족하는가"를 검사한다. mTLS/TLS는 만족하므로 통과하고, insecure는 불만족이라 거부된다(5.4절).

이 합성 패턴이 실무의 표준인 이유:

- **mTLS**가 "이 호출이 들어온 워크로드(서비스 A의 파드)가 진짜 서비스 A인가"를 보증하고,
- **토큰**이 "이 요청의 최종 주체(end user `alice`, 혹은 위임받은 앱)가 누구인가"를 운반한다.

서비스 A가 사용자 alice의 요청을 받아 서비스 B를 호출할 때, B는 mTLS로 "A가 호출했다"를 알고, 토큰으로 "원 주체가 alice다"를 안다. 두 정보가 합쳐져야 "A가 alice를 대신해 정당하게 B를 호출한다"는 완전한 그림이 된다.

---

## 7. 인증(authn)과 인가(authz)의 분리

지금까지는 전부 **인증(authentication, "누구인가")** 이었다. TLS/mTLS로 워크로드 신원을, 토큰으로 주체 신원을 확립했다. 그러나 "누구인지 안다"와 "이걸 해도 된다"는 다른 문제다. 후자가 **인가(authorization, "무엇을 해도 되는가")** 이고, gRPC에서는 보통 **서버 측 인터셉터**에서 정책으로 처리한다([[08 - 메타데이터와 인터셉터]]).

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

상태 코드 선택이 중요하다([[09 - 에러 모델 - 상태 코드와 Rich Error]]).

- **`UNAUTHENTICATED` (16)**: "너가 누구인지 모르겠다/토큰이 없거나 위조/만료". 신원 확립 실패.
- **`PERMISSION_DENIED` (7)**: "누구인지는 알겠는데, 이건 권한이 없다". 신원은 OK, 권한 부족.

이 둘을 섞으면 클라이언트가 "토큰을 갱신해야 하나(401스럽게)" vs "권한을 요청해야 하나(403스럽게)"를 구분할 수 없다.

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

순서가 의미가 있다. 인증이 먼저 와서 `Principal`을 ctx에 심고, 인가가 그걸 읽어 정책을 적용한다. 인터셉터 체이닝의 메커니즘 자체는 [[08 - 메타데이터와 인터셉터]]에서 다룬다.

### 7.2 mTLS 신원 + 토큰 신원을 함께 보는 인가

더 정교한 인가는 **두 신원을 교차 검증**한다. 예를 들어 "서비스 A만 이 내부 메서드를 호출할 수 있고(워크로드 인가, mTLS 기반), 그 안에서 토큰 주체는 자기 자신의 리소스만 건드릴 수 있다(주체 인가, 토큰 기반)".

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

이렇게 **채널 축(mTLS 워크로드)** 과 **콜 축(토큰 주체)** 의 인가를 각각 다른 인터셉터로 두면, 정책을 깔끔하게 계층화할 수 있다.

---

## 8. 서비스 메시, SPIFFE/SVID, 그리고 mTLS의 자동화

지금까지의 mTLS는 "내가 직접 인증서를 만들고 배포하고 갱신한다"는 전제였다. 서비스가 5개면 할 만하지만 500개면 재앙이다. 인증서 발급/배포/회전을 수동으로 하면 반드시 어딘가에서 만료가 터진다. 그래서 등장한 게 **서비스 메시(service mesh)** 와 **SPIFFE/SVID** 같은 워크로드 신원 표준이다.

### 8.1 SPIFFE: 워크로드에 보편적 신원 주기

SPIFFE(Secure Production Identity Framework For Everyone)는 "모든 워크로드에 암호학적으로 검증 가능한 보편적 신원을 부여하자"는 표준이다. 핵심 개념 둘.

- **SPIFFE ID**: `spiffe://<trust-domain>/<path>` 형식의 URI. 예: `spiffe://example.com/ns/orders/sa/order-service`. 이게 워크로드의 이름이다.
- **SVID(SPIFFE Verifiable Identity Document)**: SPIFFE ID를 담은 검증 가능한 문서. **X.509-SVID**는 X.509 인증서의 SAN URI 필드에 SPIFFE ID를 박아 넣은 것이다(4.3의 openssl 예시에서 `URI:spiffe://...`를 넣은 게 바로 이것). JWT-SVID도 있다.

```
   X.509-SVID = 그냥 X.509 인증서인데
                SAN의 URI 항목 = SPIFFE ID

   Subject Alternative Name:
       URI:spiffe://example.com/ns/orders/sa/order-service
                    └── 이게 곧 워크로드의 신원 ──┘
```

mTLS 핸드셰이크에서 클라이언트가 X.509-SVID를 제시하면, 서버는 SAN의 SPIFFE ID로 "어떤 워크로드인가"를 즉시 안다. 4.4절 `peerIdentity`가 `leaf.URIs[0]`을 꺼낸 게 바로 이 SPIFFE ID다. 7.2절의 인가는 이 SPIFFE ID로 워크로드 단위 정책을 건다.

### 8.2 사이드카가 mTLS를 대신 짊어진다

Istio/Linkerd 같은 메시에서는 각 파드 옆에 **사이드카 프록시(보통 Envoy)** 가 붙는다. mTLS의 무거운 짐(인증서 발급, 핸드셰이크, 회전)을 이 프록시가 대신 진다.

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

여기서 앱 ↔ 사이드카 사이는 루프백 평문(3.3, 3.5절의 정당한 insecure/local 용도)이고, **파드 ↔ 파드 사이가 mTLS**다. 앱 개발자는 TLS 코드를 한 줄도 안 짜도 mTLS를 얻는다. 단, 이게 안전하려면 메시가 **STRICT mTLS 모드**(평문 fallback 금지)로 설정돼 있어야 한다. PERMISSIVE 모드는 평문도 받아주므로, 마이그레이션 중이 아니라면 함정이다(9절).

### 8.3 xDS와 SDS: 설정을 동적으로 내려받기

메시의 control plane은 사이드카(또는 프록시리스 gRPC)에 설정을 동적으로 내려준다. 이 프로토콜군이 **xDS**(LDS/RDS/CDS/EDS/SDS 등)다. 이 중 보안과 직결되는 게 **SDS(Secret Discovery Service)** 로, 인증서/키 같은 비밀(secret)을 동적으로 배포·회전한다.

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

gRPC는 **프록시리스(proxyless) xDS**도 지원한다. 사이드카 없이 gRPC 라이브러리 자체가 xDS control plane에 연결해 클러스터/엔드포인트/보안 설정을 받아온다. 이때 mTLS 인증서도 SDS로 받아 자동 회전한다. 이름 해석·로드밸런싱이 xDS로 통합되는 큰 그림은 [[12 - 이름 해석과 로드밸런싱]]에서 다룬다. 여기서 핵심은 "메시/xDS가 mTLS 인증서의 발급·배포·회전을 자동화해서, 8.4절의 만료 장애를 구조적으로 줄인다"는 점이다.

### 8.4 인증서 회전(rotation)과 만료: 가장 흔한 대형 장애

mTLS를 도입한 조직이 겪는 1순위 장애는 거의 항상 **인증서 만료**다. 그 메커니즘이 특히 무서운 이유:

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

이걸 막는 운영 원칙들:

| 원칙                       | 이유                                                                                                                |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| **수명을 짧게, 회전을 잦게**       | SPIFFE/SPIRE는 SVID 수명을 시간 단위로 짧게 잡고 자주 회전. 짧을수록 "회전 파이프라인이 살아있는지" 상시 검증됨. 긴 수명은 회전 코드가 1년에 한 번 돌아 그때 처음 버그가 드러난다. |
| **만료 전 선제 회전**           | 만료의 절반~2/3 지점에서 미리 새 인증서로 교체. 60초 토큰 여유분(5.3)과 같은 발상.                                                             |
| **회전 시 무중단(hot reload)** | 프로세스 재시작 없이 새 인증서를 메모리에 다시 로드. Go라면 `tls.Config.GetCertificate` 콜백으로 매 핸드셰이크마다 최신 인증서를 고르게 한다.                    |
| **만료 임박 모니터링/알람**        | "D-14, D-7, D-1" 만료 알람. CA 인증서(루트/중간)는 leaf보다 훨씬 길지만 더 치명적이라 별도 추적.                                               |
| **클럭 동기화(NTP)**          | 시계가 틀어지면 "아직 유효한데 만료로 본다" 또는 그 반대. 8.5절.                                                                          |

Go에서 무중단 회전을 거는 패턴:

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

인증서와 토큰 둘 다 시간(유효기간 / `exp`·`nbf`)에 의존한다. 서버와 클라이언트의 시계가 어긋나면:

- 클라가 막 발급받은 토큰의 `nbf`(not before)가 서버 시계로는 아직 "미래"라 거부될 수 있다.
- 인증서의 `NotBefore`가 검증자 시계로 미래면 "아직 유효하지 않음"으로 실패.

그래서 (1) 모든 노드에 NTP를 강제하고, (2) 검증자는 작은 허용 오차(leeway, 보통 30초~몇 분)를 두며, (3) 토큰 갱신은 5.3처럼 만료 전 여유를 둔다. 분산 시스템에서 "시간은 공유 자원이 아니다"라는 사실을 보안에서 가장 아프게 만나는 지점이다.

---

## 9. 흔한 함정과 보안 주의사항

이론보다 사고가 더 잘 가르친다. gRPC 보안에서 반복적으로 터지는 함정들을 정리한다.

### 9.1 인증서 검증 비활성화 (`InsecureSkipVerify`)

가장 치명적이고 가장 흔하다. 개발 중 "인증서 검증이 귀찮다"며 끄고, 그대로 프로덕션에 나간다.

```go
// 절대 프로덕션 금지
tlsCfg := &tls.Config{
    InsecureSkipVerify: true, // 서버 인증서를 전혀 검증 안 함
}
```

`InsecureSkipVerify: true`는 TLS 핸드셰이크는 하지만 **상대가 누구든 묻지 않는다.** 암호화는 되지만 "암호화된 채로 공격자와 통신"하게 된다. 중간자가 자기 인증서로 끼어들어도 통과하므로, 사실상 mTLS는커녕 단방향 인증조차 무력화된다. **암호화 ≠ 인증**이라는 사실을 가장 비싸게 배우는 코드다. 차라리 사설 CA를 trust roots에 정식 등록하라(4.4의 `RootCAs`).

### 9.2 평문 fallback / PERMISSIVE 모드

"TLS가 안 되면 평문으로라도 붙자"는 fallback은 다운그레이드 공격(downgrade attack)의 문이다. 공격자가 TLS 핸드셰이크를 일부러 방해해 평문으로 떨어뜨린 뒤 도청한다. 서비스 메시의 PERMISSIVE mTLS 모드도 같은 위험이 있다(평문도 받아줌). 마이그레이션이 끝나면 반드시 STRICT로 전환하라. fallback은 "가용성을 위해 보안을 끄는" 거래인데, 대개 그 거래는 남는 장사가 아니다.

### 9.3 토큰/인증서를 로그에 노출

`authorization` 헤더 값(토큰)이나 개인키가 로그·트레이스·에러 메시지에 찍히면, 로그 수집 파이프라인 전체가 자격증명 유출 경로가 된다.

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

7.1절 인증 인터셉터에서 검증 실패 시 "invalid token"만 남기고 토큰 값 자체는 절대 안 찍은 것도 같은 이유다. 관찰성([[14 - 관찰성과 디버깅 - Reflection grpcurl]])을 높이려고 메타데이터를 통째로 덤프하다가 자격증명을 새는 사고가 흔하다.

### 9.4 grpcurl/reflection으로 보안 우회 착시

`grpcurl -plaintext`로 로컬에서 잘 되니 "보안 OK"라고 착각하기 쉽다. `-plaintext`는 TLS를 끈 것이라 프로덕션 보안과 무관하다. [[14 - 관찰성과 디버깅 - Reflection grpcurl]]에서 다루지만, 리플렉션 서비스 자체를 외부에 노출하면 공격자에게 API 지도를 그려주는 셈이니 프로덕션에선 인가 뒤에 두거나 끈다.

### 9.5 호스트네임 검증 누락 / SAN 없는 인증서

2.2절에서 본 대로, "유효한 인증서"라도 SAN이 타깃 호스트와 안 맞으면 검증을 통과시키면 안 된다. 그런데 일부 코드는 검증 에러가 나자 9.1의 `InsecureSkipVerify`로 "해결"한다. 올바른 해결은 (a) SAN을 제대로 넣어 재발급하거나, (b) 클라이언트의 검증용 권위(ServerName/authority)를 올바른 호스트명으로 지정하는 것이다.

### 9.6 토큰 만료·시계 오차

5.3, 8.5에서 다룬 그대로. 갱신 여유분이 0이면 경계에서 산발적 `UNAUTHENTICATED`가 터지는데, 재현이 어려워 디버깅이 악몽이 된다. "간헐적 인증 실패 + 특정 노드에 몰림"이면 시계 오차를 의심하라.

### 9.7 mTLS 인증서 대량 만료

8.4의 절벽형 장애. "회전 파이프라인을 평소에 자주 돌려 상시 검증"이 유일하게 믿을 만한 예방책이다. 1년짜리 인증서는 회전 코드가 1년간 안 돌아 그 코드의 버그를 아무도 모른다. 짧은 수명 + 잦은 회전이 역설적으로 더 안전하다.

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

## 10. 전체 그림: 한 RPC가 안전하게 흐르는 일대기

마지막으로 지금까지의 조각을 하나의 흐름으로 꿰어 보자. 사용자 alice가 모바일 앱에서 "할 일 생성"을 누르면 일어나는 일이다.

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

이 일대기에서 **채널 축(TLS/mTLS)** 과 **콜 축(토큰)**, 그리고 **인가(인터셉터)** 가 각자 다른 일을 하며 겹겹이 쌓여 "심층 방어(defense in depth)"를 이루는 게 보인다. 어느 한 층이 뚫려도 다른 층이 남는다. 그게 gRPC 보안 설계가 두 종류 자격증명을 굳이 분리한 이유의 최종 보상이다.

---

## 핵심 요약

- gRPC 보안은 **채널 자격증명(연결 보호)** 과 **호출 자격증명(요청별 신원)** 이라는 **직교하는 두 축**으로 분리 설계됐고, 둘의 **합성(composite)** 이 표준 패턴이다. 분리 덕에 "암호화"와 "신원"을 독립적으로 조합·교체할 수 있다.
- HTTP/2의 `h2`는 사실상 **TLS 1.2+/1.3**을 요구하며, **ALPN**으로 `h2`를, **SNI**로 대상 호스트를 핸드셰이크 안에서 추가 왕복 없이 협상한다. 토큰은 그 결과로 만들어진 **암호화된 HTTP/2 헤더** 안에 실린다.
- 서버 인증의 마지막 못은 **SAN 기반 호스트네임 검증**이다. CN은 더 이상 매칭에 쓰지 않는다. **암호화는 인증이 아니다** — `InsecureSkipVerify`는 이 둘을 혼동한 가장 비싼 코드다.
- **mTLS**는 핸드셰이크에 클라이언트 `Certificate` + `CertificateVerify`를 더해 **양방향 상호 인증**을 만든다. "네트워크 위치는 신원이 아니다"라는 제로 트러스트의 핵심을 암호학적으로 구현한다.
- **SPIFFE/SVID**는 워크로드에 `spiffe://` URI 신원을 부여하고, **서비스 메시 + xDS(SDS)** 가 mTLS 인증서의 발급·배포·**회전**을 자동화해 만료 절벽 장애를 구조적으로 줄인다.
- **호출 자격증명**은 per-RPC로 `authorization` 메타데이터에 토큰을 붙이는 훅이다. Bearer 토큰은 "소지자=권한자"라 복제 즉시 도용 가능하므로, gRPC는 **보안 채널(PRIVACY_AND_INTEGRITY) 위에서만** 전송을 허용한다(`RequireTransportSecurity`).
- 토큰은 **만료 전 선제 갱신**(여유분)으로, 인증서는 **짧은 수명 + 잦은 무중단 회전**으로, 시간은 **NTP + leeway**로 관리한다. 시계 오차와 만료는 분산 보안의 단골 장애 원인이다.
- **인증(누구인가)과 인가(무엇을 해도 되는가)는 분리**된다. 신원은 (m)TLS와 토큰으로 확립하고, 인가는 **서버 인터셉터**에서 RBAC/스코프로 검사한다. 실패 코드는 `UNAUTHENTICATED(16)`(신원 모름)과 `PERMISSION_DENIED(7)`(권한 없음)을 엄격히 구분한다.

---

## 연결 노트

- [[04 - HTTP2 깊이 보기 - 전송 계층]] — TLS가 얹히는 전송 계층, `h2` 프레이밍·멀티플렉싱·연결 preface. ALPN/SNI가 동작하는 토대.
- [[08 - 메타데이터와 인터셉터]] — 호출 자격증명이 운반되는 메타데이터(HTTP/2 헤더), 그리고 인증·인가를 구현하는 인터셉터 체인.
- [[09 - 에러 모델 - 상태 코드와 Rich Error]] — `UNAUTHENTICATED(16)` / `PERMISSION_DENIED(7)`의 정확한 의미와 Rich Error로 인가 실패 컨텍스트를 전달하는 법.
- [[10 - Deadline 취소 타임아웃]] — 토큰 갱신/핸드셰이크에 deadline을 걸어 인증 단계가 무한정 매달리지 않게 하기.
- [[12 - 이름 해석과 로드밸런싱]] — 검증용 권위(authority)와 호스트네임 검증의 관계, xDS/SDS가 보안 설정과 엔드포인트 발견을 통합하는 큰 그림.
- [[13 - 안정성 - Retry Health Check Keepalive]] — 인증서 만료 같은 비복구성 실패와 일시적 실패의 구분, keepalive와 TLS 연결 수명의 상호작용.
- [[14 - 관찰성과 디버깅 - Reflection grpcurl]] — `grpcurl -plaintext`의 착시, 리플렉션 노출 위험, 메타데이터 덤프 시 자격증명 마스킹.
