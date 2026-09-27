---
title: Compaction 전략 - STCS/LCS/TWCS/UCS, tombstone GC
date: 2026-06-26
tags: [cassandra, compaction, tombstone, 학습노트]
---

[[08 - Storage Engine 내부]]에서 Cassandra의 쓰기 경로가 **append-only LSM-tree**라는 점을 살펴봤다. 모든 쓰기는 memtable에 쌓였다가 SSTable로 flush되고, 한 번 디스크에 기록된 SSTable은 **절대 수정되지 않는다(immutable)**. UPDATE와 DELETE도 새로운 셀(cell)을 덧쓰는 동작일 뿐이다. 랜덤 I/O 없이 순차 append만 하므로, 이 설계에서는 쓰기가 매우 빠르다.

하지만 이 방식에는 대가가 따른다. **시간이 지나면 하나의 파티션 키가 수십, 수백 개의 SSTable에 흩어진다.** 같은 행(row)을 여러 번 UPDATE하면 그 조각이 여러 파일에 나뉘어 저장되고, 읽을 때마다 조각을 모아 최신 버전을 다시 구성해야 한다. 삭제도 데이터를 실제로 지우지 않고 "삭제되었음"을 기록하는 표식(tombstone)을 추가할 뿐이다. 그래서 디스크에는 삭제된 데이터와 tombstone이 함께 쌓인다.

**Compaction은 이렇게 쌓인 데이터를 정리하는 백그라운드 작업이다.** 여러 SSTable을 읽어 병합(merge)하면서 같은 키의 오래된 버전을 버리고 만료된 tombstone을 제거한다. 그런 다음 정리된 결과를 새 SSTable로 쓰고 원본을 삭제한다. 이 장에서는 이 정리 작업이 *어떤 전략으로* 이루어지는지, 그리고 전략 선택이 왜 Cassandra 운영에서 가장 중요한 의사결정 가운데 하나인지를 자세히 살펴본다.

> [!note] RDB였다면
> PostgreSQL의 `VACUUM`이나 InnoDB의 purge thread가 가장 가까운 대응 기능이다.
>
> 하지만 결정적인 차이가 있다. RDB는 페이지를 **제자리에서 수정(update-in-place)** 하므로, 죽은 튜플을 회수하는 것이 정리 작업의 전부다. Cassandra의 compaction은 정리뿐 아니라 **읽기 성능을 직접 좌우하는 데이터 재배치(re-organization)** 까지 담당한다. 그래서 compaction 전략을 잘못 고르면 디스크에 불필요한 데이터가 남는 데서 그치지 않고, 읽기 지연이 수십 배로 늘어난다.

---

## 왜 Compaction이 필요한가: 흩어진 키의 비용

먼저 compaction이 없는 상황을 생각해 보자. 결제 시스템에서 주문 하나의 상태가 여러 번 바뀐다고 하자.

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

이제 `ord-42`를 한 번 읽을 때 어떤 일이 일어나는지 따라가 보자. 디스크에는 이 행 하나의 조각이 네 곳에 흩어져 있다.

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

Cassandra는 네 SSTable을 **모두** 열고, 각 파일에서 `ord-42`의 조각을 꺼낸 뒤 **셀 단위 timestamp**로 최신 값을 골라(last-write-wins) 행을 다시 구성한다. SSTable이 많을수록 읽기 경로에서 확인해야 할 파일이 늘어난다. 각 파일은 가장 먼저 bloom filter로 걸러지고, 이를 통과한 파일만 partition index 조회와 디스크 seek로 이어진다. 이것을 **읽기 증폭(read amplification)** 이라고 한다.

여기에 세 가지 비용이 더해진다.

1. **병합 비용**: 읽을 때마다 흩어진 버전을 모으는 데 CPU와 메모리를 쓴다.
2. **tombstone 누적**: DELETE는 tombstone을 추가할 뿐이다. 삭제된 데이터와 tombstone이 SSTable마다 쌓이므로, 읽을 때 "이미 삭제되었지만 아직 스캔해야 하는" 셀이 늘어난다.
3. **공간 낭비**: `CREATED`, `PAID`, `SHIPPED`는 이미 의미가 없는 이전 버전인데도 디스크를 차지한다. 논리적으로는 행 하나지만 물리적으로는 네 벌이 저장되어 있다.

compaction은 이 네 SSTable을 하나로 합치면서 `DELIVERED`만 남기고 나머지를 버린다. 그러면 읽을 때 SSTable 하나만 보면 되고, 이전 버전과 만료된 tombstone도 사라진다.

```text
  compaction 후:
  ┌──────────────────────────────────────────┐
  │ SSTable-new                              │
  │  order_id=ord-42, status=DELIVERED,      │
  │  amount=10000   (낡은 버전 3개 제거됨)    │
  └──────────────────────────────────────────┘
```

> **읽기 성능은 "키 하나를 읽기 위해 확인해야 하는 SSTable 수"에 직접 비례한다.** compaction 전략은 결국 이 숫자를 어떻게 통제하느냐의 문제다. STCS는 이 숫자를 느슨하게 관리하고, LCS는 강하게 보장하며, TWCS는 시간 축으로 데이터를 분리한다. UCS는 STCS와 LCS 사이의 위치를 설정값으로 조절한다.

### compaction의 세 가지 증폭(amplification) 사이의 트레이드오프

compaction 전략을 이해하는 가장 좋은 기준은 **세 가지 증폭 사이의 트레이드오프**다. 어떤 전략도 세 가지를 모두 최소화할 수는 없다. 하나를 줄이면 다른 하나가 늘어난다.

| 증폭 | 정의 | 누가 손해 보나 |
|------|------|----------------|
| 읽기 증폭(read amp) | 키 하나를 읽기 위해 접근하는 SSTable/디스크 수 | 읽기 지연 |
| 쓰기 증폭(write amp) | 논리적 1바이트가 compaction으로 디스크에 총 몇 번 쓰이는지 | 디스크 수명·I/O 대역폭 |
| 공간 증폭(space amp) | 논리 데이터 크기 대비 실제 디스크 점유 비율 | 디스크 용량 |

