# Recommended Crates

| Category | Crate | Notes |
|----------|-------|-------|
| HTTP server | `axum` | Tower-based, async, first-party tokio integration |
| HTTP client | `reqwest` | async/sync, TLS, JSON, connection pooling |
| Serialization | `serde` + `serde_json` | Derive-based, zero-copy deserialization |
| CLI parsing | `clap` (derive) | Type-safe args, subcommands, shell completions |
| Database | `sqlx` | Compile-time checked queries, async, multi-DB |
| Observability | `tracing` + `tracing-subscriber` | Structured spans, async-aware |
| Error (library) | `thiserror` | Derive Error + Display for enum errors |
| Error (app) | `anyhow` | Ergonomic error handling with context |
| Testing | `proptest`, `insta` | Property-based testing, snapshot testing |
| Async runtime | `tokio` | Multi-threaded, full-featured async runtime |
