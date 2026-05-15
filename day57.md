# 📅 Day 57 — Thursday, 10 July 2026
# 🌐 REST API Deep Dive: requests, pagination, error handling

---

## 🎯 Today's Goal

APIs are how modern data engineers collect data. Every company exposes data through APIs — GrabFood has restaurant APIs, Shopee has product APIs, weather services have forecast APIs. Today you learn to call APIs professionally: with error handling, pagination, and rate limiting.

**By end of day:** You'll be able to call any REST API, handle errors gracefully, paginate through large datasets, and respect rate limits. This is a core data engineering skill.

---

## ☀️ Morning Block (2 hours): API Fundamentals + Python requests

### What is an API?

**The restaurant analogy:**

Imagine you're at a hawker center in Singapore. You can't just walk into the kitchen and cook — you:
1. Look at the **menu** (API documentation / endpoints)
2. **Order** from the counter (make a request)
3. **Receive** your food (get a response)

An API (Application Programming Interface) works the same way:
- The **menu** = available endpoints (URLs you can call)
- Your **order** = HTTP request (what data you want)
- Your **food** = HTTP response (the data you get back)

**Why data engineers care:** APIs are the #1 way to extract data from SaaS tools, government data portals, and third-party services. If you can't call APIs, you can't collect data.

---

### HTTP Methods: The Verbs of the Web

| Method | Purpose | Data Engineering Use | Analogy |
|--------|---------|---------------------|---------|
| **GET** | Read/fetch data | Extract data from APIs | Reading the menu |
| **POST** | Create new data | Ingest data into a system | Placing an order |
| **PUT** | Update entire record | Update a full record | Replacing your entire order |
| **PATCH** | Update partial record | Update specific fields | Adding extra chili |
| **DELETE** | Remove data | Rarely used in DE | Cancelling your order |

**As a data engineer, 90% of your API work is GET.** You're reading data from sources. POST/PUT are used when loading data INTO APIs (less common but important).

---

### HTTP Status Codes: The Traffic Lights

| Code | Meaning | What to do |
|------|---------|------------|
| **200** | OK — Success! | Process the response |
| **201** | Created — POST success | Note the new resource |
| **400** | Bad Request — You sent bad data | Fix your request parameters |
| **401** | Unauthorized — Missing/wrong credentials | Add API key or token |
| **403** | Forbidden — You can't access this | Check permissions |
| **404** | Not Found — Wrong URL | Check the endpoint |
| **429** | Too Many Requests — Rate limited | Slow down, add delays |
| **500** | Internal Server Error — Their problem | Retry with backoff |
| **502/503** | Gateway/Service Unavailable | Retry later |

**Key insight:** 4xx errors = YOUR fault. 5xx errors = THEIR fault. 429 = slow down!

---

### JSON: The Universal Data Language

APIs almost always return **JSON** (JavaScript Object Notation). It looks like Python dictionaries:

```json
{
    "city": "Singapore",
    "temperature": 31.5,
    "humidity": 85,
    "forecasts": [
        {"day": "Monday", "high": 33, "low": 26},
        {"day": "Tuesday", "high": 32, "low": 25}
    ],
    "metadata": {
        "source": "OpenMeteo",
        "units": "celsius"
    }
}
```

Python handles JSON natively:

```python
import json

# Parse JSON string → Python dict
data = json.loads('{"city": "Singapore", "temp": 31.5}')
print(data["city"])     # Singapore
print(data["temp"])     # 31.5

# Python dict → JSON string
payload = {"city": "KL", "temp": 33.0}
json_str = json.dumps(payload, indent=2)
```

---

### Python `requests` Library: Your API Swiss Army Knife

```python
import requests

# Basic GET request
response = requests.get("https://api.example.com/data")

# Check status
print(response.status_code)    # 200
print(response.ok)             # True (status < 400)

# Get response as JSON (dict)
data = response.json()

# Get raw text
text = response.text
```

**Adding parameters:**

```python
# Query parameters (the stuff after ? in URLs)
params = {
    "city": "Singapore",
    "units": "metric",
    "days": 7
}
response = requests.get("https://api.weather.com/forecast", params=params)

# This calls: https://api.weather.com/forecast?city=Singapore&units=metric&days=7
```

**Adding headers (API keys, content type):**

```python
headers = {
    "Authorization": "Bearer YOUR_API_KEY",
    "Accept": "application/json"
}
response = requests.get("https://api.example.com/data", headers=headers)
```

**Timeout — ALWAYS use this:**

```python
# Never do this (can hang forever):
# response = requests.get("https://api.example.com/data")

# Always set a timeout:
response = requests.get("https://api.example.com/data", timeout=10)  # 10 seconds
```

---

