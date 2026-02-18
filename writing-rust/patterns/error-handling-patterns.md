# Error Handling Patterns

Production-ready error handling for Rust libraries and applications.

## Library error enum with thiserror

Define one error enum per crate. Use `#[from]` for automatic conversion from upstream errors. Add structured fields for context.

```rust
use std::path::PathBuf;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Error {
    #[error("io error reading {path}: {source}")]
    Io {
        source: std::io::Error,
        path: PathBuf,
    },

    #[error("failed to parse config at line {line}: {message}")]
    Parse { line: usize, message: String },

    #[error("unknown format: {0}")]
    UnknownFormat(String),

    #[error("connection failed after {attempts} attempts")]
    ConnectionFailed { attempts: u32 },

    #[error(transparent)]
    Json(#[from] serde_json::Error),

    #[error(transparent)]
    Utf8(#[from] std::string::FromUtf8Error),
}

pub type Result<T> = std::result::Result<T, Error>;
```

When you need `#[from]` but also want to attach context (like a file path), implement `From` manually:

```rust
impl Error {
    pub fn io(source: std::io::Error, path: impl Into<PathBuf>) -> Self {
        Error::Io {
            source,
            path: path.into(),
        }
    }
}

// Usage: map the error manually
fn read_config(path: &Path) -> Result<String> {
    std::fs::read_to_string(path).map_err(|e| Error::io(e, path))
}
```

## Application error handling with anyhow

Application crates (binaries) use `anyhow::Result` everywhere. Add `.context()` to give each `?` a human-readable message.

```rust
use anyhow::{bail, Context, Result};
use std::path::Path;

#[derive(Debug, serde::Deserialize)]
struct AppConfig {
    database_url: String,
    port: u16,
}

fn load_config(path: &Path) -> Result<AppConfig> {
    let contents = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {}", path.display()))?;

    let config: AppConfig = toml::from_str(&contents)
        .context("failed to parse config as TOML")?;

    if config.port == 0 {
        bail!("port must be non-zero");
    }

    Ok(config)
}

async fn connect_db(url: &str) -> Result<DbPool> {
    let pool = DbPool::connect(url)
        .await
        .with_context(|| format!("failed to connect to database at {url}"))?;

    pool.ping()
        .await
        .context("database ping failed after connect")?;

    Ok(pool)
}
```

`bail!("message")` is shorthand for `return Err(anyhow!("message"))`. Use it for early returns with context.

## Error conversion across crate boundaries

When a library crate wraps another library's errors, use `#[from]` for direct conversion or map errors at the call site.

```rust
// crate: my-storage (depends on my-protocol)
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StorageError {
    #[error("protocol error: {0}")]
    Protocol(#[from] my_protocol::Error),

    #[error("disk full: {path}")]
    DiskFull { path: String },

    #[error("corrupt data at offset {offset}")]
    Corrupt { offset: u64 },
}
```

In the application layer, `anyhow` absorbs all library errors automatically because they implement `std::error::Error`:

```rust
// Application code — anyhow wraps any Error trait implementor
use my_storage::StorageClient;

async fn backup(client: &StorageClient) -> anyhow::Result<()> {
    let data = client.read("key")
        .await
        .context("backup read failed")?;  // StorageError -> anyhow::Error

    client.write("backup/key", &data)
        .await
        .context("backup write failed")?;

    Ok(())
}
```

## Result vs Option decision tree

Use `Option<T>` when:
- Absence is a normal, expected state (not an error)
- Looking up a key that might not exist
- Parsing optional fields

Use `Result<T, E>` when:
- Failure needs explanation (what went wrong)
- The caller must handle or propagate the error
- IO, network, parsing, validation

Converting between them:

```rust
// Option -> Result: attach an error when None is unexpected
let value = map.get("key")
    .ok_or_else(|| Error::NotFound("key".into()))?;

// Result -> Option: discard the error when you only care about success
let value = parse_int(input).ok();

// Option -> Result with anyhow
let port = config.port
    .context("missing required field: port")?;
```

## Custom error with backtrace

For libraries that need rich diagnostics without exposing anyhow in the public API:

```rust
use std::backtrace::Backtrace;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Error {
    #[error("internal error: {message}")]
    Internal {
        message: String,
        backtrace: Backtrace,
    },

    #[error("validation failed: {0}")]
    Validation(String),
}

impl Error {
    pub fn internal(message: impl Into<String>) -> Self {
        Error::Internal {
            message: message.into(),
            backtrace: Backtrace::capture(),
        }
    }
}
```

Set `RUST_BACKTRACE=1` at runtime to capture backtraces. Without it, `Backtrace::capture()` returns a disabled backtrace (zero cost).

## Patterns to avoid

```rust
// Bad: String as error — no type safety, no From conversion
fn parse(s: &str) -> Result<i32, String> { ... }

// Bad: Box<dyn Error> in library public API — callers can't match variants
fn load() -> Result<Data, Box<dyn std::error::Error>> { ... }

// Bad: .unwrap() in library code — panics in caller's code
let value = map.get("key").unwrap();

// Bad: ignoring errors silently
let _ = fs::remove_file("temp.txt");

// Good: explicit ignore with reason
if let Err(e) = fs::remove_file("temp.txt") {
    tracing::debug!("cleanup failed (non-fatal): {e}");
}
```

## Error response mapping (API layer)

Map domain errors to HTTP status codes at the boundary:

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};

impl IntoResponse for Error {
    fn into_response(self) -> Response {
        let status = match &self {
            Error::NotFound(_) => StatusCode::NOT_FOUND,
            Error::Validation(_) => StatusCode::BAD_REQUEST,
            Error::Unauthorized => StatusCode::UNAUTHORIZED,
            _ => {
                tracing::error!("unhandled error: {self:?}");
                StatusCode::INTERNAL_SERVER_ERROR
            }
        };
        (status, self.to_string()).into_response()
    }
}