# 인증 및 권한 (academy-api 적용)

> 작성일: 2026-05-23 · 갱신: 2026-05-23 (centralized identity 도입)
> **본 문서는 회사 공통 [`identity-policy.md`](identity-policy.md) 의 academy-api 특화 적용 (Layer 3) 만 다룬다.**
>
> Layer 1 (Firebase Auth) · Layer 2 (membership tuple, `packages/identity`) 일반 정책은 [`identity-policy.md`](identity-policy.md) 참조.

---

## 1. Layer 책임 분리

```
Layer 1: Firebase Auth      — identity (누구인가)        → identity-policy.md
   └ Email/password 또는 Custom Token (Kakao/Naver) → JWT 발급
   
Layer 2: packages/identity  — membership (어디 소속이냐)  → identity-policy.md
   └ identity_user / membership tuple / Custom Claims 관리
   
Layer 3: academy-api auth   — policy (무엇을 할 수 있냐)  ← 본 문서
   └ membership + trust_relationship → CASL ability
   └ can('action', 'subject', resource)
```

academy-api 는 Layer 1·2 의 결과 (`req.user`, `req.memberships`) 를 받아 **academy 만의 비즈니스 룰** 을 적용한다.

---

## 2. 로그인 흐름 (academy-api 관점)

```
[Vite SPA]
   firebase.auth().signInWithEmailAndPassword(...)
       ↓
   ID token (with custom claims = memberships) 획득
       ↓
   API 호출 시 Authorization: Bearer <id_token>
       ↓
[academy-api]
   FirebaseAuthGuard (packages/identity 제공)
     1. firebase-admin.verifyIdToken
     2. JWT custom claims 에서 memberships 추출 (in-token, DB hit 0)
     3. identity_user lookup (firebase_uid → user_pk)
     4. academy product 의 membership 만 필터링
       ↓
   AcademyPolicyGuard (academy-api Layer 3)
     5. trust_relationship 조회 (active grants for this user)
     6. defineAcademyAbility(user, memberships, grants) → CASL ability
     7. req.user / req.ability 부착
       ↓
   라우트 핸들러
     @CheckAbility((ability) => ability.can('view', lecture))
```

→ Read 경로는 in-token claims 로 DB hit 0. Sensitive write 는 별도 재검증 (§7 참조).

---

## 3. 사용자 생성 (academy product 관점)

| 누가 | 어떻게 |
|---|---|
| **DIRECTOR** | 본인 가입 → `identity_user` row INSERT + `organization(type='ACADEMY')` INSERT + `academy_config` INSERT + `membership(org_pk, role='DIRECTOR')` INSERT + `org_entitlement(source='MANUAL')` INSERT |
| **TEACHER** | DIRECTOR 가 초대 (이메일) → `membership_invite` INSERT → 초대 메일 발송 → 강사 첫 로그인 시 `identity_user` 활성 + `membership(role='TEACHER')` INSERT |
| 학생 (v1.0+) | DIRECTOR / TEACHER 가 등록 → `identity_user(type='HUMAN')` + `membership(role='STUDENT')` |
| Agent (예: scheduling bot) | DIRECTOR 또는 시스템이 등록 → `identity_user(type='SERVICE')` + `membership(role='AGENT_*')` |

→ academy 만의 정보 (학원 가입 폼 입력, 학원 OAuth 정책 등) 는 Layer 3 의 academy 도메인 처리. 사용자 생성 자체는 Layer 2 (`packages/identity`) 의 표준 API 사용.

---

## 4. CASL Ability 패턴

### 4.1 라이브러리

- `@casl/ability` — TypeScript native, RBAC + ABAC 지원
- `@casl/nestjs` 또는 자체 Guard 통합

### 4.2 Subject (보호 대상)

```typescript
type Subjects =
  | 'Lecture'
  | 'LectureChunk'
  | 'VideoAsset'
  | 'YoutubeVideo'
  | 'YoutubeChannel'
  | 'TrustRelationship'
  | 'User'
  | 'TeacherMaterial'    // lecture.type='material'
  | 'all';
```

### 4.3 Action

```typescript
type Actions =
  | 'view'
  | 'create'
  | 'update'
  | 'delete'
  | 'request_publish'
  | 'publish'
  | 'manage';            // 모든 액션 포함
```

### 4.4 Ability 빌더 (Layer 3 — academy-specific)

