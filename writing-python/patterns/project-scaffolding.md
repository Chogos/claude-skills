# Project Scaffolding Patterns

## Contents
- Complete pyproject.toml (library)
- Complete pyproject.toml (application)
- Full src/ layout with re-exports
- conftest.py with common fixtures
- GitHub Actions CI workflow

## pyproject.toml — library

Libraries are installed as dependencies. They need build metadata, version constraints, and a `py.typed` marker.

```toml
[project]
name = "my-library"
version = "1.2.0"
description = "A reusable Python library"
readme = "README.md"
license = "MIT"
requires-python = ">=3.11"
authors = [
    { name = "Your Name", email = "you@example.com" },
]
classifiers = [
    "Development Status :: 4 - Beta",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
    "Programming Language :: Python :: 3.14",
    "Typing :: Typed",
]
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0,<3",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "pytest-asyncio>=0.24",
    "ruff>=0.7",
    "mypy>=1.13",
]

[project.urls]
Repository = "https://github.com/org/my-library"
Documentation = "https://my-library.readthedocs.io"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_library"]

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
    "PTH",       # pathlib
    "TCH",       # type-checking imports
]

[tool.ruff.lint.isort]
known-first-party = ["my_library"]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
plugins = ["pydantic.mypy"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers --strict-config -ra"
markers = [
    "slow: marks tests as slow (deselect with '-m not slow')",
    "integration: marks integration tests",
]

[tool.coverage.run]
source = ["my_library"]
branch = true

[tool.coverage.report]
show_missing = true
fail_under = 80
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "if __name__",
]
```

## pyproject.toml — application

Applications run directly. They use exact lockfiles, entry points, and often stricter pinning.

```toml
[project]
name = "my-app"
version = "0.1.0"
description = "A Python application"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "sqlalchemy>=2.0",
    "pydantic-settings>=2.0",
    "structlog>=24.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "pytest-asyncio>=0.24",
    "pytest-httpx>=0.34",
    "ruff>=0.7",
    "mypy>=1.13",
]

[project.scripts]
my-app = "my_app.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_app"]

[tool.ruff]
target-version = "py312"
line-length = 99

[tool.ruff.lint]
select = ["E", "W", "F", "I", "N", "UP", "B", "SIM", "RUF", "PTH", "TCH"]

[tool.ruff.lint.isort]
known-first-party = ["my_app"]

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
plugins = ["pydantic.mypy"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers --strict-config -ra"
asyncio_mode = "auto"

[tool.coverage.run]
source = ["my_app"]
branch = true

[tool.coverage.report]
show_missing = true
fail_under = 80
```

## Full src/ layout

```
my-library/
├── pyproject.toml
├── uv.lock
├── src/
│   └── my_library/
│       ├── __init__.py         # public API re-exports
│       ├── py.typed            # empty — marks package as typed
│       ├── client.py           # main client class
│       ├── models.py           # data models (dataclass/Pydantic)
│       ├── exceptions.py       # exception hierarchy
│       ├── _internal.py        # private helpers (underscore prefix)
│       └── utils.py            # shared utilities
├── tests/
│   ├── conftest.py             # shared fixtures
│   ├── test_client.py
│   └── test_models.py
└── README.md
```

### __init__.py with re-exports

```python
"""my_library — a reusable Python library."""

from importlib.metadata import version

from my_library.client import Client
from my_library.exceptions import AppError, NotFoundError, ValidationError
from my_library.models import Config, User

__version__ = version("my-library")
__all__ = [
    "Client",
    "Config",
    "User",
    "AppError",
    "NotFoundError",
    "ValidationError",
]
```

### exceptions.py

```python
class AppError(Exception):
    """Base exception for the library."""

class NotFoundError(AppError):
    def __init__(self, resource: str, id: str) -> None:
        super().__init__(f"{resource} {id!r} not found")
        self.resource = resource
        self.id = id

class ValidationError(AppError):
    def __init__(self, field: str, message: str) -> None:
        super().__init__(f"{field}: {message}")
        self.field = field
```

## conftest.py with common fixtures

```python
from collections.abc import AsyncGenerator, Generator
from pathlib import Path
from unittest.mock import AsyncMock

import pytest
import httpx
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

from my_app.db import Base


@pytest.fixture(scope="session")
def db_engine():
    """SQLite in-memory engine, shared across the test session."""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()


@pytest.fixture
def db_session(db_engine) -> Generator[Session, None, None]:
    """Per-test DB session with automatic rollback."""
    connection = db_engine.connect()
    transaction = connection.begin()
    session = sessionmaker(bind=connection)()
    yield session
    session.close()
    transaction.rollback()
    connection.close()


@pytest.fixture
def sample_user_data() -> dict[str, str]:
    return {
        "name": "Ada Lovelace",
        "email": "ada@example.com",
    }


@pytest.fixture
def config_dir(tmp_path: Path) -> Path:
    """Pre-populated config directory."""
    config = tmp_path / "config"
    config.mkdir()
    (config / "settings.toml").write_text('[server]\nhost = "localhost"\nport = 8080\n')
    return config


@pytest.fixture
def mock_http_client() -> AsyncMock:
    """Mock httpx.AsyncClient for unit tests."""
    client = AsyncMock(spec=httpx.AsyncClient)
    response = AsyncMock(spec=httpx.Response)
    response.status_code = 200
    response.json.return_value = {"status": "ok"}
    response.raise_for_status = AsyncMock()
    client.get.return_value = response
    client.post.return_value = response
    return client


@pytest.fixture
def env_vars(monkeypatch: pytest.MonkeyPatch) -> dict[str, str]:
    """Set common environment variables for testing."""
    vars = {
        "APP_ENV": "test",
        "DATABASE_URL": "sqlite:///:memory:",
        "API_KEY": "test-key-abc123",
    }
    for key, value in vars.items():
        monkeypatch.setenv(key, value)
    return vars
```

## GitHub Actions CI workflow

Save as `.github/workflows/ci.yml`.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: uv sync --frozen

      - name: Format check
        run: uv run ruff format --check src/ tests/

      - name: Lint
        run: uv run ruff check src/ tests/

      - name: Type check
        run: uv run mypy src/

  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.12", "3.13", "3.14"]
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: uv sync --frozen

      - name: Run tests
        run: uv run pytest --cov=src/ --cov-report=term-missing --cov-report=xml -v

      - name: Upload coverage
        if: matrix.python-version == '3.14'
        uses: codecov/codecov-action@v4
        with:
          file: coverage.xml
          fail_ci_if_error: false
```