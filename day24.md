# 📅 Day 24 — Saturday, 7 June 2026
# 🏗️ Star Schema Deep Dive: Fact & Dimension Tables

---

## 🎯 Today's Goal

Star schema is the **most important data modeling pattern** in data engineering. Every data warehouse uses it. Today you'll understand *why* it exists, *how* to design one from scratch, and *when* to use it vs alternatives.

**Why this matters:** In interviews, you WILL be asked "design a data model for X." The answer is almost always a star schema. In real jobs, 80% of your modeling work is star schemas.

---

## ☀️ Morning Block (2 hours): Star Schema Fundamentals

### Concept 1: Why Star Schema?

Imagine a GrabFood report query on normalized (3NF) data:

```sql
-- 3NF: 6 tables joined just to get a simple report
SELECT 
    c.customer_name,
    c.city,
    r.restaurant_name,
    r.cuisine_type,
    d.dish_name,
    o.order_date,
    oi.quantity,
    oi.unit_price,
    p.payment_method
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN menu_items mi ON oi.menu_item_id = mi.item_id
JOIN payments p ON o.order_id = p.order_id;
```

**Problems with 3NF for analytics:**
- Too many JOINs = slow queries
- Complex for BI tools (Tableau, Power BI) to understand
- Business users can't write their own queries
- Adding a new report = writing complex SQL again

**Star schema solves ALL of this:**

```sql
-- Star schema: ONE fact table, direct joins to dimensions
SELECT 
    dm.full_date,
    dc.customer_name,
    dc.city,
    dr.restaurant_name,
    dr.cuisine_type,
    di.item_name,
    SUM(fs.quantity) AS total_quantity,
    SUM(fs.revenue) AS total_revenue
FROM fact_sales fs
JOIN dim_date dm ON fs.date_key = dm.date_key
JOIN dim_customer dc ON fs.customer_key = dc.customer_key
JOIN dim_restaurant dr ON fs.restaurant_key = dr.restaurant_key
JOIN dim_item di ON fs.item_key = di.item_key
WHERE dm.full_date BETWEEN '2026-01-01' AND '2026-03-31'
GROUP BY dm.full_date, dc.customer_name, dc.city, dr.restaurant_name, dr.cuisine_type, di.item_name;
```

### Concept 2: The Star Schema Anatomy

```
         dim_date ─────────┐
                            │
     dim_customer ──────────┤
                            │
    fact_sales ──────── (center, the "fact")
                            │
   dim_restaurant ──────────┤
                            │
       dim_item ────────────┘
```

**Fact Table:** The events/transactions that happened
- Contains **foreign keys** pointing to dimensions
- Contains **measures** (numbers you want to aggregate: quantity, revenue, discount)
- **One row per grain** (each row = one atomic event)
- Typically the largest table (millions to billions of rows)
- Named after the business process: `fact_sales`, `fact_orders`, `fact_deliveries`

**Dimension Tables:** The "who, what, where, when"
- Contains **descriptive attributes** (text, categories, hierarchies)
- Connected to fact via foreign key
- Typically much smaller than fact (hundreds to millions of rows)
- Named after the entity: `dim_customer`, `dim_product`, `dim_date`

### Concept 3: Grain — The Most Important Decision

The **grain** = what one row in the fact table represents. Get this wrong and everything else breaks.

```sql
-- GRAIN: One row per order item (most granular)
CREATE TABLE fact_sales (
    sales_key SERIAL PRIMARY KEY,
    date_key INT REFERENCES dim_date(date_key),
    customer_key INT REFERENCES dim_customer(customer_key),
    restaurant_key INT REFERENCES dim_restaurant(restaurant_key),
    item_key INT REFERENCES dim_item(item_key),
    promotion_key INT REFERENCES dim_promotion(promotion_key),
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    delivery_fee DECIMAL(5,2) DEFAULT 0,
    gross_revenue DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
    net_revenue DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price - discount_amount) STORED
);

-- GRAIN: One row per order (less granular)
CREATE TABLE fact_orders (
    order_key SERIAL PRIMARY KEY,
    date_key INT REFERENCES dim_date(date_key),
    customer_key INT REFERENCES dim_customer(customer_key),
    restaurant_key INT REFERENCES dim_restaurant(restaurant_key),
    total_items INT,
    total_quantity INT,
    gross_amount DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    delivery_fee DECIMAL(10,2),
    net_amount DECIMAL(10,2)
);

-- GRAIN: One row per day per restaurant (aggregated)
CREATE TABLE fact_daily_restaurant (
    date_key INT REFERENCES dim_date(date_key),
    restaurant_key INT REFERENCES dim_restaurant(restaurant_key),
    total_orders INT,
    total_revenue DECIMAL(12,2),
    avg_order_value DECIMAL(10,2),
    unique_customers INT,
    PRIMARY KEY (date_key, restaurant_key)
);
```

**💡 Rule:** Always start with the most granular grain possible. You can always aggregate later, but you can't disaggregate.

### Concept 4: Dimension Table Design

