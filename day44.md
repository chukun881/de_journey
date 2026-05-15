# 📅 Day 44 — Friday, 27 June 2026
# 🕷️ AWS Glue: Crawlers, Jobs & Visual ETL

---

## 🎯 Today's Goal

AWS Glue is a **serverless ETL service** — it runs Spark for you without managing any servers. You upload your PySpark script (or use the visual editor), and Glue handles the cluster, scaling, and execution. Today you'll learn how Glue works, create crawlers to discover data, and build your first ETL job.

**Why Glue matters:**
- Most SG/MY data engineering job postings mention "ETL on cloud"
- Glue is the AWS-native way to do Spark ETL — no servers to manage
- Serverless = pay only when running (~$0.44/DPU-hour on Free Tier)
- It's the backbone of modern AWS data pipelines

> **⏱️ Time budget:** Morning (2h) + Afternoon (2h) + Evening (1h) = 5 hours
> **💰 Cost:** Free Tier includes 1M objects crawled/month + Glue Studio jobs

---

## 🧠 Mental Model: What AWS Glue Actually Is

```
Traditional Spark (what you did yesterday):
┌─────────────────────────────────────────┐
│ Your Laptop                             │
│ ┌──────────┐   ┌───────────┐           │
│ │ Python    │──→│ Spark JVM │──→ Output │
│ │ (driver)  │   │ (local)   │           │
│ └──────────┘   └───────────┘           │
│ You manage everything!                  │
└─────────────────────────────────────────┘

AWS Glue (what you'll learn today):
┌─────────────────────────────────────────┐
│ Your Laptop                             │
│ ┌──────────┐                            │
│ │ Browser   │──→ Upload script / config │
│ │ (console) │                           │
│ └──────────┘                            │
└──────────────────┬──────────────────────┘
                   │ API call
┌──────────────────▼──────────────────────┐
│ AWS Glue (serverless)                    │
│ ┌──────────┐  ┌──────┐  ┌──────────┐  │
│ │ Crawler   │  │ Catalog │  │ Spark    │  │
│ │ discovers │  │ stores  │  │ executes │  │
│ │ schema    │  │ metadata│  │ your ETL │  │
│ └──────────┘  └──────┘  └──────────┘  │
│ AWS manages all infrastructure!         │
└─────────────────────────────────────────┘
```

**Three components of Glue:**

| Component | What it does | Analogy |
|-----------|-------------|---------|
| **Glue Catalog** | Central metadata repository (schemas, locations, formats) | Library card catalog |
| **Glue Crawler** | Scans S3/database, discovers schema, updates catalog | Librarian organizing books |
| **Glue Job** | Runs your PySpark ETL script on serverless Spark | The actual reading/processing |

---

## ☀️ Morning Block (2 hours): Glue Catalog + Crawlers

### Step 1: Set Up the Glue Catalog Database (15 min)

```bash
# Create a Glue database (metadata container — like a database name in SQL)
aws glue create-database \
    --database-input '{"Name": "makanexpress_catalog"}' \
    --region ap-southeast-1

# Verify
aws glue get-database --name makanexpress_catalog --region ap-southeast-1
```

**What is the Glue Catalog?**
- It's a **metadata store** — it knows *about* your data but doesn't store the data itself
- Stores: table names, column names, data types, S3 locations, formats, partition info
- Used by: Athena (for querying), Glue jobs (for reading/writing), Redshift Spectrum
- Think of it as a schema registry — "orders table has these columns, lives in S3 at this path, in Parquet format"

### Step 2: Create Test Data on S3 (15 min)

First, let's create sample data for Glue to crawl:

