# 📅 Day 55 — Tuesday, 8 July 2026
# 📝 Mid-Program Assessment (Weeks 1-8)

---

## 🎯 Today's Goal

This is the big one. A comprehensive assessment covering everything you've learned in 8 weeks: SQL, Python, dbt, Airflow, AWS Core, Glue, Athena, PySpark, and Docker. This simulates what a real technical interview feels like — broad coverage, time pressure, mix of conceptual and practical.

**Philosophy:** You've learned a LOT. Today proves it. Don't panic if you score low on some sections — that's the point. This tells you exactly what to focus on in the remaining 7 weeks.

---

## ☀️ Morning Block (2 hours): Timed Assessment

**Rules:**
- Time: 90 minutes total
- No notes, no Google, no AI
- Write answers on paper or a blank text file
- Be honest — you're only cheating yourself

---

### Section A: SQL (15 minutes, 5 questions)

**Q1.** Write a query to find the top 5 restaurants by total revenue in June 2026, including their cuisine type and average order value. Use `fact_orders` and `dim_restaurant`.

**Q2.** Explain the difference between `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()`. When would you use each? Give a MakanExpress example.

**Q3.** Write a recursive CTE that generates a date dimension table for all of 2026 (one row per day, columns: date_key, full_date, day_name, month_name, quarter, is_weekend).

**Q4.** A query is slow. The execution plan shows a sequential scan on `fact_orders` (50M rows). List 4 techniques to make it faster.

**Q5.** Write a query using a window function to calculate a running total of daily revenue and the 7-day moving average for MakanExpress orders.

---

### Section B: Python & Pandas (15 minutes, 5 questions)

**Q6.** Write a Python function that reads a CSV from S3, cleans it (remove nulls, standardize dates), and returns a DataFrame. Include error handling and logging.

**Q7.** You have two DataFrames: `orders` (1M rows) and `customers` (100K rows). Write a Pandas merge that:
- Keeps all orders even if customer is missing
- Adds customer_name, city, and membership_tier
- Handles duplicate column names

**Q8.** Write a function that implements a simple ETL pipeline: extract from API (with retry logic), transform with Pandas (calculate new columns), load to PostgreSQL.

**Q9.** What is the difference between `apply()`, `map()`, and `applymap()` in Pandas? When would you use each?

**Q10.** Write a Python script that uses list comprehension, dictionary comprehension, and a generator expression. Explain when a generator is better than a list.

---

### Section C: Data Modeling & dbt (12 minutes, 4 questions)

**Q11.** Design a star schema for a GrabFood-like food delivery platform. List the fact table, 4 dimension tables, and key columns in each. Explain your grain.

**Q12.** Explain SCD Type 2. Write the SQL to detect and insert a new version of a customer record when their city changes.

**Q13.** In dbt, what is the difference between a `source`, a `staging model`, and a `mart model`? Why this 3-layer approach?

**Q14.** Write a dbt macro called `surrogate_key` that takes any number of column names and generates a SQL expression to create a hash-based surrogate key.

---

### Section D: Airflow (12 minutes, 4 questions)

**Q15.** Draw (describe) a DAG for MakanExpress daily pipeline with 6 tasks: extract_orders, extract_customers, validate_data, load_warehouse, run_dbt_models, send_slack_notification. Show dependencies and trigger rules.

**Q16.** Write a PythonOperator task that:
- Queries PostgreSQL for yesterday's order count
- Returns the count via XCom
- Has retry logic (3 retries, exponential backoff)
- Is idempotent

**Q17.** Explain the difference between `logical_date` and actual execution time. A DAG runs daily at 6 AM SGT. The task processes "yesterday's" data. What `logical_date` value does it use?

**Q18.** Your Airflow task failed at 3 AM. The error is "connection refused" to PostgreSQL. List your debugging steps in order.

---

### Section E: AWS Core (12 minutes, 4 questions)

**Q19.** Design an S3 data lake structure for MakanExpress. Show the prefix hierarchy, explain format choices (CSV vs Parquet), and write a lifecycle policy.

**Q20.** Write an IAM policy that allows a Glue job to:
- Read from `s3://makanexpress-data-lake/raw/*`
- Write to `s3://makanexpress-data-lake/processed/*`
- List both prefixes
- Nothing else

