# 📅 Day 93 — Friday, 15 August 2026
# 📊 System Design: Schema Design for SG/MY Business Scenarios

---

## 🎯 Today's Goal

3 more schema design exercises for common SG/MY business scenarios. Practice thinking on your feet — interviewers often change requirements mid-discussion.

---

## ☀️ Morning Block (2 hours): Scenarios 1-2

### Scenario 1: "Design a data model for a food delivery loyalty program"

**Requirements:**
- Customers earn points per order (1 point per $1 spent)
- Bonus points for specific restaurants or promotions
- Points expire after 6 months
- Tiers: Bronze (0-999), Silver (1000-4999), Gold (5000+)
- Monthly loyalty report per customer

**Schema:**

```sql
-- Core tables
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR, email VARCHAR, phone VARCHAR,
    city VARCHAR, signup_date DATE,
    loyalty_tier VARCHAR DEFAULT 'Bronze',
    total_points INT DEFAULT 0
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    restaurant_id INT,
    order_date TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR
);

CREATE TABLE point_transactions (
    transaction_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    order_id INT REFERENCES orders(order_id),
    points_earned INT,
    points_redeemed INT,
    transaction_type VARCHAR, -- 'earn', 'redeem', 'expire', 'bonus'
    source VARCHAR, -- 'order', 'promotion', 'referral'
    earned_date DATE,
    expiry_date DATE,
    is_expired BOOLEAN DEFAULT FALSE
);

CREATE TABLE tier_rules (
    tier_name VARCHAR PRIMARY KEY,
    min_points INT,
    max_points INT,
    points_multiplier DECIMAL(3,2), -- e.g., 1.5x for Gold
    benefits TEXT
);

-- Key queries:
-- 1. Customer's current points (non-expired)
SELECT customer_id, SUM(points_earned) - SUM(points_redeemed) as available_points
FROM point_transactions
WHERE is_expired = FALSE
GROUP BY customer_id;

-- 2. Tier calculation
SELECT c.name, c.loyalty_tier,
       SUM(pt.points_earned) as total_earned
FROM customers c
JOIN point_transactions pt ON c.customer_id = pt.customer_id
GROUP BY c.customer_id, c.name, c.loyalty_tier;
```

### Scenario 2: "Design a data model for a hawker center management system"

**Requirements:**
- 100+ hawker stalls in multiple centers across Singapore
- Track: stall occupancy, rent payments, hygiene ratings, daily revenue (self-reported)
- Monthly report for NEA (National Environment Agency)
- Track which stalls are open/closed on each day

<details>
<summary>🔑 Schema design</summary>

```sql
CREATE TABLE hawker_centers (
    center_id INT PRIMARY KEY,
    name VARCHAR, -- "Maxwell Food Centre", "Lau Pa Sat"
    address VARCHAR,
    district VARCHAR,
    total_stalls INT,
    opening_year INT
);

CREATE TABLE stalls (
    stall_id INT PRIMARY KEY,
    center_id INT REFERENCES hawker_centers(center_id),
    stall_number VARCHAR, -- "01-23"
    cuisine_type VARCHAR, -- "Chinese", "Indian", "Malay", "Mixed"
    current_tenant_id INT,
    monthly_rent DECIMAL(10,2),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE tenants (
    tenant_id INT PRIMARY KEY,
    name VARCHAR,
    phone VARCHAR,
    lease_start DATE,
    lease_end DATE,
    is_current BOOLEAN DEFAULT TRUE
);

CREATE TABLE daily_operations (
    record_id INT PRIMARY KEY,
    stall_id INT REFERENCES stalls(stall_id),
    date DATE,
    is_open BOOLEAN,
    estimated_revenue DECIMAL(10,2),
    hygiene_score INT CHECK (hygiene_score BETWEEN 1 AND 5),
    notes TEXT
);

CREATE TABLE rent_payments (
    payment_id INT PRIMARY KEY,
    stall_id INT REFERENCES stalls(stall_id),
    month DATE,
    amount DECIMAL(10,2),
    paid_date DATE,
    status VARCHAR -- 'paid', 'overdue', 'waived'
);

-- NEA Monthly Report Query:
SELECT hc.name as center, s.stall_number, t.name as tenant,
       s.cuisine_type, AVG(do.hygiene_score) as avg_hygiene,
       SUM(do.estimated_revenue) as monthly_revenue,
       COUNT(*) as days_open
FROM daily_operations do
JOIN stalls s ON do.stall_id = s.stall_id
JOIN hawker_centers hc ON s.center_id = hc.center_id
JOIN tenants t ON s.current_tenant_id = t.tenant_id
WHERE do.date >= '2026-08-01' AND do.date < '2026-09-01'
GROUP BY hc.name, s.stall_number, t.name, s.cuisine_type
ORDER BY hc.name, avg_hygiene;
```
</details>

