---
title: 실전 - Event Sourcing on Cassandra (Akka Persistence)
date: 2026-06-26
tags: [cassandra, event-sourcing, akka-persistence, 학습노트]
---

지금까지는 Cassandra에서 데이터를 어떻게 모델링하고, 어떻게 쓰고 읽으며, 내부에서 어떤 일이 일어나는지를 살펴보았다.

[[08 - Storage Engine 내부]]에서 본 LSM-tree, [[09 - Compaction 전략]]에서 본 compaction과 tombstone, [[10 - Write Path와 Read Path]]에서 본 쓰기·읽기 경로, [[05 - 데이터 모델링 1 - Query First]]에서 본 쿼리 우선 설계가 한꺼번에 맞물려 동작하는 실전 사례가 바로 **Event Sourcing**이다.

마지막으로 actor 기반 결제 시스템이 왜 "command → event → persist → state transition → replay" 구조를 택하는지를 **개념 수준에서만** 설명한다. 구체적인 구현 스키마나 API 세부 사항은 다루지 않고, 일반적인 패턴과 이렇게 설계하는 이유를 설명하는 데 집중한다.

---

## 1. Event Sourcing: 상태가 아니라 사실의 역사를 저장한다

### 잔액이 아니라 입출금 내역을 저장한다

가장 이해하기 쉬운 비유는 은행 통장이다. RDB 방식으로 계좌를 설계하면 보통 다음과 같이 만든다.

```text
accounts
+------------+---------+
| account_id | balance |
+------------+---------+
| acc-1      |  17,000 |   <- 현재 잔액만 들고 있다
+------------+---------+
```

입금이나 출금이 일어날 때마다 `UPDATE accounts SET balance = ? WHERE account_id = ?`로 값을 **덮어쓴다**. 이 모델에는 현재 상태(current state)만 남고, 그 상태에 이르게 된 과정은 사라진다.

잔액이 17,000원이라는 사실은 알 수 있지만, 그 값이 (입금 20,000 → 출금 3,000)의 결과인지 (입금 50,000 → 출금 33,000)의 결과인지는 통장 테이블만 봐서는 알 수 없다. 과정을 알고 싶다면 거래 로그 테이블을 따로 만들어야 한다.

Event Sourcing은 이 우선순위를 **뒤집는다**. 일차 데이터(source of truth)는 잔액이 아니라 거래 내역, 즉 **일어난 사실들을 순서대로 나열한 시퀀스(an ordered sequence of facts)** 이다.

```text
events (acc-1 의 journal)
+--------+----------------------+----------+
| seq_nr | event_type           | amount   |
+--------+----------------------+----------+
|   1    | Deposited            | +20,000  |
|   2    | Withdrawn            |  -3,000  |
|   3    | Deposited            |  +5,000  |
|   4    | Withdrawn            |  -5,000  |
+--------+----------------------+----------+

현재 잔액 = 폴드(fold)로 계산: 0 +20000 -3000 +5000 -5000 = 17,000
```

여기서 "현재 잔액 17,000원"은 **저장된 데이터가 아니라, 이벤트를 처음부터 차례로 접어(fold) 다시 계산한 파생값(derived value)** 이다. 함수형 언어로 표현하면 다음과 같다.

```text
currentState = events.foldLeft(initialState)(applyEvent)
```

이 한 줄에 Event Sourcing의 핵심 생각이 모두 들어 있다. 상태는 일급 시민이 아니며, 이벤트로부터 계산되는 함수값일 뿐이다.

### 세 가지 불변 규칙

Event Sourcing을 이루는 규칙은 단순하지만 강력하다.

1. **이벤트는 사실(facts)이다.** 이벤트 이름은 `OrderPlaced`, `PaymentApproved`, `MoneyWithdrawn`처럼 과거형으로 짓는다. 이벤트는 의도(command)가 아니라 이미 일어난 일이기 때문이다. 이름을 과거형으로 짓는 것은 단순한 관례가 아니라, "이미 일어난 일은 취소할 수 없다"는 의미상의 약속이다.
2. **이벤트 저장소는 append-only이다.** 한 번 기록한 이벤트는 절대 수정하거나 삭제하지 않는다. 잘못된 일이 일어났다면 그 이벤트를 지우는 대신 **보정하는 새 이벤트**(예: `PaymentRefunded`)를 덧붙인다. 회계 장부에서 잘못 적은 줄을 지우개로 지우지 않고, 빨간 줄로 정정 분개를 추가하는 것과 같다.
3. **현재 상태는 replay로 복원한다.** 시스템이 재시작되거나, 새 노드가 합류하거나, 메모리에서 passivate된 actor가 다시 깨어날 때, 저장된 상태를 불러오는 것이 아니라 **이벤트를 처음부터 다시 적용(replay)** 해서 상태를 재구성한다.

> [!warning] 흔한 오해
> "Event Sourcing은 결국 audit log를 잘 남기는 것 아닌가?"라고 생각하기 쉽지만 그렇지 않다. audit log는 보통 **부수적인 기록**이고, 실제 상태는 다른 곳(테이블의 current row)에 있다.
>
> Event Sourcing에서는 이벤트 로그가 **유일한 진실(the single source of truth)** 이고, 나머지(현재 상태, read model, 검색 인덱스)는 모두 그 로그에서 파생된 캐시일 뿐이다. 이 차이 때문에 운영, 복구, 디버깅 방식이 모두 달라진다.

### 왜 이렇게까지 하는가: 결제 시스템의 관점

상태를 덮어쓰는 모델의 가장 큰 문제는 **정보가 손실(lossy)** 된다는 점이다. `balance`를 덮어쓰는 순간 직전 잔액과 변화의 맥락이 사라진다. 결제 시스템에서는 이것이 치명적이다.

- "이 결제가 왜 취소 상태가 되었는가?"에 답하려면, 상태 모델에서는 별도의 로그를 뒤져야 하고 그 로그가 코드 변경 과정에서 빠졌을 수도 있다. Event Sourcing에서는 **상태 자체가 이벤트의 역사이므로** 답이 언제나 거기에 있다.
- "3개월 전 그 시점에 이 결제는 어떤 상태였는가?" 같은 시점 복원(temporal query)도 자연스럽다. 그 시점까지의 이벤트만 fold하면 된다. 상태 모델에서는 불가능하거나, 별도의 스냅샷 인프라가 필요하다.
- "환불까지 평균 며칠이 걸리는가?" 같은 새로운 비즈니스 질문이 생기면, 이미 저장된 과거 이벤트를 새 관점으로 다시 fold해 새 read model을 만들 수 있다. 상태 모델은 과거를 버렸기 때문에 앞으로 쌓일 데이터로만 답할 수 있다.

정리하면 Event Sourcing은 **"과거를 버리지 않는 대가로, 앞으로 생길 모든 질문에 답할 수 있는 능력을 얻는"** 방식이다. 그 대가가 이 장 후반부에서 다룰 replay 비용, 스키마 진화, projection의 복잡도이다.

---

## 2. 왜 Cassandra가 event store에 잘 맞는가

Event Sourcing의 저장 계층은 사실상 **로그 데이터베이스**이다. 이 워크로드의 특성은 매우 분명하다.

