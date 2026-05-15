# 📅 Day 38 — Sunday, 21 June 2026
# 🔐 IAM: Identity and Access Management

---

## 🎯 Today's Goal

IAM is AWS's security backbone. It controls WHO can do WHAT on WHICH resources. Today you'll learn how to create users, groups, roles, and policies — the skills you need to secure any AWS data pipeline.

**Why this matters:** Every data engineering interview will ask "how do you secure your pipeline?" IAM is the answer. In real jobs, poor IAM = security breach = fired.

---

## ☀️ Morning Block (2 hours): IAM Core Concepts

### Concept 1: IAM Mental Model

```
Think of IAM as a building security system:

AWS Account = The building
IAM User = An employee (has a badge/key)
IAM Group = A department (e.g., "Data Team")
IAM Role = A temporary badge for visitors or services
IAM Policy = A permission list (what rooms can this person enter?)
IAM Permission Boundary = Max access level (even if policy says more)

Key principle: LEAST PRIVILEGE
= Give each user/role ONLY the permissions they need
= "Can you read S3?" → Yes. "Can you delete S3?" → No.
```

### Concept 2: The IAM Hierarchy

```
AWS Account (root) ← NEVER USE THIS
├── IAM Users (people or applications)
│   ├── Has: Access Key ID + Secret Key (for CLI/API)
│   ├── Has: Password (for Console login)
│   └── Has: Attached policies (permissions)
│
├── IAM Groups (collections of users)
│   ├── DataEngineers group → S3 + RDS + Glue permissions
│   ├── DataScientists group → S3 read + Athena permissions
│   └── Admins group → Full access
│
├── IAM Roles (temporary credentials)
│   ├── GlueJobRole → permissions for AWS Glue to access S3
│   ├── LambdaRole → permissions for Lambda functions
│   └── EC2Role → permissions for EC2 instances
│
└── IAM Policies (JSON permission documents)
    ├── AWS Managed (pre-built by AWS)
    └── Custom (you write them)
```

### Concept 3: IAM Policies — The Heart of IAM

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",           // Allow or Deny
            "Action": [                   // What can they do?
                "s3:GetObject",           // Read files
                "s3:ListBucket"           // List files
            ],
            "Resource": [                 // On which resources?
                "arn:aws:s3:::makanexpress-data-lake-dev",
                "arn:aws:s3:::makanexpress-data-lake-dev/*"
            ]
        }
    ]
}

// Policy anatomy:
// Effect: Allow or Deny (Deny ALWAYS wins over Allow)
// Action: AWS service operations (s3:GetObject, rds:DescribeDBInstances, etc.)
// Resource: ARN of the specific resource (or * for all)
// Condition (optional): When does this apply? (e.g., only from certain IP)
```

### Concept 4: Creating IAM Users and Groups (CLI)

```bash
# ─── Create a Data Engineer user ───

# Create user
aws iam create-user --user-name de-alice

# Create access keys for CLI
aws iam create-access-key --user-name de-alice
# SAVE the AccessKeyId and SecretAccessKey!

# Create console login profile
aws iam create-login-profile \
    --user-name de-alice \
    --password 'TempPass123!' \
    --password-reset-required

# ─── Create a Data Engineers group ───

aws iam create-group --group-name DataEngineers

# Add user to group
aws iam add-user-to-group --user-name de-alice --group-name DataEngineers

# Attach a policy to the group
# Option 1: AWS managed policy (quick but broad)
aws iam attach-group-policy \
    --group-name DataEngineers \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Option 2: Custom policy (better — least privilege)
# Create policy file first (see below)
aws iam put-group-policy \
    --group-name DataEngineers \
    --policy-name DataEngineerS3Access \
    --policy-document file://data-engineer-policy.json
