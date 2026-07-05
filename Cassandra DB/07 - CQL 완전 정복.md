---
title: CQL 완전 정복 - DDL/DML, 타입, 페이징, TTL
date: 2026-06-26
tags: [cassandra, cql, 학습노트]
---

CQL(Cassandra Query Language)은 SQL과 놀랍도록 비슷하게 생겼다. `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`이 모두 있고, `WHERE` 절도 있고, 데이터 타입도 친숙하다. 이 친숙함이 바로 함정이다. CQL은 의도적으로 SQL의 외형을 빌려 학습 곡선을 낮췄지만, 그 아래에서 일어나는 일은 RDB와 근본적으로 다르다. 같은 키워드가 전혀 다른 의미이거나, RDB라면 당연히 됐을 쿼리가 거부되거나, 성공한 쿼리가 조용히 데이터를 덮어쓴다.

이 장이 답하려는 핵심 질문은 다음과 같다.

- `CREATE KEYSPACE`와 `CREATE TABLE`의 `WITH` 옵션들은 각각 디스크와 클러스터에서 무슨 일을 결정하는가? 왜 이 옵션들이 운영의 절반을 좌우하는가?
- CQL 데이터 타입은 내부적으로 어떻게 바이트로 직렬화되며, `uuid`와 `timeuuid`, `timestamp`와 `date`는 왜 구분해서 써야 하는가?
- "INSERT는 삽입, UPDATE는 갱신"이라는 SQL의 직관은 왜 CQL에서 완전히 무너지는가? 둘 다 upsert라는 말의 진짜 의미는?
- `DELETE`는 왜 데이터를 지우지 않고 tombstone을 만드는가? TTL 만료가 왜 같은 문제를 일으키는가?
- 드라이버는 1억 행을 어떻게 메모리 폭발 없이 페이징하는가? `token()` 함수로 직접 파티션을 훑는다는 것은 무슨 의미인가?
- `IF NOT EXISTS` 한 줄이 왜 쿼리 지연을 10배로 만드는가? `USING TIMESTAMP`는 왜 위험한 무기인가?

[[05 - 데이터 모델링 1 - Query First]]에서 우리는 "쿼리를 먼저 정하고 테이블을 설계한다"는 철학을, [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]에서 컬렉션·UDT의 함정을 다뤘다. 이 장은 그 설계를 실제 CQL 문법으로 옮기면서, 각 문장이 [[08 - Storage Engine 내부]]의 SSTable과 [[10 - Write Path와 Read Path]]의 경로 위에서 실제로 무엇을 하는지 끝까지 파고든다.

---

## CQL이라는 언어의 정체

CQL을 제대로 이해하려면 먼저 "이것이 무엇이 아닌가"를 분명히 해야 한다.

CQL은 **선언적 질의 언어처럼 보이지만, 실제로는 스토리지 엔진의 얇은 명령 인터페이스**에 가깝다. SQL에서는 옵티마이저가 "이 쿼리를 어떻게 실행할지"를 자유롭게 결정한다. 같은 결과를 내는 한, 인덱스를 쓰든 풀스캔을 하든 조인 순서를 바꾸든 엔진 마음이다. CQL에는 그런 자유가 거의 없다. **거의 모든 CQL 쿼리는 "파티션 키로 노드를 찾아 → 그 파티션의 정렬된 row들을 순차 읽기"라는 단 하나의 실행 계획으로 환원된다.** 옵티마이저가 마법을 부릴 여지가 없도록 설계되었고 그래서 마법을 부릴 수 없는 쿼리는 아예 문법 수준에서 거부된다.

이 차이를 한 문장으로 줄이면 이렇다.

> RDB에서 SQL은 "무엇을 원하는가"를 말하고 엔진이 "어떻게"를 푼다.
> Cassandra에서 CQL은 "어떻게 저장됐는지"를 이미 알고 있는 사람이 "그 구조를 따라 읽어라"라고 명령하는 것에 가깝다.

그래서 CQL을 배우는 것은 문법을 외우는 일이 아니다. 각 문장이 [[08 - Storage Engine 내부]]의 LSM-tree, [[02 - 분산 아키텍처]]의 토큰 링, [[04 - Tunable Consistency]]의 정족수(quorum) 위에서 어떻게 풀리는지를 머릿속에 그리는 일이다. 이 장은 그 그림을 그리는 데 집중한다.

CQL을 실행하는 표준 도구는 `cqlsh`다. Python 기반 셸로, 아래처럼 접속한다.

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

`cqlsh`의 `TRACING ON`은 학습에 특히 강력하다. 쿼리 하나를 던지면 어느 노드가 코디네이터가 됐고, 어느 레플리카에 요청이 갔고, 각 단계가 몇 마이크로초 걸렸는지를 보여준다. 이 장에서 설명하는 모든 "내부에서 무슨 일이 일어나는가"를 직접 눈으로 확인할 수 있는 창구다.

---

## Keyspace DDL: 복제의 경계선을 긋는다

CQL의 최상위 컨테이너는 **keyspace**다. RDB의 "데이터베이스(스키마)"에 대응하지만, keyspace가 결정하는 것은 단순한 네임스페이스가 아니다. **keyspace는 "이 안의 데이터를 어떻게, 몇 벌, 어느 데이터센터에 복제할 것인가"라는 복제 정책의 경계선**이다. 이것이 keyspace의 존재 이유다.

```cql
CREATE KEYSPACE payments
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'seoul': 3,
  'tokyo': 3
}
AND durable_writes = true;
```

### replication: 데이터를 몇 벌, 어디에

`replication` 맵은 [[03 - 복제 전략과 데이터센터]]에서 깊이 다룬 복제 전략(replication strategy)과 복제 계수(replication factor)를 선언한다. 두 가지 전략이 있다.

| 전략                              | 의미                                  | 사용 시점                  |
| ------------------------------- | ----------------------------------- | ---------------------- |
| `SimpleStrategy`                | 데이터센터/랙 토폴로지를 무시하고 링에서 다음 N개 노드에 복제 | 단일 DC, 학습/개발용 **only** |
| `NetworkTopologyStrategy` (NTS) | 데이터센터별로 복제 계수를 독립 지정. 랙을 인식해 분산     | 프로덕션 **전부**            |

> **함정**: `SimpleStrategy`로 프로덕션 keyspace를 만들면 나중에 데이터센터를 추가할 때 복제 토폴로지가 망가진다. SimpleStrategy는 랙(rack)을 인식하지 않아 같은 랙에 복제본이 몰릴 수 있고, 멀티 DC 확장 경로가 막힌다. **프로덕션은 단일 DC라도 처음부터 `NetworkTopologyStrategy`로 시작하라.** 이름만 `'class': 'NetworkTopologyStrategy', 'datacenter1': 3`로 써도 된다.

`NetworkTopologyStrategy`에서 `'seoul': 3`이 정확히 무엇을 뜻하는지 짚자. 이것은 "seoul 데이터센터 안에서 각 파티션을 3개의 노드에 복제한다"는 뜻이다. NTS는 한 걸음 더 나아가, 그 3개의 복제본을 **서로 다른 랙(rack)에 배치하려 시도**한다. 랙 하나가 통째로 죽어도(전원/스위치 장애) 데이터를 잃지 않게 하기 위해서다. 토큰 링 위에서 NTS가 복제본을 고르는 과정을 그리면 이렇다.

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

`durable_writes`는 기본값 `true`이며 거의 항상 그대로 두어야 한다. 이 옵션은 [[10 - Write Path와 Read Path]]의 쓰기 경로와 직접 연결된다.

