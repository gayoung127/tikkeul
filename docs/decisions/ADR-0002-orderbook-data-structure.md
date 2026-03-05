---
status: "accepted"
date: 2026-02-23
---

# ADR-0002: 오더북 인메모리 자료구조 선택

## Context

오더북은 인메모리에서 동작하며, 가격-시간 우선순위(price-time priority)에 따라
주문을 정렬하고 매칭해야 한다. 주요 연산은 best bid/ask 조회, 주문 추가, 매칭(poll),
주문 취소(ID 기반 제거)이다.

## Decision Drivers

- 가격순 정렬 상태를 항상 유지해야 함
- best bid/ask 조회가 빈번 (오더북 조회 API)
- 같은 가격 내에서는 시간순(FIFO) 체결
- 주문 취소 시 ID로 빠르게 찾아 제거할 수 있어야 함
- 현재 규모에서 메모리는 병목이 아님

## Considered Options

1. **TreeMap + LinkedList + HashMap** — 가격 레벨은 TreeMap, 레벨 내 FIFO는 LinkedList, ID 인덱스는 HashMap
2. **PriorityQueue** — 주문 전체를 하나의 힙에 관리
3. **ConcurrentSkipListMap** — TreeMap의 동시성 안전 버전

## Decision

**Chosen option: "TreeMap + LinkedList + HashMap"**, because 각 연산에 최적화된 자료구조를 조합하여 모든 핵심 연산을 효율적으로 처리할 수 있다.

```
OrderBook
├── bids: TreeMap<BigDecimal, PriceLevel>  (reverseOrder — 높은 가격 우선)
├── asks: TreeMap<BigDecimal, PriceLevel>  (naturalOrder — 낮은 가격 우선)
└── orderIndex: HashMap<Long, OrderEntry>  (orderId → O(1) 조회)

PriceLevel
├── price: BigDecimal
├── orders: LinkedList<OrderEntry>         (FIFO)
└── totalQuantity: BigDecimal
```

## Consequences

### Positive

- best bid/ask 조회가 O(1) (TreeMap.firstEntry)
- 주문 추가가 O(log P) (P = 가격 레벨 수)
- 매칭이 O(1) (LinkedList.pollFirst)
- ID 기반 조회가 O(1) (HashMap)
- 가격 레벨별 집계(totalQuantity)를 PriceLevel이 직접 관리하므로 조회 API가 빠름

### Negative

- ID 기반 제거 시 LinkedList 내 선형 탐색 O(Q) 필요 (Q = 레벨 내 주문 수)
- 세 자료구조의 동기화를 직접 관리해야 함 (추가/제거 시 orderIndex도 갱신)
- 현재 단일 스레드에서는 문제없지만, 다중 스레드 전환 시 동기화 전략 필요

## Options Detail

### Option 2: PriorityQueue

- Good, 단일 자료구조로 단순함
- Bad, 같은 가격 내 FIFO 보장이 어려움 (Comparable에 시간 포함 시 성능 저하)
- Bad, 가격 레벨별 집계가 불가능 (오더북 조회 API에 부적합)
- Bad, ID 기반 제거가 O(N)

### Option 3: ConcurrentSkipListMap

- Good, 동시성 안전
- Bad, 현재 단일 스레드 매칭이므로 동시성 오버헤드가 불필요
- Neutral, 다중 스레드 전환 시 재검토 대상

## References

- 관련 문서: [오더북 설계 §2](../architecture/orderbook-design.md)
- 관련 문서: [오더북 도메인 모델](../domain/orderbook-domain.md)