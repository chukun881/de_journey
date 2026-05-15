# 📅 Day 29 — Thursday, 12 June 2026
# 🌬️ Airflow Basics: DAGs, Tasks, and Operators

---

## 🎯 Today's Goal

Apache Airflow is the **most popular workflow orchestration tool** in data engineering. Today you'll understand what it is, set it up locally, and write your first DAG. By end of day, you'll have Airflow running and executing pipelines on your machine.

**Why this matters:** Almost every data engineering job posting mentions Airflow. It's how companies schedule and monitor their ETL pipelines, dbt runs, and data quality checks.

---

## ☀️ Morning Block (2 hours): What is Airflow + Setup

### Concept 1: The Orchestration Problem

Imagine your food delivery pipeline:
```
1. Extract orders from API (6 AM daily)
2. Validate data (check row counts)
3. Load into staging table
4. Run dbt models
5. Run data quality tests
6. Send Slack notification with results
```

**Without orchestration:**
- Cron jobs that don't talk to each other
- No visibility into what succeeded/failed
- No retry logic
- No dependency management (what if step 1 is slow?)
- Hard to backfill historical data

**With Airflow:**
- Visual DAG (Directed Acyclic Graph) showing all steps
- Automatic retries on failure
- Dependency management (step 4 waits for step 3)
- Web UI to monitor, trigger, and debug
- Backfill historical runs with one command

### Concept 2: Core Airflow Concepts

```
DAG (Directed Acyclic Graph)
├── A collection of tasks with dependencies
├── Defined in a Python file
├── Has a schedule (e.g., daily at 6 AM)
└── "Directed" = tasks flow in one direction, "Acyclic" = no loops

Task
├── A single unit of work in a DAG
├── Defined by an Operator
└── Becomes a TaskInstance when scheduled for a specific date

Operator
├── A template for a task
├── Types:
│   ├── Action Operators (do something): PythonOperator, BashOperator
│   ├── Transfer Operators (move data): S3ToRedshiftOperator, PostgresOperator
│   └── Sensor Operators (wait for something): S3KeySensor, ExternalTaskSensor
└── You can also create custom operators

Task Instance
├── A specific run of a task for a specific date (logical_date)
└── Has states: queued, running, success, failed, skipped, up_for_retry
```

### Concept 3: Installing Airflow

```bash
# Create a dedicated directory for Airflow
mkdir -p ~/airflow-project
cd ~/airflow-project

# Create a virtual environment (IMPORTANT: Airflow should NOT be in system Python)
python3 -m venv venv
source venv/bin/activate

# Install Airflow with PostgreSQL and PostgreSQL provider
# ALWAYS set constraint file for your Python version!
AIRFLOW_VERSION=2.10.4
PYTHON_VERSION=$(python3 -c "import sys; print(f'{sys.version_info.major}.{sys.version_info.minor}')")
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

pip install "apache-airflow==${AIRFLOW_VERSION}" \
    "apache-airflow-providers-postgres" \
    --constraint "${CONSTRAINT_URL}"

# Initialize the database (SQLite by default for local dev)
airflow db init

# Create an admin user
airflow users create \
    --username admin \
    --password admin \
    --firstname Admin \
    --lastname User \
    --role Admin \
    --email admin@example.com

# Start the web server (port 8080)
airflow webserver --port 8080 &

# In another terminal, start the scheduler
# (Both must be running for Airflow to work!)
source venv/bin/activate
airflow scheduler

# Open http://localhost:8080 in your browser
# Login: admin / admin
```

**💡 Key thing:** Airflow needs TWO processes running:
1. **Webserver** — the UI (port 8080)
2. **Scheduler** — the brain that decides when to run tasks

### Concept 4: Your First DAG

Create the file `~/airflow-project/dags/hello_makanexpress.py`:

```python
"""
MakanExpress First DAG — Hello Airflow!
A simple DAG to verify Airflow is working.
"""

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

# Default arguments applied to ALL tasks in this DAG
default_args = {
    'owner': 'makanexpress',           # who owns this DAG
    'depends_on_past': False,          # don't wait for previous run
    'email_on_failure': False,         # we'll add email later
    'email_on_retry': False,
    'retries': 2,                      # retry twice on failure
    'retry_delay': timedelta(minutes=5),  # wait 5 min between retries
}

# Define the DAG
with DAG(
    dag_id='hello_makanexpress',
    default_args=default_args,
    description='My first Airflow DAG for MakanExpress',
    schedule='@daily',                 # run once per day
    start_date=datetime(2026, 6, 12),  # when to start running
    catchup=False,                     # don't backfill from start_date
    tags=['tutorial', 'makanexpress'],
) as dag:

    # Task 1: Print a greeting using BashOperator
    task_hello = BashOperator(
        task_id='say_hello',
        bash_command='echo "Hello from MakanExpress! Today is $(date)"',
    )

    # Task 2: Print current time
    task_time = BashOperator(
        task_id='show_time',
        bash_command='date',
    )

    # Task 3: Python function
    def print_menu():
        menu = [
            "🍗 Hainanese Chicken Rice — $6.50",
            "🍛 Nasi Lemak Special — $8.50",
            "🫓 Roti Prata Egg — $3.50",
            "🍣 Salmon Sashimi Set — $18.00",
        ]
        print("📋 Today's Menu:")
        for item in menu:
            print(f"  {item}")
        return "Menu printed successfully"

    task_menu = PythonOperator(
        task_id='print_menu',
        python_callable=print_menu,
    )

    # Define task dependencies (order of execution)
    # task_hello runs first, then task_time and task_menu run in parallel
    task_hello >> [task_time, task_menu]
```

