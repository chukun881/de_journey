# 📅 Day 85 — Thursday, 7 August 2026
# 💻 SQL Live Coding Practice: Basic-Intermediate (50 Problems, Timed)

---

## 🎯 Today's Goal

SQL is the #1 tested skill in DE interviews. Today you do 50 basic-intermediate problems under time pressure. Each problem: 2-3 minutes. This simulates real interview conditions where you write SQL live on a whiteboard or shared screen.

**Rules:**
- Time each problem (2-3 min target)
- Write SQL without looking at notes
- Check answer after each batch of 10
- Track your score

---

## ☀️ Morning Block (2 hours): Problems 1-25

### Batch 1: SELECT, WHERE, ORDER BY (Problems 1-10)

Using a fictional `orders` table:
```sql
-- orders: id, customer_id, restaurant_id, order_date, total_amount, status, city
-- statuses: 'pending', 'preparing', 'delivered', 'cancelled'
-- cities: 'Singapore', 'Kuala Lumpur', 'Penang', 'Johor Bahru'
```

**Q1.** Select all orders delivered in Singapore.

**Q2.** Find orders where total_amount > 50, sorted by amount descending.

**Q3.** Count total orders per status.

**Q4.** Find the top 5 highest-value orders.

**Q5.** Select distinct cities that have had cancelled orders.

**Q6.** Find orders placed in July 2026.

**Q7.** Count orders per city, only cities with > 100 orders.

**Q8.** Find the average order amount per city.

**Q9.** Select orders where status is NOT 'cancelled' and amount > 30.

**Q10.** Find the earliest and latest order dates.

<details>
<summary>🔑 Answers 1-10</summary>

```sql
-- A1
SELECT * FROM orders WHERE status = 'delivered' AND city = 'Singapore';

-- A2
SELECT * FROM orders WHERE total_amount > 50 ORDER BY total_amount DESC;

-- A3
SELECT status, COUNT(*) as order_count FROM orders GROUP BY status;

-- A4
SELECT * FROM orders ORDER BY total_amount DESC LIMIT 5;

-- A5
SELECT DISTINCT city FROM orders WHERE status = 'cancelled';

-- A6
SELECT * FROM orders WHERE order_date >= '2026-07-01' AND order_date < '2026-08-01';

-- A7
SELECT city, COUNT(*) as order_count FROM orders GROUP BY city HAVING COUNT(*) > 100;

-- A8
SELECT city, ROUND(AVG(total_amount), 2) as avg_amount FROM orders GROUP BY city;

-- A9
SELECT * FROM orders WHERE status != 'cancelled' AND total_amount > 30;

-- A10
SELECT MIN(order_date) as earliest, MAX(order_date) as latest FROM orders;
```
</details>

### Batch 2: JOINs (Problems 11-20)

Additional tables:
```sql
-- customers: id, name, email, city, signup_date
-- restaurants: id, name, cuisine, city, rating
-- order_items: id, order_id, menu_item, quantity, price
```

**Q11.** Show order id, customer name, and total amount for all orders.

**Q12.** Find all orders from customers in Singapore with their restaurant names.

**Q13.** List customers who have never placed an order.

**Q14.** For each restaurant, show the restaurant name and total revenue.

**Q15.** Find the most popular cuisine (most orders).

**Q16.** Show each customer's total spending.

**Q17.** Find restaurants with no orders in the last 30 days.

**Q18.** For each order, count the number of items.

**Q19.** Find customers who ordered from the same restaurant more than 3 times.

**Q20.** Show the top 3 restaurants by number of orders in Singapore.

<details>
<summary>🔑 Answers 11-20</summary>

