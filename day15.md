# 📅 Day 15 — Thursday, 29 May 2026
# Data Modeling: Star Schema, Fact & Dimension Tables, Normalization

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Understand what data modeling is and why it matters
- Design a star schema with fact and dimension tables
- Understand normalization (1NF, 2NF, 3NF) and when to denormalize
- Draw an Entity-Relationship Diagram (ERD) for a business scenario
- Implement a star schema in PostgreSQL

**Why this matters:** Every job posting mentions "data modeling." It's how you structure data so analysts and AI engineers can actually USE it. A bad model means slow queries, confused analysts, and data you can't trust. A good model means everyone can self-serve.

---

## ☀️ BLOCK 1: What is Data Modeling? (Morning, ~1.5 hours)

---

### Task 1: The Problem Data Modeling Solves (15 min)

Imagine you're building a data platform for an e-commerce company. Raw data comes in like this:

```
Raw order data (one big messy table):
order_id | customer_name | customer_email | product_name | category | price | qty | order_date | store_name | store_city
---------|---------------|----------------|-------------|----------|-------|-----|------------|------------|------------
001      | Alice Tan     | alice@email    | Laptop      | Electronics| 1499 | 1  | 2024-01-15 | Orchard     | Singapore
001      | Alice Tan     | alice@email    | Mouse       | Electronics| 45   | 2  | 2024-01-15 | Orchard     | Singapore
002      | Bob Lim       | bob@email      | T-Shirt     | Clothing  | 29.90| 3  | 2024-01-16 | Mid Valley  | KL
```

**Problems with this flat table:**
- Customer info repeated in every row (wastes space, hard to update)
- Product info repeated in every row
- Store info repeated in every row
- If Alice changes her email, you must update MULTIPLE rows
- Hard to answer: "How many unique customers bought Electronics?"

**Data modeling fixes this** by splitting data into structured tables with relationships.

---

### Task 2: Fact Tables vs Dimension Tables (30 min)

A **star schema** has two types of tables:

