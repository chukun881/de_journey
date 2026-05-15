# 📅 Day 67 — Sunday, 20 July 2026
# 💼 SQL Interview Practice: 30 Live Coding Problems

---

## 🎯 Today's Goal

SQL is the #1 skill tested in data engineering interviews. Employers will put you in front of a screen and say "write a query that does X." Today you practice 30 real interview problems, timed, no notes.

**Format:** Each problem has a scenario, expected output, and a hidden answer. Try to solve each in 3-5 minutes before looking at the answer.

---

## ☀️ Morning Block (2 hours): Basic-Intermediate SQL (Problems 1-15)

### Setup: The MakanExpress Schema

```sql
-- Use this schema for ALL problems today
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    city VARCHAR(50),
    signup_date DATE
);

CREATE TABLE restaurants (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    cuisine VARCHAR(50),
    city VARCHAR(50),
    rating DECIMAL(2,1),
    avg_delivery_min INT
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    restaurant_id INT REFERENCES restaurants(id),
    order_date TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    delivery_city VARCHAR(50)
);

CREATE TABLE order_items (
    id INT PRIMARY KEY,
    order_id INT REFERENCES orders(id),
    item_name VARCHAR(100),
    quantity INT,
    unit_price DECIMAL(10,2)
);
```

---

### Problem 1: Top Customers by Spending
Find the top 5 customers by total amount spent. Show customer name and total_spent.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT c.name, SUM(o.total_amount) as total_spent
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'delivered'
GROUP BY c.id, c.name
ORDER BY total_spent DESC
LIMIT 5;
```

</details>

### Problem 2: Revenue by City
Show total revenue per delivery city, ordered by highest revenue first.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT delivery_city, SUM(total_amount) as revenue
FROM orders
WHERE status = 'delivered'
GROUP BY delivery_city
ORDER BY revenue DESC;
```

</details>

### Problem 3: Orders per Customer
Count how many orders each customer has placed. Only include customers with more than 3 orders.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT c.name, COUNT(o.id) as order_count
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
HAVING COUNT(o.id) > 3
ORDER BY order_count DESC;
```

</details>

### Problem 4: Average Order Value by Cuisine
What is the average order value for each cuisine type?

<details>
<summary>🔑 Answer</summary>

```sql
SELECT r.cuisine, ROUND(AVG(o.total_amount), 2) as avg_order_value
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.id
WHERE o.status = 'delivered'
GROUP BY r.cuisine
ORDER BY avg_order_value DESC;
```

</details>

### Problem 5: Customers Who Never Ordered
Find all customers who have never placed an order.

<details>
<summary>🔑 Answer</summary>

```sql
-- Method 1: LEFT JOIN
SELECT c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;

-- Method 2: NOT EXISTS
SELECT name FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- Method 3: NOT IN
SELECT name FROM customers
WHERE id NOT IN (SELECT DISTINCT customer_id FROM orders WHERE customer_id IS NOT NULL);
```

</details>

### Problem 6: Restaurant with Most Cancelled Orders
Which restaurant has the most cancelled orders? Show restaurant name and cancel count.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT r.name, COUNT(*) as cancelled_orders
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.id
WHERE o.status = 'cancelled'
GROUP BY r.id, r.name
ORDER BY cancelled_orders DESC
LIMIT 1;
```

</details>

### Problem 7: Daily Revenue Trend
Show daily revenue for the last 30 days. Include the date and total revenue.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT DATE(order_date) as order_day,
       SUM(total_amount) as daily_revenue
FROM orders
WHERE status = 'delivered'
  AND order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE(order_date)
ORDER BY order_day;
```

</details>

### Problem 8: Revenue Comparison (This Month vs Last Month)
Compare this month's revenue to last month's revenue.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    DATE_TRUNC('month', order_date) as month,
    SUM(total_amount) as revenue
FROM orders
WHERE status = 'delivered'
  AND order_date >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

</details>

### Problem 9: Most Popular Item
What is the most ordered item across all orders? Show item name and total quantity sold.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT item_name, SUM(quantity) as total_sold
FROM order_items
GROUP BY item_name
ORDER BY total_sold DESC
LIMIT 1;
```

