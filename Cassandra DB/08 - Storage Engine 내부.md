---
title: Storage Engine 내부 - LSM-Tree, Commit Log, Memtable, SSTable
date: 2026-06-26
tags: [cassandra, storage-engine, lsm-tree, sstable, 학습노트]
---

이 장은 다음 질문들에 답한다.

- 왜 Cassandra는 RDB가 평생 써온 B-Tree를 버리고 LSM-Tree를 택했는가? 무엇을 얻고 무엇을 포기했는가?
- 쓰기 한 번이 들어오면 디스크에서 실제로 무슨 일이 일어나는가? 왜 그렇게까지 빠른가?
- Commit Log는 정확히 무엇을 보장하며, `commitlog_sync`의 세 가지 모드는 어떤 트레이드오프를 다루는가?
- Memtable은 어떤 자료구조이고, 언제·왜 flush되는가?
- SSTable은 왜 immutable이며, 그 immutability가 만들어내는 연쇄 효과(읽기 병합, compaction, "삭제했는데 디스크가 안 줄어드는" 현상)는 무엇인가?
- Cassandra 5.0이 새로 들고 온 BTI(Big Trie-Indexed) SSTable 포맷은 기존 포맷의 어떤 한계를 푸는가?

읽기 경로의 세부(Bloom Filter 적중률, key/row/chunk cache, read repair 타이밍 등)는 [[10 - Write Path와 Read Path]]에서 다시 깊게 다룬다. 이 장은 "디스크에 데이터가 어떻게 놓이는가"에 집중한다.

---

## 왜 B-Tree가 아니라 LSM-Tree인가

RDB 경험이 있는 엔지니어라면 머릿속에 이미 강력한 디스크 모델이 하나 박혀 있다. 바로 **B-Tree(정확히는 B+Tree)** 다. PostgreSQL, MySQL의 InnoDB, Oracle — 거의 모든 전통 RDB의 기본 인덱스이자 테이블 저장 구조다. B-Tree는 **읽기에 최적화된 자료구조**다. 트리 높이가 보통 3~4단계이므로 어떤 키든 디스크 페이지 3~4번만 읽으면 도달한다. 정렬된 상태로 페이지에 in-place로 데이터를 유지하기 때문에 범위 스캔도 깔끔하다.

문제는 **쓰기**다. B-Tree에 행 하나를 추가하거나 수정하면, 그 키가 속하는 리프 페이지(leaf page)를 찾아가서 그 페이지를 **제자리에서 고쳐 쓴다(update-in-place)**. 키가 디스크 전체에 흩어져 있으므로 이 페이지의 물리적 위치는 사실상 랜덤이다. 즉 **랜덤 쓰기(random write)** 가 발생한다. 페이지가 가득 차면 split이 일어나 또 다른 랜덤 위치를 건드린다.

HDD 시절 랜덤 쓰기가 느린 이유는 직관적이다. 디스크 헤드를 물리적으로 옮기는 seek 시간(수 ms)이 지배적이기 때문이다. SSD 시대에는 seek이 사라졌지만 문제가 완전히 없어진 건 아니다. SSD는 페이지 단위로 읽고 블록 단위로 지운다. in-place update는 read-modify-write와 write amplification(쓰기 증폭), 그리고 가비지 컬렉션 부담을 낳는다. 분산 시스템에서 노드 하나가 초당 수만 건의 쓰기를 받아내야 한다면, 매 쓰기마다 트리의 리프 페이지를 찾아 락을 잡고 고쳐 쓰는 모델은 동시성 측면에서도 병목이 된다.

Cassandra는 처음부터 **"쓰기가 미친 듯이 많이 들어오는 워크로드"** 를 가정하고 설계됐다(원류는 Google Bigtable + Amazon Dynamo). 그래서 B-Tree가 아니라 **LSM-Tree(Log-Structured Merge-Tree)** 를 택했다. LSM-Tree의 발상은 한 문장으로 요약된다.

> **랜덤 쓰기를 순차 쓰기로 바꾼다(turn random writes into sequential writes).**

방법은 단순하다. 디스크의 "정해진 자리"에 고쳐 쓰는 대신, **들어오는 순서대로 append만 한다.** 데이터를 메모리에 정렬된 채로 모아두었다가, 충분히 쌓이면 한꺼번에 디스크에 순차로 쏟아낸다(sequential write). 이미 디스크에 쓴 파일은 절대 고치지 않는다(immutable). 같은 키의 수정·삭제도 그냥 새 버전을 append할 뿐이다.

### B-Tree vs LSM-Tree 한눈에

| 측면 | B-Tree (RDB) | LSM-Tree (Cassandra) |
|------|--------------|----------------------|
| 쓰기 방식 | update-in-place (제자리 수정) | append-only (순차 추가) |
| 디스크 I/O 패턴(쓰기) | 랜덤 쓰기 | 순차 쓰기 |
| 쓰기 지연 | 페이지 찾기 + 락 + 수정 | 메모리 append (디스크 seek 없음) |
| 읽기 | 트리 3~4단계, 한 곳에 최신값 | 여러 SSTable 병합 필요 |
| 같은 키의 위치 | 항상 한 곳 | 여러 파일에 흩어짐 |
| 정리 작업 | split/merge(즉시) | compaction(비동기 백그라운드) |
| 최적화 대상 | 읽기 | 쓰기 |
| 삭제 | 즉시 자리 회수 | tombstone 기록 후 나중에 회수 |

### 공짜는 없다: 세 가지 증폭(amplification)

LSM-Tree가 쓰기를 싸게 만든 대가는 "증폭"이라는 세 단어로 정리된다. 이 세 가지는 LSM 계열 엔진을 이해하고 튜닝하는 기본 어휘이므로 정확히 구분해 두는 게 좋다.

- **쓰기 증폭(write amplification)**: 논리적으로 한 번 쓴 데이터가 디스크에는 여러 번 쓰인다. 처음 Commit Log + Memtable flush로 한 번, 그리고 compaction이 같은 데이터를 다시 읽어 합쳐 다시 쓰면서 여러 번 더. "순차 쓰기로 바꿨다"는 건 각 쓰기를 싸게 만든 것이지, 총 쓰기량을 줄인 게 아니다. compaction 전략에 따라 이 값이 크게 달라진다([[09 - Compaction 전략]]).
- **읽기 증폭(read amplification)**: 하나의 행을 읽으려면 그 키가 흩어져 있는 여러 SSTable을 다 뒤져야 할 수 있다. 최악의 경우 SSTable 개수만큼 디스크 접근이 늘어난다. 이걸 줄이려고 Bloom Filter, partition index, key cache가 동원된다.
- **공간 증폭(space amplification)**: 같은 키의 옛 버전, 삭제 표식(tombstone), 중복 데이터가 compaction 전까지 디스크에 함께 남아 있다. 그래서 "논리적 데이터 크기 < 실제 디스크 사용량"이 된다.

