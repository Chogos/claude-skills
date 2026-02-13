# Kubernetes Policy Patterns

## Contents
- Gatekeeper ConstraintTemplate structure
- Container restrictions
- Namespace policies
- Label requirements
- Image policies

## Gatekeeper ConstraintTemplate

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        import rego.v1

        violation contains {"msg": msg} if {
            some required in input.parameters.labels
            not input.review.object.metadata.labels[required]
            msg := sprintf("missing required label: %q", [required])
        }
```

Constraint that uses the template:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    labels: ["team", "cost-center"]
```

## Container restrictions

### Disallow privileged containers

```rego
package kubernetes.admission

import rego.v1

deny contains msg if {
    some container in input_containers
    container.securityContext.privileged == true
    msg := sprintf("privileged container %q is not allowed", [container.name])
}
```

### Require non-root

```rego
deny contains msg if {
    not input.request.object.spec.securityContext.runAsNonRoot
    msg := "pod must set runAsNonRoot: true"
}

deny contains msg if {
    some container in input_containers
    container.securityContext.runAsUser == 0
    msg := sprintf("container %q runs as root (UID 0)", [container.name])
}
```

### Drop all capabilities

```rego
deny contains msg if {
    some container in input_containers
    not drops_all_capabilities(container)
    msg := sprintf("container %q must drop ALL capabilities", [container.name])
}

drops_all_capabilities(container) if {
    "ALL" in container.securityContext.capabilities.drop
}
```

### Read-only root filesystem

```rego
deny contains msg if {
    some container in input_containers
    not container.securityContext.readOnlyRootFilesystem
    msg := sprintf("container %q must use readOnlyRootFilesystem", [container.name])
}
```

## Namespace policies

### Require ResourceQuota

```rego
package kubernetes.admission

import rego.v1

# Deny namespace creation without a corresponding ResourceQuota
deny contains msg if {
    input.request.kind.kind == "Namespace"
    input.request.operation == "CREATE"
    ns := input.request.object.metadata.name
    not data.kubernetes.resourcequotas[ns]
    msg := sprintf("namespace %q must have a ResourceQuota", [ns])
}
```

### Deny default namespace usage

```rego
deny contains msg if {
    input.request.object.metadata.namespace == "default"
    input.request.kind.kind != "Service"  # allow default kubernetes service
    msg := sprintf("%s/%s must not use the default namespace", [
        input.request.kind.kind,
        input.request.object.metadata.name,
    ])
}
```

## Label requirements

### Require standard labels

```rego
package kubernetes.admission

import rego.v1

required_labels := {
    "app.kubernetes.io/name",
    "app.kubernetes.io/instance",
    "app.kubernetes.io/managed-by",
}

deny contains msg if {
    some label in required_labels
    not input.request.object.metadata.labels[label]
    msg := sprintf("missing required label %q on %s/%s", [
        label,
        input.request.kind.kind,
        input.request.object.metadata.name,
    ])
}
```

## Image policies

### Allowed registries

```rego
package kubernetes.admission

import rego.v1

allowed_registries := {
    "ghcr.io/myorg",
    "registry.example.com",
}

deny contains msg if {
    some container in input_containers
    not image_from_allowed_registry(container.image)
    msg := sprintf("container %q image %q is not from an allowed registry", [
        container.name,
        container.image,
    ])
}

image_from_allowed_registry(image) if {
    some registry in allowed_registries
    startswith(image, registry)
}
```

### Disallow latest tag

```rego
deny contains msg if {
    some container in input_containers
    not contains(container.image, ":")
    msg := sprintf("container %q image %q has no tag (defaults to :latest)", [
        container.name,
        container.image,
    ])
}

deny contains msg if {
    some container in input_containers
    endswith(container.image, ":latest")
    msg := sprintf("container %q must not use :latest tag", [container.name])
}
```
