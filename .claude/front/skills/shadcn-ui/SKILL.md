---
name: shadcn-ui
description: >
  GamePot Tailwind/shadcn UI 구현 패턴 SSoT.
  모바일 퍼스트 className 규칙, shadcn Sidebar 레이아웃, 컴포넌트 패턴 —
  전원 제어 버튼(Button), 상태 Badge, KST 날짜·시간 포맷, 리스트 Client Component, 공유 스켈레톤 컴포넌트.
  Next.js App Router 메커니즘은 nextjs-guide, 데이터 레이어는 react-query-guide 참조.
  Tailwind, shadcn, className, Sidebar, Button, Badge, Skeleton, 반응형, UI 컴포넌트 작업 시 참조.
---

# shadcn/Tailwind — UI 구현 패턴

> Next.js App Router 메커니즘(Server/Client 경계·loading.tsx·route group) → `nextjs-guide` 스킬 참조
> 데이터 레이어(React Query·query keys/options·prefetch·mutation·Realtime) → `react-query-guide` 스킬 참조
> Supabase 클라이언트/RLS/인증 → `supabase-guide` 스킬 참조
> 라이브러리 불문 UI 원칙(모바일 퍼스트·터치 타겟·반응형·로딩/빈/에러 상태·상태 색상 일관성) → `design-system` 스킬 참조. 이 스킬은 그 원칙들의 **Tailwind/shadcn 구현**만 다룬다.

---

## 0. UI 1원칙: 모바일 퍼스트

> 모바일 퍼스트 원칙 자체(왜·판단 기준)는 `design-system` 스킬 참조. 이 섹션은 그 원칙의 **Tailwind/shadcn 구현**만 다루며, 아래 모든 섹션에 우선한다.

### Tailwind 사용 규칙

```tsx
// ✅ 모바일 기본, 데스크탑 확장
<div className="p-4 md:p-6">
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
<div className="text-sm md:text-base">

// ❌ 데스크탑 기준으로 먼저 설계
<div className="px-8 py-10">
<div className="grid grid-cols-4 gap-6">
```

### 레이아웃 — shadcn Sidebar

dashboard layout은 `SidebarProvider` + `SidebarInset` + `AppSidebar` 조합을 사용한다. 직접 `<aside>` 또는 고정 `w-60`을 사용하지 않는다.

```tsx
// app/(dashboard)/layout.tsx  ← layout.tsx 파일 컨벤션은 nextjs-guide
import { SidebarProvider, SidebarInset, SidebarTrigger } from '@workspace/ui/components/sidebar'
import { AppSidebar } from '@/components/app-sidebar'

export default function DashboardLayout({ children }) {
  return (
    <SidebarProvider>
      <AppSidebar />
      <SidebarInset>
        <header className="flex h-14 items-center gap-2 border-b px-4">
          <SidebarTrigger className="-ml-1" />    {/* 모바일 햄버거 버튼 */}
        </header>
        <main className="flex-1 overflow-y-auto bg-muted/40 p-4 md:p-6">
          {children}
        </main>
      </SidebarInset>
    </SidebarProvider>
  )
}
```

- 모바일: `SidebarTrigger` 클릭 시 Sheet 오버레이로 사이드바 열림 (`collapsible="offcanvas"`)
- 데스크탑: 사이드바가 고정 패널로 표시됨

### 터치 타겟

터치 타겟 최소 44px 원칙(→ `design-system`)의 Tailwind 구현:

```tsx
// ✅ 충분한 터치 타겟
<Button className="min-h-11 px-4">액션</Button>

// ❌ 너무 작은 타겟
<button className="h-6 px-2">액션</button>
```

### 테이블 반응형

좁은 화면 테이블 처리(카드/수평 스크롤) 원칙(→ `design-system`)의 Tailwind 구현:

```tsx
// 수평 스크롤 (우선 처리)
<div className="overflow-x-auto">
  <table className="min-w-full">...</table>
</div>
```

### 금지 패턴

```tsx
// ❌ overflow-hidden으로 모바일 스크롤 차단
<div className="h-screen overflow-hidden">

// ❌ 고정 사이드바 너비 (항상 표시)
<aside className="w-60 flex-shrink-0">

// ❌ 고정 여백 (모바일에서 너무 좁음)
<div className="px-6 py-8">  // → p-4 md:p-6 사용
```

