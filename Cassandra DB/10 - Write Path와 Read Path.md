---
title: Write Path와 Read Path - Bloom filter, partition index, caching
date: 2026-06-26
tags: [cassandra, write-path, read-path, bloom-filter, 학습노트]
---

이 장이 답하려는 핵심 질문은 다음과 같다.

- 클라이언트가 `INSERT` 한 줄을 던졌을 때, 그 요청은 어떤 노드들을 거쳐 어디에 기록되고, 클라이언트는 언제 "성공"을 돌려받는가?
- Cassandra의 쓰기가 "디스크 데이터베이스인데도" 빠른 진짜 이유는 무엇인가?
- 같은 partition key를 `SELECT` 했을 때, 데이터가 memtable과 여러 SSTable에 흩어져 있다면 그것을 어떻게 하나의 최신 행으로 조립하는가?
- 단일 노드 안에서 "이 SSTable에 내가 찾는 키가 있는가?"를 매번 디스크를 뒤지지 않고 어떻게 빠르게 판단하는가 — Bloom filter, key cache, partition index, row cache의 역할 분담.
- 왜 삭제(`DELETE`)가 읽기 성능을 망가뜨리는가, 그리고 큐(queue)를 Cassandra로 구현하면 안 되는 이유는 무엇인가?
- 같은 timestamp를 가진 두 쓰기가 충돌하면 무슨 일이 벌어지는가?

RDB에 익숙한 엔지니어라면 직관을 한 번 뒤집어야 한다. RDB에서는 보통 **쓰기가 비싸고(인덱스 갱신, 락, fsync) 읽기가 싸다(B-tree 한 번 타면 끝)**. Cassandra는 정반대다. **쓰기는 거의 공짜에 가깝고, 읽기가 본래 더 많은 일을 한다.** 이 비대칭을 이해하는 것이 이 장 전체의 뼈대다.

---

## 두 path를 한눈에: 비대칭의 기원

먼저 큰 그림을 표로 잡자. 같은 데이터를 두고 쓰기와 읽기가 내부적으로 얼마나 다른 양의 일을 하는지 대비하면 나머지가 전부 따라온다.

| 측면 | Write Path | Read Path |
|------|-----------|-----------|
| 디스크 접근 패턴 | **순차 append** (commit log에 덧붙이기) | **랜덤 read** 가능성 (여러 SSTable 조회) |
| 손대는 자료구조 | commit log + memtable(메모리) | memtable + N개 SSTable + 여러 cache |
| 정렬·병합 | 없음 (그냥 기록) | **있음** (timestamp 기준 cell 단위 병합) |
| 기존 데이터를 읽는가? | **읽지 않는다** (read-before-write 없음) | 당연히 읽는다 |
| 비용을 지불하는 시점 | 거의 없음, compaction으로 **나중에 미룸** | **지금** 전부 지불 |
| 실패 시 보완 | hinted handoff | read repair |

이 표의 핵심은 마지막에서 두 번째 줄이다. Cassandra는 쓰기 비용을 "지금" 내지 않고 **compaction이라는 백그라운드 작업으로 미룬다**([[09 - Compaction 전략]]). 그 대신 읽을 때 "여기저기 흩어진 조각을 모으는" 빚을 갚는다. LSM-tree 계열 스토리지의 본질적 트레이드오프이며 write-heavy 워크로드(로그, 이벤트, 시계열, 결제 이벤트 소싱)에서 Cassandra가 강력한 이유이기도 하다.

```text
        쓰기: 비용을 미래로 미룬다              읽기: 미뤄둔 비용을 지금 갚는다
   ┌───────────────────────────┐        ┌───────────────────────────────┐
   │ commit log append (순차)  │        │ memtable + SSTable_1 ... _N    │
   │ memtable put (메모리)     │        │ 각각에서 같은 키 조각 수집     │
   │ → 끝. ack.                │        │ → timestamp로 병합(LWW)        │
   └───────────────────────────┘        └───────────────────────────────┘
            싸다 / 빠르다                        비싸다 / 최적화 계층 필요
```

이제 각 path를 분산 레벨(노드 간)과 로컬 레벨(노드 안)으로 나누어 깊게 들어간다.

---

## Write Path (1): coordinator가 replica를 찾고 정렬한다

클라이언트가 던진 한 줄의 쓰기는 두 단계의 라우팅을 거친다. 첫째는 **노드 간(inter-node)** 단계 — 어느 노드들이 이 데이터를 책임지는가. 둘째는 **노드 안(intra-node)** 단계 — 각 노드는 그것을 어떻게 디스크에 안착시키는가. 이 절은 첫 번째 단계다.

### coordinator라는 임시 지휘자

클라이언트(정확히는 드라이버)는 클러스터의 아무 노드에나 요청을 보낼 수 있다. 그 요청을 처음 받은 노드가 그 요청 한 건에 한해 **coordinator(코디네이터)** 역할을 맡는다. coordinator는 고정된 직책이 아니다. 요청마다 달라지는 **임시 지휘자**다. 같은 클라이언트의 다음 요청은 다른 노드가 coordinator가 될 수 있다.

> **흔한 오해**: "coordinator 노드가 따로 있다." 아니다. 모든 노드가 똑같이 coordinator가 될 수 있다. Cassandra에 마스터가 없는 것(masterless)과 같은 맥락이다. 드라이버의 `TokenAwarePolicy`는 보통 **데이터를 실제로 가진 replica 중 하나를 coordinator로 고르도록** 라우팅해서 불필요한 홉(hop)을 줄인다.

### partitioner로 replica를 산출한다

coordinator는 가장 먼저 "이 쓰기가 어느 노드들로 가야 하는가"를 계산한다.

1. **partition key → token**: row의 partition key를 partitioner(기본 `Murmur3Partitioner`)에 넣어 64비트 토큰을 얻는다. 이건 순수 해시 계산이라 어떤 노드에서 해도 같은 답이 나온다. 네트워크가 필요 없다.
2. **token → primary replica**: 토큰 링 위에서 그 토큰을 "소유하는" 노드(시계 방향 첫 노드)가 primary replica다.
3. **token → 나머지 replica**: 복제 계수(replication factor, RF)가 3이면 링을 따라 다음 노드 2개를 더 고른다. 이때 단순히 "다음 노드"가 아니라 [[03 - 복제 전략과 데이터센터]]에서 본 replication strategy(`NetworkTopologyStrategy`)와 snitch가 개입해서 **같은 rack에 몰리지 않게, 각 데이터센터에 정해진 수만큼** 배치된 replica 집합을 돌려준다.

