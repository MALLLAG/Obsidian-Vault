---
title: "10 - Deadline 취소 타임아웃"
date: 2026-06-26
tags:
  - grpc
  - deadline
  - timeout
  - cancellation
  - context
  - propagation
  - deadline-exceeded
  - rst-stream
  - backpressure
  - 학습노트
---

## 들어가며: 마감이 없는 약속은 약속이 아니다

분산 시스템에서 가장 위험한 상태는 "에러"가 아니라 "끝없이 기다리는 상태"이다. 에러가 나면 빠르게 실패(fail-fast)한 뒤 재시도하거나 대체 경로로 넘어갈 수 있다. 그러나 응답도 에러도 오지 않고 멈춰 있는 호출은 호출한 쪽의 스레드와 메모리, 커넥션을 붙들고 있고, 그 호출을 기다리는 상위 호출자의 자원까지 차례로 소모한다. 한 곳에서 시작된 지연이 호출 체인을 거슬러 올라가며 시스템 전체를 마비시키는 현상을 연쇄 장애(cascading failure)라고 한다.

이 장의 주제인 deadline은 이런 "끝없는 기다림"을 구조적으로 막는 장치이다. 다른 RPC 프레임워크 대부분과 달리 gRPC는 deadline을 부가 기능이 아니라 핵심 프로토콜의 일부로 정했다. 모든 RPC에는 명시적이든 암묵적이든 마감이 있고, 이 마감은 와이어 포맷에 담겨 전달되며, 마감이 지나면 호출은 자동으로 종료된다.

gRPC가 이 개념을 "timeout(타임아웃)"이 아니라 "deadline(데드라인)"이라고 부른다는 점도 눈여겨볼 만하다. 두 단어는 일상에서 거의 같은 뜻으로 쓰이지만, gRPC의 설계에서는 분명히 구분된다. 그리고 이 구분이 deadline 전파라는 강력한 기능의 토대가 된다. 먼저 이 차이부터 정확히 짚고 넘어가자.

---

## 1. deadline vs timeout: 절대 시각과 상대 시간

가장 먼저 기억해야 할 구분이다.

| 구분 | timeout (타임아웃) | deadline (데드라인) |
|------|-------------------|---------------------|
| 의미 | "얼마 동안" 기다릴까 | "언제까지" 기다릴까 |
| 성격 | 상대 시간(duration) | 절대 시각(point in time) |
| 예시 | "지금부터 5초" | "13:00:05.000 UTC까지" |
| 기준점 | 측정을 시작하는 순간 | 벽시계(또는 단조 시계)의 한 지점 |
| 합성 | 매번 다시 계산해야 함 | 그냥 전달하면 됨 |

비유로 시작해 보자. 친구에게 "30분 줄게, 그 안에 와"라고 말하는 것이 timeout이고, "3시까지 와"라고 말하는 것이 deadline이다. 두 말은 같은 약속처럼 보이지만 결정적인 차이가 있다.

친구 A에게 "3시까지 와"라고 말했고 A가 이 약속을 다시 B에게 전달한다면, A는 똑같이 "3시까지 와"라고 말하면 된다. 하지만 "30분 줄게"라고 말했다면, A는 자신이 이미 10분을 썼다는 것을 계산해서 B에게 "20분 줄게"라고 고쳐 말해야 한다. 이렇게 계산이 끼어들면 실수가 생긴다.

분산 호출 체인이 바로 이런 구조이다. 클라이언트가 서비스 A를 호출하고, A가 B를, B가 C를 호출한다. 사용자는 "이 요청 전체가 1초 안에 끝나야 한다"고 기대하므로, A와 B, C가 모두 이 1초라는 마감을 공유해야 한다. 이를 deadline(절대 시각)으로 표현하면 "현재 시각 + 1초 = T"라는 하나의 값이 체인 전체에 그대로 전달된다. A는 T를 B에게, B는 T를 C에게 그대로 넘긴다. 각 단계가 시간을 얼마나 썼든 마감 시각 자체는 변하지 않는다.

> [!note] 핵심 직관
> deadline은 호출 체인을 따라 전달되는 동안 **변하지 않는 불변량(invariant)** 이다. 그래서 합성(compose)하기 쉽고, 그래서 gRPC가 deadline을 1급 개념으로 선택했다.

### gRPC가 내부적으로 deadline을 기준으로 동작하는 이유

프로그래머가 API에서 보는 것은 보통 timeout이다(예: `WithTimeout(ctx, time.Second)`). 그런데 왜 "내부적으로는 deadline"이라고 말할까? 핵심은 **계산이 단 한 번만 일어난다**는 점이다.

- 프로그래머는 편의상 상대 시간(timeout)으로 표현한다. "1초 줘."
- gRPC 라이브러리(또는 언어의 context)는 이 값을 받자마자 **즉시 절대 시각으로 변환**한다. `deadline = now() + 1s`.
- 이후의 모든 내부 동작, 즉 타이머 설정, 남은 시간 계산, 헤더 인코딩, 하위 호출 전파는 이 절대 시각 `deadline`을 기준으로 한다.

호출 진입점에서 이 변환을 한 번만 해 두면, 이후에는 "지금부터 얼마 남았나?"라는 질문에 언제나 `deadline - now()`라는 같은 공식으로 답할 수 있다. 만약 내부에서 상대 시간을 들고 다닌다면 큐에서 대기한 시간, 직렬화에 쓴 시간, 첫 번째 하위 호출에 쓴 시간을 단계마다 빼 주어야 하고, 그 뺄셈 하나하나가 버그의 원인이 된다. 절대 시각을 쓰면 이 모든 뺄셈이 "현재 시각을 읽는다"는 연산 하나로 바뀐다.

```text
timeout 모델 (상대 시간을 들고 다님):
  남은시간_A = 1000ms
  → 큐 대기 30ms 후: 남은시간 = 970ms  (빼야 함)
  → 직렬화 5ms 후:   남은시간 = 965ms  (또 빼야 함)
  → B 호출 시 전달:  "965ms 줄게"        (계산 필요)
  → ... 매 단계 뺄셈, 매 단계 실수 가능

deadline 모델 (절대 시각을 들고 다님):
  deadline = T (= 진입 시각 + 1000ms)  ← 단 한 번 계산
  → 큐 대기든 직렬화든 무엇이든: 남은시간 = T - now()  (그때그때 읽기만)
  → B 호출 시 전달:  "T까지"             (그냥 전달)
  → ... 뺄셈 없음, 불변량 그대로
```

### 단조 시계(monotonic clock)와 벽시계(wall clock)

여기에는 미묘하지만 중요한 세부 사항이 하나 있다. "절대 시각"이라고 하면 벽시계(wall clock, 사람이 읽는 달력 시각)를 떠올리기 쉽다. 하지만 한 프로세스 안에서 deadline까지 남은 시간을 잴 때는 단조 시계(monotonic clock)를 쓰는 것이 맞다. 벽시계는 NTP 동기화나 윤초 때문에 갑자기 뒤로 점프할 수 있고, 그러면 `deadline - now()`가 음수가 되거나 지나치게 커지기 때문이다.

Go의 `time.Time`은 단조 시계 값을 함께 가지고 있어서 `time.Until(deadline)`을 안전하게 계산할 수 있다. 다만 **서로 다른 머신 사이**에서는 단조 시계를 공유할 수 없다(각 머신의 단조 시계는 부팅 이후 경과 시간 같은 임의의 기준점을 쓴다). 그래서 와이어를 건널 때는 절대 시각을 그대로 보내지 않고 "지금부터 남은 상대 시간"으로 환산해서 보낸다. 이것이 다음 절에서 다룰 `grpc-timeout` 헤더이다. gRPC는 이렇게 두 방식의 장점을 모두 취한다.

- **프로세스 내부**: 절대 시각(deadline)과 단조 시계로 안전하고 합성하기 쉽게 관리한다.
- **머신 사이(와이어)**: 상대 시간(`grpc-timeout`)으로 환산해 시계 동기화 문제를 피한다.

---

## 2. 와이어 표현: `grpc-timeout` 헤더

