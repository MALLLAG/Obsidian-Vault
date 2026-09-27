---
title: CQL 완전 정복 - DDL/DML, 타입, 페이징, TTL
date: 2026-06-26
tags: [cassandra, cql, 학습노트]
---

CQL(Cassandra Query Language)은 겉모습이 SQL과 매우 비슷하다. `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`이 모두 있고, `WHERE` 절도 있으며, 데이터 타입도 익숙하다.

그런데 이 익숙함이 함정이다. CQL은 학습 부담을 줄이려고 의도적으로 SQL의 형태를 빌려 왔지만, 내부에서 일어나는 일은 RDB와 근본적으로 다르다. 같은 키워드가 전혀 다른 의미로 쓰이기도 하고, RDB라면 당연히 실행됐을 쿼리가 거부되기도 하며, 성공한 쿼리가 아무 경고 없이 데이터를 덮어쓰기도 한다.

[[05 - 데이터 모델링 1 - Query First]]에서는 "쿼리를 먼저 정하고 테이블을 설계한다"는 원칙을, [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]에서는 컬렉션과 UDT의 함정을 다뤘다.

이 장에서는 그 설계를 실제 CQL 문법으로 옮기면서, 각 문장이 [[08 - Storage Engine 내부]]의 SSTable과 [[10 - Write Path와 Read Path]]의 경로에서 실제로 무엇을 하는지 자세히 살펴본다.

다루는 주제는 키스페이스와 테이블 옵션, 데이터 타입의 직렬화, upsert로 동작하는 `INSERT`와 `UPDATE`, tombstone을 만드는 `DELETE`와 TTL, 페이징과 `token()` 스캔, 그리고 LWT와 `USING TIMESTAMP`의 비용과 위험이다.

---

## CQL이라는 언어의 정체

CQL을 제대로 이해하려면 먼저 CQL이 무엇이 아닌지를 분명히 해야 한다.

CQL은 **선언적 질의 언어처럼 보이지만, 실제로는 스토리지 엔진을 다루는 얇은 명령 인터페이스**에 가깝다. SQL에서는 옵티마이저가 쿼리를 어떻게 실행할지 자유롭게 결정한다. 결과만 같다면 인덱스를 쓰든 풀스캔을 하든 조인 순서를 바꾸든 엔진이 정한다.

CQL에는 그런 자유가 거의 없다. **거의 모든 CQL 쿼리는 "파티션 키로 노드를 찾은 다음, 그 파티션의 정렬된 row들을 순서대로 읽는다"는 단 하나의 실행 계획으로 귀결된다.** 옵티마이저가 실행 방식을 바꿀 여지가 없도록 설계되었고, 그래서 이 방식으로 실행할 수 없는 쿼리는 아예 문법 단계에서 거부된다.

이 차이를 한 문장으로 줄이면 다음과 같다.

> RDB에서 SQL은 "무엇을 원하는가"를 말하고 엔진이 "어떻게"를 푼다.
> Cassandra에서 CQL은 "어떻게 저장됐는지"를 이미 알고 있는 사람이 "그 구조를 따라 읽어라"라고 명령하는 것에 가깝다.

따라서 CQL을 배우는 것은 문법을 외우는 일이 아니다. 각 문장이 8장(Storage Engine 내부)의 LSM-tree, [[02 - 분산 아키텍처]]의 토큰 링, [[04 - Tunable Consistency]]의 정족수(quorum) 위에서 어떻게 처리되는지를 머릿속에 그릴 수 있어야 한다. 이 장은 그 과정을 설명하는 데 집중한다.

CQL을 실행하는 표준 도구는 `cqlsh`이다. Python 기반 셸이며, 아래처럼 접속한다.

```bash
# 로컬 노드 접속 (기본 포트 9042)
cqlsh

# 특정 호스트, 인증, CQL 버전 명시
cqlsh 10.0.1.5 9042 -u cassandra -p cassandra

# 접속 후 유용한 셸 명령
cqlsh> DESCRIBE KEYSPACES;          -- 모든 keyspace 목록
cqlsh> USE my_keyspace;             -- 작업 keyspace 전환
cqlsh> DESCRIBE TABLE payments;     -- 테이블 DDL 재구성 출력
cqlsh> CONSISTENCY QUORUM;          -- 이 세션의 일관성 레벨 설정
cqlsh> PAGING 100;                  -- 셸 페이징 크기
cqlsh> TRACING ON;                  -- 쿼리 실행 추적(노드 경로, 지연) 출력
```

`cqlsh`의 `TRACING ON`은 학습할 때 특히 유용하다. 쿼리 하나를 실행하면 어느 노드가 코디네이터가 됐는지, 어느 레플리카로 요청이 갔는지, 각 단계에 몇 마이크로초가 걸렸는지를 보여 준다. 이 장에서 설명하는 내부 동작을 모두 직접 확인할 수 있는 수단이다.

---

## Keyspace DDL: 복제의 경계를 정한다

CQL의 최상위 컨테이너는 **keyspace**이다. RDB의 "데이터베이스(스키마)"에 해당하지만, keyspace는 단순한 네임스페이스에 그치지 않는다. **keyspace는 "이 안의 데이터를 어떤 방식으로, 몇 벌, 어느 데이터센터에 복제할 것인가"를 정하는 복제 정책의 경계**이다. keyspace가 존재하는 이유가 바로 이것이다.

```cql
CREATE KEYSPACE payments
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'seoul': 3,
  'tokyo': 3
}
AND durable_writes = true;
```

### replication: 데이터를 몇 벌, 어디에 둘 것인가

`replication` 맵은 [[03 - 복제 전략과 데이터센터]]에서 자세히 다룬 복제 전략(replication strategy)과 복제 계수(replication factor)를 선언한다. 전략은 두 가지이다.

| 전략                              | 의미                                  | 사용 시점                  |
| ------------------------------- | ----------------------------------- | ---------------------- |
| `SimpleStrategy`                | 데이터센터/랙 토폴로지를 무시하고 링에서 다음 N개 노드에 복제 | 단일 DC, 학습/개발용 **only** |
| `NetworkTopologyStrategy` (NTS) | 데이터센터별로 복제 계수를 독립 지정. 랙을 인식해 분산     | 프로덕션 **전부**            |

> [!warning] 함정
> `SimpleStrategy`로 프로덕션 keyspace를 만들면 나중에 데이터센터를 추가할 때 복제 토폴로지가 망가진다. SimpleStrategy는 랙(rack)을 인식하지 않으므로 같은 랙에 복제본이 몰릴 수 있고, 멀티 DC로 확장하기도 어렵다. **프로덕션에서는 단일 DC라도 처음부터 `NetworkTopologyStrategy`로 시작해야 한다.** `'class': 'NetworkTopologyStrategy', 'datacenter1': 3`처럼 DC 이름 하나만 적어도 된다.

`NetworkTopologyStrategy`에서 `'seoul': 3`이 정확히 무엇을 뜻하는지 살펴보자. 이 설정은 "seoul 데이터센터 안에서 각 파티션을 3개의 노드에 복제한다"는 뜻이다. NTS는 여기서 더 나아가 그 3개의 복제본을 **서로 다른 랙(rack)에 배치하려고 시도**한다. 전원이나 스위치 장애로 랙 하나가 통째로 죽어도 데이터를 잃지 않게 하기 위해서다. 토큰 링에서 NTS가 복제본을 고르는 과정을 그리면 다음과 같다.

```text
          토큰 링 (seoul DC, RF=3, 3개 랙)
                       토큰 0
                         │
        node-r3-c ◄──────┼──────► node-r1-a   ← 파티션 키 해시가
       (rack3)           │       (rack1)         이 지점에 떨어짐
            │            │            │
            │     [복제본 선정: 링을 시계방향으로 돌며       ]
            │     [ 아직 안 쓴 '랙'을 우선해 3개를 채운다    ]
            │                         │
        node-r2-b ◄──────────────► node-r1-b
       (rack2)                     (rack1)

  선정 결과: node-r1-a(rack1) → node-r2-b(rack2) → node-r3-c(rack3)
            서로 다른 랙 3개에 분산 → 랙 단위 장애 내성
```

### durable_writes: commit log를 건너뛸 것인가

`durable_writes`의 기본값은 `true`이며, 거의 항상 그대로 두어야 한다. 이 옵션은 10장(Write Path와 Read Path)에서 설명하는 쓰기 경로와 직접 연결된다.

쓰기가 노드에 도착하면 정상적인 경우 두 곳에 동시에 기록된다. 하나는 디스크의 **commit log**(append-only, 내구성 보장)이고, 다른 하나는 메모리의 **memtable**(빠른 조회)이다. `durable_writes = false`로 설정하면 **commit log 기록을 건너뛴다.** 쓰기는 memtable에만 들어가므로, memtable이 SSTable로 flush되기 전에 노드가 죽으면 그 데이터는 영구히 사라진다.

엄밀히 말하면 `durable_writes = true`라도 commit log가 쓰기마다 디스크에 `fsync`되지는 않는다. 기본 동기화 모드는 **periodic**(기본 `commitlog_sync_period` 10초)이므로, 쓰기는 commit log 버퍼에 기록되는 즉시 ACK되고 실제 fsync는 주기적으로 일어난다.

