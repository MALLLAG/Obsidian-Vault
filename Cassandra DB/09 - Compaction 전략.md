---
title: Compaction 전략 - STCS/LCS/TWCS/UCS, tombstone GC
date: 2026-06-26
tags: [cassandra, compaction, tombstone, 학습노트]
---

[[08 - Storage Engine 내부]]에서 Cassandra의 쓰기가 **append-only LSM-tree**라는 사실을 봤다. 모든 쓰기는 memtable에 쌓였다가 SSTable로 flush되고, SSTable은 한 번 디스크에 내려가면 **절대 수정되지 않는다(immutable)**. UPDATE도, DELETE도 새로운 셀(cell)을 덧쓰는 행위일 뿐이다. 이 설계는 쓰기를 극단적으로 빠르게 만든다. 랜덤 I/O가 없고 순차 append만 있으니까.

그런데 이 우아함에는 대가가 있다. **하나의 파티션 키가 시간이 지나면 수십, 수백 개의 SSTable에 흩어진다.** 같은 행(row)을 여러 번 UPDATE하면 그 조각들이 여러 파일에 나뉘어 누워 있고, 읽을 때마다 그것들을 모아 최신 버전을 재구성해야 한다. 삭제는 데이터를 지우는 게 아니라 "지워졌음"을 기록하는 묘비(tombstone)를 추가할 뿐이라, 디스크에는 죽은 데이터와 그 묘비가 함께 쌓여간다.

**Compaction은 이 부채를 정리하는 백그라운드 청소 작업이다.** 여러 SSTable을 읽어 병합(merge)하고, 같은 키의 낡은 버전을 버리고, 만료된 tombstone을 제거하고, 정리된 결과를 새 SSTable로 쓴 뒤 원본을 삭제한다. 이 장은 그 청소가 *어떤 전략으로* 이뤄지는지, 그리고 전략 선택이 왜 Cassandra 운영의 가장 중요한 의사결정 중 하나인지를 깊이 파고든다.

> RDB였다면: PostgreSQL의 `VACUUM`이나 InnoDB의 purge thread가 가장 가까운 대응물이다. 하지만 결정적 차이가 있다. RDB는 페이지를 **제자리에서 수정(update-in-place)**하므로 죽은 튜플 회수가 "정리"의 전부다. Cassandra의 compaction은 정리에 더해 **읽기 성능 자체를 좌우하는 데이터 재배치(re-organization)**까지 책임진다. compaction 전략을 잘못 고르면 단순히 디스크가 더러워지는 게 아니라, 읽기 지연이 수십 배로 폭증한다.

---

## 왜 Compaction이 필요한가: 흩어진 키의 비용

먼저 compaction이 없는 세계를 상상해보자. 결제 시스템에서 한 주문의 상태가 여러 번 바뀐다고 하자.

```cql
INSERT INTO orders (order_id, status, amount) VALUES ('ord-42', 'CREATED', 10000);
-- ... memtable flush → SSTable-1
UPDATE orders SET status = 'PAID'      WHERE order_id = 'ord-42';
-- ... flush → SSTable-2
UPDATE orders SET status = 'SHIPPED'   WHERE order_id = 'ord-42';
-- ... flush → SSTable-3
UPDATE orders SET status = 'DELIVERED' WHERE order_id = 'ord-42';
-- ... flush → SSTable-4
```

이제 `ord-42`를 한 번 읽을 때 벌어지는 일을 따라가 보자. 디스크에는 이 한 행의 조각이 네 군데에 흩어져 있다.

```text
   읽기: SELECT * FROM orders WHERE order_id = 'ord-42';

   SSTable-1   SSTable-2   SSTable-3   SSTable-4
  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
  │order_id │ │order_id │ │order_id │ │order_id │
  │ status= │ │ status= │ │ status= │ │ status= │
  │ CREATED │ │ PAID    │ │ SHIPPED │ │DELIVERED│
  │amount=  │ │ (ts=t1) │ │ (ts=t2) │ │ (ts=t3) │
  │ 10000   │ │         │ │         │ │         │
  │ (ts=t0) │ │         │ │         │ │         │
  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
       │           │           │           │
       └───────────┴─────┬─────┴───────────┘
                         ▼
              merge(timestamp 기준 최신 승리)
                         ▼
       order_id=ord-42, status=DELIVERED, amount=10000
```

Cassandra는 네 개의 SSTable을 **모두** 열어, 각각에서 `ord-42`의 조각을 꺼내, **셀 단위 timestamp**로 최신 값을 골라(last-write-wins) 재구성한다. SSTable이 많을수록 읽기 경로에서 만져야 할 파일이 늘고, 각 파일마다 먼저 bloom filter로 거른 뒤(가장 앞단 게이트), 통과한 파일만 partition index 조회 + 디스크 seek로 이어진다. 이것이 **읽기 증폭(read amplification)**이다.

여기에 세 가지 비용이 더 쌓인다.

1. **병합 비용**: 읽을 때마다 흩어진 버전을 모으는 CPU·메모리 작업.
2. **tombstone 누적**: DELETE는 묘비를 추가할 뿐이다. 죽은 데이터 + 묘비가 SSTable마다 쌓여, 읽기 시 "이미 죽었지만 아직 스캔해야 하는" 셀이 늘어난다.
3. **공간 낭비**: `CREATED`, `PAID`, `SHIPPED`는 이미 무의미한 죽은 버전인데도 디스크를 차지한다. 논리적으로는 행 하나지만 물리적으로는 네 벌.

compaction은 이 네 SSTable을 하나로 합치면서 `DELIVERED`만 남기고 나머지를 버린다. 그러면 읽기는 SSTable 하나만 보면 되고, 죽은 버전과 만료 묘비는 사라진다.

```text
  compaction 후:
  ┌──────────────────────────────────────────┐
  │ SSTable-new                              │
  │  order_id=ord-42, status=DELIVERED,      │
  │  amount=10000   (낡은 버전 3개 제거됨)    │
  └──────────────────────────────────────────┘
```

> **읽기 성능은 "키 하나를 읽기 위해 만져야 하는 SSTable 수"에 직접 비례한다.** compaction 전략은 이 숫자를 어떻게 통제하느냐의 싸움이다. STCS는 이걸 느슨하게 두고, LCS는 강하게 보장하며, TWCS는 시간 축으로 분리하고, UCS는 이 둘 사이를 다이얼로 조절한다.

### compaction의 세 가지 증폭(amplification) 삼각관계

compaction 전략을 이해하는 가장 좋은 렌즈는 **세 가지 증폭의 트레이드오프**다. 어떤 전략도 셋 다 최소화할 수 없다. 하나를 누르면 다른 게 튀어나오는 풍선이다.

| 증폭 | 정의 | 누가 손해 보나 |
|------|------|----------------|
| 읽기 증폭(read amp) | 키 하나를 읽기 위해 접근하는 SSTable/디스크 수 | 읽기 지연 |
| 쓰기 증폭(write amp) | 논리적 1바이트가 compaction으로 디스크에 총 몇 번 쓰이나 | 디스크 수명·I/O 대역폭 |
| 공간 증폭(space amp) | 논리 데이터 크기 대비 실제 디스크 점유 비율 | 디스크 용량 |

