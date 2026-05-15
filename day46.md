# 📅 Day 46 — Sunday, 29 June 2026
# 🔗 AWS Glue + PySpark: Write Production ETL Jobs

---

## 🎯 Today's Goal

Yesterday you learned Athena (query S3 with SQL). Today you combine **Glue + PySpark** to write real production ETL jobs. This is the core skill: take raw data from S3, clean/transform it with PySpark in Glue, and write it back as optimized Parquet for Athena to query.

**By end of day you'll be able to:**
- Write complete Glue PySpark ETL jobs
- Handle common data engineering transformations
- Convert raw CSV/JSON → clean Parquet
- Set up proper error handling and logging

> **⏱️ Time budget:** Morning (2h) + Afternoon (2h) + Evening (1h) = 5 hours
> **💰 Cost:** Minimal — Glue Free Tier covers today's jobs

---

## 🧠 Today's Architecture

```
Raw Data (CSV/JSON)         Glue ETL Job (PySpark)        Processed Data (Parquet)
┌──────────────────┐       ┌──────────────────────┐      ┌──────────────────┐
│ s3://.../raw/    │──────→│ 1. Read from Catalog │─────→│ s3://.../        │
│   restaurants/   │       │ 2. Clean & Transform │      │   processed/     │
│   orders/        │       │ 3. Enrich & Join     │      │   restaurants/   │
│   customers/     │       │ 4. Write Parquet     │      │   orders/        │
└──────────────────┘       └──────────────────────┘      └──────────────────┘
                                    │                              │
                                    │                              ↓
                                    │                     ┌──────────────────┐
                                    └─── Glue Catalog ──→│ Athena Queries   │
                                         updates          │ (fast + cheap)   │
                                                         └──────────────────┘
```

---

## ☀️ Morning Block (2 hours): Complete ETL Job Template

### Step 1: Glue PySpark Job Template (20 min)

Every Glue PySpark job follows this structure:

```python
# file: glue_job_template.py
# AWS Glue PySpark ETL Job — Production Template

import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.sql import functions as F
from pyspark.sql.types import *
import logging

# ========================================
# 1. INITIALIZE (required boilerplate)
# ========================================
args = getResolvedOptions(sys.argv, [
    'JOB_NAME',
    # Add custom arguments here:
    # 'input_path',
    # 'output_path',
])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Logging (appears in CloudWatch)
logger = glueContext.get_logger()

# ========================================
# 2. READ DATA
# ========================================
# Option A: Read from Glue Catalog (recommended)
source_dyf = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_catalog",
    table_name="restaurants",
    transformation_ctx="source_read"
)

# Option B: Read directly from S3
# source_dyf = glueContext.create_dynamic_frame.from_options(
#     connection_type="s3",
#     connection_options={"paths": ["s3://bucket/raw/data/"]},
#     format="csv",
#     format_options={"withHeader": True, "separator": ","},
#     transformation_ctx="source_read"
# )

# Convert to DataFrame for transformations
df = source_dyf.toDF()
logger.info(f"Source record count: {df.count()}")

# ========================================
# 3. TRANSFORM DATA
# ========================================
# Your transformations go here
# Examples: filter, clean, join, aggregate, add columns
transformed_df = df  # placeholder

# ========================================
# 4. WRITE DATA
# ========================================
# Convert back to DynamicFrame for writing
output_dyf = DynamicFrame.fromDF(transformed_df, glueContext, "output")

glueContext.write_dynamic_frame.from_options(
    frame=output_dyf,
    connection_type="s3",
    connection_options={
        "path": "s3://makanexpress-datalake-dev/processed/output/"
    },
    format="glueparquet",
    format_options={"compression": "snappy"},
    transformation_ctx="output_write"
)

# ========================================
# 5. FINALIZE (required)
# ========================================
job.commit()
logger.info("Job completed successfully!")
```

### Step 2: Real ETL Job — Raw → Clean Parquet (30 min)

