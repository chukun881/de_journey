# 📅 Day 42 — Thursday, 25 June 2026
# 🏗️ PORTFOLIO PROJECT: Deploy MakanExpress Data Warehouse on AWS

---

## 🎯 Today's Goal

Deploy the **MakanExpress Food Delivery Warehouse** (built locally in Weeks 4-5) onto AWS. By end of day, you'll have:

- ✅ An S3 data lake with 3 zones (raw / processed / analytics)
- ✅ An RDS PostgreSQL instance with your star schema
- ✅ IAM roles and policies following least privilege
- ✅ Python ETL scripts (boto3) that automate the pipeline
- ✅ Complete documentation in your GitHub repo

This is your **most impressive portfolio project** — it shows employers you can deploy real infrastructure on AWS, not just run things locally.

> **⏱️ Time budget:** Morning (2h) + Afternoon (2h) + Evening (1h) = 5 hours
> **💰 Cost:** $0 if you're within the AWS Free Tier (12-month). Clean up after if needed.

---

## 📋 Project Overview

### What You're Building

```
food-delivery-warehouse/
├── README.md                        ← UPDATE with AWS deployment section
├── schema/                          ← existing (Day 28)
├── seed/                            ← existing CSV data files
├── etl/                             ← existing local ETL
├── analytics/                       ← existing SQL queries
├── quality/                         ← existing data quality checks
├── docs/                            ← existing documentation
├── dags/                            ← existing Airflow DAGs (Day 35)
├── tests/                           ← existing DAG tests
├── aws/                             ← NEW: AWS deployment
│   ├── README.md                    ← AWS setup guide
│   ├── architecture.md              ← architecture documentation
│   ├── upload_to_s3.py              ← upload CSVs to S3
│   ├── load_to_rds.py               ← load S3 data into RDS
│   ├── etl_pipeline.py              ← full S3 → RDS ETL pipeline
│   ├── create_infrastructure.sh     ← CLI commands to set up AWS
│   ├── teardown.sh                  ← cleanup (delete resources)
│   └── iam/
│       ├── etl-role-trust.json      ← trust policy for ETL role
│       ├── etl-role-permissions.json← permissions for ETL role
│       └── data-lake-policy.json    ← S3 bucket access policy
├── sql/
│   ├── 01_aws_schema.sql            ← combined schema for RDS
│   ├── 02_aws_load_raw.sql          ← COPY commands for raw data
│   └── 03_aws_load_star.sql         ← SCD + fact table loading
└── requirements.txt                 ← UPDATE with boto3
```

### Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                     AWS (ap-southeast-1 — Singapore)                  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  S3: makanexpress-datalake-[your-initials]                     │ │
│  │                                                                 │ │
│  │  raw/makanexpress/          ← Source CSVs (customers, orders…) │ │
│  │  ├── customers.csv                                             │ │
│  │  ├── restaurants.csv                                           │ │
│  │  ├── menu_items.csv                                            │ │
│  │  ├── orders.csv                                                │ │
│  │  └── order_items.csv                                           │ │
│  │                                                                 │ │
│  │  processed/makanexpress/   ← Cleaned Parquet (future: Glue)   │ │
│  │  analytics/makanexpress/   ← Aggregated reports (future)      │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│       ↑ boto3 upload              ↑ psycopg2 load                    │
│       │                            │                                  │
│  ┌────┴────────────────────────────┴──────────────────────────────┐ │
│  │  RDS PostgreSQL: makanexpress-db                               │ │
│  │  • db.t3.micro (Free Tier)                                     │ │
│  │  • 20 GB storage                                               │ │
│  │  • Schema: makanexpress                                        │ │
│  │  • Tables: raw_* + dim_* + fact_orders                         │ │
│  │  • Views: vw_daily_revenue, etc.                               │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  IAM                                                            │ │
│  │  • Role: MakanExpressETLRole                                    │ │
│  │  • Policy: MakanExpressDataLakePolicy (scoped to bucket)       │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Security                                                       │ │
│  │  • Security Group: Allow TCP 5432 from your IP only            │ │
│  │  • RDS: Publicly Accessible = Yes (dev only!)                  │ │
│  │  • S3: Server-side encryption enabled                          │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘

Your Laptop (Local Machine):
  ┌─────────────────────────────────────────────────┐
  │  Python + boto3 → Upload CSVs to S3             │
  │  Python + psycopg2 → Load data from S3 to RDS   │
  │  psql → Run schema migrations on RDS            │
  └─────────────────────────────────────────────────┘
```

---

## ☀️ Morning Block (2 hours): Set Up AWS Infrastructure

### Step 1: Create the S3 Data Lake (25 min)

S3 bucket names must be **globally unique** across all AWS accounts. We'll add your initials to make it unique.

```bash
# Set variables (change to your initials!)
export MY_INITIALS="xxx"  # ← CHANGE THIS (e.g., "jlo", "abc")
export BUCKET_NAME="makanexpress-datalake-${MY_INITIALS}"
export REGION="ap-southeast-1"

echo "Bucket name: $BUCKET_NAME"
echo "Region: $REGION"
```

#### 1a. Create the S3 bucket

```bash
# Create bucket in Singapore region
aws s3 mb "s3://${BUCKET_NAME}" \
    --region ap-southeast-1 \
    --create-bucket-configuration LocationConstraint=ap-southeast-1

# Verify
aws s3 ls | grep makanexpress
```

> **💡 Why `LocationConstraint`?** For regions other than `us-east-1`, you MUST specify the location constraint. If you get an error, double-check the bucket name is unique.

<details>
<summary>🔑 Troubleshooting: "Bucket already exists"</summary>

S3 bucket names are globally unique. If someone else took your name, try:
```bash
# Add a random suffix
export BUCKET_NAME="makanexpress-datalake-${MY_INITIALS}-$(date +%s)"
aws s3 mb "s3://${BUCKET_NAME}" --region ap-southeast-1 \
    --create-bucket-configuration LocationConstraint=ap-southeast-1
```
Or add your birth month, favourite number, etc. Just make it unique.
</details>

#### 1b. Create the folder structure

S3 doesn't have real folders — but we create zero-byte objects with trailing `/` to simulate them:

```bash
# Create zone prefixes by uploading placeholder files
echo "placeholder" > /tmp/.keep

aws s3 cp /tmp/.keep "s3://${BUCKET_NAME}/raw/.keep"
aws s3 cp /tmp/.keep "s3://${BUCKET_NAME}/processed/.keep"
aws s3 cp /tmp/.keep "s3://${BUCKET_NAME}/analytics/.keep"

# Verify structure
aws s3 ls "s3://${BUCKET_NAME}/" --recursive
```

#### 1c. Enable versioning and encryption

```bash
# Enable versioning (protects against accidental deletes)
aws s3api put-bucket-versioning \
    --bucket "${BUCKET_NAME}" \
    --versioning-configuration Status=Enabled

# Enable server-side encryption (SSE-S3)
aws s3api put-bucket-encryption \
    --bucket "${BUCKET_NAME}" \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "AES256"
            },
            "BucketKeyEnabled": false
        }]
    }'

# Block public access (security best practice)
aws s3api put-public-access-block \
    --bucket "${BUCKET_NAME}" \
    --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

echo "✅ S3 bucket configured: ${BUCKET_NAME}"
```

#### 1d. Set up lifecycle policy (cost management)

```bash
cat > /tmp/lifecycle.json << 'EOF'
{
    "Rules": [
        {
            "ID": "RawDataLifecycle",
            "Status": "Enabled",
            "Filter": {"Prefix": "raw/"},
            "Transitions": [
                {
                    "Days": 90,
                    "StorageClass": "STANDARD_IA"
                },
                {
                    "Days": 365,
                    "StorageClass": "GLACIER"
                }
            ]
        },
        {
            "ID": "ProcessedDataLifecycle",
            "Status": "Enabled",
            "Filter": {"Prefix": "processed/"},
            "Transitions": [
                {
                    "Days": 180,
                    "StorageClass": "STANDARD_IA"
                }
            ]
        },
        {
            "ID": "AnalyticsDataRetention",
            "Status": "Enabled",
            "Filter": {"Prefix": "analytics/"},
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "STANDARD_IA"
                }
            ]
        }
    ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
    --bucket "${BUCKET_NAME}" \
    --lifecycle-configuration file:///tmp/lifecycle.json

echo "✅ Lifecycle policies set"
```

> **💡 Why different policies per zone?** Raw data is rarely accessed after initial processing → cheaper storage sooner. Analytics data is accessed frequently → stays on Standard longer. This shows employers you understand **cost optimization**, a key skill.

---

### Step 2: Launch RDS PostgreSQL (30 min)

#### 2a. Get your public IP (for security group)

```bash
# Get your current public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "My IP: ${MY_IP}/32"

# Or use a service like
curl -s https://ifconfig.me
```

#### 2b. Create a security group for RDS

```bash
# Get your default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
    --filters Name=isDefault,Values=true \
    --query 'Vpcs[0].VpcId' \
    --output text \
    --region ap-southeast-1)
echo "VPC: ${VPC_ID}"

# Create security group
SG_ID=$(aws ec2 create-security-group \
    --group-name makanexpress-rds-sg \
    --description "Allow PostgreSQL access for MakanExpress RDS" \
    --vpc-id "${VPC_ID}" \
    --region ap-southeast-1 \
    --query 'GroupId' \
    --output text)
echo "Security Group: ${SG_ID}"

# Allow PostgreSQL (port 5432) from YOUR IP only
aws ec2 authorize-security-group-ingress \
    --group-id "${SG_ID}" \
    --protocol tcp \
    --port 5432 \
    --cidr "${MY_IP}/32" \
    --region ap-southeast-1

echo "✅ Security group created and rule added"
```

> **⚠️ Security note:** For this portfolio project, we allow your IP directly. In production, you'd use a bastion host or VPN — never direct internet access to a database.

#### 2c. Create a DB subnet group

```bash
# Get subnet IDs from your default VPC
SUBNET_IDS=$(aws ec2 describe-subnets \
    --filters Name=vpc-id,Values=${VPC_ID} \
    --query 'Subnets[*].SubnetId' \
    --output text \
    --region ap-southeast-1)
