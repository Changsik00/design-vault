---
tags:
  - platform-db
  - explainer
  - p0
  - security
  - bola
  - authz
  - multitenancy
aliases:
  - BOLA
  - 객체 수준 인가
  - Broken Object Level Authorization
  - IDOR
---

# BOLA (객체 수준 인가) 설명 — "로그인은 됐는데 남의 데이터가 보이는" 취약점

> **대상**: API 보안이 처음인 개발자 (공부용 — 사수와 함께 읽는 노트)
> **연관 문서**: [[architecture|architecture.md §2.2 보안]] · [[schema-reference|schema-reference.md §H.1]] · [[multitenancy-rls|멀티테넌시 격리]]

멀티테넌트 SaaS에서 **가장 흔하고, 가장 크게 터지는** API 취약점이 BOLA입니다. RBAC(역할 권한)을 아무리 잘 만들어도 BOLA가 뚫려 있으면 다른 학원 데이터가 그대로 새 나갑니다. 이 문서는 BOLA가 뭔지, 왜 위험한지, `platform_db`가 어떻게 막는지, 그리고 **어떻게 테스트로 잡는지**까지 공부하는 노트입니다.

---

## Q1. BOLA가 정확히 뭔가요?

**BOLA = Broken Object Level Authorization (객체 수준 인가 실패).** "객체(object)"는 데이터 한 건 — 학생 1명, 주문 1건, 인보이스 1장입니다. **그 한 건에 대해 "이 사람이 이걸 볼 자격이 있나?"를 검증하지 않은 상태**가 BOLA입니다.

가장 쉬운 예시:

```http
GET /students/123      ← 내 학원 학생. 정상.
GET /students/124      ← URL 숫자만 바꿈 → 남의 학원 학생이 보임 (!!)
GET /students/125
GET /students/126 ...
```

서버 코드가 이러면 BOLA입니다:

```ts
// ❌ BOLA — 객체가 누구 것인지 안 따짐
const student = await db.student.findById(studentId);
return student;   // studentId만 맞으면 누구든 조회됨
```

> 한 줄로: **"로그인은 했는데, 남의 데이터를 볼 수 있는 상태."**

BOLA는 **OWASP API Security Top 10의 1위**(API1:2023)입니다. 실제 대형 유출 사고의 다수가 여기서 납니다 — 고객정보·주문·결제내역·직원정보 조회 API가 단골입니다.

> 💡 **한 줄 요약**: BOLA는 "ID만 맞으면 남의 데이터 한 건이 그대로 나오는" 객체 단위 권한 누락이고, API 보안 1순위 위협입니다.

---

## Q2. 인증(로그인)은 통과했는데 왜 뚫리나요?

여기서 두 단어를 정확히 구분해야 합니다 — **인증(authN)**과 **인가(authZ)**.

| 구분 | 영어 | 질문 | BOLA에서 |
|---|---|---|---|
| 인증 | **Auth**e**n**tication | "너 **누구**냐?" | ✅ 통과함 (로그인 됨) |
| 인가 | **Auth**o**z**ation | "너 이거 **해도 되냐**?" | ❌ 빠짐 (객체 검증 없음) |

BOLA는 **인증은 멀쩡한데 인가가 빠진** 상태입니다. 그래서:

- **보안 스캐너가 못 잡습니다.** 응답이 `200 OK`로 정상이라 "에러"가 아닙니다. 그게 남의 데이터일 뿐이죠.
- **방화벽·WAF도 못 막습니다.** 정상 로그인 사용자의 정상 요청처럼 보입니다.
- 결국 **애플리케이션 로직만이** 막을 수 있습니다 — 자동 도구로 못 잡으니 설계·코드·테스트로 막아야 합니다.

```text
요청: GET /students/124  (유효한 토큰 첨부)
  ✅ 토큰 서명 OK       ← 인증 통과
  ✅ 토큰 만료 안 됨     ← 인증 통과
  ❓ 124가 네 학원 학생? ← 아무도 안 물어봄 = BOLA
  → 200 OK + 남의 데이터
```

> 💡 **한 줄 요약**: BOLA는 인증이 아니라 인가의 구멍이라, 정상 200 응답으로 위장돼 스캐너·WAF가 못 잡고 앱 코드만이 막을 수 있습니다.

---

## Q3. 왜 멀티테넌트 SaaS에서 제일 위험한가요?

멀티테넌트는 한 DB에 여러 학원 데이터가 **섞여** 있습니다([[multitenancy-rls|Pool 모델]]). BOLA 하나면 테넌트 경계가 통째로 무너집니다 — 한 학원 계정으로 **전체 학원 데이터**를 긁을 수 있게 됩니다.

