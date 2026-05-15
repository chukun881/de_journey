# 📅 Day 14 — Wednesday, 28 May 2026
# 🚀 PORTFOLIO PROJECT 2: Automated Data Pipeline

---

## 🎯 Today's Big Picture

Today you build your **second portfolio project** — a complete automated data pipeline using Python, Pandas, and SQL.

**What you'll build:**
A Python script that fetches data from a public API, cleans it with Pandas, transforms it, and loads it into a PostgreSQL database (or CSV if you prefer). This is a REAL ETL pipeline.

**Why this matters:** This project demonstrates:
- API data extraction (fetching real data)
- Data cleaning with Pandas
- Database loading with SQL
- Error handling
- Logging
- Project documentation

This is exactly what employers ask for: "Show me a data pipeline you built."

---

## ☀️ BLOCK 1: Project Setup (Morning, ~30 min)

---

### Task 1: Create the Project Repo (15 min)

```bash
# New repo for this project
mkdir crypto-data-pipeline
cd crypto-data-pipeline
git init

mkdir -p src data/raw data/processed logs
touch README.md
touch src/extract.py
touch src/transform.py
touch src/load.py
touch src/pipeline.py
touch src/config.py
touch requirements.txt
touch .gitignore
```

### Task 2: Project Structure (15 min)

```
crypto-data-pipeline/
├── README.md
├── requirements.txt
├── .gitignore
├── src/
│   ├── config.py        # Configuration (API URLs, DB connection)
│   ├── extract.py       # Fetch data from API
│   ├── transform.py     # Clean and transform with Pandas
│   ├── load.py          # Load to database/CSV
│   └── pipeline.py      # Main script that runs everything
├── data/
│   ├── raw/             # Original data from API
│   └── processed/       # Cleaned data
└── logs/                # Execution logs
```

Write `.gitignore`:
```
data/raw/*
data/processed/*
logs/*
__pycache__/
*.pyc
.env
```

Write `requirements.txt`:
```
pandas>=2.0
requests>=2.31
psycopg2-binary>=2.9
```

```bash
pip install pandas requests psycopg2-binary
```

---

## 🔥 BLOCK 2: Build the Pipeline (Morning + Afternoon, ~4 hours)

---

### Task 3: Config File (10 min)

```python
# src/config.py

# API Configuration
API_BASE_URL = "https://api.coingecko.com/api/v3"
COINS = ["bitcoin", "ethereum", "solana", "cardano", "polkadot",
         "chainlink", "avalanche-2", "polygon-pos", "near", "algorand"]
VS_CURRENCY = "usd"

# Database Configuration (Neon or local PostgreSQL)
DB_CONFIG = {
    "host": "localhost",       # or your Neon host
    "database": "neondb",      # or your local db name
    "user": "postgres",
    "password": "your_password",
    "port": 5432,
}

# File paths
RAW_DATA_PATH = "data/raw"
PROCESSED_DATA_PATH = "data/processed"
```

---

### Task 4: EXTRACT — Fetch Data from API (45 min)

