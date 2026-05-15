# 📅 Day 69 — Tuesday, 22 July 2026
# 📝 Week 10 Assessment + First Full Mock Interview

---

## 🎯 Today's Goal

Week 10 assessment in the morning, then your FIRST full mock interview in the afternoon. This simulates a real data engineering technical interview at a Singapore company.

---

## ☀️ Morning Block (2 hours): Week 10 Assessment

### Section A: Data Quality (15 min, 5 questions)

**Q1.** Name 6 types of data quality checks and give a MakanExpress example for each.

**Q2.** What is a "data quality gate" in a pipeline? Why is it placed between transform and load?

**Q3.** Write a SQL query to find all orders where `total_amount` is 0 but status is 'delivered' (a business rule violation).

**Q4.** Explain how Great Expectations works. What is an "expectation suite"?

**Q5.** What is data freshness? Write a SQL check that alerts if the latest order is older than 24 hours.

### Section B: Testing Patterns (15 min, 5 questions)

**Q6.** Write a Python function that validates a DataFrame's schema (checks that all expected columns exist with correct types).

**Q7.** What is the difference between a null check and a completeness check? When would you use each?

**Q8.** Write code to detect if a column's distribution has shifted compared to a baseline (mean shift detection).

**Q9.** Explain the concept of "baseline stats" for data monitoring. How do you build and maintain them?

**Q10.** Design a monitoring dashboard for MakanExpress. What 5 metrics would you track and why?

### Section C: SQL Live Coding (30 min, 5 questions)

Use the MakanExpress schema. Write complete queries:

**Q11.** Find the top 3 customers by number of distinct restaurants they've ordered from.

**Q12.** Calculate the running total of revenue by day, with a 7-day moving average.

**Q13.** Find restaurants where this month's revenue dropped by more than 20% compared to last month.

**Q14.** For each customer, find their favorite restaurant (the one they ordered from most).

**Q15.** Write a query that identifies potential duplicate orders (same customer, same restaurant, same amount, within 10 minutes).

### Section D: System Design Short Answers (10 min, 5 questions)

**Q16.** "Design a data pipeline that processes 100K food delivery orders per day." Describe the architecture in 3-4 sentences.

**Q17.** How would you handle late-arriving data in your pipeline? (Orders that arrive hours after they were placed.)

**Q18.** Your pipeline loaded incorrect data into production. What steps do you take?

**Q19.** Explain idempotency. Why is it important for data pipelines? How do you achieve it?

**Q20.** You notice a 50% drop in daily orders. How do you investigate?

---

## 🌤️ Afternoon Block (2 hours): Answers + Mock Interview

### Scoring

| Section | Points |
|---------|--------|
| A: Data Quality | 20 |
| B: Testing Patterns | 20 |
| C: SQL Live Coding | 40 |
| D: System Design | 20 |
| **Total** | **100** |

---

### ✅ Answers

<details>
<summary>🔑 Section A: Data Quality</summary>

**A1:**
1. Not Null — order_id must have a value
2. Unique — no duplicate order_ids
3. Range — total_amount between $0.01 and $10,000
4. Accepted Values — status in [pending, delivered, cancelled]
5. Freshness — latest order within 24 hours
6. Referential Integrity — every customer_id exists in customers table

**A2:** A DQ gate is a validation step that runs checks before loading data. If checks fail, the pipeline stops. It's placed between transform and load so bad data never reaches the destination.

**A3:** `SELECT * FROM orders WHERE total_amount = 0 AND status = 'delivered';`

**A4:** Great Expectations is a Python framework where you define "expectations" (rules) about your data. An expectation suite is a collection of expectations that are run together. GX validates data against the suite and generates a report.

**A5:** `SELECT CASE WHEN MAX(order_date) < NOW() - INTERVAL '24 hours' THEN 'STALE' ELSE 'FRESH' END as status FROM orders;`

</details>

<details>
<summary>🔑 Section B: Testing Patterns</summary>

**A6:** SchemaValidator class that compares DataFrame columns/dtypes against a dictionary of expected schema. Checks for missing columns, extra columns, and type mismatches.

**A7:** Null check: Is there ANY null in this column? (Binary pass/fail). Completeness check: What percentage of values are non-null? (Gradient — 95%, 98%, etc.). Use null check for required fields, completeness check for optional fields where some nulls are acceptable.

**A8:** Calculate current mean, compare against baseline mean. If `abs(current - baseline) / baseline > tolerance%`, flag as shifted. Can also use z-score or standard deviation comparison.

**A9:** Baseline stats are historical statistics (mean, std, min, max, median) saved after each pipeline run. Build by computing stats on cleaned data. Maintain by updating the baseline periodically (rolling average of last 30 days).

**A10:** 1) Freshness: latest order timestamp per table. 2) Volume: daily row count vs baseline. 3) Null rate: percentage of nulls in key columns. 4) Distribution: mean order amount vs baseline. 5) Error rate: failed pipeline runs per day.

</details>

<details>
<summary>🔑 Section C: SQL Live Coding</summary>

