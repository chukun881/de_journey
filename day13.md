# 📅 Day 13 — Tuesday, 27 May 2026
# Pandas Advanced: Merge, Pivot, Apply, DateTime, Data Cleaning

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Merge DataFrames (like SQL JOINs)
- Pivot and reshape data
- Use .apply() for custom transformations
- Work with dates and times in Pandas
- Clean messy real-world data

**Why this matters:** Yesterday was "Pandas basics." Today is "Pandas for real work." These are the operations you'll use daily as a data engineer — joining datasets, reshaping data, handling dates, and cleaning messy data.

---

## ☀️ BLOCK 1: Merge — SQL JOINs in Pandas (Morning, ~1.5 hours)

---

### Task 1: Setup — Multiple Tables (10 min)

```python
import pandas as pd

# Just like SQL, we have separate related tables
employees = pd.DataFrame([
    {"emp_id": 1, "name": "Alice Tan", "dept_id": 1, "salary": 8500},
    {"emp_id": 2, "name": "Bob Lim", "dept_id": 2, "salary": 6200},
    {"emp_id": 3, "name": "Charlie Wong", "dept_id": 1, "salary": 9200},
    {"emp_id": 4, "name": "Diana Chen", "dept_id": 3, "salary": 5800},
    {"emp_id": 5, "name": "Eve Rahman", "dept_id": 2, "salary": 7100},
    {"emp_id": 6, "name": "Frank Kumar", "dept_id": 1, "salary": 10500},
    {"emp_id": 7, "name": "Grace Lee", "dept_id": 3, "salary": 6400},
])

departments = pd.DataFrame([
    {"dept_id": 1, "dept_name": "Engineering", "budget": 500000},
    {"dept_id": 2, "dept_name": "Marketing", "budget": 300000},
    {"dept_id": 3, "dept_name": "HR", "budget": 200000},
    {"dept_id": 4, "dept_name": "Finance", "budget": 250000},  # no employees yet!
])

projects = pd.DataFrame([
    {"project_id": "P1", "project_name": "Data Platform v2", "budget": 200000},
    {"project_id": "P2", "project_name": "Mobile App", "budget": 150000},
    {"project_id": "P3", "project_name": "Marketing Analytics", "budget": 80000},
])

assignments = pd.DataFrame([
    {"emp_id": 1, "project_id": "P1", "hours": 450},
    {"emp_id": 3, "project_id": "P1", "hours": 380},
    {"emp_id": 6, "project_id": "P1", "hours": 500},
    {"emp_id": 2, "project_id": "P2", "hours": 300},
    {"emp_id": 5, "project_id": "P2", "hours": 250},
    {"emp_id": 2, "project_id": "P3", "hours": 280},
    {"emp_id": 5, "project_id": "P3", "hours": 220},
])
```

---

### Task 2: pd.merge() — Inner, Left, Right, Outer (30 min)

```python
# 🔹 INNER JOIN — only matching rows
emp_dept = pd.merge(employees, departments, on="dept_id", how="inner")
print(emp_dept)
# Only employees whose dept_id exists in departments table

# 🔹 LEFT JOIN — all rows from left, matched from right
emp_dept_left = pd.merge(employees, departments, on="dept_id", how="left")
# Every employee appears, even if their department is missing

# 🔹 Find departments with no employees (like SQL LEFT JOIN + IS NULL)
dept_no_emp = pd.merge(departments, employees, on="dept_id", how="left", indicator=True)
print(dept_no_emp[dept_no_emp["_merge"] == "left_only"])
# Finance dept has no employees!
# _merge column tells you where each row came from

# 🔹 RIGHT JOIN — all rows from right
emp_dept_right = pd.merge(employees, departments, on="dept_id", how="right")

# 🔹 OUTER JOIN — everything from both sides
emp_dept_outer = pd.merge(employees, departments, on="dept_id", how="outer")

# 🔹 Different column names
# If left has "dept_id" and right has "id":
# pd.merge(left, right, left_on="dept_id", right_on="id")
```

> 🧠 **Pandas merge = SQL JOIN. Exact same concept.**
> - `how="inner"` → INNER JOIN
> - `how="left"` → LEFT JOIN
> - `how="right"` → RIGHT JOIN
> - `how="outer"` → FULL OUTER JOIN
> - `on="col"` → the JOIN condition

---

### Task 3: Multi-Table Joins (20 min)

