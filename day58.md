# 📅 Day 58 — Friday, 11 July 2026
# 🔄 Build an API Data Pipeline: Extract → Transform → Load

---

## 🎯 Today's Goal

Yesterday you learned to call APIs. Today you build a **complete data pipeline** that:
1. **Extracts** data from multiple APIs (weather + crypto)
2. **Transforms** it (clean, normalize, join)
3. **Loads** it into PostgreSQL and S3

This is the E in ETL — and it's the most common task in data engineering job descriptions.

**By end of day:** You'll have a production-quality pipeline that fetches data from real APIs, transforms it, and loads it into both a database and a data lake.

---

## ☀️ Morning Block (2 hours): Pipeline Architecture + Extract

### Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    API DATA PIPELINE                         │
│                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ OpenMeteo │    │CoinGecko │    │ More APIs│              │
│  │ (Weather) │    │ (Crypto) │    │ (future) │              │
│  └─────┬─────┘    └────┬─────┘    └────┬─────┘              │
│        │               │               │                     │
│        ▼               ▼               ▼                     │
│  ┌─────────────────────────────────────────┐                │
│  │           EXTRACT LAYER                  │                │
│  │  • API calls with error handling         │                │
│  │  • Rate limiting + retry                 │                │
│  │  • Raw JSON stored as backup             │                │
│  └─────────────────┬───────────────────────┘                │
│                    │                                         │
│                    ▼                                         │
│  ┌─────────────────────────────────────────┐                │
│  │           TRANSFORM LAYER                │                │
│  │  • Clean data (nulls, types, units)      │                │
│  │  • Normalize (common format)             │                │
│  │  • Join data sources                     │                │
│  │  • Add metadata (timestamp, source)      │                │
│  └─────────────────┬───────────────────────┘                │
│                    │                                         │
│           ┌────────┴────────┐                               │
│           ▼                 ▼                                │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │  PostgreSQL   │  │  S3 (Parquet) │                        │
│  │  (structured  │  │  (data lake)  │                        │
│  │   queries)    │  │              │                        │
│  └──────────────┘  └──────────────┘                        │
└──────────────────────────────────────────────────────────────┘
```

---

### Step 1: Project Setup

```bash
mkdir api-data-pipeline
cd api-data-pipeline
mkdir -p src/{extract,transform,load} data/{raw,processed} logs tests
touch src/__init__.py src/extract/__init__.py src/transform/__init__.py src/load/__init__.py
touch requirements.txt config.py run_pipeline.py
```

**requirements.txt:**
```
requests>=2.31.0
psycopg2-binary>=2.9.0
boto3>=1.28.0
pandas>=2.0.0
python-dotenv>=1.0.0
```

**config.py:**
```python
import os
from dotenv import load_dotenv

load_dotenv()

# API Configuration
OPENMETEO_BASE_URL = "https://api.open-meteo.com/v1/forecast"
COINGECKO_BASE_URL = "https://api.coingecko.com/api/v3"

# Cities to track
CITIES = {
    "Singapore": {"lat": 1.3521, "lon": 103.8198, "timezone": "Asia/Singapore"},
    "Kuala Lumpur": {"lat": 3.1390, "lon": 101.6869, "timezone": "Asia/Kuala_Lumpur"},
    "Penang": {"lat": 5.4164, "lon": 100.3327, "timezone": "Asia/Kuala_Lumpur"},
    "Johor Bahru": {"lat": 1.4927, "lon": 103.7414, "timezone": "Asia/Singapore"},
}

# Crypto to track
CRYPTO_IDS = ["bitcoin", "ethereum", "solana", "cardano", "polkadot"]
CURRENCY = "sgd"

# Database
DB_HOST = os.getenv("DB_HOST", "localhost")
DB_PORT = os.getenv("DB_PORT", "5432")
DB_NAME = os.getenv("DB_NAME", "makanexpress")
DB_USER = os.getenv("DB_USER", "postgres")
DB_PASSWORD = os.getenv("DB_PASSWORD", "postgres")

# S3
S3_BUCKET = os.getenv("S3_BUCKET", "makanexpress-data-lake-dev")
S3_REGION = os.getenv("S3_REGION", "ap-southeast-1")
S3_PREFIX = os.getenv("S3_PREFIX", "api-pipeline")

# Rate limiting
API_CALLS_PER_SECOND = 1
MAX_RETRIES = 3
```

---

### Step 2: Extract — Weather Data

```python
# src/extract/weather.py

import requests
import logging
import time
from datetime import datetime
from config import OPENMETEO_BASE_URL, CITIES, API_CALLS_PER_SECOND

logger = logging.getLogger(__name__)