쓰기가 노드에 도착하면 정상적으로는 두 곳에 동시에 기록된다. (1) 디스크의 **commit log**(append-only, 내구성 보장), (2) 메모리의 **memtable**(빠른 조회). `durable_writes = false`로 설정하면 **commit log 기록을 건너뛴다.** 쓰기는 memtable에만 들어가고 memtable이 SSTable로 flush되기 전에 노드가 죽으면 그 데이터는 영원히 사라진다.

엄밀히 하자면, `durable_writes = true`라도 commit log가 매 쓰기마다 디스크에 `fsync`되는 것은 아니다. 기본 동기화 모드는 **periodic**(기본 `commitlog_sync_period` 10초)이라, 쓰기는 commit log 버퍼에 적히는 즉시 ACK되고 실제 fsync는 주기적으로 일어난다. 즉 전원 장애 시 최후 수초의 쓰기는 유실될 수 있다(매 쓰기 fsync가 필요하면 `commitlog_sync: batch` 또는 `group`을 쓰되 지연이 늘어난다). 이 sync 모드의 트레이드오프는 [[10 - Write Path와 Read Path]]에서 다룬다.

```text
  durable_writes = true (기본, 안전)
    write ──┬──► commit log (fsync, 디스크) ──► 노드 죽어도 복구 가능
            └──► memtable (메모리)

  durable_writes = false (위험)
    write ──────► memtable (메모리)  ── flush 전 노드 다운 시 영구 소실
                  (commit log 생략 → 쓰기 약간 빠름)
```

> **함정**: `durable_writes = false`는 "어차피 다른 DC에 복제되니 한 DC 내구성은 포기해도 된다"는 특수한 시나리오에서만 고려된다. 일반 결제/금융 데이터에서는 절대 끄지 마라. 약간의 쓰기 성능을 위해 데이터 영속성을 포기하는 거래는 결제 시스템에서 성립하지 않는다.

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

`ALTER KEYSPACE`로 RF를 올리는 순간 기존 데이터가 마법처럼 새 복제본으로 복사되지는 **않는다.** Cassandra는 메타데이터(이제부터 이 데이터는 5벌이어야 한다)만 바꾼다. 실제 데이터를 새 복제본에 채우려면 `nodetool repair`를 돌려 anti-entropy를 수행해야 한다. 그 전까지는 새 복제본에 데이터가 없어 `CONSISTENCY ALL` 같은 강한 읽기가 빈 결과를 섞어 반환할 수 있다. RF 변경은 [[12 - 운영과 트러블슈팅]]에서 다루는 신중한 운영 절차다.

---

## Table DDL: PRIMARY KEY가 모든 것을 결정한다

테이블 정의에서 가장 중요한 한 줄은 `PRIMARY KEY`다. RDB에서 PK는 "행을 식별하는 유니크 제약"에 불과하지만, Cassandra에서 **PRIMARY KEY는 데이터가 클러스터 어디에 저장되고(파티션 키), 디스크에서 어떻게 정렬되는지(클러스터링 키)를 동시에 규정하는 물리 설계 그 자체**다. 이 주제는 [[05 - 데이터 모델링 1 - Query First]]의 핵심이었다. 여기서는 CQL 문법의 각 형태가 정확히 무엇을 의미하는지 복습하며 못 박는다.

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

PRIMARY KEY 선언은 괄호 위치에 따라 의미가 완전히 달라진다. 이것이 CQL 초심자가 가장 많이 틀리는 지점이다.

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

**파티션 키(partition key)** 는 [[02 - 분산 아키텍처]]의 토큰 링에서 이 데이터가 어느 노드로 갈지를 결정한다. 파티션 키를 해시(기본 파티셔너는 `Murmur3Partitioner`)한 결과가 토큰이고, 그 토큰이 링의 어느 구간에 속하느냐로 담당 노드(와 복제본들)가 정해진다.

**클러스터링 키(clustering key)** 는 같은 파티션 안에서 row들이 **디스크에 물리적으로 정렬되어 저장되는 순서**를 결정한다. 이게 핵심이다. RDB에서 정렬은 읽을 때 `ORDER BY`로 하는 런타임 연산이지만, Cassandra에서 정렬은 **쓸 때 이미 끝나 있는 디스크 레이아웃**이다.

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

`CLUSTERING ORDER BY (paid_at DESC)`로 정의하면 디스크에 최신순으로 깔리므로 "이 사용자의 최근 결제 20건"이 디스크 헤드 한 번 댄 채 순차로 긁어오는 연산이 된다. 인덱스 탐색도, 정렬 연산도 없다. 이것이 Cassandra의 읽기가 빠른 이유이자 "쓰기 시점에 읽기 패턴을 결정한다"는 Query-First 철학의 물리적 실체다.

### WITH 옵션: 테이블의 런타임 성격을 빚는다

`PRIMARY KEY`가 데이터의 형태를 정한다면, `WITH` 옵션은 그 테이블이 디스크에서 어떻게 살아가는지(compaction, 압축, GC, 캐시, 만료)를 정한다. 운영 성능과 직결되는 핵심 옵션들을 하나씩 본다.

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

위에서 본 디스크 정렬 순서를 명시한다. 한번 정하면 `ALTER`로 못 바꾼다(데이터의 물리 레이아웃 자체이므로). 기본값은 모든 클러스터링 키 `ASC`. 시계열/이벤트성 데이터는 보통 시간 컬럼을 `DESC`로 두어 "최신부터" 읽기를 자연스럽게 만든다.

> **함정**: `CLUSTERING ORDER`를 지정하면 `SELECT`의 `ORDER BY`는 그 정의된 순서이거나 그것의 **완전한 역순**만 허용된다. 임의 컬럼으로 정렬할 수 없다. 디스크에 정렬된 그대로 읽거나, 거꾸로 읽는 것 둘뿐이다. 이것이 [[05 - 데이터 모델링 1 - Query First]]에서 정렬을 설계 단계에서 못 박는 이유다.

#### compaction: SSTable을 어떻게 병합할 것인가

[[09 - Compaction 전략]]의 주제다. memtable이 flush되면 불변(immutable) SSTable이 디스크에 쌓이고 같은 키가 여러 SSTable에 흩어진다. compaction은 이들을 주기적으로 병합해 읽기 효율을 회복하고 tombstone을 청소한다. 전략 선택이 곧 워크로드 적합성이다.

| 전략 | 약어 | 최적 워크로드 | 핵심 동작 |
|---|---|---|---|
| `SizeTieredCompactionStrategy` | STCS | 쓰기 많음, 일반 | 비슷한 크기 SSTable 묶어 병합. 기본값 |
| `LeveledCompactionStrategy` | LCS | 읽기 많음, 갱신 잦음 | 레벨 구조로 키당 SSTable 수 최소화. read 안정적, write 증폭 |
| `TimeWindowCompactionStrategy` | TWCS | 시계열, TTL 데이터 | 시간 창 단위로 SSTable 격리. 만료된 창 통째 삭제 |
| `UnifiedCompactionStrategy` | UCS | 범용(5.0 신규) | STCS/LCS를 파라미터로 통합. 튜닝 유연 |

