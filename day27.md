# 📅 Day 27 — Tuesday, 10 June 2026
# 📝 Assessment Day: Week 4 Review & Weak Spot Practice

---

## 🎯 Today's Goal

No new concepts today. This is **test day** — a chance to check what stuck and what didn't. You'll do timed exercises covering all of Week 4. Be honest with yourself about weak spots.

---

## ☀️ Morning Block (2 hours): Timed Assessment

### Rules
- Set a timer for each section
- Don't look at notes until AFTER attempting
- Grade yourself honestly
- Target: 70%+ correct on first attempt

---

### Section A: Complex Joins (20 minutes, 5 questions)

**Schema:** `customers(id, name, city, signup_date)`, `orders(id, customer_id, restaurant_id, date, amount, status)`, `restaurants(id, name, cuisine, city, rating)`, `order_items(id, order_id, dish, qty, price)`

**Q1.** Write a query to find all restaurants that have NEVER received an order from a customer in the same city. Show restaurant name, city, and rating.

**Q2.** Find customers who have ordered from at least 3 different cuisine types. Show customer name and the count of distinct cuisines.

**Q3.** Find pairs of customers who have ordered the exact same dish on the same day. Show both customer names, the dish, and the date.

**Q4.** For each restaurant, find the customer who has placed the most orders. Handle ties by picking the one with the highest total spending. Show restaurant name, customer name, order count, and total spent.

**Q5.** Write a self-join query to find restaurants whose rating is higher than ALL other restaurants in the same city. (Not just higher than average — higher than every single one.)

---

### Section B: Query Optimization (20 minutes, 5 questions)

**Q6.** This query is slow. Rewrite it to be faster:
```sql
SELECT c.name, 
    (SELECT SUM(amount) FROM orders WHERE customer_id = c.id) AS total,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) AS cnt
FROM customers c
ORDER BY total DESC;
```

**Q7.** You run EXPLAIN and see `Seq Scan on orders (cost=0.00..50000.00 rows=10000000)`. The query is `SELECT * FROM orders WHERE customer_id = 123`. What index would fix this?

**Q8.** A query shows `Sort Method: external merge Disk: 5000kB`. What two things can you do to fix this?

**Q9.** You see `Nested Loop (rows=500000)` where the inner side is an Index Scan. Why is this potentially slow? What would be better for large datasets?

**Q10.** Write a keyset pagination query for `orders` ordered by `order_date DESC, order_id DESC`. Show how to get page 1 (first 50 rows) and page 2 (next 50).

---

### Section C: Star Schema Design (20 minutes, 5 questions)

**Q11.** Design a star schema for a **bus ticket booking system** in Singapore. List all dimension and fact tables with key columns. State the grain of the fact table.

**Q12.** Explain the difference between a surrogate key and a natural key. Give one advantage of each.

**Q13.** What is a conformed dimension? Give an example using a food delivery AND a ride-hailing service.

**Q14.** A stakeholder wants to track daily revenue by restaurant. Another wants individual order details. Do you create one fact table or two? Explain your reasoning.

**Q15.** What is a degenerate dimension? Give an example and explain when you'd use one.

---

### Section D: SCD (15 minutes, 4 questions)

**Q16.** Explain the difference between SCD Type 1, 2, and 3 in one sentence each.

**Q17.** Write the ETL SQL for SCD Type 2 on `dim_customer` tracking changes to `city` and `segment`. Include both the expire and insert steps.

**Q18.** A fact table load joins to `dim_customer` (SCD Type 2). Write the JOIN condition that ensures the correct surrogate key is used for historical orders.

**Q19.** When would you use SCD Type 1 for `email` but Type 2 for `city` in the same dimension? How does the ETL handle this?

---

### Section E: Window Functions (25 minutes, 6 questions)

**Q20.** Write a query to calculate the **running total of revenue per restaurant**, ordered by date.

**Q21.** Calculate **month-over-month revenue growth** for the entire platform. Show month, revenue, previous month revenue, and growth %.

**Q22.** Find the **top 3 customers per city** by total spending using a window function.

**Q23.** Calculate a **7-day rolling average** of daily revenue.

**Q24.** For each customer, find their **longest streak of consecutive ordering days**. (Gap & island problem.)

