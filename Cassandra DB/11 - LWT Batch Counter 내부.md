---
title: LWT, Batch, Counter 내부 - Paxos 경량 트랜잭션
date: 2026-06-26
tags: [cassandra, lwt, paxos, batch, counter, 학습노트]
---

Cassandra를 처음 깊게 만지는 RDB 출신 엔지니어가 가장 먼저 던지는 질문은 거의 정해져 있다. "그래서 트랜잭션은 어떻게 하나요?", "여러 INSERT를 묶어서 한 번에 처리하려면 BATCH 쓰면 되죠?", "조회수나 잔액 같은 카운터는 그냥 `count = count + 1` 하면 되나요?" 이 세 질문은 각각 LWT, BATCH, Counter로 이어지고 셋 다 RDB의 직관을 그대로 적용하면 **반드시** 사고가 난다.

[[01 - Cassandra란 무엇인가]]에서 보았듯, Cassandra의 출발점은 "가용성(availability)과 분할 내성(partition tolerance)을 위해 강한 일관성(strong consistency)과 전역 트랜잭션을 포기한다"는 AP 선택이었다. [[04 - Tunable Consistency]]는 그 포기를 정량 다이얼로 되돌리는 법을 다뤘다. 이 장은 거기서 한 발 더 나아간다. **읽고-판단하고-쓰는(read-modify-write) 원자적 연산이 정말로 필요할 때**, Cassandra는 무엇을 해주고, 그 대가가 얼마이며, 어디서 멈춰야 하는지를 본다.

이 장이 답할 질문들:

- LWT는 도대체 무엇을 보장하길래 "선형화(linearizable)"라는 비싼 단어를 쓰는가? 내부에서 Paxos가 정확히 몇 번 왕복(round trip)하는가?
- 왜 같은 파티션에 동시 LWT를 때리면 처리량이 절벽처럼 떨어지는가?
- `SERIAL`과 `LOCAL_SERIAL`은 `QUORUM`/`LOCAL_QUORUM`과 무엇이 다른가?
- BATCH는 성능 최적화가 아니다 — 그럼 정확히 무엇인가? 언제 쓰면 안티패턴인가?
- Counter는 왜 "위험한" 기능인가? 타임아웃 한 번에 잔액이 틀어질 수 있는 이유는?
- 결제 시스템 관점에서, 언제 이 셋을 쓰고 언제 RDB/외부 시스템으로 넘겨야 하는가?

---

## 왜 Cassandra에 "조건부 쓰기"가 필요해졌나

기본 Cassandra 쓰기는 **last-write-wins(LWW)** 다. 같은 셀(cell)에 두 개의 쓰기가 들어오면, 타임스탬프가 더 큰 쪽이 이긴다. 충돌을 "감지"하지 않고 "덮어쓴다". 이건 [[08 - Storage Engine 내부]]의 immutable cell + timestamp 모델에서 자연스럽게 따라오는 성질이고 분산 환경에서 락(lock) 없이 수렴(convergence)을 보장하는 영리한 방법이다.

문제는 이 모델이 다음 질문에 답하지 못한다는 점이다.

> "이 `user_id`가 **아직 없을 때만** INSERT하라. 누군가 0.1초 전에 같은 아이디로 가입했다면 실패시켜라."

LWW로는 불가능하다. 두 사용자가 동시에 `shawn`이라는 아이디로 가입을 시도하면, 둘 다 INSERT가 "성공"하고 타임스탬프 큰 쪽이 조용히 다른 쪽을 덮어쓴다. 둘 다 "가입 완료" 화면을 본다. 데이터는 하나만 남는다. **유일성(uniqueness) 위반을 감지조차 못 한다.**

RDB라면 `UNIQUE` 제약 + 트랜잭션이 이 문제를 한 줄로 해결한다. 하지만 그 한 줄 뒤에는 단일 노드(혹은 합의된 리더)가 전역 순서를 강제한다는 전제가 깔려 있다. Cassandra에는 그런 중앙 권위가 없다. 모든 복제본(replica)이 대등하다.

그래서 Cassandra는 **합의 알고리즘(consensus algorithm)** 을 끌어온다. 특정 파티션 키를 두고 복제본들이 "지금 이 값을 바꿔도 되는가"를 표결로 정한다. 이것이 LWT(Lightweight Transaction)이고 그 엔진은 **Paxos** 다.

여기서 "Lightweight"라는 이름은 함정이다. RDB의 무거운 ACID 트랜잭션에 비해 "가볍다"는 상대적 표현일 뿐, 일반 Cassandra 쓰기에 비하면 **압도적으로 무겁다.** 이름에 속지 말 것.

```text
RDB의 직관:                          Cassandra의 현실:
  BEGIN;                               INSERT ... IF NOT EXISTS;
  SELECT ... FOR UPDATE;       →       (단일 행/단일 파티션에 대한
  UPDATE ...;                           조건부 원자 연산 1개,
  COMMIT;                               Paxos 합의로 구현)
  ↑ 여러 행·여러 테이블 가능            ↑ 한 파티션 안에서만, 한 연산만
```

LWT는 RDB 트랜잭션의 일반화된 대체물이 아니다. **단일 파티션 compare-and-set(CAS)** 한 방일 뿐이다. 이 한계를 처음부터 명확히 새겨야 한다.

---

## LWT의 표면: 무엇을 쓸 수 있나

내부로 들어가기 전에, CQL 표면을 먼저 정리하자. LWT는 `IF` 절로 표현한다.

```cql
-- 1) 존재하지 않을 때만 INSERT (유일성 보장)
INSERT INTO users (user_id, email, created_at)
VALUES ('shawn', 'shawn@example.com', toTimestamp(now()))
IF NOT EXISTS;

-- 2) 현재 값이 기대값일 때만 UPDATE (상태 머신 전이)
UPDATE orders
SET status = 'SHIPPED'
WHERE order_id = 'ord-1234'
IF status = 'PAID';

-- 3) 존재할 때만 DELETE
DELETE FROM sessions
WHERE session_id = 'sess-99'
IF EXISTS;

-- 4) 여러 조건 동시 (AND만 가능, OR 없음)
UPDATE inventory
SET stock = 9
WHERE sku = 'SKU-1'
IF stock = 10 AND reserved = false;
```