- **STCS**: 쓰기 증폭은 낮고, 읽기 증폭과 공간 증폭은 높다.
- **LCS**: 읽기 증폭과 공간 증폭은 낮고, 쓰기 증폭은 높다.
- **TWCS**: 시계열 데이터에 한해서는 세 가지가 모두 낮지만, 시계열이 아닌 데이터에서는 이 장점이 사라진다.
- **UCS**: 파라미터(scaling parameter)로 STCS와 LCS 사이의 어느 위치든 지정할 수 있다.

이 세 가지 증폭을 기억해 두고 나머지를 읽으면, 각 전략을 "세 증폭 가운데 무엇을 우선했는가"로 정리할 수 있다.

---

## STCS: 비슷한 크기끼리 묶는다

**SizeTieredCompactionStrategy(STCS)** 는 Cassandra에서 가장 오래된 기본 전략이고, 발상도 가장 직관적이다. 이름 그대로 **비슷한 크기(size-tiered)의 SSTable을 한 묶음(bucket)으로 모아 병합한다.**

### 내부 메커니즘: 버킷팅(bucketing)

STCS는 SSTable을 크기에 따라 버킷으로 나눈다. 기본 동작은 다음과 같다.

- SSTable 크기를 보고, 평균 크기의 `bucket_low`(기본 0.5)배에서 `bucket_high`(기본 1.5)배 범위에 드는 SSTable을 같은 버킷으로 묶는다. 즉 "대략 같은 크기"의 SSTable끼리 모인다.
- 한 버킷에 SSTable이 `min_threshold`(기본 **4**)개 이상 쌓이면 그 버킷 전체를 병합한다.
- 한 번에 묶는 최대 개수는 `max_threshold`(기본 **32**)이다.

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

작은 SSTable 4개가 모이면 중간 크기 SSTable 하나가 되고, 중간 크기 4개가 모이면 큰 SSTable 하나가 된다. 시간이 지나면 디스크에는 **크기 계층(tier)이 자연스럽게 생긴다.** 작은 SSTable 몇 개, 중간 크기 몇 개, 아주 큰 SSTable 한두 개가 함께 존재하게 된다.

### 왜 write-heavy 워크로드에 유리한가

STCS의 쓰기 증폭이 낮은 이유는 단순하다. **데이터 조각 하나가 compaction에 참여하는 횟수가 적기 때문이다.** 작은 SSTable이 만들어진 뒤 한 번 합쳐져 중간 크기가 되고, 다시 한 번 합쳐져 큰 SSTable이 된다. 즉 크기 계층을 한 단계 올라갈 때마다 딱 한 번씩만 다시 쓰인다. 계층 수가 로그 스케일로 늘어나므로 전체 재작성 횟수도 적다.

로그 적재, 이벤트 수집, append 위주 테이블처럼 쓰기가 몰리는 상황에서는 이 낮은 쓰기 증폭이 결정적이다. compaction이 쓰기 I/O를 거의 방해하지 않기 때문이다.

### STCS의 세 가지 함정

**함정 1, 공간 증폭(space amplification): 일시적으로 큰 여유 디스크가 필요하다.**

가장 큰 버킷을 병합하는 경우를 생각해 보자. 100GB짜리 SSTable 4개가 한 버킷에 모이면, STCS는 이 400GB를 읽어 새 SSTable 하나로 쓴다. **병합이 끝나 원본을 지우기 전까지는 입력 400GB와 출력(최대 400GB)이 디스크에 함께 존재한다.** 최악의 경우 데이터 크기만큼의 여유 공간이 추가로 필요하다.

> [!warning] 흔한 운영 사고
> "디스크 사용률이 50%인데 compaction이 `No space left`로 실패한다." STCS에서 compaction 한 번에 필요한 임시 공간을 고려하지 않은 경우다. STCS 테이블이 디스크의 50% 이상을 차지하면, 가장 큰 버킷의 major compaction을 수행할 공간이 없어 작업이 멈출 수 있다. **STCS는 데이터 크기만큼 디스크 여유 공간을 남겨 두는 것을 전제로 한다.**

**함정 2, 읽기 증폭: 하나의 키가 여러 크기 계층에 동시에 존재할 수 있다.**

STCS는 SSTable 사이에 키 범위가 겹치지 않는다는 점을 전혀 보장하지 않는다. 같은 키 `ord-42`가 작은 SSTable, 중간 크기 SSTable, 아주 큰 SSTable에 동시에 흩어져 있을 수 있고, 읽으려면 이들을 모두 확인해야 한다. **bloom filter가 대부분을 걸러 주지만, 최악의 경우 읽기 한 번에 확인해야 할 SSTable 수에는 상한이 없다.** read-heavy 워크로드에서 STCS의 p99 읽기 지연이 들쭉날쭉한 이유가 여기에 있다.

**함정 3, 거대 SSTable과 좀비 tombstone.**

SSTable이 클수록 다른 SSTable과 병합될 기회가 줄어든다. 비슷한 크기의 SSTable이 4개나 모이기 어렵기 때문이다. 그 결과 **오래된 데이터와, 그 데이터를 덮어쓰거나 삭제한 tombstone이 서로 다른 크기 계층에 머물러 끝내 함께 병합되지 못하는** 상황이 생긴다.

tombstone은 자신이 가리는 데이터(shadowed 데이터)와 같은 compaction에 참여해야만 둘 다 제거되는데(뒤에서 자세히 설명한다), STCS의 크기 기반 묶음은 이를 보장하지 못한다. 큰 테이블에서 tombstone이 쌓이고 읽기가 느려지는 전형적인 증상이다.

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

## LCS: 레벨로 읽기 SSTable 수를 제한한다