여기서 짚어둘 직관이 하나 있다. **replica가 어디인지 알아내는 데는 클러스터에 물어볼 필요가 없다.** 토큰 링의 소유권 정보는 gossip을 통해 모든 노드가 이미 알고 있다. 그래서 coordinator는 **순수 계산만으로** 목적지 N개를 즉시 안다. RDB의 샤딩 라우터가 메타데이터 서버에 "이 키 어디 있어?"라고 물어보는 것과 대조된다.

### snitch로 replica를 "정렬"한다

산출된 replica 집합은 순서 없는 후보일 뿐이다. coordinator는 여기에 **snitch**를 적용해 "누가 더 가까운가(network distance)"로 정렬한다. snitch는 각 노드의 데이터센터·rack 토폴로지를 알고 있어서 coordinator 자신과 같은 DC·같은 rack에 있는 replica를 "가깝다"고 판단한다. 그 위에 다시 **dynamic snitch**가 얹혀 정적 토폴로지뿐 아니라 **최근 관측된 응답 지연·부하**까지 반영해 순위를 실시간으로 재배열한다. 그래서 뒤의 읽기 절에서 말하는 "가장 가까운 replica"는 물리적 거리만이 아니라 **지금 가장 빠른 replica**에 가깝다. GC로 잠깐 느려진 노드는 자동으로 순위가 밀린다.

쓰기에서 이 정렬이 결정적으로 중요한 건 아니다(쓰기는 어차피 일관성 레벨(consistency level, CL)만큼의 replica로 동시에 보낸다). 하지만 같은 메커니즘이 **읽기에서는 "가장 가까운 replica에게만 full read를 시키는"** 핵심 최적화로 쓰인다. 그래서 여기서 정렬 개념을 잡아두면 읽기 절이 쉬워진다.

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

## Write Path (2): 각 replica 안에서 — commit log + memtable

이제 두 번째 단계, 노드 안에서 벌어지는 일이다. coordinator가 보낸 mutation을 받은 각 replica는 [[08 - Storage Engine 내부]]에서 배운 그 절차를 그대로 실행한다. **단 두 곳에만 쓴다**는 게 요점이다.

### 1단계: commit log에 append (내구성)

mutation은 먼저 **commit log**에 순차적으로 덧붙여진다. 이것은 디스크에 쓰는 동작이지만 **순차 append**라서 빠르다. 디스크의 헤드를 여기저기 움직이는 랜덤 쓰기가 아니라, 파일 끝에 계속 이어 붙이기 때문이다. SSD에서도 순차 쓰기가 랜덤 쓰기보다 유리하고 HDD에서는 그 차이가 압도적이다.

commit log의 목적은 단 하나, **내구성(durability)**이다. memtable은 메모리에 있어서 노드가 죽으면 날아간다. commit log가 디스크에 먼저 남으므로 노드가 재시작하면 commit log를 replay해서 memtable을 복원할 수 있다.

> Cassandra 5.0 기본값으로 commit log는 **`periodic` 모드**, `commitlog_sync_period`는 보통 10초(`10000ms`)다. 매 쓰기마다 fsync로 디스크에 강제 동기화하지 않고 **주기적으로 묶어서 flush**한다. 이게 "디스크 DB인데 빠른" 또 하나의 비밀이다. 대신 그 10초 창(window) 안에 OS까지 같이 죽으면 그 구간 데이터는 잃을 수 있다. 절대적 내구성이 필요하면 `batch` 모드로 바꿀 수 있지만 처리량이 크게 떨어진다. 결제처럼 손실이 곤란한 데이터는 이 트레이드오프를 의식적으로 선택해야 한다.

### 2단계: memtable에 put (조회 가능 상태)

같은 mutation이 **memtable**(메모리 안의 정렬된 자료구조, 테이블별로 존재)에도 기록된다. memtable에 들어가는 순간 그 데이터는 즉시 읽기 대상이 된다. 디스크에 SSTable로 내려가기를 기다릴 필요가 없다.

여기서 RDB와의 결정적 차이가 나온다.

> **Cassandra의 쓰기에는 read-before-write가 없다.** `UPDATE`조차 기존 값을 읽어오지 않는다. "user_42:txn_998 의 status 컬럼에 timestamp T로 PAID를 기록"이라는 셀(cell)을 그냥 append할 뿐이다. 기존에 같은 컬럼 값이 있든 없든 신경 쓰지 않는다. 충돌 해소는 **읽을 때** timestamp로 한다(last-write-wins). RDB라면 `UPDATE`가 해당 row를 찾아 락 걸고 제자리(in-place)에서 고치지만 Cassandra는 "예전 값 위에 덮어쓰기"라는 개념 자체가 없다. 모든 쓰기는 새 버전의 추가다.

이 "read-before-write 없음"이 쓰기를 빠르게 하는 가장 근본적인 이유다. 읽기·락·인덱스 트리 재균형 같은 비싼 작업이 쓰기 경로에서 전부 빠진다.

> **예외 — read-before-write가 필요한 쓰기**: 일반 `INSERT`/`UPDATE`는 기존 값을 안 읽지만, *모든* 쓰기가 그런 건 아니다. (1) **counter** 컬럼은 현재값을 알아야 증감하므로 내부적으로 read를 동반한다. (2) **list** 컬렉션의 일부 연산(인덱스 기반 set/삭제, prepend 등)도 기존 원소를 읽어야 한다. list보다 set/map이 권장되는 이유 중 하나다. (3) **LWT(경량 트랜잭션)**는 `IF` 조건을 검사하려고 Paxos read를 한다([[11 - LWT Batch Counter 내부]]). 이들이 하나같이 "비싼 쓰기"로 분류되는 근본 이유가 바로 read-before-write의 부활이다.

memtable은 영원히 메모리에 남지 않는다. 전체 memtable이 쓰는 메모리(`memtable_heap_space`/`memtable_offheap_space`)가 한도를 넘거나, commit log 총량(`commitlog_total_space`)이 한계에 닿거나, `nodetool flush`가 호출되면, 가장 큰 memtable이 **flush**되어 불변(immutable) SSTable로 디스크에 내려가고, 그에 대응하는 commit log 구간은 더 이상 필요 없으니 회수된다. (예전의 `memtable_cleanup_threshold`도 이 판단에 관여했으나 최신 버전에선 deprecated이다.) 이 flush와 이후의 compaction이 바로 "미뤄둔 비용"이다.

### 3단계: CL만큼 ack를 모아 클라이언트에 응답