```python
# file: glue_job_clean_restaurants.py
# Cleans raw restaurant CSV data and writes partitioned Parquet

import sys
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.sql import functions as F
from pyspark.sql.types import DoubleType, IntegerType

args = getResolvedOptions(sys.argv, ['JOB_NAME'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)
logger = glueContext.get_logger()

# ========================================
# READ: Raw restaurant data from catalog
# ========================================
restaurants_dyf = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_catalog",
    table_name="restaurants",
    transformation_ctx="restaurants_read"
)

df = restaurants_dyf.toDF()
logger.info(f"Raw restaurants: {df.count()} records")
df.printSchema()

# ========================================
# TRANSFORM: Clean and enrich
# ========================================

# 1. Drop duplicates
df = df.dropDuplicates()

# 2. Handle NULLs
df = df.na.fill({
    "name": "Unknown Restaurant",
    "city": "Unknown",
    "cuisine": "Other",
    "rating": 0.0,
    "monthly_orders": 0,
    "avg_price": 0.0
})

# 3. Trim strings (remove leading/trailing whitespace)
df = df.withColumn("name", F.trim(F.col("name")))
df = df.withColumn("city", F.trim(F.col("city")))
df = df.withColumn("cuisine", F.trim(F.col("cuisine")))

# 4. Standardize city names (data quality!)
df = df.withColumn("city",
    F.when(F.col("city").isin("SG", "Sg", "singapore"), "Singapore")
     .when(F.col("city").isin("KL", "kl", "kuala lumpur"), "Kuala Lumpur")
     .when(F.col("city").isin("PG", "pg", "penang"), "Penang")
     .when(F.col("city").isin("JB", "jb", "johor"), "Johor Bahru")
     .otherwise(F.col("city"))
)

# 5. Add calculated columns
df = df.withColumn("estimated_revenue",
    F.round(F.col("monthly_orders") * F.col("avg_price"), 2)
)

df = df.withColumn("tier",
    F.when(F.col("monthly_orders") >= 150, "High Volume")
     .when(F.col("monthly_orders") >= 80, "Medium")
     .otherwise("Small")
)

# 6. Filter out invalid records
df = df.filter(
    (F.col("name") != "Unknown Restaurant") &
    (F.col("city") != "Unknown") &
    (F.col("rating") > 0) &
    (F.col("avg_price") > 0)
)

logger.info(f"Cleaned restaurants: {df.count()} records")

# ========================================
# WRITE: Partitioned Parquet to processed zone
# ========================================
output_dyf = DynamicFrame.fromDF(df, glueContext, "clean_restaurants")

# Write partitioned by city (for Athena performance)
# Note: DynamicFrame doesn't support partitionBy directly — use DataFrame
df.write.mode("overwrite") \
    .partitionBy("city") \
    .parquet("s3://makanexpress-datalake-dev/processed/restaurants/")

logger.info("Written to s3://makanexpress-datalake-dev/processed/restaurants/")

# ========================================
# UPDATE CATALOG: Create table for Athena
# ========================================
# Run a separate crawler or use spark.sql to create/update the table
# For now, we'll let the crawler handle it

job.commit()
```

### Step 3: Multi-Source ETL Job with Joins (30 min)

