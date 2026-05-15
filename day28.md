# 📅 Day 28 — Wednesday, 11 June 2026
# 🏗️ PORTFOLIO PROJECT 4: SG Food Delivery Data Warehouse

---

## 🎯 Today's Goal

Build a **complete star schema data warehouse** in PostgreSQL for a GrabFood-like platform. This is a portfolio-ready project that demonstrates:
- Star schema design
- SCD Type 2 implementation
- ETL pipeline (SQL)
- Advanced analytical queries
- Data quality checks

**Outcome:** A GitHub repo called `food-delivery-warehouse` with professional documentation.

---

## 📋 Project Overview

### Business Scenario
You're a data engineer at **"MakanExpress"** — a food delivery platform operating in Singapore and Kuala Lumpur. The operations team needs a data warehouse to answer questions like:
- "What's our revenue by city and cuisine type?"
- "Which restaurants are trending up vs down?"
- "What's our customer retention rate?"
- "Which dishes drive repeat orders?"

### Source System (OLTP — normalized)
```
raw_customers (customer_id, name, email, phone, city, postal_code, signup_date)
raw_restaurants (restaurant_id, name, cuisine, city, postal_code, rating, is_active, updated_at)
raw_menu_items (item_id, restaurant_id, dish_name, category, price, is_vegetarian, spice_level)
raw_orders (order_id, customer_id, restaurant_id, order_date, total_amount, status, delivery_fee)
raw_order_items (item_id, order_id, menu_item_id, quantity, unit_price, discount)
raw_payments (payment_id, order_id, method, amount, status, paid_at)
```

### Target Warehouse (OLAP — star schema)
```
dim_date
dim_customer (SCD Type 2)
dim_restaurant (SCD Type 2)
dim_item
dim_payment_method
fact_order_items
```

---

## 🏗️ Build Steps

### Step 1: Create the Project (Morning, 30 min)

```bash
mkdir food-delivery-warehouse
cd food-delivery-warehouse
git init
```

Create folder structure:
```
food-delivery-warehouse/
├── README.md
├── schema/
│   ├── 01_dim_date.sql
│   ├── 02_dim_customer.sql
│   ├── 03_dim_restaurant.sql
│   ├── 04_dim_item.sql
│   ├── 05_dim_payment_method.sql
│   ├── 06_fact_order_items.sql
│   └── 07_indexes.sql
├── seed/
│   ├── generate_dim_date.sql
│   └── sample_data.sql
├── etl/
│   ├── 01_load_dim_customer.sql
│   ├── 02_load_dim_restaurant.sql
│   ├── 03_load_dim_item.sql
│   ├── 04_load_fact.sql
│   └── 05_full_etl.sql
├── analytics/
│   ├── revenue_by_city_cuisine.sql
│   ├── customer_retention.sql
│   ├── restaurant_trends.sql
│   ├── rfm_segmentation.sql
│   └── top_dishes.sql
├── quality/
│   ├── orphan_checks.sql
│   ├── scd_validation.sql
│   └── data_freshness.sql
└── docs/
    └── erd.md
```

### Step 2: Create Source Tables & Sample Data (30 min)

**`seed/sample_data.sql`:**

