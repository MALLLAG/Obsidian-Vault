---
title: Write Path와 Read Path - Bloom filter, partition index, caching
date: 2026-06-26
tags: [cassandra, write-path, read-path, bloom-filter, 학습노트]
---

RDB에 익숙한 엔지니어라면 먼저 직관을 한 번 뒤집어야 한다. RDB에서는 보통 **쓰기가 비싸고(인덱스 갱신, 락, fsync) 읽기가 싸다(B-tree를 한 번 타면 끝난다)**. Cassandra는 정반대다. **쓰기는 거의 공짜에 가깝고, 읽기가 본래 더 많은 일을 한다.** 이 장의 내용은 모두 이 비대칭을 이해하는 데서 출발한다.

---

## 두 path를 한눈에: 비대칭의 기원

먼저 표로 전체 구조를 잡아 보자. 같은 데이터를 두고 쓰기와 읽기가 내부적으로 얼마나 다른 양의 일을 하는지 비교해 보면, 나머지 내용은 자연스럽게 이해된다.

| 측면 | Write Path | Read Path |
|------|-----------|-----------|
| 디스크 접근 패턴 | **순차 append** (commit log에 덧붙이기) | **랜덤 read** 가능성 (여러 SSTable 조회) |
| 손대는 자료구조 | commit log + memtable(메모리) | memtable + N개 SSTable + 여러 cache |
| 정렬·병합 | 없음 (그냥 기록) | **있음** (timestamp 기준 cell 단위 병합) |
| 기존 데이터를 읽는가? | **읽지 않는다** (read-before-write 없음) | 당연히 읽는다 |
| 비용을 지불하는 시점 | 거의 없음, compaction으로 **나중에 미룸** | **지금** 전부 지불 |
| 실패 시 보완 | hinted handoff | read repair |

이 표에서 가장 중요한 것은 끝에서 두 번째 줄이다. Cassandra는 쓰기 비용을 "지금" 치르지 않고 **compaction이라는 백그라운드 작업으로 미룬다**([[09 - Compaction 전략]]). 그 대신 읽을 때 "여기저기 흩어진 조각을 모으는" 비용을 치른다. 이것은 LSM-tree 계열 스토리지의 본질적인 트레이드오프이며, write-heavy 워크로드(로그, 이벤트, 시계열, 결제 이벤트 소싱)에서 Cassandra가 강력한 이유이기도 하다.

```text
        쓰기: 비용을 미래로 미룬다              읽기: 미뤄둔 비용을 지금 갚는다
   ┌───────────────────────────┐        ┌───────────────────────────────┐
   │ commit log append (순차)  │        │ memtable + SSTable_1 ... _N    │
   │ memtable put (메모리)     │        │ 각각에서 같은 키 조각 수집     │
   │ → 끝. ack.                │        │ → timestamp로 병합(LWW)        │
   └───────────────────────────┘        └───────────────────────────────┘
            싸다 / 빠르다                        비싸다 / 최적화 계층 필요
```

이제 각 path를 분산 레벨(노드 간)과 로컬 레벨(노드 안)로 나누어 자세히 살펴본다.

---

## Write Path (1): coordinator가 replica를 찾고 정렬한다

클라이언트가 보낸 쓰기 요청 하나는 두 단계의 라우팅을 거친다. 첫째는 어느 노드들이 이 데이터를 책임지는지 정하는 **노드 간(inter-node)** 단계이다. 둘째는 각 노드가 받은 데이터를 디스크에 기록하는 **노드 안(intra-node)** 단계이다. 이 절에서는 첫 번째 단계를 다룬다.

### coordinator: 요청마다 정해지는 임시 역할

클라이언트(정확히는 드라이버)는 클러스터의 어느 노드에든 요청을 보낼 수 있다. 요청을 처음 받은 노드가 그 요청 한 건에 한해 **coordinator(코디네이터)** 역할을 맡는다. coordinator는 고정된 역할이 아니라 요청마다 달라지는 **임시 역할**이다. 같은 클라이언트가 보낸 다음 요청에서는 다른 노드가 coordinator가 될 수 있다.

> [!warning] 흔한 오해
> "coordinator 노드가 따로 있다"고 생각하기 쉽지만 그렇지 않다. 모든 노드가 똑같이 coordinator가 될 수 있다. Cassandra에 마스터가 없다(masterless)는 점과 같은 맥락이다. 드라이버의 `TokenAwarePolicy`는 보통 **데이터를 실제로 가진 replica 중 하나가 coordinator가 되도록** 라우팅해서 불필요한 홉(hop)을 줄인다.

### partitioner로 replica를 계산한다

coordinator는 가장 먼저 이 쓰기가 어느 노드들로 가야 하는지 계산한다.

1. **partition key → token**: row의 partition key를 partitioner(기본값 `Murmur3Partitioner`)에 넣어 64비트 토큰을 얻는다. 이것은 순수한 해시 계산이므로 어느 노드에서 계산해도 같은 답이 나오고, 네트워크 통신도 필요 없다.
2. **token → primary replica**: 토큰 링에서 그 토큰을 담당하는 노드(시계 방향으로 첫 번째 노드)가 primary replica이다.
3. **token → 나머지 replica**: 복제 계수(replication factor, RF)가 3이면 링을 따라 다음 노드 2개를 더 고른다. 이때 단순히 "다음 노드"를 고르는 것이 아니다. [[03 - 복제 전략과 데이터센터]]에서 본 replication strategy(`NetworkTopologyStrategy`)와 snitch가 개입해서, **같은 rack에 몰리지 않고 각 데이터센터에 정해진 수만큼** 배치된 replica 집합을 돌려준다.

여기서 알아 둘 점이 하나 있다. **replica가 어디에 있는지 알아내려고 클러스터에 물어볼 필요가 없다.** 토큰 링의 소유권 정보는 gossip을 통해 모든 노드가 이미 알고 있다. 그래서 coordinator는 **계산만으로** 목적지 N개를 바로 알아낸다. RDB의 샤딩 라우터가 메타데이터 서버에 "이 키는 어디에 있는가?"를 물어보는 방식과 대조적이다.

### snitch로 replica를 "정렬"한다

계산된 replica 집합은 순서가 없는 후보 목록일 뿐이다. coordinator는 여기에 **snitch**를 적용해 어느 replica가 더 가까운지(network distance)를 기준으로 정렬한다. snitch는 각 노드의 데이터센터·rack 토폴로지를 알고 있어서, coordinator 자신과 같은 DC·같은 rack에 있는 replica를 가깝다고 판단한다.

그 위에 **dynamic snitch**가 더해진다. dynamic snitch는 정적 토폴로지뿐 아니라 **최근 관측된 응답 지연과 부하**까지 반영해 순위를 실시간으로 다시 매긴다. 따라서 뒤의 읽기 절에서 말하는 "가장 가까운 replica"는 물리적으로 가까운 replica라기보다 **지금 가장 빠른 replica**에 가깝다. GC 때문에 잠깐 느려진 노드는 자동으로 순위가 뒤로 밀린다.

