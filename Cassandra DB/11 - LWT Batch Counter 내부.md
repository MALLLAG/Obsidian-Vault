---
title: LWT, Batch, Counter 내부 - Paxos 경량 트랜잭션
date: 2026-06-26
tags: [cassandra, lwt, paxos, batch, counter, 학습노트]
---

RDB를 쓰다가 Cassandra를 처음 깊이 다루게 된 엔지니어가 가장 먼저 묻는 질문은 거의 정해져 있다. "그래서 트랜잭션은 어떻게 하나요?", "여러 INSERT를 묶어서 한 번에 처리하려면 BATCH를 쓰면 되죠?", "조회수나 잔액 같은 카운터는 그냥 `count = count + 1` 하면 되나요?" 이 세 질문은 각각 LWT, BATCH, Counter로 이어지는데, 셋 모두 RDB의 직관을 그대로 적용하면 **반드시** 사고가 난다.

[[01 - Cassandra란 무엇인가]]에서 보았듯이 Cassandra는 "가용성(availability)과 분할 내성(partition tolerance)을 얻기 위해 강한 일관성(strong consistency)과 전역 트랜잭션을 포기한다"는 AP 선택에서 출발했다. [[04 - Tunable Consistency]]에서는 포기한 일관성을 수치로 조절해 되찾는 방법을 다뤘다.

이 장은 여기서 한 걸음 더 나아가, **읽고 판단하고 쓰는(read-modify-write) 원자적 연산이 정말로 필요할 때** Cassandra가 무엇을 해 주는지, 그 대가가 얼마인지, 어디서 멈춰야 하는지를 살펴본다.

## 왜 Cassandra에 "조건부 쓰기"가 필요해졌나

Cassandra의 기본 쓰기 방식은 **last-write-wins(LWW)** 이다. 같은 셀(cell)에 쓰기 두 개가 들어오면 타임스탬프가 더 큰 쪽이 이긴다. 충돌을 "감지"하지 않고 "덮어쓰는" 것이다. 이 성질은 [[08 - Storage Engine 내부]]에서 본 immutable cell + timestamp 모델에서 자연스럽게 나오며, 분산 환경에서 락(lock) 없이 수렴(convergence)을 보장하는 영리한 방법이다.

문제는 이 모델로는 다음 요구를 처리할 수 없다는 점이다.

> "이 `user_id`가 **아직 없을 때만** INSERT하라. 누군가 0.1초 전에 같은 아이디로 가입했다면 실패시켜라."

LWW로는 불가능하다. 두 사용자가 동시에 `shawn`이라는 아이디로 가입을 시도하면 두 INSERT가 모두 "성공"하고, 타임스탬프가 큰 쪽이 다른 쪽을 조용히 덮어쓴다. 두 사용자 모두 "가입 완료" 화면을 보지만 데이터는 하나만 남는다. **유일성(uniqueness) 위반을 감지조차 하지 못한다.**

RDB라면 `UNIQUE` 제약과 트랜잭션으로 이 문제를 한 줄에 해결한다. 하지만 그 한 줄 뒤에는 단일 노드(또는 합의로 정한 리더)가 전역 순서를 강제한다는 전제가 깔려 있다. Cassandra에는 그런 중앙 권위가 없고, 모든 복제본(replica)이 대등하다.

그래서 Cassandra는 **합의 알고리즘(consensus algorithm)** 을 도입한다. 특정 파티션 키에 대해 복제본들이 "지금 이 값을 바꿔도 되는가"를 표결로 정하는 것이다. 이것이 LWT(Lightweight Transaction)이고, 그 엔진은 **Paxos** 이다.

여기서 "Lightweight"라는 이름에 속으면 안 된다. RDB의 무거운 ACID 트랜잭션에 비해 "가볍다"는 상대적인 표현일 뿐이고, 일반 Cassandra 쓰기와 비교하면 **압도적으로 무겁다.**

```text
RDB의 직관:                          Cassandra의 현실:
  BEGIN;                               INSERT ... IF NOT EXISTS;
  SELECT ... FOR UPDATE;       →       (단일 행/단일 파티션에 대한
  UPDATE ...;                           조건부 원자 연산 1개,
  COMMIT;                               Paxos 합의로 구현)
  ↑ 여러 행·여러 테이블 가능            ↑ 한 파티션 안에서만, 한 연산만
```

LWT는 RDB 트랜잭션을 일반적으로 대체하는 기능이 아니다. **단일 파티션에 대한 compare-and-set(CAS)** 한 번일 뿐이다. 처음부터 이 한계를 분명히 알아 두어야 한다.

---

## LWT의 표면: 무엇을 쓸 수 있나

내부로 들어가기 전에 CQL에서 LWT를 어떻게 쓰는지 먼저 정리하자. LWT는 `IF` 절로 표현한다.

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

중요한 것은 LWT가 성공과 실패를 알려 주는 방식이다. 일반 쓰기는 응답 본문이 비어 있지만, LWT는 항상 `[applied]` 컬럼이 들어 있는 행 하나를 돌려준다.