흔한 착각:

```sql
-- "tenant_id(org_pk) 컬럼 넣었으니 안전하지?"  ← 아니요!
CREATE TABLE students ( ..., org_pk BIGINT NOT NULL );
```

**컬럼이 있는 것과, 쿼리에 강제되는 것은 다릅니다.** 진짜 방어는 컬럼 존재가 아니라 *모든 쿼리에 조건이 붙는 것*입니다:

```sql
-- ❌ BOLA — org_pk 컬럼은 있지만 WHERE에 안 씀
SELECT * FROM students WHERE student_id = ?;

-- ✅ 안전 — 객체 ID와 테넌트 경계를 함께 검증
SELECT * FROM students WHERE org_pk = ? AND student_id = ?;
--                            ^^^^^^^^^^ 이게 빠지면 BOLA
```

> 💡 **한 줄 요약**: `tenant_id` 컬럼이 있다고 안전한 게 아니라, **모든 조회·수정·삭제에 `WHERE org_pk = ?`가 강제돼야** 안전합니다.

---

## Q4. platform_db는 BOLA를 어떻게 막나요?

네 겹으로 막습니다(defense-in-depth, 심층 방어).

**① org_pk는 클라이언트가 주지 않는다 — 토큰에서 바인딩**

```ts
// ❌ 위험 — 클라이언트가 보낸 값 그대로 사용 (위조 가능)
const orgPk = req.body.orgPk;       // 999로 위조하면 끝

// ✅ 안전 — 서버가 토큰·멤버십에서 확정
//   x-org-pk 헤더로 org를 받더라도, membership 확인을 통과해야만 그 org로 동작
const membership = await getActiveMembership(user.pk, orgPk);
if (!membership) throw new ForbiddenException();   // 내 멤버십 아니면 차단
```

**② 모든 조회에 `org_pk` 강제 — base repository로 프레임워크화 (불변식 §2.2, SEC-1)**

개발자가 매번 손으로 `WHERE org_pk`를 붙이면 언젠가 빠뜨립니다. 그래서 **Drizzle base repo가 자동 주입**합니다 — "복합 UNIQUE 제약" 같은 게 아니라 *질의 필터*가 BOLA의 실체입니다.

```ts
const lecture = await db.query.lecture.findFirst({
  where: and(
    eq(lecture.pk, lectureId),   // ← 객체 ID
    eq(lecture.orgPk, orgPk),    // ← 테넌트 경계 (base repo가 강제)
  ),
});
if (!lecture) throw new NotFoundException();
```

**③ 없으면 403이 아니라 404 — 존재 자체를 숨긴다**

남의 학원 강의 ID로 접근하면 `403 Forbidden`이 아니라 **`404 Not Found`**를 줍니다. 403은 "있긴 한데 권한 없음"을 알려줘서 *그 ID가 실재한다*는 정보를 흘립니다. 404는 그조차 숨깁니다.

```ts
// 다른 org의 lecture.pk=999로 접근 → WHERE org_pk=내org AND pk=999 → row 없음 → 404
if (!lecture) throw new NotFoundException();  // "존재 여부 비공개"
```

**④ ULID `public_id`는 보조일 뿐, BOLA를 막지 않는다**

외부 노출 ID는 순차 정수(`123→124`)가 아니라 추측 불가한 [[pk-ulid-strategy|ULID]]입니다. 이건 *무작정 ID를 1씩 늘려 긁는* 단순 공격을 어렵게 합니다. 하지만 **ID 하나만 유출돼도(로그·리퍼러·공유 링크) 끝**이라, ULID는 방어가 아니라 *마찰*일 뿐입니다. **진짜 방어는 ①~③의 `org_pk` 강제**입니다.

> 💡 **한 줄 요약**: org_pk를 토큰에서 바인딩 + 모든 쿼리에 base repo가 강제 + 404로 존재 은닉 — 이 세 가지가 핵심이고, ULID는 보조 마찰일 뿐입니다.

---

## Q5. (★ 가장 중요한 오해) RBAC을 잘 만들면 BOLA도 막히지 않나요?

**아니요. RBAC과 BOLA 방어는 직교(orthogonal)합니다 — 서로 다른 축이라 하나로 다른 하나를 대신할 수 없습니다.**

| | 묻는 것 | 예 |
|---|---|---|
| **RBAC** (역할) | "이 **동작**을 할 자격이 있나?" (동사) | "TEACHER는 강의를 게시할 수 있나?" |
| **BOLA 방어** (객체 수준) | "이 **객체**가 네 거냐?" (목적어) | "이 강의가 **네 학원** 강의냐?" |

