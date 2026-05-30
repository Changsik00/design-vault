---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p1
  - rate-limit
  - quota
  - security
  - gateway
aliases:
  - 속도 제한
  - rate limit
  - 레이트 리밋
  - quota
  - TEN-9
  - 429
---

# 속도 제한(rate limit)과 한도(quota) 설명 — 무엇을 어디서 막나

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[requirements]] TEN-9 · [[schema-reference]] §J api_key · [[architecture]] §3.1 · [[bdd-scenarios]] P10-04

"이 학원은 월 업로드 500개까지", "이 API 키는 초당 10요청까지" — 둘 다 "쓸 수 있는 양을 제한한다"는 점은 같지만, 사실 완전히 다른 개념입니다. 앞은 **한도(quota)**, 뒤는 **속도 제한(rate limit)**입니다. 둘을 혼동하면 "왜 quota는 DB에 있는데 rate limit은 DB에 없지?"라는 의문에서 막히게 됩니다. 이 문서는 이 둘을 구분하고, `platform_db`가 *어디까지 책임지고 어디부터는 엣지(Gateway)에 넘기는지*를 설명합니다.

> 한도(quota)의 *우선순위*(어느 테이블을 읽나, `org_entitlement` SSOT)는 [[feature-limits]]에서 다룹니다. 이 문서는 "quota vs rate limit의 **개념 차이**"와 "**누가 어디서 강제하나**(위치)"에 집중합니다.

---

## Q1. quota(한도)와 rate limit(속도 제한)은 뭐가 다른가요? 같은 거 아닌가요?

둘 다 사용을 제한하지만, **제한하는 축(axis)이 다릅니다.**

- **quota(한도)** = *누적 총량*. "월 업로드 500개", "멤버 50명", "저장공간 100GB"처럼 **시간이 지나도 쌓이는 총합**을 제한합니다. 다 쓰면 끝(리셋 주기 전까지).
- **rate limit(속도 제한)** = *단위 시간당 빈도*. "초당 10요청", "분당 100건"처럼 **얼마나 빠르게** 호출하느냐를 제한합니다. 1초만 기다리면 다시 쓸 수 있습니다.

비유가 가장 빠릅니다. 휴대폰 데이터 요금제를 떠올려 보세요.

```
월 데이터 5GB           ← quota(한도). 다 쓰면 이번 달은 끝.
초당 다운로드 속도 제한   ← rate limit. 느릴 뿐, 막힌 건 아님. 기다리면 계속 받아짐.
```

또는 고속도로 톨게이트로 비유할 수 있습니다.

```
연간 통행권(1년 무제한 통과 가능 횟수)   ← quota
톨게이트 차선이 초당 통과시키는 차량 수    ← rate limit (밀리면 줄 서서 대기)
```

핵심 차이를 표로 정리하면:

| | quota(한도) | rate limit(속도 제한) |
|---|---|---|
| 제한 축 | 누적 총량 | 단위 시간당 빈도 |
| 예시 | 월 업로드 500개 | 초당 10요청 |
| 다 쓰면? | 리셋 전까지 차단 (끝) | 잠깐 기다리면 재개 |
| 누가 정의? | 비즈니스/요금제 | 보안/인프라(남용 방어) |
| `platform_db` 위치 | `org_entitlement.feature_limits` | `api_key.rate_limit_tier` (정책만) |
| 강제(enforce) 위치 | 앱 레이어 (ABAC-3) | **Gateway/WAF** (엣지) |

❌ "quota와 rate limit은 둘 다 한도니까 같은 곳에서 같은 방식으로 막으면 된다"  
✅ "quota는 누적 총량을 앱이 평가하고, rate limit은 빈도를 엣지(Gateway)가 막는다 — 개념도 위치도 다르다"

> 💡 **한 줄 요약**: quota는 *얼마나 많이*(누적 총량), rate limit은 *얼마나 빨리*(단위 시간당 빈도)를 제한합니다. 다른 개념이라 강제하는 위치도 다릅니다.

---

## Q2. org 단위 quota(한도)는 어디서 강제하나요?

