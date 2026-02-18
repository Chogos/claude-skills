# Go Standard Library Cheatsheet

Quick reference for the most-used standard library packages. Each entry shows the signature and a one-liner example.

## io

**`io.Reader`** — the universal read interface
```go
type Reader interface { Read(p []byte) (n int, err error) }
```

**`io.Writer`** — the universal write interface
```go
type Writer interface { Write(p []byte) (n int, err error) }
```

**`io.Copy(dst Writer, src Reader) (int64, error)`** — stream bytes without buffering into memory
```go
n, err := io.Copy(os.Stdout, resp.Body)
```

**`io.ReadAll(r Reader) ([]byte, error)`** — read entire stream into memory
```go
data, err := io.ReadAll(resp.Body)
```

**`io.NopCloser(r Reader) ReadCloser`** — wrap a Reader to satisfy ReadCloser
```go
rc := io.NopCloser(strings.NewReader("hello"))
```

**`io.LimitReader(r Reader, n int64) Reader`** — cap reads to n bytes
```go
limited := io.LimitReader(resp.Body, 1<<20) // 1 MB max
```

## net/http

**`http.HandlerFunc`** — adapt a function to the Handler interface
```go
mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
})
```

**`http.ServeMux`** (Go 1.22+ pattern matching) — method and path patterns
```go
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", getUser)
mux.HandleFunc("POST /users", createUser)
// Extract path value: r.PathValue("id")
```

**`http.Client`** — always set a timeout
```go
client := &http.Client{Timeout: 10 * time.Second}
resp, err := client.Get("https://api.example.com/data")
```

**`http.NewRequest(method, url, body) (*Request, error)`** — build requests with headers
```go
req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, bytes.NewReader(payload))
req.Header.Set("Content-Type", "application/json")
resp, err := client.Do(req)
```

**`http.Error(w, msg, code)`** — write a plain-text error response
```go
http.Error(w, "not found", http.StatusNotFound)
```

## encoding/json

**`json.Marshal(v any) ([]byte, error)`** — struct to JSON bytes
```go
data, err := json.Marshal(user)
```

**`json.Unmarshal(data []byte, v any) error`** — JSON bytes to struct
```go
var user User
err := json.Unmarshal(data, &user)
```

**Struct tags** — control field names, omission, string encoding
```go
type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`
    Age   int    `json:"age,string"` // encode int as JSON string
}
```

**`json.NewDecoder(r Reader)`** — stream-decode from a reader
```go
var user User
err := json.NewDecoder(r.Body).Decode(&user)
```

**`json.NewEncoder(w Writer)`** — stream-encode to a writer
```go
json.NewEncoder(w).Encode(response)
```

**`json.RawMessage`** — defer parsing of a JSON field
```go
type Event struct {
    Type    string          `json:"type"`
    Payload json.RawMessage `json:"payload"`
}
```

## fmt

**`fmt.Sprintf(format, args...) string`** — formatted string
```go
msg := fmt.Sprintf("user %s has %d items", name, count)
```

**`fmt.Errorf(format, args...) error`** — formatted error, `%w` for wrapping
```go
return fmt.Errorf("fetch user %s: %w", id, err)
```

**`fmt.Stringer`** — implement for custom string output
```go
func (u User) String() string { return fmt.Sprintf("%s <%s>", u.Name, u.Email) }
```

**`fmt.Fprintf(w Writer, format, args...)`** — write formatted to any Writer
```go
fmt.Fprintf(os.Stderr, "error: %v\n", err)
```

## os

**`os.ReadFile(name string) ([]byte, error)`** — read entire file
```go
data, err := os.ReadFile("config.json")
```

**`os.WriteFile(name string, data []byte, perm FileMode) error`** — write entire file
```go
err := os.WriteFile("output.txt", data, 0o644)
```

**`os.Getenv(key string) string`** — read env var (empty string if unset)
```go
port := os.Getenv("PORT")
```

**`os.LookupEnv(key string) (string, bool)`** — distinguish unset from empty
```go
val, ok := os.LookupEnv("DATABASE_URL")
```

**`os.MkdirAll(path string, perm FileMode) error`** — recursive mkdir
```go
err := os.MkdirAll("data/cache", 0o755)
```

## context

**`context.WithTimeout(parent, duration) (Context, CancelFunc)`** — deadline after duration
```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