deadline이 한 머신에서 다른 머신으로 넘어갈 때 실제로 어떤 모습인지 살펴보자. gRPC는 요청의 HTTP/2 HEADERS 프레임 안에 `grpc-timeout`이라는 메타데이터 키로 마감을 담는다. (HTTP/2와 HEADERS 프레임 자체는 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 다룬다. 여기서는 그 위에 얹히는 gRPC의 규약에 집중한다.)

값의 형식은 **"정수 + 단위 문자 한 글자"** 다.

```text
grpc-timeout = 정수(최대 8자리) + 단위
단위:
  H = Hour   (시간)
  M = Minute (분)
  S = Second (초)
  m = millisecond (밀리초)
  u = microsecond (마이크로초)
  n = nanosecond  (나노초)
```

예시:

| 의도한 deadline | grpc-timeout 값 | 읽는 법 |
|----------------|-----------------|---------|
| 1초 | `1S` | 1 second |
| 100밀리초 | `100m` | 100 millisecond |
| 2분 30초 | `150S` | 150 second |
| 500마이크로초 | `500u` | 500 microsecond |
| 5분 | `5M` | 5 minute |

대소문자 구분에 주의해야 한다. 대문자 `M`은 분(Minute)이고 소문자 `m`은 밀리초(millisecond)이다. 대문자 `S`는 초(Second)이지만 소문자 `s`는 정의되어 있지 않다. 이 한 글자 차이가 1000배 또는 60000배의 오차를 만든다.

정수 부분은 **최대 8자리**로 제한된다. 따라서 `100000000S`(9자리)처럼 쓸 수 없고, 더 큰 값이 필요하면 단위를 키워야 한다. 예를 들어 아주 긴 마감은 초 대신 분이나 시간 단위로 표현한다. 이 8자리 제약은 헤더를 짧게 유지하기 위한 것이다.

### 홉마다 다시 환산한다: 와이어로 나갈 때는 "지금 남은 시간"

가장 중요한 동작 원리는 다음과 같다. **클라이언트는 자신이 가진 절대 deadline에서 "지금 남은 시간"을 계산해 `grpc-timeout`으로 인코딩한다.** 이 인코딩은 RPC를 보낼 때마다, 즉 홉(hop)마다 새로 일어난다.

```text
클라이언트 내부:
  deadline = T  (절대 시각)
  RPC를 보내려는 순간 now() = t0
  남은시간 = T - t0 = 940ms 라고 하자
  → grpc-timeout: 940m  로 인코딩해서 HEADERS에 실음

서버가 다시 하위 호출 C를 부를 때:
  서버는 받은 940m 를 자기 시각 기준 deadline 으로 복원: T' = now_server() + 940ms
  ... 서버가 일을 좀 하고, C 를 부르려는 순간 now() = t1
  남은시간 = T' - t1 = 880ms
  → grpc-timeout: 880m  로 다시 인코딩
```

이렇게 하면 홉을 지날 때마다 남은 시간이 줄어드는 것이 자연스럽게 표현된다. A→B 사이의 네트워크 지연과 B가 처리에 쓴 시간만큼 줄어든 값이 B→C로 전달된다. 내부에서는 절대 시각이라는 불변량을 유지하고, 와이어로 내보낼 때만 "지금 남은 시간"으로 바꿔(project) 보내는 것이다.

### 실제 바이트로 디코딩해 보기

`grpc-timeout: 1S`가 HTTP/2 HEADERS 프레임 안에서 어떻게 인코딩되는지 직접 따라가 보자. HTTP/2는 헤더를 HPACK으로 압축한다. `grpc-timeout`은 정적 테이블에 없는 헤더 이름이므로, 처음 나올 때는 보통 "이름과 값을 모두 리터럴로 보내고 동적 테이블에 추가"하는 형태(0x40 패턴)로 보낸다.

```text
HPACK: Literal Header Field with Incremental Indexing — New Name
  첫 바이트: 0x40
    0b01000000 = 인덱스 0 (새 이름)을 의미하는 패턴

  이어서 이름 길이와 이름 문자열:
    0x0C            = 길이 12, H-bit(허프만)=0  → "grpc-timeout"는 12바이트
    67 72 70 63 2d 74 69 6d 65 6f 75 74
      g  r  p  c  -  t  i  m  e  o  u  t

  이어서 값 길이와 값 문자열:
    0x02            = 길이 2, H-bit=0  → "1S"는 2바이트
    31 53
     1  S
```

전체 바이트열(허프만 미적용 가정):

```text
40 0C 67 72 70 63 2d 74 69 6d 65 6f 75 74 02 31 53
└┬┘ └┬┘ └──────────── "grpc-timeout" ────────────┘ └┬┘ └─┬─┘
 │   │                                              │   "1S"
 │   이름 길이=12                                    값 길이=2
 리터럴 헤더(증분 인덱싱, 새 이름)
```

두 번째 요청부터는 헤더 이름 `grpc-timeout`이 동적 테이블에 들어가 있으므로, 이름은 인덱스로 참조하고 값만 리터럴로 보내서 더 짧아진다. 핵심은 `grpc-timeout`의 값이 사람이 읽을 수 있는 ASCII 문자열 `"1S"` 그대로 들어간다는 점이다. 와이어 위에서 deadline은 복잡한 바이너리가 아니라 눈으로 읽을 수 있는 짧은 텍스트이다.

한 가지 짚어 둘 점이 있다. `grpc-timeout`의 값(`"1S"`, `"940m"` …)은 요청마다 달라진다. 그래서 이름과 값 쌍 전체를 동적 테이블에 넣어도(증분 인덱싱) 다음 요청에서 재사용되지 못하고 테이블 항목만 계속 바뀐다(dynamic table churn). 이 때문에 동적 테이블 효율을 신경 쓰는 인코더는 **이름만 한 번 인덱싱하고 값은 인덱싱 없이(Literal Header Field without Indexing, `0x00` 패턴)** 보내는 방식을 택하기도 한다.

위 예시는 "이름과 값을 모두 리터럴로 보낸다"는 점을 한눈에 보여 주려고 증분 인덱싱(`0x40`) 형태로 그린 것이다. 실제로 어떤 표현을 고를지는 HPACK 인코더 구현마다 다르다. 어느 쪽이든 와이어에 실리는 값 자체는 같은 ASCII 문자열이다.

`grpcurl`이나 디버그 로그로 헤더를 살펴보면 이 값을 직접 확인할 수 있다([[14 - 관찰성과 디버깅 - Reflection grpcurl]] 참고).

```bash
# 서버 측에서 받은 메타데이터를 로깅하도록 인터셉터를 걸어 두면
# grpc-timeout: 1S 같은 형태로 관측된다.
# (인터셉터로 메타데이터를 보는 방법은 08장 참고)
```

### deadline이 없으면?

`grpc-timeout` 헤더를 아예 보내지 않으면, 서버 입장에서 그 RPC는 **마감이 없는(무한대) 호출**이 된다. 이것이 바로 피하려는 상태이다. 클라이언트가 deadline을 명시적으로 설정하지 않으면 많은 언어 구현에서 기본값이 "무한대"이므로 헤더가 빠지고, 서버는 끝없이 기다려도 되는 것으로 여긴다. 이 위험은 뒤의 안티패턴 절에서 다시 다룬다.

---

## 3. deadline 전파(propagation): 하나의 마감을 모두가 공유한다

이제 deadline의 가장 강력한 기능인 전파(propagation)를 본격적으로 살펴보자. gRPC가 deadline을 절대 시각으로 다루기로 하면서 얻는 가장 큰 이점이 바로 이것이다.

### 시나리오: A → B → C 호출 체인

사용자가 "주문 상세 보기"를 요청한다. 이 요청은 게이트웨이 A로 들어오고, A는 주문 서비스 B를 호출하며, B는 다시 재고 서비스 C와 사용자 서비스를 호출한다. 사용자는 1초 안에 화면이 뜨기를 기대한다.

```text
   사용자 ──"1초 안에"──▶  A (게이트웨이)
                            │  deadline = T = now + 1000ms 로 고정
                            │  grpc-timeout: 1S 로 B 호출
                            ▼
                          B (주문 서비스)
                            │  받은 1S 를 자기 시각 deadline T'로 복원
                            │  남은 만큼 grpc-timeout 으로 C 호출
                            ▼
                          C (재고 서비스)
                            │  받은 남은시간으로 또 deadline 복원
                            ▼
                          DB / 다른 서비스 ...
```

