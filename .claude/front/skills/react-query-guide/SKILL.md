---
name: react-query-guide
description: >
  GamePot React Query 데이터 레이어 SSoT.
  도메인 디렉토리 구조, query keys/options/prefetch, useMutation + invalidateQueries/낙관적 업데이트,
  Server Actions(ActionResult), Supabase Realtime + setQueryData, 캐시 레이어·상태 소유권, 마이그레이션 완료 기준.
  React Query, query key, query options, prefetch, mutation, invalidate, 낙관적 업데이트, Realtime,
  상태관리(nuqs/Zustand 판단), 신규 feature 데이터 레이어 추가 시 참조.
---

# React Query — 데이터 레이어 패턴

> UI/컴포넌트/Tailwind·shadcn 구현 → `server-ui` 스킬 참조
> Supabase 클라이언트/RLS/인증 → `supabase-guide` 스킬 참조
> Pterodactyl API 패턴 → `pterodactyl-domain` 스킬 참조

---

## 1. 도메인 디렉토리 구조 (필수)

```
apps/admin/src/domain/{feature}/
├── index.ts                    — public API (전체 re-export)
├── types/
│   ├── index.ts
│   ├── dto.ts                  — API 요청/응답 타입
│   └── entity.ts               — 도메인 엔티티
├── queries/
│   ├── index.ts
│   ├── {feature}.query-keys.ts
│   ├── {feature}.query-options.ts
│   └── {feature}.prefetch.ts
├── hooks/
│   ├── index.ts
│   └── {feature}.hooks.ts
├── actions/
│   └── {feature}.actions.ts    — 'use server'
└── validations/
    └── {feature}.validations.ts — Zod 스키마 (Server Action + Form 공유)
```

레이어 규칙:
- 크로스 도메인 deep import 금지: `@/domain/servers/queries/...` ❌ → `@/domain/servers` ✅
- `actions/`, `service/`, `validations/`에는 `index.ts` 불필요 (루트에서 직접 export)
- 검증(Zod) 스키마는 해당 도메인의 `validations/{feature}.validations.ts`에 둔다. 앱 루트 `lib/`는 여러 도메인이 실제로 공유하는 것만 남긴다(예: 공통 ActionResult 봉투, 인증 헬퍼, 공용 포맷터).

---

## 1-1. 복잡도별 적용 범위 (풀세트 여부 판단)

모든 feature에 `queries/hooks/actions/validations` 풀세트를 기계적으로 만들지 않는다. 신규 feature마다 아래 기준으로 먼저 적용 범위를 판단한다:

```
신규 기능 추가 → 읽기 전용인가?
                     ↓ Yes: queries/ (keys+options+prefetch) + hooks/(useQuery)만. actions/validations 생략
                     ↓ No: 서버 상태 변경(mutation)이 있는가?
                              ↓ 단순 CRUD, 인증/소유권 등 비즈니스 룰 없음: + actions/(ActionResult) + validations/(Zod)
                              ↓ 인증·소유권·쿼터 등 비즈니스 룰 있음, 또는 재사용 필요: 위 전체 + 5-1절(순수 로직 분리) + 4절(낙관적 업데이트) 적용
```

예:
- `plans`(목록 조회만) → `queries/` + `hooks/`. `actions/`/`validations/` 불필요.
- `games`(CRUD, 특별한 룰 없음) → 풀세트, 5절 기본 패턴으로 충분.
- `servers`(전원 제어 = 상태 규칙 + Realtime) → 풀세트 + 5-1절 + 4절 낙관적 업데이트 필수.

빈 `actions/`·`validations/` 폴더를 미리 만들어두는 것은 과설계다 — 쓰기 기능이 실제로 추가될 때 그 시점에 만든다.

---

## 2. Query Keys & Options

```ts
// {feature}.query-keys.ts
export const serverQueryKeys = {
  all: ['servers'] as const,
  list: () => [...serverQueryKeys.all, 'list'] as const,
  detail: (id: string) => [...serverQueryKeys.all, 'detail', id] as const,
}

// {feature}.query-options.ts
export const serverQueryOptions = {
  list: () => ({
    queryKey: serverQueryKeys.list(),
    queryFn: () => fetchServers(),
  }),
  detail: (id: string) => ({
    queryKey: serverQueryKeys.detail(id),
    queryFn: () => fetchServer(id),
    staleTime: 30_000,
  }),
}
```