- **쓰기는 거의 전부 append이다.** 새 이벤트를 시퀀스 끝에 덧붙이는 것 말고는 쓰기가 없다. UPDATE도 DELETE도 (거의) 없다.
- **읽기는 두 가지뿐이다.** 하나는 특정 엔티티의 이벤트를 `seq_nr` 순서대로 처음부터(또는 스냅샷 이후부터) 끝까지 읽는 replay이고, 다른 하나는 태그나 시간을 기준으로 이벤트를 훑는 projection이다.
- **자연스러운 파티션 키가 이미 있다.** 엔티티 식별자(Akka에서는 `persistence_id`)가 그것이다. 한 엔티티의 모든 이벤트는 언제나 이 식별자 아래에 함께 모인다.
- **데이터가 한없이 커진다.** 시간이 지날수록 이벤트는 늘어나기만 한다. 따라서 수평 확장은 선택이 아니라 필수이다.

이 네 가지를 Cassandra의 설계 방향과 나란히 놓아 보면, 둘이 같은 설계도에서 나온 것처럼 보일 정도로 잘 맞는다.

### 2.1 LSM-tree와 append-only 쓰기의 구조적 궁합

8장(Storage Engine 내부)에서 본 내용을 다시 떠올려 보자. Cassandra의 write path는 다음과 같다.

```text
WRITE 한 번이 거치는 길 (Cassandra)

  client write
      |
      v
  +-----------------+      append (순차 디스크 쓰기, fsync)
  | Commit Log      |  <---------------------------------+
  +-----------------+                                    |
      |                                                  |
      v                                                  |
  +-----------------+   메모리, 정렬된 구조             durability
  | Memtable        |   (in-memory, sorted)              보장
  +-----------------+
      |  (가득 차면 flush)
      v
  +-----------------+   불변(immutable) 정렬 파일
  | SSTable (디스크) |   한 번 쓰면 절대 수정 안 함
  +-----------------+
```

**Cassandra의 쓰기는 근본적으로 append-only이다.** Commit log에는 순차적으로 append하고, memtable에는 메모리에서 정렬된 상태로 삽입하며, flush 결과인 SSTable은 **immutable**이다. UPDATE도 내부적으로는 새 버전을 덧쓰는 것이고, DELETE조차 9장(Compaction 전략)에서 본 tombstone(삭제를 표시하는 새 쓰기)으로 처리한다. Cassandra는 어떤 쓰기든 **디스크의 기존 위치를 in-place로 고치지 않는다.**

이제 Event Sourcing의 워크로드를 겹쳐 보자. Event Sourcing도 본질적으로 append-only이다. 따라서 **애플리케이션의 쓰기 패턴(append)과 스토리지 엔진의 쓰기 방식(append)이 임피던스 불일치 없이 정확히 맞아떨어진다.**

이 궁합이 왜 중요한지는 RDB와 비교해 보면 알 수 있다. 전통적인 RDB의 B-tree는 **in-place update**를 위해 설계되었다. 페이지를 찾아가 그 자리를 고치고 인덱스를 갱신한다.

이 과정에서 무작위 I/O(random I/O)가 생기고, 페이지 분할(page split)과 잠금 경합도 발생한다. 그런데 Event Sourcing은 애초에 in-place update를 **하지 않는** 워크로드다. B-tree가 비싸게 제공하는 "임의 위치 수정" 기능은 전혀 쓰지 않으면서, 그 비용(random I/O, 잠금)만 떠안게 된다.

반대로 LSM-tree는 수정하지 않고 계속 덧붙이다가, 정리는 나중에 한꺼번에(compaction) 하는 방식이다. 이 방식은 순차 I/O(sequential I/O) 위주로 동작하고, 쓰기 처리량이 높으며, write amplification을 compaction 시점으로 미룬다. Event Sourcing에서 쏟아지는 append를 받아내기에 이보다 자연스러운 구조는 없다.

```text
B-tree (RDB)              vs        LSM-tree (Cassandra)
-----------------                   --------------------
쓰기 = 제자리 수정                  쓰기 = 끝에 덧붙임
random I/O 위주                     sequential I/O 위주
페이지 분할/잠금 경합               memtable 정렬 삽입, 락 최소
읽기 빠름(인덱스 1회 탐색)          읽기는 여러 SSTable 병합 필요
ES와의 궁합: 비용만 부담            ES와의 궁합: 쓰기 패턴 = 엔진 패턴
```

> [!caution] 주의
> LSM이라고 해서 무조건 빠른 것은 아니다. LSM의 대가는 읽기 쪽에 있다. row 하나를 읽으려면 여러 SSTable에 흩어진 조각을 병합해야 할 수도 있다(10장 Write Path와 Read Path). 다만 Event Sourcing의 읽기는 "파티션 하나를 seq_nr 순서로 쭉 스캔"하는 형태이므로 LSM의 약점(무작위 point read)을 덜 건드린다. 이 균형은 뒤에서 compaction 전략 선택과 snapshot 설계로 이어진다.

### 2.2 파티션 = persistence id: 데이터 모델이 저절로 정해진다

5장(데이터 모델링 1 - Query First)의 핵심 교훈은 "쿼리를 먼저 정하고, 그 쿼리가 단일 파티션에서 해결되도록 파티션 키를 설계하라"였다. Event Sourcing에서는 이 작업이 거의 저절로 된다.

replay 쿼리는 언제나 "이 엔티티의 이벤트를 seq_nr 순서대로 모두 가져오라"이다. 따라서 파티션 키를 **엔티티 식별자(persistence_id)** 로, clustering key를 **seq_nr**로 잡으면 replay는 정확히 파티션 하나를 정렬된 순서로 순차 읽기 하는 작업이 된다. 이것은 Cassandra가 가장 잘하는 읽기 패턴, 즉 single-partition, clustering-order range scan이다.

```text
파티션 = 엔티티 하나의 평생 이벤트 로그

  partition key: persistence_id = "payment-9f3a..."
  +------------------------------------------------------+
  | seq_nr=1 | seq_nr=2 | seq_nr=3 | ... | seq_nr=N      |   <- clustering order
  +------------------------------------------------------+
       ^                                       ^
       |                                       |
   replay 시작                             replay 끝
   (스냅샷 있으면 그 이후부터)
```

RDB라면 events 테이블에 `(entity_id, seq_nr)` 복합 인덱스를 만들고, 그 인덱스를 따라 range scan을 했을 것이다. 잘 동작하지만 데이터가 커지면 단일 머신의 한계에 부딪힌다.

Cassandra는 **데이터를 파티션 단위로 토큰 링 위에 흩어 놓기 때문에**([[02 - 분산 아키텍처]]) 엔티티가 수억 개로 늘어나도 각 엔티티의 replay는 여전히 단일 파티션, 단일 노드(+복제본) 작업으로 격리된다. 엔티티 수가 늘어도 개별 replay 비용은 늘지 않고, 전체 처리량은 노드를 추가하는 만큼 선형으로 늘어난다. 이것이 "수평 확장"의 실제 의미이다.

### 2.3 시계열 append와 토큰 링

엔티티 식별자를 파티션 키로 쓰면, 서로 다른 엔티티의 이벤트는 2장(분산 아키텍처)에서 본 것처럼 partitioner(기본값은 Murmur3Partitioner)에 의해 토큰 링 전체에 고르게 분산된다.

```text
토큰 링 — Murmur3 해시 공간 [-2^63, 2^63-1] 위에 persistence_id 가 고르게 흩뿌려진다

         -2^63 / 2^63-1  (wrap-around 지점)
                 *
        payment-A2   payment-Z9
      *                       *
   node1                       node2
      *                       *
        payment-K7   payment-B1
                 *
              token 0
```

