# 📅 Day 45 — Saturday, 28 June 2026
# 🔍 AWS Athena: Query S3 Data with SQL + Partitioning Strategies

---

## 🎯 Today's Goal

AWS Athena lets you run SQL queries directly on files in S3 — no database server, no loading data, just point and query. It's the fastest way to analyze data in your data lake. Today you'll learn Athena, write queries against the data you crawled yesterday, and master partitioning for performance and cost savings.

**Why Athena matters:**
- "SQL on S3" — the simplest analytics interface in AWS
- Pay per query: $5 per TB scanned (partitioning reduces this drastically)
- Used by: Grab analytics, Shopee BI teams, virtually every AWS data team
- Interview favorite: "How would you query 500GB of data without a database?"

> **⏱️ Time budget:** Morning (2h) + Afternoon (2h) + Evening (1h) = 5 hours
> **💰 Cost:** $5/TB scanned. Free Tier: 1TB free for 90 days.

---

## 🧠 Mental Model: What Athena Actually Does

```
Traditional Query (RDS/PostgreSQL):
  SQL → Database engine → Reads from disk → Returns results
  Data MUST be loaded into the database first

Athena Query:
  SQL → Athena (Presto engine) → Reads from S3 → Returns results
  Data stays in S3 files — no loading needed!

┌───────────────────────────────────────────────────────┐
│  Your SQL: "SELECT city, AVG(rating) FROM restaurants │
│             GROUP BY city"                             │
└──────────────────────┬────────────────────────────────┘
                       │
┌──────────────────────▼────────────────────────────────┐
│  Athena (serverless Presto)                            │
│  1. Looks up "restaurants" in Glue Catalog            │
│  2. Finds: s3://makanexpress-datalake-dev/raw/rest..  │
│  3. Reads CSV files from S3                           │
│  4. Runs Presto SQL engine on the data                │
│  5. Returns results to you                            │
└───────────────────────────────────────────────────────┘
```

**Key dependencies:**
- Athena uses the **Glue Catalog** to know WHERE your data is and WHAT schema it has
- That's why you created crawlers yesterday — Athena needs the catalog!
- Data stays in S3 → Athena reads it on every query (no caching by default)

---

## ☀️ Morning Block (2 hours): Athena Basics + First Queries

### Step 1: Set Up Athena Workgroup (10 min)

```bash
# Create a workgroup (organizes queries and controls costs)
aws athena create-work-group \
    --name makanexpress \
    --description "MakanExpress analytics queries" \
    --configuration '{
        "ResultConfiguration": {
            "OutputLocation": "s3://makanexpress-datalake-dev/athena-results/"
        },
        "EnforceWorkGroupConfiguration": true
    }' \
    --region ap-southeast-1

# Athena writes query results to S3 — set the output location
aws s3api put-object \
    --bucket makanexpress-datalake-dev \
    --key athena-results/ \
    --region ap-southeast-1
```

**Console method:**
1. Go to **Athena Console** → **Workgroups** → **Create workgroup**
2. Name: `makanexpress`
3. Query result location: `s3://makanexpress-datalake-dev/athena-results/`
4. Click **Create**

### Step 2: Your First Athena Query (15 min)

**Console method (easiest for exploration):**
1. Go to **Athena Console** → **Query editor**
2. Select workgroup: `makanexpress`
3. Make sure Database is set to: `makanexpress_catalog` (from Glue crawler)
4. You should see tables: `restaurants` and `orders`

```sql
-- First query: see all restaurants
SELECT * FROM restaurants LIMIT 10;
```

Click **Run** — results appear in seconds!

```sql
-- Filter for Singapore restaurants
SELECT name, city, rating, monthly_orders
FROM restaurants
WHERE city = 'Singapore'
ORDER BY rating DESC;
```

**CLI method:**

```bash
# Run a query via CLI
QUERY_ID=$(aws athena start-query-execution \
    --query-string "SELECT * FROM makanexpress_catalog.restaurants LIMIT 10" \
    --work-group makanexpress \
    --query 'QueryExecutionId' \
    --output text \
    --region ap-southeast-1)

echo "Query ID: $QUERY_ID"

# Wait for query to complete
aws athena get-query-execution \
    --query-execution-id "$QUERY_ID" \
    --query 'QueryExecution.Status.State' \
    --region ap-southeast-1

# Get results (when state = SUCCEEDED)
aws athena get-query-results \
    --query-execution-id "$QUERY_ID" \
    --region ap-southeast-1
```

### Step 3: Athena SQL — Full Power (30 min)

Athena uses **Presto SQL** (not exactly PostgreSQL). Most SQL works the same, but there are differences.

