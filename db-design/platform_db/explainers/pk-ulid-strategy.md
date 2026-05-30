---
tags:
  - platform-db
  - explainer
  - p0
  - identity
  - ulid
  - pk
aliases:
  - ULID
  - public_id
  - BIGINT pk
  - 식별자 전략
---

# BIGINT pk + ULID public_id 전략 설명

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[schema-reference|schema-reference.md §B]] · [[architecture|architecture.md §2.1 불변식 #2]]

`platform_db`의 모든 테이블에는 `pk`와 `public_id`가 함께 존재합니다. 처음 스키마를 보면 "왜 ID가 두 개야?"라는 의문이 자연스럽게 생깁니다. 이 문서는 그 이유와 실제 사용 방법을 설명합니다.

---

## Q1. 테이블마다 `pk`랑 `public_id`가 두 개씩 있는데, 뭘 언제 쓰면 되나요?

간단하게 말하면 이렇습니다.

| 상황 | 사용하는 ID |
|---|---|
| 테이블 JOIN, 서버 내부 로직 | `pk` (BIGINT) |
| API 응답, URL 경로, 클라이언트 전달 | `public_id` (ULID) |

`identity_user` 테이블을 예로 보면:

```sql
CREATE TABLE identity_user (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,  -- DB 내부 전용
  public_id   CHAR(26) NOT NULL,  -- 외부 노출 전용 (ULID)
  firebase_uid VARCHAR(128),
  email       VARCHAR(255),
  ...
);
```

코드로 표현하면 이런 구분입니다:

```typescript
// DB 조회: 내부에서는 pk로 JOIN
const membership = await db.query.membership.findFirst({
  where: and(
    eq(membership.userPk, user.pk),   // ← BIGINT pk 사용
    eq(membership.orgPk, org.pk)
  )
});

// API 응답: 클라이언트에게는 public_id만 노출
return {
  id: user.publicId,     // ← ULID 반환 ("01HXYZ...")
  email: user.email,
  // pk: user.pk ← 이렇게 하면 안 됨!
};

// URL 경로: public_id 사용
// GET /users/01HXYZ...  ← ULID
// GET /users/42         ← 이렇게 하면 안 됨!
```

마치 도서관 카드처럼 생각하면 됩니다. 사서가 내부 시스템에서 책을 찾을 때는 내부 관리 번호(pk)를 씁니다. 하지만 대출 영수증에는 고객이 알아볼 수 있는 이용자 번호(public_id)를 적어줍니다. 내부 관리 번호를 영수증에 적으면 보안 문제가 생기겠죠.

> 💡 **한 줄 요약**: `pk`는 DB 안에서만, `public_id`는 API/URL 등 DB 밖으로 나갈 때만 사용합니다.

---

## Q2. ULID가 뭔가요? UUID랑 다른 건가요?

ULID(Universally Unique Lexicographically Sortable Identifier)는 UUID의 단점을 보완한 ID 형식입니다.

```
ULID 예시: 01ARZ3NDEKTSV4RRFFQ69G5FAV
           ├── 앞 10자: 타임스탬프 (밀리초)
           └── 뒤 16자: 랜덤값
```

UUID와 비교하면:

| 항목 | UUID v4 | ULID |
|---|---|---|
| 형식 | `550e8400-e29b-41d4-a716-446655440000` | `01ARZ3NDEKTSV4RRFFQ69G5FAV` |
| 길이 | 36자 (하이픈 포함) | 26자 |
| 정렬 | 완전 랜덤, 정렬 불가 | 생성 시간 순 정렬 가능 |
| 단조 증가 | ❌ | ✅ (같은 밀리초면 랜덤 증가) |
| URL 친화적 | ❌ (하이픈) | ✅ |
| 언제 만들어졌는지 | 모름 | 앞 10자로 대략 알 수 있음 |

UUID v4는 완전 랜덤이라 DB 인덱스 성능이 떨어집니다. 새 row가 인덱스 어디에나 꽂힐 수 있어서 B-tree 페이지 분할이 자주 발생합니다. 반면 ULID는 시간 순으로 정렬되기 때문에 새 row가 항상 인덱스 끝에 추가됩니다. 마치 잘 정리된 파일 서랍에 날짜 순으로 서류를 꽂는 것처럼요.

코드에서는 `ulid()` 함수로 생성합니다:

```typescript
import { ulid } from 'ulid';

const newUser = await db.insert(identityUser).values({
  publicId: ulid(),       // 예: "01HXYZ3NDEKTSV4RRFFQ69G5"
  email: 'user@example.com',
  // pk는 AUTO_INCREMENT라 명시하지 않음
});
```

> 💡 **한 줄 요약**: ULID는 UUID보다 짧고, URL에 바로 쓸 수 있으며, 시간 순 정렬이 가능한 고유 ID입니다.

---

## Q3. API 응답에 pk(숫자)를 그냥 쓰면 안 되나요? 왜 귀찮게 ULID로 바꿔야 해요?

숫자 pk를 외부에 노출하면 크게 두 가지 문제가 생깁니다.

**문제 1: 정보 유출 (열거 공격)**

```
GET /organizations/1     → 행복학원 본사
GET /organizations/2     → 강남 수학학원
GET /organizations/3     → 목동 영어학원
...
GET /organizations/9999  → ???
```

pk가 1부터 순서대로 올라가는 숫자라면, 공격자는 1부터 9999까지 순서대로 요청을 보내서 시스템에 있는 모든 org 목록을 뽑아낼 수 있습니다. 존재 여부 자체가 비즈니스 정보인데 다 노출되는 거죠.

ULID를 사용하면:

```
GET /organizations/01HXYZ3NDEK...  → 예측 불가, 순차 열거 불가
GET /organizations/01HABC1XQWE...  → 다음 ID를 추측할 수 없음
```

**문제 2: 내부 규모 노출**

사용자 ID가 `42`라면 "이 서비스에는 사용자가 42명 이상 있구나"를 경쟁사가 바로 알 수 있습니다. 예컨대 경쟁사 직원이 처음 가입했을 때 `user_id = 3,241`, 6개월 뒤 다시 가입했을 때 `user_id = 15,872`라면, 그 사이 약 12,631명이 가입했다고 추측할 수 있습니다. 반면 ULID라면 아무 정보도 노출되지 않습니다.

**문제 3: IDOR(Insecure Direct Object Reference)**

```
GET /orgs/1/lectures/42  → 내 강의
GET /orgs/1/lectures/43  → 타 학원 강의? (org_pk 검증이 없으면 노출)
```

순번 리소스 ID는 "옆 번호"를 추측하기 쉬워서, 권한 검증이 한 군데라도 빠지면 곧장 타 테넌트 데이터 노출로 이어집니다.

이것은 OWASP API Top 10 중 1위인 **BOLA(Broken Object Level Authorization)** 취약점과도 직결됩니다. `platform_db`의 설계 불변식 #2가 바로 이 규칙을 강제합니다:

```
불변식 #2: 내부 PK는 BIGINT, 외부 노출은 ULID(public_id).
           시퀀셜 PK를 URL·API에 노출 금지.
```

> 💡 **한 줄 요약**: 숫자 pk를 외부에 노출하면 공격자가 순서대로 모든 리소스를 열거할 수 있어서, 예측 불가능한 ULID로 교체합니다.

**단일 PK vs BIGINT + ULID 트레이드오프**

| 항목 | 단일 AUTO_INCREMENT | BIGINT + ULID (우리) |
|---|---|---|
| 구현 단순성 | 컬럼 1개 | 컬럼 2개 |
| 노출 안전성 | 위험 | 안전 |
| JOIN 성능 | 빠름 | 빠름 (BIGINT 사용) |
| 정렬 가능성 | 가능 | 가능 (ULID 시간 prefix) |
| 비즈니스 지표 유출 | 가능 | 불가 |

컬럼 하나가 늘어나는 비용을 치르는 대신, 내부 JOIN 성능(BIGINT)과 외부 노출 안전성(ULID)을 모두 얻습니다.

---

## Q4. DB 내부 JOIN은 pk로, 외부 노출은 public_id로 — 실제 코드에서 어떻게 구분해서 쓰나요?

전형적인 흐름은 이렇습니다: 클라이언트에서 `public_id`가 들어오면, 먼저 이걸 내부 `pk`로 변환한 다음 내부 로직을 처리합니다.

```typescript
// 1. 클라이언트 요청: public_id로 들어옴
// GET /orgs/01HXYZ.../lectures/01HABC...

async function getLecture(orgPublicId: string, lecturePublicId: string) {
  
  // 2. public_id → pk 변환 (내부 ID 획득)
  const org = await db.query.organization.findFirst({
    where: eq(organization.publicId, orgPublicId),
    columns: { pk: true }  // pk만 필요
  });
  if (!org) throw new NotFoundException();

  // 3. 내부 로직은 모두 pk 사용 (JOIN, 권한 체크 등)
  const membership = await getActiveMembership(userPk, org.pk);  // ← pk
  const entitlement = await getEntitlementByService(org.pk, 'ACADEMY');  // ← pk
  
  const lecture = await db.query.lecture.findFirst({
    where: and(
      eq(lecture.publicId, lecturePublicId),
      eq(lecture.orgPk, org.pk)  // ← pk로 [[multitenancy-rls|멀티테넌시]] 격리
    )
  });

  // 4. 응답: public_id로 변환해서 반환
  return {
    id: lecture.publicId,      // ← ULID 반환
    orgId: org.publicId,       // ← ULID 반환
    title: lecture.title,
    // orgPk: org.pk ← 절대 안 됨!
  };
}
```

ASCII 흐름도로 표현하면:

```
클라이언트                 서버                     DB
    │                       │                        │
    │  public_id 전달        │                        │
    │ ─────────────────────▶ │                        │
    │                        │ public_id → pk 조회    │
    │                        │ ──────────────────────▶ │
    │                        │ ◀────────── pk 반환 ── │
    │                        │ (이제 내부는 pk만 사용)  │
    │                        │ pk로 JOIN, 권한 체크    │
    │                        │ ──────────────────────▶ │
    │                        │ ◀────── 결과 반환 ───── │
    │                        │ pk → public_id 변환     │
    │  public_id 포함 응답   │                        │
    │ ◀───────────────────── │                        │
```

**PR 리뷰 체크포인트**: API 응답 객체에 숫자로 보이는 ID 필드(`userId: 42` 같은 것)가 있으면 바로 지적하세요. `BIGINT pk`가 슬쩍 나가고 있는 겁니다.

> 💡 **한 줄 요약**: 경계에서 한 번 변환합니다. 들어올 때 `public_id → pk`, 나갈 때 `pk → public_id`.

---

## Q5. firebase_uid는 또 뭔가요? pk, public_id, [[gate-abc-flow|Gate 흐름]]에서 firebase_uid — 세 개나 있는 이유가 있나요?

각각 완전히 다른 역할을 합니다:

```
identity_user 테이블
├── pk           BIGINT   → DB 내부 조인 전용
├── public_id    CHAR(26) → API/URL 외부 노출 전용
└── firebase_uid VARCHAR  → Firebase Auth 연결 키 전용
```

역할 분리를 표로 보면:

| 필드 | 역할 | 누가 만드나 | PK/FK? |
|---|---|---|---|
| `pk` | DB 내부 조인 기준 | MySQL AUTO_INCREMENT | PK ✅ |
| `public_id` | API/URL 외부 식별 | 앱 서버 (`ulid()`) | 고유키 ✅ |
| `firebase_uid` | Firebase Auth 연결 키 | Firebase | PK/FK ❌ (조회 키만) |

**왜 firebase_uid를 PK로 안 쓰나요?**

`firebase_uid`는 `"U2FsdGVkX19..."` 같은 Firebase가 발급한 문자열입니다. 이걸 DB의 PK로 쓰면 몇 가지 문제가 생깁니다:

1. **벤더 락인**: Firebase를 교체하거나 병행 인증(카카오 로그인, Apple 로그인 등)을 추가하면 PK 체계 자체가 무너집니다.
2. **JOIN 성능**: BIGINT JOIN이 VARCHAR JOIN보다 훨씬 빠릅니다. PK가 문자열이면 모든 JOIN이 느려집니다.
3. **길이 불확실**: Firebase UID 길이가 스펙상 128자까지 가능한데, BIGINT처럼 고정 크기가 아닙니다.

실제 코드에서 firebase_uid는 로그인 시 "이 Firebase 계정이 우리 DB의 어떤 사용자냐"를 찾는 조회 키로만 씁니다:

```typescript
// 로그인 플로우
async function resolveUser(firebaseUid: string) {
  // firebase_uid로 조회 → 내부 user 획득
  const user = await db.query.identityUser.findFirst({
    where: eq(identityUser.firebaseUid, firebaseUid)  // 조회 키로만 사용
  });
  
  if (!user) throw new UnauthorizedException();
  
  // 이후 모든 로직은 user.pk 사용
  return user.pk;  // ← BIGINT pk로 변환
}
```

정리하면 세 ID의 관계는 이렇습니다:

```
Firebase ──[firebase_uid]──▶ identity_user.firebase_uid (조회 키)
                                       │
                                       │ 같은 row
                              pk       │  public_id
                           (내부용) ◀──┤──▶ (외부용)
```

불변식 #1에 이 내용이 명시되어 있습니다:

```
불변식 #1: Firebase = 인증, 인가 = 우리 DB.
           firebase_uid는 조회 키일 뿐 PK/FK 아님.
```

> 💡 **한 줄 요약**: `pk`는 DB 조인용, `public_id`는 API 노출용, `firebase_uid`는 Firebase 연결용 — 세 개가 각자 다른 역할을 맡아 충돌 없이 공존합니다.

---

## 마치며

`pk`/`public_id`/`firebase_uid` 세 필드의 역할 분리는 처음엔 번거로워 보이지만, 실제로는 각각 명확한 이유가 있는 설계입니다.

- **pk**: DB 성능 (BIGINT JOIN이 가장 빠름)
- **public_id**: 보안 (예측 불가능한 ULID로 열거 공격 방지)
- **firebase_uid**: 인증 유연성 (Firebase를 교체하거나 멀티 제공자를 붙여도 DB 구조 불변)

코드를 작성할 때 가장 실수하기 쉬운 부분은 API 응답에 `pk`가 슬쩍 섞이는 것입니다. 응답 DTO를 작성할 때 `id` 필드가 숫자인지 문자열 ULID인지 한 번 더 확인하는 습관이 중요합니다.

세 필드는 `schema-reference.md §B` 식별자 체계 표에 정리되어 있고, 불변식 #1·#2는 `architecture.md §2.1`에서 PR 반려 기준으로 명시되어 있습니다.

---

## 연결된 개념

- [[gate-abc-flow|Gate A/B/C 전체 흐름]] — firebase_uid → user_pk 변환이 Gate 흐름의 시작점
- [[multitenancy-rls|Pool 모델 + RLS]] — org_pk가 멀티테넌시 격리 키인 이유
> 소스 문서
- [[architecture]] — §2.1 불변식 #2 (내부 PK는 BIGINT, 외부 노출은 ULID)
- [[schema-reference]] — B. 식별자 체계, D.1 identity_user DDL
