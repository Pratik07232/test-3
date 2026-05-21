# 🚀 AWS Multi-Region Disaster Recovery Setup

> **Fresher DevOps Project** | Built by Pratik | AWS Certified Cloud Practitioner (CLF-C02)

A complete AWS Disaster Recovery (DR) architecture using **Pilot Light / Warm Standby** strategy across two regions — with automatic DNS failover via Route 53.

---

## 📌 Project Overview

| | Primary Region | DR Region |
|---|---|---|
| **Region** | eu-west-1 | eu-west-2 |
| **Location** | 🇮🇪 Ireland | 🇬🇧 London |
| **Status** | Always active | Standby (activates on failover) |

If the primary region becomes unavailable, **Route 53 automatically redirects traffic** to the DR region within minutes.

---

## 🏗️ Architecture Diagram

![AWS Multi-Region DR Architecture](architecture/dr-architecture.svg)

**Traffic flow (normal operation):**
```
Users → Route 53 (primary) → ALB (eu-west-1) → ASG/EC2 → RDS Primary
```

**Traffic flow (failover active):**
```
Users → Route 53 (failover) → ALB (eu-west-2) → ASG/EC2 → RDS Replica (promoted)
```

---

## 🛠️ AWS Services Used

| Service | Purpose |
|---|---|
| **EC2** | Web server instances (Apache httpd) |
| **Auto Scaling Group** | Auto-manages EC2 capacity |
| **Application Load Balancer** | Distributes traffic across EC2 |
| **Route 53** | DNS-based failover routing |
| **CloudWatch** | Health checks and alarm-based alerting |
| **RDS** | Primary DB + Read Replica (cross-region) |
| **S3** | Object storage with Cross-Region Replication |
| **IAM** | Least-privilege roles for all services |

---

## ⚙️ Step-by-Step Configuration

---

### Step 1 — Launch EC2 Instances

> Created EC2 instances in **both regions** and installed Apache web server.

**Primary region (eu-west-1):**
```bash
# Connect to EC2 and run:
sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd

# Verify Apache is running
sudo systemctl status httpd
curl http://localhost
```

**DR region (eu-west-2):**
```bash
# Same steps — keep instances in stopped state to save cost (Pilot Light strategy)
sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
```

**Security Group rules:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Your IP |
| HTTP | TCP | 80 | 0.0.0.0/0 |
| HTTPS | TCP | 443 | 0.0.0.0/0 |

---

### Step 2 — Create Auto Scaling Groups (ASG)

> ASG ensures the correct number of EC2 instances are always running.

**Launch Template settings:**
```
AMI:           Amazon Linux 2023
Instance Type: t2.micro (free tier)
Key Pair:      your-key.pem
Security Group: web-sg (port 80, 443, 22)
User Data:
  #!/bin/bash
  yum install -y httpd
  systemctl start httpd
  systemctl enable httpd
  echo "<h1>Primary Region - eu-west-1</h1>" > /var/www/html/index.html
```

**ASG Capacity settings:**

| Setting | Primary | DR |
|---|---|---|
| Minimum | 2 | 1 |
| Desired | 2 | 1 |
| Maximum | 6 | 6 |

**Scaling Policy:** Target tracking — CPU utilization at 70%

---

### Step 3 — Set Up Application Load Balancer (ALB)

> ALB distributes incoming HTTP/HTTPS traffic to healthy EC2 instances.

**Steps:**
1. Navigate to EC2 → Load Balancers → Create Load Balancer
2. Choose **Application Load Balancer**
3. Scheme: **Internet-facing**
4. Listeners: HTTP (80) and HTTPS (443)
5. Create **Target Group** → register EC2 instances from ASG

**Health Check settings:**
```
Protocol:           HTTP
Path:               /
Healthy threshold:  2
Unhealthy threshold: 3
Timeout:            5 seconds
Interval:           30 seconds
```

> ✅ Repeat the exact same steps in **eu-west-2** for the DR ALB.

---

### Step 4 — Configure RDS with Cross-Region Read Replica

> RDS Primary lives in eu-west-1. A Read Replica is created in eu-west-2 and promoted during failover.

**Primary RDS (eu-west-1):**
```
Engine:         MySQL 8.0 / PostgreSQL 15
Instance Class: db.t3.micro
Storage:        20 GB gp2
Multi-AZ:       Yes (for HA within primary region)
Backup Retention: 7 days
```

**Create Read Replica in eu-west-2:**
1. Go to RDS Console → select Primary DB
2. Actions → **Create Read Replica**
3. Destination Region: **eu-west-2**
4. Instance Class: db.t3.micro
5. Leave as Read Replica (do NOT promote yet)

**Promoting the Replica on failover:**
```bash
# Only run this during a DR event!
aws rds promote-read-replica \
  --db-instance-identifier your-dr-replica-id \
  --region eu-west-2
```

---

### Step 5 — Configure S3 Cross-Region Replication (CRR)

> S3 objects are automatically replicated from eu-west-1 to eu-west-2.

**Primary bucket setup (eu-west-1):**
```
Bucket Name:  my-app-primary-bucket
Versioning:   ENABLED  ← required for CRR
Region:       eu-west-1
```

**DR bucket setup (eu-west-2):**
```
Bucket Name:  my-app-dr-bucket
Versioning:   ENABLED
Region:       eu-west-2
```

**Replication rule (on primary bucket):**
1. S3 → Primary bucket → Management → Replication rules
2. Source: All objects
3. Destination: `my-app-dr-bucket` (eu-west-2)
4. IAM Role: Create new (S3 will auto-generate permissions)
5. Enable: **Replication Time Control (RTC)** for 15-minute SLA (optional)

