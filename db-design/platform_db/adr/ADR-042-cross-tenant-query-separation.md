# ADR-042: Cross-Tenant 조회 전략 — Admin Role 거부 + 아키텍처 분리 채택

| 항목 | 내용 |
|---|---|
| **상태** | Accepted |
| **날짜** | 2026-05-27 |
| **결정자** | dennis |
| **관련 ADR** | ADR-035 (Qdrant 멀티테넌시), ADR-036 (Neo4j 이중 색인), ADR-041 (멀티테넌시 DB 전략) |

---

## 배경

시스템에는 세 개의 데이터 저장소가 있으며, 각각 테넌트(org_pk) 단위 격리를 적용하고 있다.

| 저장소 | 격리 방식 |
|---|---|
| MySQL | `org_pk NOT NULL` + 모든 쿼리에 `WHERE org_pk = ?` |
| Qdrant | payload 필터 `must: [{ key: "org_id", match: { value: X } }]` |
| Neo4j | 노드 프로퍼티 `orgId` + `WHERE orgId = $orgId` 조건 필수 |

운영 과정에서 **전체 테넌트를 가로지르는 집계 조회**(학원 수, 강의 수, 사용량, 개념 그래프 현황 등)가 필요하다. 이를 어떻게 처리할 것인지 결정이 필요하다.

---

## 결정의 핵심 전제: "Admin Role은 Role 파괴자다"

Admin role 방식을 채택하지 않는다. 그 이유는 아래와 같다.

Admin role은 런타임 조건 분기다. 비즈니스 로직 안에 보안 우회 경로를 섞는다:

```typescript
// ❌ 채택하지 않는 패턴
if (user.role === "admin") {
  return db.select().from(lecture);           // org_pk 조건 없음 — 전 테넌트 노출
} else {
  return db.select().from(lecture).where(eq(lecture.orgPk, orgPk));
}
```

이 패턴의 문제점:

1. **단일 장애점**: admin role이 탈취되면 전체 테넌트 데이터가 즉시 노출된다
2. **의도 희석**: 비즈니스 로직과 보안 우회가 같은 파일에 섞여 코드 리뷰에서 놓치기 쉽다
3. **컨텍스트 소실**: 시간이 지나면 "왜 이 분기가 여기 있지?"라는 의문이 생기고 삭제되거나 잘못 수정될 수 있다
4. **role 남용 확산**: admin role이 한 번 생기면 "이것도 admin만 되면 되지"라는 패턴이 확산된다
5. **테넌트 격리 불변식 파괴**: 격리 원칙이 코드 레벨에서 보장되지 않고 런타임 조건에 의존하게 된다

---

## 결정: 아키텍처 분리 (Architectural Separation)

**Role 분기가 아니라 코드 경로 자체를 분리한다.**

Cross-tenant 조회는 전용 모듈 또는 전용 서비스에서만 존재한다. 일반 비즈니스 로직에는 org_pk 필터가 항상 적용되고, 이 필터를 제거하는 코드 경로는 구조적으로 일반 API와 격리된다.

### 디렉토리 구조

현 단계 (단일 서비스):

```
apps/academy-api/src/
  modules/
    lecture/         ← 테넌트 범위 — org_pk 항상 포함, 제거 불가
    rag/             ← 테넌트 범위 — org_pk 항상 포함
    internal/        ← cross-tenant 전용 — 여기서만 org_pk 없는 쿼리 허용
      controllers/internal.controller.ts   ← 외부 라우팅 미노출
      services/cross-tenant.service.ts
      services/cross-tenant-qdrant.service.ts
      services/cross-tenant-neo4j.service.ts
```

규모가 커진 이후 (서비스 분리):

```
apps/
  academy-api/       ← 일반 API — org_pk 필터 항상 적용
  academy-admin/     ← 별도 서비스 — cross-tenant 쿼리만 존재, 내부 네트워크 전용
```

---

## 저장소별 적용 방법

### MySQL (Drizzle ORM)

일반 서비스: typed query builder가 실수를 막는다.

```typescript
// lecture.service.ts — org_pk 없으면 컴파일 타임에 경고
.select().from(lecture).where(eq(lecture.orgPk, orgPk))  // 항상 포함
```

cross-tenant 서비스: raw query로 의도를 명시한다.

```typescript
// cross-tenant.service.ts — 의도적으로 org_pk 없음이 코드에서 보임
const result = await this.db.execute(
  sql`SELECT o.name        AS org_name,
             COUNT(l.pk)   AS lecture_count,
             COUNT(lc.pk)  AS chunk_count
      FROM   organization o
      LEFT JOIN lecture l  ON l.org_pk  = o.pk
      LEFT JOIN lecture_chunk lc ON lc.lecture_pk = l.pk
      GROUP BY o.pk`
);
```

> **Drizzle 선택 이유 중 하나가 바로 이 raw query 이점이다.** ORM 추상화에 가로막히지 않고 cross-tenant 집계를 명시적 SQL로 작성할 수 있다. 의도가 쿼리 자체에서 드러난다.

### Qdrant

일반 어댑터: `orgId`가 인터페이스에서 필수다.

```typescript
// qdrant-embedding.adapter.ts
interface QdrantSearchInput {
  orgId: string;       // 필수 — 없으면 타입 에러
  teacherPk?: bigint;
  query: string;
}
```

cross-tenant 어댑터: 명시적으로 분리된 클래스.

