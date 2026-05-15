# 📅 Day 19 — Monday, 2 June 2026
# dbt Advanced Part 2: Sources, Custom Tests, dbt Cloud, CI/CD Basics

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Define sources (raw data) in dbt
- Write custom SQL-based tests
- Understand dbt Cloud vs dbt Core
- Set up a basic CI/CD workflow for dbt
- Polish your dbt project for portfolio readiness

**Why this matters:** In production, you don't just write models — you connect to raw data sources, test data quality rigorously, and automate everything with CI/CD. Today gets you closer to production-ready.

---

## ☀️ BLOCK 1: Sources + Custom Tests (Morning, ~1.5 hours)

---

### Task 1: Define Sources (20 min)

Sources tell dbt where your raw data lives. Instead of `ref()` for raw tables, you use `source()`.

Create `models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: raw_data
    description: "Raw data loaded from API and CSV sources"
    schema: public  # or whatever schema your seeds are in
    tables:
      - name: raw_customers
        description: "Raw customer data from CRM export"
        columns:
          - name: customer_id
            tests:
              - unique
              - not_null
          - name: email
            tests:
              - not_null
          - name: loyalty_tier
            tests:
              - accepted_values:
                  values: ['Bronze', 'Silver', 'Gold']

      - name: raw_restaurants
        description: "Restaurant master data"
        columns:
          - name: restaurant_id
            tests:
              - unique
              - not_null
          - name: rating
            tests:
              - dbt_expectations.expect_column_values_to_be_between:
                  min_value: 0
                  max_value: 5

      - name: raw_orders
        description: "Transactional order data"
        columns:
          - name: order_id
            tests:
              - not_null
          - name: amount
            tests:
              - dbt_expectations.expect_column_values_to_be_between:
                  min_value: 0
```

Now update staging models to use `source()` instead of `ref()` for seed tables:

```sql
-- models/staging/stg_customers.sql
-- Change this line:
-- SELECT * FROM {{ ref('raw_customers') }}
-- To:
SELECT * FROM {{ source('raw_data', 'raw_customers') }}
```

> 🧠 **source() vs ref():**
> - `source()` = points to raw data that exists BEFORE dbt (loaded by ETL)
> - `ref()` = points to another dbt model (created BY dbt)
> - In the dbt DAG, sources are the entry points (no upstream dependencies)

---

### Task 2: Custom SQL Tests (30 min)

Built-in tests (unique, not_null, etc.) cover basics. Custom tests let you write ANY SQL assertion.

Create `tests/assert_positive_revenue.sql`:

```sql
-- tests/assert_positive_revenue.sql
-- Custom test: every order should have positive revenue

SELECT order_id
FROM {{ ref('stg_orders') }}
WHERE amount <= 0
-- If this returns ANY rows, the test FAILS
```

Create `tests/assert_valid_customer_restaurant_pairs.sql`:

```sql
-- tests/assert_valid_customer_restaurant_pairs.sql
-- Custom test: customer and restaurant should be in a valid country pair

SELECT 
    o.order_id,
    c.country AS customer_country,
    r.city AS restaurant_city
FROM {{ ref('stg_orders') }} o
JOIN {{ ref('stg_customers') }} c ON o.customer_id = c.customer_id
JOIN {{ ref('stg_restaurants') }} r ON o.restaurant_id = r.restaurant_id
WHERE c.country = 'Malaysia' AND r.city = 'Singapore'
-- Malaysian customers ordering from SG restaurants might be a data issue
```

Create `tests/assert_recent_orders_have_delivery_fee.sql`:

```sql
-- tests/assert_recent_orders_have_delivery_fee.sql
-- Custom test: orders in 2024 should have a delivery fee

SELECT order_id
FROM {{ ref('stg_orders') }}
WHERE order_date >= '2024-01-01'
  AND delivery_fee = 0
  AND amount > 10
-- Large orders with no delivery fee might be data errors
```

Run all tests:
```bash
dbt test
```