금지:
```ts
// ❌ 인라인 queryKey
useQuery({ queryKey: ['servers'], queryFn: fetchServers })

// ✅ options 팩토리 사용
useQuery(serverQueryOptions.list())
```

---

## 3. Prefetch (Server Component)

```ts
// {feature}.prefetch.ts
export const serverPrefetch = {
  list: () => async (queryClient: QueryClient) => {
    await queryClient.prefetchQuery(serverQueryOptions.list())
  },
}

// lib/react-query/prefetch.ts
export async function runPrefetch(...prefetchers: Array<(qc: QueryClient) => Promise<void>>) {
  const qc = getQueryClient()
  await Promise.all(prefetchers.map(fn => fn(qc)))
  return dehydrate(qc)
}

// app/(dashboard)/servers/page.tsx — Server Component
export default async function ServersPage() {
  const state = await runPrefetch(serverPrefetch.list())
  return (
    <HydrationBoundary state={state}>
      <ServerTableClient />
    </HydrationBoundary>
  )
}
```

---

## 4. Mutation + Invalidation

```ts
// {feature}.hooks.ts

// 단순 CRUD mutation
export function useCreateGame() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: createGame,             // Server Action
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: gameQueryKeys.list() })
      toast.success('게임이 추가됐습니다.')
    },
    onError: (err) => toast.error(err instanceof Error ? err.message : '오류'),
  })
}

// 낙관적 업데이트 (전원 제어)
export function useServerPower(serverId: string) {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (action: 'start' | 'stop' | 'restart') =>
      executeServerPower(serverId, action),
    onMutate: async (action) => {
      await queryClient.cancelQueries({ queryKey: serverQueryKeys.detail(serverId) })
      const previous = queryClient.getQueryData(serverQueryKeys.detail(serverId))
      queryClient.setQueryData(serverQueryKeys.detail(serverId), (old: GameServer) => ({
        ...old,
        status: action === 'start' ? 'starting' : 'stopping',
      }))
      return { previous }
    },
    onError: (_err, _action, context) => {
      queryClient.setQueryData(serverQueryKeys.detail(serverId), context?.previous)
      toast.error('서버 제어 실패')
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: serverQueryKeys.detail(serverId) })
    },
  })
}
```

금지:
```ts
// ❌ router.refresh()로 갱신
await fetch('/api/games', { method: 'POST' })
router.refresh()

// ❌ useEffect에서 fetch
useEffect(() => { fetch('/api/servers').then(...) }, [])
```

---

## 5. Server Actions (ActionResult 패턴)

```ts
// {feature}.actions.ts
'use server'

type ActionResult<T = void> =
  | { success: true; data: T }
  | { success: false; error: string }

export async function createGame(input: CreateGameInput): Promise<ActionResult<Game>> {
  const supabase = await createSupabaseServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('로그인이 필요합니다.')  // 인증 에러 → throw

  try {
    const { data, error } = await supabase.from('games').insert(input).select().single()
    if (error) return { success: false, error: '게임 추가 실패' }
    return { success: true, data: toGame(data) }
  } catch {
    return { success: false, error: '서버 오류가 발생했습니다.' }
  }
}
```

규칙:
- 인증 에러 → `throw new Error(...)` (React Query가 error state 처리)
- 비즈니스 에러 → `ActionResult.error` (사용자 친화 메시지)
- 원본 Error 객체 클라이언트 노출 ❌

---

## 5-1. 순수 로직 분리 (재사용성·테스트 용이성)

1-1절 🔴 tier(비즈니스 룰 있음) 또는 API route·cron job·유닛 테스트에서 재사용이 실제로 필요한 액션은, 파일 상단 `'use server'` 대신 **함수 단위 `'use server'`**로 좁혀서 순수 로직과 Server Action 진입점을 분리한다.