**Q21.** Your RDS PostgreSQL is in a private subnet. Three developers need to query it from their laptops. Describe 3 ways to provide access and the security trade-offs.

**Q22.** Explain VPC, subnet (public vs private), security group, and NACL using a Singapore building analogy. How does traffic flow from the internet to an RDS instance?

---

### Section F: AWS Glue + Athena + PySpark (15 minutes, 5 questions)

**Q23.** Write PySpark code to:
1. Read a CSV from S3 into a DataFrame
2. Filter orders > $50
3. Add a column `order_month` derived from `order_date`
4. Group by `restaurant_id` and `order_month`, calculating total revenue and order count
5. Write the result to S3 as Parquet

**Q24.** Explain what a Glue Crawler does, step by step. What happens when you run it? What does it create? What happens if the schema changes?

**Q25.** Write an Athena query that:
- Queries a partitioned table (partitioned by `year` and `month`)
- Uses partition pruning (only scans June 2026)
- Joins orders with restaurants
- Calculates top 10 restaurants by revenue
- Explain why partitioning saves money

**Q26.** Compare: Glue vs Athena vs Redshift. When would you use each in a data platform?

**Q27.** Design a serverless pipeline: raw CSV lands in S3 → processed by Glue → queried by Athena. Include: S3 structure, Glue crawler config, Glue job (PySpark), Athena DDL, and cost estimate for processing 10GB/day.

---

### Section G: Docker (10 minutes, 3 questions)

**Q28.** Explain Docker in 3 sentences to a non-technical PM. Then explain containers vs VMs.

**Q29.** Write a `docker-compose.yml` that runs:
- PostgreSQL (port 5432, with init script, persistent volume)
- pgAdmin (port 5050, connected to PostgreSQL)
- Airflow webserver + scheduler + Postgres backend

**Q30.** Write a Dockerfile for a Python ETL script that:
- Uses Python 3.11 slim
- Installs dependencies from requirements.txt
- Copies the ETL script
- Runs as a non-root user
- Sets environment variables for config

---

## 🌤️ Afternoon Block (2 hours): Answers & Score Breakdown

### Scoring Guide

| Section | Questions | Points Each | Total |
|---------|-----------|-------------|-------|
| A: SQL | 5 | 8 | 40 |
| B: Python | 5 | 8 | 40 |
| C: Data Modeling + dbt | 4 | 8 | 32 |
| D: Airflow | 4 | 8 | 32 |
| E: AWS Core | 4 | 8 | 32 |
| F: Glue + Athena + PySpark | 5 | 8 | 40 |
| G: Docker | 3 | 6 | 18 |
| **Total** | **30** | | **234** |

**Grade scale:**
- 200+ (85%+): 🔥 Excellent — you're on track
- 160-199 (68-85%): 👍 Good — targeted review needed
- 120-159 (51-68%): ⚠️ Needs work — significant review required
- Below 120 (<51%): 🔄 Major gaps — consider re-doing weak weeks

---

### ✅ Full Answers

<details>
<summary>🔑 Section A: SQL</summary>

**A1:**
```sql
SELECT 
    r.restaurant_name,
    r.cuisine_type,
    COUNT(*) AS total_orders,
    SUM(f.order_amount) AS total_revenue,
    ROUND(AVG(f.order_amount), 2) AS avg_order_value
FROM fact_orders f
JOIN dim_restaurant r ON f.restaurant_id = r.restaurant_id
WHERE f.order_date >= '2026-06-01' 
  AND f.order_date < '2026-07-01'
GROUP BY r.restaurant_name, r.cuisine_type
ORDER BY total_revenue DESC
LIMIT 5;
```

**A2:**
- `ROW_NUMBER()`: Always sequential (1,2,3,4,5). Use when you need exactly N rows (top 3 per category).
- `RANK()`: Same rank for ties, then gaps (1,2,2,4). Use for competitive ranking (e.g., restaurant rankings).
- `DENSE_RANK()`: Same rank for ties, no gaps (1,2,2,3). Use when you want to know "how many groups are above me."