---

### Step 6 — Set Up Route 53 Failover Routing

> Route 53 monitors the primary ALB and automatically switches DNS to DR on failure.

**Create Health Check:**
```
Monitor:        Endpoint
Protocol:       HTTP
Domain/IP:      primary-alb-dns-name.eu-west-1.elb.amazonaws.com
Path:           /
Check Interval: 30 seconds
Failure Threshold: 3 consecutive failures
```

**DNS Records:**

| Record | Type | Routing Policy | Value |
|---|---|---|---|
| `app.yourdomain.com` | A (Alias) | Failover — PRIMARY | Primary ALB DNS |
| `app.yourdomain.com` | A (Alias) | Failover — SECONDARY | DR ALB DNS |

> ✅ Attach the Health Check to the PRIMARY record only.

---

### Step 7 — Configure CloudWatch Alarms

> CloudWatch monitors health of both regions and sends alerts via SNS.

**Key alarms to create:**

```bash
# High CPU on EC2
aws cloudwatch put-metric-alarm \
  --alarm-name "High-CPU-Primary" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:eu-west-1:ACCOUNT_ID:dr-alerts

# RDS Low Storage
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-Low-Storage" \
  --metric-name FreeStorageSpace \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 5000000000 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:eu-west-1:ACCOUNT_ID:dr-alerts
```

**SNS Topic for alerts:**
```bash
aws sns create-topic --name dr-alerts --region eu-west-1
aws sns subscribe \
  --topic-arn arn:aws:sns:eu-west-1:ACCOUNT_ID:dr-alerts \
  --protocol email \
  --notification-endpoint your-email@example.com
```

---

### Step 8 — IAM Roles & Permissions

> All services use IAM roles with least-privilege access.

**Roles created:**

| Role Name | Attached To | Purpose |
|---|---|---|
| `ec2-web-role` | EC2 Instances | S3 read, CloudWatch logs |
| `s3-replication-role` | S3 CRR | Replicate objects across regions |
| `rds-monitoring-role` | RDS | Enhanced monitoring |
| `lambda-failover-role` | Lambda (optional) | Automate failover steps |

**EC2 Instance Profile policy (example):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "logs:CreateLogGroup",
        "logs:PutLogEvents",
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 🔁 Failover Runbook

When the primary region goes down:

```
1. CloudWatch alarm fires → SNS email alert sent
2. Route 53 health check detects ALB failure (3 consecutive)
3. Route 53 automatically switches DNS to DR ALB (eu-west-2)
4. Manually promote RDS Read Replica → standalone DB
5. Update app config to point to new RDS endpoint
6. Verify S3 DR bucket has latest objects
7. Scale up DR ASG if needed
```

**RTO (Recovery Time Objective):** ~15–30 minutes
**RPO (Recovery Point Objective):** ~5 minutes (RDS async replication lag)

---

## 📁 Repository Structure

```
aws-multi-region-dr/
│
├── README.md                        ← You are here
│
├── architecture/
│   └── dr-architecture.svg          ← Full architecture diagram
│
├── scripts/
│   ├── install-apache.sh            ← EC2 bootstrap script
│   ├── promote-rds-replica.sh       ← Failover: promote RDS replica
│   ├── create-cloudwatch-alarms.sh  ← Create all CloudWatch alarms
│   └── test-failover.sh             ← Simulate and test failover
│
├── terraform/
│   ├── main.tf                      ← Root module
│   ├── variables.tf                 ← Input variables
│   ├── outputs.tf                   ← Output values
│   ├── ec2.tf                       ← EC2 + ASG + ALB
│   ├── rds.tf                       ← RDS primary + replica
│   ├── s3.tf                        ← S3 buckets + CRR
│   ├── route53.tf                   ← DNS + health checks
│   └── cloudwatch.tf                ← Alarms + SNS
│
├── docs/
│   ├── FAILOVER-RUNBOOK.md          ← Step-by-step failover guide
│   ├── COST-ESTIMATE.md             ← AWS cost breakdown
│   └── TESTING.md                   ← How to test DR
│
└── screenshots/
    └── (add your AWS Console screenshots here)
```

---

## 💰 Cost Estimate (Approximate)

| Service | Primary | DR (Standby) |
|---|---|---|
| EC2 t2.micro × 2 | ~$17/month | ~$0 (stopped) |
| ALB | ~$16/month | ~$16/month |
| RDS db.t3.micro | ~$15/month | ~$12/month (replica) |
| S3 (10 GB) | ~$0.23/month | ~$0.23/month |
| Route 53 | ~$0.50/month | — |
| CloudWatch | ~$3/month | ~$1/month |
| **Total** | **~$52/month** | **~$29/month** |

> 💡 Tip: Use **Free Tier** for the first 12 months to reduce cost significantly.

---

## ✅ Skills Demonstrated

- ☁️ AWS Multi-Region architecture design
- 🔁 Disaster Recovery (DR) with Pilot Light / Warm Standby
- ⚖️ Load Balancing with ALB + Auto Scaling
- 🗄️ RDS cross-region Read Replica + promotion
- 🪣 S3 Cross-Region Replication (CRR)
- 🌐 Route 53 DNS failover routing + health checks
- 📊 CloudWatch alarms + SNS notifications
- 🔐 IAM least-privilege security model

---

## 👨‍💻 About

**Pratik** | B.Sc. Computer Science, Savitribai Phule Pune University
AWS Certified Cloud Practitioner (CLF-C02) | Aspiring Cloud/DevOps Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue)](https://linkedin.com)
[![AWS Certified](https://img.shields.io/badge/AWS-Cloud_Practitioner-orange)](https://aws.amazon.com/certification/)
