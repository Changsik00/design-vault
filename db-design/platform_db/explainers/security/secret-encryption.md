---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p0
  - security
  - encryption
  - kms
  - bcrypt
  - pii
aliases:
  - 비밀 관리
  - 암호화 경계
  - secret encryption
  - KMS
  - envelope encryption
---

# 비밀·암호화 경계 (KMS·bcrypt) 설명 — 무엇을 어디서 어떻게 보호하나

> **대상**: 암호화를 깊게 안 다뤄본 개발자 (공부용 · 사수 모드)
> **연관 문서**: [[schema-reference|schema-reference.md §H.3·§H.4·§J]] · [[requirements|requirements.md SEC-4·SEC-6·SEC-8·OPS-4]]

"비밀번호는 암호화해서 저장하면 되는 거 아냐?" — 흔하지만 **틀린** 직관입니다. 비밀번호는 사실 *암호화*하면 안 되고 *해시*해야 합니다. 반대로 OAuth refresh_token은 해시하면 안 되고 *암호화*해야 합니다. 둘은 정반대 도구이고, 무엇을 언제 쓰는지 헷갈리면 보안이 무너집니다. 이 문서는 `platform_db`의 **암호화 경계**(§H.3) — 어떤 데이터를 어디서 누가 어떻게 보호하는지 — 를 그 둘의 차이부터 설명합니다.

---

## Q1. 해시(hashing)와 암호화(encryption)는 뭐가 다른가요? 왜 자꾸 구분하라고 하나요?

이 둘을 구분하는 게 비밀 관리의 출발점입니다. 핵심 차이는 **되돌릴 수 있느냐(복호화)**입니다.

**해시 = 분쇄. 한번 갈면 못 되돌린다 (단방향)**

비유: 종이를 파쇄기에 넣으면 종잇조각이 됩니다. 같은 종이는 항상 같은 조각으로 갈리지만, **조각에서 원본 종이를 복원할 수는 없습니다.**

```
"mypassword123"  ──bcrypt──▶  "$2b$12$ Kx7...(60자)"
                                       │
       같은 입력은 항상(salt 고정 시) 같은 출력
       하지만 출력 → 입력 복원은 불가능 (단방향)
```

비밀번호·api_key처럼 *원본을 알 필요가 없는* 비밀에 씁니다. 우리는 사용자 비밀번호 원본을 알 필요가 없습니다 — 로그인 시 "사용자가 입력한 값을 *같은 방식으로 갈아서*, 저장된 조각과 일치하는지"만 보면 됩니다.

```ts
// 검증 = "다시 갈아서 비교", 복원이 아님
bcrypt.compare("mypassword123", storedHash) // → true / false
```

**암호화 = 금고. 열쇠가 있으면 되돌릴 수 있다 (양방향)**

비유: 문서를 금고에 넣고 잠급니다. 열쇠가 있으면 다시 꺼내(복호화) 원본을 읽을 수 있습니다.

```
"refresh_token_xyz"  ──암호화(키)──▶  "암호문 8f3a..."  ──복호화(키)──▶  "refresh_token_xyz"
                                                                              │
                                              열쇠(key)가 있으면 원본 복원 가능
```

OAuth refresh_token처럼 *나중에 원본을 다시 써야 하는* 비밀에 씁니다. YouTube에 다시 API 호출하려면 refresh_token 원본이 필요하므로, 해시하면 안 됩니다(해시하면 영영 못 씀).

**한 표로 정리:**

| | 해시(hashing) | 암호화(encryption) |
|---|---|---|
| 방향 | 단방향(복원 불가) | 양방향(복호화 가능) |
| 비유 | 분쇄기 | 금고 |
| 원본 복원 | ❌ 불가 | ✅ 열쇠로 가능 |
| 검증 방법 | 다시 갈아서 비교 | 복호화해서 사용 |
| 도구 | **bcrypt** | **AWS KMS** |
| 우리 용례 | 비밀번호·api_key secret | OAuth refresh_token |