LWT가 성공/실패를 어떻게 알려주는지가 중요하다. 일반 쓰기는 응답 본문이 비어 있지만 LWT는 항상 `[applied]` 컬럼을 포함한 한 행을 돌려준다.

```cql
cqlsh> UPDATE orders SET status='SHIPPED'
   ...> WHERE order_id='ord-1234' IF status='PAID';

 [applied] | status
-----------+--------
     False |   PAID   -- 조건 불일치: 적용 안 됨, 현재 실제 값(PAID 아님)을 함께 반환
```

`[applied]`가 `False`면 Cassandra는 **현재 실제 값을 같이 돌려준다.** 여기가 요점이다. 애플리케이션은 이 반환값을 보고 "내가 본 상태가 틀렸구나, 재시도하거나 분기하자"를 결정한다. 이것이 CAS(compare-and-set)의 본질이다. 비교가 실패하면 현재 값을 알려주어 호출자가 재시도 루프를 돌게 한다.

> **흔한 함정:** LWT의 결과를 일반 쓰기처럼 "에러냐 아니냐"로만 보면 `[applied]=False`를 놓친다. 드라이버에 따라 `wasApplied()` 같은 메서드를 반드시 확인해야 한다. `[applied]=False`는 **에러가 아니라 정상 응답**이다. 예외가 안 났다고 성공한 게 아니다.

LWT가 걸리는 컬럼은 정적 컬럼(static column), 일반 컬럼, primary key 존재 여부(`IF EXISTS`/`IF NOT EXISTS`) 등이다. 단, 조건은 모두 **같은 파티션** 안에 있어야 한다. LWT는 절대 파티션 경계를 넘지 못한다. 이건 우연이 아니라 Paxos가 파티션 키 단위로 동작하기 때문이며, 다음 절의 주제다.

---

## Paxos: LWT가 내부에서 실제로 하는 일

Cassandra의 LWT는 **단일 결정 Paxos(single-decree Paxos)** 의 변형을 파티션 키 단위로 실행한다. "이 파티션의 다음 변경은 무엇인가"라는 하나의 명제를 놓고 복제본들이 합의한다. Multi-Paxos처럼 안정적 리더(stable leader)를 두지 않기 때문에, 매 LWT마다 처음부터 합의를 다시 세운다. 이게 비용의 근원이다.

### 4단계, 그리고 왜 4배인가

LWT 한 번은 개념적으로 **네 라운드**를 거친다. 코디네이터(coordinator)가 해당 파티션의 복제본들과 표결하는 과정이다.

```text
일반 쓰기 (QUORUM):                LWT (SERIAL):
  Client                            Client
    │ write                          │ conditional write
    ▼                                ▼
  Coordinator                       Coordinator
    │  ──► replica (×N)               1) PREPARE / PROMISE   ──► replicas  ◄── (왕복 1)
    │  ◄── ack (quorum)                  "내가 ballot T로 제안해도 되나?"
    ▼                                 2) READ (현재값 조회)   ──► replicas  ◄── (왕복 2)
  done (왕복 1번)                       "조건(IF) 평가용 최신값 수집"
                                      3) PROPOSE / ACCEPT    ──► replicas  ◄── (왕복 3)
                                         "이 값으로 정하자"
                                      4) COMMIT              ──► replicas  ◄── (왕복 4)
                                         "확정, 실제 테이블에 반영"
                                      done (왕복 ~4번)
```

각 단계를 풀어보자.

1. **Prepare / Promise.** 코디네이터가 단조 증가하는 **ballot(제안 번호, 보통 시간 기반 UUID)** 을 생성해 복제본들에게 "이 ballot으로 제안하겠다"고 알린다. 복제본은 자기가 본 적 없는 더 큰 ballot이면 "약속(promise)"하고 동시에 자기가 알고 있는 **아직 commit되지 않은 in-progress Paxos 제안**이 있으면 그걸 응답에 실어 보낸다. 과반(quorum)의 promise를 못 받으면 실패하고 재시도한다.

2. **Read (현재값 조회).** `IF` 조건을 평가하려면 그 파티션의 **현재 진짜 값**을 알아야 한다. 코디네이터는 복제본들로부터 최신 데이터를 읽는다. 여기서 read repair에 준하는 동기화가 일어난다. LWT는 내부에 **읽기 한 라운드를 품고 있다.** (이래서 LWT는 "읽고-판단하고-쓰는" 연산이라고 부른다.)

3. **Propose / Accept.** 조건이 참이면, 코디네이터는 실제로 적용할 변경(mutation)을 같은 ballot으로 제안한다. 과반의 복제본이 이 제안을 자기 Paxos 상태에 기록(accept)하면 합의가 성립한다. 조건이 거짓이면 여기서 멈추고 `[applied]=False` + 현재값을 반환한다.

4. **Commit.** 합의된 변경을 복제본들이 실제 테이블(memtable/commitlog, [[10 - Write Path와 Read Path]] 참고)에 반영하고 Paxos 임시 상태를 정리한다. 이후 이 변경은 일반 데이터처럼 읽힌다.

일반 QUORUM 쓰기가 사실상 **1 왕복**인 데 비해, LWT는 **약 4 왕복**이다. 그래서 흔히 "LWT는 일반 쓰기의 약 4배 비용"이라고 말한다. 이건 단순 네트워크 왕복 수 비교이고 실제로는 read 단계의 디스크 접근, Paxos 상태를 위한 추가 저장, contention 시 재시도까지 겹쳐 체감 비용은 4배보다 더 클 수 있다.

> **버전 주의 (Cassandra 5.0):** 위 "4 왕복"은 전통적 Paxos(이하 **Paxos v1**) 기준이다. Cassandra 4.1부터 최적화된 **Paxos v2**(`cassandra.yaml`의 `paxos_variant: v2`)가 도입되어 5.0에서도 쓸 수 있다. v2는 prepare/promise 단계에서 조건 평가용 read를 함께 처리하고 불필요한 commit 왕복을 줄이는 식으로, 경합이 없는 정상 경로의 왕복 수를 대략 절반(2~3 수준)까지 깎는다. 다만 이는 클러스터 설정(그리고 v1↔v2 전환 절차)에 의존하므로, "LWT는 일반 쓰기보다 몇 배 비싸다"는 정확한 상수가 아니라 **자릿수 감각**으로만 받아들여야 한다. 그리고 v1이든 v2든 5.0의 LWT는 여전히 **Paxos** 기반이다 — 범용 멀티키 트랜잭션을 노리는 Accord(CEP-15)는 5.0 GA에 포함되지 않은 이후 버전의 방향이며, 이 장의 모든 한계(단일 파티션, CAS 한 방)는 5.0 기준 그대로다.