```sql
-- dim_date: The most important dimension, always needed
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,  -- 20260105 for Jan 5, 2026
    full_date DATE NOT NULL,
    day_of_week VARCHAR(10),   -- Monday, Tuesday...
    day_number INT,             -- 1-31
    day_name_short VARCHAR(3), -- Mon, Tue...
    week_number INT,            -- 1-52
    month_number INT,           -- 1-12
    month_name VARCHAR(10),    -- January, February...
    quarter INT,                -- 1-4
    year INT,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    holiday_name VARCHAR(50),
    fiscal_year INT,
    fiscal_quarter INT
);

-- Populate dim_date for 2025-2027
INSERT INTO dim_date
SELECT 
    TO_CHAR(d, 'YYYYMMDD')::INT AS date_key,
    d AS full_date,
    TO_CHAR(d, 'TMDay') AS day_of_week,
    EXTRACT(DAY FROM d)::INT AS day_number,
    TO_CHAR(d, 'Dy') AS day_name_short,
    EXTRACT(WEEK FROM d)::INT AS week_number,
    EXTRACT(MONTH FROM d)::INT AS month_number,
    TO_CHAR(d, 'TMMonth') AS month_name,
    EXTRACT(QUARTER FROM d)::INT AS quarter,
    EXTRACT(YEAR FROM d)::INT AS year,
    EXTRACT(ISODOW FROM d) IN (6, 7) AS is_weekend,
    FALSE AS is_holiday,  -- update later with actual holidays
    NULL AS holiday_name,
    EXTRACT(YEAR FROM d)::INT AS fiscal_year,  -- simplify for now
    EXTRACT(QUARTER FROM d)::INT AS fiscal_quarter
FROM generate_series(
    DATE '2025-01-01',
    DATE '2027-12-31',
    INTERVAL '1 day'
) AS d;

-- Update SG/MY holidays (example)
UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'New Year''s Day' 
WHERE full_date IN ('2026-01-01', '2027-01-01');
UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Chinese New Year' 
WHERE full_date IN ('2026-02-17', '2026-02-18');
UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Hari Raya Puasa' 
WHERE full_date IN ('2026-03-30');
UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Deepavali' 
WHERE full_date IN ('2026-11-10');
UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Merdeka Day' 
WHERE full_date IN ('2026-08-31');
UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Malaysia Day' 
WHERE full_date IN ('2026-09-16');
UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Christmas Day' 
WHERE full_date IN ('2026-12-25');

-- dim_customer
CREATE TABLE dim_customer (
    customer_key SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,       -- natural key from source
    customer_name VARCHAR(100),
    email VARCHAR(150),
    city VARCHAR(50),
    state VARCHAR(50),
    country VARCHAR(50),
    postal_code VARCHAR(10),
    signup_date DATE,
    customer_segment VARCHAR(20),   -- 'New', 'Regular', 'VIP', 'Churned'
    lifetime_value DECIMAL(12,2),
    is_active BOOLEAN,
    valid_from TIMESTAMP DEFAULT NOW(),
    valid_to TIMESTAMP DEFAULT '9999-12-31',
    is_current BOOLEAN DEFAULT TRUE
);

-- dim_restaurant
CREATE TABLE dim_restaurant (
    restaurant_key SERIAL PRIMARY KEY,
    restaurant_id INT NOT NULL,
    restaurant_name VARCHAR(100),
    cuisine_type VARCHAR(50),
    cuisine_category VARCHAR(30),    -- 'Asian', 'Western', 'Fusion'
    city VARCHAR(50),
    postal_code VARCHAR(10),
    rating DECIMAL(3,2),
    price_range VARCHAR(10),          -- 'Budget', 'Mid', 'Premium'
    is_active BOOLEAN,
    valid_from TIMESTAMP DEFAULT NOW(),
    valid_to TIMESTAMP DEFAULT '9999-12-31',
    is_current BOOLEAN DEFAULT TRUE
);

-- dim_item
CREATE TABLE dim_item (
    item_key SERIAL PRIMARY KEY,
    item_id INT NOT NULL,
    restaurant_id INT,
    dish_name VARCHAR(100),
    category VARCHAR(30),           -- 'Main', 'Side', 'Drink', 'Dessert'
    cuisine_type VARCHAR(50),
    price DECIMAL(8,2),
    is_vegetarian BOOLEAN,
    spice_level VARCHAR(10),
    is_available BOOLEAN DEFAULT TRUE
);

-- dim_promotion
CREATE TABLE dim_promotion (
    promotion_key SERIAL PRIMARY KEY,
    promotion_id INT NOT NULL,
    promotion_name VARCHAR(100),
    promotion_type VARCHAR(30),     -- 'Percentage', 'Fixed', 'Free Delivery'
    discount_value DECIMAL(8,2),
    start_date DATE,
    end_date DATE,
    minimum_order DECIMAL(8,2),
    is_active BOOLEAN
);
```

### Concept 5: Surrogate Keys vs Natural Keys

```sql
-- Natural key: The original ID from the source system
-- customer_id = 12345 (from the app database)

-- Surrogate key: A synthetic key generated for the warehouse
-- customer_key = 1, 2, 3... (auto-increment in dim_customer)

-- WHY surrogate keys?
-- 1. Source systems change (merger = duplicate IDs)
-- 2. Track history (same customer_id, different addresses = SCD Type 2)
-- 3. Performance (INT join faster than VARCHAR join)
-- 4. Decouple warehouse from source (source can change its ID scheme)

-- The fact table uses surrogate keys:
-- fact_sales.customer_key → dim_customer.customer_key (NOT customer_id)
```

---

### 🏋️ Morning Exercises (20 questions)

1. Explain in your own words why star schema is better than 3NF for analytics. Write 3 specific reasons.

2. Design the grain for these fact tables. What does one row represent?
    - A food delivery platform wants to track order items
    - A cinema chain wants to track ticket sales
    - A Grab driver wants to track daily earnings

3. For the GrabFood scenario, identify all possible dimensions for a `fact_delivery` table that tracks individual deliveries.

4. Create a complete `dim_date` table populated for the entire year 2026. Include SG/MY holidays.

5. Design a `dim_customer` table for a Shopee-like e-commerce platform. Include at least 10 attributes that would be useful for analytics.

6. Write a SQL statement to populate `dim_restaurant` from the normalized `restaurants` source table. Add derived columns like `cuisine_category` and `price_range`.

7. **Grain decision:** Your stakeholder wants to track "daily sales per restaurant." But another stakeholder wants "individual order items." Do you create two fact tables or one? Why?

8. Design a `dim_driver` dimension for a Grab-like ride-hailing platform. What attributes would be useful for analysis?

9. Write SQL to create a `dim_time` table (time of day, separate from date) with columns for hour, minute, time_of_day (Morning/Afternoon/Evening/Night), and is_peak_hour.

10. **Conformed dimension:** Explain what a conformed dimension is and give an example using the GrabFood scenario. Why is it important?

11. Design a fact table for tracking food delivery performance. Grain: one row per delivery. Include at least 4 dimensions and 5 measures.

12. Write SQL to generate surrogate keys when loading a dimension from a source that doesn't have them.

13. A stakeholder asks: "Can we put customer_city directly in the fact table instead of joining to dim_customer?" Explain why this is a bad idea.

14. Design a `dim_location` table that can serve as a conformed dimension across both a food delivery fact table and a ride-hailing fact table.

15. **Degenerate dimension:** What is a degenerate dimension? Give an example from the GrabFood scenario. When would you keep it in the fact table vs creating a dimension?

