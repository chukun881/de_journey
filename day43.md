# 📅 Day 43 — Thursday, 26 June 2026
# 🔥 PySpark Fundamentals: DataFrames & Basic Operations

---

## 🎯 Today's Goal

PySpark is the Python API for Apache Spark — the engine that powers AWS Glue, Databricks, and most big data pipelines. Today you'll learn PySpark **locally** (no AWS needed, just `pip install pyspark`) so that when you hit AWS Glue later this week, you already understand the language.

**Why PySpark matters for a Data Engineer:**
- AWS Glue runs PySpark under the hood
- Companies like Grab, Shopee, and Sea Group process petabytes with Spark
- Interview question: "How would you process a 500GB CSV file?" → PySpark
- It's the bridge between Pandas (too small) and full distributed systems (too complex)

> **⏱️ Time budget:** Morning (2h) + Afternoon (2h) + Evening (1h) = 5 hours
> **💰 Cost:** $0 — everything runs locally on your laptop

---

## 🧠 Concept Map: Where PySpark Fits

```
Data Size →    Small (<1GB)     Medium (1-100GB)    Large (100GB+)
Tool →         Pandas            PySpark Local       PySpark Cluster
                                                 (AWS Glue / EMR)

You are here → [Pandas ✅]  →  [PySpark Local ← TODAY]  →  [Glue ← Day 44-46]
```

**Key insight:** PySpark DataFrames work almost like Pandas DataFrames — but they can scale to petabytes across hundreds of machines. The API is similar, but the *execution model* is completely different.

---

## ☀️ Morning Block (2 hours): Setup + DataFrame Basics

### Step 1: Install PySpark (10 min)

```bash
# PySpark needs Java (Spark runs on JVM)
# Check if Java is installed
java -version

# If not installed (Ubuntu/Debian):
sudo apt update && sudo apt install default-jdk -y

# If macOS:
# brew install openjdk

# Install PySpark
pip install pyspark

# Verify
python -c "import pyspark; print(pyspark.__version__)"
# Should print: 3.5.x or similar
```

**Why Java?** Spark is written in Scala, which runs on the JVM. PySpark is a Python wrapper that communicates with the JVM process. When you call `df.filter()` in Python, it translates to Scala Spark operations under the hood.

### Step 2: Your First Spark Session (15 min)

```python
# file: spark_intro.py
from pyspark.sql import SparkSession

# Create a Spark session (local mode = runs on your laptop)
spark = SparkSession.builder \
    .appName("MakanExpress-Day43") \
    .master("local[*]") \
    .getOrCreate()

# local[*] = use all CPU cores on your machine
# local[2] = use exactly 2 cores
# In production (Glue/EMR), you don't set master — the cluster handles it

print(f"Spark version: {spark.version}")
print(f"Spark UI: http://localhost:4040")  # Opens when Spark runs

# Create a simple DataFrame from a list of tuples
data = [
    ("Changi Point Chicken Rice", "Singapore", 4.5, 120),
    ("Penang Road Famous Laksa", "Penang", 4.8, 200),
    ("KL Hokkien Mee King", "Kuala Lumpur", 4.3, 95),
    ("JB Bak Kut Teh House", "Johor Bahru", 4.6, 150),
    ("Tiong Bahru Hainanese Curry Rice", "Singapore", 4.1, 80),
]

columns = ["restaurant_name", "city", "rating", "monthly_orders"]

df = spark.createDataFrame(data, columns)

# Show the data (like df.head() in Pandas)
df.show()

# Print the schema (like df.dtypes in Pandas)
df.printSchema()

# Stop Spark when done (frees memory)
spark.stop()
```

**Run it:**
```bash
python spark_intro.py
```

**Expected output:**
```
+--------------------+--------------+------+-------------+
|    restaurant_name |          city|rating|monthly_orders|
+--------------------+--------------+------+-------------+
|Changi Point Chic..|     Singapore|   4.5|          120|
|Penang Road Famou..|        Penang|   4.8|          200|
|KL Hokkien Mee King|Kuala Lumpur  |   4.3|           95|
|JB Bak Kut Teh House|Johor Bahru  |   4.6|          150|
|Tiong Bahru Haina..|     Singapore|   4.1|           80|
+--------------------+--------------+------+-------------+

root
 |-- restaurant_name: string (nullable = true)
 |-- city: string (nullable = true)
 |-- rating: double (nullable = true)
 |-- monthly_orders: long (nullable = true)
```

### Step 3: Pandas vs PySpark — Key Differences (20 min)

> **This is the most important concept today.** If you understand *why* PySpark works differently from Pandas, everything else clicks.

```python
# Pandas way:
import pandas as pd
df_pandas = pd.read_csv("orders.csv")
result = df_pandas[df_pandas["city"] == "Singapore"]  # Runs immediately

# PySpark way:
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("compare").master("local[*]").getOrCreate()
df_spark = spark.read.csv("orders.csv", header=True, inferSchema=True)
filtered = df_spark.filter(df_spark.city == "Singapore")  # Does NOT run yet!
result = filtered.collect()  # NOW it runs
```

**The Big Difference: Lazy Evaluation**

| Concept | Pandas | PySpark |
|---------|--------|---------|
| Execution | **Eager** — runs immediately | **Lazy** — builds a plan, runs on action |
| Data location | In RAM (all at once) | On disk/distributed (processed in chunks) |
| `.filter()` | Filters immediately | Records the filter in a plan |
| `.show()`, `.collect()` | N/A | These are **actions** that trigger execution |
| `.groupBy()`, `.select()` | Returns result | These are **transformations** (lazy) |