**LeveledCompactionStrategy(LCS)** 는 STCS의 가장 큰 약점인 "키 하나를 읽기 위해 확인할 SSTable 수에 상한이 없다"는 문제를 직접 해결한다. Google의 LevelDB에서 영감을 받은 전략으로, **읽을 때 접근하는 SSTable 수에 이론적인 상한을 보장한다.**

### 내부 메커니즘: 레벨과 non-overlapping 불변식

LCS는 SSTable을 L0, L1, L2, ... Ln과 같은 **레벨(level)** 로 구성한다.

핵심 불변식은 세 가지다.

1. **SSTable 크기가 고정된다**(기본 `sstable_size_in_mb` = **160MB**). 모든 SSTable의 크기가 대략 같다.
2. **L1 이상에서는 같은 레벨 안의 SSTable끼리 키 범위가 절대 겹치지 않는다(non-overlapping).** 즉 L1 전체가 정렬된 하나의 큰 run처럼 동작한다.
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

중요한 것은 이 구조가 주는 보장이다. 키 범위가 겹치지 않으므로 **L1 이상의 각 레벨에서 어떤 키든 최대 한 개의 SSTable에만 존재한다.** 따라서 키 하나를 읽을 때 확인해야 할 SSTable 수는 대략 **L0의 SSTable 수 + 레벨 수**로 제한된다.

**L0를 잘 관리하면 보통 90% 이상의 경우 키 하나가 단 하나의 SSTable에만 존재**하며, 최악의 경우에도 레벨 수(보통 한 자릿수) 정도다. STCS는 상한이 없고 LCS는 상한이 있다는 점이 두 전략을 가르는 결정적인 차이다.

### compaction이 일어나는 방식

L0에 SSTable이 일정 개수(보통 4개) 쌓이면 LCS가 동작한다. L0의 SSTable과, 이들과 키 범위가 겹치는 L1의 SSTable을 함께 읽어 병합하고, 결과를 다시 160MB 단위로 잘라 L1에 쓴다. L1이 용량(약 10개)을 넘으면 넘친 SSTable 하나를 골라, 그 SSTable과 키 범위가 겹치는 L2의 SSTable과 병합해 L2로 내린다. 이 과정이 레벨을 따라 연쇄적으로(cascade) 이어진다.

```text
  L1이 넘침:
  L1 [a][b][c]...[k]  ← 한 개 초과
       │ (b를 골라 L2로 내린다)
       ▼
  L2 의 b와 키범위 겹치는 [b1][b2] 와 병합
       ▼
  L2 [a][b1'][b2'][c]...  (b의 데이터가 L2에 합쳐짐)
```

### 왜 read-heavy 워크로드에 유리한가, 그리고 그 대가

LCS가 read-heavy 워크로드에 유리한 이유는 분명하다. **읽을 때 확인할 SSTable 수가 적고 예측할 수 있다.** non-overlapping 불변식 덕분에 키 하나가 거의 항상 레벨마다 하나의 SSTable에만 있으므로, bloom filter와 함께 쓰면 대부분의 점 조회(point lookup)가 SSTable 1~2개만 읽는다.

공간 증폭도 작다. 같은 키의 중복 버전이 여러 레벨에 오래 남지 않고 빠르게 한 레벨로 합쳐지므로, 디스크 점유량이 논리 데이터 크기에 가깝다(보통 10% 안팎의 오버헤드).

대가는 **쓰기 증폭(write amplification)** 이다. non-overlapping 불변식을 유지하려면 새 데이터가 들어올 때마다 그 데이터와 키 범위가 겹치는 하위 레벨 SSTable을 계속 다시 써야 한다.

데이터 조각 하나가 L0 → L1 → L2 → ... 로 내려가면서 **레벨마다 한 번씩 재작성**되고, 단계마다 키 범위가 겹치는 이웃 SSTable까지 함께 다시 쓴다. 그래서 쓰기 증폭은 보통 **STCS의 2배 이상**이고, 원본 데이터 대비로 보면 10~30배에 이르는 경우도 흔하다.

> [!warning] 함정
> write-heavy 테이블에 LCS를 쓰면 **compaction이 쓰기 속도를 따라잡지 못한다(compaction falls behind).** L0에 SSTable이 계속 쌓여 non-overlapping 보장이 깨지고 L0가 커지면, LCS의 읽기 이점은 사라지고 쓰기 증폭만 남는 최악의 상태가 된다.
>
> `nodetool tablestats`에서 "SSTables in each level: [수십/4/40/...]"처럼 L0 숫자가 비정상적으로 크다면 이 상태라는 신호다. **LCS는 compaction이 쓰기 처리량을 따라갈 수 있을 때만 효과가 있다.**

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

### STCS와 LCS 비교

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

## TWCS: 시간을 기준으로 나눈다

**TimeWindowCompactionStrategy(TWCS)** 는 STCS나 LCS와 전혀 다른 질문에서 출발한다. "데이터에 **시간 순서**라는 구조가 있고, 오래된 데이터가 **TTL로 한꺼번에 만료**된다면, 이 구조를 compaction에 활용할 수 없을까?"

센서 측정값, 로그, 결제 이벤트 스트림, 메트릭 같은 시계열(time-series) 데이터에는 전형적인 패턴이 있다. **데이터는 (거의) 시간순으로 들어오고, 한 번 쓰면 절대 수정하지 않으며(immutable append), 일정 기간이 지나면 TTL로 사라진다.**

### 왜 STCS와 LCS는 시계열 데이터에 맞지 않는가

TWCS를 이해하려면 여기서 출발해야 한다. STCS와 LCS는 데이터의 **시간 속성을 전혀 고려하지 않는다.** 두 전략은 크기나 레벨만 본다. 그 결과는 다음과 같다.

```text
  STCS가 시계열을 병합하면:

  [오늘 데이터]──┐
  [어제 데이터]──┼─→ 크기가 비슷하니 한 버킷! ─→ [오늘+어제+그제 섞인 SSTable]
  [그제 데이터]──┘

  문제: 이제 한 SSTable 안에 여러 날짜가 뒤섞여 있다.
```

