# 📅 Day 94 — Saturday, 16 August 2026
# 🎤 Behavioral Deep Practice: STAR Stories with Live Drills

---

## 🎯 Today's Goal

You prepared 10 STAR stories on Day 75. Today you don't just re-read them — you PRACTICE them under pressure. Each story gets tested with follow-up questions, curveballs, and time limits. By the end, you can answer any behavioral question smoothly.

---

## ☀️ Morning Block (2 hours): Story Drills with Follow-ups

### How Today Works

For each of your 10 stories:
1. **Read the question** (it's a surprise — you pick which story fits)
2. **Answer in 2-3 minutes** (set a timer)
3. **Then I throw a follow-up** (the hard part)

The follow-up is what separates good candidates from great ones. Interviewers ALWAYS dig deeper.

### Drill 1: "Tell me about a time you solved a complex data problem"

**Your answer (2-3 min, use STAR)**

Then answer these follow-ups:

<details>
<summary>🔑 Follow-up questions + how to answer</summary>

**"What made this problem complex?"**
→ "Two things: first, the data came from 3 different sources with different schemas. Second, the SCD Type 2 logic needed to handle concurrent updates — the same restaurant could change cuisine AND location in the same batch."

**"What would you do differently if you had more time?"**
→ "I'd add automated data quality checks BEFORE the SCD logic runs. Currently, bad source data can create false 'changes' in the dimension table. A validation step would catch anomalies like a restaurant suddenly changing from 'Chinese' to 'Italian' (likely a data error, not a real change)."

**"How did you test your solution?"**
→ "I created a test dataset with 20 scenarios: new restaurants, changed restaurants, unchanged restaurants, multiple changes, and edge cases like null values. I verified each scenario produced the correct SCD Type 2 behavior before running on real data."

**"What was the hardest bug you encountered?"**
→ "The MERGE statement was comparing timestamps in different timezones — source data was in UTC, my logic was in SGT. Restaurants appeared to change when they hadn't. I fixed it by normalizing all timestamps to UTC before comparison."
</details>

### Drill 2: "Tell me about learning a new tool quickly"

<details>
<summary>🔑 Follow-up questions + answers</summary>

**"How do you normally approach learning new tools?"**
→ "I follow a pattern: Day 1 — read the docs + watch a tutorial. Day 2 — follow along building something. Day 3 — build something NEW without the tutorial. Day 4-5 — hit real problems, debug, read docs more carefully. Day 6-7 — build the actual project. This worked for Airflow, AWS Glue, and Docker."

**"What was confusing about [tool] at first?"**
→ "For Glue, the concept of 'crawlers' vs 'jobs' was confusing. A crawler discovers schema, a job transforms data. They sound similar but do different things. The 'aha' moment was: crawler = 'what does the data look like?', job = 'what should the data become?'"

**"How do you know when you've learned enough vs when to go deeper?"**
→ "For interviews: enough when I can explain it clearly and answer 'why this tool?' For the job: enough when I can build something working. I go deeper only when I hit a real problem I can't solve. Going too deep too early is wasted time."
</details>

### Drill 3: "Tell me about a time you found a data quality issue"

<details>
<summary>🔑 Follow-up questions + answers</summary>

**"How did you discover the issue?"**
→ "I was spot-checking query results and noticed the same crypto token appeared twice for the same timestamp. Then I checked more broadly — about 3% of records had duplicates. The API sometimes returns the same data point in consecutive pages due to a race condition on their end."

**"What was the impact? Had anyone used the bad data?"**
→ "Fortunately, I caught it before loading to the warehouse — my staging layer had the duplicates. But it taught me to ALWAYS validate before loading, not after. Now I have automated checks that run before every load."

**"How did you prevent it from happening again?"**
→ "Three layers: 1) Deduplication in the transform step using drop_duplicates, 2) A uniqueness test in dbt on (date, symbol), 3) An alert if row counts differ from expected by more than 5%."

**"How do you decide what data quality checks to implement?"**
→ "I prioritize by business impact. For a revenue column: NOT NULL, positive values only, within historical range. For a date column: not in the future, not too far in the past. For IDs: uniqueness. I use the 'what would embarrass the company?' test — if wrong data in this column would cause a bad business decision, it needs a check."
</details>

### Drill 4: "Tell me about a mistake you made"

<details>
<summary>🔑 Follow-up questions + answers</summary>

**"How long did it take you to notice the mistake?"**
→ "About 10 minutes — I saw the Airflow UI showing 90+ running tasks when it should have had 7. The DAG was running catchup for 3 months of history, firing off parallel tasks for every day."

**"What did you learn about yourself from this?"**
→ "I learn best by doing, but I need to read the defaults more carefully. `catchup=True` is the Airflow default, which is surprising. Now I always explicitly set `catchup=False` in every DAG."

**"How do you handle mistakes in general?"**
→ "Three steps: 1) Fix the immediate problem (stop the bleeding), 2) Understand why it happened (root cause), 3) Prevent it (add a guardrail). For this mistake: 1) Marked tasks as failed, 2) Learned about catchup default, 3) Added catchup=False to my DAG template."
</details>

### Drill 5: "Tell me about explaining something technical to a non-technical person"

<details>
<summary>🔑 Follow-up questions + answers</summary>

**"How do you adjust your communication for different audiences?"**
→ "Three levels: For executives — business impact ('this pipeline saves $5K/month'). For managers — what it does and why ('automated daily reports ready by 7 AM'). For engineers — how it works ('Airflow DAG with 4 tasks, SCD Type 2, incremental loading')."

**"Give me an example of a technical concept you'd explain simply."**
→ "ETL pipeline: Imagine a hawker center. Raw ingredients (raw data) arrive at the back door. The kitchen staff (transform step) cleans and prepares them. The finished dishes (processed data) go to the front for customers (analysts) to enjoy. A pipeline is just a kitchen for data."
</details>

---

## 🌤️ Afternoon Block (2 hours): Remaining Stories + Curveball Practice

### Drill 6-10: Rapid Fire

Practice the remaining 5 stories with these questions. Spend only 5 minutes per story (2 min answer + 3 min follow-up):

**Drill 6:** "Describe when requirements changed mid-project"
→ Your roadmap revision story. Follow-up: "How did you feel about dropping the cert week?"

**Drill 7:** "Tell me about working with messy data"
→ Crypto API null values and duplicates. Follow-up: "What's the messiest data you've seen?"

**Drill 8:** "How do you prioritize competing tasks?"
→ Study plan management (4-5 hrs/day, 15 weeks). Follow-up: "What do you do when you fall behind?"

**Drill 9:** "Tell me about improving a process"
→ Adding CI/CD to repos. Follow-up: "How did you measure the improvement?"

**Drill 10:** "What's your proudest achievement?"
→ Building 5 portfolio projects end-to-end. Follow-up: "Which project taught you the most?"

### Curveball Questions

These are surprise behavioral questions that DON'T map neatly to your prepared stories. Practice adapting:

**Q1.** "Tell me about a time you had to work with someone difficult."
<details>
<summary>🔑 Strategy</summary>
Adapt your "teamwork" experience. Even in solo study, you collaborated via GitHub, code reviews, or online communities. If you truly have no story: "I haven't had this situation in a professional context yet, but I'd approach it by understanding their perspective first, finding common ground, and focusing on the shared goal."
</details>

**Q2.** "What's your biggest weakness?"
<details>
<summary>🔑 Strategy</summary>
Pick a REAL weakness with evidence of improvement. Example: "I tend to over-engineer solutions. I built SCD Type 2 for a simple lookup table that only had 10 rows. I've learned to ask 'is this complexity necessary?' before implementing. Now I start simple and add complexity only when there's a proven need."
</details>

**Q3.** "Where do you see yourself in 3 years?"
<details>
<summary>🔑 Strategy</summary>
Show ambition but stay realistic for a fresh grad. "In 3 years, I want to be a mid-level data engineer who can independently design and build data platforms. I want to have deployed production pipelines that handle millions of records. I'm also interested in data platform engineering — the infrastructure side. I'd love to be at a company where I can grow from junior to owning a data domain."
</details>

**Q4.** "Why data engineering and not data science or software engineering?"
<details>
<summary>🔑 Strategy</summary>
"I enjoy building systems more than analyzing them. Data science is about finding insights — that's interesting, but I get more satisfaction from building the pipeline that makes those insights possible. Software engineering is too far from the data for me. Data engineering is the sweet spot: I get to work with data AND build systems."
</details>

**Q5.** "Why should we hire you over other fresh grads?"
<details>
<summary>🔑 Strategy</summary>
Don't compare to others — focus on YOUR differentiators. "I have 5 portfolio projects that demonstrate end-to-end ability — from raw data to analytics. Each project is on GitHub with CI/CD, documentation, and tests. I didn't just follow tutorials; I designed the architecture, made the tradeoffs, and documented my decisions. I'm ready to contribute from day one."
</details>

---

## 🌙 Evening (1 hour): Final Practice

### Exercise: Random Question Generator

Have a friend (or use a random number generator) pick questions from today's list. Answer without preparation. Practice transitioning smoothly between stories.

### 📝 Today's Checklist

- [ ] Practiced all 10 STAR stories with follow-up questions
- [ ] Can handle curveball behavioral questions
- [ ] Each story is 2-3 minutes, no filler
- [ ] Comfortable with "biggest weakness" and "why should we hire you"
- [ ] Stories feel natural, not memorized

---

*Day 94 complete! Tomorrow: Company-specific research.* 🔍
