# 📅 Day 70 — Wednesday, 23 July 2026
# 📊 Week 10 Review + Weak Spot Targeting

---

## 🎯 Today's Goal

Week 10 is done. You've covered data quality, testing patterns, and 60 SQL interview problems. Today you review your performance, target weak spots, and create a practice plan for the remaining weeks.

---

## ☀️ Morning Block (2 hours): Self-Assessment + Targeted Practice

### Week 10 Skills Assessment

Rate yourself 1-5 for each skill:

| Skill | Rating | Notes |
|-------|--------|-------|
| Data quality check types (10 types) | ? | |
| Great Expectations usage | ? | |
| Custom Python DQ checks | ? | |
| dbt testing (schema.yml, custom SQL) | ? | |
| Schema validation | ? | |
| Null analysis and completeness | ? | |
| Distribution monitoring | ? | |
| Freshness monitoring | ? | |
| SQL: Basic queries (GROUP BY, HAVING) | ? | |
| SQL: JOINs (all types) | ? | |
| SQL: Window functions (RANK, LAG, LEAD, NTILE) | ? | |
| SQL: CTEs and subqueries | ? | |
| SQL: Date manipulation | ? | |
| SQL: Complex aggregations | ? | |
| Mock interview: explaining projects | ? | |

**Anything rated 1-3 needs targeted practice.**

---

### Targeted Practice Templates

**If weak at Window Functions (Day 67-68 problems were hard):**

Practice these 5 patterns until they're muscle memory:

```sql
-- 1. ROW_NUMBER: rank items within groups
SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY amount DESC) as rn;

-- 2. RANK/DENSE_RANK: ranking with ties
SELECT *, DENSE_RANK() OVER (ORDER BY score DESC) as rank;

-- 3. LAG/LEAD: compare to previous/next row
SELECT *, LAG(amount) OVER (ORDER BY date) as prev_amount;

-- 4. Running total
SELECT *, SUM(amount) OVER (ORDER BY date) as running_total;

-- 5. Moving average
SELECT *, AVG(amount) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as ma7;
```

**If weak at Data Quality concepts:**

Re-read Day 64 morning and build a DQ checker from scratch (no notes).

**If weak at Mock Interview:**

Practice explaining your MakanExpress project to a friend or record yourself. Target: 2-minute explanation covering architecture, tech stack, and results.

---

### Practice Problems for Weak Spots

**SQL Weak Spot Drill (repeat until fast):**

1. Write a query that finds the 2nd highest spender per city
2. Write a query showing month-over-month growth rate per cuisine
3. Write a query finding customers who ordered every day this week
4. Write a query calculating percentile rankings for restaurant ratings
5. Write a query finding the longest gap between orders for each customer

<details>
<summary>🔑 Answers</summary>

