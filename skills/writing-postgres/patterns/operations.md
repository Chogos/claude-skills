# Operations Patterns

## Contents

- Partitioning
- VACUUM and autovacuum
- Connection pooling
- Bulk loading (COPY)
- EXPLAIN ANALYZE
- pg_stat_statements

---

## Partitioning

Declarative partitioning (PG 10+) for tables that grow beyond ~100M rows or need time-based retention.

```sql
-- Range partition by month
create table events (
    id         bigint generated always as identity,
    type       text        not null,
    payload    jsonb       not null default '{}',
    created_at timestamptz not null default now()
) partition by range (created_at);

-- Create partitions (automate with pg_partman or cron)
create table events_2026_01 partition of events
    for values from ('2026-01-01') to ('2026-02-01');
create table events_2026_02 partition of events
    for values from ('2026-02-01') to ('2026-03-01');

-- Default partition catches rows that don't match any partition
create table events_default partition of events default;
```

**Partition types:**

| Type    | Use when                                                           |
| ------- | ------------------------------------------------------------------ |
| `range` | Time-series, date-based retention, monotonically increasing values |
| `list`  | Tenant isolation, status categories, geographic regions            |
| `hash`  | Even distribution when no natural range/list key exists            |

**Rules:**

- Partition key must be part of the primary key and all unique constraints.
- Indexes are per-partition — create on the parent and they propagate.
- `DROP TABLE events_2026_01` is instant vs `DELETE` which generates WAL and dead tuples.
- Partition pruning (`enable_partition_pruning = on`, default) skips irrelevant partitions at query time — always filter on the partition key.

---

## VACUUM and Autovacuum

PostgreSQL uses MVCC — dead tuples accumulate from updates and deletes. VACUUM reclaims space and prevents transaction ID wraparound.

**Autovacuum triggers** (per-table, tunable):

```sql
-- High-churn table: vacuum more aggressively
alter table events set (
    autovacuum_vacuum_scale_factor = 0.01,       -- default 0.2 (20%)
    autovacuum_analyze_scale_factor = 0.005,     -- default 0.1 (10%)
    autovacuum_vacuum_cost_delay = 2             -- default 2ms (PG 12+)
);
```

**Key concepts:**

- `VACUUM` reclaims dead tuples, updates visibility map. Does not return space to OS.
- `VACUUM FULL` rewrites the table, returns space to OS. Takes ACCESS EXCLUSIVE lock — avoid on production.
- **TXID wraparound**: if autovacuum falls behind, PG forces a wraparound-prevention vacuum that freezes all rows — this can take hours and block writes. Monitor `pg_stat_activity` for `autovacuum: FREEZE`.

**Monitoring:**

```sql
-- Tables with most dead tuples (candidates for vacuum tuning)
select schemaname, relname, n_dead_tup, last_autovacuum
from pg_stat_user_tables
order by n_dead_tup desc
limit 10;

-- TXID wraparound risk (age approaching 2 billion = danger)
select relname, age(relfrozenxid) as xid_age
from pg_class
where relkind = 'r'
order by xid_age desc
limit 10;
```

---

## Connection Pooling

PostgreSQL forks a process per connection (~10 MB each). Beyond ~100 connections, use a pooler.

| Pooler    | Mode        | Notes                                                                                                  |
| --------- | ----------- | ------------------------------------------------------------------------------------------------------ |
| PgBouncer | transaction | Most common. `SET LOCAL` works. Prepared statements need `DEALLOCATE ALL` or protocol-level support.   |
| PgCat     | transaction | Rust-based, supports sharding and load balancing.                                                      |
| Supavisor | transaction | Elixir-based, multi-tenant aware.                                                                      |

**Transaction mode caveats:**

- `SET` (session-level) persists beyond your transaction — use `SET LOCAL` instead.
- Prepared statements created with `PREPARE` are session-scoped — use protocol-level prepared statements or `DEALLOCATE ALL` at transaction end.
- `LISTEN`/`NOTIFY` requires session mode.
- Advisory locks are session-scoped by default — use `pg_advisory_xact_lock()` for transaction-scoped locks.

---

## Bulk Loading (COPY)

`COPY` is 5–10x faster than multi-row `INSERT` for large datasets.

```sql
-- Load from CSV
copy products (sku, name, price)
from '/path/to/products.csv'
with (format csv, header true, null '');

-- Load from stdin (used by most drivers)
copy products (sku, name, price) from stdin with (format csv);

-- Unload to CSV
copy (select id, sku, name from products where is_active)
to '/path/to/export.csv'
with (format csv, header true);
```

**Performance tips:**

- Disable indexes and constraints, load, then recreate — faster for initial loads.
- `COPY` in a single transaction avoids per-row WAL overhead.
- For ongoing loads, use `INSERT ... ON CONFLICT` for upsert semantics.
- Binary format (`format binary`) is faster but not human-readable and version-sensitive.

---

## EXPLAIN ANALYZE

`EXPLAIN` shows the query plan. `EXPLAIN ANALYZE` executes the query and shows actual timings.

```sql
explain analyze
select u.id, u.email, count(o.id) as order_count
from users u
left join orders o on o.user_id = u.id
where u.created_at >= '2026-01-01'
group by u.id, u.email;
```

**Reading the plan:**

- **Seq Scan**: full table scan. Fine for small tables; add an index if the table is large and the filter is selective.
- **Index Scan**: uses an index to find rows, then fetches from heap. Good for selective queries.
- **Index Only Scan**: answers the query from the index alone (covering index). Best case.
- **Bitmap Index Scan → Bitmap Heap Scan**: combines multiple index results. Common for `OR` conditions or low-selectivity filters.
- **Nested Loop**: for each outer row, scan inner. Fast when inner is indexed and outer is small.
- **Hash Join**: builds hash table from smaller relation, probes with larger. Good for equijoins on large tables.
- **Sort + Merge Join**: both sides sorted, then merged. Efficient when data is pre-sorted.

**Key metrics:**

- `actual time`: first row..last row in milliseconds.
- `rows`: actual rows vs `rows` in the plan (estimated). Large mismatches mean stale statistics — run `ANALYZE`.
- `Buffers: shared hit/read`: cache hits vs disk reads. High `read` = cold cache or working set exceeds `shared_buffers`.

```sql
-- Use BUFFERS for I/O detail, SETTINGS for non-default params
explain (analyze, buffers, settings) select ...;
```

---

## pg_stat_statements

The most important extension for query performance monitoring. Tracks execution statistics for all queries.

```sql
create extension if not exists pg_stat_statements;

-- Top queries by total execution time
select
    calls,
    round(total_exec_time::numeric, 2)        as total_ms,
    round(mean_exec_time::numeric, 2)          as mean_ms,
    round(stddev_exec_time::numeric, 2)        as stddev_ms,
    rows,
    query
from pg_stat_statements
order by total_exec_time desc
limit 20;

-- Reset stats (do periodically or after deploys)
select pg_stat_statements_reset();
```

**What to look for:**

- High `total_exec_time`: optimize these first — most cumulative impact.
- High `mean_exec_time` with low `calls`: expensive one-off queries.
- High `calls` with low `mean_exec_time`: hot path — even small improvements multiply.
- `rows / calls` ratio: if returning many rows per call, check if the app needs them all.
