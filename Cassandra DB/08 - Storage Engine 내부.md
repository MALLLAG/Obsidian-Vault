---
title: Storage Engine 내부 - LSM-Tree, Commit Log, Memtable, SSTable
date: 2026-06-26
tags: [cassandra, storage-engine, lsm-tree, sstable, 학습노트]
---

이 장은 데이터가 디스크에 어떻게 놓이는지에 집중한다. Bloom Filter 적중률, key/row/chunk cache, read repair 타이밍 같은 읽기 경로의 세부는 [[10 - Write Path와 Read Path]]에서 자세히 다룬다.

---

## 왜 B-Tree가 아니라 LSM-Tree인가

RDB를 다뤄 본 엔지니어라면 이미 익숙한 디스크 모델이 하나 있다. 바로 **B-Tree(정확히는 B+Tree)** 이다. PostgreSQL, MySQL의 InnoDB, Oracle 등 거의 모든 전통적인 RDB가 이 구조를 기본 인덱스이자 테이블 저장 구조로 사용한다.

B-Tree는 **읽기에 최적화된 자료구조**이다. 트리 높이가 보통 3~4단계이므로 어떤 키든 디스크 페이지를 3~4번만 읽으면 찾을 수 있다. 또한 데이터를 페이지 안에 정렬된 상태로 in-place로 유지하기 때문에 범위 스캔도 효율적이다.

문제는 **쓰기**이다. B-Tree에 행 하나를 추가하거나 수정하면, 그 키가 속한 리프 페이지(leaf page)를 찾아가 그 페이지를 **제자리에서 고쳐 쓴다(update-in-place)**. 키는 디스크 전체에 흩어져 있으므로 이 페이지의 물리적 위치는 사실상 무작위이다. 즉 **랜덤 쓰기(random write)** 가 발생한다. 페이지가 가득 차면 split이 일어나 또 다른 무작위 위치에 쓰게 된다.

HDD 시절에 랜덤 쓰기가 느렸던 이유는 간단하다. 디스크 헤드를 물리적으로 옮기는 seek 시간(수 ms)이 대부분을 차지했기 때문이다. SSD에서는 seek이 사라졌지만 문제가 완전히 없어지지는 않았다. SSD는 페이지 단위로 읽고 블록 단위로 지운다. 그래서 in-place update는 read-modify-write와 write amplification(쓰기 증폭), 가비지 컬렉션 부담을 일으킨다.

게다가 분산 시스템에서 노드 하나가 초당 수만 건의 쓰기를 처리해야 한다면, 쓰기마다 트리의 리프 페이지를 찾아 락을 잡고 고쳐 쓰는 방식은 동시성 측면에서도 병목이 된다.

Cassandra는 처음부터 **쓰기가 매우 많이 들어오는 워크로드**를 가정하고 설계되었다(Google Bigtable과 Amazon Dynamo에서 출발했다). 그래서 B-Tree 대신 **LSM-Tree(Log-Structured Merge-Tree)** 를 선택했다. LSM-Tree의 발상은 한 문장으로 요약할 수 있다.

> **랜덤 쓰기를 순차 쓰기로 바꾼다(turn random writes into sequential writes).**

방법은 단순하다. 디스크의 정해진 자리를 고쳐 쓰는 대신, **들어온 순서대로 append만 한다.** 데이터를 메모리에 정렬된 상태로 모아 두었다가, 충분히 쌓이면 한꺼번에 디스크에 순차적으로 기록한다(sequential write). 이미 디스크에 쓴 파일은 절대 수정하지 않는다(immutable). 같은 키를 수정하거나 삭제할 때도 새 버전을 append할 뿐이다.

### B-Tree와 LSM-Tree 비교

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

### 대가: 세 가지 증폭(amplification)

LSM-Tree가 쓰기 비용을 낮춘 대가는 세 가지 "증폭"으로 정리할 수 있다. 이 세 가지는 LSM 계열 엔진을 이해하고 튜닝할 때 쓰는 기본 용어이므로 정확히 구분해 두는 것이 좋다.

- **쓰기 증폭(write amplification)**: 논리적으로 한 번 쓴 데이터가 디스크에는 여러 번 쓰인다. 처음에 Commit Log와 Memtable flush로 한 번 쓰이고, 이후 compaction이 같은 데이터를 다시 읽어 합친 뒤 다시 쓰면서 여러 번 더 쓰인다. 순차 쓰기로 바꿨다는 것은 쓰기 한 번의 비용을 낮춘 것이지 총 쓰기량을 줄인 것이 아니다. 이 값은 compaction 전략에 따라 크게 달라진다([[09 - Compaction 전략]]).
- **읽기 증폭(read amplification)**: 행 하나를 읽으려면 그 키가 흩어져 있는 여러 SSTable을 모두 확인해야 할 수 있다. 최악의 경우 SSTable 개수만큼 디스크 접근이 늘어난다. 이를 줄이기 위해 Bloom Filter, partition index, key cache를 사용한다.
- **공간 증폭(space amplification)**: 같은 키의 이전 버전, 삭제 표식(tombstone), 중복 데이터가 compaction 전까지 디스크에 함께 남아 있다. 그래서 논리적 데이터 크기보다 실제 디스크 사용량이 더 크다.

