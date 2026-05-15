# 📅 Day 50 — Thursday, 3 July 2026
# 🔄 Week 1-7 Review & Weak Spot Practice

---

## 🎯 Today's Goal

You've covered 7 weeks of material — SQL, Python, dbt, Airflow, AWS Core, Glue, Athena, and PySpark. Before moving into advanced territory (Docker, APIs, interview prep), you need to find and fix your weak spots.

**Philosophy:** Most people study what they're already good at because it feels comfortable. Real growth comes from attacking what you're bad at. Today is uncomfortable by design.

---

## ☀️ Morning Block (2 hours): Self-Assessment Checklist

### How This Works

For each week, answer the reflection questions honestly. Rate yourself:
- 🟢 **Strong** — Can do this without looking at notes
- 🟡 **Shaky** — Know the concept but need to reference docs
- 🔴 **Weak** — Can't do this, would fail in an interview

Write your ratings down (paper or a file). Be brutally honest.

---

### Week 1 Self-Assessment: SQL Foundation

1. Can you write a query with GROUP BY, HAVING, and ORDER BY from memory?
2. Can you explain the difference between LEFT JOIN and RIGHT JOIN with a MakanExpress example?
3. Can you write a CTE that finds the top 5 restaurants by order volume?
4. Can you explain what ROW_NUMBER() does vs RANK() vs DENSE_RANK()?
5. Can you write a subquery vs a CTE and explain when to use each?
6. Can you use CASE WHEN to categorize orders into price tiers?
7. Can you explain what an index does and when to add one?
8. Can you write a UNION query combining two result sets?
9. Can you use LAG/LEAD to compare this month's orders vs last month?
10. Can you explain NULL handling in SQL (IS NULL, COALESCE, NULLIF)?

<details>
<summary>🔑 Quick Self-Rating Guide for Week 1</summary>

If you rated 🔴 on 3+ questions: You need a full re-read of Day 1-5.
If you rated 🟡 on 4+ questions: Do the practice problems below, focus on those topics.
If you rated 🟢 on 8+ questions: You're solid. Just review the quick reference cards.

**Most common weak spots in Week 1:**
- Window functions (people forget the syntax)
- NULL handling (tripled in interviews)
- Difference between RANK vs DENSE_RANK

</details>

---

### Week 2 Self-Assessment: Python + Pandas + ETL

1. Can you write a function that reads a CSV, filters rows, and saves to a new CSV?
2. Can you explain the difference between a list and a tuple? When would you use each?
3. Can you use a dictionary to count occurrences (e.g., count orders per restaurant)?
4. Can you write a try/except block that handles file-not-found and API errors differently?
5. Can you use Pandas to group by a column and calculate mean, sum, count?
6. Can you merge two DataFrames on different column names?
7. Can you write a list comprehension that filters and transforms data?
8. Can you read JSON from an API using the `requests` library?
9. Can you explain what `if __name__ == "__main__":` does and why it matters?
10. Can you write a simple ETL pipeline: extract from CSV → transform with Pandas → load to PostgreSQL?

<details>
<summary>🔑 Quick Self-Rating Guide for Week 2</summary>

**Most common weak spots:**
- Dictionary operations (people forget `.get()`, `.items()`, default values)
- Pandas merge vs join vs concat (confusing!)
- Error handling patterns (people skip this in practice)

</details>

---

### Week 3 Self-Assessment: Data Modeling + dbt

1. Can you draw a star schema for MakanExpress from memory (fact + dimensions)?
2. Can you explain the difference between OLTP and OLAP with examples?
3. Can you explain SCD Type 1, 2, and 3 in plain English?
4. Can you write a dbt model that joins staging models into a mart?
5. Can you explain what `ref()` does in dbt and why you shouldn't hardcode table names?
6. Can you write a dbt test for uniqueness, not-null, and relationships?
7. Can you create a dbt macro that takes a column name as parameter?
8. Can you explain incremental models and when to use them?
9. Can you generate dbt docs and explain what they show?
10. Can you explain the dbt DAG (directed acyclic graph) and why order matters?

<details>
<summary>🔑 Quick Self-Rating Guide for Week 3</summary>

**Most common weak spots:**
- SCD Type 2 implementation (the SQL is tricky)
- Incremental model logic (the `is_incremental()` macro)
- Macros (people avoid writing custom ones)

</details>

---

### Week 4 Self-Assessment: Advanced SQL + Data Modeling

1. Can you write a self-join to find repeat customers?
2. Can you explain query execution order (FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY)?
3. Can you optimize a slow query using EXPLAIN?
4. Can you design a snowflake schema and explain when it's better than star schema?
5. Can you implement SCD Type 2 using SQL MERGE or INSERT + UPDATE?
6. Can you write a running total using window functions?
7. Can you use a window function to find the latest order per customer?
8. Can you explain normalization (1NF, 2NF, 3NF) with an example?
9. Can you write a query that pivots rows to columns?
10. Can you explain when to use a bridge table?

<details>
<summary>🔑 Quick Self-Rating Guide for Week 4</summary>

