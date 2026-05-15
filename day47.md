# 📅 Day 47 — Monday, 30 June 2026
# 🔗 Serverless Pipeline: S3 → Glue → Athena

---

## 🎯 Today's Goal

This is the **"aha" moment** of modern data engineering. You're going to tie together everything from this week — S3 storage, Glue ETL, and Athena querying — into one complete **serverless data pipeline**.

No servers to manage. No infrastructure to patch. Just data flowing from raw files to queryable analytics.

**Why this matters for your career:** When an interviewer asks "how would you build a data pipeline on AWS?", this is the answer. Serverless pipelines are what companies in Singapore are actually running — Grab, Shopee, Foodpanda all use variants of this architecture.

---

## ☀️ Morning Block (2 hours): The Complete Serverless Architecture

### The Big Picture

Before we build, let's understand the full picture. Here's the pipeline we're building today:

```
                        SERVERLESS DATA PIPELINE
                        (MakanExpress Analytics)

┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  DATA SOURCE │     │  AWS S3      │     │  AWS GLUE    │     │  AWS ATHENA  │
│              │     │  Data Lake   │     │  ETL Engine  │     │  Query Engine│
│ • CSV files  │────▶│              │────▶│              │────▶│              │
│ • JSON logs  │     │ raw/         │     │ Crawler:     │     │ SQL queries  │
│ • API dumps  │     │ processed/   │     │  discover    │     │ on S3 data   │
│              │     │ analytics/   │     │ Job:         │     │              │
│              │     │              │     │  transform   │     │ Results in   │
│              │     │              │     │              │     │ S3 too!      │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
        │                   │                    │                    │
        │           ┌──────┴──────┐      ┌──────┴──────┐     ┌──────┴──────┐
        │           │ Versioning  │      │ PySpark     │     │ Partitioned │
        │           │ Encryption  │      │ Auto-scaling│     │ Columnar    │
        │           │ Lifecycle   │      │ Bookmarking │     │ Compressed  │
        │           └─────────────┘      └─────────────┘     └─────────────┘
        │                                                                │
        └──────────────── COST: Pay only for what you use ───────────────┘
                    S3: $0.023/GB/month
                    Glue: $0.44/DPU-hour
                    Athena: $5/TB scanned
```

### Why "Serverless" Is a Big Deal

**Traditional approach:**
1. Provision EC2 instances (servers)
2. Install Spark, configure it
3. Monitor server health
4. Patch OS, update software
5. Pay 24/7 whether you use it or not
6. Scale manually when data grows

**Serverless approach:**
1. Upload data to S3
2. Configure Glue to transform it
3. Query with Athena
4. Pay only when processing
5. Auto-scales with your data
6. Zero server management

**The trade-off:** You give up fine-grained control. For most data engineering workloads — especially at fresh grad / junior level — serverless is the right choice.

### Architecture Deep Dive: Each Component's Role

**S3 = The Foundation (Storage)**
- Your data lake lives here
- Organized in zones: raw → processed → analytics
- Parquet format for processed/analytics (compressed, fast reads)
- Partitioned by date for Athena performance

**Glue = The Engine (Transformation)**
- Crawlers discover your data and create table definitions
- Jobs transform data (PySpark under the hood)
- Can be scheduled or triggered
- Bookmarks remember what's already been processed

**Athena = The Interface (Querying)**
- SQL interface over S3 files
- No loading data into a database
- Results stored back in S3
- Perfect for ad-hoc analytics

### Data Flow: Step by Step

```
Step 1: Raw data lands in S3
   s3://makanexpress-datalake/raw/orders/2026-06-30/orders.csv

Step 2: Glue Crawler scans raw/
   → Discovers schema (columns, types)
   → Creates/updates table in Glue Data Catalog

Step 3: Glue Job transforms raw → processed
   → Reads from raw table
   → Cleans, validates, converts to Parquet
   → Writes partitioned output:
     s3://makanexpress-datalake/processed/orders/year=2026/month=06/day=30/

Step 4: Another Crawler scans processed/
   → Updates processed table schema

Step 5: Athena queries the processed table
   → Scans only relevant partitions
   → Returns results in seconds
   → Results saved to S3 output bucket
```

