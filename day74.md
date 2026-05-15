# 📅 Day 74 — Sunday, 27 July 2026
# 🏗️ System Design Interview Practice: "Design a Data Pipeline"

---

## 🎯 Today's Goal

System design interviews are where they ask big-picture questions: "Design a data pipeline for X." You won't write code — you draw architecture, discuss tradeoffs, and explain your decisions. Today you learn the framework and practice with SG/MY scenarios.

**Philosophy:** System design interviews test your ability to THINK, not memorize. There's no single right answer. They want to see: Can you ask good questions? Can you make reasonable tradeoffs? Can you explain WHY you chose X over Y?

---

## ☀️ Morning Block (2 hours): System Design Framework

### The 4-Step Framework

```
Step 1: Clarify Requirements (5 min)
    ↓
Step 2: Propose High-Level Architecture (10 min)
    ↓
Step 3: Deep Dive into Components (15 min)
    ↓
Step 4: Discuss Tradeoffs & Scaling (10 min)
```

### Step 1: Clarify Requirements

Before designing ANYTHING, ask questions:

**Data questions:**
- What data sources? (APIs, databases, files, streaming?)
- How much data? (1K rows/day? 1M rows/day? 1B rows/day?)
- How often? (real-time? hourly? daily?)
- What format? (CSV, JSON, Parquet?)

**Business questions:**
- Who consumes the output? (analysts? dashboards? ML models?)
- What queries will they run? (aggregations? joins? filters?)
- How fresh does data need to be? (real-time? next day is fine?)

**Non-functional:**
- What's the SLA? (must finish by 6 AM?)
- Budget constraints?
- Team size? (1 person? 5 people?)

### Step 2: High-Level Architecture

Draw the big picture:

```
[Data Sources] → [Ingestion Layer] → [Storage Layer] → [Processing Layer] → [Serving Layer]
     │                  │                   │                  │                  │
   APIs             Airflow              S3/Raw           Spark/Glue          Athena/
   DBs              Kafka              S3/Processed       dbt                Redshift
   Files            Lambda             RDS/Data Warehouse  Python            Dashboard
```

### Step 3: Deep Dive

Pick 2-3 components and go deep:
- How does data flow between them?
- What happens when something fails?
- How do you monitor it?

### Step 4: Tradeoffs & Scaling

Always discuss:
- **Cost vs Performance:** "We could use Redshift (fast, expensive) or Athena (slower, cheaper per query)"
- **Batch vs Streaming:** "Do we need real-time, or is daily batch sufficient?"
- **Normalization vs Denormalization:** "Star schema for analytics, normalized for OLTP"
- **Build vs Buy:** "Use managed services (Glue, RDS) vs self-managed (EC2, self-hosted Postgres)"

### Common Data Pipeline Patterns

**Pattern 1: Batch ETL (most common for fresh grad roles)**
```
Source DB → Extract (Python) → S3 Raw → Transform (dbt/Glue) → Data Warehouse → BI Tool
                                ↑
                          Scheduled by Airflow
```

**Pattern 2: ELT (modern approach)**
```
Source → Extract & Load directly (Fivetran/Airbyte) → Data Warehouse → Transform (dbt) → BI
```

**Pattern 3: Serverless (AWS-native)**
```
S3 Raw → Glue Crawler → Glue Job (PySpark) → S3 Processed → Athena → QuickSight
```

### Exercise 1: Practice the Framework

Question: "Design a daily ETL pipeline for a food delivery company in Singapore."

Spend 10 minutes answering these clarifying questions before looking at the answer:

1. What data sources would you ask about?
2. How much data would you estimate?
3. What would the output be used for?
4. What questions would you ask the interviewer?

<details>
<summary>🔑 Sample clarification questions</summary>

1. **Data sources:**
   - "Is there an orders database? What type — PostgreSQL, MySQL?"
   - "Do you have a payments system? Separate from orders?"
   - "Restaurant data — is it in the same system?"
   - "Customer data — any CRM?"
   - "Any external data? Weather, events, promotions?"

2. **Volume estimation:**
   - "How many orders per day?" (GrabFood SG: ~100K-500K orders/day)
   - "How many restaurants?" (~30,000 in SG)
   - "How many customers?" (~5M Grab users in SG)
   - "Historical data needed?" ("We need 2 years of history")

3. **Output:**
   - "Who uses the data? Business analysts? Data scientists? Executives?"
   - "What kind of queries? Daily revenue, top restaurants, customer retention?"
   - "Dashboard? Automated reports? Ad-hoc analysis?"

