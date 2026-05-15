# 📅 Day 48 — Tuesday, 1 July 2026
# 📝 Week 7 Assessment: AWS Glue + Athena + PySpark

---

## 🎯 Today's Goal

Timed assessment covering all of Week 7. Grade yourself honestly and identify weak spots before tomorrow's portfolio project.

**No new concepts today.** This is test + review.

---

## ☀️ Morning Block (2 hours): Timed Assessment

### Section A: PySpark Fundamentals (15 minutes, 5 questions)

**Q1.** Explain the difference between a Spark DataFrame transformation (like `.filter()`) and an action (like `.count()`). Why does Spark use this "lazy evaluation" approach?

**Q2.** Write PySpark code to:
- Create a DataFrame from a list of tuples: `[(1, "Chicken Rice", 5.50), (2, "Laksa", 4.80), (3, "Nasi Lemak", 6.00)]`
- Add a column `price_category` that shows "budget" if price < 5, "standard" if 5-6, "premium" if > 6
- Filter to only "standard" and "premium" items
- Show the result

**Q3.** What is a Spark partition? How does the number of partitions affect performance?

**Q4.** Write PySpark code to read a CSV file and a Parquet file from S3. Explain the key differences in how Spark reads each format.

**Q5.** Explain what `.groupBy().agg()` does in PySpark. Write code to find the average order value and total order count per restaurant from an `orders` DataFrame.

### Section B: AWS Glue (20 minutes, 5 questions)

**Q6.** What is the Glue Data Catalog? How does it relate to crawlers, databases, and tables?

**Q7.** A Glue crawler scans `s3://my-data-lake/raw/sales/` and finds CSV files with columns: `date,product_id,quantity,price`. What happens in the Data Catalog? Show the table definition it creates.

**Q8.** Write the AWS CLI commands to:
1. Create a Glue database called `analytics_db`
2. Create a crawler that scans `s3://my-bucket/raw/` using role `GlueServiceRole`
3. Run the crawler
4. List all tables in `analytics_db` after the crawler completes

**Q9.** What is a Glue Job bookmark? When would you enable it? Write the argument needed to enable bookmarks.

**Q10.** Explain the difference between a Glue "Spark ETL job" and a "Python Shell job". When would you use each?

### Section C: AWS Athena (15 minutes, 5 questions)

**Q11.** What data formats does Athena support? Which format is recommended and why?

**Q12.** Your Athena query scans 2.3 TB of data. Calculate the cost. How would you reduce this cost?

**Q13.** Write SQL to create an external table in Athena that points to partitioned Parquet data in S3 with the structure:
```
s3://data-lake/processed/orders/year=2026/month=06/day=30/
```

**Q14.** Explain what `MSCK REPAIR TABLE` does. When do you need to run it?

**Q15.** Write an Athena query that uses partition pruning to efficiently query data from the last 7 days of June 2026. The table is partitioned by `year`, `month`, and `day`.

### Section D: Serverless Pipeline Design (20 minutes, 5 questions)

**Q16.** Draw the architecture (in text/ASCII) for a serverless pipeline that:
- Ingests daily CSV files from a partner API
- Transforms them to Parquet with data quality checks
- Makes them queryable for business analysts
- Include all AWS services used and data flow

**Q17.** A company has 3 data sources: PostgreSQL database, daily CSV drops in S3, and JSON logs from a web app. Design a Glue-based architecture to unify all three into a single analytics layer.

**Q18.** Write a complete Glue PySpark script (imports, read, transform, write) that:
- Reads from Glue Catalog table `raw_payments`
- Filters out null payment_ids
- Converts `amount` from string to double
- Adds a `payment_year` column extracted from `payment_date`
- Writes as partitioned Parquet to `s3://data-lake/processed/payments/`

**Q19.** Your Glue job failed with `AnalysisException: Unable to infer schema`. What are the possible causes and how do you fix each?

**Q20.** Explain the 3-zone data lake pattern (raw, processed, analytics). For each zone:
- What format is the data in?
- Who/what writes to it?
- Who/what reads from it?
- How is it organized?

---

## 🌤️ Afternoon Block (2 hours): Answers & Weak Spot Analysis

### Scoring Guide

| Section | Questions | Points Each | Total |
|---------|-----------|-------------|-------|
| A: PySpark | 5 | 6 | 30 |
| B: Glue | 5 | 7 | 35 |
| C: Athena | 5 | 5 | 25 |
| D: Pipeline Design | 5 | 2 | 10 |
| **Total** | **20** | | **100** |

