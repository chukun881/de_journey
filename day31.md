# 📅 Day 31 — Saturday, 14 June 2026
# 🔗 Airflow + PostgreSQL: Building a Real Data Pipeline DAG

---

## 🎯 Today's Goal

You know the building blocks. Today you'll build a **real end-to-end pipeline** — extracting data from source tables, transforming it, loading into the star schema warehouse, and running quality checks. All orchestrated by Airflow.

**Why this matters:** This is exactly what you'll do in a data engineering job. A complete pipeline from raw data to analytics-ready warehouse.

---

## ☀️ Morning Block (2 hours): Complete ETL DAG

### Concept 1: Pipeline Architecture

```
Source Tables (OLTP)          Airflow DAG               Warehouse (Star Schema)
─────────────────           ──────────────            ──────────────────────
raw_customers ─────┐
raw_restaurants ───┤    ┌─ extract_orders
raw_orders ────────┤    ├─ validate_data           dim_date
raw_order_items ───┤───→├─ load_dim_customer  ────→ dim_customer (SCD2)
raw_payments ──────┤    ├─ load_dim_restaurant ───→ dim_restaurant (SCD2)
                    │    ├─ load_dim_item      ────→ dim_item
                    │    ├─ load_fact          ────→ fact_order_items
                    │    ├─ quality_check
                    │    └─ notify
                    │
                    └─ update_segments ────→ dim_customer.segment
```

### Concept 2: The Complete DAG

Create `dags/makanexpress_warehouse_etl.py`:

