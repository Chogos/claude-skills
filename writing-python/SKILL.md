---
name: writing-python
description: Python development best practices. Use when writing, modifying, or reviewing .py files, pyproject.toml, setup.py, requirements.txt, pytest tests, or Python package configurations.
---

# Python Development Best Practices

## Project Structure

Use the `src/` layout. It prevents accidental imports from the working directory and forces installation before testing.

```
my-project/
├── pyproject.toml          # all metadata, deps, tool config
├── src/
│   └── my_package/
│       ├── __init__.py     # re-export public API only
│       ├── core.py
│       ├── models.py
│       └── utils.py
├── tests/
│   ├── conftest.py
│   ├── test_core.py
│   └── test_models.py
└── README.md
```

- One `pyproject.toml` for everything: metadata, dependencies, build system, tool config.
- Keep `__init__.py` minimal — re-exports and `__all__` only.
- No `setup.py` or `setup.cfg` in new projects.

Minimal `pyproject.toml`:

```toml
[project]
name = "my-package"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "ruff>=0.7",
    "mypy>=1.13",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

## Type Hints

Type-hint all public function signatures — parameters and return types.

- Use built-in generics: `list[str]`, `dict[str, int]`, `tuple[int, ...]`, `set[float]`.
- `X | None` over `Optional[X]`. `X | Y` over `Union[X, Y]`.
- `TypeAlias` for complex types.
- `TypeVar` for generic functions and classes.
- `Protocol` over `ABC` — structural subtyping, no inheritance required.
- Never use `Any` without a comment explaining why.

```python
from typing import TypeAlias, TypeVar, Protocol

JsonDict: TypeAlias = dict[str, "JsonValue"]
JsonValue: TypeAlias = str | int | float | bool | None | list["JsonValue"] | JsonDict

T = TypeVar("T")

class Serializable(Protocol):
    def to_dict(self) -> dict[str, object]: ...

def first(items: list[T]) -> T | None:
    return items[0] if items else None
```

See [type-hints-cheatsheet.md](type-hints-cheatsheet.md) for the full reference.

## Error Handling

- Define a project base exception. Derive specific exceptions from it.
- Catch specific exceptions — never bare `except:` or `except Exception:` without re-raise.
- Re-raise with `from` to preserve the chain.
- `contextlib.suppress()` for intentional ignoring.
- Never swallow exceptions silently.

```python
class AppError(Exception):
    """Base for all project exceptions."""

class NotFoundError(AppError):
    def __init__(self, resource: str, id: str) -> None:
        super().__init__(f"{resource} {id!r} not found")
        self.resource = resource
        self.id = id

class ValidationError(AppError):
    pass

# Catching and re-raising
import httpx

async def fetch_user(user_id: str) -> dict[str, object]:
    try:
        resp = await client.get(f"/users/{user_id}")
        resp.raise_for_status()
        return resp.json()
    except httpx.HTTPStatusError as exc:
        if exc.response.status_code == 404:
            raise NotFoundError("user", user_id) from exc
        raise AppError(f"HTTP {exc.response.status_code}") from exc

# Intentional suppression
from contextlib import suppress
with suppress(FileNotFoundError):
    Path("cache.json").unlink()
```

## Data Modeling

### dataclasses — internal data

Use `frozen=True` for immutable value objects. `__post_init__` for validation. `field()` for defaults.

```python
from dataclasses import dataclass, field

@dataclass(frozen=True, slots=True)
class Coordinate:
    lat: float
    lon: float

    def __post_init__(self) -> None:
        if not -90 <= self.lat <= 90:
            raise ValueError(f"lat must be -90..90, got {self.lat}")
        if not -180 <= self.lon <= 180:
            raise ValueError(f"lon must be -180..180, got {self.lon}")

@dataclass(slots=True)
class Config:
    host: str = "localhost"
    port: int = 8080
    tags: list[str] = field(default_factory=list)
```

### Pydantic — external/untrusted data

Use for API boundaries, config loading, anything requiring validation and serialization.

```python
from pydantic import BaseModel, Field, model_validator

