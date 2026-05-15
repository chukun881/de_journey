# 📅 Day 2 — Friday, 16 May 2026
# SQL JOINs: Connecting Tables Together

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Understand WHY we need JOINs (real databases have MANY tables)
- Write INNER JOIN, LEFT JOIN, RIGHT JOIN, FULL OUTER JOIN
- Use table aliases to keep queries clean
- Join 3+ tables in a single query
- Understand what a foreign key is

**Why this matters:** In the real world, data is NEVER in one table. A company has separate tables for employees, departments, orders, customers, products. JOINs are how you connect them. This is the #1 thing that separates "I know basic SQL" from "I can actually work with real data." Interviewers ALWAYS test JOINs.

---

## ☀️ BLOCK 1: Setup + Theory (Morning, ~1 hour)

---

### Task 1: Create More Tables (30 min)

Yesterday you had 1 table (`employees`). Real databases have many tables that **relate** to each other. Let's build a mini company database.

Open your SQL editor and run this:

```sql
-- ============================================
-- FIRST: Drop yesterday's employees table so we start fresh
-- ============================================
DROP TABLE IF EXISTS employees CASCADE;

-- ============================================
-- Create a proper multi-table database
-- ============================================

-- TABLE 1: Departments
CREATE TABLE departments (
    dept_id SERIAL PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL,
    budget DECIMAL(12,2),
    location VARCHAR(50)
);

-- TABLE 2: Employees (now with dept_id as FOREIGN KEY)
CREATE TABLE employees (
    emp_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    dept_id INTEGER REFERENCES departments(dept_id),  -- THIS is a foreign key!
    salary DECIMAL(10,2),
    hire_date DATE,
    city VARCHAR(50)
);

-- TABLE 3: Projects
CREATE TABLE projects (
    project_id SERIAL PRIMARY KEY,
    project_name VARCHAR(100) NOT NULL,
    start_date DATE,
    end_date DATE,
    budget DECIMAL(12,2)
);

-- TABLE 4: Employee_Project assignments (many-to-many relationship)
CREATE TABLE employee_project (
    emp_id INTEGER REFERENCES employees(emp_id),
    project_id INTEGER REFERENCES projects(project_id),
    role VARCHAR(50),
    hours_worked DECIMAL(6,2),
    PRIMARY KEY (emp_id, project_id)  -- composite primary key
);

-- ============================================
-- Insert data into departments
-- ============================================
INSERT INTO departments (dept_name, budget, location) VALUES
('Engineering', 500000.00, 'Singapore'),
('Marketing', 300000.00, 'Kuala Lumpur'),
('HR', 200000.00, 'Singapore'),
('Finance', 250000.00, 'Penang'),
('Operations', 180000.00, 'Kuala Lumpur');

-- ============================================
-- Insert data into employees (notice dept_id matches departments)
-- ============================================
INSERT INTO employees (name, email, dept_id, salary, hire_date, city) VALUES
('Alice Tan', 'alice@company.com', 1, 8500.00, '2023-01-15', 'Singapore'),
('Bob Lim', 'bob@company.com', 2, 6200.00, '2023-03-20', 'Kuala Lumpur'),
('Charlie Wong', 'charlie@company.com', 1, 9200.00, '2022-11-01', 'Singapore'),
('Diana Chen', 'diana@company.com', 3, 5800.00, '2024-02-10', 'Singapore'),
('Eve Rahman', 'eve@company.com', 2, 7100.00, '2023-07-05', 'Kuala Lumpur'),
('Frank Kumar', 'frank@company.com', 1, 10500.00, '2021-09-12', 'Singapore'),
('Grace Lee', 'grace@company.com', 3, 6400.00, '2023-05-18', 'Penang'),
('Hank Tan', 'hank@company.com', 1, 7800.00, '2024-01-03', 'Singapore'),
('Ivy Ng', 'ivy@company.com', 2, 6900.00, '2023-09-22', 'Kuala Lumpur'),
('Jack Goh', 'jack@company.com', 1, 8200.00, '2022-06-15', 'Singapore'),
('Kate Phang', 'kate@company.com', 3, 5600.00, '2024-03-01', 'Penang'),
('Leo Chang', 'leo@company.com', 1, 9800.00, '2021-12-20', 'Singapore'),
('Maya Singh', 'maya@company.com', 2, 7500.00, '2022-08-10', 'Kuala Lumpur'),
('Nick Ooi', 'nick@company.com', 1, 7200.00, '2024-04-15', 'Singapore'),
('Olivia Tan', 'olivia@company.com', 3, 6100.00, '2023-11-30', 'Singapore'),
('Paul Lee', 'paul@company.com', 4, 8800.00, '2022-04-05', 'Penang'),
('Quinn Yusof', 'quinn@company.com', 4, 6700.00, '2023-08-14', 'Penang'),
('Rachel Lim', 'rachel@company.com', 5, 5900.00, '2024-01-20', 'Kuala Lumpur'),
('Sam Wong', 'sam@company.com', 5, 6300.00, '2023-06-30', 'Kuala Lumpur');

-- ============================================
-- Insert data into projects
-- ============================================
INSERT INTO projects (project_name, start_date, end_date, budget) VALUES
('Data Platform v2', '2024-01-01', '2024-12-31', 200000.00),
('Mobile App Redesign', '2024-03-01', '2024-09-30', 150000.00),
('Marketing Analytics', '2024-02-15', '2024-08-15', 80000.00),
('HR System Upgrade', '2024-06-01', '2025-01-31', 120000.00),
('Cloud Migration', '2024-04-01', '2025-03-31', 300000.00);

-- ============================================
-- Insert data into employee_project (who works on what)
-- ============================================
INSERT INTO employee_project (emp_id, project_id, role, hours_worked) VALUES
(1, 1, 'Lead Engineer', 450.00),
(3, 1, 'Senior Engineer', 380.00),
(6, 1, 'Architect', 500.00),
(8, 1, 'Junior Engineer', 200.00),
(14, 1, 'Junior Engineer', 180.00),
(2, 2, 'PM', 300.00),
(5, 2, 'Designer', 250.00),
(9, 2, 'Content Writer', 200.00),
(2, 3, 'Lead Analyst', 280.00),
(5, 3, 'Data Analyst', 220.00),
(13, 3, 'Analyst', 200.00),
(4, 4, 'Project Lead', 350.00),
(7, 4, 'HR Specialist', 280.00),
(11, 4, 'Junior HR', 150.00),
(15, 4, 'HR Coordinator', 180.00),
(6, 5, 'Lead Architect', 400.00),
(12, 5, 'Cloud Engineer', 350.00),
(1, 5, 'Data Engineer', 300.00),
(16, 5, 'Finance Lead', 200.00),
(17, 5, 'Finance Analyst', 150.00);
```

