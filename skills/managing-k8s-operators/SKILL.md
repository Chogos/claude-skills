---
name: managing-k8s-operators
description: Kubernetes operator management best practices. Use when deploying, configuring, upgrading, or troubleshooting Kubernetes operators including Strimzi (Kafka), Keycloak, kube-prometheus-stack (Prometheus, Grafana, Alertmanager), cert-manager, or external-dns.
---

# Managing Kubernetes Operators

## Operator Quick Reference

| Operator | Primary CRDs | Typical Namespace | Project |
|----------|-------------|-------------------|---------|
| Strimzi | Kafka, KafkaTopic, KafkaUser, KafkaConnect, KafkaMirrorMaker2 | strimzi-system | strimzi.io |
| Keycloak | Keycloak, KeycloakRealmImport | keycloak-system | keycloak.org |
| kube-prometheus-stack | Prometheus, Alertmanager, ServiceMonitor, PodMonitor, PrometheusRule | monitoring | prometheus-operator |
| cert-manager | Issuer, ClusterIssuer, Certificate, CertificateRequest | cert-manager | cert-manager.io |
| external-dns | (no CRDs — annotation-driven) | external-dns | kubernetes-sigs |

## Decision Matrix

Read the pattern file(s) matching the user's intent. Load only what's needed.

| User Intent | Load |
|---|---|
| Install / compare installation methods | `installation-methods.md` |
| Deploy Kafka cluster | `patterns/strimzi/cluster-setup.md` |
| Configure Kafka listeners or storage | `patterns/strimzi/listeners-and-storage.md` |
| Create Kafka topics or users | `patterns/strimzi/topics-and-users.md` |
| Set up Kafka Connect / connectors | `patterns/strimzi/kafka-connect.md` |
| Replicate Kafka across clusters | `patterns/strimzi/mirror-maker.md` |
| Configure Keycloak realm | `patterns/keycloak/realm-configuration.md` |
| Set up SSO / identity providers | `patterns/keycloak/identity-providers.md` |
| Create Keycloak clients | `patterns/keycloak/client-setup.md` |
| Customize Keycloak themes or SPIs | `patterns/keycloak/customization.md` |
| Add Prometheus scraping for a service | `patterns/kube-prometheus-stack/service-monitors.md` |
| Create alerting rules | `patterns/kube-prometheus-stack/alerting-rules.md` |
| Provision Grafana dashboards | `patterns/kube-prometheus-stack/grafana-dashboards.md` |
| Set up Thanos for long-term metrics | `patterns/kube-prometheus-stack/thanos-integration.md` |
| Set up TLS certificates | `patterns/cert-manager/issuers.md` + `patterns/cert-manager/certificates.md` |
| Configure ACME / challenge solvers | `patterns/cert-manager/issuers.md` + `patterns/cert-manager/webhook-solvers.md` |
| Troubleshoot certificate renewal | `patterns/cert-manager/certificates.md` |
| Auto-manage DNS records | `patterns/external-dns/provider-setup.md` + `patterns/external-dns/source-configuration.md` |
| Configure DNS record ownership | `patterns/external-dns/ownership-and-policy.md` |
| Upgrade any operator | `installation-methods.md` + cross-cutting section below |
| Monitor operator health | Cross-cutting section below (no extra file) |

## Installation

Three methods exist: OLM, Helm, and raw manifests. See `installation-methods.md` for full comparison.

| Operator | Recommended Method | Notes |
|----------|-------------------|-------|
| Strimzi | Helm or OLM | `strimzi/strimzi-kafka-operator` chart |
| Keycloak | Helm or manifests | `keycloak/keycloak-operator` chart |
| kube-prometheus-stack | Helm only | Complex subchart dependencies |
| cert-manager | Helm | `cert-manager/cert-manager` chart |
| external-dns | Helm | `kubernetes-sigs/external-dns` chart |

## Cross-Cutting Concerns

### CRD Versioning

CRD API versions progress: `v1alpha1` → `v1beta1` → `v1`. Before upgrading:

```bash
# check stored versions
kubectl get crd kafkas.kafka.strimzi.io -o jsonpath='{.status.storedVersions}'

# list all CRDs for an operator
kubectl get crd | grep strimzi.io
```

- Back up CRDs before upgrades: `kubectl get crd <name> -o yaml > backup.yaml`
- Check operator release notes for deprecated API versions
- Conversion webhooks handle version translation — verify they're running after upgrade

### Upgrade Strategies

General operator upgrade checklist:

1. Read release notes and breaking changes
2. Back up CRDs and CRs: `kubectl get <crd> -A -o yaml > backup.yaml`
3. Check CRD version compatibility between old and new operator
4. Upgrade operator (Helm: `helm upgrade --atomic --wait`, OLM: update subscription channel)
5. Verify operator pod is healthy: `kubectl get pods -n <operator-ns>`
6. Verify managed resources reconciled: check CR `.status.conditions`
7. Run smoke tests against managed workloads

Rolling vs recreate:
- Most operators tolerate rolling updates (leader election handles handoff)
- Recreate if operator doesn't support leader election or runs as singleton

Canary upgrades:
- Run old + new operator in different namespaces with `WATCH_NAMESPACE` scoping
- Validate new operator on non-production namespace before cluster-wide rollout

