# 📅 Day 35 — Wednesday, 18 June 2026
# 🏗️ PORTFOLIO PROJECT: Add Airflow to Food Delivery Warehouse

---

## 🎯 Today's Goal

Take the `food-delivery-warehouse` project from Day 28 and add **Airflow orchestration**. The result: a production-ready data pipeline that orchestrates the complete ETL from raw data to star schema warehouse.

---

## 📋 Project Overview

### What You're Building

```
food-delivery-warehouse/
├── README.md                    ← update with Airflow section
├── schema/                      ← from Day 28
├── seed/                        ← from Day 28
├── etl/                         ← from Day 28
├── analytics/                   ← from Day 28
├── quality/                     ← from Day 28
├── docs/                        ← from Day 28
├── dags/                        ← NEW: Airflow DAGs
│   ├── makanexpress_warehouse_etl.py    ← main daily ETL
│   ├── makanexpress_weekly_report.py    ← weekly report DAG
│   └── makanexpress_quality_monitor.py  ← monitoring DAG
├── tests/                       ← NEW: unit tests
│   ├── test_dag_structure.py
│   └── test_tasks.py
├── sql/                         ← NEW: SQL files for operators
│   ├── load_dim_customer.sql
│   ├── load_dim_restaurant.sql
│   ├── load_dim_item.sql
│   └── load_fact.sql
├── scripts/                     ← NEW: helper scripts
│   └── setup_connections.sh
└── requirements.txt             ← NEW: Python dependencies
```

---

## 🏗️ Build Steps

### Step 1: Add Airflow to the Existing Project (30 min)

```bash
cd food-delivery-warehouse

# Create directories
mkdir -p dags tests sql scripts

# Create requirements.txt
cat > requirements.txt << 'EOF'
apache-airflow==2.10.4
apache-airflow-providers-postgres
psycopg2-binary
pandas
pytest
EOF

# Set up Airflow (if not already done)
export AIRFLOW_HOME=~/airflow
airflow db init
```

### Step 2: Create SQL Files for Operators (30 min)

**`sql/load_dim_customer.sql`:**
```sql
-- SCD Type 2: Expire changed customer records
UPDATE dim_customer d
SET valid_to = NOW(), is_current = FALSE
FROM raw_customers s
WHERE d.customer_id = s.customer_id
AND d.is_current = TRUE
AND (
    d.customer_name IS DISTINCT FROM s.name
    OR d.email IS DISTINCT FROM s.email
    OR d.city IS DISTINCT FROM s.city
    OR d.postal_code IS DISTINCT FROM s.postal_code
);

-- Insert new and changed customer records
INSERT INTO dim_customer (
    customer_id, customer_name, email, phone, 
    city, postal_code, country, signup_date,
    customer_segment, valid_from, valid_to, is_current
)
SELECT 
    s.customer_id,
    s.name,
    s.email,
    s.phone,
    s.city,
    s.postal_code,
    CASE WHEN s.city IN ('Singapore') THEN 'Singapore' ELSE 'Malaysia' END,
    s.signup_date,
    'New',
    NOW(),
    '9999-12-31 23:59:59',
    TRUE
FROM raw_customers s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer d 
    WHERE d.customer_id = s.customer_id 
    AND d.is_current = TRUE
    AND d.customer_name = s.name
    AND d.email = s.email
    AND d.city = s.city
    AND d.postal_code = s.postal_code
);
```

**`sql/load_dim_restaurant.sql`:**
```sql
-- SCD Type 2: Expire changed restaurant records
UPDATE dim_restaurant d
SET valid_to = NOW(), is_current = FALSE
FROM raw_restaurants s
WHERE d.restaurant_id = s.restaurant_id
AND d.is_current = TRUE
AND (
    d.restaurant_name IS DISTINCT FROM s.name
    OR d.cuisine_type IS DISTINCT FROM s.cuisine
    OR d.city IS DISTINCT FROM s.city
    OR d.rating IS DISTINCT FROM s.rating
);

-- Insert new and changed restaurant records
INSERT INTO dim_restaurant (
    restaurant_id, restaurant_name, cuisine_type, cuisine_category,
    city, postal_code, rating, price_range, is_active,
    valid_from, valid_to, is_current
)
SELECT 
    s.restaurant_id,
    s.name,
    s.cuisine,
    CASE 
        WHEN s.cuisine IN ('Chinese','Malay','Indian','Thai','Japanese') THEN 'Asian'
        WHEN s.cuisine IN ('Western','Dessert') THEN 'Western/Other'
        ELSE 'Fusion'
    END,
    s.city,
    s.postal_code,
    s.rating,
    CASE 
        WHEN s.rating >= 4.5 THEN 'Premium'
        WHEN s.rating >= 3.5 THEN 'Mid'
        ELSE 'Budget'
    END,
    s.is_active,
    NOW(),
    '9999-12-31 23:59:59',
    TRUE
FROM raw_restaurants s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_restaurant d 
    WHERE d.restaurant_id = s.restaurant_id 
    AND d.is_current = TRUE
    AND d.restaurant_name = s.name
    AND d.cuisine_type = s.cuisine
    AND d.rating = s.rating
);
```

