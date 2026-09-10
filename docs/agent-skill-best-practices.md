# 클로드 에이전트·스킬 지침 작성 베스트 프랙티스

> 목적: 이 저장소(개발 스택별 최고의 에이전트·스킬 킷 구성)를 위해, Anthropic 공식 문서·엔지니어링 블로그, 해외 권위자 블로그("agent skills best practices" 검색 결과 포함), 국내 기술 블로그를 조사해 정리한 문서. (조사일: 2026-09-10)

---

## 0. TL;DR — 가장 중요한 10가지

1. **description이 성패의 90%** — 스킬/에이전트가 안 쓰이는 문제 대부분은 본문이 아니라 description(트리거) 문제다. 나쁜 description은 성능 50% 손실(Philipp Schmid).
2. description 공식: **[무엇을: 동사+구체적 대상] + [언제: "Use when…" + 사용자가 실제 입력할 키워드] + [언제 아님: "Not for…"]**. 반드시 3인칭.
3. **SKILL.md 본문은 500줄 미만**, 상세는 `references/`로 분리(참조는 1단계 깊이만, 100줄+ 파일엔 목차).
4. **"Claude는 이미 똑똑하다"** — LLM이 이미 아는 일반 지식은 빼고, 프로젝트/도메인 고유 규약과 비자명한 엣지케이스(Gotchas)만 담아라. "컨텍스트 윈도우는 공공재."
5. **One skill, one capability** — 여러 범주에 걸치는 스킬은 에이전트를 혼란시킨다. 무관한 스킬은 중립이 아니라 **능동적으로 오도**한다(Red Hat).
6. 서브에이전트는 **단일 책임 + 직무형 이름**(`pr-reviewer`, `test-runner`), description엔 능력이 아닌 **호출 조건**, tools는 명시적으로 최소화.
7. **스킬 목록엔 예산이 있다**(컨텍스트의 ~1%, 기본 약 15,000자) — 초과하면 경고 없이 스킬이 목록에서 탈락한다. 스킬 수와 description 길이 자체가 라우팅 세금.
8. 결정론적·반복적 작업은 지침 대신 **스크립트**로(신뢰성↑, 토큰↓ — Red Hat 실측 비용 26% 절감, 우아한형제들 실측 컨텍스트 96.5% 절감).
9. **문서보다 평가 먼저** — 스킬 없이 베이스라인 측정 → 평가 시나리오 3개+ → 최소 지침 → 반복. 발동해야 할 케이스와 발동하면 안 되는 케이스 둘 다 테스트.
10. 스택별 킷 구성의 업계 수렴 패턴: **플러그인 하나 = 스택 하나 = 소수 에이전트(2~3) + 다수 스킬**. 자동 생성은 70점, 도메인 맥락 커스터마이징으로 90점(펀코딩).

---

## 1. 멘탈 모델: 하네스 엔지니어링