```python
# src/extract.py

import requests
import json
import logging
from datetime import datetime
from pathlib import Path
from config import API_BASE_URL, COINS, VS_CURRENCY, RAW_DATA_PATH

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler(f"logs/extract_{datetime.now().strftime('%Y%m%d')}.log"),
        logging.StreamHandler(),
    ]
)

def fetch_coin_data(coin_ids):
    """Fetch current price data for given coins from CoinGecko API.
    
    Args:
        coin_ids: list of coin identifier strings
        
    Returns:
        list of dicts with coin data, or None if failed
    """
    url = f"{API_BASE_URL}/coins/markets"
    params = {
        "vs_currency": VS_CURRENCY,
        "ids": ",".join(coin_ids),
        "order": "market_cap_desc",
        "per_page": len(coin_ids),
        "page": 1,
        "sparkline": "false",
        "price_change_percentage": "1h,24h,7d",
    }
    
    try:
        logging.info(f"Fetching data for {len(coin_ids)} coins...")
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()  # raise error for bad status codes
        
        data = response.json()
        logging.info(f"Successfully fetched data for {len(data)} coins")
        return data
        
    except requests.exceptions.Timeout:
        logging.error("API request timed out")
        return None
    except requests.exceptions.ConnectionError:
        logging.error("Cannot connect to API")
        return None
    except requests.exceptions.HTTPError as e:
        logging.error(f"HTTP error: {e}")
        return None
    except json.JSONDecodeError:
        logging.error("Invalid JSON response")
        return None


def save_raw_data(data):
    """Save raw API response to JSON file with timestamp."""
    if not data:
        logging.warning("No data to save")
        return None
    
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filepath = Path(RAW_DATA_PATH) / f"crypto_raw_{timestamp}.json"
    
    filepath.parent.mkdir(parents=True, exist_ok=True)
    
    with open(filepath, "w") as f:
        json.dump(data, f, indent=2)
    
    logging.info(f"Raw data saved to {filepath}")
    return filepath


if __name__ == "__main__":
    # Test extraction
    data = fetch_coin_data(COINS)
    if data:
        save_raw_data(data)
        print(f"Fetched {len(data)} coins")
        for coin in data:
            print(f"  {coin['name']}: ${coin['current_price']:,.2f}")
```

Test it:
```bash
cd crypto-data-pipeline
python src/extract.py
```

✅ You should see crypto prices printed and a JSON file saved in `data/raw/`.

---

### Task 5: TRANSFORM — Clean with Pandas (45 min)

