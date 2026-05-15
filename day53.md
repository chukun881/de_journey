# 📅 Day 53 — Sunday, 6 July 2026
# 🐳 Containerize Your Pipeline: Postgres + Airflow with Docker Compose

---

## 🎯 Today's Goal

Yesterday you learned Docker basics — images, containers, Dockerfiles. Today you use **docker-compose** to spin up your entire MakanExpress data platform with one command: PostgreSQL, pgAdmin, and Airflow. This is how real data teams work.

**Philosophy:** In production, your pipeline has multiple services (database, orchestrator, ETL scripts). docker-compose lets you define them all in one file, start them together, and network them automatically. This is your local "mini production environment."

---

## ☀️ Morning Block (2 hours): docker-compose with PostgreSQL

### What is docker-compose?

docker-compose is a tool for defining and running multi-container applications. You write a `docker-compose.yml` file that describes:
- What containers to run (services)
- How they connect (networks)
- What data persists (volumes)
- Environment variables and configs

```
Without docker-compose:
$ docker network create mynet
$ docker run -d --name postgres --network mynet -e POSTGRES_PASSWORD=... -v pgdata:/var/lib/postgresql/data postgres:15
$ docker run -d --name pgadmin --network mynet -p 5050:80 ...
$ docker run -d --name airflow --network mynet -p 8080:8080 ...
# Forgot a flag? Start over.

With docker-compose:
$ docker-compose up -d
# Done. Everything starts, connects, and persists.
```

### Project Setup

```bash
mkdir -p ~/projects/makanexpress-docker
cd ~/projects/makanexpress-docker
mkdir -p postgres/init dags logs output
```

### Write docker-compose.yml (Part 1: PostgreSQL + pgAdmin)

Create `docker-compose.yml`:

```yaml
version: "3.8"

services:
  # ─── PostgreSQL Database ───
  postgres:
    image: postgres:15
    container_name: makanexpress-db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: makanexpress2026
      POSTGRES_DB: makanexpress
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d makanexpress"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - makanexpress-net

  # ─── pgAdmin (Database GUI) ───
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: makanexpress-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@makanexpress.sg
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "5050:80"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - makanexpress-net

# ─── Persistent Volumes ───
volumes:
  pgdata:
    driver: local

# ─── Network ───
networks:
  makanexpress-net:
    driver: bridge
```

### Understanding Each Section

**Services:** Each service = one container.

```yaml
postgres:              # Service name (used as hostname between containers)
  image: postgres:15   # Use official PostgreSQL 15 image
  container_name: makanexpress-db  # Friendly name for docker ps
```

**Environment variables:** Pass config without hardcoding in code.

```yaml
environment:
  POSTGRES_USER: admin          # DB username
  POSTGRES_PASSWORD: makanexpress2026  # DB password
  POSTGRES_DB: makanexpress     # Database created on first start
```

**Ports:** Map host port → container port.

```yaml
ports:
  - "5432:5432"  # host:container
  # From your laptop: localhost:5432 → container's 5432
```

**Volumes:** Persist data beyond container lifetime.

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data   # Named volume (managed by Docker)
  - ./postgres/init:/docker-entrypoint-initdb.d  # Bind mount (local dir)
```

Two types of volumes:
- **Named volume** (`pgdata:`): Docker manages it. Data survives `docker-compose down`.
- **Bind mount** (`./postgres/init:`): Maps a local directory into the container. Changes on host = changes in container instantly.

**Healthcheck:** Tells Docker when the service is ready.

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U admin -d makanexpress"]
  interval: 10s   # Check every 10 seconds
  timeout: 5s     # Timeout after 5 seconds
  retries: 5      # Try 5 times before marking unhealthy
```

**depends_on + condition:** Start pgAdmin only after PostgreSQL is healthy.

```yaml
depends_on:
  postgres:
    condition: service_healthy  # Wait for healthcheck to pass
```

**Networks:** All services on the same network can communicate using service names as hostnames.

```yaml
networks:
  - makanexpress-net
```

Inside containers, `postgres` resolves to the PostgreSQL container's IP. No need to know actual IPs.

### Init Scripts

PostgreSQL's Docker image runs any `.sql` or `.sh` files in `/docker-entrypoint-initdb.d/` on first start.

