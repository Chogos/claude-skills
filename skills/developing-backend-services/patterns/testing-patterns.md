# Testing Patterns

Unit tests, integration tests, contract tests, database tests, and load tests for backend services.

## Unit Testing Service Handlers

Test business logic in isolation. Inject mock dependencies via constructor.

```
# Handler receives dependencies as interfaces — easy to mock
handler = OrderHandler(
  order_repo:  MockOrderRepo(),
  payment_svc: MockPaymentService(),
  event_bus:   MockEventBus(),
)

# Test the happy path
test "create order returns 201 with valid input":
  mock_payment_svc.charge() -> returns Success(tx_id="tx_123")
  mock_order_repo.save()    -> returns Order(id="ord_456")

  response = handler.create_order({
    user_id: "usr_1",
    items: [{ product_id: "prod_1", quantity: 2 }],
    payment_method: "pm_abc",
  })

  assert response.status == 201
  assert response.body.id == "ord_456"
  assert mock_event_bus.published == [OrderCreatedEvent(order_id="ord_456")]

# Test error paths — these catch more bugs than happy paths
test "create order returns 402 when payment fails":
  mock_payment_svc.charge() -> returns Failure("card_declined")

  response = handler.create_order({ ... })

  assert response.status == 402
  assert response.body.error.code == "PAYMENT_FAILED"
  assert mock_order_repo.save.not_called()   # no order created on payment failure

test "create order returns 422 on empty items":
  response = handler.create_order({ user_id: "usr_1", items: [] })

  assert response.status == 422
  assert response.body.error.details[0].field == "items"
```

Rules:
- One assertion per behavior. Test what happened, not how it happened.
- Mock interfaces, not concrete implementations.
- Test error paths more thoroughly than happy paths — that's where bugs hide.
- Don't test framework internals (routing, middleware). Test your handler logic.

## Integration Testing with Containers

Use testcontainers (or `docker compose`) to run real dependencies in CI.

```
# Spin up a real Postgres instance per test suite
test_db = start_container("postgres:17-alpine", {
  env: { POSTGRES_DB: "test", POSTGRES_USER: "test", POSTGRES_PASSWORD: "test" },
  port: 5432,
  wait_for: health_check("pg_isready"),
})

# Run migrations against the real database
migrate(test_db.connection_url())

# Each test gets a clean state
setup_each_test:
  begin_transaction()

teardown_each_test:
  rollback_transaction()   # fast cleanup, no leftover data

# Test actual SQL queries — mocks can't catch query bugs
test "find_orders_by_user returns orders sorted by created_at desc":
  insert_order(user_id="usr_1", created_at="2024-01-15", total=59.99)
  insert_order(user_id="usr_1", created_at="2024-01-16", total=124.50)
  insert_order(user_id="usr_2", created_at="2024-01-15", total=30.00)

  orders = order_repo.find_by_user("usr_1")

  assert len(orders) == 2
  assert orders[0].total == 124.50   # most recent first
  assert orders[1].total == 59.99
```

Container setup:
- **Postgres/MySQL**: testcontainers with health check. Run full migration chain.
- **Redis**: testcontainers. Flush between tests or use key prefixes.
- **Message queues**: testcontainers with RabbitMQ, LocalStack for SQS. Verify message consumption end-to-end.

Cleanup strategies:
- **Transaction rollback** (preferred): wrap each test in a transaction, rollback after. Fast, no leftover data.
- **Truncate tables**: `TRUNCATE ... CASCADE` between tests. Slower but works when tests need committed data.
- **Isolated databases**: create a fresh database per test suite. Slowest, use only for migration testing.

## Contract Testing

Validate that API responses match the OpenAPI spec.

```
# Automated fuzz testing against the spec
# schemathesis generates requests from the OpenAPI spec and validates responses
schemathesis run http://localhost:8080/openapi.json \
  --checks all \
  --hypothesis-max-examples 100

# Custom contract assertion in integration tests
test "GET /users/{id} response matches OpenAPI schema":
  response = http_get("/users/usr_1")

  validate_against_schema(
    schema: load_openapi_spec("openapi.yaml"),
    path: "/users/{id}",
    method: "GET",
    status: 200,
    body: response.body,
  )

# Breaking change detection in CI
# Compare current spec against the last released version
openapi-diff old-spec.yaml new-spec.yaml --fail-on-incompatible
```

What to validate:
- Response body matches the declared schema (field names, types, required fields).
- Error responses use the standard error envelope.
- Pagination responses include `next_cursor` and `has_more`.
- Status codes match the spec for each path + method combination.

## Database Test Patterns

### Test Fixtures with Factories

```
# Factory functions create valid entities with sensible defaults
function create_user(overrides = {}):
  defaults = {
    name: "Test User",
    email: f"user-{uuid()}@test.com",   # unique per call
    status: "active",
    created_at: now(),
  }
  return insert_into("users", { ...defaults, ...overrides })

# Tests override only the fields they care about
test "deactivate_user sets status to inactive":
  user = create_user(status="active")
  deactivate_user(user.id)
  assert db.get_user(user.id).status == "inactive"

test "list_active_users excludes inactive":
  active   = create_user(status="active")
  inactive = create_user(status="inactive")
  result   = list_active_users()
  assert active.id in [u.id for u in result]
  assert inactive.id not in [u.id for u in result]
```

### Migration Testing

```
# Verify the full migration chain works from scratch
test "migrations apply cleanly to empty database":
  db = create_empty_database()
  migrate_to_latest(db)
  assert db.table_exists("users")
  assert db.table_exists("orders")
  assert db.index_exists("idx_orders_user_id")

# Verify migrations are reversible (if you support rollback)
test "migrations rollback cleanly":
  migrate_to_latest(db)
  rollback_to(db, version=3)
  migrate_to_latest(db)   # re-apply should succeed
```

## Load Testing with k6

### Basic Load Test

```javascript
// k6 script: load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },    // ramp up to 50 users
    { duration: '3m', target: 50 },    // sustain
    { duration: '1m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],   // 99th percentile under 500ms
    http_req_failed: ['rate<0.01'],     // error rate under 1%
  },
};

export default function () {
  const res = http.get('https://staging.example.com/api/v1/orders');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

Run: `k6 run load-test.js`

### Test Tiers

| Tier | Users | Duration | When to run |
|------|-------|----------|-------------|
| Smoke | 1 | 30s | Every PR — verify the endpoint works |
| Load | 50-100 | 5min | Release branch — verify performance |
| Stress | 2x expected | 10min | Pre-launch — find breaking point |
| Soak | Expected load | 1h+ | Monthly — find memory leaks, connection exhaustion |

### CI Integration

```yaml
# Run smoke test on every PR, load test on release branches
- name: Smoke test
  run: k6 run --vus 1 --duration 30s load-test.js

- name: Load test (release only)
  if: startsWith(github.ref, 'refs/heads/release/')
  run: k6 run load-test.js
```

Store k6 results in a time-series database (InfluxDB, Prometheus) for trend analysis. Alert on regressions.
