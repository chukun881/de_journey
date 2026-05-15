# 📅 Day 88 — Sunday, 10 August 2026
# 🐍 Python Live Coding: ETL Problems (30 Problems)

---

## 🎯 Today's Goal

DE interviews often have a Python coding section: "Write a function that..." — usually ETL-related. Today you solve 30 Python problems focused on data cleaning, string manipulation, date handling, Pandas operations, and building simple ETL functions.

---

## ☀️ Morning Block (2 hours): Python Core (Q1-15)

### Problems 1-5: Data Cleaning

**Q1.** Write a function `clean_order_data(records)` that takes a list of dicts and:
- Removes records where `total_amount` is None or <= 0
- Fills missing `city` with 'Unknown'
- Strips whitespace from string fields

```python
records = [
    {"id": 1, "customer": "Alice  ", "city": "Singapore", "total_amount": 45.0},
    {"id": 2, "customer": "Bob", "city": None, "total_amount": -5},
    {"id": 3, "customer": " Charlie ", "city": "KL", "total_amount": None},
    {"id": 4, "customer": "Diana", "city": "Penang", "total_amount": 30.0},
]
```

**Q2.** Write a function `normalize_city(city)` that maps variations to standard names:
- "SG", "Singapore", "Sgp", "singapore" → "Singapore"
- "KL", "Kuala Lumpur", "kuala lumpur" → "Kuala Lumpur"
- "PG", "Penang", "penang" → "Penang"
- "JB", "Johor Bahru", "johor bahru" → "Johor Bahru"

**Q3.** Write a function `parse_order_date(date_str)` that handles multiple formats:
- "2026-08-08", "08/08/2026", "Aug 8, 2026", "20260808"
- Return a datetime object or None for invalid inputs

**Q4.** Write a function `deduplicate_orders(orders)` that removes duplicates based on (customer_id, restaurant_id, date), keeping the one with the highest total_amount.

**Q5.** Write a function `validate_email(email)` that returns True/False for valid email format.

<details>
<summary>🔑 Answers 1-5</summary>

```python
# A1
def clean_order_data(records):
    cleaned = []
    for r in records:
        if r.get('total_amount') is None or r.get('total_amount', 0) <= 0:
            continue
        cleaned_record = {}
        for key, value in r.items():
            if isinstance(value, str):
                cleaned_record[key] = value.strip()
            else:
                cleaned_record[key] = value
        cleaned_record['city'] = cleaned_record.get('city') or 'Unknown'
        cleaned.append(cleaned_record)
    return cleaned

# Test
result = clean_order_data(records)
# Returns: records with id 1 and 4 only, with cleaned names and filled city

# A2
def normalize_city(city):
    if not city:
        return 'Unknown'
    mapping = {
        'sg': 'Singapore', 'singapore': 'Singapore', 'sgp': 'Singapore',
        'kl': 'Kuala Lumpur', 'kuala lumpur': 'Kuala Lumpur',
        'pg': 'Penang', 'penang': 'Penang',
        'jb': 'Johor Bahru', 'johor bahru': 'Johor Bahru',
    }
    return mapping.get(city.strip().lower(), city.strip())

# A3
from datetime import datetime

def parse_order_date(date_str):
    if not date_str:
        return None
    formats = ['%Y-%m-%d', '%d/%m/%Y', '%b %d, %Y', '%Y%m%d']
    for fmt in formats:
        try:
            return datetime.strptime(date_str.strip(), fmt)
        except ValueError:
            continue
    return None

# A4
def deduplicate_orders(orders):
    best = {}
    for order in orders:
        key = (order['customer_id'], order['restaurant_id'], order['date'])
        if key not in best or order['total_amount'] > best[key]['total_amount']:
            best[key] = order
    return list(best.values())

# A5
import re
def validate_email(email):
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))
```
</details>

### Problems 6-10: List/Dict Operations