**Why lazy?** Imagine processing 500GB across 100 machines. If Spark ran each operation immediately, it would read/write 500GB for every step. Instead, Spark builds an **execution plan** (DAG) and optimizes the entire pipeline before running — reading data once, doing all transformations, then writing results.

```
Transformations (lazy — just build the plan):
  .select()  .filter()  .groupBy()  .orderBy()  .join()  .withColumn()
  
Actions (triggers execution — actually runs the plan):
  .show()    .collect()  .count()    .write()    .toPandas()   .take(n)
```

### Exercise 1: Identify Transformations vs Actions

For each operation below, classify it as a **transformation** (lazy) or **action** (triggers execution):

```python
# A
df.filter(df.rating > 4.0)

# B
df.count()

# C
df.select("restaurant_name", "rating")

# D
df.groupBy("city").avg("rating")

# E
df.show(5)

# F
df.orderBy(df.rating.desc())

# G
df.collect()

# H
df.withColumn("is_popular", df.monthly_orders > 100)
```

<details>
<summary>🔑 Answer</summary>

| Letter | Operation | Type | Why |
|--------|-----------|------|-----|
| A | `.filter()` | **Transformation** | Just records "filter by rating > 4.0" |
| B | `.count()` | **Action** | Needs to actually count rows |
| C | `.select()` | **Transformation** | Just picks columns in the plan |
| D | `.groupBy().avg()` | **Transformation** | Builds aggregation plan |
| E | `.show()` | **Action** | Must compute results to display |
| F | `.orderBy()` | **Transformation** | Adds sort to the plan |
| G | `.collect()` | **Action** | Brings all data to driver (Python) |
| H | `.withColumn()` | **Transformation** | Adds column to the plan |

**Rule of thumb:** If it returns a DataFrame → transformation. If it returns actual data or writes → action.

</details>

### Step 4: Loading Data from CSV (20 min)

Let's use the MakanExpress data from your portfolio project:

```python
# file: spark_load_data.py
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MakanExpress-LoadData") \
    .master("local[*]") \
    .getOrCreate()

# Load CSV with schema inference (Spark guesses data types)
orders = spark.read.csv(
    "seed/orders.csv",     # path to your CSV
    header=True,           # first row is column names
    inferSchema=True       # auto-detect data types
)

# Check what we got
orders.printSchema()
orders.show(5)

# Without inferSchema — everything is STRING (bad!)
orders_no_schema = spark.read.csv("seed/orders.csv", header=True, inferSchema=False)
orders_no_schema.printSchema()  # All fields show as "string"

spark.stop()
```

**Explicit Schema (Production Best Practice)**

In production (Glue jobs), you should **define schemas explicitly** — `inferSchema` scans the entire file first, which is slow on large files:

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DoubleType, TimestampType

# Define the schema explicitly
orders_schema = StructType([
    StructField("order_id", IntegerType(), False),
    StructField("customer_id", IntegerType(), True),
    StructField("restaurant_id", IntegerType(), True),
    StructField("order_date", TimestampType(), True),
    StructField("delivery_city", StringType(), True),
    StructField("total_amount", DoubleType(), True),
    StructField("order_status", StringType(), True),
    StructField("delivery_fee", DoubleType(), True),
])

# Load with explicit schema (faster, type-safe)
orders = spark.read.csv(
    "seed/orders.csv",
    header=True,
    schema=orders_schema  # No inferSchema needed!
)

orders.printSchema()
```

**Why does this matter?** On a 500GB file, `inferSchema=True` might take 10 minutes just to figure out types. With an explicit schema, Spark skips that step entirely.

### Step 5: Basic DataFrame Operations (30 min)

```python
# file: spark_operations.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, desc, asc

spark = SparkSession.builder \
    .appName("MakanExpress-Operations") \
    .master("local[*]") \
    .getOrCreate()

# Load our restaurant data
data = [
    (1, "Changi Chicken Rice", "Singapore", "Chinese", 4.5, 120, 8.50),
    (2, "Penang Famous Laksa", "Penang", "Malay", 4.8, 200, 6.00),
    (3, "KL Hokkien Mee King", "Kuala Lumpur", "Chinese", 4.3, 95, 7.50),
    (4, "JB Bak Kut Teh", "Johor Bahru", "Chinese", 4.6, 150, 12.00),
    (5, "Tiong Bahru Curry Rice", "Singapore", "Chinese", 4.1, 80, 5.50),
    (6, "Little India Banana Leaf", "Singapore", "Indian", 4.7, 180, 9.00),
    (7, "Gurney Drive Hokkien Mee", "Penang", "Chinese", 4.4, 110, 7.00),
    (8, "Bangsar Mamak Corner", "Kuala Lumpur", "Malay", 4.2, 220, 5.00),
    (9, "Katong Laksa Legend", "Singapore", "Malay", 4.6, 160, 6.50),
    (10, "Melaka Chicken Rice Ball", "Melaka", "Chinese", 4.9, 75, 8.00),
]

columns = ["id", "name", "city", "cuisine", "rating", "monthly_orders", "avg_price"]
df = spark.createDataFrame(data, columns)

# ========================================
# 1. SELECT — pick columns (like SQL SELECT)
# ========================================
df.select("name", "city", "rating").show()

# Using col() for column references (more flexible)
df.select(col("name"), col("city"), col("rating")).show()

