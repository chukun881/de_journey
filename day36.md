# 📅 Day 36 — Friday, 19 June 2026
# ☁️ AWS Basics: Console, CLI, Free Tier, and Core Concepts

---

## 🎯 Today's Goal

Today you'll set up your AWS account, understand the core concepts, and learn to navigate both the AWS Console (web UI) and AWS CLI (command line). By end of day, you'll be comfortable with the AWS ecosystem and ready to start building.

**Why this matters:** AWS is the dominant cloud platform in Singapore and Southeast Asia. Most SG/MY data engineering jobs require AWS knowledge. Even if a company uses GCP or Azure, the concepts transfer.

---

## ☀️ Morning Block (2 hours): AWS Account Setup & Core Concepts

### Concept 1: What is AWS?

```
Amazon Web Services (AWS) = Cloud Computing Platform

Instead of buying servers, you rent them on-demand.
Pay only for what you use.

For a data engineer, AWS gives you:
├── Storage: S3 (unlimited file storage)
├── Database: RDS (managed PostgreSQL/MySQL)
├── Compute: EC2 (virtual machines), Lambda (serverless)
├── ETL: Glue (managed Spark), Step Functions
├── Analytics: Athena (query S3 with SQL), Redshift (data warehouse)
├── Messaging: SNS, SQS, Kinesis
└── IAM: Identity & Access Management (security)
```

### Concept 2: AWS Global Infrastructure

```
Regions → Availability Zones (AZs) → Data Centers

Singapore: ap-southeast-1 (YOUR DEFAULT REGION)
Tokyo: ap-northeast-1
Sydney: ap-southeast-2
US East (Virginia): us-east-1 (cheapest, most services)
EU West (Ireland): eu-west-1

Why regions matter:
- Data residency: SG/MY companies often require data in SG region
- Latency: closer to users = faster
- Cost: varies by region (US East is usually cheapest)
- Service availability: not all services in all regions
```

### Concept 3: Setting Up Your AWS Account

**IMPORTANT: Stay within Free Tier!** AWS charges after free tier limits.

```bash
# Step 1: Create AWS Account
# 1. Go to https://aws.amazon.com/
# 2. Click "Create an AWS Account"
# 3. Use your email, set a strong password
# 4. Account name: something like "makanexpress-dev"
# 5. You'll need a credit card (for verification, won't be charged if you stay in free tier)
# 6. Choose "Basic Support" (free)

# Step 2: Enable billing alerts (CRITICAL — prevents surprise charges)
# 1. Go to AWS Console → Billing → Billing preferences
# 2. Enable "Receive Billing Alerts"
# 3. Go to CloudWatch → Alarms → Billing → Create alarm
# 4. Set threshold: $1.00 (you get alerted before any real charge)

# Step 3: Set your default region to Singapore
# Console: top right corner → select "Asia Pacific (Singapore) ap-southeast-1"
```

### Concept 4: AWS Free Tier Limits (What's Free for 12 Months)

```
Always Free:
- IAM (users, groups, roles) — unlimited
- Lambda: 1M requests/month
- CloudWatch: 10 custom metrics

Free for 12 Months (from account creation):
- S3: 5 GB storage, 20,000 GET requests, 2,000 PUT requests/month
- RDS: 750 hours/month of db.t3.micro PostgreSQL (enough for 1 DB running 24/7)
- EC2: 750 hours/month of t2.micro (1 instance running 24/7)
- Glue: 1M objects crawled/month, some ETL jobs

⚠️ WARNING: After 12 months, everything costs money!
⚠️ Always check: https://aws.amazon.com/free/
⚠️ Set billing alerts NOW, not later
```

### Concept 5: AWS CLI Setup

