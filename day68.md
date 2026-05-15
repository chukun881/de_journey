# 📅 Day 68 — Monday, 21 July 2026
# 💼 SQL Interview Practice: 30 Medium-Hard Problems

---

## 🎯 Today's Goal

Yesterday was warm-up. Today is the real deal — 30 medium-to-hard SQL problems that mirror what you'll see in actual SG/MY data engineering interviews. These test: window functions, CTEs, self-joins, date manipulation, and complex aggregations.

**Time yourself:** 3-5 minutes per problem. If stuck for more than 8 minutes, look at the answer and move on.

---

## ☀️ Morning Block (2 hours): Problems 31-45

Use the same MakanExpress schema from Day 67.

### Problem 31: Gap in Order Dates
Find dates where NO orders were placed (gaps in the order timeline).

<details>
<summary>🔑 Answer</summary>

```sql
WITH date_series AS (
    SELECT generate_series(
        (SELECT MIN(DATE(order_date) FROM orders),
        (SELECT MAX(DATE(order_date) FROM orders),
        INTERVAL '1 day'
    )::DATE as order_date
),
order_dates AS (
    SELECT DISTINCT DATE(order_date) as order_date
    FROM orders
)
SELECT ds.order_date as gap_date
FROM date_series ds
LEFT JOIN order_dates od ON ds.order_date = od.order_date
WHERE od.order_date IS NULL
ORDER BY ds.order_date;
```

</details>

### Problem 32: Customer's 2nd Order
Find each customer's second order (date and amount). Use LEAD/LAG.

<details>
<summary>🔑 Answer</summary>

```sql
WITH ordered AS (
    SELECT 
        customer_id,
        order_date,
        total_amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as rn
    FROM orders
    WHERE status = 'delivered'
)
SELECT c.name, o.order_date, o.total_amount
FROM ordered o
JOIN customers c ON o.customer_id = c.id
WHERE o.rn = 2;
```

</details>

### Problem 33: Median Order Value
Calculate the median order value for each city.

<details>
<summary>🔑 Answer</summary>

```sql
WITH ranked AS (
    SELECT 
        delivery_city,
        total_amount,
        ROW_NUMBER() OVER (PARTITION BY delivery_city ORDER BY total_amount) as rn,
        COUNT(*) OVER (PARTITION BY delivery_city) as cnt
    FROM orders
    WHERE status = 'delivered'
)
SELECT 
    delivery_city,
    AVG(total_amount) as median_value
FROM ranked
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0))
GROUP BY delivery_city
ORDER BY median_value DESC;
```

</details>

### Problem 34: Time Between Orders
Calculate the average time between a customer's consecutive orders.

<details>
<summary>🔑 Answer</summary>

```sql
WITH ordered AS (
    SELECT 
        customer_id,
        order_date,
        LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_order
    FROM orders
    WHERE status = 'delivered'
)
SELECT 
    c.name,
    ROUND(AVG(EXTRACT(EPOCH FROM (order_date - prev_order)) / 3600), 1) as avg_hours_between
FROM ordered o
JOIN customers c ON o.customer_id = c.id
WHERE prev_order IS NOT NULL
GROUP BY c.id, c.name
ORDER BY avg_hours_between;
```

</details>

### Problem 35: Restaurants Above Their Cuisine Average
Find restaurants whose rating is above the average rating of their cuisine type.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT r.name, r.cuisine, r.rating, cuisine_avg.avg_rating
FROM restaurants r
JOIN (
    SELECT cuisine, AVG(rating) as avg_rating
    FROM restaurants
    GROUP BY cuisine
) cuisine_avg ON r.cuisine = cuisine_avg.cuisine
WHERE r.rating > cuisine_avg.avg_rating
ORDER BY r.rating DESC;
```

</details>

### Problem 36: YTD Revenue Comparison
Show year-to-date revenue for this year and the same period last year, side by side.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    EXTRACT(MONTH FROM order_date) as month_num,
    TO_CHAR(order_date, 'Month') as month_name,
    SUM(CASE WHEN EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE) 
             THEN total_amount ELSE 0 END) as current_ytd,
    SUM(CASE WHEN EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE) - 1
             THEN total_amount ELSE 0 END) as previous_ytd
FROM orders
WHERE status = 'delivered'
  AND order_date >= DATE_TRUNC('year', CURRENT_DATE) - INTERVAL '1 year'
GROUP BY EXTRACT(MONTH FROM order_date), TO_CHAR(order_date, 'Month')
ORDER BY month_num;
```

