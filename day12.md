# 📅 Day 12 — Monday, 26 May 2026
# Pandas Basics: DataFrames, Reading Data, Filtering, Sorting

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Understand what Pandas is and why data engineers use it
- Create DataFrames from dicts, lists, and files
- Select columns and filter rows
- Sort data and handle missing values
- Use basic aggregation (groupby, value_counts)

**Why this matters:** Pandas is Python's spreadsheet on steroids. Every data engineer uses it to clean, transform, and analyze data before loading it into databases or data warehouses. It's mentioned in virtually every SG/MY data engineer job posting.

---

## ☀️ BLOCK 1: Pandas Setup + DataFrame Basics (Morning, ~1.5 hours)

---

### Task 1: Install Pandas (5 min)

```bash
pip install pandas
```

Verify:
```python
import pandas as pd
print(pd.__version__)  # should show 2.x.x
```

> 💡 **Convention:** Everyone imports pandas as `pd`. You'll see this in every tutorial, every job, every codebase. Don't import it any other way.

---

### Task 2: What is a DataFrame? (15 min)

A DataFrame is a **table** — rows and columns, just like SQL or Excel.

```
   name      department    salary      city
0  Alice     Engineering   8500.00     Singapore
1  Bob       Marketing     6200.00     Kuala Lumpur
2  Charlie   Engineering   9200.00     Singapore
3  Diana     HR            5800.00     Singapore
4  Eve       Marketing     7100.00     Kuala Lumpur
```

- Each **column** is a Series (like a list with an index)
- Each **row** has an index (0, 1, 2, 3... by default)
- Think of it as: **DataFrame = SQL table, Series = SQL column**

```python
import pandas as pd

# 🔹 Create DataFrame from a list of dicts (most common way)
employees = pd.DataFrame([
    {"name": "Alice Tan", "department": "Engineering", "salary": 8500, "city": "Singapore"},
    {"name": "Bob Lim", "department": "Marketing", "salary": 6200, "city": "Kuala Lumpur"},
    {"name": "Charlie Wong", "department": "Engineering", "salary": 9200, "city": "Singapore"},
    {"name": "Diana Chen", "department": "HR", "salary": 5800, "city": "Singapore"},
    {"name": "Eve Rahman", "department": "Marketing", "salary": 7100, "city": "Kuala Lumpur"},
    {"name": "Frank Kumar", "department": "Engineering", "salary": 10500, "city": "Singapore"},
    {"name": "Grace Lee", "department": "HR", "salary": 6400, "city": "Penang"},
    {"name": "Hank Tan", "department": "Engineering", "salary": 7800, "city": "Singapore"},
])

print(employees)
```

```python
# 🔹 Basic inspection — ALWAYS do these first when you get new data
print(employees.shape)         # (8, 4) → 8 rows, 4 columns
print(employees.columns)       # Index(['name', 'department', 'salary', 'city'], dtype='object')
print(employees.dtypes)        # data type of each column
print(employees.head(3))       # first 3 rows
print(employees.tail(2))       # last 2 rows
print(employees.info())        # summary: columns, types, non-null counts
print(employees.describe())    # statistics for numeric columns

# 🔹 Quick stats
print(f"Total employees: {len(employees)}")
print(f"Average salary: ${employees['salary'].mean():,.0f}")
print(f"Salary range: ${employees['salary'].min():,.0f} - ${employees['salary'].max():,.0f}")
```

---

### Task 3: Selecting Columns and Rows (30 min)

```python
# 🔹 Select one column → returns a Series
names = employees["name"]
print(type(names))    # <class 'pandas.core.series.Series'>
print(names)

# 🔹 Select multiple columns → returns a DataFrame
subset = employees[["name", "salary"]]
print(subset)
# Note the DOUBLE brackets: [[ ]] for multiple columns

# 🔹 .loc — select by LABEL (row index + column name)
print(employees.loc[0])                    # first row (all columns)
print(employees.loc[0, "name"])            # first row, name column
print(employees.loc[0:2, ["name", "salary"]])  # rows 0-2, specific columns
# loc is INCLUSIVE on both ends (0:2 includes 0, 1, 2)

# 🔹 .iloc — select by POSITION (integer index)
print(employees.iloc[0])          # first row
print(employees.iloc[0, 1])       # first row, second column (department)
print(employees.iloc[0:3])        # first 3 rows
# iloc is EXCLUSIVE on end (0:3 includes 0, 1, 2 — like Python lists)

# 🔹 Which to use?
# loc  → "get me rows by their index label, columns by name"
# iloc → "get me row number X, column number Y"
# In practice, you'll mostly use loc and filtering (next section)
```