따라서 전원 장애가 나면 마지막 몇 초 동안의 쓰기는 유실될 수 있다(쓰기마다 fsync가 필요하면 `commitlog_sync: batch`나 `group`을 쓰면 되지만, 지연이 늘어난다). 이 sync 모드의 트레이드오프는 10장에서 다룬다.

```text
  durable_writes = true (기본, 안전)
    write ──┬──► commit log (fsync, 디스크) ──► 노드 죽어도 복구 가능
            └──► memtable (메모리)

  durable_writes = false (위험)
    write ──────► memtable (메모리)  ── flush 전 노드 다운 시 영구 소실
                  (commit log 생략 → 쓰기 약간 빠름)
```

> [!warning] 함정
> `durable_writes = false`는 "어차피 다른 DC에 복제되니 한 DC의 내구성은 포기해도 된다"는 특수한 경우에만 고려한다. 일반적인 결제나 금융 데이터에서는 절대 끄면 안 된다. 쓰기 성능을 조금 얻으려고 데이터 영속성을 포기하는 선택은 결제 시스템에서 받아들일 수 없다.

### keyspace 변경과 삭제

```cql
-- 복제 계수 변경 (예: seoul RF 3 → 5). 변경 후 반드시 nodetool repair 필요
-- 주의: replication 맵은 통째로 '교체'된다. 빠뜨린 DC(여기 tokyo)는
--       RF 0이 되어 그 DC의 복제가 조용히 끊긴다. 항상 전체 DC를 명시하라.
ALTER KEYSPACE payments
WITH replication = {'class': 'NetworkTopologyStrategy', 'seoul': 5, 'tokyo': 3};

-- keyspace 삭제 (그 안의 모든 테이블/데이터 제거)
DROP KEYSPACE payments;
```

`ALTER KEYSPACE`로 RF를 올려도 기존 데이터가 저절로 새 복제본에 복사되지는 **않는다.** Cassandra는 "이제부터 이 데이터는 5벌이어야 한다"는 메타데이터만 바꾼다. 실제 데이터를 새 복제본에 채우려면 `nodetool repair`를 실행해 anti-entropy를 수행해야 한다.

그 전까지는 새 복제본에 데이터가 없으므로, `CONSISTENCY ALL` 같은 강한 읽기가 빈 결과를 섞어 반환할 수 있다. RF 변경은 [[12 - 운영과 트러블슈팅]]에서 다루는 신중한 운영 절차이다.

---

## Table DDL: PRIMARY KEY가 모든 것을 결정한다

테이블 정의에서 가장 중요한 줄은 `PRIMARY KEY`이다. RDB에서 PK는 "행을 식별하는 유니크 제약"일 뿐이지만, Cassandra에서 **PRIMARY KEY는 데이터가 클러스터의 어디에 저장되는지(파티션 키)와 디스크에서 어떻게 정렬되는지(클러스터링 키)를 함께 정하는 물리 설계 그 자체**이다. 이 주제는 5장(데이터 모델링 1 - Query First)의 핵심이었다. 여기서는 CQL 문법의 각 형태가 정확히 무엇을 뜻하는지 복습하며 정리한다.

```cql
CREATE TABLE payments.payment_by_user (
    user_id       text,
    bucket        text,          -- 예: '2026-06' (시간 버킷팅)
    paid_at       timestamp,
    payment_id    uuid,
    amount        bigint,
    status        text,
    pg_name       text,
    PRIMARY KEY ((user_id, bucket), paid_at, payment_id)
) WITH CLUSTERING ORDER BY (paid_at DESC, payment_id ASC);
```

### PRIMARY KEY 구문의 세 가지 형태

PRIMARY KEY 선언은 괄호 위치에 따라 의미가 완전히 달라진다. CQL을 처음 배우는 사람이 가장 많이 틀리는 부분이다.

```text
  PRIMARY KEY (a)
    └─ 파티션 키 = a, 클러스터링 키 없음
       → a 하나로 파티션 결정, 파티션당 row 1개

  PRIMARY KEY (a, b, c)
    └─ 파티션 키 = a (첫 요소만!), 클러스터링 키 = b, c
       → a로 파티션 결정, 그 안에서 (b, c) 순으로 정렬된 여러 row

  PRIMARY KEY ((a, b), c, d)
    └─ 파티션 키 = (a, b) 복합, 클러스터링 키 = c, d
       → (a, b) 둘 다 있어야 파티션 결정, 그 안에서 (c, d)로 정렬
```

**파티션 키(partition key)** 는 2장(분산 아키텍처)에서 설명한 토큰 링에서 이 데이터가 어느 노드로 갈지를 결정한다. 파티션 키를 해시(기본 파티셔너는 `Murmur3Partitioner`)한 결과가 토큰이고, 그 토큰이 링의 어느 구간에 속하는지에 따라 담당 노드와 복제본들이 정해진다.

**클러스터링 키(clustering key)** 는 같은 파티션 안에서 row들이 **디스크에 물리적으로 정렬되어 저장되는 순서**를 결정한다. 이 점이 핵심이다. RDB에서 정렬은 읽을 때 `ORDER BY`로 수행하는 런타임 연산이지만, Cassandra에서 정렬은 **쓸 때 이미 정해져 있는 디스크 레이아웃**이다.

```text
  파티션 ('user-A', '2026-06') 의 내부 (디스크 정렬 상태)
  CLUSTERING ORDER BY (paid_at DESC):

  ┌──────────────────────────────────────────────────────┐
  │ paid_at=2026-06-30T23:59  payment_id=u9  amount=5000  │ ← 최신
  │ paid_at=2026-06-30T18:02  payment_id=u8  amount=3000  │
  │ paid_at=2026-06-29T11:20  payment_id=u7  amount=9900  │
  │ ...                                                    │
  │ paid_at=2026-06-01T00:03  payment_id=u1  amount=1000  │ ← 오래됨
  └──────────────────────────────────────────────────────┘
   이미 정렬된 채로 연속 저장 → "최근 결제 N건" = 순차 읽기 = 초고속
```

`CLUSTERING ORDER BY (paid_at DESC)`로 정의하면 데이터가 디스크에 최신순으로 저장된다. 따라서 "이 사용자의 최근 결제 20건"은 디스크의 한 위치에서 시작해 순서대로 읽어 오는 연산이 된다. 인덱스 탐색도, 정렬 연산도 필요 없다. Cassandra의 읽기가 빠른 이유가 여기에 있고, "쓰기 시점에 읽기 패턴을 결정한다"는 Query-First 원칙이 물리적으로 구현되는 방식도 이것이다.

### WITH 옵션: 테이블의 런타임 특성을 정한다

`PRIMARY KEY`가 데이터의 형태를 정한다면, `WITH` 옵션은 그 테이블이 디스크에서 어떻게 관리되는지(compaction, 압축, GC, 캐시, 만료)를 정한다. 운영 성능과 직결되는 주요 옵션을 하나씩 살펴본다.

```cql
CREATE TABLE payments.payment_by_user ( ... )
WITH CLUSTERING ORDER BY (paid_at DESC, payment_id ASC)
AND compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'DAYS',
    'compaction_window_size': 1
}
AND compression = {
    'class': 'org.apache.cassandra.io.compress.LZ4Compressor',
    'chunk_length_in_kb': 16
}
AND gc_grace_seconds = 864000          -- 10일 (기본값)
AND default_time_to_live = 0           -- TTL 없음 (기본값)
AND caching = {'keys': 'ALL', 'rows_per_partition': 'NONE'}
AND speculative_retry = '99p'
AND read_repair = 'BLOCKING';
```

#### CLUSTERING ORDER BY

앞에서 본 디스크 정렬 순서를 명시한다. 데이터의 물리 레이아웃 자체이므로 한번 정하면 `ALTER`로 바꿀 수 없다. 기본값은 모든 클러스터링 키에 대해 `ASC`이다. 시계열 데이터나 이벤트성 데이터는 보통 시간 컬럼을 `DESC`로 두어 최신 데이터부터 읽기 쉽게 만든다.

> [!warning] 함정
> `CLUSTERING ORDER`를 지정하면 `SELECT`의 `ORDER BY`에는 정의한 순서나 그 **완전한 역순**만 쓸 수 있다. 임의의 컬럼으로 정렬할 수는 없다. 디스크에 정렬된 순서대로 읽거나 거꾸로 읽는 두 가지 방법뿐이다. 5장에서 정렬 방식을 설계 단계에서 확정해야 한다고 한 이유가 이것이다.

#### compaction: SSTable을 어떻게 병합할 것인가

이 주제는 [[09 - Compaction 전략]]에서 다룬다. memtable이 flush되면 불변(immutable) SSTable이 디스크에 쌓이고, 같은 키가 여러 SSTable에 흩어진다. compaction은 이 SSTable들을 주기적으로 병합해 읽기 효율을 회복하고 tombstone을 정리한다. 어떤 전략을 고르느냐가 곧 워크로드에 얼마나 잘 맞느냐를 결정한다.

