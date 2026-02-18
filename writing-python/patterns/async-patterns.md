# Async Patterns

## Contents
- asyncio.run() entry point with signal handling
- asyncio.TaskGroup for structured concurrency
- aiohttp/httpx session with connection pooling
- asyncio.Queue for producer-consumer
- asyncio.Semaphore for rate limiting
- asyncio.to_thread() for sync wrappers
- Async generators

## Entry point with signal handling

```python
import asyncio
import signal
import sys
from collections.abc import Coroutine
from typing import Any

async def shutdown(sig: signal.Signals, loop: asyncio.AbstractEventLoop) -> None:
    """Cancel all running tasks on signal."""
    tasks = [t for t in asyncio.all_tasks() if t is not asyncio.current_task()]
    for task in tasks:
        task.cancel()
    await asyncio.gather(*tasks, return_exceptions=True)
    loop.stop()

async def main() -> None:
    loop = asyncio.get_running_loop()

    # Register signal handlers (Unix only)
    if sys.platform != "win32":
        for sig in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(
                sig,
                lambda s=sig: asyncio.create_task(shutdown(s, loop)),
            )

    # Application logic
    await run_server()

if __name__ == "__main__":
    asyncio.run(main())
```

For simpler scripts without custom signal handling:

```python
async def main() -> None:
    results = await fetch_all_data()
    process(results)

if __name__ == "__main__":
    asyncio.run(main())
```

## TaskGroup — structured concurrency (3.11+)

`TaskGroup` guarantees all tasks complete (or cancel) before leaving the `async with` block. Exceptions propagate as `ExceptionGroup`.

### Fan-out requests

```python
import asyncio
import httpx

async def fetch_url(client: httpx.AsyncClient, url: str) -> dict[str, object]:
    resp = await client.get(url)
    resp.raise_for_status()
    return {"url": url, "status": resp.status_code, "length": len(resp.content)}

async def fetch_all(urls: list[str]) -> list[dict[str, object]]:
    async with httpx.AsyncClient(timeout=30.0) as client:
        async with asyncio.TaskGroup() as tg:
            tasks = [tg.create_task(fetch_url(client, url)) for url in urls]
        return [t.result() for t in tasks]
```

### Handling partial failures

```python
async def fetch_with_fallback(urls: list[str]) -> list[dict[str, object]]:
    results: list[dict[str, object]] = []
    errors: list[Exception] = []

    async with httpx.AsyncClient() as client:
        try:
            async with asyncio.TaskGroup() as tg:
                tasks = [tg.create_task(fetch_url(client, url)) for url in urls]
        except* httpx.HTTPStatusError as eg:
            errors.extend(eg.exceptions)
        except* httpx.TimeoutException as eg:
            errors.extend(eg.exceptions)

    for task in tasks:
        if not task.cancelled() and task.exception() is None:
            results.append(task.result())

    if errors:
        log.warning("Failed %d/%d requests", len(errors), len(urls))

    return results
```

## HTTP client with connection pooling

### httpx (recommended — sync + async, HTTP/2)

```python
import httpx

async def create_client() -> httpx.AsyncClient:
    return httpx.AsyncClient(
        base_url="https://api.example.com",
        timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0),
        limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
        headers={"User-Agent": "my-app/1.0"},
        http2=True,
    )

async def main() -> None:
    async with await create_client() as client:
        resp = await client.get("/users")
        users = resp.json()
```

### aiohttp (alternative — websockets, streaming)

```python
import aiohttp

async def main() -> None:
    connector = aiohttp.TCPConnector(
        limit=100,                  # total connections
        limit_per_host=30,          # per-host connections
        ttl_dns_cache=300,          # DNS cache TTL in seconds
        enable_cleanup_closed=True,
    )
    timeout = aiohttp.ClientTimeout(total=30, connect=5)

    async with aiohttp.ClientSession(
        connector=connector,
        timeout=timeout,
        base_url="https://api.example.com",
    ) as session:
        async with session.get("/users") as resp:
            users = await resp.json()
```

## Queue — producer-consumer