**DIMENSION tables** = the "WHO, WHAT, WHERE, WHEN"
- Descriptive attributes (text, categories, dates)
- Relatively slow-changing (a customer name doesn't change often)
- Used for filtering and grouping (WHERE and GROUP BY)
- Examples: dim_customers, dim_products, dim_stores, dim_date

**FACT table** = the "WHAT HAPPENED" (the numbers)
- Contains measurements/metrics (amounts, quantities, counts)
- Foreign keys pointing to dimension tables
- One row per event/transaction
- Examples: fact_sales, fact_orders, fact_page_views

```
         dim_customers          dim_products
         ┌──────────┐          ┌──────────┐
         │ cust_id  │          │ prod_id  │
         │ name     │          │ name     │
         │ email    │          │ category │
         │ city     │          │ price    │
         └────┬─────┘          └────┬─────┘
              │                     │
              │    ┌──────────────────────┐
              │    │     fact_sales       │
              └───►│ cust_id (FK)         │
                   │ prod_id (FK)         │◄─── dim_stores
                   │ store_id (FK)        │     ┌──────────┐
                   │ date_id (FK)         │     │ store_id │
                   │ quantity             │     │ name     │
                   │ amount               │     │ city     │
                   │ discount             │     └──────────┘
                   └────────┬─────────────┘
                            │
                       dim_date
                       ┌──────────┐
                       │ date_id  │
                       │ date     │
                       │ year     │
                       │ month    │
                       │ quarter  │
                       │ day_name │
                       └──────────┘
```

> 🧠 **Why "star" schema?** If you draw the fact table in the center with dimension tables around it, it looks like a star. This is the most common data model in data warehousing.

**Fact table guidelines:**
- Keep it NARROW — only foreign keys + numeric measures
- One row = one business event (one sale, one click, one order)
- Don't store descriptive text in fact tables (that goes in dimensions)

**Dimension table guidelines:**
- Wide — lots of descriptive columns
- Surrogate key (auto-incrementing integer) as primary key
- Include all attributes analysts might want to filter by

---

### Task 3: Star Schema Design Exercise (30 min)

**Scenario:** A food delivery platform (like GrabFood/Foodpanda) needs a data model.

**Business questions they need to answer:**
1. Revenue by restaurant, by month, by cuisine type
2. Average delivery time by city, by time of day
3. Top customers by total spending
4. Most popular dishes by area
5. Revenue comparison: weekday vs weekend

**Design the star schema:**

Think about what fact table and dimension tables you need. Try to identify:

1. What is the business event (grain of the fact table)?
   - Each individual food order? Or each order item?
   - → Each ORDER ITEM (one row per dish in an order)

2. What dimensions do you need?
   - Customer, Restaurant, Dish, Date, Time, Delivery Area

3. What measures (numeric facts) do you need?
   - quantity, price, delivery_fee, discount, preparation_time, delivery_time

<details>
<summary>📖 Reference Design</summary>

```
FACT TABLE: fact_order_items
- order_item_id (PK)
- order_id (degenerate dimension — we keep it but don't make a dim table)
- customer_id (FK → dim_customers)
- restaurant_id (FK → dim_restaurants)
- dish_id (FK → dim_dishes)
- date_id (FK → dim_date)
- time_id (FK → dim_time)
- delivery_area_id (FK → dim_delivery_areas)
- quantity
- unit_price
- total_amount (quantity * unit_price)
- discount_amount
- delivery_fee
- preparation_minutes
- delivery_minutes

DIMENSION: dim_customers
- customer_id (PK, surrogate key)
- customer_name
- email
- phone
- city
- country
- signup_date
- loyalty_tier (Bronze/Silver/Gold)
- total_orders (running count, updated periodically)

DIMENSION: dim_restaurants
- restaurant_id (PK)
- restaurant_name
- cuisine_type
- city
- area
- rating
- price_range ($/$$/$$$)
- is_active

DIMENSION: dim_dishes
- dish_id (PK)
- dish_name
- category (Main/Appetizer/Drink/Dessert)
- cuisine_type
- is_vegetarian
- spice_level (Mild/Medium/Hot)
- calories

DIMENSION: dim_date
- date_id (PK, integer like 20240115)
- full_date (DATE)
- day_of_week (Monday...)
- day_number (1-31)
- month_number (1-12)
- month_name (January...)
- quarter (Q1-Q4)
- year
- is_weekend (TRUE/FALSE)
- is_holiday (TRUE/FALSE)
- fiscal_period

DIMENSION: dim_time
- time_id (PK)
- hour (0-23)
- minute (0-59)
- am_pm (AM/PM)
- time_of_day (Morning/Afternoon/Evening/Night)
- meal_period (Breakfast/Lunch/Dinner/Late Night)

DIMENSION: dim_delivery_areas
- area_id (PK)
- area_name
- city
- state
- postal_code
- zone (North/South/East/West/Central)
```
</details>

---

## 🔥 BLOCK 2: Normalization (Afternoon, ~1.5 hours)

---

### Task 4: Normal Forms Explained (30 min)

Normalization is the process of organizing data to reduce redundancy and improve integrity. You need to understand this for interviews.

**1NF (First Normal Form):** Each cell contains a single value, each row is unique.

```sql
-- ❌ Not 1NF (multiple values in one cell)
CREATE TABLE bad_orders (
    order_id INT,
    products VARCHAR -- "Laptop, Mouse, Keyboard" ← multiple values!
);

-- ✅ 1NF (atomic values, one product per row)
CREATE TABLE orders_1nf (
    order_id INT,
    product VARCHAR,  -- one product per row
    quantity INT
);
```

**2NF (Second Normal Form):** 1NF + no partial dependencies (every non-key column depends on the ENTIRE primary key).

```sql
-- ❌ Not 2NF (product_name depends only on product_id, not the full key)
-- PK is (order_id, product_id)
-- product_name depends on product_id alone (partial dependency!)
CREATE TABLE bad_order_items (
    order_id INT,
    product_id INT,
    product_name VARCHAR,  -- ← depends on product_id only!
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

-- ✅ 2NF: move product_name to products table
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR
);
CREATE TABLE order_items (
    order_id INT,
    product_id INT REFERENCES products(product_id),
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

**3NF (Third Normal Form):** 2NF + no transitive dependencies (non-key columns don't depend on other non-key columns).

```sql
-- ❌ Not 3NF (city depends on postal_code, not on customer_id directly)
CREATE TABLE bad_customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR,
    postal_code VARCHAR,
    city VARCHAR  -- ← city depends on postal_code, not customer_id!
);