</details>

### Problem 10: Customers Who Ordered from Multiple Cities
Find customers who have received deliveries in more than one city.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT c.name, COUNT(DISTINCT o.delivery_city) as city_count
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
HAVING COUNT(DISTINCT o.delivery_city) > 1
ORDER BY city_count DESC;
```

</details>

### Problem 11: Day of Week Analysis
Which day of the week has the most orders?

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    TO_CHAR(order_date, 'Day') as day_of_week,
    COUNT(*) as order_count,
    ROUND(AVG(total_amount), 2) as avg_amount
FROM orders
GROUP BY TO_CHAR(order_date, 'Day'), EXTRACT(DOW FROM order_date)
ORDER BY EXTRACT(DOW FROM order_date);
```

</details>

### Problem 12: Restaurant Rating vs Order Volume
Show each restaurant's rating and their total number of delivered orders. Are higher-rated restaurants more popular?

<details>
<summary>🔑 Answer</summary>

```sql
SELECT r.name, r.rating, COUNT(o.id) as order_count
FROM restaurants r
LEFT JOIN orders o ON r.id = o.restaurant_id AND o.status = 'delivered'
GROUP BY r.id, r.name, r.rating
ORDER BY order_count DESC;
```

</details>

### Problem 13: Repeat Customers
What percentage of customers have placed more than one order?

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    COUNT(CASE WHEN order_count > 1 THEN 1 END) as repeat_customers,
    COUNT(*) as total_customers,
    ROUND(
        COUNT(CASE WHEN order_count > 1 THEN 1 END)::DECIMAL / COUNT(*) * 100, 
        1
    ) as repeat_rate_pct
FROM (
    SELECT customer_id, COUNT(*) as order_count
    FROM orders
    WHERE status = 'delivered'
    GROUP BY customer_id
) sub;
```

</details>

### Problem 14: Orders with Above-Average Value
Find all orders where the total_amount is above the overall average order value.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT o.id, o.total_amount, c.name as customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.total_amount > (SELECT AVG(total_amount) FROM orders WHERE status = 'delivered')
  AND o.status = 'delivered'
ORDER BY o.total_amount DESC;
```

</details>

### Problem 15: Delivery Performance by Restaurant
For each restaurant, calculate: average delivery time, percentage of orders delivered (vs cancelled), and total orders.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    r.name,
    r.avg_delivery_min,
    COUNT(*) as total_orders,
    SUM(CASE WHEN o.status = 'delivered' THEN 1 ELSE 0 END) as delivered,
    SUM(CASE WHEN o.status = 'cancelled' THEN 1 ELSE 0 END) as cancelled,
    ROUND(
        SUM(CASE WHEN o.status = 'delivered' THEN 1 ELSE 0 END)::DECIMAL / COUNT(*) * 100,
        1
    ) as delivery_rate_pct
FROM restaurants r
LEFT JOIN orders o ON r.id = o.restaurant_id
GROUP BY r.id, r.name, r.avg_delivery_min
ORDER BY delivery_rate_pct DESC;
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Intermediate-Advanced SQL (Problems 16-30)

