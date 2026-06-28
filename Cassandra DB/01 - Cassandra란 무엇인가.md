---
title: Cassandra란 무엇인가 - 탄생 배경과 철학
date: 2026-06-26
tags: [cassandra, distributed-systems, cap-theorem, dynamo, bigtable, 학습노트]
---

이 장은 다음 질문들에 답한다. 이 질문들을 머릿속에 띄워두고 읽으면 나머지 장 전체가 하나의 일관된 그림으로 꿰어진다.

- 왜 페이스북의 한 엔지니어가 2008년에 새 데이터베이스를 처음부터 다시 짰을까? RDB로는 무엇이 불가능했나?
- "Dynamo 계보"와 "BigTable 계보"가 정확히 무엇을 의미하며, 그 둘이 합쳐졌을 때 어떤 화학반응이 일어나는가?
- CAP 정리에서 Cassandra가 "AP"라고 불리는 건 절반만 맞는 말이다. 왜 절반만 맞나? PACELC는 무엇을 더 말해주나?
- Cassandra는 무엇을 **포기**했고, 그 포기로 무엇을 **사들였나**? (JOIN을 버리고 무엇을 얻었나?)
- 결제 시스템 엔지니어인 당신에게: 언제 Cassandra를 쓰고, 언제 절대 쓰면 안 되는가?

분산 시스템과 RDB 경험이 있다는 전제 아래, "무엇(what)"의 나열이 아니라 "왜(why)"와 "내부에서 실제로 무슨 일이 일어나는가(how internally)"에 집중한다.

---

## 탄생 배경: Inbox Search가 깨뜨린 RDB의 환상

2007~2008년 페이스북의 사용자 수는 수천만에서 수억으로 폭발하던 시기였다. 이때 사내에서 골치 아픈 기능이 하나 있었다. **Inbox Search** — 사용자가 자신이 주고받은 모든 쪽지(메시지) 안에서 단어나 상대방 이름으로 검색하는 기능이었다.

이 워크로드의 특성을 RDB 엔지니어의 눈으로 뜯어보자.

- **쓰기가 미친 듯이 많다.** 메시지가 하나 오갈 때마다 역색인(reverse index) 엔트리가 여러 개 쌓인다. 단어 → (사용자, 메시지) 매핑이 수십억 건으로 불어난다.
- **데이터가 거대하다.** 수억 사용자 × 수년치 메시지. 단일 MySQL 인스턴스의 디스크와 메모리로는 어림도 없다.
- **항상 켜져 있어야 한다.** 페이스북은 글로벌 서비스다. "점검 시간"에 검색이 멈추면 안 된다.
- **지리적으로 분산되어 있다.** 미국 서부와 동부, 나중에는 여러 대륙. 한 데이터센터(datacenter)가 통째로 죽어도 서비스는 살아야 한다.

당시의 정석은 **MySQL 샤딩(sharding)** 이었다. user_id를 해시해서 수백 대의 MySQL에 흩뿌리는 방식. 하지만 이 길은 운영 지옥으로 이어진다.

> **RDB였다면**: 샤드를 추가할 때마다 데이터를 재분배(resharding)해야 하고, 그 동안 애플리케이션은 "어느 샤드에 무엇이 있는지" 매핑 테이블을 들고 다녀야 한다. 마스터(master)가 죽으면 수동 페일오버(failover) — 누군가 새벽에 일어나 슬레이브를 마스터로 승격시키고, 그 사이 쓰기는 멈춘다. 데이터센터 간 복제는 비동기 슬레이브로 흉내 낼 수는 있지만, 마스터가 한 곳에만 있으니 다른 대륙의 쓰기는 결국 한 곳으로 모인다. **단일 장애점(SPOF, Single Point of Failure)이 구조에 박혀 있다.**

페이스북은 이 문제를 풀기 위해 **Avinash Lakshman**과 **Prashant Malik**에게 새 저장소를 맡겼다. 여기서 짚어둘 디테일이 하나 있다. Avinash Lakshman은 그 직전 **Amazon에서 Dynamo 논문의 공저자**였다. Dynamo는 아마존 장바구니(shopping cart)를 위한, "절대 쓰기를 거부하지 않는" 분산 키-값 저장소였다. 그는 Dynamo가 가진 분산 운영의 우아함을 알고 있었지만, Dynamo는 단순 키-값 모델이라 Inbox Search 같은 풍부한 질의에는 부족했다.

그래서 그는 두 세계를 합치기로 한다. **Dynamo의 분산 인프라 + BigTable의 데이터 모델.** 이것이 Cassandra의 유전자다. 이름도 의미심장하다. 카산드라(Cassandra)는 그리스 신화에서 "아무도 믿지 않는 예언자"다. 오라클(Oracle, RDB의 제왕)에 대한 가벼운 농담이 섞여 있다는 해석도 있다.

**타임라인**:

```text
2007  Amazon, Dynamo 논문 발표 (Lakshman 공저)
2008  Facebook, Cassandra 사내 개발 → 7월 오픈소스로 공개
2009  Apache Incubator 입성
2010  Apache Top-Level Project로 졸업 (정식 Apache 프로젝트)
2010~ DataStax 설립, 상용 지원·CQL 등장
2015  Cassandra 3.x — Storage Engine 재작성, Materialized View
2021  Cassandra 4.0 — 안정성·성능, 내부 정리
2024  Cassandra 5.0 — SAI, UCS, Vector Search, JDK 17 (9월 GA, 이 노트의 기준)
```

흥미로운 역설이 있다. 정작 페이스북은 나중에 Inbox Search/Messages를 **HBase로 갈아탔다.** 그럼에도 Cassandra는 오픈소스 커뮤니티와 Netflix, Apple, Instagram, Uber 등을 등에 업고 독립적으로 성장했다. Apple은 공개된 발표 기준 10만 노드를 넘는 규모로 Cassandra를 운영한다(2014년 Summit에서 75,000노드/10PB, 2015년에 10만+ 노드로 언급). Cassandra의 진짜 검증은 페이스북이 아니라 그 이후의 거대 운영 현장에서 이루어졌다.