-- ✅ 3NF: separate postal/city lookup
CREATE TABLE postal_codes (
    postal_code VARCHAR PRIMARY KEY,
    city VARCHAR,
    state VARCHAR
);
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR,
    postal_code VARCHAR REFERENCES postal_codes(postal_code)
);
```

> 🧠 **For data warehousing, we actually DENORMALIZE (go back toward 1NF).** Dimension tables are intentionally denormalized (wide, with redundant data like city AND state) because it makes queries faster and simpler. This is the star schema trade-off: more storage, but much faster analytics queries.

---

### Task 5: Implement Star Schema in SQL (45 min)

Let's build the food delivery star schema in PostgreSQL:

```sql
-- ============================================
-- Food Delivery Star Schema
-- ============================================

-- DIMENSION: Date dimension (populate for 2024)
CREATE TABLE dim_date (
    date_id INTEGER PRIMARY KEY,  -- format: YYYYMMDD e.g., 20240115
    full_date DATE NOT NULL,
    day_of_week VARCHAR(10),
    day_number INTEGER,
    month_number INTEGER,
    month_name VARCHAR(20),
    quarter VARCHAR(2),
    year INTEGER,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN DEFAULT FALSE
);

-- DIMENSION: Time dimension
CREATE TABLE dim_time (
    time_id SERIAL PRIMARY KEY,
    hour INTEGER CHECK (hour BETWEEN 0 AND 23),
    minute INTEGER CHECK (minute BETWEEN 0 AND 59),
    time_of_day VARCHAR(20),  -- Morning/Afternoon/Evening/Night
    meal_period VARCHAR(20),  -- Breakfast/Lunch/Dinner/Late Night
    UNIQUE(hour, minute)
);

