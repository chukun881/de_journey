# 📅 Day 72 — Friday, 25 July 2026
# ⚙️ GitHub Actions CI/CD: Auto-test Your Pipelines

---

## 🎯 Today's Goal

CI/CD (Continuous Integration / Continuous Deployment) means: every time you push code, tests run automatically. If something breaks, you know immediately — not 2 weeks later when your boss asks why the dashboard shows wrong numbers. Today you set up GitHub Actions for your data projects.

**Philosophy:** Manual testing doesn't scale. You tested your MakanExpress pipeline by running it yourself. But what if someone changes a SQL query and breaks the pipeline? Without CI, you find out when it fails in production. With CI, you find out in 2 minutes.

---

## ☀️ Morning Block (2 hours): GitHub Actions Fundamentals

### What is CI/CD?

- **CI (Continuous Integration):** Automatically test code when it's pushed
- **CD (Continuous Deployment):** Automatically deploy code when tests pass

For data engineers, CI means:
- SQL queries get linted (checked for style)
- dbt models get tested (do they return expected results?)
- Python scripts get tested (do they handle errors?)
- Data quality checks run on sample data

### GitHub Actions Architecture

```
You push code to GitHub
    → GitHub detects a workflow trigger (push, PR, schedule)
    → GitHub spins up a runner (virtual machine)
    → Workflow runs your steps (install, test, lint)
    → You get a ✅ or ❌ on your commit
```

### Your First Workflow

Create `.github/workflows/test.yml` in any repo:

```yaml
name: Python Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest
    
    - name: Run tests
      run: pytest tests/
```

### Understanding the YAML

| Key | What it does |
|-----|-------------|
| `name` | Workflow name (shows in GitHub UI) |
| `on` | When to trigger (push, PR, schedule, manual) |
| `jobs` | Groups of steps that run |
| `runs-on` | OS for the runner (ubuntu-latest = Linux) |
| `steps` | Individual actions in order |
| `uses` | Use a pre-built action from marketplace |
| `run` | Run a shell command |

### Exercise 1: Create Your First Workflow

1. Pick any of your repos (e.g., `crypto-data-pipeline`)
2. Create the file structure:
```
crypto-data-pipeline/
├── .github/
│   └── workflows/
│       └── test.yml
├── tests/
│   └── test_etl.py
├── requirements.txt
└── ...
```

3. Write a simple test in `tests/test_etl.py`:
```python
def test_import():
    """Test that main modules can be imported."""
    import sys
    import os
    # Add project root to path
    sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))
    # This should not raise an error
    assert True

def test_data_processing():
    """Test basic data transformation logic."""
    # Simulate processing
    raw_data = [
        {"name": "Bitcoin", "price": 50000},
        {"name": "Ethereum", "price": 3000},
    ]
    processed = [{**d, "price_usd": float(d["price"])} for d in raw_data]
    
    assert len(processed) == 2
    assert processed[0]["price_usd"] == 50000.0
    assert processed[1]["name"] == "Ethereum"

def test_empty_data_handling():
    """Test that empty data is handled gracefully."""
    raw_data = []
    processed = [{**d, "price_usd": float(d["price"])} for d in raw_data]
    assert processed == []
```

4. Push to GitHub and watch the Actions tab!

<details>
<summary>🔑 What to expect</summary>

1. Go to your repo on GitHub
2. Click the "Actions" tab
3. You'll see a workflow run triggered by your push
4. Click it to see live logs of each step
5. If everything passes: green ✅
6. The yellow dot turns green in ~1-2 minutes

If it fails, read the logs — they show exactly which step failed and why. Common issues:
- Missing dependency in requirements.txt
- Python version mismatch
- Import errors (module not found)
</details>

---

## 🌤️ Afternoon Block (2 hours): CI for dbt + Python Projects

### SQL Linting with sqlfluff

```yaml
name: SQL Lint

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
    
    - name: Install sqlfluff
      run: pip install sqlfluff sqlfluff-templater-dbt
    
    - name: Lint SQL files
      run: sqlfluff lint models/ --dialect postgres
```

### dbt Test Pipeline

For your `food-delivery-dbt` project:

