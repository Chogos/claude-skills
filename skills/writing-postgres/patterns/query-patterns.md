# Query Patterns

## Contents

- Window functions
- Recursive CTEs
- Lateral joins
- Array aggregation and FILTER
- Full-text search
- Date range queries
- SQL/JSON (PG 12+, PG 17+)
- Pivot with FILTER

---

## Window Functions

### Ranking

```sql
-- Rank users by spend within each region
select
    user_id,
    region,
    total_spend,
    rank()        over w as spend_rank,     -- gaps after ties (1, 1, 3)
    dense_rank()  over w as spend_dense,    -- no gaps (1, 1, 2)
    row_number()  over w as row_num,        -- always unique
    ntile(4)      over w as quartile
from user_spend
window w as (partition by region order by total_spend desc);
```

### Running totals and moving averages

```sql
select
    order_date,
    daily_revenue,
    sum(daily_revenue) over (order by order_date) as running_total,
    avg(daily_revenue) over (
        order by order_date
        rows between 6 preceding and current row
    ) as rolling_7d_avg
from daily_revenue
order by order_date;
```

### Lead / lag and DISTINCT ON

```sql
-- Most recent order per user (DISTINCT ON is PostgreSQL-specific)
select distinct on (user_id)
    user_id,
    id         as latest_order_id,
    created_at
from orders
order by user_id, created_at desc;
```

`lag(col, 1) over (order by ...)` / `lead(col, 1) over (...)` compare to previous/next rows.

---

## Recursive CTEs

For hierarchical data and graph traversal.

```sql
-- Ancestor path from a node up to the root
with recursive ancestors as (
    select id, parent_id, name, 0 as level
    from categories
    where id = $1

    union all

    select c.id, c.parent_id, c.name, a.level + 1
    from categories c
    join ancestors a on c.id = a.parent_id          -- walk up
)
select * from ancestors order by level desc;
```

Use `union all`; `union` (dedup) is rarely correct for tree traversal. See [schema-design.md](schema-design.md) for the subtree (walk-down) pattern with depth guards.

---

## Lateral Joins

`lateral` lets the right-hand side reference columns from the left-hand side. Useful for top-N per group without window functions.

```sql
-- Top 3 highest-value orders per user
select u.id, u.email, o.id as order_id, o.total
from users u
cross join lateral (
    select id, total
    from orders
    where user_id = u.id
    order by total desc
    limit 3
) o;

-- Most recent order per user; left join keeps users with zero orders
select u.id, u.email, o.id as order_id, o.total, o.created_at
from users u
left join lateral (
    select id, total, created_at
    from orders
    where user_id = u.id
    order by created_at desc
    limit 1
) o on true;
```

---

## Array Aggregation

```sql
-- Collect tag names per product as an array
select
    p.id,
    p.name,
    array_agg(t.name order by t.name) as tags
from products p
join product_tags pt on pt.product_id = p.id
join tags t           on t.id = pt.tag_id
group by p.id, p.name;

-- Conditional aggregation: counts per status in one pass
select
    user_id,
    count(*) filter (where status = 'pending')   as pending,
    count(*) filter (where status = 'shipped')   as shipped,
    count(*) filter (where status = 'cancelled') as cancelled
from orders
group by user_id;
```

Use `json_agg(json_build_object(...))` for JSON array output (useful for API responses).

---

## Full-Text Search

For simple substring matching, use `ilike`. For ranked search across text, use `tsvector`.

```sql
-- Generated tsvector column
-- STORED is required — GIN indexes cannot reference VIRTUAL generated columns
-- PG 18 defaults to VIRTUAL, so STORED must be explicit or the index creation fails
alter table articles
    add column search_vector tsvector
    generated always as (
        to_tsvector('english', coalesce(title, '') || ' ' || coalesce(body, ''))
    ) stored;

create index idx_articles_fts on articles using gin(search_vector);

-- Search with ranking (websearch_to_tsquery handles natural language input)
select id, title, ts_rank(search_vector, query) as rank
from articles, websearch_to_tsquery('english', $1) query
where search_vector @@ query
order by rank desc
limit 20;
```

Use `ts_headline('english', body, query, 'MaxWords=30, MinWords=10')` to highlight matches in results.

**Query syntax** (`to_tsquery`): `'cat & dog'` (both), `'cat | dog'` (either), `'!cat'` (exclude), `'cat <-> dog'` (adjacent). For user-facing search, prefer `websearch_to_tsquery` — accepts natural input like `"fat rat" or cat -dog`.

---

## Date Range Queries

```sql
-- Records in a calendar month (half-open range avoids boundary issues)
select * from orders
where created_at >= date_trunc('month', $1::timestamptz)
  and created_at <  date_trunc('month', $1::timestamptz) + interval '1 month';

-- Last N days (rolling window)
select * from events
where created_at >= now() - interval '30 days';

-- Overlap check with daterange type
select * from bookings
where daterange(start_date, end_date) && daterange($1, $2);
```

Use `generate_series(start, end, interval '1 day')` to generate date series for gap-filling charts.

---

## SQL/JSON

### Path operators (PG 12+)

More expressive than `->` / `->>` for complex queries:

```sql
select * from events where payload @? '$.address.city';     -- path existence
select * from events where payload @@ '$.score > 90';       -- path predicate
select jsonb_path_query(payload, '$.items[*].name') from events where id = $1;
```

### ISO SQL/JSON functions (PG 17+)

```sql
-- JSON_TABLE: shred JSON into rows
select e.id, j.*
from events e,
json_table(
    e.payload, '$.items[*]'
    columns (
        name  text path '$.name',
        qty   int  path '$.quantity',
        price numeric path '$.price'
    )
) j
where e.type = 'order.created';

-- JSON_VALUE: scalar extraction with type coercion
select id, json_value(payload, '$.score' returning int) as score
from events where type = 'game.finished';

-- JSON_EXISTS: conditional filtering
select * from events
where json_exists(payload, '$.items[*] ? (@.qty > 10)');
```

`json_query(payload, '$.address')` extracts sub-objects.

---

## Pivot with FILTER

```sql
select
    extract(year from created_at) as year,
    sum(total) filter (where extract(month from created_at) = 1)  as jan,
    sum(total) filter (where extract(month from created_at) = 2)  as feb,
    sum(total) filter (where extract(month from created_at) = 3)  as mar
from orders
group by year
order by year;
```