> **왜 read가 중간에 끼는가?** RDB의 `SELECT ... FOR UPDATE`는 락을 잡고 읽는다. Cassandra에는 락이 없으므로, "읽은 값이 합의 시점에도 유효함"을 Paxos ballot 순서로 보장한다. ballot이 read와 propose를 하나의 논리적 원자 단위로 묶는다. 그래서 read를 합의 프로토콜 **안**에서 해야지, 합의 밖에서 따로 `SELECT` 하면 그 사이에 값이 바뀌어 race가 난다.

### Paxos 상태는 어디에 사는가

Cassandra는 Paxos의 in-progress 상태(promised ballot, accepted proposal 등)를 `system.paxos` 테이블에 저장한다. commit이 끝나면 정리되지만 이 추가 저장·정리 자체가 LWT의 숨은 오버헤드다. Paxos 상태에는 TTL이 걸려 일정 시간 뒤 GC되며, 관련해 `cas_contention_timeout`(기본 1000ms 수준) 같은 타임아웃이 별도로 존재한다.

### SERIAL vs LOCAL_SERIAL: 합의의 범위

LWT는 일반 쓰기의 consistency level과는 **별개의 축**인 **serial consistency** 를 갖는다. 두 가지뿐이다.

| serial CL | 합의에 참여하는 복제본 범위 | linearizable 보장 범위 | 비용/지연 |
|---|---|---|---|
| `SERIAL` | **모든 DC**의 복제본 과반 | 전역(글로벌) 선형화 | DC 간 왕복 포함 → 매우 느림 |
| `LOCAL_SERIAL` | **로컬 DC**의 복제본 과반 | 로컬 DC 내 선형화 | DC 내부만 → 상대적으로 빠름 |

[[03 - 복제 전략과 데이터센터]]에서 본 멀티 DC 토폴로지를 떠올리면 이해가 쉽다. `SERIAL`은 지구 반대편 DC의 복제본까지 표결에 끌어들이므로, 한 번의 LWT에 대륙 간 왕복(수십~수백 ms)이 4번 들어간다. 글로벌 유일성(예: 전 세계에서 유일한 이메일)이 정말 필요한 게 아니라면 거의 항상 `LOCAL_SERIAL`을 쓴다.

> **혼용 금지 함정:** 같은 파티션(같은 데이터)에 대해 어떤 연산은 `SERIAL`로, 어떤 연산은 `LOCAL_SERIAL`로 섞어 던지면 **선형화 보장이 깨진다.** 두 레벨은 합의에 참여하는 quorum 집합이 다르므로, 서로 겹치지 않는 quorum이 각자 다른 값을 확정할 수 있다. 한 데이터셋에는 읽기·쓰기 모두 **하나의 serial 레벨로 일관되게** 가야 한다. 또한 `LOCAL_SERIAL`은 어디까지나 로컬 DC 내부의 선형화만 보장하므로, DC 장애로 다른 DC에 페일오버하는 순간 그 보장은 연속되지 않는다는 점도 설계 시 감안해야 한다.

짚어둘 점이 있다. LWT 한 문장에는 **두 개의 consistency level**이 관여한다.

- **serial consistency** (`SERIAL`/`LOCAL_SERIAL`): Paxos 합의(prepare/read/propose)의 범위.
- **일반 consistency** (`QUORUM`/`LOCAL_QUORUM` 등): commit 단계에서 실제 데이터를 몇 개 복제본에 확정 반영할지.

```cql
-- 드라이버 레벨 설정 예 (개념)
serial consistency = LOCAL_SERIAL
consistency        = LOCAL_QUORUM
```

흔한 실수가 LWT로 써놓고 정작 읽기는 일반 `ONE`으로 하는 경우다. 그러면 선형화가 깨진다. **LWT로 보장한 값을 선형적으로 읽으려면, 읽기도 `SERIAL`/`LOCAL_SERIAL`로 해야 한다.** 그래야 진행 중인 Paxos를 마무리(read-time repair)하고 가장 최근 합의값을 본다. LWT 쓰기 + 일반 QUORUM 읽기를 섞으면 "방금 LWT로 정한 값이 잠깐 안 보이는" 비선형 구간이 생길 수 있다.

> **함정:** "linearizable"은 공짜가 아니다. 쓰기를 LWT로 했어도, 읽기를 일반 CL로 하면 그 읽기는 선형화되지 않는다. 일관성은 **읽기·쓰기 양쪽의 CL 조합**으로 결정된다([[04 - Tunable Consistency]]의 `R + W > N` 직관을 serial 축으로 확장한 것).

---

## 경합(contention): LWT의 성능 절벽

LWT의 가장 위험한 성질은 **같은 파티션에 몰린 동시 LWT가 서로를 방해한다**는 점이다. Paxos는 본질적으로 "한 번에 하나의 제안만 통과"시키는 프로토콜이다. 동일 파티션 키에 N개의 LWT가 동시에 몰리면, ballot 경쟁이 벌어진다.

```text
같은 파티션 키에 동시 LWT 3개:

  ballot T1 ──prepare──► (promise)
  ballot T2 ──prepare──► 더 큰 ballot 등장! T1의 propose가 거부됨
  ballot T1 ──propose──► REJECTED (이미 T2를 promise함)
  ballot T1 재시도: 더 큰 ballot T4 생성 ──prepare──► ...
  ballot T3 ──prepare──► 또 끼어듦 ...

  → live-lock에 가까운 상호 방해, 재시도 폭증
  → 처리량이 동시성 증가에 따라 오히려 "감소"하는 구간 발생
```