### Real-World: How Grab Uses This

Grab (Singapore's super-app) processes millions of food delivery orders daily. Their architecture:

1. **Orders stream** into S3 raw zone (every few minutes)
2. **Glue jobs** transform raw → analytics-ready (hourly)
3. **Athena + QuickSight** for dashboards and ad-hoc queries
4. **Cost:** Millions of rows processed for pennies

You're building a mini version of this today.

### Exercise 1: Architecture Quiz

Before we build, test your understanding:

**Q1.** Why do we use Parquet instead of CSV in the processed zone?

**Q2.** What happens if you don't partition your data in S3 before querying with Athena?

**Q3.** A Glue Crawler runs and discovers a new column in your CSV. What does it do?

**Q4.** You run an Athena query that scans 500 GB of data. How much does it cost?

**Q5.** Why do we have separate crawlers for raw/ and processed/ zones?

<details>
<summary>🔑 Answers</summary>

**A1:** Parquet is columnar (read only columns you need), compressed (50-80% smaller), and typed (no guessing data types). For analytics, this means faster queries and lower Athena costs (less data scanned).

**A2:** Athena scans ALL files in the table location every query. No partitioning = full table scan every time = slow + expensive. With partitioning by date, `WHERE date = '2026-06-30'` scans only that day's folder.

**A3:** It updates the table schema in the Glue Data Catalog to include the new column. Existing data will show NULL for the new column where it doesn't exist.

**A4:** 500 GB × $5/TB = 500/1024 × $5 = **$2.44** for one query. This is why partitioning and Parquet matter — they reduce the amount of data scanned.

**A5:** Raw and processed have different schemas and formats. Raw might be CSV with messy columns; processed is Parquet with clean types. Separate tables = separate query interfaces. You query raw for debugging, processed for analytics.

</details>

---

## 🌤️ Afternoon Block (2 hours): Build the Pipeline

Now let's actually build this. We'll go step by step — each step builds on the previous one.

### Prerequisites Check

Make sure you have:
- [ ] AWS CLI configured (`aws configure`)
- [ ] An S3 bucket for the data lake (from Day 37/42)
- [ ] IAM role for Glue (from Day 38/42)
- [ ] PySpark installed locally (`pip install pyspark`)

If you need to recreate anything quickly:

```bash
# Create bucket (if not existing)
aws s3 mb s3://makanexpress-datalake-dev --region ap-southeast-1

# Create folder structure
aws s3api put-object --bucket makanexpress-datalake-dev --key raw/
aws s3api put-object --bucket makanexpress-datalake-dev --key processed/
aws s3api put-object --bucket makanexpress-datalake-dev --key analytics/
aws s3api put-object --bucket makanexpress-datalake-dev --key athena-results/

# Create Glue database
aws glue create-database --database-input '{"Name": "makanexpress_db"}'
```

### Step 1: Create Sample Data

Let's create realistic MakanExpress order data:

```python
# create_sample_data.py
"""Generate sample MakanExpress order data for the serverless pipeline."""

import csv
import random
from datetime import datetime, timedelta
import os

# SG/MY restaurants
restaurants = [
    ("R001", "Hawker Chan", "Chinatown", "Chinese"),
    ("R002", "Jumbo Seafood", "Clarke Quay", "Seafood"),
    ("R003", "Song Fa Bak Kut Teh", "Clarke Quay", "Chinese"),
    ("R004", "Ya Kun Kaya Toast", "Marina Bay", "Breakfast"),
    ("R005", "Tiong Bahru Bakery", "Tiong Bahru", "Bakery"),
    ("R006", "PappaRich KL", "Bukit Bintang", "Malaysian"),
    ("R007", "Nasi Lemak Antarabangsa", "Bangsar", "Malaysian"),
    ("R008", "IPPUDO RAMEN", "Orchard", "Japanese"),
    ("R009", "Shake Shack Liat Towers", "Orchard", "Western"),
    ("R010", "wild honey", "Scotts Square", "Brunch"),
]

customers = [f"C{str(i).zfill(4)}" for i in range(1, 101)]
riders = [f"R{str(i).zfill(3)}" for i in range(1, 21)]

items = [
    ("I001", "Chicken Rice", 5.50),
    ("I002", "Laksa", 4.80),
    ("I003", "Char Kway Teow", 5.20),
    ("I004", "Nasi Lemak", 6.00),
    ("I005", "Bak Kut Teh", 7.50),
    ("I006", "Roti Prata", 2.50),
    ("I007", "Satay (10 sticks)", 8.00),
    ("I008", "Hokkien Mee", 5.00),
    ("I009", "Chili Crab", 35.00),
    ("I010", "Kaya Toast Set", 4.50),
    ("I011", "Teh Tarik", 2.00),
    ("I012", "Milo Dinosaur", 3.50),
]

def generate_orders(num_orders=500, date_str="2026-06-30"):
    """Generate realistic order data."""
    orders = []
    order_items = []
    
    base_date = datetime.strptime(date_str, "%Y-%m-%d")
    
    for order_id in range(1, num_orders + 1):
        order_time = base_date + timedelta(
            hours=random.randint(6, 23),
            minutes=random.randint(0, 59),
            seconds=random.randint(0, 59)
        )
        
        restaurant = random.choice(restaurants)
        customer = random.choice(customers)
        rider = random.choice(riders)
        
        # 1-5 items per order
        num_items = random.randint(1, 5)
        order_item_list = random.sample(items, num_items)
        
        total_amount = sum(item[2] for item in order_item_list)
        delivery_fee = round(random.uniform(2.00, 5.00), 2)
        
        status = random.choices(
            ["delivered", "delivered", "delivered", "delivered", 
             "cancelled", "refunded"],
            weights=[0.85, 0, 0, 0, 0.12, 0.03]
        )[0]
        
        orders.append({
            "order_id": f"ORD{str(order_id).zfill(6)}",
            "order_date": date_str,
            "order_time": order_time.strftime("%H:%M:%S"),
            "customer_id": customer,
            "restaurant_id": restaurant[0],
            "restaurant_name": restaurant[1],
            "rider_id": rider,
            "total_amount": f"{total_amount:.2f}",
            "delivery_fee": f"{delivery_fee:.2f}",
            "status": status,
            "payment_method": random.choice(["GrabPay", "Credit Card", "Cash", "ShopeePay"]),
        })
        
        for item in order_item_list:
            order_items.append({
                "order_id": f"ORD{str(order_id).zfill(6)}",
                "item_id": item[0],
                "item_name": item[1],
                "quantity": random.randint(1, 3),
                "unit_price": f"{item[2]:.2f}",
            })
    
    return orders, order_items

def write_csv(data, filepath):
    """Write list of dicts to CSV."""
    if not data:
        return
    os.makedirs(os.path.dirname(filepath), exist_ok=True)
    with open(filepath, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=data[0].keys())
        writer.writeheader()
        writer.writerows(data)

if __name__ == "__main__":
    # Generate for multiple dates (simulating daily data drops)
    dates = ["2026-06-28", "2026-06-29", "2026-06-30"]
    
    for date_str in dates:
        orders, order_items = generate_orders(num_orders=200, date_str=date_str)
        
        write_csv(orders, f"sample-data/raw/orders/{date_str}/orders.csv")
        write_csv(order_items, f"sample-data/raw/order_items/{date_str}/order_items.csv")
        
        print(f"✅ Generated {len(orders)} orders and {len(order_items)} items for {date_str}")
    
    print("\n📁 Sample data ready in sample-data/raw/")
    print("Next: Upload to S3 with the upload script below")
```

```bash
# Run the generator
python create_sample_data.py
```

### Step 2: Upload Raw Data to S3

```bash
# Upload the raw data to S3
aws s3 sync sample-data/raw/ s3://makanexpress-datalake-dev/raw/ \
    --region ap-southeast-1

# Verify
aws s3 ls s3://makanexpress-datalake-dev/raw/orders/ --recursive | head -20
```

### Step 3: Create Glue Crawlers

Now we create crawlers to discover the raw data:

```bash
# Create crawler for raw orders
aws glue create-crawler \
    --name makanexpress-raw-orders-crawler \
    --role "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/GlueServiceRole" \
    --database-name makanexpress_db \
    --targets '{"S3Targets": [{"Path": "s3://makanexpress-datalake-dev/raw/orders/"}]}' \
    --table-prefix "raw_" \
    --schedule "" \
    --description "Crawl raw MakanExpress order CSV data"

# Create crawler for raw order items
aws glue create-crawler \
    --name makanexpress-raw-items-crawler \
    --role "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/GlueServiceRole" \
    --database-name makanexpress_db \
    --targets '{"S3Targets": [{"Path": "s3://makanexpress-datalake-dev/raw/order_items/"}]}' \
    --table-prefix "raw_" \
    --schedule "" \
    --description "Crawl raw MakanExpress order items CSV data"

# Run the crawlers
aws glue start-crawler --name makanexpress-raw-orders-crawler
aws glue start-crawler --name makanexpress-raw-items-crawler

# Wait and check status
sleep 30
aws glue get-crawler --name makanexpress-raw-orders-crawler \
    --query 'Crawler.{State:State, LastRuntime:LastCrawlInfo.Status}'
```

After crawlers complete, verify tables were created:

```bash
# List tables in Glue database
aws glue get-tables --database-name makanexpress_db \
    --query 'TableList[].{Name:Name, Columns:TableType}' --output table
```

### Step 4: Test with Athena Before Transformation

Before running Glue ETL, let's query the raw data to see what we're working with:

```sql
-- In Athena console, select database: makanexpress_db

-- Check what the raw data looks like
SELECT * FROM raw_orders LIMIT 10;

-- Quick count by date
SELECT order_date, COUNT(*) as order_count
FROM raw_orders
GROUP BY order_date
ORDER BY order_date;

-- Check for data quality issues
SELECT 
    COUNT(*) as total_rows,
    COUNT(DISTINCT order_id) as unique_orders,
    SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) as cancelled,
    SUM(CASE WHEN total_amount IS NULL THEN 1 ELSE 0 END) as null_amount
FROM raw_orders;
```

**Notice:** Querying CSV from Athena is slow and expensive (full scan). That's why we transform to Parquet!

### Step 5: Create the Glue ETL Job

This is the core transformation — converting raw CSV to partitioned Parquet:

```python
# glue_etl_orders.py
"""
AWS Glue ETL Job: Transform raw MakanExpress orders from CSV to Parquet.
Deploy this as a Glue PySpark job.
"""
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.context import SparkContext
from pyspark.sql import functions as F
from pyspark.sql.types import *

# Glue boilerplate
args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# ─── READ: Load from Glue Data Catalog (raw table) ───
print("📖 Reading raw orders from Glue Catalog...")
orders_datasource = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_db",
    table_name="raw_orders",
    transformation_ctx="orders_datasource"
)

# Convert to Spark DataFrame for easier transformations
orders_df = orders_datasource.toDF()
print(f"   Loaded {orders_df.count()} orders")

# ─── TRANSFORM: Clean and enrich ───
print("🔧 Transforming data...")

# Cast types (CSV everything is string)
orders_transformed = orders_df \
    .withColumn("total_amount", F.col("total_amount").cast("double")) \
    .withColumn("delivery_fee", F.col("delivery_fee").cast("double")) \
    .withColumn("order_date", F.to_date(F.col("order_date"))) \
    .withColumn("order_year", F.year(F.col("order_date"))) \
    .withColumn("order_month", F.month(F.col("order_date"))) \
    .withColumn("order_day", F.dayofmonth(F.col("order_date")))

# Data quality: filter out invalid records
orders_clean = orders_transformed \
    .filter(F.col("order_id").isNotNull()) \
    .filter(F.col("total_amount") > 0) \
    .filter(F.col("status").isin("delivered", "cancelled", "refunded"))

# Add derived columns
orders_final = orders_clean \
    .withColumn("net_amount", 
        F.when(F.col("status") == "refunded", 0.0)
         .otherwise(F.col("total_amount") - F.col("delivery_fee"))
    )

print(f"   After cleaning: {orders_final.count()} valid orders")

# ─── WRITE: Partitioned Parquet to processed/ ───
print("💾 Writing partitioned Parquet to processed zone...")

# Convert back to DynamicFrame for Glue sink
orders_dynamic = DynamicFrame.fromDF(orders_final, glueContext, "orders_final")

# Write partitioned Parquet
glueContext.write_dynamic_frame.from_options(
    frame=orders_dynamic,
    connection_type="s3",
    connection_options={
        "path": "s3://makanexpress-datalake-dev/processed/orders/",
        "partitionKeys": ["order_year", "order_month", "order_day"]
    },
    format="glueparquet",
    format_options={"compression": "snappy"},
    transformation_ctx="orders_sink"
)

print("✅ ETL Job complete!")

job.commit()
```

**Deploy the job:**

```bash
# First, upload the script to S3
aws s3 cp glue_etl_orders.py \
    s3://makanexpress-datalake-dev/scripts/glue_etl_orders.py

# Create the Glue job
aws glue create-job \
    --name makanexpress-orders-etl \
    --role "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/GlueServiceRole" \
    --command '{"Name": "glueetl", "ScriptLocation": "s3://makanexpress-datalake-dev/scripts/glue_etl_orders.py"}' \
    --default-arguments '{
        "--TempDir": "s3://makanexpress-datalake-dev/temp/",
        "--job-language": "python"
    }' \
    --glue-version "4.0" \
    --number-of-workers 2 \
    --worker-type G.1X \
    --description "Transform MakanExpress orders CSV to partitioned Parquet"

# Run the job
aws glue start-job-run --job-name makanexpress-orders-etl

# Check status (wait a few minutes)
aws glue get-job-runs --job-name makanexpress-orders-etl --max-results 1 \
    --query 'JobRuns[0].{Status:JobRunState, Started:StartedOn}'
```

### Step 6: Create Processed Crawler + Query with Athena

```bash
# Crawler for processed data
aws glue create-crawler \
    --name makanexpress-processed-crawler \
    --role "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/GlueServiceRole" \
    --database-name makanexpress_db \
    --targets '{"S3Targets": [{"Path": "s3://makanexpress-datalake-dev/processed/orders/"}]}' \
    --table-prefix "processed_" \
    --description "Crawl processed Parquet order data"

aws glue start-crawler --name makanexpress-processed-crawler
```

After the crawler finishes, query in Athena:

```sql
-- Now query the PROCESSED (Parquet) table — notice the speed difference!
SELECT * FROM processed_orders LIMIT 10;

-- This query is MUCH cheaper and faster with partitioned Parquet
SELECT 
    order_date,
    COUNT(*) as total_orders,
    SUM(CASE WHEN status = 'delivered' THEN 1 ELSE 0 END) as delivered,
    SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) as cancelled,
    ROUND(SUM(net_amount), 2) as net_revenue
FROM processed_orders
WHERE order_year = 2026 AND order_month = 6
GROUP BY order_date
ORDER BY order_date;

-- Top restaurants by revenue
SELECT 
    restaurant_name,
    COUNT(*) as order_count,
    ROUND(SUM(net_amount), 2) as total_revenue,
    ROUND(AVG(net_amount), 2) as avg_order_value
FROM processed_orders
WHERE status = 'delivered'
GROUP BY restaurant_name
ORDER BY total_revenue DESC
LIMIT 10;

-- Payment method breakdown
SELECT 
    payment_method,
    COUNT(*) as count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 1) as percentage
FROM processed_orders
GROUP BY payment_method
ORDER BY count DESC;
```

### Exercise 2: Build the Order Items Pipeline

Now do the same for order items yourself:

**Task:** Create a Glue ETL job that:
1. Reads from `raw_order_items` table
2. Casts `quantity` to integer and `unit_price` to double
3. Adds a `line_total` column = quantity × unit_price
4. Writes partitioned Parquet to `processed/order_items/`
5. Create a crawler for the processed output
6. Write 3 Athena queries to analyze the data

<details>
<summary>🔑 Solution Script</summary>

```python
# glue_etl_items.py
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.context import SparkContext
from pyspark.sql import functions as F

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read
items_datasource = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_db",
    table_name="raw_order_items",
    transformation_ctx="items_datasource"
)
items_df = items_datasource.toDF()

# Transform
items_transformed = items_df \
    .withColumn("quantity", F.col("quantity").cast("int")) \
    .withColumn("unit_price", F.col("unit_price").cast("double")) \
    .withColumn("line_total", F.col("quantity") * F.col("unit_price"))

# Join with orders to get date for partitioning
# (Alternative: extract date from order_id pattern if consistent)
# For simplicity, we'll write without date partitioning for order_items
# In production, you'd join with orders to get the date

# Write
items_dynamic = DynamicFrame.fromDF(items_transformed, glueContext, "items_final")

glueContext.write_dynamic_frame.from_options(
    frame=items_dynamic,
    connection_type="s3",
    connection_options={
        "path": "s3://makanexpress-datalake-dev/processed/order_items/"
    },
    format="glueparquet",
    format_options={"compression": "snappy"},
    transformation_ctx="items_sink"
)

job.commit()
```

**Athena queries:**

```sql
-- Top selling items
SELECT 
    item_name,
    SUM(quantity) as total_quantity,
    ROUND(SUM(line_total), 2) as total_revenue,
    COUNT(DISTINCT order_id) as orders_containing_item
FROM processed_order_items
GROUP BY item_name
ORDER BY total_revenue DESC;

-- Average items per order
SELECT 
    AVG(item_count) as avg_items_per_order
FROM (
    SELECT order_id, COUNT(*) as item_count
    FROM processed_order_items
    GROUP BY order_id
);

-- Price distribution
SELECT 
    item_name,
    MIN(unit_price) as min_price,
    MAX(unit_price) as max_price,
    ROUND(AVG(unit_price), 2) as avg_price
FROM processed_order_items
GROUP BY item_name
ORDER BY avg_price DESC;
```

</details>

### Cost Optimization Tips

This is critical — serverless doesn't mean free. Here's how to keep costs low:

**1. Athena: Reduce Data Scanned**
```sql
-- ❌ BAD: Full table scan
SELECT * FROM processed_orders;

-- ✅ GOOD: Select only needed columns + partition filter
SELECT order_id, restaurant_name, net_amount 
FROM processed_orders 
WHERE order_year = 2026 AND order_month = 6 AND order_day = 30;
```

**2. Use Parquet + Snappy Compression**
- CSV 1GB → Parquet ~200MB (80% reduction)
- Athena charges by data scanned: 1GB CSV = $0.005, 200MB Parquet = $0.001

**3. Partition Strategically**
```
# Good partitioning:
processed/orders/year=2026/month=06/day=30/  ← daily data

# Over-partitioning (BAD):
processed/orders/year=2026/month=06/day=30/hour=14/minute=30/
# Too many small files → Athena metadata overhead
```

**4. Glue: Right-size Workers**
```bash
# For small jobs (< 1GB data): 2 workers, G.1X
# For medium jobs (1-10GB): 5-10 workers, G.1X  
# For large jobs (> 10GB): 10-20 workers, G.2X

# Don't over-provision! Glue charges per DPU-hour:
# G.1X = 1 DPU = $0.44/hour
# G.2X = 2 DPUs = $0.88/hour
```

**5. S3 Lifecycle Policies**
```python
# Move old data to cheaper storage
lifecycle_config = {
    "Rules": [{
        "ID": "DataLakeLifecycle",
        "Status": "Enabled",
        "Filter": {"Prefix": "raw/"},
        "Transitions": [
            {"Days": 30, "StorageClass": "STANDARD_IA"},  # Save 40%
            {"Days": 90, "StorageClass": "GLACIER"}         # Save 70%
        ],
        "Expiration": {"Days": 365}  # Delete after 1 year
    }]
}
```

### Exercise 3: Calculate Pipeline Cost

**Scenario:** MakanExpress processes:
- 10,000 orders/day (avg 500 bytes each in CSV)
- Glue ETL runs once daily for 15 minutes (2 workers, G.1X)
- Athena averages 50 queries/day, scanning 2GB each (Parquet)

**Calculate the monthly cost (30 days).**

<details>
<summary>🔑 Answer</summary>

```
S3 Storage:
- Raw CSV: 10,000 × 500B × 30 days = 150MB/month
- At $0.023/GB: 150MB × $0.023/1024 ≈ $0.003

Glue ETL:
- 15 min/day × 30 days = 7.5 hours
- 2 workers × G.1X (1 DPU each) = 2 DPUs
- 7.5 hours × 2 DPUs × $0.44/DPU-hour = $6.60

Athena:
- 50 queries × 2GB × 30 days = 3,000GB scanned
- 3,000GB × $5/1024GB = $14.65

TOTAL MONTHLY: ~$21.25

Compare: A small EC2 instance (t3.medium) running 24/7 = ~$30/month
And you'd still need to manage it!

Serverless wins for this scale.
```

</details>

---

## 🌙 Evening Block (1 hour): Monitoring, Error Handling & Quick Reference

### Monitoring Your Pipeline

**CloudWatch Metrics for Glue:**
```bash
# Check Glue job metrics
aws cloudwatch get-metric-statistics \
    --namespace AWS/Glue \
    --metric-name glue.driver.aggregate.numCompletedTasks \
    --dimensions Name=JobName,Value=makanexpress-orders-etl \
    --start-time $(date -u -d '1 day ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average
```

**Glue Job Bookmarking** (remember what's been processed):
```bash
# Enable bookmarks when creating the job
aws glue create-job \
    --name makanexpress-orders-etl-bookmarked \
    ... \
    --default-arguments '{
        "--job-bookmark-option": "job-bookmark-enable",
        "--TempDir": "s3://makanexpress-datalake-dev/temp/",
        "--job-language": "python"
    }'

# Reset bookmark if needed (reprocess everything)
aws glue reset-job-bookmark --job-name makanexpress-orders-etl-bookmarked
```

### Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `AccessDeniedException` | IAM role missing permissions | Add S3 read/write + Glue permissions to role |
| `AnalysisException: path not found` | S3 path doesn't exist | Check bucket name and prefix |
| Crawler finds 0 records | Wrong S3 path or format | Verify data exists: `aws s3 ls s3://... --recursive` |
| Athena timeout | Too much data scanned | Add partition filters, use column projection |
| Glue job OOM | Not enough workers | Increase worker count or use G.2X |
| `Schema mismatch` | CSV columns changed | Update crawler, check schema evolution settings |

### Error Handling in Glue Jobs

```python
# Add to your Glue ETL script for better error handling
import logging

# Set up logging
logger = glueContext.get_logger()

try:
    # Your ETL logic here
    orders_df = orders_datasource.toDF()
    
    # Validate data
    record_count = orders_df.count()
    if record_count == 0:
        logger.warn("⚠️ No records found in source data!")
        # Don't fail the job, just warn
    else:
        logger.info(f"✅ Processing {record_count} records")
    
    # Check for data quality
    null_orders = orders_df.filter(F.col("order_id").isNull()).count()
    if null_orders > 0:
        logger.warn(f"⚠️ Found {null_orders} records with null order_id")
    
except Exception as e:
    logger.error(f"❌ ETL Job failed: {str(e)}")
    raise  # Re-raise to mark job as failed
```

### Quick Reference Card: Serverless Pipeline

```
┌─────────────────────────────────────────────────────────┐
│           SERVERLESS DATA PIPELINE - CHEAT SHEET         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  S3 → GLUE CRAWLER → GLUE CATALOG → GLUE JOB → ATHENA │
│                                                         │
│  S3: Storage                                            │
│  ├── raw/        (CSV, JSON - source data)              │
│  ├── processed/  (Parquet, partitioned - clean)         │
│  └── analytics/  (Parquet, aggregated - business)       │
│                                                         │
│  Glue Crawler: Discover schema                          │
│  ├── Scans S3 path → creates/updates table              │
│  ├── Run on schedule or on-demand                       │
│  └── Can detect schema changes                          │
│                                                         │
│  Glue Job: Transform data                               │
│  ├── PySpark under the hood                             │
│  ├── Read from Catalog → transform → write to S3        │
│  ├── Bookmarks for incremental processing               │
│  └── $0.44/DPU-hour (G.1X)                             │
│                                                         │
│  Athena: Query S3 data                                  │
│  ├── Standard SQL over S3 files                         │
│  ├── $5 per TB scanned                                  │
│  ├── Use partitioning + Parquet to reduce cost          │
│  └── Results stored in S3                               │
│                                                         │
│  COST OPTIMIZATION:                                     │
│  ├── Parquet > CSV (80% smaller, columnar reads)        │
│  ├── Partition by date (avoid full scans)                │
│  ├── SELECT only needed columns                         │
│  ├── S3 lifecycle for old data                          │
│  └── Right-size Glue workers                            │
│                                                         │
│  KEY AWS CLI COMMANDS:                                  │
│  aws glue create-database                               │
│  aws glue create-crawler / start-crawler                │
│  aws glue create-job / start-job-run                    │
│  aws glue get-job-runs                                  │
│  aws s3 sync local/ s3://bucket/path/                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 📝 Today's Checklist

- [ ] Understood the full serverless pipeline architecture
- [ ] Created sample data and uploaded to S3
- [ ] Created and ran Glue crawlers for raw data
- [ ] Queried raw data in Athena
- [ ] Created and deployed Glue ETL job (CSV → Parquet)
- [ ] Created crawler for processed data
- [ ] Queried processed data in Athena (noticed the improvement!)
- [ ] Completed Exercise 2 (order items pipeline)
- [ ] Completed Exercise 3 (cost calculation)
- [ ] Read monitoring and error handling section
- [ ] Saved quick reference card

### 🔗 Useful Resources

- [AWS Glue Documentation](https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html)
- [Athena Documentation](https://docs.aws.amazon.com/athena/latest/ug/what-is.html)
- [Serverless Data Lake on AWS (Workshop)](https://serverless-data-lake-workshop.awssecworkshops.com/)
- [AWS Big Data Blog](https://aws.amazon.com/blogs/big-data/)

---

*Day 47 complete! Tomorrow: Assessment Day covering PySpark, Glue, Athena, and serverless pipelines.* 📝✅
