# 📅 Day 49 — Wednesday, 2 July 2026
# 🏗️ Portfolio Project: Build Serverless Pipeline for MakanExpress

---

## 🎯 Today's Goal

This is the **Week 7 capstone** — you're building a complete, production-style serverless data pipeline for MakanExpress using **S3 + Glue + Athena**. By end of today, you'll have:

1. A 3-zone S3 data lake (raw → processed → analytics)
2. Glue crawlers that discover your data
3. Glue ETL jobs that transform CSV → partitioned Parquet
4. Athena tables for querying analytics
5. Everything documented, committed, and ready to show employers

**Why this matters:** This project demonstrates you understand modern serverless data engineering. When an interviewer asks "tell me about a data pipeline you've built", this is your answer.

---

## Architecture: What We're Building

```
                    MAKANEXPRESS SERVERLESS DATA PIPELINE
                    Portfolio Project — Week 7

┌─────────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                                 │
│  CSV Orders  │  CSV Order Items  │  CSV Restaurants  │  CSV Riders │
└──────┬───────┴────────┬──────────┴─────────┬─────────┴──────┬──────┘
       │                │                    │                │
       ▼                ▼                    ▼                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  S3 DATA LAKE: makanexpress-datalake-dev                            │
│                                                                     │
│  ┌─────────────┐   ┌──────────────┐   ┌────────────────┐           │
│  │ raw/        │   │ processed/   │   │ analytics/     │           │
│  │ (CSV as-is) │──▶│ (Parquet,    │──▶│ (Parquet,      │           │
│  │             │   │  partitioned,│   │  aggregated,   │           │
│  │ Immutable   │   │  cleaned)    │   │  business-ready│           │
│  └─────────────┘   └──────────────┘   └────────────────┘           │
│                                                                     │
│  ┌─────────────┐   ┌──────────────┐                                │
│  │ scripts/    │   │ athena-      │                                │
│  │ (Glue jobs) │   │ results/     │                                │
│  └─────────────┘   └──────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
       │                │                    │
       ▼                ▼                    ▼
┌──────────────┐  ┌──────────────┐   ┌──────────────┐
│ GLUE         │  │ GLUE         │   │ ATHENA       │
│ Crawlers     │  │ ETL Jobs     │   │ Analytics    │
│ (3 crawlers) │  │ (PySpark)    │   │ Queries      │
│              │  │              │   │              │
│ raw_orders   │  │ orders CSV   │   │ Revenue by   │
│ raw_items    │  │ → Parquet    │   │ restaurant   │
│ processed_*  │  │ items CSV    │   │ Top dishes   │
│              │  │ → Parquet    │   │ Rider stats  │
│              │  │              │   │ Peak hours   │
└──────────────┘  └──────────────┘   └──────────────┘
```

---

## ☀️ Morning Block (2 hours): Design + Setup

### Step 1: Create the Project Structure

```bash
# Local project structure
mkdir -p makanexpress-serverless-pipeline
cd makanexpress-serverless-pipeline

mkdir -p scripts/{glue-jobs,crawlers,athena-queries}
mkdir -p sample-data/{raw,processed}
mkdir -p infrastructure/{iam,cloudwatch}
mkdir -p docs
```

### Step 2: Generate Realistic Multi-Day Data

