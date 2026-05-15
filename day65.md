# 📅 Day 65 — Friday, 18 July 2026
# 🧪 Testing Patterns: Schema, Null, Distribution, Freshness

---

## 🎯 Today's Goal

Yesterday you learned the concepts. Today you go deep into **testing patterns** — reusable templates for common data quality checks that you can apply to ANY dataset. Think of these as your data quality "recipes."

---

## ☀️ Morning Block (2 hours): Schema + Null + Type Checks

### Pattern 1: Schema Validation

Ensure your data hasn't changed structure (columns added/removed/renamed).

```python
# src/data_quality/schema_check.py

import pandas as pd
import logging

logger = logging.getLogger(__name__)

class SchemaValidator:
    """Validate DataFrame schema against expected schema."""
    
    def __init__(self, expected_schema):
        """
        expected_schema: dict of {column_name: expected_type}
        Types: 'int', 'float', 'str', 'datetime', 'bool'
        """
        self.expected = expected_schema
        self.issues = []
    
    def validate(self, df):
        """Validate DataFrame against expected schema."""
        self.issues = []
        
        # Check 1: Missing columns
        expected_cols = set(self.expected.keys())
        actual_cols = set(df.columns)
        
        missing = expected_cols - actual_cols
        if missing:
            self.issues.append(f"Missing columns: {missing}")
        
        extra = actual_cols - expected_cols
        if extra:
            self.issues.append(f"Unexpected columns: {extra}")
        
        # Check 2: Data types
        for col, expected_type in self.expected.items():
            if col not in df.columns:
                continue
            
            actual_type = str(df[col].dtype)
            type_match = self._check_type(actual_type, expected_type)
            
            if not type_match:
                self.issues.append(
                    f"Column '{col}': expected {expected_type}, got {actual_type}"
                )
        
        # Check 3: Column order (warning only)
        common = [c for c in self.expected if c in df.columns]
        actual_order = [c for c in df.columns if c in common]
        if common != actual_order:
            self.issues.append(f"Column order changed (warning)")
        
        passed = len(self.issues) == 0
        return {"passed": passed, "issues": self.issues}
    
    def _check_type(self, actual, expected):
        type_map = {
            "int": ["int64", "int32", "Int64"],
            "float": ["float64", "float32"],
            "str": ["object", "string"],
            "datetime": ["datetime64[ns]", "datetime64"],
            "bool": ["bool", "boolean"],
        }
        accepted = type_map.get(expected, [expected])
        return any(a in actual for a in accepted)

# Usage
orders_schema = {
    "order_id": "int",
    "customer_id": "int",
    "restaurant_id": "int",
    "total_amount": "float",
    "status": "str",
    "order_date": "datetime",
    "delivery_city": "str"
}

validator = SchemaValidator(orders_schema)
result = validator.validate(orders_df)
if result["passed"]:
    print("✅ Schema valid")
else:
    print("❌ Schema issues:")
    for issue in result["issues"]:
        print(f"  - {issue}")
```

### Pattern 2: Null Analysis

```python
class NullAnalyzer:
    """Analyze null patterns in data."""
    
    def __init__(self, required_columns, optional_columns=None):
        self.required = required_columns
        self.optional = optional_columns or []
    
    def analyze(self, df):
        """Analyze null patterns and return report."""
        total_rows = len(df)
        report = {"total_rows": total_rows, "columns": [], "passed": True}
        
        for col in self.required + self.optional:
            null_count = df[col].isnull().sum()
            null_pct = (null_count / total_rows * 100) if total_rows > 0 else 0
            
            is_required = col in self.required
            passed = True if not is_required else null_count == 0
            
            if is_required and null_count > 0:
                report["passed"] = False
            
            report["columns"].append({
                "column": col,
                "null_count": int(null_count),
                "null_pct": round(null_pct, 2),
                "is_required": is_required,
                "passed": passed
            })
        
        return report
    
    def print_report(self, report):
        print(f"\n📊 Null Analysis — {report['total_rows']} rows")
        print("-" * 60)
        print(f"{'Column':<20} {'Nulls':<10} {'%':<10} {'Required':<10} {'Status'}")
        print("-" * 60)
        
        for col in report["columns"]:
            status = "✅" if col["passed"] else "❌"
            req = "Yes" if col["is_required"] else "No"
            print(f"{col['column']:<20} {col['null_count']:<10} "
                  f"{col['null_pct']:<10.1f} {req:<10} {status}")

# Usage
analyzer = NullAnalyzer(
    required_columns=["order_id", "customer_id", "total_amount", "status"],
    optional_columns=["delivery_city", "rating"]
)
report = analyzer.analyze(orders_df)
analyzer.print_report(report)
```

