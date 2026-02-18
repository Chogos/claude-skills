# Testing Patterns

## Contents
- Parametrize with ids and complex objects
- Fixture scoping
- conftest.py layering
- Mocking with monkeypatch
- pytest-httpx for HTTP mocking
- Async test patterns
- tmp_path for file-based tests
- caplog for log assertion

## @pytest.mark.parametrize

### Basic with ids

```python
import pytest

@pytest.mark.parametrize(
    ("input_val", "expected"),
    [
        ("hello", "HELLO"),
        ("world", "WORLD"),
        ("", ""),
        ("already UPPER", "ALREADY UPPER"),
    ],
    ids=["lowercase", "simple", "empty", "mixed-case"],
)
def test_to_upper(input_val: str, expected: str) -> None:
    assert input_val.upper() == expected
```

### Complex objects with ids

```python
from dataclasses import dataclass

@dataclass
class ParseCase:
    raw: str
    expected_host: str
    expected_port: int
    id: str

PARSE_CASES = [
    ParseCase("localhost:8080", "localhost", 8080, "basic"),
    ParseCase("0.0.0.0:443", "0.0.0.0", 443, "all-interfaces"),
    ParseCase("[::1]:9090", "::1", 9090, "ipv6"),
]

@pytest.mark.parametrize(
    "case",
    PARSE_CASES,
    ids=lambda c: c.id,
)
def test_parse_address(case: ParseCase) -> None:
    result = parse_address(case.raw)
    assert result.host == case.expected_host
    assert result.port == case.expected_port
```

### Parametrize with pytest.raises

```python
@pytest.mark.parametrize(
    ("input_val", "error_match"),
    [
        ("not-a-number", "invalid literal"),
        ("", "empty string"),
        ("99999999999999", "out of range"),
    ],
    ids=["nan", "empty", "overflow"],
)
def test_parse_int_errors(input_val: str, error_match: str) -> None:
    with pytest.raises(ValueError, match=error_match):
        strict_parse_int(input_val)
```

## Fixture scoping

```python
import pytest

@pytest.fixture(scope="session")
def db_engine():
    """Once per test session — expensive setup like DB engines."""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()

@pytest.fixture(scope="module")
def shared_data() -> dict[str, str]:
    """Once per test module — shared across all tests in one file."""
    return {"api_key": "test-key"}

@pytest.fixture  # scope="function" is the default
def user(db_session: Session) -> User:
    """Fresh per test — no state leakage between tests."""
    user = User(name="test", email="test@example.com")
    db_session.add(user)
    db_session.flush()
    return user
```

Scope hierarchy: `session` > `package` > `module` > `class` > `function`. A fixture can only depend on fixtures with equal or wider scope.

## conftest.py layering

```
tests/
├── conftest.py              # root — fixtures available everywhere
├── test_utils.py
├── api/
│   ├── conftest.py          # api-specific — client fixture, auth headers
│   ├── test_users.py
│   └── test_orders.py
└── db/
    ├── conftest.py          # db-specific — session fixture, seed data
    └── test_queries.py
```

### Root conftest.py

```python
# tests/conftest.py
import pytest

@pytest.fixture
def env_vars(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("APP_ENV", "test")
    monkeypatch.setenv("LOG_LEVEL", "DEBUG")
```

### Package-level conftest.py

```python
# tests/api/conftest.py
import pytest
from httpx import ASGITransport, AsyncClient
from my_app.main import app

@pytest.fixture
async def client() -> AsyncGenerator[AsyncClient, None]:
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
```

Package-level fixtures override root fixtures with the same name.

## Mocking with monkeypatch

### Environment variables

```python
def test_config_from_env(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("DATABASE_URL", "sqlite:///test.db")
    monkeypatch.setenv("SECRET_KEY", "hunter2")
    monkeypatch.delenv("CACHE_URL", raising=False)

    config = load_config()
    assert config.database_url == "sqlite:///test.db"
    assert config.cache_url is None
```

### Attributes and methods

```python
def test_retry_on_failure(monkeypatch: pytest.MonkeyPatch) -> None:
    call_count = 0

    def mock_request(url: str) -> dict[str, str]:
        nonlocal call_count
        call_count += 1
        if call_count < 3:
            raise ConnectionError("refused")
        return {"status": "ok"}

    monkeypatch.setattr("my_app.client.http_get", mock_request)
    result = fetch_with_retry("/health")
    assert result == {"status": "ok"}
    assert call_count == 3
```

### datetime.now

```python
from datetime import datetime, timezone

def test_expiry_check(monkeypatch: pytest.MonkeyPatch) -> None:
    fixed = datetime(2025, 6, 15, 12, 0, 0, tzinfo=timezone.utc)
    monkeypatch.setattr("my_app.auth.datetime", type("dt", (), {
        "now": staticmethod(lambda tz=None: fixed),
    }))
    assert is_token_expired(expires_at=datetime(2025, 6, 15, 11, 0, 0, tzinfo=timezone.utc))
```