```python
# src/transform.py

import pandas as pd
import json
import logging
from datetime import datetime
from pathlib import Path
from config import PROCESSED_DATA_PATH

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
)

def load_raw_data(filepath):
    """Load raw JSON data from file."""
    with open(filepath, "r") as f:
        data = json.load(f)
    logging.info(f"Loaded {len(data)} records from {filepath}")
    return data


def transform_data(raw_data):
    """Transform raw API data into clean analytical format.
    
    Steps:
    1. Convert to DataFrame
    2. Select relevant columns
    3. Clean and rename columns
    4. Handle missing values
    5. Calculate derived fields
    6. Validate data quality
    
    Returns:
        Clean DataFrame
    """
    # Step 1: Convert to DataFrame
    df = pd.DataFrame(raw_data)
    logging.info(f"Initial shape: {df.shape}")
    
    # Step 2: Select relevant columns
    columns_to_keep = [
        "id", "symbol", "name", "current_price", "market_cap",
        "total_volume", "high_24h", "low_24h", "price_change_24h",
        "price_change_percentage_24h", "market_cap_rank",
        "circulating_supply", "total_supply", "max_supply",
        "ath", "ath_date", "atl", "atl_date",
        "last_updated",
    ]
    # Only keep columns that exist
    available = [c for c in columns_to_keep if c in df.columns]
    df = df[available]
    
    # Step 3: Rename columns for clarity
    df = df.rename(columns={
        "id": "coin_id",
        "current_price": "price_usd",
        "market_cap": "market_cap_usd",
        "total_volume": "volume_24h_usd",
        "high_24h": "high_24h_usd",
        "low_24h": "low_24h_usd",
        "price_change_24h": "price_change_24h_usd",
        "price_change_percentage_24h": "price_change_pct_24h",
        "ath": "all_time_high_usd",
        "atl": "all_time_low_usd",
    })
    
    # Step 4: Clean data types
    # Parse dates
    if "last_updated" in df.columns:
        df["last_updated"] = pd.to_datetime(df["last_updated"])
    if "ath_date" in df.columns:
        df["ath_date"] = pd.to_datetime(df["ath_date"])
    if "atl_date" in df.columns:
        df["atl_date"] = pd.to_datetime(df["atl_date"])
    
    # Symbol to uppercase
    df["symbol"] = df["symbol"].str.upper()
    
    # Step 5: Calculate derived fields
    df["price_range_24h"] = df["high_24h_usd"] - df["low_24h_usd"]
    df["pct_from_ath"] = ((df["price_usd"] - df["all_time_high_usd"]) 
                          / df["all_time_high_usd"] * 100).round(2)
    df["volume_to_mcap_ratio"] = (df["volume_24h_usd"] / df["market_cap_usd"]).round(4)
    
    # Add fetch timestamp
    df["fetch_timestamp"] = datetime.now()
    df["fetch_date"] = datetime.now().strftime("%Y-%m-%d")
    
    # Step 6: Handle missing values
    numeric_cols = df.select_dtypes(include=["float64", "int64"]).columns
    for col in numeric_cols:
        null_count = df[col].isnull().sum()
        if null_count > 0:
            logging.warning(f"Column '{col}' has {null_count} null values")
    
    # Step 7: Data quality checks
    assert df["coin_id"].nunique() == len(df), "Duplicate coin IDs!"
    assert (df["price_usd"] > 0).all(), "Negative prices found!"
    
    logging.info(f"Transformed shape: {df.shape}")
    logging.info(f"Columns: {list(df.columns)}")
    
    return df


def generate_summary(df):
    """Generate summary statistics from transformed data."""
    summary = {
        "total_coins": len(df),
        "fetch_time": datetime.now().isoformat(),
        "total_market_cap": df["market_cap_usd"].sum(),
        "top_coin": df.loc[df["market_cap_usd"].idxmax(), "name"],
        "biggest_gainer": df.loc[df["price_change_pct_24h"].idxmax(), "name"],
        "biggest_loser": df.loc[df["price_change_pct_24h"].idxmin(), "name"],
        "avg_price_change_pct": round(df["price_change_pct_24h"].mean(), 2),
    }
    return summary


def save_processed_data(df):
    """Save processed data to CSV and JSON."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    
    Path(PROCESSED_DATA_PATH).mkdir(parents=True, exist_ok=True)
    
    csv_path = Path(PROCESSED_DATA_PATH) / f"crypto_processed_{timestamp}.csv"
    df.to_csv(csv_path, index=False)
    logging.info(f"Saved CSV: {csv_path}")
    
    # Also save as "latest" for easy access
    latest_path = Path(PROCESSED_DATA_PATH) / "crypto_latest.csv"
    df.to_csv(latest_path, index=False)
    
    return csv_path


if __name__ == "__main__":
    # Test with the most recent raw file
    raw_dir = Path("data/raw")
    raw_files = sorted(raw_dir.glob("crypto_raw_*.json"))
    
    if raw_files:
        latest = raw_files[-1]
        print(f"Processing: {latest}")
        
        raw = load_raw_data(latest)
        df = transform_data(raw)
        
        print("\n=== Transformed Data ===")
        print(df[["name", "symbol", "price_usd", "price_change_pct_24h", "market_cap_usd"]])
        
        summary = generate_summary(df)
        print("\n=== Summary ===")
        for key, value in summary.items():
            print(f"  {key}: {value}")
        
        save_processed_data(df)
    else:
        print("No raw data found. Run extract.py first.")
```

Test it:
```bash
python src/transform.py
```

✅ You should see clean, transformed crypto data.

---

### Task 6: LOAD — Save to Database (30 min)

