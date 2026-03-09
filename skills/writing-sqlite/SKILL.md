---
name: writing-sqlite
description: SQLite best practices. Use when writing, reviewing, or modifying SQL targeting SQLite databases — schema definitions, queries, pragmas, type affinity, date handling, transactions, migrations, or embedded/file-based SQLite configurations.
---

# SQLite Best Practices

## Contents

- [Common Extensions](#common-extensions)
- [Type Affinity](#type-affinity)
- [Naming](#naming)
- [Constraints](#constraints)
- [Connection Pragmas](#connection-pragmas)
- [Concurrency](#concurrency)
- [Indexes](#indexes)
- [Query Style](#query-style)
- [Transactions](#transactions)
- [STRICT Tables](#strict-tables)
- [WITHOUT ROWID](#without-rowid)
- [Generated Columns](#generated-columns)
- [RETURNING](#returning)
- [Backup](#backup)
- [Vacuum and Maintenance](#vacuum-and-maintenance)
- [Anti-Patterns](#anti-patterns)
- [New Schema Checklist](#new-schema-checklist)

## Common Extensions

SQLite is modular — many features are compile-time extensions. Most distributions include these by default, but verify with `PRAGMA compile_options`:

| Extension | Purpose |
|-----------|---------|
| `json1` | JSON functions (`json_extract`, `json_each`, `json_group_array`) |
| `fts5` | Full-text search with ranking and boolean queries |
| `rtree` | Spatial indexing for range and bounding-box queries |
| `math` | `ceil()`, `floor()`, `log()`, `pow()`, `sqrt()` (3.35.0+) |

## Type Affinity

SQLite determines storage type from the column's declared type — it's a hint, not a strict constraint. Declare types explicitly for clarity.

| Use | Declared type | Notes |
|-----|---------------|-------|
| Integer PK | `integer` | Must be the full word `INTEGER` (any case); abbreviations like `INT` or `BIGINT` don't create a rowid alias |
| Text | `text` | |
| Floating point | `real` | |
| Boolean | `integer` | Store as 0/1; add `check (col in (0, 1))` |
| Timestamps | `text` | ISO 8601 UTC: `'2024-01-15T10:30:00Z'` |
| UUID | `text` | Store as string |
| Money | `integer` | Store in smallest currency unit (cents); `real` loses precision |
| Binary | `blob` | |

`integer primary key` (the full word `INTEGER` in any case — not `INT` or `BIGINT`) makes the column an alias for `rowid` — efficient auto-increment. Don't add the `autoincrement` keyword unless you specifically need strict monotonicity guarantees; it's slower and rarely necessary.

## Naming

- Tables: **singular**, lowercase, snake_case — `user`, `order_item`.
- Columns: lowercase, snake_case. Avoid reserved words.
- Primary keys: `id` (always use the full word `integer`, not `int`).
- Foreign keys: `<referenced_table>_id`.
- Indexes: `idx_<table>_<columns>`.
- Booleans: positive names — `is_active`, `has_discount`.

## Constraints

```sql
create table order_item (
    id          integer primary key,
    order_id    integer not null references "order"(id) on delete cascade,  -- cascade: delete items when order is deleted
    product_id  integer not null references product(id) on delete restrict, -- restrict: prevent deleting a product with order items
    quantity    integer not null check (quantity > 0),
    unit_price  integer not null check (unit_price >= 0),  -- cents
    created_at  text    not null default (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);
```

Foreign keys are **disabled by default** — enable per connection:

```sql
pragma foreign_keys = on;
```

## Connection Pragmas

Set these on every new connection before running any queries:

```sql
pragma journal_mode = wal;      -- concurrent readers during a write
pragma synchronous  = normal;   -- safe with WAL, faster than 'full'
pragma foreign_keys = on;       -- required — disabled by default
pragma busy_timeout = 5000;     -- wait up to 5s instead of SQLITE_BUSY error
pragma cache_size   = -65536;   -- 64MB page cache (negative value = kibibytes)
pragma optimize     = 0x10002;  -- analyze all tables on connect (3.46.0+); on older versions use plain `pragma optimize` on connection close instead
```

WAL mode allows readers and one writer to operate concurrently. Without it, any write blocks all readers.

## Concurrency

SQLite allows **one writer at a time** — all other writers wait (up to `busy_timeout`). Multiple readers run concurrently in WAL mode.

For multi-threaded applications, use separate connections for reads and writes:

- **Write connection**: single long-lived connection, serialized access (mutex or channel).
- **Read connections**: pool of connections (typically 2–4), each with the same pragmas.

All connections must set the connection pragmas above independently — pragmas are per-connection, not per-database (except `journal_mode` which persists).

If you get `SQLITE_BUSY` despite `busy_timeout`, the likely cause is a long-running read transaction holding a WAL checkpoint. Keep read transactions short or use `pragma wal_autocheckpoint` to tune checkpoint frequency.

## Indexes

```sql
-- FK index (add for every FK column)
create index idx_order_item_order_id on order_item(order_id);

-- Composite (column order matches query filter or ORDER BY)
create index idx_order_status_created on "order"(status, created_at desc);

-- Partial index (supported since 3.8.9, 2014)
create index idx_order_pending on "order"(created_at) where status = 'pending';
```

No `include` clause — achieve covering scans by listing extra columns directly in the index key. Every index slows writes — measure with `explain query plan` before adding:

```sql
explain query plan
select id, title from "order" where status = 'pending' and created_at > ?;
```

## Query Style

Lowercase keywords, one clause per line. Parameterized with `?` (positional) or `?1`, `?2` (numbered):

```sql
select
    u.id,
    u.email,
    count(o.id) as order_count
from "user" u
left join "order" o on o.user_id = u.id
where u.created_at >= ?
  and u.is_active = 1
group by u.id, u.email
order by order_count desc
limit 50;
```

Named parameters also work: `:email`, `@name`, `$value`.

Never concatenate user input into SQL:

```sql
-- Good
select * from "user" where email = ?;

-- Never
select * from "user" where email = '" + input + "';  -- SQL injection
```

## Transactions

Use `begin immediate` for write transactions — it acquires the write lock upfront and avoids `SQLITE_BUSY` errors from lock upgrade mid-transaction:

```sql
begin immediate;
    update account set balance = balance - 100 where id = ?;
    update account set balance = balance + 100 where id = ?;
    insert into transfer_log (from_id, to_id, amount) values (?, ?, 100);
commit;
```

Use `begin deferred` (the default `begin`) for read-only transactions.

Use `savepoint` for nested rollback: `rollback to savepoint <name>` on error, `release savepoint <name>` on success.

## STRICT Tables

Available since SQLite 3.37.0 (2021). `STRICT` tables enforce actual column types rather than relying on affinity.

```sql
create table product (
    id    integer primary key,
    sku   text    not null unique,
    name  text    not null,
    price integer not null check (price >= 0)   -- cents
) strict;
```

Allowed types in a `STRICT` table: `integer`, `real`, `text`, `blob`, `any`. Inserting a value of the wrong type raises an error instead of silently coercing it.

Use `STRICT` when correctness matters more than flexibility. Mix with `WITHOUT ROWID` if needed: `create table t (...) strict, without rowid`.

## WITHOUT ROWID

Use for tables with non-integer or composite primary keys and small rows (under ~200 bytes for 4KiB pages). Avoids the wasted rowid overhead.

```sql
create table tag_assignment (
    tag_id  integer not null references tag(id),
    item_id integer not null references item(id),
    primary key (tag_id, item_id)
) without rowid;
```

Do **not** use `WITHOUT ROWID` with a single `integer primary key` — regular rowid tables are faster for that case.

## Generated Columns

Available since SQLite 3.31.0 (2020). Useful for indexing computed values or JSON fields. The `->>` operator used below requires 3.38.0+; use `json_extract()` for broader compatibility.

```sql
create table event (
    id      integer primary key,
    payload text not null check (json_valid(payload)),
    -- virtual: computed on read, not stored on disk
    user_id integer generated always as (payload ->> 'user_id') virtual,
    -- stored: computed on write, stored on disk (indexable)
    kind    text    generated always as (payload ->> 'kind') stored
);

create index idx_event_user_id on event(user_id);
create index idx_event_kind on event(kind);
```

`VIRTUAL` columns save disk space but recompute on every read. `STORED` columns use disk space but are faster to query and can be indexed efficiently.

## RETURNING

Available since SQLite 3.35.0 (2021). Get column values back from any `insert`, `update`, or `delete` without a second query:

```sql
insert into product (sku, name, price) values (?, ?, ?)
returning id, created_at;

update "order" set status = 'shipped' where id = ?
returning id, status;

delete from session where expires_at < strftime('%Y-%m-%dT%H:%M:%SZ', 'now')
returning id;
```

## Backup

Use `VACUUM INTO` (3.27.0+) for hot backups — it creates a consistent snapshot without blocking other connections:

```sql
vacuum into '/path/to/backup.db';
```

Never copy the `.db` file directly while connections are open — it risks corruption, especially in WAL mode (the `-wal` and `-shm` files must be consistent with the main file).

For programmatic backups, use the [Online Backup API](https://www.sqlite.org/backup.html) which allows incremental page-level copies.

## Vacuum and Maintenance

SQLite doesn't reclaim disk space after deletes — the file keeps its high-water-mark size.

```sql
-- Reclaim unused space (rewrites the entire database file; blocks all access)
vacuum;

-- Auto-vacuum: reclaim space incrementally (must be set before creating tables)
pragma auto_vacuum = incremental;   -- or 'full' for automatic
pragma incremental_vacuum(100);     -- reclaim up to 100 pages

-- Verify database integrity
pragma integrity_check;

-- Quick check (faster, less thorough)
pragma quick_check;
```

`vacuum` requires up to 2x the database size in free disk space. For large databases, prefer `auto_vacuum = incremental` with periodic `incremental_vacuum` calls.

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Missing `pragma foreign_keys = on` | Set on every connection open |
| `real` for money | `integer` in cents (divide by 100 for display) |
| Epoch integers for timestamps | ISO 8601 UTC text strings |
| `autoincrement` by default | Omit unless strict monotonicity is required |
| Bare `begin` for writes | `begin immediate` to avoid lock contention |
| `select *` in application code | Name columns explicitly |
| String-concatenated queries | Parameterized `?` placeholders |
| Missing indexes on FK columns | Add an index for every FK column |
| JSON in `text` without validation | `check (json_valid(col))` constraint |
| Single shared connection for reads+writes | Separate write connection from read pool |
| Never running `vacuum` | Schedule periodic `vacuum` or use `auto_vacuum = incremental` |
| Copying `.db` file while connections are open | Use `vacuum into` or the backup API |
| SQLite over network filesystem (NFS/SMB) | Always use a local file; network FS causes corruption |
| `INT PRIMARY KEY` instead of `INTEGER PRIMARY KEY` | Must use the full word `INTEGER` for rowid alias |

## New Schema Checklist

```
- [ ] All columns: explicit types and NOT NULL / nullable decision
- [ ] integer primary key (full word INTEGER, not INT) for auto-increment PK
- [ ] Foreign keys declared with ON DELETE behavior
- [ ] CHECK constraints for domain rules (json_valid for JSON columns)
- [ ] Index on every FK column
- [ ] pragma foreign_keys = on in connection setup
- [ ] pragma journal_mode = wal in connection setup
- [ ] Timestamps as ISO 8601 UTC text
- [ ] Money stored as integer (cents), not real
- [ ] STRICT table considered
- [ ] WITHOUT ROWID considered for composite/text PKs with small rows
- [ ] Backup strategy (VACUUM INTO or backup API)
- [ ] Run explain query plan on key queries
```

## Deep-dive References

- **Query patterns**: [patterns/query-patterns.md](patterns/query-patterns.md) — upserts, pagination, date/time, CTEs, JSON queries, window functions, FTS5
- **Schema design**: [patterns/schema-design.md](patterns/schema-design.md) — audit triggers, soft delete, multi-tenant enforcement, adjacency list
- **Migrations**: [patterns/migrations.md](patterns/migrations.md) — limited ALTER TABLE, table recreation pattern, version-gated features

## Official References

- [SQLite docs](https://www.sqlite.org/docs.html)
- [SQLite type affinity](https://www.sqlite.org/datatype3.html)
- [SQLite WAL mode](https://www.sqlite.org/wal.html)
- [SQLite FTS5](https://www.sqlite.org/fts5.html)
- [SQLite JSON](https://www.sqlite.org/json1.html)
- [SQLite query planner](https://www.sqlite.org/eqp.html)
