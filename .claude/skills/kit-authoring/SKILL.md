---
name: kit-authoring
description: >
  claude-devkits의 스킬(SKILL.md)·에이전트 정의 작성/수정 상세 절차. 킷/스킬/에이전트 추가,
  description 작성, Gotchas 추가, references 분리, evals 케이스 작성 시 반드시 사용.
  검증 절차는 kit-audit 소관. 규칙의 근거는 docs/agent-skill-best-practices.md 참조.
---

# Kit Authoring — 작성/수정 절차

저장소 CLAUDE.md의 체크리스트가 규칙의 요약본이다. 이 스킬은 실제 파일을 만들 때의 상세 절차를 담는다.

## 1. 파일 지도

```
plugins/<킷>/
├── .claude-plugin/plugin.json     ← name, version(필수), description, repository
├── agents/<이름>.md               ← 원칙 킷만 보유. frontmatter: name/description/model/tools/skills
├── skills/<이름>/SKILL.md         ← 스킬명 = 디렉터리명 = frontmatter name
│   └── references/<주제>.md       ← 조건부 상세 (마이그레이션 절차 등)
└── evals/<스킬명>-trigger/        ← 발동 테스트, no-trigger/ ← 비발동 테스트
```

원칙 킷(common/backend/frontend) = 스택 불문 판단 기준. 기술 킷(python/nextjs/supabase 등) = 스택 종속 실무 규칙.
새 내용이 어느 쪽인지부터 판정하고, 애매하면 원칙(불문 부분)과 기술(종속 부분)로 쪼갠다.

## 2. description 작성 공식

`[무엇을 — 동사+구체 대상] + [언제 — 사용자가 실제 입력할 키워드, "~시 사용"] + [다루지 않는 것 — 겹칠 수 있는 스킬명 명시]`

- 이 저장소의 실제 예를 먼저 읽고 문체를 맞춘다: `plugins/backend-kit/skills/db-architecture/SKILL.md`
- 한 줄이면 `: `(콜론+공백) 금지 — "예: X"는 "X 등"으로. 여러 줄이면 `description: >` 블록 스칼라.
- 트리거 키워드를 늘리려 길게 쓰지 않는다 — 목록 예산을 잠식해 역효과. 1,024자 한도지만 400자 내외 권장.

## 3. 본문 수정 규칙

- 타 킷 스킬 참조를 추가하면 반드시 가드를 붙인다. 표준 문구:
  `(→ \`스킬명\` 참조 — 해당 킷 미설치 시 이 참조는 건너뛴다)`
  문서 상단에 이미 문서 전체 가드가 있으면 중복해 달지 않는다.
- 새로 발견한 함정은 해당 스킬의 기존 `Gotchas` 섹션 끝에 bullet로 추가한다. 형식: `- <함정> — <대처>`.
- 조건부 상세(특정 상황에서만 읽는 절차)가 30줄을 넘으면 `references/<주제>.md`로 분리하고
  본문에 "언제 읽는지"를 명시한 포인터를 남긴다. 참조는 SKILL.md에서 1단계까지만.
- 예시는 자리표시자(`resources`, `{feature}`)로 — 특정 프로젝트 이름·경로 금지.

## 4. 에이전트 정의 수정 규칙

- frontmatter: `tools: Read, Grep, Glob`(자문형 기본), `skills:`는 **같은 킷 스킬만** 프리로드 가능.
- description = 호출 조건 + 타 에이전트 소관 경계. 타 킷 에이전트 언급엔 "(해당 킷 설치 시)".
- 본문 필수 섹션: 핵심 역할 / 작업 원칙 / 입력·출력 프로토콜(압축 요약 반환 포함) / 에러 핸들링 / 협업.

## 5. evals 케이스 작성

트리거 영향 변경(신규 스킬, description 수정) 시 함께 작성한다:

```
evals/<스킬명>-trigger/prompt.md      ← frontmatter(name, plugins:["."], runs:2, max_turns:8) + 실사용 문장
evals/<스킬명>-trigger/graders/skill-used.md
  → type: tool_used / tool: Skill / input_match: '"skill"\s*:\s*"[^"]*<스킬명>"' / min: 1
evals/no-trigger/  ← 킷과 무관한 요청 + 각 스킬 min:0,max:0 grader
```

프롬프트는 실제 사용자가 칠 법한 자연스러운 한국어 문장으로. 기존 케이스 문체를 따른다.

## 6. 릴리스 동반 수정

- 변경된 킷의 plugin.json `version` bump (에이전트/스킬 이름 변경은 minor 이상 + README 갱신).
- 에이전트 추가/이름 변경 시: README 표 2곳(구조도·에이전트 표)과 marketplace.json description 확인.
