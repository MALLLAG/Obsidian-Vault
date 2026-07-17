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

이 장이 답하는 질문:

- HTTP/2의 흐름 제어는 왜 TCP가 이미 하고 있는 혼잡 제어 위에 또 한 겹을 얹는가? 커넥션 윈도우와 스트림 윈도우라는 두 개의 창문은 각각 무엇을 막는가?
- 초기 윈도우 65535바이트라는 작은 숫자가 어떻게 100Gbps 광케이블의 처리량을 한 자릿수 Mbps로 떨어뜨리는가? BDP 자동 튜닝은 이 문제를 어떻게 푸는가?
- 흐름 제어라는 "전송 계층의 압력"이 어떻게 Java의 `onReadyHandler`, Go의 블로킹 `Send()`, ServerCallStreamObserver의 `request(n)`까지 도달해서 빠른 생산자가 느린 소비자를 압사시키지 않게 막는가?
- 4MB 메시지 한계, gzip 압축, `MAX_CONCURRENT_STREAMS`, 직렬화 비용, HPACK 헤더 압축은 각각 어떤 병목을 만들고 어떻게 완화하는가?
- ghz로 무엇을 측정해야 하고, "처리량이 안 나온다"는 막연한 증상의 진짜 원인(단일 연결 HOL, GC 압박, 동기 스레드 고갈)은 어떻게 식별하는가?

---

## 0. 들어가며: 성능은 "밀어 넣기"가 아니라 "흘려보내기"다

많은 엔지니어가 처음 gRPC 성능을 고민할 때 던지는 질문은 이렇다. "어떻게 하면 더 빨리 보낼 수 있지?" 이 질문은 절반만 맞다. 분산 시스템에서 처리량을 결정하는 것은 보내는 쪽의 속도가 아니라, **보내는 쪽과 받는 쪽 사이에서 데이터가 쌓이지 않고 균형을 이루는 지점**이다.

비유를 하나 들어보자. 당신이 거대한 물탱크(빠른 생산자)에서 작은 컵(느린 소비자)으로 물을 따른다고 하자. 탱크의 밸브를 활짝 열면 컵은 넘치고 물은 바닥에 쏟아진다. 컴퓨터에서 이 "바닥에 쏟아진 물"은 어디로 갈까? 그냥 사라지지 않는다. 받는 쪽 OS 버퍼에, 받는 쪽 애플리케이션 큐에, 보내는 쪽 송신 버퍼에 차곡차곡 쌓인다. 결국 누군가의 힙(heap)이 OutOfMemory로 터지거나, GC가 멈추거나, 컨테이너가 OOMKilled로 죽는다.

흐름 제어(flow control)와 백프레셔(backpressure)는 이 "넘치는 물"을 원천 차단하는 메커니즘이다. 핵심 아이디어는 단순하다. **받는 쪽이 "나 이만큼만 받을 수 있어"라고 명시적으로 허락하기 전까지, 보내는 쪽은 그 양을 넘겨 보내지 않는다.** 받는 쪽이 데이터를 소화하면 "이제 더 받을 수 있어"라고 신용(credit)을 돌려준다. 이 신용을 다 쓰면 보내는 쪽은 멈춘다. 멈춘 압력은 와이어를 거슬러 올라가 애플리케이션 코드의 `Send()` 호출을 블로킹시키거나, "지금은 보내지 마"라는 신호(`isReady() == false`)로 나타난다.

이 장은 그 압력이 어떻게 만들어지고, 어떻게 와이어를 타고, 어떻게 당신의 코드까지 전달되는지를 바이트 레벨에서부터 애플리케이션 콜백까지 추적한다. HTTP/2 프레임의 세부는 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서, 스트리밍 통신의 4가지 형태는 [[05 - 통신의 4가지 방식 - Unary와 Streaming]]에서 다뤘으니, 여기서는 그 위에서 "압력"이 어떻게 동작하는지에 집중한다.

---

## 1. 왜 또 흐름 제어인가: TCP 위에 한 겹 더

### 1.1 TCP는 이미 흐름 제어를 한다. 그런데 왜?

TCP를 아는 사람은 의문을 품는다. "TCP에 이미 수신 윈도우(receive window)가 있고, 혼잡 제어(congestion control)도 있는데, HTTP/2가 또 흐름 제어를 한다고?"

맞다. 그리고 이 중복은 멀티플렉싱(multiplexing) 때문에 **반드시 필요하다.**

[[04 - HTTP2 깊이 보기 - 전송 계층]]에서 봤듯, HTTP/2는 하나의 TCP 연결 위에 여러 개의 논리적 스트림(stream)을 동시에 실어 나른다. gRPC에서 각 RPC 호출은 하나의 스트림이다. 즉, 하나의 TCP 소켓 위로 수십, 수백 개의 RPC가 프레임 단위로 잘게 쪼개져 인터리빙(interleaving)되어 흐른다.

이제 문제가 보인다. TCP의 수신 윈도우는 **연결 전체**에 대해 작동한다. TCP는 그 연결 위에 몇 개의 스트림이 있는지, 어떤 스트림이 빠르고 어떤 스트림이 느린지 전혀 모른다. TCP에게는 그냥 한 줄기 바이트 스트림일 뿐이다.

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

만약 Stream 1의 소비자가 데이터를 안 읽어서 OS 수신 버퍼가 가득 차면, TCP는 윈도우를 0으로 줄여 송신 측 전체를 멈춘다. 그러면 멀쩡하게 빨리 소비되고 있던 Stream 3, Stream 5까지 같이 멈춘다. 이것이 **헤드 오브 라인 블로킹(Head-of-Line blocking, HOL)** 의 한 형태다.

HTTP/2의 흐름 제어는 이 문제를 **스트림 단위로** 풀기 위해 존재한다. 각 스트림마다 독립적인 윈도우를 두어, 느린 Stream 1만 멈추고 빠른 Stream 3은 계속 흐르게 한다. TCP가 "연결 전체"를 보는 거시 제어라면, HTTP/2 흐름 제어는 "스트림 하나하나"를 보는 미시 제어다. 둘은 층위가 다르며, 중복이 아니라 보완이다.

### 1.2 두 개의 창문: 커넥션 윈도우와 스트림 윈도우

HTTP/2(RFC 7540, 그리고 이를 개정한 RFC 9113)는 흐름 제어를 **두 레벨**로 정의한다.

| 레벨                 | 적용 범위                | 스트림 식별자      | 막는 것                     |
| ------------------ | -------------------- | ------------ | ------------------------ |
| 커넥션(connection) 레벨 | 연결 위 모든 DATA 프레임의 총합 | Stream ID 0  | 연결 전체가 받는 쪽 메모리를 압도하는 것  |
| 스트림(stream) 레벨     | 개별 스트림 하나            | 해당 Stream ID | 한 RPC가 다른 RPC의 몫을 독식하는 것 |

규칙은 단순하면서 엄격하다. **DATA 프레임을 하나 보내려면, 그 프레임의 페이로드 크기만큼 커넥션 윈도우와 스트림 윈도우 둘 다에 잔액이 있어야 한다.** 보내고 나면 두 윈도우 모두에서 그 크기만큼 차감한다. 둘 중 하나라도 0이면 그 스트림(또는 연결 전체)은 더 이상 DATA를 보낼 수 없다.

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

중요한 세부: **흐름 제어는 DATA 프레임에만 적용된다.** HEADERS, SETTINGS, PING, WINDOW_UPDATE, RST_STREAM 같은 제어성 프레임은 흐름 제어 대상이 아니다. 왜냐하면 흐름 제어가 제어 프레임까지 막아버리면, 윈도우를 늘려주는 WINDOW_UPDATE 자체가 흐름 제어에 걸려 데드락에 빠지기 때문이다. 신용을 돌려주는 메신저는 항상 통과할 수 있어야 한다.

