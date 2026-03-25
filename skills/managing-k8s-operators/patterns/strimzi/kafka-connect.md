# Strimzi: Kafka Connect

## Contents

- KafkaConnect CR
- Building connector images
- KafkaConnector CR
- Connector examples
- Connect ACLs
- Monitoring Connect
- Auto-restart and error handling

## KafkaConnect CR

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaConnect
metadata:
  name: connect-cluster
  namespace: kafka
  annotations:
    strimzi.io/use-connector-resources: "true"  # enable KafkaConnector CRs
spec:
  version: 3.8.0
  replicas: 3
  bootstrapServers: production-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: production-cluster-ca-cert
        certificate: ca.crt
  authentication:
    type: tls
    certificateAndKey:
      secretName: connect-user
      certificate: user.crt
      key: user.key
  config:
    group.id: connect-cluster
    offset.storage.topic: connect-cluster-offsets
    config.storage.topic: connect-cluster-configs
    status.storage.topic: connect-cluster-status
    offset.storage.replication.factor: 3
    config.storage.replication.factor: 3
    status.storage.replication.factor: 3
    key.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: false
    value.converter.schemas.enable: false
  resources:
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      cpu: "1"
      memory: 2Gi
  metricsConfig:
    type: jmxPrometheusExporter
    valueFrom:
      configMapKeyRef:
        name: kafka-metrics
        key: connect-metrics-config.yml
  build:
    output:
      type: docker
      image: registry.example.com/kafka-connect:latest
      pushSecret: registry-credentials
    plugins:
      - name: debezium-postgres
        artifacts:
          - type: maven
            group: io.debezium
            artifact: debezium-connector-postgres
            version: 2.7.3.Final
      - name: camel-aws-s3
        artifacts:
          - type: maven
            group: org.apache.camel.kafkaconnector
            artifact: camel-aws-s3-sink-kafka-connector
            version: 4.8.2
```

The `strimzi.io/use-connector-resources: "true"` annotation enables managing connectors via KafkaConnector CRs instead of the REST API.

## Building Connector Images

### Strimzi Build (inline)

The `build` section in KafkaConnect CR downloads plugins and builds an image automatically. Supported artifact types: `maven`, `tgz`/`zip`, `jar`.

```yaml
plugins:
  - name: custom-connector
    artifacts:
      - type: tgz
        url: https://example.com/connectors/custom-connector-1.0.tar.gz
        sha512sum: abc123...  # optional integrity check
```

### Dockerfile Approach

For complex plugins, build a custom image and reference it in the CR (omit `build` section):

```dockerfile
FROM quay.io/strimzi/kafka:0.44.0-kafka-3.8.0
USER root:root
COPY ./plugins/ /opt/kafka/plugins/
USER 1001
```

```yaml
spec:
  image: registry.example.com/kafka-connect-custom:1.0
```

## KafkaConnector CR

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaConnector
metadata:
  name: postgres-source
  namespace: kafka
  labels:
    strimzi.io/cluster: connect-cluster
spec:
  class: io.debezium.connector.postgresql.PostgresConnector
  tasksMax: 1
  autoRestart:
    enabled: true
    maxRestarts: 5
  config:
    database.hostname: postgres.db.svc
    database.port: "5432"
    database.user: debezium
    database.password: "${file:/opt/kafka/external-config/db-credentials/password}"
    database.dbname: myapp
    database.server.name: myapp
    topic.prefix: cdc.myapp
    schema.include.list: public
    table.include.list: "public.orders,public.customers"
    plugin.name: pgoutput
    slot.name: debezium_myapp
    publication.name: debezium_myapp
    heartbeat.interval.ms: "10000"
```

### Sensitive Config via External Secrets

Mount secrets as external config rather than putting credentials inline:

```yaml
# in KafkaConnect CR
spec:
  externalConfiguration:
    volumes:
      - name: db-credentials
        secret:
          secretName: postgres-credentials
# reference in connector config: ${file:/opt/kafka/external-config/db-credentials/<key>}
```

## Connector Examples

### Debezium PostgreSQL CDC (Source)

Key settings from the KafkaConnector example above:

- `plugin.name: pgoutput` — use PostgreSQL logical replication
- `slot.name` — unique per connector, survives restarts
- `heartbeat.interval.ms` — prevents WAL bloat on low-traffic tables
- `schema.include.list` / `table.include.list` — scope captured tables

### S3 Sink

```yaml
spec:
  class: io.confluent.connect.s3.S3SinkConnector
  tasksMax: 3
  config:
    topics: "orders,customers"
    s3.bucket.name: data-lake-raw
    s3.region: eu-west-1
    flush.size: "10000"
    rotate.interval.ms: "600000"
    storage.class: io.confluent.connect.s3.storage.S3Storage
    format.class: io.confluent.connect.s3.format.parquet.ParquetFormat
    partitioner.class: io.confluent.connect.storage.partitioner.TimeBasedPartitioner
    path.format: "'year'=YYYY/'month'=MM/'day'=dd"
```

## Connect ACLs

KafkaUser ACLs for a Kafka Connect service account:

The Connect service account needs ACLs for its internal topics (`connect-cluster-configs`, `connect-cluster-offsets`, `connect-cluster-status`) with `[Read, Write, Describe, Create]` operations, plus `[Read]` on the consumer group, plus ACLs for each source/sink topic.

```yaml
acls:
  - resource:
      type: group
      name: connect-cluster
      patternType: literal
    operations: [Read]
  - resource:
      type: topic
      name: connect-cluster-  # use prefix for all 3 internal topics
      patternType: prefix
    operations: [Read, Write, Describe, Create]
```

## Monitoring Connect

```bash
# connector status
kubectl get kafkaconnector -n kafka

# detailed status (REST API via port-forward)
kubectl port-forward svc/connect-cluster-connect-api -n kafka 8083:8083
curl localhost:8083/connectors/postgres-source/status | jq
```

Key metrics:

- `kafka_connect_connector_status` — running/paused/failed
- `kafka_connect_worker_connector_count` — connectors per worker
- `kafka_connect_source_task_source_record_poll_rate` — source throughput
- `kafka_connect_sink_task_sink_record_send_rate` — sink throughput

## Auto-Restart and Error Handling

### Auto-Restart

```yaml
spec:
  autoRestart:
    enabled: true
    maxRestarts: 10  # reset counter after successful run
```

### Dead Letter Queue (Sink Connectors)

```yaml
config:
  errors.tolerance: all
  errors.deadletterqueue.topic.name: dlq.orders
  errors.deadletterqueue.topic.replication.factor: "3"
  errors.deadletterqueue.context.headers.enable: "true"
  errors.log.enable: "true"
  errors.log.include.messages: "true"
```

`errors.tolerance: all` skips bad records and routes them to the DLQ — monitor the DLQ topic for data quality issues. Use `none` (default) to fail fast on any error.
