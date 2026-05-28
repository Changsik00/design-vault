# role→action 매핑은 코드 상수

## 배경

TEACHER라는 역할이 무엇을 할 수 있는지(permission)를 어디에 저장할 것인가.

```
TEACHER → [publish_video, view_students, manage_schedule]
DIRECTOR → [publish_video, view_students, manage_schedule, manage_members, view_billing]
OWNER → [모든 것]
```

이 매핑이 바뀌면 어디를 수정해야 하는가. DB인가, 코드인가.

---

## 일반 권고 — DB role_capability 레지스트리

```sql
-- role별 permission을 DB 테이블로 관리
CREATE TABLE role_capability (
  role       VARCHAR(30) NOT NULL,
  capability VARCHAR(100) NOT NULL,
  PRIMARY KEY (role, capability)
);

INSERT INTO role_capability VALUES ('TEACHER', 'publish_video');
INSERT INTO role_capability VALUES ('TEACHER', 'view_students');
-- ...
```

**권고 근거**:
- 배포 없이 운영자가 DB에서 직접 권한 변경 가능
- 테넌트(학원)마다 커스텀 역할 구성 가능
- 권한 목록 UI 화면에서 조회 용이

**문제점**:
- 권한 변경이 언제 누가 왜 바꿨는지 추적 불가 (DB 히스토리 테이블 별도 필요)
- 버그 발생 시 "어느 버전의 코드가 어느 DB 상태를 기대했나" 재현이 어려움
- 테스트: 코드가 어떤 DB 상태를 전제하는지 픽스처 관리 복잡
- 잘못된 변경이 배포 없이 즉시 프로덕션 반영 → 롤백이 어려움
- Stripe, Linear, GitHub도 permission을 코드 상수로 관리함

---

## 우리 결정 — 코드 상수 (D3)

```typescript
// packages/db-platform/src/permissions.ts
export const ROLE_PERMISSION = {
  academy: {
    OWNER:    ['publish_video', 'view_students', 'manage_members', 'view_billing', 'manage_schedule'],
    DIRECTOR: ['publish_video', 'view_students', 'manage_members', 'view_billing'],
    TEACHER:  ['publish_video', 'view_students', 'manage_schedule'],
    STUDENT:  ['view_own_lectures'],
    PARENT:   ['view_own_child'],
  },
  market: {
    SELLER:   ['list_item', 'update_item', 'view_orders'],
    BUYER:    ['place_order', 'view_own_orders'],
  },
} as const;

// Gate C에서 사용
function buildAbility(membership: Membership) {
  const perms = ROLE_PERMISSION[service][membership.role] ?? [];
  // perms를 CASL ability로 변환
}
```

**채택 근거**:
1. **추적성**: 권한 변경 = PR + 코드 리뷰 + git blame → 언제 누가 왜 바꿨는지 항상 알 수 있음
2. **테스트**: 코드에 권한이 있으므로 단위 테스트에서 DB 픽스처 불필요
3. **배포 롤백**: 잘못된 권한 변경 → 코드 revert → 즉시 롤백
4. **디버깅**: `console.log(ROLE_PERMISSION.academy.TEACHER)` 로 즉시 확인

---

## 트레이드오프

| 항목 | DB 레지스트리 | 코드 상수 (우리) |
|---|---|---|
| 권한 변경 방법 | DB 직접 수정 | PR + 배포 |
| 변경 추적 | 별도 audit 테이블 필요 | git log |
| 즉각 적용 | 가능 | 배포 후 |
| 테넌트 커스텀 롤 | 가능 | **불가** (P2 트리거) |
| 실수 위험 | 배포 없이 즉시 프로덕션 | PR 리뷰가 방어선 |

---

## 향후 조건

DB 레지스트리 도입 트리거:

1. **테넌트 커스텀 역할 실수요 발생**: 학원마다 직접 role을 구성하고 싶다는 요청이 있을 때
2. **permission 종류 수백 개 이상**: 코드 파일이 비현실적으로 커질 때

도입 시 절충안: 기본 역할은 코드 상수 유지, 테넌트 커스텀 역할만 DB — 두 레이어 혼합.

---

## 관련 문서

| 파일 | 내용 |
|---|---|
| `core/architecture.md §4 D3` | 코드 상수 결정 원문 |
| `core/architecture.md §5.3` | RBAC/ABAC/ReBAC 정의 |
| `core/schema-reference.md §E.2 Gate C` | CASL ability 빌드 코드 |
