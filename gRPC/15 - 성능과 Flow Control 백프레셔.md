---
title: "15 - 성능과 Flow Control 백프레셔"
date: 2026-06-26
tags:
  - grpc
  - flow-control
  - backpressure
  - http2
  - performance
  - bdp
  - window-update
  - compression
  - benchmarking
  - 학습노트
---

## 0. 들어가며: 성능은 "밀어 넣기"가 아니라 "흘려보내기"다

gRPC 성능을 처음 고민하는 엔지니어는 대개 "어떻게 하면 더 빨리 보낼 수 있을까?"라고 묻는다. 이 질문은 절반만 맞다. 분산 시스템에서 처리량을 결정하는 것은 보내는 쪽의 속도가 아니라, **보내는 쪽과 받는 쪽 사이에 데이터가 쌓이지 않고 균형을 이루는 지점**이다.

비유를 하나 들어 보자. 거대한 물탱크(빠른 생산자)에서 작은 컵(느린 소비자)으로 물을 따른다고 하자. 탱크의 밸브를 활짝 열면 컵이 넘치고 물은 바닥에 쏟아진다. 컴퓨터에서는 이 "바닥에 쏟아진 물"이 그냥 사라지지 않는다. 받는 쪽 OS 버퍼와 받는 쪽 애플리케이션 큐, 보내는 쪽 송신 버퍼에 차곡차곡 쌓인다. 결국 누군가의 힙(heap)이 OutOfMemory로 터지거나, GC가 멈추거나, 컨테이너가 OOMKilled로 종료된다.

흐름 제어(flow control)와 백프레셔(backpressure)는 이렇게 넘치는 물을 처음부터 막는 메커니즘이다. 핵심 아이디어는 단순하다. **받는 쪽이 "이만큼만 받을 수 있다"고 명시적으로 허락하기 전에는, 보내는 쪽이 그 양을 넘겨 보내지 않는다.** 받는 쪽은 데이터를 처리하고 나면 "이제 더 받을 수 있다"는 의미로 신용(credit)을 돌려준다.

보내는 쪽은 이 신용을 다 쓰면 멈춘다. 이렇게 생긴 압력은 와이어를 거슬러 올라가, 애플리케이션 코드의 `Send()` 호출을 블로킹하거나 "지금은 보내지 말라"는 신호(`isReady() == false`)로 나타난다.

이 장에서는 이 압력이 어떻게 만들어지고, 와이어를 거쳐 여러분의 코드까지 어떻게 전달되는지를 바이트 수준부터 애플리케이션 콜백까지 따라간다. HTTP/2 프레임의 세부 사항은 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서, 스트리밍 통신의 네 가지 방식은 [[05 - 통신의 4가지 방식 - Unary와 Streaming]]에서 다뤘으므로, 여기서는 그 위에서 압력이 어떻게 동작하는지에 집중한다.

---

## 1. 왜 흐름 제어가 또 필요한가: TCP 위에 한 겹 더

### 1.1 TCP도 흐름 제어를 하는데 왜 또 필요한가

TCP를 아는 사람이라면 의문이 들 것이다. "TCP에 이미 수신 윈도우(receive window)와 혼잡 제어(congestion control)가 있는데, HTTP/2가 흐름 제어를 또 한다고?"

그렇다. 그리고 멀티플렉싱(multiplexing) 때문에 이 중복은 **반드시 필요하다.**

4장(HTTP/2 깊이 보기)에서 봤듯이, HTTP/2는 TCP 연결 하나 위에 여러 논리적 스트림(stream)을 동시에 실어 나른다. gRPC에서는 RPC 호출 하나가 스트림 하나이다. 즉 TCP 소켓 하나 위로 수십, 수백 개의 RPC가 프레임 단위로 잘게 나뉘어 인터리빙(interleaving)된 채 흐른다.

여기서 문제가 드러난다. TCP의 수신 윈도우는 **연결 전체**를 대상으로 동작한다. TCP는 그 연결 위에 스트림이 몇 개 있는지, 어느 스트림이 빠르고 어느 스트림이 느린지 전혀 모른다. TCP 입장에서는 그저 한 줄기의 바이트 스트림일 뿐이다.

```text
TCP만 있을 때 (스트림 구분 없음):

  Stream 1 (느린 소비자)  ─┐
  Stream 3 (빠른 소비자)  ─┼─► [단일 TCP 연결] ─► 수신 측
  Stream 5 (느린 소비자)  ─┘
                              │
                     TCP 수신 윈도우 하나로만 제어
                     → Stream 1이 막히면 TCP 버퍼가 차고
                     → Stream 3까지 같이 멈춘다 (HOL 블로킹)
```

Stream 1의 소비자가 데이터를 읽지 않아 OS 수신 버퍼가 가득 차면, TCP는 윈도우를 0으로 줄여 송신 측 전체를 멈춘다. 그러면 문제없이 빠르게 소비되던 Stream 3과 Stream 5까지 함께 멈춘다. 이것이 **헤드 오브 라인 블로킹(Head-of-Line blocking, HOL)** 의 한 형태다.

HTTP/2의 흐름 제어는 이 문제를 **스트림 단위로** 해결하기 위해 존재한다. 스트림마다 독립된 윈도우를 두어, 느린 Stream 1만 멈추고 빠른 Stream 3은 계속 흐르게 한다. TCP가 연결 전체를 보는 거시적 제어라면, HTTP/2 흐름 제어는 스트림 하나하나를 보는 미시적 제어다. 둘은 동작하는 층위가 다르므로 중복이 아니라 서로를 보완한다.

### 1.2 두 개의 윈도우: 커넥션 윈도우와 스트림 윈도우

HTTP/2(RFC 7540과 이를 개정한 RFC 9113)는 흐름 제어를 **두 수준**으로 정의한다.

| 레벨                 | 적용 범위                | 스트림 식별자      | 막는 것                     |
| ------------------ | -------------------- | ------------ | ------------------------ |
| 커넥션(connection) 레벨 | 연결 위 모든 DATA 프레임의 총합 | Stream ID 0  | 연결 전체가 받는 쪽 메모리를 압도하는 것  |
| 스트림(stream) 레벨     | 개별 스트림 하나            | 해당 Stream ID | 한 RPC가 다른 RPC의 몫을 독식하는 것 |

규칙은 단순하지만 엄격하다. **DATA 프레임 하나를 보내려면 커넥션 윈도우와 스트림 윈도우 모두에 그 프레임의 페이로드 크기만큼 잔액이 있어야 한다.** 프레임을 보내면 두 윈도우에서 모두 그 크기만큼 차감한다. 둘 중 하나라도 0이면 그 스트림(또는 연결 전체)은 더 이상 DATA를 보낼 수 없다.

```text
보내는 쪽이 Stream 3에 1000바이트 DATA를 보내려 할 때:

  연결 윈도우:  [잔액 5000] ──┐
                              ├─► 둘 다 ≥ 1000 ? → 전송 허가
  Stream 3 윈도우: [잔액 2000]─┘     전송 후:
                                    연결 윈도우 5000 → 4000
                                    Stream 3   2000 → 1000

  만약 Stream 3 윈도우가 [잔액 500]이었다면?
  → 500 < 1000 → Stream 3 전송 불가 (다른 스트림은 영향 없음)

  만약 연결 윈도우가 [잔액 800]이었다면?
  → 800 < 1000 → 이 연결의 모든 스트림이 전송 불가
```

여기서 중요한 세부 사항이 있다. **흐름 제어는 DATA 프레임에만 적용된다.** HEADERS, SETTINGS, PING, WINDOW_UPDATE, RST_STREAM 같은 제어용 프레임은 흐름 제어 대상이 아니다. 흐름 제어가 제어 프레임까지 막으면, 윈도우를 늘려 주는 WINDOW_UPDATE 자체가 흐름 제어에 걸려 데드락에 빠지기 때문이다. 신용을 돌려주는 프레임은 언제나 통과할 수 있어야 한다.

### 1.3 신용 기반(credit-based) 모델: 받는 쪽이 속도를 정한다

HTTP/2 흐름 제어의 원칙을 한 문장으로 요약하면 이렇다. **보내는 쪽이 아니라 받는 쪽이 속도를 결정한다.**

이것이 신용 기반(credit-based) 모델이다. 받는 쪽은 "받을 수 있는 바이트 수"라는 신용을 가지고 있고, 이 신용을 송신 측에 광고(advertise)한다. 송신 측은 가진 신용만큼만 보낸다. 받는 쪽이 데이터를 처리해 버퍼를 비우면, 비운 만큼 `WINDOW_UPDATE` 프레임을 보내 신용을 보충해 준다.

```text
신용기반 흐름 제어의 한 사이클:

  수신측                                    송신측
    │                                         │
    │  초기 윈도우 = 65535 (SETTINGS로 광고)   │
    │ ◄───────────────────────────────────── │
    │                                         │
    │        DATA (16384 bytes)               │
    │ ◄───────────────────────────────────── │  잔액: 65535 → 49151
    │        DATA (16384 bytes)               │
    │ ◄───────────────────────────────────── │  잔액: 49151 → 32767
    │                                         │
    │  [앱이 32768바이트를 읽어서 소화함]       │
    │                                         │
    │  WINDOW_UPDATE (+32768) on stream       │
    │ ─────────────────────────────────────► │  잔액: 32767 → 65535
    │  WINDOW_UPDATE (+32768) on conn(id=0)   │
    │ ─────────────────────────────────────► │
    │                                         │
    │        DATA (16384 bytes)               │
    │ ◄───────────────────────────────────── │  다시 흐름 재개
```

여기서 핵심은 다음과 같다. **받는 쪽이 데이터를 읽지 않으면 WINDOW_UPDATE를 보내지 않고, 그러면 윈도우가 0으로 줄어들어 송신 측이 멈춘다.** 이 멈춤이 바로 와이어 수준에서 본 백프레셔의 실체다. 받는 쪽 애플리케이션이 느리면 버퍼가 비워지지 않고, WINDOW_UPDATE가 나가지 않으며, 윈도우가 닫혀 송신 측이 멈춘다. 이렇게 전송 계층에서 압력이 자동으로 만들어진다.

---

## 2. WINDOW_UPDATE와 SETTINGS: 신용을 광고하고 보충하는 프레임

### 2.1 SETTINGS_INITIAL_WINDOW_SIZE: 시작점

