---
name: writing-rust
description: Rust development best practices. Use when writing, modifying, or reviewing .rs files, Cargo.toml, Cargo.lock, Rust crates, workspaces, or build.rs scripts.
---

# Rust Development Best Practices

## Project Structure

Single crate layout:

```
my-crate/
  Cargo.toml
  src/
    lib.rs          # library root
    main.rs         # binary entry point (calls into lib.rs)
    config.rs
    error.rs
  tests/            # integration tests
    api_test.rs
  benches/
    throughput.rs
  examples/
    basic_usage.rs
```

Library + thin binary pattern — keep logic in `lib.rs`, binary is a thin wrapper:

```rust
// src/main.rs
fn main() -> anyhow::Result<()> {
    let config = my_crate::Config::from_env()?;
    my_crate::run(config)
}
```

Workspace layout for multi-crate projects:

```toml
# Cargo.toml (workspace root)
[workspace]
resolver = "2"
members = ["crates/*"]

[workspace.package]
edition = "2021"
rust-version = "1.75"
license = "MIT"

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
thiserror = "2"
anyhow = "1"
tracing = "0.1"
```

Each member crate inherits shared deps:

```toml
# crates/my-service/Cargo.toml
[package]
name = "my-service"
edition.workspace = true
rust-version.workspace = true

[dependencies]
tokio.workspace = true
serde.workspace = true
```

## Ownership & Borrowing

Accept borrowed types in function signatures — let callers decide allocation:

- `&str` over `String`
- `&[T]` over `Vec<T>`
- `&Path` over `PathBuf`
- `impl AsRef<str>` when accepting both `&str` and `String`

```rust
// Good: accepts &str, String, Cow<str>
fn process(name: impl AsRef<str>) {
    let name = name.as_ref();
    // ...
}

// Good: borrows slice, works with Vec<T>, arrays, slices
fn sum(values: &[i32]) -> i32 {
    values.iter().sum()
}
```

Use `Cow<'_, str>` when a function sometimes allocates and sometimes borrows:

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains(' ') {
        Cow::Owned(input.replace(' ', "_"))
    } else {
        Cow::Borrowed(input)
    }
}
```

Every `.clone()` must have a reason. If you clone to satisfy the borrow checker, restructure the code first.

## Error Handling

**Library crates** — define a typed error enum with `thiserror`:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Error {
    #[error("io error: {0}")]
    Io(#[from] std::io::Error),

    #[error("parse error at line {line}: {message}")]
    Parse { line: usize, message: String },

    #[error("not found: {0}")]
    NotFound(String),
}

pub type Result<T> = std::result::Result<T, Error>;
```

Never use `String` as an error type. Never use `Box<dyn std::error::Error>` in library public APIs.

**Application crates** — use `anyhow::Result` with `.context()`:

```rust
use anyhow::{Context, Result};

fn load_config(path: &Path) -> Result<Config> {
    let contents = std::fs::read_to_string(path)
        .context("failed to read config file")?;
    let config: Config = toml::from_str(&contents)
        .context("failed to parse config")?;
    Ok(config)
}
```

**Rules**:
- No `.unwrap()` in library code.
- `expect("reason")` only when provably safe (e.g., after a check or on a known-valid constant).
- Propagate with `?`. Add `.context()` when the surrounding context would otherwise be lost.

See [patterns/error-handling-patterns.md](patterns/error-handling-patterns.md) for conversion chains, backtrace, and decision trees.

## Naming Conventions

| Kind | Convention | Example |
|------|-----------|---------|
| Types, traits, enums | PascalCase | `HttpClient`, `ParseError` |
| Functions, methods, variables | snake_case | `read_file`, `total_count` |
| Constants, statics | SCREAMING_SNAKE | `MAX_RETRIES`, `DEFAULT_PORT` |
| Modules, crates | snake_case | `my_crate`, `config` |

**Constructors**:
- `new()` — default constructor, takes required fields
- `with_capacity()`, `with_timeout()` — constructor variants

**Conversions**:
- `from_bytes()`, `from_str()` — fallible or infallible construction from another type
- `to_string()`, `to_vec()` — potentially expensive conversion, returns owned
- `into_inner()`, `into_bytes()` — consumes self, returns owned
- `as_str()`, `as_bytes()` — cheap reference conversion, returns borrowed