```cql
cqlsh> UPDATE orders SET status='SHIPPED'
   ...> WHERE order_id='ord-1234' IF status='PAID';

 [applied] | status
-----------+--------
     False | CANCELLED  -- 조건 불일치: 적용 안 됨, 현재 실제 값(PAID 아님)을 함께 반환
```

`[applied]`가 `False`이면 Cassandra는 **현재 실제 값을 함께 돌려준다.** 이 점이 핵심이다. 애플리케이션은 이 반환값을 보고 "내가 알던 상태가 틀렸으니 재시도하거나 다른 분기로 가자"고 판단한다. 이것이 CAS(compare-and-set)의 본질이다. 비교가 실패하면 현재 값을 알려 주어 호출자가 재시도 루프를 돌 수 있게 한다.

> [!warning] 흔한 함정
> LWT의 결과를 일반 쓰기처럼 "에러가 났는가"로만 판단하면 `[applied]=False`를 놓친다. 드라이버마다 제공하는 `wasApplied()` 같은 메서드를 반드시 확인해야 한다. `[applied]=False`는 **에러가 아니라 정상 응답**이다. 예외가 나지 않았다고 해서 쓰기가 성공한 것은 아니다.

LWT 조건에는 정적 컬럼(static column), 일반 컬럼, primary key 존재 여부(`IF EXISTS`/`IF NOT EXISTS`) 등을 쓸 수 있다. 단, 조건은 모두 **같은 파티션** 안에 있어야 한다. LWT는 절대 파티션 경계를 넘지 못한다. 이것은 우연이 아니라 Paxos가 파티션 키 단위로 동작하기 때문이며, 다음 절에서 이 내용을 다룬다.

---

## Paxos: LWT가 내부에서 실제로 하는 일

Cassandra의 LWT는 **단일 결정 Paxos(single-decree Paxos)** 의 변형을 파티션 키 단위로 실행한다. 복제본들은 "이 파티션의 다음 변경은 무엇인가"라는 하나의 명제를 두고 합의한다. Multi-Paxos처럼 안정적인 리더(stable leader)를 두지 않기 때문에 LWT를 실행할 때마다 합의를 처음부터 다시 진행한다. 이것이 비용이 커지는 근본 원인이다.

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

각 단계를 차례로 살펴보자.

1. **Prepare / Promise.** 코디네이터는 단조 증가하는 **ballot(제안 번호, 보통 시간 기반 UUID)** 을 만들어 복제본들에게 "이 ballot으로 제안하겠다"고 알린다. 복제본은 이 ballot이 자신이 본 적 있는 것보다 크면 "약속(promise)"한다. 이때 자신이 알고 있는 **아직 commit되지 않은 in-progress Paxos 제안**이 있으면 그 제안도 응답에 실어 보낸다. 과반(quorum)의 promise를 받지 못하면 실패하고 재시도한다.

2. **Read (현재값 조회).** `IF` 조건을 평가하려면 그 파티션의 **현재 실제 값**을 알아야 한다. 코디네이터는 복제본들에서 최신 데이터를 읽고, 이 과정에서 read repair에 준하는 동기화가 일어난다. 즉 LWT는 내부에 **읽기 한 라운드를 포함하고 있다.** LWT를 "읽고 판단하고 쓰는" 연산이라고 부르는 이유가 여기에 있다.

3. **Propose / Accept.** 조건이 참이면 코디네이터는 실제로 적용할 변경(mutation)을 같은 ballot으로 제안한다. 과반의 복제본이 이 제안을 자신의 Paxos 상태에 기록(accept)하면 합의가 성립한다. 조건이 거짓이면 여기서 멈추고 `[applied]=False`와 현재 값을 반환한다.

4. **Commit.** 복제본들은 합의된 변경을 실제 테이블(memtable/commitlog, [[10 - Write Path와 Read Path]] 참고)에 반영하고 Paxos 임시 상태를 정리한다. 이후에는 이 변경을 일반 데이터처럼 읽을 수 있다.

일반 QUORUM 쓰기는 사실상 **왕복 1번**이지만 LWT는 **약 4번**이다. 그래서 흔히 "LWT는 일반 쓰기의 약 4배 비용"이라고 말한다. 다만 이것은 네트워크 왕복 수만 비교한 것이다. 실제로는 read 단계의 디스크 접근, Paxos 상태를 위한 추가 저장, contention이 생겼을 때의 재시도까지 겹치므로 체감 비용은 4배보다 클 수 있다.

