# 📅 Day 86 — Friday, 8 August 2026
# 💻 SQL Live Coding: Intermediate-Advanced (50 Problems)

---

## 🎯 Today's Goal

Yesterday was warm-up. Today the problems get harder — the kind you see in real DE interviews at companies like Grab, Shopee, and Singtel. Complex JOINs, self-joins, nested CTEs, conditional aggregation, and performance awareness.

---

## ☀️ Morning Block (2 hours): Problems 1-25 (Complex JOINs, CTEs, Subqueries)

### Schema (MakanExpress Food Delivery)

```sql
-- customers: id, name, email, city, signup_date, segment ('new','regular','vip')
-- restaurants: id, name, cuisine, city, rating, opened_date, is_active
-- orders: id, customer_id, restaurant_id, order_date, total_amount, status, delivery_time_min
-- order_items: id, order_id, menu_item, quantity, unit_price
-- payments: id, order_id, method ('card','cash','grabpay','wallet'), amount, status
-- deliveries: id, order_id, driver_id, pickup_time, dropoff_time, rating
-- drivers: id, name, city, vehicle_type, signup_date
```

### Batch 1: Complex Multi-Table JOINs (Q1-10)

**Q1.** For each order, show: customer name, restaurant name, cuisine, payment method, delivery rating.

**Q2.** Find customers who ordered from at least 3 different cuisines.

**Q3.** Show the average delivery rating per driver, with driver name and city.

**Q4.** Find orders where the payment amount doesn't match the order total.

**Q5.** For each restaurant, show the % of orders paid via GrabPay.

**Q6.** Find the most ordered menu item per cuisine category.

**Q7.** List drivers who have delivered to all 4 cities (Singapore, KL, Penang, JB).

**Q8.** Find customers whose average order value increased month-over-month for 3 consecutive months.

**Q9.** Show revenue per restaurant per month, with a column showing the previous month's revenue.

**Q10.** Find pairs of customers who live in the same city and have ordered from the same restaurant on the same day.

<details>
<summary>🔑 Answers 1-10</summary>

