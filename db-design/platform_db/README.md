# platform_db — 문서 허브

여러 서비스가 공유하는 service-agnostic 플랫폼 코어 DB. 설계·결정·학습 문서가 여기 모여 있습니다.

> 전체 소개·학습 경로는 [루트 README](../../README.md) 참고.

> 📄 **설계 문서 — 구현 코드는 없습니다.** 상태 표기는 *설계 성숙도*(✅ 설계 확정 · 🟡 부분 · 🔴 미설계)이며, 코드 블록은 설계 의도·테스트 방법 예시입니다.

## 🚪 진입 순서

1. [core/architecture.md](./core/architecture.md) — 아키텍처 & 규율 (개요·불변식·3-gate)
2. [core/requirements.md](./core/requirements.md) — 무엇을 충족해야 하나 (+ 요구 추적 매트릭스)
3. [core/schema-reference.md](./core/schema-reference.md) — 전체 DDL · ERD · billing · 보안
4. [decisions/](./decisions/) — "왜 이렇게 했나" (비교 + 결정 13편)
5. [explainers/](./explainers/README.md) — 개념을 풀어쓴 학습 문서 38편 (난이도별)
6. [core/bdd-scenarios.md](./core/bdd-scenarios.md) · [core/e2e-journeys.md](./core/e2e-journeys.md) — 행위 검증

## 📁 구성

| 폴더 | 내용 |
|---|---|
| `core/` | architecture · requirements · schema-reference · operability · bdd-scenarios · e2e-journeys |
| `decisions/` | 설계 의사결정 13편 (audit-two-lane, auth-projection, service-extensibility …) |
| `explainers/` | 학습 문서 38편 — auth · billing · data-modeling · security · compliance · operations ([인덱스](./explainers/README.md)) |
| [CONVENTIONS.md](./CONVENTIONS.md) | 📐 문서 작성 규약 (파일명·frontmatter·wikilink·다이어그램·DB 엔진 표기) |

## 🧩 핵심 한 장 요약

- **격리**: 모든 도메인 테이블 `org_pk NOT NULL`(Pool 모델) + 질의 강제 → BOLA 차단
- **인가**: 3-gate(소속·결제·정책), 역할은 코드 상수, 위임은 `delegation_grant`
- **결제**: `org_subscription`(진실) → `org_entitlement`(권한 투영), 단일 트랜잭션 + outbox
- **불변**: 금융·감사·동의는 append-only(GRANT로 강제), 식별자는 BIGINT pk + ULID public_id
