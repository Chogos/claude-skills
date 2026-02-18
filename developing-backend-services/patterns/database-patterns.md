# Database Patterns

Migration management, connection pooling, transactions, query optimization, and schema design.

## Migration Management

### Naming Convention

```
001_create_users.sql
002_create_orders.sql
003_add_users_email_index.sql
004_add_orders_status_column.sql
```

Sequential numbering or timestamps (`20240315120000_create_users.sql`). Timestamps avoid conflicts in teams; sequential numbers are easier to read.

### Rules

- **Forward-only in production**: never edit a released migration. Add a new migration to fix mistakes.
- **Idempotent when possible**: use `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX CONCURRENTLY IF NOT EXISTS`.
- **One concern per migration**: don't mix schema changes with data migrations.
- **Test rollbacks in development**: write down migrations, but never rely on them in production.
- **Review migrations in PR**: schema changes get the same review rigor as application code.

### Up / Down Structure

```sql
-- 003_add_users_email_index.sql

-- Up
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users (email);

-- Down (development only)
DROP INDEX IF EXISTS idx_users_email;
```

### Zero-Downtime Migration Strategies

Changing a column type, renaming a column, or adding a NOT NULL constraint requires a multi-step approach:

**Adding a NOT NULL column**:
```
Migration 1: ADD COLUMN new_col DEFAULT 'value'     -- nullable, with default
Migration 2: Backfill existing rows                   -- UPDATE in batches
Migration 3: ALTER COLUMN SET NOT NULL               -- after all rows have values
```

**Renaming a column**:
```
Migration 1: ADD new_column                          -- add new name
Migration 2: Deploy code writing to BOTH columns     -- dual-write
Migration 3: Backfill old rows to new_column         -- copy data
Migration 4: Deploy code reading from new_column     -- switch reads
Migration 5: DROP old_column                         -- clean up
```

**Dropping a column**:
```
Migration 1: Deploy code that no longer reads/writes the column
Migration 2: DROP COLUMN                             -- safe after no code references it
```

Never drop a column that running code still references — deploy code changes first.

## Connection Pooling

### Sizing

Starting point: `pool_size = (2 * cpu_cores) + 1`, typically 10-20 per application instance. Tune based on actual connection wait times — not a formula.

Total connections across all instances must not exceed the database's `max_connections`. With 5 instances at pool size 20, you need at least 100 `max_connections` on the DB (leave headroom for admin connections).

### Timeout Configuration

| Setting | Recommended | Purpose |
|---------|-------------|---------|
| `max_pool_size` | 10-20 | Max concurrent connections per instance |
| `min_pool_size` | 2-5 | Keep-alive connections to avoid cold starts |
| `connection_timeout` | 5s | Max wait for a connection from the pool |
| `idle_timeout` | 10min | Close idle connections after this duration |
| `max_lifetime` | 30min | Close connections after this age (prevents stale connections) |
| `statement_timeout` | 30s | Kill queries running longer than this |
| `lock_timeout` | 10s | Fail if waiting for a lock longer than this |

### Connection Leak Detection

Set `idle_timeout` and `max_lifetime` to automatically reclaim leaked connections. Log a warning when a connection is checked out for longer than expected (e.g., 60s) — likely indicates a missing `close()` or `defer conn.Release()`.

### Connection Pooler (PgBouncer / ProxySQL)

For high-instance-count deployments (50+ instances), use a connection pooler between application and database. The pooler multiplexes thousands of application connections onto a smaller number of database connections.

```
App instances (50 × 20 = 1000 app connections)
  → PgBouncer (pool_size = 100)
    → PostgreSQL (max_connections = 120)
```

## Transaction Handling

### Isolation Levels

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Use Case |
|-------|-----------|-------------------|-------------|----------|
| Read Uncommitted | Yes | Yes | Yes | Almost never — no practical use |
| Read Committed | No | Yes | Yes | Default for most databases. Good for OLTP |
| Repeatable Read | No | No | Yes* | Financial calculations, balance checks |
| Serializable | No | No | No | Critical consistency (account transfers) |

*PostgreSQL's Repeatable Read also prevents phantoms.

**Default to Read Committed.** Escalate only when business logic requires it.

### Transaction Rules

- Keep transactions short — seconds, not minutes. Long transactions hold locks and block other writers.
- Use read-only transactions for pure queries: `BEGIN READ ONLY`. Allows the DB to optimize.
- Retry serialization failures (SQLSTATE 40001 / 40P01) with backoff. These are expected under Serializable or Repeatable Read.
- Don't do network calls inside transactions — a slow HTTP call extends lock duration.