```yaml
name: dbt CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  dbt-test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
    
    - name: Install dbt
      run: |
        pip install dbt-postgres
        pip install -r requirements.txt
    
    - name: Configure dbt profiles
      run: |
        mkdir -p ~/.dbt
        cat > ~/.dbt/profiles.yml << EOF
        test_project:
          target: ci
          outputs:
            ci:
              type: postgres
              host: localhost
              user: test_user
              password: test_pass
              port: 5432
              dbname: test_db
              schema: public
              threads: 4
        EOF
    
    - name: dbt debug
      run: dbt debug
    
    - name: dbt compile
      run: dbt compile
    
    - name: dbt test
      run: dbt test
```

### Python ETL CI Pipeline

For your `crypto-data-pipeline` or `makanexpress-serverless`:

```yaml
name: ETL CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install pytest black flake8
    
    - name: Lint with flake8
      run: flake8 src/ --max-line-length=120 --exclude=__pycache__
    
    - name: Format check with black
      run: black --check src/ tests/
    
    - name: Run tests
      run: pytest tests/ -v --tb=short
    
    - name: Run ETL dry-run
      env:
        DB_HOST: localhost
        DB_NAME: test
      run: python src/main.py --dry-run
```

### Exercise 2: Set Up CI for One Project

Pick your most polished repo and add:
1. A test workflow (`.github/workflows/test.yml`)
2. At least 3 test functions
3. Push and verify it runs

<details>
<summary>🔑 Tips for success</summary>

- Start simple: just `pytest` first, add linting later
- Use `pytest -v` for verbose output (see each test name)
- If a test fails in CI but passes locally, check:
  - Python version matches
  - All dependencies in requirements.txt
  - File paths are correct (CI runs on Linux, paths are case-sensitive)
- Use `--dry-run` flag for ETL scripts to avoid actually calling APIs in CI
</details>

### Exercise 3: Add a Status Badge

Add this to your README.md:

```markdown
![Tests](https://github.com/YOUR_USERNAME/YOUR_REPO/actions/workflows/test.yml/badge.svg)
```

This shows a green "passing" or red "failing" badge on your repo homepage. Recruiters love this — it signals professional engineering practices.

---

## 🌙 Evening (1 hour): Scheduled Jobs + Quick Reference

### Scheduled Workflows (Cron in CI)

```yaml
on:
  schedule:
    - cron: '0 6 * * *'  # Run daily at 6 AM UTC (2 PM SGT)
  workflow_dispatch:       # Allow manual trigger
```

**Use case:** Run data quality checks every night. If something breaks, you get an email before you start work.

### Cache Dependencies for Speed

```yaml
- name: Cache pip packages
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

### 🗺️ Quick Reference Card

```
=== WORKFLOW TRIGGERS ===
on: push                          # Every push
on: [push, pull_request]          # Push and PR
on: schedule: cron: '0 6 * * *'   # Daily at 6 AM UTC
on: workflow_dispatch              # Manual trigger button

=== COMMON PATTERNS ===
uses: actions/checkout@v4         # Clone your repo
uses: actions/setup-python@v5     # Install Python
run: pip install -r requirements.txt
run: pytest tests/ -v
run: black --check src/
run: sqlfluff lint models/

=== SERVICES (databases in CI) ===
services:
  postgres:
    image: postgres:15
    env:
      POSTGRES_PASSWORD: test
    ports: ['5432:5432']

=== SECRETS (API keys) ===
env:
  API_KEY: ${{ secrets.API_KEY }}  # Set in repo Settings > Secrets
```

### 📝 Today's Checklist

- [ ] Created first GitHub Actions workflow
- [ ] Understood YAML structure (on, jobs, steps)
- [ ] Wrote Python tests that pass in CI
- [ ] Added SQL linting (sqlfluff) to workflow
- [ ] Set up dbt test pipeline (with PostgreSQL service)
- [ ] Added status badge to README
- [ ] Pushed and verified CI runs successfully
- [ ] Read about scheduled workflows

---

*Day 72 complete! Tomorrow: Professional repo setup — branch protection, PR templates, and contributing guides.* 🏢