</details>

### Problem 37: Find the Most Expensive Item per Order
For each order, find the most expensive single item.

<details>
<summary>🔑 Answer</summary>

```sql
WITH ranked AS (
    SELECT 
        order_id,
        item_name,
        unit_price,
        quantity,
        unit_price * quantity as line_total,
        RANK() OVER (PARTITION BY order_id ORDER BY unit_price * quantity DESC) as rn
    FROM order_items
)
SELECT r.order_id, r.item_name, r.line_total as most_expensive_item
FROM ranked r
WHERE r.rn = 1
ORDER BY r.line_total DESC;
```

</details>

### Problem 38: Customer Churn Detection
Find customers who were active (ordered) in month X but NOT in month X+1. Use a 30-day inactivity threshold.

<details>
<summary>🔑 Answer</summary>

```sql
WITH monthly_activity AS (
    SELECT DISTINCT
        customer_id,
        DATE_TRUNC('month', order_date) as active_month
    FROM orders
    WHERE status = 'delivered'
),
with_next AS (
    SELECT 
        customer_id,
        active_month,
        LEAD(active_month) OVER (PARTITION BY customer_id ORDER BY active_month) as next_active
    FROM monthly_activity
)
SELECT 
    c.name,
    w.active_month as last_active_month,
    CASE WHEN w.next_active IS NULL THEN 'Churned (no return)'
         WHEN w.next_active > w.active_month + INTERVAL '1 month' THEN 'Churned (returned later)'
         ELSE 'Active' END as status
FROM with_next w
JOIN customers c ON w.customer_id = c.id
WHERE w.next_active IS NULL OR w.next_active > w.active_month + INTERVAL '1 month'
ORDER BY w.active_month DESC;
```

</details>

### Problem 39: Revenue Contribution per Customer
For each customer, show their revenue AND what percentage of total revenue they represent. Only show top 10.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    c.name,
    SUM(o.total_amount) as customer_revenue,
    ROUND(SUM(o.total_amount) / SUM(SUM(o.total_amount)) OVER () * 100, 2) as pct_of_total
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'delivered'
GROUP BY c.id, c.name
ORDER BY customer_revenue DESC
LIMIT 10;
```

</details>

### Problem 40: Order Size Distribution
Group orders by size (Small <$15, Medium $15-40, Large $40+) and show count and revenue for each.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    CASE 
        WHEN total_amount < 15 THEN 'Small (<$15)'
        WHEN total_amount <= 40 THEN 'Medium ($15-40)'
        ELSE 'Large ($40+)'
    END as order_size,
    COUNT(*) as order_count,
    ROUND(AVG(total_amount), 2) as avg_amount,
    SUM(total_amount) as total_revenue,
    ROUND(COUNT(*)::DECIMAL / SUM(COUNT(*)) OVER () * 100, 1) as pct_of_orders
FROM orders
WHERE status = 'delivered'
GROUP BY order_size
ORDER BY MIN(total_amount);
```

</details>

### Problem 41: Find Restaurants with Declining Revenue
Find restaurants whose revenue decreased for 2 consecutive months.

<details>
<summary>🔑 Answer</summary>