---

## 🌤️ Afternoon Block (2 hours): Scenario 3 + Rapid Fire

### Scenario 3: "Design a data model for a Singapore MRT ridership analytics system"

**Requirements:**
- 180+ MRT stations across 6 lines
- Track: tap-in/tap-out per station, peak hours, crowd levels
- Daily report for LTA (Land Transport Authority)
- Predict crowding for next hour

**Exercise:** Design the schema yourself before looking at the answer.

<details>
<summary>🔑 Schema</summary>

```sql
CREATE TABLE stations (
    station_id INT PRIMARY KEY,
    name VARCHAR, -- "Orchard", "Jurong East", "Dhoby Ghaut"
    line VARCHAR, -- "NS", "EW", "NE", "CC", "DT", "TE"
    latitude DECIMAL(9,6),
    longitude DECIMAL(9,6),
    is_interchange BOOLEAN,
    opening_year INT
);

CREATE TABLE ridership (
    record_id BIGINT PRIMARY KEY,
    station_id INT REFERENCES stations(station_id),
    timestamp TIMESTAMP,
    direction VARCHAR, -- 'entry', 'exit'
    card_type VARCHAR, -- 'ezlink', 'singapore_tourist_pass', 'bank_card'
    count INT -- number of taps in 15-min interval
);

CREATE TABLE crowd_levels (
    station_id INT REFERENCES stations(station_id),
    timestamp TIMESTAMP,
    level VARCHAR, -- 'low', 'moderate', 'high', 'very_high'
    estimated_people INT,
    PRIMARY KEY (station_id, timestamp)
);

-- Peak hour analysis:
SELECT s.name, s.line,
       EXTRACT(HOUR FROM r.timestamp) as hour,
       SUM(CASE WHEN r.direction = 'entry' THEN r.count ELSE 0 END) as entries,
       SUM(CASE WHEN r.direction = 'exit' THEN r.count ELSE 0 END) as exits
FROM ridership r
JOIN stations s ON r.station_id = s.station_id
WHERE r.timestamp >= CURRENT_DATE
GROUP BY s.name, s.line, hour
ORDER BY entries DESC;
```
</details>

### Rapid Fire Design Questions (5 min each)

1. "How would you model a hotel booking system for Booking.com?"
2. "Design a schema for a GrabPay transaction ledger"
3. "How would you track inventory for a supermarket chain (Sheng Siong)?"
4. "Design a student grade tracking system for NUS"
5. "Model a COVID vaccination tracking database"

Practice talking through these out loud — focus on identifying the main entities and relationships, not perfect SQL.

---

## 🌙 Evening (1 hour): Schema Design Patterns

### 🗺️ Quick Reference

```
COMMON PATTERNS:
  Star Schema: 1 fact table + N dimension tables (analytics)
  Snowflake: Normalized dimensions (save space, slower queries)
  SCD Type 1: Overwrite (no history)
  SCD Type 2: New row with dates (full history)
  SCD Type 3: Add column for previous value (limited history)

DESIGN PROCESS:
  1. Identify entities (what are the "things"?)
  2. Identify relationships (1:1, 1:N, M:N)
  3. Determine grain (what does one row represent?)
  4. Add slowly changing dimensions
  5. Consider partitioning (usually by date)

SG/MY CONTEXT:
  Use local examples: MRT stations, hawker centers, HDB blocks
  Mention local regulations: PDPA (data privacy), MAS (financial)
  Reference local companies: Grab, Shopee, DBS, Singtel
```

### 📝 Today's Checklist

- [ ] Designed loyalty program schema
- [ ] Designed hawker center schema
- [ ] Designed MRT ridership schema
- [ ] Practiced 5 rapid-fire designs
- [ ] Comfortable with schema design patterns

---

*Day 93 complete! Tomorrow: Behavioral prep.* 🎤