deadline 전파란 **A가 정한 마감 T가 B와 C에도 그대로 적용된다**는 뜻이다. B와 C는 "얼마나 기다려야 하지?"를 각자 따로 정하지 않고, A에서부터 전달된 마감을 물려받는다.

### 왜 중요한가: 이미 지난 마감을 넘겨 일하는 낭비를 막는다

deadline 전파가 없다고 가정해 보자. A는 1초 타임아웃으로 B를 호출하는데, B는 자기 기준대로 C를 5초 타임아웃으로 호출한다. 그러면 다음과 같은 상황이 벌어진다.

```text
t=0ms     A가 B 호출 시작 (A의 마감: 1000ms)
t=1000ms  A 입장에서 마감 초과! A는 B에게서 손을 떼고
          사용자에게 DEADLINE_EXCEEDED 반환. 끝.
t=1000ms~ 그런데 B는 아직도 C의 응답을 기다리는 중 (B의 마감: 5000ms)
t=3000ms  C가 드디어 응답. B가 결과를 조립.
t=3200ms  B가 A에게 응답을 보냄... 그러나 A는 이미 떠났다.
          이 응답은 갈 곳이 없다. 버려진다.
          그 사이 C도, B도 2초 넘게 헛수고를 했다.
```

여기서 `t=1000ms` 이후에 B와 C가 한 일은 모두 **버려질 것이 이미 확정된 작업**이다. 그 결과를 받을 쪽이 없다. 그런데도 B와 C는 CPU와 메모리, DB 커넥션, 스레드를 계속 점유한다. 부하가 높을수록 이런 "좀비 작업(zombie work)"이 쌓여서 시스템이 의미 없는 일로 가득 차고, 정작 유효한 요청을 처리할 자원이 바닥난다.

deadline 전파는 이 낭비를 근본적으로 막는다. A의 마감 T가 C까지 전달되므로, A가 호출을 포기하는 순간 C도 같은 마감에 걸려 자동으로 멈춘다. 결과를 기다리는 쪽이 없는 일은 아무도 하지 않게 된다.

> [!note] 한 줄 요약
> **"호출자가 이미 포기한 일을 피호출자가 계속 붙들고 있지 않게 한다."** 이것이 deadline 전파의 본질이다.

### budget이 소진되는 과정

A가 가진 1000ms 예산이 호출 체인을 따라 어떻게 소진되는지 그림으로 나타내 보자.

```text
A의 총 예산: |████████████████████████████████| 1000ms

A→B 네트워크(20ms):
  소진      |█|
  남은 예산  ▶ 980ms 가 grpc-timeout 으로 B 에 도착

B의 자체 처리(50ms):
  소진      |██|
  남은 예산  ▶ 930ms

B→C 네트워크(15ms):
  소진      |█|
  남은 예산  ▶ 915ms 가 grpc-timeout 으로 C 에 도착

C의 처리(40ms) + DB(...):
  소진      |██|......
  ...

만약 어딘가에서 합계가 1000ms 를 넘으려 하면:
  → 그 순간 마감 T 에 도달 → 그 지점의 호출이 DEADLINE_EXCEEDED 로 끊김
  → 그 신호가 사슬을 거슬러 올라가며 모두를 멈춤
```

budget(예산)이라는 말이 잘 들어맞는다. A는 1000ms라는 예산을 가지고 시작하고, 네트워크 지연과 각 서비스의 처리 시간이 그 예산에서 빠져나간다. 호출 체인의 어느 지점에서든 예산이 0이 되면 그 자리에서 바로 `DEADLINE_EXCEEDED`가 발생한다. 누구도 예산을 초과해 쓸 수 없다.

### 전파는 자동인가: 컨텍스트를 넘겨야 자동으로 된다

여기에 결정적인 함정이 있다. deadline 전파는 저절로 일어나지 않는다. **서버 코드가 들어온 컨텍스트(context)를 하위 호출에 명시적으로 넘겨줄 때만** 일어난다. Go라면 핸들러가 받은 `ctx`를, Java라면 `Context.current()`를 하위 gRPC 클라이언트 호출에 전달해야 한다.

서버 핸들러가 받은 컨텍스트를 무시하고 `context.Background()`(마감이 없는 빈 컨텍스트)로 하위 호출을 새로 만들면, 전파는 그 지점에서 끊긴다. B가 A의 마감을 물려받았더라도 B→C 호출에 그 마감을 싣지 않으면 C는 다시 무한대 마감으로 돌아간다. 그래서 다음 절의 언어별 모델에서 "컨텍스트를 넘겨라"라는 규칙을 그토록 강조한다.

---

## 4. 취소(cancellation) 메커니즘: 멈추라는 신호는 어떻게 전달되나

deadline 초과는 취소의 한 특수한 경우이다. 그래서 먼저 일반적인 취소가 와이어 위에서 어떻게 동작하는지 보고, 그다음에 deadline 초과가 이 메커니즘 안에서 어떻게 처리되는지 살펴보자.

### 클라이언트가 취소하면: HTTP/2 RST_STREAM(CANCEL)

클라이언트가 진행 중인 RPC를 취소하는 상황을 생각해 보자. 사용자가 브라우저 탭을 닫았을 수도 있고, 상위 요청이 이미 끝났을 수도 있고, 다른 응답이 먼저 와서 이 호출이 필요 없어졌을 수도 있다. 이때 gRPC 클라이언트는 그 RPC에 해당하는 **HTTP/2 스트림을 RST_STREAM 프레임으로 닫는다.**

HTTP/2에서 각 RPC는 스트림(stream) 하나에 대응한다(멀티플렉싱(multiplexing)으로 한 커넥션 위에 여러 스트림이 동시에 흐른다. 4장 HTTP/2 깊이 보기 참고). 스트림을 중간에 끊으려면 RST_STREAM 프레임을 보낸다. gRPC 취소에서는 에러 코드로 `CANCEL`(HTTP/2 에러 코드 0x8)을 담아 보낸다.

```text
RST_STREAM 프레임 구조 (HTTP/2, RFC 7540 §6.4):
  +-----------------------------------------------+
  | Length (24)  = 4                              |   페이로드는 항상 4바이트
  +---------------+---------------+---------------+
  | Type (8)     = 0x3 (RST_STREAM)               |
  +---------------+-----------------------------+-+
  | Flags (8)    = 0x0 (없음)                     |
  +-+-------------------------------------------+-+
  |R| Stream Identifier (31)  = 끊으려는 스트림 ID  |
  +-+-------------------------------------------+-+
  | Error Code (32) = 0x8 (CANCEL)                |   ← gRPC 취소의 신호
  +-----------------------------------------------+

대표적 HTTP/2 에러 코드:
  0x0 NO_ERROR
  0x2 INTERNAL_ERROR
  0x8 CANCEL          ← 클라이언트가 의도적으로 취소
```

"이 호출은 이제 필요 없으니 그만둬"라는 메시지는 이 4바이트짜리 작은 프레임이 전부이다.

### 서버 쪽에서 일어나는 일: 컨텍스트가 취소된다

서버의 HTTP/2 전송 계층이 RST_STREAM(CANCEL)을 받으면, 그 스트림에 연결된 gRPC 호출의 컨텍스트(서버 측 context)를 **취소 상태로 전환**한다. 그 순간 다음 일이 일어난다.

- Go에서는 그 호출의 `ctx.Done()` 채널이 닫히고, `ctx.Err()`가 `context.Canceled`를 반환한다.
- Java에서는 그 호출의 `Context`가 취소되어 `Context.current().isCancelled()`가 `true`가 되고, 등록된 취소 리스너가 호출된다.

여기서 가장 중요한 책임이 나온다. **서버는 이 신호를 보고 즉시 일을 멈춰야 한다.** gRPC 런타임은 핸들러 함수를 강제로 중단시키지 않는다(스레드를 강제로 종료하는 것은 위험하기 때문이다). 런타임은 "취소됨"이라는 신호를 컨텍스트에 실어 줄 뿐이고, 그 신호를 보고 빠져나오는 것은 **핸들러 코드의 협조(cooperation)** 에 달려 있다. 핸들러가 컨텍스트를 확인하지 않으면 신호가 와도 계산을 끝까지 계속한다. 이것이 뒤에서 다룰 가장 흔한 안티패턴이다.

