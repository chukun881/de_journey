# 📅 Day 23 — Friday, 6 June 2026
# ⚡ Advanced SQL: Query Optimization & Execution Plans

---

## 🎯 Today's Goal

Yesterday you learned about join strategies and indexing. Today we go **deeper into the PostgreSQL optimizer** — understanding how it thinks, how to read complex EXPLAIN plans, and how to make queries fly. This is the stuff that makes you stand out in technical interviews.

**Why this matters:** A data engineer who can optimize queries is worth 3× one who just writes them. Your dashboards, reports, and pipelines all depend on fast SQL.

---

## ☀️ Morning Block (2 hours): Reading EXPLAIN Like a Pro

### Concept 1: EXPLAIN Output Anatomy

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT c.customer_name, SUM(o.total_amount) AS total_spent
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2026-01-01'
GROUP BY c.customer_name
ORDER BY total_spent DESC
LIMIT 10;
```

**What the output looks like (simplified):**
```
Limit  (cost=85.32..85.34 rows=10) (actual time=12.456..12.458 rows=10 loops=1)
  ->  Sort  (cost=85.32..87.50 rows=870) (actual time=12.455..12.457 rows=10 loops=1)
        Sort Key: (sum(o.total_amount)) DESC
        ->  HashAggregate  (cost=72.10..78.40 rows=870) (actual time=11.234..11.890 rows=870 loops=1)
              Group Key: c.customer_name
              ->  Hash Join  (cost=15.20..68.50 rows=870) (actual time=2.345..8.901 rows=870 loops=1)
                    Hash Cond: (o.customer_id = c.customer_id)
                    ->  Seq Scan on orders o  (cost=0.00..45.00 rows=870) (actual time=0.012..3.456 rows=870 loops=1)
                          Filter: (order_date >= '2026-01-01')
                          Rows Removed by Filter: 130
                    ->  Hash  (cost=10.00..10.00 rows=1000) (actual time=2.100..2.100 rows=1000 loops=1)
                          Buckets: 1024  Batches: 1  Memory Usage: 52kB
                          ->  Seq Scan on customers c  (cost=0.00..10.00 rows=1000) (actual time=0.005..1.200 rows=1000 loops=1)
```

**Reading order: Bottom-up, inside-out.**

| Line | What it tells you |
|------|-------------------|
| `Seq Scan on orders` | Reading the whole orders table (bottom of plan, first operation) |
| `Filter: (order_date >= ...)` | Applying WHERE clause, removed 130 rows |
| `Hash` | Building hash table from customers for the join |
| `Hash Join` | Joining orders to customers using the hash table |
| `HashAggregate` | Grouping by customer_name for the SUM |
| `Sort` | Sorting by total_spent DESC |
| `Limit` | Taking only top 10 |

**Key metrics:**
- **cost**: Planner's estimate (first number = startup cost, second = total cost)
- **actual time**: Real milliseconds measured
- **rows**: Estimated vs actual rows (big mismatch = stale statistics)
- **loops**: How many times this node ran (1 = single pass, >1 = correlated subquery)

### Concept 2: BUFFERS — The Real Performance Story

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 100;
```

**Output includes:**
```
Buffers: shared hit=5 read=2 dirtied=0 written=0
```

| Metric | Meaning | Good/Bad |
|--------|---------|-----------|
| `shared hit` | Read from cache (RAM) | ✅ Fast |
| `shared read` | Read from disk | ⚠️ Slower |
| `shared dirtied` | Modified page | Info only |
| `shared written` | Evicted to disk | ⚠️ Memory pressure |

**💡 Key insight:** If `read >> hit`, your data isn't cached. First run is slow, second run is fast. Always run EXPLAIN ANALYZE twice — the second run shows cached performance.

### Concept 3: Identifying Problem Patterns