## pytest-httpx for HTTP mocking

Intercepts `httpx` requests — both sync and async.

```python
import httpx
import pytest
from pytest_httpx import HTTPXMock

async def test_fetch_user(httpx_mock: HTTPXMock) -> None:
    httpx_mock.add_response(
        url="https://api.example.com/users/42",
        json={"id": 42, "name": "Ada"},
        status_code=200,
    )

    async with httpx.AsyncClient() as client:
        user = await fetch_user(client, user_id=42)

    assert user.name == "Ada"

async def test_api_timeout(httpx_mock: HTTPXMock) -> None:
    httpx_mock.add_exception(httpx.ReadTimeout("timed out"))

    with pytest.raises(ServiceUnavailableError):
        async with httpx.AsyncClient() as client:
            await fetch_user(client, user_id=1)

def test_multiple_requests(httpx_mock: HTTPXMock) -> None:
    httpx_mock.add_response(url="https://api.example.com/health", json={"ok": True})
    httpx_mock.add_response(url="https://api.example.com/version", json={"v": "1.2"})

    results = check_all_endpoints()
    assert results["health"]["ok"] is True
    assert results["version"]["v"] == "1.2"

    # Verify request count
    assert len(httpx_mock.get_requests()) == 2
```

## Async test patterns

Use `pytest-asyncio`. Set `asyncio_mode = "auto"` in `pyproject.toml` to avoid `@pytest.mark.asyncio` on every test.

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

### Async fixtures

```python
import pytest
from collections.abc import AsyncGenerator

@pytest.fixture
async def db_pool() -> AsyncGenerator[Pool, None]:
    pool = await create_pool(dsn="postgresql://localhost/test")
    yield pool
    await pool.close()

@pytest.fixture
async def db_conn(db_pool: Pool) -> AsyncGenerator[Connection, None]:
    async with db_pool.acquire() as conn:
        tx = conn.transaction()
        await tx.start()
        yield conn
        await tx.rollback()
```

### Async test functions

```python
async def test_create_and_fetch_user(db_conn: Connection) -> None:
    await create_user(db_conn, name="Ada", email="ada@example.com")
    user = await get_user_by_email(db_conn, "ada@example.com")
    assert user.name == "Ada"

async def test_concurrent_writes(db_conn: Connection) -> None:
    async with asyncio.TaskGroup() as tg:
        for i in range(10):
            tg.create_task(create_user(db_conn, name=f"user-{i}", email=f"u{i}@test.com"))

    users = await list_users(db_conn)
    assert len(users) == 10
```

## tmp_path for file-based tests

`tmp_path` is a built-in `pathlib.Path` fixture — unique per test, auto-cleaned.

```python
from pathlib import Path

def test_write_and_read_json(tmp_path: Path) -> None:
    data = {"users": [{"name": "Ada"}]}
    out = tmp_path / "data.json"

    write_json(out, data)
    result = read_json(out)

    assert result == data

def test_process_directory(tmp_path: Path) -> None:
    src = tmp_path / "input"
    src.mkdir()
    (src / "a.txt").write_text("hello")
    (src / "b.txt").write_text("world")
    (src / "skip.log").write_text("ignore me")

    dest = tmp_path / "output"
    process_text_files(src, dest)

    assert (dest / "a.txt").exists()
    assert (dest / "b.txt").exists()
    assert not (dest / "skip.log").exists()

def test_config_file_missing(tmp_path: Path) -> None:
    missing = tmp_path / "nonexistent.toml"
    with pytest.raises(FileNotFoundError):
        load_config(missing)
```

Use `tmp_path_factory` (session-scoped) when multiple tests share the same expensive file setup:

```python
@pytest.fixture(scope="session")
def large_dataset(tmp_path_factory: pytest.TempPathFactory) -> Path:
    path = tmp_path_factory.mktemp("data") / "big.csv"
    path.write_text("\n".join(f"row,{i}" for i in range(10_000)))
    return path
```

## caplog for log assertion

```python
import logging

def test_warns_on_deprecation(caplog: pytest.LogCaptureFixture) -> None:
    with caplog.at_level(logging.WARNING):
        call_deprecated_function()

    assert len(caplog.records) == 1
    assert "deprecated" in caplog.records[0].message
    assert caplog.records[0].levelname == "WARNING"

def test_logs_request_info(caplog: pytest.LogCaptureFixture) -> None:
    with caplog.at_level(logging.INFO, logger="my_app.client"):
        make_request("/api/health")

    messages = [r.message for r in caplog.records]
    assert any("GET /api/health" in m for m in messages)
    assert any("200" in m for m in messages)

def test_no_errors_logged(caplog: pytest.LogCaptureFixture) -> None:
    with caplog.at_level(logging.ERROR):
        process_valid_data({"key": "value"})

    assert len(caplog.records) == 0
```