MakanExpress example: Ranking restaurants by revenue — two restaurants tie at $50K:
- ROW_NUMBER: 1, 2, 3 (arbitrary order for ties)
- RANK: 1, 2, 2, 4 (next rank is 4)
- DENSE_RANK: 1, 2, 2, 3 (no gap)

**A3:**
```sql
WITH RECURSIVE date_series AS (
    SELECT 
        CAST('2026-01-01' AS DATE) AS full_date
    UNION ALL
    SELECT full_date + INTERVAL '1 day'
    FROM date_series
    WHERE full_date < '2026-12-31'
)
SELECT 
    EXTRACT(YEAR FROM full_date) * 10000 + 
    EXTRACT(MONTH FROM full_date) * 100 + 
    EXTRACT(DAY FROM full_date) AS date_key,
    full_date,
    TRIM(TO_CHAR(full_date, 'Day')) AS day_name,
    TRIM(TO_CHAR(full_date, 'Month')) AS month_name,
    EXTRACT(QUARTER FROM full_date) AS quarter,
    CASE WHEN EXTRACT(ISODOW FROM full_date) IN (6, 7) THEN TRUE ELSE FALSE END AS is_weekend
FROM date_series
ORDER BY full_date;
```

**A4:**
1. Add an index on the filter column (e.g., `CREATE INDEX ON fact_orders(order_date)`)
2. Use partitioning (partition the table by date range)
3. Rewrite the query to filter early (WHERE clause before joins)
4. Use `ANALYZE` to update statistics so the planner makes better decisions
5. Limit columns selected (don't use SELECT *)

**A5:**
```sql
SELECT 
    order_date,
    SUM(daily_revenue) AS daily_revenue,
    SUM(SUM(daily_revenue)) OVER (ORDER BY order_date) AS running_total,
    ROUND(AVG(SUM(daily_revenue)) OVER (
        ORDER BY order_date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_7d
FROM (
    SELECT order_date, SUM(order_amount) AS daily_revenue
    FROM fact_orders
    GROUP BY order_date
) daily
ORDER BY order_date;
```

</details>

<details>
<summary>🔑 Section B: Python & Pandas</summary>

**B6:**
```python
import boto3
import pandas as pd
import logging
from io import BytesIO

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def read_clean_csv_from_s3(bucket: str, key: str) -> pd.DataFrame:
    """Read CSV from S3, clean it, return DataFrame."""
    try:
        logger.info(f"Reading s3://{bucket}/{key}")
        s3 = boto3.client('s3')
        response = s3.get_object(Bucket=bucket, Key=key)
        df = pd.read_csv(BytesIO(response['Body'].read()))
        
        logger.info(f"Raw data: {len(df)} rows, {len(df.columns)} columns")
        
        # Clean
        df.dropna(subset=['order_id', 'order_date'], inplace=True)
        df['order_date'] = pd.to_datetime(df['order_date'], errors='coerce')
        df = df[df['order_date'].notna()]  # Drop rows where date parsing failed
        
        logger.info(f"Cleaned data: {len(df)} rows")
        return df
        
    except Exception as e:
        logger.error(f"Failed to read {key}: {e}")
        raise
```

**B7:**
```python
merged = orders.merge(
    customers[['customer_id', 'customer_name', 'city', 'membership_tier']],
    on='customer_id',
    how='left',  # keeps all orders even if customer missing
    suffixes=('_order', '_customer')  # handles duplicate column names
)
```

**B8:**
```python
import requests
import pandas as pd
import psycopg2
import time
import logging

logger = logging.getLogger(__name__)

def extract(api_url: str, retries: int = 3) -> dict:
    """Extract data from API with retry logic."""
    for attempt in range(retries):
        try:
            response = requests.get(api_url, timeout=30)
            response.raise_for_status()
            return response.json()
        except requests.RequestException as e:
            logger.warning(f"Attempt {attempt+1} failed: {e}")
            if attempt < retries - 1:
                time.sleep(2 ** attempt)  # exponential backoff
            else:
                raise

def transform(data: list) -> pd.DataFrame:
    """Transform raw data into analytics-ready DataFrame."""
    df = pd.DataFrame(data)
    df['total_amount'] = df['subtotal'] + df['delivery_fee'] + df['tax']
    df['order_date'] = pd.to_datetime(df['created_at']).dt.date
    return df

def load(df: pd.DataFrame, conn_string: str, table: str):
    """Load DataFrame to PostgreSQL."""
    with psycopg2.connect(conn_string) as conn:
        df.to_sql(table, conn, if_exists='append', index=False)
    logger.info(f"Loaded {len(df)} rows to {table}")

def run_pipeline(api_url: str, conn_string: str):
    """Full ETL pipeline."""
    raw_data = extract(api_url)
    df = transform(raw_data)
    load(df, conn_string, 'staging_orders')
```

**B9:**
- `map()`: Series only. Applies function element-wise. Use for transforming one column: `df['city'].map({'SG': 'Singapore'})`
- `apply()`: Series or DataFrame. Series: element-wise function. DataFrame: row/column-wise. Use for complex logic: `df.apply(lambda row: row['a'] + row['b'], axis=1)`
- `applymap()`: DataFrame only (deprecated, use `map()` now). Element-wise for entire DataFrame. Use for formatting: `df.applymap(lambda x: f"${x:.2f}")` (old style)

**B10:**
```python
# List comprehension — creates full list in memory
squares = [x**2 for x in range(1000)]

# Dict comprehension — creates dict
word_lengths = {word: len(word) for word in ['hello', 'world', 'data']}

# Generator expression — lazy evaluation, one item at a time
total = sum(x**2 for x in range(1_000_000))  # doesn't create a million-item list

# Generator is better when:
# 1. Processing large data that doesn't fit in memory
# 2. Only need to iterate once (sum, any, all, max)
# 3. Pipeline processing (read CSV row by row)
```

</details>

<details>
<summary>🔑 Section C: Data Modeling & dbt</summary>

**C11:**

**Fact table:** `fact_orders`
- Grain: One row per order item
- Columns: order_key (surrogate), order_id, customer_key, restaurant_key, date_key, menu_item_key, quantity, order_amount, delivery_fee, discount_amount

**Dimension tables:**
1. `dim_customer` — customer_key, customer_id, name, phone, email, city, membership_tier, signup_date
2. `dim_restaurant` — restaurant_key, restaurant_id, name, cuisine_type, city, avg_rating, price_range, is_active
3. `dim_date` — date_key, full_date, day_name, month_name, quarter, is_weekend, is_holiday
4. `dim_menu_item` — item_key, item_id, name, category, price, cuisine_type, is_available

**C12:**
```sql
-- Detect changes
SELECT c.customer_id, c.current_city, s.new_city
FROM dim_customer c
JOIN staging_customers s ON c.customer_id = s.customer_id
WHERE c.current_city != s.new_city
  AND c.is_current = TRUE;

-- Expire old record
UPDATE dim_customer
SET is_current = FALSE, end_date = CURRENT_DATE
WHERE customer_id IN (
    SELECT s.customer_id FROM staging_customers s
    JOIN dim_customer c ON c.customer_id = s.customer_id
    WHERE c.city != s.city AND c.is_current = TRUE
);

-- Insert new record
INSERT INTO dim_customer (customer_id, name, city, is_current, start_date, end_date)
SELECT s.customer_id, s.name, s.city, TRUE, CURRENT_DATE, NULL
FROM staging_customers s
JOIN dim_customer c ON c.customer_id = s.customer_id
WHERE c.city != s.city AND c.is_current = TRUE;
```

**C13:**
- **Source:** Raw data reference (a table or view in your database). Defined in `sources.yml`. Not managed by dbt — it's the input. Purpose: track freshness, lineage origin.
- **Staging model:** Light transformation on raw data (rename columns, cast types, remove duplicates). One staging model per source table. Purpose: clean interface, hide raw complexity.
- **Mart model:** Business logic layer. Joins, aggregations, metrics. What analysts query. Purpose: business-ready data, single source of truth.

Why 3 layers: separation of concerns. If raw schema changes, fix staging. If business logic changes, fix marts. Neither cascades into the other.

**C14:**
```sql
-- macros/surrogate_key.sql
{% macro surrogate_key(columns) %}
    MD5({% for col in columns %}COALESCE(CAST({{ col }} AS VARCHAR), ''){% if not loop.last %} || '-' || {% endif %}{% endfor %})
{% endmacro %}

-- Usage in a model:
SELECT 
    {{ surrogate_key(['restaurant_id', 'order_date', 'item_id']) }} AS order_item_key,
    *
FROM {{ ref('stg_orders') }}
```

</details>

<details>
<summary>🔑 Section D: Airflow</summary>

**D15:**
```
extract_orders ─────┐
                     ├──▶ validate_data ──▶ load_warehouse ──▶ run_dbt_models ──▶ send_slack_notification
extract_customers ──┘
```
- Dependencies: extract_orders + extract_customers → validate_data → load_warehouse → run_dbt → slack
- Trigger rules: `send_slack_notification` should use `trigger_rule=ONE_SUCCESS` or `ALL_DONE` so it sends even on failure

**D16:**
```python
from airflow.decorators import task
from airflow.providers.postgres.hooks.postgres import PostgresHook

@task(retries=3, retry_delay=exponential_backoff(min=60, max=600))
def get_order_count(**context):
    ds = context['ds']  # logical_date as YYYY-MM-DD (idempotent key)
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    sql = """
        SELECT COUNT(*) 
        FROM fact_orders 
        WHERE order_date = %s
    """
    count = hook.get_first(sql, parameters=(ds,))[0]
    return count  # auto-pushed to XCom
```

**D17:**
- `logical_date` = the date the DAG run is scheduled FOR (e.g., 2026-07-07)
- Actual execution time = when the task actually runs (e.g., 2026-07-08 06:00 AM)
- For "yesterday's data": The DAG scheduled for July 7 (logical_date) runs at 6 AM on July 8 (execution time). The task uses `{{ ds }}` = '2026-07-07' to process July 7's data.

**D18:**
1. Check Airflow task logs — exact error message, stack trace
2. Verify connection: Can Airflow scheduler reach RDS? Check connection config
3. Check RDS status: Is the instance running? Check AWS console
4. Check security group: Is Airflow's IP allowed on port 5432?
5. Check credentials: Username/password correct in Airflow connection?
6. If it was a transient failure, clear the task and retry
7. If persistent, fix the root cause, then clear and retry

</details>

<details>
<summary>🔑 Section E: AWS Core</summary>

**E19:**
```
s3://makanexpress-data-lake/
├── raw/                          # Landing zone (CSV, JSON)
│   ├── orders/year=2026/month=07/
│   ├── customers/year=2026/month=07/
│   └── menu_items/year=2026/month=07/
├── processed/                    # Cleaned + typed (Parquet)
│   ├── orders/year=2026/month=07/
│   ├── customers/year=2026/month=07/
│   └── menu_items/year=2026/month=07/
└── analytics/                    # Pre-aggregated (Parquet)
    ├── daily_revenue/
    ├── top_restaurants/
    └── customer_metrics/
```

Formats: raw = CSV (human-readable, debuggable), processed/analytics = Parquet (columnar, compressed, fast).

Lifecycle:
```python
{
    "Rules": [{
        "ID": "RawDataLifecycle",
        "Filter": {"Prefix": "raw/"},
        "Status": "Enabled",
        "Transitions": [
            {"Days": 30, "StorageClass": "STANDARD_IA"},
            {"Days": 90, "StorageClass": "GLACIER"}
        ],
        "Expiration": {"Days": 365}
    }]
}
```

**E20:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::makanexpress-data-lake",
                "arn:aws:s3:::makanexpress-data-lake/raw/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::makanexpress-data-lake/processed/*"
        }
    ]
}
```

**E21:**
1. **Bastion host (jump server):** EC2 in public subnet, SSH tunnel to RDS. Pros: standard, well-known. Cons: another server to manage, SSH key distribution.
2. **AWS Systems Manager Session Manager:** No bastion needed, connects through AWS API. Pros: no open ports, audit trail. Cons: needs SSM agent, slightly slower.
3. **VPN (AWS Client VPN):** Connects laptop to VPC network. Pros: most secure, works for all services. Cons: setup complexity, per-connection-hour cost.

Best for a small team: Session Manager (no infrastructure to manage).

**E22:**
- **VPC** = The building compound (guarded, gated community). Your private network.
- **Subnets** = Rooms in the building. Public subnet = lobby (anyone can enter). Private subnet = server room (only authorized people).
- **Security Group** = Badge reader on a specific door (instance-level firewall). Stateful — if you allow inbound, response is auto-allowed.
- **NACL** = Building security policy at the gate (subnet-level firewall). Stateless — must explicitly allow both directions.

Traffic flow: Internet → IGW → Public Subnet (bastion) → Security Group check → Private Subnet → NACL check → Security Group on RDS → RDS instance.

</details>

<details>
<summary>🔑 Section F: Glue + Athena + PySpark</summary>

**F23:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, month, sum, count

spark = SparkSession.builder.appName("MakanExpressAnalytics").getOrCreate()

# 1. Read CSV
df = spark.read.csv("s3://makanexpress-data-lake/raw/orders/", 
                     header=True, inferSchema=True)

# 2. Filter
high_value = df.filter(col("order_amount") > 50)

# 3. Add column
with_month = high_value.withColumn("order_month", month("order_date"))

# 4. Aggregate
summary = with_month.groupBy("restaurant_id", "order_month").agg(
    sum("order_amount").alias("total_revenue"),
    count("*").alias("order_count")
)

# 5. Write Parquet
summary.write.mode("overwrite").parquet(
    "s3://makanexpress-data-lake/processed/restaurant_monthly/"
)
```

