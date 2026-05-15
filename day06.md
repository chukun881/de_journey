# 📅 Day 6 — Tuesday, 20 May 2026
# SQL Assessment Day: Timed Test + Weak Spot Review

---

## 🎯 Today's Big Picture
This is NOT a learning day. This is a TEST day. You've learned 5 days of SQL — now prove it.

By end of today:
- Complete a timed SQL assessment (simulates interview conditions)
- Identify your weak spots
- Review and fix gaps
- Build a personal SQL cheat sheet for your portfolio

**Why this matters:** Data engineer interviews almost always include a live SQL test. You'll get 30-60 minutes to solve problems on a whiteboard or shared screen. Today simulates that pressure.

---

## ☀️ BLOCK 1: Timed Assessment (Morning, ~2 hours)

---

### ⏱️ THE RULES
- **No looking at notes, no Googling, no AI help** for the assessment
- **Time yourself** — write start and end time
- **Write queries in a file** called `sql-practice/day06-assessment.md`
- **Mark questions you couldn't solve** — we'll review them
- **Don't skip questions** — attempt everything, even if unsure

---

### 📝 Assessment (90 minutes max)

Use the food delivery database you created yesterday. If you didn't finish the schema, recreate it first (that counts as review!).

**Easy (5 min each, 25 min total):**

E1. Show all customers who live in Singapore
E2. Show all menu items with price above 5.00, sorted by price descending
E3. Count how many restaurants are in each city
E4. Show the 3 cheapest menu items
E5. Find all orders with status 'completed' placed in January 2024

**Medium (7 min each, 35 min total):**

M1. Show each restaurant's name and the average price of its menu items
M2. Find customers who have placed more than 2 orders
M3. Show each order's details: order_id, customer name, restaurant name, total_amount, and how many different items were ordered
M4. Find the most expensive item on each restaurant's menu
M5. Show restaurants that have NO orders yet

**Hard (10 min each, 30 min total):**

H1. For each customer, show their name, total spent, number of orders, average order value, and their favorite restaurant (most orders)
H2. Calculate month-over-month revenue growth rate for each restaurant. Show restaurant name, month, revenue, previous month revenue, and growth percentage
H3. Find the top 3 customers by total spending, and for each one show their most ordered item and how many times they ordered it

---

### 🔍 Self-Grading (15 min)