```sql
WITH monthly AS (
    SELECT 
        restaurant_id,
        DATE_TRUNC('month', order_date) as month,
        SUM(total_amount) as revenue
    FROM orders
    WHERE status = 'delivered'
    GROUP BY restaurant_id, DATE_TRUNC('month', order_date)
),
with_lag AS (
    SELECT 
        restaurant_id,
        month,
        revenue,
        LAG(revenue, 1) OVER (PARTITION BY restaurant_id ORDER BY month) as prev1,
        LAG(revenue, 2) OVER (PARTITION BY restaurant_id ORDER BY month) as prev2
    FROM monthly
)
SELECT 
    r.name,
    w.month,
    w.revenue,
    w.prev1,
    w.prev2,
    ROUND((w.revenue - w.prev1) / w.prev1 * 100, 1) as change_pct
FROM with_lag w
JOIN restaurants r ON w.restaurant_id = r.id
WHERE w.revenue < w.prev1 AND w.prev1 < w.prev2
ORDER BY change_pct;
```

</details>

### Problem 42: FIFO Order Processing
If a customer has multiple pending orders, process them in order date order. Show the processing sequence.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    customer_id,
    id as order_id,
    order_date,
    total_amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as processing_order
FROM orders
WHERE status = 'pending'
ORDER BY customer_id, processing_order;
```

</details>

### Problem 43: Price Change Detection
Using order_items, find items whose unit_price changed across orders. Show the item, old price, new price, and when it changed.

<details>
<summary>🔑 Answer</summary>

```sql
WITH price_history AS (
    SELECT DISTINCT
        item_name,
        unit_price,
        FIRST_VALUE(order_id) OVER (PARTITION BY item_name, unit_price ORDER BY order_id) as first_order,
        LAG(unit_price) OVER (PARTITION BY item_name ORDER BY MIN(order_id)) as prev_price
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    GROUP BY item_name, unit_price
)
SELECT item_name, prev_price as old_price, unit_price as new_price
FROM price_history
WHERE prev_price IS NOT NULL AND prev_price != unit_price
ORDER BY item_name;
```

</details>

### Problem 44: Cross-City Ordering
Find customers who order from restaurants in a DIFFERENT city than where they live.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT DISTINCT
    c.name as customer_name,
    c.city as customer_city,
    r.name as restaurant_name,
    r.city as restaurant_city
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN restaurants r ON o.restaurant_id = r.id
WHERE c.city != r.city
ORDER BY c.name;
```

</details>

### Problem 45: Recency-Frequency-Monetary (RFM) Analysis
Segment customers using RFM: Recency (days since last order), Frequency (total orders), Monetary (total spent).

<details>
<summary>🔑 Answer</summary>

```sql
WITH rfm AS (
    SELECT 
        c.id,
        c.name,
        CURRENT_DATE - MAX(o.order_date)::DATE as recency_days,
        COUNT(o.id) as frequency,
        SUM(o.total_amount) as monetary
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    WHERE o.status = 'delivered'
    GROUP BY c.id, c.name
)
SELECT 
    name,
    recency_days,
    frequency,
    monetary,
    NTILE(4) OVER (ORDER BY recency_days DESC) as recency_score,
    NTILE(4) OVER (ORDER BY frequency) as frequency_score,
    NTILE(4) OVER (ORDER BY monetary) as monetary_score,
    CASE 
        WHEN NTILE(4) OVER (ORDER BY recency_days DESC) >= 3 
             AND NTILE(4) OVER (ORDER BY frequency) >= 3 THEN 'Champions'
        WHEN NTILE(4) OVER (ORDER BY frequency) >= 3 THEN 'Loyal'
        WHEN NTILE(4) OVER (ORDER BY recency_days DESC) >= 3 THEN 'Recent'
        ELSE 'At Risk'
    END as segment
FROM rfm
ORDER BY monetary DESC;
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Problems 46-60

### Problem 46: Cumulative Distinct Customers
Show the running total of unique customers over time.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    DATE(order_date) as order_day,
    COUNT(DISTINCT customer_id) as daily_new_customers,
    SUM(COUNT(DISTINCT customer_id)) OVER (ORDER BY DATE(order_date)) as cumulative_customers
FROM orders
GROUP BY DATE(order_date)
ORDER BY order_day;
```