Cassandra 5.0의 큰 변화 중 하나가 `UnifiedCompactionStrategy`(UCS)의 도입이다. STCS와 LCS의 장점을 하나의 파라미터화된 전략으로 통합해 `scaling_parameters`로 동작을 연속적으로 조절할 수 있다. 다만 기존 테이블의 기본값은 여전히 STCS이며 UCS는 명시적으로 선택해야 한다. 결제/이벤트 로그처럼 "시간순으로 쌓고 일정 기간 후 만료"하는 데이터는 TWCS가 거의 정답에 가깝다. 이유는 TTL 섹션에서 tombstone과 함께 다룬다.

#### compression: 디스크 I/O를 압축으로 거래한다

Cassandra는 SSTable을 **청크(chunk) 단위로 압축**해 저장한다. 기본 압축기는 `LZ4Compressor`(5.0 기준)로 압축률보다 속도를 우선한다. `chunk_length_in_kb`는 압축 단위 청크 크기다.

```text
  chunk_length_in_kb 의 트레이드오프

  작게(예: 4KB)  →  특정 row 읽을 때 압축 해제할 양 적음 → 랜덤 읽기 유리
                    대신 압축률 낮고 청크 메타데이터 늘어남

  크게(예: 64KB) →  압축률 좋고 메타데이터 적음 → 순차 읽기/디스크 절약 유리
                    대신 한 row 읽으려 큰 청크 통째 해제 → 랜덤 읽기 손해
```

5.0 기본 `chunk_length_in_kb`는 16KB(과거 버전의 64KB에서 하향 조정됨). 랜덤 읽기가 많으면 더 줄이고 순차 스캔/디스크 절약이 우선이면 키운다. 압축기 선택지는 `LZ4`(기본, 빠름), `Snappy`, `Deflate`(압축률 높지만 느림), `Zstd`(5.0에서 균형 좋아 인기) 등이 있다.

#### gc_grace_seconds: tombstone의 수명

기본값 **864000초(10일)**. 이 값은 Cassandra에서 가장 미묘하고 위험한 옵션 중 하나다. tombstone(삭제 표식)이 생긴 뒤 `gc_grace_seconds`가 지나야 compaction이 그 tombstone을 영구 제거할 수 있다.

즉시 지우지 않고 10일이나 기다리는 이유는 **삭제가 모든 복제본에 전파될 시간을 벌기 위해서**다. 이 점을 이해하는 게 중요하다.

```text
  "좀비 데이터(zombie)" 시나리오 — gc_grace_seconds가 없다면

  t0: 3개 복제본 모두에 row X 존재
  t1: DELETE X → 복제본 A, B에는 tombstone 도착, C는 네트워크 단절로 누락
  t2: (gc_grace=0 가정) compaction이 A, B의 tombstone 즉시 제거
  t3: C가 복구됨. read repair / 가십 과정에서
      "A,B엔 X가 없는데 C엔 X가 있네?" → C의 X를 A,B로 복제!
  결과: 지웠던 X가 부활(좀비). 삭제가 무효화됨.
```

`gc_grace_seconds = 10일`은 "이 안에 `nodetool repair`를 돌려 tombstone을 모든 복제본에 확실히 전파하라"는 안전 마진이다. 그래서 **운영 규칙도 단순하다. `gc_grace_seconds`보다 짧은 주기로 정기 repair를 돌려라.** repair 주기가 gc_grace보다 길면 좀비가 부활할 수 있다. 이 메커니즘 전체는 [[09 - Compaction 전략]]과 [[12 - 운영과 트러블슈팅]]에서 더 깊이 다룬다.

#### default_time_to_live: 테이블 전역 만료

기본값 0(만료 없음). 0이 아니면 이 테이블에 들어가는 모든 row가 기본적으로 그 초만큼 살고 자동 만료된다. TTL 섹션에서 상세히 본다.

#### caching: 무엇을 메모리에 둘 것인가

```cql
caching = {'keys': 'ALL', 'rows_per_partition': 'NONE'}
```

`keys`는 **key cache**(파티션 키 → SSTable 내 위치 매핑)를 캐시할지, `rows_per_partition`은 **row cache**(실제 row 데이터)를 파티션당 몇 개나 캐시할지를 정한다. 기본값은 `keys: ALL`, `rows_per_partition: NONE`이다.

> **함정**: row cache(`rows_per_partition`)를 켜는 것은 대부분의 경우 안티패턴이다. 쓰기가 한 번이라도 일어나면 해당 파티션의 row cache 전체가 무효화되고, 넓은 파티션을 통째로 메모리에 올려 힙을 잡아먹는다. 거의 변하지 않는 작고 뜨거운(hot) 파티션에만, 신중히. key cache는 켜두는 게 거의 항상 이득이다.

#### ALTER TABLE로 바꿀 수 있는 것과 없는 것

```cql
ALTER TABLE payments.payment_by_user WITH gc_grace_seconds = 432000;   -- OK
ALTER TABLE payments.payment_by_user ADD refund_amount bigint;          -- OK (컬럼 추가)
ALTER TABLE payments.payment_by_user DROP pg_name;                      -- OK (주의)
-- PRIMARY KEY 변경? CLUSTERING ORDER 변경? → 불가. 테이블 재생성+마이그레이션
```

`WITH` 옵션과 비-PK 컬럼 추가/삭제는 자유롭지만, **PRIMARY KEY와 CLUSTERING ORDER는 데이터의 물리 구조이므로 절대 ALTER로 못 바꾼다.** 바꾸려면 새 테이블을 만들고 데이터를 옮겨야 한다. 그래서 [[05 - 데이터 모델링 1 - Query First]]에서 PK를 신중히 설계하라고 그토록 강조했다.

---

## 데이터 타입 총정리

CQL 타입을 RDB 타입처럼 "대충 맞으면 된다"고 다루면 미묘한 버그를 만난다. 각 타입은 **고정된 바이트 직렬화 규칙**을 따르며 그 직렬화가 clustering 정렬 순서, 비교 동작, 저장 크기를 결정한다. 전체 목록을 용도와 함께 정리한다.

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

> 결제 금액은 `bigint`(최소 화폐 단위, 예: 원/센트)로 다루는 것이 정석이다. `varint`는 가변 길이라 정렬·저장이 살짝 무겁고, `int`는 큰 누적 금액에서 오버플로 위험이 있다. **금액에 `float`/`double`을 쓰지 마라**(다음 항목).

### 부동소수점/고정소수점

| 타입 | 크기 | 정밀도 | 용도 |
|---|---|---|---|
| `float` | 4바이트 | IEEE 754 단정밀도 | 근사값 OK인 측정치 |
| `double` | 8바이트 | IEEE 754 배정밀도 | 근사값 OK인 측정치 |
| `decimal` | 가변 | 임의 정밀도 십진 | **돈, 정확한 십진** |

> **함정**: `0.1 + 0.2 != 0.3`. IEEE 754 이진 부동소수점은 십진 소수를 정확히 표현하지 못한다. 환율 계산처럼 십진 정확도가 필요하면 `decimal`을, 정수 단위 금액이면 `bigint`를 써라. `double`로 잔액을 누적하면 시간이 지나며 오차가 쌓인다.

### 그 외 스칼라

| 타입 | 의미 | 직렬화/정렬 특성 |
|---|---|---|
| `boolean` | true/false | 1바이트 |
| `blob` | 임의 바이트열 | 직렬화/역직렬화 없이 그대로. 16진수 리터럴 `0x...` |
| `inet` | IPv4/IPv6 주소 | 4 또는 16바이트로 저장 |
| `uuid` | 임의 버전 UUID(보통 v4 랜덤) | 128비트. 타입은 버전 검증 안 함. `uuid()` 함수가 v4 생성. 정렬 무의미 |
| `timeuuid` | v1(시간 기반) UUID | 128비트. **시간순 정렬됨**. 비교 시 시각이 1차 키 |
| `counter` | 분산 카운터 | 특수 타입. 별도 절에서 |