- **STCS**: 쓰기 증폭 낮음, 읽기·공간 증폭 높음.
- **LCS**: 읽기·공간 증폭 낮음, 쓰기 증폭 높음.
- **TWCS**: 시계열 한정으로 셋 다 낮지만, 비시계열에선 무너짐.
- **UCS**: 파라미터(scaling parameter)로 STCS↔LCS 사이 어디든 위치 지정.

이 삼각형을 머릿속에 박아두고 나머지를 읽으면 모든 전략이 "이 삼각형의 어느 꼭짓점을 택했나"로 정리된다.

---

## STCS: 비슷한 크기끼리 묶는다

**SizeTieredCompactionStrategy(STCS)**는 Cassandra의 가장 오래된 기본 전략이자 가장 직관적인 발상이다. 이름 그대로 **비슷한 크기(size-tiered)의 SSTable을 한 묶음(bucket)으로 모아 병합한다.**

### 내부 메커니즘: 버킷팅(bucketing)

STCS는 SSTable들을 크기별로 버킷에 나눈다. 기본 동작은 이렇다.

- SSTable 크기를 보고, 평균 크기의 `bucket_low`(기본 0.5) ~ `bucket_high`(기본 1.5) 배 범위 안에 드는 것들을 같은 버킷으로 묶는다. 즉 "대략 같은 크기"끼리 모인다.
- 한 버킷에 `min_threshold`(기본 **4**)개 이상의 SSTable이 쌓이면 그 버킷을 통째로 병합한다.
- 한 번에 묶는 최대 개수는 `max_threshold`(기본 **32**).

```text
  flush로 작은 SSTable이 계속 생긴다:

  L? (크기 무관, 버킷만 존재)

  [▪][▪][▪][▪]  ← 작은 것 4개 모임 → compaction!
        │
        ▼
       [▩]      ← 합쳐서 중간 크기 하나
  
  [▩][▩][▩][▩]  ← 중간 것 4개 모임 → compaction!
        │
        ▼
       [█]      ← 더 큰 하나

  [█][█][█][█]  ← 큰 것 4개 → ... 계속 커진다
```

작은 SSTable 4개가 모이면 중간 크기 하나가 되고, 중간 4개가 모이면 큰 것 하나가 된다. 시간이 지나면 디스크에는 **크기 계층(tier)이 자연스럽게 형성**된다. 작은 것 몇 개, 중간 것 몇 개, 거대한 것 한두 개가 공존한다.

### 왜 write-heavy에 좋은가

STCS의 쓰기 증폭이 낮은 이유는 단순하다. **한 데이터 조각이 compaction에 참여하는 횟수가 적다.** 작은 SSTable이 만들어지고 → 한 번 합쳐져 중간이 되고 → 한 번 합쳐져 큰 게 되는, 즉 크기 계층을 한 단계 올라갈 때마다 딱 한 번씩만 다시 쓰인다. 계층 수가 로그 스케일이라 총 재작성 횟수가 작다.

쓰기 폭주 상황(로그 적재, 이벤트 수집, append 위주 테이블)에서는 이 낮은 쓰기 증폭이 결정적이다. compaction이 쓰기 I/O를 거의 방해하지 않는다.

### STCS의 세 가지 함정

**함정 1 — 공간 증폭(space amplification): 일시적으로 거대한 여유 디스크가 필요하다.**

가장 큰 버킷이 병합될 때를 생각해보자. 100GB짜리 SSTable 4개가 한 버킷에 모이면, STCS는 이 400GB를 읽어 새 SSTable 하나로 쓴다. **병합이 끝나 원본을 지우기 전까지는 입력 400GB와 출력(최대 400GB)이 디스크에 공존한다.** 최악의 경우 데이터 크기만큼의 추가 여유 공간이 필요하다.

> 흔한 운영 사고: "디스크 사용률이 50%인데 compaction이 `No space left`로 실패한다." STCS에서 단일 compaction이 요구하는 임시 공간을 간과한 경우다. STCS 테이블이 디스크의 50% 이상을 차지하면, 가장 큰 버킷의 major compaction을 수행할 공간이 없어 막힐 수 있다. **STCS는 디스크 여유를 데이터 크기만큼 남겨두는 것을 전제로 한다.**

**함정 2 — 읽기 증폭: 하나의 키가 여러 크기 계층에 동시에 존재할 수 있다.**

STCS는 SSTable 간에 키 범위가 겹치지 않는다는 보장을 전혀 하지 않는다. 같은 키 `ord-42`가 작은 SSTable, 중간 SSTable, 거대한 SSTable에 동시에 흩어져 있을 수 있다. 읽으려면 이 모두를 확인해야 한다. **bloom filter가 대부분을 걸러주지만, 최악의 경우 읽기가 만져야 할 SSTable 수에 상한이 없다.** read-heavy 워크로드에서 STCS의 p99 읽기 지연이 들쭉날쭉한 이유다.

**함정 3 — 거대 SSTable과 좀비 tombstone.**

큰 SSTable일수록 다른 SSTable과 병합될 기회가 드물어진다(같은 크기 짝이 4개나 모이기 어려우니까). 그 결과 **오래된 데이터와 그것을 덮어쓴/삭제한 tombstone이 서로 다른 크기 계층에 갇혀 영원히 만나지 못하는** 상황이 생긴다. tombstone은 짝(shadowed 데이터)과 같은 compaction에 모여야만 둘 다 제거되는데(뒤에서 상술), STCS의 크기 기반 묶음은 이 만남을 보장하지 못한다. 거대 테이블에서 tombstone이 누적되고 읽기가 느려지는 고전적 증상이다.

```cql
-- STCS 설정 (write-heavy, 일반 append 테이블)
CREATE TABLE events (
    event_id uuid,
    payload text,
    PRIMARY KEY (event_id)
) WITH compaction = {
    'class': 'SizeTieredCompactionStrategy',
    'min_threshold': 4,
    'max_threshold': 32
};
```

---

## LCS: 레벨로 읽기 SSTable 수를 묶는다

**LeveledCompactionStrategy(LCS)**는 STCS의 가장 큰 약점 — "한 키를 읽기 위해 만질 SSTable 수에 상한이 없다" — 을 정면으로 해결한다. Google의 LevelDB에서 영감을 받았고 **읽기 시 접근할 SSTable 수에 이론적 상한을 보장**한다.

### 내부 메커니즘: 레벨과 non-overlapping 불변식

LCS는 SSTable을 **레벨(level)**로 조직한다. L0, L1, L2, ... Ln.

핵심 불변식 세 가지:

1. **SSTable 크기는 고정**된다(기본 `sstable_size_in_mb` = **160MB**). 모든 SSTable이 대략 같은 크기.
2. **L1 이상에서는 같은 레벨 내 SSTable끼리 키 범위가 절대 겹치지 않는다(non-overlapping).** 즉 L1 전체가 정렬된 하나의 거대한 run처럼 동작한다.
3. **각 레벨의 총 용량은 바로 위 레벨의 약 10배**다. L1 ≈ 10 × SSTable크기, L2 ≈ 10 × L1, L3 ≈ 10 × L2 ...(`fanout_size` 기본 10).

