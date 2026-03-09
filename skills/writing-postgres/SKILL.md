---
name: writing-postgres
description: PostgreSQL best practices. Use when writing, reviewing, or modifying SQL targeting PostgreSQL databases — schema definitions, queries, migrations, indexes, CTEs, window functions, JSONB, full-text search, row-level security, or any PostgreSQL-specific SQL.
---

# PostgreSQL Best Practices

Targets PostgreSQL 16–18. Avoid legacy patterns (`serial`, bare `timestamp`, singular reserved-word table names).

## Naming

- Tables: lowercase, snake_case. Pick **plural** or singular and be consistent — plural is a common default (`users`, `order_items`) and avoids reserved-word collisions (`user`, `order`, `group`).
- Columns: lowercase, snake_case. Avoid reserved words — prefer `user_name`, `config_value`, `item_type`.
- Primary keys: `id` for simple tables, `<table>_id` in junction tables.
- Foreign keys: `<referenced_table>_id` (`user_id` references `users.id`).
- Indexes: `idx_<table>_<columns>`. Unique constraints: `uq_<table>_<columns>`.
- Booleans: positive names — `is_active`, `has_discount`. Never `not_deleted`.

## Data Types

| Use | Type | Notes |
|-----|------|-------|
| Surrogate PK | `bigint generated always as identity` | Prefer over deprecated `serial`; `always` prevents accidental sequence drift. Use `by default` only when explicit inserts are needed (migrations, seeding) |
| UUID v4 PK | `uuid` | `gen_random_uuid()` — random, built-in since PG 13 |
| UUID v7 PK | `uuid` | `uuidv7()` — time-ordered, better index locality, built-in since PG 18 |
| Text | `text` | No practical size limit; use `check` for length |
| Money | `numeric(19,4)` | Never `float` or `double precision` |
| Timestamps | `timestamptz` | Always — never bare `timestamp` (ignores session timezone) |
| JSON | `jsonb` | Binary, indexable; `json` only to preserve raw input |
| Boolean | `boolean` | |
| Arrays | `text[]`, `int[]` | Prefer join table for relational data |

## Constraints

Always enforce constraints at the database level. Declare `on delete` behavior on every FK explicitly.

```sql
create table order_items (
    id          bigint generated always as identity primary key,
    order_id    bigint        not null references orders(id) on delete cascade,
    product_id  bigint        not null references products(id) on delete restrict,
    quantity    int           not null check (quantity > 0),
    unit_price  numeric(19,4) not null check (unit_price >= 0),
    created_at  timestamptz   not null default now()
);
```

## Indexes

```sql
-- FK index (add for every FK column)
create index idx_order_items_order_id on order_items(order_id);

-- Composite (column order matches query filter or ORDER BY)
create index idx_orders_status_created on orders(status, created_at desc);

-- Partial — only index relevant rows
create index idx_orders_pending on orders(created_at) where status = 'pending';

-- Covering — avoids heap fetch
create index idx_products_sku on products(sku) include (name, price);
```

Every index slows writes. Measure with `explain analyze` before adding.

For production tables: `create index concurrently …` to avoid blocking writes (cannot run inside a transaction block).

Set timeouts before DDL on production tables:

```sql
set lock_timeout = '5s';        -- fail fast if lock can't be acquired
set statement_timeout = '30s';  -- cap long-running backfills/validates
```

## Query Style

One clause per line. Parameterized with `$1`, `$2`, … Keyword casing is a team convention — these examples use lowercase; uppercase (`SELECT`, `FROM`) is equally common.

```sql
select
    u.id,
    u.email,
    count(o.id)  as order_count,
    sum(o.total) as lifetime_value
from users u
left join orders o on o.user_id = u.id
where u.created_at >= $1
  and u.is_active = true
group by u.id, u.email
having count(o.id) > 0
order by lifetime_value desc
limit $2;
```

Never concatenate user input into SQL:

```sql
-- Good
select * from users where email = $1;

-- Never
select * from users where email = '" + input + "';  -- SQL injection
```

## CTEs

Use CTEs to name intermediate steps. Prefer over nested subqueries for readability.

Since PG 12, simple non-recursive CTEs referenced once are inlined automatically. Use `not materialized` to force inlining for CTEs referenced multiple times; `materialized` to force caching:

```sql
with
active_users as not materialized (
    select id, email from users where is_active = true
),
recent_orders as not materialized (
    select user_id, count(*) as cnt
    from orders
    where created_at >= now() - interval '30 days'
    group by user_id
)
select
    u.email,
    coalesce(o.cnt, 0) as orders_last_30d
from active_users u
left join recent_orders o on o.user_id = u.id;
```

## Transactions

Wrap multi-statement mutations in explicit transactions. Autocommit leaves partial writes on failure.

```sql
begin;
    update accounts set balance = balance - 100 where id = $1;
    update accounts set balance = balance + 100 where id = $2;
    insert into transfer_logs (from_id, to_id, amount) values ($1, $2, 100);
commit;
```

**Isolation levels** (use the minimum needed):

| Level | Prevents | Use for |
|-------|----------|---------|
| `read committed` (default) | Dirty reads | Most OLTP operations |
| `repeatable read` | Non-repeatable reads | Consistent reports |
| `serializable` | Phantom reads | Financial operations |

Use `savepoint` for nested rollback: `rollback to savepoint <name>` on error, `release savepoint <name>` on success.

### Row-level locking

PostgreSQL has four lock modes (weakest → strongest):