```sql
-- ========================================
-- Basic Queries (same as PostgreSQL)
-- ========================================

-- Count restaurants per city
SELECT 
    city,
    COUNT(*) AS num_restaurants,
    ROUND(AVG(rating), 2) AS avg_rating
FROM restaurants
GROUP BY city
ORDER BY num_restaurants DESC;

-- Top-rated restaurants
SELECT name, city, rating
FROM restaurants
WHERE rating >= 4.5
ORDER BY rating DESC;

-- ========================================
-- Athena-Specific Functions
-- ========================================

-- JSON extraction (for JSON data)
SELECT 
    order_id,
    json_extract_scalar(json_parse(line), '$.amount') AS amount
FROM orders;

-- Date functions
SELECT 
    order_date,
    DATE_PARSE(order_date, '%Y-%m-%d') AS parsed_date,
    YEAR(DATE_PARSE(order_date, '%Y-%m-%d')) AS order_year,
    MONTH(DATE_PARSE(order_date, '%Y-%m-%d')) AS order_month
FROM orders;

-- CAST (same as PostgreSQL)
SELECT 
    CAST(amount AS DOUBLE) AS amount_double,
    CAST(order_date AS DATE) AS order_date_date
FROM orders;

-- ========================================
-- Joins (same as PostgreSQL)
-- ========================================

-- Join orders with restaurants
SELECT 
    r.name,
    r.city,
    COUNT(o.order_id) AS order_count,
    ROUND(SUM(CAST(o.amount AS DOUBLE)), 2) AS total_revenue
FROM orders o
INNER JOIN restaurants r ON o.restaurant_id = r.restaurant_id
GROUP BY r.name, r.city
ORDER BY total_revenue DESC;

-- ========================================
-- Window Functions (same as PostgreSQL!)
-- ========================================

-- Rank restaurants by rating within each city
SELECT 
    name,
    city,
    rating,
    RANK() OVER (PARTITION BY city ORDER BY rating DESC) AS city_rank,
    RANK() OVER (ORDER BY rating DESC) AS overall_rank
FROM restaurants;

-- Running total of orders per restaurant
SELECT 
    o.restaurant_id,
    r.name,
    o.order_date,
    CAST(o.amount AS DOUBLE) AS amount,
    SUM(CAST(o.amount AS DOUBLE)) OVER (
        PARTITION BY o.restaurant_id 
        ORDER BY o.order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
ORDER BY o.restaurant_id, o.order_date;

-- ========================================
-- CTEs (same as PostgreSQL)
-- ========================================

WITH city_stats AS (
    SELECT 
        city,
        COUNT(*) AS num_restaurants,
        ROUND(AVG(rating), 2) AS avg_rating,
        SUM(monthly_orders) AS total_monthly_orders
    FROM restaurants
    GROUP BY city
),
ranked AS (
    SELECT 
        *,
        RANK() OVER (ORDER BY total_monthly_orders DESC) AS order_rank
    FROM city_stats
)
SELECT * FROM ranked WHERE order_rank <= 3;
```

### Exercise 1: Write Athena Queries

Using the `restaurants` and `orders` tables:

**A.** Find the average order amount per restaurant, showing restaurant name, city, order count, and average. Round to 2 decimal places.

**B.** Find restaurants that have NO orders (LEFT JOIN where order_id IS NULL).

**C.** Using a window function, calculate what percentage each restaurant's total revenue is of the overall total revenue.

**D.** Find the city with the highest average order value.

<details>
<summary>🔑 Answer</summary>

```sql
-- A: Average order per restaurant
SELECT 
    r.name,
    r.city,
    COUNT(o.order_id) AS order_count,
    ROUND(AVG(CAST(o.amount AS DOUBLE)), 2) AS avg_order_value
FROM restaurants r
INNER JOIN orders o ON r.restaurant_id = o.restaurant_id
GROUP BY r.name, r.city
ORDER BY avg_order_value DESC;

-- B: Restaurants with NO orders
SELECT r.name, r.city
FROM restaurants r
LEFT JOIN orders o ON r.restaurant_id = o.restaurant_id
WHERE o.order_id IS NULL;

-- C: Revenue percentage
WITH restaurant_revenue AS (
    SELECT 
        r.name,
        SUM(CAST(o.amount AS DOUBLE)) AS total_revenue
    FROM restaurants r
    INNER JOIN orders o ON r.restaurant_id = o.restaurant_id
    GROUP BY r.name
)
SELECT 
    name,
    total_revenue,
    ROUND(total_revenue / SUM(total_revenue) OVER () * 100, 2) AS pct_of_total
FROM restaurant_revenue
ORDER BY total_revenue DESC;

-- D: City with highest avg order value
SELECT 
    r.city,
    ROUND(AVG(CAST(o.amount AS DOUBLE)), 2) AS avg_order_value
FROM restaurants r
INNER JOIN orders o ON r.restaurant_id = o.restaurant_id
GROUP BY r.city
ORDER BY avg_order_value DESC
LIMIT 1;
```

