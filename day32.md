# 📅 Day 32 — Sunday, 15 June 2026
# 🔍 Airflow Sensors, Branching, and Advanced Patterns

---

## 🎯 Today's Goal

Yesterday's pipeline was linear. Real pipelines need to **wait for things** (data arriving, external systems), **make decisions** (different logic for different conditions), and **handle complex flows**. Today you'll learn sensors, branching, and advanced DAG patterns.

---

## ☀️ Morning Block (2 hours): Sensors

### Concept 1: What are Sensors?

Sensors are operators that **wait** for something to happen before proceeding. They keep checking until a condition is met or they time out.

```python
from airflow.sensors.base import BaseSensorOperator

# Built-in sensors:
# - FileSensor: wait for a file to exist
# - S3KeySensor: wait for an S3 object
# - SqlSensor: wait for a SQL condition
# - ExternalTaskSensor: wait for another DAG to complete
# - TimeSensor: wait until a specific time
# - PythonSensor: wait for a Python condition
```

### Concept 2: SqlSensor — Wait for Data

```python
from airflow.sensors.sql import SqlSensor

# Wait until orders exist for today
wait_for_orders = SqlSensor(
    task_id='wait_for_orders',
    conn_id='makanexpress_db',
    sql="""
        SELECT COUNT(*) > 0 
        FROM raw_orders 
        WHERE DATE(order_date) = '{{ ds }}'
    """,
    # The query must return exactly one row with one column
    # If the value is truthy (True, non-zero), the sensor succeeds
    # If falsy (False, 0, NULL), it keeps checking
    poke_interval=60,       # check every 60 seconds
    timeout=3600,           # give up after 1 hour
    mode='poke',            # 'poke' = keep checking, 'reschedule' = free the worker slot
)
```

### Concept 3: PythonSensor — Custom Wait Logic

```python
from airflow.sensors.python import PythonSensor

def check_data_ready(**context):
    """Check if the data file for today has been uploaded."""
    import os
    ds = context['ds']
    filepath = f'/data/uploads/orders_{ds.replace("-", "")}.csv'
    
    if not os.path.exists(filepath):
        print(f"File not found: {filepath}")
        return False  # keep waiting
    
    # Check file is not empty
    if os.path.getsize(filepath) == 0:
        print(f"File is empty: {filepath}")
        return False
    
    print(f"File ready: {filepath}")
    return True  # condition met, proceed!

wait_for_upload = PythonSensor(
    task_id='wait_for_upload',
    python_callable=check_data_ready,
    poke_interval=120,      # check every 2 minutes
    timeout=7200,           # 2 hour timeout
    mode='reschedule',      # don't hold worker slot while waiting
)
```

### Concept 4: ExternalTaskSensor — Wait for Another DAG

```python
from airflow.sensors.external_task import ExternalTaskSensor

# Wait for the ingestion DAG to complete before running ETL
wait_for_ingestion = ExternalTaskSensor(
    task_id='wait_for_ingestion',
    external_dag_id='makanexpress_data_ingestion',  # the DAG to wait for
    external_task_id='load_to_database',             # specific task (or None for whole DAG)
    timeout=7200,
    poke_interval=120,
    mode='reschedule',
)

# Use case: DAG 1 ingests data from API (runs at 5 AM)
# DAG 2 processes data (runs at 6 AM, but waits for DAG 1)
```

### Concept 5: FileSensor — Wait for Files

```python
from airflow.sensors.filesystem import FileSensor

# Wait for a data file to land in the upload directory
wait_for_file = FileSensor(
    task_id='wait_for_file',
    filepath='/data/uploads/daily_orders_{{ ds_nodash }}.csv',
    poke_interval=30,
    timeout=3600,
    mode='poke',
)
```

### Concept 6: Sensor Modes — Poke vs Reschedule

