# 📅 Day 25 — Sunday, 8 June 2026
# 📊 Slowly Changing Dimensions (SCD Types 1, 2, 3)

---

## 🎯 Today's Goal

Dimensions change. Customers move cities, restaurants change cuisine types, products get re-priced. **Slowly Changing Dimensions (SCD)** are how data warehouses track these changes. Today you'll master the three main types and know exactly when to use each.

**Why this matters:** "How do you handle slowly changing dimensions?" is one of the most common data engineering interview questions. In real jobs, getting SCD wrong means your reports show wrong history.

---

## ☀️ Morning Block (2 hours): SCD Fundamentals

### Concept 1: The Problem SCD Solves

```
Customer Alice lives in Singapore in January.
In March, she moves to Kuala Lumpur.

Her orders:
- Jan 15: $25 (lives in SG)
- Feb 20: $30 (lives in SG)
- Mar 10: $20 (lives in KL now)
- Apr 05: $45 (lives in KL)

Question: "What was our revenue by city in Q1?"

If we OVERWRITE her city to KL:
- SG revenue = $0 (wrong! She was in SG for Jan-Feb)
- KL revenue = $120 (wrong! She wasn't in KL for Jan-Feb)

If we TRACK HISTORY:
- SG revenue = $55 (correct for Jan-Feb)
- KL revenue = $65 (correct for Mar onwards)
```

**SCD is about choosing how to handle this.**

### Concept 2: SCD Type 1 — Overwrite (No History)

```sql
-- Simply overwrite the old value. No history preserved.
-- The simplest approach.

-- dim_customer (Type 1 — no history columns needed)
CREATE TABLE dim_customer_scd1 (
    customer_key SERIAL PRIMARY KEY,
    customer_id INT NOT NULL UNIQUE,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    city VARCHAR(50),
    postal_code VARCHAR(10),
    customer_segment VARCHAR(20),
    is_active BOOLEAN DEFAULT TRUE
);

-- Load: Insert new + Update changed
INSERT INTO dim_customer_scd1 (customer_id, customer_name, email, city, postal_code)
SELECT s.customer_id, s.customer_name, s.email, s.city, s.postal_code
FROM source_customers s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer_scd1 d WHERE d.customer_id = s.customer_id
);

-- Update changed records (overwrite)
UPDATE dim_customer_scd1 d
SET 
    customer_name = s.customer_name,
    email = s.email,
    city = s.city,
    postal_code = s.postal_code
FROM source_customers s
WHERE d.customer_id = s.customer_id
AND (
    d.customer_name IS DISTINCT FROM s.customer_name
    OR d.email IS DISTINCT FROM s.email
    OR d.city IS DISTINCT FROM s.city
    OR d.postal_code IS DISTINCT FROM s.postal_code
);
```

**Pros:** Simple, fast, small table
**Cons:** Lost history — can't answer "what was Alice's city in January?"
**When to use:** Corrections (typos), attributes you don't need history for (email updates)

### Concept 3: SCD Type 2 — Add New Row (Full History)

```sql
-- Add a NEW row with new values. Keep old row. Track validity dates.
-- This is the MOST COMMON approach in data warehousing.

CREATE TABLE dim_customer_scd2 (
    customer_key SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,          -- natural key (NOT unique here!)
    customer_name VARCHAR(100),
    email VARCHAR(150),
    city VARCHAR(50),
    postal_code VARCHAR(10),
    customer_segment VARCHAR(20),
    is_active BOOLEAN DEFAULT TRUE,
    -- SCD2 tracking columns:
    valid_from TIMESTAMP NOT NULL DEFAULT NOW(),
    valid_to TIMESTAMP NOT NULL DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN NOT NULL DEFAULT TRUE,
    -- Composite unique on natural key + valid_from to prevent duplicates
    UNIQUE (customer_id, valid_from)
);

-- Index for looking up current records
CREATE INDEX idx_dim_cust_scd2_current ON dim_customer_scd2(customer_id) WHERE is_current = TRUE;
CREATE INDEX idx_dim_cust_scd2_natural ON dim_customer_scd2(customer_id, valid_to);

-- SCD2 ETL: Detect changes and manage history

-- Step 1: Expire old records that have changed
UPDATE dim_customer_scd2 d
SET 
    valid_to = NOW(),
    is_current = FALSE
FROM source_customers s
WHERE d.customer_id = s.customer_id
AND d.is_current = TRUE
AND (
    d.customer_name IS DISTINCT FROM s.customer_name
    OR d.email IS DISTINCT FROM s.email
    OR d.city IS DISTINCT FROM s.city
    OR d.postal_code IS DISTINCT FROM s.postal_code
);

-- Step 2: Insert new current records for changed + brand new customers
INSERT INTO dim_customer_scd2 (customer_id, customer_name, email, city, postal_code, valid_from, valid_to, is_current)
SELECT 
    s.customer_id, s.customer_name, s.email, s.city, s.postal_code,
    NOW(), '9999-12-31 23:59:59', TRUE
FROM source_customers s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer_scd2 d 
    WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
    AND d.customer_name = s.customer_name
    AND d.email = s.email
    AND d.city = s.city
    AND d.postal_code = s.postal_code
);
```

**How it looks in practice:**