```sql
-- Pattern 1: Sequential Scan on large table
-- BAD when: table has millions of rows and you need 100
Seq Scan on orders  (cost=0.00..15000.00 rows=5000000)
  Filter: (status = 'cancelled')
-- Fix: CREATE INDEX idx_orders_status ON orders(status);

-- Pattern 2: Nested Loop with large outer table
Nested Loop  (cost=0.00..999999.00 rows=1000000)
  ->  Seq Scan on orders  (rows=500000)
  ->  Index Scan on order_items  (rows=2)  -- runs 500000 times!
-- Fix: Ensure join columns indexed, consider hash join hint

-- Pattern 3: Sort spilling to disk
Sort  (cost=5000.00..5200.00 rows=100000)
  Sort Method: external merge  Disk: 2500kB
-- Fix: Increase work_mem, or add an index that provides sort order

-- Pattern 4: Row estimate mismatch
Hash Join  (rows=100) (actual rows=50000)
-- PostgreSQL expected 100 rows but got 50000
-- Fix: ANALYZE orders; -- update statistics
```

---

### 🏋️ Morning Exercises (15 questions)

Use the same GrabFood schema and data from Day 22.

1. Run `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` on this query and identify each node type:
```sql
SELECT r.cuisine_type, AVG(o.total_amount) AS avg_order
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE o.order_date >= '2026-03-01'
GROUP BY r.cuisine_type;
```
List every node from bottom to top with its type.

2. Run the query from #1 twice. Compare the `Buffers` output — how many more `shared hit` on the second run? Why?

3. Find a query where PostgreSQL chooses a Nested Loop. Hint: join two tables with a highly selective WHERE clause (e.g., single customer_id).

4. Create a query that produces a "Sort Method: external merge Disk" message. How much disk is used? Increase `work_mem` to make it sort in memory.

5. Run `ANALYZE orders;` then re-run a query. Does the estimated row count change? Why might statistics be stale?

6. Write a query with `EXPLAIN (ANALYZE, VERBOSE)` and find the "Rows Removed by Filter" value. What percentage of rows were wasted?

7. Compare these two queries' plans:
```sql
-- A
SELECT c.customer_name FROM customers c WHERE c.customer_id IN (SELECT customer_id FROM orders);
-- B  
SELECT c.customer_name FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```
Are the plans identical? Why or why not?

8. Force a sequential scan by disabling index scans temporarily:
```sql
SET enable_indexscan = off;
SET enable_bitmapscan = off;
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 100;
```
How much slower is it? Reset with `SET enable_indexscan = default;`

9. Use `EXPLAIN (ANALYZE, BUFFERS)` on a query joining all 5 tables. Identify which table contributes the most disk reads.

10. Write a query that benefits from a **covering index** (index-only scan). Create the index and verify the plan shows "Index Only Scan."

11. Investigate: What happens to the plan if you add `LIMIT 1` to a query that previously sorted 100k rows? Does the optimizer short-circuit?

12. Compare the plan for `SELECT COUNT(*) FROM orders` with and without an index on `order_id`. Does PostgreSQL use the index for counting?

13. Create a partial index on `orders WHERE status = 'cancelled'`. Then run EXPLAIN on a query filtering cancelled orders. Does it use the partial index? What about completed orders?

14. Run `EXPLAIN ANALYZE` on a query with a CTE vs the same query with a subquery. In PostgreSQL 12+, are they treated differently?

15. **Challenge:** You're given this slow query running on a production database with 50M orders:
```sql
SELECT c.customer_name, COUNT(*) AS orders, SUM(o.total_amount) AS total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2026-01-01'
GROUP BY c.customer_name
ORDER BY total DESC;
```
It takes 45 seconds. Propose 5 specific optimizations with expected impact.

---

### ✅ Morning Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1:** Reading bottom-up:
1. `Seq Scan on restaurants` — scanning all restaurants
2. `Seq Scan on orders` — scanning orders with filter `order_date >= '2026-03-01'`
3. `Hash` — building hash from restaurants
4. `Hash Join` — joining filtered orders to restaurants hash
5. `HashAggregate` — grouping by cuisine_type
6. `HashAggregate` (final) — computing final AVG