echo "Subnets: ${SUBNET_IDS}"

# Create DB subnet group (RDS needs at least 2 subnets in different AZs)
# Pick the first two subnets
SUBNET_1=$(echo $SUBNET_IDS | awk '{print $1}')
SUBNET_2=$(echo $SUBNET_IDS | awk '{print $2}')

aws rds create-db-subnet-group \
    --db-subnet-group-name makanexpress-subnet-group \
    --db-subnet-group-description "Subnet group for MakanExpress RDS" \
    --subnet-ids ${SUBNET_1} ${SUBNET_2} \
    --region ap-southeast-1

echo "✅ DB subnet group created"
```

#### 2d. Launch the RDS instance

```bash
# ⚠️ Choose a STRONG password — this is a database on the internet!
DB_PASSWORD="M@kan3xpress!2026"  # Change this to your own strong password

aws rds create-db-instance \
    --db-instance-identifier makanexpress-db \
    --db-instance-class db.t3.micro \
    --engine postgres \
    --engine-version 16.4 \
    --master-username makanexpress_admin \
    --master-user-password "${DB_PASSWORD}" \
    --allocated-storage 20 \
    --storage-type gp3 \
    --db-name makanexpress \
    --vpc-security-group-ids "${SG_ID}" \
    --db-subnet-group-name makanexpress-subnet-group \
    --publicly-accessible \
    --region ap-southeast-1 \
    --tags Key=Project,Value=MakanExpress Key=ManagedBy,Value=DataEngineer-Portfolio

echo "✅ RDS instance creation initiated"
echo "⏳ This takes 5-10 minutes. Let's move on while it creates."
```

> **💡 Why `db.t3.micro`?** Free Tier eligible (750 hours/month for 12 months). Enough for a portfolio project. In production, you'd use a larger instance with Multi-AZ.

#### 2e. Monitor RDS creation

```bash
# Check status (run this every 2 minutes until it says "available")
aws rds describe-db-instances \
    --db-instance-identifier makanexpress-db \
    --query 'DBInstances[0].[DBInstanceStatus, Endpoint.Address, Endpoint.Port]' \
    --output table \
    --region ap-southeast-1

# When status is "available", save the endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier makanexpress-db \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text \
    --region ap-southeast-1)

echo "RDS Endpoint: ${RDS_ENDPOINT}"

# Save it for later use
echo "export RDS_ENDPOINT=${RDS_ENDPOINT}" >> ~/.bashrc
echo "export RDS_USER=makanexpress_admin" >> ~/.bashrc
echo "export RDS_DB=makanexpress" >> ~/.bashrc
source ~/.bashrc
```

---

### Step 3: Set Up IAM Roles and Policies (30 min)

#### 3a. Create the ETL role trust policy

```bash
cd ~/projects/food-delivery-warehouse
mkdir -p aws/iam

cat > aws/iam/etl-role-trust.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Service": [
                "glue.amazonaws.com",
                "lambda.amazonaws.com",
                "ec2.amazonaws.com"
            ]
        },
        "Action": "sts:AssumeRole",
        "Condition": {
            "StringEquals": {
                "aws:SourceAccount": "ACCOUNT_ID_PLACEHOLDER"
            }
        }
    }]
}
EOF

# Replace placeholder with your actual account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
sed -i "s/ACCOUNT_ID_PLACEHOLDER/${ACCOUNT_ID}/" aws/iam/etl-role-trust.json

echo "✅ Trust policy created"
```

#### 3b. Create the ETL role permissions policy

```bash
# Replace BUCKET_NAME in the policy
cat > aws/iam/etl-role-permissions.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowReadRawData",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET_NAME}",
                "arn:aws:s3:::${BUCKET_NAME}/raw/*"
            ]
        },
        {
            "Sid": "AllowWriteProcessedData",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET_NAME}",
                "arn:aws:s3:::${BUCKET_NAME}/processed/*"
            ]
        },
        {
            "Sid": "AllowWriteAnalytics",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET_NAME}",
                "arn:aws:s3:::${BUCKET_NAME}/analytics/*"
            ]
        },
        {
            "Sid": "AllowRDSAccess",
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBInstances",
                "rds-data:ExecuteStatement",
                "rds-data:BatchExecuteStatement",
                "rds-data:BeginTransaction",
                "rds-data:CommitTransaction",
                "rds-data:RollbackTransaction"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DenyDelete",
            "Effect": "Deny",
            "Action": [
                "s3:DeleteObject",
                "s3:DeleteBucket",
                "rds:DeleteDBInstance"
            ],
            "Resource": "*"
        }
    ]
}
EOF

echo "✅ Permissions policy created"
```

#### 3c. Create the IAM role

```bash
# Create the role
aws iam create-role \
    --role-name MakanExpressETLRole \
    --assume-role-policy-document file://aws/iam/etl-role-trust.json \
    --description "ETL role for MakanExpress data pipeline"

# Attach the permissions policy
aws iam put-role-policy \
    --role-name MakanExpressETLRole \
    --policy-name MakanExpressETLPermissions \
    --policy-document file://aws/iam/etl-role-permissions.json

# Attach AWS managed Glue service role (for future Glue use)
aws iam attach-role-policy \
    --role-name MakanExpressETLRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

# Verify
aws iam get-role --role-name MakanExpressETLRole \
    --query 'Role.[RoleName, Arn]' --output text

echo "✅ IAM role created: MakanExpressETLRole"
```

#### 3d. Create a bucket policy for audit logging (optional but impressive)

```bash
cat > /tmp/bucket-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUnencryptedObjectUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::${BUCKET_NAME}/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "AES256"
                }
            }
        },
        {
            "Sid": "DenyIncorrectEncryptionHeader",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::${BUCKET_NAME}/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption-aws-kms-key-id": ""
                }
            }
        }
    ]
}
EOF

aws s3api put-bucket-policy \
    --bucket "${BUCKET_NAME}" \
    --policy file:///tmp/bucket-policy.json

echo "✅ Bucket policy set: Encryption enforced"
```

> **💡 Why this matters for your portfolio:** This bucket policy enforces **encryption at rest** for all uploaded objects. When an interviewer asks "how do you secure data in S3?", mentioning encryption-in-transit (HTTPS) AND encryption-at-rest (SSE) + bucket policies shows you understand data security.

---

### Step 4: Wait for RDS + Test Connection (15 min)

```bash
# Check if RDS is ready
STATUS=$(aws rds describe-db-instances \
    --db-instance-identifier makanexpress-db \
    --query 'DBInstances[0].DBInstanceStatus' \
    --output text \
    --region ap-southeast-1)

echo "RDS Status: ${STATUS}"

if [ "$STATUS" = "available" ]; then
    RDS_ENDPOINT=$(aws rds describe-db-instances \
        --db-instance-identifier makanexpress-db \
        --query 'DBInstances[0].Endpoint.Address' \
        --output text \
        --region ap-southeast-1)
    
    echo "✅ RDS is ready!"
    echo "Endpoint: ${RDS_ENDPOINT}"
    
    # Test connection (enter your password when prompted)
    PGPASSWORD="${DB_PASSWORD}" psql \
        -h "${RDS_ENDPOINT}" \
        -U makanexpress_admin \
        -d makanexpress \
        -c "SELECT version();"
else
    echo "⏳ RDS not ready yet. Status: ${STATUS}"
    echo "Check again in 2 minutes."
fi
```

<details>
<summary>🔑 Troubleshooting RDS Connection</summary>

**"Connection timed out":**
- Security group doesn't include your current IP
- Fix: Get your IP again (`curl https://checkip.amazonaws.com`) and add it:
  ```bash
  NEW_IP=$(curl -s https://checkip.amazonaws.com)
  aws ec2 authorize-security-group-ingress \
      --group-id "${SG_ID}" \
      --protocol tcp --port 5432 \
      --cidr "${NEW_IP}/32"
  ```

**"password authentication failed":**
- Wrong password. Reset it:
  ```bash
  aws rds modify-db-instance \
      --db-instance-identifier makanexpress-db \
      --master-user-password 'NewStr0ng!Pass' \
      --apply-immediately
  ```

**"Connection refused":**
- Instance still starting. Wait 5-10 minutes total.

**"database 'makanexpress' does not exist":**
- You forgot `--db-name makanexpress` when creating. Connect to `postgres` and create it:
  ```bash
  PGPASSWORD="${DB_PASSWORD}" psql -h "${RDS_ENDPOINT}" -U makanexpress_admin -d postgres \
      -c "CREATE DATABASE makanexpress;"
  ```
</details>

---

## 🌤️ Afternoon Block (2 hours): Migrate Data

### Step 5: Upload CSVs to S3 (30 min)

#### 5a. Update the upload script

You created the skeleton yesterday. Now let's finalize it:

```bash
cd ~/projects/food-delivery-warehouse

cat > aws/upload_to_s3.py << 'EOPY'
#!/usr/bin/env python3
"""
MakanExpress: Upload seed data to S3 data lake

Usage:
    python aws/upload_to_s3.py
    python aws/upload_to_s3.py --bucket my-bucket --prefix raw/makanexpress/

Environment variables (or use defaults):
    S3_BUCKET    - Target S3 bucket name
    S3_PREFIX    - Target prefix (folder) in the bucket
"""

import boto3
import os
import sys
import hashlib
from pathlib import Path
from datetime import datetime


class S3Uploader:
    """Handles uploading MakanExpress seed data to S3."""
    
    def __init__(self, bucket=None, prefix=None, region='ap-southeast-1'):
        self.bucket = bucket or os.environ.get(
            'S3_BUCKET', 'makanexpress-datalake'
        )
        self.prefix = prefix or os.environ.get(
            'S3_PREFIX', 'raw/makanexpress/'
        )
        self.region = region
        self.s3 = boto3.client('s3', region_name=region)
        self.upload_log = []
    
    def compute_md5(self, file_path):
        """Compute MD5 hash of a file for integrity checking."""
        md5 = hashlib.md5()
        with open(file_path, 'rb') as f:
            for chunk in iter(lambda: f.read(8192), b''):
                md5.update(chunk)
        return md5.hexdigest()
    
    def bucket_exists(self):
        """Check if the target S3 bucket exists."""
        try:
            self.s3.head_bucket(Bucket=self.bucket)
            return True
        except Exception:
            return False
    
    def upload_file(self, local_path, s3_key):
        """
        Upload a single file to S3 with metadata and integrity check.
        
        Returns:
            dict: Upload result with status, key, size, md5
        """
        file_size = os.path.getsize(local_path)
        md5_hash = self.compute_md5(local_path)
        
        try:
            self.s3.upload_file(
                local_path,
                self.bucket,
                s3_key,
                ExtraArgs={
                    'ContentType': 'text/csv',
                    'Metadata': {
                        'uploaded-by': 'makanexpress-etl',
                        'upload-timestamp': datetime.utcnow().isoformat(),
                        'source-md5': md5_hash,
                        'file-size-bytes': str(file_size),
                    }
                }
            )
            
            result = {
                'file': os.path.basename(local_path),
                'key': s3_key,
                'size_bytes': file_size,
                'md5': md5_hash,
                'status': 'uploaded'
            }
            self.upload_log.append(result)
            return result
            
        except Exception as e:
            result = {
                'file': os.path.basename(local_path),
                'key': s3_key,
                'status': f'failed: {str(e)}'
            }
            self.upload_log.append(result)
            return result
    
    def upload_seed_data(self, seed_dir='seed'):
        """
        Upload all CSV files from seed directory to S3.
        
        Returns:
            int: Number of successfully uploaded files
        """
        print(f"\n{'='*60}")
        print(f"  MakanExpress S3 Upload")
        print(f"{'='*60}")
        print(f"  Bucket:  s3://{self.bucket}")
        print(f"  Prefix:  {self.prefix}")
        print(f"  Region:  {self.region}")
        print(f"{'='*60}\n")
        
        # Verify bucket
        if not self.bucket_exists():
            print(f"❌ ERROR: Bucket '{self.bucket}' does not exist!")
            print(f"   Create it first: aws s3 mb s3://{self.bucket}")
            return 0
        
        # Find CSV files
        seed_path = Path(seed_dir)
        if not seed_path.exists():
            print(f"❌ ERROR: Seed directory '{seed_dir}' not found!")
            return 0
        
        csv_files = sorted(seed_path.glob('*.csv'))
        if not csv_files:
            print(f"❌ ERROR: No CSV files in '{seed_dir}/'")
            return 0
        
        print(f"📂 Found {len(csv_files)} CSV files to upload:\n")
        
        success = 0
        for csv_file in csv_files:
            s3_key = f"{self.prefix}{csv_file.name}"
            result = self.upload_file(str(csv_file), s3_key)
            
            if result['status'] == 'uploaded':
                size_kb = result['size_bytes'] / 1024
                print(f"  ✅ {csv_file.name:20s} → {s3_key} ({size_kb:.1f} KB)")
                success += 1
            else:
                print(f"  ❌ {csv_file.name:20s} → {result['status']}")
        
        # Upload manifest
        import json
        manifest = {
            'project': 'MakanExpress Food Delivery Warehouse',
            'upload_timestamp': datetime.utcnow().isoformat(),
            'bucket': self.bucket,
            'prefix': self.prefix,
            'files': {r['file']: {'key': r['key'], 'md5': r.get('md5', 'N/A')}
                      for r in self.upload_log if r['status'] == 'uploaded'},
            'total_files': len(csv_files),
            'successful_uploads': success
        }
        
        manifest_key = f"{self.prefix}manifest.json"
        self.s3.put_object(
            Bucket=self.bucket,
            Key=manifest_key,
            Body=json.dumps(manifest, indent=2),
            ContentType='application/json'
        )
        print(f"\n  📋 Manifest uploaded: {manifest_key}")
        
        print(f"\n{'='*60}")
        print(f"  Result: {success}/{len(csv_files)} files uploaded ✅")
        print(f"{'='*60}\n")
        
        return success
    
    def list_uploaded_files(self):
        """List all files in the upload prefix."""
        response = self.s3.list_objects_v2(
            Bucket=self.bucket,
            Prefix=self.prefix
        )
        
        if 'Contents' not in response:
            print("No files found.")
            return []
        
        print(f"\nFiles in s3://{self.bucket}/{self.prefix}:")
        print("-" * 70)
        for obj in response['Contents']:
            size_kb = obj['Size'] / 1024
            print(f"  {obj['Key']:50s} {size_kb:8.1f} KB")
        print("-" * 70)
        print(f"  Total: {len(response['Contents'])} objects")
        
        return response['Contents']


def main():
    import argparse
    
    parser = argparse.ArgumentParser(
        description='Upload MakanExpress seed data to S3'
    )
    parser.add_argument(
        '--bucket', 
        default=os.environ.get('S3_BUCKET'),
        help='S3 bucket name (default: from S3_BUCKET env var)'
    )
    parser.add_argument(
        '--prefix',
        default=os.environ.get('S3_PREFIX', 'raw/makanexpress/'),
        help='S3 prefix (default: raw/makanexpress/)'
    )
    parser.add_argument(
        '--seed-dir',
        default='seed',
        help='Local seed directory (default: seed/)'
    )
    parser.add_argument(
        '--list-only',
        action='store_true',
        help='List existing files in S3 without uploading'
    )
    args = parser.parse_args()
    
    uploader = S3Uploader(
        bucket=args.bucket,
        prefix=args.prefix
    )
    
    if args.list_only:
        uploader.list_uploaded_files()
    else:
        success = uploader.upload_seed_data(args.seed_dir)
        if success == 0:
            sys.exit(1)


if __name__ == '__main__':
    main()
EOPY

chmod +x aws/upload_to_s3.py
echo "✅ upload_to_s3.py finalized"
```

#### 5b. Run the upload

```bash
cd ~/projects/food-delivery-warehouse

# Set the bucket name
export S3_BUCKET="${BUCKET_NAME}"

# Run the upload
python3 aws/upload_to_s3.py \
    --bucket "${BUCKET_NAME}" \
    --prefix "raw/makanexpress/"

# Verify uploads
python3 aws/upload_to_s3.py --bucket "${BUCKET_NAME}" --list-only

# Or use AWS CLI to verify
aws s3 ls "s3://${BUCKET_NAME}/raw/makanexpress/" --human-readable
```

---

### Step 6: Migrate Schema to RDS (20 min)

#### 6a. Run the schema migration

```bash
cd ~/projects/food-delivery-warehouse

# Get the RDS endpoint (if not already set)
RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier makanexpress-db \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text \
    --region ap-southeast-1)

echo "Connecting to: ${RDS_ENDPOINT}"

# Run the schema migration (you created this file yesterday on Day 41)
PGPASSWORD="${DB_PASSWORD}" psql \
    -h "${RDS_ENDPOINT}" \
    -U makanexpress_admin \
    -d makanexpress \
    -f sql/01_aws_schema.sql

# Verify tables were created
PGPASSWORD="${DB_PASSWORD}" psql \
    -h "${RDS_ENDPOINT}" \
    -U makanexpress_admin \
    -d makanexpress \
    -c "\dt makanexpress.*"
```

**Expected output:**
```
                     List of relations
   Schema    |         Name          | Type  |    Owner     
-------------+-----------------------+-------+--------------
 makanexpress| dim_customer          | table | makanexpress_admin
 makanexpress| dim_date              | table | makanexpress_admin
 makanexpress| dim_menu_item         | table | makanexpress_admin
 makanexpress| dim_restaurant        | table | makanexpress_admin
 makanexpress| fact_orders           | table | makanexpress_admin
 makanexpress| raw_customers         | table | makanexpress_admin
 makanexpress| raw_order_items       | table | makanexpress_admin
 makanexpress| raw_orders            | table | makanexpress_admin
 makanexpress| raw_menu_items        | table | makanexpress_admin
 makanexpress| raw_restaurants       | table | makanexpress_admin
```

<details>
<summary>🔑 Troubleshooting Schema Migration</summary>

**"schema 'makanexpress' already exists":**
- You ran it twice. That's fine — the `DROP SCHEMA IF EXISTS ... CASCADE` at the top handles this. All data is reset.

**"permission denied for schema makanexpress":**
- You connected as a different user. Make sure to use `-U makanexpress_admin`.

**"connection refused" or "timeout":**
- RDS might still be creating. Check status:
  ```bash
  aws rds describe-db-instances --db-instance-identifier makanexpress-db \
      --query 'DBInstances[0].DBInstanceStatus' --output text
  ```
  Wait until it shows `available`.
</details>

---

### Step 7: Load Data from S3 to RDS (40 min)

#### 7a. Create the data loading script

