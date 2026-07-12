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

**이 장이 답하는 질문**

- deadline과 timeout은 무엇이 다르고, gRPC는 왜 굳이 deadline(절대 시각)을 1급 개념으로 삼았는가?
- 클라이언트가 정한 마감이 네트워크를 건너 서버에, 그리고 서버가 다시 부르는 또 다른 서버에 어떻게 "전파"되는가? 와이어 위에서는 실제로 어떤 바이트가 흐르는가?
- 클라이언트가 호출을 취소하면 서버의 코드는 어떻게 그것을 알아채고 멈추는가? deadline 초과는 취소와 어떻게 연결되는가?
- Go의 `context.Context`, Java의 `Context`/`Deadline`은 이 추상을 각각 어떻게 구현하며, 서버 핸들러는 왜 받은 컨텍스트를 반드시 하위 호출로 흘려보내야 하는가?
- deadline 예산(budget)을 어떻게 나눠야 허위 `DEADLINE_EXCEEDED`도, 좀비 작업(zombie work)도 피할 수 있는가?

---

## 들어가며: 마감이 없는 약속은 약속이 아니다

분산 시스템에서 가장 위험한 상태는 "에러"가 아니라 "영원히 기다림"이다. 에러는 빠르게 실패(fail-fast)하고 재시도하거나 대체 경로를 타게 해 준다. 그러나 응답도, 에러도 오지 않고 그저 멈춰 있는 호출은 호출한 쪽의 스레드, 메모리, 커넥션, 그리고 그 호출을 기다리는 상위 호출자까지 차례로 잡아먹는다. 한 군데에서 시작된 느려짐이 호출 사슬을 거슬러 올라가며 시스템 전체를 마비시키는 것, 이것이 우리가 두려워하는 연쇄 장애(cascading failure)다.

이 장의 주제인 deadline은 바로 이 "영원히 기다림"을 구조적으로 불가능하게 만드는 장치다. gRPC는 거의 모든 다른 RPC 프레임워크와 달리, deadline을 부가 기능이 아니라 핵심 프로토콜의 일부로 못 박았다. 모든 RPC는 명시적이든 암묵적이든 마감을 가지며, 그 마감은 와이어 포맷의 일부로 전달되고, 마감이 지나면 호출은 자동으로 종료된다.

흥미롭게도 gRPC는 이 개념을 "timeout(타임아웃)"이 아니라 "deadline(데드라인)"이라는 단어로 부른다. 둘은 일상어에서 거의 동의어처럼 쓰이지만, gRPC의 설계에서는 명확히 구분되며 이 구분이야말로 deadline 전파라는 강력한 기능의 토대가 된다. 우선 이 차이부터 정확히 짚고 넘어가자.

---

## 1. deadline vs timeout: 절대 시각과 상대 시간

가장 먼저 머릿속에 새겨야 할 구분이다.

| 구분 | timeout (타임아웃) | deadline (데드라인) |
|------|-------------------|---------------------|
| 의미 | "얼마 동안" 기다릴까 | "언제까지" 기다릴까 |
| 성격 | 상대 시간(duration) | 절대 시각(point in time) |
| 예시 | "지금부터 5초" | "13:00:05.000 UTC까지" |
| 기준점 | 측정을 시작하는 순간 | 벽시계(또는 단조 시계)의 한 지점 |
| 합성 | 매번 다시 계산해야 함 | 그냥 전달하면 됨 |

비유로 시작하자. 친구에게 "30분 줄게, 그 안에 와"라고 말하는 것이 timeout이다. 반면 "3시까지 와"라고 말하는 것이 deadline이다. 둘은 같은 약속처럼 보이지만 결정적인 차이가 있다. 당신이 친구 A에게 "3시까지 와"라고 말하고, A가 다시 B에게 그 약속을 전달할 때, A는 그냥 "3시까지 와"라고 똑같이 말하면 된다. 하지만 "30분 줄게"라고 말했다면, A는 자기가 이미 10분을 썼다는 걸 계산해서 B에게 "20분 줄게"라고 고쳐 말해야 한다. 계산이 끼어드는 순간 실수가 생긴다.

분산 호출 사슬이 정확히 이 구조다. 클라이언트가 서비스 A를 부르고, A가 B를, B가 C를 부른다. 사용자는 "이 요청 전체가 1초 안에 끝나야 한다"고 생각한다. 이 1초라는 마감을 A, B, C 모두가 공유해야 한다. deadline(절대 시각)으로 표현하면 "지금 시각 + 1초 = T"라는 단 하나의 진실이 사슬 전체를 관통한다. A는 T를 B에게, B는 T를 C에게 그대로 넘긴다. 누가 얼마나 시간을 썼든 마감 시각 자체는 변하지 않는다.

> 핵심 직관: deadline은 호출 사슬 위를 흐를 때 **변하지 않는 불변량(invariant)** 이다. 바로 그래서 합성(compose)이 쉽고, 바로 그래서 gRPC가 deadline을 1급 개념으로 골랐다.

### gRPC가 내부적으로 deadline 중심으로 도는 이유

그렇다면 프로그래머가 API에서 보는 것은 보통 timeout인데(예: `WithTimeout(ctx, time.Second)`), 왜 "내부적으로는 deadline"이라고 말할까? 핵심은 **계산이 단 한 번만 일어난다**는 점이다.

- 프로그래머는 편의상 상대 시간(timeout)으로 표현한다. "1초 줘."
- gRPC 라이브러리(또는 언어의 context)는 이걸 받자마자 **즉시 절대 시각으로 변환**한다. `deadline = now() + 1s`.
- 이후 모든 내부 동작 — 타이머 설정, 남은 시간 계산, 헤더 인코딩, 하위 호출 전파 — 은 이 절대 시각 `deadline`을 기준으로 한다.

