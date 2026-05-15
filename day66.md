# 📅 Day 66 — Saturday, 19 July 2026
# 🔧 Add Data Quality to All Existing Projects

---

## 🎯 Today's Goal

Theory is done. Today you add data quality checks to ALL your portfolio projects. This is what separates a junior engineer from a professional — every production pipeline has data quality gates.

---

## ☀️ Morning Block (2 hours): Add DQ to API Pipeline + dbt Project

### Project 1: API Data Pipeline

Add a data quality gate between transform and load:

```python
# Add to run_pipeline.py

from src.data_quality.custom_checks import CustomDataQualityChecker

def run_pipeline_with_dq():
    """Pipeline with data quality gate."""
    # ... extract and transform (same as Day 58) ...
    
    # === DATA QUALITY GATE ===
    logger.info("--- DATA QUALITY GATE ---")
    
    checker = CustomDataQualityChecker()
    
    # Weather data checks
    weather_rules = [
        {"type": "not_null", "column": "city"},
        {"type": "not_null", "column": "temperature_c"},
        {"type": "range", "column": "temperature_c", "min": -10, "max": 50},
        {"type": "range", "column": "humidity_pct", "min": 0, "max": 100},
        {"type": "row_count", "min": 3, "max": 10},  # At least 3 cities
    ]
    weather_result = checker.run_all(weather_df, "weather", weather_rules)
    
    # Crypto data checks
    crypto_rules = [
        {"type": "not_null", "column": "coin_id"},
        {"type": "not_null", "column": "price_sgd"},
        {"type": "range", "column": "price_sgd", "min": 0.01, "max": 1000000},
        {"type": "row_count", "min": 3, "max": 10},
    ]
    crypto_result = checker.run_all(crypto_df, "crypto", crypto_rules)
    
    # Gate decision
    all_passed = weather_result["all_passed"] and crypto_result["all_passed"]
    
    if not all_passed:
        logger.error("❌ DATA QUALITY GATE FAILED — stopping pipeline")
        checker.print_report()
        return {"status": "dq_failed", "checks": checker.get_summary()}
    
    logger.info("✅ Data quality gate passed — proceeding to load")
    
    # ... continue with load ...
```

### Project 2: Food Delivery dbt

Add advanced tests to your dbt models:

```yaml
# models/marts/schema.yml

version: 2

models:
  - name: fact_orders
    description: "Order fact table with SCD Type 2"
    tests:
      - dbt_utils.expression_is_true:
          expression: "total_amount >= 0"
      - dbt_utils.recency:
          datepart: day
          field: order_date
          interval: 2
    columns:
      - name: order_key
        tests:
          - unique
          - not_null
      - name: customer_key
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_key
      - name: restaurant_key
        tests:
          - not_null
          - relationships:
              to: ref('dim_restaurants')
              field: restaurant_key
      - name: total_amount
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 10000
              inclusive: true
      - name: order_status
        tests:
          - accepted_values:
              values: ['pending', 'preparing', 'delivered', 'cancelled']

  - name: dim_restaurants
    columns:
      - name: restaurant_key
        tests:
          - unique
          - not_null
      - name: rating
        tests:
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 5
      - name: is_current
        tests:
          - accepted_values:
              values: [true, false]
```

### Custom dbt Tests

```sql
-- tests/fact_orders_reasonable_volume.sql
SELECT order_date, COUNT(*) as cnt
FROM {{ ref('fact_orders') }}
GROUP BY order_date
HAVING COUNT(*) < 10 OR COUNT(*) > 50000
```

```sql
-- tests/no_duplicate_current_records.sql
SELECT restaurant_key, COUNT(*) as cnt
FROM {{ ref('dim_restaurants') }}
WHERE is_current = true
GROUP BY restaurant_key
HAVING COUNT(*) > 1
```

### Morning Exercises

**Exercise 1:** Add data quality checks to your Airflow DAG from Week 5. Insert a `data_quality_check` task between transform and load that fails the pipeline if checks don't pass.

<details>
<summary>🔑 Answer</summary>

