# 📅 Day 4 — Sunday, 18 May 2026
# SQL: Window Functions (OVER, ROW_NUMBER, RANK, LAG, LEAD, SUM OVER)

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Understand what window functions are and WHY they exist
- Use ROW_NUMBER, RANK, DENSE_RANK
- Use LAG and LEAD to compare rows
- Use SUM/AVG OVER for running totals and moving averages
- Know the difference between GROUP BY and window functions

**Why this matters:** Window functions are the **#1 differentiator** in SQL interviews. If you can write window functions confidently, you're ahead of most fresh graduates. They're used EVERYWHERE in data engineering — ranking, running totals, year-over-year comparisons, finding the "latest" record, and more.

---

## ☀️ BLOCK 1: Understanding Window Functions (Morning, ~1.5 hours)

---

### Task 1: The Problem Window Functions Solve (20 min)

**Scenario:** Your boss says "Show me each employee's salary AND their department's average salary."

```sql
-- ❌ With GROUP BY, you LOSE individual rows:
SELECT dept_id, AVG(salary) FROM employees GROUP BY dept_id;
-- Result: just one row per department. You lost individual employee data!

-- ✅ With window function, you KEEP individual rows AND add the aggregate:
SELECT name, salary, dept_id,
    AVG(salary) OVER (PARTITION BY dept_id) AS dept_avg
FROM employees;
-- Result: every employee still has their own row, PLUS their dept average
```

**This is the key insight:** GROUP BY collapses rows. Window functions ADD calculations WITHOUT collapsing rows.

```
GROUP BY result:
┌─────────┬────────────┐
│ dept_id │ avg_salary │
├─────────┼────────────┤
│    1    │   8720.00  │  ← only 1 row per dept
│    2    │   6925.00  │
└─────────┴────────────┘

Window function result:
┌─────────┬────────┬─────────┬───────────┐
│ name    │ salary │ dept_id │ dept_avg  │
├─────────┼────────┼─────────┼───────────┤
│ Alice   │ 8500   │    1    │  8720.00  │  ← individual row kept!
│ Charlie │ 9200   │    1    │  8720.00  │  ← same dept_avg for same dept
│ Frank   │ 10500  │    1    │  8720.00  │
│ Bob     │ 6200   │    2    │  6925.00  │
│ Eve     │ 7100   │    2    │  6925.00  │
└─────────┴────────┴─────────┴───────────┘
```

---

### Task 2: Window Function Syntax (15 min)

Every window function follows this pattern:

```sql
FUNCTION_NAME() OVER (
    PARTITION BY column    -- how to group (like GROUP BY but doesn't collapse)
    ORDER BY column        -- how to sort within each partition
    ROWS/RANGE BETWEEN ... -- which rows to include (frame)
)
```

- **PARTITION BY** = "divide data into groups" (optional — if omitted, entire table is one partition)
- **ORDER BY** = "sort within each partition" (required for ranking functions)
- **ROWS/RANGE** = "which nearby rows to include" (optional — defaults vary)

---

### Task 3: ROW_NUMBER, RANK, DENSE_RANK (40 min)

These three functions assign numbers to rows based on sorting. They're crucial for "top N per group" queries.

```sql
-- 🔹 ROW_NUMBER — simple 1, 2, 3, 4 (no ties, always unique)
SELECT name, dept_id, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num
FROM employees;
-- row_num: 1, 2, 3, 4, 5... (always sequential, no gaps)

-- 🔹 RANK — tied values get same rank, then SKIP
SELECT name, dept_id, salary,
    RANK() OVER (ORDER BY salary DESC) AS rank_num
FROM employees;
-- If two people have salary 8500, both get rank 3
-- Next person gets rank 5 (skipped 4)

-- 🔹 DENSE_RANK — tied values get same rank, NO skip
SELECT name, dept_id, salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank_num
FROM employees;
-- If two people have salary 8500, both get rank 3
-- Next person gets rank 4 (no skip)
```

**🔥 The "Top N per Group" pattern — extremely common in interviews:**