```sql
-- A1
SELECT o.id, c.name as customer, r.name as restaurant, r.cuisine,
       p.method as payment, d.rating as delivery_rating
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN restaurants r ON o.restaurant_id = r.id
LEFT JOIN payments p ON o.id = p.order_id
LEFT JOIN deliveries d ON o.id = d.order_id;

-- A2
SELECT c.name
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN restaurants r ON o.restaurant_id = r.id
GROUP BY c.id, c.name
HAVING COUNT(DISTINCT r.cuisine) >= 3;

-- A3
SELECT dr.name, dr.city, ROUND(AVG(d.rating), 2) as avg_rating, COUNT(*) as deliveries
FROM drivers dr
JOIN deliveries d ON dr.id = d.driver_id
GROUP BY dr.id, dr.name, dr.city
ORDER BY avg_rating DESC;

-- A4
SELECT o.id, o.total_amount, p.amount, o.total_amount - p.amount as difference
FROM orders o
JOIN payments p ON o.id = p.order_id
WHERE o.total_amount != p.amount;

-- A5
SELECT r.name,
       COUNT(*) as total_orders,
       SUM(CASE WHEN p.method = 'grabpay' THEN 1 ELSE 0 END) as grabpay_orders,
       ROUND(100.0 * SUM(CASE WHEN p.method = 'grabpay' THEN 1 ELSE 0 END) / COUNT(*), 2) as grabpay_pct
FROM restaurants r
JOIN orders o ON r.id = o.restaurant_id
JOIN payments p ON o.id = p.order_id
GROUP BY r.id, r.name
ORDER BY grabpay_pct DESC;

-- A6
WITH item_counts AS (
    SELECT r.cuisine, oi.menu_item, SUM(oi.quantity) as total_qty,
           ROW_NUMBER() OVER (PARTITION BY r.cuisine ORDER BY SUM(oi.quantity) DESC) as rn
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    JOIN restaurants r ON o.restaurant_id = r.id
    GROUP BY r.cuisine, oi.menu_item
)
SELECT cuisine, menu_item, total_qty
FROM item_counts WHERE rn = 1;

-- A7
SELECT dr.name
FROM drivers dr
JOIN deliveries d ON dr.id = d.driver_id
JOIN orders o ON d.order_id = o.id
JOIN customers c ON o.customer_id = c.id
GROUP BY dr.id, dr.name
HAVING COUNT(DISTINCT c.city) = 4;

-- A8
WITH monthly_avg AS (
    SELECT customer_id,
           DATE_TRUNC('month', order_date) as month,
           AVG(total_amount) as avg_amount
    FROM orders
    GROUP BY 1, 2
),
with_lags AS (
    SELECT customer_id, month, avg_amount,
           LAG(avg_amount, 1) OVER (PARTITION BY customer_id ORDER BY month) as prev1,
           LAG(avg_amount, 2) OVER (PARTITION BY customer_id ORDER BY month) as prev2
    FROM monthly_avg
)
SELECT DISTINCT customer_id
FROM with_lags
WHERE avg_amount > prev1 AND prev1 > prev2
  AND prev1 IS NOT NULL AND prev2 IS NOT NULL;

-- A9
WITH monthly_rev AS (
    SELECT r.id, r.name, DATE_TRUNC('month', o.order_date) as month,
           SUM(o.total_amount) as revenue
    FROM restaurants r
    JOIN orders o ON r.id = o.restaurant_id
    GROUP BY r.id, r.name, month
)
SELECT name, month, revenue,
       LAG(revenue) OVER (PARTITION BY id ORDER BY month) as prev_month_revenue
FROM monthly_rev
ORDER BY name, month;

-- A10
SELECT c1.name, c2.name, r.name as restaurant, o1.order_date
FROM orders o1
JOIN orders o2 ON o1.restaurant_id = o2.restaurant_id
    AND DATE(o1.order_date) = DATE(o2.order_date)
    AND o1.customer_id < o2.customer_id
JOIN customers c1 ON o1.customer_id = c1.id
JOIN customers c2 ON o2.customer_id = c2.id
JOIN restaurants r ON o1.restaurant_id = r.id
WHERE c1.city = c2.city;
```
</details>

### Batch 2: CTEs + Subqueries (Q11-18)

**Q11.** Find the top 20% of customers by total spending.

**Q12.** Calculate the average time between a customer's first and second order.

**Q13.** For each restaurant, what % of their revenue comes from their top 10% of customers?

**Q14.** Find orders where the delivery took longer than the restaurant's average delivery time.

**Q15.** Identify "power users": customers who placed orders in 4+ distinct weeks in the last month.

**Q16.** Build a funnel: how many customers progressed from signup → first order → second order → third order?

**Q17.** Find restaurants where the average order value is declining (compare last 3 months to previous 3 months).

**Q18.** For each city, find the cuisine with the fastest average delivery time.

<details>
<summary>🔑 Answers 11-18</summary>

