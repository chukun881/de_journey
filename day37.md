# 📅 Day 37 — Saturday, 20 June 2026
# 🪣 S3 Deep Dive: Versioning, Lifecycle, Encryption, and Data Lake Patterns

---

## 🎯 Today's Goal

Yesterday you learned S3 basics. Today you'll learn the features that make S3 production-ready: versioning, lifecycle policies, encryption, and proper data lake organization. These are the things that separate "I uploaded a file" from "I built a data lake."

---

## ☀️ Morning Block (2 hours): S3 Advanced Features

### Concept 1: S3 Versioning

```bash
# Enable versioning on a bucket
aws s3api put-bucket-versioning \
    --bucket makanexpress-data-lake-dev \
    --versioning-configuration Status=Enabled

# Check versioning status
aws s3api get-bucket-versioning \
    --bucket makanexpress-data-lake-dev

# Upload a file, then upload an updated version
echo "version 1" > /tmp/data.csv
aws s3 cp /tmp/data.csv s3://makanexpress-data-lake-dev/raw/test/data.csv

echo "version 2" > /tmp/data.csv
aws s3 cp /tmp/data.csv s3://makanexpress-data-lake-dev/raw/test/data.csv

# List all versions of the file
aws s3api list-object-versions \
    --bucket makanexpress-data-lake-dev \
    --prefix raw/test/data.csv

# Download a specific version
aws s3api get-object \
    --bucket makanexpress-data-lake-dev \
    --key raw/test/data.csv \
    --version-id VERSION_ID_HERE \
    /tmp/data_v1.csv

# Why versioning matters for data engineering:
# 1. Accidental overwrite? Roll back to previous version
# 2. Audit trail: see who changed what, when
# 3. Reprocess data: keep both old and new versions
# ⚠️ Cost: every version is stored and billed!
```

### Concept 2: S3 Lifecycle Policies

```python
"""
Lifecycle policies automatically transition or expire objects.
Saves money by moving old data to cheaper storage.

Example policy:
- After 30 days → move to S3 Standard-IA (infrequent access)
- After 90 days → move to Glacier (archive)
- After 365 days → delete
"""

import boto3
import json

s3 = boto3.client('s3', region_name='ap-southeast-1')

lifecycle_config = {
    "Rules": [
        {
            "ID": "RawDataLifecycle",
            "Status": "Enabled",
            "Filter": {"Prefix": "raw/"},
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "STANDARD_IA"  # infrequent access
                },
                {
                    "Days": 90,
                    "StorageClass": "GLACIER"  # archive
                }
            ],
            "Expiration": {
                "Days": 365  # delete after 1 year
            }
        },
        {
            "ID": "ProcessedDataLifecycle",
            "Status": "Enabled",
            "Filter": {"Prefix": "processed/"},
            "Transitions": [
                {
                    "Days": 90,
                    "StorageClass": "GLACIER"
                }
            ],
            "Expiration": {
                "Days": 730  # keep processed data for 2 years
            }
        },
        {
            "ID": "TempDataCleanup",
            "Status": "Enabled",
            "Filter": {"Prefix": "temp/"},
            "Expiration": {
                "Days": 7  # delete temp files after 7 days
            }
        }
    ]
}

s3.put_bucket_lifecycle_configuration(
    Bucket='makanexpress-data-lake-dev',
    LifecycleConfiguration=lifecycle_config
)

print("✅ Lifecycle policies configured")
```

### Concept 3: S3 Encryption

```python
"""
S3 encryption options:
1. Server-Side Encryption (SSE) — AWS encrypts data on disk
   - SSE-S3: AWS manages keys (default, free)
   - SSE-KMS: You manage keys via KMS (audit trail, costs money)
   - SSE-C: You provide the key
2. Client-Side Encryption — you encrypt before uploading

For data engineering: SSE-S3 is usually enough.
"""

# Enable default encryption on bucket
s3.put_bucket_encryption(
    Bucket='makanexpress-data-lake-dev',
    ServerSideEncryptionConfiguration={
        'Rules': [{
            'ApplyServerSideEncryptionByDefault': {
                'SSEAlgorithm': 'AES256'  # SSE-S3
            },
            'BucketKeyEnabled': True
        }]
    }
)

# Upload with encryption (automatic with bucket default)
s3.put_object(
    Bucket='makanexpress-data-lake-dev',
    Key='raw/secret.csv',
    Body=b'sensitive data here'
)

# Verify encryption
response = s3.head_object(
    Bucket='makanexpress-data-lake-dev',
    Key='raw/secret.csv'
)
print(f"Encryption: {response.get('ServerSideEncryption', 'None')}")
```

