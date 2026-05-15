# 📅 Day 34 — Tuesday, 17 June 2026
# 📝 Week 5 Assessment: Airflow Review & Weak Spot Practice

---

## 🎯 Today's Goal

No new concepts. Timed test covering all of Week 5 (Airflow). Grade yourself honestly and review weak spots before tomorrow's portfolio project.

---

## ☀️ Morning Block (2 hours): Timed Assessment

### Section A: Core Concepts (15 minutes, 5 questions)

**Q1.** Explain what a DAG is and why it must be acyclic. What happens if you create a cyclic dependency?

**Q2.** What is the difference between `logical_date` and the actual run time? Give a concrete example with a daily 6 AM schedule.

**Q3.** List 5 Airflow Operators and when you'd use each.

**Q4.** What does `catchup=True` do? When would you want it? When is it dangerous?

**Q5.** Explain the difference between a Sensor and a BranchPythonOperator. When would you use each?

### Section B: DAG Writing (25 minutes, 5 questions)

**Q6.** Write a complete DAG (from imports to dependencies) with:
- 4 tasks: extract, validate, load, notify
- Runs daily at 7 AM SGT
- 2 retries with exponential backoff
- catchup=False
- `load` depends on `validate`, `notify` uses `trigger_rule=ALL_DONE`

**Q7.** Write a PythonOperator task that uses PostgresHook to count orders for `{{ ds }}` and returns the count via XCom. Write a second task that pulls the count and prints a summary.

**Q8.** Write a BranchPythonOperator that checks the day of week:
- Monday/Wednesday/Friday → run `full_load`
- Tuesday/Thursday → run `incremental_load`
- Weekend → skip everything

**Q9.** Write a Sensor that waits for a SQL query to return a non-zero count before proceeding. Include proper timeout and poke_interval.

**Q10.** Write a TaskFlow API DAG with 3 tasks that pass data between them using return values (XCom).

### Section C: Connections & Configuration (15 minutes, 5 questions)

**Q11.** How do you create a PostgreSQL connection in Airflow? Show both CLI and UI methods.

**Q12.** Write code that reads an Airflow Variable `batch_size` and uses it in a task. Show the CLI command to set the variable.

**Q13.** What is the difference between an Airflow Connection and an Airflow Variable? When would you use each?

**Q14.** Write code that uses `PostgresHook.get_pandas_df()` to load data into a DataFrame and print summary statistics.

**Q15.** How do you pass data larger than 48KB between tasks? Why is XCom limited?

### Section D: Best Practices (15 minutes, 5 questions)

**Q16.** What makes a task "idempotent"? Write an idempotent INSERT statement.

**Q17.** When should you use `AirflowFailException` vs `AirflowSkipException` vs a regular `raise`?

**Q18.** Write an `on_failure_callback` that logs the DAG ID, task ID, date, and error to a PostgreSQL table.

**Q19.** List 8 items from the production DAG checklist.

**Q20.** Write a unit test that verifies a DAG loads without import errors and has the expected tasks.

---

## 🌤️ Afternoon Block (2 hours): Answers & Review

### Scoring Guide

| Section | Questions | Total |
|---------|-----------|-------|
| A: Core Concepts | 5 × 4 pts | 20 |
| B: DAG Writing | 5 × 8 pts | 40 |
| C: Connections | 5 × 4 pts | 20 |
| D: Best Practices | 5 × 4 pts | 20 |
| **Total** | **20** | **100** |

**90+**: Ready for production Airflow work
**70-89**: Good — review weak spots
**50-69**: Re-read the days you struggled with
**Below 50**: Re-read the full week

---

### ✅ Full Answers

<details>
<summary>🔑 Section A: Core Concepts</summary>

**A1:** A DAG (Directed Acyclic Graph) is a workflow definition where tasks flow in one direction without loops. "Acyclic" means no cycles — tasks can't create circular dependencies. If cyclic, Airflow would loop infinitely. Airflow validates this at parse time and rejects cyclic DAGs.

**A2:** `logical_date` is the date this run is FOR (the data being processed). Actual run time is when Airflow executes. Example: A daily 6 AM job on June 13 processes June 12's data. `logical_date` = June 12, actual run = June 13 at 6 AM. This lets you process "yesterday's data" correctly.

**A3:**
1. `BashOperator` — run shell commands (scripts, CLI tools)
2. `PythonOperator` — run Python functions (custom logic)
3. `PostgresOperator` — run SQL against PostgreSQL
4. `PythonSensor` — wait for a Python condition to be true
5. `BranchPythonOperator` — choose which task to run next

