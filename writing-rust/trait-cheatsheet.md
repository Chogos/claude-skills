# Rust Trait Cheatsheet

Derive-vs-manual guidance and short examples for standard traits.

## Formatting

### Display

User-facing string representation. Required for error messages via `thiserror`.

**Derive**: not derivable. Always implement manually.

```rust
use std::fmt;

struct Endpoint {
    host: String,
    port: u16,
}

impl fmt::Display for Endpoint {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}:{}", self.host, self.port)
    }
}
```

### Debug

Developer-facing representation for logging and `{:?}` formatting.

**Derive**: always. Manual only when you need to hide fields (passwords, tokens).

```rust
#[derive(Debug)]
struct Config {
    host: String,
    port: u16,
}

// Manual: redact sensitive fields
struct DbConfig {
    url: String,
    password: String,
}

impl fmt::Debug for DbConfig {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("DbConfig")
            .field("url", &self.url)
            .field("password", &"[REDACTED]")
            .finish()
    }
}
```

## Conversion

### From / Into

Infallible conversion. Implement `From<T>` — you get `Into<U>` for free.

**Derive**: not derivable. Implement `From` manually.

```rust
struct Meters(f64);
struct Kilometers(f64);

impl From<Kilometers> for Meters {
    fn from(km: Kilometers) -> Self {
        Meters(km.0 * 1000.0)
    }
}

// Callers use .into()
let m: Meters = Kilometers(5.0).into();
```

### TryFrom / TryInto

Fallible conversion. Implement `TryFrom<T>` — you get `TryInto<U>` for free.

**Derive**: not derivable. Implement `TryFrom` manually.

```rust
use std::convert::TryFrom;

struct Port(u16);

impl TryFrom<u32> for Port {
    type Error = &'static str;

    fn try_from(value: u32) -> Result<Self, Self::Error> {
        if value > 0 && value <= 65535 {
            Ok(Port(value as u16))
        } else {
            Err("port must be 1-65535")
        }
    }
}
```

### AsRef / AsMut

Cheap reference conversion. Use in function bounds to accept multiple types.

**Derive**: not derivable. Implement manually for newtype wrappers.

```rust
struct Username(String);

impl AsRef<str> for Username {
    fn as_ref(&self) -> &str {
        &self.0
    }
}

// Accepts &str, String, Username, or anything AsRef<str>
fn greet(name: impl AsRef<str>) {
    println!("hello, {}", name.as_ref());
}
```

### Deref / DerefMut

Smart pointer coercion. Use only for newtype wrappers around a single inner type.

**Derive**: not derivable. Implement manually. Do not abuse for inheritance-like behavior.

```rust
use std::ops::Deref;

struct Email(String);

impl Deref for Email {
    type Target = str;
    fn deref(&self) -> &str {
        &self.0
    }
}

// Now Email auto-coerces to &str
let email = Email("a@b.com".into());
assert!(email.contains('@'));
```

## Comparison

### PartialEq / Eq

Equality comparison. `Eq` is a marker trait indicating reflexivity (`a == a` is always true). Floating-point types implement `PartialEq` but not `Eq`.

**Derive**: prefer derive for both. Manual when you need to compare only a subset of fields.

```rust
#[derive(Debug, PartialEq, Eq)]
struct UserId(u64);

// Manual: ignore metadata in equality
struct Record {
    id: u64,
    data: String,
    cached_at: Instant, // excluded from equality
}

impl PartialEq for Record {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id && self.data == other.data
    }
}
impl Eq for Record {}
```

### PartialOrd / Ord

Ordering. `Ord` requires `Eq`. Derive `Ord` only when total ordering makes sense.

**Derive**: derive when field order matches desired sort order. Manual for custom ordering.

```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Priority(u8);

// Manual ordering: sort by score descending, then name ascending
#[derive(Debug, Eq, PartialEq)]
struct Player {
    name: String,
    score: u32,
}

impl Ord for Player {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        other.score.cmp(&self.score)
            .then_with(|| self.name.cmp(&other.name))
    }
}

impl PartialOrd for Player {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}
```

### Hash

Required when using a type as a `HashMap`/`HashSet` key. Must be consistent with `Eq`: if `a == b` then `hash(a) == hash(b)`.