```ts
// {feature}.actions.ts

// 순수 로직 — 'use server' 없음. API route/cron/유닛 테스트에서 직접 import해 재사용 가능
export async function createGame(input: CreateGameInput): Promise<ActionResult<Game>> {
  const supabase = await createSupabaseServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('로그인이 필요합니다.')

  try {
    const { data, error } = await supabase.from('games').insert(input).select().single()
    if (error) return { success: false, error: '게임 추가 실패' }
    return { success: true, data: toGame(data) }
  } catch {
    return { success: false, error: '서버 오류가 발생했습니다.' }
  }
}

// Server Action 진입점 — 폼/useMutation이 호출하는 얇은 wrapper만 'use server'
export async function createGameAction(input: CreateGameInput) {
  'use server'
  return createGame(input)
}
```

- `hooks/`의 `mutationFn`은 wrapper(`createGameAction`)를 호출한다. 순수 함수(`createGame`)를 클라이언트에서 직접 참조하지 않는다.
- 단순 CRUD·재사용 계획 없는 액션까지 전부 이렇게 분리하지 않는다 — 5절 기본 패턴(파일 상단 `'use server'`)으로 충분하면 그대로 둔다.

---

## 6. Supabase Realtime + React Query

```ts
// hooks/use-server-realtime.ts
export function useServerRealtime(serverId: string) {
  const queryClient = useQueryClient()
  useEffect(() => {
    const sub = supabase
      .channel(`server:${serverId}`)
      .on('postgres_changes',
        { event: 'UPDATE', schema: 'public', table: 'game_servers', filter: `id=eq.${serverId}` },
        (payload) => {
          queryClient.setQueryData(serverQueryKeys.detail(serverId), toGameServer(payload.new))
          queryClient.invalidateQueries({ queryKey: serverQueryKeys.list() })
        }
      )
      .subscribe()
    return () => { supabase.removeChannel(sub) }   // cleanup 필수
  }, [serverId, queryClient])
}
```

---

## 7. 캐시 레이어 요약

| 레이어 | 역할 |
|--------|------|
| Next.js fetch cache | ISR/revalidateTag. 클라이언트 freshness 용도 ❌ |
| React cache() | 단일 RSC 렌더 내 중복 fetch 제거 |
| React Query cache | UI 상태 단일 출처. 모든 UI는 여기서만 읽음 |

---

## 8. 상태 소유권 (어디에 둘 것인가)

| 상태 유형 | 도구 | 비고 |
|----------|------|------|
| 서버 데이터 | **React Query** | 캐싱·invalidate·낙관적 업데이트. 모든 서버 상태의 단일 출처 |
| URL 상태 (필터·정렬·페이지네이션·탭) | **nuqs** (현재 미설치) | 목록 페이지(servers·games·plans·users)의 필터/페이지는 nuqs로 URL에 동기화 — 공유·새로고침·뒤로가기 보존. **도입 시 적용하는 규칙** |
| 로컬 UI 상태 | `useState`/`useReducer` | 한 컴포넌트 내부(모달 open, 입력값 등) |
| 전역 클라이언트 상태 | **현재 미도입 (Zustand 보류)** | 아래 기준 충족 전까지 도입 금지 |

**판단 흐름:** 서버 데이터? → React Query / URL에서 복원해야 하나(필터·페이지)? → nuqs / 한 컴포넌트 내부? → useState.

**nuqs 규칙** (※ `nuqs` 미설치 — 도입 시 적용)**:**
- 목록 필터·검색·정렬·페이지·탭은 `useState` 대신 `useQueryState`/`useQueryStates`로 URL에 둔다 (`router.refresh()` 금지 규칙과 무관 — URL 변경은 정상).
- nuqs 값을 React Query `queryKey`에 포함해 필터 변경 시 자동 리페치 (인라인 queryKey 금지 규칙대로 query options 팩토리에 파라미터로 전달).
- Server Component에서 초기값은 `searchParams`로 읽어 prefetch에 반영 → 클라이언트는 nuqs로 이어받는다.
- **검색 입력은 디바운스**(약 300ms) 후 URL/`queryKey`에 반영 — 매 키 입력마다 리페치 금지. nuqs `throttleMs` 또는 입력 로컬 `useState` + 디바운스 후 `setQueryState`.
- **긴 목록은 페이지네이션이 1차 해법.** 가상화(`@tanstack/react-virtual`)는 페이지네이션으로도 부족한 1000+ 단일 뷰가 실제로 필요할 때만 도입(현재 미설치·보류).

