---
difficulty: 고
tags:
  - platform-db
  - explainer
  - p0
  - auth
  - rebac
  - delegation
  - relationship
aliases:
  - ReBAC
  - 위임
  - delegation_grant
  - 관계 기반 권한
---

# ReBAC와 위임(delegation_grant) — 관계 기반 권한

> **대상**: DB 지식이 많지 않은 개발자 (공부용)  
> **연관 문서**: [[architecture|architecture.md]] §1.2 RBAC/ABAC/ReBAC · [[schema-reference|schema-reference.md]] §D.6 delegation_grant, §D.7 org_relation, §E.4 위임 흐름 · [[requirements|requirements.md]] REBAC-1~8

권한 모델을 공부하다 보면 RBAC(역할), ABAC(속성)는 자주 보이는데 **ReBAC**는 낯섭니다. `platform_db`의 `delegation_grant` 테이블이 바로 이 ReBAC를 구현한 것입니다. role/capability의 *기본 개념*은 [[role-capability]]에서 다뤘으니, 이 문서는 **"관계로 권한을 정한다는 게 뭔지"**, 그리고 **위임(delegation)이 왜 임퍼소네이션과 다른지**에 집중합니다.

---

## Q1. ReBAC가 뭔가요? RBAC·ABAC는 들어봤는데 관계 기반은 처음이에요.

**ReBAC(Relationship-Based Access Control)**는 "두 주체(subject) 사이에 *어떤 관계*가 있는가"로 권한을 결정하는 모델입니다. 핵심 문장은 이겁니다:

> **"A가 B에게 이 capability를 위임했다"** — 이 *관계*가 존재하면 B는 그 권한을 행사할 수 있다.

세 모델을 한 줄로 비교하면 이렇게 갈립니다.

```
RBAC  → "너는 어떤 역할인가?"        (TEACHER이면 영상 업로드 가능)
ABAC  → "이 리소스의 속성이 어떤가?"  (이 강의의 owner가 나인가? 같은 org_pk인가?)
ReBAC → "너와 나 사이에 관계가 있나?"  (김 강사가 나에게 승인 권한을 위임했나?)
```

비유하자면 회사 출입을 생각해보세요.

- **RBAC**: "정직원 카드를 가졌으니 사무실 출입 가능" (역할 = 카드 등급)
- **ABAC**: "이 방은 *3층 부서 소속*만 들어갈 수 있음" (리소스 속성 + 사람 속성 매칭)
- **ReBAC**: "박 대리가 김 과장에게 *오늘 회의실 예약 권한을 위임*했음" (두 사람 사이의 관계)

ReBAC가 빛나는 지점은 RBAC로는 표현이 어색한 경우입니다. "김 강사가 *자기 권한 중 일부*를 박 조교에게 *한시적으로* 넘기고 싶다"는 요구는 새 역할을 만들 일이 아닙니다 — *그 두 사람 사이의 관계*일 뿐입니다. 그래서 `platform_db`는 RBAC/ABAC에 더해 ReBAC를 한 축으로 둡니다.

> 💡 **한 줄 요약**: ReBAC는 역할(RBAC)이나 속성(ABAC)이 아니라 "A가 B에게 위임했다"는 *관계*로 권한을 정하는 모델이고, 우리는 `delegation_grant` 테이블로 이를 구현합니다.

---

## Q2. delegation_grant 테이블은 어떻게 생겼나요? 각 컬럼이 무슨 뜻인가요?

`platform_db`의 ReBAC는 `delegation_grant` 한 테이블로 표현됩니다([[schema-reference|§D.6]]).

