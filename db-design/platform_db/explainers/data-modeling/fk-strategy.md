---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p1
  - schema
  - foreign-key
  - multitenancy
  - design-decision
aliases:
  - FK 전략
  - 외래키
  - cross-schema FK
  - fk-strategy
---

# FK 전략 설명 (언제 걸고, cross-schema에서 왜 끊나)

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[schema-reference]] §C.3·§A.1, [[architecture]] §2.1, [[multitenancy-rls]]

`platform_db`는 같은 DB 안에서는 FK를 적극적으로 걸지만, DB(스키마)가 다르면 **일부러 FK를 걸지 않습니다.** 왜 같은 도구를 어떤 데선 쓰고 어떤 데선 안 쓰는지 설명합니다.

---

## Q1. FK가 뭐고, 왜 거나요?

FK(Foreign Key, 외래키)는 "이 컬럼 값은 반드시 저 테이블에 존재해야 한다"는 **DB 레벨 보장**입니다.

```sql
-- membership.user_pk는 반드시 identity_user.pk에 존재해야 함
CONSTRAINT fk_mbr_user FOREIGN KEY (user_pk) REFERENCES identity_user(pk)
```

```
INSERT INTO membership (user_pk, ...) VALUES (999, ...);
-- user_pk=999가 identity_user에 없으면?
FK 없음: INSERT 성공 → 고아(orphan) 데이터 생성
FK 있음: ERROR 1452 → 앱이 에러 처리
```

FK가 해주는 일: ① 고아 row 방지(정합성) ② ERD에서 관계 문서화 ③ FK 컬럼 인덱스 자동 생성(JOIN 성능). 그래서 **같은 스키마 안에서는** FK를 씁니다 — 정합성 보장 비용이 낮고 효과가 큽니다.

> 💡 **한 줄 요약**: FK는 "참조 대상이 실제로 존재함"을 DB가 보장해 고아 데이터를 막아줍니다. platform_db 내부 관계엔 적극 사용합니다.

---

## Q2. 그런데 왜 cross-schema FK는 일부러 안 거나요?

`academy_db.lecture`가 `platform_db.identity_user`를 참조하지만 FK는 걸지 않습니다.

```sql
CREATE TABLE lecture (
  teacher_pk BIGINT UNSIGNED NOT NULL,  -- identity_user.pk 참조하지만 FK 없음
  ...
);
```

세 가지 이유 때문입니다.

**① 독립 백업/복원** — `academy_db`만 롤백해야 할 때, FK가 있으면 `platform_db`와 FK 불일치로 복원이 실패합니다. FK가 없으면 단독 복원이 가능합니다.

**② Lock 전파 방지** — `platform_db.identity_user` 대량 업데이트 중 FK가 있으면 부모 테이블 lock이 자식(`academy_db.lecture`)으로 전파됩니다. FK가 없으면 각 DB가 독립적으로 돕니다.

**③ 배포 독립성** — `platform_db`에 컬럼을 추가(온라인 DDL)할 때 FK가 있으면 `academy_db` 배포 타이밍까지 맞춰야 합니다. FK가 없으면 두 DB 배포가 완전히 독립적입니다.

이것이 불변식 #6 "cross-DB는 아래로만, cross-schema FK 금지"의 근거입니다([[multitenancy-rls]]의 격리 전략과 같은 맥락).

> 💡 **한 줄 요약**: cross-schema FK는 백업·lock·배포를 서로 묶어버립니다. DB를 독립적으로 운영하려고 일부러 끊습니다.

---

## Q3. FK를 안 걸면 정합성은 어떻게 보장하나요?

앱 레이어에서 검증합니다.

```typescript
async function createLecture(teacherPk: bigint, orgPk: bigint) {
  // identity_user 존재 + org 소속 확인 (암묵적 FK 역할)
  const membership = await getActiveMembership(teacherPk, orgPk);
  if (!membership) throw ForbiddenException();
  await academyDb.insert(lecture).values({ teacherPk, orgPk, ... });
}
```

근거는 두 가지입니다.
- **SSOT 상시 존재 전제**: 사용자가 없으면 JWT 자체가 발급되지 않으므로 `teacher_pk`가 가리키는 사용자는 사실상 항상 존재합니다.
- **Gate A(membership 확인)가 암묵적 FK 역할**을 합니다 — cross-DB write 전에 소속을 검증.

트레이드오프:

| 항목 | cross-schema FK | FK 없음 (우리) |
|---|---|---|
| DB 레벨 정합성 | 보장 | 앱 레이어 의존 |
| 독립 백업/복원 | 불가 | 가능 |
| Lock 전파 | 있음 | 없음 |
| 배포 독립성 | 결합됨 | 완전 독립 |

> 💡 **한 줄 요약**: cross-schema 정합성은 FK 대신 앱 레이어(Gate A + SSOT 전제)로 보장합니다. 독립 운영을 얻는 대신 검증 책임을 코드로 옮긴 것입니다.

---

## 연결된 개념

- [[multitenancy-rls]] — org_pk 행 격리와 cross-DB 방향 규칙
- [[gate-abc-flow]] — Gate A(membership 확인)가 암묵적 FK 역할을 하는 흐름
> 소스 문서
- [[schema-reference]] — §C.3 FK 관계 요약표, §A.1 cross-DB 방향 규칙
- [[architecture]] — §2.1 불변식 #6 (cross-DB는 아래로만)