```text
[클라이언트]                         [서버]
    │
    │  ... RPC 진행 중 ...
    │
 사용자가 취소 / 상위 마감 도달
    │
    ├── RST_STREAM(CANCEL) ─────────▶ HTTP/2 전송 계층이 수신
    │                                     │
    │                                 호출의 ctx 를 취소 상태로 전환
    │                                     │
    │                                 ctx.Done() 닫힘 / isCancelled()=true
    │                                     │
    │                                 (핸들러가 ctx 를 확인하면)
    │                                 진행 중인 DB 쿼리·하위 호출도
    │                                 같은 ctx 를 타고 함께 취소됨
    │                                     │
    │                                 핸들러가 일찍 return → 자원 회수
```

### deadline 초과는 자동 취소이고, 결과는 DEADLINE_EXCEEDED이다

이제 deadline 초과가 이 흐름의 어디에 들어가는지 보자. deadline은 **"미래의 특정 시각에 자동으로 작동하는 취소 타이머"** 다.

- 클라이언트가 deadline을 설정하면 클라이언트 내부에 타이머가 걸린다. 그 시각이 되면 클라이언트는 RPC를 스스로 취소한다(서버로 RST_STREAM 전송). 이때 클라이언트가 호출자에게 돌려주는 상태 코드는 `DEADLINE_EXCEEDED`(코드 4)이다.
- 서버도 전파된 deadline을 알고 있으므로, 그 시각이 되면 서버 측 컨텍스트를 자동으로 취소한다(`ctx.Err()`가 `context.DeadlineExceeded`). 그래서 RST_STREAM이 도착하기 전이라도 서버는 스스로 마감을 알고 멈출 수 있다.

즉 deadline 초과는 "타이머가 일으킨 취소"이고, 일반 취소는 "사람이나 상위 요청이 일으킨 취소"이다. 메커니즘은 같고, 호출자에게 보고되는 상태 코드만 다르다.

| 발동 원인 | 클라이언트가 받는 상태 코드 | 코드 번호 |
|-----------|---------------------------|-----------|
| 명시적 취소(사용자/상위 종료) | `CANCELLED` | 1 |
| deadline 시각 도달 | `DEADLINE_EXCEEDED` | 4 |

(상태 코드 체계 전반은 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에서 다룬다. 여기서는 이 두 가지만 짚는다.)

> [!note] 정신 모델
> deadline은 **미리 예약해 둔 취소**이다. RST_STREAM, 컨텍스트 취소, 하위 호출로의 전파 같은 취소의 전달 경로(plumbing)를 그대로 재사용하고, 취소를 일으키는 계기만 시계로 바뀐 것이다.

---

## 5. 언어별 모델: Go의 context, Java의 Context/Deadline

개념은 같지만, 언어마다 그 언어의 관용구에 맞게 구현한다. 가장 널리 쓰이는 두 모델을 살펴본다.

### Go: `context.Context`

Go에서는 `context.Context` 하나가 deadline, 취소, 요청 범위 값(request-scoped value)을 모두 담는다. gRPC뿐 아니라 표준 라이브러리 전반(`net/http`, `database/sql` 등)이 이 인터페이스를 함께 쓰기 때문에, deadline과 취소가 언어 생태계 전체에 자연스럽게 전달된다. Go에서 deadline 전파가 특히 매끄러운 이유가 여기에 있다.

핵심 API는 다음과 같다.

```go
// 상대 시간으로 마감을 건다 (내부적으로 절대 시각으로 변환됨)
ctx, cancel := context.WithTimeout(parent, 1*time.Second)
defer cancel() // 반드시 호출해 타이머/리소스 누수를 막는다

// 절대 시각으로 직접 마감을 건다
ctx, cancel := context.WithDeadline(parent, time.Now().Add(time.Second))
defer cancel()

// 마감 없이 수동 취소만 가능한 컨텍스트
ctx, cancel := context.WithCancel(parent)
defer cancel()

// 취소/마감을 관찰하는 두 가지 방법
<-ctx.Done()      // 취소되면 닫히는 채널 (select 로 대기)
err := ctx.Err()  // nil / context.Canceled / context.DeadlineExceeded
```

`context.WithTimeout`이 내부에서 `WithDeadline(parent, now+timeout)`을 호출한다는 점에 주목하자. 1절에서 말한 "프로그래머는 timeout으로 쓰지만 라이브러리는 즉시 deadline으로 변환한다"가 실제 코드로 구현된 모습이다.

**클라이언트 측: deadline 설정**

```go
func GetOrder(client orderpb.OrderServiceClient, id string) (*orderpb.Order, error) {
    // 이 호출 전체의 마감을 800ms 로 건다
    ctx, cancel := context.WithTimeout(context.Background(), 800*time.Millisecond)
    defer cancel()

    resp, err := client.GetOrder(ctx, &orderpb.GetOrderRequest{OrderId: id})
    if err != nil {
        // 마감 초과면 status.Code(err) == codes.DeadlineExceeded
        if status.Code(err) == codes.DeadlineExceeded {
            log.Printf("주문 조회가 마감을 넘겼습니다: %s", id)
        }
        return nil, err
    }
    return resp.GetOrder(), nil
}
```

gRPC-Go는 이 `ctx`에 설정된 deadline을 읽어 `grpc-timeout` 헤더로 인코딩해 보낸다. 프로그래머가 헤더를 직접 만들 필요는 없다. `context`에 deadline을 설정하기만 하면 와이어 인코딩과 전파가 함께 이루어진다.

**서버 측: 컨텍스트를 확인해 조기 종료하고 하위 호출로 전파하기**

```go
func (s *orderServer) GetOrder(ctx context.Context, req *orderpb.GetOrderRequest) (*orderpb.GetOrderResponse, error) {
    // (1) 비싼 작업을 시작하기 전에 이미 취소됐는지 값싸게 확인
    if err := ctx.Err(); err != nil {
        return nil, status.FromContextError(err).Err()
    }

    // (2) 하위 호출에는 받은 ctx 를 "그대로" 넘긴다.
    //     → A 가 정한 마감이 inventory 서비스까지 전파된다.
    inv, err := s.inventoryClient.CheckStock(ctx, &invpb.CheckStockRequest{
        OrderId: req.GetOrderId(),
    })
    if err != nil {
        return nil, err // DEADLINE_EXCEEDED 도 자연스럽게 위로 전달됨
    }

    // (3) 긴 루프나 단계 사이에서도 주기적으로 취소를 확인한다.
    items, err := s.loadItems(ctx, req.GetOrderId())
    if err != nil {
        return nil, err
    }

    return &orderpb.GetOrderResponse{
        Order: assemble(req.GetOrderId(), inv, items),
    }, nil
}

// 긴 작업 안에서 취소를 협조적으로 관찰하는 예
func (s *orderServer) loadItems(ctx context.Context, orderID string) ([]*orderpb.Item, error) {
    var items []*orderpb.Item
    for _, shard := range s.shards {
        // select 로 "취소 신호"와 "다음 작업"을 함께 기다린다
        select {
        case <-ctx.Done():
            // 마감/취소가 오면 즉시 손을 뗀다 → 좀비 작업 방지
            return nil, status.FromContextError(ctx.Err()).Err()
        default:
        }

        // DB 호출에도 같은 ctx 를 넘겨 쿼리 자체가 취소 가능하게 한다
        rows, err := s.db.QueryContext(ctx, shard.query, orderID)
        if err != nil {
            return nil, err
        }
        items = append(items, scan(rows)...)
    }
    return items, nil
}
```

이 예제에서 가장 중요한 부분은 `s.inventoryClient.CheckStock(ctx, ...)`와 `s.db.QueryContext(ctx, ...)`에서 **받은 `ctx`를 그대로 전달**하는 곳이다. 여기서 `context.Background()`를 새로 만들어 넘겼다면 전파가 끊겼을 것이다. "서버 핸들러는 받은 컨텍스트를 자신의 모든 하위 IO와 호출에 넘겨야 한다"는 규칙이 바로 이것이다.

**안티패턴: 전파 끊기**