### Pattern 3: Type Coercion Check

```python
def check_type_coercion(df, column, target_type):
    """Check how many values would fail type conversion."""
    if target_type == "numeric":
        coerced = pd.to_numeric(df[column], errors="coerce")
    elif target_type == "datetime":
        coerced = pd.to_datetime(df[column], errors="coerce")
    else:
        return {"passed": True, "failures": 0}
    
    failures = coerced.isnull() & df[column].notna()
    fail_count = failures.sum()
    
    return {
        "passed": fail_count == 0,
        "failures": int(fail_count),
        "failure_rate": round(fail_count / len(df) * 100, 2) if len(df) > 0 else 0,
        "sample_failures": df[failures][column].head(5).tolist() if fail_count > 0 else []
    }
```

### Morning Exercises

**Exercise 1:** Write a `SchemaMigrationDetector` that compares today's data schema against yesterday's (stored as a JSON file). Report any columns added, removed, or with changed types.

<details>
<summary>🔑 Answer</summary>

```python
import json
import pandas as pd

class SchemaMigrationDetector:
    """Detect schema changes between runs."""
    
    def save_schema(self, df, filepath):
        """Save current schema to JSON."""
        schema = {}
        for col in df.columns:
            schema[col] = str(df[col].dtype)
        with open(filepath, "w") as f:
            json.dump(schema, f, indent=2)
    
    def detect_changes(self, df, previous_schema_path):
        """Compare current schema against saved schema."""
        current = {col: str(df[col].dtype) for col in df.columns}
        
        try:
            with open(previous_schema_path) as f:
                previous = json.load(f)
        except FileNotFoundError:
            return {"status": "no_previous", "message": "No previous schema to compare"}
        
        changes = {
            "added_columns": set(current.keys()) - set(previous.keys()),
            "removed_columns": set(previous.keys()) - set(current.keys()),
            "type_changes": {}
        }
        
        for col in set(current.keys()) & set(previous.keys()):
            if current[col] != previous[col]:
                changes["type_changes"][col] = {
                    "was": previous[col],
                    "now": current[col]
                }
        
        has_changes = bool(changes["added_columns"] or changes["removed_columns"] or changes["type_changes"])
        changes["has_changes"] = has_changes
        
        return changes

# Usage
detector = SchemaMigrationDetector()
detector.save_schema(orders_df, "schema_orders.json")
# Next run:
changes = detector.detect_changes(orders_df, "schema_orders.json")
if changes.get("has_changes"):
    print("⚠️ Schema changes detected!")
    print(json.dumps(changes, indent=2, default=str))
```

</details>

**Exercise 2:** Write a completeness check that, for each column, calculates the fill rate (non-null / total) and flags any column below a configurable threshold (e.g., 95%).

<details>
<summary>🔑 Answer</summary>

```python
def check_completeness(df, min_fill_rate=0.95):
    """Check data completeness (fill rate) for all columns."""
    total = len(df)
    results = []
    
    for col in df.columns:
        non_null = df[col].notna().sum()
        fill_rate = non_null / total if total > 0 else 0
        passed = fill_rate >= min_fill_rate
        
        results.append({
            "column": col,
            "fill_rate": round(fill_rate, 4),
            "fill_pct": f"{fill_rate*100:.1f}%",
            "null_count": total - non_null,
            "passed": passed
        })
    
    print(f"📊 Completeness Check (threshold: {min_fill_rate*100}%)")
    print(f"{'Column':<25} {'Fill Rate':<12} {'Nulls':<8} {'Status'}")
    print("-" * 55)
    
    for r in results:
        status = "✅" if r["passed"] else "❌"
        print(f"{r['column']:<25} {r['fill_pct']:<12} {r['null_count']:<8} {status}")
    
    all_passed = all(r["passed"] for r in results)
    return {"passed": all_passed, "columns": results}

check_completeness(orders_df, min_fill_rate=0.95)
```

</details>

**Exercise 3:** Write a function that profiles a DataFrame — for each column, report: dtype, null count, unique values, min/max/mean (numeric), top 5 values (categorical).

<details>
<summary>🔑 Answer</summary>

