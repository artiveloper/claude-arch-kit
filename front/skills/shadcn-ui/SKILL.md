---
name: shadcn-ui
description: >
  GamePot Tailwind/shadcn UI 구현 패턴 SSoT. 프리미티브는 Base UI(@base-ui/react) 기준.
  모바일 퍼스트 className 규칙, Base UI 합성(render prop)·data-* 상태 스타일링, shadcn Sidebar 레이아웃,
  컴포넌트 패턴 — 전원 제어 버튼(Button), 상태 Badge, KST 날짜·시간 포맷, 리스트 Client Component, 공유 스켈레톤 컴포넌트.
  Next.js App Router 메커니즘은 nextjs-guide, 데이터 레이어는 react-query-guide 참조.
  Tailwind, shadcn, Base UI, render prop, className, Sidebar, Button, Badge, Skeleton, 반응형, UI 컴포넌트 작업 시 참조.
---

# shadcn/Tailwind — UI 구현 패턴

> Next.js App Router 메커니즘(Server/Client 경계·loading.tsx·route group) → `nextjs-guide` 스킬 참조
> 데이터 레이어(React Query·query keys/options·prefetch·mutation·Realtime) → `react-query-guide` 스킬 참조
> Supabase 클라이언트/RLS/인증 → `supabase-guide` 스킬 참조
> 라이브러리 불문 UI 원칙(모바일 퍼스트·터치 타겟·반응형·로딩/빈/에러 상태·상태 색상 일관성) → `design-system` 스킬 참조. 이 스킬은 그 원칙들의 **Tailwind/shadcn 구현**만 다룬다.

**프리미티브 기준: Base UI(`@base-ui/react`).** shadcn/ui는 2026-07부터 Base UI가 기본 프리미티브다. 신규 컴포넌트·신규 코드는 Radix가 아니라 Base UI로 작성한다(§1).

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

- 모바일: `SidebarTrigger` 클릭 시 Sheet 오버레이로 사이드바 열림 (`collapsible="offcanvas"`). 이 Sheet는 Base UI `Dialog` 위에 구현된 shadcn 컴포넌트다 — `@radix-ui/react-dialog`로 직접 대체 구현하지 않는다.
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

## 1. 프리미티브 레이어 — Base UI (`@base-ui/react`)

shadcn 컴포넌트(`@workspace/ui/components/*`)의 내부 프리미티브는 **Base UI**다. 프리미티브를 직접 다뤄야 할 때(커스텀 팝업·합성·상태 스타일링)는 아래 규칙을 따른다.

### 패키지 & import

```bash
pnpm add @base-ui/react          # 단일 패키지 (Radix처럼 컴포넌트별 패키지를 추가하지 않는다)
```

```tsx
import { Dialog } from '@base-ui/react/dialog'      // 서브패스 import
import { Popover } from '@base-ui/react/popover'
```

- shadcn 컴포넌트 추가는 기본값 그대로: `pnpm dlx shadcn add <component>` → Base UI 버전이 설치된다.
- `-b radix` 플래그는 **레거시 유지 목적 외 사용 금지**. 신규 프로젝트/신규 컴포넌트는 Base UI 기본값을 쓴다.

### 합성 — `asChild` 대신 `render`

Base UI에는 `asChild`/`Slot`이 없다. 트리거를 다른 엘리먼트/커스텀 컴포넌트로 렌더링할 때는 `render` prop을 쓴다.

```tsx
// ✅ Base UI
<Menu.Trigger render={<Button variant="outline" />}>메뉴 열기</Menu.Trigger>
<Menu.Item render={<Link href="/servers" />}>서버 목록</Menu.Item>

// ❌ Radix 시절 패턴 (Base UI에서 동작하지 않음)
<Menu.Trigger asChild><Button variant="outline">메뉴 열기</Button></Menu.Trigger>
```

- `render`에 넘기는 커스텀 컴포넌트는 **ref를 forward하고 받은 props를 DOM 노드에 모두 spread** 해야 한다. 안 하면 트리거 동작·접근성 속성이 사라진다.
- 상태에 따라 내용을 바꿔야 하면 함수 형태를 쓴다.

```tsx
<Switch.Thumb
  render={(props, state) => <span {...props}>{state.checked ? <CheckIcon /> : <XIcon />}</span>}
/>
```

- 앱 자체 컴포넌트에 "엘리먼트 교체" 기능이 필요하면 `Slot` 대신 `useRender`(`@base-ui/react/use-render`)로 구현한다.

### 구조 — `Content` 대신 `Positioner` + `Popup`