---

### Task 4: Filtering Rows — The Most Important Skill (30 min)

```python
# 🔹 Single condition (like SQL WHERE)
engineers = employees[employees["department"] == "Engineering"]
print(engineers)

# 🔹 What's actually happening?
mask = employees["department"] == "Engineering"
print(mask)
# 0     True
# 1    False
# 2     True
# 3    False
# ...
# This is a boolean mask — True/False for each row
# employees[mask] keeps only True rows

# 🔹 Multiple conditions
# AND (like SQL: WHERE dept = 'Engineering' AND salary > 8000)
senior_engineers = employees[
    (employees["department"] == "Engineering") & 
    (employees["salary"] > 8000)
]
print(senior_engineers)

# OR (like SQL: WHERE city = 'Singapore' OR city = 'Kuala Lumpur')
sg_my = employees[
    (employees["city"] == "Singapore") | (employees["city"] == "Kuala Lumpur")
]

# ⚠️ ALWAYS wrap each condition in parentheses when using & or |
# ❌ employees["dept"] == "Eng" & employees["salary"] > 8000  → ERROR!
# ✅ (employees["dept"] == "Eng") & (employees["salary"] > 8000)

# 🔹 .isin() — like SQL IN
certain_cities = employees[employees["city"].isin(["Singapore", "Penang"])]

# 🔹 String contains — like SQL LIKE
tan_people = employees[employees["name"].str.contains("Tan")]

# 🔹 Negation — like SQL NOT
not_engineers = employees[employees["department"] != "Engineering"]

# 🔹 Combine everything
result = employees[
    (employees["department"].isin(["Engineering", "Marketing"])) &
    (employees["salary"] >= 7000) &
    (employees["city"] == "Singapore")
]
print(result)
```

**🔥 Practice exercises (create this DataFrame first):**

```python
import pandas as pd

sales = pd.DataFrame([
    {"date": "2024-01-15", "product": "Laptop", "category": "Electronics", "amount": 1499, "quantity": 1, "store": "Orchard"},
    {"date": "2024-01-16", "product": "T-Shirt", "category": "Clothing", "amount": 29.90, "quantity": 3, "store": "Jurong"},
    {"date": "2024-01-17", "product": "Phone", "category": "Electronics", "amount": 899, "quantity": 1, "store": "Pavilion"},
    {"date": "2024-01-18", "product": "Coffee Beans", "category": "Food", "amount": 45, "quantity": 2, "store": "Orchard"},
    {"date": "2024-01-19", "product": "Jeans", "category": "Clothing", "amount": 59.90, "quantity": 1, "store": "Mid Valley"},
    {"date": "2024-01-20", "product": "Tablet", "category": "Electronics", "amount": 599, "quantity": 1, "store": "Gurney"},
    {"date": "2024-01-21", "product": "Tea Set", "category": "Food", "amount": 68, "quantity": 1, "store": "Orchard"},
    {"date": "2024-01-22", "product": "Headphones", "category": "Electronics", "amount": 249, "quantity": 2, "store": "Pavilion"},
    {"date": "2024-01-23", "product": "Chocolate", "category": "Food", "amount": 35, "quantity": 5, "store": "Jurong"},
    {"date": "2024-01-24", "product": "Jacket", "category": "Clothing", "amount": 119, "quantity": 1, "store": "Mid Valley"},
])
```

1. Filter only Electronics sales
2. Filter sales where amount > 100
3. Filter sales at Orchard OR Pavilion stores
4. Filter Electronics sales with quantity >= 2
5. Filter sales NOT in Food category

<details>
<summary>📖 Answers</summary>

```python
# 1
electronics = sales[sales["category"] == "Electronics"]

# 2
expensive = sales[sales["amount"] > 100]

# 3
stores = sales[sales["store"].isin(["Orchard", "Pavilion"])]

# 4
bulk_electronics = sales[(sales["category"] == "Electronics") & (sales["quantity"] >= 2)]

# 5
not_food = sales[sales["category"] != "Food"]
```
</details>

---

## 🔥 BLOCK 2: Sorting, Adding Columns, Aggregation (Afternoon, ~2 hours)

---

### Task 5: Sorting and Adding Columns (30 min)