```go
// 나쁜 예: 받은 ctx 를 버리고 새 빈 컨텍스트로 하위 호출
func (s *orderServer) GetOrderBad(ctx context.Context, req *orderpb.GetOrderRequest) (*orderpb.GetOrderResponse, error) {
    // context.Background() 는 마감도 취소도 없다 → A 의 마감이 여기서 증발
    inv, _ := s.inventoryClient.CheckStock(context.Background(), &invpb.CheckStockRequest{ /* ... */ })
    // A 가 이미 포기했어도 이 하위 호출은 끝까지 살아서 좀비 작업이 된다
    _ = inv
    return nil, nil
}
```

### Java: `Context`, `Deadline`, `CancellableContext`

Java의 gRPC는 `io.grpc.Context`로 요청 범위 정보를, `io.grpc.Deadline`으로 마감을 표현한다. Go의 `context.Context`는 인자로 명시적으로 전달되지만, Java의 `Context`는 기본적으로 현재 컨텍스트를 **스레드 로컬(thread-local)** 에 보관한다(`Context.current()`). 다만 스레드 경계를 넘을 때(스레드 풀, 비동기)는 컨텍스트를 명시적으로 전파해야 한다.

**클라이언트 측: deadline 설정**

```java
// 스텁에 마감을 건다. withDeadlineAfter 는 상대 시간을 받아 내부 Deadline 으로 변환
OrderServiceGrpc.OrderServiceBlockingStub stub =
    OrderServiceGrpc.newBlockingStub(channel)
        .withDeadlineAfter(800, TimeUnit.MILLISECONDS);

try {
    GetOrderResponse resp = stub.getOrder(
        GetOrderRequest.newBuilder().setOrderId(id).build());
    return resp.getOrder();
} catch (StatusRuntimeException e) {
    if (e.getStatus().getCode() == Status.Code.DEADLINE_EXCEEDED) {
        log.warn("주문 조회가 마감을 넘겼습니다: {}", id);
    }
    throw e;
}
```

`withDeadlineAfter(800, MILLISECONDS)`는 "지금부터 800ms"라는 상대 시간을 받지만, 내부에서는 `Deadline.after(800, MILLISECONDS)`로 절대 시각(단조 시계 기준)을 계산해 둔다.

여기에는 함정이 하나 있다. **스텁에 설정한 deadline은 그 스텁 인스턴스에 고정**되므로, 같은 스텁을 재사용해 여러 번 호출하면 모든 호출이 같은 절대 마감을 공유한다. 호출할 때마다 새 마감이 필요하면 호출 직전에 `withDeadlineAfter`로 새 스텁을 만들어야 한다.

**서버 측: 취소 확인과 전파**

```java
@Override
public void getOrder(GetOrderRequest req, StreamObserver<GetOrderResponse> responseObserver) {
    Context ctx = Context.current();

    // (1) 이미 취소/마감됐는지 확인
    if (ctx.isCancelled()) {
        responseObserver.onError(
            Status.CANCELLED.withDescription("이미 취소됨").asRuntimeException());
        return;
    }

    // (2) 취소 리스너 등록 — 취소가 오면 진행 중 작업을 정리.
    //     addListener 는 Context 자체의 메서드다. 현재 컨텍스트가 직접 취소
    //     가능하지 않더라도 가장 가까운 '취소 가능한 조상(cancellable ancestor)'에
    //     자동으로 매달리므로, Context.CancellableContext 로 캐스팅하지 않는다.
    //     (인터셉터가 ctx.withValue(...) 로 값을 덧씌우면 현재 컨텍스트는
    //      CancellableContext 인스턴스가 아니어서, 캐스팅 시 ClassCastException 이 난다.)
    ctx.addListener(c -> {
        // 마감 도달이든 클라 취소든 여기로 통지된다 (CancellationListener.cancelled)
        cleanupInFlightWork(req.getOrderId());
    }, MoreExecutors.directExecutor());

    // (3) 하위 호출에는 현재 Context 의 마감이 "자동으로" 흐른다.
    //     grpc-java 의 ClientCallImpl 은 나가는 호출의 실효 마감을
    //     min(스텁에 박힌 마감, Context.current().getDeadline()) 로 계산하므로,
    //     같은 스레드에서 부르는 한 아래처럼 명시적으로 박지 않아도 서버 호출의
    //     마감이 grpc-timeout 으로 재환산되어 전달된다. (Go 가 ctx 를 인자로
    //     넘겨야 하는 것과 달리, Java 는 스레드 로컬 Context 로 전파한다.)
    InventoryServiceGrpc.InventoryServiceBlockingStub invStub =
        InventoryServiceGrpc.newBlockingStub(inventoryChannel);

    // withDeadline 은 '필수'가 아니라 '명시'다. 같은 마감을 코드에 드러내거나,
    // 더 짧게 조이고 싶을 때(예: 하위 호출에만 200ms) 사용한다.
    Deadline deadline = ctx.getDeadline();
    if (deadline != null) {
        invStub = invStub.withDeadline(deadline);
    }
    CheckStockResponse inv = invStub.checkStock(
        CheckStockRequest.newBuilder().setOrderId(req.getOrderId()).build());

    // (4) 긴 루프에서 주기적으로 취소 확인
    for (Shard shard : shards) {
        if (Context.current().isCancelled()) {
            responseObserver.onError(Status.CANCELLED.asRuntimeException());
            return;
        }
        // ... 작업 ...
    }

    responseObserver.onNext(assemble(req.getOrderId(), inv));
    responseObserver.onCompleted();
}
```

Java에서는 **스레드 풀로 작업을 넘길 때 컨텍스트가 함께 넘어가지 않는다**는 점에 주의해야 한다. `Context`는 스레드 로컬에 묶여 있으므로, 다른 스레드에서 실행되는 작업에 현재 컨텍스트를 전파하려면 `Context.current().wrap(runnable)`이나 `ctx.run(() -> ...)`로 감싸야 한다.

```java
// 스레드 풀에 작업을 넘길 때 컨텍스트(마감 포함)를 함께 전파
Context ctx = Context.current();
executor.submit(ctx.wrap(() -> {
    // 이 안에서는 Context.current() 가 바깥의 마감을 그대로 본다
    doSubtask();
}));
```

### 참고: Python

Python의 gRPC도 같은 개념을 제공한다. 클라이언트는 호출할 때 `timeout=` 인자(상대 시간, 초 단위)를 넘기고, 서버 핸들러는 `context.is_active()`, `context.time_remaining()`, `context.add_callback(...)`으로 취소와 마감을 확인한다.

```python
# 클라이언트
try:
    resp = stub.GetOrder(order_pb2.GetOrderRequest(order_id=oid), timeout=0.8)
except grpc.RpcError as e:
    if e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
        log.warning("주문 조회 마감 초과: %s", oid)
    raise

# 서버 핸들러
def GetOrder(self, request, context):
    if not context.is_active():
        return order_pb2.GetOrderResponse()  # 이미 취소됨
    remaining = context.time_remaining()      # 남은 초 (없으면 None)
    # 하위 호출에 남은 시간을 다시 timeout 으로 넘겨 전파
    inv = self.inventory_stub.CheckStock(
        inv_pb2.CheckStockRequest(order_id=request.order_id),
        timeout=remaining)
    # ...
```

세 언어에 공통으로 적용되는 규칙은 하나이다. **"받은 마감과 취소 신호를 유지하다가, 모든 하위 호출과 IO에 그대로 넘겨라."** 이 규칙을 어기면 그 순간 전파가 끊기고 좀비 작업이 생긴다.

---

## 6. 스트리밍에서의 취소: 양방향성

지금까지 본 예는 주로 단항(Unary) 호출이었다. 스트리밍에서는 취소가 더 미묘하다. 스트리밍의 종류와 기본 동작은 [[05 - 통신의 4가지 방식 - Unary와 Streaming]]에서 다루므로, 여기서는 deadline과 취소가 스트림에 어떻게 작용하는지에 집중한다.

### 스트림 전체에 하나의 deadline

서버 스트리밍이든 클라이언트 스트리밍이든 양방향(bidi)이든, deadline은 **스트림 전체의 수명**에 적용된다. "메시지 하나당" 마감이 아니라 "이 스트림을 여는 순간부터 닫힐 때까지 모두 합쳐 N초"라는 뜻이다. 따라서 오래 열려 있어야 하는 스트림(예: 실시간 알림 구독, 채팅 채널)에는 짧은 deadline을 걸면 안 된다. 정상적으로 길게 열려 있어야 할 스트림이 마감에 걸려 `DEADLINE_EXCEEDED`로 끊기기 때문이다.

