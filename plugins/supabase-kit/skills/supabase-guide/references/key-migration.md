# Supabase 레거시 API 키 마이그레이션

> `supabase-guide` 스킬의 참조 문서. 레거시 `anon`/`service_role`(JWT) 키를 신규 Publishable/Secret 키로 교체할 때만 읽는다.

## 목차
1. [레거시 키 vs 신규 키 비교표](#레거시-키-vs-신규-키)
2. [발급 / 교체 절차](#발급--교체-절차)

### 레거시 키 vs 신규 키

| 구분 | 레거시 — 사용 금지 | 신규 — 사용 |
|------|------------------|-------------|
| 공개용 | `anon` (JWT, `eyJ...`) | **Publishable** (`sb_publishable_...`) |
| 서버용 | `service_role` (JWT, `eyJ...`) | **Secret** (`sb_secret_...`) |
| 형식 | JWT — 디코딩 시 project ref/exp 등 노출 | 불투명 문자열 |
| 개별 폐기 | 불가 (JWT secret 교체 = 전체 키 동시 무효화) | 가능 (키 단위 생성/삭제) |
| 다중 발급 | 불가 (각각 1개 고정) | Secret 키 다중 발급 (서비스/환경별 분리) |
| 유출 대응 | 전체 롤오버 필요 → 전 서비스 다운타임 | 해당 키만 삭제 후 재발급 |
| 상태 | **deprecated 예정** (폐기 일정은 Supabase 공지 확인) | 표준 |

- 레거시 키는 대시보드에서 명시적으로 disable 하기 전까지 계속 동작한다 → 마이그레이션 기간 동안 병행 가능.
- 신규 키를 발급해도 레거시 키는 자동 폐기되지 않는다. 교체 완료 후 **직접 disable** 해야 한다.

### 발급 / 교체 절차

1. Dashboard → Project Settings → **API Keys** 탭에서 Publishable / Secret 키 생성
2. 클라이언트(브라우저·SSR)에는 Publishable 키만 주입
3. 서버 전용 경로(Server Action / Route Handler / Edge Function)에만 Secret 키 주입
4. 전 환경(local / staging / production, CI Secrets, Vercel 환경변수)에서 레거시 키 참조 제거
   — **변수명까지 함께 교체한다.** `SUPABASE_SERVICE_ROLE_KEY` 가 남아 있으면 언젠가 레거시 키가 다시 주입된다.
5. 레거시 키 사용량 0 확인 후 대시보드에서 legacy keys **disable**

> Secret 키 삭제는 되돌릴 수 없다. 삭제 전 대체 키로 배포가 완료됐는지 확인한다.
> 유출 시에는 재발급 → 배포 → 기존 키 삭제. 다른 Secret 키에는 영향이 없다.