**Q25.** Perform **RFM segmentation**: score each customer 1-5 on Recency, Frequency, and Monetary using NTILE. Classify as 'Champions', 'Loyal', 'At Risk', or 'Lost'.

---

## 🌤️ Afternoon Block (2 hours): Answers & Weak Spot Review

### Scoring Guide

| Section | Questions | Points Each | Total |
|---------|-----------|-------------|-------|
| A: Joins | 5 | 4 | 20 |
| B: Optimization | 5 | 4 | 20 |
| C: Star Schema | 5 | 4 | 20 |
| D: SCD | 4 | 5 | 20 |
| E: Window Functions | 6 | ~3.3 | 20 |
| **Total** | **25** | | **100** |

**Scoring:**
- 90-100: Excellent — you're ready for Week 5
- 70-89: Good — review weak spots tonight
- 50-69: Fair — spend extra time on weak areas
- Below 50: Review the full week's material

---

### ✅ Full Answers

<details>
<summary>🔑 Section A: Complex Joins</summary>

**A1:**
```sql
SELECT r.name, r.city, r.rating
FROM restaurants r
WHERE NOT EXISTS (
    SELECT 1 FROM orders o
    JOIN customers c ON o.customer_id = c.id
    WHERE o.restaurant_id = r.id AND c.city = r.city
);
```

**A2:**
```sql
SELECT c.name, COUNT(DISTINCT r.cuisine) AS cuisine_count
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN restaurants r ON o.restaurant_id = r.id
GROUP BY c.id, c.name
HAVING COUNT(DISTINCT r.cuisine) >= 3
ORDER BY cuisine_count DESC;
```

**A3:**
```sql
SELECT 
    c1.name AS customer_a,
    c2.name AS customer_b,
    oi1.dish,
    DATE(o1.date) AS order_date
FROM order_items oi1
JOIN orders o1 ON oi1.order_id = o1.id
JOIN customers c1 ON o1.customer_id = c1.id
JOIN order_items oi2 ON oi1.dish = oi2.dish AND oi1.id < oi2.id
JOIN orders o2 ON oi2.order_id = o2.id AND DATE(o1.date) = DATE(o2.date)
JOIN customers c2 ON o2.customer_id = c2.id AND c1.id < c2.id;
```

**A4:**
```sql
WITH customer_stats AS (
    SELECT 
        o.restaurant_id,
        o.customer_id,
        COUNT(*) AS order_count,
        SUM(o.amount) AS total_spent,
        ROW_NUMBER() OVER (
            PARTITION BY o.restaurant_id 
            ORDER BY COUNT(*) DESC, SUM(o.amount) DESC
        ) AS rn
    FROM orders o
    WHERE o.status = 'completed'
    GROUP BY o.restaurant_id, o.customer_id
)
SELECT r.name AS restaurant, c.name AS top_customer, cs.order_count, cs.total_spent
FROM customer_stats cs
JOIN restaurants r ON cs.restaurant_id = r.id
JOIN customers c ON cs.customer_id = c.id
WHERE cs.rn = 1
ORDER BY cs.order_count DESC;
```

**A5:**
```sql
SELECT r1.name, r1.city, r1.rating
FROM restaurants r1
WHERE NOT EXISTS (
    SELECT 1 FROM restaurants r2 
    WHERE r2.city = r1.city 
    AND r2.id != r1.id 
    AND r2.rating >= r1.rating
);
```

</details>

<details>
<summary>🔑 Section B: Query Optimization</summary>

**B1 (Q6):**
```sql
SELECT c.name, COALESCE(SUM(o.amount), 0) AS total, COUNT(o.id) AS cnt
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total DESC;
```
One pass with JOIN instead of two correlated subqueries.

**B2 (Q7):**
```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```
This enables an Index Scan instead of Sequential Scan.

**B3 (Q8):**
1. Increase `work_mem` (e.g., `SET work_mem = '64MB';`)
2. Add an index that matches the ORDER BY clause so PostgreSQL doesn't need to sort

**B4 (Q9):** Nested Loop runs the inner scan once per outer row. 500,000 outer rows × 1 inner scan each = 500,000 index lookups. A Hash Join or Merge Join would be more efficient — both scan each table once.