---

## 두 계보의 결합: Dynamo의 뼈대 + BigTable의 살

Cassandra를 한마디로 정의하라면 **"Dynamo의 인프라 위에 BigTable의 데이터 모델을 얹은 것"** 이다. 이 문장을 진짜로 이해하면 Cassandra의 거의 모든 설계 결정을 연역할 수 있다. 둘을 갈라서 보자.

### Dynamo로부터 물려받은 것 — "어떻게 분산하고 어떻게 죽지 않는가"

Dynamo는 데이터를 어디에 두고, 어떻게 복제하고, 노드가 죽었을 때 어떻게 버티는지에 관한 청사진이다. Cassandra는 이 거의 전부를 가져왔다.

| Dynamo 기술 | 무엇을 하나 | Cassandra에서의 모습 |
|---|---|---|
| **Consistent Hashing** | 키를 토큰 링(token ring) 위에 배치, 노드 추가/제거 시 일부만 이동 | 파티션 키 해시 → 토큰 → 담당 노드 결정 ([[02 - 분산 아키텍처]]) |
| **Replication** | 데이터를 N개 노드에 복제 | 복제 계수(replication factor, RF), 링 위 다음 노드들에 사본 ([[03 - 복제 전략과 데이터센터]]) |
| **Gossip Protocol** | 노드들이 서로의 상태를 잡담하듯 전파 | 매초 gossip, 마스터 없이 클러스터 멤버십·상태 동기화 |
| **Hinted Handoff** | 죽은 노드 몫의 쓰기를 살아있는 노드가 잠시 보관 | 노드 복귀 시 힌트(hint) 전달, "항상 쓰기 가능" 보장 |
| **Eventual Consistency** | 지금 당장은 아니어도 결국 같아진다 | Tunable consistency로 일관성 수준을 쿼리마다 선택 ([[04 - Tunable Consistency]]) |
| **Read Repair / Anti-Entropy** | 읽을 때·주기적으로 사본 간 불일치 보정 | read repair + Merkle tree 기반 repair |
| **Masterless (P2P)** | 모든 노드가 동등, 리더 없음 | 어느 노드든 코디네이터(coordinator)가 될 수 있음 |

무엇보다 중요한 건 마지막 줄, **마스터리스**다. Dynamo와 Cassandra에는 마스터/프라이머리/리더가 없다. 모든 노드가 완전히 동등(peer-to-peer)하다. 클라이언트의 요청을 받은 아무 노드나 그 요청의 "코디네이터" 역할을 임시로 맡아, 데이터가 실제로 사는 노드들에게 일을 분배하고 결과를 모은다. 마스터가 없으니 "마스터가 죽으면?"이라는 질문 자체가 성립하지 않는다. 이것이 Cassandra가 **단일 장애점이 없다(no SPOF)** 고 말하는 근거다.

### BigTable으로부터 물려받은 것 — "데이터를 어떻게 모델링하고 디스크에 어떻게 적는가"

Dynamo는 단순 키-값이었다. 키 하나에 불투명한(opaque) 값 하나. 검색 같은 풍부한 질의를 하기엔 모델이 너무 빈약했다. 그래서 Cassandra는 데이터 모델과 저장 엔진을 **Google BigTable**에서 빌려왔다.

| BigTable 기술 | 무엇을 하나 | Cassandra에서의 모습 |
|---|---|---|
| **Column-Family 데이터 모델** | 행(row) 안에 동적인 컬럼들, 희소(sparse) 저장 | 파티션(partition) + 클러스터링된 행, wide row ([[05 - 데이터 모델링 1 - Query First]]) |
| **Commit Log** | 쓰기를 먼저 순차 로그에 적어 내구성 보장 | commitlog, 크래시 복구의 기반 ([[10 - Write Path와 Read Path]]) |
| **Memtable** | 메모리 안의 정렬된 쓰기 버퍼 | memtable, 5.0은 trie 기반 메모리 인덱스 |
| **SSTable** | 불변(immutable) 정렬 파일로 디스크에 플러시 | SSTable, 한 번 쓰면 절대 수정 안 함 ([[08 - Storage Engine 내부]]) |
| **LSM-Tree** | 쓰기를 순차 append로 변환, 나중에 병합 | LSM(Log-Structured Merge) + compaction ([[09 - Compaction 전략]]) |

이 BigTable 계보의 핵심은 **LSM-Tree(Log-Structured Merge Tree)** 라는 저장 철학이다. RDB의 B-Tree는 데이터를 "제자리(in place)"에서 수정한다. 디스크의 특정 페이지를 찾아가 그 자리를 덮어쓴다. 이것은 랜덤 I/O(random write)를 유발한다. 반면 LSM-Tree는 **모든 쓰기를 메모리(memtable)에 모았다가 통째로 순차적으로(sequential write) 디스크에 새 파일로 떨군다.** 기존 파일은 절대 건드리지 않는다. 수정도 삭제도 "새로운 사실의 추가"로 표현한다.

```text
[RDB / B-Tree]                  [Cassandra / LSM-Tree]
쓰기 = 디스크 특정 위치 수정      쓰기 = 메모리에 추가 → 나중에 순차 flush
랜덤 I/O (느림)                  순차 I/O (빠름)
제자리 갱신 (in-place update)    불변 파일 (immutable, append-only)
읽기 빠름, 쓰기 비쌈             쓰기 빠름, 읽기는 여러 파일 병합 필요
```

이 한 가지 선택이 Cassandra를 **쓰기 최적화(write-optimized)** 데이터베이스로 만든다. Inbox Search처럼 쓰기가 폭발하는 워크로드에 LSM-Tree가 답이었던 이유다. 대신 대가가 있다. 하나의 논리적 행이 여러 SSTable에 흩어져 있을 수 있어 읽을 때는 여러 파일을 뒤져 병합해야 한다. 그래서 Cassandra의 읽기 경로에는 bloom filter, partition index, key cache 같은 장치들이 잔뜩 붙는다. 이 메커니즘은 [[08 - Storage Engine 내부]]와 [[10 - Write Path와 Read Path]]에서 깊이 파헤친다.