이 세 가지는 **동시에 다 줄일 수 없다.** compaction을 공격적으로 돌리면 읽기·공간 증폭은 줄지만 쓰기 증폭이 커지고 느슨하게 돌리면 그 반대다. 이 삼각관계를 어떻게 타협할지 고르는 것이 바로 compaction 전략 선택의 본질이다.

> **흔한 오해**: "LSM-Tree는 쓰기가 빠르고 B-Tree는 읽기가 빠르다"는 절반만 맞다. LSM-Tree도 Bloom Filter·인덱스·캐시·compaction이 잘 굴러가면 읽기가 충분히 빠르다. 정확히 말하면 **LSM-Tree는 쓰기를 "확정적으로" 싸게 만들고, 읽기 비용을 백그라운드 작업(compaction)으로 "이연(defer)"한다.** 비용을 없앤 게 아니라 시간축에서 옮긴 것이다.

---

## 쓰기 경로 미리보기: append 한 번이면 끝

세부는 [[10 - Write Path와 Read Path]]에 있지만, 스토리지 엔진을 이해하려면 쓰기가 디스크에 닿는 그 순간을 먼저 봐야 한다. 코디네이터가 책임 노드를 골라 RPC를 보낸 뒤, **각 replica 노드 안에서** 일어나는 일은 놀랄 만큼 짧다.

```text
   클라이언트 INSERT/UPDATE/DELETE (CQL은 전부 "쓰기"로 동일하게 취급)
                       │
                       ▼
        ┌──────────────────────────────────┐
        │  replica 노드 내부 (로컬 적용)     │
        │                                  │
        │   1) Commit Log에 append  ───────┼──▶ 디스크 (순차, durability)
        │            (mutation 직렬화)      │
        │                                  │
        │   2) Memtable에 적용  ────────────┼──▶ 메모리 (정렬된 구조에 삽입)
        │                                  │
        │   3) 클라이언트에 ack ◀───────────┤  (1,2 끝나면 성공)
        └──────────────────────────────────┘
```

여기서 짚을 점이 두 가지다.

**첫째, 어느 단계에도 "디스크에서 무언가를 읽는" 과정이 없다.** B-Tree였다면 "이 키가 어느 리프 페이지에 있지? 그 페이지를 읽어와서 고쳐 쓰자"라는 read-before-write가 필요했다. Cassandra의 일반 쓰기 경로는 디스크를 읽지 않는다. 그냥 Commit Log 끝에 붙이고 메모리에 꽂는다. 그래서 디스크 seek이 0이고, 쓰기 지연이 메모리 연산 수준으로 낮다. (예외: LWT(`IF NOT EXISTS` 등)와 counter는 read-before-write가 필요하다. [[11 - LWT Batch Counter 내부]]에서 다룬다.)

**둘째, UPDATE도 DELETE도 "쓰기"다.** Cassandra에는 in-place update라는 개념이 없다. `UPDATE`는 새 값을 append하는 것이고, `DELETE`는 tombstone(삭제 표식)을 append하는 것이다. 기존 데이터를 찾아가서 고치거나 지우지 않는다. 이것이 LSM의 append-only 철학이 CQL 레벨까지 관통하는 지점이다.

> RDB였다면 `UPDATE accounts SET balance=balance-1000 WHERE id=42`는 id=42 행을 찾아 그 자리에서 balance를 고친다. Cassandra에서 같은 문장은 "id=42, balance 컬럼에 새 값, timestamp T"라는 셀(cell)을 새로 append할 뿐이다. 예전 balance 값은 디스크 어딘가에 그대로 남아 있다가, 읽을 때 timestamp 비교로 최신값이 이긴다(last-write-wins). 정산·결제 도메인에서 이 모델은 직관과 어긋날 수 있어 특히 주의해야 한다.

---

## Commit Log: durability의 마지막 보루

Memtable은 메모리에 있다. 노드가 죽거나 전원이 나가면 **flush되지 않은 Memtable의 데이터는 통째로 증발한다.** 그런데도 우리가 "쓰기 성공" ack를 클라이언트에 돌려줄 수 있는 이유는 단 하나, **그전에 Commit Log에 먼저 적었기 때문**이다. Commit Log는 노드 로컬의 **append-only WAL(Write-Ahead Log)** 이다. 역할은 명확하다.

> 노드가 비정상 종료된 뒤 재시작하면, Cassandra는 Commit Log를 처음부터 재생(replay)해서 아직 SSTable로 flush되지 못했던 Memtable 상태를 메모리에 복원한다. 그래서 "메모리에만 있던 데이터가 사라지는" 사고를 막는다.

핵심은 **순차 append**라는 점이다. Commit Log는 항상 파일 끝에만 쓴다. 디스크 헤드를 옮길 일도, 페이지를 찾을 일도 없다. 그래서 durability를 보장하면서도 빠르다. RDB의 WAL/redo log와 개념적으로 똑같다.

> **예외 — `durable_writes`**: Commit Log 기록은 keyspace 단위로 끌 수 있다(`durable_writes`, 기본 `true`). `false`로 두면 쓰기가 Commit Log를 건너뛰고 Memtable에만 적용돼 더 빠르지만, flush 전에 노드가 죽으면 그 데이터는 **복구되지 않는다**. 언제든 재생성 가능한 파생/캐시성 데이터가 아니라면 끄지 말 것. 결제·정산 같은 원장성 데이터에서는 논외다.

### segment와 재활용

Commit Log는 하나의 거대한 파일이 아니라 **세그먼트(segment)** 들로 쪼개져 있다. 세그먼트 하나의 기본 크기는 32MiB(`commitlog_segment_size`)다. 한 세그먼트가 가득 차면 새 세그먼트를 만들어 거기에 계속 append한다. (참고로 단일 mutation의 최대 크기 `max_mutation_size`는 기본적으로 세그먼트 크기의 절반인 16MiB로, 한 쓰기가 세그먼트 경계를 넘지 못하게 한다.)

