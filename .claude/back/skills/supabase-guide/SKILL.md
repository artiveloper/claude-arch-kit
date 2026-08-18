---
name: supabase-guide
description: >
  GamePot Next.js + Supabase 실무 가이드.
  Supabase, SSR 인증, @supabase/ssr, RLS, Row Level Security, 타입 생성,
  publishable key, service_role, middleware, 마이그레이션 관련 작업 시 참조.
  클라이언트 설정, 인증 패턴(getUser vs getSession), RLS 성능 최적화, 보안 규칙 포함.
  GamePot 특화: admin role(app_metadata), Pterodactyl 키 보호, server_action_logs RLS.
---

# GamePot Supabase 가이드

> 원칙: DB는 항상 RLS로 보호한다. 클라이언트는 절대 신뢰하지 않는다.

---

## 1. 클라이언트 설정

### 패키지

```bash
pnpm add @supabase/supabase-js @supabase/ssr
```

### Browser Client (Client Component용)

```ts
// apps/admin/src/lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr'
import type { Database } from '@gamepot/db'

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
  )
}
```

### Server Client (Server Component / Server Action / Route Handler용)

```ts
// apps/admin/src/lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import type { Database } from '@gamepot/db'

export async function createSupabaseServerClient() {
  const cookieStore = await cookies()
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll() },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            )
          } catch {
            // Server Component에서는 쿠키 set 불가 — 무시
          }
        },
      },
    }
  )
}
```

### Admin Client (서버 전용 — RLS 우회)

```ts
// apps/admin/src/lib/supabase/admin.ts
import { createClient } from '@supabase/supabase-js'  // ← @supabase/ssr 아님
import type { Database } from '@gamepot/db'

export function createSupabaseAdminClient() {
  return createClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    {
      auth: { autoRefreshToken: false, persistSession: false },
    }
  )
}
```

---

## 2. 인증 패턴

### getUser() vs getSession() — 중요

```ts
// ✅ getUser() — DB 검증, 서버에서 항상 이걸 사용
const { data: { user }, error } = await supabase.auth.getUser()
if (!user) return Response.json({ error: 'Unauthorized' }, { status: 401 })

// ❌ getSession() — 쿠키만 확인, 서버에서 신뢰 불가
const { data: { session } } = await supabase.auth.getSession()
```

- `getUser()` → 서버 API Route / Server Action에서 항상 사용. DB round-trip 발생하지만 검증됨
- `getSession()` → 클라이언트 사이드 UI 목적으로만 허용

### JWT Claims — Role 확인

```ts
// ✅ app_metadata — 서버에서만 설정 가능, 신뢰 가능
const { data: { user } } = await supabase.auth.getUser()
const role = user?.app_metadata?.role  // 'admin' | undefined

// ❌ user_metadata — 사용자가 직접 수정 가능, 인가에 사용 금지
const role = user?.user_metadata?.role
```

### requireAdmin 헬퍼

```ts
// apps/admin/src/lib/supabase/server.ts
export async function requireAdmin() {
  const supabase = await createSupabaseServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Unauthorized')
  if (user.app_metadata?.role !== 'admin') throw new Error('Forbidden')
  return { supabase, user }
}
```

### Middleware (토큰 갱신)

토큰은 middleware에서 반드시 갱신해야 한다. 누락 시 Server Component에서 세션 만료.

```ts
// apps/admin/src/middleware.ts
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value))
          supabaseResponse = NextResponse.next({ request })
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          )
        },
      },
    }
  )

  // 반드시 호출 — 토큰 갱신 트리거
  const { data: { user } } = await supabase.auth.getUser()

  if (!user && !request.nextUrl.pathname.startsWith('/auth')) {
    const url = request.nextUrl.clone()
    url.pathname = '/auth/login'
    return NextResponse.redirect(url)
  }

  return supabaseResponse
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)'],
}
```

---

## 3. API 키 관리

| 키 | 용도 | 노출 |
|----|------|------|
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Browser/SSR 클라이언트 | 공개 OK (RLS가 보호) |
| `SUPABASE_SERVICE_ROLE_KEY` | Admin 작업, RLS 우회 | 절대 클라이언트 노출 금지 |
| `PTERODACTYL_API_KEY` | Pterodactyl Application API (서버 생성/삭제/조회) | 절대 클라이언트 노출 금지 |
| `PTERODACTYL_CLIENT_KEY` | Pterodactyl Client API (전원 제어) | 절대 클라이언트 노출 금지 (상세 SSoT: `/pterodactyl-domain`) |

```bash
# .env.local (git 제외)
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_xxx
SUPABASE_SERVICE_ROLE_KEY=sb_secret_xxx

PTERODACTYL_API_URL=https://panel.example.com
PTERODACTYL_API_KEY=ptla_xxx
PTERODACTYL_CLIENT_KEY=ptlc_xxx
```

---

## 4. RLS (Row Level Security)

> public 스키마의 모든 테이블에 RLS를 활성화한다. 예외 없음.

### 기본 설정

```sql
-- 테이블 생성 후 항상 실행
ALTER TABLE public.game_servers ENABLE ROW LEVEL SECURITY;
-- RLS 활성화 + 정책 없음 = 완전 차단 (안전한 기본값)
```

### 성능 최적화 — `(select auth.uid())` 필수

```sql
-- ❌ 모든 행마다 함수 실행 → 느림
USING (auth.uid() = user_id)

-- ✅ 한 번 실행 후 캐시 → 빠름
USING ((select auth.uid()) = user_id)
```