- **에이전트 = 모델 + 하네스.** "모델은 지능을 담고, 하네스는 그 지능을 신뢰할 수 있는 행동으로 전환한다." 하네스 설계는 모델 업그레이드급 효과(Databricks 실측: 동일 모델에서 36.10%→52.63%). — [Databricks: AI 에이전트 하네스](https://www.databricks.com/kr/blog/ai-harness)
- 진화 계보: 프롬프트 엔지니어링 → 컨텍스트 엔지니어링 → **하네스 엔지니어링**. "프롬프트를 잘 쓰는 시대"에서 "AI가 일할 환경·맥락을 설계하는 시대"로. — [우아한형제들](https://techblog.woowahan.com/26177/)
- 하네스 3기능: **제어(가드레일) / 감시(모니터링) / 개선(피드백 루프)**. 비가역 작업(삭제·전송·배포)엔 human-in-the-loop. — [채널톡](https://channel.io/kr/blog/articles/what-is-harness-2611ddf1)
- Anthropic의 방향: **"Don't build agents, build skills"** — 용도별 커스텀 에이전트를 파편적으로 만들지 말고, 범용 에이전트를 조합 가능한 스킬로 전문화하라. 스킬 작성은 "신입사원 온보딩 가이드 작성"과 같다. — [Equipping agents for the real world](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- Agent·Skill·Context 3요소 모델: **Agent = 역할과 책임을 가진 실행 주체**(역할 단위로 정의), **Skill = 재사용 가능한 작업 단위**(npm 패키지처럼 버전 관리·테스트·공유되는 조직 자산 — 개인 종속적이고 재현성 낮은 프롬프트와의 차별점), **Context = 의도적으로 제한된 정보 공간**(역할에 맞는 범위만 허용해 예측 가능성 확보). "누가 실행해도 같은 결과가 나오는 AI 작업 흐름"이 목표. — [kt cloud](https://tech.ktcloud.com/entry/2026-04-ktcloud-claude-agent-ai%ED%99%9C%EC%9A%A9-%EA%B0%9C%EB%85%90%EC%A0%95%EB%A6%AC)
- Simon Willison: 스킬은 "마크다운 + 약간의 YAML + 선택적 스크립트"라는 극단적 단순함이 강점. MCP가 세션당 수만 토큰을 쓰는 반면 스킬은 **스킬당 수십 토큰**(메타데이터만)이라 토큰 효율이 압도적. — [Claude Skills are awesome](https://simonwillison.net/2025/Oct/16/claude-skills/)

---

## 2. 스킬(SKILL.md) 작성 규칙

### 2.1 Frontmatter — 정확한 제약

| 항목 | 규칙 |
|---|---|
| `name` | 최대 **64자**, 소문자·숫자·하이픈만, `anthropic`/`claude` 예약어 금지, 디렉터리명과 일치 |
| `description` | 최대 **1,024자**, 빈 값 금지, XML 태그 금지, **3인칭** |
| 네이밍 스타일 | 동명사형 권장(`processing-pdfs`), 명사구/동사형 허용. 컬렉션 내 한 가지 스타일로 통일. `helper`/`utils`/`tools` 같은 모호한 이름 금지 |

### 2.2 description — 트리거 신뢰성의 핵심

- **5요소 공식**: ①도메인 ②핵심 동사 ③구체적 대상 ④사용 시점("Use when…") ⑤**사용하지 않을 시점("Not for…")** — 다른 스킬과의 경계를 description에서 상호 해소.
  - 나쁨: `Helps with changelogs`
  - 좋음: `Write structured CHANGELOG.md entries from git diffs. Use when finalizing releases. Not for customer-facing announcements.`
- 사용자가 **실제로 입력하는 문구**를 트리거로 넣어라("review this draft", 파일 확장자, 팀 은어 포함). — [Claude Code 팀 운영기](https://claude.com/blog/lessons-from-building-claude-code-how-we-use-skills)
- **스킬 목록 예산 함정**: 스킬+커맨드 description 총량은 컨텍스트의 ~1%(기본 약 15,000자)로 제한되고, 초과 시 **경고 없이** 저사용 스킬부터 목록에서 제거된다(신규 스킬은 사용이력 0이라 최우선 탈락). "트리거 단어를 많이 넣어 길게 쓰라"는 통설은 역효과 — 구체적이되 짧게. 진단은 `/doctor`. — [Jesse Vincent](https://blog.fsck.com/2025/12/17/claude-code-skills-not-triggering/), [happyskills.ai](https://happyskills.ai/blog/why-your-skill-never-fires/)

### 2.3 본문

- **500줄 미만** (전 출처 일치). 스킬은 한 번 로드되면 이후 모든 턴에 토큰 비용이 발생한다.
- **점진적 공개(Progressive Disclosure)** 3단계: 메타데이터(항상) → SKILL.md 본문(트리거 시) → `references/`(필요 시). 디렉터리 관례: `scripts/`, `references/`, `assets/`.
  - 참조는 SKILL.md에서 **1단계 깊이만** (중첩 참조는 `head -100` 부분 읽기로 정보 누락 유발).
  - 100줄 넘는 참조 파일엔 **목차(TOC)** 필수. 파일명은 서술형(`form_validation_rules.md`, `doc2.md` 금지).
- 권장 본문 골격(Hermes 가이드 — SKILL.md 형식은 플랫폼 공통): **① When to use**(평어 트리거) → **② Quick reference**(명령어·환경변수·경로) → **③ Procedure**(에이전트가 즉흥적으로 버리면 안 되는 순서화된 단계) → **④ Pitfalls**(알려진 실패 모드) → **⑤ Verification**(성공 판정 기준). description은 "블로그 문단이 아니라 검색 결과처럼" 작성. — [Rost Glukhov](https://www.glukhov.org/ko/ai-systems/hermes/authoring-hermes-skill/)
- **Gotchas 섹션이 최고 신호 콘텐츠** — 운영하며 발견한 엣지케이스·함정을 축적하는 자리로 설계하라(Claude Code 팀: "The highest-signal content in any skill is the Gotchas section").
- **자유도(degrees of freedom) 매칭**:
  - 여러 접근이 유효한 판단형 작업 → 고자유도(휴리스틱 텍스트)
  - 선호 패턴이 있는 작업 → 중자유도(템플릿/의사코드)
  - 깨지기 쉽고 순서가 고정된 작업(마이그레이션 등) → 저자유도("이 스크립트를 그대로 실행, 플래그 추가 금지")
- 설명문 대신 **지시문**, 5문단 설명보다 **5줄 코드 스니펫**, 복잡 작업은 번호 단계 + 복사형 체크리스트 + **검증 루프**("validator 통과 전 진행 금지").
- 가능하면 이유를 함께("모델 Y는 deprecated라 에러 반환 → X 사용"). ALWAYS/NEVER 남발은 옐로 플래그.
- 안티패턴: 시한성 정보("2025년 8월 이전이면…"), 용어 비일관("endpoint"/"URL"/"route" 혼용), 선택지 과다(기본값 하나 + 예외 경로만), Windows 백슬래시 경로.

### 2.4 스크립트 vs 지침

- 기계적·결정론적 작업은 스크립트, 주관적 판단만 LLM에게 — 이 분리만으로 비용 26% 절감 실측(Red Hat). 우아한형제들의 전처리 스크립트 패턴(AI 자율 탐색 대신 정제 JSON 전달)은 평균 96.5% 컨텍스트 절감, Tool Call 4회→1회.
- 스크립트는 오류를 직접 처리(solve, don't defer), 상수엔 근거 주석(voodoo constant 금지), 오류 메시지는 구체적으로("Available fields: …").
- **실행 의도 명시**: "Run `x.py`"(실행) vs "See `x.py` for the algorithm"(참조). 의존 패키지는 명시(설치 가정 금지).
- 고위험/배치 작업엔 **plan–validate–execute** 패턴(중간 산출물 JSON + 검증 스크립트 후 실행).

### 2.5 스킬의 수명 관리

- **preference 스킬**(조직/프로젝트 고유 규약 — 오래가는 투자) vs **capability 스킬**(모델 능력 보완 — 모델 발전 시 소멸). capability 스킬은 주기적으로 스킬 없이 평가를 돌려 통과하면 **은퇴**시킨다.
- 스킬은 소프트웨어처럼: 버전 관리, PR 리뷰, CI 회귀 평가, 정본(canonical source) 한 곳 원칙.

---

## 3. 서브에이전트(agents/*.md) 작성 규칙

- **단일 책임**: `test-runner`(좁음) ⭕ / `dev-helper: help with anything dev-related` ❌. **직무형 이름**(`repo-explorer`, `pr-reviewer`).
- **description = 위임 트리거**: 능력 서술("security expert")보다 **호출 조건 서술**("Reviews code for security issues before commits")이 라우팅 성능이 좋다. 자동 위임을 원하면 "Use proactively after code changes" 같은 능동 트리거 포함. 짧게 유지(전체 에이전트 description 합산 15,000토큰 초과 시 경고).
- **본문 = 상세 시스템 프롬프트**: 역할, 체크 항목, 출력 형식을 구체적으로. 상세 지침은 description이 아니라 본문에.
- **도구 최소화**: `tools` 생략 = 전체 허용이므로 의도적으로 좁혀라. 리서치/리뷰 에이전트는 `Read, Grep, Glob` 읽기 전용, 구현 에이전트만 Edit/Write/Bash. 위험 작업은 `isolation: worktree`.
- **모델 매칭**: Haiku=단순·저비용(로그 파싱, 단순 검사), Sonnet=범용, Opus=복잡 추론/설계.
- **컨텍스트 특성**: 서브에이전트는 대화 히스토리 없이 시작(시스템 프롬프트+위임 메시지+CLAUDE.md만). 반복 지식은 frontmatter `skills:`로 프리로드 가능("본문에서 스킬을 로드하라"고 지시하는 것보다 확실). 반환은 **1,000–2,000토큰 압축 요약**으로 지시.
- **함정**: 여러 에이전트의 동일 파일 병렬 편집(충돌), 에이전트 로스터 과대(자동 위임 신뢰성 하락), 상호의존 작업 위임(서브에이전트끼리 직접 조율 불가).
- 스킬 vs 서브에이전트 vs CLAUDE.md: **CLAUDE.md=상시 로드**(모든 상호작용 형성), **스킬=온디맨드 지식**(반복 워크플로·팀 표준), **서브에이전트=컨텍스트 격리**(리서치 헤비, 병렬 독립 작업, 편향 없는 검증).

---

## 4. 스택별 킷/플러그인 구성 패턴

- 업계 수렴: **평면 컬렉션 → 자기완결 플러그인 마켓플레이스**. 플러그인 하나 = 스택 하나 = 소수 에이전트(2~3) + 다수 스킬(10~16). 설치 단위 격리로 미사용 스택이 컨텍스트를 오염시키지 않음. — [wshobson/agents](https://github.com/wshobson/agents)(192 에이전트→80 플러그인), [VoltAgent](https://github.com/VoltAgent/awesome-claude-code-subagents)
- wshobson의 CI 규약(참고할 만): 모든 description에 "Use when…"/"Trigger when…" 트리거 문구가 없으면 **린트 실패**(`MISSING_TRIGGER`). 스킬 8KB 캡. `$ARGUMENTS`는 라벨 블록으로 감싸 인젝션 방지.
- 플러그인 구조 주의: `.claude-plugin/`엔 plugin.json만, `skills/`·`agents/`는 플러그인 루트에. 스킬은 `plugin-name:skill-name`으로 네임스페이스. `version`을 올려야 업데이트가 전파. 테스트: `claude --plugin-dir` + `claude plugin validate`.
- 아키텍처 패턴 선택(펀코딩/revfactory harness의 6패턴): 파이프라인(순차 의존) / 팬아웃-팬인(병렬 다관점) / **전문가 풀**(스택별 전문가 자동 선택 — 이 저장소의 직접 모델) / **생성-검증**(별도 검증 에이전트가 자체 검토보다 엄격 — 저자가 가장 효과 본 패턴) / 감독자 / 계층 위임(초대형 전용). 단발성 작업은 팀 대신 서브에이전트로 시작해 복잡도에 따라 확장.
- 지침 배치 원칙: **가장 가까운 파일 우선**(AGENTS.md 중첩 배치, 우아한형제들 `globs` 스코핑) — 규칙의 적용 범위를 좁혀 토큰 낭비 방지. 정체성/제약/절차를 파일로 분리(gitagent의 SOUL/RULES/DUTIES/SKILL 분리).

---

## 5. 평가·운영

1. **문서보다 평가 먼저**: 갭 식별 → 평가 시나리오 **3개+** → 스킬 없이 베이스라인 → 최소 지침으로 통과 → 반복.
2. 테스트 프로토콜: 프롬프트 10~20개 × 3~5회(비결정성 감안), 클린 환경, **발동/비발동 케이스 모두**, 사용할 모든 모델 티어에서.
3. **Claude A/B 루프**: Claude A(전문가와 스킬 작성) → Claude B(신선한 인스턴스로 실사용) → 실패 관찰 → 지침 강화 → 반복.
4. 내비게이션 관찰: 안 읽는 참조 파일 = 제거 후보, 반복해서 읽는 파일 = SKILL.md 본문 승격 후보.
5. PreToolUse 훅으로 스킬 발동 로깅 → under-triggering 탐지. CI에 트리거 문구 린트 + 회귀 평가.
6. 운영 수칙(펀코딩): 큰 작업 전 드라이 런 + 스몰 테스트. 품질 3포인트 — 역할 혼선 / 중간 데이터 전달 누락 / **단일 에이전트 대비 실제 개선 여부**(개선 없으면 패턴 교체). 멀티에이전트는 토큰이 거의 지수적으로 증가 — 단순 작업엔 오히려 비효율.

---

## 6. Claude Code 팀 공식 가이드 — 추가 확인분

Claude Code 팀/Anthropic이 직접 낸 자료 중 위 2~5장에 아직 반영되지 않았던 것들.

### 6.1 CLAUDE.md 공식 가이드
- **약 200줄 이하** 유지 권장(과도하게 길면 지침이 무시되는 경향). 포함할 것: 자주 쓰는 Bash 커맨드, 코드 스타일, 테스트 지침, 아키텍처 결정. 제외할 것: **코드에서 유추 가능한 내용**과 자명한 정보.
- 로드 순서: 관리형(조직) → 사용자(`~/.claude/`) → 프로젝트 → 로컬 — 광범위→특정 순으로 적용(4장의 "가까운 파일 우선" 원칙과 일치). — [Best practices](https://code.claude.com/docs/en/best-practices), [Memory 문서](https://code.claude.com/docs/en/memory)

### 6.2 공식 스킬 생성 가이드 (claude.com 블로그, 2026)
- 5단계: ①측정 가능한 성과로 요구사항 정의 → ②소문자-하이픈 이름 → ③**description이 최중요**(구체적 트리거 + 경계 + 액션 동사) → ④지침 구조(개요→전제조건→실행 단계→예시→에러 처리→한계) → ⑤배포.
- **스킬화 기준: 이미 5회 이상 수행했고 앞으로 10회 이상 재수행할 작업**.
- 테스트 매트릭스: 정상 케이스 / 엣지 케이스 / **범위 밖 요청**(비발동 확인) 3종. — [How to create skills](https://claude.com/blog/how-to-create-skills-key-steps-limitations-and-examples)

### 6.3 대규모 코드베이스 5계층 하네스
- ①CLAUDE.md 레이어링(루트 + 서브디렉터리별) ②훅(문서 자동 갱신) ③스킬(경로별 스코핑) ④플러그인(팀 배포 번들) ⑤MCP 서버(내부 도구).
- **지침 재검토 주기 3~6개월**: 모델이 발전하면 예전에 필요했던 제약을 해제하라(스킬 은퇴 기준과 같은 맥락). — [Large codebases](https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start)

### 6.4 장기 실행 에이전트 하네스
- 2단계 패턴: **Initializer 에이전트**(최초 1회 — 환경 설정, 진행상황 파일, 초기 커밋) + **Coding 에이전트**(이후 세션 — 단일 기능 증분).
- 세션 간 상태 전달: 진행상황 파일(`claude-progress.txt`) + git 히스토리 + 기능 목록 JSON. 매 세션 엔드투엔드 검증. — [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

### 6.5 자체 개선 스킬 구조 (Warp 사례)
- **2-스킬 프레임워크**: 기본 스킬(도메인 지침) + **개선자 스킬**(사용 피드백을 관찰해 기본 스킬 업데이트를 제안) — Gotchas 축적을 자동화하는 패턴.
- 작성 원칙: 엄격한 규칙 나열 대신 추론 가이드, 규칙의 **"왜"를 설명**(일반화 가능해짐). 피드백은 이진 승인보다 이유가 달린 것이 가치 높음. — [Warp 사례](https://claude.com/blog/how-warp-builds-self-improving-agents-on-claude)

### 6.6 병렬 실행 4방식 공식 비교
| 방식 | 조율 | 적합 용도 | 토큰 비용 |
|---|---|---|---|
| 서브에이전트 | 메인 세션이 관리 | 빠른 초점 작업, 결과만 반환 | 낮음 |
| Agent View | 사용자 수동 | 독립 태스크 위임 | 중간 |
| Agent Teams (실험) | 팀 리드 자율 | 토론·상호 반박 필요 작업, **3~5명 권장** | 높음 |
| Dynamic Workflows | 스크립트 계획 | 대규모 팬아웃 + 적대적 검증 | 매우 높음 |

같은 파일 병렬 편집, 조율 오버헤드가 이득을 넘는 작은 태스크, 순차만 필요한 작업엔 팀을 쓰지 말 것. — [Agents 문서](https://code.claude.com/docs/en/agents), [Agent teams](https://code.claude.com/docs/en/agent-teams), [Dynamic workflows](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code)

### 6.7 공식 검증 도구
- `claude plugin validate` — 플러그인/에이전트 정의 구조 검증.
- `claude plugin eval` — 플러그인 평가 스위트 작성·실행(발동/비발동 시나리오를 CI로 회귀 테스트).
- `/skill-doctor` — 스킬 트리거·구조 진단 리포트, `/doctor` — 스킬 목록 예산 진단.

---

## 7. 국내 기업 실무 사례 (대기업·유니콘·AI 스타트업)

국내 대기업, 유니콘, AI 스타트업의 기술블로그를 전수 조사한 결과. 실무 깊이 기준 상위 사례만 수록. (네이버 D2·쿠팡·카카오뱅크·삼성SDS·LG CNS 등은 도입 보도만 있고 지침 작성법 수준의 공개 기술 글은 없음 — 2026-09 기준)

### 7.1 토스 — Claude Code를 아키텍처로 이해하기
- [소프트웨어 3.0 시대를 맞이하며](https://toss.tech/article/software-3-0-era) (김용성, 2026-01): Claude Code 구성요소를 **레이어드 아키텍처에 매핑** — Slash Command=Controller(진입점), 서브에이전트=Service Layer(여러 스킬을 조율하는 워크플로), **Skills=Domain Component(단일 책임 원칙의 기능 단위)**, MCP=Infrastructure, CLAUDE.md=package.json(설정).
- 안티패턴 명명: **"God Skill"**(한 스킬에 과다 책임)과 **"Skill 폭발"**(→ Progressive Disclosure로 해결). CLAUDE.md에는 "자주 변하지 않는 원칙"만, 동적 정보는 대화로.
- [UI 테스트 자동화 일지](https://toss.tech/article/ai-driven-ui-test-automation): SDET/문서화/Git 3개 페르소나 에이전트 분업. [취약점 분석 자동화](https://toss.tech/article/vulnerability-analysis-automation-1): Supervisor+Discovery(필터)+Analysis 3-에이전트로 경로 50% 최적화.

### 7.2 컬리 — 오케스트레이션·지침 계층화의 최다 사례 (7편)
- [148,000번의 대화에서 배운 것](https://helloworld.kurly.com/blog/ai-orchestration-1) (이재서, 월 1,000억 토큰): 6단계 워크플로(탐색→계획→분해→구현→검증→기록, 어느 단계를 건너뛰어도 품질 저하), **Read:Write 10:1**(쓰기보다 탐색·분석에 10배 투자), AI 실패 패턴 5종은 모두 컨텍스트 품질 부족에서 기인.
- [AI 에이전트 15개 동시 운용기](https://helloworld.kurly.com/blog/ai-orchestration-2): Task DAG 병렬 실행(5일 작업→20분), 전용 스킬(dag-orchestrator, branch-hunt), 940세션/2,594 서브에이전트.
- [팀장의 하루 재설계](https://helloworld.kurly.com/blog/claude-code-redesign-my-day): **CLAUDE.md에서 `@../control_tower/shared.md` import로 규칙을 중앙화·계층화**, 세션 마무리마다 4유형 이슈 점검(자기개선 루프), "수집은 스크립트, 판단은 AI".
- [예측 가능한 바이브 코딩 전략](https://helloworld.kurly.com/blog/vibe-coding-with-claude-code): Root CLAUDE.md=온보딩, 디렉토리별=영역 특화 컨벤션. **"작게 나누고, 자주 확인하고, 반복은 스킬로 만들어라"** — 스킬 작성은 "자주 나는 실수" 발견에서 출발.
- [OMS 업무 방식](https://helloworld.kurly.com/blog/oms-claude-ai-workflow): 역할별(PO/TPM/BE/FE/Infra) `.claude/` 폴더 분리, **"지식(정적)"과 "행동(동적)"의 분리**로 예측 가능한 토큰 소비, API 스펙을 JSON DSL로 변환해 토큰 1/3 절감. 4인 팀이 "16인 조직처럼".

### 7.3 LINE/LY — 지침·하네스 전 층위 커버
- [Claude Code Action 리뷰 플랫폼화](https://techblog.lycorp.co.jp/ko/building-ai-code-review-platform-with-claude-code-action): **"무엇을(WHAT)"과 "어떻게(HOW)"의 엄격 분리** — 사용자 코멘트가 분석 초점을, 시스템 프롬프트가 출력 형식을 강제. Caller-Executor 분리(중앙 저장소가 정책·프롬프트·권한 통제). 1개월 32개 리포 확산.
- [Android CLI for AI agents](https://techblog.lycorp.co.jp/ko/android-cli-for-ai-agents-at-scale): 원본 CLI를 그대로 노출하지 않고 **래퍼(오류 정규화·출력 축약)+SKILL.md**로 감싸는 3계층. 스킬 description에 트리거 문구("Find usages of")와 "grep보다 정확" 규정을 넣어 선택 유도, CLAUDE.md에 폴백 규칙 명시.
- [멀티 에이전트 토론](https://techblog.lycorp.co.jp/ko/techverse2026-219): 제안자/도전자/조율자 3역할, 에이전트 간 통신은 구조화 JSON, 리스크 등급별 토론 강도 라우팅, A→B→A 진동 감지 시 상위보고. [AX 4단계 로드맵](https://techblog.lycorp.co.jp/ko/legacy-to-ai-driven-project-ax-roadmap): AI-Ready→Assist→Development→Review, 각 단계 승인 게이트.

### 7.4 스캐터랩 — 계층적 문서 구조의 전 주기 사례
- [Zeta Labs 개발기](https://blog.scatterlab.co.kr/zeta-labs-238416) (2026-07): 첫 커밋→출시 2개월, 개발자 1인당 에이전트 세션 6~7개. **3계층 문서**: ①AGENTS.md(불변 컨벤션) ②영역별 architecture 문서(작업 전 필독) ③기능 기획 문서(design "왜"→spec "어떻게"→구현 단방향 파이프라인). **방어 프롬프트**: 문서에 최종 수정 날짜를 기록, design이 spec보다 최신이면 에이전트가 개발자에게 확인. AI 교차 검증(Codex 코드는 Claude로 검증), 사람 리뷰는 인증·결제 등 고위험만.

### 7.5 기타 주목 사례
- **무신사** [Agent Teams 온콜 사례](https://techblog.musinsa.com/설-연휴에-claude-code-agent-teams를-데려갔습니다-fa96286f6954): **"팀 코딩 컨벤션을 AI가 읽을 수 있는 형식으로 문서화하는 것이 프롬프트 엔지니어링보다 효과적."**
- **라이너** [AX 엔지니어](https://liner.com/blog/liner-axee-kai): 기술 기반을 Advanced RAG + **Agent Harness** + Data Flywheel로 명시 — 하네스를 회사 기술 스택 용어로 공식화한 국내 사례. 프로덕션 3대 과제: 암묵지 내재화 / 사내 데이터 / Action Space.
- **하이퍼리즘** [서브에이전트 & SuperClaude](https://tech.hyperithm.com/claude_code_guides_2): frontmatter 작성법, 자동 위임엔 description에 "MUST use" 키워드, Hook으로 `.env`·`rm -rf` 차단하는 보안 하네스 패턴.
- **NHN** [AI 코딩 도구 보안 가이드](https://meetup.nhncloud.com/posts/396): 실제 사고 사례(무단 파일 삭제 등) 기반 DevContainer 격리 + 화이트리스트 방화벽 — 하네스의 "안전 경계" 설계 참고. 권한 승인을 건너뛰는 YOLO 모드 사용의 위험을 경고.
- **올리브영** [AI-DLC 워크숍](https://oliveyoung.tech/2026-04-16/oliveyoung-tech-ai-dlc-workshop/): 요구사항→개발→운영 전 단계 AI 통합 + 인간 인루프, "스펙 코딩"의 가치.
- **카카오** [바이브 코딩 바이블](https://tech.kakao.com/posts/696)·[Agentic Coding 실험](https://tech.kakao.com/posts/711): 실운영 서비스에서 평균 2배 생산성 검증, 수십 개 팀 확산. 저자 황민호(revfactory)가 곧 4장의 harness 플러그인·harness-100 제작자.
- **KB국민은행**: 개발 표준·보안 정책을 자동 반영하는 **자체 하네스**를 명시적으로 도입한 금융권 사례([보도](https://www.sedaily.com/article/20063160)). KB증권은 망분리 환경용 중앙 MCP 허브([기사](https://www.itdaily.kr/news/articleView.html?idxno=241257)).
- **센드버드**: [Anthropic 고객 사례](https://claude.com/customers/sendbird) — AI 상담 에이전트를 배포 전 대규모 시뮬레이션(회귀 테스트 도구)으로 검증하는 하네스, 주간 PR +128%.
- **업스테이지**: [사내 Claude Code 워크숍 저장소](https://github.com/team-attention/workshop-upstage) — vague(요구사항 명확화)·unknown(Unknown Unknown 탐지)·team-assemble(전문가 팀 구성)·wrap/compound(인사이트 축적) 등 메타 스킬 6종 설계가 흥미로움.

### 7.6 국내 사례의 수렴 패턴 (5대 축)
1. **계획-실행 분리**: 플랜 모드, 승인 게이트, design→spec 단방향 파이프라인.
2. **출력 구조화 강제**: 체크리스트, JSON 통신, 심각도 등급.
3. **지침의 계층화**: 공통 vs 서비스/디렉토리별, CLAUDE.md import, 가까운 문서 우선.
4. **반복 검증의 스킬화**: 자주 나는 실수 → 스킬/검증 에이전트로 전환.
5. **보안 경계**: DevContainer 격리, 훅 차단, 중앙 MCP 허브, 최소 권한 토큰.

공통 결론은 하나로 수렴: **"프롬프트를 잘 쓰는 것"보다 "AI가 읽을 수 있는 문서·규칙 체계와 검증 구조를 설계하는 것"이 팀 단위 레버리지다** (무신사·컬리·토스·우아한형제들·스캐터랩 공통).

---

## 8. 이 저장소(claude-devkits)에 대한 시사점

현재 구조(원칙 킷 common/frontend/backend + 기술 킷 nextjs/python/supabase, 에이전트 7개·스킬 12개)는 업계 수렴 패턴(스택별 자기완결 플러그인)과 방향이 일치한다. 리서치 기준으로 점검할 포인트:

- [x] **description에 "Not for…" 경계 추가** *(0.2.0 적용)*: 원칙 스킬(예: `backend-architecture`)과 기술 스킬(예: `python-guide`)의 트리거가 겹칠 수 있음 — 각 description에서 상호 경계를 명시.
- [x] **트리거 키워드 점검** *(0.2.0 — 전 킷 설치 시 합산 약 3,900자로 예산 내)*: 사용자가 실제 입력하는 문구(한국어 요청 패턴 포함)가 description에 있는지. 전 킷 설치 시 예산(~15,000자) 내인지 `/doctor`로 확인.
- [x] **에이전트의 스킬 로드 방식** *(0.2.0 — frontmatter `skills:` 프리로드로 전환)*: 본문의 "스킬을 항상 먼저 로드한다" 지시 대신 frontmatter `skills:` 프리로드가 더 결정적.
- [x] **에이전트 tools 미지정** *(0.2.0 — 전 에이전트 `Read, Grep, Glob` 읽기 전용)*: 현재 전부 생략(=전체 허용). 리뷰/자문형 에이전트(architect, dba-advisor, qa-*)는 읽기 전용(`Read, Grep, Glob`)으로 좁히는 것 검토.
- [x] **Gotchas 섹션 도입** *(0.2.0 — 기술 스킬 5종에 신설·시드)*: 각 기술 스킬(nextjs/python/supabase)에 운영하며 발견한 함정을 축적하는 섹션 신설.
- [x] **평가 시나리오 부재** *(0.2.0 — 킷별 `evals/` 발동/비발동 케이스 18개)*: 스킬별 발동/비발동 테스트 프롬프트 3개+를 저장소에 함께 커밋(회귀 테스트 기반).
- [x] **긴 스킬의 분리 검토** *(0.2.0 — supabase 키 마이그레이션·shadcn Radix 이관을 `references/`로 분리)*: `supabase-guide`(430줄), `react-query-guide`(342줄), `shadcn-ui`(337줄)는 500줄 미만이지만, 조건부 상세를 `references/`로 분리하면 트리거 후 로드 비용 절감 여지.
- [x] **검증 파이프라인** *(0.2.0 — validate 통과, eval 스위트 준비. eval 실행은 early access 활성화 필요)*: `claude plugin validate` + `claude plugin eval`로 킷별 발동/비발동 평가를 CI화. 지침 재검토 주기(3~6개월) 설정.
- [ ] **capability성 내용 은퇴 기준**: 기술 스킬 중 "LLM이 이미 아는" 일반 지식이 섞여 있으면 제거(스킬 없이 평가 통과 여부로 판단).

---

## 9. 출처

**Anthropic 공식**
- [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Lessons from building Claude Code: how we use skills](https://claude.com/blog/lessons-from-building-claude-code-how-we-use-skills)
- [How and when to use subagents](https://claude.com/blog/subagents-in-claude-code) · [Subagents 문서](https://code.claude.com/docs/en/sub-agents) · [Plugins 문서](https://code.claude.com/docs/en/plugins)
- [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents) · [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [anthropics/skills](https://github.com/anthropics/skills) (spec/template/예제)

- [Claude Code: Best practices](https://code.claude.com/docs/en/best-practices) · [Memory](https://code.claude.com/docs/en/memory) · [Agents 비교](https://code.claude.com/docs/en/agents) · [Agent teams](https://code.claude.com/docs/en/agent-teams)
- [How to create skills](https://claude.com/blog/how-to-create-skills-key-steps-limitations-and-examples) · [Large codebases best practices](https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start) · [Dynamic workflows](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code)
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) · [How Warp builds self-improving agents](https://claude.com/blog/how-warp-builds-self-improving-agents-on-claude) · [How Anthropic teams use Claude Code](https://claude.com/blog/how-anthropic-teams-use-claude-code)

**해외 권위자·커뮤니티**
- [Simon Willison: Claude Skills are awesome, maybe a bigger deal than MCP](https://simonwillison.net/2025/Oct/16/claude-skills/)
- [Jesse Vincent: Claude Code skills not triggering?](https://blog.fsck.com/2025/12/17/claude-code-skills-not-triggering/)
- [Philipp Schmid: 8 Tips for Writing Agent Skills](https://www.philschmid.de/agent-skills-tips)
- [Red Hat: Building skills for AI agents — pitfalls and best practices](https://next.redhat.com/2026/07/28/building-skills-for-ai-agents-pitfalls-and-best-practices/)
- [mgechev/skills-best-practices](https://github.com/mgechev/skills-best-practices) · [wshobson/agents](https://github.com/wshobson/agents) · [VoltAgent/awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents)
- [happyskills.ai: Why your skill never fires](https://happyskills.ai/blog/why-your-skill-never-fires/) · [cashandcache: 40+ Claude Skills failures](https://cashandcache.substack.com/p/i-analyzed-40-claude-skills-failures)

**국내**
- [우아한형제들: 하네스 엔지니어링으로 팀 맞춤형 AI 환경 구축하기](https://techblog.woowahan.com/26177/)
- [펀코딩(네이버 프리미엄): Harness로 '나만의 전문가 팀' 구축하기](https://contents.premium.naver.com/codetree/funcoding/contents/260412180547796xh)
- [채널톡: 하네스 엔지니어링이란?](https://channel.io/kr/blog/articles/what-is-harness-2611ddf1)
- [Databricks: AI 에이전트 하네스](https://www.databricks.com/kr/blog/ai-harness)
- [kt cloud: Claude Code 기본 구조 이해하기 — Agent·Skill·Context](https://tech.ktcloud.com/entry/2026-04-ktcloud-claude-agent-ai%ED%99%9C%EC%9A%A9-%EA%B0%9C%EB%85%90%EC%A0%95%EB%A6%AC)
- [Rost Glukhov: 에이전트 스킬 작성 — SKILL.md 구조와 모범 사례](https://www.glukhov.org/ko/ai-systems/hermes/authoring-hermes-skill/)
- 국내 기업 실무 사례(토스·컬리·LINE·스캐터랩·무신사·라이너 등 20여 편)의 출처는 **7장 본문 링크** 참조
- [GeekNews: gitagent](https://news.hada.io/topic?id=27859) · [GeekNews: AGENTS.md](https://news.hada.io/topic?id=22635)
