# Strimzi: MirrorMaker 2

## Contents

- KafkaMirrorMaker2 CR
- Replication topologies
- Topic and group filtering
- Offset synchronization
- Failover procedure

## KafkaMirrorMaker2 CR

v1 API uses `spec.target` for the target cluster and `spec.mirrors[].source` for each source.

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaMirrorMaker2
metadata:
  name: mm2-to-dr
  namespace: kafka
spec:
  version: 3.8.0
  replicas: 2
  target:
    alias: target
    bootstrapServers: target-kafka-bootstrap.target-ns:9093
    tls:
      trustedCertificates:
        - secretName: target-cluster-ca-cert
          pattern: "*.crt"
    config:
      config.storage.replication.factor: -1
      offset.storage.replication.factor: -1
      status.storage.replication.factor: -1
  mirrors:
    - source:
        alias: source
        bootstrapServers: source-kafka-bootstrap.source-ns:9093
        tls:
          trustedCertificates:
            - secretName: source-cluster-ca-cert
              pattern: "*.crt"
      sourceConnector:
        tasksMax: 4
        config:
          replication.factor: -1
          offset-syncs.topic.replication.factor: -1
          sync.topic.acls.enabled: "false"
          refresh.topics.interval.seconds: "60"
      heartbeatConnector:
        config:
          heartbeats.topic.replication.factor: -1
      checkpointConnector:
        config:
          checkpoints.topic.replication.factor: -1
          sync.group.offsets.enabled: "true"
          sync.group.offsets.interval.seconds: "60"
      topicsPattern: "orders.*|customers.*"
      groupsPattern: ".*"
  resources:
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      cpu: "1"
      memory: 2Gi
```

Using `-1` for replication factors defers to the broker's `default.replication.factor`.

## Replication Topologies

### Active-Passive (Disaster Recovery)

One-way replication. Target cluster is a hot standby.

```text
Source ────────→ Target (standby)
```

- Single `mirrors` entry: source → target
- Target topics are prefixed: `source.orders.payment.completed`
- Consumers on target use the prefixed topic names

### Active-Active (Bidirectional)

Both clusters produce and consume. Each replicates to the other. Requires two KafkaMirrorMaker2 resources (one per direction) or two mirror entries.

```yaml
mirrors:
  - source:
      alias: cluster-a
      bootstrapServers: cluster-a-bootstrap:9093
    topicsPattern: ".*"
    topicsExcludePattern: "cluster-a\..*"
  - source:
      alias: cluster-b
      bootstrapServers: cluster-b-bootstrap:9093
    topicsPattern: ".*"
    topicsExcludePattern: "cluster-b\..*"
```

The default `DefaultReplicationPolicy` prefixes topics with the source alias, preventing loops.

### Fan-Out / Aggregation

**Fan-out:** One source replicates to multiple targets.
**Aggregation:** Multiple sources replicate to one target.

Both patterns use multiple `mirrors` entries or multiple KafkaMirrorMaker2 resources.

## Topic and Group Filtering

### Topic Patterns

```yaml
# include specific topics
topicsPattern: "orders.*|customers.*|inventory.*"

# exclude internal topics
topicsExcludePattern: ".*\\.internal|__.*"
```

Patterns use Java regex syntax. Filter as tightly as possible — replicating unnecessary topics wastes bandwidth and storage.

### Group Patterns

```yaml
# replicate all consumer group offsets
groupsPattern: ".*"

# specific groups only
groupsPattern: "order-service-.*|analytics-.*"

# exclude
groupsExcludePattern: "connect-.*|__.*"
```

### Renaming Policies

**DefaultReplicationPolicy** (default):
- Target topic: `<source-alias>.<original-topic>`
- `source.orders` on the target cluster

**IdentityReplicationPolicy:**
- Target topic keeps the same name as source
- Use only for active-passive where consumers switch clusters

```yaml
sourceConnector:
  config:
    replication.policy.class: org.apache.kafka.connect.mirror.IdentityReplicationPolicy
```

With IdentityReplicationPolicy in active-active, use `topicsExcludePattern` carefully to prevent infinite replication loops.

## Offset Synchronization

The checkpoint connector syncs consumer group offsets so consumers can resume on the target.

```yaml
checkpointConnector:
  config:
    sync.group.offsets.enabled: "true"
    sync.group.offsets.interval.seconds: "60"
    emit.checkpoints.interval.seconds: "60"
```

- Checkpoints map source offsets → target offsets
- Offset translation is approximate — consumers may reprocess a small number of messages after failover

## Failover Procedure

### Active-Passive Failover

1. **Stop producers** writing to source cluster
2. **Wait** for MirrorMaker to drain remaining messages (check lag)
3. **Verify** consumer group offsets are synced:
   ```bash
   kubectl exec -n kafka target-kafka-0 -- bin/kafka-console-consumer.sh \
     --bootstrap-server localhost:9092 \
     --topic source.checkpoints.internal \
     --from-beginning --max-messages 10
   ```
4. **Point consumers** to target cluster
   - If using IdentityReplicationPolicy: same topic names
   - If using DefaultReplicationPolicy: update topic names to `source.<topic>`
5. **Verify** consumers are processing from target
6. **Optionally stop** MirrorMaker (or reverse replication direction for failback)

### Monitoring Replication Lag

```text
kafka_connect_mirror_source_connector_replication_latency_ms
kafka_connect_mirror_source_connector_byte_rate
kafka_connect_mirror_source_connector_record_count
```

Alert if replication latency exceeds acceptable RPO (Recovery Point Objective).
