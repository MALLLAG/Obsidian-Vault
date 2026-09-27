---
title: 안정성 - Retry Health Check Keepalive
date: 2026-06-26
tags: [grpc, retry, hedging, health-check, keepalive, goaway, reliability, resilience, 학습노트]
---

## 1. 왜 "안정성"에 장 하나를 통째로 쓰는가

지금까지의 장들은 대체로 "정상 경로(happy path)"를 다뤘다. [[02 - Protocol Buffers 1 - 문법과 타입 시스템]]에서 메시지를 정의했고, [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 그 메시지를 프레임에 실어 보냈고, [[05 - 통신의 4가지 방식 - Unary와 Streaming]]에서 네 가지 호출 방식으로 주고받았다. 그러나 실제 운영 환경은 정상 경로대로만 흘러가지 않는다.

분산 시스템 엔지니어가 기억해야 할 **8가지 오류(Fallacies of Distributed Computing)** 의 첫 두 항목이 "네트워크는 신뢰할 수 있다"와 "지연은 0이다"인 데에는 이유가 있다. 현실은 그 반대다.

```text
정상 경로가 깨지는 순간들
────────────────────────────────────────────────────────
[1] 백엔드 인스턴스 1대가 막 죽었다
     → 그 인스턴스로 라우팅된 요청은 connection refused / RST
[2] 롤링 배포 중 파드가 SIGTERM 받고 드레이닝(draining) 시작
     → in-flight 요청은 받아주지만 신규 연결은 거부
[3] NAT/로드밸런서가 유휴(idle) 60초 후 조용히 연결을 끊음
     → 클라는 죽은 줄 모르고 그 연결로 요청을 쏨 → 무한 대기
[4] GC stop-the-world / CPU 스로틀 → p99.9 지연이 수 초로 튐
[5] 순간적 패킷 손실로 TCP 재전송 → 한 요청만 비정상적으로 느림
────────────────────────────────────────────────────────
```

이 다섯 가지는 성격이 서로 다르므로 대응 방법도 다르다.

| 증상 | 본질 | gRPC의 처방 |
|------|------|-------------|
| [1] 인스턴스 사망, 즉시 거부 | **재시도하면 다른 백엔드가 받아줄 수 있음** | **Retry** (재전송) |
| [2] 드레이닝 중 신규 거부 | **graceful shutdown** | **GOAWAY** + Retry |
| [3] 좀비 연결(half-open) | **죽었는데 죽은 걸 모름** | **Keepalive PING** |
| [4]·[5] 특정 요청만 느림(tail) | **느림은 실패가 아니다** | **Hedging** |
| (공통) 어디로 보낼지 결정 | **건강한 백엔드만 골라야 함** | **Health Check** + LB |

이 장은 이 표의 순서대로 구성된다. 재시도와 헤징은 "이미 실패했거나 느린 요청을 어떻게 살릴 것인가"에 답한다. 헬스 체크는 "처음부터 건강한 백엔드로만 보내려면 어떻게 해야 하는가"에, 킵얼라이브와 연결 수명 관리는 "연결이 살아 있는지 어떻게 확인하고, 오래된 연결을 어떻게 정리할 것인가"에 답한다. 세 가지는 모두 같은 원칙을 따른다. **실패를 예외가 아니라 정상 상태의 일부로 보고 설계한다.**

먼저 중요한 사실 하나를 짚고 넘어가자. gRPC의 재시도와 헤징은 대부분 **service config** 라는 선언적 JSON으로 제어한다. 비즈니스 코드를 한 줄도 고치지 않고 채널을 만들 때 정책만 넣어 주면, 클라이언트 라이브러리(C-core, grpc-java, grpc-go 등)가 요청을 버퍼링하고 재전송하는 일을 알아서 처리한다. 이렇게 정책과 코드를 분리한 것이 gRPC 안정성 설계의 핵심이다.

---

## 2. 재시도(Retry): 실패를 흡수하는 첫 번째 방어선

### 2.1 service config와 retryPolicy

재시도는 메서드 단위로 `retryPolicy` 를 지정해서 켠다. 채널은 [[12 - 이름 해석과 로드밸런싱]]에서 다루는 이름 해석(name resolution) 과정에서 service config를 받아 오거나, 클라이언트가 직접 넣어 준 service config를 쓴다. 형태는 다음과 같다.

```json
{
  "methodConfig": [
    {
      "name": [
        { "service": "orders.OrderService", "method": "GetOrder" },
        { "service": "orders.OrderService", "method": "ListOrders" }
      ],
      "retryPolicy": {
        "maxAttempts": 4,
        "initialBackoff": "0.1s",
        "maxBackoff": "1s",
        "backoffMultiplier": 2.0,
        "retryableStatusCodes": ["UNAVAILABLE", "RESOURCE_EXHAUSTED"]
      }
    }
  ]
}
```

`name` 배열은 이 정책을 어떤 RPC에 적용할지 지정한다. `service` 만 적고 `method` 를 생략하면 그 서비스의 모든 메서드에 적용된다. `name: [{}]` 처럼 둘 다 비우면 채널의 기본 정책이 된다.

```text
name 매칭의 우선순위 (가장 구체적인 것이 이긴다)
────────────────────────────────────────────────
{service:"orders.OrderService", method:"GetOrder"}  ← 가장 구체적
{service:"orders.OrderService"}                      ← 서비스 전체
{}                                                    ← 채널 디폴트
```

### 2.2 다섯 개의 파라미터 살펴보기

| 필드                     | 의미                    | 제약 / 비고                                             |
| ---------------------- | --------------------- | --------------------------------------------------- |
| `maxAttempts`          | **총** 시도 횟수(최초 시도 포함) | 2 이상 필수. 즉 `4` = 최초 1회 + 재시도 3회. 구현은 상한(예: 5)으로 클램프 |
| `initialBackoff`       | 첫 재시도 전 기본 대기         | `"0.1s"` 형태(Duration 문자열). 0보다 커야 함                 |
| `maxBackoff`           | 백오프 상한                | 지수적으로 커져도 이 값을 넘지 않음                                |
| `backoffMultiplier`    | 매 재시도마다 곱하는 배수        | 보통 2.0. 0보다 커야 함                                    |
| `retryableStatusCodes` | **이 상태 코드일 때만** 재시도   | 비어 있으면 안 됨. 코드 이름(문자열) 또는 숫자                        |

이 가운데 가장 중요한 필드는 `retryableStatusCodes` 이다. gRPC 상태 코드는 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에서 자세히 다루며, 재시도 관점에서 가장 중요한 분류는 다음과 같다.

```text
재시도해도 "안전"하고 "의미 있는" 코드 vs 그렇지 않은 코드
──────────────────────────────────────────────────────────────
UNAVAILABLE(14)        ← 일시적 장애. 다른 백엔드면 성공할 수 있음. ★재시도 1순위
RESOURCE_EXHAUSTED(8)  ← 과부하/쿼터. 백오프 후 재시도가 합리적일 수 있음(주의)
ABORTED(10)            ← 동시성 충돌. 재시도가 설계상 정상일 수 있음
──────────────────────────────────────────────────────────────
INVALID_ARGUMENT(3)    ← 요청 자체가 틀림. 100번 보내도 100번 실패. ✗ 재시도 금지
NOT_FOUND(5)           ← 없는 건 다시 보내도 없음. ✗
PERMISSION_DENIED(7)   ← 권한 문제. 재시도 무의미. ✗
UNIMPLEMENTED(12)      ← 서버가 그 메서드를 모름. ✗
OK(0)                  ← 성공은 재시도 대상이 아님
──────────────────────────────────────────────────────────────
```

`UNAVAILABLE` 을 1순위로 꼽는 이유는 이 코드가 "서버가 요청을 **처리하지 못했다**"는 뜻을 강하게 담고 있기 때문이다. 처리하지 못했다면 부작용(side-effect)도 없었을 가능성이 높으므로 재시도해도 안전하다. 반대로 `DEADLINE_EXCEEDED`(4) 를 재시도 목록에 넣을 때는 매우 조심해야 한다. 데드라인 초과는 "서버가 처리하는 도중에 시간이 다 됐다"는 뜻일 수 있고, 그렇다면 첫 시도가 이미 부작용을 일으켰을 수 있다(아래 멱등성 절 참고).

[[10 - Deadline 취소 타임아웃]]에서 보았듯이 데드라인은 전체 호출에 걸리는 절대 시각이다. 따라서 모든 재시도는 그 하나의 데드라인 예산 안에서 이루어져야 한다.

### 2.3 백오프를 손으로 계산해 보자

지수 백오프(exponential backoff)에 지터(jitter)를 섞는 것이 표준 방식이다. gRFC A6가 정한 알고리즘은 다음과 같다.

```text
초기:  currentBackoff = initialBackoff
n번째 재시도 직전 대기시간 = uniform_random(0, currentBackoff)   ← 0~현재값 사이 균등 난수(지터)
대기 후:  currentBackoff = min(currentBackoff * backoffMultiplier, maxBackoff)
```

위 JSON(initial=0.1s, max=1s, multiplier=2.0)으로 실제 순서를 따라가 보자.

```text
시도1(최초)  ──실패(UNAVAILABLE)──┐
                                  │ currentBackoff=0.1s → 대기 = rand(0, 0.1s), 예: 0.063s
시도2        ──실패──┐  (대기 후 currentBackoff = min(0.1*2, 1) = 0.2s)
                     │ 대기 = rand(0, 0.2s), 예: 0.151s
시도3        ──실패──┐  (대기 후 currentBackoff = min(0.2*2, 1) = 0.4s)
                     │ 대기 = rand(0, 0.4s), 예: 0.300s
시도4        ──성공!  (maxAttempts=4 이므로 여기가 마지막 기회)
```

여기서 두 가지를 짚고 넘어가자.

1. **지터가 왜 필수인가.** 모든 클라이언트가 정확히 `0.1s, 0.2s, 0.4s` 뒤에 재시도한다면, 장애로 동시에 실패한 클라이언트 1만 개가 정확히 같은 순간에 한꺼번에 다시 몰려든다("thundering herd"). 균등 난수 지터는 이 재시도들을 시간축에 고르게 흩어서 서버가 회복할 여유를 만든다. gRPC는 `[0, currentBackoff]` 범위의 full jitter를 쓴다.

2. **maxAttempts 에 도달하면 끝난다.** 4번째 시도까지 실패하면 마지막 시도의 상태 코드가 그대로 애플리케이션에 전달된다. 재시도는 실패를 없애 주는 장치가 아니라, 정해진 횟수만큼 기회를 더 주는 장치일 뿐이다.

### 2.4 멱등성과 부작용: 재시도해도 되는 경우

재시도에서 가장 위험한 함정은 **부작용 중복**이다. 그림으로 살펴보자.

```text
[위험 시나리오] 서버는 처리에 성공했지만 응답이 클라에 도달하지 못함
────────────────────────────────────────────────────────────────────────
클라          네트워크            서버(CreateOrder)
 │   요청 ───────────────────────►│
 │                                │  주문 생성 ✓ (DB에 row 1개 INSERT)
 │                                │  응답 생성
 │            X (응답 유실/RST)◄───┤
 │  ← 클라는 "UNAVAILABLE" 로 인식
 │   재시도 ──────────────────────►│
 │                                │  또 주문 생성 ✓ (DB에 row 또 1개 INSERT!)
 │            응답 OK ◄────────────┤
 │  ← 클라는 "성공" 으로 인식
────────────────────────────────────────────────────────────────────────
결과: 사용자는 1번 눌렀는데 주문이 2개 생겼다.
```

이 때문에 **비멱등(non-idempotent) 메서드에 자동 재시도를 켜는 것은 위험**하다. 대응 방법은 두 가지다.

- **메서드를 멱등하게 설계한다.** 클라이언트가 만든 고유 키(idempotency key, 예: UUID)를 요청에 넣는다. 서버는 같은 키로 들어온 요청을 한 번만 처리하고, 두 번째부터는 저장해 둔 결과를 그대로 돌려준다. REST의 `Idempotency-Key` 헤더와 같은 방식이며, gRPC에서는 [[08 - 메타데이터와 인터셉터]]에서 다룬 메타데이터나 요청 메시지 필드에 실어 보낸다.

  ```proto
  message CreateOrderRequest {
    string idempotency_key = 1; // 클라가 생성하는 UUID. 서버는 이 키로 중복 제거
    string user_id = 2;
    repeated LineItem items = 3;
  }
  ```

- **재시도를 켜지 않는다.** 처음부터 멱등하게 만들 수 없는 연산(예: "잔액에서 100원 차감")이라면, 그 메서드의 `retryPolicy` 를 아예 비우거나 `retryableStatusCodes` 에서 의미가 모호한 코드를 뺀다.

여기서 gRPC는 강력한 안전장치를 하나 제공한다. **투명 재시도(transparent retry)** 와 **설정 재시도(configurable retry)** 를 구분하는 것이다.

```text
투명 재시도 (transparent retry) — 항상, 정책과 무관하게 동작
─────────────────────────────────────────────────────────────
조건: "서버가 요청을 받지 못했음이 확실"한 경우에만
  (a) RPC가 와이어로 나가기도 전에 실패 (연결 실패 등)
  (b) 서버가 요청을 처리하지 않았다고 명확히 신호 (REFUSED_STREAM 등)
→ 부작용이 절대 없으므로 비멱등 메서드라도 안전하게 재시도 가능
→ 보통 같은/다른 백엔드로 1~소수 회 자동 재전송

설정 재시도 (configurable retry) — retryPolicy 가 있을 때만
─────────────────────────────────────────────────────────────
조건: 서버가 응답(상태 코드)을 돌려준 뒤, 그 코드가 retryable 일 때
→ 서버가 요청을 "받았을 수도" 있음 → 부작용 가능 → 멱등성 책임은 사용자에게
```

이렇게 구분하기 때문에 "연결이 끊겨 요청이 나가지도 못한" 흔한 경우는 비멱등 메서드라도 안전하게 자동으로 복구된다. "서버가 받았을지도 모르는" 위험한 경우에만 사용자가 정책으로 명시적으로 opt-in 하면 된다. 책임이 어느 쪽에 있는지가 설계에서 분명하게 나뉜다.

### 2.5 재시도가 "다른 백엔드"로 가는 방식

클라이언트가 재시도하면 12장(이름 해석과 로드밸런싱)에서 다룬 로드밸런싱 정책(예: round_robin)이 피커(picker)를 다시 호출한다. 재시도는 **새로운 LB pick** 을 거치므로, 방금 죽은 백엔드가 아니라 살아 있는 다른 백엔드로 갈 가능성이 높다. 그래서 재시도는 단순히 같은 곳에 다시 보내는 것보다 훨씬 효과가 크다. 재시도와 로드밸런싱이 함께 동작해 장애 인스턴스를 피해 가기 때문이다.

```text
재시도 = 백오프 + 새 LB pick
──────────────────────────────────────────────
시도1 → picker → backend-A (죽음) → UNAVAILABLE
   ↓ 백오프 + 지터
시도2 → picker → backend-B (정상) → OK ✓
```

### 2.6 서버가 재시도를 직접 통제한다: pushback과 attempt 헤더

지금까지 본 재시도는 전적으로 클라이언트가 주도했다. 하지만 gRFC A6는 재시도를 **클라이언트와 서버가 협력하는 프로토콜**로 설계했다. 이 협력에 쓰이는 것이 특별한 메타데이터(8장 메타데이터와 인터셉터) 헤더 두 개다.

**(1) `grpc-previous-rpc-attempts`: "이건 몇 번째 재시도다"**

클라이언트는 최초 시도를 제외한 재시도마다 이 헤더에 **앞서 실패한 시도 횟수**를 담아 보낸다. 첫 시도에는 이 헤더가 없고, 첫 재시도에는 `1`, 두 번째 재시도에는 `2` 가 실린다.

```text
시도1(최초):   (헤더 없음)
시도2(재시도): grpc-previous-rpc-attempts: 1
시도3(재시도): grpc-previous-rpc-attempts: 2
```

서버는 이 값을 보고 지금 들어온 요청이 재시도라는 것을 안다. 이 정보는 2.4절의 멱등성 처리에 바로 쓸 수 있다. 같은 idempotency key를 두 번째로 받았고 이 헤더까지 붙어 있다면, 첫 시도가 부작용을 남겼는지 더 확실하게 판단해서 중복 처리를 막을 수 있다. 관측 측면에서도 서버가 재시도된 요청의 비율을 직접 집계할 수 있다.

**(2) `grpc-retry-pushback-ms`: "백오프는 서버가 정한다"**

과부하 상태인 서버는 클라이언트가 계산한 지수 백오프(2.3절)를 **무시하고 자기가 원하는 대기 시간을 지시**할 수 있다. 이때 서버는 응답의 트레일러(trailing metadata)에 이 헤더를 실어 보낸다.

```text
grpc-retry-pushback-ms 의 의미
──────────────────────────────────────────────────────────────
값 ≥ 0    → "다음 재시도는 정확히 이 ms만큼 기다린 뒤에 해라"
             (클라가 계산한 지수 백오프를 이 값으로 덮어쓴다. 단 1회성)
값 < 0    → "재시도하지 마라" (retryable 코드에 시도가 남았어도 즉시 커밋)
파싱 불가 → 재시도 안 함 (보수적으로 커밋)
──────────────────────────────────────────────────────────────
```

pushback은 5절에서 다룰 클라이언트 측 token bucket을 **서버 쪽에서 보완하는 throttling 수단**이다. token bucket은 클라이언트가 자신의 성공/실패 통계로 재시도를 조절하는 방식이지만, pushback은 부하가 몰린 당사자인 서버가 "지금은 더 오래 쉬어라"나 "아예 멈춰라"를 직접 지시하는 방식이다. 두 장치를 함께 써야 5절의 메타스태빌리티 방어가 완성된다.

헤징(4절)에서도 pushback은 똑같이 동작한다. 음수 pushback은 다음 헤지 요청을 막고, 양수 pushback은 그만큼 미룬다. 다만 pushback이 무엇을 지시하든 `maxAttempts` 와 throttling 토큰 한도는 여전히 우선 적용된다.

---

## 3. 재시도 커밋 규칙: 스트리밍에서는 언제 멈추는가

유너리(unary) RPC의 재시도는 직관적이다. 요청도 하나, 응답도 하나이므로 통째로 다시 보내면 된다. 하지만 5장(통신의 4가지 방식)에서 다룬 스트리밍에서는 이미 보낸 메시지와 이미 받은 메시지를 어떻게 처리할지가 까다롭다. gRPC는 **버퍼링(buffering)** 과 **커밋(commit)** 이라는 두 개념으로 이 문제를 해결한다.

### 3.1 클라이언트는 보낸 메시지를 버퍼에 보관한다

재시도하려면 클라이언트 라이브러리가 지금까지 이 RPC에서 서버로 보낸 요청 메시지를 모두 메모리에 보관하고 있어야 한다. 재시도할 때 그 버퍼의 내용을 처음부터 새 시도의 스트림으로 다시 보내야 하기 때문이다.

```text
client-streaming RPC 재시도 시 버퍼 replay
──────────────────────────────────────────────
시도1:  send(msg1) send(msg2) send(msg3) → UNAVAILABLE
        (buffer = [msg1, msg2, msg3])
시도2:  새 스트림 열고 → replay(msg1) replay(msg2) replay(msg3) → 이후 정상 진행
```

물론 이 버퍼를 무한히 키울 수는 없다. 그래서 한계가 두 가지 있다.

| 한계 | 의미 | C-core 기본값(예시) |
|------|------|---------------------|
| per-RPC 버퍼 한계 | 한 RPC가 보관할 수 있는 버퍼 상한 | `grpc.per_rpc_retry_buffer_size` ≈ 256KB |
| 채널 전체 버퍼 한계 | 채널의 모든 RPC가 공유하는 총량 | `grpc.retry_buffer_size` ≈ 256MB |

per-RPC 버퍼 한계를 넘으면, 즉 메시지를 너무 많이 보냈거나 너무 큰 메시지를 보내서 더 이상 보관할 수 없으면 그 RPC는 **커밋(commit)** 된다. 커밋은 "이제 현재 시도의 결과를 그대로 받아들이고, 더 이상 재시도하지 않는다"는 뜻이다.

### 3.2 커밋 시점: 더 이상 재시도할 수 없게 되는 순간

RPC는 다음 사건 중 **하나라도** 일어나면 현재 시도로 커밋된다.

```text
커밋(commit)을 유발하는 사건들
──────────────────────────────────────────────────────────────
[1] 서버로부터 첫 응답 메시지를 받음
    → 서버가 이미 일을 시작/완료했다는 증거. 되돌릴 수 없음
[2] per-RPC 버퍼 한계 초과
    → 더 이상 replay할 수 없으므로 현재 시도에 운명을 맡김
[3] 서버가 non-retryable 상태 코드로 응답
    → 재시도 목록에 없는 코드면 즉시 커밋(=재시도 안 함)
[4] retry throttling 토큰 고갈 (§5)
    → 예산이 없으니 더는 재시도 불가
[5] maxAttempts 소진
    → 마지막 시도가 곧 커밋
──────────────────────────────────────────────────────────────
커밋되면 → 버퍼의 메시지들을 해제(메모리 회수). 이후엔 일반 RPC처럼 동작.
```

이 가운데 [1]번인 **첫 응답 메시지 수신**이 가장 중요하다. server-streaming이나 bidi-streaming에서 서버가 메시지를 하나라도 클라이언트 애플리케이션에 전달하기 시작하면, 그 메시지를 취소하고 처음부터 다시 받을 수는 없다. 애플리케이션이 이미 그 메시지를 봤기 때문이다. 그래서 첫 메시지를 받으면 즉시 커밋된다.

```text
server-streaming 에서의 커밋
──────────────────────────────────────────────
시도1:  요청 보냄 → (서버가 msg1 보내려는 찰나) → 연결 끊김
        아직 클라가 아무 응답 메시지도 못 받음 → 재시도 가능
시도2:  요청 다시 보냄 → 서버 msg1 수신 ★커밋★ → msg2, msg3 ...
        이 시점부터 어떤 실패가 나도 재시도 없이 그대로 에러 전파
```

이 모델에서 알 수 있는 점은 **응답이 오기 시작한 long-lived 스트림은 사실상 재시도로 보호받지 못한다**는 것이다. 따라서 스트리밍의 안정성은 재시도보다 keepalive(7절)와 애플리케이션 수준의 재개(resume) 로직에 더 크게 의존한다.

---

## 4. Hedging: 기다리지 않고 미리 보낸다

### 4.1 재시도의 약점: 느린 응답은 해결하지 못한다

재시도는 실패, 즉 에러 코드에 반응한다. 그런데 1절의 [4]번처럼 GC 멈춤이나 CPU 스로틀 때문에 **특정 요청만 비정상적으로 느린** 상황은 에러가 아니다. 서버는 결국 응답을 준다. 단지 p99.9가 3초로 튀었을 뿐이다. 이런 경우 재시도는 데드라인이 초과되기 전까지 아무것도 할 수 없다.

```text
재시도로는 못 잡는 꼬리 지연(tail latency)
──────────────────────────────────────────────
요청 ───────────────────────────────────► (3초 후 응답)
     ↑ 이 3초 동안 클라는 그냥 기다림. 재시도 트리거 없음.
     ↑ 만약 다른 백엔드로 보냈으면 50ms에 끝났을 텐데...
```

### 4.2 헤징의 아이디어

헤징(hedging)은 **응답을 기다리지 않고**, 일정 시간(hedgingDelay)이 지나면 **같은 요청을 다른 백엔드에도 병렬로 보낸다**. 그리고 가장 먼저 도착한 정상 응답을 채택하고 나머지 요청은 취소한다. 이름은 "한 마리 말에 모두 걸지 않는다(hedge your bets)"는 금융 용어에서 왔다.

```json
{
  "methodConfig": [
    {
      "name": [{ "service": "weather.WeatherService", "method": "GetForecast" }],
      "hedgingPolicy": {
        "maxAttempts": 3,
        "hedgingDelay": "0.2s",
        "nonFatalStatusCodes": ["UNAVAILABLE", "RESOURCE_EXHAUSTED"]
      }
    }
  ]
}
```

```text
hedgingDelay = 0.2s, maxAttempts = 3 일 때 타임라인
────────────────────────────────────────────────────────────
t=0.0s   요청A 발사 → backend-1  (느림...)
t=0.2s   아직 응답 없음 → 요청B 발사 → backend-2  (병렬!)
t=0.25s  backend-2가 먼저 응답 OK ✓
         → 이 응답 채택, backend-1로 간 요청A는 CANCEL
t=0.4s   (만약 둘 다 응답 없었다면 요청C 발사 예정이었음)
────────────────────────────────────────────────────────────
실효 지연 = 0.25s  (재시도였다면 데드라인 직전까지 기다렸을 것)
```

| 필드                    | 의미                                                         |
| --------------------- | ---------------------------------------------------------- |
| `maxAttempts`         | 동시에 또는 차례로 보낼 수 있는 최대 시도 수                                    |
| `hedgingDelay`        | 다음 헤지 요청을 보내기까지의 지연. `0s` 면 처음부터 전부 보냄                       |
| `nonFatalStatusCodes` | 이 코드로 실패한 헤지는 "치명적이지 않음"으로 보고 다른 헤지를 계속 기다림. 그 밖의 코드는 즉시 전체 종료 |

`nonFatalStatusCodes` 의 의미는 조금 미묘하다. 헤지 중 하나가 `UNAVAILABLE`(nonFatal로 지정) 로 실패해도 다른 헤지가 성공할 수 있으므로 RPC를 끝내지 않는다. 반대로 `INVALID_ARGUMENT`(nonFatal 아님) 가 돌아오면 즉시 모든 헤지를 취소하고 그 에러를 전파한다. 요청 자체가 틀렸다는 뜻이므로 다른 백엔드도 똑같이 거부할 것이 분명하기 때문이다.

### 4.3 재시도와 헤징의 결정적 차이

```text
            재시도(Retry)                  헤징(Hedging)
──────────────────────────────────────────────────────────────
트리거      실패(에러 코드)를 받은 뒤      시간(hedgingDelay) 경과
방식        순차적 (하나 실패→다음)        병렬적 (응답 전에 추가 발사)
잡는 문제   가용성(availability)           꼬리 지연(tail latency)
백엔드 부하 1배 (실패한 만큼만)            최대 N배 (동시 다발)
중복 실행   기본적으로 없음(순차)          ★있음★ (여러 백엔드가 동시 처리)
멱등성 요구 retryable 코드일 때            ★항상 강하게★ 요구
──────────────────────────────────────────────────────────────
```

헤징의 **위험**은 분명하다. 같은 요청을 여러 백엔드가 **동시에** 처리할 수 있다는 것이다. 비멱등 메서드에 헤징을 켜면 2.4절의 "주문 2개" 문제가 거의 틀림없이 발생한다. 그래서 헤징은 **읽기 전용(read-only)·멱등 연산**에만 쓰는 것이 원칙이다(날씨 조회, 검색, 캐시 조회 등). 또한 헤징은 시스템에 추가 부하(최대 maxAttempts배)를 주므로, 폭주를 막으려면 반드시 5절의 throttling과 함께 써야 한다.

> [!note] 참고
> 한 메서드에 `retryPolicy` 와 `hedgingPolicy` 를 **동시에** 지정할 수는 없다. 둘은 상호 배타적이다. RPC마다 느린 응답이 문제인지, 실패가 문제인지를 보고 둘 중 하나를 고른다.

---

## 5. 재시도 폭주 막기: throttling과 메타스태빌리티

### 5.1 재시도가 시스템을 무너뜨리는 과정: 메타스태빌리티

재시도에는 장점과 함께 위험도 있다. 백엔드가 과부하 때문에 요청의 5%를 거부하기 시작했다고 하자. 모든 클라이언트가 재시도를 켜 두었다면 그 5%가 **추가 트래픽**이 되어 백엔드로 다시 몰린다. 부하가 더 늘어 거부율이 10%가 되고, 그 10%가 또 재시도로 돌아온다. 이렇게 **재시도가 부하를 키워서 시스템이 스스로 회복하지 못하게 되는** 상태를 **메타스태빌리티(metastable failure)** 라고 한다.

```text
재시도 폭주(retry storm) → 메타스태빌리티
──────────────────────────────────────────────────────────────
정상:   1000 req/s ──► 백엔드(여유)
       ↓ 백엔드 살짝 느려짐(GC 등)
경계:   1000 req/s + 50 재시도 = 1050 ──► 백엔드(더 느려짐)
       ↓
폭주:   1000 + 200 재시도 = 1200 ──► 더 느려짐 → 더 많은 실패 → 더 많은 재시도
       ↓
붕괴:   원인(GC)은 끝났는데도, 재시도 트래픽만으로 과부하가 "고착"됨
       → 부하를 0으로 줄이기 전엔 회복 불가 (metastable)
──────────────────────────────────────────────────────────────
```

**원래 장애 원인이 사라져도 재시도 트래픽 때문에 시스템은 계속 무너진 상태에 머문다.** 그래서 재시도에는 반드시 "예산(budget)"이 필요하다.

### 5.2 retryThrottling: 토큰 버킷으로 예산을 정한다

gRPC는 채널(정확히는 서버 이름) 단위로 **token bucket** 기반의 retry throttling을 제공한다. 이 설정은 service config의 최상위에 둔다.

```json
{
  "methodConfig": [ /* ... 위의 retryPolicy/hedgingPolicy ... */ ],
  "retryThrottling": {
    "maxTokens": 100,
    "tokenRatio": 0.1
  }
}
```

내부 동작은 놀랄 만큼 단순하면서도 효과적이다.

```text
토큰 버킷 알고리즘 (gRFC A6)
──────────────────────────────────────────────────────────────
초기:   token_count = maxTokens            (예: 100)
임계값: threshold   = maxTokens / 2         (예: 50)

매 RPC 완료 시:
  - 실패(failure)  → token_count -= 1
  - 성공(success)  → token_count += tokenRatio   (예: +0.1)
  (token_count 는 0 ~ maxTokens 로 클램프)

재시도/헤지 허용 여부:
  - token_count > threshold  → 재시도 허용
  - token_count <= threshold → ★재시도 금지★ (그냥 실패를 전파)
──────────────────────────────────────────────────────────────
```

이 규칙이 만드는 균형을 살펴보자. `tokenRatio = 0.1` 은 "성공 10번마다 토큰 1개 충전"을 뜻한다. **실패 1회는 토큰 1개를 소비하고, 성공 10회가 그 토큰을 다시 채운다.** 시스템이 건강해서 성공률이 90% 이상이면 토큰은 늘 가득 차 있고 재시도도 자유롭게 할 수 있다.

하지만 거부율이 치솟아 실패가 급증하면 토큰이 빠르게 줄어 50 밑으로 떨어지고, **그 순간부터 재시도가 전면 중단**된다. 폭주가 시작되기 전에 재시도가 자동으로 차단되는 것이다.

```text
거부율 급증 시 토큰이 마르는 모습
──────────────────────────────────────────────
실패 多, 성공 少
token: 100 → 99 → 98 → ... → 51 → 50 ↓
                                    └─ 여기서부터 재시도 OFF
                                       (성공이 회복되면 +0.1씩 다시 차오름)
```

이 throttling은 **재시도와 헤징 모두에** 적용된다. 특히 부하를 크게 늘리는 헤징에서는 이 안전장치가 결정적인 역할을 한다.

### 5.3 "재시도 비율 상한"으로 이해하기

토큰 버킷의 정상 상태를 분석하면 흥미로운 사실을 알 수 있다. 성공률이 `p`, 실패율이 `1-p` 일 때 토큰이 고갈되지 않고 유지되려면 충전량(성공×tokenRatio)이 소비량(실패×1)을 따라잡아야 한다. 대략 `tokenRatio` 가 작을수록(예: 0.1) 허용되는 재시도 비율이 보수적으로 제한된다.

운영할 때는 이렇게 기억하면 된다. **`tokenRatio` 는 "성공 몇 건이 재시도 1건의 예산을 만드는가"의 역수**이다. `0.1` 이면 성공 10건마다 재시도 1건의 예산이 생긴다. 실패가 그보다 잦아지면 재시도가 차단된다.

---

## 6. gRPC Health Checking Protocol

재시도와 헤징이 실패한 요청을 사후에 처리하는 방법이라면, 헬스 체크는 처음부터 건강한 백엔드로만 요청을 보내는 사전 예방책이다. gRPC는 이를 위해 표준 서비스 하나를 정의해 두었다.

### 6.1 표준 proto 정의

`grpc.health.v1.Health` 는 gRPC 프로젝트가 공식으로 배포하는 proto이다. 모든 언어 구현이 이 인터페이스를 공유한다.

```proto
syntax = "proto3";

package grpc.health.v1;

message HealthCheckRequest {
  string service = 1;  // 조회 대상 서비스명. "" 이면 서버 전체
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN          = 0;
    SERVING          = 1;
    NOT_SERVING      = 2;
    SERVICE_UNKNOWN  = 3;  // Watch 응답에서만 사용됨
  }
  ServingStatus status = 1;
}

service Health {
  // 단발성 조회: 지금 이 순간 건강한가?
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);

  // 스트리밍 구독: 상태가 바뀔 때마다 push 받는다
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

### 6.2 네 가지 ServingStatus의 의미

| 상태 | 의미 | 호출자가 해야 할 일 |
|------|------|----------------------|
| `SERVING`(1) | 정상, 트래픽 받을 준비됨 | 이 백엔드로 라우팅 OK |
| `NOT_SERVING`(2) | 살아 있지만 지금은 트래픽을 받지 못함(드레이닝/초기화 중/의존성 다운) | 트래픽 보내지 말 것 |
| `UNKNOWN`(0) | 상태를 아직 모름 | 보수적으로 다루기 |
| `SERVICE_UNKNOWN`(3) | 그런 서비스명을 모름(미등록) | Watch에서만 등장 |

`SERVICE_UNKNOWN` 을 처리하는 방식에는 미묘한 비대칭이 있다.

```text
미등록 서비스명을 물었을 때
──────────────────────────────────────────────────────────────
Check("does.not.Exist")  → gRPC 상태코드 NOT_FOUND(5) 로 RPC 자체가 실패
Watch("does.not.Exist")  → RPC는 성공, 응답 메시지 status = SERVICE_UNKNOWN(3)
                            (나중에 그 서비스가 등록되면 SERVING 으로 push)
──────────────────────────────────────────────────────────────
```

`Watch` 가 `SERVICE_UNKNOWN` 을 정상 응답으로 돌려주는 이유는 아직 뜨지 않았지만 곧 뜰 수도 있는 서비스를 구독하는 시나리오를 지원하기 위해서다. 서버는 스트림을 끊지 않고 유지하다가 서비스가 등록되면 `SERVING` 을 보낸다.

### 6.3 서비스명 키와 빈 문자열의 특별한 의미

`service` 필드는 **헬스 상태의 키**이다. 한 서버가 여러 논리 서비스(예: `orders.OrderService`, `users.UserService`)를 호스팅할 때 서비스마다 상태를 따로 보고할 수 있다.

```text
서버 하나, 여러 서비스의 독립 헬스
──────────────────────────────────────────────
key=""(전체)              → SERVING      ← 서버 프로세스 자체는 살아있음
key="orders.OrderService" → SERVING      ← 주문 서비스 정상
key="users.UserService"   → NOT_SERVING  ← 유저 서비스는 DB 끊겨 일시 불능
```

**빈 문자열 `""` 은 "서버 전체"** 를 뜻하도록 약속되어 있다. 호출자가 특정 서비스가 아니라 이 프로세스가 전반적으로 트래픽을 받을 수 있는지 알고 싶을 때 쓴다. 쿠버네티스 readiness probe나 단순한 로드밸런서가 주로 이 키를 확인한다.

### 6.4 서버 측 구현 (Go)

gRPC는 표준 헬스 서버 구현을 라이브러리로 제공하므로 proto를 직접 구현할 필요가 없다.

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/health"
    healthpb "google.golang.org/grpc/health/grpc_health_v1"
)

func main() {
    grpcServer := grpc.NewServer()

    // 1) 표준 헬스 서버 생성·등록
    healthSrv := health.NewServer()
    healthpb.RegisterHealthServer(grpcServer, healthSrv)

    // 2) 초기 상태 선언
    healthSrv.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)               // 전체
    healthSrv.SetServingStatus("orders.OrderService", healthpb.HealthCheckResponse_SERVING)

    // 3) 비즈니스 서비스 등록
    orders.RegisterOrderServiceServer(grpcServer, &orderServer{})

    // 4) 런타임에 의존성이 깨지면 상태를 NOT_SERVING 으로 전환
    go watchDatabase(func(up bool) {
        if up {
            healthSrv.SetServingStatus("orders.OrderService", healthpb.HealthCheckResponse_SERVING)
        } else {
            // DB가 끊기면 트래픽 받지 않겠다고 선언 → LB가 우회
            healthSrv.SetServingStatus("orders.OrderService", healthpb.HealthCheckResponse_NOT_SERVING)
        }
    })

    // 5) graceful shutdown 시 드레이닝 신호
    //    Shutdown()은 모든 키를 NOT_SERVING 으로 만들고 이후 변경을 막는다
    defer healthSrv.Shutdown()

    lis, _ := net.Listen("tcp", ":50051")
    grpcServer.Serve(lis)
}
```

`SetServingStatus` 를 호출하면 그 서비스를 `Watch` 하고 있는 모든 클라이언트에게 새 상태가 즉시 push된다. 헬스 체크의 진짜 장점이 여기에 있다. **상태 변화를 폴링 없이 실시간으로 전파**한다.

### 6.5 헬스 체크와 로드밸런싱, 오케스트레이터

헬스 정보는 두 계층에서 사용된다. 12장(이름 해석과 로드밸런싱)과 직접 맞물리는 부분이다.

```text
헬스 정보의 두 소비자
──────────────────────────────────────────────────────────────
[A] 클라이언트측 로드밸런싱 (gRPC 클라가 직접)
    채널이 각 백엔드를 Watch → NOT_SERVING 인 백엔드는 picker에서 제외
    → "health checking LB policy"

[B] 오케스트레이터 / 인프라 LB (쿠버네티스 등)
    kubelet 이 주기적으로 Check 호출 → readiness/liveness 판정
──────────────────────────────────────────────────────────────
```

쿠버네티스에서는 과거에 `grpc_health_probe` 라는 별도 바이너리를 컨테이너에 넣고 `exec` probe로 호출했다. 최신 쿠버네티스(1.24+ GA)는 **네이티브 gRPC probe** 를 지원한다.

```yaml
# 쿠버네티스 Pod spec — 네이티브 gRPC 헬스 probe
readinessProbe:
  grpc:
    port: 50051
    service: "orders.OrderService"   # 비우면 "" (서버 전체)를 Check
  initialDelaySeconds: 5
  periodSeconds: 10
livenessProbe:
  grpc:
    port: 50051
    # service 생략 → "" 키 = 프로세스 전체 생사
  periodSeconds: 10
```

여기서 헬스 체크 설계의 핵심은 **readiness 와 liveness 의 구분**이다.

| Probe | 질문 | 실패하면 | 매핑되는 ServingStatus 활용 |
|-------|------|----------|------------------------------|
| **liveness** | "프로세스가 살아 있나? 데드락 아닌가?" | 컨테이너 **재시작** | `""`(전체)를 Check. 죽었으면 응답 자체가 안 옴 |
| **readiness** | "지금 트래픽을 받을 수 있나?" | LB 풀에서 **일시 제외**(재시작 X) | 특정 서비스 키. `NOT_SERVING` 이면 제외 |

이 둘을 혼동하면 사고가 난다. 예를 들어 DB가 잠깐 끊겼을 때 liveness를 실패시키면 멀쩡한 프로세스가 재시작되어 상황이 더 나빠진다. 올바른 설계는 **DB 의존성은 readiness에만 반영**해서 LB 풀에서만 빠지고 프로세스는 살려 두는 것이다. 그래서 6.4절의 `watchDatabase` 콜백은 `orders.OrderService` 키(=readiness 대상)만 바꾸고 `""` 키(=liveness)는 건드리지 않는다.

---

## 7. Keepalive: 죽은 연결을 어떻게 아는가

### 7.1 좀비 연결(half-open) 문제

[[07 - 채널 스텁 커넥션 생명주기]]에서 보았듯이 gRPC 채널은 TCP 연결을 오래 재사용한다. 4장(HTTP2 깊이 보기)에서 다룬 멀티플렉싱(multiplexing) 덕분에 연결 하나로 수많은 RPC를 동시에 처리한다. 효율적이지만, **연결이 조용히 죽으면** 곤란해진다.

```text
half-open 연결의 비극
──────────────────────────────────────────────────────────────
클라  ──── TCP 연결 ────  중간 NAT/LB  ──── TCP 연결 ────  서버
                              │
                        [60초 유휴 후 NAT가 매핑 삭제]
                              X 연결 끊김(양쪽엔 통보 없음)
클라는 연결이 "살아있다"고 믿음
 → 그 연결로 RPC 발사 → 패킷이 블랙홀로 → 데드라인까지 무한 대기
──────────────────────────────────────────────────────────────
```

TCP에도 keepalive가 있지만 기본값이 보통 2시간이라 장애를 제때 감지하는 데는 쓸모가 없다. 그래서 gRPC는 **HTTP/2 PING 프레임**을 이용해 애플리케이션 수준의 keepalive를 직접 구현한다.

### 7.2 HTTP/2 PING 프레임을 바이트 단위로 보기

PING은 4장에서 다룬 HTTP/2 프레임의 한 종류(type `0x6`)이며, RFC 7540 §6.7에 정의되어 있다. 길이는 **항상 8 octet**(불투명 데이터)이고, 스트림 0(연결 전체)에서만 보낸다.

```text
HTTP/2 프레임 헤더(9 byte) + PING payload(8 byte) = 17 byte
──────────────────────────────────────────────────────────────
PING 송신:
  00 00 08   06   00   00 00 00 00   01 02 03 04 05 06 07 08
  └길이=8┘  type  flags  stream=0     └── 8-byte opaque data ──┘
            0x6   0x00   (PING은 항상 stream 0)

PING ACK 응답 (수신측이 같은 데이터를 그대로 echo):
  00 00 08   06   01   00 00 00 00   01 02 03 04 05 06 07 08
                  └ACK(0x1)          └─ 받은 데이터 그대로 ─┘
──────────────────────────────────────────────────────────────
```

직접 디코딩해 보자. `00 00 08` 은 24비트 길이로 값은 8이다. `06` 은 프레임 타입으로 PING을 뜻한다. flags는 송신할 때 `00`, ACK로 응답할 때 `01`(ACK 비트)이다. `00 00 00 00` 은 스트림 식별자 0으로, 연결 수준 프레임임을 나타낸다.

이어지는 8바이트는 송신자가 정한 임의의 불투명 데이터이고, 수신자는 ACK에 **반드시 같은 8바이트를 그대로 되돌려** 보내야 한다. 이렇게 데이터를 되돌려 받기 때문에 송신자는 이 응답이 자신이 보낸 PING에 대한 것임을 확신할 수 있다.

### 7.3 keepalive의 동작 원리와 파라미터

클라이언트(또는 서버)는 주기적으로 PING을 보내고, 정해진 시간 안에 ACK가 오지 않으면 연결이 죽었다고 판단해 연결을 끊고 다시 연결한다.

```text
keepalive 라이프사이클
──────────────────────────────────────────────────────────────
       keepalive_time 경과 (예: 10초간 활동 없음)
          │
          ▼
       PING 송신 ───────────────► 서버
          │                         │
          │  keepalive_timeout      │  (정상) PING ACK
          │  타이머 시작(예: 3초)   ◄─────────────
          │                         │
   ┌──────┴──────┐
   │ ACK 도착?    │
   ├─ 예 ────────► 연결 건강. 타이머 리셋. 다시 대기.
   └─ 아니오(타임아웃) ► 연결 죽음 판정 → 연결 종료 → 모든 RPC 실패 → 재연결
──────────────────────────────────────────────────────────────
```

핵심 파라미터는 세 가지다.

| 파라미터 | 의미 | C-core 키 | 흔한 기본값 |
|----------|------|-----------|-------------|
| keepalive **time** | 활동이 없을 때 PING을 보내는 간격 | `grpc.keepalive_time_ms` | 비활성(매우 큼). 명시 권장 |
| keepalive **timeout** | PING ACK를 기다리는 시간 | `grpc.keepalive_timeout_ms` | 20000 (20s) |
| **permit_without_calls** | 진행 중인 RPC가 없어도 PING을 보낼지 여부 | `grpc.keepalive_permit_without_calls` | 0 (false) |

특히 `permit_without_calls` 가 중요하다. 기본값(false)에서는 **진행 중인 RPC(stream)가 하나라도 있을 때만** keepalive PING을 보낸다. 즉 유휴 연결에는 PING을 보내지 않는다. 그런데 7.1절의 NAT 타임아웃 시나리오는 바로 유휴 연결에서 발생한다.

따라서 유휴 연결도 유지하고 싶다면 `permit_without_calls = true` 로 켜야 한다. 다만 이 옵션을 켜면 서버 측 강제 정책(8절)과 충돌해서 오히려 연결이 끊길 수 있으므로 양쪽 설정을 맞춰야 한다.

### 7.4 클라이언트 keepalive 설정 (Go / Java / Python)

```go
// Go — 클라이언트
import "google.golang.org/grpc/keepalive"

conn, err := grpc.NewClient(target,
    grpc.WithTransportCredentials(creds),
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                10 * time.Second, // 10초 유휴마다 PING
        Timeout:             3 * time.Second,  // 3초 내 ACK 없으면 연결 폐기
        PermitWithoutStream: true,             // RPC 없어도 PING (유휴 연결 보호)
    }),
)
```

```java
// Java — 클라이언트
ManagedChannel channel = ManagedChannelBuilder.forTarget(target)
    .keepAliveTime(10, TimeUnit.SECONDS)
    .keepAliveTimeout(3, TimeUnit.SECONDS)
    .keepAliveWithoutCalls(true)
    .build();
```

```python
# Python — 클라이언트
channel = grpc.secure_channel(target, creds, options=[
    ("grpc.keepalive_time_ms", 10000),
    ("grpc.keepalive_timeout_ms", 3000),
    ("grpc.keepalive_permit_without_calls", 1),
    # 데이터 없이 보낼 수 있는 PING 최대 개수(0 = 제한 없음).
    # 서버 강제 정책과 맞춰야 함 (§8)
    ("grpc.http2.max_pings_without_data", 0),
])
```

`grpc.http2.max_pings_without_data` 는 클라이언트가 스스로 거는 안전장치이다. 기본값 2는 데이터(헤더/메시지) 프레임 없이 연속으로 PING을 2개까지만 보낸다는 뜻이다. 이 제한이 유휴 연결의 keepalive를 의도와 다르게 막을 수 있어서 `0`(무제한)으로 두는 경우가 많다.

---

## 8. 서버 측 keepalive 강제(enforcement)

### 8.1 서버가 PING을 "방어"해야 하는 이유

클라이언트가 keepalive PING을 마음대로 보내게 두면, 악의적이거나 잘못 설정된 클라이언트가 **초당 수십 개의 PING으로 서버에 부담을 줄 수** 있다. 서버는 PING마다 ACK를 만들어 보내야 하므로 CPU와 대역폭이 소모된다. 일종의 DoS이다. 그래서 서버는 지나치게 잦은 PING을 감지해 막을 수 있다. 이를 **keepalive enforcement** 라고 한다.

### 8.2 ping strike와 GOAWAY too_many_pings

서버는 다음 enforcement 파라미터로 PING을 제한한다.

| 파라미터 | 의미 | C-core 키 | 흔한 기본값 |
|----------|------|-----------|-------------|
| 최소 수신 PING 간격 | 이보다 자주 오는 PING은 "위반(strike)" | `grpc.http2.min_ping_interval_without_data_ms` (서버 **수신측** 강제. 클라가 스스로 거는 송신측 하한은 별개 키 `grpc.http2.min_time_between_pings_ms`) | 300000 (5분) |
| 무콜 PING 허용 여부 | 진행 중 RPC 없을 때 PING 허용? | grpc-go `EnforcementPolicy.PermitWithoutStream` (C-core는 "데이터 없음" 판정에 내포) | false |
| 허용 strike 수 | 몇 번 위반까지 허용하나 | `grpc.http2.max_ping_strikes` | 2 |

동작 흐름은 다음과 같다.

```text
서버측 ping enforcement
──────────────────────────────────────────────────────────────
서버: "데이터 없는 연결에서는 5분에 한 번보다 자주 PING 보내지 마"
       (min_recv_ping_interval_without_data_ms = 300000)

클라가 10초마다 PING을 쏨 → 5분 규칙 위반
  위반 1회 → ping strike 카운터 = 1  (max_ping_strikes=2 이므로 아직 봐줌)
  위반 2회 → strike = 2
  위반 3회 → strike > max(2) → ★단호한 응징★
       서버가 GOAWAY 프레임 전송:
         error code   = ENHANCE_YOUR_CALM (0xb)
         debug data   = "too_many_pings"
       → 연결을 끊는다. 클라의 모든 RPC가 깨지고 재연결해야 함.
──────────────────────────────────────────────────────────────
```

`ENHANCE_YOUR_CALM`("진정 좀 해라")은 RFC 7540 §7에 정의된 HTTP/2 에러 코드 `0xb`이며, gRPC는 PING이 지나치게 많을 때 이 코드를 쓴다. GOAWAY 프레임을 바이트 단위로 살펴보자.

```text
GOAWAY 프레임 (type 0x7), payload = lastStreamId(4) + errorCode(4) + debugData
──────────────────────────────────────────────────────────────────────────
00 00 16   07   00   00 00 00 00 | 00 00 00 09 | 00 00 00 0b | 74 6f 6f ...
└길이=22┘  type flags  stream=0  | lastStream=9| err=0xb     | "too_many_pings"
           0x7                     (마지막 수락   ENHANCE_      └ 14 byte debug
                                    스트림 ID)    YOUR_CALM       data ─┘

debug data "too_many_pings" 바이트 디코딩:
  74='t' 6f='o' 6f='o' 5f='_' 6d='m' 61='a' 6e='n' 79='y'
  5f='_' 70='p' 69='i' 6e='n' 67='g' 73='s'   → 총 14 byte
payload 길이 = 4 + 4 + 14 = 22 = 0x16  ✓ (프레임 헤더 길이 필드와 일치)
```

GOAWAY의 또 다른 핵심은 `lastStreamId` 이다. 이 값은 "마지막으로 받아들인 스트림 ID는 9번이다. 그보다 큰 ID로 시작한 요청은 처리하지 않았으니 안전하게 재시도해라"라는 뜻이다. 이 정보 덕분에 클라이언트는 GOAWAY 이후 어떤 RPC를 **투명 재시도**(2.4절)할 수 있는지 정확히 알 수 있다.

### 8.3 설정 불일치가 만드는 연결 끊김 장애: 실전 함정

이 장에서 실무적으로 가장 중요한 부분이다. 클라이언트의 keepalive_time과 서버의 min_ping_interval이 어긋나면 **멀쩡한 시스템에서 연결이 주기적으로 끊기는**, 원인을 찾기 어려운 장애가 발생한다.

```text
[장애 시나리오] 클라는 공격적, 서버는 보수적
──────────────────────────────────────────────────────────────
클라 설정:  keepalive_time = 10s, permit_without_calls = true
서버 설정:  min_recv_ping_interval = 300s(5분), permit_without_calls = false
                                     (= 기본값을 그대로 둠)

결과:
  t=0    유휴 연결. 클라가 10초마다 PING 발사 (RPC 없는데도)
  t=10   PING → 서버: "RPC도 없는데 PING? + 5분 규칙 위반" strike=1
  t=20   PING → strike=2
  t=30   PING → strike=3 > max(2) → GOAWAY(too_many_pings) → 연결 끊김!
  t=30+  클라 재연결 → 또 10초마다 PING → 또 30초 후 끊김 → 무한 반복
──────────────────────────────────────────────────────────────
증상:  로그에 주기적 reconnect, "too_many_pings", 간헐적 RPC 실패
원인:  클라 keepalive_time(10s) << 서버 min_ping_interval(300s)
       + 서버는 무콜 PING 불허인데 클라는 permit_without_calls=true
```

**설정 원칙**: 클라이언트의 `keepalive_time` 은 항상 서버의 `min_recv_ping_interval` 보다 **크거나 같아야** 한다. 또한 양쪽의 `permit_without_calls` 정책을 일치시켜야 한다.

```text
올바른 정렬
──────────────────────────────────────────────
서버:  permitKeepAliveTime = 10s   (= 10초보다 자주만 아니면 OK)
       permitKeepAliveWithoutCalls = true
클라:  keepAliveTime = 30s         (서버 허용 간격 ≥ 보다 여유 있게)
       keepAliveWithoutCalls = true
──────────────────────────────────────────────
→ 클라는 30초마다(서버 허용 10초보다 드물게) PING → strike 안 쌓임 → 안정
```

서버 측 Java 설정 예:

```java
Server server = NettyServerBuilder.forPort(50051)
    // 서버가 클라에게 거는 keepalive (서버→클라 PING)
    .keepAliveTime(5, TimeUnit.MINUTES)
    .keepAliveTimeout(20, TimeUnit.SECONDS)
    // ★enforcement: 클라가 이보다 자주 PING하면 strike★
    .permitKeepAliveTime(10, TimeUnit.SECONDS)
    .permitKeepAliveWithoutCalls(true)   // 무콜 PING도 허용
    // 연결 수명 관리(§9)
    .maxConnectionIdle(15, TimeUnit.MINUTES)
    .maxConnectionAge(30, TimeUnit.MINUTES)
    .maxConnectionAgeGrace(5, TimeUnit.MINUTES)
    .build();
```

Go 서버:

```go
grpcServer := grpc.NewServer(
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             10 * time.Second, // 이보다 자주 PING하면 strike
        PermitWithoutStream: true,             // 무콜 PING 허용
    }),
    grpc.KeepaliveParams(keepalive.ServerParameters{
        Time:                  5 * time.Minute,  // 서버→클라 PING 간격
        Timeout:               20 * time.Second,
        MaxConnectionIdle:     15 * time.Minute, // §9
        MaxConnectionAge:      30 * time.Minute,
        MaxConnectionAgeGrace: 5 * time.Minute,
    }),
)
```

---

## 9. 연결 수명 관리: 주기적 재분배와 graceful shutdown

keepalive가 죽은 연결을 빨리 감지하는 장치라면, 연결 수명 관리는 반대로 **멀쩡한 연결도 일정 시간이 지나면 일부러 정리**하는 장치이다. 왜 멀쩡한 연결을 끊으려 할까?

### 9.1 오래된 연결이 문제가 되는 이유

```text
오래 사는 연결의 부작용
──────────────────────────────────────────────────────────────
[1] 로드 불균형 고착
    클라가 한 번 backend-A에 붙으면 그 연결을 영원히 재사용.
    오토스케일로 backend-D가 새로 떠도, 기존 클라는 D로 안 옮겨감.
    → 신규 백엔드는 놀고 기존 백엔드만 과부하

[2] DNS/배포 변화 무시
    backend-A가 교체될 예정인데, 살아있는 연결은 옛 인스턴스를 계속 붙듦

[3] 리소스 누수
    오래된 연결에 누적된 상태/메모리
──────────────────────────────────────────────────────────────
```

해결 방법은 연결에 만료 시각을 두고 주기적으로 끊어서, 클라이언트가 이름 해석과 LB pick을 다시 하게 만드는 것이다. 다시 pick하면 12장(이름 해석과 로드밸런싱)의 정책이 최신 백엔드 목록을 반영해 부하를 다시 분배한다.

### 9.2 세 가지 연결 수명 파라미터

| 파라미터 | 의미 | 기본값 |
|----------|------|--------|
| `MAX_CONNECTION_IDLE` | 진행 중인 RPC가 없는 상태로 이만큼 지나면 연결을 닫음 | 무한(off) |
| `MAX_CONNECTION_AGE` | 연결이 생긴 지 이만큼 지나면(±지터) 닫기 시작 | 무한(off) |
| `MAX_CONNECTION_AGE_GRACE` | AGE 도달 후 in-flight RPC가 끝나기를 기다려 주는 유예 시간 | 무한 |

`MAX_CONNECTION_AGE` 에는 약간의 무작위 지터가 더해진다. 모든 연결이 정확히 30분에 동시에 끊기면 재연결이 한꺼번에 몰리는 thundering herd가 발생하므로, 의도적으로 시점을 분산한다(2.3절의 지터와 같은 원리).

### 9.3 GOAWAY 드레이닝: graceful shutdown의 표준 방식

연결을 끊을 때 그냥 TCP RST를 보내면 in-flight RPC가 모두 깨진다. 그래서 gRPC는 **GOAWAY를 이용한 2단계 우아한 종료(graceful drain)** 를 한다. 4장(HTTP2 깊이 보기)에서 다룬 GOAWAY 메커니즘을 안정성 관점에서 활용하는 것이다.

```text
MAX_CONNECTION_AGE 도달 시(또는 서버 graceful shutdown 시) 드레이닝
──────────────────────────────────────────────────────────────────────
[1단계] 서버 → 클라:  GOAWAY (lastStreamId = 2^31-1, NO_ERROR)
        "지금 진행 중인 스트림은 다 받아줄게. 근데 곧 끊을 거니까
         새 스트림은 다른 연결에서 시작해."
        → 클라는 이 연결에 새 RPC를 더 싣지 않고, 새 연결을 만들기 시작

[대기]  in-flight RPC들이 자연히 완료되기를 기다림 (grace period 동안)

[2단계] grace 시간이 다 되면 서버 → 클라:  GOAWAY (lastStreamId = 실제 마지막)
        "이제 진짜 끝. lastStreamId 초과분은 처리 안 했으니 재시도해."
        → 남은 연결을 닫음
──────────────────────────────────────────────────────────────────────
```

핵심은 이 "이중 GOAWAY" 패턴이다. 첫 GOAWAY는 `lastStreamId` 를 일부러 최댓값(`2^31-1`)으로 보내서 "아직 어떤 스트림도 거부하지 않았고, 진행 중인 스트림은 모두 받아 준다"는 신호를 준다. 클라이언트는 이 신호를 보고 새 연결을 준비하기 시작하고, 기존 RPC는 그대로 마무리한다.

두 번째 GOAWAY는 실제 마지막 스트림 ID를 담아 정확한 경계를 알려 준다. 그 경계를 넘는 RPC는 처리되지 않았음이 보장되므로 **투명 재시도**(2.4절)로 안전하게 복구된다.

```text
롤링 배포 시 무중단의 비밀
──────────────────────────────────────────────
파드가 SIGTERM 받음
 → grpcServer.GracefulStop()  (Go) / server.shutdown() (Java)
 → 모든 연결에 GOAWAY 드레이닝
 → in-flight RPC 완료 대기
 → 클라는 GOAWAY 보고 살아있는 다른 파드로 새 연결
 → 사용자는 단 한 건의 실패도 못 느낌 ✓
```

이것이 1절 표의 [2]번 "드레이닝 중 신규 거부" 문제에 대한 해답이다. graceful shutdown과 GOAWAY, 투명 재시도가 함께 동작해 무중단 배포를 가능하게 한다.

---

## 10. 종합: 데드라인·재시도·헤징의 상호작용과 안티패턴

### 10.1 데드라인이라는 절대 예산

10장(Deadline 취소 타임아웃)에서 보았듯이 데드라인은 RPC 전체에 걸리는 **절대 마감 시각**이다. 핵심은 데드라인이 **모든 재시도와 헤지를 포함한 전체 시간**에 적용된다는 점이다.

```text
데드라인은 시도 하나하나가 아니라 RPC 전체에 걸린다
──────────────────────────────────────────────────────────────
deadline = 2초
 ├ 시도1 (0.4s) 실패
 ├ 백오프 (0.1s)
 ├ 시도2 (0.5s) 실패
 ├ 백오프 (0.2s)
 ├ 시도3 (0.6s) 실패
 ├ 백오프 (0.4s) → 누적 2.2s > 2초!
 └ ✗ 더 재시도할 시간 예산 없음 → DEADLINE_EXCEEDED 로 종료
──────────────────────────────────────────────────────────────
maxAttempts=5 라도, 데드라인이 먼저 터지면 거기서 끝.
데드라인은 재시도/헤징보다 상위의 강제 종료선이다.
```

그래서 재시도 정책을 정할 때는 항상 데드라인과 함께 계산해야 한다. `initialBackoff` 와 `maxAttempts` 가 데드라인을 모두 소모하지 않도록 하고, 각 시도가 최소한의 처리 시간을 확보하도록 균형을 맞춘다.

데드라인은 4장(HTTP2 깊이 보기)에서 다룬 `grpc-timeout` 헤더로 서버에도 전파된다. 따라서 서버도 어차피 시간 초과될 작업을 일찍 포기(early cancellation)할 수 있다. 이것이 5절의 메타스태빌리티를 줄이는 또 다른 방어선이다.

### 10.2 안정성 안티패턴 모음

지금까지 살펴본 메커니즘을 반대로 쓰면 안티패턴이 된다. 운영 환경에서 실제로 사고를 일으키는 사례를 모았다.

| 안티패턴 | 무슨 일이 벌어지나 | 올바른 처방 |
|----------|--------------------|-------------|
| **무한/과도 재시도** (throttling 없이) | 장애 시 재시도 폭주 → 메타스태빌리티 → 회복 불능 | `retryThrottling` 필수 + `maxAttempts` 합리적으로 |
| **비멱등 메서드에 재시도/헤징** | `CreateOrder` 2번 실행 → 중복 주문/이중 부과 | idempotency key 설계 or 정책 끄기 (2.4절) |
| **`DEADLINE_EXCEEDED` 를 retryable로** | 서버가 처리 중인데 또 보냄 → 부작용 중복 + 부하 증폭 | retryable은 `UNAVAILABLE` 중심으로 |
| **헤징을 쓰기(write) 연산에** | 여러 백엔드가 동시에 같은 쓰기 실행 | 읽기 전용·멱등 연산에만 |
| **클라 keepalive_time이 서버 min_ping_interval보다 짧음** | `too_many_pings` GOAWAY로 주기적 끊김 (8.3절) | 클라 time ≥ 서버 min, permit 정책 일치 |
| **`permit_without_calls` 양쪽 불일치** | 유휴 연결에서 strike 누적 → 끊김 | 클라/서버 동일하게 설정 |
| **keepalive_time을 1초처럼 과하게** | PING 폭주로 CPU/대역폭 낭비, strike 위험 | 보통 10~60초 이상, NAT 타임아웃보다 짧게만 |
| **liveness probe에 DB 의존성 포함** | DB 깜빡임에 멀쩡한 파드가 재시작됨 | DB는 readiness에만 (6.5절) |
| **헬스 상태를 갱신 안 함** | DB 끊겨도 `SERVING` 유지 → 죽은 곳으로 트래픽 | 의존성 변화를 `SetServingStatus` 로 반영 |
| **재시도만 믿고 데드라인 미설정** | 좀비 연결에서 RPC가 영원히 매달림 | 항상 데드라인 + keepalive 동반 |

### 10.3 세 방어선의 협력: 전체 그림

마지막으로 이 장에서 다룬 메커니즘이 요청 하나에서 어떻게 함께 동작하는지 그려 보자.

```text
한 RPC가 겪는 안정성 메커니즘의 총동원
──────────────────────────────────────────────────────────────────────
[사전]  Health Check
         → LB가 SERVING 인 백엔드 풀만 유지 (NOT_SERVING 인 곳 제외)
            │
[발사]  클라가 deadline(2s) 걸고 RPC 발사 → picker가 backend-B 선택
            │
[감지]  keepalive가 백그라운드에서 연결 생사 감시 (10초마다 PING)
            │
[실패]  backend-B가 UNAVAILABLE 반환
            │
[예산]  retryThrottling 토큰 확인 → token > threshold → 재시도 허용
            │
[복구]  백오프(지터) 대기 → 새 LB pick → backend-C 선택 → 재전송
            │
[성공]  backend-C 가 OK → 토큰 += tokenRatio → 애플리케이션에 결과 전달
──────────────────────────────────────────────────────────────────────
그동안 backend-B가 롤링 배포로 내려가면:
   GOAWAY 드레이닝 → in-flight 완료 보장 → 새 연결은 backend-C/D로
──────────────────────────────────────────────────────────────────────
```

이 그림이 보여 주는 점은 분명하다. **어느 메커니즘도 혼자서는 충분하지 않다.** Health Check 없이 재시도만 하면 죽은 백엔드에 계속 요청을 보낸다. keepalive 없이 재시도하면 좀비 연결에서 데드라인까지 기다리게 된다. throttling 없이 재시도하면 재시도가 폭주하고, 데드라인 없이 모든 기능을 켜면 영원히 끝나지 않는 RPC가 생긴다. 안정성은 이 방어선들이 **함께 동작할 때(ensemble)** 확보된다.