| customer_key | customer_id | name | city | valid_from | valid_to | is_current |
|---|---|---|---|---|---|---|
| 1 | 100 | Alice | Singapore | 2026-01-01 | 2026-03-01 | FALSE |
| 5 | 100 | Alice | Kuala Lumpur | 2026-03-01 | 9999-12-31 | TRUE |

**Fact table joins to customer_key (surrogate key), so:**
- Orders from Jan-Feb → customer_key=1 → city=Singapore ✅
- Orders from Mar onwards → customer_key=5 → city=Kuala Lumpur ✅

**Pros:** Full history preserved, accurate point-in-time reporting
**Cons:** Table grows (multiple rows per entity), ETL is more complex
**When to use:** Almost always for key attributes like city, segment, category

### Concept 4: SCD Type 3 — Add Column (Previous Value Only)

```sql
-- Add a column to track the PREVIOUS value. Only one level of history.

CREATE TABLE dim_customer_scd3 (
    customer_key SERIAL PRIMARY KEY,
    customer_id INT NOT NULL UNIQUE,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    city VARCHAR(50),
    previous_city VARCHAR(50),       -- the old value
    city_changed_date TIMESTAMP,     -- when it changed
    postal_code VARCHAR(10),
    customer_segment VARCHAR(20),
    is_active BOOLEAN DEFAULT TRUE
);

-- Load: Track previous value on update
UPDATE dim_customer_scd3 d
SET 
    previous_city = d.city,
    city = s.city,
    city_changed_date = NOW()
FROM source_customers s
WHERE d.customer_id = s.customer_id
AND d.city IS DISTINCT FROM s.city;
```

**How it looks:**

| customer_id | name | city | previous_city | city_changed_date |
|---|---|---|---|---|
| 100 | Alice | Kuala Lumpur | Singapore | 2026-03-01 |

**Pros:** Simple, small table, some history
**Cons:** Only ONE previous value — if Alice moves again, Singapore is lost
**When to use:** When you only need "current vs previous" comparison (rare)

### Concept 5: Comparison Table

| Feature | Type 1 | Type 2 | Type 3 |
|---------|--------|--------|--------|
| History | None | Full | Previous only |
| Table size | Small | Grows | Small |
| ETL complexity | Simple | Complex | Medium |
| Point-in-time reporting | ❌ | ✅ | ❌ (limited) |
| Most common use | Corrections | Core dimensions | Before/after analysis |
| Interview answer | "Overwrite" | "New row + dates" | "Previous value column" |

---

### 🏋️ Morning Exercises (20 questions)

1. Explain SCD Type 2 in your own words to a non-technical stakeholder. Use a GrabFood example.

2. A restaurant changes its cuisine_type from "Chinese" to "Fusion". Which SCD type would you use? Why?

3. A customer's email has a typo that gets corrected. Which SCD type is most appropriate? Why?

4. Design a complete `dim_customer` table using SCD Type 2. Include all tracking columns. Write the CREATE TABLE statement.

5. Write the complete SCD Type 2 ETL for `dim_restaurant`:
    - Detect changes in restaurant_name, cuisine_type, city, rating
    - Expire old records
    - Insert new current records
    - Handle brand new restaurants

6. A product's price changes from RM12.50 to RM15.00. Using SCD Type 2, show what the `dim_item` table looks like before and after the change.

7. Write a query to find all customers who have moved cities (using SCD Type 2). Show their old city, new city, and when they moved.

8. Using SCD Type 3, write a query that compares current vs previous city for each customer. How many moved in the last 30 days?

9. **Hybrid approach:** Design a dimension where `email` uses Type 1 (overwrite) but `city` uses Type 2 (new row). Is this valid? Write the ETL.

10. Write a query using SCD Type 2 data that answers: "What was our revenue by city in Q1 2026?" (historical, point-in-time correct).

11. How many rows does `dim_customer_scd2` have if there are 1,000 unique customers and each customer has changed city an average of 2 times?

12. Write a query to find the customer with the MOST history rows in an SCD Type 2 table. What business insight does this give you?

13. Design an SCD Type 2 load for `dim_item` that tracks price changes. A price change should create a new row. Show the ETL SQL.

14. Write a query to reconstruct the full history of a customer's city changes from SCD Type 2 data. Output should show: from_city, to_city, change_date.

15. **Edge case:** What happens if the same customer_id appears twice in the source data with different values? How do you handle this in SCD Type 2?

16. Write a stored procedure `sp_scd2_customer_load()` that performs the full SCD Type 2 load for dim_customer. Include a parameter for the batch timestamp.

17. A stakeholder asks: "Can you show me what Alice's segment was on June 1st, even though it changed in July?" Write the query using SCD Type 2.

18. Compare the storage cost: 1 million customers, each changes city once. How many rows in SCD Type 1 vs Type 2? Estimate disk usage.

19. Write a data quality check for SCD Type 2: verify that each customer_id has exactly one row with `is_current = TRUE`.

20. **Challenge:** Design a complete SCD strategy for a `dim_employee` table with these attributes:
    - name (Type 1 — corrections only)
    - department (Type 2 — track transfers)
    - salary (Type 2 — track changes for audit)
    - manager (Type 2 — track reporting line changes)
    - email (Type 1 — overwrite)
    Write the CREATE TABLE and ETL.

---