> 💡 **한 줄 요약**: 해시는 *못 되돌리는 분쇄*(비밀번호·키처럼 원본을 알 필요 없는 것), 암호화는 *열쇠로 여는 금고*(refresh_token처럼 다시 써야 하는 것)입니다. "나중에 원본이 필요한가?"가 둘을 가르는 질문입니다.

---

## Q2. 우리 시스템에서 무엇을 어디서 보호하나요? (암호화 경계)

§H.3의 "암호화 경계" 표가 우리 설계의 핵심입니다. 데이터마다 *처리 방식*과 *책임 주체*가 다릅니다:

| 데이터 | 처리 | 누가 | 우리가 저장? |
|---|---|---|---|
| 비밀번호 | bcrypt 해시 | **Firebase Auth** | ❌ 저장 안 함 |
| OAuth refresh_token | AWS KMS envelope encryption | 우리 (KMS 위임) | ✅ 암호문으로 |
| api_key secret | bcrypt 해시 | 우리 | ✅ 해시로만 (평문은 발급 시 1회) |
| IP 주소 | VARBINARY(16) raw | 우리 | ✅ 원본 (PIPA 감사) |
| 결제 카드 정보 | PCI-DSS | **PG사(Toss 등)** | ❌ 저장 안 함 |

여기서 가장 중요한 통찰: **"우리가 저장하지 않는다"도 하나의 보안 전략**입니다.

```
비밀번호  → Firebase가 bcrypt로 보관. 우리 DB엔 firebase_uid(조회 키)만.
카드정보  → PG가 PCI-DSS 인증 환경에 보관. 우리 DB엔 pg_payment_id(참조)만.
```

가장 위험한 데이터(비밀번호·카드번호)는 *아예 우리 손에 두지 않습니다*. 우리가 가지지 않으면 유출될 수도 없습니다. 이게 "secret 관리: KMS, 평문 로그 금지"([[requirements]] SEC-4)의 가장 강한 형태 — *보유하지 않음*입니다.

각각을 왜 그 방식으로 처리하는지:

- **비밀번호 → Firebase(bcrypt)**: 인증은 Firebase가 SSOT([[requirements]] AUTHN-1). 우리는 `identity_user.firebase_uid`로 *연결*만 합니다 — 비밀번호 자체를 모릅니다.
- **refresh_token → KMS 암호화**: 다시 써야 하므로 양방향(Q3). `youtube_channel.oauth_refresh_token_kms` 컬럼에 *암호문*으로 저장.
- **api_key secret → bcrypt 해시**: 검증만 하면 되므로 단방향. 평문은 발급 응답 1회만(§J, [[machine-identity-apikey]]).
- **IP 주소 → raw 저장**: 이건 비밀이 아니라 *감사 증거*입니다. PIPA 처리방침상 누가 언제 어디서 접근했는지 기록해야 하므로 `VARBINARY(16)`에 원본 그대로(IPv4 4byte / IPv6 16byte).
- **카드정보 → PG(PCI-DSS)**: 카드 데이터를 직접 다루면 PCI-DSS 인증 부담이 막대합니다. PG에 위임하고 우리는 `pg_payment_id`만 참조.

> 💡 **한 줄 요약**: 우리는 데이터마다 다르게 보호합니다 — 비밀번호·카드는 *아예 저장 안 함*(Firebase·PG에 위임), refresh_token은 KMS로 암호화, api_key secret은 bcrypt 해시, IP는 감사 목적의 원본 저장. "안 가지는 것"도 전략입니다.

---

## Q3. envelope encryption(봉투 암호화)이 뭔가요? 왜 그냥 한 번 암호화하면 안 되나요?

refresh_token을 KMS로 보호한다고 했는데, 정확히는 **envelope encryption(봉투 암호화)**을 씁니다. 이게 KMS의 핵심 원리입니다.

