---
작성일: 2026-02-21
수정일: 2026-02-23
버전: 1.2
목적: 시스템의 전체 구조를 Application, Data, Technology 아키텍처로 나누어 정의한다
---

# 아키텍처

> 이 문서는 시스템의 **현재 구조**를 정의합니다.
> 구조적 결정의 근거는 [결정 기록(ADL)](../decisions/README.md)을 참고하세요.

---

## 1. 개요

### 1.1 아키텍처 요약

**명칭**: 모듈러 모놀리스 (DDD 스타일 계층형 아키텍처)

모놀리식으로 빠른 개발을 목표로 하되, 바운디드 컨텍스트를 명확하게 하여
추후 서비스 분리에 대비할 수 있는 구조를 선택했다.

- 단일 배포 단위 (모놀리스)
- 도메인 경계 기반 모듈 분리 (바운디드 컨텍스트)
- 3-레이어 구조 (Application, Domain, System)
- MSA 전환 준비 (컨텍스트별 분리 가능)

> 선택 근거: [ADR-0001 모듈러 모놀리스 아키텍처 선택](../decisions/ADR-0001-modular-monolith.md)

### 1.2 아키텍처 목표

| 목표 | 설명 |
|------|------|
| **자산 정합성 우선** | 잔고/락/원장 데이터의 무결성을 최우선으로 보장한다 |
| **명확한 책임 분리** | 도메인, 유스케이스, 입출력 경계를 분리한다 |
| **점진적 확장** | 단일 애플리케이션에서 이벤트/비동기/서비스 분리로 확장 가능해야 한다 |
| **장애 격리** | 특정 구성요소 장애가 전체 서비스 중단으로 확산되지 않도록 한다 |

### 1.3 문서 구조

이 문서는 EA(Enterprise Architecture) 프레임워크의 4개 영역을 따른다.

| EA 영역 | 다루는 것 | 본 문서 위치 |
|---------|----------|-------------|
| **Business Architecture** | 비즈니스 프로세스, 규칙 | 별도 문서: [domain/](../domain/) 폴더 |
| **Application Architecture** | 모듈 구조, 레이어, 컨텍스트 간 관계 | §2 |
| **Data Architecture** | 데이터 정합성 원칙, 동시성 제어 | §3 (설계 상세는 [database-design.md](database-design.md)) |
| **Technology Architecture** | 기술 스택, 인프라, 확장 전략 | §4 |

---

## 2. Application Architecture

> 모듈 구조, 레이어 원칙, 도메인 경계, 컨텍스트 간 통신을 정의한다.

### 2.1 패키지 구조

```
com.tikkeul.exchange/
├── application/        → 유스케이스 오케스트레이션, HTTP 입출력 변환
│   ├── member/
│   ├── wallet/
│   ├── trading/         → 주문, 매칭, 체결
│   ├── transfer/
│   └── market/
├── domain/             → 비즈니스 규칙, 불변식, 엔티티
│   ├── member/
│   ├── wallet/
│   ├── trading/         → Order, Trade, OrderBook
│   ├── transfer/
│   └── market/
└── system/             → 인프라, 공통 설정
    ├── config/
    ├── exception/
    ├── security/
    ├── redis/
    └── websocket/      → WebSocket 핸들러, 시세/호가 배포
```

### 2.2 레이어 원칙

#### 레이어별 책임

| 레이어 | 책임 | 포함 요소 |
|--------|------|-----------|
| **domain** | 비즈니스 규칙과 불변식. 외부 기술에 의존하지 않는다 | Entity, Repository(인터페이스), 도메인 타입 |
| **application** | 유스케이스 오케스트레이션, 트랜잭션 경계, DTO 변환 | Controller, Service, DTO, Request/Response |
| **system** | 저장소, 캐시, 보안 등 기술 구현 | Config, JPA 구현, Redis, Security |

#### 의존성 방향

```
application ──────► domain ◄────── system
     │                                │
     └────────────► system ◄──────────┘
                (설정/유틸만)
```

- domain은 어떤 레이어에도 의존하지 않는다 (최하위)
- application과 system은 domain에 의존한다
- application은 system의 설정/유틸만 사용한다

### 2.3 도메인 경계

#### 바운디드 컨텍스트

| Context | 책임 | 핵심 도메인 |
|---------|------|------------|
| **trading** | 주문, 오더북, 매칭, 체결 | Order, Trade, OrderBook, MatchingEngine |
| **wallet** | 가용/락 잔고, 정산, 원장 기록 | Wallet, Balance, BalanceHistory |
| **member** | 사용자 식별 및 거래 가능 상태 | Member, MemberSecurity |
| **market** | 자산 메타데이터, 마켓 정보 | Coin, Market |
| **transfer** | 입출금 처리 | Transaction, BankAccount |

#### 컨텍스트 간 관계

```
        ┌──────────┐
        │  member  │
        └────┬─────┘
             │ 인증된 사용자
             ▼
┌──────────────────────────────────────┐
│                                      │
▼                                      ▼
┌──────────┐    잔액 락/정산    ┌──────────┐
│ trading  │◄──────────────────►│  wallet  │
└────┬─────┘                    └────▲─────┘
     │                               │
     │ 마켓 정보 조회                  │ 잔액 변경
     ▼                               │
┌──────────┐                   ┌─────┴────┐
│  market  │                   │ transfer │
└──────────┘                   └──────────┘
```