이 변환을 호출 진입점에서 단 한 번만 하면, 그 뒤로는 "지금부터 얼마 남았나?"라는 질문에 항상 `deadline - now()`라는 동일한 공식으로 답할 수 있다. 만약 내부적으로 상대 시간을 들고 다녔다면, 큐에서 대기한 시간, 직렬화에 쓴 시간, 첫 번째 하위 호출에 쓴 시간을 매 단계마다 빼 줘야 하고, 그 뺄셈 하나하나가 버그의 온상이 된다. 절대 시각은 이 모든 뺄셈을 "현재 시각을 읽는다"라는 단 하나의 연산으로 대체한다.

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

여기서 미묘하지만 중요한 디테일이 하나 있다. "절대 시각"이라고 하면 벽시계(wall clock, 사람이 읽는 캘린더 시각)를 떠올리기 쉽지만, 한 프로세스 안에서 deadline까지 남은 시간을 재는 데는 단조 시계(monotonic clock)를 쓰는 것이 옳다. 벽시계는 NTP 동기화나 윤초 탓에 갑자기 뒤로 점프할 수 있어 `deadline - now()`가 음수로 튀거나 폭증하기 때문이다.

Go의 `time.Time`은 단조 시계 성분을 함께 들고 다녀 `time.Until(deadline)` 계산을 안전하게 만든다. 다만 **서로 다른 머신 사이**에서는 단조 시계를 공유할 수 없다(각 머신의 단조 시계는 부팅 이후 경과 시간 같은 임의 기준점을 가진다). 그래서 와이어를 건널 때는 절대 시각을 그대로 보내지 않고, "지금부터 남은 상대 시간"으로 환산해 보낸다. 이것이 다음 절의 `grpc-timeout` 헤더다. gRPC는 이렇게 양쪽의 장점을 영리하게 취한다.

- **프로세스 내부**: 절대 시각(deadline) + 단조 시계로 안전하고 합성하기 쉽게 관리
- **머신 사이(와이어)**: 상대 시간(`grpc-timeout`)으로 환산해 시계 동기화 문제를 회피

---

## 2. 와이어 표현: `grpc-timeout` 헤더

deadline이 한 머신에서 다른 머신으로 건너갈 때 실제로 어떤 모습인지 보자. gRPC는 요청의 HTTP/2 HEADERS 프레임 안에, `grpc-timeout`이라는 메타데이터 키로 마감을 싣는다. (HTTP/2와 HEADERS 프레임 자체는 [[04 - HTTP2 깊이 보기 - 전송 계층]]에서 다룬다. 여기서는 그 위에 얹히는 gRPC 약속에 집중한다.)

값의 포맷은 **"정수 + 단위 문자 한 글자"** 다.

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

여기서 대소문자 구분에 주의하자. 대문자 `M`은 분(Minute), 소문자 `m`은 밀리초(millisecond)다. 대문자 `S`는 초(Second)인데 소문자 `s`는 정의되어 있지 않다. 이 한 글자 차이가 1000배 또는 60000배의 오차를 만든다.

정수 부분은 **최대 8자리**로 제한된다. 그래서 `100000000S`(9자리)처럼 쓸 수 없고, 더 큰 값이 필요하면 단위를 키워야 한다. 예컨대 아주 긴 마감은 초 대신 분이나 시간 단위로 표현한다. 이 8자리 제약은 헤더를 짧게 유지하려는 장치다.

### 매 홉마다 재환산: 와이어로 나갈 때 "지금 남은 시간"

가장 중요한 동작 원리는 이것이다. **클라이언트는 자기가 들고 있는 절대 deadline에서 "지금 남은 시간"을 계산해 `grpc-timeout`으로 인코딩한다.** 이 인코딩은 매 RPC 발신 시점, 즉 매 홉(hop)마다 새로 일어난다.

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

매 홉에서 남은 시간이 줄어드는 것이 자연스럽게 표현된다. A→B 사이의 네트워크 지연과 B가 소비한 처리 시간만큼 차감된 채로 B→C에 전달된다. 절대 시각이라는 불변량을 들고 있되, 와이어로 내보낼 때만 "지금 남은 만큼"으로 투영(project)한다.

### 실제 바이트로 디코딩해 보기

`grpc-timeout: 1S`가 HTTP/2 HEADERS 프레임 안에서 어떻게 인코딩되는지 손으로 따라가 보자. HTTP/2는 헤더를 HPACK으로 압축한다. `grpc-timeout`은 정적 테이블에 없는 헤더 이름이므로, 첫 등장 시에는 보통 "이름과 값을 모두 리터럴로, 동적 테이블에 추가" 형태(0x40 패턴)로 보낸다.

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

두 번째 요청부터는 헤더 이름 `grpc-timeout`이 동적 테이블에 들어가 있으므로 이름은 인덱스로 참조되고 값만 리터럴로 보내져 더 짧아진다. 핵심은 `grpc-timeout`의 값이 우리가 읽을 수 있는 ASCII 문자열 `"1S"`로 그대로 들어간다는 점이다. 와이어 위에서 deadline은 거창한 바이너리가 아니라 사람이 눈으로 읽을 수 있는 짧은 텍스트다.

한 가지 정밀한 단서: `grpc-timeout`의 값(`"1S"`, `"940m"` …)은 매 요청마다 달라지므로, 이름·값 쌍 전체를 동적 테이블에 넣어 봤자(증분 인덱싱) 다음 요청에서 재사용되지 못하고 테이블만 휘젓는다(dynamic table churn). 그래서 동적 테이블 효율을 신경 쓰는 인코더는 **이름만 한 번 인덱싱하고 값은 인덱싱 없이(Literal Header Field without Indexing, `0x00` 패턴)** 보내는 쪽을 택하기도 한다. 위 예시는 "이름·값을 모두 리터럴로 보낸다"는 점을 한눈에 보이려고 증분 인덱싱(`0x40`) 형태로 그렸을 뿐, 실제로 어떤 표현을 고를지는 HPACK 인코더 구현마다 다르다. 어느 쪽이든 와이어에 흐르는 값 자체는 동일한 ASCII 문자열이다.

