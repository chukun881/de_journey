# 📅 Day 16 — Friday, 30 May 2026
# Data Modeling Part 2: Slowly Changing Dimensions, Snowflake Schema, ERD Tools

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Handle Slowly Changing Dimensions (SCD Type 1, 2, 3)
- Understand snowflake schema vs star schema
- Use dbdiagram.io to create professional ERDs
- Design a complete data model for a SaaS company

**Why this matters:** In real companies, data CHANGES. Customers move cities, products change categories, employees get promoted. How do you handle historical changes? That's what SCD solves — and it comes up in almost every data engineering interview.

---

## ☀️ BLOCK 1: Slowly Changing Dimensions (Morning, ~1.5 hours)

---

### Task 1: The Problem — Data Changes Over Time (15 min)

```sql
-- Alice moves from Singapore to Kuala Lumpur
-- Do you UPDATE the record? Keep history? Both?

-- This matters for analytics:
-- "How much revenue came from Singapore customers in Q1?"
-- If Alice moved in March and you just updated her city,
-- ALL her Q1 orders now show KL instead of Singapore → WRONG!
```

**Three strategies to handle this:**

---

### Task 2: SCD Type 1 — Overwrite (No History) (15 min)

Simply update the value. Old data is lost.

```sql
-- 🔹 Type 1: Just overwrite
UPDATE dim_customers SET city = 'Kuala Lumpur' WHERE customer_name = 'Alice Tan';

-- ✅ Simple
-- ❌ Lost history — can't analyze Singapore vs KL over time
-- Use when: history doesn't matter (fixing typos, rarely-used fields)
```

---

### Task 3: SCD Type 2 — Add New Row (Full History) ⭐ MOST IMPORTANT (30 min)

Add a new row with the new value, keep the old row. Mark which is current.

```sql
-- 🔹 Type 2: New row for each change

-- Add columns to track history
ALTER TABLE dim_customers ADD COLUMN effective_date DATE DEFAULT CURRENT_DATE;
ALTER TABLE dim_customers ADD COLUMN expiry_date DATE DEFAULT '9999-12-31';
ALTER TABLE dim_customers ADD COLUMN is_current BOOLEAN DEFAULT TRUE;

-- Old record (keep it, mark as not current)
-- customer_id=1, name='Alice Tan', city='Singapore', 
-- effective_date='2023-01-15', expiry_date='2024-03-01', is_current=FALSE

-- New record (insert new row)
-- customer_id=6, name='Alice Tan', city='Kuala Lumpur',
-- effective_date='2024-03-01', expiry_date='9999-12-31', is_current=TRUE

-- 🔹 Query for CURRENT data only
SELECT * FROM dim_customers WHERE is_current = TRUE;

-- 🔹 Query for data AS OF a specific date
SELECT * FROM dim_customers 
WHERE '2024-02-15' BETWEEN effective_date AND expiry_date;
-- Returns Alice with city='Singapore' (as of Feb 15, she hadn't moved yet)

-- 🔹 Implementation pattern:
-- Step 1: Expire the old record
UPDATE dim_customers 
SET expiry_date = CURRENT_DATE, is_current = FALSE
WHERE customer_id = 1 AND is_current = TRUE;

-- Step 2: Insert new record
INSERT INTO dim_customers (customer_name, email, city, country, signup_date, 
                           loyalty_tier, effective_date, is_current)
SELECT customer_name, email, 'Kuala Lumpur', country, signup_date,
       loyalty_tier, CURRENT_DATE, TRUE
FROM dim_customers WHERE customer_id = 1;
```

> 🧠 **SCD Type 2 is the industry standard.** Every major data warehouse uses this. dbt has built-in SCD Type 2 support. Memorize this pattern — it comes up in interviews constantly.

---

### Task 4: SCD Type 3 — Add Column (Previous Value Only) (10 min)

```sql
-- 🔹 Type 3: Keep current + previous in same row
ALTER TABLE dim_customers ADD COLUMN previous_city VARCHAR(50);
ALTER TABLE dim_customers ADD COLUMN city_changed_date DATE;

UPDATE dim_customers 
SET previous_city = city, 
    city = 'Kuala Lumpur', 
    city_changed_date = CURRENT_DATE
WHERE customer_name = 'Alice Tan';

-- ✅ Simple, keeps one level of history
-- ❌ Only one historical value, can't track multiple changes
-- Use when: you only need current + previous (rare)
```

**Summary:**

| Type | How | History | Use When |
|------|-----|---------|----------|
| Type 1 | Overwrite | None | Fixing errors, rarely-analyzed fields |
| Type 2 | New row | Full | Most dimensions (industry standard) |
| Type 3 | Extra column | Previous only | Need just one level of history |

**🔥 Interview question practice:**

