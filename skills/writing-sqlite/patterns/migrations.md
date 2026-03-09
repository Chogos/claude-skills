# Migration Patterns

## Contents

- Migration file conventions
- Safe vs breaking changes
- Table recreation pattern
- Version-gated features

---

## Migration File Conventions

```text
migrations/
├── 0001_create_user.sql
├── 0002_create_order.sql
├── 0003_add_user_phone.sql
└── 0004_index_order_status.sql
```

- Sequential integer prefix + descriptive name.
- One logical change per file — don't bundle unrelated DDL.
- Every migration has an `up` and a `down` (rollback).
- Migrations run exactly once and are never edited after being applied.

```sql
-- 0003_add_user_phone.sql
-- up
alter table "user" add column phone text;

-- down (requires 3.35.0+; use table recreation pattern on older versions)
alter table "user" drop column phone;
```

---

## Safe vs Breaking Changes

SQLite's `alter table` support is much more limited than PostgreSQL's.

**Safe** (no table recreation needed):

| Operation | Notes |
| --------- | ----- |
| Add nullable column | `alter table t add column c type` |
| Add column with constant default | e.g., `default 0`, `default ''` |
| Rename table | `alter table t rename to t2` |
| Rename column | Since 3.25.0 (2018) |
| Drop column | Since 3.35.0 (2021); recreate table on older versions |
| Create/drop index | Safe any time |
| Create/drop trigger | Safe any time |

**Requires table recreation** (any SQLite version):

| Operation | Workaround |
| --------- | ---------- |
| Change column type | Recreate table |
| Add/remove/change a constraint | Recreate table |
| Add `not null` to existing column | Recreate table |
| Drop column (pre-3.35.0) | Recreate table |

---

## Table Recreation Pattern

For any operation that requires a schema change SQLite can't apply directly:

```sql
pragma foreign_keys = off;   -- disable FK enforcement during recreation

begin;

-- Create new table with desired schema
create table product_new (
    id    integer primary key,
    sku   text    not null unique,
    name  text    not null,
    price integer not null check (price >= 0)   -- cents; newly added constraint
);

-- Copy data
insert into product_new (id, sku, name, price)
select id, sku, name, price from product;

-- Swap
drop table product;
alter table product_new rename to product;

commit;

pragma foreign_keys = on;
```

**Checklist for table recreation:**

- Disable `pragma foreign_keys` before, re-enable after.
- Wrap in a transaction — both `drop` and `rename` are atomic.
- Recreate all indexes, triggers, and views that referenced the old table name.
- Verify row counts: `select count(*) from product` before and after.
- Test `pragma foreign_key_check;` after re-enabling foreign keys.

---

## Version-Gated Features

| Feature | Minimum version | Release date |
| ------- | --------------- | ------------ |
| `on conflict … do update` (upsert) | 3.24.0 | 2018-06-04 |
| `rename column` | 3.25.0 | 2018-09-15 |
| Recursive CTEs | 3.8.3 | 2014-02-03 |
| Window functions | 3.25.0 | 2018-09-15 |
| `returning` clause | 3.35.0 | 2021-03-12 |
| `drop column` | 3.35.0 | 2021-03-12 |
| `json_extract` / JSON functions | 3.9.0 | 2015-10-14 |
| `->>` JSON operator | 3.38.0 | 2022-02-22 |
| Partial indexes | 3.8.9 | 2015-03-09 |
| `STRICT` tables | 3.37.0 | 2021-11-27 |
| `unixepoch()` function | 3.38.0 | 2022-02-22 |
| `VACUUM INTO` | 3.27.0 | 2019-02-07 |
| Generated columns | 3.31.0 | 2020-01-22 |
| `pragma optimize = 0x10002` | 3.46.0 | 2024-05-23 |

Check the runtime version with `select sqlite_version();`. For embedded deployments, pin or document the minimum SQLite version required.