```python
# file: glue_job_order_analytics.py
# Joins orders + restaurants, creates analytics-ready output

import sys
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql import functions as F

args = getResolvedOptions(sys.argv, ['JOB_NAME'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)
logger = glueContext.get_logger()

# ========================================
# READ: Multiple sources
# ========================================

# Source 1: Orders from catalog
orders_dyf = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_catalog",
    table_name="orders",
    transformation_ctx="orders_read"
)
orders = orders_dyf.toDF()
logger.info(f"Orders loaded: {orders.count()}")

# Source 2: Restaurants from S3 directly (alternative method)
restaurants_dyf = glueContext.create_dynamic_frame.from_options(
    connection_type="s3",
    connection_options={
        "paths": ["s3://makanexpress-datalake-dev/processed/restaurants/"]
    },
    format="parquet",
    transformation_ctx="restaurants_read"
)
restaurants = restaurants_dyf.toDF()
logger.info(f"Restaurants loaded: {restaurants.count()}")

# Source 3: Customers from S3 (JSON)
customers_dyf = glueContext.create_dynamic_frame.from_options(
    connection_type="s3",
    connection_options={
        "paths": ["s3://makanexpress-datalake-dev/raw/customers/"]
    },
    format="json",
    transformation_ctx="customers_read"
)
customers = customers_dyf.toDF()
logger.info(f"Customers loaded: {customers.count()}")

# ========================================
# TRANSFORM: Join + Enrich
# ========================================

# Cast types (common when reading from CSV/JSON)
orders = orders.withColumn("amount", F.col("amount").cast("double"))
orders = orders.withColumn("restaurant_id", F.col("restaurant_id").cast("int"))
orders = orders.withColumn("customer_id", F.col("customer_id").cast("int"))

# Join orders + restaurants
enriched = orders.join(
    restaurants.select("restaurant_id", "name", "city", "cuisine"),
    on="restaurant_id",
    how="left"  # LEFT JOIN: keep all orders even if restaurant missing
)

# Join + customers
enriched = enriched.join(
    customers.select("customer_id", "customer_name", "customer_city"),
    on="customer_id",
    how="left"
)

logger.info(f"Joined records: {enriched.count()}")

# Add derived columns
enriched = enriched.withColumn(
    "delivery_fee",
    F.when(F.col("amount") >= 30, 0.0)      # Free delivery over $30
     .when(F.col("amount") >= 15, 2.0)       # $2 delivery over $15
     .otherwise(3.0)                          # $3 delivery otherwise
)

enriched = enriched.withColumn(
    "total_with_fee",
    F.round(F.col("amount") + F.col("delivery_fee"), 2)
)

enriched = enriched.withColumn(
    "order_hour",
    F.hour(F.col("order_date"))  # Extract hour for analysis
)

# Filter only valid records
enriched = enriched.filter(F.col("name").isNotNull())

# ========================================
# AGGREGATE: Create analytics tables
# ========================================

# Table 1: Restaurant performance
restaurant_perf = enriched \
    .filter(F.col("status") == "delivered") \
    .groupBy("restaurant_id", "name", "city", "cuisine") \
    .agg(
        F.count("*").alias("total_orders"),
        F.round(F.sum("amount"), 2).alias("total_revenue"),
        F.round(F.avg("amount"), 2).alias("avg_order_value"),
        F.countDistinct("customer_id").alias("unique_customers"),
        F.round(F.sum("delivery_fee"), 2).alias("total_delivery_fees")
    ) \
    .orderBy(F.col("total_revenue").desc())

# Table 2: Daily summary
daily_summary = enriched \
    .filter(F.col("status") == "delivered") \
    .groupBy("order_date", "city") \
    .agg(
        F.count("*").alias("daily_orders"),
        F.round(F.sum("amount"), 2).alias("daily_revenue"),
        F.round(F.avg("amount"), 2).alias("avg_order_value"),
        F.countDistinct("restaurant_id").alias("active_restaurants")
    )

# ========================================
# WRITE: Multiple outputs
# ========================================

# Write enriched orders (partitioned by city)
enriched.write.mode("overwrite") \
    .partitionBy("city") \
    .parquet("s3://makanexpress-datalake-dev/processed/enriched_orders/")

# Write restaurant performance (single Parquet)
restaurant_perf.write.mode("overwrite") \
    .parquet("s3://makanexpress-datalake-dev/analytics/restaurant_performance/")

# Write daily summary (partitioned by city)
daily_summary.write.mode("overwrite") \
    .partitionBy("city") \
    .parquet("s3://makanexpress-datalake-dev/analytics/daily_summary/")

logger.info("All outputs written successfully!")

# Print stats for CloudWatch logs
logger.info(f"Enriched records: {enriched.count()}")
logger.info(f"Restaurant performance rows: {restaurant_perf.count()}")
logger.info(f"Daily summary rows: {daily_summary.count()}")

job.commit()
```

### Exercise 1: Add Customer Lifetime Value

Modify the `glue_job_order_analytics.py` to add a **Customer Lifetime Value (CLV)** analytics table:

**Requirements:**
1. Group by customer_id and customer_name
2. Calculate: total orders, total spent, first order date, last order date, days active
3. Label customers as "VIP" (total_spent >= $100), "Regular" ($50-99), "New" (< $50)
4. Write as Parquet to `s3://.../analytics/customer_clv/`

<details>
<summary>🔑 Answer</summary>

Add this before the WRITE section:

```python
# Table 3: Customer Lifetime Value
from pyspark.sql.functions import min as spark_min, max as spark_max, datediff

clv = enriched \
    .filter(F.col("status") == "delivered") \
    .groupBy("customer_id", "customer_name") \
    .agg(
        F.count("*").alias("total_orders"),
        F.round(F.sum("amount"), 2).alias("total_spent"),
        F.round(F.avg("amount"), 2).alias("avg_order_value"),
        F.min("order_date").alias("first_order_date"),
        F.max("order_date").alias("last_order_date"),
        F.countDistinct("restaurant_id").alias("restaurants_tried")
    ) \
    .withColumn("days_active",
        F.datediff(F.col("last_order_date"), F.col("first_order_date"))
    ) \
    .withColumn("customer_segment",
        F.when(F.col("total_spent") >= 100, "VIP")
         .when(F.col("total_spent") >= 50, "Regular")
         .otherwise("New")
    ) \
    .orderBy(F.col("total_spent").desc())

# Write
clv.write.mode("overwrite") \
    .parquet("s3://makanexpress-datalake-dev/analytics/customer_clv/")
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Error Handling, Testing & Best Practices

### Step 4: Error Handling in Glue Jobs (30 min)

```python
# file: glue_job_with_error_handling.py
# Production-ready error handling patterns

import sys
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql import functions as F
import traceback

args = getResolvedOptions(sys.argv, ['JOB_NAME'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)
logger = glueContext.get_logger()

# ========================================
# Pattern 1: Try/except around sections
# ========================================
try:
    # READ
    source_dyf = glueContext.create_dynamic_frame.from_catalog(
        database="makanexpress_catalog",
        table_name="restaurants",
        transformation_ctx="source"
    )
    df = source_dyf.toDF()
    
    if df.count() == 0:
        logger.warn("⚠️ Source table is empty! Nothing to process.")
        job.commit()
        sys.exit(0)
    
    logger.info(f"Read {df.count()} records")
    
    # TRANSFORM
    cleaned = df.filter(F.col("name").isNotNull())
    
    # VALIDATE: Check data quality before writing
    total = df.count()
    after_clean = cleaned.count()
    dropped = total - after_clean
    drop_rate = (dropped / total * 100) if total > 0 else 0
    
    logger.info(f"Records: {total} → {after_clean} (dropped {dropped}, {drop_rate:.1f}%)")
    
    if drop_rate > 50:
        raise ValueError(
            f"🚨 Data quality alert: {drop_rate:.1f}% records dropped! "
            f"Threshold is 50%. Stopping job to prevent data loss."
        )
    
    # WRITE
    cleaned.write.mode("overwrite").parquet(
        "s3://makanexpress-datalake-dev/processed/restaurants/"
    )
    
    logger.info("✅ Job completed successfully")
    
except ValueError as e:
    logger.error(f"Data quality error: {e}")
    raise  # Re-raise to mark job as FAILED
    
except Exception as e:
    logger.error(f"❌ Job failed: {e}")
    logger.error(traceback.format_exc())
    raise  # Re-raise to mark job as FAILED
    
finally:
    job.commit()

# ========================================
# Pattern 2: Dead letter queue (save bad records)
# ========================================
# Instead of dropping bad records, save them for investigation

df = source_dyf.toDF()

# Separate good and bad records
good = df.filter(
    F.col("name").isNotNull() &
    F.col("city").isNotNull() &
    (F.col("rating") > 0) &
    (F.col("avg_price") > 0)
)

bad = df.filter(
    F.col("name").isNull() |
    F.col("city").isNull() |
    (F.col("rating") <= 0) |
    (F.col("avg_price") <= 0)
)

logger.info(f"Good records: {good.count()}, Bad records: {bad.count()}")

# Write good records to processed zone
good.write.mode("overwrite").parquet(
    "s3://makanexpress-datalake-dev/processed/restaurants/"
)

# Write bad records to dead letter zone (for investigation)
if bad.count() > 0:
    bad.write.mode("overwrite").parquet(
        "s3://makanexpress-datalake-dev/quarantine/restaurants/"
    )
    logger.warn(f"⚠️ {bad.count()} bad records quarantined")
```

### Step 5: Passing Parameters to Glue Jobs (20 min)

```python
# file: glue_job_parameterized.py
# Accept parameters when running the job

import sys
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

# Accept custom arguments
args = getResolvedOptions(sys.argv, [
    'JOB_NAME',
    '--input_database',     # Which Glue database
    '--input_table',        # Which table to read
    '--output_path',        # Where to write
    '--min_rating',         # Filter threshold
    '--date_filter',        # Date to process
])

# Use the parameters
INPUT_DATABASE = args.get('input_database', 'makanexpress_catalog')
INPUT_TABLE = args.get('input_table', 'restaurants')
OUTPUT_PATH = args.get('output_path', 's3://makanexpress-datalake-dev/processed/output/')
MIN_RATING = float(args.get('min_rating', '4.0'))
DATE_FILTER = args.get('date_filter', '')

print(f"Reading from: {INPUT_DATABASE}.{INPUT_TABLE}")
print(f"Writing to: {OUTPUT_PATH}")
print(f"Minimum rating: {MIN_RATING}")
print(f"Date filter: {DATE_FILTER}")
```

**Run with parameters:**

```bash
aws glue start-job-run \
    --job-name makanexpress-parameterized-etl \
    --arguments '{
        "--input_database": "makanexpress_catalog",
        "--input_table": "restaurants",
        "--output_path": "s3://makanexpress-datalake-dev/processed/top_restaurants/",
        "--min_rating": "4.5"
    }' \
    --region ap-southeast-1