class WeatherExtractor:
    """Extract weather data from OpenMeteo API."""
    
    def __init__(self):
        self.base_url = OPENMETEO_BASE_URL
        self._last_request = 0
    
    def _rate_limit(self):
        min_interval = 1.0 / API_CALLS_PER_SECOND
        elapsed = time.time() - self._last_request
        if elapsed < min_interval:
            time.sleep(min_interval - elapsed)
    
    def fetch_current_weather(self, city_name, lat, lon, timezone="auto"):
        """Fetch current weather for a single city."""
        self._rate_limit()
        
        params = {
            "latitude": lat,
            "longitude": lon,
            "current": "temperature_2m,relative_humidity_2m,wind_speed_10m,precipitation",
            "daily": "temperature_2m_max,temperature_2m_min,precipitation_sum",
            "timezone": timezone,
            "forecast_days": 1
        }
        
        try:
            response = requests.get(self.base_url, params=params, timeout=10)
            response.raise_for_status()
            data = response.json()
            self._last_request = time.time()
            
            # Normalize into flat structure
            current = data.get("current", {})
            daily = data.get("daily", {})
            
            return {
                "city": city_name,
                "temperature_c": current.get("temperature_2m"),
                "humidity_pct": current.get("relative_humidity_2m"),
                "wind_speed_kmh": current.get("wind_speed_10m"),
                "precipitation_mm": current.get("precipitation"),
                "temp_max_c": daily.get("temperature_2m_max", [None])[0],
                "temp_min_c": daily.get("temperature_2m_min", [None])[0],
                "daily_precipitation_mm": daily.get("precipitation_sum", [None])[0],
                "fetched_at": datetime.now().isoformat(),
                "source": "openmeteo",
                "status": "success"
            }
            
        except requests.exceptions.RequestException as e:
            logger.error(f"Failed to fetch weather for {city_name}: {e}")
            return {
                "city": city_name,
                "temperature_c": None,
                "humidity_pct": None,
                "wind_speed_kmh": None,
                "precipitation_mm": None,
                "temp_max_c": None,
                "temp_min_c": None,
                "daily_precipitation_mm": None,
                "fetched_at": datetime.now().isoformat(),
                "source": "openmeteo",
                "status": f"error: {str(e)}"
            }
    
    def fetch_all_cities(self):
        """Fetch weather for all configured cities."""
        results = []
        for city_name, coords in CITIES.items():
            logger.info(f"Fetching weather for {city_name}...")
            weather = self.fetch_current_weather(
                city_name=city_name,
                lat=coords["lat"],
                lon=coords["lon"],
                timezone=coords.get("timezone", "auto")
            )
            results.append(weather)
            logger.info(f"  → {city_name}: {weather.get('temperature_c', 'N/A')}°C")
        
        logger.info(f"Fetched weather for {len(results)} cities")
        return results
```

---

### Step 3: Extract — Crypto Data

```python
# src/extract/crypto.py

import requests
import logging
import time
from datetime import datetime
from config import COINGECKO_BASE_URL, CRYPTO_IDS, CURRENCY, API_CALLS_PER_SECOND

logger = logging.getLogger(__name__)

class CryptoExtractor:
    """Extract cryptocurrency data from CoinGecko API."""
    
    def __init__(self):
        self.base_url = COINGECKO_BASE_URL
        self._last_request = 0
    
    def _rate_limit(self):
        min_interval = 1.0 / API_CALLS_PER_SECOND
        elapsed = time.time() - self._last_request
        if elapsed < min_interval:
            time.sleep(min_interval - elapsed)
    
    def fetch_prices(self):
        """Fetch current prices for tracked cryptocurrencies."""
        self._rate_limit()
        
        url = f"{self.base_url}/simple/price"
        params = {
            "ids": ",".join(CRYPTO_IDS),
            "vs_currencies": f"{CURRENCY},usd",
            "include_24hr_change": "true",
            "include_market_cap": "true",
            "include_24hr_vol": "true"
        }
        
        try:
            response = requests.get(url, params=params, timeout=15)
            
            if response.status_code == 429:
                logger.warning("CoinGecko rate limited. Waiting 60s...")
                time.sleep(60)
                response = requests.get(url, params=params, timeout=15)
            
            response.raise_for_status()
            raw_data = response.json()
            self._last_request = time.time()
            
            # Normalize into list of records
            results = []
            for coin_id, coin_data in raw_data.items():
                results.append({
                    "coin_id": coin_id,
                    "price_sgd": coin_data.get(CURRENCY),
                    "price_usd": coin_data.get("usd"),
                    "market_cap_sgd": coin_data.get(f"{CURRENCY}_market_cap"),
                    "volume_24h_sgd": coin_data.get(f"{CURRENCY}_24h_vol"),
                    "change_24h_pct": coin_data.get(f"{CURRENCY}_24h_change"),
                    "fetched_at": datetime.now().isoformat(),
                    "source": "coingecko",
                    "status": "success"
                })
            
            logger.info(f"Fetched prices for {len(results)} cryptocurrencies")
            return results
            
        except requests.exceptions.RequestException as e:
            logger.error(f"Failed to fetch crypto prices: {e}")
            return []
    
    def fetch_market_data(self):
        """Fetch detailed market data (top coins)."""
        self._rate_limit()
        
        url = f"{self.base_url}/coins/markets"
        params = {
            "vs_currency": CURRENCY,
            "order": "market_cap_desc",
            "per_page": 20,
            "page": 1,
            "sparkline": "false"
        }
        
        try:
            response = requests.get(url, params=params, timeout=15)
            response.raise_for_status()
            data = response.json()
            self._last_request = time.time()
            
            results = []
            for coin in data:
                results.append({
                    "coin_id": coin["id"],
                    "name": coin["name"],
                    "symbol": coin["symbol"].upper(),
                    "rank": coin["market_cap_rank"],
                    "price_sgd": coin["current_price"],
                    "market_cap_sgd": coin["market_cap"],
                    "volume_24h_sgd": coin["total_volume"],
                    "change_24h_pct": coin.get("price_change_percentage_24h"),
                    "high_24h_sgd": coin.get("high_24h"),
                    "low_24h_sgd": coin.get("low_24h"),
                    "fetched_at": datetime.now().isoformat(),
                    "source": "coingecko",
                    "status": "success"
                })
            
            logger.info(f"Fetched market data for {len(results)} coins")
            return results
            
        except requests.exceptions.RequestException as e:
            logger.error(f"Failed to fetch market data: {e}")
            return []
```

---

### Morning Exercises

**Exercise 1:** Add a new extractor class `NewsExtractor` that fetches headlines from a free news API (use `https://hacker-news.firebaseio.com/v0/topstories.json` as a free alternative). Return the top 10 story IDs and titles.

<details>
<summary>🔑 Answer</summary>