### Hands-On: Call OpenMeteo Weather API (FREE, no key needed!)

OpenMeteo is a free weather API. Let's get Singapore's current weather:

```python
import requests

# Singapore coordinates: 1.3521, 103.8198
url = "https://api.open-meteo.com/v1/forecast"
params = {
    "latitude": 1.3521,
    "longitude": 103.8198,
    "current": "temperature_2m,relative_humidity_2m,wind_speed_10m",
    "timezone": "Asia/Singapore"
}

response = requests.get(url, params=params, timeout=10)
print(f"Status: {response.status_code}")

if response.ok:
    data = response.json()
    print(f"Temperature: {data['current']['temperature_2m']}°C")
    print(f"Humidity: {data['current']['relative_humidity_2m']}%")
    print(f"Wind Speed: {data['current']['wind_speed_10m']} km/h")
else:
    print(f"Error: {response.status_code}")
    print(response.text)
```

**Expected output:**
```
Status: 200
Temperature: 31.2°C
Humidity: 82%
Wind Speed: 12.5 km/h
```

---

### Hands-On: Call CoinGecko API for Crypto Prices (FREE, no key!)

```python
import requests

# Get Bitcoin and Ethereum prices in SGD
url = "https://api.coingecko.com/api/v3/simple/price"
params = {
    "ids": "bitcoin,ethereum",
    "vs_currencies": "sgd,myr",
    "include_24hr_change": "true"
}

response = requests.get(url, params=params, timeout=10)
if response.ok:
    data = response.json()
    print(f"Bitcoin: SGD ${data['bitcoin']['sgd']:,.2f}")
    print(f"Bitcoin: MYR RM{data['bitcoin']['myr']:,.2f}")
    print(f"24h change: {data['bitcoin']['sgd_24h_change']:.2f}%")
    print(f"Ethereum: SGD ${data['ethereum']['sgd']:,.2f}")
else:
    print(f"Error {response.status_code}: {response.text}")
```

---

### Morning Exercises

**Exercise 1:** Write a function `get_weather(city_name, lat, lon)` that calls the OpenMeteo API and returns a dictionary with temperature, humidity, and wind speed. Include error handling.

<details>
<summary>🔑 Answer</summary>

```python
import requests

def get_weather(city_name, lat, lon):
    """Get current weather for a city using OpenMeteo API."""
    url = "https://api.open-meteo.com/v1/forecast"
    params = {
        "latitude": lat,
        "longitude": lon,
        "current": "temperature_2m,relative_humidity_2m,wind_speed_10m",
        "timezone": "auto"
    }
    
    try:
        response = requests.get(url, params=params, timeout=10)
        response.raise_for_status()  # Raises exception for 4xx/5xx
        data = response.json()
        
        return {
            "city": city_name,
            "temperature_c": data["current"]["temperature_2m"],
            "humidity_pct": data["current"]["relative_humidity_2m"],
            "wind_speed_kmh": data["current"]["wind_speed_10m"],
            "fetched_at": data["current"]["time"]
        }
    except requests.exceptions.Timeout:
        print(f"Timeout fetching weather for {city_name}")
        return None
    except requests.exceptions.RequestException as e:
        print(f"Error fetching weather for {city_name}: {e}")
        return None

# Test it
sg = get_weather("Singapore", 1.3521, 103.8198)
kl = get_weather("Kuala Lumpur", 3.1390, 101.6869)
penang = get_weather("Penang", 5.4164, 100.3327)

for city_weather in [sg, kl, penang]:
    if city_weather:
        print(f"{city_weather['city']}: {city_weather['temperature_c']}°C, "
              f"{city_weather['humidity_pct']}% humidity")
```

</details>

**Exercise 2:** Write a function `get_crypto_prices(coin_ids, currency)` that calls CoinGecko and returns a formatted dictionary. Handle the case where CoinGecko returns a 429 (rate limited).

<details>
<summary>🔑 Answer</summary>

```python
import requests
import time

def get_crypto_prices(coin_ids, currency="sgd"):
    """Get crypto prices from CoinGecko. Handles rate limiting."""
    url = "https://api.coingecko.com/api/v3/simple/price"
    params = {
        "ids": ",".join(coin_ids),
        "vs_currencies": currency,
        "include_24hr_change": "true",
        "include_market_cap": "true"
    }
    
    max_retries = 3
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            
            if response.status_code == 429:
                wait_time = 2 ** attempt  # Exponential backoff: 1, 2, 4 seconds
                print(f"Rate limited. Waiting {wait_time}s before retry {attempt + 1}...")
                time.sleep(wait_time)
                continue
            
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.RequestException as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(2)
    
    print("All retries exhausted.")
    return None

# Test
prices = get_crypto_prices(["bitcoin", "ethereum", "solana"], "sgd")
if prices:
    for coin, data in prices.items():
        print(f"{coin.title()}: SGD ${data['sgd']:,.2f} "
              f"(24h: {data.get('sgd_24h_change', 0):.1f}%)")
```