연결이 맺어지면 양쪽은 `SETTINGS` 프레임을 교환한다. 여기에 담기는 파라미터 가운데 흐름 제어의 출발점이 되는 것이 `SETTINGS_INITIAL_WINDOW_SIZE`(설정 식별자 `0x4`)다.

- **기본값: 65535바이트 (64KiB - 1)**
- 최댓값: 2³¹ - 1 = 2147483647바이트 (약 2GiB)
- 이 값은 **새로 생성되는 모든 스트림**의 초기 윈도우 크기를 정한다.
- 단, **커넥션 레벨 윈도우의 초기값은 항상 65535로 고정**이며 `SETTINGS_INITIAL_WINDOW_SIZE`의 영향을 받지 않는다. 커넥션 윈도우를 키우려면 연결 직후에 WINDOW_UPDATE를 보내야 한다.

마지막 항목은 헷갈리기 쉬우므로 표로 정리하면 다음과 같다.

| 윈도우 종류 | 초기값 | 변경 방법 |
|-------------|--------|-----------|
| 스트림 레벨 초기 윈도우 | `SETTINGS_INITIAL_WINDOW_SIZE` (기본 65535) | SETTINGS로 기본값 변경 + WINDOW_UPDATE로 개별 증감 |
| 커넥션 레벨 윈도우 | 항상 65535 (고정) | WINDOW_UPDATE로만 증가 (SETTINGS 영향 없음) |

SETTINGS로 `INITIAL_WINDOW_SIZE`를 바꾸면 **이미 열려 있는 스트림의 윈도우도 그 차이만큼 한꺼번에 조정된다.** 예를 들어 65535에서 1048576으로 늘리면, 살아 있는 모든 스트림의 윈도우에 (1048576 - 65535)만큼 더해진다. 반대로 설정을 줄이면 윈도우가 음수가 될 수도 있다. 음수 윈도우는 규격상 허용되며, 윈도우가 양수가 될 때까지 전송이 멈춘다.

### 2.2 WINDOW_UPDATE 프레임을 바이트 단위로 분석하기

`WINDOW_UPDATE`(프레임 타입 `0x8`)는 흐름 제어 신용을 보충하는 프레임이다. 4장에서 본 공통 프레임 헤더와 함께 구조를 바이트 단위로 살펴보자.

모든 HTTP/2 프레임의 9바이트 헤더 형식은 다음과 같다.

```text
+-----------------------------------------------+
|                 Length (24)                   |   3 bytes
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |               1 + 1 bytes
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                     |   4 bytes
+=+=============================================================+
|                   Frame Payload (length)                      |
+---------------------------------------------------------------+
```

WINDOW_UPDATE의 페이로드는 4바이트다. 최상위 1비트는 예약 비트(R)이고, 나머지 31비트가 "윈도우 증가량(Window Size Increment)"이다.

스트림 7번의 윈도우를 32768(0x8000)만큼 보충하는 WINDOW_UPDATE 프레임의 실제 바이트는 다음과 같다.

```text
00 00 04    →  Length = 4 (페이로드 4바이트)
08          →  Type = 0x08 (WINDOW_UPDATE)
00          →  Flags = 0 (WINDOW_UPDATE는 플래그 없음)
00 00 00 07 →  Stream ID = 7 (R 비트 = 0)
00 00 80 00 →  Window Size Increment = 0x00008000 = 32768
```

손으로 디코딩해 보자.

- `00 00 04`: 빅엔디언 24비트 정수로 4다. 페이로드가 4바이트라는 뜻이다.
- `08`: 타입 8, 즉 WINDOW_UPDATE다.
- `00`: 플래그가 없다.
- `00 00 00 07`: 최상위 R 비트(0)와 31비트 스트림 ID 7로 이루어진다. 7번 스트림의 윈도우를 늘린다는 뜻이다.
- `00 00 80 00`: R 비트(0)와 31비트 증가량 0x8000 = 32768로 이루어진다. 송신 측은 스트림 7의 윈도우 잔액에 32768을 더한다.

같은 프레임에서 Stream ID가 `00 00 00 00`(=0)이라면 **커넥션 레벨** 윈도우를 늘린다. gRPC 구현은 보통 두 종류의 WINDOW_UPDATE를 함께 보낸다. 하나는 데이터가 소비된 스트림에 대한 것이고, 다른 하나는 커넥션 전체에 대한 것이다.

증가량이 0이면 프로토콜 에러(`PROTOCOL_ERROR`)이고, 윈도우가 2³¹-1을 넘게 만드는 증가는 `FLOW_CONTROL_ERROR`다.

### 2.3 DATA 프레임과 흐름 제어의 관계

흐름 제어가 차감하는 대상은 DATA 프레임의 **페이로드 길이**다. 정확히는 패딩(padding)까지 포함한 페이로드 전체다. DATA 프레임의 구조는 다음과 같다.

```text
+---------------+
|Pad Length? (8)|  ← PADDED 플래그 있을 때만
+---------------+-----------------------------------------------+
|                            Data (*)                           |
+---------------------------------------------------------------+
|                           Padding (*)                         |
+---------------------------------------------------------------+
```

흐름 제어에서 차감되는 바이트는 Data 길이와 Padding 길이, 그리고 PADDED 플래그가 있을 때의 Pad Length 필드 1바이트를 더한 값이다. 즉 패딩도 윈도우를 소모한다. 패딩은 메시지 길이를 숨기는 보안 목적으로 쓰이지만 흐름 제어 신용을 소모한다는 점을 기억하자. gRPC는 일반적으로 패딩을 쓰지 않는다.