```bash
# Install AWS CLI v2
# macOS:
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# Linux:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
# Should show: aws-cli/2.x.x

# Configure with your credentials
# Step 1: Create an IAM user (NOT root — never use root for daily work!)
# Console → IAM → Users → Create User
# Name: "makanexpress-admin"
# Check: "Provide user access to the AWS Management Console"
# Set password
# Attach policy: "AdministratorAccess" (for learning only — use least privilege in production!)

# Step 2: Create access keys for CLI
# IAM → Users → makanexpress-admin → Security credentials → Create access key
# Choose "Command Line Interface (CLI)"
# Save the Access Key ID and Secret Access Key (shown only once!)

# Step 3: Configure CLI
aws configure
# AWS Access Key ID: [paste your key]
# AWS Secret Access Key: [paste your secret]
# Default region name: ap-southeast-1
# Default output format: json

# Test
aws sts get-caller-identity
# Should show your account info

aws ec2 describe-regions --query "Regions[].RegionName" --output table
# Lists all available regions
```

### Concept 6: The AWS Console Tour

```
Key services you'll use as a data engineer:

Storage & Database:
├── S3 (Simple Storage Service) — file storage (like Google Drive for apps)
├── RDS (Relational Database Service) — managed PostgreSQL/MySQL
└── DynamoDB — NoSQL database (serverless)

Compute:
├── EC2 — virtual machines (like renting a server)
├── Lambda — serverless functions (run code without servers)
└── Glue — managed ETL (Spark under the hood)

Analytics:
├── Athena — query S3 files with SQL (no server needed)
├── Redshift — data warehouse (like PostgreSQL but for analytics)
└── Kinesis — real-time data streaming

Security:
├── IAM — who can do what
├── KMS — encryption key management
└── Secrets Manager — store passwords securely

Monitoring:
├── CloudWatch — logs, metrics, alarms
└── CloudTrail — audit log of all API calls
```

### Concept 7: Key AWS Concepts

```
Region: Physical location (Singapore, Tokyo, etc.)
Availability Zone (AZ): Data center within a region (ap-southeast-1a, 1b, 1c)
VPC: Virtual Private Cloud — your isolated network in AWS
Subnet: A slice of your VPC (public or private)
Security Group: Firewall rules for your resources
ARN: Amazon Resource Name — unique ID for every AWS resource
  Example: arn:aws:s3:::makanexpress-data-lake
Tags: Key-value labels for organizing resources (e.g., Project=MakanExpress, Env=Dev)

The ARN format:
arn:aws:SERVICE:REGION:ACCOUNT-ID:RESOURCE
arn:aws:s3:::my-bucket                        (S3 bucket, no region)
arn:aws:rds:ap-southeast-1:123456789:db:my-db  (RDS instance)
arn:aws:iam::123456789:user/admin              (IAM user)
```

---

### 🏋️ Morning Exercises (15 questions)

1. Create your AWS account. Verify you can log in to the console.

2. Set your default region to Singapore (ap-southeast-1). Find the region selector in the console.

3. Enable billing alerts. Create a CloudWatch billing alarm for $1.00.

4. Install AWS CLI v2 and run `aws --version` to verify.

5. Create an IAM user named "makanexpress-admin" with console access and CLI access keys.

6. Run `aws configure` and set your credentials. Test with `aws sts get-caller-identity`.

7. List all AWS regions using the CLI. How many regions are there?

8. What is the difference between a Region and an Availability Zone? Why does it matter for data engineering?

9. Find the AWS Free Tier page. What are the limits for S3 and RDS?

10. Run these CLI commands to explore your account:
    ```bash
    aws iam list-users
    aws s3 ls                    # list S3 buckets (empty for now)
    aws rds describe-db-instances  # list RDS instances (empty for now)
    ```

11. Create a billing budget in AWS Console → Billing → Budgets. Set it to $5/month.

12. What is an ARN? Find the ARN of your IAM user using the CLI.

13. Navigate the AWS Console and find these services: S3, RDS, IAM, Glue, Athena. Bookmark each.

14. Why should you NEVER use the AWS root account for daily work? What's the best practice?

15. **Practical:** Document your AWS setup in a file `aws-setup.md`:
    - Account ID
    - Region
    - IAM user name
    - CLI verification output
    - Billing alert confirmation
    - Free tier limits you need to watch

---

### ✅ Morning Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1-4:** Hands-on setup. Follow the steps in Concepts 3-5.