```python
# src/extract/news.py

import requests
import logging
from datetime import datetime
from config import API_CALLS_PER_SECOND

logger = logging.getLogger(__name__)

class NewsExtractor:
    """Extract tech news from Hacker News API (free, no key needed)."""
    
    def __init__(self, max_stories=10):
        self.base_url = "https://hacker-news.firebaseio.com/v0"
        self.max_stories = max_stories
    
    def fetch_top_stories(self):
        """Fetch top story IDs, then get details for each."""
        try:
            # Get top story IDs
            response = requests.get(f"{self.base_url}/topstories.json", timeout=10)
            response.raise_for_status()
            story_ids = response.json()[:self.max_stories]
            
            stories = []
            for story_id in story_ids:
                try:
                    story_resp = requests.get(
                        f"{self.base_url}/item/{story_id}.json", 
                        timeout=10
                    )
                    story_resp.raise_for_status()
                    story = story_resp.json()
                    
                    if story and story.get("type") == "story":
                        stories.append({
                            "id": story["id"],
                            "title": story.get("title", "No title"),
                            "url": story.get("url", ""),
                            "score": story.get("score", 0),
                            "author": story.get("by", "unknown"),
                            "time": datetime.fromtimestamp(story.get("time", 0)).isoformat(),
                            "fetched_at": datetime.now().isoformat(),
                            "source": "hackernews",
                            "status": "success"
                        })
                except Exception as e:
                    logger.warning(f"Failed to fetch story {story_id}: {e}")
            
            logger.info(f"Fetched {len(stories)} stories")
            return stories
            
        except Exception as e:
            logger.error(f"Failed to fetch top stories: {e}")
            return []

# Test
if __name__ == "__main__":
    news = NewsExtractor(max_stories=10)
    stories = news.fetch_top_stories()
    for s in stories[:5]:
        print(f"[{s['score']}] {s['title']}")
```

</details>

**Exercise 2:** Modify the `WeatherExtractor` to also fetch 7-day hourly forecasts and return them as a list of hourly records (not just current weather).

<details>
<summary>🔑 Answer</summary>

```python
def fetch_hourly_forecast(self, city_name, lat, lon, timezone="auto", days=7):
    """Fetch hourly forecast for a city."""
    self._rate_limit()
    
    params = {
        "latitude": lat,
        "longitude": lon,
        "hourly": "temperature_2m,relative_humidity_2m,precipitation_probability",
        "timezone": timezone,
        "forecast_days": days
    }
    
    try:
        response = requests.get(self.base_url, params=params, timeout=10)
        response.raise_for_status()
        data = response.json()
        self._last_request = time.time()
        
        hourly = data.get("hourly", {})
        times = hourly.get("time", [])
        temps = hourly.get("temperature_2m", [])
        humidity = hourly.get("relative_humidity_2m", [])
        precip_prob = hourly.get("precipitation_probability", [])
        
        records = []
        for i in range(len(times)):
            records.append({
                "city": city_name,
                "datetime": times[i],
                "temperature_c": temps[i] if i < len(temps) else None,
                "humidity_pct": humidity[i] if i < len(humidity) else None,
                "precipitation_probability_pct": precip_prob[i] if i < len(precip_prob) else None,
                "fetched_at": datetime.now().isoformat(),
                "source": "openmeteo"
            })
        
        logger.info(f"Fetched {len(records)} hourly records for {city_name}")
        return records
        
    except Exception as e:
        logger.error(f"Failed to fetch hourly forecast for {city_name}: {e}")
        return []
```

</details>

**Exercise 3:** Write a unit test for the `WeatherExtractor.fetch_current_weather()` method that mocks the API response using `unittest.mock`.

<details>
<summary>🔑 Answer</summary>

```python
# tests/test_weather_extractor.py

import unittest
from unittest.mock import patch, MagicMock
from src.extract.weather import WeatherExtractor

class TestWeatherExtractor(unittest.TestCase):
    
    def setUp(self):
        self.extractor = WeatherExtractor()
    
    @patch("src.extract.weather.requests.get")
    def test_fetch_current_weather_success(self, mock_get):
        """Test successful weather fetch."""
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.raise_for_status = MagicMock()
        mock_response.json.return_value = {
            "current": {
                "temperature_2m": 31.5,
                "relative_humidity_2m": 82,
                "wind_speed_10m": 12.3,
                "precipitation": 0.0
            },
            "daily": {
                "temperature_2m_max": [33.0],
                "temperature_2m_min": [26.0],
                "precipitation_sum": [1.2]
            }
        }
        mock_get.return_value = mock_response
        
        result = self.extractor.fetch_current_weather("Singapore", 1.3521, 103.8198)
        
        self.assertEqual(result["city"], "Singapore")
        self.assertEqual(result["temperature_c"], 31.5)
        self.assertEqual(result["humidity_pct"], 82)
        self.assertEqual(result["status"], "success")
    
    @patch("src.extract.weather.requests.get")
    def test_fetch_current_weather_failure(self, mock_get):
        """Test weather fetch with API failure."""
        mock_get.side_effect = Exception("Connection error")
        
        result = self.extractor.fetch_current_weather("Singapore", 1.3521, 103.8198)
        
        self.assertEqual(result["city"], "Singapore")
        self.assertIsNone(result["temperature_c"])
        self.assertIn("error", result["status"])

if __name__ == "__main__":
    unittest.main()
```

</details>

**Exercise 4:** Write the `fetch_all_cities()` method so that it also saves a raw JSON backup of each city's API response to `data/raw/weather_{city}_{date}.json` before normalizing.

<details>
<summary>🔑 Answer</summary>

