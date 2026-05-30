# Firebase = 인증만 — 인가는 우리 DB

## 배경

Firebase Auth를 쓰고 있다. Firebase는 JWT를 발급하고, 그 안에 Custom Claims를 넣을 수 있다.

```typescript
// Firebase Admin SDK로 Custom Claims 설정 가능
await admin.auth().setCustomUserClaims(uid, {
  role: 'TEACHER',
  orgId: 'org_abc123',
  permissions: ['publish_video', 'view_students'],
});
```

"Firebase로 인증도 하고 인가(권한 관리)도 하면 되지 않나?" 라는 질문이 자연스럽게 나온다.

---

## 일반 권고 — Firebase Custom Claims로 인가까지

```typescript
// JWT에 role/permission 포함
const decoded = await admin.auth().verifyIdToken(token);
if (decoded.role !== 'TEACHER') throw ForbiddenException();
```

**권고 근거**:
- DB 조회 없이 JWT만으로 권한 확인 → 빠름
- Firebase가 토큰 발급·검증·만료 전부 관리
- 별도 권한 DB 불필요

**문제점**:

```
문제 1: JWT 크기 제한
  Custom Claims는 1KB 이하 권장
  → org별 role + permissions 전부 넣으면 초과

문제 2: stale claims (JWT TTL 최대 1시간)
  TEACHER → 해고(SUSPENDED) 후에도 1시간 동안 JWT가 유효
  → org에서 쫓겨난 사람이 1시간 동안 API 접근 가능

문제 3: DB 레벨 강제 불가
  DB에 membership이 없어도 JWT에 role이 있으면 통과
  → "DB 진실"과 "JWT 진실"이 불일치

문제 4: vendor lock-in
  Firebase 장애 = 전체 권한 시스템 장애
  Firebase → 다른 AuthN 제공자로 교체 시 권한 모델 전부 재설계
```

---

## 우리 결정 — Firebase = 인증(AuthN)만, 인가(AuthZ)는 우리 DB

```
Firebase Auth                        우리 platform_db
─────────────────                    ──────────────────────────────
"이 사람이 실제로 존재하는가?"  →      "이 사람이 무엇을 할 수 있는가?"
  JWT 발급 / 검증                       membership, delegation_grant
  비밀번호 관리                         org_entitlement, audit_log
  소셜 로그인                           3-gate (A소속 · B이용권 · C정책)
  이메일 인증
```

**firebase_uid는 조회 키일 뿐, PK도 FK도 아니다** — 불변식 #1

```typescript
// 올바른 패턴: firebase_uid로 우리 DB에서 user 조회
@Injectable()
export class FirebaseAuthGuard {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const token = extractToken(context);
    const decoded = await admin.auth().verifyIdToken(token); // Firebase: 진짜 사람인지만 확인

    // 권한은 우리 DB에서
    const user = await getUserByFirebaseUid(decoded.uid);
    request.user = user; // 이후 Gate A/B/C는 우리 DB만 읽음
    return !!user;
  }
}
```

**Stale JWT 방어**:
- 일반 API: JWT 유효하면 통과 (TTL 내 stale 허용 — 읽기는 무방)
- 민감한 쓰기(`@VerifyOnDb`): DB에서 membership 재확인 → revoke 후 즉시 차단

---

## 트레이드오프

| 항목 | Firebase Custom Claims | 우리 DB 인가 |
|---|---|---|
| 권한 체크 속도 | JWT 파싱만 (빠름) | DB 조회 필요 |
| stale claims 위험 | 1시간 노출 | @VerifyOnDb로 즉시 차단 |
| 권한 복잡도 지원 | 1KB 제한 | 무제한 (테이블 설계 자유) |
| vendor lock-in | Firebase 의존 | Firebase 교체 가능 (uid만 키) |
| 감사 로그 | Firebase Console만 | audit_log 완전 제어 |

---

## 향후 조건

Firebase에서 다른 AuthN 제공자(Auth0, Cognito 등)로 교체 시:
- `identity_user.firebase_uid` → `external_auth_uid` 컬럼명 변경
- AuthGuard 구현만 교체
- Gate A/B/C 및 권한 모델은 변경 없음

이것이 firebase_uid를 PK/FK로 쓰지 않은 이유다.

---

## 관련 문서

- [[gate-abc-flow]] — Firebase 인증 통과 후 우리 DB가 도는 3-gate 흐름
- [[role-as-code]] · [[auth-projection]] — "우리 DB가 인가한다"의 구체 내용
> 소스 문서
- [[architecture]] — §3.1 불변식 #1 (Firebase = 인증, 인가 = 우리 DB)
- [[schema-reference]] — §D.1 identity_user(firebase_uid), §E 3-gate 구조