### Concept 4: S3 Bucket Policies & Security

```python
"""
S3 Security layers:
1. IAM policies — who can access (user-level)
2. Bucket policies — what can access this bucket (bucket-level)
3. ACLs — object-level (legacy, avoid)
4. Block Public Access — prevents accidental public exposure (ALWAYS ENABLE)
"""

# Block all public access (ALWAYS do this for data lakes!)
s3.put_public_access_block(
    Bucket='makanexpress-data-lake-dev',
    PublicAccessBlockConfiguration={
        'BlockPublicAcls': True,
        'IgnorePublicAcls': True,
        'BlockPublicPolicy': True,
        'RestrictPublicBuckets': True
    }
)

# Example bucket policy: allow only specific IAM role to read/write
bucket_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:user/makanexpress-admin"
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::makanexpress-data-lake-dev",
                "arn:aws:s3:::makanexpress-data-lake-dev/*"
            ]
        }
    ]
}

s3.put_bucket_policy(
    Bucket='makanexpress-data-lake-dev',
    Policy=json.dumps(bucket_policy)
)
```

### Concept 5: S3 Presigned URLs

```python
"""
Presigned URLs: temporary access to private S3 objects.
Useful for: sharing a file with someone without making the whole bucket public.
"""

# Generate a presigned URL (valid for 1 hour)
url = s3.generate_presigned_url(
    'get_object',
    Params={
        'Bucket': 'makanexpress-data-lake-dev',
        'Key': 'processed/orders/dt=2026-06-12/orders.parquet'
    },
    ExpiresIn=3600  # 1 hour
)
print(f"Presigned URL (valid 1 hour): {url}")

# Use case: let a dashboard tool download data without AWS credentials
```

---

## 🌤️ Afternoon Block (2 hours): Data Lake Design Patterns

### Concept 6: The Data Lake House Pattern

```
Modern data architecture (what most SG/MY companies use):

Sources → Raw (S3) → Processed (S3) → Analytics (Athena/Redshift)
             ↑              ↑                    ↑
          Landing zone   Clean zone           Query zone

Three-zone data lake:

1. RAW zone (Bronze)
   - Exact copy of source data
   - Immutable (never modify raw data)
   - Partitioned by date: raw/orders/dt=2026-06-12/
   - Format: CSV, JSON (easy to debug)
   - Retention: 90 days Standard, then Glacier

2. PROCESSED zone (Silver)
   - Cleaned, deduplicated, typed
   - Enriched with derived columns
   - Partitioned by date: processed/orders/dt=2026-06-12/
   - Format: Parquet (compressed, fast)
   - Retention: 2 years

3. ANALYTICS zone (Gold)
   - Aggregated, business-ready
   - Star schema tables exported as Parquet
   - analytics/daily_revenue/dt=2026-06-12/
   - Format: Parquet
   - Retention: varies by business need
```

### Concept 7: Partitioning Strategy

```
Partitioning = organizing data so queries scan LESS data.

BAD (no partitioning):
  s3://bucket/all_orders/orders.parquet  (one giant file)

GOOD (partitioned by date):
  s3://bucket/processed/orders/dt=2026-06-12/orders.parquet
  s3://bucket/processed/orders/dt=2026-06-13/orders.parquet

WHY: When Athena queries "SELECT * FROM orders WHERE dt='2026-06-12'",
     it only reads that one partition, not all data.

Common partition patterns for data engineering:
  dt=YYYY-MM-DD        (daily — most common)
  dt=YYYY-MM            (monthly — for large datasets)
  year=YYYY/month=MM/day=DD  (hive-style — Spark compatible)
  region=SG/dt=YYYY-MM-DD    (multi-dimensional)

Partitioning rules:
1. Partition by the column you filter on most (usually date)
2. Avoid too many small files (< 1MB per partition is bad)
3. Aim for 100MB-1GB per partition file
4. Don't over-partition (too many small files = slow)
```