```sql
CREATE TABLE delegation_grant (
  pk          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  grantor_pk  BIGINT NOT NULL,           -- 권한을 "주는" 사람 (grantor)
  grantee_pk  BIGINT NOT NULL,           -- 권한을 "받는" 사람 (grantee)
  org_pk      BIGINT NOT NULL,           -- 어느 조직 안에서의 위임인가 (테넌트 경계)
  capability  VARCHAR(100) NOT NULL,     -- 무엇을 위임하나 ('ACADEMY.<action>')
  scope_json  JSONB,                     -- 범위 제한 (예: 특정 반/기간만)
  status      VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'  -- 살아있나/회수됐나
                CHECK (status IN ('ACTIVE','REVOKED')),
  expires_at  TIMESTAMPTZ,               -- 언제 자동 만료되나
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT chk_capability CHECK (capability IN (
    'ACADEMY.PUBLISH_VIDEO','ACADEMY.APPROVE_VIDEO','ACADEMY.VIEW_ALL_LECTURES',
    'ACADEMY.MANAGE_SCHEDULE','ACADEMY.MANAGE_MEMBERS','ACADEMY.VIEW_BILLING'
  )),
  CONSTRAINT chk_no_self_delegation CHECK (grantor_pk <> grantee_pk),  -- 자기위임 차단
  CONSTRAINT fk_grant_org      FOREIGN KEY (org_pk)     REFERENCES organization(pk),
  CONSTRAINT fk_grant_grantor  FOREIGN KEY (grantor_pk) REFERENCES identity_user(pk),
  CONSTRAINT fk_grant_grantee  FOREIGN KEY (grantee_pk) REFERENCES identity_user(pk)
);
CREATE INDEX idx_delegation_grantee_org ON delegation_grant (org_pk, grantee_pk, status);
CREATE INDEX idx_delegation_grantor     ON delegation_grant (grantor_pk);
```

각 컬럼을 시나리오로 읽으면 이해가 빠릅니다.

```
시나리오: 학원장(김원장)이 강사(박강사)에게 "회원 관리"를 7월 한 달만 위임

grantor_pk = 5    (김원장 — 주는 사람)
grantee_pk = 9    (박강사 — 받는 사람)
org_pk     = 42   (행복학원 — 이 테넌트 안에서만 유효)
capability = 'ACADEMY.MANAGE_MEMBERS'   (회원 관리 권한)
scope_json = { "class_ids": [101, 102] } (101·102반 회원만 — 전체 아님)
status     = 'ACTIVE'
expires_at = '2026-07-31 23:59:59'      (7월까지만, 지나면 자동 무효)
```

박강사는 role상으로는 평범한 TEACHER(또는 MEMBER)이지만, 이 한 행 덕분에 `ACADEMY.MANAGE_MEMBERS` 행동을 **101·102반에 한해, 7월까지** 할 수 있게 됩니다.

여기서 `scope_json`이 중요한 안전장치입니다. 위임은 통째로 넘기는 게 아니라 **범위를 좁혀서** 넘길 수 있습니다. "회원 관리"를 줬다고 *모든* 반의 회원을 만지게 하면 과합니다. `scope_json`으로 "이 반만"이라고 못 박습니다. (CHECK 제약이 없는 JSON 컬럼인 이유와 트레이드오프는 [[json-column]] 참고)

`capability`가 `ACADEMY.<action>` 네임스페이스인 이유와, CHECK가 아직 하드코딩으로 남은 부채는 [[role-capability]]에서 다뤘습니다. 이 문서에서는 "위임의 대상이 capability 하나다"라는 점만 기억하면 됩니다.

> 💡 **한 줄 요약**: `delegation_grant`는 grantor(주는 사람)→grantee(받는 사람)에게 특정 capability를 scope(범위)·expires_at(만료) 제한과 함께 넘기는 단건 위임 관계를 한 행으로 표현합니다.

---

## Q3. 위임받으면 "그 사람으로 로그인"하는 건가요? 임퍼소네이션이랑 뭐가 다른가요?

이게 ReBAC에서 가장 자주 헷갈리는 부분이고, `platform_db`가 **명시적으로 금지**한 안티패턴입니다([[requirements|REBAC-3]]).

**임퍼소네이션(impersonation)**은 "내가 김원장이 *된다*"입니다. 김원장의 신원으로 시스템에 들어가서, 김원장인 척 행동하는 것이죠. 영어로 *"becoming the user"*라고 표현합니다.

**위임(delegation)은 임퍼소네이션이 아닙니다.** 박강사는 *여전히 박강사 본인*입니다. 단지 김원장이 넘겨준 capability 하나를 **빌려서** 행사할 뿐입니다.

```
❌ 임퍼소네이션 (우리가 안 쓰는 방식)
   박강사가 로그인 → "김원장으로 전환" → audit_log에 actor = 김원장
   → 나중에 사고가 나도 "김원장이 한 일"로 보임. 실제 행위자(박강사)가 사라짐.

✅ 위임 (우리 방식)
   박강사가 본인으로 로그인 → 본인 신원 유지
   → delegation_grant에서 'ACADEMY.MANAGE_MEMBERS' 빌림 → 행동 허용
   → audit_log에 actor = 박강사 (실제 행위자 그대로 기록)
```

