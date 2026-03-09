# Schema Design Patterns

## Contents

- Audit columns
- Soft delete
- Multi-tenant (application-layer enforcement)
- JSON columns
- Self-referential (adjacency list)

---

## Audit Columns

SQLite requires a per-table trigger for `updated_at` — there is no shared trigger function.

```sql
create table product (
    id         integer primary key,
    sku        text    not null unique,
    name       text    not null,
    price      integer not null,                                    -- cents
    created_at text    not null default (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
    updated_at text    not null default (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

create trigger product_set_updated_at
after update on product
for each row
when old.updated_at = new.updated_at   -- skip if caller already set updated_at
begin
    update product
    set updated_at = strftime('%Y-%m-%dT%H:%M:%SZ', 'now')
    where id = old.id;
end;
```

The trigger fires after `update` and writes back to the same row. The `WHEN` clause avoids a redundant second write when the caller explicitly sets `updated_at`. Safe from infinite recursion because `pragma recursive_triggers` defaults to OFF.

---

## Soft Delete

Prefer hard delete unless audit history or FK integrity requires keeping rows.

```sql
alter table "user" add column deleted_at text;   -- nullable; NULL = active

-- Filter active rows with a view (name columns explicitly)
create view active_user as
select id, email, name, created_at, updated_at
from "user"
where deleted_at is null;

-- Partial index so active-row queries stay fast (supported since 3.8.9)
create index idx_user_active on "user"(id) where deleted_at is null;

-- Delete: set the timestamp
update "user"
set deleted_at = strftime('%Y-%m-%dT%H:%M:%SZ', 'now')
where id = ?;
```

**Rules:**

- All application queries go through the view.
- Enforce unique constraints for active rows only with a partial unique index:

  ```sql
  create unique index uq_user_email_active
      on "user"(email) where deleted_at is null;
  ```

---

## Multi-Tenant (Application-Layer Enforcement)

SQLite has no Row Level Security. Enforce tenant isolation entirely in the application layer by always including `tenant_id` in every query.

```sql
create table document (
    id         integer primary key,
    tenant_id  integer not null references tenant(id),
    title      text    not null,
    body       text    not null,
    created_at text    not null default (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

create index idx_document_tenant on document(tenant_id);

-- Every query must filter by tenant_id
select id, title from document where tenant_id = ? and id = ?;
insert into document (tenant_id, title, body) values (?, ?, ?);
update document set title = ? where tenant_id = ? and id = ?;
delete from document where tenant_id = ? and id = ?;
```

Wrap queries in a repository/DAO layer so `tenant_id` cannot be omitted by accident.

---

## JSON Columns

Use a `text` column for JSON. SQLite stores it as plain text; `json_extract()` and `->>` (3.38.0+) query into it.

```sql
create table event (
    id         integer primary key,
    type       text    not null,
    payload    text    not null default '{}' check (json_valid(payload)),
    created_at text    not null default (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

-- Extract fields
select payload ->> 'user_id'  as user_id  from event where type = 'order.created';
select payload ->> 'amount'   as amount   from event where type = 'payment.received';

-- Filter by JSON field (full table scan — no JSON index)
select id, type from event where payload ->> 'status' = 'pending';

-- Update a field (replace whole document — no in-place patch)
update event
set payload = json_set(payload, '$.status', 'processed')
where id = ?;

-- Build JSON for API responses
select json_object('id', id, 'type', type, 'payload', json(payload)) from event;
```

**When to use JSON columns:**

- Document structure varies per row.
- Shape is unknown at schema design time.

**Don't use JSON columns** to avoid normalizing fields you'll always query by name — model those as columns.

**Indexing limitation:** SQLite has no GIN equivalent. JSON field queries do full table scans. For frequently-filtered fields, extract them into a real column and index that instead:

```sql
-- Instead of filtering on payload ->> 'user_id' repeatedly:
alter table event add column user_id integer
    generated always as (payload ->> 'user_id') virtual;  -- 3.38.0+

create index idx_event_user_id on event(user_id);
```

---

## Self-Referential (Adjacency List)

For trees and hierarchies (categories, org charts, comment threads).

```sql
create table category (
    id         integer primary key,
    parent_id  integer references category(id) on delete restrict,
    name       text    not null,
    sort_order integer not null default 0
);

create index idx_category_parent on category(parent_id);

-- Ancestor path from a node up to the root
-- level 0 = self, 1 = parent, 2 = grandparent; order by level desc = root first
with recursive ancestors as (
    select id, parent_id, name, 0 as level
    from category
    where id = ?

    union all

    select c.id, c.parent_id, c.name, a.level + 1
    from category c
    join ancestors a on c.id = a.parent_id
)
select * from ancestors order by level desc;

-- Subtree: all descendants of a node
with recursive subtree as (
    select id, parent_id, name, 0 as depth
    from category
    where id = ?

    union all

    select c.id, c.parent_id, c.name, s.depth + 1
    from category c
    join subtree s on c.parent_id = s.id
    where s.depth < 10                          -- depth limit
)
select * from subtree order by depth, sort_order;
```

Recursive CTEs available since SQLite 3.8.3 (2014).
