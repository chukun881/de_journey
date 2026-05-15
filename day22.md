# 📅 Day 22 — Thursday, 5 June 2026
# 🔗 Advanced SQL: Complex Joins & Query Performance

---

## 🎯 Today's Goal

You can already write JOINs and CTEs. Today we go **deeper** — the kind of SQL that separates juniors from solid data engineers. By end of today you'll be comfortable with multi-table joins, self-joins, anti-joins, and understanding *why* certain queries are slow.

**Why this matters:** In interviews, you WILL be asked to write complex joins on the spot. In real jobs, poorly written joins = slow dashboards = angry stakeholders.

---

## ☀️ Morning Block (2 hours): Multi-Table Joins & Join Strategies

### Concept 1: Join Order Matters

When you join 5+ tables, the order and type of join can change performance dramatically.

```sql
-- Example: GrabFood order analysis across 6 tables
-- Orders → Customers → Items → Restaurants → Payments → Delivery

-- BAD: Start with the biggest table
SELECT *
FROM payments p
JOIN orders o ON p.order_id = o.order_id
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
JOIN delivery d ON o.order_id = d.order_id;

-- BETTER: Start with the most filtered table, use INNER JOIN to reduce rows early
SELECT 
    c.customer_name,
    r.restaurant_name,
    o.order_date,
    SUM(oi.quantity * oi.unit_price) AS order_total
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
INNER JOIN restaurants r ON o.restaurant_id = r.restaurant_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_date >= '2026-01-01'
  AND r.city = 'Singapore'
GROUP BY c.customer_name, r.restaurant_name, o.order_date;
```

**💡 The Rule:** Filter early, join late. Always put WHERE clauses as early as possible mentally — the database optimizer may handle it, but writing clean SQL helps readability AND sometimes performance.

### Concept 2: Self-Joins

A self-join is when a table joins to itself. Super common in hierarchies.

```sql
-- Find GrabFood restaurants that are in the same postal code
SELECT 
    r1.restaurant_name AS restaurant_a,
    r2.restaurant_name AS restaurant_b,
    r1.postal_code
FROM restaurants r1
INNER JOIN restaurants r2 
    ON r1.postal_code = r2.postal_code 
    AND r1.restaurant_id < r2.restaurant_id  -- avoid duplicates
WHERE r1.city = 'Singapore';

-- Employee hierarchy: find all managers and their direct reports
SELECT 
    e.employee_name AS employee,
    m.employee_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;

-- Find customers who ordered from the same restaurant more than once in a week
SELECT 
    o1.customer_id,
    o1.restaurant_id,
    COUNT(*) AS repeat_orders
FROM orders o1
INNER JOIN orders o2 
    ON o1.customer_id = o2.customer_id
    AND o1.restaurant_id = o2.restaurant_id
    AND o1.order_id != o2.order_id
    AND o2.order_date BETWEEN o1.order_date AND o1.order_date + INTERVAL '7 days'
GROUP BY o1.customer_id, o1.restaurant_id
HAVING COUNT(*) > 1;
```

**Real use cases for self-joins:**
- Manager-employee hierarchies
- Finding duplicates
- Comparing rows within the same table (current vs previous)
- Category trees (parent-child)

### Concept 3: Anti-Joins & Semi-Joins

These are the secret weapons. "Find me X that does NOT have Y" or "Find me X that has at least one Y."

```sql
-- ANTI-JOIN: Customers who have NEVER placed an order
-- (LEFT JOIN + WHERE right side IS NULL)
SELECT c.customer_id, c.customer_name, c.email
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;

-- Same result with NOT EXISTS (often faster):
SELECT c.customer_id, c.customer_name, c.email
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id
);

-- SEMI-JOIN: Customers who have placed at least one order above RM100
-- (EXISTS is usually better than IN for large datasets)
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
    AND o.total_amount > 100
);
```

**💡 Interview tip:** When someone says "find customers with no orders," immediately think LEFT JOIN + IS NULL or NOT EXISTS. Both are valid; NOT EXISTS is usually more readable and sometimes faster.

### Concept 4: CROSS JOIN & Lateral Joins