먼저 두 종류의 키를 알아야 합니다:

```
DEK (Data Encryption Key) — 실제 데이터를 암호화하는 열쇠
KEK (Key Encryption Key)  — 그 DEK를 암호화하는 더 큰 열쇠 (KMS가 보관)
```

비유: 중요한 문서를 **작은 금고(DEK)**에 넣고 잠급니다. 그 금고 열쇠는 다시 **은행 대여금고(KEK, KMS가 관리)**에 넣어둡니다. 문서를 읽으려면 ① 은행 금고를 열어 작은 금고 열쇠를 꺼내고 ② 그 열쇠로 작은 금고를 엽니다.

```
평문 refresh_token
   │ DEK로 암호화
   ▼
암호문 ───────────────┐  DB에 저장
                      │
DEK ──KEK(KMS)로 암호화─▶ 암호화된 DEK ─┘  암호문 옆에 함께 저장
                      │
KEK 자체는 KMS 안에만 존재 — 절대 우리 서버·DB로 나오지 않음
```

**왜 그냥 한 번만 암호화하지 않나요?** 단순 암호화의 딜레마는 *"암호화 열쇠를 어디 두느냐"*입니다. 열쇠를 DB나 서버에 두면, DB 유출 시 열쇠도 같이 새서 의미가 없습니다. envelope encryption은 이걸 해결합니다:

```
DB가 통째로 유출돼도:
  공격자가 얻는 것 = 암호문 + (KEK로 암호화된) DEK
  공격자가 못 얻는 것 = KEK (KMS 안에만 있음)
  → 암호화된 DEK를 풀 수 없으니 → DEK 못 얻음 → 암호문 못 풂
```

즉 *진짜 마스터키(KEK)는 절대 DB·서버 밖으로 나오지 않습니다.* 복호화가 필요할 때마다 KMS에 "이 암호화된 DEK 좀 풀어줘"라고 요청하고, KMS가 안전한 경계 안에서 풀어 평문 DEK를 잠깐 돌려줍니다.

추가 이점:
- **키 회전 효율**: KEK를 바꿔도 *DEK 암호문만* 다시 암호화하면 됩니다. 데이터 수억 건을 통째로 재암호화할 필요가 없습니다(§H.4 / OPS-4: KMS DEK 연 1회 회전).
- **감사·권한**: KMS가 "누가 언제 복호화를 요청했나"를 전부 로깅합니다.

> 💡 **한 줄 요약**: envelope encryption은 데이터를 DEK로 암호화하고, 그 DEK를 다시 KEK(KMS 안에만 있는 마스터키)로 감싸는 2겹 구조입니다. 마스터키가 DB 밖으로 안 나오므로, DB가 통째로 유출돼도 암호문을 풀 수 없습니다.

---

## Q4. api_key의 secret은 왜 "발급 시 1회만" 보여주나요? 다시 못 보면 불편하지 않나요?

`api_key.secret_hash`는 bcrypt로 해시되어 저장됩니다(§J·§H.3). 그래서 **평문은 발급 응답에 딱 1번만** 나타나고, 그 뒤로는 누구도 — 우리 DB 관리자조차 — 원본을 볼 수 없습니다.

"불편한데 왜?" — 이게 바로 안전의 이유입니다. Q1의 *분쇄* 개념을 떠올리세요. 우리는 `secret_hash`(분쇄된 조각)만 저장하므로, 거기서 평문을 *복원할 방법이 없습니다*. 못 보는 게 아니라 *애초에 가지고 있지 않은* 것입니다.

```
발급 시점:
  서버가 평문 키 생성 → 응답에 1회 노출 → bcrypt로 해시 → 해시만 DB 저장 → 평문 메모리에서 폐기
                          │                                              │
                  사용자가 이때 안전한 곳에 복사                    이 시점 이후 평문은 세상에서 사라짐

이후 인증:
  사용자가 보낸 키 ──bcrypt.compare──▶ DB의 secret_hash 와 일치? → 200 / 401
  (원본을 "꺼내 비교"하는 게 아니라 "다시 갈아서 비교")
```