이 분산은 두 가지 효과를 함께 낸다.

1. 쓰기 부하가 한 노드에 몰리지 않는다(no hotspot). 수많은 엔티티가 동시에 이벤트를 append해도 각자 다른 파티션과 다른 노드로 흩어진다.
2. 한 엔티티 안에서는 seq_nr 순서대로 시계열 append가 보장된다.

여기서 자주 보이는 안티패턴을 짚고 넘어가자. "전역 이벤트 스트림을 파티션 하나에 모두 넣으면 순서가 완벽하게 보장되지 않을까?"라고 생각할 수 있다. 하지만 그렇게 하면 **모든 쓰기가 단 하나의 파티션, 단 하나의 노드(+복제본)로 몰린다.** 이것이 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]에서 경고한 hotspot이자 unbounded partition이다.

Event Sourcing은 순서를 엔티티 단위로만 보장하고, 전역 순서가 필요하면 별도의 tag·시간 버킷 테이블로 분리하는 방식(5절의 events-by-tag)으로 이 문제를 피한다.

### 2.4 높은 write throughput과 quorum 쓰기

마지막으로, event store는 보통 읽기보다 쓰기가 많거나, 적어도 쓰기가 매우 꾸준한 워크로드다(모든 비즈니스 사건이 이벤트가 되기 때문이다). Cassandra는 10장에서 본 대로 commit log append와 memtable 삽입만으로 쓰기를 끝내고 ack하며(나머지 flush와 compaction은 비동기로 처리한다), 따라서 쓰기 지연이 낮고 처리량이 높다.

또한 [[04 - Tunable Consistency]]의 tunable consistency 덕분에 event store는 보통 다음 조합을 쓴다.

- 이벤트 append: `LOCAL_QUORUM` 쓰기. 데이터센터 안에서 과반수 복제본에 기록되어야 ack하므로 이벤트 유실 위험이 낮아진다.
- replay 읽기: 보통 `LOCAL_QUORUM` 읽기를 써서 `R + W > RF`를 만족시키고, 방금 쓴 이벤트를 확실히 다시 읽도록 한다(read-your-writes).

결제처럼 기록한 이벤트가 사라지면 안 되고, 재시작 후 replay에서 그 이벤트가 반드시 보여야 하는 도메인에서는 이 일관성 조합이 사실상 필수이다. (Akka Persistence Cassandra 플러그인은 journal의 write/read consistency를 설정으로 제공하며, 안전을 위해 보통 quorum 계열로 둔다.)

---

## 3. Akka Persistence Cassandra: journal과 snapshot의 내부 스키마

이제 추상적인 설명에서 구체적인 구현으로 내려가자. Akka Persistence는 actor의 영속화를 추상화한 것이고, 그 백엔드 가운데 하나가 **akka-persistence-cassandra** 플러그인이다(지금은 Pekko 쪽의 pekko-persistence-cassandra로도 이어진다). 이 플러그인은 저장소 두 개를 제공한다.

- **journal**: 이벤트의 append-only 로그이며, event store 그 자체이다.
- **snapshot store**: 특정 시점의 상태를 통째로 저장한 체크포인트이며, replay 비용을 줄이는 최적화 장치이다.

> 아래 스키마는 개념을 설명하기 위해 **단순화한 형태**다. 실제 플러그인의 컬럼명, 타입, 버전별 세부 사항은 다를 수 있으므로, 정확한 DDL은 사용 중인 플러그인 버전의 문서에서 확인해야 한다. 여기서는 "왜 이런 컬럼이 필요한가"라는 의도를 이해하는 것이 목적이다.

### 3.1 journal 테이블의 핵심 컬럼

개념적으로 journal 테이블은 다음과 같은 구조다.

```cql
CREATE TABLE akka.messages (
    persistence_id  text,        -- 엔티티 식별자 (= actor)
    partition_nr    bigint,      -- 긴 로그를 여러 파티션으로 쪼개는 버킷 번호
    sequence_nr     bigint,      -- 엔티티 내부의 단조 증가 이벤트 순번
    timestamp       timeuuid,    -- 쓰기 시각(정렬/태그용)
    event           blob,        -- 직렬화된 이벤트 본문
    ser_id          int,         -- 직렬화기(serializer) 식별자
    ser_manifest    text,        -- 직렬화 manifest(타입/버전 힌트)
    tags            set<text>,   -- projection을 위한 태그
    PRIMARY KEY ((persistence_id, partition_nr), sequence_nr)
) WITH CLUSTERING ORDER BY (sequence_nr ASC);
```

각 컬럼이 왜 있는지 하나씩 살펴보자.

- **`persistence_id`**: 엔티티(=actor)의 고유 식별자이다. "결제 9f3a의 모든 이벤트는 여기에 모인다"는 단위가 되며, 파티션 키의 일부이다.
- **`partition_nr`**: 이 스키마에서 가장 중요한 설계 결정이며, 3.2에서 자세히 다룬다. 파티션 키의 나머지 일부이다.
- **`sequence_nr`**: 엔티티 안에서 1부터 단조 증가하는 이벤트 순번이다. clustering key이므로 **저장될 때부터 seq 순서로 정렬**되며, replay는 이 순서대로 읽기만 하면 된다. 또한 이 번호는 **멱등성과 순서 보장의 핵심**이다(7절에서 설명한다).
- **`event` (blob) + `ser_id` + `ser_manifest`**: 이벤트 본문은 도메인 타입을 직렬화한 바이트 덩어리(blob)이다. Cassandra는 이 바이트의 의미를 모르고 보관만 한다. 역직렬화할 때 어떤 직렬화기로, 어떤 타입과 버전으로 풀어야 하는지는 `ser_id`와 `ser_manifest`가 알려 준다. 이 두 컬럼이 4절에서 다룰 **스키마 진화**의 토대가 된다.
- **`tags`**: 이 이벤트가 어떤 projection 스트림에 속하는지를 나타내는 라벨이다. "이 이벤트는 `payment` 태그와 `vbank` 태그에 속한다"는 식으로 쓰며, events-by-tag 조회의 입력이 된다(2.3절, 5절).
- **`timestamp` (timeuuid)**: 쓰기 시각이다. tag 스트림의 시간순 정렬, 디버깅, projection 오프셋 추적에 쓰인다.

여기서는 PRIMARY KEY `((persistence_id, partition_nr), sequence_nr)`의 구조가 중요하다. 파티션 키는 `(persistence_id, partition_nr)`의 복합 키이고, 그 안에서 `sequence_nr`로 정렬된다. 즉 **한 엔티티의 이벤트 로그가 partition_nr 단위로 여러 파티션에 나뉘어 저장되며, 각 파티션 안에서는 seq 순서가 보장된다.**

### 3.2 partition_nr: large partition을 피하는 결정적 장치

5장(데이터 모델링 1 - Query First)과 [[12 - 운영과 트러블슈팅]]에서 반복해서 경고한 함정이 **large partition, unbounded partition**이다. 파티션 하나가 한없이 커지면 다음과 같은 문제가 생긴다.