여기서 우아한 부분이 있다. 어떤 Memtable이 flush되어 그 안의 모든 데이터가 안전하게 SSTable로 디스크에 내려가면, **그 데이터를 담고 있던 Commit Log 세그먼트는 더 이상 필요 없다.** 재시작 시 그 데이터는 SSTable에서 복구되기 때문이다. Cassandra는 각 세그먼트가 어떤 테이블의 어느 시점까지를 담는지 추적하다가, 한 세그먼트가 커버하던 모든 Memtable이 flush 완료되면 그 세그먼트를 **재활용(recycle)하거나 삭제**한다. 그래서 Commit Log 디렉터리는 무한히 커지지 않고, 대략 "아직 flush 안 된 Memtable들이 의존하는 만큼"의 크기로 유지된다.

`commitlog_total_space`(기본값은 보통 8192MiB 또는 디스크의 1/4 중 작은 값)에 도달하면 Cassandra는 **가장 오래된 세그먼트를 비우기 위해 그 세그먼트가 의존하는 Memtable들의 flush를 강제로 트리거**한다. Commit Log 한도 초과 역시 Memtable flush의 트리거 중 하나다(아래 Memtable 절에서 다시 등장).

### commitlog_sync: durability와 지연의 흥정

가장 중요한 튜닝 포인트다. "Commit Log에 append했다"고 곧바로 데이터가 물리 디스크 플래터/플래시에 안착하는 게 아니다. OS는 보통 페이지 캐시에 모아두었다가 나중에 디스크로 내린다. 진짜로 디스크에 안착시키려면 **`fsync`** 시스템 콜로 강제 flush해야 한다. 문제는 fsync가 비싸다는 것. 그래서 Cassandra는 "언제 fsync할 것인가"를 `commitlog_sync`로 고르게 한다. 모드는 세 가지다.

| 모드 | 동작 | 클라이언트 ack 시점 | durability | 지연/처리량 |
|------|------|---------------------|------------|-------------|
| **periodic** (기본) | 주기적으로 fsync (`commitlog_sync_period`, 기본 10000ms = 10초) | fsync 기다리지 않고 즉시 ack | 최대 `period`만큼의 쓰기가 유실 위험 | 가장 빠름 |
| **batch** | 쓰기를 모아 fsync, fsync 완료까지 ack 대기 | fsync 완료 후 ack | 유실 거의 없음 | 가장 느림(fsync 동기 대기) |
| **group** | 일정 시간 윈도(`commitlog_sync_group_window`, 기본 1000ms) 동안 쓰기를 모아 한 번에 fsync | 그룹 fsync 완료 후 ack | batch와 periodic 사이 | 절충 |

각 모드의 의미를 풀어보자.

**periodic(기본값)** 은 처리량 최우선이다. 클라이언트가 쓰기를 보내면 Commit Log 버퍼에 append하고 **fsync를 기다리지 않고 바로 ack**한다. fsync는 별도로 10초마다 일어난다. 이 말은, 노드가 fsync 직전에 갑자기 죽으면 **마지막 fsync 이후 ~10초간 ack했던 쓰기들이 디스크에 없을 수 있다**는 뜻이다.

> **여기서 결제/정산 엔지니어가 반드시 짚어야 할 점**: periodic 모드에서 "성공 ack를 받았다"는 것이 "단일 노드 디스크에 영속화됐다"를 의미하지 않는다. 그 안전은 **복제(replication)와 consistency level**이 보장한다. RF=3에 `QUORUM`으로 쓰면 데이터가 2개 노드의 메모리+Commit Log에 들어간 뒤 ack된다. 한 노드가 fsync 전에 죽어도 다른 노드들이 데이터를 갖고 있고 hinted handoff/read repair로 복구된다. Cassandra의 durability 모델은 **"단일 노드 fsync"가 아니라 "여러 노드에 분산 저장"에 의존한다.** 이 철학은 [[04 - Tunable Consistency]]와 직결된다. 단일 노드 수준의 강한 durability가 꼭 필요하면 `batch`를 쓰되, 쓰기 지연이 fsync 지연에 묶인다는 비용을 받아들여야 한다.

**batch** 는 안전 최우선이다. 쓰기를 받으면 fsync가 끝날 때까지 ack를 보류한다. 그래서 "ack = 디스크 안착"이 성립한다. 대신 모든 쓰기가 디스크 fsync 지연(특히 회전 디스크라면 치명적, NVMe라도 비용)에 직렬로 묶인다. 그래서 batch는 보통 **빠른 NVMe SSD + 전용 Commit Log 디스크** 환경에서만 현실적이다.

**group** 은 batch의 "fsync 완료 후 ack" 안전성을 유지하면서, 매 쓰기마다 fsync하는 게 아니라 1초 윈도 동안 들어온 쓰기를 모아 한 번에 fsync한다. fsync 횟수를 줄여 처리량을 회복하는 절충안이다. batch가 처리량 때문에 부담스럽고 periodic의 유실 윈도가 불안할 때의 중간 선택지다.

**Commit Log 디스크를 데이터 디스크와 물리적으로 분리**하는 것은 전통적인 권장 사항이다. 순차 쓰기인 Commit Log와 랜덤성이 섞인 SSTable I/O가 같은 디스크에서 헤드/대역폭을 다투지 않게 하기 위함이다(특히 HDD에서 효과가 컸고, 단일 NVMe 환경에서는 이득이 줄어든다. 환경에 따라 다르다는 점은 짚어둔다).

---

## Memtable: 정렬된 메모리 버퍼

Commit Log가 durability를 책임진다면, **Memtable은 속도와 정렬을 책임진다.** Memtable은 테이블별로 존재하는 **메모리 내 쓰기 버퍼**이며 들어온 mutation을 **파티션 키로, 그리고 파티션 내부에서는 clustering 순서로 정렬된 상태**로 보관한다. 자료구조는 정렬된 맵(전통적으로 `ConcurrentSkipListMap` 계열, 5.0에서는 trie 기반 메모리 인덱스 등 구현이 개선됨)이다. 관건은 **항상 정렬을 유지한다**는 성질이다.

정렬을 유지하는 이유는 **flush할 때 정렬을 다시 할 필요가 없게 하기 위해서**다. Memtable이 이미 디스크 저장 순서(파티션 키 토큰 순 + clustering 순)대로 정렬돼 있으니, flush는 그저 이 정렬된 메모리를 **순차로 디스크에 쏟아붓기만** 하면 된다. 그 결과물이 정렬된 immutable 파일, 즉 SSTable이다. "Sorted String Table"의 Sorted가 바로 여기서 온다.

같은 키에 여러 번 쓰기가 들어오면 어떻게 될까? RDB라면 같은 행을 계속 덮어쓰겠지만, Memtable에서는 **같은 셀의 갱신이 메모리 안에서 병합된다.** 동일 파티션·동일 clustering·동일 컬럼의 더 새로운 쓰기가 메모리상에서 이전 값을 이긴다. 그래서 같은 키를 1초에 100번 갱신해도 그 Memtable이 flush될 때는 (대개) 최신 한 버전만 SSTable로 나간다. 이것이 LSM이 hot key 갱신을 효율적으로 흡수하는 방식이다.

