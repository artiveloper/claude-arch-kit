---
name: python-guide
description: >
  범용 Python 클린코드 실무 가이드 (프레임워크 무관).
  프로젝트 구조(src layout), 타입 힌트(mypy/pyright), 패키징(uv/poetry, pyproject.toml),
  린팅/포매팅(ruff), 테스트(pytest, fixture, mock 경계), 예외 계층 설계, asyncio 기본 패턴 포함.
  Python, 파이썬, type hint, 타입 힌트, pyproject.toml, ruff, mypy, pytest, asyncio,
  가상환경, venv 관련 작업이나 코드 리뷰 시 참조. FastAPI/Django 등 특정 프레임워크 실무는
  별도 기술 스킬(예: fastapi-guide)을 우선 참조하고, 이 스킬은 언어 차원의 공통 기반으로 사용.
---

# Python 클린코드 가이드

> 원칙: 타입과 테스트로 계약을 명시하고, 런타임에 발견될 오류를 정적으로 앞당긴다.

---

## 1. 프로젝트 구조 (src layout)

```
my-project/
├── pyproject.toml
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── domain/          # 순수 로직 (외부 의존성 없음)
│       ├── infra/            # DB, 외부 API 등 I/O 어댑터
│       └── api/               # 진입점 (CLI, HTTP 라우터 등)
└── tests/
    ├── unit/
    └── integration/
```

- `src/` 레이아웃을 사용한다. 패키지를 루트에 바로 두면 테스트가 설치되지 않은 로컬 코드를 임포트해 "테스트는 통과했는데 설치 후엔 깨지는" 문제가 생긴다.
- `domain/`(순수 로직)과 `infra/`(I/O)를 분리하면 단위 테스트에서 mock 경계가 명확해진다.

---

## 2. 타입 힌트 & 정적 검사

### 기본 원칙

```python
# ✅ 공개 함수/메서드 시그니처는 항상 타입 명시
def calculate_total(items: list[Item], discount: float = 0.0) -> Decimal:
    ...

# ❌ Any는 경계를 무너뜨림 — 외부 미지 타입(예: 서드파티 응답 파싱 직후)에서만 임시 허용
def parse(data: Any) -> Any:
    ...
```

- Python 3.9+ 는 `list[int]`, `dict[str, int]`, `X | None` 내장 제네릭을 사용한다 (`typing.List` 등 구식 표기 지양).
- `Optional[X]`보다 `X | None`을 선호한다 (3.10+).
- 도메인 경계를 넘는 데이터는 `dataclass` 또는 `pydantic.BaseModel`로 구조화한다. 딕셔너리를 그대로 넘기면 어떤 키가 필수인지 타입 체커가 알 수 없다.

### mypy 설정 (`pyproject.toml`)

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_unused_ignores = true
disallow_untyped_defs = true
```

- 신규 프로젝트는 처음부터 `strict = true`로 시작한다. 나중에 강도를 올리는 것보다 처음부터 유지하는 비용이 훨씬 낮다.
- 레거시 코드에 도입할 때는 모듈 단위로 `[[tool.mypy.overrides]]`를 걸어 점진적으로 좁혀간다.

---

## 3. 패키징 (`pyproject.toml`)

```toml
[project]
name = "my-package"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest", "pytest-cov", "ruff", "mypy"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

- `requirements.txt` 대신 `pyproject.toml` 단일 파일로 의존성·빌드·툴 설정을 통합한다.
- 의존성 관리는 `uv`(권장, 락파일 `uv.lock` + 빠른 설치) 또는 `poetry`를 사용한다. `pip freeze`로 만든 `requirements.txt`는 개발/프로덕션 의존성이 뒤섞이기 쉽다.

```bash
uv sync              # 락파일 기준 설치
uv add requests       # 의존성 추가 + pyproject.toml/lock 갱신
uv run pytest         # 가상환경 활성화 없이 실행
```

- 버전은 상한을 과도하게 걸지 않는다 (`requests>=2.0` 정도로 충분). 상한을 촘촘히 걸면 다른 패키지와 충돌하는 "dependency hell"을 자초한다. 락파일이 실제 재현성을 보장한다.

---

## 4. 린팅/포매팅 (ruff)

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]
# E/F: pycodestyle/pyflakes 기본, I: import 정렬, UP: 최신 문법 강제
# B: bugbear(흔한 실수), SIM: 불필요한 복잡도 단순화 제안
```

- `ruff`는 flake8 + isort + pyupgrade 등을 대체하는 단일 고속 도구다. `black` 포매팅도 `ruff format`으로 통합 가능하므로 별도 도구를 여럿 두지 않는다.
- CI에는 `ruff check .`와 `ruff format --check .`를 모두 넣는다. 로컬에서만 돌리면 포매팅 드리프트가 누적된다.

---

## 5. 테스트 (pytest)

### fixture와 mock 경계

```python
# ✅ 순수 로직은 mock 없이 직접 테스트
def test_calculate_total_applies_discount():
    items = [Item(price=Decimal("100"))]
    assert calculate_total(items, discount=0.1) == Decimal("90")

