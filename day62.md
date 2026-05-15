# 📅 Day 62 — Tuesday, 15 July 2026
# 📝 Week 9 Assessment: APIs + LLM + RAG Review

---

## 🎯 Today's Goal

Timed assessment covering Week 9: REST APIs, Python requests, pagination, rate limiting, LLM APIs, prompt engineering, embeddings, vector search, and RAG pipelines. Grade yourself honestly.

---

## ☀️ Morning Block (2 hours): Timed Assessment

**Rules:** 90 minutes, no notes, no Google, no AI. Write answers on paper or blank file.

### Section A: API Fundamentals (15 min, 5 questions)

**Q1.** Explain the difference between GET and POST HTTP methods. Give a data engineering example for each.

**Q2.** What does HTTP status code 429 mean? How should your code handle it?

**Q3.** Name 3 types of API pagination. Which one is best for real-time data and why?

**Q4.** Why should you ALWAYS set a timeout on API requests? What happens if you don't?

**Q5.** What is exponential backoff? Write pseudocode for a retry function that uses it.

### Section B: Python API Programming (20 min, 5 questions)

**Q6.** Write a Python function that calls the OpenMeteo API to get the current temperature for Penang (lat: 5.4164, lon: 100.3327). Include error handling and timeout.

**Q7.** Write a paginated API caller that fetches ALL pages from an API with offset-based pagination (100 items per page). Stop when an empty page is returned.

**Q8.** Write a Python class `RateLimiter` that ensures no more than N requests per second are made. Show how to use it with a loop of 10 API calls.

**Q9.** Write a function that calls the CoinGecko `/simple/price` endpoint for Bitcoin in SGD. If rate limited (429), retry up to 3 times with exponential backoff.

**Q10.** Write a complete `APIClient` class with: `get()`, `post()`, `get_all_pages()` (cursor-based pagination), and automatic retry on 5xx errors.

### Section C: ETL Pipeline (20 min, 5 questions)

**Q11.** Design a pipeline that extracts data from 2 APIs (weather + crypto), transforms it, and loads into both PostgreSQL and S3. Draw the architecture.

**Q12.** Write the transform function that takes raw weather data (JSON) and returns a cleaned DataFrame with: city, temperature, humidity, date, temperature_category.

**Q13.** Write an upsert (INSERT ... ON CONFLICT) SQL statement for loading daily weather data into PostgreSQL. Explain why upsert is better than INSERT for this use case.

**Q14.** Write a function that saves a Pandas DataFrame to S3 as both CSV and Parquet. Include error handling and local file fallback.

**Q15.** Write an Airflow DAG that runs the API pipeline every hour with: extract_weather → extract_crypto (parallel) → transform → load_postgres + load_s3 (parallel).

### Section D: LLM APIs + RAG (20 min, 5 questions)

**Q16.** Explain what embeddings are and why they're useful. Give 3 data engineering use cases.

**Q17.** Write code to generate an embedding using the OpenAI API and store it in a PostgreSQL table with pgvector.

**Q18.** Write a SQL query that performs cosine similarity search on a `documents` table with an `embedding` column. Find the top 5 most similar documents to a query vector.

**Q19.** Explain the 4 steps of a RAG pipeline. Why is RAG better than just asking the LLM directly?

**Q20.** Write a prompt for an LLM that classifies food reviews into sentiments. The prompt should: (a) set the system role, (b) specify JSON output format, (c) handle edge cases.

---

## 🌤️ Afternoon Block (2 hours): Answers + Review

### Scoring

| Section | Questions | Points |
|---------|-----------|--------|
| A: API Fundamentals | 5 × 4 | 20 |
| B: Python API | 5 × 6 | 30 |
| C: ETL Pipeline | 5 × 8 | 40 |
| D: LLM + RAG | 5 × 2 | 10 |
| **Total** | **20** | **100** |

### Grade Scale
- 90-100: Excellent — Week 9 concepts are solid
- 70-89: Good — minor gaps to review
- 50-69: Needs work — re-read weak areas
- Below 50: Significant gaps — re-do exercises for weak sections

---

### ✅ Full Answers

<details>
<summary>🔑 Section A: API Fundamentals</summary>

**A1:**
- GET: Read/fetch data. DE example: Fetching daily weather data from OpenMeteo API.
- POST: Create/send data. DE example: Loading processed records into a third-party analytics API.

**A2:** 429 = Too Many Requests (rate limited). Handle by: reading Retry-After header, waiting with exponential backoff, then retrying. Never retry immediately.

