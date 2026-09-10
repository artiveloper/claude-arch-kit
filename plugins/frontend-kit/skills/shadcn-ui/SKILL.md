---
name: shadcn-ui
description: >
  Tailwind/shadcn UI 구현 패턴 가이드. 프리미티브는 Base UI(@base-ui/react) 기준.
  모바일 퍼스트 className 규칙, Base UI 합성(render prop)·data-* 상태 스타일링, shadcn Sidebar 레이아웃,
  컴포넌트 패턴 — 액션 버튼(Button), 상태 Badge, 날짜·시간 포맷 중앙화, 리스트 Client Component, 공유 스켈레톤 컴포넌트.
  Next.js App Router 메커니즘은 nextjs-guide, 데이터 레이어는 react-query-guide 참조.
  Tailwind, shadcn, Base UI, render prop, className, Sidebar, Button, Badge, Skeleton, 반응형, UI 컴포넌트 작업 시 참조.
---

# shadcn/Tailwind — UI 구현 패턴

> Next.js App Router 메커니즘(Server/Client 경계·loading.tsx·route group) → `nextjs-guide` 스킬 참조
> 데이터 레이어(React Query·query keys/options·prefetch·mutation·실시간 구독) → `react-query-guide` 스킬 참조
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
import { SidebarProvider, SidebarInset, SidebarTrigger } from '@/components/ui/sidebar'
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

shadcn 컴포넌트(`@/components/ui/*`)의 내부 프리미티브는 **Base UI**다. 프리미티브를 직접 다뤄야 할 때(커스텀 팝업·합성·상태 스타일링)는 아래 규칙을 따른다.

### 패키지 & import

```bash
npm install @base-ui/react       # 단일 패키지 (Radix처럼 컴포넌트별 패키지를 추가하지 않는다)
                                 # pnpm/yarn/bun 등 프로젝트가 쓰는 패키지 매니저로 실행
```

```tsx
import { Dialog } from '@base-ui/react/dialog'      // 서브패스 import
import { Popover } from '@base-ui/react/popover'
```

- shadcn 컴포넌트 추가는 기본값 그대로: `npx shadcn add <component>`(또는 `pnpm dlx`/`bunx`) → Base UI 버전이 설치된다.
- `-b radix` 플래그는 **레거시 유지 목적 외 사용 금지**. 신규 프로젝트/신규 컴포넌트는 Base UI 기본값을 쓴다.

### 합성 — `asChild` 대신 `render`

Base UI에는 `asChild`/`Slot`이 없다. 트리거를 다른 엘리먼트/커스텀 컴포넌트로 렌더링할 때는 `render` prop을 쓴다.

```tsx
// ✅ Base UI
<Menu.Trigger render={<Button variant="outline" />}>메뉴 열기</Menu.Trigger>
<Menu.Item render={<Link href="/resources" />}>목록으로</Menu.Item>

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
- 이관은 `npx skills add shadcn/ui`(또는 `pnpm dlx`/`bunx`) 후 `migrate <component> to base-ui` 형태로 컴포넌트별 진행 → `.migration/<component>.md` 리포트에서 동작 차이를 확인하고 커밋을 분리한다.
- 이관 후 확인 항목: `asChild` 잔존 여부, `data-[state=...]` 셀렉터 잔존 여부, `side`/`align`이 `Positioner`로 옮겨졌는지, 트리거로 넘긴 컴포넌트의 ref forward + props spread.

### 금지 패턴

```tsx
// ❌ 프리미티브 직접 의존 (shadcn 컴포넌트를 우회)
import * as DialogPrimitive from '@radix-ui/react-dialog'

// ❌ asChild — Base UI에는 없음
<Button asChild><Link href="/resources">이동</Link></Button>
// ✅
<Button render={<Link href="/resources" />}>이동</Button>