```python
def profile_dataframe(df):
    """Generate a data profile for all columns."""
    print(f"📊 Data Profile — {len(df)} rows, {len(df.columns)} columns")
    print("=" * 70)
    
    for col in df.columns:
        print(f"\n  Column: {col}")
        print(f"  Type: {df[col].dtype}")
        print(f"  Nulls: {df[col].isnull().sum()} ({df[col].isnull().mean()*100:.1f}%)")
        print(f"  Unique: {df[col].nunique()}")
        
        if pd.api.types.is_numeric_dtype(df[col]):
            print(f"  Min: {df[col].min()}")
            print(f"  Max: {df[col].max()}")
            print(f"  Mean: {df[col].mean():.2f}")
            print(f"  Median: {df[col].median():.2f}")
        else:
            top5 = df[col].value_counts().head(5)
            print(f"  Top values:")
            for val, count in top5.items():
                print(f"    {val}: {count} ({count/len(df)*100:.1f}%)")
    
    print("\n" + "=" * 70)

profile_dataframe(orders_df)
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Distribution + Freshness + Monitoring

### Pattern 4: Distribution Checks

```python
class DistributionChecker:
    """Check if data distribution is within expected bounds."""
    
    def __init__(self, baseline_stats):
        """
        baseline_stats: dict with historical stats
        {"total_amount": {"mean": 25.0, "std": 15.0, "min": 1.0, "max": 500.0}}
        """
        self.baseline = baseline_stats
    
    def check_mean_shift(self, df, column, tolerance_pct=0.5):
        """Check if mean has shifted beyond tolerance."""
        if column not in self.baseline:
            return {"passed": True, "message": "No baseline"}
        
        current_mean = df[column].mean()
        baseline_mean = self.baseline[column]["mean"]
        
        shift = abs(current_mean - baseline_mean) / baseline_mean * 100
        passed = shift <= tolerance_pct * 100
        
        return {
            "passed": passed,
            "current_mean": round(current_mean, 2),
            "baseline_mean": baseline_mean,
            "shift_pct": round(shift, 2),
            "message": f"Mean shifted {shift:.1f}%" if not passed else "Mean stable"
        }
    
    def check_volume_change(self, current_count, expected_range):
        """Check if row count is within expected range."""
        min_count, max_count = expected_range
        passed = min_count <= current_count <= max_count
        
        return {
            "passed": passed,
            "current_count": current_count,
            "expected_range": expected_range,
            "message": f"Count {current_count} outside {expected_range}" if not passed else "Volume OK"
        }
    
    def check_zero_inflation(self, df, column, max_zero_pct=5):
        """Check if too many values are zero (common data issue)."""
        zero_pct = (df[column] == 0).sum() / len(df) * 100
        passed = zero_pct <= max_zero_pct
        
        return {
            "passed": passed,
            "zero_count": int((df[column] == 0).sum()),
            "zero_pct": round(zero_pct, 2),
            "threshold": max_zero_pct
        }
```

### Pattern 5: Freshness Monitor

```python
import psycopg2
from datetime import datetime, timedelta

class FreshnessMonitor:
    """Monitor data freshness across tables."""
    
    def __init__(self, conn_params):
        self.conn_params = conn_params
    
    def check_all_tables(self):
        """Check freshness for all tracked tables."""
        tables = [
            {"table": "orders", "column": "order_date", "max_age_hours": 24},
            {"table": "restaurants", "column": "updated_at", "max_age_hours": 168},  # 1 week
            {"table": "customers", "column": "signup_date", "max_age_hours": 48},
        ]
        
        results = []
        for t in tables:
            result = self._check_table(t["table"], t["column"], t["max_age_hours"])
            results.append(result)
        
        self._print_report(results)
        return results
    
    def _check_table(self, table, column, max_age_hours):
        conn = psycopg2.connect(**self.conn_params)
        cur = conn.cursor()
        
        try:
            cur.execute(f"SELECT MAX({column}) FROM {table}")
            latest = cur.fetchone()[0]
            
            if latest is None:
                return {"table": table, "status": "empty", "passed": False}
            
            if isinstance(latest, str):
                latest = datetime.fromisoformat(latest.replace("Z", "+00:00"))
            
            age_hours = (datetime.now(latest.tzinfo) - latest).total_seconds() / 3600
            passed = age_hours <= max_age_hours
            
            return {
                "table": table,
                "column": column,
                "latest_record": str(latest),
                "age_hours": round(age_hours, 1),
                "max_age_hours": max_age_hours,
                "passed": passed
            }
        except Exception as e:
            return {"table": table, "status": "error", "error": str(e), "passed": False}
        finally:
            cur.close()
            conn.close()
    
    def _print_report(self, results):
        print("\n⏱️ Data Freshness Report")
        print("=" * 60)
        print(f"{'Table':<20} {'Latest':<25} {'Age':<10} {'Status'}")
        print("-" * 60)
        
        for r in results:
            if r.get("status") == "error":
                print(f"{r['table']:<20} {'ERROR':<25} {'?':<10} ❌")
            elif r.get("status") == "empty":
                print(f"{r['table']:<20} {'NO DATA':<25} {'?':<10} ❌")
            else:
                status = "✅" if r["passed"] else "❌ STALE"
                print(f"{r['table']:<20} {r['latest_record'][:24]:<25} "
                      f"{r['age_hours']}h{'':<5} {status}")
