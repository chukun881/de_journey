# 📅 Day 51 — Friday, 4 July 2026
# 🔗 Portfolio Integration: Connect All Projects

---

## 🎯 Today's Goal

You have 5 GitHub repos. Right now they look like 5 separate homework assignments. Today you transform them into a **cohesive portfolio** that tells a story: "I built a data platform end-to-end, from raw data to analytics."

**Philosophy:** Employers don't hire individual projects — they hire engineers who can see the big picture. Your repos should look like they're part of one intentional system, not random exercises.

---

## ☀️ Morning Block (2 hours): Audit & Standardize

### Step 1: List Your Repos

Your 5 portfolio repos:
1. `retail-sales-analysis` — SQL analysis (Week 1)
2. `crypto-data-pipeline` — Python ETL (Week 2)
3. `food-delivery-dbt` — dbt analytics (Week 3)
4. `food-delivery-warehouse` — Star schema + SCD + Airflow + AWS (Week 4-6)
5. `makanexpress-serverless` — AWS Glue + Athena serverless pipeline (Week 7)

### Step 2: The Standard README Template

Every repo should have the same README structure. Here's the template:

```markdown
# 🍜 [Project Name]

> Brief one-liner about what this project does and why it matters.

## 📋 Overview

**Problem:** What business problem does this solve? (2-3 sentences)

**Solution:** How does this project solve it? (2-3 sentences)

**Impact:** What insights or value does this provide? (1-2 sentences)

## 🏗️ Architecture

\```text
[ASCII architecture diagram showing data flow]

Source → Extract → Transform → Load → Query → Visualize
\```

## 🛠️ Tech Stack

| Category | Technology |
|----------|-----------|
| Language | Python 3.11 / SQL |
| Database | PostgreSQL / AWS RDS |
| Orchestration | Apache Airflow |
| Transformation | dbt / PySpark |
| Storage | AWS S3 |
| Cloud | AWS (S3, Glue, Athena, RDS) |

## 📊 Data Model

\```text
[Star schema / data model diagram if applicable]
\```

## 🚀 Quick Start

\```bash
# Clone
git clone https://github.com/yourusername/[repo-name].git
cd [repo-name]

# Setup
pip install -r requirements.txt

# Run
[command to run the project]
\```

## 📁 Project Structure

\```text
[repo-name]/
├── data/               # Sample data
├── scripts/            # ETL / analysis scripts
├── sql/                # SQL queries
├── tests/              # Tests
├── docs/               # Documentation
├── README.md
└── requirements.txt
\```

## 🔍 Sample Queries / Outputs

\```sql
-- Example: Top 10 restaurants by revenue
SELECT restaurant_name, SUM(total_amount) AS revenue
FROM fact_orders
GROUP BY restaurant_name
ORDER BY revenue DESC
LIMIT 10;
\```

| Restaurant | Revenue |
|-----------|---------|
| Ah Hock Nasi Lemak | $45,230 |
| Tian Tian Chicken Rice | $38,120 |
| ... | ... |

## 🔗 Related Projects

This project is part of the **MakanExpress Data Platform**:
- 📊 [Retail Sales Analysis](link) — SQL foundation project
- 🔄 [Crypto Data Pipeline](link) — Python ETL pipeline
- 🍜 [Food Delivery dbt](link) — dbt transformation layer
- 🏗️ [Food Delivery Warehouse](link) — Data warehouse + Airflow
- ☁️ [MakanExpress Serverless](link) — AWS serverless pipeline

## 📚 What I Learned

- [Key skill 1 learned in this project]
- [Key skill 2]
- [Challenge overcome]

---

Built with ☕ by [Your Name] | [LinkedIn] | [Portfolio]
```

### Step 3: Audit Each Repo (30 minutes)

Open each repo and check:

| Check | retail-sales | crypto-pipeline | food-delivery-dbt | food-delivery-warehouse | makanexpress-serverless |
|-------|:---:|:---:|:---:|:---:|:---:|
| README exists | ☐ | ☐ | ☐ | ☐ | ☐ |
| Problem statement | ☐ | ☐ | ☐ | ☐ | ☐ |
| Architecture diagram | ☐ | ☐ | ☐ | ☐ | ☐ |
| Tech stack table | ☐ | ☐ | ☐ | ☐ | ☐ |
| Setup instructions | ☐ | ☐ | ☐ | ☐ | ☐ |
| Sample query/output | ☐ | ☐ | ☐ | ☐ | ☐ |
| `.gitignore` | ☐ | ☐ | ☐ | ☐ | ☐ |
| `requirements.txt` | ☐ | ☐ | ☐ | ☐ | ☐ |
| Consistent naming | ☐ | ☐ | ☐ | ☐ | ☐ |
| No hardcoded secrets | ☐ | ☐ | ☐ | ☐ | ☐ |