**B5 (Q10):**
```sql
-- Page 1
SELECT * FROM orders 
ORDER BY order_date DESC, id DESC 
LIMIT 50;

-- Page 2 (use last row values from page 1)
SELECT * FROM orders 
WHERE (order_date, id) < ('2026-06-09 14:30:00', 4850)  -- last row from page 1
ORDER BY order_date DESC, id DESC 
LIMIT 50;
```

</details>

<details>
<summary>🔑 Section C: Star Schema</summary>

**C1 (Q11):**
```
Fact: fact_ticket_sales (grain: one row per ticket)
  - date_key → dim_date
  - route_key → dim_route
  - bus_key → dim_bus
  - passenger_key → dim_passenger
  - departure_time_key → dim_time
  - ticket_number (degenerate dimension)
  - Measures: ticket_price, discount, net_amount, seat_number

Dimensions:
  dim_date: date, day_of_week, month, quarter, is_holiday
  dim_route: origin, destination, distance_km, route_type (express/local)
  dim_bus: bus_number, bus_type, capacity, operator_name
  dim_passenger: passenger_id, type (adult/student/senior), is_member
  dim_time: hour, time_of_day, is_peak
```

**C2 (Q12):**
- **Surrogate key:** Artificial key (e.g., auto-increment INT) generated for the warehouse. Advantage: decouples from source systems, handles merges, enables SCD Type 2 (multiple rows for same natural key).
- **Natural key:** Original ID from source system (e.g., customer_id=12345). Advantage: traceable back to source, meaningful to business users.

**C3 (Q13):** A conformed dimension is shared across multiple data marts/fact tables with consistent meaning. Example: `dim_date` is used by both `fact_food_orders` and `fact_ride_trips`. When you join through `dim_date`, "January 2026" means the same thing for both businesses.

**C4 (Q14):** Create ONE fact table at the most granular grain (individual orders). Create a materialized view or aggregate table for daily summaries. You can always aggregate down, but you can't disaggregate back up.

**C5 (Q15):** A degenerate dimension is a dimension attribute stored directly in the fact table because it has no other attributes worth creating a separate table for. Example: `order_number` or `receipt_number` — useful for drill-down but doesn't need its own dimension table.

</details>

<details>
<summary>🔑 Section D: SCD</summary>

**D1 (Q16):**
- **Type 1:** Overwrite old value, no history preserved
- **Type 2:** Insert new row with validity dates, full history preserved
- **Type 3:** Add a previous-value column, one level of history only

**D2 (Q17):**
```sql
-- Expire changed records
UPDATE dim_customer d
SET valid_to = NOW(), is_current = FALSE
FROM source_customers s
WHERE d.customer_id = s.customer_id
AND d.is_current = TRUE
AND (d.city IS DISTINCT FROM s.city OR d.segment IS DISTINCT FROM s.segment);

-- Insert new current records
INSERT INTO dim_customer (customer_id, name, email, city, segment, valid_from, valid_to, is_current)
SELECT s.customer_id, s.name, s.email, s.city, s.segment, NOW(), '9999-12-31', TRUE
FROM source_customers s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer d 
    WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
    AND d.city = s.city AND d.segment = s.segment
);
```

**D3 (Q18):**
```sql
JOIN dim_customer dc ON dc.customer_id = o.customer_id
    AND dc.valid_from <= o.order_date
    AND dc.valid_to > o.order_date
```
This matches the customer version that was active at the time of the order.

**D4 (Q19):** Email is a correction (typo fix), not a business change — no analytical value in history. City changes are business events that affect reporting. The ETL does:
1. UPDATE email on all current rows (Type 1 — just overwrite)
2. Check if city/segment changed → expire + insert new row (Type 2)

</details>

<details>
<summary>🔑 Section E: Window Functions</summary>

**E1 (Q20):**
```sql
SELECT 
    restaurant_id,
    DATE(order_date) AS day,
    SUM(amount) AS daily_rev,
    SUM(SUM(amount)) OVER (
        PARTITION BY restaurant_id ORDER BY DATE(order_date)
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM orders
WHERE status = 'completed'
GROUP BY restaurant_id, DATE(order_date);
```

**E2 (Q21):**
```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount) AS revenue
    FROM orders WHERE status = 'completed'
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT 
    TO_CHAR(month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month,
    ROUND((revenue - LAG(revenue) OVER (ORDER BY month)) /
          NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100, 2) AS growth_pct
FROM monthly ORDER BY month;
```