16. Write a complete CREATE TABLE statement for a `fact_order_items` table with proper foreign keys, at least 5 dimension keys, and 5 measures.

17. **Junk dimension:** Explain what a junk dimension is. Design one for the GrabFood scenario that consolidates low-cardinality flags (is_weekend, is_holiday, is_peak_hour, payment_type, order_source).

18. Create a `dim_promotion` table and populate it with 5 sample promotions relevant to SG/MY (e.g., "SG50 Food Festival", "Ramadan Buffet Deal").

19. Design a **snowflake schema** variant of the star schema where `dim_customer` has a sub-dimension `dim_city` which has `dim_state` which has `dim_country`. When would snowflake be better than star?

20. **Challenge:** Design a complete star schema for a **hawker centre management system** in Singapore. Requirements:
    - Track daily sales per stall per food item
    - Analyze by stall, food category, hawker centre, date, and customer type
    - Include measures for quantity, revenue, cost, and profit margin
    - Account for peak vs off-peak pricing
    Draw the schema diagram (list all tables with columns) and write all CREATE TABLE statements.

---

### ✅ Morning Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1:**
1. **Fewer JOINs** — Star schema typically needs 3-5 joins vs 8-10 in normalized schema, making queries faster and simpler
2. **BI tool friendly** — Tableau/Power BI can auto-detect star schema relationships, drag-and-drop dimensions and measures
3. **Consistent reporting** — Everyone uses the same dimension definitions (same customer_segment, same date hierarchy), preventing conflicting numbers

**Answer 2:**
- Food delivery order items: **One row per order item** (order_id + item_id combination)
- Cinema ticket sales: **One row per ticket sold** (each ticket is an atomic event)
- Grab driver daily earnings: **One row per trip** (not per day — trips are the atomic event)

**Answer 3:**
Dimensions for `fact_delivery`:
- `dim_date` (order date)
- `dim_time` (order time of day)
- `dim_customer` (who ordered)
- `dim_restaurant` (which restaurant)
- `dim_driver` (who delivered)
- `dim_location` (pickup and dropoff locations)
- `dim_delivery_status` (order status)

**Answer 4:**
```sql
-- See Concept 4 above for the full INSERT statement
-- Add SG/MY holidays with UPDATE statements
-- Verify: SELECT COUNT(*) FROM dim_date WHERE year = 2026;  -- should be 365
```

**Answer 5:**
```sql
CREATE TABLE dim_customer (
    customer_key SERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    phone VARCHAR(20),
    city VARCHAR(50),
    state VARCHAR(50),
    country VARCHAR(50),
    postal_code VARCHAR(10),
    signup_date DATE,
    age_group VARCHAR(10),         -- '18-24', '25-34', '35-44', '45+'
    customer_segment VARCHAR(20),   -- 'New', 'Bronze', 'Silver', 'Gold', 'Platinum'
    preferred_category VARCHAR(50), -- most-purchased category
    lifetime_orders INT,
    lifetime_value DECIMAL(12,2),
    avg_order_value DECIMAL(10,2),
    days_since_last_order INT,
    is_active BOOLEAN,
    valid_from TIMESTAMP,
    valid_to TIMESTAMP,
    is_current BOOLEAN
);
```

**Answer 6:**
```sql
INSERT INTO dim_restaurant (restaurant_id, restaurant_name, cuisine_type, cuisine_category, city, postal_code, rating, price_range, is_active)
SELECT 
    restaurant_id,
    restaurant_name,
    cuisine_type,
    CASE 
        WHEN cuisine_type IN ('Chinese', 'Malay', 'Indian', 'Thai', 'Japanese', 'Korean') THEN 'Asian'
        WHEN cuisine_type IN ('American', 'Italian', 'French') THEN 'Western'
        ELSE 'Fusion'
    END AS cuisine_category,
    city,
    postal_code,
    rating,
    CASE 
        WHEN rating >= 4.0 THEN 'Premium'
        WHEN rating >= 3.0 THEN 'Mid'
        ELSE 'Budget'
    END AS price_range,
    is_active
FROM source_restaurants;
```

**Answer 7:** Create ONE fact table at the most granular level (order items). You can create a view or materialized view for daily aggregations. Starting at a higher grain means you lose detail you can never recover.

**Answer 8:**
```sql
CREATE TABLE dim_driver (
    driver_key SERIAL PRIMARY KEY,
    driver_id INT NOT NULL,
    driver_name VARCHAR(100),
    phone VARCHAR(20),
    vehicle_type VARCHAR(20),     -- 'Motorcycle', 'Car', 'Bicycle'
    city VARCHAR(50),
    signup_date DATE,
    driver_tier VARCHAR(10),       -- 'Bronze', 'Silver', 'Gold', 'Platinum'
    total_trips INT,
    avg_rating DECIMAL(3,2),
    years_experience INT,
    is_active BOOLEAN,
    valid_from TIMESTAMP,
    valid_to TIMESTAMP,
    is_current BOOLEAN
);
```

**Answer 9:**
```sql
CREATE TABLE dim_time (
    time_key INT PRIMARY KEY,  -- HHMM format: 930 = 9:30 AM
    hour_24 INT NOT NULL,
    hour_12 INT NOT NULL,
    minute INT NOT NULL,
    am_pm VARCHAR(2),
    time_of_day VARCHAR(20),  -- 'Morning', 'Afternoon', 'Evening', 'Night', 'Late Night'
    is_peak_hour BOOLEAN,
    meal_period VARCHAR(20)    -- 'Breakfast', 'Lunch', 'Tea', 'Dinner', 'Supper', 'Off-peak'
);

INSERT INTO dim_time
SELECT 
    h * 100 + m AS time_key,
    h AS hour_24,
    CASE WHEN h = 0 THEN 12 WHEN h > 12 THEN h - 12 ELSE h END AS hour_12,
    m AS minute,
    CASE WHEN h < 12 THEN 'AM' ELSE 'PM' END,
    CASE 
        WHEN h BETWEEN 6 AND 9 THEN 'Morning'
        WHEN h BETWEEN 10 AND 11 THEN 'Late Morning'
        WHEN h BETWEEN 12 AND 14 THEN 'Afternoon'
        WHEN h BETWEEN 15 AND 17 THEN 'Late Afternoon'
        WHEN h BETWEEN 18 AND 21 THEN 'Evening'
        ELSE 'Night'
    END,
    h IN (7, 8, 11, 12, 18, 19),  -- peak meal hours
    CASE 
        WHEN h BETWEEN 6 AND 9 THEN 'Breakfast'
        WHEN h BETWEEN 11 AND 14 THEN 'Lunch'
        WHEN h BETWEEN 14 AND 17 THEN 'Tea'
        WHEN h BETWEEN 17 AND 21 THEN 'Dinner'
        WHEN h >= 21 OR h < 2 THEN 'Supper'
        ELSE 'Off-peak'
    END
FROM generate_series(0, 23) h
CROSS JOIN generate_series(0, 59, 15) m;  -- every 15 minutes
```

