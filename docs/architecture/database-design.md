---
작성일: 2026-02-23
버전: 1.0
목적: 데이터 모델, 테이블 설계, 인덱스 전략, 무결성 제약조건을 정의한다
---

# 데이터베이스 설계

> 이 문서는 Data Architecture의 **구현 상세**를 다룹니다.
> 데이터 정합성/동시성의 원칙은 [architecture.md §3](architecture.md)을 참고하세요.

---

## 1. 개요

### 1.1 기술 환경

| 항목 | 값 |
|------|-----|
| DBMS | MySQL 8.0+ |
| 문자셋 | utf8mb4 / utf8mb4_unicode_ci |
| 엔진 | InnoDB |
| 타임스탬프 | TIMESTAMP(6) — 마이크로초 정밀도 |
| 금액 타입 | DECIMAL(30,18) — 암호화폐 소수점 대응 |

### 1.2 테이블 목록

| 테이블 | 바운디드 컨텍스트 | 설명 |
|--------|------------------|---------|
| member | member | 회원 정보 | 
| member_security | member | 회원 보안 정보 | 
| coin | market | 코인/자산 정보 | 
| market | market | 마켓/거래쌍 | 
| wallet | wallet | 지갑 | 
| balance | wallet | 잔액 (가용/동결) |
| order | trading | 주문 | 
| trade | trading | 체결 | 
| balance_history | wallet | 잔액 변동 이력 (원장) |
| bank_account | transfer | 은행 계좌 | 
| transaction | transfer | 입출금 내역 | 

---

## 2. ERD

### 2.1 엔티티 관계

```
MEMBER ──1:1── MEMBER_SECURITY
MEMBER ──1:1── WALLET
MEMBER ──1:N── BANK_ACCOUNT
MEMBER ──1:N── ORDER
MEMBER ──1:N── TRANSACTION

WALLET ──1:N── BALANCE
BALANCE ──N:1── COIN

COIN ──1:N── MARKET (base_coin)
COIN ──1:N── MARKET (quote_coin)

MARKET ──1:N── ORDER
MARKET ──1:N── TRADE

ORDER ──1:N── TRADE (buy_order)
ORDER ──1:N── TRADE (sell_order)

BALANCE ──1:N── BALANCE_HISTORY

TRANSACTION ──N:1── COIN
TRANSACTION ──N:0..1── BANK_ACCOUNT (원화 입출금 시)
```

### 2.2 컨텍스트 간 참조

컨텍스트 간에는 FK가 아닌 **ID 참조**로 연결한다.
동일 컨텍스트 내에서만 FK 제약조건을 사용한다.

| 참조 | 방식 | 이유 |
|------|------|------|
| order.member_id → member.id | ID 참조 (FK 없음) | trading ↔ member 컨텍스트 분리 |
| trade.buyer_id → member.id | ID 참조 (FK 없음) | trading ↔ member (비정규화) |
| trade.seller_id → member.id | ID 참조 (FK 없음) | trading ↔ member (비정규화) |
| balance_history.reference_id | ID 참조 (FK 없음) | 다형적 참조 (ORDER, TRADE, TX) |
| wallet.member_id → member.id | ID 참조 (FK 없음) | wallet ↔ member 컨텍스트 분리 |
| balance.coin_id → coin.id | ID 참조 (FK 없음) | wallet ↔ market 컨텍스트 분리 |
| order.market_id → market.id | ID 참조 (FK 없음) | trading ↔ market 컨텍스트 분리 |
| trade.market_id → market.id | ID 참조 (FK 없음) | trading ↔ market 컨텍스트 분리 |
| transaction.member_id → member.id | ID 참조 (FK 없음) | transfer ↔ member 컨텍스트 분리 |
| transaction.coin_id → coin.id | ID 참조 (FK 없음) | transfer ↔ market 컨텍스트 분리 |

---

## 3. 테이블 상세

### 3.1 Member 컨텍스트