**Answer 2:** Second run shows more `shared hit` and fewer `shared read` because data is now in PostgreSQL's shared buffer cache. The OS may also cache pages.

**Answer 3:**
```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.customer_id = 1;
```
With a selective filter on customer_id, PostgreSQL may choose Nested Loop because it expects very few outer rows.

**Answer 4:**
```sql
SET work_mem = '1MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders ORDER BY order_date, total_amount DESC;
-- Should show "external merge Disk"
-- Then:
SET work_mem = '64MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders ORDER BY order_date, total_amount DESC;
-- Should show "quicksort" (in-memory)
```

**Answer 5:** ANALYZE collects fresh statistics on row counts, value distributions, and correlation. If data has changed significantly since last ANALYZE, estimates improve.

**Answer 6:**
```sql
EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM orders WHERE customer_id = 1 AND status = 'completed';
```
"Rows Removed by Filter" shows rows scanned but rejected. High percentage = index not selective enough or missing.

**Answer 7:** In modern PostgreSQL, the optimizer often rewrites IN subqueries to semi-joins, producing identical or very similar plans. The planner is smart enough to treat them equivalently in most cases.

**Answer 8:** Without index scans, PostgreSQL must Seq Scan the entire table. On our small dataset the difference is minimal, but on millions of rows it could be 1000× slower.

**Answer 9:** Typically the largest table (order_items with 15k rows) contributes the most reads. The plan should show which scan has the highest cost.

**Answer 10:**
```sql
CREATE INDEX idx_orders_cust_date_covering ON orders(customer_id, order_date) 
    INCLUDE (total_amount, status);

EXPLAIN ANALYZE
SELECT customer_id, order_date, total_amount, status
FROM orders
WHERE customer_id = 100;
-- Should show "Index Only Scan" — no table lookup needed!
```

**Answer 11:** Yes! With `LIMIT 1`, PostgreSQL can stop scanning after finding the first match. The sort may use a "top-N" heapsort instead of full sort, dramatically reducing memory and time.

**Answer 12:** PostgreSQL can use any index for COUNT(*) — it picks the smallest index. If there's a primary key index, it may scan that instead of the table. But `COUNT(*)` still needs to visit every row (or index entry) to count them.

**Answer 13:**
```sql
CREATE INDEX idx_orders_cancelled ON orders(order_date) WHERE status = 'cancelled';

EXPLAIN SELECT * FROM orders WHERE status = 'cancelled' AND order_date >= '2026-01-01';
-- Uses the partial index

EXPLAIN SELECT * FROM orders WHERE status = 'completed' AND order_date >= '2026-01-01';
-- Does NOT use the partial index (not applicable)
```

**Answer 14:** PostgreSQL 12+ can "inline" non-recursive CTEs, treating them like subqueries. Before PG12, CTEs were always "optimization fences" (materialized first). In PG12+, add `MATERIALIZED` to force old behavior:
```sql
WITH my_cte AS MATERIALIZED (SELECT ...) SELECT ...
```

**Answer 15 — Five optimizations:**
1. **Index on orders(order_date)** — enables index scan instead of seq scan on 50M rows (huge impact: reduces scan from 50M to ~5M rows)
2. **Composite index on orders(order_date, customer_id, total_amount) INCLUDE (order_id)** — covering index, avoids table lookup entirely
3. **Partition orders by month** — each partition is smaller, queries scan only relevant months
4. **Pre-aggregate into a materialized view** — refresh nightly, dashboard queries become instant
5. **Increase work_mem for the session** — allows hash aggregation and sorting in memory instead of spilling to disk

</details>

---

## 🌤️ Afternoon Block (2 hours): Advanced Optimization Techniques

### Concept 4: Common Table Expressions vs Subqueries vs Temp Tables