</details>

### Step 4: Athena vs PostgreSQL — Key Differences (15 min)

| Feature | PostgreSQL | Athena (Presto) |
|---------|-----------|-----------------|
| Data location | Inside database (disk) | S3 files |
| Data loading | INSERT, COPY | Already in S3 |
| Cost | Per-hour (RDS instance) | Per-TB scanned |
| Updates | INSERT/UPDATE/DELETE | **Read-only** (no updates!) |
| Transactions | ACID | Limited (no transactions) |
| Indexes | B-tree, GIN, etc. | **Partitioning only** |
| Data formats | Internal | CSV, JSON, Parquet, ORC |
| Concurrency | High | Up to 100 concurrent queries |
| Best for | OLTP (app database) | OLAP (analytics/queries) |

**Critical rule: Athena is READ-ONLY on S3 data.** You can't UPDATE or DELETE rows. To "update" data:
1. Write a new Parquet file with the updated data
2. Replace the old file
3. Or use partitioning — write new partition, drop old one

### Step 5: Create Athena Tables Manually (20 min)

Sometimes you don't want to use a crawler — create tables directly with SQL:

```sql
-- Create a table for partitioned order data
CREATE EXTERNAL TABLE IF NOT EXISTS makanexpress_catalog.orders_partitioned (
    order_id INT,
    restaurant_id INT,
    customer_id INT,
    order_date STRING,
    amount DOUBLE,
    status STRING
)
PARTITIONED BY (city STRING)
STORED AS PARQUET
LOCATION 's3://makanexpress-datalake-dev/processed/orders_by_city/';

-- For CSV data
CREATE EXTERNAL TABLE IF NOT EXISTS makanexpress_catalog.restaurants_csv (
    restaurant_id INT,
    name STRING,
    city STRING,
    cuisine STRING,
    rating DOUBLE,
    monthly_orders INT,
    avg_price DOUBLE
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES ('field.delim' = ',')
STORED AS TEXTFILE
LOCATION 's3://makanexpress-datalake-dev/raw/restaurants/'
TBLPROPERTIES ('skip.header.line.count' = '1');

-- For JSON data
CREATE EXTERNAL TABLE IF NOT EXISTS makanexpress_catalog.orders_json (
    order_id INT,
    restaurant_id INT,
    customer_id INT,
    order_date STRING,
    amount DOUBLE,
    status STRING
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE
LOCATION 's3://makanexpress-datalake-dev/raw/orders/';

-- For Parquet (best format!)
CREATE EXTERNAL TABLE IF NOT EXISTS makanexpress_catalog.orders_parquet (
    order_id INT,
    restaurant_id INT,
    customer_id INT,
    order_date STRING,
    amount DOUBLE,
    status STRING
)
STORED AS PARQUET
LOCATION 's3://makanexpress-datalake-dev/processed/orders/';
```

---

## 🌤️ Afternoon Block (2 hours): Partitioning — The Key to Athena Performance

### Step 6: Why Partitioning Matters (20 min)

**The cost problem without partitioning:**

```
Your data: 100GB of orders in S3
Query: "SELECT * FROM orders WHERE city = 'Singapore'"

Without partitioning:
  Athena scans ALL 100GB → cost = 100GB × $5/TB = $0.50
  Time: ~30 seconds

With partitioning by city:
  Athena scans only Singapore partition (~20GB) → cost = 20GB × $5/TB = $0.10
  Time: ~5 seconds

Savings: 80% less data scanned, 6x faster
At scale (10TB): $50 vs $10 per query — that's real money!
```

**Partitioning = organizing data into folders by column value:**

```
Without partitioning:
  s3://bucket/orders/data.parquet
  (all orders in one file — Athena scans everything)

With partitioning by city:
  s3://bucket/orders/city=Singapore/data.parquet   (20GB)
  s3://bucket/orders/city=Penang/data.parquet       (15GB)
  s3://bucket/orders/city=Kuala Lumpur/data.parquet (25GB)
  s3://bucket/orders/city=Johor Bahru/data.parquet  (10GB)

  Query: WHERE city = 'Singapore'
  Athena reads ONLY the Singapore folder → 20GB instead of 70GB
```

### Step 7: Create Partitioned Data with PySpark (20 min)