### ✅ Morning Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1:** "Imagine you order GrabFood from your home in Singapore. Then you move to KL. If we only keep your latest address, all your old Singapore orders would look like they came from KL — that's wrong. So instead, we keep a record that says 'from January to March, Alice lived in Singapore. From March onwards, KL.' This way, when we look at Singapore sales in February, your orders are correctly counted."

**Answer 2:** Type 2 — cuisine type is an important analytical attribute. You'd want to know "what was the revenue for Chinese food in January?" even after the restaurant rebrands to Fusion in March.

**Answer 3:** Type 1 — a typo correction isn't a meaningful business change. Overwriting is fine since there's no analytical value in tracking the typo.

**Answer 4:**
```sql
CREATE TABLE dim_customer (
    customer_key SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    city VARCHAR(50),
    postal_code VARCHAR(10),
    customer_segment VARCHAR(20),
    is_active BOOLEAN DEFAULT TRUE,
    valid_from TIMESTAMP NOT NULL DEFAULT NOW(),
    valid_to TIMESTAMP NOT NULL DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE (customer_id, valid_from)
);
```

**Answer 5:**
```sql
-- Step 1: Expire changed records
UPDATE dim_restaurant d
SET valid_to = NOW(), is_current = FALSE
FROM source_restaurants s
WHERE d.restaurant_id = s.restaurant_id
AND d.is_current = TRUE
AND (
    d.restaurant_name IS DISTINCT FROM s.restaurant_name
    OR d.cuisine_type IS DISTINCT FROM s.cuisine_type
    OR d.city IS DISTINCT FROM s.city
    OR d.rating IS DISTINCT FROM s.rating
);

-- Step 2: Insert new current records for changed + new
INSERT INTO dim_restaurant (restaurant_id, restaurant_name, cuisine_type, city, postal_code, rating, is_active, valid_from, valid_to, is_current)
SELECT 
    s.restaurant_id, s.restaurant_name, s.cuisine_type, s.city, s.postal_code, 
    s.rating, s.is_active, NOW(), '9999-12-31 23:59:59', TRUE
FROM source_restaurants s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_restaurant d 
    WHERE d.restaurant_id = s.restaurant_id 
    AND d.is_current = TRUE
    AND d.restaurant_name = s.restaurant_name
    AND d.cuisine_type = s.cuisine_type
    AND d.city = s.city
    AND d.rating = s.rating
);
```

**Answer 6:**

Before:
| item_key | item_id | dish_name | price | valid_from | valid_to | is_current |
|---|---|---|---|---|---|---|
| 10 | 50 | Nasi Lemak | 12.50 | 2026-01-01 | 9999-12-31 | TRUE |

After price change:
| item_key | item_id | dish_name | price | valid_from | valid_to | is_current |
|---|---|---|---|---|---|---|
| 10 | 50 | Nasi Lemak | 12.50 | 2026-01-01 | 2026-06-01 | FALSE |
| 15 | 50 | Nasi Lemak | 15.00 | 2026-06-01 | 9999-12-31 | TRUE |

Orders before June 1 join to item_key=10 (RM12.50), orders after join to item_key=15 (RM15.00).

**Answer 7:**
```sql
SELECT 
    d_old.customer_id,
    d_old.customer_name,
    d_old.city AS old_city,
    d_new.city AS new_city,
    d_new.valid_from AS change_date
FROM dim_customer d_new
JOIN dim_customer d_old 
    ON d_new.customer_id = d_old.customer_id
    AND d_old.valid_to = d_new.valid_from
WHERE d_new.city != d_old.city
ORDER BY d_new.valid_from DESC;
```

**Answer 8:**
```sql
SELECT 
    customer_id, customer_name,
    city AS current_city,
    previous_city,
    city_changed_date
FROM dim_customer_scd3
WHERE previous_city IS NOT NULL
AND city_changed_date >= NOW() - INTERVAL '30 days';
```

**Answer 9:** Yes, this is very common! Many production dimensions use hybrid SCD:
```sql
CREATE TABLE dim_customer_hybrid (
    customer_key SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    customer_name VARCHAR(100),
    email VARCHAR(150),              -- Type 1: just overwrite
    city VARCHAR(50),                -- Type 2: track with new row
    postal_code VARCHAR(10),
    valid_from TIMESTAMP DEFAULT NOW(),
    valid_to TIMESTAMP DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN DEFAULT TRUE,
    UNIQUE (customer_id, valid_from)
);

-- ETL:
-- Only create new row when TRACKED columns change (city)
-- Always update Type 1 columns (email)
UPDATE dim_customer_hybrid d
SET email = s.email
FROM source_customers s
WHERE d.customer_id = s.customer_id
AND d.email IS DISTINCT FROM s.email
AND d.is_current = TRUE;

-- Type 2: expire + insert only when city changes
UPDATE dim_customer_hybrid d
SET valid_to = NOW(), is_current = FALSE
FROM source_customers s
WHERE d.customer_id = s.customer_id
AND d.is_current = TRUE
AND d.city IS DISTINCT FROM s.city;

INSERT INTO dim_customer_hybrid (customer_id, customer_name, email, city, postal_code)
SELECT s.customer_id, s.customer_name, s.email, s.city, s.postal_code
FROM source_customers s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer_hybrid d 
    WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
    AND d.city = s.city
);
```