```python
import json
import os
from datetime import datetime

def fetch_all_cities(self):
    """Fetch weather for all cities with raw backup."""
    results = []
    today = datetime.now().strftime("%Y%m%d")
    
    for city_name, coords in CITIES.items():
        logger.info(f"Fetching weather for {city_name}...")
        
        self._rate_limit()
        params = {
            "latitude": coords["lat"],
            "longitude": coords["lon"],
            "current": "temperature_2m,relative_humidity_2m,wind_speed_10m,precipitation",
            "daily": "temperature_2m_max,temperature_2m_min,precipitation_sum",
            "timezone": coords.get("timezone", "auto"),
            "forecast_days": 1
        }
        
        try:
            response = requests.get(self.base_url, params=params, timeout=10)
            response.raise_for_status()
            raw_data = response.json()
            
            # Save raw JSON backup
            raw_dir = "data/raw"
            os.makedirs(raw_dir, exist_ok=True)
            safe_city = city_name.lower().replace(" ", "_")
            raw_file = f"{raw_dir}/weather_{safe_city}_{today}.json"
            with open(raw_file, "w") as f:
                json.dump(raw_data, f, indent=2)
            logger.info(f"  Saved raw data to {raw_file}")
            
            # Normalize
            current = raw_data.get("current", {})
            daily = raw_data.get("daily", {})
            
            weather = {
                "city": city_name,
                "temperature_c": current.get("temperature_2m"),
                "humidity_pct": current.get("relative_humidity_2m"),
                "wind_speed_kmh": current.get("wind_speed_10m"),
                "precipitation_mm": current.get("precipitation"),
                "temp_max_c": daily.get("temperature_2m_max", [None])[0],
                "temp_min_c": daily.get("temperature_2m_min", [None])[0],
                "daily_precipitation_mm": daily.get("precipitation_sum", [None])[0],
                "fetched_at": datetime.now().isoformat(),
                "source": "openmeteo",
                "status": "success"
            }
            results.append(weather)
            
        except Exception as e:
            logger.error(f"Failed to fetch weather for {city_name}: {e}")
            results.append({
                "city": city_name, "status": f"error: {e}",
                "fetched_at": datetime.now().isoformat()
            })
    
    return results
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Transform + Load

### Step 4: Transform Layer

```python
# src/transform/cleaner.py

import pandas as pd
import logging
from datetime import datetime

logger = logging.getLogger(__name__)

class DataTransformer:
    """Transform raw API data into analysis-ready format."""
    
    def clean_weather_data(self, weather_records):
        """Clean and normalize weather data."""
        df = pd.DataFrame(weather_records)
        
        # Remove failed requests
        failed = df[df["status"] != "success"]
        if len(failed) > 0:
            logger.warning(f"Dropping {len(failed)} failed weather records")
        
        df = df[df["status"] == "success"].copy()
        
        if df.empty:
            logger.warning("No successful weather records to transform")
            return df
        
        # Type conversions
        numeric_cols = ["temperature_c", "humidity_pct", "wind_speed_kmh", 
                       "precipitation_mm", "temp_max_c", "temp_min_c", 
                       "daily_precipitation_mm"]
        for col in numeric_cols:
            if col in df.columns:
                df[col] = pd.to_numeric(df[col], errors="coerce")
        
        # Add computed columns
        df["temp_range_c"] = df["temp_max_c"] - df["temp_min_c"]
        df["feels_like_category"] = df["temperature_c"].apply(self._categorize_temperature)
        df["humidity_category"] = df["humidity_pct"].apply(self._categorize_humidity)
        
        # Parse timestamps
        df["fetched_at"] = pd.to_datetime(df["fetched_at"])
        df["date"] = df["fetched_at"].dt.date
        
        # Select final columns
        final_cols = ["city", "date", "temperature_c", "temp_max_c", "temp_min_c",
                      "temp_range_c", "humidity_pct", "humidity_category",
                      "wind_speed_kmh", "precipitation_mm", "daily_precipitation_mm",
                      "feels_like_category", "fetched_at", "source"]
        
        df = df[[c for c in final_cols if c in df.columns]]
        
        logger.info(f"Transformed {len(df)} weather records")
        return df
    
    def clean_crypto_data(self, crypto_records):
        """Clean and normalize crypto data."""
        df = pd.DataFrame(crypto_records)
        
        df = df[df["status"] == "success"].copy()
        
        if df.empty:
            return df
        
        # Type conversions
        numeric_cols = ["price_sgd", "price_usd", "market_cap_sgd", 
                       "volume_24h_sgd", "change_24h_pct"]
        for col in numeric_cols:
            if col in df.columns:
                df[col] = pd.to_numeric(df[col], errors="coerce")
        
        # Add computed columns
        df["change_direction"] = df["change_24h_pct"].apply(
            lambda x: "up" if x and x > 0 else ("down" if x and x < 0 else "flat")
        )
        df["price_range_sgd"] = df.get("high_24h_sgd", 0) - df.get("low_24h_sgd", 0)
        
        # Timestamps
        df["fetched_at"] = pd.to_datetime(df["fetched_at"])
        df["date"] = df["fetched_at"].dt.date
        
        logger.info(f"Transformed {len(df)} crypto records")
        return df
    
    def create_daily_summary(self, weather_df, crypto_df):
        """Join weather + crypto into a daily summary."""
        if weather_df.empty or crypto_df.empty:
            logger.warning("Cannot create summary — missing data")
            return pd.DataFrame()
        
        # Get weather summary
        weather_summary = weather_df.groupby("date").agg({
            "temperature_c": "mean",
            "humidity_pct": "mean",
            "wind_speed_kmh": "mean",
            "precipitation_mm": "sum",
            "city": "count"
        }).rename(columns={"city": "city_count"})
        
        # Get crypto summary
        crypto_summary = crypto_df.groupby("date").agg({
            "price_sgd": "mean",
            "market_cap_sgd": "sum",
            "change_24h_pct": "mean",
            "coin_id": "count"
        }).rename(columns={"coin_id": "crypto_count"})
        
        # Join on date
        summary = weather_summary.join(crypto_summary, how="outer")
        summary = summary.reset_index()
        
        logger.info(f"Created daily summary with {len(summary)} rows")
        return summary
    
    @staticmethod
    def _categorize_temperature(temp):
        if temp is None: return "unknown"
        if temp < 25: return "cool"
        if temp < 30: return "comfortable"
        if temp < 35: return "warm"
        return "hot"
    
    @staticmethod
    def _categorize_humidity(humidity):
        if humidity is None: return "unknown"
        if humidity < 60: return "low"
        if humidity < 80: return "moderate"
        return "high"
```

---

### Step 5: Load — PostgreSQL

```python
# src/load/postgres_loader.py