```typescript
// cross-tenant-qdrant.service.ts
class CrossTenantQdrantService {
  // filter 없는 scroll — 전체 컬렉션 집계 (명칭과 위치에서 의도 드러남)
  async getAllOrgsChunkStats(): Promise<OrgChunkStat[]> {
    return this.qdrantClient.scroll("academy_lectures", {
      with_payload: ["org_id"],
      limit: 10000,
    });
  }
}
```

### Neo4j

일반 어댑터: `orgPublicId`가 `ConceptUpsertInput` 인터페이스에서 필수다.

```typescript
// neo4j-concept.adapter.ts
interface ConceptGraphInput {
  orgPublicId: string;  // 필수 — 없으면 타입 에러
  teacherPk?: bigint;   // null = 해당 org 전체 강사 조회 허용
}
```

`teacherPk: null`은 허용 범위다 — 같은 org 안에서 강사 전체를 보는 것은 테넌트 격리를 위반하지 않는다.

cross-tenant 어댑터: 별도 서비스.

```typescript
// cross-tenant-neo4j.service.ts
class CrossTenantNeo4jService {
  // orgId 조건 없는 쿼리 — 전체 개념 그래프 현황 집계
  async getAllOrgsConceptStats(): Promise<OrgConceptStat[]> {
    const session = this.driver.session();
    return session.executeRead(async (tx) => {
      const result = await tx.run(
        `MATCH (c:Concept)
         RETURN c.orgId AS orgId, count(c) AS conceptCount
         ORDER BY conceptCount DESC`
      );
      return result.records.map((r) => ({
        orgId: r.get("orgId") as string,
        conceptCount: (r.get("conceptCount") as { toNumber(): number }).toNumber(),
      }));
    });
  }
}
```

---

## 이 원칙이 보장하는 것

### 코드 구조가 의도를 설명한다

`internal/` 또는 `academy-admin/` 디렉토리 안에 있는 파일은 **명시적으로 cross-tenant 조회를 담당하는 코드**다. Role check 없이도 코드 위치가 의도를 설명한다. 코드 리뷰 시 "왜 이 파일에 org_pk 조건이 없지?"라는 의문이 생기지 않는다.

### 일반 API에서 테넌트 격리는 제거 불가능하다

일반 모듈(lecture, rag, homework)의 서비스 레이어는 항상 org_pk를 포함하는 인터페이스를 사용한다. 이를 우회하려면 cross-tenant 모듈로 코드를 이동해야 하며, 이동 자체가 "이 코드는 cross-tenant 접근이 필요하다"는 명시적 선언이 된다.

### 공격 표면이 없다

탈취할 admin role 자체가 존재하지 않는다. Cross-tenant 조회는 별도 서비스(또는 내부 네트워크 전용 컨트롤러)에서만 가능하며, 일반 API 토큰으로는 접근할 수 없다.

---

## 트레이드오프 비교

| 기준 | Admin Role 방식 | 아키텍처 분리 방식 |
|---|---|---|
| 보안 | 단일 장애점 — role 탈취 시 전 테넌트 노출 | 공격 표면 없음 |
| 코드 가독성 | 비즈니스 로직에 보안 분기 혼재 | 파일 위치로 의도 명시 |
| 코드 리뷰 | org_pk 누락을 role 분기로 정당화 가능 | org_pk 없으면 cross-tenant 모듈로만 |
| 확장성 | 서비스 커질수록 role 남용 확산 | 별도 서비스로 분리 자연스러움 |
| Drizzle 활용 | raw query 이점 희석 | raw query 이점 최대 활용 |
| 초기 구현 비용 | 낮음 | 중간 (internal/ 모듈 추가) |

---

## 구현 규칙 (Enforcement Rules)

1. `modules/lecture/`, `modules/rag/`, `modules/homework/` 내 모든 서비스 메서드는 `orgPk` 또는 `orgId`를 파라미터로 받아야 한다.
2. 위 모듈에서 org 조건 없는 쿼리가 발견되면 PR 기각.
3. Cross-tenant 조회가 필요한 경우 `modules/internal/` 또는 별도 admin 서비스로만 구현.
4. `internal/` 컨트롤러는 외부 라우팅(Guard 없는 공개 엔드포인트)에 노출 금지 — 내부 네트워크 또는 별도 인증 메커니즘 적용.
5. Qdrant / Neo4j 어댑터 인터페이스에서 `orgId`/`orgPublicId`는 필수 타입 선언 유지 — optional로 변경 금지.

---

## 향후 전환 경로

| 단계 | 조건 | 행동 |
|---|---|---|
| 현재 | 단일 서비스, 운영 초기 | `internal/` 모듈로 cross-tenant 쿼리 격리 |
| 규모 확장 시 | admin 집계 트래픽 증가 or 보안 감사 요구 | `academy-admin` 별도 서비스 분리, 내부 네트워크 전용 배포 |
| 멀티테넌트 DB 분리 시 | ADR-041 T1~T4 트리거 발동 | cross-tenant 서비스가 각 DB에 연결하는 fan-out 쿼리로 전환 |

---

## 참조

| 문서 | 내용 |
|---|---|
| ADR-035 | Qdrant 멀티테넌시 — shared-collection + payload 필터 |
| ADR-036 | Neo4j 개념 그래프 — orgId 격리 원칙 |
| ADR-041 | MySQL 멀티테넌시 — org_pk 행 레벨 격리 + 분리 트리거 |
| `apps/academy-api/src/infrastructure/neo4j/neo4j-concept.adapter.ts` | Neo4j 어댑터 실제 구현 |