> [!caution] 버전 주의 (Cassandra 5.0)
> 위의 "왕복 4번"은 전통적인 Paxos(이하 **Paxos v1**) 기준이다. Cassandra 4.1부터 최적화된 **Paxos v2**(`cassandra.yaml`의 `paxos_variant: v2`)가 도입되었고, 5.0에서도 쓸 수 있다. v2는 prepare/promise 단계에서 조건 평가용 read를 함께 처리하고 불필요한 commit 왕복을 줄여서, 경합이 없는 정상 경로의 왕복 수를 대략 절반(2~3 수준)까지 줄인다.
>
> 다만 이는 클러스터 설정(그리고 v1↔v2 전환 절차)에 따라 달라지므로, "LWT는 일반 쓰기보다 몇 배 비싸다"는 말은 정확한 상수가 아니라 **자릿수 감각**으로만 받아들여야 한다. 그리고 v1이든 v2든 5.0의 LWT는 여전히 **Paxos** 기반이다. 범용 멀티키 트랜잭션을 목표로 하는 Accord(CEP-15)는 5.0 GA에 포함되지 않았고 이후 버전에서 추진하는 방향이다. 따라서 이 장에서 다루는 한계(단일 파티션, CAS 한 번)는 5.0 기준으로도 그대로 유효하다.

> [!note] 왜 read가 중간에 끼는가?
> RDB의 `SELECT ... FOR UPDATE`는 락을 잡은 상태에서 읽는다. Cassandra에는 락이 없으므로 "읽은 값이 합의 시점에도 유효하다"는 사실을 Paxos ballot 순서로 보장한다. ballot이 read와 propose를 하나의 논리적 원자 단위로 묶는 것이다. 그래서 read는 반드시 합의 프로토콜 **안**에서 해야 한다. 합의 밖에서 따로 `SELECT`하면 그 사이에 값이 바뀌어 race가 생긴다.

### Paxos 상태는 어디에 저장되나

Cassandra는 Paxos의 in-progress 상태(promised ballot, accepted proposal 등)를 `system.paxos` 테이블에 저장한다. commit이 끝나면 이 상태를 정리하지만, 추가로 저장하고 정리하는 작업 자체가 LWT의 숨은 오버헤드다. Paxos 상태에는 TTL이 걸려 있어 일정 시간 뒤 GC되며, 이와 별도로 `cas_contention_timeout`(기본 1000ms 수준) 같은 타임아웃도 있다.

### SERIAL vs LOCAL_SERIAL: 합의의 범위

LWT에는 일반 쓰기의 consistency level과 **별개의 축**인 **serial consistency** 가 있다. 선택지는 두 가지뿐이다.

| serial CL | 합의에 참여하는 복제본 범위 | linearizable 보장 범위 | 비용/지연 |
|---|---|---|---|
| `SERIAL` | **모든 DC**의 복제본 과반 | 전역(글로벌) 선형화 | DC 간 왕복 포함 → 매우 느림 |
| `LOCAL_SERIAL` | **로컬 DC**의 복제본 과반 | 로컬 DC 내 선형화 | DC 내부만 → 상대적으로 빠름 |

[[03 - 복제 전략과 데이터센터]]에서 본 멀티 DC 토폴로지를 떠올리면 이해하기 쉽다. `SERIAL`은 지구 반대편 DC의 복제본까지 표결에 참여시키므로, LWT 한 번에 대륙 간 왕복(수십~수백 ms)이 4번 들어간다. 전 세계에서 유일한 이메일처럼 글로벌 유일성이 정말 필요한 경우가 아니라면 거의 항상 `LOCAL_SERIAL`을 쓴다.

> [!warning] 혼용 금지 함정
> 같은 파티션(같은 데이터)에 대해 어떤 연산은 `SERIAL`로, 어떤 연산은 `LOCAL_SERIAL`로 섞어 보내면 **선형화 보장이 깨진다.** 두 레벨은 합의에 참여하는 quorum 집합이 다르므로, 서로 겹치지 않는 quorum이 각자 다른 값을 확정할 수 있다. 하나의 데이터셋에는 읽기와 쓰기 모두 **하나의 serial 레벨을 일관되게** 써야 한다.
>
> 또한 `LOCAL_SERIAL`은 로컬 DC 내부의 선형화만 보장하므로, DC 장애로 다른 DC에 페일오버하는 순간 그 보장이 이어지지 않는다는 점도 설계할 때 고려해야 한다.

짚어 둘 점이 있다. LWT 문장 하나에는 **두 개의 consistency level**이 관여한다.

- **serial consistency** (`SERIAL`/`LOCAL_SERIAL`): Paxos 합의(prepare/read/propose)의 범위를 정한다.
- **일반 consistency** (`QUORUM`/`LOCAL_QUORUM` 등): commit 단계에서 실제 데이터를 몇 개의 복제본에 확정 반영할지를 정한다.

```cql
-- 드라이버 레벨 설정 예 (개념)
serial consistency = LOCAL_SERIAL
consistency        = LOCAL_QUORUM
```

흔한 실수는 쓰기는 LWT로 하고 정작 읽기는 일반 `ONE`으로 하는 것이다. 이렇게 하면 선형화가 깨진다. **LWT로 보장한 값을 선형적으로 읽으려면 읽기도 `SERIAL`/`LOCAL_SERIAL`로 해야 한다.** 그래야 진행 중인 Paxos를 마무리(read-time repair)하고 가장 최근에 합의된 값을 볼 수 있다. LWT 쓰기와 일반 QUORUM 읽기를 섞으면 "방금 LWT로 정한 값이 잠깐 보이지 않는" 비선형 구간이 생길 수 있다.