```sql
-- ============================================
-- MakanExpress Sample Data
-- ============================================

-- Customers (50 customers across SG and KL)
INSERT INTO raw_customers (customer_id, name, email, phone, city, postal_code, signup_date) VALUES
(1, 'Alice Tan', 'alice@email.com', '+65-9001', 'Singapore', '238801', '2025-06-15'),
(2, 'Bob Lim', 'bob@email.com', '+65-9002', 'Singapore', '059001', '2025-07-20'),
(3, 'Carol Wong', 'carol@email.com', '+60-17003', 'Kuala Lumpur', '50000', '2025-08-10'),
(4, 'David Chen', 'david@email.com', '+65-9004', 'Singapore', '238801', '2025-09-01'),
(5, 'Eva Kumar', 'eva@email.com', '+60-17005', 'Kuala Lumpur', '50450', '2025-10-15'),
(6, 'Farid Rahman', 'farid@email.com', '+60-17006', 'Kuala Lumpur', '50100', '2025-11-01'),
(7, 'Grace Ng', 'grace@email.com', '+65-9007', 'Singapore', '189720', '2025-12-01'),
(8, 'Henry Lee', 'henry@email.com', '+65-9008', 'Singapore', '059001', '2026-01-10'),
(9, 'Irene Poov', 'irene@email.com', '+60-17009', 'Kuala Lumpur', '50450', '2026-01-20'),
(10, 'James Tan', 'james@email.com', '+65-9010', 'Singapore', '238801', '2026-02-01'),
(11, 'Kate Singh', 'kate@email.com', '+60-17011', 'Penang', '10000', '2026-02-15'),
(12, 'Lim Wei', 'limwei@email.com', '+65-9012', 'Singapore', '189720', '2026-03-01'),
(13, 'Mei Lin', 'meilin@email.com', '+60-17013', 'Kuala Lumpur', '50100', '2026-03-10'),
(14, 'Nathan Goh', 'nathan@email.com', '+65-9014', 'Singapore', '238801', '2026-03-20'),
(15, 'Omar Ali', 'omar@email.com', '+60-17015', 'Kuala Lumpur', '50000', '2026-04-01'),
(16, 'Priya Nair', 'priya@email.com', '+60-17016', 'Kuala Lumpur', '50450', '2026-04-15'),
(17, 'Quincy Ho', 'quincy@email.com', '+65-9017', 'Singapore', '059001', '2026-04-20'),
(18, 'Rachel Chia', 'rachel@email.com', '+65-9018', 'Singapore', '189720', '2026-05-01'),
(19, 'Samy Velu', 'samy@email.com', '+60-17019', 'Penang', '10000', '2026-05-10'),
(20, 'Tina Ong', 'tina@email.com', '+65-9020', 'Singapore', '238801', '2026-05-15'),
-- Add 30 more...
(21, 'User 21', 'u21@email.com', '+65-9021', 'Singapore', '059001', '2025-07-01'),
(22, 'User 22', 'u22@email.com', '+60-17022', 'Kuala Lumpur', '50100', '2025-08-01'),
(23, 'User 23', 'u23@email.com', '+65-9023', 'Singapore', '238801', '2025-09-01'),
(24, 'User 24', 'u24@email.com', '+60-17024', 'Kuala Lumpur', '50000', '2025-10-01'),
(25, 'User 25', 'u25@email.com', '+65-9025', 'Singapore', '189720', '2025-11-01'),
(26, 'User 26', 'u26@email.com', '+60-17026', 'Penang', '10000', '2025-12-01'),
(27, 'User 27', 'u27@email.com', '+65-9027', 'Singapore', '238801', '2026-01-01'),
(28, 'User 28', 'u28@email.com', '+60-17028', 'Kuala Lumpur', '50450', '2026-02-01'),
(29, 'User 29', 'u29@email.com', '+65-9029', 'Singapore', '059001', '2026-03-01'),
(30, 'User 30', 'u30@email.com', '+60-17030', 'Kuala Lumpur', '50100', '2026-04-01'),
(31, 'User 31', 'u31@email.com', '+65-9031', 'Singapore', '238801', '2026-05-01'),
(32, 'User 32', 'u32@email.com', '+60-17032', 'Penang', '10000', '2025-06-01'),
(33, 'User 33', 'u33@email.com', '+65-9033', 'Singapore', '189720', '2025-07-01'),
(34, 'User 34', 'u34@email.com', '+60-17034', 'Kuala Lumpur', '50000', '2025-08-01'),
(35, 'User 35', 'u35@email.com', '+65-9035', 'Singapore', '059001', '2025-09-01'),
(36, 'User 36', 'u36@email.com', '+60-17036', 'Kuala Lumpur', '50450', '2025-10-01'),
(37, 'User 37', 'u37@email.com', '+65-9037', 'Singapore', '238801', '2025-11-01'),
(38, 'User 38', 'u38@email.com', '+60-17038', 'Kuala Lumpur', '50100', '2025-12-01'),
(39, 'User 39', 'u39@email.com', '+65-9039', 'Singapore', '189720', '2026-01-01'),
(40, 'User 40', 'u40@email.com', '+60-17040', 'Penang', '10000', '2026-02-01'),
(41, 'User 41', 'u41@email.com', '+65-9041', 'Singapore', '059001', '2026-03-01'),
(42, 'User 42', 'u42@email.com', '+60-17042', 'Kuala Lumpur', '50000', '2026-04-01'),
(43, 'User 43', 'u43@email.com', '+65-9043', 'Singapore', '238801', '2026-05-01'),
(44, 'User 44', 'u44@email.com', '+60-17044', 'Kuala Lumpur', '50450', '2025-07-01'),
(45, 'User 45', 'u45@email.com', '+65-9045', 'Singapore', '189720', '2025-08-01'),
(46, 'User 46', 'u46@email.com', '+60-17046', 'Penang', '10000', '2025-09-01'),
(47, 'User 47', 'u47@email.com', '+65-9047', 'Singapore', '059001', '2025-10-01'),
(48, 'User 48', 'u48@email.com', '+60-17048', 'Kuala Lumpur', '50100', '2025-11-01'),
(49, 'User 49', 'u49@email.com', '+65-9049', 'Singapore', '238801', '2025-12-01'),
(50, 'User 50', 'u50@email.com', '+60-17050', 'Kuala Lumpur', '50000', '2026-01-01');

-- Restaurants (15 restaurants)
INSERT INTO raw_restaurants (restaurant_id, name, cuisine, city, postal_code, rating, is_active, updated_at) VALUES
(1, 'Chicken Rice Paradise', 'Chinese', 'Singapore', '238801', 4.50, TRUE, '2026-01-01'),
(2, 'Nasi Lemak House', 'Malay', 'Kuala Lumpur', '50000', 4.20, TRUE, '2026-01-01'),
(3, 'Roti Prata Corner', 'Indian', 'Singapore', '059001', 4.00, TRUE, '2026-01-01'),
(4, 'Sakura Sushi', 'Japanese', 'Singapore', '189720', 4.70, TRUE, '2026-01-01'),
(5, 'Green Curry Thai', 'Thai', 'Kuala Lumpur', '50450', 4.30, TRUE, '2026-01-01'),
(6, 'Dim Sum Dynasty', 'Chinese', 'Singapore', '238801', 4.10, TRUE, '2026-01-01'),
(7, 'Mee Goreng Express', 'Malay', 'Kuala Lumpur', '50100', 3.80, TRUE, '2026-01-01'),
(8, 'Pizza Roma', 'Western', 'Singapore', '189720', 4.40, TRUE, '2026-01-01'),
(9, 'Char Kway Teow Master', 'Chinese', 'Penang', '10000', 4.60, TRUE, '2026-01-01'),
(10, 'Bak Kut Teh King', 'Chinese', 'Kuala Lumpur', '50000', 4.50, TRUE, '2026-01-01'),
(11, 'Satay Street', 'Malay', 'Singapore', '059001', 4.20, TRUE, '2026-01-01'),
(12, 'Laksa Legend', 'Chinese', 'Singapore', '238801', 4.30, TRUE, '2026-01-01'),
(13, 'Tandoori Nights', 'Indian', 'Kuala Lumpur', '50450', 4.00, TRUE, '2026-01-01'),
(14, 'Burger Junction', 'Western', 'Penang', '10000', 3.90, TRUE, '2026-01-01'),
(15, 'Ice Kacang Corner', 'Dessert', 'Singapore', '189720', 4.10, TRUE, '2026-01-01');

-- Menu items (3-5 per restaurant)
INSERT INTO raw_menu_items (item_id, restaurant_id, dish_name, category, price, is_vegetarian, spice_level) VALUES
-- Restaurant 1: Chicken Rice Paradise
(1, 1, 'Hainanese Chicken Rice', 'Main', 6.50, FALSE, 'Mild'),
(2, 1, 'Roast Chicken Rice', 'Main', 7.00, FALSE, 'Mild'),
(3, 1, 'Chicken Soup', 'Soup', 5.00, FALSE, 'Mild'),
-- Restaurant 2: Nasi Lemak House
(4, 2, 'Nasi Lemak Special', 'Main', 8.50, FALSE, 'Medium'),
(5, 2, 'Nasi Lemak Ayam Goreng', 'Main', 9.00, FALSE, 'Medium'),
(6, 2, 'Sambal Squid', 'Side', 7.50, FALSE, 'Hot'),
-- Restaurant 3: Roti Prata Corner
(7, 3, 'Prata Plain', 'Bread', 1.50, TRUE, 'Mild'),
(8, 3, 'Prata Egg', 'Bread', 3.50, FALSE, 'Mild'),
(9, 3, 'Mutton Murtabak', 'Main', 6.00, FALSE, 'Medium'),
-- Restaurant 4: Sakura Sushi
(10, 4, 'Salmon Sashimi Set', 'Main', 18.00, FALSE, 'Mild'),
(11, 4, 'Edamame', 'Side', 5.00, TRUE, 'Mild'),
(12, 4, 'Miso Soup', 'Soup', 4.00, TRUE, 'Mild'),
-- Restaurant 5: Green Curry Thai
(13, 5, 'Green Curry', 'Main', 14.00, FALSE, 'Hot'),
(14, 5, 'Tom Yum Soup', 'Soup', 10.00, FALSE, 'Hot'),
(15, 5, 'Mango Sticky Rice', 'Dessert', 8.00, TRUE, 'Mild'),
-- Restaurant 6-15 (abbreviated — add 3-5 items each)
(16, 6, 'Har Gow', 'Dim Sum', 5.00, FALSE, 'Mild'),
(17, 6, 'Siu Mai', 'Dim Sum', 5.00, FALSE, 'Mild'),
(18, 6, 'Char Siew Bao', 'Dim Sum', 3.50, FALSE, 'Mild'),
(19, 7, 'Mee Goreng', 'Main', 7.00, FALSE, 'Medium'),
(20, 7, 'Nasi Goreng', 'Main', 8.00, FALSE, 'Medium'),
(21, 8, 'Margherita Pizza', 'Main', 15.00, TRUE, 'Mild'),
(22, 8, 'Pasta Carbonara', 'Main', 14.00, FALSE, 'Mild'),
(23, 8, 'Caesar Salad', 'Salad', 10.00, TRUE, 'Mild'),
(24, 9, 'Char Kway Teow', 'Main', 7.50, FALSE, 'Medium'),
(25, 9, 'Hokkien Mee', 'Main', 8.00, FALSE, 'Medium'),
(26, 10, 'Bak Kut Teh', 'Soup', 12.00, FALSE, 'Mild'),
(27, 10, 'Dry Bak Kut Teh', 'Main', 14.00, FALSE, 'Mild'),
(28, 11, 'Chicken Satay (10pc)', 'Main', 10.00, FALSE, 'Medium'),
(29, 11, 'Beef Satay (10pc)', 'Main', 12.00, FALSE, 'Medium'),
(30, 12, 'Katong Laksa', 'Main', 7.00, FALSE, 'Medium'),
(31, 12, 'Curry Laksa', 'Main', 8.00, FALSE, 'Hot'),
(32, 13, 'Tandoori Chicken', 'Main', 12.00, FALSE, 'Medium'),
(33, 13, 'Garlic Naan', 'Bread', 3.00, TRUE, 'Mild'),
(34, 13, 'Butter Chicken', 'Main', 14.00, FALSE, 'Medium'),
(35, 14, 'Classic Burger', 'Main', 12.00, FALSE, 'Mild'),
(36, 14, 'Cheese Fries', 'Side', 6.00, TRUE, 'Mild'),
(37, 15, 'Ice Kacang', 'Dessert', 4.00, TRUE, 'Mild'),
(38, 15, 'Chendol', 'Dessert', 4.50, TRUE, 'Mild'),
(39, 15, 'Teh Tarik', 'Drink', 2.50, TRUE, 'Mild');

-- Orders (500 orders spanning Jan-May 2026)
-- Use generate_series to create realistic data
INSERT INTO raw_orders (order_id, customer_id, restaurant_id, order_date, total_amount, status, delivery_fee)
SELECT 
    g AS order_id,
    1 + (g % 50) AS customer_id,
    1 + (g % 15) AS restaurant_id,
    TIMESTAMP '2026-01-01' + (g % 150) * INTERVAL '1 day' + (g % 14 + 7) * INTERVAL '1 hour' + (g % 60) * INTERVAL '1 minute',
    (8 + (g % 40))::DECIMAL(8,2),
    CASE WHEN g % 15 = 0 THEN 'cancelled' ELSE 'completed' END,
    (1 + (g % 6))::DECIMAL(4,2)
FROM generate_series(1, 500) AS g;

-- Order items (2-3 items per order)
INSERT INTO raw_order_items (item_id, order_id, menu_item_id, quantity, unit_price, discount)
SELECT 
    g AS item_id,
    1 + (g % 500) AS order_id,
    1 + (g % 39) AS menu_item_id,
    1 + (g % 3) AS quantity,
    (3 + (g % 20))::DECIMAL(6,2),
    CASE WHEN g % 10 = 0 THEN 1.00 ELSE 0.00 END::DECIMAL(4,2)
FROM generate_series(1, 1200) AS g;

-- Payments
INSERT INTO raw_payments (payment_id, order_id, method, amount, status, paid_at)
SELECT 
    g AS payment_id,
    g AS order_id,
    CASE g % 4 WHEN 0 THEN 'GrabPay' WHEN 1 THEN 'Credit Card' WHEN 2 THEN 'Cash' ELSE 'FPX' END,
    (8 + (g % 40))::DECIMAL(8,2),
    CASE WHEN g % 25 = 0 THEN 'failed' ELSE 'success' END,
    TIMESTAMP '2026-01-01' + (g % 150) * INTERVAL '1 day' + (g % 14 + 7) * INTERVAL '1 hour'
FROM generate_series(1, 480) AS g;  -- 20 orders have no payment
```