떠 있는 컴포넌트(Popover·Menu·Select·Tooltip)는 위치 계산 레이어(`Positioner`)와 내용 레이어(`Popup`)가 분리돼 있다. `side`/`align`/`sideOffset`은 **`Positioner`**에 준다.

```tsx
<Popover.Root>
  <Popover.Trigger render={<Button variant="outline" />}>필터</Popover.Trigger>
  <Popover.Portal>
    <Popover.Positioner side="bottom" align="start" sideOffset={8}>
      <Popover.Popup className="rounded-md border bg-popover p-4 shadow-md">
        <Popover.Title>필터</Popover.Title>
        <Popover.Description>조건을 선택하세요.</Popover.Description>
      </Popover.Popup>
    </Popover.Positioner>
  </Popover.Portal>
</Popover.Root>
```

- Dialog 계열 구성: `Root` / `Trigger` / `Portal` / `Backdrop` / `Viewport` / `Popup` / `Title` / `Description` / `Close`.
- Menu·Select에서 `Label`은 아무 데나 두지 말고 **`Group` 내부에 중첩**한다(Base UI가 그룹 구조를 요구).
- 위치 계산을 직접 커스텀해야 하면 Radix 내부 로직 대신 Floating UI(`@floating-ui/react`)를 쓴다 — Base UI가 그 위에 서 있다.

### 상태 스타일링 — `data-*` 속성

Base UI는 `data-state="open"` 하나가 아니라 **상태별 개별 속성**을 붙인다. Tailwind 셀렉터도 그에 맞춰 쓴다.

```tsx
// ✅ Base UI
<Menu.Item className="data-highlighted:bg-accent data-disabled:opacity-50">복제</Menu.Item>
<Dialog.Popup className="transition data-starting-style:opacity-0 data-ending-style:opacity-0 data-ending-style:scale-95" />

// ❌ Radix 시절 셀렉터
<Menu.Item className="data-[state=open]:bg-accent data-[highlighted]:bg-accent" />
```

주요 속성: `data-open` / `data-closed`, `data-starting-style` / `data-ending-style`(진입·퇴장 트랜지션), `data-highlighted`, `data-checked` / `data-unchecked`, `data-disabled`, `data-pressed`, `data-side`.

- 열림/닫힘 애니메이션은 keyframe 클래스(`animate-in`/`animate-out`)를 새로 만들지 말고 `transition` + `data-starting-style`/`data-ending-style` 조합으로 처리한다.

### Radix → Base UI 대응표

| 영역 | Radix (레거시) | Base UI (기준) |
|------|---------------|----------------|
| 패키지 | `@radix-ui/react-*` 개별 설치 | `@base-ui/react` 단일 + 서브패스 import |
| 합성 | `asChild` + `Slot` | `render` prop |
| 합성 유틸 | `Slot` | `useRender` (`@base-ui/react/use-render`) |
| 팝업 내용 | `*.Content` | `*.Positioner` + `*.Popup` |
| 위치 prop | `Content`의 `side`/`align` | `Positioner`의 `side`/`align`/`sideOffset` |
| 열림 상태 | `data-state="open"` | `data-open` / `data-closed` |
| 애니메이션 | `data-[state=open]:animate-in` | `transition` + `data-starting-style` / `data-ending-style` |
| 라벨 배치 | 팝업 내 자유 배치 | `Group` 내부에 중첩 |
| 위치 계산 커스텀 | Radix 내부 로직 | Floating UI(`@floating-ui/react`) 직접 사용 |

### 기존 Radix 코드 이관

- Radix는 deprecated가 아니다 — **한 번에 갈아엎지 않고 컴포넌트 단위로 점진 이관**한다. 혼재 상태는 허용하되, 신규 코드는 항상 Base UI.
- 이관은 `pnpm dlx skills add shadcn/ui` 후 `migrate <component> to base-ui` 형태로 컴포넌트별 진행 → `.migration/<component>.md` 리포트에서 동작 차이를 확인하고 커밋을 분리한다.
- 이관 후 확인 항목: `asChild` 잔존 여부, `data-[state=...]` 셀렉터 잔존 여부, `side`/`align`이 `Positioner`로 옮겨졌는지, 트리거로 넘긴 컴포넌트의 ref forward + props spread.

### 금지 패턴

```tsx
// ❌ 프리미티브 직접 의존 (shadcn 컴포넌트를 우회)
import * as DialogPrimitive from '@radix-ui/react-dialog'

// ❌ asChild — Base UI에는 없음
<Button asChild><Link href="/servers">이동</Link></Button>
// ✅
<Button render={<Link href="/servers" />}>이동</Button>

// ❌ Positioner 없이 Popup에 위치 prop
<Popover.Popup side="bottom" sideOffset={8} />
```

---

## 2. 컴포넌트 패턴

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