# Rename columns
df.select(
    col("name").alias("restaurant"),
    col("city"),
    col("rating").alias("stars")
).show(3)

# ========================================
# 2. FILTER / WHERE — filter rows (like SQL WHERE)
# ========================================
# Both .filter() and .where() do the same thing
df.filter(col("city") == "Singapore").show()

# Multiple conditions
df.filter((col("city") == "Singapore") & (col("rating") >= 4.5)).show()

# OR condition
df.filter((col("cuisine") == "Chinese") | (col("cuisine") == "Malay")).show()

# NOT condition
df.filter(~(col("city") == "Singapore")).show()

# String contains (like SQL LIKE '%Laksa%')
from pyspark.sql.functions import contains
df.filter(contains(col("name"), "Laksa")).show()

# IN clause
df.filter(col("city").isin("Singapore", "Penang")).show()

# ========================================
# 3. SORT / ORDER BY — sort results
# ========================================
df.orderBy(desc("rating")).show(5)       # Highest rated first
df.sort(asc("avg_price")).show(5)        # Cheapest first
df.orderBy(desc("monthly_orders"), asc("name")).show(5)  # Multi-sort

# ========================================
# 4. WITH COLUMN — add or modify columns
# ========================================
from pyspark.sql.functions import when, lit, round as spark_round

# Add a calculated column
df_with_revenue = df.withColumn(
    "estimated_revenue",
    spark_round(col("monthly_orders") * col("avg_price"), 2)
)
df_with_revenue.select("name", "monthly_orders", "avg_price", "estimated_revenue").show()

# Conditional column (like CASE WHEN in SQL)
df_with_tier = df.withColumn(
    "tier",
    when(col("monthly_orders") >= 150, "🏆 Popular")
    .when(col("monthly_orders") >= 100, "👍 Growing")
    .otherwise("🌱 New")
)
df_with_tier.select("name", "monthly_orders", "tier").show()

# ========================================
# 5. AGGREGATE — groupBy + agg (like SQL GROUP BY)
# ========================================
from pyspark.sql.functions import count, avg, sum, max as spark_max, min as spark_min

# Simple groupBy
df.groupBy("city").count().show()

# Multiple aggregations
df.groupBy("city").agg(
    count("*").alias("num_restaurants"),
    spark_round(avg("rating"), 2).alias("avg_rating"),
    sum("monthly_orders").alias("total_orders"),
    spark_max("rating").alias("best_rating"),
    spark_round(avg("avg_price"), 2).alias("avg_price")
).orderBy(desc("total_orders")).show()

# Aggregate entire DataFrame
df.agg(
    count("*").alias("total_restaurants"),
    spark_round(avg("rating"), 2).alias("overall_avg_rating"),
    sum("monthly_orders").alias("all_orders")
).show()

# ========================================
# 6. DISTINCT — remove duplicates
# ========================================
df.select("city").distinct().show()
df.select("cuisine").distinct().orderBy("cuisine").show()

# Count distinct
from pyspark.sql.functions import countDistinct
df.agg(
    countDistinct("city").alias("num_cities"),
    countDistinct("cuisine").alias("num_cuisines")
).show()

spark.stop()
```

### Exercise 2: Write PySpark Queries

Using the `df` DataFrame above, write PySpark code for each:

**A.** Find all restaurants with rating above 4.5, show only name and city.

**B.** Count how many restaurants are in each cuisine type, sorted descending.

**C.** Add a column "price_category" that labels restaurants as "Budget" (avg_price < 7), "Mid" (7-9), "Premium" (> 9).

**D.** Find the restaurant with the highest monthly orders in each city.

**E.** Calculate the total estimated monthly revenue per city (monthly_orders × avg_price), rounded to 2 decimal places.

<details>
<summary>🔑 Answer</summary>

```python
# A: High-rated restaurants
df.filter(col("rating") > 4.5).select("name", "city").show()

# B: Restaurant count by cuisine
df.groupBy("cuisine").count().orderBy(desc("count")).show()

# C: Price category column
df_price = df.withColumn(
    "price_category",
    when(col("avg_price") < 7, "Budget")
    .when(col("avg_price") <= 9, "Mid")
    .otherwise("Premium")
)
df_price.select("name", "avg_price", "price_category").show()

# D: Top restaurant per city by orders
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number

window_spec = Window.partitionBy("city").orderBy(desc("monthly_orders"))
df.withColumn("rank", row_number().over(window_spec)) \
  .filter(col("rank") == 1) \
  .select("city", "name", "monthly_orders") \
  .show()

# E: Revenue per city
df.withColumn("revenue", spark_round(col("monthly_orders") * col("avg_price"), 2)) \
  .groupBy("city") \
  .agg(spark_round(sum("revenue"), 2).alias("total_monthly_revenue")) \
  .orderBy(desc("total_monthly_revenue")) \
  .show()
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Joins, SQL, and Advanced Operations

### Step 6: Joins in PySpark (30 min)

Joins work just like SQL — but on DataFrames that could be terabytes:

```python
# file: spark_joins.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, count, avg, sum, spark_round

spark = SparkSession.builder \
    .appName("MakanExpress-Joins") \
    .master("local[*]") \
    .getOrCreate()

# Restaurants table
restaurants = spark.createDataFrame([
    (1, "Changi Chicken Rice", "Singapore", "Chinese"),
    (2, "Penang Famous Laksa", "Penang", "Malay"),
    (3, "KL Hokkien Mee King", "Kuala Lumpur", "Chinese"),
    (4, "JB Bak Kut Teh", "Johor Bahru", "Chinese"),
    (5, "Tiong Bahru Curry Rice", "Singapore", "Chinese"),
], ["restaurant_id", "name", "city", "cuisine"])

# Orders table
orders = spark.createDataFrame([
    (101, 1, "2026-06-20", 25.50, "delivered"),
    (102, 1, "2026-06-20", 18.00, "delivered"),
    (103, 2, "2026-06-20", 12.00, "delivered"),
    (104, 3, "2026-06-21", 30.00, "delivered"),
    (105, 2, "2026-06-21", 15.00, "cancelled"),
    (106, 1, "2026-06-21", 22.00, "delivered"),
    (107, 5, "2026-06-21", 8.50, "delivered"),
    (108, 99, "2026-06-21", 10.00, "delivered"),  # restaurant_id=99 doesn't exist!
], ["order_id", "restaurant_id", "order_date", "amount", "status"])

# --- INNER JOIN (most common — only matching rows) ---
inner = orders.join(restaurants, "restaurant_id", "inner")
inner.show()
# Notice: order 108 (restaurant_id=99) is MISSING — no match in restaurants

# --- LEFT JOIN (keep all orders, even if no restaurant match) ---
left = orders.join(restaurants, "restaurant_id", "left")
left.show()
# Order 108 appears with NULL restaurant info — useful for finding orphans

# --- RIGHT JOIN (keep all restaurants, even if no orders) ---
right = orders.join(restaurants, "restaurant_id", "right")
right.show()
# JB Bak Kut Teh (id=4) appears with NULL order info — no orders yet

# --- OUTER JOIN (keep everything) ---
full = orders.join(restaurants, "restaurant_id", "full")
full.show()

# --- Join on multiple columns ---
# When column names differ:
# orders.join(restaurants, orders.rest_id == restaurants.id, "inner")

# --- Multiple joins (like SQL) ---
customers = spark.createDataFrame([
    (1, "Ah Hock", "Singapore"),
    (2, "Siti", "Penang"),
], ["customer_id", "customer_name", "customer_city"])

# You need a customer_id in orders for this to work
# orders.join(restaurants, "restaurant_id").join(customers, "customer_id")

# --- Join with aggregation ---
restaurant_revenue = orders \
    .filter(col("status") == "delivered") \
    .groupBy("restaurant_id") \
    .agg(
        count("*").alias("order_count"),
        spark_round(sum("amount"), 2).alias("total_revenue"),
        spark_round(avg("amount"), 2).alias("avg_order_value")
    )

# Join back to get restaurant names
restaurant_revenue.join(restaurants, "restaurant_id") \
    .select("name", "order_count", "total_revenue", "avg_order_value") \
    .orderBy(col("total_revenue").desc()) \
    .show()

spark.stop()
```

**Common Join Mistakes (Interview Red Flags):**

```python
# ❌ BAD: Joining without specifying the join column
# This creates a CROSS JOIN (every row × every row) — extremely expensive!
orders.join(restaurants)  # DON'T DO THIS

# ❌ BAD: Joining on different-named columns without equality
# orders.join(restaurants, orders.rest_id == restaurants.restaurant_id)
# This works but Spark may not optimize as well as single-column joins

# ✅ GOOD: Join on shared column name
orders.join(restaurants, "restaurant_id")

# ✅ GOOD: Explicit join type
orders.join(restaurants, "restaurant_id", "inner")
```

### Step 7: Spark SQL — Write SQL on DataFrames (20 min)

If you're more comfortable with SQL (you should be after Week 1!), PySpark lets you write raw SQL:

```python
# file: spark_sql.py
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MakanExpress-SQL") \
    .master("local[*]") \
    .getOrCreate()

# Create DataFrames
restaurants = spark.createDataFrame([
    (1, "Changi Chicken Rice", "Singapore", "Chinese", 4.5, 120),
    (2, "Penang Famous Laksa", "Penang", "Malay", 4.8, 200),
    (3, "KL Hokkien Mee King", "Kuala Lumpur", "Chinese", 4.3, 95),
    (4, "JB Bak Kut Teh", "Johor Bahru", "Chinese", 4.6, 150),
    (5, "Tiong Bahru Curry Rice", "Singapore", "Chinese", 4.1, 80),
    (6, "Little India Banana Leaf", "Singapore", "Indian", 4.7, 180),
], ["id", "name", "city", "cuisine", "rating", "monthly_orders"])

orders = spark.createDataFrame([
    (101, 1, "2026-06-20", 25.50),
    (102, 1, "2026-06-20", 18.00),
    (103, 2, "2026-06-20", 12.00),
    (104, 3, "2026-06-21", 30.00),
    (105, 2, "2026-06-21", 15.00),
    (106, 1, "2026-06-21", 22.00),
], ["order_id", "restaurant_id", "order_date", "amount"])

# Register as temporary SQL views
restaurants.createOrReplaceTempView("restaurants")
orders.createOrReplaceTempView("orders")

# Now write SQL!
# Simple query
spark.sql("""
    SELECT name, city, rating
    FROM restaurants
    WHERE rating >= 4.5
    ORDER BY rating DESC
""").show()

# Aggregation
spark.sql("""
    SELECT 
        city,
        COUNT(*) as num_restaurants,
        ROUND(AVG(rating), 2) as avg_rating,
        SUM(monthly_orders) as total_orders
    FROM restaurants
    GROUP BY city
    ORDER BY total_orders DESC
""").show()

# Join query
spark.sql("""
    SELECT 
        r.name,
        r.city,
        COUNT(o.order_id) as order_count,
        ROUND(SUM(o.amount), 2) as total_revenue
    FROM restaurants r
    INNER JOIN orders o ON r.id = o.restaurant_id
    GROUP BY r.name, r.city
    ORDER BY total_revenue DESC
""").show()

# Window function! (Yes, SQL works here too)
spark.sql("""
    SELECT 
        name,
        city,
        rating,
        RANK() OVER (ORDER BY rating DESC) as rating_rank,
        RANK() OVER (PARTITION BY city ORDER BY rating DESC) as city_rank
    FROM restaurants
""").show()

# The result of spark.sql() is a DataFrame — you can chain operations
result_df = spark.sql("""
    SELECT city, AVG(rating) as avg_rating
    FROM restaurants
    GROUP BY city
""")
result_df.filter(col("avg_rating") > 4.3).show()

spark.stop()
```

