# 📅 Day 52 — Saturday, 5 July 2026
# 🐳 Docker Basics: Images, Containers, Dockerfile

---

## 🎯 Today's Goal

Docker is how modern data teams deploy and share pipelines. If your Python ETL script works on your laptop but crashes on your colleague's machine, Docker solves that. Today you learn what Docker is, why data engineers need it, and how to containerize a simple Python ETL script.

**Philosophy:** "But it works on my machine" is the oldest problem in software engineering. Docker makes "your machine" identical to "production machine." For data engineers, Docker means: reproducible pipelines, consistent environments, and easy deployment.

---

## ☀️ Morning Block (2 hours): Docker Concepts + Installation

### What is Docker?

**The food court analogy (Singapore style):**

Think of Docker like a hawker center. Every stall (container) has:
- Its own equipment (runtime, libraries)
- Its own ingredients (code, dependencies)
- Its own space (isolated filesystem)
- But they share the building (host OS kernel)

Without Docker (traditional deployment):
```
Your laptop:  Python 3.11 + Pandas 2.1 + psycopg2 = ✅ Works
Server:       Python 3.9 + Pandas 1.5 + psycopg2 = ❌ Breaks
Colleague:    Python 3.12 + Pandas 2.2 + missing psycopg2 = ❌ Different error
```

With Docker:
```
Your laptop:  Docker container with Python 3.11 + exact deps = ✅
Server:       SAME Docker container = ✅
Colleague:    SAME Docker container = ✅
```

### Containers vs Virtual Machines

```
┌─────────────────────────────────┐  ┌─────────────────────────────────┐
│       VIRTUAL MACHINE           │  │         CONTAINER               │
│                                 │  │                                 │
│  ┌─────┐ ┌─────┐ ┌─────┐      │  │  ┌─────┐ ┌─────┐ ┌─────┐      │
│  │App A│ │App B│ │App C│      │  │  │App A│ │App B│ │App C│      │
│  └──┬──┘ └──┬──┘ └──┬──┘      │  │  └──┬──┘ └──┬──┘ └──┬──┘      │
│     │       │       │          │  │     │       │       │          │
│  ┌──┴──┐ ┌──┴──┐ ┌──┴──┐      │  │  ┌──┴──┐ ┌──┴──┐ ┌──┴──┐      │
│  │Bins │ │Bins │ │Bins │      │  │  │Bins │ │Bins │ │Bins │      │
│  │Libs │ │Libs │ │Libs │      │  │  │Libs │ │Libs │ │Libs │      │
│  └──┬──┘ └──┬──┘ └──┬──┘      │  │  └──┬──┘ └──┬──┘ └──┬──┘      │
│  ┌──┴────────┴────────┴──┐     │  │  ┌──┴────────┴────────┴──┐     │
│  │    Guest OS (Ubuntu)   │     │  │  │     Docker Engine      │     │
│  └───────────┬────────────┘     │  │  └───────────┬────────────┘     │
│  ┌───────────┴────────────┐     │  │  ┌───────────┴────────────┐     │
│  │    Hypervisor (VMware)  │     │  │  │     Host OS (Linux)     │     │
│  └───────────┬────────────┘     │  │  └───────────┬────────────┘     │
│  ┌───────────┴────────────┐     │  │  ┌───────────┴────────────┐     │
│  │       Host OS           │     │  │  │     Infrastructure      │     │
│  └─────────────────────────┘     │  │  └─────────────────────────┘     │
└─────────────────────────────────┘  └─────────────────────────────────┘

VM: Heavy (GBs), slow boot (minutes), full OS per app
Container: Light (MBs), instant boot (seconds), shared kernel
```

**Why data engineers use containers, not VMs:**
- Airflow workers run in containers
- dbt runs in containers
- Glue jobs are containerized under the hood
- AWS ECS/EKS run containers
- Faster CI/CD

### Install Docker

**On your laptop (pick one):**

**Windows:**
```bash
# Download Docker Desktop for Windows
# https://www.docker.com/products/docker-desktop/
# Install, restart, verify:
docker --version
docker run hello-world
```

**macOS:**
```bash
# Download Docker Desktop for Mac
# Or use Homebrew:
brew install --cask docker
# Open Docker app, wait for it to start, then:
docker --version
docker run hello-world
```