### Problem 16: Rank Customers by Monthly Spending
Rank customers by their spending each month. Use RANK() window function.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    DATE_TRUNC('month', o.order_date) as month,
    c.name,
    SUM(o.total_amount) as monthly_spent,
    RANK() OVER (PARTITION BY DATE_TRUNC('month', o.order_date) 
                 ORDER BY SUM(o.total_amount) DESC) as spending_rank
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'delivered'
GROUP BY DATE_TRUNC('month', o.order_date), c.id, c.name
ORDER BY month, spending_rank;
```

</details>

### Problem 17: Consecutive Order Days
Find customers who placed orders on 3 or more consecutive days.

<details>
<summary>🔑 Answer</summary>

```sql
WITH daily_orders AS (
    SELECT DISTINCT customer_id, DATE(order_date) as order_day
    FROM orders
),
numbered AS (
    SELECT 
        customer_id,
        order_day,
        order_day - (ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_day))::INT as grp
    FROM daily_orders
),
streaks AS (
    SELECT customer_id, grp, COUNT(*) as streak_length
    FROM numbered
    GROUP BY customer_id, grp
    HAVING COUNT(*) >= 3
)
SELECT DISTINCT c.name, MAX(s.streak_length) as longest_streak
FROM streaks s
JOIN customers c ON s.customer_id = c.id
GROUP BY c.id, c.name;
```

</details>

### Problem 18: Month-over-Month Revenue Growth
Calculate the month-over-month revenue growth rate.

<details>
<summary>🔑 Answer</summary>

```sql
WITH monthly_revenue AS (
    SELECT DATE_TRUNC('month', order_date) as month,
           SUM(total_amount) as revenue
    FROM orders
    WHERE status = 'delivered'
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT 
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) / 
        LAG(revenue) OVER (ORDER BY month) * 100, 
        1
    ) as growth_rate_pct
FROM monthly_revenue
ORDER BY month;
```

</details>

### Problem 19: Customer's First and Last Order
For each customer, show their first and last order date, and the number of days between them.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    c.name,
    MIN(o.order_date) as first_order,
    MAX(o.order_date) as last_order,
    MAX(o.order_date)::DATE - MIN(o.order_date)::DATE as days_between,
    COUNT(o.id) as total_orders
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'delivered'
GROUP BY c.id, c.name
ORDER BY days_between DESC;
```

</details>

### Problem 20: Top Item per Restaurant
Find the most popular item for each restaurant (by quantity sold).

<details>
<summary>🔑 Answer</summary>

```sql
WITH item_sales AS (
    SELECT 
        o.restaurant_id,
        oi.item_name,
        SUM(oi.quantity) as total_qty,
        RANK() OVER (PARTITION BY o.restaurant_id ORDER BY SUM(oi.quantity) DESC) as rn
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    GROUP BY o.restaurant_id, oi.item_name
)
SELECT r.name as restaurant, i.item_name, i.total_qty
FROM item_sales i
JOIN restaurants r ON i.restaurant_id = r.id
WHERE i.rn = 1;
```

</details>

### Problem 21: Running Total of Revenue
Calculate a running total of daily revenue.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    DATE(order_date) as order_day,
    SUM(total_amount) as daily_revenue,
    SUM(SUM(total_amount)) OVER (ORDER BY DATE(order_date)) as running_total
FROM orders
WHERE status = 'delivered'
GROUP BY DATE(order_date)
ORDER BY order_day;
```

</details>

### Problem 22: Customer Retention (Cohort Analysis)
Show how many customers from each signup month are still active (placed an order in the last 30 days).

<details>
<summary>🔑 Answer</summary>

```sql
WITH cohort AS (
    SELECT 
        c.id,
        DATE_TRUNC('month', c.signup_date) as signup_month
    FROM customers c
),
activity AS (
    SELECT 
        customer_id,
        MAX(order_date) as last_order
    FROM orders
    WHERE status = 'delivered'
    GROUP BY customer_id
)
SELECT 
    cohort.signup_month,
    COUNT(DISTINCT cohort.id) as total_signups,
    COUNT(DISTINCT CASE WHEN activity.last_order >= CURRENT_DATE - INTERVAL '30 days' 
                        THEN cohort.id END) as active_last_30d,
    ROUND(
        COUNT(DISTINCT CASE WHEN activity.last_order >= CURRENT_DATE - INTERVAL '30 days' 
                           THEN cohort.id END)::DECIMAL / 
        COUNT(DISTINCT cohort.id) * 100, 1
    ) as retention_rate_pct
