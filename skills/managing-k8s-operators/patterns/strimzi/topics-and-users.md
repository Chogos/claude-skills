# Strimzi: Topics and Users

## Contents

- KafkaTopic CR
- KafkaUser CR
- ACL patterns
- Secret management

## KafkaTopic CR

### Basic Topic

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: orders
  namespace: kafka
  labels:
    strimzi.io/cluster: production
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: "604800000"         # 7 days
    min.insync.replicas: "2"
    compression.type: lz4
    cleanup.policy: delete
    max.message.bytes: "1048576"      # 1 MB
    segment.bytes: "1073741824"       # 1 GB
```

The `strimzi.io/cluster` label links the topic to the Kafka cluster.

### Topic Naming

Consistent naming helps with ACL patterns and tooling:

- `<domain>.<entity>.<event>` — e.g., `orders.payment.completed`
- `<team>.<service>.<event>` — e.g., `platform.user-service.user-created`
- Avoid dots in Kafka cluster names — Strimzi uses `<cluster>.<topic>` internally which conflicts

### Config Properties

| Property | Purpose | Guidance |
|----------|---------|----------|
| `retention.ms` | How long to keep messages | Match consumer SLA; -1 for infinite |
| `min.insync.replicas` | Minimum replicas for ack | `replicas - 1` for durability |
| `compression.type` | Broker-side compression | `lz4` (fast) or `zstd` (better ratio) |
| `cleanup.policy` | `delete` or `compact` | `compact` for changelog/state topics |
| `max.message.bytes` | Max message size | Coordinate with producer `max.request.size` |

### Unmanaged Topics

To reference a topic without operator management, set `strimzi.io/managed: "false"`:

```yaml
metadata:
  annotations:
    strimzi.io/managed: "false"
```

The operator creates the topic but won't reconcile config changes.

## KafkaUser CR

### TLS Authentication User

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaUser
metadata:
  name: app-consumer
  namespace: kafka
  labels:
    strimzi.io/cluster: production
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      - resource:
          type: topic
          name: orders
          patternType: literal
        operations: [Read, Describe]
      - resource:
          type: group
          name: app-consumer-group
          patternType: literal
        operations: [Read]
```

### SCRAM-SHA-512 User

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaUser
metadata:
  name: external-producer
  namespace: kafka
  labels:
    strimzi.io/cluster: production
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      - resource:
          type: topic
          name: orders
          patternType: literal
        operations: [Write, Describe, Create]
  quotas:
    producerByteRate: 10485760    # 10 MB/s
    consumerByteRate: 20971520    # 20 MB/s
    requestPercentage: 20         # max 20% of broker request handler capacity
```

## ACL Patterns

### Producer ACLs

```yaml
acls:
  - resource:
      type: topic
      name: orders
      patternType: literal
    operations: [Write, Describe, Create]
  # idempotent producer also needs:
  - resource:
      type: cluster
      name: kafka-cluster
      patternType: literal
    operations: [IdempotentWrite]
  # transactional producer:
  - resource:
      type: transactionalId
      name: "app-producer-*"
      patternType: prefix
    operations: [Write, Describe]
```

### Consumer ACLs

```yaml
acls:
  - resource:
      type: topic
      name: orders
      patternType: literal
    operations: [Read, Describe]
  - resource:
      type: group
      name: app-consumer-group
      patternType: literal
    operations: [Read]
```

### Wildcard / Prefix Patterns

```yaml
# read all topics starting with "orders."
acls:
  - resource:
      type: topic
      name: "orders."
      patternType: prefix
    operations: [Read, Describe]
  - resource:
      type: group
      name: "orders-consumer-"
      patternType: prefix
    operations: [Read]
```

## Secret Management

The operator generates secrets for each KafkaUser:

| Auth Type | Secret Contents |
|-----------|-----------------|
| TLS | `ca.crt`, `user.crt`, `user.key`, `user.p12`, `user.password` |
| SCRAM-SHA-512 | `password`, `sasl.jaas.config` |

```bash
# retrieve generated credentials
kubectl get secret app-consumer -n kafka -o jsonpath='{.data.user\.crt}' | base64 -d

# SCRAM password
kubectl get secret external-producer -n kafka -o jsonpath='{.data.password}' | base64 -d
```

### Mounting in Application Pods

```yaml
# TLS user
volumes:
  - name: kafka-certs
    secret:
      secretName: app-consumer
containers:
  - name: app
    volumeMounts:
      - name: kafka-certs
        mountPath: /etc/kafka/certs
        readOnly: true
    env:
      - name: KAFKA_SSL_KEYSTORE_LOCATION
        value: /etc/kafka/certs/user.p12
      - name: KAFKA_SSL_KEYSTORE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: app-consumer
            key: user.password
```