4. **Questions for interviewer:**
   - "Is this batch or near-real-time?"
   - "What's the current tech stack?"
   - "Budget for cloud services?"
   - "Is there an existing pipeline we're replacing?"
   - "What's the SLA — when must data be ready by?"
</details>

---

## 🌤️ Afternoon Block (2 hours): Three Practice Scenarios

### Scenario 1: "Design a daily analytics pipeline for GrabFood Singapore"

**Requirements (assume interviewer clarified):**
- 500,000 orders/day from PostgreSQL
- 30,000 restaurants
- Need daily reports by 8 AM SGT
- Analysts run ad-hoc queries
- Budget: cost-effective (startup, not Grab-level budget)

**Architecture:**

```
┌─────────────┐     ┌──────────┐     ┌─────────────┐     ┌──────────────┐
│  Orders DB  │────→│ Airflow  │────→│  S3 Data    │────→│    Athena    │
│ (PostgreSQL)│     │ (DAG at   │     │    Lake     │     │  (ad-hoc     │
│             │     │  2 AM SGT)│     │ raw/        │     │   queries)   │
│  Payments   │     │          │     │ processed/  │     └──────┬───────┘
│  (API)      │────→│  Extract │     │ analytics/  │            │
│             │     │  scripts │     │             │     ┌──────▼───────┐
│  Restaurants│     │          │     │  Glue Job   │     │  QuickSight  │
│  (API)      │────→│  Load to │     │  (Transform)│────→│  (Dashboard) │
│             │     │  S3 Raw  │     │             │     │              │
└─────────────┘     └──────────┘     └─────────────┘     └──────────────┘
```

**Key decisions to explain:**

1. **Why S3 + Athena over Redshift?**
   - Redshift: $0.25/hour per node (minimum ~$180/month)
   - Athena: $5 per TB scanned. 500K orders/day × 365 days ≈ 50GB ≈ $0.25/query
   - For a startup with <10 analysts, Athena is much cheaper

2. **Why Airflow over simple cron?**
   - Dependencies between tasks (extract orders → then extract payments → then join)
   - Retry logic, monitoring, alerting
   - Visibility (Airflow UI shows pipeline status)

3. **Why Parquet for processed/analytics layers?**
   - 60-80% smaller than CSV
   - Columnar = fast aggregations
   - Schema embedded = no separate schema file

4. **Why Glue over self-managed Spark?**
   - No cluster to manage
   - Pay only when running
   - Built-in crawlers for schema discovery

### Scenario 2: "Design a real-time order tracking dashboard"

**Key difference:** This needs real-time (minutes, not daily).

```
┌─────────────┐     ┌──────────┐     ┌──────────────┐     ┌───────────┐
│  Orders API │────→│  Kinesis │────→│  Lambda      │────→│ DynamoDB  │
│  (streaming)│     │  Streams │     │  (process &  │     │ (current  │
│             │     │          │     │   aggregate) │     │  state)   │
│  Driver GPS │────→│          │     │              │     │           │
│  (streaming)│     │          │     │              │     │  API      │
│             │     │          │     │              │     │  Gateway  │
│  Restaurant │     │          │     │  Also write  │     │     ↓     │
│  updates    │────→│          │────→│  to S3 for   │     │ Dashboard │
│             │     │          │     │  batch layer │     │ (React)   │
└─────────────┘     └──────────┘     └──────────────┘     └───────────┘
```

**Key talking points:**
- Lambda architecture: real-time layer (Kinesis+Lambda+DynamoDB) + batch layer (S3+Athena)
- Why Kinesis over Kafka: managed service, no cluster management
- Why DynamoDB for current state: single-digit millisecond reads
- Acknowledge: "At fresh grad level, I'd implement the batch layer first. Real-time adds significant complexity."

### Scenario 3: "Design a data warehouse for Shopee Singapore"

**Requirements:**
- Multiple source systems: orders, products, sellers, customers, payments, logistics
- Need: business reporting, seller analytics, product recommendations
- 1M+ orders/day across Southeast Asia
- Analysts + data scientists use it

**Star Schema Design:**