**Answer 10:** A conformed dimension is shared across multiple fact tables / data marts, ensuring consistent reporting. Example: `dim_date` is used by both `fact_sales` (food orders) and `fact_deliveries` (driver trips). When you join them through `dim_date`, you know "January 2026" means the same thing in both contexts.

**Answer 11:**
```sql
CREATE TABLE fact_delivery (
    delivery_key SERIAL PRIMARY KEY,
    date_key INT REFERENCES dim_date(date_key),
    time_key INT REFERENCES dim_time(time_key),
    customer_key INT REFERENCES dim_customer(customer_key),
    restaurant_key INT REFERENCES dim_restaurant(restaurant_key),
    driver_key INT REFERENCES dim_driver(driver_key),
    pickup_location_key INT REFERENCES dim_location(location_key),
    dropoff_location_key INT REFERENCES dim_location(location_key),
    -- Measures
    delivery_distance_km DECIMAL(5,2),
    preparation_time_min INT,
    delivery_time_min INT,
    total_time_min INT,
    delivery_fee DECIMAL(5,2),
    tip_amount DECIMAL(5,2) DEFAULT 0,
    was_on_time BOOLEAN,
    delay_minutes INT DEFAULT 0
);
```

**Answer 12:** Using SERIAL (auto-increment) or IDENTITY column:
```sql
INSERT INTO dim_customer (customer_key, customer_id, ...)
SELECT NEXTVAL('dim_customer_customer_key_seq'), source.customer_id, ...
FROM source_customers source;
-- Or simply omit customer_key if using SERIAL PRIMARY KEY
```

**Answer 13:** Three reasons to NOT put city in the fact table:
1. **Data redundancy** — city repeated millions of times vs once per customer in dim
2. **Historical tracking** — if a customer moves, you can track both old and new city via SCD Type 2 in dim. In the fact table, you'd lose the history
3. **Data quality** — if city names need correction, update once in dim vs millions of rows in fact

**Answer 14:**
```sql
CREATE TABLE dim_location (
    location_key SERIAL PRIMARY KEY,
    location_id VARCHAR(20) NOT NULL,
    area_name VARCHAR(50),
    city VARCHAR(50),
    state VARCHAR(50),
    country VARCHAR(50),
    postal_code VARCHAR(10),
    latitude DECIMAL(9,6),
    longitude DECIMAL(9,6),
    zone VARCHAR(20),           -- 'Central', 'North', 'South', 'East', 'West'
    is_commercial_area BOOLEAN,
    population_density VARCHAR(10)  -- 'High', 'Medium', 'Low'
);
```

**Answer 15:** A degenerate dimension has no corresponding dimension table — it's stored directly in the fact table. Example: `order_number` or `receipt_number`. These are useful for grouping/drilling but don't have attributes worth making a separate table for.

**Answer 16:**
```sql
CREATE TABLE fact_order_items (
    sales_key SERIAL PRIMARY KEY,
    order_number VARCHAR(20),          -- degenerate dimension
    date_key INT REFERENCES dim_date(date_key),
    time_key INT REFERENCES dim_time(time_key),
    customer_key INT REFERENCES dim_customer(customer_key),
    restaurant_key INT REFERENCES dim_restaurant(restaurant_key),
    item_key INT REFERENCES dim_item(item_key),
    promotion_key INT REFERENCES dim_promotion(promotion_key),
    -- Measures
    quantity INT NOT NULL,
    unit_price DECIMAL(8,2) NOT NULL,
    discount_amount DECIMAL(8,2) DEFAULT 0,
    gross_amount DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
    net_amount DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price - discount_amount) STORED,
    cost_of_goods DECIMAL(8,2),
    margin DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price - discount_amount - COALESCE(quantity * cost_of_goods, 0)) STORED
);
```

**Answer 17:** A junk dimension combines several low-cardinality flags/indicators into one dimension to reduce fact table width and improve usability.
```sql
CREATE TABLE dim_order_flags (
    flag_key SERIAL PRIMARY KEY,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    is_peak_hour BOOLEAN,
    payment_type VARCHAR(20),
    order_source VARCHAR(20),      -- 'App', 'Web', 'API'
    delivery_type VARCHAR(20),     -- 'Standard', 'Express', 'Self-pickup'
    is_first_order BOOLEAN
);
-- With only a few values each, total combinations might be 2*2*2*4*3*3*2 = 192 rows max
-- Instead of 7 separate columns in the fact table, one flag_key does it all
```

**Answer 18:**
```sql
INSERT INTO dim_promotion (promotion_id, promotion_name, promotion_type, discount_value, start_date, end_date, minimum_order, is_active) VALUES
(1, 'SG National Day Food Fest', 'Percentage', 15.00, '2026-07-25', '2026-08-09', 25.00, TRUE),
(2, 'Ramadan Iftar Deal', 'Fixed', 5.00, '2026-03-01', '2026-03-30', 30.00, TRUE),
(3, 'New User Free Delivery', 'Free Delivery', 0.00, '2026-01-01', '2026-12-31', 0.00, TRUE),
(4, 'CNY Reunion Dinner Promo', 'Percentage', 20.00, '2026-02-10', '2026-02-18', 50.00, TRUE),
(5, 'Deepavali Mutton Special', 'Fixed', 8.00, '2026-11-05', '2026-11-12', 35.00, TRUE);
```

**Answer 19:** Snowflake normalizes dimensions into sub-dimensions. It saves storage (city name stored once in `dim_city` instead of repeated in every customer row) but adds JOIN complexity. Star schema is preferred for analytics because:
- Fewer JOINs = faster queries
- Simpler for BI tools
- Storage is cheap; query performance matters more
- Snowflake only makes sense when dimensions are VERY large (millions of unique values)