이게 주는 보안 이점:

```
DB가 유출돼도:
  공격자가 얻는 것 = secret_hash (bcrypt 60자 해시)
  공격자가 못 얻는 것 = 평문 키 (단방향이라 역산 불가)
  → 훔친 해시로는 인증 못 함
```

만약 평문을 저장했다면? DB 유출 = 모든 api_key 즉시 탈취였을 겁니다. 그래서 "다시 못 봄"은 버그가 아니라 **설계된 안전장치**입니다. 사용자가 키를 잃어버리면? 복구가 아니라 *새 키 발급 + 구 키 revoke*가 정답입니다(rotation — [[machine-identity-apikey]] Q4·Q6).

**bcrypt와 salt** — bcrypt는 해시할 때마다 **salt**(무작위 값)를 섞습니다. 그래서 같은 키라도 해시 결과가 매번 다르고, 미리 계산해둔 해시 사전(rainbow table)으로 한꺼번에 깨는 공격이 막힙니다.

> 💡 **한 줄 요약**: api_key secret은 bcrypt 해시로만 저장되므로 평문이 *애초에 우리 손에 없습니다*. 발급 시 1회 노출은 "못 보는 불편"이 아니라, DB가 유출돼도 키가 안 새는 *설계된 안전장치*입니다. 분실 시엔 복구가 아니라 재발급+revoke.

---

## Q5. at-rest vs in-transit, 그리고 "평문 로그 절대 금지"는 무슨 의미인가요?

암호화에는 보호하는 *상태*가 두 가지 있습니다:

```
at-rest (저장 중)   : DB·디스크에 *가만히 있을 때* 보호 — KMS·bcrypt가 담당
in-transit (전송 중) : 네트워크로 *오갈 때* 보호 — TLS(HTTPS)가 담당
```

비유: at-rest는 *금고에 보관 중인 현금*, in-transit은 *현금 수송차로 옮기는 중인 현금*입니다. 둘 다 보호해야 하고, 보호 도구가 다릅니다. 우리 §H.3 표는 *at-rest*를 다루고, in-transit은 전 구간 HTTPS/TLS로 처리합니다.

그런데 이 두 겹을 다 갖춰도 **세 번째 구멍**이 있습니다 — **로그**입니다.

```
❌ 평문이 새는 가장 흔한 경로 = 로그·에러 메시지·디버그 출력
logger.info(`api key issued: ${plainSecret}`)   // ← 평문이 로그 파일에 영구 박힘!
console.log(req.body)                            // ← refresh_token·secret 통째로 노출
```

at-rest를 아무리 잘 암호화해도, 평문이 *로그에 한 줄 찍히는 순간* 모든 노력이 무너집니다. 로그는 보통 평문이고, 여러 시스템(수집기·검색엔진·백업)으로 복제되며, 접근 권한도 DB보다 느슨하기 때문입니다. 그래서 [[requirements]] SEC-4가 **"secret 관리 — KMS, 평문 로그 금지"**를 명시합니다.

```ts
// ✅ 로그에는 절대 평문 비밀을 넣지 않는다
logger.info('api key issued', { keyPrefix: 'ak_live_', orgPk });  // prefix만 (비밀 아님)
// secret·refresh_token·password 는 어떤 로그에도 등장 금지
```

**§H.4 rotation cadence(회전 주기)** — 비밀은 오래 둘수록 샜을 위험이 누적되므로 정기 교체합니다:

| 대상 | 주기 |
|---|---|
| DB 접속 계정 비밀번호 | 90일 (Secrets Manager 자동) |
| API Key | 90일 권장 (`rotated_at` 추적) |
| Firebase Service Account | 연 1회 |
| OAuth refresh_token | 정기 재발급 |
| KMS DEK | 연 1회 (OPS-4) |