**F24:**
1. You point the crawler at an S3 path (e.g., `s3://bucket/raw/orders/`)
2. Crawler scans the data files (reads headers, infers types)
3. Creates/updates a table in the Glue Data Catalog
4. The table has: name, columns, types, location, format, partition keys
5. If schema changes (new column, type change): crawler updates the catalog. Existing queries may break if columns were removed or renamed. New columns are added.

**F25:**
```sql
-- Athena query with partition pruning
SELECT 
    r.restaurant_name,
    r.cuisine_type,
    SUM(o.order_amount) AS total_revenue
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE o.year = 2026 
  AND o.month = 6
GROUP BY r.restaurant_name, r.cuisine_type
ORDER BY total_revenue DESC
LIMIT 10;
```

Partitioning saves money because Athena charges per TB scanned. Without partitioning, scanning ALL orders ever = 500GB = $2.50/query. With partitioning (year=2026, month=6) = 5GB = $0.025/query. That's 100x cheaper.

**F26:**
- **Glue:** ETL engine. Use when you need to TRANSFORM data (clean, join, aggregate, convert format). Serverless Spark.
- **Athena:** Query engine. Use when you need to ANALYZE data already in S3. SQL only, no transformation, serverless.
- **Redshift:** Data warehouse. Use when you need a dedicated warehouse for BI dashboards, complex analytics, many concurrent users. Not serverless (provisioned clusters).

