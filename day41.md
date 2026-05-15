# 📅 Day 41 — Wednesday, 24 June 2026
# 🔧 Week 6 Weak Spot Deep Dive + Portfolio Prep

---

## 🎯 Today's Goal

No new AWS concepts today. You took the Day 40 assessment and now it's time to **deeply review your weakest areas** before tomorrow's big portfolio project.

Today's structure:
- **Morning (3 hours):** Deep review of 2 weak areas with targeted exercises
- **Afternoon (2 hours):** Plan tomorrow's portfolio project — architecture, resource checklist, setup
- **Evening (1 hour):** Light review + rest before the big day

> **💡 Important:** Tomorrow you'll deploy the entire MakanExpress warehouse onto AWS (S3 + RDS + IAM). Today is about making sure you're ready. Don't skip the planning — it'll save you hours tomorrow.

---

## ☀️ Morning Block (3 hours): Weak Spot Deep Dive

### Step 1: Identify Your Weak Spots (15 min)

Look at your Day 40 assessment scores:

| Section | My Score | Status |
|---------|----------|--------|
| A: AWS Fundamentals | ___/20 | ✅ / ⚠️ / ❌ |
| B: S3 | ___/40 | ✅ / ⚠️ / ❌ |
| C: IAM | ___/30 | ✅ / ⚠️ / ❌ |
| D: RDS & Networking | ___/10 | ✅ / ⚠️ / ❌ |

**Rule:** Any section below 70% is your weak spot. Pick your **2 weakest areas** and work through the corresponding deep-dive exercises below.

> **Common weak spots for Week 6:** Most students struggle with **IAM policies** (JSON syntax, effect ordering) and **VPC/RDS networking** (security groups, private subnets). If you scored well on both, focus on whichever was lowest.

---

### Step 2: Weak Spot Deep Dive — IAM Policies (75 min)

> Work through this section if IAM (Section C) was a weak spot.

#### 🧠 Why IAM Is Hard

IAM policies are **JSON documents that read like legal contracts**. They have explicit rules, implicit defaults, and a specific evaluation order. The tricky parts:

1. **Default Deny** — If no policy explicitly allows an action, it's denied
2. **Explicit Deny wins everything** — Even if 10 policies allow it, 1 deny blocks it
3. **Resource format** — S3 bucket ARNs vs object ARNs are different
4. **Principal confusion** — Trust policies vs permission policies are different things

#### 📝 Exercise 1: Policy Evaluation Order (15 min)

Given these 3 policies attached to user `alice`:

**Policy A (Group policy — DataEngineers):**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": "s3:*",
        "Resource": "*"
    }]
}
```

**Policy B (User policy):**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": "s3:*",
        "Resource": [
            "arn:aws:s3:::makanexpress-*",
            "arn:aws:s3:::makanexpress-*/*"
        ]
    }]
}
```

**Policy C (SCP — Service Control Policy):**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Deny",
        "Action": "s3:DeleteBucket",
        "Resource": "*"
    }]
}
```

For each scenario, answer: **ALLOWED or DENIED? Why?**

1. Alice uploads a file to `s3://makanexpress-raw/orders.csv`
2. Alice deletes `s3://makanexpress-raw/orders.csv`
3. Alice deletes the bucket `s3://makanexpress-raw`
4. Alice uploads a file to `s3://other-company-data/secret.xlsx`
5. Alice lists objects in `s3://makanexpress-analytics/`

<details>
<summary>🔑 Answers</summary>

1. **ALLOWED.** Policy A allows `s3:*` on `*` (all resources). Policy B also allows it. No deny. ✅
2. **ALLOWED.** `s3:DeleteObject` is covered by `s3:*` in Policy A. The deny in Policy C only blocks `s3:DeleteBucket`, not `s3:DeleteObject`. ✅
3. **DENIED.** Policy C explicitly denies `s3:DeleteBucket` on all resources. Explicit deny wins all allows. ❌
4. **ALLOWED by Policy A** (s3:* on *), but **DENIED by Policy B** which only covers `makanexpress-*`. Wait — Policy A allows it. Policy B is more restrictive but doesn't deny. Multiple allows are additive. So: **ALLOWED** by Policy A. This is why overly permissive policies are dangerous! ✅
5. **ALLOWED.** `s3:ListBucket` on `makanexpress-analytics` — allowed by Policy A and B. No deny. ✅

**Key lesson:** Policy A (`s3:*` on `*`) is too permissive for a real company. Policy B is better-scoped but Policy A overrides it by allowing everything.

</details>

#### 📝 Exercise 2: Write a Least-Privilege Policy (20 min)

Write an IAM policy for **MakanExpress's ETL pipeline** with these requirements:
- Read objects from `s3://makanexpress-datalake/raw/`
- Write objects to `s3://makanexpress-datalake/processed/`
- List both prefixes
- Cannot read/write to `analytics/` or any other bucket
- Cannot delete anything