**Answer 10:**
```sql
SELECT 
    d.city,
    SUM(f.net_amount) AS q1_revenue
FROM fact_sales f
JOIN dim_customer d ON f.customer_key = d.customer_key
JOIN dim_date dd ON f.date_key = dd.date_key
WHERE dd.full_date BETWEEN '2026-01-01' AND '2026-03-31'
-- The join to customer_key ensures point-in-time correctness
-- Each fact row joins to the customer row that was current WHEN the order happened
GROUP BY d.city
ORDER BY q1_revenue DESC;
```

**Answer 11:** 1,000 unique customers × (1 original + 2 changes) = **3,000 rows**.

**Answer 12:**
```sql
SELECT customer_id, customer_name, COUNT(*) AS history_rows
FROM dim_customer
GROUP BY customer_id, customer_name
ORDER BY history_rows DESC
LIMIT 1;
```
A customer with many changes might be a business that moves frequently, or a data quality issue.

**Answer 13:**
```sql
-- Expire old item record when price changes
UPDATE dim_item d
SET valid_to = NOW(), is_current = FALSE
FROM source_menu_items s
WHERE d.item_id = s.item_id
AND d.is_current = TRUE
AND d.price IS DISTINCT FROM s.price;

-- Insert new current record
INSERT INTO dim_item (item_id, restaurant_id, dish_name, category, cuisine_type, price, is_vegetarian, spice_level, valid_from, valid_to, is_current)
SELECT s.item_id, s.restaurant_id, s.dish_name, s.category, s.cuisine_type, s.price, s.is_vegetarian, s.spice_level, NOW(), '9999-12-31 23:59:59', TRUE
FROM source_menu_items s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_item d 
    WHERE d.item_id = s.item_id AND d.is_current = TRUE AND d.price = s.price
);
```

**Answer 14:**
```sql
SELECT 
    d1.city AS from_city,
    d2.city AS to_city,
    d2.valid_from AS change_date
FROM dim_customer d2
JOIN dim_customer d1 
    ON d2.customer_id = d1.customer_id
    AND d1.valid_to = d2.valid_from
WHERE d2.city != d1.city
ORDER BY d2.customer_id, d2.valid_from;
```

**Answer 15:** This is a common issue. Options:
1. Pick the latest record (use `MAX(updated_at)`)
2. Flag as data quality issue and reject
3. Use a business rule (e.g., prefer the record with more recent timestamp)
Best practice: deduplicate source data BEFORE loading into the dimension.

**Answer 16:**
```sql
CREATE OR REPLACE PROCEDURE sp_scd2_customer_load(p_batch_ts TIMESTAMP DEFAULT NOW())
LANGUAGE plpgsql AS $$
DECLARE
    v_expired INT;
    v_inserted INT;
BEGIN
    -- Expire changed records
    UPDATE dim_customer d
    SET valid_to = p_batch_ts, is_current = FALSE
    FROM source_customers s
    WHERE d.customer_id = s.customer_id
    AND d.is_current = TRUE
    AND (
        d.customer_name IS DISTINCT FROM s.customer_name
        OR d.city IS DISTINCT FROM s.city
        OR d.customer_segment IS DISTINCT FROM s.customer_segment
    );
    GET DIAGNOSTICS v_expired = ROW_COUNT;
    RAISE NOTICE 'Expired % records', v_expired;
    
    -- Insert new current records
    INSERT INTO dim_customer (customer_id, customer_name, email, city, postal_code, customer_segment, is_active, valid_from, valid_to, is_current)
    SELECT s.customer_id, s.customer_name, s.email, s.city, s.postal_code, s.customer_segment, TRUE, p_batch_ts, '9999-12-31 23:59:59', TRUE
    FROM source_customers s
    WHERE NOT EXISTS (
        SELECT 1 FROM dim_customer d 
        WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
        AND d.customer_name = s.customer_name
        AND d.city = s.city
        AND d.customer_segment = s.customer_segment
    );
    GET DIAGNOSTICS v_inserted = ROW_COUNT;
    RAISE NOTICE 'Inserted % new records', v_inserted;
END;
$$;
```

**Answer 17:**
```sql
SELECT customer_name, city, customer_segment
FROM dim_customer
WHERE customer_id = (SELECT customer_id FROM dim_customer WHERE customer_name = 'Alice' LIMIT 1)
AND valid_from <= '2026-06-01'
AND valid_to > '2026-06-01'
LIMIT 1;
```

**Answer 18:**
- Type 1: 1 million rows × ~200 bytes/row ≈ **200 MB**
- Type 2: 2 million rows × ~250 bytes/row (extra tracking columns) ≈ **500 MB**
- Difference is manageable — storage is cheap, history is valuable

**Answer 19:**
```sql
SELECT customer_id, COUNT(*) AS current_count
FROM dim_customer
WHERE is_current = TRUE
GROUP BY customer_id
HAVING COUNT(*) != 1;
-- Should return 0 rows. Any results = data quality issue.
```