Typical MakanExpress platform: S3 raw → Glue transforms → S3 processed → Athena for ad-hoc queries → (optional) Redshift for BI dashboards.

**F27:**
```
Architecture:
CSV → S3 raw/ → S3 Event → Glue Trigger → Glue PySpark Job → S3 processed/ → Glue Crawler → Data Catalog → Athena → Analytics

S3 structure:
s3://makanexpress-pipeline/
├── raw/orders/          # Incoming CSV
├── processed/orders/    # Parquet, partitioned by year/month
└── scripts/             # Glue PySpark scripts

Glue Crawler: points at processed/orders/, runs on schedule or trigger
Glue Job: PySpark script reads raw CSV, cleans, writes Parquet
Athena: CREATE EXTERNAL TABLE with partitioning

Cost estimate (10GB/day, Singapore):
- S3 storage: 10GB × 30 days = 300GB × $0.023/GB = $6.90/month
- Glue ETL: ~10 min/job × 1 DPU × $0.44/DPU-hour × 30 = ~$2.20/month
- Athena queries: ~50 queries/day × 5GB scanned × $5/TB = ~$37.50/month
- Total: ~$47/month (well within free tier limits for practice)
```

</details>

<details>
<summary>🔑 Section G: Docker</summary>

**G28:**
"Docker is like a shipping container for software. It packages your code with everything it needs to run — the right Python version, libraries, settings — so it works identically on any computer. No more 'works on my machine' problems."