**A3:**
1. Offset-based: `?offset=0&limit=100` — simple but slow for deep pages
2. Cursor-based: `?cursor=abc123` — best for real-time, consistent even if data changes
3. Page-based: `?page=1&per_page=50` — simplest, common in REST APIs

Cursor-based is best for real-time because it doesn't skip records even if new ones are inserted during pagination.

**A4:** Without timeout, `requests.get()` can hang indefinitely if the server is slow or unresponsive. This blocks your pipeline forever. Always use `timeout=10` (or appropriate value).

**A5:**
```
function callWithRetry(url, maxRetries=3):
    for attempt in 0 to maxRetries-1:
        try:
            response = makeRequest(url)
            if response.status == 200: return response
            if response.status == 429 or response.status >= 500:
                delay = 2^attempt seconds
                sleep(delay)
                continue
            else: return error
        except Exception:
            sleep(2^attempt)
    return "Max retries exceeded"
```

</details>

<details>
<summary>🔑 Section B: Python API Programming</summary>

**B6:**
```python
import requests

def get_penang_weather():
    url = "https://api.open-meteo.com/v1/forecast"
    params = {"latitude": 5.4164, "longitude": 100.3327, "current": "temperature_2m", "timezone": "auto"}
    try:
        response = requests.get(url, params=params, timeout=10)
        response.raise_for_status()
        return response.json()["current"]["temperature_2m"]
    except requests.exceptions.RequestException as e:
        print(f"Error: {e}")
        return None
```

**B7:**
```python
import requests

def fetch_all_pages(base_url):
    all_data = []
    offset = 0
    while True:
        response = requests.get(base_url, params={"offset": offset, "limit": 100}, timeout=10)
        data = response.json()
        items = data.get("results", [])
        if not items:
            break
        all_data.extend(items)
        offset += 100
    return all_data
```

**B8:**
```python
import time

class RateLimiter:
    def __init__(self, calls_per_second):
        self.min_interval = 1.0 / calls_per_second
        self._last_call = 0
    
    def wait(self):
        elapsed = time.time() - self._last_call
        if elapsed < self.min_interval:
            time.sleep(self.min_interval - elapsed)
        self._last_call = time.time()

# Usage
limiter = RateLimiter(2)  # 2 calls per second
for i in range(10):
    limiter.wait()
    # make API call here
    print(f"Call {i+1} at {time.time():.2f}")
```

**B9:**
```python
import requests, time

def get_btc_price():
    url = "https://api.coingecko.com/api/v3/simple/price"
    params = {"ids": "bitcoin", "vs_currencies": "sgd"}
    for attempt in range(3):
        try:
            response = requests.get(url, params=params, timeout=10)
            if response.status_code == 429:
                time.sleep(2 ** attempt)
                continue
            response.raise_for_status()
            return response.json()["bitcoin"]["sgd"]
        except Exception:
            time.sleep(2 ** attempt)
    return None
```

**B10:**
```python
import requests, time, logging

class APIClient:
    def __init__(self, base_url, max_retries=3):
        self.base_url = base_url.rstrip("/")
        self.max_retries = max_retries
    
    def get(self, endpoint, params=None):
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        for attempt in range(self.max_retries):
            try:
                r = requests.get(url, params=params, timeout=10)
                if r.status_code == 429 or r.status_code >= 500:
                    time.sleep(2 ** attempt)
                    continue
                r.raise_for_status()
                return r.json()
            except requests.exceptions.RequestException:
                time.sleep(2 ** attempt)
        return None
    
    def post(self, endpoint, data=None):
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        for attempt in range(self.max_retries):
            try:
                r = requests.post(url, json=data, timeout=10)
                if r.status_code >= 500:
                    time.sleep(2 ** attempt)
                    continue
                r.raise_for_status()
                return r.json()
            except requests.exceptions.RequestException:
                time.sleep(2 ** attempt)
        return None
    
    def get_all_pages(self, endpoint, cursor_key="next_cursor", params=None):
        all_data, cursor = [], None
        while True:
            p = params.copy() if params else {}
            if cursor: p[cursor_key] = cursor
            data = self.get(endpoint, params=p)
            if not data or not data.get("results"):
                break
            all_data.extend(data["results"])
            cursor = data.get(cursor_key)
            if not cursor: break
        return all_data
```

</details>

<details>
<summary>🔑 Section C: ETL Pipeline</summary>

**C11:** Architecture diagram:
```
OpenMeteo API ──┐                ┌── PostgreSQL (upsert)
                ├── Extract ────►│
CoinGecko API ──┘    │           └── S3 (CSV + Parquet)
                     ▼
               Transform
            (clean, normalize, join)
```
Key components: rate limiting in extract, type casting in transform, idempotent upsert in load.