```

### Step 6: Testing Glue Jobs Locally (20 min)

Before uploading to Glue, test your PySpark logic locally:

```python
# file: test_etl_locally.py
# Run this on your laptop — tests the PySpark logic without Glue

from pyspark.sql import SparkSession
from pyspark.sql import functions as F
import pytest

# Setup
spark = SparkSession.builder \
    .appName("Test-MakanExpress-ETL") \
    .master("local[*]") \
    .getOrCreate()

# ========================================
# Create test data (mimics raw data)
# ========================================
restaurants = spark.createDataFrame([
    (1, "Changi Chicken Rice", "Singapore", "Chinese", 4.5, 120, 8.50),
    (2, "Penang Famous Laksa", "Penang", "Malay", 4.8, 200, 6.00),
    (3, None, "Kuala Lumpur", "Chinese", 4.3, 95, 7.50),  # NULL name
    (4, "JB Bak Kut Teh", None, "Chinese", 4.6, 150, 12.00),  # NULL city
    (5, "Bad Data", "Singapore", "Chinese", -1.0, 50, 0.0),  # Invalid rating/price
], ["restaurant_id", "name", "city", "cuisine", "rating", "monthly_orders", "avg_price"])

orders = spark.createDataFrame([
    (101, 1, 1, "2026-06-20", 25.50, "delivered"),
    (102, 2, 2, "2026-06-20", 12.00, "delivered"),
    (103, 1, 3, "2026-06-21", 18.00, "cancelled"),
    (104, 99, 1, "2026-06-21", 30.00, "delivered"),  # restaurant_id=99 doesn't exist
], ["order_id", "restaurant_id", "customer_id", "order_date", "amount", "status"])

# ========================================
# Test 1: Cleaning removes NULL names
# ========================================
def test_clean_removes_null_names():
    cleaned = restaurants.filter(F.col("name").isNotNull())
    null_count = cleaned.filter(F.col("name").isNull()).count()
    assert null_count == 0, f"Found {null_count} NULL names after cleaning"
    assert cleaned.count() == 4  # 5 - 1 NULL name = 4
    print("✅ Test 1 passed: NULL names removed")

test_clean_removes_null_names()

# ========================================
# Test 2: Validation removes bad ratings
# ========================================
def test_validation_removes_negative_ratings():
    cleaned = restaurants.filter(F.col("rating") > 0)
    assert cleaned.count() == 4  # 5 - 1 negative = 4
    print("✅ Test 2 passed: Negative ratings removed")

test_validation_removes_negative_ratings()

# ========================================
# Test 3: Join keeps all orders (LEFT JOIN)
# ========================================
def test_left_join_preserves_orders():
    joined = orders.join(restaurants, "restaurant_id", "left")
    assert joined.count() == orders.count(), "LEFT JOIN should preserve all orders"
    print("✅ Test 3 passed: LEFT JOIN preserves all orders")