- 파티션 전체를 읽거나 compaction할 때 메모리와 GC에 부담이 커진다. (Cassandra는 파티션을 처리 단위로 다룬다.)
- 파티션 하나는 단일 노드(+복제본)에 저장된다. 따라서 파티션 하나가 거대해지면 그 노드만 비대해지는 불균형이 생긴다.
- 파티션 안의 row가 수십만~수백만 개가 되면 읽을 때 partition index 탐색과 SSTable 병합 비용이 커진다.
- 운영상 기준선: 이상적으로는 파티션당 100MB, 10만 row 이내를 목표로 삼는다. 수백 MB, 수백만 row를 넘어서기 시작하면 GC 부담, 읽기 지연, 복구(스트리밍) 비용이 급격히 나빠진다.

그런데 Event Sourcing의 엔티티는 **오래 살수록 이벤트가 끝없이 쌓인다.** 활성 사용자 한 명이나 오래된 계좌 하나가 몇 년 동안 수십만 개의 이벤트를 만들 수 있다. persistence_id만 파티션 키로 쓰면 이 엔티티의 파티션이 unbounded로 자라, 위의 문제를 모두 그대로 겪게 된다.

`partition_nr`이 바로 이 문제의 해법이다. 플러그인은 설정값(예: `target-partition-size`, 기본값은 대략 수십만 이벤트 단위)에 따라, seq_nr이 일정 개수를 넘으면 partition_nr을 증가시켜 **같은 엔티티의 로그를 여러 파티션으로 나눈다(bucketing).**

```text
persistence_id = "account-42", target-partition-size = 500,000 (예시)

partition_nr=0 :  seq 1       ... seq 500,000
partition_nr=1 :  seq 500,001 ... seq 1,000,000
partition_nr=2 :  seq 1,000,001 ...
                      ^
                      |
   각 partition_nr 이 Cassandra의 독립 파티션이 된다
   => 어떤 파티션도 ~500,000 row 를 넘지 않게 bounded
```

이렇게 하면 끝없이 긴 논리적 로그가 **크기가 제한된(bounded) 물리적 파티션의 연속**으로 바뀐다. partition_nr은 seq_nr로부터 산술적으로 계산할 수 있으므로(예: `partition_nr = (seq_nr - 1) / target_size`), replay할 때 어떤 파티션을 어떤 순서로 읽어야 하는지 코드가 결정적으로 알 수 있다. **partition_nr=0, 1, 2... 를 차례로 range scan**하면 전체 로그를 seq 순서대로 복원할 수 있다.

이것은 5장의 "쿼리 우선 설계"와 6·12장의 "large partition 회피"가 실전에서 어떻게 구현되는지를 보여 주는 교과서적인 사례이다. 끝없이 자라는 시계열을 시간이나 개수 기준의 버킷으로 나누는 기법은 시계열 데이터 모델링의 정석이며, partition_nr은 그 버킷 번호이다.

> [!warning] 함정
> target-partition-size를 너무 작게 잡으면 한 엔티티의 replay가 너무 많은 파티션(=너무 많은 개별 쿼리와 디스크 탐색)으로 쪼개져 오히려 replay가 느려진다. 너무 크게 잡으면 large partition 위험이 생긴다. 일반적으로는 "한 엔티티가 평생 만들 이벤트 수"와 "snapshot 주기"를 함께 고려해 정한다.
>
> snapshot을 자주 찍으면 replay는 마지막 snapshot 이후의 파티션 몇 개만 읽으면 되므로, 실제 replay 비용은 partition 분할 개수보다 snapshot 전략에 더 크게 좌우된다(3.3).

### 3.3 snapshot store: replay 비용을 줄이는 체크포인트

엔티티에 이벤트가 수십만 개 쌓였다면, 재시작할 때마다 그 전부를 처음부터 fold하는 것은 비싸다(이벤트가 N개면 N번 적용해야 한다). snapshot은 이 비용을 줄이는 장치다.

```cql
CREATE TABLE akka.snapshots (
    persistence_id  text,
    sequence_nr     bigint,     -- 이 스냅샷이 커버하는 마지막 이벤트 순번
    timestamp       bigint,
    snapshot        blob,       -- 직렬화된 "상태" 통째
    ser_id          int,
    ser_manifest    text,
    PRIMARY KEY (persistence_id, sequence_nr)
) WITH CLUSTERING ORDER BY (sequence_nr DESC);
```

snapshot은 **이벤트가 아니라 상태**를 저장한다는 점에 주목하자. "seq 480,000 시점에 이 계좌의 전체 상태는 이러했다"를 통째로 직렬화해 둔 것이다. clustering order가 `DESC`인 이유는 가장 최신 snapshot 1건을 맨 앞에서 빠르게 읽기 위해서다.

복원 절차는 다음과 같다.

```text
재시작 시 actor 상태 복원

  1. snapshot store 에서 가장 최신 snapshot 1건 로드  (seq=480,000)
  2. 그 상태를 시작점으로 삼는다
  3. journal 에서 seq > 480,000 인 이벤트만 replay
       (480,001, 480,002, ... 현재까지)
  4. 완성된 현재 상태

  => 480,000 개를 다시 적용하는 대신, 그 이후 몇십~몇백 개만 적용
```

여기서 Event Sourcing의 미묘하지만 결정적인 설계 원칙이 드러난다. **snapshot은 최적화 수단일 뿐, 진실의 원천이 아니다.** 진실은 언제나 journal(이벤트 로그)에 있다. snapshot이 손상되거나 삭제되어도 시스템은 journal을 처음부터 replay해서 같은 상태를 복원할 수 있다. 그래서 snapshot은 부담 없이 지울 수 있는 "버려도 되는 캐시"이다.

이 비대칭(journal은 절대 건드리지 않고, snapshot은 폐기할 수 있음)은 4절의 스키마 진화 규칙에도 영향을 준다. snapshot 포맷은 비교적 자유롭게 바꿀 수 있지만(필요하면 버리고 다시 만들면 되기 때문이다), journal의 이벤트 포맷은 영원히 호환되어야 한다.

> [!note] 운영 팁
> snapshot 주기는 "이벤트 N건마다" 또는 "특정 상태 전이 시점"으로 잡는다. 결제 actor라면 결제 완료나 취소처럼 안정된 상태에 도달했을 때 snapshot을 찍는 것이 합리적이다. snapshot이 잦으면 replay는 빨라지지만 snapshot 쓰기와 저장 비용이 늘고, 드물면 그 반대가 된다.
>
> 또한 오래된 snapshot은 정리(`deleteSnapshots`)해서 snapshot 테이블이 끝없이 커지지 않게 한다. journal은 보통 지우지 않지만 snapshot은 지워도 안전하다는 점을 기억하자.

---

## 4. 직렬화와 이벤트 스키마 진화: append-only가 요구하는 규율

이제 이 장에서 실무적으로 가장 까다로운 주제로 들어간다. journal에 저장된 `event` 컬럼은 단순한 blob이고, Cassandra는 그 내용을 모른다. 의미를 부여하는 것은 전적으로 애플리케이션의 직렬화 계층이다. 여기서 Event Sourcing의 가장 무거운 제약이 생긴다.

### 4.1 replay라는 제약이 모든 것을 결정한다

한 가지 사실을 다시 확인하자. **이벤트는 append-only이고, 현재 상태는 그 이벤트들을 replay해서 만든다.** 이 두 문장을 합치면 다음 결론이 나온다.

> **3년 전에 저장된 이벤트도, 오늘 배포된 최신 코드가 역직렬화하고 해석할 수 있어야 한다.**