**Most common weak spots:**
- EXPLAIN output interpretation (people don't practice this enough)
- SCD Type 2 SQL implementation
- PIVOT / conditional aggregation

</details>

---

### Week 5 Self-Assessment: Airflow Orchestration

1. Can you write a basic DAG from scratch (imports to dependencies)?
2. Can you explain the difference between `logical_date` and actual execution time?
3. Can you set up a PostgreSQL connection in Airflow?
4. Can you use PostgresHook to run a query and get results?
5. Can you pass data between tasks using XCom? What are the limitations?
6. Can you write a Sensor that waits for a file to arrive?
7. Can you use BranchPythonOperator to conditionally run tasks?
8. Can you explain idempotency and write an idempotent SQL INSERT?
9. Can you configure retry logic with exponential backoff?
10. Can you write tests for an Airflow DAG?

<details>
<summary>🔑 Quick Self-Rating Guide for Week 5</summary>

**Most common weak spots:**
- XCom size limits and alternatives
- TaskFlow API vs traditional operators
- DAG testing patterns (people don't test enough)

</details>

---

### Week 6 Self-Assessment: AWS Core

1. Can you create an S3 bucket and upload/download files using CLI?
2. Can you explain the 3-zone data lake pattern (raw/processed/analytics)?
3. Can you write a boto3 script that reads a CSV from S3 into a DataFrame?
4. Can you explain IAM least privilege with a data engineering example?
5. Can you write an IAM policy JSON that allows read-only access to one S3 bucket?
6. Can you launch an RDS PostgreSQL instance from the CLI?
7. Can you migrate a local database to RDS using pg_dump/psql?
8. Can you explain VPC, subnets, and security groups in simple terms?
9. Can you connect to an RDS instance in a private subnet from your laptop?
10. Can you explain the difference between an IAM user and an IAM role?

<details>
<summary>🔑 Quick Self-Rating Guide for Week 6</summary>

**Most common weak spots:**
- IAM policy JSON syntax (the format is tricky)
- VPC networking (understandable — DEs don't do this daily)
- boto3 specific methods (people know the concept, forget the syntax)

</details>

---

### Week 7 Self-Assessment: Glue + Athena + PySpark

1. Can you create a PySpark DataFrame from a list of dictionaries?
2. Can you filter, select, and group a PySpark DataFrame?
3. Can you explain what AWS Glue does and why it's serverless?
4. Can you create a Glue Crawler and explain what it produces?
5. Can you write a simple Glue PySpark job?
6. Can you query S3 data using Athena? What formats work best?
7. Can you explain partitioning in S3 and why it saves Athena costs?
8. Can you build a complete serverless pipeline: S3 → Glue → Athena?
9. Can you estimate AWS costs for Glue and Athena?
10. Can you explain when to use Glue vs when to use a self-managed Spark cluster?

<details>
<summary>🔑 Quick Self-Rating Guide for Week 7</summary>

**Most common weak spots:**
- PySpark syntax (similar to Pandas but different enough to confuse)
- Glue job configuration (DPUs, worker types)
- Athena partitioning strategies

</details>

---

### 📊 Tally Your Results

Count your 🔴 ratings per week:

| Week | Topic | 🔴 Count | Priority |
|------|-------|----------|----------|
| 1 | SQL Foundation | ___ | |
| 2 | Python + Pandas | ___ | |
| 3 | Data Modeling + dbt | ___ | |
| 4 | Advanced SQL | ___ | |
| 5 | Airflow | ___ | |
| 6 | AWS Core | ___ | |
| 7 | Glue + Athena + PySpark | ___ | |

**Your top 2-3 weakest weeks are your afternoon focus.**

---

## 🌤️ Afternoon Block (2 hours): Targeted Practice Problems

Work through practice problems in your weakest areas. Do NOT look at answers until you've attempted each problem.

### Practice Set A: SQL (if Week 1 or 4 is weak)

**Problem 1:** Write a query that finds each restaurant's month-over-month revenue growth rate using the MakanExpress `orders` table.

```sql
-- orders table: order_id, restaurant_id, order_date, total_amount
```

<details>
<summary>🔑 Answer</summary>

```sql
WITH monthly_revenue AS (
    SELECT
        restaurant_id,
        DATE_TRUNC('month', order_date) AS month,
        SUM(total_amount) AS revenue
    FROM orders
    GROUP BY restaurant_id, DATE_TRUNC('month', order_date)
),
with_previous AS (
    SELECT
        restaurant_id,
        month,
        revenue,
        LAG(revenue) OVER (PARTITION BY restaurant_id ORDER BY month) AS prev_revenue
    FROM monthly_revenue
)
SELECT
    restaurant_id,
    month,
    revenue,
    prev_revenue,
    ROUND(
        (revenue - prev_revenue) / NULLIF(prev_revenue, 0) * 100,
        2
    ) AS growth_rate_pct
FROM with_previous
ORDER BY restaurant_id, month;
```

**Why this matters:** Month-over-month analysis is one of the most common interview questions for data roles in SG/MY. GrabFood, Foodpanda, Shopee — they all ask this.

</details>

---

**Problem 2:** Find customers who ordered in January but NOT in February using the MakanExpress schema.

<details>
<summary>🔑 Answer</summary>

```sql
-- Method 1: Using LEFT JOIN with NULL check
SELECT DISTINCT c.customer_id, c.customer_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE DATE_TRUNC('month', o.order_date) = '2026-01-01'::date
  AND c.customer_id NOT IN (
    SELECT DISTINCT customer_id
    FROM orders
    WHERE DATE_TRUNC('month', order_date) = '2026-02-01'::date
  );

-- Method 2: Using EXCEPT (more readable)
SELECT DISTINCT customer_id
FROM orders
WHERE DATE_TRUNC('month', order_date) = '2026-01-01'::date

EXCEPT

SELECT DISTINCT customer_id
FROM orders
WHERE DATE_TRUNC('month', order_date) = '2026-02-01'::date;
```

**Interview tip:** The EXCEPT method is cleaner. Interviewers like seeing you know set operations.

</details>

---

**Problem 3:** Rank restaurants by average delivery time, but only include restaurants with at least 50 orders. Handle ties using DENSE_RANK.

<details>
<summary>🔑 Answer</summary>

```sql
WITH restaurant_stats AS (
    SELECT
        r.restaurant_id,
        r.restaurant_name,
        AVG(delivery_time_minutes) AS avg_delivery_time,
        COUNT(*) AS order_count
    FROM restaurants r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    GROUP BY r.restaurant_id, r.restaurant_name
    HAVING COUNT(*) >= 50
)
SELECT
    restaurant_name,
    avg_delivery_time,
    order_count,
    DENSE_RANK() OVER (ORDER BY avg_delivery_time ASC) AS delivery_rank
FROM restaurant_stats
ORDER BY delivery_rank;
```

**Why DENSE_RANK not RANK?** DENSE_RANK gives consecutive numbers (1,2,2,3) while RANK skips (1,2,2,4). If two restaurants tie at #2, the next should be #3, not #4. Interviewers test this.

</details>

---

**Problem 4:** Write a query to find the running total of daily revenue for MakanExpress, reset at the start of each month.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        PARTITION BY DATE_TRUNC('month', order_date)
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM (
    SELECT
        order_date,
        SUM(total_amount) AS daily_revenue
    FROM orders
    GROUP BY order_date
) daily
ORDER BY order_date;
```

**Key insight:** The `PARTITION BY DATE_TRUNC('month', order_date)` resets the running total each month. This is a common pattern for monthly reporting dashboards.

</details>

---

**Problem 5:** You have a table of food delivery driver shifts. Find drivers who worked more than 12 hours in a single day (potential labor violation in Singapore).

```sql
-- shifts: shift_id, driver_id, shift_start, shift_end, shift_date
```

<details>
<summary>🔑 Answer</summary>

```sql
WITH daily_hours AS (
    SELECT
        driver_id,
        shift_date,
        SUM(EXTRACT(EPOCH FROM (shift_end - shift_start)) / 3600) AS total_hours
    FROM shifts
    GROUP BY driver_id, shift_date
)
SELECT
    driver_id,
    shift_date,
    total_hours,
    CASE
        WHEN total_hours > 12 THEN 'VIOLATION - Over 12 hours'
        WHEN total_hours > 10 THEN 'WARNING - Over 10 hours'
        ELSE 'OK'
    END AS status
FROM daily_hours
WHERE total_hours > 10
ORDER BY total_hours DESC;
```

**Singapore context:** Singapore's Employment Act limits working hours. Data engineers at companies like Grab or Deliveroo build these compliance checks into their pipelines. This is a realistic interview scenario.

</details>

---

### Practice Set B: Python (if Week 2 is weak)

**Problem 1:** Write a function `validate_order(order: dict) -> tuple[bool, str]` that validates a MakanExpress order dict. It must check:
- `order_id` exists and is positive integer
- `customer_id` exists
- `items` is a non-empty list
- `total_amount` is positive float
- `restaurant_id` exists

Return `(True, "Valid")` or `(False, "reason")`.

<details>
<summary>🔑 Answer</summary>

```python
from typing import Any

def validate_order(order: dict) -> tuple[bool, str]:
    """Validate a MakanExpress order dictionary.
    
    Returns:
        tuple of (is_valid: bool, message: str)
    """
    # Check order_id
    if 'order_id' not in order:
        return False, "Missing order_id"
    if not isinstance(order['order_id'], int) or order['order_id'] <= 0:
        return False, f"Invalid order_id: {order['order_id']}"
    
    # Check customer_id
    if 'customer_id' not in order or not order['customer_id']:
        return False, "Missing customer_id"
    
    # Check items
    if 'items' not in order:
        return False, "Missing items"
    if not isinstance(order['items'], list) or len(order['items']) == 0:
        return False, "Items must be a non-empty list"
    
    # Check total_amount
    if 'total_amount' not in order:
        return False, "Missing total_amount"
    try:
        amount = float(order['total_amount'])
        if amount <= 0:
            return False, f"Total amount must be positive: {amount}"
    except (ValueError, TypeError):
        return False, f"Invalid total_amount: {order['total_amount']}"
    
    # Check restaurant_id
    if 'restaurant_id' not in order or not order['restaurant_id']:
        return False, "Missing restaurant_id"
    
    return True, "Valid"


# Test it
test_orders = [
    {'order_id': 1, 'customer_id': 'C001', 'items': ['Nasi Lemak'], 'total_amount': 12.50, 'restaurant_id': 'R001'},
    {'order_id': -1, 'customer_id': 'C001', 'items': ['Chicken Rice'], 'total_amount': 8.00, 'restaurant_id': 'R002'},
    {'order_id': 2, 'customer_id': 'C001', 'items': [], 'total_amount': 0, 'restaurant_id': 'R001'},
    {'order_id': 3},  # Missing everything else
]

for order in test_orders:
    valid, msg = validate_order(order)
    print(f"Order {order.get('order_id', 'N/A')}: {valid} — {msg}")

# Output:
# Order 1: True — Valid
# Order -1: False — Invalid order_id: -1
# Order 2: False — Items must be a non-empty list
# Order 3: False — Missing customer_id
```

**Why this matters:** Data validation is 30% of a data engineer's job. Every pipeline needs input validation before processing. Singapore companies like Grab have strict data quality requirements.

</details>

---

**Problem 2:** Write an ETL function that:
1. Extracts: Reads a CSV of orders from a file path
2. Transforms: Filters to completed orders only, calculates order value per restaurant, adds a `order_month` column
3. Loads: Writes the result to a new CSV and prints summary stats

<details>
<summary>🔑 Answer</summary>

```python
import pandas as pd
from datetime import datetime
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def extract(file_path: str) -> pd.DataFrame:
    """Extract orders from CSV file."""
    logger.info(f"Extracting from {file_path}")
    df = pd.read_csv(file_path)
    logger.info(f"Extracted {len(df)} rows")
    return df


def transform(df: pd.DataFrame) -> pd.DataFrame:
    """Transform: filter completed, aggregate by restaurant + month."""
    logger.info(f"Transforming {len(df)} rows")
    
    # Filter to completed orders only
    if 'status' in df.columns:
        df = df[df['status'] == 'completed'].copy()
        logger.info(f"After filtering completed: {len(df)} rows")
    
    # Add order_month
    df['order_date'] = pd.to_datetime(df['order_date'])
    df['order_month'] = df['order_date'].dt.to_period('M')
    
    # Aggregate: total revenue per restaurant per month
    result = df.groupby(['restaurant_id', 'restaurant_name', 'order_month']).agg(
        total_revenue=('total_amount', 'sum'),
        order_count=('order_id', 'count'),
        avg_order_value=('total_amount', 'mean'),
        unique_customers=('customer_id', 'nunique')
    ).reset_index()
    
    # Round monetary values
    result['total_revenue'] = result['total_revenue'].round(2)
    result['avg_order_value'] = result['avg_order_value'].round(2)
    
    # Convert Period to string for CSV compatibility
    result['order_month'] = result['order_month'].astype(str)
    
    logger.info(f"Transformed to {len(result)} rows")
    return result


def load(df: pd.DataFrame, output_path: str) -> None:
    """Load transformed data to CSV."""
    logger.info(f"Loading to {output_path}")
    df.to_csv(output_path, index=False)
    logger.info(f"Loaded {len(df)} rows to {output_path}")


def print_summary(df: pd.DataFrame) -> None:
    """Print summary statistics."""
    print("\n" + "=" * 50)
    print("📊 TRANSFORMATION SUMMARY")
    print("=" * 50)
    print(f"Total restaurants: {df['restaurant_id'].nunique()}")
    print(f"Total revenue: ${df['total_revenue'].sum():,.2f}")
    print(f"Total orders: {df['order_count'].sum():,}")
    print(f"Average order value: ${df['avg_order_value'].mean():,.2f}")
    print(f"Months covered: {df['order_month'].nunique()}")
    print(f"Top restaurant: {df.loc[df['total_revenue'].idxmax(), 'restaurant_name']}")
    print("=" * 50)


def run_pipeline(input_path: str, output_path: str) -> pd.DataFrame:
    """Full ETL pipeline."""
    raw = extract(input_path)
    transformed = transform(raw)
    load(transformed, output_path)
    print_summary(transformed)
    return transformed


# Usage:
# result = run_pipeline('data/makanexpress_orders.csv', 'data/restaurant_monthly.csv')
```

**Key patterns demonstrated:**
- Separate functions for each ETL stage (testable, maintainable)
- Logging at each stage (production pattern)
- Type hints (shows professional Python)
- Summary stats for verification
- `.copy()` to avoid SettingWithCopyWarning

</details>

---

**Problem 3:** Write a function that reads a JSON API response containing MakanExpress restaurants and normalizes the nested data into a flat Pandas DataFrame.

```python
# Input format:
# {
#     "restaurants": [
#         {"id": 1, "name": "Ah Hock Nasi Lemak", "cuisine": ["Malay", "Local"],
#          "branches": [{"area": "Orchard", "rating": 4.5}, {"area": "Jurong", "rating": 4.2}]},
#         {"id": 2, "name": "Tian Tian Hainanese", "cuisine": ["Chinese", "Local"],
#          "branches": [{"area": "Maxwell", "rating": 4.8}]}
#     ]
# }
```

<details>
<summary>🔑 Answer</summary>

```python
import pandas as pd

def normalize_restaurants(data: dict) -> pd.DataFrame:
    """Normalize nested restaurant JSON into flat DataFrame."""
    rows = []
    
    for restaurant in data.get('restaurants', []):
        rest_id = restaurant['id']
        rest_name = restaurant['name']
        cuisines = ', '.join(restaurant.get('cuisine', []))
        
        for branch in restaurant.get('branches', []):
            rows.append({
                'restaurant_id': rest_id,
                'restaurant_name': rest_name,
                'cuisine_tags': cuisines,
                'branch_area': branch.get('area'),
                'branch_rating': branch.get('rating')
            })
    
    df = pd.DataFrame(rows)
    return df


# Alternative using json_normalize (Pandas built-in):
def normalize_restaurants_v2(data: dict) -> pd.DataFrame:
    """Using pd.json_normalize for the same task."""
    restaurants = data.get('restaurants', [])
    
    df = pd.json_normalize(
        restaurants,
        record_path='branches',
        meta=['id', 'name', ['cuisine']],
        record_prefix='branch_'
    )
    
    df.columns = df.columns.str.replace('branch_', 'branch_', regex=False)
    df.rename(columns={'id': 'restaurant_id', 'name': 'restaurant_name'}, inplace=True)
    
    return df


# Test
test_data = {
    "restaurants": [
        {"id": 1, "name": "Ah Hock Nasi Lemak", "cuisine": ["Malay", "Local"],
         "branches": [{"area": "Orchard", "rating": 4.5}, {"area": "Jurong", "rating": 4.2}]},
        {"id": 2, "name": "Tian Tian Hainanese", "cuisine": ["Chinese", "Local"],
         "branches": [{"area": "Maxwell", "rating": 4.8}]}
    ]
}

result = normalize_restaurants(test_data)
print(result)
#    restaurant_id        restaurant_name  cuisine_tags branch_area  branch_rating
# 0              1     Ah Hock Nasi Lemak  Malay, Local     Orchard            4.5
# 1              1     Ah Hock Nasi Lemak  Malay, Local      Jurong            4.2
# 2              2   Tian Tian Hainanese  Chinese, Local    Maxwell            4.8
```

**Why this matters:** API data is almost always nested. Flattening it is one of the most common data engineering tasks. At Shopee or Grab, you'd do this daily.

</details>

---

**Problem 4:** Write a retry decorator that retries a function up to 3 times with exponential backoff if it raises a `ConnectionError`.

<details>
<summary>🔑 Answer</summary>

```python
import time
import functools
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def retry(max_retries=3, base_delay=1, exceptions=(ConnectionError,)):
    """Retry decorator with exponential backoff.
    
    Args:
        max_retries: Maximum number of retry attempts
        base_delay: Initial delay in seconds (doubles each retry)
        exceptions: Tuple of exception types to catch
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    if attempt < max_retries:
                        delay = base_delay * (2 ** attempt)
                        logger.warning(
                            f"Attempt {attempt + 1}/{max_retries + 1} failed: {e}. "
                            f"Retrying in {delay}s..."
                        )
                        time.sleep(delay)
                    else:
                        logger.error(
                            f"All {max_retries + 1} attempts failed for {func.__name__}"
                        )
            raise last_exception
        return wrapper
    return decorator


# Usage example
import requests

@retry(max_retries=3, base_delay=1, exceptions=(ConnectionError, requests.Timeout))
def fetch_restaurant_data(api_url: str) -> dict:
    """Fetch restaurant data from MakanExpress API."""
    response = requests.get(api_url, timeout=10)
    response.raise_for_status()
    return response.json()


# Test with a mock function
@retry(max_retries=2, base_delay=0.5)
def unstable_function(should_fail: bool = True):
    """Simulates an unstable API call."""
    import random
    if should_fail and random.random() < 0.7:
        raise ConnectionError("API temporarily unavailable")
    return {"status": "success", "data": [1, 2, 3]}

# result = unstable_function()
```

**Interview gold:** A retry decorator shows you understand production reliability patterns. At scale (Grab processes millions of orders/day), every API call needs retry logic.

</details>

---

**Problem 5:** Write a Python script that uses boto3 to list all objects in an S3 bucket and print their sizes in a human-readable format (KB, MB, GB).

<details>
<summary>🔑 Answer</summary>

```python
import boto3

def human_size(bytes_size: int) -> str:
    """Convert bytes to human readable format."""
    for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
        if bytes_size < 1024:
            return f"{bytes_size:.1f} {unit}"
        bytes_size /= 1024
    return f"{bytes_size:.1f} PB"


def list_bucket_objects(bucket_name: str, prefix: str = '') -> None:
    """List all objects in an S3 bucket with sizes."""
    s3 = boto3.client('s3', region_name='ap-southeast-1')
    
    paginator = s3.get_paginator('list_objects_v2')
    pages = paginator.paginate(Bucket=bucket_name, Prefix=prefix)
    
    total_size = 0
    object_count = 0
    
    print(f"\n📦 Contents of s3://{bucket_name}/{prefix}")
    print("-" * 80)
    print(f"{'Key':<50} {'Size':>12} {'Last Modified':>20}")
    print("-" * 80)
    
    for page in pages:
        for obj in page.get('Contents', []):
            size = human_size(obj['Size'])
            modified = obj['LastModified'].strftime('%Y-%m-%d %H:%M')
            # Truncate long keys
            key = obj['Key']
            if len(key) > 48:
                key = '...' + key[-45:]
            print(f"{key:<50} {size:>12} {modified:>20}")
            total_size += obj['Size']
            object_count += 1
    
    print("-" * 80)
    print(f"Total: {object_count} objects, {human_size(total_size)}")
    
    return object_count, total_size


# Usage:
# list_bucket_objects('makanexpress-data-lake-dev', 'raw/')
```

**Note:** Using a paginator is important! S3 `list_objects_v2` returns max 1000 objects per call. The paginator handles this automatically. Interviewers love asking "what happens if there are more than 1000 objects?"

</details>

---

### Practice Set C: AWS + Architecture (if Week 6 or 7 is weak)

**Problem 1:** Design a complete AWS architecture for MakanExpress analytics. Draw it as ASCII art and explain each component's role.

<details>
<summary>🔑 Answer</summary>

```
┌─────────────────────────────────────────────────────────────────┐
│                    MakanExpress Analytics Platform               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  📱 Source Systems                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │ Order API │  │ Restaurant│  │  Payment  │                     │
│  │ (JSON)   │  │   DB      │  │  Gateway  │                     │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                      │
│       │              │              │                            │
│       ▼              ▼              ▼                            │
│  ┌──────────────────────────────────────┐                       │
│  │         S3: Raw Zone (CSV/JSON)      │                       │
│  │  s3://makanexpress-dl/raw/orders/    │                       │
│  │  s3://makanexpress-dl/raw/restaurants│                       │
│  │  s3://makanexpress-dl/raw/payments/  │                       │
│  └──────────────┬───────────────────────┘                       │
│                 │ Glue Crawler + Glue Job                        │
│                 ▼                                                 │
│  ┌──────────────────────────────────────┐                       │
│  │      S3: Processed Zone (Parquet)     │                       │
│  │  s3://makanexpress-dl/processed/      │                       │
│  │  Partitioned by: year/month/day       │                       │
│  └──────────────┬───────────────────────┘                       │
│                 │ Glue Crawler creates table                      │
│                 ▼                                                 │
│  ┌──────────────────────────────────────┐                       │
│  │          AWS Athena (SQL on S3)       │                       │
│  │    "SELECT * FROM processed_orders    │                       │
│  │     WHERE month = '2026-06'"          │                       │
│  └──────────────┬───────────────────────┘                       │
│                 │                                                 │
│                 ▼                                                 │
│  ┌──────────────────────────────────────┐                       │
│  │   S3: Analytics Zone (Aggregations)   │                       │
│  │   + Amazon QuickSight / Metabase      │                       │
│  └──────────────────────────────────────┘                       │
│                                                                  │
│  🔐 IAM Roles:                                                   │
│  - glue-etl-role → read raw, write processed                    │
│  - athena-query-role → read processed, write analytics          │
│  - data-engineer-group → full access (for you)                  │
│                                                                  │
│  📊 RDS PostgreSQL:                                              │
│  - Star schema warehouse (from Week 4-5)                        │
│  - Connected via Airflow for scheduled loads                    │
└─────────────────────────────────────────────────────────────────┘
```

**Component explanation:**
1. **Raw Zone:** Immutable copy of source data. Never modify. Like a safety deposit box.
2. **Glue Crawler:** Auto-discovers schema from raw files, creates Data Catalog tables.
3. **Glue Job:** PySpark ETL — cleans, validates, converts to Parquet.
4. **Processed Zone:** Clean, typed, partitioned data ready for queries.
5. **Athena:** Serverless SQL engine. Pay per query. No infrastructure to manage.
6. **Analytics Zone:** Pre-computed aggregations for dashboards.
7. **IAM Roles:** Each service has least-privilege access.

**Cost estimate for SG/MY startup scale:**
- S3 storage: ~$1-5/month
- Glue jobs: ~$0.44/hour per DPU (1-2 hours/day = ~$30/month)
- Athena: $5/TB scanned (partitioned = less scanned = cheaper)
- RDS: Free tier or ~$25/month for db.t3.micro

</details>

---

**Problem 2:** Write a CloudFormation-like resource list (just the resources section) for the MakanExpress data lake. Even if you don't know CloudFormation, write what AWS resources you'd create and their key configurations.

<details>
<summary>🔑 Answer</summary>

```yaml
Resources Needed:
  1. S3 Bucket: makanexpress-data-lake-dev
     - Versioning: Enabled
     - Encryption: AES-256
     - Lifecycle: raw/ → Glacier after 90 days
     - Folders: raw/, processed/, analytics/, scripts/, temp/

  2. S3 Bucket: makanexpress-glue-scripts
     - For storing Glue PySpark scripts
     - Versioning: Enabled

  3. Glue Database: makanexpress_catalog
     - For organizing Glue Data Catalog tables

  4. Glue Crawler: makanexpress_raw_crawler
     - Source: s3://makanexpress-data-lake-dev/raw/
     - Target database: makanexpress_catalog
     - Schedule: Daily at 2 AM SGT
     - IAM Role: glue-crawler-role

  5. Glue Crawler: makanexpress_processed_crawler
     - Source: s3://makanexpress-data-lake-dev/processed/
     - Target database: makanexpress_catalog
     - Schedule: After Glue job completes

  6. Glue Job: makanexpress_orders_etl
     - Script: s3://makanexpress-glue-scripts/transform_orders.py
     - IAM Role: glue-etl-role
     - Worker type: G.1X
     - Number of workers: 2
     - Glue version: 4.0

  7. IAM Role: glue-crawler-role
     - Trust: glue.amazonaws.com
     - Permissions: S3 read on raw/ and processed/

  8. IAM Role: glue-etl-role
     - Trust: glue.amazonaws.com
     - Permissions: S3 read on raw/, write on processed/, CloudWatch Logs

  9. IAM Role: athena-query-role
     - S3 read on processed/ and analytics/
     - S3 write on query-results bucket

  10. RDS PostgreSQL: makanexpress-warehouse
      - Instance class: db.t3.micro
      - Storage: 20 GB
      - Multi-AZ: No (dev), Yes (production)
      - VPC: Private subnet
      - Security group: Allow 5432 from Airflow/EC2 only
```

**Interview tip:** You don't need to know CloudFormation syntax. But you should be able to LIST the resources needed and explain WHY. This shows architectural thinking.

</details>

---

**Problem 3:** You're optimizing an Athena query that scans 500 GB of data and costs $2.50 per run. It runs 50 times/day. How do you reduce the cost?

<details>
<summary>🔑 Answer</summary>

**Current cost:** $2.50 × 50 = $125/day = $3,750/month. That's significant!

**Optimization strategies (in order of impact):**

1. **Partitioning (BIGGEST IMPACT):**
   - If data is partitioned by `year/month/day` and you always filter by date:
   - Instead of scanning 500 GB, you scan ~17 GB/day (500/30)
   - New cost per query: ~$0.085
   - New daily cost: $0.085 × 50 = $4.25/day = $127.50/month
   - **Savings: ~97%**

2. **Columnar format (Parquet):**
   - If you only SELECT 5 out of 20 columns:
   - Parquet reads only the needed columns → ~4x less data scanned
   - Combined with partitioning: ~$32/month

3. **Data compression:**
   - Parquet with Snappy compression → ~60% smaller files
   - Less data scanned = cheaper queries

4. **Materialized aggregations:**
   - If the query is "daily revenue per restaurant", pre-compute in Glue
   - Store aggregated results in analytics/ zone
   - Query the much smaller aggregation table instead

5. **Query caching:**
   - Athena caches results for repeated identical queries (enable in workgroup)

**Expected result:** $3,750/month → ~$50-100/month with all optimizations.

**The lesson:** Always partition data AND use columnar format. This is non-negotiable for data lakes. Interviewers love asking about cost optimization.

</details>

---

**Problem 4:** Write the IAM policy JSON for a Glue ETL role that needs to:
- Read from `s3://makanexpress-dl/raw/*`
- Write to `s3://makanexpress-dl/processed/*`
- Write logs to CloudWatch
- Nothing else

<details>
<summary>🔑 Answer</summary>

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ReadRawData",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::makanexpress-dl",
                "arn:aws:s3:::makanexpress-dl/raw/*"
            ]
        },
        {
            "Sid": "WriteProcessedData",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::makanexpress-dl",
                "arn:aws:s3:::makanexpress-dl/processed/*"
            ]
        },
        {
            "Sid": "CloudWatchLogs",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        },
        {
            "Sid": "GlueCatalog",
            "Effect": "Allow",
            "Action": [
                "glue:GetDatabase",
                "glue:GetTable",
                "glue:GetTables",
                "glue:CreateTable",
                "glue:UpdateTable",
                "glue:GetPartitions"
            ],
            "Resource": "*"
        }
    ]
}
```

**Key points:**
- Separate statements for raw (read-only) and processed (read-write)
- Bucket ARN AND object ARN needed for ListBucket + GetObject
- CloudWatch logs needed for Glue job logging
- Glue Catalog access needed for crawler/table operations

</details>

---

**Problem 5:** Explain the difference between these PySpark operations with examples: `select()`, `filter()`, `groupBy()`, `withColumn()`, `join()`.

<details>
<summary>🔑 Answer</summary>

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum as spark_sum, avg, count

spark = SparkSession.builder.appName("MakanExpress").getOrCreate()

# Sample data
orders = spark.createDataFrame([
    (1, "C001", "R001", 25.50, "2026-01-15"),
    (2, "C002", "R001", 18.00, "2026-01-15"),
    (3, "C001", "R002", 32.00, "2026-01-16"),
    (4, "C003", "R002", 15.50, "2026-01-16"),
    (5, "C002", "R003", 45.00, "2026-01-17"),
], ["order_id", "customer_id", "restaurant_id", "total_amount", "order_date"])

restaurants = spark.createDataFrame([
    ("R001", "Ah Hock Nasi Lemak", "Malay"),
    ("R002", "Tian Tian Chicken Rice", "Chinese"),
    ("R003", "Prata House", "Indian"),
], ["restaurant_id", "name", "cuisine"])

# 1. SELECT — choose specific columns
orders.select("order_id", "total_amount").show()
orders.select(col("order_id"), col("total_amount") * 1.08).show()  # with GST

# 2. FILTER — keep rows matching condition (= WHERE in SQL)
orders.filter(col("total_amount") > 20).show()
orders.filter((col("customer_id") == "C001") & (col("total_amount") > 25)).show()

# 3. GROUP BY — aggregate data (= GROUP BY in SQL)
orders.groupBy("restaurant_id").agg(
    spark_sum("total_amount").alias("total_revenue"),
    count("*").alias("order_count"),
    avg("total_amount").alias("avg_order_value")
).show()

# 4. WITH COLUMN — add or transform a column (= new column in SQL)
from pyspark.sql.functions import when, lit

orders_with_tier = orders.withColumn(
    "price_tier",
    when(col("total_amount") > 40, "premium")
    .when(col("total_amount") > 20, "standard")
    .otherwise("budget")
)

# 5. JOIN — combine two DataFrames (= JOIN in SQL)
enriched = orders.join(restaurants, on="restaurant_id", how="left")
enriched.select("order_id", "name", "cuisine", "total_amount").show()

# Chained operations (like SQL pipeline)
result = (
    orders
    .join(restaurants, on="restaurant_id", how="left")
    .filter(col("cuisine") == "Malay")
    .groupBy("name")
    .agg(spark_sum("total_amount").alias("revenue"))
    .orderBy(col("revenue").desc())
)
result.show()
```

**SQL ↔ PySpark mapping:**
| SQL | PySpark |
|-----|---------|
| SELECT col | .select(col) |
| WHERE cond | .filter(cond) or .where(cond) |
| GROUP BY col | .groupBy(col).agg(...) |
| AS new_col | .withColumn("name", expr) |
| JOIN table | .join(df, on=key, how=type) |
| ORDER BY col | .orderBy(col) |

</details>

---

## 🌙 Evening (1 hour): Create Your Weak-Spot Tracker

### Template

Create a file `workspace-notes/weak-spot-tracker.md`:

```markdown
# 🎯 Weak-Spot Tracker
# Updated: [today's date]

## 🔴 Critical Weak Spots (Fix This Week)
1. [Topic] — from Week X — [specific skill]
2. [Topic] — from Week X — [specific skill]

## 🟡 Needs Practice (Review Weekly)
1. [Topic] — from Week X
2. [Topic] — from Week X

## 🟢 Strong (Maintain, Don't Obsess)
1. [Topic] — from Week X

## 📅 Practice Plan
- [ ] Monday: [weak spot 1] — 30 min exercises
- [ ] Tuesday: [weak spot 2] — 30 min exercises
- [ ] Wednesday: [weak spot 3] — 30 min exercises
- [ ] Thursday: Review progress, adjust

## 📊 Interview Readiness (Self-Rated 1-5)
| Area | Rating | Notes |
|------|--------|-------|
| SQL (basic-intermediate) | /5 | |
| SQL (advanced, window functions) | /5 | |
| Python (basics + data structures) | /5 | |
| Python (Pandas, ETL) | /5 | |
| Data Modeling | /5 | |
| dbt | /5 | |
| Airflow | /5 | |
| AWS (S3, IAM, RDS) | /5 | |
| PySpark | /5 | |
| AWS Glue + Athena | /5 | |
| Architecture / System Design | /5 | |
| Docker | /5 | |
```

### How to Use This Tracker

1. **Fill it in tonight** based on your morning self-assessment
2. **Update it weekly** — move items between categories as you improve
3. **Use it to plan** your practice sessions in remaining weeks
4. **Before interviews** — review this to know what to study

### Reflection Questions

Write brief answers (2-3 sentences each):

1. **Which week's material felt most natural?** Why do you think that is?

2. **Which week's material felt most foreign?** What would make it click?

3. **If you had an interview tomorrow, what topic would you be most worried about?**

4. **What's your biggest "aha moment" from the past 7 weeks?**

5. **What study habit is working well? What needs to change?**

---

### 📝 Today's Checklist

- [ ] Completed self-assessment for all 7 weeks (70 questions)
- [ ] Rated each week: 🟢🟡🔴
- [ ] Identified top 2-3 weakest areas
- [ ] Completed at least 3 practice problems from weak areas
- [ ] Created `weak-spot-tracker.md` with honest ratings
- [ ] Answered 5 reflection questions
- [ ] Know exactly what to practice next

---

### 🔖 Quick Reference: Self-Assessment Cheat Sheet

```
Week 1: SQL Foundation
  Key skills: SELECT, JOINs, GROUP BY, window functions, CTEs
  Common gap: Window function syntax, NULL handling

Week 2: Python + Pandas
  Key skills: Functions, data structures, file I/O, Pandas, ETL
  Common gap: Error handling, list comprehensions, merge types

Week 3: Data Modeling + dbt
  Key skills: Star schema, SCD types, dbt models, tests, macros
  Common gap: SCD Type 2 SQL, incremental models, macro writing

Week 4: Advanced SQL
  Key skills: Complex joins, query optimization, SCD implementation
  Common gap: EXPLAIN interpretation, self-joins, pivoting

Week 5: Airflow
  Key skills: DAGs, operators, connections, XCom, sensors, testing
  Common gap: XCom limits, task testing, idempotency

Week 6: AWS Core
  Key skills: S3, IAM, RDS, VPC, boto3
  Common gap: IAM policy JSON, VPC networking, boto3 methods

Week 7: Glue + Athena + PySpark
  Key skills: PySpark DataFrames, Glue jobs, Athena queries, partitioning
  Common gap: PySpark vs Pandas differences, Glue job config, Athena costs
```

---

*Day 50 complete! Tomorrow: Portfolio Integration — connect all your projects into a cohesive portfolio.* 📂✨
