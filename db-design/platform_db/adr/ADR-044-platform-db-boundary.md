# ADR-044: platform_db 경계 — identity + billing + product 통합

| 항목 | 값 |
|---|---|
| **상태** | Accepted |
| **날짜** | 2026-05-27 |
| **작성자** | dennis |
| **타입** | decision |
| **영향 범위** | `packages/db-platform`, academy-api, 미래 agent/store 서비스 |

---

## 배경

phase-16 설계 중 DB 분리 전략을 논의하는 과정에서 세 가지 안이 검토됐다.

- **안 A**: `identity_db` / `billing_db` / 서비스별 DB — 세분화된 마이크로서비스 패턴
- **안 B**: `platform_db`(identity만) + 서비스별 billing — identity만 공유
- **안 C**: `platform_db`(identity + billing + product) + 서비스별 DB — **채택**

논의 트리거: "market 페이지에서 academy·agent·store 상품을 한 곳에서 구매할 수 있어야 한다."

---

## 결정

**`platform_db` 하나에 identity + billing + product 를 통합한다.**

```
platform_db  (packages/db-platform — 모든 서비스 공유)
  ├── [identity]  organization, user_profile, membership,
  │               delegation_grant, org_relation
  ├── [product]   product, product_sku
  └── [billing]   org_subscription, org_entitlement,
                  billing_event, plan_definition

academy_db   (academy-api 전용)
  ├── lecture, video_asset, homework_chat, lecture_file, ...

agent_db     (미래 — agent-api 전용)
store_db     (미래 — store-api 전용)
```

---

## 이유

### 1. market 페이지는 cross-service 상품 카탈로그가 필요하다

market에서 "Academy Pro", "Agent Basic", "Store Premium" 상품을 한 페이지에서 보여주려면  
`product` / `product_sku` 가 **서비스 불문 단일 카탈로그** 여야 한다.  
각 서비스 DB에 상품을 쪼개 두면 market 페이지가 N개 DB를 조회해야 한다.

### 2. 각 서비스는 platform_db 한 곳만 보고 entitlement를 확인한다

```
academy-api:  platform_db → "org가 academy 구독 중?" → lecture DB 접근 허용
agent-api:    platform_db → "org가 agent 구독 중?"   → agent DB 접근 허용
store-api:    platform_db → "org가 store 구독 중?"   → store DB 접근 허용
```

서비스마다 별도 billing DB를 두면 entitlement 조회 위치가 분산된다.

### 3. 트랜잭션 경계가 명확하다

- **구매 트랜잭션**: platform_db 단일 DB 내 완결 (org_subscription INSERT + billing_event INSERT)
- **서비스 트랜잭션**: 서비스 DB 단일 DB 내 완결 (lecture INSERT 등)
- cross-DB 분산 트랜잭션 없음

### 4. 현재 규모에 적합한 모듈형 모놀리스

완전한 MSA(서비스마다 DB 분리)는 팀 규모·운영 복잡도 측면에서 현재 단계에서 오버엔지니어링이다.  
`platform_db` + 서비스별 DB 2단계 구조는 서비스 독립 배포 가능성을 열어 두면서 운영 부담을 최소화한다.

---

## 기각된 대안

### 안 A: identity_db / billing_db 세분화
- **기각 이유**: 같은 서비스(academy-api) 안에서 identity_db·billing_db 두 커넥션 관리, cross-DB 조인 불가, 운영 복잡도 증가 대비 실익 없음

### 안 B: platform_db = identity만, billing은 서비스별
- **기각 이유**: market에서 cross-service 상품 카탈로그가 필요할 때 각 서비스 DB를 별도 조회해야 함. org 통합 구독 현황 조회 불가.

---

## 영향

### 즉시 (spec-16-02)
- `drizzle.config.identity.ts` → `drizzle.config.platform.ts` rename
- `drizzle/identity/` → `drizzle/platform/` 마이그레이션 통합 재생성
- `src/billing/schema.ts` product/sku/subscription 추가 → platform_db 포함
- `package.json` 스크립트: `db:identity:*` → `db:platform:*`
- 환경변수: `IDENTITY_DB_URL` → `PLATFORM_DB_URL`

### 중기 (spec-16-03)
- academy-api: `PLATFORM_DB_URL` 단일 연결 + `ACADEMY_DB_URL` 연결

### 미래 서비스
- agent-api, store-api 추가 시: `PLATFORM_DB_URL` 연결 추가만 하면 entitlement 체크 가능
- 서비스별 product는 `product.type` 또는 `product_sku.serviceCode` 컬럼으로 구분

---

## 관련 ADR

- [[ADR-041-multitenancy-db-strategy|ADR-041]] — 멀티테넌시 DB 전략 (Pool 모델 채택)
- ADR-043 (lecture_file 원장 + S3 경로 규약) — academy 서비스 도메인 ADR로, 본 platform_db 볼트 범위 밖