```

### Concept 5: Custom IAM Policy for Data Engineers

```json
// data-engineer-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "S3DataLakeRead",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::makanexpress-data-lake-dev",
                "arn:aws:s3:::makanexpress-data-lake-dev/*"
            ]
        },
        {
            "Sid": "S3DataLakeWrite",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "arn:aws:s3:::makanexpress-data-lake-dev/raw/*",
                "arn:aws:s3:::makanexpress-data-lake-dev/processed/*",
                "arn:aws:s3:::makanexpress-data-lake-dev/temp/*"
            ]
        },
        {
            "Sid": "S3DataLakeNoDelete",
            "Effect": "Deny",
            "Action": [
                "s3:DeleteBucket",
                "s3:DeleteObjectVersion"
            ],
            "Resource": [
                "arn:aws:s3:::makanexpress-data-lake-dev",
                "arn:aws:s3:::makanexpress-data-lake-dev/*"
            ]
        },
        {
            "Sid": "RDSReadOnly",
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBInstances",
                "rds:DescribeDBSnapshots"
            ],
            "Resource": "*"
        },
        {
            "Sid": "GlueAccess",
            "Effect": "Allow",
            "Action": [
                "glue:GetDatabase",
                "glue:GetTable",
                "glue:GetTables",
                "glue:StartJobRun",
                "glue:GetJobRun"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AthenaAccess",
            "Effect": "Allow",
            "Action": [
                "athena:StartQueryExecution",
                "athena:GetQueryExecution",
                "athena:GetQueryResults"
            ],
            "Resource": "*"
        }
    ]
}
```

### Concept 6: IAM Roles for Services

```bash
# Create a role that AWS Glue can assume (to access S3)
# This is how services get permissions — they "assume" a role

# Step 1: Create trust policy (who can assume this role)
cat > trust-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Service": "glue.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
    }]
}
EOF

# Step 2: Create the role
aws iam create-role \
    --role-name MakanExpressGlueRole \
    --assume-role-policy-document file://trust-policy.json

# Step 3: Attach permissions to the role
aws iam attach-role-policy \
    --role-name MakanExpressGlueRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

aws iam put-role-policy \
    --role-name MakanExpressGlueRole \
    --policy-name S3DataLakeAccess \
    --policy-document file://data-engineer-policy.json

# Step 4: Use this role when creating Glue jobs (Week 7)
```

```python
# Create IAM role using Python (boto3)
import boto3
import json

iam = boto3.client('iam')

# Create role for Glue
trust_policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "glue.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}

iam.create_role(
    RoleName='MakanExpressGlueRole',
    AssumeRolePolicyDocument=json.dumps(trust_policy)
)

# Attach S3 access policy
s3_policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
        "Resource": [
            "arn:aws:s3:::makanexpress-data-lake-dev",
            "arn:aws:s3:::makanexpress-data-lake-dev/*"
        ]
    }]
}

iam.put_role_policy(
    RoleName='MakanExpressGlueRole',
    PolicyName='S3DataLakeAccess',
    PolicyDocument=json.dumps(s3_policy)
)

# Attach AWS managed Glue service role
iam.attach_role_policy(
    RoleName='MakanExpressGlueRole',
    PolicyArn='arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole'
)