상태를 덮어쓰는 모델에서는 스키마를 바꿀 때 마이그레이션을 한 번 돌려 모든 row를 새 포맷으로 갱신하면 끝난다. 과거 데이터가 사라졌으니 과거 포맷을 계속 지원할 필요가 없다. 하지만 Event Sourcing에서는 **과거의 모든 이벤트가 영원히 남아 있고, 재시작할 때마다 다시 읽힌다.** 따라서 이벤트 스키마는 한번 배포되면 **영원히 하위 호환(backward compatible)** 을 유지해야 한다.

이 제약이 직렬화기 선택을 좌우하며, 스키마 진화에 강한 포맷이 선호된다.

- **Protobuf / Avro / Thrift**: 필드에 번호 태그를 붙이고, 알 수 없는 필드는 무시하며, optional 필드는 기본값으로 채운다. 그래서 스키마 진화에 강하고, 이벤트 직렬화에 널리 쓰인다.
- **JSON**: 사람이 읽기 좋고 유연하지만 타입 안정성이 약하고 크기가 크다. 작은 시스템에서는 쓰이지만, 대규모 event store에서는 크기와 성능이 부담이 된다.
- **Java 기본 직렬화**: **금기다.** 클래스 구조가 바뀌면 과거 바이트를 읽지 못한다. Event Sourcing에서는 절대 쓰지 않는다.

### 4.2 호환성 규칙: 무엇이 허용되고 무엇이 금지되는가

Protobuf를 예로 들면, 이벤트 스키마를 바꿀 때 허용되는 것과 금지되는 것은 다음과 같다.

```text
이벤트 스키마 진화 규칙 (append-only + replay 제약하에서)

  ✅ 허용
    - 새 optional 필드 추가          (옛 이벤트엔 없음 -> 기본값으로 읽힘)
    - 새 이벤트 타입 추가             (oneof / 새 ser_manifest)
    - 사용 안 하는 optional 필드 무시

  ❌ 금지
    - 기존 필드 제거                 (옛 이벤트엔 그 필드가 있다)
    - 필드 번호(tag) 재사용/변경      (옛 바이트 해석이 깨진다)
    - 필드 타입 변경 (int32 -> int64) (replay 시 디코딩 실패)
    - 필드의 의미(semantics) 변경     (같은 필드인데 뜻이 달라짐 = 조용한 데이터 오염)
```

가장 위험한 것은 마지막 항목인 **의미 변경**이다. 컴파일도 되고 역직렬화도 되지만, 같은 필드가 옛 이벤트와 새 이벤트에서 다른 뜻을 가지면 replay로 만든 상태가 아무 경고 없이 틀어진다. 이런 문제는 컴파일러로도 테스트로도 잡기 어렵다. 그래서 이 규율은 **"이벤트는 append-only로 취급하라. 필드도 의미도 추가만 하라"** 로 요약된다.

### 4.3 이 규율은 event store와 관계없이 따라온다

여기서 기억할 점이 있다. 이 backward compatibility는 특정 플러그인이나 특정 팀의 관례가 아니다. **Akka Persistence Cassandra가 아닌 어떤 event store를 쓰더라도, Event Sourcing을 택하는 순간 이 규율은 물리 법칙처럼 따라온다.**

이유는 항상 같은 순서로 이어진다. 상태를 이벤트 replay로 복원하므로(상태 = 이벤트의 replay) 과거 이벤트가 영원히 다시 읽힌다. 따라서 이벤트 스키마는 영원히 하위 호환이어야 하고, 필드 제거, 타입 변경, 번호 재사용, 의미 변경이 모두 금지된다(4.1, 4.2). Protobuf든 Avro든, event sourcing을 쓰는 도메인 이벤트 정의는 모두 같은 제약을 받는다.

이 모든 규칙의 뿌리는 "이벤트 replay로 상태를 재구성하므로, 호환성을 깨는 변경은 곧 과거 데이터를 망가뜨린다"는 한 문장이다.

8장(Storage Engine 내부)의 SSTable이 immutable이듯, Event Sourcing의 이벤트도 immutable이다. 한쪽은 스토리지 엔진 수준의 불변성이고 다른 쪽은 도메인 수준의 불변성이지만, 둘 다 "한번 쓴 것은 고치지 않고 새로 덧붙인다"는 같은 원칙을 따른다. 이 일관된 원칙 덕분에 Cassandra와 Event Sourcing의 조합은 개념적으로도 깔끔하게 맞아떨어진다.

---

## 5. 이벤트 조회와 projection: events-by-tag와 CQRS

지금까지는 한 엔티티의 이벤트를 그 엔티티 기준으로 다시 읽는 경로(replay)만 보았다. 그런데 실전에서는 **여러 엔티티에 걸친 조회**가 반드시 필요하다. "오늘 발생한 모든 결제 완료 이벤트"나 "취소된 모든 주문" 같은 질문이 그렇다. 이런 질문은 여러 엔티티에 걸쳐 있기 때문에 persistence_id로는 답할 수 없다.

### 5.1 왜 별도의 tag 테이블이 필요한가

5장(데이터 모델링 1 - Query First)의 원칙을 떠올려 보자. "다른 키로 조회하려면 다른 테이블이 필요하다." Cassandra에는 임의 컬럼 필터링이 없으므로 조회 패턴마다 그에 맞는 비정규화 테이블을 따로 둔다.

legacy secondary index가 있기는 하지만, 분산 환경에서는 scatter-gather 방식이라 비싸고 위험하다. Cassandra 5.0의 SAI(Storage-Attached Index)가 이 비용을 크게 줄였지만, 시간순으로 끝없이 스트리밍하면서 offset에서 재개해야 하는 projection 용도에는 여전히 인덱스보다 전용 tag 테이블 설계가 적합하다.

그래서 Akka Persistence Cassandra는 journal과 별도로 **events-by-tag**용 테이블(보통 `tag_views` 계열)을 유지한다. 이벤트에 `tags`를 달면 플러그인이 그 이벤트를 태그별 테이블에도 기록한다. 이 테이블의 파티션 키는 대략 `(tag, 시간 버킷)`이고, clustering은 시간 순서(timeuuid)이다.

```text
journal (엔티티 기준)              tag_views (태그 기준, projection 용)
파티션키: (persistence_id, part)   파티션키: (tag, time_bucket)
정렬: sequence_nr                  정렬: timestamp(timeuuid)

  payment-A: e1, e2, e3            tag="payment", bucket=2026-06-26-10:
  payment-B: e1, e2                  [payment-A:e2, payment-B:e1, payment-A:e3, ...]
                                       ^ 여러 엔티티의 이벤트가 시간순으로 섞여 한 스트림
```

여기서도 시계열 버킷팅이 등장한다(2.3, 3.2와 같은 기법). 태그 스트림에는 모든 엔티티의 이벤트가 모이므로, 그대로 두면 거대한 파티션이 된다. 그래서 시간 버킷(예: 시간 단위, 분 단위)으로 파티션을 나눈다. read journal은 버킷을 차례로 따라가며 "이 태그에서 이 시각 이후의 이벤트"를 스트리밍한다.

### 5.2 eventual consistency라는 본질적 함정

여기에는 반드시 이해해야 할 함정이 있다. **journal 쓰기와 tag_views 쓰기는 하나의 원자적 트랜잭션이 아니다.** 이벤트는 먼저 엔티티의 journal에 기록되고, tag_views로의 전파는 별도의 tag writer가 비동기로 처리한다(설정할 수 있는 지연, 예를 들어 플러그인의 `events-by-tag.eventual-consistency-delay`만큼 묶어서 처리한다). 즉 **events-by-tag 스트림은 eventually consistent**이다.