```python
# scripts/generate_sample_data.py
"""
Generate realistic MakanExpress data for 7 days.
Simulates daily CSV drops from the order system.
"""
import csv
import random
from datetime import datetime, timedelta
import os

# Realistic SG/MY restaurants
RESTAURANTS = [
    ("R001", "Hawker Chan", "Chinatown", "Chinese", "SG"),
    ("R002", "Jumbo Seafood", "Clarke Quay", "Seafood", "SG"),
    ("R003", "Song Fa Bak Kut Teh", "Clarke Quay", "Chinese", "SG"),
    ("R004", "Ya Kun Kaya Toast", "Marina Bay", "Breakfast", "SG"),
    ("R005", "Tiong Bahru Bakery", "Tiong Bahru", "Bakery", "SG"),
    ("R006", "PappaRich", "Bukit Bintang", "Malaysian", "MY"),
    ("R007", "Nasi Lemak Antarabangsa", "Bangsar", "Malaysian", "MY"),
    ("R008", "IPPUDO RAMEN", "Orchard", "Japanese", "SG"),
    ("R009", "Shake Shack", "Orchard", "Western", "SG"),
    ("R010", "wild honey", "Scotts Square", "Brunch", "SG"),
    ("R011", "Putien", "Marina Bay Sands", "Chinese", "SG"),
    ("R012", "Burnt Ends", "Chinatown", "BBQ", "SG"),
    ("R013", "Din Tai Fung", "Orchard", "Chinese/Taiwanese", "SG"),
    ("R014", "Madam Kwan's", "KLCC", "Malaysian", "MY"),
    ("R015", "Fatty Crab", "TTDI", "Seafood", "MY"),
]

CUSTOMERS = [f"C{str(i).zfill(5)}" for i in range(1, 201)]
RIDERS = [f"RD{str(i).zfill(4)}" for i in range(1, 31)]

MENU_ITEMS = [
    ("I001", "Chicken Rice", 5.50, "R001"),
    ("I002", "Laksa", 4.80, "R001"),
    ("I003", "Chili Crab", 35.00, "R002"),
    ("I004", "Cereal Prawn", 18.00, "R002"),
    ("I005", "Bak Kut Teh (Regular)", 7.50, "R003"),
    ("I006", "Kaya Toast Set", 4.50, "R004"),
    ("I007", "Soft Boiled Eggs (2pc)", 2.00, "R004"),
    ("I008", "Croissant", 4.20, "R005"),
    ("I009", "Kopi", 2.50, "R005"),
    ("I010", "Nasi Lemak Special", 12.00, "R006"),
    ("I011", "Char Kway Teow", 5.20, "R006"),
    ("I012", "Nasi Lemak Royale", 8.50, "R007"),
    ("I013", "Tonkotsu Ramen", 16.00, "R008"),
    ("I014", "ShackBurger", 10.50, "R009"),
    ("I015", "Big Brekkie", 22.00, "R010"),
    ("I016", "Fried Rice", 6.00, "R011"),
    ("I017", "Iberico Pork Ribs", 28.00, "R012"),
    ("I018", "Xiao Long Bao (10pc)", 12.00, "R013"),
    ("I019", "Nasi Bojari", 14.00, "R014"),
    ("I020", "Black Pepper Crab", 38.00, "R015"),
]

# Hourly order distribution (peak at lunch and dinner)
HOUR_WEIGHTS = {
    6: 2, 7: 5, 8: 8, 9: 4, 10: 3, 11: 6,
    12: 15, 13: 12, 14: 5, 15: 3, 16: 4, 17: 6,
    18: 14, 19: 12, 20: 8, 21: 5, 22: 3, 23: 1
}

AREAS = ["Chinatown", "Orchard", "Marina Bay", "Clarke Quay", "Tiong Bahru",
         "Bukit Bintang", "Bangsar", "KLCC", "TTDI", "CBD"]


def generate_day_data(date_str, num_orders_range=(150, 300)):
    """Generate all data for one day."""
    num_orders = random.randint(*num_orders_range)
    base_date = datetime.strptime(date_str, "%Y-%m-%d")
    
    orders = []
    order_items = []
    hourly_counts = {h: 0 for h in range(6, 24)}
    
    for i in range(1, num_orders + 1):
        order_id = f"ORD{date_str.replace('-','')}{str(i).zfill(5)}"
        
        # Weighted hour selection for realistic distribution
        hours = list(HOUR_WEIGHTS.keys())
        weights = list(HOUR_WEIGHTS.values())
        hour = random.choices(hours, weights=weights, k=1)[0]
        hourly_counts[hour] += 1
        
        minute = random.randint(0, 59)
        second = random.randint(0, 59)
        order_time = base_date + timedelta(hours=hour, minutes=minute, seconds=second)
        
        restaurant = random.choice(RESTAURANTS)
        customer = random.choice(CUSTOMERS)
        rider = random.choice(RIDERS)
        
        # Items from this restaurant's menu + random cross-ordering
        restaurant_items = [item for item in MENU_ITEMS if item[3] == restaurant[0]]
        other_items = random.sample(MENU_ITEMS, min(3, len(MENU_ITEMS)))
        available_items = restaurant_items + other_items
        
        num_items = random.randint(1, 4)
        chosen_items = random.sample(available_items, min(num_items, len(available_items)))
        
        # Calculate amounts
        item_total = 0
        for item in chosen_items:
            qty = random.randint(1, 3)
            item_total += item[2] * qty
            order_items.append({
                "order_id": order_id,
                "item_id": item[0],
                "item_name": item[1],
                "quantity": qty,
                "unit_price": f"{item[2]:.2f}",
            })
        
        delivery_fee = round(random.uniform(2.00, 5.50), 2)
        total_amount = round(item_total + delivery_fee, 2)
        
        # Realistic status distribution
        status = random.choices(
            ["delivered", "cancelled", "refunded"],
            weights=[0.88, 0.10, 0.02]
        )[0]
        
        if status == "cancelled":
            delivery_fee = 0.00
            total_amount = round(item_total, 2)
        elif status == "refunded":
            total_amount = 0.00
            delivery_fee = 0.00
        
        orders.append({
            "order_id": order_id,
            "order_date": date_str,
            "order_time": order_time.strftime("%H:%M:%S"),
            "customer_id": customer,
            "restaurant_id": restaurant[0],
            "restaurant_name": restaurant[1],
            "restaurant_area": restaurant[2],
            "rider_id": rider,
            "total_amount": f"{total_amount:.2f}",
            "delivery_fee": f"{delivery_fee:.2f}",
            "status": status,
            "payment_method": random.choice(
                ["GrabPay", "Credit Card", "Cash", "ShopeePay", "Apple Pay"]
            ),
        })
    
    return orders, order_items


def generate_restaurants():
    """Generate restaurant reference data."""
    return [{
        "restaurant_id": r[0],
        "restaurant_name": r[1],
        "area": r[2],
        "cuisine_type": r[3],
        "country": r[4],
        "is_active": "true",
    } for r in RESTAURANTS]


def generate_riders():
    """Generate rider reference data."""
    return [{
        "rider_id": r,
        "rider_name": f"Rider_{r}",
        "vehicle_type": random.choice(["motorcycle", "bicycle", "car"]),
        "base_area": random.choice(AREAS),
        "rating": f"{random.uniform(3.5, 5.0):.1f}",
        "is_active": "true",
    } for r in RIDERS]


def write_csv(data, filepath):
    """Write list of dicts to CSV."""
    os.makedirs(os.path.dirname(filepath), exist_ok=True)
    with open(filepath, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=data[0].keys())
        writer.writeheader()
        writer.writerows(data)


if __name__ == "__main__":
    # Generate 7 days of data (Jun 26 - Jul 2, 2026 — Week 7 dates!)
    dates = [f"2026-06-{str(d).zfill(2)}" for d in range(26, 31)] + \
            [f"2026-07-{str(d).zfill(2)}" for d in range(1, 3)]
    
    total_orders = 0
    for date_str in dates:
        orders, order_items = generate_day_data(date_str)
        
        write_csv(orders, f"sample-data/raw/orders/{date_str}/orders.csv")
        write_csv(order_items, f"sample-data/raw/order_items/{date_str}/order_items.csv")
        
        total_orders += len(orders)
        print(f"  {date_str}: {len(orders)} orders, {len(order_items)} items")
    
    # Reference data (no date partitioning)
    write_csv(generate_restaurants(), "sample-data/raw/restaurants/restaurants.csv")
    write_csv(generate_riders(), "sample-data/raw/riders/riders.csv")
    
    print(f"\n✅ Generated {total_orders} total orders across {len(dates)} days")
    print("📁 Data ready in sample-data/raw/")
```