Create `postgres/init/01_schema.sql`:

```sql
-- MakanExpress Schema — Auto-loaded on first Docker start

-- Restaurants dimension
CREATE TABLE IF NOT EXISTS dim_restaurants (
    restaurant_id VARCHAR(10) PRIMARY KEY,
    restaurant_name VARCHAR(200) NOT NULL,
    cuisine_type VARCHAR(50),
    area VARCHAR(50),
    rating DECIMAL(3,2) DEFAULT 0.00,
    is_halal BOOLEAN DEFAULT FALSE,
    avg_delivery_min INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Customers dimension
CREATE TABLE IF NOT EXISTS dim_customers (
    customer_id VARCHAR(10) PRIMARY KEY,
    customer_name VARCHAR(200) NOT NULL,
    email VARCHAR(200),
    phone VARCHAR(20),
    signup_date DATE,
    preferred_area VARCHAR(50),
    is_active BOOLEAN DEFAULT TRUE,
    scd_valid_from TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    scd_valid_to TIMESTAMP DEFAULT '9999-12-31 23:59:59',
    scd_is_current BOOLEAN DEFAULT TRUE
);

-- Orders fact table
CREATE TABLE IF NOT EXISTS fact_orders (
    order_id BIGINT PRIMARY KEY,
    customer_id VARCHAR(10) REFERENCES dim_customers(customer_id),
    restaurant_id VARCHAR(10) REFERENCES dim_restaurants(restaurant_id),
    order_datetime TIMESTAMP NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    delivery_time_min INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO dim_restaurants (restaurant_id, restaurant_name, cuisine_type, area, rating, is_halal, avg_delivery_min)
VALUES
    ('R001', 'Ah Hock Nasi Lemak', 'Malay', 'Orchard', 4.50, TRUE, 25),
    ('R002', 'Tian Tian Chicken Rice', 'Chinese', 'Maxwell', 4.80, FALSE, 20),
    ('R003', 'Prata House', 'Indian', 'Jurong', 4.20, TRUE, 30),
    ('R004', 'Song Fa Bak Kut Teh', 'Chinese', 'Clarke Quay', 4.60, FALSE, 22),
    ('R005', 'Roti John Express', 'Malay', 'Tampines', 4.00, TRUE, 28)
ON CONFLICT DO NOTHING;

INSERT INTO dim_customers (customer_id, customer_name, email, signup_date, preferred_area)
VALUES
    ('C001', 'Tan Wei Ming', 'tanwm@email.com', '2025-01-15', 'Orchard'),
    ('C002', 'Lim Siew Ling', 'limsl@email.com', '2025-03-20', 'Maxwell'),
    ('C003', 'Ahmad Bin Ismail', 'ahmad@email.com', '2025-06-01', 'Jurong')
ON CONFLICT DO NOTHING;

INSERT INTO fact_orders (order_id, customer_id, restaurant_id, order_datetime, total_amount, status, delivery_time_min)
VALUES
    (1001, 'C001', 'R001', '2026-01-15 12:30:00', 25.50, 'completed', 22),
    (1002, 'C002', 'R002', '2026-01-15 13:00:00', 18.00, 'completed', 18),
    (1003, 'C001', 'R003', '2026-01-16 19:30:00', 32.00, 'completed', 35),
    (1004, 'C003', 'R002', '2026-01-16 20:00:00', 15.50, 'completed', 20),
    (1005, 'C002', 'R001', '2026-01-17 12:15:00', 45.00, 'completed', 25),
    (1006, 'C001', 'R004', '2026-01-17 18:45:00', 28.00, 'completed', 19),
    (1007, 'C003', 'R005', '2026-01-18 11:00:00', 12.00, 'completed', 30),
    (1008, 'C002', 'R003', '2026-01-18 14:30:00', 22.50, 'completed', 28)
ON CONFLICT DO NOTHING;

-- Useful views
CREATE OR REPLACE VIEW v_restaurant_summary AS
SELECT
    r.restaurant_name,
    r.cuisine_type,
    r.area,
    r.rating,
    COUNT(o.order_id) AS total_orders,
    COALESCE(SUM(o.total_amount), 0) AS total_revenue,
    COALESCE(AVG(o.total_amount), 0)::DECIMAL(10,2) AS avg_order_value
FROM dim_restaurants r
LEFT JOIN fact_orders o ON r.restaurant_id = o.restaurant_id
GROUP BY r.restaurant_id, r.restaurant_name, r.cuisine_type, r.area, r.rating
ORDER BY total_revenue DESC;
```