### on-heap vs off-heap

Memtable은 JVM 위에서 동작하므로 **GC**가 항상 고민거리다. 수 GB의 데이터를 JVM heap에 들고 있으면 GC가 그것을 매번 스캔하며 stop-the-world를 길게 만든다. 그래서 Cassandra는 Memtable의 일부를 **off-heap(네이티브 메모리)** 에 둘 수 있게 한다. `memtable_allocation_type`으로 제어한다.

- `heap_buffers`: 전부 JVM heap. 단순하지만 GC 압박.
- `offheap_buffers`: 셀 값(데이터) 버퍼를 off-heap에. 인덱스 구조는 heap.
- `offheap_objects`: 더 많은 부분을 off-heap에.

off-heap의 목적은 **GC가 스캔해야 할 heap 객체 수와 크기를 줄여 GC 일시정지를 짧고 예측 가능하게** 만드는 데 있다. 큰 Memtable을 운용하는 쓰기 헤비 클러스터에서 의미가 크다.

### flush: Memtable이 SSTable이 되는 순간

Memtable은 무한히 커질 수 없다(메모리는 유한하다). 어느 시점에 메모리 내용을 디스크로 내보내고 새 Memtable을 비워야 한다. 이 과정이 **flush**이고, **flush의 결과물이 바로 하나의 새 SSTable**이다. flush를 촉발하는 트리거는 여러 개이며 그중 무엇이든 먼저 닿으면 flush가 일어난다.

1. **`memtable_cleanup_threshold` (메모리 압력)**: 모든 테이블의 Memtable이 쓰는 메모리 풀(`memtable_heap_space`/`memtable_offheap_space`로 결정)이 일정 비율을 넘으면, Cassandra는 **현재 가장 큰 Memtable**을 골라 flush해서 공간을 회수한다. 이 임계값의 기본 동작은 `1 / (memtable_flush_writers + 1)` 로 계산된다(기본 `memtable_flush_writers`가 2이면 1/(2+1) ≈ 0.33, 즉 풀의 약 33%에서 트리거). 풀이 차오를수록 큰 Memtable부터 비워 메모리를 되찾는다. 참고로 4.0 이후 `memtable_cleanup_threshold`를 직접 지정하는 것은 **deprecated**되었고, 이 자동 계산식을 그대로 두는 것이 권장된다.
2. **Commit Log 공간 한도**: 위 Commit Log 절에서 본 것처럼, `commitlog_total_space`에 도달하면 가장 오래된 세그먼트를 비우기 위해 그 세그먼트에 의존하는 Memtable들의 flush가 강제된다. 즉 **Commit Log가 가득 차서 일어나는 flush**다.
3. **시간 기반**: `memtable_flush_period_in_ms`(테이블별 설정, 기본 0=비활성)가 설정돼 있으면 그 주기마다 강제 flush.
4. **수동/운영 트리거**: `nodetool flush`, `nodetool drain`(셧다운 전 전체 flush), 스키마 변경, 스냅샷 생성 등.

flush가 일어나면 다음 순서로 진행된다.

```text
  [Memtable 가득 참 / 메모리 압력 / commitlog 한도]
                 │
                 ▼
  ① 현재 Memtable을 "flush 대상"으로 전환(immutable로 freeze)
     동시에 새 빈 Memtable을 만들어 신규 쓰기를 거기로 받음 (쓰기 중단 없음)
                 │
                 ▼
  ② freeze된 Memtable을 정렬 순서대로 순차 write
     → 새 SSTable 컴포넌트들 생성 (Data.db, Index, Filter.db ...)
                 │
                 ▼
  ③ flush 완료 후, 이 데이터를 담던 Commit Log 세그먼트를 회수 가능 표시
                 │
                 ▼
  ④ 새 SSTable이 읽기 경로에 등록됨 (이제 쿼리가 이 파일도 본다)
```

여기서도 매끄러운 부분이 있다. **flush 중에도 쓰기는 멈추지 않는다.** freeze된 옛 Memtable이 디스크로 내려가는 동안, 신규 쓰기는 새로 만든 빈 Memtable로 들어간다. 그래서 flush가 쓰기 가용성을 막지 않는다.

> **flush가 잦으면 작은 SSTable이 우수수 쏟아진다.** Memtable이 작을 때(메모리 압력이 높거나 Commit Log가 자주 차면) flush가 자주 일어나고, 그만큼 작은 SSTable이 많아진다. SSTable 개수가 많아지면 읽기 증폭이 커지고, compaction이 따라잡지 못해 "pending compactions"가 쌓인다. 그래서 Memtable 크기·Commit Log 한도·힙 크기는 따로따로가 아니라 **하나의 균형**으로 봐야 한다. 운영 징후와 대응은 [[12 - 운영과 트러블슈팅]]에서 다룬다.

---

## SSTable: 변하지 않는 디스크 파일

flush의 결과물이자 Cassandra 디스크 데이터의 기본 단위가 **SSTable(Sorted Strings Table)** 이다. SSTable의 가장 중요한 성질은 단 하나, **immutable(불변)** 이다. 한 번 디스크에 쓰이면 그 파일은 절대 수정되지 않는다. 행을 고치지도, 지우지도, 끼워넣지도 않는다. 오직 두 가지 운명만 있다. **읽히거나, compaction으로 다른 SSTable들과 합쳐져 새 SSTable이 되고 자신은 삭제되거나.**

immutable이 주는 이점은 크다.

- **동시성이 단순**해진다. 읽는 쪽은 락 없이 SSTable을 읽을 수 있다. 누구도 그 파일을 고치지 않기 때문이다.
- **캐시·압축·인덱스가 안정**적이다. 파일이 변하지 않으니 한 번 만든 Bloom Filter, partition index, 압축 청크가 그대로 유효하다.
- **백업/스냅샷이 값싸다.** SSTable이 안 변하므로 스냅샷은 그냥 파일에 대한 하드링크(hard link)다. 데이터를 복사하지 않는다.
- **쓰기가 순차**다. 위에서 본 모든 이점의 뿌리.

하지만 immutable이 곧 모든 복잡함의 근원이기도 하다(다음 절에서 본격적으로 다룬다).

### SSTable의 구성 파일들

