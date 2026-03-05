---
작성일: 2026-02-23
버전: 1.0
목적: 인메모리 오더북의 자료구조, 매칭 엔진, 동시성 전략, DB 연동을 설계한다
---

# 오더북 설계

> 이 문서는 오더북의 **기술적 구현 설계**를 다룹니다.
> 오더북 개념과 매칭 규칙은 [오더북 도메인 모델](../domain/orderbook-domain.md)을 참고하세요.
> DB 스키마와 인덱스는 [데이터베이스 설계](database-design.md)를 참고하세요.

---

## 1. 설계 범위

### 1.1 이 문서가 다루는 것

- 인메모리 오더북의 자료구조 선택
- 클래스 구조와 책임 분배
- 매칭 엔진 알고리즘의 구현 수준 설계
- DB 트랜잭션과 오더북의 연동 흐름
- 동시성 제어 전략
- 서버 재시작 시 복구 절차

### 1.2 이 문서가 다루지 않는 것

- 매칭 규칙, 수수료 계산, 정산 로직 → [오더북 도메인 모델](../domain/orderbook-domain.md)
- Order/Trade 테이블 스키마 → [데이터베이스 설계](database-design.md)
- WebSocket 시세 배포의 상세 프로토콜/메시지 포맷 → 별도 문서 또는 구현 시 정의

---

## 2. 자료구조

### 2.1 오더북 구조

하나의 마켓(예: KRW-BTC)에 대해 하나의 오더북이 존재한다.
오더북은 **bids(매수)** 와 **asks(매도)** 두 면으로 구성된다.

```
OrderBook (KRW-BTC)
├── bids: TreeMap<BigDecimal, PriceLevel>  (가격 내림차순)
│   ├── 51,000,000 → PriceLevel [주문A, 주문B]  ← best bid
│   ├── 50,000,000 → PriceLevel [주문C]
│   └── 49,000,000 → PriceLevel [주문D, 주문E]
│
├── asks: TreeMap<BigDecimal, PriceLevel>  (가격 오름차순)
│   ├── 52,000,000 → PriceLevel [주문F]          ← best ask
│   ├── 53,000,000 → PriceLevel [주문G, 주문H]
│   └── 55,000,000 → PriceLevel [주문I]
│
└── orderIndex: HashMap<Long, OrderEntry>  (orderId → O(1) 조회/제거)
    ├── 주문A.id → 주문A
    ├── 주문B.id → 주문B
    └── ...
```

### 2.2 PriceLevel (가격 레벨)

같은 가격의 주문들이 모인 단위. **시간 우선 순서(FIFO)** 로 정렬된다.

```
PriceLevel (price = 51,000,000)
└── orders: LinkedList<OrderEntry>
    ├── [0] 주문A (created_at: 10:00:01) ← 먼저 체결
    ├── [1] 주문B (created_at: 10:00:05)
    └── [2] 주문C (created_at: 10:00:12)
```

### 2.3 OrderEntry (오더북 내 주문 표현)

오더북에는 매칭에 필요한 최소 정보만 보유한다.

```java
class OrderEntry {
    Long orderId;              // Order 테이블 PK
    Long memberId;             // self-trade 방지용
    BigDecimal price;          // 지정가
    BigDecimal remainingQty;   // 미체결 잔여 수량
    LocalDateTime createdAt;   // 시간 우선순위
}
```

Order 엔티티 전체를 보유하지 않는 이유:
- 메모리 절약
- 도메인 레이어와 결합도 낮춤
- 매칭에 불필요한 필드(status, filled_amount 등) 배제

### 2.4 자료구조 선택 근거

| 자료구조 | 역할 | 선택 이유 |
|---------|------|----------|
| **TreeMap** | 가격 레벨 관리 | O(log N) 삽입/삭제/조회, 정렬 상태 유지 |
| **LinkedList** | 같은 가격 내 주문 큐 | O(1) 앞에서 꺼내기 (FIFO), O(1) 뒤에 추가 |