```python
"""
MakanExpress Warehouse ETL Pipeline
Loads data from raw (OLTP) tables into star schema warehouse.
Runs daily at 6 AM SGT.
"""

from datetime import datetime, timedelta
import pytz
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.models import Variable

default_args = {
    'owner': 'data-engineering',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
}

CONNECTION_ID = 'makanexpress_db'

with DAG(
    dag_id='makanexpress_warehouse_etl',
    default_args=default_args,
    description='Full ETL: raw → star schema warehouse',
    schedule='0 6 * * *',
    start_date=datetime(2026, 6, 1, tzinfo=pytz.timezone('Asia/Singapore')),
    timezone=pytz.timezone('Asia/Singapore'),
    catchup=False,
    max_active_runs=1,
    tags=['makanexpress', 'etl', 'warehouse'],
) as dag:

    # ─── Task 1: Extract & Validate ───
    def extract_and_validate(**context):
        hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
        start = context['data_interval_start']
        end = context['data_interval_end']
        
        # Count source records
        order_count = hook.get_first("""
            SELECT COUNT(*) FROM raw_orders
            WHERE order_date >= %s AND order_date < %s
        """, parameters=(start, end))[0]
        
        # Validation: no zero days (unless it's a holiday or future)
        if order_count == 0:
            raise ValueError(f"No orders found between {start} and {end}")
        
        # Validation: check for NULLs in required fields
        null_count = hook.get_first("""
            SELECT COUNT(*) FROM raw_orders
            WHERE order_date >= %s AND order_date < %s
            AND (customer_id IS NULL OR restaurant_id IS NULL OR total_amount IS NULL)
        """, parameters=(start, end))[0]
        
        if null_count > 5:  # tolerate a few
            raise ValueError(f"Too many NULL values: {null_count}")
        
        # Validation: no duplicates
        dupe_count = hook.get_first("""
            SELECT COUNT(*) FROM (
                SELECT order_id FROM raw_orders
                WHERE order_date >= %s AND order_date < %s
                GROUP BY order_id HAVING COUNT(*) > 1
            ) d
        """, parameters=(start, end))[0]
        
        if dupe_count > 0:
            raise ValueError(f"Found {dupe_count} duplicate order_ids")
        
        print(f"✅ Validation passed: {order_count} orders, {null_count} NULLs, {dupe_count} dupes")
        context['ti'].xcom_push(key='order_count', value=order_count)
        return order_count

    task_validate = PythonOperator(
        task_id='extract_and_validate',
        python_callable=extract_and_validate,
    )

    # ─── Task 2: Load dim_customer (SCD Type 2) ───
    def load_dim_customer(**context):
        hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
        
        # Step 1: Expire changed records
        expired = hook.get_first("""
            WITH expired AS (
                UPDATE dim_customer d
                SET valid_to = NOW(), is_current = FALSE
                FROM raw_customers s
                WHERE d.customer_id = s.customer_id
                AND d.is_current = TRUE
                AND (
                    d.customer_name IS DISTINCT FROM s.name
                    OR d.city IS DISTINCT FROM s.city
                )
                RETURNING d.customer_key
            )
            SELECT COUNT(*) FROM expired
        """)[0]
        
        # Step 2: Insert new + changed
        hook.run("""
            INSERT INTO dim_customer (customer_id, customer_name, email, phone, city, postal_code,
                                      country, signup_date, customer_segment, valid_from, valid_to, is_current)
            SELECT 
                s.customer_id, s.name, s.email, s.phone, s.city, s.postal_code,
                CASE WHEN s.city IN ('Singapore') THEN 'Singapore' ELSE 'Malaysia' END,
                s.signup_date,
                'New', NOW(), '9999-12-31 23:59:59', TRUE
            FROM raw_customers s
            WHERE NOT EXISTS (
                SELECT 1 FROM dim_customer d 
                WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
                AND d.customer_name = s.name AND d.city = s.city
            )
        """)
        
        print(f"✅ dim_customer: expired {expired} rows, inserted new/changed")

    task_dim_customer = PythonOperator(
        task_id='load_dim_customer',
        python_callable=load_dim_customer,
    )

    # ─── Task 3: Load dim_restaurant (SCD Type 2) ───
    def load_dim_restaurant(**context):
        hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
        
        expired = hook.get_first("""
            WITH expired AS (
                UPDATE dim_restaurant d
                SET valid_to = NOW(), is_current = FALSE
                FROM raw_restaurants s
                WHERE d.restaurant_id = s.restaurant_id
                AND d.is_current = TRUE
                AND (
                    d.restaurant_name IS DISTINCT FROM s.name
                    OR d.cuisine_type IS DISTINCT FROM s.cuisine
                    OR d.rating IS DISTINCT FROM s.rating
                )
                RETURNING d.restaurant_key
            )
            SELECT COUNT(*) FROM expired
        """)[0]
        
        hook.run("""
            INSERT INTO dim_restaurant (restaurant_id, restaurant_name, cuisine_type, cuisine_category,
                                         city, postal_code, rating, is_active, valid_from, valid_to, is_current)
            SELECT 
                s.restaurant_id, s.name, s.cuisine,
                CASE 
                    WHEN s.cuisine IN ('Chinese','Malay','Indian','Thai','Japanese') THEN 'Asian'
                    WHEN s.cuisine IN ('Western','Dessert') THEN 'Western/Other'
                    ELSE 'Fusion'
                END,
                s.city, s.postal_code, s.rating, s.is_active,
                NOW(), '9999-12-31 23:59:59', TRUE
            FROM raw_restaurants s
            WHERE NOT EXISTS (
                SELECT 1 FROM dim_restaurant d 
                WHERE d.restaurant_id = s.restaurant_id AND d.is_current = TRUE
                AND d.restaurant_name = s.name AND d.cuisine_type = s.cuisine AND d.rating = s.rating
            )
        """)
        
        print(f"✅ dim_restaurant: expired {expired} rows, inserted new/changed")

    task_dim_restaurant = PythonOperator(
        task_id='load_dim_restaurant',
        python_callable=load_dim_restaurant,
    )

    # ─── Task 4: Load dim_item (SCD Type 1 — simple) ───
    task_dim_item = PostgresOperator(
        task_id='load_dim_item',
        postgres_conn_id=CONNECTION_ID,
        sql="""
            INSERT INTO dim_item (item_id, restaurant_id, dish_name, category, price, is_vegetarian, spice_level)
            SELECT item_id, restaurant_id, dish_name, category, price, is_vegetarian, spice_level
            FROM raw_menu_items s
            WHERE NOT EXISTS (
                SELECT 1 FROM dim_item d WHERE d.item_id = s.item_id
            );
            
            UPDATE dim_item d
            SET price = s.price, dish_name = s.dish_name
            FROM raw_menu_items s
            WHERE d.item_id = s.item_id
            AND (d.price IS DISTINCT FROM s.price OR d.dish_name IS DISTINCT FROM s.dish_name);
        """,
    )

    # ─── Task 5: Load fact_order_items ───
    task_load_fact = PostgresOperator(
        task_id='load_fact',
        postgres_conn_id=CONNECTION_ID,
        sql="""
            INSERT INTO fact_order_items (
                order_number, date_key, customer_key, restaurant_key,
                item_key, payment_method_key,
                quantity, unit_price, discount_amount, delivery_fee, order_status
            )
            SELECT 
                o.order_id,
                dd.date_key,
                dc.customer_key,
                dr.restaurant_key,
                di.item_key,
                COALESCE(dpm.payment_method_key, 1),
                oi.quantity,
                oi.unit_price,
                oi.discount,
                o.delivery_fee,
                o.status
            FROM raw_order_items oi
            JOIN raw_orders o ON oi.order_id = o.order_id
            JOIN dim_date dd ON dd.full_date = DATE(o.order_date)
            JOIN dim_customer dc ON dc.customer_id = o.customer_id 
                AND dc.valid_from <= o.order_date AND dc.valid_to > o.order_date
            JOIN dim_restaurant dr ON dr.restaurant_id = o.restaurant_id AND dr.is_current = TRUE
            JOIN dim_item di ON di.item_id = oi.menu_item_id
            LEFT JOIN raw_payments p ON p.order_id = o.order_id AND p.status = 'success'
            LEFT JOIN dim_payment_method dpm ON dpm.method_name = p.method
            WHERE o.order_date >= '{{ data_interval_start }}'
            AND o.order_date < '{{ data_interval_end }}'
            AND NOT EXISTS (
                SELECT 1 FROM fact_order_items f WHERE f.order_number = o.order_id
            );
        """,
    )

    # ─── Task 6: Update customer segments ───
    def update_segments(**context):
        hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
        hook.run("""
            WITH customer_stats AS (
                SELECT 
                    dc.customer_key,
                    COUNT(DISTINCT fo.order_number) AS order_count,
                    SUM(fo.net_amount) AS total_spent,
                    MIN(dd.full_date) AS first_order,
                    MAX(dd.full_date) AS last_order
                FROM dim_customer dc
                LEFT JOIN fact_order_items fo ON dc.customer_key = fo.customer_key
                LEFT JOIN dim_date dd ON fo.date_key = dd.date_key
                WHERE dc.is_current = TRUE
                GROUP BY dc.customer_key
            )
            UPDATE dim_customer d
            SET customer_segment = CASE
                WHEN cs.total_spent >= 500 THEN 'VIP'
                WHEN cs.total_spent >= 200 THEN 'Gold'
                WHEN cs.total_spent >= 50 THEN 'Regular'
                WHEN cs.order_count >= 1 THEN 'New'
                ELSE 'Prospect'
            END
            FROM customer_stats cs
            WHERE d.customer_key = cs.customer_key AND d.is_current = TRUE
            AND d.customer_segment IS DISTINCT FROM CASE
                WHEN cs.total_spent >= 500 THEN 'VIP'
                WHEN cs.total_spent >= 200 THEN 'Gold'
                WHEN cs.total_spent >= 50 THEN 'Regular'
                WHEN cs.order_count >= 1 THEN 'New'
                ELSE 'Prospect'
            END;
        """)
        print("✅ Customer segments updated")

    task_update_segments = PythonOperator(
        task_id='update_segments',
        python_callable=update_segments,
    )

    # ─── Task 7: Data Quality Checks ───
    def quality_checks(**context):
        hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
        start = context['data_interval_start']
        end = context['data_interval_end']
        errors = []
        
        # Check 1: No orphan fact rows
        orphans = hook.get_first("""
            SELECT COUNT(*) FROM fact_order_items f
            LEFT JOIN dim_date dd ON f.date_key = dd.date_key
            WHERE dd.date_key IS NULL
        """)[0]
        if orphans > 0:
            errors.append(f"Found {orphans} orphan fact rows (no matching dim_date)")
        
        # Check 2: Row count consistency
        source_count = hook.get_first("""
            SELECT COUNT(*) FROM raw_order_items oi
            JOIN raw_orders o ON oi.order_id = o.order_id
            WHERE o.order_date >= %s AND o.order_date < %s AND o.status != 'cancelled'
        """, parameters=(start, end))[0]
        
        warehouse_count = hook.get_first("""
            SELECT COUNT(*) FROM fact_order_items fo
            JOIN dim_date dd ON fo.date_key = dd.date_key
            WHERE dd.full_date >= %s AND dd.full_date < %s
        """, parameters=(start, end))[0]
        
        # Allow 5% tolerance (some orders might be cancelled/excluded)
        if abs(source_count - warehouse_count) > source_count * 0.05:
            errors.append(f"Row count mismatch: source={source_count}, warehouse={warehouse_count}")
        
        # Check 3: No negative amounts
        negatives = hook.get_first("""
            SELECT COUNT(*) FROM fact_order_items WHERE net_amount < 0
        """)[0]
        if negatives > 0:
            errors.append(f"Found {negatives} records with negative net_amount")
        
        # Check 4: SCD Type 2 integrity
        multi_current = hook.get_first("""
            SELECT COUNT(*) FROM (
                SELECT customer_id FROM dim_customer WHERE is_current = TRUE
                GROUP BY customer_id HAVING COUNT(*) > 1
            ) dupes
        """)[0]
        if multi_current > 0:
            errors.append(f"Found {multi_current} customers with multiple current rows")
        
        if errors:
            for e in errors:
                print(f"❌ {e}")
            raise ValueError(f"Quality checks failed: {len(errors)} issues found")
        
        print(f"✅ All quality checks passed (source={source_count}, warehouse={warehouse_count})")

    task_quality = PythonOperator(
        task_id='quality_checks',
        python_callable=quality_checks,
    )

    # ─── Task 8: Notify ───
    def notify(**context):
        count = context['ti'].xcom_pull(task_ids='extract_and_validate', key='order_count')
        ds = context['ds']
        print(f"""
        ╔═══════════════════════════════════════════╗
        ║   MakanExpress Warehouse ETL Complete     ║
        ╠═══════════════════════════════════════════╣
        ║   Date: {ds:<34}║
        ║   Orders: {count:<33}║
        ║   Dimensions: customer ✅ restaurant ✅   ║
        ║   Fact: loaded ✅                         ║
        ║   Quality: passed ✅                      ║
        ╚═══════════════════════════════════════════╝
        """)

    task_notify = PythonOperator(
        task_id='notify',
        python_callable=notify,
    )

    # ─── Dependencies ───
    task_validate >> [task_dim_customer, task_dim_restaurant, task_dim_item] >> task_load_fact >> [task_update_segments, task_quality] >> task_notify
```