```python
# 🔹 Chain joins: employees → departments → assignments → projects

# Step 1: employees + departments
emp_dept = pd.merge(employees, departments, on="dept_id", how="inner")

# Step 2: + assignments
emp_dept_assign = pd.merge(emp_dept, assignments, on="emp_id", how="left")

# Step 3: + projects
full = pd.merge(emp_dept_assign, projects, on="project_id", how="left")

print(full[["name", "dept_name", "project_name", "hours"]])

# 🔹 Aggregation after join
# Total hours by department
dept_hours = full.groupby("dept_name")["hours"].sum().reset_index()
print(dept_hours)

# Total hours by project
project_hours = full.groupby("project_name").agg(
    total_hours=("hours", "sum"),
    team_size=("emp_id", "nunique"),  # nunique = count distinct
).reset_index()
print(project_hours)
```

**🔥 Practice exercises:**

1. Show each employee's name, department name, and project name (3-table join)
2. Find employees who are NOT assigned to any project
3. Show each project's total hours and the department contributing the most hours

<details>
<summary>📖 Answers</summary>

```python
# 1
result = pd.merge(employees, departments, on="dept_id")
result = pd.merge(result, assignments, on="emp_id", how="left")
result = pd.merge(result, projects, on="project_id", how="left")
print(result[["name", "dept_name", "project_name"]])

# 2
emp_with_proj = pd.merge(employees, assignments, on="emp_id", how="left", indicator=True)
no_project = emp_with_proj[emp_with_proj["_merge"] == "left_only"]
print(no_project[["name"]])

# 3
full_data = pd.merge(pd.merge(pd.merge(employees, departments, on="dept_id"),
                               assignments, on="emp_id"),
                      projects, on="project_id")
proj_dept = full_data.groupby(["project_name", "dept_name"])["hours"].sum().reset_index()
idx = proj_dept.groupby("project_name")["hours"].idxmax()
print(proj_dept.loc[idx])
```
</details>

---

## 🔥 BLOCK 2: Reshape, Apply, DateTime (Afternoon, ~2 hours)

---

### Task 4: Pivot Tables and Melt (30 min)

```python
# 🔹 Pivot — reshape data (like Excel pivot table)
sales = pd.DataFrame([
    {"month": "Jan", "category": "Electronics", "revenue": 15000},
    {"month": "Jan", "category": "Clothing", "revenue": 8000},
    {"month": "Jan", "category": "Food", "revenue": 5000},
    {"month": "Feb", "category": "Electronics", "revenue": 18000},
    {"month": "Feb", "category": "Clothing", "revenue": 9000},
    {"month": "Feb", "category": "Food", "revenue": 6000},
    {"month": "Mar", "category": "Electronics", "revenue": 20000},
    {"month": "Mar", "category": "Clothing", "revenue": 7500},
    {"month": "Mar", "category": "Food", "revenue": 7000},
])

# Pivot: months as rows, categories as columns
pivot = sales.pivot(index="month", columns="category", values="revenue")
print(pivot)
# category  Clothing  Electronics  Food
# month
# Feb           9000        18000  6000
# Jan           8000        15000  5000
# Mar           7500        20000  7000

# Add total column
pivot["Total"] = pivot.sum(axis=1)
print(pivot)

# 🔹 pivot_table — more powerful, handles duplicates
# Same result but can aggregate if multiple values exist
pivot2 = sales.pivot_table(index="month", columns="category", 
                           values="revenue", aggfunc="sum")
print(pivot2)

# 🔹 Melt — unpivot (opposite of pivot)
# Turn wide format → long format
wide = pd.DataFrame({
    "name": ["Alice", "Bob", "Charlie"],
    "math": [85, 90, 78],
    "science": [92, 88, 95],
    "english": [88, 76, 82],
})

long = wide.melt(id_vars=["name"], var_name="subject", value_name="score")
print(long)
# name    subject   score
# Alice   math      85
# Alice   science   92
# Alice   english   88
# Bob     math      90
# ...
```

> 🧠 **Wide vs Long format:**
> - **Wide:** Each category is a column (good for humans reading)
> - **Long:** Each row is one observation (good for databases and analysis)
> - Data engineers convert between these constantly. SQL likes long, reports like wide.

---

### Task 5: .apply() — Custom Transformations (25 min)

