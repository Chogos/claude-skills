# GitOps Integration Patterns

ArgoCD and Flux patterns for deploying Helm charts via GitOps.

## ArgoCD Application

### Basic Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/charts
    chart: my-app
    targetRevision: 1.2.0
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true        # delete resources removed from chart
      selfHeal: true     # revert manual changes to match Git
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m0s
```

### ApplicationSet for Multiple Environments

Deploy the same chart across dev/staging/prod from a single template:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            cluster: https://dev-cluster.example.com
            revision: main
          - env: staging
            cluster: https://staging-cluster.example.com
            revision: v1.2.0
          - env: prod
            cluster: https://prod-cluster.example.com
            revision: v1.2.0
  template:
    metadata:
      name: "my-app-{{ env }}"
    spec:
      project: default
      source:
        repoURL: https://github.com/org/gitops
        path: "apps/my-app/{{ env }}"
        targetRevision: "{{ revision }}"
      destination:
        server: "{{ cluster }}"
        namespace: my-app
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## Flux HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: my-app
spec:
  interval: 5m
  chart:
    spec:
      chart: my-app
      version: "~1.2.0"
      sourceRef:
        kind: HelmRepository
        name: org-charts
      interval: 1m
  values:
    replicaCount: 3
    image:
      repository: ghcr.io/org/my-app
      tag: "1.2.0"
  valuesFrom:
    - kind: ConfigMap
      name: my-app-env-values
      valuesKey: values.yaml
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
  rollback:
    cleanupOnFail: true
    recreate: true
```

### Flux HelmRepository (OCI)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: org-charts
  namespace: flux-system
spec:
  type: oci
  interval: 5m
  url: oci://ghcr.io/org/charts
```

## Multi-Environment Values Structure

```
gitops-repo/
  apps/
    my-app/
      base/
        kustomization.yaml
        helmrelease.yaml       # or ArgoCD Application
      dev/
        kustomization.yaml     # patches base with dev values
        values.yaml
      staging/
        kustomization.yaml
        values.yaml
      prod/
        kustomization.yaml
        values.yaml
```

Each environment overrides only what differs from the base. Common pattern: base defines the chart source and defaults, overlays set replicas, resources, image tags, and environment-specific config.

### Sealed Secrets Per Environment

```yaml
# prod/sealed-secret.yaml — encrypted, safe to commit
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-app-secrets
  namespace: my-app
spec:
  encryptedData:
    DB_PASSWORD: AgBy8h...  # encrypted per-cluster, only that cluster can decrypt
```

Seal per cluster — each environment has its own sealing key. Never share sealed secrets across clusters.

## Automated Image Tag Updates

### ArgoCD Image Updater

```yaml
# Add annotations to the Application
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: app=ghcr.io/org/my-app
    argocd-image-updater.argoproj.io/app.update-strategy: semver
    argocd-image-updater.argoproj.io/app.allow-tags: "regexp:^[0-9]+\\.[0-9]+\\.[0-9]+$"
    argocd-image-updater.argoproj.io/write-back-method: git
```

Image Updater watches the registry, detects new tags matching the semver pattern, and commits the updated tag to the GitOps repo.

### Flux Image Automation

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: my-app
spec:
  imageRepositoryRef:
    name: my-app
  policy:
    semver:
      range: ">=1.0.0"

---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: my-app
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  git:
    commit:
      author:
        name: flux
        email: flux@example.com
      messageTemplate: "chore: update my-app to {{ .NewTag }}"
    push:
      branch: main
  update:
    path: ./apps/my-app
    strategy: Setters
```

Mark the value to update with a comment:
```yaml
image:
  tag: "1.2.0"  # {"$imagepolicy": "flux-system:my-app:tag"}
```

## Rollback Strategies

### ArgoCD

- **Automatic**: `selfHeal: true` reverts manual drift. For failed syncs, ArgoCD retries per the `retry` config but doesn't auto-rollback to a previous version.
- **Manual**: `argocd app rollback my-app` reverts to a previous sync revision.
- **Health-based**: configure custom health checks on the Application. If health degrades after sync, operators or automation can trigger rollback.

### Flux

- **Automatic**: `upgrade.remediation.retries` + `remediateLastFailure: true` rolls back after failed upgrades.
- **Manual**: `flux suspend hr my-app` then `helm rollback my-app` via CLI.
- **History**: Flux stores Helm release history. `helm history my-app -n my-app` shows previous revisions.

### General

- Always test upgrades in staging before prod.
- Pin chart versions in prod (`targetRevision: 1.2.0`), use ranges or `main` in dev.
- Monitor the GitOps controller's sync status and set alerts on `Degraded` or `OutOfSync` states.
