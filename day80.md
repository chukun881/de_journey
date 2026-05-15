# 📅 Day 80 — Saturday, 2 August 2026
# 🌐 Portfolio Showcase: Final Polish + GitHub Pages Enhancement

---

## 🎯 Today's Goal

Your GitHub Pages portfolio site was created in Week 8. Today you give it a final professional polish — better project descriptions, screenshots, a "hire me" section, and SEO basics so recruiters can find you.

---

## ☀️ Morning Block (2 hours): Portfolio Content Polish

### Landing Page Structure

```html
<!-- Your index.md or index.html -->

# Hi, I'm [Your Name] 👋
**Aspiring Data Engineer | Singapore**

I build data pipelines and analytics solutions. Here's what I've created.

---

## 🚀 Featured Projects

[Project Card 1]
[Project Card 2]
[Project Card 3]

---

## 🛠️ Skills
[Visual skill tags]

---

## 📫 Let's Connect
[Contact info + links]
```

### Project Card Template

For each of your 5 projects, write a card:

```markdown
### 🍜 Food Delivery Data Warehouse
**SQL · dbt · Airflow · AWS (S3, RDS)**

End-to-end data warehouse for a food delivery platform (MakanExpress). 
Designed star schema with SCD Type 2, built Airflow DAGs for daily ETL, 
deployed on AWS with proper IAM security.

**Key features:**
- 5 fact tables, 8 dimension tables with full history tracking
- Automated daily pipeline with error handling and data validation
- CI/CD with GitHub Actions

[→ View on GitHub](link) | [→ Architecture Diagram](link)
```

### Exercise 1: Rewrite All 5 Project Descriptions

Rewrite each project description using this format:
1. **Name + emoji** — visual identity
2. **Tech stack tags** — scannable keywords
3. **2-3 sentence description** — what it does, why it matters
4. **3-4 key features** — bullet points with specifics
5. **Links** — GitHub repo + any live demos

<details>
<summary>🔑 All 5 project descriptions</summary>

**🍜 Food Delivery Data Warehouse**
SQL · dbt · Airflow · AWS (S3, RDS, IAM)

Complete data warehouse for MakanExpress food delivery analytics. Star schema with SCD Type 2 for historical tracking, orchestrated by Airflow, deployed on AWS.

Key features:
- Star schema: 5 fact + 8 dimension tables
- SCD Type 2 preserves 6+ months of restaurant history
- Airflow DAGs with retry logic, XCom, and data validation
- Deployed on AWS RDS + S3 with IAM roles

**☁️ MakanExpress Serverless Pipeline**
AWS Glue · Athena · PySpark · S3

Serverless data pipeline on AWS. Raw data lands in S3, gets transformed by Glue (PySpark), and queried with Athena — no servers to manage.

Key features:
- 3-zone data lake (raw/processed/analytics) with lifecycle policies
- PySpark ETL processing 50K+ records in <3 minutes
- Athena for SQL-on-S3 analytics
- 60% storage cost reduction (CSV → Parquet)

**💰 Crypto Data Pipeline**
Python · Pandas · REST API · PostgreSQL

Automated ETL pipeline that extracts cryptocurrency prices from CoinGecko API, transforms with Pandas, and loads into PostgreSQL.

Key features:
- Paginated API extraction with rate limiting and retry logic
- Data validation: null checks, deduplication, anomaly detection
- Upsert pattern for idempotent loads
- Logging and error tracking

**📊 Food Delivery dbt Analytics**
dbt · SQL · GitHub Actions

Analytics layer built with dbt. Raw data transforms through staging → intermediate → marts, with 36+ automated tests.

Key features:
- 10+ dbt models following staging/marts pattern
- Incremental models for efficiency
- Custom tests and macros
- CI/CD pipeline with GitHub Actions

**🛒 Retail Sales Analysis**
SQL · PostgreSQL

SQL analysis of retail sales data. 15+ business queries covering revenue, customer behavior, and product performance.

Key features:
- Multi-table database with 5 tables
- Complex joins, CTEs, window functions
- Business-focused queries (top products, customer segments, trends)
- Complete data dictionary
</details>

---

## 🌤️ Afternoon Block (2 hours): Visual Enhancements

### Add Architecture Diagrams

For your top 2 projects, add ASCII architecture diagrams to both the GitHub README and the portfolio page:

```
Data Flow Architecture:

┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Source   │───→│  S3 Raw  │───→│  Glue    │───→│  S3      │
│  Data     │    │  Zone    │    │  ETL     │    │  Analytics│
│  (CSV/API)│    │          │    │ (PySpark)│    │  Zone    │
└──────────┘    └──────────┘    └──────────┘    └────┬─────┘
                                                     │
                                               ┌─────▼─────┐
                                               │  Athena   │
                                               │  (Query)  │
                                               └───────────┘
```

### Add Screenshots

If your projects have:
- Query outputs → screenshot the results
- Airflow DAG graph → screenshot the Airflow UI
- CI/CD passing → screenshot the green checks
- GitHub Pages site → screenshot the live page

Store in `assets/` folder in each repo, reference in README.

### SEO for Your Portfolio

Recruiters Google "data engineer portfolio github" — make sure they find you:

1. **Page title:** `Your Name | Data Engineer Portfolio`
2. **Meta description:** Include "data engineer", "SQL", "Python", "AWS", "Singapore"
3. **H1 tag:** Your name + "Data Engineer"
4. **Alt text on images:** "Food Delivery Data Warehouse Architecture Diagram"
5. **README.md** on your username repo: Include "Data Engineer" prominently

### Mobile-Friendly Check

Open your GitHub Pages site on your phone. Is it readable? Buttons clickable? Text not too small?

---

## 🌙 Evening (1 hour): Final Review + Publish

### Quality Checklist

- [ ] Every project has a card with tech stack, description, and link
- [ ] Architecture diagrams for top 2-3 projects
- [ ] At least 3 screenshots across projects
- [ ] Contact info visible (email, LinkedIn, GitHub)
- [ ] "Hire Me" or availability section
- [ ] Mobile-friendly
- [ ] No broken links (test every link)
- [ ] Spell-checked
- [ ] Fast loading (no huge images)

### 📝 Today's Checklist

- [ ] Rewrote all 5 project descriptions
- [ ] Added architecture diagrams to top projects
- [ ] Added 3+ screenshots
- [ ] Portfolio site is mobile-friendly
- [ ] All links tested and working
- [ ] Published final version

---

*Day 80 complete! Tomorrow: Cover letter template.* ✉️