**A4:** `catchup=True` means Airflow creates runs for all dates from `start_date` to now. Useful when intentionally backfilling. Dangerous because it can create hundreds of unexpected runs that overwhelm resources. Always use `catchup=False` unless you explicitly want backfill.

**A5:** A Sensor WAITS (polling) for an external condition before proceeding. Use when waiting for a file, data, or external event. BranchPythonOperator DECIDES which path to take based on a condition. Use when routing to different logic based on data/day/configuration.

</details>

<details>
<summary>🔑 Section B: DAG Writing</summary>

**B1 (Q6):**
```python
from datetime import datetime, timedelta
import pytz
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.trigger_rule import TriggerRule

def extract(**context):
    print(f"Extracting for {context['ds']}")

def validate(**context):
    print(f"Validating for {context['ds']}")

def load(**context):
    print(f"Loading for {context['ds']}")

def notify(**context):
    print("Pipeline complete!")

with DAG(
    dag_id='daily_pipeline',
    schedule='0 7 * * *',
    start_date=datetime(2026, 6, 12, tzinfo=pytz.timezone('Asia/Singapore')),
    timezone=pytz.timezone('Asia/Singapore'),
    catchup=False,
    default_args={
        'retries': 2,
        'retry_delay': timedelta(minutes=5),
        'retry_exponential_backoff': True,
    },
) as dag:
    t_extract = PythonOperator(task_id='extract', python_callable=extract)
    t_validate = PythonOperator(task_id='validate', python_callable=validate)
    t_load = PythonOperator(task_id='load', python_callable=load)
    t_notify = PythonOperator(task_id='notify', python_callable=notify, trigger_rule=TriggerRule.ALL_DONE)

    t_extract >> t_validate >> t_load >> t_notify
```

**B2 (Q7):**
```python
from airflow.providers.postgres.hooks.postgres import PostgresHook

def count_orders(**context):
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    count = hook.get_first(
        "SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s",
        (context['ds'],)
    )[0]
    return count

def print_summary(**context):
    count = context['ti'].xcom_pull(task_ids='count_orders')
    print(f"Orders for {context['ds']}: {count}")

t1 = PythonOperator(task_id='count_orders', python_callable=count_orders)
t2 = PythonOperator(task_id='print_summary', python_callable=print_summary)
t1 >> t2
```

**B3 (Q8):**
```python
from airflow.operators.python import BranchPythonOperator, ShortCircuitOperator
from airflow.operators.bash import BashOperator

def check_day(**context):
    dt = context['logical_date']
    day = dt.weekday()
    if day in (0, 2, 4):  # Mon, Wed, Fri
        return 'full_load'
    elif day in (1, 3):   # Tue, Thu
        return 'incremental_load'
    else:                  # Weekend
        return False  # skip all downstream

branch = BranchPythonOperator(task_id='check_day', python_callable=check_day)
full = BashOperator(task_id='full_load', bash_command='echo "Full load"')
incr = BashOperator(task_id='incremental_load', bash_command='echo "Incremental"')
branch >> [full, incr]
```

**B4 (Q9):**
```python
from airflow.sensors.sql import SqlSensor

wait_for_data = SqlSensor(
    task_id='wait_for_data',
    conn_id='makanexpress_db',
    sql="SELECT COUNT(*) > 0 FROM raw_orders WHERE DATE(order_date) = '{{ ds }}'",
    poke_interval=60,
    timeout=3600,
    mode='reschedule',
)
```

**B5 (Q10):**
```python
from airflow.decorators import dag, task

@dag(schedule='@daily', start_date=datetime(2026, 6, 12), catchup=False)
def taskflow_example():
    @task
    def extract():
        return {'orders': 42, 'revenue': 1250.50}

    @task
    def transform(data):
        data['avg'] = data['revenue'] / data['orders']
        return data

    @task
    def load(data):
        print(f"Loaded {data['orders']} orders, avg ${data['avg']:.2f}")

    data = extract()
    transformed = transform(data)
    load(transformed)

taskflow_example()
```

</details>

<details>
<summary>🔑 Section C: Connections & Configuration</summary>

**C1 (Q11):**
```bash
# CLI
airflow connections add makanexpress_db \
    --conn-type postgres --conn-host localhost \
    --conn-schema makanexpress --conn-login postgres --conn-port 5432

# UI: Admin → Connections → + (Add) → Fill in fields
```