org 단위 quota는 **앱 레이어가 `org_entitlement.feature_limits`를 읽어서 평가**합니다. 이게 ABAC-3입니다.

흐름을 보면:

```typescript
// 앱이 org_entitlement.feature_limits를 읽어 평가 (ABAC-3)
const limit = await getFeatureLimit(orgPk, 'ACADEMY', 'daily_uploads'); // 예: 10
const used  = await countTodayUploads(orgPk, 'ACADEMY');                // 예: 10

if (used >= limit) {
  throw new QuotaExceededException(); // → 403 또는 422, "한도 초과"
}
```

여기서 `feature_limits`가 한도값의 최종 권위(SSOT)입니다. *어느 테이블을 읽어야 하는지*(왜 `product_feature`나 `plan_definition`이 아니라 `org_entitlement`인지)는 [[feature-limits]]에서 자세히 다룹니다. 여기서는 위치만 짚습니다:

```
한도값 보관:  org_entitlement.feature_limits   ← platform_db (SSOT)
              {"daily_uploads": 10, "members": 50, "storage_gb": 100}
한도 평가:    앱 레이어 (ABAC-3)               ← "지금 몇 개 썼나" 카운트와 비교
```

"지금 몇 개 썼나"의 실시간 카운트 자체는 서비스측에서 셉니다([[architecture]] §3.1: *실시간 한도 카운터는 서비스측*). platform_db는 *집계 스냅샷*만 `usage_snapshot`에 보관합니다(USAGE-1, [[architecture]] O4). 즉 "498/500인지 5/500인지"의 **가시성**은 platform이 제공하되, 매 업로드마다 카운트를 platform_db가 직접 세지는 않습니다.

> 💡 **한 줄 요약**: org quota는 platform_db의 `org_entitlement.feature_limits`(SSOT)를 기준으로 앱이 평가(ABAC-3)하고, 실시간 사용량 카운트는 서비스측이 셉니다.

---

## Q3. 머신(API 키) 단위 rate limit은 어디서 강제하나요? 왜 DB가 직접 안 막나요?

머신 단위 rate limit은 **platform_db가 *정책(tier)만 보관*하고, 실제 강제는 Gateway/WAF가** 합니다. 이게 TEN-9 결정의 핵심입니다.

> **TEN-9**: rate-limit 정책 위치 — org=`feature_limits` / 머신=`api_key.rate_limit_tier`, **강제는 Gateway** ([[requirements]] §3.1 결정)

platform_db는 API 키마다 "이 키는 어느 등급(tier)인가"만 분류해서 저장합니다:

```sql
-- api_key 테이블 (§J). rate_limit_tier는 SEC-7 보강 컬럼 (P1 예정)
-- 정책(tier)만 보관 — "이 키는 standard 등급" 같은 분류
SELECT key_prefix, rate_limit_tier
FROM api_key
WHERE pk = ?;
-- 결과: rate_limit_tier = 'standard'  (예: free=초당2, standard=초당10, premium=초당100)
```

그리고 Gateway가 이 tier 값을 받아 **실제 카운트와 차단**을 합니다:

```
요청 흐름:
  클라이언트 → [Gateway/WAF]  ← 여기서 rate limit 강제 (초당 카운트, 초과 시 429)
                   │ rate_limit_tier='standard' → 초당 10 허용
                   │ Redis로 실시간 카운터 유지
                   ↓ 통과한 요청만
              [앱 서버] → Gate A/B/C → platform_db
```

**왜 DB가 직접 매 요청을 세면 안 되나요?**

이게 가장 중요한 부분입니다. rate limit을 강제하려면 *모든 요청마다* 카운터를 +1 해야 합니다. 이걸 DB로 하면:

```sql
-- ❌ 절대 이렇게 하면 안 됨 — 매 요청마다 counter UPDATE
UPDATE rate_counter
SET count = count + 1
WHERE api_key_pk = 123 AND window = '2026-05-30T10:23:00';
-- 같은 키로 초당 수백 요청 → 같은 행에 UPDATE 폭주
```

문제는 **핫스팟(hotspot)과 락 경합(lock contention)**입니다:

```
같은 api_key가 초당 500요청을 보내면...

  요청1 → UPDATE rate_counter (행 락 획득)
  요청2 → UPDATE ... 같은 행 → 락 대기 ⏳
  요청3 → UPDATE ... 같은 행 → 락 대기 ⏳⏳
  ...                                  └─ 단일 행에 쓰기 직렬화 → DB 병목
```

rate limit은 본질적으로 "초당 수천 번 카운트"하는 작업이라 **고속·휘발성 카운터**가 맞습니다. 이건 Redis(인메모리, 원자적 INCR)나 Gateway 내장 메커니즘이 잘하는 일이고, 디스크 기반 트랜잭션 DB(MySQL)는 못하는 일입니다. 그래서 *강제는 엣지로 밀고*, platform_db는 *정책(어느 tier인지)만 보관*합니다.

```
✅ platform_db:  정책 보관소  ("이 키는 standard 등급")  — 가끔 읽음
✅ Gateway/Redis: 강제 실행기  ("standard는 초당 10, 11번째는 429") — 매 요청 카운트
```

> 💡 **한 줄 요약**: platform_db는 `api_key.rate_limit_tier`로 *정책(등급)만 보관*하고, 매 요청 카운트와 차단은 Gateway/WAF(+Redis)가 합니다. DB가 매 요청 UPDATE하면 단일 행 핫스팟·락 경합으로 병목이 되기 때문입니다.

---

## Q4. rate limit을 실제로 어떻게 세나요? token bucket, sliding window가 뭔가요?

이건 **Gateway·Redis가 구현하는 알고리즘**이라 platform_db 코드에는 없습니다. 하지만 "왜 DB가 아니라 엣지인지"를 이해하려면 개념은 알아두는 게 좋습니다. 세 가지가 대표적입니다.

**① Fixed Window(고정 윈도우)** — 가장 단순

```
"1초 단위로 칸을 나눠서, 한 칸에 10개까지"

10:23:00.000 ~ 10:23:00.999  →  [요청 카운트] 10까지 OK, 11번째 429
10:23:01.000 ~ 10:23:01.999  →  카운트 리셋 → 다시 0부터
```

단순하지만 경계 문제가 있습니다: `10:23:00.900`에 10개 + `10:23:01.000`에 10개 = 0.2초에 20개가 통과할 수 있습니다(경계 burst).

**② Sliding Window(슬라이딩 윈도우)** — 경계 문제 보완

```
"지금 시각에서 뒤로 1초 구간을 매번 새로 계산"

10:23:01.050 시점 → 10:23:00.050 ~ 10:23:01.050 구간의 요청 수를 셈
                    └─ 창이 시각을 따라 미끄러짐(slide) → 경계 burst 없음
```

더 정확하지만 계산이 무겁습니다. 보통 Redis sorted set이나 근사 알고리즘으로 구현합니다.

**③ Token Bucket(토큰 버킷)** — 가장 널리 쓰임, burst 허용

```
양동이에 토큰이 들어있다. 요청 1개 = 토큰 1개 소비.
토큰은 일정 속도로 다시 채워진다(예: 초당 10개 리필).

  [🪙🪙🪙🪙🪙] 버킷(최대 10토큰)
   요청 → 토큰 1개 꺼내 씀. 버킷 비면 → 429.
   초당 10개씩 리필 → 평소엔 여유, 잠깐 몰리면 버킷에 쌓인 만큼 burst 허용
```

토큰 버킷은 "평소엔 여유롭게, 갑자기 몰리면 쌓인 토큰만큼은 허용"이라 사용자 경험이 좋아서 가장 많이 씁니다.

핵심은 **이 세 알고리즘 전부 매 요청마다 카운터/토큰을 갱신**한다는 점입니다. Q3에서 본 핫스팟 문제가 그대로 적용되므로, 이걸 MySQL로 구현하면 안 되고 Gateway/Redis가 맡습니다.

```
우리(platform_db)의 역할:  rate_limit_tier = 'standard'  (어떤 정책인지)
Gateway/Redis의 역할:      token bucket 으로 standard=초당10 강제  (어떻게 세는지)
```

