# HTTP Patterns

## Middleware

Middleware wraps an `http.Handler`, returning a new `http.Handler`:

```go
type Middleware func(http.Handler) http.Handler
```

### Logging Middleware

```go
func withLogging(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            sw := &statusWriter{ResponseWriter: w, status: http.StatusOK}

            next.ServeHTTP(sw, r)

            logger.InfoContext(r.Context(), "request completed",
                slog.String("method", r.Method),
                slog.String("path", r.URL.Path),
                slog.Int("status", sw.status),
                slog.Duration("latency", time.Since(start)),
            )
        })
    }
}

type statusWriter struct {
    http.ResponseWriter
    status int
}

func (w *statusWriter) WriteHeader(code int) {
    w.status = code
    w.ResponseWriter.WriteHeader(code)
}
```

### Recovery Middleware

```go
func withRecovery(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if v := recover(); v != nil {
                    logger.ErrorContext(r.Context(), "panic recovered",
                        slog.Any("panic", v),
                        slog.String("stack", string(debug.Stack())),
                    )
                    http.Error(w, "internal server error", http.StatusInternalServerError)
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}
```

### Request ID Middleware

```go
func withRequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = uuid.NewString()
        }
        ctx := context.WithValue(r.Context(), requestIDKey, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

type ctxKey string

const requestIDKey ctxKey = "requestID"

func RequestID(ctx context.Context) string {
    id, _ := ctx.Value(requestIDKey).(string)
    return id
}
```

### Timeout Middleware

```go
func withTimeout(d time.Duration) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), d)
            defer cancel()
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

## Middleware Chaining

Apply in order — outermost runs first:

```go
func chain(h http.Handler, mw ...Middleware) http.Handler {
    for i := len(mw) - 1; i >= 0; i-- {
        h = mw[i](h)
    }
    return h
}

// Usage: recovery → request ID → logging → timeout → handler
handler := chain(mux,
    withRecovery(logger),
    withRequestID,
    withLogging(logger),
    withTimeout(30*time.Second),
)
```

## JSON Request / Response

### Decode Request Body

```go
func decode[T any](r *http.Request) (T, error) {
    var v T
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()
    if err := dec.Decode(&v); err != nil {
        return v, fmt.Errorf("decode json: %w", err)
    }
    return v, nil
}
```

### Encode Response

```go
func encode[T any](w http.ResponseWriter, status int, v T) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(v)
}
```

### Error Response

```go
type apiError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

func writeError(w http.ResponseWriter, status int, code, message string) {
    encode(w, status, map[string]apiError{
        "error": {Code: code, Message: message},
    })
}
```

### Handler Example

```go
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    req, err := decode[CreateOrderRequest](r)
    if err != nil {
        writeError(w, http.StatusBadRequest, "INVALID_JSON", err.Error())
        return
    }
    if problems := req.validate(); len(problems) > 0 {
        encode(w, http.StatusUnprocessableEntity, map[string]any{
            "error": map[string]any{
                "code":    "VALIDATION_FAILED",
                "message": "Request validation failed",
                "details": problems,
            },
        })
        return
    }

    order, err := h.svc.CreateOrder(r.Context(), req)
    if err != nil {
        h.handleServiceError(w, err)
        return
    }
    encode(w, http.StatusCreated, order)
}
```

## Request Validation

Keep validation on the request struct:

```go
type CreateOrderRequest struct {
    ProductID string `json:"product_id"`
    Quantity  int    `json:"quantity"`
}

type fieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func (r CreateOrderRequest) validate() []fieldError {
    var problems []fieldError
    if r.ProductID == "" {
        problems = append(problems, fieldError{"product_id", "is required"})
    }
    if r.Quantity < 1 {
        problems = append(problems, fieldError{"quantity", "must be at least 1"})
    }
    return problems
}
```

## Route Organization

Group routes by domain. Use `Handle` with a prefix to mount sub-routers:

```go
func addRoutes(mux *http.ServeMux, h *Handler) {
    // Health
    mux.HandleFunc("GET /health/live", func(w http.ResponseWriter, _ *http.Request) {
        w.WriteHeader(http.StatusOK)
    })
    mux.HandleFunc("GET /health/ready", h.HealthReady)

    // Orders
    mux.HandleFunc("GET /orders", h.ListOrders)
    mux.HandleFunc("POST /orders", h.CreateOrder)
    mux.HandleFunc("GET /orders/{id}", h.GetOrder)
    mux.HandleFunc("PATCH /orders/{id}", h.UpdateOrder)
    mux.HandleFunc("DELETE /orders/{id}", h.DeleteOrder)

    // Users
    mux.HandleFunc("GET /users/{id}", h.GetUser)
    mux.HandleFunc("POST /users", h.CreateUser)
}
```

Extract path parameters with `r.PathValue("id")`.

## Server Setup

Wire everything into a single `run` function that `main` calls:

```go
func run(ctx context.Context, cfg Config, logger *slog.Logger) error {
    db, err := openDB(ctx, cfg.DatabaseURL)
    if err != nil {
        return fmt.Errorf("open db: %w", err)
    }
    defer db.Close()

    svc := NewService(db, logger)
    handler := NewHandler(svc, logger)

    mux := http.NewServeMux()
    addRoutes(mux, handler)

    srv := &http.Server{
        Addr:         cfg.Addr,
        Handler:      chain(mux, withRecovery(logger), withRequestID, withLogging(logger)),
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // Start server
    errCh := make(chan error, 1)
    go func() { errCh <- srv.ListenAndServe() }()
    logger.Info("server started", slog.String("addr", cfg.Addr))

    // Wait for shutdown signal or server error
    select {
    case err := <-errCh:
        return err
    case <-ctx.Done():
    }

    // Graceful shutdown
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    return srv.Shutdown(shutdownCtx)
}
```

### Main

```go
func main() {
    ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
    defer cancel()

    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    cfg := loadConfig()

    if err := run(ctx, cfg, logger); err != nil && !errors.Is(err, http.ErrServerClosed) {
        logger.Error("server failed", slog.String("error", err.Error()))
        os.Exit(1)
    }
}
```