**A11:**
```sql
SELECT c.name, COUNT(DISTINCT o.restaurant_id) as restaurant_count
FROM orders o JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'delivered'
GROUP BY c.id, c.name
ORDER BY restaurant_count DESC LIMIT 3;
```

**A12:**
```sql
WITH daily AS (
    SELECT DATE(order_date) as d, SUM(total_amount) as rev
    FROM orders WHERE status = 'delivered' GROUP BY DATE(order_date)
)
SELECT d, rev,
    SUM(rev) OVER (ORDER BY d) as running_total,
    AVG(rev) OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as moving_avg_7d
FROM daily ORDER BY d;
```

**A13:**
```sql
WITH monthly AS (
    SELECT restaurant_id, DATE_TRUNC('month', order_date) as m, SUM(total_amount) as rev
    FROM orders WHERE status = 'delivered'
    GROUP BY restaurant_id, DATE_TRUNC('month', order_date)
)
SELECT r.name, m.m, m.rev, LAG(m.rev) OVER (PARTITION BY m.restaurant_id ORDER BY m.m) as prev,
    ROUND((m.rev - LAG(m.rev) OVER (PARTITION BY m.restaurant_id ORDER BY m.m)) / 
          LAG(m.rev) OVER (PARTITION BY m.restaurant_id ORDER BY m.m) * 100, 1) as change_pct
FROM monthly m JOIN restaurants r ON m.restaurant_id = r.id
QUALIFY change_pct < -20;
```

**A14:**
```sql
WITH ranked AS (
    SELECT customer_id, restaurant_id, COUNT(*) as cnt,
        RANK() OVER (PARTITION BY customer_id ORDER BY COUNT(*) DESC) as rn
    FROM orders WHERE status = 'delivered'
    GROUP BY customer_id, restaurant_id
)
SELECT c.name, r.name as favorite_restaurant, rk.cnt as orders_there
FROM ranked rk
JOIN customers c ON rk.customer_id = c.id
JOIN restaurants r ON rk.restaurant_id = r.id
WHERE rk.rn = 1;
```

**A15:**
```sql
SELECT o1.id, o2.id, o1.customer_id, o1.total_amount
FROM orders o1 JOIN orders o2 ON o1.customer_id = o2.customer_id
    AND o1.restaurant_id = o2.restaurant_id
    AND o1.total_amount = o2.total_amount
    AND o1.id < o2.id
    AND ABS(EXTRACT(EPOCH FROM (o2.order_date - o1.order_date))) < 600;
```

</details>

<details>
<summary>🔑 Section D: System Design</summary>

**A16:** Use a message queue (SQS/Kafka) to ingest orders. Airflow orchestrates: extract from queue → validate with DQ checks → transform (clean, normalize, enrich) → load into PostgreSQL data warehouse (star schema) and S3 data lake (Parquet). dbt for analytics layer. Monitor with data quality checks and alerts.

**A17:** Use event time (order_date) not processing time. Store in staging with order_date. Process in daily batches using order_date partition. Add a "reprocess last 3 days" step that handles late arrivals. Use SCD Type 2 if dimensions change.

**A18:** 1) Stop the pipeline immediately. 2) Identify scope — which tables, which time range. 3) Restore from backup/snapshot if available. 4) Fix the bug. 5) Reprocess affected data. 6) Add a DQ check to prevent recurrence. 7) Document the incident.

**A19:** Idempotency = running the pipeline multiple times produces the same result. Important because pipelines get re-run (failures, backfills). Achieved with: upserts (INSERT ON CONFLICT), using business keys, truncating + reloading partitions, or SCD Type 2 with valid_from/valid_to.

**A20:** 1) Check if the source system is down (API health). 2) Check pipeline logs — did extraction succeed? 3) Compare today's volume to baseline (is 50% drop actually unusual?). 4) Check for data quality issues (nulls, errors). 5) Check if there's a calendar anomaly (public holiday?). 6) Escalate to source system team if needed.

</details>

---

### 🎙️ Mock Interview Simulation

**Pretend you're in an interview at Grab Singapore. The interviewer says:**

> "So tell me about a data pipeline you've built. Walk me through the architecture, what problems you solved, and what you'd improve."

**Your answer should cover:**
1. Problem statement (MakanExpress food delivery analytics)
2. Architecture (source → extract → transform → load → serve)
3. Tech choices (Python, Airflow, PostgreSQL, AWS S3, dbt)
4. Data quality (DQ gates, tests, monitoring)
5. What you'd improve (streaming, better monitoring, data catalog)

**Practice saying this out loud in 2-3 minutes.** Use the STAR method:
- **S**ituation: Food delivery company needed analytics
- **T**ask: Build a data warehouse and pipeline
- **A**ction: Designed star schema, built Airflow ETL, deployed on AWS
- **R**esult: Automated daily reporting, caught data quality issues, reduced manual work by 80%

---

## 🌙 Evening: Reflect

### 📝 Today's Checklist

- [ ] Completed assessment (90 min, no notes)
- [ ] Scored at least 70/100
- [ ] Practiced mock interview out loud
- [ ] Identified weak areas for Week 11
- [ ] Ready for tomorrow: Week review + weak spot targeting

---

*Day 69 complete! You just did your first mock interview. How did it feel?* 🎙️
