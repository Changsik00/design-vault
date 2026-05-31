---
type: decision
status: 채택
aliases:
  - 비대칭 분리
  - platform_db 경계
  - asymmetric split
tags:
  - platform-db
  - decision
  - architecture
  - boundary
  - multitenancy
---

# 비대칭 분리 — platform_db(공통) + 서비스 DB(도메인)

## 배경

academy / agent / market / store 네 서비스가 공통으로 필요로 하는 것이 있다:

- "이 사람이 누구인가" (identity)
- "이 org가 서비스를 쓸 수 있나" (entitlement)
- "무엇을 얼마에 구독하나" (billing)

이것을 어느 DB에 둘 것인가.

동시에, 각 서비스마다 고유한 데이터도 있다:
- academy: 강의·영상·청크·숙제
- market: 상품·주문·리뷰
- agent: 세션·툴콜·토큰 사용량

두 종류의 데이터를 하나로 합칠 것인가, 완전히 분리할 것인가.

---

## 대안 A — 완전 MSA (서비스마다 독립 DB)

```
academy-api  →  academy_db  (identity 복사본 + 강의 데이터)
agent-api    →  agent_db    (identity 복사본 + 세션 데이터)
market-api   →  market_db   (identity 복사본 + 상품 데이터)
```

**권고 근거**:
- 서비스 독립 배포·확장 가능
- 한 서비스 장애가 다른 서비스에 전파 안 됨
- 팀별 DB 스키마 완전 독립

**문제점**:
- identity 복사본 동기화 지연 → 같은 사람이 서비스마다 다르게 보임
- market에서 academy·agent 상품을 한 화면에 보여주려면 N개 DB 조회 또는 별도 집계 레이어 필요
- 결제 트랜잭션이 어느 DB에? → 분산 트랜잭션 2PC 또는 saga 필요

---

## 대안 B — 모놀리스 (단일 DB)

```
모든 서비스 → 단일 DB
  identity + billing + academy 데이터 + agent 데이터 + market 데이터 ...
```

**권고 근거**:
- 트랜잭션 단순 (BEGIN/COMMIT 하나로 끝)
- 어디서나 JOIN 자유
- 운영 인프라 단순

**문제점**:
- 고속 증가 테이블 (채팅 로그, 감사 로그)이 공통 DB를 압박 → 모든 서비스 동시 영향
- 강의 스키마 변경이 market/agent 배포에도 영향
- 서비스별 독립 확장 불가 (academy 트래픽이 agent를 압박해도 분리 불가)

---

## 우리 결정 — 비대칭 분리

> **공통은 묶고, 도메인은 뗀다.**

```
                  ┌─────────── platform_db (공통 코어) ───────────┐
                  │  identity · billing · product · consent · audit │
  academy-api ───▶│                                                 │◀── @aiagent/db-platform 패키지만 접근
  agent-api   ───▶│  = "누구 / 어디 소속 / 무엇을 구독 / 무엇에 동의"   │
  market-api  ───▶└─────────────────────────────────────────────────┘
                        ↕ (읽기만. platform_db → 서비스 금지)
  academy-api →  academy_db  (강의·영상·청크. 전부 org_pk NOT NULL)
  agent-api   →  agent_db    (세션·툴콜)
  market-api  →  market_db   (상품·주문)
```

**cross-DB 방향 규칙**:
- `서비스 DB → platform_db` 읽기: 허용
- `platform_db → 서비스 DB`: 금지
- `academy_db → market_db` (peer): 금지

**채택 근거**:
1. market의 cross-service 단일 상품 카탈로그 → `platform_db.product`에 통합
2. 결제↔권한 단일 트랜잭션 → 같은 Postgres(MVCC) 안에서 `BEGIN/COMMIT` (분산 트랜잭션 불필요)
3. 모든 서비스가 한 곳(`platform_db`)만 보고 entitlement 확인
4. 현 규모(팀 5인 이하)에서 분산 트랜잭션 운영 비용이 이익보다 큼

---

## 트레이드오프

| 항목 | 완전 MSA | 모놀리스 | 비대칭 분리 (우리) |
|---|---|---|---|
| cross-service 카탈로그 | 복잡 (동기화) | 쉬움 | 쉬움 (platform_db) |
| 분산 트랜잭션 | 필요 | 불필요 | **불필요** |
| 서비스 독립 확장 | 가능 | 불가 | 가능 (도메인 DB) |
| platform_db 비대화 위험 | 없음 | 최대 | **있음** → 아래 대응 |
| 운영 복잡도 (현 규모) | 높음 | 낮음 | 중간 |

---

## 향후 조건

`platform_db` 비대화가 감지되면 단계적 분리:

1. **지금 (예방)**: 논리 컨텍스트 모듈 분리 — `identity/` · `billing/` · `consent/` 각 모듈이 자기 테이블만 쓰기
2. **트래픽/팀 분리 시**: 컨텍스트 단위 물리 DB 분리
3. **소비 앱 3개+ / SOC2 / PCI**: Option B — `platform-api` HTTP 서비스로 전환 (인터페이스는 동일하게 유지하므로 소비 앱 변경 없음)

---

## 관련 문서

- [[payment-atomicity]] — 단일 트랜잭션이 비대칭 분리에서만 가능한 이유
- [[identity-billing-access]] — 공통 코어를 무엇으로 감싸 노출하는가
> 소스 문서
- [[architecture]] — §1.1 개요, §3 설계 여정, §2.4 비대화 대응 플레이북