```sql
-- CTE: Readable, can be referenced multiple times
WITH monthly_orders AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        restaurant_id,
        COUNT(*) AS order_count,
        SUM(total_amount) AS revenue
    FROM orders
    WHERE order_date >= '2026-01-01'
    GROUP BY DATE_TRUNC('month', order_date), restaurant_id
),
ranked AS (
    SELECT 
        month,
        restaurant_id,
        revenue,
        RANK() OVER (PARTITION BY month ORDER BY revenue DESC) AS rn
    FROM monthly_orders
)
SELECT r.restaurant_name, TO_CHAR(rk.month, 'YYYY-MM') AS month, rk.revenue
FROM ranked rk
JOIN restaurants r ON rk.restaurant_id = r.restaurant_id
WHERE rk.rn <= 3
ORDER BY rk.month, rk.rn;

-- Temp table: Materialized, indexed, good for complex multi-step
CREATE TEMP TABLE monthly_orders AS
SELECT 
    DATE_TRUNC('month', order_date) AS month,
    restaurant_id,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue
FROM orders
WHERE order_date >= '2026-01-01'
GROUP BY DATE_TRUNC('month', order_date), restaurant_id;

CREATE INDEX idx_tmp_monthly ON monthly_orders(month, restaurant_id);

-- Now query the temp table — potentially faster for complex operations
SELECT * FROM monthly_orders WHERE month = '2026-03-01';
```

**When to use what:**

| Approach | Best for | Performance |
|----------|----------|-------------|
| CTE | Readability, referenced 2-3 times | Good (PG12+ inlines) |
| Subquery | Single-use, simple | Same as CTE |
| Temp table | Complex multi-step, large data, need indexes | Best for big data |
| Materialized View | Repeated queries, dashboards | Best for read-heavy |

### Concept 5: Window Function Optimization

```sql
-- SLOW: Multiple window functions with different PARTITION BY
SELECT 
    customer_id,
    order_date,
    total_amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn_cust,
    ROW_NUMBER() OVER (PARTITION BY restaurant_id ORDER BY total_amount DESC) AS rn_rest,
    SUM(total_amount) OVER (PARTITION BY customer_id) AS customer_total,
    AVG(total_amount) OVER (PARTITION BY restaurant_id) AS rest_avg
FROM orders;

-- PostgreSQL must do separate sorts for each unique PARTITION/ORDER combination!
-- 3 different PARTITION BYs = 3 separate sort operations

-- OPTIMIZED: Split into separate queries and join
WITH customer_window AS (
    SELECT 
        order_id, customer_id,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn_cust,
        SUM(total_amount) OVER (PARTITION BY customer_id) AS customer_total
    FROM orders
),
restaurant_window AS (
    SELECT 
        order_id,
        ROW_NUMBER() OVER (PARTITION BY restaurant_id ORDER BY total_amount DESC) AS rn_rest,
        AVG(total_amount) OVER (PARTITION BY restaurant_id) AS rest_avg
    FROM orders
)
SELECT 
    o.order_id, o.customer_id, o.order_date, o.total_amount,
    cw.rn_cust, cw.customer_total,
    rw.rn_rest, rw.rest_avg
FROM orders o
JOIN customer_window cw ON o.order_id = cw.order_id
JOIN restaurant_window rw ON o.order_id = rw.order_id;
```

### Concept 6: Pagination Optimization

```sql
-- BAD: OFFSET with large values (scans and discards rows)
SELECT * FROM orders ORDER BY order_date DESC LIMIT 50 OFFSET 100000;
-- PostgreSQL must scan 100,050 rows and discard 100,000!

-- GOOD: Keyset pagination (seek method)
SELECT * FROM orders 
WHERE (order_date, order_id) < ('2026-03-15', 45000)
ORDER BY order_date DESC, order_id DESC
LIMIT 50;
-- Uses index, no wasted scanning

-- The index needed:
CREATE INDEX idx_orders_date_id ON orders(order_date DESC, order_id DESC);
```

### Concept 7: Batch Processing for Large Updates

