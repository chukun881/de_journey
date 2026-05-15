# 📅 Day 39 — Monday, 22 June 2026
# 🌐 VPC Basics + Networking for Data Engineers

---

## 🎯 Today's Goal

VPC (Virtual Private Cloud) is AWS networking. You don't need to be a network engineer, but you need enough VPC knowledge to deploy databases securely, connect services, and answer interview questions like "how do you secure a data pipeline?"

**Philosophy:** Learn enough VPC to be dangerous (in a good way). You're a data engineer, not a network admin.

---

## ☀️ Morning Block (2 hours): VPC Concepts

### Concept 1: VPC Mental Model

```
VPC = Your private network in AWS
= Like having your own office building with rooms and security doors

AWS Account (Singapore Region)
└── VPC (your private network)
    ├── Public Subnet (accessible from internet)
    │   ├── NAT Gateway (lets private subnet reach internet)
    │   └── Bastion Host (jump server for SSH)
    ├── Private Subnet (NOT accessible from internet)
    │   ├── RDS Database (your PostgreSQL)
    │   └── Glue Jobs (your ETL)
    └── Security Groups (firewalls)
        ├── RDS Security Group: allow port 5432 from your IP only
        └── Glue Security Group: allow outbound to S3
```

### Concept 2: Key Networking Concepts

```
VPC: Virtual Private Cloud — your isolated network
CIDR: IP address range (e.g., 10.0.0.0/16 = 65,536 IPs)
Subnet: A subset of your VPC's IP range
    Public subnet: has route to internet gateway
    Private subnet: no direct internet access
Route Table: rules for where traffic goes
Internet Gateway (IGW): connects VPC to the internet
NAT Gateway: lets private subnet reach internet (for updates, API calls)
Security Group: virtual firewall for instances (stateful)
    = "Allow inbound port 5432 from 10.0.1.0/24"
Network ACL: subnet-level firewall (stateless, less common)

Default VPC: AWS creates one automatically in each region
    - Has public subnets, internet gateway, route tables
    - Good for learning, NOT for production
    - Your RDS instance is probably in the default VPC
```

### Concept 3: How Data Flows

```
Your Laptop → Internet → AWS Internet Gateway → VPC → Security Group → RDS

Step by step:
1. Your laptop connects to RDS endpoint via internet
2. DNS resolves to an IP address
3. Traffic hits the Internet Gateway
4. Route table directs to the right subnet
5. Security Group checks: "Is port 5432 allowed from this IP?"
6. If yes → RDS accepts connection
7. If no → connection refused

For a data pipeline:
Your ETL (in AWS) → Private Subnet → RDS (in same VPC)
    = No internet needed = more secure = faster
```

### Concept 4: Security Groups in Practice

```bash
# List your security groups
aws ec2 describe-security-groups --query "SecurityGroups[].{Name:GroupName,ID:GroupId}" --output table

# Create a security group for RDS
aws ec2 create-security-group \
    --group-name makanexpress-rds-sg \
    --description "MakanExpress RDS PostgreSQL access"

# Allow PostgreSQL from your IP
MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
    --group-name makanexpress-rds-sg \
    --protocol tcp \
    --port 5432 \
    --cidr ${MY_IP}/32

# Allow PostgreSQL from anywhere in the VPC (for Glue/Lambda)
aws ec2 authorize-security-group-ingress \
    --group-name makanexpress-rds-sg \
    --protocol tcp \
    --port 5432 \
    --cidr 10.0.0.0/16  # your VPC CIDR

# View rules
aws ec2 describe-security-groups \
    --group-names makanexpress-rds-sg \
    --query "SecurityGroups[].IpPermissions"
```

### Concept 5: What You Actually Need to Know for Interviews

```
Interview questions about VPC/networking for data engineers:

Q: "How would you secure a database on AWS?"
A: Put it in a private subnet, use security groups to restrict access,
   enable encryption at rest and in transit, use IAM authentication.

Q: "What's the difference between a security group and a network ACL?"
A: Security group = instance-level, stateful, allow rules only.
   Network ACL = subnet-level, stateless, allow AND deny rules.
   I'd use security groups for 95% of cases.

Q: "How do you connect your local machine to a private RDS instance?"
A: Use a bastion host (jump server) in the public subnet, or
   use AWS Systems Manager Session Manager, or
   use a VPN connection.

Q: "How does your ETL pipeline access S3 securely?"
A: Use an IAM role attached to the compute resource (Glue/Lambda/EC2).
   The role has permissions for S3. No credentials hardcoded.
   Traffic stays within AWS network (no internet).
```

---

### 🏋️ Morning Exercises (10 questions)

1. Find your default VPC and list its subnets using AWS CLI.

2. Create a security group for your RDS instance. Allow PostgreSQL (port 5432) from your IP only.

