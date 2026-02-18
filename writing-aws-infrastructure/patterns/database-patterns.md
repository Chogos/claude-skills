# Database Patterns

RDS, DynamoDB, and ElastiCache best practices for AWS infrastructure.

## RDS

### Aurora vs RDS decision

| Factor | RDS | Aurora |
|--------|-----|--------|
| Cost | Lower for small/dev workloads | 20-30% more, but better $/perf at scale |
| Storage | Provisioned EBS, manual scaling | Auto-scales to 128TB, pay for usage |
| Replication | Async to read replicas | Shared storage, replicas lag <100ms |
| Failover | 60-120s | <30s |
| Read replicas | Up to 15 (MySQL/Postgres) | Up to 15, same storage layer |
| Use when | Dev/staging, small production, budget-constrained | Production at scale, high availability requirements |

### Instance sizing

- Start with `db.t4g.medium` for dev/staging (burstable, ARM, cheapest).
- Production: `db.r7g.*` (memory-optimized, Graviton3). Size based on working set fitting in memory.
- Enable Performance Insights (free for 7-day retention) to identify bottlenecks before right-sizing.
- Use `db.serverless` (Aurora Serverless v2) for unpredictable workloads — scales 0.5 to 128 ACUs.

### High availability

```yaml
# CloudFormation — Multi-AZ RDS
RdsInstance:
  Type: AWS::RDS::DBInstance
  DeletionPolicy: Retain
  Properties:
    DBInstanceClass: db.r7g.large
    Engine: postgres
    EngineVersion: "16.4"
    MultiAZ: true
    StorageEncrypted: true
    KmsKeyId: !Ref DatabaseKey
    AutoMinorVersionUpgrade: true
    BackupRetentionPeriod: 14
    PreferredBackupWindow: "03:00-04:00"
    PreferredMaintenanceWindow: "sun:05:00-sun:06:00"
    DeletionProtection: true
    CopyTagsToSnapshot: true
    MonitoringInterval: 60
    MonitoringRoleArn: !GetAtt RdsMonitoringRole.Arn
    EnablePerformanceInsights: true
    PerformanceInsightsRetentionPeriod: 7
    DBSubnetGroupName: !Ref DbSubnetGroup
    VPCSecurityGroups:
      - !Ref DbSecurityGroup
```

- `MultiAZ: true` for production — synchronous standby in a different AZ.
- Read replicas for read-heavy workloads — async, eventual consistency.
- `DeletionPolicy: Retain` — never auto-delete a database.
- `DeletionProtection: true` — blocks accidental deletion via console/CLI.
- Encryption must be enabled at creation — cannot encrypt existing unencrypted instances.

### Parameter groups

Create custom parameter groups instead of using defaults — defaults can't be modified.

Key PostgreSQL parameters:
- `shared_buffers`: 25% of instance memory
- `work_mem`: start at 16-64MB, tune per workload
- `max_connections`: match your pooling strategy (typically 100-500)
- `log_min_duration_statement`: 1000 (log queries slower than 1s)

### Backups

- Automated backups: 14-35 day retention for production.
- Manual snapshots before major changes (migrations, upgrades).
- Cross-region snapshot copies for disaster recovery.
- Test restore regularly — untested backups aren't backups.

## DynamoDB

### Partition key design

The partition key determines data distribution. Hot partitions = throttling.

**Good keys**: high cardinality, uniform distribution.
- `userId`, `orderId`, `sessionId` — unique per entity
- Composite: `customerId#orderDate` — spreads access across partitions

**Bad keys**: low cardinality, temporal clustering.
- `status` (only a few values) — all writes hit same partition
- `date` (today's partition gets all writes) — hot partition

### Capacity modes

| Mode | Use when | Pricing |
|------|----------|---------|
| On-demand | Unpredictable traffic, new tables, dev/staging | Per-request, ~6x provisioned cost per unit |
| Provisioned | Predictable traffic, cost-sensitive | Per-hour for reserved capacity |
| Provisioned + auto-scaling | Predictable with spikes | Base cost + scales on demand |

Start with on-demand. Switch to provisioned + auto-scaling once traffic patterns are clear.

### GSI / LSI patterns

- **LSI**: same partition key, different sort key. Must be created at table creation. Share throughput with base table.
- **GSI**: different partition key and/or sort key. Can be created anytime. Has its own throughput.

```json
{
  "TableName": "Orders",
  "KeySchema": [
    { "AttributeName": "customerId", "KeyType": "HASH" },
    { "AttributeName": "orderDate", "KeyType": "RANGE" }
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "StatusDateIndex",
      "KeySchema": [
        { "AttributeName": "status", "KeyType": "HASH" },
        { "AttributeName": "orderDate", "KeyType": "RANGE" }
      ],
      "Projection": { "ProjectionType": "INCLUDE", "NonKeyAttributes": ["total", "customerId"] }
    }
  ]
}
```

Project only the attributes you query — `INCLUDE` over `ALL` to reduce storage and write costs.

### Single-table design

Combine related entities in one table using overloaded keys:

| PK | SK | Entity |
|----|-----|--------|
| `CUSTOMER#123` | `PROFILE` | Customer profile |
| `CUSTOMER#123` | `ORDER#2024-01-15#abc` | Order |
| `CUSTOMER#123` | `ORDER#2024-02-20#def` | Order |
| `ORDER#abc` | `ITEM#1` | Order item |

Trade-offs: fewer tables, efficient queries for access patterns you designed for. Harder to evolve, complex for ad-hoc queries. Use single-table when access patterns are well-defined and stable.

### TTL for expiry

Enable TTL to auto-delete expired items at no cost:

```json
{
  "TableName": "Sessions",
  "TimeToLiveSpecification": {
    "AttributeName": "expiresAt",
    "Enabled": true
  }
}
```

Store `expiresAt` as Unix epoch (seconds). Items are deleted within 48 hours of expiry — filter stale items in application code.

## ElastiCache

### Redis vs Memcached

| Factor | Redis | Memcached |
|--------|-------|-----------|
| Data structures | Strings, lists, sets, hashes, sorted sets, streams | Strings only |
| Persistence | Optional RDB/AOF snapshots | None |
| Replication | Primary-replica with auto-failover | None |
| Use when | Sessions, leaderboards, queues, pub/sub, complex caching | Simple key-value caching, multi-threaded reads |

Redis is the default choice. Use Memcached only for simple caching where multi-threaded performance matters more than features.

### Cluster mode

- **Cluster mode disabled**: single shard, up to 5 replicas. Simpler. Fine for <100GB and moderate throughput.
- **Cluster mode enabled**: data partitioned across shards. Required for >100GB or high write throughput. Resharding is online but slow.

### Connection patterns

From Lambda:
```typescript
// Initialize outside handler for connection reuse across invocations
import { createClient } from "redis";

const client = createClient({ url: process.env.REDIS_URL });
const connected = client.connect();

export async function handler(event: Event) {
    await connected;
    const value = await client.get(`cache:${event.key}`);
    // ...
}
```

From ECS: use a connection pool. Set `maxRetriesPerRequest: 3` and connection timeout. Place ElastiCache in private subnets, same AZ as compute for lowest latency.