```text
  L0  [▪][▪][▪][▪]   ← flush 직후. 여기만 키 범위가 겹칠 수 있다
        │ (L0가 차면 L1로 병합)
        ▼
  L1  [─a─][─b─][─c─] ...   합 ≈ 10 SSTable, 내부 키범위 겹침 없음
        │
        ▼
  L2  [a][b][c][d] ...      합 ≈ 100 SSTable, 겹침 없음
        │
        ▼
  L3  ...                   합 ≈ 1000 SSTable, 겹침 없음

  키 범위:  ├────────── 전체 토큰 범위 ──────────┤
  L1:       [──a──][──b──][──c──]   (한 키는 정확히 하나의 SSTable에만)
  L2:       [a][b][c][d][e][f]...   (한 키는 정확히 하나의 SSTable에만)
```

이 구조가 주는 보장이 관건이다. **L1 이상의 각 레벨에서 임의의 키는 최대 한 개의 SSTable에만 존재한다**(non-overlapping이니까). 따라서 한 키를 읽을 때 만져야 할 SSTable 수는 대략 **L0의 SSTable 수 + 레벨 수**로 묶인다. 보통 **L0를 잘 관리하면 키 하나가 90% 이상의 경우 단 하나의 SSTable에만 존재**하며, 최악의 경우에도 레벨 수(보통 한 자릿수) 정도다. 이것이 STCS의 "상한 없음"과 LCS의 "상한 있음"을 가르는 결정적 차이다.

### compaction이 일어나는 방식

L0에 SSTable이 일정 수(보통 4개) 쌓이면 LCS가 작동한다. L0의 SSTable들과, 그것들과 키 범위가 겹치는 L1의 SSTable들을 함께 읽어 병합하고, 결과를 다시 160MB 단위로 잘라 L1에 쓴다. L1이 용량(약 10개)을 초과하면 초과분 SSTable 하나를 골라, 그것과 키 범위가 겹치는 L2의 SSTable들과 병합해 L2로 내린다. 이 과정이 레벨을 따라 cascade된다.

```text
  L1이 넘침:
  L1 [a][b][c]...[k]  ← 한 개 초과
       │ (b를 골라 L2로 내린다)
       ▼
  L2 의 b와 키범위 겹치는 [b1][b2] 와 병합
       ▼
  L2 [a][b1'][b2'][c]...  (b의 데이터가 L2에 합쳐짐)
```

### 왜 read-heavy에 좋은가, 그리고 그 대가

LCS가 read-heavy에 좋은 이유는 명확하다. **읽기가 만질 SSTable 수가 작고 예측 가능하다.** non-overlapping 불변식 덕분에 키 하나가 거의 항상 레벨당 하나에만 있으니, bloom filter와 결합하면 대부분의 점 조회(point lookup)가 1~2개의 SSTable만 건드린다. 공간 증폭도 작다. 같은 키의 중복 버전이 여러 레벨에 오래 살아남지 못하고 빠르게 한 레벨로 합쳐지므로 디스크 점유가 논리 데이터 크기에 가깝다(보통 10% 내외 오버헤드).

대가는 **쓰기 증폭(write amplification)**이다. non-overlapping 불변식을 유지하려면, 새 데이터가 들어올 때마다 그것과 키 범위가 겹치는 하위 레벨 SSTable들을 계속 다시 써야 한다. 한 데이터 조각이 L0 → L1 → L2 → ... 로 내려가며 **레벨마다 한 번씩 재작성**되고, 각 단계에서 겹치는 이웃들까지 끌어들이므로 실제 쓰기 증폭은 보통 **STCS의 2배 이상**, 흔히 10~30배에 이른다.

> 함정: write-heavy 테이블에 LCS를 쓰면 **compaction이 쓰기를 따라잡지 못한다(compaction falls behind).** L0에 SSTable이 계속 쌓여 non-overlapping 보장이 깨지고 L0가 비대해지면 LCS의 읽기 이점이 사라지면서 동시에 쓰기 증폭만 떠안는 최악의 상태가 된다. `nodetool tablestats`에서 "SSTables in each level: [수십/4/40/...]"처럼 L0 숫자가 비정상적으로 크면 이 신호다. **LCS는 쓰기 처리량에 compaction이 따라올 수 있을 때만 유효하다.**

```cql
-- LCS 설정 (read-heavy, 업데이트가 잦은 테이블)
CREATE TABLE user_profiles (
    user_id uuid,
    profile text,
    PRIMARY KEY (user_id)
) WITH compaction = {
    'class': 'LeveledCompactionStrategy',
    'sstable_size_in_mb': 160,
    'fanout_size': 10
};
```

### STCS vs LCS 한눈에

| 항목 | STCS | LCS |
|------|------|-----|
| 묶는 기준 | 비슷한 크기 | 레벨(L0~Ln, 10배씩) |
| 같은 레벨 키 범위 겹침 | 보장 없음 | L1+ 겹침 없음 |
| 읽기 시 SSTable 수 | 상한 없음(최악 다수) | 레벨 수로 제한(보통 1~소수) |
| 쓰기 증폭 | 낮음 | 높음(10~30×) |
| 공간 증폭 | 높음(최악 ~2×) | 낮음(~1.1×) |
| 적합 워크로드 | write-heavy, append | read-heavy, update 잦음 |
| 임시 디스크 요구 | 큼(최대 데이터 크기) | 작음(SSTable 크기 단위) |

---

## TWCS: 시간을 축으로 자른다

**TimeWindowCompactionStrategy(TWCS)**는 STCS·LCS와 전혀 다른 질문에서 출발한다. "데이터에 **시간 순서**라는 구조가 있고, 오래되면 **TTL로 통째로 만료**된다면, 그 구조를 compaction에 활용할 수 없을까?"

시계열(time-series) 데이터(센서 측정값, 로그, 결제 이벤트 스트림, 메트릭)의 전형적 패턴은 이렇다. **데이터는 (거의) 시간순으로 들어오고, 한 번 쓰이면 절대 수정되지 않으며(immutable append), 일정 기간 후 TTL로 사라진다.**

### 왜 STCS·LCS가 시계열에서 끔찍한가

이것이 TWCS를 이해하는 출발점이다. STCS와 LCS는 데이터의 **시간 속성을 전혀 모른다.** 둘은 크기나 레벨만 본다. 그 결과:

```text
  STCS가 시계열을 병합하면:

  [오늘 데이터]──┐
  [어제 데이터]──┼─→ 크기가 비슷하니 한 버킷! ─→ [오늘+어제+그제 섞인 SSTable]
  [그제 데이터]──┘

  문제: 이제 한 SSTable 안에 여러 날짜가 뒤섞여 있다.
```