**Answer 5:**
- Console → IAM → Users → Create user
- Name: makanexpress-admin
- Attach policy: AdministratorAccess
- Create access key for CLI

**Answer 6:**
```bash
aws configure
# Enter your keys, region: ap-southeast-1, output: json

aws sts get-caller-identity
# Returns:
# {
#   "UserId": "AIDAXXXXXXXXXXXX",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/makanexpress-admin"
# }
```

**Answer 7:**
```bash
aws ec2 describe-regions --query "Regions[].RegionName" --output table
# ~30 regions as of 2026
```

**Answer 8:**
- **Region**: A geographic area (e.g., Singapore). Contains multiple AZs.
- **Availability Zone**: A data center within a region (e.g., ap-southeast-1a).
- **Why it matters**: For high availability, spread resources across AZs. For data residency, choose the right region. For latency, pick a region close to users.

**Answer 9:** S3: 5GB storage, 20K GET, 2K PUT/month. RDS: 750 hours/month of db.t3.micro.

**Answer 10:**
```bash
aws iam list-users          # shows your IAM users
aws s3 ls                   # empty (no buckets yet)
aws rds describe-db-instances  # empty (no databases yet)
```

**Answer 11:** Console → AWS Billing → Budgets → Create budget → Set $5/month.

**Answer 12:**
```bash
aws sts get-caller-identity --query "Arn" --output text
# arn:aws:iam::123456789012:user/makanexpress-admin
```
ARN = Amazon Resource Name. Format: `arn:aws:service:region:account:resource`

**Answer 13:** Use the search bar at the top of AWS Console. Bookmark each service page.

**Answer 14:** The root account has unrestricted access to everything, including billing and account closure. If compromised, attacker can destroy everything. Best practice: create IAM users with minimum permissions, enable MFA on root, never share root credentials.

**Answer 15:** Create `aws-setup.md` with all the information from your setup.

</details>

---

## 🌤️ Afternoon Block (2 hours): S3 — The Most Important AWS Service

### Concept 8: S3 Overview

```
S3 = Simple Storage Service
- Unlimited file storage (like Google Drive but for applications)
- Store ANY type of file (CSV, JSON, Parquet, images, logs, etc.)
- 99.999999999% durability (11 nines — your data is safe)
- Central to almost every AWS data pipeline

Common data engineering uses:
1. Data Lake: store raw data files (CSV, JSON, Parquet)
2. ETL staging: temporary storage during pipeline processing
3. Data warehouse loading: export from RDS → S3 → load into Athena/Redshift
4. Log storage: application logs, audit trails
5. Static website hosting (for dashboards)

S3 Structure:
├── Bucket (like a folder/directory — must have globally unique name)
│   ├── Object (a file — key + data + metadata)
│   │   ├── Key: the file path (e.g., "raw/orders/2026/06/12/orders.csv")
│   │   ├── Data: the actual file content
│   │   └── Metadata: content-type, size, last-modified, etc.
│   └── Folder (virtual — S3 is actually flat, but / in key names create "folders")

S3 naming convention for data lakes:
makanexpress-data-lake/
├── raw/              # raw data from sources
│   ├── orders/
│   │   └── dt=2026-06-12/
│   │       └── orders_20260612.csv
│   ├── customers/
│   └── restaurants/
├── processed/        # cleaned/transformed data
│   └── orders/
│       └── dt=2026-06-12/
│           └── orders_clean.parquet
├── analytics/        # aggregated data for dashboards
│   └── daily_revenue/
│       └── dt=2026-06-12/
│           └── daily_revenue.csv
└── scripts/          # ETL scripts
    └── glue_jobs/
```

### Concept 9: Creating S3 Buckets