```bash
python scripts/generate_sample_data.py
```

### Step 3: Set Up S3 Data Lake

```bash
# Variables (adjust if needed)
BUCKET="makanexpress-serverless-dev"
REGION="ap-southeast-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create the data lake bucket
aws s3 mb s3://$BUCKET --region $REGION

# Create the zone structure
for zone in raw/orders raw/order_items raw/restaurants raw/riders \
            processed/orders processed/order_items processed/restaurants processed/riders \
            analytics/daily_summary analytics/restaurant_performance \
            scripts temp athena-results; do
    aws s3api put-object --bucket $BUCKET --key "$zone/"
done

echo "✅ S3 data lake structure created"

# Upload raw data
aws s3 sync sample-data/raw/ s3://$BUCKET/raw/ --region $REGION
echo "✅ Raw data uploaded"

# Verify
aws s3 ls s3://$BUCKET/raw/orders/ --recursive | wc -l
echo "CSV files in raw/orders/"
```

### Step 4: Create IAM Role for Glue

```bash
# Create trust policy document
cat > infrastructure/iam/glue-trust-policy.json << 'EOF'
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
    --assume-role-policy-document file://infrastructure/iam/glue-trust-policy.json

# Attach AWS managed Glue service policy
aws iam attach-role-policy \
    --role-name MakanExpressGlueRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

# Create and attach custom S3 policy
cat > infrastructure/iam/glue-s3-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::$BUCKET",
                "arn:aws:s3::$BUCKET/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::$BUCKET/scripts/*",
                "arn:aws:s3:::$BUCKET/temp/*",
                "arn:aws:s3:::$BUCKET/athena-results/*"
            ]
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name MakanExpressGlueRole \
    --policy-name MakanExpressS3Access \
    --policy-document file://infrastructure/iam/glue-s3-policy.json

# Add Athena permissions
cat > infrastructure/iam/glue-athena-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": [
            "athena:StartQueryExecution",
            "athena:GetQueryExecution",
            "athena:GetQueryResults",
            "athena:StopQueryExecution"
        ],
        "Resource": "*"
    }]
}
EOF

aws iam put-role-policy \
    --role-name MakanExpressGlueRole \
    --policy-name MakanExpressAthenaAccess \
    --policy-document file://infrastructure/iam/glue-athena-policy.json

echo "✅ IAM role created with S3 + Glue + Athena permissions"
echo "Role ARN: arn:aws:iam::$ACCOUNT_ID:role/MakanExpressGlueRole"
```

### Step 5: Create Glue Database and Crawlers

```bash
ROLE_ARN="arn:aws:iam::$ACCOUNT_ID:role/MakanExpressGlueRole"

# Create database
aws glue create-database \
    --database-input '{"Name": "makanexpress_analytics", "Description": "MakanExpress serverless pipeline - portfolio project"}'

# Crawler 1: Raw orders
aws glue create-crawler \
    --name mx-raw-orders \
    --role $ROLE_ARN \
    --database-name makanexpress_analytics \
    --targets '{"S3Targets": [{"Path": "s3://'$BUCKET'/raw/orders/", "Exclusions": []}]}' \
    --table-prefix "raw_" \
    --schema-change-policy '{"UpdateBehavior": "UPDATE_IN_DATABASE", "DeleteBehavior": "LOG"}' \
    --description "Crawl raw MakanExpress order CSVs"

# Crawler 2: Raw order items
aws glue create-crawler \
    --name mx-raw-items \
    --role $ROLE_ARN \
    --database-name makanexpress_analytics \
    --targets '{"S3Targets": [{"Path": "s3://'$BUCKET'/raw/order_items/"}]}' \
    --table-prefix "raw_" \
    --schema-change-policy '{"UpdateBehavior": "UPDATE_IN_DATABASE", "DeleteBehavior": "LOG"}' \
    --description "Crawl raw MakanExpress order item CSVs"

# Crawler 3: Raw restaurants (reference data)
aws glue create-crawler \
    --name mx-raw-restaurants \
    --role $ROLE_ARN \
    --database-name makanexpress_analytics \
    --targets '{"S3Targets": [{"Path": "s3://'$BUCKET'/raw/restaurants/"}]}' \
    --table-prefix "raw_" \
    --schema-change-policy '{"UpdateBehavior": "UPDATE_IN_DATABASE", "DeleteBehavior": "LOG"}' \
    --description "Crawl raw MakanExpress restaurant reference data"

# Run all crawlers
for crawler in mx-raw-orders mx-raw-items mx-raw-restaurants; do
    aws glue start-crawler --name $crawler
    echo "  Started crawler: $crawler"
done

echo "✅ Crawlers running. Wait ~2-3 minutes..."

# Check status
sleep 30
for crawler in mx-raw-orders mx-raw-items mx-raw-restaurants; do
    status=$(aws glue get-crawler --name $crawler --query 'Crawler.State' --output text)
    echo "  $crawler: $status"
done

# Once crawlers complete, verify tables
aws glue get-tables --database-name makanexpress_analytics \
    --query 'TableList[].{Name:Name,Location:StorageDescriptor.Location}' --output table
```

### 📊 Morning Checkpoint

Before moving to afternoon, verify:
```bash
# Check S3 data exists
aws s3 ls s3://$BUCKET/raw/orders/ --recursive | wc -l
# Should show 7 files (7 days)

# Check Glue tables exist
aws glue get-tables --database-name makanexpress_analytics --query 'TableList[].Name'
# Should show: raw_orders, raw_order_items, raw_restaurants

# Quick Athena test (in console)
# SELECT * FROM raw_orders LIMIT 5;
```

✅ Morning checkpoint passed? Move to afternoon for the ETL jobs.

---

## 🌤️ Afternoon Block (2 hours): Glue ETL + Athena Analytics

### Step 6: Write the Orders ETL Job

