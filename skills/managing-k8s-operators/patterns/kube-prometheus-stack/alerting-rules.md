# kube-prometheus-stack: Alerting Rules

## Contents

- PrometheusRule CR
- Common alert patterns
- Alertmanager configuration
- Alert design best practices
- Testing alerts

## PrometheusRule CR

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack  # must match Prometheus ruleSelector
spec:
  groups:
    - name: app.rules
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
            > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High 5xx error rate on {{ $labels.service }}"
            description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"
            runbook_url: "https://runbooks.example.com/high-error-rate"
```

Recording rules (pre-computed queries for dashboard performance):

```yaml
rules:
  - record: job:http_requests_total:rate5m
    expr: sum(rate(http_requests_total[5m])) by (job)
```

## Common Alert Patterns

### Pod CrashLooping

```yaml
- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 0
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
```

### Certificate Expiring Soon

```yaml
- alert: CertificateExpiringSoon
  expr: certmanager_certificate_expiration_timestamp_seconds - time() < 7 * 24 * 3600
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Certificate {{ $labels.name }} in {{ $labels.namespace }} expires in less than 7 days"
```

### High Memory Usage

```yaml
- alert: ContainerHighMemory
  expr: |
    container_memory_working_set_bytes{container!=""}
    /
    kube_pod_container_resource_limits{resource="memory"}
    > 0.9
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "{{ $labels.container }} in {{ $labels.pod }} is using >90% of memory limit"
```

### Disk Usage

```yaml
- alert: PersistentVolumeFillingUp
  expr: |
    kubelet_volume_stats_available_bytes
    /
    kubelet_volume_stats_capacity_bytes
    < 0.15
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "PVC {{ $labels.persistentvolumeclaim }} in {{ $labels.namespace }} has less than 15% free space"
```

### Kafka Consumer Lag (Strimzi)

```yaml
- alert: KafkaConsumerLagHigh
  expr: kafka_consumergroup_lag_sum > 10000
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Consumer group {{ $labels.consumergroup }} has high lag on topic {{ $labels.topic }}"
```

## Alertmanager Configuration

### AlertmanagerConfig CR (namespace-scoped)

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: app-alerts
  namespace: app
  labels:
    release: kube-prometheus-stack
spec:
  route:
    receiver: slack-app
    matchers:
      - name: severity
        matchType: "="
        value: critical
    groupBy: [alertname, namespace]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
  receivers:
    - name: slack-app
      slackConfigs:
        - apiURL:
            name: slack-webhook
            key: url
          channel: "#app-alerts"
          sendResolved: true
          title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
          text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
```

### Receivers

**PagerDuty:**
```yaml
receivers:
  - name: pagerduty-critical
    pagerdutyConfigs:
      - routingKey:
          name: pagerduty-secret
          key: routing-key
        severity: '{{ .CommonLabels.severity }}'
```

**Email:**
```yaml
receivers:
  - name: email-team
    emailConfigs:
      - to: team@example.com
        sendResolved: true
```

**Webhook:**
```yaml
receivers:
  - name: webhook
    webhookConfigs:
      - url: https://hooks.example.com/alerts
        sendResolved: true
```

### Inhibition Rules

Suppress warnings when critical alerts fire for the same service:

```yaml
inhibitRules:
  - sourceMatch:
      - name: severity
        value: critical
    targetMatch:
      - name: severity
        value: warning
    equal: [alertname, namespace]
```

## Alert Design Best Practices

**Symptom-based over cause-based:** Alert on "error rate is high" not "pod restarted" — the user cares about impact, not mechanism.

**Severity levels:**
| Severity | Action | Example |
|----------|--------|---------|
| critical | Pages on-call, immediate response | Service down, data loss risk |
| warning | Creates ticket, next business day | Disk filling up, cert expiring |
| info | Dashboard only, no notification | Deployment completed |

**`for` duration:** Avoid flapping. Use 5-15m for most alerts. Shorter for genuine emergencies (data loss). Longer for noisy metrics.

**Labels and annotations:**
- `severity` — routing and inhibition
- `summary` — one-line alert description with template variables
- `description` — detailed context
- `runbook_url` — link to remediation steps

## Testing Alerts

```bash
# validate rule syntax
promtool check rules rules.yaml

# unit test rules
promtool test rules test.yaml
```

Unit test file:

```yaml
rule_files:
  - rules.yaml
evaluation_interval: 1m
tests:
  - interval: 1m
    input_series:
      - series: 'http_requests_total{status="500", service="api"}'
        values: "0+10x15"
      - series: 'http_requests_total{status="200", service="api"}'
        values: "0+100x15"
    alert_rule_test:
      - eval_time: 10m
        alertname: HighErrorRate
        exp_alerts:
          - exp_labels:
              severity: critical
              service: api
```