// ❌ Positioner 없이 Popup에 위치 prop
<Popover.Popup side="bottom" sideOffset={8} />
```

---

## 2. 컴포넌트 패턴

> 아래 컴포넌트가 소비하는 데이터 훅(`useResource*` 등)의 정의는 `react-query-guide` 스킬 참조. Server/Client 경계·`'use client'`는 `nextjs-guide` 참조. 이 섹션은 그 훅을 소비하는 **UI 스타일링**만 다룬다.
> 도메인 이름(`resource`, `status` 값 등)은 자리표시자다 — 프로젝트의 실제 엔티티로 바꿔 읽는다.

### 상태 전이 액션 버튼

전이 중(`isPending` 또는 중간 상태)에는 **모든 액션을 함께 잠근다.** 개별 버튼만 막으면 연타로 모순된 요청이 나간다.

```tsx
// components/resource-actions.tsx
'use client'
export function ResourceActions({ id, status }: { id: string; status: ResourceStatus }) {
  const { mutate, isPending } = useResourceTransition(id)
  const isTransitioning = status === 'activating' || status === 'deactivating'
  const locked = isPending || isTransitioning

  return (
    <div className="flex gap-2">
      <Button onClick={() => mutate('activate')}
        disabled={status !== 'inactive' || locked}>활성화</Button>
      <Button variant="destructive" onClick={() => mutate('deactivate')}
        disabled={status !== 'active' || locked}>비활성화</Button>
      <Button variant="outline" onClick={() => mutate('restart')}
        disabled={status !== 'active' || locked}>재시작</Button>
    </div>
  )
}
```

### 상태 Badge

> 상태 → 라벨/색상 중앙 매핑·색상 단독 의존 금지 원칙은 `design-system` 스킬 참조. 아래는 그 구현 형태다.

상태값 집합은 프로젝트마다 다르다 — **중요한 것은 값이 아니라 "한 곳에서 매핑하고 라벨을 반드시 함께 준다"는 형태**다.

```tsx
const statusConfig: Record<ResourceStatus, { label: string; variant: string }> = {
  active:       { label: '활성', variant: 'success' },
  inactive:     { label: '비활성', variant: 'secondary' },
  activating:   { label: '활성화 중', variant: 'warning' },
  deactivating: { label: '비활성화 중', variant: 'warning' },
  pending:      { label: '대기', variant: 'default' },
  error:        { label: '오류', variant: 'destructive' },
}
```

- 이 매핑을 화면마다 다시 선언하지 않는다 — 한 모듈에서 export해 재사용한다.
- `variant`만 주고 `label`을 생략하지 않는다(색각 이상 사용자 대응).

### 날짜·시간 포맷 — 중앙 유틸 + 타임존 명시

날짜·시간 포맷을 컴포넌트마다 인라인으로 쓰지 않고 **`@/lib/format.ts` 같은 단일 모듈로 중앙화**한다. 화면마다 포맷이 갈리는 것과, 타임존이 뷰어의 브라우저 설정에 따라 달라지는 것을 동시에 막는다.

```ts
// src/lib/format.ts — 로케일·타임존은 프로젝트가 한 번 정한다
const LOCALE = 'ko-KR';
const TIME_ZONE = 'Asia/Seoul';   // 프로젝트 기준 타임존. 사용자별 타임존이 필요하면 인자로 받는다

export const formatDate = (d: string | Date) =>
  new Intl.DateTimeFormat(LOCALE, { dateStyle: 'medium', timeZone: TIME_ZONE }).format(new Date(d));

export const formatDateTime = (d: string | Date) =>
  new Intl.DateTimeFormat(LOCALE, { dateStyle: 'medium', timeStyle: 'short', timeZone: TIME_ZONE }).format(new Date(d));
```

```ts
import { formatDate, formatDateTime } from '@/lib/format';

formatDate(resource.createdAt)      // 중앙 정의된 로케일·타임존으로 렌더
formatDateTime(resource.createdAt)
```

- **`timeZone`을 생략한 `toLocaleDateString()`/`toLocaleString()` 직접 호출 금지** — 서버(UTC)와 클라이언트(로컬)의 결과가 달라져 하이드레이션 불일치가 난다.
- 숫자 포맷(금액 천단위 등)은 `toLocaleString`으로 충분하다 — 타임존과 무관하다.

### 리스트 Client Component

```tsx
'use client'
import { TableSkeleton } from '@/components/ui/skeletons'  // 아래 "공유 스켈레톤" 참조

export function ResourceListClient() {
  const { data: resources, isLoading, error } = useResources()

  if (isLoading) return <TableSkeleton rows={6} cols={5} />  // 텍스트 "불러오는 중..." 금지
  if (error) return <p className="text-destructive">목록을 불러올 수 없습니다.</p>
  if (!resources?.length) return <p className="text-muted-foreground">등록된 항목이 없습니다.</p>

  return <table>...</table>
}
```

로딩·에러·빈 목록 **세 갈래를 각각 처리**한다(원칙 → `design-system`).

> 클라이언트 `isLoading`(캐시 없음) vs 라우트 레벨 `loading.tsx`(prefetch 대기) 역할 구분 → `nextjs-guide`.

### 공유 스켈레톤 컴포넌트

shadcn이 제공하는 것은 프리미티브 `<Skeleton />` 하나뿐이다. 화면마다 스켈레톤을 새로 조립하지 말고, **프로젝트에서 조합 컴포넌트를 한 번 만들어** 재사용한다.

```
src/components/ui/skeletons.tsx   ← 프로젝트가 직접 만드는 조합 레이어
```

| 컴포넌트 | 용도 |
|---------|------|
| `<TableSkeleton rows cols />` | 테이블 리스트 (rows/cols 조절) |
| `<CardGridSkeleton count />` | 통계/요약 카드 그리드 |
| `<PageHeaderSkeleton />` | 페이지 제목 + 설명 영역 |

- 스켈레톤은 **최종 콘텐츠의 레이아웃 형태를 유지**해야 한다(→ `design-system`). 형태가 다르면 로드 후 레이아웃 점프가 난다.
- 라우트 레벨 로딩(`loading.tsx`)에서의 조합·배치는 → `nextjs-guide`.
- 로딩 분기에서 "불러오는 중..." 텍스트는 금지.