```python
# 🔹 Sort by one column
sorted_by_salary = employees.sort_values("salary")
print(sorted_by_salary)

# Sort descending
sorted_by_salary_desc = employees.sort_values("salary", ascending=False)
print(sorted_by_salary_desc)

# Sort by multiple columns
sorted_both = employees.sort_values(["department", "salary"], ascending=[True, False])
print(sorted_both)
# Sort by department A→Z, then within each department by salary high→low

# 🔹 Add a new column
employees["annual_salary"] = employees["salary"] * 12
employees["tax"] = employees["salary"] * 0.10
employees["net_salary"] = employees["salary"] - employees["tax"]
print(employees.head())

# 🔹 Add column with conditions (like SQL CASE WHEN)
import numpy as np

employees["tier"] = np.where(employees["salary"] >= 9000, "Senior",
                    np.where(employees["salary"] >= 7000, "Mid", "Junior"))
print(employees[["name", "salary", "tier"]])

# Alternative with pd.cut (for numeric bins):
employees["salary_band"] = pd.cut(employees["salary"], 
    bins=[0, 6000, 8000, 12000],
    labels=["Junior", "Mid", "Senior"])

# 🔹 Rename columns
employees_renamed = employees.rename(columns={
    "name": "employee_name",
    "salary": "monthly_salary"
})

# 🔹 Drop columns
employees_subset = employees.drop(columns=["tax", "net_salary"])

# 🔹 Add a calculated column using existing data
employees["name_length"] = employees["name"].str.len()
employees["first_name"] = employees["name"].str.split(" ").str[0]
```

---

### Task 6: Missing Values — The Reality of Data (20 min)

```python
# Real data always has missing values (NaN = Not a Number)
import numpy as np

messy_data = pd.DataFrame([
    {"name": "Alice", "salary": 8500, "dept": "Engineering"},
    {"name": "Bob", "salary": np.nan, "dept": "Marketing"},      # missing salary!
    {"name": None, "salary": 9200, "dept": "Engineering"},         # missing name!
    {"name": "Diana", "salary": 5800, "dept": None},               # missing dept!
    {"name": "Eve", "salary": 7100, "dept": "Marketing"},
])

print(messy_data)

# 🔹 Find missing values
print(messy_data.isnull())          # True where missing
print(messy_data.isnull().sum())    # count missing per column
# name      1
# salary    1
# dept      1

# 🔹 Handle missing values

# Option 1: Drop rows with ANY missing value
cleaned = messy_data.dropna()
print(f"Before: {len(messy_data)}, After dropna: {len(cleaned)}")

# Option 2: Drop rows where SPECIFIC column is missing
cleaned_salary = messy_data.dropna(subset=["salary"])

# Option 3: Fill missing values
messy_data["salary"] = messy_data["salary"].fillna(0)  # fill with 0
messy_data["name"] = messy_data["name"].fillna("Unknown")  # fill with text
messy_data["dept"] = messy_data["dept"].fillna("Unassigned")

# Option 4: Fill with mean/median (common for numeric)
mean_salary = messy_data["salary"].mean()
messy_data["salary"] = messy_data["salary"].fillna(mean_salary)

# 🔹 Replace specific values
messy_data["dept"] = messy_data["dept"].replace("Unassigned", "TBD")
```

> 🧠 **In data engineering, handling missing data is 50% of your job.** Real data is ALWAYS messy. You'll spend more time cleaning than analyzing.

---

### Task 7: Aggregation — groupby (40 min)

This is Pandas' equivalent of SQL's GROUP BY.

```python
# 🔹 Single aggregation
dept_counts = employees.groupby("department")["salary"].count()
print(dept_counts)
# department
# Engineering    4
# HR             2
# Marketing      2

# Average salary by department
dept_avg = employees.groupby("department")["salary"].mean()
print(dept_avg.round(2))

# 🔹 Multiple aggregations at once — .agg()
dept_summary = employees.groupby("department").agg(
    count=("salary", "count"),
    avg_salary=("salary", "mean"),
    min_salary=("salary", "min"),
    max_salary=("salary", "max"),
    total_salary=("salary", "sum"),
)
print(dept_summary.round(2))
# This is like SQL: 
# SELECT department, COUNT(*), AVG(salary), MIN(salary), MAX(salary), SUM(salary)
# FROM employees GROUP BY department

# 🔹 Group by multiple columns
dept_city = employees.groupby(["department", "city"]).agg(
    count=("salary", "count"),
    avg_salary=("salary", "mean"),
).reset_index()  # reset_index makes groupby columns into regular columns
print(dept_city)

# 🔹 value_counts — quick frequency count
print(employees["department"].value_counts())
print(employees["city"].value_counts())

# 🔹 With percentages
print(employees["department"].value_counts(normalize=True))
# Engineering: 0.500 (50%)
# HR:          0.250 (25%)
# Marketing:   0.250 (25%)
```

