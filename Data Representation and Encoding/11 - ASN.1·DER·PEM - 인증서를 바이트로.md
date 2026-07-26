---
title: ASN.1·DER·PEM - 인증서를 바이트로
date: 2026-06-26
tags: [encoding, asn1, der, pem, x509, tls, 학습노트]
---

이 장이 답하는 질문:

- 1984년 전화망 표준위원회가 만든 데이터 기술 언어가 어쩌다 **HTTPS 자물쇠 아이콘, LDAP 디렉터리, SNMP, Kerberos, 5G 시그널링**의 공통 기반이 되었는가? ASN.1은 왜 protobuf나 JSON으로 대체되지 않고 40년째 살아 있는가?
- ASN.1은 "추상 문법(타입 정의)"이고 BER/DER은 "인코딩 규칙"이라고들 한다. 이 **분리(decoupling)** 가 왜 결정적으로 중요한가? 같은 타입 정의가 어떻게 BER·DER·PER·XER라는 전혀 다른 바이트열로 떨어지는가?
- 모든 ASN.1 값은 결국 **Tag–Length–Value(TLV)** 세 토막이다. Tag 한 바이트 안에 class 2비트·구성 1비트·번호 5비트가 어떻게 욱여넣어지고, 번호가 31을 넘으면 무슨 일이 벌어지는가? INTEGER `-129`, OID `1.2.840.113549`, CN 문자열을 직접 손으로 바이트까지 떨어뜨릴 수 있는가?
- 디지털 서명에는 왜 BER이 아니라 **DER**이어야만 하는가? "정규형(canonical form)"이라는 단 하나의 성질이 서명 검증의 일관성을 어떻게 보장하고, 그 규칙(definite length·최소 인코딩·SET OF 정렬·BOOLEAN TRUE=0xFF)은 각각 무엇을 막는가?
- X.509 인증서 한 장을 hex로 펼치면 `30 82 ...`로 시작하는 한 그루의 나무다. version·serial·issuer·validity·subjectPublicKeyInfo·extensions가 그 안에서 어떻게 중첩되고, **확장값이 OCTET STRING 안에 또 DER로 포장되는 이중구조**는 왜 그렇게 설계됐는가?
- ASN.1 파서는 왜 보안 사고의 단골 무대인가? **길이 필드를 믿어서 생기는 오버리드, BER indefinite 중첩 DoS, CN 안의 NUL 바이트, PKCS#1 서명 위조(BERserk)** 는 각각 인코딩의 어느 빈틈을 찔렀는가?

이 시리즈는 지금까지 "하나의 값을 어떻게 바이트로 적는가"를 한 켜씩 쌓아 왔다. [[01 - 비트·바이트·엔디안]]에서 바이트와 엔디안을, [[05 - Varint·ZigZag·protobuf wire 해부]]에서 가변길이 정수를, [[10 - 체크섬 내장 인코딩 - Base58·Bech32]]에서 사람이 읽고 베껴 적을 수 있는 텍스트 인코딩을 손으로 풀었다. 이 장은 그 모든 조각이 한자리에 모이는 곳이다. **ASN.1/DER**은 가변길이 정수(OID), 길이 접두(length-prefix), 중첩 구조, 그리고 텍스트 포장(PEM의 base64)을 전부 한 포맷 안에 담는다. 인터넷의 신뢰 기반인 X.509 인증서가 바로 이 위에 서 있다.

직전 장 [[10 - 체크섬 내장 인코딩 - Base58·Bech32]]가 "DER 바이트열을 base64로 감싼 것"인 PEM의 텍스트 절반을 이미 다뤘다면, 이 장은 그 안에 든 바이너리 절반을 바이트 단위로 해부한다. 다음 장 [[12 - CBOR·Avro·zero-copy 직렬화]]는 ASN.1과 같은 "스키마 기반 바이너리 직렬화"라는 큰 흐름을 현대적으로 다시 쓴 포맷들 — TLV의 후예 CBOR, 스키마 분리의 후예 Avro — 로 이어진다. 그러니까 ASN.1은 과거의 유물이 아니라, 우리가 매일 쓰는 직렬화 사상의 **원형(prototype)** 이다. protobuf의 wire 포맷과 비교하고 싶다면 gRPC 시리즈의 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]을 곁에 두면 좋다.

그럼 가장 추상적인 질문부터 가장 구체적인 바이트까지 내려가 보자.

---

## 1. ASN.1은 어디에나 있다: 동기와 역사

### 1.1 1984년의 문제

1980년대 초, 서로 다른 컴퓨터들이 네트워크로 데이터를 주고받기 시작했을 때 근본적인 장벽이 있었다. 한쪽은 big-endian, 다른 쪽은 little-endian이다([[01 - 비트·바이트·엔디안]]). 한쪽은 EBCDIC, 다른 쪽은 ASCII다. 한쪽의 `int`는 16비트, 다른 쪽은 36비트다. "사용자 이름과 나이를 보내라"는 단순한 요구조차, 두 기계가 **같은 의미를 같은 바이트로 합의**하지 못하면 불가능했다.

CCITT(현 ITU-T)와 ISO는 이 문제를 두 단계로 쪼개서 풀었다. 이 분할이 ASN.1의 전부라 해도 과언이 아니다.

1. **무엇을 보내는가(추상 문법, abstract syntax)**: "이 메시지는 UTF8 문자열 하나와 정수 하나로 이루어진 레코드다" 같은 **타입 정의**. 기계 표현과 무관한, 순수히 의미만의 기술.
2. **그것을 어떻게 바이트로 적는가(전송 문법, transfer syntax / 인코딩 규칙)**: 위 타입을 실제 옥텟(octet) 열로 바꾸는 규칙.

ASN.1(**A**bstract **S**yntax **N**otation **One**)은 1번을 위한 언어다. 1984년 X.409로 처음 나왔고, 곧 X.208을 거쳐 오늘날 ITU-T **X.680** 계열로 정착했다. 2번은 별개의 표준, **X.690**(BER/CER/DER)·**X.691**(PER)·**X.693**(XER) 등이다. "Notation **One**"이라는 이름 자체가 "이건 표기법일 뿐, 인코딩이 아니다"라는 선언이다.

### 1.2 왜 아직도 ASN.1인가

JSON·protobuf·MessagePack이 넘쳐나는 시대에 ASN.1은 왜 안 죽었나. 답은 단순하다 — **이미 깔린 인프라가 너무 거대하다.**