```python
# 🔹 .apply() runs a function on every row or column
employees = pd.DataFrame([
    {"name": "alice tan", "salary": 8500, "dept": "engineering"},
    {"name": "bob lim", "salary": 6200, "dept": "marketing"},
    {"name": "CHARLIE WONG", "salary": 9200, "dept": "engineering"},
    {"name": "  diana chen  ", "salary": 5800, "dept": "hr"},
])

# Apply a function to a column
employees["name_clean"] = employees["name"].apply(lambda x: x.strip().title())
print(employees["name_clean"])

# Apply with a custom function
def categorize_salary(salary):
    if salary >= 9000:
        return "Senior"
    elif salary >= 7000:
        return "Mid"
    else:
        return "Junior"

employees["tier"] = employees["salary"].apply(categorize_salary)
print(employees[["name_clean", "salary", "tier"]])

# 🔹 Apply to multiple columns with axis=1
def compute_tax_row(row):
    tax = row["salary"] * 0.10
    net = row["salary"] - tax
    return pd.Series({"tax": tax, "net": net})

employees[["tax", "net"]] = employees.apply(compute_tax_row, axis=1)
print(employees)

# 🔹 .map() — for simple value replacement
employees["dept_clean"] = employees["dept"].map({
    "engineering": "Engineering",
    "marketing": "Marketing",
    "hr": "HR",
})
```

---

### Task 6: Working with Dates in Pandas (25 min)

```python
# 🔹 Create date data
orders = pd.DataFrame([
    {"order_id": 1, "date": "2024-01-15", "amount": 150},
    {"order_id": 2, "date": "2024-01-20", "amount": 85},
    {"order_id": 3, "date": "2024-02-03", "amount": 220},
    {"order_id": 4, "date": "2024-02-14", "amount": 175},
    {"order_id": 5, "date": "2024-03-01", "amount": 310},
    {"order_id": 6, "date": "2024-03-15", "amount": 95},
    {"order_id": 7, "date": "2024-03-28", "amount": 450},
    {"order_id": 8, "date": "2024-04-05", "amount": 180},
])

# Convert string to datetime
orders["date"] = pd.to_datetime(orders["date"])
print(orders.dtypes)  # date is now datetime64[ns]

# 🔹 Extract date parts (like SQL EXTRACT)
orders["year"] = orders["date"].dt.year
orders["month"] = orders["date"].dt.month
orders["day"] = orders["date"].dt.day
orders["day_of_week"] = orders["date"].dt.day_name()  # "Monday", "Tuesday"...
orders["quarter"] = orders["date"].dt.quarter
orders["week"] = orders["date"].dt.isocalendar().week.astype(int)

print(orders[["date", "month", "day_of_week", "quarter"]])

# 🔹 Date filtering
jan_orders = orders[orders["date"].dt.month == 1]
q1_orders = orders[orders["date"].dt.quarter == 1]
recent = orders[orders["date"] >= "2024-03-01"]

# 🔹 Date arithmetic
orders["date_plus_30"] = orders["date"] + pd.Timedelta(days=30)
orders["day_diff"] = (orders["date"] - orders["date"].min()).dt.days
print(orders[["date", "date_plus_30", "day_diff"]])

# 🔹 Resample — time-based grouping (VERY powerful!)
orders_indexed = orders.set_index("date")
monthly = orders_indexed.resample("M")["amount"].sum()
print(monthly)
# 2024-01-31    235
# 2024-02-29    395
# 2024-03-31    855
# 2024-04-30    180

# Weekly resample
weekly = orders_indexed.resample("W")["amount"].sum()
print(weekly)

# 🔹 Group by month
monthly_rev = orders.groupby(orders["date"].dt.to_period("M"))["amount"].sum()
print(monthly_rev)
# 2024-01    235
# 2024-02    395
# 2024-03    855
# 2024-04    180
```

---

### Task 7: Real-World Data Cleaning (30 min)

This simulates what you'll encounter in real ETL work.

