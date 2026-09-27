---
title: ASN.1·DER·PEM - 인증서를 바이트로
date: 2026-06-26
tags: [encoding, asn1, der, pem, x509, tls, 학습노트]
---

이 시리즈는 지금까지 "값 하나를 어떻게 바이트로 적는가"를 한 단계씩 쌓아 왔다. [[01 - 비트·바이트·엔디안]]에서는 바이트와 엔디안을, [[05 - Varint·ZigZag·protobuf wire 해부]]에서는 가변길이 정수를, [[10 - 체크섬 내장 인코딩 - Base58·Bech32]]에서는 사람이 읽고 옮겨 적을 수 있는 텍스트 인코딩을 직접 풀어 보았다. 이 장에서는 그 조각들이 모두 한자리에 모인다.

**ASN.1/DER**은 가변길이 정수(OID), 길이 접두(length-prefix), 중첩 구조, 그리고 텍스트 포장(PEM의 base64)을 모두 한 포맷 안에 담는다. 인터넷 신뢰 체계의 기반인 X.509 인증서가 바로 이 포맷 위에 만들어져 있다.

직전 장인 10장(체크섬 내장 인코딩)이 "DER 바이트열을 base64로 감싼 것"인 PEM의 텍스트 쪽 절반을 이미 다뤘다면, 이 장에서는 그 안에 들어 있는 바이너리 쪽 절반을 바이트 단위로 분석한다. 다음 장인 [[12 - CBOR·Avro·zero-copy 직렬화]]에서는 ASN.1과 같은 "스키마 기반 바이너리 직렬화"라는 큰 흐름을 현대적으로 다시 구현한 포맷들로 넘어간다. TLV를 이어받은 CBOR와 스키마 분리를 이어받은 Avro가 그 예이다.

즉 ASN.1은 과거의 유물이 아니라, 우리가 매일 쓰는 직렬화 방식의 **원형(prototype)** 이다. protobuf의 wire 포맷과 비교하고 싶다면 gRPC 시리즈의 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]을 함께 보면 좋다.

가장 추상적인 질문에서 출발해 가장 구체적인 바이트까지 차례로 내려가 보자.

---

## 1. ASN.1은 어디에나 있다: 동기와 역사

### 1.1 1984년의 문제

1980년대 초, 서로 다른 컴퓨터들이 네트워크로 데이터를 주고받기 시작하면서 근본적인 장벽에 부딪혔다. 한쪽은 big-endian이고 다른 쪽은 little-endian이었다(1장 비트·바이트·엔디안). 한쪽은 EBCDIC을, 다른 쪽은 ASCII를 썼다. 한쪽의 `int`는 16비트였고 다른 쪽은 36비트였다. 두 기계가 **같은 의미를 같은 바이트로 표현하기로 합의**하지 못하면 "사용자 이름과 나이를 보내라"는 단순한 요구조차 처리할 수 없었다.

CCITT(현 ITU-T)와 ISO는 이 문제를 두 단계로 나누어 풀었다. 이렇게 나눈 것이 ASN.1의 전부라 해도 과언이 아니다.

1. **무엇을 보내는가(추상 문법, abstract syntax)**: "이 메시지는 UTF8 문자열 하나와 정수 하나로 이루어진 레코드다" 같은 **타입 정의**이다. 기계 표현과 관계없이 의미만 기술한다.
2. **그것을 어떻게 바이트로 적는가(전송 문법, transfer syntax / 인코딩 규칙)**: 위 타입을 실제 옥텟(octet) 열로 바꾸는 규칙이다.

ASN.1(**A**bstract **S**yntax **N**otation **One**)은 1번을 위한 언어다. 1984년 X.409로 처음 나왔고, X.208을 거쳐 오늘날의 ITU-T **X.680** 계열로 자리 잡았다. 2번은 별개의 표준으로, **X.690**(BER/CER/DER), **X.691**(PER), **X.693**(XER) 등이 있다. "Notation **One**"이라는 이름 자체가 "이것은 표기법일 뿐 인코딩이 아니다"라는 뜻을 담고 있다.

### 1.2 왜 아직도 ASN.1인가

JSON, protobuf, MessagePack이 넘쳐나는 시대에 ASN.1은 왜 아직도 쓰일까? 답은 단순하다. **이미 구축된 인프라가 너무 크기 때문이다.**

