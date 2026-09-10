# Radix → Base UI 이관 가이드

> `shadcn-ui` 스킬의 참조 문서. 기존 Radix 기반 코드를 Base UI로 옮길 때만 읽는다. 신규 코드는 처음부터 Base UI로 작성하므로 읽을 필요 없다.

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