```python
import asyncio
from dataclasses import dataclass

@dataclass
class Job:
    id: int
    url: str

POISON = None  # sentinel to signal shutdown

async def producer(queue: asyncio.Queue[Job | None], urls: list[str]) -> None:
    for i, url in enumerate(urls):
        job = Job(id=i, url=url)
        await queue.put(job)
    await queue.put(POISON)

async def worker(
    name: str,
    queue: asyncio.Queue[Job | None],
    results: list[dict[str, object]],
) -> None:
    while True:
        job = await queue.get()
        if job is POISON:
            await queue.put(POISON)  # propagate to other workers
            queue.task_done()
            break
        try:
            result = await process_job(job)
            results.append(result)
        except Exception:
            log.exception("Worker %s failed job %d", name, job.id)
        finally:
            queue.task_done()

async def run_pipeline(urls: list[str], num_workers: int = 5) -> list[dict[str, object]]:
    queue: asyncio.Queue[Job | None] = asyncio.Queue(maxsize=100)
    results: list[dict[str, object]] = []

    async with asyncio.TaskGroup() as tg:
        tg.create_task(producer(queue, urls))
        for i in range(num_workers):
            tg.create_task(worker(f"worker-{i}", queue, results))

    return results
```

## Semaphore — rate limiting

```python
import asyncio
import httpx

async def rate_limited_fetch(
    client: httpx.AsyncClient,
    url: str,
    semaphore: asyncio.Semaphore,
) -> httpx.Response:
    async with semaphore:
        resp = await client.get(url)
        resp.raise_for_status()
        return resp

async def fetch_all_rate_limited(
    urls: list[str],
    max_concurrent: int = 10,
) -> list[httpx.Response]:
    semaphore = asyncio.Semaphore(max_concurrent)
    async with httpx.AsyncClient(timeout=30.0) as client:
        async with asyncio.TaskGroup() as tg:
            tasks = [
                tg.create_task(rate_limited_fetch(client, url, semaphore))
                for url in urls
            ]
        return [t.result() for t in tasks]
```

### Token bucket for requests-per-second

```python
import asyncio
import time

class TokenBucket:
    """Rate limiter: max `rate` requests per second."""

    def __init__(self, rate: float, burst: int = 1) -> None:
        self.rate = rate
        self.burst = burst
        self._tokens = float(burst)
        self._last_refill = time.monotonic()
        self._lock = asyncio.Lock()

    async def acquire(self) -> None:
        async with self._lock:
            now = time.monotonic()
            elapsed = now - self._last_refill
            self._tokens = min(self.burst, self._tokens + elapsed * self.rate)
            self._last_refill = now

            if self._tokens < 1:
                wait = (1 - self._tokens) / self.rate
                await asyncio.sleep(wait)
                self._tokens = 0
            else:
                self._tokens -= 1

# Usage
bucket = TokenBucket(rate=10.0, burst=20)  # 10 req/s, burst of 20

async def fetch(client: httpx.AsyncClient, url: str) -> httpx.Response:
    await bucket.acquire()
    return await client.get(url)
```

## asyncio.to_thread() — wrapping sync code

Offload blocking I/O or CPU-bound work to a thread so the event loop stays responsive.

```python
import asyncio
from pathlib import Path

def read_large_file(path: Path) -> str:
    """Blocking file read — don't call directly from async code."""
    return path.read_text()

def cpu_heavy(data: bytes) -> bytes:
    """CPU-bound compression."""
    import gzip
    return gzip.compress(data)

async def process_file(path: Path) -> bytes:
    content = await asyncio.to_thread(read_large_file, path)
    compressed = await asyncio.to_thread(cpu_heavy, content.encode())
    return compressed

# Wrapping a sync library
import boto3

async def upload_to_s3(bucket: str, key: str, data: bytes) -> None:
    s3 = boto3.client("s3")
    await asyncio.to_thread(s3.put_object, Bucket=bucket, Key=key, Body=data)
```

## Async generators

### Paginated API iteration

```python
from collections.abc import AsyncIterator

async def paginate_users(
    client: httpx.AsyncClient,
    page_size: int = 100,
) -> AsyncIterator[dict[str, object]]:
    cursor: str | None = None
    while True:
        params: dict[str, str | int] = {"limit": page_size}
        if cursor:
            params["cursor"] = cursor
        resp = await client.get("/users", params=params)
        resp.raise_for_status()
        data = resp.json()

        for user in data["users"]:
            yield user

        cursor = data.get("next_cursor")
        if not cursor:
            break

# Consume with async for
async def export_all_users(client: httpx.AsyncClient) -> list[dict[str, object]]:
    users = []
    async for user in paginate_users(client):
        users.append(user)
    return users
```

### Streaming lines from a response

```python
async def stream_lines(
    client: httpx.AsyncClient,
    url: str,
) -> AsyncIterator[str]:
    async with client.stream("GET", url) as resp:
        resp.raise_for_status()
        async for line in resp.aiter_lines():
            if line.strip():
                yield line
```