**`sql/load_dim_item.sql`:**
```sql
-- SCD Type 1: Insert new, update existing
INSERT INTO dim_item (item_id, restaurant_id, dish_name, category, price, is_vegetarian, spice_level)
SELECT item_id, restaurant_id, dish_name, category, price, is_vegetarian, spice_level
FROM raw_menu_items s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_item d WHERE d.item_id = s.item_id
);

UPDATE dim_item d
SET price = s.price, dish_name = s.dish_name, is_vegetarian = s.is_vegetarian
FROM raw_menu_items s
WHERE d.item_id = s.item_id
AND (d.price IS DISTINCT FROM s.price OR d.dish_name IS DISTINCT FROM s.dish_name);
```

**`sql/load_fact.sql`:**
```sql
-- Idempotent: delete existing data for this date range first
DELETE FROM fact_order_items
WHERE order_number IN (
    SELECT order_id FROM raw_orders
    WHERE order_date >= '{{ data_interval_start }}'
    AND order_date < '{{ data_interval_end }}'
);

-- Insert fresh data
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
AND o.order_date < '{{ data_interval_end }}';
```

### Step 3: Create the Main ETL DAG (1 hour)

**`dags/makanexpress_warehouse_etl.py`:**
```python
"""
MakanExpress Warehouse ETL Pipeline
====================================
Orchestrates the daily ETL from raw (OLTP) to star schema warehouse.

Schedule: Daily at 6 AM SGT
Source: raw_customers, raw_restaurants, raw_orders, raw_order_items, raw_payments
Target: dim_customer, dim_restaurant, dim_item, fact_order_items

Features:
- SCD Type 2 for customers and restaurants
- Idempotent fact loading
- Data quality checks
- Error handling with retries
- Callback-based monitoring
"""

from datetime import datetime, timedelta
import pytz
import logging
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.utils.trigger_rule import TriggerRule

logger = logging.getLogger(__name__)

# ─── Constants ───
CONNECTION_ID = 'makanexpress_db'
SG_TZ = pytz.timezone('Asia/Singapore')
SQL_DIR = '/opt/airflow/dags/sql'  # adjust for your setup

# ─── Callbacks ───
def on_failure(context):
    """Log failure to audit table."""
    dag_id = context['dag'].dag_id
    task_id = context['task'].task_id
    logical_date = str(context['logical_date'])
    error = str(context.get('exception', 'Unknown'))
    logger.error(f"❌ {dag_id}.{task_id} failed for {logical_date}: {error}")
    
    try:
        hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
        hook.run("""
            CREATE TABLE IF NOT EXISTS etl_audit_log (
                id SERIAL PRIMARY KEY,
                event_type VARCHAR(20),
                dag_id VARCHAR(100),
                task_id VARCHAR(100),
                logical_date VARCHAR(50),
                message TEXT,
                created_at TIMESTAMP DEFAULT NOW()
            );
            INSERT INTO etl_audit_log (event_type, dag_id, task_id, logical_date, message)
            VALUES ('failure', %s, %s, %s, %s);
        """, parameters=(dag_id, task_id, logical_date, error))
    except Exception:
        pass  # don't fail the callback

# ─── Task Functions ───
def extract_and_validate(**context):
    """Extract source data counts and validate."""
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    start = context['data_interval_start']
    end = context['data_interval_end']
    
    # Count source records
    order_count = hook.get_first("""
        SELECT COUNT(*) FROM raw_orders
        WHERE order_date >= %s AND order_date < %s
    """, parameters=(start, end))[0]
    
    if order_count == 0:
        raise ValueError(f"No orders found between {start} and {end}")
    
    # Validate: check NULLs
    null_count = hook.get_first("""
        SELECT COUNT(*) FROM raw_orders
        WHERE order_date >= %s AND order_date < %s
        AND (customer_id IS NULL OR restaurant_id IS NULL OR total_amount IS NULL)
    """, parameters=(start, end))[0]
    
    if null_count > 5:
        raise ValueError(f"Too many NULL values: {null_count}")
    
    logger.info(f"✅ Validation: {order_count} orders, {null_count} NULLs")
    context['ti'].xcom_push(key='order_count', value=order_count)
    return order_count

def update_customer_segments(**context):
    """Update customer segments based on spending."""
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    hook.run("""
        WITH customer_stats AS (
            SELECT 
                dc.customer_key,
                COALESCE(COUNT(DISTINCT fo.order_number), 0) AS order_count,
                COALESCE(SUM(fo.net_amount), 0) AS total_spent
            FROM dim_customer dc
            LEFT JOIN fact_order_items fo ON dc.customer_key = fo.customer_key
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
    logger.info("✅ Customer segments updated")

def quality_checks(**context):
    """Run data quality checks on the warehouse."""
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
        errors.append(f"{orphans} orphan fact rows")
    
    # Check 2: SCD integrity
    multi_current = hook.get_first("""
        SELECT COUNT(*) FROM (
            SELECT customer_id FROM dim_customer WHERE is_current = TRUE
            GROUP BY customer_id HAVING COUNT(*) > 1
        ) dupes
    "")[0]
    if multi_current > 0:
        errors.append(f"{multi_current} customers with multiple current rows")
    
    # Check 3: Row count sanity
    source_count = hook.get_first("""
        SELECT COUNT(*) FROM raw_orders
        WHERE order_date >= %s AND order_date < %s
    """, parameters=(start, end))[0]
    
    warehouse_count = hook.get_first("""
        SELECT COUNT(DISTINCT order_number) FROM fact_order_items fo
        JOIN dim_date dd ON fo.date_key = dd.date_key
        WHERE dd.full_date >= %s AND dd.full_date < %s
    """, parameters=(start, end))[0]
    
    if source_count > 0 and warehouse_count == 0:
        errors.append("Fact table has 0 rows despite source having data")
    
    if errors:
        for e in errors:
            logger.error(f"❌ {e}")
        raise ValueError(f"Quality checks failed: {len(errors)} issues")
    
    logger.info(f"✅ Quality checks passed: source={source_count}, warehouse={warehouse_count}")

def notify(**context):
    """Send completion notification."""
    count = context['ti'].xcom_pull(task_ids='extract_and_validate', key='order_count')
    ds = context['ds']
    
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    hook.run("""
        CREATE TABLE IF NOT EXISTS etl_audit_log (
            id SERIAL PRIMARY KEY,
            event_type VARCHAR(20),
            dag_id VARCHAR(100),
            task_id VARCHAR(100),
            logical_date VARCHAR(50),
            message TEXT,
            created_at TIMESTAMP DEFAULT NOW()
        );
        INSERT INTO etl_audit_log (event_type, dag_id, task_id, logical_date, message)
        VALUES ('success', %s, 'notify', %s, %s);
    """, parameters=(
        context['dag'].dag_id,
        ds,
        f"ETL complete: {count} orders processed",
    ))
    
    logger.info(f"""
    ╔═══════════════════════════════════════════╗
    ║   MakanExpress Warehouse ETL Complete     ║
    ╠═══════════════════════════════════════════╣
    ║   Date: {ds:<34}║
    ║   Orders: {count:<33}║
    ║   Status: ✅ SUCCESS                      ║
    ╚═══════════════════════════════════════════╝
    """)

# ─── DAG Definition ───
default_args = {
    'owner': 'data-engineering',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(hours=1),
    'on_failure_callback': on_failure,
}

with DAG(
    dag_id='makanexpress_warehouse_etl',
    default_args=default_args,
    description='Daily ETL: raw → star schema warehouse with SCD Type 2',
    schedule='0 6 * * *',
    start_date=datetime(2026, 6, 1, tzinfo=SG_TZ),
    timezone=SG_TZ,
    catchup=False,
    max_active_runs=1,
    tags=['makanexpress', 'etl', 'warehouse', 'production'],
) as dag:

    # ─── Tasks ───
    task_validate = PythonOperator(
        task_id='extract_and_validate',
        python_callable=extract_and_validate,
    )

    task_dim_customer = PostgresOperator(
        task_id='load_dim_customer',
        postgres_conn_id=CONNECTION_ID,
        sql='sql/load_dim_customer.sql',
    )

    task_dim_restaurant = PostgresOperator(
        task_id='load_dim_restaurant',
        postgres_conn_id=CONNECTION_ID,
        sql='sql/load_dim_restaurant.sql',
    )

    task_dim_item = PostgresOperator(
        task_id='load_dim_item',
        postgres_conn_id=CONNECTION_ID,
        sql='sql/load_dim_item.sql',
    )

    task_load_fact = PostgresOperator(
        task_id='load_fact',
        postgres_conn_id=CONNECTION_ID,
        sql='sql/load_fact.sql',
    )

    task_update_segments = PythonOperator(
        task_id='update_segments',
        python_callable=update_customer_segments,
    )

    task_quality = PythonOperator(
        task_id='quality_checks',
        python_callable=quality_checks,
    )

    task_notify = PythonOperator(
        task_id='notify',
        python_callable=notify,
        trigger_rule=TriggerRule.ALL_SUCCESS,
    )

    # ─── Dependencies ───
    task_validate >> [task_dim_customer, task_dim_restaurant, task_dim_item] >> task_load_fact >> [task_update_segments, task_quality] >> task_notify
```