이것이 뜻하는 바는 다음과 같다.

- 엔티티에 이벤트를 append한 직후 그 태그 스트림을 조회하면, 방금 기록한 이벤트가 전파 지연 때문에 아직 보이지 않을 수 있다.
- 따라서 projection을 방금 일어난 일을 즉시 반영하는 강한 일관성 뷰로 기대해서는 안 된다. projection은 **조금 뒤처지는(lagging) read model**이다.
- read journal은 이런 상황을 다루기 위해 오프셋(offset, 보통 timeuuid 기반)을 추적하며 스트림을 재개할 수 있도록 설계되어 있고, 소비자(projection)는 자신이 어디까지 처리했는지를 저장한다.

이 모든 것은 **CQRS(Command Query Responsibility Segregation)** 라는 큰 그림의 일부이다.

```text
CQRS + Event Sourcing 의 두 길

  쓰기(Command) 측                  읽기(Query) 측
  ----------------                  ----------------
  command -> actor                  events-by-tag 스트림 구독
  -> 이벤트 검증/생성               -> 이벤트를 받아 read model 갱신
  -> journal append (진실)          -> 조회 최적화된 테이블/검색엔진/캐시
       |                                 ^
       |   tag_views 비동기 전파         |
       +---------------------------------+
            (eventually consistent)

  진실의 원천 = journal
  read model = journal 에서 파생된, 뒤처질 수 있는 투영
```

CQRS는 쓰기 모델(이벤트, 정규화)과 읽기 모델(조회 최적화, 비정규화)을 분리하는 방식이다. Event Sourcing은 자연스럽게 CQRS로 이어진다. 이벤트 로그가 쓰기 모델이고, 그로부터 만든 모든 read model이 읽기 모델이다. read model은 5장의 "쿼리마다 테이블 하나" 원칙에 따라 조회 패턴별로 여러 개를 둘 수 있으며, 각각은 같은 이벤트 스트림에서 독립적으로 만들어진다.

### 5.3 멱등 consumer: at-least-once 전달에서 피할 수 없는 요구

projection consumer는 이벤트를 받아 read model을 갱신한다. 그런데 분산 시스템에서 메시지 전달은 보통 **at-least-once**이다. 장애, 재시작, 재구독 과정에서 같은 이벤트가 두 번 이상 전달될 수 있다. 따라서 consumer는 반드시 **멱등(idempotent)** 해야 한다.

다행히 이벤트에는 `(persistence_id, sequence_nr)`라는 자연 키가 있다. consumer는 "이 persistence_id에서 마지막으로 처리한 seq_nr"을 저장해 두고, 그보다 작거나 같은 seq_nr이 다시 오면 무시하면 된다. 또는 read model 갱신 자체를 멱등 연산(upsert, set 연산)으로 설계한다.

```text
멱등 projection consumer

  받은 이벤트: (persistence_id=P, seq_nr=S)
  if S <= last_processed[P]:   이미 처리함 -> skip
  else:
     read model 갱신 (가능하면 upsert)
     last_processed[P] = S
```

이것은 [[11 - LWT Batch Counter 내부]]에서 본 멱등성과 중복 처리 문제, 그리고 결제 시스템의 webhook 멱등 소비자(중복 webhook을 sequence나 idempotency 키로 거르는) 패턴과 정확히 같은 사고방식이다. **분산 시스템에서 "정확히 한 번"은 거의 언제나 "최소 한 번 + 멱등 소비자"로 구현된다.**

---

## 6. compaction과 tombstone 관점: event store는 성격이 다르다

9장(Compaction 전략)에서는 STCS, LCS, TWCS, UCS를 다뤘다. event store의 워크로드에서는 이 선택이 일반적인 경우와 달라진다. 그 차이는 결국 한 가지 사실에서 나온다. **이벤트는 거의 삭제하지 않는다.**

### 6.1 tombstone이 거의 없다는 사실의 의미

9장에서 본 대로 Cassandra의 DELETE는 tombstone(삭제 마커)을 쓰는 동작이다. tombstone이 쌓이면 읽을 때 스캔 비용이 커지고, tombstone은 `gc_grace_seconds`(기본 864000초 = 10일) 동안 남아 있다가 compaction으로 정리된다. tombstone 급증은 12장(운영과 트러블슈팅)에서 자주 다룬 장애 원인이다.

그런데 순수한 event store(journal)에는 **DELETE가 거의 없다.** 이벤트는 append만 하고 지우지 않는 것이 원칙이기 때문이다. 이 사실에서 다음 결과가 나온다.

- **tombstone 부담이 본질적으로 낮다.** read path가 쌓인 tombstone을 스캔하느라 느려지는 일이 거의 없다. 워크로드 자체가 LSM-tree의 약점 가운데 하나(tombstone)를 피해 간다는 뜻이다.
- 따라서 event store의 compaction은 tombstone을 빨리 치우는 것보다 **읽기 효율(한 파티션의 조각을 적은 수의 SSTable로 모으는 것)** 을 주된 목적으로 삼게 된다.

### 6.2 그렇다면 어떤 compaction 전략을 쓰는가

이 점에서 TWCS와 결정적인 차이가 생긴다. 9장에서는 TWCS(TimeWindowCompactionStrategy)가 시간 버킷별로 SSTable을 묶고 TTL로 통째로 만료시켜 버킷 단위로 drop하므로, 시계열이면서 만료되는 워크로드(로그, 메트릭)에 가장 적합하다고 했다.

event store도 시계열처럼 보이지만 **TTL로 만료시키지 않는다**는 점에서 TWCS의 전제와 맞지 않는다. 시간이 지나 이벤트가 자동으로 삭제되면 replay로 과거를 복원할 수 없게 되므로, event store의 본질과 충돌한다.

그래서 일반적인 선택은 다음과 같다.

```text
event store(journal) compaction 선택의 직관

  TWCS  : TTL 만료 + 버킷 drop 이 핵심.
          이벤트는 만료 안 시킴 -> 전제 불일치. 부적합.

  STCS  : 쓰기 효율 좋고 단순. 같은 크기 SSTable 모일 때 합침.
          쓰기 폭주(append) 받아내기엔 무난.
          단, 한 파티션 조각이 여러 SSTable 에 흩어져
          replay(파티션 전체 읽기) 시 SSTable 여러 개 터치 가능.

  LCS   : 레벨별로 SSTable 을 겹치지 않게 정리.
          "한 파티션의 데이터가 적은 수의 SSTable에 모이도록"
          유지 -> 파티션 단위 range scan(=replay) read 효율 좋음.
          대가는 compaction(쓰기 증폭) 비용 증가.
```

선택은 상황에 따라 갈린다. **읽기(replay) 지연의 안정성을 중시하면 LCS**, **쓰기 처리량과 단순함을 중시하면 STCS** 쪽으로 기운다. 결제 actor처럼 엔티티 단위 replay가 잦고 latency가 중요한 경우에는 LCS의 읽기 효율이 매력적이다. 반면 쓰기가 많고 snapshot으로 replay 범위를 짧게 유지한다면 STCS의 낮은 쓰기 증폭이 더 나을 수 있다. 결국 정답은 워크로드를 측정해 봐야 알 수 있다.