비유로 돌아가면, 택배 비유가 정확합니다. 택배기사는 *집주인이 되는* 게 아닙니다. 자기 이름표를 달고 와서, 집주인이 알려준 현관 비밀번호만 쓰는 겁니다. 사고가 나면 "그 택배기사가 했다"가 추적됩니다.

코드에서 위임 행사를 감사 기록할 때 actor는 항상 grantee입니다.

```typescript
// 박강사가 위임받은 권한으로 회원을 수정
await auditLog.record({
  orgPk,
  actorType: 'HUMAN',
  actorPk:   granteePk,        // ← 박강사 본인. 김원장 아님!
  action:    'ACADEMY.MANAGE_MEMBERS',
  resourceType: 'membership',
  resourcePk: targetMemberPk,
  result:    'ALLOW',
  // (선택) 위임 출처를 메타로 남길 수 있음 — but actor는 어디까지나 grantee
  detail:    { via: 'delegation', grantPk: grant.pk, grantorPk: grant.grantorPk },
});
```

왜 이렇게까지 구분할까요? **책임 추적성(accountability)** 때문입니다. 임퍼소네이션을 허용하면 "누가 실제로 그 버튼을 눌렀나?"가 흐려집니다. 위임은 권한은 빌려주되 *행위의 주체는 절대 바꾸지 않아서*, 감사 로그([[audit-hash-chain]])가 항상 진짜 행위자를 가리킵니다.

> 💡 **한 줄 요약**: 위임은 "그 사람이 되는 것(임퍼소네이션)"이 아니라 grantee 본인 신원으로 capability만 빌리는 것이며, audit_log의 actor는 언제나 grantee로 기록되어 책임 추적이 끊기지 않습니다.

---

## Q4. 위임을 거둬들이고 싶으면요? 만료랑 회수는 어떻게 동작하나요?

위임이 영원하면 위험합니다. 그래서 두 가지 종료 경로가 있습니다.

**① 만료(expiry) — `expires_at`이 지나면 자동 무효**

부여 시점에 끝나는 시각을 박아둡니다. 시간이 지나면 별도 조치 없이 효력을 잃습니다. 권한 조회 쿼리가 항상 만료 조건을 함께 봅니다.

```sql
-- "이 사람이 지금 이 org에서 가진 살아있는 위임"
SELECT capability, scope_json FROM delegation_grant
WHERE org_pk = ? AND grantee_pk = ?
  AND status = 'ACTIVE'
  AND (expires_at IS NULL OR expires_at > now());  -- ← 만료된 건 자동 제외
```

`idx_delegation_grantee_org (org_pk, grantee_pk, status)` 인덱스가 이 조회를 빠르게 만듭니다([[index-design]]). `expires_at IS NULL`은 "무기한 위임"을 허용하지만, 운영상 만료를 두는 걸 권장합니다.

**② 회수(revoke) — `status = 'REVOKED'`로 즉시 차단**

만료를 기다릴 수 없을 때(예: 박강사가 퇴사) grantor가 바로 거둬들입니다.

```typescript
// 위임 회수
await db.update(delegationGrant)
  .set({ status: 'REVOKED' })
  .where(eq(delegationGrant.pk, grantPk));

await bumpPermVersion(orgPk);  // ← 핵심! 권한 버전을 올린다
```

여기서 마지막 줄 `bumpPermVersion`이 핵심입니다. 회수했는데도 박강사의 **JWT 토큰(custom claims)은 최대 1시간 동안 stale**할 수 있습니다 — 토큰은 이미 발급됐으니 DB를 다시 안 봅니다. 회수 직후 박강사가 stale 토큰으로 들어오면 어떻게 막을까요?

```
회수 → bumpPermVersion(orgPk) → organization.perm_version + 1

이후 박강사가 sensitive write(회원 삭제 등) 시도:
  Gate가 @VerifyOnDb로 DB 최신 perm_version 확인
  → 토큰의 X-Perm-Version과 불일치 발견
  → 강제 토큰 재발급 → 새 토큰엔 위임 없음 → 403 DELEGATION_REVOKED
```