### Concept 3: Understanding the Dependency Graph

```
                    extract_and_validate
                    /        |         \
                   /         |          \
    load_dim_customer  load_dim_restaurant  load_dim_item
                   \         |          /
                    \        |         /
                      load_fact
                      /          \
                     /            \
        update_segments     quality_checks
                     \            /
                      \          /
                       notify
```

All three dimension loads run in **parallel** after validation. The fact load waits for ALL dimensions. Then segment update and quality check run in parallel. Finally, notify runs after both.

---

## 🌤️ Afternoon Block (2 hours): Running & Debugging the Pipeline

### Concept 4: Running the DAG

```bash
# Test individual tasks before running the full DAG
airflow tasks test makanexpress_warehouse_etl extract_and_validate 2026-06-12
airflow tasks test makanexpress_warehouse_etl load_dim_customer 2026-06-12
airflow tasks test makanexpress_warehouse_etl load_fact 2026-06-12

# Run the full DAG
airflow dags trigger makanexpress_warehouse_etl

# Check status
airflow dags list-runs -d makanexpress_warehouse_etl

# View task logs
airflow tasks logs makanexpress_warehouse_etl extract_and_validate 2026-06-12

# Trigger with custom parameters
airflow dags trigger makanexpress_warehouse_etl --conf '{"dry_run": true}'
```