```bash
# Create a bucket (name must be globally unique, lowercase, no underscores)
aws s3 mb s3://makanexpress-data-lake-dev
# "mb" = make bucket

# If name is taken, try:
aws s3 mb s3://makanexpress-datalake-yourname-2026

# List all buckets
aws s3 ls

# Create "folders" (S3 doesn't have real folders, but we can create prefix objects)
aws s3api put-object --bucket makanexpress-data-lake-dev --key "raw/"
aws s3api put-object --bucket makanexpress-data-lake-dev --key "processed/"
aws s3api put-object --bucket makanexpress-data-lake-dev --key "analytics/"

# Upload a file
echo "order_id,customer_id,amount,date" > /tmp/test_orders.csv
echo "1,101,25.50,2026-06-12" >> /tmp/test_orders.csv
echo "2,102,18.00,2026-06-12" >> /tmp/test_orders.csv

aws s3 cp /tmp/test_orders.csv s3://makanexpress-data-lake-dev/raw/orders/dt=2026-06-12/orders.csv

# List bucket contents
aws s3 ls s3://makanexpress-data-lake-dev/
aws s3 ls s3://makanexpress-data-lake-dev/raw/orders/dt=2026-06-12/

# Download a file
aws s3 cp s3://makanexpress-data-lake-dev/raw/orders/dt=2026-06-12/orders.csv /tmp/downloaded_orders.csv

# Delete a file
aws s3 rm s3://makanexpress-data-lake-dev/raw/orders/dt=2026-06-12/orders.csv

# Sync a local directory to S3 (like rsync)
mkdir -p /tmp/makanexpress_data/raw/orders
echo "3,103,30.00,2026-06-13" > /tmp/makanexpress_data/raw/orders/orders_20260613.csv
aws s3 sync /tmp/makanexpress_data/ s3://makanexpress-data-lake-dev/raw/

# Get file metadata
aws s3api head-object --bucket makanexpress-data-lake-dev --key "raw/orders/orders_20260613.csv"
```

### Concept 10: S3 with Python (boto3)

```python
"""
Using S3 with Python's boto3 library.
This is how you'd interact with S3 in an ETL pipeline.
"""

import boto3
import pandas as pd
from io import StringIO, BytesIO

# Create S3 client
s3 = boto3.client('s3', region_name='ap-southeast-1')

# ─── Upload files ───

# Upload a simple string
s3.put_object(
    Bucket='makanexpress-data-lake-dev',
    Key='raw/test/hello.txt',
    Body=b'Hello from MakanExpress!'
)

# Upload a CSV from Pandas DataFrame
df = pd.DataFrame({
    'order_id': [1, 2, 3],
    'customer_id': [101, 102, 103],
    'amount': [25.50, 18.00, 30.00],
    'date': ['2026-06-12', '2026-06-12', '2026-06-12']
})

csv_buffer = StringIO()
df.to_csv(csv_buffer, index=False)
s3.put_object(
    Bucket='makanexpress-data-lake-dev',
    Key='raw/orders/dt=2026-06-12/orders.csv',
    Body=csv_buffer.getvalue()
)
print("✅ Uploaded CSV to S3")

# Upload as Parquet (compressed columnar format — better for analytics)
parquet_buffer = BytesIO()
df.to_parquet(parquet_buffer, index=False)
s3.put_object(
    Bucket='makanexpress-data-lake-dev',
    Key='processed/orders/dt=2026-06-12/orders.parquet',
    Body=parquet_buffer.getvalue()
)
print("✅ Uploaded Parquet to S3")

# ─── Download and read files ───

# Download CSV
response = s3.get_object(
    Bucket='makanexpress-data-lake-dev',
    Key='raw/orders/dt=2026-06-12/orders.csv'
)
csv_content = response['Body'].read().decode('utf-8')
df_downloaded = pd.read_csv(StringIO(csv_content))
print(df_downloaded)

# Download Parquet
response = s3.get_object(
    Bucket='makanexpress-data-lake-dev',
    Key='processed/orders/dt=2026-06-12/orders.parquet'
)
df_parquet = pd.read_parquet(BytesIO(response['Body'].read()))
print(df_parquet)

# ─── List objects ───

# List all objects in a "folder"
response = s3.list_objects_v2(
    Bucket='makanexpress-data-lake-dev',
    Prefix='raw/orders/'
)
for obj in response.get('Contents', []):
    print(f"  {obj['Key']} ({obj['Size']} bytes)")

# ─── Delete objects ───

s3.delete_object(
    Bucket='makanexpress-data-lake-dev',
    Key='raw/test/hello.txt'
)
print("✅ Deleted test file")
```