```sql
-- BAD: Update millions of rows in one transaction
UPDATE orders SET status = 'archived' WHERE order_date < '2025-01-01';
-- Locks rows, fills WAL, can OOM

-- GOOD: Batch in chunks
DO $$
DECLARE
    batch_size INT := 10000;
    affected INT := 1;
BEGIN
    WHILE affected > 0 LOOP
        UPDATE orders 
        SET status = 'archived' 
        WHERE order_id IN (
            SELECT order_id FROM orders 
            WHERE order_date < '2025-01-01' AND status != 'archived'
            LIMIT batch_size
        );
        affected := ROW_COUNT;
        RAISE NOTICE 'Updated % rows', affected;
        COMMIT;
    END LOOP;
END $$;
```

### Concept 8: Materialized Views for Dashboards

```sql
-- Create a materialized view for a common dashboard query
CREATE MATERIALIZED VIEW mv_daily_restaurant_summary AS
SELECT 
    DATE(o.order_date) AS order_date,
    r.restaurant_name,
    r.cuisine_type,
    r.city,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.total_amount) AS total_revenue,
    AVG(o.total_amount) AS avg_order_value,
    COUNT(DISTINCT o.customer_id) AS unique_customers
FROM orders o
INNER JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE o.status = 'completed'
GROUP BY DATE(o.order_date), r.restaurant_name, r.cuisine_type, r.city;

-- Create index on the materialized view
CREATE INDEX idx_mv_daily_date ON mv_daily_restaurant_summary(order_date);
CREATE INDEX idx_mv_daily_rest ON mv_daily_restaurant_summary(restaurant_name);

-- Refresh it (schedule this as a cron job or Airflow task)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_restaurant_summary;
-- CONCURRENTLY allows reads while refreshing (requires unique index)

-- Create a unique index for CONCURRENT refresh
CREATE UNIQUE INDEX idx_mv_daily_unique ON mv_daily_restaurant_summary(order_date, restaurant_name);
```

---

### 🏋️ Afternoon Exercises (15 questions)

16. Write a CTE-based query that:
    - Finds monthly revenue per restaurant
    - Calculates each restaurant's revenue as a % of total monthly revenue
    - Shows only restaurants above their monthly average

17. Convert the CTE from #16 to use temp tables instead. Add appropriate indexes. Compare the EXPLAIN plans.

18. Write a keyset pagination query for the orders table. Fetch pages 1-5 (50 rows each) without using OFFSET.

19. Create a materialized view called `mv_customer_summary` that shows customer_name, total_orders, total_spent, favorite_restaurant (most orders), and favorite_cuisine. Refresh it and query it.

20. Optimize this query using window function splitting:
```sql
SELECT 
    o.order_id,
    o.customer_id,
    o.restaurant_id,
    o.total_amount,
    SUM(o.total_amount) OVER (PARTITION BY o.customer_id ORDER BY o.order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total_cust,
    SUM(o.total_amount) OVER (PARTITION BY o.restaurant_id ORDER BY o.order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total_rest,
    RANK() OVER (PARTITION BY o.restaurant_id ORDER BY o.total_amount DESC) AS price_rank_rest
FROM orders o;
```

21. Write a batch UPDATE statement that archives orders older than 2025-06-01 in batches of 5000. Show how to monitor progress.

22. Investigate: What happens if you try to `REFRESH MATERIALIZED VIEW CONCURRENTLY` without a unique index? What error do you get?

23. Write a query using `FILTER` clause (PostgreSQL extension) instead of CASE WHEN for conditional aggregation:
```sql
-- Convert this to use FILTER:
SELECT 
    customer_id,
    COUNT(CASE WHEN status = 'completed' THEN 1 END) AS completed_orders,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled_orders,
    AVG(CASE WHEN status = 'completed' THEN total_amount END) AS avg_completed
FROM orders
GROUP BY customer_id;
```

24. Create an optimized query for "Show me the top 10 customers by spending, with their rank, and the gap to the next customer." Use a single window function pass.

