# 📅 Day 76 — Tuesday, 29 July 2026
# 🎯 Mock Interview #2: System Design + Behavioral

---

## 🎯 Today's Goal

Full mock interview practice: system design in the morning, behavioral in the afternoon. Treat this like a real interview — no notes, no Googling, no AI. Time yourself. Speak out loud (or write detailed answers if you can't speak). Grade yourself honestly afterward.

---

## ☀️ Morning Block (2 hours): System Design Mock Interview

### Scenario (spend 40 minutes)

**Interviewer says:** "I'd like you to design a data pipeline for a food delivery company similar to GrabFood. They operate in Singapore and Malaysia. They want to track daily orders, revenue, restaurant performance, and customer behavior. They have 500K orders per day across both countries. The data team has 5 analysts who need to run ad-hoc queries and build weekly reports. Budget is moderate — they're a Series B startup."

**Instructions:**
- Spend 5 minutes listing clarifying questions you'd ask
- Spend 15 minutes drawing/sketching the architecture
- Spend 15 minutes writing key design decisions with explanations
- Spend 5 minutes on scaling/tradeoff discussion

Do this WITHOUT looking at yesterday's notes. Then compare.

<details>
<summary>🔑 Sample Answer — Clarifying Questions</summary>

1. "What are the source systems? Do they have a single orders database or multiple systems?"
2. "How do they currently move data — is there an existing pipeline or are we starting from scratch?"
3. "What's the freshness requirement? Do analysts need same-day data, or is next-day acceptable?"
4. "What tools does the team currently know? (SQL, Python, etc.)"
5. "Is there a budget constraint for cloud services?"
6. "Do they need historical data migrated, or just going forward?"
7. "What's the expected growth — will 500K orders/day double in the next year?"
</details>

<details>
<summary>🔑 Sample Answer — Architecture</summary>

```
Source Systems:
  - Orders DB (PostgreSQL) → 500K/day
  - Payments (Stripe API) → daily settlement files
  - Restaurants (internal API) → menu updates, ratings
  - Customer app → behavior events (clickstream)

Ingestion (Airflow, scheduled 2 AM SGT):
  - Task 1: Extract orders from PostgreSQL (incremental, based on created_at)
  - Task 2: Fetch payment settlements from Stripe API
  - Task 3: Pull restaurant data from API
  - Task 4: Load clickstream from S3 (app team dumps there)

Storage (S3 Data Lake):
  - s3://company-data-lake/raw/orders/
  - s3://company-data-lake/raw/payments/
  - s3://company-data-lake/processed/ (Parquet, partitioned by country/date)
  - s3://company-data-lake/analytics/ (aggregated tables)

Transformation (dbt on Athena or dbt on Redshift):
  - Staging: clean, rename, cast types
  - Intermediate: join orders + payments, handle currency (SGD/MYR)
  - Marts: star schema for analytics

Serving:
  - Athena for ad-hoc queries ($5/TB scanned, cheap for 5 analysts)
  - QuickSight or Metabase for weekly dashboards
  - Data exposed to analysts via views

Key Design Decisions:
  1. Batch (daily) not streaming — reports are weekly, next-day is fine
  2. Athena over Redshift — 5 analysts don't justify Redshift costs ($180+/month minimum)
  3. dbt for transforms — version controlled, testable, documented
  4. S3 + Parquet — cost-effective, scales to any volume
  5. Airflow for orchestration — handles dependencies, retries, monitoring

Scaling (if volume 10x to 5M/day):
  - Switch from Athena to Redshift/Snowflake for faster queries
  - Add partitioning by country + date
  - Consider Glue for transforms instead of dbt-on-Athena
  - Add data quality monitoring (Great Expectations)
```
</details>

### Grading Rubric

| Criteria | Weight | Your Score (1-5) |
|----------|--------|-----------------|
| Asked clarifying questions | 15% | |
| High-level architecture makes sense | 25% | |
| Named specific tools with reasons | 20% | |
| Addressed tradeoffs | 20% | |
| Scaling discussion | 10% | |
| Communication clarity | 10% | |

---

## 🌤️ Afternoon Block (2 hours): Behavioral Mock Interview

### Answer these 5 questions (5 minutes each, out loud or written)

**Q1.** "Tell me about a time you had to learn something completely new to complete a project."

**Q2.** "Describe a situation where you found a significant error in data. What did you do?"

**Q3.** "Tell me about a time you had to balance quality with speed."

**Q4.** "Describe a situation where you disagreed with an approach. How did you handle it?"

**Q5.** "Tell me about your proudest technical achievement."

Spend 5 minutes on each, then review.

<details>
<summary>🔑 Self-evaluation checklist</summary>

For each answer, check:
- [ ] Used STAR format (Situation, Task, Action, Result)
- [ ] Focused on YOUR actions ("I did" not "we did")
- [ ] Quantified results where possible
- [ ] Kept to 2-3 minutes
- [ ] Showed self-awareness or learning
- [ ] Connected to data engineering skills
- [ ] Was honest (didn't exaggerate)

**Red flags to avoid:**
- Vague answers without specifics
- Blaming others for problems
- No clear result or outcome
- Too long (>4 minutes per answer)
- "We did everything" without individual contribution
</details>

### Common Weak Spots for Fresh Grads

| Weakness | Fix |
|----------|-----|
| Stories too short | Add specific details: what tools, what data, what queries |
| No quantification | Even learning projects have numbers: "processed 50K rows", "reduced from 4h to 45min" |
| "We" instead of "I" | Practice saying "My role was...", "I specifically..." |
| Only positive outcomes | Add what you learned, what you'd do differently |
| Generic answers | Use YOUR specific projects, not hypothetical scenarios |

---

## 🌙 Evening (1 hour): Review + Identify Gaps

### Today's Reflection

1. Which felt stronger — system design or behavioral?
2. Which questions made you stumble?
3. What topics do you need to review?
4. What stories need more practice?

### 📝 Today's Checklist

- [ ] Completed system design mock interview (40 min, no notes)
- [ ] Self-graded using the rubric
- [ ] Completed 5 behavioral questions (25 min total)
- [ ] Self-evaluated each answer
- [ ] Identified 2-3 areas for improvement
- [ ] Updated story notes if needed

---

*Day 76 complete! Tomorrow: Portfolio final review + interview prep planning.* ✅