```python
# file: create_partitioned_data.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import col

spark = SparkSession.builder \
    .appName("MakanExpress-Partition") \
    .master("local[*]") \
    .getOrCreate()

# Create sample orders
orders = spark.createDataFrame([
    (101, 1, 1, "2026-06-20", 25.50, "delivered", "Singapore"),
    (102, 2, 2, "2026-06-20", 12.00, "delivered", "Penang"),
    (103, 1, 3, "2026-06-21", 18.00, "delivered", "Singapore"),
    (104, 3, 1, "2026-06-21", 30.00, "delivered", "Kuala Lumpur"),
    (105, 2, 4, "2026-06-22", 15.00, "cancelled", "Penang"),
    (106, 1, 2, "2026-06-22", 22.00, "delivered", "Singapore"),
    (107, 4, 1, "2026-06-22", 45.00, "delivered", "Johor Bahru"),
    (108, 5, 3, "2026-06-23", 8.50, "delivered", "Singapore"),
    (109, 6, 1, "2026-06-23", 27.00, "delivered", "Singapore"),
    (110, 8, 4, "2026-06-23", 10.00, "delivered", "Kuala Lumpur"),
    (111, 7, 2, "2026-06-24", 14.00, "delivered", "Penang"),
    (112, 3, 3, "2026-06-24", 35.00, "delivered", "Kuala Lumpur"),
], ["order_id", "restaurant_id", "customer_id", "order_date", 
    "amount", "status", "city"])

# ========================================
# Write partitioned Parquet (to local first, then upload to S3)
# ========================================

# Partition by city
orders.write.partitionBy("city").mode("overwrite") \
    .parquet("output/orders_by_city/")

# Partition by city AND order_date (multi-level partitioning)
# orders.write.partitionBy("city", "order_date").mode("overwrite") \
#     .parquet("output/orders_by_date_city/")

# Check the partitioned structure
import os
for root, dirs, files in os.walk("output/orders_by_city/"):
    level = root.replace("output/orders_by_city/", "").count(os.sep)
    indent = " " * 2 * level
    print(f"{indent}{os.path.basename(root)}/")
    if level <= 1:
        subindent = " " * 2 * (level + 1)
        for f in files[:3]:
            print(f"{subindent}{f}")

# Upload to S3
import boto3
import glob

s3 = boto3.client('s3', region_name='ap-southeast-1')
BUCKET = 'makanexpress-datalake-dev'

for root, dirs, files in os.walk("output/orders_by_city/"):
    for f in files:
        if f.endswith('.parquet'):
            local_path = os.path.join(root, f)
            s3_key = f"processed/orders_by_city/{local_path.replace('output/orders_by_city/', '')}"
            s3.upload_file(local_path, BUCKET, s3_key)
            print(f"Uploaded: s3://{BUCKET}/{s3_key}")

spark.stop()
```

### Step 8: Register Partitioned Table + Load Partitions (20 min)

```sql
-- Create the partitioned table in Athena
CREATE EXTERNAL TABLE IF NOT EXISTS makanexpress_catalog.orders_partitioned (
    order_id INT,
    restaurant_id INT,
    customer_id INT,
    order_date STRING,
    amount DOUBLE,
    status STRING
)
PARTITIONED BY (city STRING)
STORED AS PARQUET
LOCATION 's3://makanexpress-datalake-dev/processed/orders_by_city/';

-- CRITICAL: Load partitions (Athena doesn't auto-discover partitions!)
MSCK REPAIR TABLE makanexpress_catalog.orders_partitioned;

-- Or load specific partitions manually:
ALTER TABLE makanexpress_catalog.orders_partitioned ADD IF NOT EXISTS
    PARTITION (city = 'Singapore') LOCATION 's3://makanexpress-datalake-dev/processed/orders_by_city/city=Singapore/'
    PARTITION (city = 'Penang') LOCATION 's3://makanexpress-datalake-dev/processed/orders_by_city/city=Penang/'
    PARTITION (city = 'Kuala Lumpur') LOCATION 's3://makanexpress-datalake-dev/processed/orders_by_city/city=Kuala Lumpur/'
    PARTITION (city = 'Johor Bahru') LOCATION 's3://makanexpress-datalake-dev/processed/orders_by_city/city=Johor Bahru/';

-- Now query with partition pruning (only scans matching partitions!)
SELECT * FROM makanexpress_catalog.orders_partitioned 
WHERE city = 'Singapore';
-- Athena scans ONLY the city=Singapore partition!

-- Show all partitions
SHOW PARTITIONS makanexpress_catalog.orders_partitioned;
```

### Step 9: Partitioning Strategies (20 min)