> **직관 하나**: Cassandra에서 `DELETE`는 데이터를 지우지 않는다. **"이 데이터는 삭제됨"이라는 표식(tombstone, 툼스톤)을 새로 추가**할 뿐이다. 불변 파일에는 무언가를 지울 수가 없으니, 삭제조차 추가로 표현한다. 이 사실은 나중에 "삭제를 많이 하면 오히려 읽기가 느려진다"는 반직관적 함정으로 돌아온다 ([[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]]).

### 두 계보가 만든 한 문장

그림으로 그리면 이렇다.

```text
        ┌──────────────── Amazon Dynamo ────────────────┐
        │  분산·운영·가용성: 어디에 두고, 복제하고,        │
        │  죽어도 버티는가 (consistent hashing, gossip,   │
        │  replication, hinted handoff, masterless)       │
        └────────────────────┬───────────────────────────┘
                             │  +
        ┌────────────────────┴───────────────────────────┐
        │  Google BigTable                                │
        │  모델·저장: 어떻게 모델링하고 디스크에 적는가     │
        │  (column-family, commitlog, memtable, SSTable,  │
        │   LSM-tree)                                      │
        └────────────────────┬───────────────────────────┘
                             ▼
                 ┌──────────────────────┐
                 │   Apache Cassandra   │
                 │  "절대 안 죽고, 쓰기를 │
                 │   거부하지 않으며,     │
                 │   선형으로 확장된다"   │
                 └──────────────────────┘
```

---

## CAP 정리를 정밀하게: Cassandra는 정말 "AP"인가?

분산 데이터베이스를 이야기할 때 거의 종교적으로 인용되는 것이 **CAP 정리(CAP theorem)** 다. Eric Brewer가 2000년 제시하고 Gilbert·Lynch가 2002년 증명했다. 그런데 이 정리는 거의 항상 **부정확하게** 인용된다. "세 개 중 둘만 고를 수 있다"는 표현은 절반쯤 틀렸다. 정확히 짚고 가자.

### C, A, P의 정확한 정의

- **C (Consistency, 일관성)**: 여기서의 일관성은 RDB의 ACID 'C'가 아니라 **선형성(linearizability)** 에 가깝다. 모든 읽기가 가장 최근에 완료된 쓰기를, 혹은 에러를 본다. 클라이언트 입장에서 마치 단일 노드가 순차적으로 처리하는 것처럼 보인다.
- **A (Availability, 가용성)**: 장애가 없는 모든 노드는 모든 요청에 **에러가 아닌 정상 응답**을 (유한 시간 안에) 반환한다. "느리지만 응답함"이 아니라 "반드시 응답함"이다.
- **P (Partition tolerance, 분단 내성)**: 네트워크가 노드 그룹들 사이의 메시지를 임의로 잃거나 지연시켜도(즉 네트워크 분단이 일어나도) 시스템이 계속 동작한다.

여기서 놓치면 안 되는 게 있다. **P는 선택지가 아니다.** 여러 대의 머신을 네트워크로 묶는 순간, 네트워크 분단은 "일어날 수도 있는 일"이 아니라 "반드시 일어나는 일"이다. 스위치가 죽고, 케이블이 끊기고, GC로 노드가 잠깐 멈춘다. 따라서 진짜 분산 시스템에게 선택지는 "P를 포기할 것인가"가 아니라, **"분단이 일어났을 때(P를 유지해야만 할 때), C와 A 중 무엇을 희생할 것인가"** 이다.

```text
        네트워크 분단(partition) 발생!
        노드 X와 노드 Y가 서로 대화 불가
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
   [CP를 택함]              [AP를 택함]
   일관성 유지를 위해        가용성 유지를 위해
   소수파(또는 양쪽)         양쪽 다 응답은 하되,
   요청을 거부(에러)         잠시 서로 다른 값을 볼 수 있음
        │                       │
   "틀린 답 줄 바엔          "오래된 답이라도
    응답 안 함"               응답은 함"
```

### Cassandra가 "AP"로 분류되는 이유 — 그리고 그게 왜 절반만 맞나

Dynamo 계보를 그대로 물려받은 Cassandra의 **기본 성향**은 **AP**다. 네트워크 분단이 일어나도 양쪽 파티션이 모두 읽기와 쓰기를 받아들인다. 노드 하나가 죽어도, 데이터센터 하나가 통째로 단절되어도, 남은 노드들은 계속 요청에 응답한다. "항상 쓰기 가능(always writable)"이라는 슬로건이 바로 이 AP 성향의 표현이다.

하지만 여기서 멈추면 Cassandra를 절반만 이해한 셈이다. **Cassandra의 일관성은 쿼리마다 조정 가능하다(tunable consistency).** 클라이언트는 매 읽기·쓰기마다 **일관성 수준(consistency level, CL)** 을 지정한다. 정족수(quorum) 규칙으로 한 쿼리를 사실상 CP처럼 동작시킬 수도 있다.

결정적인 부등식은 다음과 같다.

```text
W + R > RF  ⇒  강한 일관성(읽기가 항상 최신 쓰기를 봄)

W  = 쓰기가 성공으로 간주되기 위해 응답해야 하는 사본 수 (write CL)
R  = 읽기가 모아야 하는 사본 수 (read CL)
RF = 복제 계수(replication factor)
```

예를 들어 RF=3에서 쓰기와 읽기를 모두 `QUORUM`(과반수 = 2)으로 하면:

```text
W=2, R=2, RF=3  →  2 + 2 = 4 > 3  ✓
```

이때 읽기가 모은 2개 사본 중 적어도 하나는 반드시 마지막 쓰기를 본 노드와 겹친다(비둘기집 원리). 따라서 이 쿼리는 **강한 일관성**을 보장한다. 이 한 쿼리에 한해서는 Cassandra가 **CP처럼** 동작하는 셈이다. 분단으로 정족수를 못 모으면 쿼리는 실패한다(가용성을 희생). 정확히 CP의 거동이다.

