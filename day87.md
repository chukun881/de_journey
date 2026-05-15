# 📅 Day 87 — Saturday, 9 August 2026
# 📊 SQL Live Coding: Window Functions Focus (30 Problems)

---

## 🎯 Today's Goal

Window functions are the most-tested SQL topic in DE interviews after basic JOINs. They're also the #1 area where candidates struggle. Today is 100% window functions — 30 problems covering every pattern you'll see.

---

## ☀️ Morning Block (2 hours): Core Window Functions (Q1-15)

### Quick Reference

```sql
-- Ranking
ROW_NUMBER() OVER (ORDER BY col)           -- 1, 2, 3, 4 (always unique)
RANK() OVER (ORDER BY col)                 -- 1, 1, 3, 4 (gaps after ties)
DENSE_RANK() OVER (ORDER BY col)           -- 1, 1, 2, 3 (no gaps)

-- Offset
LAG(col, n) OVER (ORDER BY col)            -- n rows before
LEAD(col, n) OVER (ORDER BY col)           -- n rows after
FIRST_VALUE(col) OVER (ORDER BY ...)       -- First in window
LAST_VALUE(col) OVER (ORDER BY ...)        -- Last in window (needs frame!)

-- Aggregation
SUM(col) OVER (PARTITION BY ...)           -- Running sum
AVG(col) OVER (PARTITION BY ...)           -- Running avg
COUNT(*) OVER (PARTITION BY ...)           -- Running count

-- Distribution
NTILE(n) OVER (ORDER BY col)              -- Divide into n groups
PERCENT_RANK() OVER (ORDER BY col)        -- 0 to 1 percentile
CUME_DIST() OVER (ORDER BY col)           -- Cumulative distribution
```

### Problems 1-15

Using the MakanExpress schema (orders, customers, restaurants, order_items).

**Q1.** Add a row number to each order, partitioned by customer, ordered by date.

**Q2.** Rank restaurants by revenue within each city.

**Q3.** Dense rank customers by number of orders. Show ties.

**Q4.** For each order, show the previous order's amount (same customer).

**Q5.** For each order, show the NEXT order's date (same customer).

**Q6.** Calculate a running total of revenue, ordered by date.

**Q7.** Calculate a running total per customer, ordered by order date.

**Q8.** Show the 3-day moving average of daily revenue.

**Q9.** Find each customer's first order date and last order date using window functions.

**Q10.** Divide customers into 4 quartiles by total spending (NTILE).

**Q11.** For each restaurant, show how its daily revenue compares to its previous day.

**Q12.** Calculate the percentage of total revenue each order represents.

**Q13.** Show each customer's order count and what percentage of their city's total that is.

**Q14.** Find the difference between each customer's spending and the city average.

**Q15.** Using FIRST_VALUE, find each customer's first order amount.

<details>
<summary>🔑 Answers 1-15</summary>

