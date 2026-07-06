---
title: 실전 - Event Sourcing on Cassandra (Akka Persistence)
date: 2026-06-26
tags: [cassandra, event-sourcing, akka-persistence, 학습노트]
---

지금까지 Cassandra를 "데이터를 어떻게 모델링하고, 어떻게 쓰고 읽고, 내부에서 무슨 일이 벌어지는가"라는 관점에서 따라왔다. [[08 - Storage Engine 내부]]에서 본 LSM-tree, [[09 - Compaction 전략]]에서 본 compaction과 tombstone, [[10 - Write Path와 Read Path]]에서 본 쓰기·읽기 경로, [[05 - 데이터 모델링 1 - Query First]]에서 본 쿼리 우선 설계 — 이 모든 조각이 한 장면에서 동시에 맞물리는 실전 사례가 바로 **Event Sourcing**이다.

이 장이 답하려는 질문은 다음과 같다.

- Event Sourcing이란 정확히 무엇이고, "상태를 저장한다"는 RDB의 직관과 어떻게 근본적으로 다른가?
- 왜 하필 Cassandra가 event store로 잘 맞는가? 그것이 그저 "빠르다"는 마케팅이 아니라, LSM-tree와 append-only 쓰기 패턴의 구조적 궁합이라는 것을 storage engine 수준에서 어떻게 설명할 수 있는가?
- Akka Persistence Cassandra 플러그인은 journal과 snapshot을 어떤 스키마로 저장하고, 하나의 actor가 평생 쌓는 긴 이벤트 로그를 어떻게 large partition([[05 - 데이터 모델링 1 - Query First]], [[12 - 운영과 트러블슈팅]]의 그 함정) 없이 다루는가?
- append-only라는 제약이 직렬화·스키마 진화에 어떤 규율을 강제하는가? 이것이 event sourcing을 채택한 어떤 시스템에서든 똑같이 나타나는 이벤트 호환성 규칙과 왜 같은 뿌리에서 나오는가?
- 이벤트를 "쓰는 것"만큼이나 "어떻게 다시 조회하고 read model을 만드는가"(events-by-tag, CQRS)가 왜 어렵고, 거기서 어떤 eventual consistency 함정이 도사리는가?
- 이벤트는 거의 삭제하지 않는다 — 그러면 tombstone이 거의 없다 — 그러면 compaction 전략 선택([[09 - Compaction 전략]])이 다른 워크로드와 어떻게 달라지는가?

마지막으로, actor 기반 결제 시스템이 왜 "command → event → persist → state transition → replay" 구조를 채택하는지를 **개념 수준에서만** 연결한다. 구체적 구현 스키마나 API 디테일은 다루지 않는다. 일반 패턴과 "왜 이렇게 설계하는가"가 목표다.

---

## 1. Event Sourcing: 상태가 아니라 사실의 역사를 저장한다

### 잔액이 아니라 입출금 내역을 저장하라

가장 직관적인 비유는 은행 통장이다. RDB 사고방식으로 계좌를 설계하면 보통 이렇게 한다.

```text
accounts
+------------+---------+
| account_id | balance |
+------------+---------+
| acc-1      |  17,000 |   <- 현재 잔액만 들고 있다
+------------+---------+
```

입금/출금이 일어날 때마다 `UPDATE accounts SET balance = ? WHERE account_id = ?`로 **덮어쓴다**. 이 모델에서는 현재 상태(current state)만 남고 그 상태에 이르게 된 과정은 사라진다. 잔액이 17,000원이라는 사실은 알지만, 그것이 (입금 20,000 → 출금 3,000)의 결과인지 (입금 50,000 → 출금 33,000)의 결과인지는 통장 테이블만 봐서는 알 수 없다. 과정을 알고 싶으면 별도의 거래 로그 테이블을 따로 만들어야 한다.

Event Sourcing은 이 우선순위를 **뒤집는다**. 일차 데이터(source of truth)는 잔액이 아니라 거래 내역, 즉 **일어난 사실들의 순서 있는 시퀀스(an ordered sequence of facts)** 다.

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

여기서 "현재 잔액 17,000원"은 **저장된 데이터가 아니라, 이벤트들을 처음부터 접어(fold) 다시 계산한 파생값(derived value)** 이다. 함수형 언어로 표현하면 다음과 같다.

```text
currentState = events.foldLeft(initialState)(applyEvent)
```

이 한 줄에 Event Sourcing의 모든 철학이 들어 있다. 상태는 일급 시민이 아니다. 상태는 이벤트의 함수일 뿐이다.

### 세 가지 불변 규칙

Event Sourcing을 구성하는 규칙은 단출하지만 강력하다.

1. **이벤트는 사실이다(facts).** 이벤트는 과거형으로 명명된다 — `OrderPlaced`, `PaymentApproved`, `MoneyWithdrawn`. 이것은 의도(command)가 아니라 이미 일어난 일이다. 이름이 과거형이라는 것은 단순한 컨벤션이 아니라, "이미 벌어진 일은 취소할 수 없다"는 의미론적 약속이다.
2. **이벤트 저장소는 append-only다.** 이벤트를 한 번 기록하면 절대 수정하거나 삭제하지 않는다. 잘못된 일이 일어났다면 그것을 지우는 게 아니라, 그것을 **보정하는 새 이벤트**(예: `PaymentRefunded`)를 덧붙인다. 회계 장부에서 잘못 적은 줄을 지우개로 지우지 않고 빨간 줄로 정정 분개를 추가하는 것과 똑같다.
3. **현재 상태는 replay로 복원한다.** 시스템이 재시작되거나, 새 노드가 합류하거나, 메모리에서 actor가 passivate 되었다가 다시 깨어날 때, 우리는 저장된 상태를 로드하는 게 아니라 **이벤트를 처음부터 다시 적용(replay)** 해서 상태를 재구성한다.

> **흔한 오해**: "Event Sourcing은 그냥 audit log를 잘 남기는 것 아닌가?" 아니다. audit log는 보통 **부수적 기록**이고, 진짜 상태는 다른 곳(테이블의 current row)에 있다. Event Sourcing에서는 이벤트 로그가 **유일한 진실(the single source of truth)** 이고, 다른 모든 것(현재 상태, read model, 검색 인덱스)은 그 로그에서 파생된 캐시일 뿐이다. 이 차이가 운영·복구·디버깅의 모든 것을 바꾼다.

### 왜 이렇게까지 하는가 — 결제 시스템의 관점에서

상태 덮어쓰기 모델의 가장 큰 문제는 **정보 손실(lossy)** 이다. `balance`를 덮어쓰는 순간 직전 잔액과 그 변화의 맥락이 사라진다. 결제 시스템에서 이것은 치명적이다.