여기서 한 가지를 정확히 못 박자. `W+R>RF`가 주는 "강한 일관성"은 **"마지막으로 성공한 쓰기를 읽는다"(read-your-writes·monotonic read)** 수준이지, 완전한 **선형성(linearizability)** 은 아니다. Cassandra의 쓰기는 사본들 사이에서 **격리(isolation)되지 않기 때문**이다. QUORUM 쓰기가 진행 중인 찰나에 들어온 QUORUM 읽기는 아직 정족수에 도달하지 못한 "쓰다 만" 값을 부분적으로 볼 수 있고(쓰기가 결국 성공할지 실패할지 모르는 상태에서도 노출될 수 있다), 동시 쓰기는 합의(consensus)가 아니라 **셀 타임스탬프 기반 last-write-wins**로 해소된다. 그래서 "읽고 → 조건 비교 → 갱신"(compare-and-set)처럼 **진짜 선형성**이 필요한 연산은 QUORUM으로는 부족하고 별도의 합의 프로토콜인 LWT(`SERIAL`/`LOCAL_SERIAL`, Paxos)를 써야 한다 ([[11 - LWT Batch Counter 내부]]). QUORUM은 "최신값 읽기"는 주지만 "원자적 read-modify-write"는 주지 않는다.

반대로 CL을 `ONE`으로 낮추면:

```text
W=1, R=1, RF=3  →  1 + 1 = 2 > 3 ?  ✗

→ 강한 일관성 보장 없음. 가장 빠르고, 거의 항상 쓰기/읽기 성공.
  하지만 방금 쓴 값을 다른 노드에서 읽으면 아직 못 볼 수 있음(AP).
```

> **흔한 오해**: "Cassandra는 AP니까 강한 일관성을 못 준다." → **틀렸다.** Cassandra는 **시스템 전체로는 AP 성향**이지만, **쿼리 단위로 일관성 수준을 선택**할 수 있어서 특정 쿼리를 CP처럼 만들 수 있다. CAP은 "시스템 전체에 영구히 붙는 라벨"이 아니라 "분단 상황에서의 거동"이고, Cassandra는 그 거동을 **쿼리마다** 바꾼다. 이 미묘함이 Cassandra 학습에서 가장 까다로운 관문이며, [[04 - Tunable Consistency]]에서 W/R/CL/RF의 모든 조합을 해부한다.

한 가지만 더 못 박자. 같은 CL이라도 이 보장은 **단일 파티션의 단일 행**에 대해서만 성립한다. Cassandra는 여러 파티션·여러 행에 걸친 원자성(atomicity)이나 격리(isolation)를 일반적으로 제공하지 않는다. 그것이 필요하면 경량 트랜잭션(LWT, Lightweight Transaction)이라는 별도의 무거운 메커니즘을 써야 한다 ([[11 - LWT Batch Counter 내부]]).

### PACELC: CAP이 말하지 않는 절반

CAP의 가장 큰 한계는 **"분단이 일어났을 때"만** 이야기한다는 데 있다. 그런데 분단은 드물다. 시스템은 **대부분의 시간을 분단 없이** 보낸다. 그 평상시의 트레이드오프를 CAP은 말해주지 않는다.

여기에 답하는 것이 **PACELC**(Daniel Abadi, 2012)다. 읽는 법은 이렇다:

```text
PAC ELC
 │  │
 │  └─ Else (분단이 없을 때): Latency(지연) vs Consistency(일관성)
 │
 └──── if Partition (분단이 있을 때): Availability vs Consistency  ← 이게 곧 CAP
```

풀어 쓰면: **"분단(P)이 일어나면(if) 가용성(A)과 일관성(C) 중 택하고, 그렇지 않으면(Else) 지연(L)과 일관성(C) 중 택한다."**

Cassandra의 PACELC 분류는 **PA/EL** 이다.

- **PA**: 분단 시 가용성을 택한다 (앞서 본 AP 성향).
- **EL**: **분단이 없는 평상시에도, 더 강한 일관성을 위해 더 많은 사본의 응답을 기다리면 그만큼 지연이 늘어난다.** `CL=ONE`은 빠르지만 약하고, `CL=ALL`은 느리지만 강하다. 분단이 없어도 이 트레이드오프는 매 쿼리에 살아있다.

이 EL 부분이 실무적으로 더 자주 마주치는 트레이드오프다. 멀티 데이터센터 환경에서 `LOCAL_QUORUM`을 쓰느냐 `EACH_QUORUM`을 쓰느냐는 바로 이 "평상시 지연 vs 일관성" 다이얼을 돌리는 일이다. 결제 시스템이라면 이 다이얼을 어디에 둘지가 곧 사용자 경험과 데이터 정합성의 균형점이 된다.

비교하자면 전통적 RDB(단일 마스터)는 **PC/EC**다. 분단 시에도, 평상시에도 일관성을 택한다. 대신 가용성과 지연을 희생한다. DynamoDB는 Cassandra처럼 PA/EL 성향이지만 강한 일관성 읽기 옵션을 제공한다.

---

## 설계 철학: 네 개의 기둥

지금까지의 내용을 "설계 철학"이라는 렌즈로 압축하면 네 개의 기둥이 보인다. 이 기둥들은 서로를 떠받친다.

### 기둥 1 — Masterless (단일 장애점 제거)

모든 노드가 동등하다. 리더 선출도, 마스터 페일오버도 없다. 어느 노드든 클라이언트 요청의 코디네이터가 될 수 있고 데이터는 토큰 링 위에 흩어져 RF만큼 복제된다.

```text
        Node A ───── Node B
          │  ╲      ╱  │
          │   ╲    ╱   │       ← 완전 P2P 메시 (gossip)
          │    ╲  ╱    │          마스터 없음. 누가 죽어도
        Node E   ╳   Node C       남은 노드가 계속 응답.
          │    ╱  ╲    │
          │   ╱    ╲   │
          └ Node D ────┘
```