각 replica는 commit log append + memtable put을 마치면 coordinator에게 ack를 보낸다. coordinator는 요청에 지정된 **일관성 레벨(consistency level)**만큼 ack가 모이면 클라이언트에게 성공을 반환한다([[04 - Tunable Consistency]]).

- `CL=ONE`: replica 1개의 ack만 오면 즉시 성공.
- `CL=QUORUM`(RF=3이면 2개): 2개 ack를 기다린다.
- `CL=ALL`: 3개 전부.

여기엔 미묘한 점이 있다. coordinator는 **모든 replica에게 동시에** mutation을 보낸다. CL은 "몇 개에게 보낼까"가 아니라 "**몇 개의 성공을 기다렸다가 클라이언트에 OK라고 말할까**"이다. `CL=ONE`이어도 나머지 2개 replica에도 쓰기는 간다. 단지 그 ack를 기다리지 않을 뿐이다.

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

위 그림에서 node J가 다운되었거나 느리다고 하자. `CL=QUORUM`이면 C, F 두 개의 ack로 이미 성공을 반환했다. 그런데 node J는 이 쓰기를 못 받았다. 나중에 살아나면 데이터가 빠진 채로 클러스터에 복귀한다. 이 구멍을 메우는 메커니즘이 **hinted handoff**다.

coordinator는 쓰기 시점에 어떤 replica가 다운인지(gossip으로) 알고 있다. 다운된 replica로 보낼 mutation을 **hint(힌트)**로 자기 로컬에 저장해 둔다. hint는 "node J에게 줄, 아직 못 준 쓰기 X"라는 메모다. node J가 다시 살아나면 coordinator가 보관하던 hint를 재전송해서 누락분을 채운다.

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

몇 가지 중요한 디테일.

- **hint는 무한히 보관하지 않는다.** `max_hint_window`(Cassandra 5.0 기본 **3시간**, `max_hint_window_in_ms` 또는 `max_hint_window=3h`) 동안만 보관한다. 이 창을 넘겨 다운된 노드는 hint로 복구되지 않으므로 복귀 후 반드시 `nodetool repair`로 데이터를 맞춰야 한다.
- **hint는 일관성 보장 장치가 아니다.** hint가 coordinator 디스크에 있는 동안 그 coordinator마저 죽으면 hint도 사라진다. hinted handoff는 "잠깐 깜빡인 노드를 빨리 따라잡게 하는 편의 장치"이지, 정합성의 최후 보루가 아니다. **최후 보루는 read repair와 anti-entropy repair**다.
- **CL 충족에 hint를 세지 않는다(기본).** `CL=QUORUM`을 달성하려면 hint 저장이 아니라 **실제 replica의 ack**가 필요하다. 만약 살아있는 replica 수가 CL보다 적으면 쓰기는 `UnavailableException`으로 실패한다. (예외적으로 `CL=ANY`는 hint 저장만으로도 성공으로 치는데, 데이터가 어떤 정식 replica에도 안 들어갈 수 있어 위험하다. 결제 시스템에서는 거의 쓰지 않는다.)

> **결제 엔지니어를 위한 함의**: hinted handoff 덕분에 노드 하나가 잠깐 재시작해도 쓰기가 실패하지 않고 흡수된다. 하지만 "노드가 3시간 넘게 죽어 있었다"면 hint는 만료됐을 수 있다. 운영 룰: **장시간 다운된 노드 복귀 후에는 무조건 repair**. 이를 빼먹으면 그 노드에서 읽힐 때 stale data가 나올 수 있다. ([[12 - 운영과 트러블슈팅]]에서 더 다룬다.)

### 그래서 왜 쓰기가 빠른가 — 한 문단 정리

쓰기가 빠른 이유를 한 곳에 모으면 이렇다. (1) replica 위치를 **계산으로** 알아내 라우팅 비용이 거의 없다. (2) 각 노드에서 디스크 접근은 commit log **순차 append** 하나뿐이고, 나머지는 메모리(memtable)다. (3) **read-before-write가 없어** 락·인덱스 재균형·기존값 조회가 전부 빠진다. (4) commit log fsync를 **묶어서(periodic)** 처리해 fsync 빈도를 낮춘다. (5) SSTable 정리·병합 같은 무거운 일은 **compaction으로 미룬다**. 즉 Cassandra의 쓰기는 "지금 최소한만 하고 나중에 정리한다"는 전략의 결정체다.

---

## Read Path (1): coordinator의 full read + digest 전략

이제 빚을 갚을 시간이다. 읽기도 쓰기처럼 노드 간 단계와 노드 안 단계로 나뉜다. 노드 간 단계부터.

coordinator는 쓰기와 똑같이 partitioner로 replica 집합을 산출하고 snitch로 거리순 정렬한다. 여기까지는 같다. 다른 점은 그다음이다.

읽기는 **모든 replica에게 전체 데이터를 요청하지 않는다.** 네트워크와 디스크를 아끼기 위해 영리한 분업을 한다.

- 가장 가까운(snitch 기준) replica 1개에게만 **full read(데이터 본문 전체)**를 요청한다.
- 일관성 레벨(consistency level)을 채우는 데 필요한 나머지 replica들에게는 **digest read**를 요청한다. digest는 데이터 본문이 아니라 그 데이터의 **해시(체크섬)**다.

이렇게 하는 데는 이유가 있다. CL을 만족하려면 여러 replica의 답을 비교해 "이 값이 정말 최신이고 일치하는가"를 확인해야 한다. 그런데 일치 여부만 확인하려면 **전체 데이터를 다 받을 필요 없이 해시만 비교하면 충분하다.** full read는 무겁고(수 KB~수 MB일 수 있다), digest는 가볍다(고정 크기 해시). coordinator는 full read 1개 + digest N개를 받아 **digest들이 full read의 digest와 모두 일치하는지** 본다.

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

`CL=QUORUM`, RF=3이면 coordinator는 2개 replica의 답(full 1 + digest 1)을 모은다. 만약 둘의 digest가 일치하면 그 데이터를 클라이언트에 돌려준다. 일치하지 않으면(= replica 간 데이터가 다르면) **read repair**가 발동한다. `CL=ONE`이면 가장 가까운 replica의 full read 하나만 받고 끝이다. digest 비교도, 일관성 검증도 없다. 빠르지만 stale data를 읽을 수 있다.