### Concept 11: CSV vs Parquet — Why Parquet Wins for Data Engineering

```
CSV:
- Human readable
- Universal compatibility
- SLOW to read (parse text every time)
- LARGE file size (no compression)
- No schema embedded

Parquet:
- Columnar format (read only columns you need)
- Built-in compression (50-80% smaller than CSV)
- Schema embedded (data types preserved)
- FAST for analytics queries
- Standard in data engineering (Spark, Athena, Glue all use it)

Example:
orders.csv      = 100 MB
orders.parquet  = 25 MB  (75% smaller!)

Reading 1 column from CSV    = read entire 100 MB file
Reading 1 column from Parquet = read only that column (~5 MB)

Rule of thumb:
- Raw/landing zone: CSV or JSON (for debugging)
- Processed/analytics: Always Parquet
```

```python
# Convert all CSVs in S3 to Parquet
def csv_to_parquet_in_s3(bucket, csv_key, parquet_key):
    """Read CSV from S3, convert to Parquet, upload back."""
    import boto3
    import pandas as pd
    from io import StringIO, BytesIO
    
    s3 = boto3.client('s3')
    
    # Read CSV
    response = s3.get_object(Bucket=bucket, Key=csv_key)
    df = pd.read_csv(StringIO(response['Body'].read().decode('utf-8')))
    
    # Convert to Parquet
    parquet_buffer = BytesIO()
    df.to_parquet(parquet_buffer, index=False, engine='pyarrow')
    
    # Upload Parquet
    s3.put_object(Bucket=bucket, Key=parquet_key, Body=parquet_buffer.getvalue())
    
    print(f"✅ Converted {csv_key} → {parquet_key}")
    print(f"   CSV rows: {len(df)}, Columns: {list(df.columns)}")

# Install pyarrow first: pip install pyarrow
```

### Concept 12: S3 Storage Classes

```
Not all data needs fast, expensive storage.

S3 Standard:      Hot data, frequent access (your default)
S3 Intelligent:   Auto-moves between Standard and Glacier based on usage
S3 Standard-IA:   Infrequent Access — cheaper storage, more expensive retrieval
S3 Glacier:       Archive — very cheap storage, takes hours to retrieve
S3 Deep Archive:  Cheapest — for compliance/backup, takes 12 hours to retrieve

For a data engineer:
- Last 30 days → S3 Standard (fast queries)
- 30-90 days → Standard-IA (saves money)
- 90+ days → Glacier (archive, rarely accessed)

Lifecycle policies automate this:
"After 90 days, move raw data to Glacier"
```

---

### 🏋️ Afternoon Exercises (20 questions)

16. Create an S3 bucket named `makanexpress-data-lake-dev` in Singapore region. Verify it appears in `aws s3 ls`.

17. Upload a CSV file to your bucket under `raw/orders/dt=2026-06-12/orders.csv`. Verify with `aws s3 ls`.

18. Download the file back and verify the content matches.

19. Upload 5 different files to different "folders" in your bucket (raw/, processed/, analytics/). List all files recursively.

20. Write a Python script using boto3 that:
    - Creates a DataFrame with 100 fake orders
    - Uploads it as CSV to `raw/orders/`
    - Reads it back from S3
    - Converts to Parquet and uploads to `processed/orders/`
    - Prints file sizes of both

21. Compare file sizes: upload the same data as CSV and Parquet. How much smaller is Parquet?

22. Write a Python function `upload_dataframe_to_s3(df, bucket, key)` that handles both CSV and Parquet formats based on the file extension.

23. Write a Python function `read_dataframe_from_s3(bucket, key)` that reads CSV or Parquet from S3 into a Pandas DataFrame.

24. Use `aws s3 sync` to upload an entire local directory to S3. Verify all files are present.

