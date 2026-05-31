---
difficulty: 초
tags:
  - platform-db
  - explainer
  - p1
  - schema
  - postgres
  - json
  - design-decision
aliases:
  - JSON 컬럼
  - JSON vs 정규화
  - json-column
---

# JSON 컬럼 설명 (언제 쓰고 언제 정규화하나)

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[schema-reference]] §D.12·§D.8·§I.2, [[architecture]] §2.1, [[feature-limits]]

PostgreSQL은 `JSONB` 타입 컬럼을 지원합니다. 스키마 없이 유연하게 저장할 수 있고 GIN 인덱스로 검색까지 되니 매력적이지만, "언제나 JSON"은 나쁜 선택입니다. 어디에 쓰고 어디에 쓰면 안 되는지 설명합니다.

---

## Q1. JSON 컬럼은 언제 쓰면 좋나요?

**구조가 row마다 다르고, 통째로 읽어 앱에서 해석하는 데이터**에 적합합니다.

**① 구조가 제품/플랜마다 다른 경우** — `org_entitlement.feature_limits`:

```
academy 플랜: {"daily_uploads": 6, "members": 50, "storage_gb": 10}
market 플랜:  {"listings": 100, "featured_slots": 5}
agent 플랜:   {"monthly_tokens": 1000000, "concurrent_agents": 3}
```

product마다 한도 항목이 다릅니다. 정규화하면 `feature_limit_value(entitlement_pk, feature_key, limit_value)` 같은 테이블 + JOIN이 필요한데, 한도는 항상 전체를 읽어 앱에서 해석하므로 JSON이 단순합니다. ([[feature-limits]])

**② 감사/이벤트 메타데이터** — `audit_log.meta_json`: API 호출마다 기록 내용이 다름.

```
{"http_method": "POST", "request_body_hash": "abc123", ...}
{"reason": "permission_denied", "required_role": "TEACHER", "actual_role": "STUDENT"}
```

**③ PIPA 제3자 제공 4요건** — `user_consent_event.meta_json`: 동의 종류마다 항목이 다름.

```
{"recipient": "Google Firebase", "purpose": "인증", "items": ["email"], "retention": "탈퇴 후 30일"}
```

> 💡 **한 줄 요약**: 구조가 row마다 다르고 "통째로 읽어 앱에서 해석"하는 데이터(한도·메타·4요건)는 JSON이 적합합니다.

---

## Q2. JSON을 쓰면 안 되는 경우는?

**① 검색·필터 대상 데이터** — WHERE 절에 들어가면 안 됩니다.

```sql
meta JSONB  -- {"org_pk": 1, "user_pk": 10}
-- WHERE meta->>'org_pk' = ? → 일반 B-tree 인덱스로는 못 타고, 매번 표현식 평가 →
--   격리 키를 JSON에 숨기면 실수하기 쉽고 위험 → 전체 테이블 스캔 가능성
```

**규칙**: WHERE/JOIN에 들어가는 데이터는 반드시 별도 컬럼으로 정규화합니다.

**② 권한 데이터 (의도적 거부)** — `api_key.allowed_environments JSONB` 같은 설계는 거부했습니다.

```sql
-- 거부: 어떤 값이 허용되는지 코드로 읽기 어렵고, 잘못된 값도 DB가 막지 못하고,
--       권한 로직이 JSON 파싱 속에 숨어버림
allowed_environments JSONB
-- 대신: 구조화된 목록 + 앱 검증
allowed_services JSONB  -- ["ACADEMY", "MARKET"]
```

이것이 불변식 "임의 JSON 권한 블롭 금지"입니다.

> 💡 **한 줄 요약**: WHERE/JOIN 대상은 정규화하고, 권한 판단 데이터는 JSON에 넣지 않습니다. JSON은 "읽기 전용 블롭"으로만.

---

## Q3. PostgreSQL JSONB에 특별히 알아둘 점이 있나요?

PostgreSQL의 `JSONB`는 **GIN 인덱스 + `@>` 포함 연산자**로 임의 키 검색이 1순위로 잘 됩니다. 특정 경로 하나만 인덱스해야 하는 게 아니라, 컬럼 전체에 GIN을 걸어두면 다양한 키 조건을 받습니다.

```sql
-- PostgreSQL JSONB: 컬럼 전체에 GIN → @> 포함 검색
CREATE INDEX idx_entitlement_limits ON org_entitlement USING GIN (feature_limits);

-- "daily_uploads 키를 포함하고 그 값이 10인 행" 같은 포함 조건
SELECT * FROM org_entitlement
WHERE feature_limits @> '{"daily_uploads": 10}';
-- → 위 GIN 인덱스를 탐
```

> 🐬 **MySQL이라면**: JSON 경로 인덱스가 제한적이라, 보통 특정 경로를 가상 컬럼/표현식으로 뽑아 인덱스합니다 — 예: `ALTER TABLE org_entitlement ADD INDEX idx_limit_uploads ((CAST(feature_limits->>'$.daily_uploads' AS UNSIGNED)));`. PostgreSQL의 GIN `@>`만큼 임의 키 검색이 유연하지 않습니다.

그렇다고 "JSONB면 다 GIN으로 검색하자"는 아닙니다. **검색·필터·JOIN의 1순위 대상이나 테넌트 격리 키(`org_pk` 등)는 여전히 정규 컬럼으로** 둡니다. 이유는 ① 정규 컬럼 + B-tree가 등치·범위·정렬에서 더 빠르고 예측 가능하며 ② FK·NOT NULL·CHECK 같은 제약을 걸 수 있고 ③ 격리 키가 JSON에 숨으면 누락·실수로 [[multitenancy-rls|테넌트 경계]]가 무너지기 때문입니다. JSONB GIN은 "구조가 가변인 부가 데이터를 가끔 포함 검색"하는 보조 수단이지, 격리·권한 키를 대체하지 않습니다.

판단 기준 요약:

```
"이 데이터로 WHERE 필터·JOIN·정렬·격리를 할 것인가?"  예 → 별도 정규 컬럼 / 아니오 → JSONB 가능
"구조가 row마다 다른가?"                              예 → JSONB 적합 / 아니오 → 정규화 고려
"권한·테넌트 격리 판단에 쓰이는가?"                    예 → 정규 컬럼(JSON 금지) / 아니오 → 상황에 따라
```

> 💡 **한 줄 요약**: PostgreSQL JSONB는 GIN `@>`로 임의 키 검색이 1순위로 잘 되지만, WHERE/JOIN/격리·권한 키는 여전히 정규 컬럼으로 둡니다. JSONB GIN은 가변 부가 데이터의 보조 검색 수단입니다.

---

## 연결된 개념

- [[feature-limits]] — org_entitlement.feature_limits JSON 우선순위 설계
- [[multitenancy-rls]] — org_pk는 JSON이 아니라 별도 컬럼이어야 하는 이유(검색·격리)
> 소스 문서
- [[schema-reference]] — §D.12 feature_limits, §D.8 audit_log meta_json, §I.2 user_consent_event meta_json
- [[architecture]] — §2.1 불변식(임의 JSON 권한 블롭 금지), §1.4 MySQL vs PostgreSQL