이 세 가지는 **동시에 모두 줄일 수 없다.** compaction을 공격적으로 실행하면 읽기 증폭과 공간 증폭은 줄지만 쓰기 증폭이 커지고, 느슨하게 실행하면 그 반대가 된다. 이 세 가지 사이에서 어떻게 타협할지 고르는 것이 compaction 전략 선택의 핵심이다.

> [!warning] 흔한 오해
> "LSM-Tree는 쓰기가 빠르고 B-Tree는 읽기가 빠르다"는 말은 절반만 맞다. LSM-Tree도 Bloom Filter, 인덱스, 캐시, compaction이 잘 동작하면 읽기가 충분히 빠르다. 정확히 말하면 **LSM-Tree는 쓰기 비용을 확실하게 낮추고, 읽기 비용의 일부를 백그라운드 작업(compaction)으로 미룬다(defer).** 비용을 없앤 것이 아니라 시간상 다른 시점으로 옮긴 것이다.

---

## 쓰기 경로 미리보기: append 한 번으로 끝난다

자세한 내용은 10장(Write Path와 Read Path)에서 다루지만, 스토리지 엔진을 이해하려면 쓰기가 디스크에 기록되는 순간을 먼저 살펴봐야 한다. 코디네이터가 담당 노드를 골라 RPC를 보낸 뒤 **각 replica 노드 안에서** 일어나는 일은 놀라울 만큼 짧다.

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

여기서 짚어 둘 점이 두 가지 있다.

**첫째, 어느 단계에서도 디스크에서 데이터를 읽지 않는다.** B-Tree였다면 "이 키가 어느 리프 페이지에 있는지 찾고, 그 페이지를 읽어 와서 고쳐 쓰는" read-before-write가 필요했다. Cassandra의 일반 쓰기 경로는 디스크를 읽지 않고, Commit Log 끝에 추가한 뒤 메모리에 넣기만 한다. 그래서 디스크 seek이 전혀 없고, 쓰기 지연이 메모리 연산 수준으로 낮다. 단, LWT(`IF NOT EXISTS` 등)와 counter는 read-before-write가 필요하다. 이 내용은 [[11 - LWT Batch Counter 내부]]에서 다룬다.

**둘째, UPDATE와 DELETE도 쓰기이다.** Cassandra에는 in-place update라는 개념이 없다. `UPDATE`는 새 값을 append하는 것이고, `DELETE`는 tombstone(삭제 표식)을 append하는 것이다. 기존 데이터를 찾아가서 고치거나 지우지 않는다. LSM의 append-only 원칙이 CQL 수준까지 그대로 적용되는 것이다.

> RDB라면 `UPDATE accounts SET balance=balance-1000 WHERE id=42`는 id=42 행을 찾아 그 자리에서 balance를 고친다. Cassandra에서 같은 문장은 "id=42, balance 컬럼에 새 값, timestamp T"라는 셀(cell)을 새로 append할 뿐이다.
>
> 이전 balance 값은 디스크 어딘가에 그대로 남아 있고, 읽을 때 timestamp를 비교해 최신 값이 선택된다(last-write-wins). 정산이나 결제 도메인에서는 이 모델이 직관과 어긋날 수 있으므로 특히 주의해야 한다.

---

## Commit Log: durability를 지키는 마지막 장치

Memtable은 메모리에 있다. 노드가 죽거나 전원이 나가면 **flush되지 않은 Memtable의 데이터는 모두 사라진다.** 그런데도 클라이언트에 "쓰기 성공" ack를 돌려줄 수 있는 이유는 **그 전에 Commit Log에 먼저 기록했기 때문**이다. Commit Log는 노드 로컬의 **append-only WAL(Write-Ahead Log)** 이고, 역할은 명확하다.

> 노드가 비정상 종료된 뒤 재시작하면, Cassandra는 Commit Log를 처음부터 재생(replay)해서 아직 SSTable로 flush되지 못했던 Memtable 상태를 메모리에 복원한다. 그래서 메모리에만 있던 데이터가 사라지는 사고를 막는다.

핵심은 **순차 append**라는 점이다. Commit Log는 항상 파일 끝에만 쓴다. 디스크 헤드를 옮길 일도, 페이지를 찾을 일도 없다. 그래서 durability를 보장하면서도 빠르다. 개념적으로는 RDB의 WAL이나 redo log와 같다.

> [!warning] 예외: `durable_writes`
> Commit Log 기록은 keyspace 단위로 끌 수 있다(`durable_writes`, 기본값 `true`). `false`로 두면 쓰기가 Commit Log를 건너뛰고 Memtable에만 적용되어 더 빠르지만, flush 전에 노드가 죽으면 그 데이터는 **복구되지 않는다**. 언제든 다시 만들 수 있는 파생 데이터나 캐시성 데이터가 아니라면 끄지 않는 것이 좋다. 결제나 정산 같은 원장성 데이터라면 고려할 대상이 아니다.

### segment와 재활용

Commit Log는 하나의 거대한 파일이 아니라 여러 **세그먼트(segment)** 로 나뉘어 있다. 세그먼트 하나의 기본 크기는 32MiB(`commitlog_segment_size`)이다. 세그먼트가 가득 차면 새 세그먼트를 만들어 계속 append한다. 참고로 단일 mutation의 최대 크기인 `max_mutation_size`는 기본적으로 세그먼트 크기의 절반인 16MiB이며, 쓰기 하나가 세그먼트 경계를 넘지 않도록 한다.