**`context.WithCancel(parent) (Context, CancelFunc)`** — manual cancellation
```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

**`context.WithValue(parent, key, val) Context`** — attach request-scoped metadata
```go
type ctxKey string
const traceIDKey ctxKey = "traceID"
ctx = context.WithValue(ctx, traceIDKey, "abc-123")
traceID := ctx.Value(traceIDKey).(string)
```

**`context.AfterFunc(ctx, func()) (stop func() bool)`** (Go 1.21+) — run cleanup when ctx is done
```go
stop := context.AfterFunc(ctx, func() { conn.Close() })
defer stop()
```

## sync

**`sync.WaitGroup`** — wait for a group of goroutines
```go
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func() {
        defer wg.Done()
        process(item)
    }()
}
wg.Wait()
```

**`sync.Mutex`** — mutual exclusion
```go
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()
counter++
```

**`sync.RWMutex`** — multiple concurrent readers, exclusive writer
```go
var mu sync.RWMutex
mu.RLock()         // many readers OK
defer mu.RUnlock()
return cache[key]
```

**`sync.Once`** — run a function exactly once
```go
var once sync.Once
once.Do(func() { db = connectDB() })
```

**`sync.Pool`** — reuse temporary objects to reduce allocations
```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}
buf := bufPool.Get().(*bytes.Buffer)
buf.Reset()
defer bufPool.Put(buf)
```

## time

**`time.Duration`** — use named constants, never raw integers
```go
timeout := 30 * time.Second
interval := 500 * time.Millisecond
```

**`time.NewTicker(d Duration) *Ticker`** — periodic events
```go
ticker := time.NewTicker(10 * time.Second)
defer ticker.Stop()
for range ticker.C { doWork() }
```

**`time.Parse(layout, value string) (Time, error)`** — parse time with reference layout
```go
t, err := time.Parse(time.RFC3339, "2024-01-15T10:30:00Z")
```

**Reference layouts** — the Go way (Mon Jan 2 15:04:05 MST 2006)
```go
time.RFC3339     // "2006-01-02T15:04:05Z07:00"
time.DateTime    // "2006-01-02 15:04:05"
time.DateOnly    // "2006-01-02"
time.TimeOnly    // "15:04:05"
```

**`time.Since(t Time) Duration`** — elapsed time since t
```go
start := time.Now()
doWork()
log.Printf("took %v", time.Since(start))
```

## strings / strconv

**`strings.Builder`** — efficient string concatenation
```go
var b strings.Builder
b.WriteString("hello ")
b.WriteString("world")
s := b.String()
```

**`strings.Cut(s, sep string) (before, after string, found bool)`** — split at first separator
```go
key, value, ok := strings.Cut("HOST=localhost", "=")
// key="HOST", value="localhost", ok=true
```

**`strings.Contains(s, substr string) bool`** — substring check
```go
if strings.Contains(line, "ERROR") { handle(line) }
```

**`strings.TrimSpace(s string) string`** — strip leading/trailing whitespace
```go
clean := strings.TrimSpace(input)
```

**`strconv.Atoi(s string) (int, error)`** — string to int
```go
n, err := strconv.Atoi(os.Getenv("PORT"))
```

**`strconv.Itoa(i int) string`** — int to string
```go
s := strconv.Itoa(8080)
```

**`strconv.ParseBool(str string) (bool, error)`** — "true", "1", "t" etc.
```go
debug, err := strconv.ParseBool(os.Getenv("DEBUG"))
```

## errors

**`errors.New(text string) error`** — create sentinel error
```go
var ErrNotFound = errors.New("not found")
```

**`errors.Is(err, target error) bool`** — check error identity through wrapping
```go
if errors.Is(err, os.ErrNotExist) { createFile() }
```

**`errors.As(err error, target any) bool`** — extract typed error through wrapping
```go
var pe *os.PathError
if errors.As(err, &pe) { log.Println(pe.Path) }
```

**`errors.Join(errs ...error) error`** (Go 1.20+) — combine multiple errors
```go
err := errors.Join(validate(a), validate(b), validate(c))
```

**`fmt.Errorf("...: %w", err)`** — wrap error with context
```go
return fmt.Errorf("open config: %w", err)
```

## log/slog (Go 1.21+)

**`slog.New(handler)`** — create a structured logger
```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
```

**`slog.Info(msg, attrs...)`** — log with key-value pairs
```go
slog.Info("request handled", slog.String("path", r.URL.Path), slog.Int("status", 200))
```

**`slog.With(attrs...)`** — create child logger with preset fields
```go
reqLogger := logger.With(slog.String("request_id", reqID))
reqLogger.Info("processing") // includes request_id
```

**`slog.Group(name, attrs...)`** — nest attributes under a key
```go
slog.Info("done", slog.Group("http", slog.String("method", "GET"), slog.Int("status", 200)))
// {"msg":"done","http":{"method":"GET","status":200}}
```

## iter (Go 1.23+)

**`iter.Seq[V]`** — single-value iterator type
```go
type Seq[V any] func(yield func(V) bool)
```

**`iter.Seq2[K, V]`** — two-value iterator type (key-value, value-error)
```go
type Seq2[K, V any] func(yield func(K, V) bool)
```

## slices (Go 1.21+)

**`slices.Collect(seq iter.Seq[E]) []E`** (Go 1.23+) — materialize iterator into slice
```go
keys := slices.Collect(maps.Keys(m))
```

**`slices.SortFunc(x []E, cmp func(a, b E) int)`** — sort with custom comparator
```go
slices.SortFunc(users, func(a, b User) int { return cmp.Compare(a.Name, b.Name) })
```

**`slices.Contains(s []E, v E) bool`** — membership test
```go
if slices.Contains(roles, "admin") { grant() }
```

## maps (Go 1.21+)

**`maps.Keys(m map[K]V) iter.Seq[K]`** (Go 1.23+) — iterate over map keys
```go
for k := range maps.Keys(m) { fmt.Println(k) }
```

**`maps.Values(m map[K]V) iter.Seq[V]`** (Go 1.23+) — iterate over map values
```go
for v := range maps.Values(m) { fmt.Println(v) }
```

**`maps.Clone(m map[K]V) map[K]V`** — shallow copy
```go
copy := maps.Clone(original)
```