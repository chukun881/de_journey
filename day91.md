# 📅 Day 91 — Wednesday, 13 August 2026
# 🔄 Review Weak Spots from Mock Interviews

---

## 🎯 Today's Goal

Based on yesterday's mock interview results, you know your weak spots. Today you attack them directly — re-do problems you got wrong, practice patterns that tripped you up, and build confidence before real interviews.

---

## ☀️ Morning Block (2 hours): Re-do Yesterday's Mistakes

### Step 1: List Your Mistakes (10 minutes)

Go through yesterday's mock interview. For each problem you got wrong or struggled with:

| Problem | What went wrong | Category |
|---------|----------------|----------|
| e.g., Q2 (SQL) | Forgot how to count distinct months | Window functions |
| e.g., Q5 (Python) | Z-score formula wrong | Statistics |
| e.g., Q6 (Design) | Didn't ask about data volume | Communication |

### Step 2: Re-do Each Mistake (90 minutes)

For each mistake:
1. Read the correct answer carefully
2. Close it
3. Re-do the problem from scratch
4. Check against the answer
5. If still wrong, repeat

**If SQL is weak:** Review these days:
- Basic: Day 1, Day 2
- Intermediate: Day 3, Day 4, Day 22
- Window functions: Day 26, Day 87

**If Python is weak:** Review:
- Basics: Day 8, Day 9, Day 10
- Pandas: Day 12, Day 13
- ETL patterns: Day 88

**If System Design is weak:** Review:
- Day 74, Day 89

---

## 🌤️ Afternoon Block (2 hours): Additional Practice (10 Problems)

### Practice in Your Weakest Area

Pick your weakest area and do 10 more problems:

**If SQL is weakest:**

**P1.** Find the 90th percentile of order amounts per city.

**P2.** Calculate rolling 7-day retention: % of users who returned within 7 days of their first order.

**P3.** Find the longest consecutive streak of daily orders for each restaurant.

**P4.** Build a cohort analysis: group customers by signup month, show % active each subsequent month.

**P5.** Find restaurants where weekend revenue > weekday revenue.

**P6.** For each customer, calculate the average time between their 1st and 2nd, 2nd and 3rd order.

**P7.** Find the "second best" cuisine per city (rank 2 by revenue).

**P8.** Identify seasonal trends: quarter-over-quarter growth per city.

**P9.** Find customers whose spending increased every month for 4+ months.

**P10.** Create a recommendation: for each customer, find the most popular cuisine they haven't tried.

<details>
<summary>🔑 SQL Practice Answers</summary>

```sql
-- P1
SELECT city,
       PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY total_amount) as p90
FROM orders o JOIN customers c ON o.customer_id = c.id
GROUP BY city;

-- P2
WITH first_order AS (
    SELECT customer_id, MIN(order_date) as first_date
    FROM orders GROUP BY customer_id
),
returned AS (
    SELECT f.customer_id
    FROM first_order f
    JOIN orders o ON f.customer_id = o.customer_id
        AND o.order_date > f.first_date
        AND o.order_date <= f.first_date + INTERVAL '7 days'
)
SELECT DATE_TRUNC('week', f.first_date) as cohort_week,
       COUNT(DISTINCT f.customer_id) as total_new,
       COUNT(DISTINCT r.customer_id) as returned_7d,
       ROUND(100.0 * COUNT(DISTINCT r.customer_id) / COUNT(DISTINCT f.customer_id), 2) as retention_pct
FROM first_order f
LEFT JOIN returned r ON f.customer_id = r.customer_id
GROUP BY cohort_week
ORDER BY cohort_week;

-- P3
WITH daily AS (
    SELECT DISTINCT restaurant_id, DATE(order_date) as order_date
    FROM orders
),
with_grp AS (
    SELECT restaurant_id, order_date,
           DATE(order_date) - ROW_NUMBER() OVER (PARTITION BY restaurant_id ORDER BY order_date)::int as grp
    FROM daily
)
SELECT restaurant_id, MAX(streak) as longest_streak
FROM (
    SELECT restaurant_id, grp, COUNT(*) as streak
    FROM with_grp GROUP BY restaurant_id, grp
) t
GROUP BY restaurant_id
ORDER BY longest_streak DESC;

-- P4 (simplified)
WITH cohort AS (
    SELECT customer_id, DATE_TRUNC('month', MIN(order_date)) as cohort_month
    FROM orders GROUP BY customer_id
),
activity AS (
    SELECT c.customer_id, c.cohort_month,
         DATE_TRUNC('month', o.order_date) as active_month,
         EXTRACT(YEAR FROM o.order_date)*12 + EXTRACT(MONTH FROM o.order_date) -
         EXTRACT(YEAR FROM c.cohort_month)*12 - EXTRACT(MONTH FROM c.cohort_month) as month_n
    FROM cohort c
    JOIN orders o ON c.customer_id = o.customer_id
)
SELECT cohort_month, month_n,
       COUNT(DISTINCT customer_id) as active_users
FROM activity
GROUP BY cohort_month, month_n
ORDER BY cohort_month, month_n;

-- P5
WITH daily_rev AS (
    SELECT restaurant_id,
           CASE WHEN EXTRACT(DOW FROM order_date) IN (0, 6) THEN 'weekend' ELSE 'weekday' END as day_type,
           SUM(total_amount) as revenue
    FROM orders GROUP BY 1, 2
)
SELECT restaurant_id,
       SUM(CASE WHEN day_type='weekend' THEN revenue ELSE 0 END) as weekend_rev,
       SUM(CASE WHEN day_type='weekday' THEN revenue ELSE 0 END) as weekday_rev
FROM daily_rev
GROUP BY restaurant_id
HAVING SUM(CASE WHEN day_type='weekend' THEN revenue ELSE 0 END) >
       SUM(CASE WHEN day_type='weekday' THEN revenue ELSE 0 END);

-- P10
WITH cust_cuisines AS (
    SELECT DISTINCT o.customer_id, r.cuisine
    FROM orders o JOIN restaurants r ON o.restaurant_id = r.id
),
popular AS (
    SELECT r.cuisine, COUNT(*) as order_count,
           RANK() OVER (ORDER BY COUNT(*) DESC) as popularity_rank
    FROM orders o JOIN restaurants r ON o.restaurant_id = r.id
    GROUP BY r.cuisine
)
SELECT cc.customer_id,
       (SELECT cuisine FROM popular WHERE cuisine NOT IN (
           SELECT cuisine FROM cust_cuisines WHERE customer_id = cc.customer_id
       ) ORDER BY popularity_rank LIMIT 1) as recommended_cuisine
FROM (SELECT DISTINCT customer_id FROM cust_cuisines) cc;
```
</details>

