# Installation Methods

## Contents

- OLM (Operator Lifecycle Manager)
- Helm
- Raw Manifests
- Comparison Matrix
- Per-Operator Recommendations

## OLM (Operator Lifecycle Manager)

Best for OpenShift environments or when managing many operators centrally.

```yaml
# install OLM (skip on OpenShift — already present)
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.28.0/install.sh | bash -s v0.28.0

# subscribe to an operator
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: strimzi-kafka-operator
  namespace: operators
spec:
  channel: stable
  name: strimzi-kafka-operator
  source: operatorhubio-catalog
  sourceNamespace: olm
  installPlanApproval: Manual  # Manual for production, Automatic for dev
```

Channel management:
- `stable` — production-ready releases
- `alpha` / `preview` — early access, not for production
- Change channel in Subscription to track a different release stream

Approval strategy:
- `Automatic` — operator upgrades applied immediately when new CSV appears
- `Manual` — requires admin approval per upgrade (recommended for production)

Upgrading: change Subscription channel or approve pending InstallPlan.

## Helm

Most common method. Works with any Kubernetes cluster and integrates with GitOps.

```bash
# add repo and install
helm repo add strimzi https://strimzi.io/charts/
helm repo update

helm install strimzi-operator strimzi/strimzi-kafka-operator \
  --namespace strimzi-system \
  --create-namespace \
  --values values.yaml \
  --version 0.44.0 \
  --atomic \
  --wait
```

Upgrading:

```bash
# CRD gotcha: Helm does NOT upgrade CRDs automatically
# always apply CRDs first
kubectl apply --server-side -f https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.44.0/strimzi-crds-0.44.0.yaml

helm upgrade strimzi-operator strimzi/strimzi-kafka-operator \
  --namespace strimzi-system \
  --values values.yaml \
  --version 0.44.0 \
  --atomic \
  --wait
```

values.yaml override patterns:
- Pin image tags explicitly for reproducibility
- Set resource requests/limits for the operator pod
- Configure `watchNamespaces` to restrict scope
- Enable/disable sub-components (e.g., entity operator in Strimzi)

## Raw Manifests

Best for air-gapped environments, minimal clusters, or learning.

```bash
# apply from release URL
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.0/cert-manager.yaml

# or with kustomize
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - https://github.com/cert-manager/cert-manager/releases/download/v1.16.0/cert-manager.yaml
patches:
  - target:
      kind: Deployment
      name: cert-manager
    patch: |
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: 256Mi
```

Version pinning: always use release tags, never `main` or `latest`.

Upgrading: download new version manifests, diff against current, apply.

## Comparison Matrix

| Feature | OLM | Helm | Raw Manifests |
|---------|-----|------|---------------|
| CRD lifecycle management | Automatic | Manual (apply CRDs separately) | Manual |
| Dependency management | Built-in (CSV dependencies) | Subchart dependencies | None |
| Rollback | CSV-based rollback | `helm rollback` | Manual `kubectl apply` of old version |
| GitOps friendly | Moderate (Subscription CR) | High (values files in Git) | High (manifests in Git) |
| Upgrade approval workflow | Built-in (Manual approval) | External (CI/CD gates) | External |
| Multi-operator management | Central catalog | Per-chart | Per-manifest set |
| Air-gap support | Mirror catalog | Mirror chart repo | Copy manifests |
| Configuration flexibility | Limited to CSV-exposed params | Full values.yaml | Full manifest editing |
| Learning curve | High | Medium | Low |

## Per-Operator Recommendations

| Operator | Primary Method | Chart / Source | Notes |
|----------|---------------|----------------|-------|
| Strimzi | Helm | `strimzi/strimzi-kafka-operator` | OLM also well-supported |
| Keycloak | Helm or manifests | `keycloak/keycloak-operator` | Official operator chart |
| kube-prometheus-stack | Helm only | `prometheus-community/kube-prometheus-stack` | Too many subcharts for manual management |
| cert-manager | Helm | `cert-manager/cert-manager` | Official chart |
| external-dns | Helm | `kubernetes-sigs/external-dns` or `bitnami/external-dns` | Raw manifests also straightforward |

## Post-Install Verification

After installing any operator:

```bash
# 1. operator pod running
kubectl get pods -n <operator-ns> -l app=<operator-label>

# 2. CRDs registered
kubectl get crd | grep <operator-domain>

# 3. webhooks registered (if applicable)
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations | grep <operator>

# 4. apply a minimal test CR and verify reconciliation
kubectl apply -f test-cr.yaml
kubectl get <cr> <name> -o jsonpath='{.status.conditions}'
```