#### member (회원)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK, AUTO_INCREMENT | |
| uuid | VARCHAR(36) | UK | 외부 노출용 식별자 |
| email_encrypted | VARCHAR(512) | NOT NULL | AES 암호화된 이메일 |
| email_hash | VARCHAR(64) | UK | 로그인 검색용 SHA-256 해시 |
| password_hash | VARCHAR(255) | NOT NULL | BCrypt 해시된 비밀번호 |
| nickname | VARCHAR(50) | UK | 닉네임 (공개 표시 — 평문) |
| phone_encrypted | VARCHAR(512) | NULLABLE | AES 암호화된 연락처 |
| status | ENUM | NOT NULL, DEFAULT 'PENDING' | PENDING → ACTIVE → SUSPENDED → WITHDRAWN |
| kyc_status | ENUM | NOT NULL, DEFAULT 'NONE' | NONE → PENDING → APPROVED / REJECTED |
| kyc_level | TINYINT | NOT NULL, DEFAULT 0 | 0:미인증, 1:휴대폰, 2:계좌, 3:신분증 |
| created_at | TIMESTAMP(6) | NOT NULL | |
| updated_at | TIMESTAMP(6) | NOT NULL | |
| deleted_at | TIMESTAMP(6) | NULLABLE | 소프트 삭제 |

인덱스: `idx_member_status (status)`, `idx_member_created_at (created_at)`

개인정보 암호화 원칙:

| 필드 | 방식 | 이유 |
|------|------|------|
| password | 단방향 해시 (BCrypt) | 복호화 필요 없음 — 비교만 수행 |
| email | 양방향 암호화 (AES) + 검색용 해시 (SHA-256) | PII + 로그인 시 `WHERE email_hash = ?`로 검색 |
| phone | 양방향 암호화 (AES) | PII, 본인인증 시에만 복호화 |
| nickname | 평문 | 다른 사용자에게 공개 표시되는 정보 |

로그인 흐름:

```
[사용자 입력: email]
    │
    ▼
[SHA-256 해시] → email_hash로 member 조회
    │
    ▼
[BCrypt 비교] → password_hash와 비교
    │
    ▼
[로그인 성공] → email_encrypted를 AES 복호화하여 응답에 포함
```

#### member_security (회원 보안)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| member_id | BIGINT | UK, FK | member 1:1 |
| pin_hash | VARCHAR(255) | NULLABLE | PIN 해시 |
| two_factor_enabled | BOOLEAN | DEFAULT false | 2FA 활성화 |
| two_factor_secret | VARCHAR(512) | NULLABLE | 2FA 비밀키 (AES 암호화) |
| last_login_at | TIMESTAMP(6) | NULLABLE | |
| last_login_ip | VARCHAR(45) | NULLABLE | IPv6 대응 |
| login_fail_count | INT | DEFAULT 0 | |
| locked_until | TIMESTAMP(6) | NULLABLE | 계정 잠금 해제 시각 |
| created_at | TIMESTAMP(6) | NOT NULL | |
| updated_at | TIMESTAMP(6) | NOT NULL | |

### 3.2 Market 컨텍스트

#### coin (코인/자산)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| symbol | VARCHAR(20) | UK | BTC, ETH, KRW |
| name | VARCHAR(100) | NOT NULL | Bitcoin, Ethereum |
| name_ko | VARCHAR(100) | NULLABLE | 비트코인, 이더리움 |
| decimal_places | TINYINT | NOT NULL | 소수점 자릿수 |
| min_withdrawal | DECIMAL(30,18) | NOT NULL | 최소 출금 수량 |
| withdrawal_fee | DECIMAL(30,18) | NOT NULL | 출금 수수료 |
| is_active | BOOLEAN | DEFAULT true | 활성 상태 |
| created_at | TIMESTAMP(6) | NOT NULL | |
| updated_at | TIMESTAMP(6) | NOT NULL | |