TreeMap의 Comparator 설정:
- bids: `Comparator.reverseOrder()` — 높은 가격이 first
- asks: `Comparator.naturalOrder()` — 낮은 가격이 first

> 자료구조 최종 결정은 구현 단계에서 성능 테스트 후 확정 예정.
> ConcurrentSkipListMap 등 대안도 동시성 전략에 따라 고려한다.

---

## 3. 클래스 구조

### 3.1 전체 구조

```
OrderBookManager
├── manages: Map<Long, OrderBook>          마켓ID별 오더북
├── getOrderBook(marketId): OrderBook
└── initializeAll(): void                  서버 시작 시 전체 복구

OrderBook
├── marketId: Long
├── bids: TreeMap<BigDecimal, PriceLevel>
├── asks: TreeMap<BigDecimal, PriceLevel>
├── orderIndex: HashMap<Long, OrderEntry>   orderId → O(1) 조회/제거
├── addOrder(OrderEntry): void
├── removeOrder(Long orderId): void
├── getBestBid(): PriceLevel?
├── getBestAsk(): PriceLevel?
└── getMatchableOrders(side, price?, limit): List<OrderEntry>

PriceLevel
├── price: BigDecimal
├── orders: LinkedList<OrderEntry>
├── totalQuantity: BigDecimal              해당 가격의 총 잔여 수량
├── addOrder(OrderEntry): void
├── pollFirst(): OrderEntry                FIFO 꺼내기
├── removeOrder(Long orderId): void
└── isEmpty(): boolean

MatchingEngine
├── orderBookManager: OrderBookManager
├── match(NewOrderCommand): MatchResult
└── 내부: matchLimit(), matchMarketBuy(), matchMarketSell()

MatchResult
├── trades: List<TradeResult>              체결 결과 목록
├── remainingOrder: OrderEntry?            오더북에 남길 잔여 (LIMIT-GTC만)
└── refundAmount: BigDecimal?              반환 금액 (IOC 잔여)
```

### 3.2 패키지 위치

```
domain/trading/
├── orderbook/
│   ├── OrderBook.java
│   ├── OrderBookManager.java
│   ├── PriceLevel.java
│   └── OrderEntry.java
├── matching/
│   ├── MatchingEngine.java
│   ├── MatchResult.java
│   └── TradeResult.java
└── ...
```

오더북과 매칭 엔진은 **domain 레이어**에 위치한다.
외부 기술(DB, Redis)에 의존하지 않는다.

---

## 4. 매칭 엔진

### 4.1 매칭 흐름

```
[NewOrderCommand 수신]
    │
    ├─ LIMIT-BUY
    │     │
    │     ▼
    │   [asks에서 price 이하 주문 탐색]
    │     │
    │     ├─ 체결 가능 → matchLimit() 반복
    │     │     │
    │     │     └─ 잔여 있음 → MatchResult.remainingOrder 설정
    │     │
    │     └─ 체결 불가 → MatchResult.remainingOrder = 전체 주문
    │
    ├─ LIMIT-SELL
    │     │
    │     ▼
    │   [bids에서 price 이상 주문 탐색]
    │     └─ (LIMIT-BUY와 대칭)
    │
    ├─ MARKET-BUY
    │     │
    │     ▼
    │   [asks에서 best ask부터 순서대로]
    │     │
    │     ├─ 체결 → matchMarketBuy() 반복 (quote_amount 소진까지)
    │     │
    │     └─ 소진 또는 유동성 없음 → MatchResult.refundAmount 설정
    │
    └─ MARKET-SELL
          │
          ▼
        [bids에서 best bid부터 순서대로]
          └─ (MARKET-BUY와 대칭, quantity 소진까지)
```

### 4.2 매칭 규칙 요약

도메인 문서의 규칙을 구현 관점에서 정리한다.

