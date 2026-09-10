---
name: supabase-guide
description: >
  Next.js + Supabase 실무 가이드 — @supabase/ssr 클라이언트 3종(browser/server/admin),
  인증(getUser vs getSession, app_metadata role, middleware 토큰 갱신), RLS 정책·성능,
  publishable/secret 신규 API 키, 타입 생성, 마이그레이션.
  Supabase, RLS, Row Level Security, @supabase/ssr, publishable key, secret key, middleware 관련 작업 시 사용.
  DBMS 불문 스키마 원칙(db-architecture)과 React Query 캐싱(react-query-guide)은 다루지 않는다.
---

# Supabase 가이드

> 원칙: DB는 항상 RLS로 보호한다. 클라이언트는 절대 신뢰하지 않는다.

---

## 1. 클라이언트 설정

### 패키지

```bash
npm install @supabase/supabase-js @supabase/ssr   # pnpm/yarn/bun 등 프로젝트가 쓰는 패키지 매니저로 실행
```

### Browser Client (Client Component용)

```ts
// src/lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr'
import type { Database } from '@/types/database'

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
  )
}
```

### Server Client (Server Component / Server Action / Route Handler용)

```ts
// src/lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import type { Database } from '@/types/database'

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
// src/lib/supabase/admin.ts
import { createClient } from '@supabase/supabase-js'  // ← @supabase/ssr 아님
import type { Database } from '@/types/database'

export function createSupabaseAdminClient() {
  return createClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SECRET_KEY!,
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
// src/lib/supabase/server.ts
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
// src/middleware.ts
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

> 원칙: **레거시 `anon` / `service_role` (JWT) 키는 사용하지 않는다.**
> 항상 신규 **Publishable / Secret** 키를 사용한다.

### 레거시 키 마이그레이션 → `references/key-migration.md`

레거시 `anon`/`service_role`(JWT) 키에서 신규 키로 교체할 때만 읽는다 — 키 비교표, 발급/교체 5단계 절차, 유출 대응 포함. 신규 프로젝트는 처음부터 Publishable/Secret 키만 쓰면 되므로 읽을 필요 없다.

### `anon` 키 vs `anon` 역할 — 혼동 주의

- **`anon` 키**(레거시 API 키) → 폐기 대상. `sb_publishable_...` 로 교체.
- **`anon` 역할**(Postgres role) → 그대로 존재. Publishable 키로 접근하되 로그인 세션이 없으면 여전히 `anon` 역할로 매핑된다.
- 따라서 키를 교체해도 RLS 정책의 `TO authenticated` / `TO anon` 구분은 그대로 유지된다.

### 환경변수

| 키 | 용도 | 노출 |
|----|------|------|
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Browser/SSR 클라이언트 | 공개 OK (RLS가 보호) |
| `SUPABASE_SECRET_KEY` | Admin 작업, RLS 우회 | 절대 클라이언트 노출 금지 |

```bash
# .env.local (git 제외)
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_xxx
SUPABASE_SECRET_KEY=sb_secret_xxx
```

```bash
# ❌ 레거시 — 신규/기존 코드 모두에서 제거 대상
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOi...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOi...
```

- Secret 키는 여전히 RLS를 우회한다 → `NEXT_PUBLIC_` 접두사를 붙이는 순간 전 데이터가 공개된다. 절대 금지.
- Publishable 키는 공개돼도 안전하지만, 그 전제는 **모든 테이블에 RLS가 켜져 있다는 것**이다 (→ 4장).

---

## 4. RLS (Row Level Security)

> public 스키마의 모든 테이블에 RLS를 활성화한다. 예외 없음.

### 기본 설정

```sql
-- 테이블 생성 후 항상 실행
ALTER TABLE public.resources ENABLE ROW LEVEL SECURITY;
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

### RLS 패턴 (소유자 기반)

**resources (소유자 기반 — 핵심):**
```sql
ALTER TABLE resources ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users_select_own_resources" ON resources
  FOR SELECT TO authenticated
  USING ((select auth.uid()) = user_id);

CREATE POLICY "users_insert_own_resources" ON resources
  FOR INSERT TO authenticated
  WITH CHECK ((select auth.uid()) = user_id);

CREATE POLICY "users_update_own_resources" ON resources
  FOR UPDATE TO authenticated
  USING ((select auth.uid()) = user_id)
  WITH CHECK ((select auth.uid()) = user_id);

CREATE POLICY "users_delete_own_resources" ON resources
  FOR DELETE TO authenticated
  USING ((select auth.uid()) = user_id);

-- admin: app_metadata.role 기반 (user_metadata 금지)
CREATE POLICY "admin_full_access_resources" ON resources
  FOR ALL TO authenticated
  USING ((select auth.jwt()->'app_metadata'->>'role') = 'admin');
```