여기서 gRPC 메시지와의 관계가 중요하다. [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 본 gRPC의 길이 접두사 프레이밍(length-prefixed framing)을 떠올리자. gRPC 메시지 하나는 와이어에서 다음과 같은 형태다.

```text
[1바이트 compressed-flag][4바이트 빅엔디언 메시지 길이][메시지 본문]
```

이 5바이트 접두사와 본문 전체가 HTTP/2 DATA 프레임의 페이로드로 들어간다. 큰 gRPC 메시지 하나는 여러 DATA 프레임으로 나뉜다(HTTP/2 프레임 크기 한계인 `SETTINGS_MAX_FRAME_SIZE`의 기본값은 16384바이트). 그래서 4MB짜리 메시지는 16KB DATA 프레임 최소 257개(본문 256개 분량에 5바이트 접두사가 더해짐)로 나뉘고, 그 프레임 모두가 윈도우 신용을 소모한다.

---

## 3. 작은 윈도우의 문제: BDP와 처리량 붕괴

### 3.1 BDP란 무엇인가

여기서 네트워크 성능의 가장 근본적인 공식 하나를 알아야 한다. 바로 **대역폭 지연 곱(Bandwidth-Delay Product, BDP)** 이다.

```text
BDP (바이트) = 대역폭 (바이트/초) × 왕복 지연(RTT, 초)
```

BDP는 직관적으로 "파이프 안에 동시에 떠 있을 수 있는 데이터의 양"이다. 송신 측이 보낸 데이터가 수신 측에 도착하고, 수신 측의 ACK(또는 WINDOW_UPDATE)가 다시 돌아오기까지 한 바퀴를 도는 동안 파이프를 가득 채우려면, 최소한 BDP만큼의 데이터가 "전송 중(in-flight)" 상태여야 한다.

흐름 제어 윈도우가 BDP보다 작으면 어떻게 될까? 송신 측은 윈도우를 다 쓰고 나서 WINDOW_UPDATE가 돌아올 때까지 멈춰서 기다린다. 그동안 파이프는 텅 비어 있다. 결과적으로 처리량은 다음 값으로 제한된다.

```text
최대 처리량 ≤ 윈도우 크기 / RTT
```

### 3.2 숫자로 보는 처리량 붕괴

기본 윈도우 65535바이트로 RTT별 최대 처리량을 계산해 보자.

| RTT | 윈도우 | 최대 처리량 = 65535/RTT | 환산 |
|-----|--------|--------------------------|------|
| 1ms (같은 DC) | 65535 B | 65.5 MB/s | ≈ 524 Mbps |
| 10ms (리전 내) | 65535 B | 6.55 MB/s | ≈ 52 Mbps |
| 50ms (대륙 간) | 65535 B | 1.31 MB/s | ≈ 10.5 Mbps |
| 100ms (지구 반대편) | 65535 B | 655 KB/s | ≈ 5.2 Mbps |

10Gbps(1250 MB/s) 광케이블을 깔아 두어도, RTT가 100ms인 서울-버지니아 링크에서 단일 스트림은 **5.2Mbps**밖에 내지 못한다. 이론 대역폭의 0.05%다. 파이프는 거대한데 윈도우라는 가는 빨대로 빨아들이는 셈이다.

```text
RTT 100ms, 윈도우 65535B 일 때 시간 흐름:

시간 →
0ms    송신: 65535바이트 전송 (윈도우 소진)
       │
       │  ← 윈도우 0, 송신 측 멈춤. 파이프 텅 빔.
       │
100ms  수신측 도착 → WINDOW_UPDATE 송신
200ms  WINDOW_UPDATE 도착 → 다시 65535바이트 전송
       │
       │  ← 또 멈춤
       │
       파이프라이닝해도 매 RTT(100ms)마다 65535바이트
       = 약 655 KB/s 가 한계
```

이 문제를 "긴 뚱뚱한 파이프(long fat network, LFN)" 문제라고도 부른다. 대역폭이 크고 지연도 긴 환경에서는 작은 윈도우가 처리량을 크게 떨어뜨린다.

### 3.3 해결책 1: 윈도우를 크게 잡기

가장 단순한 해결책은 초기 윈도우를 BDP 이상으로 키우는 것이다. RTT 100ms, 1Gbps를 가정하면 BDP = 125 MB/s × 0.1s = 12.5 MB다. 따라서 윈도우를 12.5MB 이상으로 잡아야 파이프를 채울 수 있다.

언어별 설정 예시는 다음과 같다.

```go
// Go: 서버
import "google.golang.org/grpc"

srv := grpc.NewServer(
    grpc.InitialWindowSize(1<<20),     // 스트림 윈도우 1 MiB
    grpc.InitialConnWindowSize(1<<22), // 커넥션 윈도우 4 MiB
)

// Go: 클라이언트
conn, _ := grpc.NewClient(target,
    grpc.WithInitialWindowSize(1<<20),
    grpc.WithInitialConnWindowSize(1<<22),
)
```

```java
// Java (Netty 기반): 서버
import io.grpc.netty.NettyServerBuilder;

NettyServerBuilder.forPort(50051)
    .flowControlWindow(4 * 1024 * 1024)   // 4 MiB
    .build();

// Java: 클라이언트
import io.grpc.netty.NettyChannelBuilder;
NettyChannelBuilder.forTarget(target)
    .flowControlWindow(4 * 1024 * 1024)
    .build();
```

하지만 무작정 키우면 다른 비용이 생긴다. 윈도우가 크면 받는 쪽이 그만큼 버퍼를 미리 확보해야 하고, 느린 소비자가 있을 때 메모리에 쌓이는 양도 늘어난다. 윈도우는 "처리량을 위한 버퍼 예산"이다. BDP에 맞춰 적절히 잡아야 하며, 무한정 키운다고 해결되지 않는다.

### 3.4 해결책 2: BDP 자동 추정과 동적 흐름 제어 (BDP estimation)

RTT는 연결마다 다르고 시간에 따라 변하기 때문에, 윈도우를 수동으로 BDP에 맞추는 것은 현실적이지 않다. 그래서 **일부** gRPC 구현은 **BDP를 실시간으로 추정해 윈도우를 자동으로 늘리는 동적 흐름 제어(dynamic flow control / BDP estimation)** 를 내장하고 있다.

대표적인 예가 gRPC-Go이며, 여기서는 이를 "BDP estimator and dynamic flow control window"라고 부른다. C 코어를 공유하는 구현(C++, Python, Ruby, C#, PHP 등)도 코어에 들어 있는 BDP estimator를 그대로 활용한다.

**중요한 예외는 gRPC-Java다.** 순수 Java(Netty 기반) 구현인 gRPC-Java는 BDP를 자동으로 추정하지 **않는다.** 대신 **고정된 흐름 제어 윈도우**를 쓰는데, 그 기본값은 HTTP/2 프로토콜 기본값인 65535가 아니라 gRPC-Java가 연결할 때 SETTINGS로 올려서 광고하는 **1MiB(1048576)** 다.

따라서 BDP가 큰 고지연·고대역폭 링크에서는 Java에서 `flowControlWindow()`로 윈도우를 직접 키워야 하며, 자동 튜닝에 기댈 수 없다. 뒤에 나오는 "BDP를 모르겠으면 손대지 말고 자동 튜닝에 맡기라"는 조언은 gRPC-Go와 C 코어 구현에 해당한다. Java는 오히려 명시적으로 튜닝해야 할 수 있다는 점이 구현 사이의 결정적인 차이다.

동작 원리를 개념 수준에서 보면 다음과 같다.

```text
BDP 추정 알고리즘 (개략):

1. 수신 측은 주기적으로 PING 프레임을 보내 RTT를 측정한다.
   (PING은 즉시 ACK되므로 왕복 시간을 잰다 — [[10 - Deadline 취소 타임아웃]]의
    keepalive PING과는 목적이 다르다)

2. 한 RTT 구간 동안 받은 총 데이터 양(samples)을 누적한다.
   "한 번의 왕복 동안 이만큼 받았다" = 그 순간의 추정 BDP.

3. 측정된 BDP가 현재 윈도우의 일정 비율(예: 2/3)을 넘으면,
   윈도우가 병목이라는 신호 → 윈도우를 2배로 키운다 (상한까지).

4. RTT가 줄거나 트래픽이 줄면 윈도우를 다시 줄인다.
```

```text
동적 윈도우 성장 그래프:

윈도우
크기
  │                                   ┌──── 상한 (예: 16MB)
16M┤                            ┌─────┘
   │                       ┌────┘
 4M┤                  ┌────┘
   │             ┌────┘
 1M┤        ┌────┘
   │   ┌────┘
64K┤───┘
   └────────────────────────────────────► 시간
   연결 시작 시 작게 → BDP 추정 따라 점점 키움
```

이 자동 튜닝 덕분에 대부분의 경우에는 윈도우를 건드릴 필요가 없다. gRPC-Go는 동적 흐름 제어가 켜져 있으면(기본값), `InitialWindowSize`를 명시적으로 지정하지 않는 한 BDP estimator가 윈도우를 관리한다. **주의할 점은 `InitialWindowSize`를 직접 설정하면 동적 흐름 제어(BDP 추정에 따른 자동 확장)가 꺼지고 윈도우가 그 고정값으로 굳는다는 것이다.**

그러므로 BDP를 잘 모르겠다면 아예 건드리지 않고 자동 튜닝에 맡기는 편이 안전하다. 명시적인 설정은 "내 환경의 BDP를 정확히 알고 있고, 자동 추정보다 더 잘 맞출 수 있다"는 확신이 있을 때만 하자.

---

## 4. 백프레셔: 전송 계층의 압력이 애플리케이션까지 전달되는 경로

지금까지는 와이어 수준의 흐름 제어를 살펴봤다. 이제 정말 중요한 질문으로 넘어가자. **이 압력은 어떻게 애플리케이션 코드까지 전달되는가?** 흐름 제어가 와이어에서만 동작하고 애플리케이션이 이를 모른다면, 빠른 생산자는 라이브러리 내부의 송신 큐에 메시지를 한없이 쌓다가 결국 메모리를 고갈시킨다. 백프레셔의 핵심은 **흐름 제어의 압력을 애플리케이션 API로 드러내는 것**이다.

### 4.1 문제의 본질: 빠른 생산자, 느린 소비자

서버 스트리밍을 생각해 보자(5장 통신의 4가지 방식). 서버가 데이터베이스에서 100만 행을 읽어 클라이언트로 스트리밍한다. 서버는 디스크나 메모리에서 행을 매우 빠르게(초당 수십만 행) 꺼낼 수 있다. 그런데 클라이언트는 느린 네트워크에 연결된 모바일 기기다.

서버가 백프레셔를 무시하고 `responseObserver.onNext(row)`를 100만 번 호출하면 어떻게 될까?

```text
백프레셔 없는 순진한 서버:

  for (Row row : queryResult) {        // 초당 50만 행 생산
      responseObserver.onNext(toProto(row));
  }
  responseObserver.onCompleted();

  내부에서 벌어지는 일:
    onNext() → 직렬화 → gRPC 송신 큐에 적재
                              │
                              ▼
              ┌──────────────────────────────┐
              │  송신 큐 (write queue)         │  ← 흐름 제어로 막혀
              │  [row1][row2]...[row999998]   │     와이어로 못 나감
              └──────────────────────────────┘
                              │
                  와이어 윈도우 = 0 (클라이언트 느림)
                              │
              큐가 무한정 자란다 → 힙 폭발 → OOM
```

흐름 제어는 데이터가 와이어로 나가는 것을 막았다. 하지만 `onNext()`는 큐에 넣기만 하고 즉시 반환되므로, 애플리케이션 루프는 멈추지 않고 계속 큐에 쌓는다. 와이어가 막혀도 메모리 큐는 한없이 커진다. 흐름 제어만으로는 메모리를 지키지 못한 것이다.

해결의 열쇠는 **와이어가 막히면 애플리케이션 루프도 생산을 멈춰야 한다**는 데 있다. 이 사실을 애플리케이션에 어떻게 알릴지는 언어마다 메커니즘이 다르다.

### 4.2 Go: 블로킹 Send/Recv, 가장 자연스러운 백프레셔

Go의 gRPC는 백프레셔를 가장 깔끔하게 처리한다. `stream.Send()`가 **블로킹** 호출이기 때문이다. 흐름 제어 윈도우가 닫혀 있으면 `Send()`는 그 자리에서 멈춰 기다린다(고루틴이 블록된다). 윈도우가 열리면 반환된다.

```go
// 서버 스트리밍 핸들러 (Go) — 백프레셔가 공짜
func (s *server) ListRows(req *pb.ListRowsRequest, stream pb.RowService_ListRowsServer) error {
    rows, err := s.db.Query(stream.Context(), req.GetFilter())
    if err != nil {
        return status.Errorf(codes.Internal, "query failed: %v", err)
    }
    defer rows.Close()

    for rows.Next() {
        row := scanRow(rows)
        // Send()는 흐름 제어 윈도우가 닫혀 있으면 여기서 블록된다.
        // 클라이언트가 느리면 이 고루틴이 멈추고, rows.Next()도 안 불린다.
        // → DB에서 더 안 읽는다. 메모리에 쌓이지 않는다. 자동 백프레셔.
        if err := stream.Send(toProto(row)); err != nil {
            return err // 클라이언트 취소/연결 끊김
        }
    }
    return rows.Err()
}
```

핵심은 `Send()`가 블록되면 `for rows.Next()` 루프 자체가 멈춘다는 점이다. 루프가 멈추면 DB에서 다음 행을 읽지 않는다. 즉 **느린 소비자의 압력이 블로킹된 Send, 멈춘 루프, 읽히지 않는 DB 커서 순으로 거슬러 올라가** 생산 자체가 느려진다. 생산 속도가 소비 속도에 자동으로 맞춰지는 것이다. 이것이 백프레셔의 이상적인 형태다.

Go에서 주의할 점이 있다. `Send()`가 블록되는 동안 그 고루틴은 점유된 상태다. 동시 스트림이 수만 개라면 고루틴 수만 개가 블록될 수 있지만, Go 고루틴은 가볍기 때문에(스택 ~2KB) 대개 문제가 없다. 다만 데드라인 없이 영원히 블록되지 않도록 `stream.Context()`의 취소와 데드라인([[10 - Deadline 취소 타임아웃]])을 항상 따라야 한다.

`RecvMsg`/`Recv`도 마찬가지로 흐름 제어와 연동된다. 받는 쪽이 `Recv()`를 호출해야 라이브러리가 데이터를 소비한 것으로 보고 WINDOW_UPDATE를 보낸다. 받는 쪽이 `Recv()`를 제때 호출하지 않으면 윈도우가 열리지 않고, 이것이 송신 측으로 전파되는 백프레셔가 된다.

### 4.3 Java: isReady()와 onReadyHandler, 콜백 기반 논블로킹 백프레셔

Java gRPC의 기본 API는 `StreamObserver`라는 **콜백 기반 논블로킹** 모델이다(5장 통신의 4가지 방식). `onNext()`는 블로킹하지 않고 즉시 반환한다. 그래서 Go처럼 루프만 돌리면 백프레셔가 저절로 걸리지는 않는다. 대신 명시적인 신호를 확인해야 한다.

핵심 도구는 `CallStreamObserver`(서버에서는 `ServerCallStreamObserver`, 클라이언트에서는 `ClientCallStreamObserver`)가 제공하는 다음 두 가지다.

- `boolean isReady()`: 지금 `onNext()`를 호출해도 송신 버퍼가 넘치지 않는지를 알려 준다. 흐름 제어 윈도우와 내부 버퍼 상태를 반영하며, `false`면 "지금은 보내지 말라"는 뜻이다.
- `setOnReadyHandler(Runnable)`: `isReady()`가 `false`에서 `true`로 바뀔 때 호출되는 콜백이다. "이제 다시 보내도 된다"는 신호다.

올바른 패턴은 다음과 같다. `isReady()`가 false인 동안 바쁜 대기(busy-wait)를 해서는 안 되고, 콜백을 기반으로 생산을 멈췄다가 재개해야 한다.

```java
// 서버 스트리밍 (Java) — isReady()/onReadyHandler 기반 백프레셔
public void listRows(ListRowsRequest req, StreamObserver<Row> responseObserver) {
    ServerCallStreamObserver<Row> serverObserver =
        (ServerCallStreamObserver<Row>) responseObserver;

    // 데이터 소스를 직접 당겨오는 이터레이터라고 가정
    Iterator<Row> rows = repository.streamRows(req.getFilter());

    // onReadyHandler: 전송 가능 상태가 될 때마다 호출됨
    serverObserver.setOnReadyHandler(() -> {
        // isReady()가 true인 동안 최대한 보낸다. false가 되면 멈추고
        // 다음 onReadyHandler 호출을 기다린다.
        while (serverObserver.isReady() && rows.hasNext()) {
            serverObserver.onNext(rows.next());
        }
        if (!rows.hasNext()) {
            responseObserver.onCompleted();
        }
        // isReady()가 false가 되면 while을 빠져나오고, 콜백이 끝난다.
        // 큐가 비워져 다시 ready가 되면 gRPC가 onReadyHandler를 또 호출한다.
    });

    // 클라이언트가 취소하면 생산 중단할 수 있게 핸들러 등록
    serverObserver.setOnCancelHandler(() -> { /* 리소스 정리 */ });
}
```

이 패턴의 흐름을 다시 정리하면 다음과 같다.

```text
isReady()/onReadyHandler 사이클:

   onReadyHandler 호출됨
        │
        ▼
   while (isReady() && hasNext())
        │  onNext()로 메시지 송신 큐에 적재
        │  큐가 차면 isReady() → false
        ▼
   isReady() == false → while 탈출, 생산 정지
        │
        │  [gRPC가 큐를 와이어로 흘려보냄, 흐름 제어 따라]
        │  [큐가 충분히 비워지면...]
        ▼
   isReady() false → true 전이 → onReadyHandler 재호출
        │
        └──► 다시 위로 (생산 재개)
```

`isReady()`를 무시하고 무작정 `onNext()`를 호출하면 어떻게 될까? 흐름 제어로 와이어가 막혀 있으므로, Java gRPC는 메시지를 버리지 않고 내부 버퍼에 한없이 쌓는다. 결국 힙이 고갈된다. 그래서 **빠른 생산자와 느린 소비자가 만나는 상황에서 Java는 반드시 isReady()를 따라야 한다.** 이것이 운영 측면에서 Go와 Java의 가장 큰 차이다.

### 4.4 받는 쪽 백프레셔: request(n)으로 수요를 표현하기

지금까지는 보내는 쪽이 멈추는 백프레셔를 살펴봤다. 반대로 **받는 쪽이 "한 번에 N개만 처리할 수 있다"고 수요를 제어**하는 것도 중요하다. 이것이 reactive streams의 `request(n)` 모델이다.

Java gRPC의 콜백 모델은 기본적으로 메시지를 자동으로 한 개씩 요청한다(`onNext` 콜백이 끝나면 자동으로 다음 1개를 request한다). 하지만 자동 흐름 제어를 끄고 수동으로 제어할 수도 있다.

```java
// 클라이언트가 양방향 스트림에서 수동 흐름 제어 (Java)
ClientResponseObserver<Request, Response> observer =
    new ClientResponseObserver<Request, Response>() {
        ClientCallStreamObserver<Request> requestStream;

        @Override
        public void beforeStart(ClientCallStreamObserver<Request> rs) {
            this.requestStream = rs;
            // 자동 흐름 제어를 끈다: 이제 내가 명시적으로 request()해야
            // 다음 메시지를 받는다.
            rs.disableAutoRequestWithInitial(1); // 처음엔 1개만 요청
        }

        @Override
        public void onNext(Response value) {
            process(value);              // 무거운 처리
            requestStream.request(1);    // 다 처리했으니 1개 더 요청
            // → 이 request(1)이 결국 WINDOW_UPDATE로 이어져
            //   송신 측에 "더 보내도 돼" 신호가 간다.
        }

        @Override public void onError(Throwable t) { /* ... */ }
        @Override public void onCompleted() { /* ... */ }
    };
```

서버 쪽에서도 `ServerCallStreamObserver.disableAutoRequest()`와 `request(n)`으로 같은 제어를 할 수 있다. 핵심 메커니즘은 이렇다. **`request(n)`은 "애플리케이션이 n개를 더 소비할 의향이 있다"는 수요 신호이고, gRPC 런타임은 이 수요를 흐름 제어 윈도우(WINDOW_UPDATE)로 변환한다.**

즉 애플리케이션의 `request(n)`이 와이어 수준의 신용으로 바뀌어 송신 측을 조절한다. 애플리케이션 백프레셔와 전송 백프레셔가 하나의 사슬로 이어지는 지점이다.

```text
애플리케이션 백프레셔 → 와이어 백프레셔 변환 사슬:

  수신 앱: request(n) 호출
        │  "n개 더 소화 가능"
        ▼
  gRPC 런타임: 소비량만큼 WINDOW_UPDATE 발행
        │  "윈도우 +크기"
        ▼
  와이어: 송신 측 윈도우 잔액 증가
        │
        ▼
  송신 앱: isReady() true / Send() 언블록
        │  "다시 생산"
        ▼
  생산 속도가 소비 속도에 묶인다 (end-to-end backpressure)
```

### 4.5 Python: 블로킹과 풀 기반

Python의 동기(synchronous) gRPC API에서 서버 스트리밍 핸들러는 제너레이터(generator)를 반환한다. gRPC 런타임이 제너레이터에서 값을 꺼내 가는 동작이 흐름 제어와 연동된다. 즉 와이어가 막히면 런타임이 제너레이터의 다음 `yield`를 꺼내 가지 않으므로 생산이 자연스럽게 멈춘다. Go와 비슷한 풀(pull) 기반 백프레셔다.

```python
# 서버 스트리밍 (Python, 동기 API)
def ListRows(self, request, context):
    cursor = self.db.query(request.filter)
    for row in cursor:                  # 런타임이 당겨갈 때만 다음 row 생산
        if context.is_active() is False:
            return                       # 클라이언트 취소 감지
        yield to_proto(row)
    # 런타임이 yield를 당기는 속도 = 와이어로 흘러나가는 속도
    # → 흐름 제어로 막히면 yield도 멈춤 → DB 커서도 멈춤 (자동 백프레셔)
```

`grpc.aio`(asyncio) API에서는 `await stream.write(msg)`가 코루틴이며, 흐름 제어로 막히면 `await`에서 대기한다. Go의 블로킹 Send와 같은 방식으로 이해하면 된다.

---

## 5. 메시지 크기 한계와 청크 분할

### 5.1 기본 4MB 수신 한계와 RESOURCE_EXHAUSTED

gRPC는 메모리 고갈을 막기 위해 메시지 크기에 **기본 한계**를 둔다.

- **수신(inbound) 메시지 기본 한계: 4MB (4 × 1024 × 1024 = 4194304바이트)**
- 송신(outbound) 메시지 기본 한계: 대부분의 구현에서 무제한(`MAX_INT`)이지만 설정할 수 있다.

이 한계는 스트림 전체 크기가 아니라 **메시지 하나(단일 protobuf 메시지)의 직렬화 크기**에 적용된다. 스트림으로 4MB짜리 메시지를 1000개 보내는 것은 괜찮지만, 5MB짜리 메시지 하나는 막힌다.

한계를 넘으면 RPC는 **`RESOURCE_EXHAUSTED`(상태 코드 8)** 로 실패한다([[09 - 에러 모델 - 상태 코드와 Rich Error]]). 에러 메시지는 보통 "Received message larger than max (5242880 vs 4194304)" 같은 형태다.

```go
// 수신 한계 늘리기 (Go) — 신중하게!
// 서버
srv := grpc.NewServer(
    grpc.MaxRecvMsgSize(16 * 1024 * 1024), // 16 MiB 수신 허용
    grpc.MaxSendMsgSize(16 * 1024 * 1024),
)
// 클라이언트 (call option)
resp, err := client.GetBlob(ctx, req,
    grpc.MaxCallRecvMsgSize(16*1024*1024),
)
```

```java
// Java
NettyServerBuilder.forPort(50051)
    .maxInboundMessageSize(16 * 1024 * 1024)
    .build();
```

```python
# Python
server = grpc.server(
    futures.ThreadPoolExecutor(),
    options=[
        ('grpc.max_receive_message_length', 16 * 1024 * 1024),
        ('grpc.max_send_message_length', 16 * 1024 * 1024),
    ],
)
```

### 5.2 큰 메시지가 나쁜 이유와 청크 스트리밍

한계를 그냥 키우면 될 것 같지만, 큰 메시지에는 본질적인 문제가 있다.

1. **전부 아니면 전무(all-or-nothing)인 역직렬화**: protobuf 메시지는 끝까지 다 받아야 파싱할 수 있다. 100MB 메시지는 100MB가 모두 도착할 때까지 받는 쪽 메모리에 통째로 버퍼링된다. 메모리 사용량이 크게 치솟고, 중간에 끊기면 전부 버려야 한다.
2. **흐름 제어 입자도(granularity)**: 큰 메시지 하나도 흐름 제어 과정에서는 잘게 나뉘지만, 애플리케이션은 메시지 하나를 원자적으로 처리하므로 진행 상황을 알 수 없다.
3. **재시도와 복원력**: 큰 메시지는 재시도([[13 - 안정성 - Retry Health Check Keepalive]]) 비용이 크다. 실패하면 전체를 다시 보내야 한다.

일반적인 해법은 **큰 페이로드를 청크(chunk)로 나눠 서버 또는 클라이언트 스트리밍으로 보내는 것**이다.

```proto
// 큰 파일/블롭을 청크 스트리밍으로 전송하는 패턴
syntax = "proto3";
package storage.v1;

message UploadChunk {
  oneof payload {
    FileMetadata metadata = 1;  // 첫 메시지: 메타데이터
    bytes data = 2;             // 이후 메시지들: 데이터 청크
  }
}

message FileMetadata {
  string filename = 1;
  string content_type = 2;
  int64 total_size = 3;
}

message UploadResult {
  string file_id = 1;
  int64 bytes_received = 2;
  string sha256 = 3;
}

service FileService {
  // 클라이언트 스트리밍: 청크를 여러 번 보내고 결과 하나 받음
  rpc Upload(stream UploadChunk) returns (UploadResult);
  // 서버 스트리밍: 청크를 여러 번 받음
  rpc Download(DownloadRequest) returns (stream UploadChunk);
}
```

청크 크기는 보통 **16KB ~ 64KB**로 잡는다. 너무 작으면(예: 1KB) 메시지마다 드는 프레이밍, HPACK, 흐름 제어 계산 오버헤드가 상대적으로 커지고, 너무 크면(예: 4MB) 앞에서 말한 큰 메시지 문제가 생긴다. 16~64KB는 HTTP/2 기본 프레임 크기(16KB)와도 잘 맞고 흐름 제어 입자도도 적절하다.

```go
// 청크 업로드 클라이언트 (Go) — 백프레셔 준수
const chunkSize = 64 * 1024 // 64 KiB

func uploadFile(ctx context.Context, client pb.FileServiceClient, path string) error {
    stream, err := client.Upload(ctx)
    if err != nil {
        return err
    }
    // 1) 메타데이터 먼저
    if err := stream.Send(&pb.UploadChunk{
        Payload: &pb.UploadChunk_Metadata{Metadata: &pb.FileMetadata{
            Filename: filepath.Base(path),
        }},
    }); err != nil {
        return err
    }
    // 2) 데이터 청크 — Send()가 흐름 제어에 따라 블록되며 백프레셔를 받는다
    f, _ := os.Open(path)
    defer f.Close()
    buf := make([]byte, chunkSize)
    for {
        n, rerr := f.Read(buf)
        if n > 0 {
            // Send는 윈도우가 막히면 여기서 대기 → 디스크 읽기도 자동으로 느려짐
            if serr := stream.Send(&pb.UploadChunk{
                Payload: &pb.UploadChunk_Data{Data: buf[:n]},
            }); serr != nil {
                return serr
            }
        }
        if rerr == io.EOF {
            break
        }
        if rerr != nil {
            return rerr
        }
    }
    res, err := stream.CloseAndRecv()
    if err != nil {
        return err
    }
    log.Printf("uploaded %d bytes, id=%s", res.GetBytesReceived(), res.GetFileId())
    return nil
}
```

`buf`를 청크마다 재사용한다는 점도 눈여겨보자(8절의 직렬화 비용 절감과 연결된다). 다만 `bytes` 필드는 마샬링할 때 복사되므로, `buf[:n]`을 재사용하더라도 protobuf가 내부적으로 복사본을 만든다는 점은 알아 두어야 한다.

---

## 6. 압축: 대역폭을 CPU와 맞바꾸기

### 6.1 메시지 단위 압축과 협상

gRPC는 **메시지 단위(per-message)** 압축을 지원한다. 스트림 전체를 압축하는 것이 아니라 gRPC 메시지를 하나씩 따로 압축한다. 이 방식은 3장(Protocol Buffers 2)에서 본 길이 접두사 프레이밍의 1바이트 **compressed-flag**와 직접 연결된다.

```text
gRPC 메시지 프레이밍 (와이어):

  +-----------+-------------------+-----------------------+
  | 1 byte    | 4 bytes (BE)      | N bytes               |
  | compressed| message length    | message data          |
  | flag      | (압축 후 길이)     | (압축됐을 수도)        |
  +-----------+-------------------+-----------------------+
       │
       ├─ 0x00: 압축 안 됨 (identity)
       └─ 0x01: 이 메시지는 grpc-encoding 헤더가 지정한 방식으로 압축됨
```

compressed-flag가 `0x01`이면 그 메시지 본문은 요청 또는 응답 헤더의 `grpc-encoding`이 지정한 알고리즘(예: `gzip`)으로 압축되어 있다. `0x00`이면 압축되지 않은 것이다. **이 플래그가 메시지마다 있으므로, 같은 스트림 안에서도 어떤 메시지는 압축하고 어떤 메시지는 압축하지 않을 수 있다.** 예를 들어 이미 압축된 JPEG 청크는 압축하지 않고(압축하면 역효과가 난다) 텍스트 청크만 압축하는 식으로 메시지마다 판단할 수 있다.

협상은 두 메타데이터 헤더([[08 - 메타데이터와 인터셉터]])로 이루어진다.

| 헤더 | 방향 | 의미 |
|------|------|------|
| `grpc-encoding` | 양방향 | "내가 보내는 이 메시지(들)는 이 방식으로 압축했다" |
| `grpc-accept-encoding` | 양방향 | "나는 이 알고리즘들을 풀 수 있다" |

표준 알고리즘은 `identity`(무압축), `gzip`, `deflate`다. 일부 구현에서는 `snappy` 등을 플러그인으로 추가할 수 있다.

```text
압축 협상 흐름:

  클라이언트 ──► 서버
    HEADERS:
      grpc-encoding: gzip              ← 이 요청 메시지는 gzip
      grpc-accept-encoding: gzip,identity ← 응답은 gzip 또는 무압축 받겠다

  서버 ──► 클라이언트
    HEADERS:
      grpc-encoding: gzip              ← 응답도 gzip으로
      grpc-accept-encoding: gzip,deflate,identity

  만약 서버가 클라이언트의 grpc-encoding을 못 풀면?
  → UNIMPLEMENTED 상태로 거부하며 grpc-accept-encoding에
    자기가 지원하는 목록을 담아 응답
```

### 6.2 설정 방법

```go
// Go: gzip 압축 등록 및 사용
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/encoding/gzip" // import만 해도 등록됨
)

// 클라이언트: 모든 호출에 gzip 적용 (기본 압축기)
conn, _ := grpc.NewClient(target,
    grpc.WithDefaultCallOptions(grpc.UseCompressor(gzip.Name)),
)

// 또는 호출별로
resp, _ := client.GetData(ctx, req, grpc.UseCompressor(gzip.Name))
```

```java
// Java: 클라이언트 스텁에 압축 지정
MyServiceGrpc.MyServiceBlockingStub stub =
    MyServiceGrpc.newBlockingStub(channel).withCompression("gzip");

// 서버 측: 인터셉터나 ServerCallStreamObserver.setCompression("gzip")
```

```python
# Python
import grpc
resp = stub.GetData(req, compression=grpc.Compression.Gzip)
```

### 6.3 압축의 트레이드오프: 언제 켜고 언제 끄는가

압축에도 비용이 든다. **대역폭을 줄이는 대신 CPU를 쓴다.** 이 트레이드오프를 정확히 이해해야 한다.

| 상황 | 압축 권장? | 이유 |
|------|-----------|------|
| 큰 텍스트/JSON/반복적 protobuf | 켜기 | 압축률이 높고(70~90%) 대역폭 절감 효과가 큼 |
| 이미 압축된 데이터(JPEG, mp4, gzip) | 끄기 | 크기가 거의 줄지 않고 CPU만 낭비하며, 때로는 더 커짐 |
| 작은 메시지(수백 바이트 이하) | 보통 끄기 | gzip 헤더/사전 오버헤드가 절감량보다 커서 역효과 |
| 같은 DC 내 고대역폭 링크 | 신중 | 대역폭이 병목이 아니면 CPU만 더 씀 |
| 대륙간/저대역폭 링크 | 켜기 | 대역폭이 실제 병목이므로 압축이 RTT당 처리량을 개선 |

**작은 메시지에서 생기는 역효과를 구체적으로 보자.** gzip에는 최소한의 헤더(10바이트)와 트레일러(8바이트, CRC32 + 길이)가 붙는다. 100바이트짜리 메시지를 gzip으로 압축하면 다음과 같다.

```text
원본 100바이트 → gzip → 18바이트(헤더+트레일러) + 압축된 본문(~90바이트?)
                       = 약 108바이트  ← 오히려 커짐!
```

작고 엔트로피가 높은 메시지는 gzip으로 압축하면 오히려 커진다. 그래서 많은 구현이 압축해서 손해를 보면 무압축(compressed-flag=0)으로 보내는 최적화를 한다. 하지만 여기에 의존하지 말자. **작은 메시지가 대부분인 서비스라면 압축을 끄는 편이 대개 더 빠르다.**

**CPU 비용의 실체**: gzip 압축은 메시지마다 수십 마이크로초에서 수 밀리초의 CPU 시간을 쓴다. 초당 수만 건의 RPC를 처리하는 서버에서 모든 메시지를 gzip으로 압축하면 CPU 코어 하나를 통째로 압축에만 쓸 수도 있다. 측정 없이 "압축을 켜면 빨라지겠지"라고 생각하는 것은 위험한 가정이다. 반드시 [[14 - 관찰성과 디버깅 - Reflection grpcurl]]에서 소개한 도구와 이 장 10절의 벤치마킹으로 검증하자.

---

## 7. 동시성과 연결: MAX_CONCURRENT_STREAMS의 벽

### 7.1 단일 연결이 병목이 되는 지점

[[07 - 채널 스텁 커넥션 생명주기]]에서 봤듯이, gRPC 채널 하나는 보통 **HTTP/2 연결 하나(또는 소수의 연결)** 위에서 동작한다. 연결 하나 위로 여러 RPC를 멀티플렉싱할 수 있다는 것이 HTTP/2의 강점이지만, 여기에도 한계가 있다.

`SETTINGS_MAX_CONCURRENT_STREAMS`(설정 식별자 `0x3`)는 **연결 하나에서 동시에 열 수 있는 스트림(=진행 중인 RPC)의 최대 개수**다.

- RFC 7540/9113은 이 설정의 기본값을 무제한으로 두지만, 실무에서 서버는 보안과 리소스 보호를 위해 **보통 100~250** 정도로 제한한다. 과거 gRPC-Go 서버는 사실상 무제한(`math.MaxUint32`)이 기본값이었으나, HTTP/2 Rapid Reset 취약점(CVE-2023-44487, 2023) 이후 **기본값이 100으로 낮아졌다.** 연결 하나로 수많은 스트림을 열자마자 RST_STREAM으로 닫는 공격을 막기 위한 조치다. Envoy나 nginx 같은 프록시도 100 안팎으로 설정하는 경우가 많다.
- 클라이언트가 이 한계를 넘겨 새 RPC를 시작하려고 하면, gRPC는 **새 스트림을 거부하지 않고 큐에 넣는다.** 진행 중인 스트림이 끝나 자리가 나면 대기 중인 RPC가 시작된다.

```text
MAX_CONCURRENT_STREAMS = 100 인 단일 연결:

  진행 중 RPC: [1][2][3]...[100]   ← 꽉 참
  대기 큐:     [101][102][103]...  ← 자리 날 때까지 대기
                  │
                  └─► 이 대기가 지연(latency)으로 나타남
                      "왜 p99가 튀지?" 의 흔한 원인
```

문제는 RPC가 오래 걸리거나(장시간 스트리밍) 동시에 수천 개를 보내야 할 때 생긴다. 이런 경우 단일 연결의 한계 100개가 병목이 된다. 101번째 RPC부터는 줄을 서야 하고, 처리량은 "100 / 평균 RPC 시간"으로 묶인다.

### 7.2 단일 연결의 HOL 블로킹

미묘한 문제가 하나 더 있다. 멀티플렉싱은 스트림별 흐름 제어로 HTTP/2 수준의 HOL 블로킹을 해결했지만, **TCP 수준의 HOL 블로킹은 여전히 남아 있다.** 단일 TCP 연결에서 패킷 하나가 손실되면, TCP는 그 패킷이 재전송될 때까지 그 뒤의 모든 바이트를 애플리케이션에 전달하지 못한다. 결국 그 연결 위의 모든 스트림이 함께 멈춘다.

```text
TCP HOL 블로킹 (단일 연결):

  Stream 1 데이터 ─┐
  Stream 3 데이터 ─┼─► [TCP 세그먼트 #5 손실!] ─► 수신
  Stream 5 데이터 ─┘         │
                       TCP가 #5 재전송 기다리는 동안
                       #6,#7... (다른 스트림 데이터 포함) 다 대기
                       → 무손실 스트림 3,5도 같이 멈춤
```

이것은 HTTP/2의 근본적인 한계이며, HTTP/3(QUIC)가 UDP 기반의 독립 스트림으로 해결한 바로 그 문제다. gRPC에서는 **연결을 여러 개로 나누면(채널 풀)** 한 연결에서 생긴 패킷 손실이 다른 연결의 스트림에 영향을 주지 않으므로 문제가 완화된다.

### 7.3 채널 풀: 연결을 늘려 병목 해소하기

해법은 **연결(채널)을 여러 개 만들어 RPC를 분산**하는 것이다. 이를 채널 풀링(channel pooling) 또는 멀티플 서브채널이라고 한다.

```go
// 간단한 라운드로빈 채널 풀 (Go)
type ChannelPool struct {
    conns []*grpc.ClientConn
    next  uint64
}

func NewChannelPool(target string, size int) (*ChannelPool, error) {
    p := &ChannelPool{conns: make([]*grpc.ClientConn, size)}
    for i := range p.conns {
        c, err := grpc.NewClient(target,
            grpc.WithTransportCredentials(insecure.NewCredentials()),
        )
        if err != nil {
            return nil, err
        }
        p.conns[i] = c
    }
    return p, nil
}

func (p *ChannelPool) Get() *grpc.ClientConn {
    // 라운드로빈으로 다음 연결 선택
    i := atomic.AddUint64(&p.next, 1)
    return p.conns[i%uint64(len(p.conns))]
}
```

채널 풀이 필요한 경우는 다음과 같다.

- 동시 RPC 수가 단일 연결의 `MAX_CONCURRENT_STREAMS`를 넘을 때
- 처리량이 매우 높아 연결당 처리량이 NIC나 CPU 한계에 닿을 때. 연결 하나는 CPU 코어 하나에서 처리되는 경향이 있으므로, 연결을 여러 개 두면 여러 코어를 활용할 수 있다.
- TCP HOL 블로킹의 영향을 분산하고 싶을 때

**주의**: 연결을 늘리면 로드밸런싱과도 얽힌다. [[12 - 이름 해석과 로드밸런싱]]에서 본 것처럼 L4 로드밸런서 뒤에서는 연결 하나가 백엔드 하나에 고정(pin)되므로, 단일 연결은 백엔드 하나에만 요청을 보낸다. 연결을 여러 개 만들거나 클라이언트 측 로드밸런싱(`round_robin`)을 쓰면 부하가 여러 백엔드로 퍼진다. 즉 채널 풀은 처리량뿐 아니라 **부하 분산**과도 직결된다.

연결당 처리량과 연결 수 사이의 트레이드오프는 다음과 같다.

| 전략 | 장점 | 단점 |
|------|------|------|
| 단일 연결 | 적은 소켓/메모리, 간단 | MAX_CONCURRENT_STREAMS 병목, TCP HOL, 단일 백엔드 |
| 소수 연결 풀(2~8) | 병목 완화, 멀티코어 활용, 부하 분산 | 약간의 리소스 증가 |
| 과도한 연결(수십~수백) | - | 소켓/메모리 낭비, keepalive PING 오버헤드 폭증, 서버 부담 |

**원칙**: 무작정 늘리지 말고 단일 연결로 시작한 뒤, 벤치마크에서 병목이 확인되면 조금씩 늘리자.

---

## 8. 직렬화 비용과 최적화

### 8.1 marshal/unmarshal에도 비용이 든다

gRPC RPC 한 번에는 눈에 보이지 않는 CPU 비용이 숨어 있다. 바로 **protobuf 직렬화(marshal)와 역직렬화(unmarshal)** 다. 메시지가 크거나 필드가 많거나 RPC가 초당 수만 번 호출되면, 이 비용이 전체 CPU 사용량의 상당 부분을 차지한다.

```text
한 번의 Unary RPC에서 일어나는 복사/변환 (개략):

  [앱 객체] ──marshal──► [protobuf 바이트] ──► [압축?] ──► [gRPC 프레이밍]
                                                              │
                                                         [HTTP/2 DATA 프레임]
                                                              │ 와이어
  [앱 객체] ◄──unmarshal── [protobuf 바이트] ◄── [압축 해제?] ◄── [디프레이밍]

  각 화살표마다 메모리 할당 + 복사가 일어날 수 있음
  → 고RPS에서 GC 압박(Java/Go)의 주범
```

### 8.2 언어별 최적화 기법

**Go: 메시지와 버퍼 재사용, proto 옵션**

- gRPC-Go는 메시지를 풀(pool)로 관리하는 옵션과 `mem` 패키지 기반의 버퍼 재사용을 점차 도입해 왔다. 핵심은 큰 `bytes` 필드를 다룰 때 불필요한 복사를 줄이는 것이다.
- 직접 할 수 있는 방법은 요청과 응답 객체를 루프에서 재사용해(`proto.Reset(msg)` 후 재사용) 할당을 줄이는 것이다. 다만 동시성 안전성에 주의해야 한다.

**Java: arena 대신 객체 풀과 zero-copy**

- Java protobuf 메시지는 불변(immutable)이라 arena 할당이 없다. 대신 `CodedInputStream`/`CodedOutputStream`을 직접 다루거나, `ByteString.unsafeWrap`으로 복사를 피할 수 있다.
- gRPC-Java는 Netty의 `ByteBuf`를 활용해 일부 zero-copy 경로를 제공한다. 큰 `bytes`는 가능하면 `ByteString` 그대로 다뤄 byte[] 복사를 피한다.

**C++: Arena 할당**

- protobuf C++는 **Arena**를 지원한다. RPC 하나에서 만드는 모든 메시지 객체를 하나의 큰 메모리 블록(arena)에 할당하고, RPC가 끝나면 arena를 통째로 해제한다. 객체마다 하던 `new`/`delete`가 없어지므로 할당과 해제 비용, 메모리 단편화가 크게 줄어든다.

```cpp
// C++ Arena 예시
google::protobuf::Arena arena;
// 메시지를 arena에 할당 — 개별 delete 불필요
MyMessage* msg = google::protobuf::Arena::CreateMessage<MyMessage>(&arena);
msg->set_field(...);
// arena가 스코프를 벗어나면 모든 객체 일괄 해제 (개별 소멸자 호출 없음)
```

### 8.3 zero-copy bytes와 복사 줄이기

직렬화와 역직렬화에서 복사 비용이 가장 큰 것은 `bytes` 필드(또는 큰 문자열)다. 핵심 전략은 다음과 같다.

- **불필요한 변환 피하기**: `bytes`로 받은 데이터를 굳이 `string`으로 바꾸거나 다른 버퍼로 다시 복사하지 않는다.
- **참조 전달**: 가능하면 역직렬화된 버퍼를 그대로 참조해서 사용한다(Java의 `ByteString`, Go의 `[]byte` 슬라이스 공유).
- **proto 설계로 줄이기**: 거대한 `bytes` 하나보다 청크 스트리밍(5.2절)이 메모리 사용량의 최고치와 복사를 줄인다.

```proto
// 비효율: 큰 데이터를 매번 base64 문자열로
message BadBlob {
  string data_base64 = 1;  // base64는 33% 더 큼 + 인코딩/디코딩 CPU
}

// 효율: bytes 사용
message GoodBlob {
  bytes data = 1;          // 바이너리 그대로, 복사/변환 최소
}
```

[[02 - Protocol Buffers 1 - 문법과 타입 시스템]]에서 본 것처럼 `bytes`와 `string` 중 무엇을 쓸지는 단순한 타입 선택이 아니라 성능에 관한 결정이다. 바이너리 데이터에 `string`을 쓰면 UTF-8 검증 오버헤드까지 더해진다.

---

## 9. 작은 RPC 다발의 오버헤드: HPACK, keepalive, 배칭

### 9.1 헤더 오버헤드와 HPACK

RPC 하나에는 페이로드뿐 아니라 **헤더**도 함께 전송된다(8장 메타데이터와 인터셉터). gRPC 요청 헤더에는 `:method: POST`, `:path: /pkg.Service/Method`, `content-type: application/grpc`, `te: trailers`, `grpc-timeout`, `authorization` 등 십수 개의 헤더가 들어간다. 이 헤더를 RPC마다 평문으로 보내면, 페이로드가 수십 바이트인 작은 RPC에서는 헤더가 페이로드보다 훨씬 커진다.

4장(HTTP/2 깊이 보기)에서 다룬 **HPACK**이 이 문제를 완화한다. HPACK은 두 가지 테이블을 쓴다.

- **정적 테이블**: 자주 쓰는 헤더(`:method: POST` 등)에 번호를 매겨 1~2바이트로 표현한다.
- **동적 테이블**: 연결이 유지되는 동안 반복되는 헤더(`:path`, `authorization`)를 한 번 보내면 인덱스에 등록하고, 이후에는 인덱스 번호만 전송한다.

```text
HPACK 효과 (같은 연결에서 두 번째 RPC부터):

  첫 RPC:  :path: /order.v1.OrderService/GetOrder  (전체 전송, 동적 테이블에 등록)
  둘째 RPC: [인덱스 62]                              (1~2바이트로 축약!)

  → 같은 메서드를 반복 호출하면 헤더 오버헤드가 거의 사라진다
```

그래서 **같은 연결로 같은 메서드를 반복해서 호출하는 패턴이 HPACK에 유리**하다. 매번 새 연결을 맺으면 동적 테이블이 초기화되어 HPACK의 이득을 보지 못한다. 연결을 재사용해야 하는(7장 채널 스텁 커넥션 생명주기) 또 하나의 이유다.

### 9.2 keepalive PING 오버헤드

13장(안정성)에서 다룰 keepalive는 죽은 연결을 빨리 감지하기 위해 주기적으로 PING 프레임을 보낸다. 하지만 지나치면 오버헤드가 된다.

- PING을 너무 자주(예: 1초마다) 보내면 네트워크와 CPU를 낭비하고, 서버가 `ENHANCE_YOUR_CALM`(too_many_pings)으로 연결을 끊을 수 있다.
- 연결이 많을수록(채널 풀이 과도할수록) PING 총량이 연결 수만큼 곱해진다. 연결 100개가 10초마다 PING을 보내면 초당 10번의 PING이 나간다.

keepalive 주기는 보통 분 단위(예: `time: 30s~5m`)로 잡고, 유휴(idle) 상태에서는 PING을 보내지 않도록 `permitWithoutStream=false`를 고려하는 것이 좋다. 자세한 튜닝 방법은 13장(안정성)을 참고하자.

### 9.3 배칭: 작은 RPC를 묶기

초당 수만 개의 작은 RPC를 보내면 RPC마다 드는 고정 비용(헤더, 프레이밍, 흐름 제어 계산, 컨텍스트 전환)이 누적되어 비효율적이다. 해법은 여러 논리적 요청을 RPC 하나로 묶는 **배칭(batching)** 이다.

```proto
// 비효율: 1건씩 N번 호출
service ItemService {
  rpc GetItem(GetItemRequest) returns (Item);  // 1000개면 1000 RPC
}

// 효율: 배치 RPC
service ItemServiceBatched {
  rpc BatchGetItems(BatchGetItemsRequest) returns (BatchGetItemsResponse);
}
message BatchGetItemsRequest {
  repeated string item_ids = 1;   // 한 번에 1000개 ID
}
message BatchGetItemsResponse {
  repeated Item items = 1;        // 한 번에 1000개 응답
}
```

배칭은 RPC 고정 비용을 N분의 1로 줄이지만, 지연(첫 결과를 받으려면 전체를 기다려야 함)과 부분 실패(일부만 성공한 경우) 처리를 신중하게 설계해야 한다. 스트리밍은 배칭의 대안이 될 수 있다. 요청을 묶지 않고도 연결을 재사용하면서 결과를 조금씩 흘려보낸다.

---

## 10. 벤치마킹: 측정 없이는 최적화도 없다

### 10.1 ghz로 부하 걸기

`ghz`는 gRPC 전용 부하 테스트 도구로, HTTP의 `wrk`나 `hey`에 해당한다.

```bash
# 기본: 단일 메서드에 200개 동시 연결로 10000 요청
ghz --insecure \
  --proto ./order.proto \
  --call order.v1.OrderService.GetOrder \
  -d '{"order_id": "ord_123"}' \
  -n 10000 \
  -c 200 \
  localhost:50051

# 지속 시간 기반 + RPS 제한
ghz --insecure \
  --proto ./order.proto \
  --call order.v1.OrderService.GetOrder \
  -d '{"order_id": "ord_123"}' \
  -z 60s \
  -c 50 \
  --rps 5000 \
  localhost:50051

# 연결 수와 연결당 동시성을 분리 (채널 풀 효과 측정)
ghz --insecure \
  --proto ./order.proto \
  --call order.v1.OrderService.GetOrder \
  -d '{"order_id": "ord_123"}' \
  --connections 10 \
  --concurrency 200 \
  -z 30s \
  localhost:50051
```

핵심은 `--connections`(연결 수)와 `--concurrency`(동시 RPC 수)를 따로 지정하는 것이다. 7절에서 본 MAX_CONCURRENT_STREAMS와 채널 풀의 효과를 이 두 옵션으로 실험할 수 있다. 예를 들어 `--connections 1 --concurrency 500`으로 단일 연결 병목을 재현하고, `--connections 10 --concurrency 500`으로 풀의 효과를 확인한다.

리플렉션(14장 관찰성과 디버깅)이 켜져 있으면 `--proto` 없이도 동작한다.

### 10.2 무엇을 측정할 것인가

ghz 출력에서 봐야 할 지표는 다음과 같다.

```text
Summary:
  Count:        300000
  Total:        30.01 s
  Slowest:      152.33 ms
  Fastest:      0.81 ms
  Average:      4.92 ms
  Requests/sec: 9996.34          ← 처리량(throughput)

Latency distribution:
  10 % in 1.42 ms
  25 % in 2.10 ms
  50 % in 3.85 ms                ← p50 (중앙값)
  75 % in 6.20 ms
  90 % in 9.51 ms
  95 % in 13.80 ms               ← p95
  99 % in 41.20 ms               ← p99 (꼬리 지연, 가장 중요)

Status code distribution:
  [OK]                299980     ← 성공
  [ResourceExhausted]     12     ← 메시지 한계? 동시성 한계?
  [DeadlineExceeded]       8     ← 타임아웃
```

| 지표 | 왜 중요한가 |
|------|-------------|
| **Throughput (RPS)** | 시스템의 처리 능력. 목표 부하를 견디는지 보여 준다. |
| **p50 (중앙값)** | 일반적인 사용자 경험. |
| **p99 / p99.9 (꼬리 지연)** | 가장 중요하다. 평균은 실상을 가린다. p99가 튀면 GC, 큐잉, HOL 블로킹을 의심한다. |
| **연결 수 / 동시성** | 병목 위치 진단(단일 연결 vs 다중). |
| **에러 분포** | RESOURCE_EXHAUSTED(한계), DEADLINE_EXCEEDED(과부하), UNAVAILABLE(연결) 비율. |

**평균(average)에 속으면 안 된다.** 평균이 5ms인데 p99가 41ms라면 요청의 1%가 8배 느리다는 뜻이고, 초당 1만 RPS라면 그런 요청이 초당 100건이다. 사용자가 체감하는 성능과 SLO는 꼬리 지연이 좌우한다.

### 10.3 흔한 성능 함정 세 가지

**함정 1: 단일 연결 HOL, "동시성을 올려도 처리량이 오르지 않는다"**

- 증상: `--concurrency`를 100에서 500으로 올려도 RPS가 그대로다.
- 원인: `--connections 1`이라서 단일 연결의 MAX_CONCURRENT_STREAMS나 단일 코어의 처리 한계에 묶여 있다.
- 진단: `--connections`를 늘려 보고 RPS가 오르면 단일 연결 병목으로 확정한다.
- 해결: 채널 풀(7.3절) 또는 클라이언트 측 로드밸런싱(12장 이름 해석과 로드밸런싱)을 쓴다.

```bash
# 진단: 연결 수만 바꿔가며 비교
for conns in 1 2 4 8 16; do
  echo "=== connections=$conns ==="
  ghz --insecure --proto ./order.proto \
    --call order.v1.OrderService.GetOrder -d '{"order_id":"x"}' \
    --connections $conns --concurrency 200 -z 20s \
    localhost:50051 | grep "Requests/sec"
done
```

**함정 2: GC 압박, "처리량은 괜찮은데 p99가 주기적으로 튄다"**

- 증상: p99 지연이 톱니 모양으로 주기적으로 치솟는다(Java/Go).
- 원인: RPC마다 메시지 객체를 새로 할당하므로 GC가 자주 돌며 stop-the-world가 발생한다.
- 진단: GC 로그(`-Xlog:gc` / `GODEBUG=gctrace=1`)와 p99 스파이크가 시간상으로 겹치는지 확인한다.
- 해결: 8절의 객체 재사용과 arena, 메시지 크기 축소, 압축으로 할당량을 줄이고 GC를 튜닝한다.

**함정 3: 동기 블로킹 스레드 고갈, "동시 요청이 많아지면 지연이 폭증한다"**

- 증상: 동시 RPC 수가 스레드풀 크기를 넘으면 지연이 계단식으로 늘어난다.
- 원인: Java나 Python의 동기 서버는 RPC마다 스레드를 하나씩 점유한다. 스레드풀(예: 200)이 꽉 차면 나머지 요청은 큐에서 기다린다. 핸들러가 블로킹 I/O(DB, 외부 호출)를 하면 스레드가 오래 묶인다.
- 진단: 스레드 덤프에서 대부분의 스레드가 블로킹 상태인지, 큐 길이가 늘어나는지 확인한다.
- 해결: 스레드풀 크기를 조정하거나, 비동기·논블로킹 핸들러를 쓰거나, 백프레셔로 유입량을 제한한다. 무작정 스레드를 늘리면 컨텍스트 전환 비용 때문에 역효과가 난다.

```text
스레드 고갈 시각화:

  스레드풀(200) ─ 전부 DB 호출에서 블록 (각 50ms)
       │
  새 RPC 201~1000 ─► [대기 큐] ─► 200개 끝날 때까지 대기
       │
  대기 시간이 처리 시간에 누적 → p99 = 큐 대기 + 처리 = 폭증

  처방: 핸들러를 논블로킹으로 → 스레드가 I/O 대기 중 다른 RPC 처리
```

---

## 11. 종합 예제: 백프레셔를 지키는 스트리밍과 튜닝된 채널

지금까지 살펴본 내용을 하나로 모아 보자. 서버 스트리밍에서 백프레셔를 지키면서 메시지 크기, 압축, keepalive, 윈도우를 튜닝한 서버와 클라이언트 코드다.

### 11.1 proto

```proto
syntax = "proto3";
package telemetry.v1;

service MetricService {
  // 서버가 대량의 메트릭 포인트를 스트리밍. 느린 클라이언트에 백프레셔 필요.
  rpc StreamMetrics(StreamMetricsRequest) returns (stream MetricPoint);
}

message StreamMetricsRequest {
  string source_id = 1;
  int64  from_unix_ms = 2;
  int64  to_unix_ms = 3;
}

message MetricPoint {
  int64  ts_unix_ms = 1;
  double value = 2;
  map<string, string> labels = 3;
}
```

### 11.2 Go 서버: 블로킹 Send로 얻는 자연스러운 백프레셔와 튜닝 옵션

```go
package main

import (
    "log"
    "net"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/keepalive"
    _ "google.golang.org/grpc/encoding/gzip"
    pb "example.com/telemetry/gen"
)

type metricServer struct {
    pb.UnimplementedMetricServiceServer
    store MetricStore
}

func (s *metricServer) StreamMetrics(
    req *pb.StreamMetricsRequest,
    stream pb.MetricService_StreamMetricsServer,
) error {
    cursor := s.store.Scan(req.GetSourceId(), req.GetFromUnixMs(), req.GetToUnixMs())
    defer cursor.Close()

    // 메시지 객체 재사용 (할당/GC 압박 감소)
    pt := &pb.MetricPoint{}

    for cursor.Next() {
        // 데드라인/취소 존중 — 영원히 블록 방지
        if err := stream.Context().Err(); err != nil {
            return err
        }
        cursor.Fill(pt) // pt 필드를 덮어씀 (새 할당 없음)

        // Send()는 흐름 제어 윈도우가 닫혀 있으면 여기서 블록된다.
        // 클라이언트가 느리면 이 고루틴이 멈추고 cursor.Next()도 안 불린다.
        // → 저장소에서 더 안 읽는다 → 메모리 안정 (자연 백프레셔)
        if err := stream.Send(pt); err != nil {
            return err
        }
    }
    return cursor.Err()
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatal(err)
    }
    srv := grpc.NewServer(
        // 메시지 크기 한계
        grpc.MaxRecvMsgSize(8*1024*1024),
        grpc.MaxSendMsgSize(8*1024*1024),
        // 흐름 제어 윈도우 (BDP가 크면 키운다; 모르면 생략해 자동튜닝)
        grpc.InitialWindowSize(1<<20),     // 1 MiB/stream
        grpc.InitialConnWindowSize(1<<22), // 4 MiB/conn
        // 동시 스트림 한계 (리소스 보호)
        grpc.MaxConcurrentStreams(250),
        // keepalive 정책 (과도한 PING 방지)
        grpc.KeepaliveParams(keepalive.ServerParameters{
            Time:    2 * time.Minute,
            Timeout: 20 * time.Second,
        }),
        grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
            MinTime:             1 * time.Minute, // 이보다 잦은 클라 PING은 거부
            PermitWithoutStream: false,
        }),
    )
    pb.RegisterMetricServiceServer(srv, &metricServer{store: NewStore()})
    log.Println("listening :50051")
    if err := srv.Serve(lis); err != nil {
        log.Fatal(err)
    }
}
```

### 11.3 Go 클라이언트: Recv 속도로 백프레셔 전파하기

```go
func consumeMetrics(ctx context.Context, client pb.MetricServiceClient) error {
    stream, err := client.StreamMetrics(ctx, &pb.StreamMetricsRequest{
        SourceId:   "sensor-42",
        FromUnixMs: 0,
        ToUnixMs:   time.Now().UnixMilli(),
    })
    if err != nil {
        return err
    }
    for {
        pt, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        // 무거운 처리. 이게 느리면 Recv 호출도 느려지고,
        // 라이브러리가 WINDOW_UPDATE를 늦게 보내 송신 측이 자동으로 느려진다.
        if err := handleSlow(pt); err != nil {
            return err
        }
    }
}
```

### 11.4 Java 서버: isReady()/onReadyHandler로 명시적 백프레셔

```java
public class MetricServiceImpl extends MetricServiceGrpc.MetricServiceImplBase {

    private final MetricStore store;

    public MetricServiceImpl(MetricStore store) { this.store = store; }

    @Override
    public void streamMetrics(StreamMetricsRequest req,
                              StreamObserver<MetricPoint> responseObserver) {
        ServerCallStreamObserver<MetricPoint> obs =
            (ServerCallStreamObserver<MetricPoint>) responseObserver;

        // 데이터 소스를 풀(pull) 방식으로 당겨오는 이터레이터
        Iterator<MetricPoint> cursor =
            store.scan(req.getSourceId(), req.getFromUnixMs(), req.getToUnixMs());

        obs.setOnReadyHandler(() -> {
            // 전송 가능한 동안만 보낸다. isReady()가 false가 되면 멈추고
            // 큐가 비워져 다시 ready가 되면 이 핸들러가 또 호출된다.
            while (obs.isReady() && cursor.hasNext()) {
                obs.onNext(cursor.next());
            }
            if (!cursor.hasNext()) {
                obs.onCompleted();
            }
        });

        // 클라이언트 취소 시 생산 중단
        obs.setOnCancelHandler(() -> {
            // cursor.close() 등 리소스 정리
        });
    }
}

// 서버 빌더 튜닝
Server server = NettyServerBuilder.forPort(50051)
    .maxInboundMessageSize(8 * 1024 * 1024)
    .flowControlWindow(4 * 1024 * 1024)
    .maxConcurrentCallsPerConnection(250)
    .keepAliveTime(2, TimeUnit.MINUTES)
    .keepAliveTimeout(20, TimeUnit.SECONDS)
    .permitKeepAliveTime(1, TimeUnit.MINUTES)
    .addService(new MetricServiceImpl(store))
    .build();
```

이 Java 서버와 11.2절의 Go 서버는 **같은 백프레셔 목표를 서로 다른 메커니즘으로** 달성한다. Go에서는 블로킹 Send가 루프를 멈춰 주고, Java에서는 isReady()/onReadyHandler가 생산을 명시적으로 멈췄다가 재개한다. 두 방식 모두 **"와이어가 막히면 생산도 멈춘다"** 는 같은 원칙을 따른다.

---

## 12. 성능 튜닝 의사결정 트리

지금까지 살펴본 설정 항목을 언제 조정해야 하는지 한눈에 정리하면 다음과 같다.

```text
"gRPC가 느리다/처리량이 안 난다" 진단 순서:

1. 먼저 측정했는가? ─── 아니오 → ghz로 throughput/p50/p99/에러 측정부터
   │ 예
   ▼
2. 에러가 있는가?
   ├─ RESOURCE_EXHAUSTED → 메시지 크기 한계 / 동시성 한계 (5절, 7절)
   ├─ DEADLINE_EXCEEDED  → 과부하 또는 데드라인 짧음 ([[10 - Deadline 취소 타임아웃]])
   └─ UNAVAILABLE        → 연결/LB 문제 ([[12 - 이름 해석과 로드밸런싱]], [[13 - 안정성 - Retry Health Check Keepalive]])
   │ 에러 없음
   ▼
3. 동시성 올려도 RPS 정체? ─── 예 → 단일 연결 병목 → 채널 풀/LB (7절)
   │ 아니오
   ▼
4. p99가 주기적으로 튐? ─── 예 → GC 압박 → 객체 재사용/arena, 할당 감소 (8절)
   │ 아니오
   ▼
5. 대륙간/저대역폭 + 큰 메시지? ─── 예 → 압축 켜기 + 윈도우 BDP 맞춤 (3절, 6절)
   │ 아니오
   ▼
6. 빠른 생산자-느린 소비자 메모리 증가? ─── 예 → 백프레셔 점검 (4절)
                                              isReady()/blocking Send/request(n)
   │ 아니오
   ▼
7. 작은 RPC 다발? ─── 예 → 배칭/스트리밍 + 연결 재사용(HPACK 이득) (9절)
```

---

## 13. 자주 하는 오해 바로잡기

**오해 1: "흐름 제어를 켜면 느려진다."**
그렇지 않다. 흐름 제어는 HTTP/2의 필수 기능이라 항상 켜져 있고 끌 수도 없다. 흐름 제어는 처리량을 줄이는 장치가 아니라, 메모리 안정성을 보장하면서 안전하게 낼 수 있는 최대 처리량을 찾는 장치다. 윈도우가 BDP에 맞게 잡혀 있으면 흐름 제어가 있어도 최대 처리량이 나온다.

**오해 2: "압축은 항상 좋다."**
그렇지 않다. 작은 메시지, 이미 압축된 데이터, 고대역폭 내부 링크에서는 CPU만 낭비하고 때로는 더 느려진다(6.3절).

**오해 3: "연결을 많이 만들수록 빠르다."**
그렇지 않다. 적정 수(2~8)를 넘으면 소켓, 메모리, keepalive PING 오버헤드가 늘고 서버 부담이 커진다(7.3절). 측정해서 최적점을 찾아야 한다.

**오해 4: "윈도우는 크게 잡을수록 좋다."**
그렇지 않다. 윈도우는 받는 쪽이 미리 확보해 두는 버퍼 예산이다. BDP를 넘는 윈도우는 메모리만 더 쓸 뿐 처리량 이득이 없고, 소비자가 느릴 때는 더 많은 데이터를 메모리에 쌓는다(3.3절).

**오해 5: "Java에서 onNext()를 그냥 호출해도 백프레셔가 알아서 걸린다."**
그렇지 않다. Java 콜백 모델에서 isReady()를 무시하면 내부 버퍼에 데이터가 한없이 쌓인다(4.3절). isReady()와 onReadyHandler를 명시적으로 써야 한다. 블로킹 Send만으로 백프레셔가 저절로 걸리는 Go와 결정적으로 다른 점이다.