**Q6.** Write a function `group_orders_by_city(orders)` that takes a list of order dicts and returns a dict mapping city → list of orders.

**Q7.** Write a function `top_n_restaurants(orders, n=5)` that returns the top N restaurants by total revenue using only dict operations (no Pandas).

**Q8.** Write a function `calculate_revenue_by_month(orders)` that returns {month_str: total_revenue}.

**Q9.** Write a function `flatten_nested_orders(data)` that flattens:
```python
data = [{"order_id": 1, "items": [{"name": "Nasi Lemak", "qty": 2}, {"name": "Teh Tarik", "qty": 1}]},
        {"order_id": 2, "items": [{"name": "Chicken Rice", "qty": 1}]}]
# → [{"order_id": 1, "item_name": "Nasi Lemak", "qty": 2}, ...]
```

**Q10.** Write a function `merge_customer_orders(customers, orders)` that joins two lists on customer_id, handling customers with no orders.

<details>
<summary>🔑 Answers 6-10</summary>

```python
# A6
from collections import defaultdict

def group_orders_by_city(orders):
    grouped = defaultdict(list)
    for order in orders:
        grouped[order['city']].append(order)
    return dict(grouped)

# A7
def top_n_restaurants(orders, n=5):
    revenue = defaultdict(float)
    for order in orders:
        revenue[order['restaurant_id']] += order['total_amount']
    sorted_rest = sorted(revenue.items(), key=lambda x: x[1], reverse=True)
    return sorted_rest[:n]

# A8
from datetime import datetime

def calculate_revenue_by_month(orders):
    monthly = defaultdict(float)
    for order in orders:
        month_key = datetime.strptime(order['order_date'], '%Y-%m-%d').strftime('%Y-%m')
        monthly[month_key] += order['total_amount']
    return dict(sorted(monthly.items()))

# A9
def flatten_nested_orders(data):
    flat = []
    for order in data:
        for item in order['items']:
            flat.append({
                'order_id': order['order_id'],
                'item_name': item['name'],
                'qty': item['qty']
            })
    return flat

# A10
def merge_customer_orders(customers, orders):
    orders_by_cust = defaultdict(list)
    for order in orders:
        orders_by_cust[order['customer_id']].append(order)
    
    result = []
    for cust in customers:
        cust_orders = orders_by_cust.get(cust['id'], [])
        result.append({
            **cust,
            'orders': cust_orders,
            'total_spent': sum(o['total_amount'] for o in cust_orders),
            'order_count': len(cust_orders)
        })
    return result
```
</details>

### Problems 11-15: String + Date + Comprehensions

**Q11.** Write a list comprehension that extracts all email domains from a list of customer dicts.

**Q12.** Write a function `format_currency(amount, currency='SGD')` that formats: `format_currency(1234.5)` → "SGD 1,234.50"

**Q13.** Write a function `get_date_range(start, end)` that returns a list of all dates between start and end (inclusive), formatted as "YYYY-MM-DD" strings.

**Q14.** Write a function `parse_csv_line(line)` that handles quoted fields with commas inside: `parse_csv_line('1,"Nasi Lemak, Ayam Goreng",12.50')` → `['1', 'Nasi Lemak, Ayam Goreng', '12.50']`

**Q15.** Write a function `mask_sensitive_data(record)` that masks email and phone: `"alice@gmail.com"` → `"a***e@gmail.com"`, `"0123456789"` → `"012***789"`

<details>
<summary>🔑 Answers 11-15</summary>