#### market (마켓/거래쌍)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| symbol | VARCHAR(20) | UK | KRW-BTC, KRW-ETH |
| base_coin_id | BIGINT | NOT NULL | 기준 자산 (BTC) |
| quote_coin_id | BIGINT | NOT NULL | 결제 자산 (KRW) |
| tick_size | DECIMAL(30,18) | NOT NULL | 호가 단위 |
| min_order_quote | DECIMAL(30,18) | NOT NULL | 최소 주문 금액 (quote 기준) |
| maker_fee_rate | DECIMAL(10,8) | NOT NULL | 메이커 수수료율 |
| taker_fee_rate | DECIMAL(10,8) | NOT NULL | 테이커 수수료율 |
| is_active | BOOLEAN | DEFAULT true | 거래 가능 상태 |
| created_at | TIMESTAMP(6) | NOT NULL | |
| updated_at | TIMESTAMP(6) | NOT NULL | |

인덱스: `idx_market_coins (base_coin_id, quote_coin_id)`

### 3.3 Wallet 컨텍스트

#### wallet (지갑)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| member_id | BIGINT | UK | 회원당 1개 지갑 |
| uuid | VARCHAR(36) | UK | 외부 노출용 |
| created_at | TIMESTAMP(6) | NOT NULL | |
| updated_at | TIMESTAMP(6) | NOT NULL | |

#### balance (잔액)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| wallet_id | BIGINT | NOT NULL | |
| coin_id | BIGINT | NOT NULL | |
| available | DECIMAL(30,18) | NOT NULL, DEFAULT 0 | 가용 잔액 |
| locked | DECIMAL(30,18) | NOT NULL, DEFAULT 0 | 동결 잔액 |
| version | BIGINT | NOT NULL, DEFAULT 0 | 동시성 제어용 |
| created_at | TIMESTAMP(6) | NOT NULL | |
| updated_at | TIMESTAMP(6) | NOT NULL | |

제약조건:
- `UK: (wallet_id, coin_id)` — 지갑당 코인별 1개 잔액
- `CHECK: available >= 0`
- `CHECK: locked >= 0`

#### balance_history (원장)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| balance_id | BIGINT | NOT NULL | |
| member_id | BIGINT | NOT NULL | 비정규화 — 조회 성능 |
| coin_id | BIGINT | NOT NULL | 비정규화 — 조회 성능 |
| type | ENUM | NOT NULL | DEPOSIT, WITHDRAWAL, BUY, SELL, FEE, LOCK, UNLOCK, CANCEL_REFUND |
| amount | DECIMAL(30,18) | NOT NULL | 변동 금액 (항상 양수, type으로 방향 표현) |
| available_before | DECIMAL(30,18) | NOT NULL | |
| available_after | DECIMAL(30,18) | NOT NULL | |
| locked_before | DECIMAL(30,18) | NOT NULL | |
| locked_after | DECIMAL(30,18) | NOT NULL | |
| idempotency_key | VARCHAR(100) | UK | 멱등성 보장 |
| reference_type | VARCHAR(50) | NULLABLE | ORDER, TRADE, TRANSACTION |
| reference_id | BIGINT | NULLABLE | 참조 엔티티 ID |
| created_at | TIMESTAMP(6) | NOT NULL | |

인덱스: `idx_balance_history (member_id, coin_id, created_at DESC)`

검증 규칙: `현재 잔액 = SUM(원장 기록)` — available과 locked의 before/after 체인이 연속적이어야 함

### 3.4 Trading 컨텍스트