```python
# In your Airflow DAG, add:

from airflow.operators.python import PythonOperator

def run_data_quality_checks(**context):
    ti = context["ti"]
    
    # Pull transformed data from XCom
    weather_data = ti.xcom_pull(key="weather_df", task_ids="transform")
    crypto_data = ti.xcom_pull(key="crypto_df", task_ids="transform")
    
    if not weather_data or not crypto_data:
        raise ValueError("No data received from transform step")
    
    import pandas as pd
    weather_df = pd.DataFrame(weather_data)
    crypto_df = pd.DataFrame(crypto_data)
    
    checker = CustomDataQualityChecker()
    
    # Check weather
    weather_rules = [
        {"type": "not_null", "column": "city"},
        {"type": "range", "column": "temperature_c", "min": -10, "max": 50},
        {"type": "row_count", "min": 3, "max": 10},
    ]
    w_result = checker.run_all(weather_df, "weather", weather_rules)
    
    # Check crypto
    crypto_rules = [
        {"type": "not_null", "column": "coin_id"},
        {"type": "range", "column": "price_sgd", "min": 0.01, "max": 1000000},
    ]
    c_result = checker.run_all(crypto_df, "crypto", crypto_rules)
    
    if not w_result["all_passed"] or not c_result["all_passed"]:
        checker.print_report()
        raise ValueError("Data quality gate failed!")
    
    return "All checks passed"

# Add to DAG
dq_check = PythonOperator(
    task_id="data_quality_check",
    python_callable=run_data_quality_checks,
)

# Update dependencies
[extract_weather, extract_crypto] >> transform >> dq_check >> [load_postgres, load_s3]
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Add DQ to Remaining Projects

### Project 3: Crypto Data Pipeline (Python ETL)

```python
# Add to crypto pipeline: after transform, before load

def validate_crypto_data(df):
    """Validate crypto data before loading."""
    issues = []
    
    # Required columns
    required = ["coin_id", "price_sgd", "fetched_at"]
    for col in required:
        if col not in df.columns:
            issues.append(f"Missing column: {col}")
        elif df[col].isnull().any():
            issues.append(f"Null values in {col}: {df[col].isnull().sum()}")
    
    # Price sanity
    if "price_sgd" in df.columns:
        negative = df[df["price_sgd"] < 0]
        if len(negative) > 0:
            issues.append(f"{len(negative)} negative prices found")
        
        extreme = df[df["price_sgd"] > 1000000]
        if len(extreme) > 0:
            issues.append(f"{len(extreme)} extreme prices found")
    
    # Volume check
    if len(df) == 0:
        issues.append("Empty DataFrame — no data to load")
    
    if issues:
        print("❌ Data quality issues found:")
        for issue in issues:
            print(f"  - {issue}")
        return False
    
    print("✅ Data quality check passed")
    return True

# In pipeline:
cleaned_df = transform(raw_df)
if not validate_crypto_data(cleaned_df):
    logger.error("Pipeline stopped: data quality check failed")
    sys.exit(1)
load_to_postgres(cleaned_df)
```

### Project 4: Food Delivery Warehouse

Add SQL checks to your warehouse loading scripts:

```python
def post_load_validation(conn_params):
    """Validate data after loading into warehouse."""
    conn = psycopg2.connect(**conn_params)
    cur = conn.cursor()
    
    checks = {
        "fact_orders_row_count": "SELECT COUNT(*) FROM fact_orders",
        "dim_customers_row_count": "SELECT COUNT(*) FROM dim_customers",
        "dim_restaurants_row_count": "SELECT COUNT(*) FROM dim_restaurants",
        "orphan_order_customers": """
            SELECT COUNT(*) FROM fact_orders f
            LEFT JOIN dim_customers c ON f.customer_key = c.customer_key
            WHERE c.customer_key IS NULL
        """,
        "orphan_order_restaurants": """
            SELECT COUNT(*) FROM fact_orders f
            LEFT JOIN dim_restaurants r ON f.restaurant_key = r.restaurant_key
            WHERE r.restaurant_key IS NULL
        """,
        "negative_amounts": "SELECT COUNT(*) FROM fact_orders WHERE total_amount < 0",
    }
    
    print("📋 Post-Load Validation")
    print("-" * 50)
    
    for name, sql in checks.items():
        cur.execute(sql)
        result = cur.fetchone()[0]
        
        if "orphan" in name or "negative" in name:
            status = "✅" if result == 0 else f"❌ ({result} issues)"
        else:
            status = f"📊 {result:,} rows"
        
        print(f"  {status} {name}")
    
    cur.close()
    conn.close()
