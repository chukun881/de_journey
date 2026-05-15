# 📅 Day 90 — Tuesday, 12 August 2026
# 🎯 Mock Interview #3: Full Technical Round

---

## 🎯 Today's Goal

This is the real deal — a full simulated technical interview just like you'd get at a Singapore data engineering role. 90 minutes covering SQL, Python, and system design. No notes, no Googling. Time yourself.

---

## ☀️ Morning Block (2 hours): Full Technical Interview

### Section 1: SQL Live Coding (30 minutes, 3 problems)

**Problem 1 (Easy — 8 minutes)**

Using tables:
```sql
-- riders: id, name, city, signup_date
-- rides: id, rider_id, driver_id, pickup_time, dropoff_time, fare, city, status
```

Write a query to find the top 3 riders by total fare spent in Singapore in July 2026. Show rider name, total fare, and number of rides.

<details>
<summary>🔑 Answer</summary>

```sql
SELECT r.name, SUM(rd.fare) as total_fare, COUNT(*) as ride_count
FROM riders r
JOIN rides rd ON r.id = rd.rider_id
WHERE rd.city = 'Singapore'
  AND rd.status = 'completed'
  AND rd.pickup_time >= '2026-07-01'
  AND rd.pickup_time < '2026-08-01'
GROUP BY r.id, r.name
ORDER BY total_fare DESC
LIMIT 3;
```
</details>

**Problem 2 (Medium — 10 minutes)**

Find riders who took at least one ride in every month of 2026 (so far). Show rider name and count of distinct months.

<details>
<summary>🔑 Answer</summary>

```sql
WITH monthly_rides AS (
    SELECT rider_id, DATE_TRUNC('month', pickup_time) as month
    FROM rides
    WHERE pickup_time >= '2026-01-01'
      AND status = 'completed'
    GROUP BY rider_id, DATE_TRUNC('month', pickup_time)
),
target_months AS (
    SELECT COUNT(DISTINCT DATE_TRUNC('month', dt)) as total_months
    FROM generate_series('2026-01-01'::date, CURRENT_DATE, '1 month'::interval) dt
)
SELECT r.name, COUNT(mr.month) as active_months
FROM monthly_rides mr
JOIN riders r ON mr.rider_id = r.id
CROSS JOIN target_months tm
GROUP BY r.id, r.name, tm.total_months
HAVING COUNT(mr.month) = tm.total_months;
```
</details>

**Problem 3 (Hard — 12 minutes)**

For each city, find the driver with the highest average rating. Also show their average fare and number of rides. Handle the case where multiple drivers tie.

<details>
<summary>🔑 Answer</summary>

```sql
WITH driver_stats AS (
    SELECT rd.driver_id, rd.city,
           AVG(rd.rating) as avg_rating,
           AVG(rd.fare) as avg_fare,
           COUNT(*) as ride_count
    FROM rides rd
    WHERE rd.status = 'completed' AND rd.rating IS NOT NULL
    GROUP BY rd.driver_id, rd.city
),
ranked AS (
    SELECT ds.driver_id, ds.city, d.name, ds.avg_rating, ds.avg_fare, ds.ride_count,
           RANK() OVER (PARTITION BY ds.city ORDER BY ds.avg_rating DESC) as city_rank
    FROM driver_stats ds
    JOIN drivers d ON ds.driver_id = d.id
)
SELECT city, name, ROUND(avg_rating, 2) as avg_rating,
       ROUND(avg_fare, 2) as avg_fare, ride_count
FROM ranked
WHERE city_rank = 1
ORDER BY city;
```
</details>

---

### Section 2: Python ETL Coding (30 minutes, 2 problems)

**Problem 4 (15 minutes)**

Write a Python function that:
1. Reads a CSV of ride data
2. Cleans it (remove null IDs, validate fare > 0, normalize city names)
3. Groups by city and date
4. Calculates: total rides, total fare, avg fare, median fare
5. Returns a DataFrame sorted by date

<details>
<summary>🔑 Answer</summary>

```python
import pandas as pd
import numpy as np

CITY_MAPPING = {
    'sg': 'Singapore', 'singapore': 'Singapore', 'sgp': 'Singapore',
    'kl': 'Kuala Lumpur', 'kuala lumpur': 'Kuala Lumpur',
    'pg': 'Penang', 'penang': 'Penang',
    'jb': 'Johor Bahru', 'johor bahru': 'Johor Bahru',
}

def process_rides(csv_path: str) -> pd.DataFrame:
    """Process ride data: clean, validate, aggregate by city and date."""
    # Read
    df = pd.read_csv(csv_path)
    
    # Clean: remove null IDs
    df = df.dropna(subset=['id'])
    
    # Validate fare
    df = df[df['fare'] > 0]
    
    # Normalize city names
    df['city'] = df['city'].str.strip().str.lower().map(CITY_MAPPING).fillna('Unknown')
    
    # Parse date
    df['date'] = pd.to_datetime(df['pickup_time']).dt.date
    
    # Aggregate
    result = df.groupby(['city', 'date']).agg(
        total_rides=('id', 'count'),
        total_fare=('fare', 'sum'),
        avg_fare=('fare', 'mean'),
        median_fare=('fare', 'median')
    ).reset_index()
    
    # Round
    result['total_fare'] = result['total_fare'].round(2)
    result['avg_fare'] = result['avg_fare'].round(2)
    result['median_fare'] = result['median_fare'].round(2)
    
    return result.sort_values('date')
```
</details>

**Problem 5 (15 minutes)**