```typescript
// academy-api/auth/ability.ts
// Layer 2 (packages/identity) 가 주입한 user + memberships + trust grants 를 받아 academy ability 빌드.

export function defineAcademyAbility(
  user: IdentityUser,                  // packages/identity 에서 옴
  memberships: Membership[],           // packages/identity 에서 옴 (academy product 만 필터링됨)
  grants: TrustGrant[],                // academy-api 의 Layer 3 데이터
): AcademyAbility {
  const { can, cannot, build } = new AbilityBuilder<AcademyAbility>(AcademyAbility);
  
  // 멀티 워크스페이스: 한 user 가 N 학원의 멤버 가능
  for (const m of memberships) {
    const tenantScope = { academyPk: m.workspacePk };
    
    if (m.role === 'DIRECTOR') {
      can('manage', 'all', tenantScope);
      cannot('view', 'TrustRelationship', { granted_by_pk: { $ne: user.userPk } });
    }
    
    if (m.role === 'TEACHER') {
      can(['view','create','update','delete'], 'Lecture', {
        ...tenantScope, teacherPk: user.userPk,
      });
      can('request_publish', 'YoutubeVideo', {
        ...tenantScope,
        lecture: { teacherPk: user.userPk },
      });
      
      // ABAC: trust_relationship 으로 확장 (academy-specific Layer 3)
      const academyGrants = grants.filter(g => g.academyPk === m.workspacePk);
      for (const grant of academyGrants) {
        if (grant.scope === 'view_all_lectures') {
          can('view', 'Lecture', tenantScope);
          can('view', 'TeacherMaterial', tenantScope);
        }
        if (grant.scope === 'delete_all_lectures') {
          can('delete', 'Lecture', tenantScope);
        }
        if (grant.scope === 'auto_publish_own') {
          can('publish', 'YoutubeVideo', {
            ...tenantScope, lecture: { teacherPk: user.userPk },
          });
        }
        if (grant.scope === 'auto_publish_all') {
          can('publish', 'YoutubeVideo', tenantScope);
        }
        if (grant.scope === 'view_all_materials') {
          can('view', 'TeacherMaterial', tenantScope);
        }
      }
    }
    
    if (m.role.startsWith('AGENT_')) {
      // Agent role 별 ability — scheduling agent 등 (v2.0+)
      can(['view'], 'Lecture', tenantScope);
      // 자세한 룰은 agent role 정의에 따라
    }
    
    cannot('manage', 'YoutubeChannel');   // 학원장 only
    cannot('manage', 'TrustRelationship'); // 학원장 only
  }
  
  return build();
}
```

---

## 5. Permission Matrix (요약)

| Action | DIRECTOR | TEACHER 기본 | TEACHER + directorGrant |
|---|---|---|---|
| `view Lecture` | 학원 모두 | 본인만 | + `view_all_lectures` |
| `delete Lecture` | 학원 모두 | 본인만 | + `delete_all_lectures` |
| `view TeacherMaterial` | 학원 모두 | 본인만 | + `view_all_materials` |
| `request_publish Video` | (불필요) | 본인 강의만 | — |
| `publish Video` | 학원 모두 | 불가 | + `auto_publish_own` (본인) / `auto_publish_all` (학원) |
| `manage YoutubeChannel` | 학원 본인 | 불가 | 불가 |
| `manage TrustRelationship` | 학원 본인 | 불가 | 불가 |

### Trust grant 의미

- `auto_publish_own` — 신뢰 받은 강사가 본인 강의 자동 publish (검수 우회)
- `auto_publish_all` — 학원의 모든 영상 publish (사실상 부원장급)
- `view_all_lectures` — 학원 내 강사들 강의 공유 / 협업 강의 참고
- `view_all_materials` — 학원 내 자료 공유
- `delete_all_lectures` — 위험 권한, 잘 안 줌
- **`manage_youtube_channel`** — 학원장이 본인 Google 계정으로 YouTube 채널 admin 불가 시 운영 직원에게 OAuth 진행 권한 위임 — BDD F38-01
- **`approve_videos`** — 학원장 부재 시 검수 권한 위임 (휴가·인수인계 시점) — BDD F40-01

### Permission Matrix (Trust Grant 확장 행)

| Action | DIRECTOR | TEACHER 기본 | TEACHER + grant |
|---|---|---|---|
| `manage YoutubeChannel` (OAuth + 정책 설정) | ✓ | ✗ | **✓ (manage_youtube_channel)** |
| `approve YoutubeVideo` (검수 승인/반려) | ✓ | ✗ | **✓ (approve_videos)** |

### Lockout 방지 (마지막 DIRECTOR)

CASL ability 검증 외에 추가 정책 — 학원의 마지막 DIRECTOR 본인 권한 회수는 차단 (BDD F37-01):