이것이 "성능 절벽(performance cliff)"이라 불리는 현상이다. 일반 쓰기는 동시성이 올라가면 처리량도 (자원 한계까지) 같이 올라간다. 그러나 **단일 파티션 LWT는 동시성이 어느 선을 넘으면 재시도·`WriteTimeoutException(CAS)`·`cas_contention_timeout` 초과가 폭증하며 처리량이 무너진다.** 한 핫 파티션(hot partition)에 LWT를 몰면, 그 파티션이 사실상 전역 직렬화 지점(global serialization point)이 되어 버린다.

교훈은 셋이다.

- **LWT는 경합이 낮은 곳에서만 빛난다.** 서로 다른 파티션 키로 LWT가 분산되면(예: `user_id`별 가입), 각 Paxos는 독립적이라 잘 확장된다.
- **하나의 인기 파티션에 여러 클라이언트가 LWT를 던지는 설계는 안티패턴이다.** 예: "전역 카운터를 LWT로 증가" → 모든 요청이 같은 파티션 → 절벽.
- 드라이버의 LWT 재시도는 **멱등하지 않을 수 있다.** `[applied]` 의미가 재시도마다 달라질 수 있으므로, 무지성 재시도는 위험하다(뒤의 Counter 절과 같은 결의 함정).

RDB와 대비하면 직관이 선다.

| | RDB (`SELECT FOR UPDATE`) | Cassandra LWT |
|---|---|---|
| 동시 충돌 처리 | 락 대기 (blocking) | ballot 경쟁 + 재시도 (optimistic-ish) |
| 핫스팟 | 락 큐가 길어짐, 하지만 진행은 함 | 재시도 폭증, 처리량 붕괴 가능 |
| 범위 | 여러 행/테이블 | 단일 파티션, 단일 CAS |
| 비용 | 트랜잭션 1회 | 일반 쓰기의 ~4배 + 재시도 |

> **결제 시스템 직관:** 결제 멱등키(idempotency key)당 한 번의 "선점"은 LWT의 이상적 사례다(키가 충분히 분산되므로). 반대로 "오늘 전체 결제 건수 카운터"를 LWT로 올리는 건 모든 결제 트래픽을 한 파티션에 직렬화하는 자살골이다.

---

## LWT를 정말 써야 할 때: 실전 패턴 3가지

### 1) 유일성 보장 — 회원 가입, 멱등키 선점

가장 정석적인 LWT 사용처. `IF NOT EXISTS`로 "처음 쓰는 사람만 성공"을 만든다.

```cql
-- 이메일을 파티션 키로 둔 lookup 테이블 (Query-First, [[05 - 데이터 모델링 1 - Query First]])
CREATE TABLE users_by_email (
    email      text PRIMARY KEY,
    user_id    uuid,
    created_at timestamp
);

INSERT INTO users_by_email (email, user_id, created_at)
VALUES ('shawn@example.com', uuid(), toTimestamp(now()))
IF NOT EXISTS;
-- [applied]=True  → 이 요청이 이메일을 선점
-- [applied]=False → 이미 누가 가입함, 현재 주인 정보 반환
```

`email`을 파티션 키로 두었기 때문에 각 이메일의 LWT는 서로 독립적이다. 핫스팟이 없다(특정 이메일에 동시 가입 폭주가 없는 한). 이것이 LWT가 잘 확장되는 전형이다.

결제 시스템의 **멱등키 선점**도 같은 구조다. `idempotency_key`를 파티션 키로 두고 `IF NOT EXISTS`로 "이 결제 요청을 처음 처리하는 주체"를 한 명만 뽑는다. 중복 요청은 `[applied]=False`로 걸러지고 이미 처리된 결과를 반환받는다.

### 2) 상태 머신 전이 — 주문/결제 상태

`IF status = '이전상태'`로 "유효한 전이만 허용"한다. [[13 - 실전 Event Sourcing on Cassandra]]의 상태 모델과 직결된다.

```cql
-- PAID 상태일 때만 SHIPPED로 (이중 배송 방지)
UPDATE orders SET status='SHIPPED', shipped_at=toTimestamp(now())
WHERE order_id='ord-1234'
IF status='PAID';

-- CANCELLED는 PAID에서만 가능, 이미 SHIPPED면 거부
UPDATE orders SET status='CANCELLED'
WHERE order_id='ord-1234'
IF status='PAID';
```

요점은 "현재 상태를 읽고 → 애플리케이션에서 판단하고 → 쓰기" 사이의 race를 LWT가 닫아준다는 데 있다. 일반 쓰기였다면 두 워커가 동시에 `PAID`를 읽고 둘 다 `SHIPPED`로 써서 이중 배송이 났을 것이다. LWT는 둘 중 하나만 성공시킨다.

다만 `order_id`별로 LWT가 분산되므로 확장성은 좋다. 한 주문에 수십 개 워커가 동시에 상태를 바꾸려 다투는 상황만 아니면 된다.

### 3) 재고 차감 — 가장 조심해야 할 사례

```cql
-- 현재 재고가 정확히 10일 때만 9로 (정수 CAS)
UPDATE inventory SET stock = 9
WHERE sku='SKU-1'
IF stock = 10;
```

재고는 단일 `sku`(파티션) 위의 단일 카운터이므로, **인기 상품의 동시 구매가 곧 동일 파티션 LWT 폭주** = 성능 절벽이다. 플래시 세일(flash sale)에서 한 상품에 LWT를 거는 건 위험하다. 그래서 실무에서는:

- 재고를 여러 샤드(shard)로 쪼개 LWT를 분산(예: `sku` + `shard_id` 100개)하거나,
- 애초에 재고를 **외부 인메모리(Redis 등)나 RDB**로 빼서 Cassandra LWT를 피하거나,
- 정합성보다 "약간의 오버셀(oversell) 허용 + 사후 보정" 비즈니스 결정을 내린다.

> 재고 차감을 LWT로 구현하기 전에 반드시 자문하라: "이 상품 하나에 초당 몇 건의 동시 차감이 들어오는가?" 수백 건이면 LWT는 무너진다. 단일 핫 파티션 LWT는 곧 단일 스레드 직렬 처리량의 상한에 갇힌다.

---

## BATCH의 진짜 의미 (가장 흔한 오해)