하나의 "SSTable"은 사실 디스크에서 **여러 개의 파일(컴포넌트)** 로 이루어진다. 같은 generation 식별자(5.0에서는 `uuid_sstable_identifiers_enabled`로 ULID 기반 식별자를 켤 수도 있다)를 공유하는 파일 묶음이 논리적으로 하나의 SSTable이다. BIG 포맷 파일명은 `<포맷버전>-<generation>-big-<Component>.db` 형태다(포맷 버전 문자는 릴리스마다 올라간다. 예컨대 `nb-1-big-Data.db`. 단정하지 말고 디스크에서 확인할 것). 각 컴포넌트와 역할은 다음과 같다.

| 컴포넌트                  | 파일                   | 역할                                                                                                             | 위치  |
| --------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------- | --- |
| **Data**              | `Data.db`            | 실제 행 데이터. 파티션 키 토큰 순 + clustering 순으로 정렬되어 저장. 유일하게 "진짜 데이터"가 있는 곳                                             | 디스크 |
| **Partition Index**   | `Index.db`           | 파티션 키 → Data.db 내 바이트 오프셋 매핑. 키로 데이터 위치를 찾는 인덱스                                                                | 디스크 |
| **Partition Summary** | `Summary.db`         | Partition Index를 일정 간격으로 샘플링한 요약. 인덱스의 "인덱스". 메모리(off-heap)에 상주해 Index.db 탐색 시작점을 좁힘                           | 메모리 |
| **Bloom Filter**      | `Filter.db`          | "이 SSTable에 이 파티션 키가 있을 수도/없음"을 빠르게 판정하는 확률적 자료구조. false negative 없음(없다고 하면 진짜 없음), false positive만 있음. 메모리 상주 | 메모리 |
| **Compression Info**  | `CompressionInfo.db` | 압축 청크별 오프셋·길이 메타. 특정 행을 읽을 때 해당 압축 청크만 풀 수 있게 함                                                                | 디스크 |
| **Statistics**        | `Statistics.db`      | 파티션 크기 분포, 컬럼별 min/max, tombstone 비율·최소/최대 timestamp, clustering min/max 등 통계·메타데이터. compaction과 쿼리 최적화에 사용    | 디스크 |
| **TOC**               | `TOC.txt`            | 이 SSTable을 구성하는 컴포넌트 목록(Table Of Contents)                                                                     | 디스크 |
| **Digest**            | `Digest.crc32`       | Data.db의 체크섬. 무결성 검증용                                                                                          | 디스크 |
| **CRC**               | `CRC.db`             | 압축되지 않은 청크의 CRC 정보                                                                                             | 디스크 |

> 참고: `CompressionInfo.db`와 `CRC.db`는 보통 함께 존재하지 않는다. 테이블 압축이 켜져 있으면(기본값, LZ4Compressor) 압축 청크 메타가 `CompressionInfo.db`에 담기고, 압축을 끄면 대신 비압축 청크 체크섬이 `CRC.db`에 담긴다. 반면 `Digest.crc32`(데이터 파일 전체의 다이제스트)는 두 경우 모두 존재한다.

이 구조를 읽기 관점에서 한 줄로 꿰면 이렇다.

```text
  특정 파티션 키 K를 이 SSTable에서 찾는다:

   Bloom Filter(Filter.db)  →  "K 있을 수도?" 아니면 즉시 skip (디스크 안 봄)
        │ (있을 수도)
        ▼
   Partition Summary(Summary.db, 메모리)  →  Index.db에서 K 근처 시작 오프셋
        │
        ▼
   Partition Index(Index.db, 디스크)  →  Data.db에서 K의 정확한 바이트 오프셋
        │
        ▼
   Data.db (압축돼 있으면 CompressionInfo로 해당 청크만 해제)  →  실제 행 읽기
```

Bloom Filter가 첫 관문인 이유가 여기서 분명해진다. **"이 SSTable엔 그 키가 없다"를 디스크 접근 없이, 메모리에서, 거의 공짜로 판정**할 수 있다면, 키를 찾기 위해 뒤져야 할 SSTable 수가 극적으로 줄어든다(= 읽기 증폭 감소). false positive가 가끔 있어 헛걸음(Index.db까지 갔다가 없음)을 할 순 있지만, false negative는 없으므로 정확성은 깨지지 않는다. Bloom Filter 적중률과 메모리 트레이드오프, partition/key cache의 상호작용은 [[10 - Write Path와 Read Path]]에서 더 깊이 본다.

### Statistics.db가 조용히 하는 일

`Statistics.db`는 눈에 잘 안 띄지만 운영에서 매우 중요하다. 여기 담긴 **min/max timestamp, tombstone 비율, clustering 범위** 같은 메타데이터 덕분에 Cassandra는 읽기·compaction 때 영리한 가지치기를 한다. 예컨대 어떤 쿼리가 특정 시간 범위만 요구하는데 이 SSTable의 max timestamp가 그보다 작으면 통째로 건너뛸 수 있다. TWCS(Time Window Compaction Strategy)가 "이 SSTable은 이 시간 윈도우에 속한다"를 판단하는 근거도 여기서 나온다([[09 - Compaction 전략]]).

---

## Cassandra 5.0의 BTI 포맷: Summary/Index를 trie로 대체

지금까지 설명한 `Summary.db` + `Index.db` 조합은 오랫동안 잘 동작했지만 구조적 한계가 있었다. **Partition Summary는 "샘플"이라 정밀하지 않다.** 메모리를 아끼려고 일정 간격으로만 샘플링하므로 실제 키를 찾으려면 Summary로 대략 위치를 잡은 뒤 Index.db를 선형 스캔하듯 추가로 뒤져야 한다. 파티션이 아주 많아지면 Summary 자체도 무시 못 할 메모리를 먹고 sampling 간격을 조절하는 튜닝(`index_summary_resize_interval` 등)이 운영 부담이 됐다.

**Cassandra 5.0**은 이를 정조준한 새 SSTable 포맷 **BTI(Big Trie-Indexed)** 를 정식 도입했다(CEP-25). 골자는 partition index와 row index를 **trie(트라이) 자료구조**로 재설계하는 것이다.

- 기존 BIG 포맷: `Summary.db`(샘플) + `Index.db`(전체 매핑)
- 새 BTI 포맷: `Partitions.db`(파티션 trie 인덱스) + `Rows.db`(행 trie 인덱스). 별도의 Summary 샘플이 필요 없다.

trie가 왜 유리한가? 키들이 공유하는 **접두사(prefix)를 공유 경로로 압축**하기 때문이다. 파티션 키나 clustering 키가 비슷한 접두사를 많이 가질수록(시계열 키, 순차 ID, prefix가 같은 문자열 키 등) trie는 메모리를 극적으로 절약한다. trie의 키 탐색은 키 길이에 비례하므로, "Summary로 대충 잡고 Index를 추가로 스캔"하던 2단계가 **하나의 결정적 탐색**으로 합쳐진다. 결과적으로:

- **메모리 효율 개선**: 같은 데이터라도 인덱스가 차지하는 힙/off-heap 메모리가 줄어든다. 노드당 더 많은 SSTable·파티션을 같은 메모리로 감당할 수 있다.
- **튜닝 부담 감소**: Summary 샘플링 간격 같은 노브를 신경 쓸 일이 줄어든다.
- **큰 파티션·많은 파티션에서 특히 이득**.

5.0에서 SSTable 포맷은 **노드 단위로 `cassandra.yaml`에서 고른다**(`sstable` 설정 블록의 `selected_format`에 `big` 또는 `bti` 지정). 이는 CQL 테이블 속성이 아니라 노드 전역 기본값이며, 그 후 *새로 쓰는* SSTable에만 적용된다. 노드는 `big`과 `bti`를 모두 읽을 수 있으므로 포맷을 바꿔도 기존 SSTable을 즉시 변환할 필요 없이 디스크에 두 포맷이 **공존**하다가 compaction을 거치며 점차 새 포맷으로 수렴한다. 5.0 기본값은 여전히 `big`이고 BTI는 옵트인이지만, 기본 채택 여부와 마이그레이션 디테일은 배포 버전·설정에 따라 다르므로 **운영 적용 시에는 해당 클러스터의 `cassandra.yaml`과 5.0 릴리스 노트를 확인하라**(여기서는 메커니즘과 동기를 설명하는 데 집중한다). 5.0의 또 다른 큰 변화로 **Storage Attached Indexes(SAI)** 가 있는데, 이는 2차 인덱스 영역이라 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]·[[07 - CQL 완전 정복]] 흐름에서 다루는 게 적절하다.

> **요점**: BTI는 "데이터를 저장하는 방식"을 바꾼 게 아니라 "데이터를 찾는 인덱스를 더 메모리 효율적이고 결정적으로" 바꾼 것이다. LSM-Tree의 큰 그림(append, immutable, compaction)은 그대로다.

---

## Immutability의 연쇄 효과: 한 키가 여러 파일에 흩어진다

이제 immutable의 청구서를 받을 차례다. SSTable을 고칠 수 없다는 단순한 규칙이 다음의 연쇄를 만든다.

**수정도 삭제도 새 SSTable에 append된다.** 어떤 파티션의 행을 시점 T1에 INSERT하면 SSTable A에 들어간다. T2에 같은 행의 컬럼을 UPDATE하면, A를 고치는 게 아니라 (그 사이 flush가 일어났다면) **SSTable B에 새 버전이 append**된다. T3에 그 행을 DELETE하면 **SSTable C에 tombstone이 append**된다. 결과적으로:

```text
   같은 파티션 키 K에 대한 데이터가 시간이 지나며 흩어진다:

   SSTable A (오래됨)   SSTable B            SSTable C (최신)
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │ K: {a=1, ts=T1│    │ K: {a=2, ts=T2│    │ K: tombstone │
   │     b=10,ts=T1│    │   (a만 갱신)  │    │   (b 컬럼 삭제│
   │     }         │    │     }         │    │    ts=T3)    │
   └──────────────┘    └──────────────┘    └──────────────┘
        ▲                    ▲                    ▲
        └────────────────────┴────────────────────┘
              읽을 때 이 셋을 전부 모아 병합해야 K의 "현재 모습"을 안다
```

**그래서 읽기는 병합(merge)이다.** 파티션 K를 읽으라는 요청이 오면, Cassandra는 K가 들어 있을 수 있는 **모든 SSTable**(+ 아직 flush 안 된 Memtable)을 후보로 잡고, 각각에서 K의 조각을 꺼내 **셀 단위로 timestamp를 비교해 가장 최신 것을 승자로** 골라 하나의 행으로 재구성한다(last-write-wins, tombstone이 있으면 그 셀은 삭제로 처리). 이 병합 비용이 바로 읽기 증폭의 실체다. 그래서 Bloom Filter(후보 SSTable 줄이기)와 Statistics(시간/키 범위로 가지치기)가 그렇게 중요하다. 병합의 정확한 알고리즘과 캐시 상호작용은 [[10 - Write Path와 Read Path]]에서.

**그래서 compaction이 필연이다.** 시간이 지날수록 같은 키 조각이 점점 더 많은 SSTable에 흩어지고 읽기 병합 비용과 디스크 공간(옛 버전·tombstone)이 계속 불어난다. 누군가는 주기적으로 **여러 SSTable을 읽어 병합하고, 옛 버전·죽은 tombstone을 정리해, 새 SSTable 하나로 다시 쓰고, 옛 SSTable들을 지워야** 한다. 그게 compaction이다. LSM-Tree에서 compaction은 선택이 아니라 **엔진이 살아 있기 위한 필수 대사 작용**이다. 이것이 다음 장 전체의 주제다([[09 - Compaction 전략]]).

```text
  LSM-Tree 전체 생애주기 (write → flush → compaction)

   쓰기 ─┬─▶ Commit Log (디스크, append, durability)
         └─▶ Memtable  (메모리, 정렬 유지)
                 │  가득 참 / 메모리 압력 / commitlog 한도
                 ▼  flush (순차 write)
            ┌──────────┐
            │ SSTable 1│  immutable
            └──────────┘
                 │   ... 시간이 지나며 flush 반복 ...
            ┌──────────┐ ┌──────────┐ ┌──────────┐
            │ SSTable 1│ │ SSTable 2│ │ SSTable 3│  같은 키가 흩어짐
            └────┬─────┘ └────┬─────┘ └────┬─────┘
                 └──────┬─────┴──────┬─────┘
                        ▼ compaction (읽어서 병합 + tombstone 정리 + 재작성)
                  ┌──────────────┐
                  │  SSTable 4   │  더 적고 더 큰 파일, 옛 버전 제거됨
                  └──────────────┘
                  (옛 1,2,3은 삭제)
```

---

## Tombstone: "삭제했는데 디스크가 안 줄어드는" 이유

분산·immutable 환경에서 삭제는 생각보다 까다롭다. **노드 3대에 복제된 데이터를, 그중 한 노드가 다운된 사이에 삭제하면 어떻게 되는가?** 만약 삭제가 "데이터를 그냥 지우는 것"이라면, 죽었다 살아난 노드는 자기에게 남아 있는 옛 데이터를 보고 **"어? 다른 노드엔 없는데 나한텐 있네, 동기화로 다시 퍼뜨려야지"** 하며 **삭제된 데이터를 좀비처럼 되살린다.** 이걸 막으려면 삭제는 "없음"이 아니라 **"명시적으로 삭제됐다는 사실"** 로 기록돼야 한다. 그 기록이 **tombstone**이다.

