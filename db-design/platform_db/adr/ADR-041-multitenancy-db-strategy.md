# ADR-041: 멀티테넌시 DB 전략 — 단일 DB 행(row) 격리 + 분리 트리거 사전 정의

| 항목 | 내용 |
|---|---|
| **상태** | Accepted |
| **날짜** | 2026-05-27 |
| **타입** | Architecture |
| **관련 문서** | `docs/academy/v3/data-model.md`, `docs/academy/v3/db-design-decisions.md`, ADR-035 (Qdrant 멀티테넌시) |

---

## 맥락

이 시스템은 SaaS 학원 플랫폼이다. 테넌트는 `organization`(학원/org)이며, 현재 `academy_db` MySQL 인스턴스 하나에 모든 테넌트 데이터를 함께 저장한다.

### 고속 증가 테이블 (폭주 가능 대상)

시스템 내에서 테넌트 규모 증가에 따라 행(row) 수가 폭발적으로 증가할 수 있는 테이블:

| 테이블 | 발생 조건 | 특성 |
|---|---|---|
| `homework_chat` | 학생이 AI 숙제 agent와 대화할 때마다 1행 | 학생 수 × 숙제 수 × 대화 턴 수 |
| `usage_log` | LLM/TTS/STT API 호출마다 1행 (이미 월별 파티셔닝) | append-only, 삭제 없음 |
| `notification` | 알림톡·이메일 발송마다 1행 | 전송 실패 재시도 포함 |
| `student_qa_chat` (future) | 학생 강의 Q&A 대화 | `homework_chat`과 같은 패턴 |
| `audit_log` (future) | 모든 데이터 변경 감사 | 전체 테넌트 합산, append-only |

---

## 현재 전략: 단일 DB + 행 수준 `org_pk` 격리

### 원칙

**`academy_db` 내 모든 테이블은 `org_pk NOT NULL`을 보유한다.** 이것은 불변식(invariant)으로, 예외 없이 적용된다.

```sql
-- 모든 조회는 반드시 org_pk 조건을 포함해야 함
SELECT * FROM homework_chat WHERE org_pk = ? AND homework_pk = ?;

-- cross-tenant 조회는 절대 없음 (운영 batch 포함)
-- ❌ SELECT * FROM homework_chat WHERE student_pk = ?  ← org_pk 누락 금지
```

### 현재 구조에서 이미 적용된 격리 장치

| 장치 | 위치 | 역할 |
|---|---|---|
| `org_pk NOT NULL` 컬럼 | 모든 `academy_db` 테이블 | 행 소속 테넌트 식별 |
| `org_pk` 복합 인덱스 | 조회 빈도 높은 테이블 전체 | 크로스-테넌트 풀스캔 방지 |
| `usage_log` 월별 파티셔닝 | `academy_db.usage_log` | 오래된 파티션 분리·아카이브 가능 |
| `org_entitlement.feature_limits` | `identity_db` | 테넌트별 사용량 상한 선 제어 |
| `llm_quota_usage` | `academy_db` | 실시간 LLM 사용량 카운터 (초과 방지) |

### 왜 지금은 단일 DB가 적합한가

- 테넌트 수가 수십~수백 수준: 단일 MySQL 인스턴스로 충분
- 운영 복잡성 최소화 (단일 백업·복구·마이그레이션 경로)
- 크로스-테넌트 통계(운영 대시보드) 단순 구현 가능
- DB-per-tenant 전환 시 코드 변경 범위를 최소화하도록 애플리케이션 레이어가 항상 `org_pk`를 전달하는 패턴 유지 중

---

## 분리 트리거 (Trigger)

다음 조건 중 하나라도 충족되면 해당 섹션의 분리 작업을 시작한다.

### T1 — 단일 테넌트 과점

```
단일 org가 homework_chat 또는 usage_log 총 행 수의 20% 이상 차지
```

→ 해당 테넌트만 전용 DB 인스턴스로 이전 (DB-per-large-tenant)

### T2 — 쿼리 성능 저하

```
homework_chat 또는 student_qa_chat 조회 P99 > 500ms (5분 평균)
```

→ 해당 테이블을 별도 MySQL 인스턴스로 분리

### T3 — usage_log 볼륨

```
usage_log 월 INSERT 수 > 500만 건 OR 총 크기 > 50GB
```

→ ClickHouse 또는 BigQuery로 분석용 usage_log 이관. `academy_db.usage_log`는 최근 3개월만 유지.

### T4 — 규제 요건

```
ISMS-P 인증 심사 또는 GDPR 적용 국가 고객 계약
```

→ 해당 테넌트 데이터를 물리적으로 격리된 DB 인스턴스(또는 별도 리전)로 이전

---

## 단계별 분리 로드맵

### Phase A — 읽기 복제본 (트리거 이전, 선제적 가능)

```
academy_db (Primary)
    └── Read Replica × 1~2
         ├── 통계/리포트 쿼리
         └── 학생 Q&A RAG 검색 (읽기 전용)
```