```sql
-- CROSS JOIN: Every combination (use carefully!)
-- Use case: generate a date × product matrix for reporting
SELECT 
    d.date,
    p.product_name
FROM dates d
CROSS JOIN products p
WHERE d.date BETWEEN '2026-01-01' AND '2026-01-31';

-- LATERAL JOIN: For each row, run a subquery
-- Use case: Get top 3 orders per customer
SELECT 
    c.customer_name,
    top_orders.order_id,
    top_orders.total_amount
FROM customers c
CROSS JOIN LATERAL (
    SELECT o.order_id, o.total_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.total_amount DESC
    LIMIT 3
) AS top_orders;
```

---

### 🏋️ Morning Exercises (20 questions)

**For all exercises, use this schema:**

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    city VARCHAR(50),
    signup_date DATE,
    postal_code VARCHAR(10)
);

CREATE TABLE restaurants (
    restaurant_id INT PRIMARY KEY,
    restaurant_name VARCHAR(100),
    cuisine_type VARCHAR(50),
    city VARCHAR(50),
    postal_code VARCHAR(10),
    rating DECIMAL(3,2),
    is_active BOOLEAN
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    restaurant_id INT REFERENCES restaurants(restaurant_id),
    order_date TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    delivery_fee DECIMAL(5,2)
);

CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT REFERENCES orders(order_id),
    dish_name VARCHAR(100),
    quantity INT,
    unit_price DECIMAL(8,2)
);

CREATE TABLE payments (
    payment_id INT PRIMARY KEY,
    order_id INT REFERENCES orders(order_id),
    payment_method VARCHAR(30),
    amount_paid DECIMAL(10,2),
    payment_date TIMESTAMP,
    status VARCHAR(20)
);
```

1. Write a query to find all customers who have ordered from a restaurant with rating > 4.5. Show customer_name, restaurant_name, and the order total. Use EXISTS.

2. Find all restaurants that have NEVER received an order. Show restaurant_name, city, and rating.

3. Find all customers who signed up in 2025 but haven't placed any orders yet. Show customer_name, email, and signup_date.

4. Write a self-join query to find pairs of restaurants that share the same postal code AND cuisine type in Singapore. Show both restaurant names and the postal code.

5. Find the top 3 most expensive orders for each customer using a LATERAL join. Show customer_name, order_id, and total_amount.

6. Find all orders where the payment method was 'GrabPay' and the order contained at least one item with unit_price > RM30. Use EXISTS.

7. Write a query to count how many customers from each city have placed at least one order. Include cities with 0 orders (use a LEFT JOIN approach).

8. Find all dishes that appear in more than 10 different orders. Show dish_name and the count of distinct orders.

9. Using a self-join on the orders table, find customers who placed 2 or more orders from the same restaurant on the same day.

10. Find all customers whose total spending (across all orders) is higher than the average customer's total spending. Use a subquery.

11. Write a CROSS JOIN query to generate all combinations of months (Jan-Dec 2026) and the top 5 cuisine types by order count.

12. Find restaurants where the average order amount is LESS THAN the restaurant's own delivery fee. Show restaurant_name, avg_order, delivery_fee.

13. List the top 5 customers by number of orders, and for each, show their most-used payment method. (Hint: requires a join + window function)

14. Find all orders that have items but no corresponding payment record. Show order_id, order_date, and total_amount.

15. For each restaurant, find the customer who has spent the most. Show restaurant_name, customer_name, and total_spent.

16. Write a query using NOT EXISTS to find dishes that have NEVER been ordered (they exist in a `menu_items` table but not in `order_items`). Assume a `menu_items(restaurant_id, dish_name, price)` table.

17. Find all pairs of customers who live in the same postal_code and have ordered from the same restaurant at least once.

18. Write a query to find the second most recent order for each customer. Handle the case where a customer only has 1 order (should return NULL).

19. Using a self-join, find restaurants whose rating dropped — compare ratings in a `restaurant_rating_history` table (columns: restaurant_id, rating_date, rating) between their earliest and latest record.

20. **Challenge:** For each month in 2026, find the restaurant with the highest revenue. Show month, restaurant_name, and total_revenue. Handle ties by picking the one with fewer orders (more efficient).

---

### ✅ Morning Answers (check after attempting!)

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1:**
```sql
SELECT DISTINCT c.customer_name, r.restaurant_name, o.total_amount
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE EXISTS (
    SELECT 1 FROM restaurants r2 
    WHERE r2.restaurant_id = o.restaurant_id AND r2.rating > 4.5
)
AND r.rating > 4.5;
```

**Answer 2:**
```sql
SELECT r.restaurant_name, r.city, r.rating
FROM restaurants r
LEFT JOIN orders o ON r.restaurant_id = o.restaurant_id
WHERE o.order_id IS NULL;
```

**Answer 3:**
```sql
SELECT c.customer_name, c.email, c.signup_date
FROM customers c
WHERE c.signup_date >= '2025-01-01' AND c.signup_date < '2026-01-01'
AND NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

