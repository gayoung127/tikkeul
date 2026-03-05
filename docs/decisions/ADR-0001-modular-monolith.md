---
status: "accepted"
date: 2026-02-21
---

# ADR-0001: 모듈러 모놀리스 아키텍처 선택

## Context

가상자산 거래소 MVP를 개발한다. 개인 프로젝트로 빠른 개발 속도가 필요하며,
거래소 특성상 주문-매칭-정산 간의 강한 트랜잭션 일관성이 요구된다.
초기 단계에서 도메인 경계를 정확히 파악하기 어렵고,
마이크로서비스의 복잡도(분산 트랜잭션, 서비스 간 통신)를 감당하기 어려운 상황이다.

## Decision Drivers

- 개인 프로젝트로 개발 속도가 중요
- 자산 정합성을 위한 트랜잭션 일관성 필수
- 초기에 도메인 경계를 정확히 나누기 어려움
- 추후 서비스 분리 가능성을 열어둬야 함

## Considered Options

1. **모듈러 모놀리스** — 단일 배포 + 바운디드 컨텍스트 기반 내부 모듈화
2. **마이크로서비스 (MSA)** — 컨텍스트별 독립 서비스 + 분산 통신
3. **순수 모놀리스** — 모듈 경계 없는 단일 애플리케이션

## Decision

**Chosen option: "모듈러 모놀리스"**, because 단일 배포의 단순함과 도메인 경계 분리의 장점을 모두 취할 수 있다.

DDD 스타일 계층형 아키텍처를 적용하여, 단일 배포 단위 내에서 바운디드 컨텍스트(trading, wallet, member, market, transfer)를 명확히 분리한다. 컨텍스트 간에는 Service를 통해서만 통신하고, Entity 직접 참조를 금지한다.

## Consequences

### Positive

- 단일 배포로 운영 복잡도가 낮다
- 모듈 간 메서드 호출로 통신이 단순하다 (HTTP/gRPC 불필요)
- 분산 트랜잭션이 불필요하여 자산 정합성 확보가 용이하다
- 이미 모듈화되어 있어 추후 MSA 전환이 용이하다

### Negative

- 모듈 간 경계가 코드 레벨에서만 강제되므로 우회 가능하다
- 단일 프로세스이므로 한 모듈의 장애가 전체에 영향을 줄 수 있다
- 특정 컨텍스트만 독립적으로 스케일링할 수 없다

## Options Detail

### Option 1: 모듈러 모놀리스

- Good, 개발 속도가 빠르다
- Good, 트랜잭션 일관성 확보가 쉽다
- Good, MSA 전환 시 컨텍스트 단위로 분리 가능하다
- Bad, 모듈 경계 우회가 가능하다 (컴파일 타임 강제 불가)

### Option 2: 마이크로서비스

- Good, 독립적 스케일링과 배포가 가능하다
- Good, 기술 스택 다양화가 가능하다
- Bad, 분산 트랜잭션 관리가 복잡하다 (SAGA 등)
- Bad, 개인 프로젝트에서 인프라 운영 부담이 크다
- Bad, 초기에 경계를 잘못 나누면 큰 비용이 발생한다

### Option 3: 순수 모놀리스

- Good, 가장 빠르게 개발할 수 있다
- Bad, 코드가 커지면 복잡도 관리가 어렵다
- Bad, MSA 전환 시 대규모 리팩토링이 필요하다

### 레이어 구조: DDD 스타일 계층형 vs 헥사고날

모듈러 모놀리스 내부 레이어 구조도 함께 결정했다.

| 항목 | 헥사고날 (Port/Adapter) | DDD + 계층형 (현재 선택) |
|------|----------------------|----------------------|
| Port/Adapter 인터페이스 | 필수 | 불필요 — 직접 의존 |
| 파일 수 | 많음 (보일러플레이트) | 적음 |
| 외부 기술 교체 | 용이 | 리팩토링 필요 |
| 학습 곡선 | 높음 | 낮음 |
| 적합 상황 | 대규모 팀, 장기 프로젝트 | 개인, 소규모 프로젝트 |

**DDD 스타일 계층형을 선택한 이유**:
- 개인 프로젝트로 시간 제약이 있어 의존성 분리에 걸리는 시간을 줄여야 함
- 외부 기술 교체 가능성이 낮음 (MySQL, Redis 고정)
- 헥사고날의 이점보다 개발 속도가 더 중요

참고: Kamil Grzybek — "Application, Domain and Infrastructure assemblies could be merged into one assembly. Some people like horizontal layering or more decomposition, some don't."

## References

- Martin Fowler, ["MonolithFirst"](https://martinfowler.com/bliki/MonolithFirst.html) (2015)
- Martin Fowler, ["BoundedContext"](https://martinfowler.com/bliki/BoundedContext.html)
- Kamil Grzybek, ["Modular Monolith with DDD"](https://github.com/kgrzybek/modular-monolith-with-ddd)
- Kamil Grzybek, ["Domain-Centric Design"](https://www.kamilgrzybek.com/blog/posts/modular-monolith-domain-centric-design)
- ["My experience of using modular monolith and DDD architectures"](https://www.thereformedprogrammer.net/my-experience-of-using-modular-monolith-and-ddd-architectures/)
- [MSA 전환 전략 - 바운디드 컨텍스트 활용](https://www.msap.ai/docs/msa-expert-from-concepts-to-practice/implementing-msa/msa-adoption-strategy/msa-migration/)
- Baeldung, ["DDD Bounded Contexts and Java Modules"](https://www.baeldung.com/java-modules-ddd-bounded-contexts)
- 관련 문서: [아키텍처](../architecture/architecture.md)