```python
# Example: Partition data by date when uploading
import pandas as pd

def upload_partitioned_data(df, bucket, table_name, date_column='order_date'):
    """Upload DataFrame partitioned by date column."""
    s3 = boto3.client('s3', region_name='ap-southeast-1')
    
    # Group by date
    for date, group in df.groupby(date_column):
        date_str = pd.to_datetime(date).strftime('%Y-%m-%d')
        key = f'processed/{table_name}/dt={date_str}/{table_name}.parquet'
        
        parquet_buffer = BytesIO()
        group.to_parquet(parquet_buffer, index=False)
        s3.put_object(Bucket=bucket, Key=key, Body=parquet_buffer.getvalue())
        
        print(f"  ✅ {date_str}: {len(group)} rows → {key}")
    
    print(f"✅ Uploaded {len(df)} total rows across {df[date_column].nunique()} partitions")
```

### Concept 8: File Format Best Practices

```python
"""
Parquet Best Practices for Data Engineering:

1. Use Snappy compression (default, fast)
2. Set row group size to ~128MB
3. Partition by date on S3
4. Use appropriate data types (don't store strings as numbers)
5. Include metadata in Parquet file
"""

import pyarrow as pa
import pyarrow.parquet as pq

# Write Parquet with custom settings
def write_optimized_parquet(df, path):
    table = pa.Table.from_pandas(df)
    pq.write_table(
        table,
        path,
        compression='snappy',       # fast compression
        row_group_size=128 * 1024 * 1024,  # 128MB row groups
        use_dictionary=True,          # better compression for repeated values
        write_statistics=True,        # enables column statistics for pruning
    )
```

### Concept 9: S3 + Athena Query Pattern

```sql
-- Athena lets you query S3 files with SQL — no database needed!
-- This is a preview of what you'll learn in Week 7.

-- Step 1: Create an external table pointing to S3
-- (In Athena console or via Glue crawler)

-- Step 2: Query it
SELECT 
    order_date,
    COUNT(*) as order_count,
    SUM(amount) as total_revenue
FROM processed_orders
WHERE dt BETWEEN '2026-06-01' AND '2026-06-12'
GROUP BY order_date
ORDER BY order_date;

-- Athena only scans the partitions you query!
-- Cost: $5 per TB of data scanned
-- With Parquet + partitioning, typical query scans < 1 GB = fractions of a cent
```

### Concept 10: Complete Data Lake Script