이런 장수명 스트림에서는 deadline 대신 keepalive로 끊어진 커넥션을 감지하는 것이 맞다. deadline과 keepalive의 차이는 9절과 [[13 - 안정성 - Retry Health Check Keepalive]]에서 다시 살펴본다.

### 양방향성: 한쪽이 취소하면 스트림 전체가 끝난다

양방향 스트리밍에서는 클라이언트와 서버가 같은 HTTP/2 스트림 위에서 양방향으로 메시지를 주고받는다. 이 스트림에 대한 취소(RST_STREAM)는 **방향과 관계없이 스트림 전체를 종료**한다.

```text
[클라이언트]  ◀══════ 양방향 스트림 ══════▶  [서버]
                   (하나의 HTTP/2 스트림)

클라이언트가 취소:
  → RST_STREAM(CANCEL) 전송
  → 서버의 수신 측 ctx 취소, 서버가 보내려던 다음 메시지도 무의미해짐
  → 양쪽 모두 스트림 종료. 서버는 ctx.Done() 으로 알아채고 송신 루프 중단

서버가 에러로 스트림 종료:
  → 서버가 트레일러(상태)와 함께 스트림 half-close/RST 처리
  → 클라이언트의 수신 측이 종료를 감지, 보내려던 다음 메시지는 갈 곳이 없음
```

Go의 양방향 스트림 서버 핸들러는 보통 송신 고루틴과 수신 고루틴을 함께 실행하는데, 두 고루틴 모두 `stream.Context()`를 확인해야 한다.

```go
func (s *chatServer) Chat(stream chatpb.ChatService_ChatServer) error {
    ctx := stream.Context() // 이 스트림의 컨텍스트 (취소/마감 관찰점)

    // 수신 루프
    for {
        // Recv 는 스트림이 취소되면 에러를 돌려준다
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil // 클라이언트가 정상적으로 송신 종료
        }
        if err != nil {
            // 취소/마감이면 status.Code 가 Canceled / DeadlineExceeded
            return err
        }

        // 보낼 때도 취소를 함께 감시
        select {
        case <-ctx.Done():
            return status.FromContextError(ctx.Err()).Err()
        default:
        }

        if err := stream.Send(&chatpb.ChatMessage{
            Text: process(msg.GetText()),
        }); err != nil {
            return err // 클라이언트가 떠났으면 Send 도 실패
        }
    }
}
```

핵심은 **양방향 스트림에서 취소가 "둘 사이의 대화 자체"를 끝낸다**는 점이다. 전화 통화에 비유하면, 한쪽이 "그만"이라고 말하는 순간 전화가 끊기고 상대가 하려던 말도 전달되지 않는다. 그래서 양쪽 모두 메시지를 보내기 전에 스트림이 아직 유효한지 확인하는 협조가 필요하다.

---

## 7. 왜 모든 RPC에 deadline을 설정해야 하는가

여기까지 읽었다면 답은 거의 분명하지만, 명시적으로 정리해 두자. **모든 RPC에 deadline을 걸어야 한다.** 예외는 의도적으로 오래 유지하는 스트림 정도뿐이다.

### deadline이 없는 호출의 세 가지 위험

**(1) 무한 대기로 인한 스레드/고루틴 고갈.** 마감이 없으면 응답도 에러도 오지 않는 호출이 끝없이 대기할 수 있다. 그 호출을 기다리는 스레드(또는 고루틴, 커넥션)는 회수되지 않는다. 이런 호출이 동시에 쌓이면 스레드 풀과 커넥션 풀이 바닥나고, 같은 자원을 쓰는 정상 요청까지 처리되지 못한다.

```text
deadline 없음 + 느린 하위 서비스:
  요청1 ──▶ [매달림] ──┐
  요청2 ──▶ [매달림] ──┤  스레드/커넥션이 회수 안 됨
  요청3 ──▶ [매달림] ──┤  → 풀 고갈
  요청4 ──▶ (풀 없음, 즉시 실패 또는 큐에서 무한 대기)
  ...
  결국 정상 요청까지 처리 불가 → 서비스 전체 다운
```

**(2) 리소스 고갈과 메모리 누수.** 멈춰 있는 호출은 스레드뿐 아니라 그 호출에 묶인 버퍼, 요청 객체, 부분 응답, 잠금(lock)까지 붙들고 있다. 좀비 작업이 쌓일수록 메모리 사용량이 늘어나고, 잠금을 쥔 채 멈춰 있으면 다른 작업까지 막힌다.

**(3) 장애 전파(연쇄 장애).** 가장 위험한 시나리오이다. deadline이 없는 상태에서 호출 체인 맨 끝의 C가 느려지면, B는 C를 무한정 기다리며 자기 자원을 소진한다. B가 느려지면 A도 B를 무한정 기다리며 자기 자원을 소진한다. 마감이라는 방화벽이 없으니 한 서비스의 지연이 상류로 번져 시스템 전체를 무너뜨린다.

deadline은 이 전파를 끊는 방화벽이다. C가 마감을 넘기면 B는 즉시 실패로 처리하고 자원을 회수하므로, B까지 함께 장애에 빠지지 않는다.

> [!note] 한 줄 요약
> **deadline은 분산 시스템에서 회로 차단기(circuit breaker)의 가장 원초적인 형태이다.** 지연이 무한 대기로, 무한 대기가 자원 고갈로, 자원 고갈이 연쇄 장애로 번지지 않게 막는다.

### 기본값에 기대지 말아야 한다

많은 gRPC 구현에서 deadline의 기본값은 "무한대"이다. 즉 **아무것도 설정하지 않으면 가장 위험한 상태**가 된다. 기본값이 안전하지 않다는 점이 함정이다. 그래서 팀 차원에서 "deadline이 없는 호출은 코드 리뷰에서 막는다", "클라이언트 인터셉터에서 deadline이 없으면 합리적인 기본값을 강제로 넣는다"([[08 - 메타데이터와 인터셉터]]) 같은 규칙을 정해 두어야 한다.

```go
// 클라이언트 인터셉터로 "마감 없는 호출"에 기본 마감을 강제 주입하는 예
func enforceDeadline(defaultTimeout time.Duration) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply any,
        cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        if _, ok := ctx.Deadline(); !ok {
            // 마감이 없으면 기본값을 건다
            var cancel context.CancelFunc
            ctx, cancel = context.WithTimeout(ctx, defaultTimeout)
            defer cancel()
        }
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}
```

---

## 8. deadline 예산(budget) 설계

deadline을 걸어야 한다면, 다음 질문은 그 값을 얼마로 정하느냐이다. 이것이 deadline 예산(budget) 설계이다. 예산은 너무 빡빡해도 문제이고 너무 느슨해도 문제이다.

### 예산을 나누는 방법

A에게 전체 1000ms의 예산이 있다고 하자. 이 예산은 호출 체인에 있는 여러 소비자에게 나뉜다.

```text
A 의 총 예산: 1000ms
├─ A 자체 처리(직렬화/조립/로직):        ~50ms
├─ A→B 네트워크 왕복:                    ~20ms
├─ B 가 쓸 몫(B 자체 + B→C + C ...):     나머지
│   ├─ B 자체 처리:                       ~50ms
│   ├─ B→C 네트워크 왕복:                 ~15ms
│   └─ C 가 쓸 몫:                        나머지
└─ 안전 여유(jitter/GC/스케줄링 흔들림):  반드시 일부 남겨 둘 것
```

deadline 전파가 "남은 예산"을 하위로 자동으로 넘겨 주므로, 각 서비스가 "하위에 얼마를 줄까"를 직접 빼서 계산할 일은 줄어든다. 하지만 한 서비스가 여러 하위 호출을 순차로 또는 병렬로 실행한다면, 그 안에서 시간을 어떻게 배분할지는 여전히 설계해야 한다.

### 너무 빡빡할 때: 허위 DEADLINE_EXCEEDED

