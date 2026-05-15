# 📅 Day 56 — Wednesday, 9 July 2026
# 🌐 Portfolio Showcase: GitHub Pages Master Page

---

## 🎯 Today's Goal

You have 5 repos. They're polished, consistent, and professional. Today you create a **single landing page** that ties everything together — your GitHub Pages portfolio site. When a recruiter googles your name, this is what they find.

**Philosophy:** Most fresh grads send a PDF resume. You'll send a link to a living portfolio that shows your work, your code, and your growth story. This is the difference between "I know SQL" and "look what I built."

---

## ☀️ Morning Block (2 hours): Set Up GitHub Pages

### What is GitHub Pages?

Free static website hosting directly from your GitHub repository. You write Markdown or HTML, GitHub serves it as a website at `https://YOUR_USERNAME.github.io`.

**Why every data engineer should have one:**
- Shows you can document (key DE skill)
- Gives recruiters a single link to explore all your work
- Demonstrates web literacy (bonus skill)
- Professional — signals you take your career seriously

### Option A: Simple Markdown Site (Recommended — Fastest)

This is the simplest approach. One repo, one Markdown file, done in 30 minutes.

**Step 1: Create the repository**

```bash
# Create a new repo named YOUR_USERNAME.github.io
# Example: if your username is "chukunyew", the repo is "chukunyew.github.io"
```

On GitHub:
1. Click "New Repository"
2. Name it: `YOUR_USERNAME.github.io` (must match your username exactly)
3. Public
4. Add a README
5. Create

**Step 2: Create the landing page**

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_USERNAME.github.io.git
cd YOUR_USERNAME.github.io
```

Create `index.md`:

```markdown
# 👋 Hi, I'm [Your Name]

**Data Engineer** | Singapore / Malaysia | Fresh Graduate

I build end-to-end data platforms — from raw data ingestion to analytics dashboards. Currently completing a 15-week intensive Data Engineering program with 5 portfolio projects.

---

## 🛠️ Tech Stack