**🔥 Practice exercises (using the sales DataFrame from Task 4):**

6. Total revenue (sum of amount) by category
7. Number of sales and average amount by store
8. Top selling product by total quantity
9. Revenue by category, sorted highest first
10. For each store, show the most expensive product sold

<details>
<summary>📖 Answers</summary>

```python
# 6
rev_by_cat = sales.groupby("category")["amount"].sum()
print(rev_by_cat)

# 7
store_stats = sales.groupby("store").agg(
    num_sales=("amount", "count"),
    avg_amount=("amount", "mean"),
).round(2)
print(store_stats)

# 8
top_product = sales.groupby("product")["quantity"].sum().sort_values(ascending=False)
print(top_product.head(1))

# 9
rev_sorted = sales.groupby("category")["amount"].sum().sort_values(ascending=False)
print(rev_sorted)

# 10
idx = sales.groupby("store")["amount"].idxmax()
top_per_store = sales.loc[idx][["store", "product", "amount"]]
print(top_per_store)
```
</details>

---

### Task 8: Reading Data from Files (20 min)

```python
# 🔹 Read CSV (most common)
df = pd.read_csv("data/employees.csv")
print(df.head())
print(df.info())

# 🔹 Read CSV with options
df = pd.read_csv("data/employees.csv", 
    encoding="utf-8",       # file encoding
    parse_dates=["hire_date"],  # auto-convert to datetime
    na_values=["N/A", "null", ""],  # treat these as NaN
)

# 🔹 Read JSON
df_json = pd.read_json("data/weather.json")
print(df_json)

# 🔹 Read only specific columns (saves memory for big files)
df_subset = pd.read_csv("data/employees.csv", usecols=["name", "salary"])

# 🔹 Read first N rows (good for exploring huge files)
df_sample = pd.read_csv("data/employees.csv", nrows=100)

# 🔹 Write to CSV
employees.to_csv("data/output/pandas_employees.csv", index=False)
# index=False prevents writing the row index as a column

# 🔹 Write to JSON
employees.to_json("data/output/pandas_employees.json", orient="records", indent=2)
```

---

## 🌙 BLOCK 3: Practice + Mini-Project (Evening, ~1.5 hours)

---

### Task 9: Employee Analysis with Pandas (30 min)

Load the CSV from your Day 7 portfolio project (or use the employees data from today). Write a script `day12_pandas_basics.py` that:

```python
import pandas as pd

# 1. Load the employee data
# 2. Show basic info (shape, dtypes, head)
# 3. Filter: show only employees in Singapore earning > 7000
# 4. Add columns: annual_salary, tax (10%), net_salary, tier
# 5. Show average salary by department, sorted descending
# 6. Show employee count by city
# 7. Find the top 3 highest-paid employees
# 8. Save the enriched data to a new CSV

# Expected output: a printed summary report + enriched CSV file
```

<details>
<summary>📖 Answer</summary>

```python
import pandas as pd

# 1
employees = pd.DataFrame([
    {"name": "Alice Tan", "department": "Engineering", "salary": 8500, "city": "Singapore"},
    {"name": "Bob Lim", "department": "Marketing", "salary": 6200, "city": "Kuala Lumpur"},
    {"name": "Charlie Wong", "department": "Engineering", "salary": 9200, "city": "Singapore"},
    {"name": "Diana Chen", "department": "HR", "salary": 5800, "city": "Singapore"},
    {"name": "Eve Rahman", "department": "Marketing", "salary": 7100, "city": "Kuala Lumpur"},
    {"name": "Frank Kumar", "department": "Engineering", "salary": 10500, "city": "Singapore"},
    {"name": "Grace Lee", "department": "HR", "salary": 6400, "city": "Penang"},
    {"name": "Hank Tan", "department": "Engineering", "salary": 7800, "city": "Singapore"},
])

# 2
print(f"Shape: {employees.shape}")
print(f"Columns: {list(employees.columns)}")
print(employees.head())

# 3
high_earners_sg = employees[
    (employees["city"] == "Singapore") & (employees["salary"] > 7000)
]
print(f"\nHigh earners in SG: {len(high_earners_sg)}")
print(high_earners_sg[["name", "salary"]])

# 4
import numpy as np
employees["annual_salary"] = employees["salary"] * 12
employees["tax"] = employees["salary"] * 0.10
employees["net_salary"] = employees["salary"] - employees["tax"]
employees["tier"] = np.where(employees["salary"] >= 9000, "Senior",
                    np.where(employees["salary"] >= 7000, "Mid", "Junior"))

# 5
dept_avg = employees.groupby("department")["salary"].mean().round(2).sort_values(ascending=False)
print(f"\nAverage salary by department:")
for dept, avg in dept_avg.items():
    print(f"  {dept}: ${avg:,.2f}")

# 6
city_counts = employees["city"].value_counts()
print(f"\nEmployees by city:")
for city, count in city_counts.items():
    print(f"  {city}: {count}")

# 7
top3 = employees.nlargest(3, "salary")[["name", "salary", "department"]]
print(f"\nTop 3 highest paid:")
for _, row in top3.iterrows():
    print(f"  {row['name']} ({row['department']}): ${row['salary']:,}")

# 8
employees.to_csv("data/output/pandas_enriched.csv", index=False)
print("\nSaved to data/output/pandas_enriched.csv")
```
</details>

