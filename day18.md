# 📅 Day 18 — Sunday, 1 June 2026
# dbt Advanced: Incremental Models, Macros, Snapshots

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Build incremental models (process only NEW data)
- Write reusable macros (like SQL functions)
- Use dbt snapshots for SCD Type 2
- Add custom tests and hooks
- Understand dbt materializations (view, table, incremental, ephemeral)

**Why this matters:** Yesterday was "dbt works." Today is "dbt works at SCALE." Incremental models save hours of compute. Macros keep your code DRY. Snapshots give you historical tracking. These are the skills that separate a junior dbt user from someone who can run dbt in production.

---

## ☀️ BLOCK 1: Materializations (Morning, ~1 hour)

---

### Task 1: The Four Materializations (20 min)

```yaml
# In dbt_project.yml or per-model config:
{{ config(materialized="table") }}
```

| Type | What dbt does | When to use |
|------|---------------|-------------|
| **view** | `CREATE VIEW AS` | Staging — lightweight, always fresh |
| **table** | `CREATE TABLE AS` (drops + recreates) | Marts — queried frequently |
| **incremental** | INSERT new rows only | Large fact tables — don't reprocess all history |
| **ephemeral** | Not stored — inlined as CTE | Helper logic used in other models |

```sql
-- 🔹 Per-model config (add at top of .sql file)
{{ config(
    materialized="incremental",
    unique_key="order_id",
    schema="analytics"
) }}

SELECT * FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

> 🧠 **The `is_incremental()` macro** returns TRUE when the table already exists (subsequent runs) and FALSE on the first run. This is how you filter to only new data.

---

### Task 2: Build an Incremental Model (30 min)

Create `models/marts/fact_orders_incremental.sql`:

```sql
-- models/marts/fact_orders_incremental.sql
{{ config(
    materialized="incremental",
    unique_key="order_id",
    incremental_strategy="merge"
) }}

WITH source AS (
    SELECT * FROM {{ ref('stg_orders') }}
),

enriched AS (
    SELECT
        o.order_id,
        o.customer_id,
        c.customer_name,
        c.loyalty_tier,
        o.restaurant_id,
        r.restaurant_name,
        r.cuisine_type,
        o.dish_name,
        o.order_date,
        o.order_month_str,
        o.day_of_week_name,
        o.is_weekend,
        o.quantity,
        o.amount,
        o.delivery_fee,
        o.total_with_delivery
    FROM source o
    INNER JOIN {{ ref('stg_customers') }} c ON o.customer_id = c.customer_id
    INNER JOIN {{ ref('stg_restaurants') }} r ON o.restaurant_id = r.restaurant_id
)

SELECT * FROM enriched

{% if is_incremental() %}
-- Only process orders newer than what's already in the table
WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

Run it:
```bash
# First run: creates table with ALL data
dbt run --select fact_orders_incremental

# Second run: only inserts NEW rows (nothing new = nothing happens)
dbt run --select fact_orders_incremental
```

> 🧠 **Why incremental matters:** Imagine 100 million orders. A `table` materialization reprocesses ALL 100M every run. An `incremental` model only processes the ~100K new orders since the last run. That's 1000x less compute!

---

### Task 3: Ephemeral Models (10 min)

```sql
-- models/staging/_int_order_ranked.sql (prefix _ = helper)
{{ config(materialized="ephemeral") }}

SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) as recency_rank,
    DENSE_RANK() OVER (ORDER BY amount DESC) as revenue_rank
FROM {{ ref('stg_orders') }}

-- This model is NOT created in the database
-- It's inlined as a CTE wherever you reference it
-- Use for intermediate calculations that no one queries directly
```

---

## 🔥 BLOCK 2: Macros — Reusable SQL (Afternoon, ~1.5 hours)

---

### Task 4: Your First Macro (20 min)

Macros are like functions for SQL. Write once, use everywhere.

Create `macros/currency_format.sql`:

```sql
-- macros/currency_format.sql

{% macro format_currency(column_name, alias=none) %}
    ROUND({{ column_name }}, 2) {% if alias %} AS {{ alias }} {% endif %}
{% endmacro %}
```

Use it in a model:

```sql
-- In any model:
SELECT
    customer_name,
    {{ format_currency('total_spent', 'formatted_spending') }}
FROM {{ ref('dim_customers') }}
```

---

### Task 5: Useful Macros for Data Engineering (30 min)

