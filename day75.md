# 📅 Day 75 — Monday, 28 July 2026
# 🎤 Behavioral Interview Prep: STAR Method + DE-Specific Stories

---

## 🎯 Today's Goal

Technical skills get you the interview. Soft skills get you the job. Behavioral interviews ask "tell me about a time when..." — and your answer reveals how you think, collaborate, and handle challenges. Today you prepare 10 stories using the STAR method, specifically tailored for data engineering roles.

**Philosophy:** You're not making up stories — you're organizing real experiences from your learning journey into interview-ready narratives. Every bug you debugged, every pipeline you built, every dataset you cleaned is a potential STAR story.

---

## ☀️ Morning Block (2 hours): STAR Method + Story Framework

### What is the STAR Method?

```
S - Situation: Context (where, when, what project)
T - Task: What YOU needed to accomplish
A - Action: What YOU specifically did (step by step)
R - Result: Outcome (quantified if possible)
```

**Example:**

> **S:** "During my data engineering internship at [company], I was building an ETL pipeline for daily sales reports."
> **T:** "The pipeline was taking 4 hours to run, which meant reports weren't ready when the team arrived at 9 AM."
> **A:** "I profiled the slowest queries using EXPLAIN ANALYZE, discovered a full table scan on a 10M-row orders table. I added a composite index on (order_date, status), rewrote the query to use a CTE instead of a correlated subquery, and implemented incremental loading instead of full refresh."
> **Result:** "Pipeline runtime dropped from 4 hours to 45 minutes. Reports were ready by 7 AM. The team lead asked me to apply the same optimization to two other pipelines."

### Why STAR Works

- **Situation** shows you understand context (not just blindly coding)
- **Task** shows you can identify problems
- **Action** shows YOUR contribution (not "we did X")
- **Result** shows business impact (quantified = credible)

### Common Behavioral Questions for Data Engineers

1. "Tell me about a time you solved a complex data problem"
2. "Describe a situation where you had to learn a new tool quickly"
3. "Tell me about a time you found a data quality issue"
4. "Describe a project where requirements changed mid-way"
5. "Tell me about a time you had to explain technical concepts to non-technical people"
6. "Describe a situation where you made a mistake and how you handled it"
7. "Tell me about a time you worked with messy or incomplete data"
8. "Describe how you prioritize competing tasks"
9. "Tell me about a time you improved an existing process"
10. "Describe a challenging bug you debugged"

### Exercise 1: Map Your Experiences to Questions

Go through the 10 questions above. For each, identify which of your projects/learning experiences could answer it:

| Question | Which Project/Experience |
|----------|------------------------|
| 1. Complex data problem | Food Delivery Warehouse (star schema design) |
| 2. Learned new tool quickly | Airflow (Week 5) or AWS Glue (Week 7) |
| 3. Data quality issue | Great Expectations testing (Week 10) |
| 4. Changed requirements | ROADMAP revision (dropped cert week, added interview prep) |
| 5. Explain to non-technical | Your GitHub Pages portfolio site |
| 6. Made a mistake | Git merge conflicts, wrong SQL joins |
| 7. Messy data | Crypto API pipeline (handling null values, rate limits) |
| 8. Prioritize tasks | Study plan management (15 weeks, 4-5 hours/day) |
| 9. Improved process | Adding CI/CD to repos (Week 11) |
| 10. Debugged a bug | Airflow DAG failures, Docker networking issues |

<details>
<summary>🔑 Tips for mapping</summary>

- You CAN use learning experiences — "While building my portfolio project, I..." is valid
- The interviewer knows you're a fresh grad — they don't expect enterprise stories
- Focus on: problem-solving process, learning speed, self-awareness
- Each story should take 2-3 minutes to tell
- Prepare MORE stories than you need (some questions will surprise you)
</details>

---

## 🌤️ Afternoon Block (2 hours): Write Your Stories

### Story Template

For each story, write this out:

```
**Question:** "Tell me about a time..."
**Situation (2-3 sentences):**
**Task (1-2 sentences):**
**Action (4-5 sentences, what YOU did):**
**Result (2-3 sentences, quantified):**
**Key takeaway (1 sentence):**
```

### Exercise 2: Write 5 Stories

Write detailed STAR stories for these 5 questions. Aim for 200-300 words each.

**Story 1: Complex data problem**
<details>
<summary>🔑 Example answer</summary>

**Situation:** "As part of my data engineering portfolio, I was building a data warehouse for MakanExpress, a food delivery analytics project. I needed to design a star schema that could handle slowly changing dimensions — specifically, when restaurants change their cuisine category or location."

**Task:** "I needed to implement SCD Type 2 to track historical changes, while keeping the schema queryable for analysts who needed to see both current and historical data."

**Action:** "First, I researched SCD patterns and decided on Type 2 (add new row with effective dates) over Type 1 (overwrite) because the business needed historical tracking. I designed the dimension tables with `effective_from`, `effective_to`, and `is_current` columns. I wrote a PostgreSQL MERGE statement (upsert) that compares incoming data with existing records and creates new rows only when actual changes occur. I also created views that filter to `is_current = true` for analysts who only need current data."

**Result:** "The schema now tracks full history while keeping queries simple for analysts. When I tested with 6 months of simulated restaurant data, the SCD logic correctly identified 47 changed records and preserved all historical versions. Query performance on current-data views was under 100ms for a 100K-row dimension table."

**Key takeaway:** "I learned that good data modeling isn't just about storing data — it's about making data easy to use for different audiences."
</details>

**Story 2: Learned a new tool quickly**
<details>
<summary>🔑 Example answer</summary>

**Situation:** "In Week 7 of my self-study program, I needed to learn PySpark and AWS Glue — tools I'd never used before — to build a serverless data pipeline."