print("✅ Created MakanExpressGlueRole with S3 + Glue permissions")
```

---

### 🏋️ Morning Exercises (15 questions)

1. Create an IAM user `de-alice` with CLI access and console login. Verify with `aws iam list-users`.

2. Create an IAM group `DataEngineers` and add `de-alice` to it. Verify with `aws iam list-groups-for-user --user-name de-alice`.

3. Write a custom IAM policy that allows reading ANY S3 bucket but writing ONLY to `makanexpress-data-lake-dev`. Attach it to the DataEngineers group.

4. What is the difference between an IAM user and an IAM role? When would you use each?

5. Write a trust policy for an EC2 instance role. Create the role and attach an S3 read-only policy.

6. Explain the "least privilege" principle. Why is `AdministratorAccess` dangerous for a data engineer?

7. Write a policy that DENIES deleting S3 objects even if another policy allows it. Remember: Deny always wins.

8. Create a `ReadOnly` group that can only read S3 and RDS (no write, no delete). Use AWS managed policies.

9. Write an IAM policy condition that only allows access from a specific IP range (e.g., your office IP).

10. What happens when two policies conflict? One says "Allow s3:GetObject" and another says "Deny s3:GetObject"? Which wins?

11. Create a role for Lambda that allows reading from S3 and writing to RDS.

12. **Scenario:** A junior data engineer accidentally deleted an S3 bucket. How would you prevent this with IAM?

13. Write a policy that allows a user to access S3 ONLY during business hours (9 AM - 6 PM). Use the `aws:CurrentTime` condition key.

14. Investigate: What is IAM MFA? Why should every IAM user have MFA enabled?

15. **Practical:** Create a complete IAM setup for the MakanExpress team:
    - 3 users: `de-alice`, `de-bob`, `ds-charlie` (data scientist)
    - 2 groups: `DataEngineers` (S3 read/write + Glue + Athena), `DataScientists` (S3 read-only + Athena)
    - 1 role: `MakanExpressGlueRole` for Glue jobs
    - Custom policies for each group

---

### ✅ Morning Answers

<details>
<summary>🔑 Click to reveal selected answers</summary>

**Answer 1:**
```bash
aws iam create-user --user-name de-alice
aws iam create-access-key --user-name de-alice
aws iam create-login-profile --user-name de-alice --password 'TempPass123!' --password-reset-required
```

**Answer 4:**
- **IAM User**: A permanent identity for a specific person or application. Has long-lived credentials (access keys). Use for: developers, CI/CD systems.
- **IAM Role**: A temporary identity that can be "assumed" by users or services. Has short-lived credentials (auto-rotated). Use for: AWS services (Glue, Lambda, EC2), cross-account access, temporary elevated access.

**Answer 7:**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Deny",
        "Action": ["s3:DeleteObject", "s3:DeleteBucket"],
        "Resource": [
            "arn:aws:s3:::makanexpress-data-lake-dev",
            "arn:aws:s3:::makanexpress-data-lake-dev/*"
        ]
    }]
}
```
Deny always wins over Allow — even if another policy grants delete access.

**Answer 10:** Deny always wins. Even if 10 policies say Allow and 1 says Deny, the Deny takes precedence. This is by design for security.

**Answer 12:** Three approaches:
1. IAM policy with explicit Deny on `s3:DeleteBucket` and `s3:DeleteObject`
2. S3 bucket policy with Deny for non-admins
3. Enable MFA Delete on the bucket (requires MFA to delete)

**Answer 13:**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:ListBucket"],
        "Resource": ["arn:aws:s3:::makanexpress-data-lake-dev/*"],
        "Condition": {
            "DateGreaterThan": {"aws:CurrentTime": "2026-01-01T09:00:00Z"},
            "DateLessThan": {"aws:CurrentTime": "2026-12-31T18:00:00Z"}
        }
    }]
}
```

**Answer 15:**
```bash
# Create users
aws iam create-user --user-name de-alice
aws iam create-user --user-name de-bob
aws iam create-user --user-name ds-charlie

# Create groups
aws iam create-group --group-name DataEngineers
aws iam create-group --group-name DataScientists

# Add users to groups
aws iam add-user-to-group --user-name de-alice --group-name DataEngineers
aws iam add-user-to-group --user-name de-bob --group-name DataEngineers
aws iam add-user-to-group --user-name ds-charlie --group-name DataScientists

# Attach policies (use custom policies from Concept 5)
aws iam put-group-policy --group-name DataEngineers \
    --policy-name DataEngineerAccess --policy-document file://data-engineer-policy.json

# DataScientists: read-only
aws iam attach-group-policy --group-name DataScientists \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam attach-group-policy --group-name DataScientists \
    --policy-arn arn:aws:iam::aws:policy/AmazonAthenaFullAccess
```

</details>

---

## 🌤️ Afternoon Block (2 hours): RDS — PostgreSQL on AWS

### Concept 7: What is RDS?

```
RDS = Relational Database Service
= Managed PostgreSQL (or MySQL, SQL Server, etc.)

What "managed" means:
- AWS handles: backups, patches, updates, failover, monitoring
- You handle: schema design, queries, data
- No SSH into the server, no OS management

Why RDS vs local PostgreSQL:
┌─────────────┬──────────────────┬──────────────────┐
│ Feature     │ Local PostgreSQL │ RDS PostgreSQL   │
├─────────────┼──────────────────┼──────────────────┤
│ Setup       │ Install yourself │ Click to create  │
│ Backups     │ Manual (pg_dump) │ Automatic daily  │
│ Scaling     │ Buy bigger server│ Click to scale   │
│ Patching    │ You do it        │ AWS does it       │
│ High avail. │ Complex          │ Multi-AZ option  │
│ Cost        │ Free (local)     │ Free tier 12 mo  │
│ Access      │ localhost        │ Endpoint URL      │
└─────────────┴──────────────────┴──────────────────┘

