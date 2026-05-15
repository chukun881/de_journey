# 📅 Day 11 — Sunday, 25 May 2026
# Python File I/O: Reading & Writing CSV, JSON, Text Files

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Read and write text files
- Read and write CSV files (with csv module)
- Read and write JSON files (with json module)
- Understand file paths and the os/pathlib modules
- Handle file errors gracefully
- Build a simple file-based ETL: read → transform → write

**Why this matters:** As a data engineer, your job is literally moving data between systems. Files are the most basic form of data exchange. You'll read CSVs from vendors, parse JSON from APIs, write processed data to new files. This is the foundation of every ETL pipeline.

---

## ☀️ BLOCK 1: Text Files (Morning, ~1 hour)

---

### Task 1: Reading Text Files (20 min)

First, let's create a sample file to work with. Create `data/sample.txt`:

```python
# Run this first to create the sample file
with open("data/sample.txt", "w") as f:
    f.write("""Employee Report
Generated: 2024-05-15
==================

Name: Alice Tan
Department: Engineering
Salary: $8,500

Name: Bob Lim
Department: Marketing
Salary: $6,200

Name: Charlie Wong
Department: Engineering
Salary: $9,200
""")
print("Sample file created.")
```

Now read it:

```python
# 🔹 Method 1: Read entire file at once
with open("data/sample.txt", "r") as f:
    content = f.read()
print(content)
# .read() returns the ENTIRE file as one string

# 🔹 Method 2: Read line by line
with open("data/sample.txt", "r") as f:
    for line in f:
        print(line.strip())  # .strip() removes newline character
# This is better for large files — doesn't load everything into memory

# 🔹 Method 3: Read all lines into a list
with open("data/sample.txt", "r") as f:
    lines = f.readlines()
print(f"Total lines: {len(lines)}")
print(f"First line: {lines[0].strip()}")

# 🔹 The `with` statement
# with open(...) as f:
# This automatically closes the file when done, even if an error occurs.
# ALWAYS use with. Never do f = open(...) without closing.
```

> 🧠 **File modes:**
> - `"r"` — read (default, file must exist)
> - `"w"` — write (creates file, OVERWRITES if exists)
> - `"a"` — append (creates file, adds to end if exists)
> - `"r+"` — read + write

---

### Task 2: Writing Text Files (15 min)

```python
# 🔹 Write a new file
with open("data/output.txt", "w") as f:
    f.write("Report Header\n")
    f.write("=" * 40 + "\n")
    f.write(f"Generated: 2024-05-15\n\n")
    
    employees = [
        ("Alice", 8500),
        ("Bob", 6200),
        ("Charlie", 9200),
    ]
    
    for name, salary in employees:
        f.write(f"{name}: ${salary:,}\n")

print("File written!")

# 🔹 Append to existing file
with open("data/output.txt", "a") as f:
    f.write("\n--- Appended later ---\n")
    f.write("Diana: $5,800\n")

# 🔹 Verify by reading it back
with open("data/output.txt", "r") as f:
    print(f.read())
```

---

### Task 3: File Paths and os/pathlib (20 min)

```python
import os
from pathlib import Path

# 🔹 Current working directory
print(os.getcwd())

# 🔹 Join paths (use os.path, NOT string concatenation)
# ❌ Wrong: "data" + "/" + "file.csv"  (breaks on Windows)
# ✅ Right:
filepath = os.path.join("data", "reports", "sales.csv")
print(filepath)  # data/reports/sales.csv (or data\reports\sales.csv on Windows)

# 🔹 pathlib (modern, cleaner way)
data_dir = Path("data")
filepath = data_dir / "reports" / "sales.csv"  # / operator joins paths!
print(filepath)

# 🔹 Check if file exists
print(Path("data/sample.txt").exists())  # True/False

# 🔹 Create directory if it doesn't exist
Path("data/reports").mkdir(parents=True, exist_ok=True)
# parents=True: create parent directories too
# exist_ok=True: don't error if already exists

# 🔹 List files in a directory
for f in Path("data").glob("*.txt"):
    print(f.name)   # filename only
    print(f.suffix)  # .txt
    print(f.stem)    # sample (filename without extension)

# 🔹 Get file size
size = Path("data/sample.txt").stat().st_size
print(f"File size: {size} bytes")
```