쓰기에서는 이 정렬이 결정적으로 중요하지 않다(쓰기는 어차피 일관성 레벨(consistency level, CL)만큼의 replica에 동시에 보낸다). 하지만 읽기에서는 같은 메커니즘이 **"가장 가까운 replica에게만 full read를 시키는"** 핵심 최적화에 쓰인다. 그래서 여기서 정렬 개념을 이해해 두면 읽기 절을 따라가기 쉽다.

```text
  Client
    │ INSERT INTO payments ... (partition key = "user_42:txn_998")
    ▼
 ┌──────────────────────────────────────────────────────────┐
 │ Coordinator (예: node A)                                  │
 │  1) Murmur3(partition key) = token  ─ 순수 계산           │
 │  2) NetworkTopologyStrategy + snitch → replica 집합       │
 │       RF=3 → {node C, node F, node J}                     │
 │  3) snitch로 거리 정렬 (쓰기엔 동시 전송)                 │
 └──────────────────────────────────────────────────────────┘
    │            │            │
    ▼            ▼            ▼
 node C       node F       node J         ← 각자 로컬 write path 수행
```

---

## Write Path (2): 각 replica 안에서 commit log와 memtable에 쓴다

이제 두 번째 단계, 즉 노드 안에서 일어나는 일을 살펴본다. coordinator가 보낸 mutation을 받은 각 replica는 [[08 - Storage Engine 내부]]에서 배운 절차를 그대로 실행한다. 요점은 **딱 두 곳에만 쓴다**는 것이다.

### 1단계: commit log에 append (내구성)

mutation은 먼저 **commit log**에 순차적으로 덧붙여진다. 디스크에 쓰는 동작이지만 **순차 append**이므로 빠르다. 디스크 헤드를 여기저기 움직이는 랜덤 쓰기가 아니라 파일 끝에 계속 이어 붙이기 때문이다. SSD에서도 순차 쓰기가 랜덤 쓰기보다 유리하며, HDD에서는 그 차이가 훨씬 크다.

commit log의 목적은 **내구성(durability)** 하나뿐이다. memtable은 메모리에 있으므로 노드가 죽으면 사라진다. 하지만 commit log가 디스크에 먼저 남아 있으므로, 노드가 재시작하면 commit log를 replay해서 memtable을 복원할 수 있다.

> Cassandra 5.0 기본값으로 commit log는 **`periodic` 모드**이고, `commitlog_sync_period`는 보통 10초(`10000ms`)다. 쓰기마다 fsync로 디스크에 강제 동기화하지 않고 **주기적으로 묶어서 flush**한다. 이것이 디스크 기반 DB인데도 빠른 또 하나의 이유다.
>
> 대신 그 10초 창(window) 안에 OS까지 함께 죽으면 그 구간의 데이터를 잃을 수 있다. 절대적인 내구성이 필요하면 `batch` 모드로 바꿀 수 있지만, 처리량이 크게 떨어진다. 결제처럼 손실되면 곤란한 데이터라면 이 트레이드오프를 의식적으로 선택해야 한다.

### 2단계: memtable에 put (조회 가능 상태)

같은 mutation이 **memtable**(메모리 안의 정렬된 자료구조로, 테이블마다 하나씩 있다)에도 기록된다. memtable에 들어가는 순간 그 데이터는 바로 읽기 대상이 된다. SSTable로 디스크에 내려가기를 기다릴 필요가 없다.

여기서 RDB와 결정적으로 다른 점이 나온다.

> **Cassandra의 쓰기에는 read-before-write가 없다.** `UPDATE`조차 기존 값을 읽어 오지 않는다. "user_42:txn_998 의 status 컬럼에 timestamp T로 PAID를 기록"이라는 셀(cell)을 그냥 append할 뿐이며, 같은 컬럼에 기존 값이 있는지는 신경 쓰지 않는다. 충돌은 **읽을 때** timestamp로 해소한다(last-write-wins).
>
> RDB에서는 `UPDATE`가 해당 row를 찾아 락을 걸고 제자리(in-place)에서 고친다. 반면 Cassandra에는 "예전 값 위에 덮어쓰기"라는 개념 자체가 없다. 모든 쓰기는 새 버전을 추가하는 것이다.

read-before-write가 없다는 점이 쓰기가 빠른 가장 근본적인 이유다. 읽기, 락, 인덱스 트리 재균형 같은 비싼 작업이 쓰기 경로에서 모두 빠진다.

> [!note] 예외: read-before-write가 필요한 쓰기
> 일반 `INSERT`/`UPDATE`는 기존 값을 읽지 않지만, *모든* 쓰기가 그런 것은 아니다.
>
> 1. **counter** 컬럼은 현재 값을 알아야 증감할 수 있으므로 내부적으로 read를 함께 수행한다.
> 2. **list** 컬렉션의 일부 연산(인덱스 기반 set/삭제, prepend 등)도 기존 원소를 읽어야 한다. list보다 set/map을 권장하는 이유 중 하나다.
> 3. **LWT(경량 트랜잭션)** 는 `IF` 조건을 검사하려고 Paxos read를 수행한다([[11 - LWT Batch Counter 내부]]).
>
> 이들이 모두 "비싼 쓰기"로 분류되는 근본적인 이유는 read-before-write가 다시 등장하기 때문이다.

memtable은 메모리에 영원히 남아 있지 않다. 다음 경우에 가장 큰 memtable이 **flush**되어 불변(immutable) SSTable로 디스크에 내려간다. 전체 memtable이 쓰는 메모리(`memtable_heap_space`/`memtable_offheap_space`)가 한도를 넘을 때, commit log 총량(`commitlog_total_space`)이 한계에 닿을 때, `nodetool flush`가 호출될 때이다.

flush가 끝나면 그에 대응하는 commit log 구간은 더 이상 필요 없으므로 회수된다. (예전에는 `memtable_cleanup_threshold`도 이 판단에 관여했지만, 최신 버전에서는 deprecated이다.) 이 flush와 이후의 compaction이 바로 "미뤄 둔 비용"이다.

### 3단계: CL만큼 ack를 모아 클라이언트에 응답

각 replica는 commit log append와 memtable put을 마치면 coordinator에게 ack를 보낸다. coordinator는 요청에 지정된 **일관성 레벨(consistency level)** 만큼 ack가 모이면 클라이언트에 성공을 반환한다([[04 - Tunable Consistency]]).

- `CL=ONE`: replica 1개의 ack만 오면 바로 성공.
- `CL=QUORUM`(RF=3이면 2개): ack 2개를 기다린다.
- `CL=ALL`: 3개 전부.

여기에는 미묘한 점이 있다. coordinator는 **모든 replica에 동시에** mutation을 보낸다. CL은 "몇 개에 보낼까"가 아니라 "**몇 개의 성공을 기다렸다가 클라이언트에 OK를 돌려줄까**"를 뜻한다. `CL=ONE`이어도 나머지 replica 2개에도 쓰기가 전달된다. 단지 그 ack를 기다리지 않을 뿐이다.