✅ **Check:** Run each of these to verify your data:
```sql
SELECT COUNT(*) FROM departments;  -- should be 5
SELECT COUNT(*) FROM employees;    -- should be 19
SELECT COUNT(*) FROM projects;     -- should be 5
SELECT COUNT(*) FROM employee_project; -- should be 20
```

---

### Task 2: Understand Foreign Keys & Relationships (20 min)

Yesterday, your `employees` table had a `department` column with text like "Engineering." That's bad design — what if someone types "Engineerig" by mistake? Or the department name changes?

**The right way:** Use a **Foreign Key** — a column that REFERENCES another table's primary key.

```
departments table:              employees table:
┌─────────┬───────────────┐    ┌────────┬─────────┬─────────┐
│ dept_id │ dept_name     │    │ emp_id │ name    │ dept_id │
├─────────┼───────────────┤    ├────────┼─────────┼─────────┤
│    1    │ Engineering   │◄───│   1    │ Alice   │    1    │
│    2    │ Marketing     │◄───│   2    │ Bob     │    2    │
│    3    │ HR            │    │   3    │ Charlie │    1    │
│    4    │ Finance       │    └────────┴─────────┴─────────┘
│    5    │ Operations    │         dept_id is a FOREIGN KEY
└─────────┴───────────────┘         pointing to departments.dept_id
```

**Three types of relationships:**