- "이 결제가 왜 취소 상태가 됐지?"라는 질문에 답하려면, 상태 모델에서는 별도의 로그를 뒤져야 하고 그 로그가 코드 변경 과정에서 누락됐을 수도 있다. Event Sourcing에서는 **상태 자체가 이벤트의 역사이므로** 답이 항상 거기 있다.
- "3개월 전 그 시점에 이 결제는 어떤 상태였나?"라는 시점 복원(temporal query)이 자연스럽다. 그 시점까지의 이벤트만 fold하면 된다. 상태 모델에서는 불가능하거나, 별도 스냅샷 인프라가 필요하다.
- 새로운 비즈니스 질문("환불까지 평균 며칠 걸리나?")이 생겼을 때, 이미 저장된 과거 이벤트를 새 관점으로 다시 fold해 새 read model을 만들 수 있다. 상태 모델은 과거를 버렸으므로 미래의 데이터부터만 답할 수 있다.

요컨대 Event Sourcing은 **"과거를 버리지 않는 대가로 미래의 모든 질문에 답할 권리를 산다"**. 그 대가가 바로 이 장의 후반부에서 다룰 replay 비용·스키마 진화·projection 복잡도다.

---

## 2. 왜 Cassandra가 event store에 잘 맞는가

Event Sourcing의 저장 계층은 사실상 **로그 데이터베이스**다. 이 워크로드의 특성은 놀랄 만큼 또렷하다.

- **쓰기가 거의 전부 append다.** 새 이벤트를 시퀀스 끝에 덧붙이는 것 외의 쓰기가 없다. UPDATE도, DELETE도 (거의) 없다.
- **읽기는 두 가지뿐이다.** (a) 특정 엔티티의 이벤트를 `seq_nr` 순서대로 처음부터(또는 스냅샷 이후부터) 끝까지 읽는 replay, (b) 태그/시간 기준으로 이벤트를 훑는 projection.
- **자연스러운 파티션 키가 이미 존재한다.** 엔티티 식별자(Akka에서는 `persistence_id`). 한 엔티티의 모든 이벤트는 항상 그 식별자로 함께 모인다.
- **무한히 커진다.** 시간이 갈수록 이벤트는 단조 증가만 한다. 수평 확장이 선택이 아니라 필수다.

이 네 가지를 Cassandra의 설계 철학과 나란히 놓아 보면, 둘이 같은 청사진에서 나온 것처럼 보일 정도다.

### 2.1 LSM-tree와 append-only 쓰기의 구조적 궁합

[[08 - Storage Engine 내부]]에서 본 것을 다시 떠올려 보자. Cassandra의 write path는 다음과 같다.

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

**Cassandra의 쓰기는 근본적으로 append-only다.** Commit log는 순차 append, memtable은 메모리상 정렬 삽입, flush 결과인 SSTable은 **immutable**이다. UPDATE도 내부적으로는 "새 버전을 덧쓰는 것"이고, DELETE조차 [[09 - Compaction 전략]]에서 본 tombstone(삭제를 표시하는 새 쓰기)으로 처리된다. Cassandra는 어떤 쓰기든 **기존 디스크 자리를 in-place로 고치지 않는다.**

이제 Event Sourcing의 워크로드를 겹쳐 보자. Event Sourcing도 본질이 append-only다. **애플리케이션의 쓰기 패턴(append)과 스토리지 엔진의 쓰기 메커니즘(append)이 한 치의 임피던스 불일치도 없이 정렬된다.**

이 맞물림이 왜 중요한지는 RDB와 대비해 보면 드러난다. 전통적 RDB의 B-tree는 **in-place update**를 위해 설계되었다. 페이지를 찾아가서 그 자리를 고치고, 인덱스를 갱신한다. 이는 무작위 I/O(random I/O)를 유발하고, 페이지 분할(page split), 잠금 경합을 만든다. 그런데 Event Sourcing은 애초에 in-place update를 **하지 않는** 워크로드다. B-tree가 비싸게 제공하는 "임의 위치 수정" 능력을 전혀 쓰지 않으면서, 그 비용(random I/O, 잠금)만 떠안는 셈이다.

반대로 LSM-tree는 "수정은 안 하고 계속 덧붙이며, 정리는 나중에 한꺼번에(compaction)"라는 철학이다. 이는 순차 I/O(sequential I/O) 위주로 동작하고, 쓰기 처리량이 높으며, write amplification을 compaction 시점으로 미룬다. Event Sourcing의 append 폭주를 받아내기에 이보다 자연스러운 구조는 없다.

```text
B-tree (RDB)              vs        LSM-tree (Cassandra)
-----------------                   --------------------
쓰기 = 제자리 수정                  쓰기 = 끝에 덧붙임
random I/O 위주                     sequential I/O 위주
페이지 분할/잠금 경합               memtable 정렬 삽입, 락 최소
읽기 빠름(인덱스 1회 탐색)          읽기는 여러 SSTable 병합 필요
ES와의 궁합: 비용만 부담            ES와의 궁합: 쓰기 패턴 = 엔진 패턴
```

> **주의**: "LSM이면 무조건 빠르다"가 아니다. LSM의 대가는 읽기 측에 있다 — 하나의 row를 읽으려면 여러 SSTable에 흩어진 조각을 병합해야 할 수 있다([[10 - Write Path와 Read Path]]). 다만 Event Sourcing의 읽기는 "한 파티션을 seq_nr 순서로 쭉 스캔"이라 LSM의 약점(무작위 point read)을 덜 건드린다. 이 균형이 뒤에서 compaction 전략 선택과 snapshot 설계로 이어진다.

### 2.2 파티션 = persistence id: 데이터 모델이 거저 주어진다

[[05 - 데이터 모델링 1 - Query First]]의 핵심 교훈은 "쿼리를 먼저 정하고, 그 쿼리가 단일 파티션에서 해결되도록 파티션 키를 설계하라"였다. Event Sourcing은 이 작업을 거의 공짜로 해 준다.

replay의 쿼리는 언제나 "이 엔티티의 이벤트를 seq_nr 순서대로 다 가져와"다. 따라서 파티션 키를 **엔티티 식별자(persistence_id)** 로 잡고, clustering key를 **seq_nr**로 잡으면, replay는 정확히 하나의 파티션을 정렬된 순서로 순차 읽기 하는 것이 된다. 이것은 Cassandra가 가장 잘하는 읽기 패턴, 곧 single-partition, clustering-order range scan이다.

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

RDB였다면 events 테이블에 `(entity_id, seq_nr)` 복합 인덱스를 만들고, 그 인덱스를 타고 range scan을 했을 것이다. 잘 동작하지만 데이터가 커지면 단일 머신의 한계에 부딪힌다. Cassandra는 **파티션 단위로 데이터를 토큰 링 위에 흩뿌리므로**([[02 - 분산 아키텍처]]) 엔티티가 수억 개로 늘어나도 각 엔티티의 replay는 여전히 단일 파티션·단일 노드(+복제본) 작업으로 격리된다. 엔티티 수가 늘어도 개별 replay 비용은 늘지 않고 전체 처리량은 노드를 추가하면 선형적으로 늘어난다. 이것이 "수평 확장"의 실체다.