**Answer 20:**
```sql
-- Hawker Centre Star Schema

CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    full_date DATE NOT NULL,
    day_of_week VARCHAR(10),
    month_name VARCHAR(10),
    quarter INT,
    year INT,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN
);

CREATE TABLE dim_stall (
    stall_key SERIAL PRIMARY KEY,
    stall_id VARCHAR(20) NOT NULL,
    stall_name VARCHAR(100),
    stall_number VARCHAR(10),
    cuisine_type VARCHAR(50),
    hawker_centre_key INT,
    owner_name VARCHAR(100),
    years_operating INT,
    is_active BOOLEAN
);

CREATE TABLE dim_hawker_centre (
    hawker_centre_key SERIAL PRIMARY KEY,
    centre_name VARCHAR(100),
    address VARCHAR(200),
    city VARCHAR(50),
    postal_code VARCHAR(10),
    number_of_stalls INT,
    has_aircon BOOLEAN,
    rating DECIMAL(3,2)
);

CREATE TABLE dim_food_item (
    item_key SERIAL PRIMARY KEY,
    item_name VARCHAR(100),
    category VARCHAR(30),        -- 'Main', 'Side', 'Drink', 'Dessert'
    sub_category VARCHAR(30),
    is_halal BOOLEAN,
    is_vegetarian BOOLEAN,
    spice_level VARCHAR(10)
);

CREATE TABLE dim_customer_type (
    customer_type_key SERIAL PRIMARY KEY,
    type_name VARCHAR(20),       -- 'Regular', 'Tourist', 'Office Worker', 'Student'
    meal_time VARCHAR(20)        -- 'Breakfast', 'Lunch', 'Dinner', 'Snack'
);

CREATE TABLE dim_time (
    time_key INT PRIMARY KEY,
    hour INT,
    time_of_day VARCHAR(20),
    is_peak_hour BOOLEAN,
    pricing_tier VARCHAR(10)     -- 'Peak', 'Off-peak'
);

CREATE TABLE fact_daily_sales (
    sales_key SERIAL PRIMARY KEY,
    date_key INT REFERENCES dim_date(date_key),
    time_key INT REFERENCES dim_time(time_key),
    stall_key INT REFERENCES dim_stall(stall_key),
    item_key INT REFERENCES dim_food_item(item_key),
    customer_type_key INT REFERENCES dim_customer_type(customer_type_key),
    -- Measures
    quantity_sold INT NOT NULL,
    unit_price DECIMAL(6,2) NOT NULL,
    total_revenue DECIMAL(10,2) NOT NULL,
    ingredient_cost DECIMAL(8,2),
    labour_cost DECIMAL(8,2),
    gross_profit DECIMAL(10,2),
    profit_margin DECIMAL(5,4),
    is_peak_pricing BOOLEAN
);
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Loading Data into Star Schema (ETL)

### Concept 6: Dimension Load Patterns

```sql
-- TYPE 1: Insert new, update existing (overwrite)
-- Simple, no history tracking

-- Insert new customers
INSERT INTO dim_customer (customer_id, customer_name, email, city, signup_date, is_active)
SELECT s.customer_id, s.customer_name, s.email, s.city, s.signup_date, TRUE
FROM source_customers s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer d WHERE d.customer_id = s.customer_id
);

-- Update changed attributes (overwrites old values)
UPDATE dim_customer d
SET 
    customer_name = s.customer_name,
    email = s.email,
    city = s.city,
    is_active = s.is_active
FROM source_customers s
WHERE d.customer_id = s.customer_id
AND (d.customer_name != s.customer_name 
     OR d.email != s.email 
     OR d.city != s.city
     OR d.is_active != s.is_active);
```

### Concept 7: Surrogate Key Pipeline

```sql
-- Step 1: Load new dimension rows (get surrogate keys automatically)
INSERT INTO dim_item (item_id, dish_name, category, cuisine_type, price, is_vegetarian, spice_level)
SELECT 
    src.item_id, src.dish_name, src.category, src.cuisine_type, 
    src.price, src.is_vegetarian, src.spice_level
FROM source_menu_items src
WHERE NOT EXISTS (
    SELECT 1 FROM dim_item d WHERE d.item_id = src.item_id
);

-- Step 2: Load fact table using surrogate keys
INSERT INTO fact_sales (date_key, customer_key, restaurant_key, item_key, quantity, unit_price, discount_amount)
SELECT 
    dd.date_key,
    dc.customer_key,
    dr.restaurant_key,
    di.item_key,
    oi.quantity,
    oi.unit_price,
    COALESCE(oi.discount, 0)
FROM source_order_items oi
JOIN source_orders o ON oi.order_id = o.order_id
JOIN dim_date dd ON dd.full_date = DATE(o.order_date)
JOIN dim_customer dc ON dc.customer_id = o.customer_id AND dc.is_current = TRUE
JOIN dim_restaurant dr ON dr.restaurant_id = o.restaurant_id AND dr.is_current = TRUE
JOIN dim_item di ON di.item_id = oi.item_id
WHERE o.processed_flag = FALSE;

-- Mark source as processed
UPDATE source_orders SET processed_flag = TRUE WHERE processed_flag = FALSE;
```

### Concept 8: Building a Complete ETL Pipeline

```sql
-- Full ETL for daily load

BEGIN;

-- 1. Update dim_date (usually pre-populated, but add new dates if needed)
-- (Already done — dim_date is populated for years in advance)

-- 2. Update dim_customer (SCD Type 1 for now)
INSERT INTO dim_customer (customer_id, customer_name, email, city, postal_code, signup_date, is_active)
SELECT customer_id, customer_name, email, city, postal_code, signup_date, TRUE
FROM source_customers
WHERE NOT EXISTS (SELECT 1 FROM dim_customer d WHERE d.customer_id = source_customers.customer_id);

UPDATE dim_customer d
SET customer_name = s.customer_name, email = s.email, city = s.city, is_active = s.is_active
FROM source_customers s
WHERE d.customer_id = s.customer_id
AND (d.customer_name IS DISTINCT FROM s.customer_name 
     OR d.email IS DISTINCT FROM s.email 
     OR d.city IS DISTINCT FROM s.city);

