---
difficulty: 초
tags:
  - platform-db
  - explainer
  - p1
  - schema
  - mysql
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

MySQL은 JSON 타입 컬럼을 지원합니다. 스키마 없이 유연하게 저장할 수 있어 매력적이지만, "언제나 JSON"은 나쁜 선택입니다. 어디에 쓰고 어디에 쓰면 안 되는지 설명합니다.

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
meta JSON  -- {"org_pk": 1, "user_pk": 10}
-- WHERE org_pk = ? → 인덱스 불가 → 전체 테이블 스캔 → 치명적 성능 문제
```

**규칙**: WHERE/JOIN에 들어가는 데이터는 반드시 별도 컬럼으로 정규화합니다.

**② 권한 데이터 (의도적 거부)** — `api_key.allowed_environments JSON` 같은 설계는 거부했습니다.

```sql
-- 거부: 어떤 값이 허용되는지 코드로 읽기 어렵고, 잘못된 값도 DB가 막지 못하고,
--       권한 로직이 JSON 파싱 속에 숨어버림
allowed_environments JSON
-- 대신: 구조화된 목록 + 앱 검증
allowed_services JSON  -- ["ACADEMY", "MARKET"]
```

이것이 불변식 "임의 JSON 권한 블롭 금지"입니다.

> 💡 **한 줄 요약**: WHERE/JOIN 대상은 정규화하고, 권한 판단 데이터는 JSON에 넣지 않습니다. JSON은 "읽기 전용 블롭"으로만.

---

## Q3. MySQL JSON에 특별히 주의할 점이 있나요?

MySQL의 JSON 경로 인덱스는 **제한적**입니다. PostgreSQL JSONB의 GIN 인덱스만큼 유연하지 않습니다.

```sql
-- MySQL: 특정 경로에만 인덱스 가능
ALTER TABLE org_entitlement
  ADD INDEX idx_limit_uploads ((CAST(feature_limits->>'$.daily_uploads' AS UNSIGNED)));

-- PostgreSQL JSONB: 임의 키로 @> 검색 가능
CREATE INDEX ON org_entitlement USING GIN (feature_limits);
```

그래서 MySQL에서 JSON을 **검색 조건에 자주 쓰면** 성능 문제가 생깁니다. 읽기 전용 블롭으로 취급하는 게 안전합니다.

판단 기준 요약:

```
"이 데이터로 WHERE 필터나 JOIN을 할 것인가?"  예 → 별도 컬럼으로 정규화 / 아니오 → JSON 가능
"구조가 row마다 다른가?"                      예 → JSON 적합 / 아니오 → 정규화 고려
"권한 판단에 쓰이는가?"                        예 → JSON 금지 / 아니오 → 상황에 따라
```

> 💡 **한 줄 요약**: MySQL JSON 인덱스는 제한적이라 검색 조건엔 부적합합니다. "WHERE로 쓸 건가, 구조가 가변인가, 권한인가" 세 질문으로 판단하세요.

---

## 연결된 개념

- [[feature-limits]] — org_entitlement.feature_limits JSON 우선순위 설계
- [[multitenancy-rls]] — org_pk는 JSON이 아니라 별도 컬럼이어야 하는 이유(검색·격리)
> 소스 문서
- [[schema-reference]] — §D.12 feature_limits, §D.8 audit_log meta_json, §I.2 user_consent_event meta_json
- [[architecture]] — §2.1 불변식(임의 JSON 권한 블롭 금지), §1.4 MySQL vs PostgreSQL