```python
# A11
domains = [c['email'].split('@')[1] for c in customers if '@' in c.get('email', '')]

# A12
def format_currency(amount, currency='SGD'):
    return f"{currency} {amount:,.2f}"

# A13
from datetime import datetime, timedelta

def get_date_range(start_str, end_str):
    start = datetime.strptime(start_str, '%Y-%m-%d')
    end = datetime.strptime(end_str, '%Y-%m-%d')
    dates = []
    current = start
    while current <= end:
        dates.append(current.strftime('%Y-%m-%d'))
        current += timedelta(days=1)
    return dates

# A14
import csv
from io import StringIO

def parse_csv_line(line):
    reader = csv.reader(StringIO(line))
    return next(reader)

# A15
def mask_sensitive_data(record):
    masked = record.copy()
    if 'email' in masked and '@' in masked['email']:
        local, domain = masked['email'].split('@')
        if len(local) > 2:
            masked['email'] = f"{local[0]}{'*' * (len(local)-2)}{local[-1]}@{domain}"
    if 'phone' in masked and masked['phone']:
        phone = masked['phone']
        masked['phone'] = phone[:3] + '***' + phone[-3:]
    return masked
```
</details>

---

## 🌤️ Afternoon Block (2 hours): Pandas + ETL Functions (Q16-30)

### Problems 16-22: Pandas Operations

**Q16.** Load a CSV of orders into a DataFrame. Show: total rows, null counts per column, data types.

**Q17.** Group by city and show: order count, total revenue, avg order value, median delivery time.

**Q18.** Merge orders with customers (LEFT JOIN). Find customers with no orders.

**Q19.** Pivot: create a table where rows = months, columns = cities, values = total revenue.

**Q20.** Find outliers: orders where total_amount is >3 std devs from the mean. Remove them.

**Q21.** Create a new column `order_hour` from order_date. Show order count by hour.

**Q22.** Calculate month-over-month revenue growth rate per city.

<details>
<summary>🔑 Answers 16-22</summary>

```python
import pandas as pd
import numpy as np

# A16
df = pd.read_csv('orders.csv')
print(f"Total rows: {len(df)}")
print(f"\nNull counts:\n{df.isnull().sum()}")
print(f"\nData types:\n{df.dtypes}")

# A17
summary = df.groupby('city').agg(
    order_count=('id', 'count'),
    total_revenue=('total_amount', 'sum'),
    avg_order_value=('total_amount', 'mean'),
    median_delivery=('delivery_time_min', 'median')
).round(2)
print(summary)

# A18
merged = pd.merge(orders_df, customers_df, left_on='customer_id', right_on='id', how='left')
no_orders = customers_df[~customers_df['id'].isin(orders_df['customer_id'])]
# OR:
merged_left = pd.merge(customers_df, orders_df, left_on='id', right_on='customer_id', how='left')
no_orders = merged_left[merged_left['order_id'].isna()]

# A19
df['month'] = pd.to_datetime(df['order_date']).dt.to_period('M')
pivot = df.pivot_table(index='month', columns='city', values='total_amount', aggfunc='sum', fill_value=0)
print(pivot)

# A20
mean = df['total_amount'].mean()
std = df['total_amount'].std()
threshold = mean + 3 * std
outliers = df[df['total_amount'] > threshold]
print(f"Found {len(outliers)} outliers (>{threshold:.2f})")
clean = df[df['total_amount'] <= threshold]

# A21
df['order_date'] = pd.to_datetime(df['order_date'])
df['order_hour'] = df['order_date'].dt.hour
hourly = df.groupby('order_hour').size()
print(hourly)

# A22
df['month'] = pd.to_datetime(df['order_date']).dt.to_period('M')
monthly = df.groupby(['city', 'month'])['total_amount'].sum().reset_index()
monthly['prev_month'] = monthly.groupby('city')['total_amount'].shift(1)
monthly['mom_growth'] = ((monthly['total_amount'] - monthly['prev_month']) / monthly['prev_month'] * 100).round(2)
print(monthly)
```
</details>

### Problems 23-30: ETL Functions

**Q23.** Write a complete ETL function:
```python
def etl_pipeline(source_path, output_path):
    """Extract CSV, transform (clean, validate), load to new CSV."""
    pass
```

**Q24.** Write a function `incremental_load(source_df, target_df, key='id')` that:
- Finds new rows (in source but not in target)
- Finds updated rows (same key, different values)
- Returns (new_rows, updated_rows)