> **RDB였다면 vs Cassandra**: MySQL 마스터-슬레이브에서 마스터가 죽으면 — 페일오버 도구(Orchestrator 등)가 슬레이브를 승격시키는 동안 **쓰기가 멈춘다**(수초~수분). Cassandra에선 노드 하나가 죽어도 "승격"이라는 개념 자체가 없다. 그 노드가 맡던 데이터는 이미 다른 노드에도 복제돼 있고, 코디네이터는 살아있는 사본에게 요청을 보낸다. 죽은 노드 몫의 쓰기는 hinted handoff로 잠시 보관됐다가 복귀 시 전달된다. **다운타임 0을 구조적으로 추구한다.**

### 기둥 2 — Linear Scalability (선형 확장)

노드를 2배로 늘리면 처리량이 (이상적으로) 2배가 된다. 마스터가 없으니 쓰기를 병목시키는 단일 지점이 없고 consistent hashing 덕에 노드 추가 시 데이터 재분배도 일부만 일어난다. Netflix의 유명한 벤치마크는 노드 수를 늘릴수록 초당 쓰기 처리량이 거의 직선으로 증가함을 보여줬다(수백 노드, 백만 단위 writes/sec).

```text
처리량
  │              ╱  Cassandra (선형)
  │            ╱
  │          ╱
  │        ╱
  │      ╱
  │    ╱        ┄┄┄┄┄ 샤딩 RDB (마스터 병목·재샤딩 비용으로 점차 평탄)
  │  ╱    ┄┄┄┄┄
  │╱┄┄┄┄
  └──────────────────────── 노드 수
```

이것이 "스케일 업(scale-up, 더 큰 서버)"이 아니라 **"스케일 아웃(scale-out, 더 많은 서버)"** 의 철학이다. 비싼 단일 대형 머신 대신 평범한 머신을 많이 늘린다(commodity hardware).

### 기둥 3 — Write-Optimized & Always Writable

LSM-Tree 덕에 쓰기는 메모리 append + commitlog 순차 기록으로 끝난다. 디스크 랜덤 I/O가 없다. 마스터리스 + hinted handoff 덕에 **노드 일부가 죽어도 쓰기를 받아들인다.** "쓰기를 절대 거부하지 않는다"는 Dynamo의 장바구니 철학이 그대로 살아있다.

> **정밀하게**: "항상 쓰기 가능"은 무조건이 아니라 **선택한 일관성 수준(CL)이 요구하는 만큼의 사본이 살아있을 때** 성립한다. CL이 낮을수록 더 많은 장애를 견딘다 — 극단인 `CL=ANY`에서는 담당 사본이 전부 죽어도 코디네이터가 **hinted handoff로 힌트만 남기고** 쓰기를 성공 처리한다(가장 공격적인 "always writable"). 반대로 `QUORUM`/`ALL`이면 정족수만큼 사본이 살아있어야 쓰기가 성공하고, 모자라면 쓰기는 실패한다. `ANY`를 제외하면 **hinted handoff가 보관한 힌트는 일관성 수준 충족 여부에 카운트되지 않는다**(힌트는 어디까지나 사후 복구용). "쓰기 거부 안 함"은 Dynamo의 *기본 성향*일 뿐, 다이얼을 어디 두느냐에 따라 강도가 달라진다.

```text
쓰기 한 번의 내부:
  1. commitlog에 순차 append (내구성 확보)   ─┐
  2. memtable(메모리)에 정렬 삽입            ─┴ 둘 다 빠른 연산
  3. 클라이언트에 ACK
  ───────────────────────────────────────
  (나중에 비동기로) memtable → SSTable flush
  (더 나중에) compaction으로 SSTable 병합
```

디스크에 무거운 일(flush, compaction)은 전부 **쓰기 경로 밖으로(비동기로)** 밀어냈다. 그래서 사용자가 체감하는 쓰기 지연이 일관되게 낮다. 자세한 메커니즘은 [[10 - Write Path와 Read Path]].

### 기둥 4 — Operational Simplicity (운영 단순성)

모든 노드가 같은 역할을 한다는 건 운영 관점에서 엄청난 단순화다. "마스터 노드용 설정"과 "슬레이브용 설정"이 따로 없다. 노드 추가는 새 머신에 같은 설정으로 Cassandra를 띄우고 클러스터에 합류시키면 끝(bootstrap). 노드를 빼는 것도 대칭적이다. 데이터센터 추가도 같은 패턴의 반복이다. **동질성(homogeneity)이 운영을 단순하게 만든다.** (물론 compaction 튜닝, repair 스케줄, tombstone 관리 같은 Cassandra 고유의 운영 숙제는 따로 있다 — [[12 - 운영과 트러블슈팅]].)

---

## 무엇을 포기했나(vs RDB): 그 포기로 무엇을 사들였나

엔지니어링은 공짜가 없다. Cassandra의 모든 장점은 RDB가 당연히 주던 것들을 **의도적으로 포기**한 대가다. 중요한 건 "무엇을 버렸나"가 아니라 **"그 포기로 무엇을 사들였나"** 이다. 거래(trade)의 양변을 같이 봐야 한다.

| 포기한 것 (RDB가 주던 것) | 왜 포기했나 | 그 대가로 사들인 것 |
|---|---|---|
| **JOIN** | JOIN은 여러 테이블의 데이터를 런타임에 모은다. 분산 환경에선 그 데이터가 여러 노드에 흩어져 있어 네트워크 셔플(shuffle)이 필요 → 확장 불가능 | 데이터를 노드 로컬에서 한 번에 읽음. 비정규화(denormalization)로 JOIN을 쓰기 시점에 미리 해둠. **예측 가능한 읽기 성능** |
| **임의 WHERE / ad-hoc 쿼리** | 인덱스 없는 컬럼으로 필터링하려면 전체 노드를 스캔(full scan) → 분산에선 재앙 | 쿼리는 파티션 키 기반으로만. **모든 읽기가 특정 노드로 라우팅**되어 O(1)에 가까운 노드 도달. Query-First 모델링 ([[05 - 데이터 모델링 1 - Query First]]) |
| **멀티키 ACID 트랜잭션** | 여러 노드에 걸친 2PC(2-phase commit)는 가용성과 지연을 죽인다(분단 시 블로킹) | 단일 파티션 원자성 + 필요 시 LWT(Paxos). **항상 쓰기 가능 + 선형 확장** ([[11 - LWT Batch Counter 내부]]) |
| **강한 일관성 default** | 모든 쓰기를 모든 사본에 동기 복제하면 가용성·지연이 무너짐 | Tunable consistency. **쿼리마다 일관성 vs 가용성/지연을 선택** ([[04 - Tunable Consistency]]) |
| **정규화(normalization)** | 정규화는 JOIN을 전제로 한다. 분산에선 JOIN이 불가능 | 의도적 비정규화·데이터 중복. 디스크는 싸고 읽기는 빠름. **쓰기 증폭을 감수하고 읽기를 단순화** |
| **단일 노드 격리(isolation) 수준** | 분산 MVCC·락은 코디네이션 비용이 큼 | last-write-wins(타임스탬프 기준 충돌 해소). **락 없는 동시 쓰기** |