### Start PostgreSQL + pgAdmin

```bash
# Start everything
docker-compose up -d

# Watch logs
docker-compose logs -f postgres

# Check status
docker-compose ps

# Test the database connection
docker exec -it makanexpress-db psql -U admin -d makanexpress -c "SELECT * FROM v_restaurant_summary;"
```

**Exercise:** Open pgAdmin in your browser at `http://localhost:5050`. Login with `admin@makanexpress.sg` / `admin123`. Add a server connection:
- Host: `postgres` (the service name!)
- Port: 5432
- Username: admin
- Password: makanexpress2026

Run `SELECT * FROM v_restaurant_summary;` in the query tool. What do you see?

<details>
<summary>🔑 Answer</summary>

You should see a table like:

| restaurant_name | cuisine_type | area | rating | total_orders | total_revenue | avg_order_value |
|----------------|-------------|------|--------|-------------|---------------|----------------|
| Ah Hock Nasi Lemak | Malay | Orchard | 4.50 | 2 | 70.50 | 35.25 |
| Prata House | Indian | Jurong | 4.20 | 2 | 54.50 | 27.25 |
| Tian Tian Chicken Rice | Chinese | Maxwell | 4.80 | 2 | 33.50 | 16.75 |
| Song Fa Bak Kut Teh | Chinese | Clarke Quay | 4.60 | 1 | 28.00 | 28.00 |
| Roti John Express | Malay | Tampines | 4.00 | 1 | 12.00 | 12.00 |

**Key insight:** Using `postgres` as the hostname works because both containers are on the same Docker network (`makanexpress-net`). Docker's internal DNS resolves service names to container IPs. This is MUCH better than hardcoding IP addresses.

</details>

---

## 🌤️ Afternoon Block (2 hours): Add Airflow to docker-compose

### The Airflow Docker Setup