</details>

**Exercise 3:** The OpenMeteo API returns hourly forecasts too. Write a script that gets the 7-day hourly forecast for Singapore and prints the hottest hour each day.

<details>
<summary>🔑 Answer</summary>

```python
import requests
from datetime import datetime

url = "https://api.open-meteo.com/v1/forecast"
params = {
    "latitude": 1.3521,
    "longitude": 103.8198,
    "hourly": "temperature_2m",
    "timezone": "Asia/Singapore",
    "forecast_days": 7
}

response = requests.get(url, params=params, timeout=10)
data = response.json()

times = data["hourly"]["time"]
temps = data["hourly"]["temperature_2m"]

# Group by day
daily_max = {}
for i, (t, temp) in enumerate(zip(times, temps)):
    date = t.split("T")[0]
    hour = t.split("T")[1]
    if date not in daily_max or temp > daily_max[date]["temp"]:
        daily_max[date] = {"temp": temp, "hour": hour, "datetime": t}

print("🔥 Hottest hour each day in Singapore:")
print("-" * 45)
for date in sorted(daily_max.keys()):
    info = daily_max[date]
    day_name = datetime.strptime(date, "%Y-%m-%d").strftime("%A")
    print(f"{date} ({day_name}): {info['temp']}°C at {info['hour']}")
```

</details>

**Exercise 4:** Write a function that calls an API and measures response time. Test it with 3 different APIs and print a comparison table.

<details>
<summary>🔑 Answer</summary>

```python
import requests
import time

def measure_api(url, params=None, name="API"):
    """Call an API and measure response time."""
    start = time.time()
    try:
        response = requests.get(url, params=params, timeout=10)
        elapsed = time.time() - start
        return {
            "name": name,
            "status": response.status_code,
            "time_ms": round(elapsed * 1000, 1),
            "size_kb": round(len(response.content) / 1024, 1)
        }
    except Exception as e:
        elapsed = time.time() - start
        return {
            "name": name,
            "status": "ERROR",
            "time_ms": round(elapsed * 1000, 1),
            "size_kb": 0
        }

# Test 3 APIs
results = [
    measure_api(
        "https://api.open-meteo.com/v1/forecast",
        {"latitude": 1.35, "longitude": 103.82, "current": "temperature_2m"},
        "OpenMeteo SG Weather"
    ),
    measure_api(
        "https://api.coingecko.com/api/v3/simple/price",
        {"ids": "bitcoin", "vs_currencies": "sgd"},
        "CoinGecko BTC Price"
    ),
    measure_api(
        "https://httpbin.org/get",
        name="HTTPBin Test"
    )
]

print(f"{'API':<25} {'Status':<8} {'Time (ms)':<12} {'Size (KB)':<10}")
print("-" * 55)
for r in results:
    print(f"{r['name']:<25} {str(r['status']):<8} {r['time_ms']:<12} {r['size_kb']:<10}")
```

</details>

**Exercise 5:** Create a `MakanExpressAPI` mock class that simulates an API with methods for getting restaurants, orders, and menu items. Use Python's `json` module to load mock data.

<details>
<summary>🔑 Answer</summary>