이 분업은 [[04 - Tunable Consistency]]의 `R + W > RF` 공식과 직접 연결된다. 읽기 CL과 쓰기 CL의 합이 RF를 넘으면 "읽은 replica 집합과 쓴 replica 집합이 반드시 겹치므로" 최신 값을 본다는 보장이 생긴다. 그 "비교"를 실제로 수행하는 메커니즘이 바로 이 full read + digest 대조다.

---

## Read Path (2): 단일 노드 안에서 — 조각을 모아 병합한다

coordinator가 node C에게 full read를 요청했다. 이제 node C **안에서** 무슨 일이 벌어지는가. 이게 읽기 path의 진짜 심장이다.

문제는 이거다. 같은 partition key "user_42:txn_998"의 데이터가 **한 군데 있지 않다.** [[08 - Storage Engine 내부]]에서 봤듯, 그 키에 여러 번의 쓰기가 시간에 걸쳐 일어났고 그 결과는:

- 아직 flush 안 된 최신 쓰기 → **memtable**에 있다.
- 예전에 flush된 쓰기들 → **SSTable_1, SSTable_2, ... SSTable_N**에 흩어져 있다. compaction이 다 합치기 전까지 같은 키의 조각이 여러 SSTable에 동시에 존재할 수 있다.

그래서 node C는 이 모든 출처에서 **같은 키의 조각들을 다 모아서** **cell(셀, 컬럼 값) 단위로 timestamp를 비교해 최신 것만 골라** 하나의 논리적 행으로 재조립한다. 이것이 **last-write-wins(LWW)** 병합이다.

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

여기서 강조할 점들.

- **병합은 행 단위가 아니라 cell 단위다.** `status`는 memtable의 t=105가 이기고, `amount`는 SSTable의 t=101이 이긴다. 한 행을 통째로 "최신 SSTable 것"으로 가져오는 게 아니다. 각 컬럼이 독립적으로 자기만의 최신 timestamp를 가진다. 그래서 부분 업데이트(`UPDATE`로 한 컬럼만 고침)가 자연스럽게 동작한다.
- **여러 SSTable을 봐야 한다는 게 읽기의 비용이다.** 같은 키가 몇 개의 SSTable에 흩어져 있느냐가 읽기 지연(latency)을 좌우한다. compaction이 SSTable을 합쳐 이 숫자를 줄여준다. [[09 - Compaction 전략]]에서 본 SizeTiered vs Leveled의 트레이드오프가 바로 여기서 체감된다. Leveled는 "한 키가 레벨당 하나의 SSTable에만 있도록" 정리해 읽기 시 봐야 할 SSTable 수를 줄인다(읽기 친화적, 대신 쓰기 증폭).
- **삭제도 하나의 "값"으로 병합에 참여한다.** `DELETE`는 데이터를 지우는 게 아니라 **tombstone(묘비)**이라는 특수 마커를 timestamp와 함께 append한다. 병합 시 tombstone의 timestamp가 가장 최신이면 그 cell은 "삭제됨"으로 판정된다. 이 tombstone이 읽기를 어떻게 망치는지는 뒤에서 따로 깊게 다룬다.

> **RDB였다면**: `SELECT`는 B-tree 인덱스를 타고 한 page를 읽으면 그 안에 완성된 row가 있다. in-place 갱신이라 "흩어진 조각"이라는 개념이 없다. Cassandra는 immutable append 구조의 대가로, 읽을 때마다 조각을 모으고 timestamp로 화해시키는 일을 한다. 이 차이가 "쓰기 싸고 읽기 비싸다"의 물리적 실체다.

그런데 잠깐, node C는 "이 partition key가 어느 SSTable에 들어있는지" 어떻게 알까? SSTable이 수십 개라면 전부 열어서 뒤질 수는 없다. 여기서 이 장의 주인공인 **읽기 최적화 계층**이 등장한다.

---

## Read Path (3): 단일 노드 내 읽기 최적화 계층 — 순서대로

node C가 한 partition key를 읽을 때, 디스크의 SSTable에 무작정 달려들지 않는다. **여러 겹의 필터와 캐시를 순서대로 통과**하면서 가능한 한 디스크 접근을 피하고, 피할 수 없으면 정확히 필요한 바이트만 읽도록 안내받는다. 이 계층을 순서대로 따라가 보자. 이게 이 장의 deep dive 본령이다.

### 0단계: row cache — 운이 좋으면 여기서 끝

가장 먼저, 그 partition(또는 partition의 앞부분)이 **row cache**에 통째로 있는지 본다. 있으면 SSTable이고 뭐고 볼 것 없이 메모리에서 즉시 반환한다. row cache는 강력하지만 위험한 옵션이라 뒤의 캐시 절에서 따로 자세히 다룬다. 기본은 **꺼져 있다**(0MB). 여기서는 "있으면 최단 경로"라고만 알아두자. 대부분의 클러스터는 row cache가 꺼져 있어서 다음 단계로 바로 간다.

### 1단계: Bloom filter — "이 SSTable엔 그 키 없음"을 즉시 판정

각 SSTable에는 **Bloom filter**가 딸려 있고 이건 항상 메모리에 올라가 있다. Bloom filter는 확률적 자료구조로, 단 하나의 질문에 매우 빠르게 답한다.

> "이 SSTable에 partition key K가 **있을 수도 있는가, 아니면 확실히 없는가**?"

Bloom filter의 두 가지 답과 그 신뢰도를 정확히 이해해야 한다.

| Bloom filter의 답 | 의미 | 신뢰할 수 있는가 |
|---|---|---|
| "없다 (negative)" | 이 SSTable엔 K가 **확실히 없다** | **100% 정확.** false negative 불가능 |
| "있을 수도 있다 (positive)" | 아마 있지만 아닐 수도 있다 | **확률적.** false positive 가능 |

이 비대칭이 Bloom filter의 전부다. **"없다"는 절대적으로 믿을 수 있다.** Bloom filter가 "없다"고 하면 그 SSTable은 디스크 접근 없이 **건너뛴다**. SSTable이 50개여도 Bloom filter가 48개에서 "없다"고 답하면 디스크는 나머지 2개만 본다. 이것이 읽기를 살리는 1차 방어선이다.

반대로 "있을 수도 있다"는 가끔 거짓말한다(**false positive**). 실제론 없는데 있다고 해서 헛디스크 읽기를 유발한다. 하지만 **false negative는 원리적으로 불가능**하다. 실제로 있는데 "없다"고 말하는 일은 없다. 그래서 데이터를 놓치는 일은 절대 없고 단지 가끔 헛수고할 뿐이다. 이 보장이 정확성(correctness)을 지킨다.