> [!note] Cassandra 5.0 메모
> Cassandra 5.0에는 UCS(Unified Compaction Strategy)가 도입되어, 파라미터화된 전략 하나로 STCS와 LCS 사이의 스펙트럼을 스케일 팩터로 조절할 수 있다(9장 Compaction 전략 참조). event store에서는 UCS를 써서 쓰기 증폭과 읽기 증폭 사이의 트레이드오프를 워크로드에 맞게 튜닝할 수 있다.
>
> 다만 어떤 전략을 쓰든 "이벤트는 만료시키지 않으므로 TTL 기반 drop에 의존하지 않는다"는 원칙은 같다. (UCS의 세부 기본 파라미터는 버전과 배포판에 따라 다를 수 있으므로, 실제 값은 문서에서 확인해야 한다.)

snapshot 테이블은 사정이 조금 다르다. snapshot은 오래된 것을 지우므로(3.3) tombstone이 생긴다. 다만 journal보다 양이 훨씬 적고 패턴이 단순해서, 보통 기본값인 STCS로 충분하다.

### 6.3 snapshot은 compaction이 아니라 replay 비용을 줄인다

여기서 두 개념을 혼동하지 말자. **compaction은 SSTable 정리(스토리지 수준)** 이고, **snapshot은 replay 단축(애플리케이션 수준)** 이다. 둘 다 쌓인 것을 정리해 비용을 낮춘다는 점에서 비슷하지만, 동작하는 층위가 다르다.

```text
   두 가지 "정리" 메커니즘 — 층위가 다르다

  애플리케이션 레벨   snapshot  : 이벤트 N개 replay 대신 상태 1개 + 그 이후만
  스토리지 레벨       compaction: 흩어진 SSTable 조각을 병합해 read 효율 회복
```

event store에서 read latency를 좌우하는 실제 요인은 대개 **snapshot 주기**이다. snapshot을 충분히 자주 찍으면 replay가 짧아지므로, compaction 전략에 따른 읽기 효율 차이의 영향이 줄어든다. 그래서 실전 튜닝은 보통 snapshot 주기를 먼저 정하고, compaction 전략은 그다음에 정하는 순서로 진행한다.

---

## 7. 일관성과 순서: sequence_nr가 뒷받침하는 정확성

Event Sourcing의 정확성은 **순서(order)** 와 **멱등성(idempotency)** 이라는 두 가지 보장에 기대고 있다. 이 두 가지를 모두 `sequence_nr`이 뒷받침한다.

### 7.1 단일 파티션 append의 순서 보장

한 엔티티의 모든 이벤트는 같은 파티션(들)에 `sequence_nr` clustering order로 저장된다(3.1). Cassandra는 한 파티션 안에서 clustering key 순서대로 데이터를 정렬하고 저장하고 반환하므로, **한 엔티티의 이벤트 순서는 물리적으로 보장된다.** replay는 언제나 seq 1, 2, 3... 순서로 읽는다.

다만 이 순서 보장은 **엔티티(파티션) 단위로만** 성립한다. 서로 다른 엔티티 사이의 전역 순서는 Cassandra가 보장하지 않으며, 보장할 필요도 없다. 도메인에서 서로 다른 엔티티의 사건은 독립적이기 때문이다. 전역 시간순이 필요하면 5절의 tag 스트림(timeuuid 기준)을 쓰되, 그것이 eventual consistency라는 약한 보장이라는 점을 받아들여야 한다.

> RDB라면 전역 auto-increment id나 트랜잭션 격리로 전역 순서를 쉽게 보장할 수 있다. 하지만 이 편리함에는 단일 쓰기 지점(중앙 시퀀스, 단일 마스터)이 필요하고, 그 지점이 곧 확장의 병목이 된다. Cassandra와 Event Sourcing은 순서 보장 범위를 엔티티 단위로 좁히는 대가로 무한한 수평 확장을 얻는다. 이것은 손해가 아니라 **의도적인 설계 트레이드오프**이다.

### 7.2 sequence_nr로 만드는 멱등성과 동시성 안전

이벤트를 append할 때 actor는 다음 seq_nr이 N이어야 한다는 것을 알고 있다(지금까지 적용한 마지막 seq + 1). 여기서 흔히 "같은 `(persistence_id, partition_nr, seq_nr)`로 두 번 쓰면 Cassandra가 중복을 막아 주지 않을까?"라고 오해한다.

**그렇지 않다.** Cassandra의 일반 쓰기는 **upsert**이므로, 같은 PRIMARY KEY에 대한 두 번째 쓰기는 충돌로 거부되지 않고 셀 타임스탬프 기반 last-write-wins에 따라 **아무 경고 없이 덮어쓴다**(10장 Write Path와 Read Path). 즉 DB는 seq_nr 중복을 스스로 감지하지 못한다(감지하려면 `IF NOT EXISTS` 같은 LWT가 필요하지만, journal append는 기본적으로 LWT를 쓰지 않는다).

그래서 순번이 단조 증가하고 충돌하지 않도록 보장하는 것은 DB가 아니라 **애플리케이션**이다. Akka Persistence는 단일 writer 가정(한 시점에 한 엔티티에는 actor 인스턴스 하나만 쓴다)을 두고, 그 actor가 메모리에 들고 있는 마지막 seq에 1을 더해 쓴다. 따라서 같은 seq_nr이 두 번 발급될 일이 애초에 없다.

플러그인은 쓰기마다 writer를 식별하는 메타데이터(writer UUID)를 함께 기록해, 이 가정이 깨져 둘 이상이 같은 엔티티에 동시에 쓰는 비정상 상황을 나중에 찾아낼 수 있는 단서를 남긴다.

이 "엔티티당 단일 writer"는 다음 절에서 다룰 actor 모델의 동시성 안전성과 바로 연결된다. 또한 단일 writer 가정 덕분에 event store는 쓰기마다 11장(LWT Batch Counter 내부)의 LWT(Paxos 기반 compare-and-set)를 쓸 필요가 없다. LWT는 여러 번의 라운드트립이 필요해 비싸다. 대신 "엔티티당 actor 하나"라는 애플리케이션 수준의 직렬화로 동시성 충돌을 원천적으로 막고, 일반 quorum 쓰기로 빠르게 append한다.

이것이 actor 모델과 event sourcing 조합의 영리한 점이다. **DB 수준의 무거운 합의 대신 애플리케이션 수준의 actor 직렬화로 정확성을 얻는다.**

---

## 8. 개념 연결: actor 기반 결제 시스템은 왜 이렇게 설계되는가

이제 지금까지 본 내용을 actor 기반 결제 시스템의 큰 그림에 **개념 수준에서만** 맞춰 보자. (구체적인 구현, 스키마, API는 다루지 않고, 일반적인 패턴과 그 이유만 살펴본다.)

### 8.1 command → event → persist → state transition

actor 기반 event-sourced 시스템에서 사건 하나가 처리되는 일반적인 흐름은 다음과 같다.

```text
   하나의 명령이 처리되는 보편 루프 (event-sourced actor)

  command 도착 (예: "이 결제를 승인하라")
      |
      v
  [1] 현재 상태(메모리에 복원된)로 명령 검증
      "이미 결제됐나? 취소된 상태인가?"  <- 비즈니스 규칙
      |
      v
  [2] 유효하면 이벤트 생성 (예: PaymentApproved)
      이벤트 = 일어난 사실
      |
      v
  [3] 이벤트를 journal 에 persist (Cassandra append)
      여기 성공해야 다음으로 진행 (durability 경계)
      |
      v
  [4] 영속화 성공 후 상태 전이 (state transition)
      메모리 상태를 이벤트 적용으로 갱신
      |
      v
  [5] 응답 / 부수효과(webhook 등) 발행
```