> 💡 **한 줄 요약**: at-rest(저장·KMS/bcrypt)와 in-transit(전송·TLS)을 둘 다 보호해도, *평문이 로그에 찍히면* 전부 무너집니다 — 그래서 "평문 로그 절대 금지"가 KMS만큼 중요한 규칙이고, 비밀은 §H.4 주기대로 회전시킵니다.

---

## Q6. 데이터 분류(SEC-8)가 뭔가요? 왜 모든 걸 똑같이 암호화하지 않나요?

[[requirements]] SEC-8은 데이터를 **분류(classification)**하라고 요구합니다 — PII·민감·미성년·결제 4부류. 왜 다 똑같이 최고 수준으로 암호화하지 않을까요?

"전부 암호화하면 안전하지 않나?"는 직관적이지만 비현실적입니다. 암호화·복호화는 *비용*(성능·키 관리·운영 복잡도)이 있고, 무차별 적용하면 정작 중요한 데이터에 집중을 못 합니다. 그래서 **"무엇이 얼마나 민감한가"로 분류해서, 보호 수준을 차등 적용**합니다.

```
데이터 분류(SEC-8) → 보호 수준 결정

결제(payment)    : PG에 위임(PCI-DSS) — 우리가 안 가짐
비밀(secret)     : api_key·DEK 등 — bcrypt 해시 / KMS 암호화 (app-level)
미성년(under14)  : guardian(법정대리인) 정보 — 선별 app-level 암호화
민감(sensitive)  : PIPA 민감정보 — 선별 암호화
PII(일반 개인정보): email·phone 등 — at-rest 기본 + 접근통제·감사
```

여기서 **app-level 암호화(선별)**가 핵심입니다(SEC-8: "선별 app-level 암호화 — secret·guardian"). 모든 컬럼이 아니라 *secret과 미성년 보호자 정보처럼 특별히 민감한 것만* 애플리케이션 레벨에서 추가 암호화한다는 뜻입니다.

- **PII(개인식별정보)**: 이름·이메일·전화처럼 *개인을 식별*하는 정보. 기본 at-rest 보호 + 누가 접근했는지 감사(`audit_log`, IP 기록).
- **민감정보·미성년**: 법이 특별 보호를 요구하는 부류(PIPA). guardian 정보 등은 app-level 추가 암호화 대상.
- **결제**: 우리가 안 가짐(PG 위임)이 곧 보호.
- **비밀**: Q1~Q5에서 다룬 해시·KMS 대상.

분류가 먼저 있어야 *어디에 무슨 보호를 쓸지* 정할 수 있습니다. SEC-8이 "분류는 P0, 암호화 구현은 P1"인 이유 — *무엇이 민감한지부터 정의*하는 게 먼저입니다.

