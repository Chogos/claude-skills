# API Design Patterns

Resource naming, HTTP semantics, pagination, error formats, versioning, and idempotency.

## REST Resource Naming

Use plural nouns. Nest for relationships. No verbs in URLs.

```
# Good
GET    /users
GET    /users/{user_id}
GET    /users/{user_id}/orders
POST   /users/{user_id}/orders
GET    /users/{user_id}/orders/{order_id}

# Bad
GET    /getUser/{id}
POST   /createOrder
GET    /user/{id}/getOrders
POST   /users/{id}/activateAccount    # use PATCH with state field instead
```

Resource naming rules:
- Plural nouns: `/users`, `/orders`, `/products`
- Lowercase, hyphen-separated: `/order-items`, not `/orderItems` or `/order_items`
- No trailing slashes: `/users`, not `/users/`
- No file extensions: `/users/123`, not `/users/123.json`
- Query params for filtering: `/orders?status=pending&created_after=2024-01-01`
- Actions as sub-resources when unavoidable: `POST /orders/{id}/cancel` (only when no state field maps cleanly)

## HTTP Method Semantics

| Method | Purpose | Safe | Idempotent | Request Body | Success Code |
|--------|---------|------|------------|--------------|-------------|
| GET | Read resource(s) | Yes | Yes | No | 200 |
| POST | Create resource | No | No | Yes | 201 |
| PUT | Full replace | No | Yes | Yes | 200 |
| PATCH | Partial update | No | No* | Yes | 200 |
| DELETE | Remove resource | No | Yes | No | 204 |

*PATCH can be made idempotent with `If-Match` / version checks.

**Safe** = no side effects. **Idempotent** = same request repeated produces the same result.

## Status Code Reference

| Code | Name | When to use |
|------|------|-------------|
| 200 | OK | Successful GET, PUT, PATCH, or DELETE that returns a body |
| 201 | Created | Successful POST that creates a resource. Include `Location` header |
| 204 | No Content | Successful DELETE or PUT/PATCH with no response body |
| 400 | Bad Request | Malformed JSON, missing required fields, type mismatches |
| 401 | Unauthorized | Missing or invalid authentication credentials |
| 403 | Forbidden | Valid credentials but insufficient permissions |
| 404 | Not Found | Resource does not exist (also use for unauthorized when hiding existence) |
| 409 | Conflict | Resource state conflict (duplicate email, version mismatch) |
| 422 | Unprocessable Entity | Well-formed request but semantic validation failed (business rules) |
| 429 | Too Many Requests | Rate limit exceeded. Include `Retry-After` header |
| 500 | Internal Server Error | Unhandled server error. Log details, return generic message |
| 502 | Bad Gateway | Upstream service returned invalid response |
| 503 | Service Unavailable | Service temporarily down (maintenance, overload). Include `Retry-After` |

Choose 422 over 400 when the syntax is valid but business rules reject it (e.g., "quantity must be positive").

## Cursor-Based Pagination

Offset-based pagination breaks under concurrent writes — rows shift, causing duplicates or skips. Cursor-based pagination uses a stable pointer.

### Request

```
GET /orders?cursor=eyJpZCI6MTAwLCJjcmVhdGVkIjoiMjAyNC0wMS0xNSJ9&limit=25
```

The cursor is an opaque, base64-encoded token containing sort keys. Clients must not parse or construct cursors.

### Response

```json
{
  "data": [
    { "id": 101, "total": 59.99, "created_at": "2024-01-15T10:30:00Z" },
    { "id": 102, "total": 124.50, "created_at": "2024-01-15T11:00:00Z" }
  ],
  "pagination": {
    "next_cursor": "eyJpZCI6MTI1LCJjcmVhdGVkIjoiMjAyNC0wMS0xNlQwOTowMDowMFoifQ",
    "has_more": true,
    "limit": 25
  }
}
```

### Cursor Encoding

```
# Encode: serialize sort fields to JSON, base64-encode
cursor_data = { "id": last_item.id, "created_at": last_item.created_at }
cursor = base64_encode(json_encode(cursor_data))

# Decode: base64-decode, parse JSON, use in WHERE clause
WHERE (created_at, id) > (cursor.created_at, cursor.id)
ORDER BY created_at ASC, id ASC
LIMIT 26  -- fetch limit+1 to determine has_more
```

Fetch `limit + 1` rows. If you get `limit + 1` results, `has_more = true` and drop the extra row.

## Error Response Format

### Standard Error Envelope

Every error response uses the same structure:

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Order 456 not found",
    "request_id": "req_abc123"
  }
}
```

Fields:
- `code`: machine-readable error code (UPPER_SNAKE_CASE). Clients switch on this.
- `message`: human-readable description. May change without notice — clients must not parse this.
- `request_id`: correlation ID for support and debugging.

### Field-Level Validation Errors

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request validation failed",
    "request_id": "req_abc123",
    "details": [
      { "field": "email", "code": "INVALID_FORMAT", "message": "must be a valid email address" },
      { "field": "quantity", "code": "OUT_OF_RANGE", "message": "must be between 1 and 1000" },
      { "field": "shipping.zip", "code": "REQUIRED", "message": "is required" }
    ]
  }
}
```

