# 📅 Day 7 — Wednesday, 21 May 2026
# SQL Portfolio Project: End-to-End Data Analysis

---

## 🎯 Today's Big Picture
Today you build your **first portfolio project** — a complete SQL data analysis from scratch.

This is NOT an exercise. This is a real project that goes on your GitHub and your resume.

**What you'll build:** An analysis of Singapore/Malaysia retail sales data
- Create the database schema
- Generate realistic sample data
- Write 10+ business-critical queries
- Create a summary report
- Push to GitHub as a standalone project

**Why this matters:** When interviewers ask "show me your portfolio," this is what you show. A repo with a well-structured database, clean SQL queries, and a written analysis. This proves you can do real work, not just tutorials.

---

## ☀️ BLOCK 1: Project Setup (Morning, ~1 hour)

---

### Task 1: Create Your Project Repository (15 min)

This is a SEPARATE repo from your learning notes. This is a portfolio piece.

```bash
# Create a new GitHub repo called "retail-sales-analysis"
mkdir retail-sales-analysis
cd retail-sales-analysis
git init

# Create project structure
mkdir -p sql/schema sql/queries sql/analysis docs
touch README.md
touch sql/schema/01_create_tables.sql
touch sql/schema/02_seed_data.sql
touch sql/queries/basic_queries.sql
touch sql/queries/advanced_queries.sql
touch sql/analysis/business_report.sql
touch docs/data_dictionary.md
```

---

### Task 2: Design the Schema (30 min)

You're designing the database for a retail chain with stores across Singapore and Malaysia. Think about what tables you need.

**Business context:** A retail company with 5 stores across SG/MY. They sell electronics, clothing, and food. They want to understand sales patterns.

**Tables to create in `sql/schema/01_create_tables.sql`:**

```sql
-- Design these tables yourself first, then check the reference

-- 1. stores (id, name, city, country, opening_date, square_meters)
-- 2. products (id, name, category, subcategory, unit_price, cost_price, is_active)
-- 3. customers (id, name, email, phone, city, country, joined_date, loyalty_tier)
-- 4. sales (id, store_id FK, customer_id FK, sale_date, total_amount, payment_method, status)
-- 5. sale_items (id, sale_id FK, product_id FK, quantity, unit_price, discount_pct, line_total)
```

**Requirements:**
- Proper data types (use what you learned Day 5)
- Foreign keys for all relationships
- CHECK constraints: prices > 0, discount between 0-100, quantity > 0
- DEFAULT values where appropriate
- At least 5 stores, 20 products, 30 customers, 100+ sales

<details>
<summary>📖 Reference Schema</summary>

```sql
-- sql/schema/01_create_tables.sql

DROP TABLE IF EXISTS sale_items CASCADE;
DROP TABLE IF EXISTS sales CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS stores CASCADE;

CREATE TABLE stores (
    store_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    city VARCHAR(50) NOT NULL,
    country VARCHAR(50) NOT NULL,
    opening_date DATE NOT NULL,
    square_meters INTEGER CHECK (square_meters > 0),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    category VARCHAR(50) NOT NULL,
    subcategory VARCHAR(50),
    unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price > 0),
    cost_price DECIMAL(10,2) NOT NULL CHECK (cost_price > 0),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    city VARCHAR(50) NOT NULL,
    country VARCHAR(50) NOT NULL,
    joined_date DATE NOT NULL,
    loyalty_tier VARCHAR(20) DEFAULT 'Bronze',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,
    store_id INTEGER NOT NULL REFERENCES stores(store_id),
    customer_id INTEGER NOT NULL REFERENCES customers(customer_id),
    sale_date TIMESTAMPTZ NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL CHECK (total_amount >= 0),
    payment_method VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'completed',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sale_items (
    item_id SERIAL PRIMARY KEY,
    sale_id INTEGER NOT NULL REFERENCES sales(sale_id),
    product_id INTEGER NOT NULL REFERENCES products(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price > 0),
    discount_pct DECIMAL(5,2) DEFAULT 0 CHECK (discount_pct >= 0 AND discount_pct <= 100),
    line_total DECIMAL(12,2) GENERATED ALWAYS AS (quantity * unit_price * (1 - discount_pct/100)) STORED
);
```
</details>

---

### Task 3: Generate Sample Data (45 min)

Write INSERT statements in `sql/schema/02_seed_data.sql`. Make the data realistic — use real Singapore/Malaysia context.

**Stores:**
- Orchard Road, Singapore
- Jurong Point, Singapore
- Pavilion KL, Kuala Lumpur
- Mid Valley, Kuala Lumpur
- Gurney Plaza, Penang

**Products (20 items across 3 categories):**
- Electronics: laptops, phones, headphones, chargers, tablets
- Clothing: t-shirts, jeans, dresses, jackets, sneakers
- Food: coffee beans, tea sets, chocolate, cookies, honey, protein bars

**Customers (30 people):** Mix of Singapore and Malaysian names, cities, loyalty tiers

**Sales (100+):** Spread across 2024, realistic patterns (more sales on weekends, holiday spikes, etc.)

