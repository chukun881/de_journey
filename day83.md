# 📅 Day 83 — Tuesday, 5 August 2026
# 🗣️ Practice: Explaining Your Projects Concisely

---

## 🎯 Today's Goal

In interviews, you'll have 2-3 minutes to explain each project. Today you practice the "project pitch" — a structured, concise explanation that shows technical depth AND business thinking.

---

## ☀️ Morning Block (2 hours): The Project Pitch Framework

### The 3-Level Explanation

```
Level 1 (15 seconds): What is it? (for HR/recruiters)
Level 2 (90 seconds): How does it work? (for hiring managers)  
Level 3 (3 minutes): Technical deep dive (for engineers)
```

### Level 1: The One-Liner

For each project, write ONE sentence a non-technical person would understand:

| Project | One-Liner |
|---------|-----------|
| Food Delivery Warehouse | "I built a system that stores food delivery data and tracks how restaurants change over time" |
| Serverless Pipeline | "I built a cloud-based data processing system that transforms raw food delivery data into analytics without managing any servers" |
| Crypto Pipeline | "I built a program that automatically downloads cryptocurrency prices every day and stores them for analysis" |
| dbt Analytics | "I built an analytics layer that transforms raw food delivery data into business-ready reports with automated quality checks" |
| Retail Analysis | "I analyzed retail sales data to find business insights like top products and customer trends" |

### Level 2: The 90-Second Pitch

Template:
```
1. Problem (15 sec): What business problem does this solve?
2. Solution (30 sec): What did you build? High-level architecture.
3. Tech choices (30 sec): Why these tools? What tradeoffs?
4. Result (15 sec): What did you achieve? Quantified.
```

**Example — Food Delivery Warehouse:**

> "MakanExpress is a food delivery company that needs to analyze orders, track restaurant performance, and understand customer behavior. I built a complete data warehouse for this.
>
> The architecture is a star schema — a central orders fact table surrounded by dimension tables for customers, restaurants, products, dates, and payments. The pipeline runs daily on Airflow: it extracts new data, applies SCD Type 2 for tracking restaurant changes, and loads into the warehouse on AWS.
>
> I chose a star schema because it's optimized for analytical queries — analysts can slice and dice by any dimension. SCD Type 2 was important because restaurants change cuisines and locations, and the business needs to track that history. I used AWS RDS + S3 because it's cost-effective for a startup.
>
> The result: a fully automated pipeline that processes 50K+ records daily with data quality checks at every stage."

### Level 3: Technical Deep Dive (Be Ready For)

Interviewers may ask: "Walk me through your ETL pipeline step by step."

For each project, be ready to explain:
1. Data flow diagram (draw it on a whiteboard/virtually)
2. Schema design (tables, columns, relationships, why)
3. Error handling (what happens when something fails?)
4. Data quality (how do you know the data is correct?)
5. Performance (how long does it take? any optimizations?)
6. What you'd improve with more time

---

## 🌤️ Afternoon Block (2 hours): Practice All 5 Projects

### Exercise 1: Write Level 2 Pitches

Write a 90-second pitch for each of your 5 projects. Then practice saying them out loud with a timer.

<details>
<summary>🔑 Serverless Pipeline pitch example</summary>

"MakanExpress needs to analyze food delivery trends, but they don't want to manage servers. I built a completely serverless data pipeline on AWS.

Raw data lands in S3 — that's the data lake. AWS Glue runs PySpark jobs to clean and transform the data: parsing JSON, fixing schemas, converting to Parquet for compression. Then Athena lets analysts query the processed data with standard SQL, without moving it anywhere.

I chose this architecture because it's truly pay-per-use. Glue costs $0.44 per DPU-hour and Athena is $5 per terabyte scanned. For a startup processing 50K records daily, this costs under $10/month versus $180+ for a Redshift cluster.

The pipeline processes 50K records in under 3 minutes and reduces storage by 60% thanks to Parquet compression. All the infrastructure is defined in code and documented on GitHub."
</details>

### Exercise 2: Anticipate Technical Questions

For each project, write answers to:

1. **"Why did you choose [tool]?"** — always have a reason
2. **"What was the hardest part?"** — shows real experience
3. **"What would you do differently?"** — shows growth mindset
4. **"How would you scale this?"** — shows system thinking

<details>
<summary>🔑 Example answers for Food Delivery Warehouse</summary>

**Why dbt?** "dbt gives me version-controlled SQL transformations with built-in testing. Before dbt, I had SQL scripts scattered everywhere. With dbt, every model is tested, documented, and runs in a dependency graph."

**Hardest part?** "Getting SCD Type 2 right. The MERGE logic for detecting changes and inserting new rows was tricky — I had 3 bugs before it worked correctly. One bug was comparing timestamps in different timezones."

**What would you do differently?** "I'd add data quality monitoring from the start. I built the pipeline first and added quality checks later. In retrospect, tests should come before the pipeline."

**How would you scale?** "Move from RDS PostgreSQL to Redshift or Snowflake for the warehouse. Add partitioning by date. Use Glue instead of local Python for transforms. Add a streaming layer with Kinesis for real-time orders."
</details>

---

## 🌙 Evening (1 hour): Mock Practice

### Exercise 3: Simulated Interview

Set a timer. Spend 3 minutes on each project explaining it as if an interviewer asked "Tell me about this project on your GitHub."

Record yourself (phone voice memo) and listen back. Check:
- Is it under 3 minutes?
- Do you explain WHY, not just WHAT?
- Is there a business context?
- Does it sound natural (not scripted)?

### 📝 Today's Checklist

- [ ] Level 1 one-liners written for all 5 projects
- [ ] Level 2 pitches written and timed (90 seconds each)
- [ ] Practiced saying all 5 pitches out loud
- [ ] Prepared answers for "why this tool" for each project
- [ ] Prepared answers for "hardest part" for each project
- [ ] Recorded at least 2 practice runs

---

*Day 83 complete! Tomorrow: Week 12 assessment.* 📝