**Q25.** Write a function `validate_data(df, rules)` that checks:
```python
rules = {
    'total_amount': {'min': 0, 'max': 10000, 'not_null': True},
    'city': {'allowed': ['Singapore', 'Kuala Lumpur', 'Penang']},
    'customer_id': {'not_null': True}
}
```

**Q26.** Write a function `safe_api_call(url, max_retries=3, backoff=2)` with exponential backoff.

**Q27.** Write a function `batch_insert(connection, table, records, batch_size=1000)`.

**Q28.** Write a function `detect_schema_drift(expected_schema, actual_df)` that compares columns and types.

**Q29.** Write a function `build_data_quality_report(df)` that returns: row count, null %, unique counts, min/max for numeric, top values for categorical.

**Q30.** Write a function `merge_multiple_sources(sources)` that:
- Reads multiple CSVs
- Standardizes column names (lowercase, strip, replace spaces with _)
- Unions them into one DataFrame
- Deduplicates

<details>
<summary>🔑 Answers 23-30</summary>

```python
# A23
import pandas as pd
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def etl_pipeline(source_path, output_path):
    """Extract CSV, transform (clean, validate), load to new CSV."""
    # Extract
    logger.info(f"Reading {source_path}")
    df = pd.read_csv(source_path)
    logger.info(f"Extracted {len(df)} rows")
    
    # Transform
    # Drop rows with null IDs
    df = df.dropna(subset=['id'])
    
    # Clean strings
    for col in df.select_dtypes(include='object').columns:
        df[col] = df[col].str.strip()
    
    # Validate amounts
    if 'total_amount' in df.columns:
        df = df[df['total_amount'] > 0]
    
    # Parse dates
    if 'order_date' in df.columns:
        df['order_date'] = pd.to_datetime(df['order_date'], errors='coerce')
        df = df.dropna(subset=['order_date'])
    
    logger.info(f"After transform: {len(df)} rows")
    
    # Load
    df.to_csv(output_path, index=False)
    logger.info(f"Loaded to {output_path}")
    return len(df)

# A24
def incremental_load(source_df, target_df, key='id'):
    target_keys = set(target_df[key]) if len(target_df) > 0 else set()
    
    new_rows = source_df[~source_df[key].isin(target_keys)]
    
    common_keys = set(source_df[key]) & target_keys
    updated_rows = pd.DataFrame()
    for k in common_keys:
        src = source_df[source_df[key] == k].iloc[0]
        tgt = target_df[target_df[key] == k].iloc[0]
        if not src.equals(tgt):
            updated_rows = pd.concat([updated_rows, source_df[source_df[key] == k]])
    
    return new_rows, updated_rows

# A25
def validate_data(df, rules):
    issues = []
    for col, checks in rules.items():
        if col not in df.columns:
            issues.append(f"Missing column: {col}")
            continue
        if checks.get('not_null') and df[col].isnull().any():
            issues.append(f"{col}: {df[col].isnull().sum()} null values")
        if 'min' in checks and (df[col] < checks['min']).any():
            issues.append(f"{col}: values below minimum {checks['min']}")
        if 'max' in checks and (df[col] > checks['max']).any():
            issues.append(f"{col}: values above maximum {checks['max']}")
        if 'allowed' in checks:
            invalid = ~df[col].isin(checks['allowed'])
            if invalid.any():
                issues.append(f"{col}: {invalid.sum()} values not in allowed list")
    return issues

# A26
import requests
import time

def safe_api_call(url, max_retries=3, backoff=2):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.RequestException as e:
            wait = backoff ** attempt
            print(f"Attempt {attempt+1} failed: {e}. Retrying in {wait}s...")
            time.sleep(wait)
    raise Exception(f"Failed after {max_retries} retries")

# A27
def batch_insert(connection, table, records, batch_size=1000):
    cursor = connection.cursor()
    columns = records[0].keys()
    placeholders = ', '.join(['%s'] * len(columns))
    cols = ', '.join(columns)
    sql = f"INSERT INTO {table} ({cols}) VALUES ({placeholders})"
    
    for i in range(0, len(records), batch_size):
        batch = records[i:i+batch_size]
        values = [tuple(r[c] for c in columns) for r in batch]
        cursor.executemany(sql, values)
        connection.commit()
        print(f"Inserted batch {i//batch_size + 1}: {len(batch)} rows")
    cursor.close()

# A28
def detect_schema_drift(expected_schema, actual_df):
    drift = {'missing_columns': [], 'new_columns': [], 'type_changes': []}
    expected_cols = set(expected_schema.keys())
    actual_cols = set(actual_df.columns)
    
    drift['missing_columns'] = list(expected_cols - actual_cols)
    drift['new_columns'] = list(actual_cols - expected_cols)
    
    for col in expected_cols & actual_cols:
        expected_type = expected_schema[col]
        actual_type = str(actual_df[col].dtype)
        if expected_type not in actual_type:
            drift['type_changes'].append({col: {'expected': expected_type, 'actual': actual_type}})
    
    return drift

# A29
def build_data_quality_report(df):
    report = {}
    report['row_count'] = len(df)
    report['null_pct'] = (df.isnull().sum() / len(df) * 100).round(2).to_dict()
    
    for col in df.columns:
        col_report = {}
        if df[col].dtype in ['int64', 'float64']:
            col_report['min'] = df[col].min()
            col_report['max'] = df[col].max()
            col_report['mean'] = round(df[col].mean(), 2)
        else:
            col_report['unique_count'] = df[col].nunique()
            col_report['top_values'] = df[col].value_counts().head(5).to_dict()
        report[col] = col_report
    
    return report

# A30
def merge_multiple_sources(sources):
    frames = []
    for path in sources:
        df = pd.read_csv(path)
        df.columns = [c.strip().lower().replace(' ', '_') for c in df.columns]
        df['source_file'] = path
        frames.append(df)
    
    combined = pd.concat(frames, ignore_index=True)
    combined = combined.drop_duplicates()
    return combined
```
</details>

