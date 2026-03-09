# Testing Patterns

## Table-Driven Test Template

The standard Go test pattern. Each case is a struct with inputs and expected outputs.

```go
func TestCalculateDiscount(t *testing.T) {
    t.Parallel()
    tests := []struct {
        name    string
        price   float64
        qty     int
        want    float64
        wantErr bool
    }{
        {name: "no discount under 5", price: 10.0, qty: 3, want: 30.0},
        {name: "10% off for 5+", price: 10.0, qty: 5, want: 45.0},
        {name: "20% off for 10+", price: 10.0, qty: 10, want: 80.0},
        {name: "zero quantity", price: 10.0, qty: 0, want: 0},
        {name: "negative price", price: -5.0, qty: 1, wantErr: true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got, err := CalculateDiscount(tt.price, tt.qty)
            if (err != nil) != tt.wantErr {
                t.Fatalf("CalculateDiscount(%v, %d) error = %v, wantErr %v",
                    tt.price, tt.qty, err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("CalculateDiscount(%v, %d) = %v, want %v",
                    tt.price, tt.qty, got, tt.want)
            }
        })
    }
}
```

Key points:
- `t.Fatalf` for precondition failures (stops the subtest). `t.Errorf` for value mismatches (continues).
- `t.Parallel()` on both the parent and each subtest.
- Include the input values in error messages so failures are self-describing.

## httptest.NewRecorder for Handler Unit Tests

Test handlers in isolation without a running server.

```go
func TestCreateUserHandler(t *testing.T) {
    body := strings.NewReader(`{"name":"alice","email":"alice@example.com"}`)
    req := httptest.NewRequest(http.MethodPost, "/users", body)
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()

    store := &mockUserStore{} // see "Interface Mocking" below
    handler := NewUserHandler(store)
    handler.ServeHTTP(rec, req)

    if rec.Code != http.StatusCreated {
        t.Errorf("status = %d, want %d", rec.Code, http.StatusCreated)
    }

    var resp User
    if err := json.NewDecoder(rec.Body).Decode(&resp); err != nil {
        t.Fatalf("decode response: %v", err)
    }
    if resp.Name != "alice" {
        t.Errorf("name = %q, want %q", resp.Name, "alice")
    }
}
```

## httptest.NewServer for Integration Tests

Spin up a real HTTP server bound to a random port. Use for tests that need a full HTTP round-trip.

```go
func TestAPIClient(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/api/v1/status" {
            http.NotFound(w, r)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprint(w, `{"status":"ok"}`)
    }))
    t.Cleanup(srv.Close)

    client := NewAPIClient(srv.URL) // point client at test server
    status, err := client.GetStatus(context.Background())
    if err != nil {
        t.Fatalf("GetStatus: %v", err)
    }
    if status != "ok" {
        t.Errorf("status = %q, want %q", status, "ok")
    }
}
```

## Interface Mocking by Hand

Define a mock struct that implements the interface. No frameworks needed.

```go
// Interface defined in the consumer package
type UserStore interface {
    GetUser(ctx context.Context, id string) (*User, error)
    SaveUser(ctx context.Context, u *User) error
}

// Mock in the test file
type mockUserStore struct {
    getUserFunc func(ctx context.Context, id string) (*User, error)
    saveUserFunc func(ctx context.Context, u *User) error
}

func (m *mockUserStore) GetUser(ctx context.Context, id string) (*User, error) {
    return m.getUserFunc(ctx, id)
}

func (m *mockUserStore) SaveUser(ctx context.Context, u *User) error {
    return m.saveUserFunc(ctx, u)
}
```

Use in tests by setting the function fields:

```go
func TestServiceGetUser(t *testing.T) {
    store := &mockUserStore{
        getUserFunc: func(_ context.Context, id string) (*User, error) {
            if id == "abc" {
                return &User{ID: "abc", Name: "alice"}, nil
            }
            return nil, ErrNotFound
        },
    }
    svc := NewService(store)

    u, err := svc.GetUser(context.Background(), "abc")
    if err != nil {
        t.Fatalf("GetUser: %v", err)
    }
    if u.Name != "alice" {
        t.Errorf("name = %q, want %q", u.Name, "alice")
    }
}
```