예산을 실제 처리 시간보다 짧게 잡으면, 시스템이 정상적으로 일하고 있는데도 마감에 걸려 실패한다. 이를 허위(false) `DEADLINE_EXCEEDED`라고 부르자.

- 정상적으로 처리되던 요청이 마감 초과로 끊기고, 사용자에게는 에러로 보인다.
- 끊긴 요청을 재시도하면 부하가 늘어 시스템이 더 느려지고, 그 결과 마감 초과가 더 많이 발생하는 악순환(retry storm)이 생긴다.
- 특히 꼬리 지연(tail latency, p99)을 무시하고 평균만 보고 예산을 잡으면, 평소에는 괜찮다가 부하가 조금만 올라가도 마감 초과가 한꺼번에 쏟아진다.

빡빡한 예산은 "조금만 느려져도 요청을 전부 끊어 버리는" 과민한 시스템을 만든다.

### 너무 느슨할 때: 좀비 작업과 늦은 실패

반대로 예산을 지나치게 넉넉하게(예: 30초) 잡으면 deadline이 사실상 무한대에 가까워지고, 원래 막으려던 문제가 다시 나타난다.

- 하위 서비스가 죽었거나 끝내 응답하지 않는데도 30초 동안 자원을 붙들고 기다린다. 이것이 좀비 작업이다.
- 사용자는 "안 되면 빨리 알려 주기라도 하지"라고 느낀다. 30초 뒤에 실패를 통보받는 것은 사용자에게 거의 쓸모가 없다.
- 빠르게 실패(fail-fast)하고 대체 경로(fallback)로 넘어갈 기회를 놓친다.

### 좋은 예산의 원칙

- **사용자가 체감하는 마감에서 출발한다.** "이 화면은 1초 안에 떠야 한다"가 최상위 예산이다. 내부에서 일어나는 모든 일이 이 예산 안에 들어가야 한다.
- **꼬리 지연에 여유를 더해 잡는다.** 평균이 아니라 p99/p999 같은 실제 분포의 꼬리에 약간의 여유(GC 멈춤, 스케줄링 지터)를 더한다.
- **상류의 마감을 하류보다 약간 길게 잡는다.** A의 마감이 B의 마감보다 약간 길어야, B가 마감 직전에 실패를 만들어 A에게 의미 있는 에러를 돌려줄 시간이 생긴다. A와 B의 마감이 같으면 B의 마감 초과 응답이 A에 도착하기 전에 A도 마감을 넘기므로, A는 자기 마감 초과만 보게 된다(B가 보낸 자세한 에러를 받지 못한다).
- **deadline 전파를 신뢰하되, 전파가 끊기지 않게 한다.** 5절의 규칙대로 컨텍스트를 넘기면, 단계마다 직접 예산을 빼서 계산하지 않아도 된다.

---

## 9. 다른 메커니즘과의 상호작용

### 재시도(retry)와 deadline

재시도는 deadline과 밀접하게 얽혀 있다. 핵심 규칙은 **재시도는 남은 예산 안에서만 한다**는 것이다. 전체 마감 T가 정해져 있으면, 1차 시도가 실패했을 때 재시도는 `T - now()`가 양수일 때만, 그리고 그 남은 시간 안에서만 할 수 있다. 마감이 이미 지났으면 재시도하지 않고 바로 `DEADLINE_EXCEEDED`로 끝낸다.

```text
전체 마감 T = 1000ms
  시도1: t=0 시작, t=300ms 에 실패(UNAVAILABLE)
  백오프 50ms 대기 → t=350ms
  남은 예산 = 1000 - 350 = 650ms > 0 → 시도2 가능 (이 650ms 안에서)
  시도2: t=350 시작, t=900ms 에 실패
  백오프 → t=1000ms 도달 → 남은 예산 0 → 더는 재시도 안 함 → DEADLINE_EXCEEDED
```

gRPC의 내장 재시도 정책은 이 "남은 deadline 안에서만 재시도한다"는 규칙을 자동으로 지킨다. 또한 deadline 초과(`DEADLINE_EXCEEDED`)는 보통 재시도 대상에 넣지 않는다. 마감을 이미 넘긴 호출은 재시도해도 쓸 예산이 없기 때문이다. (재시도 정책의 세부 사항, 재시도 가능한 코드, 백오프 구성은 13장(안정성)에서 다룬다.)

### keepalive/커넥션과의 차이

deadline과 헷갈리기 쉬운 것이 keepalive이다. 둘 다 "오랫동안 끝나지 않는 상황"을 다루지만, 동작하는 계층이 다르다.

| 구분 | deadline | keepalive |
|------|----------|-----------|
| 대상 | 개별 RPC(논리적 호출) | 커넥션(물리적 연결) |
| 질문 | "이 호출이 마감 안에 끝났나?" | "이 커넥션의 상대가 아직 살아 있나?" |
| 신호 | grpc-timeout, RST_STREAM(CANCEL) | HTTP/2 PING 프레임 |
| 결과 | DEADLINE_EXCEEDED | 죽은 커넥션 감지/종료 |
| 적용 | 모든 RPC | 특히 장수명 스트림/유휴 커넥션 |

deadline은 "이 요청이 너무 오래 걸리는가?"를 다루고, keepalive는 "연결이 끊긴 줄도 모르고 대기하고 있는가?"를 다룬다. 장수명 스트림에 짧은 deadline 대신 keepalive를 쓰는 이유가 여기에 있다. 이런 스트림은 정상적으로 길게 열려 있어야 하므로 deadline으로 끊으면 안 되지만, 상대가 소리 없이 죽었는지는 PING으로 확인해야 한다. keepalive의 자세한 동작은 13장(안정성)을 참고하자.

### 채널/스텁 수준과 호출 수준

deadline은 보통 호출(call) 단위로 건다. 채널과 스텁의 생명주기는 [[07 - 채널 스텁 커넥션 생명주기]]에서 다루는데, 핵심만 말하면 deadline은 커넥션을 닫지 않는다. RPC 하나가 마감을 넘겨 그 스트림이 RST_STREAM으로 닫히더라도, 그 아래의 HTTP/2 커넥션과 채널은 그대로 유지되어 다른 RPC에 재사용된다. deadline은 커넥션 수준이 아니라 스트림 수준의 이벤트이다.

---

## 10. 안티패턴 모음

실무에서 반복해서 나타나는 deadline과 취소 관련 실수를 모았다.

### (1) 서버가 컨텍스트를 무시하고 끝까지 계산한다

가장 흔하면서 비용도 가장 큰 실수이다. 클라이언트는 이미 취소했거나 마감을 넘겼는데, 서버 핸들러는 `ctx`를 한 번도 확인하지 않고 무거운 계산을 끝까지 수행한다. 아무도 받지 않을 결과를 만들려고 CPU와 DB를 소모하는 것이다.

```go
// 나쁜 예
func (s *server) Heavy(ctx context.Context, req *pb.Req) (*pb.Resp, error) {
    result := 0
    for i := 0; i < 1_000_000_000; i++ {
        result += expensiveStep(i) // ctx 를 절대 보지 않는다
    }
    return &pb.Resp{Value: int64(result)}, nil
}

// 좋은 예: 주기적으로 ctx 확인
func (s *server) Heavy(ctx context.Context, req *pb.Req) (*pb.Resp, error) {
    result := 0
    for i := 0; i < 1_000_000_000; i++ {
        if i%10000 == 0 { // 매 스텝마다 확인하면 비싸니 주기적으로
            if err := ctx.Err(); err != nil {
                return nil, status.FromContextError(err).Err()
            }
        }
        result += expensiveStep(i)
    }
    return &pb.Resp{Value: int64(result)}, nil
}
```

### (2) 받은 컨텍스트를 버리고 빈 컨텍스트로 하위 호출한다 (전파 끊기)

5절에서 본 그대로이다. `context.Background()`나 `Context.ROOT`로 하위 호출을 새로 만들면 상류의 마감이 사라진다. "왜 우리 서비스에 좀비 작업이 가득하지?"라는 문제의 가장 흔한 원인이다.

### (3) deadline을 무한대로 두거나 아예 걸지 않는다

7절에서 설명한 위험 그대로이다. "일단 끊기지 않게 넉넉히 잡자"는 유혹은 결국 무한 대기와 연쇄 장애로 되돌아온다. 합리적인 기본값을 인터셉터로 강제하자.