```sql
-- A11
WITH ranked AS (
    SELECT customer_id, SUM(total_amount) as total,
           NTILE(5) OVER (ORDER BY SUM(total_amount) DESC) as quintile
    FROM orders
    GROUP BY customer_id
)
SELECT customer_id, total FROM ranked WHERE quintile = 1;

-- A12
WITH order_numbered AS (
    SELECT customer_id, order_date,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as rn
    FROM orders
),
first_two AS (
    SELECT customer_id,
           MIN(CASE WHEN rn = 1 THEN order_date END) as first,
           MIN(CASE WHEN rn = 2 THEN order_date END) as second
    FROM order_numbered
    WHERE rn <= 2
    GROUP BY customer_id
    HAVING COUNT(*) = 2
)
SELECT AVG(EXTRACT(EPOCH FROM (second - first)) / 3600) as avg_hours_between
FROM first_two;

-- A13
WITH cust_rev AS (
    SELECT o.restaurant_id, o.customer_id, SUM(o.total_amount) as revenue
    FROM orders o
    GROUP BY o.restaurant_id, o.customer_id
),
top10pct AS (
    SELECT restaurant_id, customer_id, revenue,
           NTILE(10) OVER (PARTITION BY restaurant_id ORDER BY revenue DESC) as decile
    FROM cust_rev
)
SELECT t.restaurant_id, r.name,
       SUM(CASE WHEN decile = 1 THEN revenue ELSE 0 END) as top10_revenue,
       SUM(revenue) as total_revenue,
       ROUND(100.0 * SUM(CASE WHEN decile = 1 THEN revenue ELSE 0 END) / SUM(revenue), 2) as pct_from_top10
FROM top10pct t
JOIN restaurants r ON t.restaurant_id = r.id
GROUP BY t.restaurant_id, r.name;

-- A14
WITH rest_avg AS (
    SELECT restaurant_id, AVG(delivery_time_min) as avg_time
    FROM orders
    GROUP BY restaurant_id
)
SELECT o.id, r.name, o.delivery_time_min, ra.avg_time
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.id
JOIN rest_avg ra ON o.restaurant_id = ra.restaurant_id
WHERE o.delivery_time_min > ra.avg_time;

-- A15
WITH weekly AS (
    SELECT customer_id, DATE_TRUNC('week', order_date) as week
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '1 month'
)
SELECT customer_id, COUNT(DISTINCT week) as active_weeks
FROM weekly
GROUP BY customer_id
HAVING COUNT(DISTINCT week) >= 4;

-- A16
WITH order_counts AS (
    SELECT customer_id, COUNT(*) as order_count
    FROM orders
    GROUP BY customer_id
)
SELECT
    (SELECT COUNT(*) FROM customers) as total_signed_up,
    (SELECT COUNT(*) FROM order_counts WHERE order_count >= 1) as first_order,
    (SELECT COUNT(*) FROM order_counts WHERE order_count >= 2) as second_order,
    (SELECT COUNT(*) FROM order_counts WHERE order_count >= 3) as third_order;

-- A17
WITH monthly_rev AS (
    SELECT restaurant_id,
           DATE_TRUNC('month', order_date) as month,
           AVG(total_amount) as avg_order_val
    FROM orders
    GROUP BY 1, 2
),
compared AS (
    SELECT restaurant_id,
           AVG(CASE WHEN month >= CURRENT_DATE - INTERVAL '3 months' THEN avg_order_val END) as recent_avg,
           AVG(CASE WHEN month < CURRENT_DATE - INTERVAL '3 months'
                    AND month >= CURRENT_DATE - INTERVAL '6 months' THEN avg_order_val END) as prev_avg
    FROM monthly_rev
    GROUP BY restaurant_id
)
SELECT r.name, c.recent_avg, c.prev_avg,
       ROUND(100.0 * (c.recent_avg - c.prev_avg) / c.prev_avg, 2) as pct_change
FROM compared c
JOIN restaurants r ON c.restaurant_id = r.id
WHERE c.recent_avg < c.prev_avg;

-- A18
WITH city_cuisine AS (
    SELECT c.city, r.cuisine, AVG(o.delivery_time_min) as avg_time,
           ROW_NUMBER() OVER (PARTITION BY c.city ORDER BY AVG(o.delivery_time_min)) as rn
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    JOIN restaurants r ON o.restaurant_id = r.id
    GROUP BY c.city, r.cuisine
)
SELECT city, cuisine, ROUND(avg_time, 1) as avg_delivery_min
FROM city_cuisine WHERE rn = 1;
```
</details>

### Batch 3: Self-Joins + Hierarchical (Q19-25)

**Q19.** Find customers who ordered the same menu item on consecutive orders.

**Q20.** For each restaurant, compare each month's revenue to the same month last year (YoY).

**Q21.** Find restaurants whose rating improved by >0.5 after being open for 6 months.

**Q22.** Identify orders where the same customer ordered from the same restaurant within 24 hours.

**Q23.** For each driver, show their delivery count trend (increasing, decreasing, stable).

**Q24.** Find the longest gap between orders for each customer.

**Q25.** Calculate the probability that a customer who ordered once will order again (1-week retention).

<details>
<summary>🔑 Answers 19-25</summary>