25. Delete all test files from your S3 bucket using the CLI. Then delete the bucket itself.

26. **S3 security:** Create a bucket with public access BLOCKED (default). Verify you can't access it via a web browser.

27. Write a Python script that lists all objects in a bucket and prints: key name, size (human-readable), and last modified date.

28. Create a script that uploads your MakanExpress sample data (from Day 28) to S3 in the correct data lake structure (raw/customers/, raw/orders/, etc.).

29. Write a function that partitions data by date when uploading: given a DataFrame with an `order_date` column, split by date and upload each date to a separate prefix (e.g., `raw/orders/dt=2026-06-12/`, `raw/orders/dt=2026-06-13/`).

30. **Practical:** Build a complete S3 data lake utility:
    ```python
    # s3_utils.py
    class S3DataLake:
        def __init__(self, bucket_name):
            ...
        def upload_raw(self, df, table_name, date):
            """Upload DataFrame to raw/ zone as CSV."""
            ...
        def upload_processed(self, df, table_name, date):
            """Upload DataFrame to processed/ zone as Parquet."""
            ...
        def list_raw(self, table_name, date=None):
            """List raw files for a table, optionally filtered by date."""
            ...
        def read_processed(self, table_name, date):
            """Read Parquet from processed/ zone into DataFrame."""
            ...
        def get_size_report(self):
            """Print total size by zone (raw/ vs processed/)."""
            ...
    ```
    Test it with your MakanExpress data.

31. What happens if you try to create a bucket with a name that already belongs to someone else?

32. How would you structure an S3 data lake for a company with 3 data sources (orders, customers, products) and 2 processing stages (raw, processed)? Draw the folder structure.

33. Research: What is S3 Event Notifications? How could you trigger an ETL pipeline when a new file lands in S3?

34. Install `pyarrow` and verify Parquet support: `python -c "import pyarrow; print(pyarrow.__version__)"`.

35. **Challenge:** Build a complete data pipeline script:
    1. Generate fake order data for the past 7 days (100 orders/day)
    2. Upload each day's data as CSV to `raw/orders/dt=YYYY-MM-DD/`
    3. Read each day's CSV, clean it (remove nulls, fix types), convert to Parquet
    4. Upload Parquet to `processed/orders/dt=YYYY-MM-DD/`
    5. Print a summary: total files uploaded, total size raw vs processed, compression ratio

---

### ✅ Afternoon Answers

<details>
<summary>🔑 Click to reveal selected answers</summary>

**Answer 16:**
```bash
aws s3 mb s3://makanexpress-data-lake-dev --region ap-southeast-1
aws s3 ls
# Should show: 2026-XX-XX HH:MM:SS makanexpress-data-lake-dev
```

**Answer 20:**
```python
import boto3
import pandas as pd
from io import StringIO, BytesIO
import numpy as np

# Generate fake orders
np.random.seed(42)
df = pd.DataFrame({
    'order_id': range(1, 101),
    'customer_id': np.random.randint(1, 50, 100),
    'restaurant_id': np.random.randint(1, 15, 100),
    'amount': np.round(np.random.uniform(5, 50, 100), 2),
    'order_date': '2026-06-12'
})

s3 = boto3.client('s3', region_name='ap-southeast-1')
bucket = 'makanexpress-data-lake-dev'

# Upload CSV
csv_buffer = StringIO()
df.to_csv(csv_buffer, index=False)
csv_size = len(csv_buffer.getvalue().encode('utf-8'))
s3.put_object(Bucket=bucket, Key='raw/orders/dt=2026-06-12/orders.csv', Body=csv_buffer.getvalue())

# Upload Parquet
parquet_buffer = BytesIO()
df.to_parquet(parquet_buffer, index=False)
parquet_size = parquet_buffer.tell()
s3.put_object(Bucket=bucket, Key='processed/orders/dt=2026-06-12/orders.parquet', Body=parquet_buffer.getvalue())

print(f"CSV size: {csv_size} bytes")
print(f"Parquet size: {parquet_size} bytes")
print(f"Compression ratio: {round((1 - parquet_size/csv_size) * 100, 1)}%")

# Read back
resp = s3.get_object(Bucket=bucket, Key='processed/orders/dt=2026-06-12/orders.parquet')
df_read = pd.read_parquet(BytesIO(resp['Body'].read()))
print(f"\nRead back {len(df_read)} rows from Parquet")
print(df_read.head())
```