import psycopg2
import psycopg2.extras
import logging
import pandas as pd
from config import DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD

logger = logging.getLogger(__name__)

class PostgresLoader:
    """Load data into PostgreSQL."""
    
    def __init__(self):
        self.conn_params = {
            "host": DB_HOST,
            "port": DB_PORT,
            "dbname": DB_NAME,
            "user": DB_USER,
            "password": DB_PASSWORD
        }
    
    def _get_connection(self):
        return psycopg2.connect(**self.conn_params)
    
    def create_tables(self):
        """Create tables if they don't exist."""
        conn = self._get_connection()
        cur = conn.cursor()
        
        cur.execute("""
            CREATE TABLE IF NOT EXISTS api_weather (
                id SERIAL PRIMARY KEY,
                city VARCHAR(100) NOT NULL,
                date DATE,
                temperature_c FLOAT,
                temp_max_c FLOAT,
                temp_min_c FLOAT,
                temp_range_c FLOAT,
                humidity_pct FLOAT,
                humidity_category VARCHAR(20),
                wind_speed_kmh FLOAT,
                precipitation_mm FLOAT,
                daily_precipitation_mm FLOAT,
                feels_like_category VARCHAR(20),
                source VARCHAR(50),
                fetched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                UNIQUE(city, date, source)
            );
        """)
        
        cur.execute("""
            CREATE TABLE IF NOT EXISTS api_crypto (
                id SERIAL PRIMARY KEY,
                coin_id VARCHAR(50) NOT NULL,
                name VARCHAR(100),
                symbol VARCHAR(20),
                rank INT,
                date DATE,
                price_sgd FLOAT,
                price_usd FLOAT,
                market_cap_sgd FLOAT,
                volume_24h_sgd FLOAT,
                change_24h_pct FLOAT,
                change_direction VARCHAR(10),
                source VARCHAR(50),
                fetched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                UNIQUE(coin_id, date, source)
            );
        """)
        
        cur.execute("""
            CREATE TABLE IF NOT EXISTS api_daily_summary (
                id SERIAL PRIMARY KEY,
                date DATE UNIQUE,
                avg_temperature_c FLOAT,
                avg_humidity_pct FLOAT,
                avg_wind_speed_kmh FLOAT,
                total_precipitation_mm FLOAT,
                city_count INT,
                avg_crypto_price_sgd FLOAT,
                total_market_cap_sgd FLOAT,
                avg_change_24h_pct FLOAT,
                crypto_count INT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );
        """)
        
        conn.commit()
        cur.close()
        conn.close()
        logger.info("Tables created/verified")
    
    def load_weather(self, df):
        """Upsert weather data into PostgreSQL."""
        if df.empty:
            logger.warning("No weather data to load")
            return 0
        
        conn = self._get_connection()
        cur = conn.cursor()
        rows_loaded = 0
        
        for _, row in df.iterrows():
            try:
                cur.execute("""
                    INSERT INTO api_weather (city, date, temperature_c, temp_max_c, 
                        temp_min_c, temp_range_c, humidity_pct, humidity_category,
                        wind_speed_kmh, precipitation_mm, daily_precipitation_mm,
                        feels_like_category, source, fetched_at)
                    VALUES (%(city)s, %(date)s, %(temperature_c)s, %(temp_max_c)s,
                        %(temp_min_c)s, %(temp_range_c)s, %(humidity_pct)s, 
                        %(humidity_category)s, %(wind_speed_kmh)s, %(precipitation_mm)s,
                        %(daily_precipitation_mm)s, %(feels_like_category)s, 
                        %(source)s, %(fetched_at)s)
                    ON CONFLICT (city, date, source) 
                    DO UPDATE SET
                        temperature_c = EXCLUDED.temperature_c,
                        humidity_pct = EXCLUDED.humidity_pct,
                        fetched_at = EXCLUDED.fetched_at
                """, row.to_dict())
                rows_loaded += 1
            except Exception as e:
                logger.error(f"Failed to insert weather for {row.get('city')}: {e}")
        
        conn.commit()
        cur.close()
        conn.close()
        logger.info(f"Loaded {rows_loaded} weather records")
        return rows_loaded
    
    def load_crypto(self, df):
        """Upsert crypto data into PostgreSQL."""
        if df.empty:
            return 0
        
        conn = self._get_connection()
        cur = conn.cursor()
        rows_loaded = 0
        
        for _, row in df.iterrows():
            try:
                cur.execute("""
                    INSERT INTO api_crypto (coin_id, date, price_sgd, price_usd,
                        market_cap_sgd, volume_24h_sgd, change_24h_pct,
                        change_direction, source, fetched_at)
                    VALUES (%(coin_id)s, %(date)s, %(price_sgd)s, %(price_usd)s,
                        %(market_cap_sgd)s, %(volume_24h_sgd)s, %(change_24h_pct)s,
                        %(change_direction)s, %(source)s, %(fetched_at)s)
                    ON CONFLICT (coin_id, date, source)
                    DO UPDATE SET
                        price_sgd = EXCLUDED.price_sgd,
                        change_24h_pct = EXCLUDED.change_24h_pct,
                        fetched_at = EXCLUDED.fetched_at
                """, row.to_dict())
                rows_loaded += 1
            except Exception as e:
                logger.error(f"Failed to insert crypto {row.get('coin_id')}: {e}")
        
        conn.commit()
        cur.close()
        conn.close()
        logger.info(f"Loaded {rows_loaded} crypto records")
        return rows_loaded
```

---

### Step 6: Load — S3 (CSV + Parquet)

```python
# src/load/s3_loader.py

import boto3
import pandas as pd
import logging
import io
from datetime import datetime
from config import S3_BUCKET, S3_REGION, S3_PREFIX

logger = logging.getLogger(__name__)

