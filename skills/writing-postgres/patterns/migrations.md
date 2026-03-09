# Migration Patterns

## Contents

- Migration file conventions
- Safe vs breaking changes
- Zero-downtime: adding columns
- Zero-downtime: renaming columns
- Zero-downtime: changing column types
- Zero-downtime: dropping columns/tables
- Concurrent index creation

---

## Migration File Conventions

```text
migrations/
├── 0001_create_users.sql
├── 0002_create_orders.sql
├── 0003_add_users_phone.sql
└── 0004_index_orders_status.sql
```

- Sequential integer prefix + descriptive name.
- One logical change per file — don't bundle unrelated DDL.
- Every migration has an `up` and a `down` (rollback). Either in the same file with markers or in separate `up/`/`down/` directories depending on the tool.
- Migrations run exactly once and are never edited after being applied to any environment.
- Set timeouts before DDL on production tables:

```sql
set lock_timeout = '5s';        -- fail fast instead of blocking queries
set statement_timeout = '30s';  -- cap long-running operations
```

```sql
-- 0003_add_users_phone.sql
-- up
alter table users add column phone text;

-- down
alter table users drop column phone;
```

---

## Safe vs Breaking Changes

**Safe** (can deploy anytime, no coordination with app deploys):

| Operation | Notes |
| --------- | ----- |
| Add nullable column | Existing rows get NULL, instant |
| Add column with non-volatile default | PG 11+: instant, no table rewrite. Works for constants (`''`, `0`, `false`), `now()`, `gen_random_uuid()`. Only VOLATILE functions (`clock_timestamp()`, `random()`) trigger a rewrite. |
| `create index concurrently` | No write lock; see below |
| Add constraint `not valid` | Validates new rows only; validate separately |
| Create new table | |
| Expand enum (`add value`) | `alter type … add value` |

**`now()` as a default**: `now()` is STABLE — instant on PG 11+, no table rewrite. All existing rows get the same timestamp (the value of `now()` at ALTER time). Only truly VOLATILE functions like `clock_timestamp()` or `random()` trigger a full table rewrite.

**Breaking** (require coordination — deploy app first or last):

| Operation | Risk |
| --------- | ---- |
| Drop column | App still referencing it will error |
| Rename column | All queries using old name break |
| Drop table | |
| Add `not null` without default | Fails if existing rows are NULL |
| Change column type | May cast incompatibly |
| Shrink text length | Existing data may exceed new limit |

---

## Zero-Downtime: Adding Columns

```sql
-- Step 1: add nullable (safe, instant)
alter table orders add column notes text;

-- Step 2 (optional): backfill in batches — run each batch as a separate statement
-- to avoid holding locks for the full table and bloating WAL.
-- Repeat until 0 rows updated:
update orders set notes = ''
where id in (
    select id from orders where notes is null
    limit 5000
    for update skip locked
);

-- Step 3 (optional): add not null after backfill — use NOT VALID check to avoid full table scan
alter table orders
    add constraint orders_notes_not_null
    check (notes is not null)
    not valid;

alter table orders validate constraint orders_notes_not_null;

-- PG 12+: SET NOT NULL is instant (skips table scan because the valid check proves no NULLs)
alter table orders alter column notes set not null;
alter table orders drop constraint orders_notes_not_null;
alter table orders alter column notes set default '';
```

---

## Zero-Downtime: Renaming Columns

Direct rename breaks running app instances. Use a two-phase approach:

**Phase 1** (before deploying new app version):

```sql
-- Add new column
alter table users add column full_name text;

-- Sync trigger: propagate whichever column changed
create or replace function sync_user_name()
returns trigger language plpgsql as $$
begin
    if tg_op = 'INSERT' then
        new.full_name    := coalesce(new.full_name,    new.display_name);
        new.display_name := coalesce(new.display_name, new.full_name);
    else  -- UPDATE: detect which column changed
        if new.full_name is distinct from old.full_name then
            new.display_name := new.full_name;
        elsif new.display_name is distinct from old.display_name then
            new.full_name := new.display_name;
        end if;
    end if;
    return new;
end;
$$;

create trigger users_sync_name
before insert or update on users
for each row execute function sync_user_name();

-- Backfill existing rows
update users set full_name = display_name where full_name is null;
```

**Phase 2** (after new app version fully deployed):

```sql
drop trigger users_sync_name on users;
drop function sync_user_name();
alter table users drop column display_name;
```

---

## Zero-Downtime: Changing Column Types

Example: `int` → `bigint`, or `varchar(50)` → `text`.

```sql
-- Step 1: add new column
alter table products add column price_new numeric(19,4);

-- Step 2: backfill existing rows
update products set price_new = price::numeric(19,4);

-- Step 3: sync old column → new column while old app is still running
create or replace function sync_product_price()
returns trigger language plpgsql as $$
begin
    if new.price is not null then
        new.price_new := new.price::numeric(19,4);
    end if;
    return new;
end;
$$;

create trigger products_sync_price
before insert or update on products
for each row execute function sync_product_price();

-- Step 4: deploy app using new column (price_new)

-- Step 5: drop old column and trigger
drop trigger products_sync_price on products;
drop function sync_product_price();
alter table products drop column price;
alter table products rename column price_new to price;
```

---

## Zero-Downtime: Dropping Columns / Tables

**Never drop in the same migration that you stop using it.**

1. Deploy app version that no longer references the column/table.
2. Wait for all instances to roll over.
3. Apply the drop migration.

```sql
-- Safe drop (only after app no longer references it)
alter table users drop column legacy_token;
```

To add a constraint on a large table without a full lock:

```sql
-- Add not valid: only validates new/updated rows
alter table orders
    add constraint orders_total_positive
    check (total >= 0)
    not valid;

-- Validate separately (ShareUpdateExclusiveLock — does not block reads or writes)
alter table orders validate constraint orders_total_positive;
```

---

## Concurrent Index Creation

Regular `create index` takes a lock that blocks writes for the duration.

```sql
-- Blocks writes (avoid on busy tables)
create index idx_orders_user_id on orders(user_id);

-- Non-blocking (takes longer, allows concurrent writes)
create index concurrently idx_orders_user_id on orders(user_id);
```

**Rules for `concurrently`:**

- Cannot run inside a transaction block — migration tools that wrap in a transaction must handle this as a separate non-transactional step.
- If it fails, it leaves an invalid index. Clean up with `drop index concurrently idx_name`.
- Use `reindex concurrently idx_name` (PG 12+) to rebuild an existing index without blocking writes — alternative to drop + recreate.
- Check for invalid indexes after deploy:

```sql
select schemaname, tablename, indexname
from pg_indexes i
join pg_class c  on c.relname = i.indexname
join pg_index ix on ix.indexrelid = c.oid
where not ix.indisvalid;
```