이제 이 장에서 오해가 가장 심한 주제로 간다. RDB 출신은 거의 100% BATCH를 이렇게 이해한다.

> (틀린 직관) "여러 INSERT/UPDATE를 BATCH로 묶으면 네트워크 왕복이 줄어서 **빨라진다.** JDBC batch처럼."

**Cassandra의 BATCH는 성능 최적화 도구가 아니다.** 이름이 같아서 생긴 가장 비싼 오해 중 하나다. Cassandra BATCH의 목적은 **원자성(atomicity)** 이지 처리량이 아니다. 오히려 대부분의 경우 BATCH는 처리량을 **떨어뜨린다.**

### Logged batch: batchlog로 사는 원자성

기본 BATCH(= logged batch)는 "묶인 모든 변경이 **전부 적용되거나 전부 적용되지 않거나**(all-or-nothing)"를 보장한다. 그 구현은 **batchlog** 다.

```text
Logged batch 처리 흐름:

  Client ── BATCH(4 mutations) ──► Coordinator
                                      │
            1) batchlog 기록  ───────►│  복제본 2곳에 batchlog 저장
               (직렬화된 batch 전체)   │  (coordinator가 죽어도 복구 가능하게)
                                      │
            2) 각 mutation을 ─────────►│  서로 다른 파티션/노드로 분배 전송
               대상 복제본으로 전송      │
                                      │
            3) 충분히 성공하면 ────────►│  batchlog에서 해당 batch 제거
                                      ▼
                                   done
```

코디네이터는 먼저 batch 전체를 **batchlog**(다른 노드 2곳에 복제)에 적어 둔다. 그 다음 개별 mutation들을 각자의 대상 복제본으로 보낸다. 만약 코디네이터가 중간에 죽어도, batchlog를 보유한 노드가 미완료 batch를 발견해 **남은 mutation을 재생(replay)** 한다. 그래서 "전부 적용"이 결국 보장된다.

여기서 보장되는 것과 **안 되는 것**을 정확히 구분하자.

| | Logged batch가 보장하는가? |
|---|---|
| 원자성(atomicity, all-or-nothing 결국 적용) | ✅ batchlog 재생으로 보장 |
| 격리(isolation, 중간 상태가 안 보임) | ❌ **단일 파티션 batch일 때만** |
| 순서·고립된 트랜잭션 | ❌ 다른 클라이언트는 부분 적용 중간 상태를 볼 수 있음 |
| 롤백(rollback) | ❌ Cassandra에 롤백 없음. "전부 적용"으로 수렴할 뿐 |

> **결정적 오해 정정:** Logged batch는 "실패하면 되돌린다"가 아니다. "결국 전부 적용된다"이다. 롤백은 없다. 그리고 다른 읽기 클라이언트는 batch가 절반만 적용된 **중간 상태를 볼 수 있다**(multi-partition인 경우). 즉 ACID의 A는 (eventual하게) 주지만 I는 일반적으로 주지 않는다.

### Multi-partition logged batch = 안티패턴

여기서 BATCH의 두 번째 큰 오해가 깨진다. "여러 파티션에 흩어진 쓰기를 BATCH로 묶으면 효율적"이라는 생각은 **정반대**다.

```text
좋은 batch (single partition):        나쁜 batch (multi partition):
  파티션 P 하나로 가는 mutation들        여러 파티션 P1,P2,...로 흩어진 mutation들
    ┌─────────────┐                      ┌─────────────────────────┐
    │ coordinator │                      │ coordinator             │
    │   = replica │                      │  batchlog 기록(추가 I/O) │
    │  한 번에 적용 │                      │  P1→node A               │
    │  (격리도 보장)│                      │  P2→node B               │
    └─────────────┘                      │  P3→node C ... 부하 집중   │
                                         └─────────────────────────┘
```

Multi-partition logged batch의 문제:

1. **batchlog 오버헤드.** 모든 mutation을 직렬화해 별도 노드 2곳에 먼저 적고 끝나면 지우는 추가 쓰기 I/O가 발생한다. 같은 쓰기를 개별로 보냈을 때보다 **느리다.**
2. **코디네이터 부하 집중.** 코디네이터가 여러 파티션의 복제본들에게 흩뿌리는 fan-out 허브가 된다. 이 노드의 CPU/네트워크가 병목이 된다.
3. **격리 없음.** 여러 파티션이므로 다른 클라이언트가 부분 적용 상태를 본다. 원자성도 "결국"일 뿐 즉시가 아니다.
4. **로그 경고.** Cassandra는 batch 크기가 `batch_size_warn_threshold`(기본 5KB)를 넘으면 WARN, `batch_size_fail_threshold`(기본 50KB)를 넘으면 batch를 **거부**한다. 큰 multi-partition batch를 던지면 이 한계에 부딪힌다.

multi-partition logged batch가 정당한 경우는 하나뿐이다. **여러 테이블(여러 denormalized view)에 같은 사실을 원자적으로 반영해야 할 때**다. [[05 - 데이터 모델링 1 - Query First]]에서 본 "같은 데이터를 쿼리별로 여러 테이블에 중복 저장" 패턴에서, "두 view가 동시에 갱신되어야 한다"를 보장하려고 batch를 쓴다. 이때조차 batch는 성능이 아니라 **정합성을 위해** 쓰며, mutation 수는 작게 유지해야 한다.

```cql
-- 정당한 multi-table batch: 같은 사실을 두 view에 원자적 반영
BEGIN BATCH
  INSERT INTO orders_by_id   (order_id, user_id, amount) VALUES ('o1','u1',1000);
  INSERT INTO orders_by_user (user_id, order_id, amount) VALUES ('u1','o1',1000);
APPLY BATCH;
-- 둘 다 적용되거나(결국) 둘 다 안 됨. denormalized view 간 정합성 목적.
```

### Unlogged batch: 같은 파티션 묶음에만

`UNLOGGED` 키워드를 붙이면 batchlog를 건너뛴다. 대신 **원자성 보장을 버리고** 오버헤드만 없앤다.