-- 3. Update dim_restaurant
INSERT INTO dim_restaurant (restaurant_id, restaurant_name, cuisine_type, city, postal_code, rating, is_active)
SELECT restaurant_id, restaurant_name, cuisine_type, city, postal_code, rating, is_active
FROM source_restaurants
WHERE NOT EXISTS (SELECT 1 FROM dim_restaurant d WHERE d.restaurant_id = source_restaurants.restaurant_id);

-- 4. Update dim_item
INSERT INTO dim_item (item_id, dish_name, category, cuisine_type, price, is_vegetarian, spice_level)
SELECT item_id, dish_name, category, cuisine_type, price, is_vegetarian, spice_level
FROM source_menu_items
WHERE NOT EXISTS (SELECT 1 FROM dim_item d WHERE d.item_id = source_menu_items.item_id);

-- 5. Load fact_sales
INSERT INTO fact_sales (date_key, customer_key, restaurant_key, item_key, quantity, unit_price, discount_amount)
SELECT 
    dd.date_key,
    dc.customer_key,
    dr.restaurant_key,
    di.item_key,
    oi.quantity,
    oi.unit_price,
    COALESCE(oi.discount_amount, 0)
FROM staging_order_items oi
JOIN staging_orders o ON oi.order_id = o.order_id
JOIN dim_date dd ON dd.full_date = DATE(o.order_date)
JOIN dim_customer dc ON dc.customer_id = o.customer_id AND dc.is_current = TRUE
JOIN dim_restaurant dr ON dr.restaurant_id = o.restaurant_id AND dr.is_current = TRUE
JOIN dim_item di ON di.item_id = oi.item_id;

-- 6. Update customer_segment based on new data
UPDATE dim_customer d
SET customer_segment = CASE
    WHEN lifetime_orders >= 100 THEN 'VIP'
    WHEN lifetime_orders >= 20 THEN 'Regular'
    WHEN lifetime_orders >= 1 THEN 'New'
    ELSE 'Prospect'
END;

COMMIT;
```

---

### 🏋️ Afternoon Exercises (15 questions)

21. Write a complete ETL script that loads today's new orders into `fact_sales`. Handle the case where a customer or restaurant doesn't exist in the dimension yet.

22. Design a query that validates the star schema: check for orphan fact rows (fact rows with no matching dimension). Write checks for all dimension keys.

23. Write a stored procedure `sp_daily_etl()` that runs the complete daily load. Include error handling and a log table to track each run.

24. Create a view `vw_customer_sales` that joins fact_sales with all dimensions to provide a flat, queryable table for analysts.

25. Write a query to find the top 10 restaurants by revenue using the star schema. Include cuisine_type and city in the output.

26. Design a slowly changing dimension load for `dim_customer` that detects when a customer's city changes and updates the record (SCD Type 1). Show the UPDATE statement.

27. Write a query using the star schema to answer: "What's the average order value by cuisine type and by time of day (Breakfast/Lunch/Dinner)?"

28. Create an aggregate table `fact_monthly_sales` that pre-computes monthly totals per restaurant. Populate it from `fact_sales`.

29. Write a data quality check: "Find any orders in fact_sales that have a date_key not in dim_date." What should happen if you find any?

30. Design a mini ETL pipeline for a Shopee order system:
    - Source tables: `raw_orders`, `raw_products`, `raw_users`
    - Target: star schema with `fact_orders`, `dim_product`, `dim_user`, `dim_date`
    - Write the complete SQL

31. Write a query to calculate **customer lifetime value (CLV)** using the star schema. CLV = total revenue from a customer, grouped by customer segment.

32. Investigate: What happens if you try to INSERT a fact row with a customer_id that doesn't exist in dim_customer? How do foreign key constraints protect you?

33. Write a query that uses the star schema to find **which day of the week has the highest average order value**. Include holidays vs non-holidays comparison.

34. Design a star schema for a **MRT ridership tracking system** in Singapore. Fact table tracks individual tap-ins/tap-outs.

35. **Practical challenge:** Build the complete star schema in PostgreSQL:
    - Create all dimension and fact tables
    - Populate dim_date for 2026
    - Load sample data (100+ fact rows)
    - Write 5 analytical queries against it
    - Validate data quality (no orphans, no NULLs in required fields)

---

### ✅ Afternoon Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 21:**
```sql
BEGIN;

-- First, ensure all dimensions are up to date
-- Insert new customers
INSERT INTO dim_customer (customer_id, customer_name, email, city, signup_date, is_active)
SELECT DISTINCT o.customer_id, o.customer_name, o.email, o.city, CURRENT_DATE, TRUE
FROM staging_orders o
WHERE NOT EXISTS (SELECT 1 FROM dim_customer d WHERE d.customer_id = o.customer_id);

-- Insert new restaurants
INSERT INTO dim_restaurant (restaurant_id, restaurant_name, cuisine_type, city, rating, is_active)
SELECT DISTINCT o.restaurant_id, o.restaurant_name, o.cuisine_type, o.city, o.rating, TRUE
FROM staging_orders o
WHERE NOT EXISTS (SELECT 1 FROM dim_restaurant d WHERE d.restaurant_id = o.restaurant_id);

-- Insert new items
INSERT INTO dim_item (item_id, dish_name, category, cuisine_type, price, is_vegetarian, spice_level)
SELECT DISTINCT oi.item_id, oi.dish_name, oi.category, oi.cuisine_type, oi.price, oi.is_vegetarian, oi.spice_level
FROM staging_order_items oi
WHERE NOT EXISTS (SELECT 1 FROM dim_item d WHERE d.item_id = oi.item_id);

-- Now load facts
INSERT INTO fact_sales (date_key, customer_key, restaurant_key, item_key, quantity, unit_price, discount_amount)
SELECT 
    dd.date_key,
    dc.customer_key,
    dr.restaurant_key,
    di.item_key,
    oi.quantity,
    oi.unit_price,
    COALESCE(oi.discount_amount, 0)
FROM staging_order_items oi
JOIN staging_orders o ON oi.order_id = o.order_id
JOIN dim_date dd ON dd.full_date = DATE(o.order_date)
JOIN dim_customer dc ON dc.customer_id = o.customer_id AND dc.is_current = TRUE
JOIN dim_restaurant dr ON dr.restaurant_id = o.restaurant_id AND dr.is_current = TRUE
JOIN dim_item di ON di.item_id = oi.item_id
WHERE o.order_date >= CURRENT_DATE;