```sql
-- A19
WITH numbered AS (
    SELECT o.customer_id, oi.menu_item, o.order_date,
           LAG(oi.menu_item) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) as prev_item
    FROM orders o
    JOIN order_items oi ON o.id = oi.order_id
)
SELECT DISTINCT customer_id, menu_item
FROM numbered
WHERE menu_item = prev_item AND prev_item IS NOT NULL;

-- A20
WITH monthly AS (
    SELECT restaurant_id,
           DATE_TRUNC('month', order_date) as month,
           SUM(total_amount) as revenue
    FROM orders
    GROUP BY 1, 2
)
SELECT r.name, m1.month, m1.revenue as this_year,
       m2.revenue as last_year,
       ROUND(100.0 * (m1.revenue - m2.revenue) / NULLIF(m2.revenue, 0), 2) as yoy_change
FROM monthly m1
JOIN monthly m2 ON m1.restaurant_id = m2.restaurant_id
    AND m2.month = m1.month - INTERVAL '1 year'
JOIN restaurants r ON m1.restaurant_id = r.id;

-- A21
SELECT r.name, r.rating as current_rating, r.opened_date
FROM restaurants r
WHERE r.rating > 4.0
  AND r.opened_date < CURRENT_DATE - INTERVAL '6 months'
  AND r.is_active = true;

-- Note: This requires historical ratings which we don't have. In an interview, explain:
-- "If we had a restaurant_ratings_history table, I'd compare avg rating in months 1-6 vs months 7-12"

-- A22
SELECT o1.customer_id, o1.restaurant_id, o1.order_date as order1, o2.order_date as order2,
       EXTRACT(EPOCH FROM (o2.order_date - o1.order_date)) / 3600 as hours_between
FROM orders o1
JOIN orders o2 ON o1.customer_id = o2.customer_id
    AND o1.restaurant_id = o2.restaurant_id
    AND o2.order_date > o1.order_date
    AND o2.order_date < o1.order_date + INTERVAL '24 hours';

-- A23
WITH monthly_deliveries AS (
    SELECT driver_id, DATE_TRUNC('month', dropoff_time) as month, COUNT(*) as deliveries
    FROM deliveries
    GROUP BY 1, 2
),
with_trend AS (
    SELECT driver_id, month, deliveries,
           deliveries - LAG(deliveries) OVER (PARTITION BY driver_id ORDER BY month) as change
    FROM monthly_deliveries
)
SELECT driver_id,
       CASE
           WHEN AVG(change) > 1 THEN 'increasing'
           WHEN AVG(change) < -1 THEN 'decreasing'
           ELSE 'stable'
       END as trend
FROM with_trend
WHERE change IS NOT NULL
GROUP BY driver_id;

-- A24
WITH order_gaps AS (
    SELECT customer_id, order_date,
           order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as gap
    FROM orders
)
SELECT customer_id, MAX(gap) as longest_gap
FROM order_gaps
WHERE gap IS NOT NULL
GROUP BY customer_id
ORDER BY longest_gap DESC;

-- A25
WITH first_order AS (
    SELECT customer_id, MIN(order_date) as first_date
    FROM orders
    GROUP BY customer_id
),
returned AS (
    SELECT f.customer_id
    FROM first_order f
    JOIN orders o ON f.customer_id = o.customer_id
        AND o.order_date > f.first_date
        AND o.order_date <= f.first_date + INTERVAL '7 days'
)
SELECT
    (SELECT COUNT(*) FROM first_order) as total_customers,
    (SELECT COUNT(*) FROM returned) as returned_customers,
    ROUND(100.0 * (SELECT COUNT(*) FROM returned) / (SELECT COUNT(*) FROM first_order), 2) as retention_pct;
```
</details>

---

## 🌤️ Afternoon Block (2 hours): Problems 26-50 (Window Functions, Performance)

### Batch 4: Advanced Window Functions (Q26-35)

**Q26.** Calculate a 7-day rolling average of daily revenue.

**Q27.** Find the cumulative distribution of order amounts (what percentile is each order).

**Q28.** For each customer, show their order sequence number and gap from previous order.

**Q29.** Find the top 3 revenue-generating restaurants per city, per month.

**Q30.** Detect gaps in ordering: customers who haven't ordered in 30+ days but were previously active.

