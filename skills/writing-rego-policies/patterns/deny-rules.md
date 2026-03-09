# Deny Rule Patterns

## Contents
- Handling undefined values
- RBAC authorization
- Resource constraints
- Network restrictions
- Multi-condition deny rules

## Handling undefined values

The most common Rego bug: an undefined intermediate value silently passes the policy.

```rego
package authz

import rego.v1

# WRONG — if input.user is missing, authenticated_user is undefined,
# so "not authenticated_user" is true, but deny never fires because
# the comprehension body also can't evaluate.

# RIGHT — split into two rules:

deny contains msg if {
    not authenticated_user
    msg := "user is not authenticated"
}

authenticated_user if {
    input.user.id != ""
}
```

For every `not X` check, ensure `X` is a rule that evaluates to `true` or is undefined — never a rule that could error.

## RBAC authorization

```rego
package rbac

import rego.v1

default allow := false

allow if {
    some role in user_roles
    some grant in data.role_permissions[role]
    grant.action == input.action
    grant.resource == input.resource
}

user_roles contains role if {
    some role in data.user_roles[input.user.id]
}

# Deny with reason for audit logging
deny contains msg if {
    not allow
    msg := sprintf("user %q is not allowed to %s %s", [
        input.user.id,
        input.action,
        input.resource,
    ])
}
```

## Resource constraints

Enforce limits on Kubernetes resources:

```rego
package kubernetes.admission

import rego.v1

deny contains msg if {
    some container in input_containers
    not container.resources.limits.memory
    msg := sprintf("container %q has no memory limit", [container.name])
}

deny contains msg if {
    some container in input_containers
    not container.resources.limits.cpu
    msg := sprintf("container %q has no CPU limit", [container.name])
}

deny contains msg if {
    some container in input_containers
    memory_limit := units.parse_bytes(container.resources.limits.memory)
    memory_limit > 4 * units.parse_bytes("1Gi")
    msg := sprintf("container %q memory limit %s exceeds 4Gi", [
        container.name,
        container.resources.limits.memory,
    ])
}

input_containers contains container if {
    some container in input.request.object.spec.containers
}

input_containers contains container if {
    some container in input.request.object.spec.initContainers
}
```

## Network restrictions

```rego
package network

import rego.v1

deny contains msg if {
    some host in input.spec.rules[_].host
    not endswith(host, ".example.com")
    msg := sprintf("ingress host %q is not under .example.com", [host])
}

deny contains msg if {
    input.spec.type == "LoadBalancer"
    not input.metadata.annotations["service.beta.kubernetes.io/aws-load-balancer-internal"]
    msg := "LoadBalancer services must be internal"
}
```

## Multi-condition deny rules

Combine conditions with helper rules for readability:

```rego
package deploy

import rego.v1

deny contains msg if {
    is_production
    not has_resource_limits
    msg := "production deployments must have resource limits"
}

deny contains msg if {
    is_production
    uses_latest_tag
    msg := "production deployments must not use :latest tag"
}

is_production if {
    input.metadata.namespace == "production"
}

is_production if {
    input.metadata.labels["env"] == "production"
}

has_resource_limits if {
    every container in input.spec.template.spec.containers {
        container.resources.limits
    }
}

uses_latest_tag if {
    some container in input.spec.template.spec.containers
    endswith(container.image, ":latest")
}
```