**Answer 4:**
```sql
SELECT 
    r1.restaurant_name AS restaurant_a,
    r2.restaurant_name AS restaurant_b,
    r1.postal_code,
    r1.cuisine_type
FROM restaurants r1
INNER JOIN restaurants r2 
    ON r1.postal_code = r2.postal_code
    AND r1.cuisine_type = r2.cuisine_type
    AND r1.restaurant_id < r2.restaurant_id
WHERE r1.city = 'Singapore';
```

**Answer 5:**
```sql
SELECT 
    c.customer_name,
    top3.order_id,
    top3.total_amount
FROM customers c
CROSS JOIN LATERAL (
    SELECT o.order_id, o.total_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.total_amount DESC
    LIMIT 3
) top3
ORDER BY c.customer_name, top3.total_amount DESC;
```

**Answer 6:**
```sql
SELECT o.order_id, o.total_amount, o.order_date
FROM orders o
INNER JOIN payments p ON o.order_id = p.order_id
WHERE p.payment_method = 'GrabPay'
AND EXISTS (
    SELECT 1 FROM order_items oi 
    WHERE oi.order_id = o.order_id AND oi.unit_price > 30
);
```

**Answer 7:**
```sql
SELECT 
    c.city,
    COUNT(DISTINCT o.customer_id) AS customers_with_orders,
    COUNT(DISTINCT c.customer_id) AS total_customers
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.city;
```

**Answer 8:**
```sql
SELECT dish_name, COUNT(DISTINCT order_id) AS order_count
FROM order_items
GROUP BY dish_name
HAVING COUNT(DISTINCT order_id) > 10
ORDER BY order_count DESC;
```

**Answer 9:**
```sql
SELECT 
    o1.customer_id,
    o1.restaurant_id,
    DATE(o1.order_date) AS order_day,
    COUNT(*) AS order_count
FROM orders o1
INNER JOIN orders o2 
    ON o1.customer_id = o2.customer_id
    AND o1.restaurant_id = o2.restaurant_id
    AND DATE(o1.order_date) = DATE(o2.order_date)
    AND o1.order_id < o2.order_id
GROUP BY o1.customer_id, o1.restaurant_id, DATE(o1.order_date);
```

**Answer 10:**
```sql
SELECT c.customer_name, SUM(o.total_amount) AS total_spending
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
HAVING SUM(o.total_amount) > (
    SELECT AVG(customer_total)
    FROM (
        SELECT SUM(total_amount) AS customer_total
        FROM orders
        GROUP BY customer_id
    ) AS avg_tbl
);
```

**Answer 11:**
```sql
WITH months AS (
    SELECT generate_series(
        DATE '2026-01-01', 
        DATE '2026-12-01', 
        INTERVAL '1 month'
    )::DATE AS month_date
),
top_cuisines AS (
    SELECT cuisine_type
    FROM (
        SELECT r.cuisine_type, COUNT(*) AS order_count,
               RANK() OVER (ORDER BY COUNT(*) DESC) AS rn
        FROM orders o
        JOIN restaurants r ON o.restaurant_id = r.restaurant_id
        GROUP BY r.cuisine_type
    ) ranked
    WHERE rn <= 5
)
SELECT 
    TO_CHAR(m.month_date, 'YYYY-MM') AS month,
    tc.cuisine_type
FROM months m
CROSS JOIN top_cuisines tc
ORDER BY month, cuisine_type;
```

**Answer 12:**
```sql
SELECT 
    r.restaurant_name,
    AVG(o.total_amount) AS avg_order,
    AVG(o.delivery_fee) AS avg_delivery_fee
FROM restaurants r
INNER JOIN orders o ON r.restaurant_id = o.restaurant_id
GROUP BY r.restaurant_id, r.restaurant_name
HAVING AVG(o.total_amount) < AVG(o.delivery_fee);
```

