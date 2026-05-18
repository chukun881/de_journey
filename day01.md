# 📅 Day 1 — Friday, 15 May 2026
# SQL Foundation: SELECT, WHERE, ORDER BY, LIMIT, DISTINCT

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Read ANY basic SQL query and understand what it does
- Write queries to SELECT, FILTER, SORT, and LIMIT data
- Explain what a database, table, row, column, and schema are
- Run SQL in both PostgreSQL (local) and Neon (cloud)

**Why this matters:** SQL is the #1 skill for data engineers. You will use it every single day at work. Even senior engineers with 10 years experience still write SQL daily. This is not a "learn and move on" skill — this is your bread and butter.

---

## ☀️ BLOCK 1: Setup + Theory

---

### Task 1: Setup Neon Cloud Database

Neon is a serverless PostgreSQL — your database lives in the cloud, accessible from any browser.

**Steps:**
1. Go to https://console.neon.tech
2. Sign up (use GitHub or Google)
3. Click "Create Project" → name it `de-learning`
4. Select region closest to you (Singapore if available, otherwise closest)
5. Once created, you'll see:
   - **Connection string** (looks like `postgresql://username:password@ep-xxx.neon.tech/neondb`)
   - **Dashboard** with your database stats
6. Click "SQL Editor" in the left sidebar
7. You should see an empty query editor — this is where you'll write SQL

✅ **Check:** Type `SELECT 1;` and hit Run. If you get a result showing `1`, your Neon is working.

**Also setup your local PostgreSQL:**
- Open pgAdmin 4 from your Start menu / Applications
- Connect to your local server (password you set during install)
- Open Query Tool (right-click your database → Query Tool)

> 💡 **Why both?** Use Neon when you're on someone else's laptop. Use local when you're on your own machine. The SQL you write is **identical** — PostgreSQL is PostgreSQL whether it's local or cloud.

---

### Task 2: Understand What a Database Actually Is

Before writing code, you need to understand what you're working with.

**A database is like a filing cabinet.** Here's the analogy:

```
📁 Filing Cabinet = DATABASE
  📂 Manila Folder = TABLE
    📄 One Sheet of Paper = ROW (one record)
      📋 The columns on that sheet = COLUMNS (fields)
```

**Real example — a company's database:**

```
DATABASE: company_db
│
├── TABLE: employees
│   ├── ROW 1: id=1, name="Alice", department="Engineering", salary=8500
│   ├── ROW 2: id=2, name="Bob", department="Marketing", salary=6200
│   └── ROW 3: id=3, name="Charlie", department="Engineering", salary=9200
│
├── TABLE: departments
│   ├── ROW 1: id=1, name="Engineering", budget=500000, head_count=45
│   └── ROW 2: id=2, name="Marketing", budget=300000, head_count=20
│
└── TABLE: sales
    ├── ROW 1: id=1, product="Widget A", amount=150.00, date="2024-01-15"
    └── ROW 2: id=2, product="Widget B", amount=230.00, date="2024-01-16"
```

**Key terms you MUST know (memorize these):**

| Term | What it is | Example |
|------|-----------|---------|
| **Database** | A collection of related tables | `company_db` |
| **Table** | A structured set of data with rows and columns | `employees` |
| **Column (Field)** | A specific attribute, like a category | `name`, `salary`, `department` |
| **Row (Record)** | One complete entry in a table | Alice's entire employee record |
| **Primary Key** | A column that uniquely identifies each row | `id` (no two employees have same id) |
| **Schema** | The structure/blueprint of your tables | "employees has columns: id, name, department, salary" |
| **Query** | A question you ask the database using SQL | `SELECT * FROM employees` |
| **Result Set** | The data your query returns | The table of results you see after running a query |
| **NULL** | Means "no value" or "unknown" — NOT the same as 0 or empty string | If Charlie has no phone number, phone = NULL |

---

### Task 3: What is SQL?

**SQL = Structured Query Language**

It's the language you use to talk to relational databases. Think of it as English-like commands that tell the database what data you want.

