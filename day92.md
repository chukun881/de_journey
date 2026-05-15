# 📅 Day 92 — Thursday, 14 August 2026
# 🏗️ System Design Deep Dive: Data Warehouse Architecture

---

## 🎯 Today's Goal

Two advanced system design walkthroughs focused on data warehouse architecture — the most common DE interview topic after SQL. Today covers batch vs streaming tradeoffs, warehouse design patterns, and how to discuss architecture choices in interviews.

**Philosophy:** In system design interviews, you're not expected to know everything. You're expected to THINK clearly, ask good questions, and make reasonable tradeoffs. Today you practice exactly that.

---

## ☀️ Morning Block (2 hours): Batch vs Streaming + Warehouse Patterns

### Batch vs Streaming: The Core Tradeoff

```
BATCH (what you've built):
┌──────────┐    ┌──────────┐    ┌──────────┐
│  Source   │───→│  Airflow │───→│  Data    │
│  DB/API   │    │  (daily) │    │  Warehouse│
└──────────┘    └──────────┘    └──────────┘
- Latency: hours to next day
- Cost: low (compute only when running)
- Complexity: medium
- Best for: reports, analytics, ML training

STREAMING (what interviewers ask about):
┌──────────┐    ┌──────────┐    ┌──────────┐
│  Events  │───→│  Kafka/  │───→│  Real-time│
│  (clicks, │    │  Kinesis │    │  Dashboard│
│   orders) │    │          │    │           │
└──────────┘    └──────────┘    └──────────┘
- Latency: seconds to minutes
- Cost: high (always-on compute)
- Complexity: high
- Best for: fraud detection, live dashboards, alerts
```

### When Interviewers Ask: "Batch or Streaming?"

**Always start with:** "It depends on the business requirement. Can you tell me how fresh the data needs to be?"

| Business Need | Answer |
|---------------|--------|
| "Daily revenue report by 8 AM" | Batch (Airflow + dbt, runs at 2 AM) |
| "Detect fraudulent transactions immediately" | Streaming (Kinesis + Lambda, real-time) |
| "Customer 360 dashboard" | Batch for most data, streaming for recent activity |
| "Product recommendations" | Batch (retrain model daily, serve from cache) |

### Common Warehouse Architecture Patterns

**Pattern 1: Traditional Data Warehouse**
```
Source DBs → ETL (Airflow) → Data Warehouse (Redshift/Snowflake) → BI Tool
```
- Simple, proven, well-understood
- Single source of truth
- Limitation: hard to handle unstructured data

**Pattern 2: Data Lake + Warehouse (what you built)**
```
Sources → S3 Data Lake → Glue/dbt Transform → Warehouse (RDS/Athena) → Analytics
```
- Handles structured AND unstructured data
- Cost-effective (S3 cheap, Athena pay-per-query)
- Decoupled storage from compute
- Most common pattern in SG/MY startups

**Pattern 3: Lakehouse (modern, 2025+)**
```
Sources → Data Lake (Delta Lake/Iceberg) → Unified Query Engine → Analytics + ML
```
- Combines lake flexibility with warehouse performance
- ACID transactions on lake data
- Databricks, Apache Iceberg, Delta Lake
- Good to mention in interviews: "I'm aware of the lakehouse trend"

### Scenario Walkthrough: "Design a data warehouse for a food delivery company"

**Step 1: Clarify**
- "How many orders per day?" → 500K across SG/MY
- "What analyses?" → Revenue, restaurant performance, customer behavior, promotions
- "Who uses it?" → 10 analysts + 5 data scientists
- "Budget?" → Cost-effective (Series B startup)