**Answer 20:**
```sql
CREATE TABLE dim_employee (
    employee_key SERIAL PRIMARY KEY,
    employee_id INT NOT NULL,
    employee_name VARCHAR(100),       -- Type 1
    email VARCHAR(150),                -- Type 1
    department VARCHAR(50),            -- Type 2
    salary DECIMAL(10,2),              -- Type 2
    manager_id INT,                    -- Type 2
    manager_name VARCHAR(100),         -- denormalized for convenience
    valid_from TIMESTAMP DEFAULT NOW(),
    valid_to TIMESTAMP DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN DEFAULT TRUE
);

-- ETL Procedure
CREATE OR REPLACE PROCEDURE sp_load_dim_employee()
LANGUAGE plpgsql AS $$
BEGIN
    -- Step 1: Update Type 1 attributes on ALL current rows
    UPDATE dim_employee d
    SET employee_name = s.employee_name, email = s.email
    FROM source_employees s
    WHERE d.employee_id = s.employee_id
    AND d.is_current = TRUE
    AND (d.employee_name IS DISTINCT FROM s.employee_name OR d.email IS DISTINCT FROM s.email);
    
    -- Step 2: Expire rows where Type 2 attributes changed
    UPDATE dim_employee d
    SET valid_to = NOW(), is_current = FALSE
    FROM source_employees s
    WHERE d.employee_id = s.employee_id
    AND d.is_current = TRUE
    AND (
        d.department IS DISTINCT FROM s.department
        OR d.salary IS DISTINCT FROM s.salary
        OR d.manager_id IS DISTINCT FROM s.manager_id
    );
    
    -- Step 3: Insert new current rows for changed + new employees
    INSERT INTO dim_employee (employee_id, employee_name, email, department, salary, manager_id)
    SELECT s.employee_id, s.employee_name, s.email, s.department, s.salary, s.manager_id
    FROM source_employees s
    WHERE NOT EXISTS (
        SELECT 1 FROM dim_employee d 
        WHERE d.employee_id = s.employee_id AND d.is_current = TRUE
        AND d.department = s.department
        AND d.salary = s.salary
        AND d.manager_id = s.manager_id
    );
END;
$$;
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Advanced SCD Patterns

### Concept 6: SCD Type 2 with Multiple Changed Columns

```sql
-- What if BOTH city AND segment change at the same time?
-- SCD Type 2 handles this naturally — one new row captures all changes.

-- Example: Alice moves to KL AND gets upgraded to VIP
-- Old row: customer_key=5, city=Singapore, segment=Regular, is_current=FALSE
-- New row: customer_key=8, city=KL, segment=VIP, is_current=TRUE

-- The ETL from Concept 3 already handles this — any change in tracked
-- columns triggers a new row with ALL current values.
```

### Concept 7: SCD Type 2 Lookup for Fact Table Loading

```sql
-- When loading facts, you need the correct surrogate key for the order date.
-- The customer's state at the TIME of the order matters.

-- WRONG: Just joining on customer_id
-- This might pick up the wrong version of the customer!

-- RIGHT: Join on customer_id AND match the order date to valid_from/valid_to
INSERT INTO fact_sales (date_key, customer_key, restaurant_key, item_key, quantity, unit_price)
SELECT 
    dd.date_key,
    dc.customer_key,  -- surrogate key that was valid on order date
    dr.restaurant_key,
    di.item_key,
    oi.quantity,
    oi.unit_price
FROM source_order_items oi
JOIN source_orders o ON oi.order_id = o.order_id
JOIN dim_date dd ON dd.full_date = DATE(o.order_date)
JOIN dim_customer dc ON dc.customer_id = o.customer_id
    AND dc.valid_from <= o.order_date
    AND dc.valid_to > o.order_date
JOIN dim_restaurant dr ON dr.restaurant_id = o.restaurant_id
    AND dr.is_current = TRUE
JOIN dim_item di ON di.item_id = oi.item_id;
```

**💡 Critical:** For SCD Type 2, the fact table join must match the time window. If you just join on `is_current = TRUE`, you'll assign ALL historical orders to the current version of the customer — destroying the point of SCD Type 2!

### Concept 8: SCD Type 6 (Hybrid 1+2+3)

```sql
-- Some teams use "Type 6" which combines:
-- Type 1: current value (overwritten)
-- Type 2: full history via rows
-- Type 3: previous value column (for quick comparison)

CREATE TABLE dim_customer_scd6 (
    customer_key SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    city VARCHAR(50),               -- current value (Type 1/2)
    previous_city VARCHAR(50),      -- previous value (Type 3)
    customer_segment VARCHAR(20),
    valid_from TIMESTAMP DEFAULT NOW(),
    valid_to TIMESTAMP DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN DEFAULT TRUE
);
```

### Concept 9: SCD Type 4 — Mini-Dimension

```sql
-- For high-cardinality attributes that change frequently,
-- separate them into their own "mini-dimension"

-- Main dimension (slowly changing)
CREATE TABLE dim_customer (
    customer_key SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    signup_date DATE,
    valid_from TIMESTAMP DEFAULT NOW(),
    valid_to TIMESTAMP DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN DEFAULT TRUE
);

-- Mini-dimension (frequently changing attributes)
CREATE TABLE dim_customer_profile (
    profile_key SERIAL PRIMARY KEY,
    city VARCHAR(50),
    customer_segment VARCHAR(20),
    lifetime_value_band VARCHAR(10),   -- '0-100', '100-500', '500+'
    days_since_last_order_band VARCHAR(10),
    is_active BOOLEAN
);