### 시간 타입 — 가장 헷갈리는 영역

| 타입 | 의미 | 저장 | 예 |
|---|---|---|---|
| `timestamp` | 특정 순간(밀리초) | epoch부터의 ms (8바이트, UTC) | `'2026-06-26 09:00:00+0900'` |
| `date` | 날짜(시각 없음) | epoch 기준 일수(4바이트 부호없음) | `'2026-06-26'` |
| `time` | 하루 중 시각 | 자정부터의 나노초(8바이트) | `'09:00:00.000000000'` |
| `duration` | 기간(달/일/나노초) | 3개 가변 정수 | `12h30m`, `1mo`, `89h4m48s` |

`timestamp`는 내부적으로 **UTC 밀리초 단위 epoch 정수**일 뿐이다. 타임존 정보를 저장하지 않는다. `'2026-06-26 09:00:00+0900'`을 넣으면 `+0900`을 적용해 UTC로 변환한 ms만 저장된다. 읽을 때 `cqlsh`는 클라이언트 타임존으로 표시한다. 그래서 **애플리케이션에서 타임존을 일관되게(보통 UTC) 다루는 규율**이 없으면 표시 혼란이 생긴다.

`duration`은 특히 조심해야 한다. "1달"이 며칠인지는 시작 날짜에 따라 다르므로 `duration`은 **달/일/나노초를 분리 저장**한다. 그래서 두 `duration`을 단순 비교(`>`, `<`)할 수 없고 클러스터링 키로 쓸 수도 없다. 순수한 "구간 길이" 표현용이다.

### uuid vs timeuuid — 왜 둘을 구분하는가

이 구분은 실전에서 매우 중요하다. 둘 다 128비트 식별자지만 결정적 차이가 있다.

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

`timeuuid`를 클러스터링 키로 쓰면 "고유성"과 "시간순 정렬"을 한 컬럼으로 동시에 얻는다. 이벤트 소싱([[13 - 실전 Event Sourcing on Cassandra]])에서 이벤트 ID로 자주 쓰는 이유다. `timestamp`만으로는 같은 밀리초에 발생한 이벤트들의 순서·유일성을 보장 못 하지만 `timeuuid`는 클럭 시퀀스로 그것까지 해결한다.

관련 내장 함수들:

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

> **주의 — `SELECT`에는 `FROM`이 필수다**: 위 `SELECT now();`, `SELECT uuid();`처럼 함수만 단독 평가하고 싶어도 Cassandra CQL 문법은 `FROM` 절 없는 `SELECT`를 허용하지 않는다(PostgreSQL식 `SELECT now();`는 문법 오류). cqlsh에서는 보통 단일 row짜리 시스템 테이블을 빌려 `SELECT now() FROM system.local;`처럼 쓴다. 실제 쓰기에서는 `INSERT INTO events (id, ...) VALUES (now(), ...)`처럼 값 위치에 함수를 직접 넣는다.

`minTimeuuid`/`maxTimeuuid`가 만드는 값은 **실제 식별자로 쓰면 안 되는 경계값**이다(같은 시각에 모두 동일). 오직 범위 비교의 하한/상한으로만 쓴다.

### 컬렉션과 UDT — 요약과 경고

`set<T>`, `list<T>`, `map<K,V>`, 그리고 사용자 정의 타입(UDT)과 `tuple`은 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]에서 깊이 다뤘다. 여기서는 CQL 문법과 핵심 함정만 못 박는다.

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

> **함정**: 컬렉션은 **단일 파티션 안의 작은 보조 데이터**용이다. 컬렉션에 수천 개 원소를 넣으면 read 시 통째로 역직렬화되고, 비-frozen 컬렉션의 원소들은 각각 별도 셀(cell)로 저장돼 tombstone을 양산한다. "관계형 1:N을 컬렉션으로 대체"하려는 충동은 거의 항상 안티패턴이다. 별도 테이블 + 클러스터링 키로 풀어라. 상세는 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]].

---

## DML의 충격적 진실: INSERT도 UPDATE도 upsert다

이제 CQL을 SQL과 가장 극적으로 갈라놓는 지점에 왔다. RDB 경험자가 반드시 마음에 새겨야 할 단 하나의 진실이 있다.

> **`INSERT`와 `UPDATE`는 둘 다 단순히 "이 위치에 이 값을 써라"일 뿐이다. 존재 여부를 검사하지 않는다. CQL에는 "삽입"과 "갱신"의 구분이 없다. 둘 다 upsert다.**

### 왜 구분이 없는가 — LSM-tree의 본질

이것은 Cassandra의 기벽이 아니라 [[08 - Storage Engine 내부]]의 LSM-tree 구조에서 나오는 필연이다. 쓰기는 항상 **append-only**다. memtable에 셀을 하나 추가하고 commit log에 적을 뿐, 디스크의 기존 데이터를 찾아가 고치지 않는다. "이 키가 이미 있나?"를 확인하려면 디스크의 모든 SSTable을 읽어야 하는데(read), 그러면 쓰기가 읽기만큼 느려진다. LSM-tree는 그 확인을 **포기**함으로써 쓰기를 빠르게 한다.

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

이 진실의 실전 함의는 무겁다.

> **함정 1 — 중복 결제 방지가 안 된다**: "결제 ID가 이미 있으면 INSERT 실패"를 기대하면 안 된다. CQL의 평범한 `INSERT`는 기존 결제를 조용히 덮어쓴다. PK 유니크 제약으로 멱등성을 보장하던 RDB 패턴이 **그대로 통하지 않는다.** 진짜 "없을 때만 삽입"이 필요하면 `IF NOT EXISTS`(경량 트랜잭션, 뒤에서)를 써야 하고, 그건 비싸다.

> **함정 2 — 부분 갱신이 정상이다**: `UPDATE payments SET status='paid' WHERE id='p1'`은 `p1`이 없으면 `status`만 있고 나머지 컬럼은 null인 "반쪽 row"를 만든다. RDB라면 0 rows affected였을 일이 Cassandra에선 새 row 생성이다.

### INSERT와 UPDATE의 미묘한 차이 두 가지

"사실상 같다"고 했지만 완전히 같지는 않다. 알아둬야 할 두 가지 비대칭이 있다.

**1) 컬렉션 연산은 UPDATE에만 있다.**

```cql
-- set에 원소 추가/삭제는 UPDATE 문법으로만 가능
UPDATE customers SET emails = emails + {'new@x.com'} WHERE id = ...;
UPDATE customers SET emails = emails - {'old@x.com'} WHERE id = ...;
UPDATE customers SET attributes['theme'] = 'dark' WHERE id = ...;
-- INSERT는 컬렉션 '전체 값'을 통째로 쓸 수만 있다
INSERT INTO customers (id, emails) VALUES (..., {'a@x.com','b@x.com'});
```

**2) "row 존재" 개념과 정적 컬럼.** 미묘하지만 `INSERT`로 클러스터링 키만 있고 일반 컬럼이 전혀 없는 row를 쓰면 Cassandra는 그 row의 "존재"를 표시하는 빈 셀(row marker)을 남긴다. `UPDATE`로 일반 컬럼을 모두 null로 만들면 그 row는 "셀이 없는" 상태가 되어 읽기에서 사라질 수 있다. 이 미세한 차이는 정적(static) 컬럼이나 row-존재 판정이 걸린 모델에서 가끔 문제를 일으킨다. 대부분의 경우 신경 쓸 필요는 없지만 "분명히 INSERT했는데 SELECT에 안 보인다" 류의 미스터리를 만나면 이 지점을 의심하라.