**Step 2: High-Level Design**
```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Orders DB│───→│          │───→│ S3 Data  │───→│ dbt      │───→│ Athena / │
│ (PG)     │    │ Airflow  │    │ Lake     │    │ Transform│    │ Redshift │
│          │    │          │    │ raw/     │    │ staging →│    │          │
│ Payments │───→│ Extract  │───→│ processed│───→│ marts    │───→│ Analysts │
│ (API)    │    │ Load     │    │ analytics│    │          │    │          │
│          │    │          │    │          │    │          │    │ QuickSight│
│ Restaurant│──→│          │───→│          │    │          │───→│ Dashboard│
│ (API)    │    │          │    │          │    │          │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

**Step 3: Key Decisions**

| Decision | Choice | Why |
|----------|--------|-----|
| Storage | S3 + Parquet | $23/TB/month, compressed, columnar |
| Transform | dbt | Version-controlled SQL, tested, documented |
| Query engine | Athena (MVP) → Redshift (scale) | $5/TB scanned vs $180+/month |
| Orchestration | Airflow on ECS | Managed, handles dependencies |
| Schema | Star schema | Optimized for analytical queries |

**Step 4: Scaling Discussion**

"If data volume grows 10x (5M orders/day):"
- Switch from Athena to Redshift/Snowflake for concurrent query performance
- Add partitioning by country + date (already in S3 structure)
- Implement incremental dbt models (already done)
- Add data quality monitoring (Great Expectations)
- Consider streaming layer (Kinesis) for real-time order tracking

### Exercise 1: Walk Through This Scenario

Spend 10 minutes explaining this architecture out loud. Use the 4-step framework:
1. Clarify requirements
2. Propose architecture
3. Explain key decisions
4. Discuss scaling

<details>
<summary>🔑 What to emphasize when explaining</summary>

1. **Start with business context:** "MakanExpress needs daily analytics for 500K orders..."
2. **Draw the flow:** Source → Ingest → Store → Transform → Serve
3. **Explain each tool choice with a WHY:** "Athena because pay-per-query fits a startup budget"
4. **Acknowledge tradeoffs:** "Athena is slower than Redshift for complex queries, but we can migrate later"
5. **Show awareness of growth:** "If we need real-time, we'd add Kinesis + Lambda layer"

The key: sound like you've BUILT this (because you have — MakanExpress!)
</details>

---

## 🌤️ Afternoon Block (2 hours): Schema Design Practice

### Scenario 2: "Design a schema for Shopee SG order analytics"

**Requirements:**
- Track orders, products, sellers, customers, payments, logistics
- Need: daily revenue reports, seller performance, customer segmentation
- Historical tracking needed (seller changes shop name/category)

**Walkthrough:**

```sql
-- Star Schema Design

-- Fact Table: fact_orders
CREATE TABLE fact_orders (
    order_key        BIGINT PRIMARY KEY,
    date_key         INT REFERENCES dim_date(date_key),
    customer_key     INT REFERENCES dim_customer(customer_key),
    product_key      INT REFERENCES dim_product(product_key),
    seller_key       INT REFERENCES dim_seller(seller_key),
    payment_key      INT REFERENCES dim_payment(payment_key),
    quantity         INT,
    unit_price       DECIMAL(10,2),
    discount_amount  DECIMAL(10,2),
    total_amount     DECIMAL(10,2),
    delivery_fee     DECIMAL(10,2),
    order_status     VARCHAR(20)
);

-- Dimension: dim_seller (with SCD Type 2)
CREATE TABLE dim_seller (
    seller_key       SERIAL PRIMARY KEY,
    seller_id        INT NOT NULL,        -- natural key from source
    shop_name        VARCHAR(200),
    category         VARCHAR(100),
    city             VARCHAR(100),
    rating           DECIMAL(3,2),
    is_active        BOOLEAN DEFAULT true,
    effective_from   DATE NOT NULL,
    effective_to     DATE DEFAULT '9999-12-31',
    is_current       BOOLEAN DEFAULT true
);

-- Dimension: dim_customer
CREATE TABLE dim_customer (
    customer_key     SERIAL PRIMARY KEY,
    customer_id      INT NOT NULL,
    name             VARCHAR(200),
    email            VARCHAR(200),
    city             VARCHAR(100),
    segment          VARCHAR(20),  -- 'new', 'regular', 'vip', 'churned'
    signup_date      DATE,
    is_current       BOOLEAN DEFAULT true
);

-- Dimension: dim_date
CREATE TABLE dim_date (
    date_key         INT PRIMARY KEY,  -- 20260814
    full_date        DATE NOT NULL,
    day_of_week      INT,
    day_name         VARCHAR(10),
    month            INT,
    month_name       VARCHAR(10),
    quarter          INT,
    year             INT,
    is_weekend       BOOLEAN,
    is_holiday       BOOLEAN DEFAULT false,
    holiday_name     VARCHAR(100)
);

-- Dimension: dim_product (with SCD Type 2 for price changes)
CREATE TABLE dim_product (
    product_key      SERIAL PRIMARY KEY,
    product_id       INT NOT NULL,
    product_name     VARCHAR(500),
    category         VARCHAR(100),
    subcategory      VARCHAR(100),
    price            DECIMAL(10,2),
    effective_from   DATE NOT NULL,
    effective_to     DATE DEFAULT '9999-12-31',
    is_current       BOOLEAN DEFAULT true
);