3. What is the CIDR range of your default VPC? How many IP addresses does it have?

4. Explain the difference between a public subnet and a private subnet. Which should RDS be in?

5. Create a security group for "data pipeline" access that allows:
   - Outbound: all traffic (for API calls, S3 access)
   - Inbound: PostgreSQL from VPC CIDR only

6. **Scenario:** Your RDS is in a private subnet. Your local machine can't connect. What are 3 ways to fix this?

7. List all security groups in your VPC. Which ones are being used by your RDS instance?

8. Create a diagram (text-based) showing your MakanExpress AWS architecture:
    ```
    Internet → IGW → Public Subnet → [Your Laptop]
                              → Private Subnet → [RDS PostgreSQL]
                                           → [S3 via VPC Endpoint]
    ```

9. What is a VPC Endpoint? Why would you use one for S3?

10. **Practical:** Apply your new security group to the RDS instance. Verify you can still connect from your IP.

---

### ✅ Morning Answers

<details>
<summary>🔑 Click to reveal answers</summary>

**Answer 1:**
```bash
aws ec2 describe-vpcs --query "Vpcs[].{ID:VpcId,CIDR:CidrBlock,Default:IsDefault}" --output table
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-XXXXX" --query "Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}" --output table
```

**Answer 4:** Public subnet has a route to the Internet Gateway (can be reached from the internet). Private subnet has no direct internet route. RDS should be in a private subnet for security — only accessible from within the VPC.

**Answer 6:** Three ways:
1. SSH tunnel through a bastion host in the public subnet
2. AWS Systems Manager Session Manager (no SSH needed)
3. VPN connection (AWS Client VPN or third-party)

**Answer 9:** VPC Endpoint lets resources in your VPC access AWS services (like S3) WITHOUT going through the internet. Traffic stays on the AWS network = faster + more secure. For S3, use a "Gateway Endpoint" (free).

</details>

---

## 🌤️ Afternoon Block (2 hours): RDS Deep Dive + Backup + Monitoring

### Concept 6: RDS Backups

```bash
# RDS has two backup types:

# 1. Automated backups (daily, retained 7 days by default)
aws rds describe-db-instances \
    --db-instance-identifier makanexpress-db \
    --query "DBInstances[0].{Backup:BackupRetentionPeriod,Window:PreferredBackupWindow}"

# Modify retention period
aws rds modify-db-instance \
    --db-instance-identifier makanexpress-db \
    --backup-retention-period 14 \
    --apply-immediately

# 2. Manual snapshots (on-demand)
aws rds create-db-snapshot \
    --db-instance-identifier makanexpress-db \
    --db-snapshot-identifier makanexpress-manual-snapshot

# List snapshots
aws rds describe-db-snapshots \
    --db-instance-identifier makanexpress-db

# Restore from snapshot (creates a new instance)
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier makanexpress-restored \
    --db-snapshot-identifier makanexpress-manual-snapshot

# Point-in-time restore (restore to any point in the retention period)
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier makanexpress-db \
    --target-db-instance-identifier makanexpress-pitr \
    --restore-time 2026-06-22T10:00:00Z
```

### Concept 7: RDS Monitoring

```bash
# Check RDS status
aws rds describe-db-instances \
    --db-instance-identifier makanexpress-db \
    --query "DBInstances[0].{Status:DBInstanceStatus,Storage:AllocatedStorage,Class:DBInstanceClass}"

# View CloudWatch metrics for RDS
aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=makanexpress-db \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average \
    --region ap-southeast-1

# Enable Performance Insights (free tier includes 7 days)
aws rds modify-db-instance \
    --db-instance-identifier makanexpress-db \
    --enable-performance-insights \
    --apply-immediately
```

### Concept 8: Connecting Airflow to RDS

```bash
# Add RDS connection to Airflow
airflow connections add makanexpress_rds \
    --conn-type postgres \
    --conn-host makanexpress-db.xxxxxx.ap-southeast-1.rds.amazonaws.com \
    --conn-schema makanexpress \
    --conn-login makanexpress_admin \
    --conn-password 'YourStrongPassword123!' \
    --conn-port 5432

# Test the connection
airflow connections test makanexpress_rds
```

### Concept 9: Cost Management

```bash
# Check your AWS spending
aws ce get-cost-and-usage \
    --time-period Start=2026-06-01,End=2026-06-30 \
    --granularity MONTHLY \
    --metrics BlendedCost \
    --group-by Type=DIMENSION,Key=SERVICE

# What to shut down when not in use:
# 1. RDS: stop the instance (saves compute cost, not storage)
aws rds stop-db-instance --db-instance-identifier makanexpress-db

# 2. RDS: start when needed
aws rds start-db-instance --db-instance-identifier makanexpress-db

# ⚠️ RDS auto-starts after 7 days of being stopped
# ⚠️ Stopped RDS still costs storage ($0.115/GB/month in Singapore)
```

