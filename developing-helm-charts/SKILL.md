---
name: developing-helm-charts
description: Helm chart development best practices. Use when creating, modifying, or reviewing Helm charts, templates, values files, Chart.yaml, or any Kubernetes resource templates under a helm/ directory.
---

# Helm Chart Development Best Practices

## Chart Structure

- `Chart.yaml` (capitalized), all template files lowercase with dashed notation (`my-configmap.yaml`).
- Template files: `.yaml` for YAML output, `.tpl` for non-formatted content.
- Each resource definition in its own template file, named after the resource kind.
- Standard layout:
  ```
  mychart/
    Chart.yaml
    values.yaml
    templates/
      deployment.yaml
      service.yaml
      _helpers.tpl
      tests/
    charts/
    crds/
    .helmignore
  ```

## Chart.yaml

- Always `apiVersion: v2`. Semantic versioning for `version`, separate `appVersion`.
- Required: `name`, `description`, `type`, `version`.
- Include `maintainers`, `home`, `sources`, `keywords` for discoverability.
- `type`: `application` (deployable) or `library` (reusable).

## Values

- **camelCase** starting lowercase (`serverHost`, not `server-host` or `ServerHost`).
- Prefer flat over nested. Use maps over arrays for `--set` compatibility.
- Quote all strings to avoid YAML type coercion (`yes`→bool, `0777`→octal).
- Store large integers as strings, convert with `{{ int $value }}`.
- Every value must have a comment starting with the property name.
- Group related configs (`image.*`, `service.*`, `ingress.*`).
- Use `global` sparingly. Support env-specific value files.

## Templates

- `{{- }}` for whitespace control. Two spaces, never tabs.
- Whitespace after `{{` and before `}}`.
- Namespace defined templates: `{{- define "nginx.fullname" }}`.
- Always quote strings: `{{ .Values.image.tag | quote }}`.
- Use `required` for mandatory values: `{{ required "Image repo required" .Values.image.repository }}`.
- Prefer `include` over `template` (pipeable, better errors).
- Use `_helpers.tpl` for reusable snippets.

## Labels

- Use the Kubernetes recommended labels (`app.kubernetes.io/*`):
  ```yaml
  app.kubernetes.io/name: {{ include "chart.name" . }}
  app.kubernetes.io/instance: {{ .Release.Name }}
  app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
  app.kubernetes.io/component: webserver          # role within the architecture
  app.kubernetes.io/part-of: my-platform          # higher-level application
  app.kubernetes.io/managed-by: {{ .Release.Service }}
  helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
  ```
- At minimum, every chart must include `name`, `instance`, and `managed-by`.
- Use a shared `selectorLabels` helper — same labels in Deployment selector and Service selector.
- Helm hooks: always annotations, never labels.

## Security

- Default `rbac.create: true` and `serviceAccount.create: true`.
- Security contexts at both pod and container levels.
- Run as non-root, `readOnlyRootFilesystem: true`, drop all capabilities.
- Default `automountServiceAccountToken: false` unless required.
- Never hardcode secrets — use `existingSecret` pattern.

## Pods & Resources

- Fixed image tags or SHA — never `latest` or floating tags.
- Separate `image.repository` and `image.tag`. Default `pullPolicy: IfNotPresent`.
- Always set resource requests and limits.
- Configure `livenessProbe` and `readinessProbe`.

## Dependencies

- Patch-level version matching: `version: ~1.2.3`.
- Prefer `https://` repository URLs.
- Conditional: `condition: somechart.enabled`.
- Run `helm dependency update` after Chart.yaml changes.

## CRDs

- Place in `crds/` directory (static YAML, not templated).
- Helm doesn't upgrade or delete CRDs for safety.

## New chart workflow

```
- [ ] Scaffold chart structure (see patterns/chart-scaffolding.md)
- [ ] Configure Chart.yaml metadata
- [ ] Design values.yaml with sensible defaults
- [ ] Implement templates with standard labels
- [ ] Add security contexts and RBAC
- [ ] Write helm-unittest tests
- [ ] Run validation loop (below)
```

## Validation loop

1. `helm lint charts/*` — fix any warnings or errors
2. `helm template charts/* --validate` — fix rendering issues
3. `helm-unittest` — fix failing tests
4. Repeat until all three pass clean
5. Security scan with `checkov` or `kube-score`
6. `ct lint` (chart-testing) for CI-level validation

## Chart Testing

- **Local rendering**: `helm template mychart/ --values values.yaml` — verify output without a cluster.
- **Unit tests**: `helm-unittest` for assertion-based template tests. Place in `templates/tests/`:
  ```yaml
  # templates/tests/deployment_test.yaml
  suite: deployment
  tests:
    - it: should set replicas from values
      set:
        replicaCount: 3
      asserts:
        - equal:
            path: spec.replicas
            value: 3
    - it: should use the correct image
      asserts:
        - matchRegex:
            path: spec.template.spec.containers[0].image
            pattern: ^my-app:.*
  ```
- **CI validation**: `ct lint` (chart-testing) combines lint + install in a kind cluster. Use in CI pipelines.
- **Security scan**: `kube-score` or `checkov` on rendered templates.

## Hooks Lifecycle

Hooks execute at specific release lifecycle points. Annotate resources (never use labels):

| Hook | Runs | Use case |
|------|------|----------|
| `pre-install` | Before first install | DB migration, pre-flight checks |
| `post-install` | After first install | Seed data, notifications |
| `pre-upgrade` | Before upgrade | DB migration, backup |
| `post-upgrade` | After upgrade | Cache warm-up, smoke test |
| `pre-delete` | Before delete | Backup, cleanup external resources |
| `pre-rollback` | Before rollback | Snapshot current state |

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation
```

- **hook-weight**: lower runs first (string, e.g., `"-5"`, `"0"`, `"10"`).
- **hook-delete-policy**: `before-hook-creation` (delete previous hook resource before creating new), `hook-succeeded` (delete after success), `hook-failed` (delete after failure).

## Versioning & CI

- Semantic versioning. Increment chart version for any change.
- Sign charts with `helm package --sign`.
- CI: lint, template, validate, unittest, security scan.

## Deep-dive references

**Full chart template**: See [patterns/chart-scaffolding.md](patterns/chart-scaffolding.md) for a production-ready scaffold
**Values design**: See [patterns/values-design.md](patterns/values-design.md) for grouping, type coercion, overrides
**Security hardening**: See [patterns/security-hardening.md](patterns/security-hardening.md) for SecurityContext, RBAC, NetworkPolicy, secrets
**Anti-patterns**: See [anti-patterns.md](anti-patterns.md) for common mistakes with corrected versions

## Official references

- [Helm Chart Best Practices Guide](https://helm.sh/docs/chart_best_practices/) — conventions, values, templates, dependencies, labels, pods, CRDs, RBAC
- [Kubernetes Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) — full `app.kubernetes.io/*` label specification