**No `get_` prefix** for getters. Use the field name directly:

```rust
impl Config {
    pub fn port(&self) -> u16 { self.port }        // not get_port()
    pub fn is_debug(&self) -> bool { self.debug }   // booleans: is_/has_/can_
}
```

**Builder pattern** — `set_` or bare name with `mut self`:

```rust
impl ServerBuilder {
    pub fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }
    pub fn build(self) -> Result<Server, Error> { /* ... */ }
}
```

## Traits

Derive `Debug` on everything. Derive `Clone`, `PartialEq` when the type semantics support it.

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Endpoint {
    pub host: String,
    pub port: u16,
}
```

**Standard trait usage**:

| Trait | When |
|-------|------|
| `Display` | User-facing output, error messages |
| `Debug` | Always — developer output, logging |
| `From<T>` / `TryFrom<T>` | Infallible / fallible type conversions |
| `Default` | Type has a meaningful zero/empty state |
| `Clone` | Value can be duplicated |
| `PartialEq` / `Eq` | Type supports equality comparison |
| `Hash` | Type used as HashMap key (requires `Eq`) |
| `Serialize` / `Deserialize` | Data crosses a boundary (network, disk, config) |

Prefer trait bounds over concrete types in public APIs:

```rust
// Good: accepts any iterator of strings
pub fn join_names(names: impl IntoIterator<Item = impl AsRef<str>>) -> String {
    names.into_iter()
        .map(|n| n.as_ref().to_owned())
        .collect::<Vec<_>>()
        .join(", ")
}
```

Implement `From<T>` for infallible conversions — callers use `.into()`:

```rust
impl From<Config> for Settings {
    fn from(config: Config) -> Self {
        Settings {
            timeout: config.timeout_ms,
            retries: config.max_retries,
        }
    }
}
```

See [trait-cheatsheet.md](trait-cheatsheet.md) for derive-vs-manual guidance per trait.

## Async (tokio)

Use `#[tokio::main]` for the entry point. Configure flavor and worker threads only when needed:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::init();
    let config = Config::from_env()?;
    run(config).await
}
```

**Rules**:
- `tokio::spawn` for independent concurrent tasks.
- `tokio::select!` for racing multiple futures (first one wins).
- Never block the runtime — use `tokio::task::spawn_blocking` for CPU-bound or synchronous IO work.
- `tokio::sync::Mutex` when holding the lock across `.await` points. `std::sync::Mutex` when the critical section is synchronous and short.
- Channels: `mpsc` for multi-producer work queues, `oneshot` for single request-response, `broadcast` for fan-out.

**Graceful shutdown**:

```rust
use tokio::signal;
use tokio_util::sync::CancellationToken;

async fn run(config: Config) -> anyhow::Result<()> {
    let token = CancellationToken::new();
    let shutdown_token = token.clone();

    tokio::spawn(async move {
        signal::ctrl_c().await.expect("failed to listen for ctrl-c");
        shutdown_token.cancel();
    });

    let server = start_server(config, token.clone());
    let worker = run_worker(token.clone());

    tokio::select! {
        res = server => res?,
        res = worker => res?,
        _ = token.cancelled() => {
            tracing::info!("shutting down gracefully");
        }
    }

    Ok(())
}
```

See [patterns/async-patterns.md](patterns/async-patterns.md) for channels, connection pooling, and select! patterns.

## Testing

**Unit tests** — colocated in the same file:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_valid_input() {
        let result = parse("42").unwrap();
        assert_eq!(result, 42, "should parse integer string");
    }
}
```

**Integration tests** — in `tests/` directory, test the public API only.

**Async tests** — use `#[tokio::test]`:

```rust
#[tokio::test]
async fn fetch_returns_ok() {
    let client = Client::new();
    let response = client.fetch("https://example.com").await.unwrap();
    assert_eq!(response.status(), 200);
}
```

**Table-driven tests**:

```rust
#[test]
fn parse_duration_variants() {
    let cases = [
        ("10s", Duration::from_secs(10)),
        ("5m", Duration::from_secs(300)),
        ("1h", Duration::from_secs(3600)),
    ];
    for (input, expected) in cases {
        let result = parse_duration(input).unwrap();
        assert_eq!(result, expected, "input: {input}");
    }
}
```

