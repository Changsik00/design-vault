# Academy AI Platform — DB 설계 핸드북

> 작성일: 2026-05-25  
> 상태: v0.1 MVP 설계 완료 / v0.5~v2.0 확장 설계 포함

학원 AI 플랫폼(Academy)의 **멀티테넌트 DB 설계** 전체 문서 모음.  
STT → Claude → YouTube 자동 파이프라인 + RAG 기반 학원장 Q&A를 MVP로 하는 SaaS 제품.

---

## 문서 구조

### 핵심 설계 문서
| 파일 | 내용 | 우선순위 |
|---|---|---|
| [platform-data-design.md](./platform-data-design.md) | 플랫폼 전체 DB 스키마 (identity/billing/academy) | **필독** |
| [data-model.md](./data-model.md) | Academy 도메인 데이터 모델 (lecture, chunk, video 등) | **필독** |
| [db-design-decisions.md](./db-design-decisions.md) | 설계 결정 이유 & 트레이드오프 기록 | **필독** |
| [auth-and-policy.md](./auth-and-policy.md) | 인증/인가 레이어 (3-gate canXXX() 구조) | **필독** |

### 정책 & 전략
| 파일 | 내용 |
|---|---|
| [identity-policy.md](./identity-policy.md) | Identity 정책 (HUMAN/SERVICE 분리, perm_version) |
| [rag-strategy.md](./rag-strategy.md) | RAG 아키텍처 (Qdrant + Neo4j, org 네임스페이스) |
| [pipeline.md](./pipeline.md) | STT→Claude→TTS→YouTube 파이프라인 설계 |
| [tech-stack.md](./tech-stack.md) | 기술 스택 선택 이유 |

### 제품 & 비즈니스
| 파일 | 내용 |
|---|---|
| [vision.md](./vision.md) | 제품 비전 & 미션 |
| [personas.md](./personas.md) | 페르소나 (학원장/강사/학생/학부모) |
| [features.md](./features.md) | 기능 목록 F1~F47 |
| [mvp-scope.md](./mvp-scope.md) | MVP 스코프 & 졸업 조건 |
| [phase-roadmap.md](./phase-roadmap.md) | v0.1→v0.5→v1.0→v2.0 Phase 로드맵 |
| [business-model.md](./business-model.md) | 수익 모델 & 플랜 구조 |
| [competitive-edge.md](./competitive-edge.md) | 경쟁 우위 분석 |
| [risks.md](./risks.md) | 리스크 & 대응 전략 |

### 검증
| 파일 | 내용 |
|---|---|
| [bdd-scenarios.md](./bdd-scenarios.md) | BDD 시나리오 F1~F51 (Gherkin 형식) |

---

## 핵심 설계 원칙

### 1. 멀티스키마 구조
```
identity_db   — identity_user, user_profile, organization, membership, delegation_grant
billing_db    — plan, subscription, payment_ledger, org_entitlement
academy_db    — lecture, lecture_chunk, youtube_video, student, usage_log, ...
```

### 2. Tenant Root: `organization(type='ACADEMY')`
- `academy_config`는 학원 도메인 설정만 (STT 언어, 유튜브 채널 등)
- Org 자체가 tenant SSOT — academy_config를 tenant로 쓰지 않음

### 3. PK 전략
- 내부: `BIGINT UNSIGNED AUTO_INCREMENT` (pk) — JOIN 성능
- 외부: `CHAR(26) ULID public_id` — URL/API 노출용
- 외부에 절대 pk 노출 금지

### 4. 3-Gate 인가 구조
```
Gate A: membership.status = ACTIVE          (이 학원 소속인가?)
Gate B: org_entitlement.status = ACTIVE|GRACE  (플랜 유효한가?)
Gate C: CASL policy (role+trust_relationship)   (이 행위 허용되나?)
```
`canXXX()` 함수는 항상 세 게이트를 순서대로 통과.  
`org_entitlement`가 런타임 권한의 유일한 진실 — `payment_ledger` 직접 참조 금지.

### 5. 프랜차이즈 / 다학원 구조
```sql
-- org_relation 행 없음 = 독립 학원
-- org_relation 행 있음 = 프랜차이즈
org_relation(parent_org_pk, child_org_pk, relation_type='HQ_BRANCH'|'HOLDING')
```
한 학원장이 여러 학원 운영 가능 (membership 각 org에 개별 보유).  
권한 상속 없음 — 각 org의 membership을 명시적으로 부여.

### 6. 파티셔닝 (P0)
| 테이블 | 기준 | 보존 |
|---|---|---|
| audit_log | created_at RANGE(MONTHLY) | 3년 |
| usage_log | created_at RANGE(MONTHLY) | 1년 |
| notification | created_at RANGE(MONTHLY) | 90일 |

---

## Phase 로드맵 요약

| Phase | 기간 | 핵심 |
|---|---|---|
| v0.1 | 8주 | 음성→YouTube 자동 파이프라인 + RAG 인프라 + student/usage_log |
| v0.5 | 8주 | 학원장 RAG Q&A + 운영 대시보드 5개 |
| v1.0 | 12주 | 학생 RAG 채팅 + 강사 평가 + 학부모 PWA |
| v2.0 | 16주 | Android 자동 녹음 + 프랜차이즈 코드 활성화 |

---

## 관련 기술 스택

- **DB**: MySQL 8 (멀티스키마) + Qdrant (벡터) + Neo4j (그래프)
- **ORM**: Drizzle ORM — `findOne(pk, orgPk)` 항상 org 격리
- **Auth**: CASL (TypeScript) + trust_relationship 위임
- **Queue**: BullMQ + Redis
- **Pipeline**: Google STT → Claude → Google TTS → HyperFrames → YouTube
- **Embedding**: Upstage Embedding API
