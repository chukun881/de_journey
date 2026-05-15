# 📅 Day 26 — Monday, 9 June 2026
# 🪟 Advanced Window Functions & Analytics SQL

---

## 🎯 Today's Goal

You learned basic window functions in Week 1. Today we go **advanced** — the kind of window functions that solve real business problems in data engineering. By the end, you'll write window functions that most junior devs can't.

**Why this matters:** Window functions are the #1 SQL skill that separates beginners from intermediates. They appear in almost every data engineering interview and every real analytics query.

---

## ☀️ Morning Block (2 hours): Window Function Mastery

### Concept 1: Window Function Refresher

```sql
-- Syntax:
FUNCTION_NAME() OVER (
    [PARTITION BY column]
    [ORDER BY column [ASC|DESC]]
    [frame_clause]
)

-- The "window" = a set of rows related to the current row
-- PARTITION BY = define the group
-- ORDER BY = define the order within the group
-- Frame = define which rows in the partition to use
```

### Concept 2: Frame Clauses — The Secret Power

```sql
-- ROWS vs RANGE (critical difference!)

-- Running total: cumulative sum from start to current row
SELECT 
    customer_id,
    order_date,
    total_amount,
    SUM(total_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM orders;

-- 3-row moving average
SELECT 
    order_date,
    total_amount,
    AVG(total_amount) OVER (
        ORDER BY order_date 
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS moving_avg_3
FROM orders;

-- Year-to-date total (RANGE uses ORDER BY value, not row position)
SELECT 
    order_date,
    total_amount,
    SUM(total_amount) OVER (
        ORDER BY order_date
        RANGE BETWEEN INTERVAL '1 year' PRECEDING AND CURRENT ROW
    ) AS ytd_total
FROM orders;

-- All common frame options:
-- ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- from start to here
-- ROWS BETWEEN 7 PRECEDING AND CURRENT ROW           -- last 7 rows
-- ROWS BETWEEN 7 PRECEDING AND 7 FOLLOWING           -- ±7 rows
-- ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING   -- from here to end
-- RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW  -- last 7 days by value
```

**ROWS vs RANGE:**
- `ROWS` = physical row count (fast, predictable)
- `RANGE` = logical value range (handles ties, date arithmetic)

### Concept 3: Advanced Ranking Functions

```sql
-- ROW_NUMBER: strict sequential, no ties
-- RANK: ties get same rank, gaps after ties (1,2,2,4)
-- DENSE_RANK: ties get same rank, no gaps (1,2,2,3)
-- PERCENT_RANK: rank as percentage (0 to 1)
-- NTILE(n): divide into n equal groups

-- Find top 3 restaurants per city by revenue
SELECT * FROM (
    SELECT 
        r.city,
        r.restaurant_name,
        SUM(o.total_amount) AS total_revenue,
        RANK() OVER (PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) AS city_rank,
        DENSE_RANK() OVER (PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) AS city_drank,
        PERCENT_RANK() OVER (PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) AS pct_rank,
        NTILE(4) OVER (PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) AS revenue_quartile
    FROM orders o
    JOIN restaurants r ON o.restaurant_id = r.restaurant_id
    WHERE o.status = 'completed'
    GROUP BY r.city, r.restaurant_name
) ranked
WHERE city_rank <= 3;

-- FIRST_VALUE / LAST_VALUE: get value from first/last row in window
SELECT 
    customer_id,
    order_date,
    total_amount,
    FIRST_VALUE(order_date) OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_order_date,
    LAST_VALUE(order_date) OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_order_date
FROM orders;

-- NTH_VALUE: get value from Nth row in window
SELECT 
    customer_id,
    order_date,
    total_amount,
    NTH_VALUE(total_amount, 2) OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS second_order_amount
FROM orders;
```

### Concept 4: LAG, LEAD, and Time-Series Analysis

```sql
-- LAG: look back N rows
-- LEAD: look forward N rows

-- Month-over-month revenue growth
WITH monthly AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        SUM(total_amount) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT 
    TO_CHAR(month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS revenue_change,
    ROUND(
        (revenue - LAG(revenue, 1) OVER (ORDER BY month)) / 
        LAG(revenue, 1) OVER (ORDER BY month) * 100, 2
    ) AS pct_growth
FROM monthly
ORDER BY month;

-- Customer retention: days between consecutive orders
SELECT 
    customer_id,
    order_date,
    LAG(order_date, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order,
    order_date - LAG(order_date, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS days_between
FROM orders
WHERE status = 'completed'
ORDER BY customer_id, order_date;

-- Same-period comparison: this month vs same month last year
WITH monthly AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        SUM(total_amount) AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT 
    TO_CHAR(month, 'YYYY-MM') AS month,
    revenue AS current,
    LAG(revenue, 12) OVER (ORDER BY month) AS same_month_last_year,
    ROUND(
        (revenue - LAG(revenue, 12) OVER (ORDER BY month)) / 
        NULLIF(LAG(revenue, 12) OVER (ORDER BY month), 0) * 100, 2
    ) AS yoy_growth_pct
FROM monthly
ORDER BY month;

-- Detect churn: customer hasn't ordered in 90+ days
SELECT DISTINCT customer_id
FROM (
    SELECT 
        customer_id,
        order_date,
        LEAD(order_date, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order,
        LEAD(order_date, 1) OVER (PARTITION BY customer_id ORDER BY order_date) - order_date AS gap_days
    FROM orders
    WHERE status = 'completed'
) gaps
WHERE gap_days > 90 OR (next_order IS NULL AND CURRENT_DATE - order_date > 90);
```

