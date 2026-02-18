# Observability Patterns

Structured logging, request correlation, RED metrics, distributed tracing, and alerting rules.

## Structured Logging

Emit logs as JSON. Every log line must be machine-parseable.

### Standard Fields

```json
{
  "timestamp": "2024-03-15T10:30:00.123Z",
  "level": "error",
  "message": "failed to process payment",
  "service": "payment-service",
  "version": "1.4.2",
  "environment": "production",
  "request_id": "req_abc123",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "user_id": "usr_789",
  "error": {
    "type": "PaymentGatewayTimeout",
    "message": "upstream timeout after 5000ms",
    "stack": "..."
  },
  "duration_ms": 5023
}
```

### Required Fields Per Level

| Level | When to use |
|-------|-------------|
| `error` | Operation failed, requires attention. Include error type, message, stack |
| `warn` | Degraded behavior (retry succeeded, fallback used, approaching limit) |
| `info` | Significant business events (order created, payment processed, user registered) |
| `debug` | Diagnostic detail for development. Disabled in production by default |

### Rules

- Timestamp in ISO 8601 / RFC 3339 with milliseconds, always UTC.
- Never log: passwords, tokens, API keys, credit card numbers, SSNs, full request bodies containing PII. Mask or omit.
- Log at request boundaries: one log on entry (info), one on completion (info with duration), one on error (error).
- Include `request_id` and `trace_id` on every log line — makes correlation trivial.
- Use structured fields, not string interpolation: `{ "user_id": 123 }` not `"Processing user 123"`.

### Setup Pseudocode

```
logger = new StructuredLogger(
  output: stdout,           # never log to files — let the platform collect stdout
  format: "json",
  default_fields: {
    service: config.service_name,
    version: config.version,
    environment: config.environment,
  }
)

# Per-request logger with correlation IDs
function request_logger(request):
  return logger.with({
    request_id: request.header("X-Request-ID"),
    trace_id: request.trace_context.trace_id,
    span_id: request.trace_context.span_id,
  })
```

## Request Correlation

Generate or propagate a unique request ID through the entire call chain.

### Middleware

```
function correlation_middleware(request, next):
  request_id = request.header("X-Request-ID")
  if request_id is null:
    request_id = generate_uuid_v7()  # v7 = time-sortable

  # Attach to request context for downstream use
  request.context.set("request_id", request_id)

  response = next(request)

  # Echo back in response for client correlation
  response.header("X-Request-ID", request_id)
  return response
```

### Propagation

When calling downstream services, forward both:
- `X-Request-ID` — application-level correlation
- `traceparent` — W3C distributed trace header

Every log line, metric label, and trace span includes the request ID. A single search by request ID returns the full request lifecycle across all services.

## RED Metrics Instrumentation

RED = Rate, Errors, Duration. Instrument every service endpoint.

### Counter: Request Rate and Errors

```
# Total requests — rate() gives requests/second
http_requests_total{service, method, path, status_code}

# Instrument in middleware:
function metrics_middleware(request, next):
  response = next(request)
  counter_increment("http_requests_total", {
    service: config.service_name,
    method: request.method,
    path: normalize_path(request.path),  # /users/123 → /users/{id}
    status_code: response.status,
  })
  return response
```

Error rate = `rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m])`

### Histogram: Request Duration

```
# Latency distribution
http_request_duration_seconds{service, method, path}

# Instrument in middleware:
function metrics_middleware(request, next):
  start = clock_now()
  response = next(request)
  duration = clock_now() - start

  histogram_observe("http_request_duration_seconds", duration, {
    service: config.service_name,
    method: request.method,
    path: normalize_path(request.path),
  })
  return response
```

Histogram buckets: `[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]` seconds.

### Path Normalization

Replace dynamic path segments with placeholders to avoid cardinality explosion:
```
/users/123          → /users/{id}
/orders/456/items/7 → /orders/{id}/items/{id}
```

Never use raw paths as metric labels — unbounded cardinality kills your metrics backend.

### Additional Metrics

```
# Database connection pool
db_pool_active_connections{service, db_name}        # gauge
db_pool_idle_connections{service, db_name}           # gauge
db_query_duration_seconds{service, db_name, query}   # histogram

# External service calls
external_request_duration_seconds{service, target, method, status}  # histogram

# Queue depth
queue_messages_pending{service, queue_name}          # gauge
queue_messages_processed_total{service, queue_name}  # counter
```

## Distributed Tracing

### W3C Trace Context Headers

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^-^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^-^^^^^^^^^^^^^^^^-^^
             version    trace_id (32 hex)        parent_id (16 hex)  flags

tracestate: vendor1=value1,vendor2=value2
```

Propagate both headers on every outgoing HTTP call. Libraries (OpenTelemetry SDK) handle this automatically — configure the propagator once.

### Span Naming

```
# HTTP server spans: "METHOD /path-template"
"GET /users/{id}"
"POST /orders"

