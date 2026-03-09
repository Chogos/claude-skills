# Schema Design Patterns

## Contents

- Audit columns
- Soft delete
- Multi-tenant (row-level isolation)
- JSONB columns
- Enum types
- Exclusion constraints
- Self-referential (adjacency list)

---

## Audit Columns

Add to every mutable table. Use a shared trigger function to auto-set `updated_at`.

```sql
create or replace function set_updated_at()
returns trigger language plpgsql as $$
begin
    new.updated_at = now();
    return new;
end;
$$;

create table products (
    id          bigint generated always as identity primary key,
    sku         text          not null unique,
    name        text          not null,
    price       numeric(19,4) not null,
    created_at  timestamptz   not null default now(),
    updated_at  timestamptz   not null default now()
);

create trigger products_set_updated_at
before update on products
for each row execute function set_updated_at();
```

One trigger function shared across all tables — only the trigger itself is table-specific.

---

## Soft Delete

Prefer hard delete unless audit history or FK integrity requires keeping rows. Soft delete adds complexity everywhere.

When soft delete is necessary:

```sql
alter table users add column deleted_at timestamptz;

-- Filter active rows with a view (name columns explicitly — select * breaks on schema changes)
create view active_users as
select id, email, name, created_at, updated_at
from users
where deleted_at is null;

-- Partial index so active-row queries stay fast
create index idx_users_active on users(id) where deleted_at is null;

-- Delete: set the timestamp, don't remove the row
update users set deleted_at = now() where id = $1;
```

**Rules:**

- All application queries go through the view, not the base table.
- Unique constraints need partial indexes to apply only to active rows:

  ```sql
  create unique index uq_users_email_active
      on users(email) where deleted_at is null;
  ```

- Consider a separate `deleted_<table>` archive table instead if rows accumulate unboundedly.
- **FK cascade trap**: if a parent table has `on delete cascade` and a child uses soft delete, the cascade will hard-delete child rows, bypassing the soft delete pattern. Use `on delete restrict` or `on delete set null` on FKs pointing to soft-deletable parents, and handle cascading deletes in application code.

---

## Multi-Tenant (Row-Level Isolation)

Two patterns: shared schema with `tenant_id` column, or separate schemas per tenant.

### Shared schema with tenant_id

```sql
create table documents (
    id          bigint generated always as identity primary key,
    tenant_id   bigint      not null references tenants(id),
    title       text        not null,
    body        text        not null,
    created_at  timestamptz not null default now()
);

create index idx_documents_tenant on documents(tenant_id);

alter table documents enable row level security;
-- Required if app connects as table owner — without this, RLS is silently bypassed
alter table documents force row level security;

create policy tenant_isolation on documents
    using     (tenant_id = pg_catalog.current_setting('app.tenant_id')::bigint)
    with check (tenant_id = pg_catalog.current_setting('app.tenant_id')::bigint);

-- Set per transaction from the application layer
set local app.tenant_id = '42';
```

`set local` only works correctly with transaction-mode connection pooling. Each transaction must set the tenant context.

### Separate schemas per tenant

```sql
create schema tenant_42;
create table tenant_42.documents ( ... );

-- Set search_path per connection
set search_path to tenant_42, public;
```

Use separate schemas when tenants need different table structures or strict data isolation. Harder to manage at scale — migrations must run once per tenant schema.

---

## JSONB Columns

Use `jsonb` when:

- The document structure varies per row.
- You need to query inside the document.
- The shape is unknown at schema design time.

Don't use `jsonb` to avoid normalizing data you'll always query by specific fields — model those as columns.

```sql
create table events (
    id         bigint generated always as identity primary key,
    type       text        not null,
    payload    jsonb       not null default '{}',
    created_at timestamptz not null default now()
);

-- GIN index enables containment (@>) and key existence (?) queries
create index idx_events_payload on events using gin(payload);

-- Query: events where payload contains a specific user_id
select id, type, payload
from events
where payload @> '{"user_id": 42}';

-- Extract a field
select payload->>'email' as email from events where type = 'user.created';

-- Update a field without replacing the whole document
-- Caveat: any jsonb update rewrites the entire TOAST tuple (no partial updates)
-- For large documents with frequent partial updates, consider normalizing hot fields into columns
update events
set payload = jsonb_set(payload, '{status}', '"processed"')
where id = $1;
```

---

## Enum Types

Use `create type` enums for closed, stable value sets. Use a lookup table for open or frequently changing sets.

```sql
create type order_status as enum ('pending', 'processing', 'shipped', 'cancelled');

create table orders (
    id         bigint generated always as identity primary key,
    status     order_status not null default 'pending',
    created_at timestamptz  not null default now()
);

-- Add a value (safe, no table rewrite)
alter type order_status add value 'returned' after 'shipped';

-- Rename a value
alter type order_status rename value 'cancelled' to 'canceled';
```

**Limitations:**

- `ADD VALUE` inside a transaction: the new value cannot be used until after the transaction commits.
- Cannot remove values without recreating the type.
- Enum ordering is insertion order, not alphabetical — affects `ORDER BY`.

For frequently changing sets, prefer a lookup table:

```sql
create table order_statuses (
    code        text primary key,
    description text not null
);

insert into order_statuses values
    ('pending',    'Awaiting processing'),
    ('processing', 'Being fulfilled'),
    ('shipped',    'In transit'),
    ('cancelled',  'Cancelled by user');

alter table orders
    add column status text not null default 'pending'
        references order_statuses(code);
```

---

## Exclusion Constraints

Prevent overlapping ranges at the database level — essential for scheduling, booking, and reservation systems.

```sql
create extension if not exists btree_gist;

create table bookings (
    id          bigint generated always as identity primary key,
    room_id     bigint      not null references rooms(id),
    during      tstzrange   not null,
    created_at  timestamptz not null default now(),

    -- No two bookings for the same room can overlap
    exclude using gist (room_id with =, during with &&)
);

-- Insert succeeds
insert into bookings (room_id, during)
values (1, tstzrange('2026-03-10 14:00', '2026-03-10 16:00'));

-- Overlapping insert fails with exclusion violation
insert into bookings (room_id, during)
values (1, tstzrange('2026-03-10 15:00', '2026-03-10 17:00'));
```

Range types: `tstzrange` (timestamps with timezone), `daterange` (dates), `int4range` / `int8range` (integers). Use `[)` bounds convention (lower inclusive, upper exclusive) for clean adjacency.

---

## Self-Referential (Adjacency List)

For trees and hierarchies (categories, org charts, comment threads).

```sql
create table categories (
    id         bigint generated always as identity primary key,
    parent_id  bigint references categories(id) on delete restrict,
    name       text   not null,
    sort_order int    not null default 0
);

create index idx_categories_parent on categories(parent_id);

-- Recursive CTE: fetch entire subtree under a node
with recursive subtree as (
    select id, parent_id, name, 0 as depth
    from categories
    where id = $1

    union all

    select c.id, c.parent_id, c.name, s.depth + 1
    from categories c
    join subtree s on c.parent_id = s.id
    where s.depth < 10  -- guard against runaway on deep/cyclic graphs
)
select * from subtree order by depth, sort_order;
```

For deep trees with frequent ancestry queries, consider the `ltree` extension or the nested set model.