핵심은 [3]과 [4]의 순서이다. **이벤트를 먼저 영속화하고, 그다음에 상태를 바꾼다.** 영속화에 실패하면 상태를 바꾸지 않는다. 이렇게 해서 메모리 상태와 저장된 이벤트가 어긋나는 상황을 막는다. 재시작하면 [4]의 메모리 상태는 사라지지만 [3]의 이벤트는 남아 있으므로, replay로 정확히 같은 상태를 복원할 수 있다. 메모리 상태는 언제나 journal로부터 계산된 값이라는 불변식이 유지된다.

### 8.2 재시작 = replay로 상태 복원

actor가 메모리에서 내려갔다가(passivation, 장애, 배포) 다시 깨어나면, 자신의 persistence_id로 snapshot과 그 이후의 이벤트를 읽어 상태를 재구성한다(3.3). 이 방식은 결제 시스템 운영에 큰 안정감을 준다. 어떤 pod가 죽거나 어떤 노드가 재시작되어도, 결제 actor는 자신이 마지막으로 알던 정확한 상태로 돌아온다. 진실이 메모리가 아니라 Cassandra의 이벤트 로그에 있기 때문이다. 상태를 잃을까 봐 걱정할 필요가 없다.

### 8.3 단일 actor 직렬 처리 = 동시성 문제와 중복 결제 방지

가장 결정적인 설계 이유는 다음과 같다. **결제(엔티티) 하나는 actor 하나가 책임지고, 그 actor는 자신에게 온 명령을 메일박스에서 한 번에 하나씩 순서대로 처리한다.** 같은 결제에 대한 명령 두 개가 동시에 와도, actor 메일박스에 줄을 서서 차례로 처리된다.

```text
   동시 요청 두 개 — 같은 결제에 대해

  요청1: "승인"  --+
                   |   같은 persistence_id -> 같은 actor 메일박스
  요청2: "승인"  --+

  actor 처리 (순차):
    요청1 처리: 상태 확인(미결제) -> PaymentApproved 영속화 -> 상태=Paid
    요청2 처리: 상태 확인(이미 Paid!) -> 거부 (중복 결제 방지)
```

이 점에서 sleep이나 DB row lock 같은 잠금·대기로 경합을 막으려던 전통적인 방식과 근본적으로 다르다. 그런 방식은 비결정적이고 깨지기 쉬운 race 방어였다. actor와 event sourcing은 **race가 아예 생길 수 없는 구조**를 만든다. 동시성을 막는 것이 아니라, 직렬화로 없애는 것이다.

더블클릭, 브라우저 뒤로 가기로 인한 재결제, webhook과 redirect의 경합처럼 결제 시스템에서 오래된 골칫거리들이 "엔티티당 단일 actor의 직렬 처리 + 영속화된 상태를 이용한 멱등 검증"이라는 한 가지 원리로 해결된다.

이 모든 것을 뒷받침하는 저장 계층의 요구 사항은 엔티티별로 분리된 파티션, 빠른 append, 순서 보장, 영속성, 수평 확장이다. 이것은 정확히 2절에서 본 Cassandra의 강점 목록과 같다. 결제 시스템이 event store로 Cassandra를 택하는 것은 우연이 아니라, **워크로드의 요구와 엔진의 강점이 정확히 맞아떨어진** 필연에 가까운 결과이다.

---

## 9. 한계와 운영 주의 사항: 공짜 점심은 없다

Event Sourcing on Cassandra는 강력하지만 만능은 아니다. 이 패턴에 따르는 비용을 있는 그대로 살펴보자.

### 9.1 replay 비용

엔티티의 이벤트가 많아질수록 replay는 느려진다. snapshot으로 완화할 수 있지만(3.3), 그 대신 snapshot 주기, snapshot 직렬화 비용, snapshot 저장소 관리라는 새로운 부담이 생긴다. 수명이 매우 길고 이벤트가 폭발적으로 쌓이는 엔티티는 Event Sourcing과 가장 맞지 않는다. 이런 경우에는 도메인을 다시 나눠(엔티티를 더 잘게 쪼개서) 한 엔티티의 이벤트 수가 제한되도록(bounded) 만드는 방안을 고려해야 한다.

### 9.2 이벤트 스키마 진화의 영구적 부담

4절에서 본 backward compatibility는 **영원히 유지해야 하는 제약**이다. 한번 잘못 설계한 이벤트는 계속 떠안고 가야 한다. 필드의 의미를 잘못 정하면, 과거 이벤트를 해석하기 위한 보정 로직(upcasting, 이벤트 어댑터)을 코드에 영구히 남겨야 한다. 그러므로 이벤트는 "지금 편한 것"이 아니라 "10년 뒤에도 읽을 수 있는 것"을 기준으로 설계해야 한다. 신중하게 설계하는 비용을 처음부터 치러야 하는 셈이다.

### 9.3 대용량 파티션 관리

partition_nr(3.2)과 tag 시간 버킷(5.1)으로 large partition을 막을 수 있지만, 이 파라미터를 잘못 잡으면 12장(운영과 트러블슈팅)에서 본 large partition 장애가 그대로 재현된다. `nodetool tablehistograms`로 파티션 크기를 주기적으로 모니터링하고, target-partition-size와 tag 버킷 크기를 워크로드에 맞게 조정해야 한다. 한 번 설정하고 잊어도 되는 값이 아니다.

### 9.4 projection의 eventual consistency와 멱등 consumer

5절에서 본 대로 read model은 뒤처질 수 있고, 이벤트는 중복 전달될 수 있다. UI나 API가 방금 쓴 내용이 projection에 즉시 보일 것이라고 가정하면 버그가 생긴다. read-your-writes는 projection에 기대하지 말고, 엔티티를 직접 조회해서 해결해야 한다.

consumer는 항상 멱등해야 하며, projection이 깨지면 이벤트를 처음부터 다시 읽어 read model을 재구축(rebuild)할 수 있어야 한다. 이 rebuild 능력은 Event Sourcing의 안전장치이면서, 동시에 운영 부담이기도 하다.

### 9.5 디버깅과 사고방식의 전환

상태 모델에 익숙한 팀에게는 "현재 상태가 테이블에 없고, 이벤트를 fold해야 보인다"는 사고방식의 전환이 진입 장벽이 된다. 지금 이 결제 상태가 왜 이렇게 되었는지 보려면 이벤트 로그를 읽고 머릿속으로(또는 도구로) replay해야 한다. 그래서 사람이 이벤트 로그를 디코딩해 열람하고, replay 결과를 재구성해 보여 주는 운영 도구가 사실상 필수이다. 이런 도구 없이 Event Sourcing을 운영하는 것은 계기판 없이 비행하는 것과 같다.

```text
   Event Sourcing on Cassandra — 트레이드오프 한눈에

  얻는 것                          치르는 것
  -----------------------          -----------------------
  완전한 이력/감사                 replay 비용 (-> snapshot)
  시점 복원/새 read model 자유      이벤트 스키마 영구 호환 부담
  append = 엔진 친화 (LSM)          large partition 관리 필요
  엔티티 단위 동시성 안전           projection eventual consistency
  무한 수평 확장                    멱등 consumer 필수
  tombstone 압박 낮음              상태 모델보다 높은 정신적 복잡도
```