<details>
<summary>📖 Reference Seed Data (use as guide, write your own version)</summary>

```sql
-- Stores
INSERT INTO stores (name, city, country, opening_date, square_meters) VALUES
('Orchard Electronics Hub', 'Singapore', 'Singapore', '2020-01-15', 500),
('Jurong Lifestyle Store', 'Singapore', 'Singapore', '2021-06-01', 350),
('Pavilion Premium Outlet', 'Kuala Lumpur', 'Malaysia', '2019-09-20', 600),
('Mid Valley Megastore', 'Kuala Lumpur', 'Malaysia', '2018-03-10', 750),
('Gurney Bay Store', 'Penang', 'Malaysia', '2022-11-01', 280);

-- Products
INSERT INTO products (name, category, subcategory, unit_price, cost_price) VALUES
-- Electronics
('MacBook Air M3', 'Electronics', 'Laptops', 1499.00, 1200.00),
('iPhone 15', 'Electronics', 'Phones', 999.00, 750.00),
('AirPods Pro', 'Electronics', 'Audio', 249.00, 180.00),
('USB-C Fast Charger', 'Electronics', 'Accessories', 39.90, 15.00),
('iPad Mini', 'Electronics', 'Tablets', 599.00, 420.00),
('Samsung Galaxy S24', 'Electronics', 'Phones', 899.00, 650.00),
('Logitech MX Master', 'Electronics', 'Accessories', 129.00, 70.00),
-- Clothing
('Cotton Crew Tee', 'Clothing', 'T-Shirts', 29.90, 8.00),
('Slim Fit Jeans', 'Clothing', 'Jeans', 59.90, 18.00),
('Summer Maxi Dress', 'Clothing', 'Dresses', 79.90, 25.00),
('Lightweight Jacket', 'Clothing', 'Jackets', 119.00, 40.00),
('Running Sneakers', 'Clothing', 'Shoes', 149.00, 55.00),
-- Food
('Kopi Luwak Beans 250g', 'Food', 'Coffee', 45.00, 15.00),
('Chinese Tea Gift Set', 'Food', 'Tea', 68.00, 22.00),
('Belgian Chocolate Box', 'Food', 'Chocolate', 35.00, 12.00),
('Artisan Cookies Tin', 'Food', 'Snacks', 28.00, 8.00),
('Manuka Honey 500g', 'Food', 'Health', 89.00, 35.00),
('Protein Bar Pack (12)', 'Food', 'Health', 42.00, 14.00),
('Local Kaya Spread', 'Food', 'Spreads', 15.00, 4.50),
('Durian Mooncake Set', 'Food', 'Festive', 55.00, 18.00);

-- Generate customers yourself! (30 customers, mix of SG/MY)
-- Generate sales yourself! (100+ sales over 2024)
-- Generate sale_items yourself! (link sales to products)

-- TIP: For generating lots of sales, you can use generate_series:
-- INSERT INTO sales (store_id, customer_id, sale_date, total_amount, payment_method)
-- SELECT ... this is an advanced technique, feel free to just write them out
```
</details>

> 💡 **Tip:** Writing 100 INSERT statements manually is tedious but educational — it forces you to think about realistic data patterns. In Week 2, you'll learn Python to generate data automatically!

---

## 🔥 BLOCK 2: Business Analysis Queries (Afternoon, ~2.5 hours)

---

### Task 4: Basic Queries (45 min)

Write these in `sql/queries/basic_queries.sql`:

```sql
-- Q1: Total revenue by store
-- Q2: Total revenue by month (for 2024)
-- Q3: Top 10 best-selling products by quantity
-- Q4: Top 10 products by revenue
-- Q5: Revenue by category
-- Q6: Average order value by payment method
-- Q7: Sales count by day of week
-- Q8: Revenue by country (Singapore vs Malaysia)
```

### Task 5: Advanced Queries (1 hour)

Write these in `sql/queries/advanced_queries.sql`:

```sql
-- Q9: Month-over-month revenue growth rate (use LAG)
-- Q10: Top product in each category (use ROW_NUMBER)
-- Q11: Customer segmentation: classify customers as VIP (spent > 500), 
--       Regular (100-500), New (< 100) using CASE WHEN
-- Q12: Store performance comparison: for each store, show revenue, 
--       unique customers, avg order value, and rank stores by revenue
-- Q13: Find customers who made purchases in both Singapore AND Malaysia
-- Q14: Products that sell better on weekends vs weekdays
-- Q15: Customer retention: what % of customers made a repeat purchase 
--       within 30 days of their first purchase?
```

### Task 6: Executive Summary (30 min)

Write in `sql/analysis/business_report.sql` — a single comprehensive query (or CTE chain) that produces an executive dashboard:

```
Store | Month | Revenue | Orders | Unique Customers | Avg Order Value | Top Category | YoY Growth
```

---

## 🌙 BLOCK 3: Documentation + Polish (Evening, ~1.5 hours)

---

### Task 7: Write README.md (30 min)

This is IMPORTANT. Your README is what recruiters and interviewers see first.