이 구조에는 잘 설계된 부분이 있다. 어떤 Memtable이 flush되어 그 안의 데이터가 모두 SSTable로 디스크에 안전하게 기록되면, **그 데이터를 담고 있던 Commit Log 세그먼트는 더 이상 필요 없다.** 재시작할 때 그 데이터는 SSTable에서 복구되기 때문이다.

Cassandra는 각 세그먼트가 어떤 테이블의 어느 시점까지를 담고 있는지 추적한다. 그리고 한 세그먼트가 담고 있던 Memtable이 모두 flush되면 그 세그먼트를 **재활용(recycle)하거나 삭제**한다. 그래서 Commit Log 디렉터리는 무한히 커지지 않고, 대략 아직 flush되지 않은 Memtable이 의존하는 만큼의 크기로 유지된다.

`commitlog_total_space`(기본값은 보통 8192MiB와 디스크의 1/4 중 작은 값)에 도달하면, Cassandra는 **가장 오래된 세그먼트를 비우기 위해 그 세그먼트가 의존하는 Memtable의 flush를 강제로 실행**한다. 즉 Commit Log 한도 초과도 Memtable flush를 일으키는 조건 중 하나이다(아래 Memtable 절에서 다시 다룬다).

### commitlog_sync: durability와 지연 사이의 선택

가장 중요한 튜닝 지점이다. Commit Log에 append했다고 해서 데이터가 곧바로 물리 디스크의 플래터나 플래시에 기록되는 것은 아니다. OS는 보통 데이터를 페이지 캐시에 모아 두었다가 나중에 디스크로 내린다. 실제로 디스크에 기록하려면 **`fsync`** 시스템 콜로 강제로 flush해야 한다.

문제는 fsync의 비용이 크다는 점이다. 그래서 Cassandra는 언제 fsync할지를 `commitlog_sync`로 선택하게 한다. 모드는 세 가지이다.

| 모드 | 동작 | 클라이언트 ack 시점 | durability | 지연/처리량 |
|------|------|---------------------|------------|-------------|
| **periodic** (기본) | 주기적으로 fsync (`commitlog_sync_period`, 기본 10000ms = 10초) | fsync 기다리지 않고 즉시 ack | 최대 `period`만큼의 쓰기가 유실 위험 | 가장 빠름 |
| **batch** | 쓰기를 모아 fsync, fsync 완료까지 ack 대기 | fsync 완료 후 ack | 유실 거의 없음 | 가장 느림(fsync 동기 대기) |
| **group** | 일정 시간 윈도(`commitlog_sync_group_window`, 기본 1000ms) 동안 쓰기를 모아 한 번에 fsync | 그룹 fsync 완료 후 ack | batch와 periodic 사이 | 절충 |

각 모드의 의미를 자세히 살펴보자.

**periodic(기본값)** 은 처리량을 가장 우선한다. 클라이언트가 쓰기를 보내면 Commit Log 버퍼에 append하고 **fsync를 기다리지 않고 바로 ack**한다. fsync는 이와 별도로 10초마다 실행된다. 따라서 노드가 fsync 직전에 갑자기 죽으면 **마지막 fsync 이후 약 10초 동안 ack했던 쓰기가 디스크에 없을 수 있다**.

> [!warning] 결제/정산 엔지니어가 반드시 짚어야 할 점
> periodic 모드에서 성공 ack를 받았다고 해서 단일 노드의 디스크에 영속화되었다는 뜻은 아니다. 이 안전성은 **복제(replication)와 consistency level**이 보장한다. RF=3에 `QUORUM`으로 쓰면 데이터가 2개 노드의 메모리와 Commit Log에 들어간 뒤 ack된다. 한 노드가 fsync 전에 죽어도 다른 노드가 데이터를 가지고 있고, hinted handoff나 read repair로 복구된다.
>
> 즉 Cassandra의 durability 모델은 **단일 노드의 fsync가 아니라 여러 노드에 분산 저장하는 것에 의존한다.** 이 원칙은 [[04 - Tunable Consistency]]와 직접 연결된다. 단일 노드 수준의 강한 durability가 꼭 필요하면 `batch`를 쓰되, 쓰기 지연이 fsync 지연에 묶이는 비용을 받아들여야 한다.

**batch** 는 안전성을 가장 우선한다. 쓰기를 받으면 fsync가 끝날 때까지 ack를 보류한다. 그래서 ack를 받으면 디스크에 기록되었다고 볼 수 있다. 대신 모든 쓰기가 디스크 fsync 지연에 순서대로 묶인다(회전 디스크라면 치명적이고, NVMe라도 비용이 있다). 그래서 batch는 보통 **빠른 NVMe SSD와 전용 Commit Log 디스크**를 갖춘 환경에서만 현실적이다.

**group** 은 batch처럼 fsync가 끝난 뒤 ack하는 안전성을 유지하면서도, 쓰기마다 fsync하지 않고 1초 윈도 동안 들어온 쓰기를 모아 한 번에 fsync한다. fsync 횟수를 줄여 처리량을 되찾는 절충안이다. batch는 처리량 때문에 부담스럽고 periodic의 유실 윈도는 불안할 때 고를 수 있는 중간 선택지이다.