### Concept 5: Conditional Aggregation with Window Functions

```sql
-- Running count of completed vs cancelled orders per customer
SELECT 
    customer_id,
    order_date,
    status,
    COUNT(*) FILTER (WHERE status = 'completed') OVER (
        PARTITION BY customer_id ORDER BY order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_completed,
    COUNT(*) FILTER (WHERE status = 'cancelled') OVER (
        PARTITION BY customer_id ORDER BY order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_cancelled
FROM orders;

-- Cohort analysis: running retention by signup month
WITH user_cohorts AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', signup_date) AS cohort_month
    FROM customers
),
order_months AS (
    SELECT DISTINCT
        o.customer_id,
        DATE_TRUNC('month', o.order_date) AS order_month
    FROM orders o
    WHERE o.status = 'completed'
),
cohort_data AS (
    SELECT 
        uc.cohort_month,
        om.order_month,
        EXTRACT(YEAR FROM om.order_month) * 12 + EXTRACT(MONTH FROM om.order_month) -
        EXTRACT(YEAR FROM uc.cohort_month) * 12 - EXTRACT(MONTH FROM uc.cohort_month) AS month_offset,
        COUNT(DISTINCT uc.customer_id) AS active_users
    FROM user_cohorts uc
    JOIN order_months om ON uc.customer_id = om.customer_id
    GROUP BY uc.cohort_month, om.order_month
),
cohort_sizes AS (
    SELECT cohort_month, COUNT(DISTINCT customer_id) AS cohort_size
    FROM user_cohorts
    GROUP BY cohort_month
)
SELECT 
    TO_CHAR(cd.cohort_month, 'YYYY-MM') AS cohort,
    cs.cohort_size,
    cd.month_offset,
    cd.active_users,
    ROUND(cd.active_users::DECIMAL / cs.cohort_size * 100, 1) AS retention_pct
FROM cohort_data cd
JOIN cohort_sizes cs ON cd.cohort_month = cs.cohort_month
ORDER BY cd.cohort_month, cd.month_offset;
```

---

### 🏋️ Morning Exercises (20 questions)

**Use the GrabFood schema from Day 22.**

1. Write a query to calculate the **running total of revenue** per restaurant, ordered by date. Show order_date, daily_revenue, and running_total.

2. Calculate a **7-day rolling average** of daily revenue across ALL restaurants. Use `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`.

3. Find the **top 2 customers per city** by total spending. Handle ties with RANK(). Show city, customer_name, total_spent, rank.

4. For each order, show the **previous order date** for the same customer and the **days between** the two orders. Filter to show only gaps > 7 days.

5. Calculate **month-over-month revenue growth** per restaurant. Show month, revenue, prev_month_revenue, and growth percentage.

6. Write a query using NTILE(4) to divide all customers into **4 quartiles by total spending**. Label them 'Q1-Top', 'Q2', 'Q3', 'Q4-Bottom'.

7. Find the **first and last order** for each customer using FIRST_VALUE and LAST_VALUE. Show customer_name, first_order_date, last_order_date, and customer_lifetime (days between).

8. Calculate the **year-over-year growth** for monthly revenue. Compare each month to the same month last year using LAG with offset 12.

9. Write a query to find **consecutive cancelled orders** — customers who have 2+ cancellations in a row (no successful orders in between).

10. For each restaurant, calculate the **cumulative average order value** over time. Show how the average changes with each new order.

11. Find **restaurants whose revenue dropped more than 20%** compared to the previous month. Use LAG to compare.

12. Calculate the **median order value** per restaurant using PERCENT_RANK() or NTILE(). (PostgreSQL has `PERCENTILE_CONT` but practice the window approach.)

13. Write a query to identify **new vs returning customers** per month. A customer is "new" in their first month, "returning" after that.

14. Calculate the **30-day repeat purchase rate**: what percentage of customers who ordered in a given month also ordered within the next 30 days?

15. Find the **streak** — for each customer, find their longest streak of consecutive days with at least one order.

16. Write a query using NTH_VALUE to get each customer's **2nd and 3rd order amounts**.

17. Calculate the **contribution of each order** to the customer's total spending: `order_amount / customer_total * 100`.

18. Find the **most improved restaurant** — the one with the biggest positive change in average monthly revenue between Q1 and Q2 2026.

19. Build a simple **cohort retention table**: for each signup month, show how many customers were still active in months 0, 1, 2, 3.

20. **Challenge:** Write a query that assigns a **customer lifetime value segment** using window functions:
    - Calculate total spending per customer
    - Rank customers into 5 equal groups using NTILE(5)
    - For each group, show min/max spending, count of customers, and average order frequency

---

### ✅ Morning Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1:**
```sql
SELECT 
    DATE(o.order_date) AS order_date,
    o.restaurant_id,
    SUM(o.total_amount) AS daily_revenue,
    SUM(SUM(o.total_amount)) OVER (
        PARTITION BY o.restaurant_id 
        ORDER BY DATE(o.order_date) 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM orders o
WHERE o.status = 'completed'
GROUP BY DATE(o.order_date), o.restaurant_id
ORDER BY o.restaurant_id, order_date;
```