### Step 3: Create Warehouse Schema (30 min)

**`schema/01_dim_date.sql`:**
```sql
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,  -- YYYYMMDD format
    full_date DATE NOT NULL UNIQUE,
    day_of_week VARCHAR(10),
    day_number INT,
    day_name_short VARCHAR(3),
    week_of_year INT,
    month_number INT,
    month_name VARCHAR(10),
    quarter INT,
    year INT,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    holiday_name VARCHAR(50)
);
```

**`schema/02_dim_customer.sql`:**
```sql
CREATE TABLE dim_customer (
    customer_key SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    phone VARCHAR(20),
    city VARCHAR(50),
    postal_code VARCHAR(10),
    country VARCHAR(50),
    signup_date DATE,
    customer_segment VARCHAR(20) DEFAULT 'New',
    is_active BOOLEAN DEFAULT TRUE,
    -- SCD Type 2 columns
    valid_from TIMESTAMP NOT NULL DEFAULT NOW(),
    valid_to TIMESTAMP NOT NULL DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_dim_customer_natural ON dim_customer(customer_id, is_current);
```

**`schema/03_dim_restaurant.sql`:**
```sql
CREATE TABLE dim_restaurant (
    restaurant_key SERIAL PRIMARY KEY,
    restaurant_id INT NOT NULL,
    restaurant_name VARCHAR(100),
    cuisine_type VARCHAR(50),
    cuisine_category VARCHAR(30),
    city VARCHAR(50),
    postal_code VARCHAR(10),
    rating DECIMAL(3,2),
    price_range VARCHAR(10),
    is_active BOOLEAN DEFAULT TRUE,
    valid_from TIMESTAMP NOT NULL DEFAULT NOW(),
    valid_to TIMESTAMP NOT NULL DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_dim_restaurant_natural ON dim_restaurant(restaurant_id, is_current);
```