```
                    ┌──────────────┐
                    │ dim_customer │
                    │ - customer_id│
                    │ - name       │
                    │ - country    │
                    │ - segment    │
                    └──────┬───────┘
                           │
┌──────────────┐     ┌─────▼──────────┐     ┌──────────────┐
│ dim_product  │────→│  fact_orders   │←────│ dim_seller   │
│ - product_id │     │ - order_id     │     │ - seller_id  │
│ - name       │     │ - date_key     │     │ - shop_name  │
│ - category   │     │ - customer_key │     │ - country    │
│ - price      │     │ - product_key  │     └──────────────┘
└──────────────┘     │ - seller_key   │
                     │ - quantity     │     ┌──────────────┐
┌──────────────┐     │ - amount       │←────│ dim_date     │
│ dim_payment  │     │ - discount     │     │ - date_key   │
│ - payment_id │     │ - status       │     │ - year/month │
│ - method     │     └───────┬────────┘     │ - weekday    │
│ - gateway    │             │              │ - holiday    │
└──────────────┘             │              └──────────────┘
                     ┌───────▼────────┐
                     │ dim_logistics  │
                     │ - order_id     │
                     │ - courier      │
                     │ - status       │
                     │ - delivery_date│
                     └────────────────┘
```

**Technology choices to discuss:**

| Component | Budget Option | Enterprise Option |
|-----------|--------------|-------------------|
| Data Warehouse | RDS PostgreSQL | Redshift / Snowflake |
| Orchestration | Airflow (self-hosted) | MWAA (managed Airflow) |
| Transformation | dbt | dbt Cloud |
| Storage | S3 | S3 (same) |
| Query Engine | Athena | Redshift Spectrum |
| BI | Metabase (free) | Tableau / QuickSight |

**As a fresh grad, explain the budget option — it shows practical thinking.**

### Exercise 2: Walk Through a Scenario

Pick one of the 3 scenarios above. Spend 15 minutes practicing explaining it out loud as if you're in an interview. Key phrases to use:

- "First, I'd like to clarify the requirements..."
- "Let me start with a high-level architecture..."
- "For this component, I chose X because..."
- "The main tradeoff here is between A and B..."
- "If data volume doubled, I would..."

<details>
<summary>🔑 What interviewers look for</summary>

1. **Structured thinking** — Do you follow a framework? (Clarify → Design → Deep dive → Tradeoffs)
2. **Asking questions** — Do you clarify before designing?
3. **Practical knowledge** — Do you know real tools and when to use them?
4. **Tradeoff awareness** — Do you understand that every choice has pros and cons?
5. **Communication** — Can you explain clearly?
6. **Honesty** — Do you admit what you don't know? ("I haven't used Kinesis directly, but conceptually it's like Kafka for AWS...")
</details>

---

## 🌙 Evening (1 hour): Common Follow-ups + Quick Reference

### Common Follow-up Questions

| Question | Good Answer |
|----------|-------------|
| "What if data volume 10x?" | "Add partitioning, consider Redshift/Snowflake, optimize queries" |
| "What if pipeline fails at 3 AM?" | "Airflow retries + alerts (Slack/email). Dead letter queue for bad records" |
| "How do you handle late-arriving data?" | "SCD Type 2 for dimensions. Upsert pattern for facts. Watermark for streaming" |
| "How do you ensure data quality?" | "Great Expectations, dbt tests, row count checks, null checks, freshness monitoring" |
| "How do you secure the pipeline?" | "IAM roles (not keys), VPC for databases, encryption at rest (S3 SSE), secrets manager" |
| "What's the cost?" | "Break down: S3 (~$23/TB/month), Athena ($5/TB scanned), RDS (~$15/month free tier)" |

### 🗺️ Quick Reference: System Design Framework

```
1. CLARIFY (5 min)
   - Data volume? Sources? Format?
   - Who uses it? What queries?
   - Real-time or batch?
   - Budget? Team size?

2. HIGH-LEVEL DESIGN (10 min)
   - Draw architecture (Source → Ingest → Store → Transform → Serve)
   - Name specific tools for each component
   - Explain data flow

3. DEEP DIVE (15 min)
   - Pick 2-3 components
   - Explain WHY you chose them
   - Show schema design if relevant

4. TRADEOFFS (10 min)
   - Cost vs performance
   - Batch vs streaming
   - Build vs buy
   - Scaling considerations

KEY PHRASES:
- "Let me clarify the requirements first..."
- "I'd choose X over Y because..."
- "The main tradeoff is..."
- "If we need to scale, we could..."
- "For a startup, I'd recommend X. For enterprise, Y."
```

### 📝 Today's Checklist

- [ ] Learned the 4-step system design framework
- [ ] Practiced clarifying requirements
- [ ] Walked through GrabFood daily analytics scenario
- [ ] Walked through real-time order tracking scenario
- [ ] Walked through Shopee data warehouse scenario
- [ ] Practiced explaining out loud
- [ ] Can answer common follow-up questions
- [ ] Framework cheat sheet saved

---

*Day 74 complete! Tomorrow: Behavioral interview prep — the STAR method and DE-specific stories.* 🎤