**Answer 13:**
```sql
WITH customer_orders AS (
    SELECT 
        c.customer_name,
        p.payment_method,
        COUNT(*) AS usage_count,
        ROW_NUMBER() OVER (PARTITION BY c.customer_id ORDER BY COUNT(*) DESC) AS rn,
        COUNT(*) OVER (PARTITION BY c.customer_id) AS total_orders
    FROM customers c
    INNER JOIN orders o ON c.customer_id = o.customer_id
    INNER JOIN payments p ON o.order_id = p.order_id
    GROUP BY c.customer_id, c.customer_name, p.payment_method
),
ranked AS (
    SELECT 
        customer_name,
        total_orders,
        payment_method AS preferred_payment,
        ROW_NUMBER() OVER (ORDER BY total_orders DESC) AS customer_rank
    FROM customer_orders
    WHERE rn = 1
)
SELECT customer_name, total_orders, preferred_payment
FROM ranked
WHERE customer_rank <= 5;
```

**Answer 14:**
```sql
SELECT o.order_id, o.order_date, o.total_amount
FROM orders o
WHERE EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.order_id = o.order_id
)
AND NOT EXISTS (
    SELECT 1 FROM payments p WHERE p.order_id = o.order_id
);
```

**Answer 15:**
```sql
WITH customer_spending AS (
    SELECT 
        o.restaurant_id,
        o.customer_id,
        SUM(o.total_amount) AS total_spent
    FROM orders o
    GROUP BY o.restaurant_id, o.customer_id
),
ranked AS (
    SELECT 
        cs.restaurant_id,
        cs.customer_id,
        cs.total_spent,
        ROW_NUMBER() OVER (PARTITION BY cs.restaurant_id ORDER BY cs.total_spent DESC) AS rn
    FROM customer_spending cs
)
SELECT 
    r.restaurant_name,
    c.customer_name,
    rk.total_spent
FROM ranked rk
JOIN restaurants r ON rk.restaurant_id = r.restaurant_id
JOIN customers c ON rk.customer_id = c.customer_id
WHERE rk.rn = 1
ORDER BY rk.total_spent DESC;
```

**Answer 16:**
```sql
SELECT mi.dish_name, mi.restaurant_id, mi.price
FROM menu_items mi
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi 
    WHERE oi.dish_name = mi.dish_name
);
```

**Answer 17:**
```sql
SELECT 
    c1.customer_name AS customer_a,
    c2.customer_name AS customer_b,
    c1.postal_code
FROM customers c1
INNER JOIN customers c2 
    ON c1.postal_code = c2.postal_code
    AND c1.customer_id < c2.customer_id
WHERE EXISTS (
    SELECT 1 FROM orders o1
    JOIN orders o2 ON o1.restaurant_id = o2.restaurant_id
    WHERE o1.customer_id = c1.customer_id
    AND o2.customer_id = c2.customer_id
);
```

**Answer 18:**
```sql
WITH ranked_orders AS (
    SELECT 
        o.customer_id,
        o.order_id,
        o.order_date,
        ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.order_date DESC) AS rn
    FROM orders o
)
SELECT 
    c.customer_name,
    ro.order_id AS second_most_recent,
    ro.order_date
FROM customers c
LEFT JOIN ranked_orders ro ON c.customer_id = ro.customer_id AND ro.rn = 2;
```

**Answer 19:**
```sql
SELECT 
    rh1.restaurant_id,
    r.restaurant_name,
    rh1.rating AS earliest_rating,
    rh1.rating_date AS earliest_date,
    rh2.rating AS latest_rating,
    rh2.rating_date AS latest_date,
    rh2.rating - rh1.rating AS rating_change
FROM restaurant_rating_history rh1
INNER JOIN restaurant_rating_history rh2 
    ON rh1.restaurant_id = rh2.restaurant_id
INNER JOIN restaurants r ON rh1.restaurant_id = r.restaurant_id
WHERE rh1.rating_date = (
    SELECT MIN(rating_date) FROM restaurant_rating_history WHERE restaurant_id = rh1.restaurant_id
)
AND rh2.rating_date = (
    SELECT MAX(rating_date) FROM restaurant_rating_history WHERE restaurant_id = rh2.restaurant_id
)
AND rh2.rating < rh1.rating;
```

