# 📅 Day 78 — Thursday, 31 July 2026
# 📄 Resume Writing: Data Engineer Resume for SG/MY Market

---

## 🎯 Today's Goal

Your resume has ONE job: get you past the initial screen to a technical interview. In Singapore and Malaysia, recruiters spend 6-10 seconds on a resume. Today you write a data engineer resume that passes both human screening AND ATS (Applicant Tracking System) software.

**Philosophy:** A resume is not your life story. It's a marketing document with one purpose: prove you can do the job. Every line should answer "so what? Why should they care?"

---

## ☀️ Morning Block (2 hours): Resume Structure + Content

### Resume Structure (1 page, strictly)

```
┌─────────────────────────────────────────────────┐
│  YOUR NAME                                       │
│  Singapore | email@example.com | LinkedIn URL     │
│  GitHub: github.com/yourname                     │
├─────────────────────────────────────────────────┤
│  PROFESSIONAL SUMMARY (3-4 lines)                │
│  Aspiring Data Engineer with hands-on experience │
│  building ETL pipelines, data warehouses, and    │
│  analytics solutions using SQL, Python, AWS, dbt │
│  and Airflow. 5 portfolio projects on GitHub.    │
├─────────────────────────────────────────────────┤
│  SKILLS                                          │
│  Languages: SQL, Python                           │
│  Data: dbt, Pandas, PySpark                      │
│  Cloud: AWS (S3, Glue, Athena, RDS, IAM)         │
│  Tools: Airflow, Docker, Git, GitHub Actions      │
├─────────────────────────────────────────────────┤
│  EXPERIENCE                                      │
│  Data Engineering Intern | [Company]              │
│  [Dates]                                         │
│  - Bullet points with RESULTS                    │
├─────────────────────────────────────────────────┤
│  PROJECTS                                        │
│  Food Delivery Data Warehouse                    │
│  - Tech: SQL, dbt, Airflow, AWS                  │
│  - What you built + impact                       │
├─────────────────────────────────────────────────┤
│  EDUCATION                                       │
│  [Degree] | [University] | [Year]                │
└─────────────────────────────────────────────────┘
```

### Professional Summary Formula

```
[Title] with [X months/years] of experience [doing what]. 
Skilled in [top 3-4 skills relevant to the job]. 
[Brief achievement or differentiator].
```

**Examples:**

> "Aspiring Data Engineer with hands-on experience building end-to-end data pipelines and analytics solutions. Skilled in SQL, Python, AWS (S3, Glue, Athena), dbt, and Airflow. Built 5 portfolio projects including a serverless data pipeline on AWS and a star schema data warehouse with SCD Type 2. Currently completing a 15-week intensive data engineering program."

> "Data Engineering intern with experience in ETL pipeline development and data modeling. Proficient in SQL, Python, dbt, Airflow, and AWS cloud services. Built a food delivery analytics platform processing 50K+ records daily using serverless AWS architecture. Strong foundation in data quality, testing, and CI/CD practices."

### Skills Section: Keywords Matter

ATS systems scan for exact keywords from the job description. Include BOTH the skill AND context:

```
❌ "AWS" (too vague)
✅ "AWS (S3, Glue, Athena, RDS, IAM)" (specific)

❌ "SQL" (everyone says this)
✅ "SQL (complex joins, CTEs, window functions, query optimization)"

❌ "Python" 
✅ "Python (Pandas, ETL scripting, API integration, pytest)"

❌ "Databases"
✅ "PostgreSQL, RDS, S3 Data Lake, Athena"
```

### Experience Bullet Formula

```
[ACTION VERB] + [WHAT YOU DID] + [TOOLS USED] + [QUANTIFIED RESULT]

Example:
- Built automated ETL pipeline using Python, Airflow, and AWS S3 that processes 50K+ food delivery records daily with 99.9% data quality score
- Designed star schema data warehouse with SCD Type 2 for 5M+ historical records, reducing query time from 30 seconds to under 1 second
- Implemented data quality checks using Great Expectations and dbt tests, catching 15+ data issues before production
- Set up CI/CD pipeline with GitHub Actions for automated testing, reducing deployment time by 60%
```

### Action Verbs for Data Engineers

| Category | Verbs |
|----------|-------|
| Building | Built, Designed, Developed, Implemented, Created, Constructed |
| Improving | Optimized, Improved, Enhanced, Reduced, Accelerated, Streamlined |
| Analyzing | Analyzed, Investigated, Identified, Discovered, Diagnosed |
| Automating | Automated, Scheduled, Orchestrated, Configured |
| Collaborating | Collaborated, Partnered, Documented, Presented |

---

## 🌤️ Afternoon Block (2 hours): Project Descriptions + ATS Optimization

