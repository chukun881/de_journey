# 📅 Day 33 — Monday, 16 June 2026
# 🏗️ Airflow Best Practices: Error Handling, Testing, and Production Patterns

---

## 🎯 Today's Goal

Writing DAGs is 30% of the job. Making them **reliable, testable, and maintainable** is the other 70%. Today you'll learn production best practices that separate amateur DAGs from professional ones.

---

## ☀️ Morning Block (2 hours): Error Handling & Retries

### Concept 1: Retry Strategies

```python
from datetime import timedelta

default_args = {
    # Basic retry
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    
    # Exponential backoff (5min → 10min → 20min → 40min...)
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(hours=2),
    
    # Don't retry on certain exceptions
    # (implement in your callable)
}

# Custom retry logic in your function
from airflow.exceptions import AirflowSkipException, AirflowFailException

def smart_process(**context):
    try:
        result = risky_operation()
    except ConnectionError as e:
        # Transient error — let Airflow retry
        raise  # re-raise → Airflow will retry
    except ValueError as e:
        # Business logic error — don't retry, fail immediately
        raise AirflowFailException(f"Data validation failed: {e}")
        # AirflowFailException prevents retries
    except EmptyDataError:
        # Expected scenario — skip, don't fail
        raise AirflowSkipException("No data to process today")
```

### Concept 2: Custom Exception Handling

```python
from airflow.exceptions import AirflowFailException, AirflowSkipException, AirflowSensorTimeout

class DataQualityError(Exception):
    """Raised when data quality checks fail."""
    pass

class ConfigurationError(Exception):
    """Raised when DAG configuration is invalid."""
    pass

def load_data(**context):
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    ds = context['ds']
    
    # 1. Validate configuration
    table = context['params'].get('table')
    if not table:
        raise AirflowFailException("Missing 'table' parameter")  # no retry
    
    # 2. Check data exists
    count = hook.get_first(
        f"SELECT COUNT(*) FROM {table} WHERE DATE(created_at) = %s", (ds,)
    )[0]
    
    if count == 0:
        raise AirflowSkipException(f"No data in {table} for {ds}")  # skip, don't fail
    
    if count > 1000000:
        raise AirflowFailException(f"Too many rows ({count}) — possible duplication")  # no retry
    
    # 3. Process with error handling
    try:
        process_records(hook, table, ds, count)
    except Exception as e:
        # Log the error but allow retry
        import logging
        logging.getLogger(__name__).error(f"Processing failed: {e}")
        raise  # will retry
```

### Concept 3: Circuit Breaker Pattern

```python
def process_with_circuit_breaker(**context):
    """Stop processing if too many errors in recent runs."""
    from airflow.utils.session import create_session
    from airflow.models import TaskInstance
    from airflow.utils.state import State
    
    dag_id = context['dag'].dag_id
    task_id = context['task'].task_id
    
    with create_session() as session:
        # Check last 5 runs
        recent = session.query(TaskInstance).filter(
            TaskInstance.dag_id == dag_id,
            TaskInstance.task_id == task_id,
            TaskInstance.state == State.FAILED,
        ).order_by(TaskInstance.execution_date.desc()).limit(5).all()
        
        if len(recent) >= 5:
            raise AirflowFailException(
                "Circuit breaker: task has failed 5 consecutive times. "
                "Manual intervention required."
            )
    
    # Proceed with processing
    print("Circuit breaker passed, processing...")
```

### Concept 4: Dead Letter Queue Pattern

```python
def load_with_dlq(**context):
    """Load data, send failures to a dead letter queue table."""
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    ds = context['ds']
    
    # Get records to process
    records = hook.get_records("""
        SELECT * FROM staging_orders WHERE DATE(order_date) = %s
    """, parameters=(ds,))
    
    success_count = 0
    failure_count = 0
    
    for record in records:
        try:
            # Validate each record
            if not record['customer_id'] or not record['total_amount']:
                raise ValueError(f"Missing required field in order {record['order_id']}")
            
            # Load valid record
            hook.run("""
                INSERT INTO fact_order_items (...) VALUES (...)
            """, parameters=(...))
            success_count += 1
            
        except Exception as e:
            # Send to dead letter queue
            hook.run("""
                INSERT INTO dead_letter_queue (source_table, record_id, error_message, failed_date, raw_data)
                VALUES (%s, %s, %s, %s, %s)
            """, parameters=(
                'staging_orders', record['order_id'], str(e), ds, str(record)
            ))
            failure_count += 1
    
    print(f"✅ Loaded: {success_count}, ❌ Failed: {failure_count}")
    
    # Fail if too many errors
    if failure_count > success_count * 0.1:  # >10% failure rate
        raise ValueError(f"Too many failures: {failure_count}/{len(records)}")
    
    context['ti'].xcom_push(key='success_count', value=success_count)
    context['ti'].xcom_push(key='failure_count', value=failure_count)
```