**Zustand는 다음 중 하나가 실제로 생기기 전까지 도입하지 않는다:**
- 부모-자식이 아닌, 멀리 떨어진 컴포넌트 간 공유되는 **순수 클라이언트** 상태 (예: 서버 목록 일괄 선택 → 툴바와 테이블이 선택셋 공유)
- 라우트를 가로지르는 멀티스텝 위저드 상태 (URL로 표현하기 부적합한 경우)

서버 상태(React Query)·URL 상태(nuqs)·로컬 상태(useState)로 해결되면 Zustand를 추가하지 않는다. Realtime은 이미 `setQueryData`로 React Query에 흡수되므로 Zustand 대상이 아니다.

---

## 9. 구현 순서

0. 적용 범위 판단(1-1절) — 이 feature에 어떤 하위 폴더가 실제로 필요한지 먼저 결정
1. `lib/react-query/` — `getQueryClient`, `runPrefetch`, `QueryProvider`
2. `domain/{feature}/queries/` — keys → options → prefetch
3. `domain/{feature}/actions/` — Server Actions (ActionResult 패턴)
4. `domain/{feature}/hooks/` — `useQuery` / `useMutation` 래퍼
5. `domain/{feature}/index.ts` — public API re-export
6. Page (Server Component) — `runPrefetch` + `HydrationBoundary`
7. Client Components — hooks 사용, UI만 담당 (→ UI 구현은 `server-ui` 스킬)

---

## 10. 마이그레이션 완료 — 패턴 레퍼런스

> ✅ GamePot 코드는 **React Query 전환 완료** 상태다 (`@tanstack/react-query` 설치됨, `domain/{servers,games,plans}` 구축됨). 아래는 신규 feature 작성 시 따라야 할 **완료 사례**다.
> UI 컴포넌트는 `@workspace/ui` 패키지에서 import한다 (예: `@workspace/ui/components/button`).

### 완료된 화면 (참고 레퍼런스)

| 파일 | 적용 패턴 |
|------|----------|
| `(dashboard)/games/{page,games-client}.tsx` | `runPrefetch`+`HydrationBoundary` → `useGames()` / `useCreateGame()` / `useUpdateGame()` |
| `(dashboard)/plans/{page,plans-client}.tsx` | `usePlans()` / `useCreatePlan()` / `useUpdatePlan()` |
| `(dashboard)/servers/page.tsx` + `components/servers/server-table.tsx` | `serverPrefetch.list(filters)` → `useServers()` + `useServerRealtime()` (Supabase Realtime) |

데이터용 `router.refresh()`는 코드에서 제거됨 (잔존 2곳은 로그인/로그아웃 세션 갱신 — 정상).

### 신규 feature 추가 절차 (feature 단위)

```
0. 적용 범위 판단(1-1절) — 읽기 전용이면 queries/hooks만, 쓰기 있으면 actions/validations 추가
1. domain/{feature}/ 레이어 생성 (판단된 범위만큼: query-keys/options/prefetch/hooks/actions)
2. page.tsx (Server Component): runPrefetch + HydrationBoundary
3. {feature}-client.tsx: useQuery(...QueryOptions.list()) — initialData props 없음
4. mutation: useMutation + invalidateQueries (router.refresh 금지)
5. 실시간 필요 시: use-server-realtime.ts 패턴(Supabase Realtime → setQueriesData) 참고
```

> **미구축:** `users`는 페이지만 있고 도메인 레이어가 없다 — 위 절차의 1순위 적용 대상.
> code-reviewer는 모든 코드에 RQ 규칙을 적용한다 (legacy 예외 없음 — 마이그레이션이 끝났으므로).