Containers vs VMs:
- VM: Full operating system per VM (heavy, GBs of RAM, minutes to start)
- Container: Shares the host OS kernel, only packages the app + dependencies (lightweight, MBs, seconds to start)
- Analogy: VM = owning a whole apartment. Container = renting a room in a shared house.

**G29:**
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: makanexpress
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d makanexpress"]
      interval: 10s
      timeout: 5s
      retries: 5

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@makanexpress.sg
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "5050:80"
    depends_on:
      postgres:
        condition: service_healthy

  airflow-db:
    image: postgres:15
    environment:
      POSTGRES_DB: airflow
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow123
    volumes:
      - airflow_db_data:/var/lib/postgresql/data

  airflow-init:
    image: apache/airflow:2.7.0
    depends_on:
      - airflow-db
    environment:
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow123@airflow-db/airflow
    command: airflow db init

  airflow-webserver:
    image: apache/airflow:2.7.0
    depends_on:
      airflow-init:
        condition: service_completed_successfully
    ports:
      - "8080:8080"
    environment:
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow123@airflow-db/airflow
      AIRFLOW__CORE__EXECUTOR: SequentialExecutor
    command: airflow webserver
    volumes:
      - ./dags:/opt/airflow/dags

  airflow-scheduler:
    image: apache/airflow:2.7.0
    depends_on:
      airflow-init:
        condition: service_completed_successfully
    environment:
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow123@airflow-db/airflow
      AIRFLOW__CORE__EXECUTOR: SequentialExecutor
    command: airflow scheduler
    volumes:
      - ./dags:/opt/airflow/dags

