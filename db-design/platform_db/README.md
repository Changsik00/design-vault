# platform_db — 문서 허브

> 진입점. 어디서 무엇을 읽을지 이 페이지에서 결정한다.

---

## 폴더 구조

```
platform_db/
  core/        핵심 레퍼런스 — 아키텍처·스키마·요구사항·운영
  decisions/   설계 결정 — 왜 이렇게 만들었나 (gate-b 스타일)
  adr/         ADR 원문 — 의사결정 기록 (ADR-025 ~ ADR-044)
  explainers/  온보딩 Q&A — DB 지식이 적은 개발자를 위한 18개 설명 문서
  study/       DB 학습 — 이 프로젝트 사례로 배우는 일반 DB 개념
```

---

## 목적별 진입점

| 목적 | 시작점 |
|---|---|
| 전체 그림이 보고 싶다 | [[architecture]] |
| 스키마·DDL이 궁금하다 | [[schema-reference]] |
| "왜 이렇게 설계했나?" | `decisions/` |
| ADR 원문이 필요하다 | `adr/` |
| 처음 합류해 Q&A로 배우고 싶다 | [[explainers/README\|explainers/]] |
| DB 개념을 공부하고 싶다 | `study/` |

---

## core/ — 핵심 레퍼런스

| 파일 | 내용 |
|---|---|
| [[architecture]] | 아키텍처 핸드북 (진입점) — 8라운드 설계 여정, D1~D12 결정, 운영 플레이북, ADR 부록 |
| [[schema-reference]] | ERD · DDL · 3-gate · billing 흐름 · 멀티테넌시 · 보안 · consent 모델 |
| [[requirements]] | 요구사항 추적표 104건 + BDD 시나리오 Domain 1~10 |
| [[review-checklist]] | 설계 검토 체크리스트 P0~P5 (42건, 23 완료) |
| [[migration-ops]] | Drizzle 마이그레이션 운영 가이드 (0000~0006) |

---

## decisions/ — 설계 결정

> `gate-b-billing-grace`와 동일 형식.  
> **배경 → 대안 → 우리 결정(이탈 근거) → 트레이드오프 → 향후 조건 → 관련 파일**

| 파일 | 한 줄 요약 |
|---|---|
| [[decisions/gate-b-billing-grace\|gate-b-billing-grace]] | Gate B validUntil 복합체크 — status-only 권고를 이탈한 이유 |
| [[design-asymmetry]] | 비대칭 분리 — MSA도 모놀리스도 아닌 platform_db + 서비스 DB |
| [[auth-projection]] | billing truth → auth projection — org_entitlement 분리 이유 |
| [[payment-atomicity]] | 결제↔권한 단일 트랜잭션 — Kafka를 쓰지 않은 이유 |
| [[multitenancy-pool]] | Pool 모델 — schema-per-tenant를 거부한 이유 |
| [[firebase-boundary]] | Firebase = 인증만 — 인가를 우리 DB에 둔 이유 |
| [[role-as-code]] | role→action 코드 상수 — DB 레지스트리를 거부한 이유 |

---

## adr/ — ADR 원문

| 파일 | 결정 |
|---|---|
| [[ADR-044-platform-db-boundary\|ADR-044]] | platform_db 경계: identity + billing + product 통합 |
| [[ADR-041-multitenancy-db-strategy\|ADR-041]] | 멀티테넌시: 단일 DB + 행 격리 + 분리 트리거 |
| [[ADR-042-cross-tenant-query-separation\|ADR-042]] | cross-tenant 조회: Admin Role 거부 + 아키텍처 분리 |
| [[ADR-032-platform-identity-billing-access-strategy\|ADR-032]] | identity/billing 접근: Option A(공유 패키지) + Option B 전환 준비 |
| [[ADR-035-rag-shared-collection-multitenancy\|ADR-035]] | Qdrant: shared collection + payload filter |
| [[ADR-036-qdrant-neo4j-dual-index\|ADR-036]] | RAG 이중 색인: Qdrant(dense) + Neo4j(graph) |
| [[ADR-025-domain-bounded-context\|ADR-025]] | 도메인 바운디드 컨텍스트 / YAGNI |

---

## explainers/ — 온보딩 Q&A

> DB 지식이 많지 않은 개발자를 위한 Q&A 18편. 전체 목록·읽는 순서는 [[explainers/README\|explainers/README]] 참고.

처음 합류했다면 P0부터:

| # | 파일 | 핵심 질문 |
|---|---|---|
| 1 | [[gate-abc-flow]] | API 요청 하나가 DB에서 어떤 3단계를 거치나요? |
| 2 | [[gate-b-entitlement]] | 결제 상태랑 서비스 접근이 왜 다른 테이블인가요? |
| 3 | [[role-capability]] | platform_role이 뭐고 service_role이 뭔가요? |
| 4 | [[pk-ulid-strategy]] | pk랑 public_id가 두 개인데 뭘 쓰면 되나요? |
| 5 | [[multitenancy-rls]] | MySQL은 RLS가 없는데 다른 org 데이터를 어떻게 막나요? |

---

## study/ — DB 학습

> 이 프로젝트 사례로 배우는 **일반 DB 모델링 개념**.  
> (인덱스·파티셔닝·ENUM·outbox·멱등키·PK 전략 등 프로젝트 맥락이 강한 주제는 `explainers/`로 통합됨.)

| 파일 | 배울 것 |
|---|---|
| [[soft-delete]] | 삭제 3종 패턴: status / deleted_at / append-only |
| [[fk-strategy]] | FK: 언제 걸고, cross-schema에서 왜 끊나 |
| [[json-column]] | JSON 컬럼: 언제 쓰고 언제 정규화하나 |
| [[append-only]] | append-only 패턴: 왜 UPDATE를 안 하나 |