**Commit Log 디스크를 데이터 디스크와 물리적으로 분리**하는 것은 오래된 권장 사항이다. 순차 쓰기인 Commit Log와 랜덤 접근이 섞인 SSTable I/O가 같은 디스크에서 헤드와 대역폭을 두고 경쟁하지 않게 하려는 것이다. 이 방법은 특히 HDD에서 효과가 컸고, 단일 NVMe 환경에서는 이득이 줄어든다. 환경에 따라 효과가 다르다는 점은 알아 두자.

---

## Memtable: 정렬된 메모리 버퍼

Commit Log가 durability를 맡는다면, **Memtable은 속도와 정렬을 맡는다.** Memtable은 테이블마다 하나씩 있는 **메모리 내 쓰기 버퍼**이다. 들어온 mutation을 **파티션 키 순으로, 파티션 안에서는 clustering 순으로 정렬한 상태**로 보관한다.

자료구조는 정렬된 맵이다. 전통적으로 `ConcurrentSkipListMap` 계열을 사용했고, 5.0에서는 trie 기반 메모리 인덱스 등으로 구현이 개선되었다. 중요한 것은 **항상 정렬된 상태를 유지한다**는 성질이다.

정렬을 유지하는 이유는 **flush할 때 다시 정렬하지 않아도 되게 하기 위해서**이다. Memtable은 이미 디스크 저장 순서(파티션 키 토큰 순 + clustering 순)로 정렬되어 있으므로, flush할 때는 정렬된 메모리 내용을 **디스크에 순차적으로 기록하기만** 하면 된다. 그 결과물이 정렬된 immutable 파일, 즉 SSTable이다. "Sorted String Table"의 Sorted는 여기서 나온 이름이다.

같은 키에 쓰기가 여러 번 들어오면 어떻게 될까? RDB라면 같은 행을 계속 덮어쓰겠지만, Memtable에서는 **같은 셀의 갱신이 메모리 안에서 병합된다.** 같은 파티션, 같은 clustering, 같은 컬럼에 대한 더 새로운 쓰기가 메모리에서 이전 값을 대체한다. 그래서 같은 키를 1초에 100번 갱신해도, 그 Memtable이 flush될 때는 대개 최신 버전 하나만 SSTable에 기록된다. LSM은 이런 방식으로 hot key 갱신을 효율적으로 처리한다.

### on-heap과 off-heap

Memtable은 JVM 위에서 동작하므로 **GC**를 항상 고려해야 한다. 수 GB의 데이터를 JVM heap에 두면 GC가 매번 그 데이터를 스캔해야 하므로 stop-the-world 시간이 길어진다. 그래서 Cassandra는 Memtable의 일부를 **off-heap(네이티브 메모리)** 에 둘 수 있게 하며, 이는 `memtable_allocation_type`으로 설정한다.

- `heap_buffers`: 모두 JVM heap에 둔다. 단순하지만 GC 부담이 크다.
- `offheap_buffers`: 셀 값(데이터) 버퍼를 off-heap에 둔다. 인덱스 구조는 heap에 둔다.
- `offheap_objects`: 더 많은 부분을 off-heap에 둔다.

off-heap을 쓰는 목적은 **GC가 스캔해야 할 heap 객체의 수와 크기를 줄여서 GC 일시 정지를 짧고 예측 가능하게** 만드는 것이다. 큰 Memtable을 운용하는 쓰기 중심 클러스터에서 효과가 크다.

### flush: Memtable이 SSTable이 되는 순간

메모리는 유한하므로 Memtable이 무한히 커질 수는 없다. 어느 시점에는 메모리 내용을 디스크로 내보내고 새 Memtable로 교체해야 한다. 이 과정이 **flush**이고, **flush의 결과물이 새 SSTable 하나**이다. flush를 일으키는 조건은 여러 가지이며, 그중 하나라도 먼저 충족되면 flush가 일어난다.

1. **`memtable_cleanup_threshold` (메모리 압력)**: 모든 테이블의 Memtable이 사용하는 메모리 풀(`memtable_heap_space`/`memtable_offheap_space`로 결정)이 일정 비율을 넘으면, Cassandra는 **현재 가장 큰 Memtable**을 골라 flush해서 공간을 확보한다. 이 임계값은 기본적으로 `1 / (memtable_flush_writers + 1)` 로 계산된다. 기본 `memtable_flush_writers`가 2이면 1/(2+1) ≈ 0.33이므로, 풀의 약 33%에서 flush가 일어난다. 풀이 찰수록 큰 Memtable부터 비워 메모리를 확보한다. 참고로 4.0부터는 `memtable_cleanup_threshold`를 직접 지정하는 방식이 **deprecated**되었고, 이 자동 계산식을 그대로 두는 것이 권장된다.
2. **Commit Log 공간 한도**: 앞의 Commit Log 절에서 본 것처럼 `commitlog_total_space`에 도달하면, 가장 오래된 세그먼트를 비우기 위해 그 세그먼트에 의존하는 Memtable의 flush가 강제된다. 즉 **Commit Log가 가득 차서 일어나는 flush**이다.
3. **시간 기반**: `memtable_flush_period_in_ms`(테이블별 설정, 기본값 0은 비활성)가 설정되어 있으면 그 주기마다 강제로 flush한다.
4. **수동 또는 운영 작업**: `nodetool flush`, `nodetool drain`(셧다운 전 전체 flush), 스키마 변경, 스냅샷 생성 등이 flush를 일으킨다.