**Task:** "I had 7 days to go from zero to building a functional pipeline that processes MakanExpress data using Glue and queries it with Athena."

**Action:** "I broke the week into phases: Days 1-2 for PySpark fundamentals (installed locally with pip, practiced DataFrame operations), Days 3-4 for AWS Glue (studied crawlers, jobs, visual ETL), Day 5 for Athena (SQL on S3), and Days 6-7 for the portfolio project. I took notes on every error I encountered and created a troubleshooting guide. When a Glue job failed due to incorrect IAM permissions, I traced the error logs, identified the missing policy, and documented the fix."

**Result:** "By Day 7, I had a working serverless pipeline: S3 → Glue (PySpark ETL) → Athena (queryable). The pipeline processes 50K food delivery records in under 3 minutes. I pushed everything to GitHub with detailed documentation, including the troubleshooting guide that would help others avoid the same issues."

**Key takeaway:** "Structured learning with clear daily goals helped me go from zero to productive in a week. I now apply this approach to any new tool."
</details>

**Story 3: Data quality issue**
<details>
<summary>🔑 Example answer</summary>

**Situation:** "While building my API data pipeline that fetches cryptocurrency prices and weather data for Singapore, I noticed that some days had duplicate records and others had missing data."

**Task:** "I needed to identify the root cause, fix the data quality issue, and prevent it from happening again."

**Action:** "I added logging to track API response codes and discovered two issues: the CoinGecko API sometimes returned duplicate data for the same timestamp, and the weather API returned null values for certain hours. I implemented three fixes: 1) Added a deduplication step using `df.drop_duplicates(subset=['date', 'symbol'])`, 2) Added null checks with `df.isnull().sum()` and forward-fill for missing weather data, 3) Created a data validation function that checks row counts, null percentages, and value ranges before loading. I also set up alerts to notify me if validation fails."

**Result:** "After the fix, the pipeline has run for 30 consecutive days with zero data quality issues. The validation catches 2-3 potential problems per week (usually API hiccups) before they reach the database. I later applied the same validation framework to all my projects using Great Expectations."

**Key takeaway:** "Trust your data, but verify it. Automated data quality checks are not optional — they're how you sleep at night."
</details>

**Story 4: Explain technical concepts**
<details>
<summary>🔑 Example answer</summary>

**Situation:** "I created a GitHub Pages portfolio to showcase my data engineering projects. I realized the audience includes both technical recruiters and non-technical hiring managers."

**Task:** "I needed to explain complex projects (star schema, ETL pipelines, AWS serverless architecture) in a way that non-technical people could understand and find impressive."

**Action:** "I restructured each project description with three layers: 1) A one-sentence 'What it does' in plain English, 2) A 'Why it matters' section explaining the business problem, 3) A 'Technical details' section for engineers who want depth. For example, instead of saying 'Implemented SCD Type 2 with effective date ranges,' I wrote: 'Tracks how restaurants change over time — when a hawker stall upgrades from "local" to "famous" cuisine, the system keeps the history so analysts can study the impact.' I also added architecture diagrams and sample query outputs."

**Result:** "A technical recruiter at a Singapore data company commented that my portfolio was one of the clearest they'd seen from a fresh graduate. The three-layer approach means both HR and engineering leads can find value in my portfolio."

**Key takeaway:** "Communication is a data engineering superpower. If you can't explain your work, it doesn't matter how good the code is."
</details>

**Story 5: Made a mistake**
<details>
<summary>🔑 Example answer</summary>

**Situation:** "While building my Airflow DAG for the MakanExpress pipeline, I accidentally ran a DAG with `catchup=True` and a start date 3 months in the past."

**Task:** "I needed to stop the runaway DAG, clean up the mess, and prevent it from happening again."

**Action:** "The DAG started running 90 backfill tasks simultaneously, overwhelming my local PostgreSQL. First, I marked all running tasks as failed to stop the cascade. Then I cleared the duplicate data by identifying records with the same business key but different Airflow execution dates. I updated the DAG with `catchup=False`, added a clear comment about the start date behavior, and wrote a pre-flight checklist in my DAG file. I also added a data validation step that checks for duplicate records before any downstream processing."

**Result:** "The cleanup took 2 hours but recovered all data correctly. More importantly, I documented the mistake and the fix in my project README. In a later interview, I used this story to demonstrate my debugging process and my ability to learn from mistakes. The interviewer appreciated the honesty and the systematic approach to prevention."

**Key takeaway:** "Everyone makes mistakes. What matters is how quickly you detect them, how you fix them, and what you do to prevent them from recurring."
</details>

---

## 🌙 Evening (1 hour): Prepare 5 More Stories + Practice

### Exercise 3: Write Remaining Stories

Write STAR stories for questions 6-10 (from the list above). Use the same template. These can be shorter — 150-200 words each.

### Tips for Delivery

1. **Use "I" not "we"** — interviewers want to know YOUR contribution
2. **Quantify results** — "reduced by 80%" is stronger than "made it faster"
3. **Keep it to 2-3 minutes** — practice with a timer
4. **Be honest** — if the result wasn't perfect, say so and explain what you learned
5. **Show self-awareness** — "Looking back, I would have..." shows growth

### 📝 Today's Checklist

- [ ] Understood the STAR method (Situation, Task, Action, Result)
- [ ] Mapped 10 experiences to common behavioral questions
- [ ] Wrote 5 detailed STAR stories (200-300 words each)
- [ ] Wrote 5 shorter STAR stories (150-200 words each)
- [ ] Practiced telling at least 3 stories out loud
- [ ] Stories saved for review before interviews

---

*Day 75 complete! Tomorrow: Mock interview #2 (system design + behavioral).* 🎤