COMMIT;
```

**Answer 22:**
```sql
-- Check for orphan fact rows
SELECT 'orphan_date' AS check_name, COUNT(*) AS orphan_count
FROM fact_sales fs LEFT JOIN dim_date dd ON fs.date_key = dd.date_key
WHERE dd.date_key IS NULL
UNION ALL
SELECT 'orphan_customer', COUNT(*)
FROM fact_sales fs LEFT JOIN dim_customer dc ON fs.customer_key = dc.customer_key
WHERE dc.customer_key IS NULL
UNION ALL
SELECT 'orphan_restaurant', COUNT(*)
FROM fact_sales fs LEFT JOIN dim_restaurant dr ON fs.restaurant_key = dr.restaurant_key
WHERE dr.restaurant_key IS NULL
UNION ALL
SELECT 'orphan_item', COUNT(*)
FROM fact_sales fs LEFT JOIN dim_item di ON fs.item_key = di.item_key
WHERE di.item_key IS NULL;
-- All should return 0 if foreign keys are enforced
```

**Answer 23:**
```sql
CREATE TABLE etl_log (
    log_id SERIAL PRIMARY KEY,
    run_date TIMESTAMP DEFAULT NOW(),
    step_name VARCHAR(100),
    rows_affected INT,
    status VARCHAR(20),
    error_message TEXT
);

CREATE OR REPLACE PROCEDURE sp_daily_etl()
LANGUAGE plpgsql AS $$
DECLARE
    v_rows INT;
BEGIN
    -- Log start
    INSERT INTO etl_log (step_name, status) VALUES ('ETL START', 'running');
    
    BEGIN
        -- Load dimensions
        INSERT INTO dim_customer (customer_id, customer_name, email, city, signup_date, is_active)
        SELECT customer_id, customer_name, email, city, signup_date, TRUE
        FROM source_customers s
        WHERE NOT EXISTS (SELECT 1 FROM dim_customer d WHERE d.customer_id = s.customer_id);
        GET DIAGNOSTICS v_rows = ROW_COUNT;
        INSERT INTO etl_log (step_name, rows_affected, status) VALUES ('dim_customer load', v_rows, 'success');
    EXCEPTION WHEN OTHERS THEN
        INSERT INTO etl_log (step_name, status, error_message) VALUES ('dim_customer load', 'error', SQLERRM);
        RAISE;
    END;
    
    -- Similar blocks for other dimensions and fact table...
    
    INSERT INTO etl_log (step_name, status) VALUES ('ETL END', 'success');
END;
$$;
```

**Answer 24:**
```sql
CREATE VIEW vw_customer_sales AS
SELECT 
    dd.full_date,
    dd.day_of_week,
    dd.month_name,
    dd.quarter,
    dc.customer_name,
    dc.city AS customer_city,
    dc.customer_segment,
    dr.restaurant_name,
    dr.cuisine_type,
    dr.city AS restaurant_city,
    di.dish_name,
    di.category AS item_category,
    dp.promotion_name,
    fs.quantity,
    fs.unit_price,
    fs.discount_amount,
    fs.gross_amount,
    fs.net_amount
FROM fact_sales fs
JOIN dim_date dd ON fs.date_key = dd.date_key
JOIN dim_customer dc ON fs.customer_key = dc.customer_key
JOIN dim_restaurant dr ON fs.restaurant_key = dr.restaurant_key
JOIN dim_item di ON fs.item_key = di.item_key
LEFT JOIN dim_promotion dp ON fs.promotion_key = dp.promotion_key;
```

**Answer 25:**
```sql
SELECT 
    dr.restaurant_name,
    dr.cuisine_type,
    dr.city,
    SUM(fs.net_amount) AS total_revenue,
    COUNT(*) AS total_items_sold
FROM fact_sales fs
JOIN dim_restaurant dr ON fs.restaurant_key = dr.restaurant_key
GROUP BY dr.restaurant_name, dr.cuisine_type, dr.city
ORDER BY total_revenue DESC
LIMIT 10;
```

**Answer 26:**
```sql
UPDATE dim_customer d
SET city = s.city
FROM source_customers s
WHERE d.customer_id = s.customer_id
AND d.city IS DISTINCT FROM s.city
AND d.is_current = TRUE;
```

**Answer 27:**
```sql
SELECT 
    dr.cuisine_type,
    dt.time_of_day AS meal_period,
    ROUND(AVG(fs.net_amount), 2) AS avg_order_value,
    COUNT(*) AS order_count
FROM fact_sales fs
JOIN dim_restaurant dr ON fs.restaurant_key = dr.restaurant_key
JOIN dim_time dt ON fs.time_key = dt.time_key
GROUP BY dr.cuisine_type, dt.time_of_day
ORDER BY dr.cuisine_type, avg_order_value DESC;
```

**Answer 28:**
```sql
CREATE TABLE fact_monthly_sales (
    month_key INT,  -- YYYYMM format
    restaurant_key INT REFERENCES dim_restaurant(restaurant_key),
    total_orders INT,
    total_revenue DECIMAL(12,2),
    total_discount DECIMAL(10,2),
    net_revenue DECIMAL(12,2),
    avg_order_value DECIMAL(10,2),
    unique_customers INT,
    PRIMARY KEY (month_key, restaurant_key)
);

INSERT INTO fact_monthly_sales
SELECT 
    TO_CHAR(dd.full_date, 'YYYYMM')::INT AS month_key,
    fs.restaurant_key,
    COUNT(DISTINCT fs.sales_key) AS total_orders,
    SUM(fs.gross_amount) AS total_revenue,
    SUM(fs.discount_amount) AS total_discount,
    SUM(fs.net_amount) AS net_revenue,
    ROUND(AVG(fs.net_amount), 2) AS avg_order_value,
    COUNT(DISTINCT fs.customer_key) AS unique_customers