class S3Loader:
    """Load data to S3 as CSV and Parquet."""
    
    def __init__(self):
        self.s3 = boto3.client("s3", region_name=S3_REGION)
        self.bucket = S3_BUCKET
        self.prefix = S3_PREFIX
    
    def _get_key(self, layer, filename):
        """Build S3 key: prefix/layer/date/filename"""
        today = datetime.now().strftime("%Y-%m-%d")
        return f"{self.prefix}/{layer}/{today}/{filename}"
    
    def save_csv(self, df, name, layer="processed"):
        """Save DataFrame as CSV to S3."""
        if df.empty:
            logger.warning(f"No data to save as CSV: {name}")
            return None
        
        csv_buffer = io.StringIO()
        df.to_csv(csv_buffer, index=False)
        
        key = self._get_key(layer, f"{name}.csv")
        
        try:
            self.s3.put_object(
                Bucket=self.bucket,
                Key=key,
                Body=csv_buffer.getvalue(),
                ContentType="text/csv"
            )
            logger.info(f"Saved CSV: s3://{self.bucket}/{key}")
            return key
        except Exception as e:
            logger.error(f"Failed to save CSV to S3: {e}")
            return None
    
    def save_parquet(self, df, name, layer="processed"):
        """Save DataFrame as Parquet to S3."""
        if df.empty:
            logger.warning(f"No data to save as Parquet: {name}")
            return None
        
        parquet_buffer = io.BytesIO()
        df.to_parquet(parquet_buffer, engine="pyarrow", index=False)
        
        key = self._get_key(layer, f"{name}.parquet")
        
        try:
            self.s3.put_object(
                Bucket=self.bucket,
                Key=key,
                Body=parquet_buffer.getvalue(),
                ContentType="application/octet-stream"
            )
            logger.info(f"Saved Parquet: s3://{self.bucket}/{key}")
            return key
        except Exception as e:
            logger.error(f"Failed to save Parquet to S3: {e}")
            return None
    
    def save_local_fallback(self, df, name, layer="processed"):
        """Save locally if S3 fails (for development)."""
        import os
        today = datetime.now().strftime("%Y-%m-%d")
        local_dir = f"data/{layer}/{today}"
        os.makedirs(local_dir, exist_ok=True)
        
        csv_path = f"{local_dir}/{name}.csv"
        df.to_csv(csv_path, index=False)
        logger.info(f"Saved local CSV: {csv_path}")
        
        parquet_path = f"{local_dir}/{name}.parquet"
        try:
            df.to_parquet(parquet_path, index=False)
            logger.info(f"Saved local Parquet: {parquet_path}")
        except Exception as e:
            logger.warning(f"Parquet save failed: {e}")
        
        return csv_path
```

---

### Step 7: The Pipeline Orchestrator

```python
# run_pipeline.py

import logging
import sys
from datetime import datetime

from config import *
from src.extract.weather import WeatherExtractor
from src.extract.crypto import CryptoExtractor
from src.transform.cleaner import DataTransformer
from src.load.postgres_loader import PostgresLoader
from src.load.s3_loader import S3Loader

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler(f"logs/pipeline_{datetime.now().strftime('%Y%m%d')}.log")
    ]
)

logger = logging.getLogger("pipeline")

def run_pipeline():
    """Run the complete API data pipeline."""
    start_time = datetime.now()
    logger.info("=" * 60)
    logger.info("API DATA PIPELINE STARTED")
    logger.info("=" * 60)
    
    # === EXTRACT ===
    logger.info("--- EXTRACT PHASE ---")
    
    weather_extractor = WeatherExtractor()
    crypto_extractor = CryptoExtractor()
    
    weather_data = weather_extractor.fetch_all_cities()
    crypto_data = crypto_extractor.fetch_prices()
    
    logger.info(f"Extracted: {len(weather_data)} weather, {len(crypto_data)} crypto records")
    
    # === TRANSFORM ===
    logger.info("--- TRANSFORM PHASE ---")
    
    transformer = DataTransformer()
    
    weather_df = transformer.clean_weather_data(weather_data)
    crypto_df = transformer.clean_crypto_data(crypto_data)
    summary_df = transformer.create_daily_summary(weather_df, crypto_df)
    
    logger.info(f"Transformed: {len(weather_df)} weather, {len(crypto_df)} crypto, "
                f"{len(summary_df)} summary records")
    
    # === LOAD ===
    logger.info("--- LOAD PHASE ---")
    
    # Load to PostgreSQL
    try:
        pg_loader = PostgresLoader()
        pg_loader.create_tables()
        weather_rows = pg_loader.load_weather(weather_df)
        crypto_rows = pg_loader.load_crypto(crypto_df)
        logger.info(f"PostgreSQL: {weather_rows} weather, {crypto_rows} crypto rows loaded")
    except Exception as e:
        logger.error(f"PostgreSQL load failed: {e}")
        logger.info("Falling back to local CSV storage...")
        pg_loader = None
    
    # Load to S3 (with local fallback)
    try:
        s3_loader = S3Loader()
        s3_loader.save_csv(weather_df, "weather_data", "processed")
        s3_loader.save_parquet(weather_df, "weather_data", "processed")
        s3_loader.save_csv(crypto_df, "crypto_data", "processed")
        s3_loader.save_parquet(crypto_df, "crypto_data", "processed")
        if not summary_df.empty:
            s3_loader.save_csv(summary_df, "daily_summary", "analytics")
            s3_loader.save_parquet(summary_df, "daily_summary", "analytics")
    except Exception as e:
        logger.warning(f"S3 load failed: {e}")
        logger.info("Falling back to local storage...")
        s3_loader_fallback = S3Loader()
        s3_loader_fallback.save_local_fallback(weather_df, "weather_data", "processed")
        s3_loader_fallback.save_local_fallback(crypto_df, "crypto_data", "processed")
        if not summary_df.empty:
            s3_loader_fallback.save_local_fallback(summary_df, "daily_summary", "analytics")
    
    # === SUMMARY ===
    elapsed = (datetime.now() - start_time).total_seconds()
    logger.info("=" * 60)
    logger.info(f"PIPELINE COMPLETE in {elapsed:.1f}s")
    logger.info(f"  Weather: {len(weather_df)} records")
    logger.info(f"  Crypto: {len(crypto_df)} records")
    logger.info(f"  Summary: {len(summary_df)} records")
    logger.info("=" * 60)
    
    return {
        "status": "success",
        "elapsed_seconds": elapsed,
        "weather_records": len(weather_df),
        "crypto_records": len(crypto_df),
        "summary_records": len(summary_df)
    }