**resource_logs (부모 소유자 기반):**
```sql
ALTER TABLE resource_logs ENABLE ROW LEVEL SECURITY;

-- ✅ 최적화된 서브쿼리 방향
CREATE POLICY "users_own_resource_logs" ON resource_logs
  FOR SELECT TO authenticated
  USING (
    resource_id IN (
      SELECT id FROM resources
      WHERE user_id = (select auth.uid())
    )
  );

CREATE POLICY "admin_full_access_logs" ON resource_logs
  FOR ALL TO authenticated
  USING ((select auth.jwt()->'app_metadata'->>'role') = 'admin');
```

**catalog_items (공개 읽기 + admin 쓰기):**
```sql
ALTER TABLE catalog_items ENABLE ROW LEVEL SECURITY;

CREATE POLICY "public_read_catalog" ON catalog_items
  FOR SELECT TO authenticated
  USING (is_active = true);

CREATE POLICY "admin_manage_catalog" ON catalog_items
  FOR ALL TO authenticated
  USING ((select auth.jwt()->'app_metadata'->>'role') = 'admin');
```

### 정책 컬럼 인덱싱 (필수)

```sql
-- 정책의 USING/WITH CHECK에 등장하는 컬럼은 반드시 인덱스
CREATE INDEX ix_resources_user_id ON resources (user_id);          -- 소유자 기반 정책
CREATE INDEX ix_resource_logs_resource_id ON resource_logs (resource_id);  -- 부모 참조 정책
```

### Security Definer 함수 (복잡한 권한 체크)

```sql
CREATE OR REPLACE FUNCTION public.is_resource_owner(p_resource_id uuid)
RETURNS boolean
LANGUAGE sql
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT EXISTS(
    SELECT 1 FROM public.resources
    WHERE id = p_resource_id
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
SELECT * FROM pg_policies WHERE tablename = 'resources';
```

---

## 5. 타입 생성

```bash
# 로컬 Supabase에서 생성
supabase gen types typescript --local > src/types/database.ts

# 원격 프로젝트에서 생성
supabase gen types typescript --project-id <project-id> > src/types/database.ts
```

스키마 변경 시 반드시 재생성. CI에 타입 생성 + 커밋 체크 추가 권장.

---

## 6. 마이그레이션

```bash
supabase init
supabase start

# 마이그레이션 생성
supabase migration new add-resources-table

# 적용
supabase db push           # 로컬
supabase db push --linked  # 원격 (주의)

# 타입 재생성
supabase gen types typescript --local > src/types/database.ts
```

- `supabase/migrations/` 는 git 커밋
- 프로덕션 마이그레이션은 항상 staging 먼저 검증

---

## 7. 체크리스트

### 보안
- [ ] public 스키마 모든 테이블 RLS 활성화
- [ ] SELECT/INSERT/UPDATE/DELETE 정책 각각 설정
- [ ] `to authenticated` 명시 (비로그인 `anon` **역할** 차단 — 레거시 anon 키와 다른 개념)
- [ ] 레거시 `anon` / `service_role` (JWT) 키 미사용 → Publishable / Secret 키만 사용
- [ ] `SUPABASE_SECRET_KEY` 서버 전용, git 제외 (`NEXT_PUBLIC_` 접두사 금지)
- [ ] 교체 완료 후 대시보드에서 레거시 키 disable (자동 폐기되지 않음)
- [ ] `user_metadata`로 인가 처리 금지 → `app_metadata` 사용
- [ ] 서버에서 `getSession()` 사용 금지 → `getUser()` 사용

### 성능
- [ ] 정책 컬럼 인덱스 추가 (user_id, resource_id 등)
- [ ] `auth.uid()` → `(select auth.uid())` 래핑
- [ ] 복잡한 권한 체크는 security definer 함수로 분리

### 개발
- [ ] 스키마 변경 시 타입 재생성
- [ ] 미들웨어에서 `getUser()` 호출 (토큰 갱신)
- [ ] Admin Client는 `@supabase/supabase-js`의 `createClient` 사용 (`@supabase/ssr` 아님)


---

## 8. Gotchas (운영하며 축적)

- RLS의 여러 permissive 정책은 **OR로 결합**된다 — admin `FOR ALL` 정책이 있으면 소유자 정책을 아무리 좁혀도 admin은 항상 통과한다. 교집합(AND)이 필요하면 `AS RESTRICTIVE` 정책을 쓴다.
- Server Component에서는 쿠키를 쓸 수 없어 `setAll`이 실패한다(server.ts의 try/catch가 그 이유) — 토큰 갱신은 전적으로 middleware 책임이다. middleware `matcher`에서 경로가 빠지면 그 경로의 세션이 조용히 만료된다.
- `supabase gen types`가 생성하는 **뷰(view) 타입은 모든 컬럼이 nullable**이다 — 뷰 기반 조회는 non-null 보정(타입 가드/변환 함수)이 필요하다.

> 운영 중 새로 발견한 함정은 이 섹션에 계속 축적한다.
