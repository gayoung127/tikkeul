# Architecture Decision Log (ADL)

> 주요 아키텍처/설계 결정을 기록합니다.
> 새로운 결정이 필요할 때 ADR을 작성하고, 이 목록에 추가합니다.

---

## 상태 정의

| 상태 | 의미 |
|------|------|
| 🔄 Proposed | 검토 중 |
| ✅ Accepted | 채택됨 |
| ❌ Rejected | 기각됨 (기각 사유 기록) |
| 🔀 Superseded | 다른 ADR로 대체됨 |

---

## 결정 목록

| ID | 제목 | 상태 |
|----|------|------|
| [ADR-0001](ADR-0001-modular-monolith.md) | 모듈러 모놀리스 아키텍처 선택 (DDD 스타일 계층형) | ✅ Accepted | 
| [ADR-0002](ADR-0002-orderbook-data-structure.md) | 오더북 인메모리 자료구조 선택 | ✅ Accepted | 
| [ADR-0003](ADR-0003-available-locked-balance.md) | 가용/동결 잔액 분리 방식 | ✅ Accepted | 
| [ADR-0004](ADR-0004-idempotency-key-ledger.md) | 멱등성 키 기반 원장 중복 방지 | ✅ Accepted |

<!-- 향후 ADR 후보:
| ADR-0005 | 잔액 동시성 제어 방식 선택 | 🔄 Proposed | 구현 단계에서 결정 |
-->


---

## ADR 작성 가이드

### 작성 절차

1. `ADR-template.md`를 복사하여 새 파일 생성
2. 파일명 규칙: `ADR-{번호 4자리}-{kebab-case 제목}.md`
   - 예: `ADR-0001-modular-monolith.md`
3. status를 `proposed`로 설정하고 내용 작성
4. 결정 확정 시 status를 `accepted`로 변경
5. 이 README의 결정 목록 테이블에 추가

### 템플릿 설명

[ADR-template.md](ADR-template.md)는 Michael Nygard의 서술형 흐름과 MADR의 구조화된 선택지 비교를 결합한 형식입니다.

| 섹션 | 출처 | 필수 | 설명 |
|------|------|------|------|
| Context | Nygard | ✅ | 결정이 필요한 상황 (사실만, 의견 없이) |
| Decision Drivers | MADR | ✅ | 결정에 영향을 미치는 핵심 요인 |
| Considered Options | MADR | ✅ | 검토한 선택지 목록 |
| Decision | Nygard + MADR | ✅ | 선택한 옵션 + 핵심 근거 한 문장 + 세부 서술 |
| Consequences | Nygard + MADR | ✅ | Positive/Negative로 구분 |
| Options Detail | MADR | 선택 | 선택지별 Good/Bad 비교 (복잡한 결정만) |
| References | 공통 | 선택 | 참고 자료, 관련 ADR 링크 |

### 참고

- Michael Nygard, ["Documenting Architecture Decisions"](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions) (2011)
- MADR Project: https://adr.github.io/madr/
- Joel Parker Henderson, [Architecture Decision Record](https://github.com/joelparkerhenderson/architecture-decision-record)