```text
 coordinator
   ├──mutation──▶ node C  ─ commitlog append + memtable put ─ ack ─┐
   ├──mutation──▶ node F  ─ commitlog append + memtable put ─ ack ─┤
   └──mutation──▶ node J  ─ (느림/다운)                            │
                                                                    ▼
   CL=QUORUM → ack 2개 모임 → 클라이언트에 "성공" 반환 (node J 기다리지 않음)
```

---

## Write Path (3): down replica와 hinted handoff

위 그림에서 node J가 다운되었거나 느리다고 하자. `CL=QUORUM`이면 C와 F의 ack 2개로 이미 성공을 반환했다. 하지만 node J는 이 쓰기를 받지 못했다. 나중에 살아나면 데이터가 빠진 채로 클러스터에 복귀하게 된다. 이 공백을 메우는 메커니즘이 **hinted handoff**다.

coordinator는 쓰기 시점에 어떤 replica가 다운되었는지 gossip으로 알고 있다. 그래서 다운된 replica에 보낼 mutation을 **hint(힌트)** 로 자기 로컬에 저장해 둔다. hint는 "node J에게 전달해야 하지만 아직 전달하지 못한 쓰기 X"를 적어 둔 기록이다. node J가 다시 살아나면 coordinator가 보관하던 hint를 다시 보내서 누락된 쓰기를 채운다.

```text
 coordinator(node A)
   ├──▶ node C  ✔ (ack)
   ├──▶ node F  ✔ (ack)
   └──▶ node J  ✘ DOWN
            │
            ▼
   node A가 "node J에게 줄 mutation"을 hint로 로컬 저장
            │
            ⌛ ... node J 복귀 ...
            ▼
   node A가 hint를 node J에게 재전송 → 누락 보충
```

몇 가지 중요한 세부 사항이 있다.

- **hint는 무한히 보관하지 않는다.** `max_hint_window`(Cassandra 5.0 기본값 **3시간**, `max_hint_window_in_ms` 또는 `max_hint_window=3h`) 동안만 보관한다. 이 시간을 넘겨 다운되어 있던 노드는 hint로 복구되지 않으므로, 복귀한 뒤 반드시 `nodetool repair`로 데이터를 맞춰야 한다.
- **hint는 일관성을 보장하는 장치가 아니다.** hint가 coordinator 디스크에 있는 동안 그 coordinator까지 죽으면 hint도 사라진다. hinted handoff는 "잠깐 끊겼던 노드가 빨리 따라잡게 해 주는 편의 장치"일 뿐, 정합성을 지키는 최종 안전장치가 아니다. **최종 안전장치는 read repair와 anti-entropy repair**다.
- **기본적으로 hint는 CL 충족에 포함되지 않는다.** `CL=QUORUM`을 달성하려면 hint 저장이 아니라 **실제 replica의 ack**가 필요하다. 살아 있는 replica 수가 CL보다 적으면 쓰기는 `UnavailableException`으로 실패한다. (예외적으로 `CL=ANY`는 hint 저장만으로도 성공으로 간주하는데, 데이터가 어떤 정식 replica에도 들어가지 않을 수 있어 위험하다. 결제 시스템에서는 거의 쓰지 않는다.)

> [!note] 결제 엔지니어를 위한 함의
> hinted handoff 덕분에 노드 하나가 잠깐 재시작해도 쓰기가 실패하지 않고 처리된다. 하지만 노드가 3시간 넘게 죽어 있었다면 hint가 만료되었을 수 있다. 운영 규칙은 **오래 다운되었던 노드가 복귀하면 무조건 repair한다**는 것이다. 이를 빠뜨리면 그 노드에서 읽을 때 stale data가 나올 수 있다. ([[12 - 운영과 트러블슈팅]]에서 더 다룬다.)

### 쓰기가 빠른 이유: 한 문단 정리

쓰기가 빠른 이유를 한데 모으면 다음과 같다.

1. replica 위치를 **계산으로** 알아내므로 라우팅 비용이 거의 없다.
2. 각 노드에서 디스크 접근은 commit log **순차 append** 하나뿐이고, 나머지는 메모리(memtable)에서 처리한다.
3. **read-before-write가 없으므로** 락, 인덱스 재균형, 기존 값 조회가 모두 빠진다.
4. commit log fsync를 **묶어서(periodic)** 처리해 fsync 빈도를 낮춘다.
5. SSTable 정리와 병합 같은 무거운 작업은 **compaction으로 미룬다**.

즉 Cassandra의 쓰기는 "지금은 최소한만 하고 나중에 정리한다"는 전략을 그대로 구현한 것이다.

---

## Read Path (1): coordinator의 full read + digest 전략

이제 미뤄 둔 비용을 치를 차례다. 읽기도 쓰기처럼 노드 간 단계와 노드 안 단계로 나뉜다. 먼저 노드 간 단계부터 살펴본다.

coordinator는 쓰기와 똑같이 partitioner로 replica 집합을 계산하고 snitch로 거리순으로 정렬한다. 여기까지는 쓰기와 같고, 그다음부터가 다르다.

읽기는 **모든 replica에 전체 데이터를 요청하지 않는다.** 네트워크와 디스크를 아끼기 위해 replica마다 역할을 나눈다.

- snitch 기준으로 가장 가까운 replica 1개에만 **full read(데이터 본문 전체)** 를 요청한다.
- 일관성 레벨(consistency level)을 채우는 데 필요한 나머지 replica에는 **digest read**를 요청한다. digest는 데이터 본문이 아니라 그 데이터의 **해시(체크섬)** 다.

이렇게 하는 데는 이유가 있다. CL을 만족하려면 여러 replica의 응답을 비교해 이 값이 정말 최신이고 서로 일치하는지 확인해야 한다. 그런데 일치 여부만 확인하려면 **전체 데이터를 다 받을 필요 없이 해시만 비교해도 충분하다.** full read는 무겁고(수 KB~수 MB일 수 있다) digest는 가볍다(고정 크기 해시). coordinator는 full read 1개와 digest N개를 받아, **digest가 모두 full read의 digest와 일치하는지** 확인한다.

```text
 Client: SELECT ... WHERE partition_key = "user_42:txn_998"  (CL=QUORUM, RF=3)
    │
    ▼
 ┌───────────────────────────────────────────────────────────┐
 │ Coordinator                                               │
 │   replica 집합 {C, F, J}, snitch로 거리 정렬              │
 │   → C(가장 가까움): FULL read 요청                        │
 │   → F: DIGEST read 요청                                   │
 │   (QUORUM=2 충족: C와 F. J는 이번엔 안 물어봄)            │
 └───────────────────────────────────────────────────────────┘
        │ full                  │ digest
        ▼                       ▼
     node C                  node F
   (데이터 본문)            (해시만)
        │                       │
        └──────────┬────────────┘
                   ▼
        digest 일치? ── 예 → 클라이언트에 즉시 반환
                    └─ 아니오 → read repair 발동 (뒤에서)
```

### consistency level이 읽기에서 의미하는 것