```

---

### Afternoon Exercises

**Exercise 2:** Create a `data_quality_summary.md` file in each of your portfolio projects that documents: what checks are implemented, what they catch, and how to run them.

<details>
<summary>🔑 Answer</summary>

```markdown
# Data Quality Checks

## Implemented Checks

### Extract Layer
- API response status code validation
- JSON parse error handling
- Timeout handling with retry

### Transform Layer
- Null check on required fields (city, coin_id, price)
- Range check on numeric values (temperature -10 to 50°C, price > 0)
- Accepted values check (status field)
- Row count bounds check

### Load Layer
- PostgreSQL upsert conflict handling
- S3 upload error handling with local fallback

### Post-Load
- Row count validation
- Referential integrity check
- Negative amount detection

## Running Checks
\`\`\`bash
# Run pipeline with DQ gate
python run_pipeline.py

# Run standalone DQ checks
python -m src.data_quality.run_checks

# Run dbt tests
dbt test --project-dir food-delivery-dbt
\`\`\`

## What Gets Caught
- API downtime or rate limiting → retry + alert
- Missing/null fields → pipeline stops before load
- Out-of-range values → pipeline stops before load
- Duplicate records → upsert handles gracefully
- Schema changes → migration detector alerts
```

</details>

**Exercise 3:** Add a daily data quality report script that runs all checks across all projects and writes a single report to `data_quality/reports/YYYY-MM-DD.md`.

<details>
<summary>🔑 Answer</summary>

```python
import os
from datetime import datetime

def generate_daily_dq_report(conn_params, output_dir="data_quality/reports"):
    """Generate a daily DQ report across all projects."""
    os.makedirs(output_dir, exist_ok=True)
    
    today = datetime.now().strftime("%Y-%m-%d")
    report_path = f"{output_dir}/{today}.md"
    
    report = f"""# Data Quality Report — {today}

## API Data Pipeline
"""
    
    # Check weather API
    try:
        import requests
        r = requests.get("https://api.open-meteo.com/v1/forecast",
                        params={"latitude": 1.35, "longitude": 103.82, "current": "temperature_2m"},
                        timeout=10)
        report += f"- Weather API: {'✅ OK' if r.ok else '❌ DOWN'}\n"
    except:
        report += "- Weather API: ❌ UNREACHABLE\n"
    
    # Check database
    try:
        conn = psycopg2.connect(**conn_params)
        cur = conn.cursor()
        
        tables = ["orders", "restaurants", "customers", "fact_orders", "dim_customers"]
        report += "\n## Database Tables\n"
        for t in tables:
            try:
                cur.execute(f"SELECT COUNT(*) FROM {t}")
                count = cur.fetchone()[0]
                report += f"- {t}: {count:,} rows ✅\n"
            except:
                report += f"- {t}: not found ⚠️\n"
        
        conn.close()
    except:
        report += "\n## Database: ❌ CONNECTION FAILED\n"
    
    report += f"\n---\nGenerated: {datetime.now().isoformat()}\n"
    
    with open(report_path, "w") as f:
        f.write(report)
    
    print(f"✅ Report saved to {report_path}")
    return report_path

generate_daily_dq_report(db_params)
```

</details>

---

## 🌙 Evening: Push + Review

### 📝 Today's Checklist

- [ ] Added DQ gate to API Data Pipeline
- [ ] Added dbt tests to Food Delivery Analytics
- [ ] Added validation to Crypto Data Pipeline
- [ ] Added post-load checks to Food Delivery Warehouse
- [ ] Created DQ documentation for each project
- [ ] All changes pushed to GitHub
- [ ] Ready for tomorrow: SQL Interview Practice

---

*Day 66 complete! Your pipelines now have professional-grade data quality.* 🛡️