-- Fact table references BOTH
CREATE TABLE fact_sales (
    sales_key SERIAL PRIMARY KEY,
    date_key INT,
    customer_key INT,       -- → dim_customer (slowly changing)
    profile_key INT,        -- → dim_customer_profile (frequently changing)
    restaurant_key INT,
    item_key INT,
    quantity INT,
    net_amount DECIMAL(10,2)
);
```

### Concept 10: Practical dbt SCD2 Macro

```sql
-- When using dbt (which you learned in Week 3), SCD Type 2 is common.
-- Here's a simplified pattern:

-- models/staging/stg_customers.sql
{{ config(
    materialized='incremental',
    unique_key='customer_id'
) }}

WITH source AS (
    SELECT * FROM {{ source('raw', 'customers') }}
),
deduped AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) AS rn
    FROM source
)
SELECT 
    customer_id,
    customer_name,
    email,
    city,
    customer_segment
FROM deduped
WHERE rn = 1
```

---

### 🏋️ Afternoon Exercises (15 questions)

21. Write an SCD Type 2 ETL for `dim_restaurant` that detects changes in `cuisine_type` AND `rating`. Show what the table looks like after a restaurant changes cuisine from "Chinese" to "Fusion".

22. Write the fact table load SQL that correctly looks up the SCD Type 2 surrogate key for `dim_customer` based on the order timestamp.

23. A customer's segment changes from "New" to "Regular" to "VIP" over 3 months. Show the dim_customer table with all 3 rows. Write a query showing their segment progression.

24. Write a data quality query that finds gaps in SCD Type 2 history (where valid_to of one row doesn't match valid_from of the next row for the same customer_id).

25. Design an SCD Type 2 load for `dim_employee` that tracks department transfers. Write the complete ETL with error handling.

26. **Mini-dimension design:** Design `dim_customer_profile` as a mini-dimension with these frequently-changing attributes: city, segment, lifetime_value_band, activity_status. Populate with sample combinations.

27. Write a query to "reconstruct" a customer's history: for each day in 2026, show what their city and segment were on that day (using SCD Type 2 data).

28. Compare SCD Type 2 vs Type 4 (mini-dimension) for tracking a customer's `lifetime_value` which changes with every order. Which approach is better? Why?

29. Write a dbt model that implements SCD Type 2 for `dim_product`. Use `snapshots` if possible (dbt built-in feature from Week 3).

30. Design a real-time SCD Type 2 pipeline: orders come in every second, and customer attributes may change at any time. How do you ensure the correct surrogate key is always used?

31. Write a stored procedure that performs SCD Type 2 load AND logs every change to an audit table (`scd_audit_log`).

32. **Edge case:** A customer's city changes BACK to a previous value (Singapore → KL → Singapore). How does SCD Type 2 handle this? Show the table state.

33. Write a query to count how many SCD Type 2 rows were created per month for `dim_customer`. This tells you how volatile the customer data is.

34. Design an SCD strategy for a `dim_product` table in an e-commerce platform (like Shopee). Decide SCD type for each attribute: name, category, price, description, brand, stock_count.

35. **Challenge:** Build a complete SCD Type 2 pipeline in PostgreSQL:
    - Create `source_customers` and `dim_customer` tables
    - Insert 10 customers
    - Simulate 3 rounds of changes (some customers change city, some change segment)
    - After each round, run the SCD Type 2 ETL
    - Write 3 analytical queries that demonstrate point-in-time correctness
    - Show that the same customer_id can have different attributes at different times

---

### ✅ Afternoon Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 21:**
```sql
-- Before: Restaurant 42, "Golden Dragon", Chinese, rating 4.2
-- After cuisine change to "Fusion":

-- Expire old row
UPDATE dim_restaurant SET valid_to = NOW(), is_current = FALSE
WHERE restaurant_id = 42 AND is_current = TRUE;

-- Insert new current row
INSERT INTO dim_restaurant (restaurant_id, restaurant_name, cuisine_type, city, rating, is_active, valid_from)
VALUES (42, 'Golden Dragon', 'Fusion', 'Singapore', 4.2, TRUE, NOW());

-- Table now shows:
-- restaurant_key=15, restaurant_id=42, cuisine=Chinese, valid_to=2026-06-08, is_current=FALSE
-- restaurant_key=22, restaurant_id=42, cuisine=Fusion, valid_to=9999-12-31, is_current=TRUE
```

**Answer 22:**
```sql
INSERT INTO fact_sales (date_key, customer_key, ...)
SELECT 
    dd.date_key,
    dc.customer_key,  -- the surrogate key valid at order time
    ...
FROM source_orders o
JOIN dim_date dd ON dd.full_date = DATE(o.order_date)
JOIN dim_customer dc ON dc.customer_id = o.customer_id
    AND dc.valid_from <= o.order_date
    AND dc.valid_to > o.order_date
...;
```

**Answer 23:**
```sql
-- Table state:
-- key=1, id=100, Alice, segment=New, valid_from=Jan 1, valid_to=Mar 1, is_current=FALSE
-- key=4, id=100, Alice, segment=Regular, valid_from=Mar 1, valid_to=May 1, is_current=FALSE
-- key=7, id=100, Alice, segment=VIP, valid_from=May 1, valid_to=9999-12-31, is_current=TRUE

-- Query: segment progression
SELECT 
    customer_segment,
    valid_from AS segment_start,
    valid_to AS segment_end