| 전략 | 약어 | 최적 워크로드 | 핵심 동작 |
|---|---|---|---|
| `SizeTieredCompactionStrategy` | STCS | 쓰기 많음, 일반 | 비슷한 크기 SSTable 묶어 병합. 기본값 |
| `LeveledCompactionStrategy` | LCS | 읽기 많음, 갱신 잦음 | 레벨 구조로 키당 SSTable 수 최소화. read 안정적, write 증폭 |
| `TimeWindowCompactionStrategy` | TWCS | 시계열, TTL 데이터 | 시간 창 단위로 SSTable 격리. 만료된 창 통째 삭제 |
| `UnifiedCompactionStrategy` | UCS | 범용(5.0 신규) | STCS/LCS를 파라미터로 통합. 튜닝 유연 |

Cassandra 5.0의 큰 변화 중 하나는 `UnifiedCompactionStrategy`(UCS)의 도입이다. UCS는 STCS와 LCS의 장점을 파라미터로 조절하는 하나의 전략으로 통합했으며, `scaling_parameters`로 동작을 연속적으로 조절할 수 있다. 다만 기존 테이블의 기본값은 여전히 STCS이므로 UCS는 명시적으로 선택해야 한다.

결제나 이벤트 로그처럼 "시간순으로 쌓이고 일정 기간이 지나면 만료되는" 데이터에는 TWCS가 가장 적합한 경우가 많다. 그 이유는 TTL 절에서 tombstone과 함께 설명한다.

#### compression: 압축으로 디스크 I/O를 줄인다

Cassandra는 SSTable을 **청크(chunk) 단위로 압축**해 저장한다. 기본 압축기는 `LZ4Compressor`(5.0 기준)이며, 압축률보다 속도를 우선한다. `chunk_length_in_kb`는 압축 단위인 청크의 크기이다.

```text
  chunk_length_in_kb 의 트레이드오프

  작게(예: 4KB)  →  특정 row 읽을 때 압축 해제할 양 적음 → 랜덤 읽기 유리
                    대신 압축률 낮고 청크 메타데이터 늘어남

  크게(예: 64KB) →  압축률 좋고 메타데이터 적음 → 순차 읽기/디스크 절약 유리
                    대신 한 row 읽으려 큰 청크 통째 해제 → 랜덤 읽기 손해
```

5.0의 기본 `chunk_length_in_kb`는 16KB이다(과거 버전의 64KB에서 낮아졌다). 랜덤 읽기가 많으면 더 줄이고, 순차 스캔이나 디스크 절약이 우선이면 늘린다. 압축기로는 `LZ4`(기본, 빠름), `Snappy`, `Deflate`(압축률은 높지만 느림), `Zstd`(5.0에서 균형이 좋아 많이 쓰임) 등을 선택할 수 있다.

#### gc_grace_seconds: tombstone의 수명

기본값은 **864000초(10일)**. 이 값은 Cassandra에서 가장 미묘하고 위험한 옵션 중 하나이다. tombstone(삭제 표식)이 생긴 뒤 `gc_grace_seconds`가 지나야 compaction이 그 tombstone을 영구히 제거할 수 있다.

바로 지우지 않고 10일이나 기다리는 이유는 **삭제가 모든 복제본에 전파될 시간을 확보하기 위해서**다. 이 점을 이해하는 것이 중요하다.

```text
  "좀비 데이터(zombie)" 시나리오 — gc_grace_seconds가 없다면

  t0: 3개 복제본 모두에 row X 존재
  t1: DELETE X → 복제본 A, B에는 tombstone 도착, C는 네트워크 단절로 누락
  t2: (gc_grace=0 가정) compaction이 A, B의 tombstone 즉시 제거
  t3: C가 복구됨. read repair / 가십 과정에서
      "A,B엔 X가 없는데 C엔 X가 있네?" → C의 X를 A,B로 복제!
  결과: 지웠던 X가 부활(좀비). 삭제가 무효화됨.
```

`gc_grace_seconds = 10일`은 "이 기간 안에 `nodetool repair`를 실행해 tombstone을 모든 복제본에 확실히 전파하라"는 안전 마진이다. 그래서 **운영 규칙도 단순하다. `gc_grace_seconds`보다 짧은 주기로 정기 repair를 실행해야 한다.** repair 주기가 gc_grace보다 길면 삭제한 데이터가 좀비로 되살아날 수 있다. 이 메커니즘 전체는 9장(Compaction 전략)과 12장(운영과 트러블슈팅)에서 더 자세히 다룬다.

#### default_time_to_live: 테이블 전체의 만료 시간

기본값은 0(만료 없음)이다. 0이 아니면 이 테이블에 들어가는 모든 row가 기본적으로 그 초만큼 유지된 뒤 자동으로 만료된다. 자세한 내용은 TTL 절에서 살펴본다.

#### caching: 무엇을 메모리에 둘 것인가

```cql
caching = {'keys': 'ALL', 'rows_per_partition': 'NONE'}
```

`keys`는 **key cache**(파티션 키에서 SSTable 내 위치로의 매핑)를 캐시할지를 정하고, `rows_per_partition`은 **row cache**(실제 row 데이터)를 파티션당 몇 개까지 캐시할지를 정한다. 기본값은 `keys: ALL`, `rows_per_partition: NONE`이다.

> [!warning] 함정
> row cache(`rows_per_partition`)를 켜는 것은 대부분의 경우 안티패턴이다. 쓰기가 한 번이라도 일어나면 해당 파티션의 row cache 전체가 무효화되고, 넓은 파티션을 통째로 메모리에 올리므로 힙을 많이 차지한다. 거의 변하지 않고 작으면서 자주 읽히는(hot) 파티션에만 신중하게 써야 한다. 반면 key cache는 켜 두는 것이 거의 항상 이득이다.

#### ALTER TABLE로 바꿀 수 있는 것과 없는 것

```cql
ALTER TABLE payments.payment_by_user WITH gc_grace_seconds = 432000;   -- OK
ALTER TABLE payments.payment_by_user ADD refund_amount bigint;          -- OK (컬럼 추가)
ALTER TABLE payments.payment_by_user DROP pg_name;                      -- OK (주의)
-- PRIMARY KEY 변경? CLUSTERING ORDER 변경? → 불가. 테이블 재생성+마이그레이션
```

`WITH` 옵션과 PK가 아닌 컬럼의 추가와 삭제는 자유롭게 할 수 있다. 그러나 **PRIMARY KEY와 CLUSTERING ORDER는 데이터의 물리 구조이므로 ALTER로 절대 바꿀 수 없다.** 바꾸려면 새 테이블을 만들고 데이터를 옮겨야 한다. 5장에서 PK를 신중하게 설계하라고 거듭 강조한 이유가 이것이다.

---

## 데이터 타입 총정리

CQL 타입을 RDB 타입처럼 "대충 맞으면 된다"는 식으로 다루면 미묘한 버그가 생긴다. 각 타입은 **고정된 바이트 직렬화 규칙**을 따르며, 이 직렬화 방식이 clustering 정렬 순서와 비교 동작, 저장 크기를 결정한다. 전체 목록을 용도와 함께 정리한다.

### 문자열 타입

| 타입 | 의미 | 인코딩/검증 | 비고 |
|---|---|---|---|
| `text` | UTF-8 문자열 | UTF-8 검증함 | `varchar`와 **완전 동일**(별칭) |
| `varchar` | `text`의 별칭 | UTF-8 | 그냥 `text` 쓰면 됨 |
| `ascii` | US-ASCII 문자열 | ASCII만 허용(검증) | 비-ASCII 거부. 순수 ASCII 키에 약간 효율적 |

### 정수 타입

| 타입 | 크기 | 범위 |
|---|---|---|
| `tinyint` | 1바이트 | -128 ~ 127 |
| `smallint` | 2바이트 | -32,768 ~ 32,767 |
| `int` | 4바이트 | 약 -21억 ~ 21억 |
| `bigint` | 8바이트 | 약 -9.2×10^18 ~ 9.2×10^18 |
| `varint` | 가변 | 임의 정밀도 정수(무제한) |

> 결제 금액은 `bigint`(원이나 센트 같은 최소 화폐 단위)로 다루는 것이 정석이다. `varint`는 가변 길이라 정렬과 저장이 조금 무겁고, `int`는 누적 금액이 커지면 오버플로가 날 위험이 있다. **금액에 `float`나 `double`을 쓰면 안 된다**(다음 항목 참고).

### 부동소수점/고정소수점

| 타입 | 크기 | 정밀도 | 용도 |
|---|---|---|---|
| `float` | 4바이트 | IEEE 754 단정밀도 | 근사값 OK인 측정치 |
| `double` | 8바이트 | IEEE 754 배정밀도 | 근사값 OK인 측정치 |
| `decimal` | 가변 | 임의 정밀도 십진 | **돈, 정확한 십진** |

> [!warning] 함정
> `0.1 + 0.2 != 0.3`. IEEE 754 이진 부동소수점은 십진 소수를 정확히 표현하지 못한다. 환율 계산처럼 십진 정확도가 필요하면 `decimal`을, 정수 단위 금액이면 `bigint`를 써야 한다. `double`로 잔액을 누적하면 시간이 지날수록 오차가 쌓인다.