| 규칙 | 구현 |
|------|------|
| 체결 가격 = maker의 price | `tradePrice = makerEntry.price` |
| 체결 수량 = min(taker 잔여, maker 잔여) | `tradeQty = taker.remainingQty.min(maker.remainingQty)` |
| 시장가 매수: 금액 기반 | `tradeQty = min(remainingQuote / makerPrice, maker.remainingQty)` |
| self-trade 금지 | `if (taker.memberId == maker.memberId) → skip 또는 cancel` |
| 가격 우선 → 시간 우선 | TreeMap 정렬 + LinkedList FIFO |

### 4.3 self-trade 처리

```
[매칭 루프 중 maker 주문 확인]
    │
    ├─ maker.memberId != taker.memberId → 정상 체결
    │
    └─ maker.memberId == taker.memberId → self-trade
          │
          └─ 해당 maker 주문 건너뜀 (skip)
              다음 주문으로 계속 매칭 시도
```

> 현재는 skip 방식을 사용한다.
> maker 주문을 취소하는 방식(cancel)은 필요 시 검토한다.

### 4.4 TradeResult 구조

하나의 매칭에서 발생하는 체결 정보:

```java
class TradeResult {
    Long makerOrderId;
    Long takerOrderId;
    Long buyerMemberId;
    Long sellerMemberId;
    BigDecimal price;          // 체결가 (maker price)
    BigDecimal quantity;       // 체결 수량
    BigDecimal quoteAmount;    // 체결 금액 (price × quantity)
    BigDecimal buyerFee;       // 매수자 수수료
    BigDecimal sellerFee;      // 매도자 수수료
    boolean makerFullyFilled;  // maker 주문 전량 체결 여부
    boolean takerFullyFilled;  // taker 주문 전량 체결 여부
}
```

---

## 5. DB 연동 흐름

### 5.1 주문 생성 → 매칭 → 정산 전체 흐름

```
[API: 주문 요청]
    │
    ▼
[트랜잭션 #1: 주문 저장 + 잔액 락]
    ├── Balance.lock(amount)
    ├── Order INSERT (status = OPEN)
    ├── BalanceHistory INSERT (type = LOCK)
    └── COMMIT
    │
    ▼
[매칭 엔진 호출] ← 트랜잭션 밖에서 실행
    ├── orderBook.getMatchableOrders(...)
    ├── 매칭 루프 수행
    └── MatchResult 반환
    │
    ▼
[트랜잭션 #2: 체결 결과 반영] (MatchResult의 각 trade에 대해)
    ├── Trade INSERT
    ├── 매수자 Balance 정산 (locked 감소, 상대 available 증가)
    ├── 매도자 Balance 정산
    ├── 매수자/매도자 BalanceHistory INSERT
    ├── Order 상태 업데이트 (filled_quantity, remaining_quantity, status)
    └── COMMIT
    │
    ▼
[오더북 업데이트] ← 트랜잭션 커밋 후
    ├── 체결된 maker 주문 제거 또는 수량 감소
    └── 잔여가 있는 taker 주문 → 오더북에 추가 (LIMIT-GTC만)
    │
    ▼
[WebSocket 배포]
    ├── 체결 내역 (trade) → 구독자에게 푸시
    ├── 호가 변동 (orderbook snapshot) → 구독자에게 푸시
    └── 시세 업데이트 (ticker) → 구독자에게 푸시
```

### 5.2 트랜잭션 분리 이유

| 트랜잭션 | 범위 | 이유 |
|---------|------|------|
| #1 주문 저장 | Order + Balance + History | 주문이 확정되어야 매칭 가능 |
| (매칭) | 트랜잭션 밖 | 매칭은 순수 연산, DB 접근 불필요 |
| #2 체결 반영 | Trade + Balance + History + Order | 체결 결과는 원자적으로 반영 |

