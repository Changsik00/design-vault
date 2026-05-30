---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p0
  - operability
  - usage
  - metering
  - billing
aliases:
  - usage_snapshot
  - 사용량 미터링
  - 미터링
  - USAGE-1
  - 사용량 가시성
---

# 사용량 가시성과 미터링 설명 — usage_snapshot

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[operability|operability.md O4]] · [[requirements]] USAGE-1, ABAC-3

[[feature-limits]]는 "이 학원은 영상을 하루 10개까지 올릴 수 있다"는 **한도(limit)** 를 알려줍니다. 그런데 정작 사용자가 가장 궁금해하는 건 다른 질문입니다: **"그래서 지금 몇 개나 썼는데?"** 이 문서는 "지금 얼마나 썼나"를 platform이 어떻게 보관하고 보여주는지(`usage_snapshot`), 그리고 미터링·한도·과금이 어떻게 다른지 설명합니다.

---

## Q1. "한도가 얼마"는 아는데 "지금 얼마나 썼나"는 왜 따로 필요한가요?

`feature_limits`는 **상한선**만 담습니다. 사용량은 담지 않습니다.

```json
// org_entitlement.feature_limits — "한도가 얼마인가"만 안다
{ "members": 500, "daily_uploads": 10 }
// → "이 학원은 학생 500명까지"는 알지만
// → "지금 498명인지 5명인지"는 이 안에 없다
```

사용자(학원 관리자)가 백오피스에서 보고 싶은 건 한도 숫자 하나가 아니라 **`498 / 500`** 이라는 두 숫자입니다. 분자가 없으면 화면에 막대그래프를 그릴 수 없습니다.

이게 단순한 UX 문제가 아닌 이유는, **업그레이드 결정**이 여기 달려 있기 때문입니다.

```
가시성 없음 → 학생 500명 꽉 참 → 501번째 등록 시도 실패 →
"왜 안 돼요?" 문의 → CS 대응 → 그제서야 업그레이드
   (사용자도 우리도 모두 손해)

가시성 있음 → "80% 찼어요. 곧 한도예요" 미리 경고 →
사용자가 여유 있게 PRO로 업그레이드
   (사고가 아니라 자연스러운 매출 전환)
```

[[operability|operability.md]] O4는 이 갭을 운영의 **치명상**이라고 부릅니다 — "학생 500명 제한인데 지금 498인지 5인지 백오피스가 알 수 없다." 한도만 있고 사용량 가시성이 없으면, 한도는 사고가 터진 뒤에야 체감되는 숫자가 됩니다.

> 💡 **한 줄 요약**: `feature_limits`는 분모(상한)만 알려줍니다. 분자(현재 사용량)가 있어야 `498/500`을 보여주고, 한도에 닿기 전에 업그레이드를 유도할 수 있습니다.

---

## Q2. usage_snapshot이 뭔가요? raw 이벤트랑 뭐가 다른가요?

`usage_snapshot`은 사용량의 **집계 스냅샷(aggregate snapshot)** 을 platform_db에 보관하는 테이블입니다. "이벤트 하나하나"가 아니라 "합쳐진 결과"만 담습니다.

```sql
-- usage_snapshot: 집계된 결과만 (O4)
CREATE TABLE usage_snapshot (
  org_pk      BIGINT UNSIGNED NOT NULL,
  service     VARCHAR(32)     NOT NULL,  -- 'ACADEMY' 등
  metric      VARCHAR(64)     NOT NULL,  -- 'members', 'daily_uploads' ...
  period      VARCHAR(16)     NOT NULL,  -- '2026-05', '2026-05-30' (집계 단위)
  used        BIGINT          NOT NULL,  -- 분자: 지금까지 쓴 양
  `limit`     BIGINT          NULL,      -- 분모: feature_limits 대비 (NULL=무제한)
  source_ts   DATETIME        NOT NULL,  -- 이 집계가 어느 시점 기준인지
  PRIMARY KEY (org_pk, service, metric, period)
);
```

핵심은 **무엇을 어디에 두느냐**의 분리입니다(O4 원칙).

```
이벤트 전량 (raw)        →  서비스 DB
  매 업로드, 매 AI 쿼리 로그 한 줄 한 줄

집계 스냅샷 (aggregate)  →  platform_db.usage_snapshot
  "이번 달 영상 업로드 합계 = 7건" 같은 결과만
```

비유하자면, raw 이벤트는 **가게의 모든 영수증**이고 `usage_snapshot`은 **월말 매출 요약표**입니다. 요약표 한 장 보려고 영수증 수만 장을 회계실에 다 쌓아둘 필요는 없습니다.

**왜 raw를 platform에 다 넣지 않나요?**