---

### 🏋️ Afternoon Exercises (10 questions)

11. Create a manual snapshot of your RDS instance. Verify it appears in the snapshots list.

12. Stop your RDS instance. Verify it's stopped. Then start it again.

13. Check CloudWatch metrics for your RDS instance. What's the CPU utilization?

14. Modify your RDS backup retention to 14 days. Verify the change.

15. Connect Airflow to your RDS instance. Run a test query using PostgresHook.

16. **Cost exercise:** Estimate your monthly AWS costs after the free tier expires:
    - RDS db.t3.micro (20GB): ~$XX/month
    - S3 (5GB Standard): ~$XX/month
    - Data transfer: ~$XX/month
    - Total: ~$XX/month

17. Write a Python script that monitors your RDS instance:
    - Checks if it's running
    - Shows storage used
    - Shows CPU utilization from CloudWatch
    - Alerts if CPU > 80%

18. **Scenario:** You accidentally dropped a table in RDS. How do you recover it? List the steps.

19. Write a script that creates a daily RDS snapshot and cleans up snapshots older than 7 days.

20. **Practical:** Document your complete AWS setup:
    - VPC and subnets
    - Security groups and their rules
    - RDS instance details
    - S3 bucket configuration
    - IAM users, groups, and roles
    - Estimated monthly costs
    Save as `aws-infrastructure.md` in your project.

---

### ✅ Selected Answers

<details>
<summary>🔑 Click to reveal selected answers</summary>

**Answer 16:** Cost estimates (Singapore region, approximate):
- RDS db.t3.micro (20GB): ~$18/month
- S3 (5GB Standard): ~$0.13/month
- Data transfer (first 1GB free, then $0.114/GB): ~$1/month
- **Total: ~$19/month** (very affordable for a portfolio project)

**Answer 17:**
```python
import boto3

def monitor_rds(instance_id):
    rds = boto3.client('rds', region_name='ap-southeast-1')
    cw = boto3.client('cloudwatch', region_name='ap-southeast-1')
    
    # Check status
    response = rds.describe_db_instances(DBInstanceIdentifier=instance_id)
    db = response['DBInstances'][0]
    
    print(f"📊 RDS Status: {db['DBInstanceStatus']}")
    print(f"   Storage: {db['AllocatedStorage']} GB")
    print(f"   Class: {db['DBInstanceClass']}")
    
    # CPU utilization (last hour)
    import datetime
    end = datetime.datetime.utcnow()
    start = end - datetime.timedelta(hours=1)
    
    cpu = cw.get_metric_statistics(
        Namespace='AWS/RDS',
        MetricName='CPUUtilization',
        Dimensions=[{'Name': 'DBInstanceIdentifier', 'Value': instance_id}],
        StartTime=start, EndTime=end,
        Period=300, Statistics=['Average']
    )
    
    if cpu['Datapoints']:
        avg_cpu = sum(d['Average'] for d in cpu['Datapoints']) / len(cpu['Datapoints'])
        print(f"   CPU (1h avg): {avg_cpu:.1f}%")
        if avg_cpu > 80:
            print("   ⚠️ HIGH CPU ALERT!")
    else:
        print("   CPU: No data (instance may be stopped)")

monitor_rds('makanexpress-db')
```

**Answer 18:** Recovery options:
1. **Point-in-time restore**: Restore to a time before the DROP TABLE
2. **Manual snapshot**: Restore from the most recent snapshot
3. **pg_dump**: If you have a local backup, re-import just that table

</details>

---

## 🌙 Evening: Review

### VPC Minimum Knowledge for Data Engineers

```
✅ Know: VPC, subnet (public/private), security group, route table
✅ Know: RDS should be in private subnet
✅ Know: Security groups = firewall rules
✅ Know: IAM roles for service access (no hardcoded credentials)
✅ Know: VPC endpoints for S3 (no internet needed)
❌ Don't need: BGP, VPN, Direct Connect, Transit Gateway
❌ Don't need: Custom route tables, Network ACLs
```

### 📝 Today's Checklist

- [ ] Understand VPC, subnets, and security groups
- [ ] Created security groups for RDS access
- [ ] RDS backups configured (automated + manual snapshot)
- [ ] Can monitor RDS via CloudWatch
- [ ] Airflow connected to RDS
- [ ] Cost management understood
- [ ] aws-infrastructure.md documentation written
- [ ] Ready for tomorrow: Assessment + Portfolio Project

---

*Day 39 complete! Tomorrow: Assessment + Portfolio Project — deploy everything on AWS!* 🏗️
