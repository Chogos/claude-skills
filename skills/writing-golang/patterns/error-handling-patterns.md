# Error Handling Patterns

## Error Wrapping Chain

Build context as errors propagate up the call stack. Each layer adds its own context with `%w`.

```go
// repository layer
func (r *Repo) GetOrder(ctx context.Context, id string) (*Order, error) {
    row := r.db.QueryRowContext(ctx, "SELECT id, total FROM orders WHERE id = $1", id)
    var o Order
    if err := row.Scan(&o.ID, &o.Total); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("get order %s: %w", id, ErrNotFound)
        }
        return nil, fmt.Errorf("get order %s: %w", id, err)
    }
    return &o, nil
}

// service layer
func (s *Service) ProcessOrder(ctx context.Context, id string) error {
    order, err := s.repo.GetOrder(ctx, id)
    if err != nil {
        return fmt.Errorf("process order: %w", err)
    }
    if err := s.validate(order); err != nil {
        return fmt.Errorf("process order: %w", err)
    }
    return nil
}

// handler layer
func (h *Handler) HandleOrder(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    if err := h.svc.ProcessOrder(r.Context(), id); err != nil {
        h.writeError(w, err)
        return
    }
    w.WriteHeader(http.StatusOK)
}
```

The resulting error reads like a stack trace: `process order: get order abc123: not found`.

## Sentinel Errors

Define package-level sentinel errors for expected conditions callers need to handle.

```go
package order

import "errors"

var (
    ErrNotFound    = errors.New("order not found")
    ErrOutOfStock  = errors.New("item out of stock")
    ErrPaymentFail = errors.New("payment failed")
)
```

Check sentinels with `errors.Is` — it unwraps the chain automatically:

```go
err := svc.ProcessOrder(ctx, id)

// Works even if err is: "process order: get order abc: order not found"
if errors.Is(err, order.ErrNotFound) {
    http.Error(w, "order not found", http.StatusNotFound)
    return
}
if errors.Is(err, order.ErrOutOfStock) {
    http.Error(w, "out of stock", http.StatusConflict)
    return
}
```

## Custom Error Types

Use when callers need structured data from the error, not just identity.

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: field %q %s", e.Field, e.Message)
}

func ValidateOrder(o *Order) error {
    if o.Quantity <= 0 {
        return &ValidationError{Field: "quantity", Message: "must be positive"}
    }
    if o.Total < 0 {
        return &ValidationError{Field: "total", Message: "must not be negative"}
    }
    return nil
}
```

Extract with `errors.As` — also unwraps through the chain:

```go
var ve *ValidationError
if errors.As(err, &ve) {
    log.Printf("bad field: %s — %s", ve.Field, ve.Message)
    // respond with 400 and field-level detail
}
```

Implement `Unwrap() error` on custom types that wrap another error:

```go
type QueryError struct {
    Query string
    Err   error
}

func (e *QueryError) Error() string { return fmt.Sprintf("query %q: %v", e.Query, e.Err) }
func (e *QueryError) Unwrap() error { return e.Err }
```

## Multi-Error Aggregation (Go 1.20+)

Use `errors.Join` when collecting errors from multiple independent operations.

```go
func ValidateConfig(cfg *Config) error {
    var errs []error
    if cfg.Host == "" {
        errs = append(errs, errors.New("host is required"))
    }
    if cfg.Port < 1 || cfg.Port > 65535 {
        errs = append(errs, fmt.Errorf("invalid port: %d", cfg.Port))
    }
    if cfg.Timeout <= 0 {
        errs = append(errs, errors.New("timeout must be positive"))
    }
    return errors.Join(errs...)
}
```

The joined error's message concatenates with newlines. `errors.Is` and `errors.As` check each wrapped error:

```go
err := ValidateConfig(cfg)
// errors.Is(err, target) returns true if ANY of the joined errors match
```

Useful in cleanup paths too:

```go
func cleanup(db *sql.DB, f *os.File) error {
    return errors.Join(db.Close(), f.Close())
}
```

## Generic Error Extraction (Go 1.26+)

`errors.AsType` is a generic, type-safe alternative to `errors.As` — no need to declare a pointer variable:

```go
// Before (errors.As)
var ve *ValidationError
if errors.As(err, &ve) {
    log.Printf("bad field: %s", ve.Field)
}

// After (errors.AsType)
if ve, ok := errors.AsType[*ValidationError](err); ok {
    log.Printf("bad field: %s", ve.Field)
}
```

Works through wrapping chains just like `errors.As`.

## HTTP Error Response Mapping

Map domain errors to HTTP status codes in one place. Keep domain packages free of HTTP concerns.

```go
package api

import (
    "encoding/json"
    "errors"
    "net/http"

    "github.com/org/app/internal/order"
)

type ErrorResponse struct {
    Error  string `json:"error"`
    Detail string `json:"detail,omitempty"`
}

func writeError(w http.ResponseWriter, err error) {
    var (
        status int
        resp   ErrorResponse
    )

    var ve *order.ValidationError
    switch {
    case errors.Is(err, order.ErrNotFound):
        status = http.StatusNotFound
        resp = ErrorResponse{Error: "not_found"}
    case errors.As(err, &ve):
        status = http.StatusBadRequest
        resp = ErrorResponse{Error: "validation_error", Detail: ve.Message}
    case errors.Is(err, order.ErrOutOfStock):
        status = http.StatusConflict
        resp = ErrorResponse{Error: "out_of_stock"}
    default:
        // Log the full error; return generic message to client
        slog.Error("unhandled error", "err", err)
        status = http.StatusInternalServerError
        resp = ErrorResponse{Error: "internal_error"}
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(resp)
}
```

Use in handlers:

```go
func (h *Handler) GetOrder(w http.ResponseWriter, r *http.Request) {
    o, err := h.svc.GetOrder(r.Context(), r.PathValue("id"))
    if err != nil {
        writeError(w, err)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(o)
}
```