`CL=QUORUM`, RF=3이면 coordinator는 replica 2개의 응답(full 1개 + digest 1개)을 모은다. 두 digest가 일치하면 그 데이터를 클라이언트에 돌려준다. 일치하지 않으면, 즉 replica 사이에 데이터가 다르면 **read repair**가 동작한다. `CL=ONE`이면 가장 가까운 replica의 full read 하나만 받고 끝난다. digest 비교도 일관성 검증도 없으므로, 빠르지만 stale data를 읽을 수 있다.

이 역할 분담은 4장(Tunable Consistency)의 `R + W > RF` 공식과 직접 연결된다. 읽기 CL과 쓰기 CL의 합이 RF를 넘으면 "읽은 replica 집합과 쓴 replica 집합이 반드시 겹치므로" 최신 값을 본다는 보장이 생긴다. 그 비교를 실제로 수행하는 메커니즘이 바로 이 full read와 digest의 대조다.

---

## Read Path (2): 단일 노드 안에서 조각을 모아 병합한다

coordinator가 node C에 full read를 요청했다. 이제 node C **안에서** 무슨 일이 일어나는지 살펴보자. 읽기 path에서 가장 핵심적인 부분이다.

문제는 같은 partition key "user_42:txn_998"의 데이터가 **한곳에 모여 있지 않다**는 것이다. 8장(Storage Engine 내부)에서 보았듯이 그 키에는 시간에 걸쳐 여러 번 쓰기가 일어났고, 그 결과는 다음과 같이 흩어져 있다.

- 아직 flush되지 않은 최신 쓰기는 **memtable**에 있다.
- 예전에 flush된 쓰기들은 **SSTable_1, SSTable_2, ... SSTable_N**에 흩어져 있다. compaction이 모두 합치기 전까지는 같은 키의 조각이 여러 SSTable에 동시에 존재할 수 있다.

그래서 node C는 이 모든 출처에서 **같은 키의 조각을 모두 모은 다음**, **cell(셀, 컬럼 값) 단위로 timestamp를 비교해 가장 최신 것만 골라** 하나의 논리적인 행으로 다시 조립한다. 이것이 **last-write-wins(LWW)** 병합이다.

```text
   같은 partition key "user_42:txn_998" 의 조각들

   memtable     : { status: PAID  @ t=105 }
   SSTable_3    : { status: READY @ t=103, amount: 5000 @ t=101 }
   SSTable_1    : { status: INIT  @ t=100, amount: 5000 @ t=101, buyer: kim @ t=100 }
                          │
                          ▼  cell 단위 timestamp 비교 (last-write-wins)
   ┌──────────────────────────────────────────────────────────┐
   │ 병합 결과 (클라이언트에게 보일 한 행)                    │
   │   status : PAID   (t=105 가 최신)                         │
   │   amount : 5000   (t=101)                                 │
   │   buyer  : kim    (t=100)                                 │
   └──────────────────────────────────────────────────────────┘
```

여기서 강조할 점이 몇 가지 있다.

- **병합은 행 단위가 아니라 cell 단위다.** `status`는 memtable의 t=105가 선택되고, `amount`는 SSTable의 t=101이 선택된다. 한 행을 통째로 "최신 SSTable의 것"으로 가져오는 것이 아니라, 각 컬럼이 독립적으로 자신의 최신 timestamp를 가진다. 그래서 부분 업데이트(`UPDATE`로 컬럼 하나만 고치는 경우)가 자연스럽게 동작한다.
- **여러 SSTable을 봐야 한다는 점이 읽기의 비용이다.** 같은 키가 몇 개의 SSTable에 흩어져 있는지가 읽기 지연(latency)을 좌우한다. compaction은 SSTable을 합쳐 이 숫자를 줄여 준다. 9장(Compaction 전략)에서 본 SizeTiered와 Leveled의 트레이드오프가 바로 여기서 드러난다. Leveled는 한 키가 레벨마다 SSTable 하나에만 있도록 정리해서, 읽을 때 확인해야 할 SSTable 수를 줄인다(읽기에 유리한 대신 쓰기 증폭이 생긴다).
- **삭제도 하나의 "값"으로 병합에 참여한다.** `DELETE`는 데이터를 지우는 것이 아니라 **tombstone(묘비)** 이라는 특수 마커를 timestamp와 함께 append한다. 병합할 때 tombstone의 timestamp가 가장 최신이면 그 cell은 삭제된 것으로 판정된다. tombstone이 읽기 성능을 어떻게 떨어뜨리는지는 뒤에서 따로 자세히 다룬다.

> [!note] RDB였다면
> `SELECT`는 B-tree 인덱스를 타고 page 하나를 읽으면 그 안에 완성된 row가 있다. in-place 갱신이므로 "흩어진 조각"이라는 개념이 없다. Cassandra는 immutable append 구조를 택한 대가로, 읽을 때마다 조각을 모으고 timestamp로 조정하는 일을 한다. 이 차이가 "쓰기는 싸고 읽기는 비싸다"는 말의 물리적인 실체다.

그런데 node C는 이 partition key가 어느 SSTable에 들어 있는지를 어떻게 알까? SSTable이 수십 개라면 전부 열어서 뒤질 수는 없다. 여기서 이 장의 핵심 주제인 **읽기 최적화 계층**이 등장한다.

---

## Read Path (3): 단일 노드 안의 읽기 최적화 계층(순서대로)

node C는 partition key 하나를 읽을 때 디스크의 SSTable부터 무작정 읽지 않는다. **여러 겹의 필터와 캐시를 순서대로 거치면서** 가능한 한 디스크 접근을 피하고, 피할 수 없다면 꼭 필요한 바이트만 읽는다. 이 계층을 순서대로 따라가 보자.

### 0단계 row cache: 운이 좋으면 여기서 끝난다

가장 먼저 그 partition(또는 partition의 앞부분)이 **row cache**에 통째로 있는지 확인한다. 있으면 SSTable을 볼 필요 없이 메모리에서 바로 반환한다. row cache는 강력하지만 위험한 옵션이어서 뒤의 캐시 절에서 따로 자세히 다룬다. 기본값은 **꺼져 있다**(0MB). 여기서는 "있으면 가장 짧은 경로"라는 정도만 알아 두자. 대부분의 클러스터는 row cache가 꺼져 있으므로 바로 다음 단계로 넘어간다.

### 1단계 Bloom filter: "이 SSTable에는 그 키가 없다"를 즉시 판정한다

SSTable마다 **Bloom filter**가 하나씩 딸려 있고, 이것은 항상 메모리에 올라가 있다. Bloom filter는 확률적 자료구조로, 다음 질문 하나에 매우 빠르게 답한다.

> "이 SSTable에 partition key K가 **있을 수도 있는가, 아니면 확실히 없는가**?"

Bloom filter가 내놓는 두 가지 답과 각각의 신뢰도를 정확히 이해해야 한다.

| Bloom filter의 답 | 의미 | 신뢰할 수 있는가 |
|---|---|---|
| "없다 (negative)" | 이 SSTable엔 K가 **확실히 없다** | **100% 정확.** false negative 불가능 |
| "있을 수도 있다 (positive)" | 아마 있지만 아닐 수도 있다 | **확률적.** false positive 가능 |

