---
작성일: 2026-02-03
수정일: 2026-02-22
버전: 1.1
목적: 우리 서비스의 확정된 비즈니스 정책과 규칙을 정의한다
---

# 비즈니스 규칙

> 이 문서는 **우리 서비스**의 정책과 규칙을 정의합니다.
> 실제 거래소(업비트)의 분석은 [업비트 서비스 분석](upbit-analysis.md)을 참고하세요.
> 오더북과 매칭의 개념은 [오더북 도메인 모델](orderbook-domain.md)에서 다룹니다.

---

## 1. 용어 정의

| 용어 | 정의 | 예시 |
|------|------|------|
| **base** | 수량으로 거래되는 자산 | BTC, ETH |
| **quote** | 가격 단위로 사용되는 결제 자산 | KRW |
| **maker** | 호가창(오더북)에 먼저 주문을 등록한 사람 | 지정가 주문 후 대기 |
| **taker** | 호가창의 주문을 체결시킨 사람 | 시장가 주문자 |
| **tick_size** | 호가 단위 (주문 가능한 최소 가격 단위) | 1원 |
| **오더북** | 미체결 주문이 가격-시간 순으로 정렬되어 있는 자료구조 | — |
| **GTC** | Good Till Cancelled — 취소 전까지 유효 | 지정가 기본 |
| **IOC** | Immediate Or Cancel — 즉시 체결 후 잔여 취소 | 시장가 기본 |
| **FOK** | Fill Or Kill — 전량 체결 또는 전량 취소 | Phase 2 |

---

## 2. 비즈니스 정책

### 2.1 주문 정책

| 항목 | 값 | 비고 |
|------|-----|------|
| 최소 주문금액 | **5,000 KRW** | quote 기준 |
| 최대 주문금액 | 제한 없음 (잔액 범위 내) | |
| 호가 단위 (tick_size) | **1 KRW** | 모든 가격대 동일 |
| 주문 유효기간 | GTC (지정가) / IOC (시장가) | |

### 2.2 수수료 정책

| 항목 | 값 | 비고 |
|------|-----|------|
| 거래 수수료율 | **0.05%** | 매수/매도 동일 |
| 메이커/테이커 구분 | 미적용 (MVP) | Phase 2 고려 |
| 수수료 계산 시점 | 체결 시 | |
| 수수료 통화 | **받는 자산에서 차감** | 매수 → base(BTC), 매도 → quote(KRW) |

### 2.3 입출금 정책

| 항목 | 1회 한도 | 1일 한도 |
|------|----------|----------|
| 원화 입금 | 500만원 | 1억원 |
| 원화 출금 | 500만원 | 1억원 |
| 코인 입금 | 제한 없음 | 제한 없음 |
| 코인 출금 | 제한 없음 | 1억원 (원화 환산) |

| 항목 | 정책 |
|------|------|
| 24시간 출금 지연 | 미적용 (MVP) |
| 출금 수수료 | 원화 1,000원 / 코인 네트워크별 |

---

## 3. 주문 규칙

### 3.1 주문 유형별 입력 필드

| 주문 유형 | 필수 입력 | 금지 (NULL) | 설명 |
|-----------|-----------|-------------|------|
| LIMIT-BUY | price, quantity | quote_amount | 지정가 매수 |
| LIMIT-SELL | price, quantity | quote_amount | 지정가 매도 |
| MARKET-BUY | quote_amount | price, quantity | 시장가 매수 |
| MARKET-SELL | quantity | price, quote_amount | 시장가 매도 |

### 3.2 입력값 검증

```
# 호가 단위 검증 (LIMIT 주문만)
price % tick_size == 0

# 최소 주문금액 검증
LIMIT       : price × quantity >= 5,000
MARKET-BUY  : quote_amount >= 5,000
MARKET-SELL : 검증 없음 (수량 기반)
```

### 3.3 주문 상태 전이

```
                    ┌───────────────────────────────────────┐
                    │                                       │
                    ▼                                       │
[PENDING] ──▶ [OPEN] ──▶ [PARTIALLY_FILLED] ──▶ [FILLED] │
    │           │                │                          │
    │           │                │                          │
    │           ▼                ▼                          │
    │      [CANCELLED] ◀────────┘                          │
    │                                                       │
    └──▶ [REJECTED] ───────────────────────────────────────┘
```

| 상태 | 설명 | 전이 조건 |
|------|------|-----------|
| PENDING | 주문 생성됨 (검증 전) | 주문 요청 시 |
| OPEN | 오더북에 등록됨 | 검증 통과 + 잔액 락 완료 |
| PARTIALLY_FILLED | 일부 체결됨 | 0 < filled < quantity |
| FILLED | 전량 체결 완료 | filled == quantity |
| CANCELLED | 취소됨 | 사용자 취소 / 시장가 잔여 반환 |
| REJECTED | 거부됨 | 검증 실패 / 잔액 부족 |

---

## 4. 체결 규칙

### 4.1 체결의 전제조건

- 오더북이 존재해야 한다
- 시장가 주문의 경우 오더북에 상대방 주문(유동성)이 있어야 한다
- 상대방 주문이 없으면 시장가 주문은 즉시 취소된다 (IOC)

### 4.2 체결 우선순위

```
1순위: 가격 우선
  - 매수: 높은 가격 우선
  - 매도: 낮은 가격 우선

2순위: 시간 우선
  - 동일 가격일 경우 먼저 제출한 주문 우선
```

### 4.3 체결 가격

```
체결 가격 = maker 주문의 price (항상)
```

Taker가 시장가 주문 시 Maker의 지정가로 체결된다.
가격 개선(Price Improvement) 가능: Taker에게 유리한 가격으로 체결될 수 있다.