`grpcurl`이나 디버그 로그로 헤더를 들여다보면 이 값을 직접 확인할 수 있다. ([[14 - 관찰성과 디버깅 - Reflection grpcurl]] 참고.)

```bash
# 서버 측에서 받은 메타데이터를 로깅하도록 인터셉터를 걸어 두면
# grpc-timeout: 1S 같은 형태로 관측된다.
# (인터셉터로 메타데이터를 보는 방법은 08장 참고)
```

### deadline이 없으면?

`grpc-timeout` 헤더를 아예 보내지 않으면 서버 입장에서 그 RPC는 **마감이 없는(무한대) 호출**이 된다. 이것이 바로 우리가 피하려는 상태다. 클라이언트가 명시적으로 deadline을 설정하지 않으면 많은 언어 구현에서 기본값이 "무한대"이므로 헤더가 누락되고 서버는 영원히 기다려도 되는 줄 안다. 뒤의 안티패턴 절에서 이 위험을 다시 짚는다.

---

## 3. deadline 전파(propagation): 하나의 마감을 모두가 공유한다

이제 deadline의 가장 강력한 기능, 전파(propagation)를 본격적으로 보자. 이것이 gRPC가 deadline을 절대 시각으로 다루기로 한 진짜 보상이다.

### 시나리오: A → B → C 호출 사슬

사용자가 "주문 상세 보기"를 요청한다. 이 요청은 게이트웨이 A로 들어오고, A는 주문 서비스 B를 부르고, B는 다시 재고 서비스 C와 사용자 서비스를 부른다. 사용자는 1초 안에 화면이 떠야 한다고 기대한다.

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

deadline 전파가 의미하는 바는 이것이다. **A가 정한 마감 T가 B와 C에게도 그대로 적용된다.** B와 C는 각자 "나는 얼마나 기다려야 하지?"를 독립적으로 정하지 않는다. 둘은 A로부터 흘러온 마감을 물려받는다.

### 왜 이게 중요한가: 이미 끝난 마감을 넘겨 일하는 낭비를 막는다

deadline 전파가 없다고 상상해 보자. A는 1초 타임아웃으로 B를 부른다. 그런데 B는 자기 사정대로 C를 5초 타임아웃으로 부른다. 이제 다음 상황이 벌어진다.

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

여기서 `t=1000ms` 이후에 B와 C가 한 모든 일은 **이미 폐기될 것이 확정된 작업**이다. 아무도 그 결과를 받지 않는다. 그런데도 B와 C는 CPU, 메모리, DB 커넥션, 스레드를 계속 점유한다. 부하가 높을수록 이런 "좀비 작업(zombie work)"이 쌓여 시스템이 의미 없는 일로 가득 찬다. 정작 살아 있는 요청을 처리할 자원이 고갈된다.

deadline 전파는 이 낭비를 원천 차단한다. A의 마감 T가 C까지 흘러갔으므로, A가 손을 떼는 순간 C도 같은 마감에 걸려 자동으로 멈춘다. 아무도 기다리지 않는 일을 아무도 하지 않는다.

> 한 줄로: **"호출자가 이미 포기한 일을, 피호출자가 계속 붙들고 있지 않게 한다."** 이것이 deadline 전파의 본질이다.

### budget 소진을 그림으로

A가 가진 1000ms 예산이 사슬을 따라 어떻게 소진되는지 시각화해 보자.

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

budget(예산)이라는 단어가 잘 어울린다. A는 1000ms라는 예산을 들고 시작하고, 네트워크 지연과 각 서비스의 처리 시간이 그 예산에서 차감된다. 사슬 어디서든 예산이 0이 되면 그 지점에서 즉시 `DEADLINE_EXCEEDED`로 터진다. 누구도 예산을 초과해 쇼핑할 수 없다.

### 전파는 자동인가? — 컨텍스트를 흘려보내야 자동이다

여기에 결정적인 함정이 있다. deadline 전파는 "마법처럼 저절로" 되는 것이 아니라, **서버 코드가 들어온 컨텍스트(context)를 하위 호출에 명시적으로 넘겨줄 때만** 일어난다. Go라면 핸들러가 받은 `ctx`를, Java라면 `Context.current()`를 하위 gRPC 클라이언트 호출에 전달해야 한다.

만약 서버 핸들러가 받은 컨텍스트를 무시하고 `context.Background()`(아무 마감도 없는 빈 컨텍스트)로 하위 호출을 새로 만들면, 전파의 사슬이 그 지점에서 끊긴다. B는 A의 마감을 물려받았지만, B→C 호출에 그 마감을 안 실으면 C는 다시 무한대 마감으로 돌아간다. 그래서 다음 절의 언어별 모델에서 "컨텍스트를 흘려보내라"는 규칙이 그토록 강조된다.

---

## 4. 취소(cancellation) 메커니즘: 멈추라는 신호는 어떻게 전달되나

deadline 초과는 취소의 특수한 한 경우다. 그래서 먼저 일반적인 취소가 와이어 위에서 어떻게 동작하는지 보고, 그다음 deadline 초과가 어떻게 이 메커니즘으로 흡수되는지 보자.

### 클라이언트가 취소하면: HTTP/2 RST_STREAM(CANCEL)

클라이언트가 진행 중인 RPC를 취소한다고 하자. 사용자가 브라우저 탭을 닫았거나, 상위 요청이 이미 끝났거나, 더 빠른 다른 응답이 먼저 와서 이 호출이 필요 없어졌을 수 있다. 이때 gRPC 클라이언트는 그 RPC에 해당하는 **HTTP/2 스트림을 RST_STREAM 프레임으로 닫는다.**