**Linux (Ubuntu/Debian):**
```bash
# Update packages
sudo apt-get update

# Install Docker
sudo apt-get install -y docker.io

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to docker group (avoid sudo)
sudo usermod -aG docker $USER

# Log out and log back in, then:
docker --version
docker run hello-world
```

**Exercise:** Run `docker run hello-world`. What does the output tell you about how Docker works?

<details>
<summary>🔑 Answer</summary>

The output shows:
1. Docker client sends request to Docker daemon
2. Daemon pulls the `hello-world` image from Docker Hub (registry)
3. Daemon creates a new container from that image
4. Container runs, prints the message, exits

This proves: client → daemon → registry → container pipeline works correctly.

The message literally says: "This message shows that your installation appears to be working correctly."

</details>

### Docker Architecture

```
┌──────────────────────────────────────────────┐
│                 DOCKER ARCHITECTURE           │
│                                              │
│  ┌──────────────┐     ┌──────────────────┐   │
│  │ Docker Client │────▶│  Docker Daemon   │   │
│  │ (docker CLI)  │     │   (dockerd)      │   │
│  └──────────────┘     └────────┬─────────┘   │
│                                │              │
│                    ┌───────────┴──────────┐   │
│                    │                      │   │
│                    ▼                      ▼   │
│             ┌──────────┐         ┌──────────┐ │
│             │  Images   │         │Containers│ │
│             │ (templates│         │ (running │ │
│             │  for      │         │ instances│ │
│             │ containers│         │ of images│ │
│             └────┬─────┘         └──────────┘ │
│                  │                              │
│                  ▼                              │
│          ┌──────────────┐                       │
│          │  Docker Hub   │  ← Registry of       │
│          │  (Registry)   │    pre-built images   │
│          └──────────────┘                       │
└──────────────────────────────────────────────┘
```

**Key terms:**
- **Image:** A read-only template with instructions for creating a container (like a class in OOP)
- **Container:** A running instance of an image (like an object in OOP)
- **Dockerfile:** A text file with instructions to build an image (like a recipe)
- **Registry:** Where images are stored and shared (Docker Hub = public, ECR = AWS private)
- **Daemon:** Background service that manages containers
- **Client:** CLI tool you interact with (`docker` command)

### First Container: Alpine Linux

```bash
# Pull and run a minimal Linux container
docker run -it alpine sh

# You're now inside the container! Try:
cat /etc/os-release
ls /
echo "Hello from inside Docker!"
whoami

# Exit the container
exit
```

**Exercise:** What happens if you run `docker run -it alpine sh` again and check for files you created in the previous run? Why?

<details>
<summary>🔑 Answer</summary>

Any files you created inside the first container are **gone**. Each `docker run` creates a **new** container from the image. Containers are ephemeral — when they stop, their filesystem disappears unless you use volumes.