**Grading:**
- 90-100: 🌟 Excellent — ready for portfolio day
- 75-89: ✅ Good — minor weak spots to review
- 60-74: ⚠️ OK — spend extra time on weak areas tonight
- Below 60: 🔴 Need more review — revisit the relevant days

---

### ✅ Full Answers

<details>
<summary>🔑 Section A: PySpark Fundamentals</summary>

**A1:**
- **Transformations** (`.filter()`, `.select()`, `.withColumn()`, `.groupBy()`) are lazy — they build an execution plan but don't compute anything yet. Think of it as a recipe.
- **Actions** (`.count()`, `.show()`, `.collect()`, `.write()`) trigger actual computation. Think of it as cooking the recipe.
- **Why lazy?** Spark optimizes the entire chain of transformations before executing. If you `.filter().select().count()`, Spark can push the filter down to the data source (skip reading filtered-out data) and only read the selected columns. Without lazy evaluation, each step would execute independently, wasting resources.

**A2:**
```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder.appName("Practice").getOrCreate()

# Create DataFrame
data = [(1, "Chicken Rice", 5.50), (2, "Laksa", 4.80), (3, "Nasi Lemak", 6.00)]
df = spark.createDataFrame(data, ["id", "name", "price"])

# Add price_category column
df_categorized = df.withColumn(
    "price_category",
    F.when(F.col("price") < 5, "budget")
     .when(F.col("price") <= 6, "standard")
     .otherwise("premium")
)

# Filter
result = df_categorized.filter(
    F.col("price_category").isin("standard", "premium")
)

result.show()
# +---+------------+-----+--------------+
# | id|        name|price|price_category|
# +---+------------+-----+--------------+
# |  1|Chicken Rice|  5.5|      standard|
# |  3|  Nasi Lemak|  6.0|      standard|
# +---+------------+-----+--------------+
```

**A3:**
A partition is a chunk of data that Spark processes on one executor. Think of it as dividing a big task into smaller subtasks that can run in parallel.
- Too few partitions: not enough parallelism, underutilizing cluster
- Too many partitions: overhead from scheduling and shuffling
- Rule of thumb: 2-4 partitions per CPU core
- `df.repartition(n)` or `df.coalesce(n)` to control partitions

**A4:**
```python
# Read CSV — Spark reads the entire file, infers schema (slow first time)
df_csv = spark.read.csv("s3a://bucket/data.csv", header=True, inferSchema=True)

# Read Parquet — Spark reads only the metadata and needed columns (fast)
df_parquet = spark.read.parquet("s3a://bucket/data.parquet")
```
Key differences:
- CSV: text-based, must read entire file, schema inference required, no compression
- Parquet: columnar, reads only needed columns, schema embedded in file, compressed
- Parquet is ~10-100x faster for analytical queries on large datasets

**A5:**
`.groupBy()` groups rows by one or more columns. `.agg()` applies aggregate functions to each group.
```python
from pyspark.sql import functions as F

result = orders.groupBy("restaurant_name").agg(
    F.round(F.avg("total_amount"), 2).alias("avg_order_value"),
    F.count("*").alias("total_orders")
)

result.orderBy(F.desc("total_orders")).show()
```

</details>

<details>
<summary>🔑 Section B: AWS Glue</summary>

**B1 (Q6):**
The Glue Data Catalog is a **central metadata repository** — it stores table definitions (schema, location, format) for your data in S3. Think of it as a "card catalog" in a library — it tells you what data exists and where to find it, without actually storing the data.

Structure:
- **Database** → logical grouping (like `makanexpress_db`)
- **Table** → one dataset (like `raw_orders`) with schema, location, format
- **Crawler** → scans S3 and creates/updates table definitions automatically

Without the Catalog, Athena and Glue jobs wouldn't know what tables exist or what columns they have.

**B2 (Q7):**
The crawler creates a table in the specified Glue database with:
- **Table name:** derived from the S3 folder name (e.g., `sales`)
- **Columns:** `date` (string), `product_id` (string), `quantity` (int), `price` (double)
- **Location:** `s3://my-data-lake/raw/sales/`
- **Format:** CSV
- **SerDe:** OpenCSVSerDe (serializer/deserializer for CSV)
- **Classification:** csv

**B3 (Q8):**
```bash
# 1. Create database
aws glue create-database \
    --database-input '{"Name": "analytics_db"}'

# 2. Create crawler
aws glue create-crawler \
    --name analytics-raw-crawler \
    --role "arn:aws:iam::ACCOUNT:role/GlueServiceRole" \
    --database-name analytics_db \
    --targets '{"S3Targets": [{"Path": "s3://my-bucket/raw/"}]}'

# 3. Run the crawler
aws glue start-crawler --name analytics-raw-crawler

# 4. Wait, then list tables
aws glue get-tables --database-name analytics_db \
    --query 'TableList[].Name'
```

