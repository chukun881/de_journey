# 📅 Day 3 — Saturday, 17 May 2026
# SQL: Subqueries, CTEs (WITH clause), HAVING, CASE WHEN

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Write subqueries (queries inside queries)
- Use CTEs (Common Table Expressions) with WITH — cleaner than subqueries
- Use HAVING to filter groups (vs WHERE which filters rows)
- Use CASE WHEN for conditional logic in SQL
- Combine all of these to solve complex business questions

**Why this matters:** Simple SELECT + JOIN gets you 60% of the way. Subqueries, CTEs, and CASE WHEN get you to 90%. These are the tools that let you answer questions like "show me employees who earn above their department's average" or "categorize customers into VIP/regular/new based on spending."

---

## ☀️ BLOCK 1: Subqueries (Morning, ~1.5 hours)

---

### Task 1: What is a Subquery? (15 min)

A subquery is a query inside another query. Think of it as "first get this data, then use it."

```sql
-- 🔹 Basic subquery in WHERE
-- "Who earns more than the company average?"
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
-- Step 1: inner query calculates average salary
-- Step 2: outer query uses that number to filter

-- 🔹 Subquery with IN
-- "Who works on the Data Platform v2 project?"
SELECT name
FROM employees
WHERE emp_id IN (
    SELECT emp_id FROM employee_project
    WHERE project_id = (SELECT project_id FROM projects WHERE project_name = 'Data Platform v2')
);
-- Three levels! Inner → middle → outer
```

> 🧠 **How to read subqueries:** Always read from the INSIDE out. The innermost query runs first, its result feeds into the next level, and so on.

---

### Task 2: Types of Subqueries (45 min)

**Type 1: Scalar subquery (returns one value)**

```sql
-- "Show each employee's salary and how it compares to company average"
SELECT name, salary,
       salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees
ORDER BY salary DESC;
```

**Type 2: Subquery with IN (returns a list of values)**

```sql
-- "Find employees who work on multiple projects"
SELECT name
FROM employees
WHERE emp_id IN (
    SELECT emp_id
    FROM employee_project
    GROUP BY emp_id
    HAVING COUNT(*) > 1
);
```

**Type 3: Subquery in FROM (returns a table — also called a "derived table")**

```sql
-- "Show department average salary, but only for departments with 3+ employees"
SELECT dept_summary.dept_name, dept_summary.avg_salary
FROM (
    SELECT d.dept_name, AVG(e.salary) AS avg_salary, COUNT(*) AS emp_count
    FROM employees e
    INNER JOIN departments d ON e.dept_id = d.dept_id
    GROUP BY d.dept_name
) dept_summary
WHERE dept_summary.emp_count >= 3;
-- The subquery creates a temporary "table" called dept_summary
-- Then we query FROM that temporary table
```

**Type 4: EXISTS (checks if any rows match)**

```sql
-- "Find departments that have at least one employee earning above 9000"
SELECT d.dept_name
FROM departments d
WHERE EXISTS (
    SELECT 1 FROM employees e
    WHERE e.dept_id = d.dept_id AND e.salary > 9000
);
-- EXISTS returns TRUE if the subquery finds ANY rows, FALSE if empty
-- It's often faster than IN for large datasets because it stops at the first match
```

**🔥 Practice exercises (use the company database from Day 2):**