Airflow has several components:
- **Webserver:** UI (port 8080)
- **Scheduler:** Runs tasks on schedule
- **Metadata database:** Stores DAG state (we'll reuse PostgreSQL or add a separate one)

For local development, we use the official Airflow Docker image with `standalone` mode.

### Updated docker-compose.yml

Replace your `docker-compose.yml` with this full version:

```yaml
version: "3.8"

x-airflow-common: &airflow-common
  image: apache/airflow:2.8.1
  environment:
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://admin:makanexpress2026@postgres:5432/airflow
    AIRFLOW__CORE__LOAD_EXAMPLES: "false"
    AIRFLOW__CORE__FERNET_KEY: "46BKJoQYlPPOexq0OhDZnIlNepKFf87WFwLbfzqDDho="
    AIRFLOW__WEBSERVER__EXPOSE_CONFIG: "true"
    AIRFLOW__WEBSERVER__SECRET_KEY: "makanexpress-secret-2026"
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ./output:/opt/airflow/output
  depends_on:
    postgres:
      condition: service_healthy
  networks:
    - makanexpress-net

services:
  # ─── PostgreSQL (shared by app + Airflow) ───
  postgres:
    image: postgres:15
    container_name: makanexpress-db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: makanexpress2026
      POSTGRES_DB: makanexpress
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d makanexpress"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - makanexpress-net

  # ─── pgAdmin ───
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: makanexpress-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@makanexpress.sg
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "5050:80"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - makanexpress-net

  # ─── Airflow Init (runs once to set up DB) ───
  airflow-init:
    <<: *airflow-common
    container_name: makanexpress-airflow-init
    entrypoint: /bin/bash
    command:
      - -c
      - |
        airflow db init
        airflow users create \
          --username admin \
          --firstname Admin \
          --lastname User \
          --role Admin \
          --email admin@makanexpress.sg \
          --password admin123 || true
    restart: "no"

  # ─── Airflow Webserver ───
  airflow-webserver:
    <<: *airflow-common
    container_name: makanexpress-airflow-web
    command: airflow webserver --port 8080
    ports:
      - "8080:8080"
    restart: always
    depends_on:
      airflow-init:
        condition: service_completed_successfully

  # ─── Airflow Scheduler ───
  airflow-scheduler:
    <<: *airflow-common
    container_name: makanexpress-airflow-scheduler
    command: airflow scheduler
    restart: always
    depends_on:
      airflow-init:
        condition: service_completed_successfully

volumes:
  pgdata:
    driver: local

networks:
  makanexpress-net:
    driver: bridge
```

### Understanding the YAML Anchors

The `x-airflow-common: &airflow-common` section is a YAML anchor — a reusable template:

```yaml
x-airflow-common: &airflow-common     # Define anchor
  image: apache/airflow:2.8.1
  environment: { ... }
  volumes: [ ... ]

services:
  airflow-webserver:
    <<: *airflow-common                # Merge anchor here
    command: airflow webserver          # Add service-specific config
    ports: ["8080:8080"]

  airflow-scheduler:
    <<: *airflow-common                # Same base config
    command: airflow scheduler          # Different command
```

This avoids repeating the same config for webserver and scheduler.

### Create the Airflow Database

Add a second init script `postgres/init/02_airflow_db.sql`:

```sql
-- Create Airflow database (separate from MakanExpress)
CREATE DATABASE airflow;
```

### Create a Test DAG

Create `dags/makanexpress_daily_summary.py`:

```python
"""
MakanExpress Daily Summary DAG — Docker Edition
Runs inside the Airflow container, connects to PostgreSQL container.
"""
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from datetime import datetime, timedelta


default_args = {
    'owner': 'makanexpress',
    'depends_on_past': False,
    'email_on_failure': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}


def generate_daily_report(**context):
    """Generate a daily summary and save results."""
    import psycopg2
    import json
    
    execution_date = context['ds']
    
    # Connect to PostgreSQL using Docker network hostname
    conn = psycopg2.connect(
        host='postgres',          # Docker service name!
        port=5432,
        database='makanexpress',
        user='admin',
        password='makanexpress2026'
    )
    
    cursor = conn.cursor()
    
    # Get restaurant summary
    cursor.execute("""
        SELECT 
            restaurant_name,
            cuisine_type,
            total_orders,
            total_revenue,
            avg_order_value
        FROM v_restaurant_summary
        ORDER BY total_revenue DESC
    """)
    
    results = cursor.fetchall()
    
    print(f"\n📊 MakanExpress Daily Summary for {execution_date}")
    print("=" * 60)
    print(f"{'Restaurant':<25} {'Cuisine':<10} {'Orders':>7} {'Revenue':>10}")
    print("-" * 60)
    for row in results:
        print(f"{row[0]:<25} {row[1]:<10} {row[2]:>7} ${row[3]:>9.2f}")
    print("=" * 60)
    print(f"Total restaurants: {len(results)}")
    
    cursor.close()
    conn.close()
    
    return f"Report generated for {execution_date} with {len(results)} restaurants"


with DAG(
    dag_id='makanexpress_daily_summary',
    default_args=default_args,
    description='Generate daily restaurant summary for MakanExpress',
    schedule_interval='@daily',
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['makanexpress', 'docker'],
) as dag:

    generate_report = PythonOperator(
        task_id='generate_daily_report',
        python_callable=generate_daily_report,
    )
    
    log_completion = SQLExecuteQueryOperator(
        task_id='log_completion',
        conn_id='makanexpress_postgres',
        sql="""
            CREATE TABLE IF NOT EXISTS dag_run_log (
                run_id SERIAL PRIMARY KEY,
                dag_id VARCHAR(100),
                execution_date DATE,
                status VARCHAR(20),
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );
            INSERT INTO dag_run_log (dag_id, execution_date, status)
            VALUES ('makanexpress_daily_summary', '{{ ds }}', 'completed');
        """,
    )
    
    generate_report >> log_completion
```

### Start Everything

```bash
# Stop previous containers first
docker-compose down

# Start fresh (this builds and starts all services)
docker-compose up -d

# Watch Airflow init (wait for it to complete)
docker-compose logs -f airflow-init

# Check all services are running
docker-compose ps
```

**Expected output of `docker-compose ps`:**
```
NAME                           STATUS          PORTS
makanexpress-db                Up (healthy)    0.0.0.0:5432->5432/tcp
makanexpress-pgadmin           Up              0.0.0:80->80/tcp
makanexpress-airflow-web       Up              0.0.0:8080->8080/tcp
makanexpress-airflow-scheduler Up
```

**Exercise:** Open Airflow UI at `http://localhost:8080`. Login with `admin` / `admin123`. Find the `makanexpress_daily_summary` DAG. Enable it and trigger a manual run. Check the logs.

<details>
<summary>🔑 Answer</summary>

Steps:
1. Open `http://localhost:8080` in your browser
2. Login: username = `admin`, password = `admin123`
3. You should see the `makanexpress_daily_summary` DAG
4. Toggle the DAG on (top-left switch)
5. Click the "Trigger DAG" button (play icon)
6. Click into the DAG run → click on `generate_daily_report` task → check Logs
7. You should see the restaurant summary printed in the logs

If the `log_completion` task fails with "Connection not found", that's expected — you need to create the Airflow connection. Go to Admin → Connections → Add:
- Conn Id: `makanexpress_postgres`
- Conn Type: `PostgreSQL`
- Host: `postgres`
- Schema: `makanexpress`
- Login: `admin`
- Password: `makanexpress2026`
- Port: 5432

</details>

### Add Airflow Connection via CLI

Alternatively, add the connection via command line:

```bash
docker exec -it makanexpress-airflow-web airflow connections add makanexpress_postgres \
    --conn-type postgres \
    --conn-host postgres \
    --conn-schema makanexpress \
    --conn-login admin \
    --conn-password makanexpress2026 \
    --conn-port 5432
```

### Directory Structure

Your project should look like this:

```
makanexpress-docker/
├── docker-compose.yml
├── postgres/
│   └── init/
│       ├── 01_schema.sql
│       └── 02_airflow_db.sql
├── dags/
│   └── makanexpress_daily_summary.py
├── logs/
│   └── (Airflow logs go here)
├── output/
│   └── (ETL output files go here)
├── .env
└── .gitignore
```

---

## 🌙 Evening (1 hour): Best Practices + Troubleshooting

### Docker Best Practices for Data Engineers

1. **Use `.env` files for secrets:**

Create `.env`:
```
POSTGRES_USER=admin
POSTGRES_PASSWORD=makanexpress2026
POSTGRES_DB=makanexpress
AIRFLOW_PASSWORD=admin123
```

Update `docker-compose.yml`:
```yaml
postgres:
  image: postgres:15
  environment:
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    POSTGRES_DB: ${POSTGRES_DB}
```

**IMPORTANT:** Add `.env` to `.gitignore`! Never commit secrets.

2. **Use health checks for startup order:**
```yaml
depends_on:
  postgres:
    condition: service_healthy  # Not just "started"
```

3. **Use named volumes for persistent data:**
```yaml
volumes:
  pgdata:          # Survives docker-compose down
    driver: local
```

4. **Use bind mounts for code during development:**
```yaml
volumes:
  - ./dags:/opt/airflow/dags  # Edit locally, changes reflected immediately
```

5. **Pin image versions:**
```yaml
image: postgres:15          # ✅ Specific major version
image: postgres:15.5        # ✅ Exact version
image: postgres:latest      # ❌ Could break unexpectedly
```

6. **One process per container:**
```
✅ postgres container → runs PostgreSQL
✅ airflow-webserver → runs webserver
✅ airflow-scheduler → runs scheduler
❌ don't put everything in one container
```

### Common Issues & Fixes

**Problem 1: Port already in use**
```
Error: Bind for 0.0.0.0:5432 failed: port is already allocated
```

Fix:
```bash
# Find what's using the port
lsof -i :5432
# or
netstat -tulpn | grep 5432

# Stop the conflicting container
docker stop <container_name>

# Or change the port in docker-compose.yml
ports:
  - "5433:5432"  # Use 5433 on host instead
```

**Problem 2: Permission denied on volume mount**
```
Error: mkdir: cannot create directory '/var/lib/postgresql/data': Permission denied
```

Fix:
```bash
# For PostgreSQL, the PGDATA directory needs specific permissions
# Use a named volume instead of bind mount for data
volumes:
  - pgdata:/var/lib/postgresql/data  # Named volume (Docker manages permissions)
```

**Problem 3: Container keeps restarting**
```bash
# Check logs
docker-compose logs <service_name>

# Common causes:
# - Missing environment variables
# - Database not ready yet (add health check)
# - Wrong image version
# - Syntax error in docker-compose.yml
```

**Problem 4: Can't connect between containers**
```
psycopg2.OperationalError: could not connect to server: Connection refused
```

Fix:
```python
# Use the SERVICE NAME as hostname, not localhost!
conn = psycopg2.connect(
    host='postgres',  # ✅ Service name from docker-compose.yml
    # host='localhost',  # ❌ This is the container's own localhost
)
```

**Problem 5: Init scripts not running**
```
# PostgreSQL init scripts only run on FIRST start (when data volume is empty)
# If you change the SQL scripts, you need to reset:
docker-compose down -v  # -v removes volumes too
docker-compose up -d    # Rebuilds fresh
```

### Cleanup Commands

```bash
# Stop all services
docker-compose down

# Stop and remove volumes (fresh start)
docker-compose down -v

# Remove all stopped containers
docker container prune

# Remove all unused images
docker image prune -a

# Nuclear option: remove everything Docker
docker system prune -a --volumes
```

### Exercise: Verify Your Setup

<details>
<summary>🔑 Answer</summary>

Run these commands to verify everything works:

```bash
# 1. All services running
docker-compose ps
# Should show: postgres (healthy), pgadmin, airflow-webserver, airflow-scheduler

# 2. Database has data
docker exec -it makanexpress-db psql -U admin -d makanexpress -c "SELECT COUNT(*) FROM fact_orders;"
# Should return: 8

# 3. Airflow UI accessible
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080
# Should return: 200

# 4. pgAdmin accessible
curl -s -o /dev/null -w "%{http_code}" http://localhost:5050
# Should return: 200

# 5. DAG loaded
docker exec -it makanexpress-airflow-web airflow dags list
# Should include: makanexpress_daily_summary
```

</details>

### Quick Reference Card

```markdown
# 🐳 docker-compose Quick Reference for Data Engineers

## Core Commands
docker-compose up -d          # Start all services (detached)
docker-compose down           # Stop all services
docker-compose down -v        # Stop + remove volumes
docker-compose ps             # List running services
docker-compose logs -f [svc]  # Follow logs (optional: filter by service)
docker-compose build          # Rebuild images
docker-compose restart [svc]  # Restart a service
docker-compose exec [svc] bash # Shell into running service

## docker-compose.yml Anatomy
version: "3.8"
services:           # Each service = one container
  postgres:
    image: postgres:15
    environment: {} # Config via env vars
    ports: []       # host:container port mapping
    volumes: []     # Persist data / mount code
    healthcheck: {} # Ready-check
    networks: []    # Connect services
    depends_on: {}  # Startup order
    restart: always # Auto-restart on crash

volumes: {}         # Named volumes (persistent)
networks: {}        # Custom networks (service discovery)

## Key Patterns
- Service name = hostname (postgres, airflow-webserver)
- ${VAR} reads from .env file
- ./local:/container = bind mount (dev)
- named_volume:/container = named volume (data)
- YAML anchors (&name / <<: *name) for shared config

## Troubleshooting
- Port conflict → change host port or stop conflicting service
- Connection refused → use service name, not localhost
- Init scripts not running → docker-compose down -v and restart
- Permission denied → use named volumes for DB data
- Container restarting → check logs with docker-compose logs

## Cost: $0
All of this runs locally. No cloud charges.
```

---

### 📝 Today's Checklist

- [ ] docker-compose.yml with PostgreSQL + pgAdmin created and running
- [ ] Init scripts load schema + sample data automatically
- [ ] pgAdmin accessible and connected to database
- [ ] Airflow added to docker-compose (webserver + scheduler)
- [ ] Airflow UI accessible at localhost:8080
- [ ] Test DAG loads and runs successfully
- [ ] Understand Docker networking (service names as hostnames)
- [ ] Understand volumes (named vs bind mounts)
- [ ] Read troubleshooting guide
- [ ] Can tear down and rebuild from scratch

---

*Day 53 complete! You now have a containerized data platform. This is how real data teams develop.* 🐳🏗️