박교사가 `강의 게시` 권한(RBAC ✅)을 가졌더라도, **다른 학원의 강의**는 건드릴 수 없어야 합니다(BOLA 방어). RBAC만 통과시키고 객체 소유를 안 보면:

```ts
// RBAC은 통과했지만 ("TEACHER는 강의 수정 가능") BOLA는 뚫림
if (can(user, 'update', 'lecture')) {        // ✅ 역할 OK
  await lectureRepo.update(lectureId, data); // ❌ 그게 누구 학원 강의인지 안 봄
}
```

그래서 **RBAC을 완벽히 만들어도 BOLA는 별도로 막아야** 합니다. 이 설계에서 BOLA 방어가 RBAC보다 더 근본적인 이유입니다 — RBAC이 뚫리면 "권한 상승"이지만, BOLA가 뚫리면 "전 테넌트 유출"입니다.

> 💡 **한 줄 요약**: RBAC은 "무엇을 할 수 있나", BOLA 방어는 "그 객체가 네 거냐" — 직교하는 두 축이라 둘 다 통과해야 하고, RBAC이 BOLA를 대신하지 못합니다.

---

## Q6. IDOR랑 같은 건가요?

거의 같은 계열입니다.

- **IDOR** (Insecure Direct Object Reference) — 예전 용어. "`/users/123`을 `/users/124`로 바꿔 직접 참조"하는 취약점.
- **BOLA** — OWASP가 API 보안 맥락에서 더 넓게 재정의한 이름. IDOR을 포함하는 상위 개념.

실무에서는 **거의 동의어**로 보면 됩니다. 요즘 API 보안 문서·리포트는 BOLA를 씁니다.

