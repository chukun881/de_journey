# 📅 Day 40 — Tuesday, 23 June 2026
# 📝 Week 6 Assessment: AWS Core Review

---

## 🎯 Today's Goal

Timed assessment covering Week 6 (AWS Core). This tests your understanding of S3, IAM, RDS, and VPC — the foundation of cloud data engineering.

---

## ☀️ Morning Block (2 hours): Timed Assessment

### Section A: AWS Fundamentals (15 minutes, 5 questions)

**Q1.** What is the difference between an AWS Region and an Availability Zone? Why does it matter for RDS?

**Q2.** List 5 AWS services a data engineer commonly uses and explain each in one sentence.

**Q3.** What is the AWS Free Tier? Name 3 services that are free for 12 months and their limits.

**Q4.** Explain the principle of "least privilege" in IAM. Give a data engineering example.

**Q5.** What is an ARN? Show the ARN for an S3 bucket and an RDS instance.

### Section B: S3 (20 minutes, 5 questions)

**Q6.** Write AWS CLI commands to: create a bucket, upload a CSV file, list bucket contents, download a file, delete the file.

**Q7.** Write a Python function using boto3 that reads a Parquet file from S3 into a Pandas DataFrame.

**Q8.** Explain the 3-zone data lake pattern (raw/processed/analytics). What format should each zone use and why?

**Q9.** Design an S3 lifecycle policy: raw data → Standard-IA after 30 days → Glacier after 90 days → delete after 1 year.

**Q10.** What is the difference between CSV and Parquet? Why is Parquet preferred for analytics?

### Section C: IAM (15 minutes, 5 questions)

**Q11.** Write a JSON IAM policy that allows reading and writing to one specific S3 bucket but denies deleting any objects.

**Q12.** What is the difference between an IAM user and an IAM role? When would you use each in a data pipeline?

**Q13.** Create an IAM role that AWS Glue can assume to read/write S3. Show the trust policy and the permissions policy.

**Q14.** Two policies are attached to a user: one allows `s3:*` and another denies `s3:DeleteBucket`. Can the user delete objects? Can they delete the bucket?

**Q15.** Design the IAM setup for a 3-person data team: 2 data engineers and 1 data scientist. Specify groups, policies, and access levels.

### Section D: RDS & Networking (15 minutes, 5 questions)

**Q16.** Write the CLI command to create a PostgreSQL RDS instance in Singapore. Include all required parameters.

**Q17.** How do you migrate a local PostgreSQL database to RDS? Show the exact commands.

**Q18.** What is a VPC security group? Write rules that allow PostgreSQL access from a specific IP and from within the VPC.

**Q19.** Your RDS instance is in a private subnet. List 3 ways to connect from your local machine.

**Q20.** A data pipeline needs to access both S3 and RDS. How do you securely provide credentials without hardcoding them?

---

## 🌤️ Afternoon Block (2 hours): Answers & Weak Spot Review

### Scoring

| Section | Questions | Points |
|---------|-----------|--------|
| A: Fundamentals | 5 × 4 | 20 |
| B: S3 | 5 × 8 | 40 |
| C: IAM | 5 × 6 | 30 |
| D: RDS & Networking | 5 × 2 | 10 |
| **Total** | **20** | **100** |

---

### ✅ Full Answers

<details>
<summary>🔑 Section A: Fundamentals</summary>

**A1:** Region = physical geographic area (Singapore). AZ = data center within a region (ap-southeast-1a). For RDS, Multi-AZ deployment replicates data to another AZ for high availability. If one AZ fails, RDS fails over automatically.

**A2:**
1. S3 — unlimited file storage for data lakes
2. RDS — managed relational database (PostgreSQL)
3. Glue — managed ETL service (Spark-based)
4. Athena — query S3 data with SQL (serverless)
5. IAM — identity and access management (security)

**A3:**
- S3: 5GB storage, 20K GET, 2K PUT requests/month
- RDS: 750 hours of db.t3.micro/month (1 DB running 24/7)
- EC2: 750 hours of t2.micro/month

**A4:** Least privilege = grant only the permissions needed for the job. Example: A Glue ETL job needs to read from `raw/` and write to `processed/` on one S3 bucket. Don't give it full S3 access or access to other buckets.

**A5:**
- S3 bucket: `arn:aws:s3:::makanexpress-data-lake-dev`
- RDS instance: `arn:aws:rds:ap-southeast-1:123456789012:db:makanexpress-db`

</details>

<details>
<summary>🔑 Section B: S3</summary>

**B1 (Q6):**
```bash
aws s3 mb s3://my-bucket --region ap-southeast-1
aws s3 cp data.csv s3://my-bucket/raw/data.csv
aws s3 ls s3://my-bucket/raw/
aws s3 cp s3://my-bucket/raw/data.csv ./downloaded.csv
aws s3 rm s3://my-bucket/raw/data.csv
```

