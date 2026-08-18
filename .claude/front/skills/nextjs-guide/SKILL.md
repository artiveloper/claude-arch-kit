---
name: nextjs-guide
description: >
  GamePot Next.js App Router 메커니즘 SSoT.
  파일 컨벤션(page/layout/loading/error), Server vs Client Component 경계,
  route group, 라우트 레벨 로딩(loading.tsx + Suspense 스트리밍), prefetch/HydrationBoundary 배치.
  Next.js, App Router, Server Component, Client Component, 'use client', loading.tsx, Suspense,
  route group, RSC, 스트리밍 관련 작업 시 참조.
---

# Next.js — App Router 메커니즘

> UI 스타일링(모바일 퍼스트·shadcn 컴포넌트·스켈레톤 컴포넌트) → `shadcn-ui` 스킬 참조
> 데이터 레이어(React Query·query options·prefetch·mutation) → `react-query-guide` 스킬 참조

---

## 1. App Router 파일 컨벤션

```
apps/admin/src/app/(dashboard)/
├── layout.tsx                  — 공유 레이아웃 (Server Component)
├── loading.tsx                 — 대시보드 홈 라우트 로딩
├── servers/
│   ├── page.tsx                — Server Component (prefetch + HydrationBoundary)
│   └── loading.tsx             — 세그먼트 로딩
├── games/{page,loading}.tsx
├── plans/{page,loading}.tsx
└── users/{page,loading}.tsx
```

- `(dashboard)` 같은 **route group**은 URL에 노출되지 않는 그룹 — 공유 layout 적용용.
- 특수 파일: `page`(라우트 진입), `layout`(공유 껍데기), `loading`(세그먼트 로딩 UI), `error`(에러 바운더리).

---

## 2. Server Component vs Client Component

기본은 **Server Component**. `'use client'`는 아래가 필요할 때만 붙인다.
- 브라우저 상호작용/이벤트 핸들러, `useState`/`useEffect` 등 훅
- React Query 훅(`useQuery`/`useMutation`) 소비

경계 규칙:
- **데이터 fetch는 Server Component에서 prefetch** → `HydrationBoundary`로 Client에 전달 (→ `react-query-guide` 3절).
- **UI 상호작용·RQ 훅 소비는 Client Component**가 담당 (컴포넌트 스타일링은 → `shadcn-ui`).
- UI 컴포넌트에서 `fetch`/`useEffect`로 직접 패칭 금지 → domain hooks 사용 (→ `react-query-guide`).

```tsx
// app/(dashboard)/servers/page.tsx — Server Component
export default async function ServersPage() {
  const state = await runPrefetch(serverPrefetch.list())   // ← react-query-guide
  return (
    <HydrationBoundary state={state}>
      <ServerTableClient />                                 {/* 'use client' */}
    </HydrationBoundary>
  )
}
```

---

## 3. 라우트 레벨 로딩 (loading.tsx)

라우트 레벨 로딩(서버 prefetch 중)은 **`loading.tsx`** 로 처리한다. Next.js가 해당 세그먼트를 자동으로 `<Suspense>`로 감싸므로 `runPrefetch` 완료 전까지 로딩 UI가 표시된다(스트리밍).

```tsx
// app/(dashboard)/servers/loading.tsx
import { Skeleton } from '@workspace/ui/components/skeleton';
import { TableSkeleton, PageHeaderSkeleton } from '@/components/ui/skeletons';  // ← shadcn-ui

export default function ServersLoading() {
  return (
    <div>
      <PageHeaderSkeleton />
      <div className="space-y-4">
        <div className="flex gap-3">
          <Skeleton className="h-9 w-28 rounded-md" />
        </div>
        <TableSkeleton rows={6} cols={7} />
      </div>
    </div>
  );
}
```

**역할 구분:**
- `loading.tsx` — **라우트 전환 시**(서버 prefetch 대기) 세그먼트 로딩. Next.js Suspense가 처리.
- 클라이언트 `isLoading` 분기 — RQ 캐시가 비어 클라이언트에서 fetch할 때. (→ `shadcn-ui` 리스트 컴포넌트)
- 두 경우 모두 로딩 UI는 스켈레톤을 쓰고 "불러오는 중..." 텍스트는 금지 (원칙 → `design-system`, 스켈레톤 컴포넌트 → `shadcn-ui`).

---

## 4. 데이터/캐시 경계 (요약)

App Router에는 여러 캐시 레이어가 있으나, **클라이언트 freshness의 단일 출처는 React Query**다. Next.js fetch cache는 ISR/`revalidateTag` 용도이지 UI 상태 소스가 아니다. 상세 표·상태 소유권은 → `react-query-guide` 7·8절.