> 💡 **한 줄 요약**: 모든 데이터를 똑같이 암호화하는 건 비효율적이라, SEC-8은 PII·민감·미성년·결제로 *분류*해서 보호 수준을 차등 적용합니다. secret·미성년 보호자 정보처럼 특히 민감한 것만 app-level로 추가 암호화합니다.

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| **해시(hashing)** | 단방향 변환(복원 불가). 비밀번호·키처럼 원본 불필요한 비밀에 사용 |
| **암호화(encryption)** | 양방향(열쇠로 복호화 가능). refresh_token처럼 다시 써야 하는 것에 사용 |
| **bcrypt** | salt 내장 단방향 해싱 함수. 비밀번호·api_key secret |
| **salt** | 해시에 섞는 무작위 값. 같은 입력도 매번 다른 해시 → rainbow table 방어 |
| **KMS** | Key Management Service(AWS). 마스터키(KEK)를 안전 경계 안에 보관·관리 |
| **KEK** | Key Encryption Key. DEK를 암호화하는 마스터키. KMS 안에만 존재 |
| **DEK** | Data Encryption Key. 실제 데이터를 암호화하는 열쇠 |
| **envelope encryption** | DEK로 데이터를, KEK로 DEK를 감싸는 2겹 구조 |
| **at-rest** | 저장 중 암호화(DB·디스크). KMS·bcrypt 담당 |
| **in-transit** | 전송 중 암호화(네트워크). TLS/HTTPS 담당 |
| **PCI-DSS** | 카드 데이터 보안 표준. 우리는 PG에 위임해 미보유 |
| **PII** | 개인식별정보(이름·이메일·전화). PIPA 보호 대상 |
| **데이터 분류** | 민감도별 분류(PII/민감/미성년/결제). 보호 수준 차등 근거(SEC-8) |
| **rotation cadence** | 비밀 정기 교체 주기(§H.4, 예: api_key 90일·DEK 연 1회) |

---

## 테스트 방법
> 🧪 *실제 DB·ORM·운영에서 돌리는 법*: [[testing-strategy]] · [[orm-testing-drizzle]]


비밀 관리의 테스트 핵심은 **"평문이 응답·로그·DB 어디에도 새지 않는가"**를 negative test로 확인하는 것입니다 — "되면 안 되는 게 안 되는가".

**① 통합 테스트 (vitest + supertest) — 평문 비노출·해시 검증**

```ts
// secret-handling.spec.ts
describe('비밀 처리 — 평문 비노출', () => {
  it('api_key 발급 응답에만 secret 평문이 1회 노출된다', async () => {
    const res = await request(app)
      .post('/v1/api-keys')
      .set('Authorization', `Bearer ${ownerToken}`)
      .set('x-org-pk', org.pk)
      .send({ scopes: ['lecture:read'] });
    expect(res.status).toBe(201);
    expect(res.body.secret).toMatch(/^ak_live_/);   // ★ 발급 시엔 평문 노출
    expect(res.body.keyPrefix).toBe('ak_live_');
  });

  it('재조회하면 prefix만 보이고 secret 평문은 "없다"', async () => {
    const issued = await issueApiKey(org);
    const res = await request(app)
      .get(`/v1/api-keys/${issued.id}`)
      .set('Authorization', `Bearer ${ownerToken}`);
    expect(res.body.keyPrefix).toBe('ak_live_');
    expect(res.body.secret).toBeUndefined();        // ★ 평문 부재 단언 (없음)
    expect(res.body.secretHash).toBeUndefined();    // 해시조차 응답에 안 내보냄
  });

  it('DB에 저장된 건 평문이 아니라 해시다 (평문 ≠ 저장값)', async () => {
    const issued = await issueApiKey(org);
    const row = await db.query.apiKey.findFirst({ where: eq(apiKey.id, issued.id) });
    expect(row.secretHash).not.toBe(issued.secret); // ★ 저장값은 평문과 다름
    expect(row.secretHash).toMatch(/^\$2[aby]\$/);  // bcrypt 포맷
    // 그래도 검증은 통과 — "다시 갈아서 비교"
    expect(await bcrypt.compare(issued.secret, row.secretHash)).toBe(true);
    // 틀린 키는 거부
    expect(await bcrypt.compare('wrong_key', row.secretHash)).toBe(false);
  });
});
```

**② 로그에 비밀이 안 찍히는지 — 로그 스파이로 검증**