**Answer 20:**
```sql
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', o.order_date) AS month,
        r.restaurant_name,
        SUM(o.total_amount) AS total_revenue,
        COUNT(*) AS order_count,
        RANK() OVER (
            PARTITION BY DATE_TRUNC('month', o.order_date) 
            ORDER BY SUM(o.total_amount) DESC, COUNT(*) ASC
        ) AS rn
    FROM orders o
    INNER JOIN restaurants r ON o.restaurant_id = r.restaurant_id
    WHERE o.order_date >= '2026-01-01' AND o.order_date < '2027-01-01'
    GROUP BY DATE_TRUNC('month', o.order_date), r.restaurant_name
)
SELECT 
    TO_CHAR(month, 'YYYY-MM') AS month,
    restaurant_name,
    total_revenue
FROM monthly_revenue
WHERE rn = 1
ORDER BY month;
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Join Performance & EXPLAIN

### Concept 5: Understanding EXPLAIN (PostgreSQL)

```sql
-- Basic EXPLAIN (shows plan but doesn't run)
EXPLAIN
SELECT c.customer_name, COUNT(*) AS order_count
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_name;

-- EXPLAIN ANALYZE (actually runs the query - use with caution on production!)
EXPLAIN ANALYZE
SELECT c.customer_name, COUNT(*) AS order_count
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_name;
```

**What to look for in EXPLAIN output:**

| Term | Meaning | Should you worry? |
|------|---------|-------------------|
| Seq Scan | Reading entire table row by row | ⚠️ On large tables, yes |
| Index Scan | Using an index to find rows | ✅ Good |
| Hash Join | Building hash table in memory | ✅ Fine for medium tables |
| Merge Join | Both inputs sorted, merging | ✅ Very efficient |
| Nested Loop | For each row in A, scan B | ⚠️ Can be slow if both are large |
| Sort | Sorting results | ⚠️ Expensive on large data |

### Concept 6: Index Strategy for Joins

```sql
-- The #1 rule: index your foreign keys!
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_restaurant_id ON orders(restaurant_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Index for common WHERE clauses
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_status ON orders(status);

-- Composite index for common query patterns
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date);

-- Check if an index is being used
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 123;
-- Should show "Index Scan" instead of "Seq Scan"
```

**💡 Rule of thumb:** If you're joining on a column, it should have an index. Foreign keys are the #1 candidate.

### Concept 7: Common Join Performance Killers

```sql
-- KILLER 1: Joining on different data types
-- BAD: customer_id is INT, but customer_id in orders is VARCHAR
-- Always ensure join columns match in type!

-- KILLER 2: Functions on join columns
-- BAD:
SELECT * FROM orders o
JOIN customers c ON LOWER(c.email) = LOWER(o.customer_email);
-- Index can't be used because of LOWER()

-- BETTER: Store normalized data, or use expression indexes
CREATE INDEX idx_customers_email_lower ON customers(LOWER(email));

-- KILLER 3: OR conditions in joins
-- BAD:
SELECT * FROM orders o
JOIN customers c ON c.customer_id = o.customer_id 
    OR c.email = o.billing_email;
-- Splits into two separate scans

-- KILLER 4: Joining without filtering = processing millions of rows
-- Always add date range or status filters
```

### Concept 8: Query Rewriting for Performance

```sql
-- SLOW: Subquery in SELECT (runs for every row)
SELECT 
    c.customer_name,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) AS order_count,
    (SELECT SUM(total_amount) FROM orders WHERE customer_id = c.customer_id) AS total_spent
FROM customers c;