25. Write a query that uses `LATERAL` to efficiently fetch the latest 3 orders per restaurant. Compare its performance to the equivalent window function approach (`ROW_NUMBER` + filter).

26. Optimize a dashboard query that currently joins 4 tables and aggregates. Propose 3 different optimization strategies and rank them.

27. Write a query that calculates a 7-day rolling average of daily revenue. Optimize it to avoid recalculating the entire window for each row.

28. Investigate the difference between `REFRESH MATERIALIZED VIEW` vs `REFRESH MATERIALIZED VIEW CONCURRENTLY`. When is CONCURRENTLY worth the overhead?

29. Write a complex reporting query (3+ CTEs, multiple window functions) and then rewrite it to use temp tables. Measure the performance difference.

30. **Practical challenge:** You inherit a slow query from a colleague:
```sql
SELECT c.customer_name, r.restaurant_name, o.order_date, o.total_amount,
  (SELECT AVG(total_amount) FROM orders o2 WHERE o2.restaurant_id = o.restaurant_id) AS rest_avg,
  (SELECT COUNT(*) FROM orders o3 WHERE o3.customer_id = o.customer_id) AS cust_order_count,
  (SELECT SUM(total_amount) FROM orders o4 WHERE o4.customer_id = o.customer_id) AS cust_total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE o.order_date >= '2026-01-01'
ORDER BY o.total_amount DESC;
```
Rewrite this query to be 10× faster. Show the before/after EXPLAIN ANALYZE.

---

### ✅ Afternoon Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 16:**
```sql
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', o.order_date) AS month,
        r.restaurant_name,
        SUM(o.total_amount) AS revenue
    FROM orders o
    JOIN restaurants r ON o.restaurant_id = r.restaurant_id
    WHERE o.status = 'completed'
    GROUP BY DATE_TRUNC('month', o.order_date), r.restaurant_name
),
monthly_total AS (
    SELECT month, SUM(revenue) AS total_month_revenue
    FROM monthly_revenue
    GROUP BY month
),
with_pct AS (
    SELECT 
        mr.month,
        mr.restaurant_name,
        mr.revenue,
        (mr.revenue / mt.total_month_revenue * 100) AS pct_of_month,
        AVG(mr.revenue) OVER (PARTITION BY mr.month) AS month_avg
    FROM monthly_revenue mr
    JOIN monthly_total mt ON mr.month = mt.month
)
SELECT TO_CHAR(month, 'YYYY-MM') AS month,
       restaurant_name,
       revenue,
       ROUND(pct_of_month::numeric, 2) AS pct_of_total,
       month_avg
FROM with_pct
WHERE revenue > month_avg
ORDER BY month, revenue DESC;
```

**Answer 17:**
```sql
CREATE TEMP TABLE tmp_monthly_revenue AS
SELECT 
    DATE_TRUNC('month', o.order_date) AS month,
    r.restaurant_name,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE o.status = 'completed'
GROUP BY DATE_TRUNC('month', o.order_date), r.restaurant_name;

CREATE INDEX idx_tmp_mr ON tmp_monthly_revenue(month, restaurant_name);

-- Then query with joins and aggregation
-- The temp table approach can be faster when the CTE result is large and reused
```

**Answer 18:**
```sql
-- Page 1
SELECT order_id, customer_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC, order_id DESC
LIMIT 50;

-- Page 2 (use last values from page 1)
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE (order_date, order_id) < ('2026-06-01 14:30:00', 4500)  -- replace with actual last values
ORDER BY order_date DESC, order_id DESC
LIMIT 50;

-- Pages 3-5: continue with the last row's values from previous page
```

**Answer 19:**
```sql
CREATE MATERIALIZED VIEW mv_customer_summary AS
SELECT 
    c.customer_name,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.total_amount) AS total_spent,
    MODE() WITHIN GROUP (ORDER BY r.restaurant_name) AS favorite_restaurant,
    MODE() WITHIN GROUP (ORDER BY r.cuisine_type) AS favorite_cuisine
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
GROUP BY c.customer_name;

REFRESH MATERIALIZED VIEW mv_customer_summary;

SELECT * FROM mv_customer_summary ORDER BY total_spent DESC LIMIT 10;
```