### 4.4 부분 체결 후 처리

| 주문 유형 | 부분 체결 후 잔여 처리 |
|-----------|----------------------|
| LIMIT (GTC) | 잔여 수량 오더북에 유지 |
| MARKET (IOC) | 잔여 금액/수량 즉시 반환 |

### 4.5 자기 체결 방지 (Self-Trade Prevention)

```
if (maker_order.member_id == taker_order.member_id) {
    // 매칭 스킵 → 다음 호가로 이동
}
```

---

## 5. 잔액 락/언락/정산 규칙

### 5.1 주문 시 락 (Lock)

| 주문 유형 | 락 대상 | 락 금액 계산 |
|-----------|---------|-------------|
| LIMIT-BUY | KRW (quote) | `price × quantity` |
| LIMIT-SELL | BTC (base) | `quantity` |
| MARKET-BUY | KRW (quote) | `quote_amount` |
| MARKET-SELL | BTC (base) | `quantity` |

### 5.2 체결 시 정산

**매수자 (Buyer):**

```
quote.locked   -= 체결금액
base.available += (체결수량 - 수수료)

수수료 = 체결수량 × 0.0005
```

**매도자 (Seller):**

```
base.locked     -= 체결수량
quote.available += (체결금액 - 수수료)

수수료 = 체결금액 × 0.0005
```

### 5.3 잔여금 처리

**LIMIT (GTC):**
- 부분 체결: 잔여 수량은 계속 locked 유지
- 더 유리한 가격에 체결된 경우: 차액을 locked → available로 반환

**MARKET (IOC):**
- 체결 불가 잔여: 즉시 locked → available로 반환
- 주문 상태: CANCELLED

### 5.4 주문 취소 시 언락

```
취소 가능 조건: status IN ('OPEN', 'PARTIALLY_FILLED')

처리:
1. remaining_quantity 기반 locked 금액 계산
2. locked 감소, available 증가
3. 주문 상태 → CANCELLED
```

---

## 6. 원장 기록 정책 (Balance History)

### 6.1 기록 원칙

1. 모든 잔액 변동에 대해 1건 이상 기록
2. amount는 항상 양수, type으로 방향 표현
3. 멱등성 키(idempotency_key)로 중복 방지
4. 원장 기록은 잔액 업데이트와 동일 트랜잭션 내 처리

```
검증 규칙: 현재 잔액 = SUM(원장 기록)
```

### 6.2 변동 유형

| type | 설명 | available | locked |
|------|------|-----------|--------|
| DEPOSIT | 입금 | +amount | — |
| WITHDRAWAL | 출금 | -amount | — |
| LOCK | 주문 시 동결 | -amount | +amount |
| UNLOCK | 취소 시 해제 | +amount | -amount |
| BUY | 매수 체결 (base 수령) | +amount | — |
| BUY_SETTLE | 매수 정산 (quote 차감) | — | -amount |
| SELL | 매도 체결 (quote 수령) | +amount | — |
| SELL_SETTLE | 매도 정산 (base 차감) | — | -amount |
| FEE | 수수료 차감 | -amount | — |
| REFUND | 잔여금 반환 | +amount | -amount |

### 6.3 멱등성 키 규칙

```
idempotency_key = {reference_type}:{reference_id}:{type}:{coin_id}

예시:
- ORDER:100:LOCK:1       → 주문 100번의 KRW 동결
- TRADE:200:BUY:2        → 체결 200번의 BTC 수령
- TRADE:200:BUY_SETTLE:1 → 체결 200번의 KRW 차감
- TRADE:200:FEE:2        → 체결 200번의 BTC 수수료
```

### 6.4 기록 예시: 지정가 매수 → 체결

```
1. 주문 생성 (BTC 0.01개를 1,000만원에 매수)
   [LOCK] KRW: available -100,000 / locked +100,000

2. 체결 (0.01 BTC @ 10,000,000)
   [BUY_SETTLE] KRW: locked -100,000
   [BUY]        BTC: available +0.00995 (0.01 - 수수료)
   [FEE]        BTC: available -0.00005 (수수료)
```

---

## 7. 무결성 규칙

### 7.1 Balance 제약

```sql
CHECK (available >= 0)  -- 음수 잔액 방지
CHECK (locked >= 0)
```

### 7.2 Order 제약

```sql
CHECK (quantity > 0 OR quote_amount > 0)       -- 둘 중 하나 필수
CHECK (type != 'LIMIT' OR price > 0)           -- 지정가는 가격 필수
CHECK (type != 'MARKET' OR side != 'BUY' OR quote_amount > 0)   -- 시장가 매수는 금액 필수
CHECK (type != 'MARKET' OR side != 'SELL' OR quantity > 0)      -- 시장가 매도는 수량 필수
```

### 7.3 Trade 제약

```sql
CHECK (price > 0)
CHECK (quantity > 0)
CHECK (buy_order_id != sell_order_id)   -- 동일 주문 체결 방지
CHECK (buyer_id != seller_id)           -- 자기 체결 방지
```

---

## 8. MVP 범위

### 현재 포함

- [x] 지정가 주문 (LIMIT-GTC)
- [x] 시장가 주문 (MARKET-IOC)
- [x] **오더북** (인메모리, 매칭 엔진 포함)
- [x] 가격-시간 우선순위 체결
- [x] 부분 체결
- [x] 자기 체결 방지
- [x] 잔액 락/언락/정산
- [x] 원장 기록
- [x] 기본 입출금 (한도 적용)
- [x] 실시간 시세/호가/체결 (WebSocket)

### 향후 고려

- [ ] FOK 주문
- [ ] 예약 주문
- [ ] 메이커/테이커 수수료 차등
- [ ] 24시간 출금 지연
- [ ] 트래블룰