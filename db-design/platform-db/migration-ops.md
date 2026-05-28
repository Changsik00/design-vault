# platform_db 마이그레이션 운영 가이드

> 작성일: 2026-05-28 · 대상: 운영 DB 최초 구축 또는 기존 DB 마이그레이션 담당자

---

## 1. 현재 마이그레이션 상태

| 파일 | 내용 | journal 등록 | 비고 |
|---|---|---|---|
| `0000_uneven_swordsman.sql` | 최초 스키마 (drizzle-kit 자동 생성) | ✅ idx=0 | snapshot 유일 기준점 |
| `0001_identity_complete.sql` | identity 테이블 완성 | ✅ idx=1 | 수동 작성 |
| `0002_billing_complete.sql` | billing 테이블 완성 | ✅ idx=2 | 수동 작성 |
| `0003_schema_invariants.sql` | FK·CHECK 제약 추가 | ✅ idx=3 | 수동 작성 |
| `0004_service_type_extensible.sql` | service 타입 확장 | ✅ idx=4 | 수동 작성 |
| `0005_identity_verification.sql` | email/phone 인증 컬럼 | ✅ idx=5 | 수동 작성 |
| `0006_schema_p1_p3_fixes.sql` | P1/P2/P3 스키마 수정 | ✅ idx=6 | 수동 작성 |

**0006 변경 내역 (P1/P2/P3)**:
- `org_subscription.sku_pk` 제거 (N:M 전환)
- `subscription_item.product_pk` → `sku_pk` + FK 추가
- `org_entitlement` 만료 인덱스 추가
- `audit_log.break_glass` 컬럼 추가
- `user_consent_event` 신규 테이블 생성 (PIPA §17)

---

## 2. 로컬 초기 구축 (DB 없는 상태)

```bash
# DB 생성
mysql -u root -e "CREATE DATABASE IF NOT EXISTS platform_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# TS 스키마 직접 push (migration 없이, 가장 빠름)
cd packages/db-platform
PLATFORM_DB_URL="mysql://aiagent@localhost:3306/platform_db" pnpm run db:platform:push
```

> `push`는 `_journal.json`·snapshot을 사용하지 않고 현재 TS 스키마를 그대로 DB에 반영한다.
> 로컬 개발에만 사용. 운영 DB에는 절대 사용 금지.

---

## 3. 운영 DB 최초 구축 (신규 서버)

```bash
# 1. DB 생성
mysql -u <user> -h <host> -e "CREATE DATABASE platform_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# 2. 전체 마이그레이션 순차 적용 (0000 → 0006)
PLATFORM_DB_URL="mysql://<user>:<pass>@<host>:3306/platform_db" \
  pnpm --filter @aiagent/db-platform run db:platform:migrate
```

`drizzle-kit migrate`는 `__drizzle_migrations` 테이블을 생성하고 journal 순서대로 적용.
이미 적용된 항목은 건너뜀.

---

## 4. 운영 DB 점진적 마이그레이션 (기존 DB에 추가)

```bash
# journal 기준으로 미적용 항목만 자동 적용
PLATFORM_DB_URL="mysql://<user>:<pass>@<host>:3306/platform_db" \
  pnpm --filter @aiagent/db-platform run db:platform:migrate
```

`__drizzle_migrations` 테이블에 이미 기록된 migration은 스킵.

---

## 5. 알려진 기술 부채

### ⚠️ drizzle-kit generate BigInt 직렬화 버그

**증상**:
```
TypeError: Do not know how to serialize a BigInt
```

**원인**: drizzle-orm의 `check()` 제약 내부 BigInt 값이 snapshot JSON 직렬화 시 오류.

**영향**:
- `db:platform:generate` 명령 사용 불가
- snapshot (`0000_snapshot.json`) 이 0000 시점에 멈춰있음
- 다음 번 generate 실행 시 0001-0006 전체를 포함한 메가 마이그레이션이 생성될 수 있음

**대응**:
- 신규 스키마 변경은 **수동 SQL 작성 + journal 수동 등록** 방식 유지
- drizzle-kit 버전 업그레이드 시 BigInt 버그 해결 확인 후 generate 재활성화
- 차기 마이그레이션 파일 이름 규칙: `0007_<slug>.sql`, journal idx=7

**journal 수동 등록 방법** (`drizzle/platform/meta/_journal.json`):
```json
{
  "idx": 7,
  "version": "5",
  "when": <unix_ms>,
  "tag": "0007_<slug>",
  "breakpoints": true
}
```

### ⚠️ snapshot과 실제 스키마 불일치

`0000_snapshot.json`은 초기 상태만 반영. 현재 스키마와 0001-0006 diff 누적 중.
→ generate 버그 해결 전까지는 push(로컬) 또는 migrate(운영) 사용.

---

## 6. 롤백 절차

migration에는 DOWN이 없음. 롤백 필요 시:

```sql
-- 0006 롤백 (순서 역순)
DROP TABLE IF EXISTS user_consent_event;
ALTER TABLE audit_log DROP COLUMN break_glass, DROP INDEX idx_audit_break_glass;
ALTER TABLE org_entitlement DROP INDEX idx_entitlement_expiry;

-- subscription_item 롤백 (FK 먼저 제거)
ALTER TABLE subscription_item
  DROP FOREIGN KEY fk_sub_item_sku,
  DROP FOREIGN KEY fk_sub_item_sub,
  DROP INDEX uq_sub_sku,
  DROP COLUMN sku_pk,
  ADD COLUMN product_pk BIGINT UNSIGNED NOT NULL,
  ADD UNIQUE KEY uq_sub_product (subscription_pk, product_pk);

-- org_subscription 롤백
ALTER TABLE org_subscription
  ADD COLUMN sku_pk BIGINT UNSIGNED NOT NULL AFTER payer_user_pk,
  ADD INDEX idx_org_subscription_sku (sku_pk);
```

---

## 7. 체크리스트 (운영 배포 전)

- [ ] `platform_db` DB 존재 확인
- [ ] `PLATFORM_DB_URL` 환경변수 설정 확인
- [ ] 배포 전 DB 백업 완료
- [ ] `drizzle-kit migrate` dry-run으로 적용 목록 확인
- [ ] 적용 후 `__drizzle_migrations` 테이블에서 0000-0006 전부 기록됐는지 확인
- [ ] `org_entitlement`, `audit_log`, `user_consent_event` 테이블 구조 확인
