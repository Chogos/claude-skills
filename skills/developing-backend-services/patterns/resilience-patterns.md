# Resilience Patterns

Protect against cascading failures when calling external services. These patterns compose — the typical stack is: timeout → retry → circuit breaker → bulkhead.

## Retry with Backoff and Jitter

Retry transient failures (network timeouts, 503s, connection resets). Never retry client errors (4xx) — they won't succeed on retry.

```
function retry(operation, config):
  last_error = null
  for attempt in 0..config.max_retries:
    try:
      return operation()
    catch error:
      if not is_retryable(error):
        raise error
      last_error = error
      if attempt < config.max_retries:
        delay = min(config.base_delay * 2^attempt, config.max_delay)
        jitter = random(0, delay * 0.5)
        sleep(delay + jitter)
  raise last_error

function is_retryable(error):
  if error is timeout:       return true
  if error is connection_reset: return true
  if error.status_code >= 500: return true
  if error.status_code == 429:  return true   # rate limited — respect Retry-After
  return false
```

### Configuration

| Parameter | Typical value | Notes |
|-----------|--------------|-------|
| `max_retries` | 3 | Total attempts = max_retries + 1 |
| `base_delay` | 500ms | First retry delay before jitter |
| `max_delay` | 30s | Cap exponential growth |
| `jitter` | 0–50% of delay | Prevents thundering herd |

### Retry Budget

Limit retries across the system, not just per request. If 20% of requests are already retries, stop retrying — the downstream is overloaded and retries make it worse.

```
function should_retry(metrics):
  recent_requests = metrics.total_requests(window: 1_minute)
  recent_retries = metrics.retry_requests(window: 1_minute)
  if recent_requests == 0:
    return true
  return (recent_retries / recent_requests) < 0.2   # max 20% retry ratio
```

## Circuit Breaker

Prevent repeated calls to a failing service. Three states:

```
CLOSED  ──(failure threshold reached)──▸  OPEN
  ▴                                        │
  │                                   (recovery timeout)
  │                                        ▾
  └──(probe succeeds)──  HALF-OPEN  ──(probe fails)──▸  OPEN
```

### Implementation

```
class CircuitBreaker:
  state = CLOSED
  failure_count = 0
  last_failure_time = null

  function call(operation):
    if state == OPEN:
      if now() - last_failure_time > config.recovery_timeout:
        state = HALF_OPEN
      else:
        raise CircuitOpenError()

    try:
      result = operation()
      on_success()
      return result
    catch error:
      on_failure()
      raise error

  function on_success():
    failure_count = 0
    if state == HALF_OPEN:
      state = CLOSED

  function on_failure():
    failure_count += 1
    last_failure_time = now()
    if failure_count >= config.failure_threshold:
      state = OPEN
```

### Configuration

| Parameter | Typical value | Notes |
|-----------|--------------|-------|
| `failure_threshold` | 5 | Consecutive failures to trip open |
| `recovery_timeout` | 30s | How long to stay open before probing |
| `half_open_max` | 3 | Max probe requests in half-open state |

### Per-Dependency Breakers

Create one circuit breaker per downstream dependency. A failing payment service should not open the circuit for the user service.

```
breakers = {
  "payment-service": CircuitBreaker(failure_threshold: 5, recovery_timeout: 30s),
  "email-service":   CircuitBreaker(failure_threshold: 3, recovery_timeout: 60s),
}

function call_payment(request):
  return breakers["payment-service"].call(
    () => http_client.post("https://payment-svc/charge", request)
  )
```

## Timeout Cascade

Set timeouts per dependency. Each layer must have a shorter timeout than the layer above it — otherwise the caller times out before the callee, wasting resources.

```
Gateway timeout:  30s
  └─ Service A timeout:  10s
       ├─ Database timeout:  3s
       └─ Service B timeout:  5s
            └─ Cache timeout:  1s
```

### Implementation

Propagate deadlines through context. The inner timeout is always shorter than the outer:

```
function handle_request(ctx):    # ctx has 10s deadline from gateway
  # Database call: 3s max, but respect parent deadline
  db_ctx = with_timeout(ctx, 3s)
  user = db.query(db_ctx, "SELECT ...")

  # External call: 5s max, but respect parent deadline
  api_ctx = with_timeout(ctx, 5s)
  result = http_client.get(api_ctx, "https://service-b/data")

  return combine(user, result)
```

Leave margin — if the gateway gives you 10s, don't set your DB timeout to 10s. Reserve time for processing between calls.

## Bulkhead

Isolate failures by limiting concurrent calls per dependency. Prevents one slow service from consuming all threads/connections.

```
class Bulkhead:
  semaphore = Semaphore(config.max_concurrent)

  function call(operation):
    acquired = semaphore.try_acquire(timeout: config.wait_timeout)
    if not acquired:
      raise BulkheadFullError("dependency at capacity")
    try:
      return operation()
    finally:
      semaphore.release()
```

### Configuration

| Parameter | Typical value | Notes |
|-----------|--------------|-------|
| `max_concurrent` | 10–25 | Per dependency, based on its capacity |
| `wait_timeout` | 500ms | How long to wait for a slot before rejecting |

Size the bulkhead based on the dependency's capacity. If the payment service handles 100 req/s and you're one of four consumers, set `max_concurrent` to ~25.

## Fallback Strategies

When a dependency fails, degrade gracefully instead of propagating the error:

```
function get_product_recommendations(user_id):
  try:
    return recommendation_service.get(user_id)       # primary
  catch error:
    try:
      return cache.get("recommendations:" + user_id)  # cached fallback
    catch:
      return default_recommendations()                 # static fallback
```

| Strategy | When to use |
|----------|-------------|
| **Cached response** | Return stale data from cache when the source is down |
| **Degraded response** | Return partial data (e.g., product without reviews) |
| **Default value** | Return a safe static default (e.g., empty recommendations) |
| **Fail fast** | Return an error immediately — use for critical paths that can't degrade |

Non-critical features (recommendations, analytics, chat widgets) should always have a fallback. Critical features (payment, authentication) should fail fast with a clear error.

## Combining Patterns

Apply patterns in this order — outermost wraps innermost:

```
function call_payment_service(request):
  return bulkhead["payment"].call(         # 4. limit concurrency
    () => circuit_breaker["payment"].call(  # 3. stop calling if broken
      () => retry(                          # 2. retry transient failures
        () => http_client.post(             # 1. actual call with timeout
          "https://payment-svc/charge",
          request,
          timeout: 5s
        ),
        max_retries: 2,
        base_delay: 500ms
      )
    )
  )
```

Order matters:
- **Timeout** (innermost) — bounds individual call duration
- **Retry** — retries timed-out or failed calls
- **Circuit breaker** — stops retrying if the service is down
- **Bulkhead** (outermost) — limits how many requests even attempt the call

### Observable Resilience

Emit metrics for every resilience event:

```
retry_attempts_total{service, dependency, attempt}    # counter
circuit_breaker_state{service, dependency}             # gauge: 0=closed, 1=open, 2=half-open
circuit_breaker_trips_total{service, dependency}       # counter
bulkhead_rejected_total{service, dependency}           # counter
bulkhead_active_calls{service, dependency}             # gauge
fallback_used_total{service, dependency, strategy}     # counter
```

Alert when circuit breakers trip or bulkheads reject — these are symptoms of downstream problems.
