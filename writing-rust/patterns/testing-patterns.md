# Testing Patterns

Production-ready testing patterns for Rust crates.

## Table-driven unit tests

Use a struct array for inputs and expected outputs. Include a `name` field for failure diagnostics.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_size_string() {
        struct Case {
            name: &'static str,
            input: &'static str,
            expected: u64,
        }

        let cases = [
            Case { name: "bytes",     input: "512",   expected: 512 },
            Case { name: "kilobytes", input: "1KB",   expected: 1_024 },
            Case { name: "megabytes", input: "5MB",   expected: 5_242_880 },
            Case { name: "gigabytes", input: "2GB",   expected: 2_147_483_648 },
        ];

        for case in cases {
            let result = parse_size(case.input).unwrap();
            assert_eq!(result, case.expected, "case: {}", case.name);
        }
    }

    #[test]
    fn parse_size_errors() {
        struct Case {
            name: &'static str,
            input: &'static str,
        }

        let cases = [
            Case { name: "empty string",    input: "" },
            Case { name: "negative",        input: "-5MB" },
            Case { name: "unknown unit",    input: "10XB" },
            Case { name: "overflow",        input: "99999999TB" },
        ];

        for case in cases {
            let result = parse_size(case.input);
            assert!(result.is_err(), "case '{}' should fail", case.name);
        }
    }
}
```

## Async test setup

Use `#[tokio::test]` for async tests. Default is current-thread runtime — use `flavor = "multi_thread"` if tests spawn tasks that need multiple threads.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tokio::net::TcpListener;

    #[tokio::test]
    async fn server_responds_ok() {
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();

        let server = tokio::spawn(start_server(listener));

        let response = reqwest::get(format!("http://{addr}/health"))
            .await
            .unwrap();
        assert_eq!(response.status(), 200);

        server.abort();
    }

    #[tokio::test]
    async fn timeout_on_slow_response() {
        let result = tokio::time::timeout(
            Duration::from_millis(100),
            slow_operation(),
        ).await;

        assert!(result.is_err(), "should have timed out");
    }
}
```

## Integration test structure

Integration tests live in `tests/` and test the crate's public API. Each file is compiled as a separate crate.

```
tests/
  common/
    mod.rs          # shared helpers (not a test file)
  api_test.rs
  storage_test.rs
```

```rust
// tests/common/mod.rs
use my_crate::Config;

pub struct TestContext {
    pub config: Config,
    pub db: TestDb,
}

impl TestContext {
    pub async fn new() -> Self {
        let config = Config {
            database_url: "postgres://localhost/test".into(),
            port: 0, // random port
        };
        let db = TestDb::setup(&config.database_url).await;
        Self { config, db }
    }

    pub async fn cleanup(self) {
        self.db.teardown().await;
    }
}
```

```rust
// tests/api_test.rs
mod common;

use common::TestContext;

#[tokio::test]
async fn create_and_fetch_user() {
    let ctx = TestContext::new().await;
    let client = my_crate::Client::new(&ctx.config);

    let user = client.create_user("alice").await.unwrap();
    assert_eq!(user.name, "alice");

    let fetched = client.get_user(user.id).await.unwrap();
    assert_eq!(fetched.name, "alice");

    ctx.cleanup().await;
}
```

## mockall for trait mocking

Define traits for external dependencies. Use `mockall` to generate mocks in tests.

```rust
// src/storage.rs
#[cfg_attr(test, mockall::automock)]
pub trait Storage: Send + Sync {
    async fn get(&self, key: &str) -> Result<Option<Vec<u8>>>;
    async fn put(&self, key: &str, value: &[u8]) -> Result<()>;
    async fn delete(&self, key: &str) -> Result<()>;
}

// src/service.rs
pub struct Service<S: Storage> {
    storage: S,
}

impl<S: Storage> Service<S> {
    pub fn new(storage: S) -> Self {
        Self { storage }
    }

    pub async fn process(&self, key: &str) -> Result<String> {
        let data = self.storage.get(key)
            .await?
            .ok_or_else(|| Error::NotFound(key.into()))?;
        Ok(String::from_utf8(data)?)
    }
}
```

```rust
// tests or #[cfg(test)]
use crate::storage::MockStorage;

