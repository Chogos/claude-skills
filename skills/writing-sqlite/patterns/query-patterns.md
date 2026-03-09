# Query Patterns

## Contents

- Upserts
- Pagination
- Date and time
- CTEs
- JSON queries
- Window functions
- Full-text search (FTS5)

---

## Upserts

Available since SQLite 3.24.0 (2018):

```sql
insert into product (sku, name, price)
values (?, ?, ?)
on conflict(sku) do update set
    name  = excluded.name,
    price = excluded.price;
```

Use `on conflict(col) do nothing` to silently skip duplicates.

---

## Pagination

```sql
-- Keyset: next page after last seen (created_at, id)
select id, name, created_at
from product
where (created_at < ? or (created_at = ? and id < ?))
order by created_at desc, id desc
limit ?;

-- OFFSET only for small datasets
select id, name from product order by id limit ? offset ?;
```

Avoid `offset` on large tables — it scans and discards rows.

---

## Date and Time

Timestamps are ISO 8601 UTC text strings: `'2024-01-15T10:30:00Z'`. Lexicographic string comparison gives correct chronological order.

```sql
-- Current UTC timestamp
strftime('%Y-%m-%dT%H:%M:%SZ', 'now')

-- Subtract an interval
strftime('%Y-%m-%dT%H:%M:%SZ', 'now', '-30 days')
strftime('%Y-%m-%dT%H:%M:%SZ', 'now', '-7 days', '-3 hours')

-- Truncate to start of month
strftime('%Y-%m-01T00:00:00Z', created_at)

-- Records in a rolling 30-day window
where created_at >= strftime('%Y-%m-%dT%H:%M:%SZ', 'now', '-30 days')

-- Extract year/month
strftime('%Y', created_at)                        -- text
cast(strftime('%m', created_at) as integer)       -- integer

-- Day difference
julianday('now') - julianday(created_at)          -- real

-- Unix epoch seconds (3.38.0+; prefer over julianday arithmetic)
unixepoch('now')                                  -- integer
unixepoch(created_at)                             -- integer
```

---

## CTEs

CTEs work the same as PostgreSQL, with no materialization hints:

```sql
with
active_users as (
    select id, email from "user" where is_active = 1
),
recent_orders as (
    select user_id, count(*) as cnt
    from "order"
    where created_at >= strftime('%Y-%m-%dT%H:%M:%SZ', 'now', '-30 days')
    group by user_id
)
select
    u.email,
    coalesce(o.cnt, 0) as orders_last_30d
from active_users u
left join recent_orders o on o.user_id = u.id;
```

Recursive CTEs work identically to PostgreSQL. Use `union all`; `union` (dedup) is rarely correct for tree traversal.

---

## JSON Queries

SQLite has built-in JSON support since 3.9.0. Extract fields with `json_extract()` or the `->>` operator (3.38.0+):

```sql
create table event (
    id         integer primary key,
    type       text    not null,
    payload    text    not null default '{}' check (json_valid(payload)),
    created_at text    not null default (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

-- Extract a field (returns typed value; ->> requires 3.38.0+)
select payload ->> 'user_id' as user_id from event where type = 'order.created';

-- Equivalent with json_extract (works on 3.9.0+)
select json_extract(payload, '$.user_id') as user_id from event;

-- Filter by field value
select id, type from event where payload ->> 'status' = 'pending';

-- Build a JSON object
select json_object('id', id, 'type', type) from event limit 10;
```

Use JSON columns when the document structure varies per row or is unknown at schema design time. Don't use JSON to avoid normalizing fields you'll always query by name — model those as columns.

SQLite has no GIN-equivalent for JSON indexing. Queries filtering on JSON fields do full table scans. Extract frequently-queried JSON fields into real columns or generated columns and index those.

---

## Window Functions

Available since SQLite 3.25.0 (2018). Same syntax as PostgreSQL:

```sql
-- Row number for pagination or deduplication
select id, email,
    row_number() over (order by created_at desc) as rn
from "user";

-- Running total
select id, amount,
    sum(amount) over (order by created_at rows unbounded preceding) as running_total
from payment;

-- Rank within a partition
select product_id, rating,
    rank() over (partition by product_id order by rating desc) as rating_rank
from review;

-- Previous/next row values
select id, status, created_at,
    lag(status) over (order by created_at) as prev_status
from order_event where order_id = ?;
```

Supported functions: `row_number()`, `rank()`, `dense_rank()`, `ntile()`, `lag()`, `lead()`, `first_value()`, `last_value()`, `nth_value()`. All aggregate functions (`sum`, `count`, `avg`, etc.) also work as window functions.

---

## Full-Text Search (FTS5)

FTS5 is included by default in amalgamation builds (most environments including system SQLite and Python's `sqlite3`).

```sql
-- Create a virtual table
create virtual table article_fts using fts5(title, body, content=article, content_rowid=id);

-- Populate from existing table
insert into article_fts(rowid, title, body) select id, title, body from article;

-- Search
select a.id, a.title, rank
from article_fts fts
join article a on a.id = fts.rowid
where article_fts match 'sqlite AND performance'
order by rank;

-- Prefix search
select rowid, title from article_fts where article_fts match 'optim*';
```

Keep the FTS index in sync with triggers on the content table:

```sql
create trigger article_fts_insert after insert on article begin
    insert into article_fts(rowid, title, body) values (new.id, new.title, new.body);
end;

create trigger article_fts_delete after delete on article begin
    insert into article_fts(article_fts, rowid, title, body)
        values ('delete', old.id, old.title, old.body);
end;

create trigger article_fts_update after update on article begin
    insert into article_fts(article_fts, rowid, title, body)
        values ('delete', old.id, old.title, old.body);
    insert into article_fts(rowid, title, body) values (new.id, new.title, new.body);
end;
```