**Understanding the code:**

```python
# DAG definition — the "container" for all tasks
with DAG(
    dag_id='hello_makanexpress',   # unique name across all DAGs
    schedule='@daily',             # cron expression or preset
    start_date=datetime(2026, 6, 12),  # earliest date to run
    catchup=False,                 # CRITICAL: don't run all missed dates!
) as dag:

# Task using BashOperator — runs a shell command
task_hello = BashOperator(
    task_id='say_hello',           # unique within this DAG
    bash_command='echo "Hello"',   # the command to run
)

# Task using PythonOperator — calls a Python function
def my_function():
    return "done"

task_python = PythonOperator(
    task_id='run_python',
    python_callable=my_function,   # the function to call
)

# Dependencies — order matters!
task_hello >> task_time        # task_time runs AFTER task_hello
task_hello >> task_menu        # task_menu also runs AFTER task_hello
# Shorter syntax:
task_hello >> [task_time, task_menu]  # both run after hello (parallel)

# Other dependency syntax:
# task_a << task_b   means task_b runs before task_a
# task_a >> task_b >> task_c   means a → b → c (sequential)
# [task_a, task_b] >> task_c   means c runs after both a and b
```

### Concept 5: Schedule Presets

```python
# Preset schedules:
schedule='@daily'          # Every day at midnight
schedule='@hourly'         # Every hour
schedule='@weekly'         # Every Sunday at midnight
schedule='@monthly'        # First day of month at midnight
schedule='@yearly'         # January 1st at midnight
schedule=None              # Manual trigger only (no automatic scheduling)
schedule='0 6 * * *'       # Cron: every day at 6 AM
schedule='0 6 * * 1-5'     # Cron: weekdays at 6 AM
schedule='*/15 * * * *'    # Cron: every 15 minutes
schedule='0 0 1 */3 *'     # Cron: first day of every quarter

# Common cron patterns for data engineering:
# Daily ETL:           '0 6 * * *'       — 6 AM daily
# Hourly ingestion:    '0 * * * *'       — every hour
# Weekday reports:     '0 9 * * 1-5'     — 9 AM Mon-Fri
# Monthly close:       '0 2 1 * *'       — 2 AM on 1st of month
# Weekend batch:       '0 4 * * 6,0'     — 4 AM Sat & Sun
```

---

### 🏋️ Morning Exercises (15 questions)

1. Explain in your own words: what problem does Airflow solve that cron doesn't?

2. Install Airflow locally following the setup steps. Verify both webserver and scheduler are running. Take a screenshot of the Airflow UI.

3. What does DAG stand for? Why must it be "acyclic"? What would happen if there were a cycle?

4. Create the `hello_makanexpress` DAG and verify it appears in the Airflow UI. Trigger it manually and check the logs.

5. What is the difference between a DAG, a Task, and a Task Instance?

6. Modify the DAG to add a 4th task that prints the current working directory using BashOperator.