```bash
cd ~/projects/food-delivery-warehouse

cat > aws/load_to_rds.py << 'EOPY'
#!/usr/bin/env python3
"""
MakanExpress: Load seed data from S3 into RDS PostgreSQL

Two approaches:
1. Direct COPY from local CSV files (for initial load)
2. Download from S3, then COPY into RDS (production pattern)

Usage:
    python aws/load_to_rds.py --from-local
    python aws/load_to_rds.py --from-s3
"""

import boto3
import psycopg2
import os
import sys
import csv
import tempfile
from pathlib import Path
from io import StringIO


class RDSLoader:
    """Load MakanExpress data into RDS PostgreSQL."""
    
    def __init__(self, host, database, user, password, port=5432):
        self.conn_params = {
            'host': host,
            'database': database,
            'user': user,
            'password': password,
            'port': port,
            'connect_timeout': 10
        }
        self.conn = None
    
    def connect(self):
        """Establish connection to RDS."""
        try:
            self.conn = psycopg2.connect(**self.conn_params)
            self.conn.autocommit = True
            print(f"✅ Connected to RDS: {self.conn_params['host']}")
            return True
        except psycopg2.OperationalError as e:
            print(f"❌ Connection failed: {e}")
            return False
    
    def close(self):
        """Close the database connection."""
        if self.conn:
            self.conn.close()
            print("🔌 Connection closed.")
    
    def load_csv_to_table(self, csv_path, table_name, schema='makanexpress'):
        """
        Load a CSV file into a PostgreSQL table using COPY.
        
        Uses psycopg2's copy_expert for efficient bulk loading.
        """
        if not Path(csv_path).exists():
            print(f"  ❌ File not found: {csv_path}")
            return False
        
        full_table = f"{schema}.{table_name}"
        
        try:
            with open(csv_path, 'r') as f:
                # Read first line to check header
                header = f.readline().strip().lower()
                f.seek(0)
                
                # Use COPY command with CSV format
                copy_sql = f"""
                    COPY {full_table} 
                    FROM STDIN 
                    WITH (FORMAT CSV, HEADER TRUE, DELIMITER ',', NULL '')
                """
                
                cursor = self.conn.cursor()
                cursor.copy_expert(copy_sql, f)
                
                # Verify row count
                cursor.execute(f"SELECT COUNT(*) FROM {full_table}")
                count = cursor.fetchone()[0]
                
                print(f"  ✅ {csv_path.name:20s} → {full_table:40s} ({count:,} rows)")
                return True
                
        except Exception as e:
            print(f"  ❌ {csv_path.name} → {full_table}: {e}")
            return False
    
    def load_from_local(self, seed_dir='seed'):
        """Load all CSV files from local seed directory."""
        print(f"\n📦 Loading data from LOCAL files...")
        print("-" * 70)
        
        # Mapping: CSV filename → RDS table name
        file_table_map = {
            'customers.csv': 'raw_customers',
            'restaurants.csv': 'raw_restaurants',
            'menu_items.csv': 'raw_menu_items',
            'orders.csv': 'raw_orders',
            'order_items.csv': 'raw_order_items',
        }
        
        seed_path = Path(seed_dir)
        success = 0
        
        for csv_file, table_name in file_table_map.items():
            file_path = seed_path / csv_file
            if self.load_csv_to_table(file_path, table_name):
                success += 1
        
        print("-" * 70)
        print(f"  Result: {success}/{len(file_table_map)} tables loaded\n")
        return success
    
    def load_from_s3(self, bucket, prefix='raw/makanexpress/'):
        """Download CSVs from S3, then load into RDS."""
        print(f"\n☁️ Loading data from S3: s3://{bucket}/{prefix}")
        print("-" * 70)
        
        s3 = boto3.client('s3', region_name='ap-southeast-1')
        
        # List CSV files in S3
        response = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
        if 'Contents' not in response:
            print("  ❌ No files found in S3!")
            return 0
        
        csv_files = [
            obj for obj in response['Contents']
            if obj['Key'].endswith('.csv')
        ]
        
        print(f"  Found {len(csv_files)} CSV files in S3")
        
        # Mapping: S3 filename → RDS table
        file_table_map = {
            'customers.csv': 'raw_customers',
            'restaurants.csv': 'raw_restaurants',
            'menu_items.csv': 'raw_menu_items',
            'orders.csv': 'raw_orders',
            'order_items.csv': 'raw_order_items',
        }
        
        success = 0
        with tempfile.TemporaryDirectory() as tmpdir:
            for obj in csv_files:
                filename = Path(obj['Key']).name
                if filename not in file_table_map:
                    continue
                
                # Download from S3
                local_path = Path(tmpdir) / filename
                s3.download_file(bucket, obj['Key'], str(local_path))
                print(f"  ⬇️  Downloaded: {filename}")
                
                # Load into RDS
                table_name = file_table_map[filename]
                if self.load_csv_to_table(local_path, table_name):
                    success += 1
        
        print("-" * 70)
        print(f"  Result: {success}/{len(csv_files)} files loaded\n")
        return success
    
    def populate_star_schema(self):
        """
        Populate dimension and fact tables from raw staging tables.
        
        This runs the SCD Type 2 logic and fact table loading.
        """
        print(f"\n⭐ Populating star schema tables...")
        print("-" * 70)
        
        cursor = self.conn.cursor()
        
        # 1. Populate dim_date from order dates
        cursor.execute("""
            INSERT INTO makanexpress.dim_date (date_key, day_of_week, day_name, 
                month, month_name, quarter, year, is_weekend)
            SELECT DISTINCT 
                order_date::date,
                EXTRACT(ISODOW FROM order_date::date)::int,
                TO_CHAR(order_date::date, 'Day'),
                EXTRACT(MONTH FROM order_date::date)::int,
                TO_CHAR(order_date::date, 'Month'),
                EXTRACT(QUARTER FROM order_date::date)::int,
                EXTRACT(YEAR FROM order_date::date)::int,
                EXTRACT(ISODOW FROM order_date::date) IN (6, 7)
            FROM makanexpress.raw_orders
            ON CONFLICT (date_key) DO NOTHING
        """)
        cursor.execute("SELECT COUNT(*) FROM makanexpress.dim_date")
        print(f"  ✅ dim_date:        {cursor.fetchone()[0]:,} rows")
        
        # 2. Populate dim_customer (SCD Type 1 for initial load — no changes yet)
        cursor.execute("""
            INSERT INTO makanexpress.dim_customer 
                (customer_id, customer_name, email, phone, city, valid_from, is_current)
            SELECT customer_id, name, email, phone, city, 
                   MIN(registered_date)::date, TRUE
            FROM makanexpress.raw_customers
            GROUP BY customer_id, name, email, phone, city
            ON CONFLICT (customer_id, valid_from) DO NOTHING
        """)
        cursor.execute("SELECT COUNT(*) FROM makanexpress.dim_customer")
        print(f"  ✅ dim_customer:    {cursor.fetchone()[0]:,} rows")
        
        # 3. Populate dim_restaurant
        cursor.execute("""
            INSERT INTO makanexpress.dim_restaurant
                (restaurant_id, restaurant_name, cuisine, city, rating, is_active, valid_from, is_current)
            SELECT restaurant_id, name, cuisine, city, rating, is_active, CURRENT_DATE, TRUE
            FROM makanexpress.raw_restaurants
            ON CONFLICT (restaurant_id, valid_from) DO NOTHING
        """)
        cursor.execute("SELECT COUNT(*) FROM makanexpress.dim_restaurant")
        print(f"  ✅ dim_restaurant:  {cursor.fetchone()[0]:,} rows")
        
        # 4. Populate dim_menu_item
        cursor.execute("""
            INSERT INTO makanexpress.dim_menu_item
                (item_id, restaurant_id, item_name, category, price, is_available)
            SELECT item_id, restaurant_id, item_name, category, price, is_available
            FROM makanexpress.raw_menu_items
        """)
        cursor.execute("SELECT COUNT(*) FROM makanexpress.dim_menu_item")
        print(f"  ✅ dim_menu_item:   {cursor.fetchone()[0]:,} rows")
        
        # 5. Populate fact_orders (join raw tables with dimension tables)
        cursor.execute("""
            INSERT INTO makanexpress.fact_orders 
                (order_id, customer_sk, restaurant_sk, date_sk, item_sk,
                 quantity, unit_price, subtotal, total_amount, status, delivery_city)
            SELECT 
                o.order_id,
                dc.customer_sk,
                dr.restaurant_sk,
                dd.date_sk,
                dm.item_sk,
                oi.quantity,
                oi.unit_price,
                oi.subtotal,
                o.total_amount,
                o.status,
                o.delivery_city
            FROM makanexpress.raw_orders o
            JOIN makanexpress.raw_order_items oi ON o.order_id = oi.order_id
            JOIN makanexpress.dim_customer dc 
                ON o.customer_id = dc.customer_id AND dc.is_current = TRUE
            JOIN makanexpress.dim_restaurant dr 
                ON o.restaurant_id = dr.restaurant_id AND dr.is_current = TRUE
            JOIN makanexpress.dim_date dd 
                ON o.order_date::date = dd.date_key
            JOIN makanexpress.dim_menu_item dm 
                ON oi.item_id = dm.item_id
        """)
        cursor.execute("SELECT COUNT(*) FROM makanexpress.fact_orders")
        print(f"  ✅ fact_orders:     {cursor.fetchone()[0]:,} rows")
        
        print("-" * 70)
        print(f"  ⭐ Star schema populated!\n")
    
    def run_analytics_queries(self):
        """Run sample analytics queries to verify the data warehouse."""
        print(f"\n📊 Running sample analytics queries...")
        print("-" * 70)
        
        cursor = self.conn.cursor()
        
        # Query 1: Revenue by city
        print("\n  📈 Revenue by City (Top 5):")
        cursor.execute("""
            SELECT delivery_city, 
                   COUNT(DISTINCT order_id) as orders,
                   SUM(total_amount) as revenue
            FROM makanexpress.fact_orders
            GROUP BY delivery_city
            ORDER BY revenue DESC
            LIMIT 5
        """)
        for row in cursor.fetchall():
            print(f"     {row[0]:15s}  {row[1]:5d} orders  ${row[2]:>10,.2f}")
        
        # Query 2: Top cuisines
        print("\n  🍜 Orders by Cuisine:")
        cursor.execute("""
            SELECT r.cuisine, 
                   COUNT(DISTINCT f.order_id) as orders,
                   SUM(f.subtotal) as revenue
            FROM makanexpress.fact_orders f
            JOIN makanexpress.dim_restaurant r ON f.restaurant_sk = r.restaurant_sk
            GROUP BY r.cuisine
            ORDER BY orders DESC
            LIMIT 5
        """)
        for row in cursor.fetchall():
            print(f"     {row[0]:15s}  {row[1]:5d} orders  ${row[2]:>10,.2f}")
        
        # Query 3: Daily revenue view
        print("\n  📅 Daily Revenue (using vw_daily_revenue):")
        cursor.execute("""
            SELECT date_key, total_orders, total_revenue
            FROM makanexpress.vw_daily_revenue
            ORDER BY date_key
            LIMIT 5
        """)
        for row in cursor.fetchall():
            print(f"     {row[0]}  {row[1]:3d} orders  ${row[2]:>10,.2f}")
        
        print("-" * 70)


def main():
    import argparse
    
    parser = argparse.ArgumentParser(
        description='Load MakanExpress data into RDS PostgreSQL'
    )
    parser.add_argument(
        '--from-local', action='store_true',
        help='Load from local seed/ directory'
    )
    parser.add_argument(
        '--from-s3', action='store_true',
        help='Download from S3, then load into RDS'
    )
    parser.add_argument(
        '--populate-star', action='store_true',
        help='Populate star schema tables from raw data'
    )
    parser.add_argument(
        '--analytics', action='store_true',
        help='Run sample analytics queries'
    )
    parser.add_argument('--host', default=os.environ.get('RDS_ENDPOINT'))
    parser.add_argument('--database', default=os.environ.get('RDS_DB', 'makanexpress'))
    parser.add_argument('--user', default=os.environ.get('RDS_USER', 'makanexpress_admin'))
    parser.add_argument('--password', default=os.environ.get('DB_PASSWORD'))
    parser.add_argument('--bucket', default=os.environ.get('S3_BUCKET'))
    parser.add_argument('--prefix', default='raw/makanexpress/')
    parser.add_argument('--seed-dir', default='seed')
    args = parser.parse_args()
    
    if not args.host:
        print("❌ RDS endpoint required. Set RDS_ENDPOINT env var or use --host")
        sys.exit(1)
    
    loader = RDSLoader(
        host=args.host,
        database=args.database,
        user=args.user,
        password=args.password
    )
    
    if not loader.connect():
        sys.exit(1)
    
    try:
        if args.from_local:
            loader.load_from_local(args.seed_dir)
        
        if args.from_s3:
            if not args.bucket:
                print("❌ S3 bucket required. Set S3_BUCKET env var or use --bucket")
                sys.exit(1)
            loader.load_from_s3(args.bucket, args.prefix)
        
        if args.populate_star:
            loader.populate_star_schema()
        
        if args.analytics:
            loader.run_analytics_queries()
        
        # If no flags, run everything
        if not any([args.from_local, args.from_s3, args.populate_star, args.analytics]):
            print("\n🚀 Running full pipeline: local load → star schema → analytics\n")
            loader.load_from_local(args.seed_dir)
            loader.populate_star_schema()
            loader.run_analytics_queries()
    
    finally:
        loader.close()


if __name__ == '__main__':
    main()
EOPY

chmod +x aws/load_to_rds.py
echo "✅ load_to_rds.py created"
```