HTTP/2에서 각 RPC는 하나의 스트림(stream)에 대응한다(멀티플렉싱(multiplexing)으로 한 커넥션 위에 여러 스트림이 동시에 흐른다, [[04 - HTTP2 깊이 보기 - 전송 계층]]). 스트림을 중간에 끊으려면 RST_STREAM 프레임을 보낸다. gRPC 취소의 경우 에러 코드로 `CANCEL`(HTTP/2 에러 코드 0x8)을 실어 보낸다.

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

이 4바이트짜리 작은 프레임이 "이 호출은 이제 필요 없으니 그만둬"라는 메시지의 전부다.

### 서버 쪽에서 벌어지는 일: 컨텍스트가 취소된다

서버의 HTTP/2 전송 계층이 RST_STREAM(CANCEL)을 수신하면, 그 스트림에 매여 있던 gRPC 호출의 컨텍스트(서버 측 context)를 **취소 상태로 전환**한다. 그 순간:

- Go라면 그 호출의 `ctx.Done()` 채널이 닫히고, `ctx.Err()`가 `context.Canceled`를 반환한다.
- Java라면 그 호출의 `Context`가 취소되어 `Context.current().isCancelled()`가 `true`가 되고, 등록된 취소 리스너가 호출된다.

여기서 가장 중요한 책임이 등장한다. **서버는 이 신호를 보고 즉시 일을 멈춰야 한다.** gRPC 런타임이 강제로 핸들러 함수를 중단시켜 주지는 않는다(스레드를 강제로 죽이는 것은 위험하므로). 대신 런타임은 "취소됨"이라는 신호만 컨텍스트에 실어 줄 뿐이고, 그 신호를 보고 빠져나오는 것은 **핸들러 코드의 협조(cooperation)** 에 달려 있다. 핸들러가 컨텍스트를 들여다보지 않으면, 신호가 와도 끝까지 계산을 계속한다. 이것이 뒤에서 다룰 가장 흔한 안티패턴이다.

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

### deadline 초과는 자동 취소 → DEADLINE_EXCEEDED

이제 deadline 초과가 이 그림에 어떻게 끼워지는지 보자. deadline은 그냥 **"미래의 특정 시각에 자동으로 발동되는 취소 타이머"** 다.

- 클라이언트가 deadline을 설정하면, 클라이언트 내부에 타이머가 걸린다. 그 시각이 되면 클라이언트는 RPC를 스스로 취소한다(서버로 RST_STREAM 전송). 클라이언트가 호출자에게 돌려주는 상태 코드는 `DEADLINE_EXCEEDED`(코드 4)다.
- 서버 쪽에서도, 전파된 deadline을 알고 있으므로 그 시각이 되면 서버 측 컨텍스트를 자동으로 취소한다(`ctx.Err()`가 `context.DeadlineExceeded`). RST_STREAM이 도착하기 전이라도 서버는 스스로 마감을 알고 멈출 수 있다.

즉 deadline 초과는 "타이머가 발동시킨 취소"이고, 일반 취소는 "사람/상위 요청이 발동시킨 취소"다. 메커니즘은 동일하다. 차이는 호출자에게 보고되는 상태 코드뿐이다.

| 발동 원인 | 클라이언트가 받는 상태 코드 | 코드 번호 |
|-----------|---------------------------|-----------|
| 명시적 취소(사용자/상위 종료) | `CANCELLED` | 1 |
| deadline 시각 도달 | `DEADLINE_EXCEEDED` | 4 |

(상태 코드 체계 전반은 [[09 - 에러 모델 - 상태 코드와 Rich Error]]에서 다룬다. 여기서는 이 둘만 짚는다.)

> 정신 모델: **deadline = 미리 예약해 둔 취소**. 취소의 모든 배관(plumbing) — RST_STREAM, 컨텍스트 취소, 하위 호출로의 전파 — 을 그대로 재사용하되, 방아쇠가 시계일 뿐이다.

---

## 5. 언어별 모델: Go의 context, Java의 Context/Deadline

추상은 같지만, 각 언어가 이를 구현하는 방식은 그 언어의 관용구를 따른다. 가장 널리 쓰이는 두 모델을 본다.

### Go: `context.Context`

Go에서는 `context.Context`가 deadline·취소·요청 범위 값(request-scoped value)을 모두 담는 단일 추상이다. gRPC뿐 아니라 표준 라이브러리 전반(`net/http`, `database/sql` 등)이 이 인터페이스를 공유하기 때문에, deadline과 취소가 언어 생태계 전체에 자연스럽게 흐른다. 이것이 Go에서 deadline 전파가 특히 매끄러운 이유다.

핵심 API:

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

`context.WithTimeout`이 내부적으로 `WithDeadline(parent, now+timeout)`을 호출한다는 점에 주목하자. 이것이 1절에서 말한 "프로그래머는 timeout으로 쓰지만 라이브러리는 즉시 deadline으로 변환한다"의 실제 코드다.

**클라이언트 측 — deadline 설정**:

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

gRPC-Go는 이 `ctx`에 박힌 deadline을 읽어 `grpc-timeout` 헤더로 인코딩해 보낸다. 프로그래머는 헤더를 직접 만들 필요가 없다. `context`에 deadline을 심는 것만으로 와이어 인코딩과 전파가 따라온다.

**서버 측 — 컨텍스트를 확인하며 조기 종료, 그리고 하위 호출로 전파**:

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

이 예제에서 가장 중요한 줄은 `s.inventoryClient.CheckStock(ctx, ...)`와 `s.db.QueryContext(ctx, ...)`에서 **받은 `ctx`를 그대로 전달**하는 부분이다. 만약 여기서 `context.Background()`를 새로 만들어 넘겼다면 전파의 사슬이 끊긴다. "서버 핸들러는 받은 컨텍스트를 자신의 모든 하위 IO와 호출에 흘려보내야 한다"는 규칙이 바로 이것이다.

**안티패턴 (전파 끊기)**:

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