"How would you handle a customer changing their loyalty tier from Silver to Gold?"

<details>
<summary>📖 Answer framework</summary>

"I'd use SCD Type 2 — add effective/expiry dates and is_current flag. When the tier changes, expire the old record and insert a new row with the new tier. This way we can analyze: 'What tier was this customer during Q1?' and 'How many Gold-tier customers did we have in March?' which is critical for loyalty program analysis."
</details>

---

## 🔥 BLOCK 2: Snowflake Schema + ERD Tools (Afternoon, ~1.5 hours)

---

### Task 5: Snowflake Schema vs Star Schema (20 min)

```
STAR SCHEMA (what we built yesterday):
fact → dim_customers (flat, denormalized)
     → dim_products (flat, denormalized)

SNOWFLAKE SCHEMA (more normalized):
fact → dim_customers → dim_cities → dim_countries
     → dim_products → dim_categories → dim_departments

The dimension tables themselves have sub-dimensions (like snowflake branches)
```

```sql
-- 🔹 Snowflake example: normalize the location

-- Instead of city + country directly in dim_customers:
CREATE TABLE dim_countries (
    country_id SERIAL PRIMARY KEY,
    country_name VARCHAR(50)
);

CREATE TABLE dim_cities (
    city_id SERIAL PRIMARY KEY,
    city_name VARCHAR(50),
    country_id INTEGER REFERENCES dim_countries(country_id)
);

CREATE TABLE dim_customers_snowflake (
    customer_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100),
    city_id INTEGER REFERENCES dim_cities(city_id),  -- references sub-dimension
    email VARCHAR(100),
    loyalty_tier VARCHAR(20)
);
```

| Aspect | Star | Snowflake |
|--------|------|-----------|
| Structure | Flat dimensions | Normalized dimensions |
| Joins needed | Fewer (1 level) | More (2+ levels) |
| Query simplicity | Simpler | More complex |
| Storage | More (redundant) | Less (normalized) |
| Performance | Faster (fewer joins) | Slower (more joins) |
| Industry use | Standard for analytics | Rare in modern warehouses |

> 🧠 **In practice, almost everyone uses star schema.** Storage is cheap, query speed matters more. Snowflake is mainly an interview topic.

---

### Task 6: Create Professional ERD with dbdiagram.io (30 min)

1. Go to https://dbdiagram.io
2. Create a free account
3. Use their DSL (simple text syntax) to create your food delivery schema:

```
// Copy this into dbdiagram.io

Table dim_customers {
  customer_id integer [pk, increment]
  customer_name varchar
  email varchar
  phone varchar
  city varchar
  country varchar
  signup_date date
  loyalty_tier varchar
  is_active boolean
}

Table dim_restaurants {
  restaurant_id integer [pk, increment]
  restaurant_name varchar
  cuisine_type varchar
  city varchar
  area varchar
  rating decimal
  price_range varchar
  is_active boolean
}

Table dim_dishes {
  dish_id integer [pk, increment]
  dish_name varchar
  restaurant_id integer
  category varchar
  cuisine_type varchar
  is_vegetarian boolean
  price decimal
}

Table dim_date {
  date_id integer [pk]
  full_date date
  day_of_week varchar
  month_name varchar
  quarter varchar
  year integer
  is_weekend boolean
}

Table dim_time {
  time_id integer [pk, increment]
  hour integer
  minute integer
  time_of_day varchar
  meal_period varchar
}

Table dim_delivery_areas {
  area_id integer [pk, increment]
  area_name varchar
  city varchar
  postal_code varchar
  zone varchar
}

Table fact_order_items {
  order_item_id integer [pk, increment]
  order_id integer
  customer_id integer
  restaurant_id integer
  dish_id integer
  date_id integer
  time_id integer
  delivery_area_id integer
  quantity integer
  unit_price decimal
  total_amount decimal
  discount_amount decimal
  delivery_fee decimal
  preparation_minutes integer
  delivery_minutes integer
}

Ref: fact_order_items.customer_id > dim_customers.customer_id
Ref: fact_order_items.restaurant_id > dim_restaurants.restaurant_id
Ref: fact_order_items.dish_id > dim_dishes.dish_id
Ref: fact_order_items.date_id > dim_date.date_id
Ref: fact_order_items.time_id > dim_time.time_id
Ref: fact_order_items.delivery_area_id > dim_delivery_areas.area_id
Ref: dim_dishes.restaurant_id > dim_restaurants.restaurant_id
```

4. Click "Export" → save as PNG
5. Save this image in your portfolio project's `docs/` folder

---

### Task 7: Design a SaaS Data Model (30 min)

**Scenario:** A project management SaaS (like Trello/Asana) needs a data model.