이 stale window를 메우는 메커니즘이 [[perm-version-propagation]]입니다. ReBAC 위임의 "회수"가 *진짜 즉시* 먹히려면 perm_version 전파가 반드시 함께 동작해야 합니다. 둘은 한 세트입니다.

> 💡 **한 줄 요약**: 위임은 `expires_at`으로 자동 만료되거나 `status='REVOKED'`로 즉시 회수되며, 회수 후 stale 토큰은 `bumpPermVersion`+`@VerifyOnDb`([[perm-version-propagation]])가 sensitive write에서 막아줍니다.

---

## Q5. 본사-지점 관계(org_relation)가 있던데, 본사면 지점 데이터를 자동으로 볼 수 있나요?

**아니요. 못 봅니다.** 이건 ReBAC에서 매우 중요한 원칙이라 따로 짚습니다([[requirements|REBAC-5]]: "계층은 권한 근거가 아니다").

`org_relation` 테이블은 조직 사이의 **그래프(계층)**를 표현합니다([[schema-reference|§D.7]]).

```sql
CREATE TABLE org_relation (
  pk              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  parent_org_pk   BIGINT NOT NULL,  -- 본사 / 지주사
  child_org_pk    BIGINT NOT NULL,  -- 지점 / 자회사
  relation_type   VARCHAR(20) NOT NULL  -- 본사-지점 / 지주-자회사
                    CHECK (relation_type IN ('HQ_BRANCH','HOLDING')),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT chk_no_self_ref CHECK (parent_org_pk != child_org_pk),  -- 자기참조 차단
  CONSTRAINT uq_org_relation_parent_child UNIQUE (parent_org_pk, child_org_pk)
);
CREATE INDEX idx_org_relation_child ON org_relation (child_org_pk);
```

여기서 흔한 오해가 생깁니다. "강남본원이 강남지점의 *parent*니까, 본원 원장은 지점 데이터를 다 볼 수 있겠지?" — **틀립니다.**

```
❌ 잘못된 가정: 계층 = 권한
   org_relation(강남본원 → 강남지점, HQ_BRANCH)가 있으니
   본원 사람은 지점 데이터에 자동 접근 가능   ← 우리 시스템은 이렇게 안 함

✅ 우리 원칙: 계층은 그냥 "지도"일 뿐, 권한 근거가 아님
   본원 원장이 지점 데이터를 보려면:
     (a) 그 지점에 명시적 membership/service_membership을 갖거나
     (b) 지점 OWNER가 명시적으로 delegation_grant를 줘야 함
   org_relation 자체로는 단 한 줄의 데이터 접근도 생기지 않음
```

왜 이렇게 깐깐할까요? 만약 "계층 = 자동 권한"이면, 본사 계정 하나가 털리면 *모든 지점*이 동시에 노출됩니다. 또 "본사니까 당연히 볼 수 있겠지" 하는 암묵적 가정이 코드 곳곳에 스며들어 [[multitenancy-rls|테넌트 격리]]가 무너집니다. 그래서 권한의 근거는 **언제나 명시적인 membership 또는 delegation_grant**여야 합니다. `org_relation`은 "이 조직들이 한 그룹이다"라는 *사실*만 기록하고, 권한과는 분리합니다.

`chk_no_self_ref`는 "조직이 자기 자신의 부모/자식"이 되는 말도 안 되는 사이클을 DB가 막는 것입니다 — `delegation_grant`의 `chk_no_self_delegation`("자기가 자기에게 위임")과 **대칭**을 이루는 무결성 제약입니다.

> 💡 **한 줄 요약**: `org_relation`(HQ_BRANCH/HOLDING)은 조직 계층을 그린 *지도*일 뿐 권한 근거가 아니며, 본사라도 지점 데이터를 보려면 명시적 membership이나 위임이 있어야 합니다(계층 ≠ 권한).

---

## Q6. 구글 Zanzibar 같은 "범용 관계 그래프"를 쓰는 게 더 강력하지 않나요? 왜 안 쓰나요?

좋은 질문이고, 실제로 진지하게 검토한 뒤 **의도적으로 보류**한 결정입니다([[requirements|REBAC-7]]: ⛔ 트리거 시 재검토).