**Choosing the right partition column is CRITICAL:**

```
Good partition columns:
  ✅ city (4-20 distinct values, frequently filtered)
  ✅ order_date / year-month (time-based queries)
  ✅ status (delivered, cancelled, pending — low cardinality)
  ✅ country, region (geographic queries)

Bad partition columns:
  ❌ order_id (millions of values → millions of tiny files)
  ❌ customer_id (thousands of values → thousands of tiny files)
  ❌ amount (continuous values → infinite partitions)
  ❌ rating (too many distinct values)
```

**Common partitioning patterns:**

```python
# Pattern 1: Date-based partitioning (most common for logs/events)
# s3://bucket/orders/year=2026/month=06/day=20/
df.write.partitionBy("year", "month", "day").parquet("output/orders/")

# Pattern 2: Hive-style partitioning (column=value in path)
# s3://bucket/orders/city=Singapore/date=2026-06-20/
df.write.partitionBy("city", "order_date").parquet("output/orders/")

# Pattern 3: Single-level partitioning (simple)
# s3://bucket/orders/status=delivered/
df.write.partitionBy("status").parquet("output/orders/")

# ⚠️ Optimal partition size: 128MB - 1GB per file
# Too many small files → slow query startup (Athena lists files)
# Too few large files → slow reads, no parallelism
```

**Dynamic partition pruning (Athena does this automatically):**

```sql
-- Athena will only scan matching partitions
-- This is called "partition pruning" — it's automatic!

-- Only scans Singapore partition
SELECT * FROM orders_partitioned WHERE city = 'Singapore';

-- Only scans Singapore AND Penang partitions
SELECT * FROM orders_partitioned WHERE city IN ('Singapore', 'Penang');

-- Only scans June 2026 partitions (if partitioned by month)
SELECT * FROM orders_by_month WHERE year = 2026 AND month = 6;
```

### Exercise 2: Partitioning Challenge

You have 3 years of MakanExpress order data (2024-2026), totaling 500GB. Each day has ~450MB of orders. Queries are typically:

1. "Show me this month's orders" (80% of queries)
2. "Show me orders for Singapore" (15% of queries)
3. "Show me all cancelled orders" (5% of queries)

**Questions:**

**A.** What partitioning strategy would you use? Show the S3 path structure.

**B.** How much data would Athena scan for query #1 (this month, June 2026)?

**C.** How much data would Athena scan for query #2 (Singapore orders)?

**D.** What's the cost difference vs no partitioning?

<details>
<summary>🔑 Answer</summary>

**A: Partition by year/month (primary) since 80% of queries are time-based:**

```
s3://bucket/orders/year=2024/month=01/data.parquet  (~14GB, 31 days)
s3://bucket/orders/year=2024/month=02/data.parquet  (~13GB, 28 days)
...
s3://bucket/orders/year=2026/month=06/data.parquet  (~14GB, 30 days)
```

Why not partition by city? Because only 15% of queries filter by city, and time-based queries are the vast majority. Pick the partition that matches the most common query pattern.

**B: This month (June 2026):**
- 30 days × 450MB/day = ~13.5GB scanned
- Cost: 13.5GB × $5/TB = ~$0.07