**When to use DataFrame API vs Spark SQL:**
- **DataFrame API (`.filter()`, `.select()`):** Better for programmatic pipelines, dynamic conditions, complex logic
- **Spark SQL (`spark.sql()`):** Better for ad-hoc analysis, familiar SQL syntax, Athena/Glue uses SQL heavily
- **In AWS Glue:** You'll use both — PySpark for transformations, SQL for queries
- **In interviews:** Show you know both

### Step 8: Reading and Writing Files (20 min)

```python
# file: spark_file_io.py
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MakanExpress-FileIO") \
    .master("local[*]") \
    .getOrCreate()

# ========================================
# READING FILES
# ========================================

# CSV
df_csv = spark.read.csv("data/orders.csv", header=True, inferSchema=True)

# JSON
df_json = spark.read.json("data/orders.json")

# Parquet (the BEST format for Spark — columnar, compressed, fast)
df_parquet = spark.read.parquet("data/orders.parquet")

# Read all files in a folder (Spark reads the whole directory!)
df_all = spark.read.csv("data/orders/", header=True, inferSchema=True)
# This reads ALL CSV files in the folder — perfect for daily partitions

# ========================================
# WRITING FILES
# ========================================

# Write as CSV
df_csv.write.csv("output/orders_csv", header=True, mode="overwrite")
# Creates a FOLDER with part files (not a single file!)
# output/orders_csv/part-00000-xxx.csv
# output/orders_csv/part-00001-xxx.csv
# This is because Spark writes in parallel across partitions

# Write as Parquet (recommended!)
df_csv.write.parquet("output/orders_parquet", mode="overwrite")

# Write as JSON
df_csv.write.json("output/orders_json", mode="overwrite")

# Coalesce to single file (for small outputs only!)
df_csv.coalesce(1).write.csv("output/orders_single.csv", header=True, mode="overwrite")
# ⚠️ coalesce(1) defeats parallelism — only use for small files!
# For large files, let Spark write multiple parts

# Write with partitioning (important for S3/Athena!)
df_csv.write.partitionBy("city").parquet("output/orders_by_city", mode="overwrite")
# Creates: output/orders_by_city/city=Singapore/part-00000.parquet
#          output/orders_by_city/city=Penang/part-00000.parquet
# Athena LOVES partitioned data — much faster + cheaper queries!

spark.stop()
```

**Why Parquet > CSV for Spark:**
1. **Smaller:** 50-80% smaller files (columnar compression)
2. **Faster:** Only reads columns you need (column pruning)
3. **Typed:** Schema is embedded in the file (no guessing)
4. **Splittable:** Spark can read different parts in parallel
5. **Industry standard:** AWS Glue, Athena, Redshift, BigQuery all prefer Parquet

### Step 9: Handling Missing Data & Data Cleaning (20 min)

```python
# file: spark_cleaning.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, when, lit, isnull, count as spark_count

spark = SparkSession.builder \
    .appName("MakanExpress-Cleaning") \
    .master("local[*]") \
    .getOrCreate()

# Messy data (common in real data engineering!)
messy = spark.createDataFrame([
    (1, "Changi Chicken Rice", "Singapore", 4.5, 120, 8.50),
    (2, None, "Penang", 4.8, 200, 6.00),           # NULL name
    (3, "KL Hokkien Mee", "Kuala Lumpur", None, 95, 7.50),  # NULL rating
    (4, "JB Bak Kut Teh", None, 4.6, 150, None),    # NULL city + NULL price
    (5, "Tiong Bahru Curry", "Singapore", 4.1, None, 5.50),  # NULL orders
    (6, "", "Singapore", 3.5, 50, 4.00),             # Empty string name
], ["id", "name", "city", "rating", "monthly_orders", "avg_price"])

messy.show()

# ========================================
# Detecting NULLs
# ========================================

# Count NULLs per column
messy.select([
    spark_count(when(isnull(c), c)).alias(c) for c in messy.columns
]).show()

# Filter rows with any NULL
messy.filter(
    col("name").isNull() | col("city").isNull() | col("rating").isNull()
).show()

# ========================================
# Handling NULLs — Strategies
# ========================================

# Strategy 1: Drop rows with NULLs
messy.na.drop().show()              # Drop if ANY column is NULL
messy.na.drop(subset=["name"]).show()  # Drop only if name is NULL
messy.na.drop(thresh=4).show()      # Keep rows with at least 4 non-NULL values

# Strategy 2: Fill NULLs with defaults
messy.na.fill({
    "name": "Unknown Restaurant",
    "city": "Unknown",
    "rating": 0.0,
    "monthly_orders": 0,
    "avg_price": 0.0
}).show()

# Strategy 3: Fill with computed values (like mean)
from pyspark.sql.functions import mean as spark_mean

avg_rating = messy.agg(spark_mean(col("rating"))).collect()[0][0]
print(f"Average rating: {avg_rating}")

messy_filled = messy.na.fill({"rating": avg_rating})
messy_filled.show()

# Strategy 4: Replace empty strings with NULL, then handle
from pyspark.sql.functions import trim, when, lit, null as spark_null

cleaned = messy.withColumn(
    "name",
    when(trim(col("name")) == "", None).otherwise(col("name"))
)
cleaned.show()

# Strategy 5: Drop duplicate rows
df_with_dupes = spark.createDataFrame([
    (1, "Changi Chicken Rice", 4.5),
    (1, "Changi Chicken Rice", 4.5),  # exact duplicate
    (2, "Penang Laksa", 4.8),
], ["id", "name", "rating"])

df_with_dupes.dropDuplicates().show()            # Remove exact duplicates
df_with_dupes.dropDuplicates(["id"]).show()      # Keep first row per id

spark.stop()
```