```sql
-- "Find the highest-paid employee in EACH department"
-- Step 1: Number employees within each department by salary
WITH ranked AS (
    SELECT e.name, d.dept_name, e.salary,
        ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rn
    FROM employees e
    INNER JOIN departments d ON e.dept_id = d.dept_id
)
-- Step 2: Filter to only rank 1
SELECT name, dept_name, salary
FROM ranked
WHERE rn = 1;
```

> 🧠 **This pattern appears in almost every data engineering interview.** Memorize it:
> 1. Use ROW_NUMBER() with PARTITION BY and ORDER BY
> 2. Wrap in a CTE
> 3. Filter WHERE rn = 1 (or 2, 3 for top N)

**🔥 Practice exercises:**

1. Rank all employees by salary (highest first). Show name, salary, and rank.
2. Find the top 2 highest-paid employees in EACH department.
3. Find the newest hire in each city.
4. Find the employee with the most hours worked on each project.

<details>
<summary>📖 Answers</summary>

```sql
-- 1
SELECT name, salary, RANK() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees;

-- 2
WITH ranked AS (
    SELECT e.name, d.dept_name, e.salary,
        ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rn
    FROM employees e
    INNER JOIN departments d ON e.dept_id = d.dept_id
)
SELECT name, dept_name, salary FROM ranked WHERE rn <= 2;

-- 3
WITH ranked AS (
    SELECT name, city, hire_date,
        ROW_NUMBER() OVER (PARTITION BY city ORDER BY hire_date DESC) AS rn
    FROM employees
)
SELECT name, city, hire_date FROM ranked WHERE rn = 1;

-- 4
WITH ranked AS (
    SELECT e.name, p.project_name, ep.hours_worked,
        ROW_NUMBER() OVER (PARTITION BY ep.project_id ORDER BY ep.hours_worked DESC) AS rn
    FROM employees e
    INNER JOIN employee_project ep ON e.emp_id = ep.emp_id
    INNER JOIN projects p ON ep.project_id = p.project_id
)
SELECT name, project_name, hours_worked FROM ranked WHERE rn = 1;
```
</details>

---

## 🔥 BLOCK 2: LAG, LEAD, Running Totals (Afternoon, ~2 hours)

---

### Task 4: LAG and LEAD — Comparing Rows (30 min)

LAG looks BACK (previous row), LEAD looks FORWARD (next row).

```sql
-- 🔹 LAG — look at the previous row
-- "Show each employee's hire date and the PREVIOUS employee's hire date"
SELECT name, hire_date,
    LAG(hire_date, 1) OVER (ORDER BY hire_date) AS prev_hire_date
FROM employees;
-- LAG(hire_date, 1) = get the hire_date from 1 row before
-- First row has no previous → NULL

-- 🔹 LEAD — look at the next row
SELECT name, hire_date,
    LEAD(hire_date, 1) OVER (ORDER BY hire_date) AS next_hire_date
FROM employees;

-- 🔹 Practical use: month-over-month comparison
-- Let's create some monthly data first:
CREATE TABLE monthly_revenue (
    month DATE,
    revenue DECIMAL(10,2)
);

INSERT INTO monthly_revenue VALUES
('2024-01-01', 45000),
('2024-02-01', 52000),
('2024-03-01', 48000),
('2024-04-01', 61000),
('2024-05-01', 58000),
('2024-06-01', 67000);

-- "Show revenue with previous month and month-over-month change"
SELECT 
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS change,
    ROUND(
        (revenue - LAG(revenue, 1) OVER (ORDER BY month)) * 100.0 
        / LAG(revenue, 1) OVER (ORDER BY month), 
    2) AS pct_change
FROM monthly_revenue;
-- This is EXACTLY what business dashboards show!
-- "Revenue grew 15.56% from Jan to Feb"
```

**🔥 Practice exercises:**

5. Show each employee's salary and the difference from the person who earns just above them (hint: ORDER BY salary ASC, then use LEAD)
6. For each month, show revenue and whether it increased or decreased vs previous month (use CASE WHEN + LAG)
7. Find months where revenue dropped compared to the previous month.

<details>
<summary>📖 Answers</summary>