### DELETE는 지우지 않는다 — tombstone을 만든다

`DELETE`도 직관을 배신한다. LSM-tree에서 디스크의 SSTable은 불변이다. 그래서 "삭제"는 데이터를 찾아 제거하는 게 아니라 **"이 셀은 이 timestamp 이후로 죽었다"는 묘비(tombstone)를 새로 append**한다. 이것 역시 쓰기다.

```text
  DELETE FROM payments WHERE id='p1';

  SSTable-1 (오래됨):  p1 → amount=5000, status='paid'  (살아있는 듯 보임)
  SSTable-2 (최신):    p1 → [TOMBSTONE @ ts=T2]          ← DELETE가 만든 묘비

  READ 시: 두 SSTable 병합 → tombstone(T2)이 데이터(T1<T2)를 가림
           → "삭제됨"으로 보고됨. 하지만 디스크엔 둘 다 존재!
```

tombstone의 수명은 앞서 본 `gc_grace_seconds`(기본 10일)다. 그 기간이 지나고 compaction이 와야 비로소 원본 데이터와 묘비가 함께 디스크에서 사라진다. 이로부터 두 가지 운영적 진실이 나온다.

1. **삭제는 즉시 디스크를 비우지 않는다.** 오히려 잠시 데이터가 늘어난다(원본 + 묘비). 디스크 압박을 줄이려 대량 DELETE를 돌리면 역효과가 날 수 있다.
2. **tombstone이 쌓이면 읽기가 느려진다.** 살아있는 row를 읽으려고 그 앞을 가로막은 수많은 묘비를 스캔해야 한다. 이것이 악명 높은 "tombstone hell"이다. 상세 메커니즘과 대응은 [[09 - Compaction 전략]].

```cql
DELETE FROM payments WHERE id = 'p1';                    -- 파티션 전체 삭제(partition tombstone)
DELETE status FROM payments WHERE id = 'p1';             -- 특정 컬럼만(cell tombstone)
DELETE FROM payments WHERE id='p1' AND paid_at < '2026-01-01';  -- range tombstone
DELETE emails FROM customers WHERE id = ...;             -- 컬렉션 통째 삭제
```

> **함정 — 큐를 Cassandra로 만들지 마라**: "넣고 빼고(INSERT/DELETE)"를 반복하는 큐/작업 테이블을 Cassandra로 모델링하면, 처리 완료마다 tombstone이 쌓여 곧 읽기가 묘비밭이 된다. "Cassandra anti-pattern: queue"는 거의 고전이다. 이런 워크로드는 Kafka/SQS 같은 전용 도구로 풀어라. 상세는 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]].

---

## SELECT의 제약 — 자유는 설계에서 온다

[[05 - 데이터 모델링 1 - Query First]]에서 SELECT의 제약을 깊이 다뤘지만 CQL 장에서 다시 못 박을 가치가 있다. CQL의 `SELECT`가 SQL과 다른 점은 "무엇을 못 하는가"의 목록이 길다는 데 있다. 이 제약들은 모두 "옵티마이저가 풀스캔/정렬을 몰래 하지 못하게 막는다"는 하나의 원칙에서 나온다.

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

핵심 규칙을 표로 정리한다.

| 절 | 규칙 | 이유 |
|---|---|---|
| `WHERE` 파티션 키 | 모든 파티션 키 컬럼을 **등호**(또는 `IN`)로 | 토큰을 계산해야 노드를 찾음 |
| `WHERE` 클러스터링 키 | 앞에서부터 순서대로. 마지막 하나만 범위(`<`,`>`) | 디스크 정렬 순서를 따라 슬라이스 |
| `ORDER BY` | 클러스터링 순서이거나 그 완전 역순만 | 디스크에 정렬된 것을 읽거나 거꾸로 읽을 뿐 |
| 일반 컬럼 조건 | 기본 거부. `ALLOW FILTERING` 필요 | 파티션 못 좁히면 풀스캔 |

`ALLOW FILTERING`은 "느려도 좋으니 전체를 훑어라"를 명시적으로 허락하는 탈출구다. 이름 그대로 위험 표지판이다.

> **함정 — `ALLOW FILTERING`은 프로덕션 금지에 가깝다**: 이 키워드는 코디네이터가 매칭되는 row를 찾으려 여러 파티션(최악엔 전체)을 스캔하게 한다. 데이터가 작은 개발/관리 쿼리가 아니면 쓰지 마라. "쿼리가 거부돼서 `ALLOW FILTERING`을 붙였더니 됐다"는 거의 항상 **데이터 모델이 그 쿼리를 지원하지 못한다는 신호**다. 정답은 그 쿼리를 위한 테이블을 새로 만드는 것이다([[05 - 데이터 모델링 1 - Query First]]).

세컨더리 인덱스(`CREATE INDEX`)와 SAI(Storage-Attached Index, 5.0의 강화된 인덱스)는 이 제약을 부분적으로 완화하지만 만능이 아니며 고유한 함정이 있다. 이 부분은 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]에서 다룬다.

---

## 페이징: 1억 행을 메모리 폭발 없이 읽기

`SELECT * FROM huge_table`이 1억 행을 반환한다고 하자. RDB라면 커서/LIMIT으로 다루겠지만 Cassandra는 코디네이터와 드라이버 양쪽 모두에서 메모리를 보호하기 위해 **자동 페이징(automatic paging)** 을 기본 내장한다. 이것이 어떻게 동작하는지가 이 절의 핵심이다.

### 드라이버 자동 페이징 — fetch size와 paging state

드라이버는 `SELECT` 결과를 한 번에 다 받지 않는다. **fetch size**(기본 5000행)만큼만 받고 다음 페이지가 필요할 때 서버에 **paging state**라는 불투명(opaque) 토큰을 보내 "여기 이어서 줘"라고 요청한다.

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

`paging_state`는 "마지막으로 읽은 row의 위치(파티션/클러스터링 좌표 등)"를 서버가 인코딩한 토큰이다. 드라이버는 이걸 그냥 다음 요청에 실어 보낼 뿐, 내용을 해석하지 않는다. 대부분의 드라이버(Java, Python 등)에서는 결과 이터레이터를 그냥 끝까지 돌리면 페이징이 **투명하게** 일어난다. 다음은 Java 드라이버의 전형이다.

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

이 "상태 저장 후 이어받기"는 무한 스크롤 UI를 만들 때 유용하다. 단, `paging_state`는 **그 쿼리·그 클러스터 상태에 묶인 토큰**이라 다른 쿼리에 재사용하거나 영구 저장(북마크처럼)하면 안 된다. 짧은 수명의 연속 페이징용이다.

> **함정 — 페이징은 정렬된 전역 순서가 아니다**: 자동 페이징은 "한 파티션 안에서"는 클러스터링 순서를 보장하지만, 파티션 키가 여러 개인 풀스캔(`SELECT * FROM t`)에서는 토큰 순서로 돌 뿐 어떤 비즈니스적 정렬도 없다. "최신순으로 페이징"하려면 그 정렬이 클러스터링 키로 설계돼 있어야 한다.

### token() 함수 — 파티션을 직접 훑는 수동 페이징