After finishing (or when time's up), check each answer:

| Question | Solved? | Time taken | Struggled? |
|----------|---------|------------|------------|
| E1 | ☐ | | |
| E2 | ☐ | | |
| E3 | ☐ | | |
| E4 | ☐ | | |
| E5 | ☐ | | |
| M1 | ☐ | | |
| M2 | ☐ | | |
| M3 | ☐ | | |
| M4 | ☐ | | |
| M5 | ☐ | | |
| H1 | ☐ | | |
| H2 | ☐ | | |
| H3 | ☐ | | |

---

## 🔥 BLOCK 2: Review & Fix Weak Spots (Afternoon, ~2 hours)

---

### Task 1: Check Your Answers (30 min)

Compare your solutions with these reference answers. **If your answer is different but works, that's fine!** SQL has many ways to solve the same problem.

<details>
<summary>📖 Assessment Answers</summary>

```sql
-- E1
SELECT * FROM customers WHERE city = 'Singapore';

-- E2
SELECT name, price FROM menu_items WHERE price > 5.00 ORDER BY price DESC;

-- E3
SELECT city, COUNT(*) AS restaurant_count FROM restaurants GROUP BY city;

-- E4
SELECT name, price FROM menu_items ORDER BY price ASC LIMIT 3;

-- E5
SELECT * FROM orders 
WHERE status = 'completed' 
  AND order_date >= '2024-01-01' 
  AND order_date < '2024-02-01';

-- M1
SELECT r.name AS restaurant, ROUND(AVG(mi.price), 2) AS avg_price
FROM restaurants r
INNER JOIN menu_items mi ON r.restaurant_id = mi.restaurant_id
GROUP BY r.restaurant_id, r.name;

-- M2
SELECT c.name, COUNT(o.order_id) AS order_count
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name
HAVING COUNT(o.order_id) > 2;

-- M3
SELECT o.order_id, c.name AS customer, r.name AS restaurant, 
       o.total_amount, COUNT(od.detail_id) AS item_count
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
INNER JOIN restaurants r ON o.restaurant_id = r.restaurant_id
INNER JOIN order_details od ON o.order_id = od.order_id
GROUP BY o.order_id, c.name, r.name, o.total_amount;

-- M4
WITH ranked AS (
    SELECT mi.name, mi.price, r.name AS restaurant,
        ROW_NUMBER() OVER (PARTITION BY r.restaurant_id ORDER BY mi.price DESC) AS rn
    FROM menu_items mi
    INNER JOIN restaurants r ON mi.restaurant_id = r.restaurant_id
)
SELECT restaurant, name AS most_expensive_item, price
FROM ranked WHERE rn = 1;

-- M5
SELECT r.name
FROM restaurants r
LEFT JOIN orders o ON r.restaurant_id = o.restaurant_id
WHERE o.order_id IS NULL;

-- H1
WITH customer_stats AS (
    SELECT c.customer_id, c.name,
           SUM(o.total_amount) AS total_spent,
           COUNT(o.order_id) AS order_count,
           ROUND(AVG(o.total_amount), 2) AS avg_order_value
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_id, c.name
),
favorite AS (
    SELECT o.customer_id, r.name AS favorite_restaurant,
        COUNT(*) AS orders_at_restaurant,
        ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY COUNT(*) DESC) AS rn
    FROM orders o
    INNER JOIN restaurants r ON o.restaurant_id = r.restaurant_id
    GROUP BY o.customer_id, r.restaurant_id, r.name
)
SELECT cs.name, cs.total_spent, cs.order_count, cs.avg_order_value,
       f.favorite_restaurant
FROM customer_stats cs
LEFT JOIN favorite f ON cs.customer_id = f.customer_id AND f.rn = 1;

-- H2
WITH monthly AS (
    SELECT r.name AS restaurant,
           DATE_TRUNC('month', o.order_date) AS month,
           SUM(o.total_amount) AS revenue
    FROM restaurants r
    INNER JOIN orders o ON r.restaurant_id = o.restaurant_id
    GROUP BY r.restaurant_id, r.name, DATE_TRUNC('month', o.order_date)
)
SELECT restaurant, month, revenue,
    LAG(revenue) OVER (PARTITION BY restaurant ORDER BY month) AS prev_month,
    ROUND(
        (revenue - LAG(revenue) OVER (PARTITION BY restaurant ORDER BY month)) * 100.0
        / NULLIF(LAG(revenue) OVER (PARTITION BY restaurant ORDER BY month), 0), 2
    ) AS growth_pct
FROM monthly
ORDER BY restaurant, month;

-- H3
WITH spending AS (
    SELECT c.customer_id, c.name, SUM(o.total_amount) AS total_spent,
        DENSE_RANK() OVER (ORDER BY SUM(o.total_amount) DESC) AS spend_rank
    FROM customers c
    INNER JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_id, c.name
),
top_items AS (
    SELECT o.customer_id, mi.name AS item_name,
        SUM(od.quantity) AS total_qty,
        ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY SUM(od.quantity) DESC) AS rn
    FROM orders o
    INNER JOIN order_details od ON o.order_id = od.order_id
    INNER JOIN menu_items mi ON od.item_id = mi.item_id
    GROUP BY o.customer_id, mi.name
)
SELECT s.name, s.total_spent, t.item_name AS most_ordered, t.total_qty
FROM spending s
INNER JOIN top_items t ON s.customer_id = t.customer_id AND t.rn = 1
WHERE s.spend_rank <= 3;
```
</details>

---

### Task 2: Targeted Review (45 min)

Based on which questions you struggled with, review the specific topic:

| Struggled with | Review from |
|---------------|-------------|
| E1-E5 (basics) | Day 1 notes — SELECT, WHERE, ORDER BY, GROUP BY |
| M1 (JOIN + aggregate) | Day 2 — JOINs + Day 1 — GROUP BY |
| M2 (HAVING) | Day 3 — HAVING section |
| M3 (multi-table JOIN) | Day 2 — 3+ table JOINs |
| M4 (top N per group) | Day 4 — ROW_NUMBER pattern |
| M5 (LEFT JOIN for missing) | Day 2 — LEFT JOIN |
| H1 (complex CTE) | Day 3 — CTEs |
| H2 (LAG window function) | Day 4 — LAG |
| H3 (nested CTE + window) | Day 3 + Day 4 combined |

For each weak area:
1. Re-read the relevant day's notes
2. Re-do 2-3 exercises from that day
3. Try the assessment question again from scratch

---

### Task 3: HackerRank SQL Advanced Badge Challenge (45 min)

Go to https://www.hackerrank.com/domains/sql

Try to solve **5 problems** from the "Advanced" or "Aggregation" section. Pick ones that challenge your weak spots.

Suggested:
1. **The Blunder** — string + math
2. **Top Earners** — aggregate + max
3. **Weather Observation Station 13** — SUM with conditions
4. **Weather Observation Station 14** — rounding
5. **Population Census** — JOIN + SUM

---

## 🌙 BLOCK 3: Build Your SQL Cheat Sheet (Evening, ~1 hour)

---

### Task 4: Personal SQL Reference Card (45 min)

Create `sql-practice/sql-cheat-sheet.md` — this goes in your GitHub portfolio.

Write it IN YOUR OWN WORDS. Don't copy-paste from my notes. If you can explain it, you understand it.

```markdown
# SQL Reference Card

## Basics
SELECT, WHERE, ORDER BY, LIMIT, DISTINCT

## Aggregates
COUNT, SUM, AVG, MIN, MAX, GROUP BY, HAVING

## JOINs
INNER JOIN, LEFT JOIN, RIGHT JOIN, FULL OUTER JOIN

## Subqueries & CTEs
(Subquery types, WITH clause)

## Window Functions
(ROW_NUMBER, RANK, LAG, LEAD, SUM OVER)

## String Functions
(UPPER, LOWER, SPLIT_PART, CONCAT, LIKE, LENGTH)

## Date Functions
(EXTRACT, AGE, DATE_TRUNC, TO_CHAR, INTERVAL)

## DDL
(CREATE TABLE, data types, constraints, indexes)

## Performance
(EXPLAIN, index strategy)
```

Fill in each section with 2-3 example queries that YOU wrote (not copied). This is proof you understand.

---

### Task 5: Reflection (15 min)

```markdown
# Day 6 — SQL Assessment
Date: 2026-05-20

## Assessment Score: X/13

## My Weak Spots
1. 
2. 
3. 

## What I Reviewed
- 

## Tomorrow: Portfolio Project Day!
```

```bash
git add .
git commit -m "Day 6: SQL assessment + cheat sheet"
git push
```

---

## ✅ Day 6 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Completed timed assessment (13 questions) | ☐ |
| 2 | Self-graded all answers | ☐ |
| 3 | Reviewed weak spots with targeted exercises | ☐ |
| 4 | Solved 5 HackerRank advanced problems | ☐ |
| 5 | Built personal SQL cheat sheet | ☐ |
| 6 | Day 6 notes pushed to GitHub | ☐ |