**Answer 2:**
```sql
WITH daily AS (
    SELECT DATE(order_date) AS day, SUM(total_amount) AS daily_rev
    FROM orders WHERE status = 'completed'
    GROUP BY DATE(order_date)
)
SELECT 
    day,
    daily_rev,
    AVG(daily_rev) OVER (
        ORDER BY day 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_avg
FROM daily
ORDER BY day;
```

**Answer 3:**
```sql
SELECT * FROM (
    SELECT 
        c.city,
        c.customer_name,
        SUM(o.total_amount) AS total_spent,
        RANK() OVER (PARTITION BY c.city ORDER BY SUM(o.total_amount) DESC) AS city_rank
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.status = 'completed'
    GROUP BY c.city, c.customer_name
) ranked
WHERE city_rank <= 2
ORDER BY city, city_rank;
```

**Answer 4:**
```sql
SELECT *
FROM (
    SELECT 
        customer_id,
        order_date,
        LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order,
        order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS gap_days
    FROM orders
    WHERE status = 'completed'
) gaps
WHERE gap_days > 7
ORDER BY customer_id, order_date;
```

**Answer 5:**
```sql
WITH monthly AS (
    SELECT 
        r.restaurant_name,
        DATE_TRUNC('month', o.order_date) AS month,
        SUM(o.total_amount) AS revenue
    FROM orders o
    JOIN restaurants r ON o.restaurant_id = r.restaurant_id
    WHERE o.status = 'completed'
    GROUP BY r.restaurant_name, DATE_TRUNC('month', o.order_date)
)
SELECT 
    restaurant_name,
    TO_CHAR(month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue) OVER (PARTITION BY restaurant_name ORDER BY month) AS prev_month,
    ROUND(
        (revenue - LAG(revenue) OVER (PARTITION BY restaurant_name ORDER BY month)) /
        NULLIF(LAG(revenue) OVER (PARTITION BY restaurant_name ORDER BY month), 0) * 100, 2
    ) AS growth_pct
FROM monthly
ORDER BY restaurant_name, month;
```

**Answer 6:**
```sql
WITH spending AS (
    SELECT 
        c.customer_name,
        SUM(o.total_amount) AS total_spent
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_name
)
SELECT 
    customer_name,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent DESC) AS quartile,
    CASE NTILE(4) OVER (ORDER BY total_spent DESC)
        WHEN 1 THEN 'Q1-Top'
        WHEN 2 THEN 'Q2'
        WHEN 3 THEN 'Q3'
        WHEN 4 THEN 'Q4-Bottom'
    END AS quartile_label
FROM spending
ORDER BY total_spent DESC;
```

**Answer 7:**
```sql
SELECT DISTINCT
    c.customer_name,
    FIRST_VALUE(o.order_date) OVER (
        PARTITION BY c.customer_id ORDER BY o.order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_order_date,
    LAST_VALUE(o.order_date) OVER (
        PARTITION BY c.customer_id ORDER BY o.order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_order_date,
    LAST_VALUE(o.order_date) OVER (
        PARTITION BY c.customer_id ORDER BY o.order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) - FIRST_VALUE(o.order_date) OVER (
        PARTITION BY c.customer_id ORDER BY o.order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lifetime_days
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';
```

**Answer 8:**
```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date) AS month, SUM(total_amount) AS revenue
    FROM orders WHERE status = 'completed'
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT 
    TO_CHAR(month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue, 12) OVER (ORDER BY month) AS same_month_prev_year,
    ROUND(
        (revenue - LAG(revenue, 12) OVER (ORDER BY month)) /
        NULLIF(LAG(revenue, 12) OVER (ORDER BY month), 0) * 100, 2
    ) AS yoy_growth_pct
FROM monthly
ORDER BY month;
```

**Answer 9:**
```sql
WITH ordered AS (
    SELECT 
        customer_id,
        order_date,
        status,
        LAG(status) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_status,
        LAG(status, 2) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_prev_status
    FROM orders
)
SELECT DISTINCT customer_id
FROM ordered
WHERE status = 'cancelled' 
AND prev_status = 'cancelled';
```

**Answer 10:**
```sql
SELECT 
    o.restaurant_id,
    o.order_date,
    o.total_amount,
    AVG(o.total_amount) OVER (
        PARTITION BY o.restaurant_id 
        ORDER BY o.order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cum_avg
FROM orders o
WHERE o.status = 'completed'
ORDER BY o.restaurant_id, o.order_date;
```

**Answer 11:**
```sql
WITH monthly AS (
    SELECT 
        restaurant_id,
        DATE_TRUNC('month', order_date) AS month,
        SUM(total_amount) AS revenue
    FROM orders WHERE status = 'completed'
    GROUP BY restaurant_id, DATE_TRUNC('month', order_date)
)
SELECT 
    restaurant_id,
    TO_CHAR(month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue) OVER (PARTITION BY restaurant_id ORDER BY month) AS prev_rev,
    ROUND(
        (revenue - LAG(revenue) OVER (PARTITION BY restaurant_id ORDER BY month)) /
        NULLIF(LAG(revenue) OVER (PARTITION BY restaurant_id ORDER BY month), 0) * 100, 2
    ) AS change_pct
FROM monthly
WHERE (revenue - LAG(revenue) OVER (PARTITION BY restaurant_id ORDER BY month)) /
      NULLIF(LAG(revenue) OVER (PARTITION BY restaurant_id ORDER BY month), 0) < -0.20
ORDER BY change_pct;
```