```
mode='poke':
- Worker slot is HELD while waiting
- Good for: short waits (< 5 min)
- Bad for: long waits (wastes worker slots)

mode='reschedule':
- Worker slot is RELEASED between pokes
- Good for: long waits (hours)
- Bad for: adds small scheduling overhead

Rule of thumb:
- Wait < 5 minutes → mode='poke'
- Wait > 5 minutes → mode='reschedule'
```

---

## 🌤️ Afternoon Block (2 hours): Branching & Advanced Patterns

### Concept 7: BranchPythonOperator — Conditional Execution

```python
from airflow.operators.python import BranchPythonOperator

def decide_pipeline(**context):
    """Decide which branch to execute based on data volume."""
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    ds = context['ds']
    
    count = hook.get_first(
        "SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s", (ds,)
    )[0]
    
    if count > 1000:
        return 'heavy_load'      # task_id to run
    elif count > 0:
        return 'normal_load'     # different task_id
    else:
        return 'skip_load'       # another task_id

branch = BranchPythonOperator(
    task_id='decide_pipeline',
    python_callable=decide_pipeline,
)

heavy = PythonOperator(task_id='heavy_load', python_callable=lambda: print("Heavy processing..."))
normal = PythonOperator(task_id='normal_load', python_callable=lambda: print("Normal processing..."))
skip = PythonOperator(task_id='skip_load', python_callable=lambda: print("No data to process"))

branch >> [heavy, normal, skip]

# Note: only ONE of these tasks will run. The others are skipped.
```

### Concept 8: ShortCircuitOperator — Skip All Downstream

```python
from airflow.operators.python import ShortCircuitOperator

def is_working_day(**context):
    """Return False to skip ALL downstream tasks."""
    from datetime import date
    dt = context['logical_date'].date()
    
    # Skip weekends
    if dt.weekday() >= 5:
        print(f"🏖️ {dt} is a weekend — skipping")
        return False
    
    # Skip holidays
    import json
    holidays = json.loads(Variable.get('sg_holidays', default_var='[]'))
    if dt.isoformat() in holidays:
        print(f"🎉 {dt} is a holiday — skipping")
        return False
    
    return True  # proceed with pipeline

check_working_day = ShortCircuitOperator(
    task_id='check_working_day',
    python_callable=is_working_day,
)

# ALL downstream tasks are skipped if this returns False
check_working_day >> extract >> validate >> load
```

### Concept 9: Trigger Rules

```python
from airflow.utils.trigger_rule import TriggerRule

# Default: all_success (all upstream tasks must succeed)
# Other options:

task = PythonOperator(
    task_id='cleanup',
    python_callable=cleanup,
    trigger_rule=TriggerRule.ALL_DONE,       # run regardless of upstream success/failure
)

task = PythonOperator(
    task_id='notify_on_failure',
    python_callable=alert_failure,
    trigger_rule=TriggerRule.ONE_FAILED,      # run if at least one upstream failed
)

task = PythonOperator(
    task_id='notify_on_success',
    python_callable=alert_success,
    trigger_rule=TriggerRule.ALL_SUCCESS,     # run only if ALL upstream succeeded (default)
)

task = PythonOperator(
    task_id='combine_results',
    python_callable=combine,
    trigger_rule=TriggerRule.NONE_FAILED,     # run if no upstream failed (skipped is OK)
)

# Common pattern: always run cleanup, regardless of success/failure
extract >> transform >> load
[extract, transform, load] >> cleanup  # cleanup uses ALL_DONE
```

### Concept 10: Dynamic Task Generation

```python
from airflow import DAG
from airflow.operators.python import PythonOperator

# Generate tasks dynamically based on configuration
TABLES_TO_PROCESS = [
    {'name': 'customers', 'source': 'raw_customers', 'target': 'dim_customer'},
    {'name': 'restaurants', 'source': 'raw_restaurants', 'target': 'dim_restaurant'},
    {'name': 'items', 'source': 'raw_menu_items', 'target': 'dim_item'},
]

def process_table(table_config, **context):
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    source = table_config['source']
    target = table_config['target']
    ds = context['ds']
    
    count = hook.get_first(f"SELECT COUNT(*) FROM {source}")[0]
    print(f"Processing {source} → {target}: {count} rows for {ds}")
    return count

with DAG('dynamic_tables', schedule='@daily', ...) as dag:
    tasks = []
    for table in TABLES_TO_PROCESS:
        task = PythonOperator(
            task_id=f"process_{table['name']}",
            python_callable=process_table,
            op_kwargs={'table_config': table},
        )
        tasks.append(task)
    
    # Run all table loads in parallel
    # Then run a single validation task
    validate = PythonOperator(task_id='validate_all', python_callable=lambda: print("Validating..."))
    tasks >> validate
```