**Q31.** Show a year-over-year comparison table: rows = months, columns = years.

**Q32.** Calculate the Gini coefficient of order amounts (measure of inequality).

**Q33.** Find restaurants whose daily order count deviates >2 standard deviations from their mean.

**Q34.** Create a user engagement score: (orders last 30 days) × 3 + (distinct restaurants) × 2 + (total spent / 100).

**Q35.** Calculate the "time to first order" for each customer (days between signup and first order).

<details>
<summary>🔑 Answers 26-35</summary>

```sql
-- A26
SELECT order_date, SUM(total_amount) as daily_rev,
       ROUND(AVG(SUM(total_amount)) OVER (
           ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ), 2) as rolling_7d_avg
FROM orders
GROUP BY order_date
ORDER BY order_date;

-- A27
SELECT id, total_amount,
       PERCENT_RANK() OVER (ORDER BY total_amount) as percentile,
       CUME_DIST() OVER (ORDER BY total_amount) as cumulative_dist
FROM orders;

-- A28
SELECT customer_id, order_date, total_amount,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as order_seq,
       order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as gap_from_prev
FROM orders;

-- A29
WITH ranked AS (
    SELECT DATE_TRUNC('month', o.order_date) as month,
           r.city, r.name,
           SUM(o.total_amount) as revenue,
           ROW_NUMBER() OVER (PARTITION BY DATE_TRUNC('month', o.order_date), r.city
                              ORDER BY SUM(o.total_amount) DESC) as rn
    FROM orders o
    JOIN restaurants r ON o.restaurant_id = r.id
    GROUP BY month, r.city, r.name, r.id
)
SELECT * FROM ranked WHERE rn <= 3
ORDER BY month, city, rn;

-- A30
WITH last_order AS (
    SELECT customer_id, MAX(order_date) as last_date
    FROM orders
    GROUP BY customer_id
    HAVING MAX(order_date) < CURRENT_DATE - INTERVAL '30 days'
),
was_active AS (
    SELECT customer_id
    FROM orders
    WHERE order_date < CURRENT_DATE - INTERVAL '30 days'
    GROUP BY customer_id
    HAVING COUNT(*) >= 3
)
SELECT lo.customer_id, lo.last_date,
       CURRENT_DATE - lo.last_date as days_since_last
FROM last_order lo
JOIN was_active wa ON lo.customer_id = wa.customer_id;

-- A31
SELECT DATE_TRUNC('month', order_date) as month,
       SUM(CASE WHEN EXTRACT(YEAR FROM order_date) = 2025 THEN total_amount ELSE 0 END) as rev_2025,
       SUM(CASE WHEN EXTRACT(YEAR FROM order_date) = 2026 THEN total_amount ELSE 0 END) as rev_2026
FROM orders
WHERE order_date >= '2025-01-01'
GROUP BY month
ORDER BY month;

-- A32
-- Note: Full Gini requires a recursive approach. Here's the simplified interview answer:
WITH sorted AS (
    SELECT total_amount,
           ROW_NUMBER() OVER (ORDER BY total_amount) as rn,
           COUNT(*) OVER () as n,
           SUM(total_amount) OVER () as total_sum
)
SELECT 1 - 2.0 * SUM((n + 1 - rn) * total_amount) / (n * SUM(total_amount)) as gini
FROM sorted;

-- A33
WITH rest_stats AS (
    SELECT restaurant_id, DATE(order_date) as day, COUNT(*) as daily_count
    FROM orders
    GROUP BY restaurant_id, day
),
stats AS (
    SELECT restaurant_id, AVG(daily_count) as mean, STDDEV(daily_count) as sd
    FROM rest_stats
    GROUP BY restaurant_id
)
SELECT rs.restaurant_id, r.name, rs.day, rs.daily_count, s.mean, s.sd
FROM rest_stats rs
JOIN stats s ON rs.restaurant_id = s.restaurant_id
JOIN restaurants r ON rs.restaurant_id = r.id
WHERE ABS(rs.daily_count - s.mean) > 2 * s.sd;

-- A34
SELECT c.id, c.name,
       COALESCE(o30.order_count, 0) * 3 +
       COALESCE(o_all.distinct_restaurants, 0) * 2 +
       COALESCE(o_all.total_spent, 0) / 100.0 as engagement_score
FROM customers c
LEFT JOIN (
    SELECT customer_id, COUNT(*) as order_count
    FROM orders WHERE order_date >= CURRENT_DATE - 30 GROUP BY customer_id
) o30 ON c.id = o30.customer_id
LEFT JOIN (
    SELECT customer_id, COUNT(DISTINCT restaurant_id) as distinct_restaurants,
           SUM(total_amount) as total_spent
    FROM orders GROUP BY customer_id
) o_all ON c.id = o_all.customer_id
ORDER BY engagement_score DESC;

-- A35
SELECT c.id, c.name, c.signup_date, MIN(o.order_date) as first_order,
       MIN(o.order_date) - c.signup_date as days_to_first_order
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.signup_date
ORDER BY days_to_first_order;
```
</details>