```typescript
async function canRevokeMembership(
  actor: User,
  target: User,
  product: string,
  workspaceId: number,
  role: string,
): Promise<boolean> {
  if (role !== 'DIRECTOR') return true;  // DIRECTOR 외에는 항상 허용
  
  const activeDirectors = await db.membership.count({
    where: { product, workspace_pk: workspaceId, role: 'DIRECTOR', status: 'active' },
  });
  
  if (activeDirectors <= 1) {
    throw new BadRequestException(
      '마지막 DIRECTOR 권한은 회수 불가. 다른 사용자에게 DIRECTOR 부여 후 재시도.',
    );
  }
  return true;
}
```

---

## 6. NestJS 통합

### 6.1 Auth Guard (`packages/identity` 제공 — Layer 1+2)

```typescript
// packages/identity 가 제공
import { FirebaseAuthGuard } from '@aiagent/identity';

@Injectable()
export class FirebaseAuthGuard implements CanActivate {
  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const req = ctx.switchToHttp().getRequest();
    const token = extractBearerToken(req);
    
    // Layer 1: identity 인증
    const decoded = await this.firebase.verifyIdToken(token);
    
    // Layer 2: identity_user + membership lookup
    // JWT custom claims 우선 사용 (in-token, fast path)
    // 없거나 stale 시 DB fetch
    const user = await this.identityService.findByFirebaseUid(decoded.uid);
    const allMemberships = decoded.memberships
      ?? await this.identityService.findActiveMemberships(user.user_pk);
    
    req.user = user;
    req.allMemberships = allMemberships;
    return true;
  }
}
```

### 6.2 Academy Policy Guard (Layer 3, academy-api 자체)

```typescript
@Injectable()
export class AcademyPolicyGuard implements CanActivate {
  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const req = ctx.switchToHttp().getRequest();
    
    // academy product 의 membership 만 필터링
    const academyMemberships = req.allMemberships
      .filter(m => m.product === 'academy');
    
    if (academyMemberships.length === 0) {
      throw new ForbiddenException('No academy access');
    }
    
    // Layer 3: academy-specific grants
    const grants = await this.trustService.findActiveGrants(req.user.userPk);
    
    req.academyMemberships = academyMemberships;
    req.ability = defineAcademyAbility(req.user, academyMemberships, grants);
    return true;
  }
}
```

### 6.3 Policy Decorator

```typescript
@Controller('lectures')
@UseGuards(FirebaseAuthGuard, AcademyPolicyGuard, AbilityGuard)
export class LectureController {
  @Get(':id')
  @CheckAbility((ability, req) => ability.can('view', req.params.lecture))
  async getLecture(@Param('id') id: number) { ... }
  
  @Delete(':id')
  @CheckAbility((ability, req) => ability.can('delete', req.params.lecture))
  @VerifyOnDb()  // sensitive: DB 재검증 (§7)
  async deleteLecture(@Param('id') id: number) { ... }
}
```

### 6.4 멀티테넌트 Interceptor

```typescript
@Injectable()
export class AcademyScopeInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler) {
    const req = ctx.switchToHttp().getRequest();
    // 멀티 워크스페이스 사용자는 헤더 또는 URL path 로 학원 컨텍스트 명시
    const academyPk = req.headers['x-academy-pk'] ?? req.params.academyPk;
    
    // 권한 검증: 사용자가 해당 academy 의 membership 보유?
    const hasAccess = req.academyMemberships.some(m => m.workspacePk === Number(academyPk));
    if (!hasAccess) throw new ForbiddenException('No access to academy');
    
    // AsyncLocalStorage 에 주입 → ORM middleware 가 WHERE 자동 추가
    return this.als.run({ academyPk: Number(academyPk) }, () => next.handle());
  }
}
```

→ 한 사용자가 N 학원 가입 시 컨텍스트 명시 (headers / URL). 사용자 잘못된 학원 contexts 시도 시 차단.

---

## 7. Sensitive Write DB 재검증

JWT custom claims 는 1시간 TTL 동안 stale 가능. Sensitive mutation 은 DB membership 재 fetch 로 stale 권한 차단.

**Sensitive 대상 (예시):**
- `POST /videos/:id/publish` — YouTube 업로드 트리거
- `DELETE /lectures/:id` — 강의 영구 삭제
- `POST /trust-relationships` — 권한 grant
- `DELETE /trust-relationships/:id` — 권한 revoke
- `POST /youtube-channel/oauth` — OAuth 연결

**구현:**