-- FAST: Single join with aggregation
SELECT 
    c.customer_name,
    COUNT(o.order_id) AS order_count,
    COALESCE(SUM(o.total_amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name;

-- SLOW: SELECT * with multiple joins
SELECT *
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN customers c ON o.customer_id = c.customer_id
JOIN restaurants r ON o.restaurant_id = r.restaurant_id;

-- FAST: Select only what you need
SELECT 
    o.order_id,
    c.customer_name,
    r.restaurant_name,
    SUM(oi.quantity * oi.unit_price) AS order_total
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN customers c ON o.customer_id = c.customer_id
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
GROUP BY o.order_id, c.customer_name, r.restaurant_name;
```

---

### 🏋️ Afternoon Exercises (15 questions)

For these exercises, you'll need to run queries in PostgreSQL and check EXPLAIN plans.

**Setup — create sample data:**

```sql
-- Run this to create test data for today's exercises
-- Customers
INSERT INTO customers (customer_id, customer_name, email, city, signup_date, postal_code)
SELECT 
    g,
    'Customer ' || g,
    'cust' || g || '@email.com',
    CASE WHEN g % 3 = 0 THEN 'Singapore' 
         WHEN g % 3 = 1 THEN 'Kuala Lumpur' 
         ELSE 'Penang' END,
    DATE '2025-01-01' + (g % 365) * INTERVAL '1 day',
    CASE WHEN g % 3 = 0 THEN '0' || (100000 + g % 50) 
         WHEN g % 3 = 1 THEN '50' || (100 + g % 200)
         ELSE '10' || (100 + g % 100) END
FROM generate_series(1, 1000) AS g;

-- Restaurants
INSERT INTO restaurants (restaurant_id, restaurant_name, cuisine_type, city, postal_code, rating, is_active)
SELECT 
    g,
    'Restaurant ' || g,
    CASE g % 5 WHEN 0 THEN 'Chinese' WHEN 1 THEN 'Malay' WHEN 2 THEN 'Indian' 
         WHEN 3 THEN 'Western' ELSE 'Japanese' END,
    CASE WHEN g % 2 = 0 THEN 'Singapore' ELSE 'Kuala Lumpur' END,
    CASE WHEN g % 2 = 0 THEN '0' || (100000 + g % 50) ELSE '50' || (100 + g % 200) END,
    (2.5 + (g % 30) * 0.1)::DECIMAL(3,2),
    (g % 10 != 0)  -- 10% inactive
FROM generate_series(1, 200) AS g;

-- Orders (5000 orders)
INSERT INTO orders (order_id, customer_id, restaurant_id, order_date, total_amount, status, delivery_fee)
SELECT 
    g,
    1 + (g % 1000),
    1 + (g % 200),
    TIMESTAMP '2026-01-01' + (g % 180) * INTERVAL '1 day' + (g % 24) * INTERVAL '1 hour',
    (5 + (g % 95))::DECIMAL(10,2),
    CASE g % 20 WHEN 0 THEN 'cancelled' WHEN 1 THEN 'refunded' ELSE 'completed' END,
    (1 + (g % 8))::DECIMAL(5,2)
FROM generate_series(1, 5000) AS g;

-- Order items (2-4 items per order)
INSERT INTO order_items (item_id, order_id, dish_name, quantity, unit_price)
SELECT 
    g,
    1 + (g % 5000),
    CASE (g % 10) WHEN 0 THEN 'Nasi Lemak' WHEN 1 THEN 'Chicken Rice' 
         WHEN 2 THEN 'Char Kway Teow' WHEN 3 THEN 'Laksa' 
         WHEN 4 THEN 'Roti Prata' WHEN 5 THEN 'Mee Goreng'
         WHEN 6 THEN 'Hokkien Mee' WHEN 7 THEN 'Satay' 
         WHEN 8 THEN 'Bak Kut Teh' ELSE 'Dim Sum' END,
    1 + (g % 4),
    (3 + (g % 25))::DECIMAL(8,2)
FROM generate_series(1, 15000) AS g;

-- Payments
INSERT INTO payments (payment_id, order_id, payment_method, amount_paid, payment_date, status)
SELECT 
    g,
    g,
    CASE g % 4 WHEN 0 THEN 'GrabPay' WHEN 1 THEN 'Credit Card' 
         WHEN 2 THEN 'Cash' ELSE 'FPX' END,
    (5 + (g % 95))::DECIMAL(10,2),
    TIMESTAMP '2026-01-01' + (g % 180) * INTERVAL '1 day' + (g % 24) * INTERVAL '1 hour',
    CASE WHEN g % 50 = 0 THEN 'failed' ELSE 'success' END
FROM generate_series(1, 4800) AS g;  -- 200 orders have no payment
```

21. Run `EXPLAIN ANALYZE` on a simple query joining customers and orders. What type of join does PostgreSQL choose? What is the actual execution time?

22. Create an index on `orders(customer_id)` if it doesn't exist. Run the same EXPLAIN ANALYZE again. Compare the execution plan — did it change?

23. Create a composite index on `orders(customer_id, order_date)`. Then run EXPLAIN on:
```sql
SELECT * FROM orders WHERE customer_id = 100 ORDER BY order_date DESC LIMIT 10;
```
Does it use your new index?

24. Write a query that intentionally triggers a Sequential Scan (no index usage). Then add an appropriate index and verify it switches to an Index Scan.

25. Compare the performance of `IN` vs `EXISTS` for finding customers with orders above RM50:
```sql
-- Version A: IN
SELECT customer_name FROM customers 
WHERE customer_id IN (SELECT customer_id FROM orders WHERE total_amount > 50);

-- Version B: EXISTS  
SELECT customer_name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id AND o.total_amount > 50);
```
Run EXPLAIN ANALYZE on both. Which is faster? Why?

26. Find the execution plan difference between `SELECT *` vs `SELECT specific_columns` when joining 3 tables. Does column selection affect the plan?

27. Write a query with a function on a join column (e.g., `UPPER()` or `LOWER()`). Check the EXPLAIN plan — does it still use indexes? Then create an expression index and check again.

28. Create a query that joins all 5 tables (customers, orders, order_items, restaurants, payments) and measures execution time. Then optimize it by:
    - Adding missing indexes
    - Selecting only needed columns
    - Adding a WHERE clause
   Measure the improvement with EXPLAIN ANALYZE.

29. Test the difference between `COUNT(*)` and `COUNT(column_name)` on a large result set. When would `COUNT(1)` be meaningfully different from `COUNT(*)`? (Spoiler: in PostgreSQL, never — but it's a common interview question!)

30. **Challenge:** You have a slow dashboard query that takes 5+ seconds. The query joins orders, customers, restaurants, and order_items with no WHERE clause and returns 6 months of data. Write 3 different optimization strategies and rank them by expected impact.

31. Compare `JOIN` vs `SUBQUERY` performance for this task: "For each restaurant, show the name of their most recent customer." Write both versions and compare EXPLAIN plans.

32. Investigate: Does PostgreSQL optimize `WHERE customer_id = 123 AND order_date > '2026-01-01'` differently when there are two separate single-column indexes vs one composite index? Show the EXPLAIN output difference.

33. Write a query that benefits from a `HASH JOIN` vs one that benefits from a `MERGE JOIN`. What data characteristics determine which join type PostgreSQL picks?

34. Investigate the `work_mem` setting impact: Run a query with a large GROUP BY and SORT. Check if increasing `work_mem` changes the plan (use `SET work_mem = '256MB';` for your session).

35. **Practical:** Design the optimal index strategy for the GrabFood schema above. Consider:
    - The 10 most common query patterns
    - Write vs read trade-offs
    - Index maintenance overhead
    Present your strategy as a list of CREATE INDEX statements with justifications.

---

### ✅ Afternoon Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 21:** Run and observe. Typically you'll see a Hash Join between customers and orders, with a Seq Scan on both tables (since there's no index on orders.customer_id yet).

**Answer 22:** After creating the index:
```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```
The plan should change to use an Index Scan on orders when filtering by customer_id, or at minimum the Hash Join becomes more efficient.

**Answer 23:**
```sql
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date DESC);
```
This should produce an "Index Scan using idx_orders_cust_date" — the composite index covers both the filter and the sort, making it very efficient.

**Answer 24:** A query like `SELECT * FROM orders WHERE status = 'cancelled'` will Seq Scan without an index. Adding:
```sql
CREATE INDEX idx_orders_status ON orders(status);
```
Should switch to Index Scan (though with only 5% cancelled orders, the optimizer might still choose Seq Scan if it deems it cheaper — this is normal!).

**Answer 25:** In PostgreSQL, the optimizer is smart and often rewrites IN to EXISTS internally. They may produce identical plans. But EXISTS is generally preferred for readability and portability. In older MySQL versions, EXISTS was significantly faster.

**Answer 26:** With `SELECT *`, PostgreSQL may need to read more data from disk. The *plan shape* is often the same, but the actual I/O cost is higher. Column selection matters more for wide tables (many columns).

**Answer 27:**
```sql
-- This won't use an index:
EXPLAIN SELECT * FROM customers WHERE LOWER(email) = 'cust100@email.com';
-- Add expression index:
CREATE INDEX idx_customers_email_lower ON customers(LOWER(email));
-- Now it uses the index:
EXPLAIN SELECT * FROM customers WHERE LOWER(email) = 'cust100@email.com';
```

**Answer 28:** Key optimizations:
1. Index on foreign keys: `orders(customer_id)`, `orders(restaurant_id)`, `order_items(order_id)`, `payments(order_id)`
2. Select specific columns instead of `SELECT *`
3. Add `WHERE o.order_date >= '2026-03-01'` to reduce rows

**Answer 29:** In PostgreSQL, `COUNT(*)`, `COUNT(1)`, and `COUNT('anything')` are all identical — the planner treats them the same. `COUNT(column_name)` is different because it counts non-NULL values only. This is a common interview gotcha!

**Answer 30:** Three strategies ranked by impact:
1. **Add date range filter** (highest impact) — reduces data from 6 months to what's needed
2. **Add indexes on join columns + WHERE columns** — enables index scans
3. **Materialized view / pre-aggregation** — compute once, query many times

**Answer 31:** The subquery approach with a lateral join is often cleaner and equally performant in PostgreSQL:
```sql
-- JOIN approach (with window function)
WITH latest AS (
    SELECT customer_id, restaurant_id,
           ROW_NUMBER() OVER (PARTITION BY restaurant_id ORDER BY order_date DESC) AS rn
    FROM orders
)
SELECT r.restaurant_name, c.customer_name
FROM restaurants r
JOIN latest l ON r.restaurant_id = l.restaurant_id AND l.rn = 1
JOIN customers c ON l.customer_id = c.customer_id;

