# 📅 Day 95 — Sunday, 17 August 2026
# 🔍 Company-Specific Prep: Research + Tailor Answers

---

## 🎯 Today's Goal

Generic interview prep works for 70% of the interview. The last 30% — the "why THIS company?" and "tell me about yourself" — requires company-specific research. Today you build a research framework and practice tailoring your answers for 3 real companies.

---

## ☀️ Morning Block (2 hours): Research Framework

### The 20-Minute Company Research Method

Before ANY interview, spend 20 minutes on this exact checklist:

**Step 1: Company Website (5 min)**
- What's their product/service? (1 sentence)
- What's their mission/vision?
- How big is the company? (headcount, revenue if public)
- Where are their offices? (SG? MY? Regional?)

**Step 2: Job Description Deep Read (5 min)**
- Top 3 required skills → highlight them
- Nice-to-have skills → note which you have
- Team size / reporting structure
- Any specific tools mentioned? (dbt? Airflow? Snowflake?)

**Step 3: Recent News (5 min)**
- Google: "[Company] news 2026"
- Any recent funding? Product launches? Expansion?
- Any data/AI-related announcements?

**Step 4: LinkedIn Research (5 min)**
- Find the hiring manager (if possible)
- Find current data engineers at the company
- What tools do they list in their profiles?
- Any shared connections?

### Research Template (fill this for each company)

```markdown
## Company Research: [Company Name]

### Quick Facts
- Product/Service:
- Size:
- HQ:
- Tech stack (from JD):
- Data team size:

### Why I'm Interested (1 sentence)
[SPECIFIC — not "I think your company is great"]

### What They're Looking For (from JD)
1. [Skill 1] → I have this from [project/experience]
2. [Skill 2] → I have this from [project/experience]
3. [Skill 3] → I have this from [project/experience]

### My Best Matching Project
[Which portfolio project is most relevant and why]

### Recent News
- [Any recent development that shows I researched them]

### Questions I Want to Ask Them
1. [About their data stack]
2. [About team culture]
3. [About growth opportunities]

### Custom "Why This Company" Answer
[2-3 sentences, referencing specific things I found]
```

---

## 🌤️ Afternoon Block (2 hours): Practice with 3 Real Companies

### Company 1: Grab (Data Engineer, Singapore)

**Research Template filled in:**

<details>
<summary>🔑 Grab research + tailored answers</summary>

**Quick Facts:**
- Product: Super-app (ride-hailing, food delivery, payments)
- Size: 8,000+ employees
- HQ: Singapore
- Tech stack (likely): Python, SQL, Airflow, Spark, Kafka, AWS
- Data team: Large, multiple sub-teams

**Why I'm interested:** "Grab's data infrastructure handles millions of transactions daily across Southeast Asia. Building pipelines at that scale is exactly what I want to learn."

**What they're looking for → My match:**
1. SQL → Week 1+4, 130+ practice problems
2. Python → Week 2, ETL pipeline project
3. ETL pipelines → 5 portfolio projects, Airflow experience
4. AWS → Week 6-7, serverless pipeline project

**Best matching project:** Food Delivery Warehouse — directly relevant to GrabFood's analytics needs.

**Custom "Why Grab?" answer:** "I use GrabFood almost daily, and I've always wondered how you process 500K+ food orders across Southeast Asia. My MakanExpress project actually models a simplified version of this — star schema, SCD Type 2, Airflow orchestration, AWS deployment. I'd love to see how it works at Grab's scale."

**Questions to ask:**
1. "How does Grab handle real-time vs batch processing for food delivery analytics?"
2. "What does the data engineering onboarding look like for new grads?"
3. "How does the data team collaborate with product and engineering?"
</details>

### Company 2: Shopee (Junior Data Engineer, Singapore/Malaysia)

<details>
<summary>🔑 Shopee research + tailored answers</summary>

**Quick Facts:**
- Product: E-commerce platform
- Size: 10,000+ employees
- HQ: Singapore (Sea Group)
- Tech stack: Python-heavy, SQL, data pipelines, analytics

**Why I'm interested:** "Shopee's growth in Southeast Asia is incredible — going from startup to dominant e-commerce platform. The data challenges behind product recommendations and seller analytics fascinate me."

**What they're looking for → My match:**
1. Python → Week 2, API pipeline, Pandas
2. SQL → Strong, 130+ problems
3. Data pipelines → Airflow, end-to-end ETL
4. Analytics mindset → Business-focused projects

**Best matching project:** Crypto Data Pipeline (Python + API) and Food Delivery Warehouse (analytics-focused)