```python
import pandas as pd
import numpy as np

# 🔹 Messy data (typical of what you'd get from a vendor API)
messy = pd.DataFrame([
    {"id": 1, "name": "  Alice Tan  ", "email": "ALICE@COMPANY.COM", "phone": "+65-1234-5678", "salary": "8,500.00", "dept": "Engineering", "join_date": "15-Jan-2023"},
    {"id": 2, "name": "bob lim", "email": "bob@email", "phone": "60-12-3456-7890", "salary": "6200", "dept": "marketing", "join_date": "2023/03/20"},
    {"id": 3, "name": "CHARLIE WONG", "email": "charlie@company.com", "phone": "+6598765432", "salary": "9,200.50", "dept": "Engineering ", "join_date": "01-11-2022"},
    {"id": 4, "name": "Diana Chen", "email": None, "phone": "N/A", "salary": "N/A", "dept": "HR", "join_date": "2024.02.10"},
    {"id": 5, "name": "Eve   Rahman", "email": "eve@company.com", "phone": "+60-12-3456789", "salary": "7,100", "dept": "Marketing", "join_date": "05 Jul 2023"},
    {"id": 6, "name": "Frank Kumar", "email": "frank@company.com", "phone": "+65-1111-2222", "salary": "10500.00", "dept": "engineering", "join_date": "12-Sep-2021"},
    {"id": "", "name": "", "email": "", "phone": "", "salary": "", "dept": "", "join_date": ""},  # empty row
])

print("=== BEFORE CLEANING ===")
print(messy)
print(f"\nMissing values:\n{messy.isnull().sum()}")

# 🔹 CLEANING PIPELINE
clean = messy.copy()

# Step 1: Drop completely empty rows
clean = clean.replace("", np.nan)  # treat empty strings as NaN
clean = clean.dropna(how="all")    # drop rows where ALL values are NaN

# Step 2: Fix data types
clean["id"] = clean["id"].astype(int)

# Step 3: Clean strings
clean["name"] = clean["name"].str.strip().str.title().str.replace(r"\s+", " ", regex=True)
clean["dept"] = clean["dept"].str.strip().str.title()

# Step 4: Clean email
clean["email"] = clean["email"].str.strip().str.lower()

# Step 5: Clean salary (remove commas, convert to float)
clean["salary"] = clean["salary"].replace("N/A", np.nan)
clean["salary"] = clean["salary"].str.replace(",", "").astype(float)

# Step 6: Clean phone (keep only digits)
clean["phone"] = clean["phone"].replace("N/A", np.nan)
clean["phone_digits"] = clean["phone"].str.replace(r"[^\d]", "", regex=True)

# Step 7: Parse dates (multiple formats!)
clean["join_date"] = pd.to_datetime(clean["join_date"], format="mixed", dayfirst=True)

# Step 8: Fill missing values
clean["email"] = clean["email"].fillna("unknown@company.com")
clean["salary"] = clean["salary"].fillna(clean["salary"].median())

print("\n=== AFTER CLEANING ===")
print(clean[["id", "name", "email", "salary", "dept", "join_date"]])
print(clean.dtypes)
print(f"\nMissing values:\n{clean.isnull().sum()}")
```

> 🧠 **This is real data engineering.** 80% of your time is cleaning data like this. Every source has different formats, missing values, inconsistencies. Your job is to make it uniform and usable.

---

## 🌙 BLOCK 3: Practice + Consolidation (Evening, ~1.5 hours)

---

### Task 8: Sales Analysis with Merge + Dates (30 min)

Create two CSVs and perform a complete analysis:

```python
import pandas as pd

# Create sample data
products = pd.DataFrame([
    {"product_id": "P001", "name": "Laptop", "category": "Electronics", "cost": 1000},
    {"product_id": "P002", "name": "Phone", "category": "Electronics", "cost": 600},
    {"product_id": "P003", "name": "T-Shirt", "category": "Clothing", "cost": 8},
    {"product_id": "P004", "name": "Coffee Beans", "category": "Food", "cost": 12},
    {"product_id": "P005", "name": "Headphones", "category": "Electronics", "cost": 80},
])

transactions = pd.DataFrame([
    {"txn_id": "T001", "product_id": "P001", "qty": 1, "price": 1499, "date": "2024-01-05", "store": "Orchard"},
    {"txn_id": "T002", "product_id": "P003", "qty": 5, "price": 29.90, "date": "2024-01-12", "store": "Jurong"},
    {"txn_id": "T003", "product_id": "P002", "qty": 2, "price": 899, "date": "2024-02-03", "store": "Pavilion"},
    {"txn_id": "T004", "product_id": "P004", "qty": 10, "price": 45, "date": "2024-02-14", "store": "Orchard"},
    {"txn_id": "T005", "product_id": "P001", "qty": 1, "price": 1450, "date": "2024-03-01", "store": "Mid Valley"},
    {"txn_id": "T006", "product_id": "P005", "qty": 3, "price": 249, "date": "2024-03-15", "store": "Gurney"},
    {"txn_id": "T007", "product_id": "P003", "qty": 8, "price": 32, "date": "2024-04-02", "store": "Pavilion"},
    {"txn_id": "T008", "product_id": "P002", "qty": 1, "price": 879, "date": "2024-04-20", "store": "Orchard"},
])

# Answer these using Pandas:
# 1. Merge transactions with products. Add revenue column (qty * price) and profit (revenue - qty * cost)
# 2. Total revenue and profit by category
# 3. Monthly revenue trend
# 4. Best performing store by profit
# 5. Profit margin by product
```