Java의 gRPC는 `io.grpc.Context`로 요청 범위 정보를, `io.grpc.Deadline`으로 마감을 표현한다. Go의 `context.Context`가 인자로 명시적으로 전달되는 반면, Java의 `Context`는 **스레드 로컬(thread-local)**에 현재 컨텍스트를 보관하는 모델을 기본으로 한다(`Context.current()`). 다만 스레드 경계를 넘을 때(스레드 풀, 비동기)는 명시적으로 컨텍스트를 전파해야 한다.

**클라이언트 측 — deadline 설정**:

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

`withDeadlineAfter(800, MILLISECONDS)`는 "지금부터 800ms"라는 상대 시간을 받지만, 내부적으로는 `Deadline.after(800, MILLISECONDS)`로 절대 시각(단조 시계 기준)을 계산해 둔다. 여기서 한 가지 함정. **스텁에 박은 deadline은 그 스텁 인스턴스에 고정**되므로, 같은 스텁을 재사용해 여러 번 호출하면 모두 같은 절대 마감을 공유한다. 매 호출마다 새 마감이 필요하면 호출 직전에 `withDeadlineAfter`로 새 스텁을 파생시켜야 한다.

**서버 측 — 취소 확인과 전파**:

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

Java에서는 **스레드 풀로 작업을 넘길 때 컨텍스트가 따라가지 않는다**는 점을 주의해야 한다. `Context`는 스레드 로컬에 묶이므로, 다른 스레드에서 실행되는 작업에 현재 컨텍스트를 전파하려면 `Context.current().wrap(runnable)` 또는 `ctx.run(() -> ...)`로 감싸야 한다.

```java
// 스레드 풀에 작업을 넘길 때 컨텍스트(마감 포함)를 함께 전파
Context ctx = Context.current();
executor.submit(ctx.wrap(() -> {
    // 이 안에서는 Context.current() 가 바깥의 마감을 그대로 본다
    doSubtask();
}));
```

### Python: 참고

Python의 gRPC도 동일한 추상을 제공한다. 클라이언트는 호출에 `timeout=` 인자(상대 시간, 초 단위)를 주고, 서버 핸들러는 `context.is_active()`, `context.time_remaining()`, `context.add_callback(...)`으로 취소/마감을 관찰한다.

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

세 언어를 관통하는 규칙은 단 하나다. **"받은 마감/취소 신호를 들고 다니다가, 모든 하위 호출과 IO에 그대로 흘려보내라."** 이 규칙을 어기는 순간 전파가 끊기고 좀비 작업이 태어난다.

---

## 6. 스트리밍에서의 취소: 양방향성

지금까지의 예는 주로 단항(Unary) 호출이었다. 스트리밍에서는 취소가 더 미묘하다. 스트리밍의 종류와 기본 동작은 [[05 - 통신의 4가지 방식 - Unary와 Streaming]]에서 다루므로, 여기서는 deadline·취소가 스트림에 어떻게 작용하는지에 집중한다.

### 스트림 전체에 하나의 deadline

서버 스트리밍이든 클라이언트 스트리밍이든 양방향(bidi)이든, deadline은 **스트림 전체의 수명**에 걸린다. "메시지 하나당" 마감이 아니라 "이 스트림을 여는 순간부터 닫힐 때까지 전부 합쳐 N초" 같은 의미다. 그래서 오래 살아 있어야 하는 스트림(예: 실시간 알림 구독, 채팅 채널)에는 짧은 deadline을 걸면 안 된다. 정상적으로 길게 열려 있는 스트림이 마감에 걸려 `DEADLINE_EXCEEDED`로 끊겨 버린다.

이런 장수명 스트림에는 deadline 대신 keepalive로 죽은 커넥션을 감지하는 편이 맞다. deadline과 keepalive의 구분은 9절과 [[13 - 안정성 - Retry Health Check Keepalive]]에서 다시 본다.

### 양방향성: 한쪽의 취소가 스트림을 끝낸다

양방향 스트리밍에서 클라이언트와 서버는 같은 HTTP/2 스트림 위에서 양방향으로 메시지를 주고받는다. 이 스트림에 대한 취소(RST_STREAM)는 **방향과 무관하게 스트림 전체를 종료**시킨다.

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

Go에서 양방향 스트림 서버 핸들러는 보통 송신 고루틴과 수신 고루틴을 함께 돌리는데, 둘 다 `stream.Context()`를 관찰해야 한다.

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

핵심 직관: **양방향 스트림에서 취소는 "둘 사이의 대화 자체"를 끝낸다.** 한쪽이 "그만"이라고 하면 그 한 마디로 전화가 끊기고, 상대가 하려던 말도 전달되지 않는다. 그래서 양쪽 모두 송신 전에 스트림이 아직 살아 있는지 확인하는 협조가 필요하다.

---

## 7. 왜 모든 RPC에 deadline을 설정해야 하는가

이쯤 되면 답이 거의 자명해졌지만, 명시적으로 정리하자. **모든 RPC에 deadline을 걸어야 한다.** 예외는 의도적으로 장수명인 스트림 정도뿐이다.

### deadline 없는 호출의 3가지 위험

**(1) 무한 대기로 인한 스레드/고루틴 고갈.** 마감이 없으면 응답도 에러도 없는 호출이 영원히 매달려 있을 수 있다. 그 호출을 기다리는 스레드(또는 고루틴, 커넥션)는 회수되지 않는다. 동시 호출이 쌓이면 스레드 풀과 커넥션 풀이 고갈되고, 그 자원을 공유하는 멀쩡한 요청들까지 처리되지 못한다.

```text
deadline 없음 + 느린 하위 서비스:
  요청1 ──▶ [매달림] ──┐
  요청2 ──▶ [매달림] ──┤  스레드/커넥션이 회수 안 됨
  요청3 ──▶ [매달림] ──┤  → 풀 고갈
  요청4 ──▶ (풀 없음, 즉시 실패 또는 큐에서 무한 대기)
  ...
  결국 정상 요청까지 처리 불가 → 서비스 전체 다운
```