class CreateUserRequest(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: str = Field(pattern=r"^[\w.+-]+@[\w-]+\.[\w.]+$")
    age: int = Field(ge=0, le=150)

    @model_validator(mode="after")
    def normalize_email(self) -> "CreateUserRequest":
        self.email = self.email.lower().strip()
        return self

# Serialize / deserialize
user = CreateUserRequest.model_validate({"name": "Ada", "email": "ADA@EXAMPLE.COM", "age": 36})
data = user.model_dump(exclude_none=True)
```

## Virtual Environments & Dependencies

**uv** (preferred) — fast, replaces pip, pip-tools, virtualenv, and pyenv.

```bash
uv init my-project           # scaffold pyproject.toml + .venv
uv add httpx pydantic        # add to [project.dependencies]
uv add --group dev pytest ruff mypy  # add to [dependency-groups]
uv sync                      # install all deps from lockfile
uv lock                      # regenerate uv.lock
uv run pytest                # run inside .venv without activating
uv run ruff check src/
```

**poetry** (alternative):

```bash
poetry new my-project --src
poetry add httpx pydantic
poetry add --group dev pytest ruff mypy
poetry install
poetry run pytest
```

Rules:
- Always commit the lockfile (`uv.lock` or `poetry.lock`).
- Separate dependency groups: `dev`, `test`, `docs`.
- Pin direct dependencies to minimum compatible version: `httpx>=0.27`.
- Never install into the system Python.

## Linting & Formatting

**ruff** replaces flake8, isort, black, pycodestyle, and pydocstyle.

```bash
ruff format src/ tests/      # auto-format (replaces black)
ruff check src/ tests/ --fix  # lint + autofix (replaces flake8/isort)
mypy src/                    # type check
```

Configure in `pyproject.toml`:

```toml
[tool.ruff]
target-version = "py311"
line-length = 99

[tool.ruff.lint]
select = [
    "E", "W",    # pycodestyle
    "F",         # pyflakes
    "I",         # isort
    "N",         # pep8-naming
    "UP",        # pyupgrade
    "B",         # bugbear
    "SIM",       # simplify
    "RUF",       # ruff-specific
]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
```

Run mypy with `--strict` to enforce type completeness.

## Testing (pytest)

- Files: `test_*.py` or `*_test.py`. Functions: `test_*`. No class-based tests unless grouping is needed.
- Shared fixtures go in `conftest.py`.
- `@pytest.mark.parametrize` for data-driven tests.
- `pytest-cov` for coverage.
- `tmp_path` (built-in fixture) for file tests.
- `monkeypatch` for env vars, attributes, and function overrides.

```python
import pytest
from my_package.core import parse_duration

@pytest.mark.parametrize(
    ("input_str", "expected_seconds"),
    [
        ("30s", 30),
        ("5m", 300),
        ("2h", 7200),
        ("1d", 86400),
    ],
    ids=["seconds", "minutes", "hours", "days"],
)
def test_parse_duration(input_str: str, expected_seconds: int) -> None:
    assert parse_duration(input_str) == expected_seconds

def test_parse_duration_invalid() -> None:
    with pytest.raises(ValueError, match="invalid duration"):
        parse_duration("abc")

def test_write_config(tmp_path: Path) -> None:
    config_file = tmp_path / "config.toml"
    write_config(config_file, {"key": "value"})
    assert config_file.read_text() == 'key = "value"\n'

def test_api_key_from_env(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("API_KEY", "test-key-123")
    assert get_api_key() == "test-key-123"
```

See [patterns/testing-patterns.md](patterns/testing-patterns.md) for advanced patterns.

## Logging

```python
import logging

logger = logging.getLogger(__name__)
```

- Use `logging.getLogger(__name__)` in every module — never the root logger.
- Configure once at application startup, not in libraries:
  ```python
  logging.basicConfig(
      level=logging.INFO,
      format="%(asctime)s %(levelname)s %(name)s %(message)s",
  )
  ```
- For JSON structured logging (production), use `structlog`:
  ```python
  import structlog

  structlog.configure(
      processors=[
          structlog.stdlib.add_log_level,
          structlog.processors.TimeStamper(fmt="iso"),
          structlog.processors.JSONRenderer(),
      ],
      wrapper_class=structlog.stdlib.BoundLogger,
  )
  log = structlog.get_logger()
  log.info("order_created", order_id="abc123", total=42.50)
  ```
- Levels: `DEBUG` for development tracing, `INFO` for business events, `WARNING` for recoverable issues, `ERROR` for failures needing attention.
- Never log secrets, tokens, passwords, or PII.

## Async (asyncio)

- `async def` for I/O-bound operations (HTTP, DB, file, network).
- `asyncio.run()` as the single entry point.
- `asyncio.TaskGroup` (3.11+) for structured concurrency — exceptions propagate cleanly.
- `async with` for resource management (sessions, connections).
- `asyncio.to_thread()` to offload blocking/sync code.
- `pytest-asyncio` for async tests. Requires `asyncio_mode = "auto"` in `pyproject.toml` since 0.23+:
  ```toml
  [tool.pytest-asyncio]
  asyncio_mode = "auto"
  ```

```python
import asyncio
import httpx

async def fetch_all(urls: list[str]) -> list[str]:
    async with httpx.AsyncClient() as client:
        async with asyncio.TaskGroup() as tg:
            tasks = [tg.create_task(client.get(url)) for url in urls]
        return [t.result().text for t in tasks]

async def main() -> None:
    results = await fetch_all(["https://example.com", "https://httpbin.org/get"])
    for r in results:
        print(r[:100])

if __name__ == "__main__":
    asyncio.run(main())
```

See [patterns/async-patterns.md](patterns/async-patterns.md) for producer-consumer, rate limiting, and signal handling patterns.

## Packaging

- Build system in `pyproject.toml` — `hatchling`, `setuptools`, or `flit-core`.
- Single-source `__version__` via `pyproject.toml` `[project] version` field.
- Add `py.typed` marker file to `src/my_package/py.typed` (empty file) so type checkers recognize the package.
- Use `hatch version` or `importlib.metadata.version("my-package")` to read version at runtime.

```python
# src/my_package/__init__.py
from importlib.metadata import version

__version__ = version("my-package")
```

## New project workflow

```
- [ ] uv init my-project && cd my-project
- [ ] Verify src/ layout with __init__.py
- [ ] Add dependencies: uv add <deps>
- [ ] Add dev deps: uv add --group dev pytest ruff mypy pytest-cov
- [ ] Configure ruff and mypy in pyproject.toml
- [ ] Create tests/conftest.py with shared fixtures
- [ ] Add py.typed marker to src/my_package/
- [ ] Write first test, run validation loop
- [ ] Set up CI (GitHub Actions)
```

## Validation loop

Run after every change. Fix errors in order — formatting issues often disappear after `ruff format`.

1. `ruff format src/ tests/` — auto-format
2. `ruff check src/ tests/ --fix` — lint and autofix
3. `mypy src/` — type check (fix all errors, no `type: ignore` without comment)
4. `pytest --cov=src/ --cov-report=term-missing` — run tests, check coverage
5. Repeat until all four pass clean

## Deep-dive references

- **Project scaffolding**: See [patterns/project-scaffolding.md](patterns/project-scaffolding.md) for full pyproject.toml, CI workflow, conftest fixtures
- **Testing patterns**: See [patterns/testing-patterns.md](patterns/testing-patterns.md) for parametrize, mocking, async tests, fixtures
- **Async patterns**: See [patterns/async-patterns.md](patterns/async-patterns.md) for TaskGroup, queues, rate limiting, signal handling
- **Type hints**: See [type-hints-cheatsheet.md](type-hints-cheatsheet.md) for the full typing reference

## Official references

- [PEP 8 — Style Guide](https://peps.python.org/pep-0008/) — naming, formatting, imports
- [typing module docs](https://docs.python.org/3/library/typing.html) — full typing reference
- [pytest docs](https://docs.pytest.org/en/stable/) — fixtures, parametrize, plugins, configuration
- [ruff docs](https://docs.astral.sh/ruff/) — rules, configuration, formatter
- [mypy docs](https://mypy.readthedocs.io/en/stable/) — type checking configuration and error codes
- [Pydantic docs](https://docs.pydantic.dev/latest/) — models, validators, serialization