**Answer 12:**
```sql
SELECT 
    restaurant_id,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total_amount) AS median_order
FROM orders
WHERE status = 'completed'
GROUP BY restaurant_id;
```

**Answer 13:**
```sql
WITH first_months AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) AS first_month
    FROM orders
    GROUP BY customer_id
),
monthly_orders AS (
    SELECT DISTINCT
        o.customer_id,
        DATE_TRUNC('month', o.order_date) AS order_month
    FROM orders o
)
SELECT 
    TO_CHAR(mo.order_month, 'YYYY-MM') AS month,
    COUNT(*) AS total_customers,
    COUNT(CASE WHEN fm.first_month = mo.order_month THEN 1 END) AS new_customers,
    COUNT(CASE WHEN fm.first_month < mo.order_month THEN 1 END) AS returning_customers
FROM monthly_orders mo
JOIN first_months fm ON mo.customer_id = fm.customer_id
GROUP BY mo.order_month
ORDER BY mo.order_month;
```

**Answer 14:**
```sql
WITH monthly_customers AS (
    SELECT DISTINCT
        customer_id,
        DATE_TRUNC('month', order_date) AS order_month
    FROM orders
    WHERE status = 'completed'
),
with_next AS (
    SELECT 
        customer_id,
        order_month,
        LEAD(order_month) OVER (PARTITION BY customer_id ORDER BY order_month) AS next_month
    FROM monthly_customers
)
SELECT 
    TO_CHAR(order_month, 'YYYY-MM') AS month,
    COUNT(DISTINCT customer_id) AS total_customers,
    COUNT(DISTINCT CASE WHEN next_month IS NOT NULL 
          AND next_month - order_month <= INTERVAL '30 days' THEN customer_id END) AS repeat_customers,
    ROUND(
        COUNT(DISTINCT CASE WHEN next_month IS NOT NULL 
              AND next_month - order_month <= INTERVAL '30 days' THEN customer_id END)::DECIMAL / 
        NULLIF(COUNT(DISTINCT customer_id), 0) * 100, 1
    ) AS repeat_rate_pct
FROM with_next
GROUP BY order_month
ORDER BY order_month;
```

**Answer 15:**
```sql
WITH daily_orders AS (
    SELECT DISTINCT customer_id, DATE(order_date) AS order_day
    FROM orders WHERE status = 'completed'
),
with_streak AS (
    SELECT 
        customer_id,
        order_day,
        order_day - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_day)::INT AS streak_group
    FROM daily_orders
)
SELECT 
    customer_id,
    streak_group,
    COUNT(*) AS streak_length,
    MIN(order_day) AS streak_start,
    MAX(order_day) AS streak_end
FROM with_streak
GROUP BY customer_id, streak_group
ORDER BY streak_length DESC;
```
The trick: consecutive dates minus a sequential number produce the same value for a streak. `date - row_number` is constant when dates are consecutive!

**Answer 16:**
```sql
SELECT DISTINCT
    customer_id,
    NTH_VALUE(total_amount, 2) OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_order,
    NTH_VALUE(total_amount, 3) OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS third_order
FROM orders
WHERE status = 'completed';
```

**Answer 17:**
```sql
SELECT 
    o.customer_id,
    o.order_id,
    o.total_amount,
    SUM(o.total_amount) OVER (PARTITION BY o.customer_id) AS customer_total,
    ROUND(o.total_amount / SUM(o.total_amount) OVER (PARTITION BY o.customer_id) * 100, 2) AS contribution_pct
FROM orders o
WHERE o.status = 'completed';
```

**Answer 18:**
```sql
WITH quarterly AS (
    SELECT 
        restaurant_id,
        CASE 
            WHEN EXTRACT(QUARTER FROM order_date) = 1 THEN 'Q1'
            WHEN EXTRACT(QUARTER FROM order_date) = 2 THEN 'Q2'
        END AS quarter,
        AVG(total_amount) AS avg_monthly_rev
    FROM orders
    WHERE status = 'completed'
    AND EXTRACT(QUARTER FROM order_date) IN (1, 2)
    AND EXTRACT(YEAR FROM order_date) = 2026
    GROUP BY restaurant_id, CASE 
        WHEN EXTRACT(QUARTER FROM order_date) = 1 THEN 'Q1'
        WHEN EXTRACT(QUARTER FROM order_date) = 2 THEN 'Q2'
    END
)
SELECT 
    restaurant_id,
    MAX(CASE WHEN quarter = 'Q1' THEN avg_monthly_rev END) AS q1_avg,
    MAX(CASE WHEN quarter = 'Q2' THEN avg_monthly_rev END) AS q2_avg,
    MAX(CASE WHEN quarter = 'Q2' THEN avg_monthly_rev END) - 
    MAX(CASE WHEN quarter = 'Q1' THEN avg_monthly_rev END) AS improvement
FROM quarterly
GROUP BY restaurant_id
ORDER BY improvement DESC
LIMIT 1;
```