#### 7b. Run the data load

```bash
cd ~/projects/food-delivery-warehouse

# Set environment variables
export RDS_ENDPOINT="${RDS_ENDPOINT}"
export RDS_USER="makanexpress_admin"
export RDS_DB="makanexpress"
export DB_PASSWORD="${DB_PASSWORD}"

# Run the full pipeline (local load → star schema → analytics)
python3 aws/load_to_rds.py --from-local --populate-star --analytics
```

**Expected output:**
```
🚀 Running full pipeline: local load → star schema → analytics

📦 Loading data from LOCAL files...
----------------------------------------------------------------------
  ✅ customers.csv       → makanexpress.raw_customers               (500 rows)
  ✅ restaurants.csv     → makanexpress.raw_restaurants              (50 rows)
  ✅ menu_items.csv      → makanexpress.raw_menu_items               (200 rows)
  ✅ orders.csv          → makanexpress.raw_orders                   (5,000 rows)
  ✅ order_items.csv     → makanexpress.raw_order_items              (15,000 rows)
----------------------------------------------------------------------
  Result: 5/5 tables loaded

⭐ Populating star schema tables...
----------------------------------------------------------------------
  ✅ dim_date:        365 rows
  ✅ dim_customer:    500 rows
  ✅ dim_restaurant:  50 rows
  ✅ dim_menu_item:   200 rows
  ✅ fact_orders:     15,000 rows
----------------------------------------------------------------------
  ⭐ Star schema populated!

📊 Running sample analytics queries...
----------------------------------------------------------------------
  📈 Revenue by City (Top 5):
       Singapore       3200 orders  $  128,000.00
       Kuala Lumpur    2500 orders  $   97,500.00
       Penang          1100 orders  $   42,000.00
       Johor Bahru      800 orders  $   31,200.00
       George Town      400 orders  $   15,600.00
----------------------------------------------------------------------
```

> **🎉 If you see the analytics output with real numbers — congratulations!** Your MakanExpress warehouse is fully deployed on AWS RDS!

---

### Step 8: Load from S3 to RDS (Production Pattern) (20 min)

Now let's test the **S3 → RDS** path (this is what a real production pipeline would do):

```bash
cd ~/projects/food-delivery-warehouse

# First, truncate the raw tables (simulate a fresh load)
PGPASSWORD="${DB_PASSWORD}" psql \
    -h "${RDS_ENDPOINT}" \
    -U makanexpress_admin \
    -d makanexpress \
    -c "TRUNCATE makanexpress.raw_order_items, makanexpress.raw_orders, 
        makanexpress.raw_menu_items, makanexpress.raw_restaurants, 
        makanexpress.raw_customers CASCADE;"

# Now load FROM S3 (download from S3 → load to RDS)
export S3_BUCKET="${BUCKET_NAME}"

python3 aws/load_to_rds.py --from-s3 --populate-star --analytics \
    --bucket "${BUCKET_NAME}" \
    --prefix "raw/makanexpress/"
```

This tests the **production pattern**: S3 (data lake) → Python ETL → RDS (warehouse).

<details>
<summary>🔑 What if the S3 load fails?</summary>

Common issues:
1. **S3 bucket name wrong:** `echo $BUCKET_NAME` and verify with `aws s3 ls`
2. **Files not in S3:** `aws s3 ls s3://${BUCKET_NAME}/raw/makanexpress/`
3. **boto3 credentials:** `aws sts get-caller-identity` — should show your IAM user
4. **RDS timeout:** Check security group allows your current IP

</details>

---

## 🌙 Evening Block (1 hour): Automation + Documentation + Push

### Step 9: Create the Full ETL Pipeline Script (20 min)

This script combines everything into a single pipeline:

```bash
cd ~/projects/food-delivery-warehouse

cat > aws/etl_pipeline.py << 'EOPY'
#!/usr/bin/env python3
"""
MakanExpress: Full ETL Pipeline — S3 to RDS

This script demonstrates a complete data pipeline:
1. Extract: Download CSV files from S3 data lake
2. Transform: Validate and clean data using Pandas
3. Load: Insert into RDS PostgreSQL star schema

Usage:
    python aws/etl_pipeline.py --env dev
    
Environment variables:
    S3_BUCKET       - Source S3 bucket
    RDS_ENDPOINT    - RDS PostgreSQL endpoint
    RDS_USER        - Database username
    RDS_DB          - Database name
    DB_PASSWORD     - Database password
"""

import boto3
import psycopg2
import pandas as pd
import os
import sys
import tempfile
import logging
from pathlib import Path
from datetime import datetime


# ─── Logging Setup ─────────────────────────────────────────────────────────

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logger = logging.getLogger('makanexpress-etl')


# ─── Extract: Download from S3 ─────────────────────────────────────────────

def extract_from_s3(bucket, prefix, local_dir):
    """
    Download CSV files from S3 data lake.
    
    Args:
        bucket: S3 bucket name
        prefix: S3 prefix (folder path)
        local_dir: Local directory to download to
    
    Returns:
        dict: {filename: local_path}
    """
    s3 = boto3.client('s3', region_name='ap-southeast-1')
    
    response = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
    if 'Contents' not in response:
        logger.error("No files found in s3://%s/%s", bucket, prefix)
        return {}
    
    csv_files = {}
    for obj in response['Contents']:
        if obj['Key'].endswith('.csv'):
            filename = Path(obj['Key']).name
            local_path = Path(local_dir) / filename
            s3.download_file(bucket, obj['Key'], str(local_path))
            csv_files[filename] = str(local_path)
            logger.info("Downloaded: %s (%.1f KB)", filename, obj['Size']/1024)
    
    return csv_files


# ─── Transform: Validate & Clean ────────────────────────────────────────────

def validate_and_clean(csv_files):
    """
    Validate data quality and clean CSV files.
    
    Returns:
        dict: {table_name: clean DataFrame}
    """
    table_map = {
        'customers.csv': 'raw_customers',
        'restaurants.csv': 'raw_restaurants',
        'menu_items.csv': 'raw_menu_items',
        'orders.csv': 'raw_orders',
        'order_items.csv': 'raw_order_items',
    }
    
    clean_data = {}
    
    for filename, filepath in csv_files.items():
        if filename not in table_map:
            continue
        
        table_name = table_map[filename]
        df = pd.read_csv(filepath)
        
        # Basic validation
        row_count = len(df)
        null_counts = df.isnull().sum()
        
        # Log any data quality issues
        if null_counts.any():
            for col, count in null_counts[null_counts > 0].items():
                logger.warning(
                    "%s.%s has %d null values (%.1f%%)",
                    table_name, col, count, count/row_count*100
                )
        
        # Clean: fill numeric nulls with 0, string nulls with 'Unknown'
        for col in df.columns:
            if df[col].dtype in ['float64', 'int64']:
                df[col] = df[col].fillna(0)
            else:
                df[col] = df[col].fillna('Unknown')
        
        # Remove exact duplicates
        before = len(df)
        df = df.drop_duplicates()
        after = len(df)
        if before != after:
            logger.warning(
                "%s: Removed %d duplicate rows",
                table_name, before - after
            )
        
        clean_data[table_name] = df
        logger.info(
            "Validated %s: %d rows, %d columns",
            table_name, len(df), len(df.columns)
        )
    
    return clean_data


# ─── Load: Insert into RDS ──────────────────────────────────────────────────

def load_to_rds(clean_data, conn_params):
    """
    Load validated DataFrames into RDS PostgreSQL.
    
    Uses TRUNCATE + COPY for idempotent loads.
    """
    conn = psycopg2.connect(**conn_params)
    conn.autocommit = False
    cursor = conn.cursor()
    
    try:
        # Truncate all raw tables first (idempotent)
        raw_tables = [t for t in clean_data.keys() if t.startswith('raw_')]
        if raw_tables:
            truncate_sql = f"TRUNCATE {', '.join(f'makanexpress.{t}' for t in raw_tables)} CASCADE"
            cursor.execute(truncate_sql)
            logger.info("Truncated %d raw tables", len(raw_tables))
        
        # Load each DataFrame
        for table_name, df in clean_data.items():
            full_table = f"makanexpress.{table_name}"
            
            # Use StringIO + copy_expert for fast bulk insert
            buffer = StringIO()
            df.to_csv(buffer, index=False, header=True)
            buffer.seek(0)
            
            copy_sql = f"""
                COPY {full_table} 
                FROM STDIN 
                WITH (FORMAT CSV, HEADER TRUE, DELIMITER ',', NULL '')
            """
            cursor.copy_expert(copy_sql, buffer)
            logger.info("Loaded %d rows into %s", len(df), full_table)
        
        conn.commit()
        logger.info("✅ All raw tables loaded successfully")
        
    except Exception as e:
        conn.rollback()
        logger.error("❌ Load failed, rolled back: %s", e)
        raise
    finally:
        cursor.close()
        conn.close()


def populate_star_schema(conn_params):
    """Populate dimension and fact tables from raw staging data."""
    conn = psycopg2.connect(**conn_params)
    conn.autocommit = True
    cursor = conn.cursor()
    
    try:
        # Truncate star schema tables (reverse dependency order)
        cursor.execute("""
            TRUNCATE makanexpress.fact_orders CASCADE;
            TRUNCATE makanexpress.dim_menu_item CASCADE;
            TRUNCATE makanexpress.dim_date CASCADE;
            TRUNCATE makanexpress.dim_restaurant CASCADE;
            TRUNCATE makanexpress.dim_customer CASCADE;
        """)
        
        # dim_date
        cursor.execute("""
            INSERT INTO makanexpress.dim_date 
                (date_key, day_of_week, day_name, month, month_name, 
                 quarter, year, is_weekend)
            SELECT DISTINCT 
                order_date::date,
                EXTRACT(ISODOW FROM order_date::date)::int,
                TRIM(TO_CHAR(order_date::date, 'Day')),
                EXTRACT(MONTH FROM order_date::date)::int,
                TRIM(TO_CHAR(order_date::date, 'Month')),
                EXTRACT(QUARTER FROM order_date::date)::int,
                EXTRACT(YEAR FROM order_date::date)::int,
                EXTRACT(ISODOW FROM order_date::date) IN (6, 7)
            FROM makanexpress.raw_orders
            ON CONFLICT (date_key) DO NOTHING
        """)
        logger.info("Populated dim_date")
        
        # dim_customer
        cursor.execute("""
            INSERT INTO makanexpress.dim_customer 
                (customer_id, customer_name, email, phone, city, valid_from, is_current)
            SELECT customer_id, name, email, phone, city,
                   MIN(registered_date)::date, TRUE
            FROM makanexpress.raw_customers
            GROUP BY customer_id, name, email, phone, city
            ON CONFLICT (customer_id, valid_from) DO NOTHING
        """)
        logger.info("Populated dim_customer")
        
        # dim_restaurant
        cursor.execute("""
            INSERT INTO makanexpress.dim_restaurant
                (restaurant_id, restaurant_name, cuisine, city, rating, 
                 is_active, valid_from, is_current)
            SELECT restaurant_id, name, cuisine, city, rating, is_active,
                   CURRENT_DATE, TRUE
            FROM makanexpress.raw_restaurants
            ON CONFLICT (restaurant_id, valid_from) DO NOTHING
        """)
        logger.info("Populated dim_restaurant")
        
        # dim_menu_item
        cursor.execute("""
            INSERT INTO makanexpress.dim_menu_item
                (item_id, restaurant_id, item_name, category, price, is_available)
            SELECT item_id, restaurant_id, item_name, category, price, is_available
            FROM makanexpress.raw_menu_items
        """)
        logger.info("Populated dim_menu_item")
        
        # fact_orders
        cursor.execute("""
            INSERT INTO makanexpress.fact_orders 
                (order_id, customer_sk, restaurant_sk, date_sk, item_sk,
                 quantity, unit_price, subtotal, total_amount, status, delivery_city)
            SELECT 
                o.order_id,
                dc.customer_sk,
                dr.restaurant_sk,
                dd.date_sk,
                dm.item_sk,
                oi.quantity,
                oi.unit_price,
                oi.subtotal,
                o.total_amount,
                o.status,
                o.delivery_city
            FROM makanexpress.raw_orders o
            JOIN makanexpress.raw_order_items oi ON o.order_id = oi.order_id
            JOIN makanexpress.dim_customer dc 
                ON o.customer_id = dc.customer_id AND dc.is_current = TRUE
            JOIN makanexpress.dim_restaurant dr 
                ON o.restaurant_id = dr.restaurant_id AND dr.is_current = TRUE
            JOIN makanexpress.dim_date dd 
                ON o.order_date::date = dd.date_key
            JOIN makanexpress.dim_menu_item dm 
                ON oi.item_id = dm.item_id
        """)
        logger.info("Populated fact_orders")
        
        logger.info("✅ Star schema populated successfully")
        
    finally:
        cursor.close()
        conn.close()


# ─── Main Pipeline ──────────────────────────────────────────────────────────

def run_pipeline(env='dev'):
    """
    Run the complete MakanExpress ETL pipeline.
    
    Pipeline: S3 → Download → Validate → Load Raw → Populate Star Schema
    """
    start_time = datetime.now()
    logger.info("=" * 60)
    logger.info("  MakanExpress ETL Pipeline — %s", env.upper())
    logger.info("  Started: %s", start_time.isoformat())
    logger.info("=" * 60)
    
    # Configuration
    conn_params = {
        'host': os.environ.get('RDS_ENDPOINT'),
        'database': os.environ.get('RDS_DB', 'makanexpress'),
        'user': os.environ.get('RDS_USER', 'makanexpress_admin'),
        'password': os.environ.get('DB_PASSWORD'),
        'port': 5432,
        'connect_timeout': 10
    }
    bucket = os.environ.get('S3_BUCKET')
    prefix = os.environ.get('S3_PREFIX', 'raw/makanexpress/')
    
    if not conn_params['host']:
        logger.error("RDS_ENDPOINT not set")
        sys.exit(1)
    if not bucket:
        logger.error("S3_BUCKET not set")
        sys.exit(1)
    
    # Step 1: Extract
    logger.info("STEP 1: Extract — Downloading from S3...")
    with tempfile.TemporaryDirectory() as tmpdir:
        csv_files = extract_from_s3(bucket, prefix, tmpdir)
        if not csv_files:
            logger.error("No CSV files found. Pipeline aborted.")
            sys.exit(1)
        
        # Step 2: Transform
        logger.info("STEP 2: Transform — Validating and cleaning data...")
        clean_data = validate_and_clean(csv_files)
        
        # Step 3: Load raw
        logger.info("STEP 3: Load — Inserting into RDS raw tables...")
        load_to_rds(clean_data, conn_params)
    
    # Step 4: Populate star schema
    logger.info("STEP 4: Transform — Populating star schema...")
    populate_star_schema(conn_params)
    
    # Done
    elapsed = (datetime.now() - start_time).total_seconds()
    logger.info("=" * 60)
    logger.info("  ✅ Pipeline completed in %.1f seconds", elapsed)
    logger.info("=" * 60)


if __name__ == '__main__':
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument('--env', default='dev', choices=['dev', 'prod'])
    args = parser.parse_args()
    run_pipeline(env=args.env)
EOPY

chmod +x aws/etl_pipeline.py
echo "✅ etl_pipeline.py created"
```

#### Test the full pipeline

```bash
cd ~/projects/food-delivery-warehouse

# Run the full pipeline: S3 → RDS
python3 aws/etl_pipeline.py --env dev
```

---

### Step 10: Create the Infrastructure Script (10 min)

Document all CLI commands so you (or an employer reviewing your repo) can reproduce the setup:

```bash
cd ~/projects/food-delivery-warehouse

cat > aws/create_infrastructure.sh << 'EOSH'
#!/bin/bash
# ============================================================
# MakanExpress AWS Infrastructure Setup
# ============================================================
# This script creates all AWS resources for the portfolio project.
# All resources are within AWS Free Tier (12-month).
#
# Prerequisites:
#   - AWS CLI configured (aws configure)
#   - jq installed
#   - curl available
#
# Usage:
#   chmod +x aws/create_infrastructure.sh
#   ./aws/create_infrastructure.sh
#
# Cost: $0 within Free Tier
# Region: ap-southeast-1 (Singapore)
# ============================================================

set -e  # Exit on error

# ─── Configuration ──────────────────────────────────────────────
REGION="ap-southeast-1"
INITIALS="${1:-dev}"  # Pass your initials as first argument
BUCKET_NAME="makanexpress-datalake-${INITIALS}"
DB_INSTANCE="makanexpress-db"
DB_CLASS="db.t3.micro"
DB_ENGINE="postgres"
DB_VERSION="16.4"
DB_USER="makanexpress_admin"
DB_PASSWORD="${2:-changeme}"  # Pass password as second argument
DB_NAME="makanexpress"
SG_NAME="makanexpress-rds-sg"

echo "============================================================"
echo "  MakanExpress Infrastructure Setup"
echo "============================================================"
echo "  Region:   ${REGION}"
echo "  Bucket:   ${BUCKET_NAME}"
echo "  Database: ${DB_INSTANCE}"
echo "============================================================"

# ─── 1. S3 Bucket ───────────────────────────────────────────────
echo ""
echo "📦 Creating S3 bucket..."
if aws s3 ls "s3://${BUCKET_NAME}" 2>/dev/null; then
    echo "  Bucket already exists: ${BUCKET_NAME}"
else
    aws s3 mb "s3://${BUCKET_NAME}" \
        --region ${REGION} \
        --create-bucket-configuration LocationConstraint=${REGION}
    echo "  ✅ Created: ${BUCKET_NAME}"
fi

# Create folder structure
echo "placeholder" > /tmp/.keep
aws s3 cp /tmp/.keep "s3://${BUCKET_NAME}/raw/.keep" 2>/dev/null
aws s3 cp /tmp/.keep "s3://${BUCKET_NAME}/processed/.keep" 2>/dev/null
aws s3 cp /tmp/.keep "s3://${BUCKET_NAME}/analytics/.keep" 2>/dev/null
echo "  ✅ Folder structure created (raw/, processed/, analytics/)"

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket "${BUCKET_NAME}" \
    --versioning-configuration Status=Enabled 2>/dev/null || true
echo "  ✅ Versioning enabled"

# Block public access
aws s3api put-public-access-block \
    --bucket "${BUCKET_NAME}" \
    --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true \
    2>/dev/null || true
echo "  ✅ Public access blocked"

# ─── 2. Security Group ─────────────────────────────────────────
echo ""
echo "🔒 Creating security group..."
VPC_ID=$(aws ec2 describe-vpcs \
    --filters Name=isDefault,Values=true \
    --query 'Vpcs[0].VpcId' --output text --region ${REGION})

SG_ID=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=${SG_NAME} Name=vpc-id,Values=${VPC_ID} \
    --query 'SecurityGroups[0].GroupId' --output text --region ${REGION} 2>/dev/null || echo "")

if [ -z "$SG_ID" ] || [ "$SG_ID" = "None" ]; then
    SG_ID=$(aws ec2 create-security-group \
        --group-name ${SG_NAME} \
        --description "PostgreSQL access for MakanExpress RDS" \
        --vpc-id ${VPC_ID} \
        --region ${REGION} \
        --query 'GroupId' --output text)
    echo "  ✅ Created security group: ${SG_ID}"
else
    echo "  Security group already exists: ${SG_ID}"
fi

MY_IP=$(curl -s https://checkip.amazonaws.com)/32
aws ec2 authorize-security-group-ingress \
    --group-id ${SG_ID} \
    --protocol tcp --port 5432 \
    --cidr "${MY_IP}" \
    --region ${REGION} 2>/dev/null || echo "  Rule already exists"
echo "  ✅ PostgreSQL access allowed from ${MY_IP}"

# ─── 3. RDS Instance ───────────────────────────────────────────
echo ""
echo "🗄️  Creating RDS PostgreSQL instance..."
EXISTING=$(aws rds describe-db-instances \
    --db-instance-identifier ${DB_INSTANCE} \
    --query 'DBInstances[0].DBInstanceStatus' --output text \
    --region ${REGION} 2>/dev/null || echo "")

if [ -n "$EXISTING" ]; then
    echo "  RDS instance already exists. Status: ${EXISTING}"
else
    # Get subnet IDs
    SUBNET_IDS=$(aws ec2 describe-subnets \
        --filters Name=vpc-id,Values=${VPC_ID} \
        --query 'Subnets[*].SubnetId' --output text --region ${REGION})
    SUBNET_1=$(echo ${SUBNET_IDS} | awk '{print $1}')
    SUBNET_2=$(echo ${SUBNET_IDS} | awk '{print $2}')
    
    # Create DB subnet group
    aws rds create-db-subnet-group \
        --db-subnet-group-name makanexpress-subnet-group \
        --db-subnet-group-description "MakanExpress RDS subnet group" \
        --subnet-ids ${SUBNET_1} ${SUBNET_2} \
        --region ${REGION} 2>/dev/null || true
    
    # Create RDS instance
    aws rds create-db-instance \
        --db-instance-identifier ${DB_INSTANCE} \
        --db-instance-class ${DB_CLASS} \
        --engine ${DB_ENGINE} \
        --engine-version ${DB_VERSION} \
        --master-username ${DB_USER} \
        --master-user-password "${DB_PASSWORD}" \
        --allocated-storage 20 \
        --storage-type gp3 \
        --db-name ${DB_NAME} \
        --vpc-security-group-ids ${SG_ID} \
        --db-subnet-group-name makanexpress-subnet-group \
        --publicly-accessible \
        --region ${REGION} \
        --tags Key=Project,Value=MakanExpress Key=Environment,Value=Portfolio
    
    echo "  ✅ RDS instance creation initiated"
    echo "  ⏳ Wait 5-10 minutes for it to become available"
fi

# ─── Summary ────────────────────────────────────────────────────
echo ""
echo "============================================================"
echo "  Infrastructure Summary"
echo "============================================================"
echo "  S3 Bucket:     s3://${BUCKET_NAME}"
echo "  Security Group: ${SG_ID}"
echo "  RDS Instance:  ${DB_INSTANCE}"
echo "  RDS Endpoint:  (check with: aws rds describe-db-instances)"
echo ""
echo "  Next steps:"
echo "  1. Wait for RDS to become 'available'"
echo "  2. python3 aws/upload_to_s3.py --bucket ${BUCKET_NAME}"
echo "  3. python3 aws/load_to_rds.py --from-s3 --populate-star"
echo "============================================================"
EOSH

chmod +x aws/create_infrastructure.sh

# Also create a teardown script
cat > aws/teardown.sh << 'EOSH'
#!/bin/bash
# ============================================================
# MakanExpress AWS Teardown
# ============================================================
# ⚠️ WARNING: This deletes all AWS resources created for the 
# portfolio project. Data will be lost!
#
# Usage:
#   ./aws/teardown.sh
# ============================================================

set -e

echo "⚠️  This will delete all MakanExpress AWS resources!"
read -p "Are you sure? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Cancelled."
    exit 0
fi

INITIALS="${1:-dev}"
BUCKET_NAME="makanexpress-datalake-${INITIALS}"
REGION="ap-southeast-1"

# Delete S3 bucket (must be empty first)
echo "🗑️  Deleting S3 bucket..."
aws s3 rm "s3://${BUCKET_NAME}" --recursive 2>/dev/null || true
aws s3 rb "s3://${BUCKET_NAME}" --force 2>/dev/null || true
echo "  ✅ S3 bucket deleted"

# Delete RDS instance
echo "🗑️  Deleting RDS instance..."
aws rds delete-db-instance \
    --db-instance-identifier makanexpress-db \
    --skip-final-snapshot \
    --delete-automated-backups \
    --region ${REGION} 2>/dev/null || echo "  Instance not found"
echo "  ✅ RDS deletion initiated (takes a few minutes)"

# Delete security group
echo "🗑️  Deleting security group..."
SG_ID=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=makanexpress-rds-sg \
    --query 'SecurityGroups[0].GroupId' --output text --region ${REGION} 2>/dev/null || echo "")
if [ -n "$SG_ID" ] && [ "$SG_ID" != "None" ]; then
    aws ec2 delete-security-group --group-id ${SG_ID} --region ${REGION} 2>/dev/null || true
    echo "  ✅ Security group deleted"
fi

# Delete IAM role
echo "🗑️  Deleting IAM role..."
aws iam detach-role-policy \
    --role-name MakanExpressETLRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole 2>/dev/null || true
aws iam delete-role-policy \
    --role-name MakanExpressETLRole \
    --policy-name MakanExpressETLPermissions 2>/dev/null || true
aws iam delete-role --role-name MakanExpressETLRole 2>/dev/null || true
echo "  ✅ IAM role deleted"

# Delete DB subnet group
aws rds delete-db-subnet-group \
    --db-subnet-group-name makanexpress-subnet-group \
    --region ${REGION} 2>/dev/null || true

echo ""
echo "✅ Teardown complete! All MakanExpress AWS resources deleted."
echo "⚠️  Remember to also remove any saved environment variables."
EOSH

chmod +x aws/teardown.sh
echo "✅ Infrastructure and teardown scripts created"
```

---

### Step 11: Write the AWS README (15 min)

```bash
cd ~/projects/food-delivery-warehouse

cat > aws/README.md << 'EOMD'
# ☁️ AWS Deployment — MakanExpress Data Warehouse

This directory contains the AWS deployment for the MakanExpress Food Delivery Data Warehouse.

## 📐 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  AWS (ap-southeast-1 — Singapore)                           │
│                                                              │
│  S3 Data Lake: makanexpress-datalake                        │
│  ├── raw/makanexpress/        ← Source CSV files           │
│  ├── processed/makanexpress/  ← Cleaned Parquet (future)   │
│  └── analytics/makanexpress/  ← Aggregated reports (future)│
│                                                              │
│  RDS PostgreSQL: makanexpress-db                            │
│  ├── Schema: makanexpress                                   │
│  ├── Raw staging tables (raw_*)                             │
│  ├── Star schema (dim_* + fact_orders)                      │
│  └── Analytics views (vw_daily_revenue, etc.)               │
│                                                              │
│  IAM: MakanExpressETLRole (S3 + RDS access)                │
└─────────────────────────────────────────────────────────────┘
```

## 🚀 Quick Start

### Prerequisites

- AWS CLI configured (`aws configure`)
- Python 3.8+ with: `boto3`, `psycopg2-binary`, `pandas`
- `psql` client (PostgreSQL)
- Active AWS account (Free Tier OK)

### 1. Create Infrastructure

```bash
# Replace 'xyz' with your initials, 'YourPassword123!' with a strong password
./aws/create_infrastructure.sh xyz 'YourStr0ng!Password123'

# Wait for RDS to become available (5-10 minutes)
aws rds describe-db-instances --db-instance-identifier makanexpress-db \
    --query 'DBInstances[0].DBInstanceStatus' --output text
```

### 2. Upload Data to S3

```bash
export S3_BUCKET=makanexpress-datalake-xyz
python3 aws/upload_to_s3.py --bucket $S3_BUCKET
```

### 3. Load Data into RDS

```bash
# Get the RDS endpoint
export RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier makanexpress-db \
    --query 'DBInstances[0].Endpoint.Address' --output text)

# Run schema migration
PGPASSWORD='YourPassword' psql -h $RDS_ENDPOINT -U makanexpress_admin \
    -d makanexpress -f sql/01_aws_schema.sql

# Load data from S3 → RDS
python3 aws/load_to_rds.py --from-s3 --populate-star --analytics
```

### 4. Run the Full ETL Pipeline

```bash
python3 aws/etl_pipeline.py --env dev
```

## 🧹 Cleanup (Important!)

To avoid charges after the Free Tier:

```bash
./aws/teardown.sh
```

## 📁 Files

| File | Purpose |
|------|---------|
| `create_infrastructure.sh` | Create all AWS resources (S3, RDS, IAM) |
| `upload_to_s3.py` | Upload CSV seed data to S3 data lake |
| `load_to_rds.py` | Load data from S3/local into RDS |
| `etl_pipeline.py` | Full ETL pipeline (extract → transform → load) |
| `teardown.sh` | Delete all AWS resources |
| `iam/` | IAM policy JSON files |