### Step 4: Create Weekly Report DAG (30 min)

**`dags/makanexpress_weekly_report.py`:**
```python
"""MakanExpress Weekly Report Generator — runs every Monday."""

from datetime import datetime, timedelta
import pytz
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook

def generate_weekly_report(**context):
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    end = context['data_interval_end']
    start = context['data_interval_start']
    
    # Weekly revenue by city and cuisine
    report = hook.get_pandas_df("""
        SELECT 
            dr.city,
            dr.cuisine_type,
            COUNT(DISTINCT fo.order_number) AS total_orders,
            SUM(fo.net_amount) AS total_revenue,
            ROUND(AVG(fo.net_amount), 2) AS avg_order_value,
            COUNT(DISTINCT fo.customer_key) AS unique_customers
        FROM fact_order_items fo
        JOIN dim_date dd ON fo.date_key = dd.date_key
        JOIN dim_restaurant dr ON fo.restaurant_key = dr.restaurant_key
        WHERE dd.full_date >= %s AND dd.full_date < %s
        AND fo.order_status = 'completed'
        GROUP BY dr.city, dr.cuisine_type
        ORDER BY dr.city, total_revenue DESC
    """, parameters=(start, end))
    
    print(f"📊 Weekly Report: {start} to {end}")
    print(report.to_string(index=False))
    
    # Store in report table
    hook.run("""
        CREATE TABLE IF NOT EXISTS weekly_reports (
            id SERIAL PRIMARY KEY,
            week_start DATE,
            week_end DATE,
            city VARCHAR(50),
            cuisine_type VARCHAR(50),
            total_orders INT,
            total_revenue DECIMAL(12,2),
            avg_order_value DECIMAL(10,2),
            unique_customers INT,
            created_at TIMESTAMP DEFAULT NOW()
        );
    """)
    
    for _, row in report.iterrows():
        hook.run("""
            INSERT INTO weekly_reports (week_start, week_end, city, cuisine_type, 
                                         total_orders, total_revenue, avg_order_value, unique_customers)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
        """, parameters=(
            start.date(), end.date(),
            row['city'], row['cuisine_type'],
            int(row['total_orders']), float(row['total_revenue']),
            float(row['avg_order_value']), int(row['unique_customers']),
        ))

with DAG(
    dag_id='makanexpress_weekly_report',
    schedule='0 9 * * 1',  # Monday 9 AM SGT
    start_date=datetime(2026, 6, 15, tzinfo=pytz.timezone('Asia/Singapore')),
    timezone=pytz.timezone('Asia/Singapore'),
    catchup=False,
    tags=['makanexpress', 'report', 'weekly'],
) as dag:
    generate = PythonOperator(
        task_id='generate_report',
        python_callable=generate_weekly_report,
    )
```