> 🧠 **Always use `os.path.join()` or `pathlib` for file paths.** Never hardcode `/` or `\` — your code might run on different operating systems.

---

## 🔥 BLOCK 2: CSV Files — The Data Engineer's Daily Bread (Morning/Afternoon, ~1.5 hours)

---

### Task 4: Reading CSV Files (30 min)

First create a sample CSV. Create `data/employees.csv`:

```python
# Create sample CSV
with open("data/employees.csv", "w") as f:
    f.write("""name,department,salary,city,hire_date
Alice Tan,Engineering,8500.00,Singapore,2023-01-15
Bob Lim,Marketing,6200.00,Kuala Lumpur,2023-03-20
Charlie Wong,Engineering,9200.00,Singapore,2022-11-01
Diana Chen,HR,5800.00,Singapore,2024-02-10
Eve Rahman,Marketing,7100.00,Kuala Lumpur,2023-07-05
Frank Kumar,Engineering,10500.00,Singapore,2021-09-12
Grace Lee,HR,6400.00,Penang,2023-05-18
""")
```

Now read it:

```python
import csv

# 🔹 Method 1: csv.reader (returns lists)
with open("data/employees.csv", "r") as f:
    reader = csv.reader(f)
    header = next(reader)  # read the first row (header)
    print(f"Columns: {header}")
    
    for row in reader:
        # Each row is a list: ['Alice Tan', 'Engineering', '8500.00', ...]
        print(f"Name: {row[0]}, Dept: {row[1]}, Salary: ${float(row[2]):,.0f}")

# 🔹 Method 2: csv.DictReader (returns dicts — MUCH better!)
with open("data/employees.csv", "r") as f:
    reader = csv.DictReader(f)  # uses first row as keys
    for row in reader:
        # Each row is a dict: {'name': 'Alice Tan', 'department': 'Engineering', ...}
        print(f"{row['name']} ({row['department']}): ${float(row['salary']):,.0f}")

# 🔹 Load entire CSV into a list of dicts
with open("data/employees.csv", "r") as f:
    employees = list(csv.DictReader(f))

print(f"Total employees: {len(employees)}")
print(employees[0])  # first row as dict

# 🔹 Now you can do all your data structure tricks!
# Filter
engineers = [e for e in employees if e["department"] == "Engineering"]
print(f"Engineers: {len(engineers)}")

# Average salary
avg_salary = sum(float(e["salary"]) for e in employees) / len(employees)
print(f"Average salary: ${avg_salary:,.0f}")
```

> 🧠 **Always use `DictReader` over `reader`.** Dicts with named keys (`row["name"]`) are infinitely more readable than index access (`row[0]`). When columns get reordered or new columns are added, DictReader still works — reader breaks.

---

### Task 5: Writing CSV Files (20 min)

```python
# 🔹 Write from a list of dicts
report = [
    {"name": "Alice Tan", "department": "Engineering", "salary": 8500, "tax": 850},
    {"name": "Bob Lim", "department": "Marketing", "salary": 6200, "tax": 620},
    {"name": "Charlie Wong", "department": "Engineering", "salary": 9200, "tax": 920},
]

with open("data/salary_report.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "department", "salary", "tax"])
    writer.writeheader()   # write column names
    writer.writerows(report)  # write all rows at once

print("CSV written!")

# 🔹 Append to existing CSV
with open("data/salary_report.csv", "a", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "department", "salary", "tax"])
    writer.writerow({"name": "Diana Chen", "department": "HR", "salary": 5800, "tax": 580})

# 🔹 Verify
with open("data/salary_report.csv", "r") as f:
    print(f.read())
```

---

### Task 6: Mini-ETL with CSV (20 min)

This is your first real ETL pattern! **Extract** from CSV → **Transform** data → **Load** to new CSV.

```python
import csv
from pathlib import Path

# === EXTRACT: Read raw data ===
with open("data/employees.csv", "r") as f:
    raw_employees = list(csv.DictReader(f))

print(f"Extracted {len(raw_employees)} records")

