# 📅 Day 63 — Wednesday, 16 July 2026
# 🎯 Week 9 Wrap-Up: Portfolio Enhancement + Reflection

---

## 🎯 Today's Goal

Week 9 is done! You've learned APIs, LLMs, RAG, and built some powerful tools. Today you:
1. Polish the API pipeline project for your portfolio
2. Ensure all Week 9 work is pushed to GitHub
3. Update your GitHub Pages site with new projects
4. Reflect on progress and plan for Week 10

---

## ☀️ Morning Block (2 hours): Portfolio Polish

### Project 1: API Data Pipeline — Final Polish

Ensure your `api-data-pipeline` repo has:

```markdown
# 📡 API Data Pipeline

Automated data pipeline that collects weather and cryptocurrency data from free APIs,
transforms it, and loads into PostgreSQL and AWS S3.

## Architecture
\`\`\`
OpenMeteo API ──┐                    ┌── PostgreSQL (upsert)
                ├── Extract ────────►│
CoinGecko API ──┘      │            └── S3 (CSV + Parquet)
                       ▼
                  Transform
               (clean, normalize, join)
\`\`\`

## Features
- ✅ Multi-source extraction (weather + crypto)
- ✅ Rate limiting and exponential backoff
- ✅ Data validation before loading
- ✅ PostgreSQL upsert (idempotent)
- ✅ S3 Parquet + CSV output
- ✅ Airflow DAG for hourly scheduling
- ✅ Local file fallback when S3 unavailable
- ✅ Comprehensive logging

## Data Sources
| Source | Data | Cost |
|--------|------|------|
| OpenMeteo | Weather for SG/KL/Penang/JB | Free |
| CoinGecko | Crypto prices (BTC, ETH, SOL) | Free tier |

## Quick Start
\`\`\`bash
pip install -r requirements.txt
cp .env.example .env
python run_pipeline.py
\`\`\`

## Tech Stack
![Python](https://img.shields.io/badge/Python-3.11-blue)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue)
![AWS S3](https://img.shields.io/badge/AWS-S3-orange)
![Airflow](https://img.shields.io/badge/Airflow-2.8-green)
```

---

### Project 2: AI Query Tool — Final Polish

Ensure the repo includes:
- `engine.py` — the main AI query engine
- `demo.py` — example usage
- `tests/` — test file
- `README.md` — with screenshots of sample queries
- `.env.example` — with required env vars

---

### Update GitHub Pages Portfolio

Add to your portfolio site:

```markdown
## 🚀 Projects

### 1. Retail Sales Analysis (SQL)
Full SQL analysis of retail data with complex joins, CTEs, and window functions.
`PostgreSQL` `SQL`

### 2. Crypto Data Pipeline (Python ETL)
Automated ETL pipeline extracting crypto prices from CoinGecko API.
`Python` `Pandas` `API` `PostgreSQL`

### 3. Food Delivery Analytics (dbt)
Complete dbt analytics project with star schema, incremental models, and CI/CD.
`dbt` `SQL` `Data Modeling`

### 4. Food Delivery Warehouse (Star Schema + Airflow + AWS)
Data warehouse with SCD Type 2, Airflow orchestration, and AWS deployment.
`PostgreSQL` `Airflow` `AWS` `Docker`

### 5. Serverless Data Pipeline (AWS)
Serverless pipeline using S3, Glue, and Athena for MakanExpress data.
`AWS Glue` `Athena` `PySpark` `S3`

### 6. API Data Pipeline ⭐ NEW
Multi-source data pipeline with weather + crypto APIs, PostgreSQL, S3, and Airflow.
`Python` `REST API` `Airflow` `PostgreSQL` `S3`

### 7. AI Query Tool ⭐ NEW
Natural language to SQL engine powered by LLM + RAG + pgvector.
`OpenAI` `pgvector` `PostgreSQL` `Python`
```

---

### Morning Exercises

**Exercise 1:** Run the API pipeline end-to-end one more time. Take screenshots of:
- Console output showing successful extraction
- PostgreSQL query results
- S3/local file listing

Add these screenshots to your repo's README.

**Exercise 2:** Write a `CONTRIBUTING.md` for the API pipeline repo that explains how someone else could:
- Add a new data source
- Run tests
- Set up their own environment

<details>
<summary>🔑 Answer</summary>