### Concept 5: Debugging Common Issues

```python
# Issue 1: "Connection not found"
# Fix: Check connection exists
airflow connections list | grep makanexpress

# Issue 2: "Task failed with no useful error"
# Fix: Check logs in detail
airflow tasks logs <dag_id> <task_id> <date> --raw

# Issue 3: "DAG not appearing in UI"
# Fix: Check for import errors
airflow dags list-import-errors

# Issue 4: "DAG import error" — Python syntax issue
# Fix: Validate the DAG file
python dags/makanexpress_warehouse_etl.py
# If this crashes, you have a Python syntax error

# Issue 5: "Task keeps failing on retry"
# Fix: Check if the data was partially loaded
# Use idempotent SQL (INSERT ... ON CONFLICT, or check before insert)
```

### Concept 6: Idempotent Tasks (Critical for Retries)

```python
# ❌ BAD: Non-idempotent — loading duplicates on retry
def load_fact_bad(**context):
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    hook.run("""
        INSERT INTO fact_order_items SELECT ...
    """)
    # If this fails halfway and retries, you get DUPLICATES!

# ✅ GOOD: Idempotent — safe to retry
def load_fact_good(**context):
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    ds = context['ds']
    
    # Step 1: Delete any existing data for this date (rollback partial load)
    hook.run("DELETE FROM fact_order_items WHERE order_number IN (SELECT order_id FROM raw_orders WHERE DATE(order_date) = %s)", (ds,))
    
    # Step 2: Insert fresh
    hook.run("INSERT INTO fact_order_items SELECT ...")

# ✅ ALSO GOOD: Use NOT EXISTS (as in our DAG above)
```