#### order (주문)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| uuid | VARCHAR(36) | UK | 외부 노출용 |
| member_id | BIGINT | NOT NULL, INDEX | |
| market_id | BIGINT | NOT NULL | |
| side | ENUM('BUY','SELL') | NOT NULL | 매수/매도 |
| type | ENUM('LIMIT','MARKET') | NOT NULL | 지정가/시장가 |
| time_in_force | ENUM('GTC','IOC') | NOT NULL | GTC: 잔여 대기, IOC: 잔여 즉시 취소 |
| status | ENUM('OPEN','FILLED','CANCELLED') | NOT NULL, DEFAULT 'OPEN' | |
| price | DECIMAL(30,18) | NULLABLE | LIMIT 주문의 지정가 (MARKET은 NULL) |
| quantity | DECIMAL(30,18) | NULLABLE | 주문 수량 (MARKET-BUY는 NULL) |
| quote_amount | DECIMAL(30,18) | NULLABLE | MARKET-BUY의 총 주문 금액 |
| filled_quantity | DECIMAL(30,18) | NOT NULL, DEFAULT 0 | 체결된 수량 |
| remaining_quantity | DECIMAL(30,18) | NULLABLE | 미체결 잔여 수량 |
| filled_amount | DECIMAL(30,18) | NOT NULL, DEFAULT 0 | 체결된 총 금액 (quote 기준) |
| created_at | TIMESTAMP(6) | NOT NULL | 주문 시각 = 시간 우선순위 |
| updated_at | TIMESTAMP(6) | NOT NULL | |

제약조건:
- LIMIT: `price IS NOT NULL AND quantity IS NOT NULL AND quote_amount IS NULL`
- MARKET-BUY: `price IS NULL AND quantity IS NULL AND quote_amount IS NOT NULL`
- MARKET-SELL: `price IS NULL AND quantity IS NOT NULL AND quote_amount IS NULL`
- `CHECK: filled_quantity >= 0`
- `CHECK: remaining_quantity >= 0 OR remaining_quantity IS NULL`

인덱스 — 아래 §4.2 참고

#### trade (체결)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| uuid | VARCHAR(36) | UK | |
| market_id | BIGINT | NOT NULL | |
| buy_order_id | BIGINT | NOT NULL | 매수 주문 |
| sell_order_id | BIGINT | NOT NULL | 매도 주문 |
| buyer_id | BIGINT | NOT NULL | 비정규화 — 조회 성능 |
| seller_id | BIGINT | NOT NULL | 비정규화 — 조회 성능 |
| price | DECIMAL(30,18) | NOT NULL | 체결가 (maker 주문의 price) |
| quantity | DECIMAL(30,18) | NOT NULL | 체결 수량 |
| quote_amount | DECIMAL(30,18) | NOT NULL | 체결 금액 (price × quantity) |
| buyer_fee | DECIMAL(30,18) | NOT NULL | 매수자 수수료 |
| seller_fee | DECIMAL(30,18) | NOT NULL | 매도자 수수료 |
| created_at | TIMESTAMP(6) | NOT NULL | |

인덱스:
- `idx_trade_market_time (market_id, created_at DESC)` — 마켓별 체결 내역
- `idx_trade_buyer (buyer_id, created_at DESC)` — 매수자별 체결 내역
- `idx_trade_seller (seller_id, created_at DESC)` — 매도자별 체결 내역

### 3.5 Transfer 컨텍스트 (Phase 2)

#### bank_account (은행 계좌)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| member_id | BIGINT | NOT NULL, INDEX | |
| bank_code | VARCHAR(10) | NOT NULL | 은행 코드 |
| account_no | VARCHAR(512) | NOT NULL | 계좌번호 (AES 암호화) |
| holder_name | VARCHAR(512) | NOT NULL | 예금주 (AES 암호화) |
| is_primary | BOOLEAN | DEFAULT false | 대표 계좌 여부 |
| verified_at | TIMESTAMP(6) | NULLABLE | 인증 완료 시각 |
| created_at | TIMESTAMP(6) | NOT NULL | |
| deleted_at | TIMESTAMP(6) | NULLABLE | |