작동 원리는 간단하다. Bloom filter는 비트 배열 + 여러 해시 함수다. 키를 넣을 때 여러 해시 위치의 비트를 1로 켠다. 조회 시 그 위치들이 **모두 1이면 "있을 수도"**, **하나라도 0이면 "확실히 없음"**. 0이 하나라도 있다는 건 그 키를 넣은 적이 절대 없다는 뜻이므로 false negative가 불가능하다.

```text
   "user_42:txn_998" → h1,h2,h3 → 비트 위치 [5, 19, 88]

   SSTable_A bloom: ...1...1...1...  (5,19,88 모두 1) → "있을 수도" → 디스크 확인 필요
   SSTable_B bloom: ...1...0...1...  (19가 0)        → "확실히 없음" → SKIP! (디스크 안 봄)
```

#### bloom_filter_fp_chance — 정확도와 메모리의 거래

false positive가 얼마나 자주 나는지는 테이블 속성 **`bloom_filter_fp_chance`**로 조절한다. 값이 작을수록(예: 0.001 = 0.1%) false positive가 줄어 헛디스크 읽기가 줄지만 Bloom filter 비트 배열이 커져 **off-heap 메모리를 더 먹는다**(Cassandra의 Bloom filter는 off-heap에 상주하므로 JVM heap/GC에는 직접 부담을 주지 않지만, 노드 전체 메모리 예산은 갉아먹는다). 값이 클수록 메모리는 아끼지만 헛읽기가 늘어난다.

Cassandra 5.0의 기본값은 compaction 전략에 따라 다르다(이건 정확한 수치이니 외워둘 만하다).

| compaction 전략 | `bloom_filter_fp_chance` 기본값 |
|---|---|
| SizeTieredCompactionStrategy / UnifiedCompactionStrategy | **0.01** (1%) |
| LeveledCompactionStrategy | **0.1** (10%) |

왜 LCS의 기본 false positive가 더 헐겁(10%)을까? LCS는 구조상 한 키가 레벨당 하나의 SSTable에만 있어서 어차피 봐야 할 SSTable 수가 적다. Bloom filter에 메모리를 덜 써도 읽기가 충분히 빠르므로 메모리를 아끼는 쪽으로 기본값을 잡았다. 반대로 STCS/UCS는 같은 키가 여러 SSTable에 퍼질 수 있어 Bloom filter 정확도가 더 중요하니 1%로 빡빡하게 잡는다.

```cql
-- 읽기가 많고 메모리 여유가 있으면 false positive를 더 낮춰 헛읽기를 줄인다
ALTER TABLE payments.transactions
  WITH bloom_filter_fp_chance = 0.001;
```

> **함정**: `bloom_filter_fp_chance`를 0으로 만들 수는 없다. 0에 가까울수록 메모리가 가파르게 증가한다. 또 partition이 매우 많은(키 카디널리티가 높은) 테이블일수록 Bloom filter 메모리 총량이 커진다. 무작정 낮추면 heap 압박과 GC 문제로 번질 수 있다. [[12 - 운영과 트러블슈팅]]에서 메모리 진단과 함께 봐야 한다.

### 2단계: key cache — partition key의 디스크 위치를 기억

Bloom filter가 "있을 수도"라고 통과시켰다. 이제 "그럼 그 키는 이 SSTable의 Data.db 파일 **어느 바이트 오프셋**에 있나?"를 알아야 한다. 매번 인덱스를 디스크에서 뒤지면 느리다. 그래서 **key cache**가 있다.

key cache는 메모리에 `(SSTable, partition key) → Data.db 내 정확한 오프셋`을 캐시한다. **cache hit이면 partition index 조회 단계(아래 3·4단계)를 통째로 건너뛰고** Data.db의 그 위치로 바로 점프한다. 반복 조회되는 hot key의 읽기를 크게 가속한다.

Cassandra 5.0 기본으로 key cache는 **켜져 있다**. 크기는 `key_cache_size`로 정하며 기본은 "heap의 5% 또는 100MB 중 작은 값"(`min(5% of heap, 100MB)`)이다. 대체로 켜두는 게 이득이고 무효화(invalidation) 부담도 작아서 row cache처럼 위험하지 않다.

### 3단계: partition summary — 인덱스의 인덱스(샘플)

key cache가 miss면 partition index를 봐야 한다. 그런데 partition index 자체가 디스크에 있고 클 수 있다. 그래서 그 인덱스의 **샘플(sample)**을 메모리에 둔 것이 **partition summary**다.

partition summary는 partition index를 일정 간격(예: 매 128번째 키)으로 샘플링해, "키 K는 partition index 파일의 대략 이 근방에서 시작한다"를 알려준다. 인덱스 파일 전체를 스캔하지 않고 **좁은 구간으로 점프**하게 해주는 이정표다. "인덱스의 인덱스"라고 부르는 이유다.

샘플링 간격은 `min_index_interval`/`max_index_interval`(기본 128/2048)로 조절된다. 촘촘할수록 점프가 정확해 디스크 탐색이 줄지만 summary가 커져 메모리를 더 쓴다. summary는 메모리에 상주한다.

> 참고: Cassandra의 SSTable 포맷이 진화하면서 인덱스 구조의 세부(예: 5.0의 BTI/`trie` 인덱스 포맷 옵션)는 버전에 따라 달라질 수 있다. 개념적으로는 "메모리 상의 성긴 이정표(summary) → 디스크 상의 촘촘한 인덱스(index) → 데이터(data)"라는 3단 점프 구조로 이해하면 어떤 포맷에서도 통한다. (세부 포맷명은 버전 의존적이므로 단정하지 않는다.)

### 4단계: partition index → Data.db

summary가 가리킨 partition index 구간을 디스크에서 읽어, 찾는 partition key의 **정확한 Data.db 오프셋**을 얻는다. 그리고 마침내 **Data.db**의 그 위치에서 실제 데이터(셀들)를 읽는다. (key cache hit이었다면 2·3·4단계를 다 건너뛰고 바로 여기로 왔을 것이다.)

이 4단계 점프를 한 줄로 요약하면 이렇다.

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

이 과정을 **읽어야 할 SSTable마다** 반복한 뒤(Bloom filter가 통과시킨 것들만), 앞 절의 cell 단위 timestamp 병합으로 최종 행을 만든다.

### 그 아래의 캐시들: chunk cache와 OS page cache

Data.db를 실제로 디스크에서 읽을 때도 한 겹의 캐시가 더 있다.