```markdown
# Retail Sales Analysis 📊

SQL analysis of a multi-store retail chain across Singapore and Malaysia.

## Overview
Analysis of retail sales data covering 5 stores, 20 products, and 30 customers 
over the period of 2024. Built with PostgreSQL.

## Database Schema
- **stores** — 5 retail locations across SG/MY
- **products** — 20 items across Electronics, Clothing, Food categories
- **customers** — 30 customers with loyalty tiers
- **sales** — 100+ transactions in 2024
- **sale_items** — individual line items per sale

See [data dictionary](docs/data_dictionary.md) for full schema details.

## Key Findings
(Write 3-5 bullet points from your analysis)
- Top performing store: ...
- Best selling category: ...
- Peak sales period: ...
- Customer insights: ...

## Queries
| File | Description |
|------|-------------|
| [basic_queries.sql](sql/queries/basic_queries.sql) | Revenue, top products, monthly trends |
| [advanced_queries.sql](sql/queries/advanced_queries.sql) | MoM growth, customer segmentation, retention |
| [business_report.sql](sql/analysis/business_report.sql) | Executive dashboard query |

## Tech Stack
- PostgreSQL 16
- SQL (CTEs, Window Functions, JOINs, Aggregations)

## How to Run
1. Clone this repo
2. Run `sql/schema/01_create_tables.sql` to create tables
3. Run `sql/schema/02_seed_data.sql` to load data
4. Run any query from `sql/queries/`

## What I Learned
- Database schema design with proper constraints
- Complex SQL queries using CTEs and window functions
- Data analysis patterns for retail business
```

---

### Task 8: Data Dictionary (15 min)

Write `docs/data_dictionary.md`:

```markdown
# Data Dictionary

## stores
| Column | Type | Description |
|--------|------|-------------|
| store_id | SERIAL PK | Unique store identifier |
| name | VARCHAR(100) | Store name |
| ... | ... | ... |

## products
(fill in)

## customers
(fill in)

## sales
(fill in)

## sale_items
(fill in)
```

---

### Task 9: Final Push + Week 1 Reflection (15 min)

```bash
git add .
git commit -m "Initial commit: retail sales analysis with PostgreSQL"
git push
```

Then in your learning repo (`data-engineering-journey`):

```markdown
# Day 7 — Portfolio Project Day
Date: 2026-05-21

## Project Created
- Repo: retail-sales-analysis
- Tables: 5
- Queries: 15+
- Key finding: 

## Week 1 Summary
- Days completed: 7/7
- Total queries written: 200+
- Topics mastered: SELECT, JOINs, subqueries, CTEs, window functions, DDL, performance
- Portfolio project: ✅ 
- Cheat sheet: ✅

## Week 2 Preview: Python for Data Engineering
```

```bash
cd data-engineering-journey
git add .
git commit -m "Day 7: Portfolio project complete. Week 1 SQL done!"
git push
```

---

## ✅ Day 7 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Created separate GitHub repo for portfolio | ☐ |
| 2 | Designed schema with 5 tables + constraints | ☐ |
| 3 | Generated realistic sample data (100+ sales) | ☐ |
| 4 | Wrote 8 basic business queries | ☐ |
| 5 | Wrote 7 advanced queries (window functions, CTEs) | ☐ |
| 6 | Created executive summary query | ☐ |
| 7 | Wrote professional README.md | ☐ |
| 8 | Wrote data dictionary | ☐ |
| 9 | Pushed to GitHub | ☐ |
| 10 | Completed Week 1 reflection | ☐ |

---

## 🏁 WEEK 1 CHECKPOINT — Are We On Track?

### Skills gained this week vs. job requirements:

| Job Requirement (from real SG/MY postings) | Status | Evidence |
|---------------------------------------------|--------|----------|
| SQL proficiency (complex queries, JOINs) | ✅ Done | 200+ queries, portfolio project |
| CTEs and window functions | ✅ Done | Used in portfolio advanced queries |
| Database design (schema, data types, constraints) | ✅ Done | Built 3 databases from scratch |
| Query optimization awareness | ✅ Introduced | EXPLAIN, indexes |
| Python | 🔜 Week 2 | — |
| Pandas (data manipulation) | 🔜 Week 2 | — |
| ETL pipeline building | 🔜 Week 2-3 | — |
| dbt | 🔜 Week 3 | — |
| Cloud (AWS) | 🔜 Week 3-4 | — |
| Git/GitHub | ✅ Ongoing | Daily commits, 2 repos |
| Data modeling | ✅ Introduced | Star schema in food delivery project |
| Portfolio projects | ✅ 1 done | retail-sales-analysis |

### Week 2 Preview:
- **Day 8-9:** Python basics (variables, loops, functions, file I/O)
- **Day 10-11:** Python Pandas (data cleaning, transformation)
- **Day 12:** Python + SQL (connect to PostgreSQL with Python)
- **Day 13:** Python + APIs (fetch data from public APIs)
- **Day 14:** Portfolio Project 2 — automated data pipeline

**We are ON TRACK.** SQL is the foundation, and you've built it solid. Week 2 adds Python, which combined with SQL gives you the core skillset that every SG/MY job posting asks for.