```sql
-- A11
SELECT o.id, c.name, o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- A12
SELECT o.id, c.name, r.name as restaurant_name
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN restaurants r ON o.restaurant_id = r.id
WHERE c.city = 'Singapore';

-- A13
SELECT c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;

-- A14
SELECT r.name, SUM(o.total_amount) as revenue
FROM restaurants r
JOIN orders o ON r.id = o.restaurant_id
GROUP BY r.name;

-- A15
SELECT r.cuisine, COUNT(*) as order_count
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.id
GROUP BY r.cuisine
ORDER BY order_count DESC
LIMIT 1;

-- A16
SELECT c.name, SUM(o.total_amount) as total_spending
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;

-- A17
SELECT r.name
FROM restaurants r
LEFT JOIN orders o ON r.id = o.restaurant_id AND o.order_date >= CURRENT_DATE - 30
WHERE o.id IS NULL;

-- A18
SELECT o.id, COUNT(oi.id) as item_count
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id;

-- A19
SELECT c.name, r.name as restaurant, COUNT(*) as order_count
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN restaurants r ON o.restaurant_id = r.id
GROUP BY c.name, r.name
HAVING COUNT(*) > 3;

-- A20
SELECT r.name, COUNT(*) as order_count
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.id
WHERE r.city = 'Singapore'
GROUP BY r.name
ORDER BY order_count DESC
LIMIT 3;
```
</details>

### Batch 3: Subqueries + CTEs (Problems 21-25)

**Q21.** Find customers whose total spending is above the average.

**Q22.** Find the highest-value order for each customer.

**Q23.** Show orders that are above the average amount for their city.

**Q24.** Find the 2nd highest order amount overall.

**Q25.** For each restaurant, find the customer who spent the most.

<details>
<summary>🔑 Answers 21-25</summary>

```sql
-- A21
SELECT c.name, SUM(o.total_amount) as total
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.name
HAVING SUM(o.total_amount) > (
    SELECT AVG(total_amount) FROM orders
);

-- Better version using CTE:
WITH customer_totals AS (
    SELECT c.name, SUM(o.total_amount) as total
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    GROUP BY c.name
)
SELECT * FROM customer_totals
WHERE total > (SELECT AVG(total_amount) FROM orders);

-- A22
WITH ranked AS (
    SELECT o.customer_id, o.total_amount,
           ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.total_amount DESC) as rn
    FROM orders o
)
SELECT r.customer_id, r.total_amount
FROM ranked r
WHERE r.rn = 1;

-- A23
SELECT o.*
FROM orders o
WHERE o.total_amount > (
    SELECT AVG(o2.total_amount)
    FROM orders o2
    WHERE o2.city = o.city
);

-- A24
SELECT DISTINCT total_amount
FROM orders
ORDER BY total_amount DESC
OFFSET 1 LIMIT 1;

-- Or using DENSE_RANK:
WITH ranked AS (
    SELECT total_amount, DENSE_RANK() OVER (ORDER BY total_amount DESC) as dr
    FROM orders
)
SELECT total_amount FROM ranked WHERE dr = 2 LIMIT 1;

-- A25
WITH ranked AS (
    SELECT o.restaurant_id, o.customer_id, SUM(o.total_amount) as total,
           ROW_NUMBER() OVER (PARTITION BY o.restaurant_id ORDER BY SUM(o.total_amount) DESC) as rn
    FROM orders o
    GROUP BY o.restaurant_id, o.customer_id
)
SELECT r.restaurant_id, r.customer_id, r.total
FROM ranked r
WHERE r.rn = 1;
```
</details>

---

## 🌤️ Afternoon Block (2 hours): Problems 26-50

### Batch 4: Window Functions (Problems 26-35)

**Q26.** Add a running total of daily revenue.

**Q27.** Rank orders by amount within each city.

**Q28.** Find the difference between each order and the previous order (same customer).

**Q29.** Show each customer's first and last order date.

**Q30.** Calculate a 3-day moving average of daily revenue.

**Q31.** Find the top 2 orders per customer.

**Q32.** Show the percentage each order contributes to the customer's total.

**Q33.** Identify the 2nd order for each customer.

**Q34.** Calculate cumulative count of orders per day.