**(2) 리소스 고갈과 메모리 누수.** 매달린 호출은 스레드만이 아니라 그 호출에 묶인 버퍼, 요청 객체, 부분 응답, 잠금(lock)까지 붙들고 있다. 좀비 작업이 쌓일수록 메모리가 차오르고, 잠금을 쥔 채 멈춰 있으면 다른 작업까지 막힌다.

**(3) 장애 전파(연쇄 장애).** 가장 무서운 시나리오다. 사슬 맨 끝의 C가 느려지면, deadline이 없을 경우 B가 C를 무한정 기다리며 B의 자원을 소진하고, B가 느려지니 A가 무한정 기다리며 A의 자원을 소진한다. 한 서비스의 느려짐이 마감이라는 방화벽 없이 상류로 번져 전체를 무너뜨린다. deadline은 이 전파를 끊는 방화벽이다. C가 마감을 넘기면 B는 즉시 실패로 처리하고 자원을 회수해, 자기까지 같이 죽는 것을 막는다.

> 한 줄로: **deadline은 분산 시스템의 회로 차단기(circuit breaker)의 가장 원초적인 형태다.** 느림이 무한 대기가 되지 않게, 무한 대기가 자원 고갈이 되지 않게, 자원 고갈이 연쇄 장애가 되지 않게 한다.

### 기본값에 기대지 말 것

많은 gRPC 구현에서 deadline의 기본값은 "무한대"다. **아무것도 설정하지 않으면 가장 위험한 상태**가 된다. 안전한 기본값이 아니라는 점이 함정이다. 그래서 팀 차원에서 "deadline 없는 호출은 코드 리뷰에서 막는다", "클라이언트 인터셉터에서 deadline이 없으면 합리적 기본값을 강제로 주입한다"([[08 - 메타데이터와 인터셉터]]) 같은 규율이 필요하다.

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

deadline을 거는 것이 옳다면, 그 값을 얼마로 할 것인가가 다음 질문이다. 이것이 deadline 예산(budget) 설계다. 너무 빡빡해도, 너무 느슨해도 문제다.

### 예산을 나누는 사고법

A가 전체 1000ms 예산을 가졌다고 하자. 이 예산은 사슬의 여러 소비자에게 나뉜다.

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

deadline 전파가 자동으로 "남은 예산"을 하위로 넘겨 주므로, 각 서비스가 직접 "내 하위에 얼마를 줄까"를 빼서 계산할 필요는 줄어든다. 하지만 한 서비스가 여러 하위 호출을 순차로/병렬로 한다면, 그 안에서 어떻게 시간을 배분할지는 여전히 설계 대상이다.

### 너무 빡빡할 때: 허위 DEADLINE_EXCEEDED

예산을 실제 처리 시간보다 박하게 잡으면, 시스템이 멀쩡히 일하고 있는데도 마감에 걸려 실패한다. 이를 허위(false) `DEADLINE_EXCEEDED`라 부르자.

- 정상 작동 중인 요청이 마감 초과로 끊긴다 → 사용자에게는 에러로 보인다.
- 끊긴 요청을 재시도하면 부하가 늘어, 시스템이 더 느려지고, 더 많은 마감 초과를 부른다 → 악순환(retry storm).
- 특히 꼬리 지연(tail latency, p99)을 무시하고 평균만 보고 예산을 잡으면, 평소엔 괜찮다가 부하가 조금만 올라도 무더기로 마감 초과가 터진다.

박한 예산은 "조금만 느려도 다 죽이는" 신경질적인 시스템을 만든다.

### 너무 느슨할 때: 좀비 작업과 늦은 실패

반대로 예산을 지나치게 후하게(예: 30초) 잡으면, deadline이 사실상 무한대에 가까워져 본래 막으려던 문제가 되살아난다.

- 하위 서비스가 죽었거나 영영 응답하지 않는데도, 30초 동안 자원을 붙들고 기다린다 → 좀비 작업.
- 사용자는 "안 되면 빨리 알려 주기라도 하지"라고 느낀다. 30초 뒤에 실패를 통보받는 것은 거의 아무 가치가 없다.
- 빠른 실패(fail-fast)와 대체 경로(fallback)로 갈 기회를 놓친다.

### 좋은 예산의 원칙

- **사용자 체감 마감에서 출발한다.** "이 화면은 1초 안에 떠야 한다"가 최상위 예산이다. 안에서 일어나는 모든 일이 그 안에 들어가야 한다.
- **꼬리 지연 + 여유를 기준으로 잡는다.** 평균이 아니라 p99/p999 같은 실제 분포의 꼬리에, 약간의 여유(GC 멈춤, 스케줄링 지터)를 더한다.
- **상류는 하류보다 약간 더 길게.** A의 마감이 B의 마감보다 약간 길어야, B가 마감 직전에 실패를 만들어 A에게 의미 있는 에러를 돌려줄 시간이 있다. A와 B의 마감이 같으면, B의 마감 초과 응답이 A에게 도착하기도 전에 A도 마감을 넘겨 버려, A는 그냥 자기 마감 초과만 본다(B가 준 풍부한 에러를 못 본다).
- **deadline 전파를 신뢰하되 끊기지 않게 한다.** 5절의 규칙대로 컨텍스트를 흘려보내면, 굳이 매 단계에서 손으로 예산을 빼지 않아도 된다.

---

## 9. 다른 메커니즘과의 상호작용

### 재시도(retry)와 deadline

재시도는 deadline과 긴밀하게 얽힌다. 핵심 규칙: **재시도는 남은 예산 안에서만 한다.** 전체 마감 T가 정해져 있으면, 1차 시도가 실패했을 때 재시도는 `T - now()`가 양수일 때만, 그리고 그 남은 시간 안에서만 가능하다. 마감이 이미 지났으면 재시도하지 않고 바로 `DEADLINE_EXCEEDED`로 끝낸다.