### Concept 7: Adding Logging and Monitoring

```python
import logging

logger = logging.getLogger(__name__)

def extract_with_logging(**context):
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    ds = context['ds']
    
    logger.info(f"Starting extraction for {ds}")
    
    try:
        count = hook.get_first("SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s", (ds,))[0]
        logger.info(f"Found {count} orders for {ds}")
    except Exception as e:
        logger.error(f"Extraction failed for {ds}: {e}")
        raise
    
    logger.info(f"Extraction complete: {count} records")
    return count
```

---

### 🏋️ Afternoon Exercises (15 questions)

1. Set up the complete warehouse tables (dim_date, dim_customer, dim_restaurant, dim_item, dim_payment_method, fact_order_items) in your PostgreSQL database.

2. Run the `makanexpress_warehouse_etl` DAG end-to-end. Verify all tasks complete successfully.

3. Simulate a customer city change: update a customer in `raw_customers` and re-run the DAG. Verify the SCD Type 2 change in `dim_customer`.

4. Add a new column `avg_delivery_time` to the fact table and modify the DAG to calculate it from `raw_orders`.

5. Write a task that generates a daily revenue summary report. Create a `daily_revenue_summary` table and load it after the fact table.

6. **Debugging exercise:** The `load_fact` task fails with a foreign key violation. What are the possible causes? How do you fix it?

7. Make the `load_fact` task idempotent. It should delete existing data for the date before inserting.

8. Add logging to all tasks. Each task should log: start time, records processed, and end time.

9. Create a second DAG `makanexpress_weekly_report` that runs every Monday and generates a weekly summary from the warehouse.

10. Write a task that calculates and stores the row counts of all warehouse tables in a `table_stats` audit table after each ETL run.

11. **Performance:** The `load_fact` task is slow because it joins 6 tables. Add an index strategy and measure the improvement.

12. Add error handling: if `quality_checks` fails, send a notification (print an alert message) instead of failing silently.

13. Create a "reprocess" DAG that lets you specify a date range to re-run the ETL for. Use DAG params.

14. Test backfill: run the ETL for 5 historical dates and verify each date loaded correctly.

15. **Challenge:** Add a complete audit layer to the pipeline:
    - An `etl_run_log` table that records every DAG run (start_time, end_time, status, rows_loaded)
    - Each task writes to this log on start and completion
    - A query to show the last 10 ETL runs with their durations and row counts

---

### ✅ Afternoon Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1:** Use the CREATE TABLE statements from Day 28's portfolio project.

**Answer 2:** Hands-on. Trigger the DAG and check each task's logs.