**Zanzibar**는 구글이 만든 범용 권한 시스템으로, ReBAC를 극한까지 일반화했습니다. 핵심은 **relation tuple**이라는 단위입니다.

```
Zanzibar relation tuple 형식:
  <object>#<relation>@<subject>

예시:
  document:readme#owner@user:김원장        (readme의 owner는 김원장)
  document:readme#viewer@group:academy42   (readme를 academy42 그룹이 볼 수 있음)
  folder:root#parent@document:readme       (readme의 부모 폴더는 root)
  → 그리고 "폴더 owner는 그 안의 모든 문서 owner" 같은 규칙을 재귀로 계산
```

이러면 *모든 것*을 관계로 표현할 수 있습니다. 문서-폴더-그룹-사용자가 임의로 얽힌 그래프에서 "이 사람이 이 문서에 접근 가능한가?"를 그래프 순회로 답합니다. 매우 강력하지만, 그만큼 무겁습니다.

```
delegation_grant (현재)          vs    Zanzibar full relation_tuple (보류)
────────────────────────────────       ─────────────────────────────────────
단일 테이블, 단건 위임                   범용 튜플 저장소 + 재귀 평가 엔진
"A가 B에게 capability 위임"만            임의 object#relation@subject 그래프
SQL 한 방 조회로 끝                      그래프 순회 / 캐시(Leopard) 필요
운영 단순                                전용 인프라(권한 서비스) 운영 부담
```

현재 `platform_db`의 위임 수요는 "학원장이 강사에게 capability 하나를 한시 위임" 수준입니다. 이건 `delegation_grant` 단일 테이블로 **충분하고 명료**합니다. Zanzibar를 지금 도입하면 쓰지도 않을 그래프 엔진을 운영하는 과잉 설계가 됩니다.

대신 **재검토 트리거**를 명시해 뒀습니다. 다음이 현실이 되면 그때 다시 봅니다.

```
Zanzibar(또는 범용 relation_tuple) 재검토 트리거:
  - 커스텀 롤(조직마다 임의로 역할을 정의)을 사용자에게 열어줘야 할 때
  - 위임/관계가 다단계로 얽혀 SQL 한 방으로 못 푸는 복잡 관계 수요가 생길 때
  - 문서/리소스 단위의 세밀한 공유(폴더 상속 등)가 핵심 기능이 될 때
```

이건 [[multitenancy-rls|Pool 모델의 분리 트리거]]와 같은 철학입니다 — "지금은 단순하게, 트리거에 닿으면 그때 진화한다".

> 💡 **한 줄 요약**: Zanzibar식 범용 relation_tuple 그래프는 현재 위임 수요엔 과하므로 단순한 `delegation_grant`로 충분하며, 커스텀롤·복잡 관계 수요라는 트리거에 도달하면 재검토합니다.

---

## Q7. delegation_grant랑 서비스 쪽 trust_relationship은 같은 건가요? 둘 다 "관계"인데요.

이름이 비슷해서 헷갈리지만 **다른 층(layer)에 있는 다른 것**입니다([[requirements|REBAC-8]]: 경계 규칙).

```
┌─────────────────────────────────────────────────────────┐
│  platform_db (코어 — 모든 서비스 공통)                    │
│    delegation_grant                                       │
│      → 위임 대상 = "플랫폼 capability" (ACADEMY.<action>) │
│      → "학원장이 강사에게 회원관리 권한 위임" 같은 것      │
└─────────────────────────────────────────────────────────┘
                          ↑ 다른 층 (섞지 않음)
┌─────────────────────────────────────────────────────────┐
│  서비스 도메인 DB (예: academy 서비스)                    │
│    trust_relationship (도메인 테이블 — 예시)              │
│      → 도메인 관계 = "학생-학부모", "주치의-환자" 등      │
│      → "이 학부모가 이 학생의 보호자인가" 같은 도메인 사실 │
└─────────────────────────────────────────────────────────┘
```

구분 기준은 단순합니다. **"이 관계가 플랫폼 capability를 빌려주는가, 아니면 도메인의 사실(fact)을 표현하는가?"**