```python
# file: create_glue_test_data.py
import boto3
import json
import csv
from io import StringIO

s3 = boto3.client('s3', region_name='ap-southeast-1')

BUCKET = 'makanexpress-datalake-dev'  # Use your bucket from Day 42

# ========================================
# 1. Create sample restaurant data (CSV)
# ========================================
csv_data = """restaurant_id,name,city,cuisine,rating,monthly_orders,avg_price
1,Changi Chicken Rice,Singapore,Chinese,4.5,120,8.50
2,Penang Famous Laksa,Penang,Malay,4.8,200,6.00
3,KL Hokkien Mee King,Kuala Lumpur,Chinese,4.3,95,7.50
4,JB Bak Kut Teh,Johor Bahru,Chinese,4.6,150,12.00
5,Tiong Bahru Curry Rice,Singapore,Chinese,4.1,80,5.50
6,Little India Banana Leaf,Singapore,Indian,4.7,180,9.00
7,Gurney Drive Hokkien Mee,Penang,Chinese,4.4,110,7.00
8,Bangsar Mamak Corner,Kuala Lumpur,Malay,4.2,220,5.00
"""

s3.put_object(
    Bucket=BUCKET,
    Key='raw/restaurants/restaurants.csv',
    Body=csv_data.encode('utf-8')
)

# ========================================
# 2. Create sample orders data (JSON)
# ========================================
orders = [
    {"order_id": 101, "restaurant_id": 1, "customer_id": 1, "order_date": "2026-06-20", "amount": 25.50, "status": "delivered"},
    {"order_id": 102, "restaurant_id": 2, "customer_id": 2, "order_date": "2026-06-20", "amount": 12.00, "status": "delivered"},
    {"order_id": 103, "restaurant_id": 1, "customer_id": 3, "order_date": "2026-06-21", "amount": 18.00, "status": "delivered"},
    {"order_id": 104, "restaurant_id": 3, "customer_id": 1, "order_date": "2026-06-21", "amount": 30.00, "status": "delivered"},
    {"order_id": 105, "restaurant_id": 2, "customer_id": 4, "order_date": "2026-06-22", "amount": 15.00, "status": "cancelled"},
    {"order_id": 106, "restaurant_id": 1, "customer_id": 2, "order_date": "2026-06-22", "amount": 22.00, "status": "delivered"},
    {"order_id": 107, "restaurant_id": 4, "customer_id": 1, "order_date": "2026-06-22", "amount": 45.00, "status": "delivered"},
    {"order_id": 108, "restaurant_id": 5, "customer_id": 3, "order_date": "2026-06-23", "amount": 8.50, "status": "delivered"},
]

# Write each order as a JSON line
jsonl_data = '\n'.join(json.dumps(o) for o in orders)

s3.put_object(
    Bucket=BUCKET,
    Key='raw/orders/orders.json',
    Body=jsonl_data.encode('utf-8')
)

# ========================================
# 3. Create partitioned data (by city)
# ========================================
for city in ['Singapore', 'Penang', 'Kuala Lumpur']:
    city_orders = [o for o in orders if o.get('city', '') == city]
    if city_orders:
        s3.put_object(
            Bucket=BUCKET,
            Key=f'raw/orders_partitioned/city={city}/data.json',
            Body='\n'.join(json.dumps(o) for o in city_orders).encode('utf-8')
        )

print("✅ Test data uploaded to S3!")
print(f"  s3://{BUCKET}/raw/restaurants/restaurants.csv")
print(f"  s3://{BUCKET}/raw/orders/orders.json")
```

**Run it:**
```bash
python create_glue_test_data.py
```

### Step 3: Create an IAM Role for Glue (10 min)

Glue needs a role to access S3 and the catalog:

```bash
# Create trust policy file
cat > /tmp/glue-trust-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "glue.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}
EOF

# Create the role
aws iam create-role \
    --role-name MakanExpressGlueRole \
    --assume-role-policy-document file:///tmp/glue-trust-policy.json

# Attach managed policy (Glue service + S3 read/write)
aws iam attach-role-policy \
    --role-name MakanExpressGlueRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

# Create custom S3 policy
cat > /tmp/glue-s3-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
        "Resource": [
            "arn:aws:s3:::makanexpress-datalake-dev",
            "arn:aws:s3:::makanexpress-datalake-dev/*"
        ]
    }]
}
EOF

aws iam put-role-policy \
    --role-name MakanExpressGlueRole \
    --policy-name MakanExpressS3Access \
    --policy-document file:///tmp/glue-s3-policy.json

echo "✅ Role created: MakanExpressGlueRole"
echo "Role ARN: $(aws iam get-role --role-name MakanExpressGlueRole --query 'Role.Arn' --output text)"
```

### Step 4: Create and Run a Glue Crawler (30 min)

