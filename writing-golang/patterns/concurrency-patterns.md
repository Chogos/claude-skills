# Concurrency Patterns

## Goroutine with Context Cancellation

Every goroutine must select on `ctx.Done()` to support clean shutdown.

```go
func produce(ctx context.Context, out chan<- Event) error {
    defer close(out)
    for {
        event, err := pollEvent()
        if err != nil {
            return fmt.Errorf("poll: %w", err)
        }
        select {
        case out <- event:
        case <-ctx.Done():
            return ctx.Err()
        }
    }
}
```

## errgroup.Group for Parallel Tasks

First error cancels the derived context, stopping all other goroutines.

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]Response, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]Response, len(urls))

    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            resp, err := fetch(ctx, url)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", url, err)
            }
            results[i] = resp
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

Limit concurrency with `g.SetLimit(n)`:

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10) // max 10 goroutines at once

for _, item := range items {
    g.Go(func() error {
        return process(ctx, item)
    })
}
return g.Wait()
```

## Worker Pool with Bounded Concurrency

Use a buffered channel as a semaphore when you need more control than `errgroup.SetLimit`.

```go
func processPool(ctx context.Context, jobs []Job, workers int) error {
    sem := make(chan struct{}, workers)
    g, ctx := errgroup.WithContext(ctx)

    for _, job := range jobs {
        select {
        case sem <- struct{}{}:
        case <-ctx.Done():
            break
        }

        g.Go(func() error {
            defer func() { <-sem }()
            return handle(ctx, job)
        })
    }

    return g.Wait()
}
```

## Fan-Out / Fan-In

Distribute work across N goroutines, collect results into a single channel.

```go
func fanOutFanIn(ctx context.Context, input <-chan Task, workers int) <-chan Result {
    results := make(chan Result)
    var wg sync.WaitGroup

    // Fan-out: launch N workers
    for range workers {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range input {
                select {
                case results <- doWork(ctx, task):
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    // Fan-in: close results when all workers finish
    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```

The sender closes `input` when done. Workers drain it and exit. The fan-in goroutine closes `results` after all workers complete.

## sync.Once for Expensive Initialization

Safe for concurrent access. The function runs exactly once regardless of how many goroutines call it.

```go
type Client struct {
    initOnce sync.Once
    conn     *grpc.ClientConn
    initErr  error
}

func (c *Client) getConn() (*grpc.ClientConn, error) {
    c.initOnce.Do(func() {
        c.conn, c.initErr = grpc.Dial("localhost:9090", grpc.WithInsecure())
    })
    return c.conn, c.initErr
}
```

Use `sync.OnceValue` (Go 1.21+) for cleaner single-value init:

```go
var getConfig = sync.OnceValue(func() *Config {
    cfg, err := loadConfig()
    if err != nil {
        panic(fmt.Sprintf("load config: %v", err))
    }
    return cfg
})
```

## Periodic Task with time.Ticker

Combine `time.Ticker` with context for cancellable periodic work.

```go
func runPeriodic(ctx context.Context, interval time.Duration, fn func(context.Context) error) error {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    // Run immediately on start
    if err := fn(ctx); err != nil {
        return err
    }

    for {
        select {
        case <-ticker.C:
            if err := fn(ctx); err != nil {
                return err
            }
        case <-ctx.Done():
            return ctx.Err()
        }
    }
}
```

Usage:

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

err := runPeriodic(ctx, 30*time.Second, func(ctx context.Context) error {
    return healthCheck(ctx)
})
```

## Graceful HTTP Server Shutdown

Accept new connections until signal, then drain in-flight requests.

```go
func runServer(ctx context.Context, addr string, handler http.Handler) error {
    srv := &http.Server{
        Addr:         addr,
        Handler:      handler,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // Shutdown goroutine: wait for context cancellation
    shutdownErr := make(chan error, 1)
    go func() {
        <-ctx.Done()
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        shutdownErr <- srv.Shutdown(shutdownCtx)
    }()

    // Start serving — blocks until Shutdown is called
    if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        return fmt.Errorf("listen: %w", err)
    }

    // Wait for in-flight requests to drain
    if err := <-shutdownErr; err != nil {
        return fmt.Errorf("shutdown: %w", err)
    }
    return nil
}
```

Wire it up in `main`:

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    mux := http.NewServeMux()
    mux.HandleFunc("GET /health", healthHandler)

    if err := runServer(ctx, ":8080", mux); err != nil {
        log.Fatal(err)
    }
}
```