**Custom "Why Shopee?" answer:** "I designed a star schema for an e-commerce food delivery platform — similar to what Shopee does but for food. I included seller analytics, customer segmentation, and product performance tracking. I'd be excited to apply these same patterns at Shopee's scale — especially the challenge of handling millions of SKUs across multiple countries."

**Questions to ask:**
1. "How does Shopee handle data from multiple countries with different currencies and languages?"
2. "What's the data stack — is it more batch or real-time for product recommendations?"
3. "How does the data team work with the ML team?"
</details>

### Company 3: DBS Bank (Data Engineer, Singapore)

<details>
<summary>🔑 DBS research + tailored answers</summary>

**Quick Facts:**
- Product: Banking and financial services
- Size: 30,000+ employees
- HQ: Singapore
- Known for: "World's Best Digital Bank" — strong data culture
- Tech stack: Likely enterprise (SQL, Python, data quality focus, compliance)

**Why I'm interested:** "DBS is known for its data-driven culture — being named 'World's Best Digital Bank' multiple times shows they take data seriously. I want to learn data engineering in an environment where data quality and governance matter."

**What they're looking for → My match:**
1. SQL → Strong, including optimization
2. Data quality → Week 10, Great Expectations, dbt tests
3. Python → ETL scripting
4. Compliance/security awareness → IAM policies, encryption

**Best matching project:** Food Delivery dbt (36+ tests, CI/CD) — shows data quality focus

**Custom "Why DBS?" answer:** "In my data quality week, I learned that bad data in banking can mean regulatory fines, not just wrong reports. I built automated testing into every project — 36+ tests in my dbt project alone, CI/CD pipelines that catch issues before they reach production. I know banking data requires an even higher standard, and I'm prepared for that level of rigor."

**Questions to ask:**
1. "How does DBS balance data democratization with data governance?"
2. "What data quality standards does the data engineering team follow?"
3. "How is the data team organized — centralized or embedded in business units?"
</details>

### Exercise 1: Research a Real Job Posting

Go to LinkedIn Jobs or MyCareersFuture right now. Find a real Data Engineer posting. Fill in the research template for it.

<details>
<summary>🔑 What to look for</summary>

- Does the JD mention specific tools? → Tailor your "skills" section of your answer
- Does it mention "fast-paced environment"? → Emphasize your quick learning (learned Airflow in a week)
- Does it mention "team collaboration"? → Mention your CI/CD, PR template, code review practices
- Does it mention "data quality"? → Highlight Great Expectations, dbt tests, automated checks
- Does it mention "cloud"? → Lead with AWS experience

The pattern: read their JD like a cheat sheet for what they want to hear.
</details>

---

## 🌙 Evening (1 hour): "Tell Me About Yourself" + Questions to Ask

### The Perfect "Tell Me About Yourself" (90 seconds)

```
"I'm [Name], an aspiring Data Engineer based in Singapore.

[Background] I'm currently completing my [degree/internship] and have spent 
the last 3 months building a portfolio of data engineering projects.

[Technical] I've built 5 end-to-end projects using SQL, Python, AWS, dbt, 
and Airflow — including a serverless pipeline on AWS and a star schema data 
warehouse with automated data quality checks.

[Why this company] What excites me about [Company] is [specific thing from 
research]. I believe my hands-on experience building production-quality 
pipelines would let me contribute from day one.

[Personal] Outside of data engineering, I enjoy [something relatable]."
```

### Questions YOU Should Ask (always prepare 3)

**Good questions to ask any company:**
1. "What does a typical day look like for a junior data engineer on your team?"
2. "What's the biggest data challenge your team is facing right now?"
3. "How does the data engineering team collaborate with data scientists and analysts?"
4. "What does the onboarding process look like for new data engineers?"
5. "What tools and technologies does the team use day-to-day?"
6. "How do you measure success for this role in the first 6 months?"

**Questions that show you're serious:**
1. "I noticed [Company] recently [news]. How does that affect the data team's priorities?"
2. "My MakanExpress project uses a similar architecture to what I imagine [Company] uses. How close is that to your actual setup?"
3. "What's the data team's philosophy on build vs buy for tools?"

### 📝 Today's Checklist

- [ ] Research template filled for 3 companies (Grab, Shopee, DBS)
- [ ] Researched 1 real job posting
- [ ] "Tell me about yourself" practiced (90 seconds)
- [ ] Prepared 3 questions to ask interviewers
- [ ] Can tailor "why this company" for any company in 10 minutes

---

*Day 95 complete! Tomorrow: Full mock interview.* 🎯