자동 페이징은 "한 파티션을 넘어 전체 테이블을 순회"하는 데도 쓰이지만 대규모 데이터 마이그레이션·풀스캔에서는 **`token()` 함수로 파티션을 직접 분할 정복**하는 방식이 더 강력하다.

`token()`은 파티션 키를 [[02 - 분산 아키텍처]]의 토큰(Murmur3 해시, 64비트 정수)으로 변환하는 함수다. 데이터는 디스크에 이 토큰 순서로 저장되므로 토큰 범위를 슬라이스하면 테이블을 결정적으로 쪼갤 수 있다.

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

이 패턴의 위력은 **병렬화**에 있다. 토큰 공간을 N등분해 워커 N개에 분배하면 각 워커가 겹치지 않는 토큰 구간을 독립적으로 스캔한다. 토큰 링과 워커를 그림으로 보면 이렇다.

```text
  토큰 공간 [-2^63, 2^63) 을 4개 워커로 분할 스캔

  -2^63        -2^61          0          +2^61        +2^63
    │────worker1────│────worker2────│────worker3────│────worker4────│
    token(pk)>=lo AND token(pk)<hi 로 각 워커가 자기 구간만 담당
    → 서로 다른 노드 범위를 병렬로 훑음 → 풀 클러스터 처리량 활용
```

`token()` 페이징의 결정적 장점은 **재시작 가능성(resumability)** 이다. paging_state는 휘발성이지만 "마지막으로 처리한 토큰 T"는 그냥 64비트 정수다. 이 값을 어딘가에 적어두면 작업이 죽어도 `token(pk) > T`로 정확히 그 지점부터 재개할 수 있다. 대규모 백필(backfill)·마이그레이션·전체 리포팅 잡의 표준 패턴이다.

> 참고: `WHERE token(pk) > T`로 슬라이스할 때, 파티션 키가 복합이면 `token(pk1, pk2)`처럼 전체 파티션 키를 함께 넘겨야 한다. 토큰은 파티션 키 전체에 대해 계산되기 때문이다. 또한 같은 토큰에 여러 파티션 키가 매핑될 수 있으므로(드물지만 해시 충돌/근접), 경계에서 `>`와 `>=`의 선택과 중복 처리에 주의해야 한다.

DataStax의 `dsbulk` 같은 전용 로딩 툴이나 Spark-Cassandra 커넥터는 내부적으로 바로 이 `token()` 분할을 사용해 클러스터 전체를 병렬로 읽고 쓴다. 직접 풀스캔 잡을 짠다면 이 패턴을 따르는 것이 정석이다.

---

## TTL: 시간이 지나면 스스로 죽는 데이터

Cassandra는 **각 셀(cell) 단위로 만료 시각(time-to-live)** 을 둘 수 있다. 지정한 초가 지나면 그 셀은 자동으로 만료되어 사라진다. 세션 데이터, 캐시, 일정 기간만 보관하는 이벤트 로그에 안성맞춤이다. 하지만 TTL의 내부 동작을 모르면 또 다른 tombstone 함정에 빠진다.

### per-column TTL과 default_time_to_live

TTL은 세 층위에서 지정할 수 있다.

```cql
-- 1) 이 INSERT/UPDATE에만 적용되는 TTL (초 단위). 86400 = 1일
INSERT INTO sessions (id, token) VALUES ('s1', 'abc') USING TTL 86400;
UPDATE sessions USING TTL 3600 SET token = 'xyz' WHERE id = 's1';

-- 2) 테이블 기본 TTL — 모든 쓰기에 자동 적용
CREATE TABLE sessions (...) WITH default_time_to_live = 86400;

-- 3) TTL 제거(영구 보관으로 전환) — USING TTL 0
UPDATE sessions USING TTL 0 SET token = 'permanent' WHERE id = 's1';
```

한 가지 중요한 디테일이 있다. **TTL은 컬럼(셀)별로 독립**이다. 한 row의 컬럼마다 TTL이 다를 수 있다. 예컨대 `INSERT ... USING TTL 100`으로 모든 컬럼에 100초를 걸고 나중에 `UPDATE ... USING TTL 50 SET status=...`로 `status`만 50초로 갱신하면 `status`는 50초 후, 나머지는 100초 후에 만료된다. row가 통째로 사라지는 게 아니라 셀들이 각자의 시간표대로 죽는다.

### TTL 만료 = tombstone — 9장으로 가는 다리

여기가 TTL의 가장 중요한 진실이다.

> **TTL이 만료되어도 데이터가 즉시 디스크에서 사라지지 않는다. 만료된 셀은 tombstone으로 변한다.** 그리고 그 tombstone은 `gc_grace_seconds`가 지나고 compaction이 와야 비로소 정리된다.

```text
  USING TTL 86400 으로 쓴 셀의 일생

  t=0           셀 작성 (살아있음, 만료 예정 시각 = t+86400 기록됨)
  t=86400       만료 시점 도달 → 읽으면 '없음'으로 보임
                하지만 디스크에는 expired cell(=tombstone)로 남아있음
  t=86400+gc_grace  compaction이 오면 비로소 물리적으로 제거
```

이 메커니즘이 만드는 함정은 [[09 - Compaction 전략]]의 핵심 주제와 직결된다. **TTL 데이터를 STCS로 다루면 만료된 tombstone과 살아있는 데이터가 한 SSTable에 섞여 만료분을 정리하려고 살아있는 데이터까지 반복해서 다시 쓰게 된다.** 디스크는 안 줄고 읽기는 묘비밭이 된다.

해결책이 바로 `TimeWindowCompactionStrategy`(TWCS)다. TWCS는 같은 시간 창에 쓰인 데이터를 같은 SSTable 묶음에 격리한다. 그 창의 모든 데이터가 같은 시기에 TTL 만료되면 **SSTable 파일을 통째로 drop**하면 끝이다. 셀 단위로 묘비를 정리할 필요가 없다.

```text
  TWCS + TTL 의 우아함

  [6/24 창 SSTable] [6/25 창] [6/26 창] [6/27 창] ...
        │                                              
   TTL로 6/24 데이터 전부 만료 →  파일 통째로 삭제(drop). 끝.
   (셀 하나하나 tombstone 스캔/재작성 불필요)
```

> **실전 규칙**: "시간순으로 쌓고 일정 기간 후 만료"하는 데이터(이벤트 로그, 세션, 시계열)는 **TWCS + default_time_to_live** 조합이 거의 정답이다. 결제 이벤트 보관 정책(예: 13개월 후 만료)을 이 조합으로 구현하면 디스크가 자동으로 회수된다.

### WRITETIME()와 TTL() — 셀의 메타데이터를 들여다보기

모든 셀은 값 외에 두 가지 메타데이터를 품는다. **쓰기 timestamp**(마이크로초 단위, 충돌 해결의 기준)와 **남은 TTL**이다. 이를 조회하는 함수가 있다.

```cql
-- amount 컬럼이 언제 쓰였는지(epoch 마이크로초), 남은 TTL은 몇 초인지
SELECT amount,
       WRITETIME(amount) AS written_us,
       TTL(amount) AS ttl_remaining
FROM payments WHERE id = 'p1';
```

`WRITETIME()`은 디버깅과 데이터 포렌식에 강력하다. "이 값이 정확히 언제 마지막으로 갱신됐나"를 알 수 있고 충돌 해결(last-write-wins)에서 어느 쓰기가 이겼는지 추적할 수 있다. `TTL()`은 "이 세션이 앞으로 몇 초 더 사나"를 확인한다. 두 함수 모두 **단일 셀(일반 컬럼)에만** 적용된다. 파티션/클러스터링 키에는 적용할 수 없다(키는 별도 셀이 아니라 row의 좌표이기 때문).