**Answer 19:**
```sql
WITH user_cohorts AS (
    SELECT customer_id, DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id
),
activity AS (
    SELECT DISTINCT
        uc.customer_id,
        uc.cohort_month,
        DATE_TRUNC('month', o.order_date) AS active_month,
        EXTRACT(YEAR FROM DATE_TRUNC('month', o.order_date)) * 12 + EXTRACT(MONTH FROM DATE_TRUNC('month', o.order_date))
        - EXTRACT(YEAR FROM uc.cohort_month) * 12 - EXTRACT(MONTH FROM uc.cohort_month) AS month_offset
    FROM user_cohorts uc
    JOIN orders o ON uc.customer_id = o.customer_id AND o.status = 'completed'
)
SELECT 
    TO_CHAR(cohort_month, 'YYYY-MM') AS cohort,
    COUNT(DISTINCT CASE WHEN month_offset = 0 THEN customer_id END) AS m0,
    COUNT(DISTINCT CASE WHEN month_offset = 1 THEN customer_id END) AS m1,
    COUNT(DISTINCT CASE WHEN month_offset = 2 THEN customer_id END) AS m2,
    COUNT(DISTINCT CASE WHEN month_offset = 3 THEN customer_id END) AS m3
FROM activity
GROUP BY cohort_month
ORDER BY cohort_month;
```

**Answer 20:**
```sql
WITH customer_metrics AS (
    SELECT 
        c.customer_id,
        c.customer_name,
        SUM(o.total_amount) AS total_spending,
        COUNT(*) AS order_count,
        MIN(o.order_date) AS first_order,
        MAX(o.order_date) AS last_order
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_id, c.customer_name
),
segmented AS (
    SELECT 
        *,
        NTILE(5) OVER (ORDER BY total_spending DESC) AS segment
    FROM customer_metrics
)
SELECT 
    CASE segment
        WHEN 1 THEN 'Top 20%'
        WHEN 2 THEN '20-40%'
        WHEN 3 THEN '40-60%'
        WHEN 4 THEN '60-80%'
        WHEN 5 THEN 'Bottom 20%'
    END AS segment_label,
    COUNT(*) AS customer_count,
    MIN(total_spending) AS min_spending,
    MAX(total_spending) AS max_spending,
    ROUND(AVG(order_count), 1) AS avg_order_freq
FROM segmented
GROUP BY segment
ORDER BY segment;
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Real-World Analytics Patterns

### Concept 6: RFM Analysis (Recency, Frequency, Monetary)

```sql
-- RFM: Classic customer segmentation technique
-- Recency: How recently did they order?
-- Frequency: How often do they order?
-- Monetary: How much do they spend?

WITH rfm_raw AS (
    SELECT 
        customer_id,
        CURRENT_DATE - MAX(order_date) AS recency_days,
        COUNT(*) AS frequency,
        SUM(total_amount) AS monetary
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),
rfm_scores AS (
    SELECT 
        customer_id,
        recency_days,
        frequency,
        monetary,
        -- Score 1-5 (5 is best). Lower recency_days = better score
        NTILE(5) OVER (ORDER BY recency_days DESC) AS recency_score,
        NTILE(5) OVER (ORDER BY frequency) AS frequency_score,
        NTILE(5) OVER (ORDER BY monetary) AS monetary_score
    FROM rfm_raw
)
SELECT 
    customer_id,
    recency_score || frequency_score || monetary_score AS rfm_segment,
    CASE
        WHEN recency_score >= 4 AND frequency_score >= 4 AND monetary_score >= 4 THEN 'Champions'
        WHEN recency_score >= 3 AND frequency_score >= 3 AND monetary_score >= 3 THEN 'Loyal'
        WHEN recency_score >= 4 AND frequency_score <= 2 THEN 'New Customers'
        WHEN recency_score <= 2 AND frequency_score >= 3 AND monetary_score >= 3 THEN 'At Risk'
        WHEN recency_score <= 2 AND frequency_score <= 2 THEN 'Lost'
        ELSE 'Average'
    END AS customer_type,
    recency_days,
    frequency,
    monetary
FROM rfm_scores
ORDER BY monetary DESC;
```

### Concept 7: Funnel Analysis

```sql
-- E-commerce funnel: Browse → Add to Cart → Checkout → Payment → Complete
-- Or for GrabFood: Search → View Menu → Add to Cart → Checkout → Delivered

CREATE TABLE user_events (
    event_id SERIAL,
    user_id INT,
    event_type VARCHAR(20),  -- 'search', 'view_menu', 'add_to_cart', 'checkout', 'delivered'
    event_timestamp TIMESTAMP,
    restaurant_id INT
);

-- Funnel analysis with window functions
WITH funnel_steps AS (
    SELECT 
        user_id,
        MIN(CASE WHEN event_type = 'search' THEN event_timestamp END) AS step1_search,
        MIN(CASE WHEN event_type = 'view_menu' THEN event_timestamp END) AS step2_view,
        MIN(CASE WHEN event_type = 'add_to_cart' THEN event_timestamp END) AS step3_cart,
        MIN(CASE WHEN event_type = 'checkout' THEN event_timestamp END) AS step4_checkout,
        MIN(CASE WHEN event_type = 'delivered' THEN event_timestamp END) AS step5_delivered
    FROM user_events
    WHERE event_timestamp >= '2026-05-01'
    GROUP BY user_id
)
SELECT 
    COUNT(*) AS total_users,
    COUNT(step1_search) AS searched,
    COUNT(step2_view) AS viewed_menu,
    COUNT(step3_cart) AS added_to_cart,
    COUNT(step4_checkout) AS checked_out,
    COUNT(step5_delivered) AS delivered,
    ROUND(COUNT(step2_view)::DECIMAL / NULLIF(COUNT(step1_search), 0) * 100, 1) AS search_to_view_pct,
    ROUND(COUNT(step3_cart)::DECIMAL / NULLIF(COUNT(step2_view), 0) * 100, 1) AS view_to_cart_pct,
    ROUND(COUNT(step4_checkout)::DECIMAL / NULLIF(COUNT(step3_cart), 0) * 100, 1) AS cart_to_checkout_pct,
    ROUND(COUNT(step5_delivered)::DECIMAL / NULLIF(COUNT(step4_checkout), 0) * 100, 1) AS checkout_to_delivered_pct,
    ROUND(COUNT(step5_delivered)::DECIMAL / NULLIF(COUNT(step1_search), 0) * 100, 1) AS overall_conversion_pct