```sql
-- A1
SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as order_seq
FROM orders;

-- A2
SELECT r.city, r.name, SUM(o.total_amount) as revenue,
       RANK() OVER (PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) as city_rank
FROM orders o JOIN restaurants r ON o.restaurant_id = r.id
GROUP BY r.city, r.id, r.name;

-- A3
SELECT customer_id, COUNT(*) as order_count,
       DENSE_RANK() OVER (ORDER BY COUNT(*) DESC) as dr
FROM orders
GROUP BY customer_id;

-- A4
SELECT customer_id, order_date, total_amount,
       LAG(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_amount
FROM orders;

-- A5
SELECT customer_id, order_date,
       LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as next_order_date
FROM orders;

-- A6
SELECT order_date, SUM(total_amount) as daily_rev,
       SUM(SUM(total_amount)) OVER (ORDER BY order_date) as running_total
FROM orders
GROUP BY order_date;

-- A7
SELECT customer_id, order_date, total_amount,
       SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) as customer_running
FROM orders;

-- A8
SELECT order_date, SUM(total_amount) as daily_rev,
       ROUND(AVG(SUM(total_amount)) OVER (
           ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ), 2) as moving_avg_3d
FROM orders
GROUP BY order_date;

-- A9
SELECT DISTINCT customer_id,
       FIRST_VALUE(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as first_order,
       FIRST_VALUE(order_date) OVER (PARTITION BY customer_id ORDER BY order_date DESC) as last_order
FROM orders;

-- A10
WITH spending AS (
    SELECT customer_id, SUM(total_amount) as total
    FROM orders GROUP BY customer_id
)
SELECT customer_id, total,
       NTILE(4) OVER (ORDER BY total DESC) as quartile
FROM spending;

-- A11
SELECT restaurant_id, DATE(order_date) as day, SUM(total_amount) as daily_rev,
       SUM(total_amount) - LAG(SUM(total_amount)) OVER (
           PARTITION BY restaurant_id ORDER BY DATE(order_date)
       ) as diff_from_prev_day
FROM orders
GROUP BY restaurant_id, DATE(order_date);

-- A12
SELECT customer_id, order_date, total_amount,
       ROUND(total_amount * 100.0 / SUM(total_amount) OVER (), 4) as pct_of_total
FROM orders;

-- A13
WITH city_totals AS (
    SELECT c.city, SUM(o.total_amount) as city_total
    FROM orders o JOIN customers c ON o.customer_id = c.id
    GROUP BY c.city
),
cust_totals AS (
    SELECT c.id, c.city, c.name, SUM(o.total_amount) as cust_total
    FROM customers c JOIN orders o ON c.id = o.customer_id
    GROUP BY c.id, c.city, c.name
)
SELECT ct.id, ct.name, ct.city, ct.cust_total,
       ct.cust_total * 100.0 / cv.city_total as pct_of_city
FROM cust_totals ct
JOIN city_totals cv ON ct.city = cv.city;

-- A14
SELECT c.id, c.name, c.city,
       COALESCE(SUM(o.total_amount), 0) as customer_spending,
       COALESCE(SUM(o.total_amount), 0) - AVG(SUM(o.total_amount)) OVER (PARTITION BY c.city) as diff_from_avg
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.city;

-- A15
SELECT customer_id, order_date, total_amount,
       FIRST_VALUE(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) as first_order_amount
FROM orders;
```
</details>

---

## 🌤️ Afternoon Block (2 hours): Advanced Window Patterns (Q16-30)

**Q16.** Find the top 2 restaurants per cuisine by revenue.

**Q17.** Calculate the cumulative sum per customer, resetting each month.

**Q18.** Detect gaps: find customers who didn't order for 30+ days between orders.

**Q19.** Year-over-year revenue comparison per restaurant per month.

**Q20.** Session analysis: group orders into "sessions" (orders within 4 hours = same session).

**Q21.** Find the longest streak of consecutive ordering days per customer.

**Q22.** Show the median order amount per city using PERCENTILE_CONT as window function.

**Q23.** Calculate customer lifetime value (CLV) progression over time.

**Q24.** Find restaurants where the latest month's revenue is in the bottom 25% of their history.

**Q25.** Build a "month-over-month change" report for all cities.

**Q26.** Identify seasonal patterns: average daily revenue by day-of-week.

**Q27.** Calculate the exponential moving average of daily revenue.

**Q28.** Find customers who accelerated ordering (orders per month increasing 3+ months in a row).

**Q29.** Create a "customer health score" using multiple window function metrics.

**Q30.** Partition and rank: for each city-cuisine combo, rank restaurants by both revenue AND rating.

<details>
<summary>🔑 Answers 16-30</summary>