매칭을 트랜잭션 안에서 수행하지 않는 이유:
- 매칭 연산 중 DB 커넥션을 점유하면 커넥션 풀 고갈 위험
- 매칭은 인메모리 연산이므로 DB 트랜잭션이 불필요
- 체결 결과만 트랜잭션으로 묶으면 충분

### 5.3 실패 시 복구

```
[트랜잭션 #1 실패] → 주문 자체가 생성되지 않음. 복구 불필요.

[매칭 중 서버 다운] → 주문은 DB에 OPEN 상태로 존재.
                       서버 재시작 시 오더북 복구 → 다시 매칭 가능.

[트랜잭션 #2 실패] → 체결이 DB에 반영되지 않음.
                       오더북에도 아직 반영 전이므로 정합성 유지.
                       재시도 또는 주문 상태 롤백 필요.
```

---

## 6. 동시성 전략

### 6.1 단일 스레드 매칭

```
[주문 API 스레드]                    [매칭 스레드 (단일)]
      │                                    │
      ├── 트랜잭션 #1 (주문 저장)            │
      │                                    │
      └── 매칭 큐에 NewOrderCommand 전송 ──►│
                                           ├── 매칭 수행
                                           ├── 트랜잭션 #2 (체결 반영)
                                           └── 오더북 업데이트
```

| 설계 결정 | 내용 |
|----------|------|
| 매칭은 단일 스레드 | 오더북에 대한 동시 접근 문제를 원천 차단 |
| API → 매칭은 큐로 연결 | API 응답 속도와 매칭 처리를 분리 |
| 오더북 락 불필요 | 단일 스레드이므로 동기화 비용 없음 |

> 매칭 큐는 Java의 `BlockingQueue`로 구현한다.
> 확장 시 메시지 큐(Kafka/RabbitMQ)로 전환할 수 있다.

### 6.2 주문 API 응답 시점

```
[주문 요청] → [잔액 락 + Order 저장] → [202 Accepted 응답]
                                           │
                                    [비동기로 매칭 진행]
```

주문 API는 **주문 접수 완료** 시점에 응답한다.
체결 결과는 비동기로 처리되며, 클라이언트는 WebSocket 구독 또는 주문 상태 조회로 확인한다.

### 6.3 마켓별 독립성

```
OrderBookManager
├── KRW-BTC 오더북 ← 매칭 스레드 A
├── KRW-ETH 오더북 ← 매칭 스레드 B
└── ...
```

마켓별로 오더북과 매칭 큐가 독립적이므로, 마켓 간 간섭이 없다.
마켓별로 오더북과 매칭 큐가 독립적이므로, 마켓 간 간섭이 없다.
현재는 단일 스레드가 모든 마켓을 순회하지만,
마켓별 스레드 분리도 가능한 구조로 설계한다.

---

## 7. 서버 재시작 복구

### 7.1 복구 절차

```
[서버 시작]
    │
    ▼
[1. 마켓 목록 로딩]
    │   SELECT * FROM market WHERE is_active = true
    │
    ▼
[2. 마켓별 빈 오더북 생성]
    │
    ▼ 마켓별 반복
[3. OPEN 상태 주문 조회]
    │   SELECT * FROM `order`
    │   WHERE market_id = ? AND status = 'OPEN' AND type = 'LIMIT'
    │   ORDER BY created_at ASC
    │
    ▼
[4. OrderEntry로 변환하여 오더북에 삽입]
    │   - remaining_quantity 기준
    │   - 매칭은 수행하지 않음
    │
    ▼
[5. 매칭 큐 수신 시작]
    │
    ▼
[복구 완료 → 새 주문 처리 가능]
```

### 7.2 복구 대상

| 조건 | 복구 대상 여부 | 이유 |
|------|:------------:|------|
| status = OPEN, type = LIMIT | ✅ | GTC로 오더북에 남아야 함 |
| status = OPEN, type = MARKET | ❌ | MARKET 주문은 오더북에 등록되지 않음 |
| status = FILLED | ❌ | 이미 체결 완료 |
| status = CANCELLED | ❌ | 이미 취소 완료 |