# === TRANSFORM: Clean and enrich data ===
transformed = []
for emp in raw_employees:
    # Clean salary to float
    salary = float(emp["salary"])
    
    # Calculate derived fields
    annual_salary = salary * 12
    tax = salary * 0.10
    net_monthly = salary - tax
    
    # Determine tier
    if salary >= 9000:
        tier = "Senior"
    elif salary >= 7000:
        tier = "Mid"
    else:
        tier = "Junior"
    
    transformed.append({
        "name": emp["name"].strip(),
        "department": emp["department"].strip(),
        "city": emp["city"].strip(),
        "monthly_gross": round(salary, 2),
        "annual_gross": round(annual_salary, 2),
        "monthly_tax": round(tax, 2),
        "monthly_net": round(net_monthly, 2),
        "tier": tier,
    })

# === LOAD: Write transformed data ===
Path("data/output").mkdir(exist_ok=True)

with open("data/output/employee_report.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=transformed[0].keys())
    writer.writeheader()
    writer.writerows(transformed)

print("ETL complete! Check data/output/employee_report.csv")

# Quick validation
print(f"\nTier summary:")
from collections import Counter
tiers = Counter(e["tier"] for e in transformed)
for tier, count in tiers.items():
    print(f"  {tier}: {count}")
```

---

## 🔥 BLOCK 3: JSON Files — API Data Format (Afternoon, ~1.5 hours)

---

### Task 7: JSON Basics (25 min)

JSON (JavaScript Object Notation) is THE format for APIs and config files. It looks almost exactly like Python dicts and lists.

```python
import json

# 🔹 JSON structure maps to Python types:
# JSON object  → Python dict
# JSON array   → Python list
# JSON string  → Python str
# JSON number  → Python int/float
# JSON true    → Python True
# JSON false   → Python False
# JSON null    → Python None

# 🔹 Python to JSON string (serialization / encoding)
employee = {
    "name": "Alice Tan",
    "age": 28,
    "department": "Engineering",
    "salary": 8500.00,
    "skills": ["Python", "SQL", "Airflow"],
    "active": True,
    "manager": None,
}

json_string = json.dumps(employee, indent=2)  # indent=2 makes it pretty
print(json_string)
print(type(json_string))  # <class 'str'>

# 🔹 JSON string to Python (deserialization / decoding)
parsed = json.loads(json_string)
print(parsed["name"])    # Alice Tan
print(parsed["skills"])  # ['Python', 'SQL', 'Airflow']

# 🔹 Write JSON to file
with open("data/employee.json", "w") as f:
    json.dump(employee, f, indent=2)
# Note: dump (to file) vs dumps (to string)

# 🔹 Read JSON from file
with open("data/employee.json", "r") as f:
    loaded = json.load(f)
# Note: load (from file) vs loads (from string)

print(loaded["department"])  # Engineering
```

> 🧠 **Remember:**
> - `json.dumps()` → Python → JSON string (s = string)
> - `json.loads()` → JSON string → Python (s = string)
> - `json.dump()` → Python → JSON file (no s = file)
> - `json.load()` → JSON file → Python (no s = file)

---

### Task 8: Working with Nested JSON (25 min)

Real API data is deeply nested. This is where most beginners get lost.

```python
# 🔹 Nested JSON — like real API responses
api_response = {
    "status": "success",
    "data": {
        "users": [
            {
                "id": 1,
                "name": "Alice Tan",
                "contact": {
                    "email": "alice@company.com",
                    "phone": "+65-1234-5678"
                },
                "orders": [
                    {"order_id": 101, "amount": 150.00, "status": "completed"},
                    {"order_id": 102, "amount": 85.50, "status": "pending"},
                ]
            },
            {
                "id": 2,
                "name": "Bob Lim",
                "contact": {
                    "email": "bob@company.com",
                    "phone": "+60-1234-5678"
                },
                "orders": [
                    {"order_id": 103, "amount": 220.00, "status": "completed"},
                ]
            }
        ],
        "pagination": {
            "page": 1,
            "total_pages": 5,
            "total_records": 48
        }
    }
}

# 🔹 Navigate nested JSON step by step:
# Get the list of users
users = api_response["data"]["users"]
print(f"Users: {len(users)}")

# Get Alice's email
alice_email = api_response["data"]["users"][0]["contact"]["email"]
print(f"Alice's email: {alice_email}")

# Get all completed orders across all users
completed_orders = []
for user in users:
    for order in user["orders"]:
        if order["status"] == "completed":
            completed_orders.append({
                "user": user["name"],
                "order_id": order["order_id"],
                "amount": order["amount"],
            })

print(f"Completed orders: {completed_orders}")

# Total revenue from all completed orders
total_revenue = sum(o["amount"] for o in completed_orders)
print(f"Total revenue: ${total_revenue:,.2f}")

# 🔹 Flatten nested JSON into rows (VERY common ETL task)
flat_rows = []
for user in users:
    for order in user["orders"]:
        flat_rows.append({
            "user_id": user["id"],
            "user_name": user["name"],
            "email": user["contact"]["email"],
            "order_id": order["order_id"],
            "order_amount": order["amount"],
            "order_status": order["status"],
        })

# Write flattened data to CSV
with open("data/output/flattened_orders.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=flat_rows[0].keys())
    writer.writeheader()
    writer.writerows(flat_rows)

print("Flattened JSON → CSV complete!")
```

---

### Task 9: Real-World JSON — API Response Handling (20 min)

Save this as `data/api_sample.json`:

```python
# Simulate a real API response (like a weather API or stock API)
weather_data = {
    "city": "Singapore",
    "timestamp": "2024-05-15T14:30:00+08:00",
    "current": {
        "temp_c": 31.5,
        "humidity": 82,
        "condition": "Partly Cloudy",
        "wind_kph": 15.3,
    },
    "forecast": [
        {"date": "2024-05-16", "high_c": 33, "low_c": 26, "condition": "Sunny", "rain_pct": 10},
        {"date": "2024-05-17", "high_c": 32, "low_c": 25, "condition": "Thunderstorm", "rain_pct": 80},
        {"date": "2024-05-18", "high_c": 30, "low_c": 25, "condition": "Rain", "rain_pct": 60},
        {"date": "2024-05-19", "high_c": 31, "low_c": 26, "condition": "Cloudy", "rain_pct": 30},
        {"date": "2024-05-20", "high_c": 33, "low_c": 26, "condition": "Sunny", "rain_pct": 5},
    ]
}

with open("data/weather.json", "w") as f:
    json.dump(weather_data, f, indent=2)
```

Now process it:

```python
# Read and analyze weather data
with open("data/weather.json", "r") as f:
    data = json.load(f)

print(f"Weather in {data['city']}:")
print(f"Current temp: {data['current']['temp_c']}°C")
print(f"Humidity: {data['current']['humidity']}%")

# Find rainy days
rainy_days = [d for d in data["forecast"] if d["rain_pct"] > 50]
print(f"\nRainy days coming up: {len(rainy_days)}")
for day in rainy_days:
    print(f"  {day['date']}: {day['condition']} ({day['rain_pct']}% rain)")

# Average high temperature
avg_high = sum(d["high_c"] for d in data["forecast"]) / len(data["forecast"])
print(f"\nAverage high: {avg_high:.1f}°C")
```

---

## 🌙 BLOCK 4: Practice + Consolidation (Evening, ~1.5 hours)

---

### Task 10: Build a Complete File ETL (45 min)

Write `day11_file_etl.py` — a complete ETL pipeline:

**Scenario:** You receive messy sales data in JSON. Clean it and output a CSV report.

```python
# Create the messy input file first:
import json

raw_sales = {
    "report_date": "2024-05-15",
    "store": "Orchard Electronics Hub",
    "transactions": [
        {"txn_id": "T001", "customer": "  alice tan  ", "items": [
            {"product": "Laptop", "price": "1499.00", "qty": "1"},
            {"product": "Mouse", "price": "45.00", "qty": "2"},
        ]},
        {"txn_id": "T002", "customer": "BOB LIM", "items": [
            {"product": "Phone", "price": "999.00", "qty": "1"},
        ]},
        {"txn_id": "T003", "customer": "charlie wong", "items": [
            {"product": "Tablet", "price": "599.00", "qty": "1"},
            {"product": "Case", "price": "29.90", "qty": "3"},
            {"product": "Charger", "price": "39.90", "qty": "1"},
        ]},
        {"txn_id": "T004", "customer": "  DIANA CHEN  ", "items": [
            {"product": "Laptop", "price": "1499.00", "qty": "1"},
        ]},
    ]
}

with open("data/raw_sales.json", "w") as f:
    json.dump(raw_sales, f, indent=2)
```

Now write the ETL:

```python
# EXTRACT: Read the JSON
# TRANSFORM: For each transaction, flatten all items into individual rows
#   - Clean customer name (strip, title case)
#   - Convert price to float, qty to int
#   - Calculate line_total = price * qty
#   - Add store name and report date to each row
# LOAD: Write to data/output/sales_report.csv

# Expected output columns: txn_id, customer, product, price, qty, line_total, store, report_date
# Expected output: 7 rows (2 + 1 + 3 + 1 items)
```

<details>
<summary>📖 Answer</summary>

```python
import json
import csv
from pathlib import Path

# EXTRACT
with open("data/raw_sales.json", "r") as f:
    raw = json.load(f)

# TRANSFORM
rows = []
store = raw["store"]
report_date = raw["report_date"]

for txn in raw["transactions"]:
    txn_id = txn["txn_id"]
    customer = txn["customer"].strip().title()
    
    for item in txn["items"]:
        price = float(item["price"])
        qty = int(item["qty"])
        line_total = round(price * qty, 2)
        
        rows.append({
            "txn_id": txn_id,
            "customer": customer,
            "product": item["product"],
            "unit_price": price,
            "qty": qty,
            "line_total": line_total,
            "store": store,
            "report_date": report_date,
        })

# LOAD
Path("data/output").mkdir(exist_ok=True)
with open("data/output/sales_report.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=rows[0].keys())
    writer.writeheader()
    writer.writerows(rows)

# Validation
print(f"ETL complete! {len(rows)} rows written.")
total_revenue = sum(r["line_total"] for r in rows)
print(f"Total revenue: ${total_revenue:,.2f}")

# Quick summary
from collections import Counter
products = Counter(r["product"] for r in rows)
print("\nProducts sold:")
for product, count in products.most_common():
    print(f"  {product}: {count}")
```
</details>

---

### Task 11: Online Practice (30 min)

On https://www.learnpython.org/ or https://www.freecodecamp.org/learn/:
- [ ] **File I/O** exercises
- [ ] **JSON** exercises

---

### Task 12: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 11 — File I/O
Date: 2026-05-25

## What I Can Now Do
- Read/write text files with open() and with
- Read/write CSV with csv.DictReader/DictWriter
- Read/write JSON with json.load/dump
- Navigate nested JSON structures
- Flatten nested JSON into flat CSV rows
- Build Extract → Transform → Load pipelines

## The ETL Pattern
1. EXTRACT: Read data from source (CSV, JSON, API)
2. TRANSFORM: Clean, enrich, calculate, filter
3. LOAD: Write processed data to destination

## Tomorrow: Pandas Basics
```

```bash
git add .
git commit -m "Day 11: File I/O — CSV, JSON, text, ETL pipeline"
git push
```

---

## ✅ Day 11 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Read text files with open() and with | ☐ |
| 2 | Write text files with write() and append mode | ☐ |
| 3 | Use os.path.join and pathlib for paths | ☐ |
| 4 | Read CSV with csv.DictReader | ☐ |
| 5 | Write CSV with csv.DictWriter | ☐ |
| 6 | Built a CSV-based ETL pipeline | ☐ |
| 7 | Read/write JSON with json module | ☐ |
| 8 | Navigated nested JSON structures | ☐ |
| 9 | Flattened nested JSON to CSV | ☐ |
| 10 | Built the complete sales ETL project | ☐ |
| 11 | Completed online exercises | ☐ |
| 12 | Day 11 notes pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card

```python
# Text files
with open("file.txt", "r") as f:
    content = f.read()        # entire file
    lines = f.readlines()     # list of lines
    for line in f: ...        # line by line (memory efficient)

with open("file.txt", "w") as f:
    f.write("text\n")

# CSV
import csv
with open("data.csv", "r") as f:
    reader = csv.DictReader(f)
    rows = list(reader)       # list of dicts

with open("out.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=[...])
    writer.writeheader()
    writer.writerows(data)

# JSON
import json
with open("data.json", "r") as f:
    data = json.load(f)       # file → Python
with open("out.json", "w") as f:
    json.dump(data, f, indent=2)  # Python → file

# String conversion
json_str = json.dumps(data)   # Python → JSON string
parsed = json.loads(json_str) # JSON string → Python

# Paths
from pathlib import Path
Path("data").mkdir(exist_ok=True)
filepath = Path("data") / "file.csv"
filepath.exists()
```