- **TTL 만료의 비효율**: 어제 데이터의 TTL이 모두 만료되었다고 하자. 그런데 만료된 데이터가 오늘, 그제 데이터와 한 SSTable에 섞여 있다. 살아 있는 데이터가 함께 있으므로 그 SSTable을 통째로 버릴 수 없다. **만료된 데이터를 회수하려면 살아 있는 데이터까지 다시 읽어 재작성하는 compaction을 해야 한다.** 시계열 테이블이 끝없이 compaction을 돌리며 I/O를 소모하는 전형적인 원인이다.
- **읽기 비효율**: 시계열 조회는 보통 "최근 1시간", "오늘"처럼 시간 범위를 지정한다. 그런데 한 SSTable에 모든 날짜가 섞여 있으면 "오늘"을 조회할 때도 예전 데이터가 담긴 SSTable을 모두 읽게 된다.
- **tombstone과 만료 데이터 누적**: TTL이 만료되면 Cassandra는 만료된 셀을 tombstone처럼 취급한다. 이 셀이 살아 있는 데이터와 섞여 있으면 회수되지 못하고 계속 쌓인다.

### TWCS의 메커니즘: 윈도우별 격리

TWCS는 단순하면서도 강력한 규칙을 따른다. **각 SSTable은 하나의 시간 윈도우(time window)에만 속한다.** 윈도우 크기는 `compaction_window_unit`(예: HOURS, DAYS)과 `compaction_window_size`(예: 1)로 정한다.

- 현재(active) 윈도우 안에서는 STCS 방식으로 작은 SSTable을 합친다.
- 윈도우가 끝나면 그 윈도우의 모든 SSTable을 **하나의 SSTable로 major compaction** 한 뒤 **봉인(seal)** 한다. 봉인된 윈도우의 SSTable은 이후 다른 윈도우의 SSTable과 다시 섞이지 않는다.

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

효과는 분명하다. **한 윈도우의 데이터가 모두 TTL로 만료되면 그 윈도우의 SSTable을 통째로 `DROP`한다. 단 한 바이트도 재작성하지 않는다.** 파일 하나를 `rm`하는 것과 같다. 만료된 데이터를 회수하는 비용이 사실상 0이므로, 시계열 데이터에서는 TWCS가 STCS나 LCS보다 훨씬 유리하다.

읽기도 빨라진다. "오늘"을 조회하면 오늘 윈도우의 SSTable만 보면 되고, 다른 윈도우의 SSTable은 bloom filter와 min/max timestamp 메타데이터로 통째로 건너뛴다.

다만 중요한 예외가 하나 있다. **완전히 만료된 SSTable이라도 timestamp 기준으로 다른 SSTable과 겹치면 기본적으로 통째로 DROP되지 않는다.** 뒤에서 살펴볼 overlapping 안전장치가 여기서도 동작하기 때문이다. Cassandra는 "이 SSTable을 지웠을 때 다른 SSTable에 남은 더 오래된 데이터가 되살아나지 않는가"를 보수적으로 따진다.

그래서 backfill이나 out-of-order 쓰기 때문에 윈도우 사이의 timestamp가 뒤섞이면, 만료된 SSTable이 삭제되지 못하고 쌓인다(아래 철칙 3이 중요한 또 다른 이유다).

이 검사를 끄고 "완전히 만료된 SSTable은 겹침 여부와 관계없이 즉시 삭제"하도록 만드는 옵션이 `unsafe_aggressive_sstable_expiration`(기본 `false`)이다. 이름 그대로 위험을 감수하는 옵션이므로, append-only이고 out-of-order 쓰기가 절대 없다고 확신할 수 있는 순수 시계열 데이터에서만 켜는 것이 안전하다.

### TWCS 사용 원칙

TWCS는 **올바르게 쓰면 효과가 매우 크지만, 잘못 쓰면 심각한 문제를 일으키는** 전략이다. 다음 규칙을 어기면 TWCS의 이점이 모두 사라진다.

> [!warning] 철칙 1
> **데이터에는 반드시 TTL이 있어야 한다.** TTL이 없으면 TWCS의 윈도우 SSTable은 봉인된 채 계속 쌓이기만 한다. 통째로 DROP할 일이 없으므로 디스크 사용량이 끝없이 늘어난다.

> [!warning] 철칙 2
> **명시적 DELETE를 (거의) 하지 않는다.** TWCS는 "데이터가 오직 TTL 만료로만 사라진다"는 전제를 둔다. 봉인된 예전 윈도우의 데이터를 DELETE하면 tombstone이 생기는데, 이 tombstone은 새 윈도우에 들어간다. 예전 윈도우의 데이터는 다른 SSTable에 격리되어 있으므로 tombstone과 함께 병합될 일이 없다. 결국 삭제는 동작하지 않고 tombstone만 쌓인다.

> [!warning] 철칙 3
> **과거 시점으로 쓰지 않는다(no out-of-order / backfill writes).** TWCS는 데이터의 write timestamp를 보고 윈도우를 정한다. 오래된 timestamp로 backfill하면 이미 봉인되었어야 할 예전 윈도우에 새 SSTable이 추가되어 봉인이 깨지고, 그 윈도우가 다시 active 윈도우처럼 compaction 대상이 된다. 시계열의 시간 순서 가정이 무너지는 것이다.

> [!warning] 함정: 윈도우 개수
> 윈도우 크기를 너무 작게 잡으면(예: TTL은 1년인데 윈도우는 1시간) SSTable이 수천 개로 늘어난다. **전체 TTL 기간에 윈도우가 20~30개 정도** 생기도록 잡는 것을 권장한다. TTL이 30일이면 윈도우는 1일이 적당하다.

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

