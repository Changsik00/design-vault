---
type: index
aliases:
  - decisions 인덱스
  - 의사결정 인덱스
tags:
  - platform-db
  - index
---

# 🧭 decisions — 설계 의사결정 인덱스

"왜 이렇게 했나"를 비교와 함께 남긴 결정 기록 13편. 모두 *채택* 상태.

| 결정 | 한 줄 |
|---|---|
| [[multitenancy-pool]] | Pool 모델 — 공유 DB + org_pk 행 격리 |
| [[payment-atomicity]] | 결제 단일 Postgres 트랜잭션 (2PC·Kafka 거부) |
| [[auth-projection]] | 권한 투영 — org_subscription → org_entitlement |
| [[audit-two-lane]] | 감사 2-lane (audit_log vs payment_ledger) |
| [[role-as-code]] | role→action을 코드 상수로 |
| [[service-extensibility]] | 서비스 확장과 결합 (VARCHAR+CHECK) |
| [[design-asymmetry]] | 통합 platform_db (identity/billing 비분리) |
| [[firebase-boundary]] | Firebase 경계 — firebase_uid는 조회 키 |
| [[identity-billing-access]] | identity·billing 접근 전략 |
| [[cross-tenant-separation]] | cross-tenant 조회는 아키텍처 분리 |
| [[operator-plane]] | 운영자 신원 평면 (org_pk 없는 operator) |
| [[decisions/gate-b-billing-grace|gate-b-billing-grace]] | Gate B 유예 기간 (status + validUntil) |
| [[rag-multitenancy]] | RAG(Qdrant·Neo4j) 멀티테넌시 격리 |

> 상위 허브: [platform_db README](../README.md) · 작성 규약: [CONVENTIONS](../CONVENTIONS.md)