- `delegation_grant`(코어): "박강사가 `ACADEMY.MANAGE_MEMBERS`를 빌렸다" → **플랫폼 권한 위임**. 코어가 관리.
- `trust_relationship`(도메인): "이 학부모가 이 학생의 보호자다" → academy 서비스 내부의 **도메인 관계**. 코어가 몰라야 함.

**왜 분리할까요?** 코어가 도메인 어휘(학생-학부모, 트레이너-회원 등)를 알기 시작하면, [[role-capability|role 2단 분리]]에서 academy ENUM을 걷어낸 노력이 무위로 돌아갑니다. 코어는 **플랫폼 capability만** 다뤄야 멀티서비스 확장이 깨지지 않습니다. 학생-학부모 같은 관계는 academy가 자기 DB에서 알아서 표현하고, 거기에 권한이 필요하면 그 서비스 안에서 처리합니다.

규칙으로 외우면 이렇습니다.

```
✅ delegation_grant.capability에 들어갈 수 있는 것:  'ACADEMY.MANAGE_MEMBERS' (플랫폼 capability)
❌ delegation_grant에 넣으면 안 되는 것:             "student-parent" 같은 도메인 관계
   → 이건 서비스 도메인의 trust_relationship에 둬야 함
```

> 💡 **한 줄 요약**: 코어의 `delegation_grant`는 플랫폼 capability 위임만 다루고, 학생-학부모 같은 도메인 관계는 서비스 쪽 `trust_relationship`에 두어 — 코어가 도메인 어휘에 오염되지 않게 층을 분리합니다.

---

## 용어 정리

| 용어 | 뜻 | 한 줄 메모 |
|---|---|---|
| ReBAC | Relationship-Based Access Control | "관계로 권한을 정한다". RBAC·ABAC와 함께 쓰는 세 번째 축 |
| delegation | 위임 | 내 권한 일부를 남에게 한시적으로 빌려주는 것 (becoming ≠ 위임) |
| grantor | 권한을 *주는* 사람 | `delegation_grant.grantor_pk`. 예: 학원장 |
| grantee | 권한을 *받는* 사람 | `delegation_grant.grantee_pk`. 예: 강사 |
| capability | 위임의 대상이 되는 단건 권한 | `ACADEMY.<action>` 네임스페이스. role과 구분([[role-capability]]) |
| scope (scope_json) | 위임 범위 제한 | "전체 말고 101·102반만" 같은 JSON 제약 |
| expires_at | 만료 시각 | 지나면 자동 무효. `IS NULL`은 무기한 |
| status | ACTIVE / REVOKED | 회수하면 REVOKED → 즉시 차단(+perm_version) |
| impersonation | 임퍼소네이션 | "그 사람이 *되는* 것". 우리는 **금지**(REBAC-3) |
| Zanzibar | 구글의 범용 ReBAC 시스템 | relation_tuple 그래프. 현재 **보류**(REBAC-7) |
| relation tuple | Zanzibar의 단위 | `object#relation@subject`. 우리는 안 씀 |
| hierarchy (계층) | org_relation의 본사-지점 그래프 | **계층 ≠ 권한**(REBAC-5). 지도일 뿐 |
| chk_no_self_delegation | 자기위임 차단 CHECK | `grantor_pk <> grantee_pk` |
| chk_no_self_ref | 자기참조 차단 CHECK | org_relation. `parent != child`. 위와 대칭 |

---

## 테스트 방법
> 🧪 *실제 DB·ORM·운영에서 돌리는 법*: [[testing-strategy]] · [[orm-testing-drizzle]]


ReBAC/위임은 "부여 → 행사 → 회수 → 차단"의 **생애주기(lifecycle)** 전체가 정확해야 합니다([[e2e-journeys|e2e-journeys.md]] C-04 정상 lifecycle / D3-07 오부여 정정). 아래는 vitest + supertest 통합 테스트 예시입니다.

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import request from 'supertest';
import { app } from '../app';
import { db } from '../db';