| Mode | Blocks | Use when |
|------|--------|----------|
| `for key share` | deletes, key updates | FK checks (auto) |
| `for share` | any modification | read-and-check patterns |
| `for no key update` | deletes, key updates | updating non-key columns |
| `for update` | deletes, any update, FK inserts | about to DELETE or change PK/unique |

**The trap**: `for update` also blocks inserts into tables that have a FK referencing the locked row. Use `for no key update` for any update that doesn't touch the primary key — this is what `UPDATE` itself acquires internally.

```sql
-- Bad: blocks FK inserts (e.g. inserting a transaction for this account)
select * from accounts where id = $1 for update;

-- Good: allows FK inserts while still preventing conflicting updates
select * from accounts where id = $1 for no key update;
```

Use `skip locked` for queue workers — each worker atomically claims rows no other worker is processing:

```sql
begin;
    select id, payload
    from jobs
    where status = 'pending'
    order by created_at
    limit 10
    for no key update skip locked;

    update jobs set status = 'processing' where id = any($1);
commit;
```

Use `nowait` to fail immediately instead of waiting when a row is already locked:

```sql
select * from resources where id = $1 for no key update nowait;
```

## Upserts

```sql
insert into products (sku, name, price)
values ($1, $2, $3)
on conflict (sku) do update set
    name       = excluded.name,
    price      = excluded.price,
    updated_at = now();
```

Use `on conflict (col) do nothing` to silently skip duplicates.

Use `returning` to get column values back from any `insert`, `update`, or `delete` without a second query:

```sql
insert into products (sku, name, price)
values ($1, $2, $3)
returning id, created_at;

update orders set status = 'shipped' where id = $1
returning id, status, updated_at;

delete from sessions where expires_at < now()
returning id;
```

`returning` also exposes old and new values in a single statement:

```sql
-- PG 18+
update products set price = $1 where sku = $2
returning sku, old.price as old_price, new.price as new_price;
```

## Pagination

Keyset (cursor) pagination is faster than `offset` for large datasets — `offset N` scans and discards N rows.

```sql
-- Keyset: next page after last seen (created_at, id)
select id, name, created_at
from products
where (created_at, id) < ($1, $2)
order by created_at desc, id desc
limit $3;

-- OFFSET: small datasets or admin UIs only
select id, name from products order by id limit $1 offset $2;
```

## JSONB

Use `jsonb` when the document structure varies per row or you need to query inside it. Don't use it to avoid normalizing fields you'll always query by name — model those as columns.

```sql
-- GIN index enables containment (@>) and key existence (?) queries
create index idx_events_payload on events using gin(payload);

-- Containment query
select id, type from events where payload @> '{"user_id": 42}';

-- Extract as text
select payload->>'email' from events where type = 'user.created';
```

See [patterns/schema-design.md](patterns/schema-design.md) for JSONB column design, `jsonb_set()` updates (note: rewrites entire TOAST tuple), and GIN indexing. See [patterns/query-patterns.md](patterns/query-patterns.md) for SQL/JSON path (PG 12+), JSON_TABLE (PG 17+), and JSON_VALUE/JSON_QUERY.

## Row Level Security

Always `force row level security` — without it, RLS is silently bypassed when the app connects as table owner. Use `set local` to set tenant context per transaction (requires transaction-mode connection pooling).

See [patterns/schema-design.md](patterns/schema-design.md) for full RLS setup, multi-tenant patterns (shared schema vs separate schemas), and pooling caveats.

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| `select *` in application code | Name columns explicitly |
| `float`/`double precision` for money | `numeric(19,4)` |
| Bare `timestamp` | `timestamptz` |
| String-concatenated queries | Parameterized `$1`, `$2` |
| `not in (subquery)` with nullable column | Use `not exists` |
| `offset` pagination on large tables | Keyset pagination |
| `or` on separately indexed columns | `union all` |
| Index on every column | Index selectively; measure with `explain analyze` |
| Missing `not null` on required columns | Constrain at schema level |
| `for update` when not changing PK/unique | `for no key update` — avoids blocking FK inserts |
| `serial` / `bigserial` | `bigint generated always as identity` |
| `order by random()` on large tables | `tablesample bernoulli(1)` or application-level sampling |

## New Schema Checklist

```
- [ ] All columns: explicit types and NOT NULL / nullable decision
- [ ] Primary key: bigint generated always as identity
- [ ] Foreign keys declared with explicit ON DELETE behavior
- [ ] CHECK constraints for domain rules (quantity > 0, status in (...))
- [ ] Indexes on every FK column and common filter/sort columns
- [ ] timestamptz for all timestamps
- [ ] No float/double for money
- [ ] Reviewed with EXPLAIN ANALYZE after inserting representative data
```

## Deep-dive References

- **Schema design**: [patterns/schema-design.md](patterns/schema-design.md) — audit tables, soft delete, multi-tenant/RLS, JSONB, enum types, exclusion constraints, adjacency list
- **Query patterns**: [patterns/query-patterns.md](patterns/query-patterns.md) — window functions, recursive CTEs, lateral joins, array aggregation, full-text search, SQL/JSON, pivot
- **Migrations**: [patterns/migrations.md](patterns/migrations.md) — zero-downtime migrations, concurrent index creation, constraint validation
- **Operations**: [patterns/operations.md](patterns/operations.md) — partitioning, vacuum/autovacuum, connection pooling, COPY, EXPLAIN ANALYZE, pg_stat_statements

## Official References

- [PostgreSQL 18 docs](https://www.postgresql.org/docs/18/)
- [Use the Index, Luke](https://use-the-index-luke.com/) — index tuning guide
- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html) — query performance monitoring
