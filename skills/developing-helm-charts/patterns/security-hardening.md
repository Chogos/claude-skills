# Security Hardening Patterns

## Pod Security Standards

Align with the Kubernetes `restricted` profile by default.

### Pod-level SecurityContext

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault
```

### Container-level SecurityContext

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  capabilities:
    drop:
      - ALL
```

If the container needs specific capabilities (e.g., binding to port 80):
```yaml
capabilities:
  drop:
    - ALL
  add:
    - NET_BIND_SERVICE
```

## RBAC patterns

### Minimal Role for a workload

```yaml
{{- if .Values.rbac.create -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["{{ include "my-app.fullname" . }}-config"]
    verbs: ["update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "my-app.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ include "my-app.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
{{- end }}
```

Key points:
- Use `Role` (namespace-scoped) over `ClusterRole` when possible
- Restrict `resourceNames` to specific resources when applicable
- Never grant `*` verbs or resources

### ClusterRole for cluster-wide access

Only for controllers/operators that need cross-namespace access:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "my-app.fullname" . }}
rules:
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch"]
```

## NetworkPolicy

Default-deny with explicit allowlist:

```yaml
{{- if .Values.networkPolicy.enabled -}}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - port: http
          protocol: TCP
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: postgresql
      ports:
        - port: 5432
          protocol: TCP
    # Allow DNS
    - to: []
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
{{- end }}
```

## Secrets management

Never store secrets in values.yaml. Patterns for secret injection:

### External Secrets Operator
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "my-app.fullname" . }}
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: vault-backend
  target:
    name: {{ include "my-app.fullname" . }}
  data:
    - secretKey: database-password
      remoteRef:
        key: my-app/database
        property: password
```

### Sealed Secrets
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: {{ include "my-app.fullname" . }}
spec:
  encryptedData:
    database-password: AgBy8w...
```

### CSI Secret Store (mount from vault)
```yaml
volumeMounts:
  - name: secrets
    mountPath: /mnt/secrets
    readOnly: true
volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: {{ include "my-app.fullname" . }}
```

## ServiceAccount hardening

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "my-app.serviceAccountName" . }}
automountServiceAccountToken: false
```

Only set `automountServiceAccountToken: true` on the Pod spec when the workload needs Kubernetes API access, and pair it with a tight RBAC Role.