### 그 밖의 스칼라 타입

| 타입 | 의미 | 직렬화/정렬 특성 |
|---|---|---|
| `boolean` | true/false | 1바이트 |
| `blob` | 임의 바이트열 | 직렬화/역직렬화 없이 그대로. 16진수 리터럴 `0x...` |
| `inet` | IPv4/IPv6 주소 | 4 또는 16바이트로 저장 |
| `uuid` | 임의 버전 UUID(보통 v4 랜덤) | 128비트. 타입은 버전 검증 안 함. `uuid()` 함수가 v4 생성. 정렬 무의미 |
| `timeuuid` | v1(시간 기반) UUID | 128비트. **시간순 정렬됨**. 비교 시 시각이 1차 키 |
| `counter` | 분산 카운터 | 특수 타입. 별도 절에서 |

### 시간 타입: 가장 헷갈리는 영역

| 타입 | 의미 | 저장 | 예 |
|---|---|---|---|
| `timestamp` | 특정 순간(밀리초) | epoch부터의 ms (8바이트, UTC) | `'2026-06-26 09:00:00+0900'` |
| `date` | 날짜(시각 없음) | epoch 기준 일수(4바이트 부호없음) | `'2026-06-26'` |
| `time` | 하루 중 시각 | 자정부터의 나노초(8바이트) | `'09:00:00.000000000'` |
| `duration` | 기간(달/일/나노초) | 3개 가변 정수 | `12h30m`, `1mo`, `89h4m48s` |

`timestamp`는 내부적으로 **UTC 기준 밀리초 단위의 epoch 정수**일 뿐이며, 타임존 정보는 저장하지 않는다. `'2026-06-26 09:00:00+0900'`을 넣으면 `+0900`을 적용해 UTC로 변환한 ms 값만 저장된다. 읽을 때 `cqlsh`는 클라이언트 타임존으로 표시한다. 그래서 **애플리케이션에서 타임존을 일관되게(보통 UTC로) 다루는 규칙**이 없으면 표시가 혼란스러워진다.

`duration`은 특히 조심해야 한다. "1달"이 며칠인지는 시작 날짜에 따라 다르므로, `duration`은 **달, 일, 나노초를 따로 저장**한다. 그래서 두 `duration`을 단순히 비교(`>`, `<`)할 수 없고, 클러스터링 키로 쓸 수도 없다. 순수하게 "구간의 길이"를 표현하는 용도이다.

### uuid와 timeuuid: 왜 둘을 구분하는가

이 구분은 실무에서 매우 중요하다. 둘 다 128비트 식별자이지만 결정적인 차이가 있다.

```text
  uuid (version 4, 랜덤)
    ┌──────────────────────────────────────────┐
    │ 122비트가 난수. 충돌 사실상 0.             │
    │ 정렬해도 무의미(랜덤 순서).               │
    │ 용도: 그냥 고유 ID가 필요할 때.           │
    └──────────────────────────────────────────┘

  timeuuid (version 1, 시간 기반)
    ┌──────────────────────────────────────────┐
    │ 60비트 타임스탬프(100ns 단위) + 노드 MAC  │
    │  + 클럭 시퀀스로 구성.                    │
    │ → 생성 시각순으로 정렬된다!               │
    │ → 같은 ms에 여러 개 생성돼도 유일+정렬됨. │
    │ 용도: 시간순 정렬이 필요한 이벤트 ID.     │
    └──────────────────────────────────────────┘
```

`timeuuid`를 클러스터링 키로 쓰면 고유성과 시간순 정렬을 한 컬럼으로 동시에 얻을 수 있다. 이벤트 소싱([[13 - 실전 Event Sourcing on Cassandra]])에서 이벤트 ID로 자주 쓰는 이유가 이것이다. `timestamp`만으로는 같은 밀리초에 발생한 이벤트들의 순서와 유일성을 보장할 수 없지만, `timeuuid`는 클럭 시퀀스로 그 문제까지 해결한다.

관련 내장 함수는 다음과 같다.

```cql
SELECT now();                          -- 현재 시각 기반 새 timeuuid 생성
SELECT uuid();                         -- 랜덤 uuid 생성
SELECT toTimestamp(now());             -- timeuuid → timestamp 추출
SELECT toDate(now());                  -- timeuuid → date 추출
SELECT minTimeuuid('2026-06-26 00:00+0900');  -- 그 시각의 '최소' timeuuid
SELECT maxTimeuuid('2026-06-26 23:59+0900');  -- 그 시각의 '최대' timeuuid

-- 범위 쿼리에 minTimeuuid/maxTimeuuid를 쓰면
-- "특정 시간 구간의 이벤트"를 timeuuid 클러스터링 키로 슬라이스할 수 있다
SELECT * FROM events
WHERE stream_id = 'payment-123'
  AND event_id > minTimeuuid('2026-06-26 00:00+0900')
  AND event_id < maxTimeuuid('2026-06-26 23:59+0900');
```

> [!warning] 주의: `SELECT`에는 `FROM`이 필수다
> 위의 `SELECT now();`, `SELECT uuid();`처럼 함수만 단독으로 평가하고 싶더라도, Cassandra CQL 문법은 `FROM` 절이 없는 `SELECT`를 허용하지 않는다(PostgreSQL 방식의 `SELECT now();`는 문법 오류이다).
>
> cqlsh에서는 보통 row가 하나뿐인 시스템 테이블을 빌려 `SELECT now() FROM system.local;`처럼 쓴다. 실제로 쓰기를 할 때는 `INSERT INTO events (id, ...) VALUES (now(), ...)`처럼 값 위치에 함수를 직접 넣는다.

`minTimeuuid`와 `maxTimeuuid`가 만드는 값은 **실제 식별자로 쓰면 안 되는 경계값**이다(같은 시각이면 값이 모두 같다). 범위 비교의 하한과 상한으로만 써야 한다.

### 컬렉션과 UDT: 요약과 경고

`set<T>`, `list<T>`, `map<K,V>`, 그리고 사용자 정의 타입(UDT)과 `tuple`은 6장(데이터 모델링 2 - 고급 타입과 안티패턴)에서 자세히 다뤘다. 여기서는 CQL 문법과 핵심 함정만 정리한다.

```cql
CREATE TYPE address (
    line1 text, city text, zipcode text
);

CREATE TABLE customers (
    id uuid PRIMARY KEY,
    emails set<text>,                     -- 중복 없음, 정렬 저장
    recent_logins list<timestamp>,        -- 순서 유지, 중복 허용
    attributes map<text, text>,           -- 키-값
    home frozen<address>                  -- UDT, frozen 통째 저장
);
```

| 컬렉션 | 특성 | 함정 |
|---|---|---|
| `set<T>` | 정렬·중복없음 | 안전한 편. 원소 추가/삭제 멱등 |
| `list<T>` | 순서유지·중복허용 | 인덱스 연산(`[i]`)이 read-before-write 유발. 가급적 set 권장 |
| `map<K,V>` | 키-값 | 키 단위 갱신은 OK. 무한정 커지면 위험 |
| `frozen<T>` | 통째로만 읽기/쓰기 | 한 필드만 갱신해도 전체 재기록 |

> [!warning] 함정
> 컬렉션은 **단일 파티션 안의 작은 보조 데이터**를 위한 것이다. 컬렉션에 원소를 수천 개 넣으면 읽을 때 통째로 역직렬화되고, frozen이 아닌 컬렉션의 원소들은 각각 별도의 셀(cell)로 저장되어 tombstone을 대량으로 만든다. 관계형의 1:N 관계를 컬렉션으로 대체하려는 시도는 거의 항상 안티패턴이다. 별도 테이블과 클러스터링 키로 풀어야 한다. 자세한 내용은 6장을 참고한다.

---

## DML의 의외의 사실: INSERT도 UPDATE도 upsert다

이제 CQL이 SQL과 가장 크게 다른 지점에 왔다. RDB에 익숙한 사람이 반드시 기억해야 할 사실이 하나 있다.

> **`INSERT`와 `UPDATE`는 둘 다 단순히 "이 위치에 이 값을 써라"일 뿐이다. 존재 여부를 검사하지 않는다. CQL에는 "삽입"과 "갱신"의 구분이 없다. 둘 다 upsert다.**

### 왜 구분이 없는가: LSM-tree의 본질

이것은 Cassandra의 특이한 동작이 아니라 8장(Storage Engine 내부)에서 설명한 LSM-tree 구조에서 필연적으로 나오는 결과이다. 쓰기는 항상 **append-only**이다. memtable에 셀을 하나 추가하고 commit log에 기록할 뿐, 디스크의 기존 데이터를 찾아가 고치지 않는다. 키가 이미 있는지 확인하려면 디스크의 모든 SSTable을 읽어야 하므로 쓰기가 읽기만큼 느려진다. LSM-tree는 이 확인을 **하지 않는** 대신 쓰기를 빠르게 만든다.