### Step 5: Create Unit Tests (30 min)

**`tests/test_dag_structure.py`:**
```python
"""Unit tests for MakanExpress DAGs."""
import pytest
from airflow.models import DagBag

@pytest.fixture
def dagbag():
    return DagBag(dag_folder='dags/', include_examples=False)

class TestWarehouseETL:
    def test_dag_loads(self, dagbag):
        dag = dagbag.get_dag('makanexpress_warehouse_etl')
        assert dag is not None
        assert len(dagbag.import_errors) == 0

    def test_expected_tasks(self, dagbag):
        dag = dagbag.get_dag('makanexpress_warehouse_etl')
        task_ids = {t.task_id for t in dag.tasks}
        expected = {
            'extract_and_validate', 'load_dim_customer', 'load_dim_restaurant',
            'load_dim_item', 'load_fact', 'update_segments', 'quality_checks', 'notify'
        }
        assert expected == task_ids

    def test_catchup_false(self, dagbag):
        dag = dagbag.get_dag('makanexpress_warehouse_etl')
        assert dag.catchup is False

    def test_max_active_runs(self, dagbag):
        dag = dagbag.get_dag('makanexpress_warehouse_etl')
        assert dag.max_active_runs == 1

    def test_has_retries(self, dagbag):
        dag = dagbag.get_dag('makanexpress_warehouse_etl')
        assert dag.default_args['retries'] >= 2

    def test_dependencies(self, dagbag):
        dag = dagbag.get_dag('makanexpress_warehouse_etl')
        validate = dag.get_task('extract_and_validate')
        downstream = {t.task_id for t in validate.downstream_list}
        assert 'load_dim_customer' in downstream
        assert 'load_dim_restaurant' in downstream
        assert 'load_dim_item' in downstream

class TestWeeklyReport:
    def test_dag_loads(self, dagbag):
        dag = dagbag.get_dag('makanexpress_weekly_report')
        assert dag is not None

    def test_schedule(self, dagbag):
        dag = dagbag.get_dag('makanexpress_weekly_report')
        assert dag.schedule_interval == '0 9 * * 1'
```