---

### Task 3: Generic Tests with Parameters (20 min)

Create `macros/test_at_least_one.sql`:

```sql
-- macros/test_at_least_one.sql
-- Generic test: ensure a model has at least N rows

{% test at_least_n(model, column_name, n=1) %}

SELECT COUNT(*)
FROM {{ model }}
HAVING COUNT({{ column_name }}) < {{ n }}

{% endtest %}
```

Use it in schema.yml:
```yaml
models:
  - name: dim_customers
    columns:
      - name: customer_id
        tests:
          - at_least_n:
              n: 1
```

---

## 🔥 BLOCK 2: dbt Cloud + CI/CD Basics (Afternoon, ~1.5 hours)

---

### Task 4: dbt Cloud vs dbt Core (15 min)

| Feature | dbt Core (CLI) | dbt Cloud |
|---------|---------------|-----------|
| Cost | Free | Free tier available |
| Setup | Local install | Web-based |
| IDE | Your editor | Built-in web IDE |
| Scheduling | Manual or external | Built-in |
| CI/CD | You build it | Built-in |
| Logs | Local files | Web dashboard |
| Team features | None | Collaboration, access control |

**For learning:** dbt Core (what you're using) is perfect.
**For jobs:** Most companies use dbt Cloud or dbt Core + Airflow for scheduling.

---

### Task 5: GitHub Actions for dbt CI/CD (30 min)

This is how real teams run dbt automatically.

Create `.github/workflows/dbt_ci.yml`:

```yaml
name: dbt CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  dbt-run-and-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dbt
        run: pip install dbt-postgres
      
      - name: dbt deps
        run: dbt deps
      
      - name: dbt debug
        run: dbt debug
      
      - name: dbt run
        run: dbt run --target prod
      
      - name: dbt test
        run: dbt test --target prod
```

> 💡 This is a basic CI pipeline. In production, you'd add:
> - Separate targets for dev/prod
> - Slack notifications on failure
> - `dbt source freshness` checks
> - Artifacts storage

---

### Task 6: dbt Source Freshness (15 min)

Source freshness checks if your raw data is up-to-date:

Create `models/staging/sources.yml` (add freshness):

```yaml
sources:
  - name: raw_data
    freshness:
      warn_after: {count: 24, period: hour}
      error_after: {count: 48, period: hour}
    loaded_at_field: signup_date  # or whatever timestamp column
    tables:
      - name: raw_customers
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 24, period: hour}
```

Run freshness check:
```bash
dbt source freshness
```

This tells you: "Is my data fresh enough? When was it last updated?"

---

## 🔥 BLOCK 3: Polish Your dbt Project (Afternoon/Evening, ~2 hours)

---

### Task 7: Complete Documentation (30 min)

Update ALL schema.yml files with proper descriptions:

```yaml
# models/marts/schema.yml
version: 2

models:
  - name: dim_customers
    description: >
      Customer dimension table with aggregated order metrics.
      One row per customer with lifetime spending, order counts,
      and customer segmentation.
    columns:
      - name: customer_id
        description: "Surrogate primary key"
        tests: [unique, not_null]
      - name: customer_name
        description: "Full name of customer"
      - name: email
        description: "Customer email address (lowercase)"
      - name: city
        description: "Customer's city of residence"
      - name: country
        description: "Customer's country"
      - name: loyalty_tier
        description: "Current loyalty program tier (Bronze/Silver/Gold)"
        tests:
          - accepted_values:
              values: ['Bronze', 'Silver', 'Gold']
      - name: signup_date
        description: "Date the customer first signed up"
      - name: total_orders
        description: "Lifetime count of orders placed"
      - name: total_spent
        description: "Lifetime total spending in USD"
      - name: avg_order_value
        description: "Average order value in USD"
      - name: spending_segment
        description: >
          Customer segment based on spending:
          VIP (>$30), Regular ($15-30), New (<$15)

  - name: fact_orders_enriched
    description: >
      Enriched fact table joining orders with customer and restaurant details.
      One row per order line item.
    columns:
      - name: order_id
        description: "Unique order identifier"
        tests: [not_null]
      - name: customer_name
        description: "Name of the ordering customer"
      - name: restaurant_name
        description: "Name of the restaurant"
      - name: cuisine_type
        description: "Type of cuisine (Chinese, Malay, Indian, etc.)"
      - name: total_with_delivery
        description: "Order amount plus delivery fee"

  - name: restaurant_summary
    description: >
      Aggregated restaurant performance metrics.
      One row per restaurant with total revenue, customer counts,
      and average order values.

  - name: daily_order_summary
    description: >
      Incremental model aggregating daily order metrics.
      Processes only new data on subsequent runs.
```

---

### Task 8: Final Project Structure Check (15 min)

Your project should look like this:

```
food-delivery-dbt/
├── README.md
├── dbt_project.yml
├── packages.yml
├── profiles.yml (NEVER commit this!)
├── .gitignore
├── .github/workflows/dbt_ci.yml
├── macros/
│   ├── currency_format.sql
│   ├── date_dimension.sql
│   ├── log_msg.sql
│   ├── safe_divide.sql
│   ├── surrogate_key.sql
│   └── test_at_least_one.sql
├── models/
│   ├── staging/
│   │   ├── sources.yml
│   │   ├── stg_customers.sql
│   │   ├── stg_restaurants.sql
│   │   └── stg_orders.sql
│   └── marts/
│       ├── schema.yml
│       ├── dim_customers.sql
│       ├── fact_orders_enriched.sql
│       ├── fact_orders_incremental.sql
│       ├── restaurant_summary.sql
│       └── daily_order_summary.sql
├── seeds/
│   ├── raw_customers.csv
│   ├── raw_restaurants.csv
│   └── raw_orders.csv
├── snapshots/
│   ├── customers_snapshot.sql
│   └── restaurants_snapshot.sql
├── tests/
│   ├── assert_positive_revenue.sql
│   ├── assert_valid_customer_restaurant_pairs.sql
│   └── assert_recent_orders_have_delivery_fee.sql
└── docs/
    └── erd.png (from dbdiagram.io)
```

---

### Task 9: Verify Everything Works (15 min)

```bash
# Full clean run
dbt clean
dbt deps
dbt seed
dbt run
dbt test
dbt snapshot
dbt source freshness
dbt docs generate
```

All should pass with no errors.

---

### Task 10: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 19 — dbt Advanced Part 2
Date: 2026-06-02

## Key Skills Added
- Sources: define raw data with source() instead of ref()
- Custom tests: write SQL assertions for data quality
- Generic tests: parameterized reusable tests
- CI/CD: GitHub Actions for automated dbt runs
- Source freshness: detect stale data
- Documentation: comprehensive schema.yml with descriptions

## dbt Vocabulary I Now Know
- model, ref, source, seed, snapshot
- materialization (view/table/incremental/ephemeral)
- macro, test, hook, package
- DAG, lineage, selector (+, *)
- dbt Cloud vs dbt Core

## Tomorrow: PORTFOLIO PROJECT 3 — Final dbt project + Week 3 checkpoint
```

```bash
git add .
git commit -m "Day 19: dbt advanced — sources, custom tests, CI/CD, full documentation"
git push
```

---

## ✅ Day 19 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Defined sources in sources.yml | ☐ |
| 2 | Updated models to use source() | ☐ |
| 3 | Wrote 3 custom SQL tests | ☐ |
| 4 | Created a generic test macro | ☐ |
| 5 | Set up GitHub Actions CI/CD | ☐ |
| 6 | Added source freshness checks | ☐ |
| 7 | Comprehensive schema.yml with all descriptions | ☐ |
| 8 | Full dbt clean run passes (seed + run + test + snapshot) | ☐ |
| 9 | Day 19 notes pushed to GitHub | ☐ |