Bloom filter의 핵심은 이 비대칭이다. **"없다"는 답은 절대적으로 믿을 수 있다.** Bloom filter가 "없다"고 답하면 그 SSTable은 디스크에 접근하지 않고 **건너뛴다**. SSTable이 50개여도 Bloom filter가 그중 48개에 "없다"고 답하면 디스크에서는 나머지 2개만 확인한다. Bloom filter는 읽기 성능을 지키는 첫 번째 장치다.

반대로 "있을 수도 있다"는 답은 가끔 틀린다(**false positive**). 실제로는 없는데 있다고 답해서 불필요한 디스크 읽기를 일으킨다. 하지만 **false negative는 원리적으로 불가능**하다. 실제로 있는데 "없다"고 답하는 일은 없다. 따라서 데이터를 놓치는 일은 절대 없고, 가끔 불필요한 읽기를 할 뿐이다. 이 보장 덕분에 정확성(correctness)이 유지된다.

동작 원리는 간단하다. Bloom filter는 비트 배열과 여러 해시 함수로 이루어진다. 키를 넣을 때 여러 해시 위치의 비트를 1로 켠다. 조회할 때 그 위치가 **모두 1이면 "있을 수도"**, **하나라도 0이면 "확실히 없음"** 이다. 0이 하나라도 있다면 그 키를 넣은 적이 절대 없다는 뜻이므로 false negative가 생길 수 없다.

```text
   "user_42:txn_998" → h1,h2,h3 → 비트 위치 [5, 19, 88]

   SSTable_A bloom: ...1...1...1...  (5,19,88 모두 1) → "있을 수도" → 디스크 확인 필요
   SSTable_B bloom: ...1...0...1...  (19가 0)        → "확실히 없음" → SKIP! (디스크 안 봄)
```

#### bloom_filter_fp_chance: 정확도와 메모리의 트레이드오프

false positive가 얼마나 자주 나는지는 테이블 속성 **`bloom_filter_fp_chance`** 로 조절한다. 값이 작을수록(예: 0.001 = 0.1%) false positive가 줄어 불필요한 디스크 읽기도 줄어든다.

대신 Bloom filter 비트 배열이 커져 **off-heap 메모리를 더 많이 쓴다**(Cassandra의 Bloom filter는 off-heap에 상주하므로 JVM heap과 GC에 직접 부담을 주지는 않지만, 노드 전체의 메모리 예산을 차지한다). 값이 클수록 메모리는 아끼지만 불필요한 읽기가 늘어난다.

Cassandra 5.0의 기본값은 compaction 전략에 따라 다르다(정확한 수치이므로 외워 둘 만하다).

| compaction 전략 | `bloom_filter_fp_chance` 기본값 |
|---|---|
| SizeTieredCompactionStrategy / UnifiedCompactionStrategy | **0.01** (1%) |
| LeveledCompactionStrategy | **0.1** (10%) |

LCS의 기본 false positive 비율이 더 느슨한(10%) 이유는 무엇일까? LCS에서는 구조상 한 키가 레벨마다 SSTable 하나에만 있으므로, 어차피 확인해야 할 SSTable 수가 적다. Bloom filter에 메모리를 덜 써도 읽기가 충분히 빠르기 때문에 메모리를 아끼는 쪽으로 기본값을 정했다. 반대로 STCS/UCS에서는 같은 키가 여러 SSTable에 퍼질 수 있어 Bloom filter의 정확도가 더 중요하므로, 1%로 엄격하게 잡는다.

```cql
-- 읽기가 많고 메모리 여유가 있으면 false positive를 더 낮춰 헛읽기를 줄인다
ALTER TABLE payments.transactions
  WITH bloom_filter_fp_chance = 0.001;
```

> [!warning] 함정
> `bloom_filter_fp_chance`를 0으로 만들 수는 없다. 0에 가까워질수록 메모리 사용량이 가파르게 늘어난다. 또한 partition이 매우 많은(키 카디널리티가 높은) 테이블일수록 Bloom filter 메모리 총량이 커진다. 무작정 낮추면 off-heap 메모리가 부족해져 노드 전체의 메모리 압박으로 이어질 수 있다. 12장(운영과 트러블슈팅)의 메모리 진단과 함께 살펴봐야 한다.

### 2단계 key cache: partition key의 디스크 위치를 기억한다

Bloom filter가 "있을 수도 있다"며 통과시켰다. 이제 그 키가 이 SSTable의 Data.db 파일에서 **몇 번째 바이트 오프셋**에 있는지 알아야 한다. 매번 디스크에서 인덱스를 뒤지면 느리기 때문에 **key cache**를 둔다.

key cache는 `(SSTable, partition key) → Data.db 내 정확한 오프셋`을 메모리에 캐시한다. **cache hit이면 partition index 조회 단계(아래 3·4단계)를 통째로 건너뛰고** Data.db의 해당 위치로 바로 이동한다. 반복해서 조회되는 hot key의 읽기를 크게 빠르게 해 준다.

Cassandra 5.0에서 key cache는 기본적으로 **켜져 있다**. 크기는 `key_cache_size`로 정하며, 기본값은 "heap의 5%와 100MB 중 작은 값"(`min(5% of heap, 100MB)`)이다. 대체로 켜 두는 편이 이득이고 무효화(invalidation) 부담도 작아서 row cache처럼 위험하지 않다.

### 3단계 partition summary: 인덱스의 인덱스(샘플)

key cache에서 miss가 나면 partition index를 봐야 한다. 그런데 partition index 자체가 디스크에 있고 크기도 클 수 있다. 그래서 그 인덱스의 **샘플(sample)** 을 메모리에 둔 것이 **partition summary**다.

partition summary는 partition index를 일정 간격(예: 128번째 키마다)으로 샘플링해서, 키 K가 partition index 파일의 대략 어느 부근에서 시작하는지 알려 준다. 덕분에 인덱스 파일 전체를 스캔하지 않고 **좁은 구간으로 바로 이동**할 수 있다. 이런 역할 때문에 "인덱스의 인덱스"라고 부른다.

샘플링 간격은 `min_index_interval`/`max_index_interval`(기본값 128/2048)로 조절한다. 간격이 촘촘할수록 위치를 정확히 찾아 디스크 탐색이 줄지만, summary가 커져 메모리를 더 쓴다. summary는 메모리에 상주한다.

> [!note] 참고
> Cassandra의 SSTable 포맷이 발전하면서 인덱스 구조의 세부 사항(예: 5.0의 BTI/`trie` 인덱스 포맷 옵션)은 버전에 따라 달라질 수 있다. 개념적으로는 "메모리의 성긴 색인(summary) → 디스크의 촘촘한 인덱스(index) → 데이터(data)"로 이어지는 3단계 구조로 이해하면 어떤 포맷에도 적용할 수 있다. (세부 포맷 이름은 버전에 따라 다르므로 단정하지 않는다.)

### 4단계: partition index → Data.db

summary가 가리킨 partition index 구간을 디스크에서 읽어, 찾는 partition key의 **정확한 Data.db 오프셋**을 얻는다. 그리고 마지막으로 **Data.db**의 그 위치에서 실제 데이터(셀들)를 읽는다. (key cache hit였다면 2·3·4단계를 모두 건너뛰고 바로 이 단계로 왔을 것이다.)