Use dot notation for nested fields (`shipping.zip`). Return all validation errors at once — don't make clients fix one field at a time.

## API Versioning Strategies

| Strategy | Format | Pros | Cons |
|----------|--------|------|------|
| URL path | `/v1/users` | Simple routing, easy to test | URL changes on version bump |
| Header | `Accept: application/vnd.api+json;version=2` | Clean URLs | Harder to test in browser |
| Query param | `/users?version=2` | Easy to test | Clutters query params |

**Recommendation**: URL path for public APIs (simplest for consumers), header-based for internal APIs.

Version bumping rules:
- **Breaking change** (new major version): removing a field, renaming a field, changing a field type, removing an endpoint
- **Non-breaking change** (no version bump): adding an optional field, adding a new endpoint, adding an enum value

Support at most 2 major versions concurrently. Deprecate with `Sunset` and `Deprecation` headers.

## Idempotency Key Implementation

Prevent duplicate side effects when clients retry failed requests (network timeout, ambiguous response).

### Flow

```
Client sends: POST /payments
  Headers: Idempotency-Key: pay_abc123

Server:
  1. Check idempotency store for key "pay_abc123"
  2. If found → return stored response (same status code, same body)
  3. If not found → process request, store key + response, return response
```

### Middleware Pseudocode

```
function idempotency_middleware(request, next):
  if request.method not in [POST, PATCH]:
    return next(request)

  key = request.header("Idempotency-Key")
  if key is null:
    return next(request)  # optional: require key on POST

  # Check for existing result
  cached = idempotency_store.get(key)
  if cached is not null:
    if cached.status == "processing":
      return 409 Conflict { "error": "Request is already being processed" }
    return cached.response

  # Lock the key
  acquired = idempotency_store.set(key, { status: "processing" }, ttl: 24h)
  if not acquired:
    return 409 Conflict { "error": "Duplicate request" }

  # Process the request
  response = next(request)

  # Store the result
  idempotency_store.update(key, {
    status: "complete",
    response: { status_code: response.status, headers: response.headers, body: response.body }
  }, ttl: 24h)

  return response
```

### Storage

Use Redis with TTL (24h is typical). Key format: `idempotency:{client_id}:{key}`.

Rules:
- Keys are scoped per client/API key — different clients can use the same key value
- TTL must be long enough for clients to retry (24h is standard)
- Return the same status code, not just the same body
- Only cache successful responses (2xx) and client errors (4xx). Retry server errors (5xx)

## Webhook Delivery

### Outbound Webhook Flow

On event trigger, deliver to subscriber's registered URL:

```
POST {subscriber_url}
Content-Type: application/json
X-Webhook-ID: evt_abc123
X-Webhook-Timestamp: 1705312200
X-Webhook-Signature: sha256=<hmac_of_timestamp.body>

{ "event": "order.completed", "data": { "order_id": "ord_456", "total": 59.99 } }
```

### Signature Generation (sender)

```
timestamp = unix_timestamp_seconds()
payload   = f"{timestamp}.{json_body}"
signature = hmac_sha256(webhook_secret, payload)
header    = f"sha256={signature}"
```

Include the timestamp in the signed payload to prevent replay attacks. Receivers reject requests older than 5 minutes.

### Signature Verification (receiver)

```
function verify_webhook(request, secret):
  timestamp = request.header("X-Webhook-Timestamp")
  signature = request.header("X-Webhook-Signature").strip_prefix("sha256=")

  # Reject stale deliveries (replay protection)
  if abs(now() - timestamp) > 300:
    return 403 "Timestamp too old"

  expected = hmac_sha256(secret, f"{timestamp}.{request.body}")
  if not constant_time_compare(expected, signature):
    return 401 "Invalid signature"

  return 200   # ACK immediately, process async via queue
```

### Retry Policy

| Attempt | Delay | Cumulative |
|---------|-------|-----------|
| 1 | Immediate | 0s |
| 2 | 60s | 1 min |
| 3 | 300s | 6 min |
| 4 | 3600s | 1 hr 6 min |
| 5 | 86400s | ~25 hrs |

After 5 failures: mark endpoint as disabled. Notify subscriber via email or dashboard. Store every delivery attempt (timestamp, HTTP status, latency) for debugging. Expose delivery log via admin API.

### Registration

```
POST /webhooks
{
  "url": "https://partner.example.com/hooks/orders",
  "events": ["order.completed", "order.refunded"],
  "secret": "whsec_..."   # generated by your system, shown once
}
```

Let subscribers filter by event type. Generate the signing secret server-side — don't let subscribers choose it.