### 7.3 복구 시 매칭을 수행하지 않는 이유

- 서버 다운 동안 시장 상황이 변했을 수 있음
- 의도하지 않은 체결을 방지
- 복구는 **상태 복원**이지 **새로운 행위**가 아님
- 복구 후 새 주문이 들어오면 그때 자연스럽게 매칭됨

> 이 정책은 필요 시 재검토할 수 있다.
> 복구 후 크로스된 주문(체결 가능한 상태)이 있으면
> 별도 배치로 처리하는 방안을 고려한다.

---

## 8. 주문 취소

### 8.1 취소 흐름

```
[취소 요청 (orderId)]
    │
    ▼
[DB 조회: Order 상태 확인]
    │
    ├─ status != OPEN → 취소 불가 (이미 체결/취소됨)
    │
    └─ status == OPEN
          │
          ▼
    [트랜잭션: 취소 처리]
        ├── Order.status = CANCELLED
        ├── Balance.unlock(remaining locked amount)
        ├── BalanceHistory INSERT (type = REFUND)
        └── COMMIT
          │
          ▼
    [오더북에서 주문 제거]
        └── orderBook.removeOrder(orderId)
```

### 8.2 동시성 고려

취소 요청과 매칭이 동시에 발생할 수 있다.

```
시나리오: 취소 요청과 매칭이 동시에 실행

[매칭 스레드]                          [취소 API 스레드]
  매칭 루프에서 해당 주문 발견            │
  체결 처리 중...                       DB에서 Order 조회 (OPEN)
  트랜잭션 #2 커밋                      취소 트랜잭션 시작
  Order.status = FILLED                Order.status = CANCELLED ← 충돌!
```

**해결 방법**: 취소도 매칭 큐를 통해 처리한다.

```
[취소 요청] → [매칭 큐에 CancelOrderCommand 전송]
                    │
                    ▼ (단일 스레드에서 순차 처리)
              [오더북에서 제거 + DB 반영]
```

매칭과 취소가 같은 스레드에서 순차 처리되므로 동시성 문제가 발생하지 않는다.

---

## 9. 성능 고려사항

### 9.1 시간 복잡도

| 연산 | 복잡도 | 설명 |
|------|--------|------|
| 주문 추가 | O(log P) | P = 가격 레벨 수, TreeMap 삽입 + orderIndex put |
| 주문 제거 (ID) | O(1) 탐색 + O(Q) 제거 | orderIndex로 O(1) 탐색, LinkedList 내 제거 O(Q) |
| best bid/ask 조회 | O(1) | TreeMap.firstEntry() |
| 매칭 1건 | O(1) | PriceLevel에서 FIFO poll + orderIndex remove |
| N건 연속 매칭 | O(N + log P) | N건 poll + 빈 레벨 제거 |

### 9.2 orderIndex의 역할

주문 취소 시 orderId로 해당 주문을 즉시 찾아야 한다.
orderIndex(HashMap)가 이를 담당하며, 추가/제거/매칭 시 항상 동기화한다.

- addOrder: TreeMap + LinkedList에 추가하면서 orderIndex에도 put
- removeOrder: orderIndex에서 O(1) 탐색 → 해당 PriceLevel의 LinkedList에서 제거
- 매칭(poll): LinkedList.pollFirst 후 orderIndex에서도 remove

### 9.3 메모리 사용량 추정

| 항목 | 크기 | 비고 |
|------|------|------|
| OrderEntry | ~80 bytes | 5개 필드 |
| PriceLevel | ~40 bytes + orders | LinkedList 오버헤드 |
| 1,000 활성 주문 | ~120 KB | 충분히 작음 |
| 10,000 활성 주문 | ~1.2 MB | 현재 규모에서는 도달하지 않을 규모 |

메모리는 병목이 되지 않는다.