```cql
BEGIN UNLOGGED BATCH
  INSERT INTO events (pk, ck, v) VALUES ('p1', 1, 'a');
  INSERT INTO events (pk, ck, v) VALUES ('p1', 2, 'b');
  INSERT INTO events (pk, ck, v) VALUES ('p1', 3, 'c');
APPLY BATCH;
-- 모두 같은 파티션 키 'p1' → 코디네이터가 한 노드로 한 번에 보냄 → 실제로 효율적
```

unlogged batch가 의미 있는 **유일한** 경우는 **모든 mutation이 같은 파티션 키**일 때다. 이때는 코디네이터가 그 파티션의 복제본 한 세트로 단일 메시지를 보내므로, 네트워크 왕복이 실제로 줄고 단일 파티션이라 **격리(isolation)와 원자성**도 사실상 함께 얻는다(한 노드에서 한 번에 memtable에 반영되므로).

반대로 **서로 다른 파티션 키를 unlogged batch로 묶는 것은 명백한 안티패턴**이다. 원자성도 없고(batchlog 없음), 코디네이터 fan-out 부하만 떠안는다. 차라리 개별 비동기 쓰기를 병렬로 날리는 게 빠르다. Cassandra 자신도 이를 경계해서 하나의 unlogged batch가 일정 개수(`unlogged_batch_across_partitions_warn_threshold`, 기본 10개)를 넘는 파티션에 걸치면 로그에 WARN을 남긴다. 이 경고가 보이면 그 batch는 십중팔구 잘못 쓰이고 있다는 신호다.

| batch 종류 | 원자성 | 격리 | 언제 쓰나 | 흔한 오용 |
|---|---|---|---|---|
| **Logged, single-partition** | ✅ | ✅ | 한 파티션에 여러 행 원자 반영 | (드물게 적절) |
| **Logged, multi-partition** | ✅(결국) | ❌ | 여러 view 정합성(소량) | 성능 목적의 대량 묶기 ← 안티패턴 |
| **Unlogged, single-partition** | ✅(사실상) | ✅ | 같은 파티션 묶음 쓰기 | (적절) |
| **Unlogged, multi-partition** | ❌ | ❌ | (거의 없음) | "왕복 줄이려고" 묶기 ← 안티패턴 |

> **한 줄 요약:** BATCH로 성능을 얻으려면 **반드시 같은 파티션 키**여야 한다. 파티션이 흩어지는 순간 BATCH는 정합성 도구일 뿐이고, 그마저도 비싸다. "여러 파티션을 묶어 빠르게"는 존재하지 않는다 — 그건 개별 병렬 쓰기(`executeAsync` fan-out)가 한다.

### BATCH + LWT의 조합

한 가지 더 있다. logged batch 안에 LWT(`IF`)를 넣을 수 있지만 **모든 조건이 같은 파티션**이어야 한다. "단일 파티션 conditional batch"만 가능하다는 뜻이다. 이건 한 파티션 안에서 여러 행을 원자적·조건부로 바꾸는 강력한 도구지만, 역시 단일 파티션 Paxos이므로 경합 비용은 그대로다.

---

## Counter: 분산 환경에서 "더하기"가 어려운 이유

마지막 주제, Counter. "조회수 +1"처럼 단순해 보이지만 분산 환경에서 **덧셈은 의외로 가장 어려운 연산** 중 하나다.

### 왜 일반 컬럼으로는 안 되나

일반 Cassandra 쓰기는 LWW(덮어쓰기)다. "현재값 + 1"을 하려면 현재값을 읽어야 하는데, 읽고 쓰는 사이에 다른 쓰기가 끼면 증가분이 사라진다(lost update). 그래서 Cassandra는 별도의 `counter` 타입을 둔다. counter는 클라이언트가 **절대값을 지정할 수 없고 오직 증감(delta)만 표현**하는 특수 컬럼이다.

내부 표현을 정확히 짚자. counter는 단순히 "델타들을 로그처럼 쌓아 두는" 게 아니다. 각 복제본은 자기 몫의 누적값을 담은 **counter context** — 대략 `(host_id, logical_clock, count)` 튜플들의 묶음 — 를 들고 있고 읽을 때 이 복제본별 누적분을 병합해 최종 합계를 만든다. 구조적으로는 **상태 기반(state-based) CRDT(PN-counter)** 에 가깝다. 클라이언트 API에서 델타만 보이는 것과, 저장소가 복제본별 누적 상태를 들고 있는 것은 별개임을 구분해야 뒤에 나올 read-before-write와 비멱등성이 자연스럽게 이해된다.

```cql
CREATE TABLE page_views (
    page_id text PRIMARY KEY,
    views   counter        -- 일반 컬럼과 함께 둘 수 없음 (PK 제외)
);

UPDATE page_views SET views = views + 1 WHERE page_id = 'home';
UPDATE page_views SET views = views - 1 WHERE page_id = 'home';
```

제약부터 못 박자.

- **counter 컬럼은 일반 컬럼과 같은 테이블에 섞을 수 없다.** primary key 컬럼을 제외하면, 한 테이블의 모든 non-PK 컬럼이 counter이거나 모두 아니거나 둘 중 하나다. (counter의 충돌 병합 규칙이 일반 셀과 근본적으로 다르기 때문.)
- counter에는 `INSERT`가 없다. 오직 `UPDATE`의 `+`/`-` 만. TTL도 못 건다.
- counter 값을 직접 특정 숫자로 "세팅"할 수 없다(`= 100` 불가). 오직 상대 증감만.

### 내부: read-before-write 경로

counter `+1`의 내부는 일반 쓰기보다 무겁다. 코디네이터가 받은 increment를 처리하려면, **선택된 복제본(leader replica)이 현재 누적값을 읽어 delta를 더한 뒤 새 누적값을 기록**해야 한다. counter 쓰기는 내부적으로 **read-before-write** 다.

```text
일반 쓰기:                  Counter 쓰기:
  값을 그냥 적는다            1) 코디네이터 → 리더 복제본 선택
  (blind write)              2) 리더가 현재 counter 로컬값 read
                             3) delta(+1) 적용한 새 값 계산
                             4) 새 값을 자신 + 다른 복제본에 기록(복제)
                             ↑ read 단계 때문에 일반 쓰기보다 느리고
                                idempotent하지 않음
```