-- LATERAL approach
SELECT r.restaurant_name, c.customer_name
FROM restaurants r
CROSS JOIN LATERAL (
    SELECT o.customer_id
    FROM orders o
    WHERE o.restaurant_id = r.restaurant_id
    ORDER BY o.order_date DESC
    LIMIT 1
) latest
JOIN customers c ON latest.customer_id = c.customer_id;
```

**Answer 32:** With two single-column indexes, PostgreSQL can only use one and must filter the other condition. With a composite index `(customer_id, order_date)`, it can use both conditions in a single index scan. The composite wins for this specific query pattern.

**Answer 33:**
- **Hash Join**: Best when one table is small enough to fit in memory (builds a hash table from the smaller table). Common for dimension-to-fact joins.
- **Merge Join**: Best when both inputs are already sorted (or can be cheaply sorted via index). Common when both sides have an index on the join key.

**Answer 34:**
```sql
SET work_mem = '256MB';
EXPLAIN ANALYZE SELECT customer_id, COUNT(*), SUM(total_amount) 
FROM orders GROUP BY customer_id ORDER BY SUM(total_amount) DESC;
```
With higher `work_mem`, PostgreSQL can do sorting and hashing in memory instead of spilling to disk. Watch for "external merge Disk" disappearing from the plan.

**Answer 35 — Optimal Index Strategy:**
```sql
-- Foreign keys (mandatory for join performance)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_restaurant_id ON orders(restaurant_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_payments_order_id ON payments(order_id);

-- Common WHERE filters
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_restaurants_city ON restaurants(city);

-- Composite for common dashboard queries
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date DESC);
CREATE INDEX idx_orders_rest_date ON orders(restaurant_id, order_date DESC);