FROM cohort
LEFT JOIN activity ON cohort.id = activity.customer_id
GROUP BY cohort.signup_month
ORDER BY cohort.signup_month;
```

</details>

### Problem 23: Find Duplicate Orders
Find orders that have the same customer_id, restaurant_id, and total_amount within 5 minutes of each other (possible duplicates).

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    o1.id as order1_id,
    o2.id as order2_id,
    o1.customer_id,
    o1.restaurant_id,
    o1.total_amount,
    o1.order_date as order1_time,
    o2.order_date as order2_time
FROM orders o1
JOIN orders o2 ON o1.customer_id = o2.customer_id
    AND o1.restaurant_id = o2.restaurant_id
    AND o1.total_amount = o2.total_amount
    AND o1.id < o2.id
    AND ABS(EXTRACT(EPOCH FROM (o2.order_date - o1.order_date))) < 300;
```

</details>

### Problem 24: Revenue Share per Cuisine
Calculate what percentage of total revenue each cuisine type contributes.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    r.cuisine,
    SUM(o.total_amount) as cuisine_revenue,
    ROUND(
        SUM(o.total_amount) / SUM(SUM(o.total_amount)) OVER () * 100,
        2
    ) as revenue_share_pct
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.id
WHERE o.status = 'delivered'
GROUP BY r.cuisine
ORDER BY cuisine_revenue DESC;
```

</details>

### Problem 25: Orders with Items Exceeding Order Total
Find orders where the sum of (quantity × unit_price) from order_items doesn't match the order's total_amount.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    o.id as order_id,
    o.total_amount as order_total,
    SUM(oi.quantity * oi.unit_price) as items_total,
    o.total_amount - SUM(oi.quantity * oi.unit_price) as discrepancy
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id, o.total_amount
HAVING ABS(o.total_amount - SUM(oi.quantity * oi.unit_price)) > 0.01
ORDER BY ABS(discrepancy) DESC;
```

</details>

### Problem 26: Customer Lifetime Value Segments
Segment customers into High/Medium/Low value based on total spending:
- High: > $500
- Medium: $100-$500
- Low: < $100

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    CASE 
        WHEN total_spent > 500 THEN 'High'
        WHEN total_spent >= 100 THEN 'Medium'
        ELSE 'Low'
    END as segment,
    COUNT(*) as customer_count,
    ROUND(AVG(total_spent), 2) as avg_spent,
    ROUND(MIN(total_spent), 2) as min_spent,
    ROUND(MAX(total_spent), 2) as max_spent
FROM (
    SELECT customer_id, SUM(total_amount) as total_spent
    FROM orders
    WHERE status = 'delivered'
    GROUP BY customer_id
) sub
GROUP BY segment
ORDER BY avg_spent DESC;
```

</details>

### Problem 27: Find the Nth Highest Spending Customer
Find the 3rd highest spending customer overall.

<details>
<summary>🔑 Answer</summary>

```sql
-- Method 1: Using DENSE_RANK
WITH ranked AS (
    SELECT c.name, SUM(o.total_amount) as total_spent,
           DENSE_RANK() OVER (ORDER BY SUM(o.total_amount) DESC) as rnk
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    WHERE o.status = 'delivered'
    GROUP BY c.id, c.name
)
SELECT name, total_spent FROM ranked WHERE rnk = 3;