**B4 (Q9):**
A Glue Job Bookmark records which data has already been processed. When you re-run the job, it only processes **new or changed data** since the last run.

Enable it by adding:
```
"--job-bookmark-option": "job-bookmark-enable"
```
in the job's default arguments.

**When to enable:** Incremental ETL jobs that run on a schedule (hourly/daily) processing new files. Without bookmarks, every run reprocesses ALL data.

**B5 (Q10):**
- **Spark ETL job:** Runs Apache Spark (distributed). Use for: large-scale data transformation (GBs-TBs), joins, aggregations, complex transformations. Minimum 2 DPUs, costs $0.44/DPU-hour.
- **Python Shell job:** Runs a single Python process (no Spark). Use for: small tasks, API calls, file operations, orchestration scripts. Cheaper ($0.044/DPU-hour), limited to single node.

Choose Spark for data transformation, Python Shell for lightweight tasks like triggering APIs or small file operations.

</details>

<details>
<summary>🔑 Section C: AWS Athena</summary>

**C1 (Q11):**
Supported formats: CSV, JSON, ORC, Parquet, Avro
**Recommended: Parquet** because:
- Columnar storage → read only needed columns
- Built-in compression (Snappy, GZIP) → smaller files
- Schema embedded → no separate schema management
- Type-safe → no string/number ambiguity
- Athena charges by data scanned → Parquet = less scanned = cheaper

**C2 (Q12):**
Cost: 2.3 TB × $5/TB = 2.3 × $5 = **$11.50** for one query.

Reduce cost by:
1. Convert to Parquet (reduces data ~80%)
2. Partition by date → `WHERE year=2026 AND month=6` scans only that partition
3. Select only needed columns (avoid `SELECT *`)
4. Use columnar compression (Snappy)
5. Optimize file sizes (128MB-1GB per file)

**C3 (Q13):**
```sql
CREATE EXTERNAL TABLE processed_orders (
    order_id STRING,
    customer_id STRING,
    restaurant_id STRING,
    total_amount DOUBLE,
    delivery_fee DOUBLE,
    status STRING,
    payment_method STRING
)
PARTITIONED BY (year INT, month INT, day INT)
STORED AS PARQUET
LOCATION 's3://data-lake/processed/orders/';

-- Then load partitions
MSCK REPAIR TABLE processed_orders;
```