> [!note] 결제 시스템 관점
> 이벤트 소싱([[13 - 실전 Event Sourcing on Cassandra]])에서 "최근 N일의 이벤트만 빠르게 조회하고 오래된 이벤트는 cold storage로 보낸다" 같은 패턴에 TWCS가 잘 맞는다. 단, 이벤트를 **수정하거나 삭제하지 않는 append-only** 구조일 때만 그렇다. 결제 이벤트는 immutable이라 TWCS와 잘 맞지만, "정정(amendment)을 DELETE+INSERT로 구현"하는 순간 철칙 2를 어기게 된다는 점에 유의해야 한다.

---

## UCS: 파라미터 하나로 통합한다 (Cassandra 5.0)

Cassandra 5.0에서 가장 큰 compaction 변화는 **UnifiedCompactionStrategy(UCS)** 의 도입이다. UCS의 발상은 **STCS와 LCS가 사실 같은 알고리즘의 양 끝**이라는 것이다. 두 전략은 모두 "여러 SSTable을 묶어 병합한다"는 골격이 같고, 차이는 "얼마나 적극적으로 묶느냐"라는 파라미터 하나뿐이다. 그렇다면 두 전략을 **연속적으로 조절할 수 있는 파라미터 하나(scaling parameter)** 로 통합할 수 있다.

### scaling parameter `w`: STCS와 LCS를 잇는 연속체

UCS의 핵심은 레벨별 **scaling parameter `w`** 이다. `w`는 각 레벨이 허용하는 SSTable run의 개수를 결정한다.

- **`w > 0` (양수): tiered, STCS처럼 동작한다.** `w`가 클수록 한 레벨에 SSTable을 더 많이 쌓았다가 한꺼번에 묶는다. 쓰기 증폭은 줄고 읽기 증폭은 늘어난다.
- **`w < 0` (음수): leveled, LCS처럼 동작한다.** `|w|`가 클수록 레벨당 run 수를 적게 유지하려고 더 자주 병합한다. 읽기 증폭은 줄고 쓰기 증폭은 늘어난다.
- **`w = 0`: 두 동작의 경계다.**

```text
  쓰기증폭 ←─────────────────────────────────→ 읽기증폭
   많음                                          많음
  
  w = -2   w = -1    w = 0     w = +2   w = +4 ...
   │         │         │          │        │
  강한 LCS  약한 LCS  경계      약한 STCS 강한 STCS
   │                                          │
  읽기 빠름                                쓰기 빠름
```

운영자는 "STCS냐 LCS냐"라는 양자택일 대신, **`w` 하나로 읽기와 쓰기 사이의 트레이드오프 곡선에서 원하는 지점을 고른다.** 워크로드가 바뀌어도 전략 전체를 교체할 필요 없이 `w`만 조정하면 된다. 레벨마다 다른 `w`를 줄 수도 있으므로(`scaling_parameters`에 레벨별로 지정), "하위 레벨은 tiered로 두어 쓰기를 빠르게, 상위 레벨은 leveled로 두어 읽기를 빠르게" 하는 혼합 전략도 가능하다.

### 샤딩(sharding): 거대 SSTable 문제의 해법

UCS가 가져온 두 번째 큰 변화는 **샤딩**이다. STCS에는 오래된 문제가 있었다. 데이터가 커지면 SSTable 하나가 수백 GB까지 커지고, 그러면 병합할 때 큰 임시 공간이 필요하며, compaction 한 번이 오래 걸리고, 병렬로 처리할 수도 없다.

UCS는 토큰 범위를 여러 **shard**로 나누고, 각 레벨의 데이터를 shard 경계에서 잘라 **적당한 크기의 SSTable 여러 개**로 관리한다. 그 결과는 다음과 같다.

- SSTable 하나가 끝없이 커지지 않는다(공간 증폭과 임시 공간 요구가 통제된다).
- 서로 다른 shard의 compaction을 **병렬로** 실행할 수 있어 멀티코어를 활용한다.
- compaction 단위가 작아지므로 compaction 한 번이 빨리 끝나고, 필요한 디스크 여유 공간도 줄어든다.

샤딩 정도는 `target_sstable_size`(목표 SSTable 크기, 기본 약 1GiB 안팎)와 데이터 양에 따라 자동으로 조절된다. 데이터가 적을 때는 적게 나누고, 데이터가 많아지면 자동으로 더 잘게 나눈다.

### UCS의 의미: 운영 단순화

UCS의 진짜 가치는 알고리즘이 새롭다는 데 있지 않고 **운영을 단순하게 만든다**는 데 있다. 이전에는 "이 테이블은 읽기가 많으니 LCS, 저 테이블은 쓰기가 많으니 STCS, 시계열은 TWCS"처럼 테이블마다 전략을 고민해야 했다. 워크로드가 바뀌면 `ALTER TABLE`로 전략 전체를 교체하고, 그에 따른 전면 재compaction 부담도 감수해야 했다. UCS는 이 결정을 연속적인 파라미터 조정으로 바꾼다.

> [!caution] 추측 주의
> UCS는 5.0에서 정식 도입되었지만, 모든 워크로드에서 기존 전략보다 무조건 나은 만능 전략은 아니다. 순수 시계열+TTL 패턴(통째로 DROP)처럼 TWCS가 특히 강한 영역은 여전히 TWCS의 고유 영역이라고 보는 것이 안전하다. 5.0 환경에서는 신규 테이블의 범용 기본값으로 UCS를 고려하되, 이미 검증된 워크로드(특히 시계열)는 기존 전략을 유지하는 보수적인 접근이 합리적이다. 조직별 권장 기본값은 실측으로 확정해야 한다.

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

## Tombstone과 GC: 삭제된 데이터를 다루는 방법

이제 compaction에서 가장 미묘하고 사고도 잦은 영역인 **tombstone 회수**를 살펴본다. 8장(Storage Engine 내부)에서 봤듯이 Cassandra에서 DELETE는 데이터를 지우지 않고 **tombstone을 쓰는** 동작이다. tombstone은 "이 시각 이후 이 데이터는 삭제되었다"는 표식이다. 문제는 이 tombstone을 *언제* 실제로 치울 수 있느냐다.