**E3 (Q22):**
```sql
SELECT * FROM (
    SELECT c.city, c.name, SUM(o.amount) AS total_spent,
           RANK() OVER (PARTITION BY c.city ORDER BY SUM(o.amount) DESC) AS city_rank
    FROM customers c JOIN orders o ON c.id = o.customer_id
    WHERE o.status = 'completed'
    GROUP BY c.city, c.name
) ranked WHERE city_rank <= 3;
```

**E4 (Q23):**
```sql
WITH daily AS (
    SELECT DATE(order_date) AS day, SUM(amount) AS rev
    FROM orders WHERE status = 'completed' GROUP BY DATE(order_date)
)
SELECT day, rev,
    AVG(rev) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7d_avg
FROM daily ORDER BY day;
```

**E5 (Q24):**
```sql
WITH daily AS (
    SELECT DISTINCT customer_id, DATE(order_date) AS day
    FROM orders WHERE status = 'completed'
),
islands AS (
    SELECT customer_id, day,
        day - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY day)::INT AS grp
    FROM daily
)
SELECT customer_id, MAX(streak) AS longest_streak
FROM (
    SELECT customer_id, grp, COUNT(*) AS streak
    FROM islands GROUP BY customer_id, grp
) sub
GROUP BY customer_id
ORDER BY longest_streak DESC;
```

**E6 (Q25):**
```sql
WITH rfm AS (
    SELECT 
        customer_id,
        CURRENT_DATE - MAX(order_date) AS recency_days,
        COUNT(*) AS frequency,
        SUM(amount) AS monetary
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id
),
scored AS (
    SELECT *,
        NTILE(5) OVER (ORDER BY recency_days DESC) AS r_score,
        NTILE(5) OVER (ORDER BY frequency) AS f_score,
        NTILE(5) OVER (ORDER BY monetary) AS m_score
    FROM rfm
)
SELECT customer_id, r_score, f_score, m_score,
    CASE 
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
        WHEN r_score >= 3 AND f_score >= 3 THEN 'Loyal'
        WHEN r_score <= 2 AND f_score >= 3 THEN 'At Risk'
        WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost'
        ELSE 'Average'
    END AS segment
FROM scored;
```

</details>

---

### 📊 Self-Assessment Tracker

After grading, fill in your scores:

| Section | Score | Weak Spots | Action Plan |
|---------|-------|-----------|-------------|
| A: Joins | /20 | | |
| B: Optimization | /20 | | |
| C: Star Schema | /20 | | |
| D: SCD | /20 | | |
| E: Window Functions | /20 | | |
| **Total** | **/100** | | |

---

## 🌙 Evening Block (1 hour): Targeted Review

### Based on Your Weak Spots

**If you scored < 14/20 in Section A (Joins):**
- Re-read Day 22 morning section
- Practice: SQLZoo JOIN exercises, HackerRank "Advanced Joins"
- Focus on: NOT EXISTS, self-joins, LATERAL

**If you scored < 14/20 in Section B (Optimization):**
- Re-read Day 23 afternoon section
- Practice: Run EXPLAIN ANALYZE on 10 different queries
- Focus on: Index strategy, query rewriting, materialized views

**If you scored < 14/20 in Section C (Star Schema):**
- Re-read Day 24 morning section
- Practice: Design star schemas for 3 different business scenarios
- Focus on: Grain decision, dimension attributes, ETL loading

**If you scored < 14/20 in Section D (SCD):**
- Re-read Day 25 morning section
- Practice: Implement full SCD Type 2 in PostgreSQL
- Focus on: Expire + insert pattern, fact table join condition

**If you scored < 14/20 in Section E (Window Functions):**
- Re-read Day 26 full day
- Practice: Mode Analytics SQL Window Functions tutorial
- Focus on: Frame clauses, LAG/LEAD, gap & island

### 📝 Today's Checklist

- [ ] I completed the timed assessment honestly
- [ ] I graded myself and identified weak spots
- [ ] I reviewed at least 2 weak areas with targeted practice
- [ ] I'm ready for tomorrow's portfolio project
- [ ] I pushed any practice code to GitHub

---

*Day 27 complete! Tomorrow: Portfolio Project — building a complete star schema data warehouse.* 🏗️🔥
