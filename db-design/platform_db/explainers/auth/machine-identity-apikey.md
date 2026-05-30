---
difficulty: 고급
tags:
  - platform-db
  - explainer
  - p1
  - security
  - api-key
  - machine-identity
  - authn
  - abac
aliases:
  - 머신 신원
  - machine identity
  - api_key
  - service account
  - B2B 신원
---

# 머신·B2B 신원 (api_key) 설명 — 사람만 사용자가 아니다

> **대상**: 인증/인가를 깊게 안 다뤄본 개발자 (공부용 · 사수 모드)
> **연관 문서**: [[schema-reference|schema-reference.md §J·§B·§H.3·§H.4]] · [[requirements|requirements.md P6 · AUTHN-5·RBAC-4·SEC-5·SEC-7·ABAC-6]] · [[e2e-journeys|e2e-journeys.md C-07]]

"우리 시스템의 사용자는 사람이다." — 대부분의 개발자가 무의식적으로 가진 전제입니다. 그런데 agent가 우리 API를 호출하거나, B2B 파트너 회사의 서버가 자동으로 강의 데이터를 긁어가야 한다면? 거기엔 로그인 화면 앞에 앉은 사람이 없습니다. **머신도 신원이 필요합니다.** 이 문서는 `platform_db`가 머신 신원을 어떻게 모델링하고, 핵심 원칙인 "머신도 사람과 똑같은 게이트를 통과한다"를 어떻게 강제하는지 설명합니다.

---

## Q1. 머신 신원(machine identity)이 뭔가요? 사람 계정이랑 뭐가 다른가요?

**머신 신원**은 사람이 아니라 *프로그램*(서버·스크립트·agent)이 우리 시스템에 대해 갖는 신원입니다. "서비스 계정(service account)"이라고도 부릅니다.

비유를 들면 이렇습니다. 사람이 건물에 들어올 땐 **지문**을 찍습니다(우리 시스템의 Firebase 로그인). 그런데 매일 새벽 자재를 배달하는 협력업체 트럭 기사는 지문 등록이 안 돼 있습니다. 대신 관리실에서 발급한 **출입 카드(열쇠)**를 받습니다. 사람은 지문, 머신은 발급된 카드 — **인증 수단은 다르지만, 둘 다 똑같은 출입 게이트를 통과**합니다. 카드가 있다고 직원 전용 구역에 그냥 들어갈 수 있는 건 아니죠.

사람과 머신의 차이는 *인증 수단(credential)*에만 있습니다:

```
사람(HUMAN)   : 지문(Firebase ID 토큰)으로 "나 김지영이야" 증명
머신(SERVICE) : 발급된 열쇠(api_key)로 "나 등록된 agent야" 증명
                    │
                    └─ 그 다음부터는 둘 다 완전히 동일한 경로
```

`platform_db`는 사람과 머신을 **같은 테이블**(`identity_user`)에 담습니다. `type` 컬럼으로만 구분합니다(§D.1):

```sql
-- identity_user.type 으로 사람/머신 구분 (테이블은 하나)
type ENUM('HUMAN','SERVICE','SYSTEM') NOT NULL DEFAULT 'HUMAN'
```

머신 신원은 `type='SERVICE'`이고, 테넌트 권위는 `membership.platform_role='SERVICE'`로 표현합니다(§D.4). 즉 머신도 *어느 org에 소속된 멤버*입니다 — 떠다니는 슈퍼유저가 아닙니다.

```
identity_user(type='SERVICE')  ← 머신의 신원
        │
   membership(platform_role='SERVICE', org_pk=한울학원)  ← 어느 org 소속인지
        │
   api_key(user_pk=이 머신, org_pk=한울학원, secret_hash=...)  ← 인증 수단(열쇠)
```

> 💡 **한 줄 요약**: 머신 신원은 `identity_user.type='SERVICE'`인 사용자입니다. 사람이 지문(Firebase)으로 인증하듯, 머신은 발급된 열쇠(api_key)로 인증하지만 — 그 뒤는 사람과 똑같이 org에 소속된 멤버로 취급됩니다.

---

## Q2. "머신도 사람과 똑같이 3-gate를 통과한다"는 게 무슨 뜻인가요? 왜 중요한가요?

이게 이 설계의 **핵심 원칙**입니다. 우리 시스템의 모든 요청은 3개의 게이트를 통과합니다(§E):

```
Gate A — 소속 : 너 이 org의 멤버 맞아?       (membership)
Gate B — 결제 : 이 org가 이 서비스 구독 중이야?  (org_entitlement)
Gate C — 정책 : 너 이 행동 해도 되는 역할이야?    (CASL ability)
```