### tombstone의 종류

먼저 Cassandra가 만드는 tombstone이 한 종류가 아니라는 점을 알아야 한다.

- **Cell tombstone**: 특정 컬럼 하나를 삭제한다(`UPDATE ... SET col = null` 또는 컬럼 DELETE).
- **Row tombstone**: 행 하나를 삭제한다(`DELETE FROM t WHERE pk=.. AND ck=..`).
- **Range tombstone**: 범위를 삭제한다(`DELETE ... WHERE pk=.. AND ck > ..`). tombstone 하나가 여러 행을 덮는다.
- **Partition tombstone**: 파티션 전체를 삭제한다(`DELETE FROM t WHERE pk=..`).
- **TTL 만료**: TTL이 지난 셀은 만료 시점에 자동으로 tombstone과 같게 취급된다(expired cell, 만료 tombstone이라고 부른다). **TTL을 쓰는 것도 결국 tombstone을 대량으로 만드는 일**이다.

### 왜 즉시 지우지 못하는가: gc_grace_seconds

데이터를 DELETE해서 tombstone을 만들었다면, 다음 compaction에서 삭제된 데이터와 tombstone을 함께 치우면 될 것처럼 보인다. 하지만 **그렇게 할 수 없다.** 그 사이에 **`gc_grace_seconds`(기본값 864000초 = 10일)** 라는 유예 기간이 있다.

이유는 Cassandra가 분산 시스템이라는 데 있다. 삭제는 **모든 복제본(replica)에 전파되어야** 한다. 그런데 복제 계수(replication factor)가 3인 클러스터에서 어떤 노드가 다운된 동안 DELETE가 일어나면, 그 노드는 tombstone을 받지 못한다. 이때 tombstone을 즉시 회수해 버리면 다음과 같은 심각한 문제가 생긴다.

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

이것이 잘 알려진 **좀비(zombie) 데이터, 또는 유령 데이터 부활** 문제다. tombstone을 너무 일찍 지우면 삭제 사실을 모르는 노드에 남은 예전 데이터가 다시 살아난다.

`gc_grace_seconds`는 이런 부활을 막는 안전장치다. 즉 **"tombstone은 최소 10일 동안 유지한다. 그 안에 repair가 실행되어 모든 복제본에 삭제가 전파될 시간을 준다"** 는 약속이다.

기본값이 10일인 이유는 운영 권장사항이 **`gc_grace_seconds` 주기 안에 전체 repair를 최소 한 번은 완료하라**는 것이기 때문이다(보통 repair를 주 1회 실행하므로 10일이면 여유가 충분하다). repair가 tombstone을 모든 복제본에 전파한 뒤에야 그 tombstone을 안전하게 회수할 수 있다.

> [!caution] 치명적 안티패턴
> "tombstone이 너무 많아 읽기가 느리니 `gc_grace_seconds`를 0으로 낮추자." 이렇게 하면 tombstone을 즉시 회수할 수 있어 읽기는 빨라지지만, **repair가 삭제를 전파하기 전에 tombstone이 사라져 좀비 데이터가 되살아날 수 있다.** `gc_grace_seconds`를 줄이려면 먼저 **repair가 그 주기 안에 확실히 완료된다는 보장**이 있어야 한다.
>
> (예외: 단일 노드 개발 환경이나, TTL만 쓰고 명시적 DELETE가 전혀 없어 데이터가 자연히 만료되는 특수한 테이블에서는 의도적으로 낮추기도 한다. 하지만 RF≥2인 프로덕션 환경에서는 신중해야 한다.)

repair가 누락된 노드가 있어도 좀비 데이터를 구조적으로 막는 방법이 있다. **incremental repair**와 함께 쓰는 `only_purge_repaired_tombstones` compaction 옵션이다. 이 값이 `true`이면 compaction은 **이미 repair된(repaired) SSTable의 tombstone만** 회수하고, 아직 repair되지 않은(unrepaired) 데이터의 tombstone은 gc_grace가 지났더라도 남겨 둔다.

"삭제가 모든 복제본에 전파되었다(= repair 완료)"는 사실이 실제로 확인된 tombstone만 지우므로, 시간(gc_grace)이 아니라 *증거*를 근거로 회수하는 셈이다.

덧붙여, compaction은 처음부터 **repaired SSTable과 unrepaired SSTable을 같은 compaction에 섞지 않는다**(둘은 분리된 풀로 관리된다).

이 분리 때문에 함정도 생긴다. incremental repair를 실행하지 않는 클러스터에서 `only_purge_repaired_tombstones=true`를 켜면 repaired 데이터가 전혀 생기지 않으므로 tombstone이 **전혀** 회수되지 않는다. 이 옵션은 반드시 incremental repair 운영과 함께 켜야 한다.

### 회수의 또 다른 조건: overlapping (tombstone과 데이터가 같은 compaction에 참여해야 한다)

`gc_grace_seconds`가 지났다고 tombstone이 자동으로 사라지지는 않는다. **두 번째 조건**이 있으며, 운영 현장에서 가장 많이 놓치는 부분이 바로 이 조건이다.

**tombstone을 회수하려면, 그 tombstone이 가리는(shadow하는) 실제 데이터가 같은 compaction에 함께 참여해야 한다.**

tombstone을 지운다는 것은 "이 데이터는 삭제되었다"는 증거를 없애는 일이다. 그런데 tombstone이 덮어야 할 예전 데이터가 **다른 SSTable**에 아직 남아 있다면, tombstone을 지우는 순간 그 예전 데이터가 다시 유효한 것처럼 보인다. 같은 노드 안에서도 좀비 데이터가 생기는 것이다.

그래서 Cassandra는 보수적으로 동작한다. **"이 tombstone이 가리는 데이터가 다른 어떤 SSTable에도 남아 있지 않다고 확신할 수 있을 때만" tombstone을 버린다.**