- **TTL 만료의 비효율**: 어제 데이터의 TTL이 만료되어 전부 죽었다고 하자. 그런데 그 죽은 데이터는 오늘·그제 데이터와 한 SSTable에 섞여 있다. 그 SSTable을 통째로 버릴 수 없다. 산 데이터가 같이 있으니까. **만료 데이터를 회수하려면 살아있는 데이터까지 다시 읽어 재작성하는 compaction을 해야 한다.** 시계열 테이블이 끝없이 compaction을 돌리며 I/O를 태우는 전형적 원인이다.
- **읽기 비효율**: 보통 시계열 조회는 "최근 1시간", "오늘"처럼 시간 범위로 들어온다. 그런데 한 SSTable에 모든 날짜가 섞여 있으면, "오늘" 조회조차 옛날 데이터가 든 SSTable들을 다 건드리게 된다.
- **tombstone·만료 데이터 누적**: TTL이 만료되면 Cassandra는 만료 셀을 tombstone처럼 취급하는데, 이것들이 살아있는 데이터와 섞여 회수되지 못하고 쌓인다.

### TWCS의 메커니즘: 윈도우별 격리

TWCS는 단순하고 강력한 규칙을 따른다. **각 SSTable은 하나의 시간 윈도우(time window)에만 속한다.** 윈도우 크기는 `compaction_window_unit`(예: HOURS, DAYS)과 `compaction_window_size`(예: 1)로 정한다.

- 현재(active) 윈도우 안에서는 STCS로 작은 SSTable들을 합친다.
- 윈도우가 끝나면, 그 윈도우의 모든 SSTable을 **하나의 SSTable로 major compaction**한 뒤 **봉인(seal)**한다. 그 후 그 윈도우의 SSTable은 다시는 다른 윈도우와 섞이지 않는다.

```text
  시간 →
  ┌──────────┬──────────┬──────────┬──────────┐
  │ Window-1 │ Window-2 │ Window-3 │ Window-4 │  (각 1일)
  │  (봉인)  │  (봉인)  │  (봉인)  │ (active) │
  ├──────────┼──────────┼──────────┼──────────┤
  │  [████]  │  [████]  │  [████]  │ [▪][▪][▪]│ ← active만 STCS로 합쳐짐
  │ 1 SSTable│ 1 SSTable│ 1 SSTable│ 합쳐지는중│
  └──────────┴──────────┴──────────┴──────────┘

  Window-1 전체 TTL 만료 → [████] 통째로 DROP (재작성 없음!)
  ┌╌╌╌╌╌╌╌╌╌╌┬──────────┬──────────┬──────────┐
  ┊ (사라짐) ┊ Window-2 │ Window-3 │ Window-4 │
  └╌╌╌╌╌╌╌╌╌╌┴──────────┴──────────┴──────────┘
```

효과는 분명하다. **한 윈도우의 데이터가 전부 TTL 만료되면, 그 윈도우의 SSTable을 통째로 `DROP`한다. 단 한 바이트도 재작성하지 않는다.** 파일 하나를 `rm`하는 것과 같다. 이것이 시계열에서 TWCS가 STCS·LCS를 압도하는 이유다. 만료 데이터를 회수하는 비용이 사실상 0이다.

읽기도 빨라진다. "오늘" 조회는 오늘 윈도우의 SSTable만 보면 되고, bloom filter와 min/max timestamp 메타데이터로 다른 윈도우 SSTable은 통째로 건너뛴다.

한 가지 중요한 단서가 있다. **완전히 만료된 SSTable이라도, 그것이 다른 SSTable과 timestamp 기준으로 겹치면 기본적으로는 통째 DROP되지 않는다.** 뒤에서 볼 overlapping 안전장치가 여기서도 작동하기 때문이다. Cassandra는 "이 SSTable을 지웠을 때 다른 SSTable에 남은 더 오래된 데이터가 부활하지 않는가"를 보수적으로 따진다. 그래서 backfill이나 out-of-order 쓰기로 윈도우 간 timestamp가 뒤섞이면, 만료된 SSTable이 드롭되지 못하고 쌓인다(아래 철칙 3이 중요한 또 다른 이유다). 이 검사를 끄고 "완전히 만료된 SSTable은 overlap 여부와 무관하게 즉시 드롭"하게 만드는 옵션이 `unsafe_aggressive_sstable_expiration`(기본 `false`)이다. 이름 그대로 위험을 감수하는 옵션이며, append-only이고 out-of-order 쓰기가 절대 없다는 것이 확실한 순수 시계열에서만 켜는 것이 안전하다.

### TWCS를 쓰는 철칙

TWCS는 **올바르게 쓰면 마법, 잘못 쓰면 재앙**인 전략이다. 다음 규칙을 어기면 TWCS의 모든 이점이 사라진다.

> **철칙 1 — 데이터는 반드시 TTL을 가져야 한다.** TTL 없는 TWCS는 윈도우 SSTable이 영원히 봉인된 채 누적되기만 한다. 통째 DROP할 일이 없으니 디스크가 무한정 자란다.

> **철칙 2 — 명시적 DELETE를 (거의) 하지 마라.** TWCS의 전제는 "오직 TTL 만료로만 데이터가 사라진다"이다. 봉인된 옛 윈도우의 데이터를 DELETE하면 tombstone이 생기는데, 이 tombstone은 새 윈도우에 들어가고 옛 윈도우의 데이터와 영영 만나지 못한다(다른 SSTable에 격리되어 있으니). 결과적으로 삭제가 작동하지 않고 tombstone만 쌓인다.

> **철칙 3 — 과거 시점으로 쓰지 마라(no out-of-order / backfill writes).** TWCS는 데이터의 write timestamp를 보고 윈도우를 정한다. 오래된 timestamp로 backfill하면, 이미 봉인됐어야 할 옛 윈도우에 새 SSTable이 추가되어 봉인이 깨지고, 그 윈도우가 다시 active처럼 compaction에 끌려든다. 시계열의 시간 순서 가정이 무너진다.

> **함정 — 윈도우 개수**: 윈도우 크기를 너무 작게 잡으면(예: TTL 1년인데 윈도우 1시간) SSTable이 수천 개로 폭증한다. 권장은 **전체 TTL 기간에 윈도우가 20~30개 정도** 생기게 잡는 것이다. TTL 30일이면 윈도우 1일이 적당하다.

```cql
-- TWCS 설정 (시계열 + TTL)
CREATE TABLE sensor_readings (
    sensor_id text,
    reading_time timestamp,
    value double,
    PRIMARY KEY (sensor_id, reading_time)
) WITH CLUSTERING ORDER BY (reading_time DESC)
  AND default_time_to_live = 2592000   -- 30일 TTL
  AND compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'DAYS',
    'compaction_window_size': 1          -- 1일 윈도우 → 약 30개 윈도우
  };
```

> 결제 시스템 관점: 이벤트 소싱([[13 - 실전 Event Sourcing on Cassandra]])에서 "최근 N일 이벤트만 빠르게 조회하고 오래된 건 cold storage로 보낸다" 같은 패턴에 TWCS가 들어맞는다. 단, 이벤트를 **수정·삭제하지 않는 append-only**일 때만이다. 결제 이벤트는 immutable이라 궁합이 좋지만, "정정(amendment)을 DELETE+INSERT로 구현"하는 순간 철칙 2를 어기게 됨에 유의하라.

