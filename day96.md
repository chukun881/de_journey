# 📅 Day 96 — Monday, 18 August 2026
# 🎯 Mock Interview #4: Full Loop (SQL + Python + System Design + Behavioral)

---

## 🎯 Today's Goal

The ultimate practice: a full 2-hour interview loop covering all 4 areas. This simulates a real interview day at a Singapore tech company. No notes, no breaks between sections.

---

## ☀️ Morning Block (2 hours): Full Interview Loop

### Section 1: SQL (25 minutes)

**Q1 (8 min):** Find the top 3 drivers by total earnings in each city. Show driver name, city, total earnings, and number of completed rides.

**Q2 (8 min):** Calculate month-over-month revenue growth for each restaurant. Flag restaurants with negative growth for 2+ consecutive months.

**Q3 (9 min):** Build a customer cohort analysis: group by signup month, show how many ordered in month 0, 1, 2, 3 after signup.

<details>
<summary>🔑 SQL Answers</summary>

```sql
-- Q1
WITH driver_earnings AS (
    SELECT d.name, d.city, dr.id as driver_id,
           SUM(r.fare) as total_earnings,
           COUNT(*) as completed_rides,
           ROW_NUMBER() OVER (PARTITION BY d.city ORDER BY SUM(r.fare) DESC) as city_rank
    FROM rides r
    JOIN drivers d ON r.driver_id = d.id
    WHERE r.status = 'completed'
    GROUP BY d.id, d.name, d.city
)
SELECT name, city, total_earnings, completed_rides
FROM driver_earnings
WHERE city_rank <= 3
ORDER BY city, city_rank;

-- Q2
WITH monthly AS (
    SELECT restaurant_id, DATE_TRUNC('month', order_date) as month,
           SUM(total_amount) as revenue
    FROM orders GROUP BY 1, 2
),
with_growth AS (
    SELECT restaurant_id, month, revenue,
           LAG(revenue) OVER (PARTITION BY restaurant_id ORDER BY month) as prev_month,
           SIGN(revenue - LAG(revenue) OVER (PARTITION BY restaurant_id ORDER BY month)) as direction
    FROM monthly
)
SELECT restaurant_id,
       SUM(CASE WHEN direction = -1 THEN 1 ELSE 0 END)
           OVER (PARTITION BY restaurant_id ORDER BY month ROWS BETWEEN 1 PRECEDING AND CURRENT ROW) as consecutive_decline
FROM with_growth;

-- Q3
WITH cohort AS (
    SELECT customer_id, DATE_TRUNC('month', MIN(order_date)) as cohort_month
    FROM orders GROUP BY customer_id
),
activity AS (
    SELECT c.customer_id, c.cohort_month,
           DATE_TRUNC('month', o.order_date) as active_month,
           EXTRACT(YEAR FROM o.order_date)*12 + EXTRACT(MONTH FROM o.order_date) -
           EXTRACT(YEAR FROM c.cohort_month)*12 - EXTRACT(MONTH FROM c.cohort_month) as month_n
    FROM cohort c JOIN orders o ON c.customer_id = o.customer_id
)
SELECT cohort_month,
       COUNT(DISTINCT CASE WHEN month_n = 0 THEN customer_id END) as m0,
       COUNT(DISTINCT CASE WHEN month_n = 1 THEN customer_id END) as m1,
       COUNT(DISTINCT CASE WHEN month_n = 2 THEN customer_id END) as m2,
       COUNT(DISTINCT CASE WHEN month_n = 3 THEN customer_id END) as m3
FROM activity
GROUP BY cohort_month
ORDER BY cohort_month;
```
</details>

### Section 2: Python (20 minutes)

**Q4 (10 min):** Write a function `etl_daily_summary(csv_path, output_path)` that:
- Reads daily ride data
- Validates (positive fare, non-null IDs, valid city)
- Groups by city + date
- Calculates: count, sum fare, avg fare, avg rating
- Writes to CSV