**`schema/04_dim_item.sql`:**
```sql
CREATE TABLE dim_item (
    item_key SERIAL PRIMARY KEY,
    item_id INT NOT NULL,
    restaurant_id INT,
    dish_name VARCHAR(100),
    category VARCHAR(30),
    price DECIMAL(8,2),
    is_vegetarian BOOLEAN,
    spice_level VARCHAR(10),
    is_available BOOLEAN DEFAULT TRUE
);
```

**`schema/05_dim_payment_method.sql`:**
```sql
CREATE TABLE dim_payment_method (
    payment_method_key SERIAL PRIMARY KEY,
    method_name VARCHAR(30) NOT NULL UNIQUE,
    method_category VARCHAR(20)  -- 'Digital', 'Cash', 'Bank'
);

INSERT INTO dim_payment_method (method_name, method_category) VALUES
('GrabPay', 'Digital'),
('Credit Card', 'Digital'),
('Cash', 'Cash'),
('FPX', 'Bank');
```

**`schema/06_fact_order_items.sql`:**
```sql
CREATE TABLE fact_order_items (
    fact_id SERIAL PRIMARY KEY,
    order_number INT,                    -- degenerate dimension
    date_key INT REFERENCES dim_date(date_key),
    customer_key INT REFERENCES dim_customer(customer_key),
    restaurant_key INT REFERENCES dim_restaurant(restaurant_key),
    item_key INT REFERENCES dim_item(item_key),
    payment_method_key INT REFERENCES dim_payment_method(payment_method_key),
    -- Measures
    quantity INT NOT NULL,
    unit_price DECIMAL(8,2) NOT NULL,
    discount_amount DECIMAL(6,2) DEFAULT 0,
    gross_amount DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
    net_amount DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price - discount_amount) STORED,
    delivery_fee DECIMAL(5,2) DEFAULT 0,
    order_status VARCHAR(20)
);

-- Performance indexes
CREATE INDEX idx_fact_date ON fact_order_items(date_key);
CREATE INDEX idx_fact_customer ON fact_order_items(customer_key);
CREATE INDEX idx_fact_restaurant ON fact_order_items(restaurant_key);
CREATE INDEX idx_fact_item ON fact_order_items(item_key);
```