test_left_join_preserves_orders()

# ========================================
# Test 4: Revenue calculation is correct
# ========================================
def test_revenue_calculation():
    df = restaurants.withColumn("revenue", F.col("monthly_orders") * F.col("avg_price"))
    row1 = df.filter(F.col("restaurant_id") == 1).first()
    expected_revenue = 120 * 8.50  # 1020.00
    assert abs(row1["revenue"] - expected_revenue) < 0.01
    print(f"✅ Test 4 passed: Revenue = {row1['revenue']} (expected {expected_revenue})")

test_revenue_calculation()

# ========================================
# Test 5: Dead letter queue catches bad records
# ========================================
def test_dead_letter_queue():
    good = restaurants.filter(
        F.col("name").isNotNull() &
        F.col("city").isNotNull() &
        (F.col("rating") > 0) &
        (F.col("avg_price") > 0)
    )
    bad = restaurants.filter(
        F.col("name").isNull() |
        F.col("city").isNull() |
        (F.col("rating") <= 0) |
        (F.col("avg_price") <= 0)
    )
    assert good.count() == 3, f"Expected 3 good, got {good.count()}"
    assert bad.count() == 2, f"Expected 2 bad, got {bad.count()}"
    print(f"✅ Test 5 passed: {good.count()} good, {bad.count()} quarantined")

test_dead_letter_queue()

print("\n🎉 All tests passed!")

spark.stop()
```

```bash
python test_etl_locally.py
```

### Exercise 2: Write Tests for Order Analytics

Write 3 local tests for the `glue_job_order_analytics.py` logic:

1. Test that cancelled orders are excluded from restaurant performance
2. Test that delivery fee calculation is correct ($0 for orders >= $30, $2 for >= $15, $3 otherwise)
3. Test that customers with no orders still appear in CLV (or don't — justify your choice)

<details>
<summary>🔑 Answer</summary>

```python
# Test 1: Cancelled orders excluded from performance
def test_cancelled_excluded():
    delivered = orders.filter(F.col("status") == "delivered")
    perf = delivered.groupBy("restaurant_id").count()
    # Restaurant 1 has orders 101 (delivered), 103 (cancelled)
    r1_count = perf.filter(F.col("restaurant_id") == 1).first()["count"]
    assert r1_count == 1, "Cancelled orders should be excluded"
    print("✅ Test 1: Cancelled orders excluded")

test_cancelled_excluded()

# Test 2: Delivery fee calculation
def test_delivery_fee():
    test_orders = spark.createDataFrame([
        (1, 35.00),   # >= 30 → $0
        (2, 20.00),   # >= 15 → $2
        (3, 10.00),   # < 15 → $3
        (4, 30.00),   # >= 30 → $0 (boundary)
        (5, 15.00),   # >= 15 → $2 (boundary)
    ], ["id", "amount"])
    
    with_fee = test_orders.withColumn("delivery_fee",
        F.when(F.col("amount") >= 30, 0.0)
         .when(F.col("amount") >= 15, 2.0)
         .otherwise(3.0)
    )
    
    results = {row["id"]: row["delivery_fee"] for row in with_fee.collect()}
    assert results[1] == 0.0, f"Expected 0.0, got {results[1]}"
    assert results[2] == 2.0, f"Expected 2.0, got {results[2]}"
    assert results[3] == 3.0, f"Expected 3.0, got {results[3]}"
    assert results[4] == 0.0, f"Expected 0.0, got {results[4]}"
    assert results[5] == 2.0, f"Expected 2.0, got {results[5]}"
    print("✅ Test 2: Delivery fee calculation correct")

test_delivery_fee()

# Test 3: CLV only includes customers WITH orders
# Justification: CLV (Customer Lifetime VALUE) is meaningless for 
# customers with no orders — they have no value yet.
# A separate "potential customer" table could track signups without orders.
def test_clv_only_ordered_customers():
    # CLV should only contain customers who have placed orders
    enriched = orders.filter(F.col("status") == "delivered")
    customer_ids = [row["customer_id"] for row in enriched.select("customer_id").distinct().collect()]
    
    # Customer 4 has NO delivered orders (only 105 which is cancelled)
    assert 4 not in customer_ids, "Customer 4 has no delivered orders, should not be in CLV"
    print("✅ Test 3: CLV only includes customers with delivered orders")