Deploy the [Strimzi Drain Cleaner](https://github.com/strimzi/drain-cleaner) for graceful pod eviction during node maintenance — it ensures brokers are rolled safely when nodes are drained.

### Monitoring Operator Health

All controller-runtime operators expose standard metrics:

```yaml
# key metrics to watch
controller_runtime_reconcile_total{result="error"}    # reconciliation failures
controller_runtime_reconcile_time_seconds              # reconciliation latency
workqueue_depth                                        # pending reconciliations
workqueue_longest_running_processor_seconds            # stuck reconciliations
```

ServiceMonitor for operator pods:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: strimzi-operator
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames: [strimzi-system]
  selector:
    matchLabels:
      app: strimzi-cluster-operator
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

Alert on reconciliation failures:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: operator-health
  namespace: monitoring
spec:
  groups:
    - name: operator-health
      rules:
        - alert: OperatorReconcileErrors
          expr: rate(controller_runtime_reconcile_total{result="error"}[5m]) > 0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Operator {{ $labels.controller }} has reconciliation errors"
```

### Resource Sizing

Use Guaranteed QoS (requests == limits for CPU and memory) for all operator-managed workloads in production. Burstable pods under node pressure get OOM-killed first, triggering cascading failures.

### RBAC Patterns

- Operators need cluster-wide RBAC for CRDs by default
- Namespace-scoped operators: set `WATCH_NAMESPACE` env var to restrict scope
- Audit operator ClusterRole after install — remove unused rules for least-privilege
- Multi-tenant: one operator per namespace vs cluster-wide with namespace filtering

```bash
# audit what an operator service account can do
kubectl auth can-i --list --as=system:serviceaccount:strimzi-system:strimzi-cluster-operator
```

### Namespace Strategy

- Dedicated namespace per operator: `strimzi-system`, `cert-manager`, `monitoring`, `external-dns`
- Workload CRs (Kafka, Certificate, ServiceMonitor) live in application namespaces
- Operators watch across namespaces unless restricted via `WATCH_NAMESPACE`
- Label namespaces for selective operator targeting where supported

## Troubleshooting Workflow

Generic operator troubleshooting checklist:

1. **Operator pod healthy?**
   `kubectl get pods -n <operator-ns>` — check Ready, restarts
2. **Operator logs?**
   `kubectl logs -n <operator-ns> deploy/<operator-name> --tail=100`
3. **CR status conditions?**
   `kubectl get <cr> <name> -o yaml` — check `.status.conditions` for errors
4. **Events?**
   `kubectl get events -n <ns> --sort-by='.lastTimestamp' --field-selector=reason!=Pulled`
5. **RBAC?**
   `kubectl auth can-i --as=system:serviceaccount:<ns>:<sa> <verb> <resource>`
6. **CRDs installed?**
   `kubectl get crd | grep <operator-domain>`
7. **Webhooks?**
   `kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations`

## Validation Loop

1. Apply CR → check operator logs → check CR `.status` → fix → repeat
2. For upgrades: pre-upgrade checks → upgrade → post-upgrade verification → rollback if needed
3. For new operator installs: install operator → verify CRDs exist → apply test CR → verify reconciliation

## Deep-dive References

### Strimzi (Kafka)
- `patterns/strimzi/cluster-setup.md` — KRaft cluster provisioning, node pools
- `patterns/strimzi/listeners-and-storage.md` — Listener types, authentication, JBOD, volume expansion
- `patterns/strimzi/topics-and-users.md` — KafkaTopic, KafkaUser, ACLs
- `patterns/strimzi/kafka-connect.md` — Connectors, plugin builds, Connect ACLs
- `patterns/strimzi/mirror-maker.md` — MirrorMaker2, cross-cluster replication

### Keycloak
- `patterns/keycloak/realm-configuration.md` — Realm CRs, export/import, settings
- `patterns/keycloak/identity-providers.md` — OIDC, SAML, social providers
- `patterns/keycloak/client-setup.md` — Clients, scopes, service accounts
- `patterns/keycloak/customization.md` — Themes, SPIs, custom providers

### kube-prometheus-stack
- `patterns/kube-prometheus-stack/service-monitors.md` — ServiceMonitor, PodMonitor
- `patterns/kube-prometheus-stack/alerting-rules.md` — PrometheusRule, routing
- `patterns/kube-prometheus-stack/grafana-dashboards.md` — Dashboard provisioning
- `patterns/kube-prometheus-stack/thanos-integration.md` — Long-term storage

### cert-manager
- `patterns/cert-manager/issuers.md` — Issuer, ClusterIssuer, ACME, CA, Vault
- `patterns/cert-manager/certificates.md` — Certificate lifecycle, renewal
- `patterns/cert-manager/webhook-solvers.md` — HTTP01, DNS01 solvers

### external-dns
- `patterns/external-dns/provider-setup.md` — DNS provider configuration
- `patterns/external-dns/source-configuration.md` — Ingress/Service/Gateway sources
- `patterns/external-dns/ownership-and-policy.md` — TXT records, policy modes

## Official References

- [Strimzi Documentation](https://strimzi.io/documentation/)
- [Keycloak Operator Guide](https://www.keycloak.org/operator/installation)
- [kube-prometheus-stack Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [external-dns Documentation](https://kubernetes-sigs.github.io/external-dns/)