For learning: local PostgreSQL is fine
For portfolio: deploying on RDS shows cloud skills
```

### Concept 8: Creating an RDS Instance

```bash
# Create a PostgreSQL RDS instance (Free Tier eligible)
aws rds create-db-instance \
    --db-instance-identifier makanexpress-db \
    --db-instance-class db.t3.micro \
    --engine postgres \
    --engine-version 15.7 \
    --master-username makanexpress_admin \
    --master-user-password 'YourStrongPassword123!' \
    --allocated-storage 20 \
    --storage-type gp2 \
    --db-name makanexpress \
    --publicly-accessible \
    --region ap-southeast-1

# Check status (takes 5-10 minutes to create)
aws rds describe-db-instances \
    --db-instance-identifier makanexpress-db \
    --query "DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint.Address,Port:Endpoint.Port}"

# Wait until status = "available"
aws rds wait db-instance-available --db-instance-identifier makanexpress-db
echo "✅ Database is ready!"

# Get the endpoint (connection URL)
aws rds describe-db-instances \
    --db-instance-identifier makanexpress-db \
    --query "DBInstances[0].Endpoint" \
    --output json
# Returns: { "Address": "makanexpress-db.xxxxxx.ap-southeast-1.rds.amazonaws.com", "Port": 5432 }
```

### Concept 9: Connecting to RDS from Your Machine

```bash
# Install PostgreSQL client if not already
# macOS: brew install postgresql
# Linux: sudo apt install postgresql-client

# Connect via psql
psql \
    --host=makanexpress-db.xxxxxx.ap-southeast-1.rds.amazonaws.com \
    --port=5432 \
    --username=makanexpress_admin \
    --dbname=makanexpress
# Enter password when prompted

# Or via Python
pip install psycopg2-binary
```

```python
# Connect to RDS PostgreSQL from Python
import psycopg2

conn = psycopg2.connect(
    host='makanexpress-db.xxxxxx.ap-southeast-1.rds.amazonaws.com',
    port=5432,
    dbname='makanexpress',
    user='makanexpress_admin',
    password='YourStrongPassword123!'
)

cursor = conn.cursor()
cursor.execute("SELECT version();")
print(cursor.fetchone())

# Verify connection works
cursor.execute("""
    SELECT current_database(), current_user, inet_server_addr()
""")
print(cursor.fetchone())
conn.close()
```

### Concept 10: Migrating Local Data to RDS

```python
"""
migrate_to_rds.py
Migrate MakanExpress warehouse from local PostgreSQL to AWS RDS.
"""

import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_AUTOCOMMIT

# ─── Connection settings ───
LOCAL = {
    'host': 'localhost',
    'port': 5432,
    'dbname': 'makanexpress',
    'user': 'postgres',
    'password': ''  # your local password
}

RDS = {
    'host': 'makanexpress-db.xxxxxx.ap-southeast-1.rds.amazonaws.com',
    'port': 5432,
    'dbname': 'makanexpress',
    'user': 'makanexpress_admin',
    'password': 'YourStrongPassword123!'
}

def migrate_schema(local_conn, rds_conn):
    """Migrate table schemas from local to RDS."""
    local_cur = local_conn.cursor()
    rds_cur = rds_conn.cursor()
    
    # Get all table creation SQL
    tables = [
        'dim_date', 'dim_customer', 'dim_restaurant', 'dim_item',
        'dim_payment_method', 'fact_order_items',
        'raw_customers', 'raw_restaurants', 'raw_menu_items',
        'raw_orders', 'raw_order_items', 'raw_payments'
    ]
    
    for table in tables:
        # Get CREATE TABLE statement from local
        local_cur.execute(f"""
            SELECT 'CREATE TABLE ' || table_name || ' (' ||
            string_agg(column_name || ' ' || data_type || 
                CASE WHEN character_maximum_length IS NOT NULL 
                     THEN '(' || character_maximum_length || ')' ELSE '' END ||
                CASE WHEN is_nullable = 'NO' THEN ' NOT NULL' ELSE '' END,
                ', ') || ');'
            FROM information_schema.columns
            WHERE table_name = '{table}'
            GROUP BY table_name
        """)
        # This is simplified — in practice, use pg_dump
        
    print("✅ Schema migration complete")