---

## UCS: 하나의 다이얼로 통합한다 (Cassandra 5.0)

Cassandra 5.0이 가져온 가장 큰 compaction 변화가 **UnifiedCompactionStrategy(UCS)**다. 발상은 이렇다. **STCS와 LCS는 사실 같은 알고리즘의 양 극단이다.** 둘 다 "여러 SSTable을 묶어 병합한다"는 골격이 같고, 차이는 오직 "얼마나 공격적으로 묶느냐"라는 하나의 파라미터에 있다. 그렇다면 그것을 **하나의 연속적인 다이얼(scaling parameter)**로 통합하자.

### scaling parameter `w`: STCS와 LCS를 잇는 연속체

UCS의 핵심은 레벨별 **scaling parameter `w`**다. 각 레벨이 몇 개의 SSTable run을 허용할지를 `w`가 결정한다.

- **`w > 0` (양수): tiered, STCS처럼 동작.** `w`가 클수록 한 레벨에 SSTable을 더 많이 쌓고 한꺼번에 묶는다. 쓰기 증폭↓, 읽기 증폭↑.
- **`w < 0` (음수): leveled, LCS처럼 동작.** `|w|`가 클수록 레벨당 run 수를 적게 유지하려 더 자주 병합한다. 읽기 증폭↓, 쓰기 증폭↑.
- **`w = 0`: 두 동작의 경계.**

```text
  쓰기증폭 ←─────────────────────────────────→ 읽기증폭
   많음                                          많음
  
  w = -2   w = -1    w = 0     w = +2   w = +4 ...
   │         │         │          │        │
  강한 LCS  약한 LCS  경계      약한 STCS 강한 STCS
   │                                          │
  읽기 빠름                                쓰기 빠름
```

운영자는 "STCS냐 LCS냐"라는 이분법 대신, **`w` 하나로 read↔write 트레이드오프 곡선 위 임의의 점을 고른다.** 워크로드가 바뀌면 전략을 통째로 갈아치우는 대신 `w`만 조정하면 된다. 심지어 레벨마다 다른 `w`를 줄 수도 있어(`scaling_parameters`에 레벨별로 지정), "하위 레벨은 tiered로 쓰기 빠르게, 상위 레벨은 leveled로 읽기 빠르게" 같은 혼합 전략이 가능하다.

### 샤딩(sharding): 거대 SSTable 문제의 해법

UCS의 두 번째 큰 기여는 **샤딩**이다. STCS의 고질병 중 하나는 데이터가 커지면 단일 SSTable이 수백 GB로 비대해져, 병합 시 거대한 임시 공간이 필요하고 compaction 한 번이 오래 걸리며 병렬화가 안 된다는 것이었다.

UCS는 토큰 범위를 여러 **shard**로 나누고, 각 레벨의 데이터를 shard 경계에서 잘라 **여러 개의 적당한 크기 SSTable**로 관리한다. 이로써:

- 단일 SSTable이 무한정 커지지 않는다(공간 증폭과 임시 공간 요구가 통제됨).
- 서로 다른 shard의 compaction을 **병렬로** 돌릴 수 있어 멀티코어를 활용한다.
- compaction 단위가 작아져 한 번의 compaction이 빨리 끝나고, 디스크 여유 요구가 줄어든다.

샤딩 정도는 `target_sstable_size`(목표 SSTable 크기, 기본 약 1GiB 안팎)와 데이터 양에 따라 자동 조절된다. 데이터가 적을 땐 적게, 많아지면 자동으로 더 잘게 나눈다.

### UCS의 의미: 운영 단순화

UCS의 진짜 가치는 알고리즘의 새로움이 아니라 **운영 단순화**다. 기존에는 "이 테이블은 읽기가 많으니 LCS, 저 테이블은 쓰기가 많으니 STCS, 시계열은 TWCS"처럼 테이블마다 전략을 고민하고 워크로드가 바뀌면 `ALTER TABLE`로 전략을 통째 교체하며 전면 재compaction의 고통을 겪어야 했다. UCS는 이 결정을 연속 파라미터 조정으로 바꾼다.

> 추측 주의: UCS는 5.0에서 정식 도입되었지만, 모든 워크로드에서 기존 전략을 무조건 능가하는 만능은 아니다. TWCS가 빛나는 순수 시계열+TTL 패턴(통째 DROP)은 여전히 TWCS의 고유 영역이라고 보는 것이 안전하다. 5.0 환경에서 신규 테이블의 범용 기본값으로 UCS를 고려하되, 검증된 워크로드(특히 시계열)는 기존 전략을 유지하는 보수적 접근이 합리적이다. 조직별 권장 기본값은 실측으로 확정하라.

```cql
-- UCS 설정 (Cassandra 5.0)
CREATE TABLE generic_table (
    id uuid,
    data text,
    PRIMARY KEY (id)
) WITH compaction = {
    'class': 'UnifiedCompactionStrategy',
    'scaling_parameters': 'T4',   -- T4 ≈ tiered, STCS와 유사 (min_threshold 4 느낌)
    'target_sstable_size': '1GiB'
};
-- scaling_parameters 예: 'L10'(leveled, fanout 10), 'T4'(tiered),
--   'N'(=T2 경계), 레벨별 지정 'T4, T4, L10' 등도 가능
```

---

## Tombstone과 GC: 죽음을 다루는 법

이제 compaction의 가장 미묘하고 사고가 잦은 영역, **tombstone 회수**로 들어간다. [[08 - Storage Engine 내부]]에서 봤듯 Cassandra에서 DELETE는 데이터를 지우는 게 아니라 **묘비(tombstone)를 쓰는** 행위다. tombstone은 "이 시각 이후 이 데이터는 죽었다"는 표식이다. 문제는 이 묘비를 *언제* 진짜로 치울 수 있느냐다.

### tombstone의 종류

먼저 Cassandra가 만드는 묘비가 한 종류가 아니라는 걸 알아야 한다.

- **Cell tombstone**: 특정 컬럼 하나 삭제(`UPDATE ... SET col = null` 또는 컬럼 DELETE).
- **Row tombstone**: 행 하나 삭제(`DELETE FROM t WHERE pk=.. AND ck=..`).
- **Range tombstone**: 범위 삭제(`DELETE ... WHERE pk=.. AND ck > ..`). 한 묘비가 여러 행을 덮는다.
- **Partition tombstone**: 파티션 전체 삭제(`DELETE FROM t WHERE pk=..`).
- **TTL 만료**: TTL이 지난 셀은 만료 시점에 자동으로 tombstone과 동등하게 취급된다(이른바 expired cell / 만료 묘비). **TTL을 쓰는 것도 결국 tombstone을 양산하는 일**이다.

### 왜 즉시 못 지우는가: gc_grace_seconds

데이터를 DELETE해서 tombstone을 만들었으면, 그다음 compaction에서 죽은 데이터와 묘비를 같이 치우면 될 것처럼 보인다. 하지만 **안 된다.** 그 사이에 끼어드는 것이 **`gc_grace_seconds`(기본값 864000초 = 10일)**다.