7. What does `catchup=False` do? Set it to `True` with a start_date of 30 days ago. How many DAG runs does Airflow try to execute? (Don't actually run them!)

8. Write a DAG with 5 tasks that execute in this order:
    ```
    extract >> validate >> [load_staging, send_alert] >> notify_complete
    ```

9. Change the schedule of your DAG to run every 15 minutes. Verify in the UI that the schedule updated.

10. What happens if a task in the middle of your pipeline fails? Do the downstream tasks run?

11. Write a PythonOperator task that generates a random number and prints it. Use `import random`.

12. Create a DAG that runs only on weekdays at 9 AM Singapore time. What schedule string do you use?

13. In the Airflow UI, find the "Graph" view and "Tree" view of your DAG. What does each show?

14. What is the difference between `logical_date` (previously `execution_date`) and the actual run time?

15. **Practical:** Create a DAG called `makanexpress_daily_check` with 3 tasks:
    - Check if today is a holiday (print SG/MY holiday check)
    - Print "It's a working day!" or "Holiday — enjoy!"
    - List all files in the Airflow dags directory

---

### ✅ Morning Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1:** Airflow provides: dependency management (task B waits for task A), automatic retries, visual monitoring UI, backfill capability, and alerting. Cron just runs commands on a schedule with no awareness of success/failure or dependencies.

**Answer 2:** Hands-on. Verify at http://localhost:8080.

**Answer 3:** Directed Acyclic Graph. "Acyclic" means no cycles/loops — tasks flow in one direction only. If there were a cycle, Airflow would never finish executing (infinite loop). Airflow validates this at DAG parsing time and rejects cyclic DAGs.

**Answer 4:** Hands-on. Trigger via UI (▶️ button) or CLI: `airflow dags trigger hello_makanexpress`.

**Answer 5:**
- **DAG**: The entire workflow definition (the recipe)
- **Task**: One step in the DAG (one instruction in the recipe)
- **Task Instance**: A specific run of a task for a specific date (one execution of that instruction)

**Answer 6:**
```python
task_pwd = BashOperator(
    task_id='print_working_dir',
    bash_command='pwd',
)
task_hello >> [task_time, task_menu, task_pwd]
```

**Answer 7:** `catchup=False` means Airflow only runs from NOW. `catchup=True` with a start_date 30 days ago means Airflow tries to run 30 missed DAG runs (one per day), which can be dangerous on production!

**Answer 8:**
```python
extract = BashOperator(task_id='extract', bash_command='echo "Extracting..."')
validate = BashOperator(task_id='validate', bash_command='echo "Validating..."')
load_staging = BashOperator(task_id='load_staging', bash_command='echo "Loading staging..."')
send_alert = BashOperator(task_id='send_alert', bash_command='echo "Sending alert..."')
notify = BashOperator(task_id='notify_complete', bash_command='echo "Done!"')

extract >> validate >> [load_staging, send_alert] >> notify
```

**Answer 9:**
```python
schedule='*/15 * * * *'
```
Check UI → the "Schedule" column should show "Every 15 minutes."

**Answer 10:** No. Downstream tasks are skipped if an upstream task fails. This is by design — you don't want to load data that wasn't properly extracted.

**Answer 11:**
```python
def generate_random():
    import random
    number = random.randint(1, 100)
    print(f"Random number: {number}")
    return number

task_random = PythonOperator(
    task_id='random_number',
    python_callable=generate_random,
)
```

**Answer 12:**
```python
schedule='0 9 * * 1-5'  # 9 AM Mon-Fri
# Note: Airflow uses UTC by default. Singapore is UTC+8.
# For 9 AM SGT, use 1 AM UTC:
schedule='0 1 * * 1-5'  # 1 AM UTC = 9 AM SGT
# Or set timezone in DAG:
import pytz
with DAG(..., timezone=pytz.timezone('Asia/Singapore'), schedule='0 9 * * 1-5'):
```

**Answer 13:**
- **Graph view**: Shows the DAG structure as a flowchart (boxes and arrows). Best for understanding dependencies.
- **Tree view**: Shows a grid of runs over time. Each column is a date, each row is a task. Color-coded by status. Best for seeing patterns of failures.

**Answer 14:**
- `logical_date` (formerly `execution_date`): The date this DAG run is FOR (e.g., "process data for June 12th")
- Actual run time: When the task actually executes (could be June 13th at 6 AM for a daily job processing June 12th's data)
- This distinction matters: a daily job at 6 AM on June 13 processes data FROM June 12. The logical_date is June 12.

**Answer 15:**
```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

def check_holiday():
    from datetime import date
    sg_holidays_2026 = [
        '2026-01-01', '2026-02-17', '2026-02-18',  # New Year, CNY
        '2026-05-01', '2026-05-31',  # Labour Day, Vesak Day
        '2026-08-09', '2026-12-25',  # National Day, Christmas
    ]
    today = date.today().isoformat()
    is_holiday = today in sg_holidays_2026
    print(f"Today: {today}, Holiday: {is_holiday}")
    return is_holiday

def holiday_message():
    from datetime import date
    sg_holidays_2026 = ['2026-01-01', '2026-02-17', '2026-02-18',
                         '2026-05-01', '2026-05-31', '2026-08-09', '2026-12-25']
    today = date.today().isoformat()
    if today in sg_holidays_2026:
        print("🎉 Holiday — enjoy your day off!")
    else:
        print("💼 It's a working day!")

with DAG(
    dag_id='makanexpress_daily_check',
    schedule='@daily',
    start_date=datetime(2026, 6, 12),
    catchup=False,
) as dag:

    task_check = PythonOperator(task_id='check_holiday', python_callable=check_holiday)
    task_message = PythonOperator(task_id='holiday_message', python_callable=holiday_message)
    task_list = BashOperator(task_id='list_dags', bash_command='ls $AIRFLOW_HOME/dags')

    task_check >> task_message
    task_list  # runs independently (no dependency on others)
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Operators Deep Dive

### Concept 6: BashOperator — Running Shell Commands

```python
from airflow.operators.bash import BashOperator

# Simple command
task = BashOperator(
    task_id='run_script',
    bash_command='python /path/to/script.py',
)

# With environment variables
task = BashOperator(
    task_id='run_etl',
    bash_command='python /scripts/etl.py --date {{ ds }}',
    # {{ ds }} is a Jinja template = logical_date as YYYY-MM-DD
)

# Run a shell script
task = BashOperator(
    task_id='run_shell_script',
    bash_command='/scripts/daily_load.sh ',
    # NOTE: trailing space prevents Airflow from treating it as a template file!
)

# With Jinja templates (very powerful!)
task = BashOperator(
    task_id='process_data',
    bash_command="""
        echo "Processing data for {{ ds }}"
        echo "Logical date: {{ logical_date }}"
        echo "Tomorrow: {{ tomorrow_ds }}"
        echo "Yesterday: {{ yesterday_ds }}"
        echo "Data interval start: {{ data_interval_start }}"
        echo "Data interval end: {{ data_interval_end }}"
    """,
)
```

**Common Jinja templates:**

| Template | Value | Example |
|----------|-------|---------|
| `{{ ds }}` | logical_date as YYYY-MM-DD | `2026-06-12` |
| `{{ ds_nodash }}` | YYYYMMDD (great for filenames) | `20260612` |
| `{{ yesterday_ds }}` | Previous day | `2026-06-11` |
| `{{ tomorrow_ds }}` | Next day | `2026-06-13` |
| `{{ logical_date }}` | Full datetime object | `2026-06-12T00:00:00+00:00` |
| `{{ data_interval_start }}` | Start of data window | `2026-06-11T00:00:00+00:00` |
| `{{ data_interval_end }}` | End of data window | `2026-06-12T00:00:00+00:00` |

### Concept 7: PythonOperator — Running Python Functions

```python
from airflow.operators.python import PythonOperator

# Basic Python function
def process_orders(**context):
    """Process orders for a specific date."""
    # Access Airflow context variables
    logical_date = context['logical_date']
    ds = context['ds']  # shorthand
    
    print(f"Processing orders for {ds}")
    
    # Your actual logic here
    import psycopg2
    conn = psycopg2.connect(
        host='localhost',
        database='makanexpress',
        user='postgres',
    )
    cursor = conn.cursor()
    cursor.execute(f"""
        SELECT COUNT(*) FROM raw_orders 
        WHERE DATE(order_date) = '{ds}'
    """)
    count = cursor.fetchone()[0]
    print(f"Found {count} orders for {ds}")
    conn.close()
    
    return count  # Return value available via XCom

task = PythonOperator(
    task_id='process_orders',
    python_callable=process_orders,
)

# With arguments
def load_data(target_table, batch_size=1000, **context):
    ds = context['ds']
    print(f"Loading {target_table} for {ds} with batch_size={batch_size}")

task = PythonOperator(
    task_id='load_data',
    python_callable=load_data,
    op_kwargs={
        'target_table': 'fact_order_items',
        'batch_size': 5000,
    },
)

# Using op_args (positional arguments)
task = PythonOperator(
    task_id='load_data',
    python_callable=load_data,
    op_args=['fact_order_items'],  # positional
    op_kwargs={'batch_size': 5000},  # keyword
)
```

### Concept 8: PostgresOperator — Running SQL Directly

```python
from airflow.providers.postgres.operators.postgres import PostgresOperator

# Run a SQL command
task = PostgresOperator(
    task_id='create_staging_table',
    postgres_conn_id='makanexpress_db',  # connection defined in Airflow UI
    sql="""
        CREATE TABLE IF NOT EXISTS staging_orders (
            order_id INT,
            customer_id INT,
            restaurant_id INT,
            order_date TIMESTAMP,
            total_amount DECIMAL(10,2),
            status VARCHAR(20)
        );
        
        TRUNCATE TABLE staging_orders;
    """,
)

# Run SQL from a file
task = PostgresOperator(
    task_id='run_etl',
    postgres_conn_id='makanexpress_db',
    sql='/scripts/sql/load_fact.sql',  # path relative to DAG
)

# With Jinja templates in SQL
task = PostgresOperator(
    task_id='load_daily_orders',
    postgres_conn_id='makanexpress_db',
    sql="""
        INSERT INTO fact_order_items (date_key, ...)
        SELECT ...
        FROM raw_orders o
        WHERE DATE(o.order_date) = '{{ ds }}'
        AND NOT EXISTS (
            SELECT 1 FROM fact_order_items f 
            WHERE f.order_date = '{{ ds }}'
        );
    """,
)
```

### Concept 9: TaskFlow API — Modern Pythonic DAGs

```python
"""
TaskFlow API (Airflow 2.0+) — cleaner way to write Python DAGs.
Uses decorators instead of operators.
"""

from datetime import datetime
from airflow.decorators import dag, task

@dag(
    dag_id='makanexpress_taskflow',
    schedule='@daily',
    start_date=datetime(2026, 6, 12),
    catchup=False,
    tags=['makanexpress', 'taskflow'],
)
def makanexpress_pipeline():

    @task
    def extract_orders(**context):
        """Extract orders from source for today."""
        ds = context['ds']
        # Simulate extraction
        orders = [
            {'id': 1, 'amount': 25.50, 'restaurant': 'Chicken Rice Paradise'},
            {'id': 2, 'amount': 18.00, 'restaurant': 'Sakura Sushi'},
            {'id': 3, 'amount': 8.50, 'restaurant': 'Nasi Lemak House'},
        ]
        print(f"Extracted {len(orders)} orders for {ds}")
        return orders  # Automatically pushed to XCom

    @task
    def validate_orders(orders):
        """Validate orders — filter out negative amounts."""
        valid = [o for o in orders if o['amount'] > 0]
        print(f"Valid: {len(valid)}/{len(orders)} orders")
        return valid

    @task
    def transform_orders(orders):
        """Add derived fields."""
        for order in orders:
            order['gst'] = round(order['amount'] * 0.08, 2)  # 8% GST in SG
            order['total_with_gst'] = round(order['amount'] + order['gst'], 2)
        return orders

    @task
    def load_orders(orders, **context):
        """Load to database."""
        ds = context['ds']
        print(f"Loading {len(orders)} orders for {ds} to database")
        # In real code: database INSERT here
        for o in orders:
            print(f"  Order {o['id']}: ${o['total_with_gst']} (incl GST)")
        return len(orders)

    @task
    def send_report(count, **context):
        """Send summary notification."""
        ds = context['ds']
        print(f"📊 Report for {ds}: {count} orders loaded successfully")

    # Define dependencies (clean, functional style!)
    orders = extract_orders()
    valid = validate_orders(orders)
    transformed = transform_orders(valid)
    count = load_orders(transformed)
    send_report(count)

# Instantiate the DAG
makanexpress_pipeline()
```

### Concept 10: Common DAG Patterns

```python
# Pattern 1: Sequential (simplest)
task_a >> task_b >> task_c >> task_d

# Pattern 2: Parallel branching
task_start >> [task_x, task_y, task_z] >> task_end

# Pattern 3: Fan-out, fan-in
task_extract >> [task_load_customers, task_load_orders, task_load_items] >> task_validate

# Pattern 4: Conditional (with BranchPythonOperator)
from airflow.operators.python import BranchPythonOperator

def choose_branch(**context):
    """Decide which task to run next based on some condition."""
    from datetime import date
    if date.today().weekday() < 5:  # weekday
        return 'task_weekday_load'
    else:
        return 'task_weekend_skip'

branch = BranchPythonOperator(
    task_id='branch',
    python_callable=choose_branch,
)

task_weekday = BashOperator(task_id='task_weekday_load', bash_command='echo "Full load"')
task_weekend = BashOperator(task_id='task_weekend_skip', bash_command='echo "Skip"')

branch >> [task_weekday, task_weekend]
```

---

### 🏋️ Afternoon Exercises (20 questions)

16. Create a DAG with 3 BashOperator tasks that run sequentially: `extract_data`, `transform_data`, `load_data`. Each should echo the task name and `{{ ds }}`.

17. Write a PythonOperator task that connects to your PostgreSQL database and counts orders for `{{ ds }}`. Print the count.

18. Create a DAG that uses both BashOperator and PythonOperator. The BashOperator creates a directory, the PythonOperator writes a file to it.

19. Write a PostgresOperator task that truncates a staging table and inserts data for `{{ ds }}`. Define the connection in Airflow UI first.

20. Rewrite the DAG from exercise 16 using TaskFlow API (decorators). Compare the code length.

21. Create a DAG with a fan-out/fan-in pattern:
    - `extract_all` → [`load_customers`, `load_restaurants`, `load_items`] → `validate_all`

22. Use Jinja templates to create a DAG that:
    - Creates a file named `orders_{{ ds_nodash }}.csv`
    - Writes "Report for {{ ds }}" into it
    - Shows the file contents

23. Write a PythonOperator that takes `op_kwargs` with `table_name` and `batch_size` parameters. Use them in the function.

24. Create a DAG that runs only on the 1st of every month. What schedule do you use?

25. Write a BranchPythonOperator that checks if today is a weekend. If weekday → run `full_load`, if weekend → run `light_load`.

26. **Debugging exercise:** This DAG has 3 errors. Find and fix them:
```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

dag = DAG('buggy_dag', start_date=datetime(2026, 6, 12))

task1 = BashOperator(task_id='task1', bash_command='echo hello', dag=dag)
task2 = BashOperator(task_id='task2', bash_command='exit 1', dag=dag)  # This will fail!
task3 = BashOperator(task_id='task3', bash_command='echo done', dag=dag)

task1 >> task3
task2 >> task3
```
What happens when task2 fails? How do you make task3 run regardless?

27. Create a DAG with `retries=3` and `retry_delay=timedelta(minutes=2)`. Verify that a failing task retries 3 times by checking the logs.

28. Write a DAG that uses `params` to pass a configurable value:
```python
with DAG(..., params={'city': 'Singapore'}) as dag:
    task = BashOperator(
        task_id='show_param',
        bash_command='echo "City: {{ params.city }}"',
    )
```

29. Create a DAG that generates a daily report file with the date in the filename, using `{{ ds_nodash }}`.

30. **Practical:** Build a complete ETL DAG for MakanExpress:
    - Task 1: Extract — count orders from PostgreSQL for `{{ ds }}`
    - Task 2: Validate — check if count > 0 (fail if no orders)
    - Task 3: Transform — calculate daily revenue by restaurant
    - Task 4: Load — insert results into a summary table
    - Task 5: Notify — print summary
    Use PythonOperator for all tasks.

31. What is the difference between `schedule_interval` (deprecated) and `schedule` (new)?

32. Create a DAG that has no schedule (`schedule=None`) and can only be triggered manually. Why would you want this?

33. Write a DAG that processes data for the PREVIOUS day (use `{{ yesterday_ds }}` or `data_interval_start`). Explain why this is the common pattern.

34. Experiment: Set `catchup=True` and `start_date=datetime(2026, 1, 1)`. How many DAG runs appear in the UI? (Don't trigger them all!) Now set `catchup=False`.

35. **Challenge:** Create a complete Airflow project with folder structure:
    ```
    airflow-project/
    ├── dags/
    │   ├── makanexpress_etl.py
    │   └── makanexpress_daily_check.py
    ├── scripts/
    │   ├── extract.py
    │   └── transform.py
    ├── sql/
    │   ├── create_tables.sql
    │   └── load_fact.sql
    ├── requirements.txt
    └── README.md
    ```
    The main DAG should orchestrate the full ETL pipeline using tasks from scripts/ and sql/.

---

### ✅ Afternoon Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 16:**
```python
from datetime import datetime
from airflow import DAG
from airflow.operators.bash import BashOperator

with DAG('simple_etl', schedule='@daily', start_date=datetime(2026, 6, 12), catchup=False) as dag:
    extract = BashOperator(task_id='extract_data', bash_command='echo "Extracting data for {{ ds }}"')
    transform = BashOperator(task_id='transform_data', bash_command='echo "Transforming data for {{ ds }}"')
    load = BashOperator(task_id='load_data', bash_command='echo "Loading data for {{ ds }}"')
    
    extract >> transform >> load
```

**Answer 17:**
```python
def count_orders(**context):
    import psycopg2
    ds = context['ds']
    conn = psycopg2.connect(host='localhost', database='makanexpress', user='postgres')
    cur = conn.cursor()
    cur.execute("SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s", (ds,))
    count = cur.fetchone()[0]
    print(f"Orders for {ds}: {count}")
    conn.close()
    return count

task = PythonOperator(task_id='count_orders', python_callable=count_orders)
```

**Answer 18:**
```python
def write_file(**context):
    ds = context['ds']
    with open(f'/tmp/airflow_output_{ds}.txt', 'w') as f:
        f.write(f"Data processed for {ds}")
    print(f"File written for {ds}")

with DAG('bash_python_mix', schedule='@daily', start_date=datetime(2026, 6, 12), catchup=False) as dag:
    mkdir = BashOperator(task_id='create_dir', bash_command='mkdir -p /tmp/airflow_output')
    write = PythonOperator(task_id='write_file', python_callable=write_file)
    mkdir >> write
```

**Answer 19:** First, add connection in Airflow UI: Admin → Connections → Add:
- Conn Id: `makanexpress_db`
- Conn Type: `PostgreSQL`
- Host: `localhost`
- Database: `makanexpress`
- Login: `postgres`

```python
from airflow.providers.postgres.operators.postgres import PostgresOperator

task = PostgresOperator(
    task_id='load_daily',
    postgres_conn_id='makanexpress_db',
    sql="""
        TRUNCATE staging_daily_orders;
        INSERT INTO staging_daily_orders
        SELECT * FROM raw_orders WHERE DATE(order_date) = '{{ ds }}';
    """,
)
```

**Answer 20:**
```python
from airflow.decorators import dag, task
from datetime import datetime

@dag(schedule='@daily', start_date=datetime(2026, 6, 12), catchup=False)
def simple_etl_taskflow():
    @task
    def extract():
        print("Extracting data for {{ ds }}")
    
    @task
    def transform():
        print("Transforming data")
    
    @task
    def load():
        print("Loading data")
    
    extract() >> transform() >> load()

simple_etl_taskflow()
```

**Answer 21:**
```python
with DAG('fan_out_in', schedule='@daily', start_date=datetime(2026, 6, 12), catchup=False) as dag:
    extract = BashOperator(task_id='extract_all', bash_command='echo "Extract all sources"')
    load_cust = BashOperator(task_id='load_customers', bash_command='echo "Load customers"')
    load_rest = BashOperator(task_id='load_restaurants', bash_command='echo "Load restaurants"')
    load_items = BashOperator(task_id='load_items', bash_command='echo "Load items"')
    validate = BashOperator(task_id='validate_all', bash_command='echo "Validate"')
    
    extract >> [load_cust, load_rest, load_items] >> validate
```

**Answer 22:**
```python
with DAG('file_creation', schedule='@daily', start_date=datetime(2026, 6, 12), catchup=False) as dag:
    create = BashOperator(
        task_id='create_file',
        bash_command='echo "Report for {{ ds }}" > /tmp/orders_{{ ds_nodash }}.csv',
    )
    show = BashOperator(
        task_id='show_file',
        bash_command='cat /tmp/orders_{{ ds_nodash }}.csv',
    )
    create >> show
```

**Answer 23:**
```python
def load_with_params(table_name, batch_size, **context):
    ds = context['ds']
    print(f"Loading {table_name} for {ds} with batch_size={batch_size}")

task = PythonOperator(
    task_id='load_data',
    python_callable=load_with_params,
    op_kwargs={'table_name': 'fact_sales', 'batch_size': 5000},
)
```

**Answer 24:** `schedule='0 0 1 * *'` (midnight on the 1st of every month)

**Answer 25:**
```python
from airflow.operators.python import BranchPythonOperator
from airflow.operators.bash import BashOperator

def check_weekend():
    from datetime import date
    return 'light_load' if date.today().weekday() >= 5 else 'full_load'

branch = BranchPythonOperator(task_id='check_day', python_callable=check_weekend)
full = BashOperator(task_id='full_load', bash_command='echo "Full load"')
light = BashOperator(task_id='light_load', bash_command='echo "Light load"')
branch >> [full, light]
```

**Answer 26:** The 3 issues:
1. `exit 1` in task2 intentionally fails — this is fine for testing
2. task3 depends on BOTH task1 and task2 — if task2 fails, task3 won't run
3. To fix: use `trigger_rule='all_done'` on task3 so it runs regardless:
```python
task3 = BashOperator(
    task_id='task3', 
    bash_command='echo done', 
    dag=dag,
    trigger_rule='all_done',  # runs even if upstream fails
)
```
Or use `trigger_rule='one_success'` to run if at least one upstream succeeds.

**Answer 27:**
```python
with DAG('retry_test', default_args={
    'retries': 3, 
    'retry_delay': timedelta(minutes=2),
}, ...) as dag:
    failing_task = BashOperator(
        task_id='always_fail',
        bash_command='exit 1',
    )
```
Check in UI: task retries 3 times, then shows "failed" state. Logs show 4 attempts (1 initial + 3 retries).

**Answer 28:** Already provided in the question. Verify by triggering the DAG and checking logs.

**Answer 29:**
```python
BashOperator(
    task_id='generate_report',
    bash_command='echo "Daily report" > /tmp/report_{{ ds_nodash }}.txt',
)
```

**Answer 30:**
```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator

default_args = {
    'owner': 'makanexpress',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

def extract_orders(**context):
    import psycopg2
    ds = context['ds']
    conn = psycopg2.connect(host='localhost', database='makanexpress', user='postgres')
    cur = conn.cursor()
    cur.execute("SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s", (ds,))
    count = cur.fetchone()[0]
    conn.close()
    if count == 0:
        raise ValueError(f"No orders found for {ds}")
    print(f"Extracted {count} orders for {ds}")
    return count

def validate_orders(**context):
    # Pull XCom from extract task
    count = context['ti'].xcom_pull(task_ids='extract')
    print(f"Validating {count} orders...")
    if count < 0:
        raise ValueError("Invalid order count")

def transform_orders(**context):
    import psycopg2
    ds = context['ds']
    conn = psycopg2.connect(host='localhost', database='makanexpress', user='postgres')
    cur = conn.cursor()
    cur.execute("""
        SELECT r.restaurant_name, SUM(o.total_amount) as revenue
        FROM raw_orders o
        JOIN raw_restaurants r ON o.restaurant_id = r.restaurant_id
        WHERE DATE(o.order_date) = %s AND o.status = 'completed'
        GROUP BY r.restaurant_name
    """, (ds,))
    results = cur.fetchall()
    conn.close()
    print(f"Transformed data for {len(results)} restaurants")
    return results

def load_summary(**context):
    results = context['ti'].xcom_pull(task_ids='transform')
    ds = context['ds']
    import psycopg2
    conn = psycopg2.connect(host='localhost', database='makanexpress', user='postgres')
    cur = conn.cursor()
    for restaurant_name, revenue in results:
        cur.execute("""
            INSERT INTO daily_revenue_summary (date, restaurant_name, revenue)
            VALUES (%s, %s, %s)
            ON CONFLICT (date, restaurant_name) DO UPDATE SET revenue = EXCLUDED.revenue
        """, (ds, restaurant_name, revenue))
    conn.commit()
    conn.close()
    print(f"Loaded {len(results)} records")

def notify(**context):
    count = context['ti'].xcom_pull(task_ids='extract')
    ds = context['ds']
    print(f"📊 Daily Report for {ds}: {count} orders processed successfully")

with DAG('makanexpress_etl', default_args=default_args,
         schedule='@daily', start_date=datetime(2026, 6, 12), catchup=False) as dag:
    
    t_extract = PythonOperator(task_id='extract', python_callable=extract_orders)
    t_validate = PythonOperator(task_id='validate', python_callable=validate_orders)
    t_transform = PythonOperator(task_id='transform', python_callable=transform_orders)
    t_load = PythonOperator(task_id='load', python_callable=load_summary)
    t_notify = PythonOperator(task_id='notify', python_callable=notify)
    
    t_extract >> t_validate >> t_transform >> t_load >> t_notify
```

**Answer 31:** `schedule_interval` is the old name (Airflow 1.x), `schedule` is the new name (Airflow 2.x). They do the same thing. Use `schedule`.

**Answer 32:**
```python
schedule=None  # No automatic scheduling, manual trigger only
```
Use for: one-off loads, manual backfills, ad-hoc jobs, migration scripts.

**Answer 33:** Most ETL jobs process YESTERDAY's data because today isn't over yet. `{{ ds }}` = logical_date = the day this run is FOR. For a daily job running at 6 AM on June 13, `{{ ds }}` = June 12, which is correct — process yesterday's data. You can also use `{{ data_interval_start }}` and `{{ data_interval_end }}` for explicit windowing.

**Answer 34:** With `catchup=True` and start_date of Jan 1, Airflow creates ~162 DAG runs (Jan 1 to Jun 12). With `catchup=False`, only future runs are created.

**Answer 35:** Hands-on — create the folder structure and implement the DAGs.

</details>

---

## 🌙 Evening Block (1 hour): Review & Practice

### Airflow Concept Cheat Sheet

| Concept | What It Is |
|---------|-----------|
| DAG | Workflow definition (Python file) |
| Task | One step in a DAG |
| Task Instance | One run of a task for a specific date |
| Operator | Template for a task (Bash, Python, Postgres, etc.) |
| Scheduler | Process that decides when to run DAGs |
| Webserver | The UI (port 8080) |
| Executor | How tasks are actually run (SequentialExecutor, LocalExecutor) |
| XCom | Cross-task communication (small data) |
| Connection | Stored credentials for external systems |
| Variable | Stored key-value configuration |
| Catchup | Whether to run missed DAG runs from start_date |

### 📝 Today's Checklist

- [ ] Airflow is installed and running locally
- [ ] I can access the Airflow UI at localhost:8080
- [ ] I understand DAG, Task, Operator, and Task Instance
- [ ] I've written and run at least 3 different DAGs
- [ ] I can use BashOperator, PythonOperator, and PostgresOperator
- [ ] I understand Jinja templates ({{ ds }}, {{ ds_nodash }})
- [ ] I know how to define task dependencies (>>, <<)
- [ ] I understand TaskFlow API (decorators)
- [ ] I pushed code to GitHub

### 📖 Further Reading (Free)

- [Airflow Official Tutorial](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/)
- [Airflow Concepts](https://airflow.apache.org/docs/apache-airflow/stable/concepts.html)
- [Marc Lamberti's Airflow Course (free YouTube series)](https://www.youtube.com/playlist?list=PLLYz5u7kKx3YxF3MpoWTHyUfJn6i7KZ3J)

---

*Day 29 complete! Tomorrow: Airflow Scheduling, Connections, and Variables — making DAGs production-ready.* 🔧