### 2.3 시계열적 append와 토큰 링

엔티티 식별자를 파티션 키로 쓰면, 서로 다른 엔티티의 이벤트는 [[02 - 분산 아키텍처]]에서 본 것처럼 partitioner(기본 Murmur3Partitioner)에 의해 토큰 링 전체에 고르게 분산된다.

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

이 분산은 두 가지를 동시에 준다. (1) 쓰기 부하가 한 노드에 몰리지 않는다(no hotspot) — 수많은 엔티티가 동시에 이벤트를 append 해도 각자 다른 파티션·다른 노드로 흩어진다. (2) 한 엔티티 내부에서는 seq_nr 순서로 시계열 append가 보장된다.

여기서 자주 나오는 안티패턴을 짚자. "전역 이벤트 스트림을 하나의 파티션에 다 넣으면 순서가 완벽하게 보장되지 않을까?" — 그렇게 하면 **모든 쓰기가 단 하나의 파티션, 단 하나의 노드(+복제본)로 몰린다.** 이것이 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]에서 경고한 hotspot이자 unbounded partition이다. Event Sourcing이 이를 피하는 방식은 "순서는 엔티티 단위로만 보장하고, 전역 순서가 필요하면 별도의 tag/시간 버킷 테이블로 분리한다"(2.4의 events-by-tag)이다.

### 2.4 높은 write throughput과 quorum 쓰기

마지막으로, event store는 보통 read보다 write가 많거나 최소한 write가 매우 꾸준한 워크로드다(모든 비즈니스 사건이 이벤트가 되므로). Cassandra는 [[10 - Write Path와 Read Path]]에서 본 대로 쓰기를 commit log append + memtable 삽입만으로 끝내고 ack 하므로(나머지는 비동기 flush/compaction) 쓰기 지연이 낮고 처리량이 높다. 또한 [[04 - Tunable Consistency]]의 tunable consistency 덕분에 event store는 보통 다음 조합을 쓴다.

- 이벤트 append: `LOCAL_QUORUM` 쓰기 — 데이터센터 내 과반 복제본에 기록되어야 ack. 이벤트 유실 위험을 낮춘다.
- replay 읽기: 보통 `LOCAL_QUORUM` 읽기로, `R + W > RF`를 만족시켜 방금 쓴 이벤트를 확실히 다시 읽도록 한다(read-your-writes).

결제처럼 "쓴 이벤트가 사라지면 안 되고, 재시작 후 replay에서 그 이벤트가 반드시 보여야 하는" 도메인에서는 이 일관성 조합이 사실상 필수다. (Akka Persistence Cassandra 플러그인은 journal write/read consistency를 설정으로 노출하며, 안전을 위해 보통 quorum 계열로 둔다.)

---

## 3. Akka Persistence Cassandra: journal과 snapshot의 내부 스키마

이제 추상에서 구체로 내려가자. Akka Persistence는 actor의 영속화 추상이고, 그 백엔드 중 하나가 **akka-persistence-cassandra**(현재는 Pekko 진영의 pekko-persistence-cassandra로도 이어진다) 플러그인이다. 이 플러그인은 두 개의 저장소를 제공한다.

- **journal**: 이벤트의 append-only 로그. event store 그 자체.
- **snapshot store**: 특정 시점의 상태를 통째로 저장한 체크포인트. replay 비용을 줄이는 최적화 장치.

> 아래 스키마는 개념을 설명하기 위한 **단순화된 형태**다. 실제 플러그인의 컬럼명·타입·버전별 세부는 다를 수 있으니, 정확한 DDL은 사용 중인 플러그인 버전의 문서를 확인해야 한다. 여기서는 "왜 이런 컬럼이 필요한가"의 의도를 읽는 게 목적이다.

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

각 컬럼이 존재하는 이유를 하나씩 뜯어 보자.

- **`persistence_id`**: 엔티티(=actor)의 고유 식별자. "결제 9f3a의 모든 이벤트는 여기 모인다"는 단위. 파티션 키의 일부.
- **`partition_nr`**: 이것이 이 스키마에서 가장 중요한 설계 결정이다. 자세히는 3.2에서 다룬다. 파티션 키의 또 다른 일부.
- **`sequence_nr`**: 엔티티 내부에서 1부터 단조 증가하는 이벤트 순번. clustering key이므로 **저장 자체가 seq 순서로 정렬**된다. replay는 이 순서대로 읽으면 된다. 또한 이 번호가 **멱등성과 순서 보장의 열쇠**다(3.4, 7장 연결).
- **`event` (blob) + `ser_id` + `ser_manifest`**: 이벤트 본문은 도메인 타입을 직렬화한 바이트 덩어리(blob)다. Cassandra는 이 바이트의 의미를 모른다 — 단지 보관할 뿐이다. 역직렬화할 때 어떤 직렬화기로, 어떤 타입/버전으로 풀어야 하는지를 `ser_id`와 `ser_manifest`가 알려준다. 이 두 컬럼이 4장에서 다룰 **스키마 진화**의 토대다.
- **`tags`**: 이 이벤트가 어떤 projection 스트림에 속하는지를 나타내는 라벨. "이 이벤트는 `payment` 태그와 `vbank` 태그에 속한다" 같은 식. events-by-tag 조회의 입력(2.4, 5장).
- **`timestamp` (timeuuid)**: 쓰기 시각. tag 스트림의 시간 정렬, 디버깅, projection 오프셋 추적에 쓰인다.

여기서 PRIMARY KEY `((persistence_id, partition_nr), sequence_nr)`의 구조가 중요하다. 파티션 키가 `(persistence_id, partition_nr)` 복합이고 그 안에서 `sequence_nr`로 정렬된다. **한 엔티티의 이벤트 로그가 partition_nr 단위로 여러 파티션에 쪼개져 저장되며, 각 파티션 안에서는 seq 순서가 보장된다.**

### 3.2 partition_nr: large partition을 피하는 결정적 장치

[[05 - 데이터 모델링 1 - Query First]]와 [[12 - 운영과 트러블슈팅]]에서 반복적으로 경고한 함정이 **large partition / unbounded partition**이다. 한 파티션이 무한정 커지면 다음 문제가 터진다.

- 파티션 전체를 읽거나 compaction 할 때 메모리·GC 압박. (Cassandra는 파티션을 처리 단위로 다룬다.)
- 단일 파티션은 단일 노드(+복제본)에 산다 — 한 파티션이 거대해지면 그 노드만 비대해지는 불균형.
- 파티션 내 row가 수십만~수백만이면 read 시 partition index 탐색과 SSTable 병합 비용이 커진다.
- 운영 경고선: 이상적으로는 파티션당 100MB·10만 row 이내를 목표로 보고, 수백 MB·수백만 row를 넘기 시작하면 GC 압박·읽기 지연·복구(스트리밍) 비용이 급격히 나빠진다.