flush는 다음 순서로 진행된다.

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

여기에도 잘 설계된 부분이 있다. **flush하는 동안에도 쓰기는 멈추지 않는다.** freeze된 이전 Memtable이 디스크로 기록되는 동안 새 쓰기는 새로 만든 빈 Memtable로 들어간다. 그래서 flush가 쓰기 가용성을 떨어뜨리지 않는다.

> **flush가 잦으면 작은 SSTable이 많이 생긴다.** 메모리 압력이 높거나 Commit Log가 자주 차서 Memtable이 작은 상태로 flush되면, flush가 자주 일어나고 그만큼 작은 SSTable이 늘어난다. SSTable 개수가 많아지면 읽기 증폭이 커지고, compaction이 따라가지 못해 "pending compactions"가 쌓인다.
>
> 그래서 Memtable 크기, Commit Log 한도, 힙 크기는 따로 보지 말고 **하나의 균형**으로 봐야 한다. 운영 중에 나타나는 징후와 대응 방법은 [[12 - 운영과 트러블슈팅]]에서 다룬다.

---

## SSTable: 변하지 않는 디스크 파일

flush의 결과물이자 Cassandra 디스크 데이터의 기본 단위가 **SSTable(Sorted Strings Table)** 이다. SSTable의 가장 중요한 성질은 **immutable(불변)** 이라는 것이다. 한 번 디스크에 쓰인 파일은 절대 수정되지 않는다. 행을 고치거나 지우거나 중간에 끼워 넣지도 않는다. SSTable에 일어나는 일은 두 가지뿐이다. **읽히거나, compaction으로 다른 SSTable과 합쳐져 새 SSTable이 만들어진 뒤 자신은 삭제되는 것이다.**

immutable이 주는 이점은 크다.

- **동시성이 단순해진다.** 아무도 파일을 수정하지 않으므로, 읽는 쪽은 락 없이 SSTable을 읽을 수 있다.
- **캐시, 압축, 인덱스가 안정적이다.** 파일이 변하지 않으므로 한 번 만든 Bloom Filter, partition index, 압축 청크가 계속 유효하다.
- **백업과 스냅샷의 비용이 낮다.** SSTable이 변하지 않으므로 스냅샷은 파일에 대한 하드링크(hard link)일 뿐이다. 데이터를 복사하지 않는다.
- **쓰기가 순차적이다.** 앞에서 본 모든 이점이 여기서 나온다.

하지만 immutable은 모든 복잡함의 원인이기도 하다. 이 내용은 다음 절에서 자세히 다룬다.

### SSTable을 구성하는 파일

SSTable 하나는 사실 디스크에서 **여러 개의 파일(컴포넌트)** 로 이루어진다. 같은 generation 식별자를 공유하는 파일 묶음이 논리적으로 하나의 SSTable이다(5.0에서는 `uuid_sstable_identifiers_enabled`로 ULID 기반 식별자를 켤 수도 있다).

BIG 포맷의 파일명은 `<포맷버전>-<generation>-big-<Component>.db` 형태이다. 예를 들면 `nb-1-big-Data.db`와 같다. 포맷 버전 문자는 릴리스마다 올라가므로, 단정하지 말고 디스크에서 직접 확인하는 것이 좋다. 각 컴포넌트와 역할은 다음과 같다.

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

> [!note] 참고
> `CompressionInfo.db`와 `CRC.db`는 보통 함께 존재하지 않는다. 테이블 압축이 켜져 있으면(기본값, LZ4Compressor) 압축 청크 메타데이터가 `CompressionInfo.db`에 담기고, 압축을 끄면 그 대신 비압축 청크의 체크섬이 `CRC.db`에 담긴다. 반면 데이터 파일 전체의 다이제스트인 `Digest.crc32`는 두 경우 모두 존재한다.

이 구조를 읽기 관점에서 순서대로 정리하면 다음과 같다.

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

이 흐름을 보면 Bloom Filter가 첫 번째 단계인 이유가 분명해진다. **"이 SSTable에는 그 키가 없다"는 사실을 디스크에 접근하지 않고 메모리에서 거의 비용 없이 판정**할 수 있다면, 키를 찾기 위해 확인해야 할 SSTable 수가 크게 줄어든다. 즉 읽기 증폭이 줄어든다.

false positive 때문에 가끔 Index.db까지 갔다가 키가 없는 경우도 있지만, false negative는 없으므로 정확성은 깨지지 않는다. Bloom Filter 적중률과 메모리 사이의 트레이드오프, partition/key cache의 상호작용은 10장(Write Path와 Read Path)에서 더 자세히 다룬다.

### Statistics.db의 역할

`Statistics.db`는 눈에 잘 띄지 않지만 운영에서 매우 중요하다. 여기에 담긴 **min/max timestamp, tombstone 비율, clustering 범위** 같은 메타데이터 덕분에 Cassandra는 읽기와 compaction에서 불필요한 SSTable을 효율적으로 걸러 낸다.

예를 들어 어떤 쿼리가 특정 시간 범위만 요구하는데 이 SSTable의 max timestamp가 그보다 작으면 파일 전체를 건너뛸 수 있다. TWCS(Time Window Compaction Strategy)가 어떤 SSTable이 어느 시간 윈도우에 속하는지 판단하는 근거도 이 파일이다(9장 Compaction 전략).

---