This is by design! Containers should be stateless. If you need persistent data, you use **volumes** or **bind mounts** (we'll cover this afternoon).

</details>

### Useful Docker Commands (First Set)

```bash
# List running containers
docker ps

# List ALL containers (including stopped)
docker ps -a

# List images you have
docker images

# Remove a stopped container
docker rm <container_id>

# Remove an image
docker rmi <image_id>

# Pull an image without running
docker pull python:3.11-slim

# Run a container in the background
docker run -d --name my-python python:3.11-slim sleep infinity

# Stop a running container
docker stop my-python

# Remove it
docker rm my-python

# See container logs
docker logs <container_id>

# Execute a command inside a running container
docker exec -it <container_id> bash
```

**Exercise:** Pull `python:3.11-slim`, run it, and execute Python inside to print "Hello from containerized Python!"

<details>
<summary>🔑 Answer</summary>

```bash
# Method 1: Run Python directly
docker run python:3.11-slim python -c "print('Hello from containerized Python!')"

# Method 2: Interactive
docker run -it python:3.11-slim python
>>> print("Hello from containerized Python!")
>>> exit()

# Method 3: Run in background and exec
docker run -d --name test-python python:3.11-slim sleep 300
docker exec -it test-python python -c "print('Hello from containerized Python!')"
docker stop test-python
docker rm test-python
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Dockerfile + Build + Run

### What is a Dockerfile?

A Dockerfile is a text file with step-by-step instructions to build a Docker image. Think of it as a recipe.

```dockerfile
# Dockerfile for MakanExpress ETL Script
# Each instruction creates a new "layer" in the image

# 1. Base image — start from something
FROM python:3.11-slim

# 2. Set working directory inside the container
WORKDIR /app

# 3. Copy requirements first (for caching — more on this below)
COPY requirements.txt .

# 4. Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# 5. Copy your code
COPY . .

# 6. Set environment variables
ENV PYTHONUNBUFFERED=1

# 7. Run the script
CMD ["python", "etl.py"]
```

### Dockerfile Instructions Explained

| Instruction | What it does | Example |
|------------|-------------|---------|
| `FROM` | Base image to start from | `FROM python:3.11-slim` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy files from host to container | `COPY . .` |
| `RUN` | Execute command during build | `RUN pip install -r requirements.txt` |
| `CMD` | Default command when container starts | `CMD ["python", "main.py"]` |
| `ENTRYPOINT` | Command that always runs (can't be overridden easily) | `ENTRYPOINT ["python"]` |
| `ENV` | Set environment variable | `ENV DB_HOST=localhost` |
| `EXPOSE` | Document which port the container listens on | `EXPOSE 8080` |
| `ARG` | Build-time variable | `ARG VERSION=1.0` |
| `VOLUME` | Declare a mount point for persistent data | `VOLUME /data` |

### Build Your First Image

Create a project directory:

```bash
mkdir -p ~/projects/docker-etl-demo
cd ~/projects/docker-etl-demo
```

Create `requirements.txt`:
```
pandas==2.1.4
psycopg2-binary==2.9.9
boto3==1.34.0
```

Create `etl.py`:
```python
"""
MakanExpress Daily ETL — Containerized Version
Extracts order data, transforms, outputs summary.
"""
import pandas as pd
from datetime import datetime
import logging
import os

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


def extract() -> pd.DataFrame:
    """Generate sample order data (in real life, this reads from S3/API)."""
    logger.info("Extracting order data...")
    
    data = [
        {"order_id": 1, "restaurant": "Ah Hock Nasi Lemak", "amount": 12.50, "area": "Orchard"},
        {"order_id": 2, "restaurant": "Tian Tian Chicken Rice", "amount": 8.00, "area": "Maxwell"},
        {"order_id": 3, "restaurant": "Prata House", "amount": 6.50, "area": "Jurong"},
        {"order_id": 4, "restaurant": "Ah Hock Nasi Lemak", "amount": 15.00, "area": "Orchard"},
        {"order_id": 5, "restaurant": "Tian Tian Chicken Rice", "amount": 9.50, "area": "Maxwell"},
        {"order_id": 6, "restaurant": "Prata House", "amount": 11.00, "area": "Jurong"},
        {"order_id": 7, "restaurant": "Ah Hock Nasi Lemak", "amount": 22.00, "area": "Tampines"},
        {"order_id": 8, "restaurant": "Tian Tian Chicken Rice", "amount": 7.50, "area": "Maxwell"},
    ]
    
    df = pd.DataFrame(data)
    logger.info(f"Extracted {len(df)} orders")
    return df


def transform(df: pd.DataFrame) -> pd.DataFrame:
    """Aggregate orders by restaurant."""
    logger.info("Transforming data...")
    
    result = df.groupby('restaurant').agg(
        total_revenue=('amount', 'sum'),
        order_count=('order_id', 'count'),
        avg_order_value=('amount', 'mean'),
        areas_served=('area', 'nunique')
    ).reset_index()
    
    result['total_revenue'] = result['total_revenue'].round(2)
    result['avg_order_value'] = result['avg_order_value'].round(2)
    
    logger.info(f"Transformed to {len(result)} restaurant summaries")
    return result


def load(df: pd.DataFrame) -> None:
    """Save results (in real life, load to RDS/S3)."""
    output_path = os.environ.get('OUTPUT_PATH', '/app/output/summary.csv')
    
    # Ensure output directory exists
    os.makedirs(os.path.dirname(output_path), exist_ok=True)
    
    df.to_csv(output_path, index=False)
    logger.info(f"Results saved to {output_path}")


def main():
    """Run the ETL pipeline."""
    logger.info("=" * 50)
    logger.info("MakanExpress ETL Pipeline — Docker Edition")
    logger.info(f"Run time: {datetime.now().isoformat()}")
    logger.info("=" * 50)
    
    # Extract
    raw = extract()
    
    # Transform
    transformed = transform(raw)
    
    # Load
    load(transformed)
    
    # Print summary
    print("\n" + "=" * 50)
    print("📊 RESTAURANT SUMMARY")
    print("=" * 50)
    print(transformed.to_string(index=False))
    print("=" * 50)
    logger.info("ETL pipeline completed successfully!")


if __name__ == "__main__":
    main()
```

Create `Dockerfile`:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY etl.py .

ENV OUTPUT_PATH=/app/output/summary.csv

CMD ["python", "etl.py"]
```

Now build and run:

```bash
# Build the image
docker build -t makanexpress-etl:v1 .

# Run the container
docker run --name etl-test makanexpress-etl:v1

# Check the logs
docker logs etl-test

# Clean up
docker rm etl-test
```

**Exercise:** What happens if you change `etl.py` and re-run `docker run` without rebuilding? Why?

<details>
<summary>🔑 Answer</summary>

The **old** version of `etl.py` runs. The image was built with the old code. `docker run` creates a container from the **image**, not from your current files.

You need to `docker build` again to include the new code. This is a common mistake!

```bash
# After changing etl.py:
docker build -t makanexpress-etl:v2 .
docker run --name etl-test-v2 makanexpress-etl:v2
```

</details>

### Layers and Caching

This is the KEY concept for efficient Dockerfiles:

```dockerfile
# GOOD ORDER (fast rebuilds):
FROM python:3.11-slim        # Layer 1: Base (cached, rarely changes)
WORKDIR /app                  # Layer 2: Dir (cached)
COPY requirements.txt .       # Layer 3: Deps file (changes sometimes)
RUN pip install -r requirements.txt  # Layer 4: Install (expensive, cached if reqs unchanged)
COPY etl.py .                 # Layer 5: Your code (changes often)

# BAD ORDER (slow rebuilds):
FROM python:3.11-slim
WORKDIR /app
COPY . .                      # ← Copies EVERYTHING first
RUN pip install -r requirements.txt  # ← Re-runs EVERY time any file changes
```

**Why order matters:**
- Docker builds images layer by layer
- Each instruction = one layer
- If a layer hasn't changed, Docker reuses the cached version
- If layer 3 changes, layers 3, 4, 5 must rebuild
- `pip install` is slow (minutes). Only re-run when `requirements.txt` changes.

**Rule: Copy dependency files FIRST, install them, THEN copy your code.**

**Exercise:** Your Dockerfile takes 3 minutes to build because `pip install` runs every time. You only changed one line in `main.py`. How do you fix this?

<details>
<summary>🔑 Answer</summary>

Restructure the Dockerfile to copy `requirements.txt` and install dependencies BEFORE copying your source code:

```dockerfile
FROM python:3.11-slim
WORKDIR /app

# Dependencies first (cached if requirements.txt unchanged)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Code second (changes often, but doesn't invalidate dependency cache)
COPY . .

CMD ["python", "main.py"]
```

Now when you change `main.py`, only the `COPY . .` layer and after rebuild. The expensive `pip install` layer is cached.

</details>

### CMD vs ENTRYPOINT

```dockerfile
# CMD: Default command, can be overridden easily
CMD ["python", "etl.py"]
# Override: docker run my-image python debug.py  ← replaces CMD

# ENTRYPOINT: Command that always runs
ENTRYPOINT ["python"]
# Append: docker run my-image debug.py  ← adds to ENTRYPOINT
# Result: python debug.py

# Best practice for ETL: Use both
ENTRYPOINT ["python", "etl.py"]
CMD ["--mode", "daily"]
# Default: python etl.py --mode daily
# Override args: docker run my-image --mode full
# Result: python etl.py --mode full
```

### .dockerignore

Create `.dockerignore` (like `.gitignore` for Docker builds):

```
__pycache__/
*.pyc
.git/
.env
venv/
*.csv
*.parquet
output/
.DS_Store
README.md
```

**Why:** `COPY . .` copies everything. `.dockerignore` prevents unnecessary files from bloating your image.

### Docker Hub: Sharing Images

```bash
# Login to Docker Hub (create free account at hub.docker.com)
docker login

# Tag your image
docker tag makanexpress-etl:v1 yourusername/makanexpress-etl:v1

# Push to Docker Hub
docker push yourusername/makanexpress-etl:v1

# Anyone can now pull and run:
docker pull yourusername/makanexpress-etl:v1
docker run yourusername/makanexpress-etl:v1
```

**For data engineering:** You won't push to Docker Hub often. More likely:
- Push to AWS ECR (Elastic Container Registry) for AWS deployments
- Use Docker Compose locally for development
- Use pre-built images (Airflow, PostgreSQL, etc.)

---

## 🌙 Evening (1 hour): docker-compose Preview + Quick Reference

### What is docker-compose?

docker-compose lets you define **multi-container applications** in one YAML file. Instead of running:

```bash
docker run -d --name postgres -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:15
docker run -d --name airflow -p 8080:8080 apache/airflow
docker run -d --name my-etl --link postgres:db my-etl-image
```

You write one file:

```yaml
# docker-compose.yml (PREVIEW — tomorrow you'll write this for real)
version: "3.8"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: makanexpress
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  etl:
    build: .
    depends_on:
      - postgres
    environment:
      DB_HOST: postgres
      DB_NAME: makanexpress

volumes:
  pgdata:
```

Then run everything with:
```bash
docker-compose up -d        # Start all services
docker-compose logs -f       # See all logs
docker-compose down          # Stop and remove
```

**Why this matters for data engineers:**
- Airflow + PostgreSQL + your ETL = one command to start
- Same environment on every developer's machine
- Easy teardown and recreation
- This is how real teams develop data pipelines

### MakanExpress Container Architecture (Tomorrow's Build)

```
┌─────────────────────────────────────────────────┐
│          docker-compose (MakanExpress)           │
│                                                  │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐ │
│  │ PostgreSQL  │  │  Airflow   │  │  pgAdmin   │ │
│  │ Port: 5432  │  │ Port: 8080 │  │ Port: 5050│ │
│  │             │  │            │  │           │ │
│  │ Database:   │  │ DAGs:      │  │ DB admin  │ │
│  │ makanexpress│  │ /dags/     │  │ GUI       │ │
│  └──────┬──────┘  └──────┬─────┘  └─────┬─────┘ │
│         │                │               │       │
│         └────────────────┴───────────────┘       │
│                    Docker Network                 │
└─────────────────────────────────────────────────┘
```

### Quick Reference Card

```markdown
# 🐳 Docker Quick Reference for Data Engineers

## Essential Commands
docker build -t name:tag .         # Build image from Dockerfile
docker run name:tag                 # Run container
docker run -d --name X name:tag    # Run in background
docker run -it name:tag bash       # Interactive shell
docker run -v /host:/container X   # Mount volume
docker run -p 8080:80 X            # Map port
docker run -e KEY=VALUE X          # Set env variable

docker ps                           # List running containers
docker ps -a                        # List all containers
docker images                       # List images
docker logs <id>                    # View container logs
docker exec -it <id> bash           # Shell into running container
docker stop <id>                    # Stop container
docker rm <id>                      # Remove container
docker rmi <id>                     # Remove image

docker-compose up -d                # Start all services
docker-compose down                 # Stop all services
docker-compose logs -f              # Follow all logs
docker-compose build                # Rebuild images
docker-compose ps                   # List running services

## Dockerfile Template (Python ETL)
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "main.py"]

## Key Concepts
- Image = template (like a class)
- Container = running instance (like an object)
- Layer = each Dockerfile instruction creates one
- Cache = layers reused if unchanged (put code COPY last!)
- Volume = persistent data outside container
- Bind mount = link host directory to container

## Best Practices
1. Use slim/alpine base images (smaller)
2. Copy requirements.txt first, then code (caching!)
3. Never put secrets in Dockerfile (use ENV + .env)
4. Add .dockerignore
5. One process per container
6. Pin dependency versions (pandas==2.1.4 not pandas)
```

---

### 📝 Today's Checklist

- [ ] Docker installed and `docker run hello-world` works
- [ ] Understand containers vs VMs
- [ ] Ran `alpine` container interactively
- [ ] Wrote a Dockerfile for Python ETL script
- [ ] Built image with `docker build`
- [ ] Ran container successfully
- [ ] Understand layers and caching (why order matters)
- [ ] Understand CMD vs ENTRYPOINT
- [ ] Created `.dockerignore`
- [ ] Read docker-compose preview (ready for tomorrow)

---

*Day 52 complete! Tomorrow: Containerize your entire MakanExpress pipeline with docker-compose.* 🐳🚀