**Exercise:** Go through each repo. For every missing checkbox, write it down. We'll fix them this afternoon.

<details>
<summary>🔑 Common Issues Checklist</summary>

**Things you'll probably find:**
1. README has no architecture diagram → Fix: Add ASCII art diagram
2. No `.gitignore` → Fix: Add standard Python `.gitignore`
3. Hardcoded passwords/connection strings → Fix: Use environment variables
4. No `requirements.txt` → Fix: Run `pip freeze > requirements.txt`
5. Inconsistent repo naming (some with hyphens, some with underscores) → Fix: Rename on GitHub
6. No sample output → Fix: Add a table or screenshot of results
7. Missing "What I Learned" section → Fix: Add 3-5 bullet points per repo
8. Old placeholder text from initial creation → Fix: Replace with real content
9. No LICENSE file → Fix: Add MIT License
10. Commit messages like "update" or "fix" → Fix: Can't easily fix, but improve going forward

**Critical:** Check for hardcoded secrets NOW. Use `grep -r "password" .` and `grep -r "secret" .` in each repo. If found, rotate those credentials immediately.

</details>

---

## 🌤️ Afternoon Block (2 hours): Fix & Cross-Link

### Step 4: Add Architecture Diagrams

For each repo, add an ASCII architecture diagram to the README. Here are templates for each:

**Repo 1: retail-sales-analysis**
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  CSV Files    │────▶│  PostgreSQL   │────▶│  SQL Queries  │
│  (raw data)   │     │  (database)   │     │  (analysis)   │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                 │
                                                 ▼
                                          ┌──────────────┐
                                          │   Results     │
                                          │  (tables/charts)│
                                          └──────────────┘
```

**Repo 2: crypto-data-pipeline**
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ CoinGecko API │────▶│  Python ETL  │────▶│  PostgreSQL  │────▶│  CSV Export  │
│  (JSON data)  │     │  (transform) │     │  (storage)   │     │  (analysis)  │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

**Repo 3: food-delivery-dbt**
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Raw PostgreSQL│────▶│  dbt Models  │────▶│  Analytics   │
│  (source)     │     │ staging→marts│     │  (insights)  │
└──────────────┘     └──────────────┘     └──────────────┘
                            │
                     ┌──────┴──────┐
                     │ dbt tests   │
                     │ + docs      │
                     └─────────────┘
```

**Repo 4: food-delivery-warehouse**
```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ CSV/JSON  │─▶│  Airflow  │─▶│  Star    │─▶│   SCD    │─▶│ AWS RDS  │
│ Sources   │  │  DAGs     │  │ Schema   │  │ Type 2   │  │ + S3     │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

**Repo 5: makanexpress-serverless**
```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  S3 Raw   │─▶│AWS Glue  │─▶│S3 Processed│─▶│ Athena   │─▶│ Analytics│
│  Zone     │  │  ETL Job │  │  Zone     │  │ Queries  │  │ Results │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### Step 5: Add Cross-Links

Edit each repo's README to include the "Related Projects" section. The links should form a connected graph:

```
retail-sales-analysis
    ↓ (skills from SQL project applied to...)
food-delivery-warehouse (star schema design)
    ↓ (warehouse feeds data to...)
food-delivery-dbt (transformation layer)
    ↓ (data infrastructure managed by...)
food-delivery-warehouse (Airflow orchestration)
    ↓ (entire pipeline deployed to...)
makanexpress-serverless (AWS deployment)

crypto-data-pipeline
    ↓ (independent project showing ETL skills)
[sideline project showing breadth]
```

**Exercise:** Write the exact "Related Projects" section for `food-delivery-warehouse`. Include all 5 repos with one-line descriptions.

<details>
<summary>🔑 Answer</summary>

