# 📅 Day 17 — Saturday, 31 May 2026
# dbt Fundamentals: Setup, Models, Ref, Documentation

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Understand what dbt is and why data engineers use it
- Install dbt and connect it to PostgreSQL
- Create your first dbt models (SQL files)
- Use `ref()` to chain models
- Generate documentation
- Run and test your models

**Why this matters:** dbt (data build tool) is the HOTTEST tool in data engineering right now. It appears in almost every modern data job posting. dbt lets you write SQL transformations as code, version control them, test them, and document them. It turns your SQL into a proper software engineering workflow.

---

## ☀️ BLOCK 1: What is dbt? + Setup (Morning, ~1.5 hours)

---

### Task 1: What is dbt? (15 min)

**dbt = data build tool**

Before dbt, data transformation was a mess:
- SQL scripts scattered everywhere
- No version control
- No way to test if data is correct
- No documentation
- No dependency management (which script runs first?)

**dbt solves ALL of this:**

```
Traditional approach:
script1.sql → script2.sql → script3.sql (who runs first? who depends on who?)
                  ↓
dbt approach:
models/
├── staging/
│   ├── stg_customers.sql    ← reads raw data
│   ├── stg_orders.sql       ← reads raw data
│   └── stg_products.sql
├── intermediate/
│   └── int_customer_orders.sql  ← reads staging models
└── marts/
    └── dim_customers.sql    ← reads intermediate (final output)

dbt automatically figures out the dependency order!
```

**dbt key concepts:**
- **Model** = a SQL file that transforms data (SELECT statement)
- **ref()** = function to reference another model (like a pointer)
- **Materialization** = how dbt creates the model (table, view, incremental)
- **Test** = assertions about your data (unique, not_null, relationships)
- **Docs** = auto-generated documentation website

> 🧠 **dbt does NOT move data.** It does NOT extract or load. dbt only TRANSFORMS data that's already in your database. The ELT pattern: Extract → Load → Transform (with dbt).

---

### Task 2: Install dbt (15 min)

```bash
pip install dbt-postgres
```

Verify:
```bash
dbt --version
# Should show dbt-core 1.7+ or 1.8+
```

> 💡 dbt-postgres includes dbt-core + the PostgreSQL adapter. If you're using Neon, it's still PostgreSQL — same adapter.

---

### Task 3: Initialize a dbt Project (30 min)

```bash
mkdir food-delivery-dbt
cd food-delivery-dbt
dbt init food_delivery
```

When prompted:
- **Database:** postgres
- **Host:** localhost (or your Neon host)
- **Port:** 5432
- **User:** postgres (or your Neon user)
- **Password:** your password
- **Database:** neondb (or your local db name)
- **Schema:** dbt_food (or any name)
- **Threads:** 4

This creates the dbt project structure:

```
food_delivery/
├── dbt_project.yml      # Project configuration
├── profiles.yml         # Database connection (auto-created in ~/.dbt/)
├── models/
│   └── example/         # Sample models (delete these)
├── seeds/               # CSV files to load
├── tests/               # Custom tests
├── macros/              # Reusable SQL snippets
├── snapshots/           # SCD Type 2 tracking
├── analysis/            # Ad-hoc analysis queries
└── docs/                # Documentation blocks
```

**Verify connection:**
```bash
dbt debug
```

✅ You should see "All checks passed!" If not, check your `profiles.yml`.

> 💡 **profiles.yml location:** `~/.dbt/profiles.yml` (not in your project folder). This file contains database credentials — NEVER commit it to Git!

---

### Task 4: Configure Your Project (15 min)

Edit `dbt_project.yml`:

```yaml
name: 'food_delivery'
version: '1.0.0'
config-version: 2

profile: 'food_delivery'

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  food_delivery:
    +materialized: view    # default: create models as views
    staging:
      +materialized: view
    marts:
      +materialized: table  # final models as tables (faster queries)
```

Delete the example models:
```bash
rm -rf models/example/
mkdir -p models/staging models/marts
```

Add to `.gitignore`:
```
target/
dbt_packages/
logs/
```

---

## 🔥 BLOCK 2: Build Your First dbt Models (Afternoon, ~2 hours)

---

### Task 5: Seed — Load Raw Data (20 min)

dbt "seeds" let you load CSV files into your database. Perfect for small reference data.

Create `seeds/raw_customers.csv`:

```csv
customer_id,name,email,city,country,signup_date,loyalty_tier
1,Alice Tan,alice@email.com,Singapore,Singapore,2023-01-15,Gold
2,Bob Lim,bob@email.com,Kuala Lumpur,Malaysia,2023-03-20,Silver
3,Cathy Ng,cathy@email.com,Singapore,Singapore,2023-08-10,Silver
4,David Chen,david@email.com,Penang,Malaysia,2024-01-05,Bronze
5,Eva Rahman,eva@email.com,Kuala Lumpur,Malaysia,2023-06-01,Gold
```

Create `seeds/raw_restaurants.csv`:

```csv
restaurant_id,name,cuisine_type,city,area,rating,price_range
1,Chicken Rice King,Chinese,Singapore,Orchard,4.50,$
2,Nasi Lemak House,Malay,Kuala Lumpur,Bangsar,4.30,$
3,Roti Prata Corner,Indian,Singapore,Little India,4.10,$
4,Sushi Express,Japanese,Singapore,Jurong,4.00,$$
5,Tom Yum Palace,Thai,Kuala Lumpur,Pavilion,4.60,$$
```

Create `seeds/raw_orders.csv`:

```csv
order_id,customer_id,restaurant_id,order_date,amount,quantity,dish_name,delivery_fee
1001,1,1,2024-01-15,13.00,2,Hainanese Chicken Rice,3.00
1002,1,1,2024-01-15,7.00,1,Roast Chicken Rice,0.00
1003,2,2,2024-01-16,8.50,1,Nasi Lemak Special,4.00
1004,3,3,2024-01-17,7.50,3,Roti Prata,2.00
1005,1,4,2024-01-20,18.00,1,Salmon Sashimi Set,3.50
1006,4,5,2024-01-22,14.00,1,Green Curry,5.00
1007,5,2,2024-02-03,17.00,2,Nasi Lemak Special,4.00
1008,2,5,2024-02-14,8.00,1,Mango Sticky Rice,4.00
1009,3,1,2024-03-01,6.50,1,Hainanese Chicken Rice,3.00
1010,1,3,2024-03-15,10.50,3,Prata Egg,3.00
```

Load seeds:
```bash
dbt seed
```

✅ You should see 3 CSVs loaded into your database.

---

### Task 6: Staging Models — Clean Raw Data (30 min)

Staging models read raw data and clean it. They're the first transformation layer.

Create `models/staging/stg_customers.sql`:

```sql
-- models/staging/stg_customers.sql

WITH source AS (
    SELECT * FROM {{ ref('raw_customers') }}
)

SELECT
    customer_id,
    TRIM(name) AS customer_name,
    LOWER(TRIM(email)) AS email,
    TRIM(city) AS city,
    TRIM(country) AS country,
    signup_date::DATE AS signup_date,
    TRIM(loyalty_tier) AS loyalty_tier,
    -- Derived fields
    UPPER(LEFT(city, 2)) AS city_code,
    CASE 
        WHEN loyalty_tier = 'Gold' THEN TRUE
        ELSE FALSE
    END AS is_premium
FROM source
WHERE customer_id IS NOT NULL
```

> 🧠 **`{{ ref('raw_customers') }}`** is dbt magic. It tells dbt "this model depends on raw_customers." dbt automatically:
> - Figures out the correct schema/table name
> - Builds models in the right order (raw_customers before stg_customers)
> - Creates a dependency graph you can visualize

Create `models/staging/stg_restaurants.sql`:

```sql
-- models/staging/stg_restaurants.sql

WITH source AS (
    SELECT * FROM {{ ref('raw_restaurants') }}
)

SELECT
    restaurant_id,
    TRIM(name) AS restaurant_name,
    TRIM(cuisine_type) AS cuisine_type,
    TRIM(city) AS city,
    TRIM(area) AS area,
    rating,
    TRIM(price_range) AS price_range,
    CASE 
        WHEN price_range = '$' THEN 'Budget'
        WHEN price_range = '$$' THEN 'Mid-Range'
        ELSE 'Premium'
    END AS price_category
FROM source
WHERE restaurant_id IS NOT NULL
```

Create `models/staging/stg_orders.sql`:

```sql
-- models/staging/stg_orders.sql

WITH source AS (
    SELECT * FROM {{ ref('raw_orders') }}
)

SELECT
    order_id,
    customer_id,
    restaurant_id,
    order_date::DATE AS order_date,
    EXTRACT(YEAR FROM order_date::DATE) AS order_year,
    EXTRACT(MONTH FROM order_date::DATE) AS order_month,
    TO_CHAR(order_date::DATE, 'YYYY-MM') AS order_month_str,
    EXTRACT(DOW FROM order_date::DATE) AS day_of_week_num,
    TO_CHAR(order_date::DATE, 'Day') AS day_of_week_name,
    CASE WHEN EXTRACT(DOW FROM order_date::DATE) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend,
    amount,
    quantity,
    TRIM(dish_name) AS dish_name,
    delivery_fee,
    amount + delivery_fee AS total_with_delivery
FROM source
WHERE order_id IS NOT NULL
```

---

### Task 7: Mart Models — Business Logic (30 min)

Mart models are the final analytical tables — what analysts and dashboards query.

Create `models/marts/dim_customers.sql`:

```sql
-- models/marts/dim_customers.sql

WITH stg AS (
    SELECT * FROM {{ ref('stg_customers') }}
),

orders_agg AS (
    SELECT 
        customer_id,
        COUNT(*) AS total_orders,
        SUM(amount) AS total_spent,
        SUM(quantity) AS total_items_ordered,
        ROUND(AVG(amount), 2) AS avg_order_value,
        MIN(order_date) AS first_order_date,
        MAX(order_date) AS last_order_date
    FROM {{ ref('stg_orders') }}
    GROUP BY customer_id
)

SELECT
    s.customer_id,
    s.customer_name,
    s.email,
    s.city,
    s.country,
    s.loyalty_tier,
    s.is_premium,
    s.signup_date,
    COALESCE(o.total_orders, 0) AS total_orders,
    COALESCE(o.total_spent, 0) AS total_spent,
    COALESCE(o.total_items_ordered, 0) AS total_items_ordered,
    COALESCE(o.avg_order_value, 0) AS avg_order_value,
    o.first_order_date,
    o.last_order_date,
    -- Customer segmentation
    CASE 
        WHEN COALESCE(o.total_spent, 0) >= 30 THEN 'VIP'
        WHEN COALESCE(o.total_spent, 0) >= 15 THEN 'Regular'
        ELSE 'New'
    END AS spending_segment
FROM stg s
LEFT JOIN orders_agg o ON s.customer_id = o.customer_id
```

Create `models/marts/fact_orders_enriched.sql`:

```sql
-- models/marts/fact_orders_enriched.sql

SELECT
    o.order_id,
    o.customer_id,
    c.customer_name,
    c.loyalty_tier,
    c.spending_segment,
    o.restaurant_id,
    r.restaurant_name,
    r.cuisine_type,
    r.price_category,
    o.dish_name,
    o.order_date,
    o.order_month_str,
    o.day_of_week_name,
    o.is_weekend,
    o.quantity,
    o.amount,
    o.delivery_fee,
    o.total_with_delivery,
    o.amount * o.quantity AS line_total
FROM {{ ref('stg_orders') }} o
INNER JOIN {{ ref('stg_customers') }} c ON o.customer_id = c.customer_id
INNER JOIN {{ ref('stg_restaurants') }} r ON o.restaurant_id = r.restaurant_id
```

Create `models/marts/restaurant_summary.sql`:

```sql
-- models/marts/restaurant_summary.sql

SELECT
    restaurant_name,
    cuisine_type,
    city,
    price_category,
    COUNT(*) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(amount) AS total_revenue,
    SUM(quantity) AS total_items_sold,
    ROUND(AVG(amount), 2) AS avg_order_value,
    SUM(delivery_fee) AS total_delivery_fees
FROM {{ ref('fact_orders_enriched') }}
GROUP BY restaurant_name, cuisine_type, city, price_category
ORDER BY total_revenue DESC
```

---

### Task 8: Run Your Models (15 min)

```bash
# Run all models
dbt run

# Run specific model
dbt run --select stg_customers

# Run staging only
dbt run --select staging.*

# Run marts only
dbt run --select marts.*
```

✅ You should see something like:
```
Completed successfully
Done. PASS=5 WARN=0 ERROR=0 SKIP=0 TOTAL=5
```

---

### Task 9: Add Basic Tests (15 min)

Tests validate your data. They're essential for production pipelines.

Create `models/staging/schema.yml`:

```yaml
version: 2

models:
  - name: stg_customers
    description: "Cleaned customer data from raw source"
    columns:
      - name: customer_id
        description: "Unique customer identifier"
        tests:
          - unique
          - not_null
      - name: email
        tests:
          - not_null
          - unique
      - name: loyalty_tier
        tests:
          - accepted_values:
              values: ['Bronze', 'Silver', 'Gold']

  - name: stg_restaurants
    description: "Cleaned restaurant data"
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

  - name: stg_orders
    description: "Cleaned order data"
    columns:
      - name: order_id
        tests:
          - not_null
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers')
              field: customer_id
      - name: restaurant_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_restaurants')
              field: restaurant_id
```