FROM funnel_steps;
```

### Concept 8: Sessionization

```sql
-- Group events into "sessions" — 30-minute gap = new session
-- Critical for web analytics

WITH events_with_prev AS (
    SELECT 
        user_id,
        event_type,
        event_timestamp,
        LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp) AS prev_event
    FROM user_events
),
session_flags AS (
    SELECT 
        *,
        CASE 
            WHEN prev_event IS NULL THEN 1  -- first event ever
            WHEN event_timestamp - prev_event > INTERVAL '30 minutes' THEN 1  -- new session
            ELSE 0  -- same session
        END AS is_new_session
    FROM events_with_prev
),
sessions AS (
    SELECT 
        *,
        SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_timestamp) AS session_id
    FROM session_flags
)
SELECT 
    user_id,
    session_id,
    MIN(event_timestamp) AS session_start,
    MAX(event_timestamp) AS session_end,
    MAX(event_timestamp) - MIN(event_timestamp) AS session_duration,
    COUNT(*) AS events_in_session,
    ARRAY_AGG(DISTINCT event_type) AS event_types
FROM sessions
GROUP BY user_id, session_id
ORDER BY user_id, session_start;
```

### Concept 9: Gap & Island Analysis

```sql
-- Find continuous periods of activity ("islands") and gaps between them
-- Common use case: subscription analysis, device uptime, employee attendance

-- Example: Find continuous periods each restaurant was active
WITH daily_activity AS (
    SELECT DISTINCT 
        restaurant_id,
        DATE(order_date) AS active_date
    FROM orders
    WHERE status = 'completed'
),
island_groups AS (
    SELECT 
        restaurant_id,
        active_date,
        active_date - ROW_NUMBER() OVER (PARTITION BY restaurant_id ORDER BY active_date)::INT AS island_id
    FROM daily_activity
)
SELECT 
    restaurant_id,
    COUNT(*) AS island_length_days,
    MIN(active_date) AS island_start,
    MAX(active_date) AS island_end
FROM island_groups
GROUP BY restaurant_id, island_id
ORDER BY restaurant_id, island_start;
```

---

### 🏋️ Afternoon Exercises (15 questions)

21. Perform a complete **RFM analysis** on the GrabFood data. Segment customers into Champions, Loyal, At Risk, Lost. Count customers in each segment.

22. Build a **conversion funnel** for the GrabFood platform with these steps: App Open → Search → View Restaurant → Place Order → Delivery Complete. Calculate drop-off rates between each step.

23. **Sessionize** user events into 30-minute sessions. For each session, calculate duration, event count, and the unique event types.

24. Use **Gap & Island** analysis to find the longest consecutive streak of days with orders for each restaurant.

25. Calculate **customer churn rate** by month: % of customers who ordered last month but not this month.

26. Write a query to find **which day of the week** has the highest average revenue. Include a comparison between weekends and weekdays.

27. Calculate the **average time to second order** — how many days between a customer's first and second order on average?

28. Build a **restaurant health score** using window functions:
    - Revenue trend (growing = +1, declining = -1)
    - Order volume trend
    - Average rating trend
    - Customer retention rate
    Combine into a single score.

29. Write a query to find **seasonal patterns**: which cuisine types perform best in which months? Use a heatmap-style output.

30. Calculate the **average order value decay curve**: for each subsequent order (1st, 2nd, 3rd...), what's the average amount? Does spending increase or decrease over time?

31. Perform a **market basket analysis** lite: which pairs of dishes are most frequently ordered together? Use self-join on order_items.

32. Calculate the **Gini coefficient** of restaurant revenue — how unequal is the revenue distribution? (Hint: use cumulative percentages with window functions.)

33. Write a query that identifies **power users**: customers whose spending is in the top 5% AND who order at least 3× per month. Show their monthly spending trend.

34. Design a **real-time dashboard query** that shows: today's revenue, yesterday's revenue, WoW change, MoM change, top 5 restaurants today, and alert if today's revenue is >20% below the 7-day average. Use window functions.

35. **Challenge:** Build a complete **customer health dashboard** using only window functions:
    - Per customer: recency, frequency, monetary, trend (last 3 months vs prior 3 months)
    - Segment each customer (Growing, Stable, Declining, Churned)
    - Calculate overall portfolio health (% in each segment)
    - Identify the top 10 customers most likely to churn (declining spend + increasing gap between orders)

---

### ✅ Afternoon Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 21:** See Concept 6 above — the RFM analysis code is complete. Add this to count by segment:
```sql
-- Add after the main query
SELECT customer_type, COUNT(*), ROUND(AVG(monetary), 2) AS avg_monetary
FROM (...rfm query...)
GROUP BY customer_type
ORDER BY avg_monetary DESC;
```

**Answer 22:** See Concept 7. Create the `user_events` table with sample data, then run the funnel query.

**Answer 23:** See Concept 8 for the complete sessionization code.

**Answer 24:**
```sql
WITH daily AS (
    SELECT DISTINCT restaurant_id, DATE(order_date) AS active_date
    FROM orders WHERE status = 'completed'
),
islands AS (
    SELECT 
        restaurant_id,
        active_date,
        active_date - ROW_NUMBER() OVER (PARTITION BY restaurant_id ORDER BY active_date)::INT AS grp
    FROM daily
)
SELECT 
    restaurant_id,
    COUNT(*) AS streak_length,
    MIN(active_date) AS start_date,
    MAX(active_date) AS end_date