```

### Pattern 6: Automated Monitoring with Alerting

```python
class DataQualityMonitor:
    """Orchestrate all DQ checks and generate alerts."""
    
    def __init__(self, conn_params):
        self.conn_params = conn_params
        self.checks = []
        self.alerts = []
    
    def add_check(self, name, check_fn, severity="error"):
        self.checks.append({"name": name, "fn": check_fn, "severity": severity})
    
    def run_all(self):
        """Run all registered checks."""
        results = []
        
        for check in self.checks:
            try:
                result = check["fn"]()
                result["name"] = check["name"]
                result["severity"] = check["severity"]
                results.append(result)
                
                if not result.get("passed", True):
                    self.alerts.append({
                        "check": check["name"],
                        "severity": check["severity"],
                        "message": result.get("message", "Check failed"),
                        "timestamp": datetime.now().isoformat()
                    })
            except Exception as e:
                results.append({
                    "name": check["name"],
                    "passed": False,
                    "error": str(e),
                    "severity": check["severity"]
                })
        
        return results
    
    def print_alerts(self):
        if not self.alerts:
            print("✅ No alerts — all checks passed")
            return
        
        print(f"\n🚨 DATA QUALITY ALERTS ({len(self.alerts)})")
        print("=" * 50)
        for alert in self.alerts:
            icon = "🔴" if alert["severity"] == "error" else "🟡"
            print(f"{icon} [{alert['severity'].upper()}] {alert['check']}")
            print(f"   {alert['message']}")

# Usage
monitor = DataQualityMonitor(db_params)
monitor.add_check("orders_freshness", 
    lambda: freshness_check("orders", "order_date", 24))
monitor.add_check("orders_no_null_ids", 
    lambda: not_null_check("orders", "order_id"))
monitor.add_check("orders_volume", 
    lambda: volume_check("orders", 100, 50000))
monitor.add_check("restaurants_freshness", 
    lambda: freshness_check("restaurants", "updated_at", 168), "warning")

results = monitor.run_all()
monitor.print_alerts()
```

---

### Afternoon Exercises

**Exercise 4:** Create a baseline stats file. After running the API pipeline for a few days, calculate mean/std/min/max for `total_amount` and save to JSON. Then write a check that compares today's stats against the baseline.

<details>
<summary>🔑 Answer</summary>

```python
import json

def save_baseline_stats(df, filepath="baseline_stats.json"):
    """Save baseline statistics for numeric columns."""
    stats = {}
    for col in df.select_dtypes(include="number").columns:
        stats[col] = {
            "mean": float(df[col].mean()),
            "std": float(df[col].std()),
            "min": float(df[col].min()),
            "max": float(df[col].max()),
            "median": float(df[col].median()),
            "saved_at": datetime.now().isoformat()
        }
    
    with open(filepath, "w") as f:
        json.dump(stats, f, indent=2)
    print(f"✅ Baseline saved to {filepath}")

def compare_to_baseline(df, filepath="baseline_stats.json", tolerance=0.5):
    """Compare current data stats against baseline."""
    with open(filepath) as f:
        baseline = json.load(f)
    
    print("📊 Baseline Comparison")
    print("-" * 50)
    
    for col, base in baseline.items():
        if col not in df.columns:
            print(f"⚠️ {col}: Column not in current data")
            continue
        
        current_mean = df[col].mean()
        shift_pct = abs(current_mean - base["mean"]) / base["mean"] * 100
        passed = shift_pct <= tolerance * 100
        
        status = "✅" if passed else f"❌ shifted {shift_pct:.1f}%"
        print(f"  {status} {col}: baseline={base['mean']:.2f}, current={current_mean:.2f}")