```python
# scripts/glue-jobs/etl_orders.py
"""
MakanExpress Orders ETL: raw CSV → processed Parquet
Deploy as AWS Glue PySpark job.

Transformations:
1. Cast string types to proper types
2. Add date partition columns
3. Clean invalid records
4. Calculate derived fields (net_amount)
5. Write partitioned Parquet output
"""
import sys
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.context import SparkContext
from pyspark.sql import functions as F

# ─── Initialization ───
args = getResolvedOptions(sys.argv, [
    'JOB_NAME',
    'output_bucket'
])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

OUTPUT_BUCKET = args['output_bucket']
logger = glueContext.get_logger()

# ─── READ: From Glue Data Catalog ───
logger.info("📖 Reading raw orders from Glue Catalog...")

orders_source = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_analytics",
    table_name="raw_orders",
    transformation_ctx="read_orders"
)

orders_df = orders_source.toDF()
initial_count = orders_df.count()
logger.info(f"   Loaded {initial_count} raw order records")

# ─── TRANSFORM: Clean and Enrich ───
logger.info("🔧 Transforming orders...")

orders_typed = orders_df \
    .withColumn("total_amount", F.col("total_amount").cast("double")) \
    .withColumn("delivery_fee", F.col("delivery_fee").cast("double")) \
    .withColumn("order_date_parsed", F.to_date(F.col("order_date"), "yyyy-MM-dd")) \
    .withColumn("order_hour", F.hour(F.to_timestamp(F.col("order_time"), "HH:mm:ss")))

# Add partition columns
orders_partitioned = orders_typed \
    .withColumn("order_year", F.year(F.col("order_date_parsed"))) \
    .withColumn("order_month", F.month(F.col("order_date_parsed"))) \
    .withColumn("order_day", F.dayofmonth(F.col("order_date_parsed")))

# Data quality: remove invalid records
orders_clean = orders_partitioned \
    .filter(F.col("order_id").isNotNull()) \
    .filter(F.col("total_amount") >= 0) \
    .filter(F.col("status").isin("delivered", "cancelled", "refunded"))

# Derived fields
orders_final = orders_clean \
    .withColumn(
        "net_amount",
        F.when(F.col("status") == "refunded", F.lit(0.0))
         .when(F.col("status") == "cancelled", F.lit(0.0))
         .otherwise(F.col("total_amount") - F.col("delivery_fee"))
    ) \
    .withColumn(
        "time_period",
        F.when(F.col("order_hour").between(6, 10), "breakfast")
         .when(F.col("order_hour").between(11, 14), "lunch")
         .when(F.col("order_hour").between(15, 17), "afternoon")
         .when(F.col("order_hour").between(18, 21), "dinner")
         .otherwise("late_night")
    )

# Drop helper columns
orders_output = orders_final.drop("order_date_parsed")

clean_count = orders_output.count()
rejected = initial_count - clean_count
logger.info(f"   Clean: {clean_count} records ({rejected} rejected)")

# ─── WRITE: Partitioned Parquet ───
logger.info(f"💾 Writing to s3://{OUTPUT_BUCKET}/processed/orders/...")

orders_dynamic = DynamicFrame.fromDF(orders_output, glueContext, "orders_output")

glueContext.write_dynamic_frame.from_options(
    frame=orders_dynamic,
    connection_type="s3",
    connection_options={
        "path": f"s3://{OUTPUT_BUCKET}/processed/orders/",
        "partitionKeys": ["order_year", "order_month", "order_day"]
    },
    format="glueparquet",
    format_options={"compression": "snappy"},
    transformation_ctx="write_orders"
)

logger.info(f"✅ Orders ETL complete! {clean_count} records written")
job.commit()
```

### Step 7: Write the Order Items ETL Job

```python
# scripts/glue-jobs/etl_order_items.py
"""
MakanExpress Order Items ETL: raw CSV → processed Parquet
"""
import sys
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.context import SparkContext
from pyspark.sql import functions as F

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'output_bucket'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

OUTPUT_BUCKET = args['output_bucket']
logger = glueContext.get_logger()

# Read
logger.info("📖 Reading raw order items...")
items_source = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_analytics",
    table_name="raw_order_items",
    transformation_ctx="read_items"
)
items_df = items_source.toDF()
logger.info(f"   Loaded {items_df.count()} order item records")

# Transform
logger.info("🔧 Transforming order items...")
items_transformed = items_df \
    .withColumn("quantity", F.col("quantity").cast("int")) \
    .withColumn("unit_price", F.col("unit_price").cast("double")) \
    .withColumn("line_total", F.col("quantity") * F.col("unit_price")) \
    .filter(F.col("order_id").isNotNull()) \
    .filter(F.col("quantity") > 0)

# Extract date from order_id pattern: ORD2026062600001
# Format: ORD + YYYYMMDD + sequential number
items_with_date = items_transformed \
    .withColumn("order_date_str", F.substring(F.col("order_id"), 4, 8)) \
    .withColumn("order_date", F.to_date(F.col("order_date_str"), "yyyyMMdd")) \
    .withColumn("order_year", F.year(F.col("order_date"))) \
    .withColumn("order_month", F.month(F.col("order_date"))) \
    .withColumn("order_day", F.dayofmonth(F.col("order_date")))

items_final = items_with_date.drop("order_date_str")

logger.info(f"   Transformed: {items_final.count()} items")

# Write
logger.info(f"💾 Writing to s3://{OUTPUT_BUCKET}/processed/order_items/...")

items_dynamic = DynamicFrame.fromDF(items_final, glueContext, "items_output")

glueContext.write_dynamic_frame.from_options(
    frame=items_dynamic,
    connection_type="s3",
    connection_options={
        "path": f"s3://{OUTPUT_BUCKET}/processed/order_items/",
        "partitionKeys": ["order_year", "order_month", "order_day"]
    },
    format="glueparquet",
    format_options={"compression": "snappy"},
    transformation_ctx="write_items"
)

logger.info("✅ Order Items ETL complete!")
job.commit()
```

### Step 8: Deploy and Run Glue Jobs

