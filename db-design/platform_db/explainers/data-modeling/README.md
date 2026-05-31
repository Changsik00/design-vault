---
type: index
aliases:
  - data-modeling 인덱스
tags:
  - platform-db
  - index
---

# 🗃️ data-modeling — 데이터 모델링

식별자·삭제·인덱스·파티셔닝·DDL 마이그레이션.

| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[delete-patterns]] | 삭제·이력 보존 패턴 (status/deleted_at/append-only) |
| 🟢 초 | [[enum-vs-varchar-check]] | ENUM vs VARCHAR+CHECK (D6) |
| 🟢 초 | [[json-column]] | JSONB 컬럼 (언제 쓰고 언제 정규화) |
| 🟢 초 | [[pk-ulid-strategy]] | BIGINT pk + ULID public_id 전략 |
| 🟡 중 | [[fk-strategy]] | FK 전략 (cross-schema에서 왜 끊나) |
| 🟡 중 | [[index-design]] | 인덱스 설계 원리 |
| 🟡 중 | [[partitioning]] | 선언적 파티셔닝 (audit_log) |
| 🔴 고 | [[online-ddl-migration]] | 온라인 DDL과 대형 테이블 마이그레이션 |
| 🔴 고 | [[concurrency-control]] | 동시성 제어 — race condition과 락·조건부 UPDATE |

> 전체 주제 인덱스: [explainers 마스터 인덱스](../README.md) · 작성 규약: [CONVENTIONS](../../CONVENTIONS.md)