1. Find employees who earn more than the average salary of their OWN department (not the company average — hint: you need a correlated subquery where the inner query references the outer query's dept_id)
2. Find the highest-paid employee in each department (show name, salary, dept_name)
3. Find departments where the total salary expenditure is above the overall average department expenditure
4. Find projects that have NO Engineering employees assigned
5. Find the second-highest salary in the company (tricky! hint: use LIMIT and OFFSET, or a subquery)

<details>
<summary>📖 Answers</summary>

```sql
-- 1 (correlated subquery — inner query references outer)
SELECT e.name, e.salary, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.dept_id = e.dept_id
);

-- 2
SELECT e.name, e.salary, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary = (
    SELECT MAX(e2.salary)
    FROM employees e2
    WHERE e2.dept_id = e.dept_id
);

-- 3
SELECT d.dept_name, SUM(e.salary) AS total_salary
FROM departments d
INNER JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_name
HAVING SUM(e.salary) > (
    SELECT AVG(dept_total)
    FROM (
        SELECT SUM(e3.salary) AS dept_total
        FROM employees e3
        GROUP BY e3.dept_id
    ) avg_calc
);

-- 4
SELECT p.project_name
FROM projects p
WHERE NOT EXISTS (
    SELECT 1 FROM employee_project ep
    INNER JOIN employees e ON ep.emp_id = e.emp_id
    WHERE ep.project_id = p.project_id AND e.dept_id = (
        SELECT dept_id FROM departments WHERE dept_name = 'Engineering'
    )
);

-- 5
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
-- Alternative with LIMIT:
-- SELECT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;
```
</details>

---

### Task 3: HAVING — Filtering Groups (30 min)

You know WHERE filters rows BEFORE grouping. HAVING filters groups AFTER grouping.

```sql
-- 🔹 WHERE vs HAVING
-- WHERE: filters individual rows before GROUP BY
-- HAVING: filters the groups after GROUP BY

-- "Show departments with average salary above 7000"
SELECT d.dept_name, AVG(e.salary) AS avg_salary
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name
HAVING AVG(e.salary) > 7000;
-- You CANNOT use WHERE AVG(salary) > 7000 — WHERE can't use aggregates!
-- HAVING exists specifically for filtering on aggregate results

-- 🔹 Full order of execution:
-- SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
-- In SQL, the database processes: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

**The SQL execution order (memorize this):**

```
1. FROM        — which tables
2. JOIN        — how they connect  
3. WHERE       — filter individual rows
4. GROUP BY    — group the remaining rows
5. HAVING      — filter the groups
6. SELECT      — choose columns to show
7. ORDER BY    — sort
8. LIMIT       — restrict rows
```

**Why this matters:** You can't use an alias from SELECT in WHERE (because WHERE runs before SELECT). But you CAN use it in ORDER BY (because ORDER BY runs after SELECT).

```sql
-- ❌ This will ERROR (can't use avg_sal in WHERE):
SELECT dept_id, AVG(salary) AS avg_sal
FROM employees
WHERE avg_sal > 7000  -- ERROR! avg_sal doesn't exist yet when WHERE runs
GROUP BY dept_id;

-- ✅ Correct:
SELECT dept_id, AVG(salary) AS avg_sal
FROM employees
GROUP BY dept_id
HAVING AVG(salary) > 7000;  -- Use HAVING for aggregate filters
```

**🔥 Practice exercises:**

6. Show departments with more than 3 employees
7. Show cities where the total salary paid is above 30,000
8. Show projects with total hours worked above 500
9. Show hire years where more than 4 employees were hired

<details>
<summary>📖 Answers</summary>

```sql
-- 6
SELECT d.dept_name, COUNT(*) AS emp_count
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name
HAVING COUNT(*) > 3;

-- 7
SELECT city, SUM(salary) AS total_salary
FROM employees
GROUP BY city
HAVING SUM(salary) > 30000;

-- 8
SELECT p.project_name, SUM(ep.hours_worked) AS total_hours
FROM projects p
INNER JOIN employee_project ep ON p.project_id = ep.project_id
GROUP BY p.project_name
HAVING SUM(ep.hours_worked) > 500;

-- 9
SELECT EXTRACT(YEAR FROM hire_date) AS hire_year, COUNT(*) AS hire_count
FROM employees
GROUP BY hire_year
HAVING COUNT(*) > 4;
```
</details>

---

## 🔥 BLOCK 2: CTEs and CASE WHEN (Afternoon, ~2 hours)

---

### Task 4: CTEs — WITH Clause (45 min)

A CTE (Common Table Expression) is a named temporary result that you can reference in your main query. Think of it as a "readable subquery."

```sql
-- 🔹 Same problem, two ways to write it:

-- WAY 1: Subquery (hard to read)
SELECT dept_name, avg_salary
FROM (
    SELECT d.dept_name, AVG(e.salary) AS avg_salary
    FROM employees e
    INNER JOIN departments d ON e.dept_id = d.dept_id
    GROUP BY d.dept_name
) dept_avg
WHERE avg_salary > 7000;

-- WAY 2: CTE (much cleaner!)
WITH dept_avg AS (
    SELECT d.dept_name, AVG(e.salary) AS avg_salary
    FROM employees e
    INNER JOIN departments d ON e.dept_id = d.dept_id
    GROUP BY d.dept_name
)
SELECT dept_name, avg_salary
FROM dept_avg
WHERE avg_salary > 7000;
```

> 🧠 **CTE vs Subquery:** They do the same thing. But CTEs are:
> - Easier to read (top-to-bottom instead of inside-out)
> - Reusable (reference the same CTE multiple times)
> - Easier to debug (run the CTE separately to check its output)
> 
> **In data engineering, CTEs are the standard.** Use them. Every production SQL query in dbt, BigQuery, Snowflake uses CTEs.

**Multiple CTEs:**

```sql
-- 🔹 You can chain multiple CTEs
WITH 
-- CTE 1: department stats
dept_stats AS (
    SELECT d.dept_name, 
           COUNT(*) AS emp_count, 
           AVG(e.salary) AS avg_salary
    FROM employees e
    INNER JOIN departments d ON e.dept_id = d.dept_id
    GROUP BY d.dept_name
),
-- CTE 2: project stats  
project_stats AS (
    SELECT p.project_name, 
           SUM(ep.hours_worked) AS total_hours,
           COUNT(DISTINCT ep.emp_id) AS team_size
    FROM projects p
    INNER JOIN employee_project ep ON p.project_id = ep.project_id
    GROUP BY p.project_name
)
-- Main query: use both CTEs
SELECT * FROM dept_stats
UNION ALL
SELECT NULL, NULL, NULL;  -- just to show you can use both
-- (In reality you'd do something meaningful with both)
```

**🔥 Practice exercises (rewrite these using CTEs):**

10. "Show employees who earn more than their department's average" (you did this with subquery in exercise 1 — rewrite with CTE)
11. "Show each employee, their salary, their department average, and whether they're above or below average"
12. "Find the top earner in each city and show how much more they earn than the city average"

<details>
<summary>📖 Answers</summary>

```sql
-- 10
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
SELECT e.name, e.salary, d.dept_name, da.avg_salary
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
INNER JOIN dept_avg da ON e.dept_id = da.dept_id
WHERE e.salary > da.avg_salary;

-- 11
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
SELECT e.name, e.salary, da.avg_salary AS dept_avg,
       e.salary - da.avg_salary AS diff_from_avg
FROM employees e
INNER JOIN dept_avg da ON e.dept_id = da.dept_id
ORDER BY diff_from_avg DESC;

-- 12
WITH city_stats AS (
    SELECT city, 
           AVG(salary) AS avg_salary,
           MAX(salary) AS max_salary
    FROM employees
    GROUP BY city
)
SELECT e.name, e.city, e.salary, cs.avg_salary,
       e.salary - cs.avg_salary AS above_avg
FROM employees e
INNER JOIN city_stats cs ON e.city = cs.city
WHERE e.salary = cs.max_salary;
```
</details>

---

### Task 5: CASE WHEN — Conditional Logic (30 min)

CASE WHEN is SQL's version of if/else. It lets you create new columns based on conditions.

```sql
-- 🔹 Basic CASE WHEN
SELECT name, salary,
    CASE 
        WHEN salary >= 9000 THEN 'Senior'
        WHEN salary >= 7000 THEN 'Mid'
        WHEN salary >= 6000 THEN 'Junior'
        ELSE 'Entry'
    END AS level
FROM employees
ORDER BY salary DESC;

-- 🔹 CASE WHEN with aggregates
-- "Categorize employees by salary tier, then count per tier"
SELECT 
    CASE 
        WHEN salary >= 9000 THEN 'Senior'
        WHEN salary >= 7000 THEN 'Mid'
        WHEN salary >= 6000 THEN 'Junior'
        ELSE 'Entry'
    END AS salary_tier,
    COUNT(*) AS employee_count,
    AVG(salary) AS avg_salary_in_tier
FROM employees
GROUP BY salary_tier
ORDER BY avg_salary_in_tier DESC;

-- 🔹 CASE WHEN for pivoting (turning rows into columns)
-- "Count employees by department and city, with cities as columns"
SELECT d.dept_name,
    COUNT(CASE WHEN e.city = 'Singapore' THEN 1 END) AS singapore_count,
    COUNT(CASE WHEN e.city = 'Kuala Lumpur' THEN 1 END) AS kl_count,
    COUNT(CASE WHEN e.city = 'Penang' THEN 1 END) AS penang_count
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name;
-- This is called a "pivot" — very common in reporting!
```

**🔥 Practice exercises:**

13. Categorize projects as "Large" (budget >= 200k), "Medium" (100k-200k), or "Small" (< 100k). Show project name, budget, and category.
14. Using the e-commerce database from Day 2, categorize orders: "High Value" if total > 200, "Medium" if 100-200, "Low" if < 100. Show order_id, customer name, total value, and category.
15. Create a report showing for each department: how many seniors (salary >= 9000), how many mid (7000-8999), how many juniors (< 7000) — all in one row per department.

<details>
<summary>📖 Answers</summary>

```sql
-- 13
SELECT project_name, budget,
    CASE 
        WHEN budget >= 200000 THEN 'Large'
        WHEN budget >= 100000 THEN 'Medium'
        ELSE 'Small'
    END AS project_size
FROM projects
ORDER BY budget DESC;

-- 14
WITH order_totals AS (
    SELECT o.order_id, c.name AS customer_name,
           SUM(oi.quantity * oi.unit_price) AS total_value
    FROM orders o
    INNER JOIN customers c ON o.customer_id = c.customer_id
    INNER JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.order_id, c.name
)
SELECT order_id, customer_name, total_value,
    CASE 
        WHEN total_value >= 200 THEN 'High Value'
        WHEN total_value >= 100 THEN 'Medium'
        ELSE 'Low'
    END AS value_category
FROM order_totals
ORDER BY total_value DESC;

-- 15
SELECT d.dept_name,
    COUNT(CASE WHEN e.salary >= 9000 THEN 1 END) AS senior_count,
    COUNT(CASE WHEN e.salary >= 7000 AND e.salary < 9000 THEN 1 END) AS mid_count,
    COUNT(CASE WHEN e.salary < 7000 THEN 1 END) AS junior_count
FROM departments d
INNER JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_name;
```
</details>

---

## 🌙 BLOCK 3: Practice + Consolidate (Evening, ~2 hours)

---

### Task 6: HackerRank Problems (45 min)

Complete on https://www.hackerrank.com/domains/sql:

1. **Type of Triangle** — CASE WHEN
2. **The PADS** — string concatenation + CASE WHEN
3. **Occupations** — pivot using CASE WHEN (harder)
4. **New Companies** — multiple JOINs with COUNT

---

### Task 7: The Big Challenge — Business Report (45 min)

Using the e-commerce database from Day 2, write a single query that produces this report:

```
Customer Report:
- Customer name
- City
- Total orders placed
- Total money spent
- Average order value
- Customer tier: "VIP" if total spent >= 300, "Regular" if >= 100, "New" if < 100
- Favorite category (the category they bought the most items from)
```

<details>
<summary>📖 Hint: Break it into CTEs</summary>

```sql
WITH order_totals AS (
    -- Calculate total spent per order
),
customer_spending AS (
    -- Aggregate per customer: total spent, order count, avg
),
category_prefs AS (
    -- Find each customer's most-bought category
)
SELECT ... FROM customer_spending cs
LEFT JOIN category_prefs cp ON ...
```
</details>

<details>
<summary>📖 Full Answer</summary>

```sql
WITH order_totals AS (
    SELECT o.order_id, o.customer_id,
           SUM(oi.quantity * oi.unit_price) AS order_value
    FROM orders o
    INNER JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.order_id, o.customer_id
),
customer_spending AS (
    SELECT customer_id,
           COUNT(*) AS total_orders,
           SUM(order_value) AS total_spent,
           ROUND(AVG(order_value), 2) AS avg_order_value
    FROM order_totals
    GROUP BY customer_id
),
category_counts AS (
    SELECT o.customer_id, p.category, SUM(oi.quantity) AS total_qty,
           ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY SUM(oi.quantity) DESC) AS rn
    FROM orders o
    INNER JOIN order_items oi ON o.order_id = oi.order_id
    INNER JOIN products p ON oi.product_id = p.product_id
    GROUP BY o.customer_id, p.category
)
SELECT c.name, c.city,
       COALESCE(cs.total_orders, 0) AS total_orders,
       COALESCE(cs.total_spent, 0) AS total_spent,
       COALESCE(cs.avg_order_value, 0) AS avg_order_value,
    CASE 
        WHEN cs.total_spent >= 300 THEN 'VIP'
        WHEN cs.total_spent >= 100 THEN 'Regular'
        ELSE 'New'
    END AS customer_tier,
    cc.category AS favorite_category
FROM customers c
LEFT JOIN customer_spending cs ON c.customer_id = cs.customer_id
LEFT JOIN category_counts cc ON c.customer_id = cc.customer_id AND cc.rn = 1
ORDER BY cs.total_spent DESC NULLS LAST;
```

Note: This uses `ROW_NUMBER()` which you'll learn more about tomorrow (window functions). Don't worry if it's new — just see how it fits together.
</details>

---

### Task 8: SQLZoo (30 min)

Complete on https://sqlzoo.net:

- [ ] **Section 3: SELECT from Nobel** — practice subqueries
- [ ] **Section 8: Using Null** — COALESCE, IS NULL practice

---

### Task 9: Daily Reflection + GitHub Push (15 min)

Create `sql-practice/day03-notes.md`:

```markdown
# Day 3 — Subqueries, CTEs, HAVING, CASE WHEN
Date: 2026-05-17

## Key Concepts
- Subquery = 
- CTE = 
- HAVING = 
- CASE WHEN = 
- SQL execution order: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT

## What I Found Tricky
- 

## Favorite Query Today
```sql
```

## Tomorrow's Goal: Window Functions
```

```bash
git add .
git commit -m "Day 3: Subqueries, CTEs, HAVING, CASE WHEN"
git push
```

---

## ✅ Day 3 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Can explain subquery types (scalar, IN, FROM, EXISTS) | ☐ |
| 2 | Solved 5 subquery exercises | ☐ |
| 3 | Understand HAVING vs WHERE | ☐ |
| 4 | Solved 4 HAVING exercises | ☐ |
| 5 | Wrote CTEs with WITH clause | ☐ |
| 6 | Solved 3 CTE exercises | ☐ |
| 7 | Wrote CASE WHEN for categorization | ☐ |
| 8 | Solved 3 CASE WHEN exercises | ☐ |
| 9 | Solved 4 HackerRank problems | ☐ |
| 10 | Completed the Big Challenge report | ☐ |
| 11 | Completed SQLZoo Sections 3 + 8 | ☐ |
| 12 | Day 3 notes pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card

```
-- Subquery
SELECT * FROM table WHERE col = (SELECT ...)

-- CTE
WITH name AS (SELECT ...)
SELECT * FROM name;

-- HAVING (filter after GROUP BY)
SELECT dept, COUNT(*) FROM employees
GROUP BY dept HAVING COUNT(*) > 3;

-- CASE WHEN
SELECT name,
  CASE WHEN salary >= 9000 THEN 'Senior'
       WHEN salary >= 7000 THEN 'Mid'
       ELSE 'Junior' END AS level
FROM employees;
```