### 1.3 신용기반(credit-based) 모델: 받는 쪽이 운전대를 쥔다

HTTP/2 흐름 제어의 철학을 한 문장으로 요약하면 이렇다. **보내는 쪽이 아니라 받는 쪽이 속도를 결정한다.**

이것이 신용기반(credit-based) 모델이다. 받는 쪽은 "내가 받을 수 있는 바이트 수"라는 신용을 쥐고 있고 이 신용을 송신 측에 광고(advertise)한다. 송신 측은 가진 신용만큼만 보낸다. 받는 쪽이 데이터를 처리해서 버퍼를 비우면, 비운 만큼 `WINDOW_UPDATE` 프레임을 보내 신용을 보충해준다.

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

여기서 결정적인 통찰: **받는 쪽이 데이터를 읽지 않으면 WINDOW_UPDATE를 보내지 않고, 그러면 윈도우는 0으로 수렴하고, 송신 측은 멈춘다.** 이 "멈춤"이 바로 백프레셔의 와이어 레벨 실체다. 받는 쪽 애플리케이션이 느리면 → 버퍼가 안 비워지고 → WINDOW_UPDATE가 안 나가고 → 윈도우가 닫히고 → 송신 측이 멈춘다. 압력이 전송 계층에서 자동으로 만들어진다.

---

## 2. WINDOW_UPDATE와 SETTINGS: 신용을 광고하고 보충하는 프레임

### 2.1 SETTINGS_INITIAL_WINDOW_SIZE: 시작점

연결이 맺어지면 양쪽은 `SETTINGS` 프레임을 교환한다. 그 안에 들어가는 파라미터 중 흐름 제어의 출발점이 `SETTINGS_INITIAL_WINDOW_SIZE`(설정 식별자 `0x4`)다.

- **기본값: 65535바이트 (64KiB - 1)**
- 최댓값: 2³¹ - 1 = 2147483647바이트 (약 2GiB)
- 이 값은 **새로 생성되는 모든 스트림**의 초기 윈도우 크기를 정한다.
- 단, **커넥션 레벨 윈도우의 초기값은 항상 65535로 고정**이며 `SETTINGS_INITIAL_WINDOW_SIZE`의 영향을 받지 않는다. 커넥션 윈도우를 키우려면 연결 직후 WINDOW_UPDATE를 보내야 한다.

이 마지막 디테일이 미묘하다. 정리하면:

| 윈도우 종류 | 초기값 | 변경 방법 |
|-------------|--------|-----------|
| 스트림 레벨 초기 윈도우 | `SETTINGS_INITIAL_WINDOW_SIZE` (기본 65535) | SETTINGS로 기본값 변경 + WINDOW_UPDATE로 개별 증감 |
| 커넥션 레벨 윈도우 | 항상 65535 (고정) | WINDOW_UPDATE로만 증가 (SETTINGS 영향 없음) |

SETTINGS로 `INITIAL_WINDOW_SIZE`를 바꾸면, **이미 열려 있는 스트림들의 윈도우도 그 차이만큼 일제히 조정된다.** 예컨대 65535에서 1048576으로 늘리면, 살아있는 모든 스트림의 윈도우에 (1048576 - 65535)만큼 더해진다. 이 때문에 윈도우가 음수가 될 수도 있는데(설정을 줄였을 때), 음수 윈도우는 합법이며 양수가 될 때까지 전송이 멈춘다.

### 2.2 WINDOW_UPDATE 프레임을 바이트로 해부하기

`WINDOW_UPDATE`(프레임 타입 `0x8`)는 흐름 제어 신용을 보충하는 프레임이다. 구조를 [[04 - HTTP2 깊이 보기 - 전송 계층]]의 공통 프레임 헤더와 함께 바이트로 뜯어보자.

HTTP/2 모든 프레임의 9바이트 헤더 형식:

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

WINDOW_UPDATE의 페이로드는 4바이트로, 최상위 1비트는 예약(R), 나머지 31비트가 "윈도우 증가량(Window Size Increment)"이다.

스트림 7번에 32768(0x8000)만큼 윈도우를 보충하는 WINDOW_UPDATE 프레임의 실제 바이트:

```text
00 00 04    →  Length = 4 (페이로드 4바이트)
08          →  Type = 0x08 (WINDOW_UPDATE)
00          →  Flags = 0 (WINDOW_UPDATE는 플래그 없음)
00 00 00 07 →  Stream ID = 7 (R 비트 = 0)
00 00 80 00 →  Window Size Increment = 0x00008000 = 32768
```

손으로 디코딩해보자:
- `00 00 04`: 빅엔디언 24비트 정수 → 4. 페이로드가 4바이트라는 뜻.
- `08`: 타입 8 = WINDOW_UPDATE.
- `00`: 플래그 없음.
- `00 00 00 07`: 최상위 R 비트(0) + 31비트 스트림 ID = 7. 이 7번 스트림의 윈도우를 늘린다.
- `00 00 80 00`: R 비트(0) + 31비트 증가량 = 0x8000 = 32768. 송신 측은 스트림 7의 윈도우 잔액에 32768을 더한다.

만약 같은 프레임에서 Stream ID가 `00 00 00 00`(=0)이라면, 이것은 **커넥션 레벨** 윈도우를 늘린다. gRPC 구현은 보통 두 종류의 WINDOW_UPDATE를 함께 보낸다: 소비된 스트림에 대한 것 하나, 그리고 커넥션 전체에 대한 것 하나.

증가량 0은 프로토콜 에러(`PROTOCOL_ERROR`)이고, 윈도우가 2³¹-1을 초과하게 만드는 증가는 `FLOW_CONTROL_ERROR`다.

### 2.3 DATA 프레임과 흐름 제어의 관계

흐름 제어가 차감하는 대상은 DATA 프레임의 **페이로드 길이**다. 정확히는 패딩(padding)을 포함한 페이로드 전체다. DATA 프레임 구조:

```text
+---------------+
|Pad Length? (8)|  ← PADDED 플래그 있을 때만
+---------------+-----------------------------------------------+
|                            Data (*)                           |
+---------------------------------------------------------------+
|                           Padding (*)                         |
+---------------------------------------------------------------+
```

흐름 제어 회계상 차감되는 바이트 = Data 길이 + Padding 길이 + (PADDED면 Pad Length 필드 1바이트). 즉 패딩도 윈도우를 소모한다. 패딩은 보안(메시지 길이 은닉)을 위해 쓰이지만 흐름 제어 신용을 잡아먹는다는 점을 기억하자. gRPC는 일반적으로 패딩을 쓰지 않는다.