**Q35.** Find customers whose latest order amount is less than their first order.

<details>
<summary>🔑 Answers 26-35</summary>

```sql
-- A26
SELECT order_date, SUM(total_amount) as daily_revenue,
       SUM(SUM(total_amount)) OVER (ORDER BY order_date) as running_total
FROM orders
GROUP BY order_date
ORDER BY order_date;

-- A27
SELECT *, RANK() OVER (PARTITION BY city ORDER BY total_amount DESC) as city_rank
FROM orders;

-- A28
SELECT customer_id, order_date, total_amount,
       total_amount - LAG(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) as diff_from_prev
FROM orders;

-- A29
SELECT customer_id,
       MIN(order_date) OVER (PARTITION BY customer_id) as first_order,
       MAX(order_date) OVER (PARTITION BY customer_id) as last_order
FROM orders
GROUP BY customer_id;

-- Better:
SELECT customer_id, MIN(order_date) as first_order, MAX(order_date) as last_order
FROM orders
GROUP BY customer_id;

-- A30
SELECT order_date, SUM(total_amount) as daily_revenue,
       AVG(SUM(total_amount)) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_avg_3d
FROM orders
GROUP BY order_date
ORDER BY order_date;

-- A31
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total_amount DESC) as rn
    FROM orders
)
SELECT * FROM ranked WHERE rn <= 2;

-- A32
SELECT customer_id, id, total_amount,
       ROUND(total_amount * 100.0 / SUM(total_amount) OVER (PARTITION BY customer_id), 2) as pct_of_total
FROM orders;

-- A33
WITH numbered AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as order_num
    FROM orders
)
SELECT * FROM numbered WHERE order_num = 2;

-- A34
SELECT order_date, COUNT(*) as daily_count,
       SUM(COUNT(*)) OVER (ORDER BY order_date) as cumulative_count
FROM orders
GROUP BY order_date
ORDER BY order_date;

-- A35
WITH first_last AS (
    SELECT customer_id,
           FIRST_VALUE(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) as first_amt,
           FIRST_VALUE(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date DESC) as last_amt
    FROM orders
)
SELECT DISTINCT customer_id
FROM first_last
WHERE last_amt < first_amt;
```
</details>

### Batch 5: Mixed Problems (Problems 36-50)

**Q36.** Find the month with the highest revenue.

**Q37.** Calculate customer retention: what % of customers who ordered in January also ordered in February?

**Q38.** Find restaurants whose revenue dropped >20% compared to last month.

**Q39.** Create a histogram: group orders into buckets (0-20, 20-40, 40-60, 60-80, 80-100, 100+).

**Q40.** Find the median order amount.

**Q41.** Identify consecutive days where a customer ordered (streak detection).

**Q42.** Pivot: show monthly revenue per city (columns = cities).

**Q43.** Find the average time between orders per customer.

**Q44.** Flag orders that are outliers (> 3 standard deviations from mean).

**Q45.** Calculate month-over-month growth rate per city.

**Q46.** Find customers who ordered every month for the past 6 months.

**Q47.** Generate a cohort analysis: group customers by signup month, show retention.

**Q48.** Find the most ordered menu item across all restaurants.

**Q49.** Rank restaurants by revenue within their cuisine category.

**Q50.** Create a summary: for each city, show total orders, total revenue, avg order value, top restaurant, top customer.

<details>
<summary>🔑 Answers 36-50</summary>