## 💰 Cost

All resources are within AWS Free Tier (12-month):

| Resource | Free Tier | Est. After Free Tier |
|----------|-----------|---------------------|
| S3 (2 GB) | $0/month | ~$0.05/month |
| RDS (db.t3.micro) | $0/month | ~$15/month |
| Data Transfer | $0 (first 1GB) | ~$0.40/month |

**⚠️ No NAT Gateway used** (would cost ~$36/month!)

## 🔒 Security

- S3: Public access blocked, server-side encryption enforced
- RDS: Access restricted to specific IP via security group
- IAM: Least-privilege policies for ETL role
- Credentials: Environment variables, not hardcoded

---

*Part of the [MakanExpress Food Delivery Warehouse](../) portfolio project.*
EOMD

echo "✅ aws/README.md created"
```

---

### Step 12: Update Main README + Push to GitHub (15 min)

#### Update the main project README

Add an **AWS Deployment** section to your existing README:

```bash
cd ~/projects/food-delivery-warehouse

# Add AWS section to existing README (append)
cat >> README.md << 'EOF'

## ☁️ AWS Deployment (Week 6)

This project is deployed on AWS using:

- **S3 Data Lake** (`makanexpress-datalake`) — 3-zone architecture (raw/processed/analytics)
- **RDS PostgreSQL** (`makanexpress-db`) — Managed database with star schema
- **IAM Roles** — Least-privilege access for ETL pipeline
- **Python ETL** (boto3 + psycopg2) — Automated S3 → RDS pipeline

### Architecture

```
S3 Data Lake (raw CSVs)
       │
       ▼
  Python ETL (boto3 + psycopg2)
       │
       ▼
  RDS PostgreSQL
  ├── Raw staging tables
  ├── Star schema (SCD Type 2)
  └── Analytics views
```

### Skills Demonstrated

- ✅ AWS S3 (bucket creation, lifecycle policies, encryption, versioning)
- ✅ AWS RDS (PostgreSQL instance, security groups, connectivity)
- ✅ AWS IAM (roles, policies, least-privilege access)
- ✅ Python boto3 (S3 operations, programmatic uploads)
- ✅ Python psycopg2 (database operations, bulk loading)
- ✅ Data pipeline automation (extract → transform → load)
- ✅ Security best practices (encryption, access control, no public data)
- ✅ Cost awareness (Free Tier optimization, no NAT Gateway)

See [`aws/README.md`](aws/README.md) for deployment instructions.

EOF

echo "✅ Main README updated"
```

#### Push everything to GitHub

```bash
cd ~/projects/food-delivery-warehouse

# Stage all new AWS files
git add aws/
git add sql/01_aws_schema.sql
git add README.md
git add requirements.txt

# Check what's staged
git status

# Commit
git commit -m "feat: deploy MakanExpress warehouse to AWS (S3 + RDS + IAM)

- S3 data lake with 3-zone architecture (raw/processed/analytics)
- RDS PostgreSQL with star schema and analytics views
- IAM role with least-privilege policies for ETL
- Python ETL pipeline: S3 → validate → load → star schema
- Infrastructure-as-code scripts (create/teardown)
- Security: encryption, IP-restricted access, no public data
- Cost: $0 within AWS Free Tier"

# Push
git push origin main
```

---

### Step 13: Portfolio Reflection (5 min)

#### 📝 What Employers Will See

When a hiring manager looks at your `food-delivery-warehouse` repo, they'll see:

```
food-delivery-warehouse/
├── 📊 Star Schema Design (Week 4)     → "Knows data modeling"
├── 🔄 SCD Type 2 Implementation (Week 4) → "Understands slowly changing dimensions"
├── 🌊 Airflow DAGs (Week 5)           → "Can orchestrate ETL pipelines"
├── ☁️ AWS Deployment (Week 6)         → "Can deploy to cloud"
├── 🐍 Python ETL Scripts (Week 6)     → "Can automate data pipelines"
├── 🔒 IAM + Security (Week 6)        → "Follows security best practices"
└── 📝 Complete Documentation          → "Communicates well"
```

This is a **strong portfolio project** because it shows the full stack:
1. **Design** → Star schema + SCD
2. **Build** → SQL + Python + dbt
3. **Orchestrate** → Airflow DAGs
4. **Deploy** → AWS (S3 + RDS + IAM)
5. **Automate** → Python ETL scripts
6. **Document** → README, architecture, security

#### 💬 How to Talk About This in Interviews

**When asked: "Tell me about a data project you've built"**

> "I built a food delivery data warehouse for a fictional company called MakanExpress — think GrabFood or foodpanda. I designed a star schema with fact and dimension tables, implemented SCD Type 2 for tracking customer and restaurant changes over time, built Airflow DAGs to orchestrate the daily ETL, and deployed the whole thing on AWS with S3 for the data lake and RDS PostgreSQL for the warehouse. The ETL pipeline is fully automated with Python — it downloads data from S3, validates it, loads it into staging tables, and populates the star schema. I set up IAM roles with least-privilege policies and encrypted everything."

**Key metrics to mention:**
- 5 raw tables → 4 dimensions + 1 fact table
- SCD Type 2 tracking for customers and restaurants
- Full S3 data lake with 3-zone architecture
- Automated ETL pipeline: S3 → validate → load → star schema
- Zero-cost deployment on AWS Free Tier

---

## ✅ Daily Checklist

- [ ] S3 bucket created with 3-zone structure (raw/processed/analytics)
- [ ] S3 versioning + encryption + lifecycle policies configured
- [ ] RDS PostgreSQL instance launched in Singapore (ap-southeast-1)
- [ ] Security group allows PostgreSQL from your IP only
- [ ] IAM role `MakanExpressETLRole` created with least-privilege policy
- [ ] CSV seed data uploaded to S3 (raw/makanexpress/)
- [ ] Schema migrated to RDS (01_aws_schema.sql)
- [ ] Raw data loaded from S3 into RDS staging tables
- [ ] Star schema populated (dim_* + fact_orders)
- [ ] Analytics views return correct results
- [ ] ETL pipeline script tested end-to-end
- [ ] Infrastructure creation script saved (create_infrastructure.sh)
- [ ] Teardown script saved (teardown.sh)
- [ ] AWS README written (aws/README.md)
- [ ] Main project README updated with AWS section
- [ ] All changes committed and pushed to GitHub
- [ ] AWS Billing Dashboard checked — no unexpected charges
- [ ] Screenshot or note of the working analytics queries (for portfolio)

---

## 🗂️ Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│        DAY 42 — AWS PORTFOLIO PROJECT QUICK REFERENCE       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  S3 Commands:                                                │
│  aws s3 mb s3://BUCKET --region ap-southeast-1              │
│  aws s3 cp file.csv s3://BUCKET/raw/path/                   │
│  aws s3 ls s3://BUCKET/raw/ --recursive                     │
│  aws s3 rm s3://BUCKET --recursive  # Delete all            │
│  aws s3 rb s3://BUCKET --force    # Delete bucket           │
│                                                              │
│  RDS Commands:                                               │
│  aws rds create-db-instance ...                              │
│  aws rds describe-db-instances --db-instance-identifier X   │
│  aws rds delete-db-instance --db-instance-identifier X \    │
│      --skip-final-snapshot                                   │
│                                                              │
│  psql Connection:                                            │
│  psql -h ENDPOINT -U USER -d DBNAME                         │
│  PGPASSWORD='X' psql -h HOST -U USER -d DB -f schema.sql   │
│                                                              │
│  Python boto3 S3:                                            │
│  s3 = boto3.client('s3', region_name='ap-southeast-1')      │
│  s3.upload_file(local, bucket, key)                          │
│  s3.download_file(bucket, key, local)                        │
│  s3.list_objects_v2(Bucket=bucket, Prefix=prefix)           │
│                                                              │
│  Python psycopg2:                                            │
│  conn = psycopg2.connect(host=..., database=..., user=...)  │
│  cursor.copy_expert("COPY table FROM STDIN CSV", file)      │
│  cursor.execute("SELECT ...")                                │
│                                                              │
│  IAM:                                                        │
│  aws iam create-role --role-name X --assume-role-policy ...  │
│  aws iam put-role-policy --role-name X --policy-name X ...  │
│  aws iam delete-role --role-name X                           │
│                                                              │
│  Cost Monitoring:                                            │
│  aws cloudwatch get-metric-statistics ...                    │
│  Console: https://console.aws.amazon.com/billing/           │
│                                                              │
│  Pipeline Flow:                                              │
│  CSV files → S3 raw/ → Python ETL → RDS staging → Star     │
│                                                              │
│  Cleanup:                                                    │
│  ./aws/teardown.sh                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🎉 Week 6 Complete!

You've finished **Week 6: AWS Core**! Here's what you accomplished:

| Day | What You Did |
|-----|-------------|
| Day 36 | AWS Basics — Account, CLI, Regions, Free Tier |
| Day 37 | S3 Deep Dive — Buckets, lifecycle, data lake patterns |
| Day 38 | IAM — Users, groups, roles, policies, security |
| Day 39 | RDS + VPC — PostgreSQL on AWS, networking |
| Day 40 | Week 6 Assessment |
| Day 41 | Weak Spot Review + Portfolio Planning |
| **Day 42** | **🏆 Portfolio Project: Deployed MakanExpress on AWS** |

### What's Next: Week 7 (AWS Glue + Athena + PySpark Intro)

| Day | Topic |
|-----|-------|
| Day 43 | PySpark fundamentals (local mode) |
| Day 44 | AWS Glue: crawlers, jobs, visual ETL |
| Day 45 | AWS Athena: query S3 data with SQL |
| Day 46 | Glue + PySpark: write a Glue job |
| Day 47 | Serverless pipeline: S3 → Glue → Athena |
| Day 48 | Week 7 Assessment |
| Day 49 | Portfolio: Build serverless pipeline for MakanExpress |

---

*Day 42 complete! MakanExpress is live on AWS! 🏗️☁️🎉*