```ts
it('비밀 발급 흐름의 어떤 로그에도 secret 평문이 등장하지 않는다', async () => {
  const logged: string[] = [];
  const spy = vi.spyOn(logger, 'info')
    .mockImplementation((msg, meta) => { logged.push(JSON.stringify({ msg, meta })); });

  const res = await request(app).post('/v1/api-keys')
    .set('Authorization', `Bearer ${ownerToken}`).set('x-org-pk', org.pk)
    .send({ scopes: ['lecture:read'] });

  const plainSecret = res.body.secret;
  const allLogs = logged.join('\n');
  expect(allLogs).not.toContain(plainSecret);        // ★ 평문 로그 금지(SEC-4)
  expect(allLogs).toContain('ak_live_');             // prefix는 찍혀도 됨(비밀 아님)
  spy.mockRestore();
});
```

**③ 무엇을 단언하나 (체크리스트)**

```text
□ api_key 발급 응답에만 secret 평문 1회 / 재조회 응답엔 prefix만 (secret undefined)
□ DB 저장값(secret_hash)이 평문과 다름 + bcrypt 포맷($2b$...)
□ bcrypt.compare: 올바른 키 → true, 틀린 키 → false (복원 아닌 재해시 비교)
□ 어떤 로그·에러 메시지에도 평문 secret/refresh_token/password 미등장
□ refresh_token DB 저장값이 평문이 아님 (KMS 암호문) — 복호화해야 원본
□ 비밀번호·카드정보는 우리 DB 어디에도 없음 (Firebase·PG 위임)
```

> 💡 **테스트 한 줄 요약**: "평문이 응답(발급 1회 제외)·로그·DB 저장값 어디에도 없는가"를 *없음(undefined)·불일치(≠ 평문)* 단언으로 강제하세요. bcrypt는 복원이 아니라 *재해시 비교*가 통과해야 합니다.

---

## 마치며

비밀 관리는 화려한 알고리즘이 아니라 **규율**입니다. 그 규율은 몇 개의 질문으로 압축됩니다:

1. **이 비밀, 나중에 원본이 필요한가?** → 아니면 해시(bcrypt), 맞으면 암호화(KMS). 이걸 거꾸로 하면 비밀번호를 복호화 가능하게 두거나 refresh_token을 영영 못 쓰게 됩니다.
2. **이 비밀, 우리가 꼭 가져야 하나?** → 비밀번호·카드처럼 안 가져도 되면 Firebase·PG에 위임하세요. 안 가진 데이터는 안 샙니다.
3. **암호화 열쇠는 어디 있나?** → DB 옆이 아니라 KMS 안(envelope encryption). 마스터키가 데이터와 같이 새면 암호화는 무의미합니다.
4. **이 비밀이 로그에 찍히나?** → at-rest·in-transit을 아무리 잘해도 평문 로그 한 줄이면 끝입니다.

새 기능에서 어떤 비밀(토큰·키·개인정보)을 다루게 되면, 코드를 짜기 전에 위 네 질문을 먼저 던지고 §H.3 암호화 경계 표 어디에 속하는지를 확인하세요. 가장 안전한 비밀은 *우리가 평문으로 가지고 있지 않은 비밀*입니다.

---

## 연결된 개념

- [[machine-identity-apikey|머신 신원 (api_key)]] — secret_hash가 발급 1회 노출·bcrypt로 보호되는 그 키
- [[pk-ulid-strategy|BIGINT pk + ULID public_id]] — 식별자는 비밀이 아님(노출 가능) vs secret은 비밀(해시)
- [[idempotency-key|멱등성 키]] — idempotency_key는 비밀이 아니라 중복 방지 식별자(혼동 주의)
- [[pipa-consent|PIPA 동의]] — PII·미성년 데이터 분류와 법적 보호 맥락
- [[bola-object-authz|BOLA 객체 수준 인가]] — 데이터 접근 자체를 막는 인가 계층(암호화와 상보)
> 소스 문서
- [[schema-reference]] — §H.3 암호화 경계, §H.4 secret rotation cadence, §J api_key(secret_hash·prefix)
- [[requirements]] — SEC-4(KMS·평문 로그 금지)·SEC-6(rotation)·SEC-8(데이터 분류·선별 암호화)·OPS-4(KMS DEK 회전)