### Batch 5: Performance + Optimization (Q36-50)

**Q36.** Write a query to find duplicate orders (same customer, restaurant, date, amount).

**Q37.** Optimize: "This query is slow. How would you speed it up?" `SELECT * FROM orders WHERE YEAR(order_date) = 2026`

**Q38.** Write a query using EXISTS instead of IN for better performance.

**Q39.** Explain when to use UNION vs UNION ALL. Give an example.

**Q40.** Write an incremental load query (only new/changed rows since last run).

**Q41.** Create a materialized view for daily restaurant performance.

**Q42.** Write a query that handles division by zero safely.

**Q43.** Explain the difference between COUNT(*), COUNT(1), and COUNT(column).

**Q44.** Write a query using COALESCE to handle NULLs in a report.

**Q45.** Explain what a covering index is and when to use one.

**Q46.** Write an UPSERT (INSERT ... ON CONFLICT) for the orders table.

**Q47.** Design an index strategy for: `SELECT * FROM orders WHERE customer_id = ? AND order_date BETWEEN ? AND ?`

**Q48.** Explain query execution order (FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY).

**Q49.** Write a recursive CTE: find the management chain (driver → team lead → manager → director).

**Q50.** Explain the N+1 query problem with an example from data engineering.

<details>
<summary>🔑 Answers 36-50</summary>

**A36:**
```sql
SELECT customer_id, restaurant_id, DATE(order_date) as order_day, total_amount, COUNT(*) as dupes
FROM orders
GROUP BY customer_id, restaurant_id, DATE(order_date), total_amount
HAVING COUNT(*) > 1;
```

**A37:** The function `YEAR(order_date)` prevents index usage. Rewrite as:
```sql
SELECT * FROM orders
WHERE order_date >= '2026-01-01' AND order_date < '2027-01-01';
-- This is "sargable" — the database can use an index on order_date
```

**A38:**
```sql
-- Slower (IN with subquery):
SELECT * FROM customers WHERE id IN (SELECT customer_id FROM orders WHERE city = 'Singapore');

-- Faster (EXISTS — stops scanning once match found):
SELECT c.* FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id AND o.city = 'Singapore'
);
```

**A39:** UNION removes duplicates (slower, needs sort). UNION ALL keeps duplicates (faster). Use UNION ALL when you know there are no duplicates or when duplicates are acceptable.

**A40:**
```sql
INSERT INTO orders_dim
SELECT * FROM orders_source o
WHERE o.updated_at > (SELECT COALESCE(MAX(last_sync), '1970-01-01') FROM sync_tracker)
ON CONFLICT (id) DO UPDATE SET
    total_amount = EXCLUDED.total_amount,
    status = EXCLUDED.status,
    updated_at = EXCLUDED.updated_at;
```

**A41:**
```sql
CREATE MATERIALIZED VIEW mv_daily_restaurant_performance AS
SELECT restaurant_id, DATE(order_date) as order_date,
       COUNT(*) as order_count,
       SUM(total_amount) as revenue,
       AVG(total_amount) as avg_order_value,
       AVG(delivery_time_min) as avg_delivery_time
FROM orders
GROUP BY restaurant_id, DATE(order_date);

-- Refresh daily:
REFRESH MATERIALIZED VIEW mv_daily_restaurant_performance;
```

