# ai-guide — 아키텍처 리뷰 하네스 (레퍼런스)

여러 프로젝트가 공통으로 재사용하는 **아키텍처/품질 리뷰 지식**을 에이전트·스킬로 모아둔 **레퍼런스 저장소**입니다.

> **이 저장소 안에서는 에이전트·스킬이 동작하지 않습니다.** Claude Code는 `.claude/agents/`·`.claude/skills/<이름>/SKILL.md`만 탐색하므로, 여기 있는 `back/`·`front/`·`common/` 파일은 로드되지 않습니다. 이건 의도된 것입니다 — 이 저장소는 **읽고 참고해서 프로젝트에 새로 적용하는 원본**이지, 그대로 실행되는 하네스가 아닙니다. 적용 방법은 아래 [프로젝트에 적용하기](#프로젝트에-적용하기) 참조.

각 에이전트는 독립적으로 호출되는 것을 전제로 작성됐습니다(에이전트 간 통신 없음).

## 설계 철학

- **원칙 스킬은 스택 무관하게** 담는다. 언어·프레임워크에 종속된 규칙(예: Next.js App Router, Supabase 인증)은 원칙 스킬에 섞지 않고 별도 **기술 스킬**로 분리한다.
- **에이전트는 얇게, 원칙은 스킬에** 둔다. 에이전트는 역할·판단 태도만 정의하고, 실제 판단 기준은 짝이 되는 스킬에서 로드한다.
- **오케스트레이션은 프로젝트 몫이다.** 여러 에이전트를 순서대로 엮는 것과 프로젝트 특이사항은 각 프로젝트에서 구성한다. 이 저장소의 원본은 항상 범용성을 유지한다.

## 디렉터리 구조

도메인(`common`/`back`/`front`)별로 `agents/`(에이전트 정의)와 `skills/`(판단 기준)를 짝지어 둔다.

```
common/
├─ agents/   architect.md
└─ skills/   system-architecture/
back/
├─ agents/   backend-architect.md, dba-advisor.md, qa-backend.md
└─ skills/   backend-architecture/, db-architecture/, qa-backend-strategy/
             python-guide/, supabase-guide/
front/
├─ agents/   design-system.md, frontend-architect.md, qa-frontend.md
└─ skills/   design-system/, frontend-architecture/, qa-frontend-strategy/
             nextjs-guide/, react-query-guide/, shadcn-ui/
```

## 에이전트 ↔ 원칙 스킬

### `common/` — 횡단(시스템) 설계

| 에이전트 | 스킬 | 담당 |
|----------|------|------|
| `architect` | `system-architecture` | 요구사항(FR/NFR) 구조화, 규모별 기술 스택 선정 규율, KISS·확장 지점, 도메인 간 경계 조율 |

### `back/` — 백엔드

| 에이전트 | 스킬 | 담당 |
|----------|------|------|
| `backend-architect` | `backend-architecture` | 계층 분리, API 계약·응답 봉투, 입력 검증, 트랜잭션 경계, 인증/인가, 보안 기본, 에러 처리 |
| `dba-advisor` | `db-architecture` | 정규화, 인덱스 전략, 마이그레이션 안전성, N+1/락 이슈 |
| `qa-backend` | `qa-backend-strategy` | API 계약 검증, 통합 테스트 우선순위, 동시성·회귀 방지 |

### `front/` — 프론트엔드

| 에이전트 | 스킬 | 담당 |
|----------|------|------|
| `frontend-architect` | `frontend-architecture` | 상태 분류, 컴포넌트 경계, 데이터 fetching, 낙관적 업데이트 롤백, 에러 경계, 폼 검증 이중화 |
| `qa-frontend` | `qa-frontend-strategy` | 행동 기반 컴포넌트 테스트, E2E 우선순위, 안정적 셀렉터 |
| `design-system` | `design-system` | 디자인 토큰, 컴포넌트 재사용, 반응형/모바일 퍼스트, 접근성, 로딩·빈·에러 상태 |

각 원칙 스킬 하단에는 `## 리뷰 시 체크 우선순위`가 있어, 리뷰 결과가 호출마다 흔들리지 않도록 판단 순서를 고정한다.

## 기술 스킬

원칙 스킬과 달리 특정 언어·프레임워크에 종속된 실무 규칙을 담는다. 에이전트에 고정으로 묶이지 않고, 해당 기술을 다룰 때 스킬 설명(description) 매칭으로 로드되도록 작성됐다.

| 스킬 | 위치 | 범위 |
|------|------|------|
| `python-guide` | `back` | 범용 Python 클린코드 — src layout, 타입 힌트(mypy/pyright), uv/poetry·pyproject.toml, ruff, pytest, 예외 계층, asyncio |
| `supabase-guide` | `back` | Supabase SSR 인증(`@supabase/ssr`), RLS·성능, 타입 생성, 신규 publishable/secret 키 마이그레이션 |
| `nextjs-guide` | `front` | App Router 메커니즘 — 파일 컨벤션, Server/Client Component 경계, route group, `loading.tsx`+Suspense 스트리밍 |
| `react-query-guide` | `front` | 데이터 레이어 — query keys/options/prefetch, mutation·invalidate·낙관적 업데이트, Server Actions, Realtime, 상태 소유권 |
| `shadcn-ui` | `front` | Tailwind/shadcn UI 구현 패턴. 프리미티브는 **Base UI(`@base-ui/react`)** 기준 — 합성(render prop), `data-*` 상태 스타일링, Sidebar 레이아웃 |

> 기술 스킬도 **특정 프로젝트에 종속되지 않는다.** 코드 예시는 중립적인 경로(`src/...`, `@/components/ui/*`)와 예시 도메인(`resources`, `catalog_items` 등)을 쓰므로, 프로젝트에 적용할 때 자기 경로·테이블명으로 바꿔 읽으면 된다.

## 프로젝트에 적용하기

1. 필요한 에이전트/스킬을 골라 프로젝트의 `.claude/agents/`·`.claude/skills/`로 옮긴다 — 도메인 폴더(`back`/`front`/`common`) 없이 평평하게 둬야 Claude Code가 탐색한다.
   - 에이전트: `.claude/agents/backend-architect.md`
   - 스킬: `.claude/skills/backend-architecture/SKILL.md`
2. 프로젝트 특수 규칙은 파일 하단에 `## 프로젝트 특이사항` 섹션으로 추가한다. 원본 본문은 건드리지 않아야 나중에 원본 갱신분을 다시 반영하기 쉽다.
3. 기술 스킬은 프로젝트 스택에 맞는 것만 가져가고, 예시 경로·테이블명은 자기 프로젝트 컨벤션으로 교체한다.
4. 여러 에이전트를 묶어 실행하는 오케스트레이션은 그 프로젝트의 하네스에서 구성한다.

원본(이 저장소)의 원칙 스킬은 스택 무관 범용 원칙만 유지한다.
