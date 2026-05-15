# 📅 Day 30 — Friday, 13 June 2026
# 🔧 Airflow Connections, Variables, and Scheduling

---

## 🎯 Today's Goal

Yesterday you wrote DAGs. Today you make them **production-ready** — connections to databases, configurable variables, proper scheduling, and understanding how Airflow manages time. This is what separates a toy DAG from a real pipeline.

---

## ☀️ Morning Block (2 hours): Connections & Variables

### Concept 1: Airflow Connections

Connections store credentials for external systems (databases, APIs, cloud services). **Never hardcode passwords in DAG files!**

```python
# ❌ BAD — credentials in code
conn = psycopg2.connect(
    host='localhost',
    database='makanexpress',
    user='postgres',
    password='my_secret_123'  # NEVER do this!
)

# ✅ GOOD — use Airflow connection
from airflow.hooks.base import BaseHook

def get_connection():
    # Get connection details from Airflow
    conn = BaseHook.get_connection('makanexpress_db')
    # conn has: host, login, password, schema, port, extra
    
    import psycopg2
    db_conn = psycopg2.connect(
        host=conn.host,
        database=conn.schema,
        user=conn.login,
        password=conn.password,
        port=conn.port,
    )
    return db_conn
```

**Adding connections via UI:**
1. Go to Admin → Connections → Add Connection
2. Fill in:
   - Conn Id: `makanexpress_db`
   - Conn Type: `PostgreSQL`
   - Host: `localhost`
   - Schema: `makanexpress`
   - Login: `postgres`
   - Password: (your password)
   - Port: `5432`

**Adding connections via CLI:**
```bash
airflow connections add makanexpress_db \
    --conn-type postgres \
    --conn-host localhost \
    --conn-schema makanexpress \
    --conn-login postgres \
    --conn-password your_password \
    --conn-port 5432
```

**Adding connections via code (environment variables):**
```bash
# In your .bashrc or .env file:
export AIRFLOW_CONN_MAKANEXPRESS_DB="postgresql://postgres:your_password@localhost:5432/makanexpress"
```

### Concept 2: Airflow Variables

Variables store reusable configuration values. Change them from the UI without modifying DAG code.

```python
# ❌ BAD — hardcoded configuration
BATCH_SIZE = 1000
S3_BUCKET = 'my-data-lake-prod'
NOTIFICATION_EMAIL = 'team@company.com'

# ✅ GOOD — use Airflow Variables
from airflow.models import Variable

batch_size = Variable.get('batch_size', default_var=1000)
bucket = Variable.get('s3_bucket')
email = Variable.get('notification_email')

# In your DAG:
def load_data(**context):
    batch_size = int(Variable.get('batch_size', default_var='1000'))
    print(f"Loading with batch_size={batch_size}")
```

**Setting variables:**
```bash
# Via CLI
airflow variables set batch_size 5000
airflow variables set s3_bucket makanexpress-data-lake
airflow variables set notification_email data-team@makanexpress.com

# List all
airflow variables list
```

**Via UI:** Admin → Variables → Add

**JSON variables (for complex config):**
```bash
airflow variables set etl_config '{"batch_size": 5000, "retry_on_fail": true, "tables": ["orders", "customers"]}'
```

```python
import json
config = json.loads(Variable.get('etl_config'))
batch_size = config['batch_size']
tables = config['tables']
```

### Concept 3: Using Hooks (The Right Way)

Hooks are Airflow's interface to external systems. Use them instead of raw connections.

```python
from airflow.providers.postgres.hooks.postgres import PostgresHook

def count_orders_with_hook(**context):
    ds = context['ds']
    
    # Use the PostgresHook — handles connection pooling, closing, etc.
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    
    # Run a query and get results
    records = hook.get_records("""
        SELECT restaurant_id, COUNT(*) as order_count, SUM(total_amount) as revenue
        FROM raw_orders
        WHERE DATE(order_date) = %s AND status = 'completed'
        GROUP BY restaurant_id
    """, parameters=(ds,))
    
    print(f"Found {len(records)} restaurants with orders on {ds}")
    for row in records:
        print(f"  Restaurant {row[0]}: {row[1]} orders, ${row[2]}")
    
    return records

# Get a single value
def get_order_count(**context):
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    count = hook.get_first(
        "SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s",
        parameters=(context['ds'],)
    )[0]
    print(f"Order count: {count}")
    return count

# Run a SQL command (no return)
def truncate_staging(**context):
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    hook.run("TRUNCATE TABLE staging_orders")
    hook.run("""
        INSERT INTO staging_orders
        SELECT * FROM raw_orders WHERE DATE(order_date) = %s
    """, parameters=(context['ds'],))

# Get a pandas DataFrame directly
def analyze_orders(**context):
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    df = hook.get_pandas_df("""
        SELECT cuisine_type, AVG(total_amount) as avg_order, COUNT(*) as cnt
        FROM raw_orders o
        JOIN raw_restaurants r ON o.restaurant_id = r.restaurant_id
        WHERE DATE(o.order_date) = %s
        GROUP BY cuisine_type
    """, parameters=(context['ds'],))
    
    print(df.to_string())
    return df.to_dict('records')
```