```markdown
# Contributing to API Data Pipeline

## Adding a New Data Source

1. Create a new extractor in `src/extract/`:
   \`\`\`python
   # src/extract/my_new_source.py
   class MyNewExtractor:
       def fetch_data(self):
           # Your extraction logic
           pass
   \`\`\`

2. Add configuration to `config.py`:
   \`\`\`python
   MY_NEW_API_URL = "https://api.example.com"
   \`\`\`

3. Update `run_pipeline.py` to call your extractor

4. Add transform logic in `src/transform/cleaner.py`

5. Add a loader if the data needs a new table

## Running Tests
\`\`\`bash
python -m pytest tests/ -v
\`\`\`

## Environment Setup
\`\`\`bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your credentials
\`\`\`
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Progress Reflection + Week 10 Prep

### 9-Week Progress Check

You've come a long way! Let's measure:

| Skill | Week Learned | Comfort Level (1-5) |
|-------|-------------|-------------------|
| SQL (complex queries) | 1, 4 | ? |
| Python (Pandas, ETL) | 2 | ? |
| Data Modeling (star schema, SCD) | 3-4 | ? |
| dbt | 3 | ? |
| Airflow | 5 | ? |
| AWS Core (S3, IAM, RDS, VPC) | 6 | ? |
| PySpark + Glue + Athena | 7 | ? |
| Docker | 8 | ? |
| REST APIs + pagination | 9 | ? |
| LLM APIs + prompt engineering | 9 | ? |
| RAG + vector search | 9 | ? |

**Rate yourself 1-5 for each.** Anything below 3 needs targeted review in Week 10.

### Portfolio Status

| # | Project | Status | Repo URL |
|---|---------|--------|----------|
| 1 | Retail Sales Analysis | ✅ Complete | |
| 2 | Crypto Data Pipeline | ✅ Complete | |
| 3 | Food Delivery Analytics (dbt) | ✅ Complete | |
| 4 | Food Delivery Warehouse | ✅ Complete | |
| 5 | Serverless Pipeline (AWS) | ✅ Complete | |
| 6 | API Data Pipeline | ✅ NEW | |
| 7 | AI Query Tool | ✅ NEW | |

**That's 7 portfolio projects!** Most fresh grads have 0-2. You're ahead of 95% of applicants.

---

### What's Coming in Week 10

Week 10 is where things shift from **learning** to **interview prep**:

| Day | Topic |
|-----|-------|
| Day 64 | Data Quality: Great Expectations, dbt tests, custom checks |
| Day 65 | Testing patterns: schema validation, null checks, freshness |
| Day 66 | Add data quality to all existing projects |
| Day 67 | SQL interview practice: 30 live coding problems |
| Day 68 | SQL interview practice: 30 medium-hard problems |
| Day 69 | Assessment + First full mock interview (SQL) |
| Day 70 | Week review + weak spot targeting |

**Key shift:** From "learn new tools" to "practice explaining what you know." Interview prep starts NOW.

---

## 🌙 Evening (1 hour): Push + Rest

### Final Push Checklist

- [ ] All Week 9 code pushed to GitHub
- [ ] API Data Pipeline repo has complete README
- [ ] AI Query Tool repo has complete README
- [ ] GitHub Pages updated with new projects
- [ ] All 7 repos have consistent formatting
- [ ] Reflection written (what went well, what was hard)

### 📝 Today's Checklist

- [ ] Polished API Data Pipeline README with badges and screenshots
- [ ] Polished AI Query Tool README
- [ ] Updated GitHub Pages with all 7 projects
- [ ] Self-assessed all 11 skills (rated 1-5)
- [ ] Pushed everything to GitHub
- [ ] Ready for Week 10: Data Quality + Interview Prep!

---

## 🎉 Week 9 Complete!

**What you learned this week:**
- 🌐 REST APIs: HTTP methods, status codes, JSON, requests library
- 📄 Pagination: offset, cursor, page-based patterns
- ⏱️ Rate limiting and exponential backoff
- 🔄 Complete ETL pipeline: extract → transform → load
- 🤖 LLM APIs: OpenAI, prompt engineering, cost management
- 🔍 Embeddings and vector similarity search
- 🏗️ RAG pipeline: retrieve → augment → generate
- 🛠️ Built 2 new portfolio projects

**Total portfolio projects: 7** — this is exceptional for a fresh grad. Most candidates have 1-2.

---

*Week 9 complete! Rest well — Week 10 starts interview prep.* 🚀💪