```python
import json

class MakanExpressAPI:
    """Mock API client for MakanExpress food delivery."""
    
    def __init__(self):
        self.restaurants = [
            {"id": 1, "name": "Tian Tian Hainanese Chicken Rice", "cuisine": "Chinese", "rating": 4.5, "location": "Maxwell Food Centre"},
            {"id": 2, "name": "Nasi Lemak Antarabangsa", "cuisine": "Malay", "rating": 4.3, "location": "Kuala Lumpur"},
            {"id": 3, "name": "Song Fa Bak Kut Teh", "cuisine": "Chinese", "rating": 4.4, "location": "Clarke Quay"},
            {"id": 4, "name": "Penang Road Famous Teochew Chendul", "cuisine": "Dessert", "rating": 4.6, "location": "Penang"},
        ]
        self.orders = [
            {"id": 101, "restaurant_id": 1, "items": ["Chicken Rice", "Soup"], "total": 8.50, "status": "delivered"},
            {"id": 102, "restaurant_id": 2, "items": ["Nasi Lemak", "Teh Tarik"], "total": 12.00, "status": "preparing"},
            {"id": 103, "restaurant_id": 3, "items": ["Bak Kut Teh (for 2)"], "total": 22.00, "status": "delivered"},
        ]
    
    def get_restaurants(self, cuisine=None, min_rating=0):
        """Get restaurants, optionally filtered."""
        results = self.restaurants
        if cuisine:
            results = [r for r in results if r["cuisine"].lower() == cuisine.lower()]
        results = [r for r in results if r["rating"] >= min_rating]
        return {"status": 200, "data": results, "count": len(results)}
    
    def get_orders(self, status=None):
        """Get orders, optionally filtered by status."""
        results = self.orders
        if status:
            results = [o for o in results if o["status"] == status]
        return {"status": 200, "data": results, "count": len(results)}
    
    def get_order(self, order_id):
        """Get a specific order."""
        for order in self.orders:
            if order["id"] == order_id:
                return {"status": 200, "data": order}
        return {"status": 404, "error": f"Order {order_id} not found"}

# Test the mock API
api = MakanExpressAPI()
print("All restaurants:", json.dumps(api.get_restaurants(), indent=2))
print("\nChinese restaurants:", json.dumps(api.get_restaurants(cuisine="Chinese"), indent=2))
print("\nDelivered orders:", json.dumps(api.get_orders(status="delivered"), indent=2))
print("\nOrder 999:", json.dumps(api.get_order(999), indent=2))
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Pagination, Rate Limiting, Error Handling

### Why APIs Paginate

Imagine GrabFood returning ALL 50,000 restaurants in one API call. The response would be:
- **Huge** — megabytes of JSON
- **Slow** — takes minutes to transfer
- **Expensive** — both for Grab's servers and your bandwidth
- **Useless** — you probably only need page 1

**Pagination** = the API returns data in chunks (pages). You request page 1, then page 2, etc.

---

### 3 Common Pagination Patterns

**Pattern 1: Offset-based (most common)**

```python
# Page 1: offset=0, limit=100
# Page 2: offset=100, limit=100
# Page 3: offset=200, limit=100

url = "https://api.example.com/restaurants"
page = 0
all_data = []

while True:
    params = {"offset": page * 100, "limit": 100}
    response = requests.get(url, params=params, timeout=10)
    data = response.json()
    
    if not data["results"]:  # Empty page = done
        break
    
    all_data.extend(data["results"])
    page += 1
    
    print(f"Page {page}: got {len(data['results'])} records")
```

**Pattern 2: Cursor-based (better for real-time data)**

```python
# Each response includes a "cursor" or "next_token" for the next page

url = "https://api.example.com/orders"
cursor = None
all_data = []

while True:
    params = {"limit": 100}
    if cursor:
        params["cursor"] = cursor
    
    response = requests.get(url, params=params, timeout=10)
    data = response.json()
    
    all_data.extend(data["results"])
    
    cursor = data.get("next_cursor")
    if not cursor:  # No more pages
        break
    
    print(f"Fetched {len(all_data)} records so far...")
```

**Pattern 3: Page-based (simplest)**

```python
# Page 1, 2, 3... until empty

url = "https://api.example.com/menu-items"
page = 1
all_data = []

while True:
    params = {"page": page, "per_page": 50}
    response = requests.get(url, params=params, timeout=10)
    data = response.json()
    
    if not data["items"]:
        break
    
    all_data.extend(data["items"])
    print(f"Page {page}/{data.get('total_pages', '?')}: {len(data['items'])} items")
    page += 1
```

---

### Rate Limiting: Don't Get Banned

Most APIs limit how many requests you can make per minute/hour:

| API | Free Tier Limit |
|-----|----------------|
| CoinGecko | ~10-30 requests/minute |
| OpenMeteo | ~10,000 requests/day |
| GitHub API | 60 requests/hour (unauthenticated) |

**How to handle rate limits:**

```python
import time

def api_call_with_rate_limit(url, params, requests_per_second=2):
    """Make API calls with rate limiting."""
    min_interval = 1.0 / requests_per_second  # e.g., 0.5s between calls
    
    last_call_time = getattr(api_call_with_rate_limit, '_last_call', 0)
    elapsed = time.time() - last_call_time
    
    if elapsed < min_interval:
        time.sleep(min_interval - elapsed)
    
    response = requests.get(url, params=params, timeout=10)
    api_call_with_rate_limit._last_call = time.time()
    return response
```

---

### Exponential Backoff: The Smart Retry

When you get a 429 or 5xx error, don't retry immediately. Wait longer each time:

```python
import time
import requests