**There are 5 main things you do with SQL:**

| Action | Type | Example |
|--------|------|---------|
| **Read data** | SELECT | `SELECT name FROM employees` |
| **Insert data** | INSERT | `INSERT INTO employees (name) VALUES ('Dave')` |
| **Update data** | UPDATE | `UPDATE employees SET salary = 7000 WHERE name = 'Dave'` |
| **Delete data** | DELETE | `DELETE FROM employees WHERE name = 'Dave'` |
| **Create structure** | CREATE | `CREATE TABLE employees (...)` |

Today we focus on **SELECT** — reading data. This is 80% of what a data engineer does.

> ⚠️ **Important:** SQL keywords are NOT case-sensitive. `SELECT` = `select` = `SeLeCt`. But we write them in UPPERCASE by convention so humans can read them easily. Table and column names ARE case-sensitive in some databases, so be careful.

---

## 🔥 BLOCK 2: Hands-On SQL — Build & Query

---

### Task 4: Create Your Practice Table

Open your SQL editor (Neon or pgAdmin). Copy and run this ENTIRE block:

```sql
-- ============================================
-- STEP 1: Create the table (the blueprint)
-- ============================================
-- This tells PostgreSQL: "Create a table called employees with these columns"

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,        -- SERIAL = auto-incrementing number (1, 2, 3...)
    name VARCHAR(100),            -- VARCHAR(100) = text up to 100 characters
    department VARCHAR(50),       -- department name, text up to 50 chars
    salary DECIMAL(10,2),         -- DECIMAL(10,2) = number with 2 decimal places, max 10 digits total
    hire_date DATE,               -- DATE = a date (year-month-day)
    manager_id INTEGER,           -- reference to another employee's id (we'll use this later for JOINs)
    city VARCHAR(50)              -- which city they work in
);

-- What each line means:
-- id SERIAL PRIMARY KEY → Every employee gets a unique number automatically (1, 2, 3...)
--                         PRIMARY KEY means: no duplicates, cannot be NULL
--                         This is how we uniquely identify each employee
-- name VARCHAR(100)     → Text field, stores names like "Alice Tan"
-- salary DECIMAL(10,2)  → Stores money, like 8500.00 (up to 8 digits before decimal, 2 after)
-- hire_date DATE        → Stores dates like '2023-01-15' (year-month-day format)
```

Now run this to add data:

```sql
-- ============================================
-- STEP 2: Insert sample data (the actual records)
-- ============================================

INSERT INTO employees (name, department, salary, hire_date, manager_id, city) VALUES
('Alice Tan', 'Engineering', 8500.00, '2023-01-15', NULL, 'Singapore'),
('Bob Lim', 'Marketing', 6200.00, '2023-03-20', 5, 'Kuala Lumpur'),
('Charlie Wong', 'Engineering', 9200.00, '2022-11-01', NULL, 'Singapore'),
('Diana Chen', 'HR', 5800.00, '2024-02-10', 7, 'Singapore'),
('Eve Rahman', 'Marketing', 7100.00, '2023-07-05', NULL, 'Kuala Lumpur'),
('Frank Kumar', 'Engineering', 10500.00, '2021-09-12', NULL, 'Singapore'),
('Grace Lee', 'HR', 6400.00, '2023-05-18', NULL, 'Penang'),
('Hank Tan', 'Engineering', 7800.00, '2024-01-03', 1, 'Singapore'),
('Ivy Ng', 'Marketing', 6900.00, '2023-09-22', 5, 'Kuala Lumpur'),
('Jack Goh', 'Engineering', 8200.00, '2022-06-15', 3, 'Singapore'),
('Kate Phang', 'HR', 5600.00, '2024-03-01', 7, 'Penang'),
('Leo Chang', 'Engineering', 9800.00, '2021-12-20', NULL, 'Singapore'),
('Maya Singh', 'Marketing', 7500.00, '2022-08-10', NULL, 'Kuala Lumpur'),
('Nick Ooi', 'Engineering', 7200.00, '2024-04-15', 1, 'Singapore'),
('Olivia Tan', 'HR', 6100.00, '2023-11-30', 7, 'Singapore');
```