```python
"""
makanexpress_data_lake.py
Complete data lake setup for MakanExpress.
Run this to set up the entire S3 structure.
"""

import boto3
import pandas as pd
import numpy as np
from io import StringIO, BytesIO
from datetime import datetime, timedelta

class MakanExpressDataLake:
    """Manages the MakanExpress data lake on S3."""
    
    def __init__(self, bucket_name, region='ap-southeast-1'):
        self.bucket = bucket_name
        self.region = region
        self.s3 = boto3.client('s3', region_name=region)
    
    def setup_bucket(self):
        """Create and configure the S3 bucket."""
        try:
            self.s3.create_bucket(
                Bucket=self.bucket,
                CreateBucketConfiguration={'LocationConstraint': self.region}
            )
            print(f"✅ Created bucket: {self.bucket}")
        except self.s3.exceptions.BucketAlreadyOwnedByYou:
            print(f"ℹ️  Bucket already exists: {self.bucket}")
        
        # Block public access
        self.s3.put_public_access_block(
            Bucket=self.bucket,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
        
        # Enable versioning
        self.s3.put_bucket_versioning(
            Bucket=self.bucket,
            VersioningConfiguration={'Status': 'Enabled'}
        )
        
        # Enable default encryption
        self.s3.put_bucket_encryption(
            Bucket=self.bucket,
            ServerSideEncryptionConfiguration={
                'Rules': [{
                    'ApplyServerSideEncryptionByDefault': {'SSEAlgorithm': 'AES256'},
                    'BucketKeyEnabled': True
                }]
            }
        )
        
        # Set lifecycle policy
        self.s3.put_bucket_lifecycle_configuration(
            Bucket=self.bucket,
            LifecycleConfiguration={
                'Rules': [
                    {
                        'ID': 'RawDataArchive',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'raw/'},
                        'Transitions': [
                            {'Days': 30, 'StorageClass': 'STANDARD_IA'},
                            {'Days': 90, 'StorageClass': 'GLACIER'}
                        ],
                        'Expiration': {'Days': 365}
                    },
                    {
                        'ID': 'TempCleanup',
                        'Status': 'Enabled',
                        'Filter': {'Prefix': 'temp/'},
                        'Expiration': {'Days': 7}
                    }
                ]
            }
        )
        
        print("✅ Configured: public access blocked, versioning, encryption, lifecycle")
    
    def upload_raw_csv(self, df, table_name, date):
        """Upload raw data as CSV."""
        key = f'raw/{table_name}/dt={date}/{table_name}.csv'
        csv_buffer = StringIO()
        df.to_csv(csv_buffer, index=False)
        self.s3.put_object(Bucket=self.bucket, Key=key, Body=csv_buffer.getvalue())
        return key
    
    def upload_processed_parquet(self, df, table_name, date):
        """Upload processed data as Parquet."""
        key = f'processed/{table_name}/dt={date}/{table_name}.parquet'
        parquet_buffer = BytesIO()
        df.to_parquet(parquet_buffer, index=False)
        self.s3.put_object(Bucket=self.bucket, Key=key, Body=parquet_buffer.getvalue())
        return key
    
    def upload_analytics(self, df, table_name, date):
        """Upload analytics-ready aggregated data."""
        key = f'analytics/{table_name}/dt={date}/{table_name}.parquet'
        parquet_buffer = BytesIO()
        df.to_parquet(parquet_buffer, index=False)
        self.s3.put_object(Bucket=self.bucket, Key=key, Body=parquet_buffer.getvalue())
        return key
    
    def list_files(self, prefix=''):
        """List all files under a prefix."""
        files = []
        paginator = self.s3.get_paginator('list_objects_v2')
        for page in paginator.paginate(Bucket=self.bucket, Prefix=prefix):
            for obj in page.get('Contents', []):
                files.append({
                    'key': obj['Key'],
                    'size_bytes': obj['Size'],
                    'last_modified': obj['LastModified'].isoformat()
                })
        return files
    
    def read_parquet(self, key):
        """Read a Parquet file from S3."""
        response = self.s3.get_object(Bucket=self.bucket, Key=key)
        return pd.read_parquet(BytesIO(response['Body'].read()))
    
    def get_inventory(self):
        """Get a summary of all data in the lake."""
        zones = {'raw': {'files': 0, 'bytes': 0}, 
                 'processed': {'files': 0, 'bytes': 0},
                 'analytics': {'files': 0, 'bytes': 0}}
        
        paginator = self.s3.get_paginator('list_objects_v2')
        for page in paginator.paginate(Bucket=self.bucket):
            for obj in page.get('Contents', []):
                for zone in zones:
                    if obj['Key'].startswith(zone + '/'):
                        zones[zone]['files'] += 1
                        zones[zone]['bytes'] += obj['Size']
        
        print(f"\n📊 Data Lake Inventory: s3://{self.bucket}/")
        print(f"{'Zone':<15} {'Files':<10} {'Size':<15}")
        print("-" * 40)
        for zone, info in zones.items():
            size_mb = info['bytes'] / (1024 * 1024)
            print(f"{zone + '/':<15} {info['files']:<10} {size_mb:.1f} MB")

# Usage
if __name__ == '__main__':
    lake = MakanExpressDataLake('makanexpress-data-lake-dev')
    lake.setup_bucket()
    
    # Upload sample data for the past 7 days
    for day_offset in range(7):
        date = (datetime(2026, 6, 19) - timedelta(days=day_offset)).strftime('%Y-%m-%d')
        
        # Generate fake orders
        orders = pd.DataFrame({
            'order_id': range(day_offset * 100 + 1, day_offset * 100 + 101),
            'customer_id': np.random.randint(1, 50, 100),
            'restaurant_id': np.random.randint(1, 15, 100),
            'amount': np.round(np.random.uniform(5, 50, 100), 2),
            'order_date': date
        })
        
        # Raw zone: CSV
        lake.upload_raw_csv(orders, 'orders', date)
        
        # Processed zone: Parquet
        lake.upload_processed_parquet(orders, 'orders', date)
    
    # Analytics zone: daily summary
    all_orders = []
    for day_offset in range(7):
        date = (datetime(2026, 6, 19) - timedelta(days=day_offset)).strftime('%Y-%m-%d')
        key = f'processed/orders/dt={date}/orders.parquet'
        df = lake.read_parquet(key)
        all_orders.append(df)
    
    combined = pd.concat(all_orders)
    daily_summary = combined.groupby('order_date').agg(
        total_orders=('order_id', 'count'),
        total_revenue=('amount', 'sum'),
        avg_order=('amount', 'mean')
    ).reset_index()
    
    for _, row in daily_summary.iterrows():
        date = row['order_date']
        summary_df = pd.DataFrame([row])
        lake.upload_analytics(summary_df, 'daily_summary', date)
    
    lake.get_inventory()
```