def call_with_retry(url, params=None, max_retries=5, base_delay=1):
    """Call API with exponential backoff retry."""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            
            # Success!
            if response.status_code == 200:
                return response.json()
            
            # Rate limited — always retry
            if response.status_code == 429:
                delay = base_delay * (2 ** attempt)  # 1, 2, 4, 8, 16 seconds
                print(f"Rate limited (429). Retrying in {delay}s... (attempt {attempt + 1})")
                time.sleep(delay)
                continue
            
            # Client error — don't retry (it's our fault)
            if 400 <= response.status_code < 500:
                print(f"Client error {response.status_code}: {response.text}")
                return None
            
            # Server error — retry
            if response.status_code >= 500:
                delay = base_delay * (2 ** attempt)
                print(f"Server error {response.status_code}. Retrying in {delay}s...")
                time.sleep(delay)
                continue
                
        except requests.exceptions.Timeout:
            delay = base_delay * (2 ** attempt)
            print(f"Timeout. Retrying in {delay}s...")
            time.sleep(delay)
            
        except requests.exceptions.ConnectionError:
            delay = base_delay * (2 ** attempt)
            print(f"Connection error. Retrying in {delay}s...")
            time.sleep(delay)
    
    print(f"Failed after {max_retries} retries.")
    return None
```

---

### Complete Error Handling Pattern

```python
import requests
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_call(url, params=None, headers=None, max_retries=3, timeout=10):
    """
    Production-ready API call with full error handling.
    
    Returns: (data, error)
    - data: parsed JSON or None
    - error: error message or None
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(
                url, 
                params=params, 
                headers=headers, 
                timeout=timeout
            )
            
            # Successful response
            if response.status_code == 200:
                return response.json(), None
            
            # Rate limited
            if response.status_code == 429:
                retry_after = int(response.headers.get("Retry-After", 2 ** attempt))
                logger.warning(f"Rate limited. Waiting {retry_after}s...")
                time.sleep(retry_after)
                continue
            
            # Client error (4xx) — don't retry
            if 400 <= response.status_code < 500:
                error_msg = f"Client error {response.status_code}: {response.text[:200]}"
                logger.error(error_msg)
                return None, error_msg
            
            # Server error (5xx) — retry
            if response.status_code >= 500:
                delay = 2 ** attempt
                logger.warning(f"Server error {response.status_code}. Retry {attempt+1}/{max_retries} in {delay}s")
                time.sleep(delay)
                continue
                
        except requests.exceptions.Timeout:
            logger.warning(f"Timeout after {timeout}s. Retry {attempt+1}/{max_retries}")
            time.sleep(2 ** attempt)
            
        except requests.exceptions.ConnectionError:
            logger.warning(f"Connection error. Retry {attempt+1}/{max_retries}")
            time.sleep(2 ** attempt)
            
        except requests.exceptions.JSONDecodeError:
            error_msg = f"Invalid JSON response from {url}"
            logger.error(error_msg)
            return None, error_msg
            
        except Exception as e:
            error_msg = f"Unexpected error: {e}"
            logger.error(error_msg)
            return None, error_msg
    
    return None, f"Failed after {max_retries} retries"
```

---

### Build a Reusable API Client Class

```python
import requests
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class APIClient:
    """Reusable API client with pagination, rate limiting, and error handling."""
    
    def __init__(self, base_url, headers=None, rate_limit_per_sec=2, max_retries=3):
        self.base_url = base_url.rstrip("/")
        self.headers = headers or {}
        self.rate_limit_per_sec = rate_limit_per_sec
        self.max_retries = max_retries
        self._last_request_time = 0
        self._request_count = 0
    
    def _enforce_rate_limit(self):
        """Ensure we don't exceed rate limit."""
        min_interval = 1.0 / self.rate_limit_per_sec
        elapsed = time.time() - self._last_request_time
        if elapsed < min_interval:
            time.sleep(min_interval - elapsed)
    
    def _make_request(self, method, endpoint, params=None, data=None):
        """Make a single API request with retry logic."""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        
        for attempt in range(self.max_retries):
            self._enforce_rate_limit()
            
            try:
                response = requests.request(
                    method, url,
                    params=params,
                    json=data,
                    headers=self.headers,
                    timeout=10
                )
                self._last_request_time = time.time()
                self._request_count += 1
                
                if response.status_code == 200:
                    return response.json(), None
                
                if response.status_code == 429:
                    wait = 2 ** attempt
                    logger.warning(f"Rate limited. Waiting {wait}s...")
                    time.sleep(wait)
                    continue
                
                if 400 <= response.status_code < 500:
                    return None, f"Client error {response.status_code}: {response.text[:200]}"
                
                if response.status_code >= 500:
                    wait = 2 ** attempt
                    logger.warning(f"Server error. Retry in {wait}s...")
                    time.sleep(wait)
                    continue
                    
            except requests.exceptions.RequestException as e:
                logger.warning(f"Request failed: {e}. Retry {attempt+1}/{self.max_retries}")
                time.sleep(2 ** attempt)
        
        return None, f"Failed after {self.max_retries} retries"
    
    def get(self, endpoint, params=None):
        """Make a GET request."""
        return self._make_request("GET", endpoint, params=params)
    
    def post(self, endpoint, data=None):
        """Make a POST request."""
        return self._make_request("POST", endpoint, data=data)
    
    def get_all_pages(self, endpoint, params=None, page_key="page", 
                      per_page=100, data_key="data"):
        """Paginate through ALL pages of results."""
        if params is None:
            params = {}
        
        all_data = []
        page = 1
        
        while True:
            params.update({page_key: page, "per_page": per_page})
            data, error = self.get(endpoint, params=params)
            
            if error:
                logger.error(f"Error on page {page}: {error}")
                break
            
            page_data = data.get(data_key, []) if isinstance(data, dict) else data
            
            if not page_data:
                break
            
            all_data.extend(page_data)
            logger.info(f"Page {page}: {len(page_data)} records (total: {len(all_data)})")
            page += 1
            
            # Safety: stop after 100 pages
            if page > 100:
                logger.warning("Hit 100-page safety limit")
                break
        
        return all_data
    
    @property
    def stats(self):
        return {"total_requests": self._request_count}


