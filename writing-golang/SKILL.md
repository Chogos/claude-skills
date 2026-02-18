---
name: writing-golang
description: Go development best practices. Use when writing, modifying, or reviewing .go files, go.mod, go.sum, Go packages, or Go module configurations.
---

# Go Development Best Practices

## Project Layout

```
myapp/
├── cmd/
│   ├── server/
│   │   └── main.go        # binary entry point
│   └── cli/
│       └── main.go
├── internal/               # private application code
│   ├── auth/
│   ├── order/
│   └── platform/
│       └── postgres/
├── pkg/                    # only if other repos import it
│   └── client/
├── go.mod
└── go.sum
```

- `cmd/<name>/main.go` — one directory per binary, minimal code (parse flags, wire dependencies, call `run()`).
- `internal/` — all private application code. The compiler enforces the boundary.
- `pkg/` — only create when external consumers exist. Default to `internal/`.
- No `src/`, `util/`, `common/`, `helpers/` packages. Each package has a single clear purpose.
- Package names: short, lowercase, singular, no underscores. `order` not `orders`, `http` not `httpUtils`.

## Error Handling

Return errors; don't panic. Panics are for unrecoverable programmer bugs, not runtime failures.

**Wrap with context using `%w`:**
```go
func (r *Repo) FindUser(ctx context.Context, id string) (*User, error) {
    row := r.db.QueryRowContext(ctx, "SELECT ...", id)
    var u User
    if err := row.Scan(&u.Name, &u.Email); err != nil {
        return nil, fmt.Errorf("find user %s: %w", id, err)
    }
    return &u, nil
}
```

**Sentinel errors for expected conditions:**
```go
var ErrNotFound = errors.New("not found")

// Check with errors.Is — works through wrapping chains
if errors.Is(err, ErrNotFound) {
    http.Error(w, "not found", http.StatusNotFound)
    return
}
```

**Custom error types for structured error data:**
```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: %s %s", e.Field, e.Message)
}

// Extract with errors.As
var ve *ValidationError
if errors.As(err, &ve) {
    log.Printf("invalid field: %s", ve.Field)
}
```

**Rules:**
- Never ignore errors. Use `_ = f.Close()` with a comment if intentional.
- Never `log.Fatal` or `os.Exit` in library code — only in `main`.
- Standard pattern: `if err != nil { return ..., fmt.Errorf("context: %w", err) }`.
- Use `errors.Join` (Go 1.20+) when aggregating multiple errors.

## Interfaces

Define interfaces where they are consumed, not where they are implemented.

```go
// In the consumer package (service), not in the provider package (postgres)
type UserStore interface {
    FindUser(ctx context.Context, id string) (*User, error)
}

type Service struct {
    store UserStore // accept interface
}

func NewService(s UserStore) *Service {
    return &Service{store: s} // return struct
}
```

- Keep interfaces small — 1-2 methods. Compose larger ones from smaller.
- Accept interfaces, return concrete structs.
- Don't export interfaces until a second consumer exists.
- Standard library models: `io.Reader` (1 method), `io.ReadWriter` (composed).

## Context

Pass `context.Context` as the first parameter to any function that does I/O or may be long-running.

```go
func (s *Service) Process(ctx context.Context, id string) error {
    // Derive child context with timeout; always defer cancel
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    result, err := s.store.Fetch(ctx, id)
    if err != nil {
        return fmt.Errorf("fetch %s: %w", id, err)
    }

    // Check cancellation before expensive work
    if err := ctx.Err(); err != nil {
        return fmt.Errorf("process %s: %w", id, err)
    }

    return s.transform(ctx, result)
}
```

- Never store `context.Context` in a struct field.
- Always `defer cancel()` when using `WithTimeout` or `WithCancel`.
- `context.WithValue` — only for request-scoped metadata (trace IDs, auth tokens), never for function parameters.
- `context.AfterFunc` (Go 1.21+) — register cleanup when context is done.

## Goroutines & Channels

Every goroutine must have a clear shutdown path. Leaked goroutines are memory leaks.

**Use `errgroup.Group` for managed goroutine lifecycles:**
```go
import "golang.org/x/sync/errgroup"

func processAll(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    for _, item := range items {
        g.Go(func() error {
            return process(ctx, item)
        })
    }
    return g.Wait() // returns first non-nil error, cancels ctx
}
```

**Channel rules:**
- The creator (sender) closes the channel, never the receiver.
- Buffered channels for known-size work queues. Unbuffered for synchronization.
- Always pair channel operations with `select` and `ctx.Done()`:

```go
select {
case result := <-ch:
    handle(result)
case <-ctx.Done():
    return ctx.Err()
}
```

## Testing