### Concept 5: Timeout Handling

```python
# Task-level timeout
task = PythonOperator(
    task_id='slow_task',
    python_callable=slow_function,
    execution_timeout=timedelta(minutes=30),  # fail if takes >30 min
)

# DAG-level timeout
with DAG(
    dag_id='timed_dag',
    dagrun_timeout=timedelta(hours=2),  # entire DAG must complete in 2 hours
    ...
) as dag:
    ...
```

---

## 🌤️ Afternoon Block (2 hours): Testing & Monitoring

### Concept 6: Unit Testing DAGs

```python
# tests/test_dag.py
import pytest
from airflow.models import DagBag

@pytest.fixture
def dagbag():
    return DagBag(dag_folder='dags/', include_examples=False)

def test_dag_loaded(dagbag):
    """Test that the DAG loads without errors."""
    dag = dagbag.get_dag(dag_id='makanexpress_warehouse_etl')
    assert dag is not None
    assert len(dag.import_errors) == 0

def test_dag_has_expected_tasks(dagbag):
    """Test that the DAG has all expected tasks."""
    dag = dagbag.get_dag(dag_id='makanexpress_warehouse_etl')
    task_ids = [t.task_id for t in dag.tasks]
    
    expected = [
        'extract_and_validate',
        'load_dim_customer',
        'load_dim_restaurant',
        'load_dim_item',
        'load_fact',
        'update_segments',
        'quality_checks',
        'notify',
    ]
    
    for task_id in expected:
        assert task_id in task_ids, f"Missing task: {task_id}"

def test_dag_dependencies(dagbag):
    """Test that task dependencies are correct."""
    dag = dagbag.get_dag(dag_id='makanexpress_warehouse_etl')
    
    # extract_and_validate should be upstream of load tasks
    extract = dag.get_task('extract_and_validate')
    downstream = [t.task_id for t in extract.downstream_list]
    
    assert 'load_dim_customer' in downstream
    assert 'load_dim_restaurant' in downstream
    assert 'load_dim_item' in downstream

def test_default_args(dagbag):
    """Test that default args are properly set."""
    dag = dagbag.get_dag(dag_id='makanexpress_warehouse_etl')
    assert dag.default_args['retries'] >= 2
    assert dag.catchup is False
    assert dag.max_active_runs == 1

def test_no_circular_dependencies(dagbag):
    """Test that there are no cycles in the DAG."""
    dag = dagbag.get_dag(dag_id='makanexpress_warehouse_etl')
    # If the DAG loaded, Airflow already validated no cycles
    assert dag is not None

# Run tests:
# pytest tests/test_dag.py -v
```

### Concept 7: Testing Individual Tasks