Create `macros/date_dimension.sql`:

```sql
-- macros/date_dimension.sql
-- Generate a date dimension table for a given range

{% macro generate_date_dimension(start_date, end_date) %}

WITH date_spine AS (
    {{ dbt_utils.date_spine(
        datepart="day",
        start_date="'" ~ start_date ~ "'::date",
        end_date="'" ~ end_date ~ "'::date"
    ) }}
)

SELECT
    TO_CHAR(date_day, 'YYYYMMDD')::INTEGER AS date_id,
    date_day::DATE AS full_date,
    TRIM(TO_CHAR(date_day, 'Day')) AS day_of_week,
    EXTRACT(DAY FROM date_day)::INTEGER AS day_number,
    EXTRACT(MONTH FROM date_day)::INTEGER AS month_number,
    TRIM(TO_CHAR(date_day, 'Month')) AS month_name,
    'Q' || EXTRACT(QUARTER FROM date_day) AS quarter,
    EXTRACT(YEAR FROM date_day)::INTEGER AS year,
    EXTRACT(DOW FROM date_day) IN (0, 6) AS is_weekend,
    EXTRACT(WEEK FROM date_day)::INTEGER AS week_number

FROM date_spine

{% endmacro %}
```

Create `macros/surrogate_key.sql`:

```sql
-- macros/surrogate_key.sql
-- Generate a hash-based surrogate key from multiple columns

{% macro surrogate_key(columns) %}
    MD5({% for col in columns %}COALESCE(CAST({{ col }} AS VARCHAR), ''){% if not loop.last %} || '-' || {% endif %}{% endfor %})
{% endmacro %}
```

Use surrogate_key in a model:
```sql
-- In a model:
SELECT
    {{ surrogate_key(['customer_id', 'order_date']) }} AS customer_date_key,
    customer_id,
    order_date
FROM {{ ref('stg_orders') }}
```

Create `macros/safe_divide.sql`:

```sql
-- macros/safe_divide.sql
-- Division that returns 0 instead of division by zero error

{% macro safe_divide(numerator, denominator, decimals=2) %}
    ROUND(
        CASE 
            WHEN {{ denominator }} = 0 OR {{ denominator }} IS NULL THEN 0
            ELSE {{ numerator }} / {{ denominator }}
        END,
        {{ decimals }}
    )
{% endmacro %}
```

Use it:
```sql
-- In restaurant_summary model:
SELECT
    restaurant_name,
    total_revenue,
    total_orders,
    {{ safe_divide('total_revenue', 'total_orders') }} AS avg_order_value
FROM {{ ref('some_model') }}
```

---

### Task 6: dbt_utils Package (15 min)

dbt has a package ecosystem. The most useful is `dbt_utils`.

Create `packages.yml`:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: "1.1.1"
```

Install:
```bash
dbt deps
```

Now you can use dbt_utils macros:

```sql
-- Useful dbt_utils macros:

-- Generate a date spine
{{ dbt_utils.date_spine(
    datepart="day",
    start_date="'2024-01-01'",
    end_date="'2024-12-31'"
) }}

-- Safe pivot
{{ dbt_utils.pivot('category', dbt_utils.get_column_values(ref('stg_orders'), 'category')) }}

-- Get column values as a list
{{ dbt_utils.get_column_values(ref('stg_restaurants'), 'cuisine_type') }}

-- Star macro (select all columns from a ref)
{{ dbt_utils.star(from=ref('stg_customers')) }}

-- Add leading zeros
{{ dbt_utils.string_literal('hello') }}
```

---

## 🔥 BLOCK 3: Snapshots — SCD Type 2 in dbt (Afternoon, ~1 hour)

---

### Task 7: dbt Snapshots (30 min)

Snapshots are dbt's built-in SCD Type 2 implementation. They track changes to ANY table over time.

Create `snapshots/customers_snapshot.sql`:

```sql
-- snapshots/customers_snapshot.sql

{% snapshot customers_snapshot %}

{{
    config(
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='timestamp',
        updated_at='signup_date',
        invalidate_hard_deletes=True
    )
}}

SELECT * FROM {{ ref('stg_customers') }}

{% endsnapshot %}
```

Create `snapshots/restaurants_snapshot.sql`:

```sql
-- snapshots/restaurants_snapshot.sql

{% snapshot restaurants_snapshot %}