Design a star schema for analyzing:
- Task completion rates by team, by project, by user
- Time tracking (hours logged per task)
- Sprint velocity (tasks completed per sprint)

**Identify the tables yourself:**

<details>
<summary>📖 Reference Design</summary>

```
DIMENSIONS:
- dim_users (user_id, name, email, role, team, department, hire_date)
- dim_projects (project_id, name, type, status, start_date, team_id)
- dim_tasks (task_id, title, priority, complexity, task_type)
- dim_sprints (sprint_id, name, start_date, end_date, sprint_number)
- dim_date (same as before)

FACT TABLES (can have multiple!):
- fact_task_events (one row per task status change)
  - task_event_id, task_id, user_id, project_id, sprint_id, date_id
  - old_status, new_status, hours_spent

- fact_time_entries (one row per time log)
  - time_entry_id, task_id, user_id, project_id, date_id
  - hours_logged, billable_flag

KEY METRICS THIS ENABLES:
- Task completion rate = completed tasks / total tasks per sprint
- Velocity = story points completed per sprint
- Time per task = avg hours from start to done
- Bottleneck = avg time in each status
```
</details>

---

## 🌙 BLOCK 3: Practice + Consolidation (Evening, ~1.5 hours)

---

### Task 8: Implement SCD Type 2 in SQL (30 min)

Add SCD Type 2 tracking to the food delivery customers:

```sql
-- Step 1: Add SCD columns (you did this in Task 3)
-- Step 2: Create a function that handles the change automatically

-- Simulate Alice's loyalty tier changing from Gold to Platinum
-- and Bob moving from KL to Singapore

-- Write the full SQL for both changes:
-- a) Expire old records
-- b) Insert new records with new values
-- c) Verify by querying current vs historical data

-- Then write these queries:
-- d) What was Alice's tier on 2024-02-01?
-- e) How many customers were in each city on 2024-01-01 vs 2024-06-01?
-- f) List all changes that happened in March 2024
```

---

### Task 9: Review — Data Modeling Interview Questions (30 min)

Answer these in your own words:

1. "Explain star schema and when you'd use it"
2. "What's the difference between star and snowflake schema?"
3. "How do you handle slowly changing dimensions?"
4. "What's a surrogate key and why use one?"
5. "What's the difference between OLTP and OLAP data models?"

<details>
<summary>📖 Reference Answers (compare with yours)</summary>

1. **Star schema** is a data warehouse design with one central fact table surrounded by dimension tables. Used for analytics/OLAP. Fact tables store numeric measures, dimensions store descriptive attributes. It's denormalized for query speed.

2. **Star** has flat, denormalized dimensions (one join level). **Snowflake** has normalized dimensions with sub-dimensions (multiple join levels). Star is more common because storage is cheap and fewer joins = faster queries.

3. **SCD** handles changing dimension data. Type 1 overwrites (no history). Type 2 adds new rows with effective/expiry dates (full history). Type 3 adds a column for previous value. Type 2 is the industry standard.

4. **Surrogate key** is an artificial primary key (usually auto-incrementing integer), independent of business data. Use it because business keys can change (email, phone), and surrogate keys never change — which is essential for SCD Type 2 and referential integrity.

5. **OLTP** (transactional): normalized (3NF), optimized for writes, one row per transaction (e.g., production database). **OLAP** (analytical): denormalized (star schema), optimized for reads, aggregated queries (e.g., data warehouse).
</details>

---

### Task 10: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 16 — Advanced Data Modeling
Date: 2026-05-30

## Key Concepts
- SCD Type 1: Overwrite (no history)
- SCD Type 2: New row with dates (full history) ← INDUSTRY STANDARD
- SCD Type 3: Extra column (previous value only)
- Snowflake schema: normalized dimensions (rare in practice)
- ERD tools: dbdiagram.io for professional diagrams
- Surrogate keys: always use them
- OLTP vs OLAP: transactional vs analytical design

## Interview Questions I Can Answer
1. Star schema ✅
2. Star vs snowflake ✅
3. SCD types ✅
4. Surrogate keys ✅
5. OLTP vs OLAP ✅

## Tomorrow: dbt Fundamentals
```

```bash
git add .
git commit -m "Day 16: SCD types, snowflake schema, ERD tools, data modeling interview prep"
git push
```

---

## ✅ Day 16 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Can explain SCD Type 1, 2, 3 | ☐ |
| 2 | Implemented SCD Type 2 in SQL | ☐ |
| 3 | Queried historical data with effective/expiry dates | ☐ |
| 4 | Understand star vs snowflake schema | ☐ |
| 5 | Created ERD on dbdiagram.io | ☐ |
| 6 | Designed SaaS data model | ☐ |
| 7 | Can answer 5 data modeling interview questions | ☐ |
| 8 | Day 16 notes pushed to GitHub | ☐ |
