# Serde Patterns

Common serde configurations for API types:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
#[serde(deny_unknown_fields)]
struct CreateUserRequest {
    display_name: String,
    #[serde(default)]
    tags: Vec<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    avatar_url: Option<String>,
}

// Internally tagged enum — {"type": "email", "address": "..."}
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
enum Notification {
    Email { address: String },
    Sms { phone: String },
    Push { device_id: String },
}
```

- `rename_all = "camelCase"` for JSON APIs, `rename_all = "snake_case"` for config files.
- `deny_unknown_fields` on request types to catch typos. Omit on response types for forward compatibility.
- `tag = "type"` for internally tagged enums (most common in APIs). `untagged` for transparent parsing of multiple formats.
- `flatten` to inline nested struct fields into the parent.