`platform_db`는 모든 서비스가 인증을 위해 매 요청 의존하는 공통 코어입니다(SPOF). 여기에 업로드/쿼리 raw 로그를 전부 쏟아부으면:

```
학원 1000개 × 하루 AI 쿼리 수천 건 = 일 수백만 row INSERT
  → platform_db 쓰기 폭증 + 핫스팟
  → 인증 쿼리까지 같이 느려짐 (전 서비스 영향)
```

이건 [[rate-limiting]]이 "실시간 카운터를 platform DB에 두지 않는" 이유와 정확히 같습니다 — 고빈도 쓰기는 공통 코어에서 격리합니다. platform은 결과(요약)만 받습니다.

> 💡 **한 줄 요약**: `usage_snapshot`은 사용량의 *집계 결과*만 담는 테이블입니다. raw 이벤트 전량은 서비스 DB에 두고, platform에는 "498/500" 같은 요약만 올려 공통 코어의 쓰기 폭증을 막습니다.

---

## Q3. 미터링, 한도, 과금이 헷갈려요. 같은 거 아닌가요?

셋 다 "사용량"을 다루지만 역할이 완전히 다릅니다. 전기로 비유하면 깔끔합니다.

| 개념 | 영어 | 하는 일 | 전기 비유 | 사는 곳 |
|---|---|---|---|---|
| **미터링** | metering | 얼마 썼나 *측정·집계* | 계량기(이번 달 350kWh) | `usage_snapshot` |
| **한도** | limit / quota | 상한선 *정의* | 계약 용량(5kW 이상 못 씀) | [[feature-limits]] |
| **과금형** | usage-based billing | 쓴 만큼 *청구* | 요금 청구서 | `usage_snapshot` → billing |

```
미터링: "이번 달 영상 7개 올렸음"        (사실 측정)
한도:   "이번 달 10개까지 가능"          (규칙)
과금:   "초과분 1개당 1000원 청구"       (돈 계산)
```

세 개는 **순서대로 연결**됩니다. 미터링이 `used`를 세고 → 한도와 비교해 가시성/차단 판단을 하고 → 과금형이면 그 `used`로 청구액을 계산합니다.

**중요: 과금형도 같은 usage_snapshot 경로를 씁니다 (USAGE-1).**

"쓴 만큼 청구하는" 과금형(metered/usage-based billing)을 새로 만들 때, 별도의 정산 파이프라인을 짤 필요가 없습니다. 정확성 요구가 더 높을 뿐(돈이니까) 데이터 경로는 동일합니다.

```
가시성용:  서비스 집계 → usage_snapshot → "498/500" 화면
과금용:    서비스 집계 → usage_snapshot → 청구액 산출 (정산 정확성↑)
           └────────── 같은 테이블, 같은 경로 ──────────┘
```