> **함정 — WRITETIME과 USING TIMESTAMP의 위험한 결합**: 쓰기 timestamp는 충돌 해결의 절대 기준이다. 곧 볼 `USING TIMESTAMP`로 이 값을 인위적으로 조작하면, "과거 timestamp로 쓴 값이 최신 값에 가려 영원히 안 보이는" 좀비/유령 현상이 생긴다. WRITETIME으로 그 흔적을 추적할 수는 있지만, 애초에 timestamp를 손대지 않는 게 상책이다.

---

## 고급 DML: LWT, BATCH, prepared, USING, CONSISTENCY, JSON

마지막으로 실전 CQL의 무기고를 훑는다. 각 기능의 깊은 내부는 [[11 - LWT Batch Counter 내부]]에서 다루지만 여기서 문법과 "왜 조심해야 하는가"를 분명히 해둔다.

### 경량 트랜잭션(LWT): IF NOT EXISTS / IF 조건

앞서 "INSERT는 존재 검사를 안 한다"고 했다. 진짜로 "없을 때만 삽입", "현재 상태가 X일 때만 갱신"이 필요할 때 쓰는 것이 **경량 트랜잭션(lightweight transaction, LWT)** 이다. `IF` 절을 붙이면 된다.

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

`IF NOT EXISTS`는 "결제 ID 중복 방지", `IF status='pending'`은 "이미 처리된 결제를 두 번 처리하지 않기" 같은 **상태 기계 전이의 원자성**을 보장한다. RDB의 유니크 제약/낙관적 락에 대응하는 도구다.

그런데 이게 왜 "경량(lightweight)"인가? 사실 이름이 역설적이다. 내부적으로는 **Paxos 합의 프로토콜**을 돌려 복제본들 사이에 "지금 이 조건이 참인가"를 합의한다. 이는 일반 쓰기와 비교해 **네트워크 왕복이 약 4배**(prepare → promise → propose → accept, 그리고 commit) 더 든다.

```text
  일반 쓰기 vs LWT 비용

  일반 INSERT:   코디네이터 → 레플리카들 (1 라운드트립)
  LWT (IF ...):  Paxos 합의 → 4 단계(왕복 다수) → CAS read → 적용
                 → 지연 수배~10배, 처리량 급감
```

> **함정 — LWT 남용**: `IF NOT EXISTS`를 모든 INSERT에 습관처럼 붙이면 클러스터 처리량이 무너진다. LWT는 "정말로 선형성(linearizability)이 필요한 소수의 임계 경로"(중복 결제 방지, 유일 ID 발급, 상태 전이)에만 써라. 또한 LWT는 같은 파티션에 집중되면 Paxos 경합으로 더 느려진다. 상세한 Paxos 동작과 격리 한계(LWT는 ACID 트랜잭션이 아니다)는 [[11 - LWT Batch Counter 내부]].

### BATCH — 원자성 보장이지만 성능 최적화가 아니다

`BATCH`는 여러 DML을 한 묶음으로 보낸다. RDB의 트랜잭션처럼 보이지만 **BATCH의 가장 흔한 오해는 "성능을 위한 것"이라는 착각**이다.

```cql
BEGIN BATCH
  INSERT INTO payment_by_id (id, user_id, amount) VALUES ('p1','A',5000);
  INSERT INTO payment_by_user (user_id, bucket, paid_at, id, amount)
    VALUES ('A','2026-06','2026-06-26 09:00','p1',5000);
APPLY BATCH;
```

CQL BATCH의 진짜 목적은 **원자성**이다. 위 예처럼 "같은 결제를 두 개의 비정규화 테이블에 동시에 쓸 때, 둘 다 성공하거나 둘 다 실패"를 보장한다([[05 - 데이터 모델링 1 - Query First]]의 비정규화 동기화). RDB 트랜잭션과 다른 점:

- **격리(isolation)는 단일 파티션 안에서만** 보장된다. 여러 파티션에 걸친 BATCH는 다른 읽기가 "절반만 적용된 중간 상태"를 볼 수 있다.
- **롤백이 없다.** 코디네이터가 batchlog에 기록한 뒤 재시도로 "결국 모두 적용"을 보장할 뿐, 실패 시 되돌리지 않는다.

> **함정 — multi-partition BATCH로 성능 챙기기**: 서로 다른 파티션의 쓰기를 BATCH로 묶으면 빨라질 거라 기대하면 정반대다. 코디네이터가 batchlog를 두 노드에 먼저 기록(durability)한 뒤 흩어진 파티션들로 쓰기를 보내야 하므로 **오히려 느려지고 코디네이터에 부하가 쏠린다.** 성능을 위해서라면 비동기 개별 쓰기를 병렬로 날리는 게 빠르다. BATCH는 "같은 파티션 묶음의 원자성" 또는 "비정규화 테이블 동기화"에만 써라. 상세는 [[11 - LWT Batch Counter 내부]].

### Counter — 분산 증가/감소 카운터

`counter` 타입은 특수하다. 분산 환경에서 안전하게 증감하는 카운터를 위한 별도 타입으로, 일반 컬럼과 같은 테이블에 섞을 수 없다(PK + counter 컬럼만 가능).

```cql
CREATE TABLE page_views (
    page_id text PRIMARY KEY,
    views counter
);
UPDATE page_views SET views = views + 1 WHERE page_id = 'home';
```

counter는 `INSERT`로 쓸 수 없고 오직 `UPDATE`의 증감(`+`/`-`)으로만 다룬다. **멱등하지도 않다**(같은 `+1`을 재시도하면 두 번 더해질 수 있다). 내부적으로 복제본별 부분합을 합산하는 복잡한 메커니즘을 쓴다. 그래서 정확한 회계(돈 계산)에는 부적합하다. 상세는 [[11 - LWT Batch Counter 내부]].

### Prepared Statement — 거의 항상 써야 하는 것

`prepared statement`는 쿼리 문자열을 한 번 서버에 등록(parse + 메타데이터 캐시)해두고 이후엔 바인딩 값만 보내는 방식이다. 성능과 보안 양면에서 거의 항상 정답이다.

```java
// 한 번 prepare (드라이버가 query id를 캐시)
PreparedStatement ps = session.prepare(
    "INSERT INTO payments (id, amount, status) VALUES (?, ?, ?)");

// 이후엔 바인딩만 — 파싱 생략, 토큰 인지 라우팅, SQL 인젝션 차단
session.execute(ps.bind("p1", 5000L, "paid"));
session.execute(ps.bind("p2", 3000L, "pending"));
```

이점은 세 가지다. (1) 매번 쿼리 파싱을 생략해 빠르다. (2) 드라이버가 파티션 키 위치를 알아 **토큰 인지 라우팅**(token-aware routing)으로 코디네이터를 거치지 않고 데이터 보유 노드에 직접 보낸다. (3) 값이 바인딩이라 **CQL 인젝션이 원천 차단**된다. 문자열 연결로 쿼리를 만드는 것은 피하고 반복 쿼리는 prepare하라.

### USING TIMESTAMP / USING TTL — 정밀하지만 위험한 무기

쓰기마다 Cassandra는 충돌 해결용 timestamp(마이크로초)를 자동 부여한다. `USING TIMESTAMP`로 이걸 직접 지정할 수 있다.

```cql
INSERT INTO payments (id, status) VALUES ('p1', 'paid')
USING TIMESTAMP 1782000000000000 AND TTL 3600;
```