**C: Singapore orders (without city partitioning):**
- ALL 500GB must be scanned (can't prune without city partition)
- Cost: 500GB × $5/TB = $2.50

If you also needed fast city queries, consider **dual partitioning:**
```
s3://bucket/orders/city=Singapore/year=2026/month=06/data.parquet
```
Then Singapore query scans: 3 years × ~100GB/year × 20% (SG proportion) = ~60GB → $0.30

**D: Cost comparison for "this month" query:**
- No partitioning: 500GB scanned → $2.50
- Year/month partitioning: 13.5GB → $0.07
- **Savings: 97%** ($2.50 → $0.07 per query)
- At 10 queries/day: $25/day vs $0.70/day = **$740/month savings**

</details>

### Step 10: Converting Formats with CTAS (Create Table As Select) (20 min)

Athena can transform data using CTAS — create a new table from a query result:

```sql
-- Convert CSV to Parquet (and partition!)
CREATE TABLE makanexpress_catalog.restaurants_parquet
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    external_location = 's3://makanexpress-datalake-dev/processed/restaurants_parquet/',
    partitioned_by = ARRAY['city']
) AS
SELECT 
    restaurant_id,
    name,
    cuisine,
    rating,
    monthly_orders,
    avg_price,
    city
FROM makanexpress_catalog.restaurants;

-- Now query the Parquet version (faster + cheaper!)
SELECT * FROM makanexpress_catalog.restaurants_parquet WHERE city = 'Singapore';

-- Aggregate and store results
CREATE TABLE makanexpress_catalog.city_summary
WITH (
    format = 'PARQUET',
    external_location = 's3://makanexpress-datalake-dev/analytics/city_summary/'
) AS
SELECT 
    city,
    COUNT(*) AS num_restaurants,
    ROUND(AVG(rating), 2) AS avg_rating,
    SUM(monthly_orders) AS total_orders,
    ROUND(AVG(avg_price), 2) AS avg_price
FROM makanexpress_catalog.restaurants
GROUP BY city;
```

### Step 11: Athena Cost Optimization Tips (15 min)

```sql
-- 1. Always limit columns (don't SELECT *)
-- ❌ Bad: scans ALL columns
SELECT * FROM orders_partitioned WHERE city = 'Singapore';

-- ✅ Good: scans only needed columns
SELECT order_id, amount FROM orders_partitioned WHERE city = 'Singapore';

-- 2. Use partition pruning
-- ✅ Partition column in WHERE clause
SELECT * FROM orders_partitioned WHERE city = 'Singapore' AND amount > 20;

-- 3. Estimate cost before running
-- Athena shows "Data scanned" after each query
-- Before running expensive queries, estimate:
--   Partitioned data: partition_size
--   Unpartitioned: full_table_size

-- 4. Use Parquet (columnar = only read needed columns)
-- CSV: reads entire row even if you only need 1 column
-- Parquet: reads only the columns in SELECT

-- 5. Compress data
-- Snappy compression (default in Parquet): fast, decent ratio
-- GZIP: better ratio, slower
-- For Athena: Snappy is recommended (balance of speed + size)

-- 6. Bucketing (advanced — for join optimization)
-- If you frequently join on restaurant_id:
CREATE EXTERNAL TABLE orders_bucketed (...)
PARTITIONED BY (city STRING)
CLUSTERED BY (restaurant_id) INTO 16 BUCKETS
STORED AS PARQUET
LOCATION 's3://...';
```

### Exercise 3: Athena Cost Estimation

You have:
- `orders` table: 200GB, stored as CSV, NOT partitioned
- `orders_partitioned`: Same data, Parquet format, partitioned by city (5 cities)
- Query: `SELECT order_id, amount FROM orders WHERE city = 'Singapore'`

**Calculate:**
1. Cost with CSV unpartitioned
2. Cost with Parquet partitioned (assume Singapore = 20% of data, column pruning saves 70% of Parquet size)

<details>
<summary>🔑 Answer</summary>

**1. CSV unpartitioned:**
- Athena scans ALL 200GB (no partition pruning, CSV reads entire rows)
- Cost: 200GB × $5/TB = **$1.00**

**2. Parquet partitioned:**
- Singapore partition: 200GB × 20% = 40GB
- Column pruning (only order_id, amount — 2 of 7 columns): 40GB × 30% = 12GB
- Cost: 12GB × $5/TB = **$0.06**

**Savings: $1.00 → $0.06 = 94% reduction per query**

At 50 queries/day: $50/day → $3/day = **$1,400/month savings**

This is why partitioning + Parquet is the #1 Athena optimization!

</details>

---

## 🌙 Evening Block (1 hour): Advanced Queries + Review

### Step 12: Useful Athena Queries for Data Engineering (20 min)

```sql
-- ========================================
-- Data profiling (understand your data)
-- ========================================

-- Check for NULLs
SELECT 
    COUNT(*) AS total_rows,
    COUNT(order_id) AS non_null_order_id,
    COUNT(amount) AS non_null_amount,
    COUNT(*) - COUNT(amount) AS null_amount_count
FROM orders;

-- Value distribution
SELECT 
    status,
    COUNT(*) AS count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct
FROM orders
GROUP BY status
ORDER BY count DESC;

-- Data freshness (latest date)
SELECT MAX(DATE_PARSE(order_date, '%Y-%m-%d')) AS latest_order_date
FROM orders;

-- ========================================
-- Unnesting arrays (for JSON data with arrays)
-- ========================================

-- If orders had an array of items:
-- {"order_id": 1, "items": [{"name": "Chicken Rice", "qty": 2}, ...]}
SELECT 
    order_id,
    item.name AS item_name,
    item.qty AS quantity
FROM orders
CROSS JOIN UNNEST(items) AS t(item);

-- ========================================
-- Date-based analytics
-- ========================================

-- Daily revenue trend
SELECT 
    order_date,
    COUNT(*) AS daily_orders,
    ROUND(SUM(CAST(amount AS DOUBLE)), 2) AS daily_revenue,
    ROUND(AVG(CAST(amount AS DOUBLE)), 2) AS avg_order_value
FROM orders
GROUP BY order_date
ORDER BY order_date;

-- Month-over-month comparison
WITH monthly AS (
    SELECT 
        DATE_TRUNC('month', DATE_PARSE(order_date, '%Y-%m-%d')) AS month,
        SUM(CAST(amount AS DOUBLE)) AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', DATE_PARSE(order_date, '%Y-%m-%d'))
)
SELECT 
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month,
    ROUND((revenue - LAG(revenue, 1) OVER (ORDER BY month)) / 
          LAG(revenue, 1) OVER (ORDER BY month) * 100, 2) AS mom_change_pct
FROM monthly
ORDER BY month;

-- ========================================
-- Finding data quality issues
-- ========================================

-- Duplicate check
SELECT order_id, COUNT(*) AS count
FROM orders
GROUP BY order_id
HAVING COUNT(*) > 1;

-- Outlier detection
SELECT *
FROM orders
WHERE CAST(amount AS DOUBLE) > (
    SELECT PERCENTILE(CAST(amount AS DOUBLE), 0.95) FROM orders
);
```

### Step 13: Athena with Python (boto3) (15 min)

```python
# file: athena_query.py
import boto3
import time

athena = boto3.client('athena', region_name='ap-southeast-1')

DATABASE = 'makanexpress_catalog'
OUTPUT = 's3://makanexpress-datalake-dev/athena-results/'
WORKGROUP = 'makanexpress'

def run_query(sql, database=DATABASE, workgroup=WORKGROUP):
    """Run an Athena query and wait for results."""
    
    # Start query
    response = athena.start_query_execution(
        QueryString=sql,
        QueryExecutionContext={'Database': database},
        ResultConfiguration={'OutputLocation': OUTPUT},
        WorkGroup=workgroup
    )
    
    query_id = response['QueryExecutionId']
    print(f"Query started: {query_id}")
    
    # Wait for completion
    while True:
        result = athena.get_query_execution(QueryExecutionId=query_id)
        state = result['QueryExecution']['Status']['State']
        
        if state == 'SUCCEEDED':
            data_scanned = result['QueryExecution']['Statistics'].get(
                'DataScannedInBytes', 0)
            print(f"✅ Query succeeded! Scanned: {data_scanned / 1024**3:.2f} GB")
            break
        elif state in ['FAILED', 'CANCELLED']:
            reason = result['QueryExecution']['Status'].get(
                'StateChangeReason', 'Unknown')
            print(f"❌ Query {state}: {reason}")
            return None
        else:
            print(f"⏳ State: {state}...")
            time.sleep(2)
    
    # Get results
    results = athena.get_query_results(QueryExecutionId=query_id)
    
    # Parse results into list of dicts
    columns = [col['Name'] for col in results['ResultSet']['ResultSetMetadata']['ColumnInfo']]
    rows = []
    for row in results['ResultSet']['Rows'][1:]:  # Skip header
        values = [col.get('VarCharValue', None) for col in row['Data']]
        rows.append(dict(zip(columns, values)))
    
    return rows

# Run queries
print("=== Top Restaurants by Revenue ===")
results = run_query("""
    SELECT 
        r.name,
        r.city,
        COUNT(o.order_id) AS order_count,
        ROUND(SUM(CAST(o.amount AS DOUBLE)), 2) AS total_revenue
    FROM restaurants r
    INNER JOIN orders o ON r.restaurant_id = o.restaurant_id
    GROUP BY r.name, r.city
    ORDER BY total_revenue DESC
    LIMIT 5
""")

if results:
    for row in results:
        print(f"  {row['name']:30s} | {row['city']:15s} | "
              f"{row['order_count']:3s} orders | ${row['total_revenue']}")

print("\n=== Data Scan Stats ===")
results = run_query("SELECT COUNT(*) as total FROM orders")
if results:
    print(f"  Total orders: {results[0]['total']}")
```

### Exercise 4: Build a Query Library

Write these Athena queries for your MakanExpress project:

**A.** Top 5 restaurants by monthly revenue (amount × frequency)

**B.** Daily order count trend for the past week

**C.** Customer retention: customers who ordered more than once

**D.** City comparison: revenue per city, orders per city, avg order value

Save all queries in a file: `athena/makanexpress_queries.sql`

<details>
<summary>🔑 Answer</summary>

```sql
-- athena/makanexpress_queries.sql

-- A: Top 5 restaurants by revenue
SELECT 
    r.name,
    r.city,
    COUNT(o.order_id) AS total_orders,
    ROUND(SUM(CAST(o.amount AS DOUBLE)), 2) AS total_revenue,
    ROUND(AVG(CAST(o.amount AS DOUBLE)), 2) AS avg_order_value
FROM restaurants r
INNER JOIN orders o ON r.restaurant_id = o.restaurant_id
WHERE o.status = 'delivered'
GROUP BY r.name, r.city
ORDER BY total_revenue DESC
LIMIT 5;

-- B: Daily order count trend
SELECT 
    order_date,
    COUNT(*) AS daily_orders,
    ROUND(SUM(CAST(amount AS DOUBLE)), 2) AS daily_revenue
FROM orders
GROUP BY order_date
ORDER BY order_date;

-- C: Customer retention (repeat customers)
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    ROUND(SUM(CAST(amount AS DOUBLE)), 2) AS total_spent
FROM orders
WHERE status = 'delivered'
GROUP BY customer_id
HAVING COUNT(*) > 1
ORDER BY order_count DESC;

-- D: City comparison
SELECT 
    r.city,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    ROUND(SUM(CAST(o.amount AS DOUBLE)), 2) AS total_revenue,
    ROUND(SUM(CAST(o.amount AS DOUBLE)) / COUNT(DISTINCT o.order_id), 2) AS avg_order_value
FROM orders o
INNER JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE o.status = 'delivered'
GROUP BY r.city
ORDER BY total_revenue DESC;
```

</details>

---

## 📚 Day 45 Summary

### What You Learned

| Topic | Key Takeaway |
|-------|-------------|
| Athena | Serverless SQL engine on S3 — no database needed |
| Glue Catalog dependency | Athena uses catalog for schema/location info |
| Athena SQL (Presto) | 95% same as PostgreSQL, some function differences |
| Partitioning | Organize S3 data by column → massive cost + speed savings |
| CTAS | Create new tables from query results (ETL with SQL!) |
| Cost optimization | Partition + Parquet = 90%+ cost reduction |
| Partition pruning | Athena auto-skips irrelevant partitions |
| boto3 Athena | Run queries programmatically from Python |

### Partitioning Cheat Sheet

| Strategy | When to Use | S3 Pattern |
|----------|-------------|------------|
| Date-based | Time-series data (80% of use cases) | `year=2026/month=06/day=20/` |
| Region/city | Geographic queries | `city=Singapore/` |
| Status | Small cardinality, frequent filter | `status=delivered/` |
| Multi-level | Multiple common filter patterns | `city=SG/year=2026/month=06/` |
| None | Small data (<1GB), ad-hoc queries | Single folder |

---

## ✅ Daily Checklist

- [ ] Athena workgroup created and configured
- [ ] First query run successfully (SELECT * FROM restaurants)
- [ ] Completed join queries (restaurants + orders)
- [ ] Completed Exercise 1 (Athena queries)
- [ ] Understand Athena vs PostgreSQL differences
- [ ] Created tables manually with SQL
- [ ] Created partitioned data with PySpark
- [ ] Registered partitioned table with MSCK REPAIR
- [ ] Understand partition pruning
- [ ] Completed Exercise 2 (partitioning strategy)
- [ ] Used CTAS to convert CSV → Parquet
- [ ] Completed Exercise 3 (cost estimation)
- [ ] Built query library (Exercise 4)
- [ ] Understand Athena cost optimization
- [ ] Ready for tomorrow: Glue + PySpark combined

---

## 🔖 Quick Reference Card

```
# ATHENA SQL BASICS
SELECT col1, col2 FROM table WHERE condition;

# CREATE EXTERNAL TABLE (Parquet)
CREATE EXTERNAL TABLE db.table (col1 INT, col2 STRING)
PARTITIONED BY (city STRING)
STORED AS PARQUET
LOCATION 's3://bucket/path/';

# LOAD PARTITIONS
MSCK REPAIR TABLE db.table;

# CTAS (convert format + partition)
CREATE TABLE db.new_table
WITH (format='PARQUET', partitioned_by=ARRAY['city'],
      external_location='s3://bucket/path/')
AS SELECT * FROM db.old_table WHERE ...;

# COST FORMULA
Cost = (Data scanned in GB / 1024) × $5

# PARTITIONING RULES
- Low cardinality columns (city, status, date)
- 128MB-1GB per partition file
- Match most common WHERE clauses
- Always use Parquet for column pruning

# BOTO3
athena.start_query_execution(QueryString=sql, ...)
athena.get_query_execution(QueryExecutionId=id)
athena.get_query_results(QueryExecutionId=id)
```

---

*Day 45 complete! Tomorrow: Glue + PySpark — write production ETL jobs.* 🔍🕷️🔥