def migrate_data(local_conn, rds_conn, table_name):
    """Migrate data from local to RDS."""
    local_cur = local_conn.cursor()
    rds_cur = rds_conn.cursor()
    
    # Read all data from local
    local_cur.execute(f"SELECT * FROM {table_name}")
    rows = local_cur.fetchall()
    columns = [desc[0] for desc in local_cur.description]
    
    if rows:
        # Insert into RDS
        placeholders = ','.join(['%s'] * len(columns))
        col_names = ','.join(columns)
        insert_sql = f"INSERT INTO {table_name} ({col_names}) VALUES ({placeholders})"
        
        rds_cur.executemany(insert_sql, rows)
        rds_conn.commit()
    
    print(f"✅ Migrated {len(rows)} rows from {table_name}")

# ─── Run migration ───
if __name__ == '__main__':
    # Better approach: use pg_dump
    # pg_dump -h localhost -U postgres makanexpress > makanexpress_dump.sql
    # psql -h RDS_ENDPOINT -U makanexpress_admin -d makanexpress < makanexpress_dump.sql
    
    print("Best migration method: pg_dump → psql")
    print("Run:")
    print("  pg_dump -h localhost -U postgres makanexpress > dump.sql")
    print("  psql -h RDS_ENDPOINT -U makanexpress_admin -d makanexpress < dump.sql")
```

### Concept 11: RDS Security

```bash
# ─── Security Group (firewall for RDS) ───

# Get your public IP (for allowing access)
curl https://checkip.amazonaws.com

# Create a security group
aws ec2 create-security-group \
    --group-name makanexpress-rds-sg \
    --description "Allow PostgreSQL access from my IP"

# Allow inbound PostgreSQL (port 5432) from YOUR IP only
aws ec2 authorize-security-group-ingress \
    --group-name makanexpress-rds-sg \
    --protocol tcp \
    --port 5432 \
    --cidr YOUR_IP/32
# Example: --cidr 203.0.113.50/32

# Apply security group to RDS
aws rds modify-db-instance \
    --db-instance-identifier makanexpress-db \
    --vpc-security-group-ids sg-xxxxxxxx \
    --apply-immediately

# ⚠️ NEVER use 0.0.0.0/0 for RDS! (allows entire internet)
# ⚠️ Always restrict to specific IPs
```

---

### 🏋️ Afternoon Exercises (15 questions)

16. Create an RDS PostgreSQL instance in Singapore region. Wait until it's available.

17. Connect to your RDS instance via psql. Verify the connection by running `SELECT version();`.

18. Connect to RDS via Python (psycopg2). Run a test query.

19. Use `pg_dump` to export your local MakanExpress database. Import it into RDS.

20. Verify all tables exist in RDS: `SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';`

21. Run one of your analytical queries from Week 4 against RDS. Does it work the same as local?

22. Create a security group that allows PostgreSQL access from YOUR IP only. Apply it to your RDS instance.

23. **Cost estimation:** Your RDS instance is db.t3.micro (free tier). After free tier ends, what's the monthly cost in Singapore? Check AWS pricing.

24. Write a Python script that benchmarks query performance: run the same query on local PostgreSQL and RDS. Compare execution times.

25. Create a read replica of your RDS instance. What's the benefit of a read replica?

26. Set up automated backups on your RDS instance. Verify backup window is configured.

27. **Security exercise:** Try connecting to RDS from a different IP (use a phone hotspot). Does the connection fail? Why?

28. Write a migration script that can migrate data in both directions: local → RDS and RDS → local.

29. Configure your Airflow DAG (from Week 5) to use RDS instead of local PostgreSQL. Create a new connection `makanexpress_rds_db`.

30. **Challenge:** Deploy the complete MakanExpress warehouse on RDS:
    - Create RDS instance
    - Migrate all schema + data
    - Update Airflow connection
    - Run the full ETL DAG against RDS
    - Verify data quality checks pass
    - Document the setup in your README

---

### ✅ Selected Answers

<details>
<summary>🔑 Click to reveal selected answers</summary>