if __name__ == "__main__":
    result = run_pipeline()
    print(f"\n✅ Pipeline result: {result}")
```

---

### Afternoon Exercises

**Exercise 5:** Add data validation to the `DataTransformer` — before returning the cleaned DataFrame, check that:
- No duplicate city+date combinations in weather data
- Price values are positive in crypto data
- Required fields are not null
- Log warnings for any violations but don't drop rows

<details>
<summary>🔑 Answer</summary>

```python
def validate_weather(self, df):
    """Validate weather data before loading."""
    issues = []
    
    # Check for duplicates
    if not df.empty:
        dupes = df[df.duplicated(subset=["city", "date"], keep=False)]
        if len(dupes) > 0:
            issues.append(f"Found {len(dupes)} duplicate city+date records")
            logger.warning(issues[-1])
    
    # Check for null required fields
    required = ["city", "temperature_c", "date"]
    for col in required:
        if col in df.columns:
            null_count = df[col].isnull().sum()
            if null_count > 0:
                issues.append(f"{col}: {null_count} null values")
                logger.warning(issues[-1])
    
    # Check reasonable ranges
    if "temperature_c" in df.columns:
        bad_temps = df[(df["temperature_c"] < -50) | (df["temperature_c"] > 60)]
        if len(bad_temps) > 0:
            issues.append(f"{len(bad_temps)} temperatures outside -50 to 60°C")
            logger.warning(issues[-1])
    
    if not issues:
        logger.info("✅ Weather data validation passed")
    
    return issues

def validate_crypto(self, df):
    """Validate crypto data before loading."""
    issues = []
    
    if df.empty:
        return ["Empty DataFrame"]
    
    # Check positive prices
    if "price_sgd" in df.columns:
        negative = df[df["price_sgd"] < 0]
        if len(negative) > 0:
            issues.append(f"{len(negative)} negative prices found")
            logger.warning(issues[-1])
    
    # Check for null coin_id
    if "coin_id" in df.columns:
        null_coins = df["coin_id"].isnull().sum()
        if null_coins > 0:
            issues.append(f"{null_coins} null coin_id values")
            logger.warning(issues[-1])
    
    # Check for duplicates
    dupes = df[df.duplicated(subset=["coin_id", "date"], keep=False)]
    if len(dupes) > 0:
        issues.append(f"{len(dupes)} duplicate coin_id+date records")
        logger.warning(issues[-1])
    
    if not issues:
        logger.info("✅ Crypto data validation passed")
    
    return issues
```

</details>

**Exercise 6:** Write a cron schedule expression and a shell script that runs the pipeline every hour at the top of the hour (e.g., 1:00, 2:00, 3:00).

<details>
<summary>🔑 Answer</summary>

```bash
#!/bin/bash
# run_hourly.sh — Runs the API data pipeline every hour

set -e

PIPELINE_DIR="/path/to/api-data-pipeline"
LOG_DIR="$PIPELINE_DIR/logs"
PYTHON="/usr/bin/python3"

cd "$PIPELINE_DIR"

# Create log directory
mkdir -p "$LOG_DIR"

# Run pipeline with timestamped log
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
echo "[$(date)] Starting pipeline run..." >> "$LOG_DIR/cron.log"

$PYTHON run_pipeline.py >> "$LOG_DIR/cron_${TIMESTAMP}.log" 2>&1

EXIT_CODE=$?
echo "[$(date)] Pipeline finished with exit code $EXIT_CODE" >> "$LOG_DIR/cron.log"

# Alert on failure
if [ $EXIT_CODE -ne 0 ]; then
    echo "Pipeline FAILED at $(date)" >> "$LOG_DIR/alerts.log"
fi

exit $EXIT_CODE
```

**Crontab entry:**
```bash
# Edit crontab: crontab -e
# Run every hour at minute 0
0 * * * * /path/to/api-data-pipeline/run_hourly.sh
```

**Or with Airflow DAG:**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    "owner": "data-engineer",
    "depends_on_past": False,
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
}

with DAG(
    "api_data_pipeline",
    default_args=default_args,
    schedule_interval="0 * * * *",  # Every hour
    start_date=datetime(2026, 7, 11),
    catchup=False,
) as dag:
    
    run_pipeline = PythonOperator(
        task_id="run_api_pipeline",
        python_callable=run_pipeline,  # Import from run_pipeline.py
    )
```

</details>

**Exercise 7:** Add an Airflow DAG version of this pipeline that splits extract, transform, and load into separate tasks. Use XCom to pass data between tasks.

<details>
<summary>🔑 Answer</summary>