### Step 4: Write ETL Scripts (1 hour)

**`etl/01_load_dim_customer.sql`** — SCD Type 2:
```sql
-- MakanExpress ETL: Load dim_customer (SCD Type 2)
-- Run this AFTER source data is loaded

-- Step 1: Expire changed records
UPDATE dim_customer d
SET 
    valid_to = NOW(),
    is_current = FALSE
FROM raw_customers s
WHERE d.customer_id = s.customer_id
AND d.is_current = TRUE
AND (
    d.customer_name IS DISTINCT FROM s.name
    OR d.email IS DISTINCT FROM s.email
    OR d.city IS DISTINCT FROM s.city
    OR d.postal_code IS DISTINCT FROM s.postal_code
);

-- Step 2: Insert new records (both changed + brand new customers)
INSERT INTO dim_customer (customer_id, customer_name, email, phone, city, postal_code, 
                          country, signup_date, customer_segment, is_active, valid_from, valid_to, is_current)
SELECT 
    s.customer_id,
    s.name,
    s.email,
    s.phone,
    s.city,
    s.postal_code,
    CASE WHEN s.city IN ('Singapore') THEN 'Singapore' ELSE 'Malaysia' END,
    s.signup_date,
    CASE 
        WHEN s.signup_date >= CURRENT_DATE - INTERVAL '30 days' THEN 'New'
        ELSE 'Regular'
    END,
    TRUE,
    NOW(),
    '9999-12-31 23:59:59',
    TRUE
FROM raw_customers s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer d 
    WHERE d.customer_id = s.customer_id 
    AND d.is_current = TRUE
    AND d.customer_name = s.name
    AND d.email = s.email
    AND d.city = s.city
    AND d.postal_code = s.postal_code
);

-- Step 3: Update segments based on order history
WITH customer_orders AS (
    SELECT 
        dc.customer_key,
        COUNT(DISTINCT fo.order_number) AS order_count,
        SUM(fo.net_amount) AS total_spent
    FROM dim_customer dc
    LEFT JOIN fact_order_items fo ON dc.customer_key = fo.customer_key
    WHERE dc.is_current = TRUE
    GROUP BY dc.customer_key
)
UPDATE dim_customer d
SET customer_segment = CASE
    WHEN co.total_spent >= 500 THEN 'VIP'
    WHEN co.total_spent >= 200 THEN 'Gold'
    WHEN co.total_spent >= 50 THEN 'Regular'
    WHEN co.order_count >= 1 THEN 'New'
    ELSE 'Prospect'
END
FROM customer_orders co
WHERE d.customer_key = co.customer_key AND d.is_current = TRUE;
```