```python
# src/load.py

import pandas as pd
import logging
from datetime import datetime
from config import DB_CONFIG

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
)

def create_table_if_not_exists(engine):
    """Create the crypto_prices table if it doesn't exist."""
    create_sql = """
    CREATE TABLE IF NOT EXISTS crypto_prices (
        id SERIAL PRIMARY KEY,
        coin_id VARCHAR(50) NOT NULL,
        symbol VARCHAR(20),
        name VARCHAR(100),
        price_usd DECIMAL(20,8),
        market_cap_usd BIGINT,
        volume_24h_usd BIGINT,
        high_24h_usd DECIMAL(20,8),
        low_24h_usd DECIMAL(20,8),
        price_change_24h_usd DECIMAL(20,8),
        price_change_pct_24h DECIMAL(10,4),
        market_cap_rank INTEGER,
        circulating_supply DECIMAL(20,4),
        total_supply DECIMAL(20,4),
        max_supply DECIMAL(20,4),
        all_time_high_usd DECIMAL(20,8),
        all_time_low_usd DECIMAL(20,8),
        price_range_24h DECIMAL(20,8),
        pct_from_ath DECIMAL(10,4),
        volume_to_mcap_ratio DECIMAL(10,6),
        fetch_timestamp TIMESTAMPTZ,
        fetch_date DATE,
        created_at TIMESTAMPTZ DEFAULT NOW()
    );
    """
    with engine.connect() as conn:
        conn.execute(create_sql)
        conn.commit()
    logging.info("Table crypto_prices ready")


def load_to_database(df, db_config):
    """Load DataFrame to PostgreSQL database.
    
    If database is unavailable, fall back to CSV.
    """
    try:
        from sqlalchemy import create_engine, text
        
        db_url = (f"postgresql://{db_config['user']}:{db_config['password']}"
                  f"@{db_config['host']}:{db_config['port']}/{db_config['database']}")
        engine = create_engine(db_url)
        
        # Create table
        create_table_if_not_exists(engine)
        
        # Insert data
        df.to_sql("crypto_prices", engine, if_exists="append", index=False)
        
        row_count = len(df)
        logging.info(f"✅ Loaded {row_count} rows to database")
        return True
        
    except Exception as e:
        logging.warning(f"Database unavailable: {e}")
        logging.info("Falling back to CSV...")
        return False


def load_to_csv(df, filepath="data/processed/crypto_final.csv"):
    """Save to CSV as fallback or primary output."""
    df.to_csv(filepath, index=False)
    logging.info(f"✅ Saved {len(df)} rows to {filepath}")
    return filepath


def load_data(df):
    """Try database first, fall back to CSV."""
    success = load_to_database(df, DB_CONFIG)
    if not success:
        load_to_csv(df)
    
    # Always also save a CSV backup
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    load_to_csv(df, f"data/processed/crypto_backup_{timestamp}.csv")


if __name__ == "__main__":
    # Test with latest processed file
    from pathlib import Path
    processed = Path("data/processed/crypto_latest.csv")
    
    if processed.exists():
        df = pd.read_csv(processed)
        load_data(df)
    else:
        print("No processed data found. Run transform.py first.")
```

> 💡 **Note on SQLAlchemy:** You may need to install it: `pip install sqlalchemy`. If connecting to Neon gives trouble, that's OK — the CSV fallback works fine. You can fix the database connection later.

---

### Task 7: PIPELINE — Tie It All Together (20 min)

