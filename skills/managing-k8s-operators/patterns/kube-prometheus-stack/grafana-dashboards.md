# kube-prometheus-stack: Grafana Dashboards

## Contents

- Dashboard provisioning via ConfigMap
- Dashboard-as-code
- Common dashboard patterns
- Dashboard management
- Variables and templating

## Dashboard Provisioning via ConfigMap

The Grafana sidecar auto-discovers ConfigMaps with the label `grafana_dashboard: "1"`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  app-dashboard.json: |
    {
      "dashboard": {
        "title": "App Overview",
        "uid": "app-overview",
        "panels": [...],
        "templating": {
          "list": [...]
        }
      },
      "overwrite": true
    }
```

The sidecar label is configurable in values.yaml:

```yaml
grafana:
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
      searchNamespace: ALL  # or specific namespaces
      folderAnnotation: grafana_folder
```

### Folder Organization

Use the `grafana_folder` annotation to organize dashboards:

```yaml
metadata:
  annotations:
    grafana_folder: "Application Dashboards"
```

## Dashboard-as-Code

### Storing Dashboards in Git

Export dashboard JSON from Grafana UI → store in Git → deploy via ConfigMap or Helm.

```
dashboards/
  app-overview.json
  infrastructure.json
  kafka-cluster.json
```

### Helm Values Approach

```yaml
grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: custom
          orgId: 1
          folder: "Custom"
          type: file
          disableDeletion: true
          editable: false
          options:
            path: /var/lib/grafana/dashboards/custom
  dashboardsConfigMaps:
    custom: app-dashboards-configmap
```

### Grafonnet (Jsonnet)

Programmatic dashboards for consistent styling across teams:

```jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local prometheus = grafana.prometheus;
local graphPanel = grafana.graphPanel;

dashboard.new(
  'App Overview',
  uid='app-overview',
  tags=['app', 'generated'],
)
.addPanel(
  graphPanel.new(
    'Request Rate',
    datasource='Prometheus',
  ).addTarget(
    prometheus.target('sum(rate(http_requests_total[5m])) by (service)')
  ),
  gridPos={ x: 0, y: 0, w: 12, h: 8 },
)
```

Build: `jsonnet -J vendor dashboard.jsonnet > dashboard.json`

## Common Dashboard Patterns

### RED Metrics (Rate, Errors, Duration)

Standard service overview with three panels:

```
Rate:     sum(rate(http_requests_total[5m])) by (service)
Errors:   sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)
Duration: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
```

### USE Metrics (Utilization, Saturation, Errors)

Infrastructure dashboard:

```
CPU Utilization:    1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)
Memory Utilization: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
Disk Saturation:    rate(node_disk_io_time_weighted_seconds_total[5m])
```

### Operator Health Dashboard

```
Reconcile Rate:    sum(rate(controller_runtime_reconcile_total[5m])) by (controller)
Error Rate:        sum(rate(controller_runtime_reconcile_total{result="error"}[5m])) by (controller)
Queue Depth:       workqueue_depth{name=~".*"}
Reconcile Latency: histogram_quantile(0.99, sum(rate(controller_runtime_reconcile_time_seconds_bucket[5m])) by (le, controller))
```

## Dashboard Management

### Preventing Manual Edits

Provisioned dashboards should be read-only to prevent drift:

```yaml
grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: custom
          disableDeletion: true
          editable: false          # prevents UI edits
```

### Dashboard Versioning

- Export JSON from Grafana API: `curl <grafana>/api/dashboards/uid/<uid>`
- Strip `id` and `version` fields before committing (they're instance-specific)
- Keep `uid` stable for consistent URLs and references

## Variables and Templating

Common variable definitions for reusable dashboards:

```json
{
  "templating": {
    "list": [
      {
        "name": "datasource",
        "type": "datasource",
        "query": "prometheus"
      },
      {
        "name": "namespace",
        "type": "query",
        "datasource": "${datasource}",
        "query": "label_values(kube_namespace_labels, namespace)",
        "multi": true,
        "includeAll": true
      },
      {
        "name": "pod",
        "type": "query",
        "datasource": "${datasource}",
        "query": "label_values(kube_pod_info{namespace=~\"$namespace\"}, pod)",
        "multi": true,
        "includeAll": true
      }
    ]
  }
}
```

Use `$namespace` and `$pod` in panel queries: `sum(rate(http_requests_total{namespace=~"$namespace"}[5m])) by (pod)`