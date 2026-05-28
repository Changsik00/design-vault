# PK 설계 — BIGINT + ULID를 왜 두 개 두나

## 배경

DB 테이블에는 각 row를 고유하게 식별하는 PK(Primary Key)가 필요하다.

가장 흔한 방식:

```sql
CREATE TABLE user (
  id INT AUTO_INCREMENT PRIMARY KEY,  -- 1, 2, 3, 4, ...
  name VARCHAR(100)
);
```

이 `id`를 API 응답에 그대로 실어보내면 어떤 일이 생기는가?

---

## 문제 — 순번 PK를 그대로 노출하면

```
GET /users/1   → 사용자 김지영
GET /users/2   → 사용자 박민준
GET /users/3   → 사용자 ...
```

**공격 1 — Enumeration Attack**: `id`를 1씩 증가시키며 전체 사용자 목록 수집 가능.

**공격 2 — Business Intelligence 유출**:
```
경쟁사 직원이 처음 가입: user_id = 3,241
6개월 후 재가입:         user_id = 15,872
→ 6개월 동안 회원 12,631명 가입 추측
```

**공격 3 — IDOR(Insecure Direct Object Reference)**:
```
GET /orgs/1/lectures/42  → 내 강의
GET /orgs/1/lectures/43  → 타 학원 강의? (org_pk 검증 없으면 노출)
```

---

## 우리 결정 — 내부 PK와 외부 노출 ID를 분리

```sql
CREATE TABLE identity_user (
  pk        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,  -- 내부 전용
  public_id CHAR(26) NOT NULL,                           -- 외부 노출 전용 (ULID)
  ...
  UNIQUE KEY uq_identity_user_public_id (public_id)
);
```

```
내부 JOIN:  WHERE user_pk = 10          (BIGINT, 빠름)
API 응답:   { "id": "01HXYZ123ABC456789012345" }  (ULID, 안전)
API 요청:   GET /users/01HXYZ123ABC456789012345
```

---

## ULID란

**Universally Unique Lexicographically Sortable Identifier**

```
01ARZ3NDEKTSV4RRFFQ69G5FAV
└─────────┘└─────────────┘
  타임스탬프    랜덤
  (48비트)    (80비트)
```

- **26자**: UUID(36자)보다 짧고 URL-safe
- **시간순 정렬 가능**: 앞 10자가 타임스탬프 → `ORDER BY public_id` = 생성순
- **충돌 없음**: 같은 밀리초에 생성해도 랜덤 80비트가 충돌 확률 사실상 0
- **예측 불가**: 랜덤 80비트로 다음 값 예측 불가

**UUID v4 비교**:

| 항목 | UUID v4 | ULID |
|---|---|---|
| 정렬 가능 | 불가 (완전 랜덤) | 가능 (시간 prefix) |
| 인덱스 성능 | 나쁨 (랜덤 → B-tree 분산) | 좋음 (시간순 → locality) |
| 길이 | 36자 (`550e8400-e29b-41d4-...`) | 26자 (`01HXYZ123ABC...`) |

---

## 트레이드오프

| 항목 | 단일 AUTO_INCREMENT | BIGINT + ULID (우리) |
|---|---|---|
| 구현 단순성 | 컬럼 1개 | 컬럼 2개 |
| 노출 안전성 | 위험 | 안전 |
| JOIN 성능 | 빠름 | 빠름 (BIGINT 사용) |
| 정렬 가능성 | 가능 | 가능 (ULID 시간 prefix) |
| 비즈니스 지표 유출 | 가능 | 불가 |

---

## 핵심 규칙

- API 요청/응답, URL: `public_id` (ULID) 만 사용
- DB JOIN, FK 참조: `pk` (BIGINT) 만 사용
- `pk`를 로그 외부, API에 노출하면 안 됨

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §B` | 식별자 체계 전체 |
| `core/architecture.md §3.1` | 불변식 #2: 내부 PK는 BIGINT, 외부 노출은 ULID |