그런데 Event Sourcing의 엔티티는 **수명이 길수록 이벤트가 무한히 쌓인다.** 활성 사용자 한 명, 오래된 계좌 하나가 수년간 수십만 개의 이벤트를 만들 수 있다. persistence_id만 파티션 키로 쓰면 이 엔티티의 파티션이 unbounded로 자라 위의 모든 문제를 정통으로 맞는다.

`partition_nr`은 바로 이 문제의 해법이다. 플러그인은 설정값(예: `target-partition-size`, 기본 대략 수십만 이벤트 단위)에 따라, seq_nr이 일정 개수를 넘어가면 partition_nr을 증가시켜 **같은 엔티티의 로그를 여러 파티션으로 분할(bucketing)** 한다.

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

이렇게 하면 무한히 긴 논리적 로그가 **bounded한 물리적 파티션들의 연속**으로 바뀐다. partition_nr은 seq_nr로부터 산술적으로 계산 가능하므로(예: `partition_nr = (seq_nr - 1) / target_size`), replay 시 어느 파티션들을 어떤 순서로 읽어야 하는지 코드가 결정적으로 안다. **순차적으로 partition_nr=0, 1, 2... 를 차례로 range scan**하면 전체 로그를 seq 순서로 복원할 수 있다.

이것이 5장의 "쿼리 우선 설계"와 6·12장의 "large partition 회피"가 실전에서 어떻게 구현되는지를 보여주는 교과서적 사례다. 무한히 자라는 시계열을 시간/개수 버킷으로 쪼개는 이 기법은 시계열 데이터 모델링의 정석이며, partition_nr은 그 버킷 번호다.

> **함정**: target-partition-size를 너무 작게 잡으면 한 엔티티의 replay가 너무 많은 파티션(=너무 많은 별개의 쿼리/디스크 탐색)으로 쪼개져 오히려 replay가 느려진다. 너무 크게 잡으면 large partition 위험. 일반적으로 "한 엔티티가 평생 만들 이벤트 수"와 "snapshot 주기"를 함께 고려해 정한다. snapshot이 자주 찍히면 replay는 마지막 snapshot 이후 파티션 몇 개만 읽으면 되므로, partition 분할 개수보다 snapshot 전략이 실효 replay 비용을 더 좌우한다(3.3).

### 3.3 snapshot store: replay 비용을 잘라내는 체크포인트

엔티티가 수십만 개의 이벤트를 쌓았다면, 재시작할 때마다 그 전부를 처음부터 fold 하는 것은 비싸다(N개 이벤트면 N번의 적용). snapshot은 이 비용을 자르는 장치다.

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

snapshot은 **이벤트가 아니라 상태**를 저장한다는 점에 주목하라. "seq 480,000 시점에서 이 계좌의 전체 상태는 이러했다"를 통째로 직렬화해 둔 것이다. clustering order가 `DESC`인 이유는 "가장 최신 snapshot을 맨 앞에서 1건만 빠르게 읽기" 위해서다.

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

여기서 Event Sourcing의 미묘하지만 결정적인 설계 원칙이 드러난다. **snapshot은 최적화일 뿐, 진실의 원천이 아니다.** 진실은 언제나 journal(이벤트 로그)이다. snapshot이 손상되거나 삭제되어도 시스템은 journal을 처음부터 replay 해서 동일한 상태를 복원할 수 있다. 그래서 snapshot은 마음 편히 삭제할 수 있는 "버려도 되는 캐시"다. 이 비대칭(journal은 신성불가침, snapshot은 폐기 가능)이 4장의 스키마 진화 규칙에도 영향을 준다 — snapshot 포맷은 비교적 자유롭게 바꿀 수 있지만(필요하면 버리고 다시 만들면 되니까), journal 이벤트 포맷은 영원히 호환되어야 한다.

> **운영 팁**: snapshot 주기를 "이벤트 N건마다" 또는 "특정 상태 전이 시"로 잡는다. 결제 actor라면 "결제 완료/취소 같은 안정적 상태에 도달했을 때 snapshot"이 합리적이다. snapshot이 잦으면 replay는 빨라지지만 snapshot 쓰기·저장 비용이 늘고, 드물면 그 반대다. 또한 오래된 snapshot은 정리(`deleteSnapshots`)해 snapshot 테이블이 무한히 커지는 것을 막는다. journal은 보통 안 지우지만 snapshot은 지워도 안전하다는 점을 기억하라.

---

## 4. 직렬화와 이벤트 스키마 진화: append-only가 강제하는 규율

이제 이 장에서 가장 실무적으로 아픈 주제로 들어간다. journal에 저장된 `event` 컬럼은 그냥 blob이다. Cassandra는 그 내용을 모른다. 의미를 부여하는 것은 전적으로 애플리케이션의 직렬화 계층이다. 여기서 Event Sourcing의 가장 무거운 제약이 나온다.

### 4.1 replay라는 제약이 모든 것을 규정한다

한 가지 사실을 다시 못 박자. **이벤트는 append-only이고 현재 상태는 그 이벤트들을 replay해서 만든다.** 이 두 문장을 합치면 다음이 따라온다.

> **3년 전에 저장된 이벤트도, 오늘 배포된 최신 코드가 역직렬화하고 해석할 수 있어야 한다.**

상태 덮어쓰기 모델에서는 스키마를 바꾸면 마이그레이션을 한 번 돌려 모든 row를 새 포맷으로 갱신하면 끝이다. 과거는 사라졌으니 과거 포맷을 영원히 지원할 필요가 없다. 하지만 Event Sourcing에서는 **과거의 모든 이벤트가 영원히 살아 있고 재시작할 때마다 다시 읽힌다.** 따라서 이벤트 스키마는 한번 세상에 나가면 **영원히 하위 호환(backward compatible)** 이어야 한다.

이것이 직렬화기 선택을 좌우한다. 스키마 진화에 강한 포맷이 선호된다.

- **Protobuf / Avro / Thrift**: 필드에 번호 태그를 붙이고, 알 수 없는 필드는 무시하며, optional 필드를 기본값으로 채운다 — 스키마 진화에 강하다. 이벤트 직렬화에 널리 쓰인다.
- **JSON**: 사람이 읽기 좋고 유연하지만 타입 안정성이 약하고 크기가 크다. 작은 시스템에선 쓰이나 대규모 event store에선 크기/성능이 부담.
- **Java 기본 직렬화**: **금기.** 클래스 구조가 바뀌면 과거 바이트를 못 읽는다. Event Sourcing에서 절대 쓰지 말 것.

### 4.2 호환성 규칙 — 무엇이 허용되고 무엇이 금지되는가

Protobuf를 예로 들면, 이벤트 진화에서 허용/금지 규칙은 다음과 같다.

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

마지막 항목인 **의미 변경**이 가장 음흉하다. 컴파일은 되고 역직렬화도 되지만, 같은 필드가 옛 이벤트와 새 이벤트에서 다른 뜻을 가지면, replay가 만들어내는 상태가 조용히 틀어진다. 이것은 컴파일러도 테스트도 잡기 어렵다. 그래서 규율은 **"이벤트는 append-only로 취급하라 — 필드도, 의미도 append만 하라"** 로 요약된다.