### Concept 11: Task Groups (Visual Organization)

```python
from airflow.utils.task_group import TaskGroup

with DAG('makanexpress_with_groups', schedule='@daily', ...) as dag:
    
    # Group 1: Extract tasks
    with TaskGroup(group_id='extract') as extract_group:
        t1 = PythonOperator(task_id='orders', python_callable=extract_orders)
        t2 = PythonOperator(task_id='customers', python_callable=extract_customers)
        t3 = PythonOperator(task_id='restaurants', python_callable=extract_restaurants)
    
    # Group 2: Transform tasks
    with TaskGroup(group_id='transform') as transform_group:
        t4 = PythonOperator(task_id='clean', python_callable=clean_data)
        t5 = PythonOperator(task_id='enrich', python_callable=enrich_data)
        t4 >> t5
    
    # Group 3: Load tasks
    with TaskGroup(group_id='load') as load_group:
        t6 = PythonOperator(task_id='dimensions', python_callable=load_dims)
        t7 = PythonOperator(task_id='facts', python_callable=load_facts)
        t6 >> t7
    
    extract_group >> transform_group >> load_group
```

### Concept 12: SubDAGs (Legacy — prefer Task Groups)

```python
# Task Groups are preferred over SubDAGs (which are deprecated in Airflow 2.x)
# Always use TaskGroup instead of SubDagOperator for new DAGs
```

---

### 🏋️ Full Day Exercises (30 questions)

1. Create a DAG that uses a `SqlSensor` to wait until at least 10 orders exist for `{{ ds }}` before processing.

2. Write a `PythonSensor` that checks if a file `/tmp/makanexpress_data_{{ ds_nodash }}.csv` exists. Create the file manually and verify the sensor detects it.

3. Create two DAGs: `ingestion` and `etl`. The `etl` DAG uses an `ExternalTaskSensor` to wait for `ingestion` to complete.

4. What's the difference between `mode='poke'` and `mode='reschedule'`? When would you use each?

5. Write a `BranchPythonOperator` that routes to `process_weekday` on weekdays and `process_weekend` on weekends.

6. A sensor times out after 1 hour. What happens to the DAG run? How do you handle this gracefully?

7. Create a `ShortCircuitOperator` that skips the pipeline if today is a Singapore public holiday.

8. Write a DAG with 5 tasks where the last task always runs (success or failure of upstream). Use `trigger_rule=ALL_DONE`.

9. Create a DAG with a "failure notification" task that only runs when an upstream task fails. Use `trigger_rule=ONE_FAILED`.

10. **Dynamic tasks:** Write a DAG that reads a list of cuisine types from a Variable and creates one task per cuisine type.

11. Use `TaskGroup` to organize a DAG with 3 groups: `extract`, `transform`, `load`. Each group has 2-3 tasks.

12. Write a sensor that waits for a specific table to have more than 0 rows (e.g., raw_orders for today).

13. Create a DAG that branches based on data quality: if quality passes → load to production; if quality fails → load to quarantine table + send alert.

14. **Advanced branching:** Create a DAG that processes differently based on the day:
    - Monday: full reload
    - Wednesday: incremental load
    - Friday: validation only
    - Other days: normal daily load

15. Write a sensor that checks the row count of a table and waits until it exceeds a threshold stored in a Variable.

16. Create a pipeline with parallel extraction from 3 source tables, then a join/merge task, then parallel loading to 2 target tables.