```text
  묘비 회수가 막히는 전형적 상황:

  SSTable-A (작은 계층) : [ord-42 DELETE 묘비, gc_grace 지남]
  SSTable-Z (거대 계층) : [ord-42 데이터(옛날 값)]   ← 다른 SSTable에 산 데이터!

  compaction이 A만 처리하면:
    "A의 묘비를 지우면 Z의 옛 데이터가 부활한다" → 묘비 회수 보류!

  A와 Z가 같은 compaction에 들어와야:
    [ord-42 데이터] + [ord-42 묘비] 만남 → 둘 다 제거 가능 ✓
```

여기서 앞에서 본 STCS의 함정 3이 다시 등장한다. tombstone은 보통 작은 새 SSTable에 있고 예전 데이터는 크고 오래된 SSTable에 있는데, STCS는 **크기가 다른 두 SSTable을 같은 버킷에 넣지 못한다.** 그래서 둘이 끝내 함께 병합되지 못하고 tombstone이 쌓인다. "DELETE를 많이 해서 디스크를 비웠는데 공간이 줄지 않는" 현상의 원인이 이것이다. tombstone도, tombstone이 가리는 데이터도 여전히 디스크에 남아 있다.

> Cassandra는 이 overlapping 검사를 보수적으로 수행한다. 정확히는 tombstone의 timestamp보다 오래된 데이터가 담긴 다른 SSTable이 키 범위상 겹치는지를 확인하고(메타데이터의 min/max timestamp와 min/max token을 활용한다), 겹치는 SSTable이 없을 때만 회수한다. 그래서 LCS처럼 키 범위가 정돈된 전략이 tombstone 회수에도 유리하다.

### Droppable tombstone과 single-SSTable tombstone compaction

tombstone이 쌓이기만 하는 STCS 테이블에도 대처 방법은 있다. Cassandra는 **"한 SSTable 안에 만료된(droppable) tombstone이 너무 많으면 그 SSTable 하나만이라도 정리하는" 특별한 compaction**을 제공한다.

관련 파라미터(compaction subproperties)는 다음과 같다.

| 파라미터 | 기본값 | 의미 |
|----------|--------|------|
| `tombstone_threshold` | **0.2** | SSTable 안의 droppable tombstone 비율이 이 값(20%)을 넘으면 단일 SSTable tombstone compaction 후보가 된다 |
| `tombstone_compaction_interval` | **86400**(1일) | SSTable이 만들어진 뒤 이 시간이 지나야 tombstone compaction 대상이 된다(너무 자주 실행되지 않도록) |
| `unchecked_tombstone_compaction` | **false** | true이면 overlapping 검사를 건너뛰고 더 적극적으로 tombstone 정리를 시도한다 |

동작 방식은 다음과 같다.

- Cassandra는 compaction이 끝날 때마다 각 SSTable의 **droppable tombstone 비율**(gc_grace가 지나 회수할 수 있는 tombstone의 추정 비율)을 추적한다.
- 어떤 SSTable의 이 비율이 `tombstone_threshold`(0.2)를 넘고 생성된 지 `tombstone_compaction_interval`(1일)이 지났다면, **그 SSTable 하나만 입력으로 하는 compaction**을 실행한다. 다른 SSTable과 합치지 않고, 자신 안의 만료된 tombstone과 데이터만 정리해 새 SSTable로 다시 쓴다.

문제는 앞에서 본 overlapping 조건이다. **single-SSTable compaction은 다른 SSTable을 보지 않으므로, tombstone이 가리는 데이터가 다른 SSTable에 있으면 그 tombstone을 회수하지 못한다.** 그래서 single-SSTable tombstone compaction을 실행해도 tombstone이 줄지 않는 경우가 흔하다.

`unchecked_tombstone_compaction = true`는 이 안전 검사를 끄고 "overlapping이 있어도 일단 tombstone을 회수하라"고 지시한다. 위험을 감수하는 옵션이므로 기본값은 false다. **gc_grace가 지났고 repair가 확실히 실행되고 있다는 전제** 아래에서만 켜는 것이 안전하다. 켜면 tombstone은 더 잘 회수되지만, overlapping 데이터를 놓칠 위험(노드 안에서의 부활)은 운영자가 책임져야 한다.

```cql
-- tombstone이 잘 쌓이는 테이블을 더 공격적으로 청소
ALTER TABLE orders WITH compaction = {
    'class': 'SizeTieredCompactionStrategy',
    'tombstone_threshold': 0.1,              -- 10%만 넘어도 청소 시도
    'tombstone_compaction_interval': 3600,   -- 1시간마다 후보 검사
    'unchecked_tombstone_compaction': 'true' -- overlapping 무시 (repair 보장 시)
};
```

### tombstone이 읽기 성능을 떨어뜨리는 방식과 운영 임계값

tombstone의 진짜 위험은 디스크 낭비가 아니라 **읽기 성능 저하**다. Cassandra는 행을 읽을 때 만료된 tombstone이라도 **스캔은 해야 한다**(데이터가 삭제되었는지 확인해야 하기 때문이다). 특히 range 조회에서 tombstone이 많으면, 살아 있는 행 몇 개를 반환하려고 tombstone 수만 개를 스캔하는 일이 생긴다.

Cassandra는 이를 막기 위해 두 가지 임계값을 둔다(`cassandra.yaml`).

```text
  tombstone_warn_threshold:  1000   (기본) — 한 쿼리가 1000개 넘는 묘비 스캔 시 WARN 로그
  tombstone_failure_threshold: 100000 (기본) — 100000개 넘으면 쿼리를 강제 중단(TombstoneOverwhelmingException)
```