### Optimistic Locking

Prevent lost updates without holding database locks:

```sql
-- Read with version
SELECT id, name, email, version FROM users WHERE id = 42;
-- Returns: { id: 42, name: "Alice", email: "alice@example.com", version: 7 }

-- Update with version check
UPDATE users
SET name = 'Alice Smith', version = version + 1
WHERE id = 42 AND version = 7;

-- Check affected rows
-- If 0 rows affected → someone else modified the row → return 409 Conflict
-- If 1 row affected → success
```

Application-side:
```
function update_user(id, changes, expected_version):
  rows_affected = db.execute(
    "UPDATE users SET name=$1, version=version+1 WHERE id=$2 AND version=$3",
    [changes.name, id, expected_version]
  )
  if rows_affected == 0:
    raise ConflictError("User was modified by another request")
```

Use optimistic locking for entities with low contention. For high-contention resources (inventory counters), consider `SELECT ... FOR UPDATE` or atomic operations (`UPDATE SET quantity = quantity - 1 WHERE quantity > 0`).

## Query Optimization

### EXPLAIN ANALYZE

Run on every slow query. Read the output bottom-up (innermost operation first).

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2024-01-01'
GROUP BY u.name;
```

Red flags in EXPLAIN output:
- **Seq Scan** on large tables — missing index
- **Nested Loop** with high row counts — consider Hash Join (might need more `work_mem`)
- **Sort** with `external merge` — `work_mem` too low for in-memory sort
- **Rows** estimate wildly different from actual — stale statistics, run `ANALYZE`
- **Buffers: shared read** high — data not in cache, may indicate missing index or too-broad scan

### Index Types

| Type | Use Case | Example |
|------|----------|---------|
| B-tree | Default. Equality, range, sorting, prefix LIKE | `CREATE INDEX idx_users_email ON users (email)` |
| Hash | Equality only (rare, B-tree usually better) | `CREATE INDEX idx_users_id_hash ON users USING hash (id)` |
| GIN | Full-text search, JSONB containment, arrays | `CREATE INDEX idx_docs_body ON docs USING gin (to_tsvector('english', body))` |
| GiST | Geometric, range types, nearest-neighbor | `CREATE INDEX idx_locations_geo ON locations USING gist (coordinates)` |
| Partial | Index a subset of rows | `CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending'` |
| Composite | Multi-column queries | `CREATE INDEX idx_orders_user_date ON orders (user_id, created_at DESC)` |

Composite index column order matters: put equality columns first, range/sort columns last. `(user_id, created_at DESC)` supports `WHERE user_id = ? ORDER BY created_at DESC` but not `WHERE created_at > ? ORDER BY user_id`.

### N+1 Query Detection

The ORM loads a list, then issues one query per item for related data:

```
# N+1 problem — 1 query + N queries
users = db.query("SELECT * FROM users LIMIT 100")
for user in users:
  orders = db.query("SELECT * FROM orders WHERE user_id = ?", user.id)  # 100 queries
```

Fix: eager load or join:
```
# Single query with JOIN
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id

# Or batch: IN clause
SELECT * FROM orders WHERE user_id IN (1, 2, 3, ..., 100)
```

Enable query logging in development. Flag any endpoint issuing more than ~10 queries per request.

## Schema Design Guidelines

- **Normalize by default**: third normal form (3NF). Denormalize only when read performance demands it and you accept the write complexity.
- **Primary keys**: prefer UUIDs (v7 for time-sortability) for distributed systems. Auto-increment is fine for single-database setups. Never expose sequential IDs externally — they leak information (total count, creation rate).
- **Timestamps**: always UTC, store as `TIMESTAMPTZ` (not `TIMESTAMP`). Include `created_at` and `updated_at` on every table. Set `updated_at` via trigger or application code.
- **Soft deletes**: add `deleted_at TIMESTAMPTZ NULL` instead of deleting rows. Trade-offs: simpler audit trail and undo, but every query needs `WHERE deleted_at IS NULL`, indexes grow, and GDPR hard-delete requirements still apply. Use soft deletes for user-facing data; hard delete for internal/transient data.
- **Enums**: use a `VARCHAR` check constraint or a lookup table — not database-native ENUMs. Adding values to native ENUMs requires `ALTER TYPE` which can be painful in migrations.
- **Foreign keys**: always add them. They enforce data integrity at the database level. The minor write overhead is worth the protection against orphaned records.
- **Column naming**: `snake_case`, descriptive. `user_id` not `uid`. `created_at` not `createdAt`. Boolean columns: `is_active`, `has_verified_email`.