```python
# dags/api_data_pipeline_dag.py

from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import json

default_args = {
    "owner": "data-engineer",
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
}

def extract_weather(**context):
    """Extract weather data and push to XCom."""
    from src.extract.weather import WeatherExtractor
    extractor = WeatherExtractor()
    data = extractor.fetch_all_cities()
    # XCom can handle JSON-serializable data
    context["ti"].xcom_push(key="weather_data", value=data)
    return len(data)

def extract_crypto(**context):
    """Extract crypto data and push to XCom."""
    from src.extract.crypto import CryptoExtractor
    extractor = CryptoExtractor()
    data = extractor.fetch_prices()
    context["ti"].xcom_push(key="crypto_data", value=data)
    return len(data)

def transform_data(**context):
    """Pull raw data, transform, push results."""
    from src.transform.cleaner import DataTransformer
    
    ti = context["ti"]
    weather_raw = ti.xcom_pull(key="weather_data", task_ids="extract_weather")
    crypto_raw = ti.xcom_pull(key="crypto_data", task_ids="extract_crypto")
    
    transformer = DataTransformer()
    weather_df = transformer.clean_weather_data(weather_raw or [])
    crypto_df = transformer.clean_crypto_data(crypto_raw or [])
    summary_df = transformer.create_daily_summary(weather_df, crypto_df)
    
    # Push DataFrames as dicts (XCom limitation)
    ti.xcom_push(key="weather_records", value=weather_df.to_dict("records"))
    ti.xcom_push(key="crypto_records", value=crypto_df.to_dict("records"))
    ti.xcom_push(key="summary_records", value=summary_df.to_dict("records"))
    
    return {"weather": len(weather_df), "crypto": len(crypto_df)}

def load_postgres(**context):
    """Load transformed data to PostgreSQL."""
    import pandas as pd
    from src.load.postgres_loader import PostgresLoader
    
    ti = context["ti"]
    weather = pd.DataFrame(ti.xcom_pull(key="weather_records", task_ids="transform_data") or [])
    crypto = pd.DataFrame(ti.xcom_pull(key="crypto_records", task_ids="transform_data") or [])
    
    loader = PostgresLoader()
    loader.create_tables()
    w_rows = loader.load_weather(weather)
    c_rows = loader.load_crypto(crypto)
    
    return {"postgres_weather": w_rows, "postgres_crypto": c_rows}

def load_s3(**context):
    """Load transformed data to S3."""
    import pandas as pd
    from src.load.s3_loader import S3Loader
    
    ti = context["ti"]
    weather = pd.DataFrame(ti.xcom_pull(key="weather_records", task_ids="transform_data") or [])
    crypto = pd.DataFrame(ti.xcom_pull(key="crypto_records", task_ids="transform_data") or [])
    summary = pd.DataFrame(ti.xcom_pull(key="summary_records", task_ids="transform_data") or [])
    
    loader = S3Loader()
    loader.save_parquet(weather, "weather_data")
    loader.save_parquet(crypto, "crypto_data")
    if not summary.empty:
        loader.save_parquet(summary, "daily_summary", "analytics")
    
    return "S3 load complete"

with DAG(
    "api_data_pipeline",
    default_args=default_args,
    schedule_interval="@hourly",
    start_date=datetime(2026, 7, 11),
    catchup=False,
    description="Extract weather + crypto data from APIs, transform, load to PG + S3",
) as dag:
    
    t_extract_weather = PythonOperator(
        task_id="extract_weather",
        python_callable=extract_weather,
    )
    
    t_extract_crypto = PythonOperator(
        task_id="extract_crypto",
        python_callable=extract_crypto,
    )
    
    t_transform = PythonOperator(
        task_id="transform_data",
        python_callable=transform_data,
    )
    
    t_load_postgres = PythonOperator(
        task_id="load_postgres",
        python_callable=load_postgres,
    )
    
    t_load_s3 = PythonOperator(
        task_id="load_s3",
        python_callable=load_s3,
    )
    
    # Dependencies
    [t_extract_weather, t_extract_crypto] >> t_transform >> [t_load_postgres, t_load_s3]
```

</details>

---

## 🌙 Evening (1 hour): End-to-End Test + Push

### Testing the Full Pipeline

```bash
# 1. Run the pipeline
python run_pipeline.py

# 2. Verify PostgreSQL
psql -h localhost -U postgres -d makanexpress -c "SELECT * FROM api_weather ORDER BY fetched_at DESC LIMIT 5;"
psql -h localhost -U postgres -d makanexpress -c "SELECT * FROM api_crypto ORDER BY fetched_at DESC LIMIT 5;"

# 3. Verify local files (if S3 unavailable)
ls -la data/processed/$(date +%Y-%m-%d)/
ls -la data/analytics/$(date +%Y-%m-%d)/

# 4. Check logs
cat logs/pipeline_$(date +%Y%m%d).log
```

### GitHub Push

```bash
cd api-data-pipeline
git init
git add .
git commit -m "feat: API data pipeline - weather + crypto ETL

- Extract: OpenMeteo weather (SG/KL/Penang/JB) + CoinGecko crypto prices
- Transform: clean, normalize, join into daily summary
- Load: PostgreSQL (upsert) + S3 (CSV + Parquet)
- Error handling: retry logic, rate limiting, fallback to local
- Airflow DAG included for scheduling"

git remote add origin https://github.com/YOUR_USERNAME/api-data-pipeline.git
git push -u origin main
```

### README Structure

```markdown
# API Data Pipeline

Real-time data pipeline that collects weather and cryptocurrency data from free APIs, transforms it, and loads into PostgreSQL and S3.

## Architecture
[ASCII diagram from above]

## Data Sources
- OpenMeteo (weather) — Free, no API key
- CoinGecko (crypto) — Free tier

## Setup
\`\`\`bash
pip install -r requirements.txt
cp .env.example .env  # Edit with your DB/S3 credentials
python run_pipeline.py
\`\`\`

## Schedule
- Cron: Every hour
- Airflow DAG included

## Tech Stack
Python, requests, pandas, psycopg2, boto3, Airflow
```

---

### 📝 Today's Checklist

- [ ] Built a complete Extract → Transform → Load pipeline
- [ ] Extract: Called OpenMeteo + CoinGecko APIs with error handling
- [ ] Transform: Cleaned data, added computed columns, created daily summary
- [ ] Load: Saved to PostgreSQL (upsert) and S3 (CSV + Parquet)
- [ ] Added data validation before loading
- [ ] Created Airflow DAG with proper task dependencies
- [ ] Pipeline runs end-to-end successfully
- [ ] Pushed to GitHub with README
- [ ] Ready for tomorrow: LLM API basics

---

*Day 58 complete! Tomorrow: LLM API basics — use AI APIs for data tasks like classification and SQL generation.* 🤖