```text
  RDB (B-tree, in-place update)
    INSERT → "키 있나?" 확인 → 있으면 에러(PK 위반)
    UPDATE → "키 있나?" 확인 → 없으면 0 rows affected
    → 둘 다 '존재 확인'이라는 read가 선행됨

  Cassandra (LSM-tree, append-only)
    INSERT → 그냥 memtable에 셀 추가 (확인 안 함)
    UPDATE → 그냥 memtable에 셀 추가 (확인 안 함)
    → 둘 다 동일하게 '새 버전 셀을 append'할 뿐
    → 나중에 읽을 때 timestamp 가장 큰 셀이 이김 (LWW)
```

그래서 다음 네 문장은 **결과가 사실상 같다.**

```cql
-- 모두 (id='p1')에 amount=5000, status='paid'를 쓴다. 기존 유무 무관.
INSERT INTO payments (id, amount, status) VALUES ('p1', 5000, 'paid');
UPDATE payments SET amount = 5000, status = 'paid' WHERE id = 'p1';

-- 이미 'p1'이 있어도 INSERT는 에러 안 남. 조용히 덮어씀.
INSERT INTO payments (id, amount, status) VALUES ('p1', 9999, 'failed');
-- 'p1'이 없어도 UPDATE는 '0 rows' 같은 거 없음. 그냥 새로 만듦.
UPDATE payments SET amount = 9999 WHERE id = 'p1';
```

이 사실은 실무에 큰 영향을 준다.

> [!warning] 함정 1: 중복 결제를 막을 수 없다
> "결제 ID가 이미 있으면 INSERT가 실패한다"고 기대하면 안 된다. CQL의 일반 `INSERT`는 기존 결제를 아무 경고 없이 덮어쓴다. PK 유니크 제약으로 멱등성을 보장하던 RDB 패턴이 **그대로 통하지 않는다.** 정말로 "없을 때만 삽입"해야 한다면 `IF NOT EXISTS`(경량 트랜잭션, 뒤에서 설명)를 써야 하는데, 비용이 크다.

> [!warning] 함정 2: 부분 갱신이 정상 동작이다
> `UPDATE payments SET status='paid' WHERE id='p1'`은 `p1`이 없으면 `status`만 있고 나머지 컬럼은 null인 "반쪽 row"를 만든다. RDB라면 0 rows affected로 끝났을 일이 Cassandra에서는 새 row 생성이 된다.

### INSERT와 UPDATE의 미묘한 차이 두 가지

"사실상 같다"고 했지만 완전히 같지는 않다. 알아 둬야 할 비대칭이 두 가지 있다.

**1) 컬렉션 연산은 UPDATE에만 있다.**

```cql
-- set에 원소 추가/삭제는 UPDATE 문법으로만 가능
UPDATE customers SET emails = emails + {'new@x.com'} WHERE id = ...;
UPDATE customers SET emails = emails - {'old@x.com'} WHERE id = ...;
UPDATE customers SET attributes['theme'] = 'dark' WHERE id = ...;
-- INSERT는 컬렉션 '전체 값'을 통째로 쓸 수만 있다
INSERT INTO customers (id, emails) VALUES (..., {'a@x.com','b@x.com'});
```

**2) "row 존재" 개념과 정적 컬럼.** 미묘한 부분이지만, `INSERT`로 클러스터링 키만 있고 일반 컬럼이 전혀 없는 row를 쓰면 Cassandra는 그 row가 존재한다는 것을 표시하는 빈 셀(row marker)을 남긴다. 반면 `UPDATE`로 일반 컬럼을 모두 null로 만들면 그 row는 셀이 없는 상태가 되어 읽기 결과에서 사라질 수 있다. 이 작은 차이는 정적(static) 컬럼이나 row 존재 여부를 판정하는 모델에서 가끔 문제를 일으킨다. 대부분은 신경 쓸 필요가 없지만, "분명히 INSERT했는데 SELECT에 보이지 않는다" 같은 이상한 현상을 만나면 이 부분을 의심해 봐야 한다.

### DELETE는 지우지 않는다: tombstone을 만든다

`DELETE`도 직관과 다르게 동작한다. LSM-tree에서 디스크의 SSTable은 불변이다. 그래서 삭제는 데이터를 찾아 제거하는 것이 아니라, **"이 셀은 이 timestamp 이후로 삭제되었다"는 표식(tombstone)을 새로 append**하는 것이다. 즉 삭제 역시 쓰기이다.

```text
  DELETE FROM payments WHERE id='p1';

  SSTable-1 (오래됨):  p1 → amount=5000, status='paid'  (살아있는 듯 보임)
  SSTable-2 (최신):    p1 → [TOMBSTONE @ ts=T2]          ← DELETE가 만든 묘비

  READ 시: 두 SSTable 병합 → tombstone(T2)이 데이터(T1<T2)를 가림
           → "삭제됨"으로 보고됨. 하지만 디스크엔 둘 다 존재!
```

tombstone의 수명은 앞에서 본 `gc_grace_seconds`(기본 10일)이다. 그 기간이 지난 뒤 compaction이 실행되어야 원본 데이터와 tombstone이 함께 디스크에서 사라진다. 여기서 운영에 관한 두 가지 사실이 나온다.

1. **삭제해도 디스크가 바로 비워지지 않는다.** 오히려 원본과 tombstone이 함께 있으므로 한동안 데이터가 늘어난다. 디스크 부족을 해결하려고 대량 DELETE를 실행하면 역효과가 날 수 있다.
2. **tombstone이 쌓이면 읽기가 느려진다.** 살아 있는 row를 읽으려면 그 앞에 쌓인 수많은 tombstone을 스캔해야 한다. 이것이 악명 높은 "tombstone hell"이다. 자세한 메커니즘과 대응 방법은 9장을 참고한다.

```cql
DELETE FROM payments WHERE id = 'p1';                    -- 파티션 전체 삭제(partition tombstone)
DELETE status FROM payments WHERE id = 'p1';             -- 특정 컬럼만(cell tombstone)
DELETE FROM payments WHERE id='p1' AND paid_at < '2026-01-01';  -- range tombstone
DELETE emails FROM customers WHERE id = ...;             -- 컬렉션 통째 삭제
```

> [!warning] 함정: 큐를 Cassandra로 만들지 않는다
> 넣고 빼는 작업(INSERT/DELETE)을 반복하는 큐나 작업 테이블을 Cassandra로 모델링하면, 처리가 끝날 때마다 tombstone이 쌓여 곧 읽기가 tombstone을 훑는 작업이 된다. "Cassandra anti-pattern: queue"는 잘 알려진 고전적 사례이다. 이런 워크로드는 Kafka나 SQS 같은 전용 도구로 처리해야 한다. 자세한 내용은 6장을 참고한다.

---

## SELECT의 제약: 쿼리의 자유는 설계에서 나온다

5장에서 SELECT의 제약을 자세히 다뤘지만, CQL을 설명하는 이 장에서도 다시 정리할 가치가 있다. CQL의 `SELECT`가 SQL과 다른 점은 할 수 없는 일의 목록이 길다는 것이다. 이 제약들은 모두 "옵티마이저가 사용자 모르게 풀스캔이나 정렬을 하지 못하게 막는다"는 하나의 원칙에서 나온다.

```cql
-- 허용: 파티션 키로 한 파티션 지정 후, 클러스터링 키로 슬라이스
SELECT * FROM payment_by_user
WHERE user_id = 'A' AND bucket = '2026-06'      -- 파티션 키 전체 등호
  AND paid_at >= '2026-06-01' AND paid_at < '2026-07-01';  -- 클러스터링 범위

-- 거부되는 것들:
SELECT * FROM payment_by_user WHERE status = 'failed';
--   → 파티션 키 없이 일반 컬럼 조건. "전체 노드 스캔" 필요 → 거부
SELECT * FROM payment_by_user WHERE bucket = '2026-06';
--   → 복합 파티션 키의 일부만 지정. 파티션 못 찾음 → 거부
SELECT * FROM payment_by_user WHERE user_id='A' AND bucket='2026-06'
  ORDER BY amount;
--   → 클러스터링 순서가 아닌 컬럼으로 정렬 → 거부
```

핵심 규칙을 표로 정리하면 다음과 같다.

| 절 | 규칙 | 이유 |
|---|---|---|
| `WHERE` 파티션 키 | 모든 파티션 키 컬럼을 **등호**(또는 `IN`)로 | 토큰을 계산해야 노드를 찾음 |
| `WHERE` 클러스터링 키 | 앞에서부터 순서대로. 마지막 하나만 범위(`<`,`>`) | 디스크 정렬 순서를 따라 슬라이스 |
| `ORDER BY` | 클러스터링 순서이거나 그 완전 역순만 | 디스크에 정렬된 것을 읽거나 거꾸로 읽을 뿐 |
| 일반 컬럼 조건 | 기본 거부. `ALLOW FILTERING` 필요 | 파티션 못 좁히면 풀스캔 |

`ALLOW FILTERING`은 "느려도 괜찮으니 전체를 훑어라"라고 명시적으로 허락하는 예외 수단이다. 이름 자체가 위험을 알리는 표시이다.