```sql
-- A16
WITH ranked AS (
    SELECT r.cuisine, r.name, SUM(o.total_amount) as revenue,
           ROW_NUMBER() OVER (PARTITION BY r.cuisine ORDER BY SUM(o.total_amount) DESC) as rn
    FROM orders o JOIN restaurants r ON o.restaurant_id = r.id
    GROUP BY r.cuisine, r.id, r.name
)
SELECT cuisine, name, revenue FROM ranked WHERE rn <= 2;

-- A17
SELECT customer_id, DATE_TRUNC('month', order_date) as month, order_date, total_amount,
       SUM(total_amount) OVER (
           PARTITION BY customer_id, DATE_TRUNC('month', order_date)
           ORDER BY order_date
       ) as monthly_cumulative
FROM orders;

-- A18
WITH with_prev AS (
    SELECT customer_id, order_date,
           order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as gap
    FROM orders
)
SELECT customer_id, order_date, gap
FROM with_prev
WHERE gap > 30
ORDER BY gap DESC;

-- A19
WITH monthly AS (
    SELECT restaurant_id, DATE_TRUNC('month', order_date) as month, SUM(total_amount) as revenue
    FROM orders GROUP BY 1, 2
)
SELECT restaurant_id, month, revenue,
       LAG(revenue, 12) OVER (PARTITION BY restaurant_id ORDER BY month) as yoy_revenue,
       ROUND(100.0 * (revenue - LAG(revenue, 12) OVER (PARTITION BY restaurant_id ORDER BY month))
             / NULLIF(LAG(revenue, 12) OVER (PARTITION BY restaurant_id ORDER BY month), 0), 2) as yoy_pct
FROM monthly;

-- A20
WITH with_lag AS (
    SELECT customer_id, order_date,
           EXTRACT(EPOCH FROM (order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date))) / 3600 as hours_since_prev
    FROM orders
),
sessions AS (
    SELECT customer_id, order_date,
           SUM(CASE WHEN hours_since_prev > 4 OR hours_since_prev IS NULL THEN 1 ELSE 0 END)
               OVER (PARTITION BY customer_id ORDER BY order_date) as session_id
    FROM with_lag
)
SELECT customer_id, session_id, MIN(order_date) as session_start,
       MAX(order_date) as session_end, COUNT(*) as orders_in_session
FROM sessions
GROUP BY customer_id, session_id;

-- A21
WITH daily AS (
    SELECT DISTINCT customer_id, DATE(order_date) as order_date
),
with_grp AS (
    SELECT customer_id, order_date,
           DATE(order_date) - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)::int as grp
    FROM daily
)
SELECT customer_id, COUNT(*) as streak_length,
       MIN(order_date) as start_date, MAX(order_date) as end_date
FROM with_grp
GROUP BY customer_id, grp
ORDER BY streak_length DESC;

-- A22
SELECT DISTINCT city,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total_amount)
           OVER (PARTITION BY city) as median_order
FROM orders o JOIN customers c ON o.customer_id = c.id;

-- A23
SELECT customer_id, order_date, total_amount,
       SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) as clv_to_date
FROM orders;

-- A24
WITH monthly AS (
    SELECT restaurant_id, DATE_TRUNC('month', order_date) as month, SUM(total_amount) as revenue
    FROM orders GROUP BY 1, 2
),
with_pctile AS (
    SELECT restaurant_id, month, revenue,
           PERCENT_RANK() OVER (PARTITION BY restaurant_id ORDER BY revenue) as pctile
    FROM monthly
),
latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY restaurant_id ORDER BY month DESC) as rn
    FROM with_pctile
)
SELECT r.name, l.month, l.revenue, l.pctile
FROM latest l
JOIN restaurants r ON l.restaurant_id = r.id
WHERE l.rn = 1 AND l.pctile < 0.25;

-- A25
WITH monthly_city AS (
    SELECT c.city, DATE_TRUNC('month', o.order_date) as month, SUM(o.total_amount) as revenue
    FROM orders o JOIN customers c ON o.customer_id = c.id
    GROUP BY 1, 2
)
SELECT city, month, revenue,
       LAG(revenue) OVER (PARTITION BY city ORDER BY month) as prev_month,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (PARTITION BY city ORDER BY month))
             / NULLIF(LAG(revenue) OVER (PARTITION BY city ORDER BY month), 0), 2) as mom_change_pct
FROM monthly_city
ORDER BY city, month;

-- A26
SELECT EXTRACT(DOW FROM order_date) as day_of_week,
       TO_CHAR(order_date, 'Day') as day_name,
       ROUND(AVG(daily_rev), 2) as avg_daily_revenue
FROM (
    SELECT DATE(order_date) as order_date, SUM(total_amount) as daily_rev
    FROM orders GROUP BY DATE(order_date)
) daily
GROUP BY day_of_week, day_name
ORDER BY day_of_week;

-- A27
WITH daily AS (
    SELECT order_date, SUM(total_amount) as rev
    FROM orders GROUP BY order_date
),
ema AS (
    SELECT order_date, rev,
           SUM(rev * POWER(0.3, ROW_NUMBER() OVER (ORDER BY order_date) - 1))
               / SUM(POWER(0.3, ROW_NUMBER() OVER (ORDER BY order_date) - 1))
               OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as ema
    FROM daily
)
SELECT * FROM ema ORDER BY order_date;

-- A28
WITH monthly_counts AS (
    SELECT customer_id, DATE_TRUNC('month', order_date) as month, COUNT(*) as orders
    FROM orders GROUP BY 1, 2
),
with_change AS (
    SELECT customer_id, month, orders,
           SIGN(orders - LAG(orders) OVER (PARTITION BY customer_id ORDER BY month)) as direction
    FROM monthly_counts
),
streaks AS (
    SELECT customer_id, month,
           SUM(CASE WHEN direction = 1 THEN 1 ELSE 0 END)
               OVER (PARTITION BY customer_id ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as increasing_count
    FROM with_change
)
SELECT DISTINCT customer_id
FROM streaks
WHERE increasing_count >= 3;

-- A29
SELECT c.id, c.name,
       COUNT(*) OVER (PARTITION BY c.id) as total_orders,
       SUM(o.total_amount) OVER (PARTITION BY c.id) as total_spent,
       AVG(o.total_amount) OVER (PARTITION BY c.id) as avg_order_value,
       COUNT(DISTINCT o.restaurant_id) OVER (PARTITION BY c.id) as distinct_restaurants,
       -- Recency: days since last order (lower = better)
       CURRENT_DATE - MAX(o.order_date) OVER (PARTITION BY c.id) as days_since_last
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, o.total_amount, o.restaurant_id, o.order_date;

-- A30
SELECT r.city, r.cuisine, r.name,
       SUM(o.total_amount) as revenue,
       r.rating,
       RANK() OVER (PARTITION BY r.city, r.cuisine ORDER BY SUM(o.total_amount) DESC) as revenue_rank,
       RANK() OVER (PARTITION BY r.city, r.cuisine ORDER BY r.rating DESC) as rating_rank
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.id
GROUP BY r.id, r.city, r.cuisine, r.name, r.rating;
```
</details>