```text
전체 마감 T = 1000ms
  시도1: t=0 시작, t=300ms 에 실패(UNAVAILABLE)
  백오프 50ms 대기 → t=350ms
  남은 예산 = 1000 - 350 = 650ms > 0 → 시도2 가능 (이 650ms 안에서)
  시도2: t=350 시작, t=900ms 에 실패
  백오프 → t=1000ms 도달 → 남은 예산 0 → 더는 재시도 안 함 → DEADLINE_EXCEEDED
```

gRPC의 내장 재시도 정책은 이 "남은 deadline 안에서만 재시도"를 자동으로 지킨다. 또 deadline 초과(`DEADLINE_EXCEEDED`)는 보통 재시도 대상에 넣지 않는다. 마감을 이미 넘긴 호출을 재시도해 봐야 예산이 없기 때문이다. (재시도 정책의 상세, 재시도 가능 코드, 백오프 구성은 [[13 - 안정성 - Retry Health Check Keepalive]]에서 다룬다.)

### keepalive/커넥션과의 구분

deadline과 헷갈리기 쉬운 것이 keepalive다. 둘 다 "오래 안 끝나는 상황"에 관여하지만 층위가 다르다.

| 구분 | deadline | keepalive |
|------|----------|-----------|
| 대상 | 개별 RPC(논리적 호출) | 커넥션(물리적 연결) |
| 질문 | "이 호출이 마감 안에 끝났나?" | "이 커넥션의 상대가 아직 살아 있나?" |
| 신호 | grpc-timeout, RST_STREAM(CANCEL) | HTTP/2 PING 프레임 |
| 결과 | DEADLINE_EXCEEDED | 죽은 커넥션 감지/종료 |
| 적용 | 모든 RPC | 특히 장수명 스트림/유휴 커넥션 |

deadline은 "이 요청이 너무 오래 걸리나?"를, keepalive는 "연결 자체가 끊긴 줄도 모르고 매달려 있나?"를 다룬다. 장수명 스트림에는 짧은 deadline 대신 keepalive를 쓰는 이유가 이것이다. 정상적으로 길게 열려 있어야 하므로 deadline으로 끊으면 안 되지만, 상대가 조용히 죽었는지는 PING으로 확인해야 한다. 자세한 keepalive 동작은 [[13 - 안정성 - Retry Health Check Keepalive]]를 보라.

### 채널/스텁 수준과 호출 수준

deadline은 보통 호출(call) 단위로 건다. 채널·스텁의 생명주기 자체는 [[07 - 채널 스텁 커넥션 생명주기]]에서 다루는데, 핵심만 말하면 deadline은 커넥션을 닫지 않는다. RPC 하나가 마감을 넘겨 그 스트림이 RST_STREAM으로 닫혀도, 그 위의 HTTP/2 커넥션과 채널은 멀쩡히 살아 다른 RPC에 재사용된다. deadline은 스트림 수준의 이벤트이지 커넥션 수준의 이벤트가 아니다.

---

## 10. 안티패턴 모음

실무에서 반복적으로 보이는 deadline/취소 관련 실수를 모았다.

### (1) 서버가 컨텍스트를 무시하고 끝까지 계산

가장 흔하고 가장 비싼 실수다. 클라이언트는 이미 취소했거나 마감을 넘겼는데, 서버 핸들러는 `ctx`를 한 번도 들여다보지 않고 무거운 계산을 끝까지 한다. 아무도 받지 않을 결과를 만들려고 CPU와 DB를 태운다.

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

### (2) 받은 컨텍스트를 버리고 빈 컨텍스트로 하위 호출 (전파 끊기)

5절에서 본 그대로. `context.Background()`나 `Context.ROOT`로 하위 호출을 새로 만들면 상류 마감이 증발한다. "왜 우리 서비스가 좀비 작업으로 가득하지?"의 가장 흔한 원인이다.

### (3) deadline을 무한대로 두기 (또는 아예 안 걸기)

7절의 위험 그 자체. "일단 안 끊기게 넉넉히"라는 유혹은 결국 무한 대기와 연쇄 장애로 돌아온다. 합리적 기본값을 인터셉터로 강제하라.

### (4) 취소/마감을 에러로 오인해 로깅 폭주

취소(`CANCELLED`)와 마감 초과(`DEADLINE_EXCEEDED`)는 정상적인 운영의 일부다. 사용자가 탭을 닫거나, 상위가 마감을 넘기거나, 더 빠른 응답이 와서 나머지를 취소하는 일은 늘 일어난다. 이걸 서버에서 에러 레벨로 시끄럽게 로깅하면, 로그가 의미 없는 취소 메시지로 도배되고 정작 진짜 에러가 묻힌다.

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

마감 초과가 비정상적으로 늘었는지는 로그가 아니라 **메트릭**으로 추적하는 편이 낫다. `DEADLINE_EXCEEDED` 비율이 평소보다 치솟으면 하위 서비스가 느려졌다는 신호다.

### (5) `defer cancel()` 누락 (Go 한정)

`context.WithTimeout`/`WithCancel`이 돌려준 `cancel` 함수를 호출하지 않으면, 그 컨텍스트와 연결된 타이머·고루틴이 부모 컨텍스트가 끝날 때까지 회수되지 않아 누수가 생긴다. Go의 `go vet`이 이 실수를 잡아 준다. **`ctx, cancel := ...; defer cancel()`는 한 묶음으로 외워야 한다.**

### (6) 스트림에 잘못된 짧은 deadline

장수명 스트림(알림 구독, 채팅)에 단항 호출처럼 짧은 deadline을 거는 실수. 정상 스트림이 마감에 걸려 끊긴다. 장수명 스트림은 deadline 대신 keepalive로 관리하라(9절).

### (7) 마감을 넘긴 뒤에도 재시도