```markdown
## 🔗 Related Projects

This project is part of the **MakanExpress Data Platform** portfolio:

- 📊 [Retail Sales Analysis](https://github.com/yourusername/retail-sales-analysis) — SQL foundation: complex joins, CTEs, window functions on retail data
- 🔄 [Crypto Data Pipeline](https://github.com/yourusername/crypto-data-pipeline) — Python ETL: API extraction, Pandas transformation, PostgreSQL loading
- 🍜 [Food Delivery dbt](https://github.com/yourusername/food-delivery-dbt) — dbt transformation: staging → marts, incremental models, testing, documentation
- 🏗️ **Food Delivery Warehouse** *(this repo)* — Data warehouse: star schema, SCD Type 2, Airflow orchestration, AWS deployment
- ☁️ [MakanExpress Serverless](https://github.com/yourusername/makanexpress-serverless) — AWS serverless: S3 data lake, Glue ETL, Athena analytics
```

</details>

### Step 6: Add Sample Outputs

Every README needs at least one sample query + output table. This proves your code actually works.

**Exercise:** For `retail-sales-analysis`, add a sample query that shows the top 5 product categories by revenue, with the output table.

<details>
<summary>🔑 Answer</summary>

```markdown
## 🔍 Sample Output

\```sql
-- Top 5 product categories by total revenue
SELECT 
    c.category_name,
    COUNT(oi.order_item_id) AS items_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    ROUND(AVG(oi.unit_price), 2) AS avg_unit_price
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
GROUP BY c.category_name
ORDER BY total_revenue DESC
LIMIT 5;
\```

| Category | Items Sold | Total Revenue | Avg Unit Price |
|----------|-----------|---------------|----------------|
| Electronics | 1,247 | $89,340.00 | $71.66 |
| Home & Garden | 892 | $34,560.50 | $38.75 |
| Sports | 634 | $28,190.00 | $44.46 |
| Books | 2,103 | $21,030.00 | $10.00 |
| Clothing | 756 | $18,900.00 | $25.00 |
```

</details>

### Step 7: Security Check

Run this in each repo directory:

```bash
# Check for hardcoded secrets
grep -rn "password\|secret\|api_key\|token" --include="*.py" --include="*.sql" --include="*.yml" . | grep -v ".git" | grep -v "example" | grep -v "template"

# Check .gitignore exists and covers common files
cat .gitignore | grep -E "\.env|__pycache__|\.csv|\.parquet|node_modules"
```

**Exercise:** What should your `.gitignore` contain for a Python data engineering project?

<details>
<summary>🔑 Answer</summary>

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.egg-info/
dist/
build/
.eggs/

# Environment
.env
.env.local
venv/
env/
.venv/