> [!warning] 실전 사고 시나리오
> "큐(queue)를 Cassandra로 구현했더니 어느 날 갑자기 조회가 `TombstoneOverwhelmingException`으로 실패한다." 한 파티션에 행을 INSERT하고 처리한 뒤 DELETE하는 큐 패턴은 **tombstone 안티패턴의 대표적인 예**다.
>
> 같은 파티션에 tombstone이 끝없이 쌓이고, 맨 앞의 살아 있는 행 몇 개를 읽으려면 그 앞의 tombstone을 전부 스캔해야 한다. gc_grace 10일 동안은 tombstone이 회수되지도 않으므로 임계값을 금방 넘긴다.
>
> **Cassandra를 큐나 워크리스트로 쓰지 않는다.** 꼭 써야 한다면 파티션을 시간 단위로 나누고(버킷팅) TWCS+TTL로 통째로 만료시키는 설계가 필요하다([[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]] 참고).

진단할 때는 다음 명령을 쓴다.

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

`Estimated droppable tombstones` 값이 0.2를 넘는데도 줄지 않는다면, overlapping 때문에 회수가 막혔다는 강한 신호다. 이럴 때는 LCS로 전환하거나 `nodetool garbagecollect`(단일 또는 전체 SSTable을 강제로 읽어 tombstone과 만료 데이터를 정리)를 고려하고, 마지막 수단으로 major compaction(`nodetool compact`)을 검토한다.

major compaction은 모든 SSTable을 하나로 합쳐 overlapping을 강제로 해소하지만, 거대한 단일 SSTable이 남는다는 부작용이 있으므로 STCS에서는 신중해야 한다(UCS와 LCS는 샤딩과 레벨 구조 덕분에 부작용이 작다).

---

## 전략 선택: 워크로드에 맞는 전략 고르기

지금까지 살펴본 내부 메커니즘은 결국 **이 테이블에 어떤 compaction 전략을 적용할 것인가**라는 하나의 운영 의사결정으로 모인다. 그 답은 언제나 **데이터의 접근 패턴(워크로드)** 에서 나온다. [[05 - 데이터 모델링 1 - Query First]]의 "쿼리가 모델을 결정한다"는 원칙이 여기에도 그대로 적용된다.

### 의사결정 표

| 워크로드 특성 | 권장 전략 | 이유 |
|---------------|-----------|------|
| 시계열 + TTL + append-only (수정/삭제 없음) | **TWCS** | 윈도우 통째 DROP, 만료 회수 비용 0, 시간 범위 조회가 빠름 |
| read-heavy, 업데이트 잦음, 한 키를 자주 덮어씀 | **LCS** | non-overlapping으로 읽기 SSTable 수 최소, 공간 증폭↓ |
| write-heavy, append 위주, 읽기는 가끔 | **STCS** | 쓰기 증폭 최소, compaction이 쓰기를 방해하지 않음 |
| 범용/불확실, 5.0 환경, 운영 단순화 원함 | **UCS** | scaling parameter로 읽기와 쓰기 사이 균형 조정, 샤딩 |
| 삭제(DELETE)가 매우 잦고 tombstone 누적 우려 | **LCS** 또는 UCS(leveled 쪽) | tombstone과 데이터의 overlapping 보장이 STCS보다 나음 |
| 큐/워크리스트 패턴 | **(전략 문제 아님)** | 모델 재설계 필요, Cassandra를 큐로 쓰지 말 것 |

### 전략 전환의 비용

`ALTER TABLE ... WITH compaction = {...}`로 전략을 바꾸는 것은 한 줄이면 되지만, 그 뒤에 **전면 재compaction**이 따른다. 새 전략에 맞게 모든 SSTable을 재배치하므로, 큰 테이블에서는 수 시간에서 수일 동안 추가 I/O가 발생하고 그동안 읽기 지연이 불안정해진다. 따라서 다음 사항을 지키는 것이 좋다.

- 전환은 트래픽이 적은 시간대에 한 노드씩(rolling) 진행하는 것이 안전하다.
- `nodetool compactionstats`로 진행 상황을 확인하고, `nodetool getcompactionthroughput`/`setcompactionthroughput`으로 compaction이 사용하는 I/O 대역폭을 조절한다.

```bash
# 현재 진행 중인 compaction과 대기열 확인
nodetool compactionstats -H

# compaction 처리량 제한 (MB/s). 0이면 무제한.
nodetool setcompactionthroughput 64

# 특정 테이블 강제 정리 (전략 전환 후 빨리 정돈하고 싶을 때)
nodetool garbagecollect keyspace table
```

### 결제 시스템에서의 실전 매핑

결제 도메인의 테이블을 이 표에 대입해 보면 감을 잡을 수 있다(예시).

- **트랜잭션 이벤트 로그**(이벤트 소싱, append-only, immutable): TWCS는 TTL이 있을 때만 쓴다. 결제 이벤트는 영구 보존하는 경우가 많으므로, TTL이 없다면 STCS나 UCS(tiered)를 쓴다. 영구 보존하면서 가끔 과거를 조회한다면 UCS가 무난하다.
- **결제 상태 current view**(결제 하나의 최신 상태, 업데이트와 조회가 모두 잦음): **LCS**를 쓴다. 한 키를 거듭 덮어쓰고 point 조회가 많으므로 LCS의 non-overlapping 특성이 큰 효과를 낸다.
- **정산/리포트 집계 테이블**(주기적 대량 쓰기, 읽기는 배치): STCS나 UCS(tiered)를 쓴다.
- **멱등성 키/dedup 테이블**(짧은 TTL로 만료, 쓰기와 읽기 혼합): TWCS와 짧은 TTL 조합이 깔끔하다. 단, DELETE는 쓰지 말고 TTL에만 의존한다.

> [!warning] 함정
> "결제 이벤트는 시계열이니까 무조건 TWCS"라는 판단은 위험하다. TWCS는 **TTL로 통째로 만료되는** 데이터에만 적합하다. 결제·정산 데이터는 보통 법적 보존 의무 때문에 수년 동안 남아 있어야 하고, 정정(amendment)이 생기면 같은 파티션을 다시 수정한다. 이런 경우에는 TWCS의 전제가 깨지므로 STCS나 UCS가 안전하다. **TWCS의 진짜 조건은 "시계열 모양"이 아니라 "TTL로 통째로 만료 + append-only"이다.**