> [!warning] 함정
> "linearizable"은 공짜가 아니다. 쓰기를 LWT로 했더라도 읽기를 일반 CL로 하면 그 읽기는 선형화되지 않는다. 일관성은 **읽기와 쓰기 양쪽의 CL 조합**으로 결정된다(4장 Tunable Consistency에서 본 `R + W > N` 직관을 serial 축으로 확장한 것이다).

---

## 경합(contention): LWT의 성능 절벽

LWT의 가장 위험한 성질은 **같은 파티션에 몰린 동시 LWT가 서로를 방해한다**는 점이다. Paxos는 본질적으로 "한 번에 하나의 제안만 통과시키는" 프로토콜이다. 같은 파티션 키에 LWT N개가 동시에 몰리면 ballot 경쟁이 벌어진다.

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

이 현상을 "성능 절벽(performance cliff)"이라고 부른다. 일반 쓰기는 동시성이 올라가면 처리량도 (자원 한계까지) 함께 올라간다.

그러나 **단일 파티션에 대한 LWT는 동시성이 어느 선을 넘으면 재시도, `WriteTimeoutException(CAS)`, `cas_contention_timeout` 초과가 폭증하면서 처리량이 무너진다.** 핫 파티션(hot partition) 하나에 LWT를 몰면, 그 파티션이 사실상 전역 직렬화 지점(global serialization point)이 된다.

여기서 얻을 교훈은 세 가지다.

- **LWT는 경합이 낮은 곳에서만 제 역할을 한다.** LWT가 서로 다른 파티션 키로 분산되면(예: `user_id`별 가입) 각 Paxos가 독립적으로 동작하므로 잘 확장된다.
- **인기 있는 파티션 하나에 여러 클라이언트가 LWT를 보내는 설계는 안티패턴이다.** 예를 들어 "전역 카운터를 LWT로 증가"시키면 모든 요청이 같은 파티션에 몰려 절벽에 부딪힌다.
- 드라이버의 LWT 재시도는 **멱등하지 않을 수 있다.** 재시도할 때마다 `[applied]`의 의미가 달라질 수 있으므로, 생각 없이 재시도하는 것은 위험하다(뒤의 Counter 절과 같은 종류의 함정이다).

RDB와 비교하면 차이가 분명해진다.

| | RDB (`SELECT FOR UPDATE`) | Cassandra LWT |
|---|---|---|
| 동시 충돌 처리 | 락 대기 (blocking) | ballot 경쟁 + 재시도 (optimistic-ish) |
| 핫스팟 | 락 큐가 길어지지만 진행은 됨 | 재시도 폭증, 처리량 붕괴 가능 |
| 범위 | 여러 행/테이블 | 단일 파티션, 단일 CAS |
| 비용 | 트랜잭션 1회 | 일반 쓰기의 ~4배 + 재시도 |

> [!note] 결제 시스템 직관
> 결제 멱등키(idempotency key)마다 한 번씩 "선점"하는 작업은 LWT를 쓰기에 이상적인 사례다(키가 충분히 분산되기 때문이다). 반대로 "오늘 전체 결제 건수 카운터"를 LWT로 올리면 모든 결제 트래픽이 한 파티션에서 직렬화되므로 스스로 병목을 만드는 셈이다.

---

## LWT를 정말 써야 할 때: 실전 패턴 3가지

### 1) 유일성 보장: 회원 가입, 멱등키 선점

LWT를 쓰는 가장 정석적인 경우다. `IF NOT EXISTS`로 "처음 쓰는 요청만 성공"하게 만든다.

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

`email`을 파티션 키로 두었기 때문에 이메일마다 LWT가 서로 독립적으로 실행된다. 특정 이메일로 동시 가입이 폭주하지 않는 한 핫스팟도 생기지 않는다. LWT가 잘 확장되는 전형적인 경우다.

결제 시스템의 **멱등키 선점**도 구조가 같다. `idempotency_key`를 파티션 키로 두고 `IF NOT EXISTS`로 "이 결제 요청을 처음 처리하는 주체"를 하나만 뽑는다. 중복 요청은 `[applied]=False`로 걸러지고, 이미 처리된 결과를 돌려받는다.

### 2) 상태 머신 전이: 주문/결제 상태

`IF status = '이전상태'`로 "유효한 전이만 허용"한다. [[13 - 실전 Event Sourcing on Cassandra]]의 상태 모델과 바로 연결되는 내용이다.

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

핵심은 "현재 상태를 읽고, 애플리케이션에서 판단하고, 쓰는" 과정 사이에 생기는 race를 LWT가 막아 준다는 점이다. 일반 쓰기였다면 두 워커가 동시에 `PAID`를 읽고 둘 다 `SHIPPED`로 써서 이중 배송이 일어났을 것이다. LWT는 둘 중 하나만 성공시킨다.

또한 LWT가 `order_id`별로 분산되므로 확장성도 좋다. 주문 하나에 워커 수십 개가 동시에 상태를 바꾸려고 다투는 상황만 아니면 된다.

### 3) 재고 차감: 가장 조심해야 할 사례

