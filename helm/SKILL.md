---
name: helm
description: Helm chart development best practices. Use when creating, modifying, or reviewing Helm charts, templates, values files, Chart.yaml, or any Kubernetes resource templates under a helm/ directory.
---

# Helm Chart Development Best Practices

## Chart Structure & Naming
- Chart names: lowercase letters, numbers, and dashes only (e.g., `nginx-ingress`).
- No uppercase, underscores, or dots in chart names.
- `Chart.yaml` (capitalized), all template files lowercase.
- Template files: `.yaml` for YAML output, `.tpl` for non-formatted content.
- Template file names: dashed notation (`my-example-configmap.yaml`), not camelCase.
- Each resource definition in its own template file, named after the resource kind.
- Standard directory structure:
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

## Chart.yaml Metadata
- Always `apiVersion: v2` for Helm 3.
- Semantic versioning for `version`, separate `appVersion` for the application.
- Required: `name`, `description`, `type`, `version`.
- Include `maintainers`, `home`, `sources`, `keywords` for discoverability.
- `dependencies` with version constraints (e.g., `~> 1.2.0`).
- Chart `type`: `application` (deployable) or `library` (reusable).

## Values & Configuration
- **camelCase** for variable names starting lowercase (e.g., `chickenNoodleSoup`).
- Never use hyphens or initial caps in value names.
- Prefer flat over nested values when possible.
- Use maps over arrays for `--set` compatibility.
- Quote all strings to avoid YAML type coercion.
- Store large integers as strings, use `{{ int $value }}` in templates.
- Every value must have a comment starting with the property name:
  ```yaml
  # serverHost is the host name for the webserver
  serverHost: example
  ```
- Group related configs (e.g., `image.*`, `service.*`, `ingress.*`).
- Use `global` sparingly. Support env-specific value files (`values-dev.yaml`, `values-prod.yaml`).

## Templates & Functions
- Use `{{- }}` for whitespace control.
- Two spaces for indentation, never tabs.
- Whitespace after `{{` and before `}}`: `{{ .foo }}` not `{{.foo}}`.
- Namespace defined templates: `{{- define "nginx.fullname" }}` not `{{- define "fullname" }}`.
- Always quote string values: `{{ .Values.image.tag | quote }}`.
- Use `required` for mandatory values: `{{ required "Image repository required" .Values.image.repository }}`.
- Prefer `include` over `template` for better error handling.
- Use `_helpers.tpl` for reusable snippets.
- Template comments: `{{- /* comment */ -}}`. YAML comments: `#` for user-visible info.

## Labels & Annotations
- Labels for K8s identification/querying, annotations for non-queryable metadata.
- Required labels:
  ```yaml
  app.kubernetes.io/name: {{ include "chart.name" . }}
  app.kubernetes.io/instance: {{ .Release.Name }}
  helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
  app.kubernetes.io/managed-by: {{ .Release.Service }}
  ```
- Helm hooks: always annotations, never labels.
- Keep label selectors consistent across all resources.

## Security & RBAC
- Default `rbac.create: true` and `serviceAccount.create: true`.
- Use standardized ServiceAccount helper template.
- Security contexts at both pod and container levels.
- Run as non-root when possible. Use `readOnlyRootFilesystem: true` where applicable.
- Default `automountServiceAccountToken: false` unless required.
- Never hardcode secrets.

## Pod & Container Configuration
- Fixed image tags or SHA — never `latest`, `head`, `canary`, or floating tags.
- Separate `image.repository` and `image.tag` in values.
- Default `ImagePullPolicy: IfNotPresent`.
- Always declare explicit pod selectors using stable labels.

## Resource Management
- Always set resource requests and limits.
- Configure `livenessProbe` and `readinessProbe`.
- Use `nodeSelector`, `affinity`, `tolerations` for scheduling.

## Dependencies & Subcharts
- Patch-level version matching: `version: ~1.2.3`.
- Prefer `https://` repository URLs.
- Conditional dependencies: `condition: somechart.enabled`.
- Run `helm dependency update` after Chart.yaml changes.
- Use `alias` for multiple instances of the same dependency.

## CRDs
- Place in `crds/` directory (not templated, installed as static YAML).
- Helm doesn't support CRD upgrades or deletions for safety.
- Consider separate chart for CRDs vs resources.

## Testing & Validation
- Include `templates/tests/` for `helm test`.
- `helm lint` for static validation.
- `helm template --debug` for rendering issues.
- `helm-unittest` for unit testing templates.

## Versioning
- Semantic versioning. Increment chart version for any change.
- Separate `appVersion` for the application version.
- Sign charts with `helm package --sign`.

## CI/CD
- `helm lint charts/*` and `helm template charts/* --validate` in CI.
- Security scanning with `checkov` or `kube-score`.
- Automated testing with `chart-testing` (ct).