이유는 Cassandra의 분산성에 있다. 삭제는 **모든 복제본(replica)에 전파되어야** 한다. 그런데 복제 계수(replication factor)가 3인 클러스터에서 어떤 노드가 다운되어 있는 동안 DELETE가 일어나면, 그 노드는 묘비를 받지 못한다. 만약 묘비를 즉시 회수해버리면 다음과 같은 끔찍한 일이 벌어진다.

```text
  RF=3, 노드 C가 잠시 다운된 상태에서 DELETE 발생:

  시각 t0:  A[데이터] B[데이터] C[데이터]   ← 원래 3 복제본 모두 데이터 보유
  시각 t1:  DELETE 도착 (C는 다운)
            A[묘비]   B[묘비]   C[데이터]   ← C만 묘비를 못 받음
  시각 t2:  (gc_grace 없이) compaction이 A,B의 묘비 즉시 회수
            A[(없음)] B[(없음)] C[데이터]   ← A,B는 깨끗, C엔 데이터 남음
  시각 t3:  C가 복구되어 합류, read repair / 일반 read
            C의 [데이터]가 "A,B엔 없는 최신 정보"로 보임
            → 삭제했던 데이터가 부활!  💀 ZOMBIE
```

이것이 그 유명한 **좀비(zombie) / 유령 데이터 부활** 문제다. 묘비를 너무 일찍 지우면, 삭제 사실을 모르는 노드의 옛 데이터가 다시 살아 돌아온다.

`gc_grace_seconds`는 이 부활을 막는 안전 장치다. **"묘비는 최소 10일간은 살려둔다. 그 안에 repair가 돌아 모든 복제본에 삭제가 전파될 시간을 준다"**는 약속이다. 기본 10일인 이유는, 운영 권장사항이 **`gc_grace_seconds` 주기 내에 최소 한 번은 전체 repair를 완료하라**는 것이기 때문이다(보통 주 1회 repair → 10일 여유면 충분). repair가 묘비를 모든 복제본에 전파한 뒤에야 그 묘비를 안전하게 회수할 수 있다.

> 치명적 안티패턴: "tombstone이 너무 많아 읽기가 느리니 `gc_grace_seconds`를 0으로 낮추자." 이렇게 하면 묘비가 즉시 회수 가능해져 읽기는 빨라지지만, **repair가 삭제를 전파하기 전에 묘비가 사라져 좀비 데이터가 부활할 수 있다.** `gc_grace_seconds`를 줄이려면 **repair가 그 주기 안에 확실히 완료된다는 보장**이 선행되어야 한다. (예외: 단일 노드 개발 환경이나, TTL만 쓰고 명시적 DELETE가 절대 없으며 데이터가 자연 만료되는 특수 테이블에서는 의도적으로 낮추기도 한다. 하지만 프로덕션 RF≥2에서는 신중해야 한다.)

repair가 누락된 노드가 있어도 좀비를 구조적으로 막는 방법이 있다. **incremental repair**와 짝을 이루는 `only_purge_repaired_tombstones` compaction 옵션이 바로 그 방법이다. 이 값이 `true`면 compaction은 **이미 repair된(repaired) SSTable의 묘비만** 회수하고, 아직 repair되지 않은(unrepaired) 데이터의 묘비는 gc_grace가 지났더라도 남겨둔다. "삭제가 모든 복제본에 전파됐다(= repair 완료)"는 사실이 물리적으로 증명된 묘비만 지우므로, 시간(gc_grace)이 아니라 *증거*에 기반해 회수한다. 한 가지 더, compaction은 애초에 **repaired SSTable과 unrepaired SSTable을 같은 compaction에 섞지 않는다**(둘은 분리된 풀로 관리된다). 이 분리 때문에 함정도 생긴다. incremental repair를 돌리지 않는 클러스터에서 `only_purge_repaired_tombstones=true`를 켜면 repaired 데이터가 영영 생기지 않아 묘비가 **전혀** 회수되지 않는다. 이 옵션은 incremental repair 운영과 반드시 한 세트로만 켜야 한다.

### 회수의 또 다른 조건: overlapping (묘비와 데이터가 같은 compaction에 모여야 한다)

`gc_grace_seconds`가 지났다고 묘비가 자동으로 사라지는 게 아니다. **두 번째 조건**이 있고, 이것이 운영 현장에서 가장 많이 간과된다.

**tombstone을 회수하려면, 그 묘비가 가리는(shadow하는) 실제 데이터가 같은 compaction에 함께 들어와야 한다.**

묘비를 지운다는 건 "이 데이터는 죽었다"는 증거를 없애는 일이다. 그런데 그 묘비가 덮어야 할 옛 데이터가 **다른 SSTable**에 아직 살아 있다면, 묘비를 지우는 순간 그 옛 데이터가 다시 유효한 것처럼 보인다. 같은 노드 안에서도 좀비가 생긴다. 그래서 Cassandra는 보수적으로 행동한다. **"이 묘비가 가리는 데이터가 다른 어떤 SSTable에도 남아있지 않다고 확신할 수 있을 때만" 묘비를 버린다.**

```text
  묘비 회수가 막히는 전형적 상황:

  SSTable-A (작은 계층) : [ord-42 DELETE 묘비, gc_grace 지남]
  SSTable-Z (거대 계층) : [ord-42 데이터(옛날 값)]   ← 다른 SSTable에 산 데이터!

  compaction이 A만 처리하면:
    "A의 묘비를 지우면 Z의 옛 데이터가 부활한다" → 묘비 회수 보류!

  A와 Z가 같은 compaction에 들어와야:
    [ord-42 데이터] + [ord-42 묘비] 만남 → 둘 다 제거 가능 ✓
```

여기서 앞서 본 STCS의 함정(함정 3)이 다시 등장한다. STCS는 묘비(보통 작은 새 SSTable에 있음)와 옛 데이터(거대한 오래된 SSTable에 있음)를 **크기가 달라 같은 버킷에 못 넣는다.** 그래서 둘이 영원히 못 만나고 묘비가 쌓인다. 이것이 "디스크를 비웠는데(DELETE 많이 했는데) 공간이 안 줄어드는" 미스터리의 정체다. 묘비도, 그것이 가리는 데이터도 여전히 디스크에 있다.

> Cassandra는 이 overlapping 검사를 보수적으로 한다. 정확히는 묘비의 timestamp보다 오래된 데이터가 든 다른 SSTable이 키 범위상 겹치는지를 확인하고(메타데이터의 min/max timestamp, min/max token 활용), 겹치는 게 없을 때만 회수한다. 그래서 LCS처럼 키 범위가 정돈된 전략이 묘비 회수에도 유리하다.

### Droppable tombstone과 single-SSTable tombstone compaction

묘비가 쌓이기만 하는 STCS 테이블에도 방법은 있다. Cassandra는 **"한 SSTable 안에 만료된(droppable) 묘비가 너무 많으면, 그 SSTable 하나만이라도 정리하는" 특별 compaction**을 제공한다.