```cql
-- 현재 재고가 정확히 10일 때만 9로 (정수 CAS)
UPDATE inventory SET stock = 9
WHERE sku='SKU-1'
IF stock = 10;
```

재고는 단일 `sku`(파티션)에 있는 단일 카운터다. 따라서 **인기 상품의 동시 구매는 곧 같은 파티션에 대한 LWT 폭주**이고, 이는 성능 절벽으로 이어진다. 플래시 세일(flash sale)에서 상품 하나에 LWT를 거는 것은 위험하다. 그래서 실무에서는 다음 중 하나를 택한다.

- 재고를 여러 샤드(shard)로 나눠 LWT를 분산한다(예: `sku` + `shard_id` 100개).
- 처음부터 재고를 **외부 인메모리 저장소(Redis 등)나 RDB**로 옮겨 Cassandra LWT를 피한다.
- 정합성 대신 "약간의 오버셀(oversell)을 허용하고 사후에 보정한다"는 비즈니스 결정을 내린다.

> 재고 차감을 LWT로 구현하기 전에 반드시 스스로 물어보자. "이 상품 하나에 초당 몇 건의 동시 차감이 들어오는가?" 수백 건이라면 LWT는 무너진다. 단일 핫 파티션에 대한 LWT는 단일 스레드로 직렬 처리할 때의 처리량을 넘을 수 없다.

---

## BATCH의 진짜 의미 (가장 흔한 오해)

이제 이 장에서 오해가 가장 심한 주제로 넘어간다. RDB를 쓰던 엔지니어는 거의 예외 없이 BATCH를 이렇게 이해한다.

> (틀린 직관) "여러 INSERT/UPDATE를 BATCH로 묶으면 네트워크 왕복이 줄어서 **빨라진다.** JDBC batch처럼."

**Cassandra의 BATCH는 성능 최적화 도구가 아니다.** 이름이 같아서 생긴 오해 가운데 가장 비싼 대가를 치르는 것 중 하나다. Cassandra BATCH의 목적은 처리량이 아니라 **원자성(atomicity)** 이다. 오히려 대부분의 경우 BATCH는 처리량을 **떨어뜨린다.**

### Logged batch: batchlog로 얻는 원자성

기본 BATCH(= logged batch)는 묶인 변경이 **모두 적용되거나 모두 적용되지 않는다**(all-or-nothing)는 것을 보장한다. 이를 구현하는 장치가 **batchlog** 이다.

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

코디네이터는 먼저 batch 전체를 **batchlog**(다른 노드 2곳에 복제)에 기록한다. 그다음 개별 mutation을 각각의 대상 복제본으로 보낸다. 코디네이터가 중간에 죽더라도 batchlog를 가진 노드가 완료되지 않은 batch를 발견해 **남은 mutation을 재생(replay)** 한다. 그래서 결국에는 "모두 적용"이 보장된다.

여기서 보장되는 것과 **보장되지 않는 것**을 정확히 구분하자.

| | Logged batch가 보장하는가? |
|---|---|
| 원자성(atomicity, all-or-nothing 결국 적용) | ✅ batchlog 재생으로 보장 |
| 격리(isolation, 중간 상태가 안 보임) | ❌ **단일 파티션 batch일 때만** |
| 순서·고립된 트랜잭션 | ❌ 다른 클라이언트는 부분 적용 중간 상태를 볼 수 있음 |
| 롤백(rollback) | ❌ Cassandra에 롤백 없음. "전부 적용"으로 수렴할 뿐 |

> [!warning] 결정적 오해 정정
> Logged batch는 "실패하면 되돌린다"는 뜻이 아니라 "결국 모두 적용된다"는 뜻이다. 롤백은 없다. 그리고 multi-partition batch라면 다른 읽기 클라이언트가 batch가 절반만 적용된 **중간 상태를 볼 수 있다**. 즉 ACID의 A는 (결과적으로) 제공하지만 I는 일반적으로 제공하지 않는다.

### Multi-partition logged batch는 안티패턴이다

여기서 BATCH에 대한 두 번째 큰 오해가 깨진다. "여러 파티션에 흩어진 쓰기를 BATCH로 묶으면 효율적"이라는 생각은 사실과 **정반대**다.

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

Multi-partition logged batch에는 다음 문제가 있다.

1. **batchlog 오버헤드.** 모든 mutation을 직렬화해 다른 노드 2곳에 먼저 기록하고, 끝나면 다시 지우는 추가 쓰기 I/O가 생긴다. 같은 쓰기를 개별로 보낼 때보다 **느리다.**
2. **코디네이터 부하 집중.** 코디네이터가 여러 파티션의 복제본으로 요청을 흩뿌리는 fan-out 허브가 되므로, 이 노드의 CPU와 네트워크가 병목이 된다.
3. **격리 없음.** 여러 파티션에 걸쳐 있으므로 다른 클라이언트가 부분 적용 상태를 볼 수 있다. 원자성도 즉시가 아니라 "결국" 보장될 뿐이다.
4. **로그 경고.** Cassandra는 batch 크기가 `batch_size_warn_threshold`(기본 5KB)를 넘으면 WARN을 남기고, `batch_size_fail_threshold`(기본 50KB)를 넘으면 batch를 **거부**한다. 큰 multi-partition batch를 보내면 이 한계에 부딪힌다.