[[feature-limits]]의 한도 *우선순위*(어느 테이블을 SSOT로 읽느냐, 불변식 #10)는 그 문서가 다룹니다. 여기서는 한도가 미터링·과금과 **다른 역할**이라는 구분만 잡으면 됩니다.

> 💡 **한 줄 요약**: 미터링=얼마 썼나 측정(`usage_snapshot`), 한도=상한선([[feature-limits]]), 과금형=쓴 만큼 청구. 셋은 별개지만 과금형도 가시성과 똑같은 `usage_snapshot` 경로를 씁니다.

---

## Q4. 사용량 집계가 정확하지 않으면 안 되나요? 실시간 카운터랑은 뭐가 다른가요?

여기서 두 가지 다른 목적의 카운터를 구분해야 합니다.

```
실시간 enforcement 카운터  →  서비스측 (Redis 등), ABAC-3
  "501번째 업로드를 지금 막아야 하나?" — 즉각·정확

집계 스냅샷             →  platform usage_snapshot
  "이번 달 80% 찼어요" — 주기적·결과적 일관성으로 충분
```

`usage_snapshot`은 **결과적 일관성(eventual consistency)** 으로 충분합니다([[consistency-model]]). 왜냐하면 이 데이터의 목적이 **차단이 아니라 가시성·경고**이기 때문입니다.

```
용도가 "한도 근처 경고"라면:
  498인지 499인지 1초 차이 → 무관 (둘 다 "거의 다 찼다")
  스냅샷이 5분 전 기준 → 괜찮음 (사람이 5분마다 한도 채우진 않음)
```

반대로, **실제 차단(enforcement)** 은 서비스측 빠른 카운터가 담당합니다(ABAC-3). "501번째 업로드를 막아야 하는" 순간엔 강한 정확도가 필요하고, 이건 요청 경로 안에서 빠르게 도는 Redis 카운터의 일입니다. `usage_snapshot`은 그 책임을 지지 않습니다.

```typescript
// ❌ usage_snapshot으로 실시간 차단 — 잘못된 사용
const snap = await getUsageSnapshot(orgPk, 'ACADEMY', 'daily_uploads');
if (snap.used >= snap.limit) throw new LimitExceeded();
//  스냅샷은 주기 집계라 지금 이 순간 정확하지 않음 → 차단 누락/오차단

// ✅ 차단은 서비스측 실시간 카운터 (ABAC-3)
const current = await redis.incr(`uploads:${orgPk}:${today}`);
if (current > limit) throw new LimitExceeded();   // 즉각·정확

// ✅ usage_snapshot은 가시성·경고·과금에만
// "80% 찼습니다" 배너, "498/500" 막대그래프, 월말 청구
```

전기 비유로 돌아오면: 계량기 숫자를 **매초 0.001kWh 단위로 정밀하게** 들여다볼 필요는 없습니다. "이번 달 80% 썼어요"라는 경고면 사용자가 행동(절약/증설)하기에 충분합니다. 차단(누전 차단기)은 별도의 즉각 장치가 합니다.

> 💡 **한 줄 요약**: `usage_snapshot`은 가시성·경고·과금용이라 결과적 일관성으로 충분합니다. "501번째를 지금 막는" 실시간 차단은 서비스측 빠른 카운터(ABAC-3)의 책임이고, 둘은 목적이 다릅니다.

---

## Q5. 집계가 1초 뒤처져도 괜찮다면, 서비스 raw 합계랑 스냅샷이 안 맞아도 되나요?

"결과적 일관성 OK"가 "아무 숫자나 OK"라는 뜻은 아닙니다. 스냅샷은 결국 **서비스 raw 이벤트의 합과 (허용 오차 안에서) 일치**해야 합니다.

```
서비스 DB raw 이벤트 합   =  usage_snapshot.used   (허용 오차 내)

차이가 나는 정상 원인:
  - 집계 주기 (5분마다 push → 최대 5분치 누락 가능)
  - in-flight 이벤트 (집계 시점 직후 들어온 것)

차이가 나면 안 되는 비정상:
  - 집계 로직 버그 (이벤트 누락/중복 카운트)
  - push 실패 후 스냅샷이 옛날 값에 멈춤 (source_ts가 안 갱신)
```

그래서 `source_ts`가 중요합니다. 스냅샷이 "어느 시점 기준 집계인지"를 담아야, 백오피스가 "이 숫자는 5분 전 기준입니다" 같은 신선도를 표시하고, 운영자가 "스냅샷이 며칠째 안 갱신됐다 = 집계 파이프라인 고장"을 감지할 수 있습니다.

검증의 핵심은 **방향**입니다.

```
정합성 검증: |snapshot.used - Σ(서비스 raw 이벤트)| ≤ 허용오차
신선도 검증: NOW() - snapshot.source_ts ≤ 집계주기 + 여유
```

> 💡 **한 줄 요약**: 결과적 일관성이라도 스냅샷은 서비스 raw 합과 허용 오차 내에서 맞아야 합니다. `source_ts`로 "언제 기준 집계인지"를 추적해 정합성과 신선도를 모두 검증합니다.

---

## 용어 정리

| 용어 | 영어 | 뜻 | 이 시스템에서 |
|---|---|---|---|
| 미터링 | metering | 사용량을 측정·집계하는 행위 | `usage_snapshot`이 결과를 보관 |
| 집계 스냅샷 | aggregate snapshot | raw가 아닌 합쳐진 결과만 담은 기록 | `usage_snapshot` 테이블 |
| raw 이벤트 | raw event | 업로드·쿼리 하나하나의 원본 로그 | **서비스 DB**에만 존재 |
| 한도 | limit / quota | 사용 가능한 상한선 | [[feature-limits]] `feature_limits` |
| 과금형 | usage-based / metered billing | 쓴 만큼 청구하는 요금 모델 | 같은 `usage_snapshot`에서 산출 |
| 실시간 카운터 | real-time counter | 요청 경로에서 즉각 차단용 카운터 | **서비스측 Redis**(ABAC-3) |
| 결과적 일관성 | eventual consistency | 잠시 뒤처져도 결국 맞는 일관성 모델 | 스냅샷의 일관성 수준([[consistency-model]]) |
| source_ts | — | 스냅샷이 어느 시점 기준 집계인지 | 신선도·정합성 검증 키 |

---

## 테스트 방법

### 1. 스냅샷이 서비스 raw 합과 일치하나 (허용 오차)

```typescript
// 서비스 DB의 raw 이벤트를 직접 합산
const rawSum = await serviceDb.query(`
  SELECT COUNT(*) AS cnt FROM upload_event
  WHERE org_pk = ? AND DATE(created_at) = ?
`, [orgPk, today]);

const snap = await getUsageSnapshot(orgPk, 'ACADEMY', 'daily_uploads', today);

// 집계 주기/in-flight 고려한 허용 오차
const TOLERANCE = 5;
expect(Math.abs(snap.used - rawSum.cnt)).toBeLessThanOrEqual(TOLERANCE);
```

### 2. 한도 80% / 100% 도달 시 가시성 데이터가 나오나

```typescript
// limit=500, used=400 (80%) 상황을 만든 뒤
const snap = await getUsageSnapshot(orgPk, 'ACADEMY', 'members');
const ratio = snap.used / snap.limit;
expect(ratio).toBeCloseTo(0.8);
// → 프론트가 "80% 사용" 경고 배너를 그릴 수 있는 used/limit이 둘 다 존재
expect(snap.used).toBeDefined();
expect(snap.limit).toBeDefined();
```

### 3. raw 이벤트는 platform_db에 없어야 한다

```typescript
// platform_db에는 raw 이벤트 테이블이 없어야 함 (O4 분리 원칙)
const tables = await platformDb.query(`SHOW TABLES LIKE '%upload_event%'`);
expect(tables.length).toBe(0);   // raw는 서비스 DB 소관

// platform에는 집계 스냅샷만
const snapExists = await platformDb.query(`SHOW TABLES LIKE 'usage_snapshot'`);
expect(snapExists.length).toBe(1);
```

### 4. 과금형도 같은 스냅샷에서 산출되나 (USAGE-1)

```typescript
// 과금 계산이 별도 파이프라인이 아니라 usage_snapshot을 읽는지 확인
const snap = await getUsageSnapshot(orgPk, 'AGENT', 'tokens', '2026-05');
const charge = calculateMeteredCharge(snap);   // 같은 used를 입력으로
expect(charge.basedOn).toBe('usage_snapshot');
expect(charge.usedTokens).toBe(snap.used);
```

### 5. 신선도 — source_ts가 갱신되나

```typescript
const snap = await getUsageSnapshot(orgPk, 'ACADEMY', 'members');
const ageMs = Date.now() - new Date(snap.source_ts).getTime();
const AGG_INTERVAL_MS = 5 * 60_000;        // 집계 주기 5분
expect(ageMs).toBeLessThan(AGG_INTERVAL_MS + 60_000);  // 주기 + 여유
// 며칠째 안 갱신 = 집계 파이프라인 고장 신호
```

### 체크리스트

```
□ usage_snapshot.used 가 서비스 raw 이벤트 합과 허용 오차 내 일치
□ used/limit 둘 다 제공되어 "498/500" 가시화 가능
□ 80% / 100% 도달이 경고 데이터로 드러남
□ raw 이벤트 테이블은 platform_db에 없음 (서비스 DB에만)
□ 과금형 청구액이 같은 usage_snapshot에서 산출됨 (별도 파이프라인 아님)
□ 실시간 차단은 서비스측 카운터(ABAC-3)로, usage_snapshot으로 차단하지 않음
□ source_ts로 신선도·정합성 검증 가능
```

---

## 마치며

사용량 미터링은 결국 두 가지 분리로 요약됩니다.

**무엇을 어디에 두나 (저장 분리)**
```
raw 이벤트 전량  →  서비스 DB    (고빈도 쓰기, platform에 넣으면 핫스팟)
집계 스냅샷     →  platform     (498/500 요약만, 공통 코어 보호)
```

**무엇이 정확해야 하나 (책임 분리)**
```
실시간 차단     →  서비스 카운터  (강한 정확도, ABAC-3)
가시성·경고·과금 →  usage_snapshot (결과적 일관성으로 충분)
```

새 기능에서 사용량을 다룰 때 스스로 물어보세요: **"이 숫자가 사람을 막는 데 쓰이나(차단), 보여주는 데 쓰이나(가시성)?"** 막는 거라면 서비스측 실시간 카운터, 보여주거나 청구하는 거라면 `usage_snapshot`입니다. 그리고 raw 로그를 platform_db에 넣고 싶은 충동이 들면 — 그건 [[rate-limiting]]이 막았던 바로 그 핫스팟입니다.

---

## 연결된 개념

- [[feature-limits]] — 한도(분모)의 SSOT와 3중 정의 우선순위 (이 문서는 분자=사용량)
- [[rate-limiting]] — 고빈도 카운터를 platform 코어에서 격리하는 같은 원리
- [[consistency-model]] — 스냅샷이 결과적 일관성으로 충분한 이유
- [[gate-b-entitlement|Gate B & 엔타이틀먼트]] — entitlement.feature_limits가 한도 평가의 입력
> 소스 문서
- [[operability]] — O4 Usage & Metering (이벤트=서비스 / 집계=platform usage_snapshot)
- [[requirements]] — USAGE-1 (사용량 가시성), ABAC-3 (한도 평가는 entitlement, 카운터는 서비스측)