관련 파라미터(compaction subproperties):

| 파라미터 | 기본값 | 의미 |
|----------|--------|------|
| `tombstone_threshold` | **0.2** | SSTable 내 droppable 묘비 비율이 이 값(20%)을 넘으면 단일 SSTable tombstone compaction 후보 |
| `tombstone_compaction_interval` | **86400**(1일) | 한 SSTable이 만들어진 뒤 이 시간이 지나야 tombstone compaction 대상이 됨(너무 자주 돌지 않게) |
| `unchecked_tombstone_compaction` | **false** | true면 overlapping 검사를 건너뛰고 더 공격적으로 묘비 정리 시도 |

작동 방식:

- compaction이 끝날 때마다 Cassandra는 각 SSTable의 **droppable tombstone 비율**(gc_grace가 지나 회수 가능한 묘비의 추정 비율)을 추적한다.
- 어떤 SSTable의 이 비율이 `tombstone_threshold`(0.2)를 넘고, 생성된 지 `tombstone_compaction_interval`(1일)이 지났으면, **그 SSTable 하나만을 입력으로 하는 compaction**을 띄운다. 다른 SSTable과 안 합치고 혼자 자기 안의 만료 묘비·데이터만 청소해 새 SSTable로 다시 쓴다.

문제는 위에서 본 overlapping 조건이다. **single-SSTable compaction은 다른 SSTable을 보지 않으므로, 묘비가 가리는 데이터가 다른 SSTable에 있으면 그 묘비를 회수하지 못한다.** 그래서 single-SSTable tombstone compaction을 돌려도 묘비가 안 줄어드는 경우가 흔하다.

`unchecked_tombstone_compaction = true`는 이 안전 검사를 끄고 "overlapping이 있어도 일단 묘비를 회수하라"고 지시한다. 위험을 감수하는 옵션이라 기본은 false다. **gc_grace가 지났고 repair가 확실히 돌고 있다는 전제** 아래에서만 켜는 것이 안전하다. 켜면 묘비가 더 잘 회수되지만, overlapping 데이터를 놓칠 위험(노드 내 부활)을 운영자가 책임진다.

```cql
-- tombstone이 잘 쌓이는 테이블을 더 공격적으로 청소
ALTER TABLE orders WITH compaction = {
    'class': 'SizeTieredCompactionStrategy',
    'tombstone_threshold': 0.1,              -- 10%만 넘어도 청소 시도
    'tombstone_compaction_interval': 3600,   -- 1시간마다 후보 검사
    'unchecked_tombstone_compaction': 'true' -- overlapping 무시 (repair 보장 시)
};
```

### tombstone이 읽기를 죽이는 방식, 그리고 운영 임계값

tombstone의 진짜 위험은 디스크 낭비가 아니라 **읽기 성능 붕괴**다. Cassandra가 행을 읽을 때, 만료된 묘비라도 **스캔은 해야 한다**(이게 죽었다는 걸 확인하려고). 특히 range 조회에서 묘비가 많으면, 살아있는 행 몇 개를 돌려주려고 수만 개의 묘비를 스캔하는 일이 벌어진다.

Cassandra는 이를 방어하는 두 임계값을 둔다(`cassandra.yaml`):

```text
  tombstone_warn_threshold:  1000   (기본) — 한 쿼리가 1000개 넘는 묘비 스캔 시 WARN 로그
  tombstone_failure_threshold: 100000 (기본) — 100000개 넘으면 쿼리를 강제 중단(TombstoneOverwhelmingException)
```

> 실전 사고 시나리오: "큐(queue)를 Cassandra로 구현했더니 어느 날 갑자기 조회가 `TombstoneOverwhelmingException`으로 죽는다." 큐 패턴(한 파티션에 행을 INSERT하고 처리 후 DELETE)은 **묘비 안티패턴의 교과서**다. 같은 파티션에 묘비가 끝없이 쌓이고, 맨 앞 "살아있는" 행 몇 개를 읽으려면 그 앞의 죽은 묘비들을 전부 스캔해야 한다. gc_grace 10일 동안 묘비가 회수도 안 되니 임계값을 금세 넘긴다. **Cassandra를 큐/워크리스트로 쓰지 마라.** 굳이 써야 한다면 파티션을 시간 단위로 쪼개고(버킷팅) TWCS+TTL로 통째 만료시키는 설계가 필요하다([[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]] 참고).

진단에 쓰는 명령:

```bash
# 테이블별 tombstone 통계 (평균/최대 스캔된 묘비 수)
nodetool tablestats keyspace.table | grep -i tombstone
# 예시 출력:
#   Average tombstones per slice (last five minutes): 12.5
#   Maximum tombstones per slice (last five minutes): 4096

# 특정 SSTable의 droppable tombstone 비율, min/max timestamp 확인
sstablemetadata /var/lib/cassandra/data/ks/tbl-xxxx/nb-1-big-Data.db | \
  grep -iE 'Estimated droppable tombstones|Minimum timestamp|Maximum timestamp'
```

`Estimated droppable tombstones` 값이 0.2를 넘는데도 안 줄어들면, overlapping 때문에 회수가 막혔다는 강한 신호다. 이럴 때 LCS 전환이나 `nodetool garbagecollect`(단일/전체 SSTable을 강제로 읽어 묘비·만료 데이터 정리), 최후엔 major compaction(`nodetool compact`)을 고려한다. major compaction은 모든 SSTable을 하나로 합쳐 overlapping을 강제 해소하지만, 거대 단일 SSTable이라는 부작용을 남기므로 STCS에선 신중해야 한다(UCS/LCS는 샤딩·레벨 덕에 부작용이 작다).

---

## 전략 선택: 워크로드에서 전략으로

지금까지의 모든 내부 메커니즘은 결국 하나의 운영 의사결정으로 수렴한다. **이 테이블에 어떤 compaction 전략을 줄 것인가.** 그 답은 항상 **데이터의 접근 패턴(워크로드)**에서 나온다. [[05 - 데이터 모델링 1 - Query First]]의 "쿼리가 모델을 결정한다"는 원칙이 여기서도 이어진다.

### 의사결정 표

| 워크로드 특성 | 권장 전략 | 이유 |
|---------------|-----------|------|
| 시계열 + TTL + append-only (수정/삭제 없음) | **TWCS** | 윈도우 통째 DROP, 만료 회수 비용 0, 시간범위 조회 빠름 |
| read-heavy, 업데이트 잦음, 한 키를 자주 덮어씀 | **LCS** | non-overlapping으로 읽기 SSTable 수 최소, 공간 증폭↓ |
| write-heavy, append 위주, 읽기는 가끔 | **STCS** | 쓰기 증폭 최소, compaction이 쓰기를 방해 안 함 |
| 범용/불확실, 5.0 환경, 운영 단순화 원함 | **UCS** | scaling parameter로 read↔write 다이얼 조정, 샤딩 |
| 삭제(DELETE)가 매우 잦고 묘비 누적 우려 | **LCS** 또는 UCS(leveled쪽) | 묘비-데이터 overlapping 보장이 STCS보다 나음 |
| 큐/워크리스트 패턴 | **(전략 문제 아님)** | 모델 재설계 — Cassandra를 큐로 쓰지 말 것 |