**Console Method (recommended for first time):**

1. Go to AWS Glue Console → **Crawlers** → **Create crawler**
2. Name: `makanexpress-restaurant-crawler`
3. Data source: `S3` → Browse → `s3://makanexpress-datalake-dev/raw/restaurants/`
4. IAM role: `MakanExpressGlueRole`
5. Target database: `makanexpress_catalog`
6. Crawler source type: **Data stores**
7. Frequency: **On demand** (we'll run it manually)
8. Click **Create**

**CLI Method:**

```bash
# Create the crawler
aws glue create-crawler \
    --name makanexpress-restaurant-crawler \
    --role "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/MakanExpressGlueRole" \
    --database-name makanexpress_catalog \
    --targets '{"S3Targets":[{"Path":"s3://makanexpress-datalake-dev/raw/restaurants/"}]}' \
    --description "Crawl restaurant CSV data" \
    --region ap-southeast-1

# Run the crawler
aws glue start-crawler \
    --name makanexpress-restaurant-crawler \
    --region ap-southeast-1

# Check status
aws glue get-crawler \
    --name makanexpress-restaurant-crawler \
    --query 'Crawler.State' \
    --region ap-southeast-1
# Wait until it says "READY" (takes 1-3 minutes)

# Check what tables were created
aws glue get-tables \
    --database-name makanexpress_catalog \
    --query 'TableList[].{Name:Name,Columns:StorageDescriptor.Columns}' \
    --region ap-southeast-1
```

**What the crawler did:**

```
1. Scanned s3://makanexpress-datalake-dev/raw/restaurants/
2. Found: restaurants.csv
3. Inferred schema: restaurant_id (bigint), name (string), city (string), ...
4. Created a table "restaurants" in makanexpress_catalog
5. Recorded: format=CSV, location=s3://..., columns=[...]
```

Now Athena (tomorrow) can query this data using `SELECT * FROM restaurants` without knowing it's a CSV on S3!

### Exercise 1: Create a Second Crawler

Create a crawler for the orders JSON data:

**Requirements:**
- Name: `makanexpress-orders-crawler`
- Source: `s3://makanexpress-datalake-dev/raw/orders/`
- Same role and database
- Run it and verify the `orders` table appears in the catalog

After both crawlers run, list all tables in `makanexpress_catalog`.

<details>
<summary>🔑 Answer</summary>

```bash
# Create orders crawler
aws glue create-crawler \
    --name makanexpress-orders-crawler \
    --role "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/MakanExpressGlueRole" \
    --database-name makanexpress_catalog \
    --targets '{"S3Targets":[{"Path":"s3://makanexpress-datalake-dev/raw/orders/"}]}' \
    --region ap-southeast-1

# Run it
aws glue start-crawler --name makanexpress-orders-crawler --region ap-southeast-1

# Wait, then check tables
aws glue get-tables \
    --database-name makanexpress_catalog \
    --query 'TableList[].Name' \
    --region ap-southeast-1
# Should show: ["orders", "restaurants"]
```

</details>

### Step 5: Understanding the Glue Catalog Schema (20 min)

```bash
# Inspect the restaurants table schema
aws glue get-table \
    --database-name makanexpress_catalog \
    --name restaurants \
    --region ap-southeast-1
```

**Key fields in the response:**

```json
{
    "Table": {
        "Name": "restaurants",
        "DatabaseName": "makanexpress_catalog",
        "StorageDescriptor": {
            "Columns": [
                {"Name": "restaurant_id", "Type": "bigint"},
                {"Name": "name", "Type": "string"},
                {"Name": "city", "Type": "string"},
                {"Name": "cuisine", "Type": "string"},
                {"Name": "rating", "Type": "double"},
                {"Name": "monthly_orders", "Type": "bigint"},
                {"Name": "avg_price", "Type": "double"}
            ],
            "Location": "s3://makanexpress-datalake-dev/raw/restaurants/",
            "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
            "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
            "SerdeInfo": {
                "SerializationLibrary": "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
                "Parameters": {"field.delim": ","}
            }
        },
        "TableType": "EXTERNAL_TABLE"
    }
}
```

**What this tells us:**
- **Columns + Types:** The schema (auto-detected by crawler)
- **Location:** Where the data lives on S3
- **InputFormat/OutputFormat:** How to read/write (text = CSV)
- **SerDe:** Serializer/Deserializer — how to parse each row (comma-delimited)
- **TableType:** EXTERNAL_TABLE = data lives outside Glue (on S3), Glue just knows about it

**Manual table creation (when you know the schema):**

```bash
# Sometimes you know the schema and don't need a crawler
aws glue create-table \
    --database-name makanexpress_catalog \
    --table-input '{
        "Name": "restaurants_manual",
        "StorageDescriptor": {
            "Columns": [
                {"Name": "restaurant_id", "Type": "int"},
                {"Name": "name", "Type": "string"},
                {"Name": "city", "Type": "string"},
                {"Name": "rating", "Type": "double"}
            ],
            "Location": "s3://makanexpress-datalake-dev/raw/restaurants/",
            "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
            "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
            "SerdeInfo": {
                "SerializationLibrary": "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
                "Parameters": {"field.delim": ","}
            }
        },
        "TableType": "EXTERNAL_TABLE"
    }' \
    --region ap-southeast-1
```

---

## 🌤️ Afternoon Block (2 hours): Glue Jobs + Visual ETL

### Step 6: Understanding Glue Job Types (15 min)

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Glue Job Types                        │
├──────────────────┬──────────────────┬───────────────────────┤
│  Spark ETL       │  Python Shell    │  Ray                  │
│  (PySpark)       │  (plain Python)  │  (new, advanced)      │
├──────────────────┼──────────────────┼───────────────────────┤
│ Full Spark power │ Just Python      │ Distributed Python    │
│ ~1 min startup   │ ~5 sec startup   │ ~30 sec startup       │
│ 2-100 DPUs       │ 0.0625-1 DPU     │ Custom                │
│ Use for:         │ Use for:         │ Use for:              │
│ - Big data       │ - Small files    │ - ML workloads        │
│ - Transformations│ - API calls      │ - Python-native       │
│ - Joins/agg      │ - Light ETL      │   scaling             │
│ This week → ✅   │ Week 8+         │ Awareness only        │
└──────────────────┴──────────────────┴───────────────────────┘
```

**DPU = Data Processing Unit**
- 1 DPU = 4 vCPU + 16 GB memory
- Minimum for Spark: 2 DPUs (1 driver + 1 executor)
- Cost: $0.44/DPU-hour (Free Tier: 10 DPU-hours/day for 30 days)
- A 2-DPU job running 10 minutes = 2 × 0.166 hours × $0.44 = **$0.15**

### Step 7: Your First Glue Job — Visual ETL (30 min)

**Glue Studio Visual Editor** (no code):

1. Go to **AWS Glue Studio** → **ETL jobs** → **Visual ETL**
2. Click **Create**
3. You'll see a visual canvas with nodes

**Build this pipeline:**

```
[S3 Source] → [Transform] → [S3 Destination]
 restaurants     Filter         Parquet output
   CSV           rating > 4.0
```

**Steps:**

1. **Source node:**
   - Type: `S3` source
   - S3 path: `s3://makanexpress-datalake-dev/raw/restaurants/`
   - Data format: `CSV`
   - Check "Has header"

2. **Transform node:**
   - Click "+" to add transform
   - Type: `Filter`
   - Condition: `rating >= 4.5`
   - (This filters out low-rated restaurants)

3. **Destination node:**
   - Type: `S3` destination
   - S3 path: `s3://makanexpress-datalake-dev/processed/top_restaurants/`
   - Data format: `Parquet`
   - Compression: `Snappy`

4. **Job details:**
   - Name: `makanexpress-filter-top-restaurants`
   - IAM role: `MakanExpressGlueRole`
   - DPU: 2 (minimum for Spark)
   - Requested number of workers: 2
   - Job timeout: 10 minutes

5. Click **Save** → **Run**

**Monitor the job:**
- Go to **Glue Studio** → **Runs** tab
- Watch status: STARTING → RUNNING → SUCCEEDED (or FAILED)
- Click the run to see logs, metrics, and execution time

**Check the output:**
```bash
aws s3 ls s3://makanexpress-datalake-dev/processed/top_restaurants/ --recursive
```

You should see Parquet files written by Glue!

### Step 8: Glue Job — PySpark Script (Script Mode) (40 min)

The visual editor generates PySpark code. Let's write it ourselves:

```python
# file: glue_job_filter_restaurants.py
# This runs INSIDE AWS Glue (not on your laptop)
# Glue provides the Spark session automatically

import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql.functions import col, round as spark_round

# ========================================
# Glue boilerplate (required for every Glue job)
# ========================================
args = getResolvedOptions(sys.argv, ['JOB_NAME'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# ========================================
# Method 1: Read from Glue Catalog (recommended)
# ========================================
# This uses the catalog table we created with the crawler
restaurants_dyf = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_catalog",
    table_name="restaurants",
    transformation_ctx="restaurants"
)

# DynamicFrame vs DataFrame:
# - DynamicFrame: Glue's own format, handles schema flexibility
# - DataFrame: Standard Spark DataFrame (from yesterday)
# - You can convert between them!

# Convert DynamicFrame → DataFrame (for more operations)
restaurants_df = restaurants_dyf.toDF()
print(f"Total restaurants: {restaurants_df.count()}")
restaurants_df.printSchema()

# ========================================
# Transform: Filter top-rated + add calculated column
# ========================================
top_restaurants = restaurants_df \
    .filter(col("rating") >= 4.5) \
    .withColumn(
        "estimated_revenue",
        spark_round(col("monthly_orders") * col("avg_price"), 2)
    )

print(f"Top restaurants (rating >= 4.5): {top_restaurants.count()}")
top_restaurants.show()

# ========================================
# Method 2: Read directly from S3 (without catalog)
# ========================================
# Useful when you don't have a crawler set up
orders_dyf = glueContext.create_dynamic_frame.from_options(
    connection_type="s3",
    connection_options={"paths": ["s3://makanexpress-datalake-dev/raw/orders/"]},
    format="json",
    transformation_ctx="orders"
)

orders_df = orders_dyf.toDF()
print(f"Total orders: {orders_df.count()}")

# ========================================
# Join: Top restaurants + their orders
# ========================================
joined = top_restaurants.join(orders_df, "restaurant_id", "inner")
print(f"Joined records: {joined.count()}")
joined.show()

# ========================================
# Write output to S3 as Parquet
# ========================================
# Convert DataFrame → DynamicFrame for writing
from awsglue.dynamicframe import DynamicFrame
joined_dyf = DynamicFrame.fromDF(joined, glueContext, "joined")

glueContext.write_dynamic_frame.from_options(
    frame=joined_dyf,
    connection_type="s3",
    connection_options={
        "path": "s3://makanexpress-datalake-dev/processed/top_restaurant_orders/"
    },
    format="glueparquet",
    transformation_ctx="write_output"
)

print("✅ Job complete! Output written to processed/top_restaurant_orders/")

# ========================================
# Glue boilerplate (end of job)
# ========================================
job.commit()
```

**Upload and run this script:**

```bash
# Upload the script to S3 (Glue reads scripts from S3)
aws s3 cp glue_job_filter_restaurants.py \
    s3://makanexpress-datalake-dev/scripts/glue_job_filter_restaurants.py

# Create the Glue job
aws glue create-job \
    --name makanexpress-filter-restaurants \
    --role "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/MakanExpressGlueRole" \
    --command '{"Name":"glueetl","ScriptLocation":"s3://makanexpress-datalake-dev/scripts/glue_job_filter_restaurants.py","PythonVersion":"3"} \
    --default-arguments '{
        "--TempDir": "s3://makanexpress-datalake-dev/temp/glue/",
        "--job-language": "python"
    }' \
    --glue-version "4.0" \
    --number-of-workers 2 \
    --worker-type "G.1X" \
    --region ap-southeast-1

# Run the job
aws glue start-job-run \
    --job-name makanexpress-filter-restaurants \
    --region ap-southeast-1

# Check status (returns RunId)
aws glue get-job-runs \
    --job-name makanexpress-filter-restaurants \
    --max-results 1 \
    --query 'JobRuns[0].{Status:JobRunState,Started:StartedOn}' \
    --region ap-southeast-1
```

### Exercise 2: Modify the Glue Job

Modify `glue_job_filter_restaurants.py` to:

1. Read restaurants from the catalog AND orders from S3 (already done above)
2. Filter to only **delivered** orders
3. Group by restaurant to calculate: total revenue, order count, average order value
4. Write the aggregated result as Parquet to `s3://makanexpress-datalake-dev/analytics/restaurant_performance/`

<details>
<summary>🔑 Answer</summary>

```python
# Add after the join section:

from pyspark.sql.functions import count, sum as spark_sum, avg as spark_avg, col, round as spark_round

# Filter delivered orders only
delivered = joined.filter(col("status") == "delivered")

# Aggregate per restaurant
performance = delivered.groupBy("restaurant_id", "name", "city").agg(
    count("*").alias("total_orders"),
    spark_round(sum("amount"), 2).alias("total_revenue"),
    spark_round(avg("amount"), 2).alias("avg_order_value")
).orderBy(col("total_revenue").desc())

performance.show()

# Write to analytics zone
performance_dyf = DynamicFrame.fromDF(performance, glueContext, "performance")

glueContext.write_dynamic_frame.from_options(
    frame=performance_dyf,
    connection_type="s3",
    connection_options={
        "path": "s3://makanexpress-datalake-dev/analytics/restaurant_performance/"
    },
    format="glueparquet",
    transformation_ctx="write_analytics"
)
```

</details>

### Step 9: DynamicFrame vs DataFrame (15 min)

```python
# DynamicFrame = Glue's enhanced DataFrame
# DataFrame = Standard Spark DataFrame

# When to use DynamicFrame:
# ✅ Reading from Glue Catalog (required)
# ✅ Handling messy/evolving schemas
# ✅ Using Glue's built-in transforms (ApplyMapping, Relationalize)
# ✅ Writing with Glue's sink options

# When to use DataFrame:
# ✅ Complex transformations (join, groupBy, window functions)
# ✅ SQL operations
# ✅ Performance-critical code
# ✅ You're comfortable with Pandas-like API

# Convert: DynamicFrame → DataFrame
df = dynamic_frame.toDF()

# Convert: DataFrame → DynamicFrame
from awsglue.dynamicframe import DynamicFrame
dyf = DynamicFrame.fromDF(df, glueContext, "name")

# Glue built-in transforms (DynamicFrame only):
# ApplyMapping - rename and cast columns
mapped = ApplyMapping.apply(
    frame=restaurants_dyf,
    mappings=[
        ("restaurant_id", "long", "id", "int"),     # cast long → int
        ("name", "string", "restaurant_name", "string"),  # rename
        ("city", "string", "city", "string"),        # keep as-is
    ]
)

# DropFields - remove columns
cleaned = DropFields.apply(
    frame=restaurants_dyf,
    paths=["monthly_orders", "avg_price"]  # remove these columns
)

# Filter (simpler than DataFrame filter)
from awsglue.transforms import Filter
top_rated = Filter.apply(
    frame=restaurants_dyf,
    f=lambda x: x["rating"] >= 4.5
)

# ResolveChoice - handle schema ambiguities
resolved = ResolveChoice.apply(
    frame=restaurants_dyf,
    specs=[("rating", "cast:double")]
)
```

### Step 10: Glue Job Monitoring & Troubleshooting (20 min)

```bash
# View job run history
aws glue get-job-runs \
    --job-name makanexpress-filter-restaurants \
    --region ap-southeast-1

# Get specific run details
aws glue get-job-run \
    --job-name makanexpress-filter-restaurants \
    --run-id <RUN_ID> \
    --region ap-southeast-1

# View CloudWatch logs (where Glue sends logs)
aws logs get-log-events \
    --log-group-name "/aws-glue/jobs/logs-v2/summary/makanexpress-filter-restaurants" \
    --log-stream-name <STREAM_NAME> \
    --region ap-southeast-1
```

**Common Glue Job Errors:**

| Error | Cause | Fix |
|-------|-------|-----|
| `AccessDenied` | IAM role missing permissions | Add S3/Glue permissions to role |
| `AnalysisException: path not found` | Wrong S3 path | Check bucket name and prefix |
| `OutOfMemoryError` | Too much data for allocated DPUs | Increase workers or optimize |
| `SparkException: Job aborted` | Many causes — check logs | View CloudWatch logs |
| `Timeout` | Job ran too long | Increase timeout or optimize query |
| `Catalog table not found` | Crawler hasn't run | Run crawler first, check database name |

**Performance Tips:**
1. Use Parquet instead of CSV (smaller, faster, typed)
2. Partition your output (`partitionBy` in Spark)
3. Use `push_down_predicate` to skip reading irrelevant partitions
4. Enable job bookmarks (skip already-processed data)
5. Right-size workers: G.1X (1 DPU) for small jobs, G.2X for memory-intensive

---

## 🌙 Evening Block (1 hour): Glue Triggers + Review

### Step 11: Glue Triggers — Automate Your Jobs (20 min)

```bash
# Create a trigger to run a job on schedule (like a cron job)
aws glue create-trigger \
    --name makanexpress-daily-etl-trigger \
    --type SCHEDULED \
    --schedule "cron(0 6 ? * * *)" \
    --actions '[{"JobName": "makanexpress-filter-restaurants"}]' \
    --description "Run daily at 6 AM SGT (UTC+8 = 22:00 UTC previous day)" \
    --region ap-southeast-1

# Wait — cron in Glue uses UTC, not SGT!
# 6 AM SGT = 10 PM UTC previous day
# cron(0 22 ? * * *) = 10 PM UTC = 6 AM SGT
# Actually let's fix that:
aws glue delete-trigger --name makanexpress-daily-etl-trigger --region ap-southeast-1

aws glue create-trigger \
    --name makanexpress-daily-etl-trigger \
    --type SCHEDULED \
    --schedule "cron(0 22 ? * * *)" \
    --actions '[{"JobName": "makanexpress-filter-restaurants"}]' \
    --start-on-creation \
    --region ap-southeast-1

# Create a trigger that runs after another job completes (pipeline)
aws glue create-trigger \
    --name makanexpress-after-crawler \
    --type CONDITIONAL \
    --predicate '{
        "Logical": "AND",
        "Conditions": [{
            "CrawlerName": "makanexpress-restaurant-crawler",
            "CrawlState": "SUCCEEDED"
        }]
    }' \
    --actions '[{"JobName": "makanexpress-filter-restaurants"}]' \
    --region ap-southeast-1
```

### Step 12: Job Bookmarks (Incremental Processing) (15 min)

```python
# Enable bookmarks to skip already-processed data
# In your Glue job arguments:
{
    "--enable-job-bookmarks": "true",
    "--job-bookmark-option": "job-bookmark-enable"
}

# In your Glue job code:
# Just use the same source — Glue tracks what's been processed
source_dyf = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_catalog",
    table_name="restaurants",
    transformation_ctx="restaurants",  # Required for bookmarks!
    additional_options={
        "useCatalogSchema": True
    }
)

# First run: processes all data
# Second run: only processes NEW data since last run
# Like Airflow's catchup=False + incremental models in dbt
```

**Why bookmarks matter:** If you receive daily CSV files in S3, you don't want to reprocess yesterday's data every day. Bookmarks track what's been processed and only handle new files.

### Exercise 3: Glue Architecture Questions

Answer these without looking at notes:

**Q1.** What are the 3 main components of AWS Glue? What does each do?

**Q2.** What's the difference between a DynamicFrame and a DataFrame? When would you use each?

**Q3.** A Glue job needs to read 50GB of CSV, filter rows, and write Parquet. How many DPUs would you allocate? What's the estimated cost?

**Q4.** Your crawler created a table but the column types are wrong (everything is STRING). How do you fix this?

**Q5.** You want your Glue job to run every day at 9 AM SGT, but only process new files. How do you set this up?

<details>
<summary>🔑 Answer</summary>

**A1:**
- **Glue Catalog:** Metadata store (schemas, locations, formats) — like a library catalog
- **Glue Crawler:** Scans S3/databases, discovers schemas, updates the catalog — like a librarian
- **Glue Job:** Runs PySpark/Python ETL scripts on serverless Spark — the actual processing

**A2:**
- DynamicFrame: Glue's format, handles flexible/messy schemas, works with catalog. Use for reading from catalog and writing.
- DataFrame: Standard Spark format, more operations (SQL, window functions), Pandas-like API. Use for complex transformations.
- Convert between them: `.toDF()` and `DynamicFrame.fromDF()`

**A3:**
- Start with 2 DPUs (minimum for Spark). If it's slow, increase to 5-10.
- 2 DPUs × ~15 min run × $0.44/DPU-hour = 2 × 0.25 × $0.44 = **$0.22 per run**
- Monthly (daily): ~$6.60/month — very affordable

**A4:**
Option 1: Edit the table schema in the catalog manually
Option 2: Re-run the crawler with a custom classifier
Option 3: Cast types in your Glue job using `ApplyMapping` or DataFrame operations
Option 4: Define the schema explicitly when creating the table

**A5:**
- Create a scheduled trigger: `cron(0 1 ? * * *)` (9 AM SGT = 1 AM UTC)
- Enable job bookmarks: `--enable-job-bookmarks: true`
- The bookmark tracks processed files, so only new ones get processed

</details>

---

## 📚 Day 44 Summary

### What You Learned

| Topic | Key Takeaway |
|-------|-------------|
| Glue Catalog | Central metadata store for schemas, locations, formats |
| Glue Crawler | Auto-discovers data schema from S3/databases |
| Glue Job Types | Spark ETL (big data), Python Shell (small tasks), Ray (ML) |
| Visual ETL | Drag-and-drop pipeline builder — good for simple jobs |
| PySpark Script | Write Glue jobs in PySpark — full control, production-ready |
| DynamicFrame | Glue's DataFrame — handles messy schemas, built-in transforms |
| Job Monitoring | CloudWatch logs, run history, error troubleshooting |
| Triggers | Schedule jobs or chain them (like Airflow for Glue) |
| Bookmarks | Incremental processing — skip already-processed data |
| DPU & Cost | 1 DPU = 4 vCPU + 16GB RAM, $0.44/DPU-hour, min 2 for Spark |

---

## ✅ Daily Checklist

- [ ] Created Glue database `makanexpress_catalog`
- [ ] Created IAM role `MakanExpressGlueRole`
- [ ] Uploaded test data to S3
- [ ] Created and ran restaurant crawler
- [ ] Inspected catalog schema via CLI
- [ ] Completed Exercise 1 (orders crawler)
- [ ] Created Visual ETL job in Glue Studio
- [ ] Wrote PySpark Glue script (`glue_job_filter_restaurants.py`)
- [ ] Uploaded and ran the script job
- [ ] Completed Exercise 2 (modify job for aggregation)
- [ ] Understand DynamicFrame vs DataFrame
- [ ] Created a scheduled trigger
- [ ] Understand job bookmarks
- [ ] Completed Exercise 3 (architecture questions)
- [ ] Ready for tomorrow: AWS Athena (SQL on S3)

---

## 🔖 Quick Reference Card

```
# GLUE CRAWLER
aws glue create-crawler --name X --role ARN --database DB \
    --targets '{"S3Targets":[{"Path":"s3://bucket/path/"}]}'
aws glue start-crawler --name X
aws glue get-crawler --name X --query 'Crawler.State'

# GLUE CATALOG
aws glue get-tables --database-name DB
aws glue get-table --database-name DB --name TABLE

# GLUE JOB (PySpark boilerplate)
from awsglue.context import GlueContext
from awsglue.job import Job
glueContext = GlueContext(SparkContext())
spark = glueContext.spark_session
# Read from catalog → DynamicFrame → DataFrame → transform → write

# READ
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="db", table_name="tbl")
df = dyf.toDF()

# WRITE
glueContext.write_dynamic_frame.from_options(
    frame=dyf, connection_type="s3",
    connection_options={"path": "s3://bucket/path/"},
    format="glueparquet")

# GLUE JOB CLI
aws glue start-job-run --job-name X
aws glue get-job-runs --job-name X --max-results 1

# TRIGGER (scheduled)
aws glue create-trigger --name X --type SCHEDULED \
    --schedule "cron(0 6 ? * * *)" \
    --actions '[{"JobName":"my-job"}]'
```

---

*Day 44 complete! Tomorrow: AWS Athena — query your S3 data with pure SQL.* 🕷️🔍
