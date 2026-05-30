---
type: decision
status: 채택
aliases:
  - Pool 모델
  - 멀티테넌시 행 격리
  - org_pk isolation
tags:
  - platform-db
  - decision
  - multitenancy
  - isolation
  - pool
---

# Pool 모델 — 공유 DB + org_pk 행 격리

## 배경

수백~수천 개의 학원(org)이 같은 DB를 공유한다. 학원 A의 데이터가 학원 B에 노출되어서는 안 된다.

SaaS에서 **테넌트 격리(tenant isolation)** 는 핵심 보안 요건이다. 이를 어떻게 구현할 것인가.

그리고 격리하면서도 운영 비용이 폭발하지 않아야 한다.

---

## 대안 A — schema-per-tenant

```sql
-- 학원마다 별도 스키마(DB)
CREATE DATABASE academy_org_1;
CREATE DATABASE academy_org_2;
-- ...

-- 각 DB에 동일 구조 복제
```

**권고 근거**:
- DB 레벨에서 완전 격리
- 한 학원의 쿼리가 다른 학원 성능에 영향 없음

**문제점**:
- MySQL에서 schema = database → 학원마다 별도 DB = connection pool 폭발
  - 학원 500개 → 동시 connection 수 급증 → MySQL max_connections 한계
- 스키마 마이그레이션이 학원 수만큼 반복 실행
- 학원 간 집계 쿼리 불가 (어드민 대시보드에서 전체 학원 현황 보기)

---

## 대안 B — DB-per-tenant

```
academy_org_1 DB → 서버 A
academy_org_2 DB → 서버 B
...
```

**권고 근거**:
- 완벽한 물리적 격리
- GDPR/ISMS-P 계약 학원에 전용 인프라 제공 가능

**문제점**:
- 학원 1개당 MySQL 인스턴스 → 비용 10배+
- 학원 수가 늘어날수록 운영 오버헤드 선형 증가
- 현 규모(수십~수백 학원)에서는 비현실적

---

## 우리 결정 — Pool 모델

**공유 DB + `org_pk` 행 격리**

```sql
-- 모든 도메인 테이블에 org_pk NOT NULL — 불변식 #3
CREATE TABLE lecture (
  pk     BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk BIGINT UNSIGNED NOT NULL,  -- 이 컬럼이 테넌트 경계
  ...
);

-- 모든 조회에 org_pk 필터 강제
SELECT * FROM lecture
WHERE org_pk = ?  -- 이게 없으면 타 학원 데이터 노출
  AND pk = ?;

-- org_pk 없는 쿼리 → 404 반환 (데이터 존재 여부도 숨김)
```

**이중 차단 구조**:
- ① 개발자가 `org_pk` 빼먹으면 → `NOT NULL` constraint로 INSERT 자체 불가
- ② 클라이언트가 `tenant_id` 위조하면 → JWT 검증 후 server-side에서 orgPk 바인딩 (클라이언트 값 무시)

**솔직한 제약**: MySQL은 PostgreSQL의 RLS(Row Level Security) 같은 DB 네이티브 격리가 없음.
→ 대신 CI 린트로 보완: `WHERE org_pk` 없는 쿼리를 빌드 시 탐지 (P1 미완)

---

## 분리 트리거 — 언제 Pool 모델을 벗어나나

미리 정의된 조건이 충족되면 단계적으로 분리:

| 트리거 | 조건 | 행동 |
|---|---|---|
| T1 | 단일 org가 고속 테이블 행수의 20%+ | 해당 org 전용 DB |
| T2 | 고속 테이블 P99 > 500ms | 해당 테이블 별도 인스턴스 |
| T3 | usage_log 월 500만 건 또는 50GB+ | OLAP(ClickHouse/BigQuery) 이관 |
| T4 | ISMS-P / GDPR 계약 | 물리 격리 DB + 감사 외부 WORM |

`org_pk` 단위 격리라 특정 org만 무중단 마이그레이션 가능.

---

## 트레이드오프

| 항목 | schema-per-tenant | DB-per-tenant | Pool 모델 (우리) |
|---|---|---|---|
| 격리 강도 | DB 레벨 | 물리 서버 | 행 레벨 (앱 레이어) |
| 운영 비용 | 중간 (connection pool) | 높음 (서버 증가) | **낮음** |
| 마이그레이션 | 학원수만큼 반복 | 서버마다 | **1번** |
| 학원 간 집계 | 불가 | 불가 | 가능 (admin 전용 모듈) |
| RLS 네이티브 | 없음 (MySQL) | 없음 | **없음 → CI 린트로 보완** |

---

## 향후 조건

T1~T4 트리거 도달 시 단계 실행:
1. Read replica 추가
2. 고속 테이블만 별도 DB (`chat_db`, `analytics_db`)
3. 대형 테넌트 전용 DB-per-tenant

---

## 관련 문서

- [[cross-tenant-separation]] — 격리를 의도적으로 벗어나는 cross-tenant 집계 처리
- [[rag-multitenancy]] — 같은 "공유 + 격리" 철학의 Qdrant 적용
> 소스 문서
- [[architecture]] — §8 멀티테넌시 격리 전략 전문
- [[schema-reference]] — §G Qdrant·Neo4j·Redis 격리 구현 현황