```typescript
// @VerifyOnDb() 데코레이터 (Layer 3)
export function VerifyOnDb(): MethodDecorator {
  return SetMetadata('verifyOnDb', true);
}

@Injectable()
export class AbilityGuard implements CanActivate {
  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const req = ctx.switchToHttp().getRequest();
    const requiresDbVerify = this.reflector.get<boolean>('verifyOnDb', ctx.getHandler());
    
    // 1차: JWT claims 기반 ability 빠른 체크
    if (!req.ability.can(action, subject)) return false;
    
    // 2차: sensitive 시 DB 재검증
    if (requiresDbVerify) {
      const freshMemberships = await this.identityService.findActiveMemberships(req.user.userPk);
      const freshGrants = await this.trustService.findActiveGrants(req.user.userPk);
      const freshAbility = defineAcademyAbility(req.user, freshMemberships, freshGrants);
      
      if (!freshAbility.can(action, subject)) {
        // Stale claims — 권한 회수됨
        throw new ForbiddenException('Permission revoked');
      }
    }
    
    return true;
  }
}
```

**성능 영향:**
- Routine read: 0 추가 hit (JWT 만)
- Sensitive write: 1 DB hit (~5ms) — 학원당 일 N건 수준이라 부담 없음

→ 보안 정밀성 + 성능 양립.

---

## 8. trust_relationship 관리 UI

학원장 화면에서:

```
강사 목록:
  김지영  ──────────────  [view_all] ✓  [auto_publish_own] ✓  [view_all_materials] ✓
  박민준  ──────────────  [view_all] ☐  [auto_publish_own] ☐
  
[수정] [감사 로그]
```

- 토글로 grant 부여/철회
- 변경 이력 audit_log 에 기록 (v0.5+ 정식 audit)
- `effective_to` 만료 시간 설정 가능

---

## 9. Firebase Auth 정책

| 항목 | 설정 |
|---|---|
| Sign-in methods | Email/password (MVP), Google OAuth (v0.5+), Kakao/Naver (v1.0+ Custom Token) |
| Email verification | 필수 |
| 비밀번호 강도 | 8자+ 영문/숫자/특수 |
| 다중 디바이스 | 허용 |
| 세션 timeout | 1시간 (refresh token 자동 갱신) |
| 패스워드 reset | Firebase 표준 (이메일 링크) |
| Multi-factor (MFA) | v0.5+ DIRECTOR 만 |

---

## 10. ReBAC 확장 (v0.5+)

관계 기반 권한 확장 시 trust_relationship 테이블에 새 scope 추가:

```
scope ENUM 확장:
  ... (기존)
  'view_student_X'        -- 강사가 특정 학생 X 정보 view (학생 면담 prep)
  'co_teach_class_Y'      -- 강사가 클래스 Y 공동 강사 (영상 view/edit)
  'assist_teacher_Z'      -- 강사 Z 의 조교 (자료 view)
```

또는 별도 `relationship` 테이블 추가:
```
relationship
  ├ actor_pk, target_pk
  ├ type ENUM('teaches','co_teaches','assists','manages',...)
  ├ effective_from, effective_to
```

CASL ability 빌더가 이 관계를 읽어 조건 추가. 코드 구조는 동일.

---

## 11. Secret 관리

### 11.1 MVP (env 기반)

| Secret | 보관 |
|---|---|
| Firebase admin private key | env var (K8s Secret) |
| Anthropic / Google / OpenAI keys | env var |
| HyperFrames key | env var |
| NHN / SOLAPI keys | env var |
| MySQL / Redis 패스워드 | env var |

### 11.2 KMS-required (학원별 동적)

| Secret | 보관 |
|---|---|
| **YouTube OAuth refresh_token** | **AWS KMS envelope encrypted** → `youtube_channel.oauth_refresh_token_kms` |

각 학원의 refresh_token 은 학원별 KEK(Key Encryption Key) 로 암호화. 학원 데이터 격리.

### 11.3 절대 금지

- `.env*` git commit (본 monorepo 의 known issue 인지)
- Production refresh_token 의 평문 로그 출력
- Slack/이메일에 키 첨부

---

## 12. PIPA / 국외 이전

Firebase Auth 는 사용자 데이터 (이메일·해시된 비번) 를 US 에 저장. 학원 가입 시 약관에 명시:

> "본 서비스는 사용자 인증을 위해 Google Firebase Authentication 을 사용합니다. 이메일·비밀번호 해시·로그인 시각 등은 미국 Google 서버에 저장됩니다."

자세한 PIPA 트랙은 [`risks.md`](risks.md) 참조.
