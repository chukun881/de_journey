# 📅 Day 64 — Thursday, 17 July 2026
# 🛡️ Data Quality: Great Expectations, dbt Tests, Custom Checks

---

## 🎯 Today's Goal

Bad data is worse than no data. If your pipeline loads incorrect data, every downstream report, dashboard, and decision is wrong. Today you learn to build **data quality checks** — automated tests that catch problems before they reach production.

**Real-world scenario:** Your MakanExpress pipeline loads 10,000 orders daily. One day, a bug causes all `total_amount` values to be 0. Without data quality checks, nobody notices until the finance team asks "why is revenue $0 this week?"

**By end of day:** You'll have data quality checks using 3 approaches: Great Expectations, dbt tests, and custom Python checks.

---

## ☀️ Morning Block (2 hours): Why Data Quality Matters + Great Expectations

### The Data Quality Framework

```
┌──────────────────────────────────────────────────────┐
│              DATA QUALITY PYRAMID                     │
│                                                       │
│                    ┌─────────┐                        │
│                    │  FRESH  │ ← Is data up to date?  │
│                   ┌┴─────────┴┐                       │
│                   │ CONSISTENT │ ← Do values match?   │
│                  ┌┴───────────┴┐                      │
│                  │  COMPLETE   │ ← Any missing data?  │
│                 ┌┴─────────────┴┐                     │
│                 │   ACCURATE    │ ← Are values right? │
│                ┌┴───────────────┴┐                    │
│                │    UNIQUE       │ ← Any duplicates?  │
│               ┌┴─────────────────┴┐                   │
│               │     NOT NULL      │ ← Required fields? │
│               └───────────────────┘                   │
│                                                       │
│ Each layer builds on the one below it.                │
└──────────────────────────────────────────────────────┘
```

### Common Data Quality Checks for Data Engineers

| Check Type | What It Tests | Example |
|-----------|--------------|---------|
| **Not Null** | Required fields have values | `order_id` must never be NULL |
| **Unique** | No duplicate records | Each `order_id` appears once |
| **Range** | Values within expected bounds | `total_amount` between 0 and 10000 |
| **Referential Integrity** | Foreign keys exist | Every `restaurant_id` exists in restaurants table |
| **Accepted Values** | Enum/categorical checks | `status` is one of: pending, delivered, cancelled |
| **Freshness** | Data is recent enough | Last order was within 24 hours |
| **Volume** | Row count in expected range | Between 100 and 50,000 orders per day |
| **Schema** | Column names/types haven't changed | `total_amount` is still DECIMAL |
| **Distribution** | Statistical distribution is normal | Revenue doesn't suddenly drop 90% |
| **Custom SQL** | Business-specific rules | No orders with negative amounts |

---

### Great Expectations: Introduction