### 4.3 이 규율은 event store와 무관하게 따라온다

여기서 기억할 점이 있다. 이 backward compatibility는 특정 플러그인이나 특정 팀의 컨벤션이 아니다. **Akka Persistence Cassandra가 아니라 어떤 event store를 쓰든, Event Sourcing을 채택하는 순간 이 규율은 물리 법칙처럼 따라온다.** 이유의 사슬은 늘 같다 — 상태를 이벤트 replay로 복원하므로(상태 = 이벤트의 replay), 과거 이벤트가 영원히 다시 읽히고, 따라서 이벤트 스키마는 영원히 하위 호환이어야 하며, 필드 제거·타입 변경·번호 재사용·의미 변경이 모두 금지된다(4.1, 4.2). Protobuf든 Avro든, event sourcing을 쓰는 어떤 도메인 이벤트 정의도 같은 제약 아래 놓인다. "이벤트 replay가 상태를 재구성하므로, 깨지는 변경은 곧 과거의 붕괴"라는 한 문장이 그 모든 규칙의 뿌리다.

[[08 - Storage Engine 내부]]의 SSTable이 immutable이듯, Event Sourcing의 이벤트도 immutable이다. 한쪽은 스토리지 엔진 레벨의 불변성이고 다른 한쪽은 도메인 레벨의 불변성이지만, "한번 쓴 것은 고치지 않고 새로 덧붙인다"는 동일한 정신을 공유한다. 이 일관된 철학이 Cassandra + Event Sourcing 조합을 개념적으로도 아름답게 만든다.

---

## 5. 이벤트 조회와 projection: events-by-tag와 CQRS

지금까지는 "한 엔티티의 이벤트를 그 엔티티로 다시 읽는" 경로(replay)만 봤다. 그런데 실전에선 **엔티티를 가로지르는 조회**가 반드시 필요하다. "오늘 발생한 모든 결제 완료 이벤트", "취소된 모든 주문" 같은 질문이다. 이 질문은 persistence_id로는 답할 수 없다 — 여러 엔티티에 걸쳐 있으니까.

### 5.1 왜 별도 tag 테이블이 필요한가

[[05 - 데이터 모델링 1 - Query First]]의 철칙을 떠올리자. "다른 키로 조회하려면 다른 테이블이 필요하다." Cassandra에는 임의 컬럼 필터링이 없고(있어도 legacy secondary index는 분산 환경에서 scatter-gather라 비싸고 위험하다. Cassandra 5.0의 SAI(Storage-Attached Index)가 이 비용을 크게 개선했지만, 시간순 무한 스트리밍 + offset 재개가 필요한 projection 용도에는 인덱스보다 전용 tag 테이블 설계가 여전히 맞다), 조회 패턴마다 그에 맞는 비정규화 테이블을 따로 둔다.

그래서 Akka Persistence Cassandra는 journal 외에 **events-by-tag**용 별도 테이블(보통 `tag_views` 류)을 유지한다. 이벤트에 `tags`를 달면, 플러그인이 그 이벤트를 태그별 테이블에도 기록한다. 이 테이블의 파티션 키는 대략 `(tag, 시간 버킷)`이고, clustering은 시간 순서(timeuuid)다.

```text
journal (엔티티 기준)              tag_views (태그 기준, projection 용)
파티션키: (persistence_id, part)   파티션키: (tag, time_bucket)
정렬: sequence_nr                  정렬: timestamp(timeuuid)

  payment-A: e1, e2, e3            tag="payment", bucket=2026-06-26-10:
  payment-B: e1, e2                  [payment-A:e2, payment-B:e1, payment-A:e3, ...]
                                       ^ 여러 엔티티의 이벤트가 시간순으로 섞여 한 스트림
```

여기서 또 시계열 버킷팅이 등장한다(2.3, 3.2와 같은 기법). 태그 스트림은 전 엔티티의 이벤트가 모이므로 가만두면 거대 파티션이 된다. 그래서 시간 버킷(예: 시간 단위, 분 단위)으로 파티션을 나눈다. read journal은 "이 태그의 이 시각 이후 이벤트"를 버킷을 따라가며 스트리밍한다.

### 5.2 eventual consistency라는 본질적 함정

여기에 반드시 이해해야 할 함정이 있다. **journal 쓰기와 tag_views 쓰기는 하나의 원자적 트랜잭션이 아니다.** 이벤트는 먼저 엔티티의 journal에 기록되고, tag_views로의 전파는 별도의 tag writer가 비동기적으로(설정 가능한 지연, 예: 플러그인의 `events-by-tag.eventual-consistency-delay` 만큼 묶어서) 처리한다. **events-by-tag 스트림은 eventually consistent**다.

이것이 뜻하는 바는 다음과 같다.

- 엔티티에 이벤트를 append하고 즉시 그 태그 스트림을 조회하면, 방금 그 이벤트가 아직 안 보일 수 있다(전파 지연).
- 따라서 projection을 "방금 일어난 일을 즉시 반영하는 강한 일관성 뷰"로 기대하면 안 된다. projection은 **약간 뒤처지는(lagging) read model**이다.
- read journal은 이를 다루기 위해 오프셋(offset, 보통 timeuuid 기반)을 추적하며 스트림을 재개할 수 있게 설계되어 있고, 소비자(projection)는 자신이 어디까지 처리했는지를 저장한다.

이 모든 것이 **CQRS(Command Query Responsibility Segregation)** 라는 큰 그림의 일부다.

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

쓰기 모델(이벤트, 정규화)과 읽기 모델(조회 최적화, 비정규화)을 분리하는 것이 CQRS다. Event Sourcing은 자연스럽게 CQRS로 이어진다 — 이벤트 로그가 쓰기 모델이고, 거기서 만든 모든 read model이 읽기 모델이다. read model은 [[05 - 데이터 모델링 1 - Query First]]의 "쿼리마다 테이블 하나" 원칙대로 조회 패턴별로 여러 개를 둘 수 있고, 각각은 같은 이벤트 스트림에서 독립적으로 만들어진다.

### 5.3 멱등 consumer: at-least-once 전달의 필연

projection consumer는 이벤트를 받아 read model을 갱신한다. 그런데 분산 시스템에서 메시지 전달은 보통 **at-least-once**다 — 장애·재시작·재구독 시 같은 이벤트가 두 번 이상 전달될 수 있다. 따라서 consumer는 반드시 **멱등(idempotent)** 해야 한다.

다행히 이벤트에는 `(persistence_id, sequence_nr)`라는 자연 키가 있다. consumer는 "이 persistence_id에서 내가 마지막으로 처리한 seq_nr"을 저장해 두고, 그보다 작거나 같은 seq_nr이 또 오면 무시하면 된다. 또는 read model 갱신 자체를 멱등 연산(upsert, set 연산)으로 설계한다.