**Answer 20:**
```sql
WITH cust_window AS (
    SELECT 
        order_id, customer_id,
        SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total_cust
    FROM orders
),
rest_window AS (
    SELECT 
        order_id, restaurant_id,
        SUM(total_amount) OVER (PARTITION BY restaurant_id ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total_rest,
        RANK() OVER (PARTITION BY restaurant_id ORDER BY total_amount DESC) AS price_rank_rest
    FROM orders
)
SELECT 
    o.order_id, o.customer_id, o.restaurant_id, o.total_amount,
    cw.running_total_cust,
    rw.running_total_rest,
    rw.price_rank_rest
FROM orders o
JOIN cust_window cw ON o.order_id = cw.order_id
JOIN rest_window rw ON o.order_id = rw.order_id;
```

**Answer 21:**
```sql
DO $$
DECLARE
    batch_size INT := 5000;
    affected INT := 1;
    total_affected INT := 0;
BEGIN
    WHILE affected > 0 LOOP
        UPDATE orders 
        SET status = 'archived' 
        WHERE order_id IN (
            SELECT order_id FROM orders 
            WHERE order_date < '2025-06-01' AND status != 'archived'
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        );
        GET DIAGNOSTICS affected = ROW_COUNT;
        total_affected := total_affected + affected;
        RAISE NOTICE 'Batch: updated % rows (total: %)', affected, total_affected;
        COMMIT;
    END LOOP;
END $$;
```

**Answer 22:** Without a unique index, you get:
```
ERROR: cannot refresh materialized view "..." concurrently
HINT: Create a unique index with no WHERE clause on the materialized view.
```

**Answer 23:**
```sql
SELECT 
    customer_id,
    COUNT(*) FILTER (WHERE status = 'completed') AS completed_orders,
    COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled_orders,
    AVG(total_amount) FILTER (WHERE status = 'completed') AS avg_completed
FROM orders
GROUP BY customer_id;
```
FILTER is cleaner and often faster than CASE WHEN.

**Answer 24:**
```sql
WITH ranked AS (
    SELECT 
        c.customer_name,
        SUM(o.total_amount) AS total_spent,
        RANK() OVER (ORDER BY SUM(o.total_amount) DESC) AS spending_rank,
        LEAD(SUM(o.total_amount)) OVER (ORDER BY SUM(o.total_amount) DESC) AS next_customer_spent
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_id, c.customer_name
)
SELECT 
    customer_name,
    total_spent,
    spending_rank,
    total_spent - COALESCE(next_customer_spent, 0) AS gap_to_next
FROM ranked
WHERE spending_rank <= 10
ORDER BY spending_rank;
```

**Answer 25:**
```sql
-- LATERAL approach
SELECT r.restaurant_name, latest.*
FROM restaurants r
CROSS JOIN LATERAL (
    SELECT order_id, order_date, total_amount
    FROM orders o
    WHERE o.restaurant_id = r.restaurant_id
    ORDER BY o.order_date DESC
    LIMIT 3
) latest
ORDER BY r.restaurant_name, latest.order_date DESC;

-- Window function approach
WITH ranked AS (
    SELECT 
        o.*,
        ROW_NUMBER() OVER (PARTITION BY o.restaurant_id ORDER BY o.order_date DESC) AS rn
    FROM orders o
)
SELECT r.restaurant_name, rk.order_id, rk.order_date, rk.total_amount
FROM ranked rk
JOIN restaurants r ON rk.restaurant_id = r.restaurant_id
WHERE rk.rn <= 3
ORDER BY r.restaurant_name, rk.order_date DESC;
```
LATERAL is often faster when restaurants >> orders per restaurant, because it doesn't need to rank ALL orders.