multi-partition logged batch를 써도 되는 경우는 하나뿐이다. **여러 테이블(여러 denormalized view)에 같은 사실을 원자적으로 반영해야 할 때**다. [[05 - 데이터 모델링 1 - Query First]]에서 본 "같은 데이터를 쿼리별로 여러 테이블에 중복 저장하는" 패턴에서, 두 view가 함께 갱신된다는 것을 보장하려고 batch를 쓴다. 이때도 batch는 성능이 아니라 **정합성을 위해** 쓰는 것이며, mutation 수는 작게 유지해야 한다.

```cql
-- 정당한 multi-table batch: 같은 사실을 두 view에 원자적 반영
BEGIN BATCH
  INSERT INTO orders_by_id   (order_id, user_id, amount) VALUES ('o1','u1',1000);
  INSERT INTO orders_by_user (user_id, order_id, amount) VALUES ('u1','o1',1000);
APPLY BATCH;
-- 둘 다 적용되거나(결국) 둘 다 안 됨. denormalized view 간 정합성 목적.
```

### Unlogged batch: 같은 파티션을 묶을 때만

`UNLOGGED` 키워드를 붙이면 batchlog를 건너뛴다. 오버헤드는 없어지지만, 그 대가로 **원자성 보장을 포기한다.**

```cql
BEGIN UNLOGGED BATCH
  INSERT INTO events (pk, ck, v) VALUES ('p1', 1, 'a');
  INSERT INTO events (pk, ck, v) VALUES ('p1', 2, 'b');
  INSERT INTO events (pk, ck, v) VALUES ('p1', 3, 'c');
APPLY BATCH;
-- 모두 같은 파티션 키 'p1' → 코디네이터가 한 노드로 한 번에 보냄 → 실제로 효율적
```

unlogged batch가 의미 있는 **유일한** 경우는 **모든 mutation의 파티션 키가 같을 때**다. 이때 코디네이터는 그 파티션의 복제본 한 세트로 메시지 하나를 보내므로 네트워크 왕복이 실제로 줄어든다. 또한 단일 파티션이므로 **격리(isolation)와 원자성**도 사실상 함께 얻는다(한 노드에서 한 번에 memtable에 반영되기 때문이다).

반대로 **서로 다른 파티션 키를 unlogged batch로 묶는 것은 명백한 안티패턴**이다. batchlog가 없으니 원자성도 없고, 코디네이터의 fan-out 부하만 떠안는다. 차라리 개별 비동기 쓰기를 병렬로 보내는 편이 빠르다.

Cassandra도 이를 경계해서, unlogged batch 하나가 일정 개수(`unlogged_batch_across_partitions_warn_threshold`, 기본 10개)를 넘는 파티션에 걸치면 로그에 WARN을 남긴다. 이 경고가 보이면 그 batch는 십중팔구 잘못 쓰이고 있는 것이다.

| batch 종류 | 원자성 | 격리 | 언제 쓰나 | 흔한 오용 |
|---|---|---|---|---|
| **Logged, single-partition** | ✅ | ✅ | 한 파티션에 여러 행 원자 반영 | (드물게 적절) |
| **Logged, multi-partition** | ✅(결국) | ❌ | 여러 view 정합성(소량) | 성능 목적의 대량 묶기 ← 안티패턴 |
| **Unlogged, single-partition** | ✅(사실상) | ✅ | 같은 파티션 묶음 쓰기 | (적절) |
| **Unlogged, multi-partition** | ❌ | ❌ | (거의 없음) | "왕복 줄이려고" 묶기 ← 안티패턴 |

> [!note] 한 줄 요약
> BATCH로 성능을 얻으려면 **반드시 파티션 키가 같아야** 한다. 파티션이 흩어지는 순간 BATCH는 정합성 도구일 뿐이고, 그마저도 비싸다. "여러 파티션을 묶어서 빠르게" 처리하는 방법은 BATCH에 없다. 그 역할은 개별 병렬 쓰기(`executeAsync` fan-out)가 맡는다.

### BATCH와 LWT의 조합

한 가지를 더 짚어 두자. logged batch 안에 LWT(`IF`)를 넣을 수 있지만, **모든 조건이 같은 파티션**에 있어야 한다. 즉 "단일 파티션 conditional batch"만 가능하다. 한 파티션 안에서 여러 행을 원자적이고 조건부로 바꾸는 강력한 도구이지만, 이것도 단일 파티션 Paxos이므로 경합 비용은 그대로다.

---

## Counter: 분산 환경에서 "더하기"가 어려운 이유

마지막 주제는 Counter다. "조회수 +1"은 단순해 보이지만, 분산 환경에서 **덧셈은 의외로 가장 어려운 연산** 중 하나다.

### 왜 일반 컬럼으로는 안 되나