-- Covering index for order listing (avoids table lookup)
CREATE INDEX idx_orders_covering ON orders(customer_id, order_date DESC) 
    INCLUDE (total_amount, status, restaurant_id);

-- Expression index for case-insensitive email lookup
CREATE INDEX idx_customers_email_lower ON customers(LOWER(email));

-- Partial index for active restaurants only
CREATE INDEX idx_restaurants_active ON restaurants(city, cuisine_type) 
    WHERE is_active = TRUE;
```

</details>

---

## 🌙 Evening Block (1 hour): Review & Practice

### Concept 9: Interview-Style Questions

Practice these without looking at notes. Set a timer for 5 minutes per question.

1. "Write a query to find the top 5 restaurants by revenue, but exclude cancelled orders, and show their rank."

2. "Find all customers who signed up last month but haven't placed an order yet."

3. "For each cuisine type, find the restaurant with the highest average order value."

4. "Find duplicate orders — same customer, same restaurant, same day, same total amount."

5. "Write a query to calculate month-over-month revenue growth for each restaurant."

### 📝 Today's Checklist

- [ ] I understand when to use LEFT JOIN vs NOT EXISTS for anti-joins
- [ ] I can write self-joins for hierarchical data
- [ ] I can read an EXPLAIN plan and identify performance issues
- [ ] I know which columns to index for join performance
- [ ] I can optimize a slow query by rewriting it
- [ ] I completed all exercises (or at least attempted them)
- [ ] I pushed any practice code to GitHub

### 📖 Further Reading (Free)

- [PostgreSQL EXPLAIN Documentation](https://www.postgresql.org/docs/current/sql-explain.html)
- [Use The Index, Luke](https://use-the-index-luke.com/) — excellent free resource on indexing
- [PostgreSQL Query Tuning](https://www.postgresql.org/docs/current/performance-tips.html)

---

*Day 22 complete! Tomorrow: Query Optimization & Execution Plans — going even deeper into making SQL fast.* 🔥