---

### 🏋️ Full Day Exercises (20 questions)

1. Enable versioning on your S3 bucket. Upload the same file twice with different content. List both versions and download the older one.

2. Create a lifecycle policy that moves `raw/` files to Glacier after 60 days and deletes `temp/` files after 7 days.

3. Verify your bucket has default encryption enabled. Upload a file and check its encryption status.

4. Block all public access on your bucket. Verify by trying to access a file via its public URL.

5. Generate a presigned URL for a file in your bucket. Open it in a browser — does it work? Wait 5 minutes and try with `ExpiresIn=10` — does it expire?

6. **Data lake design:** Design the S3 folder structure for a company with 3 data sources (orders, customers, products) across 3 zones (raw, processed, analytics). Draw the full tree.

7. Implement the `MakanExpressDataLake` class from Concept 10. Run it and verify all files are uploaded.

8. Write a function that reads ALL Parquet files across multiple date partitions and combines them into a single DataFrame.

9. Create a lifecycle policy that keeps `analytics/` data indefinitely but archives `processed/` after 180 days.

10. Write a script that calculates the total size of each zone in your data lake (raw/, processed/, analytics/) and prints a cost estimate (S3 Standard: ~$0.025/GB/month in Singapore).

11. **Partitioning exercise:** Generate 30 days of fake order data and upload each day as a separate Parquet partition. How many files did you create? What's the average file size?

12. Write a function that deletes all files for a specific date (e.g., to reprocess a day):
```python
def delete_partition(bucket, table_name, date):
    """Delete all files for a specific date partition."""
    ...
```

13. Upload a 10MB CSV file. Measure the time to upload vs download. Now convert to Parquet and compare sizes.

14. Write a Python script that syncs your local `food-delivery-warehouse` project's seed data to S3 in the correct data lake structure.

15. **Error handling:** Write a robust upload function that:
    - Retries on failure (3 attempts)
    - Validates the file exists before uploading
    - Logs each upload attempt
    - Returns success/failure status

16. Investigate S3 Select: can you query a CSV file in S3 without downloading it? Write a Python script using `select_object_content`.

17. Create a bucket policy that allows only your IAM user to access the bucket. Verify by checking the policy.

18. Write a script that generates a daily data lake report:
    - Total files per zone
    - Total size per zone
    - Oldest and newest file
    - Files added in the last 24 hours

19. **Real-world scenario:** Your ETL pipeline processes 50 CSV files per day (each ~5MB). Design the lifecycle: how much data after 30 days? 90 days? 1 year? What's the monthly cost?

20. **Challenge:** Build a complete S3 data lake management tool:
    - `setup` command: create bucket, configure security, lifecycle, encryption
    - `upload` command: upload a CSV/Parquet file to the correct zone and partition
    - `download` command: download files by table name and date range
    - `list` command: list all files, optionally filtered by zone/table/date
    - `delete` command: delete a specific date partition
    - `report` command: show size, file count, and cost estimate by zone
    - `convert` command: convert all CSV in raw/ to Parquet in processed/

---

### ✅ Selected Answers

<details>
<summary>🔑 Click to reveal selected answers</summary>

**Answer 1:**
```bash
# Enable versioning
aws s3api put-bucket-versioning --bucket makanexpress-data-lake-dev \
    --versioning-configuration Status=Enabled

# Upload v1
echo "version 1" | aws s3 cp - s3://makanexpress-data-lake-dev/raw/test/data.csv

# Upload v2
echo "version 2" | aws s3 cp - s3://makanexpress-data-lake-dev/raw/test/data.csv

# List versions
aws s3api list-object-versions --bucket makanexpress-data-lake-dev --prefix raw/test/data.csv
# Note the VersionId for each

# Download v1 (replace VERSION_ID)
aws s3api get-object --bucket makanexpress-data-lake-dev \
    --key raw/test/data.csv --version-id VERSION_ID /tmp/data_v1.csv
cat /tmp/data_v1.csv  # should show "version 1"
```