```python
# src/pipeline.py
"""
Crypto Data Pipeline — Main Orchestrator

Runs the full ETL pipeline:
1. EXTRACT: Fetch crypto data from CoinGecko API
2. TRANSFORM: Clean and enrich with Pandas
3. LOAD: Save to database (PostgreSQL) and CSV

Usage:
    python src/pipeline.py
"""

import sys
import logging
from datetime import datetime
from pathlib import Path

from extract import fetch_coin_data, save_raw_data
from transform import load_raw_data, transform_data, generate_summary, save_processed_data
from load import load_data
from config import COINS

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler(f"logs/pipeline_{datetime.now().strftime('%Y%m%d')}.log"),
        logging.StreamHandler(),
    ]
)

def run_pipeline():
    """Execute the full ETL pipeline."""
    start_time = datetime.now()
    logging.info("=" * 50)
    logging.info("PIPELINE STARTED")
    logging.info("=" * 50)
    
    # === EXTRACT ===
    logging.info("Phase 1: EXTRACT")
    raw_data = fetch_coin_data(COINS)
    
    if not raw_data:
        logging.error("Extract failed. Aborting pipeline.")
        sys.exit(1)
    
    raw_path = save_raw_data(raw_data)
    logging.info(f"Extract complete: {len(raw_data)} records from {raw_path}")
    
    # === TRANSFORM ===
    logging.info("Phase 2: TRANSFORM")
    raw = load_raw_data(raw_path)
    df = transform_data(raw)
    summary = generate_summary(df)
    
    logging.info("Summary:")
    for key, value in summary.items():
        logging.info(f"  {key}: {value}")
    
    processed_path = save_processed_data(df)
    
    # === LOAD ===
    logging.info("Phase 3: LOAD")
    load_data(df)
    
    # === DONE ===
    duration = (datetime.now() - start_time).total_seconds()
    logging.info("=" * 50)
    logging.info(f"PIPELINE COMPLETE in {duration:.1f} seconds")
    logging.info(f"Records processed: {len(df)}")
    logging.info("=" * 50)
    
    # Print a nice summary
    print("\n" + "=" * 50)
    print("📊 PIPELINE SUMMARY")
    print("=" * 50)
    print(f"Coins tracked: {summary['total_coins']}")
    print(f"Top coin: {summary['top_coin']}")
    print(f"Biggest gainer: {summary['biggest_gainer']}")
    print(f"Biggest loser: {summary['biggest_loser']}")
    print(f"Avg 24h change: {summary['avg_price_change_pct']}%")
    print(f"Duration: {duration:.1f}s")
    print("=" * 50)


if __name__ == "__main__":
    run_pipeline()
```

Test the full pipeline:
```bash
python src/pipeline.py
```

✅ You should see the full ETL run: extract → transform → load with logs.

---

## 🌙 BLOCK 3: Documentation + Polish (Evening, ~1.5 hours)

---

### Task 8: Write Professional README.md (30 min)

```markdown
# Crypto Data Pipeline 🪙

Automated ETL pipeline that fetches cryptocurrency data from CoinGecko API, 
transforms it with Pandas, and loads it into PostgreSQL.

## Architecture

```
CoinGecko API → Python (Extract) → Pandas (Transform) → PostgreSQL (Load)
                                      ↓
                                   CSV Backup
```

## Features
- **Extract**: Fetches real-time data for 10 cryptocurrencies via REST API
- **Transform**: Cleans, validates, and enriches data with calculated fields
- **Load**: Inserts into PostgreSQL with automatic table creation; CSV fallback
- **Error handling**: Graceful handling of API failures, DB unavailability
- **Logging**: Full execution logs with timestamps
- **Data quality**: Assertions and null-value checks

## Pipeline Steps

### 1. Extract (`src/extract.py`)
- Calls CoinGecko `/coins/markets` endpoint
- Fetches price, volume, market cap, 24h changes
- Saves raw JSON response with timestamp

### 2. Transform (`src/transform.py`)
- Converts JSON to Pandas DataFrame
- Selects and renames 19 relevant columns
- Parses dates, cleans strings
- Calculates derived fields:
  - `price_range_24h` = high - low
  - `pct_from_ath` = distance from all-time high
  - `volume_to_mcap_ratio` = trading activity ratio
- Validates: no duplicates, no negative prices

### 3. Load (`src/load.py`)
- Creates PostgreSQL table if not exists
- Inserts records with `df.to_sql()`
- Falls back to CSV if database unavailable

## Quick Start

```bash
pip install -r requirements.txt

# Run the pipeline
python src/pipeline.py

# Or run individual steps
python src/extract.py
python src/transform.py
python src/load.py
```

## Sample Output

| coin_id | name | price_usd | price_change_pct_24h | market_cap_usd |
|---------|------|-----------|---------------------|----------------|
| bitcoin | Bitcoin | 67,234.50 | +2.34 | 1,320,000,000,000 |
| ethereum | Ethereum | 3,456.78 | -1.12 | 415,000,000,000 |

## Tech Stack
- **Python 3.12**
- **Pandas** — data transformation
- **Requests** — API calls
- **PostgreSQL** — data storage (via SQLAlchemy)
- **CoinGecko API** — free data source

## What I Learned
- REST API integration with error handling
- ETL pipeline design patterns
- Data quality validation
- Graceful degradation (DB fallback to CSV)
- Project structure for data engineering
```

