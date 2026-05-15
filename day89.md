# 📅 Day 89 — Monday, 11 August 2026
# 🏗️ System Design Practice: 5 Common DE Scenarios

---

## 🎯 Today's Goal

5 full system design walkthroughs for the most common DE interview scenarios. Each includes: requirements clarification, architecture, tech choices, tradeoffs, and discussion points.

---

## ☀️ Morning Block (2 hours): Scenarios 1-2

### Scenario 1: Design a Batch ETL Pipeline

**Prompt:** "Design a daily ETL pipeline for MakanExpress that processes 500K food delivery orders."

**Clarifying Questions:**
- Source: PostgreSQL orders DB + Stripe payments API
- Volume: 500K orders/day, ~50MB
- Schedule: Daily by 6 AM SGT
- Consumers: 5 analysts, weekly reports
- Budget: Cost-effective (startup)

**Architecture:**
```
┌───────────┐     ┌──────────┐     ┌─────────────┐     ┌──────────────┐
│ Orders DB │────→│          │     │ S3 Data Lake│     │              │
│ (Postgres)│     │ Airflow  │────→│ raw/        │────→│ dbt          │
│           │     │ (2 AM    │     │ staging/    │     │ (Transform)  │
│ Stripe API│────→│  daily)  │     │ marts/      │     │              │
│           │     │          │     │             │     └──────┬───────┘
│ Restaurant│     │ Python   │     │ Glue Crawler│            │
│ API       │────→│ extract  │     │ (Schema)    │     ┌──────▼───────┐
└───────────┘     └──────────┘     └─────────────┘     │ Athena/      │
                                                         │ Redshift     │
                                                         │ (Query)      │
                                                         └──────────────┘
```

**Tech Choices:**
| Component | Choice | Why |
|-----------|--------|-----|
| Orchestration | Airflow | Dependency management, retry, monitoring |
| Storage | S3 + Parquet | Cheap, scalable, columnar for analytics |
| Transform | dbt | Version-controlled SQL, tested, documented |
| Query | Athena (start) → Redshift (scale) | Pay-per-query initially |
| Monitoring | Airflow alerts + Slack | Fail-fast notification |

**Tradeoffs:**
- Batch vs Streaming: Batch is simpler and cheaper. Reports are weekly = batch is fine.
- Athena vs Redshift: Athena is $5/TB scanned, Redshift is $180+/month. Start with Athena.
- dbt vs custom SQL: dbt adds structure, testing, lineage. Worth the learning curve.

**Scaling (if volume 10x):**
- Partition S3 by date
- Move to Redshift for faster queries
- Add Glue for heavy transforms
- Consider streaming (Kinesis) for real-time needs

---

### Scenario 2: Design a Real-Time Dashboard

**Prompt:** "Design a dashboard that shows live order count and revenue for MakanExpress."

**Architecture:**
```
┌───────────┐     ┌──────────┐     ┌──────────────┐     ┌───────────┐
│ Orders    │────→│ Kinesis  │────→│ Lambda       │────→│ DynamoDB  │
│ Stream    │     │ Data     │     │ (Aggregate)  │     │ (Current  │
│           │     │ Streams  │     │              │     │  State)   │
│ Driver    │────→│          │     │ Also → S3    │     │           │
│ Updates   │     │          │     │ (Raw Archive)│     │ API       │
│           │     │          │     │              │     │ Gateway   │
│ Restaurant│────→│          │     │              │     │     ↓     │
│ Events    │     │          │     │              │     │ Dashboard │
└───────────┘     └──────────┘     └──────────────┘     └───────────┘
```

**Key Points:**
- Lambda architecture: real-time layer + batch layer
- DynamoDB for sub-second reads (current order count, revenue)
- S3 archive for batch reprocessing
- At fresh grad level: "I'd implement the batch layer first, then add real-time"

---

## 🌤️ Afternoon Block (2 hours): Scenarios 3-5

### Scenario 3: Design a Data Warehouse

**Prompt:** "Design a data warehouse for Shopee Singapore — millions of orders, products, sellers."