## Cassandra 5.0의 BTI 포맷: Summary/Index를 trie로 대체

지금까지 설명한 `Summary.db` + `Index.db` 조합은 오랫동안 잘 동작했지만 구조적인 한계가 있었다. **Partition Summary는 샘플이라서 정밀하지 않다.** 메모리를 아끼려고 일정 간격으로만 샘플링하므로, 실제 키를 찾으려면 Summary로 대략적인 위치를 잡은 뒤 Index.db를 선형 스캔하듯 추가로 확인해야 한다.

파티션이 아주 많아지면 Summary 자체도 적지 않은 메모리를 차지하고, 샘플링 간격을 조절하는 튜닝(`index_summary_resize_interval` 등)이 운영 부담이 되었다.

**Cassandra 5.0**은 이 문제를 해결하려고 새 SSTable 포맷인 **BTI(Big Trie-Indexed)** 를 정식으로 도입했다(CEP-25). 핵심은 partition index와 row index를 **trie(트라이) 자료구조**로 다시 설계한 것이다.

- 기존 BIG 포맷: `Summary.db`(샘플) + `Index.db`(전체 매핑)
- 새 BTI 포맷: `Partitions.db`(파티션 trie 인덱스) + `Rows.db`(행 trie 인덱스). 별도의 Summary 샘플이 필요 없다.

trie가 유리한 이유는 키들이 공유하는 **접두사(prefix)를 공통 경로로 압축**하기 때문이다. 시계열 키, 순차 ID, prefix가 같은 문자열 키처럼 파티션 키나 clustering 키에 비슷한 접두사가 많을수록 trie는 메모리를 크게 절약한다. 또한 trie에서 키를 탐색하는 비용은 키 길이에 비례하므로, Summary로 대략적인 위치를 잡고 Index를 추가로 스캔하던 두 단계가 **하나의 결정적인 탐색**으로 합쳐진다. 그 결과는 다음과 같다.

- **메모리 효율 개선**: 같은 데이터라도 인덱스가 차지하는 heap/off-heap 메모리가 줄어든다. 같은 메모리로 노드당 더 많은 SSTable과 파티션을 처리할 수 있다.
- **튜닝 부담 감소**: Summary 샘플링 간격 같은 설정을 신경 쓸 일이 줄어든다.
- **큰 파티션이나 파티션이 많은 경우에 특히 유리하다**.

5.0에서 SSTable 포맷은 **노드 단위로 `cassandra.yaml`에서 선택한다**(`sstable` 설정 블록의 `selected_format`에 `big` 또는 `bti`를 지정한다). 이 설정은 CQL 테이블 속성이 아니라 노드 전역 기본값이며, 설정한 뒤 *새로 쓰는* SSTable에만 적용된다.

노드는 `big`과 `bti`를 모두 읽을 수 있다. 따라서 포맷을 바꿔도 기존 SSTable을 즉시 변환할 필요가 없고, 디스크에 두 포맷이 **공존**하다가 compaction을 거치면서 점차 새 포맷으로 바뀐다.

5.0의 기본값은 여전히 `big`이고 BTI는 옵트인이다. 다만 기본 채택 여부와 마이그레이션 세부 사항은 배포 버전과 설정에 따라 다르므로, **운영에 적용할 때에는 해당 클러스터의 `cassandra.yaml`과 5.0 릴리스 노트를 확인해야 한다**. 여기서는 동작 원리와 도입 동기를 설명하는 데 집중한다.

5.0의 또 다른 큰 변화로 **Storage Attached Indexes(SAI)** 가 있다. SAI는 2차 인덱스 영역이므로 [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]과 [[07 - CQL 완전 정복]]에서 다루는 것이 적절하다.

> [!note] 요점
> BTI는 데이터를 저장하는 방식을 바꾼 것이 아니라, 데이터를 찾는 인덱스를 더 메모리 효율적이고 결정적인 방식으로 바꾼 것이다. append, immutable, compaction이라는 LSM-Tree의 큰 구조는 그대로이다.

---

## Immutability의 연쇄 효과: 한 키가 여러 파일에 흩어진다

이제 immutable의 대가를 살펴볼 차례이다. SSTable을 수정할 수 없다는 단순한 규칙이 다음과 같은 결과를 연쇄적으로 만든다.

**수정과 삭제도 새 SSTable에 append된다.** 어떤 파티션의 행을 시점 T1에 INSERT하면 그 데이터는 SSTable A에 들어간다. T2에 같은 행의 컬럼을 UPDATE하면, A를 고치는 것이 아니라 (그 사이에 flush가 일어났다면) **SSTable B에 새 버전이 append**된다. T3에 그 행을 DELETE하면 **SSTable C에 tombstone이 append**된다. 결과는 다음과 같다.

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

**그래서 읽기는 병합(merge)이다.** 파티션 K를 읽으라는 요청이 오면, Cassandra는 K가 들어 있을 수 있는 **모든 SSTable**과 아직 flush되지 않은 Memtable을 후보로 잡는다. 그리고 각 후보에서 K의 조각을 꺼내 **셀 단위로 timestamp를 비교해 가장 최신 값을 선택**하고, 이를 하나의 행으로 재구성한다(last-write-wins, tombstone이 있으면 그 셀은 삭제된 것으로 처리한다).