핵심은: **머신용 우회 경로가 없다**는 것입니다. api_key로 인증한 머신도 위 3-gate를 *정확히 같은 코드*로 통과합니다(§J 설계 포인트: "3-gate 동일 적용").

왜 이게 중요할까요? 흔한 실수는 머신용 "뒷문"을 만드는 것입니다:

```typescript
// ❌ 위험한 안티패턴 — 머신은 게이트를 건너뛴다
if (req.apiKey) {
  // "api_key가 있으면 신뢰할 수 있는 시스템이니까 통과시키자"
  return next();   // ← Gate A/B/C 전부 우회! 구독 만료돼도 통과, 권한 없어도 통과
}

// ✅ 우리 설계 — 머신도 사람과 같은 게이트
// api_key를 검증해서 SERVICE 신원을 확정한 뒤,
// 그 신원으로 사람과 동일하게 Gate A → B → C 통과
const principal = await resolveApiKey(rawKey);   // 인증만 다름
await checkGateA(principal.userPk, orgPk);        // 소속 — 사람과 동일
await checkGateB(orgPk, service);                 // 결제 — 사람과 동일
const ability = buildAbility(principal);          // 정책 — 사람과 동일
if (!ability.can(action, resource)) throw ForbiddenException;
```

뒷문을 만들면 어떻게 될까요? 그 머신 키 하나가 탈취되면 구독·권한·소속을 전부 무시하고 모든 데이터에 접근하는 마스터키가 됩니다. "머신은 신뢰할 수 있다"는 가정이 단일 장애점이 됩니다. **api_key는 인증 수단일 뿐, 인가 면제권이 아닙니다.**

구체적으로 머신이 게이트에서 막히는 상황:

| 상황 | 사람 | 머신(api_key) | 결과 |
|---|---|---|---|
| org 멤버 아님 | Gate A 차단 | Gate A 동일 차단 | 403 |
| 구독 EXPIRED | Gate B 402 | Gate B **동일** 402 | 402 |
| 권한 없는 action | Gate C 차단 | Gate C 동일 차단 | 403 |