```python
# tests/test_tasks.py
import pytest
from datetime import datetime
from unittest.mock import MagicMock, patch

def test_extract_and_validate():
    """Test the extract_and_validate function."""
    # Mock the context
    context = {
        'ds': '2026-06-12',
        'data_interval_start': datetime(2026, 6, 12),
        'data_interval_end': datetime(2026, 6, 13),
        'ti': MagicMock(),  # mock TaskInstance
    }
    
    # Mock the database hook
    with patch(' airflow.providers.postgres.hooks.postgres.PostgresHook') as MockHook:
        mock_hook = MockHook.return_value
        mock_hook.get_first.return_value = (42,)  # 42 orders
        
        # Import and run the function
        from dags.makanexpress_warehouse_etl import extract_and_validate
        result = extract_and_validate(**context)
        
        assert result == 42
        context['ti'].xcom_push.assert_called_once_with(key='order_count', value=42)

def test_quality_checks_pass():
    """Test quality checks with valid data."""
    context = {
        'ds': '2026-06-12',
        'data_interval_start': datetime(2026, 6, 12),
        'data_interval_end': datetime(2026, 6, 13),
    }
    
    with patch('airflow.providers.postgres.hooks.postgres.PostgresHook') as MockHook:
        mock_hook = MockHook.return_value
        # All checks return 0 (no issues)
        mock_hook.get_first.side_effect = [(0,), (100,), (95,), (0,)]
        
        from dags.makanexpress_warehouse_etl import quality_checks
        quality_checks(**context)  # Should not raise

def test_quality_checks_fail_on_orphans():
    """Test quality checks fails when orphans found."""
    context = {
        'ds': '2026-06-12',
        'data_interval_start': datetime(2026, 6, 12),
        'data_interval_end': datetime(2026, 6, 13),
    }
    
    with patch('airflow.providers.postgres.hooks.postgres.PostgresHook') as MockHook:
        mock_hook = MockHook.return_value
        mock_hook.get_first.side_effect = [(5,)]  # 5 orphans!
        
        from dags.makanexpress_warehouse_etl import quality_checks
        with pytest.raises(ValueError, match="orphan"):
            quality_checks(**context)
```

### Concept 8: DAG Structure Best Practices

```python
# ✅ GOOD DAG structure

"""
1. Imports at the top
2. Constants and configuration
3. Helper functions
4. DAG definition
5. Task definitions
6. Dependencies
7. Instantiate (if using factory pattern)
"""

# File: dags/makanexpress_warehouse_etl.py

# ─── Imports ───
from datetime import datetime, timedelta
import pytz
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook

# ─── Constants ───
CONNECTION_ID = 'makanexpress_db'
SG_TIMEZONE = pytz.timezone('Asia/Singapore')
DEFAULT_RETRIES = 2
BATCH_SIZE = 5000  # can override with Variable

# ─── Helper Functions ───
def get_hook():
    return PostgresHook(postgres_conn_id=CONNECTION_ID)

def format_report(data):
    ...

# ─── Task Functions ───
def extract(**context):
    ...

def transform(**context):
    ...

# ─── DAG Definition ───
default_args = {
    'owner': 'data-engineering',
    'retries': DEFAULT_RETRIES,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    dag_id='makanexpress_warehouse_etl',
    default_args=default_args,
    schedule='0 6 * * *',
    start_date=datetime(2026, 6, 1, tzinfo=SG_TIMEZONE),
    timezone=SG_TIMEZONE,
    catchup=False,
    max_active_runs=1,
    tags=['makanexpress', 'etl'],
) as dag:

    # ─── Tasks ───
    task_extract = PythonOperator(task_id='extract', python_callable=extract)
    task_transform = PythonOperator(task_id='transform', python_callable=transform)
    task_load = PythonOperator(task_id='load', python_callable=load)

    # ─── Dependencies ───
    task_extract >> task_transform >> task_load
```

### Concept 9: Monitoring and Alerting Patterns

```python
# Pattern 1: Slack notification on failure (if you have Slack)
# Set up in Airflow UI: Admin → Connections → Add "slack_webhook"

# Pattern 2: Email on failure
default_args = {
    'email': ['data-team@makanexpress.com'],
    'email_on_failure': True,
    'email_on_retry': False,  # don't spam on retries
}

# Pattern 3: Custom callback
def on_failure_callback(context):
    """Runs when a task fails (after all retries exhausted)."""
    import logging
    logger = logging.getLogger(__name__)
    
    dag_id = context['dag'].dag_id
    task_id = context['task'].task_id
    logical_date = context['logical_date']
    exception = context.get('exception')
    
    logger.error(f"""
    ❌ Task Failed!
    DAG: {dag_id}
    Task: {task_id}
    Date: {logical_date}
    Error: {exception}
    """)
    
    # Could also: write to alert table, call webhook, etc.
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    hook.run("""
        INSERT INTO etl_alerts (dag_id, task_id, logical_date, error_message, alert_time)
        VALUES (%s, %s, %s, %s, NOW())
    """, parameters=(dag_id, task_id, str(logical_date), str(exception)))

def on_success_callback(context):
    """Runs when a task succeeds."""
    logger = logging.getLogger(__name__)
    logger.info(f"✅ {context['task'].task_id} succeeded for {context['ds']}")

default_args = {
    'on_failure_callback': on_failure_callback,
    'on_success_callback': on_success_callback,
}
```