이 단계들을 그림 하나로 요약하면 다음과 같다.

```text
 Bloom filter (있을 수도?)
      │ 예
      ▼
 key cache ──hit──▶ Data.db 오프셋 즉시 획득 ─┐
      │ miss                                  │
      ▼                                       │
 partition summary (메모리, 성긴 이정표)      │
      │ 좁은 구간 지목                        │
      ▼                                       │
 partition index (디스크, 촘촘한 인덱스)      │
      │ 정확한 오프셋                         │
      ▼                                       ▼
 Data.db (디스크) ──────────────────▶ 셀들을 읽어 병합 대상에 추가
```

이 과정을 **읽어야 할 SSTable마다**(Bloom filter를 통과한 것만) 반복한 뒤, 앞 절에서 설명한 cell 단위 timestamp 병합으로 최종 행을 만든다.

### 그 아래의 캐시: chunk cache와 OS page cache

Data.db를 실제로 디스크에서 읽을 때도 캐시가 한 겹 더 있다.

- **chunk cache**(과거 이름을 그대로 쓰면 file/row cache와 헷갈리므로 주의한다): SSTable의 데이터는 보통 LZ4 등으로 압축되어 **압축 청크(chunk)** 단위로 저장된다. 디스크에서 읽어 압축을 푼 청크를 메모리에 캐시해 두면, 같은 청크를 다시 읽을 때 디스크 I/O와 압축 해제 비용을 아낄 수 있다. chunk cache는 off-heap에 둔다.
- **OS page cache**: 그 아래에서는 운영체제가 파일 페이지를 캐시한다. Cassandra는 디스크에서 읽는다고 생각하지만, 실제로는 OS page cache에서 메모리 속도로 반환되는 경우가 많다. 그래서 메모리가 넉넉한 노드는 "디스크 DB"라기보다 사실상 메모리에서 읽는 것처럼 동작한다. Cassandra 노드에 RAM을 넉넉히 주라는 운영 권고는 여기에 근거한다.

정리하면, 읽기 한 번이 거치는 계층은 위에서 아래로 **row cache → (Bloom filter) → key cache → partition summary → partition index → chunk cache → OS page cache → 디스크** 순서다. 위 단계에서 답이 나오면 아래 단계는 건드리지 않는다.

---

## 읽기 한 번의 전체 시퀀스: 끝까지 추적

지금까지 따로 살펴본 단계를 하나의 타임라인으로 묶어 보자. 아래는 RF=3, CL=QUORUM에서 `SELECT ... WHERE partition_key = 'user_42:txn_998'` 한 번이 처음부터 끝까지 거치는 모든 단계다.

```text
[노드 간]
 1. Client → 아무 노드(coordinator) 로 read 요청
 2. coordinator: Murmur3(key)=token → replica {C,F,J} 산출, snitch로 거리 정렬
 3. coordinator: C(최근접)에 FULL read, F에 DIGEST read 요청 (QUORUM=2)

[노드 안 — node C 에서, F도 병렬로 유사 수행]
 4. row cache 확인 ── hit이면 즉시 반환하고 종료(보통 꺼져 있음)
 5. 읽을 후보 SSTable 각각에 대해:
      a. Bloom filter 질의
           - "없음" → 그 SSTable SKIP (디스크 접근 0)
           - "있을 수도" → 계속
      b. key cache 확인
           - hit → Data.db 오프셋 즉시 획득 (c,d 건너뜀)
           - miss → c로
      c. partition summary(메모리) 로 index 구간 좁히기
      d. partition index(디스크) 에서 정확한 오프셋 획득
      e. Data.db(chunk cache→OS page cache→디스크) 에서 셀 읽기
 6. memtable + 통과한 SSTable들의 셀을 cell 단위 timestamp 병합 (last-write-wins)
      - tombstone이 최신이면 해당 cell은 삭제로 판정
 7. node C는 병합 결과(full), node F는 digest를 coordinator에 반환

[노드 간 — 검증과 보정]
 8. coordinator: F의 digest == C 데이터의 digest ?
      - 일치 → 9로
      - 불일치 → read repair: 최신 데이터를 stale replica에 비동기/동기로 써서 수렴
 9. coordinator → Client 에 최종 행 반환
```

이 시퀀스에서 latency를 결정하는 변수는 분명하다.

1. 확인해야 할 SSTable 수(= compaction 상태)
2. Bloom filter false positive 때문에 생기는 불필요한 읽기
3. key cache hit율
4. 병합 중에 만난 tombstone 수
5. digest 불일치로 read repair가 끼어드는 빈도

운영 중에 읽기가 느리다면 이 다섯 가지를 순서대로 의심해 보면 된다.

---

## read repair: 읽으면서 고친다

8단계에서 digest가 일치하지 않았다면 **replica들의 데이터가 서로 다르다**는 뜻이다. 어떤 replica는 최신 쓰기를 받았고, 어떤 replica는 hint 만료, 일시적인 다운, 패킷 유실 등으로 쓰기를 받지 못해 stale 상태다. coordinator는 이 상황을 그냥 넘기지 않는다.

coordinator는 불일치를 감지하면 관련된 모든 replica에서 데이터를 모아 timestamp로 **최신 값을 결정**하고, 뒤처진 replica에 그 최신 값을 **다시 써서** 맞춘다. 이것이 **read repair**다. 이름 그대로 "읽는 김에 고친다." 읽기 트래픽이 발생할 때마다 정합성이 자연스럽게 맞춰지는, 가벼운 형태의 anti-entropy다.

단, read repair는 **이번에 읽은 범위(쿼리한 partition/row)에 한해서만** 고친다. 파티션 전체나 테이블 전체를 훑어서 맞추는 것이 아니다. 그래서 한 번도 읽히지 않은 데이터는 read repair만으로는 끝내 맞춰지지 않는다.

- `CL=QUORUM` 이상으로 읽으면, 불일치가 있을 때 최신 값을 확정해 **클라이언트에는 올바른 값**을 주고 동시에 stale replica를 고친다. 이를 blocking read repair라고 하며, CL을 만족시키기 위해 동기적으로 보정한다.
- 자주 읽히지 않는 데이터에는 read repair가 동작할 기회 자체가 없다. 그래서 read repair만 믿어서는 안 되며, 전체 정합성을 지키는 최종 안전장치는 **주기적인 `nodetool repair`(anti-entropy repair)** 다.

> 과거 버전에 있던 `read_repair_chance`/`dc_local_read_repair_chance` 같은 "확률적 백그라운드 read repair" 테이블 옵션은 현대 Cassandra(4.0+, 5.0 포함)에서 제거되었다. 지금의 read repair는 **CL을 기준으로 불일치가 감지될 때 동작**하는 방식(`read_repair = BLOCKING` 기본)이다. 오래된 블로그 글을 보고 `read_repair_chance`를 설정하려다 에러를 만나는 경우가 흔하다.

---

## speculative retry: 느린 replica를 우회한다