**`etl/04_load_fact.sql`:**
```sql
-- MakanExpress ETL: Load fact_order_items

BEGIN;

INSERT INTO fact_order_items (
    order_number, date_key, customer_key, restaurant_key, 
    item_key, payment_method_key,
    quantity, unit_price, discount_amount, delivery_fee, order_status
)
SELECT 
    o.order_id,
    dd.date_key,
    dc.customer_key,
    dr.restaurant_key,
    di.item_key,
    COALESCE(dpm.payment_method_key, 1),
    oi.quantity,
    oi.unit_price,
    oi.discount,
    o.delivery_fee,
    o.status
FROM raw_order_items oi
JOIN raw_orders o ON oi.order_id = o.order_id
JOIN dim_date dd ON dd.full_date = DATE(o.order_date)
JOIN dim_customer dc ON dc.customer_id = o.customer_id 
    AND dc.valid_from <= o.order_date 
    AND dc.valid_to > o.order_date
JOIN dim_restaurant dr ON dr.restaurant_id = o.restaurant_id 
    AND dr.is_current = TRUE
JOIN dim_item di ON di.item_id = oi.menu_item_id
LEFT JOIN raw_payments p ON p.order_id = o.order_id AND p.status = 'success'
LEFT JOIN dim_payment_method dpm ON dpm.method_name = p.method
WHERE NOT EXISTS (
    SELECT 1 FROM fact_order_items f WHERE f.order_number = o.order_id AND f.item_key = di.item_key
);

COMMIT;
```

### Step 5: Write 5 Analytical Queries (1 hour)

**`analytics/revenue_by_city_cuisine.sql`:**
```sql
-- Revenue by city and cuisine type
SELECT 
    dr.city,
    dr.cuisine_type,
    dd.month_name,
    dd.year,
    COUNT(DISTINCT fo.order_number) AS total_orders,
    SUM(fo.net_amount) AS total_revenue,
    ROUND(AVG(fo.net_amount), 2) AS avg_order_value,
    COUNT(DISTINCT fo.customer_key) AS unique_customers
FROM fact_order_items fo
JOIN dim_date dd ON fo.date_key = dd.date_key
JOIN dim_restaurant dr ON fo.restaurant_key = dr.restaurant_key
WHERE fo.order_status = 'completed'
GROUP BY dr.city, dr.cuisine_type, dd.month_name, dd.year
ORDER BY dr.city, total_revenue DESC;
```

**`analytics/customer_retention.sql`:**
```sql
-- Monthly cohort retention
WITH first_orders AS (
    SELECT 
        dc.customer_key,
        DATE_TRUNC('month', MIN(dd.full_date)) AS cohort_month
    FROM fact_order_items fo
    JOIN dim_date dd ON fo.date_key = dd.date_key
    JOIN dim_customer dc ON fo.customer_key = dc.customer_key
    WHERE fo.order_status = 'completed'
    GROUP BY dc.customer_key
),
active_months AS (
    SELECT DISTINCT
        fo.customer_key,
        DATE_TRUNC('month', dd.full_date) AS active_month
    FROM fact_order_items fo
    JOIN dim_date dd ON fo.date_key = dd.date_key
    WHERE fo.order_status = 'completed'
),
retention AS (
    SELECT 
        fo.cohort_month,
        am.active_month,
        EXTRACT(YEAR FROM am.active_month) * 12 + EXTRACT(MONTH FROM am.active_month)
        - EXTRACT(YEAR FROM fo.cohort_month) * 12 - EXTRACT(MONTH FROM fo.cohort_month) AS month_offset,
        COUNT(DISTINCT am.customer_key) AS active_users
    FROM first_orders fo
    JOIN active_months am ON fo.customer_key = am.customer_key
    GROUP BY fo.cohort_month, am.active_month
),
cohort_sizes AS (
    SELECT cohort_month, COUNT(DISTINCT customer_key) AS size
    FROM first_orders GROUP BY cohort_month
)
SELECT 
    TO_CHAR(r.cohort_month, 'YYYY-MM') AS cohort,
    cs.size AS cohort_size,
    TO_CHAR(r.active_month, 'YYYY-MM') AS active_month,
    r.month_offset,
    r.active_users,
    ROUND(r.active_users::DECIMAL / cs.size * 100, 1) AS retention_pct
FROM retention r
JOIN cohort_sizes cs ON r.cohort_month = cs.cohort_month
ORDER BY r.cohort_month, r.month_offset;
```