For verifying calls were made, add a counter:

```go
store := &mockUserStore{
    saveUserFunc: func(_ context.Context, u *User) error {
        saveCalled++
        return nil
    },
}
// ... run test ...
if saveCalled != 1 {
    t.Errorf("SaveUser called %d times, want 1", saveCalled)
}
```

## Golden File Testing

Compare output against a known-good file in `testdata/`. Use `-update` flag to regenerate.

```go
var update = flag.Bool("update", false, "update golden files")

func TestRenderTemplate(t *testing.T) {
    got, err := RenderTemplate(testInput)
    if err != nil {
        t.Fatalf("RenderTemplate: %v", err)
    }

    golden := filepath.Join("testdata", t.Name()+".golden")

    if *update {
        os.MkdirAll("testdata", 0o755)
        if err := os.WriteFile(golden, got, 0o644); err != nil {
            t.Fatalf("write golden: %v", err)
        }
    }

    want, err := os.ReadFile(golden)
    if err != nil {
        t.Fatalf("read golden (run with -update to create): %v", err)
    }
    if !bytes.Equal(got, want) {
        t.Errorf("output mismatch (run with -update to regenerate)\ngot:\n%s\nwant:\n%s", got, want)
    }
}
```

Run `go test -update ./...` once to create baselines, then commit the `testdata/` files.

## t.Setenv for Environment-Dependent Tests

`t.Setenv` sets an env var for the duration of a single test and restores it on cleanup. Incompatible with `t.Parallel()`.

```go
func TestConfigFromEnv(t *testing.T) {
    t.Setenv("APP_PORT", "9090")
    t.Setenv("APP_HOST", "0.0.0.0")

    cfg, err := LoadConfigFromEnv()
    if err != nil {
        t.Fatalf("LoadConfigFromEnv: %v", err)
    }
    if cfg.Port != 9090 {
        t.Errorf("port = %d, want 9090", cfg.Port)
    }
    if cfg.Host != "0.0.0.0" {
        t.Errorf("host = %q, want %q", cfg.Host, "0.0.0.0")
    }
}
```

## testing/synctest for Concurrent Code (Go 1.25+)

Test goroutine-based code with virtualized time. Goroutines inside `synctest.Test` use a fake clock that advances instantaneously when all goroutines are blocked.

```go
func TestPeriodicRefresh(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        var count atomic.Int32
        ctx, cancel := context.WithCancel(context.Background())
        defer cancel()

        go func() {
            ticker := time.NewTicker(10 * time.Second)
            defer ticker.Stop()
            for {
                select {
                case <-ticker.C:
                    count.Add(1)
                case <-ctx.Done():
                    return
                }
            }
        }()

        // Advance fake clock by 30s — ticker fires 3 times
        time.Sleep(30 * time.Second)
        synctest.Wait() // wait for goroutine to process all ticks

        if got := count.Load(); got != 3 {
            t.Errorf("refresh count = %d, want 3", got)
        }
    })
}
```

`synctest.Wait()` blocks until all goroutines in the test bubble are idle. The fake clock only advances when all goroutines are blocked on time operations.

## Benchmark Template

Use `b.ResetTimer()` after expensive setup to exclude it from timing.

```go
func BenchmarkJSONMarshal(b *testing.B) {
    data := generateLargeStruct() // expensive setup
    b.ResetTimer()

    for range b.N {
        _, err := json.Marshal(data)
        if err != nil {
            b.Fatal(err)
        }
    }
}

func BenchmarkConcurrentMap(b *testing.B) {
    m := &sync.Map{}
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            m.Store(i, i)
            i++
        }
    })
}
```

Run: `go test -bench=. -benchmem ./...`

Output columns: `ns/op`, `B/op`, `allocs/op`. Compare across changes with `benchstat`.