일반 Cassandra 쓰기는 LWW(덮어쓰기)다. "현재값 + 1"을 하려면 현재값을 읽어야 하는데, 읽고 쓰는 사이에 다른 쓰기가 끼어들면 증가분이 사라진다(lost update). 그래서 Cassandra는 별도의 `counter` 타입을 둔다. counter는 클라이언트가 **절댓값을 지정할 수 없고 증감(delta)만 표현할 수 있는** 특수 컬럼이다.

내부 표현을 정확히 짚어 보자. counter는 단순히 "델타를 로그처럼 쌓아 두는" 방식이 아니다. 각 복제본은 자신이 맡은 누적값을 담은 **counter context**, 즉 대략 `(host_id, logical_clock, count)` 튜플의 묶음을 가지고 있고, 읽을 때 복제본별 누적분을 병합해 최종 합계를 만든다. 구조적으로는 **상태 기반(state-based) CRDT(PN-counter)** 에 가깝다.

클라이언트 API에서 델타만 보인다는 것과 저장소가 복제본별 누적 상태를 가지고 있다는 것은 별개다. 이 둘을 구분해야 뒤에 나오는 read-before-write와 비멱등성을 자연스럽게 이해할 수 있다.

```cql
CREATE TABLE page_views (
    page_id text PRIMARY KEY,
    views   counter        -- 일반 컬럼과 함께 둘 수 없음 (PK 제외)
);

UPDATE page_views SET views = views + 1 WHERE page_id = 'home';
UPDATE page_views SET views = views - 1 WHERE page_id = 'home';
```

제약부터 분명히 해 두자.

- **counter 컬럼은 일반 컬럼과 같은 테이블에 섞을 수 없다.** primary key 컬럼을 제외한 한 테이블의 non-PK 컬럼은 모두 counter이거나 모두 counter가 아니어야 한다. counter의 충돌 병합 규칙이 일반 셀과 근본적으로 다르기 때문이다.
- counter에는 `INSERT`가 없다. `UPDATE`의 `+`/`-` 만 쓸 수 있고, TTL도 걸 수 없다.
- counter 값을 특정 숫자로 직접 "설정"할 수 없다(`= 100` 불가). 상대적인 증감만 할 수 있다.

### 내부: read-before-write 경로

counter `+1`은 내부적으로 일반 쓰기보다 무겁다. 코디네이터가 받은 increment를 처리하려면 **선택된 복제본(leader replica)이 현재 누적값을 읽고, delta를 더한 뒤, 새 누적값을 기록**해야 한다. 즉 counter 쓰기는 내부적으로 **read-before-write** 이다.

```text
일반 쓰기:                  Counter 쓰기:
  값을 그냥 적는다            1) 코디네이터 → 리더 복제본 선택
  (blind write)              2) 리더가 현재 counter 로컬값 read
                             3) delta(+1) 적용한 새 값 계산
                             4) 새 값을 자신 + 다른 복제본에 기록(복제)
                             ↑ read 단계 때문에 일반 쓰기보다 느리고
                                idempotent하지 않음
```

이 read 단계 때문에 두 가지 결과가 생긴다. 첫째, counter 쓰기는 일반 쓰기보다 느리다(특히 디스크에서 현재값을 읽어야 할 때 그렇다). 둘째, 이것이 더 중요한데, counter 쓰기는 **비멱등(non-idempotent)** 이다.

### 비멱등성: 타임아웃 재시도의 함정

이것이 counter의 가장 위험한 본질이다. `views = views + 1`을 보냈는데 **타임아웃**이 났다고 하자. 클라이언트는 이 요청이 적용되었는지 **알 수 없다.**

```text
시나리오:
  Client ── "+1" ──► Coordinator ── 적용 성공 ──► (그러나 ack가 네트워크에서 유실)
  Client: 타임아웃! 재시도 ── "+1" ──► 또 적용됨
  결과: 실제로는 +2 가 되어버림 (중복 증가)

  vs. 반대로 처음에 진짜 실패였다면:
  재시도 안 하면 → +0 (증가 누락)
```

- 재시도하면 **중복 카운트(over-count)** 위험이 있다.
- 재시도하지 않으면 **누락(under-count)** 위험이 있다.

일반 `INSERT`/`UPDATE`는 멱등하다. 같은 타임스탬프와 같은 값으로 다시 써도 결과가 같다(10장 Write Path와 Read Path에서 본 timestamp 기반 수렴). 그래서 타임아웃이 나도 안전하게 재시도할 수 있다.

그러나 **counter의 `+1`은 같은 연산을 두 번 적용하면 결과가 달라진다.** 그래서 드라이버는 기본적으로 counter 쓰기를 **자동으로 재시도하지 않는다**(또는 매우 보수적으로 다룬다). 타임아웃이 나면 "적용됐는지 모르는" 상태를 애플리케이션이 감수해야 한다.

> [!note] 결제 시스템 직관
> 이런 이유로 **돈(잔액)을 counter로 관리하면 안 된다.** 타임아웃 한 번에 잔액이 한 단위 틀어질 수 있고, counter 자체에는 그 오차를 사후에 검증하고 교정할 방법이 없다(절댓값을 읽어 와서 "맞춰" 쓸 수도 없다). counter는 조회수, 좋아요, 대략적인 카운트처럼 **근사치여도 되는 통계**에만 적합하다. 정확성이 돈과 직결된다면 counter를 쓰면 안 된다.