# HTTP client spans: "METHOD host"
"GET payment-service"
"POST email-service"

# Database spans: "db.operation db.name.table"
"SELECT users"
"INSERT orders"

# Queue spans: "queue.operation queue.name"
"publish order-events"
"process order-events"
```

### Span Attributes

| Attribute | Example | When |
|-----------|---------|------|
| `http.method` | `GET` | HTTP spans |
| `http.url` | `https://api.example.com/users` | HTTP client spans |
| `http.route` | `/users/{id}` | HTTP server spans |
| `http.status_code` | `200` | HTTP spans |
| `db.system` | `postgresql` | DB spans |
| `db.statement` | `SELECT * FROM users WHERE id = $1` | DB spans (sanitize params) |
| `db.operation` | `SELECT` | DB spans |
| `messaging.system` | `rabbitmq` | Queue spans |
| `messaging.destination` | `order-events` | Queue spans |
| `error` | `true` | Any failed span |
| `error.message` | `connection refused` | Any failed span |

### Creating Spans

```
function get_user(user_id):
  span = tracer.start_span("SELECT users", kind: CLIENT, attributes: {
    "db.system": "postgresql",
    "db.operation": "SELECT",
    "db.statement": "SELECT * FROM users WHERE id = $1",
  })
  try:
    result = db.query("SELECT * FROM users WHERE id = $1", [user_id])
    return result
  catch error:
    span.set_attribute("error", true)
    span.set_attribute("error.message", error.message)
    span.record_exception(error)
    raise
  finally:
    span.end()
```

Create a span for every external call: HTTP requests, database queries, cache lookups, queue publishes, file I/O. Internal function calls generally don't need spans unless they represent significant processing.

### Sampling Strategies

At high volume, tracing every request is expensive. Use sampling to balance observability with cost.

**Head-based sampling** — decide at trace start whether to sample:

- Simplest approach. Set a fixed rate (e.g., 10% of requests).
- Downside: may miss rare errors if they fall outside the sample.

**Tail-based sampling** — decide after the trace completes:

- Sample 100% of errors and high-latency requests, plus a percentage of successes.
- Requires a trace collector (OpenTelemetry Collector with `tail_sampling` processor).
- More expensive to run, but catches all interesting traces.

**Rules of thumb**:

- Always sample 100% of errors (`status_code >= 500`).
- Always sample 100% of traces above latency threshold (e.g., p99).
- Sample a fixed percentage (1-10%) of successful requests for baseline visibility.
- Low-traffic services: sample 100%. High-traffic (>10K req/s): tail-based or 1-5% head-based.

## Alerting Rules

Alert on symptoms (user impact), not causes (CPU high). SLO-based alerting reduces noise.

### SLO-Based Alerts (Burn Rate)

Define SLOs, then alert when the error budget burns too fast.

```
# SLO: 99.9% of requests succeed (0.1% error budget over 30 days)
# Burn rate 1x = consume entire budget in exactly 30 days
# Burn rate 14.4x = consume entire budget in 2 days → page immediately

# Prometheus: fast burn (page)
- alert: HighErrorBurnRate
  expr: |
    (
      sum(rate(http_requests_total{status_code=~"5.."}[1h]))
      / sum(rate(http_requests_total[1h]))
    ) > (14.4 * 0.001)
    and
    (
      sum(rate(http_requests_total{status_code=~"5.."}[5m]))
      / sum(rate(http_requests_total[5m]))
    ) > (14.4 * 0.001)
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning 14.4x faster than normal"

# Prometheus: slow burn (ticket)
- alert: SlowErrorBurnRate
  expr: |
    (
      sum(rate(http_requests_total{status_code=~"5.."}[6h]))
      / sum(rate(http_requests_total[6h]))
    ) > (1 * 0.001)
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Error budget burning faster than sustainable"
```

### Latency SLO

```
# SLO: 99% of requests complete within 500ms
- alert: HighLatencyBurnRate
  expr: |
    (
      1 - (
        sum(rate(http_request_duration_seconds_bucket{le="0.5"}[1h]))
        / sum(rate(http_request_duration_seconds_count[1h]))
      )
    ) > (14.4 * 0.01)
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Latency SLO burn rate critical"
```

### Symptom-Based Alerts

```
# Service completely down
- alert: ServiceDown
  expr: up{job="my-service"} == 0
  for: 1m
  labels:
    severity: critical

# Queue backing up
- alert: QueueBacklog
  expr: queue_messages_pending{queue_name="orders"} > 10000
  for: 5m
  labels:
    severity: warning

# Database connection pool saturated
- alert: DBPoolExhausted
  expr: db_pool_active_connections / db_pool_max_connections > 0.9
  for: 5m
  labels:
    severity: warning
```

### Alert Hygiene

- Every alert must have a runbook link in annotations.
- Critical = page someone. Warning = create ticket. Info = dashboard only.
- If an alert fires and no one acts on it, delete it or lower severity.
- Test alerts with synthetic failures before relying on them.