**C4 (Q14):**
`MSCK REPAIR TABLE` scans the S3 location for partition folders and adds any new partitions to the Glue Data Catalog. You need to run it after:
- Adding new date partitions (new day's data)
- Creating a new partitioned table
- After Glue ETL writes new partition directories

Without it, Athena doesn't know new partitions exist and won't query them.

**C5 (Q15):**
```sql
SELECT *
FROM processed_orders
WHERE year = 2026 
  AND month = 6 
  AND day BETWEEN 24 AND 30;
```
This triggers partition pruning — Athena only scans the folders for days 24-30, not the entire dataset.

</details>

<details>
<summary>🔑 Section D: Serverless Pipeline Design</summary>

**D1 (Q16):**
```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Partner  │    │    S3        │    │   Glue       │    │   Athena     │
│ API      │───▶│   Data Lake  │───▶│   ETL Job    │───▶│   Queries    │
│          │    │              │    │              │    │              │
│ (Python  │    │ raw/ (CSV)   │    │ Crawler:     │    │ Analysts run │
│ script   │    │ processed/   │    │  discover    │    │ SQL queries  │
│ pulls    │    │  (Parquet)   │    │ Job:         │    │ on processed │
│ daily)   │    │ analytics/   │    │  CSV→Parquet │    │ data         │
└──────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                        │                                       │
                   Lifecycle                              Results in S3
                   (archive old)                           Dashboards
```

Services used:
- **Lambda or EC2** (or local script): pulls data from partner API daily
- **S3**: stores raw CSV + processed Parquet
- **Glue Crawler**: discovers schema
- **Glue Job**: transforms CSV → Parquet (PySpark)
- **Athena**: SQL interface for analysts

**D2 (Q17):**
```
Three sources → One analytics layer:

PostgreSQL ──▶ Glue JDBC Connection ──▶ Glue Job (extract) ──┐
                                                              │
CSV in S3 ────▶ Glue Crawler ─────────▶ Glue Job (transform)─┼─▶ S3 Unified
                                                              │   Zone
JSON logs ────▶ Glue Crawler ─────────▶ Glue Job (transform)─┘
                                            │
                                     All write to:
                                     s3://lake/processed/
                                     as Partitioned Parquet
                                            │
                                     Glue Crawler scans
                                     processed/
                                            │
                                     Athena queries
                                     unified tables
```

Key design decisions:
- Each source gets its own crawler and ETL job (independent failure)
- All outputs in same Parquet format with consistent column naming
- Single processed zone = single source of truth for analytics

**D3 (Q18):**
```python
import sys
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

# Read from Glue Catalog
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="makanexpress_db",
    table_name="raw_payments",
    transformation_ctx="datasource"
)
df = datasource.toDF()

# Transform
df_clean = df \
    .filter(F.col("payment_id").isNotNull()) \
    .withColumn("amount", F.col("amount").cast("double")) \
    .withColumn("payment_year", F.year(F.col("payment_date")))

# Write partitioned Parquet
dynamic_out = DynamicFrame.fromDF(df_clean, glueContext, "cleaned")
glueContext.write_dynamic_frame.from_options(
    frame=dynamic_out,
    connection_type="s3",
    connection_options={
        "path": "s3://data-lake/processed/payments/",
        "partitionKeys": ["payment_year"]
    },
    format="glueparquet",
    format_options={"compression": "snappy"}
)

job.commit()
```

**D4 (Q19):**
Possible causes and fixes:
1. **Empty S3 path** — no files at the location. Fix: verify data exists with `aws s3 ls`
2. **Empty files** — 0-byte CSV files. Fix: delete empty files, add validation before processing
3. **No headers** — CSV without header row but `header=True` in options. Fix: set header correctly or provide explicit schema
4. **Corrupt files** — malformed CSV/JSON. Fix: inspect files, clean data at source
5. **Wrong path** — typo in S3 bucket/key. Fix: double-check the path in the crawler/job config

**D5 (Q20):**

| Zone | Format | Written by | Read by | Organization |
|------|--------|-----------|---------|-------------|
| **Raw** | CSV, JSON (as-is) | Ingestion scripts, API pulls | Glue ETL jobs, debugging | By source and date |
| **Processed** | Parquet (cleaned, typed) | Glue ETL jobs | Analytics queries, Glue jobs | Partitioned by date |
| **Analytics** | Parquet (aggregated) | Glue jobs / Athena CTAS | Dashboards, reports | By business domain |

- Raw: immutable, never modify, keep as audit trail
- Processed: single source of truth, clean and typed
- Analytics: business-ready aggregations for dashboards

</details>

---

## 🌙 Evening Block (1 hour): Targeted Review

### Weak Spot Guide

| If you scored < 70% in... | Review these days |
|---------------------------|-------------------|
| Section A (PySpark) | Day 43 — practice DataFrame operations locally |
| Section B (Glue) | Day 44 + Day 46 — re-read crawler and job sections |
| Section C (Athena) | Day 45 — practice SQL queries, partitioning |
| Section D (Pipeline Design) | Day 47 — re-read architecture, draw your own diagram |

### Targeted Practice Exercises

**If PySpark was weak:**
```python
# Practice: Load a CSV and do these transformations
# 1. Read the CSV into a DataFrame
# 2. Filter rows where status = 'delivered'
# 3. Group by restaurant_name, get count and average total_amount
# 4. Sort by count descending
# 5. Show top 10
```

**If Glue was weak:**
- Re-create a crawler from scratch using the AWS Console (not CLI)
- Write a simple Glue job that reads from one table and writes to another
- Practice checking job status and reading error logs in CloudWatch

**If Athena was weak:**
- Write 5 SQL queries against the MakanExpress data
- Practice with partitioned tables — verify partition pruning with `SHOW PARTITIONS`
- Calculate query costs for different query patterns

**If Pipeline Design was weak:**
- Draw the architecture for a different scenario: "Design a pipeline for a Shopee seller analytics dashboard"
- Write out the data flow step by step
- Identify which AWS service handles each step

### 📝 Today's Checklist

- [ ] Completed all 20 assessment questions honestly (timed)
- [ ] Scored myself using the grading scale
- [ ] Identified weak spots (sections scoring < 70%)
- [ ] Completed targeted practice exercises for weak areas
- [ ] Ready for tomorrow's portfolio project (Day 49)

---

*Day 48 complete! Tomorrow: The big portfolio day — build a complete serverless pipeline for MakanExpress!* 🏗️🚀