**Answer 22:**
```python
def upload_dataframe_to_s3(df, bucket, key):
    import boto3
    from io import StringIO, BytesIO
    
    s3 = boto3.client('s3')
    
    if key.endswith('.csv'):
        buffer = StringIO()
        df.to_csv(buffer, index=False)
        body = buffer.getvalue()
    elif key.endswith('.parquet'):
        buffer = BytesIO()
        df.to_parquet(buffer, index=False)
        body = buffer.getvalue()
    else:
        raise ValueError("Unsupported format. Use .csv or .parquet")
    
    s3.put_object(Bucket=bucket, Key=key, Body=body)
    print(f"✅ Uploaded {len(df)} rows to s3://{bucket}/{key}")

def read_dataframe_from_s3(bucket, key):
    import boto3
    import pandas as pd
    from io import StringIO, BytesIO
    
    s3 = boto3.client('s3')
    response = s3.get_object(Bucket=bucket, Key=key)
    
    if key.endswith('.csv'):
        return pd.read_csv(StringIO(response['Body'].read().decode('utf-8')))
    elif key.endswith('.parquet'):
        return pd.read_parquet(BytesIO(response['Body'].read()))
    else:
        raise ValueError("Unsupported format")
```

**Answer 30:**
```python
import boto3
import pandas as pd
from io import StringIO, BytesIO

class S3DataLake:
    def __init__(self, bucket_name, region='ap-southeast-1'):
        self.bucket = bucket_name
        self.s3 = boto3.client('s3', region_name=region)
    
    def upload_raw(self, df, table_name, date):
        key = f"raw/{table_name}/dt={date}/{table_name}.csv"
        csv_buffer = StringIO()
        df.to_csv(csv_buffer, index=False)
        self.s3.put_object(Bucket=self.bucket, Key=key, Body=csv_buffer.getvalue())
        print(f"✅ Uploaded {len(df)} rows to {key}")
    
    def upload_processed(self, df, table_name, date):
        key = f"processed/{table_name}/dt={date}/{table_name}.parquet"
        parquet_buffer = BytesIO()
        df.to_parquet(parquet_buffer, index=False)
        self.s3.put_object(Bucket=self.bucket, Key=key, Body=parquet_buffer.getvalue())
        print(f"✅ Uploaded {len(df)} rows to {key}")
    
    def list_raw(self, table_name, date=None):
        prefix = f"raw/{table_name}/"
        if date:
            prefix += f"dt={date}/"
        response = self.s3.list_objects_v2(Bucket=self.bucket, Prefix=prefix)
        return [obj['Key'] for obj in response.get('Contents', [])]
    
    def read_processed(self, table_name, date):
        key = f"processed/{table_name}/dt={date}/{table_name}.parquet"
        response = self.s3.get_object(Bucket=self.bucket, Key=key)
        return pd.read_parquet(BytesIO(response['Body'].read()))
    
    def get_size_report(self):
        zones = {'raw': 0, 'processed': 0, 'analytics': 0}
        paginator = self.s3.get_paginator('list_objects_v2')
        for page in paginator.paginate(Bucket=self.bucket):
            for obj in page.get('Contents', []):
                for zone in zones:
                    if obj['Key'].startswith(zone + '/'):
                        zones[zone] += obj['Size']
        
        for zone, size in zones.items():
            print(f"  {zone}/: {size / 1024:.1f} KB")

# Usage
lake = S3DataLake('makanexpress-data-lake-dev')
lake.upload_raw(df_orders, 'orders', '2026-06-12')
lake.upload_processed(df_orders, 'orders', '2026-06-12')
print(lake.list_raw('orders'))
print(lake.read_processed('orders', '2026-06-12').head())
lake.get_size_report()
```