-- Method 2: Using OFFSET (PostgreSQL specific)
SELECT c.name, SUM(o.total_amount) as total_spent
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'delivered'
GROUP BY c.id, c.name
ORDER BY total_spent DESC
LIMIT 1 OFFSET 2;  -- Skip first 2, take next
```

</details>

### Problem 28: Self-Join — Customers Who Share a City
Find pairs of customers who live in the same city and have ordered from the same restaurant.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT DISTINCT
    c1.name as customer_1,
    c2.name as customer_2,
    c1.city,
    r.name as restaurant
FROM orders o1
JOIN orders o2 ON o1.restaurant_id = o2.restaurant_id
    AND o1.customer_id < o2.customer_id
JOIN customers c1 ON o1.customer_id = c1.id
JOIN customers c2 ON o2.customer_id = c2.id
JOIN restaurants r ON o1.restaurant_id = r.id
WHERE c1.city = c2.city;
```

</details>

### Problem 29: Pivot — Revenue by City per Month
Show monthly revenue with cities as columns (pivot/crosstab).

<details>
<summary>🔑 Answer</summary>

```sql
SELECT 
    DATE_TRUNC('month', order_date) as month,
    SUM(CASE WHEN delivery_city = 'Singapore' THEN total_amount ELSE 0 END) as singapore,
    SUM(CASE WHEN delivery_city = 'Kuala Lumpur' THEN total_amount ELSE 0 END) as kl,
    SUM(CASE WHEN delivery_city = 'Penang' THEN total_amount ELSE 0 END) as penang,
    SUM(CASE WHEN delivery_city = 'Johor Bahru' THEN total_amount ELSE 0 END) as jb,
    SUM(total_amount) as total
FROM orders
WHERE status = 'delivered'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

</details>

### Problem 30: Complex — Restaurant Performance Score
Create a composite score for each restaurant: 40% rating, 30% order volume, 30% revenue. Rank them.

<details>
<summary>🔑 Answer</summary>

```sql
WITH restaurant_stats AS (
    SELECT 
        r.id,
        r.name,
        r.rating,
        COALESCE(COUNT(o.id), 0) as order_count,
        COALESCE(SUM(o.total_amount), 0) as total_revenue
    FROM restaurants r
    LEFT JOIN orders o ON r.id = o.restaurant_id AND o.status = 'delivered'
    GROUP BY r.id, r.name, r.rating
),
normalized AS (
    SELECT 
        name,
        rating,
        order_count,
        total_revenue,
        -- Normalize each metric to 0-100 scale
        (rating / 5.0 * 100) as rating_score,
        (order_count::DECIMAL / MAX(order_count) OVER () * 100) as volume_score,
        (total_revenue / NULLIF(MAX(total_revenue) OVER (), 0) * 100) as revenue_score
    FROM restaurant_stats
)
SELECT 
    name,
    rating,
    order_count,
    total_revenue,
    ROUND(
        rating_score * 0.4 + volume_score * 0.3 + revenue_score * 0.3,
        1
    ) as composite_score,
    RANK() OVER (ORDER BY 
        rating_score * 0.4 + volume_score * 0.3 + revenue_score * 0.3 DESC
    ) as ranking
FROM normalized
ORDER BY composite_score DESC;
```

</details>

---

## 🌙 Evening: Review + Track Progress

### Score Yourself

| Problems | Level | Target Time |
|----------|-------|-------------|
| 1-10 | Basic | 2-3 min each |
| 11-20 | Intermediate | 3-5 min each |
| 21-30 | Advanced | 5-10 min each |

**If you solved < 20 without hints:** You're in good shape. Review the ones you missed.
**If you solved 20-25:** Strong — focus on the advanced window function problems.
**If you solved 25+:** Excellent — you're interview-ready for SQL.

### 📝 Today's Checklist

- [ ] Attempted all 30 problems
- [ ] Solved at least 20 without looking at answers
- [ ] Reviewed all answers for problems I missed
- [ ] Practiced writing queries (not just reading them)
- [ ] Identified weak areas (JOINs? Window functions? Subqueries?)
- [ ] Ready for tomorrow: More SQL interview practice (medium-hard)

---

*Day 67 complete! 30 SQL problems down. Tomorrow: 30 more, harder.* 💼💪