# Usage example: CoinGecko client
coingecko = APIClient(
    base_url="https://api.coingecko.com/api/v3",
    rate_limit_per_sec=1  # CoinGecko free tier is limited
)

# Get top 10 coins by market cap
data, error = coingecko.get("/coins/markets", params={
    "vs_currency": "sgd",
    "order": "market_cap_desc",
    "per_page": 10,
    "page": 1
})

if data:
    for coin in data[:5]:
        print(f"{coin['name']}: SGD ${coin['current_price']:,.2f}")
```

---

### Afternoon Exercises

**Exercise 6:** Write a function that paginates through ALL pages of a mock API (use the `MakanExpressAPI` class from Exercise 5, but add pagination support with 2 items per page).

<details>
<summary>🔑 Answer</summary>

```python
import json

class MakanExpressPaginatedAPI:
    """MakanExpress API with pagination support."""
    
    def __init__(self):
        self.restaurants = [
            {"id": i, "name": f"Restaurant {i}", "cuisine": "Mixed", "rating": 4.0 + (i % 5) * 0.1}
            for i in range(1, 21)  # 20 restaurants
        ]
    
    def get_restaurants(self, page=1, per_page=5):
        """Get restaurants with pagination."""
        start = (page - 1) * per_page
        end = start + per_page
        page_data = self.restaurants[start:end]
        
        return {
            "data": page_data,
            "page": page,
            "per_page": per_page,
            "total": len(self.restaurants),
            "total_pages": (len(self.restaurants) + per_page - 1) // per_page,
            "has_next": end < len(self.restaurants)
        }

# Paginate through ALL restaurants
api = MakanExpressPaginatedAPI()
all_restaurants = []
page = 1

while True:
    result = api.get_restaurants(page=page, per_page=5)
    all_restaurants.extend(result["data"])
    print(f"Page {result['page']}/{result['total_pages']}: "
          f"got {len(result['data'])} restaurants")
    
    if not result["has_next"]:
        break
    page += 1

print(f"\nTotal collected: {len(all_restaurants)} restaurants")
```

</details>

**Exercise 7:** Write a rate-limited API caller that makes 10 requests to `httpbin.org/get` but never exceeds 2 requests per second. Print the time for each request.

<details>
<summary>🔑 Answer</summary>

```python
import requests
import time

def rate_limited_requests(url, count=10, requests_per_second=2):
    """Make N requests without exceeding rate limit."""
    min_interval = 1.0 / requests_per_second
    last_time = 0
    results = []
    
    for i in range(count):
        # Enforce rate limit
        now = time.time()
        elapsed = now - last_time
        if elapsed < min_interval and last_time > 0:
            sleep_time = min_interval - elapsed
            time.sleep(sleep_time)
        
        start = time.time()
        try:
            response = requests.get(url, timeout=10)
            elapsed_ms = (time.time() - start) * 1000
            results.append({"attempt": i+1, "status": response.status_code, "time_ms": round(elapsed_ms, 1)})
            print(f"Request {i+1}: {response.status_code} in {elapsed_ms:.0f}ms")
        except Exception as e:
            print(f"Request {i+1}: FAILED - {e}")
        
        last_time = time.time()
    
    total = (time.time() - results[0]["time_ms"]/1000 if results else time.time())
    print(f"\nTotal time: {sum(r['time_ms'] for r in results)/1000:.1f}s for {count} requests")
    print(f"Average: {sum(r['time_ms'] for r in results)/len(results):.0f}ms per request")
    return results