</details>

### Problem 47: Find Orders with All Required Items
Find orders that contain BOTH "Chicken Rice" AND "Kopi" (must have both).

<details>
<summary>🔑 Answer</summary>

```sql
SELECT order_id
FROM order_items
WHERE item_name IN ('Chicken Rice', 'Kopi')
GROUP BY order_id
HAVING COUNT(DISTINCT item_name) = 2;
```

</details>

### Problem 48: Revenue per Hour of Day
Show which hours of the day generate the most revenue.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    EXTRACT(HOUR FROM order_date) as hour_of_day,
    COUNT(*) as order_count,
    SUM(total_amount) as revenue,
    ROUND(AVG(total_amount), 2) as avg_order
FROM orders
WHERE status = 'delivered'
GROUP BY EXTRACT(HOUR FROM order_date)
ORDER BY revenue DESC;
```

</details>

### Problem 49: Weighted Average Rating
Calculate a weighted average restaurant rating where the weight is the number of orders. (Restaurants with more orders contribute more to the average.)

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    SUM(r.rating * COALESCE(oc.order_count, 0)) / 
    NULLIF(SUM(COALESCE(oc.order_count, 0)), 0) as weighted_avg_rating
FROM restaurants r
LEFT JOIN (
    SELECT restaurant_id, COUNT(*) as order_count
    FROM orders WHERE status = 'delivered'
    GROUP BY restaurant_id
) oc ON r.id = oc.restaurant_id;
```

</details>

### Problem 50: Moving Average Revenue (7-day)
Calculate a 7-day moving average of daily revenue.

<details>
<summary>🔑 Answer</summary>

```sql
WITH daily AS (
    SELECT DATE(order_date) as order_day, SUM(total_amount) as daily_revenue
    FROM orders WHERE status = 'delivered'
    GROUP BY DATE(order_date)
)
SELECT 
    order_day,
    daily_revenue,
    ROUND(AVG(daily_revenue) OVER (
        ORDER BY order_day 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) as moving_avg_7d
FROM daily
ORDER BY order_day;
```

</details>

### Problem 51: Find Restaurant-Specific Price Ranges
For each restaurant, find the cheapest and most expensive menu item (from order_items).

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    r.name as restaurant,
    MIN(oi.unit_price) as cheapest_item_price,
    MAX(oi.unit_price) as most_expensive_item_price,
    MAX(oi.unit_price) - MIN(oi.unit_price) as price_range
FROM order_items oi
JOIN orders o ON oi.order_id = o.id
JOIN restaurants r ON o.restaurant_id = r.id
GROUP BY r.id, r.name
ORDER BY price_range DESC;
```

</details>

### Problem 52: Session-Based Analysis
Group orders into "sessions" — if a customer places multiple orders within 1 hour, count as one session.

<details>
<summary>🔑 Answer</summary>

```sql
WITH with_prev AS (
    SELECT 
        *,
        LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_time
    FROM orders
    WHERE status = 'delivered'
),
flagged AS (
    SELECT 
        *,
        CASE WHEN prev_time IS NULL 
              OR EXTRACT(EPOCH FROM (order_date - prev_time)) > 3600
             THEN 1 ELSE 0 END as new_session
    FROM with_prev
),
sessions AS (
    SELECT 
        *,
        SUM(new_session) OVER (PARTITION BY customer_id ORDER BY order_date) as session_id
    FROM flagged
)
SELECT 
    customer_id,
    session_id,
    MIN(order_date) as session_start,
    MAX(order_date) as session_end,
    COUNT(*) as orders_in_session,
    SUM(total_amount) as session_total
