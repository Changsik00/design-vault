# FK 전략 — 언제 걸고, cross-schema에서 왜 끊나

## 배경

FK(Foreign Key, 외래키)는 "이 컬럼의 값은 반드시 저 테이블에 존재해야 한다"는 DB 레벨 보장이다.

```sql
-- membership.user_pk는 반드시 identity_user.pk에 존재해야 함
CONSTRAINT fk_mbr_user FOREIGN KEY (user_pk) REFERENCES identity_user(pk)
```

존재하지 않는 user_pk를 넣으려 하면 MySQL이 에러를 낸다. 앱 코드가 실수해도 DB가 막아준다.

그런데 우리는 어떤 FK는 걸고, 어떤 FK는 의도적으로 걸지 않는다. 왜인가?

---

## FK가 하는 일

```
INSERT INTO membership (user_pk, org_pk, role)
VALUES (999, 1, 'TEACHER');
-- user_pk=999가 identity_user에 없으면?

FK 없음: INSERT 성공 → 고아 데이터 생성
FK 있음: ERROR 1452 (ER_NO_REFERENCED_ROW) → 앱이 에러 처리
```

FK의 역할:
1. **데이터 정합성 보장**: 고아(orphan) row 생성 방지
2. **관계 문서화**: ERD에서 FK가 곧 관계를 표현
3. **인덱스 자동 생성**: FK 컬럼에 인덱스 자동 추가 (JOIN 성능)

---

## 우리 전략 — 스키마 내부는 FK, cross-schema는 FK 없음

### 스키마 내부: FK 사용

```sql
-- platform_db 안에서: FK 사용
CREATE TABLE membership (
  user_pk BIGINT UNSIGNED NOT NULL,
  org_pk  BIGINT UNSIGNED NOT NULL,
  ...
  CONSTRAINT fk_mbr_user FOREIGN KEY (user_pk) REFERENCES identity_user(pk),
  CONSTRAINT fk_mbr_org  FOREIGN KEY (org_pk)  REFERENCES organization(pk)
);
```

같은 DB 안이므로 정합성 보장 비용이 낮고, 고아 데이터 방지 효과가 크다.

### cross-schema: FK 없음

```sql
-- academy_db의 lecture → platform_db.identity_user 참조 필요
CREATE TABLE lecture (
  teacher_pk BIGINT UNSIGNED NOT NULL,  -- identity_user.pk를 참조하지만
  -- FK 없음! 앱 레이어에서 검증
  ...
);
```

---

## cross-schema FK를 끊은 이유

**이유 1 — 독립 백업/복원**

```
상황: academy_db만 롤백해야 함
FK 있을 때: academy_db 복원 → platform_db와 FK 불일치 → 복원 실패
FK 없을 때: academy_db 단독 복원 가능
```

**이유 2 — Lock 전파 방지**

```
상황: platform_db.identity_user 대량 업데이트 중
FK 있을 때: academy_db.lecture도 잠김 (부모 테이블 lock이 자식 테이블에 전파)
FK 없을 때: 각 DB 독립적으로 운영
```

**이유 3 — 배포 독립성**

```
상황: platform_db에 identity_user 컬럼 추가 (온라인 DDL)
FK 있을 때: academy_db 배포 타이밍도 맞춰야 함 (FK 참조 변경 가능성)
FK 없을 때: platform_db 배포와 academy_db 배포 완전 독립
```

---

## 대신 뭘로 정합성을 보장하나

```typescript
// 앱 레이어에서 검증
async function createLecture(teacherPk: bigint, orgPk: bigint) {
  // identity_user 존재 + org 소속 확인
  const membership = await getActiveMembership(teacherPk, orgPk);
  if (!membership) throw ForbiddenException();

  // 검증 통과 후 academy_db에 INSERT
  await academyDb.insert(lecture).values({ teacherPk, orgPk, ... });
}
```

- SSOT(identity_user) 상시 존재 전제: 사용자가 없으면 JWT 자체가 발급 안 됨
- Gate A(membership 확인)가 암묵적 FK 역할

---

## 트레이드오프

| 항목 | cross-schema FK | FK 없음 (우리) |
|---|---|---|
| DB 레벨 정합성 | 보장 | 앱 레이어 의존 |
| 독립 백업/복원 | 불가 | 가능 |
| Lock 전파 | 있음 | 없음 |
| 배포 독립성 | 결합됨 | 완전 독립 |

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §C.3` | FK 관계 요약표 |
| `core/schema-reference.md §A.1` | cross-DB 방향 규칙 |
| `core/architecture.md §3.1` | 불변식 #6: cross-DB는 아래로만 |