가장 가까운 replica(node C)에 full read를 보냈는데, 그 노드가 죽지는 않았지만 **느리다면(GC 멈춤, 디스크 지연 등)** 어떻게 될까? 그 노드 하나 때문에 전체 읽기 latency가 늘어난다. 이를 막는 것이 **speculative retry(추측성 재시도)** 다.

coordinator는 full read를 보낸 replica가 정해진 시간 안에 응답하지 않으면 **기다리지 않고 다른 replica에 같은 요청을 추가로 보낸다.** 그리고 둘 중 먼저 온 응답을 쓴다. 이렇게 느린 노드를 우회해 꼬리 지연(tail latency, p99)을 줄인다.

speculative retry는 테이블 속성 `speculative_retry`로 제어하며, Cassandra 5.0의 기본값은 **`99p`**(또는 `99PERCENTILE`)다. 이 테이블 읽기 지연의 99 백분위수를 넘기면 추가 요청을 보내는 적응형 정책이다. 고정 시간(`50ms`), `ALWAYS`, `NONE` 등으로도 설정할 수 있다.

```cql
-- 결제 조회처럼 p99를 타이트하게 잡고 싶을 때 (예시)
ALTER TABLE payments.transactions
  WITH speculative_retry = '95p';
```

> [!note] 트레이드오프
> speculative retry는 가끔 같은 읽기를 두 replica에 보내므로 **읽기 부하가 늘어날 수 있다.** 클러스터가 이미 read I/O로 포화 상태라면 추측성 재시도가 오히려 부하를 키워 상황을 악화시킬 수 있다. p99를 줄이려다 평균을 망치지 않도록, 공격적인 설정은 부하에 여유가 있을 때만 쓴다.

---

## last-write-wins의 함정: 동일 timestamp 충돌

앞에서 병합 규칙은 "timestamp가 큰 cell이 이긴다(last-write-wins)"라고 했다. 그렇다면 **두 cell의 timestamp가 정확히 같으면** 어떻게 될까? 이것은 단순한 호기심의 문제가 아니라, 결제 시스템에서 실제로 데이터를 잃게 만드는 함정이다.

Cassandra의 write timestamp 기본 단위는 **마이크로초**이고, 클라이언트가 명시하지 않으면 coordinator가 현재 시각으로 자동 부여한다. 충분히 정밀해 보이지만, 동시성이 높으면 같은 마이크로초에 두 쓰기가 들어오는 일이 얼마든지 생긴다. 같은 키의 같은 컬럼에 같은 timestamp로 두 값이 들어오면 Cassandra는 시각으로 우열을 가리지 못한다. 이때의 tie-break 규칙은 다음과 같다.

> **timestamp가 같으면, cell 값(value)을 바이트로(unsigned) 비교해 사전순으로 큰 쪽이 이긴다.** (deterministic하지만 의미론적으로는 무의미한 기준)

즉 "나중에 보낸 쓰기"가 이긴다는 보장이 전혀 없고, 값의 바이트가 더 큰 쪽이 이긴다. 의미상 동등하지 않은 두 쓰기의 timestamp가 같으면, **하나는 조용히 사라지고, 어느 쪽이 남을지는 비즈니스 관점에서 임의적**이다. 충돌이 났다는 경고조차 없다. RDB의 `UPDATE`라면 락으로 직렬화되어 두 쓰기가 순서대로 반영되지만, Cassandra는 둘 중 하나를 아무 알림 없이 버린다.

한 가지 규칙만은 예외적으로 결정적이다. **같은 timestamp에서는 삭제(tombstone)가 일반 쓰기를 항상 이긴다.** 같은 마이크로초에 "값 쓰기"와 `DELETE`가 충돌하면, 값이 무엇이든 데이터가 사라지는 쪽으로 결정된다. "재생성(upsert)"과 "삭제"가 같은 시각에 경합하는 모델에서는 삭제가 눈에 띄지 않게 이겨 버리는 또 다른 함정이 된다.

이 함정이 특히 위험한 경우는 다음과 같다.

- **클라이언트가 timestamp를 직접 지정(`USING TIMESTAMP`)** 하면서 같은 값을 재사용할 때. "멱등성을 위해 timestamp를 고정한다"는 패턴을 잘못 적용하면 의도와 다른 쓰기가 무시될 수 있다.
- **서로 다른 노드의 시계가 어긋났을 때**. coordinator가 timestamp를 부여하는데 노드 간 시계 동기화(NTP)가 깨지면, 물리적으로 나중에 일어난 쓰기가 더 작은 timestamp를 받아 **과거의 쓰기에 밀려 버린다.** 새 쓰기가 무시되는 역전이 생기는 것이다. Cassandra 클러스터에서 **NTP 시계 동기화가 정합성의 전제 조건**인 이유가 여기에 있다.

> [!note] 결제 엔지니어를 위한 함의
> LWW 위에서 "마지막 상태 업데이트가 적용되었을 것"이라고 가정해서는 안 된다. 상태 전이가 중요한 도메인(예: `READY → PAID → CANCELLED`)에서는 같은 cell을 덮어쓰는 모델 대신, **append-only 이벤트로 쌓고 읽을 때 재구성**하는 모델([[13 - 실전 Event Sourcing on Cassandra]])을 쓰거나, 진짜 선형성이 필요하면 **LWT(경량 트랜잭션, 11장 LWT Batch Counter 내부)** 를 써야 한다.
>
> LWT는 Paxos로 직렬화해 "조건부 쓰기"를 보장하지만 비용이 훨씬 크다. LWW에서는 데이터가 경고 없이 사라질 수 있다는 점을 의식하고 모델을 설계해야 한다.

---

## tombstone이 읽기 성능을 떨어뜨리는 방식

이제 읽기 path에 가장 심각한 피해를 주는 주제인 **tombstone**을 살펴보자. 앞에서 "`DELETE`는 데이터를 지우지 않고 tombstone을 append한다"고 했다. 왜 바로 지우지 않을까? 분산 환경에서 삭제를 안전하게 전파하려면 삭제했다는 사실 자체를 **데이터처럼** 복제하고 병합해야 하기 때문이다.

어떤 replica가 다운된 사이에 삭제가 일어났는데 데이터를 즉시 물리적으로 지워 버렸다고 하자. 그 replica가 옛 데이터를 가진 채 복귀하면, read repair가 옛 데이터를 더 최신이라고 잘못 판단해 **삭제한 데이터가 부활(zombie)** 할 수 있다. tombstone은 "이 데이터는 t 시점에 삭제되었다"는 사실을 명시적으로 남겨 이런 부활을 막는다.

문제는 읽기다. tombstone은 `gc_grace_seconds`(기본 **864000초 = 10일**) 동안, 그리고 compaction이 실제로 정리할 때까지 SSTable에 남는다. 그동안 그 키를 읽으면 Cassandra는 **tombstone도 모두 읽어서 병합 과정에 포함**해야 한다. 어떤 cell이 삭제되었는지 확인하려면 tombstone을 봐야 하기 때문이다.

### 진짜 문제: range scan 범위에 쌓인 tombstone