17. Use `TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS` to create a task that runs if at least one upstream succeeded (some can be skipped).

18. Write a DAG that generates tasks dynamically from a SQL query: "SELECT table_name FROM etl_registry WHERE active = true".

19. Create a sensor that checks if a stored procedure in PostgreSQL has completed (check a status table).

20. **Error recovery pattern:** Create a DAG where:
    - Task 1: Extract data
    - Task 2: Process data (might fail)
    - Task 3: On failure, send alert and try a fallback query
    - Task 4: Load results
    - Task 5: Always clean up temp tables

21. Write a DAG with nested TaskGroups: `extract.raw_files`, `extract.api_data`, `transform.clean`, `transform.enrich`, `load.warehouse`.

22. Create a conditional pipeline where the branch decision is stored in a Variable (can be changed from the UI).

23. Write a sensor that waits for a specific time of day (e.g., "don't start processing until 7 AM SGT").

24. **Multi-DAG coordination:** Create 3 DAGs that run in sequence:
    - DAG 1: Ingest raw data
    - DAG 2: Process and load warehouse
    - DAG 3: Generate reports
    Use `ExternalTaskSensor` to coordinate them.

25. Create a DAG that handles both full loads and incremental loads based on a DAG parameter (`--conf '{"mode": "full"}'`).

26. Write a pattern for "run once and mark done" — a DAG that should only ever run once (first-time setup). Use a Variable to track if it's been run.

27. **Production pattern:** Create a DAG with:
    - Sensor to wait for upstream data
    - BranchPython to choose processing path
    - Parallel execution with TaskGroups
    - Quality check with conditional failure handling
    - Always-run cleanup
    - Notification on both success and failure

28. Write a custom sensor that checks if a PostgreSQL query returns results within a specific time range.

29. Create a DAG that uses `@task.branch` decorator (TaskFlow API branching). Compare with BranchPythonOperator.

30. **Challenge:** Build a complete multi-DAG data platform:
    - `makanexpress_ingestion` — ingests data from "API" (simulate with Python)
    - `makanexpress_warehouse_etl` — loads warehouse (from Day 31)
    - `makanexpress_quality` — runs quality checks
    - `makanexpress_reports` — generates reports
    All coordinated with sensors and proper dependencies.

---

### ✅ Answers

<details>
<summary>🔑 Click to reveal selected answers</summary>

**Answer 1:**
```python
from airflow.sensors.sql import SqlSensor

wait = SqlSensor(
    task_id='wait_for_orders',
    conn_id='makanexpress_db',
    sql="SELECT COUNT(*) >= 10 FROM raw_orders WHERE DATE(order_date) = '{{ ds }}'",
    poke_interval=30,
    timeout=600,
    mode='poke',
)
```

**Answer 2:**
```python
from airflow.sensors.python import PythonSensor
import os

def check_file(**context):
    ds_nodash = context['ds_nodash']
    return os.path.exists(f'/tmp/makanexpress_data_{ds_nodash}.csv')

wait = PythonSensor(
    task_id='wait_for_file',
    python_callable=check_file,
    poke_interval=30,
    timeout=300,
)
```

**Answer 5:**
```python
from airflow.operators.python import BranchPythonOperator
from datetime import datetime

def check_day(**context):
    dt = context['logical_date']
    if dt.weekday() < 5:
        return 'process_weekday'
    return 'process_weekend'

branch = BranchPythonOperator(task_id='check_day', python_callable=check_day)
weekday = BashOperator(task_id='process_weekday', bash_command='echo "Weekday load"')
weekend = BashOperator(task_id='process_weekend', bash_command='echo "Weekend load"')
branch >> [weekday, weekend]
```

**Answer 7:**
```python
from airflow.operators.python import ShortCircuitOperator

def check_holiday(**context):
    holidays = ['2026-01-01', '2026-02-17', '2026-08-09', '2026-12-25']
    return context['ds'] not in holidays

check = ShortCircuitOperator(task_id='check_holiday', python_callable=check_holiday)
check >> extract >> load
```

