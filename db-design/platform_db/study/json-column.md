# JSON 컬럼 — 언제 쓰고 언제 정규화하나

## 배경

MySQL 5.7부터 JSON 타입 컬럼을 지원한다.

```sql
CREATE TABLE something (
  meta JSON  -- 아무 구조나 저장 가능
);

INSERT INTO something VALUES ('{"key": "value", "count": 42, "tags": ["a", "b"]}');
```

스키마 없이 유연하게 저장할 수 있다는 게 매력적이다. 그러나 "언제나 JSON"은 나쁜 선택이다.

---

## JSON을 쓰면 좋은 경우

### 1. 구조가 제품/플랜마다 다른 경우

```sql
-- org_entitlement.feature_limits
-- academy 플랜: {"daily_uploads": 6, "members": 50, "storage_gb": 10}
-- market 플랜:  {"listings": 100, "featured_slots": 5}
-- agent 플랜:   {"monthly_tokens": 1000000, "concurrent_agents": 3}
```

product마다 한도 항목이 다르다. 정규화하면:
```sql
-- 정규화 시 이렇게 됨
CREATE TABLE feature_limit_value (
  entitlement_pk BIGINT, feature_key VARCHAR(50), limit_value BIGINT
);
-- 쿼리: SELECT limit_value WHERE feature_key = 'daily_uploads'
-- → JOIN 추가, 쿼리 복잡도 증가 vs JSON은 단순 SELECT
```

한도 읽기는 항상 전체를 읽고 앱에서 해석 → JSON이 적합.

### 2. 감사/이벤트 메타데이터

```sql
-- audit_log.meta_json
-- API 호출별로 기록 내용이 다름
{"http_method": "POST", "request_body_hash": "abc123", "user_agent": "..."}
{"reason": "permission_denied", "required_role": "TEACHER", "actual_role": "STUDENT"}
```

이벤트마다 다른 컨텍스트 정보 → 정규화 불필요, JSON이 적합.

### 3. 제3자 제공 4요건 (PIPA §17)

```sql
-- user_consent_event.meta_json (제3자 제공 시)
{
  "recipient": "Google Firebase",
  "purpose": "이메일/비밀번호 인증",
  "items": ["email", "password_hash"],
  "retention": "서비스 탈퇴 후 30일"
}
```

항목이 동의 종류마다 다름 → JSON으로 자유형 저장.

---

## JSON을 쓰면 안 되는 경우

### 1. 검색·필터 대상 데이터

```sql
-- 나쁜 예: org_pk를 JSON에 넣으면
meta JSON  -- {"org_pk": 1, "user_pk": 10}

-- WHERE org_pk = ? → 인덱스 불가 (MySQL JSON 경로 인덱스는 제한적)
-- 전체 테이블 스캔 → 치명적 성능 문제
```

**규칙**: WHERE 절에 들어가는 데이터는 반드시 별도 컬럼으로.

### 2. 권한 데이터 (의도적 거부)

```sql
-- 거부한 설계: api_key.allowed_environments JSON
allowed_environments JSON  -- {"env": ["production", "staging"]}

-- 이렇게 하면 안 됨:
-- - 어떤 값이 허용되는지 코드 읽기 어려움
-- - 잘못된 값 넣어도 DB가 막지 않음
-- - 권한 로직이 JSON 파싱에 숨어버림
```

대신:
```sql
allowed_services JSON  -- ["ACADEMY", "MARKET"] — 구조화된 서비스 목록
-- 허용값 목록이 명확하고 앱에서 검증
```

---

## MySQL JSON의 한계

```sql
-- MySQL: JSON 경로 인덱스 (제한적)
ALTER TABLE org_entitlement
  ADD INDEX idx_limit_uploads ((CAST(feature_limits->>'$.daily_uploads' AS UNSIGNED)));
-- 특정 경로에만 인덱스 가능, GIN처럼 유연하지 않음

-- PostgreSQL JSONB: GIN 인덱스 (훨씬 강력)
CREATE INDEX ON org_entitlement USING GIN (feature_limits);
-- 임의 키로 @> 연산자 검색 가능
```

MySQL에서 JSON을 **검색 조건에 자주 쓰면** 성능 문제가 생긴다. 읽기 전용 블롭으로 취급.

---

## 판단 기준 요약

```
"이 데이터로 WHERE 필터나 JOIN을 할 것인가?"
  예 → 반드시 별도 컬럼으로 정규화
  아니오 → JSON 가능

"구조가 row마다 다른가?"
  예 → JSON 적합
  아니오 → 정규화 고려

"권한 판단에 쓰이는가?"
  예 → JSON 금지 (불변식: 임의 JSON 권한 블롭 금지)
  아니오 → 상황에 따라
```

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §D.12` | feature_limits JSON 컬럼 |
| `core/schema-reference.md §D.8` | audit_log meta_json |
| `core/schema-reference.md §I.2` | user_consent_event meta_json |
| `core/architecture.md §3.1` | 불변식 #6: 임의 JSON 권한 블롭 금지 |