---

## 🌙 Evening (1 hour): Python Interview Cheat Sheet

### 🗺️ Python Cheat Sheet for DE Interviews

```
COMMON PATTERNS:

1. File I/O:
   df = pd.read_csv('file.csv')
   df.to_csv('out.csv', index=False)
   with open('file.json') as f: data = json.load(f)

2. String cleaning:
   s.strip().lower().replace(' ', '_')
   re.sub(r'[^a-zA-Z0-9]', '', s)  # remove special chars

3. Date handling:
   pd.to_datetime(col, errors='coerce')
   dt.strftime('%Y-%m-%d')
   from datetime import timedelta; dt + timedelta(days=7)

4. Pandas essentials:
   df.groupby('col').agg(sum_col=('col2','sum'), avg_col=('col3','mean'))
   pd.merge(left, right, on='key', how='left')
   df.pivot_table(index='row', columns='col', values='val', aggfunc='sum')

5. Error handling:
   try/except with specific exceptions
   logging instead of print
   retry with exponential backoff

6. Performance:
   Use vectorized operations (no iterrows)
   Batch inserts (1000 rows at a time)
   Use generators for large files

INTERVIEW TIPS:
  - Write functions, not scripts
  - Include docstrings
  - Handle edge cases (empty input, None, wrong type)
  - Use type hints: def process(data: list[dict]) -> pd.DataFrame:
```

### 📝 Today's Checklist

- [ ] Completed 30 Python problems
- [ ] Comfortable with data cleaning functions
- [ ] Comfortable with Pandas groupby/merge/pivot
- [ ] Can write ETL functions with error handling
- [ ] Saved cheat sheet

---

*Day 88 complete! Tomorrow: System design practice.* 🏗️