여기서 gRPC 메시지와의 관계가 중요하다. [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 본 gRPC 길이-접두사 프레이밍(length-prefixed framing)을 떠올리자. gRPC 메시지 하나는 와이어 위에서:

```text
[1바이트 compressed-flag][4바이트 빅엔디언 메시지 길이][메시지 본문]
```

이 5바이트 접두사 + 본문 전체가 HTTP/2 DATA 프레임(들)의 페이로드로 들어간다. 큰 gRPC 메시지 하나는 여러 개의 DATA 프레임으로 쪼개진다(HTTP/2 프레임 크기 한계 `SETTINGS_MAX_FRAME_SIZE`, 기본 16384바이트). 그래서 4MB짜리 메시지는 최소 256개의 16KB DATA 프레임으로 나뉘고, 그 256개 프레임 전부가 윈도우 신용을 소모한다.

---

## 3. 작은 윈도우의 저주: BDP와 처리량 붕괴

### 3.1 BDP가 뭐길래

여기서 네트워크 성능의 가장 근본적인 공식 하나를 만나야 한다. **대역폭 지연 곱(Bandwidth-Delay Product, BDP)** 이다.

```text
BDP (바이트) = 대역폭 (바이트/초) × 왕복 지연(RTT, 초)
```

BDP는 직관적으로 "파이프 안에 동시에 떠 있을 수 있는 데이터의 양"이다. 송신 측이 데이터를 보내고, 그 데이터가 수신 측에 도착하고, 수신 측의 ACK(또는 WINDOW_UPDATE)가 다시 돌아오기까지 한 바퀴 도는 동안 파이프를 가득 채우려면 최소한 BDP만큼의 데이터가 "비행 중(in-flight)"이어야 한다.

흐름 제어 윈도우가 BDP보다 작으면 어떻게 될까? 송신 측은 윈도우를 다 쓰고 나서 WINDOW_UPDATE가 돌아올 때까지 멈춰서 기다린다. 그동안 파이프는 텅 빈다. 결과적으로 처리량은 다음으로 제한된다:

```text
최대 처리량 ≤ 윈도우 크기 / RTT
```

### 3.2 숫자로 보는 처리량 붕괴

기본 윈도우 65535바이트로 다양한 RTT에서 낼 수 있는 최대 처리량을 계산해보자:

| RTT | 윈도우 | 최대 처리량 = 65535/RTT | 환산 |
|-----|--------|--------------------------|------|
| 1ms (같은 DC) | 65535 B | 65.5 MB/s | ≈ 524 Mbps |
| 10ms (리전 내) | 65535 B | 6.55 MB/s | ≈ 52 Mbps |
| 50ms (대륙 간) | 65535 B | 1.31 MB/s | ≈ 10.5 Mbps |
| 100ms (지구 반대편) | 65535 B | 655 KB/s | ≈ 5.2 Mbps |

10Gbps(1250 MB/s) 광케이블을 깔아놓고도, 서울-버지니아 간 RTT 100ms 링크에서 단일 스트림은 **5.2Mbps**밖에 못 낸다. 이론 대역폭의 0.05%다. 파이프는 거대한데 윈도우라는 빨대로 빨아먹는 격이다.

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

이 문제를 "긴 뚱뚱한 파이프(long fat network, LFN)" 문제라고도 부른다. 고대역폭 + 고지연 조합에서 작은 윈도우가 처리량을 죽인다.

### 3.3 해결책 1: 윈도우를 크게 잡기

가장 단순한 처방은 초기 윈도우를 BDP 이상으로 키우는 것이다. RTT 100ms, 1Gbps를 가정하면 BDP = 125 MB/s × 0.1s = 12.5 MB. 따라서 윈도우를 12.5MB 이상으로 잡아야 파이프를 채운다.

언어별 설정 예시:

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

하지만 무작정 키우면 다른 비용이 생긴다. 윈도우가 크면 받는 쪽이 그만큼의 버퍼를 미리 확보해야 하고, 느린 소비자가 있을 때 메모리에 쌓이는 양도 늘어난다. 윈도우는 "처리량을 위한 버퍼 예산"이며 BDP에 맞춰 적절히 잡아야지 무한정 키우는 게 답이 아니다.

### 3.4 해결책 2: BDP 자동 추정과 동적 흐름 제어 (BDP estimation)

수동으로 윈도우를 BDP에 맞추는 건 RTT가 연결마다 다르고 시간에 따라 변하기 때문에 비현실적이다. 그래서 **일부** gRPC 구현은 **BDP를 실시간으로 추정해서 윈도우를 자동으로 늘리는 동적 흐름 제어(dynamic flow control / BDP estimation)** 를 내장하고 있다. gRPC-Go가 대표적이며 이를 "BDP estimator and dynamic flow control window"라 부른다. C 코어를 공유하는 구현들(C++, Python, Ruby, C#, PHP 등)도 코어에 들어 있는 BDP estimator의 혜택을 받는다.

**중요한 예외 — gRPC-Java:** 순수 Java(Netty 기반)인 gRPC-Java는 BDP 자동 추정을 하지 **않는다.** 대신 **고정된 흐름 제어 윈도우**를 쓰며, 그 기본값은 HTTP/2 프로토콜 기본인 65535가 아니라 gRPC-Java가 연결 시 SETTINGS로 올려 광고하는 **1MiB(1048576)** 다. 따라서 BDP가 큰 고지연·고대역폭 링크에서 Java는 `flowControlWindow()`로 윈도우를 직접 키워야 하며 자동 튜닝에 기댈 수 없다. 뒤에 나오는 "BDP를 모르겠으면 손대지 말고 자동 튜닝에 맡겨라"는 조언은 gRPC-Go·C 코어 구현에 해당하고, Java는 오히려 명시적 튜닝이 필요할 수 있다는 점이 구현 간 결정적 차이다.

작동 원리(개념):

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

이 자동 튜닝 덕분에 대부분의 경우 윈도우를 손댈 필요가 없다. gRPC-Go는 동적 흐름 제어가 켜져 있으면(기본), `InitialWindowSize`를 명시적으로 지정하지 않는 한 BDP estimator가 윈도우를 관리한다. **주의: `InitialWindowSize`를 직접 설정하면 동적 흐름 제어(BDP 추정 기반 자동 확장)가 비활성화되고 그 고정값으로 못박힌다.** 그래서 BDP를 잘 모르겠으면 차라리 손대지 않고 자동 튜닝에 맡기는 편이 안전하다. 명시 설정은 "내 환경의 BDP를 정확히 알고, 자동 추정보다 잘할 수 있다"는 확신이 있을 때만 하라.

---

## 4. 백프레셔: 전송 계층의 압력이 애플리케이션까지 닿는 길

지금까지는 와이어 레벨 흐름 제어였다. 이제 진짜 중요한 질문. **이 압력이 어떻게 내 애플리케이션 코드까지 전달되는가?** 흐름 제어가 와이어에서만 작동하고 애플리케이션은 그걸 모른다면, 빠른 생산자는 라이브러리 내부 송신 큐에 메시지를 무한정 쌓다가 메모리를 터뜨릴 것이다. 백프레셔의 핵심은 **흐름 제어의 압력을 애플리케이션 API로 노출하는 것**이다.

### 4.1 문제의 본질: 빠른 생산자, 느린 소비자

서버 스트리밍을 생각해보자([[05 - 통신의 4가지 방식 - Unary와 Streaming]]). 서버가 데이터베이스에서 100만 행을 읽어 클라이언트로 스트리밍한다. 서버는 디스크/메모리에서 행을 매우 빠르게 뽑아낼 수 있다(초당 수십만 행). 그런데 클라이언트는 모바일 기기이고 느린 네트워크에 있다.

만약 서버가 백프레셔를 무시하고 `responseObserver.onNext(row)`를 100만 번 호출하면 어떻게 될까?

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

흐름 제어는 와이어로 나가는 걸 막았다. 하지만 `onNext()`는 큐에 넣기만 하고 즉시 반환되므로, 애플리케이션 루프는 멈추지 않고 계속 큐에 쌓는다. 와이어가 막혀도 메모리 큐는 무한대로 자란다. 흐름 제어가 메모리를 지켜주지 못했다.

해결의 열쇠: **애플리케이션 루프가 "와이어가 막혔으면 생산도 멈춰야" 한다.** 이걸 어떻게 알릴까? 언어마다 메커니즘이 다르다.

### 4.2 Go: 블로킹 Send/Recv — 가장 자연스러운 백프레셔

Go의 gRPC는 백프레셔를 가장 우아하게 다룬다. `stream.Send()`가 **블로킹**이기 때문이다. 흐름 제어 윈도우가 닫혀 있으면 `Send()`는 그냥 거기서 멈춰서 기다린다(고루틴이 블록된다). 윈도우가 열리면 반환된다.

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

핵심은 `Send()`가 블록되면 `for rows.Next()` 루프 자체가 멈춘다는 것이다. 루프가 멈추면 DB에서 다음 행을 안 읽는다. 즉, **느린 소비자의 압력이 블로킹 Send → 멈춘 루프 → 안 읽는 DB 커서까지 거슬러 올라가** 생산 자체가 느려진다. 생산 속도가 소비 속도에 자동으로 맞춰진다. 이것이 백프레셔의 이상적 형태다.

Go에서 주의할 점: `Send()`가 블록되는 동안 그 고루틴은 점유된다. 동시 스트림이 수만 개라면 고루틴이 수만 개 블록될 수 있는데, Go 고루틴은 가볍기 때문에(스택 ~2KB) 대개 문제없다. 하지만 데드라인 없이 영원히 블록되지 않도록 `stream.Context()`의 취소/데드라인([[10 - Deadline 취소 타임아웃]])을 항상 존중해야 한다.

`RecvMsg`/`Recv`도 마찬가지로 흐름 제어와 연동된다. 받는 쪽이 `Recv()`를 호출해야 라이브러리가 데이터를 소비한 것으로 보고 WINDOW_UPDATE를 보낸다. 받는 쪽이 `Recv()`를 게을리 호출하면 윈도우가 안 열리고, 그게 송신 측으로 전파되는 백프레셔다.

### 4.3 Java: isReady()와 onReadyHandler — 콜백 기반 논블로킹 백프레셔

Java gRPC는 기본 API가 `StreamObserver`라는 **콜백 기반 논블로킹** 모델이다([[05 - 통신의 4가지 방식 - Unary와 Streaming]]). `onNext()`는 블로킹하지 않고 즉시 반환한다. 그래서 Go처럼 "그냥 루프 돌리면 알아서 백프레셔" 가 안 된다. 대신 명시적인 신호를 봐야 한다.

핵심 도구는 `CallStreamObserver`(서버에선 `ServerCallStreamObserver`, 클라이언트에선 `ClientCallStreamObserver`)가 제공하는 두 가지:

- `boolean isReady()`: 지금 `onNext()`를 호출해도 송신 버퍼가 넘치지 않는지 여부. 흐름 제어 윈도우와 내부 버퍼 상태를 반영한다. `false`면 "지금은 보내지 마".
- `setOnReadyHandler(Runnable)`: `isReady()`가 `false` → `true`로 전이될 때 호출되는 콜백. "이제 다시 보내도 돼" 신호.

올바른 패턴은 다음과 같다. 단순히 `isReady()`가 false인 동안 바쁜 대기(busy-wait)하면 안 되고 콜백 기반으로 생산을 멈췄다 재개해야 한다.

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

이 패턴의 핵심을 다시 보자:

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

만약 `isReady()`를 무시하고 무작정 `onNext()`를 호출하면? Java gRPC는 메시지를 버리지 않고 내부 버퍼에 무한정 쌓는다(흐름 제어로 와이어는 막혔으므로). 결국 힙이 터진다. 그래서 **빠른 생산자-느린 소비자 시나리오에서 Java는 반드시 isReady()를 존중해야 한다.** 이것이 Go와 Java의 가장 큰 운영상 차이다.

### 4.4 받는 쪽 백프레셔: request(n)으로 수요를 표현하기

지금까지는 "보내는 쪽이 멈추는" 백프레셔였다. 반대로 **받는 쪽이 "나는 한 번에 N개만 처리할 수 있어"라고 수요를 제어**하는 것도 중요하다. 이것이 reactive streams의 `request(n)` 모델이다.

Java gRPC의 콜백 모델은 기본적으로 메시지를 자동으로 한 개씩 요청한다(`onNext` 콜백이 끝나면 자동으로 다음 1개 request). 하지만 자동 흐름 제어를 끄고 수동으로 제어할 수 있다.

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

서버 측에서도 `ServerCallStreamObserver.disableAutoRequest()` + `request(n)`으로 같은 제어가 가능하다. 핵심 메커니즘: **`request(n)`은 "애플리케이션이 n개를 더 소비할 의향이 있다"는 수요 신호이고, gRPC 런타임은 이 수요를 흐름 제어 윈도우(WINDOW_UPDATE)로 번역한다.** 즉, 애플리케이션의 `request(n)`이 와이어 레벨 신용으로 변환되어 송신 측을 조절한다. 애플리케이션 백프레셔와 전송 백프레셔가 하나의 사슬로 연결되는 지점이다.

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

Python의 동기(synchronous) gRPC API에서 서버 스트리밍은 제너레이터(generator)를 반환하는데, gRPC 런타임이 제너레이터에서 값을 당겨갈 때 흐름 제어와 연동된다. 즉 와이어가 막히면 런타임이 제너레이터의 다음 `yield`를 안 당기므로 생산이 자연스럽게 멈춘다 — Go와 비슷한 풀(pull) 기반 백프레셔다.

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

`grpc.aio`(asyncio) API에서는 `await stream.write(msg)`가 코루틴이며, 흐름 제어로 막히면 `await`가 대기한다 — Go의 블로킹 Send와 같은 직관이다.

---

## 5. 메시지 크기 한계와 청크 분할

### 5.1 기본 4MB 수신 한계와 RESOURCE_EXHAUSTED

gRPC는 메모리 폭발을 막기 위해 메시지 크기에 **기본 한계**를 둔다.

- **수신(inbound) 메시지 기본 한계: 4MB (4 × 1024 × 1024 = 4194304바이트)**
- 송신(outbound) 메시지 기본 한계: 대부분 구현에서 무제한(`MAX_INT`)이지만 설정 가능.

이 한계는 **메시지 하나(단일 protobuf 메시지)의 직렬화 크기**에 적용된다. 스트림 전체 크기가 아니다. 스트림으로 4MB짜리 메시지를 1000개 보내는 건 괜찮지만 5MB짜리 메시지 하나는 막힌다.

한계를 초과하면 RPC는 **`RESOURCE_EXHAUSTED`(상태 코드 8)** 로 실패한다([[09 - 에러 모델 - 상태 코드와 Rich Error]]). 에러 메시지는 보통 "Received message larger than max (5242880 vs 4194304)" 같은 형태다.

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

### 5.2 왜 큰 메시지는 나쁜가, 그리고 청크 스트리밍

한계를 그냥 키우면 될 것 같지만, 큰 메시지에는 본질적 문제가 있다:

1. **전부-아니면-전무(all-or-nothing) 역직렬화**: protobuf 메시지는 끝까지 다 받아야 파싱할 수 있다. 100MB 메시지는 100MB가 다 도착할 때까지 받는 쪽 메모리에 통째로 버퍼링된다. 메모리 스파이크가 크고 중간에 끊기면 전부 버린다.
2. **흐름 제어 입자도(granularity)**: 큰 메시지 하나는 흐름 제어상 잘게 쪼개지긴 하지만, 애플리케이션은 메시지 하나를 원자적으로 처리하므로 진행 상황을 알 수 없다.
3. **재시도/복원력**: 큰 메시지는 재시도([[13 - 안정성 - Retry Health Check Keepalive]]) 비용이 크다. 실패하면 전체를 다시 보내야 한다.

정석 해법은 **큰 페이로드를 청크(chunk)로 쪼개 서버/클라이언트 스트리밍으로 보내는 것**이다.

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

청크 크기는 흔히 **16KB ~ 64KB**로 잡는다. 너무 작으면(예: 1KB) 메시지당 프레이밍/HPACK/흐름제어 회계 오버헤드가 상대적으로 커지고 너무 크면(예: 4MB) 위에서 말한 큰 메시지 문제가 생긴다. 16~64KB는 HTTP/2 기본 프레임 크기(16KB)와도 잘 맞고 흐름 제어 입자도도 적절하다.

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

`buf`를 매 청크마다 재사용한다는 점도 눈여겨보자(5.4의 직렬화 비용 절감과 연결된다). 다만 `bytes` 필드는 마샬링 시 복사되므로, `buf[:n]`을 재사용해도 protobuf가 내부적으로 복사본을 만든다는 점을 알아둬야 한다.

---

## 6. 압축: 대역폭을 CPU와 맞바꾸기

### 6.1 per-message 압축과 협상

gRPC는 **메시지 단위(per-message)** 압축을 지원한다. 전체 스트림을 압축하는 게 아니라 각 gRPC 메시지를 개별적으로 압축한다. 이는 [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]]에서 본 길이-접두사 프레이밍의 1바이트 **compressed-flag**와 직접 연결된다.

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

compressed-flag가 `0x01`이면, 그 메시지 본문은 요청/응답 헤더의 `grpc-encoding`이 지정한 알고리즘(예: `gzip`)으로 압축돼 있다. `0x00`이면 압축 안 됨. **이 플래그가 메시지마다 있으므로 같은 스트림 안에서도 어떤 메시지는 압축하고 어떤 메시지는 안 할 수 있다.** 예컨대 이미 압축된 JPEG 청크는 압축 안 하고(역효과), 텍스트 청크는 압축하는 식의 메시지별 판단이 가능하다.

협상은 두 메타데이터 헤더([[08 - 메타데이터와 인터셉터]])로 이뤄진다:

| 헤더 | 방향 | 의미 |
|------|------|------|
| `grpc-encoding` | 양방향 | "내가 보내는 이 메시지(들)는 이 방식으로 압축했다" |
| `grpc-accept-encoding` | 양방향 | "나는 이 알고리즘들을 풀 수 있다" |

표준 알고리즘: `identity`(무압축), `gzip`, `deflate`. 일부 구현은 `snappy` 등을 플러그인으로 추가할 수 있다.

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

압축은 공짜가 아니다. **대역폭을 줄이는 대신 CPU를 쓴다.** 이 트레이드오프를 정확히 이해해야 한다.

| 상황 | 압축 권장? | 이유 |
|------|-----------|------|
| 큰 텍스트/JSON/반복적 protobuf | 켜기 | 압축률 높음(70~90%), 대역폭 절감 큼 |
| 이미 압축된 데이터(JPEG, mp4, gzip) | 끄기 | 거의 안 줄고 CPU만 낭비, 때론 더 커짐 |
| 작은 메시지(수백 바이트 이하) | 보통 끄기 | gzip 헤더/사전 오버헤드가 절감보다 큼 (역효과) |
| 같은 DC 내 고대역폭 링크 | 신중 | 대역폭이 병목 아니면 CPU만 더 씀 |
| 대륙간/저대역폭 링크 | 켜기 | 대역폭이 진짜 병목, 압축이 RTT당 처리량 개선 |

**작은 메시지에 대한 역효과를 구체적으로 보자.** gzip에는 최소한의 헤더(10바이트)와 트레일러(8바이트, CRC32 + 길이)가 붙는다. 100바이트짜리 메시지를 gzip하면:

```text
원본 100바이트 → gzip → 18바이트(헤더+트레일러) + 압축된 본문(~90바이트?)
                       = 약 108바이트  ← 오히려 커짐!
```

작고 엔트로피 높은 메시지는 gzip 후 오히려 커진다. 그래서 많은 구현은 압축이 손해면 무압축(compressed-flag=0)으로 보내는 최적화를 한다. 하지만 의존하지 말고, **작은 메시지가 대다수인 서비스라면 압축을 끄는 게 보통 더 빠르다.**

**CPU 비용의 실체**: gzip 압축은 메시지당 수십 마이크로초~수 밀리초의 CPU를 쓴다. 초당 수만 RPC를 처리하는 서버에서 모든 메시지를 gzip하면 CPU 코어 하나를 통째로 압축에만 쓸 수도 있다. 측정 없이 "압축 켜면 빨라지겠지"는 위험한 가정이다. 반드시 [[14 - 관찰성과 디버깅 - Reflection grpcurl]]의 도구와 이 장 §10의 벤치마킹으로 검증하라.

---

## 7. 동시성과 연결: MAX_CONCURRENT_STREAMS의 벽

### 7.1 단일 연결이 병목이 되는 지점

[[07 - 채널 스텁 커넥션 생명주기]]에서 봤듯, gRPC 채널 하나는 보통 **하나의 HTTP/2 연결(또는 소수의 연결)** 위에서 동작한다. 하나의 연결 위로 여러 RPC가 멀티플렉싱되는 게 HTTP/2의 강점이지만, 여기에는 한계가 있다.

`SETTINGS_MAX_CONCURRENT_STREAMS`(설정 식별자 `0x3`)는 **한 연결 위에서 동시에 열 수 있는 스트림(=진행 중인 RPC)의 최대 개수**다.

- RFC 7540/9113은 이 설정의 기본값을 무제한으로 두지만, 실무에서 서버는 보안/리소스 보호를 위해 **흔히 100~250** 정도로 제한한다. 과거 gRPC-Go 서버는 사실상 무제한(`math.MaxUint32`)이 기본이었으나, HTTP/2 Rapid Reset 취약점(CVE-2023-44487, 2023) 이후 **기본값이 100으로 낮아졌다.** 단일 연결로 수많은 스트림을 즉시 열었다 RST_STREAM으로 닫는 공격을 막기 위한 조치다. Envoy·nginx 같은 프록시도 100 근처가 흔하다.
- 클라이언트가 이 한계를 초과해 새 RPC를 시작하려 하면, gRPC는 **새 스트림을 거부하지 않고 큐잉**한다. 진행 중 스트림이 끝나 자리가 나면 대기 중인 RPC가 시작된다.

```text
MAX_CONCURRENT_STREAMS = 100 인 단일 연결:

  진행 중 RPC: [1][2][3]...[100]   ← 꽉 참
  대기 큐:     [101][102][103]...  ← 자리 날 때까지 대기
                  │
                  └─► 이 대기가 지연(latency)으로 나타남
                      "왜 p99가 튀지?" 의 흔한 원인
```

문제: 만약 RPC들이 오래 걸리거나(롱 스트리밍), 동시에 수천 개를 보내야 한다면, 단일 연결의 100개 한계가 병목이 된다. 101번째부터는 줄을 서야 하고 처리량이 "100 / 평균 RPC 시간"으로 묶인다.

### 7.2 단일 연결 HOL 블로킹

또 하나의 미묘한 문제. 멀티플렉싱은 HTTP/2 레벨의 HOL 블로킹은 해결했지만(스트림별 흐름 제어), **TCP 레벨의 HOL 블로킹은 여전히 남는다.** 단일 TCP 연결에서 패킷 하나가 손실되면, TCP는 그 패킷이 재전송될 때까지 그 뒤의 모든 바이트를 애플리케이션에 전달하지 못한다. 그 연결 위 모든 스트림이 같이 멈춘다.

```text
TCP HOL 블로킹 (단일 연결):

  Stream 1 데이터 ─┐
  Stream 3 데이터 ─┼─► [TCP 세그먼트 #5 손실!] ─► 수신
  Stream 5 데이터 ─┘         │
                       TCP가 #5 재전송 기다리는 동안
                       #6,#7... (다른 스트림 데이터 포함) 다 대기
                       → 무손실 스트림 3,5도 같이 멈춤
```

이것은 HTTP/2의 근본적 한계이고 HTTP/3(QUIC)가 UDP 기반 독립 스트림으로 해결한 바로 그 문제다. gRPC에서는 **연결을 여러 개로 분산(채널 풀)** 하면 한 연결의 패킷 손실이 다른 연결의 스트림에 영향을 주지 않으므로 완화된다.

### 7.3 채널 풀: 연결을 늘려 병목 깨기

해법은 **여러 개의 연결(채널)을 만들어 RPC를 분산**하는 것이다. 이를 채널 풀링(channel pooling) 또는 멀티플 서브채널이라 한다.

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

언제 채널 풀이 필요한가:
- 단일 연결의 `MAX_CONCURRENT_STREAMS`를 동시 RPC 수가 초과할 때.
- 초고처리량(연결당 처리량이 NIC/CPU 한계에 닿을 때) — 한 연결은 한 CPU 코어에서 처리되는 경향이 있어, 여러 연결이 여러 코어를 활용.
- TCP HOL 블로킹의 영향을 분산하고 싶을 때.

**주의**: 연결을 늘리면 로드밸런싱과 상호작용한다. [[12 - 이름 해석과 로드밸런싱]]에서 본 것처럼, L4 로드밸런서 뒤에서는 연결 하나가 백엔드 하나에 핀(pin)되므로, 단일 연결은 단일 백엔드만 때린다. 여러 연결을 만들거나 클라이언트 측 로드밸런싱(`round_robin`)을 쓰면 부하가 여러 백엔드로 퍼진다. 즉 채널 풀은 처리량뿐 아니라 **부하 분산**과도 직결된다.

연결당 처리량 vs 연결 수의 트레이드오프:

| 전략 | 장점 | 단점 |
|------|------|------|
| 단일 연결 | 적은 소켓/메모리, 간단 | MAX_CONCURRENT_STREAMS 병목, TCP HOL, 단일 백엔드 |
| 소수 연결 풀(2~8) | 병목 완화, 멀티코어 활용, 부하 분산 | 약간의 리소스 증가 |
| 과도한 연결(수십~수백) | — | 소켓/메모리 낭비, keepalive PING 오버헤드 폭증, 서버 부담 |

**황금률**: 무작정 늘리지 말고 단일 연결로 시작해 벤치마크에서 병목이 확인되면 점진적으로 늘려라.

---

## 8. 직렬화 비용과 최적화

### 8.1 marshal/unmarshal은 공짜가 아니다

gRPC RPC 한 번에는 보이지 않는 CPU 비용이 숨어 있다: **protobuf 직렬화(marshal)와 역직렬화(unmarshal).** 메시지가 크거나 필드가 많거나 RPC가 초당 수만 번이면, 이 비용이 전체 CPU의 상당 부분을 차지한다.

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

**Go: 메시지/버퍼 재사용과 proto 옵션**
- gRPC-Go는 메시지를 풀(pool)링하는 옵션과 `mem` 패키지 기반의 버퍼 재사용을 점진적으로 도입해왔다. 큰 `bytes` 필드를 다룰 때 불필요한 복사를 줄이는 게 핵심.
- 직접 할 수 있는 것: 요청/응답 객체를 루프에서 재사용(`proto.Reset(msg)` 후 재사용)해 할당을 줄인다. 다만 동시성 안전성에 주의.

**Java: arena 없음, 대신 객체 풀과 zero-copy**
- Java protobuf는 불변(immutable) 메시지라 arena 할당이 없다. 대신 `CodedInputStream`/`CodedOutputStream`을 직접 다루거나, `ByteString.unsafeWrap`으로 복사를 피할 수 있다.
- gRPC-Java는 Netty의 `ByteBuf`를 활용해 일부 zero-copy 경로를 제공. 큰 `bytes`는 가능하면 `ByteString` 그대로 다뤄 byte[] 복사를 피한다.

**C++: Arena 할당**
- protobuf C++는 **Arena**를 지원한다. 한 RPC에서 만드는 모든 메시지 객체를 하나의 큰 메모리 블록(arena)에 할당하고, RPC 끝나면 arena를 통째로 해제한다. 객체별 `new`/`delete`를 없애 할당/해제 비용과 메모리 단편화를 크게 줄인다.

```cpp
// C++ Arena 예시
google::protobuf::Arena arena;
// 메시지를 arena에 할당 — 개별 delete 불필요
MyMessage* msg = google::protobuf::Arena::CreateMessage<MyMessage>(&arena);
msg->set_field(...);
// arena가 스코프를 벗어나면 모든 객체 일괄 해제 (개별 소멸자 호출 없음)
```

### 8.3 zero-copy bytes와 복사 줄이기

`bytes` 필드(또는 큰 문자열)는 직렬화/역직렬화에서 복사 비용이 가장 크다. 핵심 전략:

- **불필요한 변환 피하기**: `bytes`로 받은 데이터를 굳이 `string`으로 바꾸거나, 다시 다른 버퍼로 복사하지 말 것.
- **참조 전달**: 가능하면 역직렬화된 버퍼를 그대로 참조해 사용(Java `ByteString`, Go의 `[]byte` 슬라이스 공유).
- **proto 설계로 줄이기**: 거대한 단일 `bytes`보다 청크 스트리밍(5.2)이 메모리 피크와 복사를 줄인다.

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

[[02 - Protocol Buffers 1 - 문법과 타입 시스템]]에서 본 것처럼 `bytes`와 `string`의 차이는 단순한 타입 선택이 아니라 성능 결정이다. 바이너리 데이터에 `string`을 쓰면 UTF-8 검증 오버헤드까지 더해진다.

---

## 9. 작은 RPC 다발의 오버헤드: HPACK, keepalive, 배칭

### 9.1 헤더 오버헤드와 HPACK

RPC 하나에는 페이로드뿐 아니라 **헤더**가 따라붙는다([[08 - 메타데이터와 인터셉터]]). gRPC 요청 헤더에는 `:method: POST`, `:path: /pkg.Service/Method`, `content-type: application/grpc`, `te: trailers`, `grpc-timeout`, `authorization` 등 십수 개의 헤더가 들어간다. 이걸 매 RPC마다 평문으로 보내면 작은 RPC(페이로드 수십 바이트)에서 헤더가 페이로드보다 훨씬 커진다.

[[04 - HTTP2 깊이 보기 - 전송 계층]]에서 다룬 **HPACK**이 이 문제를 완화한다. HPACK은:
- **정적 테이블**: 자주 쓰는 헤더(`:method: POST` 등)에 번호를 매겨 1~2바이트로 표현.
- **동적 테이블**: 연결 동안 반복되는 헤더(`:path`, `authorization`)를 한 번 보내면 인덱싱해서, 이후엔 인덱스 번호만 전송.

```text
HPACK 효과 (같은 연결에서 두 번째 RPC부터):

  첫 RPC:  :path: /order.v1.OrderService/GetOrder  (전체 전송, 동적 테이블에 등록)
  둘째 RPC: [인덱스 62]                              (1~2바이트로 축약!)

  → 같은 메서드를 반복 호출하면 헤더 오버헤드가 거의 사라진다
```

이 때문에 **같은 연결로 같은 메서드를 반복 호출하는 패턴이 HPACK에 유리**하다. 매번 새 연결을 맺으면 동적 테이블이 초기화돼 HPACK 이득을 못 본다 — 또 하나의 연결 재사용([[07 - 채널 스텁 커넥션 생명주기]]) 이유다.

### 9.2 keepalive PING 오버헤드

[[13 - 안정성 - Retry Health Check Keepalive]]에서 다룰 keepalive는 죽은 연결을 빨리 감지하기 위해 주기적으로 PING 프레임을 보낸다. 하지만 과도하면 오버헤드다:

- PING이 너무 잦으면(예: 매 1초) 네트워크/CPU 낭비, 서버가 `ENHANCE_YOUR_CALM`(too_many_pings)으로 연결을 끊을 수 있다.
- 연결이 많을수록(채널 풀 과다) PING 총량이 곱해진다. 연결 100개 × 매 10초 PING = 초당 10 PING.

권장: keepalive 주기는 보통 분 단위(예: `time: 30s~5m`), idle일 때는 PING 안 보내도록 `permitWithoutStream=false` 고려. 자세한 튜닝은 [[13 - 안정성 - Retry Health Check Keepalive]] 참조.

### 9.3 배칭: 작은 RPC를 묶기

초당 수만 개의 작은 RPC는 RPC당 고정 비용(헤더, 프레이밍, 흐름제어 회계, 컨텍스트 전환)이 누적돼 비효율적이다. 해법은 **배칭(batching)**: 여러 논리적 요청을 하나의 RPC로 묶는다.

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

배칭은 RPC 고정 비용을 N분의 1로 줄이지만 지연(첫 결과를 받기까지 전체를 기다림)과 부분 실패 처리(일부만 성공 시)를 신중히 설계해야 한다. 스트리밍은 배칭의 대안으로, 묶지 않고도 연결을 재사용하며 점진적으로 결과를 흘려보낸다.

---

## 10. 벤치마킹: 측정 없이는 최적화도 없다

### 10.1 ghz로 부하 걸기

`ghz`는 gRPC 전용 부하 테스트 도구다. HTTP의 `wrk`/`hey`에 해당한다.

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

`--connections`(연결 수)와 `--concurrency`(동시 RPC 수)를 분리해서 거는 게 핵심이다. 7장에서 본 MAX_CONCURRENT_STREAMS / 채널 풀의 효과를 이 두 노브로 실험할 수 있다. 예컨대 `--connections 1 --concurrency 500`으로 단일 연결 병목을 재현하고, `--connections 10 --concurrency 500`으로 풀의 효과를 본다.

리플렉션([[14 - 관찰성과 디버깅 - Reflection grpcurl]])이 켜져 있으면 `--proto` 없이도 동작한다.

### 10.2 무엇을 측정할 것인가

ghz 출력에서 봐야 할 지표:

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
| **Throughput (RPS)** | 시스템의 처리 능력. 목표 부하를 견디는지. |
| **p50 (중앙값)** | 전형적 사용자 경험. |
| **p99 / p99.9 (꼬리 지연)** | 진짜 중요. 평균은 거짓말한다. p99가 튀면 GC, 큐잉, HOL 블로킹을 의심. |
| **연결 수 / 동시성** | 병목 위치 진단(단일 연결 vs 다중). |
| **에러 분포** | RESOURCE_EXHAUSTED(한계), DEADLINE_EXCEEDED(과부하), UNAVAILABLE(연결) 비율. |

**평균(average)에 속지 마라.** 평균 5ms인데 p99가 41ms라는 건, 1%의 요청이 8배 느리다는 뜻이고 초당 1만 RPS면 그게 초당 100건이다. 사용자 체감과 SLO는 꼬리 지연이 좌우한다.

### 10.3 흔한 성능 함정 3종 세트

**함정 1: 단일 연결 HOL — "동시성을 올려도 처리량이 안 오른다"**

증상: `--concurrency`를 100→500으로 올려도 RPS가 그대로다.
원인: `--connections 1`이라 단일 연결의 MAX_CONCURRENT_STREAMS나 단일 코어 처리 한계에 묶임.
진단: `--connections`를 늘려보고 RPS가 오르면 단일 연결 병목 확정.
처방: 채널 풀(7.3) 또는 클라이언트 측 로드밸런싱([[12 - 이름 해석과 로드밸런싱]]).

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

**함정 2: GC 압박 — "처리량은 OK인데 p99가 주기적으로 튄다"**

증상: p99 지연이 톱니 모양으로 주기적으로 치솟음(Java/Go).
원인: 매 RPC마다 메시지 객체를 새로 할당 → GC가 자주 돌며 stop-the-world.
진단: GC 로그(`-Xlog:gc` / `GODEBUG=gctrace=1`)와 p99 스파이크의 시간 상관관계 확인.
처방: 8장의 객체 재사용/arena, 메시지 크기 축소, 압축으로 할당량 감소, GC 튜닝.

**함정 3: 동기 블로킹 스레드 고갈 — "동시 요청이 많아지면 latency가 폭증한다"**

증상: 동시 RPC 수가 스레드풀 크기를 넘으면 지연이 계단식으로 증가.
원인: Java/Python 동기 서버가 RPC당 스레드를 점유. 스레드풀(예: 200)이 꽉 차면 나머지는 큐에서 대기. 핸들러가 블로킹 I/O(DB, 외부 호출)를 하면 스레드가 오래 묶인다.
진단: 스레드 덤프에서 대부분이 블로킹 상태인지, 큐 길이가 늘어나는지 확인.
처방: 스레드풀 크기 조정, 비동기/논블로킹 핸들러, 또는 백프레셔로 유입 제한. 무작정 스레드를 늘리면 컨텍스트 전환 비용으로 역효과.

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

지금까지의 조각을 하나로 모아보자. 서버 스트리밍에서 백프레셔를 지키고, 메시지 크기/압축/keepalive/윈도우를 튜닝한 양쪽 코드.

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

### 11.2 Go 서버: 블로킹 Send로 자연 백프레셔 + 튜닝 옵션

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

### 11.3 Go 클라이언트: Recv 속도로 백프레셔 전파

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

이 Java 서버와 11.2의 Go 서버는 **같은 백프레셔 목표를 다른 메커니즘으로** 달성한다. Go는 블로킹 Send가 루프를 멈춰주고, Java는 isReady()/onReadyHandler가 명시적으로 생산을 멈췄다 재개한다. 둘 다 핵심은 **"와이어가 막히면 생산도 멈춘다"** 는 동일한 원칙이다.

---

## 12. 성능 튜닝 의사결정 트리

지금까지 본 노브들을 언제 돌릴지 한눈에 정리한다.

```text
"gRPC가 느리다/처리량이 안 난다" 진단 순서:

1. 먼저 측정했는가? ─── 아니오 → ghz로 throughput/p50/p99/에러 측정부터
   │ 예
   ▼
2. 에러가 있는가?
   ├─ RESOURCE_EXHAUSTED → 메시지 크기 한계 / 동시성 한계 (5장, 7장)
   ├─ DEADLINE_EXCEEDED  → 과부하 또는 데드라인 짧음 ([[10 - Deadline 취소 타임아웃]])
   └─ UNAVAILABLE        → 연결/LB 문제 ([[12 - 이름 해석과 로드밸런싱]], [[13 - 안정성 - Retry Health Check Keepalive]])
   │ 에러 없음
   ▼
3. 동시성 올려도 RPS 정체? ─── 예 → 단일 연결 병목 → 채널 풀/LB (7장)
   │ 아니오
   ▼
4. p99가 주기적으로 튐? ─── 예 → GC 압박 → 객체 재사용/arena, 할당 감소 (8장)
   │ 아니오
   ▼
5. 대륙간/저대역폭 + 큰 메시지? ─── 예 → 압축 켜기 + 윈도우 BDP 맞춤 (3장, 6장)
   │ 아니오
   ▼
6. 빠른 생산자-느린 소비자 메모리 증가? ─── 예 → 백프레셔 점검 (4장)
                                              isReady()/blocking Send/request(n)
   │ 아니오
   ▼
7. 작은 RPC 다발? ─── 예 → 배칭/스트리밍 + 연결 재사용(HPACK 이득) (9장)
```

---

## 13. 자주 하는 오해 바로잡기

**오해 1: "흐름 제어를 켜면 느려진다."**
아니다. 흐름 제어는 항상 켜져 있고(HTTP/2 필수), 끌 수 없다. 흐름 제어는 처리량을 줄이는 게 아니라 메모리 안정성을 보장하면서 안전한 최대 처리량을 찾는다. 윈도우가 BDP에 맞으면 흐름 제어가 있어도 풀 처리량이 난다.

**오해 2: "압축은 항상 좋다."**
아니다. 작은 메시지, 이미 압축된 데이터, 고대역폭 내부 링크에서는 CPU만 낭비하고 때론 느려진다(6.3).

**오해 3: "연결을 많이 만들수록 빠르다."**
아니다. 적정 수(2~8)를 넘으면 소켓/메모리/keepalive PING 오버헤드가 늘고 서버 부담이 커진다(7.3). 측정으로 최적점을 찾아라.

**오해 4: "윈도우를 무한정 키우면 좋다."**
아니다. 윈도우는 받는 쪽이 미리 확보하는 버퍼 예산이다. BDP를 넘는 윈도우는 메모리만 더 쓰고 처리량 이득이 없으며, 느린 소비자 시 더 많은 데이터를 메모리에 쌓는다(3.3).

**오해 5: "Java에서 onNext()를 그냥 부르면 알아서 백프레셔가 걸린다."**
아니다. Java 콜백 모델은 isReady()를 무시하면 내부 버퍼에 무한 적재한다(4.3). 명시적으로 isReady()/onReadyHandler를 써야 한다. 이것이 Go(블로킹 Send로 공짜 백프레셔)와의 결정적 차이다.

---

## 핵심 요약

- gRPC 성능의 본질은 "빨리 보내기"가 아니라 "받는 쪽이 소화할 만큼만 흘려보내기"다. HTTP/2의 신용기반(credit-based) 흐름 제어가 이 균형을 와이어 위에서 강제한다.
- 흐름 제어는 **커넥션 레벨(Stream ID 0)** 과 **스트림 레벨** 두 윈도우로 작동한다. DATA 프레임을 보내려면 둘 다 신용이 있어야 하고, 받는 쪽이 `WINDOW_UPDATE`로 신용을 보충한다. 제어 프레임(HEADERS, WINDOW_UPDATE, PING)은 흐름 제어 대상이 아니다(데드락 방지).
- 초기 윈도우 기본값은 **65535바이트**(`SETTINGS_INITIAL_WINDOW_SIZE`, 스트림 레벨에만 적용; 커넥션 윈도우 초기값은 항상 65535 고정). 작은 윈도우는 BDP 큰 링크에서 처리량을 `윈도우/RTT`로 묶어 처리량을 붕괴시킨다.
- **BDP = 대역폭 × RTT.** 윈도우가 BDP보다 작으면 파이프가 빈다. gRPC-Go와 C 코어 기반 구현은 PING으로 RTT/BDP를 추정해 윈도우를 자동 확장하는 **동적 흐름 제어**를 내장한다(이때 `InitialWindowSize`를 명시 설정하면 자동 튜닝이 꺼지므로 BDP를 모르면 손대지 마라). 반면 **gRPC-Java는 BDP 자동 추정이 없고 고정 윈도우(기본 1MiB)** 를 쓰므로 고BDP 링크에선 `flowControlWindow()`로 직접 키워야 한다.
- **백프레셔는 흐름 제어의 압력을 애플리케이션까지 전달하는 것.** Go는 블로킹 `Send`/`Recv`로 자연스럽게, Java는 `isReady()`/`onReadyHandler`(논블로킹 콜백)로 명시적으로, 받는 쪽은 `request(n)`(reactive)으로 수요를 제어한다. `request(n)`은 결국 WINDOW_UPDATE로 번역돼 와이어 백프레셔와 한 사슬로 연결된다.
- **수신 메시지 기본 한계 4MB**, 초과 시 `RESOURCE_EXHAUSTED`. 큰 페이로드는 16~64KB 청크 스트리밍으로 쪼개 메모리 피크와 재시도 비용을 줄여라.
- **압축은 대역폭을 CPU와 맞바꾼다.** per-message 단위(compressed-flag 비트), `grpc-encoding`/`grpc-accept-encoding`으로 협상. 작은 메시지나 이미 압축된 데이터에는 역효과.
- **`MAX_CONCURRENT_STREAMS`**(흔히 100~250)는 단일 연결을 병목으로 만든다. 채널 풀/클라이언트 LB로 분산하면 동시성, 멀티코어 활용, TCP HOL 영향 분산, 부하 분산을 동시에 얻는다. 단, 과도한 연결은 오버헤드.
- **직렬화(marshal/unmarshal)는 숨은 CPU 비용.** 객체 재사용, C++ arena, zero-copy `bytes`, 불필요한 변환 제거로 GC 압박을 줄여라. 바이너리엔 `string` 대신 `bytes`.
- **HPACK**은 반복 헤더를 인덱싱해 작은 RPC 다발의 헤더 오버헤드를 없앤다(연결 재사용이 전제). keepalive PING은 분 단위로, 배칭으로 RPC 고정 비용을 분산하라.
- **측정 없이 최적화 없다.** ghz로 throughput/p50/p99/연결수/에러를 측정하고, 평균이 아닌 **p99 꼬리 지연**을 보라. 3대 함정: 단일 연결 HOL(동시성 올려도 정체), GC 압박(p99 톱니), 동기 블로킹 스레드 고갈(동시성 증가 시 폭증).

---

## 연결 노트

- [[04 - HTTP2 깊이 보기 - 전송 계층]] — 프레임 구조, WINDOW_UPDATE/SETTINGS/DATA 프레임, HPACK, MAX_FRAME_SIZE의 토대.
- [[05 - 통신의 4가지 방식 - Unary와 Streaming]] — 스트리밍 모델과 StreamObserver/Send/Recv API의 기본기.
- [[03 - Protocol Buffers 2 - 인코딩과 와이어 포맷]] — gRPC 길이-접두사 프레이밍과 compressed-flag 비트.
- [[02 - Protocol Buffers 1 - 문법과 타입 시스템]] — `bytes` vs `string` 등 성능에 영향을 주는 타입 선택.
- [[07 - 채널 스텁 커넥션 생명주기]] — 채널/연결 재사용, 채널 풀의 기반.
- [[08 - 메타데이터와 인터셉터]] — `grpc-encoding`/`grpc-accept-encoding` 등 헤더 협상.
- [[09 - 에러 모델 - 상태 코드와 Rich Error]] — RESOURCE_EXHAUSTED 등 성능 한계 초과 시 상태 코드.
- [[10 - Deadline 취소 타임아웃]] — 블로킹 Send/Recv가 영원히 멈추지 않도록 하는 데드라인/취소, DEADLINE_EXCEEDED.
- [[12 - 이름 해석과 로드밸런싱]] — 채널 풀과 부하 분산, 단일 연결 핀 문제.
- [[13 - 안정성 - Retry Health Check Keepalive]] — keepalive PING 튜닝, 재시도 비용.
- [[14 - 관찰성과 디버깅 - Reflection grpcurl]] — 리플렉션 기반 ghz/grpcurl, 성능 디버깅 도구.