results = rate_limited_requests("https://httpbin.org/get", count=10, requests_per_second=2)
```

</details>

**Exercise 8:** Write a function that tries to call a non-existent URL and demonstrates the exponential backoff (max 3 retries). Print the wait time between retries.

<details>
<summary>🔑 Answer</summary>

```python
import requests
import time

def demonstrate_backoff(url, max_retries=3):
    """Show exponential backoff with a failing URL."""
    base_delay = 1
    
    for attempt in range(max_retries):
        try:
            print(f"Attempt {attempt + 1}: Calling {url}")
            response = requests.get(url, timeout=5)
            print(f"Success! Status: {response.status_code}")
            return response.json()
            
        except requests.exceptions.RequestException as e:
            delay = base_delay * (2 ** attempt)
            print(f"  Failed: {e}")
            if attempt < max_retries - 1:
                print(f"  Waiting {delay}s before retry...\n")
                time.sleep(delay)
            else:
                print(f"  All {max_retries} retries exhausted.")
    
    return None

# This URL intentionally doesn't exist
demonstrate_backoff("https://this-api-does-not-exist-12345.com/endpoint")
```

</details>

**Exercise 9:** Using the `APIClient` class from above, write a script that fetches weather data for 3 SG/MY cities (Singapore, KL, Penang) and compares temperatures. Handle errors for each city independently.

<details>
<summary>🔑 Answer</summary>

```python
import requests

def compare_cities_weather():
    """Compare weather across SG/MY cities."""
    cities = [
        {"name": "Singapore", "lat": 1.3521, "lon": 103.8198},
        {"name": "Kuala Lumpur", "lat": 3.1390, "lon": 101.6869},
        {"name": "Penang", "lat": 5.4164, "lon": 100.3327},
        {"name": "Johor Bahru", "lat": 1.4927, "lon": 103.7414},
    ]
    
    results = []
    
    for city in cities:
        try:
            url = "https://api.open-meteo.com/v1/forecast"
            params = {
                "latitude": city["lat"],
                "longitude": city["lon"],
                "current": "temperature_2m,relative_humidity_2m",
                "timezone": "Asia/Singapore"
            }
            
            response = requests.get(url, params=params, timeout=10)
            
            if response.ok:
                data = response.json()
                results.append({
                    "city": city["name"],
                    "temp": data["current"]["temperature_2m"],
                    "humidity": data["current"]["relative_humidity_2m"],
                    "status": "OK"
                })
            else:
                results.append({
                    "city": city["name"],
                    "temp": None,
                    "humidity": None,
                    "status": f"HTTP {response.status_code}"
                })
                
        except Exception as e:
            results.append({
                "city": city["name"],
                "temp": None,
                "humidity": None,
                "status": f"Error: {e}"
            })
    
    # Print comparison
    print(f"{'City':<20} {'Temp (°C)':<12} {'Humidity (%)':<14} {'Status'}")
    print("-" * 58)
    for r in results:
        temp_str = f"{r['temp']:.1f}" if r['temp'] is not None else "N/A"
        hum_str = f"{r['humidity']:.0f}" if r['humidity'] is not None else "N/A"
        print(f"{r['city']:<20} {temp_str:<12} {hum_str:<14} {r['status']}")
    
    # Find hottest
    valid = [r for r in results if r['temp'] is not None]
    if valid:
        hottest = max(valid, key=lambda x: x['temp'])
        coolest = min(valid, key=lambda x: x['temp'])
        print(f"\n🔥 Hottest: {hottest['city']} at {hottest['temp']:.1f}°C")
        print(f"❄️  Coolest: {coolest['city']} at {coolest['temp']:.1f}°C")

compare_cities_weather()
```

</details>

**Exercise 10:** Write a script that calls the CoinGecko API to get the top 50 cryptocurrencies by market cap (in SGD) and saves them to a CSV file. Include proper error handling and rate limiting.

<details>
<summary>🔑 Answer</summary>

```python
import requests
import csv
import time