# Data files (too large for Git)
*.csv
*.parquet
*.json
!data/sample/*.csv

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# AWS
.aws/credentials

# dbt
dbt_packages/
logs/
target/

# Airflow
airflow.db
webserver_config.py
```

**Key rules:**
1. **Never commit** `.env` files, credentials, or large data files
2. **Do commit** sample/small data files in a `data/sample/` directory
3. **Do commit** `.env.example` with placeholder values (shows required env vars)
4. Add `*.csv` to gitignore but exclude `data/sample/*.csv` (the `!` negation)

</details>

---

## 🌙 Evening (1 hour): Shared Data Dictionary + Push

### Step 8: Create a Shared Data Dictionary

Create `data-dictionary.md` in your main portfolio or as a Gist. This shows employers you understand data documentation.

```markdown
# 📖 MakanExpress Data Dictionary

## Core Entities

### orders (fact_orders)
| Column | Type | Description | Example |
|--------|------|-------------|---------|
| order_id | BIGINT | Unique order identifier | 100001 |
| customer_id | VARCHAR(10) | Customer who placed the order | C0042 |
| restaurant_id | VARCHAR(10) | Restaurant fulfilling the order | R0015 |
| order_datetime | TIMESTAMP | When the order was placed | 2026-01-15 12:30:00 |
| delivery_address | TEXT | Delivery location | "123 Orchard Road, #05-01" |
| total_amount | DECIMAL(10,2) | Order total including GST | 25.50 |
| status | VARCHAR(20) | Order status | completed |
| delivery_time_min | INTEGER | Actual delivery duration | 32 |

### customers (dim_customers)
| Column | Type | Description | Example |
|--------|------|-------------|---------|
| customer_id | VARCHAR(10) | Unique customer ID | C0042 |
| customer_name | VARCHAR(100) | Full name | "Tan Wei Ming" |
| email | VARCHAR(200) | Email address | tan@example.com |
| signup_date | DATE | Account creation date | 2025-06-15 |
| preferred_area | VARCHAR(50) | Default delivery area | "Orchard" |
| is_active | BOOLEAN | Currently active? | true |
| scd_valid_from | TIMESTAMP | SCD Type 2 start | 2025-06-15 00:00:00 |
| scd_valid_to | TIMESTAMP | SCD Type 2 end | 9999-12-31 23:59:59 |
| scd_is_current | BOOLEAN | Latest version? | true |

### restaurants (dim_restaurants)
| Column | Type | Description | Example |
|--------|------|-------------|---------|
| restaurant_id | VARCHAR(10) | Unique restaurant ID | R0015 |
| restaurant_name | VARCHAR(200) | Business name | "Ah Hock Nasi Lemak" |
| cuisine_type | VARCHAR(50) | Primary cuisine | "Malay" |
| area | VARCHAR(50) | Operating area | "Orchard" |
| rating | DECIMAL(3,2) | Average rating (1-5) | 4.50 |
| is_halal | BOOLEAN | Halal certified? | true |
| avg_delivery_min | INTEGER | Average delivery time | 28 |

### menu_items (dim_menu_items)
| Column | Type | Description | Example |
|--------|------|-------------|---------|
| item_id | VARCHAR(10) | Unique menu item ID | I0100 |
| restaurant_id | VARCHAR(10) | Parent restaurant | R0015 |
| item_name | VARCHAR(200) | Dish name | "Nasi Lemak Special" |
| category | VARCHAR(50) | Menu category | "Main" |
| price | DECIMAL(8,2) | Unit price (SGD) | 8.50 |
| is_available | BOOLEAN | Currently on menu? | true |

### order_items (bridge table)
| Column | Type | Description | Example |
|--------|------|-------------|---------|
| order_item_id | BIGINT | Unique line item ID | 500001 |
| order_id | BIGINT | Parent order | 100001 |
| item_id | VARCHAR(10) | Menu item | I0100 |
| quantity | INTEGER | Quantity ordered | 2 |
| unit_price | DECIMAL(8,2) | Price per unit | 8.50 |

## Business Rules
- All monetary values in SGD (Singapore Dollars)
- Timestamps in SGT (UTC+8)
- Orders < 30 min delivery = "fast" benchmark
- Active customer = ordered in last 90 days
- Restaurant rating updated daily from customer reviews

## Data Flow
\```text
Raw CSV/JSON → S3 Raw Zone → Glue ETL → S3 Processed → 
Airflow → RDS Warehouse → dbt → Analytics Mart → Athena Queries
\```
```

### Step 9: Push Everything

```bash
# For each repo:
cd ~/projects/[repo-name]
git add .
git commit -m "docs: standardized README, added architecture diagram, cross-links, and data dictionary reference"
git push origin main
```

### Step 10: Verify Everything

Open each repo on GitHub and check:
- [ ] README renders correctly (check formatting, tables, code blocks)
- [ ] Architecture diagram is readable
- [ ] Cross-links work (click each one)
- [ ] No secrets visible in code
- [ ] Sample output looks professional

### 📝 Reflection

Answer these in your notes:

1. **Which repo has the strongest README?** What makes it good?
2. **Which repo needs the most work?** What's missing?
3. **If an employer saw only one repo, which would you want it to be?** Why?
4. **Does your portfolio tell a story?** Can someone see progression from SQL → Python → dbt → Airflow → AWS?

---

### 📝 Today's Checklist

- [ ] Audited all 5 repos against the standard template
- [ ] Added/updated README for each repo using template
- [ ] Created ASCII architecture diagrams for each repo
- [ ] Added cross-links connecting all 5 repos
- [ ] Added at least 1 sample query + output per repo
- [ ] Ran security check (no hardcoded secrets)
- [ ] Verified `.gitignore` is complete for all repos
- [ ] Created shared data dictionary
- [ ] Pushed all changes to GitHub
- [ ] Verified all repos look professional on GitHub

---

*Day 51 complete! Tomorrow: Docker Basics — containerize your world.* 🐳