그래서 `DELETE`는 데이터를 지우는 게 아니라 **삭제 표식을 append하는 쓰기**다. tombstone에도 timestamp가 있고 읽기 병합 시 그 timestamp보다 오래된 같은 셀의 데이터는 "삭제됨"으로 가려진다. tombstone이 더 새 데이터에 의해 덮일 수도 있다(삭제 후 재삽입). 이 모든 걸 timestamp 비교로 해결한다.

tombstone의 디스크 표현은 종류가 여럿이다.

- **셀(컬럼) tombstone**: 특정 컬럼 하나를 NULL로 만드는 삭제. (참고로 CQL에서 컬럼에 `NULL`을 INSERT/UPDATE하는 것도 tombstone을 만든다. 의도치 않게 tombstone을 양산하는 흔한 함정이다.)
- **행(row) tombstone**: 특정 clustering 행 전체 삭제.
- **range tombstone**: `DELETE ... WHERE pk=? AND ck >= ? AND ck < ?` 같은 범위 삭제. 한 표식이 범위 전체를 가린다.
- **partition tombstone**: 파티션 전체 삭제(`DELETE FROM t WHERE pk=?`).
- **TTL 만료**: TTL이 지난 셀은 만료 시점에 자동으로 tombstone과 동일하게 취급(만료된 셀 → tombstone으로 변환).

이제 그 유명한 현상으로 이어진다.

> **"방금 대량으로 DELETE 했는데 왜 디스크 사용량이 안 줄어드나요?"** — 정상이다. DELETE는 tombstone을 *추가*했을 뿐, 어떤 데이터도 즉시 지우지 않았다. 원래 데이터도, 새 tombstone도 둘 다 디스크에 그대로 있다. 회수는 **두 단계**로 나뉘어 일어난다는 점이 핵심이다. (1) **tombstone이 가리는 옛 데이터**는 그 데이터와 tombstone이 *같은 compaction에 함께 들어올 때* 제거된다. 이 단계는 `gc_grace`를 기다리지 않는다. tombstone이 그 데이터를 확실히 덮으므로 compaction 출력에는 tombstone만 남기고 옛 값은 버려도 안전하기 때문이다. (2) **tombstone 표식 자체**는 **`gc_grace_seconds`(기본 864000초 = 10일)** 가 지나고, 그 tombstone이 덮을 수 있는 더 오래된 데이터가 이 compaction 밖의 다른 SSTable에 없다고 확인될 때 비로소 purge된다. 그래서 대량 삭제 직후엔 디스크가 거의 그대로였다가, compaction이 돌면서 먼저 옛 데이터분이, 한참 뒤(gc_grace 경과 후)에 tombstone분이 단계적으로 빠진다.

왜 10일이나 기다리는가? 위에서 본 좀비 부활 문제 때문이다. tombstone은 클러스터의 **모든 replica가 이 삭제를 확실히 전파받을 때까지** 살아 있어야 한다. `gc_grace_seconds`는 "이 기간 안에는 (anti-entropy repair 등을 통해) 모든 노드가 이 삭제를 알게 될 것"이라고 가정하는 안전 마진이다. 이 기간이 지나야 compaction은 tombstone을 "이제 모두가 알았으니 진짜로 버려도 된다"고 판단하고 **purge**한다. 단, **그 tombstone이 가리는 옛 데이터가 들어 있는 SSTable들이 같은 compaction에 함께 들어와야** 데이터+tombstone이 같이 사라진다. 옛 데이터가 다른 SSTable에 떨어져 있고 함께 compaction되지 않으면, tombstone은 grace를 넘겼어도 데이터를 안전하게 지울 수 없어 더 남는다(droppable tombstone이 안 줄어드는 흔한 이유).

이 모든 사슬의 마지막 매듭이 **결제·정산 도메인에서의 함정**이다.

> **안티패턴 경고**: tombstone은 "읽기를 느리게 만드는 비용"이다. 같은 파티션을 읽을 때 그 안에 tombstone이 많으면, Cassandra는 살아 있는 데이터를 찾으려고 수많은 죽은 표식을 스캔해야 한다. `tombstone_warn_threshold`(기본 1000) / `tombstone_failure_threshold`(기본 100000)에 걸리면 경고·쿼리 실패가 난다. 그래서 **큐(queue)처럼 INSERT 후 곧바로 DELETE를 반복하는 패턴**, **시간이 지난 행을 DELETE로 정리하는 패턴**은 Cassandra에서 악명 높은 안티패턴이다. 정산 배치에서 "처리 완료한 행 삭제" 같은 워크로드를 무심코 짜면 tombstone 지옥에 빠진다. 대안은 삭제 대신 **TTL로 자연 만료 + TWCS로 통째 드롭**, 또는 시간 윈도 파티셔닝으로 옛 파티션 전체를 한 번에 버리는 것이다. 자세한 처방은 [[09 - Compaction 전략]]과 [[12 - 운영과 트러블슈팅]]에서.

---

## 데이터는 디스크에 "정렬되어" 누워 있다

SSTable의 S가 Sorted라는 걸 다시 강조하자. Data.db 안에서 데이터는 **두 단계로 정렬**되어 물리적으로 인접하게 저장된다.

1. **파티션 간**: 파티션 키의 **토큰(token)** 순서. 즉 partitioner(기본 Murmur3Partitioner)가 파티션 키를 해시한 토큰 값으로 정렬된다. 그래서 SSTable 안에서 파티션들은 토큰 순으로 나열된다(이것이 토큰 링과 직접 연결된다 — [[02 - 분산 아키텍처]]).
2. **파티션 내부**: **clustering 컬럼**이 정의한 순서. 테이블 정의의 `CLUSTERING ORDER BY`가 곧 디스크에 누운 물리적 순서다.

두 번째 사실이 데이터 모델링의 가장 중요한 직관을 만든다. **한 파티션 안의 행들은 clustering 순서대로 디스크에 연속으로 붙어 있다.** 그래서 다음과 같은 테이블에서:

```cql
CREATE TABLE payments_by_user (
    user_id     text,
    paid_at     timestamp,
    payment_id  text,
    amount      bigint,
    PRIMARY KEY ((user_id), paid_at, payment_id)
) WITH CLUSTERING ORDER BY (paid_at DESC, payment_id ASC);
```