이 표 전체를 한 문장으로 줄이면 이렇다. **"Cassandra는 '읽기 시점의 유연함'을 버리고 '쓰기 시점의 결정'으로 옮겼다."** RDB는 데이터를 정규화해서 저장해두고 읽을 때 JOIN·WHERE로 자유롭게 조합한다. Cassandra는 **"어떻게 읽을지"를 먼저 정하고, 그 읽기에 맞춰 데이터를 미리 가공해서 저장**한다. 이것이 그 유명한 **Query-First 데이터 모델링**이며, RDB 출신 엔지니어가 가장 크게 사고를 전환해야 하는 지점이다.

> **결제 엔지니어를 위한 번역**: RDB에서 `SELECT * FROM payments WHERE user_id = ? AND status = ? AND created_at > ?`를 즉흥적으로 날리던 습관을 버려야 한다. Cassandra에선 "사용자별 최근 결제 목록"이라는 **쿼리를 먼저 선언**하고, 그 쿼리가 단일 파티션에서 클러스터링 컬럼만으로 처리되도록 테이블을 설계한다. 새 쿼리 패턴이 생기면? **새 테이블(또는 비정규화된 사본)을 만든다.** "테이블 = 쿼리"라는 등식이 Cassandra 모델링의 출발점이다.

---

## 적합/부적합 워크로드: 언제 쓰고, 언제 쓰지 말 것인가

Cassandra는 만능이 아니다. 오히려 잘못된 곳에 쓰면 RDB보다 훨씬 고통스럽다. 도구의 모양을 알았으니 어떤 못에 이 망치를 쓸지 판단하자.

### Cassandra가 빛나는 워크로드

- **쓰기가 매우 많은(write-heavy) 시계열·이벤트 데이터.** IoT 센서, 로그, 메트릭, 클릭스트림, 그리고 **이벤트 소싱(event sourcing)**. append-only LSM 모델과 완벽히 들어맞는다 ([[13 - 실전 Event Sourcing on Cassandra]]).
- **알려진 접근 패턴이 고정된** 대규모 데이터. "사용자별 타임라인", "기기별 최근 측정값"처럼 파티션 키가 자연스럽게 정해지는 경우.
- **항상 켜져 있어야 하고(고가용성)**, 일부 노드/DC 장애를 견뎌야 하는 서비스.
- **지리적으로 분산**되어 멀티 DC 액티브-액티브가 필요한 경우. Cassandra의 멀티 DC 복제는 일급 시민이다 ([[03 - 복제 전략과 데이터센터]]).
- **거대한 데이터셋**(TB~PB)을 commodity hardware로 선형 확장하며 다뤄야 할 때.

### Cassandra를 피해야 할 워크로드 (안티패턴)

- **강한 트랜잭션 정합성이 본질인 도메인.** 계좌 잔액 이체처럼 "A에서 빼고 B에 더하는" 멀티키 원자성이 핵심이라면, Cassandra는 자연스러운 도구가 아니다. LWT로 흉내 낼 수는 있으나 비싸고(Paxos 라운드트립) 제한적이다.
- **즉흥적/분석적 쿼리(ad-hoc, OLAP).** "지난달 카테고리별 매출 집계를 갑자기 보고 싶다" 같은 임의 집계·JOIN은 Cassandra의 약점이다. 이런 건 분석 전용 시스템(Spark 같은 분석 엔진을 Cassandra 위에 얹거나, 데이터를 별도 데이터 웨어하우스로 빼서)으로 처리해야 한다.
- **데이터가 작은데(수 GB) 복잡한 관계가 많은** 경우. 그냥 PostgreSQL 한 대가 훨씬 단순하고 강력하다. **"우리도 스케일이 필요할지 몰라서"라는 막연한 이유로 Cassandra를 도입하는 것이 가장 흔한 실수다.**
- **읽기마다 조건이 바뀌는** 검색/필터 위주 워크로드. (5.0의 SAI가 이 영역을 일부 넓혀줬지만, RDB의 자유로운 WHERE를 대체하진 못한다.)
- **높은 카디널리티의 일관성 있는 카운팅**이 핵심인 경우. Cassandra 카운터는 존재하지만 까다롭고 멱등(idempotent)하지 않다 ([[11 - LWT Batch Counter 내부]]).

> **한 줄 판단 기준**: "내 접근 패턴을 **미리** 알 수 있고, 그게 **파티션 키로 라우팅** 가능하며, 무엇보다 **쓰기 처리량·가용성·확장성**이 절실한가?" 셋 다 yes면 Cassandra. 하나라도 흔들리면 RDB부터 의심하라.

---

## Cassandra 5.0 개요: 무엇이 새로워졌나

이 노트 전체의 기준 버전인 **Cassandra 5.0**(2024년 정식 릴리스)은 단순 버그픽스가 아니라 "Cassandra를 현대적 워크로드(특히 AI/검색)로 끌어올린" 메이저 릴리스다. 중요한 것만 짚는다. 각 항목의 내부는 해당 장에서 깊이 다룬다.

