# Async Patterns

Production-ready async Rust patterns with tokio.

## Tokio runtime setup

Default multi-threaded runtime:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt::init();
    let config = Config::from_env()?;
    run(config).await
}
```

Explicit configuration for control over worker threads and thread names:

```rust
#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() -> anyhow::Result<()> {
    run().await
}
```

Single-threaded runtime for lightweight tools or tests:

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() -> anyhow::Result<()> {
    run().await
}
```

## Graceful shutdown with CancellationToken

Use `tokio_util::sync::CancellationToken` to propagate shutdown across tasks. Child tokens cancel when the parent cancels.

```rust
use anyhow::Result;
use tokio::signal;
use tokio_util::sync::CancellationToken;
use tracing::{info, error};

async fn run(config: Config) -> Result<()> {
    let token = CancellationToken::new();

    // Spawn the signal listener
    let shutdown_token = token.clone();
    tokio::spawn(async move {
        let ctrl_c = signal::ctrl_c();
        let mut sigterm = signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to register SIGTERM handler");

        tokio::select! {
            _ = ctrl_c => info!("received SIGINT"),
            _ = sigterm.recv() => info!("received SIGTERM"),
        }
        shutdown_token.cancel();
    });

    // Spawn application tasks with child tokens
    let server_handle = tokio::spawn(serve_http(config.clone(), token.child_token()));
    let worker_handle = tokio::spawn(process_queue(config, token.child_token()));

    // Wait for all tasks to complete
    let (server_res, worker_res) = tokio::join!(server_handle, worker_handle);

    if let Err(e) = server_res? {
        error!("server error: {e:#}");
    }
    if let Err(e) = worker_res? {
        error!("worker error: {e:#}");
    }

    info!("shutdown complete");
    Ok(())
}

async fn serve_http(config: Config, token: CancellationToken) -> Result<()> {
    let listener = tokio::net::TcpListener::bind(&config.bind_addr).await?;
    info!("listening on {}", config.bind_addr);

    loop {
        tokio::select! {
            accept = listener.accept() => {
                let (stream, addr) = accept?;
                let child_token = token.child_token();
                tokio::spawn(async move {
                    if let Err(e) = handle_connection(stream, child_token).await {
                        error!("connection {addr} error: {e:#}");
                    }
                });
            }
            _ = token.cancelled() => {
                info!("server shutting down");
                break;
            }
        }
    }
    Ok(())
}
```

## mpsc + oneshot channel patterns

Actor-like pattern: a task owns mutable state, receives commands via mpsc, responds via oneshot.

```rust
use tokio::sync::{mpsc, oneshot};
use anyhow::Result;

#[derive(Debug)]
enum CacheCommand {
    Get {
        key: String,
        reply: oneshot::Sender<Option<String>>,
    },
    Set {
        key: String,
        value: String,
        reply: oneshot::Sender<()>,
    },
    Delete {
        key: String,
        reply: oneshot::Sender<bool>,
    },
}

struct CacheActor {
    store: std::collections::HashMap<String, String>,
    rx: mpsc::Receiver<CacheCommand>,
}

impl CacheActor {
    fn new(rx: mpsc::Receiver<CacheCommand>) -> Self {
        Self {
            store: std::collections::HashMap::new(),
            rx,
        }
    }

    async fn run(mut self) {
        while let Some(cmd) = self.rx.recv().await {
            match cmd {
                CacheCommand::Get { key, reply } => {
                    let _ = reply.send(self.store.get(&key).cloned());
                }
                CacheCommand::Set { key, value, reply } => {
                    self.store.insert(key, value);
                    let _ = reply.send(());
                }
                CacheCommand::Delete { key, reply } => {
                    let _ = reply.send(self.store.remove(&key).is_some());
                }
            }
        }
    }
}

#[derive(Clone)]
struct CacheHandle {
    tx: mpsc::Sender<CacheCommand>,
}

impl CacheHandle {
    fn new(buffer: usize) -> Self {
        let (tx, rx) = mpsc::channel(buffer);
        tokio::spawn(CacheActor::new(rx).run());
        Self { tx }
    }

    async fn get(&self, key: &str) -> Result<Option<String>> {
        let (reply, rx) = oneshot::channel();
        self.tx.send(CacheCommand::Get {
            key: key.to_owned(),
            reply,
        }).await?;
        Ok(rx.await?)
    }

    async fn set(&self, key: &str, value: &str) -> Result<()> {
        let (reply, rx) = oneshot::channel();
        self.tx.send(CacheCommand::Set {
            key: key.to_owned(),
            value: value.to_owned(),
            reply,
        }).await?;
        Ok(rx.await?)
    }
}
```