> 💡 **한 줄 요약**: fixed window/sliding window/token bucket은 *빈도를 세는 방법*이고, 셋 다 매 요청 갱신이라 Gateway·Redis가 구현합니다. platform_db는 알고리즘을 모르고, 어떤 tier인지만 압니다.

---

## Q5. rate limit에 걸리면 어떤 응답이 오나요? quota 초과와 응답이 다른가요?

rate limit 초과는 표준적으로 **429 Too Many Requests**입니다.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 1
Content-Type: application/json

{ "error": "rate_limit_exceeded", "message": "요청이 너무 잦습니다. 잠시 후 다시 시도하세요." }
```

`Retry-After` 헤더로 "몇 초 뒤에 다시 시도하라"를 알려주는 게 좋은 관행입니다. rate limit은 *기다리면 풀리는* 제한이기 때문입니다.

quota 초과와 의미가 다르므로 상태 코드도 다르게 쓰는 게 보통입니다:

| 상황 | 의미 | 상태 코드 | 회복 방법 |
|---|---|---|---|
| rate limit 초과 | "너무 빠름" | **429** Too Many Requests | 잠깐 기다리면 재개 |
| quota 초과 | "한도 소진" | **403** Forbidden 또는 **422** | 플랜 업그레이드 / 리셋 대기 |
| 권한 없음 | "허가 안 됨" | **403** Forbidden | 권한 부여 필요 |

핵심: **429는 "지금은 안 되지만 곧 됨"**, **403/422 quota는 "이번 주기에는 더 못 씀"**이라는 다른 메시지를 클라이언트에게 줍니다. 클라이언트는 429를 받으면 `Retry-After`만큼 기다렸다가 재시도(backoff)하면 되지만, quota 초과는 재시도해봐야 소용없습니다.

> 💡 **한 줄 요약**: rate limit 초과는 `Retry-After`를 동반한 **429**(곧 다시 됨), quota 초과는 **403/422**(이번 주기엔 끝)로 구분해 응답합니다.

---

## Q6. OTP 브루트포스 방어도 rate limit인가요? P10-04는 어떻게 동작하나요?

네. "OTP를 계속 찍어서 맞추려는 공격(브루트포스)"을 막는 것도 rate limit의 한 종류입니다 — *인증 시도 빈도*를 제한하는 것이죠. 이게 [[bdd-scenarios]] P10-04입니다.

```gherkin
Scenario: OTP 5회 연속 실패 → 차단
  Given 동일 phone_e164로 OTP 검증이 5회 연속 실패했다
  When 6번째 OTP 시도
  Then 429 Too Many Requests ("잠시 후 다시 시도하세요")
  And OTP 무효화 (재발송 필요)
  And audit_log에 (action='phone_otp_brute_force', result='DENY') INSERT
  # Note: rate-limit은 Gateway 레이어에서 강제 (TEN-9, api_key.rate_limit_tier)
```

여기서도 원칙은 같습니다:

```
빈도 카운트("같은 번호로 몇 번 실패했나") → Gateway 레이어에서 강제 (TEN-9)
                                            ❌ platform_db가 매 시도마다 카운트하지 않음