- **SAI (Storage-Attached Indexing)** — 새로운 2차 인덱스(secondary index) 엔진. 기존 2i(secondary index)는 노드별 로컬 인덱스라 카디널리티에 따라 성능이 들쭉날쭉했고 실무에서 기피 대상이었다. SAI는 SSTable에 밀착(storage-attached)되어 컬럼당 인덱스 비용이 낮고, 여러 인덱스를 조합한 쿼리, 범위·prefix 검색을 훨씬 효율적으로 처리한다. "Cassandra에서 인덱스는 쓰지 마라"는 오래된 격언을 부분적으로 뒤집은 변화다. (그래도 만능은 아니다 — [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]])
- **UCS (Unified Compaction Strategy)** — compaction 전략의 통합. 기존엔 STCS(Size-Tiered)/LCS(Leveled)/TWCS(Time-Window)를 워크로드에 맞춰 골라야 했고, 잘못 고르면 쓰기 증폭(write amplification)이나 공간 증폭(space amplification)으로 고생했다. UCS는 하나의 파라미터화된 전략으로 이들을 흡수해 튜닝을 단순화한다 ([[09 - Compaction 전략]]).
- **Vector Search / ANN** — `VECTOR<FLOAT, N>` 타입과 근사 최근접 이웃(ANN, Approximate Nearest Neighbor) 검색 지원. 내부적으로 JVector 라이브러리 기반. 임베딩(embedding)을 저장하고 코사인 유사도로 검색할 수 있어, Cassandra가 **벡터 데이터베이스/RAG 백엔드**로도 쓰일 수 있게 됐다. AI 시대를 겨냥한 가장 화제성 높은 기능이다.
- **JDK 17 지원** — 4.x는 JDK 8·11을 지원했다. 5.0은 JDK 11과 더불어 **JDK 17**을 지원해(향후 18+로 진행) 최신 GC(ZGC 등)와 언어/런타임 개선의 혜택을 받는다. GC 일시정지(pause)는 Cassandra의 꼬리 지연(tail latency)에 직접적이라 의미가 크다.
- **Trie 기반 메모리/디스크 인덱스** — memtable과 SSTable 인덱스에 trie(BTI, Big Trie-Indexed SSTable format) 자료구조를 도입. 기존 정렬 배열/스킵리스트 대비 메모리 효율과 조회 성능을 개선했다. 같은 힙으로 더 많은 데이터를 memtable에 담을 수 있어 flush 빈도와 compaction 부하가 줄어든다 ([[08 - Storage Engine 내부]]).

> **버전 주의**: SAI, UCS, Vector Search는 모두 5.0에서 정식화된 기능이다. 4.x 클러스터에서 이 기능들을 기대하면 안 된다. 또한 5.0의 일부 기능(예: trie 인덱스 SSTable 포맷)은 업그레이드 시 SSTable 포맷·호환성을 함께 점검해야 한다.

---

## 경쟁 비교: 분산 데이터 저장소 지형도에서 Cassandra의 자리

Cassandra를 진짜로 이해하는 좋은 방법은 비슷해 보이지만 다른 선택을 한 이웃들과 나란히 놓는 것이다. 각자가 어떤 트레이드오프를 골랐는지 보면 Cassandra의 정체성이 또렷해진다.

| | **Cassandra 5.0** | **DynamoDB** | **ScyllaDB** | **HBase** | **MongoDB** |
|---|---|---|---|---|---|
| **계보** | Dynamo + BigTable | Dynamo (의 상용 후예) | Cassandra 호환 재구현 | BigTable (의 OSS 구현) | 독자 문서 모델 |
| **아키텍처** | 마스터리스 P2P | 마스터리스(완전 관리형) | 마스터리스 P2P | 마스터/리전서버 + HDFS | 레플리카셋(프라이머리 1) + 샤딩 |
| **SPOF** | 없음 | 없음(관리형) | 없음 | HMaster/ZK 의존(완화됨) | 프라이머리 페일오버 필요 |
| **데이터 모델** | wide-column (CQL) | 키-값/문서 | wide-column (CQL 호환) | wide-column | 문서(BSON/JSON) |
| **구현 언어** | Java (JVM) | (비공개, 관리형) | **C++** (no GC) | Java (JVM) | C++ |
| **일관성** | tunable (PA/EL) | eventual / strong 선택 | tunable (Cassandra 호환) | 강한 일관성(행 단위) | tunable(write/read concern) |
| **운영 모델** | 셀프 호스팅/Astra | **완전 관리형(AWS 종속)** | 셀프/Cloud | 셀프(Hadoop 생태계) | 셀프/Atlas |
| **강점** | 멀티 DC, 쓰기 처리량, 성숙도 | 무운영, AWS 통합, 서버리스 | 같은 모델에 더 낮은 지연·높은 처리량 | Hadoop·강한 일관성 | 풍부한 쿼리·인덱스·집계 |
| **약점** | 운영 숙제, ad-hoc 쿼리 | AWS 락인, 비용 예측·핫키 | 생태계가 Cassandra만큼 넓진 않음 | 운영 복잡, 멀티 DC 약함 | 대규모 쓰기 확장은 Cassandra만 못함 |

각 비교를 한 줄로 요약한다.