describe('ReBAC delegation_grant 생애주기', () => {
  let ownerToken: string;   // 김원장 (grantor)
  let teacherToken: string; // 박강사 (grantee)
  const orgPk = 42;

  beforeEach(async () => {
    await seedOrgWithOwnerAndTeacher(orgPk); // owner=5, teacher=9
    ownerToken   = await loginAs({ userPk: 5, orgPk });
    teacherToken = await loginAs({ userPk: 9, orgPk });
  });

  // ── 1. 부여 → grantee가 행사 → 200 ──────────────────────────
  it('위임 부여 후 grantee가 capability를 행사하면 200', async () => {
    // 강사는 위임 전엔 회원관리 불가 → 403
    await request(app)
      .post('/members/100/role').set('Authorization', `Bearer ${teacherToken}`)
      .send({ role: 'STUDENT' })
      .expect(403);

    // 학원장이 위임 부여
    const grant = await request(app)
      .post('/delegation-grants').set('Authorization', `Bearer ${ownerToken}`)
      .send({ grantee: 9, capability: 'ACADEMY.MANAGE_MEMBERS' })
      .expect(201);

    // perm_version이 올라가야 함 (stale 토큰 방지의 전제)
    teacherToken = await refreshToken({ userPk: 9, orgPk });

    // 이제 강사가 회원관리 행사 → 200
    await request(app)
      .post('/members/100/role').set('Authorization', `Bearer ${teacherToken}`)
      .send({ role: 'STUDENT' })
      .expect(200);

    expect(grant.body.status).toBe('ACTIVE');
  });

  // ── 2. 회수 → 즉시 403 (stale 토큰도 차단) ───────────────────
  it('위임 회수 후에는 stale 토큰이라도 403 DELEGATION_REVOKED', async () => {
    const { body: grant } = await request(app)
      .post('/delegation-grants').set('Authorization', `Bearer ${ownerToken}`)
      .send({ grantee: 9, capability: 'ACADEMY.MANAGE_MEMBERS' }).expect(201);

    teacherToken = await refreshToken({ userPk: 9, orgPk }); // 위임 포함 토큰

    // 학원장이 회수 (status=REVOKED + bumpPermVersion)
    await request(app)
      .delete(`/delegation-grants/${grant.pk}`)
      .set('Authorization', `Bearer ${ownerToken}`).expect(200);

    // 강사는 아직 옛 토큰(위임 포함)을 들고 있음 → sensitive write 시도
    const res = await request(app)
      .post('/members/100/role').set('Authorization', `Bearer ${teacherToken}`)
      .send({ role: 'STUDENT' });

    // @VerifyOnDb가 perm_version 불일치 잡아 차단
    expect(res.status).toBe(403);
    expect(res.body.code).toBe('DELEGATION_REVOKED');
  });

  // ── 3. 자기위임은 DB CHECK가 거부 ────────────────────────────
  it('grantor==grantee 자기위임 INSERT는 chk_no_self_delegation으로 거부', async () => {
    await expect(
      db.insert('delegation_grant', {
        grantor_pk: 9, grantee_pk: 9, // ← 동일인
        org_pk: orgPk, capability: 'ACADEMY.MANAGE_MEMBERS', status: 'ACTIVE',
      })
    ).rejects.toThrow(/chk_no_self_delegation|CONSTRAINT/i);
  });

  // ── 4. 만료된 grant는 권한 조회에서 제외 ─────────────────────
  it('expires_at이 지난 grant는 활성 위임 조회에서 빠진다', async () => {
    await db.insert('delegation_grant', {
      grantor_pk: 5, grantee_pk: 9, org_pk: orgPk,
      capability: 'ACADEMY.MANAGE_MEMBERS', status: 'ACTIVE',
      expires_at: '2020-01-01 00:00:00', // 과거 = 이미 만료
    });

    const active = await getActiveGrants({ orgPk, granteePk: 9 });
    expect(active.find(g => g.capability === 'ACADEMY.MANAGE_MEMBERS')).toBeUndefined();
  });

  // ── 5. 위임 행사는 audit_log에 grantee로 기록 (임퍼소네이션 아님) ─
  it('위임 행사 시 audit_log.actor_pk는 grantor가 아니라 grantee', async () => {
    await request(app)
      .post('/delegation-grants').set('Authorization', `Bearer ${ownerToken}`)
      .send({ grantee: 9, capability: 'ACADEMY.MANAGE_MEMBERS' }).expect(201);
    teacherToken = await refreshToken({ userPk: 9, orgPk });

    await request(app)
      .post('/members/100/role').set('Authorization', `Bearer ${teacherToken}`)
      .send({ role: 'STUDENT' }).expect(200);

    const log = await getLatestAuditLog({ orgPk, action: 'ACADEMY.MANAGE_MEMBERS' });
    expect(log.actor_pk).toBe(9);   // ✅ 박강사 (실제 행위자)
    expect(log.actor_pk).not.toBe(5); // ❌ 김원장(grantor)으로 둔갑하지 않음
  });
});
```

**자가진단 체크리스트:**

```
□ 위임 부여 전 grantee는 그 capability 행동이 403인가?
□ 부여 후 (토큰 갱신 뒤) 200으로 바뀌는가?
□ 회수 시 status='REVOKED' + bumpPermVersion이 함께 일어나는가?
□ 회수 후 stale 토큰의 sensitive write가 @VerifyOnDb로 403되는가?
□ grantor==grantee INSERT가 chk_no_self_delegation으로 거부되는가?
□ expires_at 지난 grant가 활성 조회(status=ACTIVE AND expires_at>NOW)에서 빠지는가?
□ scope_json으로 좁힌 범위 밖 리소스 접근이 막히는가?
□ 위임 행사 audit_log의 actor_pk가 grantee인가? (grantor 아님 — 임퍼소네이션 금지)
□ org_relation(본사-지점)만으로는 지점 데이터 접근이 안 되는가? (계층 ≠ 권한)
□ org_relation에 parent==child INSERT가 chk_no_self_ref로 거부되는가?
```

---

## 마치며

ReBAC와 위임은 결국 하나의 질문으로 귀결됩니다: **"이 권한의 근거가 *명시적 관계*인가, 아니면 *암묵적 가정*인가?"**

`platform_db`의 답은 일관됩니다.

1. 권한은 role(RBAC)·속성(ABAC)·**관계(ReBAC)** 세 축 중 하나로 *명시적으로* 생긴다.
2. 위임은 "그 사람이 되는 것"이 아니다 — grantee 본인 신원으로 capability만 빌리고, audit_log는 항상 진짜 행위자를 가리킨다.
3. 계층(org_relation)은 지도일 뿐, 권한이 아니다 — 본사라고 지점 데이터를 공짜로 못 본다.
4. 지금은 단순한 `delegation_grant`로 충분하다 — Zanzibar는 트리거에 닿으면 그때.

위임 기능을 만들거나 리뷰할 때는 "이 capability가 만료·회수되면 *즉시* 닫히는가? audit_log에 *진짜 행위자*가 남는가?"를 꼭 확인하세요. 이 두 가지가 ReBAC를 "편리하지만 추적 가능한" 권한 모델로 만들어 줍니다.

---

## 연결된 개념

- [[role-capability|role 2단 분리와 capability]] — role vs capability 차이, RBAC/ABAC/ReBAC 3축, `ACADEMY.<action>` 네임스페이스
- [[casl-ability|CASL ability]] — Gate C에서 위임 capability가 RBAC·ABAC와 하나의 ability로 합쳐지는 지점
- [[perm-version-propagation|perm_version 전파]] — 위임 회수가 stale 토큰에도 즉시 먹히게 하는 메커니즘 (`bumpPermVersion`·`@VerifyOnDb`)
- [[gate-abc-flow|Gate A/B/C 전체 흐름]] — Gate C에서 위임 capability가 평가되는 위치
- [[bola-object-authz|객체 수준 인가(BOLA)]] — scope_json 범위 제한과 리소스 소유권 체크의 관계
- [[multitenancy-rls|Pool 모델 + RLS]] — org_pk 테넌트 경계(위임도 org 안에서만 유효) + 분리 트리거 철학
- [[break-glass|break-glass]] — 위임과 구분되는 긴급 권한 상승 경로
- [[audit-hash-chain|감사 로그 해시 체인]] — 위임 행사가 grantee로 기록되는 감사 추적의 무결성
> 소스 문서
- [[architecture]] — §1.2 RBAC/ABAC/ReBAC 정의, ReBAC 결정사항
- [[schema-reference]] — §D.6 delegation_grant DDL, §D.7 org_relation DDL, §E.4 위임 흐름
- [[requirements]] — REBAC-1~8 (위임·임퍼소네이션 금지·계층≠권한·Zanzibar 보류·trust_relationship 경계)
- [[e2e-journeys]] — C-04 위임 가치흐름(정상 lifecycle), D3-07 오부여 정정(회수→차단)