차단 사실의 흔적(audit_log INSERT)         → platform_db에 남김 (감사/포렌식용)
```

즉 *빈도를 세서 막는 행위*는 엣지가 하고, *"막았다"는 사실의 감사 기록*은 platform_db의 `audit_log`에 append-only로 남깁니다. 이 분리가 일관됩니다 — platform_db는 "빠르게 세는 카운터"가 아니라 "신뢰할 수 있는 영구 기록·정책 보관소"입니다.

이건 [[fail-closed]] 원칙과도 맞닿아 있습니다: 인증 시도가 의심스러우면(브루트포스 패턴) **막는 쪽으로(429)** 기웁니다. 정상 사용자는 OTP 재발송으로 회복할 수 있으니, 약간의 불편을 감수하고 공격을 차단하는 게 안전한 기본값입니다.

> 💡 **한 줄 요약**: OTP 브루트포스 방어(P10-04)는 *인증 시도 빈도*에 대한 rate limit입니다. 빈도 카운트·429 차단은 Gateway가, "막았다"는 감사 기록은 platform_db의 `audit_log`가 담당합니다.

---

## 용어 정리

| 용어 | 뜻 |
|---|---|
| **quota(한도)** | *누적 총량* 제한. "월 업로드 500개", "멤버 50명". `org_entitlement.feature_limits`에 보관(SSOT). |
| **rate limit(속도 제한)** | *단위 시간당 빈도* 제한. "초당 10요청". `api_key.rate_limit_tier`로 정책만 보관, 강제는 Gateway. |
| **throttling(스로틀링)** | rate limit을 적용해 요청을 *늦추거나 거절*하는 행위. "스로틀 걸렸다" = 빈도 제한에 걸렸다. |
| **token bucket** | 가장 널리 쓰이는 rate limit 알고리즘. 토큰을 일정 속도로 리필, 요청마다 1개 소비, 비면 거절. burst 허용. |
| **fixed window** | 고정 시간 칸마다 카운트 리셋. 단순하지만 경계 burst 문제. |
| **sliding window** | 현재 시각 기준으로 창을 미끄러뜨려 카운트. 경계 문제 보완, 계산 무거움. |
| **429 Too Many Requests** | rate limit 초과 시 HTTP 상태 코드. `Retry-After`로 재시도 시점 안내. "곧 다시 됨". |
| **rate_limit_tier** | api_key의 rate limit 등급(예: free/standard/premium). platform_db는 이 *분류값만* 보관(SEC-7, P1). |
| **Gateway / WAF** | 앱 앞단의 엣지 레이어. 실제 rate limit 카운트·차단·IP 필터링을 수행. |
| **핫스팟(hotspot)** | 같은 행/키에 쓰기가 몰려 락 경합·병목이 생기는 지점. rate 카운터를 DB로 하면 발생. |
| **usage_snapshot** | platform_db의 사용량 *집계 스냅샷*. "498/500 썼다"는 가시성 제공(O4). 실시간 카운터 아님. |

---

## 테스트 방법

rate limit은 platform_db 단위 테스트가 아니라 *Gateway 통합 테스트*로 검증한다는 점이 중요합니다. platform_db 쪽에서 테스트할 것은 "정책(tier) 보관"과 "차단 감사 기록"뿐입니다.

**① rate limit 초과 시 429 (Gateway 통합 테스트 / 모킹)**

```typescript
// Gateway 레이어 통합 테스트 — 빠르게 11번 호출하면 11번째는 429
it('standard tier: 초당 11번째 요청은 429', async () => {
  const key = await issueApiKey({ rateLimitTier: 'standard' }); // 초당 10
  const results = await Promise.all(
    Array.from({ length: 11 }, () => callApi(key))
  );
  const tooMany = results.filter((r) => r.status === 429);
  expect(tooMany.length).toBeGreaterThanOrEqual(1); // 적어도 1개는 429
  expect(results.find((r) => r.status === 429)?.headers['retry-after']).toBeDefined();
});
```

```typescript
// platform_db 단위 테스트에서는 DB가 카운트하지 않음을 보장
// → rate 카운터 테이블이 없어야 정상 (있으면 핫스팟 설계 위반)
it('platform_db에는 매 요청 카운트 테이블이 없다', async () => {
  const tables = await listTables();
  expect(tables).not.toContain('rate_counter'); // 휘발성 카운터는 Redis/Gateway
});
```

**② quota 초과 시 feature_limit 평가로 차단 (앱 레이어)**

```typescript
it('일 업로드 한도 초과 시 차단 (quota, ABAC-3)', async () => {
  // org_entitlement.feature_limits = {"daily_uploads": 10}
  await seedFeatureLimits(orgPk, { daily_uploads: 10 });
  await seedTodayUploads(orgPk, 10); // 이미 10개 사용

  await expect(uploadVideo(orgPk)).rejects.toThrow(QuotaExceededException);
  // quota는 429가 아니라 403/422 — "한도 소진"
});
```

**③ OTP 브루트포스 429 + 감사 기록 (P10-04)**

```typescript
it('OTP 5회 실패 후 6번째는 429 + audit_log DENY', async () => {
  for (let i = 0; i < 5; i++) {
    await verifyOtp(phone, 'wrong-code'); // 5회 실패
  }
  const sixth = await verifyOtp(phone, 'wrong-code');
  expect(sixth.status).toBe(429);

  // platform_db에 차단 사실이 append-only로 기록되었는가
  const log = await getLatestAuditLog(phone);
  expect(log.action).toBe('phone_otp_brute_force');
  expect(log.result).toBe('DENY');
});
```

**체크리스트**

```
□ rate limit 강제 로직이 platform_db 안에 있지 않은가? (있으면 핫스팟 — Gateway로 이동)
□ api_key.rate_limit_tier는 "정책 분류값"만 담고, 카운트는 담지 않는가?
□ rate limit 초과 응답이 429 + Retry-After 인가? (quota는 403/422)
□ quota 평가는 org_entitlement.feature_limits(SSOT)를 읽는가? (product_feature 직접 조회 금지 — [[feature-limits]])
□ OTP 브루트포스 차단(429) 시 audit_log에 DENY가 append 되는가?
□ 실시간 사용량 카운트는 서비스측/Redis, platform_db는 usage_snapshot 집계만 갖는가?
```

---

## 마치며

이 문서가 막히기 쉬운 한 가지를 풀어준다면, 그건 **"왜 quota는 우리 DB에 있는데 rate limit은 없지?"**입니다. 답은 두 단어입니다 — **개념**과 **빈도**.

- **개념이 다르다**: quota는 *누적 총량*(월 500개), rate limit은 *단위 시간당 빈도*(초당 10개). 다 쓰면 끝인 것과, 기다리면 풀리는 것의 차이.
- **빈도가 다르다**: quota 평가는 가끔(업로드할 때마다) 일어나지만, rate limit 카운트는 *모든 요청마다* 일어납니다. 후자를 디스크 DB로 하면 단일 행 핫스팟·락 경합으로 무너집니다.

그래서 `platform_db`의 역할은 명확합니다:

```
platform_db는 "정책 보관소 + 신뢰할 수 있는 영구 기록"이다.
  - quota 정책:    org_entitlement.feature_limits (SSOT)
  - rate 정책:     api_key.rate_limit_tier (어떤 등급인지)
  - 차단의 흔적:    audit_log (DENY 기록)