### (4) 취소와 마감을 에러로 오인해 로그가 폭주한다

취소(`CANCELLED`)와 마감 초과(`DEADLINE_EXCEEDED`)는 정상적인 운영의 일부이다. 사용자가 탭을 닫거나, 상위 호출이 마감을 넘기거나, 더 빠른 응답이 와서 나머지를 취소하는 일은 늘 일어난다. 이를 서버에서 에러 레벨로 로깅하면 로그가 의미 없는 취소 메시지로 가득 차고, 정작 진짜 에러가 묻힌다.

```go
// 나쁜 예: 취소/마감도 ERROR 로 토해 냄 → 로그 폭주
if err != nil {
    log.Errorf("핸들러 실패: %v", err) // CANCELLED 도 여기로 쏟아짐
    return nil, err
}

// 좋은 예: 취소/마감은 낮은 레벨로, 진짜 에러만 ERROR
if err != nil {
    switch status.Code(err) {
    case codes.Canceled, codes.DeadlineExceeded:
        log.Debugf("호출 취소/마감(정상): %v", err) // 또는 메트릭만 증가
    default:
        log.Errorf("핸들러 실패: %v", err)
    }
    return nil, err
}
```

마감 초과가 비정상적으로 늘었는지는 로그가 아니라 **메트릭**으로 추적하는 것이 좋다. `DEADLINE_EXCEEDED` 비율이 평소보다 크게 오르면 하위 서비스가 느려졌다는 신호이다.

### (5) `defer cancel()`을 빠뜨린다 (Go 한정)

`context.WithTimeout`이나 `WithCancel`이 돌려준 `cancel` 함수를 호출하지 않으면, 그 컨텍스트에 연결된 타이머와 고루틴이 부모 컨텍스트가 끝날 때까지 회수되지 않아 누수가 생긴다. Go의 `go vet`이 이 실수를 잡아 준다. **`ctx, cancel := ...; defer cancel()`는 한 묶음으로 외워야 한다.**

### (6) 스트림에 짧은 deadline을 잘못 건다

장수명 스트림(알림 구독, 채팅)에 단항 호출처럼 짧은 deadline을 거는 실수이다. 그러면 정상 스트림이 마감에 걸려 끊긴다. 장수명 스트림은 deadline 대신 keepalive로 관리하자(9절).

### (7) 마감을 넘긴 뒤에도 재시도한다

`DEADLINE_EXCEEDED`를 받고도 같은 호출을 재시도하는 코드이다. 마감을 이미 넘겼으므로 재시도해도 곧바로 다시 마감 초과가 된다. 재시도는 남은 예산이 있을 때만 해야 한다(9절).

---

## 11. 종합 예제: deadline, 전파, 취소, 조기 종료를 한 번에

지금까지 살펴본 내용을 모아 실제로 동작할 수준으로 만든 예제이다. proto부터 보자.

```proto
syntax = "proto3";

package shop.v1;
option go_package = "example.com/shop/v1;shoppb";

// 주문 조회 서비스 (게이트웨이가 호출)
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
}

// 재고 확인 서비스 (주문 서비스가 다시 호출하는 하위 서비스)
service InventoryService {
  rpc CheckStock(CheckStockRequest) returns (CheckStockResponse);
}

message GetOrderRequest  { string order_id = 1; }
message GetOrderResponse { Order order = 1; }

message Order {
  string order_id = 1;
  repeated Item items = 2;
  bool in_stock = 3;
}
message Item { string sku = 1; int32 qty = 2; }

message CheckStockRequest  { string order_id = 1; }
message CheckStockResponse { bool in_stock = 1; }
```

**클라이언트(게이트웨이): 800ms 마감을 걸고 호출한다**

```go
func main() {
    conn, err := grpc.NewClient("order-service:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()
    client := shoppb.NewOrderServiceClient(conn)

    // 전체 호출의 마감: 지금부터 800ms (→ 내부적으로 절대 시각 T 로 변환)
    ctx, cancel := context.WithTimeout(context.Background(), 800*time.Millisecond)
    defer cancel()

    resp, err := client.GetOrder(ctx, &shoppb.GetOrderRequest{OrderId: "ORD-42"})
    if err != nil {
        switch status.Code(err) {
        case codes.DeadlineExceeded:
            log.Printf("마감 초과: 800ms 안에 주문을 못 받음")
        case codes.Canceled:
            log.Printf("호출이 취소됨")
        default:
            log.Printf("실패: %v", err)
        }
        return
    }
    log.Printf("주문: %v (재고: %v)", resp.GetOrder().GetOrderId(), resp.GetOrder().GetInStock())
}
```

**주문 서비스(서버이자 클라이언트): 받은 ctx를 전파하고 조기 종료한다**

```go
type orderServer struct {
    shoppb.UnimplementedOrderServiceServer
    inv shoppb.InventoryServiceClient
    db  *sql.DB
}

func (s *orderServer) GetOrder(ctx context.Context, req *shoppb.GetOrderRequest) (*shoppb.GetOrderResponse, error) {
    // 1) 비싼 일 전에 이미 취소/마감인지 값싸게 확인
    if err := ctx.Err(); err != nil {
        return nil, status.FromContextError(err).Err()
    }

    // 2) 하위 호출에 받은 ctx 를 "그대로" 전달 → 800ms 마감이 재고 서비스까지 전파.
    //    이 시점 grpc-go 가 (T - now) 를 grpc-timeout 으로 재환산해 인코딩한다.
    stockResp, err := s.inv.CheckStock(ctx, &shoppb.CheckStockRequest{OrderId: req.GetOrderId()})
    if err != nil {
        return nil, err // 하위의 DEADLINE_EXCEEDED 도 자연스럽게 위로
    }

    // 3) DB 조회에도 같은 ctx → 마감 도달 시 쿼리 자체가 취소됨
    rows, err := s.db.QueryContext(ctx,
        "SELECT sku, qty FROM order_items WHERE order_id = $1", req.GetOrderId())
    if err != nil {
        // ctx 취소로 인한 실패면 DeadlineExceeded/Canceled 로 매핑
        if ctx.Err() != nil {
            return nil, status.FromContextError(ctx.Err()).Err()
        }
        return nil, status.Errorf(codes.Internal, "db: %v", err)
    }
    defer rows.Close()

    var items []*shoppb.Item
    for rows.Next() {
        // 4) 긴 스캔 루프에서도 주기적으로 취소 확인
        select {
        case <-ctx.Done():
            return nil, status.FromContextError(ctx.Err()).Err()
        default:
        }
        var it shoppb.Item
        if err := rows.Scan(&it.Sku, &it.Qty); err != nil {
            return nil, status.Errorf(codes.Internal, "scan: %v", err)
        }
        items = append(items, &it)
    }

    return &shoppb.GetOrderResponse{
        Order: &shoppb.Order{
            OrderId: req.GetOrderId(),
            Items:   items,
            InStock: stockResp.GetInStock(),
        },
    }, nil
}
```

**시간 순서로 본 정상 처리와 마감 초과**

```text
[정상] 총 예산 800ms 안에 끝남
  t=0    게이트웨이 GetOrder 시작 (마감 T=800ms)
  t=20   주문서비스 도착, grpc-timeout ≈ 780m 재환산되어 도착해 있었음
  t=25   CheckStock 호출(ctx 전파) → grpc-timeout ≈ 775m 로 재고서비스에
  t=120  CheckStock 응답
  t=130  DB 쿼리(ctx 전파)
  t=300  DB 응답, 조립
  t=320  게이트웨이가 응답 수신 ✅ (320 < 800)

[마감 초과] 재고서비스가 느림
  t=0    GetOrder 시작 (마감 T=800ms)
  t=25   CheckStock 호출(ctx 전파)
  t=800  마감 도달! 게이트웨이 ctx 의 타이머 발동
         → 게이트웨이가 RST_STREAM(CANCEL) 송신, 호출자에게 DEADLINE_EXCEEDED
         → 주문서비스 ctx 취소됨(전파) → CheckStock 의 ctx 도 취소
         → 재고서비스 ctx 취소됨 → 진행 중 작업 중단(협조 시)
  t=800+ 아무도 좀비 작업을 계속하지 않음 ✅
```