이 read 단계가 두 가지 결과를 낳는다. 첫째, counter 쓰기는 일반 쓰기보다 느리다(특히 디스크에서 현재값을 읽어야 할 때). 둘째, 더 중요하게는 **비멱등(non-idempotent)** 이다.

### 비멱등성: 타임아웃 재시도의 함정

이것이 counter의 가장 위험한 본질이다. `views = views + 1`을 보냈는데 **타임아웃**이 났다고 하자. 이 요청이 적용되었는지 안 되었는지 클라이언트는 **알 수 없다.**

```text
시나리오:
  Client ── "+1" ──► Coordinator ── 적용 성공 ──► (그러나 ack가 네트워크에서 유실)
  Client: 타임아웃! 재시도 ── "+1" ──► 또 적용됨
  결과: 실제로는 +2 가 되어버림 (중복 증가)

  vs. 반대로 처음에 진짜 실패였다면:
  재시도 안 하면 → +0 (증가 누락)
```

- 재시도하면 → **중복 카운트(over-count)** 위험.
- 재시도 안 하면 → **누락(under-count)** 위험.

일반 `INSERT`/`UPDATE`는 멱등하다. 같은 타임스탬프·같은 값으로 다시 써도 결과가 같다([[10 - Write Path와 Read Path]]의 timestamp 기반 수렴). 그래서 타임아웃 시 안전하게 재시도할 수 있다. 그러나 **counter의 `+1`은 같은 연산을 두 번 적용하면 결과가 달라진다.** 그래서 드라이버는 기본적으로 counter 쓰기를 **자동 재시도하지 않는다**(혹은 매우 보수적으로 다룬다). 타임아웃 시 "적용됐는지 모름" 상태를 애플리케이션이 감수해야 한다.

> **결제 시스템 직관:** 이래서 **돈(잔액)을 counter로 관리하면 안 된다.** 타임아웃 한 번에 잔액이 한 단위 틀어질 수 있고, 그 오차를 사후에 검증·교정할 방법이 counter 자체에는 없다(절대값을 못 읽어와서 "맞춰" 쓸 수도 없다). counter는 **근사치여도 되는 통계** — 조회수, 좋아요, 대략적 카운트 — 에나 적합하다. 정확성이 돈과 직결되면 counter는 금물.

### 4.0+ 개선점과 한계

Cassandra의 counter는 역사적으로 악명 높았다. 2.1 이전 구버전 counter는 복제본 간 read-during-write 충돌, replay 시 중복 누적 등으로 값이 드리프트(drift)하는 버그가 잦았다. 2.1에서 **counter 구현이 전면 재작성**되어 안정성이 크게 올랐고(리더 복제본 기반의 명확한 조정), 이후 4.0/5.0 계열에서도 안정성·성능 측면 개선이 이어졌다.

다만 **근본적 비멱등성은 사라지지 않았다.** read-before-write라는 본질 때문에, 4.0+ counter도 여전히:

- 타임아웃 시 적용 여부 불확실 → 무지성 재시도 금지.
- 일반 컬럼과 혼합 불가.
- 정확한 절대값 보장이 아니라 "정상 경로에서는 정확, 장애 경로에서는 근사" 수준.

> (확신이 낮은 부분은 표시) 구체적인 버전별 내부 변경 디테일(예: 5.0에서의 특정 최적화)은 버전 릴리스 노트로 확인할 것. 여기서 단정할 수 있는 것은 "2.1 재작성으로 신뢰성이 크게 올랐으나 비멱등성·혼합 불가·재시도 위험이라는 본질적 제약은 그대로"라는 점이다.

대안 패턴:

- 정확한 합계가 필요하면 → **개별 이벤트를 일반 테이블에 append**(멱등)하고, 합계는 배치/스트리밍 집계로 계산. [[13 - 실전 Event Sourcing on Cassandra]]의 이벤트 누적 + 집계가 정석.
- 근사치로 충분하면 → counter 사용 OK. 단, 핫 파티션 주의(인기 페이지의 counter는 한 파티션에 쓰기 집중 → write 핫스팟).

---

## 결정 가이드: 언제 Cassandra가 아니라 RDB/외부 시스템인가

이 세 기능을 관통하는 메타 교훈은 같다. **Cassandra는 일부러 포기한 강한 일관성을 LWT/BATCH/Counter로 부분적으로 되사올 수 있지만, 그 값은 비싸고 한계가 뚜렷하다.** 그래서 "되사올 수 있다"와 "되사와야 한다"는 다르다. 결제 시스템 엔지니어의 판단 기준을 정리한다.

```text
이 연산이 필요하다 ──┐
                    │
   ┌────────────────┴───────────────────────────┐
   │ 단일 파티션 + 낮은 경합 + 가끔만 조건부 쓰기?  │── 예 ──► Cassandra LWT 적합
   └────────────────┬───────────────────────────┘          (멱등키 선점, 상태전이)
                    │ 아니오
   ┌────────────────┴───────────────────────────┐
   │ 여러 행/여러 테이블에 걸친 진짜 트랜잭션?      │── 예 ──► RDB (또는 별도 합의 계층)
   │ (multi-row ACID, rollback 필요)              │          Cassandra BATCH로는 부족
   └────────────────┬───────────────────────────┘
                    │ 아니오
   ┌────────────────┴───────────────────────────┐
   │ 한 핫 파티션에 고빈도 동시 증감/조건부 쓰기?  │── 예 ──► Redis/RDB 또는 샤딩
   │ (인기상품 재고, 전역 카운터)                  │          Cassandra 핫 파티션은 절벽
   └────────────────┬───────────────────────────┘
                    │ 아니오
   ┌────────────────┴───────────────────────────┐
   │ 정확한 금액/잔액?                            │── 예 ──► RDB (트랜잭션 + 정합성)
   │ counter는 비멱등 → 돈에 부적합               │          이벤트 append + 집계도 대안
   └─────────────────────────────────────────────┘
```

구체적 판단 기준:

1. **다중 행/다중 테이블 ACID + 롤백이 필요한가?** 그렇다면 Cassandra가 아니다. LWT는 단일 파티션 CAS 한 방, BATCH는 격리·롤백을 안 준다. 진짜 트랜잭션이 필요하면 RDB(혹은 Cassandra 위 합의 계층)다. 결제의 "원장(ledger) 차변/대변 동시 기입"은 RDB의 영역이다.