volumes:
  postgres_data:
  airflow_db_data:
```

**G30:**
```dockerfile
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY etl_script.py .

# Create non-root user
RUN useradd -m appuser
USER appuser

# Environment variables (override at runtime)
ENV DB_HOST=makanexpress-db.cxxxxx.ap-southeast-1.rds.amazonaws.com
ENV DB_NAME=makanexpress
ENV S3_BUCKET=makanexpress-data-lake

# Run the ETL script
CMD ["python", "etl_script.py"]
```

</details>

---

## 🌙 Evening (1 hour): Score Analysis + Improvement Plan

### Score Breakdown Template

```
Section                    Score    Max    %
─────────────────────────────────────────
A: SQL                     ___/40       ___%
B: Python                  ___/40       ___%
C: Data Modeling + dbt     ___/32       ___%
D: Airflow                 ___/32       ___%
E: AWS Core                ___/32       ___%
F: Glue + Athena + PySpark ___/40       ___%
G: Docker                  ___/18       ___%
─────────────────────────────────────────
TOTAL                      ___/234      ___%
```

### Targeted Improvement Plan

Based on your weakest 2-3 sections, create a focused plan:

```markdown
## My Weak Spots (Ordered by Priority)

1. **[Weakest Section]** — Score: ___%
   - Specific gaps: [list what you got wrong]
   - Days to review: [which days to re-read]
   - Practice plan: [what exercises to re-do]
   - Target: 70%+ by Week 12

2. **[Second Weakest]** — Score: ___%
   - Specific gaps: ...
   - Days to review: ...
   - Practice plan: ...
   - Target: 70%+ by Week 12

3. **[Third Weakest]** — Score: ___%
   - ...
```

### What the Remaining Weeks Cover

| Week | Focus | Which Weak Spots It Helps |
|------|-------|--------------------------|
| 9 | Advanced Python + APIs + LLM | Python, API integration |
| 10 | Data Quality + Testing + Interview Prep | SQL, Python, all areas |
| 11 | Git Advanced + CI/CD + Mock Interviews | Professional skills |
| 12 | Resume + LinkedIn + Portfolio | Communication |
| 13-14 | Interview Prep (SQL + System Design) | SQL, Python, architecture |
| 15 | Final Push + Applications | Everything |

### 📝 Today's Checklist

- [ ] Completed 30-question assessment honestly (no notes)
- [ ] Scored yourself using the answer key
- [ ] Filled in the score breakdown table
- [ ] Identified top 3 weak areas
- [ ] Created targeted improvement plan
- [ ] Noted specific gaps within each weak area
- [ ] Ready to start Week 9 tomorrow with awareness of weak spots

---

*Day 55 complete! Tomorrow: GitHub Pages master portfolio — your public face to employers.* 🌐