> 💡 The `relationships` test is like a foreign key check — it verifies that every order's customer_id exists in the customers table.

Create `models/marts/schema.yml`:

```yaml
version: 2

models:
  - name: dim_customers
    description: "Customer dimension with aggregated metrics"
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null
      - name: total_spent
        description: "Lifetime total spending"
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0

  - name: fact_orders_enriched
    description: "Enriched fact table with customer and restaurant details"

  - name: restaurant_summary
    description: "Aggregated restaurant performance metrics"
```

Run tests:
```bash
dbt test
```

---

## 🌙 BLOCK 3: Documentation + Graph (Evening, ~1 hour)

---

### Task 10: Generate dbt Docs (15 min)

dbt auto-generates a beautiful documentation website:

```bash
dbt docs generate
dbt docs serve
```

Open http://localhost:8080 in your browser. You'll see:
- All your models listed
- Column descriptions
- Test results
- **A dependency graph** showing how models connect

> 🎯 **The dependency graph is impressive in interviews.** It shows you understand modern data engineering workflows.

---

### Task 11: View the DAG (10 min)

Run:
```bash
dbt compile && dbt run --debug
```

Or generate a visual graph:
```bash
dbt run --select +dim_customers
```

The `+` means "include all upstream dependencies."

Try these selectors:
```bash
# Run dim_customers and everything it depends on
dbt run --select +dim_customers

# Run stg_orders and everything that depends on it
dbt run --select stg_orders+

# Run only models that changed and their dependents
dbt run --select state:modified+
```

---

### Task 12: Daily Reflection + GitHub Push (15 min)

```bash
cd food-delivery-dbt
git init
git add .
git commit -m "Initial commit: dbt food delivery project with staging + marts"
# Create GitHub repo, then push
```

```markdown
# Day 17 — dbt Fundamentals
Date: 2026-05-31

## What is dbt?
- Data build tool — transforms data already in your warehouse
- SQL SELECT statements as models
- ref() for dependency management
- Tests for data quality
- Auto-generated documentation

## dbt Project Structure
- seeds/ → raw CSV data loaded to database
- models/staging/ → clean raw data
- models/marts/ → final analytical tables
- schema.yml → tests and documentation

## Key Commands
- dbt seed → load CSVs
- dbt run → execute all models
- dbt test → run data quality tests
- dbt docs generate/serve → documentation website

## Tomorrow: dbt Advanced (incremental models, macros, more tests)
```

```bash
cd data-engineering-journey
git add .
git commit -m "Day 17: dbt fundamentals — models, ref, tests, docs"
git push
```

---

## ✅ Day 17 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | dbt installed and connected to PostgreSQL | ☐ |
| 2 | Understand dbt concepts (model, ref, materialization, test) | ☐ |
| 3 | Created dbt project with proper structure | ☐ |
| 4 | Loaded seed data (3 CSVs) | ☐ |
| 5 | Created 3 staging models with CTEs and ref() | ☐ |
| 6 | Created 3 mart models with joins and aggregation | ☐ |
| 7 | All models run successfully (dbt run) | ☐ |
| 8 | Added schema tests (unique, not_null, relationships, accepted_values) | ☐ |
| 9 | All tests pass (dbt test) | ☐ |
| 10 | Generated and viewed dbt docs | ☐ |
| 11 | Understand dbt DAG and model selectors | ☐ |
| 12 | Pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card

```bash
# dbt commands
dbt init <project>      # create new project
dbt debug                # test connection
dbt seed                 # load CSVs
dbt run                  # execute all models
dbt test                 # run data tests
dbt docs generate        # build docs
dbt docs serve           # serve docs on localhost:8080

# Selectors
dbt run --select staging.*      # run staging only
dbt run --select +dim_customers # model + upstream deps
dbt run --select stg_orders+    # model + downstream deps
```

```sql
-- dbt model pattern
WITH source AS (
    SELECT * FROM {{ ref('upstream_model') }}
)
SELECT ...
FROM source

-- schema.yml tests
tests:
  - unique
  - not_null
  - relationships:
      to: ref('other_model')
      field: id
  - accepted_values:
      values: ['A', 'B', 'C']
```