**Answer 35:**
```python
import boto3
import pandas as pd
import numpy as np
from io import StringIO, BytesIO
from datetime import datetime, timedelta

s3 = boto3.client('s3', region_name='ap-southeast-1')
bucket = 'makanexpress-data-lake-dev'

np.random.seed(42)
total_raw_size = 0
total_processed_size = 0
file_count = 0

for day_offset in range(7):
    date = (datetime(2026, 6, 12) - timedelta(days=day_offset)).strftime('%Y-%m-%d')
    
    # Generate fake data
    n = 100
    df = pd.DataFrame({
        'order_id': range(day_offset * 100 + 1, day_offset * 100 + n + 1),
        'customer_id': np.random.randint(1, 50, n),
        'restaurant_id': np.random.randint(1, 15, n),
        'amount': np.round(np.random.uniform(5, 50, n), 2),
        'order_date': date
    })
    
    # Upload CSV (raw)
    csv_buffer = StringIO()
    df.to_csv(csv_buffer, index=False)
    csv_data = csv_buffer.getvalue()
    total_raw_size += len(csv_data.encode('utf-8'))
    s3.put_object(Bucket=bucket, Key=f'raw/orders/dt={date}/orders.csv', Body=csv_data)
    
    # Clean and upload Parquet (processed)
    df_clean = df.dropna()
    df_clean['amount'] = df_clean['amount'].astype(float)
    parquet_buffer = BytesIO()
    df_clean.to_parquet(parquet_buffer, index=False)
    total_processed_size += parquet_buffer.tell()
    s3.put_object(Bucket=bucket, Key=f'processed/orders/dt={date}/orders.parquet', Body=parquet_buffer.getvalue())
    
    file_count += 2
    print(f"  {date}: {len(df_clean)} orders uploaded")

print(f"\n📊 Summary:")
print(f"  Files uploaded: {file_count}")
print(f"  Raw (CSV): {total_raw_size / 1024:.1f} KB")
print(f"  Processed (Parquet): {total_processed_size / 1024:.1f} KB")
print(f"  Compression: {round((1 - total_processed_size / total_raw_size) * 100, 1)}%")
```

</details>

---

## 🌙 Evening Block (1 hour): Review & Practice

### AWS CLI Cheat Sheet

```bash
# S3 operations
aws s3 ls                                    # list buckets
aws s3 ls s3://bucket-name/                  # list bucket contents
aws s3 mb s3://bucket-name                   # create bucket
aws s3 rb s3://bucket-name --force           # delete bucket + contents
aws s3 cp file.csv s3://bucket/path/         # upload file
aws s3 cp s3://bucket/path/file.csv ./       # download file
aws s3 sync ./local/ s3://bucket/path/       # sync directory
aws s3 rm s3://bucket/path/file.csv          # delete file
aws s3 rm s3://bucket/path/ --recursive      # delete all in "folder"

# General
aws sts get-caller-identity                  # who am I?
aws iam list-users                           # list IAM users
aws configure                                # set credentials
aws --region ap-southeast-1 s3 ls            # override region
```

### 📝 Today's Checklist

- [ ] AWS account created with billing alerts
- [ ] IAM user created (NOT using root)
- [ ] AWS CLI installed and configured
- [ ] S3 bucket created in Singapore region
- [ ] Uploaded and downloaded files via CLI and Python
- [ ] Understand CSV vs Parquet
- [ ] Can use boto3 to interact with S3
- [ ] aws-setup.md documentation written
- [ ] Ready for tomorrow: S3 advanced features

### 📖 Further Reading (Free)

- [AWS Free Tier Details](https://aws.amazon.com/free/)
- [AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/)
- [S3 Getting Started](https://docs.aws.amazon.com/AmazonS3/latest/userguide/GetStartedWithS3.html)
- [boto3 S3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html)

---

*Day 36 complete! Tomorrow: S3 Deep Dive — versioning, lifecycle policies, and data lake patterns.* 🪣