---

## 1. 컴포넌트 패턴

> 아래 컴포넌트가 사용하는 `useServerPower`/`useServers` 등 React Query 훅의 정의는 `react-query-guide` 스킬 참조. Server/Client 경계·`'use client'`는 `nextjs-guide` 참조. 이 섹션은 그 훅을 소비하는 **UI 스타일링**만 다룬다.

### 전원 제어 버튼

```tsx
// components/power-controls.tsx
'use client'
export function PowerControls({ serverId, status }: { serverId: string; status: ServerStatus }) {
  const { mutate, isPending } = useServerPower(serverId)
  const isTransitioning = status === 'starting' || status === 'stopping'

  return (
    <div className="flex gap-2">
      <Button onClick={() => mutate('start')}
        disabled={status !== 'stopped' || isPending || isTransitioning}>시작</Button>
      <Button variant="destructive" onClick={() => mutate('stop')}
        disabled={status !== 'running' || isPending || isTransitioning}>중지</Button>
      <Button variant="outline" onClick={() => mutate('restart')}
        disabled={status !== 'running' || isPending || isTransitioning}>재시작</Button>
    </div>
  )
}
```

### 상태 Badge

> 상태 → 라벨/색상 중앙 매핑·색상 단독 의존 금지 원칙은 `design-system` 스킬 참조. 아래는 GamePot 상태값에 대한 구체 매핑.

```tsx
const statusConfig: Record<ServerStatus, { label: string; variant: string }> = {
  running:      { label: '실행 중', variant: 'success' },
  stopped:      { label: '중지됨', variant: 'secondary' },
  starting:     { label: '시작 중', variant: 'warning' },
  stopping:     { label: '중지 중', variant: 'warning' },
  provisioning: { label: '프로비저닝', variant: 'default' },
  suspended:    { label: '정지됨', variant: 'destructive' },
  error:        { label: '오류', variant: 'destructive' },
}
```

### 날짜·시간 포맷 (KST 필수)

서비스 대상이 한국이므로 모든 날짜·시간 표시는 `@/lib/format.ts` 유틸을 사용한다.

```ts
import { formatDate, formatDateTime } from '@/lib/format';

formatDate(server.createdAt)     // "2026. 06. 28." (KST)
formatDateTime(server.createdAt) // "2026. 06. 28. 오후 03:00" (KST)
```

- `new Date(x).toLocaleDateString('ko-KR')` — `timeZone` 누락 형태 금지
- `toLocaleString('ko-KR')` 숫자 포맷(금액 천단위)은 허용

### 리스트 Client Component

```tsx
'use client'
import { TableSkeleton } from '@/components/ui/skeletons'  // 공유 스켈레톤 컴포넌트

export function ServerTableClient() {
  const { data: servers, isLoading, error } = useServers()

  if (isLoading) return <TableSkeleton rows={6} cols={7} />  // 텍스트 "불러오는 중..." 금지
  if (error) return <p className="text-destructive">서버 목록을 불러올 수 없습니다.</p>
  if (!servers?.length) return <p className="text-muted-foreground">등록된 서버가 없습니다.</p>

  return <table>...</table>
}
```

> 클라이언트 `isLoading`(RQ 캐시 없음) vs 라우트 레벨 `loading.tsx`(prefetch 대기) 역할 구분 → `nextjs-guide`.

### 공유 스켈레톤 컴포넌트

로딩 UI는 화면마다 새로 만들지 말고 `components/ui/skeletons.tsx`의 공유 컴포넌트를 재사용한다. 라우트 레벨 로딩(`loading.tsx`)에서의 조합·배치는 → `nextjs-guide`.

```
apps/admin/src/components/ui/skeletons.tsx  ← 공유 스켈레톤 컴포넌트
```

| 컴포넌트 | 용도 |
|---------|------|
| `<TableSkeleton rows cols />` | 테이블 리스트 (rows/cols 조절) |
| `<CardGridSkeleton />` | 대시보드 통계 카드 4개 그리드 |
| `<PageHeaderSkeleton />` | 페이지 제목 + 설명 영역 |

> 로딩 분기에서 "불러오는 중..." 텍스트는 금지 → 위 스켈레톤 컴포넌트 사용.