### Projects Section (most important for fresh grads)

Since you may not have extensive work experience, projects ARE your experience. List your top 3-4:

**Project 1: Food Delivery Data Warehouse**
```
Tech: SQL, dbt, Airflow, AWS (S3, RDS, IAM)
- Designed star schema with 5 fact tables and 8 dimension tables for MakanExpress food delivery analytics
- Implemented SCD Type 2 for tracking restaurant changes, preserving 6+ months of historical data
- Built Airflow DAGs for automated daily ETL with error handling, retry logic, and data validation
- Deployed on AWS (RDS PostgreSQL + S3 data lake) with proper IAM roles and security
- GitHub: github.com/yourname/food-delivery-warehouse
```

**Project 2: Serverless Data Pipeline (AWS)**
```
Tech: AWS Glue, Athena, PySpark, S3
- Built serverless ETL pipeline: S3 → AWS Glue (PySpark) → Athena for ad-hoc analytics
- Created 3-zone data lake architecture (raw/processed/analytics) with lifecycle policies
- Wrote PySpark transformations processing 50K+ records in under 3 minutes
- Reduced storage costs 60% by converting CSV to Parquet format
- GitHub: github.com/yourname/makanexpress-serverless
```

**Project 3: Crypto Data Pipeline (Python ETL)**
```
Tech: Python, Pandas, REST API, PostgreSQL
- Built automated ETL pipeline extracting data from CoinGecko API with pagination and rate limiting
- Implemented error handling, retry logic, and data validation for API reliability
- Loaded transformed data into PostgreSQL with upsert pattern for idempotent operations
- Added logging and monitoring for pipeline health tracking
- GitHub: github.com/yourname/crypto-data-pipeline
```

### ATS Optimization Checklist

- [ ] Save as PDF (not Word — formatting can break)
- [ ] Use standard section headers: "Experience", "Education", "Skills", "Projects"
- [ ] No tables, columns, or fancy formatting (ATS can't read them)
- [ ] Include exact keywords from the job description
- [ ] Use standard fonts (Arial, Calibri, Times New Roman)
- [ ] File name: `YourName_DataEngineer_Resume.pdf`
- [ ] No photos (not standard in SG/MY professional resumes)
- [ ] Include LinkedIn and GitHub URLs

### Exercise 1: Write Your First Draft

Write your complete 1-page resume. Include:
1. Header with contact info
2. Professional summary (3-4 lines)
3. Skills section (grouped by category)
4. Experience (internship, if any)
5. Projects (top 3-4)
6. Education

<details>
<summary>🔑 Common mistakes to avoid</summary>

1. **Too long** — Fresh grad resume = 1 page MAX. No exceptions.
2. **Generic summary** — "Hardworking team player" means nothing. Use specific skills.
3. **No GitHub link** — For data engineers, GitHub IS your portfolio. Always include it.
4. **Vague bullets** — "Worked on data pipeline" → "Built Python ETL pipeline processing 50K records/day"
5. **Spelling errors** — Run spell check. Then run it again. Then have someone else read it.
6. **Wrong file format** — Always PDF. Word docs can look different on different computers.
7. **No keywords** — If the job says "dbt" and your resume says "data build tool", ATS might miss it.
</details>

---

## 🌙 Evening (1 hour): Tailoring + Review

### Tailoring for Different Job Posts

You don't send the same resume everywhere. Adjust the:
- **Professional summary** — match their job title ("Data Engineer" vs "ETL Developer" vs "Analytics Engineer")
- **Skills order** — put their most-mentioned skills first
- **Project emphasis** — highlight the project most relevant to their stack

### Tailoring Example

**Job asks for:** SQL, Python, ETL, AWS, Airflow
→ Put SQL and Python first in skills, highlight Airflow + AWS projects

**Job asks for:** dbt, Snowflake, Python, data modeling
→ Put dbt first, emphasize star schema project, mention Snowflake knowledge (even if Athena-based, the concepts transfer)

### Free Resume Tools

- [Overleaf](https://www.overleaf.com/) — LaTeX templates (professional, ATS-friendly)
- [Resume.io](https://resume.io/) — Good free templates
- [VMock](https://www.vmock.com/) — AI resume feedback (free for some universities)
- [Jobscan](https://www.jobscan.co/) — Compare resume against job description for ATS match

### 📝 Today's Checklist

- [ ] Wrote complete 1-page resume draft
- [ ] Professional summary is specific and keyword-rich
- [ ] Skills section matches common DE job postings
- [ ] Top 3-4 projects listed with quantified bullets
- [ ] Saved as PDF
- [ ] ATS optimization checklist completed
- [ ] GitHub and LinkedIn links included

---

*Day 78 complete! Tomorrow: LinkedIn optimization.* 💼
