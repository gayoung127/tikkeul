---
status: "accepted"
date: 2026-02-23
---

# ADR-0003: 가용/동결 잔액 분리 방식

## Context

거래소에서 사용자가 주문을 생성하면 잔액의 일부를 동결(lock)해야 한다.
동결하지 않으면 같은 잔액으로 여러 주문을 동시에 낼 수 있어 초과 체결이 발생한다.
잔액을 어떤 구조로 관리할지 결정해야 한다.

## Decision Drivers

- 주문 시 즉시 동결하여 이중 주문 방지
- 체결/취소 시 정확한 정산 가능
- `총 자산 = available + locked`이 항상 성립해야 함
- 원장(BalanceHistory)과의 정합성 검증이 가능해야 함

## Considered Options

1. **available + locked 분리** — Balance 테이블에 두 컬럼으로 분리 관리
2. **단일 balance + 별도 lock 테이블** — Balance는 총액만, 동결은 주문별로 별도 기록
3. **단일 balance만** — 주문 생성 시 바로 차감, 취소 시 복원

## Decision

**Chosen option: "available + locked 분리"**, because 가장 단순하면서도 정합성 검증이 용이하다.

```sql
CREATE TABLE balance (
    wallet_id   BIGINT,
    coin_id     BIGINT,
    available   DECIMAL(30,18) NOT NULL DEFAULT 0,  -- 즉시 사용 가능
    locked      DECIMAL(30,18) NOT NULL DEFAULT 0,  -- 주문/출금으로 동결
    version     BIGINT NOT NULL DEFAULT 0,
    CHECK (available >= 0),
    CHECK (locked >= 0)
);
```

잔액 변동 흐름:
- 주문 생성: `available -= amount, locked += amount`
- 체결: `locked -= amount` (매도자), `available += received` (매수자)
- 취소: `locked -= amount, available += amount`

## Consequences

### Positive

- 총 자산 = available + locked으로 직관적 검증
- CHECK 제약조건으로 음수 잔액 방지
- BalanceHistory의 before/after 체인과 대조 가능
- 쿼리 한 번으로 가용 잔액과 총 자산 모두 조회 가능

### Negative

- 주문별로 얼마가 동결되었는지 추적하려면 BalanceHistory를 조회해야 함
- 부분 체결 시 locked에서 정확한 금액을 차감하는 로직이 필요

## Options Detail

### Option 2: 단일 balance + 별도 lock 테이블

- Good, 주문별 동결 금액을 개별 추적 가능
- Bad, 가용 잔액 조회 시 `balance - SUM(locks)` 계산 필요 (조인)
- Bad, lock 레코드 생성/삭제 관리가 복잡

### Option 3: 단일 balance만

- Good, 구조가 가장 단순
- Bad, 동결 개념이 없어 이중 주문 방지가 어려움
- Bad, 취소 시 "원래 얼마였는지" 복원 기준이 모호

## References

- 관련 문서: [데이터베이스 설계 §3.3](../architecture/database-design.md)
- 관련 문서: [아키텍처 §3.2 원장 원칙](../architecture/architecture.md)
- 관련 문서: [오더북 도메인 모델 — 지갑 락/정산 규칙](../domain/orderbook-domain.md)