**`analytics/restaurant_trends.sql`:**
```sql
-- Restaurant performance trends with MoM growth
WITH monthly AS (
    SELECT 
        dr.restaurant_name,
        dr.cuisine_type,
        dr.city,
        DATE_TRUNC('month', dd.full_date) AS month,
        SUM(fo.net_amount) AS revenue,
        COUNT(DISTINCT fo.order_number) AS orders,
        COUNT(DISTINCT fo.customer_key) AS customers
    FROM fact_order_items fo
    JOIN dim_date dd ON fo.date_key = dd.date_key
    JOIN dim_restaurant dr ON fo.restaurant_key = dr.restaurant_key
    WHERE fo.order_status = 'completed'
    GROUP BY dr.restaurant_name, dr.cuisine_type, dr.city, DATE_TRUNC('month', dd.full_date)
)
SELECT 
    restaurant_name,
    cuisine_type,
    city,
    TO_CHAR(month, 'YYYY-MM') AS month,
    revenue,
    orders,
    customers,
    LAG(revenue) OVER (PARTITION BY restaurant_name ORDER BY month) AS prev_month_rev,
    ROUND(
        (revenue - LAG(revenue) OVER (PARTITION BY restaurant_name ORDER BY month)) /
        NULLIF(LAG(revenue) OVER (PARTITION BY restaurant_name ORDER BY month), 0) * 100, 1
    ) AS mom_growth_pct,
    CASE 
        WHEN revenue > LAG(revenue) OVER (PARTITION BY restaurant_name ORDER BY month) THEN '📈 Growing'
        WHEN revenue < LAG(revenue) OVER (PARTITION BY restaurant_name ORDER BY month) THEN '📉 Declining'
        ELSE '➡️ Stable'
    END AS trend
FROM monthly
ORDER BY restaurant_name, month;
```

**`analytics/rfm_segmentation.sql`:**
```sql
-- RFM Customer Segmentation
WITH rfm AS (
    SELECT 
        dc.customer_key,
        dc.customer_name,
        dc.city,
        CURRENT_DATE - MAX(dd.full_date) AS recency_days,
        COUNT(DISTINCT fo.order_number) AS frequency,
        SUM(fo.net_amount) AS monetary
    FROM fact_order_items fo
    JOIN dim_date dd ON fo.date_key = dd.date_key
    JOIN dim_customer dc ON fo.customer_key = dc.customer_key
    WHERE fo.order_status = 'completed'
    GROUP BY dc.customer_key, dc.customer_name, dc.city
),
scored AS (
    SELECT 
        *,
        NTILE(5) OVER (ORDER BY recency_days DESC) AS r_score,
        NTILE(5) OVER (ORDER BY frequency) AS f_score,
        NTILE(5) OVER (ORDER BY monetary) AS m_score
    FROM rfm
)
SELECT 
    customer_name,
    city,
    recency_days,
    frequency,
    monetary,
    r_score || f_score || m_score AS rfm_code,
    CASE
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN '🏆 Champions'
        WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN '💚 Loyal'
        WHEN r_score >= 4 AND f_score <= 2 THEN '🆕 New'
        WHEN r_score <= 2 AND f_score >= 3 AND m_score >= 3 THEN '⚠️ At Risk'
        WHEN r_score <= 2 AND f_score <= 2 AND m_score <= 2 THEN '💀 Lost'
        ELSE '😊 Average'
    END AS segment
FROM scored
ORDER BY monetary DESC;
```

**`analytics/top_dishes.sql`:**
```sql
-- Top dishes by revenue and order frequency
SELECT 
    di.dish_name,
    di.category,
    dr.restaurant_name,
    dr.cuisine_type,
    SUM(fo.quantity) AS total_qty,
    SUM(fo.net_amount) AS total_revenue,
    COUNT(DISTINCT fo.order_number) AS times_ordered,
    COUNT(DISTINCT fo.customer_key) AS unique_customers,
    ROUND(AVG(fo.unit_price), 2) AS avg_price
FROM fact_order_items fo
JOIN dim_item di ON fo.item_key = di.item_key
JOIN dim_restaurant dr ON fo.restaurant_key = dr.restaurant_key
WHERE fo.order_status = 'completed'
GROUP BY di.dish_name, di.category, dr.restaurant_name, dr.cuisine_type
ORDER BY total_revenue DESC
LIMIT 20;
```