FROM sessions
GROUP BY customer_id, session_id
ORDER BY session_total DESC;
```

</details>

### Problem 53: Pareto Analysis (80/20 Rule)
Show whether 20% of customers generate 80% of revenue.

<details>
<summary>🔑 Answer</summary>

```sql
WITH customer_revenue AS (
    SELECT 
        customer_id,
        SUM(total_amount) as revenue
    FROM orders
    WHERE status = 'delivered'
    GROUP BY customer_id
),
ranked AS (
    SELECT 
        customer_id,
        revenue,
        ROW_NUMBER() OVER (ORDER BY revenue DESC) as rn,
        SUM(revenue) OVER () as total_revenue,
        SUM(revenue) OVER (ORDER BY revenue DESC) as cumulative_revenue
    FROM customer_revenue
)
SELECT 
    COUNT(*) as total_customers,
    SUM(CASE WHEN cumulative_revenue <= total_revenue * 0.8 THEN 1 
             ELSE 0 END) as customers_for_80pct_revenue,
    ROUND(SUM(CASE WHEN cumulative_revenue <= total_revenue * 0.8 THEN 1 ELSE 0 END)::DECIMAL / 
          COUNT(*) * 100, 1) as pct_customers_for_80pct
FROM ranked;
```

</details>

### Problem 54: Department-Level Aggregation with Filtering
Find restaurants where the average order amount is above the global average, but only for restaurants with at least 10 orders.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    r.name,
    r.cuisine,
    COUNT(*) as order_count,
    ROUND(AVG(o.total_amount), 2) as avg_order,
    ROUND((SELECT AVG(total_amount) FROM orders WHERE status = 'delivered'), 2) as global_avg
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.id
WHERE o.status = 'delivered'
GROUP BY r.id, r.name, r.cuisine
HAVING COUNT(*) >= 10
   AND AVG(o.total_amount) > (SELECT AVG(total_amount) FROM orders WHERE status = 'delivered')
ORDER BY avg_order DESC;
```

</details>

### Problem 55: Hierarchical Data — Customers and Their Referrals
If customers table has a `referred_by` column (FK to customers.id), find each customer's referral chain (who referred them, who referred their referrer, etc.).

<details>
<summary>🔑 Answer</summary>

```sql
-- Using recursive CTE
WITH RECURSIVE referral_chain AS (
    -- Base: customers with no referrer
    SELECT id, name, referred_by, name::TEXT as chain, 1 as depth
    FROM customers
    WHERE referred_by IS NULL
    
    UNION ALL
    
    -- Recursive: join to referrer
    SELECT c.id, c.name, c.referred_by, 
           (rc.chain || ' → ' || c.name)::TEXT,
           rc.depth + 1
    FROM customers c
    JOIN referral_chain rc ON c.referred_by = rc.id
)
SELECT name, chain, depth
FROM referral_chain
ORDER BY chain;
```

</details>

### Problem 56: Percentage Change per Category
For each cuisine type, show the revenue change between last month and the month before.

<details>
<summary>🔑 Answer</summary>

```sql
WITH monthly AS (
    SELECT 
        r.cuisine,
        DATE_TRUNC('month', o.order_date) as month,
        SUM(o.total_amount) as revenue
    FROM orders o
    JOIN restaurants r ON o.restaurant_id = r.id
    WHERE o.status = 'delivered'
    GROUP BY r.cuisine, DATE_TRUNC('month', o.order_date)
)
SELECT 
    cuisine,
    month,
    revenue,
    LAG(revenue) OVER (PARTITION BY cuisine ORDER BY month) as prev_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (PARTITION BY cuisine ORDER BY month)) / 
        LAG(revenue) OVER (PARTITION BY cuisine ORDER BY month) * 100,
        1
    ) as change_pct
FROM monthly
ORDER BY cuisine, month;
```

</details>