**Answer 26:** Three strategies ranked:
1. **Materialized view** — pre-compute aggregations, refresh periodically
2. **Covering indexes** — avoid table lookups for common columns
3. **Query rewrite** — eliminate correlated subqueries, reduce JOINs

**Answer 27:**
```sql
SELECT 
    DATE(order_date) AS day,
    SUM(total_amount) AS daily_revenue,
    AVG(SUM(total_amount)) OVER (
        ORDER BY DATE(order_date) 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_avg
FROM orders
WHERE status = 'completed'
GROUP BY DATE(order_date)
ORDER BY day;
```
This is already optimized — the window only looks at 7 rows per step. Ensure there's an index on `orders(order_date)`.

**Answer 28:** 
- Regular REFRESH: Locks the view, blocks reads, faster to build
- CONCURRENTLY: Allows reads during refresh, requires unique index, slower to build
- Use CONCURRENTLY when: dashboard queries can't tolerate downtime (most cases)
- Use regular when: off-hours refresh or no concurrent readers

**Answer 29:** Generally, for complex queries on large datasets, temp tables with indexes outperform CTEs because:
- CTEs may be inlined (no materialization benefit)
- Temp tables allow index creation
- Temp tables break the query into smaller, optimizable chunks

**Answer 30 — Optimized:**
```sql
-- Replace 3 correlated subqueries with JOINs
SELECT 
    c.customer_name, 
    r.restaurant_name, 
    o.order_date, 
    o.total_amount,
    rest_stats.rest_avg,
    cust_stats.cust_order_count,
    cust_stats.cust_total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
JOIN (
    SELECT restaurant_id, AVG(total_amount) AS rest_avg
    FROM orders GROUP BY restaurant_id
) rest_stats ON o.restaurant_id = rest_stats.restaurant_id
JOIN (
    SELECT customer_id, COUNT(*) AS cust_order_count, SUM(total_amount) AS cust_total
    FROM orders GROUP BY customer_id
) cust_stats ON o.customer_id = cust_stats.customer_id
WHERE o.order_date >= '2026-01-01'
ORDER BY o.total_amount DESC;
```
Key improvement: 3 correlated subqueries (each scanning orders table per row) → 2 pre-computed subqueries (each scanning once). This changes O(n×m) to O(n+m).

</details>

---

## 🌙 Evening Block (1 hour): Review & Git Commit

### Concept 9: Interview Optimization Cheat Sheet

Memorize this decision tree for optimizing slow queries:

```
Slow query?
├── Seq Scan on large table?
│   ├── Add index on WHERE/JOIN columns
│   └── Consider partial index if filtering on constant
├── Nested Loop with many rows?
│   ├── Ensure join columns are indexed
│   └── Consider work_mem increase for hash join
├── Sort spilling to disk?
│   ├── Increase work_mem
│   └── Add index matching ORDER BY
├── Correlated subqueries?
│   └── Rewrite as JOINs
├── Multiple window functions?
│   └── Split into CTEs with same PARTITION BY
├── OFFSET pagination?
│   └── Switch to keyset pagination
└── Dashboard query running repeatedly?
    └── Create materialized view
```

### 📝 Today's Checklist

- [ ] I can read an EXPLAIN plan bottom-to-top and identify bottlenecks
- [ ] I understand the difference between Index Scan, Seq Scan, and Index Only Scan
- [ ] I can write keyset pagination instead of OFFSET
- [ ] I know when to use CTEs vs temp tables vs materialized views
- [ ] I can optimize correlated subqueries by rewriting as JOINs
- [ ] I completed at least 20 of the 30 exercises
- [ ] I pushed any practice code to GitHub

### 📖 Further Reading (Free)

- [PostgreSQL Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)
- [Use The Index, Luke — Chapter on Joins](https://use-the-index-luke.com/sql/join)
- [PGMustard EXPLAIN Guide](https://www.pgmustard.com/docs/explain) — visual EXPLAIN interpretation

---

*Day 23 complete! Tomorrow: Star Schema Deep Dive — the backbone of every data warehouse.* 🏗️
