---
type: decision
status: 채택
aliases:
  - cross-tenant 분리
  - Admin Role 거부
  - 아키텍처 분리
tags:
  - platform-db
  - decision
  - multitenancy
  - security
  - isolation
---

# cross-tenant 조회 — Admin Role 거부, 코드 경로 분리

> 상태: 채택 · 영역: 멀티테넌시 격리 · 형식: 비교 → 결정

## 결정

전체 테넌트를 가로지르는 집계 조회(전 학원 강의 수·사용량 등)는 **Admin role 분기로 풀지 않는다.** cross-tenant 쿼리는 전용 모듈(`internal/`) 또는 별도 admin 서비스에서만 존재하고, 일반 비즈니스 로직에는 `org_pk` 필터가 항상 적용된다 — **코드 *위치*로 의도를 드러낸다.**

---

## 맥락 — 왜 결정이 필요했나

PostgreSQL·Qdrant·Neo4j 모두 `org_pk`/`org_id` 단위로 테넌트를 격리한다([[multitenancy-pool]]). 그런데 운영 대시보드는 **격리를 의도적으로 벗어나는** 전 테넌트 집계가 필요하다. "격리를 어떻게 안전하게 깨는가"를 정해야 한다.

---

## 비교한 선택지

### 선택지 A — Admin Role 런타임 분기

```typescript
if (user.role === "admin") {
  return db.select().from(lecture);                       // org_pk 없음 — 전 테넌트
} else {
  return db.select().from(lecture).where(eq(lecture.orgPk, orgPk));
}
```

- **장점**: 초기 구현 비용 낮음, 코드 한 곳에서 분기
- **단점**:
  - 단일 장애점 — admin role 탈취 시 전 테넌트 즉시 노출
  - 비즈니스 로직에 보안 우회가 섞여 코드 리뷰에서 놓치기 쉬움
  - "왜 이 분기가 여기 있지?" 컨텍스트 소실 → 잘못 수정/삭제
  - role 남용 확산("이것도 admin만 되면 되지")
  - 격리 불변식이 런타임 조건에 의존하게 됨

### 선택지 B — 아키텍처 분리 (채택)

cross-tenant 조회를 **코드 경로 자체로 분리**한다. 일반 모듈은 `org_pk`를 필수 타입으로 받고, 필터를 제거하는 코드는 `internal/` 모듈(또는 `*-admin` 서비스)에만 둔다.

```
modules/
  lecture/     ← 테넌트 범위 — org_pk 항상 포함, 제거 불가
  rag/         ← 테넌트 범위
  internal/    ← cross-tenant 전용 — 여기서만 org_pk 없는 쿼리 허용
                 (외부 라우팅 미노출, 내부망/별도 인증)
```

- **장점**:
  - 탈취할 admin role 자체가 없음 → 공격 표면 0
  - 파일 *위치*가 의도를 설명 — 리뷰에서 "왜 org_pk가 없지?" 의문이 안 생김
  - 일반 API에서 격리는 구조적으로 제거 불가(우회하려면 코드를 internal/로 옮겨야 하고, 그 이동 자체가 명시적 선언)
- **단점**: 초기 구현 비용 중간(`internal/` 모듈 분리 필요)

| 기준 | A: Admin Role | B: 아키텍처 분리 (우리) |
|---|---|---|
| 보안 | 단일 장애점(탈취 시 전 노출) | 공격 표면 없음 |
| 코드 가독성 | 보안 분기 혼재 | 파일 위치로 의도 명시 |
| 코드 리뷰 | role 분기로 org_pk 누락 정당화 가능 | org_pk 없으면 internal/로만 |
| 확장성 | role 남용 확산 | 별도 서비스 분리 자연스러움 |
| 초기 비용 | 낮음 | 중간 |

---

## 왜 B인가

Admin role은 본질적으로 "role 파괴자"다 — 격리 불변식을 코드가 아니라 런타임 값에 맡긴다. 반면 아키텍처 분리는 격리를 **타입과 디렉토리 구조로 강제**해, 보안이 사람의 주의력이 아니라 구조에 의존하게 만든다. cross-tenant 집계는 raw SQL로 의도를 명시적으로 드러낸다(ORM 추상화에 가려지지 않음).

---

## 트레이드오프 — 우리가 받아들인 것

- `internal/` 모듈을 따로 두는 초기 비용. 단순 if-분기보다 손이 더 감.
- cross-tenant 집계 코드는 typed query builder의 보호를 일부 포기하고 raw SQL을 쓴다(의도 명시의 대가).

---

## 전환 조건 — 이 결정을 다시 볼 때

| 단계 | 조건 | 행동 |
|---|---|---|
| 현재 | 단일 서비스 | `internal/` 모듈로 cross-tenant 격리 |
| 규모 확장 | admin 집계 트래픽 증가 / 보안 감사 요구 | 별도 admin 서비스 분리, 내부망 전용 배포 |
| DB 분리 시 | [[multitenancy-pool]] 분리 트리거 발동 | cross-tenant가 각 DB에 fan-out 쿼리로 전환 |

## 구현 규칙

1. 테넌트 범위 모듈의 모든 서비스 메서드는 `org_pk`/`org_id`를 파라미터로 받는다.
2. 해당 모듈에서 org 조건 없는 쿼리 발견 시 PR 기각.
3. cross-tenant 조회는 `internal/` 또는 별도 admin 서비스로만 구현.
4. `internal/` 컨트롤러는 외부 라우팅에 노출 금지(내부망/별도 인증).
5. 벡터·그래프 어댑터 인터페이스의 `org_id`는 필수 타입 — optional 변경 금지.

---

## 관련 문서

- [[multitenancy-pool]] — org_pk 행 격리 본문 (이 분리가 보호하는 불변식)
- [[rag-multitenancy]] — RAG 격리 (벡터 Qdrant payload 필터 · 그래프 Neo4j 경로 강제)
> 소스 문서
- [[architecture]] — §1.4 멀티테넌시 격리 전략
- [[schema-reference]] — §G 저장소별 격리 구현 현황