### 전략 전환의 비용

전략을 `ALTER TABLE ... WITH compaction = {...}`로 바꾸는 것은 한 줄이지만, 그 뒤에 **전면 재compaction**이 따른다. 새 전략에 맞게 모든 SSTable이 재배치되므로, 큰 테이블에서는 수 시간~수일의 추가 I/O가 발생하고 그동안 읽기 지연이 출렁인다. 그래서:

- 전환은 트래픽이 적은 시간대에, 한 노드씩(rolling) 진행하는 것이 안전하다.
- `nodetool compactionstats`로 진행 상황을, `nodetool getcompactionthroughput`/`setcompactionthroughput`으로 compaction이 먹는 I/O 대역폭을 조절한다.

```bash
# 현재 진행 중인 compaction과 대기열 확인
nodetool compactionstats -H

# compaction 처리량 제한 (MB/s). 0이면 무제한.
nodetool setcompactionthroughput 64

# 특정 테이블 강제 정리 (전략 전환 후 빨리 정돈하고 싶을 때)
nodetool garbagecollect keyspace table
```

### 결제 시스템에서의 실전 매핑

결제 도메인의 테이블을 이 표에 대입하면 직관이 선다(예시).

- **트랜잭션 이벤트 로그**(이벤트 소싱, append-only, immutable): TWCS는 TTL이 있을 때만. 결제 이벤트는 영구 보존이 흔하므로 TTL이 없다면 STCS 또는 UCS(tiered). 영구 보존 + 가끔 과거 조회면 UCS가 무난.
- **결제 상태 current view**(한 결제의 최신 상태, 자주 업데이트·자주 조회): **LCS**. 한 키를 거듭 덮어쓰고 point 조회가 많으니 LCS의 non-overlapping이 빛난다.
- **정산/리포트 집계 테이블**(주기적 대량 쓰기, 읽기는 배치): STCS 또는 UCS(tiered).
- **멱등성 키/dedup 테이블**(짧은 TTL로 만료, write·read 혼합): TWCS+짧은 TTL이 깔끔. 단 DELETE는 쓰지 말고 TTL에만 의존.

> 함정: "결제 이벤트는 시계열이니까 무조건 TWCS"는 위험하다. TWCS는 **TTL로 통째 만료되는** 데이터에만 옳다. 결제·정산 데이터는 보통 법적 보존 의무로 수년간 살아있어야 하고 정정(amendment)이 발생하면 같은 파티션을 다시 건드린다. 이 경우 TWCS의 전제가 깨지므로 STCS/UCS가 안전하다. **"시계열 모양"이 아니라 "TTL 통째 만료 + append-only"가 TWCS의 진짜 조건이다.**

---

## 핵심 요약

- **compaction은 LSM-tree의 부채 청소다.** append-only·immutable 설계 때문에 한 키가 여러 SSTable에 흩어지고, 죽은 버전과 묘비가 쌓인다. compaction은 이를 병합해 읽기 SSTable 수·공간·묘비를 줄인다.
- **세 가지 증폭(read/write/space)은 트레이드오프 삼각형**이다. 어떤 전략도 셋 다 최소화 못 한다. 전략 선택 = 이 삼각형에서 어느 꼭짓점을 택하느냐.
- **STCS**: 비슷한 크기 SSTable을 `min_threshold`(기본 4)개 모아 병합. 쓰기 증폭↓로 write-heavy에 적합. 단점은 공간 증폭(병합 시 데이터 크기만큼 임시 디스크 필요), 읽기 SSTable 수 상한 없음, 거대 SSTable에 묘비가 갇힘.
- **LCS**: L0~Ln 레벨, 레벨당 용량 10배, L1+ non-overlapping. 키 하나가 레벨당 최대 1개 SSTable에만 존재 → 읽기 SSTable 수 보장, 공간 증폭↓. 대가는 쓰기 증폭(10~30×). read-heavy·잦은 업데이트에 적합. compaction이 쓰기를 못 따라가면 L0 비대로 무너진다.
- **TWCS**: 시간 윈도우별로 SSTable 격리, 윈도우 만료 시 통째 DROP(재작성 0). 시계열+TTL+append-only에 최적. STCS·LCS는 시간 속성을 몰라 신·구 데이터를 섞어 만료 회수를 망친다. 철칙: TTL 필수, 명시적 DELETE 금지, backfill 금지, 윈도우 20~30개.
- **UCS(5.0)**: scaling parameter `w`로 STCS(tiered, w>0)↔LCS(leveled, w<0)를 하나의 연속 다이얼로 통합 + 샤딩으로 거대 SSTable·병렬화 문제 해결. 운영 단순화가 핵심 가치.
- **tombstone GC**: `gc_grace_seconds`(기본 864000=10일)는 repair가 삭제를 모든 복제본에 전파할 시간을 벌어 좀비 데이터 부활을 막는다. 함부로 0으로 낮추지 말 것(repair 완료 보장 시에만). 시간이 아닌 *증거* 기반으로 안전하게 회수하려면 incremental repair + `only_purge_repaired_tombstones`(repaired SSTable의 묘비만 회수)를 한 세트로 쓴다.
- **묘비 회수의 숨은 조건은 overlapping**: 묘비와 그것이 가리는 데이터가 같은 compaction에 모여야 회수된다. STCS에서 묘비가 안 줄어드는 근본 원인.
- **droppable tombstone 제어**: `tombstone_threshold`(0.2), `tombstone_compaction_interval`(1일), `unchecked_tombstone_compaction`(false), single-SSTable tombstone compaction. 읽기 임계값 `tombstone_warn_threshold`(1000)/`tombstone_failure_threshold`(100000).
- **전략 선택은 워크로드에서 나온다.** "시계열 모양"이 아니라 "TTL 통째 만료 + append-only"가 TWCS의 진짜 조건. 큐 패턴은 전략이 아니라 모델 재설계 문제.

## 연결 노트

- [[08 - Storage Engine 내부]] — LSM-tree, memtable/SSTable, immutability. 이 장의 전제. 왜 compaction이 필요한지의 뿌리.
- [[10 - Write Path와 Read Path]] — compaction이 읽기 경로(SSTable 병합, bloom filter, tombstone 스캔)에 어떻게 작용하는지의 큰 그림.
- [[03 - 복제 전략과 데이터센터]] — `gc_grace_seconds`와 repair가 좀비 방지에서 맞물리는 복제·일관성 맥락.
- [[04 - Tunable Consistency]] — 삭제 전파와 read repair, 일관성 수준이 tombstone 가시성에 미치는 영향.
- [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]] — 큐 패턴·과도한 DELETE 등 tombstone 안티패턴.
- [[12 - 운영과 트러블슈팅]] — `nodetool compactionstats`/`tablestats`, tombstone 폭주 진단, 전략 전환 운영.
- [[13 - 실전 Event Sourcing on Cassandra]] — append-only 이벤트 로그와 TWCS/UCS 선택의 실전.