#### transaction (입출금)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK | |
| uuid | VARCHAR(36) | UK | |
| member_id | BIGINT | NOT NULL | |
| coin_id | BIGINT | NOT NULL | |
| bank_account_id | BIGINT | NULLABLE | 원화 입출금 시 |
| type | ENUM('DEPOSIT','WITHDRAWAL') | NOT NULL | |
| status | ENUM('PENDING','COMPLETED','FAILED','CANCELLED') | NOT NULL | |
| amount | DECIMAL(30,18) | NOT NULL | 요청 금액 |
| fee | DECIMAL(30,18) | NOT NULL, DEFAULT 0 | 수수료 |
| tx_hash | VARCHAR(255) | NULLABLE | 블록체인 트랜잭션 해시 |
| network | VARCHAR(50) | NULLABLE | 네트워크 (Ethereum, Bitcoin 등) |
| created_at | TIMESTAMP(6) | NOT NULL | |
| updated_at | TIMESTAMP(6) | NOT NULL | |
| completed_at | TIMESTAMP(6) | NULLABLE | |

인덱스: `idx_tx_member_type (member_id, type, created_at DESC)`

---

## 4. 오더북과 DB의 관계

### 4.1 설계 원칙

오더북은 **인메모리 자료구조**로 동작하고, DB는 **영속성 계층**으로 사용한다.

```
[주문 API] → [Order 테이블에 저장] → [인메모리 오더북에 추가] → [매칭]
                                                                    │
                                                          [Trade 테이블에 저장]
                                                          [Order 상태 업데이트]
                                                          [Balance 정산]
```

| 역할 | 인메모리 오더북 | DB (order 테이블) |
|------|---------------|------------------|
| 목적 | 빠른 매칭 수행 | 영속성, 감사 추적 |
| 포함 범위 | OPEN 상태의 LIMIT-GTC 주문만 | 모든 주문 (OPEN, FILLED, CANCELLED) |
| 정렬 기준 | price-time priority | 인덱스 기반 |
| 정합성 기준 | DB가 원본(source of truth) | — |

### 4.2 오더북 지원 인덱스

오더북 복구 시 OPEN 상태의 주문을 가격-시간 순서로 로딩해야 한다.

```sql
-- 매수 오더북 복구: 높은 가격 우선, 같은 가격이면 먼저 들어온 주문 우선
CREATE INDEX idx_order_book_buy 
ON `order` (market_id, side, status, price DESC, created_at ASC)
WHERE side = 'BUY' AND status = 'OPEN';

-- 매도 오더북 복구: 낮은 가격 우선, 같은 가격이면 먼저 들어온 주문 우선
CREATE INDEX idx_order_book_sell
ON `order` (market_id, side, status, price ASC, created_at ASC)
WHERE side = 'SELL' AND status = 'OPEN';
```

> MySQL은 partial index를 지원하지 않으므로, 실제로는 복합 인덱스로 구현한다:

```sql
CREATE INDEX idx_order_matching
ON `order` (market_id, side, status, price, created_at);
```

매칭 쿼리 예시 — 매수 주문이 들어왔을 때 매도 오더북에서 대상 조회:

```sql
SELECT * FROM `order`
WHERE market_id = ?
  AND side = 'SELL'
  AND status = 'OPEN'
  AND price <= ?           -- taker의 매수 가격 이하
ORDER BY price ASC, created_at ASC
LIMIT 10;
```

### 4.3 서버 재시작 시 오더북 복구

서버 재시작 시 DB에서 OPEN 상태의 주문을 읽어 인메모리 오더북을 재구성한다.

```
[서버 시작]
    │
    ▼
[market 목록 로딩]
    │
    ▼ 마켓별 반복
[OPEN 주문 조회] ── SELECT * FROM `order` 
    │                WHERE market_id = ? AND status = 'OPEN'
    │                ORDER BY price, created_at
    ▼
[인메모리 오더북에 삽입]
    │
    ▼
[매칭 엔진 준비 완료]
```

복구 대상: `status = 'OPEN'`이고 `type = 'LIMIT'`인 주문만.
MARKET 주문은 오더북에 들어가지 않으므로 복구 대상이 아니다.

### 4.4 데이터 일관성 보장

주문 생성과 오더북 추가가 원자적으로 일어나야 한다.