## tokio::select! with cancellation safety

`select!` drops unfinished futures when one branch completes. This matters for futures that hold partial state.

**Safe**: operations that can be retried without data loss.

```rust
use tokio::time::{sleep, Duration};

async fn fetch_with_timeout(url: &str) -> Result<Response> {
    tokio::select! {
        response = reqwest::get(url) => {
            Ok(response?.error_for_status()?)
        }
        _ = sleep(Duration::from_secs(30)) => {
            anyhow::bail!("request timed out after 30s")
        }
    }
}
```

**Unsafe to cancel**: `mpsc::Receiver::recv()` is cancellation-safe (no message lost), but reading from a stream where partial data has been consumed is not. Use `tokio::pin!` and loop with `&mut future` to resume:

```rust
use tokio::pin;

async fn read_or_shutdown(
    reader: &mut tokio::io::BufReader<tokio::net::TcpStream>,
    token: &CancellationToken,
) -> Result<Option<String>> {
    let mut line = String::new();
    let read_fut = reader.read_line(&mut line);
    pin!(read_fut);

    tokio::select! {
        result = &mut read_fut => {
            let n = result?;
            if n == 0 { return Ok(None); }
            Ok(Some(line))
        }
        _ = token.cancelled() => Ok(None),
    }
}
```

## Connection pooling

Use `bb8` or `deadpool` for async connection pools. Both integrate with tokio.

```rust
use bb8::Pool;
use bb8_redis::RedisConnectionManager;
use anyhow::{Context, Result};

async fn create_redis_pool(url: &str) -> Result<Pool<RedisConnectionManager>> {
    let manager = RedisConnectionManager::new(url)
        .context("invalid redis URL")?;

    Pool::builder()
        .max_size(20)
        .min_idle(Some(5))
        .connection_timeout(Duration::from_secs(5))
        .build(manager)
        .await
        .context("failed to build redis pool")
}

async fn get_value(pool: &Pool<RedisConnectionManager>, key: &str) -> Result<Option<String>> {
    let mut conn = pool.get()
        .await
        .context("failed to get redis connection from pool")?;

    let value: Option<String> = redis::cmd("GET")
        .arg(key)
        .query_async(&mut *conn)
        .await?;

    Ok(value)
}
```

## spawn_blocking for CPU-bound work

Never run CPU-intensive or synchronous blocking IO on the tokio runtime. Offload to the blocking thread pool.

```rust
use anyhow::Result;

async fn hash_password(password: String) -> Result<String> {
    tokio::task::spawn_blocking(move || {
        bcrypt::hash(&password, bcrypt::DEFAULT_COST)
            .map_err(|e| anyhow::anyhow!("bcrypt failed: {e}"))
    })
    .await?
}

async fn compress_data(data: Vec<u8>) -> Result<Vec<u8>> {
    tokio::task::spawn_blocking(move || {
        let mut encoder = zstd::Encoder::new(Vec::new(), 3)?;
        std::io::Write::write_all(&mut encoder, &data)?;
        Ok(encoder.finish()?)
    })
    .await?
}
```

## Rate limiting with semaphore

Limit concurrent operations without a full connection pool:

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;
use anyhow::Result;

async fn fetch_all(urls: Vec<String>, max_concurrent: usize) -> Result<Vec<String>> {
    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    let mut handles = Vec::with_capacity(urls.len());

    for url in urls {
        let permit = semaphore.clone().acquire_owned().await?;
        handles.push(tokio::spawn(async move {
            let result = reqwest::get(&url).await?.text().await;
            drop(permit);
            result
        }));
    }

    let mut results = Vec::with_capacity(handles.len());
    for handle in handles {
        results.push(handle.await??);
    }
    Ok(results)
}