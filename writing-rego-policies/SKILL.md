---
name: writing-rego-policies
description: Rego policy development best practices for OPA. Use when writing, modifying, or reviewing .rego files, OPA policies, policy rules, or policy tests.
---

# Rego Development Best Practices

## Core Conventions

- Always use `import rego.v1` (OPA v0.59.0+) for modern Rego features. For OPA < v0.59.0, use `import future.keywords` instead.
- Always use `opa fmt` for consistent formatting.
- Use `opa check --strict` in build pipelines.
- Prioritize clear code over assumed performance optimizations — OPA handles optimization.

## Naming & Style

- `snake_case` for all rule names and variables (`user_is_admin`).
- Leading underscore for internal helpers (`_is_developer(user)`).
- Use descriptive rule names: `users` not `get_users`. Exception: `is_`/`has_` for booleans.
- Break down complex logic into well-named helper rules.

## Rule Design

### Handle undefined values

The most common Rego bug — undefined intermediates silently bypass policy:

```rego
deny contains "User is anonymous" if not authenticated_user
authenticated_user if input.user_id != "anonymous"
```

For every `not X` check, ensure `X` evaluates to `true` or is undefined — never errors.

### Prefer helpers over comprehensions

Partial helper rules are more debuggable, reusable, and testable than inline comprehensions.

### Unconditional assignments in the rule head

```rego
full_name := concat(", ", [input.first_name, input.last_name])
```

## Variables & Data Types

### Membership and iteration

```rego
allow if "admin" in input.user.roles

internal_hosts contains hostname if {
    some host in data.network.hosts
    host.internal == true
    hostname := host.name
}
```

### Universal quantification

```rego
allow if {
    every container in input.request.object.spec.containers {
        not startswith(container.image, "old.docker.registry/")
    }
}
```

### Assignment vs comparison

- `:=` for assignment, `==` for comparison. Avoid `=` except for pattern matching.
- Always declare variables with `some` or `:=`.

### Sets over arrays

Sets for unordered unique collections (O(1) lookups, set operations):
```rego
required_roles := {"accountant", "reports-writer"}
allow if required_roles & provided_roles == required_roles
```

## Functions

- Depend only on arguments, not `input`, `data`, or other rules.
- Use `:=` for return values.

## Documentation

Use metadata annotations:
```rego
# METADATA
# title: Deny non admin users
# description: Only admin users are allowed to access these resources
# custom:
#   code: 401
#   error_id: E123
```

## Packages & Imports

- Package name matches file location.
- Import packages, not individual rules:
  ```rego
  import data.user
  allow if user.is_admin
  ```
- Don't import from `input` — keep the data source obvious.

## Debugging

### print() for trace output

```rego
allow if {
    print("user:", input.user, "roles:", input.user.roles)
    "admin" in input.user.roles
}
```

`print()` writes to stderr during `opa eval` and `opa test -v`. Remove before production.

### opa eval with explain

```bash
# Show full evaluation trace
opa eval --data policy.rego --input input.json "data.authz.allow" --explain=full

# Format output for readability
opa eval --data policy.rego --input input.json "data.authz.allow" --format=pretty
```

### Common reasons for unexpected undefined

1. **Missing input field** — `input.user.role` is undefined when `input.user` doesn't exist. Guard with `input.user` check first or use default values.
2. **Typo in field name** — Rego doesn't error on missing fields, just returns undefined. Use `opa check --strict` to catch unused variables.
3. **Type mismatch** — comparing string `"80"` to number `80` silently fails. Use `to_number()` or ensure consistent types.
4. **Negation on undefined** — `not x` is true when `x` is undefined AND when `x` is false. Be explicit about what you're negating.

## New policy workflow

```
- [ ] Design policy rules and identify input schema
- [ ] Write deny/allow rules with undefined value handling
- [ ] Extract complex conditions into helper rules
- [ ] Write tests (positive, negative, missing input)
- [ ] Run validation loop (below)
```

## Linting with Regal

[Regal](https://github.com/StyraInc/regal) checks 7 rule categories:
- **bugs** — common mistakes and inefficiencies
- **idiomatic** — non-idiomatic Rego constructs
- **imports** — import statement issues
- **performance** — suboptimal patterns
- **style** — style guide violations
- **testing** — test quality issues
- **custom** — organization-specific rules

Configure per-project in `.regal/config.yaml`. Use `regal lint --format json` for CI.

## Validation loop

1. `opa fmt --write` — auto-format
2. `opa check --strict .` — fix any type errors
3. `regal lint .` — fix linter warnings (see categories above)
4. `opa test . -v` — fix failing tests
5. Repeat until all four pass clean

## Deep-dive references

**Deny rule patterns**: See [patterns/deny-rules.md](patterns/deny-rules.md) for RBAC, resource constraints, network rules
**Kubernetes policies**: See [patterns/kubernetes-policies.md](patterns/kubernetes-policies.md) for Gatekeeper, admission control
**Testing**: See [patterns/testing-patterns.md](patterns/testing-patterns.md) for table-driven tests, mocks, edge cases
**Built-ins**: See [builtins-cheatsheet.md](builtins-cheatsheet.md) for grouped OPA built-in functions with examples

## Official references

- [OPA Rego Style Guide](https://www.openpolicyagent.org/docs/latest/style-guide/) — naming, rules, variables, functions, imports
- [OPA Built-in Functions](https://www.openpolicyagent.org/docs/latest/policy-reference/#built-in-functions) — full 150+ function reference
- [Regal Linter](https://docs.styra.com/regal) — rule categories, configuration, editor integration