**Assertions**:
- `assert_eq!(left, right, "context message with {variable}")` — always include a message.
- `assert!(matches!(result, Err(Error::NotFound(_))))` — match enum variants.
- `LazyLock` for expensive test fixtures shared across tests.

See [patterns/testing-patterns.md](patterns/testing-patterns.md) for mockall, proptest, and helper module patterns.

## Unsafe Code

- Avoid unless there is no safe alternative.
- Minimize the unsafe block scope to the smallest possible expression.
- Document every unsafe block with `// SAFETY: reason` explaining the invariant:

```rust
// SAFETY: pointer is guaranteed non-null by the C API contract,
// and the buffer has been initialized by init_buffer() above.
let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
```

- Wrap unsafe code in a safe public API. Callers must never need `unsafe` themselves.
- Test every unsafe block — undefined behavior hides until production.

## Performance

1. **Profile first** — don't guess. Use `cargo bench` with criterion:
   ```toml
   [dev-dependencies]
   criterion = { version = "0.5", features = ["html_reports"] }

   [[bench]]
   name = "throughput"
   harness = false
   ```

2. **Iterators over manual loops** — the compiler optimizes iterator chains aggressively.
3. **Avoid allocations** in hot paths — reuse buffers, use `&str` over `String`, `SmallVec` for typically-small collections.
4. **`#[inline]`** only for small functions called across crate boundaries. Never on large functions — let the compiler decide.
5. **`String::with_capacity(n)`** when the final size is known or estimable.

## Cargo Conventions

**Cargo.toml ordering**: `[package]`, `[features]`, `[dependencies]`, `[dev-dependencies]`, `[build-dependencies]`, `[[bin]]`, `[[bench]]`, `[profile.*]`.

**Dependency pinning**:
- Libraries: use semver ranges (`"1"`, `"0.5"`). Let downstream resolve.
- Applications: use `Cargo.lock` (commit it) with semver ranges. Use `=` pinning only when a specific version is required for compatibility.

**Features**:
- No default features unless genuinely needed by most users.
- Additive only — enabling a feature must never remove functionality.
- Gate optional heavy dependencies behind features.

**MSRV**: set `rust-version` in `[package]`. Test against it in CI.

**Release profile**:

```toml
[profile.release]
lto = "thin"
codegen-units = 1
strip = true
```

Use `lto = "fat"` for maximum binary size reduction when build time is not a concern.

## New crate workflow

```
- [ ] cargo init --name my-crate (or --lib for library)
- [ ] Set edition, rust-version, license, description in Cargo.toml
- [ ] Create src/error.rs with thiserror enum (library) or add anyhow (application)
- [ ] Create src/lib.rs with public API surface
- [ ] Add #[derive(Debug)] on all types
- [ ] Add integration test in tests/
- [ ] Configure CI (fmt, clippy, test, doc)
- [ ] Run validation loop (below)
```

## Validation loop

1. `cargo fmt --check` — fix formatting violations
2. `cargo clippy -- -D warnings` — fix all clippy lints (treat as errors)
3. `cargo test` — fix failing tests
4. `cargo doc --no-deps` — fix documentation warnings
5. `cargo audit` — check for known vulnerabilities in dependencies
6. Repeat until all five pass clean

## Deep-dive references

**Error handling**: See [patterns/error-handling-patterns.md](patterns/error-handling-patterns.md) for thiserror/anyhow patterns, conversion chains, backtrace
**Async**: See [patterns/async-patterns.md](patterns/async-patterns.md) for tokio runtime, channels, select!, shutdown, connection pooling
**Testing**: See [patterns/testing-patterns.md](patterns/testing-patterns.md) for table-driven, mockall, proptest, test helpers
**Traits**: See [trait-cheatsheet.md](trait-cheatsheet.md) for derive-vs-manual guidance per standard trait

## Official references

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — naming, interoperability, documentation, type safety
- [The Rust Reference](https://doc.rust-lang.org/reference/) — language specification, syntax, semantics
- [Clippy Lints](https://rust-lang.github.io/rust-clippy/master/index.html) — full lint list with explanations and examples