특히 **구독이 만료되면 api_key도 똑같이 402**라는 점이 중요합니다(C-07 시나리오 #4). "우리 시스템끼리 호출인데 결제랑 무슨 상관?"이 아니라, 머신도 *어느 org를 대신해* 호출하는 것이므로 그 org의 구독 상태에 묶입니다.

> 💡 **한 줄 요약**: 머신은 인증 수단(api_key)만 다를 뿐, Gate A(소속)·B(결제)·C(정책)를 사람과 *완전히 같은 코드*로 통과합니다. 머신 전용 우회 경로는 의도적으로 없습니다 — 그게 있으면 키 하나가 마스터키가 되기 때문입니다.

---

## Q3. B2B 클라이언트는 Firebase JWT 없이 어떻게 API를 호출하나요? HMAC을 쓰나요?

이게 핵심 실무 질문입니다. 사람은 브라우저·앱에서 Firebase로 로그인해 **JWT(ID 토큰)**를 받습니다. 그런데 B2B 파트너의 서버나 agent에는 로그인 화면 앞에 앉은 사람이 없습니다. **Firebase JWT를 받을 경로가 없습니다.**

답은: **머신은 Firebase JWT 대신 api_key를 인증 수단으로 씁니다.** 발급받은 키를 `Authorization` 헤더에 실어 보냅니다.

```http
# 사람 (Firebase JWT)
Authorization: Bearer eyJhbGciOi...        ← Firebase ID 토큰
                       (FirebaseJwtGuard가 검증)

# 머신 (api_key)
Authorization: Bearer ak_live_3f8a9b...     ← 발급된 api_key
                       (ApiKeyGuard가 검증)
```

진입 가드만 갈라집니다. 토큰 prefix(`ak_live_` 등)나 별도 헤더로 "이건 api_key 요청"임을 알아채면, FirebaseJwtGuard 대신 **ApiKeyGuard**가 키를 검증해 `type='SERVICE'` 신원을 확정합니다. **그 이후는 Q2에서 본 것처럼 사람과 완전히 같은 Gate A/B/C입니다.** 즉 "Firebase JWT 없이"의 답 = *인증(현관) 수단만 api_key로 바뀌고, 인가(내부)는 동일*.

```
사람:   요청 → FirebaseJwtGuard(JWT 검증) → Gate A → B → C → 처리
머신:   요청 → ApiKeyGuard(api_key 검증)   → Gate A → B → C → 처리
                  └─ 여기만 다름 ─┘          └──── 완전히 동일 ────┘
```

**그럼 HMAC을 쓰나요?** — 좋은 질문이고, 두 가지를 구분해야 합니다.

**(1) 우리 api_key 인증 = Bearer 토큰 방식 (HMAC 아님).** 머신은 발급받은 키 *그 자체*를 헤더에 실어 보내고, 서버는 [[secret-encryption|bcrypt]]로 해시 비교합니다(`secret_hash`). 키가 네트워크를 흐르지만 **TLS(HTTPS)가 전송 구간을 암호화**해 보호합니다. 이게 가장 흔한 B2B API 인증 방식입니다(Stripe·OpenAI의 API 키와 동일 계열).

**(2) HMAC은 우리 설계에서 *다른 곳*에 씁니다 — PG webhook 서명 검증.** PG(Toss 등)가 우리에게 결제 알림(webhook)을 보낼 때, 본문을 공유 비밀로 **HMAC-SHA256 서명**해 함께 보냅니다. 우리는 같은 비밀로 서명을 재계산해 일치하는지 확인합니다(`pg_webhook_event.signature_ok`, §F.4 · [[webhook-processing]]). 이건 *"우리가 키를 모르는 외부가 보낸 요청이 진짜인지·위변조 안 됐는지"*를 검증하는 **인바운드 webhook** 용도입니다 — *우리가 외부를 호출하는* B2B 인증과는 방향이 반대입니다.

두 방식의 차이를 알아두면 좋습니다:

| | Bearer api_key (우리 B2B 인증) | HMAC 요청 서명 (AWS SigV4 스타일) |
|---|---|---|
| 보내는 것 | 키 *자체*를 헤더에 | 키로 *요청을 서명한 값*만 (키는 안 보냄) |
| 전송 보호 | TLS가 담당 | 서명 자체가 위변조·도청 방지에 기여 |
| replay 방지 | 별도 없음(TLS 의존) | 타임스탬프·nonce를 서명에 포함 |
| 클라이언트 구현 | 단순 (헤더 하나) | 복잡 (요청마다 서명 계산) |
| 우리 채택 | ✅ **api_key 인증** | ❌ 미채택 (단, webhook *수신* 검증엔 HMAC 사용) |

우리가 Bearer api_key를 택한 이유: **단순함 + 충분함**. TLS로 전송을 보호하고, 유출 위험은 `allowed_ip_cidr`(Q6 Confused Deputy 방어)·즉시 revoke(Q7)·rotation(Q5)으로 보강합니다. HMAC 요청 서명은 더 강력하지만(키가 네트워크에 안 흐르고 replay를 막음) 클라이언트 구현이 복잡해, *고보안 B2B 정식화*가 필요할 때 고려하는 향후 옵션입니다([[requirements]] B2BAPI-1 범위 — OAuth Client Credentials 등과 함께).

> 💡 **한 줄 요약**: B2B는 Firebase JWT 대신 **api_key를 `Authorization: Bearer`로** 보내 인증합니다(진입 가드만 다르고 Gate A/B/C는 동일). 우리 api_key는 HMAC 서명이 아니라 *키 자체 + bcrypt 비교 + TLS* 방식이고, HMAC-SHA256은 별개로 *PG webhook 수신 검증*에 씁니다.

---

## Q4. api_key 테이블은 어떻게 생겼나요? 컬럼 하나하나 무슨 의미인가요?

§J의 DDL을 보며 컬럼의 *설계 의도*를 하나씩 봅시다:

```sql
CREATE TABLE api_key (
  pk               BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk           BIGINT UNSIGNED NOT NULL,     -- 어느 테넌트 소속 키인가 (격리 기준)
  user_pk          BIGINT UNSIGNED NOT NULL,     -- 소유자 identity_user(type='SERVICE')
  key_prefix       VARCHAR(10) NOT NULL,         -- 평문 prefix (예: 'ak_live_') — 식별/조회용
  secret_hash      VARCHAR(255) NOT NULL,        -- bcrypt hash. 평문은 발급 시 1회만 노출
  scopes           JSON NOT NULL,                -- ["lecture:read","membership:write"]
  allowed_ip_cidr  VARCHAR(50),                  -- 허용 IP 범위 (NIST 환경속성)
  allowed_services JSON,                         -- ["ACADEMY","MARKET"]
  rotated_at       TIMESTAMP,                    -- 마지막 rotation 시각
  last_used_at     TIMESTAMP,                    -- 최근 사용 (유휴 키 탐지)
  expires_at       TIMESTAMP,                    -- 만료 시각
  revoked_at       TIMESTAMP,                    -- 즉시 폐기 시각 (NULL이면 유효)
  revoked_reason   VARCHAR(255),                 -- 폐기 사유 (감사)
  created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
  INDEX idx_api_key_org (org_pk),
  INDEX idx_api_key_user (user_pk)
);
```

**`key_prefix` + `secret_hash` — 열쇠를 둘로 나누는 이유**

발급된 실제 키는 `ak_live_3f8a9b2c...` 같은 긴 문자열입니다. 우리는 이걸 둘로 나눠 다룹니다:

```
ak_live_3f8a9b2c1d4e5f6a7b8c9d0e...
└──┬───┘ └──────────────┬──────────────┘
 prefix              secret(비밀)
   │                     │
 평문 그대로 저장        bcrypt로 해시해서 저장 (원본은 버림)
 (이 키가 누구 건지     (평문은 발급 응답에 딱 1번만 보여주고
  로그·조회로 식별)      DB에는 절대 평문 저장 안 함 → §H.3)
```

- **`key_prefix`(평문)**: "이 요청이 어느 키냐"를 빠르게 찾는 검색 키이자, 로그에 찍어도 안전한 식별자입니다(비밀이 아님).
- **`secret_hash`(bcrypt)**: 진짜 비밀 부분. **평문은 DB에 절대 저장하지 않습니다.** 발급 응답에서 단 1회만 보여주고, 그 뒤로는 누구도(우리 DB 관리자조차) 원본을 복원할 수 없습니다. 자세한 해시 원리는 [[secret-encryption]] 참고.

이 "1회만 노출"이 왜 안전을 높일까요? DB가 통째로 유출돼도 공격자는 `secret_hash`(해시값)만 얻습니다. bcrypt는 단방향이라 해시에서 원본 키를 역산할 수 없습니다. 평문을 저장했다면 유출 = 모든 키 탈취였을 겁니다.

**`scopes`(JSON) — 키가 할 수 있는 일의 범위**

```json
["lecture:read", "membership:write"]
```

`scope`는 "이 키로 무엇까지 할 수 있나"를 좁힙니다(최소권한 원칙 — [[least-privilege-grant]]). 강의를 읽기만 하면 되는 통합이라면 `["lecture:read"]`만 발급합니다. 이 키가 유출돼도 *읽기*밖에 못 합니다. scope는 Gate C(정책) 평가의 입력이 됩니다.

**`allowed_ip_cidr` — 어디서 와야 하는가 (NIST 환경속성)**

```
allowed_ip_cidr = "203.0.113.0/24"
→ 이 IP 대역에서 온 요청만 이 키를 받아준다
```

CIDR은 IP 범위를 표기하는 방식입니다(`203.0.113.0/24` = `203.0.113.0` ~ `203.0.113.255`). 이건 NIST SP 800-162 ABAC의 **환경 속성(environment attribute)** — "주체가 누구냐"(신원)가 아니라 "*어떤 상황에서* 요청했냐"(IP·시간 등)로 판단하는 속성입니다([[requirements]] ABAC-6·SEC-2). 이게 **Confused Deputy 공격**을 막습니다(Q6).

**`allowed_services`(JSON) — 어느 서비스에 쓸 수 있나**

```json
["ACADEMY", "MARKET"]
```

이 키가 통할 서비스 목록입니다. academy용으로 발급한 키가 market API까지 건드리지 못하게 가둡니다.

**`rotated_at` / `revoked_at` — 키의 생애 관리**

- `rotated_at`: 마지막으로 갈아끼운 시각(Q5 rotation).
- `revoked_at`: 즉시 폐기 시각. **NULL이면 유효, 값이 있으면 죽은 키**입니다(Q7 revoke). `expires_at`(자연 만료)과 별개로, 사고 시 즉시 죽이는 비상 스위치입니다.

> 💡 **한 줄 요약**: api_key는 평문 prefix(식별용)와 bcrypt 해시된 secret(비밀)로 키를 둘로 나누고, scopes·allowed_ip_cidr·allowed_services로 "무엇을·어디서·어느 서비스에" 쓸 수 있는지를 좁히며, rotated_at·revoked_at으로 생애를 관리합니다.

---

## Q5. key rotation(키 교체)이 뭔가요? "overlap"은 또 뭐고요?

**key rotation**은 비밀번호를 주기적으로 바꾸듯, api_key를 **정기적으로 새 키로 갈아끼우는 것**입니다. 오래된 키일수록 어딘가에 흘렸을 위험이 누적되니까요. 우리 정책은 **90일 권장**입니다(§H.4 — 단 만료 강제는 안 함, `rotated_at`으로 추적).

문제는 *갈아끼우는 순간*입니다. 단순하게 하면 이렇게 됩니다:

```
❌ 무중단 불가능한 방식
12:00:00 구 키 삭제
12:00:00 신 키 발급
12:00:01 파트너 서버가 아직 구 키로 호출 → 401 → 서비스 중단!
         (파트너가 신 키로 설정 바꾸기 전까지 다운타임)
```

그래서 **overlap(겹침 구간)**이 필요합니다. 구 키와 신 키를 *동시에 유효*하게 두는 기간입니다:

```
✅ overlap 방식 (무중단)
        구 키(A) ████████████░░░░░░  (만료 예정)
        신 키(B)      ████████████████████
                      └──┬──┘
                    overlap 구간
              여기선 A·B 둘 다 200 OK
              파트너가 B로 설정을 바꿀 시간을 줌
              바꾼 뒤 A는 자연 만료(또는 revoke)
```

우리 설계가 이걸 어떻게 지원할까요? `api_key` 테이블은 **동일 org에 다중 키를 허용**합니다(§J 설계 포인트: "rotation 중 overlap 지원"). 키마다 독립된 row라서, A와 B가 나란히 ACTIVE 상태로 공존할 수 있습니다.

```sql
-- 같은 org·같은 SERVICE 사용자에 대해 두 키가 공존
-- 키 A: rotated_at은 있지만 expires_at이 아직 미래 → 유효
-- 키 B: 방금 발급 → 유효
-- → overlap 구간엔 둘 다 인증 통과
```

C-07 시나리오 #6이 이걸 검증합니다: 키B 발급 후에도 *키A가 만료 전까지* `GET /v1/lectures`에 200, 키B도 200 — 둘이 동시 동작.

> 💡 **한 줄 요약**: rotation은 오래된 키를 새 키로 교체하는 것이고(90일 권장), overlap은 그 교체 순간에 구·신 키를 잠시 함께 유효하게 둬서 무중단으로 전환하는 기법입니다. 우리는 org당 다중 키 허용으로 이를 지원합니다.

---

## Q6. allowed_ip_cidr가 막는 "Confused Deputy"는 어떤 공격인가요?

**Confused Deputy(혼란스러운 대리인)**는 *권한 있는 정직한 시스템을 속여서, 공격자가 자기 권한으로는 못 할 일을 대신 시키는* 공격입니다.

비유: 회사 경비원(deputy)이 "사장님 지시"라는 위조 메모를 받고 금고를 열어줍니다. 경비원은 금고 열 권한이 *있고*, 메모를 의심 없이 따랐을 뿐입니다. 공격자는 자기 권한이 아니라 *경비원의 권한*을 빌려 금고를 턴 거죠.

api_key 맥락에서:

```
공격자가 api_key를 탈취 (어딘가 로그·git에 노출된 키)
   │
   └─ 자기 노트북(IP 198.51.100.7)에서 그 키로 우리 API 호출
        │
        └─ 우리 서버: "유효한 키네, 권한대로 처리" ← Confused Deputy
```

`allowed_ip_cidr`가 이 사슬을 끊습니다. 정당한 머신은 *정해진 인프라*(예: 파트너의 서버 대역 `203.0.113.0/24`)에서만 호출합니다. 키를 훔쳐도 *그 IP 대역 밖*에서 쓰면 막힙니다:

```typescript
// Gate 진입 전 환경속성(IP) 검증 — 설계 의도
const clientIp = getRealClientIp(req);          // 게이트웨이가 박은 진짜 IP
if (apiKey.allowedIpCidr &&
    !ipInCidr(clientIp, apiKey.allowedIpCidr)) {
  throw new ForbiddenException('IP_NOT_ALLOWED'); // ★ 403, 게이트 도달 전 차단
}
```

이게 NIST의 *환경 속성*인 이유: 판단 근거가 "키가 유효하냐"(신원)가 아니라 "*어디서 왔냐*"(환경)이기 때문입니다. 신원이 멀쩡해도 환경이 틀리면 거부합니다 — 탈취된 정당한 키를 무력화하는 핵심 장치입니다.

C-07 시나리오 #3이 이를 검증합니다: 허용 대역 밖 IP에서 같은 키로 호출 → 403 `IP_NOT_ALLOWED`.

> 💡 **한 줄 요약**: Confused Deputy는 탈취한 정당한 키로 우리 시스템을 속여 권한을 빌려 쓰는 공격이고, `allowed_ip_cidr`는 "정해진 IP 대역에서 온 요청만 받음"으로 — 키를 훔쳐도 엉뚱한 곳에서는 못 쓰게 막습니다.

---

## Q7. 키가 유출됐어요. 어떻게 즉시 막나요? (revoke)

키가 git에 커밋됐거나 로그로 샜다면, 만료(`expires_at`)를 기다릴 여유가 없습니다. **즉시 무효화(revoke)**가 필요합니다.

우리 설계는 `revoked_at` 컬럼으로 이걸 합니다 — *행을 지우지 않고* 폐기 시각을 기록합니다:

```sql
-- revoke = 삭제가 아니라 폐기 마킹 (감사 추적 보존)
UPDATE api_key
SET revoked_at = NOW(), revoked_reason = 'leaked in public repo'
WHERE pk = ?;
```

왜 DELETE가 아니라 UPDATE일까요?

1. **감사 추적**: "언제·왜 폐기됐나"가 `revoked_reason`에 남습니다. 행을 지우면 사고 기록이 사라집니다([[delete-patterns]]).
2. **즉시성**: 인증 검증이 `revoked_at IS NULL`을 확인하므로, 다음 요청부터 바로 막힙니다.

```typescript
// 인증 시 revoke 확인 — 설계 의도
const apiKey = await findByPrefix(prefix);
if (apiKey.revokedAt !== null) {
  throw new UnauthorizedException('API_KEY_REVOKED'); // ★ 401, 즉시
}
if (apiKey.expiresAt && apiKey.expiresAt < new Date()) {
  throw new UnauthorizedException('API_KEY_EXPIRED');
}
// secret 검증(bcrypt.compare)은 그 다음 → [[secret-encryption]]
```

`revoked_at`(즉시·비상)과 `expires_at`(예정된 자연 만료)은 역할이 다릅니다:

```
expires_at : "이 키는 6개월 뒤 만료" — 계획된 수명
revoked_at : "지금 당장 죽여" — 비상 스위치 (유출·해고·계약종료)
```

C-07 시나리오 #5가 이를 검증합니다: `DELETE /v1/api-keys/{id}`로 revoke(204) → 같은 키로 호출 시 즉시 401 `API_KEY_REVOKED`.

> 💡 **한 줄 요약**: revoke는 행을 지우지 않고 `revoked_at`에 폐기 시각을 찍는 것입니다 — 감사 기록은 남기면서, 다음 요청부터 즉시 401로 막습니다(예정된 `expires_at`과 달리 비상 스위치).

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| **머신 신원(machine identity)** | 사람이 아닌 프로그램(서버·agent)이 시스템에 갖는 신원. `identity_user.type='SERVICE'` |
| **서비스 계정(service account)** | 머신 신원의 다른 이름. 자동화된 호출 주체 |
| **api_key** | 머신이 인증에 쓰는 발급된 비밀 키. prefix(평문)+secret(해시)로 구성 |
| **Bearer 토큰** | `Authorization: Bearer <키>` 형식으로 키 자체를 실어 보내는 인증 방식 |
| **key_prefix** | 키 앞부분 평문(`ak_live_`). 식별·조회용, 비밀 아님 |
| **secret_hash** | 키 비밀 부분의 bcrypt 해시. 평문은 발급 시 1회만 노출 |
| **bcrypt** | 단방향 비밀번호/키 해싱 함수. salt 내장, 역산 불가 ([[secret-encryption]]) |
| **HMAC** | 공유 비밀로 메시지를 서명/검증하는 방식. 우리는 PG webhook 수신 검증에 사용(api_key 인증엔 미사용) |
| **scope** | 키가 할 수 있는 일의 범위(`lecture:read`). 최소권한 적용 |
| **CIDR** | IP 범위 표기법(`203.0.113.0/24` = 256개 IP 대역) |
| **NIST ABAC 환경속성** | 신원이 아닌 *상황*(IP·시간 등)으로 판단하는 속성. NIST SP 800-162 |
| **Confused Deputy** | 권한 있는 정직한 시스템을 속여 그 권한을 빌려 쓰는 공격 |
| **key rotation** | 키를 정기적으로 새 키로 교체(우리: 90일 권장) |
| **overlap** | rotation 시 구·신 키를 잠시 함께 유효하게 둬 무중단 전환 |
| **revoke** | 키 즉시 폐기(`revoked_at` 마킹). 만료 대기 없는 비상 스위치 |
| **3-gate** | Gate A(소속)·B(결제)·C(정책). 사람·머신 공통 인가 경로 |

---

## 테스트 방법

api_key는 **머신용 우회 경로가 없다는 것**이 핵심이라, 테스트의 본질은 "**머신도 사람과 똑같이 게이트에서 막히는가**"를 확인하는 것입니다. ([[e2e-journeys]] C-07이 black-box 기준선입니다.)

**① 통합 테스트 (vitest + supertest) — 발급·인증·게이트 동등성**

```ts
// api-key.spec.ts
describe('api_key — 머신 신원 3-gate 동등', () => {
  let org, issued, machineKey;

  beforeEach(async () => {
    org = await createOrg('한울학원');
    await activateSubscription(org, 'ACADEMY');           // 구독 ACTIVE
    issued = await issueApiKey(org, {                     // OWNER가 발급
      scopes: ['lecture:read', 'lecture:write'],
      allowedIpCidr: '203.0.113.0/24',
    });
    machineKey = issued.secret;                           // 평문 — 응답 1회만
  });

  it('발급 응답엔 secret 평문이 1회 노출된다', () => {
    expect(issued.secret).toMatch(/^ak_live_/);           // 평문 키
    expect(issued.keyPrefix).toBe('ak_live_');
  });

  it('재조회하면 prefix만 보이고 secret 평문은 없다', async () => {
    const res = await request(app)
      .get(`/v1/api-keys/${issued.id}`)
      .set('Authorization', `Bearer ${ownerToken}`);
    expect(res.status).toBe(200);
    expect(res.body.keyPrefix).toBe('ak_live_');
    expect(res.body.secret).toBeUndefined();              // ★ 평문 절대 재노출 안 됨
  });

  it('허용 IP 안에서 api_key로 호출하면 3-gate 통과 200 (SERVICE 신원)', async () => {
    const res = await request(app)
      .get('/v1/lectures')
      .set('Authorization', `Bearer ${machineKey}`)
      .set('x-org-pk', org.pk)
      .set('X-Forwarded-For', '203.0.113.10');           // 허용 대역
    expect(res.status).toBe(200);                         // 사람과 동일 경로
  });

  it('허용 IP 밖에서 호출하면 환경속성에서 403', async () => {
    const res = await request(app)
      .get('/v1/lectures')
      .set('Authorization', `Bearer ${machineKey}`)
      .set('x-org-pk', org.pk)
      .set('X-Forwarded-For', '198.51.100.7');           // 대역 밖
    expect(res.status).toBe(403);
    expect(res.body.code).toBe('IP_NOT_ALLOWED');
  });

  it('구독이 만료되면 api_key도 Gate B 동일 적용으로 402', async () => {
    await expireSubscription(org, 'ACADEMY');             // EXPIRED 전이
    const res = await request(app)
      .post('/v1/lectures')
      .set('Authorization', `Bearer ${machineKey}`)
      .set('x-org-pk', org.pk)
      .set('X-Forwarded-For', '203.0.113.10')
      .send({ title: '머신이 만든 강의' });
    expect(res.status).toBe(402);                         // ★ 사람과 동일한 결제 게이트
    expect(res.body.code).toBe('ENTITLEMENT_REQUIRED');
  });

  it('revoke 후 같은 키로 호출하면 즉시 401', async () => {
    await request(app).delete(`/v1/api-keys/${issued.id}`)
      .set('Authorization', `Bearer ${ownerToken}`);     // 204
    const res = await request(app)
      .get('/v1/lectures')
      .set('Authorization', `Bearer ${machineKey}`)
      .set('x-org-pk', org.pk)
      .set('X-Forwarded-For', '203.0.113.10');
    expect(res.status).toBe(401);
    expect(res.body.code).toBe('API_KEY_REVOKED');
  });

  it('rotation 중에는 구 키·신 키가 둘 다 200 (overlap)', async () => {
    const keyB = await issueApiKey(org, {                 // 신 키 발급
      scopes: ['lecture:read'], allowedIpCidr: '203.0.113.0/24',
    });
    // 키A(기존)는 아직 만료 전 — 둘 다 유효해야 함
    for (const key of [machineKey, keyB.secret]) {
      const res = await request(app)
        .get('/v1/lectures')
        .set('Authorization', `Bearer ${key}`)
        .set('x-org-pk', org.pk)
        .set('X-Forwarded-For', '203.0.113.10');
      expect(res.status).toBe(200);                       // ★ A·B 동시 동작
    }
  });
});
```

**② e2e 블랙박스 — 사용자 여정으로 검증**

[[e2e-journeys|e2e 여정]]의 **C-07**이 위 흐름을 API 경계에서만(DB row 단언 없이) 검증합니다: 발급→3-gate 통과(200)→IP 차단(403)→구독 만료(402)→revoke(401)→rotation overlap(200).

**③ 무엇을 단언하나 (체크리스트)**

```text
□ 발급 응답에 secret 평문 1회 노출 / 재조회 시 prefix만(secret undefined)
□ 허용 IP 안 + api_key → 200 (사람과 동일한 3-gate 통과)
□ 허용 IP 밖 → 403 IP_NOT_ALLOWED (Confused Deputy 방어)
□ 구독 EXPIRED → 402 ENTITLEMENT_REQUIRED (Gate B는 머신에도 동일)
□ revoke 후 → 즉시 401 API_KEY_REVOKED (만료 대기 없음)
□ rotation overlap 구간 → 구 키·신 키 둘 다 200
□ ★ 어떤 경우에도 "api_key면 게이트 우회" 경로가 없는지 (200이 게이트 통과 결과인지)
```

> 💡 **테스트 한 줄 요약**: "머신이 사람과 *같은* 게이트에서 *같은* 결과로 막히는가"를 핵심으로 단언하세요 — IP는 403, 구독 만료는 402, revoke는 401. 머신용 통과 뒷문이 없다는 게 진짜 검증 대상입니다.

---

## 마치며

머신 신원 설계는 결국 하나의 규율로 요약됩니다: **"머신이라고 봐주지 않는다."**

사람과 머신의 차이는 *현관에서 신분을 증명하는 방법*(지문/Firebase JWT vs 발급된 열쇠/api_key)뿐이고, 그 안에서는 똑같이 org에 소속된 멤버로서 Gate A·B·C를 통과합니다. 이 동등성이 깨지는 순간 — 즉 "api_key면 통과" 같은 뒷문이 생기는 순간 — 그 키 하나가 구독·권한·격리를 전부 무시하는 마스터키가 됩니다.

그래서 api_key 설계의 모든 컬럼은 *권한을 넓히는 게 아니라 좁히는* 방향입니다: `scopes`(할 수 있는 일을 좁힘), `allowed_ip_cidr`(올 수 있는 곳을 좁힘), `allowed_services`(쓸 수 있는 서비스를 좁힘), `revoked_at`(살아 있는 시간을 좁힘). 새 머신 통합을 설계할 때는 "이 키에 무엇을 *줄까*"가 아니라 "무엇만 주면 *충분*한가(최소권한)"를 먼저 물으세요.

마지막으로 — 이 전체가 **아직 미구현(설계 확정)**임을 기억하세요. 두 번째 서비스가 agent라면, P6(머신 신원)는 academy 잔여 기능보다 먼저 와야 하는 1급 후보입니다([[requirements]] §2 우선순위 비판).

---

## 연결된 개념

- [[secret-encryption|비밀·암호화 경계]] — secret_hash(bcrypt)가 어떻게 1회만 노출되고 역산 불가한지
- [[gate-abc-flow|Gate A/B/C 전체 흐름]] — 머신이 통과하는 바로 그 3-gate
- [[webhook-processing|PG 웹훅 처리]] — HMAC-SHA256 서명 검증이 쓰이는 인바운드 경로
- [[role-capability|role 2단 분리 & capability]] — `platform_role='SERVICE'`와 권한 모델
- [[least-privilege-grant|최소권한 grant]] — scopes·allowed_services가 권한을 좁히는 원리
- [[bola-object-authz|BOLA 객체 수준 인가]] — api_key도 org_pk 격리에 묶임
- [[multitenancy-rls|멀티테넌시 & RLS]] — 머신도 org_pk로 테넌트 격리됨
- [[idempotency-key|멱등성 키]] — 머신의 자동 재시도와 안전성
> 소스 문서
- [[schema-reference]] — §J api_key DDL·설계 포인트, §B 식별자(API Key 행), §H.3 암호화 경계, §H.4 rotation cadence, §F.4 webhook 서명, §E 3-gate
- [[requirements]] — P6 머신 신원, AUTHN-5·RBAC-4·SEC-5·SEC-7·ABAC-6·SEC-2, B2BAPI-1(B2B 정식화)
- [[e2e-journeys]] — C-07 api_key B2B 통합 (발급→3-gate→IP→구독→revoke→rotation)