**Answer 8:**
```python
cleanup = PythonOperator(
    task_id='cleanup',
    python_callable=cleanup_fn,
    trigger_rule=TriggerRule.ALL_DONE,
)
[task1, task2, task3, task4] >> cleanup
```

**Answer 10:**
```python
import json
from airflow.models import Variable

cuisines = json.loads(Variable.get('active_cuisines', default_var='["Chinese","Malay","Indian","Western"]'))

with DAG('per_cuisine_etl', schedule='@daily', ...) as dag:
    for cuisine in cuisines:
        PythonOperator(
            task_id=f'process_{cuisine.lower()}',
            python_callable=process_cuisine,
            op_kwargs={'cuisine': cuisine},
        )
```

**Answer 11:**
```python
from airflow.utils.task_group import TaskGroup

with DAG('grouped_dag', schedule='@daily', ...) as dag:
    with TaskGroup('extract') as tg_extract:
        e1 = PythonOperator(task_id='orders', ...)
        e2 = PythonOperator(task_id='customers', ...)
        e3 = PythonOperator(task_id='restaurants', ...)
    
    with TaskGroup('transform') as tg_transform:
        t1 = PythonOperator(task_id='clean', ...)
        t2 = PythonOperator(task_id='enrich', ...)
        t1 >> t2
    
    with TaskGroup('load') as tg_load:
        l1 = PythonOperator(task_id='dimensions', ...)
        l2 = PythonOperator(task_id='facts', ...)
        l1 >> l2
    
    tg_extract >> tg_transform >> tg_load
```

**Answer 20:**
```python
extract >> process
process >> load
process >> alert  # alert uses trigger_rule=ONE_FAILED
alert >> fallback_query  # only runs if process failed
load >> cleanup  # uses trigger_rule=ALL_DONE
fallback_query >> cleanup
```

**Answer 26:**
```python
def run_once_check(**context):
    from airflow.models import Variable
    done = Variable.get('initial_load_done', default_var='false')
    if done == 'true':
        raise AirflowSkipException("Initial load already done!")
    Variable.set('initial_load_done', 'true')
    print("Running initial load...")

check = PythonOperator(task_id='run_once', python_callable=run_once_check)
```

**Answer 29:**
```python
from airflow.decorators import dag, task

@dag(schedule='@daily', ...)
def branched_taskflow():
    @task.branch
    def decide(**context):
        # returns task_id to run
        if context['logical_date'].weekday() < 5:
            return 'weekday_process'
        return 'weekend_process'
    
    @task
    def weekday_process():
        print("Full weekday processing")
    
    @task
    def weekend_process():
        print("Light weekend processing")
    
    @task(trigger_rule=TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS)
    def finalize():
        print("Done")
    
    decision = decide()
    decision >> [weekday_process(), weekend_process()] >> finalize()

branched_taskflow()
```

</details>

---

## 🌙 Evening Block (1 hour): Review

### Sensor vs Branch vs ShortCircuit

| Pattern | Use Case | Effect |
|---------|----------|--------|
| Sensor | Wait for external condition (file, data, time) | Blocks until condition met |
| BranchPython | Choose one of multiple paths | Only one path runs, others skipped |
| ShortCircuit | Skip all downstream based on condition | Everything downstream is skipped |
| TriggerRule | Control when a task runs based on upstream results | Custom success/failure handling |

### 📝 Today's Checklist

- [ ] I can use SqlSensor and PythonSensor to wait for conditions
- [ ] I understand poke vs reschedule mode
- [ ] I can use BranchPythonOperator for conditional execution
- [ ] I can use ShortCircuitOperator to skip pipelines
- [ ] I understand trigger rules (ALL_DONE, ONE_FAILED, etc.)
- [ ] I can create dynamic tasks and TaskGroups
- [ ] I completed at least 20 of the 30 exercises
- [ ] I pushed code to GitHub

---

*Day 32 complete! Tomorrow: Airflow Best Practices — making pipelines production-ready.* 🏗️