<details>
<summary>📖 Answers</summary>

```python
# 1
merged = pd.merge(transactions, products, on="product_id")
merged["revenue"] = merged["qty"] * merged["price"]
merged["profit"] = merged["revenue"] - (merged["qty"] * merged["cost"])
merged["date"] = pd.to_datetime(merged["date"])
print(merged[["name", "qty", "revenue", "profit"]])

# 2
cat_summary = merged.groupby("category").agg(
    total_revenue=("revenue", "sum"),
    total_profit=("profit", "sum"),
).round(2)
cat_summary["margin_pct"] = (cat_summary["total_profit"] / cat_summary["total_revenue"] * 100).round(1)
print(cat_summary)

# 3
merged["month"] = merged["date"].dt.to_period("M")
monthly = merged.groupby("month")["revenue"].sum()
print(monthly)

# 4
store_profit = merged.groupby("store")["profit"].sum().sort_values(ascending=False)
print(f"Best store: {store_profit.index[0]} (${store_profit.iloc[0]:,.2f} profit)")

# 5
product_margin = merged.groupby("name").agg(
    revenue=("revenue", "sum"),
    profit=("profit", "sum"),
)
product_margin["margin_pct"] = (product_margin["profit"] / product_margin["revenue"] * 100).round(1)
print(product_margin.sort_values("margin_pct", ascending=False))
```
</details>

---

### Task 9: HackerRank Python or Kaggle Exercise (30 min)

Option A: https://www.hackerrank.com/domains/python — do 3 problems from "Basic Data Types" or "Numpy"

Option B: Continue exploring the Kaggle dataset from Day 12 with your new skills:
- [ ] Merge two related tables
- [ ] Create a pivot table
- [ ] Use groupby with multiple aggregations
- [ ] Work with date columns

---

### Task 10: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 13 — Pandas Advanced
Date: 2026-05-27

## Key Skills
- pd.merge() = SQL JOIN (inner, left, right, outer)
- pivot_table = rows → columns (wide format)
- melt = columns → rows (long format)
- apply() = custom function per row/column
- pd.to_datetime() + dt accessor = date operations
- Data cleaning pipeline: strip, title, replace, fillna, dropna, astype

## The Data Cleaning Pattern
1. Load raw data
2. Replace empty/invalid → NaN
3. Drop empty rows
4. Fix data types
5. Clean strings
6. Parse dates
7. Handle missing values
8. Validate

## Tomorrow: PORTFOLIO PROJECT 2! 🚀
```

```bash
git add .
git commit -m "Day 13: Pandas advanced — merge, pivot, apply, datetime, data cleaning"
git push
```

---

## ✅ Day 13 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Merged DataFrames with inner, left, right, outer | ☐ |
| 2 | Chained multi-table joins | ☐ |
| 3 | Created pivot tables | ☐ |
| 4 | Used melt to reshape data | ☐ |
| 5 | Used apply() for custom transformations | ☐ |
| 6 | Parsed dates with pd.to_datetime | ☐ |
| 7 | Extracted date parts (month, quarter, day_name) | ☐ |
| 8 | Used resample for time-based aggregation | ☐ |
| 9 | Built a complete data cleaning pipeline | ☐ |
| 10 | Completed sales analysis with merge + dates | ☐ |
| 11 | Completed HackerRank or Kaggle exercises | ☐ |
| 12 | Day 13 notes pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card

```python
# Merge (= SQL JOIN)
pd.merge(df1, df2, on="key", how="inner")   # inner/left/right/outer
pd.merge(df1, df2, left_on="a", right_on="b")  # different column names

# Pivot (wide format)
df.pivot(index="row_col", columns="col_col", values="val")
df.pivot_table(index="row", columns="col", values="val", aggfunc="sum")

# Melt (long format)
df.melt(id_vars=["keep"], var_name="name", value_name="val")

# Apply
df["new"] = df["col"].apply(lambda x: x.upper())
df["new"] = df["col"].apply(custom_function)
df[["a","b"]] = df.apply(func_returning_series, axis=1)

# DateTime
df["date"] = pd.to_datetime(df["date_str"])
df["month"] = df["date"].dt.month
df["day_name"] = df["date"].dt.day_name()
df.set_index("date").resample("M")["val"].sum()

# Cleaning
df["col"] = df["col"].str.strip().str.title()
df["col"] = df["col"].str.replace(",", "").astype(float)
df["col"] = df["col"].fillna(df["col"].median())
```