### Step 6: Setup Script (15 min)

**`scripts/setup_connections.sh`:**
```bash
#!/bin/bash
# Setup Airflow connections and variables for MakanExpress

echo "Setting up MakanExpress Airflow connections..."

# PostgreSQL connection
airflow connections add makanexpress_db \
    --conn-type postgres \
    --conn-host localhost \
    --conn-schema makanexpress \
    --conn-login postgres \
    --conn-port 5432 \
    2>/dev/null || echo "Connection already exists"

# Variables
airflow variables set batch_size 5000
airflow variables set environment dev

echo "✅ Setup complete!"
```

### Step 7: Update README.md (30 min)

Add an Airflow section to your existing README:

```markdown
## 🌬️ Airflow Orchestration

### DAGs

| DAG | Schedule | Description |
|-----|----------|-------------|
| `makanexpress_warehouse_etl` | Daily 6 AM SGT | Full ETL: raw → warehouse |
| `makanexpress_weekly_report` | Monday 9 AM SGT | Weekly revenue report |

### Setup

1. Install dependencies: `pip install -r requirements.txt`
2. Initialize Airflow: `airflow db init`
3. Run setup: `bash scripts/setup_connections.sh`
4. Copy DAGs: `cp -r dags/ $AIRFLOW_HOME/dags/`
5. Copy SQL: `cp -r sql/ $AIRFLOW_HOME/dags/sql/`
6. Start Airflow: `airflow webserver & airflow scheduler`
7. Trigger manually: `airflow dags trigger makanexpress_warehouse_etl`

### Architecture

```
extract_and_validate
    ├── load_dim_customer (SCD Type 2)
    ├── load_dim_restaurant (SCD Type 2)
    └── load_dim_item (SCD Type 1)
            └── load_fact (idempotent)
                ├── update_segments
                └── quality_checks
                        └── notify
```

### Running Tests

```bash
pytest tests/ -v
```
```

---

## 🌙 Evening: Push to GitHub & Celebrate!

### Final Checklist
- [ ] Main ETL DAG runs end-to-end
- [ ] Weekly report DAG runs successfully
- [ ] Unit tests pass
- [ ] All SQL files are modular and tested
- [ ] README updated with Airflow documentation
- [ ] Setup script works
- [ ] Quality checks pass after ETL
- [ ] Error handling tested (simulate failures)
- [ ] Committed and pushed to GitHub

### 🎉 Week 5 Complete!

**What you learned this week:**
- Airflow core: DAGs, tasks, operators, scheduling
- Connections, Variables, Hooks, XCom
- Sensors, branching, trigger rules, TaskGroups
- Error handling, retries, testing, best practices
- Built a production-ready orchestrated pipeline

**Portfolio projects:**
1. ✅ Retail Sales Analysis (SQL)
2. ✅ Crypto Data Pipeline (Python ETL)
3. ✅ Food Delivery Analytics (dbt)
4. ✅ Food Delivery Warehouse (Star Schema + SCD) — **now with Airflow!**

---

*Week 5 done! Next week: AWS Core — taking your pipelines to the cloud.* ☁️🎊