#### 컨텍스트 간 통신 규칙

1. **다른 컨텍스트의 domain 직접 참조 금지** — Entity를 직접 import하지 않음. ID로 참조하거나 DTO로 통신
2. **Service 레이어를 통해서만 통신** — Repository 직접 호출 금지
3. **순환 의존 금지** — trading → wallet (O), wallet → trading (X)

### 2.4 외부 시스템 통합

코어 도메인은 외부 시스템 구현 세부사항에 결합하지 않는다.
인터페이스를 통해 통합하고, MVP에서는 Mock 구현을 사용한다.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Transfer   │     │  BankClient  │     │   External   │
│   Service    │────►│ (Interface)  │────►│   Bank API   │
└──────────────┘     └──────────────┘     └──────────────┘
                           ▲
                           │ 구현
                     ┌─────┴─────┐
                     │ MockBank  │  ← MVP에서 사용
                     │ Client    │
                     └───────────┘
```

---

## 3. Data Architecture

> 데이터 정합성, 트랜잭션 경계, 동시성 제어의 **원칙**을 정의한다.
> 구현 상세(ERD, DDL, 인덱스)는 [database-design.md](database-design.md)를 참고한다.

### 3.1 트랜잭션 경계

쓰기 트랜잭션은 정합성의 최종 경계다. write 단계에서 최종 검증이 가능해야 한다.

```java
@Transactional
public Order createOrder(OrderRequest request) {
    // 1. Read: 잔액 조회 (검증)
    Balance balance = balanceRepository.findByWalletAndCoin(...);

    // 2. Write: 잔액 락 (최종 검증 + 쓰기)
    balance.lock(amount);  // 내부에서 available >= amount 재검증

    // 3. Write: 주문 생성
    return orderRepository.save(order);
}
```

### 3.2 원장 원칙

- 모든 잔액 변동은 반드시 원장(BalanceHistory)에 기록
- 원장 기록과 잔액 업데이트는 **동일 트랜잭션** 내에서 처리
- `현재 잔액 = SUM(원장 기록)`으로 검증 가능해야 함

### 3.3 멱등성 원칙

- 요청 키(idempotency_key) 기준으로 처리
- DB 제약조건(UNIQUE)을 최종 방어선으로 유지

### 3.4 동시성 제어

#### 기본 원칙

경합은 애플리케이션 선검증만으로 해결하지 않는다.
최종 쓰기 경계에서 **제약조건/락/버전 기반**으로 충돌을 제어한다.

#### 후보 방식

| 방식 | 메커니즘 | 적합 상황 |
|------|---------|----------|
| 낙관적 락 | JPA @Version, 충돌 시 재시도 | 경합 빈도 낮음 |
| 비관적 락 | SELECT FOR UPDATE | 경합 빈도 높음, 순서 보장 필요 |
| 분산 락 | Redis (Redisson) | 다중 인스턴스 환경 |

> 구체적인 방식은 구현 단계에서 결정 예정

#### 오더북 동시성

매칭/정산과 같이 순서가 중요한 흐름은 중복 실행 방지 정책을 가져야 한다.
현재는 단일 스레드로 처리하여 동시성 문제를 원천 차단한다.
다중 인스턴스 환경으로 확장 시 Redis 분산 락이나 메시지 큐 순서 보장을 고려할 수 있다.

---

## 4. Technology Architecture

> 기술 스택, 인프라 구성, 확장 전략을 정의한다.

### 4.1 기술 스택

| 구분 | 기술 | 버전 | 선택 근거 |
|------|------|------|-----------|
| Language | Java | 21 | 금융 IT 취업 목표, Virtual Threads |
| Framework | Spring Boot | 3.x | 안정적인 생태계 |
| Build | Gradle | Kotlin DSL | |
| DB | MySQL | 8.0+ | 금융권 널리 사용, 트랜잭션 지원 |
| ORM | JPA (Hibernate) | | |
| Cache | Redis | | 로그아웃 토큰 관리 |
| WebSocket | Spring WebSocket | | 실시간 시세/호가/체결 배포 |
| API Docs | SpringDoc (Swagger) | | |
| Test | JUnit5 + Mockito | | |
| Message Queue | — | 현재 미사용 | 확장 시 Kafka/RabbitMQ 고려 |

### 4.2 확장 시 고려할 수 있는 방향

현재는 단일 애플리케이션, 동기 처리, 단일 DB로 운영한다.
트래픽이나 조직 규모가 커질 경우 아래 방향을 고려할 수 있다.

- **이벤트 기반 전환**: 주문 → 매칭 → 정산 → 알림을 비동기 파이프라인으로 분리. 메시지 큐 도입.
- **서비스 분리**: 컨텍스트 단위로 독립 서비스 분리. DB 분리, SAGA 패턴.

### 4.3 MSA 전환 기준

| 기준 | 설명 |
|------|------|
| 독립적 스케일링 필요 | 특정 컨텍스트만 트래픽 급증 |
| 팀 규모 확대 | 독립 배포 필요 |
| 기술 스택 다양화 | 컨텍스트별 다른 기술 필요 |

### 4.4 전환 방법 (Strangler Fig Pattern)

가장 독립적인 컨텍스트부터 분리 (예: market) → API Gateway로 라우팅 → 점진적으로 다른 컨텍스트 분리