FROM dim_customer
WHERE customer_id = 100
ORDER BY valid_from;
```

**Answer 24:**
```sql
SELECT 
    d1.customer_id,
    d1.valid_to AS gap_start,
    d2.valid_from AS gap_end,
    d2.valid_from - d1.valid_to AS gap_duration
FROM dim_customer d1
JOIN dim_customer d2 
    ON d1.customer_id = d2.customer_id 
    AND d2.valid_from > d1.valid_to
WHERE d1.is_current = FALSE
AND d2.valid_from - d1.valid_to > INTERVAL '1 second'
ORDER BY d1.customer_id, d1.valid_from;
```

**Answer 25:**
```sql
CREATE OR REPLACE PROCEDURE sp_load_dim_employee()
LANGUAGE plpgsql AS $$
DECLARE
    v_expired INT;
    v_inserted INT;
BEGIN
    -- Expire
    UPDATE dim_employee d
    SET valid_to = NOW(), is_current = FALSE
    FROM source_employees s
    WHERE d.employee_id = s.employee_id
    AND d.is_current = TRUE
    AND d.department IS DISTINCT FROM s.department;
    
    GET DIAGNOSTICS v_expired = ROW_COUNT;
    
    -- Insert
    INSERT INTO dim_employee (employee_id, employee_name, email, department, salary, manager_id, valid_from, valid_to, is_current)
    SELECT s.employee_id, s.employee_name, s.email, s.department, s.salary, s.manager_id, NOW(), '9999-12-31 23:59:59', TRUE
    FROM source_employees s
    WHERE NOT EXISTS (
        SELECT 1 FROM dim_employee d 
        WHERE d.employee_id = s.employee_id AND d.is_current = TRUE AND d.department = s.department
    );
    
    GET DIAGNOSTICS v_inserted = ROW_COUNT;
    RAISE NOTICE 'Expired % rows, inserted % rows', v_expired, v_inserted;
    
    EXCEPTION WHEN OTHERS THEN
        RAISE NOTICE 'Error: %', SQLERRM;
END;
$$;
```

**Answer 26:**
```sql
CREATE TABLE dim_customer_profile (
    profile_key SERIAL PRIMARY KEY,
    city VARCHAR(50),
    customer_segment VARCHAR(20),
    lifetime_value_band VARCHAR(10),
    activity_status VARCHAR(10)
);

INSERT INTO dim_customer_profile (city, customer_segment, lifetime_value_band, activity_status) VALUES
('Singapore', 'New', '0-100', 'Active'),
('Singapore', 'Regular', '100-500', 'Active'),
('Singapore', 'VIP', '500+', 'Active'),
('Kuala Lumpur', 'New', '0-100', 'Active'),
('Kuala Lumpur', 'Regular', '100-500', 'Active'),
('Penang', 'New', '0-100', 'Active'),
('Singapore', 'New', '0-100', 'Inactive'),
('Kuala Lumpur', 'VIP', '500+', 'Active');
```

**Answer 27:**
```sql
WITH dates AS (
    SELECT generate_series(DATE '2026-01-01', DATE '2026-12-31', INTERVAL '1 day')::DATE AS day
)
SELECT 
    d.day,
    dc.customer_id,
    dc.city,
    dc.customer_segment
FROM dates d
CROSS JOIN (SELECT DISTINCT customer_id FROM dim_customer WHERE customer_id = 100) cid
JOIN dim_customer dc ON dc.customer_id = cid.customer_id
    AND dc.valid_from <= d.day
    AND dc.valid_to > d.day
ORDER BY d.day;
```

**Answer 28:** For `lifetime_value` that changes with every order:
- **SCD Type 2** would create a new row for every order — potentially hundreds of rows per customer. Table explodes.
- **Mini-dimension (Type 4)** stores `lifetime_value_band` (bucketed ranges like "0-100", "100-500"). Changes less frequently, manageable row count.
- **Better:** Pre-aggregate and store in a separate fact table, or update a single "current profile" table without history.
- **Best answer:** Don't track `lifetime_value` with SCD at all — it's a derived measure, not an attribute. Compute it on the fly from `fact_sales`.

**Answer 29:**
```yaml
# snapshots/products_snapshot.yml
snapshots:
  - name: products_snapshot
    relation: source('raw', 'products')
    config:
      unique_key: product_id
      strategy: timestamp
      updated_at: updated_at