test_clv_only_ordered_customers()
```

</details>

---

## 🌙 Evening Block (1 hour): Glue Best Practices + Review

### Step 7: Production Best Practices (20 min)

```
╔══════════════════════════════════════════════════════════════╗
║            GLUE ETL JOB — PRODUCTION CHECKLIST              ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  DATA QUALITY                                                ║
║  □ Validate input count > 0 before processing               ║
║  □ Track drop rate (alert if > 50%)                          ║
║  □ Quarantine bad records (dead letter queue)                ║
║  □ Log record counts at each step                            ║
║                                                              ║
║  ERROR HANDLING                                              ║
║  □ Try/except around each section (read, transform, write)   ║
║  □ Raise on data quality violations                          ║
║  □ Log full stack traces                                     ║
║  □ Set job timeout (prevent runaway jobs)                    ║
║                                                              ║
║  PERFORMANCE                                                 ║
║  □ Use Parquet format (never output CSV)                     ║
║  □ Partition output by most common filter column             ║
║  □ Enable job bookmarks for incremental processing           ║
║  □ Right-size workers (start with 2, increase if slow)       ║
║  □ Use push_down_predicate to skip partitions on read        ║
║                                                              ║
║  MONITORING                                                  ║
║  □ Add logging at key steps (logger.info/warn/error)         ║
║  □ Check CloudWatch logs after each run                      ║
║  □ Set up Glue job metrics dashboard                         ║
║  □ Alert on job failures (CloudWatch alarm → SNS)            ║
║                                                              ║
║  SECURITY                                                    ║
║  □ Use IAM roles (never hardcode credentials)                ║
║  □ Least privilege (only needed S3 paths)                    ║
║  □ Enable S3 bucket encryption                               ║
║  □ Don't log sensitive data (PII, passwords)                  ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

### Step 8: Glue Job Cost Estimation (15 min)

```python
# Cost calculator for Glue jobs

def estimate_glue_cost(workers, worker_type, run_minutes, runs_per_month):
    """Estimate monthly Glue job cost."""
    
    dpu_map = {
        "G.1X": 1,    # 1 DPU, 16GB memory
        "G.2X": 2,    # 2 DPUs, 32GB memory
        "G.025X": 0.25  # Python Shell only
    }
    
    dpus = dpu_map.get(worker_type, 1) * workers
    hours_per_run = run_minutes / 60
    dpu_hours = dpus * hours_per_run
    cost_per_run = dpu_hours * 0.44  # $0.44/DPU-hour
    
    monthly_cost = cost_per_run * runs_per_month
    
    print(f"Worker type: {worker_type} × {workers} = {dpus} DPUs")
    print(f"Run time: {run_minutes} minutes ({hours_per_run:.2f} hours)")
    print(f"DPU-hours per run: {dpu_hours:.2f}")
    print(f"Cost per run: ${cost_per_run:.2f}")
    print(f"Runs per month: {runs_per_month}")
    print(f"Monthly cost: ${monthly_cost:.2f}")
    print()
    return monthly_cost

# Example: MakanExpress daily ETL
print("=== MakanExpress Daily Restaurant ETL ===")
estimate_glue_cost(
    workers=2,
    worker_type="G.1X",
    run_minutes=10,
    runs_per_month=30
)

# Example: Large order processing
print("=== Large Order Processing (weekly) ===")
estimate_glue_cost(
    workers=5,
    worker_type="G.2X",
    run_minutes=45,
    runs_per_month=4
)

# Example: Python Shell script (lightweight)
print("=== Python Shell - File check ===")
# Python shell: $0.44/DPU-hour, min 0.0625 DPU
print(f"Worker: Python Shell (0.0625 DPU)")
print(f"Cost per run: ${0.0625 * (2/60) * 0.44:.4f}")
print(f"Monthly (daily): ${0.0625 * (2/60) * 0.44 * 30:.2f}")
```

### Exercise 3: Architecture Decision

Your boss at MakanExpress asks: "We have 10GB of restaurant data updated daily. Should we use Glue or Athena for these tasks?"