FROM islands
GROUP BY restaurant_id, grp
ORDER BY streak_length DESC;
```

**Answer 25:**
```sql
WITH monthly AS (
    SELECT DISTINCT
        customer_id,
        DATE_TRUNC('month', order_date) AS month
    FROM orders WHERE status = 'completed'
),
with_next AS (
    SELECT 
        customer_id,
        month,
        LEAD(month) OVER (PARTITION BY customer_id ORDER BY month) AS next_month
    FROM monthly
)
SELECT 
    TO_CHAR(month, 'YYYY-MM') AS month,
    COUNT(DISTINCT customer_id) AS active_customers,
    COUNT(DISTINCT CASE WHEN next_month IS NULL OR next_month != month + INTERVAL '1 month' 
          THEN customer_id END) AS churned,
    ROUND(
        COUNT(DISTINCT CASE WHEN next_month IS NULL OR next_month != month + INTERVAL '1 month' 
              THEN customer_id END)::DECIMAL / 
        NULLIF(COUNT(DISTINCT customer_id), 0) * 100, 1
    ) AS churn_rate_pct
FROM with_next
GROUP BY month
ORDER BY month;
```

**Answer 26:**
```sql
SELECT 
    TO_CHAR(order_date, 'TMDay') AS day_of_week,
    EXTRACT(ISODOW FROM order_date) AS dow,
    ROUND(AVG(total_amount), 2) AS avg_revenue,
    SUM(total_amount) AS total_revenue,
    COUNT(*) AS order_count,
    CASE WHEN EXTRACT(ISODOW FROM order_date) IN (6, 7) THEN 'Weekend' ELSE 'Weekday' END AS day_type
FROM orders
WHERE status = 'completed'
GROUP BY TO_CHAR(order_date, 'TMDay'), EXTRACT(ISODOW FROM order_date)
ORDER BY avg_revenue DESC;
```

**Answer 27:**
```sql
WITH orders_ranked AS (
    SELECT 
        customer_id,
        order_date,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_num
    FROM orders WHERE status = 'completed'
)
SELECT 
    ROUND(AVG(
        (SELECT order_date FROM orders_ranked o2 
         WHERE o2.customer_id = o1.customer_id AND o2.order_num = 2) - o1.order_date
    ), 1) AS avg_days_to_second_order
FROM orders_ranked o1
WHERE o1.order_num = 1
AND EXISTS (
    SELECT 1 FROM orders_ranked o2 
    WHERE o2.customer_id = o1.customer_id AND o2.order_num = 2
);
```

**Answer 28:** This is a complex analytical exercise. The approach:
1. Calculate 3-month revenue trend using LAG
2. Calculate order volume trend
3. Check if average customer return rate is improving
4. Combine into a weighted score

**Answer 29:**
```sql
SELECT 
    r.cuisine_type,
    EXTRACT(MONTH FROM o.order_date) AS month,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE o.status = 'completed'
GROUP BY r.cuisine_type, EXTRACT(MONTH FROM o.order_date)
ORDER BY r.cuisine_type, month;
-- Visualize as a heatmap in your BI tool
```

**Answer 30:**
```sql
WITH order_numbered AS (
    SELECT 
        customer_id,
        total_amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_n
    FROM orders WHERE status = 'completed'
)
SELECT 
    order_n AS order_number,
    COUNT(*) AS customers_with_this_many,
    ROUND(AVG(total_amount), 2) AS avg_amount
FROM order_numbered
GROUP BY order_n
ORDER BY order_n
LIMIT 20;
```

**Answer 31:**
```sql
SELECT 
    oi1.dish_name AS dish_a,
    oi2.dish_name AS dish_b,
    COUNT(DISTINCT oi1.order_id) AS times_together
FROM order_items oi1
JOIN order_items oi2 ON oi1.order_id = oi2.order_id AND oi1.item_id < oi2.item_id
GROUP BY oi1.dish_name, oi2.dish_name
ORDER BY times_together DESC
LIMIT 10;
```

**Answer 32:** Gini coefficient requires cumulative distribution:
```sql
WITH restaurant_rev AS (
    SELECT restaurant_id, SUM(total_amount) AS revenue
    FROM orders WHERE status = 'completed'
    GROUP BY restaurant_id
),
ranked AS (
    SELECT 
        restaurant_id,
        revenue,
        ROW_NUMBER() OVER (ORDER BY revenue) AS rn,
        COUNT(*) OVER () AS n,
        SUM(revenue) OVER () AS total_rev,
        SUM(revenue) OVER (ORDER BY revenue) AS cum_rev
    FROM restaurant_rev
)
SELECT 
    ROUND(1 - 2.0 * SUM((rn::DECIMAL / n) * (cum_rev - revenue + revenue / 2)) / SUM(revenue / total_rev), 4) AS gini