| 시스템 | ASN.1이 쓰이는 곳 |
|---|---|
| **X.509 / TLS** | 모든 HTTPS 인증서, 인증서 체인, CRL, OCSP 응답 |
| **PKCS** | 키 포맷(#1 RSA, #8 PKCS8), CSR(#10), 서명/봉투(#7/CMS), 키 묶음(#12 .pfx) |
| **LDAP** | 디렉터리 프로토콜의 모든 요청/응답(BER) |
| **SNMP** | 네트워크 장비 관리 MIB·PDU(BER) |
| **Kerberos** | 티켓·인증자(authenticator) 구조(DER) |
| **이동통신** | 3G/4G/5G의 시그널링(NAS/RRC는 PER로 비트 단위까지 압축) |
| **전자여권(eMRTD)** | ICAO 9303 칩의 데이터그룹 |
| **EMV** | 칩카드 결제 메시지의 TLV(ASN.1 BER-TLV 계열) |

이 가운데 핵심은 **X.509**다. 지금 이 노트를 읽는 동안에도 브라우저는 수십 개의 ASN.1/DER 인증서를 파싱하고 있다. 인터넷 PKI(공개키 기반구조) 전체가 이 포맷 위에 만들어져 있으므로, ASN.1 파서의 버그는 곧 인터넷의 보안 사고로 이어진다(14절).

> ASN.1은 "타입 시스템"이고 X.690은 "직렬화 규칙"이다. 둘을 한데 묶어 "ASN.1 = DER"이라고 생각하는 순간부터 거의 모든 혼란이 시작된다. ASN.1은 protobuf의 `.proto` 문법([[02 - Protocol Buffers 1 - 문법과 타입 시스템]])에, DER은 protobuf의 wire 포맷(gRPC 시리즈 3장 Protocol Buffers 2)에 대응한다고 보면 정확하다.

---

## 2. 두 개의 표준: 추상 문법(X.680)과 인코딩 규칙(X.690)

### 2.1 추상 문법: 타입을 적는 언어

ASN.1 모듈은 아래와 같이 생겼다. 익숙한 구조체나 레코드 정의처럼 읽힌다.

```text
Person ::= SEQUENCE {
    name      UTF8String,
    age       INTEGER (0..150),
    email     IA5String OPTIONAL,
    married   BOOLEAN DEFAULT FALSE
}
```

`::=`는 "정의된다"는 뜻이고, `SEQUENCE`는 순서가 있는 필드 묶음(구조체), `OPTIONAL`은 생략할 수 있는 필드, `DEFAULT`는 기본값을 뜻한다. `(0..150)`은 값의 범위를 제한하는 **서브타입 제약(subtype constraint)** 이다. 여기에는 바이트에 관한 내용이 한 줄도 없다. `age`가 1바이트인지 4바이트인지, big-endian인지는 **인코딩 규칙이 결정한다**.

### 2.2 인코딩 규칙: 같은 타입, 여러 바이트열

X.690과 관련 표준들은 위의 `Person { name="Al", age=42 }`라는 값 하나를 서로 다른 바이트열로 만든다.

| 인코딩 규칙 | 표준 | 성격 | 한 마디 |
|---|---|---|---|
| **BER** (Basic) | X.690 | TLV, 선택지 많음 | 가장 너그러움. 같은 값에 여러 표현 허용 |
| **CER** (Canonical) | X.690 | BER의 정규 부분집합 | 스트리밍용. 긴 값은 indefinite + 1000옥텟 청크 |
| **DER** (Distinguished) | X.690 | BER의 정규 부분집합 | **저장·서명용**. 같은 값 → 유일한 바이트열 |
| **PER** (Packed) | X.691 | 비트 단위 압축 | TLV 안 씀. 5G/항공이 대역폭 아끼려 사용 |
| **OER** (Octet) | X.696 | 옥텟 정렬 압축 | PER보다 빠른 파싱, 적당한 크기 |
| **XER** (XML) | X.693 | XML 텍스트 | 디버깅·상호운용 |
| **JER** (JSON) | X.697 | JSON 텍스트 | 현대적 상호운용 |

이 장은 **BER과 그 정규형인 DER**에 집중한다. X.509, PKCS, Kerberos는 모두 DER을 쓰고, BER은 LDAP와 SNMP가 쓴다. PER은 비트 단위로 인코딩하면서 TLV를 아예 쓰지 않는다. 그래서 양쪽이 스키마를 똑같이 알고 있어야만 풀 수 있는, 자기서술적(self-describing)이지 않은 인코딩이다. 이 방식은 12장(CBOR·Avro·zero-copy 직렬화)에서 다루는 Avro(스키마 분리, 태그 없는 인코딩)와 매우 닮았다.

> TLV 계열(BER/DER)은 **자기서술적**이다. 스키마가 없어도 "여기에 길이 9짜리 OBJECT IDENTIFIER가 있다"는 것까지는 파싱할 수 있다. 반면 PER/Avro는 스키마가 없으면 한 비트도 해석할 수 없는 대신 크기가 더 작다. 자기서술과 압축 사이의 이 트레이드오프는 직렬화 포맷 설계에서 늘 고민해야 하는 축이다.

---

## 3. ASN.1의 여러 타입

DER로 내려가기 전에 어떤 타입들이 있는지 분류해 두자. 타입마다 **universal tag number**가 정해져 있고(4절), 그 번호가 그대로 바이트가 된다.

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
| CHOICE | - | 여러 대안 중 하나(자기 tag 없음, 선택된 타입의 tag를 그대로 씀) |

`SEQUENCE`와 `SET`의 tag 번호는 각각 16과 17이지만, 항상 **구성(constructed)** 타입이므로 6번 비트가 켜져서 `0x30`과 `0x31`로 나타난다(이유는 4.1절에서 설명한다).

### 3.3 문자열 타입이 이렇게 많은 이유

`UTF8String`, `PrintableString`, `IA5String`, `BMPString`처럼 같은 "문자열"인데 왜 이렇게 종류가 많을까? 역사적인 이유가 크다. ASN.1이 만들어진 1984년에는 유니코드가 없었다. 텔렉스와 전화망 시절의 제한된 문자 집합들(`PrintableString`은 전신 단말에서 인쇄할 수 있는 글자)이 그대로 굳어졌고, X.509는 이 유산을 그대로 물려받았다.

```text
PrintableString 허용 문자(딱 이만큼):
   A–Z  a–z  0–9  (공백)  ' ( ) + , - . / : = ?
   → '@', '_', '*', '&' 등은 불허! 이메일·언더스코어 불가
```

이 미묘한 차이는 보안 문제로 번진다. CN(common name)이 `PrintableString`인지, `UTF8String`인지, `IA5String`인지에 따라 같은 글자도 다른 바이트로 인코딩된다.

비교나 검증 로직이 이 차이를 무시하면 **homograph·confusable 공격**의 표면이 된다([[09 - 유니코드 보안 - confusables와 homograph]]). RFC 5280은 이 혼란을 줄이기 위해 "2004년 이후 발급하는 인증서의 DirectoryString에는 가급적 `UTF8String`을 쓰라"고 명시했다.

> "ASN.1 문자열은 다 같은 것 아니냐"고 흔히 생각한다. 그렇지 않다. tag 번호가 다르면 **DER 바이트가 다르고**, DER이 다르면 **서명 해시가 다르다**. `PrintableString "US"`(13 02 55 53)와 `UTF8String "US"`(0C 02 55 53)는 서로 다른 인증서를 만든다. 발급할 때 어떤 타입으로 적었는지가 영구히 남는다.

---

## 4. TLV: Tag–Length–Value 한 바이트씩

BER/DER의 모든 값은 예외 없이 세 부분으로 이루어진다.

```text
 ┌──────────┬───────────┬─────────────────────┐
 │   Tag    │  Length   │       Value         │
 │ (1+ 바이트)│ (1+ 바이트) │   (Length만큼의 옥텟) │
 └──────────┴───────────┴─────────────────────┘
```

구성 타입(SEQUENCE 등)의 Value는 다시 TLV들의 나열이다. 따라서 DER 한 덩어리는 **TLV가 재귀적으로 이어진 트리**다. 이 트리 구조를 머릿속에 그릴 수 있으면 ASN.1은 거의 다 읽을 수 있다.

### 4.1 Tag 바이트의 구조

첫 Tag 바이트는 (번호가 30 이하인 흔한 경우) 다음과 같이 나뉜다.

```text
   bit8 bit7   bit6   bit5 bit4 bit3 bit2 bit1
  ┌────┬────┬───────┬─────────────────────────┐
  │ class   │  P/C  │      tag number (0–30)   │
  │ (2비트) │ (1비트)│        (5비트)            │
  └─────────┴───────┴─────────────────────────┘
```

- **class (bit 8–7)**: 태그가 어느 네임스페이스에 속하는지를 나타낸다.

| 비트 | class | 의미 |
|---|---|---|
| `00` | **universal** | ASN.1 내장 타입(INTEGER, SEQUENCE…). 전 세계 공통 |
| `01` | **application** | 특정 응용/모듈 안에서만 의미 |
| `10` | **context-specific** | 어떤 구조 안의 필드 위치로 의미가 결정됨(X.509의 `[0]`, `[1]`…) |
| `11` | **private** | 사기업/조직 전용 |

- **P/C (bit 6)**: **primitive(0)** 인지 **constructed(1)** 인지를 나타낸다. Value가 가공하지 않은 옥텟이면 primitive이고, 다른 TLV들의 나열이면 constructed이다. SEQUENCE와 SET은 원래 constructed이므로 이 비트가 항상 1이다.
- **tag number (bit 5–1)**: 0–30의 값이다. INTEGER=2, OCTET STRING=4, SEQUENCE=16…

**SEQUENCE의 tag가 왜 0x30인지** 직접 조립해 보자.

```text
SEQUENCE: universal, constructed, number 16
   class      = 00       (universal)
   P/C        = 1        (constructed)
   number 16  = 1 0000   (5비트)

   조립:  00 | 1 | 10000  =  0011 0000  =  0x30   ✓
```

다른 타입도 같은 방식으로 조립한다.

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

> `0x30`을 보면 곧바로 "SEQUENCE 시작"이라고 읽을 수 있어야 한다. 모든 DER 인증서, 키, CSR이 `30 82 ...`로 시작하는 이유가 여기에 있다(`82`는 길이의 long form 표시다. 4.3절 참고). hex 덤프에서 `30`, `31`, `02`, `06`, `A0`만 구별해도 구조의 80%가 보인다.

### 4.2 high-tag-number form: 번호가 31 이상일 때

tag number 5비트로는 0–30까지만 적을 수 있다. 31 이상은 어떻게 적을까? **5비트를 모두 1(11111)로 채워 "확장 신호"를 보내고**, 이어지는 바이트(들)에 실제 번호를 base-128로 적는다. 이것은 5장(Varint·ZigZag·protobuf wire)에서 다룬 가변길이 정수와 같은 방식이다. 다만 OID와 마찬가지로 **big-endian(상위 그룹 먼저)** 순서를 쓰는 MIDI VLQ 방식이다.

```text
context-specific, constructed, tag number 1000 을 인코딩

첫 바이트:  10 1 11111  = 1011 1111 = 0xBF   ("확장 신호")
1000을 base-128:
   1000 = 7×128 + 104        → 그룹 7, 104
   바이트:  (7 | 0x80)=0x87,  104=0x68
결과:  BF 87 68
```

검산해 보자. 디코더는 첫 바이트의 하위 5비트가 `11111`(=31)인 것을 보고 high-tag-number form임을 알아챈다. 그리고 뒤따르는 `87 68`을 `(0x07<<7) | 0x68 = 896 + 104 = 1000`으로 해석한다. 결과가 정확히 맞는다.

실무에서 high-tag-number form은 드물다. X.509가 쓰는 context 태그 `[0]`–`[3]`은 모두 5비트 안에 들어가므로 한 바이트로 끝난다. 하지만 악의적인 입력은 이 형식을 이용해 **거대한 tag number로 정수 오버플로**를 노릴 수 있다(14절).

### 4.3 Length: 세 가지 형식

Value의 길이를 적는 방식은 세 가지다.

**(1) short form**: 길이가 128보다 작으면 한 바이트에 그대로 적는다. 최상위 비트는 0이다.

```text
길이 5   →  0x05
길이 127 →  0x7F
```

**(2) long form**: 길이가 128 이상이면 첫 바이트의 최상위 비트를 1로 켜고, 하위 7비트에 "**뒤따르는 길이 바이트의 개수**"를 적는다. 그다음 그 개수만큼의 바이트에 실제 길이를 big-endian으로 적는다.

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

따라서 큰 인증서가 `30 82 03 4F ...`로 시작하면 "SEQUENCE이고, 길이는 뒤의 2바이트 `03 4F`, 즉 847바이트"라고 읽는다.

**(3) indefinite form**: **BER 전용**이며 DER에서는 금지된다. 길이를 모르는 상태로 스트리밍할 때 쓴다. 첫 바이트를 `0x80`(최상위 비트 1, 하위 값 0개)으로 적고, 내용 뒤에 **end-of-contents 마커 `00 00`** 을 붙인다.

```text
definite:   SEQUENCE { INTEGER 5 }
            30 03 02 01 05          (길이 3 명시)

indefinite: 30 80 02 01 05 00 00    (BER 전용)
            ↑   ↑           ↑
            │   길이=indefinite  end-of-contents
            SEQUENCE
```

> DER은 indefinite length를 **절대** 쓰지 않는다(definite-only). 그 이유는 8절의 정규형 논의와 14절의 indefinite 중첩 DoS에서 분명해진다. 거꾸로 말하면, 입력의 길이 자리에서 `0x80`을 보는 순간 "BER이거나, DER로 위장한 공격"이라고 의심해야 한다.

### 4.4 Value

Value는 Length가 알려 준 만큼의 옥텟이다. primitive라면 가공하지 않은 바이트(예: INTEGER의 2의 보수 바이트)이고, constructed라면 그 안이 다시 TLV들의 나열이다. 다음 절부터는 타입별로 Value를 직접 만들어 본다.

---

## 5. 손으로 인코딩하기: 기본 타입들

이제 실제 바이트를 만들어 보자. 모든 예제는 **DER 규칙**을 따른다.

### 5.1 BOOLEAN

값 하나는 1바이트다.

```text
FALSE :  01 01 00
TRUE  :  01 01 FF      ← DER: TRUE는 반드시 0xFF
```

BER은 0이 아닌 어떤 값(`0x01`, `0x7F`…)이든 TRUE로 받아들인다. 반면 DER은 **TRUE를 0xFF로 고정한다**. 정규형이어야 하기 때문이다. "참"을 적는 방법이 하나뿐이어야 같은 값이 같은 바이트가 되고, 같은 서명이 된다(8절).

### 5.2 NULL

내용이 없으므로 길이는 0이다.

```text
NULL :  05 00
```

`AlgorithmIdentifier`의 parameters 자리에 "파라미터 없음"을 적을 때 자주 등장한다(`RSA + NULL`).

### 5.3 INTEGER: 2의 보수, big-endian, 최소 바이트

INTEGER는 임의 정밀도 정수를 **2의 보수, big-endian**으로 적되, **꼭 필요한 만큼의 바이트만** 쓴다.

```text
0    →  02 01 00
1    →  02 01 01
127  →  02 01 7F
-1   →  02 01 FF
-128 →  02 01 80
```

여기서 핵심 규칙 두 가지가 나온다.

**(a) 양수인데 최상위 비트가 1이면 앞에 0x00을 붙인다.** 그렇게 하지 않으면 음수로 읽힌다.

```text
128  →  0x80 한 바이트로 적으면 -128로 읽힘!
        그래서:  02 02 00 80
255  →  02 02 00 FF
256  →  02 02 01 00
200  →  0xC8(=1100_1000)는 MSB=1 → 음수로 오해
        그래서:  02 02 00 C8
```

**(b) 음수도 최소 바이트로 적는다.** `-129`는 1바이트 범위(-128..127)를 벗어나므로 2바이트가 필요하다.

```text
-129 → 16비트 2의 보수 = 0xFF7F
       검산: 0xFF7F = 65407, 65407 - 65536 = -129  ✓
       →  02 02 FF 7F
```

그렇다면 "앞의 0xFF를 더 떼어 낼 수 있지 않을까?"라는 의문이 생긴다. DER에는 이를 막는 규칙이 있다. X.690 8.3.2는 다음과 같다.

> INTEGER 내용이 2바이트 이상이면, **첫 옥텟의 모든 비트와 둘째 옥텟의 bit 8이 (a) 모두 1이어서도 안 되고 (b) 모두 0이어서도 안 된다.**

`FF 7F`는 둘째 옥텟의 bit 8이 0이므로 첫 `FF`를 뗄 수 없다. 따라서 이미 최소다. 반대로 `00 80`(=128)은 둘째 옥텟의 bit 8이 1이므로 `00`을 떼면 음수가 되어 뗄 수 없다. 이것 역시 최소다. 이 규칙은 **양수의 불필요한 `00 ...`과 음수의 불필요한 `FF ...`를 동시에 금지**한다. 이것이 "INTEGER 최소 인코딩"의 정확한 의미다.

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

> "INTEGER는 항상 4바이트나 8바이트"라고 생각하기 쉽다. 하지만 ASN.1 INTEGER는 임의 정밀도다. RSA 모듈러스(2048비트)는 INTEGER 한 개이고, 앞에 `00` 한 바이트가 붙어서 길이가 257바이트다. 시리얼 번호도 큰 INTEGER다. 고정폭 정수에 대한 직관(1장 비트·바이트·엔디안)은 여기서 버려야 한다.

### 5.4 OCTET STRING

가공하지 않은 바이트 열을 그대로 담는다.

```text
바이트 "Hi" (0x48 0x69) → 04 02 48 69
빈 OCTET STRING        → 04 00
```

### 5.5 BIT STRING: 첫 바이트는 "unused bit 개수"

BIT STRING은 길이를 비트 단위로 다룬다. 그래서 Value의 **첫 옥텟은 "마지막 바이트에서 쓰지 않는 비트 수(0–7)"** 이고, 그다음에 실제 비트들이 MSB부터 채워진다.

X.690의 대표 예제인 18비트 `'011011100101110111'B`를 인코딩해 보자.

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

X.509에서 BIT STRING은 핵심적인 두 곳에 쓰인다. 바로 **subjectPublicKey**(공개키 자체)와 **signatureValue**(서명값)다. 이 둘은 보통 unused가 0이므로 `03 ... 00 ...`으로 시작한다.

```text
서명값(예시, 256바이트 RSA 서명):
   03 82 01 01 00 <256바이트 서명>
   ↑           ↑
   BIT STRING  unused=0
   길이=0x0101=257 (1바이트 unused + 256바이트)
```

> subjectPublicKey BIT STRING 맨 앞의 `00`을 "정수의 선행 0"으로 착각하기 쉽다. 그것은 **unused-bit 개수**(=0)다. 그리고 그 BIT STRING의 *내용*은 다시 DER(예: RSA라면 `SEQUENCE { modulus INTEGER, exponent INTEGER }`)이다. 즉 BIT STRING은 DER을 감싸는 포장지 역할을 한다.

**DER의 named-bit 규칙**: 비트마다 이름이 붙은 경우(예: KeyUsage)에는 **뒤쪽의 0 비트를 모두 제거**해야 한다(X.690 11.2). KeyUsage에서 `keyCertSign(5)`와 `cRLSign(6)`만 켠 CA 인증서를 예로 들어 보자. 비트 n은 첫 옥텟의 (왼쪽부터) n번째 위치다. 즉 bit0=0x80, bit1=0x40, …, bit5=0x04, bit6=0x02, bit7=0x01이다.

```text
keyCertSign(5)=0x04, cRLSign(6)=0x02  →  0x06
가장 높은 켜진 비트가 6번 → 0..6까지 7비트 필요 → unused=1
DER:  03 02 01 06
```

실제 CA 인증서의 KeyUsage가 정확히 `03 02 01 06`("Certificate Sign, CRL Sign")으로 나오는 이유가 이것이다.

### 5.6 ENUMERATED

INTEGER와 똑같이 인코딩하고 tag만 `0x0A`로 바꾼다. 예를 들어 CRLReason의 `keyCompromise(1)`은 다음과 같다.

```text
ENUMERATED 1 → 0A 01 01
```

### 5.7 문자열들

내용은 해당 문자 집합의 바이트이고, tag만 다르다.

```text
PrintableString "US" :  13 02 55 53           ('U'=0x55, 'S'=0x53)
IA5String "a@b.com"  :  16 07 61 40 62 2E 63 6F 6D
UTF8String "Example" :  0C 07 45 78 61 6D 70 6C 65
UTF8String "한"       :  0C 03 ED 95 9C          (U+D55C → UTF-8 ED 95 9C)
BMPString "AB"       :  1E 04 00 41 00 42       (UCS-2 big-endian)
```

`"한"`의 UTF-8 바이트가 왜 `ED 95 9C`인지는 [[08 - 유니코드 정규화와 grapheme cluster]] 계열에서 다룬 UTF-8 인코딩 규칙을 따른다(U+D55C는 3바이트 시퀀스다). `IA5String`은 `@`를 허용하지만 `PrintableString`은 허용하지 않는다는 점, 그리고 `BMPString`은 2바이트 고정이어서 ASCII도 `00 41`처럼 적힌다는 점을 눈여겨보자.

### 5.8 시각: UTCTime와 GeneralizedTime

**UTCTime**(tag 0x17)은 연도를 2자리로 적는다. DER에서는 형식이 `YYMMDDHHMMSSZ`로 고정되어 있어서, 초까지 적고 끝에 `Z`(UTC)를 붙인다.

```text
2023-08-15 12:00:00 UTC → "230815120000Z" (13글자)
   17 0D 32 33 30 38 31 35 31 32 30 30 30 30 5A
   │  │  '2''3''0''8''1''5''1''2''0''0''0''0''Z'
   │  길이 0x0D=13
   UTCTime
```

**GeneralizedTime**(tag 0x18)은 연도를 4자리로 적는 `YYYYMMDDHHMMSSZ` 형식이다.

```text
2023-08-15 12:00:00 UTC → "20230815120000Z" (15글자)
   18 0F 32 30 32 33 30 38 31 35 31 32 30 30 30 30 5A
```

> 2자리 연도는 악명 높은 함정이다. UTCTime의 `YY`는 19xx일까, 20xx일까? RFC 5280은 **YY ≥ 50이면 19YY, YY < 50이면 20YY**로 정했다(2050년이 분기점이다).
>
> 그래서 만료일이 **2049년까지면 UTCTime을, 2050년부터는 GeneralizedTime**을 쓰도록 규정한다. "100년 유효" 루트 인증서가 GeneralizedTime을 쓰는 이유가 이것이다. 두 표현이 섞여 있으므로, 파서는 같은 validity 안에서도 notBefore와 notAfter의 tag가 다를 수 있다고 가정해야 한다.

---

## 6. OBJECT IDENTIFIER: OID를 바이트로

OID는 ASN.1에서 가장 우아하면서도 가장 많이 틀리는 부분이다. OID는 `1.2.840.113549.1.1.11`처럼 정수(아크, arc)를 점으로 이은 열이고, 전 세계가 합의한 **계층적 이름 등록 트리**다.

### 6.1 두 가지 인코딩 트릭

규칙은 두 가지뿐이다.

1. **첫 두 아크 X.Y는 정수 하나 `40×X + Y`로 합친다.**
2. **각 아크(합쳐진 첫 아크 포함)는 base-128 big-endian varint로 적는다.** 7비트씩 나누어 적고, 마지막 바이트만 MSB가 0이다(5장(Varint·ZigZag·protobuf wire)에서 본 MIDI VLQ와 같다).

첫 두 아크를 합치는 이유는 무엇일까? 최상위 아크(root)는 0, 1, 2(`itu-t`, `iso`, `joint-iso-itu-t`) 셋뿐이어서 정보량이 작다. 둘째 아크도 root가 0이나 1이면 0–39로 제한된다. 그래서 `40×X+Y`라는 정수 하나에 두 아크를 손실 없이 담을 수 있다.

### 6.2 손계산: `1.2.840.113549` (RSA/PKCS의 루트)

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

`2A 86 48 86 F7 0D`라는 6바이트는 PKCS 인증서 어디에서나 보이는 RSA 계열의 표식이다. hex 덤프에서 `2A 86 48 86 F7 0D`를 보면 곧바로 "RSA 계열 OID구나"라고 알 수 있다.

### 6.3 손계산: `2.5.4.3` (commonName, CN)

```text
아크:  2 . 5 . 4 . 3

(1) 40×2 + 5 = 85 = 0x55
(2) 4 = 0x04
(3) 3 = 0x03

값:  55 04 03
TLV: 06 03 55 04 03
```

CN을 가리키는 `55 04 03`은 인증서의 subject와 issuer 이름 어디에서나 나온다. 마찬가지로 `2.5.4.x`(X.500 디렉터리 속성)는 모두 `55 04 ..`로 시작한다.

| 속성 | OID | DER 값 |
|---|---|---|
| commonName (CN) | 2.5.4.3 | `55 04 03` |
| countryName (C) | 2.5.4.6 | `55 04 06` |
| organizationName (O) | 2.5.4.10 | `55 04 0A` |
| organizationalUnit (OU) | 2.5.4.11 | `55 04 0B` |
| localityName (L) | 2.5.4.7 | `55 04 07` |
| stateOrProvince (ST) | 2.5.4.8 | `55 04 08` |

### 6.4 둘째 아크가 39를 넘는 경우 (root=2)

root가 `2`(joint-iso-itu-t)이면 둘째 아크가 39를 넘을 수 있다. 이때도 `40×2+Y`가 정수 하나가 되고, 그 값이 127을 넘으면 base-128로 여러 바이트가 된다. X.690 표준에 실린 예제 `{joint-iso-itu-t 100 3}`을 보자.

```text
40×2 + 100 = 180
180 = 1×128 + 52  → [1, 52] → (1|0x80)=0x81, 52=0x34  →  81 34
다음 아크 3 = 0x03

값:  81 34 03
TLV: 06 03 81 34 03
```

`60 86 48 ...`로 시작하는 OID(예: NIST의 SHA-256 OID `2.16.840.1.101.3.4.2.1` → `60 86 48 01 65 03 04 02 01`)도 같은 원리다. `40×2+16=96=0x60`이기 때문이다.

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

`2.5.29.x`(id-ce, 인증서 확장)는 모두 `55 1D ..`로 시작한다. `55 04`는 속성, `55 1D`는 확장, `2A 86 48 86 F7 0D`는 RSA 계열, `2A 86 48 CE 3D`는 EC 계열이라는 패턴만 외워도 인증서 hex를 훨씬 쉽게 읽을 수 있다.

> OID 인코딩은 5장(Varint·ZigZag·protobuf wire)의 가변길이 정수가 1984년에 이미 쓰이고 있었다는 증거다. 다만 protobuf의 LEB128은 little-endian인 데 비해 OID는 **big-endian base-128**(MIDI VLQ와 같은 방향)이라는 점만 다르다. "자주 쓰는 작은 아크는 1바이트, 드문 큰 아크는 여러 바이트"라는 빈도 기반 발상은 같다.

---

## 7. SEQUENCE·SET과 중첩: Name 20바이트를 바이트 단위로 읽기

이제 구성 타입으로 트리를 만들어 보자. X.509의 `Name`(issuer/subject)을 처음부터 끝까지 직접 만들어 본다. `Name`은 다음과 같이 정의된다.

```text
Name              ::= RDNSequence
RDNSequence       ::= SEQUENCE OF RelativeDistinguishedName
RelativeDistinguishedName ::= SET OF AttributeTypeAndValue
AttributeTypeAndValue ::= SEQUENCE { type OBJECT IDENTIFIER, value ANY }
```

구조는 세 겹이다. SEQUENCE OF(RDN 목록) 안에 SET OF(한 RDN 안의 속성들)가 있고, 그 안에 SEQUENCE(타입+값)가 있다. `CN = Example` 하나로 된 이름을 안쪽부터 조립해 보자.

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

완성된 20바이트를 트리로 그리면 다음과 같다.

```text
30 12                                  SEQUENCE (RDNSequence), len 18
└─ 31 10                               SET (RelativeDistinguishedName), len 16
   └─ 30 0E                            SEQUENCE (AttributeTypeAndValue), len 14
      ├─ 06 03 55 04 03                OBJECT IDENTIFIER = commonName (2.5.4.3)
      └─ 0C 07 45 78 61 6D 70 6C 65    UTF8String = "Example"
                  E  x  a  m  p  l  e
```

이 20바이트(`30 12 31 10 30 0E 06 03 55 04 03 0C 07 45 78 61 6D 70 6C 65`)가 바로 `CN=Example`이라는 이름의 정식 DER이다. 인증서에서는 issuer와 subject로 같은 구조가 두 번 나온다.

> RDN은 왜 SET OF일까? "이름 성분(RDN)" 하나에 여러 속성을 묶을 수 있기 때문이다(예: `CN=Foo + serialNumber=123`이 한 RDN에 함께 들어간다). SET이므로 순서에 의미가 없고, 따라서 DER은 그 안을 정렬해야 한다(8절). 보통은 RDN 하나에 속성이 하나뿐이라 정렬이 눈에 띄지 않지만, 속성이 여러 개인 RDN에서는 정렬이 결정적인 역할을 한다.

---

## 8. BER vs CER vs DER: 정규형이라는 계약

BER에서는 같은 값을 여러 방식으로 적을 수 있다. TRUE를 `01`로도 `FF`로도 적을 수 있고, 길이도 short form, long form, indefinite form 가운데 무엇으로든 적을 수 있다. 하지만 이 **자유는 서명에 해롭다**. 서명은 "이 바이트열의 해시"에 대해 만드는 것인데, 같은 논리값이 여러 바이트열이 될 수 있으면 서명한 쪽과 검증하는 쪽이 서로 다른 바이트를 해시할 수 있다. 그러면 검증이 실패하거나, 더 나쁘게는 **공격자가 서명을 우회**할 수 있다.

**DER**(Distinguished Encoding Rules)은 BER의 선택지를 모두 하나로 고정해서, **어떤 값이든 정확히 하나의 바이트열**만 갖게 한다(canonical form). DER의 규칙을 모으면 다음과 같다.

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

**CER**도 정규형이지만 반대 방향을 택한다. 긴 구성 값에는 indefinite length를 쓰고, 긴 문자열은 1000옥텟 청크로 자른다. 그래서 **스트리밍**(길이를 미리 모르는 큰 값)에 유리하다. DER은 반대로 모든 길이를 definite로 적으므로 **저장과 서명**에 유리하다. 그래서 X.509는 DER을 쓰고, 일부 메시징 시스템은 CER을 쓴다.

### 8.1 SET OF 정렬을 손으로

DER의 SET OF는 성분들의 **DER 인코딩을 옥텟 문자열로 보고 오름차순으로 정렬**한다(X.690 11.6). 길이가 짧은 쪽은 뒤를 0으로 채워 비교한다. `SET OF INTEGER { 10, 5, 256 }`을 정렬해 보자.

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

같은 세 정수라면 입력 순서와 관계없이 DER은 항상 이 바이트열 하나만 만든다. 이것이 정규형의 장점이다.

### 8.2 DEFAULT 생략: version 필드

TBSCertificate의 `version`은 `[0] EXPLICIT Version DEFAULT v1`이다. v1(=INTEGER 0)이면 기본값이므로 **필드 전체를 생략**하고, v3(=INTEGER 2)이면 적는다.

```text
v3 인증서:  A0 03 02 01 02     (version 필드 존재)
v1 인증서:  (version 필드 없음 → serialNumber가 바로 옴)
```

그래서 v1 인증서를 파싱하면 SEQUENCE의 첫 성분이 `A0...`이 아니라 곧바로 `02 ...`(serialNumber)이다. 파서는 "`A0`로 시작하면 version이 있고, 아니면 v1이다"라는 방식으로 분기한다.

> 서명은 왜 DER이어야 할까? 인증서의 서명은 `signatureValue = Sign(SHA256(DER(tbsCertificate)))`이다. 검증자는 받은 인증서에서 tbsCertificate 부분의 바이트를 **그대로 다시 해시**해서 비교한다. 인코딩이 정규형이 아니라면 "의미는 같지만 바이트가 다른" tbsCertificate가 생길 수 있고, 서명은 바이트에 대해 만든 것이므로 검증이 깨진다. 그래서 X.509는 "tbsCertificate는 DER이어야 한다"고 강제한다.
>
> 더 위험한 것은 **파서가 BER처럼 너그럽게 비정규 인코딩을 받아 주는 경우**다. 그러면 공격자가 서명된 정규 바이트와 다른 변형을 같은 의미로 통과시킬 여지가 생긴다(14절 BERserk).

---

## 9. 태깅: IMPLICIT vs EXPLICIT, context-specific

SEQUENCE에 OPTIONAL 필드가 여러 개 있으면 문제가 생긴다. 두 OPTIONAL 필드가 같은 타입(둘 다 INTEGER)이라면 파서는 "지금 나온 INTEGER가 어느 필드인지" 구별할 수 없다. 해결책은 **태깅**이다. 필드마다 context-specific 번호 `[0]`, `[1]`…을 붙여 위치 정보를 태그에 담는다.

```text
Example ::= SEQUENCE {
    a  [0] INTEGER OPTIONAL,
    b  [1] INTEGER OPTIONAL
}
```

이제 `[0]`이 보이면 a이고, `[1]`이 보이면 b이다. 태깅에는 두 가지 방식이 있다.

### 9.1 EXPLICIT: 원래 타입을 그대로 감싼다

EXPLICIT 태깅은 원래 TLV 전체를 **새 태그로 한 번 더 감싼다**. INTEGER 2를 `[0] EXPLICIT`로 태깅하면 다음과 같다.

```text
원래:      02 01 02                  (INTEGER 2)
[0] EXPLICIT로 포장:
           A0 03 02 01 02
           │  │  └────────── 원래 INTEGER TLV 그대로
           │  길이 3
           context [0], constructed (포장지라 constructed!)
```

`A0`는 "context-specific(10), constructed(1), 번호 0"이다(4.1절). 안에 원래 INTEGER가 온전히 들어 있으므로 **타입 정보가 보존**된다. X.509의 version이 정확히 이 형식이다(`A0 03 02 01 02` = v3).

### 9.2 IMPLICIT: 원래 태그를 바꿔 끼운다

IMPLICIT 태깅은 감싸지 않고 **원래 타입의 태그 바이트만 context 태그로 교체**한다. 그래서 더 짧다.

```text
원래:      02 01 02                  (INTEGER 2, tag 02)
[0] IMPLICIT로 태깅:
           80 01 02
           │  │  └─ 값 02 (그대로)
           │  길이 1
           context [0], primitive (INTEGER는 primitive라 0 유지)
```

`80`은 "context(10), primitive(0), 번호 0"이다. EXPLICIT은 5바이트, IMPLICIT은 3바이트다. IMPLICIT은 크기가 작은 대신 타입 정보를 보존하지 않는다. 받는 쪽은 스키마를 봐야만 "이 `[0]`이 원래 INTEGER였다"는 것을 알 수 있다.

| | EXPLICIT | IMPLICIT |
|---|---|---|
| 바이트 | 더 김(원래 TLV를 감쌈) | 더 짧음(태그만 교체) |
| 원래 타입 보존 | 예(안에 원래 태그) | 아니오(스키마 필요) |
| constructed 비트 | 항상 constructed | 원래 타입을 따름 |
| CHOICE/ANY에 적용 | 가능 | **불가**(원래 태그가 사라지면 대안 구별 불가) |

### 9.3 X.509에서의 혼용

X.509(RFC 5280)는 두 방식을 섞어 쓴다. 모듈의 기본값은 EXPLICIT이지만 일부 필드는 IMPLICIT으로 지정되어 있다.

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

`version`은 INTEGER를 안전하게 감싸기 위해 EXPLICIT(`A0`)을 쓰고, `extensions`도 SEQUENCE를 감싸기 위해 EXPLICIT(`A3`)을 쓴다. 반면 uniqueID는 BIT STRING이므로 IMPLICIT(`81`/`82`)을 써서 한 바이트를 아꼈다.

> CHOICE에는 IMPLICIT 태깅을 쓰면 안 된다. CHOICE는 자기 태그가 없고 "선택된 대안의 태그"로 자신을 드러내는데, IMPLICIT이 그 태그를 덮어쓰면 어느 대안인지 알 방법이 없어진다. 그래서 컴파일러는 CHOICE, ANY, 열린 타입에는 자동으로 EXPLICIT을 적용한다. X.509의 `Time ::= CHOICE { utcTime, generalTime }`이 태깅 없이 그대로 쓰이는 이유가 이것이다.

---

## 10. X.509 v3 인증서의 구조

이제 지금까지의 조각을 모두 모아 실제 인증서를 읽어 보자. 최상위 구조는 성분이 세 개뿐인 SEQUENCE다.

```text
Certificate ::= SEQUENCE {
    tbsCertificate       TBSCertificate,      -- 서명 대상 본문
    signatureAlgorithm   AlgorithmIdentifier, -- 서명에 쓴 알고리즘(중복 기재)
    signatureValue       BIT STRING           -- 서명값
}
```

`tbsCertificate`("**T**o **B**e **S**igned")가 본문이고, CA는 이 본문의 DER을 해시하고 서명해서 `signatureValue`에 넣는다. `signatureAlgorithm`에는 tbs 안의 `signature` 필드와 같은 값을 한 번 더 적는다. 이 중복은 "본문 밖에서 알고리즘을 바꿔치기하는 공격"을 막기 위한 것이다(둘이 다르면 거부한다).

### 10.1 TBSCertificate 본문을 직접 만들기

앞 절들에서 만든 조각으로 TBS의 앞부분을 실제 바이트로 조립해 보자.

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

`validity`의 두 UTCTime 길이를 검산해 보자. 각 `17 0D ...`는 2+13=15바이트이고, 두 개를 합치면 30=0x1E이다. 그래서 `30 1E`가 된다. `signature`의 SEQUENCE 내용은 OID(11바이트)+NULL(2바이트)=13=0x0D이므로 `30 0D`이다. 둘 다 정확히 맞는다.

### 10.2 subjectPublicKeyInfo의 이중 포장

공개키 부분을 보면 ASN.1에서 포장 안에 다시 포장이 들어가는 구조가 잘 드러난다.

```text
SubjectPublicKeyInfo ::= SEQUENCE {
   algorithm        AlgorithmIdentifier,   -- 어떤 키 종류인가
   subjectPublicKey BIT STRING             -- 키의 실제 바이트
}
```

RSA라면 `subjectPublicKey` BIT STRING의 *내용*이 다시 DER이다.

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

`publicExponent`가 `02 03 01 00 01`인 것을 볼 수 있다. 65537은 0x010001이고, 최상위 바이트 0x01의 MSB가 0이므로 선행 0이 필요 없어 3바이트로 끝난다(5.3절). 거의 모든 RSA 키가 이 `02 03 01 00 01`로 끝난다. EC 키라면 algorithm이 `id-ecPublicKey + 곡선 OID`이고, BIT STRING에는 곡선 위의 점(`04 || X || Y`)이 들어간다.

### 10.3 개념적인 asn1parse 덤프

위에서 직접 만든 앞부분을 `openssl asn1parse`로 보면 다음과 같이 나온다(offset은 TBS 시작 기준이다).

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

`d`는 깊이, `hl`은 헤더 길이(tag와 length의 바이트 수), `l`은 Value 길이다. offset 29의 OID(`06 03 55 04 03`)와 offset 34의 UTF8String(`0C 07 ...`)은 7절에서 직접 만든 20바이트와 정확히 일치한다. `30 12`가 offset 23, `31 10`이 25, `30 0E`가 27, OID가 29, 문자열이 34에 있다. 손계산과 도구의 결과가 조금도 어긋나지 않는다.

---

## 11. 확장(Extensions): 이중 포장의 이유

X.509 v3의 강점은 **확장(extensions)** 에 있다. SAN(도메인 목록), KeyUsage, BasicConstraints(CA 여부) 등이 모두 확장이다. 구조는 다음과 같다.

```text
extensions ::= [3] EXPLICIT SEQUENCE OF Extension
Extension  ::= SEQUENCE {
    extnID     OBJECT IDENTIFIER,
    critical   BOOLEAN DEFAULT FALSE,
    extnValue  OCTET STRING            -- 확장값의 DER을 OCTET STRING으로 감쌈!
}
```

핵심은 `extnValue`가 **OCTET STRING이고, 그 안에 다시 확장별 DER이 들어간다**는 점이다. 왜 이렇게 이중으로 감쌀까? **확장성** 때문이다. 파서는 모르는 확장을 만나도 "OCTET STRING 한 덩어리"로 보고 건너뛸 수 있다. 안의 내용을 해석할 필요 없이 길이만큼 건너뛰면 된다.

아는 확장일 때만 OCTET STRING을 열어 안의 DER을 다시 파싱한다. critical=TRUE인데 모르는 확장이면 인증서를 거부해야 한다는 규칙(RFC 5280)도 이 구조 덕분에 깔끔하게 동작한다.

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

트리로 그리면 다음과 같다.

```text
30 0F                          SEQUENCE (Extension)
├─ 06 03 55 1D 13              OID basicConstraints (2.5.29.19)
├─ 01 01 FF                    BOOLEAN critical = TRUE
└─ 04 05                       OCTET STRING (extnValue), len 5
   └─ 30 03                    SEQUENCE (BasicConstraints)   ← 또 DER!
      └─ 01 01 FF              BOOLEAN cA = TRUE
```

`04 05`(OCTET STRING) 안에 `30 03 ...`(또 다른 SEQUENCE)이 들어 있는 것이 바로 이중 포장이다.

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

### 11.3 SubjectAltName (SAN): `example.com`은 hex로 어떻게 보이는가

SAN은 오늘날 인증서에서 가장 중요한 확장이다(브라우저는 CN이 아니라 SAN으로 호스트를 검증한다). 구조는 다음과 같다.

```text
SubjectAltName ::= GeneralNames ::= SEQUENCE OF GeneralName
GeneralName ::= CHOICE {
   ...
   dNSName  [2] IMPLICIT IA5String,   -- 도메인은 여기
   iPAddress[7] IMPLICIT OCTET STRING,
   ...
}
```

`dNSName`은 `[2] IMPLICIT IA5String`이므로, 도메인 문자열의 태그가 IA5String(`16`)이 아니라 **context `[2]` primitive(`82`)** 로 바뀐다(9.2절).

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

트리로 그리면 다음과 같다.

```text
30 16                                    SEQUENCE (Extension)
├─ 06 03 55 1D 11                        OID subjectAltName (2.5.29.17)
└─ 04 0F                                 OCTET STRING (extnValue), len 15
   └─ 30 0D                              SEQUENCE (GeneralNames)
      └─ 82 0B 65 78 ... 6F 6D           [2] dNSName = "example.com"
```

hex 덤프에서 `82 0B 65 78 61 6D 70 6C 65 ...`를 보면 곧바로 "SAN의 도메인이 여기 있다"는 것을 알 수 있다. `82`(context [2])가 dNSName의 표식이다. 도메인이 여러 개라면 `82 ..` `82 ..`가 연달아 이어진다.

> "도메인은 인증서 안에 그냥 문자열로 들어 있겠지"라고 생각하면 `16`(IA5String)을 찾게 된다. 하지만 실제로는 IMPLICIT 태깅 때문에 `82`로 나타난다. IMPLICIT가 태그를 바꿔 끼운다는 9.2절의 규칙이 실제로 적용되는 사례다.

---

## 12. PEM: DER을 텍스트로

DER은 바이너리이므로 이메일, 설정 파일, 복사해서 붙여넣기에 적합하지 않다. **PEM**(Privacy-Enhanced Mail, RFC 7468)은 DER 바이트를 10장(체크섬 내장 인코딩)에서 다룬 **base64**로 인코딩하고, 사람이 읽을 수 있는 머리줄과 꼬리줄로 감싼 것이다.

```text
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJA0... (base64, 보통 64자마다 줄바꿈) ...
...wIDAQAB
-----END CERTIFICATE-----
```

규칙은 단순하다.

1. 객체의 DER을 만든다.
2. 그 바이트를 base64(표준 알파벳 `A–Z a–z 0–9 + /`, 패딩 `=`)로 인코딩한다.
3. 64자마다 줄을 바꾼다(RFC 7468 권장).
4. `-----BEGIN <라벨>-----`과 `-----END <라벨>-----`로 감싼다.

### 12.1 base64를 손으로: `30 03 01 01 FF`

11.1절에서 만든 BasicConstraints의 안쪽 5바이트 `30 03 01 01 FF`를 base64로 바꿔 보자. base64는 3바이트(24비트)를 6비트씩 4글자로 바꾼다.

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

base64 인덱스를 검산해 보자. `M`=12, `A`=0, `B`=1(A=0, B=1…), `f`=31(a=26, …, f=31), `8`=60(0=52, …, 8=60)이다. 디코더로 `MAMBAf8=`를 되돌리면 정확히 `30 03 01 01 FF`가 나온다. base64에서 6비트와 8비트 단위를 오가는 비트 재배열은 1장(비트·바이트·엔디안)에서 다룬 비트 슬라이싱과 같다.

### 12.2 PEM 라벨들

라벨(`-----BEGIN <X>-----`의 X)은 안에 든 객체의 종류를 알려 준다.

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

### 12.3 한 파일에 여러 객체: 체인

PEM은 BEGIN/END 블록을 **여러 개 이어 붙일 수 있다**. TLS 서버가 보내는 인증서 체인이 대표적인 예다.

```text
-----BEGIN CERTIFICATE-----   ← 리프(서버) 인증서
...
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----   ← 중간 CA
...
-----END CERTIFICATE-----
```

파서는 BEGIN/END 쌍을 순서대로 읽고, base64 바깥의 텍스트(블록 사이의 주석 등)는 무시한다. 반면 **DER 파일(.der/.cer)** 에는 바이너리 객체 하나만 담긴다. 여러 객체를 DER 파일 하나에 담으려면 PKCS#7 같은 컨테이너가 필요하다.

| | PEM | DER |
|---|---|---|
| 형식 | base64 텍스트 + 헤더 | 순수 바이너리 |
| 확장자 | `.pem .crt .cer .key` | `.der .cer` |
| 여러 객체 | 가능(블록 이어붙임) | 한 객체(컨테이너 필요) |
| 편집기/이메일 | 안전 | 깨짐 |
| 크기 | DER의 약 137%(base64 + 헤더) | 최소 |

> PEM은 "새로운 인코딩"이 아니라 **DER을 텍스트 채널로 안전하게 전달하는 봉투**일 뿐이다. 실제 내용은 항상 DER이다. `openssl x509 -inform PEM -outform DER`로 봉투만 벗기면 똑같은 바이트가 나온다. 그래서 PEM과 DER 사이의 변환은 base64 인코딩과 디코딩이 전부다.

---

## 13. PKCS 가족과 관련 포맷

X.509 주변에는 ASN.1로 정의된 여러 표준이 함께 쓰인다. 그 가운데 핵심은 PKCS(Public-Key Cryptography Standards)다. RSA사가 제정했고, 이후 상당수가 RFC로 옮겨졌다.

| 표준 | 정의(RFC) | 내용 | ASN.1 최상위 타입 |
|---|---|---|---|
| **PKCS#1** | RFC 8017 | RSA 키·서명·암호화 | `RSAPublicKey`, `RSAPrivateKey` |
| **PKCS#7 / CMS** | RFC 5652 | 서명·봉투 데이터 컨테이너 | `ContentInfo`, `SignedData` |
| **PKCS#8** | RFC 5958 | 알고리즘 무관 개인키 | `OneAsymmetricKey`(`PrivateKeyInfo`) |
| **PKCS#10** | RFC 2986 | 인증서 서명 요청(CSR) | `CertificationRequest` |
| **PKCS#12** | RFC 7292 | 키+인증서+체인 묶음(`.pfx`/`.p12`) | `PFX` |
| **SPKI** | RFC 5280 | 공개키 포장 | `SubjectPublicKeyInfo` |
| **SEC1** | SEC1 v2 | EC 개인키 | `ECPrivateKey` |

### 13.1 PKCS#8: 키 포맷의 통일

PKCS#1은 RSA 전용이어서 `RSAPrivateKey`의 첫 INTEGER 바로 다음이 모듈러스다. EC나 Ed25519 같은 알고리즘이 등장하면서 알고리즘과 무관한 래퍼가 필요해졌고, 그것이 PKCS#8이다.

```text
PrivateKeyInfo ::= SEQUENCE {
    version              INTEGER,
    privateKeyAlgorithm  AlgorithmIdentifier,   -- 어떤 알고리즘 키인가
    privateKey           OCTET STRING           -- 알고리즘별 키 DER을 감쌈
}
```

여기서도 **OCTET STRING 안에 다시 DER**이 들어간다(11절과 같은 패턴). RSA라면 그 안에 PKCS#1 `RSAPrivateKey`가, EC라면 SEC1 `ECPrivateKey`가 들어간다. 그래서 같은 RSA 키가 `BEGIN RSA PRIVATE KEY`(PKCS#1)로 저장될 수도 있고, `BEGIN PRIVATE KEY`(PKCS#8로 한 겹 더 감싼 형태)로 저장될 수도 있다.

### 13.2 CSR: 신청자가 직접 서명한다

PKCS#10 CSR은 "이 공개키로 인증서를 발급해 달라"는 요청이다. 구조는 인증서와 비슷하다.

```text
CertificationRequest ::= SEQUENCE {
    certificationRequestInfo  SEQUENCE { version, subject, SPKI, attributes },
    signatureAlgorithm        AlgorithmIdentifier,
    signature                 BIT STRING
}
```

차이는 서명하는 주체에 있다. 인증서는 **CA가** tbsCertificate에 서명하지만, CSR은 **신청자 본인이** 자기 개인키로 certificationRequestInfo에 서명한다. 이것은 "이 공개키와 짝을 이루는 개인키를 내가 가지고 있다"는 증명(proof of possession)이다. CA는 이 자가 서명을 검증한 뒤, 필요한 내용을 골라 새 인증서를 만들고 자기 키로 다시 서명한다.

### 13.3 PKCS#12: 비밀번호로 보호하는 묶음

`.pfx`/`.p12`는 개인키, 인증서, 체인을 한 파일에 담고 비밀번호로 암호화한 묶음이다. 윈도와 맥의 키체인 가져오기·내보내기 기능이 이 형식을 쓴다. 내부는 PKCS#7 컨테이너, PKCS#8 키, 암호화 계층이 ASN.1로 중첩된 꽤 복잡한 트리다. "파일 하나로 모두 가지고 다닐 수 있다"는 편리함을 얻는 대신, 구조가 복잡해지고 (기본 암호 설정이 약할 경우) 보안 위험도 감수해야 한다.

---

## 14. 파싱은 공격의 최전선이다: 함정과 CVE

ASN.1 파서는 신뢰할 수 없는 입력(인증서, 서명, LDAP 요청)을 직접 받는 최전선에 있다. 그래서 역사적으로 보안 사고가 자주 일어난 곳이다. [[06 - CRC의 수학과 체크섬]]이 "우발적인 손상"을 다뤘다면, 여기서는 **악의적인 입력**이 인코딩의 어떤 빈틈을 노리는지 살펴본다.

### 14.1 길이 필드를 믿어서 생기는 오버리드와 오버플로

가장 흔한 유형이다. 파서는 length 필드를 읽고 그만큼 메모리를 할당하거나 복사하는데, **실제 버퍼보다 큰 길이**를 검증 없이 믿으면 힙 오버리드나 오버플로가 발생한다.

```text
공격 입력:  04 84 7F FF FF FF <실제로는 8바이트만 존재>
            │  │  └──────────── "길이 약 21억" 주장
            │  long form, 길이 바이트 4개
            OCTET STRING

순진한 파서:  malloc(0x7FFFFFFF) 또는 buf+len 까지 읽기 → 폭발
```

방어하려면 **선언된 길이가 남은 버퍼 크기 이하인지**를 항상 먼저 검사해야 한다. long form의 길이 바이트가 정수 타입의 범위를 넘으면(예: 5바이트 이상) 즉시 거부한다. "길이는 입력이 주장하는 값일 뿐 사실이 아니다"라는 원칙을 반드시 지켜야 한다. Heartbleed(TLS heartbeat)도 본질은 "선언된 길이 ≠ 실제 길이"였고, 같은 유형의 실수가 ASN.1 파서에서도 수없이 일어났다.

### 14.2 BER indefinite length 중첩 DoS

BER의 indefinite length(`0x80 ... 00 00`, 4.3절)는 깊이 제한이 없으면 **무한 중첩**으로 스택을 고갈시킨다.

```text
30 80 30 80 30 80 30 80 ... (수만 겹) ... 00 00 00 00 ...
재귀 파서가 깊이 N으로 들어가 스택 오버플로(DoS)
```

방어 방법은 다음과 같다. **DER은 indefinite 자체를 금지**(definite-only)하므로, X.509 파서는 길이 자리에서 `0x80`을 보면 즉시 거부한다. BER을 받아야 하는 LDAP/SNMP 파서는 **최대 중첩 깊이**(예: 수십)를 강제한다.

### 14.3 OID 아크 정수 오버플로

OID 아크는 base-128 varint여서 길이 제한이 없다(6절). 거대한 아크를 보내 파서의 정수 변수를 오버플로시키면, 다른 OID로 오인하거나 계산이 어긋난다.

```text
06 08 2A 86 48 90 80 80 80 00   ← 마지막 아크가 32비트를 넘김(2³²)
파서가 u32에 누적하면 wrap-around → 엉뚱한 OID로 해석
```

방어하려면 아크를 임의 정밀도(또는 64비트와 범위 검사)로 누적하고, **OID 정규화**(알려진 OID 테이블과 바이트 단위로 비교하고, 디코드한 뒤 다시 비교하지 않음)로 처리한다. "OID는 디코드해서 비교하지 말고 바이트로 비교하라"는 것이 좋은 습관이다.

### 14.4 CN 안의 NUL 바이트 (null-prefix 공격)

2009년 Moxie Marlinspike가 시연한 고전적인 공격이다. 공격자가 CA에 `CN = "www.bank.com\0.attacker.com"`으로 CSR을 제출한다. CA가 `attacker.com`의 소유 여부만 검증하고 인증서를 발급하면, **C 문자열로 CN을 다루는** 클라이언트는 `\0`에서 문자열을 잘라 `www.bank.com`으로 인식한다.

```text
ASN.1 문자열:  ... 77 77 77 2E 62 61 6E 6B 2E 63 6F 6D 00 2E 61 74 ...
                  w  w  w  .  b  a  n  k  .  c  o  m  \0 .  a  t ...
길이 기반 ASN.1: "www.bank.com\0.attacker.com" (NUL은 그냥 한 바이트)
C 문자열 파서:    "www.bank.com"               (NUL에서 종료)  ← 불일치!
```

근본 원인은 표현 방식의 불일치다. **ASN.1 문자열은 길이 접두 방식이라 NUL을 데이터로 허용**하는데, 검증 코드는 이를 NUL로 끝나는 C 문자열로 다뤘다. 관련 CVE로는 CVE-2009-2408(NSS/Firefox) 등이 있다. 방어하려면 길이를 기준으로 끝까지 비교하고, CN과 SAN에 제어 문자와 NUL을 금지해야 한다. 같은 유형의 **시각적 혼동**(키릴 `а`와 라틴 `a`)은 9장(유니코드 보안)에서 다룬 homograph 문제와 직접 연결된다.

### 14.5 BERserk: 너그러운 파싱이 서명 위조를 허용한다 (CVE-2014-1568)

가장 교훈적인 사례다. PKCS#1 v1.5 서명을 검증할 때는 복호화 후 나오는 `DigestInfo`(`SEQUENCE { 해시알고리즘, OCTET STRING 해시 }`)를 파싱한다. RSA 공개지수가 `e=3`인 키에서 **파서가 DigestInfo 뒤의 잉여 바이트를 무시하거나 길이를 느슨하게 검사**하면, 공격자는 세제곱근을 맞춰 **개인키 없이 서명을 위조**할 수 있다(Bleichenbacher 2006 공격을 실제로 적용한 사례다).

```text
정상 검증: 복호 결과가 [패딩 || DigestInfo] 정확히여야 함
취약 파서: [패딩 || DigestInfo || 임의 잉여바이트]를 통과시킴
   → 공격자가 잉여 자리를 조작해 완전세제곱수를 만들어 e=3 서명 위조
```

8절에서 말한 "DER의 엄격함이 곧 보안"이라는 점이 여기서 가장 극적으로 드러난다. **인코딩을 정확하게, 끝까지, 정규형으로 검사**하지 않으면 서명의 수학이 멀쩡해도 시스템이 뚫린다. 방어하려면 DigestInfo를 바이트 단위로 재구성해 정확히 일치하는지 비교하고(파싱이 아니라 비교), 트레일링 바이트는 하나도 허용하지 않으며, 가능하면 RSA-PSS를 사용한다.

### 14.6 그 밖의 빈틈

| 함정 | 메커니즘 | 방어 |
|---|---|---|
| 비정규 DER | 선행 0 INTEGER, indefinite, 비최소 길이 통과 | DER 엄격 검증, 비정규 거부 |
| UTCTime 2자리 연도 | 2050 분기 오해로 만료 판정 오류 | RFC 5280 분기 규칙 준수, 신규는 GeneralizedTime |
| 문자열 타입 혼동 | PrintableString vs UTF8String 비교 누락 | 정규화 후 비교, 타입 명시 |
| 음의 시리얼/0 시리얼 | INTEGER 음수·0 시리얼로 충돌 유발 | 양의 0이 아닌 시리얼 강제(RFC 5280) |
| 깊은 중첩 | 재귀 파서 스택 고갈 | 깊이 제한 |

> ASN.1 파서를 직접 작성하지 말자. 검증된 라이브러리를 쓰되, 그 라이브러리조차 1장(비트·바이트·엔디안)에서 강조한 "라이브러리 불신" 원칙으로 대해야 한다. 길이는 검증하기 전까지는 거짓일 수 있고, 인코딩은 정규형이 아닐 수 있으며, 문자열에는 NUL이 들어 있을 수 있다. **모든 입력 길이는 남은 버퍼와 대조하고, 모든 서명 대상은 DER 정규형으로 강제하며, 모든 OID는 바이트로 비교해야 한다**.

---

## 15. 도구: 바이트를 사람의 말로

마지막으로 실무에서 ASN.1을 들여다볼 때 쓰는 도구들을 살펴보자. 손계산을 검증하고 디버깅할 때 꼭 필요하다.

### 15.1 openssl asn1parse: 만능 TLV 덤프 도구

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

`-i`는 깊이에 따라 들여쓰기를 하고, `-strparse <offset>`은 그 offset에 있는 OCTET STRING 안을 다시 파싱한다(11절의 이중 포장을 열 때 유용하다). `cons`/`prim`은 4.1절에서 본 constructed/primitive 비트다.

### 15.2 openssl x509 -text: 사람이 읽을 수 있는 해석

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

`asn1parse`가 "바이트의 구조"를 보여 준다면, `x509 -text`는 "그 구조의 의미"를 보여 준다. 10절과 11절에서 직접 만든 필드들이 여기서 사람이 읽을 수 있는 말로 나타난다. `CA:TRUE`(`30 03 01 01 FF`), `Certificate Sign, CRL Sign`(`03 02 01 06`), `DNS:example.com`(`82 0B ...`)이 그 예다.

### 15.3 dumpasn1 / der2ascii

| 도구 | 만든이 | 쓰임 |
|---|---|---|
| `dumpasn1` | Peter Gutmann | 가장 상세한 TLV 덤프(OID·태그 주석 풍부) |
| `der2ascii` / `ascii2der` | Google | DER ↔ 사람이 편집 가능한 텍스트(테스트 케이스 제작) |
| `openssl asn1parse` | OpenSSL | 빠른 구조 확인 |
| `xxd` / `hexdump` | 표준 | 가공하지 않은 hex(태그를 직접 식별할 때) |

`der2ascii`는 특히 **악의적인 인증서나 엣지 케이스 인증서를 직접 만들** 때 유용하다. `SEQUENCE { INTEGER { 0x00 0x7F } }` 같은 비정규 인코딩을 텍스트로 적고 `ascii2der`로 바이트를 만든 뒤, 파서가 이를 거부하는지 테스트한다.

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

## 16. 정리: ASN.1에서 배운 것

ASN.1/DER을 한 바이트씩 따라오면서 직렬화 설계의 원형을 거의 모두 만나 보았다.

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

이 장에서 배운 내용을 시리즈의 다른 장들과 연결해 보자.

- **가변길이 정수는 1984년에 이미 있었다.** OID의 base-128 인코딩은 5장(Varint·ZigZag·protobuf wire)의 varint와 같은 발상이다. 방향(big endian과 little endian)만 다르다.
- **정규형은 보안이다.** DER이 BOOLEAN TRUE를 `0xFF`로 고정하고 길이를 최소화하는 이유는 "같은 값 → 같은 바이트 → 같은 서명"을 보장하기 위해서다. 6장(CRC의 수학과 체크섬)에서 체크섬이 위변조를 막지 못한다고 했듯이, 무결성과 정규형은 별개의 보장이며 둘 다 필요하다.
- **포장 안의 포장.** extnValue가 OCTET STRING 안에 다시 DER을 담는 구조는 "모르면 건너뛰고(skip), 알면 연다(parse)"는 확장성 패턴이다. 같은 패턴이 PKCS#8 키와 BIT STRING 공개키에서도 반복된다.
- **텍스트 봉투.** PEM은 DER을 10장(체크섬 내장 인코딩)에서 다룬 base64로 감싼 운반 수단일 뿐이며, 실제 내용은 항상 바이너리 DER이다.
- **현대의 후속 포맷.** 12장(CBOR·Avro·zero-copy 직렬화)에서 다루는 포맷들은 TLV의 자기서술(CBOR)과 스키마 분리·태그 없는 압축(Avro)을 현대적으로 다시 구현한다. ASN.1의 BER(자기서술)과 PER(스키마 의존 압축)이 그 두 갈래의 원조다.