#[tokio::test]
async fn process_returns_stored_value() {
    let mut mock = MockStorage::new();
    mock.expect_get()
        .with(mockall::predicate::eq("my-key"))
        .times(1)
        .returning(|_| Ok(Some(b"hello".to_vec())));

    let service = Service::new(mock);
    let result = service.process("my-key").await.unwrap();
    assert_eq!(result, "hello");
}

#[tokio::test]
async fn process_returns_not_found() {
    let mut mock = MockStorage::new();
    mock.expect_get()
        .returning(|_| Ok(None));

    let service = Service::new(mock);
    let result = service.process("missing").await;
    assert!(matches!(result, Err(Error::NotFound(_))));
}
```

## Property-based testing with proptest

Generate random inputs to find edge cases that hand-written tests miss.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn roundtrip_encode_decode(input in "\\PC{0,256}") {
            let encoded = encode(&input);
            let decoded = decode(&encoded).unwrap();
            assert_eq!(decoded, input);
        }

        #[test]
        fn parse_never_panics(input in "\\PC{0,100}") {
            // Just verify it doesn't panic — error is fine
            let _ = parse_duration(&input);
        }

        #[test]
        fn sort_preserves_length(mut items in prop::collection::vec(any::<i32>(), 0..100)) {
            let original_len = items.len();
            items.sort();
            assert_eq!(items.len(), original_len);
        }
    }

    // Targeted strategy for domain-specific types
    fn valid_port() -> impl Strategy<Value = u16> {
        1024..=65535u16
    }

    fn valid_host() -> impl Strategy<Value = String> {
        "[a-z]{1,10}(\\.[a-z]{1,5}){1,3}"
    }

    proptest! {
        #[test]
        fn parse_endpoint_roundtrip(host in valid_host(), port in valid_port()) {
            let input = format!("{host}:{port}");
            let endpoint = Endpoint::parse(&input).unwrap();
            assert_eq!(endpoint.host, host);
            assert_eq!(endpoint.port, port);
        }
    }
}
```

## Test helper module organization

Shared test utilities in a module gated behind `#[cfg(test)]` (for unit tests) or in `tests/common/` (for integration tests).

```rust
// src/lib.rs
#[cfg(test)]
pub(crate) mod test_helpers {
    use std::sync::LazyLock;

    pub static SAMPLE_CONFIG: LazyLock<crate::Config> = LazyLock::new(|| {
        crate::Config {
            host: "localhost".into(),
            port: 8080,
            timeout: std::time::Duration::from_secs(5),
        }
    });

    pub fn sample_request(method: &str, path: &str) -> crate::Request {
        crate::Request {
            method: method.into(),
            path: path.into(),
            headers: Default::default(),
            body: None,
        }
    }

    /// Asserts that two JSON values are structurally equal, ignoring key order.
    pub fn assert_json_eq(left: &serde_json::Value, right: &serde_json::Value) {
        assert_eq!(
            left, right,
            "JSON mismatch:\n  left:  {}\n  right: {}",
            serde_json::to_string_pretty(left).unwrap(),
            serde_json::to_string_pretty(right).unwrap(),
        );
    }
}
```

Using `LazyLock` (stable since Rust 1.80) for expensive fixtures avoids re-initialization across tests. Each test gets a shared reference to the same data.

## Snapshot testing with insta

For complex output (JSON, formatted strings, error messages), snapshot tests catch unintended changes:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use insta::assert_snapshot;

    #[test]
    fn format_report() {
        let report = generate_report(&sample_data());
        assert_snapshot!(report);
    }

    #[test]
    fn serialize_config() {
        let config = Config::default();
        insta::assert_json_snapshot!(config);
    }
}
```

Run `cargo insta review` to accept or reject snapshot changes.

## Assertion patterns

```rust
// Exact match with context
assert_eq!(result, 42, "expected 42 for input '{input}'");

// Pattern matching on enum variants
assert!(matches!(result, Ok(Value::Number(_))));
assert!(matches!(result, Err(Error::NotFound(ref s)) if s.contains("user")));

// Floating point comparison
assert!((result - expected).abs() < f64::EPSILON, "result {result} != {expected}");

// Collection checks
assert!(items.contains(&target), "missing {target} in {items:?}");
assert!(items.is_empty(), "expected empty, got {items:?}");

// Error message content
let err = operation().unwrap_err();
assert!(err.to_string().contains("timeout"), "unexpected error: {err}");
```