```sql
-- A36
SELECT DATE_TRUNC('month', order_date) as month, SUM(total_amount) as revenue
FROM orders
GROUP BY month
ORDER BY revenue DESC LIMIT 1;

-- A37
WITH jan_customers AS (
    SELECT DISTINCT customer_id FROM orders
    WHERE order_date >= '2026-01-01' AND order_date < '2026-02-01'
),
feb_customers AS (
    SELECT DISTINCT customer_id FROM orders
    WHERE order_date >= '2026-02-01' AND order_date < '2026-03-01'
)
SELECT 
    COUNT(DISTINCT j.customer_id) as jan_count,
    COUNT(DISTINCT f.customer_id) as feb_returned,
    ROUND(100.0 * COUNT(DISTINCT f.customer_id) / COUNT(DISTINCT j.customer_id), 2) as retention_pct
FROM jan_customers j
LEFT JOIN feb_customers f ON j.customer_id = f.customer_id;

-- A38
WITH monthly AS (
    SELECT restaurant_id,
           DATE_TRUNC('month', order_date) as month,
           SUM(total_amount) as revenue
    FROM orders
    GROUP BY 1, 2
),
compared AS (
    SELECT restaurant_id, month, revenue,
           LAG(revenue) OVER (PARTITION BY restaurant_id ORDER BY month) as prev_revenue
    FROM monthly
)
SELECT restaurant_id, month, revenue, prev_revenue,
       ROUND(100.0 * (revenue - prev_revenue) / prev_revenue, 2) as pct_change
FROM compared
WHERE prev_revenue IS NOT NULL AND revenue < prev_revenue * 0.8;

-- A39
SELECT 
    CASE 
        WHEN total_amount BETWEEN 0 AND 20 THEN '0-20'
        WHEN total_amount BETWEEN 20.01 AND 40 THEN '20-40'
        WHEN total_amount BETWEEN 40.01 AND 60 THEN '40-60'
        WHEN total_amount BETWEEN 60.01 AND 80 THEN '60-80'
        WHEN total_amount BETWEEN 80.01 AND 100 THEN '80-100'
        ELSE '100+'
    END as bucket,
    COUNT(*) as order_count
FROM orders
GROUP BY bucket
ORDER BY bucket;

-- A40
WITH ordered AS (
    SELECT total_amount, ROW_NUMBER() OVER (ORDER BY total_amount) as rn,
           COUNT(*) OVER () as total_count
    FROM orders
)
SELECT AVG(total_amount) as median
FROM ordered
WHERE rn IN (FLOOR(total_count/2.0), CEIL(total_count/2.0));

-- Or simply: SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total_amount) FROM orders;

-- A41
WITH daily AS (
    SELECT DISTINCT customer_id, DATE(order_date) as order_date
),
numbered AS (
    SELECT customer_id, order_date,
           order_date - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)::int as grp
    FROM daily
)
SELECT customer_id, MIN(order_date) as streak_start, MAX(order_date) as streak_end,
       COUNT(*) as streak_length
FROM numbered
GROUP BY customer_id, grp
HAVING COUNT(*) > 1
ORDER BY streak_length DESC;

-- A42
SELECT DATE_TRUNC('month', order_date) as month,
       SUM(CASE WHEN city = 'Singapore' THEN total_amount ELSE 0 END) as sg_revenue,
       SUM(CASE WHEN city = 'Kuala Lumpur' THEN total_amount ELSE 0 END) as kl_revenue,
       SUM(CASE WHEN city = 'Penang' THEN total_amount ELSE 0 END) as pg_revenue
FROM orders
GROUP BY month
ORDER BY month;

-- A43
WITH order_gaps AS (
    SELECT customer_id, order_date,
           order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as gap_days
    FROM orders
)
SELECT customer_id, ROUND(AVG(gap_days), 1) as avg_days_between
FROM order_gaps
WHERE gap_days IS NOT NULL
GROUP BY customer_id;

-- A44
WITH stats AS (
    SELECT AVG(total_amount) as mean, STDDEV(total_amount) as sd FROM orders
)
SELECT o.*, s.mean, s.sd
FROM orders o
CROSS JOIN stats s
WHERE ABS(o.total_amount - s.mean) > 3 * s.sd;

-- A45
WITH monthly AS (
    SELECT city, DATE_TRUNC('month', order_date) as month, SUM(total_amount) as revenue
    FROM orders
    GROUP BY 1, 2
),
with_prev AS (
    SELECT *, LAG(revenue) OVER (PARTITION BY city ORDER BY month) as prev_revenue
    FROM monthly
)
SELECT city, month, revenue, prev_revenue,
       ROUND(100.0 * (revenue - prev_revenue) / NULLIF(prev_revenue, 0), 2) as growth_pct
FROM with_prev
ORDER BY city, month;

-- A46
WITH monthly_cust AS (
    SELECT DISTINCT customer_id, DATE_TRUNC('month', order_date) as month
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '6 months'
)
SELECT customer_id
FROM monthly_cust
GROUP BY customer_id
HAVING COUNT(DISTINCT month) = 6;

-- A47
WITH cohorts AS (
    SELECT c.id as customer_id, DATE_TRUNC('month', c.signup_date) as cohort_month
    FROM customers c
),
orders_month AS (
    SELECT DISTINCT customer_id, DATE_TRUNC('month', order_date) as order_month
    FROM orders
),
cohort_data AS (
    SELECT c.cohort_month, o.order_month,
           EXTRACT(YEAR FROM o.order_month)*12 + EXTRACT(MONTH FROM o.order_month) -
           EXTRACT(YEAR FROM c.cohort_month)*12 - EXTRACT(MONTH FROM c.cohort_month) as month_n
    FROM cohorts c
    JOIN orders_month o ON c.customer_id = o.customer_id
)
SELECT cohort_month, month_n, COUNT(DISTINCT customer_id) as active_customers
FROM cohort_data
GROUP BY cohort_month, month_n
ORDER BY cohort_month, month_n;

-- A48
SELECT oi.menu_item, SUM(oi.quantity) as total_ordered
FROM order_items oi
GROUP BY oi.menu_item
ORDER BY total_ordered DESC
LIMIT 1;

-- A49
SELECT r.name, r.cuisine,
       SUM(o.total_amount) as revenue,
       RANK() OVER (PARTITION BY r.cuisine ORDER BY SUM(o.total_amount) DESC) as cuisine_rank
FROM restaurants r
JOIN orders o ON r.id = o.restaurant_id
GROUP BY r.id, r.name, r.cuisine;

-- A50
WITH city_stats AS (
    SELECT o.city,
           COUNT(*) as total_orders,
           SUM(o.total_amount) as total_revenue,
           ROUND(AVG(o.total_amount), 2) as avg_order_value
    FROM orders o
    GROUP BY o.city
),
top_rest AS (
    SELECT o.city, r.name as top_restaurant
    FROM orders o
    JOIN restaurants r ON o.restaurant_id = r.id
    GROUP BY o.city, r.name
    ORDER BY COUNT(*) DESC
),
top_cust AS (
    SELECT o.city, c.name as top_customer
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    GROUP BY o.city, c.name
    ORDER BY SUM(o.total_amount) DESC
)
SELECT cs.*, tr.top_restaurant, tc.top_customer
FROM city_stats cs
LEFT JOIN (SELECT DISTINCT ON (city) city, top_restaurant FROM top_rest) tr ON cs.city = tr.city
LEFT JOIN (SELECT DISTINCT ON (city) city, top_customer FROM top_cust) tc ON cs.city = tc.city;
```
</details>

---

## 🌙 Evening (1 hour): Score + Review

### Scoring

| Score | Assessment |
|-------|-----------|
| 45-50 | 🔥 SQL interview-ready |
| 35-44 | ✅ Solid, practice weak spots |
| 25-34 | ⚠️ Need more practice |
| <25 | 🔴 Review Week 1 + 4 material |

### 📝 Today's Checklist

- [ ] Completed all 50 problems (timed)
- [ ] Score: __/50
- [ ] Identified weak areas (JOINs? Window functions? CTEs?)
- [ ] Reviewed answers for mistakes
- [ ] Saved difficult problems for re-practice

---

*Day 85 complete! Tomorrow: SQL medium-advanced problems.* 💻