이 병합 비용이 바로 읽기 증폭의 실체이다. 그래서 후보 SSTable을 줄이는 Bloom Filter와, 시간이나 키 범위로 SSTable을 걸러 내는 Statistics가 중요하다. 병합의 정확한 알고리즘과 캐시의 상호작용은 10장(Write Path와 Read Path)에서 다룬다.

**그래서 compaction은 반드시 필요하다.** 시간이 지날수록 같은 키의 조각이 점점 더 많은 SSTable에 흩어지고, 읽기 병합 비용과 이전 버전 및 tombstone이 차지하는 디스크 공간이 계속 늘어난다. 따라서 주기적으로 **여러 SSTable을 읽어 병합하고, 이전 버전과 더 이상 필요 없는 tombstone을 정리한 뒤, 새 SSTable 하나로 다시 쓰고, 이전 SSTable을 지우는** 작업이 필요하다. 이 작업이 compaction이다.

LSM-Tree에서 compaction은 선택 사항이 아니라 **엔진이 계속 동작하기 위해 반드시 필요한 작업**이다. 다음 장(9장 Compaction 전략)은 전체가 이 주제를 다룬다.

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

## Tombstone: 삭제했는데 디스크 사용량이 줄지 않는 이유

분산 환경이면서 파일이 immutable인 환경에서는 삭제가 생각보다 까다롭다. **노드 3대에 복제된 데이터를, 그중 한 노드가 다운된 동안 삭제하면 어떻게 될까?** 삭제가 단순히 데이터를 지우는 것이라면, 다시 살아난 노드는 자신에게 남아 있는 이전 데이터를 보고 "다른 노드에는 없는데 나에게는 있으니, 동기화로 다시 퍼뜨려야겠다"고 판단한다. 그 결과 **삭제된 데이터가 좀비처럼 되살아난다.**

이를 막으려면 삭제를 "데이터 없음"으로 두지 않고 **"명시적으로 삭제되었다는 사실"** 로 기록해야 한다. 그 기록이 **tombstone**이다.

그래서 `DELETE`는 데이터를 지우는 것이 아니라 **삭제 표식을 append하는 쓰기**이다. tombstone에도 timestamp가 있으며, 읽기 병합 때 그 timestamp보다 오래된 같은 셀의 데이터는 삭제된 것으로 처리된다. 삭제한 뒤 다시 삽입하면 tombstone이 더 새로운 데이터로 덮일 수도 있다. 이 모든 경우를 timestamp 비교로 해결한다.

tombstone은 디스크에서 여러 종류로 표현된다.

- **셀(컬럼) tombstone**: 특정 컬럼 하나를 NULL로 만드는 삭제이다. 참고로 CQL에서 컬럼에 `NULL`을 INSERT/UPDATE해도 tombstone이 생긴다. 의도하지 않게 tombstone을 대량으로 만드는 흔한 함정이다.
- **행(row) tombstone**: 특정 clustering 행 전체를 삭제한다.
- **range tombstone**: `DELETE ... WHERE pk=? AND ck >= ? AND ck < ?` 같은 범위 삭제이다. 표식 하나가 범위 전체를 가린다.
- **partition tombstone**: 파티션 전체를 삭제한다(`DELETE FROM t WHERE pk=?`).
- **TTL 만료**: TTL이 지난 셀은 만료 시점에 자동으로 tombstone과 같이 취급된다(만료된 셀이 tombstone으로 변환된다).

이제 잘 알려진 현상 하나를 설명할 수 있다.

> **"방금 대량으로 DELETE했는데 왜 디스크 사용량이 줄지 않나요?"** 이것은 정상이다. DELETE는 tombstone을 *추가*했을 뿐이고, 어떤 데이터도 즉시 지우지 않았다. 원래 데이터와 새 tombstone이 모두 디스크에 그대로 남아 있다. 핵심은 공간이 **두 단계**에 걸쳐 회수된다는 점이다.
>
> 1. **tombstone이 가리는 이전 데이터**는 그 데이터와 tombstone이 *같은 compaction에 함께 들어올 때* 제거된다. 이 단계는 `gc_grace`를 기다리지 않는다. tombstone이 그 데이터를 확실히 덮으므로, compaction 출력에 tombstone만 남기고 이전 값은 버려도 안전하기 때문이다.
> 2. **tombstone 표식 자체**는 **`gc_grace_seconds`(기본 864000초 = 10일)** 가 지나고, 그 tombstone이 덮을 수 있는 더 오래된 데이터가 이 compaction에 포함되지 않은 다른 SSTable에 없다고 확인된 뒤에야 purge된다.
>
> 그래서 대량 삭제 직후에는 디스크 사용량이 거의 그대로이다가, compaction이 실행되면서 먼저 이전 데이터만큼 줄고, gc_grace가 지난 한참 뒤에 tombstone만큼 단계적으로 줄어든다.

왜 10일이나 기다릴까? 앞에서 본 좀비 부활 문제 때문이다. tombstone은 클러스터의 **모든 replica에 이 삭제가 확실히 전파될 때까지** 남아 있어야 한다. `gc_grace_seconds`는 이 기간 안에 anti-entropy repair 등을 통해 모든 노드가 이 삭제를 알게 될 것이라고 가정하는 안전 마진이다. 이 기간이 지나야 compaction은 모든 노드가 삭제를 알게 되었으니 tombstone을 버려도 된다고 판단하고 **purge**한다.