Write a function `detect_anomalies(df)` that:
1. Calculates the mean and std dev of daily ride counts per city
2. Flags days where the count is >2 std devs from the mean
3. Returns a DataFrame with flagged rows and the z-score

<details>
<summary>🔑 Answer</summary>

```python
def detect_anomalies(df: pd.DataFrame) -> pd.DataFrame:
    """Detect anomalous ride counts per city (>2 std devs from mean)."""
    # Calculate daily counts per city
    daily = df.groupby(['city', pd.to_datetime(df['pickup_time']).dt.date]).size().reset_index(name='ride_count')
    daily.columns = ['city', 'date', 'ride_count']
    
    # Calculate stats per city
    stats = daily.groupby('city')['ride_count'].agg(['mean', 'std']).reset_index()
    stats.columns = ['city', 'mean_count', 'std_count']
    
    # Merge and calculate z-score
    result = daily.merge(stats, on='city')
    result['z_score'] = (result['ride_count'] - result['mean_count']) / result['std_count']
    result['is_anomaly'] = result['z_score'].abs() > 2
    
    # Return only anomalies
    anomalies = result[result['is_anomaly']].copy()
    anomalies = anomalies[['city', 'date', 'ride_count', 'mean_count', 'z_score']]
    
    return anomalies.sort_values('z_score', key=abs, ascending=False)
```
</details>

---

### Section 3: System Design (30 minutes, 1 scenario)

**Problem 6: "Design a ride-sharing analytics platform"**

You have 30 minutes. Cover:
1. Clarifying questions (5 min)
2. Architecture diagram (10 min)
3. Schema design (10 min)
4. Tradeoffs + scaling (5 min)

<details>
<summary>🔑 Model Answer</summary>

**Clarifying Questions:**
- What analytics? Driver performance, rider behavior, revenue, city-level metrics?
- Data volume? (assume: 1M rides/day across SEA)
- Real-time or batch? (assume: daily batch + near-real-time dashboard)
- Who uses it? (analysts, ops team, executives)

**Architecture:**
```
Sources:
  - Rides DB (PostgreSQL): ride events, fare, ratings
  - Driver API: location, status, earnings
  - Rider API: profiles, preferences
  - Payment gateway: transactions

Ingestion (Airflow, 2 AM SGT):
  - Extract rides (incremental, last 24h)
  - Extract driver snapshots
  - Extract payment settlements
  - Load to S3 raw zone

Storage (S3 Data Lake):
  - raw/rides/ (JSON, partitioned by date)
  - raw/drivers/ (CSV snapshots)
  - raw/payments/ (CSV settlement files)
  - processed/ (Parquet, optimized)

Transform (dbt on Redshift/Athena):
  - staging: clean, cast, deduplicate
  - intermediate: join rides + payments + drivers
  - marts: star schema

Star Schema:
  fact_rides: ride_key, date_key, rider_key, driver_key, city_key
              fare, distance_km, duration_min, rating
  dim_rider: rider_key, id, name, city, segment, signup_date
  dim_driver: driver_key, id, name, city, vehicle, rating
  dim_city: city_key, name, country, population
  dim_date: date_key, date, day, month, quarter, is_holiday

Serving:
  - Redshift/Athena for ad-hoc queries
  - Metabase/QuickSight for dashboards
  - S3 for data science exports

Scaling (if 10x to 10M rides/day):
  - Partition by date + city
  - Use Glue/Spark for heavy transforms
  - Consider Snowflake (auto-scaling)
  - Add streaming layer for real-time
```
</details>

---

## 🌤️ Afternoon Block (2 hours): Self-Grading + Model Answers

### Grading Rubric

**SQL (30 points):**
| Problem | Points | Criteria |
|---------|--------|----------|
| Problem 1 | 8 | Correct JOIN, WHERE, GROUP BY, ORDER BY |
| Problem 2 | 10 | CTE/subquery, DISTINCT months, HAVING logic |
| Problem 3 | 12 | Window function, multiple joins, RANK, handling ties |

**Python (30 points):**
| Problem | Points | Criteria |
|---------|--------|----------|
| Problem 4 | 15 | Clean code, handles edge cases, correct aggregation |
| Problem 5 | 15 | Correct z-score, grouping, anomaly detection |

**System Design (40 points):**
| Criteria | Points |
|----------|--------|
| Asked clarifying questions | 5 |
| Architecture makes sense | 10 |
| Correct schema design | 10 |
| Named specific tools with reasons | 10 |
| Discussed tradeoffs + scaling | 5 |

### Total Score Interpretation

| Score | Assessment |
|-------|-----------|
| 85-100 | 🔥 Interview-ready, go apply! |
| 70-84 | ✅ Strong, review weak spots |
| 50-69 | ⚠️ More practice needed |
| <50 | 🔴 Focus on fundamentals |

---

## 🌙 Evening (1 hour): Weak Spot Plan

### Identify + Plan

For each section you scored <70%:
1. List the specific skill gap
2. Find which day in the study plan covers it
3. Schedule 1 hour tomorrow to review + re-practice

### 📝 Today's Checklist

- [ ] Completed 90-minute mock interview (no notes)
- [ ] Self-graded honestly using rubric
- [ ] SQL score: __/30
- [ ] Python score: __/30
- [ ] System design score: __/40
- [ ] Total: __/100
- [ ] Identified top 3 weak spots
- [ ] Plan created for tomorrow's review

---

*Day 90 complete! Tomorrow: Review weak spots.* 🔄