**C2 (Q12):**
```bash
airflow variables set batch_size 5000
```
```python
from airflow.models import Variable

def my_task(**context):
    batch_size = int(Variable.get('batch_size', default_var='1000'))
    print(f"Processing with batch_size={batch_size}")
```

**C3 (Q13):**
- **Connection**: Stores credentials for external systems (host, user, password, port). Use for databases, APIs, cloud services.
- **Variable**: Stores configuration values (settings, thresholds, flags). Use for batch sizes, table names, feature flags.

**C4 (Q14):**
```python
from airflow.providers.postgres.hooks.postgres import PostgresHook

def analyze(**context):
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    df = hook.get_pandas_df("""
        SELECT cuisine_type, COUNT(*) as orders, AVG(total_amount) as avg_amount
        FROM raw_orders o
        JOIN raw_restaurants r ON o.restaurant_id = r.restaurant_id
        WHERE DATE(o.order_date) = %s
        GROUP BY cuisine_type
    """, parameters=(context['ds'],))
    print(df.describe())
    print(df)
```

**C4 (Q15):** XCom is limited to 48KB because it's stored in the Airflow metadata database. For large data:
1. Write data to a file/table and pass the PATH via XCom
2. Use an intermediate storage (S3, GCS, shared filesystem)
3. Use TaskFlow API with custom XCom backend

</details>

<details>
<summary>🔑 Section D: Best Practices</summary>

**D1 (Q16):** Idempotent = safe to run multiple times with the same result.
```sql
-- Idempotent INSERT
INSERT INTO fact_table (date_key, ...)
SELECT ... FROM source
WHERE date = '{{ ds }}'
AND NOT EXISTS (
    SELECT 1 FROM fact_table WHERE date_key = dd.date_key AND ...
);

-- Or: DELETE first, then INSERT
DELETE FROM fact_table WHERE date = '{{ ds }}';
INSERT INTO fact_table ...;
```

**D2 (Q17):**
- `AirflowFailException`: Task fails immediately, NO retry. Use for configuration errors, business logic violations.
- `AirflowSkipException`: Task is skipped (white in UI), downstream tasks may or may not run depending on trigger_rule. Use for "no data to process" scenarios.
- Regular `raise`: Task fails and Airflow retries (up to `retries` count). Use for transient errors (connection timeouts, temporary issues).

**D3 (Q18):**
```python
def on_failure(context):
    from airflow.providers.postgres.hooks.postgres import PostgresHook
    hook = PostgresHook(postgres_conn_id='makanexpress_db')
    hook.run("""
        INSERT INTO etl_failures (dag_id, task_id, logical_date, error, timestamp)
        VALUES (%s, %s, %s, %s, NOW())
    """, parameters=(
        context['dag'].dag_id,
        context['task'].task_id,
        str(context['logical_date']),
        str(context.get('exception', '')),
    ))
```

**D4 (Q19):** Eight checklist items:
1. catchup=False
2. max_active_runs=1
3. retries >= 2
4. retry_exponential_backoff=True
5. No hardcoded credentials
6. Idempotent tasks
7. on_failure_callback
8. Data quality checks

**D5 (Q20):**
```python
from airflow.models import DagBag

def test_dag_loads():
    dagbag = DagBag(dag_folder='dags/', include_examples=False)
    assert len(dagbag.import_errors) == 0
    dag = dagbag.get_dag('makanexpress_warehouse_etl')
    assert dag is not None
    expected_tasks = ['extract', 'validate', 'load', 'notify']
    task_ids = {t.task_id for t in dag.tasks}
    for t in expected_tasks:
        assert t in task_ids
```

</details>

---

## 🌙 Evening: Targeted Review

**If you scored < 14/20 in Section A:** Re-read Day 29 morning
**If you scored < 28/40 in Section B:** Re-read Day 29-30, practice writing DAGs
**If you scored < 14/20 in Section C:** Re-read Day 30 morning
**If you scored < 14/20 in Section D:** Re-read Day 33

### 📝 Today's Checklist

- [ ] I completed the assessment honestly
- [ ] I graded myself and identified weak spots
- [ ] I reviewed at least 2 weak areas
- [ ] I'm ready for tomorrow's portfolio project

---

*Day 34 complete! Tomorrow: Portfolio Project — adding Airflow orchestration to your food delivery warehouse.* 🏗️