FROM fact_sales fs
JOIN dim_date dd ON fs.date_key = dd.date_key
GROUP BY TO_CHAR(dd.full_date, 'YYYYMM')::INT, fs.restaurant_key;
```

**Answer 29:**
```sql
SELECT fs.sales_key, fs.date_key
FROM fact_sales fs
LEFT JOIN dim_date dd ON fs.date_key = dd.date_key
WHERE dd.date_key IS NULL;
-- If foreign keys are properly enforced, this returns 0 rows.
-- If orphans exist, investigate the ETL pipeline — dimension load may have failed.
-- Fix: either add the missing dimension rows or delete the orphan facts.
```

**Answer 30:**
```sql
-- Dimensions
INSERT INTO dim_user (user_id, user_name, email, city, signup_date, is_active)
SELECT user_id, user_name, email, city, signup_date, TRUE
FROM raw_users
WHERE NOT EXISTS (SELECT 1 FROM dim_user d WHERE d.user_id = raw_users.user_id);

INSERT INTO dim_product (product_id, product_name, category, brand, price)
SELECT product_id, product_name, category, brand, price
FROM raw_products
WHERE NOT EXISTS (SELECT 1 FROM dim_product d WHERE d.product_id = raw_products.product_id);

-- Fact
INSERT INTO fact_orders (date_key, user_key, product_key, quantity, total_amount)
SELECT 
    dd.date_key,
    du.user_key,
    dp.product_key,
    ro.quantity,
    ro.total_amount
FROM raw_orders ro
JOIN dim_date dd ON dd.full_date = DATE(ro.order_date)
JOIN dim_user du ON du.user_id = ro.user_id
JOIN dim_product dp ON dp.product_id = ro.product_id
WHERE ro.etl_processed = FALSE;

UPDATE raw_orders SET etl_processed = TRUE WHERE etl_processed = FALSE;
```

**Answer 31:**
```sql
SELECT 
    dc.customer_segment,
    COUNT(DISTINCT dc.customer_key) AS customer_count,
    ROUND(SUM(fs.net_amount), 2) AS total_clv,
    ROUND(AVG(fs.net_amount), 2) AS avg_order_value,
    ROUND(SUM(fs.net_amount) / COUNT(DISTINCT dc.customer_key), 2) AS avg_clv_per_customer
FROM fact_sales fs
JOIN dim_customer dc ON fs.customer_key = dc.customer_key
GROUP BY dc.customer_segment
ORDER BY avg_clv_per_customer DESC;
```

**Answer 32:** If foreign keys are enforced, PostgreSQL will raise an error:
```
ERROR: insert or update on table "fact_sales" violates foreign key constraint "fact_sales_customer_key_fkey"
DETAIL: Key (customer_key)=(999) is not present in table "dim_customer".
```
This is WHY we load dimensions BEFORE facts — the FK constraint catches data quality issues.

**Answer 33:**
```sql
SELECT 
    dd.day_of_week,
    dd.is_holiday,
    ROUND(AVG(fs.net_amount), 2) AS avg_order_value,
    COUNT(*) AS order_count
FROM fact_sales fs
JOIN dim_date dd ON fs.date_key = dd.date_key
GROUP BY dd.day_of_week, dd.is_holiday
ORDER BY avg_order_value DESC;
```

**Answer 34:**
```sql
-- MRT Ridership Star Schema
CREATE TABLE dim_station (
    station_key SERIAL PRIMARY KEY,
    station_name VARCHAR(50),
    station_code VARCHAR(10),     -- 'NS1', 'EW2', 'CC3'
    line_name VARCHAR(20),         -- 'North-South', 'East-West', 'Circle'
    city VARCHAR(50),
    is_interchange BOOLEAN
);

CREATE TABLE dim_rider (
    rider_key SERIAL PRIMARY KEY,
    card_id VARCHAR(20),
    rider_type VARCHAR(20),        -- 'Adult', 'Student', 'Senior', 'Child'
    is_season_pass BOOLEAN
);

CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    full_date DATE,
    day_of_week VARCHAR(10),
    is_weekend BOOLEAN,
    is_holiday BOOLEAN
);

CREATE TABLE dim_time (
    time_key INT PRIMARY KEY,
    hour INT,
    time_of_day VARCHAR(20),
    is_peak_hour BOOLEAN
);

CREATE TABLE fact_ridership (
    trip_key SERIAL PRIMARY KEY,
    date_key INT REFERENCES dim_date(date_key),
    time_key INT REFERENCES dim_time(time_key),
    entry_station_key INT REFERENCES dim_station(station_key),
    exit_station_key INT REFERENCES dim_station(station_key),
    rider_key INT REFERENCES dim_rider(rider_key),
    travel_time_min INT,
    num_stops INT,
    fare DECIMAL(5,2)
);
```

**Answer 35:** This is a hands-on exercise. Create the tables from Concepts 4 and 6, populate them, and verify with the queries from exercises 25, 27, 31, 33, and the orphan check from exercise 22.

</details>

---

## 🌙 Evening Block (1 hour): Review & Flashcards

### Key Concepts to Memorize

| Term | Definition |
|------|-----------|
| Star Schema | Central fact table connected to dimension tables like a star |
| Fact Table | Contains measures (numbers) and foreign keys to dimensions |
| Dimension Table | Contains descriptive attributes (who, what, where, when) |
| Grain | What one row in the fact table represents |
| Surrogate Key | Artificial key generated for the warehouse (INT auto-increment) |
| Natural Key | Original key from the source system |
| Conformed Dimension | Dimension shared across multiple fact tables |
| Degenerate Dimension | Dimension stored in fact table (e.g., order number) |
| Junk Dimension | Combines low-cardinality flags into one dimension |
| Snowflake Schema | Normalized dimensions with sub-dimensions |

### 📝 Today's Checklist

- [ ] I can explain star schema and why it's used for analytics
- [ ] I can identify the grain of a fact table
- [ ] I can design dimension tables with appropriate attributes
- [ ] I understand surrogate vs natural keys
- [ ] I can write ETL SQL to load a star schema from source data
- [ ] I completed at least 25 of the 35 exercises
- [ ] I pushed any practice code to GitHub

### 📖 Further Reading (Free)

- [Kimball Group Articles](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/) — THE authority on dimensional modeling
- [Star Schema Wikipedia](https://en.wikipedia.org/wiki/Star_schema) — good overview
- [The Data Warehouse Toolkit (Chapter 1 free)](https://www.wiley.com/en-us/The+Data+Warehouse+Toolkit%3A+The+Definitive+Guide+to+Dimensional+Modeling%2C+3rd+Edition-p-9781118530801)

---

*Day 24 complete! Tomorrow: Slowly Changing Dimensions — tracking history like a pro.* 📊
