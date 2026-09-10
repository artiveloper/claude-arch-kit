---
name: kit-improve
description: >
  claude-devkits 마켓플레이스의 킷 품질 개선 오케스트레이터. 스킬 수정, 에이전트 수정,
  description 개선, Gotcha 추가, 새 킷/스킬/에이전트 추가, 킷 감사, 릴리스 준비,
  그리고 "다시 실행", "재검토", "업데이트", "보완", "이전 결과 개선" 같은 후속 요청 시 반드시 사용.
  kit-editor(편집)와 kit-auditor(감사)를 생성-검증 루프로 조율한다.
  단순 조회성 질문(파일 위치, 규칙 내용)은 이 스킬 없이 직접 답해도 된다.
---

# Kit Improve — 킷 개선 오케스트레이터

**실행 모드: 하이브리드** — 단일 파일 소규모 수정은 서브 에이전트(순차), 여러 킷에 걸치거나 신규 킷 추가면 에이전트 팀. 판단 기준은 Phase 2에 명시.

## Phase 0: 컨텍스트 확인

1. `_workspace/` 존재 확인:
   - 존재 + 부분 수정 요청 → **부분 재실행**: 해당 단계 에이전트만 재호출, 이전 요약을 입력으로 전달
   - 존재 + 새 요청 → 기존을 `_workspace_prev/`로 이동 후 **새 실행**
   - 미존재 → **초기 실행**
2. `git status`로 미커밋 변경 확인 — 있으면 사용자에게 이번 작업과의 관계를 확인한다.

## Phase 1: 변경 분석

- 요청에서 대상 킷·파일·변경 유형(스킬 본문 / description / 에이전트 / 신규 추가 / 감사만 / 릴리스만)을 특정한다.
- 애매하면 후보를 제시하고 확인받는다. 감사만 요청이면 Phase 3으로 직행한다.
- 분석 결과를 `_workspace/00_plan.md`에 기록한다.

## Phase 2: 실행 모드 결정 + 편집

| 조건 | 모드 |
|------|------|
| 파일 1~2개, 단일 킷 | **서브 에이전트**: kit-editor 호출(Agent 도구, model: "opus") → 완료 후 Phase 3 |
| 3개 킷 이상 또는 신규 킷/에이전트 추가 | **에이전트 팀**: kit-editor + kit-auditor 팀 구성, 태스크 할당. editor가 킷 단위로 완료할 때마다 auditor가 점진 감사(incremental) |

- kit-editor에게 전달: 변경 요청 원문 + `00_plan.md` + (재작업 시) auditor 발견사항.
- editor 산출: 변경 파일 목록 + 요약 → `_workspace/01_editor_changes.md`.

## Phase 3: 감사

- kit-auditor 호출(model: "opus"). 입력: 변경 파일 목록(diff 범위).
- auditor는 audit.py + validate 실행 후 판단형 검사를 수행, 리포트를 `_workspace/02_audit_report.md`에 기록.

## Phase 4: 수정 루프

- FAIL 존재 → 발견사항을 kit-editor에 전달해 수정 → 재감사. **최대 2회 반복**.
- 2회 후에도 FAIL이 남으면 남은 항목과 양측 근거를 사용자에게 보고하고 판단을 받는다.
- WARN은 수정하지 않고 최종 보고에 포함한다(사용자 판단).

## Phase 5: 릴리스 준비

1. 변경된 킷의 plugin.json `version` bump (editor에게 위임)
2. README 표·marketplace.json 동기화 확인 (audit 리포트의 동기화 WARN 기준)
3. 저장소 CLAUDE.md의 하네스 변경 이력 테이블에 기록
4. 커밋 메시지 초안 제시 — **커밋/푸시는 사용자 확인 후에만**

## 데이터 전달

- 파일 기반: `_workspace/{순번}_{에이전트}_{산출물}.md` (감사 추적용, 보존)
- 팀 모드에서는 태스크 기반(조율) + 메시지 기반(editor↔auditor 발견사항 교환) 병행

## 에러 핸들링

- 에이전트 실패 시 1회 재시도, 재실패 시 해당 단계 결과 없이 진행하고 최종 보고에 누락 명시
- editor·auditor 의견 상충 시 1회 상호 토론 후 미합의면 양측 근거를 병기해 사용자에게 이관
- audit.py 실행 불가 시 auditor가 수동 검사로 대체하고 "스크립트 미실행"을 리포트에 명시

## 테스트 시나리오

**정상 흐름**: "supabase-guide에 Storage RLS Gotcha 추가해줘" → Phase 1(supabase-kit, Gotcha 추가로 특정)
→ 서브 모드: editor가 Gotchas 섹션에 1 bullet 추가 → auditor 통과 → version bump(patch) → 커밋 메시지 제안.

**에러 흐름**: "새 fastapi-kit 만들어줘" → 팀 모드 → editor가 python-guide와 겹치는 description 작성
→ auditor FAIL(트리거 near-miss: "FastAPI 라우터 구조" 요청이 양쪽 발동) → editor가 양쪽 description에
상호 경계 추가 → 재감사 통과. 2회 루프 후에도 남으면 사용자 이관.

**트리거 검증 기준** (이 스킬 자신):
- should: "스킬 설명 고쳐줘", "backend-kit 감사해줘", "새 킷 추가하자", "아까 수정한 거 보완해줘"
- should-NOT: "python-guide 내용이 뭐야?"(조회 — 직접 답), "이 프로젝트 코드 리뷰해줘"(킷 개선 아님 — 설치된 킷의 에이전트 소관)