**C12:**
```python
def transform_weather(raw_data):
    df = pd.DataFrame(raw_data)
    df = df[df["status"] == "success"]
    df["temperature_c"] = pd.to_numeric(df["temperature_c"], errors="coerce")
    df["humidity_pct"] = pd.to_numeric(df["humidity_pct"], errors="coerce")
    df["date"] = pd.to_datetime(df["fetched_at"]).dt.date
    df["temperature_category"] = df["temperature_c"].apply(
        lambda x: "hot" if x >= 35 else ("warm" if x >= 30 else ("comfortable" if x >= 25 else "cool"))
    )
    return df[["city", "temperature_c", "humidity_pct", "date", "temperature_category"]]
```

**C13:** Upsert is better because if you re-run the pipeline, it updates existing records instead of creating duplicates.
```sql
INSERT INTO api_weather (city, date, temperature_c, humidity_pct)
VALUES (%s, %s, %s, %s)
ON CONFLICT (city, date)
DO UPDATE SET temperature_c = EXCLUDED.temperature_c, humidity_pct = EXCLUDED.humidity_pct;
```

**C14:**
```python
import boto3, pandas as pd, io

def save_to_s3(df, name, bucket, prefix="processed"):
    s3 = boto3.client("s3", region_name="ap-southeast-1")
    key = f"{prefix}/{name}"
    try:
        # CSV
        csv_buf = io.StringIO()
        df.to_csv(csv_buf, index=False)
        s3.put_object(Bucket=bucket, Key=f"{key}.csv", Body=csv_buf.getvalue())
        # Parquet
        pq_buf = io.BytesIO()
        df.to_parquet(pq_buf, index=False)
        s3.put_object(Bucket=bucket, Key=f"{key}.parquet", Body=pq_buf.getvalue())
    except Exception as e:
        # Fallback to local
        df.to_csv(f"{name}.csv", index=False)
```

**C15:** DAG with task dependencies:
```python
[extract_weather, extract_crypto] >> transform >> [load_postgres, load_s3]
```
Both extract tasks run in parallel, transform waits for both, then load tasks run in parallel.

</details>

<details>
<summary>🔑 Section D: LLM + RAG</summary>

**D16:** Embeddings are numerical vector representations of text that capture semantic meaning. Similar text = similar vectors.
Use cases: (1) Semantic search over data catalog, (2) Document Q&A / RAG, (3) Anomaly detection by finding dissimilar records.

**D17:**
```python
from openai import OpenAI
import psycopg2

client = OpenAI(api_key="...")
response = client.embeddings.create(input="chicken rice", model="text-embedding-3-small")
embedding = response.data[0].embedding

conn = psycopg2.connect(...)
cur = conn.cursor()
cur.execute("INSERT INTO docs (content, embedding) VALUES (%s, %s::vector)", ("chicken rice", str(embedding)))
conn.commit()
```

**D18:**
```sql
SELECT id, content,
       1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;
```

**D19:**
1. Embed: Convert user question to vector
2. Retrieve: Search vector database for similar documents
3. Augment: Combine question + retrieved documents into prompt
4. Generate: LLM produces answer grounded in real data

RAG is better because: answers are based on YOUR data (not training data), reduces hallucinations, can cite sources, works with private/current data.

**D20:**
```python
messages = [
    {"role": "system", "content": "Classify food delivery reviews. Return JSON: {\"sentiment\": \"positive|negative|neutral\", \"confidence\": 0.0-1.0, \"category\": \"food|delivery|service|price|other\"}. For unclear reviews, use neutral with low confidence."},
    {"role": "user", "content": f"Review: {review_text}"}
]
```

</details>

---

## 🌙 Evening: Targeted Review

### Weak Spot Guide

| Score < 70% in | Action |
|----------------|--------|
| Section A | Re-read Day 57 morning concepts |
| Section B | Re-read Day 57 afternoon, practice requests library |
| Section C | Re-read Day 58, re-build the pipeline from scratch |
| Section D | Re-read Day 59-60, practice embeddings + pgvector |

### 📝 Today's Checklist

- [ ] Completed assessment honestly (90 min, no notes)
- [ ] Scored at least 70/100
- [ ] Identified 2-3 weak areas
- [ ] Reviewed weak areas with exercises
- [ ] Ready for Week 10: Data Quality + Interview Prep

---

*Day 62 complete! Week 9 nearly done. Tomorrow: Portfolio Enhancement day.* 📝