![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)
![Apache Airflow](https://img.shields.io/badge/Airflow-2.7-017CEE?logo=apache-airflow&logoColor=white)
![dbt](https://img.shields.io/badge/dbt-1.7-FF694B?logo=dbt&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-S3%20%7C%20Glue%20%7C%20Athena-FF9900?logo=amazon-aws&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-3.5-E25A1C?logo=apachespark&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)

---

## 📂 Portfolio Projects

### 🍜 [MakanExpress Serverless Pipeline](https://github.com/YOUR_USERNAME/makanexpress-serverless)
*Serverless data pipeline for a Singapore food delivery platform*

**What it does:** End-to-end serverless pipeline — raw CSV data lands in S3, AWS Glue transforms it, Athena queries it for analytics.

**Tech:** Python, PySpark, AWS Glue, Athena, S3, IAM

**Key features:**
- 3-zone S3 data lake (raw → processed → analytics)
- Glue crawlers with auto-schema detection
- PySpark ETL jobs for data transformation
- Athena queries for business analytics

---

### 🏪 [Food Delivery Data Warehouse](https://github.com/YOUR_USERNAME/food-delivery-warehouse)
*Star schema data warehouse with SCD Type 2, orchestrated by Airflow*

**What it does:** A complete data warehouse for food delivery analytics — star schema design, slowly changing dimensions, automated ETL pipeline.

**Tech:** PostgreSQL, Airflow, Python, AWS (S3, RDS)

**Key features:**
- Star schema with 4 dimension tables and 1 fact table
- SCD Type 2 for tracking customer and restaurant changes
- Airflow DAG for daily ETL automation
- Deployed on AWS RDS + S3

---

### 📊 [Food Delivery dbt Analytics](https://github.com/YOUR_USERNAME/food-delivery-dbt)
*Analytics layer built with dbt — models, tests, and documentation*

**What it does:** Transform warehouse data into business-ready analytics using dbt. Staging → intermediate → marts architecture.

**Tech:** dbt, SQL, PostgreSQL, GitHub Actions CI/CD

**Key features:**
- 10+ dbt models following best practices
- 36+ data quality tests
- Automated CI/CD pipeline
- Generated documentation site

---

### 🔄 [Crypto Data Pipeline](https://github.com/YOUR_USERNAME/crypto-data-pipeline)
*Automated ETL pipeline: API → Python → Database*

**What it does:** Extracts cryptocurrency data from CoinGecko API, transforms with Pandas, loads to PostgreSQL with error handling and logging.

**Tech:** Python, Pandas, REST API, PostgreSQL

**Key features:**
- API extraction with retry logic and rate limiting
- Pandas transformation with data cleaning
- PostgreSQL loading with CSV fallback
- Comprehensive logging and error handling

---

### 🛒 [Retail Sales Analysis](https://github.com/YOUR_USERNAME/retail-sales-analysis)
*Advanced SQL analytics on retail data*

**What it does:** Comprehensive analysis of retail sales data using advanced SQL techniques — window functions, CTEs, complex joins.

**Tech:** PostgreSQL, SQL

**Key features:**
- Multi-table database with 5 tables
- 15+ business insight queries
- Window functions, CTEs, subqueries
- Data dictionary and documentation

---

## 👨‍💻 About Me

I'm a fresh graduate passionate about data engineering, currently based in [Your City]. I'm building my skills to land a Data Engineer role in Singapore or Malaysia.

**What I'm learning:** SQL, Python, ETL pipelines, data modeling, cloud (AWS), orchestration (Airflow), dbt, and modern data stack tools.

**Fun fact:** All my projects use Singaporean and Malaysian food delivery examples because I believe the best way to learn is with data you can relate to. 🍜

---

## 📫 Get in Touch

- 📧 Email: your.email@example.com
- 💼 LinkedIn: [linkedin.com/in/yourprofile](https://linkedin.com/in/yourprofile)
- 🐙 GitHub: [github.com/YOUR_USERNAME](https://github.com/YOUR_USERNAME)

---

*Built with ❤️ and lots of kopi*
```

**Step 3: Enable GitHub Pages**

1. Go to repo Settings → Pages
2. Source: "Deploy from a branch"
3. Branch: `main` / `/ (root)`
4. Save
5. Wait 1-2 minutes
6. Visit `https://YOUR_USERNAME.github.io`

**Exercise 1:** Create your GitHub Pages site with the template above. Customize the content.

<details>
<summary>🔑 Customization tips</summary>

1. **Add a profile photo:** Create an `images/` folder, add your photo, reference it:
   ```markdown
   ![My Photo](images/profile.jpg){:height="150px"}
   ```
   Note: GitHub Pages Markdown doesn't support size attributes. Use HTML:
   ```html
   <img src="images/profile.jpg" height="150" align="left" style="border-radius: 50%; margin-right: 20px;">
   ```

2. **Add a theme:** Create a `_config.yml` file:
   ```yaml
   theme: jekyll-theme-minimal
   title: Your Name - Data Engineer
   description: Portfolio and projects
   ```

3. **Add Google Analytics** (track who visits): 
   Not essential for a portfolio, but shows analytics thinking.

4. **Custom domain** (optional): If you own a domain like `yourname.dev`, point it to GitHub Pages.

</details>

### Option B: Use a Template (Slightly More Effort, Better Looking)

If you want a more polished look, use a pre-built template:

**Recommended templates for data engineers:**
1. **Minimal Mistakes** — Clean, professional, widely used
2. **al-folio** — Academic/portfolio template, great for project showcases
3. **Beautiful Jekyll** — Simple and elegant

**Exercise 2:** If you want a template, fork `al-folio` and customize it.

<details>
<summary>🔑 Setting up al-folio</summary>

```bash
# 1. Fork https://github.com/alshedivat/al-folio to your account
# 2. Rename the fork to YOUR_USERNAME.github.io
# 3. Clone locally
git clone https://github.com/YOUR_USERNAME/YOUR_USERNAME.github.io.git
cd YOUR_USERNAME.github.io

# 4. Edit _config.yml — update name, description, url
# 5. Edit _pages/about.md — your bio
# 6. Create project pages in _projects/ folder
# 7. Push changes
git add .
git commit -m "docs: customize portfolio site"
git push origin main
```

Project file example (`_projects/makanexpress.md`):
```markdown
---
title: "MakanExpress Serverless Pipeline"
layout: single
author_profile: true
---

Serverless data pipeline for Singapore food delivery analytics...
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Project Descriptions + Personal Bio

### Part 1: Writing Project Descriptions That Sell

Each project description should answer 3 questions:
1. **What is it?** (1 sentence)
2. **What does it do?** (3-5 bullet points)
3. **What skills does it demonstrate?** (tech stack)

**Bad description:**
> "I made a food delivery project using SQL and Python."

**Good description:**
> "Serverless data pipeline for a Singapore food delivery platform. Raw CSV data lands in S3, AWS Glue transforms it with PySpark, and Athena enables SQL analytics. Demonstrates cloud ETL, data lake architecture, and cost-optimized serverless design."

**Exercise 3:** Write a compelling description for each of your 5 projects.

<details>
<summary>🔑 Template for project descriptions</summary>

```markdown
### 🏷️ [Project Name](link-to-repo)

**One-liner:** [What it is in one sentence]

**Problem:** [What business problem does it solve?]

**Solution:** [How does your project solve it?]

**Architecture:**
```
[Simple ASCII or Mermaid diagram]
```

**Tech Stack:** [List technologies]

**Key Features:**
- [Feature 1 — what it does and WHY it matters]
- [Feature 2]
- [Feature 3]

**What I Learned:** [1-2 sentences about your growth from this project]

**Sample Output:**
```
[Sample query or pipeline output]
```
```

**Example for MakanExpress Serverless:**
> **Problem:** Food delivery companies generate millions of orders. How do you analyze this data cost-effectively without managing servers?
> **Solution:** Build a serverless pipeline on AWS — S3 stores data at $0.023/GB, Glue transforms it on-demand, Athena queries it with standard SQL. Zero servers to manage.
> **What I Learned:** Serverless doesn't mean "no architecture." You still need to design S3 paths, choose formats (Parquet > CSV), set up partitioning for cost optimization, and manage IAM properly.

</details>

### Part 2: Skills Demonstrated Tags

Add skill tags to each project — recruiters scan for keywords:

```markdown
**Skills:** ETL Pipeline | Data Lake Architecture | AWS (S3, Glue, Athena) | PySpark | SQL Analytics | IAM Security | Cost Optimization
```

**Tagging strategy for SG/MY job applications:**

| Skill | Which Project(s) |
|-------|------------------|
| SQL (Complex Queries) | Retail Sales, Food Delivery dbt |
| Python (ETL) | Crypto Pipeline, Food Delivery Warehouse |
| Data Modeling (Star Schema) | Food Delivery Warehouse |
| dbt | Food Delivery dbt |
| Airflow (Orchestration) | Food Delivery Warehouse |
| AWS (Cloud) | MakanExpress Serverless |
| PySpark | MakanExpress Serverless |
| Docker | All (containerized) |
| Data Quality / Testing | Food Delivery dbt |
| Documentation | All |

### Part 3: Personal Bio Section

Your bio should be authentic, not corporate-speak. Here's a template:

```markdown
## 👨‍💻 About Me

I'm a fresh Computer Science graduate from [University], passionate about building data systems 
that turn raw information into business insights. 

**My journey:** I started with SQL basics and worked my way up to building complete data platforms 
— from designing star schemas to deploying serverless pipelines on AWS. Every project in my 
portfolio uses real-world scenarios from Singapore and Malaysia's food delivery ecosystem.

**What excites me:** The moment when a messy CSV becomes a clean, queryable table that reveals 
something unexpected — like discovering that nasi lemak orders spike 300% on rainy mornings.

**Currently:** Completing my Data Engineering intensive program (15 weeks) and actively seeking 
junior Data Engineer roles in Singapore or Malaysia.
```

**Exercise 4:** Write your personal bio. Keep it real — avoid buzzwords.

<details>
<summary>🔑 Bio dos and don'ts</summary>

**Do:**
- Mention specific technologies you've used
- Show personality (food delivery examples, local context)
- Be honest about being a fresh grad — it's not a weakness
- Mention what you're currently learning
- Include a hook (interesting fact or observation)

**Don't:**
- Say "passionate about leveraging synergies" — meaningless
- Claim expertise you don't have ("expert in distributed systems")
- Write a wall of text — keep it under 200 words
- Use third person ("John is a dedicated professional") — be authentic

**Good hooks for SG/MY context:**
- "I discovered that hawkers near MRT stations get 40% more delivery orders during lunch hours."
- "My first pipeline crashed because I didn't handle the DST timezone shift between Singapore and KL."
- "I chose food delivery as my domain because GrabFood and Foodpanda are what I use daily."

</details>

---

## 🌙 Evening (1 hour): Final Review + Publish + Reflection

### Final Review Checklist

```markdown
## Portfolio Site Checklist

### Content
- [ ] All 5 projects listed with descriptions
- [ ] Each project has: what it does, tech stack, key features
- [ ] Skills tags match job posting keywords
- [ ] Personal bio is authentic and concise
- [ ] Contact info is current (email, LinkedIn, GitHub)

### Visual
- [ ] Badges render correctly
- [ ] Links all work (test every one!)
- [ ] Mobile-friendly (check on your phone)
- [ ] No typos or broken formatting

### SEO (so recruiters find you)
- [ ] GitHub profile is complete (bio, location, link)
- [ ] Repos have descriptions and topics/tags
- [ ] LinkedIn profile links to GitHub Pages URL

### Professional
- [ ] No unprofessional content visible
- [ ] GitHub contribution graph looks active
- [ ] Pinned repos are your 5 portfolio projects
```

### Pin Your Best Repos

On your GitHub profile page:
1. Go to your profile → "Customize your pins"
2. Pin all 5 portfolio projects
3. This is the first thing recruiters see

### Push and Publish

```bash
cd YOUR_USERNAME.github.io
git add .
git commit -m "feat: create portfolio landing page with project showcase"
git push origin main
```

Visit `https://YOUR_USERNAME.github.io` — your portfolio is live! 🎉

---

## 🔮 Reflection: 8 Weeks of Progress

You've come an incredible distance. Take 15 minutes to reflect honestly:

```markdown
## 📊 Week 8 Reflection

### What I've Built (8 weeks):
- ✅ 5 portfolio projects on GitHub
- ✅ SQL: complex queries, window functions, CTEs, optimization
- ✅ Python: ETL scripts, Pandas, API integration, error handling
- ✅ Data Modeling: star schema, SCD types, dimensional modeling
- ✅ dbt: models, tests, macros, docs, CI/CD
- ✅ Airflow: DAGs, operators, scheduling, XCom, sensors
- ✅ AWS: S3, IAM, RDS, Glue, Athena, VPC
- ✅ PySpark: DataFrames, transformations, Glue integration
- ✅ Docker: containers, docker-compose, containerized pipeline
- ✅ Portfolio: polished READMEs, badges, diagrams, GitHub Pages

### What Surprised Me:
[What was harder/easier than expected?]

### My Strongest Areas:
[From assessment results]

### Areas That Need Work:
[From assessment results]

### Goals for Weeks 9-15:
1. Strengthen weak spots identified in assessment
2. Add advanced skills (APIs, LLM integration)
3. Professional polish (resume, LinkedIn)
4. Interview preparation — SQL live coding, system design
5. Start applying to jobs by Week 14
```

### Setting Intentions

Write down 3 specific goals for the next phase:

```
By end of Week 15 (Aug 28), I will:
1. _______________________________________________
2. _______________________________________________
3. _______________________________________________
```

### 📝 Today's Checklist

- [ ] Created GitHub Pages site (YOUR_USERNAME.github.io)
- [ ] Landing page with all 5 projects listed
- [ ] Each project has description, tech stack, key features
- [ ] Skills tags added for recruiter keyword matching
- [ ] Personal bio written (authentic, not corporate)
- [ ] Contact info current (email, LinkedIn)
- [ ] All links tested and working
- [ ] Site is mobile-friendly
- [ ] Pinned 5 repos on GitHub profile
- [ ] Completed Week 8 reflection
- [ ] Set 3 goals for Weeks 9-15
- [ ] Site is live at YOUR_USERNAME.github.io 🎉

---

## 📋 Quick Reference: GitHub Pages Setup

```
# 1. Create repo: YOUR_USERNAME.github.io
# 2. Create index.md with portfolio content
# 3. (Optional) Create _config.yml for theme
# 4. Settings → Pages → Branch: main / root
# 5. Wait 1-2 min → visit https://YOUR_USERNAME.github.io

# _config.yml (minimal)
theme: jekyll-theme-minimal
title: Your Name — Data Engineer
description: Portfolio and projects

# Useful themes:
# jekyll-theme-minimal     — Clean, simple
# jekyll-theme-cayman      — Colorful header
# jekyll-theme-slate       — Professional
# jekyll-theme-hacker      — Dark, dev-friendly
```

---

*Day 56 complete! Week 8 finished! 🎉*

*8 weeks down, 7 to go. You've built a solid foundation. Weeks 9-15 will sharpen your skills and prepare you for the job market. Take a moment to appreciate how far you've come — from SELECT * to serverless pipelines on AWS. That's real growth.* 🚀

*Tomorrow starts Week 9: Advanced Python + APIs + LLM Integration — the modern data engineer's toolkit.*
