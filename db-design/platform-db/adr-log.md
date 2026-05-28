---
tags:
  - platform-db
  - adr
  - decision
aliases:
  - ADR 통합 기록
  - ADR-044
  - ADR-041
  - ADR-042
  - ADR-032
  - ADR-035
  - ADR-036
  - ADR-025
---

# platform_db — ADR 통합 기록

> 출처: [[architecture|architecture.md]] §18 부록 A  
> platform_db 설계에서 참조하는 ADR의 원문 핵심(배경·결정·이유·기각)을 한 곳에 통합.

---

## ADR-044 — `platform_db` 경계: identity + billing + product 통합

- **배경**: DB 분리 3안 검토 — (A)`identity_db`/`billing_db`/서비스별 세분화, (B)`platform_db`(identity만)+서비스별 billing, (C)`platform_db`(identity+billing+product)+서비스별 DB. 트리거: "market에서 academy·agent·store 상품을 한 곳에서 구매."
- **결정**: **안 C 채택** — `platform_db` 하나에 identity+billing+product 통합, 서비스 도메인만 분리.
- **이유**: ① market이 cross-service 단일 상품 카탈로그 필요(N개 DB 조회 회피) ② 각 서비스가 한 곳(platform_db)만 보고 entitlement 확인 ③ 구매 트랜잭션이 단일 DB 내 완결(분산 트랜잭션 없음) ④ 현 규모엔 완전 MSA보다 모듈형 모놀리스가 적합.
- **기각**: 안A(같은 서비스가 identity/billing 두 커넥션·cross-DB 조인 불가·운영복잡 대비 실익 없음), 안B(cross-service 카탈로그/통합 구독현황 조회 불가).

---

## ADR-041 — 멀티테넌시 DB 전략: 단일 DB 행 격리 + 분리 트리거

- **배경**: SaaS, 테넌트=`organization`. 고속 증가 테이블(homework_chat, usage_log, notification, audit_log) 존재.
- **결정**: **단일 DB + `org_pk` 행 격리.** 모든 도메인 테이블 `org_pk NOT NULL` 불변식. 미래 분리를 위해 앱이 항상 org_pk 전달.
- **분리 트리거**: T1(단일 org가 고속 테이블 20%+) · T2(조회 P99>500ms) · T3(usage_log 월 500만/50GB+) · T4(ISMS-P/GDPR). 단계: Read replica → 고속 테이블 분리(chat_db/analytics_db) → DB-per-large-tenant.
- **기각**: schema-per-tenant(MySQL은 db=schema, 테넌트마다 DB면 connection pool 복잡), 즉시 DB-per-tenant(현 규모 운영비만↑), NoSQL 전환(MySQL+Drizzle 투자 보호).

---

## ADR-042 — cross-tenant 조회: Admin Role 거부 + 아키텍처 분리

- **배경**: 운영 집계(전 테넌트 가로지르는 학원수·강의수·사용량) 필요. MySQL/Qdrant/Neo4j 모두 org 격리 중.
- **결정**: **Admin role 방식 거부.** cross-tenant 조회는 전용 모듈(`internal/`) 또는 별도 서비스(`*-admin`)에서만 — raw SQL로 의도를 코드 *위치*로 드러냄.
- **이유**: Admin role = ① 단일 장애점(탈취 시 전 테넌트 노출) ② 비즈니스 로직에 보안우회 혼재 ③ role 남용 확산 ④ 격리 불변식이 런타임 조건에 의존하게 됨.
- **규칙**: 일반 모듈은 `org_pk`/`org_id` 필수 타입, 위반 PR reject, `internal/` 컨트롤러는 외부 라우팅 노출 금지(내부망/별도 인증).

---

## ADR-032 — identity/billing 접근: 공유 패키지(Option A) + 분리 서비스(Option B) 전환 준비

- **배경**: academy-api 외 agent/market도 동일 identity/billing 인프라 공유 필요. "어떤 앱이 어떻게 접근하나"를 초기에 결정해야(잘못하면 앱 증가 시 마이그레이션 비용 지수증가).
- **결정**: **Option A 채택**(`@aiagent/db-platform` 공유 패키지로만 접근, Drizzle 직접 금지). 단 **인터페이스를 Option B처럼 설계**해 전환 비용 최소화 — 마이그레이션 비용의 실체는 "A vs B"가 아니라 "경계가 있나 없나".
- **Option B 전환 트리거**(`platform-api` HTTP 서비스): 소비 앱 3개 이상 동시 / platform 전담 팀 분리 / SOC2·PCI-DSS. 전환 시 소비 앱 0 변경(`getPermissionContext()` 시그니처 유지, 구현만 HTTP로 교체).
- **기각**: Option C(각 앱이 Drizzle 직접 — 스키마 변경 시 전 앱 동시 수정·통제 불가), 즉시 Option B(소비 앱 1개 단계에 ROI 음수).

---

## ADR-035 — Qdrant 멀티테넌시: shared collection + payload 필터

- **결정**: 단일 collection + payload 필터(`org_id`). collection-per-tenant 아님(인덱스가 RAM 상주 → 메모리 비용 폭탄). 대규모 시 `org_id` payload **인덱스 + `is_tenant` 최적화**(동일 테넌트 벡터 디스크 co-locate). Qdrant는 filterable HNSW라 post-filter recall 저하가 덜함.
- **현재 구현**: 색인(`qdrant-index.adapter.ts`)과 검색(`qdrant-search.service.ts`) 모두 `org_id` must 필터 강제. `is_tenant` 마커는 P1 추가 예정.

---

## ADR-036 — Neo4j 격리: orgId 속성 + Cypher 강제

- **결정**: 단일 그래프 + 모든 노드 `org_id` 속성 + Cypher `WHERE org_id` 자동 주입. 동적 라벨(`:TenantA`) 아님(테넌트 수백~수천 → 라벨 카디널리티 폭발). **멀티홉 경로는 anchor만이 아니라 경로 전체 노드 org_id 강제**, 쓰기측 교차 관계 차단(APOC 트리거 P1 검토).
- **현재 구현**: `neo4j-concept.adapter.ts` — `LectureChunk`, `Concept` 노드 생성 시 `orgId`를 MERGE key에 포함. 모든 읽기 쿼리에 `{ orgId: $orgId }` 강제. 멀티홉 관계(`RELATED_TO`) 양 끝 노드 모두 orgId 필터. APOC 쓰기측 차단은 P1.

---

## ADR-025 — 도메인 바운디드 컨텍스트 / YAGNI 원칙

- **결정(원래 academy content-api 맥락)**: `domain/` 레이어 경계 명확화(content/stt/upload), 첫 사용처 없는 디렉토리 안 만듦(YAGNI), dispatch 신호(SourceType)는 단일 출처, `shared/`는 순수 cross-cutting만.
- **platform 계승**: "컨텍스트 경계를 코드로 강하게, 빈 추상 미리 만들지 않기"가 platform_db **논리 소유권(§12.1)**·**YAGNI 보류 목록(§11)**의 뿌리.

> 참고: `agent_db` 관련 ADR-038(video pipeline nodejs)·ADR-039(3pod worker/qdrant chunking)·ADR-043(lecture ownership)은 서비스 도메인 영역이라 본 platform 핸드북 범위 밖.

---

## 연결된 개념

- [[architecture]] — §2 설계 여정(R0~R8), §4 D1~D12, §8 멀티테넌시, §10 비교 분석
- [[multitenancy-rls]] — ADR-041/042 멀티테넌시 격리 전략 상세 설명
- [[role-capability]] — ADR-042 Admin role 거부 배경