**Star Schema:**
```
fact_orders: order_id, date_key, customer_key, product_key, seller_key,
             quantity, amount, discount, status

dim_customer: customer_key, id, name, segment, city, country, signup_date
dim_product: product_key, id, name, category, subcategory, price
dim_seller: seller_key, id, shop_name, rating, location
dim_date: date_key, date, day, month, quarter, year, is_holiday
```

**Tech Stack:**
- Storage: S3 (raw) + Redshift/Snowflake (warehouse)
- Transform: dbt (SQL transforms, incremental models)
- Orchestration: Airflow
- BI: Metabase (free) or QuickSight

**Discussion Points:**
- Snowflake vs Redshift: Snowflake separates compute/storage, auto-scales. Redshift is cheaper for steady workloads.
- SCD Type 2 for dimensions that change (seller ratings, product prices)
- Partition fact table by date for query performance

### Scenario 4: Design a Data Quality Framework

**Prompt:** "Design a data quality system that catches issues before they reach dashboards."

**Architecture:**
```
Data arrives → Schema validation → Row-level checks → Aggregate checks
                    ↓                     ↓                    ↓
              Alert if schema       Alert if >5% nulls    Alert if count
              changed               or values out of      drops >20%
                                    range

Tools: Great Expectations (Python) + dbt tests + custom SQL checks
```

**Types of Checks:**
| Check Type | Example | Tool |
|-----------|---------|------|
| Schema | Column exists, correct type | Great Expectations |
| Completeness | Null % < threshold | dbt test |
| Uniqueness | No duplicate order IDs | dbt test |
| Range | Order amount between 0-10000 | Custom SQL |
| Freshness | Data updated within 24h | Airflow sensor |
| Referential | All customer_ids exist in dim | dbt relationship test |
| Distribution | Order count within 2 std devs | Custom Python |

### Scenario 5: Design an Event-Driven Pipeline

**Prompt:** "Design a pipeline that processes events (order placed → confirmed → prepared → delivered)."

**Architecture:**
```
App → EventBridge → Lambda → S3 (raw events)
                              ↓
                    Glue (batch processing)
                              ↓
                    S3 (processed) → Athena
                              ↓
                    SNS/Slack (alerts for anomalies)
```

**Key Concepts:**
- Events are immutable (never modify, only append)
- At-least-once delivery (handle duplicates with idempotent processing)
- Event ordering: use sequence numbers or timestamps
- Dead letter queue for failed events

---

## 🌙 Evening (1 hour): Cheat Sheet + Practice Plan

### 🗺️ System Design Cheat Sheet

```
FRAMEWORK (4 steps):
1. Clarify requirements (5 min)
2. High-level architecture (10 min)
3. Deep dive 2-3 components (15 min)
4. Tradeoffs + scaling (10 min)

COMMON COMPONENTS:
  Ingestion: Airflow, Lambda, Kinesis, EventBridge
  Storage: S3, RDS, Redshift, Snowflake, DynamoDB
  Transform: dbt, Glue, PySpark, Python
  Query: Athena, Redshift, Snowflake
  Monitoring: CloudWatch, Slack alerts, Great Expectations

COMMON TRADEOFFS:
  Batch vs Streaming: simpler vs real-time
  SQL vs PySpark: simple transforms vs complex/big data
  Managed vs Self-hosted: cost vs control
  Denormalized vs Normalized: query speed vs storage

SCALING ANSWERS:
  10x volume → partition, columnar format, bigger warehouse
  100x volume → Spark/Databricks, streaming, micro-batch
  More users → caching, read replicas, materialized views
```

### 📝 Today's Checklist

- [ ] Walked through all 5 scenarios
- [ ] Can draw architecture for batch ETL pipeline
- [ ] Can discuss real-time vs batch tradeoffs
- [ ] Can design a star schema for common business domains
- [ ] Can explain data quality check types
- [ ] Saved system design cheat sheet

---

*Day 89 complete! Tomorrow: Full mock interview.* 🎯