```text
멱등 projection consumer

  받은 이벤트: (persistence_id=P, seq_nr=S)
  if S <= last_processed[P]:   이미 처리함 -> skip
  else:
     read model 갱신 (가능하면 upsert)
     last_processed[P] = S
```

이것은 [[11 - LWT Batch Counter 내부]]에서 본 멱등성·중복 처리 주제, 그리고 결제 시스템의 webhook 멱등 소비자(중복 webhook을 sequence/idempotency로 거르는) 패턴과 정확히 같은 사고방식이다. **분산 시스템에서 "정확히 한 번"은 거의 항상 "최소 한 번 + 멱등 소비자"로 구현된다.**

---

## 6. compaction과 tombstone 관점: event store는 다른 동물이다

[[09 - Compaction 전략]]에서 STCS / LCS / TWCS / UCS를 다뤘다. event store의 워크로드는 이 선택을 일반적인 경우와 다르게 만든다. 그 차이의 뿌리는 단 하나 — **이벤트는 거의 삭제하지 않는다.**

### 6.1 tombstone이 거의 없다는 사실의 무게

[[09 - Compaction 전략]]에서 본 대로, Cassandra의 DELETE는 tombstone(삭제 마커)을 쓰는 것이고, tombstone이 누적되면 read 시 스캔 비용이 커지고 `gc_grace_seconds`(기본 864000초 = 10일) 동안 살아 있다가 compaction으로 정리된다. tombstone 폭증은 [[12 - 운영과 트러블슈팅]]의 단골 장애 원인이다.

그런데 순수 event store(journal)에서는 **DELETE가 거의 없다.** 이벤트는 append만 하고 지우지 않는 것이 원칙이니까. 이 사실이 낳는 결과는 이렇다.

- **tombstone 압박이 본질적으로 낮다.** read path가 tombstone 더미를 스캔하느라 느려지는 일이 거의 없다. 이는 LSM-tree의 약점 중 하나(tombstone)를 워크로드 자체가 회피한다는 뜻이다.
- 따라서 event store의 compaction은 "tombstone을 빨리 치우기 위한" 것이 아니라, **읽기 효율(한 파티션의 조각들을 적은 수의 SSTable로 모으기)을 위한** 것이 주 목적이 된다.

### 6.2 그래서 어떤 compaction 전략인가

이것이 TWCS와의 결정적 차이를 만든다. [[09 - Compaction 전략]]에서 TWCS(TimeWindowCompactionStrategy)는 "시간 버킷별로 SSTable을 묶고, TTL로 통째 만료시켜 버킷 단위로 drop하는" 시계열·만료형 워크로드(로그, 메트릭)에 최적이라고 했다. event store도 시계열처럼 보이지만 **TTL로 만료시키지 않는다**는 점에서 TWCS의 전제를 벗어난다. 이벤트를 시간이 지나면 자동 삭제해 버리면 replay로 과거를 복원할 수 없게 되니, event store의 본질과 충돌한다.

그래서 일반적 선택은 다음과 같다.

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

선택은 갈린다. **읽기(replay) 지연 안정성을 중시하면 LCS**, **쓰기 처리량과 단순함을 중시하면 STCS** 쪽으로 기운다. 결제 actor처럼 "엔티티 단위 replay가 빈번하고 latency가 중요한" 경우 LCS의 read 효율이 매력적이지만, write-heavy하고 snapshot으로 replay 범위를 짧게 유지한다면 STCS의 낮은 쓰기 증폭이 더 나을 수 있다. 정답은 워크로드 측정에 달려 있다.

> **Cassandra 5.0 메모**: Cassandra 5.0에는 UCS(Unified Compaction Strategy)가 도입되어, 하나의 파라미터화된 전략으로 STCS와 LCS 스펙트럼 사이를 스케일 팩터로 조절할 수 있다([[09 - Compaction 전략]] 참조). event store에서는 UCS로 "쓰기 증폭과 읽기 증폭 사이의 트레이드오프"를 워크로드에 맞춰 튜닝하는 접근이 가능하다. 다만 어떤 전략을 쓰든, "이벤트는 만료시키지 않으므로 TTL 기반 drop에 의존하지 않는다"는 원칙은 동일하다. (UCS의 세부 기본 파라미터는 버전·배포에 따라 다를 수 있으므로 실제 값은 문서로 확인하라.)

snapshot 테이블은 이야기가 약간 다르다. snapshot은 오래된 것을 지우므로(3.3) tombstone이 생긴다. 다만 양이 journal에 비해 훨씬 적고 패턴이 단순해서 보통 기본 STCS로 충분하다.

### 6.3 snapshot은 compaction이 아니라 replay 비용을 줄인다

여기서 개념을 헷갈리지 말자. **compaction은 SSTable 정리(스토리지 레벨)** 이고, **snapshot은 replay 단축(애플리케이션 레벨)** 이다. 둘 다 "누적된 것을 정리해 비용을 낮춘다"는 점에서 닮았지만 층위가 다르다.

```text
   두 가지 "정리" 메커니즘 — 층위가 다르다

  애플리케이션 레벨   snapshot  : 이벤트 N개 replay 대신 상태 1개 + 그 이후만
  스토리지 레벨       compaction: 흩어진 SSTable 조각을 병합해 read 효율 회복
```

event store에서 read latency를 좌우하는 진짜 지렛대는 대개 **snapshot 주기**다. snapshot이 충분히 잦으면 replay가 짧아져, compaction 전략에 의한 read 효율 차이의 영향이 줄어든다. 그래서 실전 튜닝은 보통 "snapshot 주기 먼저, compaction 전략은 그다음"의 순서로 접근한다.

---

## 7. 일관성과 순서: sequence_nr가 떠받치는 정확성

Event Sourcing의 정확성은 두 가지 보장 위에 서 있다. **순서(order)** 와 **멱등(idempotency)**. 둘 다 `sequence_nr`이 떠받친다.

### 7.1 단일 파티션 append의 순서 보장

한 엔티티의 모든 이벤트는 같은 파티션(들)에 `sequence_nr` clustering order로 저장된다(3.1). Cassandra는 한 파티션 안에서 clustering key 순서로 데이터를 정렬·저장·반환하므로, **한 엔티티의 이벤트 순서는 물리적으로 보장**된다. replay는 항상 seq 1, 2, 3... 순으로 읽힌다.

다만 이 순서 보장은 **엔티티(파티션) 단위로만** 성립한다. 서로 다른 엔티티 간의 전역 순서는 Cassandra가 보장하지 않으며, 보장할 필요도 없다 — 도메인 상 서로 다른 엔티티의 사건은 독립적이기 때문이다. 전역 시간순이 필요하면 5장의 tag 스트림(timeuuid 기준)을 쓰되, 그것은 eventual consistency라는 약한 보장임을 받아들여야 한다.