### Problem 57: Find the "Most Consistent" Restaurant
Find the restaurant with the lowest standard deviation in daily revenue (most consistent earnings).

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    r.name,
    ROUND(AVG(daily_rev), 2) as avg_daily,
    ROUND(STDDEV(daily_rev), 2) as stddev_daily,
    ROUND(STDDEV(daily_rev) / NULLIF(AVG(daily_rev), 0) * 100, 1) as coefficient_of_variation
FROM (
    SELECT restaurant_id, DATE(order_date) as day, SUM(total_amount) as daily_rev
    FROM orders WHERE status = 'delivered'
    GROUP BY restaurant_id, DATE(order_date)
) daily
JOIN restaurants r ON daily.restaurant_id = r.id
GROUP BY r.id, r.name
HAVING COUNT(*) >= 7  -- At least 7 days of data
ORDER BY coefficient_of_variation
LIMIT 5;
```

</details>

### Problem 58: Upsert Simulation
Write a query that simulates an upsert: if a customer already exists (by name), update their city. Otherwise, insert a new record.

<details>
<summary>🔑 Answer</summary>

```sql
INSERT INTO customers (id, name, city, signup_date)
VALUES (999, 'New Customer', 'Singapore', CURRENT_DATE)
ON CONFLICT (id) DO UPDATE SET
    city = EXCLUDED.city;
```

</details>

### Problem 59: Find Time-Based Patterns
Show order volume by hour-of-day and day-of-week as a heatmap query.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    TO_CHAR(order_date, 'Day') as day_of_week,
    EXTRACT(HOUR FROM order_date) as hour,
    COUNT(*) as order_count
FROM orders
WHERE status = 'delivered'
GROUP BY TO_CHAR(order_date, 'Day'), EXTRACT(HOUR FROM order_date), EXTRACT(DOW FROM order_date)
ORDER BY EXTRACT(DOW FROM order_date), hour;
```

</details>

### Problem 60: Interview-Style: Explain the Query
Given this query, explain what it does and how to optimize it:

```sql
SELECT c.name, SUM(oi.quantity * oi.unit_price) as total
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN order_items oi ON o.id = oi.order_id
WHERE o.order_date >= '2026-01-01'
GROUP BY c.name
HAVING SUM(oi.quantity * oi.unit_price) > 100
ORDER BY total DESC;
```

<details>
<summary>🔑 Answer</summary>

**What it does:** Finds customers who spent more than $100 on items since Jan 2026, ordered by total spending.

**How it works:**
1. Joins customers → orders → order_items
2. Filters to orders from 2026 onwards
3. Calculates line-level totals (quantity × unit_price)
4. Groups by customer name
5. Filters groups with HAVING (> $100)
6. Orders by total descending

**Optimization:**
- Add index on `orders(customer_id, order_date)` — for the JOIN + WHERE
- Add index on `order_items(order_id)` — for the JOIN
- Consider materialized view if this query runs frequently
- Use `EXPLAIN ANALYZE` to check the actual execution plan
- If customers table is large, consider partitioning orders by date
</details>

---

## 🌙 Evening: Track Progress

### Scoring

| Problems | Level | Your Score |
|----------|-------|-----------|
| 31-40 | Medium | /10 |
| 41-50 | Medium-Hard | /10 |
| 51-60 | Hard | /10 |

**Total across both days: /60**

- 50+: Interview-ready for SQL
- 40-49: Good — practice weak areas
- 30-39: Needs more practice — re-do problems 31-45
- Below 30: Review Day 1-5 material and re-do these

### 📝 Today's Checklist

- [ ] Attempted all 30 medium-hard problems
- [ ] Solved at least 20 without hints
- [ ] Practiced window functions (ROW_NUMBER, RANK, LAG, LEAD, NTILE)
- [ ] Practiced recursive CTEs
- [ ] Practiced complex joins and subqueries
- [ ] Can explain query optimization concepts
- [ ] Ready for tomorrow: Assessment + Mock Interview

---

*Day 68 complete! 60 SQL problems across 2 days — you're getting sharp.* 💼🔥