```
[트랜잭션 시작]
    │
    ├── Balance 락 (available → locked)
    ├── Order INSERT (status = 'OPEN')
    ├── BalanceHistory INSERT (type = 'LOCK')
    │
[트랜잭션 커밋]
    │
    ▼
[인메모리 오더북에 추가] ← 커밋 성공 후에만 실행
    │
    ▼
[매칭 시도]
```

커밋 실패 시 오더북에 추가되지 않으므로 정합성이 유지된다.
오더북에 추가된 뒤 매칭 결과도 동일한 패턴으로 DB에 반영한다.

---

## 5. 인덱스 전략

### 5.1 인덱스 설계 원칙

- 조회 패턴에 맞춘 **복합 인덱스** 우선
- 쓰기 성능을 고려하여 불필요한 인덱스 최소화
- 카디널리티가 높은 컬럼을 앞에 배치

### 5.2 주요 인덱스 목록

| 테이블 | 인덱스 | 용도 |
|--------|--------|------|
| member | `uk_member_email` | 로그인 조회 |
| balance | `uk_balance_wallet_coin (wallet_id, coin_id)` | 잔액 조회/업데이트 |
| balance_history | `idx_balance_history (member_id, coin_id, created_at DESC)` | 원장 조회 |
| balance_history | `uk_balance_history_idempotency (idempotency_key)` | 멱등성 보장 |
| order | `idx_order_matching (market_id, side, status, price, created_at)` | 오더북 매칭/복구 |
| order | `idx_order_member (member_id, status, created_at DESC)` | 내 주문 조회 |
| trade | `idx_trade_market_time (market_id, created_at DESC)` | 마켓별 체결 내역 |
| trade | `idx_trade_buyer (buyer_id, created_at DESC)` | 내 체결 내역 |
| trade | `idx_trade_seller (seller_id, created_at DESC)` | 내 체결 내역 |

---

## 6. 초기 데이터 (Seed)

```sql
-- 코인
INSERT INTO coin (symbol, name, name_ko, decimal_places, min_withdrawal, withdrawal_fee, is_active)
VALUES 
  ('KRW', 'Korean Won', '원화', 0, 5000, 0, true),
  ('BTC', 'Bitcoin', '비트코인', 8, 0.0005, 0.0001, true),
  ('ETH', 'Ethereum', '이더리움', 8, 0.01, 0.005, true);

-- 마켓
INSERT INTO market (symbol, base_coin_id, quote_coin_id, tick_size, min_order_quote, maker_fee_rate, taker_fee_rate, is_active)
VALUES 
  ('KRW-BTC', 2, 1, 1000, 5000, 0.0005, 0.0005, true),
  ('KRW-ETH', 3, 1, 100, 5000, 0.0005, 0.0005, true);
```

---

## 7. 확장 고려사항

### 7.1 파티셔닝 (Phase 2 이후)

| 테이블 | 파티셔닝 전략 | 기준 |
|--------|-------------|------|
| trade | 시간 기반 (월별) | created_at |
| balance_history | 시간 기반 (월별) | created_at |
| order | 상태 기반 | OPEN vs FILLED/CANCELLED 분리 |

### 7.2 유용한 뷰

```sql
-- 회원별 총 자산 조회
CREATE VIEW v_member_total_balance AS
SELECT m.id AS member_id, m.nickname,
       c.symbol AS coin_symbol,
       b.available, b.locked,
       (b.available + b.locked) AS total
FROM member m
JOIN wallet w ON m.id = w.member_id
JOIN balance b ON w.id = b.wallet_id
JOIN coin c ON b.coin_id = c.id
WHERE m.deleted_at IS NULL;

-- 활성 주문 조회 (오더북 대상)
CREATE VIEW v_active_orders AS
SELECT o.id, o.uuid, o.member_id,
       mk.symbol AS market_symbol,
       o.side, o.type, o.status,
       o.price, o.quantity, o.remaining_quantity,
       o.filled_quantity, o.created_at
FROM `order` o
JOIN market mk ON o.market_id = mk.id
WHERE o.status = 'OPEN';
```