### Concept 4: XCom — Cross-Task Communication

XCom (Cross-Communication) lets tasks pass small amounts of data to each other.

```python
# Task 1: Push data to XCom
def extract(**context):
    ds = context['ds']
    # ... extract data ...
    order_count = 42
    revenue = 1250.50
    
    # XCom is automatically pushed when a function returns a value
    return {'order_count': order_count, 'revenue': revenue}
    # This automatically creates an XCom with key='return_value'

# Task 2: Pull data from XCom
def transform(**context):
    # Pull from a specific task
    data = context['ti'].xcom_pull(task_ids='extract')
    # data = {'order_count': 42, 'revenue': 1250.50}
    
    order_count = data['order_count']
    revenue = data['revenue']
    print(f"Processing {order_count} orders, revenue ${revenue}")

# Explicit XCom push
def check_data(**context):
    ds = context['ds']
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    count = hook.get_first("SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s", (ds,))[0]
    
    # Explicit push with custom key
    context['ti'].xcom_push(key='order_count', value=count)
    context['ti'].xcom_push(key='is_valid', value=count > 0)

# Pull with custom key
def load(**context):
    count = context['ti'].xcom_pull(task_ids='check', key='order_count')
    is_valid = context['ti'].xcom_pull(task_ids='check', key='is_valid')
    
    if not is_valid:
        print("No data to load!")
        return
    
    print(f"Loading {count} records")
```

