# Strimzi: Kafka Cluster Setup

## Contents

- Minimal Kafka cluster
- Production Kafka cluster
- KafkaNodePool roles
- Combined controller+broker (small clusters)
- Health checks
- Migrating from ZooKeeper

## Minimal Kafka Cluster

Development / testing setup with ephemeral storage:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: combined
  labels:
    strimzi.io/cluster: dev-cluster
spec:
  replicas: 1
  roles: [controller, broker]
  storage:
    type: ephemeral
---
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: dev-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.8.0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

## Production Kafka Cluster

Separate controller and broker node pools. Guaranteed QoS (requests == limits) for all pools.

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: controller
  labels:
    strimzi.io/cluster: production
spec:
  replicas: 3
  roles: [controller]
  storage:
    type: persistent-claim
    size: 20Gi
    class: kafka-gp3
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 500m
      memory: 2Gi
---
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: broker
  labels:
    strimzi.io/cluster: production
spec:
  replicas: 3
  roles: [broker]
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        class: kafka-gp3
        deleteClaim: false
  resources:
    requests:
      cpu: "2"
      memory: 8Gi
    limits:
      cpu: "2"
      memory: 8Gi
  jvmOptions:
    -Xms: 2048m
    -Xmx: 2048m
  template:
    pod:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: strimzi.io/pool-name
                    operator: In
                    values: [broker]
              topologyKey: kubernetes.io/hostname
---
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: production
  namespace: kafka
spec:
  kafka:
    version: 3.8.0
    listeners:
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
      - name: external
        port: 9094
        type: loadbalancer
        tls: true
        authentication:
          type: scram-sha-512
    authorization:
      type: simple
    config:
      auto.create.topics.enable: false
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      num.partitions: 12
      log.retention.ms: 604800000
      log.retention.bytes: -1
    rack:
      topologyKey: topology.kubernetes.io/zone
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: kafka-metrics-config.yml
  entityOperator:
    topicOperator:
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: 100m
          memory: 256Mi
    userOperator:
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: 100m
          memory: 256Mi
  kafkaExporter:
    topicRegex: ".*"
    groupRegex: ".*"
  cruiseControl: {}
```

JVM heap should be ≤50% of container memory — the rest serves as page cache for Kafka's hot path. With 8Gi container memory and 2Gi heap, 6Gi is available for page cache.

## KafkaNodePool Roles

| Role | Purpose |
|------|---------|
| `controller` | Metadata management (replaces ZooKeeper). 3 replicas, low CPU/memory. |
| `broker` | Message storage and serving. Scale based on throughput. |
| `controller` + `broker` | Combined for small clusters (dev, staging). |

Resources, storage, and replicas are defined per node pool, not on the Kafka CR.

Controller quorum is static — you cannot change from combined to separated pools (or vice versa) after deployment. Choose your topology upfront.

## Combined Controller+Broker (Small Clusters)

For dev/staging where separate pools are overkill:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: dual-role
  labels:
    strimzi.io/cluster: staging
spec:
  replicas: 3
  roles: [controller, broker]
  storage:
    type: persistent-claim
    size: 100Gi
    class: kafka-gp3
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 500m
      memory: 2Gi
```

## Health Checks

```bash
# cluster status
kubectl get kafka production -n kafka -o jsonpath='{.status.conditions}'

# check node pools
kubectl get kafkanodepool -n kafka -l strimzi.io/cluster=production

# check individual pods
kubectl get pods -n kafka -l strimzi.io/pool-name=broker
```

Key status conditions:

- `Ready` — cluster is fully operational
- `NotReady` — check `.status.conditions[].message` for details
- `ReconciliationPaused` — operator paused via annotation

Key metrics (via JMX exporter):

- `kafka_server_replicamanager_underreplicatedpartitions` — should be 0
- `kafka_controller_kafkacontroller_offlinepartitionscount` — should be 0
- `kafka_server_replicamanager_partitioncount` — total partitions per broker

JMX Exporter does not cover consumer lag — deploy `kafkaExporter` in the Kafka CR (see production example). Use PodMonitor (not ServiceMonitor) for broker metrics when using node pools — target with `strimzi.io/pool-name` label.

## Migrating from ZooKeeper

Strimzi supports in-place migration from ZooKeeper to KRaft. Follow the [Strimzi migration guide](https://strimzi.io/docs/operators/latest/deploying#proc-deploy-migrate-kraft-str) — the process involves creating KafkaNodePool resources and letting the operator handle metadata migration.