### Exercise 3: Data Cleaning Challenge

You receive a messy dataset of GrabFood orders:

```python
grab_orders = spark.createDataFrame([
    (101, "user_001", "restaurant_005", "2026-06-20", 25.50, "delivered"),
    (102, "user_002", "restaurant_003", "2026-06-20", None, "delivered"),     # missing amount
    (103, "user_001", "restaurant_005", None, 18.00, "delivered"),            # missing date
    (104, "user_003", None, "2026-06-21", 30.00, "delivered"),                # missing restaurant
    (105, "user_002", "restaurant_001", "2026-06-21", 12.00, "cancelled"),
    (106, None, "restaurant_003", "2026-06-21", 22.00, "delivered"),          # missing user
    (107, "user_004", "restaurant_002", "2026-06-22", 15.00, None),           # missing status
    (101, "user_001", "restaurant_005", "2026-06-20", 25.50, "delivered"),    # DUPLICATE
], ["order_id", "user_id", "restaurant_id", "order_date", "amount", "status"])
```

**Tasks:**
1. Remove exact duplicates
2. Drop rows where `restaurant_id` or `user_id` is NULL (can't use orders without these)
3. Fill missing `amount` with 0 (will be marked as cancelled/unknown)
4. Fill missing `status` with "unknown"
5. Count how many rows remain, and show the cleaned data

<details>
<summary>🔑 Answer</summary>

```python
# Step 1: Remove exact duplicates
deduped = grab_orders.dropDuplicates()

# Step 2: Drop rows missing critical fields
critical_clean = deduped.na.drop(subset=["restaurant_id", "user_id"])

# Step 3: Fill missing amounts with 0
filled_amount = critical_clean.na.fill({"amount": 0.0})

# Step 4: Fill missing status
cleaned = filled_amount.na.fill({"status": "unknown"})

# Step 5: Results
print(f"Rows remaining: {cleaned.count()}")
cleaned.orderBy("order_id").show()
```

After cleaning, order_id 103 (missing date but has user/restaurant) stays. Order 104 (missing restaurant) gets dropped. The duplicate 101 is removed.

</details>

### Step 10: Window Functions in PySpark (20 min)

Yes — window functions from SQL Day 4 work in PySpark too!

```python
# file: spark_windows.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, desc, rank, dense_rank, row_number, lag, lead, sum as spark_sum, avg as spark_avg, count as spark_count
from pyspark.sql.window import Window

spark = SparkSession.builder \
    .appName("MakanExpress-Windows") \
    .master("local[*]") \
    .getOrCreate()

# Order data with dates
orders = spark.createDataFrame([
    (101, 1, "2026-06-20", 25.50),
    (102, 1, "2026-06-20", 18.00),
    (103, 2, "2026-06-20", 12.00),
    (104, 1, "2026-06-21", 30.00),
    (105, 2, "2026-06-21", 15.00),
    (106, 1, "2026-06-22", 22.00),
    (107, 2, "2026-06-22", 28.00),
    (108, 1, "2026-06-22", 16.00),
], ["order_id", "restaurant_id", "order_date", "amount"])

# ========================================
# Window specification
# ========================================
# PARTITION BY restaurant_id, ORDER BY order_date
daily_window = Window.partitionBy("restaurant_id").orderBy("order_date")

# Running total per restaurant
running_total = orders.withColumn(
    "running_total",
    spark_sum("amount").over(daily_window.rowsBetween(
        Window.unboundedPreceding, Window.currentRow
    ))
)

running_total.orderBy("restaurant_id", "order_date").show()

# Ranking per restaurant
ranked = orders.withColumn("row_num", row_number().over(daily_window)) \
               .withColumn("rank", rank().over(daily_window)) \
               .withColumn("dense_rank", dense_rank().over(daily_window))
ranked.orderBy("restaurant_id", "order_date").show()

# LAG and LEAD — compare with previous/next order
compared = orders.withColumn(
    "prev_order_amount", lag("amount", 1).over(daily_window)
).withColumn(
    "next_order_amount", lead("amount", 1).over(daily_window)
).withColumn(
    "amount_change",
    col("amount") - lag("amount", 1).over(daily_window)
)
compared.orderBy("restaurant_id", "order_date").show()

# Rolling average (3-day window)
rolling_window = Window.partitionBy("restaurant_id") \
    .orderBy("order_date") \
    .rowsBetween(-1, 0)  # current row + 1 previous row

rolling_avg = orders.withColumn(
    "2_order_avg",
    spark_avg("amount").over(rolling_window)
)
rolling_avg.orderBy("restaurant_id", "order_date").show()

# ========================================
# Window vs GroupBy — Key difference
# ========================================
# GroupBy: one row per group (collapses rows)
# Window: keeps all rows, adds computed columns

# GroupBy — one row per restaurant
orders.groupBy("restaurant_id").sum("amount").show()

# Window — all rows kept, with total per restaurant
total_window = Window.partitionBy("restaurant_id")
orders.withColumn("restaurant_total", spark_sum("amount").over(total_window)).show()

spark.stop()
```

### Exercise 4: Window Function Practice

Using the `orders` DataFrame above, write PySpark code to:

**A.** Find the highest-value order for each restaurant (show all columns + the max amount per restaurant).

**B.** Add a column showing what percentage each order is of the restaurant's total revenue.

**C.** For each restaurant, find the order amount that increased the most compared to the previous order.

<details>
<summary>🔑 Answer</summary>

```python
# A: Highest-value order per restaurant (keep all rows with max shown)
max_window = Window.partitionBy("restaurant_id")
with_max = orders.withColumn("max_amount", spark_sum("amount").over(max_window))
# Actually, use max():
from pyspark.sql.functions import max as spark_max
with_max = orders.withColumn("max_order", spark_max("amount").over(max_window))

# To get ONLY the highest:
from pyspark.sql.functions import row_number
rank_window = Window.partitionBy("restaurant_id").orderBy(desc("amount"))
orders.withColumn("rn", row_number().over(rank_window)) \
     .filter(col("rn") == 1) \
     .drop("rn") \
     .show()

# B: Percentage of restaurant total
total_window = Window.partitionBy("restaurant_id")
orders.withColumn(
    "pct_of_total",
    spark_round(col("amount") / spark_sum("amount").over(total_window) * 100, 2)
).show()

# C: Biggest increase vs previous order
daily_window = Window.partitionBy("restaurant_id").orderBy("order_date")
orders.withColumn(
    "increase",
    col("amount") - lag("amount", 1).over(daily_window)
).orderBy(desc("increase")).show(1)
```

</details>

---

## 🌙 Evening Block (1 hour): UDFs, Performance Basics + Review

### Step 11: User-Defined Functions (UDFs) (20 min)

When built-in functions aren't enough, write your own:

```python
# file: spark_udfs.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import udf, col
from pyspark.sql.types import StringType, DoubleType

spark = SparkSession.builder \
    .appName("MakanExpress-UDFs") \
    .master("local[*]") \
    .getOrCreate()

df = spark.createDataFrame([
    (1, "Changi Chicken Rice", "Singapore", 4.5, 120),
    (2, "Penang Famous Laksa", "Penang", 4.8, 200),
    (3, "KL Hokkien Mee", "Kuala Lumpur", 4.3, 95),
], ["id", "name", "city", "rating", "monthly_orders"])

# ========================================
# UDF: Custom logic
# ========================================

# Example 1: Classify restaurant size
@udf(StringType())
def classify_restaurant(monthly_orders):
    if monthly_orders is None:
        return "Unknown"
    elif monthly_orders >= 150:
        return "🏆 High Volume"
    elif monthly_orders >= 80:
        return "👍 Medium"
    else:
        return "🌱 Small"

df.withColumn("category", classify_restaurant(col("monthly_orders"))).show()

# Example 2: Singapore GST calculation (9% as of 2024)
@udf(DoubleType())
def add_gst(amount):
    if amount is None:
        return None
    return round(amount * 1.09, 2)

df.withColumn("price_with_gst", add_gst(col("monthly_orders").cast("double"))).show()

# ⚠️ UDF Performance Warning
# UDFs are slow because Spark can't optimize them (black box)
# Always prefer built-in functions when possible:
#   ✅ when(col("x") > 5, "big").otherwise("small")  # Fast (Spark native)
#   ❌ @udf ... if x > 5: return "big"                # Slow (Python per row)

spark.stop()
```

### Step 12: Partitioning & Performance Concepts (20 min)

```python
# file: spark_partitions.py
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MakanExpress-Partitions") \
    .master("local[*]") \
    .getOrCreate()

df = spark.createDataFrame([
    (i, f"order_{i}", "Singapore" if i % 3 == 0 else "Penang" if i % 3 == 1 else "KL", 
     10.0 + i, "2026-06-20")
    for i in range(1, 101)
], ["id", "order_name", "city", "amount", "date"])

# Check number of partitions
print(f"Partitions: {df.rdd.getNumPartitions()}")

# Repartition (expensive — full shuffle)
df_repartitioned = df.repartition(4, "city")  # 4 partitions, partitioned by city
print(f"After repartition: {df_repartitioned.rdd.getNumPartitions()}")

# Coalesce (cheap — reduces partitions without full shuffle)
df_coalesced = df.coalesce(2)
print(f"After coalesce: {df_coalesced.rdd.getNumPartitions()}")

# When to repartition:
# - Before writing partitioned output (e.g., partitionBy city)
# - After a filter that removed most rows (skewed partitions)
# - Before a join (if partitioned on join key)

# When to coalesce:
# - Before writing a small output file
# - Reducing partitions before an action (less overhead)

# Cache / Persist — keep data in memory for reuse
from pyspark import StorageLevel
df_cached = df.cache()  # Keep in memory
df_cached.count()  # First action — caches data
df_cached.filter(col("city") == "Singapore").count()  # Uses cached data (faster!)
df_cached.unpersist()  # Free memory when done

# ⚠️ Only cache if you use the DataFrame multiple times!
# Caching once and using once = wasted memory

spark.stop()
```

### Quick Concept: PySpark Execution Model

```
Your Code:
  df.filter().select().groupBy().agg()
       ↓
Spark builds a LOGICAL PLAN (DAG):
  "Read CSV → Filter → Select → GroupBy → Aggregate"
       ↓
Catalyst Optimizer transforms the plan:
  "Push filter before select (less data)"
  "Combine adjacent operations"
       ↓
Physical Plan:
  "Read CSV → Filter + Select (fused) → GroupBy → Aggregate"
       ↓
Executed on partitions (in parallel):
  Partition 1: [rows 1-25] → filter → select → group
  Partition 2: [rows 26-50] → filter → select → group
  Partition 3: [rows 51-75] → filter → select → group
  Partition 4: [rows 76-100] → filter → select → group
       ↓
Results combined → .show() or .collect()
```

---

## 📚 Day 43 Summary

### What You Learned

| Topic | Key Takeaway |
|-------|-------------|
| Spark Architecture | Spark runs on JVM, PySpark is Python wrapper, local vs cluster mode |
| Lazy Evaluation | Transformations build a plan, actions trigger execution |
| DataFrames | Like Pandas but distributed — select, filter, groupBy, agg |
| Spark SQL | Write SQL directly on DataFrames via `spark.sql()` |
| Joins | inner, left, right, full — same as SQL, but on big data |
| File I/O | CSV in, Parquet out (always prefer Parquet) |
| Window Functions | row_number, rank, lag/lead — same as SQL, PySpark syntax |
| UDFs | Custom Python functions — useful but slow, prefer built-ins |
| Partitions | Data split across workers — repartition/coalesce for optimization |
| Caching | Keep data in memory for reuse — only if used multiple times |

### Pandas vs PySpark Cheat Sheet

| Operation | Pandas | PySpark |
|-----------|--------|---------|
| Create | `pd.DataFrame(data)` | `spark.createDataFrame(data)` |
| Read CSV | `pd.read_csv("f.csv")` | `spark.read.csv("f.csv", header=True)` |
| Show | `df.head()` | `df.show()` |
| Schema | `df.dtypes` | `df.printSchema()` |
| Select | `df[["a","b"]]` | `df.select("a","b")` |
| Filter | `df[df.x > 5]` | `df.filter(col("x") > 5)` |
| Sort | `df.sort_values("x")` | `df.orderBy("x")` |
| GroupBy | `df.groupby("x").agg(...)` | `df.groupBy("x").agg(...)` |
| Join | `pd.merge(a, b, on="id")` | `a.join(b, "id")` |
| New col | `df["new"] = df.x * 2` | `df.withColumn("new", col("x") * 2)` |
| Rename | `df.rename(columns={...})` | `df.withColumnRenamed("old", "new")` |
| Drop nulls | `df.dropna()` | `df.na.drop()` |
| Fill nulls | `df.fillna(0)` | `df.na.fill(0)` |
| SQL | N/A | `spark.sql("SELECT ...")` |
| Write | `df.to_csv("f.csv")` | `df.write.csv("path")` |
| Count | `len(df)` | `df.count()` |

---

## ✅ Daily Checklist

- [ ] PySpark installed and working (`pip install pyspark`)
- [ ] Java installed (required for Spark)
- [ ] Completed spark_intro.py — first DataFrame created
- [ ] Understand lazy evaluation vs eager execution
- [ ] Completed all DataFrame operations (select, filter, groupBy)
- [ ] Completed Exercise 1 (transformations vs actions)
- [ ] Completed spark_joins.py — all join types
- [ ] Completed spark_sql.py — SQL on DataFrames
- [ ] Completed spark_cleaning.py — NULL handling
- [ ] Completed Exercise 2 (PySpark queries)
- [ ] Completed Exercise 3 (data cleaning challenge)
- [ ] Completed spark_windows.py — window functions
- [ ] Completed Exercise 4 (window practice)
- [ ] Understand UDFs and when to avoid them
- [ ] Understand partitioning concepts
- [ ] All code files saved in `spark-practice/` folder
- [ ] Ready for tomorrow: AWS Glue (serverless ETL)

---

## 🔖 Quick Reference Card

```
# SETUP
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("name").master("local[*]").getOrCreate()

# READ
df = spark.read.csv("file.csv", header=True, inferSchema=True)
df = spark.read.parquet("file.parquet")

# TRANSFORMATIONS (lazy)
df.select("col1", "col2")
df.filter(col("x") > 5)
df.withColumn("new", col("a") + col("b"))
df.groupBy("x").agg(sum("y"), avg("z"))
df.join(other_df, "key", "inner")
df.orderBy(desc("x"))
df.distinct()
df.na.drop()
df.na.fill({"col": default})

# ACTIONS (trigger execution)
df.show()
df.count()
df.collect()
df.write.parquet("output/", mode="overwrite")

# SQL
df.createOrReplaceTempView("table")
spark.sql("SELECT * FROM table WHERE x > 5").show()

# WINDOW
from pyspark.sql.window import Window
w = Window.partitionBy("col").orderBy("date")
df.withColumn("rank", row_number().over(w))

# STOP
spark.stop()
```

---

*Day 43 complete! Tomorrow: AWS Glue — the serverless ETL service that runs PySpark for you.* 🚀🔥
