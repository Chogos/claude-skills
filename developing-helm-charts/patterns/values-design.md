# Values Design Patterns

## Grouping strategy

Group related configuration under a common key. Standard groups:

```yaml
image:          # container image settings
service:        # service type, ports
ingress:        # ingress rules, TLS
resources:      # requests/limits
autoscaling:    # HPA config
serviceAccount: # SA creation, name
podSecurityContext: # pod-level security
securityContext:    # container-level security
```

Custom application config goes under a dedicated key:

```yaml
app:
  logLevel: info
  featureFlags:
    newDashboard: true
```

## Maps over arrays for `--set` compatibility

Arrays can't be partially overridden with `--set`. Use maps when users need per-item overrides.

Bad — array (can't override a single port):
```yaml
ports:
  - name: http
    port: 8080
  - name: metrics
    port: 9090
```

Good — map (each port addressable via `--set`):
```yaml
ports:
  http:
    port: 8080
  metrics:
    port: 9090
```

## Type coercion gotchas

YAML auto-converts unquoted values. Always quote strings to prevent surprises.

| Value | YAML interprets as | Fix |
|-------|-------------------|-----|
| `yes` | `true` (boolean) | `"yes"` |
| `no` | `false` (boolean) | `"no"` |
| `on` | `true` (boolean) | `"on"` |
| `1.0` | `1` (float) | `"1.0"` |
| `0777` | `511` (octal → int) | `"0777"` |
| `1e3` | `1000` (float) | `"1e3"` |

In templates, always use `| quote` for string values:
```gotemplate
annotations:
  version: {{ .Values.appVersion | quote }}
```

For large integers, store as strings and convert:
```gotemplate
replicas: {{ int .Values.replicaCount }}
```

## Environment-specific overrides

Use layered value files:

```
values.yaml          # defaults (committed)
values-dev.yaml      # dev overrides
values-staging.yaml  # staging overrides
values-prod.yaml     # prod overrides
```

Install with:
```bash
helm install my-app ./chart -f values.yaml -f values-prod.yaml
```

Later files override earlier ones. Keep `values.yaml` as the complete reference with all keys documented.

## Flat vs nested

Prefer flat when the value stands alone. Nest when values are logically grouped and used together.

Flat — independent settings:
```yaml
logLevel: info
metricsEnabled: true
```

Nested — tightly coupled settings:
```yaml
database:
  host: localhost
  port: 5432
  name: mydb
```

Avoid nesting beyond 3 levels — it makes `--set` flags verbose and error-prone:
```bash
# too deep
--set app.database.connection.pool.maxSize=10
```

## Conditional features

Use `enabled` booleans to gate optional resources:

```yaml
ingress:
  enabled: false

monitoring:
  enabled: false
  serviceMonitor:
    interval: 30s
```

In templates:
```gotemplate
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

## Required values with defaults

Use `required` for values that must be explicitly set (no safe default):

```gotemplate
image: {{ required "image.repository is required" .Values.image.repository }}
```

Use `default` for values with safe fallbacks:
```gotemplate
replicas: {{ .Values.replicaCount | default 1 }}
```