### Concept 10: Production Checklist

```python
"""
Production DAG Checklist:
✅ catchup=False (unless intentional)
✅ max_active_runs=1 (prevent concurrent runs)
✅ retries >= 2
✅ retry_delay with exponential backoff
✅ execution_timeout on long tasks
✅ Idempotent tasks (safe to retry)
✅ No hardcoded credentials
✅ Connections and Variables used
✅ Error handling (AirflowFailException, AirflowSkipException)
✅ Logging (start/end, counts, timing)
✅ Data quality checks
✅ on_failure_callback
✅ Description and tags
✅ Unit tests
✅ README with setup instructions
✅ No circular dependencies
✅ Proper timezone handling
✅ start_date in the past (but catchup=False)
✅ No dynamic start_date (causes issues)
"""
```

---

### 🏋️ Full Day Exercises (25 questions)

1. Add `retry_exponential_backoff` to your MakanExpress DAG. Test by making a task fail and observe the retry timing.

2. Write a function that raises `AirflowFailException` for invalid configuration and `AirflowSkipException` for empty data. Test both paths.

3. Implement the dead letter queue pattern: create a `dead_letter_queue` table and modify a task to log failed records.

4. Add `execution_timeout=timedelta(minutes=10)` to a task. What happens if the task takes 11 minutes?

5. Write `on_failure_callback` and `on_success_callback` functions that log to an `etl_audit_log` table.

6. Create unit tests for your MakanExpress DAG:
    - Test DAG loads without errors
    - Test all expected tasks exist
    - Test dependencies are correct
    - Test default_args are set properly

7. Write a unit test that mocks `PostgresHook` and tests the `extract_and_validate` function in isolation.

8. Add `dagrun_timeout=timedelta(hours=2)` to your DAG. What happens if the entire pipeline takes 3 hours?

9. Refactor your MakanExpress DAG to follow the "good DAG structure" from Concept 8.

10. Write a custom exception `DataQualityError` that triggers a different handling path (log to DLQ + skip vs fail).

11. Implement a circuit breaker: if a task fails 3 consecutive times across different DAG runs, raise `AirflowFailException` with a message to investigate manually.

12. Write a test that verifies your DAG has `catchup=False` and `max_active_runs=1`.

13. Create a monitoring DAG that runs hourly and checks:
    - Are any DAGs in "failed" state?
    - Are any tasks running longer than expected?
    - Is the warehouse data fresh (last ETL < 24 hours ago)?

14. **Error simulation:** Deliberately introduce 3 different types of errors in your DAG and observe how Airflow handles each:
    - Connection error (wrong connection ID)
    - SQL error (bad query)
    - Business logic error (validation failure)

15. Add proper logging to all tasks: log start time, end time, and records processed.

16. Write a test for the quality_checks function that covers:
    - All checks pass
    - Orphan records found
    - Row count mismatch
    - SCD integrity failure

17. Implement idempotent loading: ensure every task can be safely retried without duplicating data.

18. Create a "DAG health dashboard" SQL query that shows:
    - Each DAG's last run time, status, duration
    - Average duration over last 7 days
    - Any failures in the last 24 hours

19. Write a callback that creates a Jira-style ticket in a `tasks` table when a DAG fails.

20. **Documentation:** Write a complete README for your Airflow project with:
    - Setup instructions
    - DAG descriptions
    - Connection configuration
    - Variable list
    - Troubleshooting guide

21. Add parameterized runs: allow triggering the DAG with `--conf '{"start_date": "2026-06-01", "end_date": "2026-06-10"}'` for custom backfills.

22. Create a "dry run" mode: when triggered with `{"dry_run": true}`, all tasks log what they WOULD do without actually doing it.

23. Write integration tests that run tasks against a real (test) database.

24. Implement a SLA monitoring pattern: if the daily ETL hasn't completed by 8 AM, trigger an alert.

25. **Challenge:** Apply ALL best practices from today to your MakanExpress warehouse ETL DAG. The final result should be production-ready with:
    - Proper error handling
    - Retries with exponential backoff
    - Idempotent tasks
    - Data quality checks
    - Dead letter queue
    - Logging and monitoring
    - Unit tests
    - on_failure_callback
    - Complete documentation

---

### ✅ Selected Answers