| 시스템 | ASN.1이 쓰이는 곳 |
|---|---|
| **X.509 / TLS** | 모든 HTTPS 인증서, 인증서 체인, CRL, OCSP 응답 |
| **PKCS** | 키 포맷(#1 RSA, #8 PKCS8), CSR(#10), 서명/봉투(#7/CMS), 키 묶음(#12 .pfx) |
| **LDAP** | 디렉터리 프로토콜의 모든 요청/응답(BER) |
| **SNMP** | 네트워크 장비 관리 MIB·PDU(BER) |
| **Kerberos** | 티켓·인증자(authenticator) 구조(DER) |
| **이동통신** | 3G/4G/5G의 시그널링(NAS/RRC는 PER로 비트까지 압축) |
| **전자여권(eMRTD)** | ICAO 9303 칩의 데이터그룹 |
| **EMV** | 칩카드 결제 메시지의 TLV(ASN.1 BER-TLV 계열) |

핵심은 **X.509**다. 당신이 지금 이 노트를 읽는 동안에도 브라우저는 수십 개의 ASN.1/DER 인증서를 파싱하고 있다. 인터넷 PKI(공개키 기반구조) 전체가 이 포맷 위에 서 있고, 그래서 ASN.1 파서의 버그는 곧 인터넷의 보안 사고다(14절).

> ASN.1은 "타입 시스템"이고 X.690은 "직렬화 규칙"이다. 둘을 한 단어로 뭉뚱그려 "ASN.1 = DER"이라고 생각하는 순간 거의 모든 혼란이 시작된다. ASN.1은 protobuf의 `.proto` 문법([[02 - Protocol Buffers 1 - 문법과 타입 시스템]])에, DER은 protobuf의 wire 포맷([[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]])에 대응한다고 보면 정확하다.

---

## 2. 두 개의 표준: 추상 문법(X.680)과 인코딩 규칙(X.690)

### 2.1 추상 문법 — 타입을 적는 언어

ASN.1 모듈은 이렇게 생겼다. 익숙한 구조체/레코드 정의처럼 읽힌다.

```text
Person ::= SEQUENCE {
    name      UTF8String,
    age       INTEGER (0..150),
    email     IA5String OPTIONAL,
    married   BOOLEAN DEFAULT FALSE
}
```

`::=`는 "정의된다"이고, `SEQUENCE`는 순서 있는 필드 묶음(구조체), `OPTIONAL`은 생략 가능, `DEFAULT`는 기본값이다. `(0..150)`은 **서브타입 제약(subtype constraint)** — 값의 범위를 제한한다. 여기에는 바이트 이야기가 한 줄도 없다. `age`가 1바이트인지 4바이트인지, big-endian인지는 **인코딩 규칙이 결정할 일**이다.

### 2.2 인코딩 규칙 — 같은 타입, 여러 바이트열

X.690과 친척들은 위 `Person { name="Al", age=42 }` 한 값을 서로 다른 바이트열로 만든다.

| 인코딩 규칙 | 표준 | 성격 | 한 마디 |
|---|---|---|---|
| **BER** (Basic) | X.690 | TLV, 선택지 많음 | 가장 너그러움. 같은 값에 여러 표현 허용 |
| **CER** (Canonical) | X.690 | BER의 정규 부분집합 | 스트리밍용. 긴 값은 indefinite + 1000옥텟 청크 |
| **DER** (Distinguished) | X.690 | BER의 정규 부분집합 | **저장·서명용**. 같은 값 → 유일한 바이트열 |
| **PER** (Packed) | X.691 | 비트 단위 압축 | TLV 안 씀. 5G/항공이 대역폭 아끼려 사용 |
| **OER** (Octet) | X.696 | 옥텟 정렬 압축 | PER보다 빠른 파싱, 적당한 크기 |
| **XER** (XML) | X.693 | XML 텍스트 | 디버깅·상호운용 |
| **JER** (JSON) | X.697 | JSON 텍스트 | 현대적 상호운용 |

이 장은 **BER과 그 정규형 DER**에 집중한다. X.509·PKCS·Kerberos가 모두 DER을 쓰고, BER은 LDAP·SNMP가 쓴다. PER은 비트 단위로 TLV를 아예 버리고 스키마를 양쪽이 똑같이 알아야만 풀 수 있는 "자기서술 없는(self-describing가 아닌)" 인코딩인데, 그 사상은 [[12 - CBOR·Avro·zero-copy 직렬화]]의 Avro(스키마 분리, 태그 없는 인코딩)와 정확히 닮았다.

> TLV 계열(BER/DER)은 **자기서술적**이다. 스키마 없이도 "여기 길이 9짜리 OBJECT IDENTIFIER가 있구나"까지는 파싱된다. 반면 PER/Avro는 스키마가 있어야만 한 비트도 해석할 수 없는 대신 더 작다. 이 트레이드오프 — 자기서술 vs 압축 — 는 직렬화 포맷 설계의 영원한 축이다.

---

## 3. ASN.1 타입 동물원

DER로 내려가기 전에, 어떤 타입들이 있는지 분류해 두자. 각 타입에는 **universal tag number**가 박혀 있고(4절), 그 번호가 곧 바이트가 된다.

### 3.1 단순 타입

| 타입 | tag(10진) | tag(16진) | 설명 |
|---|---|---|---|
| BOOLEAN | 1 | 0x01 | 참/거짓 |
| INTEGER | 2 | 0x02 | 임의 정밀도 정수(2의 보수, big-endian) |
| BIT STRING | 3 | 0x03 | 비트 열(끝에 unused-bit 개수 한 바이트) |
| OCTET STRING | 4 | 0x04 | 임의 바이트 열 |
| NULL | 5 | 0x05 | 값 없음 |
| OBJECT IDENTIFIER | 6 | 0x06 | OID(점으로 잇는 아크) |
| ENUMERATED | 10 | 0x0A | 열거형(INTEGER처럼 인코딩) |
| UTF8String | 12 | 0x0C | UTF-8 문자열 |
| NumericString | 18 | 0x12 | 0–9와 공백만 |
| PrintableString | 19 | 0x13 | A–Z a–z 0–9와 일부 기호 |
| IA5String | 22 | 0x16 | ASCII(IA5=국제 알파벳 5호) |
| UTCTime | 23 | 0x17 | 2자리 연도 시각 |
| GeneralizedTime | 24 | 0x18 | 4자리 연도 시각 |
| VisibleString | 26 | 0x1A | 인쇄가능 ASCII |
| BMPString | 30 | 0x1E | UCS-2(2바이트 고정, big-endian) |

### 3.2 구조 타입

| 타입 | tag(16진, constructed 포함) | 설명 |
|---|---|---|
| SEQUENCE / SEQUENCE OF | 0x30 | 순서 있는 묶음 / 같은 타입의 정렬된 목록 |
| SET / SET OF | 0x31 | 순서 없는 묶음(서로 다른 타입) / 같은 타입의 집합 |
| CHOICE | — | 여러 대안 중 하나(자기 tag 없음, 선택된 타입의 tag를 그대로 씀) |

`SEQUENCE`와 `SET`은 tag 번호가 각각 16·17인데, 항상 **구성(constructed)** 이므로 6번 비트가 켜져 `0x30`·`0x31`로 나타난다(4.1절에서 이유 설명).

### 3.3 문자열 타입이 이렇게 많은 이유

`UTF8String`·`PrintableString`·`IA5String`·`BMPString`… 같은 "문자열"이 왜 이렇게 갈래가 많을까. 역사적 이유가 크다. ASN.1이 태어난 1984년에는 유니코드가 없었다. 텔렉스·전화망 시절의 제한된 문자 집합들(`PrintableString`은 전신 단말의 인쇄 가능 글자)이 그대로 굳었다. X.509는 이 유산을 떠안고 있다.

```text
PrintableString 허용 문자(딱 이만큼):
   A–Z  a–z  0–9  (공백)  ' ( ) + , - . / : = ?
   → '@', '_', '*', '&' 등은 불허! 이메일·언더스코어 불가
```

이 미묘한 차이가 보안 문제로 번진다. CN(common name)이 `PrintableString`이냐 `UTF8String`이냐 `IA5String`이냐에 따라 같은 글자가 다른 바이트로 인코딩되고, 비교/검증 로직이 그 차이를 무시하면 **homograph·confusable 공격**의 표면이 된다([[09 - 유니코드 보안 - confusables와 homograph]]). RFC 5280은 이 혼란을 줄이려 "2004년 이후 발급 인증서의 DirectoryString은 가급적 `UTF8String`을 쓰라"고 못 박았다.

> "ASN.1 문자열은 다 같은 거 아니냐"고 흔히들 여긴다. 아니다. tag 번호가 다르면 **DER 바이트가 다르고**, DER이 다르면 **서명 해시가 다르다**. `PrintableString "US"`(13 02 55 53)와 `UTF8String "US"`(0C 02 55 53)는 다른 인증서다. 발급 시점에 어떤 타입으로 적었느냐가 영구히 박힌다.

---

## 4. TLV: Tag–Length–Value 한 바이트씩

BER/DER의 모든 값은 예외 없이 세 부분이다.

```text
 ┌──────────┬───────────┬─────────────────────┐
 │   Tag    │  Length   │       Value         │
 │ (1+ 바이트)│ (1+ 바이트) │   (Length만큼의 옥텟) │
 └──────────┴───────────┴─────────────────────┘
```

구성 타입(SEQUENCE 등)의 Value는 또다시 TLV들의 나열이다. 그래서 DER 한 덩어리는 **TLV의 재귀적 나무**다. 이 나무 구조를 머릿속에 그리는 것이 ASN.1 독해의 전부다.

### 4.1 Tag 바이트의 해부

첫 Tag 바이트(번호 ≤ 30인 흔한 경우)는 다음과 같이 쪼개진다.

```text
   bit8 bit7   bit6   bit5 bit4 bit3 bit2 bit1
  ┌────┬────┬───────┬─────────────────────────┐
  │ class   │  P/C  │      tag number (0–30)   │
  │ (2비트) │ (1비트)│        (5비트)            │
  └─────────┴───────┴─────────────────────────┘
```

- **class (bit 8–7)**: 태그가 어느 네임스페이스에 속하는지.

| 비트 | class | 의미 |
|---|---|---|
| `00` | **universal** | ASN.1 내장 타입(INTEGER, SEQUENCE…). 전 세계 공통 |
| `01` | **application** | 특정 응용/모듈 안에서만 의미 |
| `10` | **context-specific** | 어떤 구조 안의 필드 위치로 의미가 결정됨(X.509의 `[0]`, `[1]`…) |
| `11` | **private** | 사기업/조직 전용 |

- **P/C (bit 6)**: **primitive(0)** 인가 **constructed(1)** 인가. Value가 "날것의 옥텟"이면 primitive, "또 다른 TLV들의 나열"이면 constructed. SEQUENCE·SET은 본디 constructed라 이 비트가 항상 1이다.
- **tag number (bit 5–1)**: 0–30. INTEGER=2, OCTET STRING=4, SEQUENCE=16…

**SEQUENCE의 tag가 왜 0x30인가**를 직접 조립해 보자.

```text
SEQUENCE: universal, constructed, number 16
   class      = 00       (universal)
   P/C        = 1        (constructed)
   number 16  = 1 0000   (5비트)

   조립:  00 | 1 | 10000  =  0011 0000  =  0x30   ✓
```

같은 방식으로:

```text
INTEGER (universal, primitive, 2) :  00 0 00010 = 0000 0010 = 0x02
OCTET STRING (uni, prim, 4)       :  00 0 00100 = 0000 0100 = 0x04
OID (uni, prim, 6)                :  00 0 00110 = 0000 0110 = 0x06
SET (uni, constructed, 17)        :  00 1 10001 = 0011 0001 = 0x31
NULL (uni, prim, 5)               :  00 0 00101 = 0000 0101 = 0x05
BIT STRING (uni, prim, 3)         :  00 0 00011 = 0000 0011 = 0x03

context-specific [0] EXPLICIT (constructed):
   10 1 00000 = 1010 0000 = 0xA0   ← X.509 version 필드
context-specific [2] IMPLICIT IA5String (primitive):
   10 0 00010 = 1000 0010 = 0x82   ← SAN의 dNSName
```

> `0x30`을 보면 "SEQUENCE 시작"이라고 즉시 읽을 수 있어야 한다. 모든 DER 인증서·키·CSR이 `30 82 ...`로 시작하는 이유다(`82`는 길이 long form 표시, 4.3절). hex 덤프를 보고 `30`·`31`·`02`·`06`·`A0`만 식별해도 구조의 8할이 보인다.

### 4.2 high-tag-number form: 번호가 31 이상일 때

tag number 5비트로는 0–30까지밖에 못 적는다. 31 이상은 어떻게? **5비트를 전부 1(11111)로 채워 "확장 신호"를 보내고**, 이어지는 바이트(들)에 실제 번호를 base-128로 적는다 — 이건 정확히 [[05 - Varint·ZigZag·protobuf wire 해부]]의 가변길이 정수다. 단 OID와 마찬가지로 **big-endian(상위 그룹 먼저)**, MIDI VLQ 방식이다.

```text
context-specific, constructed, tag number 1000 을 인코딩

첫 바이트:  10 1 11111  = 1011 1111 = 0xBF   ("확장 신호")
1000을 base-128:
   1000 = 7×128 + 104        → 그룹 7, 104
   바이트:  (7 | 0x80)=0x87,  104=0x68
결과:  BF 87 68
```

검산: 디코더는 첫 바이트의 하위 5비트가 `11111`(=31)임을 보고 "고-태그 형식"을 알아채고, 뒤따르는 `87 68`을 `(0x07<<7) | 0x68 = 896 + 104 = 1000`으로 푼다. 정확하다.

실무에서 high-tag-number form은 드물다. X.509가 쓰는 context 태그 `[0]`–`[3]`은 전부 5비트 안에 들어가므로 한 바이트로 끝난다. 하지만 악의적 입력은 이 형식을 악용해 **거대한 tag number로 정수 오버플로**를 노릴 수 있다(14절).

### 4.3 Length: 세 가지 형식

Value의 길이를 적는 방식이 세 가지다.

**(1) short form** — 길이 < 128: 한 바이트에 그대로. 최상위 비트가 0.

```text
길이 5   →  0x05
길이 127 →  0x7F
```

**(2) long form** — 길이 ≥ 128: 첫 바이트의 최상위 비트를 1로 켜고, 하위 7비트에 "**뒤따르는 길이 바이트의 개수**"를 적는다. 그다음 그 개수만큼의 바이트에 실제 길이를 big-endian으로.

```text
길이 200  → 200=0xC8, 한 바이트면 충분
            0x81 0xC8       (81 = 1000_0001 = "길이 바이트 1개")

길이 435  → 435=0x01B3
            0x82 0x01 0xB3  (82 = "길이 바이트 2개")

길이 1000 → 1000=0x03E8
            0x82 0x03 0xE8

길이 69473 → 0x010F61, 3바이트
            0x83 0x01 0x0F 0x61
```

그래서 큰 인증서가 `30 82 03 4F ...`로 시작하면 "SEQUENCE, 길이는 뒤 2바이트 `03 4F`=847바이트"로 읽는다.

**(3) indefinite form** — **BER 전용**, DER 금지: 길이를 모르고 스트리밍할 때. 첫 바이트를 `0x80`(최상위 1, 하위 0개)으로 적고, 내용 뒤에 **end-of-contents 마커 `00 00`** 을 붙인다.

```text
definite:   SEQUENCE { INTEGER 5 }
            30 03 02 01 05          (길이 3 명시)

indefinite: 30 80 02 01 05 00 00    (BER 전용)
            ↑   ↑           ↑
            │   길이=indefinite  end-of-contents
            SEQUENCE
```

> DER은 indefinite length를 **절대** 쓰지 않는다(definite-only). 그 이유는 8절의 정규형 논의와, 14절의 indefinite-중첩 DoS에서 분명해진다. 거꾸로, 입력에서 `0x80`을 길이 자리에 본 순간 "이건 BER이거나, DER을 가장한 공격"이라고 의심해야 한다.

### 4.4 Value

Value는 Length가 말한 만큼의 옥텟이다. primitive면 날것(예: INTEGER의 2의 보수 바이트), constructed면 그 안이 다시 TLV들의 나열이다. 다음 절부터 타입별로 Value를 손으로 만든다.

---

## 5. 손으로 인코딩하기: 기본 타입들

이제 진짜 바이트를 만든다. 모든 예제는 **DER 규칙**을 따른다.

### 5.1 BOOLEAN

값 하나는 1바이트.

```text
FALSE :  01 01 00
TRUE  :  01 01 FF      ← DER: TRUE는 반드시 0xFF
```

BER은 0이 아닌 어떤 값(`0x01`, `0x7F`…)도 TRUE로 받는다. DER은 **TRUE를 0xFF로 못 박는다**. 왜? 정규형이어야 하니까 — "참"을 적는 방법이 유일해야 같은 값이 같은 바이트, 같은 서명이 된다(8절).

### 5.2 NULL

내용이 없다. 길이 0.

```text
NULL :  05 00
```

`AlgorithmIdentifier`의 parameters 자리에 "파라미터 없음"을 적을 때 자주 등장한다(`RSA + NULL`).

### 5.3 INTEGER — 2의 보수, big-endian, 최소 바이트

INTEGER는 임의 정밀도 정수를 **2의 보수, big-endian**으로, **꼭 필요한 만큼의 바이트만** 쓴다.

```text
0    →  02 01 00
1    →  02 01 01
127  →  02 01 7F
-1   →  02 01 FF
-128 →  02 01 80
```

여기서 두 개의 핵심 규칙이 나온다.

**(a) 양수인데 최상위 비트가 1이면 0x00을 앞에 붙인다.** 안 그러면 음수로 읽힌다.

```text
128  →  0x80 한 바이트로 적으면 -128로 읽힘!
        그래서:  02 02 00 80
255  →  02 02 00 FF
256  →  02 02 01 00
200  →  0xC8(=1100_1000)는 MSB=1 → 음수로 오해
        그래서:  02 02 00 C8
```

**(b) 음수도 최소 바이트로.** `-129`는 1바이트(-128..127)를 벗어나니 2바이트.

```text
-129 → 16비트 2의 보수 = 0xFF7F
       검산: 0xFF7F = 65407, 65407 - 65536 = -129  ✓
       →  02 02 FF 7F
```

이때 "선행 0xFF를 더 떼어낼 수 있지 않나?"를 막는 규칙이 DER에 있다. X.690 8.3.2:

> INTEGER 내용이 2바이트 이상이면, **첫 옥텟의 모든 비트와 둘째 옥텟의 bit 8이 (a) 모두 1이어서도 안 되고 (b) 모두 0이어서도 안 된다.**

`FF 7F`은 둘째 옥텟 bit8이 0이므로 첫 `FF`를 못 뗀다 → 최소다. 반대로 `00 80`(=128)은 둘째 bit8이 1이므로 `00`을 떼면 음수가 되어 못 뗀다 → 역시 최소다. 이 규칙이 **양수의 불필요한 `00 ...`과 음수의 불필요한 `FF ...`을 동시에 금지**한다. 이것이 "INTEGER 최소 인코딩"의 정확한 의미다.

```python
def der_integer(n: int) -> bytes:
    # 2의 보수 최소 바이트 길이 계산
    if n == 0:
        body = b"\x00"
    else:
        length = (n.bit_length() + 8) // 8   # 부호비트 여유 포함
        body = n.to_bytes(length, "big", signed=True)
        # to_bytes가 이미 최소 길이를 주지만, 안전하게 선행 중복 제거
        while len(body) > 1 and (
            (body[0] == 0x00 and body[1] & 0x80 == 0) or
            (body[0] == 0xFF and body[1] & 0x80 != 0)
        ):
            body = body[1:]
    return bytes([0x02, len(body)]) + body
```

> "INTEGER는 항상 4/8바이트"라고 생각하기 쉽다. ASN.1 INTEGER는 임의 정밀도다. RSA 모듈러스(2048비트)는 INTEGER 한 개이고 그 길이는 257바이트(앞에 `00` 한 바이트 붙어서)다. 시리얼 번호도 큰 INTEGER다. 고정폭 정수의 직관([[01 - 비트·바이트·엔디안]])을 여기서 버려야 한다.

### 5.4 OCTET STRING

날 바이트 열. 그대로 담는다.

```text
바이트 "Hi" (0x48 0x69) → 04 02 48 69
빈 OCTET STRING        → 04 00
```

### 5.5 BIT STRING — 첫 바이트는 "unused bit 개수"

BIT STRING은 비트 단위 길이를 다룬다. 그래서 Value의 **첫 옥텟이 "마지막 바이트에서 안 쓰는 비트 수(0–7)"** 이고, 그다음이 실제 비트들(MSB부터 채움)이다.

X.690의 정전 예제, 18비트 `'011011100101110111'B`를 인코딩해 보자.

```text
비트:  0110 1110  0101 1101  11
       (18비트 → 3옥텟=24비트, 6비트가 남음)

옥텟으로(MSB부터, 빈 자리는 0):
   옥텟1: 0110 1110 = 0x6E
   옥텟2: 0101 1101 = 0x5D
   옥텟3: 11 00 0000 = 0xC0    ← 뒤 6비트는 패딩
   unused = 6

DER:  03 04 06 6E 5D C0
      │  │  │  └──────── 비트 데이터 3옥텟
      │  │  └─────────── unused=6
      │  └────────────── 길이 4 (1 + 3)
      └───────────────── BIT STRING
```

X.509에서 BIT STRING은 두 군데 핵심에 쓰인다. **subjectPublicKey**(공개키 자체)와 **signatureValue**(서명값)다. 이 둘은 보통 unused=0이라 `03 ... 00 ...`으로 시작한다.

```text
서명값(예시, 256바이트 RSA 서명):
   03 82 01 01 00 <256바이트 서명>
   ↑           ↑
   BIT STRING  unused=0
   길이=0x0101=257 (1바이트 unused + 256바이트)
```

> subjectPublicKey BIT STRING 맨 앞의 `00`을 "정수 선행 0"으로 착각하기 쉽다. 그건 **unused-bit 개수**(=0)다. 그리고 그 BIT STRING의 *내용*은 또다시 DER(예: RSA면 `SEQUENCE { modulus INTEGER, exponent INTEGER }`)이다 — BIT STRING이 DER을 감싸는 포장지다.

**DER의 named-bit 규칙**: 비트들에 이름이 붙은 경우(예: KeyUsage), **뒤쪽의 0 비트를 모두 제거**해야 한다(X.690 11.2). KeyUsage에서 `keyCertSign(5)`+`cRLSign(6)`만 켠 CA 인증서를 보자. 비트 n은 첫 옥텟의 (왼쪽부터) n번째 위치다 — bit0=0x80, bit1=0x40, …, bit5=0x04, bit6=0x02, bit7=0x01.

```text
keyCertSign(5)=0x04, cRLSign(6)=0x02  →  0x06
가장 높은 켜진 비트가 6번 → 0..6까지 7비트 필요 → unused=1
DER:  03 02 01 06
```

실제 CA 인증서의 KeyUsage가 정확히 `03 02 01 06`("Certificate Sign, CRL Sign")으로 나오는 이유다.

### 5.6 ENUMERATED

INTEGER와 똑같이 인코딩하되 tag만 `0x0A`. 예: CRLReason의 `keyCompromise(1)`.

```text
ENUMERATED 1 → 0A 01 01
```

### 5.7 문자열들

내용은 해당 문자집합의 바이트, tag만 다르다.

```text
PrintableString "US" :  13 02 55 53           ('U'=0x55, 'S'=0x53)
IA5String "a@b.com"  :  16 07 61 40 62 2E 63 6F 6D
UTF8String "Example" :  0C 07 45 78 61 6D 70 6C 65
UTF8String "한"       :  0C 03 ED 95 9C          (U+D55C → UTF-8 ED 95 9C)
BMPString "AB"       :  1E 04 00 41 00 42       (UCS-2 big-endian)
```

`"한"`의 UTF-8 바이트가 왜 `ED 95 9C`인지는 [[08 - 유니코드 정규화와 grapheme cluster]] 계열의 UTF-8 인코딩 규칙이다(U+D55C는 3바이트 시퀀스). `IA5String`은 `@`를 허용하지만 `PrintableString`은 불허라는 점, `BMPString`이 2바이트 고정이라 ASCII도 `00 41`처럼 적힌다는 점을 눈여겨보라.

### 5.8 시각: UTCTime와 GeneralizedTime

**UTCTime**(tag 0x17)은 2자리 연도. DER에서는 형식이 `YYMMDDHHMMSSZ`로 고정 — 초까지 적고 끝에 `Z`(UTC).

```text
2023-08-15 12:00:00 UTC → "230815120000Z" (13글자)
   17 0D 32 33 30 38 31 35 31 32 30 30 30 30 5A
   │  │  '2''3''0''8''1''5''1''2''0''0''0''0''Z'
   │  길이 0x0D=13
   UTCTime
```

**GeneralizedTime**(tag 0x18)은 4자리 연도 `YYYYMMDDHHMMSSZ`.

```text
2023-08-15 12:00:00 UTC → "20230815120000Z" (15글자)
   18 0F 32 30 32 33 30 38 31 35 31 32 30 30 30 30 5A
```

> 2자리 연도는 악명 높은 함정이다. UTCTime의 `YY`는 19xx인가 20xx인가? RFC 5280은 **YY ≥ 50 → 19YY, YY < 50 → 20YY**로 못 박았다(2050년이 분기점). 그래서 만료일이 **2049년까지는 UTCTime, 2050년부터는 GeneralizedTime**을 쓰도록 규정한다. "100년 유효" 루트 인증서가 GeneralizedTime을 쓰는 이유다. 두 표현이 섞여 있어서, 파서는 같은 validity 안에서도 notBefore/notAfter의 tag가 다를 수 있음을 가정해야 한다.

---

## 6. OBJECT IDENTIFIER: OID를 바이트로

OID는 ASN.1에서 가장 우아하고, 가장 많이 틀리는 부분이다. `1.2.840.113549.1.1.11` 같은 점으로 잇는 정수 열(아크, arc)이고, 전 세계가 합의한 **계층적 이름 등록 트리**다.

### 6.1 두 가지 인코딩 트릭

규칙은 두 줄이다.

1. **첫 두 아크 X.Y는 하나의 정수 `40×X + Y`로 합친다.**
2. **각 아크(합쳐진 첫 아크 포함)는 base-128 big-endian varint로 적는다** — 7비트씩, 마지막 바이트만 MSB=0([[05 - Varint·ZigZag·protobuf wire 해부]]의 MIDI VLQ와 동일).

왜 첫 두 아크를 합치나? 최상위 아크(root)는 0·1·2 셋뿐이라(`itu-t`·`iso`·`joint-iso-itu-t`) 정보량이 작다. 둘째 아크도 root가 0·1일 땐 0–39로 제한된다. 그래서 `40×X+Y` 한 정수에 두 아크가 손실 없이 들어간다.

### 6.2 손계산 — `1.2.840.113549` (RSA/PKCS의 루트)

```text
아크:  1 . 2 . 840 . 113549

(1) 첫 두 아크 합치기:  40×1 + 2 = 42 = 0x2A

(2) 840을 base-128:
    840 = 6×128 + 72   → 그룹 [6, 72]
    바이트: (6|0x80)=0x86,  72=0x48        →  86 48

(3) 113549를 base-128:
    113549 ÷ 128 = 887  나머지 13
    887    ÷ 128 = 6    나머지 119
    6      ÷ 128 = 0    나머지 6
    그룹(상위부터): [6, 119, 13]
    바이트: (6|0x80)=0x86, (119|0x80)=0xF7, 13=0x0D  →  86 F7 0D

값 옥텟:  2A  86 48  86 F7 0D
TLV:      06 06 2A 86 48 86 F7 0D
          │  │
          │  길이 6
          OBJECT IDENTIFIER
```

`2A 86 48 86 F7 0D` — 이 6바이트는 PKCS 인증서 어디서나 보이는 RSA의 지문이다. hex 덤프에서 `2A 86 48 86 F7 0D`를 보면 "아, RSA 계열 OID구나"라고 즉시 알 수 있다.

### 6.3 손계산 — `2.5.4.3` (commonName, CN)

```text
아크:  2 . 5 . 4 . 3

(1) 40×2 + 5 = 85 = 0x55
(2) 4 = 0x04
(3) 3 = 0x03

값:  55 04 03
TLV: 06 03 55 04 03
```

CN을 가리키는 `55 04 03`은 인증서의 subject/issuer 이름 어디서나 나온다. 마찬가지로 `2.5.4.x`(X.500 디렉터리 속성)는 전부 `55 04 ..`로 시작한다.

| 속성 | OID | DER 값 |
|---|---|---|
| commonName (CN) | 2.5.4.3 | `55 04 03` |
| countryName (C) | 2.5.4.6 | `55 04 06` |
| organizationName (O) | 2.5.4.10 | `55 04 0A` |
| organizationalUnit (OU) | 2.5.4.11 | `55 04 0B` |
| localityName (L) | 2.5.4.7 | `55 04 07` |
| stateOrProvince (ST) | 2.5.4.8 | `55 04 08` |

### 6.4 둘째 아크가 39를 넘는 경우 (root=2)

root가 `2`(joint-iso-itu-t)면 둘째 아크가 39를 넘을 수 있다. 그때도 `40×2+Y`가 한 정수가 되고, 그게 127을 넘으면 base-128로 여러 바이트가 된다. X.690 표준 자신의 예제 `{joint-iso-itu-t 100 3}`:

```text
40×2 + 100 = 180
180 = 1×128 + 52  → [1, 52] → (1|0x80)=0x81, 52=0x34  →  81 34
다음 아크 3 = 0x03

값:  81 34 03
TLV: 06 03 81 34 03
```

`60 86 48 ...`로 시작하는 OID(예: NIST의 SHA-256 OID `2.16.840.1.101.3.4.2.1` → `60 86 48 01 65 03 04 02 01`)도 같은 원리다: `40×2+16=96=0x60`.

### 6.5 자주 보는 OID 사전

| OID | 이름 | DER 값 옥텟 |
|---|---|---|
| 1.2.840.113549.1.1.1 | rsaEncryption | `2A 86 48 86 F7 0D 01 01 01` |
| 1.2.840.113549.1.1.11 | sha256WithRSAEncryption | `2A 86 48 86 F7 0D 01 01 0B` |
| 1.2.840.10045.2.1 | id-ecPublicKey | `2A 86 48 CE 3D 02 01` |
| 1.2.840.10045.3.1.7 | prime256v1 (P-256) | `2A 86 48 CE 3D 03 01 07` |
| 2.5.29.17 | subjectAltName | `55 1D 11` |
| 2.5.29.19 | basicConstraints | `55 1D 13` |
| 2.5.29.15 | keyUsage | `55 1D 0F` |
| 2.5.29.35 | authorityKeyIdentifier | `55 1D 23` |
| 2.5.29.14 | subjectKeyIdentifier | `55 1D 0E` |

`2.5.29.x`(id-ce, 인증서 확장)는 전부 `55 1D ..`로 시작한다. 이 패턴(`55 04`=속성, `55 1D`=확장, `2A 86 48 86 F7 0D`=RSA계열, `2A 86 48 CE 3D`=EC계열)만 외워도 인증서 hex가 훨씬 잘 읽힌다.

> OID 인코딩은 [[05 - Varint·ZigZag·protobuf wire 해부]]의 가변길이 정수가 1984년에 이미 쓰이고 있었다는 증거다. 단 protobuf의 LEB128이 little-endian인 데 비해 OID는 **big-endian base-128**(MIDI VLQ와 같은 방향)이라는 점만 다르다. "흔한 작은 아크는 1바이트, 드문 큰 아크는 여러 바이트"라는 빈도-기반 발상은 동일하다.

---

## 7. SEQUENCE·SET과 중첩: Name 20바이트 완전 분해

이제 구성 타입으로 나무를 만든다. X.509의 `Name`(issuer/subject)을 통째로 손으로 빌드해 보자. `Name`은 이렇게 정의된다.

```text
Name              ::= RDNSequence
RDNSequence       ::= SEQUENCE OF RelativeDistinguishedName
RelativeDistinguishedName ::= SET OF AttributeTypeAndValue
AttributeTypeAndValue ::= SEQUENCE { type OBJECT IDENTIFIER, value ANY }
```

세 겹이다: SEQUENCE OF(RDN 목록) → SET OF(한 RDN 안의 속성들) → SEQUENCE(타입+값). `CN = Example` 하나짜리 이름을 안쪽부터 조립한다.

```text
[1] AttributeTypeAndValue = SEQUENCE { CN-OID, UTF8String "Example" }
    CN OID:            06 03 55 04 03                         (5바이트)
    UTF8String:        0C 07 45 78 61 6D 70 6C 65             (9바이트)
    내용 합 = 14 = 0x0E
    →  30 0E 06 03 55 04 03 0C 07 45 78 61 6D 70 6C 65        (16바이트)

[2] RelativeDistinguishedName = SET OF { 위 SEQUENCE }
    내용 = 위 16바이트 = 0x10
    →  31 10 30 0E 06 03 55 04 03 0C 07 45 78 61 6D 70 6C 65  (18바이트)

[3] RDNSequence = SEQUENCE OF { 위 SET }
    내용 = 위 18바이트 = 0x12
    →  30 12 31 10 30 0E 06 03 55 04 03 0C 07 45 78 61 6D 70 6C 65  (20바이트)
```

완성된 20바이트를 나무로 그리면:

```text
30 12                                  SEQUENCE (RDNSequence), len 18
└─ 31 10                               SET (RelativeDistinguishedName), len 16
   └─ 30 0E                            SEQUENCE (AttributeTypeAndValue), len 14
      ├─ 06 03 55 04 03                OBJECT IDENTIFIER = commonName (2.5.4.3)
      └─ 0C 07 45 78 61 6D 70 6C 65    UTF8String = "Example"
                  E  x  a  m  p  l  e
```

이 20바이트(`30 12 31 10 30 0E 06 03 55 04 03 0C 07 45 78 61 6D 70 6C 65`)가 곧 `CN=Example`이라는 이름의 정식 DER이다. 인증서에서 issuer와 subject가 같은 구조로 두 번 나온다.

> RDN이 왜 SET OF인가. 한 "이름 성분(RDN)"에 여러 속성을 묶을 수 있다(예: `CN=Foo + serialNumber=123`이 동시에 한 RDN). SET이므로 순서가 의미 없고, 따라서 DER은 그 안을 정렬해야 한다(8절). 보통은 RDN 하나에 속성 하나라서 정렬이 눈에 안 띄지만, 다속성 RDN에서는 결정적이다.

---

## 8. BER vs CER vs DER: 정규형이라는 계약

BER은 같은 값을 여러 방식으로 적을 수 있다. TRUE를 `01`로도 `FF`로도, 길이를 short로도 long으로도, indefinite로도. 이 **자유가 서명에는 독**이다. 서명은 "이 바이트열의 해시"에 거는 것인데, 같은 논리값이 여러 바이트가 될 수 있으면 서명을 만든 쪽과 검증하는 쪽이 다른 바이트를 해시할 수 있다. 그러면 검증이 깨지거나, 더 나쁘게는 **공격자가 서명을 우회**한다.

**DER**(Distinguished Encoding Rules)은 BER의 자유를 전부 못 박아, **어떤 값이든 정확히 하나의 바이트열**만 갖게 한다(canonical form). DER의 규칙을 모으면:

| 규칙 | BER | DER |
|---|---|---|
| 길이 형식 | short/long/indefinite 자유 | **definite만, 최소 바이트**(길이 5는 `05`, `81 05` 금지) |
| BOOLEAN TRUE | 0이 아닌 아무 값 | **반드시 `FF`** |
| INTEGER | 선행 `00`/`FF` 허용 | **최소 바이트**(5.3절 규칙) |
| BIT STRING unused 비트 | 임의 | **반드시 0**, named-bit는 뒤 0비트 제거 |
| 문자열 분할(constructed) | 가능 | **불가**(반드시 primitive 단일 옥텟열) |
| SET OF 순서 | 임의 | **인코딩 오름차순 정렬** |
| SET 성분 순서 | 임의 | **태그 오름차순** |
| DEFAULT 값 | 적어도 됨 | **생략**(기본값과 같으면 안 적음) |
| 시각 | 다양한 형식 | **`...SSZ` 고정**, 분수초 trailing 0 금지 |

**CER**은 또 다른 정규형인데, 반대 선택을 한다: 긴 구성 값에는 indefinite length를 쓰고 긴 문자열을 1000옥텟 청크로 자른다. **스트리밍**(길이를 미리 모르는 큰 값)에 유리하다. DER은 정반대로 전부 definite라서 **저장·서명**에 유리하다. 그래서 X.509는 DER, 일부 메시징은 CER을 쓴다.

### 8.1 SET OF 정렬을 손으로

DER의 SET OF는 "성분들의 **DER 인코딩을 옥텟 문자열로 보고 오름차순 정렬**"한다(X.690 11.6). 짧은 쪽은 뒤를 0으로 채워 비교한다. `SET OF INTEGER { 10, 5, 256 }`을 정렬해 보자.

```text
각 INTEGER의 DER:
   5   → 02 01 05
   10  → 02 01 0A
   256 → 02 02 01 00

옥텟열로 사전식 비교:
   02 01 05  ← 셋째 바이트 05
   02 01 0A  ← 셋째 바이트 0A  (05 < 0A)
   02 02 01 00 ← 둘째 바이트 02 (> 01)
정렬 결과:  5, 10, 256

SET OF:  31 0A  02 01 05  02 01 0A  02 02 01 00
         │  └ 길이 10
         SET
```

같은 세 정수라도 입력 순서가 어떻든 DER은 항상 이 한 가지 바이트열이다. 그게 정규형의 힘이다.

### 8.2 DEFAULT 생략 — version 필드

TBSCertificate의 `version`은 `[0] EXPLICIT Version DEFAULT v1`이다. v1(=INTEGER 0)이면 기본값이므로 **필드 전체를 생략**한다. v3(=INTEGER 2)이면 적는다.

```text
v3 인증서:  A0 03 02 01 02     (version 필드 존재)
v1 인증서:  (version 필드 없음 → serialNumber가 바로 옴)
```

그래서 v1 인증서를 파싱하면 SEQUENCE 첫 성분이 `A0...`이 아니라 곧장 `02 ...`(serialNumber)다. 파서는 "`A0`로 시작하면 version, 아니면 v1"로 분기한다.

> 서명은 왜 DER이어야 하는가. 인증서의 서명은 `signatureValue = Sign(SHA256(DER(tbsCertificate)))`이다. 검증자는 받은 인증서에서 tbsCertificate 부분의 바이트를 **그대로 다시 해시**해서 비교한다. 만약 인코딩이 정규형이 아니라면, "의미는 같지만 바이트가 다른" tbsCertificate가 가능해지고, 서명은 바이트에 걸리므로 깨진다. 그래서 X.509는 "tbsCertificate는 DER이어야 한다"를 강제한다. 더 무서운 건 **파서가 BER 관용을 부려 비정규 인코딩을 받아주면**, 공격자가 서명된 정규 바이트와 다른 변형을 같은 의미로 통과시킬 여지가 생긴다는 점이다(14절 BERserk).

---

## 9. 태깅: IMPLICIT vs EXPLICIT, context-specific

SEQUENCE에 OPTIONAL 필드가 여럿 있으면 문제가 생긴다. 두 OPTIONAL 필드가 같은 타입(둘 다 INTEGER)이면, 파서가 "지금 나온 INTEGER가 어느 필드냐"를 구별할 수 없다. 해결책이 **태깅** — 필드마다 context-specific 번호 `[0]`, `[1]`…을 붙여 위치를 태그에 새긴다.

```text
Example ::= SEQUENCE {
    a  [0] INTEGER OPTIONAL,
    b  [1] INTEGER OPTIONAL
}
```

이제 `[0]`이 보이면 a, `[1]`이면 b다. 태깅에는 두 방식이 있다.

### 9.1 EXPLICIT — 원래 타입을 그대로 감싼다

EXPLICIT 태깅은 원래 TLV를 통째로, **새 태그로 한 번 더 포장**한다. INTEGER 2를 `[0] EXPLICIT`로 태깅:

```text
원래:      02 01 02                  (INTEGER 2)
[0] EXPLICIT로 포장:
           A0 03 02 01 02
           │  │  └────────── 원래 INTEGER TLV 그대로
           │  길이 3
           context [0], constructed (포장지라 constructed!)
```

`A0`는 "context-specific(10), constructed(1), 번호 0"이다(4.1절). 안에 원래 INTEGER가 온전히 들어 있어 **타입 정보가 보존**된다. X.509 version이 정확히 이 형식(`A0 03 02 01 02` = v3).

### 9.2 IMPLICIT — 원래 태그를 갈아끼운다

IMPLICIT 태깅은 포장하지 않고 **원래 타입의 태그 바이트만 context 태그로 교체**한다. 더 짧다.

```text
원래:      02 01 02                  (INTEGER 2, tag 02)
[0] IMPLICIT로 태깅:
           80 01 02
           │  │  └─ 값 02 (그대로)
           │  길이 1
           context [0], primitive (INTEGER는 primitive라 0 유지)
```

`80`은 "context(10), primitive(0), 번호 0". 5바이트(EXPLICIT) vs 3바이트(IMPLICIT)다. 작지만 보존을 포기한다 — 받는 쪽은 스키마를 봐야만 "이 `[0]`이 사실 INTEGER였다"를 안다.

| | EXPLICIT | IMPLICIT |
|---|---|---|
| 바이트 | 더 김(원래 TLV를 감쌈) | 더 짧음(태그만 교체) |
| 원래 타입 보존 | 예(안에 원래 태그) | 아니오(스키마 필요) |
| constructed 비트 | 항상 constructed | 원래 타입을 따름 |
| CHOICE/ANY에 적용 | 가능 | **불가**(원래 태그가 사라지면 대안 구별 불가) |

### 9.3 X.509에서의 혼용

X.509(RFC 5280)는 둘을 섞어 쓴다. 모듈 기본은 EXPLICIT이지만 일부 필드는 IMPLICIT으로 지정돼 있다.

```text
TBSCertificate ::= SEQUENCE {
   version        [0] EXPLICIT Version DEFAULT v1,        -- A0 03 02 01 02
   serialNumber       CertificateSerialNumber,           -- 02 ...
   signature          AlgorithmIdentifier,
   issuer             Name,
   validity           Validity,
   subject            Name,
   subjectPublicKeyInfo SubjectPublicKeyInfo,
   issuerUniqueID  [1] IMPLICIT UniqueIdentifier OPTIONAL, -- 81 ... (BIT STRING)
   subjectUniqueID [2] IMPLICIT UniqueIdentifier OPTIONAL, -- 82 ...
   extensions      [3] EXPLICIT Extensions OPTIONAL        -- A3 ...
}
```

`version`은 INTEGER를 안전하게 감싸려 EXPLICIT(`A0`), `extensions`도 SEQUENCE를 감싸려 EXPLICIT(`A3`), 반면 uniqueID는 BIT STRING이라 IMPLICIT(`81`/`82`)으로 한 바이트 아꼈다.

> CHOICE를 IMPLICIT 태깅하면 안 된다. CHOICE는 자기 태그가 없고 "선택된 대안의 태그"로 자신을 드러내는데, IMPLICIT이 그 태그를 덮어쓰면 어느 대안인지 알 길이 사라진다. 그래서 컴파일러는 CHOICE·ANY·열린 타입에는 자동으로 EXPLICIT을 강제한다. X.509의 `Time ::= CHOICE { utcTime, generalTime }`이 태깅 없이 raw하게 쓰이는 이유다.

---

## 10. X.509 v3 인증서 해부

이제 모든 조각을 모아 진짜 인증서를 읽는다. 최상위 구조는 단 세 성분의 SEQUENCE다.

```text
Certificate ::= SEQUENCE {
    tbsCertificate       TBSCertificate,      -- 서명 대상 본문
    signatureAlgorithm   AlgorithmIdentifier, -- 서명에 쓴 알고리즘(중복 기재)
    signatureValue       BIT STRING           -- 서명값
}
```

`tbsCertificate`("**T**o **B**e **S**igned")가 본문이고, CA가 이 본문의 DER을 해시·서명해 `signatureValue`에 넣는다. `signatureAlgorithm`은 tbs 안의 `signature` 필드와 같은 값을 한 번 더 적는데, 이 중복은 "본문 밖에서 알고리즘을 바꿔치기하는 공격"을 막기 위한 것이다(둘이 다르면 거부).

### 10.1 TBSCertificate 본문을 손으로 빌드

앞 절들에서 만든 조각으로 TBS의 앞부분을 실제 바이트로 조립하자.

```text
version (v3):
   A0 03 02 01 02

serialNumber (INTEGER 13):
   02 01 0D

signature (AlgorithmIdentifier: sha256WithRSAEncryption, NULL):
   30 0D
      06 09 2A 86 48 86 F7 0D 01 01 0B      (OID, 6.5절)
      05 00                                  (NULL parameters)

issuer (Name: CN=Example):
   30 12 31 10 30 0E 06 03 55 04 03 0C 07 45 78 61 6D 70 6C 65   (7절)

validity (SEQUENCE { UTCTime, UTCTime }):
   30 1E
      17 0D 32 33 30 31 30 31 30 30 30 30 30 30 5A   (notBefore 230101000000Z)
      17 0D 32 34 30 31 30 31 30 30 30 30 30 30 5A   (notAfter  240101000000Z)

subject (Name: 보통 자기 CN, issuer와 같은 구조)
   30 12 31 10 30 0E 06 03 55 04 03 0C 07 ...

subjectPublicKeyInfo (SEQUENCE { AlgorithmIdentifier, BIT STRING })
   30 ..
      30 0D 06 09 2A 86 48 86 F7 0D 01 01 01 05 00   (rsaEncryption, NULL)
      03 82 01 0F 00 30 82 01 0A 02 82 01 01 00 ...  (공개키를 감싼 BIT STRING)

extensions (11절)
   A3 ..
      30 ..  ...
```

`validity`의 두 UTCTime 길이를 검산하면, 각 `17 0D ...`가 2+13=15바이트이고 둘이면 30=0x1E. 그래서 `30 1E`. 정확하다. `signature`의 SEQUENCE 내용은 OID(11바이트)+NULL(2바이트)=13=0x0D → `30 0D`. 역시 맞는다.

### 10.2 subjectPublicKeyInfo의 이중 포장

공개키 부분에서 ASN.1의 "포장 안의 포장"이 잘 드러난다.

```text
SubjectPublicKeyInfo ::= SEQUENCE {
   algorithm        AlgorithmIdentifier,   -- 어떤 키 종류인가
   subjectPublicKey BIT STRING             -- 키의 실제 바이트
}
```

RSA라면 `subjectPublicKey` BIT STRING의 *내용*이 또다시 DER이다:

```text
03 82 01 0F 00  30 82 01 0A  02 82 01 01 00 <256바이트 모듈러스>  02 03 01 00 01
│           │   └ RSAPublicKey = SEQUENCE { modulus, publicExponent }
│           unused=0
│           └ BIT STRING 길이 0x010F=271
BIT STRING

내부 SEQUENCE 풀어보면:
   30 82 01 0A           SEQUENCE (RSAPublicKey), len 266
      02 82 01 01 00 ..  INTEGER modulus (선행 00 + 256바이트 = 2048비트)
      02 03 01 00 01     INTEGER publicExponent = 65537 (0x010001)
```

`publicExponent`가 `02 03 01 00 01`인 게 보인다 — 65537 = 0x010001, 최상위 바이트 0x01의 MSB가 0이라 선행 0 불필요, 그냥 3바이트(5.3절). 거의 모든 RSA 키가 이 `02 03 01 00 01`로 끝난다. EC 키라면 algorithm이 `id-ecPublicKey + 곡선 OID`이고 BIT STRING이 곡선 위 점(`04 || X || Y`)을 담는다.

### 10.3 개념적 asn1parse 덤프

위에서 손으로 만든 앞부분을 `openssl asn1parse`로 보면 이렇게 나온다(offset은 TBS 시작 기준).

```text
    0:d=0  hl=2 l=   3 cons: cont [ 0 ]                 ; A0 03  (version 래퍼)
    2:d=1  hl=2 l=   1 prim: INTEGER           :02      ; 02 01 02  (v3)
    5:d=0  hl=2 l=   1 prim: INTEGER           :0D      ; 02 01 0D  (serial=13)
    8:d=0  hl=2 l=  13 cons: SEQUENCE                   ; 30 0D
   10:d=1  hl=2 l=   9 prim: OBJECT            :sha256WithRSAEncryption
   21:d=1  hl=2 l=   0 prim: NULL                       ; 05 00
   23:d=0  hl=2 l=  18 cons: SEQUENCE                   ; 30 12  (issuer)
   25:d=1  hl=2 l=  16 cons: SET                        ; 31 10
   27:d=2  hl=2 l=  14 cons: SEQUENCE                   ; 30 0E
   29:d=3  hl=2 l=   3 prim: OBJECT            :commonName
   34:d=3  hl=2 l=   7 prim: UTF8STRING        :Example
```

`d`는 깊이, `hl`은 헤더 길이(tag+length 바이트 수), `l`은 Value 길이다. offset 29의 OID(`06 03 55 04 03`)와 offset 34의 UTF8String(`0C 07 ...`)이 7절에서 손으로 만든 그 20바이트와 정확히 일치한다 — `30 12`가 offset 23, `31 10`이 25, `30 0E`가 27, OID가 29, 문자열이 34. 손계산과 도구가 한 치도 안 어긋난다.

---

## 11. 확장(Extensions): 이중 포장의 비밀

X.509 v3의 힘은 **확장(extensions)**에 있다. SAN(도메인 목록), KeyUsage, BasicConstraints(CA 여부) 등이 전부 확장이다. 구조는:

```text
extensions ::= [3] EXPLICIT SEQUENCE OF Extension
Extension  ::= SEQUENCE {
    extnID     OBJECT IDENTIFIER,
    critical   BOOLEAN DEFAULT FALSE,
    extnValue  OCTET STRING            -- 확장값의 DER을 OCTET STRING으로 감쌈!
}
```

핵심은 `extnValue`가 **OCTET STRING이고, 그 안에 또 확장별 DER이 들어간다**는 것. 왜 이 이중 포장인가? **확장성** 때문이다. 파서는 모르는 확장을 만나도 "OCTET STRING 한 덩어리"로 건너뛸 수 있다 — 안의 내용을 해석할 필요 없이 길이만큼 건너뛰면 된다. 아는 확장만 OCTET STRING을 열어 안의 DER을 다시 파싱한다. critical=TRUE인데 모르는 확장이면 인증서를 거부해야 한다는 규칙(RFC 5280)도 이 구조 덕에 깔끔히 동작한다.

### 11.1 BasicConstraints (CA:TRUE) 손계산

```text
안쪽 값 BasicConstraints ::= SEQUENCE { cA BOOLEAN }
   CA:TRUE →  30 03 01 01 FF          (cA=TRUE)

extnValue OCTET STRING으로 감싸기:
   04 05 30 03 01 01 FF               (OCTET STRING len 5 = 위 5바이트)

extnID (basicConstraints 2.5.29.19):  06 03 55 1D 13
critical TRUE:                        01 01 FF

Extension = SEQUENCE { extnID, critical, extnValue }
   내용 = (5) + (3) + (7) = 15 = 0x0F
   →  30 0F 06 03 55 1D 13 01 01 FF 04 05 30 03 01 01 FF
```

나무로:

```text
30 0F                          SEQUENCE (Extension)
├─ 06 03 55 1D 13              OID basicConstraints (2.5.29.19)
├─ 01 01 FF                    BOOLEAN critical = TRUE
└─ 04 05                       OCTET STRING (extnValue), len 5
   └─ 30 03                    SEQUENCE (BasicConstraints)   ← 또 DER!
      └─ 01 01 FF              BOOLEAN cA = TRUE
```

`04 05`(OCTET STRING) 안에 `30 03 ...`(또 다른 SEQUENCE)이 들어 있는 게 이중 포장이다.

### 11.2 KeyUsage 손계산

```text
KeyUsage BIT STRING { keyCertSign(5), cRLSign(6) } → 03 02 01 06   (5.5절)
extnValue OCTET STRING:  04 04 03 02 01 06
extnID keyUsage (2.5.29.15):  06 03 55 1D 0F
critical TRUE:  01 01 FF

Extension:
   내용 = (5)+(3)+(6) = 14 = 0x0E
   →  30 0E 06 03 55 1D 0F 01 01 FF 04 04 03 02 01 06
```

### 11.3 SubjectAltName (SAN) — `example.com`이 hex로 어떻게 보이나

SAN은 오늘날 인증서에서 가장 중요한 확장이다(브라우저는 CN이 아니라 SAN으로 호스트를 검증한다). 구조:

```text
SubjectAltName ::= GeneralNames ::= SEQUENCE OF GeneralName
GeneralName ::= CHOICE {
   ...
   dNSName  [2] IMPLICIT IA5String,   -- 도메인은 여기
   iPAddress[7] IMPLICIT OCTET STRING,
   ...
}
```

`dNSName`이 `[2] IMPLICIT IA5String`이므로, 도메인 문자열의 태그가 IA5String(`16`)이 아니라 **context `[2]` primitive(`82`)** 로 바뀐다(9.2절).

```text
"example.com" (11바이트):
   65 78 61 6D 70 6C 65 2E 63 6F 6D
   e  x  a  m  p  l  e  .  c  o  m

dNSName [2] IMPLICIT:  82 0B 65 78 61 6D 70 6C 65 2E 63 6F 6D

GeneralNames = SEQUENCE OF:  30 0D 82 0B ...   (내용 13 = 0x0D)

extnValue OCTET STRING:  04 0F 30 0D 82 0B 65 78 61 6D 70 6C 65 2E 63 6F 6D

extnID SAN (2.5.29.17):  06 03 55 1D 11
(critical 생략 — SAN은 보통 non-critical)

Extension:
   내용 = (5) + (17) = 22 = 0x16
   →  30 16 06 03 55 1D 11 04 0F 30 0D 82 0B 65 78 61 6D 70 6C 65 2E 63 6F 6D
```

나무로:

```text
30 16                                    SEQUENCE (Extension)
├─ 06 03 55 1D 11                        OID subjectAltName (2.5.29.17)
└─ 04 0F                                 OCTET STRING (extnValue), len 15
   └─ 30 0D                              SEQUENCE (GeneralNames)
      └─ 82 0B 65 78 ... 6F 6D           [2] dNSName = "example.com"
```

hex 덤프에서 `82 0B 65 78 61 6D 70 6C 65 ...`를 보면 "아, SAN의 도메인이 여기 있다"를 바로 안다. `82`(context [2])가 곧 dNSName의 표식이다. 여러 도메인이면 `82 ..` `82 ..`가 줄줄이 이어진다.

> "도메인은 인증서 안에 문자열로 그냥 있겠지"라고 생각하면 `16`(IA5String)을 찾는다. 실제로는 IMPLICIT 태깅 때문에 `82`로 나타난다. IMPLICIT가 태그를 갈아끼운다는 9.2절의 규칙이 여기서 실전으로 보인다.

---

## 12. PEM: DER을 텍스트로

DER은 바이너리라 이메일·설정파일·복사붙여넣기에 부적합하다. **PEM**(Privacy-Enhanced Mail, RFC 7468)은 DER 바이트를 [[10 - 체크섬 내장 인코딩 - Base58·Bech32]]에서 다룬 **base64**로 인코딩하고 사람이 읽는 머리/꼬리줄로 감싼 것이다.

```text
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJA0... (base64, 보통 64자마다 줄바꿈) ...
...wIDAQAB
-----END CERTIFICATE-----
```

규칙은 단순하다.

1. 객체의 DER을 만든다.
2. 그 바이트를 base64(표준 알파벳 `A–Z a–z 0–9 + /`, 패딩 `=`)로 인코딩.
3. 64자마다 줄바꿈(RFC 7468 권장).
4. `-----BEGIN <라벨>-----` / `-----END <라벨>-----`로 감싼다.

### 12.1 base64를 손으로 — `30 03 01 01 FF`

11.1절에서 만든 BasicConstraints 안쪽 5바이트 `30 03 01 01 FF`를 base64로 바꿔 보자. base64는 3바이트(24비트)를 4글자(6비트씩)로 바꾼다.

```text
바이트:  30        03        01       | 01        FF
         00110000  00000011  00000001 | 00000001  11111111

[첫 3바이트 30 03 01 → 24비트]
   001100 000000 001100 000001
     12     0      12     1
   base64:  M      A      M      B          ("MAMB")

[남은 2바이트 01 FF → 16비트, 0으로 2비트 패딩 → 18비트]
   000000 011111 111100   (마지막 6비트 자리는 패딩 처리)
     0     31     60   + '='
   base64:  A      f      8      =          ("Af8=")

전체:  MAMBAf8=
```

base64 인덱스 검산: `M`=12, `A`=0, `B`=1(A=0,B=1…), `f`=31(a=26,…,f=31), `8`=60(0=52,…,8=60). 디코더가 `MAMBAf8=`를 되돌리면 정확히 `30 03 01 01 FF`. base64의 6비트↔8비트 비트 재배열은 [[01 - 비트·바이트·엔디안]]의 비트 슬라이싱 그 자체다.

### 12.2 PEM 라벨들

라벨(`-----BEGIN <X>-----`의 X)이 안에 든 객체의 종류를 알린다.

| 라벨 | 내용 | DER 구조 |
|---|---|---|
| `CERTIFICATE` | X.509 인증서 | `Certificate` |
| `CERTIFICATE REQUEST` | CSR | PKCS#10 `CertificationRequest` |
| `X509 CRL` | 폐기 목록 | `CertificateList` |
| `PUBLIC KEY` | 공개키 | `SubjectPublicKeyInfo`(SPKI) |
| `PRIVATE KEY` | 개인키(알고리즘 무관) | PKCS#8 `PrivateKeyInfo` |
| `ENCRYPTED PRIVATE KEY` | 암호화된 개인키 | PKCS#8 `EncryptedPrivateKeyInfo` |
| `RSA PRIVATE KEY` | RSA 개인키 | PKCS#1 `RSAPrivateKey` |
| `EC PRIVATE KEY` | EC 개인키 | SEC1 `ECPrivateKey` |
| `PKCS7` | 서명/봉투 데이터 | PKCS#7/CMS |

### 12.3 한 파일에 여러 객체 — 체인

PEM은 BEGIN/END 블록을 **여러 개 이어 붙일 수 있다**. TLS 서버가 보내는 인증서 체인이 대표적이다.

```text
-----BEGIN CERTIFICATE-----   ← 리프(서버) 인증서
...
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----   ← 중간 CA
...
-----END CERTIFICATE-----
```

파서는 BEGIN/END 쌍을 순서대로 읽는다. base64 바깥의 텍스트(블록 사이 주석 등)는 무시한다. 반면 **DER 파일(.der/.cer)** 은 바이너리 한 객체만 담는다 — 여러 객체를 한 DER 파일에 담으려면 PKCS#7 같은 컨테이너가 필요하다.

| | PEM | DER |
|---|---|---|
| 형식 | base64 텍스트 + 헤더 | 순수 바이너리 |
| 확장자 | `.pem .crt .cer .key` | `.der .cer` |
| 여러 객체 | 가능(블록 이어붙임) | 한 객체(컨테이너 필요) |
| 편집기/이메일 | 안전 | 깨짐 |
| 크기 | DER의 약 137%(base64 + 헤더) | 최소 |

> PEM은 "새 인코딩"이 아니라 **DER을 텍스트 채널로 안전히 나르는 봉투**일 뿐이다. 안의 진실은 항상 DER이다. `openssl x509 -inform PEM -outform DER`로 봉투만 벗기면 똑같은 바이트가 나온다. 그래서 PEM↔DER 변환은 base64 인코딩/디코딩이 전부다.

---

## 13. PKCS 가족과 친척 포맷

X.509 둘레에는 ASN.1로 정의된 여러 표준이 위성처럼 돈다. PKCS(Public-Key Cryptography Standards, RSA사 제정 → 다수 RFC화)가 핵심이다.

| 표준 | 정의(RFC) | 내용 | ASN.1 최상위 타입 |
|---|---|---|---|
| **PKCS#1** | RFC 8017 | RSA 키·서명·암호화 | `RSAPublicKey`, `RSAPrivateKey` |
| **PKCS#7 / CMS** | RFC 5652 | 서명·봉투 데이터 컨테이너 | `ContentInfo`, `SignedData` |
| **PKCS#8** | RFC 5958 | 알고리즘 무관 개인키 | `OneAsymmetricKey`(`PrivateKeyInfo`) |
| **PKCS#10** | RFC 2986 | 인증서 서명 요청(CSR) | `CertificationRequest` |
| **PKCS#12** | RFC 7292 | 키+인증서+체인 묶음(`.pfx`/`.p12`) | `PFX` |
| **SPKI** | RFC 5280 | 공개키 포장 | `SubjectPublicKeyInfo` |
| **SEC1** | SEC1 v2 | EC 개인키 | `ECPrivateKey` |

### 13.1 PKCS#8 — 키 포맷의 통일

PKCS#1은 RSA 전용이라 `RSAPrivateKey`의 첫 INTEGER가 곧장 모듈러스다. EC·Ed25519 등이 등장하면서 "알고리즘 무관" 래퍼가 필요해졌고, 그게 PKCS#8이다.

```text
PrivateKeyInfo ::= SEQUENCE {
    version              INTEGER,
    privateKeyAlgorithm  AlgorithmIdentifier,   -- 어떤 알고리즘 키인가
    privateKey           OCTET STRING           -- 알고리즘별 키 DER을 감쌈
}
```

여기서도 **OCTET STRING 안에 또 DER**(11절과 같은 패턴)이다. RSA면 그 안에 PKCS#1 `RSAPrivateKey`가, EC면 SEC1 `ECPrivateKey`가 들어간다. 그래서 같은 RSA 키가 `BEGIN RSA PRIVATE KEY`(PKCS#1)일 수도, `BEGIN PRIVATE KEY`(PKCS#8로 한 겹 더 감쌈)일 수도 있다.

### 13.2 CSR — 서명을 자기가 한다

PKCS#10 CSR은 "이 공개키로 인증서를 발급해 달라"는 요청이다. 구조가 인증서와 닮았다.

```text
CertificationRequest ::= SEQUENCE {
    certificationRequestInfo  SEQUENCE { version, subject, SPKI, attributes },
    signatureAlgorithm        AlgorithmIdentifier,
    signature                 BIT STRING
}
```

차이는 서명 주체다. 인증서는 **CA가** tbsCertificate를 서명하지만, CSR은 **신청자 본인이** 자기 개인키로 certificationRequestInfo를 서명한다("이 공개키의 짝 개인키를 내가 갖고 있다"는 증명, proof of possession). CA는 이 자가서명을 검증한 뒤, 내용을 골라 새 인증서를 만들어 자기 키로 다시 서명한다.

### 13.3 PKCS#12 — 비밀번호로 묶은 보따리

`.pfx`/`.p12`는 개인키 + 인증서 + 체인을 한 파일에 담고 비밀번호로 암호화한 묶음이다. 윈도/맥의 키체인 가져오기·내보내기가 이 형식을 쓴다. 내부는 PKCS#7 컨테이너 + PKCS#8 키 + 암호화 계층이 ASN.1로 중첩된 꽤 복잡한 나무다. "한 파일로 다 들고 다니기"의 편의 대신 구조 복잡도와 (약한 기본 암호 설정 시) 보안 위험을 진다.

---

## 14. 파싱은 전장이다: 함정과 CVE

ASN.1 파서는 신뢰할 수 없는 입력(인증서·서명·LDAP 요청)을 직접 받는 최전선이다. 그래서 역사적으로 보안 사고의 단골이다. [[06 - CRC의 수학과 체크섬]]이 "우발적 손상"을 다뤘다면, 여기서는 **악의적 입력**이 인코딩의 어느 빈틈을 찌르는지 본다.

### 14.1 길이 필드를 믿어서 생기는 오버리드/오버플로

가장 흔한 부류. 파서가 length 필드를 읽고 그만큼 메모리를 할당/복사하는데, **실제 버퍼보다 큰 길이**를 검증 없이 믿으면 힙 오버리드나 오버플로가 난다.

```text
공격 입력:  04 84 7F FF FF FF <실제로는 8바이트만 존재>
            │  │  └──────────── "길이 약 21억" 주장
            │  long form, 길이 바이트 4개
            OCTET STRING

순진한 파서:  malloc(0x7FFFFFFF) 또는 buf+len 까지 읽기 → 폭발
```

방어: **선언된 길이 ≤ 남은 버퍼**를 항상 먼저 검사. long form 길이 바이트가 정수 타입을 넘으면(예: 5바이트 이상) 즉시 거부. "길이는 입력이 주장하는 값일 뿐, 사실이 아니다"가 철칙이다. Heartbleed(TLS heartbeat)도 본질은 "선언 길이 ≠ 실제 길이"였고, 같은 부류의 실수가 ASN.1 파서에 수없이 났다.

### 14.2 BER indefinite length 중첩 DoS

BER의 indefinite length(`0x80 ... 00 00`, 4.3절)는 깊이 제한이 없으면 **무한 중첩**으로 스택을 터뜨린다.

```text
30 80 30 80 30 80 30 80 ... (수만 겹) ... 00 00 00 00 ...
재귀 파서가 깊이 N으로 들어가 스택 오버플로(DoS)
```

방어: **DER은 indefinite 자체를 금지**(definite-only)하므로 X.509 파서는 `0x80`을 길이 자리에서 보면 즉시 거부한다. BER을 받아야 하는 LDAP/SNMP 파서는 **최대 중첩 깊이**(예: 수십)를 강제한다.

### 14.3 OID 아크 정수 오버플로

OID 아크는 base-128 varint라 길이 제한이 없다(6절). 거대한 아크를 보내 파서의 정수 변수를 오버플로시키면, 다른 OID로 오인되거나 계산이 어긋난다.

```text
06 06 2A 86 48 FF FF 7F   ← 마지막 아크가 32비트를 넘김
파서가 u32에 누적하면 wrap-around → 엉뚱한 OID로 해석
```

방어: 아크 누적을 임의 정밀도(또는 64비트 + 범위 검사)로, 그리고 **OID 정규화**(아는 OID 테이블과 바이트 단위로 비교, 디코드 후 재비교 금지)로 처리한다. "OID는 디코드해서 비교하지 말고 바이트로 비교하라"가 좋은 습관이다.

### 14.4 CN 안의 NUL 바이트 (null-prefix 공격)

2009년 Moxie Marlinspike가 시연한 고전. 공격자가 CA에 `CN = "www.bank.com\0.attacker.com"`으로 CSR을 낸다. CA가 `attacker.com` 소유만 검증하고 발급하면, **C 문자열로 CN을 다루는** 클라이언트는 `\0`에서 잘라 `www.bank.com`으로 본다.

```text
ASN.1 문자열:  ... 77 77 77 2E 62 61 6E 6B 2E 63 6F 6D 00 2E 61 74 ...
                  w  w  w  .  b  a  n  k  .  c  o  m  \0 .  a  t ...
길이 기반 ASN.1: "www.bank.com\0.attacker.com" (NUL은 그냥 한 바이트)
C 문자열 파서:    "www.bank.com"               (NUL에서 종료)  ← 불일치!
```

근본 원인은 **ASN.1 문자열은 길이 접두라 NUL을 데이터로 허용**하는데, 검증 코드가 NUL-종료 C 문자열로 다뤘다는 표현 불일치다. 관련 CVE: CVE-2009-2408(NSS/Firefox) 등. 방어: 길이 기반으로 끝까지 비교, CN/SAN에 제어문자·NUL 금지. 같은 부류의 **시각적 혼동**(키릴 `а` vs 라틴 `a`)은 [[09 - 유니코드 보안 - confusables와 homograph]]에서 다룬 homograph 문제와 직결된다.

### 14.5 BERserk — 너그러운 파싱이 서명을 위조한다 (CVE-2014-1568)

가장 교훈적인 사례. PKCS#1 v1.5 서명 검증은 복호화 후 나오는 `DigestInfo`(`SEQUENCE { 해시알고리즘, OCTET STRING 해시 }`)를 파싱한다. RSA 공개지수 `e=3`인 키에서, **파서가 DigestInfo 뒤의 잉여 바이트를 무시하거나 길이를 느슨하게 검사**하면, 공격자는 세제곱근을 맞춰 **개인키 없이 서명을 위조**할 수 있다(Bleichenbacher 2006 공격의 실전판).

```text
정상 검증: 복호 결과가 [패딩 || DigestInfo] 정확히여야 함
취약 파서: [패딩 || DigestInfo || 임의 잉여바이트]를 통과시킴
   → 공격자가 잉여 자리를 조작해 완전세제곱수를 만들어 e=3 서명 위조
```

8절의 "DER 엄격성 = 보안"이 여기서 가장 극적으로 드러난다. **인코딩을 정확히, 끝까지, 정규형으로 검사**하지 않으면 서명의 수학이 멀쩡해도 시스템이 뚫린다. 방어: DigestInfo를 바이트 단위로 재구성해 정확히 일치하는지 비교(파싱이 아니라 비교), 트레일링 바이트 0 허용 금지, 가능하면 RSA-PSS 사용.

### 14.6 그 밖의 빈틈

| 함정 | 메커니즘 | 방어 |
|---|---|---|
| 비정규 DER | 선행 0 INTEGER, indefinite, 비최소 길이 통과 | DER 엄격 검증, 비정규 거부 |
| UTCTime 2자리 연도 | 2050 분기 오해로 만료 판정 오류 | RFC 5280 분기 규칙 준수, 신규는 GeneralizedTime |
| 문자열 타입 혼동 | PrintableString vs UTF8String 비교 누락 | 정규화 후 비교, 타입 명시 |
| 음의 시리얼/0 시리얼 | INTEGER 음수·0 시리얼로 충돌 유발 | 양의 0이 아닌 시리얼 강제(RFC 5280) |
| 깊은 중첩 | 재귀 파서 스택 고갈 | 깊이 제한 |

> ASN.1 파서를 직접 쓰지 말라. 검증된 라이브러리를 쓰되, 그마저도 [[01 - 비트·바이트·엔디안]]에서 강조한 "라이브러리 불신" 원칙으로 대하라. 길이는 검증 전엔 거짓말일 수 있고, 인코딩은 정규형이 아닐 수 있으며, 문자열은 NUL을 품을 수 있다. **모든 입력 길이는 남은 버퍼와 대조, 모든 서명 대상은 DER 정규형으로 강제, 모든 OID는 바이트로 비교**.

---

## 15. 도구: 바이트를 사람의 말로

마지막으로 실무에서 ASN.1을 들여다보는 도구들. 손계산을 검증하고 디버깅할 때 필수다.

### 15.1 openssl asn1parse — 만능 TLV 덤퍼

```text
$ openssl asn1parse -in cert.pem -i

    0:d=0  hl=4 l= 871 cons: SEQUENCE
    4:d=1  hl=4 l= 591 cons:  SEQUENCE
    8:d=2  hl=2 l=   3 cons:   cont [ 0 ]
   10:d=3  hl=2 l=   1 prim:    INTEGER           :02
   13:d=2  hl=2 l=  20 prim:   INTEGER           :0A2B...   (serial)
   35:d=2  hl=2 l=  13 cons:   SEQUENCE
   37:d=3  hl=2 l=   9 prim:    OBJECT            :sha256WithRSAEncryption
   ...
```

`-i`는 깊이별 들여쓰기, `-strparse <offset>`은 그 offset의 OCTET STRING 안을 다시 파싱(11절 이중 포장을 열 때 유용)한다. `cons`/`prim`이 4.1절의 constructed/primitive 비트다.

### 15.2 openssl x509 -text — 사람이 읽는 해석

```text
$ openssl x509 -in cert.pem -text -noout

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: ...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=Example
        Validity:
            Not Before: Jan  1 00:00:00 2023 GMT
            Not After : Jan  1 00:00:00 2024 GMT
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Subject Alternative Name:
                DNS:example.com
```

`asn1parse`가 "바이트의 구조"라면 `x509 -text`는 "그 구조의 의미"다. 10·11절에서 손으로 만든 필드들이 여기 사람 말로 나타난다 — `CA:TRUE`(`30 03 01 01 FF`), `Certificate Sign, CRL Sign`(`03 02 01 06`), `DNS:example.com`(`82 0B ...`).

### 15.3 dumpasn1 / der2ascii

| 도구 | 만든이 | 쓰임 |
|---|---|---|
| `dumpasn1` | Peter Gutmann | 가장 상세한 TLV 덤프(OID·태그 주석 풍부) |
| `der2ascii` / `ascii2der` | Google | DER ↔ 사람이 편집 가능한 텍스트(테스트 케이스 제작) |
| `openssl asn1parse` | OpenSSL | 빠른 구조 확인 |
| `xxd` / `hexdump` | 표준 | 날 hex(태그를 직접 식별할 때) |

`der2ascii`는 특히 **악의적/엣지 케이스 인증서를 손으로 만들** 때 좋다. `SEQUENCE { INTEGER { 0x00 0x80 } }` 같은 비정규 인코딩을 텍스트로 적어 `ascii2der`로 바이트를 뽑고, 파서가 거부하는지 테스트한다.

```text
# der2ascii 텍스트 예
SEQUENCE {
  OBJECT_IDENTIFIER { 2.5.4.3 }   # commonName
  UTF8String { "Example" }
}
# → ascii2der로 컴파일하면
#   30 0E 06 03 55 04 03 0C 07 45 78 61 6D 70 6C 65   (7절의 그 바이트!)
```

---

## 16. 정리: ASN.1이 가르쳐 준 것

ASN.1/DER을 한 바이트씩 따라오며 우리는 직렬화 설계의 거의 모든 원형을 만났다.

```text
ASN.1/DER 한 장 요약
─────────────────────────────────────────────────────────────
추상 vs 인코딩   X.680(타입)과 X.690(바이트)의 분리 — 핵심 사상
TLV             모든 값 = Tag(class·P/C·번호) + Length + Value, 재귀 나무
Tag             0x30=SEQUENCE 0x02=INTEGER 0x06=OID 0xA0=[0] 0x82=[2]
Length          <128 short / 81·82.. long / 80..0000 indefinite(BER만)
INTEGER         2의보수 big-endian 최소바이트, 양수 MSB=1이면 00 선행
OID             40X+Y 합치고 base-128 big-endian varint (varint 05장)
DER             정규형: definite·최소길이·TRUE=FF·SET OF 정렬·DEFAULT 생략
태깅            EXPLICIT(감쌈, 보존) vs IMPLICIT(태그 교체, 짧음)
X.509           Certificate{ tbs, sigAlg, sigValue }, 확장은 OCTET STRING 이중포장
PEM             DER을 base64로(10장) + BEGIN/END 봉투
보안            길이 불신·indefinite 금지·OID 바이트비교·NUL 차단·DER 엄격(BERserk)
```

이 장의 교훈을 시리즈의 다른 점들과 잇자.

- **가변길이 정수는 1984년에 이미 있었다.** OID의 base-128 인코딩은 [[05 - Varint·ZigZag·protobuf wire 해부]]의 varint와 같은 발상이다. 방향(big vs little endian)만 다르다.
- **정규형은 보안이다.** DER이 BOOLEAN TRUE를 `0xFF`로 못 박고 길이를 최소화하는 이유는 "같은 값 → 같은 바이트 → 같은 서명"을 위해서다. [[06 - CRC의 수학과 체크섬]]에서 체크섬이 위변조를 못 막는다고 했듯, 무결성과 정규형은 별개의 보장이고 둘 다 필요하다.
- **포장 안의 포장.** extnValue가 OCTET STRING 안에 또 DER을 품는 구조는 "모르면 건너뛰기(skip), 알면 열기(parse)"라는 확장성 패턴이다. 같은 패턴이 PKCS#8 키, BIT STRING 공개키에 반복된다.
- **텍스트 봉투.** PEM은 DER을 [[10 - 체크섬 내장 인코딩 - Base58·Bech32]]의 base64로 감싼 운반 수단일 뿐, 진실은 항상 바이너리 DER이다.
- **현대의 후예.** [[12 - CBOR·Avro·zero-copy 직렬화]]는 TLV의 자기서술(CBOR)과 스키마 분리·태그 없는 압축(Avro)을 현대적으로 다시 쓴다. ASN.1의 BER(자기서술)과 PER(스키마 의존 압축)이 그 두 갈래의 원조다.

ASN.1은 못생겼고, 파서는 위험하며, 문자열 타입은 일곱 가지다. 그럼에도 이 포맷이 인터넷의 신뢰를 떠받치는 이유는, **플랫폼 독립적 구조화 데이터**라는 문제를 인류가 처음으로 진지하게, 그리고 끝까지 풀어낸 결과물이기 때문이다. 다음에 브라우저 자물쇠를 클릭해 인증서를 열어 본다면, 그 뒤에 `30 82 ...`로 시작하는 한 그루의 TLV 나무가 서 있음을, 그리고 이제 그 나무를 가지마다 바이트로 읽을 수 있음을 떠올리자.