> RDB였다면 전역 auto-increment id나 트랜잭션 격리로 전역 순서를 쉽게 줄 수 있다. 하지만 그 편리함은 단일 쓰기 지점(중앙 시퀀스, 단일 마스터)을 요구하고, 그것이 곧 확장의 병목이 된다. Cassandra/Event Sourcing은 "순서 보장을 엔티티 단위로 좁히는" 대가로 무한한 수평 확장을 얻는다. 이것은 손해가 아니라 **의도적인 설계 트레이드오프**다.

### 7.2 sequence_nr로 만드는 멱등성과 동시성 안전

이벤트를 append할 때, actor는 "다음 seq_nr은 N이어야 한다"를 안다(현재까지 적용한 마지막 seq + 1). 여기서 흔히 오해하는 지점이 있다 — "같은 `(persistence_id, partition_nr, seq_nr)`로 두 번 쓰면 Cassandra가 중복을 막아주지 않을까?" **아니다.** Cassandra의 일반 쓰기는 **upsert**라서, 같은 PRIMARY KEY에 대한 두 번째 쓰기는 충돌로 거부되는 게 아니라 셀 타임스탬프 기반 last-write-wins로 **조용히 덮어쓴다**([[10 - Write Path와 Read Path]]). 즉 DB는 seq_nr 중복을 스스로 감지하지 못한다(감지하려면 `IF NOT EXISTS` 같은 LWT가 필요한데, journal append는 기본적으로 LWT를 쓰지 않는다).

그래서 순번의 단조 증가와 무충돌은 DB가 아니라 **애플리케이션이 보장한다.** Akka Persistence는 단일 writer 가정(한 시점에 한 엔티티는 한 actor 인스턴스만 쓴다)을 두고, 그 actor가 메모리에 들고 있는 마지막 seq에 +1 해서 쓰므로 같은 seq_nr이 두 번 발급될 일 자체가 없다. 플러그인은 각 쓰기에 writer를 식별하는 메타데이터(writer UUID)를 함께 기록해, 이 가정이 깨지고 둘 이상이 같은 엔티티를 동시에 쓰는 비정상 상황을 사후에 탐지할 단서를 남긴다.

이 "단일 writer per entity"가 바로 다음 절에서 다룰 actor 모델의 동시성 안전성과 직결된다. 그리고 그 단일 writer 가정 덕분에, event store는 매 쓰기마다 [[11 - LWT Batch Counter 내부]]의 LWT(Paxos 기반 compare-and-set)를 쓸 필요가 없다. LWT는 비싸다(여러 라운드트립). 대신 "엔티티당 하나의 actor"라는 애플리케이션 레벨의 직렬화로 동시성 충돌을 원천 차단하고, 일반 quorum 쓰기로 빠르게 append한다. 이것이 actor 모델 + event sourcing 조합의 영리한 점이다 — **DB 레벨의 무거운 합의 대신 애플리케이션 레벨의 actor 직렬화로 정확성을 얻는다.**

---

## 8. 개념 연결: actor 기반 결제 시스템은 왜 이렇게 설계되는가

이제 모든 조각을 actor 기반 결제 시스템의 큰 그림에 **개념 수준에서만** 맞춰 보자. (구체적 구현·스키마·API는 다루지 않는다. 일반 패턴과 "왜"만 본다.)

### 8.1 command → event → persist → state transition

actor 기반 event-sourced 시스템에서 하나의 사건이 처리되는 보편적 흐름은 다음과 같다.

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

[3]과 [4]의 순서가 결정적이다. **이벤트를 먼저 영속화하고, 그 다음에 상태를 바꾼다.** 영속화에 실패하면 상태를 바꾸지 않는다. 이로써 "메모리 상태와 저장된 이벤트가 어긋나는" 사태를 막는다. 재시작하면 [4]의 메모리 상태는 사라지지만, [3]의 이벤트는 살아 있으니 replay로 정확히 같은 상태가 복원된다. 메모리 상태는 언제나 "journal의 함수"라는 불변식이 유지된다.

### 8.2 재시작 = replay로 상태 복원

actor가 메모리에서 내려갔다가(passivation, 장애, 배포) 다시 깨어날 때, 그는 자신의 persistence_id로 snapshot + 이후 이벤트를 읽어 상태를 재구성한다(3.3). 이것이 결제 시스템에 주는 운영적 안정감은 막대하다. 어떤 pod가 죽어도, 어떤 노드가 재시작해도, 결제 actor는 자신이 마지막으로 알던 정확한 상태로 되돌아온다 — 진실이 메모리가 아니라 Cassandra의 이벤트 로그에 있기 때문이다. 상태를 잃을까 봐 두려워할 필요가 없다.

### 8.3 단일 actor 직렬 처리 = 동시성/중복결제 방지

가장 결정적인 설계 이유가 이것이다. **하나의 결제(엔티티)는 하나의 actor가 책임지고, 그 actor는 자신에게 온 명령을 메일박스에서 순차적으로(한 번에 하나씩) 처리한다.** 동시에 같은 결제에 대한 두 명령이 와도, actor 메일박스에서 줄을 서서 차례로 처리된다.

```text
   동시 요청 두 개 — 같은 결제에 대해

  요청1: "승인"  --+
                   |   같은 persistence_id -> 같은 actor 메일박스
  요청2: "승인"  --+

  actor 처리 (순차):
    요청1 처리: 상태 확인(미결제) -> PaymentApproved 영속화 -> 상태=Paid
    요청2 처리: 상태 확인(이미 Paid!) -> 거부 (중복 결제 방지)
```

이것이 sleep·DB row lock 같은 잠금/대기 기반으로 경합을 막으려던 전통적 방식과 근본적으로 다른 점이다. 그런 방식은 비결정적이고 깨지기 쉬운 race 방어였다. actor + event sourcing은 **race 자체가 발생할 수 없는 구조**를 만든다 — 동시성을 "막는" 게 아니라 "직렬화로 제거"한다. 더블클릭, 브라우저 뒤로가기 재결제, webhook과 redirect의 경합 같은 결제 시스템의 고전적 골칫거리들이, "엔티티당 단일 actor 직렬 처리 + 영속화된 상태로 멱등 검증"이라는 한 가지 원리로 해소된다.

이 모든 것을 떠받치는 저장 계층의 요구사항을 정리하면 — "엔티티별로 분리된 파티션, 고속 append, 순서 보장, 영속성, 수평 확장" — 이것이 정확히 2장에서 본 Cassandra의 강점 목록이다. 결제 시스템이 event store로 Cassandra를 택하는 것은 우연이 아니라, **워크로드의 요구와 엔진의 강점이 한 점에서 만나는** 필연에 가깝다.

---

## 9. 한계와 운영 주의사항: 공짜 점심은 없다

Event Sourcing on Cassandra는 강력하지만 만병통치약이 아니다. 이 패턴이 청구하는 비용을 정직하게 본다.

### 9.1 replay 비용