-- Dimension: dim_payment
CREATE TABLE dim_payment (
    payment_key      SERIAL PRIMARY KEY,
    payment_id       INT NOT NULL,
    method           VARCHAR(20),  -- 'credit_card', 'grabpay', 'cash', 'shopee_pay'
    gateway          VARCHAR(50),
    status           VARCHAR(20)   -- 'success', 'failed', 'refunded'
);
```

### Why This Schema Works for Interviews

1. **Star schema** → analysts can query any dimension easily
2. **SCD Type 2 on seller + product** → tracks history ("did sellers change category?")
3. **Separate dim_date** → enables time-based analysis (weekday vs weekend, holiday vs normal)
4. **Payment dimension** → enables payment method analysis
5. **Surrogate keys** → best practice, protects against source system changes

### Common Follow-Up Questions

**"How would you handle returns/refunds?"**
```
Add a fact_returns table with negative amounts, linked to the original order.
Or add a credit_debit column to fact_orders: 'C' for credit (refund), 'D' for debit (order).
```

**"How do you populate dim_date?"**
```sql
-- Generate 5 years of dates
INSERT INTO dim_date (date_key, full_date, day_of_week, day_name, month, month_name, quarter, year, is_weekend)
SELECT 
    TO_CHAR(d::date, 'YYYYMMDD')::INT as date_key,
    d::date as full_date,
    EXTRACT(ISODOW FROM d::date)::INT as day_of_week,
    TO_CHAR(d::date, 'Day') as day_name,
    EXTRACT(MONTH FROM d::date)::INT as month,
    TO_CHAR(d::date, 'Month') as month_name,
    EXTRACT(QUARTER FROM d::date)::INT as quarter,
    EXTRACT(YEAR FROM d::date)::INT as year,
    EXTRACT(ISODOW FROM d::date)::INT IN (6, 7) as is_weekend
FROM generate_series('2024-01-01', '2029-12-31', '1 day'::interval) d;
```

**"What if we need to track customer location changes?"**
```
Add SCD Type 2 to dim_customer with effective_from/to. 
Or create a separate dim_customer_location if location changes frequently.
```

### Exercise 2: Design a Schema

Design a star schema for **Grab ride analytics**:
- Track rides (pickup, dropoff, fare, distance, duration)
- Dimensions: driver, rider, time, location (zone), vehicle type
- Need to analyze: driver earnings, peak hours, zone demand, rider retention

<details>
<summary>🔑 Sample Answer</summary>

```sql
-- Fact: fact_rides
CREATE TABLE fact_rides (
    ride_key         BIGINT PRIMARY KEY,
    date_key         INT REFERENCES dim_date(date_key),
    driver_key       INT REFERENCES dim_driver(driver_key),
    rider_key        INT REFERENCES dim_rider(rider_key),
    pickup_zone_key  INT REFERENCES dim_zone(zone_key),
    dropoff_zone_key INT REFERENCES dim_zone(zone_key),
    vehicle_key      INT REFERENCES dim_vehicle(vehicle_key),
    fare_amount      DECIMAL(10,2),
    surge_multiplier DECIMAL(3,2),
    distance_km      DECIMAL(6,2),
    duration_min     INT,
    wait_time_min    INT,
    ride_status      VARCHAR(20),  -- 'completed', 'cancelled', 'no_show'
    cancel_reason    VARCHAR(50)
);

-- Dimension: dim_driver (SCD Type 2)
CREATE TABLE dim_driver (
    driver_key       SERIAL PRIMARY KEY,
    driver_id        INT NOT NULL,
    name             VARCHAR(200),
    vehicle_type     VARCHAR(20),
    city             VARCHAR(100),
    rating           DECIMAL(3,2),
    is_active        BOOLEAN DEFAULT true,
    effective_from   DATE NOT NULL,
    effective_to     DATE DEFAULT '9999-12-31',
    is_current       BOOLEAN DEFAULT true
);

-- Dimension: dim_zone
CREATE TABLE dim_zone (
    zone_key         SERIAL PRIMARY KEY,
    zone_id          INT NOT NULL,
    zone_name        VARCHAR(100),  -- 'Orchard', 'CBD', 'Jurong East'
    city             VARCHAR(100),
    is_airport       BOOLEAN DEFAULT false,
    is_mall          BOOLEAN DEFAULT false
);

