---
status: "accepted"
date: 2026-02-23
---

# ADR-0004: 멱등성 키 기반 원장 중복 방지

## Context

원장(BalanceHistory)은 모든 잔액 변동의 감사 추적 기록이다.
네트워크 재시도, 서버 장애 복구, 동시 요청 등으로 동일한 잔액 변동이
중복 기록될 위험이 있다. 중복 기록은 잔액 불일치로 이어진다.

## Decision Drivers

- 동일 원인에 의한 원장 중복 INSERT를 확실하게 방지해야 함
- 애플리케이션 레벨 체크만으로는 동시 요청을 완전히 방지할 수 없음
- `현재 잔액 = SUM(원장 기록)` 검증이 항상 성립해야 함
- 재시도 안전(retry-safe)한 설계여야 함

## Considered Options

1. **idempotency_key + DB UNIQUE 제약조건** — 원장 레코드에 고유 키를 부여하고 DB에서 강제
2. **애플리케이션 레벨 중복 체크** — INSERT 전에 SELECT로 존재 여부 확인
3. **Redis 기반 멱등성 캐시** — 처리 완료된 키를 Redis에 기록하고 확인

## Decision

**Chosen option: "idempotency_key + DB UNIQUE 제약조건"**, because DB 레벨에서 중복을 강제하는 것이 가장 확실한 방어선이다.

```sql
ALTER TABLE balance_history
ADD UNIQUE KEY uk_balance_history_idempotency (idempotency_key);
```

키 생성 규칙 — `{이벤트 유형}:{원인 ID}:{동작}:{자산 ID}`:

```
주문 락:     ORDER:{orderId}:LOCK:{coinId}
체결 정산:   TRADE:{tradeId}:BUY_SETTLE:{coinId}
체결 수령:   TRADE:{tradeId}:BUY:{coinId}
수수료:      TRADE:{tradeId}:FEE:{coinId}
잔여 반환:   ORDER:{orderId}:REFUND:{coinId}
```

하나의 Trade에서 매수자/매도자 각각에 대해 여러 BalanceHistory가 생기지만,
coinId와 동작(BUY_SETTLE/BUY/FEE 등)이 다르므로 키가 겹치지 않는다.

## Consequences

### Positive

- DB UNIQUE 제약조건으로 동시 요청도 확실하게 방지 (최종 방어선)
- 재시도 시 중복 INSERT가 실패하므로 안전하게 무시 가능
- 키 형식이 규칙적이라 원인 추적이 용이 (어떤 주문/체결에 의한 변동인지)
- reference_type + reference_id와 함께 사용하면 완전한 감사 추적

### Negative

- 키 생성 규칙을 일관되게 지켜야 함 (개발자 실수 가능)
- UNIQUE 인덱스 추가로 INSERT 성능에 약간의 오버헤드
- 키가 길어질 수 있음 (VARCHAR(100)으로 제한)

## Options Detail

### Option 2: 애플리케이션 레벨 중복 체크

- Good, 구현이 간단 (SELECT → 없으면 INSERT)
- Bad, 동시 요청 시 race condition 발생 (둘 다 SELECT에서 미존재로 확인)
- Bad, DB 제약조건 없이는 최종 방어선이 없음

### Option 3: Redis 기반 멱등성 캐시

- Good, 빠른 체크 가능
- Bad, Redis 장애 시 멱등성 보장 불가
- Bad, Redis와 DB 간 동기화 문제 (Redis에는 있는데 DB에는 없는 상태)
- Neutral, Option 1과 조합하면 보조 수단으로 활용 가능

## References

- 관련 문서: [데이터베이스 설계 §3.3 balance_history](../architecture/database-design.md)
- 관련 문서: [아키텍처 §3.3 멱등성 원칙](../architecture/architecture.md)
- 관련 문서: [오더북 도메인 모델 — BalanceHistory 예시](../domain/orderbook-domain.md)