**A42:**
```sql
-- Use NULLIF to prevent division by zero:
SELECT city, SUM(total_amount) / NULLIF(COUNT(*), 0) as avg_per_order
FROM orders
GROUP BY city;
```

**A43:** `COUNT(*)` counts all rows including NULLs. `COUNT(1)` is the same as COUNT(*) in most databases. `COUNT(column)` counts non-NULL values in that column. Use COUNT(*) for row counting, COUNT(column) when you need to exclude NULLs.

**A44:**
```sql
SELECT c.name,
       COALESCE(SUM(o.total_amount), 0) as total_spent,
       COALESCE(COUNT(o.id), 0) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;
```

**A45:** A covering index includes ALL columns needed by a query, so the database doesn't need to look up the actual row. Example:
```sql
CREATE INDEX idx_orders_covering ON orders (customer_id, order_date) INCLUDE (total_amount, status);
-- Now this query only reads the index, not the table:
SELECT total_amount, status FROM orders WHERE customer_id = 123 AND order_date > '2026-01-01';
```

**A46:**
```sql
INSERT INTO orders (id, customer_id, restaurant_id, order_date, total_amount, status)
VALUES (1001, 5, 12, '2026-08-08', 45.50, 'delivered')
ON CONFLICT (id) DO UPDATE SET
    total_amount = EXCLUDED.total_amount,
    status = EXCLUDED.status;
```

**A47:**
```sql
CREATE INDEX idx_orders_cust_date ON orders (customer_id, order_date) INCLUDE (total_amount, status);
-- Composite index on (customer_id, order_date) supports both filter conditions
-- INCLUDE covers common select columns without table lookup
```

**A48:** SQL logical execution order:
1. FROM/JOIN — determine data sources
2. WHERE — filter rows
3. GROUP BY — group rows
4. HAVING — filter groups
5. SELECT — compute columns
6. DISTINCT — remove duplicates
7. ORDER BY — sort
8. LIMIT/OFFSET — paginate

This is why you can't use column aliases in WHERE (it runs before SELECT).

**A49:**
```sql
WITH RECURSIVE management_chain AS (
    -- Base case: start with a specific driver
    SELECT id, name, manager_id, 0 as level
    FROM employees WHERE id = 101  -- starting employee
    
    UNION ALL
    
    -- Recursive: go up the chain
    SELECT e.id, e.name, e.manager_id, mc.level + 1
    FROM employees e
    JOIN management_chain mc ON e.id = mc.manager_id
)
SELECT name, level FROM management_chain ORDER BY level;
```

**A50:** The N+1 problem: if you query 1000 customers, then run 1 query per customer to get their orders, that's 1001 queries instead of 1 JOIN. In data engineering, this happens when:
- Loading data row-by-row instead of batching
- Making individual API calls instead of bulk requests
- Running a SQL query inside a loop

Fix: Use JOINs, batch operations, or window functions to do it in one query.
</details>

---

## 🌙 Evening (1 hour): Score + Weak Pattern Analysis

### Scoring

| Score | Assessment |
|-------|-----------|
| 45-50 | 🔥 Ready for advanced interviews |
| 35-44 | ✅ Good, review mistakes |
| 25-34 | ⚠️ Practice window functions + CTEs |
| <25 | 🔴 Revisit Weeks 1 + 4 |

### Pattern Analysis

| If you struggled with | Review |
|----------------------|--------|
| Multi-table JOINs | Day 2, Day 22 |
| Self-joins | Practice more Q19-25 |
| Window functions | Day 4, Day 26, Day 87 |
| CTEs | Day 3 |
| Performance | Day 23, Day 40 |
| NULL handling | Practice COALESCE, LEFT JOINs |

### 📝 Today's Checklist

- [ ] Completed all 50 problems (timed, 2-3 min each)
- [ ] Score: __/50
- [ ] Identified weak patterns
- [ ] Noted problems to re-do tomorrow
- [ ] Updated weak spot tracker

---

*Day 86 complete! Tomorrow: Window functions focus day.* 🪟
