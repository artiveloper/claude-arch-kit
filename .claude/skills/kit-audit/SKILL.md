---
name: kit-audit
description: >
  claude-devkits 변경분 감사 절차. 스킬/에이전트/매니페스트 수정 후 검증, 감사, 검수,
  릴리스 전 점검, 트리거 충돌 확인 요청 시 반드시 사용. 파일 작성/수정 방법은 kit-authoring 소관.
---

# Kit Audit — 변경분 감사 절차

## 1. 기계적 검사 (스크립트 실행)

```bash
python3 .claude/skills/kit-audit/scripts/audit.py .
claude plugin validate .          # 마켓플레이스 전체
claude plugin validate ./plugins/<변경된 킷>
```

audit.py가 검사하는 것: frontmatter YAML 파싱, description 존재·1,024자·콜론 함정·경계 휴리스틱,
본문 500줄, 타 킷 참조의 미설치 가드, references/ 목차, 에이전트 tools/skills 프리로드 유효성,
plugin.json version, eval 커버리지, README 에이전트명 동기화, description 예산 합산.

- 스크립트 출력의 WARN은 오탐 여부를 판단해 걸러서 보고한다 (예: 의도적으로 유지하는 근접 줄 수).
- `claude plugin validate`는 관대하다 — audit.py가 잡는 YAML 함정을 통과시키니 둘 다 실행한다.

## 2. 판단형 검사 (직접 읽고 평가)

기계적으로 못 잡는 것들. 변경된 파일에 대해서만 수행한다:

| 항목 | 확인 방법 |
|------|----------|
| **트리거 near-miss** | 변경된 description을 두고, 인접 스킬(원칙↔기술, 같은 킷 내)과 겹치는 요청 문장을 3개 이상 상상해 어느 쪽이 발동해야 하는지 판별한다. 판별이 안 되면 경계 문구 보강 필요 |
| **킷 성격 위반** | 원칙 킷에 스택 종속 내용이, 기술 킷에 범용 판단 기준이 들어갔는지 |
| **God Skill 징후** | 한 스킬이 Anthropic 9범주(레퍼런스/검증/분석/자동화/스캐폴딩/품질/CI/런북/인프라) 중 둘 이상에 걸치는지 |
| **일반 지식 혼입** | LLM이 이미 아는 내용(언어 기본 문법 등)이 새로 들어갔는지 |
| **Gotchas 위치** | 새 함정이 별도 섹션이 아니라 기존 Gotchas 섹션에 추가됐는지 |
| **에이전트 단일 책임** | 에이전트 역할이 늘었으면 분리가 낫지 않은지 |

## 3. 리포트 형식

```
FAIL  <파일>:<라인>  <문제>  →  <권장 수정>
WARN  <파일>:<라인>  <문제>  →  <권장 수정>
변경 외 발견: (이번 수정 범위 밖 — 별도 처리)
미검증: (plugin eval 실행 등 이 환경에서 불가한 항목)
```

FAIL은 머지 차단, WARN은 권장. 발견 없으면 "통과 — 검사 항목 N개" 한 줄.

## 4. 릴리스 전 최종 점검

- [ ] audit.py FAIL 0 + validate 통과
- [ ] 변경된 킷의 plugin.json `version` bump 됐는가 (안 올리면 전파 안 됨)
- [ ] 트리거 영향 변경이면 evals 케이스 추가/갱신됐는가
- [ ] README 표·상시 토큰 표가 실제와 일치하는가 (description 총량이 크게 변했으면 갱신)