---

## 🌙 Evening (1 hour): Window Function Cheat Sheet + Review

### 🗺️ Window Function Cheat Sheet

```
SYNTAX:
  FUNCTION() OVER (
      [PARTITION BY col]           -- grouping (optional)
      [ORDER BY col]               -- sorting (required for ranking/offset)
      [frame_clause]               -- row range (optional)
  )

FRAME CLAUSES:
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW    -- fixed 3-row window
  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- running total
  RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW -- running with same values

COMMON PATTERNS:
  Running total:     SUM(col) OVER (ORDER BY date)
  Running per group: SUM(col) OVER (PARTITION BY grp ORDER BY date)
  Moving average:    AVG(col) OVER (ORDER BY date ROWS BETWEEN N-1 PRECEDING AND CURRENT ROW)
  Rank top N:        ROW_NUMBER() OVER (PARTITION BY grp ORDER BY col DESC) WHERE rn <= N
  Previous/Next:     LAG(col) / LEAD(col) OVER (PARTITION BY grp ORDER BY date)
  Percentile:        NTILE(4) OVER (ORDER BY col) -- quartiles
  % of total:        col * 100.0 / SUM(col) OVER ()

INTERVIEW TIPS:
  - Always include ORDER BY for ROW_NUMBER/RANK/LAG/LEAD
  - LAST_VALUE needs explicit frame: ROWS BETWEEN ... AND UNBOUNDED FOLLOWING
  - Use CTEs + window functions for top-N-per-group problems
  - Window functions process AFTER WHERE/GROUP BY/HAVING
```

### 📝 Today's Checklist

- [ ] Completed 30 window function problems
- [ ] Comfortable with ROW_NUMBER, RANK, DENSE_RANK
- [ ] Comfortable with LAG, LEAD, running totals
- [ ] Can solve top-N-per-group pattern
- [ ] Can solve session/gap detection
- [ ] Saved cheat sheet for review

---

*Day 87 complete! Tomorrow: Python live coding.* 🐍