platform_db는 "고속 휘발성 카운터"가 아니다.
  - 매 요청 빈도 카운트 → Gateway/WAF + Redis
  - 실시간 사용량 카운트 → 서비스측 (platform은 usage_snapshot 집계만)
```

새 기능에서 "요청을 제한해야 한다"는 요구가 나오면, 먼저 자문하세요: **"이게 누적 총량(quota)인가, 빈도(rate limit)인가?"** 그리고 빈도라면 **"이걸 DB로 세려는 건 아닌가?"**. 이 두 질문이 TEN-9를 코드로 옮기는 나침반입니다.

---

## 연결된 개념

- [[feature-limits]] — org quota(`feature_limits`)의 우선순위·SSOT. 이 문서의 quota 쪽 심화.
- [[machine-identity-apikey|머신 신원 & api_key]] — `rate_limit_tier`가 붙는 api_key 테이블의 전체 맥락.
- [[gate-b-entitlement|Gate B & 엔타이틀먼트]] — quota의 근거인 `org_entitlement`을 읽는 게이트.
- [[fail-closed]] — 의심스러운 빈도(브루트포스)는 막는 쪽으로 기우는 기본값.
- [[multitenancy-rls]] — "강제는 어느 레이어에서?"라는 같은 질문의 멀티테넌시 버전.
> 소스 문서
- [[requirements]] — TEN-9(rate-limit 정책 위치), ABAC-3(feature_limit 평가), SEC-7(rate_limit_tier 보강), USAGE-1
- [[schema-reference]] — §J api_key, §D.12 org_entitlement.feature_limits
- [[architecture]] — §3.1 TEN-9 결정, O4 usage_snapshot
- [[bdd-scenarios]] — P10-04 OTP 브루트포스 429