<details>
<summary>🔑 Answer</summary>

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowListRawAndProcessed",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::makanexpress-datalake",
            "Condition": {
                "StringLike": {
                    "s3:prefix": ["raw/*", "processed/*"]
                }
            }
        },
        {
            "Sid": "AllowReadRaw",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::makanexpress-datalake/raw/*"
        },
        {
            "Sid": "AllowWriteProcessed",
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::makanexpress-datalake/processed/*"
        },
        {
            "Sid": "DenyDelete",
            "Effect": "Deny",
            "Action": [
                "s3:DeleteObject",
                "s3:DeleteObjectVersion",
                "s3:DeleteBucket"
            ],
            "Resource": "*"
        }
    ]
}
```

**Key points:**
- `s3:ListBucket` applies to the **bucket** resource (no `/*`), not the object resource
- `s3:GetObject` / `s3:PutObject` apply to **object** resources (with `/*`)
- The `Condition` block limits listing to only `raw/` and `processed/` prefixes
- Explicit deny on delete ensures no other policy can accidentally grant delete permission

</details>

#### 📝 Exercise 3: Role vs User Scenario (15 min)

For each scenario, choose: **IAM User with access keys** or **IAM Role**? Explain why.

1. A developer named Priya needs CLI access to S3 from her laptop
2. An AWS Glue ETL job needs to read from S3 and write to RDS
3. A cron job on an EC2 instance runs a daily data load
4. Your CI/CD pipeline (GitHub Actions) deploys infrastructure
5. A Lambda function processes uploaded CSV files
6. Your team uses Airflow on EC2 to orchestrate pipelines

<details>
<summary>🔑 Answers</summary>

1. **IAM User.** Human with a permanent identity. Create user `priya`, add to `DataEngineers` group. She gets access keys for CLI. Consider MFA requirement for security.

2. **IAM Role.** AWS service assuming a role. Create `GlueETLRole` with trust policy for `glue.amazonaws.com`. Attach S3 read + RDS write permissions. Credentials auto-rotate.

3. **IAM Role (Instance Profile).** Attach a role to the EC2 instance. The cron job uses the instance's temporary credentials — no access keys to manage or rotate. Way better than storing keys in crontab!

4. **OIDC Identity Provider.** Not exactly user or role — use GitHub OIDC to let GitHub Actions assume an IAM role. This is the modern best practice — no long-lived access keys stored as GitHub secrets.

5. **IAM Role.** Lambda execution role with trust policy for `lambda.amazonaws.com`. Attach CloudWatch Logs + S3 permissions. Auto-provisioned temporary credentials.

6. **IAM Role (Instance Profile).** Like #3 — attach to the EC2 instance running Airflow. Each DAG task inherits the instance role's permissions. For more granular control, use Airflow connections with IAM role chaining.

**Pattern:** If it's a **service** → Role. If it's a **human** → User. If it's **external CI/CD** → OIDC.

</details>

#### 📝 Exercise 4: Trust Policy Debugging (15 min)

This trust policy is supposed to let an EC2 instance assume a role, but it doesn't work. **Find and fix the 3 errors:**

```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Service": "ec2.amazonaws.org"
        },
        "Action": "sts:Assume",
        "Condition": {
            "StringEquals": {
                "aws:SourceAccount": "123456789012"
            }
        }
    }]
}
```

<details>
<summary>🔑 Answers</summary>

**Error 1:** `"Service": "ec2.amazonaws.org"` — Wrong domain. Should be `"ec2.amazonaws.com"` (not .org).

**Error 2:** `"Action": "sts:Assume"` — Wrong action. Should be `"sts:AssumeRole"` (Assume is not a valid STS action).

**Error 3:** (Subtle) The policy looks like it could work with just those two fixes, but there's actually a potential issue: if this role is in a different account than `123456789012`, the `SourceAccount` condition could cause failures. However, the two hard errors above are the main issues.

**Corrected:**
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole",
        "Condition": {
            "StringEquals": {
                "aws:SourceAccount": "123456789012"
            }
        }
    }]
}
```

</details>

#### 📝 Exercise 5: IAM Policy for RDS Access (10 min)

Write a policy for a data analyst named Wei Ling who needs:
- Connect to the RDS `makanexpress-db` instance (via RDS Data API)
- Read-only access to S3 analytics outputs: `s3://makanexpress-datalake/analytics/*`
- Cannot access raw or processed data
- Cannot modify RDS instances

<details>
<summary>🔑 Answer</summary>

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowRDSDataAPI",
            "Effect": "Allow",
            "Action": [
                "rds-data:ExecuteStatement",
                "rds-data:BatchExecuteStatement",
                "rds-data:BeginTransaction",
                "rds-data:CommitTransaction",
                "rds-data:RollbackTransaction"
            ],
            "Resource": "arn:aws:rds:ap-southeast-1:123456789012:db:makanexpress-db"
        },
        {
            "Sid": "AllowDescribeRDS",
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBInstances",
                "rds:DescribeDBClusters"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowReadAnalytics",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::makanexpress-datalake",
                "arn:aws:s3:::makanexpress-datalake/analytics/*"
            ]
        },
        {
            "Sid": "DenyModifyRDS",
            "Effect": "Deny",
            "Action": [
                "rds:ModifyDBInstance",
                "rds:DeleteDBInstance",
                "rds:RebootDBInstance"
            ],
            "Resource": "*"
        }
    ]
}
```

</details>

---

### Step 3: Weak Spot Deep Dive — VPC & RDS Networking (75 min)

> Work through this section if RDS/Networking (Section D) was a weak spot.

#### 🧠 VPC Mental Model Refresher

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR VPC                              │
│              10.0.0.0/16 (65,536 IPs)                    │
│                                                          │
│  ┌──────────────────────┐  ┌──────────────────────┐     │
│  │  Public Subnet        │  │  Private Subnet       │     │
│  │  10.0.1.0/24         │  │  10.0.2.0/24          │     │
│  │                       │  │                        │     │
│  │  ┌─────────┐         │  │  ┌──────────────┐     │     │
│  │  │ Bastion │──SSH────┼──┼─→│  RDS Postgres │     │     │
│  │  │  Host   │         │  │  │  Port 5432    │     │     │
│  │  └─────────┘         │  │  └──────────────┘     │     │
│  │       ↑              │  │                        │     │
│  │   Internet GW        │  │  ┌──────────────┐     │     │
│  └──────────────────────┘  │  │  EC2 Airflow  │     │     │
│                             │  └──────────────┘     │     │
│                             └──────────────────────┘     │
│                                      ↑                   │
│                              (No direct internet)        │
└─────────────────────────────────────────────────────────┘
```

**Key rules:**
- **RDS goes in PRIVATE subnet** — no direct internet access
- **Bastion host goes in PUBLIC subnet** — has public IP, you SSH to it first
- **Security groups** act as firewalls: inbound rules + outbound rules
- **NAT Gateway** lets private subnet reach internet (for updates, downloads) without being reachable FROM internet

#### 📝 Exercise 6: Security Group Rules (15 min)

Your RDS PostgreSQL instance is in a private subnet with security group `sg-rds`. Your EC2 Airflow instance is in the same VPC with security group `sg-airflow`. Your office IP is `203.0.113.50`.

Write the security group rules for `sg-rds`:

| Type | Protocol | Port | Source | Why? |
|------|----------|------|--------|------|
| 1. | | | | |
| 2. | | | | |

Write the security group rules for `sg-airflow`:

| Type | Protocol | Port | Source | Why? |
|------|----------|------|--------|------|
| 1. | | | | |
| 2. | | | | |
| 3. | | | | |

<details>
<summary>🔑 Answers</summary>

**sg-rds (RDS PostgreSQL):**

| Type | Protocol | Port | Source | Why? |
|------|----------|------|--------|------|
| Inbound | TCP | 5432 | `sg-airflow` | Allow Airflow to connect to DB |
| Outbound | All | All | `0.0.0.0/0` | Allow DB to reach internet for updates, S3 (via NAT) |

Actually, for least privilege, outbound could be restricted. But RDS managed instances need outbound for:
- AWS service communication
- S3 integration (log exports, backups)

So outbound `0.0.0.0/0` is standard for RDS.

**sg-airflow (EC2 Airflow):**

| Type | Protocol | Port | Source | Why? |
|------|----------|------|--------|------|
| Inbound | TCP | 22 | `203.0.113.50/32` | SSH from office only |
| Inbound | TCP | 8080 | `203.0.113.50/32` | Airflow WebUI from office only |
| Outbound | All | All | `0.0.0.0/0` | Allow Airflow to reach S3, RDS, internet |

**Important:** The RDS inbound source is `sg-airflow` (the security group itself, not a CIDR). This means any EC2 instance in `sg-airflow` can connect. This is the AWS best practice for inter-service communication — reference security groups, not IP addresses.

</details>

#### 📝 Exercise 7: RDS Connection Troubleshooting (15 min)

You created an RDS PostgreSQL instance but can't connect from your laptop. For each scenario, identify the likely cause and fix:

1. **Error:** `could not connect to server: Connection timed out`
   - Your RDS endpoint is: `makanexpress-db.xxxxx.ap-southeast-1.rds.amazonaws.com`
   - You're trying to connect from: your laptop in KL

2. **Error:** `FATAL: password authentication failed for user "admin"`
   - You're using: `psql -h ENDPOINT -U admin -d makanexpress`

3. **Error:** `FATAL: no pg_hba.conf entry for host "203.0.113.50"`
   - Your IP is `203.0.113.50`

4. **Error:** `could not connect to server: Connection refused`
   - The instance was created 3 minutes ago

5. **Error:** `FATAL: database "makanexpress" does not exist`
   - You created the instance but forgot one parameter

<details>
<summary>🔑 Answers</summary>

1. **Cause:** RDS is in a private subnet (no public IP) or security group doesn't allow your IP.
   **Fix:** Either:
   - Set RDS to "Publicly Accessible: Yes" AND add your IP to the security group inbound rule (TCP 5432)
   - OR use a bastion host / VPN to reach the private subnet
   - **⚠️ For production:** Never use "Publicly Accessible: Yes". Use bastion/VPN only.

2. **Cause:** Wrong password or the username is case-sensitive.
   **Fix:** Double-check the master password. If you forgot it, reset it: `aws rds modify-db-instance --db-instance-identifier makanexpress-db --master-user-password 'NewPassword123!' --apply-immediately`

3. **Cause:** Your IP `203.0.113.50` is not in the RDS security group's allowed inbound sources.
   **Fix:** Add your IP to the security group:
   ```bash
   aws ec2 authorize-security-group-ingress \
       --group-id sg-xxx --protocol tcp --port 5432 \
       --cidr 203.0.113.50/32
   ```

4. **Cause:** RDS instance is still being created. It takes 5-10 minutes for a new RDS instance to become available.
   **Fix:** Wait. Check status: `aws rds describe-db-instances --db-instance-identifier makanexpress-db --query 'DBInstances[0].DBInstanceStatus'`

5. **Cause:** You didn't specify `--db-name makanexpress` when creating the instance. Only `postgres` database exists.
   **Fix:** Connect to the `postgres` database first, then create the database:
   ```bash
   psql -h ENDPOINT -U admin -d postgres -c "CREATE DATABASE makanexpress;"
   ```
   Or recreate the instance with `--db-name makanexpress`.

</details>

#### 📝 Exercise 8: Cost-Estimate a Data Setup (15 min)

You're setting up MakanExpress's AWS infrastructure for the portfolio project. Estimate the **monthly cost** for these resources (all in `ap-southeast-1`):

| Resource | Specification | Free Tier? |
|----------|--------------|------------|
| S3 | 2 GB storage, 1000 PUT, 5000 GET/month | First 5 GB free (12 months) |
| RDS PostgreSQL | db.t3.micro, 20 GB storage, running 24/7 | 750 hours free (12 months) |
| Data Transfer | 5 GB out/month | First 1 GB free |
| NAT Gateway | 1 gateway, 10 GB processed | **NOT free** ⚠️ |

Calculate:
1. What's the total monthly cost if you're within the 12-month free tier?
2. What happens after the free tier expires?
3. What's the cheapest way to avoid NAT Gateway charges during development?

<details>
<summary>🔑 Answers</summary>

1. **During Free Tier (first 12 months):**
   - S3: $0 (2 GB < 5 GB free tier)
   - RDS: $0 (750 hours covers 24/7 for one db.t3.micro)
   - Data Transfer: ~$0.40 (5 GB - 1 GB free = 4 GB × $0.114/GB for Singapore)
   - NAT Gateway: **~$36.50/month** ($0.045/hour × 24 × 30 + $0.01/GB × 10 GB) ← This is the expensive one!
   - **Total: ~$37/month** (almost entirely NAT Gateway)

2. **After Free Tier:**
   - S3: ~$0.05 (2 GB × $0.025/GB)
   - RDS: ~$15 (db.t3.micro on-demand)
   - Data Transfer: ~$0.40
   - NAT Gateway: ~$36.50
   - **Total: ~$52/month**

3. **Avoiding NAT Gateway during dev:**
   - **Don't create a NAT Gateway at all** during development
   - Instead, temporarily make RDS "Publicly Accessible" and restrict security group to your IP only
   - Or use a bastion host in the public subnet (cheaper — just an EC2 t2.micro, which IS free tier)
   - Or use AWS Systems Manager Session Manager (free, no public IP needed)
   - **For the portfolio project:** Skip the NAT Gateway entirely. Use Publicly Accessible RDS with IP-restricted security group. This costs $0.

**💡 Interview tip:** When an interviewer asks "how would you estimate cloud costs," show this kind of breakdown. Mentioning NAT Gateway costs specifically shows real-world awareness — it's a common surprise in AWS bills.

</details>

#### 📝 Exercise 9: End-to-End Security Scenario (15 min)

Design the **complete security setup** for MakanExpress's AWS data platform:

**Requirements:**
- 1 S3 bucket: `makanexpress-datalake` with 3 prefixes (raw/, processed/, analytics/)
- 1 RDS PostgreSQL: `makanexpress-db`
- 3 users: `etl-pipeline` (service), `data-analyst` (Wei Ling), `de-lead` (you)
- Follow least privilege

Fill in the table:

| Entity | Type | Access Level | S3 Access | RDS Access |
|--------|------|-------------|-----------|------------|
| etl-pipeline | | | | |
| data-analyst | | | | |
| de-lead | | | | |

<details>
<summary>🔑 Answers</summary>

| Entity | Type | Access Level | S3 Access | RDS Access |
|--------|------|-------------|-----------|------------|
| etl-pipeline | IAM Role | Read raw + write processed | `raw/*` (read), `processed/*` (write) | Read + Write all tables |
| data-analyst | IAM User (in DataAnalysts group) | Read analytics only | `analytics/*` (read-only) | Read-only SELECT queries |
| de-lead | IAM User (in DataEngineers group) | Full data platform | Full bucket access | Full DB access |

**Additional details:**
- `etl-pipeline` is a **role** assumed by Glue/Lambda/EC2 — no long-lived credentials
- `data-analyst` gets **user** credentials with MFA required + read-only everywhere
- `de-lead` gets broader access but still shouldn't have IAM admin or billing access
- All RDS connections go through security groups, not IP-based
- S3 bucket policy should deny unencrypted uploads (server-side encryption enforced)

</details>

---

## 🌤️ Afternoon Block (2 hours): Portfolio Project Planning

### Step 4: Plan Tomorrow's Architecture (30 min)

Tomorrow (Day 42) you'll deploy the **MakanExpress Food Delivery Warehouse** onto AWS. The project was built locally in Weeks 4-5 with:
- Star schema (fact_orders, dim_customer, dim_restaurant, dim_date, dim_menu_item)
- SCD Type 2 on dim_customer and dim_restaurant
- Airflow DAGs for ETL orchestration
- dbt models for transformation

Now you'll deploy it to AWS using **S3 + RDS + IAM**.

#### 📐 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS (ap-southeast-1)                          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  S3: makanexpress-datalake                               │   │
│  │  ┌──────────┐  ┌───────────┐  ┌───────────┐            │   │
│  │  │  raw/    │  │ processed/ │  │ analytics/ │            │   │
│  │  │ CSV files│  │ Parquet    │  │ Aggregated │            │   │
│  │  │ (source) │  │ (cleaned)  │  │ (reports)  │            │   │
│  │  └──────────┘  └───────────┘  └───────────┘            │   │
│  └──────────────────────────────────────────────────────────┘   │
│       ↑ upload              ↑ ETL              ↑ query          │
│       │                     │                   │               │
│  ┌────┴─────────────────────┴───────────────────┴──────────┐   │
│  │  RDS PostgreSQL: makanexpress-db                         │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │  Schema: makanexpress                             │   │   │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐         │   │   │
│  │  │  │raw_*     │ │star schema│ │analytics │         │   │   │
│  │  │  │(staging) │ │(facts/dim)│ │(reports) │         │   │   │
│  │  │  └──────────┘ └──────────┘ └──────────┘         │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  IAM                                                      │   │
│  │  • Role: MakanExpressETLRole (S3 read/write + RDS)       │   │
│  │  • User: de-lead (you — admin for setup)                 │   │
│  │  • Policy: DataLakeAccess (scoped to bucket + prefixes)  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VPC (default is fine for portfolio)                      │   │
│  │  • Security Group: Allow PostgreSQL from your IP          │   │
│  │  • RDS: Publicly Accessible (dev only, IP-restricted)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

Local Machine:
  ┌──────────────────────┐
  │ Python + boto3       │──── upload CSVs ────→ S3 raw/
  │                      │──── load to RDS ────→ PostgreSQL
  │ pg_dump / psql       │──── migrate schema ─→ RDS
  └──────────────────────┘
```

#### 📋 Resource Checklist for Tomorrow

Before tomorrow, verify these AWS resources are ready (or will be created):

| # | Resource | Status | Action |
|---|----------|--------|--------|
| 1 | AWS Account | ✅ Created Day 36 | Verify login works |
| 2 | IAM User `de-lead` | ✅ Created Day 38 | Verify CLI access: `aws sts get-caller-identity` |
| 3 | S3 Bucket `makanexpress-datalake` | ⬜ Create tomorrow | `aws s3 mb s3://makanexpress-datalake` |
| 4 | S3 Prefixes (raw/, processed/, analytics/) | ⬜ Create tomorrow | Upload placeholder files |
| 5 | RDS PostgreSQL `makanexpress-db` | ⬜ Create tomorrow | db.t3.micro, 20GB, Singapore |
| 6 | Security Group for RDS | ⬜ Create tomorrow | Allow TCP 5432 from your IP |
| 7 | IAM Role `MakanExpressETLRole` | ⬜ Create tomorrow | For future Glue/Lambda use |
| 8 | Local Python env with boto3 | ⬜ Verify tomorrow | `pip install boto3 psycopg2-binary pandas` |

---

### Step 5: Prepare Your Local Files (45 min)

#### 📝 Exercise 10: Organize Source Data

Your local `food-delivery-warehouse` repo from Day 28/35 has seed data. Let's prepare it for AWS upload.

```bash
cd ~/projects/food-delivery-warehouse

# Check what seed data you have
ls -la seed/
# Should have: customers.csv, restaurants.csv, menu_items.csv, orders.csv, order_items.csv

# Verify each file has data
for f in seed/*.csv; do echo "$f: $(wc -l < $f) rows"; done

# Create a manifest file for S3 uploads
cat > seed/manifest.json << 'EOF'
{
    "project": "MakanExpress Food Delivery Warehouse",
    "version": "1.0",
    "upload_date": "2026-06-25",
    "files": {
        "customers": "seed/customers.csv",
        "restaurants": "seed/restaurants.csv",
        "menu_items": "seed/menu_items.csv",
        "orders": "seed/orders.csv",
        "order_items": "seed/order_items.csv"
    },
    "target_s3_bucket": "makanexpress-datalake",
    "target_s3_prefix": "raw/makanexpress/",
    "source": "local_food_delivery_warehouse_project"
}
EOF

echo "✅ Manifest created"
```

#### 📝 Exercise 11: Prepare Schema SQL

Create a combined schema file for RDS migration:

```bash
cd ~/projects/food-delivery-warehouse

# Combine all schema files into one migration script
cat > sql/01_aws_schema.sql << 'EOSQL'
-- MakanExpress Data Warehouse Schema — AWS Deployment
-- Deploy to: RDS PostgreSQL (makanexpress-db)

-- Drop existing if any (fresh deployment)
DROP SCHEMA IF EXISTS makanexpress CASCADE;
CREATE SCHEMA makanexpress;

-- Set search path
SET search_path TO makanexpress;

-- Raw/Staging tables
CREATE TABLE makanexpress.raw_customers (
    customer_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(150),
    phone VARCHAR(20),
    city VARCHAR(50),
    registered_date DATE
);

CREATE TABLE makanexpress.raw_restaurants (
    restaurant_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    cuisine VARCHAR(50),
    city VARCHAR(50),
    rating DECIMAL(3,2),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE makanexpress.raw_menu_items (
    item_id INTEGER PRIMARY KEY,
    restaurant_id INTEGER,
    item_name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(8,2),
    is_available BOOLEAN DEFAULT TRUE
);

CREATE TABLE makanexpress.raw_orders (
    order_id INTEGER PRIMARY KEY,
    customer_id INTEGER,
    restaurant_id INTEGER,
    order_date TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    delivery_city VARCHAR(50)
);

CREATE TABLE makanexpress.raw_order_items (
    order_item_id INTEGER PRIMARY KEY,
    order_id INTEGER,
    item_id INTEGER,
    quantity INTEGER,
    unit_price DECIMAL(8,2),
    subtotal DECIMAL(10,2)
);

-- Star Schema: Dimension Tables (SCD Type 2)
CREATE TABLE makanexpress.dim_customer (
    customer_sk SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    customer_name VARCHAR(100),
    email VARCHAR(150),
    phone VARCHAR(20),
    city VARCHAR(50),
    valid_from DATE NOT NULL DEFAULT CURRENT_DATE,
    valid_to DATE,
    is_current BOOLEAN DEFAULT TRUE,
    CONSTRAINT uk_dim_customer_nat UNIQUE (customer_id, valid_from)
);

CREATE TABLE makanexpress.dim_restaurant (
    restaurant_sk SERIAL PRIMARY KEY,
    restaurant_id INTEGER NOT NULL,
    restaurant_name VARCHAR(100),
    cuisine VARCHAR(50),
    city VARCHAR(50),
    rating DECIMAL(3,2),
    is_active BOOLEAN,
    valid_from DATE NOT NULL DEFAULT CURRENT_DATE,
    valid_to DATE,
    is_current BOOLEAN DEFAULT TRUE,
    CONSTRAINT uk_dim_restaurant_nat UNIQUE (restaurant_id, valid_from)
);

CREATE TABLE makanexpress.dim_date (
    date_sk SERIAL PRIMARY KEY,
    date_key DATE NOT NULL UNIQUE,
    day_of_week INTEGER,
    day_name VARCHAR(10),
    month INTEGER,
    month_name VARCHAR(10),
    quarter INTEGER,
    year INTEGER,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN DEFAULT FALSE,
    holiday_name VARCHAR(50)
);

CREATE TABLE makanexpress.dim_menu_item (
    item_sk SERIAL PRIMARY KEY,
    item_id INTEGER NOT NULL,
    restaurant_id INTEGER,
    item_name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(8,2),
    is_available BOOLEAN DEFAULT TRUE
);

-- Star Schema: Fact Table
CREATE TABLE makanexpress.fact_orders (
    order_sk SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    customer_sk INTEGER REFERENCES makanexpress.dim_customer(customer_sk),
    restaurant_sk INTEGER REFERENCES makanexpress.dim_restaurant(restaurant_sk),
    date_sk INTEGER REFERENCES makanexpress.dim_date(date_sk),
    item_sk INTEGER REFERENCES makanexpress.dim_menu_item(item_sk),
    quantity INTEGER,
    unit_price DECIMAL(8,2),
    subtotal DECIMAL(10,2),
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    delivery_city VARCHAR(50)
);

-- Analytics Views
CREATE VIEW makanexpress.vw_daily_revenue AS
SELECT
    d.date_key,
    d.day_name,
    d.month_name,
    r.restaurant_name,
    r.cuisine,
    COUNT(DISTINCT f.order_id) AS total_orders,
    SUM(f.subtotal) AS total_revenue,
    AVG(f.subtotal) AS avg_order_value
FROM makanexpress.fact_orders f
JOIN makanexpress.dim_date d ON f.date_sk = d.date_sk
JOIN makanexpress.dim_restaurant r ON f.restaurant_sk = r.restaurant_sk
GROUP BY d.date_key, d.day_name, d.month_name, r.restaurant_name, r.cuisine;

-- Grant usage (for future analyst access)
-- GRANT USAGE ON SCHEMA makanexpress TO data_analyst;
-- GRANT SELECT ON ALL TABLES IN SCHEMA makanexpress TO data_analyst;

EOSQL

echo "✅ Schema file created"
```

#### 📝 Exercise 12: Prepare the ETL Python Script Skeleton

Create the skeleton for tomorrow's boto3 ETL script:

```bash
cd ~/projects/food-delivery-warehouse

mkdir -p aws

cat > aws/upload_to_s3.py << 'EOPY'
#!/usr/bin/env python3
"""
MakanExpress: Upload seed data to S3 data lake
Usage: python upload_to_s3.py --bucket makanexpress-datalake --prefix raw/makanexpress/
"""

import boto3
import os
import sys
import json
from pathlib import Path
from datetime import datetime


def get_s3_client():
    """Get S3 client using default credentials (from aws configure)."""
    return boto3.client('s3', region_name='ap-southeast-1')


def upload_file(s3_client, bucket, key, file_path):
    """Upload a single file to S3."""
    try:
        s3_client.upload_file(
            file_path, bucket, key,
            ExtraArgs={
                'ContentType': 'text/csv',
                'Metadata': {
                    'uploaded-by': 'makanexpress-etl',
                    'upload-date': datetime.now().isoformat()
                }
            }
        )
        print(f"  ✅ Uploaded: s3://{bucket}/{key}")
        return True
    except Exception as e:
        print(f"  ❌ Failed: {file_path} → {e}")
        return False


def upload_seed_data(bucket, prefix, seed_dir='seed'):
    """Upload all CSV files from seed directory to S3."""
    s3 = get_s3_client()
    
    # Verify bucket exists
    try:
        s3.head_bucket(Bucket=bucket)
        print(f"✅ Bucket exists: {bucket}")
    except Exception:
        print(f"❌ Bucket not found: {bucket}. Create it first.")
        sys.exit(1)
    
    # Find all CSV files
    seed_path = Path(seed_dir)
    csv_files = list(seed_path.glob('*.csv'))
    
    if not csv_files:
        print(f"❌ No CSV files found in {seed_dir}/")
        sys.exit(1)
    
    print(f"\n📤 Uploading {len(csv_files)} files to s3://{bucket}/{prefix}")
    print("-" * 50)
    
    success_count = 0
    for csv_file in csv_files:
        key = f"{prefix}{csv_file.name}"
        if upload_file(s3, bucket, key, str(csv_file)):
            success_count += 1
    
    print("-" * 50)
    print(f"✅ Uploaded {success_count}/{len(csv_files)} files")
    
    return success_count == len(csv_files)


if __name__ == '__main__':
    import argparse
    parser = argparse.ArgumentParser(description='Upload MakanExpress seed data to S3')
    parser.add_argument('--bucket', default='makanexpress-datalake')
    parser.add_argument('--prefix', default='raw/makanexpress/')
    parser.add_argument('--seed-dir', default='seed')
    args = parser.parse_args()
    
    upload_seed_data(args.bucket, args.prefix, args.seed_dir)
EOPY

chmod +x aws/upload_to_s3.py
echo "✅ Upload script skeleton created"
```

---

### Step 6: Write the Project Plan (30 min)

#### 📋 Tomorrow's Schedule (Day 42 — Thursday, 25 June)

| Time | Block | What You'll Do |
|------|-------|----------------|
| **Morning** (2h) | AWS Infrastructure | Create S3 bucket + prefixes, launch RDS PostgreSQL, set up security groups |
| **Afternoon** (2h) | Data Migration | Upload CSVs to S3, run schema migration on RDS, load data, verify |
| **Evening** (1h) | ETL Automation | Write boto3 scripts, test end-to-end, update README, push to GitHub |

#### 📋 Pre-Launch Checklist (fill in during planning)

```
AWS Account:
  [ ] Can log into AWS Console
  [ ] CLI configured: `aws sts get-caller-identity` works
  [ ] Region set to ap-southeast-1 (Singapore)
  [ ] Billing alerts set up (from Day 36)

Local Environment:
  [ ] Python 3.x installed
  [ ] boto3 installed: `pip install boto3`
  [ ] psycopg2 installed: `pip install psycopg2-binary`
  [ ] pandas installed: `pip install pandas`
  [ ] psql client available: `psql --version`

Project Files:
  [ ] food-delivery-warehouse repo exists locally
  [ ] seed/ directory has CSV files
  [ ] schema SQL files ready
  [ ] aws/ directory created

Cost Awareness:
  [ ] Everything is within Free Tier
  [ ] Will NOT create NAT Gateway ($36/month)
  [ ] Will delete RDS if not needed after project
  [ ] Estimated cost: $0 (within free tier)
```

#### 📝 Exercise 13: Verify Your Local Setup

Run these commands to verify you're ready:

```bash
# 1. AWS CLI
aws sts get-caller-identity
# Expected: Your account ID, user ARN

# 2. Python + libraries
python3 -c "import boto3; print('boto3:', boto3.__version__)"
python3 -c "import psycopg2; print('psycopg2: OK')"
python3 -c "import pandas; print('pandas:', pandas.__version__)"

# 3. psql client
psql --version

# 4. Project files
ls ~/projects/food-delivery-warehouse/seed/*.csv 2>/dev/null | wc -l
# Expected: 5 (customers, restaurants, menu_items, orders, order_items)

# 5. Disk space (need room for dumps)
df -h ~ | tail -1
```

<details>
<summary>🔑 What to do if something's missing</summary>

```bash
# Missing boto3
pip install boto3

# Missing psycopg2
pip install psycopg2-binary

# Missing pandas
pip install pandas

# Missing psql (Ubuntu/Debian)
sudo apt-get install postgresql-client

# Missing psql (macOS)
brew install libpq && brew link libpq

# Missing AWS CLI
# See Day 36 for installation instructions

# AWS CLI not configured
aws configure
# Enter your Access Key ID, Secret Access Key, region (ap-southeast-1), output (json)

# Project files missing
# You should have completed Day 28 (Food Delivery Warehouse) and Day 35 (Airflow)
# Check if the repo exists:
ls ~/projects/food-delivery-warehouse/
```

</details>

---

## 🌙 Evening Block (1 hour): Light Review + Rest

### Step 7: Quick Review of Key Concepts (20 min)

Flip through these in 2 minutes each — just remind yourself, don't deep-dive:

| Topic | Key Thing to Remember |
|-------|----------------------|
| S3 Buckets | Names are globally unique. Use lifecycle rules for cost savings. |
| S3 Objects | Max 5TB per object. Use multipart upload for >100MB. |
| S3 Data Lake | raw/ → processed/ → analytics/ (3-zone pattern) |
| IAM Users | For humans. Long-lived credentials. Enable MFA. |
| IAM Roles | For services. Temporary credentials. Auto-rotated. |
| IAM Policies | JSON. Explicit deny > explicit allow > default deny. |
| RDS | Managed database. Automated backups. Multi-AZ for prod. |
| Security Groups | Stateful firewalls. Allow by security group reference (not just IP). |
| VPC | Your private network. Private subnets for databases. |
| Free Tier | 12 months from account creation. Monitor with Billing Dashboard. |

### Step 8: Reflection & Rest (40 min)

#### 📝 Journal Entry

Write in your learning journal (`workspace-notes/journal.md` or any notes app):

```
Week 6 Reflection (Day 41):

1. My strongest AWS area so far: ___________________
2. My weakest AWS area: ___________________
3. What surprised me most about AWS: ___________________
4. What I'm most excited to build tomorrow: ___________________
5. One thing I want to review again before interviews: ___________________

AWS services I've hands-on with:
- [ ] S3 (create bucket, upload, download, lifecycle)
- [ ] IAM (create user, group, role, policy)
- [ ] RDS (launch PostgreSQL, connect, query)
- [ ] VPC (security groups, understand subnets)
```

#### 😴 Rest Before the Big Day

Tomorrow is the **most important day of Week 6** — you're deploying your entire project to the cloud. That's a real portfolio piece.

**Before you sleep:**
- Close unnecessary tabs
- Make sure your laptop is charged
- Have a good breakfast tomorrow ☕🥐
- You've got this! 💪

---

## ✅ Daily Checklist

- [ ] Identified my 2 weakest areas from Day 40 assessment
- [ ] Completed deep-dive exercises for weak spot #1
- [ ] Completed deep-dive exercises for weak spot #2
- [ ] Reviewed architecture diagram for tomorrow's project
- [ ] Resource checklist filled out
- [ ] Local setup verified (AWS CLI, Python, psql, project files)
- [ ] upload_to_s3.py skeleton created
- [ ] Schema SQL prepared (01_aws_schema.sql)
- [ ] Pre-launch checklist completed
- [ ] Wrote reflection / journal entry
- [ ] Ready for Day 42 portfolio project! 🚀

---

## 🗂️ Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│           DAY 41 — WEAK SPOT + PORTFOLIO PREP           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  IAM Policy Evaluation:                                 │
│  1. Default: DENY (no policy = no access)               │
│  2. Explicit Allow in any policy → ALLOW                 │
│  3. Explicit Deny in ANY policy → DENY (wins all)       │
│                                                         │
│  IAM User vs Role:                                      │
│  • User = human, long-lived keys, MFA                   │
│  • Role = service, temporary creds, auto-rotate         │
│  • EC2 → Instance Profile (role attached to instance)   │
│                                                         │
│  S3 Data Lake Zones:                                    │
│  • raw/ → CSV/JSON (source copy, immutable)             │
│  • processed/ → Parquet (cleaned, typed)                │
│  • analytics/ → Parquet (aggregated, business-ready)    │
│                                                         │
│  RDS Connection Troubleshooting:                        │
│  • Timeout → Security group / VPC issue                 │
│  • Auth failed → Wrong password                         │
│  • No pg_hba → IP not in allowed list                   │
│  • Connection refused → Instance not ready (wait 5min)  │
│  • DB not found → Missing --db-name parameter           │
│                                                         │
│  Free Tier Cost Traps:                                  │
│  • NAT Gateway: ~$36/month (NOT free!) ⚠️              │
│  • EBS snapshots after termination                      │
│  • Data transfer out (>1GB/month)                       │
│                                                         │
│  Tomorrow (Day 42): Deploy MakanExpress on AWS          │
│  • Morning: S3 + RDS infrastructure                     │
│  • Afternoon: Data migration + loading                  │
│  • Evening: ETL scripts + README + push                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

*Day 41 complete! Tomorrow: The big one — Deploy MakanExpress Data Warehouse on AWS! 🏗️☁️*