```sql
-- 1. 2nd highest spender per city
WITH ranked AS (
    SELECT delivery_city, customer_id, SUM(total_amount) as spent,
        DENSE_RANK() OVER (PARTITION BY delivery_city ORDER BY SUM(total_amount) DESC) as rn
    FROM orders WHERE status = 'delivered'
    GROUP BY delivery_city, customer_id
)
SELECT * FROM ranked WHERE rn = 2;

-- 2. MoM growth per cuisine
WITH monthly AS (
    SELECT r.cuisine, DATE_TRUNC('month', o.order_date) as m, SUM(o.total_amount) as rev
    FROM orders o JOIN restaurants r ON o.restaurant_id = r.id
    WHERE o.status = 'delivered' GROUP BY r.cuisine, DATE_TRUNC('month', o.order_date)
)
SELECT cuisine, m, rev,
    LAG(rev) OVER (PARTITION BY cuisine ORDER BY m) as prev,
    ROUND((rev - LAG(rev) OVER (PARTITION BY cuisine ORDER BY m)) / 
          LAG(rev) OVER (PARTITION BY cuisine ORDER BY m) * 100, 1) as growth_pct
FROM monthly;

-- 3. Ordered every day this week
SELECT customer_id
FROM orders
WHERE order_date >= DATE_TRUNC('week', CURRENT_DATE)
  AND status = 'delivered'
GROUP BY customer_id
HAVING COUNT(DISTINCT DATE(order_date)) = EXTRACT(DOW FROM CURRENT_DATE) + 1;

-- 4. Percentile rankings
SELECT name, rating,
    PERCENT_RANK() OVER (ORDER BY rating) as pct_rank,
    NTILE(4) OVER (ORDER BY rating) as quartile
FROM restaurants;

-- 5. Longest gap between orders
WITH gaps AS (
    SELECT customer_id, order_date,
        order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as gap
    FROM orders WHERE status = 'delivered'
)
SELECT customer_id, MAX(gap) as longest_gap
FROM gaps WHERE gap IS NOT NULL
GROUP BY customer_id ORDER BY longest_gap DESC;
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Portfolio + Practice Plan

### Update Portfolio with DQ Additions

Ensure each repo now has data quality checks documented:

```markdown
## Data Quality
This project includes automated data quality checks:
- **Extract validation:** API response status, JSON parsing
- **Transform validation:** null checks, range checks, accepted values
- **Load validation:** row count bounds, referential integrity
- **Monitoring:** freshness checks, distribution monitoring
```

### Remaining Weeks Practice Plan

| Week | Focus | SQL Practice | Other Practice |
|------|-------|-------------|---------------|
| 11 | Git + CI/CD + Mock Interviews | 10 problems/day | 1 mock interview |
| 12 | Resume + LinkedIn + Portfolio | 10 problems/day | Resume writing |
| 13 | SQL Live Coding Intensive | 50 problems/day | Speed drills |
| 14 | System Design + Behavioral | 10 problems/day | Mock interviews |
| 15 | Apply everywhere | 5 problems/day | Applications |

### 10-Week Progress Summary

```
Week 1:  ✅ SQL Foundation          (Day 1-7)
Week 2:  ✅ Python + Pandas + ETL    (Day 8-14)
Week 3:  ✅ Data Modeling + dbt      (Day 15-21)
Week 4:  ✅ Advanced SQL             (Day 22-28)
Week 5:  ✅ Airflow                  (Day 29-35)
Week 6:  ✅ AWS Core                 (Day 36-42)
Week 7:  ✅ Glue + Athena + PySpark  (Day 43-49)
Week 8:  ✅ Portfolio + Docker       (Day 50-56)
Week 9:  ✅ APIs + LLM + RAG         (Day 57-63)
Week 10: ✅ Data Quality + Interview Prep (Day 64-70)

Progress: 10/15 weeks complete (67%)
Portfolio projects: 7 repos
SQL problems solved: 60+
```

---

## 🌙 Evening: Rest + Plan

### 📝 Today's Checklist

- [ ] Self-assessed all Week 10 skills (rated 1-5)
- [ ] Identified 2-3 weak spots
- [ ] Practiced targeted exercises for weak spots
- [ ] Updated portfolio repos with DQ documentation
- [ ] Created practice plan for Weeks 11-15
- [ ] All Week 10 changes pushed to GitHub
- [ ] Ready for Week 11: Git Advanced + CI/CD + Mock Interviews

---

## 🎉 Week 10 Complete!

**Week 10 achievements:**
- 🛡️ Mastered data quality frameworks (Great Expectations + custom)
- 🧪 Built reusable testing patterns (schema, null, distribution, freshness)
- 💼 Solved 60 SQL interview problems
- 🎙️ Completed first mock interview
- 📊 Added DQ checks to all portfolio projects

**You're 2/3 through the program!** The foundation is solid. Weeks 11-15 focus on polishing, practicing, and applying.

---

*Week 10 complete! Interview prep ramps up from here. Let's go!* 🚀💪