<details>
<summary>🔑 Click to reveal selected answers</summary>

**Answer 2:**
```python
from airflow.exceptions import AirflowFailException, AirflowSkipException

def smart_load(**context):
    table = context['params'].get('table')
    if not table:
        raise AirflowFailException("Missing table parameter — will NOT retry")
    
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    count = hook.get_first(f"SELECT COUNT(*) FROM {table}")[0]
    
    if count == 0:
        raise AirflowSkipException(f"No data in {table} — skipping")
    
    print(f"Processing {count} rows from {table}")
```

**Answer 5:**
```python
def on_failure(context):
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    hook.run("""
        INSERT INTO etl_audit_log (event_type, dag_id, task_id, logical_date, error, event_time)
        VALUES ('failure', %s, %s, %s, %s, NOW())
    """, parameters=(
        context['dag'].dag_id,
        context['task'].task_id,
        str(context['logical_date']),
        str(context.get('exception', 'Unknown'))
    ))

def on_success(context):
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    hook.run("""
        INSERT INTO etl_audit_log (event_type, dag_id, task_id, logical_date, event_time)
        VALUES ('success', %s, %s, %s, NOW())
    """, parameters=(
        context['dag'].dag_id,
        context['task'].task_id,
        str(context['logical_date']),
    ))
```

**Answer 6:**
```python
# tests/test_dag.py
import pytest
from airflow.models import DagBag

@pytest.fixture
def dagbag():
    return DagBag(dag_folder='dags/', include_examples=False)

def test_dag_loads(dagbag):
    assert len(dagbag.import_errors) == 0
    dag = dagbag.get_dag('makanexpress_warehouse_etl')
    assert dag is not None

def test_tasks_exist(dagbag):
    dag = dagbag.get_dag('makanexpress_warehouse_etl')
    task_ids = {t.task_id for t in dag.tasks}
    expected = {'extract_and_validate', 'load_dim_customer', 'load_fact', 'quality_checks', 'notify'}
    assert expected.issubset(task_ids)

def test_catchup_false(dagbag):
    dag = dagbag.get_dag('makanexpress_warehouse_etl')
    assert dag.catchup is False

def test_max_active_runs(dagbag):
    dag = dagbag.get_dag('makanexpress_warehouse_etl')
    assert dag.max_active_runs == 1

def test_dependencies(dagbag):
    dag = dagbag.get_dag('makanexpress_warehouse_etl')
    extract = dag.get_task('extract_and_validate')
    downstream_ids = {t.task_id for t in extract.downstream_list}
    assert 'load_dim_customer' in downstream_ids
```

**Answer 22:**
```python
def load_fact(**context):
    dry_run = context['params'].get('dry_run', False)
    ds = context['ds']
    
    if dry_run:
        print(f"[DRY RUN] Would load fact data for {ds}")
        hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
        count = hook.get_first("SELECT COUNT(*) FROM raw_orders WHERE DATE(order_date) = %s", (ds,))[0]
        print(f"[DRY RUN] Would insert ~{count} records")
        return count
    
    # Real execution
    hook = PostgresHook(postgres_conn_id=CONNECTION_ID)
    hook.run("INSERT INTO fact_order_items SELECT ...")
```

</details>

---

## 🌙 Evening Block (1 hour): Review

### Production Readiness Checklist

```
□ catchup=False
□ max_active_runs=1
□ retries >= 2
□ retry_exponential_backoff=True
□ execution_timeout on long tasks
□ Idempotent tasks (DELETE + INSERT or NOT EXISTS)
□ No hardcoded credentials (use Connections)
□ Configurable values use Variables
□ on_failure_callback set
□ Proper logging
□ Data quality checks
□ Unit tests pass
□ Documentation complete
```

### 📝 Today's Checklist

- [ ] I understand retry strategies and when NOT to retry
- [ ] I can implement dead letter queue pattern
- [ ] I can write unit tests for DAGs
- [ ] I understand idempotency and why it matters
- [ ] I can set up monitoring and alerting callbacks
- [ ] I can apply the production checklist to any DAG
- [ ] I completed at least 15 of the 25 exercises
- [ ] I pushed code to GitHub

---

*Day 33 complete! That's the 3-day checkpoint done (Day 31-33).* ✅
*Next: Day 34 (Assessment) and Day 35 (Portfolio Project).* 📝🏗️