2. **경합이 한 파티션에 집중되는가?** LWT 절벽과 counter 핫스팟은 같은 뿌리(단일 파티션 직렬화)다. 트래픽이 한 키에 몰리면 Cassandra는 그 키의 처리량 상한에 갇힌다. Redis(원자 INCR/Lua), RDB row lock, 또는 샤딩으로 분산해야 한다.

3. **정확성이 돈과 직결되는가?** counter의 비멱등성은 타협 불가능한 위험이다. 잔액·정산 금액은 멱등한 이벤트 append + 결정적 집계로 가야 한다. counter는 "대시보드용 근사 카운트"까지만.

4. **유일성·상태전이를 가끔, 분산된 키로 거는가?** 이건 LWT의 황금 사례다. 멱등키 선점, 주문 상태 전이처럼 키가 잘게 흩어지고 빈도가 폭주하지 않으면 LWT가 정확히 맞는 도구다.

> **결제 시스템에서의 실전 분담(가이드 톤):** 멱등키 선점·웹훅 중복 방지 같은 "한 키당 한 번" 류는 LWT가 잘 맞는다. 반면 원장 정합성·정확한 잔액·환불 가능액 검증처럼 다중 엔티티 ACID가 필요한 영역은 합의/트랜잭션 계층(RDB 또는 액터 기반 이벤트 소싱)으로 분리하는 편이 안전하다. 엔티티당 단일 액터로 순차 처리하는 설계 역시, 본질적으로 Cassandra LWT의 파티션 직렬화를 애플리케이션 레벨로 끌어올려 경합을 통제하는 것과 같은 발상이다.

---

## 핵심 요약

- **LWT = 단일 파티션 compare-and-set.** `IF NOT EXISTS`/`IF col=val`로 선형화(linearizable) 조건부 쓰기를 제공한다. RDB 트랜잭션의 일반 대체물이 아니라, 한 파티션에 대한 CAS 한 방일 뿐이다. `[applied]=False`는 에러가 아니라 정상 응답이며 현재값을 함께 돌려준다.
- **LWT 내부 = Paxos 4라운드(prepare/promise → read → propose/accept → commit).** 일반 쓰기의 약 4배 왕복(이건 Paxos v1 기준 — 4.1+/5.0의 `paxos_variant: v2`는 정상 경로 왕복을 대략 절반으로 줄인다). read를 합의 안에 품어 락 없이 read-modify-write를 원자화한다. Paxos 상태는 `system.paxos`에 저장된다. 5.0의 LWT는 여전히 Paxos이며, 범용 트랜잭션(Accord/CEP-15)은 5.0에 미포함.
- **serial consistency는 별도 축.** `SERIAL`(전 DC)/`LOCAL_SERIAL`(로컬 DC). 멀티 DC면 거의 항상 `LOCAL_SERIAL`. 선형 읽기를 원하면 읽기도 serial CL로 해야 한다(쓰기만 LWT로는 부족).
- **LWT 경합 = 성능 절벽.** 같은 파티션 동시 LWT는 ballot 경쟁·재시도로 처리량이 무너진다. 키가 분산되면(멱등키, user_id) 잘 확장된다. 핫 파티션 LWT는 금물.
- **BATCH는 성능 도구가 아니라 원자성 도구다.** Logged batch는 batchlog로 all-or-nothing(결국 적용)을 보장하나 롤백·격리는 안 준다(격리는 단일 파티션일 때만). **Multi-partition logged batch는 batchlog 오버헤드 + 코디네이터 부하로 안티패턴.** `batch_size_warn_threshold` 5KB / `batch_size_fail_threshold` 50KB.
- **빠른 BATCH = 같은 파티션 unlogged batch.** 다른 파티션을 묶으면 원자성도 없고 fan-out 부하만 진다. 차라리 개별 비동기 병렬 쓰기.
- **Counter = 비멱등 read-before-write.** 내부적으로는 복제본별 누적 상태(counter context, PN-counter류 CRDT)를 병합하는 구조라, API에선 델타만 표현 가능. 타임아웃 재시도 시 중복/누락 위험. 일반 컬럼과 혼합 불가, `INSERT`/TTL/절대값 세팅 불가. 2.1 재작성으로 신뢰성↑이나 비멱등성은 여전. **돈·잔액에 쓰지 말 것.** 근사 통계용으로만.
- **결정 기준:** 다중 행 ACID·롤백·정확한 금액·핫 파티션 고경합이면 RDB/Redis/외부 시스템. 분산된 키의 유일성·상태전이는 LWT의 황금 사례.

---

## 연결 노트

- [[04 - Tunable Consistency]] — LWT의 serial consistency는 일반 R/W 일관성과 별개의 축. 이 장의 `SERIAL`/`LOCAL_SERIAL`은 거기서 본 `QUORUM`/`LOCAL_QUORUM` 직관의 합의 버전이다.
- [[03 - 복제 전략과 데이터센터]] — `SERIAL` vs `LOCAL_SERIAL`이 DC 토폴로지에 따라 왜 그렇게 비싸지는지의 배경.
- [[08 - Storage Engine 내부]] — LWW(last-write-wins)·timestamp 셀 모델. LWT/Counter가 왜 이 모델의 "예외 장치"인지.
- [[10 - Write Path와 Read Path]] — LWT의 내부 read 라운드, counter의 read-before-write, 일반 쓰기의 멱등성이 모두 여기서 배운 경로 위에서 일어난다.
- [[05 - 데이터 모델링 1 - Query First]] — multi-table batch가 정당해지는 유일한 맥락(denormalized view 정합성)의 출발점.
- [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]] — counter 혼합 불가·핫 파티션 같은 안티패턴의 큰 그림.
- [[13 - 실전 Event Sourcing on Cassandra]] — counter 대신 이벤트 append + 집계, LWT 기반 상태 전이의 실전 적용.
- [[12 - 운영과 트러블슈팅]] — LWT 재시도 폭주·`WriteTimeoutException(CAS)`·batchlog 적체 같은 운영 증상 대응.