-- Key queries this enables:
-- Peak hours: SELECT hour, COUNT(*) FROM fact_rides JOIN dim_date GROUP BY hour
-- Zone demand: SELECT zone_name, COUNT(*) FROM fact_rides JOIN dim_zone GROUP BY zone_name
-- Driver earnings: SELECT driver_id, SUM(fare) FROM fact_rides GROUP BY driver_id
-- Surge analysis: SELECT zone, AVG(surge) FROM fact_rides WHERE hour BETWEEN 7 AND 9
```
</details>

### Exercise 3: Schema Interview Question

**Question:** "A food delivery company wants to track both orders AND customer reviews. How would you model this?"

<details>
<summary>🔑 Answer</summary>

Two fact tables sharing dimensions:

```sql
-- Fact 1: fact_orders (already exists)
-- Fact 2: fact_reviews (NEW)
CREATE TABLE fact_reviews (
    review_key       SERIAL PRIMARY KEY,
    date_key         INT REFERENCES dim_date(date_key),
    customer_key     INT REFERENCES dim_customer(customer_key),
    restaurant_key   INT REFERENCES dim_restaurant(restaurant_key),
    order_key        BIGINT,  -- link to original order (optional)
    food_rating      INT CHECK (food_rating BETWEEN 1 AND 5),
    delivery_rating  INT CHECK (delivery_rating BETWEEN 1 AND 5),
    overall_rating   INT CHECK (overall_rating BETWEEN 1 AND 5),
    review_text      TEXT,
    is_flagged       BOOLEAN DEFAULT false  -- for inappropriate content
);

-- Shared dimensions: dim_date, dim_customer, dim_restaurant
-- This is called a "conformed dimension" pattern

-- Key analysis:
-- "Do 5-star restaurants get more orders?"
SELECT r.overall_rating, AVG(o.order_count) as avg_daily_orders
FROM fact_reviews rv
JOIN dim_restaurant r ON rv.restaurant_key = r.restaurant_key
JOIN (...) order_counts o ON ...
GROUP BY r.overall_rating;
```

The key insight: **conformed dimensions** (shared across fact tables) enable cross-analysis. This is what interviewers want to hear.
</details>

---

## 🌙 Evening (1 hour): Architecture Cheat Sheet + Review

### 🗺️ System Design Quick Reference

```
COMMON PATTERNS:
1. Batch ETL: Source → Airflow → S3 → dbt → Warehouse → BI
2. ELT: Source → Fivetran → Snowflake → dbt → BI  
3. Serverless: Source → S3 → Glue → Athena → Dashboard
4. Streaming: Events → Kafka → Flink/Spark Streaming → DB → Dashboard

TECHNOLOGY CHOICES:
- Small startup: S3 + Athena + dbt (cost: <$50/month)
- Growing company: Redshift/Snowflake + dbt + Airflow (cost: $200-500/month)
- Enterprise: Snowflake + Fivetran + dbt Cloud + Airflow (cost: $1K+/month)

SCHEMA DESIGN:
- Always start with star schema for analytics
- SCD Type 2 for slowly changing dimensions
- Surrogate keys, not natural keys
- Separate fact and dimension tables
- Conformed dimensions for cross-analysis

INTERVIEW FRAMEWORK:
1. Clarify: volume? sources? freshness? users? budget?
2. Draw: Source → Ingest → Store → Transform → Serve
3. Explain: WHY each tool, WHAT tradeoffs
4. Scale: "If volume 10x, I would..."

SG/MY SPECIFIC:
- Singapore data residency: keep data in ap-southeast-1
- PDPA compliance: handle PII carefully
- Multi-currency: SGD + MYR, normalize in transform layer
- Timezone: SGT (UTC+8), store in UTC, convert in presentation
```

### 📝 Today's Checklist

- [ ] Can explain batch vs streaming with examples
- [ ] Can draw 3 warehouse architecture patterns
- [ ] Walked through MakanExpress warehouse design
- [ ] Designed Shopee SG schema (star schema)
- [ ] Designed Grab ride analytics schema
- [ ] Can handle common follow-up questions
- [ ] System design cheat sheet saved

---

*Day 92 complete! Tomorrow: Schema design for SG/MY business scenarios.* 📊