def fetch_top_crypto(limit=50, currency="sgd"):
    """Fetch top cryptocurrencies from CoinGecko and save to CSV."""
    url = "https://api.coingecko.com/api/v3/coins/markets"
    all_coins = []
    
    # CoinGecko returns up to 250 per page
    per_page = min(limit, 250)
    
    params = {
        "vs_currency": currency,
        "order": "market_cap_desc",
        "per_page": per_page,
        "page": 1,
        "sparkline": "false"
    }
    
    try:
        time.sleep(1)  # Rate limit courtesy
        response = requests.get(url, params=params, timeout=15)
        
        if response.status_code == 429:
            print("Rate limited. Wait 60 seconds and try again.")
            return None
        
        response.raise_for_status()
        coins = response.json()
        
        # Save to CSV
        filename = f"top_{len(coins)}_crypto_{currency}.csv"
        with open(filename, "w", newline="") as f:
            writer = csv.writer(f)
            writer.writerow(["rank", "name", "symbol", f"price_{currency}", 
                           "market_cap", "24h_change_pct", "volume_24h"])
            
            for coin in coins:
                writer.writerow([
                    coin["market_cap_rank"],
                    coin["name"],
                    coin["symbol"].upper(),
                    coin["current_price"],
                    coin["market_cap"],
                    coin["price_change_percentage_24h"],
                    coin["total_volume"]
                ])
        
        print(f"✅ Saved {len(coins)} coins to {filename}")
        
        # Print top 10
        print(f"\nTop 10 Cryptocurrencies (SGD):")
        print(f"{'#':<4} {'Name':<20} {'Price (SGD)':<15} {'24h Change'}")
        print("-" * 55)
        for coin in coins[:10]:
            change = coin.get("price_change_percentage_24h", 0) or 0
            emoji = "🟢" if change >= 0 else "🔴"
            print(f"{coin['market_cap_rank']:<4} {coin['name']:<20} "
                  f"${coin['current_price']:>12,.2f}  {emoji} {change:+.1f}%")
        
        return coins
        
    except requests.exceptions.RequestException as e:
        print(f"Error: {e}")
        return None

fetch_top_crypto(50, "sgd")
```

</details>

---

## 🌙 Evening (1 hour): Quick Reference + Practice

### 🔖 API Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│               REST API QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────┤
│ HTTP METHODS                                            │
│ GET    → Read data (data engineers use this 90%)       │
│ POST   → Create data                                   │
│ PUT    → Update entire record                           │
│ DELETE → Remove data                                    │
│                                                         │
│ STATUS CODES                                            │
│ 200 = OK  |  201 = Created  |  400 = Bad Request       │
│ 401 = Unauthorized  |  403 = Forbidden                 │
│ 404 = Not Found  |  429 = Rate Limited                 │
│ 500 = Server Error  |  503 = Unavailable               │
│                                                         │
│ PYTHON REQUESTS                                         │
│ response = requests.get(url, params={}, timeout=10)    │
│ response.ok          → True if status < 400            │
│ response.status_code → 200, 404, etc.                  │
│ response.json()      → Parse JSON → dict               │
│ response.text        → Raw text                        │
│ response.headers     → Response headers                │
│                                                         │
│ ERROR HANDLING                                          │
│ try/except with:                                        │
│   requests.exceptions.Timeout                           │
│   requests.exceptions.ConnectionError                   │
│   requests.exceptions.RequestException (catch-all)      │
│                                                         │
│ PAGINATION                                              │
│ Offset:   ?offset=0&limit=100 → ?offset=100&limit=100 │
│ Cursor:   ?cursor=abc123   → ?cursor=def456           │
│ Page:     ?page=1&per_page=50 → ?page=2&per_page=50   │
│                                                         │
│ RATE LIMITING                                           │
│ - Always check API docs for limits                      │
│ - Use time.sleep() between requests                    │
│ - Exponential backoff: 1s, 2s, 4s, 8s, 16s            │
│ - Check Retry-After header on 429 responses            │
│                                                         │
│ ALWAYS:                                                 │
│ ✅ Set timeout                                          │
│ ✅ Handle errors                                        │
│ ✅ Respect rate limits                                  │
│ ✅ Log failures                                         │
│ ✅ Use pagination for large datasets                    │
└─────────────────────────────────────────────────────────┘
```

---

### Practice Problems

**Problem 1:** Call the OpenMeteo API for both Singapore and KL. Calculate the temperature difference. If either call fails, show "Data unavailable" for that city.

**Problem 2:** Simulate pagination: create a list of 100 MakanExpress orders and write a function that returns them in pages of 20. Write a consumer that collects all pages.

**Problem 3:** Write a function that calls any URL 5 times with exponential backoff simulation (don't actually sleep — just print what the delay would be). Show the total wait time.

---

### 📝 Today's Checklist

- [ ] Understand what REST APIs are and why data engineers use them
- [ ] Can explain HTTP methods (GET/POST/PUT/DELETE) and status codes
- [ ] Successfully called OpenMeteo API for Singapore weather
- [ ] Successfully called CoinGecko API for crypto prices
- [ ] Understand JSON format and can parse nested JSON
- [ ] Can implement pagination (offset, cursor, page-based)
- [ ] Can implement rate limiting with time.sleep()
- [ ] Can implement exponential backoff for retries
- [ ] Built a reusable APIClient class
- [ ] Completed at least 7 of 10 exercises
- [ ] Ready for tomorrow: Build a complete API data pipeline

---

*Day 57 complete! Tomorrow: Build a full API data pipeline — extract from multiple APIs, transform, load into PostgreSQL and S3.* 🚀