**Derive**: always derive alongside `Eq`. Manual only when `PartialEq` is manual (hash the same fields).

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct CacheKey {
    namespace: String,
    id: u64,
}
```

## Iteration

### Iterator

Lazy sequence of values. Implement `next()` — all other methods (`map`, `filter`, `collect`) come free.

**Derive**: not derivable. Always implement manually.

```rust
struct Counter {
    current: u32,
    max: u32,
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.current < self.max {
            self.current += 1;
            Some(self.current)
        } else {
            None
        }
    }
}
```

### IntoIterator

Allows a type to be used in `for` loops. Implement for both owned and borrowed forms.

```rust
struct Batch<T> {
    items: Vec<T>,
}

impl<T> IntoIterator for Batch<T> {
    type Item = T;
    type IntoIter = std::vec::IntoIter<T>;

    fn into_iter(self) -> Self::IntoIter {
        self.items.into_iter()
    }
}

impl<'a, T> IntoIterator for &'a Batch<T> {
    type Item = &'a T;
    type IntoIter = std::slice::Iter<'a, T>;

    fn into_iter(self) -> Self::IntoIter {
        self.items.iter()
    }
}
```

### FromIterator

Enables `.collect()` into your type.

```rust
impl<T> FromIterator<T> for Batch<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        Batch {
            items: iter.into_iter().collect(),
        }
    }
}

// Now works: let batch: Batch<i32> = (0..10).collect();
```

## Construction

### Default

Meaningful zero/empty state. Required by many APIs (builder patterns, `Option::unwrap_or_default()`).

**Derive**: derive when every field has a `Default`. Manual when you need non-zero defaults.

```rust
// Derived: all fields use their Default (0, "", false, None)
#[derive(Debug, Default)]
struct Counters {
    requests: u64,
    errors: u64,
}

// Manual: meaningful defaults
struct ServerConfig {
    port: u16,
    max_connections: usize,
    timeout: Duration,
}

impl Default for ServerConfig {
    fn default() -> Self {
        Self {
            port: 8080,
            max_connections: 1000,
            timeout: Duration::from_secs(30),
        }
    }
}
```

### Clone

Explicit duplication. Derive when all fields are `Clone`.

**Derive**: prefer derive. Manual only for types with special cloning logic (e.g., `Arc` bumps refcount).

```rust
#[derive(Debug, Clone)]
struct Request {
    method: String,
    path: String,
    headers: HashMap<String, String>,
}
```

### Copy

Implicit bitwise copy for small, stack-allocated types. Requires `Clone`. Only for types where copying is trivially cheap.

**Derive**: derive for small value types. Never for types containing heap allocations.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}

// Cannot Copy: contains String (heap allocated)
// #[derive(Copy)]  // compile error
// struct Name(String);
```

## Serialization

### Serialize / Deserialize (serde)

Data crossing a boundary: JSON, TOML, YAML, database rows, wire protocols.

**Derive**: always derive with `serde(derive)`. Use field attributes for customization.

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct ApiResponse {
    status_code: u16,
    #[serde(skip_serializing_if = "Option::is_none")]
    error_message: Option<String>,
    #[serde(default)]
    items: Vec<Item>,
    #[serde(rename = "type")]
    kind: String,
}
```

Common serde attributes:
- `#[serde(rename_all = "camelCase")]` — field name mapping
- `#[serde(skip_serializing_if = "Option::is_none")]` — omit null fields
- `#[serde(default)]` — use `Default::default()` if missing
- `#[serde(rename = "...")]` — rename a single field
- `#[serde(deny_unknown_fields)]` — reject extra fields on deserialize
- `#[serde(tag = "type")]` — internally tagged enum representation

## Error

### Error trait

Marker for error types. Implemented automatically by `thiserror`. Manual implementation requires `Display` + `Debug`.

```rust
// Preferred: thiserror derives Error, Display, and From
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("not found: {0}")]
    NotFound(String),
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

// Manual (rare): when thiserror is not a dependency
#[derive(Debug)]
struct SimpleError(String);

impl fmt::Display for SimpleError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl std::error::Error for SimpleError {}
```