운영 코드 변경 없이 DB 드라이버 레벨에서 read/write 분기.

### Phase B — 고속 증가 테이블 분리 (T2, T3 도달 시)

```
academy_db        → 강의·강사·학원 마스터 데이터
chat_db           → homework_chat, student_qa_chat
analytics_db      → usage_log (ClickHouse or MySQL)
```

**애플리케이션 영향**: `DatabaseModule`을 `AcademyDatabaseModule` + `ChatDatabaseModule` + `AnalyticsDatabaseModule`으로 분리. NestJS 모듈 경계가 이 분리를 자연스럽게 수용한다.

```typescript
// 분리 후 변경 범위 예시
@Module({
  providers: [
    { provide: CHAT_DB_TOKEN, useFactory: ... },   // chat_db
    { provide: ACADEMY_DB_TOKEN, useFactory: ... }, // academy_db
  ]
})
```

### Phase C — DB-per-large-tenant (T1, T4 도달 시)

```
academy_db (shared, SMB)
    ├── org_pk 1~999  (소규모 학원)
    └── org_pk 1000+  → 각각 dedicated 인스턴스
```

**라우팅**: 연결 시작 시 `org_pk → DB connection string` 매핑 테이블 조회 (Redis 캐시). 코드 레벨에서 `org_pk`를 이미 항상 전달하므로 라우터 레이어만 추가하면 된다.

---

## 대화 데이터(`homework_chat`) 설계 상세

대화 기록은 양이 많고 삭제 요청(PIPA/GDPR)이 발생할 수 있어 별도 관리가 필요하다.

### 현재 스키마 (`academy_db.homework_chat`)

```sql
homework_chat (
  pk          BIGINT UNSIGNED AUTO_INCREMENT,
  org_pk      BIGINT UNSIGNED NOT NULL,   -- 테넌트 경계
  homework_pk BIGINT UNSIGNED NOT NULL,
  student_pk  BIGINT UNSIGNED NOT NULL,
  turn_seq    SMALLINT UNSIGNED NOT NULL, -- 대화 순서
  role        ENUM('student', 'assistant'),
  content     TEXT NOT NULL,
  created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_org_hw (org_pk, homework_pk, turn_seq)
)
```

### 보존·삭제 정책

| 이벤트 | 처리 |
|---|---|
| 학생 탈퇴 | `homework_chat.content` → `'[삭제됨]'` soft-delete (PIPA §22) |
| 학원 해지 | org 데이터 전체 90일 보관 후 하드 삭제 |
| 오래된 대화 아카이브 | 1년 초과 행 → S3 Parquet 이관 + DB 행 삭제 (비용 관리) |

### chat_db 분리 시 마이그레이션 절차 (Phase B 참고)

```bash
# 1. chat_db 인스턴스 생성 + 동일 스키마 적용
# 2. 배치: SELECT org_pk 순서로 INSERT INTO chat_db.homework_chat
# 3. 애플리케이션 배포: CHAT_DB_TOKEN → chat_db 연결
# 4. org_pk별 컷오버 (무중단 가능)
# 5. academy_db.homework_chat 삭제
```

`org_pk` 단위로 데이터가 완전히 독립되어 있으므로 org 단위 무중단 마이그레이션이 가능하다.

---

## 운영 모니터링 항목

분리 트리거를 조기에 감지하기 위한 주간 확인 항목:

```sql
-- 테넌트별 homework_chat 점유율
SELECT org_pk, COUNT(*) as rows,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 1) as pct
FROM homework_chat
GROUP BY org_pk ORDER BY rows DESC LIMIT 10;

-- usage_log 월별 증가량
SELECT DATE_FORMAT(created_at, '%Y-%m') as month,
       COUNT(*) as rows, ROUND(SUM(cost_usd), 2) as total_cost_usd
FROM usage_log
GROUP BY month ORDER BY month DESC LIMIT 6;
```

---

## 폐기 결정

| 검토 옵션 | 기각 사유 |
|---|---|
| schema-per-tenant (MySQL) | MySQL은 database = schema. 테넌트마다 database 생성 시 connection pool 관리 복잡 |
| 즉시 DB-per-tenant | 현재 테넌트 규모에서 운영 비용만 증가, 실질 이점 없음 |
| NoSQL(MongoDB) 전환 | 기존 MySQL + Drizzle ORM 투자 보호, 관계형 모델이 현재 요구사항에 적합 |

---

## 요약

> **지금**: 단일 `academy_db`, `org_pk` 행 격리. 모든 테이블에 `org_pk NOT NULL` 불변식.  
> **중기**: `usage_log` → ClickHouse, `homework_chat` → `chat_db` 분리 (트리거 T2/T3).  
> **장기**: 대형 테넌트 → DB-per-tenant, SMB는 shared pool 유지.  
> **지금 코드에서 보장할 것**: 모든 쿼리에 `org_pk` 필터 필수, cross-tenant JOIN 없음.