| Type | Example | Meaning |
|------|---------|---------|
| **One-to-Many** | Department → Employees | One department has many employees. Each employee belongs to ONE department. |
| **Many-to-Many** | Employees ↔ Projects | One employee can work on many projects. One project has many employees. |
| **One-to-One** | Employee → Employee Detail | One employee has one detailed profile. (rare, we won't focus on this) |

The `employee_project` table is a **junction table** (or bridge table) — it connects employees and projects in a many-to-many relationship.

> 🧠 **Think of it like this:** If you have students and classes, one student takes many classes, and one class has many students. You need a "enrollment" table in the middle to track who is in which class. That's exactly what `employee_project` does.

---

### Task 3: What is a JOIN? (10 min)

A JOIN combines rows from two (or more) tables based on a related column between them.

```
Before JOIN:
employees table:              departments table:
┌────────┬─────────┬─────────┐    ┌─────────┬───────────────┐
│ emp_id │ name    │ dept_id │    │ dept_id │ dept_name     │
├────────┼─────────┼─────────┤    ├─────────┼───────────────┤
│   1    │ Alice   │    1    │    │    1    │ Engineering   │
│   2    │ Bob     │    2    │    │    2    │ Marketing     │
└────────┴─────────┴─────────┘    └─────────┴───────────────┘

After INNER JOIN (on dept_id):
┌────────┬─────────┬─────────┬───────────────┐
│ emp_id │ name    │ dept_id │ dept_name     │
├────────┼─────────┼─────────┼───────────────┤
│   1    │ Alice   │    1    │ Engineering   │
│   2    │ Bob     │    2    │ Marketing     │
└────────┴─────────┴─────────┴───────────────┘
```

The JOIN matches `employees.dept_id = departments.dept_id` and stitches the tables together.

---

## 🔥 BLOCK 2: Hands-On JOINs (Morning, ~2 hours)

---

### Task 4: Table Aliases (10 min)

Before we write JOINs, learn aliases — they make queries readable:

```sql
-- Without alias (verbose):
SELECT employees.name, departments.dept_name
FROM employees
INNER JOIN departments ON employees.dept_id = departments.dept_id;

-- With alias (clean):
SELECT e.name, d.dept_name
FROM employees e                          -- e = alias for employees
INNER JOIN departments d ON e.dept_id = d.dept_id;  -- d = alias for departments
```

> 💡 **Always use aliases in JOINs.** Once you start joining 3-4 tables, typing full table names is painful and error-prone. Single-letter aliases are common: `e` for employees, `d` for departments, `p` for projects.

---

### Task 5: INNER JOIN — The Most Common JOIN (30 min)

INNER JOIN returns only rows that have a match in BOTH tables.

```sql
-- 🔹 Basic INNER JOIN
-- "Show me each employee's name and their department name"
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- 🔹 With more columns
SELECT e.name, d.dept_name, e.salary, d.location
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- 🔹 With WHERE filter
-- "Show me only Engineering employees with their department info"
SELECT e.name, e.salary, d.dept_name, d.location
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Engineering';

-- 🔹 With ORDER BY
-- "Show me all employees sorted by department, then by salary"
SELECT e.name, d.dept_name, e.salary
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, e.salary DESC;
```

> 🧠 **Memory trick:** INNER JOIN = "only the matches." If an employee has no department (dept_id is NULL), they DISAPPEAR from the results. If a department has no employees, it also disappears.

**🔥 Practice exercises:**

1. Show each employee's name, email, and their department's location
2. Show only employees in Kuala Lumpur — include their department name
3. Show employee name, salary, and department budget for Engineering department only
4. Count how many employees are in each department (show dept_name, not dept_id)

<details>
<summary>📖 Answers</summary>

```sql
-- 1
SELECT e.name, e.email, d.location
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- 2
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE e.city = 'Kuala Lumpur';

-- 3
SELECT e.name, e.salary, d.budget AS dept_budget
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Engineering';

-- 4
SELECT d.dept_name, COUNT(*) AS employee_count
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name;
```
</details>

---

### Task 6: LEFT JOIN — Keep Everything from the Left (30 min)

LEFT JOIN returns ALL rows from the LEFT table, and matched rows from the RIGHT table. If there's no match, you get NULL for the right table's columns.

```sql
-- 🔹 LEFT JOIN example
-- "Show me ALL departments, even if they have no employees yet"
SELECT d.dept_name, e.name
FROM departments d                          -- LEFT table
LEFT JOIN employees e ON d.dept_id = e.dept_id;  -- RIGHT table
-- If a department has no employees, e.name will be NULL

-- 🔹 Find departments with NO employees
SELECT d.dept_name
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
WHERE e.emp_id IS NULL;
-- The WHERE e.emp_id IS NULL means "no match found in employees"
-- This is a VERY common interview question!

-- 🔹 With count — show ALL departments including those with 0 employees
SELECT d.dept_name, COUNT(e.emp_id) AS employee_count
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_name;
-- ⚠️ Use COUNT(e.emp_id) not COUNT(*)!
-- COUNT(*) counts rows (including NULLs) = always at least 1
-- COUNT(e.emp_id) counts only non-NULL values = 0 when no employees
```

> 🧠 **When to use LEFT JOIN vs INNER JOIN:**
> - "Show me employees and their departments" → You probably want all employees, even those without a dept → LEFT JOIN
> - "Show me data that EXISTS in both tables" → INNER JOIN
> - In practice: when in doubt, draw it out. Which table might have "orphan" records with no match?

**🔥 Practice exercises:**

5. Show ALL employees and which project(s) they're assigned to (use LEFT JOIN on employee_project). Include employees with no projects.
6. Find employees who are NOT assigned to any project.
7. Show each project with a count of employees working on it. Include projects with 0 employees.

<details>
<summary>📖 Answers</summary>

```sql
-- 5
SELECT e.name, p.project_name
FROM employees e
LEFT JOIN employee_project ep ON e.emp_id = ep.emp_id
LEFT JOIN projects p ON ep.project_id = p.project_id;

-- 6
SELECT e.name
FROM employees e
LEFT JOIN employee_project ep ON e.emp_id = ep.emp_id
WHERE ep.project_id IS NULL;

-- 7
SELECT p.project_name, COUNT(ep.emp_id) AS team_size
FROM projects p
LEFT JOIN employee_project ep ON p.project_id = ep.project_id
GROUP BY p.project_name;
```
</details>

---

### Task 7: RIGHT JOIN and FULL OUTER JOIN (15 min)

```sql
-- 🔹 RIGHT JOIN = same as LEFT JOIN, just flipped
-- "Show me all employees, even those without a department"
SELECT e.name, d.dept_name
FROM departments d
RIGHT JOIN employees e ON d.dept_id = e.dept_id;
-- This gives the same result as LEFT JOIN with tables swapped:
-- FROM employees e LEFT JOIN departments d ON ...

-- In practice, most people just use LEFT JOIN and swap the table order.
-- RIGHT JOIN is rare in real code because LEFT JOIN is easier to read (left-to-right).

-- 🔹 FULL OUTER JOIN = show EVERYTHING from both sides
-- "Show me all departments AND all employees, matched where possible"
SELECT e.name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
-- If an employee has no department → dept_name is NULL
-- If a department has no employees → name is NULL
-- Useful for data quality checks!
```

> 🧠 **JOIN Cheat Sheet:**
> ```
> INNER JOIN:  Only rows that match in BOTH tables
> LEFT JOIN:   ALL rows from left + matches from right (NULL if no match)
> RIGHT JOIN:  ALL rows from right + matches from left (same as LEFT, just flipped)
> FULL OUTER:  ALL rows from BOTH tables (NULLs where no match)
> ```

---

### Task 8: Joining 3+ Tables (25 min)

This is where real data engineering lives. You'll often need to join 4-5 tables in one query.

```sql
-- 🔹 3-table join: Employees → Departments → Projects
-- "Show each employee, their department, and their project"
SELECT e.name, d.dept_name, p.project_name, ep.role
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
INNER JOIN employee_project ep ON e.emp_id = ep.emp_id
INNER JOIN projects p ON ep.project_id = p.project_id;
-- This chains: employee → their department (via dept_id)
--              employee → their project assignments (via employee_project)
--              project assignments → project details (via project_id)

-- 🔹 With filters and sorting
-- "Show all Engineers working on Data Platform v2"
SELECT e.name, e.salary, ep.role, ep.hours_worked
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
INNER JOIN employee_project ep ON e.emp_id = ep.emp_id
INNER JOIN projects p ON ep.project_id = p.project_id
WHERE d.dept_name = 'Engineering' AND p.project_name = 'Data Platform v2'
ORDER BY ep.hours_worked DESC;

-- 🔹 Aggregation across 3 tables
-- "Total hours worked per department per project"
SELECT d.dept_name, p.project_name, SUM(ep.hours_worked) AS total_hours
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
INNER JOIN employee_project ep ON e.emp_id = ep.emp_id
INNER JOIN projects p ON ep.project_id = p.project_id
GROUP BY d.dept_name, p.project_name
ORDER BY d.dept_name, total_hours DESC;
```

> 🧠 **How to read multi-table JOINs:**
> Read from left to right. Start with the "main" table (usually the one you care most about), then JOIN outward. Each JOIN adds a new relationship.
> 
> Think of it like a chain: Employee → (works in) → Department → (has budget) → ...

**🔥 Practice exercises:**

8. Show each employee's name, department, project name, and role. Order by employee name.
9. Find the total hours worked on each project. Show project name and total hours.
10. Which department has the most total hours across all projects?
11. Show all employees in Singapore who are working on "Cloud Migration" — include their role and hours.
12. Find employees who work on multiple projects (count > 1). Show their name and project count.

<details>
<summary>📖 Answers</summary>

```sql
-- 8
SELECT e.name, d.dept_name, p.project_name, ep.role
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
INNER JOIN employee_project ep ON e.emp_id = ep.emp_id
INNER JOIN projects p ON ep.project_id = p.project_id
ORDER BY e.name;

-- 9
SELECT p.project_name, SUM(ep.hours_worked) AS total_hours
FROM projects p
INNER JOIN employee_project ep ON p.project_id = ep.project_id
GROUP BY p.project_name
ORDER BY total_hours DESC;

-- 10
SELECT d.dept_name, SUM(ep.hours_worked) AS total_hours
FROM departments d
INNER JOIN employees e ON d.dept_id = e.dept_id
INNER JOIN employee_project ep ON e.emp_id = ep.emp_id
GROUP BY d.dept_name
ORDER BY total_hours DESC
LIMIT 1;

-- 11
SELECT e.name, ep.role, ep.hours_worked
FROM employees e
INNER JOIN employee_project ep ON e.emp_id = ep.emp_id
INNER JOIN projects p ON ep.project_id = p.project_id
WHERE e.city = 'Singapore' AND p.project_name = 'Cloud Migration';

-- 12
SELECT e.name, COUNT(ep.project_id) AS project_count
FROM employees e
INNER JOIN employee_project ep ON e.emp_id = ep.emp_id
GROUP BY e.emp_id, e.name
HAVING COUNT(ep.project_id) > 1;
-- ⚠️ HAVING is like WHERE but for groups! You'll learn more about this.
```
</details>

---

## 🌙 BLOCK 3: Practice + Consolidate (Evening, ~2 hours)

---

### Task 9: HackerRank JOIN Problems (45 min)

Go to https://www.hackerrank.com/domains/sql

Under **"Basic Join"**, complete these:

1. **Asian Population** — join City + Country, filter by continent
2. **African Cities** — join City + Country, filter
3. **Average Population of Each Continent** — join + group by + avg

These are harder than yesterday's. Take your time.

---

### Task 10: Real-World Scenario — E-commerce Database (45 min)

Let's create a second mini-database to practice. Run this:

```sql
-- E-commerce database
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    city VARCHAR(50),
    joined_date DATE
);

CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10,2)
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date DATE,
    status VARCHAR(20)  -- 'completed', 'shipped', 'cancelled', 'pending'
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER,
    unit_price DECIMAL(10,2)
);

-- Sample data
INSERT INTO customers (name, email, city, joined_date) VALUES
('Amy Wong', 'amy@email.com', 'Singapore', '2023-05-10'),
('Ben Tan', 'ben@email.com', 'Kuala Lumpur', '2023-08-22'),
('Cathy Lee', 'cathy@email.com', 'Singapore', '2024-01-15'),
('David Ng', 'david@email.com', 'Penang', '2023-11-03'),
('Eva Chang', 'eva@email.com', 'Kuala Lumpur', '2024-03-20'),
('Farid Ali', 'farid@email.com', 'Singapore', '2023-06-18');

INSERT INTO products (product_name, category, price) VALUES
('Wireless Mouse', 'Electronics', 45.00),
('USB-C Hub', 'Electronics', 89.00),
('Notebook A5', 'Stationery', 12.00),
('Standing Desk Mat', 'Office', 65.00),
('Mechanical Keyboard', 'Electronics', 150.00),
('Water Bottle', 'Lifestyle', 25.00),
('Monitor Stand', 'Office', 85.00);

INSERT INTO orders (customer_id, order_date, status) VALUES
(1, '2024-04-10', 'completed'),
(1, '2024-05-02', 'shipped'),
(2, '2024-04-15', 'completed'),
(2, '2024-04-20', 'cancelled'),
(3, '2024-05-01', 'completed'),
(4, '2024-04-25', 'pending'),
(5, '2024-05-05', 'completed'),
(6, '2024-04-18', 'shipped'),
(6, '2024-05-10', 'completed');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1, 1, 2, 45.00),
(1, 2, 1, 89.00),
(2, 5, 1, 150.00),
(3, 3, 5, 12.00),
(3, 6, 2, 25.00),
(4, 4, 1, 65.00),
(5, 2, 1, 89.00),
(5, 7, 1, 85.00),
(6, 1, 3, 45.00),
(7, 5, 1, 150.00),
(7, 3, 10, 12.00),
(8, 4, 2, 65.00),
(9, 6, 5, 25.00);
```

Now answer these business questions:

1. Show each customer's name and the total number of orders they've placed
2. Show each order with the customer name and total order value (quantity × unit_price, summed)
3. Which product category generates the most revenue?
4. Find customers who have never placed an order (use LEFT JOIN)
5. Show the top 3 best-selling products by total quantity sold
6. What is the average order value per customer?
7. Which city has the highest total revenue from completed orders?

<details>
<summary>📖 Answers</summary>

```sql
-- 1
SELECT c.name, COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;

-- 2
SELECT o.order_id, c.name, SUM(oi.quantity * oi.unit_price) AS total_value
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, c.name
ORDER BY total_value DESC;

-- 3
SELECT p.category, SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
INNER JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category
ORDER BY revenue DESC;

-- 4
SELECT c.name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;

-- 5
SELECT p.product_name, SUM(oi.quantity) AS total_qty
FROM products p
INNER JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_name
ORDER BY total_qty DESC
LIMIT 3;

-- 6
SELECT c.name, ROUND(AVG(order_total), 2) AS avg_order_value
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN (
    SELECT order_id, SUM(quantity * unit_price) AS order_total
    FROM order_items
    GROUP BY order_id
) oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, c.name;

-- 7
SELECT c.city, SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status = 'completed'
GROUP BY c.city
ORDER BY total_revenue DESC
LIMIT 1;
```
</details>

---

### Task 11: SQLZoo JOIN Sections (30 min)

Complete on https://sqlzoo.net:

- [ ] **Section 6: The JOIN operation** — all questions
- [ ] **Section 7: More JOIN operations** — all questions

---

### Task 12: Daily Reflection + GitHub Push (15 min)

Create `sql-practice/day02-notes.md`:

```markdown
# Day 2 — SQL JOINs
Date: 2026-05-16

## 3 Things I Learned Today
1. 
2. 
3. 

## JOIN Types I Can Explain
- INNER JOIN → 
- LEFT JOIN → 
- RIGHT JOIN → 
- FULL OUTER JOIN → 

## Foreign Key = 
## Junction Table (bridge table) = 

## The Tricky Part Today
- 

## Query I'm Most Proud Of
```sql
```

## Tomorrow I Want To Learn
- 
```

```bash
cd data-engineering-journey
git add .
git commit -m "Day 2: SQL JOINs (INNER, LEFT, RIGHT, FULL OUTER, multi-table)"
git push
```

---

## ✅ Day 2 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | 4 tables created (departments, employees, projects, employee_project) | ☐ |
| 2 | Can explain foreign key, one-to-many, many-to-many | ☐ |
| 3 | Wrote INNER JOIN queries (4 exercises) | ☐ |
| 4 | Wrote LEFT JOIN queries (3 exercises) | ☐ |
| 5 | Understand RIGHT JOIN and FULL OUTER JOIN | ☐ |
| 6 | Wrote 3+ table JOIN queries (5 exercises) | ☐ |
| 7 | Solved 3 HackerRank Join problems | ☐ |
| 8 | Built e-commerce database, solved 7 business questions | ☐ |
| 9 | Completed SQLZoo Sections 6 + 7 | ☐ |
| 10 | Day 2 notes written and pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card — JOINs

```
INNER JOIN  →  Only matching rows from BOTH tables
LEFT JOIN   →  ALL rows from left + matches from right (NULL if none)
RIGHT JOIN  →  ALL rows from right + matches from left
FULL OUTER  →  ALL rows from BOTH tables

Syntax:
SELECT columns
FROM table1 t1
[INNER|LEFT|RIGHT|FULL OUTER] JOIN table2 t2
  ON t1.key = t2.key
WHERE conditions
GROUP BY columns
ORDER BY columns;

Tips:
- Always use table aliases (t1, t2 or e, d, p)
- COUNT(column) skips NULLs, COUNT(*) doesn't
- LEFT JOIN + WHERE right.id IS NULL = "find orphans"
- Read multi-table JOINs left to right like a chain
```