- **chunk cache**(과거 이름 그대로 쓰면 file/row cache와 헷갈리니 주의): SSTable의 데이터는 보통 압축(LZ4 등)되어 **압축 청크(chunk)** 단위로 저장된다. 한 번 디스크에서 읽어 압축 해제한 청크를 메모리에 캐시해 두면, 같은 청크를 다시 읽을 때 디스크 I/O와 압축 해제 비용을 아낀다. off-heap에 둔다.
- **OS page cache**: 그 아래, 운영체제 레벨에서 파일 페이지를 캐시한다. Cassandra가 디스크라고 믿고 읽어도 실제로는 OS page cache에서 메모리 속도로 반환되는 경우가 많다. 그래서 메모리가 넉넉한 노드는 "디스크 DB"라기보다 사실상 메모리에서 읽는 것처럼 동작한다. 이게 Cassandra 노드에 RAM을 넉넉히 주라는 운영 권고의 근거다.

정리하면, 읽기 한 번이 마주치는 메모리 계층은 위에서 아래로: **row cache → (Bloom filter) → key cache → partition summary → partition index → chunk cache → OS page cache → 디스크**. 윗단에서 답이 나오면 아랫단은 건드리지 않는다.

---

## 읽기 한 번의 전체 시퀀스 — 끝까지 추적

흩어진 부품을 하나의 타임라인으로 묶자. `SELECT ... WHERE partition_key = 'user_42:txn_998'`, RF=3, CL=QUORUM 한 번이 처음부터 끝까지 거치는 단계 전부다.

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

이 시퀀스에서 latency를 결정하는 변수는 명확하다. (1) 봐야 할 SSTable 수(= compaction 상태), (2) Bloom filter false positive로 인한 헛읽기, (3) key cache hit율, (4) 병합 중 마주친 tombstone 수, (5) digest 불일치로 read repair가 끼어드는 빈도. 운영에서 읽기가 느리면 이 다섯 개를 순서대로 의심하면 된다.

---

## read repair — 읽으면서 고친다

8단계에서 digest가 불일치했다는 건 **replica들의 데이터가 서로 다르다**는 뜻이다. 어떤 replica는 최신 쓰기를 받았고 어떤 replica는 (hint 만료, 일시 다운, 패킷 유실 등으로) 못 받아 stale 상태다. coordinator는 이걸 그냥 넘기지 않는다.

coordinator는 불일치를 감지하면 모든 관련 replica에서 데이터를 모아 timestamp로 **최신값을 결정**하고 뒤처진 replica들에게 그 최신값을 **다시 써서** 맞춘다. 이것이 **read repair**다. 이름 그대로 "읽는 김에 고친다." 정합성이 읽기 트래픽을 타고 자연스럽게 수렴하는, anti-entropy의 가벼운 형태다. 단, read repair는 **이번에 읽은 범위(쿼리한 partition/row)에 한해서만** 고친다. 파티션 전체나 테이블 전체를 훑어 맞추는 게 아니다. 그래서 "한 번도 안 읽힌 데이터"는 read repair만으로는 영영 수렴하지 않는다.

- `CL=QUORUM` 이상으로 읽으면, 불일치 시 최신값을 확정해 **클라이언트에는 올바른 값**을 주고 동시에 stale replica를 고친다(이른바 blocking read repair — CL을 만족시키기 위해 동기적으로 보정).
- 자주 안 읽히는 데이터는 read repair가 발동할 기회 자체가 없다. 그래서 read repair만 믿으면 안 되고 **주기적인 `nodetool repair`(anti-entropy repair)**가 전체 정합성의 최후 보루다.

> 과거 버전의 `read_repair_chance`/`dc_local_read_repair_chance` 같은 "확률적 백그라운드 read repair" 테이블 옵션은 현대 Cassandra(4.0+, 5.0 포함)에서 제거되었다. 지금의 read repair는 **CL 기반으로 불일치가 감지될 때 동작**하는 방식(`read_repair = BLOCKING` 기본)이다. 오래된 블로그를 보고 `read_repair_chance`를 설정하려다 에러를 만나는 흔한 함정이다.

---

## speculative retry — 느린 replica를 추월한다

읽기에서 가장 가까운 replica(node C)에게 full read를 보냈는데, 그 노드가 죽지는 않았지만 **느리면(GC 멈춤, 디스크 지연 등)** 어떻게 될까? 그 한 노드 때문에 전체 읽기 latency가 끌려간다. 이를 막는 것이 **speculative retry(추측성 재시도)**다.

coordinator는 "full read를 보낸 replica가 정해진 시간 안에 응답하지 않으면, **기다리지 않고 다른 replica에게 같은 요청을 추가로 보낸다.**" 둘 중 먼저 오는 답을 쓴다. 느린 노드를 우회해 꼬리 지연(tail latency, p99)을 깎는다.

테이블 속성 `speculative_retry`로 제어하며, Cassandra 5.0 기본은 **`99p`**(또는 `99PERCENTILE`)다. "이 테이블 읽기 지연의 99 백분위수를 넘기면 추가 요청을 던진다"는 적응형 정책이다. 고정 시간(`50ms`), `ALWAYS`, `NONE` 등으로도 설정 가능하다.

```cql
-- 결제 조회처럼 p99를 타이트하게 잡고 싶을 때 (예시)
ALTER TABLE payments.transactions
  WITH speculative_retry = '95p';
```

> **트레이드오프**: speculative retry는 가끔 같은 읽기를 두 replica에 보내므로 **읽기 부하가 늘 수 있다.** 클러스터가 이미 read I/O로 포화 상태라면, 추측성 재시도가 오히려 부하를 키워 상황을 악화시킬 수 있다. p99를 줄이려다 평균을 망치지 않도록 부하 여유가 있을 때 공격적으로 잡는다.

---

## last-write-wins의 함정: 동일 timestamp 충돌

병합 규칙이 "timestamp가 큰 cell이 이긴다(last-write-wins)"라고 했다. 그런데 **두 cell의 timestamp가 정확히 같으면** 어떻게 될까? 이건 단순한 호기심이 아니라 결제 시스템에서 실제로 데이터를 잃게 하는 함정이다.

Cassandra의 write timestamp 기본 단위는 **마이크로초**이고 클라이언트가 명시하지 않으면 coordinator가 현재 시각으로 자동 부여한다. 충분히 정밀해 보이지만 동시성이 높으면 같은 마이크로초에 두 쓰기가 떨어지는 일이 충분히 일어난다. 같은 키의 같은 컬럼에 같은 timestamp로 두 값이 들어오면, Cassandra는 시각으로 우열을 못 가린다. 이때의 tie-break 규칙은:

> **timestamp가 같으면, cell 값(value)을 바이트로(unsigned) 비교해 사전순으로 큰 쪽이 이긴다.** (deterministic하지만 의미론적으로는 무의미한 기준)

즉 "나중에 보낸 쓰기"가 이긴다는 보장이 전혀 없다. 값의 바이트가 더 큰 쪽이 이긴다. 두 쓰기가 의미상 동등하지 않은데 timestamp가 같으면, **하나는 조용히 사라지고 어느 쪽이 살아남을지는 비즈니스 관점에서 임의적**이다. 충돌이 났다는 경고조차 없다. RDB의 `UPDATE`라면 락으로 직렬화돼 둘 다 순서대로 반영되지만 Cassandra는 둘 중 하나를 소리 없이 버린다.

한 가지 규칙만은 예외적으로 결정적이다. **같은 timestamp에서는 삭제(tombstone)가 일반 쓰기를 항상 이긴다.** 같은 마이크로초에 "값 쓰기"와 `DELETE`가 부딪히면, 값이 무엇이든 데이터가 사라지는 쪽으로 수렴한다. "재생성(upsert)"과 "삭제"가 같은 시각에 경합하는 모델에서 삭제가 슬그머니 이겨버리는 또 하나의 함정이다.

이 함정이 특히 위험한 경우들:

- **클라이언트가 timestamp를 직접 지정(`USING TIMESTAMP`)**하면서 같은 값을 재사용할 때. "멱등성을 위해 timestamp를 고정한다"는 패턴이 잘못 적용되면, 의도와 다른 쓰기가 무시될 수 있다.
- **서로 다른 노드의 시계가 어긋났을 때**. coordinator가 timestamp를 부여하는데 노드 간 시계 동기(NTP)가 깨지면, "물리적으로 나중인" 쓰기가 더 작은 timestamp를 받아 **과거 쓰기에게 져버린다.** 새 쓰기가 무시되는 역전이 생긴다. Cassandra 클러스터에서 **NTP 시계 동기화가 정합성의 전제 조건**인 이유다.

> **결제 엔지니어를 위한 함의**: "마지막 상태 업데이트가 적용됐을 것"이라는 가정을 LWW 위에서 하면 안 된다. 상태 전이가 중요한 도메인(예: `READY → PAID → CANCELLED`)에서는 같은 cell을 덮어쓰는 모델 대신, **append-only 이벤트로 쌓고 읽을 때 재구성**하는 모델([[13 - 실전 Event Sourcing on Cassandra]])이나, 진짜 선형성이 필요하면 **LWT(경량 트랜잭션, [[11 - LWT Batch Counter 내부]])**를 써야 한다. LWT는 Paxos로 직렬화해 "조건부 쓰기"를 보장하지만 훨씬 비싸다. LWW의 침묵하는 손실을 의식하고 모델을 설계하라.

---

## tombstone이 읽기를 망치는 방식

이제 읽기 path를 가장 잔인하게 망가뜨리는 주제, **tombstone**이다. 앞에서 "`DELETE`는 데이터를 지우지 않고 tombstone을 append한다"고 했다. 왜 즉시 안 지우는가? 분산 환경에서 "삭제"를 안전하게 전파하려면 삭제 사실 자체를 **데이터처럼** 복제·병합해야 하기 때문이다. 어떤 replica가 다운된 사이 삭제가 일어났는데 즉시 물리 삭제해버리면, 그 replica가 복귀해 옛 데이터를 갖고 있을 때 read repair가 "옛 데이터가 더 최신"이라 착각해 **삭제한 데이터가 부활(zombie)**할 수 있다. tombstone은 "이건 t 시점에 삭제됐다"는 사실을 명시적으로 남겨, 그 부활을 막는다.

문제는 읽기다. tombstone은 `gc_grace_seconds`(기본 **864000초 = 10일**) 동안, 그리고 compaction이 실제로 정리할 때까지 SSTable에 남는다. 그동안 그 키를 읽으면 Cassandra는 **tombstone들도 다 읽어서 병합 과정에 포함**해야 한다. "이 cell은 삭제됐는지 확인"하려면 tombstone을 봐야 하기 때문이다.

### 진짜 재앙: range scan 위의 tombstone 누적

단일 cell 삭제 하나는 별것 아니다. 재앙은 **한 partition 안에서 여러 row를 읽는 스캔**(clustering 범위 쿼리)에서 일어난다. 그 범위에 tombstone이 잔뜩 끼어 있으면, Cassandra는 클라이언트에게 돌려줄 **살아있는 row 1개를 찾으려고 수천 개의 tombstone을 읽고 버리는** 일을 한다. 읽기 응답에는 1건이 나오는데 내부적으로는 수만 개를 스캔한 셈이다.

Cassandra는 이걸 감지하려고 두 개의 임계값을 둔다(둘 다 정확한 5.0 기본값이다).

| 설정 (`cassandra.yaml`) | 기본값 | 동작 |
|---|---|---|
| `tombstone_warn_threshold` | **1000** | 한 쿼리가 이 수만큼 tombstone을 읽으면 로그에 **경고** |
| `tombstone_failure_threshold` | **100000** | 이 수를 넘으면 쿼리를 **중단하고 `TombstoneOverwhelmingException`** |

tombstone이 10만 개를 넘으면 그 쿼리는 **아예 실패한다.** 데이터가 멀쩡히 있어도 읽을 수가 없게 된다. 운영 로그에서 `TombstoneOverwhelmingException`을 보면 이 절을 떠올려야 한다.

```text
   SELECT * FROM queue WHERE topic = 'jobs' LIMIT 1;   -- 살아있는 job 1개만 원함

   partition "jobs" 의 clustering 순서:
   [✝][✝][✝][✝][✝][✝]...(처리되어 DELETE된 수만 개)...[✝][✝][LIVE]
    └───────────────────── 다 읽고 버림 ─────────────────────┘  └─ 겨우 이 1건

   tombstone 1,000개 → WARN 로그
   tombstone 100,000개 → TombstoneOverwhelmingException, 쿼리 실패
```

### 큐 안티패턴 — Cassandra로 큐를 만들면 안 되는 이유

이제 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]의 그 악명 높은 안티패턴이 왜 그렇게 위험한지 tombstone 관점에서 정확히 보인다.

큐(queue)의 본질은 "**한쪽 끝에서 넣고 다른 쪽 끝에서 빼는**" 것이다. Cassandra로 큐를 만들면:

1. 작업을 한 partition에 clustering으로 계속 `INSERT` (한쪽에 쌓임).
2. 처리한 작업을 `DELETE` → **partition 앞쪽에 tombstone이 끝없이 누적**.
3. "다음 처리할 작업"을 읽으려고 `SELECT ... LIMIT 1`을 하면, **앞쪽에 쌓인 수만 개의 tombstone을 전부 건너뛴 뒤에야** 살아있는 작업에 도달.

처리량이 늘수록 tombstone이 빠르게 쌓이고 어느 순간 `tombstone_failure_threshold`를 넘겨 **큐를 읽는 쿼리 자체가 실패**한다. 데이터를 지웠는데 오히려 읽기가 점점 느려지다가 죽는, 직관에 정면으로 반하는 현상이다. 게다가 tombstone은 `gc_grace_seconds`(10일) 동안 못 지우니 트래픽이 많으면 compaction이 따라잡지도 못한다.

> **규칙**: "삭제가 잦은 좁은 범위를 반복해서 range scan"하는 모든 패턴은 Cassandra와 상극이다. 큐, 최신 N개를 계속 갈아치우는 버퍼, TTL로 대량 만료되는 시계열의 앞부분 스캔 등이 전부 여기 해당한다. 큐가 필요하면 Kafka/SQS 같은 전용 시스템을 쓰고 Cassandra는 "삭제보다 추가가 압도적으로 많은" 워크로드에 두어라. 굳이 만료가 필요하면 `DELETE` 대신 **TTL + 적절한 compaction 전략(TWCS 등, [[09 - Compaction 전략]])**으로 partition 단위로 통째로 떨궈서 cell-level tombstone 스캔을 피한다.

tombstone을 줄이는 실전 레버는 [[12 - 운영과 트러블슈팅]]에서 더 다루지만, 요점만 짚으면 `gc_grace_seconds`를 무작정 줄이면 zombie 위험이 커지므로 repair 주기와 함께 봐야 하고, 데이터 모델 단계에서 "삭제 패턴"을 아예 안 만드는 게 최선이다.

---

## 두 path를 나란히 — 최종 비교

마지막으로 이 장 전체를 두 다이어그램과 한 표로 압축한다.

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

이 비대칭이 Cassandra 데이터 모델링의 제1원칙([[05 - 데이터 모델링 1 - Query First]])과 직결된다. 읽기가 비싸기 때문에 **"읽을 쿼리를 먼저 정하고 거기에 맞춰 테이블(=partition)을 설계"**한다. 쓰기는 중복을 마다하지 않고 여러 테이블에 같은 데이터를 박아 넣어도 된다 — 쓰기는 싸니까. 이 장에서 본 내부 메커니즘이 그 모델링 철학의 물리적 근거다.

---

## 핵심 요약

- **쓰기가 빠른 이유**: replica를 계산으로 찾고(라우팅 비용 0에 가까움), 각 노드는 commit log **순차 append** + memtable 메모리 쓰기뿐이며, **read-before-write가 없고**, fsync를 묶고(periodic, 기본 10초), 무거운 정리는 compaction으로 미룬다.
- **Write path 흐름**: client → coordinator(partitioner로 replica 산출, snitch로 정렬) → 각 replica에서 commit log + memtable → **CL만큼 ack** 모이면 클라이언트에 성공. 다운된 replica는 **hinted handoff**(기본 `max_hint_window=3h`)로 복귀 시 보충, 초과 시 `repair` 필수.
- **읽기가 비싼 이유**: 같은 키 조각이 memtable + 여러 SSTable에 흩어져 있어, **cell 단위 timestamp 병합(last-write-wins)**으로 매번 재조립해야 한다.
- **Read path 흐름**: coordinator → 최근접 replica에 **full read**, 나머지에 **digest read** → 노드 안에서 최적화 계층 통과 → 병합 → **digest 비교** → 불일치면 **read repair**.
- **단일 노드 읽기 최적화 계층(순서)**: row cache(기본 off) → **Bloom filter**(false negative 불가, false positive만 가능; "없음"은 100% 신뢰) → **key cache**(기본 on, `min(5% heap, 100MB)`) → **partition summary**(메모리 샘플) → **partition index**(디스크) → **Data.db** → chunk cache → OS page cache.
- **`bloom_filter_fp_chance` 5.0 기본값**: STCS/UCS = **0.01**, LCS = **0.1**. 낮출수록 헛읽기↓·메모리↑.
- **last-write-wins 함정**: timestamp가 같으면 **cell 값의 바이트 사전순**으로 승자가 결정된다(의미 없는 기준). NTP 시계 어긋남은 "나중 쓰기가 과거에게 지는" 역전을 만든다. 상태 전이엔 event sourcing 또는 LWT를 고려.
- **tombstone이 읽기를 망친다**: `DELETE`는 tombstone을 append하고 `gc_grace_seconds`(기본 **10일**) 동안 남는다. 스캔 중 누적되면 `tombstone_warn_threshold`(**1000**)에서 경고, `tombstone_failure_threshold`(**100000**)에서 `TombstoneOverwhelmingException`으로 **쿼리 실패**. **큐 안티패턴**의 근본 원인.
- **speculative retry**(기본 `99p`): 느린 replica를 추월해 p99를 깎되, 읽기 부하 여유가 있을 때만 공격적으로.

## 연결 노트

- [[08 - Storage Engine 내부]] — commit log·memtable·SSTable·Bloom filter·인덱스 파일의 구조. 이 장의 모든 부품의 출처.
- [[09 - Compaction 전략]] — "봐야 할 SSTable 수"와 tombstone 정리를 결정. 읽기 성능의 숨은 핸들.
- [[04 - Tunable Consistency]] — CL이 full/digest 분업과 `R+W>RF`로 정합성을 만드는 방식.
- [[03 - 복제 전략과 데이터센터]] — partitioner·snitch가 replica를 산출·정렬하는 토대.
- [[02 - 분산 아키텍처]] — coordinator·토큰 링·gossip이 라우팅을 가능케 하는 배경.
- [[05 - 데이터 모델링 1 - Query First]] / [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]] — 읽기 비용과 tombstone이 모델링 규칙(큐 안티패턴 포함)을 결정하는 이유.
- [[11 - LWT Batch Counter 내부]] — LWW의 한계를 넘어서는 조건부 쓰기(Paxos).
- [[12 - 운영과 트러블슈팅]] — tombstone·hint·캐시·repair의 실전 진단과 튜닝.
- [[13 - 실전 Event Sourcing on Cassandra]] — append-only 모델로 LWW·tombstone 함정을 피하는 설계.