**If Python is weakest:**

**P1.** Write a function to merge two DataFrames and handle conflicting columns.

**P2.** Write a function that reads a large CSV in chunks (100K rows at a time) and filters.

**P3.** Write a retry decorator that retries a function on failure.

**P4.** Write a data class `Order` with validation (amount > 0, valid status).

**P5.** Write a function to calculate running totals without Pandas (pure Python).

**P6.** Write a generator that yields CSV rows one at a time (memory efficient).

**P7.** Write a function to compare two DataFrames and return the diff.

**P8.** Write a connection pool manager for database connections.

**P9.** Write a function that normalizes column names across multiple datasets.

**P10.** Write a simple logging decorator that logs function calls, args, and execution time.

<details>
<summary>🔑 Python Practice Answers</summary>

```python
# P1
def safe_merge(left, right, on, how='left', suffixes=('_left', '_right')):
    merged = pd.merge(left, right, on=on, how=how, suffixes=suffixes)
    return merged

# P2
def process_large_csv(path, filter_func, chunksize=100000):
    results = []
    for chunk in pd.read_csv(path, chunksize=chunksize):
        filtered = filter_func(chunk)
        results.append(filtered)
    return pd.concat(results, ignore_index=True)

# P3
import functools, time

def retry(max_retries=3, backoff=2):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    wait = backoff ** attempt
                    print(f"Attempt {attempt+1} failed: {e}. Retrying in {wait}s...")
                    time.sleep(wait)
        return wrapper
    return decorator

# P4
from dataclasses import dataclass
from typing import Optional

@dataclass
class Order:
    id: int
    customer_id: int
    amount: float
    status: str
    
    def __post_init__(self):
        if self.amount <= 0:
            raise ValueError(f"Amount must be positive, got {self.amount}")
        valid_statuses = {'pending', 'confirmed', 'delivered', 'cancelled'}
        if self.status not in valid_statuses:
            raise ValueError(f"Invalid status: {self.status}")

# P5
def running_total(values):
    total = 0
    result = []
    for v in values:
        total += v
        result.append(total)
    return result

# P6
import csv

def read_csv_generator(path):
    with open(path, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield row

# P10
import functools, time, logging

logger = logging.getLogger(__name__)

def log_execution(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        logger.info(f"{func.__name__} took {elapsed:.3f}s")
        return result
    return wrapper
```
</details>

---

## 🌙 Evening (1 hour): Final Interview Prep Checklist

### ✅ The "Are You Ready?" Checklist

**Technical:**
- [ ] Can write complex SQL queries (JOINs, CTEs, window functions) in <3 min each
- [ ] Can write Python ETL functions with error handling in <10 min
- [ ] Can walk through a system design scenario in 30 min
- [ ] Know the STAR method for behavioral questions

**Portfolio:**
- [ ] 5 GitHub repos polished with READMEs, CI badges, architecture diagrams
- [ ] GitHub Pages portfolio live
- [ ] GitHub profile optimized (bio, pinned repos)

**Application:**
- [ ] Resume saved as PDF (1 page)
- [ ] LinkedIn optimized ("Open to Work" enabled)
- [ ] Cover letter template ready
- [ ] Target company list (20+ companies)

**Mindset:**
- [ ] It's OK to not know everything — show your thinking process
- [ ] Ask clarifying questions before diving in
- [ ] "I haven't used X directly, but conceptually..." is a valid answer
- [ ] Communication > Memorization

### 💪 Confidence Boost

You've spent 13 weeks building real skills:
- 5 portfolio projects on GitHub
- 105 days of structured learning
- 100+ SQL problems practiced
- 30+ Python problems practiced
- 5 system design scenarios practiced
- Resume + LinkedIn + portfolio ready

**You ARE ready.** The interview is just showing them what you can already do.

### 📝 Today's Checklist

- [ ] Re-did all problems from Day 90 that I got wrong
- [ ] Completed 10 additional practice problems in weakest area
- [ ] Score improved on re-test
- [ ] Final checklist all ✅
- [ ] Ready for Week 14: More interview prep + start applying!

---

*Week 13 complete! Heavy interview prep done. Week 14-15: Apply everywhere.* 🚀