> [!warning] 함정: `ALLOW FILTERING`은 프로덕션에서 사실상 금지다
> 이 키워드를 쓰면 코디네이터가 조건에 맞는 row를 찾으려고 여러 파티션(최악의 경우 전체)을 스캔한다. 데이터가 적은 개발용이나 관리용 쿼리가 아니라면 쓰지 않아야 한다. "쿼리가 거부돼서 `ALLOW FILTERING`을 붙였더니 실행됐다"는 상황은 거의 항상 **데이터 모델이 그 쿼리를 지원하지 못한다는 신호**이다. 올바른 해결책은 그 쿼리를 위한 테이블을 새로 만드는 것이다(5장 Query First).

세컨더리 인덱스(`CREATE INDEX`)와 SAI(Storage-Attached Index, 5.0에서 강화된 인덱스)는 이 제약을 부분적으로 완화하지만, 만능이 아니며 나름의 함정이 있다. 이 내용은 6장에서 다룬다.

---

## 페이징: 1억 행을 메모리 부족 없이 읽기

`SELECT * FROM huge_table`이 1억 행을 반환한다고 하자. RDB라면 커서나 LIMIT으로 처리하겠지만, Cassandra는 코디네이터와 드라이버 양쪽의 메모리를 보호하려고 **자동 페이징(automatic paging)** 을 기본으로 제공한다. 이 절에서는 자동 페이징이 어떻게 동작하는지 살펴본다.

### 드라이버 자동 페이징: fetch size와 paging state

드라이버는 `SELECT` 결과를 한 번에 모두 받지 않는다. **fetch size**(기본 5000행)만큼만 받고, 다음 페이지가 필요하면 서버에 **paging state**라는 불투명(opaque) 토큰을 보내 이어서 달라고 요청한다.

```text
  자동 페이징의 흐름

  앱: SELECT * FROM events WHERE stream_id='s1'   (fetch_size=5000)
        │
        ▼
  코디네이터: 처음 5000행 + paging_state(=마지막 위치 인코딩) 반환
        │
        ▼
  앱: 5000행 처리... 더 필요 → 같은 쿼리 + paging_state 재전송
        │
        ▼
  코디네이터: paging_state가 가리킨 지점부터 다음 5000행 + 새 paging_state
        │
       ... 반복 ...
        ▼
  paging_state == null 이면 끝
```

`paging_state`는 마지막으로 읽은 row의 위치(파티션과 클러스터링 좌표 등)를 서버가 인코딩한 토큰이다. 드라이버는 이 토큰을 다음 요청에 담아 보낼 뿐, 내용을 해석하지 않는다. Java나 Python 같은 대부분의 드라이버에서는 결과 이터레이터를 끝까지 순회하기만 하면 페이징이 **투명하게** 처리된다. 다음은 Java 드라이버의 전형적인 사용 예이다.

```java
// fetch size 설정 (기본 5000)
SimpleStatement stmt = SimpleStatement.builder("SELECT * FROM events WHERE stream_id=?")
    .addPositionalValue("s1")
    .setPageSize(2000)            // 페이지당 2000행
    .build();

ResultSet rs = session.execute(stmt);
for (Row row : rs) {             // 이터레이터를 돌리면 페이지 경계에서
    process(row);                // 드라이버가 알아서 다음 페이지를 가져옴
}

// 웹 페이지네이션처럼 '상태를 저장했다가 나중에 이어받기'도 가능
ByteBuffer state = rs.getExecutionInfo().getPagingState();
// state를 클라이언트/세션에 저장 → 다음 요청에 setPagingState(state)로 복원
```

상태를 저장했다가 이어받는 방식은 무한 스크롤 UI를 만들 때 유용하다. 다만 `paging_state`는 **특정 쿼리와 당시의 클러스터 상태에 묶인 토큰**이므로, 다른 쿼리에 재사용하거나 북마크처럼 영구 저장하면 안 된다. 짧은 시간 동안 연속으로 페이징할 때만 쓴다.

> [!warning] 함정: 페이징 순서는 정렬된 전역 순서가 아니다
> 자동 페이징은 한 파티션 안에서는 클러스터링 순서를 보장한다. 하지만 여러 파티션을 훑는 풀스캔(`SELECT * FROM t`)에서는 토큰 순서로 읽을 뿐, 비즈니스 관점의 정렬은 전혀 없다. 최신순으로 페이징하려면 그 정렬이 클러스터링 키로 설계되어 있어야 한다.

### token() 함수: 파티션을 직접 훑는 수동 페이징

자동 페이징으로 여러 파티션에 걸쳐 전체 테이블을 순회할 수도 있다. 하지만 대규모 데이터 마이그레이션이나 풀스캔에서는 **`token()` 함수로 파티션을 직접 나누어 처리**하는 방식이 더 효과적이다.

`token()`은 파티션 키를 2장에서 설명한 토큰(Murmur3 해시, 64비트 정수)으로 변환하는 함수이다. 데이터는 디스크에 이 토큰 순서로 저장되므로, 토큰 범위를 슬라이스하면 테이블을 결정적인 방식으로 나눌 수 있다.

```cql
-- 토큰 공간 전체: -2^63 ~ 2^63-1
-- 가장 작은 토큰부터 시작
SELECT token(user_id), user_id, ...
FROM accounts
WHERE token(user_id) >= -9223372036854775808
LIMIT 1000;

-- 위 결과의 마지막 token 값을 T라 하면, 다음 청크:
SELECT token(user_id), user_id, ...
FROM accounts
WHERE token(user_id) > T
LIMIT 1000;
-- ... 토큰이 2^63-1에 닿을 때까지 반복
```

이 패턴의 강점은 **병렬화**에 있다. 토큰 공간을 N등분해 워커 N개에 나눠 주면, 각 워커가 겹치지 않는 토큰 구간을 독립적으로 스캔한다. 토큰 링과 워커의 관계를 그림으로 보면 다음과 같다.

```text
  토큰 공간 [-2^63, 2^63) 을 4개 워커로 분할 스캔

  -2^63        -2^61          0          +2^61        +2^63
    │────worker1────│────worker2────│────worker3────│────worker4────│
    token(pk)>=lo AND token(pk)<hi 로 각 워커가 자기 구간만 담당
    → 서로 다른 노드 범위를 병렬로 훑음 → 풀 클러스터 처리량 활용
```

`token()` 페이징의 결정적인 장점은 **재시작 가능성(resumability)** 이다. paging_state는 휘발성이지만, 마지막으로 처리한 토큰 T는 단순한 64비트 정수이다. 이 값을 어딘가에 기록해 두면 작업이 중단되어도 `token(pk) > T`로 정확히 그 지점부터 다시 시작할 수 있다. 대규모 백필(backfill), 마이그레이션, 전체 리포팅 작업에서 표준으로 쓰는 패턴이다.

> [!note] 참고
> `WHERE token(pk) > T`로 슬라이스할 때 파티션 키가 복합 키라면 `token(pk1, pk2)`처럼 파티션 키 전체를 함께 넘겨야 한다. 토큰은 파티션 키 전체에 대해 계산되기 때문이다. 또한 드물지만 해시 충돌 등으로 같은 토큰에 여러 파티션 키가 매핑될 수 있으므로, 경계에서 `>`와 `>=` 중 무엇을 쓸지와 중복 처리에 주의해야 한다.

DataStax의 `dsbulk` 같은 전용 로딩 도구나 Spark-Cassandra 커넥터도 내부적으로 이 `token()` 분할 방식을 사용해 클러스터 전체를 병렬로 읽고 쓴다. 풀스캔 작업을 직접 작성한다면 이 패턴을 따르는 것이 정석이다.

---

## TTL: 시간이 지나면 저절로 만료되는 데이터

Cassandra는 **셀(cell) 단위로 만료 시간(time-to-live)** 을 지정할 수 있다. 지정한 초가 지나면 그 셀은 자동으로 만료되어 사라진다. 세션 데이터나 캐시, 일정 기간만 보관하는 이벤트 로그에 잘 맞는다. 하지만 TTL의 내부 동작을 모르면 또 다른 tombstone 함정에 빠진다.

### per-column TTL과 default_time_to_live

TTL은 세 가지 수준에서 지정할 수 있다.

```cql
-- 1) 이 INSERT/UPDATE에만 적용되는 TTL (초 단위). 86400 = 1일
INSERT INTO sessions (id, token) VALUES ('s1', 'abc') USING TTL 86400;
UPDATE sessions USING TTL 3600 SET token = 'xyz' WHERE id = 's1';

-- 2) 테이블 기본 TTL — 모든 쓰기에 자동 적용
CREATE TABLE sessions (...) WITH default_time_to_live = 86400;

-- 3) TTL 제거(영구 보관으로 전환) — USING TTL 0
UPDATE sessions USING TTL 0 SET token = 'permanent' WHERE id = 's1';
```