> 💡 **한 줄 요약**: IDOR는 BOLA의 옛 이름이자 부분집합 — 같은 취약점으로 이해하면 됩니다.

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| **BOLA** | Broken Object Level Authorization. 객체(데이터 1건) 단위 권한 검증 누락. OWASP API #1 |
| **IDOR** | Insecure Direct Object Reference. BOLA의 옛 이름·부분집합 |
| **인증(authN)** | "너 누구냐" — 신원 확인 (Firebase 로그인) |
| **인가(authZ)** | "너 이거 해도 되냐" — 권한 확인 (3-gate) |
| **객체 수준 인가** | 데이터 *한 건*에 대한 인가. 컬렉션이 아니라 개별 row 단위 |
| **테넌트(tenant)** | 멀티테넌트에서 분리 단위 = `organization`(학원/회사) |
| **소유권(ownership)** | 그 객체가 특정 user/org에 귀속되는 관계 (ABAC의 한 형태) |
| **BFLA** | Broken Function Level Authorization. 객체가 아니라 *기능/엔드포인트* 단위 인가 누락(OWASP API #5). BOLA의 형제 |
| **defense-in-depth** | 심층 방어. 한 겹이 뚫려도 다음 겹이 막도록 여러 층으로 방어 |
| **negative test** | "되면 안 되는 게 안 되는지" 확인하는 테스트. BOLA 방어 검증의 핵심 |
| **enumeration** | ID를 1씩 늘려가며 긁는 공격(`123→124→125`). ULID가 어렵게 함 |

---

## 테스트 방법

BOLA는 **자동 스캐너가 못 잡으니 직접 테스트를 짜야** 합니다. 핵심은 **negative test** — "되면 안 되는 게 정말 안 되는가".

**① 통합 테스트 (vitest + supertest) — 두 테넌트를 만들고 교차 접근을 막는다**

```ts
// lecture.bola.spec.ts
describe('BOLA — 강의 객체 수준 인가', () => {
  let orgA, orgB, lectureInA, tokenB;

  beforeEach(async () => {
    orgA = await createOrg('한울학원');
    orgB = await createOrg('B학원');
    lectureInA = await createLecture(orgA, { title: '중1 수학' }); // A 소유
    tokenB = await loginAs(orgB.owner);                            // B로 로그인
  });

  it('다른 학원(B)이 A의 강의 ID로 조회하면 404 — 403 아님(존재 은닉)', async () => {
    const res = await request(app)
      .get(`/v1/lectures/${lectureInA.publicId}`)
      .set('Authorization', `Bearer ${tokenB}`)
      .set('x-org-pk', orgB.pk);
    expect(res.status).toBe(404);   // ★ 핵심 단언
    expect(res.status).not.toBe(200);
  });

  it('B가 A의 강의를 수정 시도해도 404 (RBAC 통과해도 객체 소유 차단)', async () => {
    const res = await request(app)
      .patch(`/v1/lectures/${lectureInA.publicId}`)
      .set('Authorization', `Bearer ${tokenB}`)
      .set('x-org-pk', orgB.pk)
      .send({ title: '탈취 시도' });
    expect(res.status).toBe(404);
  });
});
```

**② 핵심 원칙 — 모든 `:id` 엔드포인트에 "교차 테넌트 404" 테스트를 강제**

BOLA는 한 군데만 빠져도 터집니다. 그래서 *엔드포인트마다 negative test가 있는지*를 매트릭스로 관리합니다(우리 [[bdd-scenarios]] 추적 매트릭스·[[e2e-journeys]] D3-02와 같은 방식):

```ts
// 데이터 기반 테스트 — :id 받는 모든 엔드포인트를 한 번에
const PROTECTED = [
  { method: 'get',   path: (id) => `/v1/lectures/${id}` },
  { method: 'patch', path: (id) => `/v1/members/${id}/role` },
  { method: 'get',   path: (id) => `/v1/invoices/${id}` },
  // ... :id 받는 엔드포인트 전부 등록
];
it.each(PROTECTED)('$method $path 는 타 org 객체에 404', async ({ method, path }) => {
  const res = await request(app)[method](path(lectureInA.publicId))
    .set('Authorization', `Bearer ${tokenB}`).set('x-org-pk', orgB.pk);
  expect([403, 404]).toContain(res.status);
  expect(res.status).not.toBe(200);   // 절대 허용으로 새면 안 됨
});
```

**③ e2e 블랙박스 — 사용자 여정으로 검증**

[[e2e-journeys|e2e 여정]]의 **C-05**(멀티 워크스페이스: org 헤더 바꿔도 404)와 **D3-02**(타 org 멤버 ID로 작업 → 404)가 BOLA의 black-box 검증입니다. API 경계에서 "같은 ID라도 org가 다르면 안 보인다"를 확인합니다.

**④ 무엇을 단언하나 (체크리스트)**

```text
□ 타 org 객체 조회 → 404 (200 아님, 403도 아님)
□ 타 org 객체 수정/삭제 → 404 (RBAC 권한이 있어도)
□ org_pk를 body/header로 위조 → 내 멤버십 없으면 차단
□ ULID가 아닌 순차 ID 추측 시도 → 어차피 org_pk로 막힘
□ "정상 200"이 절대 남의 데이터가 아님을 응답 내용으로 확인
```

> 💡 **테스트 한 줄 요약**: "되면 안 되는 게 안 되는가"를 모든 `:id` 엔드포인트에서 *교차 테넌트 404* negative test로 강제하세요. 스캐너는 못 잡습니다.

---

## 마치며

BOLA는 결국 한 질문으로 압축됩니다: **"이 요청이 가리키는 객체가, 정말 이 사람 테넌트의 것인가?"**

이 설계에서 "예"라고 답하려면:

1. `org_pk`는 클라이언트가 아니라 **토큰·멤버십에서** 확정한다.
2. 모든 조회/수정/삭제에 `WHERE org_pk = ?`가 **base repo로 강제**된다.
3. 없으면 **404**(존재 은닉)로 응답한다.
4. RBAC을 통과해도 **객체 소유는 별도로** 검증한다.
5. 모든 `:id` 엔드포인트에 **교차 테넌트 404 테스트**가 있다.

RBAC을 멋지게 만드는 것보다, `:id`를 받는 모든 API에서 BOLA를 막는 것이 멀티테넌트 SaaS에선 더 근본적입니다. 새 엔드포인트를 만들 때마다 "이거 BOLA 테스트 있나?"를 먼저 물어보세요.

---

## 연결된 개념

- [[multitenancy-rls|Pool 모델 멀티테넌시와 RLS]] — `org_pk` 행 격리가 BOLA 방어의 토대
- [[fail-closed|fail-closed 원칙]] — 의심스러우면 막는다(404/403/401), 허용으로 새지 않는다
- [[role-capability|role 2단 분리와 capability]] — RBAC/ReBAC 축(BOLA와 직교)
- [[pk-ulid-strategy|BIGINT pk + ULID public_id]] — 외부 ID가 순차가 아닌 이유(보조 마찰)
- [[gate-abc-flow|Gate A/B/C 흐름]] — org_pk 바인딩이 일어나는 위치
> 소스 문서
- [[architecture]] — §2.2 보안 규율(OWASP BOLA, org_pk 질의 강제 프레임워크화)
- [[schema-reference]] — §H.1 BOLA 방어 패턴, §G 멀티테넌시 격리
- [[requirements]] — SEC-1(BOLA 방어), TEN-2(모든 조회 WHERE org_pk·타 org 404)
- [[e2e-journeys]] — C-05·D3-02 (BOLA black-box 검증)