```sql
-- 5
SELECT name, salary,
    LEAD(salary, 1) OVER (ORDER BY salary ASC) AS next_higher_salary,
    LEAD(salary, 1) OVER (ORDER BY salary ASC) - salary AS gap
FROM employees;

-- 6
SELECT month, revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month,
    CASE 
        WHEN revenue > LAG(revenue, 1) OVER (ORDER BY month) THEN 'Increased'
        WHEN revenue < LAG(revenue, 1) OVER (ORDER BY month) THEN 'Decreased'
        ELSE 'Same'
    END AS trend
FROM monthly_revenue;

-- 7
SELECT month, revenue, 
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month
FROM monthly_revenue
WHERE revenue < LAG(revenue, 1) OVER (ORDER BY month);
```
</details>

---

### Task 5: Running Totals and Moving Averages (30 min)

```sql
-- 🔹 Running total = cumulative sum
SELECT month, revenue,
    SUM(revenue) OVER (ORDER BY month) AS running_total
FROM monthly_revenue;
-- Jan: 45000, Feb: 45000+52000=97000, Mar: 97000+48000=145000...

-- 🔹 Running total with PARTITION BY
-- If you had multiple product lines, you'd do:
-- SUM(revenue) OVER (PARTITION BY product_line ORDER BY month)

-- 🔹 Moving average (last 3 months)
SELECT month, revenue,
    AVG(revenue) OVER (
        ORDER BY month 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3m
FROM monthly_revenue;
-- ROWS BETWEEN 2 PRECEDING AND CURRENT ROW means:
-- "Average this row and the 2 rows before it"
-- Jan: avg(Jan only) = 45000 (no 2 prior rows exist)
-- Feb: avg(Jan, Feb) = 48500
-- Mar: avg(Jan, Feb, Mar) = 48333.33
-- Apr: avg(Feb, Mar, Apr) = 53666.67

-- 🔹 Common frame specifications:
-- ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  = from start to now (default for ORDER BY)
-- ROWS BETWEEN X PRECEDING AND Y FOLLOWING          = window around current row
-- ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING = entire partition
```

**🔥 Practice exercises:**

8. Show each employee's salary and the running total of salaries when ordered by hire date.
9. Show each employee's salary and the cumulative average salary in their department (ordered by hire date).
10. Using monthly_revenue, show a 2-month moving average.

<details>
<summary>📖 Answers</summary>

```sql
-- 8
SELECT name, hire_date, salary,
    SUM(salary) OVER (ORDER BY hire_date) AS running_total
FROM employees;

-- 9
SELECT e.name, d.dept_name, e.salary, e.hire_date,
    ROUND(AVG(e.salary) OVER (
        PARTITION BY e.dept_id ORDER BY e.hire_date
    ), 2) AS dept_cum_avg
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- 10
SELECT month, revenue,
    AVG(revenue) OVER (
        ORDER BY month ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
    ) AS moving_avg_2m
FROM monthly_revenue;
```
</details>

---

### Task 6: More Window Functions (20 min)

```sql
-- 🔹 FIRST_VALUE / LAST_VALUE
-- "What's the first hire date in each department?"
SELECT e.name, d.dept_name, e.hire_date,
    FIRST_VALUE(e.hire_date) OVER (PARTITION BY e.dept_id ORDER BY e.hire_date) AS first_hire
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- 🔹 NTILE — divide rows into N equal groups
-- "Divide employees into 4 quartiles by salary"
SELECT name, salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS salary_quartile
FROM employees;
-- Quartile 1 = top 25%, Quartile 4 = bottom 25%
-- Useful for customer segmentation!

-- 🔹 PERCENT_RANK
SELECT name, salary,
    PERCENT_RANK() OVER (ORDER BY salary ASC) AS pct_rank
FROM employees;
-- 0.0 = lowest salary, 1.0 = highest
-- 0.5 = median area
```

---

## 🌙 BLOCK 3: Practice + Consolidate (Evening, ~1.5 hours)

---

### Task 7: HackerRank Window Function Problems (45 min)

Complete on https://www.hackerrank.com/domains/sql — search for these:

1. **Weather Observation Station 18** — MIN/MAX with window
2. **Weather Observation Station 19** — distance calculation
3. **Top Earners** — find max salary + count (use window or subquery)
4. **The Report** — CASE WHEN + window functions

---

### Task 8: Interview-Style Challenge (30 min)

**Question you might get in an interview:**

> "We have a table of employee salary changes. Write a query to find employees whose salary increased by more than 20% between consecutive changes."

```sql
CREATE TABLE salary_history (
    emp_id INTEGER,
    salary DECIMAL(10,2),
    change_date DATE
);

INSERT INTO salary_history VALUES
(1, 5000, '2023-01-01'),
(1, 5500, '2023-06-01'),
(1, 7000, '2024-01-01'),
(2, 6000, '2023-01-01'),
(2, 6500, '2023-07-01'),
(2, 6600, '2024-01-01'),
(3, 4500, '2023-01-01'),
(3, 5500, '2023-08-01');
```

Try solving it yourself using LAG.

<details>
<summary>📖 Answer</summary>

```sql
WITH salary_changes AS (
    SELECT emp_id, salary, change_date,
        LAG(salary) OVER (PARTITION BY emp_id ORDER BY change_date) AS prev_salary
    FROM salary_history
)
SELECT emp_id, prev_salary, salary, change_date,
    ROUND((salary - prev_salary) * 100.0 / prev_salary, 2) AS pct_increase
FROM salary_changes
WHERE prev_salary IS NOT NULL
  AND (salary - prev_salary) * 100.0 / prev_salary > 20;
-- Employee 1's jump from 5500→7000 is 27.27% > 20 ✅
-- Employee 3's jump from 4500→5500 is 22.22% > 20 ✅
```
</details>

---

### Task 9: Daily Reflection + GitHub Push (15 min)

Create `sql-practice/day04-notes.md`:

```markdown
# Day 4 — Window Functions
Date: 2026-05-18

## Key Window Functions I Now Know
- ROW_NUMBER() → 
- RANK() → 
- DENSE_RANK() → 
- LAG() → 
- LEAD() → 
- SUM() OVER → 
- AVG() OVER → 
- FIRST_VALUE() → 
- NTILE() → 

## The "Top N per Group" Pattern
1. 
2. 
3. 

## GROUP BY vs Window Functions
- GROUP BY → 
- Window Function → 

## What I Found Hard
- 

## Favorite Query
```sql
```
```

```bash
git add .
git commit -m "Day 4: Window functions (ROW_NUMBER, RANK, LAG, LEAD, SUM OVER)"
git push
```

---

## ✅ Day 4 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Can explain window functions vs GROUP BY | ☐ |
| 2 | Used ROW_NUMBER, RANK, DENSE_RANK | ☐ |
| 3 | Solved 4 ranking exercises including "Top N per group" | ☐ |
| 4 | Used LAG and LEAD | ☐ |
| 5 | Solved 3 LAG/LEAD exercises | ☐ |
| 6 | Wrote running totals and moving averages | ☐ |
| 7 | Solved 3 running total exercises | ☐ |
| 8 | Used FIRST_VALUE, NTILE, PERCENT_RANK | ☐ |
| 9 | Solved 4 HackerRank problems | ☐ |
| 10 | Completed interview-style salary challenge | ☐ |
| 11 | Day 4 notes pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card — Window Functions

```
-- Ranking
ROW_NUMBER() OVER (ORDER BY col)        -- 1,2,3,4 (no ties)
RANK() OVER (ORDER BY col)              -- 1,2,2,4 (ties + gap)
DENSE_RANK() OVER (ORDER BY col)        -- 1,2,2,3 (ties, no gap)

-- Comparison
LAG(col, n) OVER (ORDER BY col)         -- n rows back
LEAD(col, n) OVER (ORDER BY col)        -- n rows forward

-- Aggregates
SUM(col) OVER (ORDER BY col)            -- running total
AVG(col) OVER (ORDER BY col ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)  -- moving avg

-- Top N per group (INTERVIEW PATTERN)
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY group_col ORDER BY sort_col DESC) AS rn
    FROM table
)
SELECT * FROM ranked WHERE rn <= N;
```