### 정책 구조 원칙

| 작업 | USING | WITH CHECK |
|------|-------|------------|
| SELECT | ✅ | ❌ |
| INSERT | ❌ | ✅ |
| UPDATE | ✅ (기존 행) | ✅ (새 값) |
| DELETE | ✅ | ❌ |

### GamePot RLS 패턴

**game_servers (핵심):**
```sql
ALTER TABLE game_servers ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users_select_own_servers" ON game_servers
  FOR SELECT TO authenticated
  USING ((select auth.uid()) = user_id);

CREATE POLICY "users_insert_own_servers" ON game_servers
  FOR INSERT TO authenticated
  WITH CHECK ((select auth.uid()) = user_id);

CREATE POLICY "users_update_own_servers" ON game_servers
  FOR UPDATE TO authenticated
  USING ((select auth.uid()) = user_id)
  WITH CHECK ((select auth.uid()) = user_id);

CREATE POLICY "users_delete_own_servers" ON game_servers
  FOR DELETE TO authenticated
  USING ((select auth.uid()) = user_id);

-- admin: app_metadata.role 기반 (user_metadata 금지)
CREATE POLICY "admin_full_access_servers" ON game_servers
  FOR ALL TO authenticated
  USING ((select auth.jwt()->'app_metadata'->>'role') = 'admin');
```

**server_action_logs (서버 소유자 기반):**
```sql
ALTER TABLE server_action_logs ENABLE ROW LEVEL SECURITY;

-- ✅ 최적화된 서브쿼리 방향
CREATE POLICY "users_own_server_logs" ON server_action_logs
  FOR SELECT TO authenticated
  USING (
    server_id IN (
      SELECT id FROM game_servers
      WHERE user_id = (select auth.uid())
    )
  );

CREATE POLICY "admin_full_access_logs" ON server_action_logs
  FOR ALL TO authenticated
  USING ((select auth.jwt()->'app_metadata'->>'role') = 'admin');
```

**games, hosting_plans (공개 읽기 + admin 쓰기):**
```sql
ALTER TABLE games ENABLE ROW LEVEL SECURITY;

CREATE POLICY "public_read_games" ON games
  FOR SELECT TO authenticated
  USING (is_active = true);

CREATE POLICY "admin_manage_games" ON games
  FOR ALL TO authenticated
  USING ((select auth.jwt()->'app_metadata'->>'role') = 'admin');
```

### 정책 컬럼 인덱싱 (필수)

```sql
-- 정책에서 사용하는 컬럼은 반드시 인덱스
CREATE INDEX ix_game_servers_user_id ON game_servers (user_id);
CREATE INDEX ix_server_action_logs_server_id ON server_action_logs (server_id);
CREATE INDEX ix_subscriptions_user_id ON subscriptions (user_id);
```

### Security Definer 함수 (복잡한 권한 체크)

```sql
CREATE OR REPLACE FUNCTION public.is_server_owner(p_server_id uuid)
RETURNS boolean
LANGUAGE sql
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT EXISTS(
    SELECT 1 FROM public.game_servers
    WHERE id = p_server_id
      AND user_id = (SELECT auth.uid())
  );
$$;
```

### RLS 디버깅

```sql
-- RLS 꺼진 테이블 찾기
SELECT tablename FROM pg_tables t
JOIN pg_class c ON c.relname = t.tablename
WHERE t.schemaname = 'public' AND c.relrowsecurity = false;

-- 테이블 정책 확인
SELECT * FROM pg_policies WHERE tablename = 'game_servers';
```

---

## 5. 타입 생성

```bash
# 로컬 Supabase에서 생성
supabase gen types typescript --local > packages/db/src/types/supabase.ts

# 원격 프로젝트에서 생성
supabase gen types typescript --project-id <project-id> > packages/db/src/types/supabase.ts
```

스키마 변경 시 반드시 재생성. CI에 타입 생성 + 커밋 체크 추가 권장.

---

## 6. 마이그레이션

```bash
supabase init
supabase start

# 마이그레이션 생성
supabase migration new add-game-servers-table

# 적용
supabase db push           # 로컬
supabase db push --linked  # 원격 (주의)

# 타입 재생성
supabase gen types typescript --local > packages/db/src/types/supabase.ts
```

- `supabase/migrations/` 는 git 커밋
- 프로덕션 마이그레이션은 항상 staging 먼저 검증

---

## 7. 체크리스트

### 보안
- [ ] public 스키마 모든 테이블 RLS 활성화
- [ ] SELECT/INSERT/UPDATE/DELETE 정책 각각 설정
- [ ] `to authenticated` 명시 (anon 차단)
- [ ] `SUPABASE_SERVICE_ROLE_KEY`, `PTERODACTYL_API_KEY` 서버 전용, git 제외
- [ ] `user_metadata`로 인가 처리 금지 → `app_metadata` 사용
- [ ] 서버에서 `getSession()` 사용 금지 → `getUser()` 사용

### 성능
- [ ] 정책 컬럼 인덱스 추가 (user_id, server_id 등)
- [ ] `auth.uid()` → `(select auth.uid())` 래핑
- [ ] 복잡한 권한 체크는 security definer 함수로 분리

### 개발
- [ ] 스키마 변경 시 타입 재생성
- [ ] 미들웨어에서 `getUser()` 호출 (토큰 갱신)
- [ ] Admin Client는 `@supabase/supabase-js`의 `createClient` 사용 (`@supabase/ssr` 아님)