- **vs DynamoDB**: 같은 Dynamo 철학이지만 DynamoDB는 **완전 관리형(AWS 종속)** 이다. "운영을 아예 안 하겠다"면 DynamoDB, "멀티 클라우드/온프렘에서 직접 통제하고 멀티 DC를 자유롭게 깔겠다"면 Cassandra. DynamoDB는 운영 부담이 0에 가깝지만 비용 모델(RCU/WCU)과 핫 파티션 제약, AWS 락인이 따라온다.
- **vs ScyllaDB**: ScyllaDB는 **Cassandra를 C++로 다시 쓴 것**이다. CQL과 프로토콜이 호환되며, JVM의 GC를 없애고 shard-per-core 아키텍처로 같은 워크로드에서 더 낮은 꼬리 지연과 더 높은 처리량을 낸다고 주장한다. "Cassandra 모델은 맞는데 JVM/GC 튜닝이 지긋지긋하다"면 검토 대상. 다만 커뮤니티·생태계·검증 사례의 폭은 Cassandra가 여전히 넓다.
- **vs HBase**: HBase는 BigTable의 직계 OSS 구현으로 **강한 일관성**(행 단위)을 기본 제공하지만, HDFS·ZooKeeper·HMaster에 의존해 운영이 무겁고 멀티 DC가 약하다. 페이스북이 Cassandra에서 HBase로 옮긴 것도 강한 일관성과 Hadoop 생태계 통합 때문이었다. "이미 Hadoop 생태계 안이고 강한 일관성이 필요하다"면 HBase, "멀티 DC 고가용성과 운영 단순성"이면 Cassandra.
- **vs MongoDB**: MongoDB는 애초에 다른 종족이다. **문서(document) 모델**에 풍부한 ad-hoc 쿼리·2차 인덱스·집계 파이프라인을 제공해 개발 편의성이 높다. 하지만 기본 구조가 **프라이머리 1 + 세컨더리**의 레플리카셋이라, 쓰기 확장은 샤딩에 의존하고 그 운영이 Cassandra의 대칭적 확장만큼 매끈하지 않다. "유연한 스키마와 풍부한 쿼리"가 우선이면 MongoDB, "거대한 쓰기 처리량과 마스터리스 가용성"이 우선이면 Cassandra.

지형도로 그리면 두 축이 보인다.

```text
        강한 일관성 / 풍부한 쿼리
                  ▲
         RDB ·    │   · HBase
        MongoDB   │
                  │
 ←────────────────┼────────────────→
 단일/관리 단순    │      대규모 쓰기·멀티DC·가용성
                  │
              DynamoDB · ScyllaDB
                  │   · Cassandra
                  ▼
        약한(조정형) 일관성 / 고정 접근 패턴
```

Cassandra는 이 지도의 오른쪽 아래(**대규모 쓰기 처리량, 마스터리스 고가용성, 멀티 DC, 조정 가능한 일관성**) 영역의 가장 성숙하고 검증된 좌표에 서 있다.

---

## 핵심 요약

- Cassandra는 **Facebook Inbox Search**(쓰기 폭발 + 항상 켜짐 + 지리 분산)를 위해 2008년 **Avinash Lakshman**(Dynamo 공저자)·**Prashant Malik**이 만들었고, 오픈소스 → Apache Top-Level Project로 졸업했다.
- 정체성은 한 문장: **Dynamo의 분산 인프라(consistent hashing·gossip·replication·hinted handoff·masterless·eventual consistency) + BigTable의 데이터 모델·저장(column-family·commitlog·memtable·SSTable·LSM-tree)**.
- **LSM-Tree**는 모든 쓰기를 순차 append로 바꿔 Cassandra를 **쓰기 최적화**로 만든다. 대신 읽기는 여러 불변 SSTable을 병합해야 하고, 삭제조차 **tombstone 추가**로 표현된다.
- **CAP**: P는 선택이 아니라 전제. Cassandra의 기본 성향은 **AP**지만, **tunable consistency**(W+R>RF)로 특정 쿼리를 **CP처럼** 만들 수 있다 — "Cassandra=AP라 강한 일관성 불가"는 오해.
- **PACELC**로 보강하면 Cassandra는 **PA/EL**: 분단 시 가용성, 평상시엔 일관성↔지연 트레이드오프가 매 쿼리에 살아있다.
- 설계 철학 네 기둥: **masterless(SPOF 없음) · linear scalability · write-optimized/always-writable · operational simplicity**.
- RDB 대비 **JOIN·임의 WHERE·멀티키 ACID·강한 일관성 default·정규화**를 버리고 → **예측 가능한 읽기·O(1) 노드 라우팅·항상 쓰기 가능·선형 확장**을 샀다. 그 결과가 **Query-First 모델링**("테이블=쿼리").
- **적합**: write-heavy 시계열·이벤트 소싱, 고정 접근 패턴, 고가용성, 멀티 DC, 거대 데이터셋. **부적합**: 멀티키 트랜잭션, ad-hoc/OLAP, 소규모 복잡관계, 임의 검색.
- **Cassandra 5.0** 하이라이트: **SAI**(현대적 2차 인덱스), **UCS**(통합 compaction), **Vector Search/ANN**(AI/RAG), **JDK 17**, **trie 기반 인덱스**.
- 지형도에서의 자리: DynamoDB(관리형·AWS 종속), ScyllaDB(C++ 재구현·저지연), HBase(강한 일관성·Hadoop), MongoDB(문서·풍부한 쿼리) 사이에서, Cassandra는 **대규모 쓰기·마스터리스 가용성·멀티 DC**의 성숙한 좌표를 차지한다.

## 연결 노트

- [[02 - 분산 아키텍처]] — 토큰 링·consistent hashing·gossip·코디네이터의 실제 동작 (이 장의 "Dynamo 계보"를 메커니즘으로)
- [[03 - 복제 전략과 데이터센터]] — replication factor·NetworkTopologyStrategy·멀티 DC
- [[04 - Tunable Consistency]] — W/R/CL/RF 조합, `LOCAL_QUORUM` vs `EACH_QUORUM`, CAP의 실전판
- [[05 - 데이터 모델링 1 - Query First]] — "테이블=쿼리" 사고 전환, 파티션/클러스터링 키 설계
- [[06 - 데이터 모델링 2 - 고급 타입과 안티패턴]] — tombstone·SAI의 한계·핫 파티션 등 함정
- [[08 - Storage Engine 내부]] — memtable·SSTable·trie 인덱스·bloom filter ("BigTable 계보"의 속살)
- [[09 - Compaction 전략]] — STCS/LCS/TWCS와 5.0의 UCS
- [[10 - Write Path와 Read Path]] — commitlog→memtable→flush, 읽기 시 SSTable 병합의 전 과정
- [[11 - LWT Batch Counter 내부]] — 포기한 ACID를 Paxos로 일부 되사오는 LWT
- [[13 - 실전 Event Sourcing on Cassandra]] — append-only LSM과 이벤트 소싱의 궁합 (결제 도메인 적용)