**Answer 3:**
```sql
UPDATE raw_customers SET city = 'Penang' WHERE customer_id = 1;
```
Re-run DAG. Check:
```sql
SELECT customer_key, customer_id, city, is_current FROM dim_customer WHERE customer_id = 1;
-- Should show 2 rows: old (SG, is_current=FALSE) and new (Penang, is_current=TRUE)
```

**Answer 4:**
```sql
ALTER TABLE fact_order_items ADD COLUMN avg_delivery_time_min DECIMAL(5,2);
```
Update the INSERT in `load_fact` to include delivery time calculation.

**Answer 5:**
```python
def generate_daily_report(**context):
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    ds = context['ds']
    hook.run(f"""
        INSERT INTO daily_revenue_summary (report_date, city, cuisine_type, total_orders, total_revenue, avg_order_value, unique_customers)
        SELECT 
            dd.full_date, dr.city, dr.cuisine_type,
            COUNT(DISTINCT fo.order_number), SUM(fo.net_amount),
            ROUND(AVG(fo.net_amount), 2), COUNT(DISTINCT fo.customer_key)
        FROM fact_order_items fo
        JOIN dim_date dd ON fo.date_key = dd.date_key
        JOIN dim_restaurant dr ON fo.restaurant_key = dr.restaurant_key
        WHERE dd.full_date = '{ds}' AND fo.order_status = 'completed'
        GROUP BY dd.full_date, dr.city, dr.cuisine_type
        ON CONFLICT (report_date, city, cuisine_type) DO UPDATE SET
            total_orders = EXCLUDED.total_orders,
            total_revenue = EXCLUDED.total_revenue;
    """)
```

**Answer 6:** Possible causes:
1. A customer/restaurant in raw_orders doesn't exist in the dimension yet (dim load failed or didn't run)
2. The SCD2 join condition is wrong (valid_from/valid_to doesn't cover the order date)
3. The dim_item load ran but missed some items

Fix: Ensure dimension loads complete BEFORE fact load (which our DAG does via dependencies).

**Answer 7:** Already implemented — the `NOT EXISTS` clause in our INSERT prevents duplicates. Also add a DELETE before insert:
```sql
DELETE FROM fact_order_items
WHERE order_number IN (
    SELECT order_id FROM raw_orders 
    WHERE order_date >= '{{ data_interval_start }}' AND order_date < '{{ data_interval_end }}'
);
```

**Answer 8:**
```python
import logging
logger = logging.getLogger(__name__)

def logged_task(**context):
    import time
    start = time.time()
    logger.info(f"[START] Task for {context['ds']}")
    
    # ... do work ...
    count = 42
    
    elapsed = time.time() - start
    logger.info(f"[END] Task for {context['ds']}: {count} records in {elapsed:.2f}s")
```

**Answer 9:**
```python
with DAG('makanexpress_weekly_report', schedule='0 9 * * 1',  # Monday 9 AM
         start_date=datetime(2026, 6, 15, tzinfo=pytz.timezone('Asia/Singapore')),
         timezone=pytz.timezone('Asia/Singapore'), catchup=False) as dag:
    # ... tasks to generate weekly summary ...
```

**Answer 10:**
```python
def log_table_stats(**context):
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    tables = ['dim_customer', 'dim_restaurant', 'dim_item', 'fact_order_items']
    for table in tables:
        count = hook.get_first(f"SELECT COUNT(*) FROM {table}")[0]
        hook.run("""
            INSERT INTO table_stats (table_name, row_count, run_date)
            VALUES (%s, %s, %s)
        """, parameters=(table, count, context['ds']))
```

**Answers 11-15:** Hands-on exercises. Apply the concepts from Concepts 5-7.

</details>

---

## 🌙 Evening Block (1 hour): Review

### 📝 Today's Checklist

- [ ] I built and ran a complete ETL pipeline in Airflow
- [ ] I understand the dependency graph (fan-out, fan-in)
- [ ] I can debug common Airflow issues
- [ ] I understand idempotent tasks and why they matter
- [ ] I can add logging and quality checks to my pipeline
- [ ] I pushed code to GitHub

---

*Day 31 complete! Tomorrow: Sensors, Branching, and advanced XCom patterns.* 🔧