{{
    config(
        target_schema='snapshots',
        unique_key='restaurant_id',
        strategy='check',
        check_cols=['cuisine_type', 'city', 'rating'],
        invalidate_hard_deletes=True
    )
}}

SELECT * FROM {{ ref('stg_restaurants') }}

{% endsnapshot %}
```

Run snapshots:
```bash
dbt snapshot
```

Now check the snapshots table:
```sql
SELECT * FROM snapshots.customers_snapshot;

-- You'll see additional columns:
-- dbt_scd_id          — unique hash per version
-- dbt_updated_at      — when this version became current
-- dbt_valid_from      — start of validity
-- dbt_valid_to        — end of validity (NULL = current)
-- dbt_change_reason   — why the change happened
```

> 🧠 **Two snapshot strategies:**
> - **timestamp:** Detects changes by checking if `updated_at` column changed
> - **check:** Detects changes by checking specific columns you list
> Use `check` when there's no reliable updated_at timestamp.

---

### Task 8: Hooks — Run Code at Specific Times (15 min)

Add to `dbt_project.yml`:

```yaml
on-run-start:
  - "{{ log_msg('Pipeline started at ' ~ run_started_at) }}"

on-run-end:
  - "{{ log_msg('Pipeline completed. Models: ' ~ results | length) }}"
  - >
    {% if exceptions %}
      {{ log_msg('ERRORS: ' ~ exceptions | length, info=True) }}
    {% endif %}
```

Create `macros/log_msg.sql`:

```sql
{% macro log_msg(message) %}
    {% do log(message, info=True) %}
{% endmacro %}
```

---

## 🌙 BLOCK 4: Practice + Consolidation (Evening, ~1.5 hours)

---

### Task 9: Enhance Your dbt Project (30 min)

Add these to your food-delivery-dbt project:

1. **Incremental model** for daily order aggregation
2. **A macro** that calculates revenue with optional tax rate parameter
3. **A snapshot** on the stg_restaurants table
4. **Add tests** to all mart models in schema.yml
5. **Add column descriptions** to all models

Create `models/marts/daily_order_summary.sql`:

```sql
{{ config(
    materialized="incremental",
    unique_key="order_month_str",
    incremental_strategy="merge"
) }}

SELECT
    order_month_str,
    COUNT(DISTINCT order_id) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(quantity) AS total_items,
    SUM(amount) AS total_revenue,
    SUM(delivery_fee) AS total_delivery_fees,
    {{ safe_divide('SUM(amount)', 'COUNT(DISTINCT order_id)') }} AS avg_order_value
FROM {{ ref('stg_orders') }}
GROUP BY order_month_str

{% if is_incremental() %}
WHERE order_month_str > (SELECT MAX(order_month_str) FROM {{ this }})
{% endif %}
```

---

### Task 10: Explore dbt Docs Site (15 min)

```bash
dbt docs generate
dbt docs serve
```

Explore the generated documentation. Notice:
- The **DAG** (directed acyclic graph) showing model dependencies
- Column descriptions
- Test results
- Source-to-target lineage

---

### Task 11: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 18 — dbt Advanced
Date: 2026-06-01

## Key Concepts
- Incremental models: process only new data (merge/append strategies)
- Ephemeral models: inline CTEs, not materialized
- Macros: reusable SQL functions (format_currency, safe_divide, surrogate_key)
- dbt_utils package: community macros for common patterns
- Snapshots: built-in SCD Type 2 tracking
- Hooks: run code before/after dbt execution
- Materializations: view vs table vs incremental vs ephemeral

## dbt Commands I Know
- dbt seed, run, test, docs, snapshot, deps, debug
- dbt run --select +model_name (with upstream)
- dbt run --select model_name+ (with downstream)

## Tomorrow: dbt Advanced Part 2 + Project Polish
```

```bash
git add .
git commit -m "Day 18: dbt advanced — incremental, macros, snapshots, hooks"
git push
```

---

## ✅ Day 18 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Understand 4 materialization types | ☐ |
| 2 | Built an incremental model | ☐ |
| 3 | Created 3+ custom macros | ☐ |
| 4 | Installed dbt_utils package | ☐ |
| 5 | Created snapshots for SCD Type 2 | ☐ |
| 6 | Added hooks for logging | ☐ |
| 7 | Created daily_order_summary incremental model | ☐ |
| 8 | All models run + tests pass | ☐ |
| 9 | Day 18 notes pushed to GitHub | ☐ |
