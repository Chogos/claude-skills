# Rego Testing Patterns

## Contents
- Test file structure
- Testing deny rules
- Table-driven tests
- Testing with mock data
- Testing undefined/missing input
- Negative test cases

## Test file structure

Test files live alongside the policy they test, with `_test` suffix:

```
policies/
├── authz.rego
├── authz_test.rego
├── kubernetes/
│   ├── admission.rego
│   └── admission_test.rego
```

Run all tests:
```bash
opa test . -v
```

## Testing deny rules

Test both the positive case (rule fires) and negative case (rule doesn't fire):

```rego
package authz_test

import rego.v1
import data.authz

# Test: unauthenticated user is denied
test_deny_unauthenticated if {
    result := authz.deny with input as {}
    "user is not authenticated" in result
}

# Test: authenticated user is not denied
test_allow_authenticated if {
    result := authz.deny with input as {"user": {"id": "alice"}}
    not "user is not authenticated" in result
}
```

## Table-driven tests

For rules with multiple input variations, use a data-driven approach:

```rego
package admission_test

import rego.v1
import data.kubernetes.admission

deny_cases := [
    {
        "name": "no memory limit",
        "input": {"request": {"object": {"spec": {"containers": [
            {"name": "app", "resources": {"limits": {"cpu": "100m"}}},
        ]}}}},
        "expected": "container \"app\" has no memory limit",
    },
    {
        "name": "no cpu limit",
        "input": {"request": {"object": {"spec": {"containers": [
            {"name": "app", "resources": {"limits": {"memory": "128Mi"}}},
        ]}}}},
        "expected": "container \"app\" has no CPU limit",
    },
]

test_deny_cases if {
    every tc in deny_cases {
        result := admission.deny with input as tc.input
        tc.expected in result
    }
}
```

## Testing with mock data

Use `with` to override `data` for external data dependencies:

```rego
package rbac_test

import rego.v1
import data.rbac

mock_role_permissions := {
    "admin": [{"action": "read", "resource": "reports"}],
    "viewer": [{"action": "read", "resource": "reports"}],
}

mock_user_roles := {
    "alice": ["admin"],
    "bob": ["viewer"],
}

test_admin_can_read_reports if {
    rbac.allow
        with input as {"user": {"id": "alice"}, "action": "read", "resource": "reports"}
        with data.role_permissions as mock_role_permissions
        with data.user_roles as mock_user_roles
}

test_viewer_cannot_write_reports if {
    not rbac.allow
        with input as {"user": {"id": "bob"}, "action": "write", "resource": "reports"}
        with data.role_permissions as mock_role_permissions
        with data.user_roles as mock_user_roles
}
```

## Testing undefined/missing input

Explicitly test that policies handle missing fields:

```rego
test_deny_when_user_missing if {
    result := authz.deny with input as {}
    count(result) > 0
}

test_deny_when_user_id_empty if {
    result := authz.deny with input as {"user": {"id": ""}}
    count(result) > 0
}

test_no_panic_on_missing_nested_field if {
    # Should not error, just produce deny messages
    result := admission.deny with input as {"request": {"object": {"spec": {}}}}
    is_set(result)
}
```

## Negative test cases

Verify rules do NOT fire for valid input:

```rego
test_allow_valid_deployment if {
    result := admission.deny with input as valid_deployment
    count(result) == 0
}

valid_deployment := {"request": {"object": {"spec": {
    "securityContext": {"runAsNonRoot": true},
    "containers": [{
        "name": "app",
        "image": "ghcr.io/myorg/app:1.0.0",
        "securityContext": {
            "readOnlyRootFilesystem": true,
            "allowPrivilegeEscalation": false,
            "capabilities": {"drop": ["ALL"]},
        },
        "resources": {
            "limits": {"cpu": "500m", "memory": "256Mi"},
            "requests": {"cpu": "100m", "memory": "128Mi"},
        },
    }],
}}}}

# Test that each individual rule passes
test_no_deny_on_valid_image if {
    result := admission.deny with input as valid_deployment
    not startswith_any(result, "container")
}

# Helper to verify no message starts with a prefix
startswith_any(set, prefix) if {
    some msg in set
    startswith(msg, prefix)
}
```

## Test naming conventions

- `test_<rule>_<scenario>` — e.g., `test_deny_unauthenticated`
- `test_allow_<scenario>` for positive/negative pairs
- Group related tests with descriptive names that explain the scenario, not the implementation