-- DIMENSION: Customers
CREATE TABLE dim_customers (
    customer_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    phone VARCHAR(20),
    city VARCHAR(50),
    country VARCHAR(50),
    signup_date DATE,
    loyalty_tier VARCHAR(20) DEFAULT 'Bronze',
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- DIMENSION: Restaurants
CREATE TABLE dim_restaurants (
    restaurant_id SERIAL PRIMARY KEY,
    restaurant_name VARCHAR(100) NOT NULL,
    cuisine_type VARCHAR(50),
    city VARCHAR(50),
    area VARCHAR(50),
    rating DECIMAL(3,2) CHECK (rating BETWEEN 0 AND 5),
    price_range VARCHAR(5),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- DIMENSION: Dishes
CREATE TABLE dim_dishes (
    dish_id SERIAL PRIMARY KEY,
    dish_name VARCHAR(100) NOT NULL,
    restaurant_id INTEGER REFERENCES dim_restaurants(restaurant_id),
    category VARCHAR(50),
    cuisine_type VARCHAR(50),
    is_vegetarian BOOLEAN DEFAULT FALSE,
    spice_level VARCHAR(10) DEFAULT 'Mild',
    price DECIMAL(8,2) NOT NULL CHECK (price > 0),
    is_available BOOLEAN DEFAULT TRUE
);

-- DIMENSION: Delivery Areas
CREATE TABLE dim_delivery_areas (
    area_id SERIAL PRIMARY KEY,
    area_name VARCHAR(50) NOT NULL,
    city VARCHAR(50),
    postal_code VARCHAR(10),
    zone VARCHAR(20)
);

-- FACT TABLE: Order Items
CREATE TABLE fact_order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,               -- degenerate dimension
    customer_id INTEGER REFERENCES dim_customers(customer_id),
    restaurant_id INTEGER REFERENCES dim_restaurants(restaurant_id),
    dish_id INTEGER REFERENCES dim_dishes(dish_id),
    date_id INTEGER REFERENCES dim_date(date_id),
    time_id INTEGER REFERENCES dim_time(time_id),
    delivery_area_id INTEGER REFERENCES dim_delivery_areas(area_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(8,2) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(8,2) DEFAULT 0,
    delivery_fee DECIMAL(8,2) DEFAULT 0,
    preparation_minutes INTEGER,
    delivery_minutes INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for common query patterns
CREATE INDEX idx_fact_date ON fact_order_items(date_id);
CREATE INDEX idx_fact_customer ON fact_order_items(customer_id);
CREATE INDEX idx_fact_restaurant ON fact_order_items(restaurant_id);
CREATE INDEX idx_fact_dish ON fact_order_items(dish_id);
```

---

### Task 6: Populate the Date Dimension (20 min)

Date dimensions are special — you pre-populate them for a range of dates.

```sql
-- Generate date dimension for all of 2024
INSERT INTO dim_date (date_id, full_date, day_of_week, day_number, month_number, month_name, quarter, year, is_weekend)
SELECT 
    TO_CHAR(d, 'YYYYMMDD')::INTEGER AS date_id,
    d AS full_date,
    TO_CHAR(d, 'Day') AS day_of_week,
    EXTRACT(DAY FROM d)::INTEGER AS day_number,
    EXTRACT(MONTH FROM d)::INTEGER AS month_number,
    TO_CHAR(d, 'Month') AS month_name,
    'Q' || EXTRACT(QUARTER FROM d) AS quarter,
    EXTRACT(YEAR FROM d)::INTEGER AS year,
    EXTRACT(DOW FROM d) IN (0, 6) AS is_weekend
FROM generate_series('2024-01-01'::DATE, '2024-12-31'::DATE, '1 day'::INTERVAL) AS d;

-- Verify
SELECT COUNT(*) FROM dim_date;  -- should be 366 (2024 is leap year)
SELECT * FROM dim_date WHERE full_date = '2024-01-15';
```

---

## 🌙 BLOCK 3: Practice (Evening, ~1.5 hours)

---

### Task 7: Populate Sample Data + Write Analytical Queries (45 min)

```sql
-- Insert sample dimensions
INSERT INTO dim_time (hour, minute, time_of_day, meal_period) VALUES
(7, 0, 'Morning', 'Breakfast'),
(8, 30, 'Morning', 'Breakfast'),
(12, 0, 'Afternoon', 'Lunch'),
(12, 30, 'Afternoon', 'Lunch'),
(13, 0, 'Afternoon', 'Lunch'),
(18, 30, 'Evening', 'Dinner'),
(19, 0, 'Evening', 'Dinner'),
(20, 0, 'Evening', 'Dinner'),
(22, 0, 'Night', 'Late Night'),
(23, 30, 'Night', 'Late Night');

INSERT INTO dim_customers (customer_name, email, city, country, signup_date, loyalty_tier) VALUES
('Alice Tan', 'alice@email.com', 'Singapore', 'Singapore', '2023-01-15', 'Gold'),
('Bob Lim', 'bob@email.com', 'Kuala Lumpur', 'Malaysia', '2023-03-20', 'Silver'),
('Cathy Ng', 'cathy@email.com', 'Singapore', 'Singapore', '2023-08-10', 'Silver'),
('David Chen', 'david@email.com', 'Penang', 'Malaysia', '2024-01-05', 'Bronze'),
('Eva Rahman', 'eva@email.com', 'Kuala Lumpur', 'Malaysia', '2023-06-01', 'Gold');

INSERT INTO dim_restaurants (restaurant_name, cuisine_type, city, area, rating, price_range) VALUES
('Chicken Rice King', 'Chinese', 'Singapore', 'Orchard', 4.50, '$'),
('Nasi Lemak House', 'Malay', 'Kuala Lumpur', 'Bangsar', 4.30, '$'),
('Roti Prata Corner', 'Indian', 'Singapore', 'Little India', 4.10, '$'),
('Sushi Express', 'Japanese', 'Singapore', 'Jurong', 4.00, '$$'),
('Tom Yum Palace', 'Thai', 'Kuala Lumpur', 'Pavilion', 4.60, '$$');

INSERT INTO dim_dishes (dish_name, restaurant_id, category, cuisine_type, is_vegetarian, price) VALUES
('Hainanese Chicken Rice', 1, 'Main', 'Chinese', FALSE, 6.50),
('Roast Chicken Rice', 1, 'Main', 'Chinese', FALSE, 7.00),
('Nasi Lemak Special', 2, 'Main', 'Malay', FALSE, 8.50),
('Roti Prata', 3, 'Main', 'Indian', TRUE, 2.50),
('Prata Egg', 3, 'Main', 'Indian', FALSE, 3.50),
('Salmon Sashimi Set', 4, 'Main', 'Japanese', FALSE, 18.00),
('Green Curry', 5, 'Main', 'Thai', FALSE, 14.00),
('Mango Sticky Rice', 5, 'Dessert', 'Thai', TRUE, 8.00);

INSERT INTO dim_delivery_areas (area_name, city, postal_code, zone) VALUES
('Orchard', 'Singapore', '238801', 'Central'),
('Jurong', 'Singapore', '609631', 'West'),
('Little India', 'Singapore', '207821', 'Central'),
('Bangsar', 'Kuala Lumpur', '59000', 'West'),
('Pavilion', 'Kuala Lumpur', '55100', 'Central'),
('Gurney', 'Penang', '10250', 'North');

-- Insert some fact rows (you add more!)
INSERT INTO fact_order_items (order_id, customer_id, restaurant_id, dish_id, date_id, time_id, delivery_area_id, quantity, unit_price, total_amount, discount_amount, delivery_fee, preparation_minutes, delivery_minutes) VALUES
(1001, 1, 1, 1, 20240115, 3, 1, 2, 6.50, 13.00, 0, 3.00, 15, 25),
(1001, 1, 1, 2, 20240115, 3, 1, 1, 7.00, 7.00, 0, 0, 15, 25),
(1002, 2, 2, 3, 20240116, 4, 4, 1, 8.50, 8.50, 1.00, 4.00, 20, 30),
(1003, 3, 3, 4, 20240117, 1, 3, 3, 2.50, 7.50, 0, 2.00, 10, 15),
(1004, 1, 4, 6, 20240120, 6, 1, 1, 18.00, 18.00, 2.00, 3.50, 25, 20),
(1005, 4, 5, 7, 20240122, 7, 6, 1, 14.00, 14.00, 0, 5.00, 20, 35),
(1006, 5, 2, 3, 20240203, 3, 5, 2, 8.50, 17.00, 1.50, 4.00, 20, 30),
(1007, 2, 5, 8, 20240214, 7, 5, 1, 8.00, 8.00, 0, 4.00, 15, 25),
(1008, 3, 1, 1, 20240301, 3, 2, 1, 6.50, 6.50, 0, 3.00, 15, 20);
```

Now write these analytical queries:

```sql
-- 1. Total revenue by month
-- 2. Top 3 restaurants by revenue
-- 3. Revenue by cuisine type
-- 4. Average delivery time by city
-- 5. Weekend vs weekday revenue comparison
-- 6. Most popular dish overall
-- 7. Customer spending ranking
-- 8. Revenue by meal period
-- 9. Average order value by loyalty tier
-- 10. Monthly revenue growth rate (use LAG)
```

<details>
<summary>📖 Answers</summary>

```sql
-- 1
SELECT dd.month_name, dd.year, SUM(foi.total_amount) AS revenue
FROM fact_order_items foi
JOIN dim_date dd ON foi.date_id = dd.date_id
GROUP BY dd.year, dd.month_number, dd.month_name
ORDER BY dd.year, dd.month_number;

-- 2
SELECT dr.restaurant_name, SUM(foi.total_amount) AS revenue
FROM fact_order_items foi
JOIN dim_restaurants dr ON foi.restaurant_id = dr.restaurant_id
GROUP BY dr.restaurant_name
ORDER BY revenue DESC LIMIT 3;

-- 3
SELECT dr.cuisine_type, SUM(foi.total_amount) AS revenue
FROM fact_order_items foi
JOIN dim_restaurants dr ON foi.restaurant_id = dr.restaurant_id
GROUP BY dr.cuisine_type;

-- 4
SELECT da.city, ROUND(AVG(foi.delivery_minutes), 1) AS avg_delivery_min
FROM fact_order_items foi
JOIN dim_delivery_areas da ON foi.delivery_area_id = da.area_id
GROUP BY da.city;

-- 5
SELECT dd.is_weekend, SUM(foi.total_amount) AS revenue, COUNT(*) AS orders
FROM fact_order_items foi
JOIN dim_date dd ON foi.date_id = dd.date_id
GROUP BY dd.is_weekend;

-- 6
SELECT dd.dish_name, SUM(foi.quantity) AS total_qty
FROM fact_order_items foi
JOIN dim_dishes dd ON foi.dish_id = dd.dish_id
GROUP BY dd.dish_name
ORDER BY total_qty DESC LIMIT 1;

-- 7
SELECT dc.customer_name, SUM(foi.total_amount) AS total_spent
FROM fact_order_items foi
JOIN dim_customers dc ON foi.customer_id = dc.customer_id
GROUP BY dc.customer_name
ORDER BY total_spent DESC;

-- 8
SELECT dt.meal_period, SUM(foi.total_amount) AS revenue
FROM fact_order_items foi
JOIN dim_time dt ON foi.time_id = dt.time_id
GROUP BY dt.meal_period
ORDER BY revenue DESC;

-- 9
SELECT dc.loyalty_tier, ROUND(AVG(foi.total_amount), 2) AS avg_order_value
FROM fact_order_items foi
JOIN dim_customers dc ON foi.customer_id = dc.customer_id
GROUP BY dc.loyalty_tier;

-- 10
WITH monthly AS (
    SELECT dd.year, dd.month_number, SUM(foi.total_amount) AS revenue
    FROM fact_order_items foi
    JOIN dim_date dd ON foi.date_id = dd.date_id
    GROUP BY dd.year, dd.month_number
)
SELECT year, month_number, revenue,
    LAG(revenue) OVER (ORDER BY year, month_number) AS prev_month,
    ROUND((revenue - LAG(revenue) OVER (ORDER BY year, month_number)) * 100.0
          / NULLIF(LAG(revenue) OVER (ORDER BY year, month_number), 0), 2) AS growth_pct
FROM monthly;
```
</details>

---

### Task 8: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 15 — Data Modeling
Date: 2026-05-29

## Key Concepts
- Star Schema = 1 fact table + multiple dimension tables
- Fact Table = events/transactions (numeric measures + foreign keys)
- Dimension Table = descriptive attributes (WHO, WHAT, WHERE, WHEN)
- Normalization = reduce redundancy (1NF → 2NF → 3NF)
- Data warehousing = intentional DENORMALIZATION for query speed
- Date dimension = pre-populated for all dates

## My Food Delivery Star Schema
- 6 dimensions: date, time, customers, restaurants, dishes, delivery_areas
- 1 fact table: order_items
- Wrote 10 analytical queries

## Tomorrow: More Data Modeling + Slowly Changing Dimensions
```

```bash
git add .
git commit -m "Day 15: Data modeling — star schema, fact/dimension tables, normalization"
git push
```

---

## ✅ Day 15 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Can explain fact vs dimension tables | ☐ |
| 2 | Understand star schema layout | ☐ |
| 3 | Can explain 1NF, 2NF, 3NF | ☐ |
| 4 | Designed food delivery star schema | ☐ |
| 5 | Created all dimension tables in PostgreSQL | ☐ |
| 6 | Created fact table with foreign keys | ☐ |
| 7 | Populated date dimension with generate_series | ☐ |
| 8 | Wrote 10 analytical queries against star schema | ☐ |
| 9 | Day 15 notes pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card

```
STAR SCHEMA
├── FACT TABLE (center)
│   ├── Foreign keys → dimensions
│   └── Numeric measures (qty, amount, time)
└── DIMENSION TABLES (surrounding)
    ├── Surrogate key (auto-increment)
    └── Descriptive attributes

FACT = WHAT HAPPENED (numbers)
DIMENSION = WHO/WHAT/WHERE/WHEN (descriptions)

NORMALIZATION (for OLTP/transactional):
1NF → atomic values
2NF → no partial dependencies
3NF → no transitive dependencies

WAREHOUSING (OLAP/analytics):
→ Denormalize into star schema
→ Trade storage for query speed
```