### Step 6: Write Data Quality Checks (30 min)

**`quality/orphan_checks.sql`:**
```sql
-- Data quality: Check for orphan records in fact table
SELECT 'orphan_date' AS check_name,
    COUNT(*) AS orphan_count
FROM fact_order_items fo
LEFT JOIN dim_date dd ON fo.date_key = dd.date_key
WHERE dd.date_key IS NULL

UNION ALL

SELECT 'orphan_customer',
    COUNT(*)
FROM fact_order_items fo
LEFT JOIN dim_customer dc ON fo.customer_key = dc.customer_key
WHERE dc.customer_key IS NULL

UNION ALL

SELECT 'orphan_restaurant',
    COUNT(*)
FROM fact_order_items fo
LEFT JOIN dim_restaurant dr ON fo.restaurant_key = dr.restaurant_key
WHERE dr.restaurant_key IS NULL

UNION ALL

SELECT 'orphan_item',
    COUNT(*)
FROM fact_order_items fo
LEFT JOIN dim_item di ON fo.item_key = di.item_key
WHERE di.item_key IS NULL;

-- All should return 0
```

**`quality/scd_validation.sql`:**
```sql
-- SCD Type 2 validation: each natural key should have exactly 1 current row
SELECT 'customer_multiple_current' AS issue, customer_id, COUNT(*) AS cnt
FROM dim_customer WHERE is_current = TRUE
GROUP BY customer_id HAVING COUNT(*) > 1

UNION ALL

SELECT 'restaurant_multiple_current', restaurant_id, COUNT(*)
FROM dim_restaurant WHERE is_current = TRUE
GROUP BY restaurant_id HAVING COUNT(*) > 1;

-- Check for gaps in validity periods
SELECT 'customer_gap' AS issue, d1.customer_id,
    d1.valid_to AS gap_start, d2.valid_from AS gap_end
FROM dim_customer d1
JOIN dim_customer d2 ON d1.customer_id = d2.customer_id
    AND d2.valid_from > d1.valid_to
WHERE d1.is_current = FALSE
AND d2.valid_from - d1.valid_to > INTERVAL '1 second'
LIMIT 10;
```

### Step 7: Write README.md (30 min)

Create a professional README with:

```markdown
# 🍜 MakanExpress Food Delivery Data Warehouse

A complete star schema data warehouse for a food delivery platform operating in Singapore and Kuala Lumpur, built with PostgreSQL.

## 🏗️ Architecture

**Source (OLTP) → ETL (SQL) → Warehouse (Star Schema) → Analytics**

### Schema Design

```
                    dim_date ──────────┐
                                       │
              dim_customer ────────────┤
                                       │
dim_payment_method ──── fact_order_items ──── dim_restaurant
                                       │
                  dim_item ────────────┘
```

### Grain
`fact_order_items`: **One row per order line item** (most granular)

## 📊 Key Features
- SCD Type 2 for customers and restaurants (track history)
- Surrogate keys throughout
- Conformed dim_date dimension
- 5 analytical queries (revenue, retention, trends, RFM, top dishes)
- Data quality checks (orphan detection, SCD validation)

## 🚀 Quick Start
1. Create source tables: `psql -f seed/sample_data.sql`
2. Create warehouse schema: `psql -f schema/01_dim_date.sql` ... etc.
3. Populate dim_date: `psql -f seed/generate_dim_date.sql`
4. Run ETL: `psql -f etl/05_full_etl.sql`
5. Run analytics: `psql -f analytics/revenue_by_city_cuisine.sql`

## 🛠️ Tech Stack
- PostgreSQL 15+
- Pure SQL (no external tools)
- Git for version control
```

---

## 🌙 Evening: Push to GitHub & Celebrate

### Final Checklist
- [ ] All schema files created and tested
- [ ] ETL scripts run without errors
- [ ] All 5 analytical queries produce results
- [ ] Data quality checks all pass (0 orphans)
- [ ] README.md is professional and complete
- [ ] ERD diagram included in docs/
- [ ] Committed and pushed to GitHub

### 🎉 Week 4 Complete!

**What you learned this week:**
- Advanced SQL joins (self-joins, anti-joins, lateral joins)
- Query optimization (EXPLAIN, indexing, materialized views)
- Star schema design (facts, dimensions, grain, surrogate keys)
- Slowly Changing Dimensions (Types 1, 2, 3)
- Advanced window functions (RFM, cohort, sessionization)
- Built a complete data warehouse portfolio project

**Portfolio projects so far:**
1. ✅ Retail Sales Analysis (SQL)
2. ✅ Crypto Data Pipeline (Python ETL)
3. ✅ Food Delivery Analytics (dbt)
4. ✅ **Food Delivery Warehouse (Star Schema + SCD)** ← NEW!

---

*Week 4 done! Next week: Airflow — orchestrating all your pipelines.* 🎊
