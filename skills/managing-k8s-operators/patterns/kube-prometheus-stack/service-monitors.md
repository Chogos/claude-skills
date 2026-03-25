# kube-prometheus-stack: ServiceMonitor & PodMonitor

## Contents

- ServiceMonitor basics
- PodMonitor basics
- ServiceMonitor vs PodMonitor
- Prometheus service discovery
- Common application patterns
- Metric relabeling

## ServiceMonitor

Declares how Prometheus should scrape a Service's endpoints.

### Basic ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
  namespace: app
  labels:
    release: kube-prometheus-stack  # must match Prometheus serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: http-metrics       # must match Service port name
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```

### Cross-Namespace Monitoring

```yaml
spec:
  namespaceSelector:
    matchNames:
      - app-production
      - app-staging
  # or watch all namespaces:
  # namespaceSelector:
  #   any: true
  selector:
    matchLabels:
      app: my-app
```

### TLS Scraping

```yaml
endpoints:
  - port: https-metrics
    scheme: https
    tlsConfig:
      insecureSkipVerify: true  # for self-signed certs
      # or provide CA:
      # ca:
      #   secret:
      #     name: metrics-tls
      #     key: ca.crt
    bearerTokenSecret:
      name: metrics-token
      key: token
```

## PodMonitor

Scrapes pods directly without requiring a Service. Use when pods expose metrics but have no Service (batch jobs, DaemonSets with host networking).

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: batch-jobs
  namespace: app
spec:
  selector:
    matchLabels:
      app: batch-processor
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 60s
```

## ServiceMonitor vs PodMonitor

| Use Case | Choose |
|----------|--------|
| App has a Service | ServiceMonitor |
| Headless Service with per-pod scraping | PodMonitor |
| DaemonSet with host networking | PodMonitor |
| Batch jobs / CronJobs | PodMonitor |
| Sidecars (e.g., exporter containers) | Either — ServiceMonitor if Service exists |

## Prometheus Service Discovery

Prometheus operator discovers ServiceMonitors/PodMonitors via label selectors configured on the Prometheus CR.

```yaml
# in kube-prometheus-stack values.yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelector: {}          # {} = match all ServiceMonitors
    serviceMonitorNamespaceSelector: {} # {} = all namespaces
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}
```

Common pitfall: ServiceMonitor exists but Prometheus doesn't scrape it.

Debugging checklist:
1. Check labels match `serviceMonitorSelector` on the Prometheus CR
2. Check namespace is included in `serviceMonitorNamespaceSelector`
3. Verify the Service exists and port names match
4. Check Prometheus targets UI: `<prometheus-url>/targets`
5. Check prometheus-operator logs for reconciliation errors

```bash
# quick check: does Prometheus know about the ServiceMonitor?
kubectl get prometheus -n monitoring -o yaml | grep -A5 serviceMonitorSelector
```

## Common Application Patterns

### Spring Boot / Java Actuator

```yaml
# Service exposes actuator on management port
endpoints:
  - port: management    # typically 8081
    path: /actuator/prometheus
    interval: 30s
```

### Go Application

```yaml
# standard /metrics endpoint from prometheus/client_golang
endpoints:
  - port: http
    path: /metrics
    interval: 15s
```

### NGINX with Exporter Sidecar

```yaml
# deploy nginx-prometheus-exporter as sidecar, expose on port 9113
endpoints:
  - port: nginx-exporter
    path: /metrics
    interval: 30s
```

### Redis Exporter

```yaml
# redis-exporter as sidecar on port 9121
endpoints:
  - port: redis-exporter
    path: /metrics
    interval: 30s
```

## Metric Relabeling

### Drop High-Cardinality Metrics

```yaml
endpoints:
  - port: http-metrics
    metricRelabelings:
      - sourceLabels: [__name__]
        regex: "go_gc_.*|process_.*"
        action: drop
```

### Rename Metrics

```yaml
    metricRelabelings:
      - sourceLabels: [__name__]
        regex: "http_request_duration_seconds"
        targetLabel: __name__
        replacement: "app_request_latency_seconds"
```

### Add Static Labels

```yaml
endpoints:
  - port: http-metrics
    relabelings:
      - targetLabel: environment
        replacement: production
```

### Filter by Label Value

```yaml
    metricRelabelings:
      - sourceLabels: [method]
        regex: "GET|POST|PUT|DELETE"
        action: keep
```

`relabelings` — applied before scrape (target labels). `metricRelabelings` — applied after scrape (metric labels). Use `metricRelabelings` to reduce cardinality and storage.