[Great Expectations](https://greatexpectations.io/) (GX) is a Python data quality framework. You define **expectations** (rules) and it validates your data against them.

```bash
pip install great_expectations
```

#### Basic Usage

```python
import great_expectations as gx
import pandas as pd

# Create sample data (MakanExpress orders)
df = pd.DataFrame({
    "order_id": [1001, 1002, 1003, 1004, 1005, 1006, 1007],
    "customer_id": [201, 202, 203, 204, 205, 206, 201],
    "restaurant_id": [10, 11, 10, 12, 10, 11, 13],
    "total_amount": [25.50, 15.00, 42.00, 8.50, 0, 33.00, 120.00],
    "status": ["delivered", "delivered", "cancelled", "delivered", "delivered", "pending", "delivered"],
    "order_date": pd.to_datetime(["2026-07-17", "2026-07-17", "2026-07-16", "2026-07-17", 
                                   "2026-07-17", "2026-07-17", "2026-07-17"]),
    "delivery_city": ["Singapore", "Kuala Lumpur", "Singapore", "Penang", "Singapore", "KL", "Singapore"]
})

# Create a GX context
context = gx.get_context()

# Create a data source and asset
data_source = context.data_sources.add_pandas("makanexpress_datasource")
data_asset = data_source.add_dataframe_asset(name="orders_asset")

# Create a batch definition and batch
batch_definition = data_asset.add_batch_definition_whole_dataframe("orders_batch")
batch = batch_definition.get_batch(batch_parameters={"dataframe": df})

# Create an expectation suite
suite = context.suites.add(gx.ExpectationSuite(name="orders_expectations"))

# Create a validation definition
validation = context.validation_definitions.add(
    gx.ValidationDefinition(
        name="orders_validation",
        data=batch_definition,
        suite=suite
    )
)
```

#### Defining Expectations

```python
# Add expectations to the suite
suite = context.suites.get("orders_expectations")

# 1. Not Null — order_id must never be null
suite.add_expectation(
    gx.expectations.ExpectColumnValuesToNotBeNull(column="order_id")
)

# 2. Unique — no duplicate order_ids
suite.add_expectation(
    gx.expectations.ExpectColumnValuesToBeUnique(column="order_id")
)

# 3. Range — total_amount between 0.01 and 1000
suite.add_expectation(
    gx.expectations.ExpectColumnValuesToBeBetween(
        column="total_amount",
        min_value=0.01,
        max_value=1000
    )
)

# 4. Accepted Values — status must be valid
suite.add_expectation(
    gx.expectations.ExpectColumnValuesToBeInSet(
        column="status",
        value_set=["pending", "preparing", "on_the_way", "delivered", "cancelled"]
    )
)

# 5. Table row count — reasonable range
suite.add_expectation(
    gx.expectations.ExpectTableRowCountToBeBetween(
        min_value=1,
        max_value=100000
    )
)

# 6. Column exists
suite.add_expectation(
    gx.expectations.ExpectColumnToExist(column="order_date")
)

# Save the suite
suite = context.suites.add(suite)
```

#### Running Validation

```python
# Run validation
result = validation.run(batch_parameters={"dataframe": df})

# Check results
print(f"Success: {result.success}")
print(f"Evaluated expectations: {result.results}")

for r in result.results:
    expectation_type = r["expectation_config"]["type"]
    success = r["success"]
    status = "✅ PASS" if success else "❌ FAIL"
    print(f"  {status} — {expectation_type}")
```

---

### Building a Reusable Data Quality Checker

```python
# src/data_quality/checker.py

import great_expectations as gx
import pandas as pd
import logging
from datetime import datetime

logger = logging.getLogger(__name__)

class DataQualityChecker:
    """Reusable data quality checker using Great Expectations."""
    
    def __init__(self, name="data_quality"):
        self.context = gx.get_context()
        self.name = name
        self.results = []
    
    def check_dataframe(self, df, table_name, rules):
        """
        Validate a DataFrame against rules.
        
        rules format: list of dicts
        [
            {"type": "not_null", "column": "order_id"},
            {"type": "unique", "column": "order_id"},
            {"type": "range", "column": "total_amount", "min": 0, "max": 10000},
            {"type": "in_set", "column": "status", "values": ["pending", "delivered"]},
            {"type": "row_count", "min": 1, "max": 100000},
        ]
        """
        # Setup GX
        data_source = self.context.data_sources.add_pandas(f"{table_name}_ds")
        data_asset = data_source.add_dataframe_asset(name=f"{table_name}_asset")
        batch_def = data_asset.add_batch_definition_whole_dataframe(f"{table_name}_batch")
        
        # Create suite
        suite = self.context.suites.add(
            gx.ExpectationSuite(name=f"{table_name}_suite")
        )
        
        # Add expectations from rules
        for rule in rules:
            exp = self._rule_to_expectation(rule)
            if exp:
                suite.add_expectation(exp)
        
        # Validate
        validation = self.context.validation_definitions.add(
            gx.ValidationDefinition(
                name=f"{table_name}_validation",
                data=batch_def,
                suite=suite
            )
        )
        
        result = validation.run(batch_parameters={"dataframe": df})
        
        # Collect results
        check_results = {
            "table": table_name,
            "timestamp": datetime.now().isoformat(),
            "success": result.success,
            "total_checks": len(result.results),
            "passed": sum(1 for r in result.results if r["success"]),
            "failed": sum(1 for r in result.results if not r["success"]),
            "details": []
        }
        
        for r in result.results:
            check_results["details"].append({
                "expectation": r["expectation_config"]["type"],
                "success": r["success"],
                "column": r["expectation_config"]["kwargs"].get("column", "N/A")
            })
        
        self.results.append(check_results)
        return check_results
    
    def _rule_to_expectation(self, rule):
        """Convert a rule dict to a Great Expectations expectation."""
        rule_type = rule["type"]
        
        if rule_type == "not_null":
            return gx.expectations.ExpectColumnValuesToNotBeNull(column=rule["column"])
        elif rule_type == "unique":
            return gx.expectations.ExpectColumnValuesToBeUnique(column=rule["column"])
        elif rule_type == "range":
            return gx.expectations.ExpectColumnValuesToBeBetween(
                column=rule["column"], min_value=rule.get("min"), max_value=rule.get("max")
            )
        elif rule_type == "in_set":
            return gx.expectations.ExpectColumnValuesToBeInSet(
                column=rule["column"], value_set=rule["values"]
            )
        elif rule_type == "row_count":
            return gx.expectations.ExpectTableRowCountToBeBetween(
                min_value=rule.get("min", 0), max_value=rule.get("max", 10**9)
            )
        elif rule_type == "exists":
            return gx.expectations.ExpectColumnToExist(column=rule["column"])
        else:
            logger.warning(f"Unknown rule type: {rule_type}")
            return None
    
    def print_report(self):
        """Print a summary report of all checks."""
        print("\n" + "=" * 60)
        print("📊 DATA QUALITY REPORT")
        print("=" * 60)
        
        for result in self.results:
            status = "✅ PASS" if result["success"] else "❌ FAIL"
            print(f"\n{status} {result['table']}: {result['passed']}/{result['total_checks']} checks passed")
            
            for detail in result["details"]:
                check_status = "✅" if detail["success"] else "❌"
                print(f"  {check_status} {detail['expectation']} on {detail['column']}")
        
        total_passed = sum(r["passed"] for r in self.results)
        total_checks = sum(r["total_checks"] for r in self.results)
        print(f"\n{'='*60}")
        print(f"Overall: {total_passed}/{total_checks} checks passed")
```

---

### Morning Exercises

**Exercise 1:** Define 8 data quality rules for the `restaurants` table: id not null, id unique, name not null, rating between 0-5, city in accepted values (Singapore, KL, Penang, JB), cuisine not null, avg_delivery_time between 5-120 minutes, and table has at least 1 row. Write the rule definitions as a list of dicts.

<details>
<summary>🔑 Answer</summary>

```python
restaurant_rules = [
    {"type": "not_null", "column": "id"},
    {"type": "unique", "column": "id"},
    {"type": "not_null", "column": "name"},
    {"type": "range", "column": "rating", "min": 0, "max": 5},
    {"type": "in_set", "column": "city", "values": ["Singapore", "Kuala Lumpur", "Penang", "Johor Bahru"]},
    {"type": "not_null", "column": "cuisine"},
    {"type": "range", "column": "avg_delivery_time_min", "min": 5, "max": 120},
    {"type": "row_count", "min": 1},
]

# Test with sample data
restaurants_df = pd.DataFrame({
    "id": [1, 2, 3, 4],
    "name": ["Tian Tian", "Village Park", "Song Fa", "Roti King"],
    "cuisine": ["Chinese", "Malay", "Chinese", "Indian-Muslim"],
    "city": ["Singapore", "Kuala Lumpur", "Singapore", "KL"],
    "rating": [4.5, 4.3, 4.4, 4.1],
    "avg_delivery_time_min": [25, 35, 30, 20]
})

checker = DataQualityChecker("makanexpress")
result = checker.check_dataframe(restaurants_df, "restaurants", restaurant_rules)
checker.print_report()
```

</details>

**Exercise 2:** Write a custom data quality check using plain SQL (no GX). The check should find: orders with negative amounts, orders with future dates, and orders with invalid restaurant_ids. Return the count of violations for each.

<details>
<summary>🔑 Answer</summary>

```python
import psycopg2

def run_custom_sql_checks(conn_params):
    """Run custom SQL data quality checks."""
    conn = psycopg2.connect(**conn_params)
    cur = conn.cursor()
    
    checks = {
        "negative_amounts": """
            SELECT COUNT(*) FROM orders WHERE total_amount < 0
        """,
        "future_orders": """
            SELECT COUNT(*) FROM orders 
            WHERE order_date > NOW() + INTERVAL '1 day'
        """,
        "invalid_restaurant": """
            SELECT COUNT(*) FROM orders o
            LEFT JOIN restaurants r ON o.restaurant_id = r.id
            WHERE r.id IS NULL
        """,
        "duplicate_orders": """
            SELECT COUNT(*) - COUNT(DISTINCT id) FROM orders
        """,
        "zero_amount_orders": """
            SELECT COUNT(*) FROM orders WHERE total_amount = 0 AND status = 'delivered'
        """
    }
    
    print("🔍 Custom SQL Data Quality Checks")
    print("-" * 45)
    
    for check_name, sql in checks.items():
        try:
            cur.execute(sql)
            count = cur.fetchone()[0]
            status = "✅" if count == 0 else f"❌ ({count} violations)"
            print(f"  {status} {check_name}")
        except Exception as e:
            print(f"  ⚠️ {check_name}: Error — {e}")
    
    cur.close()
    conn.close()

# Test
db_params = {"host": "localhost", "port": "5432", "dbname": "makanexpress",
             "user": "postgres", "password": "postgres"}
run_custom_sql_checks(db_params)
```

</details>

**Exercise 3:** Write a freshness check that queries the `orders` table and alerts if the most recent order is older than 24 hours.

<details>
<summary>🔑 Answer</summary>

```python
from datetime import datetime, timedelta

def check_data_freshness(conn_params, table="orders", date_column="order_date", 
                         max_age_hours=24):
    """Check if data is fresh (recent enough)."""
    conn = psycopg2.connect(**conn_params)
    cur = conn.cursor()
    
    cur.execute(f"""
        SELECT MAX({date_column}) FROM {table}
    """)
    
    latest = cur.fetchone()[0]
    cur.close()
    conn.close()
    
    if latest is None:
        print(f"❌ No data found in {table}")
        return False
    
    if isinstance(latest, str):
        latest = datetime.fromisoformat(latest)
    
    age = datetime.now() - latest
    age_hours = age.total_seconds() / 3600
    
    is_fresh = age_hours <= max_age_hours
    status = "✅ FRESH" if is_fresh else "❌ STALE"
    
    print(f"{status} {table}.{date_column}")
    print(f"  Latest record: {latest}")
    print(f"  Age: {age_hours:.1f} hours (max: {max_age_hours})")
    
    return is_fresh

check_data_freshness(db_params, max_age_hours=24)
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Testing Patterns

### dbt Tests (Review + Advanced)

You learned dbt basics in Week 3. Now let's add serious data quality:

```yaml
# models/schema.yml — Enhanced tests for MakanExpress

version: 2

models:
  - name: stg_orders
    description: "Cleaned orders data from MakanExpress"
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers')
              field: customer_id
      
      - name: total_amount
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0.01
              max_value: 10000
      
      - name: status
        tests:
          - accepted_values:
              values: ['pending', 'preparing', 'on_the_way', 'delivered', 'cancelled']
      
      - name: order_date
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: "'2024-01-01'"
              max_value: "'2027-01-01'"

  - name: stg_restaurants
    description: "Restaurant master data"
    columns:
      - name: restaurant_id
        tests:
          - unique
          - not_null
      
      - name: rating
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 5
      
      - name: cuisine_type
        tests:
          - not_null
          - accepted_values:
              values: ['Chinese', 'Malay', 'Indian', 'Western', 'Japanese', 'Korean', 'Thai', 'Nyonya', 'Other']
```

### Custom dbt Tests

```sql
-- tests/no_cancelled_orders_with_positive_amount.sql
-- Business rule: Cancelled orders should have total_amount of 0

SELECT order_id, total_amount, status
FROM {{ ref('stg_orders') }}
WHERE status = 'cancelled' AND total_amount > 0
```

```sql
-- tests/reasonable_daily_order_volume.sql
-- Volume check: Each day should have between 50 and 10,000 orders

SELECT order_date, COUNT(*) as order_count
FROM {{ ref('stg_orders') }}
GROUP BY order_date
HAVING COUNT(*) < 50 OR COUNT(*) > 10000
```

```sql
-- tests/no_future_order_dates.sql
-- Freshness check: No orders with dates in the future

SELECT order_id, order_date
FROM {{ ref('stg_orders') }}
WHERE order_date > CURRENT_DATE + INTERVAL '1 day'
```

---

### Python Custom Checks (Without GX)

```python
# src/data_quality/custom_checks.py

import pandas as pd
import numpy as np
import logging
from datetime import datetime, timedelta

logger = logging.getLogger(__name__)

class CustomDataQualityChecker:
    """Lightweight data quality checks without external dependencies."""
    
    def __init__(self):
        self.results = []
    
    def _record(self, check_name, passed, details="", severity="error"):
        self.results.append({
            "check": check_name,
            "passed": passed,
            "details": details,
            "severity": severity,
            "timestamp": datetime.now().isoformat()
        })
        status = "✅" if passed else "❌"
        logger.info(f"{status} {check_name}: {details}")
    
    def check_not_null(self, df, column):
        """Check that column has no null values."""
        null_count = df[column].isnull().sum()
        passed = null_count == 0
        self._record(f"not_null({column})", passed, 
                     f"{null_count} null values found" if not passed else "No nulls")
        return passed
    
    def check_unique(self, df, column):
        """Check that column values are unique."""
        dupes = df[column].duplicated().sum()
        passed = dupes == 0
        self._record(f"unique({column})", passed,
                     f"{dupes} duplicate values found" if not passed else "All unique")
        return passed
    
    def check_range(self, df, column, min_val=None, max_val=None):
        """Check that values fall within expected range."""
        violations = 0
        if min_val is not None:
            violations += (df[column] < min_val).sum()
        if max_val is not None:
            violations += (df[column] > max_val).sum()
        
        passed = violations == 0
        self._record(f"range({column}, {min_val}-{max_val})", passed,
                     f"{violations} out-of-range values" if not passed else "All in range")
        return passed
    
    def check_accepted_values(self, df, column, accepted):
        """Check that column only contains accepted values."""
        invalid = df[~df[column].isin(accepted)]
        passed = len(invalid) == 0
        if not passed:
            bad_values = invalid[column].unique().tolist()
            self._record(f"accepted_values({column})", passed, 
                         f"Invalid values: {bad_values}")
        else:
            self._record(f"accepted_values({column})", passed, "All values accepted")
        return passed
    
    def check_row_count(self, df, min_rows=1, max_rows=10**9):
        """Check row count is within expected range."""
        count = len(df)
        passed = min_rows <= count <= max_rows
        self._record(f"row_count({min_rows}-{max_rows})", passed,
                     f"Got {count} rows")
        return passed
    
    def check_referential_integrity(self, df, column, ref_df, ref_column):
        """Check foreign key integrity."""
        orphan = df[~df[column].isin(ref_df[ref_column])]
        passed = len(orphan) == 0
        self._record(f"referential_integrity({column} → {ref_column})", passed,
                     f"{len(orphan)} orphan records" if not passed else "All references valid")
        return passed
    
    def check_freshness(self, df, date_column, max_age_hours=24):
        """Check data is fresh enough."""
        latest = pd.to_datetime(df[date_column]).max()
        age = (pd.Timestamp.now() - latest).total_seconds() / 3600
        passed = age <= max_age_hours
        self._record(f"freshness({date_column}, {max_age_hours}h)", passed,
                     f"Latest: {latest}, Age: {age:.1f}h")
        return passed
    
    def check_no_outliers(self, df, column, z_threshold=3):
        """Check for statistical outliers using z-score."""
        mean = df[column].mean()
        std = df[column].std()
        if std == 0:
            self._record(f"no_outliers({column})", True, "No variation in data")
            return True
        
        z_scores = ((df[column] - mean) / std).abs()
        outliers = (z_scores > z_threshold).sum()
        passed = outliers == 0
        self._record(f"no_outliers({column}, z>{z_threshold})", passed,
                     f"{outliers} outlier values detected" if not passed else "No outliers")
        return passed
    
    def run_all(self, df, table_name, config):
        """Run all configured checks."""
        print(f"\n🔍 Running data quality checks for {table_name}")
        print("-" * 45)
        
        for check in config:
            check_type = check["type"]
            if check_type == "not_null":
                self.check_not_null(df, check["column"])
            elif check_type == "unique":
                self.check_unique(df, check["column"])
            elif check_type == "range":
                self.check_range(df, check["column"], check.get("min"), check.get("max"))
            elif check_type == "accepted_values":
                self.check_accepted_values(df, check["column"], check["values"])
            elif check_type == "row_count":
                self.check_row_count(df, check.get("min", 1), check.get("max", 10**9))
            elif check_type == "freshness":
                self.check_freshness(df, check["column"], check.get("max_age_hours", 24))
            elif check_type == "no_outliers":
                self.check_no_outliers(df, check["column"], check.get("z_threshold", 3))
        
        return self.get_summary()
    
    def get_summary(self):
        """Get summary of all checks."""
        total = len(self.results)
        passed = sum(1 for r in self.results if r["passed"])
        failed = total - passed
        
        return {
            "total": total,
            "passed": passed,
            "failed": failed,
            "success_rate": f"{passed/total*100:.1f}%" if total > 0 else "N/A",
            "all_passed": failed == 0,
            "details": self.results
        }
    
    def print_report(self):
        """Print formatted report."""
        summary = self.get_summary()
        status = "✅ ALL PASSED" if summary["all_passed"] else "❌ FAILURES DETECTED"
        
        print(f"\n{'='*50}")
        print(f"DATA QUALITY REPORT — {status}")
        print(f"{'='*50}")
        print(f"Total checks: {summary['total']}")
        print(f"Passed: {summary['passed']} | Failed: {summary['failed']}")
        print(f"Success rate: {summary['success_rate']}")
        print(f"{'-'*50}")
        
        for r in self.results:
            icon = "✅" if r["passed"] else "❌"
            print(f"  {icon} {r['check']}: {r['details']}")
        
        print(f"{'='*50}")
```

---

### Using the Custom Checker

```python
# Usage example
checker = CustomDataQualityChecker()

orders_config = [
    {"type": "not_null", "column": "order_id"},
    {"type": "unique", "column": "order_id"},
    {"type": "not_null", "column": "customer_id"},
    {"type": "range", "column": "total_amount", "min": 0.01, "max": 10000},
    {"type": "accepted_values", "column": "status", 
     "values": ["pending", "preparing", "on_the_way", "delivered", "cancelled"]},
    {"type": "row_count", "min": 1, "max": 100000},
    {"type": "no_outliers", "column": "total_amount", "z_threshold": 3},
]

# Run checks
summary = checker.run_all(df, "orders", orders_config)
checker.print_report()
```

---

### Afternoon Exercises

**Exercise 4:** Define data quality rules for 3 MakanExpress tables (orders, restaurants, customers) and run the custom checker on each. Create a combined report.

<details>
<summary>🔑 Answer</summary>

```python
# Define rules for each table
orders_rules = [
    {"type": "not_null", "column": "order_id"},
    {"type": "unique", "column": "order_id"},
    {"type": "not_null", "column": "total_amount"},
    {"type": "range", "column": "total_amount", "min": 0.01, "max": 10000},
    {"type": "accepted_values", "column": "status", 
     "values": ["pending", "preparing", "on_the_way", "delivered", "cancelled"]},
    {"type": "row_count", "min": 1, "max": 100000},
]

restaurants_rules = [
    {"type": "not_null", "column": "id"},
    {"type": "unique", "column": "id"},
    {"type": "not_null", "column": "name"},
    {"type": "range", "column": "rating", "min": 0, "max": 5},
    {"type": "accepted_values", "column": "city", 
     "values": ["Singapore", "Kuala Lumpur", "Penang", "Johor Bahru"]},
]

customers_rules = [
    {"type": "not_null", "column": "id"},
    {"type": "unique", "column": "id"},
    {"type": "not_null", "column": "name"},
    {"type": "not_null", "column": "email"},
    {"type": "accepted_values", "column": "city",
     "values": ["Singapore", "Kuala Lumpur", "Penang", "Johor Bahru"]},
]

# Run all checks
checker = CustomDataQualityChecker()
checker.run_all(orders_df, "orders", orders_rules)
checker.run_all(restaurants_df, "restaurants", restaurants_rules)
checker.run_all(customers_df, "customers", customers_rules)
checker.print_report()
```

</details>

**Exercise 5:** Write a function that runs data quality checks as part of an Airflow pipeline. If any check fails, send an alert (just print for now) and don't proceed to the load step.

<details>
<summary>🔑 Answer</summary>

```python
def data_quality_gate(**context):
    """Airflow task: run DQ checks. Fail the pipeline if checks don't pass."""
    ti = context["ti"]
    
    # Pull data from previous task
    orders_df = ti.xcom_pull(key="orders_df", task_ids="extract")
    
    checker = CustomDataQualityChecker()
    rules = [
        {"type": "not_null", "column": "order_id"},
        {"type": "unique", "column": "order_id"},
        {"type": "range", "column": "total_amount", "min": 0.01, "max": 10000},
        {"type": "row_count", "min": 1, "max": 100000},
    ]
    
    summary = checker.run_all(orders_df, "orders", rules)
    checker.print_report()
    
    if not summary["all_passed"]:
        failed_checks = [r for r in checker.results if not r["passed"]]
        error_msg = f"Data quality gate FAILED: {len(failed_checks)} checks failed"
        logger.error(error_msg)
        for check in failed_checks:
            logger.error(f"  ❌ {check['check']}: {check['details']}")
        raise ValueError(error_msg)  # This fails the Airflow task
    
    logger.info("✅ Data quality gate passed — proceeding to load")
    return summary

# In your DAG:
# extract >> data_quality_gate >> load
# If quality_gate raises exception, load won't run
```

</details>

---

## 🌙 Evening (1 hour): Quick Reference + Practice

### 🔖 Data Quality Quick Reference

```
┌─────────────────────────────────────────────────────────┐
│          DATA QUALITY QUICK REFERENCE                    │
├─────────────────────────────────────────────────────────┤
│ CHECK TYPES                                              │
│ • Not Null     — required fields have values             │
│ • Unique       — no duplicate records                    │
│ • Range        — values within bounds                    │
│ • Accepted     — categorical values valid                │
│ • Referential  — foreign keys exist                      │
│ • Freshness    — data is recent enough                   │
│ • Volume       — row count in range                      │
│ • Schema       — column types/names unchanged            │
│ • Outliers     — no statistical anomalies                │
│                                                          │
│ TOOLS                                                    │
│ Great Expectations — Python framework, rich reporting    │
│ dbt tests          — built into dbt, YAML + SQL          │
│ Custom Python      — lightweight, full control           │
│ Custom SQL         — database-level checks                │
│                                                          │
│ WHEN TO CHECK                                            │
│ ✅ After extract (raw data validation)                    │
│ ✅ After transform (cleaned data validation)              │
│ ✅ Before load (gate: fail if quality is bad)             │
│ ✅ Periodically (monitor existing data health)            │
│                                                          │
│ PATTERN: Quality Gate                                    │
│ extract → transform → [DQ CHECK] → load                  │
│                        ↑ fails → alert + stop             │
│                                                          │
│ INTERVIEW TALKING POINT                                  │
│ "I implemented a data quality gate in our Airflow DAG.   │
│ Before loading, we run 12 automated checks covering      │
│ null values, range violations, freshness, and volume.    │
│ If any check fails, the pipeline stops and alerts the    │
│ team on Slack. This caught a source API bug that would   │
│ have loaded 10K zero-amount orders."                     │
└─────────────────────────────────────────────────────────┘
```

### 📝 Today's Checklist

- [ ] Understand why data quality is critical for data engineers
- [ ] Know the 10 common data quality check types
- [ ] Used Great Expectations to validate a DataFrame
- [ ] Written custom SQL data quality checks
- [ ] Built a reusable data quality checker class
- [ ] Defined quality rules for MakanExpress tables
- [ ] Understand how to integrate DQ checks into Airflow
- [ ] Completed at least 4 of 5 exercises
- [ ] Ready for tomorrow: Testing patterns deep dive

---

*Day 64 complete! Tomorrow: Testing patterns — schema validation, distribution checks, freshness monitoring.* 🛡️