```bash
BUCKET="makanexpress-serverless-dev"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ROLE_ARN="arn:aws:iam::$ACCOUNT_ID:role/MakanExpressGlueRole"

# Upload scripts to S3
aws s3 cp scripts/glue-jobs/etl_orders.py s3://$BUCKET/scripts/etl_orders.py
aws s3 cp scripts/glue-jobs/etl_order_items.py s3://$BUCKET/scripts/etl_order_items.py

# Create Orders ETL Job
aws glue create-job \
    --name mx-etl-orders \
    --role $ROLE_ARN \
    --command '{
        "Name": "glueetl",
        "ScriptLocation": "s3://'$BUCKET'/scripts/etl_orders.py"
    }' \
    --default-arguments '{
        "--TempDir": "s3://'$BUCKET'/temp/",
        "--job-language": "python",
        "--output_bucket": "'$BUCKET'",
        "--job-bookmark-option": "job-bookmark-enable",
        "--enable-metrics": "true"
    }' \
    --glue-version "4.0" \
    --number-of-workers 2 \
    --worker-type G.1X \
    --description "Transform MakanExpress raw orders to processed Parquet"

# Create Order Items ETL Job
aws glue create-job \
    --name mx-etl-order-items \
    --role $ROLE_ARN \
    --command '{
        "Name": "glueetl",
        "ScriptLocation": "s3://'$BUCKET'/scripts/etl_order_items.py"
    }' \
    --default-arguments '{
        "--TempDir": "s3://'$BUCKET'/temp/",
        "--job-language": "python",
        "--output_bucket": "'$BUCKET'",
        "--job-bookmark-option": "job-bookmark-enable",
        "--enable-metrics": "true"
    }' \
    --glue-version "4.0" \
    --number-of-workers 2 \
    --worker-type G.1X \
    --description "Transform MakanExpress raw order items to processed Parquet"

# Run both jobs
echo "🚀 Starting ETL jobs..."
aws glue start-job-run --job-name mx-etl-orders
aws glue start-job-run --job-name mx-etl-order-items

# Monitor (check every 30 seconds)
echo "⏳ Waiting for jobs to complete..."
for i in {1..20}; do
    sleep 30
    orders_status=$(aws glue get-job-runs --job-name mx-etl-orders --max-results 1 \
        --query 'JobRuns[0].JobRunState' --output text 2>/dev/null || echo "PENDING")
    items_status=$(aws glue get-job-runs --job-name mx-etl-order-items --max-results 1 \
        --query 'JobRuns[0].JobRunState' --output text 2>/dev/null || echo "PENDING")
    echo "  [$i] Orders: $orders_status | Items: $items_status"
    
    if [[ "$orders_status" == "SUCCEEDED" && "$items_status" == "SUCCEEDED" ]]; then
        echo "✅ Both jobs completed successfully!"
        break
    fi
done
```

### Step 9: Create Processed Zone Crawlers + Query with Athena

```bash
# Crawler for processed orders
aws glue create-crawler \
    --name mx-processed-orders \
    --role $ROLE_ARN \
    --database-name makanexpress_analytics \
    --targets '{"S3Targets": [{"Path": "s3://'$BUCKET'/processed/orders/"}]}' \
    --table-prefix "processed_" \
    --description "Crawl processed MakanExpress orders (Parquet)"

# Crawler for processed order items
aws glue create-crawler \
    --name mx-processed-items \
    --role $ROLE_ARN \
    --database-name makanexpress_analytics \
    --targets '{"S3Targets": [{"Path": "s3://'$BUCKET'/processed/order_items/"}]}' \
    --table-prefix "processed_" \
    --description "Crawl processed MakanExpress order items (Parquet)"

# Run processed crawlers
aws glue start-crawler --name mx-processed-orders
aws glue start-crawler --name mx-processed-items

echo "⏳ Crawlers running..."
sleep 60

# Verify processed tables
aws glue get-tables --database-name makanexpress_analytics \
    --query 'TableList[].{Name:Name, Format:StorageDescriptor.InputFormat}' --output table
```

### Step 10: Athena Analytics Queries

Now the fun part — analytics! Save these queries for your portfolio:

```sql
-- scripts/athena-queries/analytics_queries.sql
-- MakanExpress Analytics Queries
-- Run these in AWS Athena against makanexpress_analytics database

-- ═══════════════════════════════════════════
-- QUERY 1: Daily Revenue Dashboard
-- ═══════════════════════════════════════════
SELECT 
    order_date,
    COUNT(*) as total_orders,
    SUM(CASE WHEN status = 'delivered' THEN 1 ELSE 0 END) as delivered,
    SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) as cancelled,
    SUM(CASE WHEN status = 'refunded' THEN 1 ELSE 0 END) as refunded,
    ROUND(SUM(net_amount), 2) as net_revenue,
    ROUND(AVG(CASE WHEN status = 'delivered' THEN net_amount END), 2) as avg_order_value
FROM processed_orders
WHERE order_year = 2026
GROUP BY order_date
ORDER BY order_date;


-- ═══════════════════════════════════════════
-- QUERY 2: Top 10 Restaurants by Revenue
-- ═══════════════════════════════════════════
SELECT 
    restaurant_name,
    restaurant_area,
    COUNT(*) as total_orders,
    ROUND(SUM(net_amount), 2) as total_revenue,
    ROUND(AVG(net_amount), 2) as avg_order_value,
    ROUND(SUM(delivery_fee), 2) as total_delivery_fees
FROM processed_orders
WHERE status = 'delivered'
GROUP BY restaurant_name, restaurant_area
ORDER BY total_revenue DESC
LIMIT 10;


-- ═══════════════════════════════════════════
-- QUERY 3: Peak Hours Analysis
-- ═══════════════════════════════════════════
SELECT 
    order_hour,
    time_period,
    COUNT(*) as order_count,
    ROUND(SUM(net_amount), 2) as revenue,
    ROUND(SUM(net_amount) / COUNT(*), 2) as avg_per_order
FROM processed_orders
WHERE status = 'delivered'
GROUP BY order_hour, time_period
ORDER BY order_hour;


-- ═══════════════════════════════════════════
-- QUERY 4: Most Popular Menu Items
-- ═══════════════════════════════════════════
SELECT 
    item_name,
    SUM(quantity) as total_quantity_sold,
    COUNT(DISTINCT order_id) as orders_appearing_in,
    ROUND(SUM(line_total), 2) as total_item_revenue,
    ROUND(AVG(unit_price), 2) as avg_price
FROM processed_order_items
GROUP BY item_name
ORDER BY total_quantity_sold DESC
LIMIT 15;


-- ═══════════════════════════════════════════
-- QUERY 5: Customer Segmentation (by order frequency)
-- ═══════════════════════════════════════════
SELECT 
    CASE 
        WHEN order_count = 1 THEN 'New (1 order)'
        WHEN order_count BETWEEN 2 AND 3 THEN 'Regular (2-3)'
        WHEN order_count BETWEEN 4 AND 7 THEN 'Frequent (4-7)'
        ELSE 'Power User (8+)'
    END as customer_segment,
    COUNT(*) as customers_in_segment,
    ROUND(AVG(total_spent), 2) as avg_total_spent,
    ROUND(AVG(order_count), 1) as avg_orders
FROM (
    SELECT 
        customer_id,
        COUNT(*) as order_count,
        ROUND(SUM(net_amount), 2) as total_spent
    FROM processed_orders
    WHERE status = 'delivered'
    GROUP BY customer_id
)
GROUP BY 
    CASE 
        WHEN order_count = 1 THEN 'New (1 order)'
        WHEN order_count BETWEEN 2 AND 3 THEN 'Regular (2-3)'
        WHEN order_count BETWEEN 4 AND 7 THEN 'Frequent (4-7)'
        ELSE 'Power User (8+)'
    END
ORDER BY avg_total_spent DESC;


-- ═══════════════════════════════════════════
-- QUERY 6: Payment Method Trends
-- ═══════════════════════════════════════════
SELECT 
    payment_method,
    COUNT(*) as transaction_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 1) as pct_of_total,
    ROUND(SUM(net_amount), 2) as total_revenue_via_method,
    ROUND(AVG(net_amount), 2) as avg_transaction
FROM processed_orders
WHERE status = 'delivered'
GROUP BY payment_method
ORDER BY transaction_count DESC;


-- ═══════════════════════════════════════════
-- QUERY 7: Cancellation Rate by Restaurant
-- ═══════════════════════════════════════════
SELECT 
    restaurant_name,
    COUNT(*) as total_orders,
    SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) as cancelled,
    ROUND(
        SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 
        1
    ) as cancellation_rate_pct
FROM processed_orders
GROUP BY restaurant_name
HAVING COUNT(*) > 10
ORDER BY cancellation_rate_pct DESC;


-- ═══════════════════════════════════════════
-- QUERY 8: Basket Analysis (items frequently ordered together)
-- ═══════════════════════════════════════════
SELECT 
    a.item_name as item_1,
    b.item_name as item_2,
    COUNT(*) as times_ordered_together
FROM processed_order_items a
JOIN processed_order_items b 
    ON a.order_id = b.order_id 
    AND a.item_name < b.item_name  -- avoid duplicates
GROUP BY a.item_name, b.item_name
ORDER BY times_ordered_together DESC
LIMIT 10;
```

### Step 11: Verify the Full Pipeline

```bash
# Check S3 processed zone
echo "📦 Processed orders:"
aws s3 ls s3://$BUCKET/processed/orders/ --recursive | grep ".parquet" | wc -l
echo "Parquet files created"

echo "📦 Processed order items:"
aws s3 ls s3://$BUCKET/processed/order_items/ --recursive | grep ".parquet" | wc -l
echo "Parquet files created"

# Full pipeline summary
echo ""
echo "═══════════════════════════════════════════"
echo "   MAKANEXPRESS PIPELINE STATUS"
echo "═══════════════════════════════════════════"
echo ""
echo "S3 Zones:"
echo "  raw/        → $(aws s3 ls s3://$BUCKET/raw/ --recursive | wc -l) files"
echo "  processed/  → $(aws s3 ls s3://$BUCKET/processed/ --recursive | wc -l) files"
echo ""
echo "Glue Tables:"
aws glue get-tables --database-name makanexpress_analytics \
    --query 'TableList[].Name' --output text | tr '\t' '\n' | sed 's/^/  /'
echo ""
echo "Crawlers:"
for c in mx-raw-orders mx-raw-items mx-raw-restaurants mx-processed-orders mx-processed-items; do
    status=$(aws glue get-crawler --name $c --query 'Crawler.State' --output text 2>/dev/null || echo "N/A")
    echo "  $c: $status"
done
echo ""
echo "ETL Jobs:"
for j in mx-etl-orders mx-etl-order-items; do
    status=$(aws glue get-job-runs --job-name $j --max-results 1 \
        --query 'JobRuns[0].JobRunState' --output text 2>/dev/null || echo "NOT RUN")
    echo "  $j: $status"
done
echo "═══════════════════════════════════════════"
```

---

## 🌙 Evening Block (1 hour): Document + Push to GitHub

### Step 12: Write the README