이건 데이터 마이그레이션(원본 시각 보존)이나 특정 충돌 해결 시나리오에서 유용하지만 매우 위험하다.

> **함정 — USING TIMESTAMP의 유령 데이터**: 충돌 해결은 timestamp가 큰 쪽이 무조건 이긴다(last-write-wins). 미래 timestamp로 값을 한 번 써버리면, 이후의 모든 정상 쓰기(현재 시각 timestamp)가 그 미래 값에 가려 **영원히 적용되지 않는다.** "분명히 UPDATE했는데 값이 안 바뀐다"는 미스터리의 단골 원인이다. DELETE의 tombstone도 timestamp를 가지므로, 과거 timestamp로 INSERT하면 더 큰 timestamp의 tombstone에 가려 데이터가 안 보이는 일도 생긴다. **꼭 필요한 게 아니면 timestamp를 손대지 마라.**

### CONSISTENCY — 읽기/쓰기마다 일관성을 고른다

CQL은 문장마다 일관성 레벨(consistency level)을 고를 수 있다. [[04 - Tunable Consistency]]의 핵심이다.

```cql
-- cqlsh 세션 레벨로 설정
CONSISTENCY QUORUM;        -- 이후 쿼리는 과반 복제본 응답 대기
CONSISTENCY LOCAL_QUORUM;  -- 로컬 DC의 과반만 (멀티 DC 표준)
CONSISTENCY ONE;           -- 한 복제본만 응답하면 OK (빠름, 약함)
```

드라이버에서는 statement 단위로 지정한다. 핵심 공식은 `R + W > RF`(읽기 정족수 + 쓰기 정족수 > 복제 계수)다. 이를 만족하면 강한 일관성(strong consistency)이 보장된다. 결제 시스템은 보통 쓰기·읽기 모두 `LOCAL_QUORUM`을 써서 단일 DC 내에서 `R+W>RF`를 만족시키면서도 DC 간 지연을 피한다. 자세한 트레이드오프(ONE/QUORUM/ALL/EACH_QUORUM, 힌티드 핸드오프, read repair)는 [[04 - Tunable Consistency]].

### JSON 지원 — INSERT JSON / SELECT JSON / fromJson / toJson

CQL은 row를 JSON으로 주고받는 문법을 제공한다. API 경계에서 편리하다.

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

알아둘 점 두 가지. (1) `INSERT JSON`은 일반 `INSERT`와 동일하게 **upsert**이며 JSON에 없는 컬럼은 (기본적으로) null로 처리된다 — `DEFAULT UNSET`을 지정하면 누락 컬럼을 건드리지 않는다. (2) JSON은 편의 기능일 뿐, **타입 직렬화는 동일**하다. 다만 JSON 텍스트 표현 규칙이 있으니 API 계약에서 주의해야 한다 — `timestamp`는 ISO-8601 유사 문자열(`"2026-06-26 00:00:00.000Z"`, 공백 구분), `blob`은 CQL 리터럴과 같은 16진수 문자열(`"0x..."`, base64 아님)로 나간다. `decimal`/`varint`는 JSON 숫자로 출력되지만, 클라이언트 JSON 파서의 정밀도 손실을 피하려 입력 시 따옴표로 감싼 문자열로도 받아준다. 성능 임계 경로에서는 JSON 파싱 오버헤드를 감안하라.

---

## RDB였다면 vs Cassandra — 한눈에

이 장 전체를 직관으로 묶는 대조표다.

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

이 표의 모든 행은 한 가지로 수렴한다. **CQL은 LSM-tree 위의 분산 append-only 스토리지를 다루는 언어**다. SQL의 외형을 빌렸을 뿐, 그 아래 물리학은 완전히 다르다. 문법이 같아 보일 때일수록 "이 문장이 디스크와 토큰 링에서 실제로 무슨 일을 하는가"를 떠올리는 것이 이 언어를 제대로 쓰는 길이다.

---

## 핵심 요약

- **keyspace는 복제의 경계선**이다. `NetworkTopologyStrategy` + DC별 RF를 프로덕션 기본으로, `durable_writes=true`(기본)는 끄지 마라. RF 변경 후엔 반드시 `nodetool repair`.
- **PRIMARY KEY가 물리 설계 전부**다. 괄호 위치로 파티션 키 vs 클러스터링 키가 갈린다. 파티션 키 = 노드 위치, 클러스터링 키 = 디스크 정렬 순서. PK와 CLUSTERING ORDER는 ALTER로 못 바꾼다.
- `WITH` 옵션이 운영을 좌우한다: **compaction**(워크로드 적합성), **gc_grace_seconds**(기본 10일, 좀비 방지 안전 마진, repair 주기와 연동), **compression**(chunk 16KB 기본), caching(row cache는 대개 안티패턴).
- 데이터 타입은 바이트 직렬화가 정렬·비교를 결정한다. **돈은 `bigint`/`decimal`(절대 `float`/`double` 금지)**, 시간순 정렬+유일성은 **`timeuuid`**, `timestamp`는 타임존 없는 UTC ms.
- **INSERT = UPDATE = upsert**. 존재 검사 없음. LSM-tree append-only의 필연. "삽입/갱신" 구분이 CQL엔 없다.
- **DELETE와 TTL 만료는 tombstone을 만든다.** 즉시 안 지워지고 gc_grace 후 compaction에서 정리. 시계열/TTL 데이터는 **TWCS**로 SSTable 통째 drop이 정답.
- **SELECT는 파티션 키 등호 + 클러스터링 슬라이스**만 자유롭다. `ALLOW FILTERING`은 모델 설계 실패 신호.
- **페이징**: 드라이버 자동 페이징(fetch size + 휘발성 paging state)은 투명하게. 대규모 풀스캔/마이그레이션은 **`token()` 분할**로 병렬화·재시작 가능하게.
- **LWT(`IF`)는 Paxos라 ~4배 왕복**, 임계 경로에만. **BATCH는 원자성용이지 성능용이 아니다**(multi-partition은 오히려 느림). **prepared statement는 거의 항상 사용**(파싱 생략 + 토큰 라우팅 + 인젝션 차단). `USING TIMESTAMP`는 유령 데이터를 만드는 위험한 무기.

## 연결 노트

- [[05 - 데이터 모델링 1 - Query First]] — PRIMARY KEY 설계와 SELECT 제약의 근거. 이 장의 DDL/SELECT 절은 이 모델링 철학의 CQL 구현이다.
- [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]] — 컬렉션·UDT·세컨더리 인덱스·큐 안티패턴의 상세.
- [[08 - Storage Engine 내부]] — memtable/SSTable/commit log. upsert와 tombstone이 왜 필연인지의 물리적 토대.
- [[09 - Compaction 전략]] — gc_grace_seconds, tombstone 정리, TWCS와 TTL의 상호작용 심화.
- [[10 - Write Path와 Read Path]] — durable_writes, 쓰기 1회 vs 읽기 병합, paging이 경로 위에서 도는 모습.
- [[11 - LWT Batch Counter 내부]] — IF/Paxos, BATCH batchlog, counter의 내부 메커니즘.
- [[04 - Tunable Consistency]] — CONSISTENCY 레벨과 R+W>RF.
- [[02 - 분산 아키텍처]] / [[03 - 복제 전략과 데이터센터]] — token()과 replication 전략의 토대.
- [[13 - 실전 Event Sourcing on Cassandra]] — timeuuid·TTL·TWCS·idempotency를 실전 이벤트 소싱으로 엮는 종합편.