중요한 세부 사항이 하나 있다. **TTL은 컬럼(셀)마다 독립적**이므로, 한 row 안에서도 컬럼마다 TTL이 다를 수 있다. 예를 들어 `INSERT ... USING TTL 100`으로 모든 컬럼에 100초를 걸고, 나중에 `UPDATE ... USING TTL 50 SET status=...`로 `status`만 50초로 갱신하면 `status`는 50초 후에, 나머지 컬럼은 100초 후에 만료된다. row가 통째로 사라지는 것이 아니라 셀마다 각자의 시간에 맞춰 만료된다.

### TTL 만료는 곧 tombstone이다: 9장과의 연결

TTL에서 가장 중요한 사실은 다음과 같다.

> **TTL이 만료되어도 데이터가 즉시 디스크에서 사라지지 않는다. 만료된 셀은 tombstone으로 변한다.** 그리고 그 tombstone은 `gc_grace_seconds`가 지나고 compaction이 와야 비로소 정리된다.

```text
  USING TTL 86400 으로 쓴 셀의 일생

  t=0           셀 작성 (살아있음, 만료 예정 시각 = t+86400 기록됨)
  t=86400       만료 시점 도달 → 읽으면 '없음'으로 보임
                하지만 디스크에는 expired cell(=tombstone)로 남아있음
  t=86400+gc_grace  compaction이 오면 비로소 물리적으로 제거
```

이 메커니즘 때문에 생기는 함정은 9장(Compaction 전략)의 핵심 주제와 직결된다. **TTL 데이터를 STCS로 다루면 만료된 tombstone과 살아 있는 데이터가 한 SSTable에 섞이므로, 만료된 부분을 정리하려고 살아 있는 데이터까지 반복해서 다시 쓰게 된다.** 디스크 사용량은 줄지 않고, 읽기는 tombstone을 훑느라 느려진다.

해결책이 바로 `TimeWindowCompactionStrategy`(TWCS)이다. TWCS는 같은 시간 창에 쓰인 데이터를 같은 SSTable 묶음으로 분리한다. 그 창의 모든 데이터가 비슷한 시기에 TTL로 만료되면 **SSTable 파일을 통째로 drop**하면 된다. 셀 단위로 tombstone을 정리할 필요가 없다.

```text
  TWCS + TTL 의 우아함

  [6/24 창 SSTable] [6/25 창] [6/26 창] [6/27 창] ...
        │                                              
   TTL로 6/24 데이터 전부 만료 →  파일 통째로 삭제(drop). 끝.
   (셀 하나하나 tombstone 스캔/재작성 불필요)
```

> [!note] 실전 규칙
> "시간순으로 쌓이고 일정 기간이 지나면 만료되는" 데이터(이벤트 로그, 세션, 시계열)에는 **TWCS + default_time_to_live** 조합이 가장 적합한 경우가 많다. 결제 이벤트 보관 정책(예: 13개월 후 만료)을 이 조합으로 구현하면 디스크 공간이 자동으로 회수된다.

### WRITETIME()와 TTL(): 셀의 메타데이터 확인하기

모든 셀은 값 외에 두 가지 메타데이터를 가진다. 하나는 **쓰기 timestamp**(마이크로초 단위, 충돌 해결의 기준)이고, 다른 하나는 **남은 TTL**이다. 이 값들을 조회하는 함수가 있다.

```cql
-- amount 컬럼이 언제 쓰였는지(epoch 마이크로초), 남은 TTL은 몇 초인지
SELECT amount,
       WRITETIME(amount) AS written_us,
       TTL(amount) AS ttl_remaining
FROM payments WHERE id = 'p1';
```

`WRITETIME()`은 디버깅과 데이터 포렌식에 특히 유용하다. 어떤 값이 정확히 언제 마지막으로 갱신됐는지 알 수 있고, 충돌 해결(last-write-wins)에서 어느 쓰기가 이겼는지 추적할 수 있다. `TTL()`로는 이 세션이 앞으로 몇 초 더 유지되는지를 확인할 수 있다. 두 함수 모두 **단일 셀(일반 컬럼)에만** 적용된다. 파티션 키나 클러스터링 키에는 적용할 수 없는데, 키는 별도의 셀이 아니라 row의 좌표이기 때문이다.

> [!warning] 함정: WRITETIME과 USING TIMESTAMP의 위험한 결합
> 쓰기 timestamp는 충돌 해결의 절대적인 기준이다. 곧 살펴볼 `USING TIMESTAMP`로 이 값을 인위적으로 조작하면, 과거 timestamp로 쓴 값이 최신 값에 가려 영원히 보이지 않는 좀비나 유령 같은 현상이 생긴다. WRITETIME으로 그 흔적을 추적할 수는 있지만, 처음부터 timestamp를 건드리지 않는 것이 가장 좋다.

---

## 고급 DML: LWT, BATCH, prepared, USING, CONSISTENCY, JSON

마지막으로 실무에서 쓰는 CQL의 주요 기능을 살펴본다. 각 기능의 자세한 내부 동작은 [[11 - LWT Batch Counter 내부]]에서 다루고, 여기서는 문법과 각 기능을 왜 조심해서 써야 하는지를 분명히 해 둔다.

### 경량 트랜잭션(LWT): IF NOT EXISTS / IF 조건

앞에서 INSERT는 존재 여부를 검사하지 않는다고 했다. 정말로 "없을 때만 삽입"하거나 "현재 상태가 X일 때만 갱신"해야 할 때 쓰는 것이 **경량 트랜잭션(lightweight transaction, LWT)** 이다. `IF` 절을 붙이면 된다.

```cql
-- 없을 때만 삽입 (진짜 멱등 보장). 결과로 [applied]=true/false 반환
INSERT INTO payments (id, amount, status) VALUES ('p1', 5000, 'paid')
IF NOT EXISTS;

-- 현재 status가 'pending'일 때만 'paid'로 (compare-and-set)
UPDATE payments SET status = 'paid'
WHERE id = 'p1'
IF status = 'pending';

-- row가 존재할 때만 삭제
DELETE FROM payments WHERE id = 'p1' IF EXISTS;
```

`IF NOT EXISTS`는 결제 ID 중복을 막는 데, `IF status='pending'`은 이미 처리된 결제를 두 번 처리하지 않는 데 쓰인다. 즉 LWT는 **상태 기계 전이의 원자성**을 보장하며, RDB의 유니크 제약이나 낙관적 락에 해당하는 도구이다.

그런데 왜 "경량(lightweight)"이라고 부를까? 사실 이 이름은 실제와 반대이다. LWT는 내부적으로 **Paxos 합의 프로토콜**을 실행해 복제본들 사이에서 "지금 이 조건이 참인가"에 대해 합의한다. 그래서 일반 쓰기보다 **네트워크 왕복이 약 4배**(prepare → promise → propose → accept, 그리고 commit) 더 필요하다.

```text
  일반 쓰기 vs LWT 비용

  일반 INSERT:   코디네이터 → 레플리카들 (1 라운드트립)
  LWT (IF ...):  Paxos 합의 → 4 단계(왕복 다수) → CAS read → 적용
                 → 지연 수배~10배, 처리량 급감
```

> [!warning] 함정: LWT 남용
> `IF NOT EXISTS`를 모든 INSERT에 습관적으로 붙이면 클러스터 처리량이 크게 떨어진다. LWT는 정말로 선형성(linearizability)이 필요한 소수의 중요한 경로(중복 결제 방지, 유일 ID 발급, 상태 전이)에만 써야 한다. 또한 LWT가 같은 파티션에 몰리면 Paxos 경합 때문에 더 느려진다. 자세한 Paxos 동작과 격리 수준의 한계(LWT는 ACID 트랜잭션이 아니다)는 11장을 참고한다.

### BATCH: 원자성을 보장하지만 성능 최적화 수단은 아니다

`BATCH`는 여러 DML을 한 묶음으로 보낸다. RDB의 트랜잭션처럼 보이지만, **BATCH에 대한 가장 흔한 오해는 성능을 위한 기능이라는 생각**이다.

```cql
BEGIN BATCH
  INSERT INTO payment_by_id (id, user_id, amount) VALUES ('p1','A',5000);
  INSERT INTO payment_by_user (user_id, bucket, paid_at, id, amount)
    VALUES ('A','2026-06','2026-06-26 09:00','p1',5000);
APPLY BATCH;
```

CQL BATCH의 실제 목적은 **원자성**이다. 위 예처럼 같은 결제를 비정규화된 두 테이블에 동시에 쓸 때, 둘 다 성공하거나 둘 다 실패하도록 보장한다(5장에서 다룬 비정규화 테이블 동기화). RDB 트랜잭션과 다른 점은 다음과 같다.

- **격리(isolation)는 단일 파티션 안에서만** 보장된다. 여러 파티션에 걸친 BATCH에서는 다른 읽기가 절반만 적용된 중간 상태를 볼 수 있다.
- **롤백이 없다.** 코디네이터가 batchlog에 기록한 뒤 재시도해서 결국 모두 적용되도록 보장할 뿐, 실패했을 때 되돌리지는 않는다.