`DEADLINE_EXCEEDED`를 받고도 같은 호출을 재시도하는 코드. 마감을 이미 넘겼으니 재시도해도 즉시 또 마감 초과다. 재시도는 남은 예산이 있을 때만(9절).

---

## 11. 종합 예제: deadline·전파·취소·조기 종료를 한 번에

지금까지의 조각을 모은, 동작 가능한 수준의 예제다. proto부터 보자.

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

**클라이언트(게이트웨이): 800ms 마감을 걸고 호출**:

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

**주문 서비스(서버이자 클라이언트): 받은 ctx를 전파하고, 조기 종료**:

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

**시간선으로 본 정상/마감 초과 두 경우**:

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

이 한 장의 코드가 이 챕터의 모든 주제를 담는다. 클라이언트가 절대 마감 T를 정하고(1, 2절), 그 마감이 ctx 전파로 하위 서비스까지 흐르며 매 홉마다 `grpc-timeout`으로 재환산되고(2, 3절), 마감 도달 시 RST_STREAM과 컨텍스트 취소로 사슬 전체가 멈추고(4절), 서버는 ctx를 들여다보며 협조적으로 조기 종료한다(5, 10절).

---

## 핵심 요약

- **deadline은 절대 시각(언제까지), timeout은 상대 시간(얼마 동안)이다.** gRPC는 프로그래머에게 timeout 편의를 주되 진입점에서 즉시 deadline으로 변환해, 이후 모든 내부 동작을 "변하지 않는 마감 시각"이라는 불변량 위에서 처리한다.
- **와이어 위에서 마감은 `grpc-timeout` 헤더(정수+단위 H/M/S/m/u/n)로 표현된다.** 대문자 M=분, 소문자 m=밀리초이며, 정수는 최대 8자리다. 클라이언트는 매 홉마다 "지금 남은 시간"을 다시 환산해 인코딩한다. 머신 간에는 시계 동기화 문제를 피하려고 절대 시각이 아니라 상대 시간으로 보낸다.
- **deadline 전파는 하나의 마감을 호출 사슬 전체가 공유하게 한다.** 이로써 호출자가 이미 포기한 일을 피호출자가 계속 붙들고 있는 좀비 작업과 예산 낭비를 막는다. 단, 전파는 자동이 아니라 **서버가 받은 컨텍스트를 모든 하위 호출·IO에 흘려보낼 때만** 일어난다.
- **취소는 HTTP/2 RST_STREAM(CANCEL, 에러코드 0x8)으로 전달되고, 서버 컨텍스트를 취소 상태로 만든다.** gRPC 런타임은 신호만 줄 뿐 핸들러를 강제 중단하지 않으므로, 서버 코드가 협조적으로 컨텍스트를 확인하고 멈춰야 한다.
- **deadline 초과는 "예약된 취소"다.** 같은 RST_STREAM·컨텍스트 취소 배관을 재사용하되 방아쇠가 시계이며, 호출자에게는 `DEADLINE_EXCEEDED`(코드 4), 명시적 취소는 `CANCELLED`(코드 1)로 보고된다.
- **언어별로 Go는 `context.Context`(WithTimeout/WithDeadline/WithCancel, ctx.Done(), ctx.Err()), Java는 `Context`/`Deadline`/`CancellableContext`로 이 추상을 구현한다.** 공통 규칙은 단 하나: 받은 마감/취소를 들고 다니다가 모든 하위로 흘려보내라.
- **양방향 스트림에서 취소는 대화 자체를 끝낸다.** 한쪽의 취소가 스트림 전체를 종료시키므로 양쪽 모두 송신 전 스트림 생존을 확인해야 한다. 장수명 스트림에는 짧은 deadline 대신 keepalive를 쓴다.
- **모든 RPC에 deadline을 걸어야 한다.** 없으면 무한 대기 → 자원 고갈 → 연쇄 장애로 번진다. 기본값이 무한대인 경우가 많으므로(가장 위험한 기본값) 인터셉터로 합리적 기본 마감을 강제하라.
- **예산 설계는 너무 빡빡하면 허위 DEADLINE_EXCEEDED와 재시도 폭풍을, 너무 느슨하면 좀비 작업과 늦은 실패를 부른다.** 사용자 체감 마감에서 출발해 꼬리 지연 + 여유로 잡고, 상류를 하류보다 약간 길게 둔다.
- **재시도는 남은 예산 안에서만 한다.** deadline은 RPC(스트림) 수준 사건이지 커넥션 수준 사건이 아니다 — keepalive와 혼동하지 말 것.

---

## 연결 노트

- [[04 - HTTP2 깊이 보기 - 전송 계층]] — `grpc-timeout` 헤더가 실리는 HEADERS 프레임, 취소를 전달하는 RST_STREAM 프레임, 멀티플렉싱과 스트림 개념의 토대.
- [[05 - 통신의 4가지 방식 - Unary와 Streaming]] — 스트리밍에서 deadline이 스트림 전체 수명에 걸리는 방식과 양방향 취소의 의미.
- [[08 - 메타데이터와 인터셉터]] — `grpc-timeout`도 메타데이터의 일종이며, 인터셉터로 기본 deadline을 강제 주입하거나 취소/마감을 관측·로깅하는 패턴.
- [[09 - 에러 모델 - 상태 코드와 Rich Error]] — `DEADLINE_EXCEEDED`(4)와 `CANCELLED`(1)의 자리, 상태 코드 체계 전반.
- [[07 - 채널 스텁 커넥션 생명주기]] — deadline은 스트림을 닫을 뿐 커넥션·채널은 유지된다는 구분.
- [[13 - 안정성 - Retry Health Check Keepalive]] — 재시도가 남은 예산 안에서만 일어나는 규칙, deadline과 keepalive의 층위 차이, 장수명 스트림 관리.
- [[15 - 성능과 Flow Control 백프레셔]] — 취소/마감이 흐름 제어·백프레셔와 만나는 지점(읽지 않는 수신자, 멈춘 스트림).