**Answer 8:**
```python
def read_all_partitions(bucket, table_name, start_date, end_date):
    import pandas as pd
    import boto3
    from io import BytesIO
    
    s3 = boto3.client('s3')
    dfs = []
    
    current = pd.to_datetime(start_date)
    end = pd.to_datetime(end_date)
    
    while current <= end:
        date_str = current.strftime('%Y-%m-%d')
        key = f'processed/{table_name}/dt={date_str}/{table_name}.parquet'
        
        try:
            response = s3.get_object(Bucket=bucket, Key=key)
            df = pd.read_parquet(BytesIO(response['Body'].read()))
            dfs.append(df)
        except s3.exceptions.NoSuchKey:
            print(f"  ⚠️ No data for {date_str}")
        
        current += timedelta(days=1)
    
    return pd.concat(dfs, ignore_index=True) if dfs else pd.DataFrame()
```

**Answer 12:**
```python
def delete_partition(bucket, table_name, date, zone='processed'):
    s3 = boto3.client('s3')
    prefix = f'{zone}/{table_name}/dt={date}/'
    
    # List all objects with this prefix
    response = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
    
    if 'Contents' not in response:
        print(f"No files found at {prefix}")
        return
    
    # Delete each object
    for obj in response['Contents']:
        s3.delete_object(Bucket=bucket, Key=obj['Key'])
        print(f"  Deleted: {obj['Key']}")
    
    print(f"✅ Deleted {len(response['Contents'])} files for dt={date}")
```

**Answer 15:**
```python
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def robust_upload(df, bucket, key, max_retries=3):
    from io import BytesIO, StringIO
    import boto3
    
    s3 = boto3.client('s3')
    
    for attempt in range(1, max_retries + 1):
        try:
            if key.endswith('.parquet'):
                buffer = BytesIO()
                df.to_parquet(buffer, index=False)
                body = buffer.getvalue()
            elif key.endswith('.csv'):
                buffer = StringIO()
                df.to_csv(buffer, index=False)
                body = buffer.getvalue()
            else:
                raise ValueError(f"Unsupported format: {key}")
            
            s3.put_object(Bucket=bucket, Key=key, Body=body)
            logger.info(f"✅ Uploaded {len(df)} rows to s3://{bucket}/{key}")
            return True
            
        except Exception as e:
            logger.warning(f"⚠️ Attempt {attempt}/{max_retries} failed: {e}")
            if attempt < max_retries:
                time.sleep(2 ** attempt)  # exponential backoff
    
    logger.error(f"❌ Failed to upload after {max_retries} attempts: {key}")
    return False
```

**Answer 19:**
```
Daily: 50 files × 5MB = 250MB/day

After 30 days (Standard):
  Raw: 7.5 GB → $0.19/month
  Processed (Parquet, ~1.25GB): $0.03/month

After 90 days (Standard for 30d, IA for 60d):
  Raw: 22.5 GB → ~$0.75/month
  Processed: ~3.75 GB → ~$0.15/month

After 1 year (Standard 30d, IA 60d, Glacier 275d):
  Raw: 91.25 GB → ~$1.50/month
  Processed: ~15 GB → ~$0.50/month

Total monthly cost after 1 year: ~$2-3/month
(Very affordable for a data lake!)
```

</details>

---

## 🌙 Evening Block (1 hour): Review

### S3 Decision Tree

```
Need to store files?
├── How often accessed?
│   ├── Frequently (daily) → S3 Standard
│   ├── Sometimes (monthly) → Standard-IA
│   └── Rarely (archive) → Glacier
├── What format?
│   ├── Raw/landing → CSV or JSON (debuggable)
│   └── Processed/analytics → Parquet (compressed, fast)
├── How to organize?
│   ├── By date partition → raw/table/dt=YYYY-MM-DD/
│   └── By category → raw/table/category=X/dt=YYYY-MM-DD/
└── Security?
    ├── Block public access → ALWAYS
    ├── Encryption → SSE-S3 (default, free)
    └── Access → IAM policies or bucket policies
```

### 📝 Today's Checklist

- [ ] S3 versioning enabled and tested
- [ ] Lifecycle policies configured for cost optimization
- [ ] Default encryption enabled
- [ ] Public access blocked
- [ ] Understand the 3-zone data lake pattern (raw/processed/analytics)
- [ ] Can partition data by date when uploading
- [ ] Understand CSV vs Parquet tradeoffs
- [ ] Built a data lake management utility class
- [ ] Ready for tomorrow: IAM — Security & Access Control

### 📖 Further Reading (Free)

- [S3 Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/best-practices.html)
- [S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
- [Parquet Format Documentation](https://parquet.apache.org/documentation/latest/)

---

*Day 37 complete! 2-day checkpoint done. Next up: Day 38-40 (IAM, RDS, VPC).* 🔐