**Table-driven tests:**
```go
func TestParseSize(t *testing.T) {
    t.Parallel()
    tests := []struct {
        name    string
        input   string
        want    int64
        wantErr bool
    }{
        {name: "bytes", input: "100B", want: 100},
        {name: "kilobytes", input: "2KB", want: 2048},
        {name: "empty", input: "", wantErr: true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got, err := ParseSize(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("ParseSize(%q) error = %v, wantErr %v", tt.input, err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("ParseSize(%q) = %d, want %d", tt.input, got, tt.want)
            }
        })
    }
}
```

**Test helpers:**
```go
func newTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() { db.Close() })
    return db
}
```

**HTTP testing:**
```go
func TestHealthHandler(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/health", nil)
    rec := httptest.NewRecorder()
    HealthHandler(rec, req)

    if rec.Code != http.StatusOK {
        t.Errorf("status = %d, want %d", rec.Code, http.StatusOK)
    }
}
```

**Rules:**
- Tests live in `_test.go` files in the same package (whitebox) or `_test` package (blackbox).
- No assertion libraries — use `if/t.Errorf`. The standard library is sufficient.
- Always run with `-race`: `go test -race -cover ./...`.
- Use `t.Parallel()` in tests and subtests when there are no shared mutable resources.
- Use `fstest.MapFS` for testing code that reads files.

## Build Tags

Use `//go:build` (Go 1.17+, replaces `// +build`) for conditional compilation:

```go
//go:build linux

package myapp
// This file is only compiled on Linux
```

Common patterns:
```go
//go:build integration        // only in: go test -tags=integration ./...
//go:build !windows            // all platforms except Windows
//go:build linux && amd64      // Linux on x86-64 only
```

Name files with platform suffix for automatic build tags: `file_linux.go`, `file_darwin.go`, `file_test.go`.

## Range Over Integers (Go 1.22+)

```go
for i := range 10 {
    fmt.Println(i) // 0, 1, 2, ..., 9
}
```

Replaces `for i := 0; i < 10; i++` for simple counted loops.

## Modules

```bash
go mod init github.com/org/repo     # full repo URL
go mod tidy                          # after adding/removing imports
```

- Commit both `go.mod` and `go.sum`.
- `go.work` is for local multi-module development only — never commit it.
- Set minimum Go version in `go.mod`:
  ```
  go 1.22
  toolchain go1.22.5
  ```

## Linting

Run `golangci-lint` with a `.golangci.yml` config at the repo root.

```yaml
# .golangci.yml
linters:
  enable:
    # minimum — always on
    - errcheck
    - govet
    - staticcheck
    - unused
    - gosimple
    - ineffassign
    # recommended
    - revive
    - gosec
    - prealloc
    - noctx
linters-settings:
  revive:
    rules:
      - name: exported
        arguments: [checkPrivateReceivers]
  errcheck:
    check-type-assertions: true
```

Run: `golangci-lint run ./...`

## Naming

- Short names in small scopes: `i`, `r` for reader, `ctx` for context, `err` for error.
- Longer descriptive names in larger scopes: `userRepository`, `orderService`.
- Exported: `PascalCase`. Unexported: `camelCase`.
- Acronyms: all caps — `ID`, `HTTP`, `URL`, `API`. `userID` not `userId`.
- No `Get` prefix on getters: `u.Name()` not `u.GetName()`. Setters: `u.SetName()`.
- Packages: no underscores, no mixedCase. `httputil` not `http_util` or `httpUtil`.

## New project workflow

```
- [ ] go mod init github.com/org/repo
- [ ] Create cmd/<name>/main.go with minimal main()
- [ ] Create internal/ packages for domain logic
- [ ] Add .golangci.yml with linter config
- [ ] Add Makefile with build/test/lint targets
- [ ] go mod tidy
- [ ] Run validation loop (below)
- [ ] Commit go.mod, go.sum, and source files
```

## Validation loop

1. `gofmt -w .` — format all files (non-negotiable)
2. `go vet ./...` — catch common mistakes
3. `golangci-lint run ./...` — extended static analysis
4. `go test -race -cover ./...` — run tests with race detector
5. `go build ./...` — verify everything compiles
6. Repeat until all pass cleanly

## Deep-dive references

**Error handling patterns**: See [patterns/error-handling-patterns.md](patterns/error-handling-patterns.md) for wrapping chains, sentinel errors, custom types, multi-error aggregation, HTTP error mapping
**Concurrency patterns**: See [patterns/concurrency-patterns.md](patterns/concurrency-patterns.md) for worker pools, fan-out/fan-in, graceful shutdown, periodic tasks
**Testing patterns**: See [patterns/testing-patterns.md](patterns/testing-patterns.md) for table-driven tests, httptest, mocking, golden files, benchmarks
**Standard library**: See [stdlib-cheatsheet.md](stdlib-cheatsheet.md) for quick reference on io, net/http, encoding/json, context, sync, and more

## Official references

- [Effective Go](https://go.dev/doc/effective_go) — canonical style and idioms
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) — common review feedback, naming, error strings
- [Standard Library](https://pkg.go.dev/std) — package documentation and source