---

### Task 9: Final Push (10 min)

```bash
cd crypto-data-pipeline
git add .
git commit -m "Initial commit: crypto data pipeline with ETL (extract, transform, load)"
git push
```

---

### Task 10: Week 2 Reflection (20 min)

In your learning repo:

```markdown
# Day 14 — Portfolio Project 2 Complete
Date: 2026-05-28

## Project: crypto-data-pipeline
- GitHub: [link]
- Fetches real crypto data from API
- Cleans with Pandas (19 columns, 3 derived fields)
- Loads to PostgreSQL with CSV fallback
- Full logging and error handling

## Week 2 Summary
- Day 8: Python basics (variables, types, strings, numbers)
- Day 9: Control flow (if/else, loops, functions, try/except)
- Day 10: Data structures (lists, dicts, sets, tuples)
- Day 11: File I/O (CSV, JSON, text files)
- Day 12: Pandas basics (DataFrames, filtering, groupby)
- Day 13: Pandas advanced (merge, pivot, apply, datetime, cleaning)
- Day 14: Portfolio Project 2 — automated data pipeline

## Skills Gained
✅ Python fundamentals
✅ Pandas for data manipulation
✅ CSV and JSON file handling
✅ API data extraction
✅ ETL pipeline design
✅ Error handling and logging
✅ Data cleaning techniques

## Portfolio Status
1. Retail Sales Analysis (SQL) ✅
2. Crypto Data Pipeline (Python ETL) ✅
3. dbt Analytics Project — Week 3 🔜
```

```bash
cd data-engineering-journey
git add .
git commit -m "Day 14: Portfolio Project 2 complete — crypto ETL pipeline. Week 2 done!"
git push
```

---

## ✅ Day 14 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Created project repo with proper structure | ☐ |
| 2 | Built extract module (API fetch + error handling) | ☐ |
| 3 | Built transform module (Pandas cleaning + validation) | ☐ |
| 4 | Built load module (PostgreSQL + CSV fallback) | ☐ |
| 5 | Built pipeline orchestrator | ☐ |
| 6 | Pipeline runs end-to-end successfully | ☐ |
| 7 | Professional README written | ☐ |
| 8 | requirements.txt created | ☐ |
| 9 | Pushed to GitHub | ☐ |
| 10 | Week 2 reflection written | ☐ |

---

## 🏁 WEEK 2 CHECKPOINT — Are We On Track?

### Skills gained vs. job requirements:

| Job Requirement | Status | Evidence |
|----------------|--------|----------|
| SQL proficiency | ✅ Week 1 | 200+ queries, portfolio project |
| Python | ✅ Week 2 | Scripts, functions, error handling |
| Pandas | ✅ Week 2 | Data cleaning, merge, aggregation |
| ETL pipeline building | ✅ Week 2 | Crypto pipeline project |
| API data extraction | ✅ Week 2 | CoinGecko integration |
| CSV/JSON handling | ✅ Week 2 | Read/write both formats |
| Git/GitHub | ✅ Ongoing | Daily commits, 3 repos |
| dbt | 🔜 Week 3 | — |
| Data modeling (star schema) | 🔜 Week 3 | — |
| Airflow | 🔜 Week 5 | — |
| Cloud (AWS) | 🔜 Week 6-8 | — |

### Week 3 Preview:
- **Day 15-16:** Data Modeling (star schema, fact/dimension tables)
- **Day 17-19:** dbt fundamentals (models, tests, documentation)
- **Day 20-21:** Portfolio Project 3: dbt Analytics Project

**We are ON TRACK.** Python + SQL + ETL is the core skillset. Week 3 adds dbt and data modeling, which are the next most requested skills in SG/MY job postings.