---

### Task 10: Pandas Practice on Kaggle Dataset (30 min)

Go to https://www.kaggle.com/datasets and find a small CSV dataset that interests you. Some good ones for beginners:

- **Titanic dataset** — classic beginner dataset
- **Supermarket Sales** — retail data
- **Netflix Shows** — entertainment data

Download a CSV, load it with Pandas, and explore:
- [ ] `df.head()`, `df.info()`, `df.describe()`
- [ ] Filter rows with a condition
- [ ] Group by a column and calculate aggregates
- [ ] Sort by a column
- [ ] Find missing values

---

### Task 11: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 12 — Pandas Basics
Date: 2026-05-26

## Pandas = SQL in Python
- DataFrame = Table
- Series = Column
- df[df["col"] == "x"] = WHERE
- df.groupby("col").agg(...) = GROUP BY
- df.sort_values("col") = ORDER BY
- df.head(5) = LIMIT 5
- df.isnull().sum() = check for NULLs

## Key Methods
- read_csv / to_csv
- head, info, describe, shape
- filtering with boolean masks
- groupby + agg
- sort_values
- value_counts
- fillna / dropna

## Tomorrow: Pandas Advanced (merge, pivot, apply, datetime)
```

```bash
git add .
git commit -m "Day 12: Pandas basics — DataFrames, filtering, sorting, groupby, aggregation"
git push
```

---

## ✅ Day 12 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Pandas installed and imported | ☐ |
| 2 | Created DataFrames from dicts and files | ☐ |
| 3 | Used head, info, describe, shape, dtypes | ☐ |
| 4 | Selected columns with [] and .loc | ☐ |
| 5 | Filtered rows with boolean masks (5 exercises) | ☐ |
| 6 | Sorted data with sort_values | ☐ |
| 7 | Added calculated columns | ☐ |
| 8 | Handled missing values (isnull, fillna, dropna) | ☐ |
| 9 | Used groupby and agg (5 exercises) | ☐ |
| 10 | Read and wrote CSV files with Pandas | ☐ |
| 11 | Built employee analysis mini-project | ☐ |
| 12 | Explored a Kaggle dataset | ☐ |
| 13 | Day 12 notes pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card

```python
import pandas as pd

# Create / Read
df = pd.DataFrame([{"col": "val"}])
df = pd.read_csv("file.csv")

# Inspect
df.shape, df.head(), df.info(), df.describe()

# Select
df["col"]              # one column (Series)
df[["col1", "col2"]]   # multiple columns (DataFrame)
df.loc[0, "col"]       # by label
df.iloc[0, 1]          # by position

# Filter (WHERE)
df[df["col"] > 100]
df[(df["a"] > 100) & (df["b"] == "x")]
df[df["col"].isin(["a", "b"])]
df[df["col"].str.contains("text")]

# Add column
df["new"] = df["old"] * 2
df["tier"] = np.where(df["sal"] >= 9000, "Senior", "Junior")

# Sort (ORDER BY)
df.sort_values("col", ascending=False)

# Aggregate (GROUP BY)
df.groupby("col")["val"].mean()
df.groupby("col").agg(count=("val", "count"), avg=("val", "mean"))

# Missing values
df.isnull().sum()
df.dropna()
df["col"].fillna(0)

# Read/Write
df.to_csv("out.csv", index=False)
pd.read_csv("in.csv", parse_dates=["date_col"])
```