cell 하나를 삭제한 것은 별문제가 아니다. 심각한 문제는 **한 partition 안에서 여러 row를 읽는 스캔**(clustering 범위 쿼리)에서 생긴다. 그 범위에 tombstone이 잔뜩 끼어 있으면, Cassandra는 클라이언트에 돌려줄 **살아 있는 row 1개를 찾으려고 수천 개의 tombstone을 읽고 버리는** 일을 한다. 응답에는 1건만 나오지만 내부적으로는 수만 개를 스캔한 셈이다.

Cassandra는 이런 상황을 감지하려고 임계값 두 개를 둔다(둘 다 정확한 5.0 기본값이다).

| 설정 (`cassandra.yaml`) | 기본값 | 동작 |
|---|---|---|
| `tombstone_warn_threshold` | **1000** | 한 쿼리가 이 수만큼 tombstone을 읽으면 로그에 **경고** |
| `tombstone_failure_threshold` | **100000** | 이 수를 넘으면 쿼리를 **중단하고 `TombstoneOverwhelmingException`** |

tombstone이 10만 개를 넘으면 그 쿼리는 **아예 실패한다.** 데이터가 멀쩡히 있어도 읽을 수 없게 된다. 운영 로그에서 `TombstoneOverwhelmingException`을 보면 이 절의 내용을 떠올려야 한다.

```text
   SELECT * FROM queue WHERE topic = 'jobs' LIMIT 1;   -- 살아있는 job 1개만 원함

   partition "jobs" 의 clustering 순서:
   [✝][✝][✝][✝][✝][✝]...(처리되어 DELETE된 수만 개)...[✝][✝][LIVE]
    └───────────────────── 다 읽고 버림 ─────────────────────┘  └─ 겨우 이 1건

   tombstone 1,000개 → WARN 로그
   tombstone 100,000개 → TombstoneOverwhelmingException, 쿼리 실패
```

### 큐 안티패턴: Cassandra로 큐를 만들면 안 되는 이유

이제 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]에서 다룬 악명 높은 안티패턴이 왜 그렇게 위험한지 tombstone 관점에서 정확히 설명할 수 있다.

큐(queue)의 본질은 "**한쪽 끝에서 넣고 다른 쪽 끝에서 빼는**" 것이다. Cassandra로 큐를 만들면 다음과 같은 일이 일어난다.

1. 작업을 한 partition에 clustering으로 계속 `INSERT`한다(한쪽에 쌓인다).
2. 처리한 작업을 `DELETE`하면 **partition 앞쪽에 tombstone이 끝없이 쌓인다**.
3. 다음에 처리할 작업을 읽으려고 `SELECT ... LIMIT 1`을 실행하면, **앞쪽에 쌓인 수만 개의 tombstone을 모두 건너뛴 뒤에야** 살아 있는 작업에 도달한다.

처리량이 늘수록 tombstone이 빠르게 쌓이고, 어느 순간 `tombstone_failure_threshold`를 넘겨 **큐를 읽는 쿼리 자체가 실패**한다. 데이터를 지웠는데 오히려 읽기가 점점 느려지다가 결국 실패하는, 직관과 정반대인 현상이다. 게다가 tombstone은 `gc_grace_seconds`(10일) 동안 지울 수 없으므로, 트래픽이 많으면 compaction이 따라잡지도 못한다.

> [!note] 규칙
> "삭제가 잦은 좁은 범위를 반복해서 range scan"하는 패턴은 모두 Cassandra와 맞지 않는다. 큐, 최신 N개를 계속 교체하는 버퍼, TTL로 대량 만료되는 시계열의 앞부분 스캔 등이 모두 여기에 해당한다. 큐가 필요하면 Kafka/SQS 같은 전용 시스템을 쓰고, Cassandra는 "삭제보다 추가가 압도적으로 많은" 워크로드에 써야 한다.
>
> 만료가 꼭 필요하다면 `DELETE` 대신 **TTL + 적절한 compaction 전략(TWCS 등, 9장 Compaction 전략)** 으로 partition 단위로 통째로 제거해서 cell 단위 tombstone 스캔을 피한다.

tombstone을 줄이는 실전 방법은 12장(운영과 트러블슈팅)에서 더 다룬다. 요점만 짚으면, `gc_grace_seconds`를 무작정 줄이면 zombie 위험이 커지므로 repair 주기와 함께 검토해야 하고, 데이터 모델 단계에서 삭제 패턴을 아예 만들지 않는 것이 가장 좋다.

---

## 두 path를 나란히: 최종 비교

마지막으로 두 path를 다이어그램 두 개와 표 하나로 나란히 비교한다.

```text
            ┌──────────────── WRITE PATH ────────────────┐
  Client ──▶│ Coordinator                                │
            │  partitioner→replica, snitch로 정렬        │
            └──┬──────────┬──────────┬───────────────────┘
               ▼          ▼          ▼
            node C     node F     node J(down)
          commitlog+  commitlog+   ✘ → coordinator가
          memtable    memtable        hint 저장 (복귀시 재전송)
               │          │
               └────┬─────┘
                    ▼  CL만큼 ack 모이면
              Client에 "성공"  (compaction은 나중에)


            ┌──────────────── READ PATH ─────────────────┐
  Client ──▶│ Coordinator                                │
            │  partitioner→replica, snitch로 정렬        │
            │  최근접=FULL, 나머지=DIGEST                │
            └──┬──────────────────┬──────────────────────┘
               ▼ full             ▼ digest
            node C              node F
       row cache?               (해시)
       └Bloom filter(SKIP/통과)    │
        └key cache→summary→index→Data.db
         └memtable+SSTable들 cell병합(LWW, tombstone판정)
               │                  │
               └───────┬──────────┘
                       ▼ digest 비교
              일치→반환 / 불일치→read repair
                       ▼
                    Client
```

| 항목 | Write | Read |
|---|---|---|
| 노드 간 라우팅 | partitioner+snitch (동일) | partitioner+snitch (동일) |
| replica에 보내는 것 | 모두에게 mutation | 최근접 full + 나머지 digest |
| 노드 안 동작 | commitlog append + memtable put | row cache→Bloom→keycache→summary→index→Data, 병합 |
| 충돌 해소 | 안 함(그냥 append) | cell 단위 LWW (동timestamp는 값 바이트 비교) |
| 실패 보완 | hinted handoff | read repair, speculative retry |
| 주된 비용 | 거의 없음(미룸) | SSTable 수·tombstone·캐시 miss |
| 최악의 적 | 거의 없음 | tombstone 누적 (큐 안티패턴) |

이 비대칭은 Cassandra 데이터 모델링의 제1원칙([[05 - 데이터 모델링 1 - Query First]])과 직접 연결된다. 읽기가 비싸기 때문에 **"읽을 쿼리를 먼저 정하고 거기에 맞춰 테이블(=partition)을 설계"** 한다. 반면 쓰기는 싸므로, 중복을 감수하고 여러 테이블에 같은 데이터를 넣어도 된다. 이 장에서 본 내부 메커니즘이 그 모델링 원칙의 물리적 근거다.