```markdown
<!-- Save as README.md in project root -->
# 🍜 MakanExpress Serverless Data Pipeline

A complete **serverless data pipeline** for MakanExpress, a fictional Singapore/Malaysia food delivery platform. Built with AWS S3, Glue, and Athena.

## 🏗️ Architecture

```
CSV Files → S3 (raw/) → Glue Crawler → Glue ETL Job → S3 (processed/) → Athena → Analytics
```

### Data Flow
1. **Ingestion**: Daily CSV order files land in S3 raw zone
2. **Discovery**: Glue Crawlers scan raw data and update the Data Catalog
3. **Transformation**: Glue PySpark jobs clean, type-cast, and convert to partitioned Parquet
4. **Query**: Athena provides SQL interface for analytics on processed data

### S3 Data Lake Zones
| Zone | Format | Purpose |
|------|--------|---------|
| `raw/` | CSV | Immutable source data (as-is from source) |
| `processed/` | Parquet (Snappy) | Cleaned, typed, partitioned by date |
| `analytics/` | Parquet | Pre-aggregated business summaries |

## 📊 Sample Analytics

<details>
<summary>📈 Key Queries</summary>

- Daily revenue and order trends
- Top restaurants by revenue and order volume
- Peak ordering hours (lunch/dinner rush)
- Most popular menu items
- Customer segmentation by order frequency
- Payment method distribution
- Cancellation rate analysis by restaurant
- Basket analysis (items ordered together)

</details>

## 🛠️ Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Storage | AWS S3 | Data lake (3-zone architecture) |
| ETL | AWS Glue (PySpark) | CSV → Parquet transformation |
| Catalog | AWS Glue Data Catalog | Metadata management |
| Query | AWS Athena | Serverless SQL on S3 |
| Security | AWS IAM | Role-based access control |
| Scripting | Python, PySpark | ETL logic and data generation |

## 💰 Cost Estimate

For ~1,500 orders/day in Singapore:
- **S3**: ~$0.50/month (raw + processed storage)
- **Glue**: ~$3/month (15 min daily ETL, 2 workers)
- **Athena**: ~$2/month (queries on Parquet, partitioned)
- **Total**: ~$5.50/month

## 📁 Project Structure

```
makanexpress-serverless-pipeline/
├── scripts/
│   ├── generate_sample_data.py      # Data generator
│   ├── glue-jobs/
│   │   ├── etl_orders.py            # Orders ETL (Glue job)
│   │   └── etl_order_items.py       # Order items ETL (Glue job)
│   └── athena-queries/
│       └── analytics_queries.sql    # 8 analytics queries
├── infrastructure/
│   └── iam/
│       ├── glue-trust-policy.json
│       ├── glue-s3-policy.json
│       └── glue-athena-policy.json
├── docs/
│   └── architecture.md              # Architecture documentation
└── README.md
```

## 🚀 How to Recreate

1. Create S3 bucket with 3-zone structure
2. Create IAM role for Glue (trust + S3 + Athena policies)
3. Generate sample data: `python scripts/generate_sample_data.py`
4. Upload to S3: `aws s3 sync sample-data/raw/ s3://BUCKET/raw/`
5. Create Glue database + crawlers → run crawlers
6. Create Glue ETL jobs → run jobs
7. Create processed zone crawlers → run crawlers
8. Query with Athena!

## 📝 Skills Demonstrated

- ✅ Serverless data pipeline architecture
- ✅ S3 data lake design (3-zone pattern)
- ✅ AWS Glue ETL with PySpark
- ✅ Data cataloging and schema management
- ✅ Athena SQL analytics
- ✅ IAM security (least privilege)
- ✅ Cost optimization (Parquet, partitioning)
- ✅ Production error handling and monitoring
```

### Step 13: Create Architecture Documentation

```markdown
<!-- docs/architecture.md -->
# MakanExpress Pipeline Architecture

## Overview

This document describes the serverless data pipeline built for MakanExpress food delivery analytics.

## Data Sources

| Source | Format | Frequency | Volume |
|--------|--------|-----------|--------|
| Orders | CSV | Daily | ~1,500 rows/day |
| Order Items | CSV | Daily | ~3,500 rows/day |
| Restaurants | CSV | Static | 15 rows |
| Riders | CSV | Static | 30 rows |

## Pipeline Components

### 1. S3 Data Lake

```
makanexpress-serverless-dev/
├── raw/
│   ├── orders/{date}/orders.csv          # Daily order files
│   ├── order_items/{date}/order_items.csv # Daily item files
│   ├── restaurants/restaurants.csv       # Reference data
│   └── riders/riders.csv                # Reference data
├── processed/
│   ├── orders/year=XX/month=XX/day=XX/  # Partitioned Parquet
│   └── order_items/year=XX/month=XX/day=XX/
├── analytics/                            # Aggregated outputs
├── scripts/                              # Glue job scripts
├── temp/                                 # Glue temp directory
└── athena-results/                       # Athena query output
```

**Storage class strategy:**
- raw/: S3 Standard (30 days) → Standard-IA (90 days) → Glacier (365 days)
- processed/: S3 Standard (active data)
- athena-results/: S3 Standard (7 day lifecycle delete)

### 2. Glue Data Catalog

Database: `makanexpress_analytics`

| Table | Source | Format | Partitioned |
|-------|--------|--------|-------------|
| raw_orders | S3 raw/orders/ | CSV | No |
| raw_order_items | S3 raw/order_items/ | CSV | No |
| raw_restaurants | S3 raw/restaurants/ | CSV | No |
| processed_orders | S3 processed/orders/ | Parquet | Yes (year/month/day) |
| processed_order_items | S3 processed/order_items/ | Parquet | Yes (year/month/day) |

### 3. Glue ETL Jobs

| Job | Input | Output | Transformations |
|-----|-------|--------|----------------|
| mx-etl-orders | raw_orders | processed/orders/ | Type casting, date parsing, derived fields, quality filters |
| mx-etl-order-items | raw_order_items | processed/order_items/ | Type casting, line_total calculation, date extraction |

### 4. Athena Queries

8 pre-built analytics queries covering:
- Revenue analysis
- Restaurant performance
- Peak hour patterns
- Popular items
- Customer segmentation
- Payment trends
- Cancellation rates
- Basket analysis

## Security

- IAM role with least-privilege policies
- S3 bucket encryption (SSE-S3)
- No public access
- Glue service role separate from user access

## Monitoring

- Glue job metrics in CloudWatch
- Crawler status tracking
- Athena query history
- S3 access logs (if enabled)

## Cost Analysis

| Service | Monthly Cost (est.) | Notes |
|---------|-------------------|-------|
| S3 | $0.50 | ~500MB raw + processed |
| Glue | $3.00 | 15 min/day × 2 DPUs |
| Athena | $2.00 | ~400GB scanned/month |
| **Total** | **$5.50** | Pay-as-you-go |
```

### Step 14: Git Init + Push

```bash
cd makanexpress-serverless-pipeline