### 4.0+ 개선점과 한계

Cassandra의 counter는 역사적으로 악명이 높았다. 2.1 이전의 구버전 counter는 복제본 간 read-during-write 충돌, replay 시 중복 누적 등으로 값이 드리프트(drift)하는 버그가 잦았다. 2.1에서 **counter 구현을 전면 재작성**하면서 안정성이 크게 올라갔고(리더 복제본을 기준으로 한 명확한 조정), 이후 4.0/5.0 계열에서도 안정성과 성능 개선이 이어졌다.

다만 **근본적인 비멱등성은 사라지지 않았다.** read-before-write라는 본질 때문에 4.0 이후의 counter도 여전히 다음 제약이 있다.

- 타임아웃이 나면 적용 여부가 불확실하므로, 생각 없이 재시도하면 안 된다.
- 일반 컬럼과 섞을 수 없다.
- 정확한 절댓값을 보장하지 않는다. "정상 경로에서는 정확하고, 장애 경로에서는 근사치"인 수준이다.

> [!warning] 버전 주의
> 버전별 내부 변경의 구체적인 내용(예: 5.0의 특정 최적화)은 각 버전의 릴리스 노트로 확인해야 한다. 여기서 단정할 수 있는 것은 "2.1 재작성으로 신뢰성은 크게 올라갔지만, 비멱등성·혼합 불가·재시도 위험이라는 본질적 제약은 그대로"라는 점이다.

대안 패턴은 다음과 같다.

- 정확한 합계가 필요하면 **개별 이벤트를 일반 테이블에 append**(멱등)하고, 합계는 배치나 스트리밍 집계로 계산한다. 13장(실전 Event Sourcing on Cassandra)에서 다루는 이벤트 누적 + 집계가 정석이다.
- 근사치로 충분하면 counter를 써도 된다. 단, 핫 파티션에 주의해야 한다(인기 페이지의 counter는 한 파티션에 쓰기가 몰려 write 핫스팟이 된다).

---

## 결정 가이드: 언제 Cassandra 대신 RDB나 외부 시스템을 써야 하나

세 기능에서 얻을 수 있는 교훈은 같다. **Cassandra는 일부러 포기한 강한 일관성을 LWT/BATCH/Counter로 일부 되찾을 수 있지만, 그 대가는 비싸고 한계도 뚜렷하다.** 그러므로 "되찾을 수 있다"와 "되찾아야 한다"는 다른 이야기다. 결제 시스템 엔지니어가 판단할 때 쓸 기준을 정리하면 다음과 같다.

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

구체적인 판단 기준은 다음과 같다.

1. **여러 행이나 여러 테이블에 걸친 ACID와 롤백이 필요한가?** 그렇다면 Cassandra를 쓸 곳이 아니다. LWT는 단일 파티션 CAS 한 번일 뿐이고, BATCH는 격리와 롤백을 제공하지 않는다. 진짜 트랜잭션이 필요하면 RDB(또는 Cassandra 위의 합의 계층)를 써야 한다. 결제에서 "원장(ledger)의 차변과 대변을 동시에 기입"하는 작업은 RDB가 맡아야 한다.

2. **경합이 한 파티션에 집중되는가?** LWT의 성능 절벽과 counter 핫스팟은 원인이 같다(단일 파티션 직렬화). 트래픽이 키 하나에 몰리면 Cassandra의 처리량은 그 키의 상한을 넘지 못한다. Redis(원자적 INCR/Lua), RDB row lock, 또는 샤딩으로 분산해야 한다.

3. **정확성이 돈과 직결되는가?** counter의 비멱등성은 타협할 수 없는 위험이다. 잔액과 정산 금액은 멱등한 이벤트 append와 결정적 집계로 처리해야 한다. counter는 "대시보드용 근사 카운트"까지만 쓴다.

4. **유일성 검사나 상태 전이를 가끔, 분산된 키에 거는가?** 이것이 LWT를 쓰기에 가장 좋은 경우다. 멱등키 선점이나 주문 상태 전이처럼 키가 잘게 흩어져 있고 빈도가 폭주하지 않는다면 LWT가 정확히 맞는 도구다.

> [!note] 결제 시스템에서의 실전 분담
> 멱등키 선점이나 웹훅 중복 방지처럼 "키 하나당 한 번"인 작업에는 LWT가 잘 맞는다. 반면 원장 정합성, 정확한 잔액, 환불 가능액 검증처럼 여러 엔티티에 걸친 ACID가 필요한 영역은 합의/트랜잭션 계층(RDB 또는 액터 기반 이벤트 소싱)으로 분리하는 편이 안전하다. 엔티티마다 액터 하나가 요청을 순차 처리하는 설계도, 본질적으로는 Cassandra LWT의 파티션 직렬화를 애플리케이션 레벨로 끌어올려 경합을 통제한다는 같은 발상이다.