엔티티의 이벤트가 많아질수록 replay가 느려진다. snapshot으로 완화하지만(3.3), snapshot 주기·snapshot 직렬화 비용·snapshot 저장소 관리라는 새 부담이 생긴다. "수명이 매우 길고 이벤트가 폭발적으로 쌓이는 엔티티"는 Event Sourcing의 천적이다. 이런 엔티티는 도메인을 다시 쪼개(엔티티를 더 잘게) 한 엔티티의 이벤트 수를 bounded하게 만드는 것을 고려해야 한다.

### 9.2 이벤트 스키마 진화의 영구적 부담

4장에서 본 backward compatibility는 **영원한 제약**이다. 한번 잘못 설계한 이벤트는 영원히 짊어진다. 필드 의미를 잘못 정하면 과거 이벤트를 해석하는 보정 로직(upcasting/이벤트 어댑터)을 코드에 영구히 남겨야 한다. 이벤트 설계는 "지금 편한 것"이 아니라 "10년 뒤에도 읽힐 것"을 기준으로 해야 한다. 신중함의 비용이 선불로 청구된다.

### 9.3 대용량 파티션 관리

partition_nr(3.2)과 tag 시간 버킷(5.1)으로 large partition을 방어하지만, 이 파라미터들을 잘못 잡으면 [[12 - 운영과 트러블슈팅]]의 large partition 장애가 그대로 재현된다. `nodetool tablehistograms`로 파티션 크기를 주기적으로 모니터링하고, target-partition-size와 tag 버킷 크기를 워크로드에 맞게 조정해야 한다. "설정해두고 잊는" 값이 아니다.

### 9.4 projection의 eventual consistency와 멱등 consumer

5장에서 본 대로, read model은 뒤처질 수 있고 이벤트는 중복 전달될 수 있다. UI나 API가 "방금 쓴 것이 projection에 즉시 보이리라" 가정하면 버그가 난다(read-your-writes를 projection에 기대지 말 것 — 그건 엔티티 직접 조회로 풀어야 한다). consumer는 항상 멱등해야 하고, projection이 깨지면 이벤트를 처음부터 다시 읽어 read model을 재구축(rebuild)할 수 있어야 한다. 이 rebuild 능력이 Event Sourcing의 보험이자 동시에 운영 부담이다.

### 9.5 디버깅과 정신 모델의 전환

상태 모델에 익숙한 팀에게 "현재 상태가 테이블에 없고 이벤트를 fold해야 보인다"는 사고 전환은 진입 장벽이다. "지금 이 결제 상태가 왜 이런가"를 보려면 이벤트 로그를 읽고 머릿속(또는 도구)으로 replay해야 한다. 그래서 이벤트 로그를 사람이 디코딩·열람하고 replay 결과를 재구성해 보여주는 운영 도구가 사실상 필수가 된다. 이런 도구 없이 Event Sourcing을 운영하는 것은 계기판 없이 비행하는 것과 같다.

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

---

## 핵심 요약

- **Event Sourcing은 상태가 아니라 사실의 시퀀스(이벤트)를 append-only로 저장하고, 현재 상태는 replay(fold)로 복원한다.** 이벤트 로그가 유일한 진실의 원천이며, 상태·read model은 모두 거기서 파생된 캐시다.
- **Cassandra가 event store에 잘 맞는 이유는 구조적이다.** LSM-tree의 append-only 쓰기 메커니즘이 Event Sourcing의 append-only 워크로드와 임피던스 불일치 없이 정렬되고, persistence_id를 파티션 키로 쓰면 replay가 단일 파티션 range scan이 되며, 토큰 링 분산으로 엔티티 수가 늘어도 수평 확장된다.
- **Akka Persistence Cassandra는 journal(이벤트)과 snapshot(상태 체크포인트)을 둔다.** journal PRIMARY KEY는 `((persistence_id, partition_nr), sequence_nr)`이고, **partition_nr이 무한히 긴 로그를 bounded한 파티션들로 쪼개 large partition을 회피**한다. snapshot은 진실이 아닌 최적화이므로 폐기 가능하다.
- **append-only + replay는 이벤트 스키마의 영구 backward compatibility를 강제한다.** 필드 제거·타입 변경·번호 변경·의미 변경 금지. event sourcing을 쓰는 어떤 시스템의 도메인 이벤트 호환성 규칙도 같은 뿌리(replay가 과거 이벤트를 영원히 다시 읽기 때문)에서 나온다.
- **엔티티를 가로지르는 조회는 events-by-tag 테이블(시간 버킷팅)로 푼다.** journal→tag 전파는 비동기라 projection은 eventually consistent이며, consumer는 `(persistence_id, sequence_nr)`로 멱등해야 한다. 이것이 CQRS의 읽기 측이다.
- **이벤트는 거의 삭제하지 않으므로 tombstone 압박이 낮다.** TTL 만료 전제의 TWCS는 부적합하고, replay read 효율을 중시하면 LCS, 쓰기 처리량을 중시하면 STCS(5.0에선 UCS로 스펙트럼 조절). read latency의 진짜 지렛대는 compaction 전략보다 snapshot 주기다.
- **정확성은 sequence_nr이 떠받친다.** 단일 파티션 clustering order로 엔티티 단위 순서를 보장하고, 단일 writer(=엔티티당 단일 actor) 가정으로 LWT 없이 멱등·동시성 안전을 얻는다. 결제 시스템의 중복결제 방지는 "동시성을 막는" 게 아니라 "actor 직렬 처리로 race를 제거"하는 방식이다.
- **공짜 점심은 없다.** replay 비용, 영구적 스키마 진화 부담, large partition 관리, projection의 eventual consistency, 멱등 consumer, 높은 정신적 복잡도가 그 대가다.

## 연결 노트

- [[08 - Storage Engine 내부]] — LSM-tree/SSTable immutability. 이 장의 "왜 append가 엔진과 궁합이 좋은가"의 토대.
- [[09 - Compaction 전략]] — STCS/LCS/TWCS/UCS. event store가 왜 TWCS와 결이 다르고 LCS/STCS로 기우는지.
- [[10 - Write Path와 Read Path]] — commit log/memtable/quorum. append 쓰기와 replay 읽기의 실제 경로.
- [[05 - 데이터 모델링 1 - Query First]] / [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]] — 파티션=persistence_id 설계, partition_nr/tag 버킷팅으로 large partition 회피.
- [[12 - 운영과 트러블슈팅]] — large partition·tombstone 모니터링. event store 운영의 실전 감각.
- [[04 - Tunable Consistency]] — LOCAL_QUORUM read/write로 read-your-writes(replay 안전성) 확보.
- [[11 - LWT Batch Counter 내부]] — 멱등성·동시성. event sourcing이 LWT 대신 actor 직렬화를 택하는 이유의 대조군.
- [[02 - 분산 아키텍처]] / [[03 - 복제 전략과 데이터센터]] — 토큰 링 분산과 복제. 수평 확장의 기반.