FROM ranked;
-- Gini close to 0 = equal distribution, close to 1 = highly unequal
```

**Answer 33:**
```sql
WITH monthly_spending AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', order_date) AS month,
        SUM(total_amount) AS monthly_spend,
        COUNT(*) AS monthly_orders
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id, DATE_TRUNC('month', order_date)
),
user_stats AS (
    SELECT 
        customer_id,
        SUM(monthly_spend) AS total_spend,
        PERCENT_RANK() OVER (ORDER BY SUM(monthly_spend)) AS spend_pct,
        AVG(monthly_orders) AS avg_monthly_orders
    FROM monthly_spending
    GROUP BY customer_id
)
SELECT *
FROM user_stats
WHERE spend_pct >= 0.95 AND avg_monthly_orders >= 3
ORDER BY total_spend DESC;
```

**Answer 34:**
```sql
WITH today AS (
    SELECT SUM(total_amount) AS rev_today FROM orders 
    WHERE DATE(order_date) = CURRENT_DATE AND status = 'completed'
),
yesterday AS (
    SELECT SUM(total_amount) AS rev_yesterday FROM orders 
    WHERE DATE(order_date) = CURRENT_DATE - 1 AND status = 'completed'
),
last_week AS (
    SELECT SUM(total_amount) AS rev_last_week FROM orders 
    WHERE DATE(order_date) BETWEEN CURRENT_DATE - 7 AND CURRENT_DATE - 1 AND status = 'completed'
),
last_month AS (
    SELECT SUM(total_amount) AS rev_last_month FROM orders 
    WHERE DATE(order_date) BETWEEN CURRENT_DATE - 30 AND CURRENT_DATE - 1 AND status = 'completed'
),
avg_7day AS (
    SELECT AVG(daily_rev) AS avg_7d FROM (
        SELECT DATE(order_date) AS d, SUM(total_amount) AS daily_rev
        FROM orders WHERE DATE(order_date) BETWEEN CURRENT_DATE - 7 AND CURRENT_DATE - 1 AND status = 'completed'
        GROUP BY DATE(order_date)
    ) sub
)
SELECT 
    t.rev_today,
    y.rev_yesterday,
    ROUND((t.rev_today - y.rev_yesterday) / NULLIF(y.rev_yesterday, 0) * 100, 1) AS dod_change_pct,
    ROUND((t.rev_today - lw.rev_last_week) / NULLIF(lw.rev_last_week, 0) * 100, 1) AS wow_change_pct,
    ROUND((t.rev_today - lm.rev_last_month / 30.0) / NULLIF(lm.rev_last_month / 30.0, 0) * 100, 1) AS vs_monthly_avg_pct,
    a.avg_7d,
    CASE WHEN t.rev_today < a.avg_7d * 0.8 THEN '⚠️ ALERT: >20% below 7-day avg' ELSE '✅ Normal' END AS status
FROM today t, yesterday y, last_week lw, last_month lm, avg_7day a;
```

**Answer 35:** This combines RFM (Concept 6), trend analysis (LAG), and churn detection (gap analysis). The complete solution is a multi-CTE query that:
1. Calculates per-customer metrics with window functions
2. Compares last 3 months vs prior 3 months using LAG
3. Segments based on spending trend + order gap trend
4. Ranks by churn probability

</details>

---

## 🌙 Evening Block (1 hour): Review & Practice

### Window Function Cheat Sheet

```sql
-- RANKING
ROW_NUMBER() OVER (ORDER BY col)        -- 1,2,3,4 (no ties)
RANK() OVER (ORDER BY col)              -- 1,2,2,4 (gaps)
DENSE_RANK() OVER (ORDER BY col)        -- 1,2,2,3 (no gaps)
NTILE(n) OVER (ORDER BY col)            -- divide into n groups

-- OFFSET
LAG(col, n) OVER (PARTITION BY ... ORDER BY ...)    -- look back n rows
LEAD(col, n) OVER (PARTITION BY ... ORDER BY ...)   -- look forward n rows
FIRST_VALUE(col) OVER (...)             -- first in window
LAST_VALUE(col) OVER (...)              -- last in window (careful with frame!)
NTH_VALUE(col, n) OVER (...)            -- Nth in window

-- AGGREGATE (all work as window functions)
SUM(col) OVER (...)
AVG(col) OVER (...)
COUNT(*) OVER (...)
MIN(col) OVER (...)
MAX(col) OVER (...)

-- FRAMES
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- running
ROWS BETWEEN N PRECEDING AND CURRENT ROW            -- last N
ROWS BETWEEN N PRECEDING AND N FOLLOWING            -- ±N
RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW  -- last 7 days
```

### 📝 Today's Checklist

- [ ] I can write window functions with correct PARTITION BY, ORDER BY, and frame clauses
- [ ] I understand ROWS vs RANGE in frame specifications
- [ ] I can calculate running totals, moving averages, and growth rates
- [ ] I can use LAG/LEAD for time-series comparisons
- [ ] I can perform RFM analysis and cohort analysis
- [ ] I understand sessionization and gap/island patterns
- [ ] I completed at least 25 of the 35 exercises
- [ ] I pushed any practice code to GitHub

### 📖 Further Reading (Free)

- [PostgreSQL Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)
- [Window Functions in SQL — Mode Analytics](https://mode.com/sql-tutorial/sql-window-functions/)
- [SQL Window Functions Cheat Sheet](https://www.sqlshack.com/sql-window-functions-cheat-sheet/)

---

*Day 26 complete! Tomorrow: Assessment Day — test everything you've learned this week.* 📝