For each task, recommend Glue, Athena, or both, and explain why:

**A.** Daily import of 500MB CSV → clean → write as Parquet (remove duplicates, fix city names)

**B.** Ad-hoc query: "What's the average order value in Singapore this week?"

**C.** Monthly report: Join 3 tables, aggregate, calculate metrics, write to S3

**D.** Real-time fraud detection (flag orders over $500)

<details>
<summary>🔑 Answer</summary>

**A: Glue** — This is a transformation task (CSV → clean → Parquet). Athena can't transform data (read-only). Glue is the right tool for ETL.

**B: Athena** — This is a simple query on existing data. No transformation needed. Athena handles this perfectly and costs pennies (scans maybe 1-2GB = ~$0.01).

**C: Both** — Use Glue for the ETL (join, aggregate, calculate) and write results to S3. Then Athena can query the pre-computed results for the report. Or use Athena CTAS for simpler cases.

**D: Neither** — Glue and Athena are batch processing tools. Real-time fraud detection needs streaming (Kinesis, Lambda, Flink). This is out of scope for this week, but important to know the limitation.

</details>

---

## 📚 Day 46 Summary

### What You Learned

| Topic | Key Takeaway |
|-------|-------------|
| Glue Job Template | Boilerplate: init → read → transform → write → commit |
| Multi-source ETL | Read from catalog + S3, join, enrich, write multiple outputs |
| Error Handling | Try/except, dead letter queue, data quality validation |
| Parameters | Pass arguments to Glue jobs for reusability |
| Local Testing | Test PySpark logic locally before uploading to Glue |
| Best Practices | Data quality, error handling, performance, monitoring, security |
| Cost Estimation | DPU × hours × $0.44, calculate monthly cost |

### Glue Job Template Cheat Sheet

```
# 1. INIT
args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# 2. READ (from catalog)
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="db", table_name="tbl")
df = dyf.toDF()

# 3. TRANSFORM (PySpark DataFrame API)
df = df.filter(...).withColumn(...).groupBy(...).agg(...)

# 4. WRITE
df.write.mode("overwrite").partitionBy("col").parquet("s3://...")

# 5. COMMIT
job.commit()
```

---

## ✅ Daily Checklist

- [ ] Glue job template understood and saved
- [ ] `glue_job_clean_restaurants.py` written and tested locally
- [ ] `glue_job_order_analytics.py` written with multi-source joins
- [ ] Exercise 1 completed (Customer CLV)
- [ ] Error handling patterns understood (try/except, dead letter queue)
- [ ] Parameter passing tested
- [ ] Local tests written (`test_etl_locally.py`)
- [ ] Exercise 2 completed (order analytics tests)
- [ ] Production best practices reviewed
- [ ] Cost estimation calculator run
- [ ] Exercise 3 completed (architecture decisions)
- [ ] Ready for tomorrow: Full serverless pipeline

---

## 🔖 Quick Reference Card

```
# GLUE PYSPARK BOILERPLATE
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame

# READ from catalog
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="db", table_name="tbl")
df = dyf.toDF()

# READ from S3
dyf = glueContext.create_dynamic_frame.from_options(
    connection_type="s3",
    connection_options={"paths": ["s3://bucket/path/"]},
    format="parquet")

# WRITE (DataFrame — supports partitionBy)
df.write.mode("overwrite").partitionBy("col").parquet("s3://out/")

# WRITE (DynamicFrame)
DynamicFrame.fromDF(df, glueContext, "name")
glueContext.write_dynamic_frame.from_options(
    frame=dyf, connection_type="s3",
    connection_options={"path": "s3://out/"},
    format="glueparquet")

# ERROR HANDLING
try:
    # ETL logic
    if drop_rate > 50:
        raise ValueError("Data quality alert")
except Exception as e:
    logger.error(f"Failed: {e}")
    raise
finally:
    job.commit()

# PARAMETERS
args = getResolvedOptions(sys.argv, ['JOB_NAME', '--my_param'])
my_param = args['my_param']
# Run: --arguments '{"--my_param": "value"}'
```

---

*Day 46 complete! Tomorrow: Tie it all together — serverless pipeline from S3 → Glue → Athena → dashboard.* 🔗🚀