**Q5 (10 min):** Write a function `detect_fraud(rides_df)` that flags suspicious rides:
- Fare > 3x the city average
- Duration > 2 hours
- Same rider + driver pairing > 5 times in one day

<details>
<summary>🔑 Python Answers</summary>

```python
import pandas as pd

def etl_daily_summary(csv_path, output_path):
    df = pd.read_csv(csv_path)
    # Validate
    df = df.dropna(subset=['id', 'rider_id', 'driver_id'])
    df = df[df['fare'] > 0]
    valid_cities = {'Singapore', 'Kuala Lumpur', 'Penang', 'Johor Bahru'}
    df = df[df['city'].isin(valid_cities)]
    
    df['date'] = pd.to_datetime(df['pickup_time']).dt.date
    
    summary = df.groupby(['city', 'date']).agg(
        ride_count=('id', 'count'),
        total_fare=('fare', 'sum'),
        avg_fare=('fare', 'mean'),
        avg_rating=('rating', 'mean')
    ).round(2).reset_index()
    
    summary.to_csv(output_path, index=False)
    return summary

def detect_fraud(rides_df):
    suspicious = pd.DataFrame()
    city_avg = rides_df.groupby('city')['fare'].transform('mean')
    suspicious = pd.concat([suspicious, rides_df[rides_df['fare'] > 3 * city_avg]])
    suspicious = pd.concat([suspicious, rides_df[rides_df['duration_min'] > 120]])
    
    pair_counts = rides_df.groupby(['rider_id', 'driver_id', pd.to_datetime(rides_df['pickup_time']).dt.date]).size()
    frequent_pairs = pair_counts[pair_counts > 5].reset_index()[['rider_id', 'driver_id']]
    frequent_rides = rides_df.merge(frequent_pairs, on=['rider_id', 'driver_id'])
    suspicious = pd.concat([suspicious, frequent_rides]).drop_duplicates()
    
    suspicious['fraud_reason'] = 'Multiple indicators'
    return suspicious
```
</details>

### Section 3: System Design (20 minutes)

**Q6:** "Design a notification system for GrabFood — notify customers when their order status changes (placed → preparing → on the way → delivered)."

Discuss: architecture, message queue, delivery guarantees, scaling.

<details>
<summary>🔑 Model Answer</summary>

```
Architecture:
  Order Service → EventBridge/SNS → Lambda → Push notification (FCM/APNS)
                                  → SMS (Twilio)
                                  → Email (SES)

Key design decisions:
1. Event-driven: Each status change is an event
2. Message queue (SNS/SQS): Decouples order service from notification service
3. At-least-once delivery: Customers might get duplicate notifications (better than missing one)
4. Idempotency: Each notification has an event_id, client deduplicates
5. Priority: Push notification (fast) + SMS (fallback) + Email (digest)

Scaling:
  - Lambda auto-scales per event
  - SNS handles millions of messages
  - Store notification history in DynamoDB for debugging

Monitoring:
  - CloudWatch for delivery success rate
  - Alert if delivery rate drops below 99%
```
</details>

### Section 4: Behavioral (15 minutes)

Answer these 3 questions using STAR method:

**Q7.** "Why do you want to be a data engineer?"

**Q8.** "Tell me about a time you had to learn something completely new."

**Q9.** "Where do you see yourself in 2 years?"

---

## 🌤️ Afternoon Block (2 hours): Grade + Review

### Scoring

| Section | Points | Your Score |
|---------|--------|-----------|
| SQL (Q1-Q3) | 30 | |
| Python (Q4-Q5) | 25 | |
| System Design (Q6) | 25 | |
| Behavioral (Q7-Q9) | 20 | |
| **Total** | **100** | |

### 📝 Today's Checklist

- [ ] Completed 2-hour mock interview (no notes)
- [ ] Self-graded: __/100
- [ ] Identified remaining weak spots
- [ ] Applied to 5+ positions today
- [ ] Updated tracking spreadsheet

---

*Day 96 complete! Start applying!* 🚀
