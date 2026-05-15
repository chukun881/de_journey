# 📅 Day 5 — Monday, 19 May 2026
# SQL Advanced: UNION, DDL, Data Types, Indexes, Query Performance, String/Date Functions

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Use UNION, INTERSECT, EXCEPT to combine query results
- Create, alter, and drop tables (DDL — Data Definition Language)
- Understand PostgreSQL data types and when to use which
- Write queries with string and date functions (you'll use these daily)
- Understand what indexes are and why they matter for performance
- Use EXPLAIN to see how PostgreSQL executes your queries

**Why this matters:** Days 1-4 taught you to WRITE queries. Today teaches you to MANAGE the database itself — create tables, choose the right data types, and write performant queries. As a data engineer, you'll design tables and optimize queries that run on millions of rows.

---

## ☀️ BLOCK 1: UNION, INTERSECT, EXCEPT (Morning, ~1 hour)

---

### Task 1: UNION — Combine Results (20 min)

UNION stacks the results of two queries vertically (one on top of the other).

```sql
-- 🔹 UNION — combine results from two queries
-- "Show ALL names: both employee names and customer names"
SELECT name FROM employees
UNION
SELECT name FROM customers;
-- Rules: both queries must have same number of columns
--         column types must be compatible
--         UNION removes duplicates by default

-- 🔹 UNION ALL — keep duplicates
SELECT city FROM employees
UNION ALL
SELECT city FROM customers;
-- UNION ALL is faster because it doesn't check for duplicates
-- Use UNION ALL when you know there are no dupes or you WANT dupes

-- 🔹 Practical use: combine data from similar tables
-- "Show all activity: both orders and project assignments in one timeline"
SELECT 'order' AS activity_type, c.name, o.order_date AS activity_date
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
UNION ALL
SELECT 'project' AS activity_type, e.name, p.start_date AS activity_date
FROM employee_project ep
INNER JOIN employees e ON ep.emp_id = e.emp_id
INNER JOIN projects p ON ep.project_id = p.project_id
ORDER BY activity_date;
```

### Task 2: INTERSECT and EXCEPT (20 min)

```sql
-- 🔹 INTERSECT — rows that appear in BOTH queries
-- "Which cities have BOTH employees and customers?"
SELECT city FROM employees
INTERSECT
SELECT city FROM customers;

-- 🔹 EXCEPT — rows in first query but NOT in second
-- "Which cities have employees but NO customers?"
SELECT city FROM employees
EXCEPT
SELECT city FROM customers;
```

**🔥 Practice exercises:**

1. Find names that appear in BOTH the employees table AND the customers table
2. List all departments that have employees but NO projects assigned
3. Combine all people (employees + customers) into one list with a column showing their role ('Employee' or 'Customer')

<details>
<summary>📖 Answers</summary>

```sql
-- 1
SELECT name FROM employees
INTERSECT
SELECT name FROM customers;

-- 2
SELECT DISTINCT d.dept_name
FROM departments d
INNER JOIN employees e ON d.dept_id = e.dept_id
EXCEPT
SELECT DISTINCT d.dept_name
FROM departments d
INNER JOIN employees e ON d.dept_id = e.dept_id
INNER JOIN employee_project ep ON e.emp_id = ep.emp_id;

-- 3
SELECT name, 'Employee' AS role FROM employees
UNION ALL
SELECT name, 'Customer' AS role FROM customers
ORDER BY name;
```
</details>

---

### Task 3: SQL Execution Order Recap + Quiz (20 min)

Answer these WITHOUT looking at notes. Write down your answers, then check.

1. Which runs first: WHERE or HAVING?
2. Which runs first: SELECT or ORDER BY?
3. Can you use a column alias in WHERE? Why or why not?
4. Can you use a column alias in ORDER BY?
5. What's the full execution order?

<details>
<summary>📖 Answers</summary>

1. WHERE runs first (it filters rows before grouping)
2. FROM runs first, but between SELECT and ORDER BY: SELECT runs first
3. No — WHERE runs before SELECT, so aliases don't exist yet
4. Yes — ORDER BY runs after SELECT
5. FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
</details>

---

## 🔥 BLOCK 2: DDL — Creating & Managing Tables (Morning/Afternoon, ~1.5 hours)

---

### Task 4: Data Types in PostgreSQL (30 min)

When you create a table, every column needs a data type. Here are the ones you'll use 95% of the time:

| Data Type | What it stores | Example | When to use |
|-----------|---------------|---------|-------------|
| `INTEGER` / `BIGINT` | Whole numbers | 42, 1000000 | IDs, counts, quantities |
| `SERIAL` / `BIGSERIAL` | Auto-incrementing integer | 1, 2, 3... | Primary keys |
| `DECIMAL(p,s)` | Exact decimal | 12345.67 | Money, precise measurements |
| `NUMERIC(p,s)` | Same as DECIMAL | 12345.67 | Same as DECIMAL |
| `VARCHAR(n)` | Variable-length text | "Hello" | Names, emails, addresses |
| `TEXT` | Unlimited text | Long descriptions | Blog posts, descriptions |
| `BOOLEAN` | True or False | TRUE, FALSE | Flags (is_active, is_verified) |
| `DATE` | Date only | '2024-01-15' | Hire dates, birthdays |
| `TIMESTAMP` | Date + time | '2024-01-15 14:30:00' | Created_at, updated_at |
| `TIMESTAMPTZ` | Date + time + timezone | '2024-01-15 14:30:00+08' | Event logs (always use this!) |
| `JSONB` | JSON data | {"key": "value"} | Semi-structured data |

> 🧠 **Key decisions:**
> - `INTEGER` vs `BIGINT`: If you might have > 2 billion rows, use BIGINT for IDs
> - `VARCHAR(n)` vs `TEXT`: VARCHAR limits length, TEXT is unlimited. For short fields like emails, VARCHAR is better (validation). For descriptions, TEXT is fine.
> - `TIMESTAMP` vs `TIMESTAMPTZ`: ALWAYS use TIMESTAMPTZ for anything user-facing or multi-region. Singapore and Malaysia are in different timezones (sort of — both GMT+8, but in general, always store with timezone).
> - `DECIMAL` vs `FLOAT`: Use DECIMAL for money (exact). FLOAT has rounding errors. Never use FLOAT for currency.
> - `JSONB` is PostgreSQL superpower — stores JSON natively, queryable. Other databases need separate tables for flexible data.

---

### Task 5: CREATE, ALTER, DROP (30 min)

```sql
-- 🔹 CREATE TABLE with all the bells and whistles
CREATE TABLE products_new (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,       -- NOT NULL = required field
    description TEXT,                          -- optional (can be NULL)
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),  -- must be positive
    stock INTEGER NOT NULL DEFAULT 0,          -- defaults to 0 if not specified
    category VARCHAR(50),
    created_at TIMESTAMPTZ DEFAULT NOW(),      -- auto-fills with current time
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE
);

-- 🔹 Constraints explained:
-- PRIMARY KEY → unique + not null (identifies each row)
-- NOT NULL → this field is required
-- CHECK → custom validation rule
-- DEFAULT → auto-fill value if not provided
-- REFERENCES → foreign key (links to another table)
-- UNIQUE → no duplicate values allowed

-- 🔹 ALTER TABLE — modify existing table
ALTER TABLE products_new ADD COLUMN weight DECIMAL(6,2);  -- add column
ALTER TABLE products_new DROP COLUMN weight;                -- remove column
ALTER TABLE products_new RENAME COLUMN product_name TO name; -- rename column
ALTER TABLE products_new ALTER COLUMN price SET DEFAULT 0.00; -- set default

-- 🔹 DROP TABLE — delete table permanently
DROP TABLE IF EXISTS products_new;
-- ⚠️ THIS DELETES ALL DATA. Be careful!
-- IF EXISTS prevents error if table doesn't exist

-- 🔹 TRUNCATE — delete all rows but keep the table structure
TRUNCATE TABLE products_new;
-- Faster than DELETE FROM products_new (no logging)
```

**🔥 Practice exercises:**

4. Create a table `order_logs` with: log_id (auto PK), order_id (int, not null), action (varchar(20), not null), action_timestamp (auto-filled current time), details (text, optional)
5. Add a column `ip_address VARCHAR(45)` to the order_logs table
6. Create a table `categories` with proper constraints: category_id (PK), name (unique, not null), description (optional), created_at (auto-filled)

<details>
<summary>📖 Answers</summary>

```sql
-- 4
CREATE TABLE order_logs (
    log_id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    action VARCHAR(20) NOT NULL,
    action_timestamp TIMESTAMPTZ DEFAULT NOW(),
    details TEXT
);

-- 5
ALTER TABLE order_logs ADD COLUMN ip_address VARCHAR(45);

-- 6
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```
</details>

---

### Task 6: Indexes — Making Queries Fast (30 min)

```sql
-- 🔹 What is an index?
-- An index is like a book's table of contents.
-- Without index: PostgreSQL reads EVERY row to find matches (full table scan)
-- With index: PostgreSQL jumps directly to the right rows

-- 🔹 When you use PRIMARY KEY or UNIQUE, PostgreSQL auto-creates an index
-- For other columns, you create them manually:

-- Create index on a column you frequently filter by
CREATE INDEX idx_employees_dept ON employees(dept_id);
-- Now: WHERE dept_id = 1 is MUCH faster

-- Create index on multiple columns (composite index)
CREATE INDEX idx_employees_dept_city ON employees(dept_id, city);
-- Good for: WHERE dept_id = 1 AND city = 'Singapore'

-- 🔹 When to create indexes:
-- ✅ Columns used in WHERE clauses frequently
-- ✅ Columns used in JOIN conditions
-- ✅ Columns used in ORDER BY
-- ❌ Columns with very few unique values (like boolean)
-- ❌ Tables with few rows (index overhead > benefit)
-- ❌ Columns that are updated frequently (each update also updates the index)
```

### Task 7: EXPLAIN — See How PostgreSQL Runs Your Query (15 min)

```sql
-- 🔹 EXPLAIN shows the query execution plan
EXPLAIN SELECT * FROM employees WHERE dept_id = 1;
-- Look for "Seq Scan" (sequential scan = reads every row = slow on big tables)
-- vs "Index Scan" (uses index = fast)

-- 🔹 EXPLAIN ANALYZE actually runs the query and shows real timing
EXPLAIN ANALYZE SELECT * FROM employees WHERE dept_id = 1;
-- Shows actual time in milliseconds
-- On our tiny 19-row table, difference is negligible
-- On 10 million rows, the difference is 10ms vs 5000ms

-- ⚠️ Don't run EXPLAIN ANALYZE on INSERT/UPDATE/DELETE — it actually modifies data!
-- Use just EXPLAIN (without ANALYZE) for write queries
```

> 🧠 **As a data engineer, you WILL be asked to optimize slow queries.** The workflow:
> 1. Run `EXPLAIN ANALYZE` on the slow query
> 2. Look for Seq Scan on large tables → add an index
> 3. Look for nested loops with huge row counts → maybe restructure the query
> 4. Re-run EXPLAIN ANALYZE to verify improvement

---

## 🔥 BLOCK 3: String & Date Functions (Afternoon, ~1.5 hours)

---

### Task 8: String Functions (30 min)

You'll manipulate strings constantly — cleaning names, formatting emails, parsing log data.

```sql
-- 🔹 Basic string operations
SELECT 
    name,
    UPPER(name) AS upper_name,           -- 'ALICE TAN'
    LOWER(name) AS lower_name,           -- 'alice tan'
    LENGTH(name) AS name_length,         -- 9
    LEFT(name, 3) AS first_3,            -- 'Ali'
    RIGHT(name, 3) AS last_3,            -- 'Tan'
    SUBSTRING(name, 1, 5) AS first_5,    -- 'Alice' (position 1, length 5)
    TRIM('  hello  ') AS trimmed,        -- 'hello' (removes spaces)
    REPLACE(name, 'Tan', '***') AS censored -- 'Alice ***'
FROM employees
LIMIT 5;

-- 🔹 Concatenation
SELECT 
    name || ' (' || city || ')' AS name_with_city,  -- using ||
    CONCAT(name, ' - ', city) AS concat_version     -- using CONCAT()
FROM employees
LIMIT 5;

-- 🔹 String to split (very common in data cleaning!)
-- Parsing email to get domain
SELECT 
    email,
    SPLIT_PART(email, '@', 2) AS email_domain
FROM employees;
-- SPLIT_PART splits by delimiter and returns the Nth part
-- 'alice@company.com' split by '@' → part 2 = 'company.com'

-- 🔹 Pattern matching
SELECT name FROM employees WHERE name LIKE '%Tan%';     -- contains 'Tan'
SELECT name FROM employees WHERE name ILIKE '%tan%';    -- case-insensitive
SELECT name FROM employees WHERE name ~ '^[A-E]';       -- regex: starts with A-E
```

**🔥 Practice exercises:**

7. Show each employee's first name only (split by space, take first part)
8. Create email-style usernames: first initial + last name in lowercase (e.g., 'Alice Tan' → 'atan')
9. Find employees whose name and city start with the same letter

<details>
<summary>📖 Answers</summary>

```sql
-- 7
SELECT name, SPLIT_PART(name, ' ', 1) AS first_name FROM employees;

-- 8
SELECT name, 
    LOWER(LEFT(name, 1) || SPLIT_PART(name, ' ', 2)) AS username
FROM employees;

-- 9
SELECT name, city
FROM employees
WHERE LEFT(name, 1) = LEFT(city, 1);
```
</details>

---

### Task 9: Date Functions (30 min)

Date manipulation is EVERYWHERE in data engineering — filtering by date, calculating durations, grouping by month/year.

```sql
-- 🔹 Extracting parts of a date
SELECT name, hire_date,
    EXTRACT(YEAR FROM hire_date) AS hire_year,
    EXTRACT(MONTH FROM hire_date) AS hire_month,
    EXTRACT(DAY FROM hire_date) AS hire_day,
    EXTRACT(DOW FROM hire_date) AS day_of_week,  -- 0=Sunday, 6=Saturday
    EXTRACT(QUARTER FROM hire_date) AS quarter
FROM employees;

-- 🔹 Date arithmetic
SELECT name, hire_date,
    hire_date + INTERVAL '30 days' AS thirty_days_later,
    hire_date - INTERVAL '6 months' AS six_months_before,
    CURRENT_DATE AS today,
    CURRENT_DATE - hire_date AS days_since_hire,     -- returns integer (days)
    AGE(CURRENT_DATE, hire_date) AS tenure             -- returns interval like "2 years 3 mons"
FROM employees;

-- 🔹 Date truncation (round down to nearest unit)
SELECT name, hire_date,
    DATE_TRUNC('month', hire_date) AS first_of_month,    -- '2023-01-01'
    DATE_TRUNC('year', hire_date) AS first_of_year,      -- '2023-01-01'
    DATE_TRUNC('week', hire_date) AS monday_of_week       -- rounds to Monday
FROM employees;

-- 🔹 Formatting dates
SELECT name, hire_date,
    TO_CHAR(hire_date, 'DD/MM/YYYY') AS formatted_date,    -- '15/01/2023'
    TO_CHAR(hire_date, 'YYYY-MM') AS year_month,           -- '2023-01'
    TO_CHAR(hire_date, 'Day, DD Mon YYYY') AS nice_date    -- 'Sunday, 15 Jan 2023'
FROM employees;
```

**🔥 Practice exercises:**

10. Show employees hired in Q1 (January-March) of any year
11. Calculate each employee's tenure in years (rounded to 1 decimal)
12. Group employees by the quarter they were hired in. Show quarter and count.
13. Show employees whose anniversary (same month-day) is coming up in the next 30 days from today.

<details>
<summary>📖 Answers</summary>

```sql
-- 10
SELECT name, hire_date FROM employees
WHERE EXTRACT(MONTH FROM hire_date) IN (1, 2, 3);

-- 11
SELECT name, hire_date,
    ROUND(EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date)) 
          + EXTRACT(MONTH FROM AGE(CURRENT_DATE, hire_date)) / 12.0, 1) AS tenure_years
FROM employees;

-- or simpler approximation:
-- ROUND((CURRENT_DATE - hire_date) / 365.25, 1) AS tenure_years

-- 12
SELECT 
    EXTRACT(YEAR FROM hire_date) AS hire_year,
    'Q' || EXTRACT(QUARTER FROM hire_date) AS quarter,
    COUNT(*) AS hires
FROM employees
GROUP BY hire_year, quarter
ORDER BY hire_year, quarter;

-- 13
SELECT name, hire_date,
    TO_CHAR(hire_date, 'MM-DD') AS anniversary
FROM employees
WHERE TO_CHAR(hire_date, 'MM-DD') BETWEEN 
    TO_CHAR(CURRENT_DATE, 'MM-DD') 
    AND TO_CHAR(CURRENT_DATE + INTERVAL '30 days', 'MM-DD');
```
</details>

---

## 🌙 BLOCK 4: Practice + Consolidate (Evening, ~1.5 hours)

---

### Task 10: HackerRank (30 min)

Complete these on https://www.hackerrank.com/domains/sql:

1. **Weather Observation Station 5** — shortest and longest city names (LENGTH)
2. **Weather Observation Station 8** — even ID cities
3. **Weather Observation Station 12** — exclude specific patterns
4. **Employee Names** — basic select + order
5. **Employee Salaries** — filter + order

---

### Task 11: Mini-Project — Create a Complete Database Schema (45 min)

Design and create the database for a **food delivery app** (like GrabFood/Foodpanda). This is realistic and shows up in interviews.

```sql
-- Create these tables with proper data types, constraints, and foreign keys:

-- 1. restaurants (id, name, cuisine_type, city, rating, is_active, created_at)
-- 2. menu_items (id, restaurant_id FK, name, description, price, category, is_available)
-- 3. customers (id, name, email UNIQUE, phone, address, city, joined_date)
-- 4. orders (id, customer_id FK, restaurant_id FK, order_date, total_amount, status, delivery_address)
-- 5. order_details (id, order_id FK, menu_item_id FK, quantity, unit_price, subtotal)

-- Requirements:
-- - All IDs should auto-increment
-- - Use NOT NULL where appropriate
-- - Prices must be positive (CHECK constraint)
-- - Status should have a DEFAULT of 'pending'
-- - All tables should have created_at with DEFAULT NOW()
-- - Add at least 3 restaurants, 10 menu items, 5 customers, 8 orders

-- After creating, write these queries:
-- a) Top 3 restaurants by number of orders
-- b) Most popular menu item overall (by total quantity ordered)
-- c) Customer who spent the most money
-- d) Average order value per restaurant
-- e) Revenue by cuisine type
-- f) Month-over-month order growth
```

<details>
<summary>📖 Full Schema + Answers</summary>

```sql
-- Schema
CREATE TABLE restaurants (
    restaurant_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    cuisine_type VARCHAR(50),
    city VARCHAR(50) NOT NULL,
    rating DECIMAL(3,2) CHECK (rating >= 0 AND rating <= 5),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE menu_items (
    item_id SERIAL PRIMARY KEY,
    restaurant_id INTEGER NOT NULL REFERENCES restaurants(restaurant_id),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price DECIMAL(8,2) NOT NULL CHECK (price > 0),
    category VARCHAR(50),
    is_available BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    address TEXT,
    city VARCHAR(50),
    joined_date DATE DEFAULT CURRENT_DATE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(customer_id),
    restaurant_id INTEGER NOT NULL REFERENCES restaurants(restaurant_id),
    order_date TIMESTAMPTZ DEFAULT NOW(),
    total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
    status VARCHAR(20) DEFAULT 'pending',
    delivery_address TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_details (
    detail_id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(order_id),
    item_id INTEGER NOT NULL REFERENCES menu_items(item_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(8,2) NOT NULL,
    subtotal DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);
-- GENERATED ALWAYS AS ... STORED = computed column, auto-calculates!

-- Sample data (insert yourself for practice)
INSERT INTO restaurants (name, cuisine_type, city, rating) VALUES
('Chicken Rice King', 'Chinese', 'Singapore', 4.50),
('Nasi Lemak House', 'Malay', 'Kuala Lumpur', 4.30),
('Roti Prata Corner', 'Indian', 'Singapore', 4.10);

INSERT INTO menu_items (restaurant_id, name, price, category) VALUES
(1, 'Hainanese Chicken Rice', 6.50, 'Rice'),
(1, 'Roast Chicken Rice', 7.00, 'Rice'),
(1, 'Chicken Soup', 5.00, 'Soup'),
(2, 'Nasi Lemak Special', 8.50, 'Rice'),
(2, 'Nasi Lemak Ayam Goreng', 9.00, 'Rice'),
(2, 'Sambal Squid', 7.50, 'Seafood'),
(3, 'Prata Plain', 1.50, 'Bread'),
(3, 'Prata Egg', 2.50, 'Bread'),
(3, 'Prata Cheese', 3.50, 'Bread'),
(3, 'Mutton Murtabak', 6.00, 'Bread');

INSERT INTO customers (name, email, phone, city) VALUES
('Ahmad', 'ahmad@email.com', '0123456789', 'Kuala Lumpur'),
('Sarah', 'sarah@email.com', '9876543210', 'Singapore'),
('Wei Ming', 'weiming@email.com', '8765432109', 'Singapore'),
('Priya', 'priya@email.com', '7654321098', 'Singapore'),
('Hassan', 'hassan@email.com', '6543210987', 'Kuala Lumpur');

-- Insert orders and order_details yourself for more practice!

-- Queries:
-- a)
SELECT r.name, COUNT(o.order_id) AS order_count
FROM restaurants r
LEFT JOIN orders o ON r.restaurant_id = o.restaurant_id
GROUP BY r.restaurant_id, r.name
ORDER BY order_count DESC LIMIT 3;

-- b) 
SELECT mi.name, SUM(od.quantity) AS total_qty
FROM menu_items mi
INNER JOIN order_details od ON mi.item_id = od.item_id
GROUP BY mi.name
ORDER BY total_qty DESC LIMIT 1;

-- c)
SELECT c.name, SUM(o.total_amount) AS total_spent
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name
ORDER BY total_spent DESC LIMIT 1;

-- d)
SELECT r.name, ROUND(AVG(o.total_amount), 2) AS avg_order
FROM restaurants r
INNER JOIN orders o ON r.restaurant_id = o.restaurant_id
GROUP BY r.restaurant_id, r.name;

-- e)
SELECT r.cuisine_type, SUM(o.total_amount) AS revenue
FROM restaurants r
INNER JOIN orders o ON r.restaurant_id = o.restaurant_id
GROUP BY r.cuisine_type;

-- f)
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date) AS month,
           SUM(total_amount) AS revenue
    FROM orders
    GROUP BY month
)
SELECT month, revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month,
    ROUND((revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 
          / LAG(revenue) OVER (ORDER BY month), 2) AS growth_pct
FROM monthly;
```
</details>

---

### Task 12: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 5 — SQL Advanced
Date: 2026-05-19

## New Skills Today
- UNION / UNION ALL → 
- INTERSECT / EXCEPT → 
- DDL (CREATE, ALTER, DROP) → 
- Data types (INTEGER, VARCHAR, DECIMAL, TIMESTAMPTZ, JSONB) → 
- Indexes → 
- EXPLAIN / EXPLAIN ANALYZE → 
- String functions (UPPER, LOWER, SPLIT_PART, CONCAT, LIKE) → 
- Date functions (EXTRACT, AGE, DATE_TRUNC, TO_CHAR, INTERVAL) → 

## Food Delivery Schema
- Designed X tables with proper constraints
- Wrote X business queries

## What I Found Interesting
- 

## Tomorrow: SQL Review + Assessment
```

```bash
git add .
git commit -m "Day 5: UNION, DDL, data types, indexes, string/date functions, food delivery schema"
git push
```

---

## ✅ Day 5 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Used UNION, INTERSECT, EXCEPT | ☐ |
| 2 | Completed SQL execution order quiz | ☐ |
| 3 | Can name 8+ PostgreSQL data types and when to use them | ☐ |
| 4 | Created tables with constraints (NOT NULL, CHECK, DEFAULT, FK) | ☐ |
| 5 | Understand indexes and when to create them | ☐ |
| 6 | Used EXPLAIN to analyze a query | ☐ |
| 7 | Used 5+ string functions | ☐ |
| 8 | Used 5+ date functions | ☐ |
| 9 | Solved 5 HackerRank problems | ☐ |
| 10 | Built food delivery database schema + queries | ☐ |
| 11 | Day 5 notes pushed to GitHub | ☐ |