**Answer 16:**
```bash
aws rds create-db-instance \
    --db-instance-identifier makanexpress-db \
    --db-instance-class db.t3.micro \
    --engine postgres \
    --master-username makanexpress_admin \
    --master-user-password 'StrongPassword123!' \
    --allocated-storage 20 \
    --db-name makanexpress \
    --publicly-accessible \
    --region ap-southeast-1

aws rds wait db-instance-available --db-instance-identifier makanexpress-db
```

**Answer 19:**
```bash
# Export
pg_dump -h localhost -U postgres makanexpress > /tmp/makanexpress_dump.sql

# Import
psql \
    -h makanexpress-db.xxxxxx.ap-southeast-1.rds.amazonaws.com \
    -U makanexpress_admin \
    -d makanexpress \
    < /tmp/makanexpress_dump.sql
```

**Answer 22:**
```bash
# Get your IP
MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "My IP: $MY_IP"

# Create security group
SG_ID=$(aws ec2 create-security-group \
    --group-name makanexpress-rds-sg \
    --description "MakanExpress RDS access" \
    --query 'GroupId' --output text)

# Allow your IP only
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp --port 5432 \
    --cidr ${MY_IP}/32

# Apply to RDS
aws rds modify-db-instance \
    --db-instance-identifier makanexpress-db \
    --vpc-security-group-ids $SG_ID \
    --apply-immediately
```

**Answer 24:**
```python
import psycopg2
import time

queries = [
    ("Daily Revenue", """
        SELECT DATE(o.order_date), SUM(o.total_amount)
        FROM orders o WHERE o.status = 'completed'
        GROUP BY DATE(o.order_date)
    """),
    ("Top Customers", """
        SELECT c.customer_name, SUM(o.total_amount)
        FROM orders o JOIN customers c ON o.customer_id = c.customer_id
        GROUP BY c.customer_name ORDER BY SUM(o.total_amount) DESC LIMIT 10
    """),
]

for name, query in queries:
    for env, config in [('Local', LOCAL_CONFIG), ('RDS', RDS_CONFIG)]:
        conn = psycopg2.connect(**config)
        cur = conn.cursor()
        
        start = time.time()
        cur.execute(query)
        rows = cur.fetchall()
        elapsed = time.time() - start
        
        print(f"{name} ({env}): {len(rows)} rows in {elapsed:.3f}s")
        conn.close()
```

**Answer 29:**
```bash
# Add RDS connection in Airflow
airflow connections add makanexpress_rds_db \
    --conn-type postgres \
    --conn-host makanexpress-db.xxxxxx.ap-southeast-1.rds.amazonaws.com \
    --conn-schema makanexpress \
    --conn-login makanexpress_admin \
    --conn-password 'StrongPassword123!' \
    --conn-port 5432

# Update DAG: change CONNECTION_ID
# CONNECTION_ID = 'makanexpress_rds_db'  # use RDS instead of local
```

</details>

---

## 🌙 Evening Block (1 hour): Review & VPC Preview

### IAM Cheat Sheet

```
User:     Person/application with long-lived credentials
Group:    Collection of users with shared permissions
Role:     Temporary credentials for AWS services
Policy:   JSON document defining permissions
Rule:     Least privilege — only what's needed
Deny:     Always wins over Allow
```

### RDS Cheat Sheet

```bash
aws rds create-db-instance ...      # create database
aws rds describe-db-instances ...   # check status
aws rds wait db-instance-available  # wait for ready
aws rds delete-db-instance ...      # delete (careful!)
pg_dump > dump.sql                  # export
psql < dump.sql                     # import
```

### 📝 Today's Checklist

- [ ] IAM users, groups, and roles created
- [ ] Custom IAM policies written and attached
- [ ] Understand least privilege principle
- [ ] RDS PostgreSQL instance created in Singapore
- [ ] Connected to RDS from local machine
- [ ] Data migrated from local to RDS
- [ ] Security group configured (IP-restricted access)
- [ ] Ready for tomorrow: VPC + Assessment

### 📖 Further Reading (Free)

- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [IAM Policy Simulator](https://policysim.aws.amazon.com/) — test policies before applying
- [RDS Getting Started](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.PostgreSQL.html)

---

*Day 38 complete! Tomorrow: VPC basics + Day 40 assessment checkpoint.* 🌐
