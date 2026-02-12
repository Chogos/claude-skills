---
name: rego
description: Rego policy development best practices for OPA. Use when writing, modifying, or reviewing .rego files, OPA policies, policy rules, or policy tests.
---

# Rego Development Best Practices

## General Philosophy
- Prioritize clear, obvious code over assumed performance optimizations.
- OPA handles optimization — focus on expressing *what* you want, not *how*.
- Always use `opa fmt` for consistent formatting.
- Use `opa check --strict` in build pipelines for extra validation.

## Modern Rego (OPA v0.59.0+)
- Use `import rego.v1` to enable modern Rego features automatically.

## Naming Conventions
- `snake_case` for all rule names and variables (e.g., `user_is_admin`).
- Leading underscore for internal/helper rules (e.g., `_is_developer(user)`).

## Style Guidelines
- Line length <= 120 characters.
- Break long comprehensions and rule bodies across multiple lines.
- Use helper rules and functions to break down complex logic.
- Extract repeated conditions into well-named helper rules.
- Group related rules together.

## Documentation
- Use metadata annotations for structured info:
  ```rego
  # METADATA
  # title: Deny non admin users
  # description: Only admin users are allowed to access these resources
  # custom:
  #   code: 401
  #   error_id: E123
  ```
- Consider JSON schemas for `input` and `data` type checking.

## Rule Design Patterns

### Handle Undefined Values
```rego
deny contains "User is anonymous" if not authenticated_user
authenticated_user if input.user_id != "anonymous"
```
Essential for "deny" rules to prevent undefined values from bypassing policy.

### Prefer Helper Rules Over Comprehensions
Use partial helper rules instead of complex comprehensions — more debuggable, reusable, testable.

### Avoid Get/List Prefixes
Use `users` not `get_users`. Exception: boolean helpers may use `is_`/`has_` prefixes.

### Unconditional Assignments
Place directly in the rule head:
```rego
full_name := concat(", ", [input.first_name, input.last_name])
```

## Variables and Data Types

### Use `in` for Membership
```rego
allow if "admin" in input.user.roles
```

### Iteration with `some .. in`
```rego
internal_hosts contains hostname if {
    some host in data.network.hosts
    host.internal == true
    hostname := host.name
}
```

### Use `every` for Universal Quantification
```rego
allow if {
    every container in input.request.object.spec.containers {
        not startswith(container.image, "old.docker.registry/")
    }
}
```

### Assignment vs Unification
- `:=` for assignment, `==` for comparison.
- Avoid `=` except for pattern matching.

### Declare Variables
Always declare with `some` or `:=` — no undeclared variables.

### Prefer Sets Over Arrays
Sets for unordered unique collections (O(1) lookups, set operations):
```rego
required_roles := {"accountant", "reports-writer"}
allow if required_roles & provided_roles == required_roles
```

## Functions
- Depend only on arguments, not `input`, `data`, or other rules.
- Use `:=` for return values, not output parameters.

## Regular Expressions
Use raw strings to avoid escaping:
```rego
regex.match(`[\d]+`, "12345")
```

## Package Organization
- Package name should match file location.
- Choose directory+filename or directory-only convention consistently.

## Imports
- Import packages, not individual rules:
  ```rego
  import data.user
  allow if user.is_admin
  ```
- Don't import from `input` — keep the data source obvious.

## Built-in Functions
Use OPA's 150+ built-ins — they're optimized for policy evaluation. Prefer over custom implementations.

## Linting
- Use [Regal](https://github.com/StyraInc/regal) linter for automated quality checks.
- `.editorconfig` for consistent formatting:
  ```
  [*.rego]
  end_of_line = lf
  insert_final_newline = true
  charset = utf-8
  indent_style = tab
  indent_size = 4
  ```

## Testing
- Comprehensive tests for all rules, especially edge cases.
- Test both positive and negative cases.
- Descriptive test names explaining the scenario.
- Test with missing/undefined values to ensure proper handling.