**B2 (Q7):**
```python
import boto3
import pandas as pd
from io import BytesIO

def read_parquet_from_s3(bucket, key):
    s3 = boto3.client('s3')
    response = s3.get_object(Bucket=bucket, Key=key)
    return pd.read_parquet(BytesIO(response['Body'].read()))
```

**B3 (Q8):**
- Raw: CSV/JSON — human-readable, debuggable, exact source copy
- Processed: Parquet — compressed, typed, fast for queries
- Analytics: Parquet — pre-aggregated, business-ready

**B4 (Q9):**
```python
lifecycle = {
    "Rules": [{
        "ID": "RawDataLifecycle",
        "Status": "Enabled",
        "Filter": {"Prefix": "raw/"},
        "Transitions": [
            {"Days": 30, "StorageClass": "STANDARD_IA"},
            {"Days": 90, "StorageClass": "GLACIER"}
        ],
        "Expiration": {"Days": 365}
    }]
}
```

**B5 (Q10):**
CSV: text-based, human-readable, large files, no schema, slow queries
Parquet: columnar, compressed (50-80% smaller), schema embedded, fast columnar reads
Parquet wins because: smaller files, faster queries (read only needed columns), type safety

</details>

<details>
<summary>🔑 Section C: IAM</summary>

**C1 (Q11):**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
            "Resource": [
                "arn:aws:s3:::makanexpress-data-lake-dev",
                "arn:aws:s3:::makanexpress-data-lake-dev/*"
            ]
        },
        {
            "Effect": "Deny",
            "Action": ["s3:DeleteObject", "s3:DeleteObjectVersion", "s3:DeleteBucket"],
            "Resource": [
                "arn:aws:s3:::makanexpress-data-lake-dev",
                "arn:aws:s3:::makanexpress-data-lake-dev/*"
            ]
        }
    ]
}
```

**C2 (Q12):**
- User: permanent identity with long-lived credentials (access keys). Use for: developers, CI/CD systems.
- Role: temporary identity assumed by users or services with auto-rotated credentials. Use for: AWS services (Glue, Lambda, EC2), cross-account access.

**C3 (Q13):**
Trust policy:
```json
{"Version": "2012-10-17", "Statement": [{"Effect": "Allow", "Principal": {"Service": "glue.amazonaws.com"}, "Action": "sts:AssumeRole"}]}
```
Permissions: AWSGlueServiceRole managed policy + custom S3 read/write policy for the data lake bucket.

**C4 (Q14):** Can delete objects: Yes (s3:* includes s3:DeleteObject, no deny on that). Can delete the bucket: No (explicit deny on s3:DeleteBucket wins).

**C5 (Q15):**
- Group "DataEngineers": S3 full access to data lake, Glue, Athena, RDS read/write
- Group "DataScientists": S3 read-only to processed/analytics, Athena, RDS read-only
- de-alice, de-bob → DataEngineers; ds-charlie → DataScientists

</details>

<details>
<summary>🔑 Section D: RDS & Networking</summary>

**D1 (Q16):**
```bash
aws rds create-db-instance \
    --db-instance-identifier makanexpress-db \
    --db-instance-class db.t3.micro \
    --engine postgres \
    --master-username admin \
    --master-user-password 'Password123!' \
    --allocated-storage 20 \
    --db-name makanexpress \
    --region ap-southeast-1
```

**D2 (Q17):**
```bash
pg_dump -h localhost -U postgres makanexpress > dump.sql
psql -h RDS_ENDPOINT -U admin -d makanexpress < dump.sql
```

**D3 (Q18):**
```bash
# Allow from specific IP
aws ec2 authorize-security-group-ingress --group-id sg-xxx \
    --protocol tcp --port 5432 --cidr 203.0.113.50/32

# Allow from VPC
aws ec2 authorize-security-group-ingress --group-id sg-xxx \
    --protocol tcp --port 5432 --cidr 10.0.0.0/16
```

**D4 (Q19):**
1. Bastion host (jump server) in public subnet
2. AWS Systems Manager Session Manager
3. VPN connection (AWS Client VPN)

**D5 (Q20):**
Use IAM roles. For Glue: attach a role with S3 + RDS permissions. For EC2: attach an instance profile. The role provides temporary credentials automatically — no hardcoded passwords. Use environment variables or AWS Secrets Manager for database passwords.

</details>

---

## 🌙 Evening: Targeted Review + Portfolio Prep

### Weak Spot Guide

| Score < 70% in | Action |
|----------------|--------|
| Section A | Re-read Day 36 morning |
| Section B | Re-read Day 36-37, practice S3 CLI + boto3 |
| Section C | Re-read Day 38 morning, practice writing IAM policies |
| Section D | Re-read Day 38-39, practice RDS setup |

### 📝 Today's Checklist

- [ ] Completed assessment honestly
- [ ] Identified weak spots
- [ ] Reviewed at least 2 weak areas
- [ ] Ready for Day 41-42 portfolio project
- [ ] aws-setup.md and aws-infrastructure.md updated

---

*Day 40 complete! Days 41-42: Portfolio Project — Deploy MakanExpress on AWS.* 🏗️☁️