# Initialize git
git init
echo "# Ignore local sample data (it's generated)" > .gitignore
echo "sample-data/" >> .gitignore
echo "__pycache__/" >> .gitignore
echo ".DS_Store" >> .gitignore

# Add everything
git add .
git commit -m "Initial commit: MakanExpress serverless data pipeline

- S3 data lake (3-zone: raw/processed/analytics)
- Glue ETL jobs (orders + order_items, CSV → Parquet)
- Athena analytics queries (8 business queries)
- IAM policies (least privilege)
- Sample data generator
- Full documentation

Tech: AWS S3, Glue, Athena, PySpark, Python"

# Create GitHub repo and push
gh repo create makanexpress-serverless-pipeline --public \
    --description "Serverless data pipeline for food delivery analytics - S3 + Glue + Athena"

git remote add origin https://github.com/YOUR_USERNAME/makanexpress-serverless-pipeline.git
git branch -M main
git push -u origin main

echo "✅ Pushed to GitHub!"
```

### Step 15: Portfolio Talking Points

When an interviewer asks about this project, be ready:

**30-second pitch:**
> "I built a serverless data pipeline for a food delivery platform using AWS S3, Glue, and Athena. Raw CSV data lands in S3, Glue crawlers discover the schema, PySpark ETL jobs transform it to partitioned Parquet, and Athena provides SQL analytics. The whole thing costs about $5/month to run."

**Key points to highlight:**
1. **Architecture decisions**: Why 3-zone data lake? Why Parquet over CSV? Why partitioned?
2. **Cost awareness**: You know how to estimate and optimize cloud costs
3. **Data quality**: Your ETL jobs have validation, filtering, and error handling
4. **Security**: IAM roles with least privilege, no hardcoded credentials
5. **Practical SQL**: 8 real analytics queries, not just `SELECT *`

---

## 📝 Today's Checklist

- [ ] Generated sample data (7 days of orders)
- [ ] Created S3 data lake with 3-zone structure
- [ ] Created IAM role with least-privilege policies
- [ ] Created Glue database + crawlers for raw data
- [ ] Ran crawlers successfully → tables created
- [ ] Wrote and deployed Glue ETL job for orders (CSV → Parquet)
- [ ] Wrote and deployed Glue ETL job for order items
- [ ] Created processed zone crawlers → ran them
- [ ] Ran analytics queries in Athena (at least 5 of the 8)
- [ ] Wrote README.md with architecture diagram
- [ ] Wrote docs/architecture.md
- [ ] Pushed to GitHub
- [ ] Prepared portfolio talking points

---

## 🔗 Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│     MAKANEXPRESS SERVERLESS PIPELINE — QUICK REFERENCE       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  AWS SERVICES USED:                                          │
│  ├── S3: Data lake storage (3 zones)                         │
│  ├── Glue Crawlers: Schema discovery (5 crawlers)            │
│  ├── Glue Jobs: PySpark ETL (2 jobs)                         │
│  ├── Glue Data Catalog: Metadata store (1 database, 5 tables)│
│  ├── Athena: SQL analytics on S3 data                        │
│  └── IAM: Security (1 role, 3 policies)                      │
│                                                              │
│  DATA FLOW:                                                  │
│  CSV → S3 raw/ → Glue Crawler → Catalog → Glue Job →        │
│  S3 processed/ (Parquet) → Glue Crawler → Athena Queries    │
│                                                              │
│  KEY FILES:                                                  │
│  ├── scripts/generate_sample_data.py                         │
│  ├── scripts/glue-jobs/etl_orders.py                         │
│  ├── scripts/glue-jobs/etl_order_items.py                    │
│  ├── scripts/athena-queries/analytics_queries.sql            │
│  └── infrastructure/iam/*.json                               │
│                                                              │
│  COST: ~$5.50/month (1,500 orders/day)                       │
│                                                              │
│  CLONE & RECREATE:                                           │
│  1. Create S3 bucket + zones                                 │
│  2. Create IAM role                                          │
│  3. Generate + upload data                                   │
│  4. Create + run crawlers                                    │
│  5. Create + run Glue jobs                                   │
│  6. Create + run processed crawlers                          │
│  7. Query with Athena!                                       │
│                                                              │
│  CLEANUP (avoid charges):                                    │
│  aws s3 rb s3://BUCKET --force                               │
│  aws glue delete-database --name makanexpress_analytics      │
│  aws iam delete-role --role-name MakanExpressGlueRole        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🧹 Cleanup Reminder

**If you're done exploring and don't want ongoing charges:**

```bash
# Delete all crawlers
for c in mx-raw-orders mx-raw-items mx-raw-restaurants mx-processed-orders mx-processed-items; do
    aws glue delete-crawler --name $c 2>/dev/null
done

# Delete Glue jobs
for j in mx-etl-orders mx-etl-order-items; do
    aws glue delete-job --job-name $j
done

# Delete Glue database
aws glue delete-database --name makanexpress_analytics

# Delete IAM role policies + role
aws iam delete-role-policy --role-name MakanExpressGlueRole --policy-name MakanExpressS3Access
aws iam delete-role-policy --role-name MakanExpressGlueRole --policy-name MakanExpressAthenaAccess
aws iam detach-role-policy --role-name MakanExpressGlueRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole
aws iam delete-role --role-name MakanExpressGlueRole

# Delete S3 bucket
aws s3 rb s3://makanexpress-serverless-dev --force

echo "✅ All AWS resources cleaned up"
```

**⚠️ BUT:** If you want to show this to employers, keep it running! It costs ~$5/month and gives you a live demo.

---

*Day 49 complete! Week 7 finished! 🎉*

*You now have a complete serverless data pipeline in your portfolio. Next week: Catch-up + Portfolio Integration + Docker Basics.*

---

*Portfolio Project 5: MakanExpress Serverless Pipeline — COMPLETE* ✅🚀