> [!warning] 함정: 여러 파티션을 묶는 BATCH로 성능 높이기
> 서로 다른 파티션의 쓰기를 BATCH로 묶으면 빨라질 것이라고 기대하지만, 실제로는 정반대이다. 코디네이터는 batchlog를 먼저 두 노드에 기록(durability)한 뒤 흩어진 파티션들로 쓰기를 보내야 하므로, **오히려 느려지고 코디네이터에 부하가 몰린다.** 성능이 목적이라면 개별 쓰기를 비동기로 병렬 실행하는 편이 빠르다. BATCH는 같은 파티션 묶음의 원자성이나 비정규화 테이블 동기화에만 써야 한다. 자세한 내용은 11장을 참고한다.

### Counter: 분산 증감 카운터

`counter`는 특수한 타입이다. 분산 환경에서 안전하게 값을 늘리고 줄이는 카운터를 위한 별도 타입이며, 일반 컬럼과 같은 테이블에 섞어 쓸 수 없다(PK와 counter 컬럼만 둘 수 있다).

```cql
CREATE TABLE page_views (
    page_id text PRIMARY KEY,
    views counter
);
UPDATE page_views SET views = views + 1 WHERE page_id = 'home';
```

counter는 `INSERT`로 쓸 수 없고, `UPDATE`의 증감 연산(`+`/`-`)으로만 다룬다. 또한 **멱등하지도 않다**(같은 `+1`을 재시도하면 두 번 더해질 수 있다). 내부적으로는 복제본별 부분합을 합산하는 복잡한 메커니즘을 쓴다. 그래서 정확해야 하는 회계(돈 계산)에는 적합하지 않다. 자세한 내용은 11장을 참고한다.

### Prepared Statement: 거의 항상 써야 하는 기능

`prepared statement`는 쿼리 문자열을 서버에 한 번 등록(파싱과 메타데이터 캐시)해 두고, 이후에는 바인딩 값만 보내는 방식이다. 성능과 보안 모두에서 거의 항상 올바른 선택이다.

```java
// 한 번 prepare (드라이버가 query id를 캐시)
PreparedStatement ps = session.prepare(
    "INSERT INTO payments (id, amount, status) VALUES (?, ?, ?)");

// 이후엔 바인딩만 — 파싱 생략, 토큰 인지 라우팅, SQL 인젝션 차단
session.execute(ps.bind("p1", 5000L, "paid"));
session.execute(ps.bind("p2", 3000L, "pending"));
```

이점은 세 가지이다.

1. 매번 쿼리를 파싱하지 않아도 되므로 빠르다.
2. 드라이버가 파티션 키의 위치를 알기 때문에 **토큰 인지 라우팅**(token-aware routing)으로 코디네이터를 거치지 않고 데이터를 가진 노드에 직접 요청을 보낸다.
3. 값을 바인딩으로 전달하므로 **CQL 인젝션이 원천 차단**된다.

문자열을 이어 붙여 쿼리를 만드는 방식은 피하고, 반복해서 실행하는 쿼리는 prepare해서 쓴다.

### USING TIMESTAMP / USING TTL: 정밀하지만 위험한 도구

Cassandra는 쓰기마다 충돌 해결에 쓰는 timestamp(마이크로초)를 자동으로 부여한다. `USING TIMESTAMP`를 쓰면 이 값을 직접 지정할 수 있다.

```cql
INSERT INTO payments (id, status) VALUES ('p1', 'paid')
USING TIMESTAMP 1782000000000000 AND TTL 3600;
```

이 기능은 데이터 마이그레이션(원본 시각 보존)이나 특정한 충돌 해결 상황에서 유용하지만, 매우 위험하다.

> [!warning] 함정: USING TIMESTAMP가 만드는 유령 데이터
> 충돌을 해결할 때는 timestamp가 큰 쪽이 무조건 이긴다(last-write-wins). 미래 timestamp로 값을 한 번 써 버리면, 이후 현재 시각 timestamp로 하는 모든 정상 쓰기가 그 미래 값에 가려 **영원히 적용되지 않는다.** "분명히 UPDATE했는데 값이 바뀌지 않는다"는 이상한 현상의 흔한 원인이다.
>
> DELETE의 tombstone도 timestamp를 가지므로, 과거 timestamp로 INSERT하면 더 큰 timestamp를 가진 tombstone에 가려 데이터가 보이지 않는 일도 생긴다. **꼭 필요한 경우가 아니면 timestamp를 건드리지 않아야 한다.**

### CONSISTENCY: 읽기와 쓰기마다 일관성 수준을 고른다

CQL에서는 문장마다 일관성 레벨(consistency level)을 고를 수 있다. 4장(Tunable Consistency)의 핵심 내용이다.

```cql
-- cqlsh 세션 레벨로 설정
CONSISTENCY QUORUM;        -- 이후 쿼리는 과반 복제본 응답 대기
CONSISTENCY LOCAL_QUORUM;  -- 로컬 DC의 과반만 (멀티 DC 표준)
CONSISTENCY ONE;           -- 한 복제본만 응답하면 OK (빠름, 약함)
```

드라이버에서는 statement 단위로 지정한다. 핵심 공식은 `R + W > RF`(읽기 정족수 + 쓰기 정족수 > 복제 계수)이며, 이 조건을 만족하면 강한 일관성(strong consistency)이 보장된다. 결제 시스템은 보통 쓰기와 읽기에 모두 `LOCAL_QUORUM`을 써서, 단일 DC 안에서 `R+W>RF`를 만족하면서 DC 간 지연도 피한다. ONE/QUORUM/ALL/EACH_QUORUM, 힌티드 핸드오프, read repair 같은 자세한 트레이드오프는 4장을 참고한다.

### JSON 지원: INSERT JSON / SELECT JSON / fromJson / toJson

CQL은 row를 JSON으로 주고받는 문법을 제공한다. API 경계에서 편리하게 쓸 수 있다.

```cql
-- JSON 한 덩어리로 INSERT (키 이름 = 컬럼명)
INSERT INTO payments JSON '{"id":"p1","amount":5000,"status":"paid"}';

-- 결과를 JSON으로 받기 (각 row가 [json] 컬럼 하나로)
SELECT JSON id, amount, status FROM payments WHERE id = 'p1';
-- 반환: {"id": "p1", "amount": 5000, "status": "paid"}

-- 개별 컬럼 수준 변환 함수
INSERT INTO payments (id, attributes) VALUES ('p1', fromJson('{"k":"v"}'));
SELECT id, toJson(amount) FROM payments WHERE id = 'p1';
```

알아 둘 점이 두 가지 있다. 첫째, `INSERT JSON`도 일반 `INSERT`와 마찬가지로 **upsert**이며, JSON에 없는 컬럼은 기본적으로 null로 처리된다. `DEFAULT UNSET`을 지정하면 누락된 컬럼을 건드리지 않는다.

둘째, JSON은 편의 기능일 뿐 **타입 직렬화 방식은 동일**하다. 다만 JSON 텍스트 표현 규칙이 따로 있으므로 API 계약에서 주의해야 한다.

`timestamp`는 ISO-8601과 비슷한 문자열(`"2026-06-26 00:00:00.000Z"`, 공백으로 구분)로, `blob`은 CQL 리터럴과 같은 16진수 문자열(`"0x..."`, base64 아님)로 출력된다. `decimal`과 `varint`는 JSON 숫자로 출력되지만, 클라이언트 JSON 파서의 정밀도 손실을 피할 수 있도록 입력할 때는 따옴표로 감싼 문자열도 받아 준다. 성능이 중요한 경로에서는 JSON 파싱 오버헤드를 고려해야 한다.

---

## RDB와 Cassandra 비교: 한눈에 보기

이 장의 내용을 RDB와 대조해 한 표로 정리하면 다음과 같다.

| 상황 | RDB(SQL) | Cassandra(CQL) |
|---|---|---|
| `INSERT` 중복 키 | PK 위반 에러 | 조용히 덮어씀(upsert) |
| `UPDATE` 없는 row | 0 rows affected | 새 row 생성(upsert) |
| `DELETE` | 데이터 즉시 제거 | tombstone append, 10일 뒤 정리 |
| 임의 컬럼 `WHERE` | 인덱스/풀스캔 자동 | 거부(또는 `ALLOW FILTERING`) |
| `ORDER BY 아무_컬럼` | 런타임 정렬 | 클러스터링 순서/역순만 |
| "없을 때만 삽입" | `INSERT ... ON CONFLICT` | `IF NOT EXISTS`(Paxos, 비쌈) |
| 여러 row 트랜잭션 | ACID 트랜잭션 | BATCH(단일 파티션 원자성만) |
| 큐/작업 테이블 | 흔한 패턴 | 안티패턴(tombstone hell) |
| 스키마 변경 | PK도 변경 가능(고비용) | 비-PK만. PK는 재설계 |

이 표의 모든 행은 한 가지 사실에서 나온다. **CQL은 LSM-tree 위에 구축된 분산 append-only 스토리지를 다루는 언어**라는 것이다. SQL의 형태를 빌렸을 뿐, 내부의 물리적 동작은 완전히 다르다. 문법이 비슷해 보일수록 "이 문장이 디스크와 토큰 링에서 실제로 무슨 일을 하는가"를 떠올려야 이 언어를 제대로 쓸 수 있다.