# ✅ 외부 I/O 경계만 mock — 도메인 로직까지 mock하면 테스트가 구현을 그대로 베낀 것이 됨
def test_order_service_calls_payment_gateway(mocker):
    gateway = mocker.patch("my_package.infra.payment.PaymentGateway.charge")
    OrderService(gateway).checkout(order)
    gateway.assert_called_once_with(order.total)
```

- `domain/`은 mock 없이 직접 테스트한다 (순수 함수이므로 가능해야 정상). mock이 많이 필요하다면 `domain/`에 I/O가 섞여 있다는 신호다.
- `infra/` 어댑터는 실제 대상(테스트 DB, 테스트 컨테이너)으로 통합 테스트하거나, 계약(contract)이 검증된 fake로 대체한다. 프로덕션 코드와 무관하게 항상 True를 리턴하는 stub mock은 회귀를 못 잡는다.
- fixture는 `conftest.py`에 스코프(`function`/`module`/`session`)를 명시해 재사용하되, 테스트 간 상태가 새는지(특히 `session` 스코프의 가변 객체) 주의한다.

### 커버리지보다 경계

- 라인 커버리지 수치를 목표로 삼지 않는다. 대신 "이 함수가 잘못된 값을 받으면 무슨 일이 일어나는가"를 우선 테스트한다 (예외 케이스, 빈 입력, 경계값).

---

## 6. 에러 처리

```python
# ✅ 도메인별 예외 계층 — 호출자가 구체적으로 분기 가능
class OrderError(Exception):
    """주문 도메인의 베이스 예외."""

class InsufficientStockError(OrderError):
    def __init__(self, item_id: str, requested: int, available: int):
        self.item_id = item_id
        super().__init__(f"stock shortage: {item_id} ({requested}/{available})")

# ❌ 광범위한 catch — 원인 불명의 버그를 삼킴
try:
    process_order(order)
except Exception:
    pass
```

- 라이브러리 최상위 예외(`OrderError`)를 두고 구체적 예외가 이를 상속하게 하면, 호출자는 필요에 따라 넓게(`except OrderError`) 또는 좁게(`except InsufficientStockError`) 잡을 수 있다.
- `except Exception: pass`처럼 원인을 숨기는 catch는 지양한다. 최소한 로깅하고, 재발생(`raise`)하거나 명시적으로 변환한 예외를 던진다.
- 복구 불가능한 프로그래밍 오류(잘못된 인자 등)는 커스텀 예외 대신 `assert`나 표준 예외(`ValueError`, `TypeError`)로 즉시 실패시킨다 — 굳이 감싸서 늦게 터지게 만들지 않는다.

---

## 7. 비동기 (asyncio)

```python
# ✅ I/O 바운드 작업은 gather로 병렬화
async def fetch_all(ids: list[str]) -> list[Result]:
    return await asyncio.gather(*(fetch_one(id_) for id_ in ids))

# ❌ 동기 블로킹 호출을 async 함수 안에 그대로 두면 이벤트 루프 전체가 멈춤
async def bad_fetch():
    return requests.get(url)  # 블로킹 — httpx.AsyncClient 등으로 교체

# ✅ 블로킹이 불가피한 코드(레거시 sync 라이브러리)는 스레드로 위임
async def wrap_blocking():
    return await asyncio.to_thread(legacy_sync_call)
```

- `async def` 안에서 블로킹 I/O(동기 `requests`, 동기 DB 드라이버 등)를 호출하면 다른 코루틴이 전부 대기한다. 반드시 비동기 클라이언트(`httpx`, `asyncpg` 등)로 교체하거나 `asyncio.to_thread`로 위임한다.
- CPU 바운드 작업(무거운 연산)은 `asyncio`로 빨라지지 않는다 — `multiprocessing`이나 별도 워커로 분리한다. asyncio는 동시성(concurrency)이지 병렬성(parallelism)이 아니다.
- 여러 독립적인 await를 순차적으로 나열하지 않는다. 서로 의존하지 않으면 `asyncio.gather`로 묶어 지연시간을 합이 아닌 최대값으로 줄인다.

---

## 8. 체크리스트

### 구조
- [ ] `src/` 레이아웃, `domain/`과 `infra/` 분리
- [ ] `pyproject.toml` 단일 파일로 의존성/빌드/툴 설정 통합

### 타입/린트
- [ ] 공개 함수 시그니처에 타입 힌트, `Any` 최소화
- [ ] mypy `strict = true` (레거시는 모듈별 점진 적용)
- [ ] ruff lint + format이 CI에 포함됨

### 테스트
- [ ] `domain/` 로직은 mock 없이 직접 테스트
- [ ] `infra/` 경계만 mock/fake, 커버리지 수치보다 예외·경계값 우선

### 에러/비동기
- [ ] 도메인별 예외 계층 존재, 광범위한 `except Exception: pass` 없음
- [ ] async 함수 내 블로킹 호출 없음, 독립적 await는 `gather`로 병렬화