✅ **Check:** Run this to verify:
```sql
SELECT * FROM employees;
```
You should see 15 rows of employee data.

---

### Task 5: SELECT — Reading Data

The most basic SQL query pattern:

```sql
SELECT column1, column2, ...
FROM table_name;
```

**Practice each of these. Type them out (don't copy-paste — typing builds muscle memory):**

```sql
-- 🔹 Get EVERYTHING from the table
SELECT * FROM employees;
-- The * means "all columns"
-- ⚠️ In real work, avoid SELECT * on large tables — it's slow and expensive

-- 🔹 Get only specific columns (better practice)
SELECT name, salary FROM employees;

-- 🔹 Get name, department, and city
SELECT name, department, city FROM employees;

-- 🔹 You can also do math on columns
SELECT name, salary, salary * 12 AS annual_salary FROM employees;
-- "AS" creates an alias — renames the column in your results
```

> 🧠 **Why not always use SELECT *?**
> Imagine a table with 50 columns and 10 million rows. SELECT * means you're pulling ALL 50 columns × 10M rows. That's slow, expensive (cloud databases charge by data scanned), and you probably only need 3-4 columns. Always select only what you need.

**🔥 Try these yourself (write the query, don't look at hints):**

1. Show only the `name` and `hire_date` for all employees
2. Show `name`, `salary`, and a new column called `monthly_tax` that is 10% of salary (hint: `salary * 0.10`)
3. Show `name` and `department` for all employees, but rename the department column to `dept` using AS

<details>
<summary>📖 Answers (try yourself first!)</summary>

```sql
-- 1
SELECT name, hire_date FROM employees;

-- 2
SELECT name, salary, salary * 0.10 AS monthly_tax FROM employees;

-- 3
SELECT name, department AS dept FROM employees;
```
</details>

---

### Task 6: WHERE — Filtering Data

SELECT gives you everything. WHERE lets you filter — "give me ONLY the rows that match this condition."

```sql
SELECT column1, column2
FROM table_name
WHERE condition;
```

**Comparison operators:**

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Equal to | `WHERE department = 'Engineering'` |
| `!=` or `<>` | Not equal to | `WHERE city != 'Singapore'` |
| `>` | Greater than | `WHERE salary > 8000` |
| `<` | Less than | `WHERE salary < 6000` |
| `>=` | Greater than or equal | `WHERE salary >= 7000` |
| `<=` | Less than or equal | `WHERE salary <= 6500` |

**Practice — type each one and understand the result:**

```sql
-- 🔹 Exact match
SELECT name, salary FROM employees WHERE department = 'Engineering';

-- 🔹 Numeric comparison
SELECT name, salary FROM employees WHERE salary > 8000;

-- 🔹 Date comparison
SELECT name, hire_date FROM employees WHERE hire_date > '2023-06-01';
-- Dates are compared in YYYY-MM-DD format

-- 🔹 Not equal
SELECT name, city FROM employees WHERE city != 'Singapore';
```

**Combining conditions with AND / OR:**

```sql
-- 🔹 AND = both conditions must be true
SELECT name, salary, city
FROM employees
WHERE department = 'Engineering' AND salary > 8500;
-- "Give me engineers earning more than 8500"

-- 🔹 OR = at least one condition must be true
SELECT name, department, city
FROM employees
WHERE city = 'Singapore' OR city = 'Kuala Lumpur';
-- "Give me employees in SG or KL"

-- 🔹 You can combine AND + OR (use parentheses!)
SELECT name, department, salary
FROM employees
WHERE (department = 'Engineering' OR department = 'Marketing')
  AND salary > 7000;
-- ⚠️ Parentheses matter! This means:
--   (Eng OR Marketing) AND salary > 7000
-- Without parentheses, AND binds tighter than OR, which could give wrong results
```

**Special operators:**

```sql
-- 🔹 IN = shorthand for multiple OR conditions
SELECT name, city FROM employees
WHERE city IN ('Singapore', 'Kuala Lumpur');
-- Same as: WHERE city = 'Singapore' OR city = 'Kuala Lumpur'
-- But cleaner when you have many values

-- 🔹 BETWEEN = range (inclusive on both ends)
SELECT name, salary FROM employees
WHERE salary BETWEEN 6000 AND 8000;
-- Same as: WHERE salary >= 6000 AND salary <= 8000

-- 🔹 LIKE = pattern matching with wildcards
SELECT name FROM employees WHERE name LIKE 'A%';
-- % means "any characters" → 'A%' = starts with A
-- '%tan' = ends with 'tan'
-- '%ee%' = contains 'ee' anywhere

-- 🔹 IS NULL / IS NOT NULL = check for missing values
SELECT name, manager_id FROM employees WHERE manager_id IS NULL;
-- NULL means "no value" — these people have no manager (they are managers themselves)

SELECT name, manager_id FROM employees WHERE manager_id IS NOT NULL;
-- These people DO have a manager
```

> 🧠 **Common mistake:** `WHERE column = NULL` does NOT work! You must use `WHERE column IS NULL`. NULL is special — it means "unknown", and you can't compare unknown with =.

**🔥 Practice exercises (write queries yourself):**

4. Find all employees in Kuala Lumpur
5. Find all employees earning between 7000 and 9000
6. Find all HR employees in Singapore
7. Find all employees whose name contains 'Tan'
8. Find all employees who have a manager (manager_id is not null)
9. Find all employees hired in 2024
10. Find all Marketing employees OR anyone earning more than 9000

<details>
<summary>📖 Answers (try yourself first!)</summary>

```sql
-- 4
SELECT * FROM employees WHERE city = 'Kuala Lumpur';

-- 5
SELECT name, salary FROM employees WHERE salary BETWEEN 7000 AND 9000;

-- 6
SELECT name, salary FROM employees WHERE department = 'HR' AND city = 'Singapore';

-- 7
SELECT name FROM employees WHERE name LIKE '%Tan%';

-- 8
SELECT name, manager_id FROM employees WHERE manager_id IS NOT NULL;

-- 9
SELECT name, hire_date FROM employees WHERE hire_date >= '2024-01-01' AND hire_date < '2025-01-01';

-- 10
SELECT name, department, salary FROM employees
WHERE department = 'Marketing' OR salary > 9000;
```
</details>

---

### Task 7: ORDER BY, LIMIT, DISTINCT

```sql
-- 🔹 ORDER BY = sort your results
SELECT name, salary FROM employees ORDER BY salary ASC;
-- ASC = ascending (lowest first, this is the default)
-- DESC = descending (highest first)

SELECT name, salary FROM employees ORDER BY salary DESC;
-- Now highest salary is first

-- 🔹 Sort by multiple columns
SELECT name, department, salary FROM employees
ORDER BY department ASC, salary DESC;
-- First sort by department (A→Z), then within each department, sort by salary (high→low)
-- You'll see Engineering people together, HR together, Marketing together
-- And within each group, highest paid first

-- 🔹 LIMIT = only get first N rows
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 3;
-- "Give me the top 3 highest paid employees"
-- VERY useful in real work: "show me top 10 customers by revenue"

-- 🔹 DISTINCT = remove duplicates
SELECT DISTINCT department FROM employees;
-- Shows only unique department names: Engineering, Marketing, HR
-- Without DISTINCT, you'd see each department repeated for every employee

SELECT DISTINCT city FROM employees;
-- Singapore, Kuala Lumpur, Penang

SELECT DISTINCT department, city FROM employees;
-- Unique combinations of department + city
-- Like: (Engineering, Singapore), (Marketing, Kuala Lumpur), etc.
```

**🔥 Practice exercises:**

11. Show the 5 lowest-paid employees
12. Show all employees sorted by hire date (newest first)
13. Show unique cities in the database
14. Show top 3 highest-paid employees in Engineering
15. Show all employees sorted by city, then by name alphabetically

<details>
<summary>📖 Answers (try yourself first!)</summary>

```sql
-- 11
SELECT name, salary FROM employees ORDER BY salary ASC LIMIT 5;

-- 12
SELECT name, hire_date FROM employees ORDER BY hire_date DESC;

-- 13
SELECT DISTINCT city FROM employees;

-- 14
SELECT name, salary FROM employees
WHERE department = 'Engineering'
ORDER BY salary DESC
LIMIT 3;

-- 15
SELECT name, city FROM employees ORDER BY city ASC, name ASC;
```
</details>

---

### Task 8: COUNT — Your First Aggregate

```sql
-- 🔹 COUNT = how many rows?
SELECT COUNT(*) FROM employees;
-- "How many employees total?" → 15

SELECT COUNT(*) FROM employees WHERE department = 'Engineering';
-- "How many engineers?" → count them yourself and verify!

-- 🔹 COUNT with DISTINCT
SELECT COUNT(DISTINCT department) FROM employees;
-- "How many different departments exist?" → 3

-- 🔹 GROUP BY = count per group (VERY important!)
SELECT department, COUNT(*) AS employee_count
FROM employees
GROUP BY department;
-- "How many employees in each department?"

SELECT city, COUNT(*) AS employee_count
FROM employees
GROUP BY city;
-- "How many employees in each city?"

-- 🔹 GROUP BY with multiple aggregates
SELECT
    department,
    COUNT(*) AS employee_count,
    ROUND(AVG(salary), 2) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary,
    SUM(salary) AS total_salary
FROM employees
GROUP BY department;
-- This is a POWERFUL query — real data engineers write this daily
-- ROUND(value, 2) rounds to 2 decimal places
-- AVG = average, MIN = minimum, MAX = maximum, SUM = total
```

**🔥 Practice exercises:**

16. Count how many employees are in each city
17. Find the average salary in each department
18. Find the highest salary in each city
19. Count how many employees were hired in each year (hint: `EXTRACT(YEAR FROM hire_date)`)

<details>
<summary>📖 Answers (try yourself first!)</summary>

```sql
-- 16
SELECT city, COUNT(*) AS employee_count FROM employees GROUP BY city;

-- 17
SELECT department, ROUND(AVG(salary), 2) AS avg_salary FROM employees GROUP BY department;

-- 18
SELECT city, MAX(salary) AS highest_salary FROM employees GROUP BY city;

-- 19
SELECT EXTRACT(YEAR FROM hire_date) AS hire_year, COUNT(*) AS hires
FROM employees
GROUP BY hire_year
ORDER BY hire_year;
```
</details>

---

## 🌙 BLOCK 3: Practice + Consolidate

---

### Task 9: HackerRank SQL Problems

Go to https://www.hackerrank.com/domains/sql

Complete these specific problems (all under "Basic Select"):

1. **Revising the Select Query I** — filter by population
2. **Revising the Select Query II** — filter with multiple conditions
3. **Select All** — basic SELECT *
4. **Select By ID** — filter by ID
5. **Japanese Cities' Attributes** — filter by country code
6. **Japanese Cities' Names** — select specific columns with filter
7. **Weather Observation Station 1** — basic select
8. **Weather Observation Station 3** — WHERE with odd numbers (use `MOD(id, 2) != 0`)

For each problem:
- [ ] Read the problem carefully (they can be wordy)
- [ ] Write your query
- [ ] Test it
- [ ] If stuck for >10 minutes, look at discussions, but understand the answer before submitting

---

### Task 10: Write Your Own Queries — Mini Scenario

**Scenario:** Your boss (the Head of HR) sends you these questions. Write SQL to answer each one.

Using the `employees` table you created:

1. "How many employees do we have in Singapore?"
2. "Who is our highest-paid employee and what's their salary?"
3. "List all departments and their average salary, sorted highest average first"
4. "Which employees were hired in 2023? I need their names and hire dates."
5. "How many employees report to a manager vs don't have a manager?"
6. "I want to see employees grouped by city AND department — how many in each combination?"
7. "Who are the employees earning above the overall average salary?" (hint: use a subquery `(SELECT AVG(salary) FROM employees)`)

<details>
<summary>📖 Answers</summary>

```sql
-- 1
SELECT COUNT(*) FROM employees WHERE city = 'Singapore';

-- 2
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 1;

-- 3
SELECT department, ROUND(AVG(salary), 2) AS avg_salary
FROM employees GROUP BY department
ORDER BY avg_salary DESC;

-- 4
SELECT name, hire_date FROM employees
WHERE hire_date >= '2023-01-01' AND hire_date < '2024-01-01';

-- 5
SELECT
    CASE WHEN manager_id IS NULL THEN 'No Manager' ELSE 'Has Manager' END AS status,
    COUNT(*)
FROM employees GROUP BY status;

-- 6
SELECT city, department, COUNT(*) AS count
FROM employees GROUP BY city, department
ORDER BY city, department;

-- 7
SELECT name, salary FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```
</details>

---

### Task 11: SQLZoo Homework

Complete these sections on https://sqlzoo.net:

- [ ] **Section 0: SELECT basics** — all 5 questions
- [ ] **Section 1: SELECT name** — all 8 questions

If you finish early, start Section 2 "SELECT from World."

---

### Task 12: Daily Reflection + GitHub Push

Create this file on your GitHub: `sql-practice/day01-notes.md`

Write in your own words:

```markdown
# Day 1 — SQL Basics
Date: 2026-05-15

## 3 Things I Learned Today
1. (write in your own words)
2. 
3. 

## Key SQL Commands I Can Now Use
- SELECT → ...
- WHERE → ...
- ORDER BY → ...
- LIMIT → ...
- DISTINCT → ...
- COUNT → ...
- GROUP BY → ...

## What I Found Confusing
- 

## Query I'm Most Proud Of
```sql
-- paste the query you wrote today that felt the most satisfying
```

## Tomorrow I Want To Learn
- 
```

Push to GitHub:
```bash
cd data-engineering-journey
git add .
git commit -m "Day 1: SQL SELECT, WHERE, ORDER BY, LIMIT, DISTINCT, COUNT, GROUP BY"
git push
```

---

## ✅ Day 1 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Neon account created, can run `SELECT 1` | ☐ |
| 2 | Practice table created with 15 rows | ☐ |
| 3 | Can explain: database, table, row, column, primary key, NULL | ☐ |
| 4 | Wrote all queries in Task 5 (SELECT) | ☐ |
| 5 | Wrote all queries in Task 6 (WHERE, 10 exercises) | ☐ |
| 6 | Wrote all queries in Task 7 (ORDER BY, LIMIT, DISTINCT, 5 exercises) | ☐ |
| 7 | Wrote all queries in Task 8 (COUNT, GROUP BY, 4 exercises) | ☐ |
| 8 | Solved 8 HackerRank problems | ☐ |
| 9 | Completed boss scenario (7 questions) | ☐ |
| 10 | Completed SQLZoo Sections 0 + 1 | ☐ |
| 11 | Day 1 notes written and pushed to GitHub | ☐ |

**Total queries written today: ~40+**

---

## 📌 Quick Reference Card (Screenshot This)

```
SELECT column(s)          -- what columns to show
FROM table                -- which table
WHERE condition           -- filter rows
GROUP BY column           -- group for aggregates
ORDER BY column ASC/DESC  -- sort results
LIMIT n;                  -- only first n rows

Operators: =  !=  >  <  >=  <=
Special: IN  BETWEEN  LIKE  IS NULL  IS NOT NULL
Combine: AND  OR  (use parentheses!)
Aggregates: COUNT(*)  AVG()  SUM()  MIN()  MAX()
```