**⚠️ XCom limits:**
- Default max size: 48KB (don't pass DataFrames!)
- Use for metadata (counts, dates, file paths) not actual data
- For large data, write to a file/table and pass the path via XCom

### Concept 5: The Airflow Context Dictionary

Every PythonOperator function receives a `context` dict with rich metadata:

```python
def show_context(**context):
    """Print all available context variables."""
    print(f"DS: {context['ds']}")                         # 2026-06-12
    print(f"DS_NODASH: {context['ds_nodash']}")           # 20260612
    print(f"Logical date: {context['logical_date']}")     # datetime(2026, 6, 12)
    print(f"Data interval start: {context['data_interval_start']}")  # 2026-06-11T00:00
    print(f"Data interval end: {context['data_interval_end']}")      # 2026-06-12T00:00
    print(f"DAG ID: {context['dag'].dag_id}")             # my_dag
    print(f"Task ID: {context['task'].task_id}")           # my_task
    print(f"Run ID: {context['run_id']}")                  # scheduled__2026-06-12
    print(f"TI: {context['ti']}")                          # TaskInstance object
    print(f"Params: {context.get('params', {})}")         # user params
    print(f"Conf: {context.get('conf', {})}")             # run configuration

    # Available via ti (TaskInstance):
    ti = context['ti']
    print(f"Try number: {ti.try_number}")                  # 0, 1, 2...
    print(f"State: {ti.state}")                            # running, success, failed
    print(f"Max retries: {ti.max_tries}")
```

---

### 🏋️ Morning Exercises (15 questions)

1. Create a PostgreSQL connection in Airflow called `makanexpress_db`. Test it using `airflow connections test makanexpress_db`.

2. Set 3 Airflow Variables: `batch_size=5000`, `source_schema=raw`, `target_schema=warehouse`. Retrieve them in a Python function.

3. Write a Python function that uses `PostgresHook` to count orders for `{{ ds }}`. Return the count via XCom.

4. Write two tasks: Task A pushes `{'revenue': 1250, 'orders': 42}` to XCom. Task B pulls it and prints "Revenue: $1250, Orders: 42".

5. What happens if Task B tries to `xcom_pull` from Task A but Task A hasn't run yet? What if Task A failed?

6. Create a JSON variable called `pipeline_config` with settings for your ETL. Parse it in a DAG and use the values.

7. Write a DAG that uses `PostgresHook.get_pandas_df()` to load orders into a DataFrame, calculate summary stats, and print them.

8. Use `context['data_interval_start']` and `context['data_interval_end']` to query a date range (not just a single day). Write the query.

9. Create a DAG where the first task reads a Variable to determine which table to process. The Variable can be changed from the UI without modifying the DAG.

10. **Security exercise:** Find and fix the security issue:
```python
def connect():
    import psycopg2
    conn = psycopg2.connect(
        host='prod-db.makanexpress.com',
        database='warehouse',
        user='admin',
        password='SuperSecret123!'
    )
    return conn
```

11. Write a function that pushes multiple XCom values with custom keys: `valid_count`, `invalid_count`, `total_count`. Write another function that pulls all three.

12. Use `PostgresHook.run()` to execute a stored procedure or multi-statement SQL. Show an example of running BEGIN...COMMIT.

13. Create a DAG that uses a connection and a variable together: read the connection for the database, and a variable for the batch size.

14. **Debugging:** A task fails with "Connection 'my_db' not found". What are 3 possible causes?

15. Write a function that uses `BaseHook.get_connection()` to print all connection details (host, port, login, schema) without the password.

---

### ✅ Morning Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1:**
```bash
airflow connections add makanexpress_db \
    --conn-type postgres \
    --conn-host localhost \
    --conn-schema makanexpress \
    --conn-login postgres \
    --conn-port 5432

airflow connections test makanexpress_db
```

**Answer 2:**
```bash
airflow variables set batch_size 5000
airflow variables set source_schema raw
airflow variables set target_schema warehouse
```
```python
from airflow.models import Variable
batch_size = int(Variable.get('batch_size'))  # 5000
source = Variable.get('source_schema')        # 'raw'
target = Variable.get('target_schema')        # 'warehouse'
```

**Answer 3:**
```python
def count_orders(**context):
    from airflow.providers.postgres.hooks.postgres import PostgresHook
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    count = hook.get_first(
        "SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s",
        parameters=(context['ds'],)
    )[0]
    print(f"Orders for {context['ds']}: {count}")
    return count  # automatically pushed to XCom
```

**Answer 4:**
```python
def push_data(**context):
    return {'revenue': 1250, 'orders': 42}

def pull_data(**context):
    data = context['ti'].xcom_pull(task_ids='task_a')
    print(f"Revenue: ${data['revenue']}, Orders: {data['orders']}")

task_a = PythonOperator(task_id='task_a', python_callable=push_data)
task_b = PythonOperator(task_id='task_b', python_callable=pull_data)
task_a >> task_b
```

**Answer 5:** If Task A hasn't run: `xcom_pull` returns `None`. If Task A failed but returned a value before failing: XCom IS available (XCom is pushed on return, regardless of task success). If Task A never reached the return statement: XCom is not available.

**Answer 6:**
```bash
airflow variables set pipeline_config '{"batch_size": 5000, "tables": ["orders", "customers"], "retries": 3}'
```
```python
import json
config = json.loads(Variable.get('pipeline_config'))
batch_size = config['batch_size']
tables = config['tables']
```

**Answer 7:**
```python
def analyze_orders(**context):
    from airflow.providers.postgres.hooks.postgres import PostgresHook
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    df = hook.get_pandas_df("""
        SELECT r.cuisine_type, COUNT(*) as orders, SUM(o.total_amount) as revenue
        FROM raw_orders o
        JOIN raw_restaurants r ON o.restaurant_id = r.restaurant_id
        WHERE DATE(o.order_date) = %s
        GROUP BY r.cuisine_type
    """, parameters=(context['ds'],))
    print(df.describe())
    print(df.to_string())
```

**Answer 8:**
```python
def query_range(**context):
    from airflow.providers.postgres.hooks.postgres import PostgresHook
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    start = context['data_interval_start']
    end = context['data_interval_end']
    records = hook.get_records("""
        SELECT * FROM raw_orders
        WHERE order_date >= %s AND order_date < %s
    """, parameters=(start, end))
    return len(records)
```

**Answer 9:**
```python
def process_table(**context):
    from airflow.models import Variable
    from airflow.providers.postgres.hooks.postgres import PostgresHook
    table = Variable.get('target_table', default_var='raw_orders')
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    count = hook.get_first(f"SELECT COUNT(*) FROM {table}")[0]
    print(f"Processing table {table}: {count} rows")
```

**Answer 10:** The password is hardcoded in the DAG file! Fix:
```python
def connect():
    from airflow.hooks.base import BaseHook
    conn = BaseHook.get_connection('makanexpress_prod_db')
    import psycopg2
    return psycopg2.connect(
        host=conn.host, database=conn.schema,
        user=conn.login, password=conn.password, port=conn.port
    )
```

**Answer 11:**
```python
def push_counts(**context):
    from airflow.providers.postgres.hooks.postgres import PostgresHook
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    ds = context['ds']
    total = hook.get_first("SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s", (ds,))[0]
    valid = hook.get_first("SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s AND status = 'completed'", (ds,))[0]
    
    context['ti'].xcom_push(key='total_count', value=total)
    context['ti'].xcom_push(key='valid_count', value=valid)
    context['ti'].xcom_push(key='invalid_count', value=total - valid)

def pull_counts(**context):
    total = context['ti'].xcom_pull(task_ids='push', key='total_count')
    valid = context['ti'].xcom_pull(task_ids='push', key='valid_count')
    invalid = context['ti'].xcom_pull(task_ids='push', key='invalid_count')
    print(f"Total: {total}, Valid: {valid}, Invalid: {invalid}")
```

**Answer 12:**
```python
def run_multi_sql(**context):
    from airflow.providers.postgres.hooks.postgres import PostgresHook
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    ds = context['ds']
    hook.run("""
        BEGIN;
        DELETE FROM staging_orders WHERE DATE(order_date) = '%s';
        INSERT INTO staging_orders SELECT * FROM raw_orders WHERE DATE(order_date) = '%s';
        COMMIT;
    """ % (ds, ds))
```

**Answer 13:**
```python
def smart_load(**context):
    from airflow.models import Variable
    from airflow.providers.postgres.hooks.postgres import PostgresHook
    
    batch_size = int(Variable.get('batch_size', default_var='1000'))
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    ds = context['ds']
    
    count = hook.get_first("SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s", (ds,))[0]
    print(f"Loading {count} records with batch_size={batch_size}")
```

**Answer 14:** Three possible causes:
1. Connection was never created (forgot to add it)
2. Typo in the connection name (case-sensitive!)
3. Connection was defined via environment variable but the env var isn't set

**Answer 15:**
```python
from airflow.hooks.base import BaseHook
conn = BaseHook.get_connection('makanexpress_db')
print(f"Host: {conn.host}, Port: {conn.port}, Login: {conn.login}, Schema: {conn.schema}")
# Don't print conn.password!
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Scheduling & Time Zones

### Concept 6: How Airflow Scheduling Works

```
Airflow scheduling is CONFUSING at first. Here's the key insight:

A DAG run with schedule='0 6 * * *' (6 AM daily):
- At 6 AM on June 13, Airflow creates a DAG run
- That DAG run's logical_date is June 12 (the PREVIOUS day)
- Why? Because at 6 AM on June 13, you're processing June 12's data!

The logical_date represents the DATA you're processing, not when you run.
```

```
Timeline:
June 12              June 13              June 14
  |                    |                    |
  |<--- data window --→|                    |
  |                    |<--- data window --→|
  |                    |                    |
  |                 6AM run              6AM run
  |                 (processes           (processes
  |                  June 12 data)        June 13 data)
```

**Key terms:**
- `logical_date`: The start of the data window (what data to process)
- `data_interval_start`: Same as logical_date for daily schedules
- `data_interval_end`: The end of the data window (usually logical_date + 1 day)
- Actual run time: When Airflow actually executes (after data_interval_end)

```python
# In your DAG, use data_interval for correct time windows:
@task
def process_data(**context):
    start = context['data_interval_start']  # e.g., 2026-06-12 00:00
    end = context['data_interval_end']      # e.g., 2026-06-13 00:00
    
    # Query data BETWEEN start and end (not just for one day!)
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    records = hook.get_records("""
        SELECT * FROM raw_orders
        WHERE order_date >= %s AND order_date < %s
    """, parameters=(start, end))
```

### Concept 7: Time Zones

```python
import pytz

# Run at 6 AM Singapore time every day
with DAG(
    dag_id='makanexpress_morning_etl',
    schedule='0 6 * * *',
    start_date=datetime(2026, 6, 12, tzinfo=pytz.timezone('Asia/Singapore')),
    timezone=pytz.timezone('Asia/Singapore'),  # makes schedule use SGT
    catchup=False,
) as dag:
    ...

# Without timezone, Airflow uses UTC
# 6 AM UTC = 2 PM SGT (wrong!)
# Always set timezone for SG/MY deployments
```

### Concept 8: Timetable (Custom Scheduling)

```python
# For complex schedules, use cron presets or custom timetables

# Every 4 hours during business hours (8 AM - 8 PM SGT)
schedule='0 0,4,8,12,16 * * *'  # UTC: 0,4,8,12,16 = SGT: 8,12,16,20,0

# Last day of month (not supported by standard cron!)
# Use a workaround with @monthly and skip if not last day:
from airflow.operators.python import ShortCircuitOperator
from datetime import datetime
import calendar

def is_last_day_of_month(**context):
    dt = context['logical_date']
    return dt.day == calendar.monthrange(dt.year, dt.month)[1]

check_last_day = ShortCircuitOperator(
    task_id='check_last_day',
    python_callable=is_last_day_of_month,
)
```

### Concept 9: Backfilling

```python
# Run a DAG for historical dates
airflow dags backfill makanexpress_etl \
    --start-date 2026-01-01 \
    --end-date 2026-03-31 \
    --reset-dagruns

# Useful when:
# 1. Adding a new DAG that should have been running for months
# 2. Fixing a bug and reprocessing affected dates
# 3. Loading historical data

# In DAG code, catchup=True means Airflow auto-backfills from start_date:
with DAG(
    catchup=True,  # Airflow creates runs for all missed dates!
    start_date=datetime(2026, 1, 1),
    max_active_runs=3,  # limit parallel backfills
) as dag:
    ...

# Backfill specific tasks only (not the whole DAG):
airflow dags backfill makanexpress_etl \
    --start-date 2026-06-01 \
    --end-date 2026-06-10 \
    --task-ids extract,load  # only these tasks
```

### Concept 10: Full Production-Ready DAG Pattern

```python
"""
MakanExpress Daily ETL — Production Pattern
A complete example showing all best practices.
"""

from datetime import datetime, timedelta
import pytz
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.models import Variable

# ─── Default Arguments ───
default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'email': ['data-team@makanexpress.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,  # 5min, 10min, 20min...
    'max_retry_delay': timedelta(hours=1),
}

# ─── DAG Definition ───
with DAG(
    dag_id='makanexpress_daily_etl',
    default_args=default_args,
    description='MakanExpress daily ETL: extract → validate → transform → load → quality check',
    schedule='0 6 * * *',  # 6 AM daily (set timezone!)
    start_date=datetime(2026, 6, 12, tzinfo=pytz.timezone('Asia/Singapore')),
    timezone=pytz.timezone('Asia/Singapore'),
    catchup=False,
    max_active_runs=1,  # don't overlap runs
    tags=['makanexpress', 'etl', 'production'],
    params={
        'dry_run': False,  # can override when triggering manually
    },
) as dag:

    # ─── Task 1: Extract ───
    def extract(**context):
        hook = PostgresHook(postgres_conn_id='makanexpress_db')
        start = context['data_interval_start']
        end = context['data_interval_end']
        
        count = hook.get_first("""
            SELECT COUNT(*) FROM raw_orders
            WHERE order_date >= %s AND order_date < %s
        """, parameters=(start, end))[0]
        
        if count == 0:
            raise ValueError(f"No orders found between {start} and {end}")
        
        print(f"✅ Extracted {count} orders")
        context['ti'].xcom_push(key='order_count', value=count)
        return count

    task_extract = PythonOperator(
        task_id='extract',
        python_callable=extract,
    )

    # ─── Task 2: Validate ───
    def validate(**context):
        hook = PostgresHook(postgres_conn_id='makanexpress_db')
        start = context['data_interval_start']
        end = context['data_interval_end']
        
        # Check for NULL values
        null_count = hook.get_first("""
            SELECT COUNT(*) FROM raw_orders
            WHERE order_date >= %s AND order_date < %s
            AND (customer_id IS NULL OR restaurant_id IS NULL OR total_amount IS NULL)
        """, parameters=(start, end))[0]
        
        if null_count > 0:
            raise ValueError(f"Found {null_count} orders with NULL values!")
        
        # Check for duplicate order IDs
        dupes = hook.get_first("""
            SELECT COUNT(*) FROM (
                SELECT order_id, COUNT(*) as cnt FROM raw_orders
                WHERE order_date >= %s AND order_date < %s
                GROUP BY order_id HAVING COUNT(*) > 1
            ) dupes
        """, parameters=(start, end))[0]
        
        if dupes > 0:
            raise ValueError(f"Found {dupes} duplicate order IDs!")
        
        print("✅ Validation passed")

    task_validate = PythonOperator(
        task_id='validate',
        python_callable=validate,
    )

    # ─── Task 3: Load Dimensions (SCD Type 2) ───
    task_load_dims = PostgresOperator(
        task_id='load_dimensions',
        postgres_conn_id='makanexpress_db',
        sql="""
            -- SCD Type 2: Expire changed customers
            UPDATE dim_customer SET valid_to = NOW(), is_current = FALSE
            FROM raw_customers s
            WHERE dim_customer.customer_id = s.customer_id
            AND dim_customer.is_current = TRUE
            AND (dim_customer.city IS DISTINCT FROM s.city);
            
            -- Insert new/changed customers
            INSERT INTO dim_customer (customer_id, customer_name, email, city, valid_from)
            SELECT s.customer_id, s.name, s.email, s.city, NOW()
            FROM raw_customers s
            WHERE NOT EXISTS (
                SELECT 1 FROM dim_customer d 
                WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
                AND d.city = s.city
            );
        """,
    )

    # ─── Task 4: Load Fact ───
    task_load_fact = PostgresOperator(
        task_id='load_fact',
        postgres_conn_id='makanexpress_db',
        sql="""
            INSERT INTO fact_order_items (order_number, date_key, customer_key, restaurant_key, item_key, quantity, unit_price, discount_amount, delivery_fee, order_status)
            SELECT 
                o.order_id,
                dd.date_key,
                dc.customer_key,
                dr.restaurant_key,
                di.item_key,
                oi.quantity,
                oi.unit_price,
                oi.discount,
                o.delivery_fee,
                o.status
            FROM raw_order_items oi
            JOIN raw_orders o ON oi.order_id = o.order_id
            JOIN dim_date dd ON dd.full_date = DATE(o.order_date)
            JOIN dim_customer dc ON dc.customer_id = o.customer_id AND dc.is_current = TRUE
            JOIN dim_restaurant dr ON dr.restaurant_id = o.restaurant_id AND dr.is_current = TRUE
            JOIN dim_item di ON di.item_id = oi.menu_item_id
            WHERE o.order_date >= '{{ data_interval_start }}'
            AND o.order_date < '{{ data_interval_end }}'
            AND NOT EXISTS (
                SELECT 1 FROM fact_order_items f 
                WHERE f.order_number = o.order_id
            );
        """,
    )

    # ─── Task 5: Quality Check ───
    def quality_check(**context):
        hook = PostgresHook(postgres_conn_id='makanexpress_db')
        
        orphans = hook.get_first("""
            SELECT COUNT(*) FROM fact_order_items f
            LEFT JOIN dim_customer d ON f.customer_key = d.customer_key
            WHERE d.customer_key IS NULL
        """)[0]
        
        if orphans > 0:
            raise ValueError(f"Found {orphans} orphan fact rows!")
        
        print("✅ Data quality check passed")

    task_quality = PythonOperator(
        task_id='quality_check',
        python_callable=quality_check,
    )

    # ─── Task 6: Notify ───
    def notify(**context):
        count = context['ti'].xcom_pull(task_ids='extract', key='order_count')
        ds = context['ds']
        print(f"""
        ╔══════════════════════════════════════╗
        ║   MakanExpress Daily ETL Report     ║
        ╠══════════════════════════════════════╣
        ║   Date: {ds:<28}║
        ║   Orders processed: {count:<16}║
        ║   Status: ✅ SUCCESS                  ║
        ╚══════════════════════════════════════╝
        """)

    task_notify = PythonOperator(
        task_id='notify',
        python_callable=notify,
        trigger_rule='all_success',
    )

    # ─── Dependencies ───
    task_extract >> task_validate >> task_load_dims >> task_load_fact >> task_quality >> task_notify
```

---

### 🏋️ Afternoon Exercises (15 questions)

16. Set up the `makanexpress_db` connection and test it. Run the production DAG from Concept 10.

17. Modify the production DAG to use TaskFlow API instead of traditional operators. Keep the same logic.

18. Write a DAG that processes data for the previous HOUR (runs hourly). Use `data_interval_start` and `data_interval_end` in the query.

19. Set up backfill: run `makanexpress_daily_etl` for all dates from June 1 to June 10. Verify each date processed correctly.

20. Create a DAG that runs at 9 AM Singapore time on weekdays only. Verify the schedule is correct.

21. Write a task that checks if it's a Singapore public holiday (read from a Variable containing holiday dates). If it is, skip the rest of the pipeline using `ShortCircuitOperator`.

22. Implement exponential backoff retry: a task that fails should retry after 1min, 2min, 4min. Use `retry_exponential_backoff=True`.

23. Create a Variable that stores the list of tables to process. Write a DAG that dynamically generates tasks — one per table.

24. Write a DAG with `max_active_runs=1`. Trigger it twice quickly. What happens to the second run?

25. **Debugging:** Your DAG runs at 6 AM but processes today's data instead of yesterday's. You're using `{{ ds }}` in your SQL. What's wrong?

26. Create a "dev" and "prod" version of the same DAG that uses different connections. Use Variables to switch between them.

27. Write a task that calculates the delta (new/changed records) between two runs. Use XCom to pass the previous run's max timestamp.

28. Create a DAG that uses both PostgresOperator (for SQL tasks) and PythonOperator (for logic tasks). Mix them in a single pipeline.

29. **Monitoring:** Write a DAG that runs every hour and checks if today's daily DAG has completed successfully. Alert if it hasn't by 8 AM.

30. **Challenge:** Create a complete production-grade Airflow setup:
    - Folder structure with dags/, scripts/, sql/, tests/
    - At least 3 DAGs (daily ETL, hourly check, weekly report)
    - Connections and Variables configured
    - Proper error handling and retries
    - Data quality checks
    - Notification on success/failure
    - README with setup instructions

---

### ✅ Afternoon Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 16:** Hands-on. Run the DAG and check task logs.

**Answer 17:**
```python
from airflow.decorators import dag, task
import pytz

@dag(
    schedule='0 6 * * *',
    start_date=datetime(2026, 6, 12, tzinfo=pytz.timezone('Asia/Singapore')),
    catchup=False,
)
def makanexpress_taskflow_etl():

    @task
    def extract(**context):
        from airflow.providers.postgres.hooks.postgres import PostgresHook
        hook = PostgresHook(postgres_conn_id='makanexpress_db')
        start, end = context['data_interval_start'], context['data_interval_end']
        count = hook.get_first(
            "SELECT COUNT(*) FROM raw_orders WHERE order_date >= %s AND order_date < %s",
            (start, end)
        )[0]
        if count == 0:
            raise ValueError("No orders!")
        return count

    @task
    def validate(count):
        print(f"Validating {count} orders...")

    @task
    def load(count, **context):
        print(f"Loading {count} orders for {context['ds']}")

    @task
    def notify(count, **context):
        print(f"✅ Processed {count} orders for {context['ds']}")

    count = extract()
    validate(count)
    load(count)
    notify(count)

makanexpress_taskflow_etl()
```

**Answer 18:**
```python
schedule='0 * * * *'  # every hour

# In task:
def process_hour(**context):
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    start = context['data_interval_start']
    end = context['data_interval_end']
    # data_interval is 1 hour for hourly schedule
    records = hook.get_records(
        "SELECT * FROM raw_orders WHERE order_date >= %s AND order_date < %s",
        (start, end)
    )
```

**Answer 19:**
```bash
airflow dags backfill makanexpress_daily_etl \
    --start-date 2026-06-01 \
    --end-date 2026-06-10
```

**Answer 20:**
```python
import pytz
with DAG(
    schedule='0 9 * * 1-5',
    start_date=datetime(2026, 6, 12, tzinfo=pytz.timezone('Asia/Singapore')),
    timezone=pytz.timezone('Asia/Singapore'),
    ...
):
```

**Answer 21:**
```python
from airflow.operators.python import ShortCircuitOperator

def is_working_day(**context):
    import json
    holidays = json.loads(Variable.get('sg_holidays', default_var='[]'))
    ds = context['ds']
    if ds in holidays:
        print(f"🎉 {ds} is a holiday — skipping pipeline")
        return False
    return True

check = ShortCircuitOperator(task_id='check_holiday', python_callable=is_working_day)
# downstream tasks only run if this returns True
check >> [extract, validate, ...]
```

**Answer 22:**
```python
default_args = {
    'retries': 3,
    'retry_delay': timedelta(minutes=1),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(minutes=10),
}
```

**Answer 23:**
```python
import json
from airflow import DAG
from airflow.operators.python import PythonOperator

tables = json.loads(Variable.get('etl_tables', default_var='["orders", "customers", "restaurants"]'))

with DAG('dynamic_tables', schedule='@daily', ...) as dag:
    for table in tables:
        PythonOperator(
            task_id=f'process_{table}',
            python_callable=process_table,
            op_kwargs={'table_name': table},
        )
```

**Answer 24:** The second run is queued and waits for the first to complete. `max_active_runs=1` prevents concurrent DAG runs.

**Answer 25:** Nothing is wrong! `{{ ds }}` IS the logical_date, which for a daily 6 AM job is yesterday's date. If you're seeing today's date, check your schedule and start_date configuration.

**Answer 26:**
```python
env = Variable.get('environment', default_var='dev')
conn_id = f'makanexpress_{env}_db'  # makanexpress_dev_db or makanexpress_prod_db
```

**Answer 27:**
```python
def get_delta(**context):
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    prev_max = context['ti'].xcom_pull(task_ids='get_delta', key='max_timestamp',
        include_prior_dates=True)  # pull from previous DAG run
    current_max = hook.get_first("SELECT MAX(order_date) FROM raw_orders")[0]
    context['ti'].xcom_push(key='max_timestamp', value=str(current_max))
    
    new_records = hook.get_records(
        "SELECT * FROM raw_orders WHERE order_date > %s",
        (prev_max or '1970-01-01',)
    )
    return len(new_records)
```

**Answer 28:** See the production DAG in Concept 10 — it already mixes both.

**Answer 29:**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator

def check_daily_dag(**context):
    from airflow.utils.session import create_session
    from airflow.models import DagRun
    ds = context['ds']
    
    with create_session() as session:
        dag_run = session.query(DagRun).filter(
            DagRun.dag_id == 'makanexpress_daily_etl',
            DagRun.logical_date == ds,
        ).first()
        
        if not dag_run:
            print(f"⚠️ Daily ETL hasn't started for {ds}!")
            return False
        
        if dag_run.state != 'success':
            print(f"⚠️ Daily ETL status: {dag_run.state} for {ds}")
            return False
        
        print(f"✅ Daily ETL completed for {ds}")
        return True

with DAG('hourly_dag_check', schedule='0 * * * *', ...) as dag:
    check = PythonOperator(task_id='check_daily', python_callable=check_daily_dag)
```

**Answer 30:** Hands-on — create the complete project structure.

</details>

---

## 🌙 Evening Block (1 hour): Review & Flashcards

### Connection vs Variable vs XCom

| What | Purpose | Example | Size Limit |
|------|---------|---------|------------|
| Connection | Credentials for external systems | DB host, user, password | N/A |
| Variable | Reusable configuration | batch_size, table_names | N/A |
| XCom | Pass data between tasks | order_count, file_path | 48KB |

### Jinja Template Cheat Sheet

| Template | Value | Use Case |
|----------|-------|----------|
| `{{ ds }}` | 2026-06-12 | Daily SQL queries |
| `{{ ds_nodash }}` | 20260612 | File names |
| `{{ data_interval_start }}` | 2026-06-11T00:00 | Range queries |
| `{{ data_interval_end }}` | 2026-06-12T00:00 | Range queries |
| `{{ logical_date }}` | Full datetime | Logging |
| `{{ params.x }}` | User parameter | Manual overrides |

### 📝 Today's Checklist

- [ ] I can create Airflow Connections via UI and CLI
- [ ] I can create and use Airflow Variables
- [ ] I understand Hooks and why they're better than raw connections
- [ ] I can use XCom for cross-task communication
- [ ] I understand logical_date vs actual run time
- [ ] I can write a production-ready DAG with error handling
- [ ] I know how to backfill historical data
- [ ] I pushed code to GitHub

### 📖 Further Reading (Free)

- [Airflow Connections](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html)
- [Airflow Variables](https://airflow.apache.org/docs/apache-airflow/stable/howto/variable.html)
- [Airflow XComs](https://airflow.apache.org/docs/apache-airflow/stable/concepts/xcoms.html)
- [Airflow Scheduling](https://airflow.apache.org/docs/apache-airflow/stable/scheduler.html)

---

*Day 29-30 complete! You now have a checkpoint — 2 days done. Ready for Day 31-33 (3-day batch)?* ✅
