# Helm Anti-Patterns

Common mistakes with corrected versions.

## Floating image tags

Bad:
```yaml
image:
  repository: nginx
  tag: latest
```

Good:
```yaml
image:
  repository: nginx
  tag: "1.25.3"
  # or pin by digest:
  # tag: "1.25.3@sha256:abc123..."
```

`latest`, `head`, `canary` — any mutable tag means deployments aren't reproducible.

**Fix for existing charts**: update `values.yaml` default to a pinned version. Add CI check that rejects `latest`/`HEAD` tags. For digest pinning, use `crane digest` to resolve and pin automatically.

## Missing resource limits

Bad:
```yaml
# no resources specified — unbounded pod can starve the node
containers:
  - name: app
    image: my-app:1.0
```

Good:
```yaml
containers:
  - name: app
    image: my-app:1.0
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi
```

**Fix for existing charts**: audit with `helm template | grep -A5 resources:`. Add default requests/limits in `values.yaml`. Use `kube-score` or `checkov` in CI to enforce resource requirements.

## Label selector drift

Bad — labels differ between Deployment selector and Service selector:
```yaml
# deployment.yaml
selector:
  matchLabels:
    app: my-app
    release: {{ .Release.Name }}

# service.yaml
selector:
  app: my-app
  # missing release label — matches ALL my-app pods across releases
```

Good — use a shared helper for selectors:
```gotemplate
{{/* in _helpers.tpl */}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

Both Deployment and Service use `{{ include "my-app.selectorLabels" . }}`.

**Fix for existing charts**: create a `selectorLabels` helper in `_helpers.tpl`. Update all Deployment selectors and Service selectors to use it. Note: changing Deployment `selector.matchLabels` requires deleting and recreating the Deployment (selectors are immutable).

## Hardcoded secrets in values

Bad:
```yaml
database:
  password: "s3cret123"
```

Good:
```yaml
database:
  existingSecret: my-app-db-credentials
  existingSecretKey: password
```

Reference in template:
```gotemplate
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ .Values.database.existingSecret }}
        key: {{ .Values.database.existingSecretKey }}
```

## Deep nesting in values

Bad:
```yaml
app:
  server:
    http:
      listener:
        port: 8080
        host: "0.0.0.0"
```

`--set app.server.http.listener.port=9090` is painful. Flatten unless grouping is meaningful.

**Fix for existing charts**: flatten values and update templates. Provide a migration note in the chart's CHANGELOG if this is a breaking change (it changes `--set` paths).

Good:
```yaml
serverPort: 8080
serverHost: "0.0.0.0"
```

## Arrays where maps work better

Bad — can't `--set` a single port:
```yaml
extraPorts:
  - name: metrics
    port: 9090
  - name: debug
    port: 9091
```

Good — each entry addressable:
```yaml
extraPorts:
  metrics:
    port: 9090
  debug:
    port: 9091
```

## Missing `required` for mandatory values

Bad — fails with a cryptic template error:
```gotemplate
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Good — fails with a clear message:
```gotemplate
image: "{{ required "image.repository is required" .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

## Using `template` instead of `include`

Bad — can't pipe output:
```gotemplate
labels:
  {{ template "my-app.labels" . }}
```

Good — pipeable, better error handling:
```gotemplate
labels:
  {{- include "my-app.labels" . | nindent 4 }}
```

## Hooks as labels

Bad:
```yaml
labels:
  helm.sh/hook: pre-install
```

Good:
```yaml
annotations:
  "helm.sh/hook": pre-install
  "helm.sh/hook-weight": "-5"
  "helm.sh/hook-delete-policy": before-hook-creation
```

Helm hooks are always annotations, never labels.

## Missing values.schema.json

Bad — no type validation, errors surface only at deploy time:
```yaml
# values.yaml without a schema — helm install accepts any type
replicaCount: "three"   # string where integer expected — template renders, K8s rejects
```

Good — type errors caught at lint/install time:
```json
// values.schema.json
{
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 }
  }
}
```

Without a schema, `--set replicaCount=abc` silently sets a string where an integer is expected. The template renders, Kubernetes rejects the manifest, and the error message points to the rendered YAML — not the values file where the mistake was made.

**Fix for existing charts**: generate a starter schema from values.yaml using `helm-schema-gen` or write manually. Add `helm lint --strict` to CI — it validates against the schema.

## Unquoted strings that YAML coerces

Bad:
```yaml
nodePort: 030    # parsed as octal 24
enabled: yes     # parsed as boolean true
version: 1.0    # parsed as float, loses trailing zero
```

Good:
```yaml
nodePort: 30
enabled: true
version: "1.0"
```