```
dbt handles SCD Type 2 automatically with snapshots.

**Answer 30:** In real-time, use a lookup cache (e.g., Redis) that maps `(customer_id, timestamp)` → `customer_key`. When a customer attribute changes, the SCD Type 2 ETL runs and the cache is updated. Fact loading queries the cache for the correct surrogate key.

**Answer 31:**
```sql
CREATE TABLE scd_audit_log (
    audit_id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    natural_key VARCHAR(50),
    change_type VARCHAR(10),     -- 'expired' or 'inserted'
    old_values JSONB,
    new_values JSONB,
    change_timestamp TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE PROCEDURE sp_scd2_customer_audited()
LANGUAGE plpgsql AS $$
BEGIN
    -- Log expired rows BEFORE expiring
    INSERT INTO scd_audit_log (table_name, natural_key, change_type, old_values)
    SELECT 'dim_customer', d.customer_id::TEXT, 'expired', 
           jsonb_build_object('city', d.city, 'segment', d.customer_segment)
    FROM dim_customer d
    JOIN source_customers s ON d.customer_id = s.customer_id
    WHERE d.is_current = TRUE
    AND (d.city IS DISTINCT FROM s.city OR d.customer_segment IS DISTINCT FROM s.customer_segment);
    
    -- Expire
    UPDATE dim_customer d
    SET valid_to = NOW(), is_current = FALSE
    FROM source_customers s
    WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
    AND (d.city IS DISTINCT FROM s.city OR d.customer_segment IS DISTINCT FROM s.customer_segment);
    
    -- Insert
    INSERT INTO dim_customer (customer_id, customer_name, email, city, customer_segment, valid_from, valid_to, is_current)
    SELECT s.customer_id, s.customer_name, s.email, s.city, s.customer_segment, NOW(), '9999-12-31 23:59:59', TRUE
    FROM source_customers s
    WHERE NOT EXISTS (
        SELECT 1 FROM dim_customer d WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
        AND d.city = s.city AND d.customer_segment = s.customer_segment
    );
    
    -- Log inserted rows
    INSERT INTO scd_audit_log (table_name, natural_key, change_type, new_values)
    SELECT 'dim_customer', s.customer_id::TEXT, 'inserted',
           jsonb_build_object('city', s.city, 'segment', s.customer_segment)
    FROM source_customers s
    WHERE NOT EXISTS (
        SELECT 1 FROM dim_customer d WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
        AND d.city = s.city AND d.customer_segment = s.customer_segment
    );
END;
$$;
```

**Answer 32:** SCD Type 2 handles this naturally — it creates a THIRD row:
```
key=1: Singapore, valid Jan-Mar, is_current=FALSE
key=2: KL, valid Mar-Jun, is_current=FALSE
key=3: Singapore, valid Jun-present, is_current=TRUE
```
Each row is independent. The fact table joins to the correct key based on the order date.

**Answer 33:**
```sql
SELECT 
    DATE_TRUNC('month', valid_from) AS month,
    COUNT(*) AS new_scd_rows
FROM dim_customer
WHERE customer_id IN (SELECT customer_id FROM dim_customer GROUP BY customer_id HAVING COUNT(*) > 1)
GROUP BY DATE_TRUNC('month', valid_from)
ORDER BY month;
```

**Answer 34:**
| Attribute | SCD Type | Reason |
|-----------|----------|--------|
| name | Type 1 | Corrections only |
| category | Type 2 | Product re-categorization affects analytics |
| price | Type 2 | Price changes are business events to track |
| description | Type 1 | Minor text changes, not analytical |
| brand | Type 2 | Brand changes are rare but significant |
| stock_count | None | Don't store in dimension — it's a fact/measure |

**Answer 35:** Hands-on exercise — follow the steps in Concept 3 to create tables, run 3 rounds of changes, and verify with analytical queries.

</details>

---

## 🌙 Evening Block (1 hour): Review & Flashcards

### SCD Decision Tree (memorize this for interviews!)

```
Attribute changing?
├── Is the change a correction (typo, data quality)?
│   └── YES → SCD Type 1 (overwrite)
├── Do you need to track history for reporting?
│   ├── YES → Do you need FULL history or just previous?
│   │   ├── Full history → SCD Type 2 (new row + dates)
│   │   └── Previous only → SCD Type 3 (previous column)
│   └── NO → SCD Type 1 (overwrite)
├── Does it change very frequently (every transaction)?
│   └── YES → Mini-dimension (Type 4) or compute on-the-fly
└── Multiple attributes with different rates?
    └── Hybrid (Type 1 for some, Type 2 for others)
```

### Key Terms

| Term | Definition |
|------|-----------|
| SCD | Slowly Changing Dimension — tracking attribute changes over time |
| Type 1 | Overwrite old value, no history |
| Type 2 | New row with validity dates, full history |
| Type 3 | Previous value column, limited history |
| Type 4 | Mini-dimension for frequently changing attributes |
| Type 6 | Hybrid of 1+2+3 |
| valid_from | When this version of the row became active |
| valid_to | When this version expired (9999-12-31 = still current) |
| is_current | Quick flag to find the active version |
| Surrogate Key | Auto-generated key that uniquely identifies each version |

### 📝 Today's Checklist

- [ ] I can explain SCD Types 1, 2, and 3 with examples
- [ ] I can write SCD Type 2 ETL (expire + insert pattern)
- [ ] I know when to use each SCD type
- [ ] I can write fact table joins that correctly look up SCD Type 2 keys
- [ ] I understand hybrid SCD (different types for different columns)
- [ ] I completed at least 25 of the 35 exercises
- [ ] I pushed any practice code to GitHub

### 📖 Further Reading (Free)

- [Kimball SCD Article](https://www.kimballgroup.com/2008/09/slowly-changing-dimensions/) — THE reference
- [dbt Snapshots Documentation](https://docs.getdbt.com/docs/building-a-dbt-project/snapshots) — dbt's built-in SCD Type 2
- [PostgreSQL SCD Patterns](https://www.postgresql.org/docs/current/sql-insert.html) — see UPSERT patterns

---

*Day 25 complete! Tomorrow: Advanced Window Functions — the SQL power-up you've been waiting for.* 🪟