save_baseline_stats(orders_df)
compare_to_baseline(orders_df)
```

</details>

**Exercise 5:** Build a complete monitoring dashboard (just print to console) that shows: freshness for each table, row counts, null rates, and distribution alerts. Run it as a single function.

<details>
<summary>🔑 Answer</summary>

```python
def run_full_monitoring(conn_params):
    """Run complete DQ monitoring suite."""
    print("📊 MAKANEXPRESS DATA QUALITY DASHBOARD")
    print(f"   Generated: {datetime.now().strftime('%Y-%m-%d %H:%M')}")
    print("=" * 60)
    
    conn = psycopg2.connect(**conn_params)
    
    # Table overview
    tables = ["orders", "restaurants", "customers", "order_items"]
    print(f"\n📋 Table Overview")
    print("-" * 50)
    for table in tables:
        try:
            cur = conn.cursor()
            cur.execute(f"SELECT COUNT(*) FROM {table}")
            count = cur.fetchone()[0]
            print(f"  {table}: {count:,} rows")
        except:
            print(f"  {table}: ⚠️ table not found")
    
    # Freshness
    print(f"\n⏱️ Freshness")
    print("-" * 50)
    freshness_checks = [
        ("orders", "order_date", 24),
        ("restaurants", "updated_at", 168),
    ]
    for table, col, max_h in freshness_checks:
        try:
            cur = conn.cursor()
            cur.execute(f"SELECT MAX({col}) FROM {table}")
            latest = cur.fetchone()[0]
            if latest:
                age_h = (datetime.now() - latest.replace(tzinfo=None)).total_seconds() / 3600
                status = "✅" if age_h <= max_h else f"❌ {age_h:.0f}h old"
                print(f"  {status} {table}.{col} (max: {max_h}h)")
        except:
            pass
    
    # Null rates for orders
    print(f"\n📊 Null Rates (orders)")
    print("-" * 50)
    null_cols = ["order_id", "customer_id", "total_amount", "status"]
    for col in null_cols:
        try:
            cur = conn.cursor()
            cur.execute(f"SELECT COUNT(*) - COUNT({col}) FROM orders")
            nulls = cur.fetchone()[0]
            cur.execute("SELECT COUNT(*) FROM orders")
            total = cur.fetchone()[0]
            null_pct = nulls/total*100 if total > 0 else 0
            status = "✅" if null_pct == 0 else f"❌ {null_pct:.1f}%"
            print(f"  {status} {col}: {nulls} nulls")
        except:
            pass
    
    conn.close()
    print(f"\n{'='*60}")
    print("Dashboard complete.")

run_full_monitoring(db_params)
```

</details>

---

## 🌙 Evening: Quick Reference

### 🔖 Testing Patterns Quick Reference

```
┌─────────────────────────────────────────────────────────┐
│          TESTING PATTERNS QUICK REFERENCE                │
├─────────────────────────────────────────────────────────┤
│ SCHEMA CHECKS                                            │
│ • Column exists?     df[col].any()                       │
│ • Type correct?      df[col].dtype == expected           │
│ • Columns added/removed? Compare against baseline        │
│                                                          │
│ NULL CHECKS                                              │
│ • No nulls:          df[col].isnull().sum() == 0         │
│ • Fill rate:         df[col].notna().mean() >= 0.95      │
│ • Required vs optional columns                           │
│                                                          │
│ RANGE CHECKS                                             │
│ • Numeric bounds:    df[col].between(min, max).all()     │
│ • Date bounds:       dates within [start, end]           │
│ • Zero inflation:    (df[col]==0).mean() < 0.05          │
│                                                          │
│ DISTRIBUTION CHECKS                                      │
│ • Mean shift:        abs(mean-baseline)/baseline < 50%   │
│ • Volume change:     count within [min, max]             │
│ • Outliers:          z-score > 3                         │
│                                                          │
│ FRESHNESS CHECKS                                         │
│ • Latest record age < threshold                          │
│ • Row count today > 0                                    │
│ • No future dates                                        │
│                                                          │
│ MONITORING                                               │
│ • Save baseline stats after each run                     │
│ • Compare current vs baseline                            │
│ • Alert on failures (Slack/email/print)                  │
│ • Dashboard: freshness + nulls + volume + distribution   │
└─────────────────────────────────────────────────────────┘
```

### 📝 Today's Checklist

- [ ] Built schema validation checker
- [ ] Built null analysis tool
- [ ] Built distribution checker with baseline comparison
- [ ] Built freshness monitor
- [ ] Built complete DQ monitoring dashboard
- [ ] Completed at least 4 of 5 exercises
- [ ] Ready for tomorrow: Add DQ to all existing projects

---

*Day 65 complete! Tomorrow: Add data quality to all your portfolio projects.* 🧪