단, **그 tombstone이 가리는 이전 데이터가 들어 있는 SSTable이 같은 compaction에 함께 들어와야** 데이터와 tombstone이 같이 사라진다. 이전 데이터가 다른 SSTable에 있고 함께 compaction되지 않으면, tombstone은 grace 기간이 지났어도 데이터를 안전하게 지울 수 없으므로 계속 남는다. droppable tombstone이 줄지 않는 흔한 이유가 이것이다.

지금까지 설명한 내용은 결국 **결제·정산 도메인의 함정**으로 이어진다.

> [!warning] 안티패턴 경고
> tombstone은 읽기를 느리게 만드는 비용이다. 같은 파티션을 읽을 때 그 안에 tombstone이 많으면, Cassandra는 살아 있는 데이터를 찾기 위해 수많은 삭제 표식을 스캔해야 한다. `tombstone_warn_threshold`(기본 1000)나 `tombstone_failure_threshold`(기본 100000)를 넘으면 경고가 나거나 쿼리가 실패한다.
>
> 그래서 **큐(queue)처럼 INSERT한 뒤 곧바로 DELETE하기를 반복하는 패턴**과 **시간이 지난 행을 DELETE로 정리하는 패턴**은 Cassandra에서 잘 알려진 안티패턴이다. 정산 배치에서 "처리 완료한 행 삭제" 같은 워크로드를 별생각 없이 작성하면 tombstone이 쌓여 심각한 문제가 생긴다.
>
> 대안은 삭제 대신 **TTL로 자연스럽게 만료시키고 TWCS로 SSTable을 통째로 드롭**하거나, 시간 윈도로 파티셔닝해서 이전 파티션 전체를 한 번에 버리는 것이다. 자세한 해결 방법은 9장(Compaction 전략)과 12장(운영과 트러블슈팅)에서 다룬다.

---

## 데이터는 디스크에 정렬된 상태로 저장된다

SSTable의 S가 Sorted라는 점을 다시 강조하자. Data.db 안에서 데이터는 **두 단계로 정렬**되어 물리적으로 인접하게 저장된다.

1. **파티션 사이**: 파티션 키의 **토큰(token)** 순서로 정렬된다. 즉 partitioner(기본 Murmur3Partitioner)가 파티션 키를 해시한 토큰 값을 기준으로 정렬한다. 그래서 SSTable 안에서 파티션은 토큰 순으로 나열되며, 이는 토큰 링과 직접 연결된다([[02 - 분산 아키텍처]]).
2. **파티션 내부**: **clustering 컬럼**이 정의한 순서로 정렬된다. 테이블 정의의 `CLUSTERING ORDER BY`가 곧 디스크에 저장되는 물리적 순서이다.

두 번째 사실에서 데이터 모델링의 가장 중요한 직관이 나온다. **한 파티션 안의 행은 clustering 순서대로 디스크에 연속해서 저장된다.** 다음 테이블을 예로 들어 보자.

```cql
CREATE TABLE payments_by_user (
    user_id     text,
    paid_at     timestamp,
    payment_id  text,
    amount      bigint,
    PRIMARY KEY ((user_id), paid_at, payment_id)
) WITH CLUSTERING ORDER BY (paid_at DESC, payment_id ASC);
```

`user_id`가 같은 결제 행은 디스크에 `paid_at DESC` 순으로 나란히 저장된다. 따라서 "특정 사용자의 최근 결제 N건"이나 "특정 기간의 결제"를 읽는 쿼리는 **디스크의 연속된 한 구간을 순차적으로 잘라 읽기만** 하면 된다. 무작위로 위치를 옮겨 다니지 않는다.

이것이 Cassandra에서 **범위 쿼리가 빠른 이유**이자, **읽고 싶은 순서대로 clustering order를 미리 정해야 하는 이유**이다. 정렬은 쓰기 시점(Memtable)에 이미 끝나 있으므로, 읽을 때의 `ORDER BY`는 사실상 비용이 없거나 아예 불가능하다. 저장 순서와 다른 정렬은 비용이 크거나 금지된다.

> RDB라면 `WHERE user_id=? ORDER BY paid_at DESC LIMIT 10`을 처리하려고 인덱스를 사용하거나, 최악의 경우 정렬을 위해 임시 테이블을 만들고 소트를 실행했을 것이다. Cassandra에서는 **데이터가 이미 디스크에 그 순서로 정렬되어 있기 때문에** 그 파티션의 앞부분 10행을 잘라 내기만 하면 된다.
>
> 대신 다른 순서로 정렬해서 보고 싶다면 **테이블을 하나 더 만들어 다른 순서로 정렬해 저장**해야 한다. Query-First 모델링([[05 - 데이터 모델링 1 - Query First]])이 필요한 물리적 이유가 바로 이 정렬 저장 방식이다.

이 정렬은 LSM의 생애주기 전체에서 유지된다. Memtable이 정렬을 유지하고, flush가 정렬된 상태 그대로 기록하므로 SSTable도 정렬되어 있으며, compaction은 정렬된 SSTable들을 merge-sort로 합쳐 다시 정렬된 SSTable을 만든다. 한 번 만들어진 정렬은 모든 단계에서 다시 정렬하지 않고도 유지된다. 이것이 LSM이 정렬된 범위 읽기를 적은 비용으로 제공할 수 있는 이유이다.