`user_id`가 같은 결제 행들은 디스크에서 `paid_at DESC` 순으로 줄지어 누워 있다. 따라서 "특정 사용자의 최근 결제 N건"이나 "특정 기간의 결제"를 읽는 쿼리는 **디스크의 연속된 한 구간을 순차로 슬라이스**하면 끝난다. 랜덤 점프가 없다. 이것이 Cassandra에서 **범위 쿼리가 빠른 이유**이자, 동시에 **"읽고 싶은 순서대로 clustering order를 미리 정해야 하는 이유"** 다. 정렬은 쓰기 시점(Memtable)에 이미 끝나 있으므로 읽기 때 `ORDER BY`는 사실상 공짜이거나 불가능하다(저장 순서를 거스르는 정렬은 비싸거나 금지된다).

> RDB였다면 `WHERE user_id=? ORDER BY paid_at DESC LIMIT 10`을 위해 인덱스를 타거나, 최악엔 정렬을 위해 임시 테이블/소트를 돌렸을 것이다. Cassandra에서는 **데이터가 디스크에 이미 그 순서로 정렬되어 누워 있기 때문에** 그저 그 파티션의 앞부분 10행을 떼어내면 된다. 대신 그 대가로, "다른 순서로 정렬해서 보고 싶다"면 **다른 테이블을 하나 더 만들어 다르게 정렬해 저장**해야 한다. Query-First 모델링([[05 - 데이터 모델링 1 - Query First]])이 강제되는 물리적 뿌리가 바로 이 정렬 저장 방식이다.

이 정렬은 LSM 생애주기 내내 보존된다. Memtable이 정렬을 유지하고 → flush가 정렬된 채로 쏟아내 SSTable이 정렬되고 → compaction이 정렬된 SSTable들을 merge-sort로 합쳐 다시 정렬된 SSTable을 만든다. 정렬은 한 번 만들어지면 모든 단계에서 (재정렬 없이) 유지된다. 이게 LSM이 "정렬된 범위 읽기"를 싸게 제공할 수 있는 비결이다.

---

## 핵심 요약

- **LSM-Tree는 랜덤 쓰기를 순차 쓰기로 바꾸는 자료구조**다. B-Tree의 update-in-place(랜덤 쓰기) 대신 append-only(순차 쓰기)를 택해 **쓰기를 확정적으로 싸게** 만든다. 대가는 읽기·공간·쓰기 **증폭(amplification)** 의 삼각 트레이드오프이며, 이 균형을 고르는 것이 compaction 전략이다.
- **쓰기 경로는 짧다**: Commit Log append(디스크, durability) + Memtable 적용(메모리, 정렬). **디스크 read나 seek이 없어** 빠르다. UPDATE·DELETE도 전부 append하는 "쓰기"다.
- **Commit Log**는 노드 로컬 WAL로 crash 후 replay로 미flush Memtable을 복원한다. `commitlog_sync`는 **periodic(기본, 10초 주기 fsync, 빠름·유실 윈도 있음) / batch(fsync 후 ack, 안전·느림) / group(1초 윈도 묶음 fsync, 절충)**. 단일 노드 fsync보다 **복제+consistency level**이 진짜 durability를 책임진다.
- **Memtable**은 테이블별 정렬 메모리 버퍼다. on/off-heap 선택으로 GC를 관리하고, **메모리 압력(`memtable_cleanup_threshold`)·Commit Log 한도·시간·수동 명령**이 flush를 트리거한다. **flush = 새 SSTable 생성**이며 flush 중에도 쓰기는 새 Memtable로 계속 들어간다.
- **SSTable은 immutable**이며 `Data.db`(데이터) + `Index.db`/`Summary.db`(인덱스/샘플) + `Filter.db`(Bloom Filter) + `CompressionInfo.db` + `Statistics.db` + `TOC.txt` + `Digest`/`CRC` 등 여러 컴포넌트로 구성된다. 읽기는 **Bloom Filter → Summary → Index → Data** 순으로 좁혀간다.
- **Cassandra 5.0 BTI(Big Trie-Indexed) 포맷**은 Summary/Index 2단계를 **trie 기반 partition/row index**로 대체해 메모리 효율과 탐색 결정성을 높인다. 데이터 저장 방식이 아니라 인덱싱을 개선한 것.
- **Immutability의 연쇄**: 수정·삭제가 새 파일에 append → 같은 키가 여러 SSTable에 흩어짐 → 읽기는 timestamp 기반 **병합(LWW)** → 그래서 **compaction이 필연**이다.
- **Tombstone**: 삭제는 표식의 append다. 좀비 부활을 막으려 `gc_grace_seconds`(기본 10일)와 compaction을 거쳐야 디스크가 회수된다. **"DELETE 후 디스크가 안 줄어드는 것"은 정상**이며, 큐/대량삭제 패턴은 tombstone 안티패턴이다.
- **데이터는 토큰 순 + clustering 순으로 정렬되어 디스크에 연속 저장**된다. 그래서 범위 읽기가 싸고, `ORDER BY`가 저장 순서로 고정되며, Query-First 모델링이 물리적으로 강제된다.

## 연결 노트

- [[02 - 분산 아키텍처]] — 토큰 링과 partitioner. SSTable이 파티션 키 토큰 순으로 정렬되는 이유의 출발점.
- [[04 - Tunable Consistency]] — periodic Commit Log의 유실 윈도를 복제와 CL이 어떻게 메우는지. durability의 진짜 주체.
- [[05 - 데이터 모델링 1 - Query First]] / [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]] — 정렬 저장과 clustering order가 모델링을 강제하는 방식, tombstone/NULL 함정.
- [[07 - CQL 완전 정복]] — DELETE·TTL·`USING TIMESTAMP`가 스토리지 레벨에서 무엇을 append하는지.
- [[09 - Compaction 전략]] — immutability가 만든 흩어짐과 tombstone을 정리하는 백그라운드 대사. 이 장의 직접적 후속.
- [[10 - Write Path와 Read Path]] — 본 장의 메커니즘을 노드 간 좌표계(coordinator/replica)와 읽기 병합·캐시까지 확장.
- [[11 - LWT Batch Counter 내부]] — read-before-write가 필요한 예외 경로(LWT/counter)가 이 쓰기 모델과 어떻게 다른지.
- [[12 - 운영과 트러블슈팅]] — pending compactions, tombstone 임계값, Commit Log 디스크 분리 등 운영 징후와 대응.
- [[13 - 실전 Event Sourcing on Cassandra]] — append-only 스토리지 엔진과 이벤트 소싱의 자연스러운 궁합, 그리고 tombstone을 피하는 설계.
