# 🛠️ Build Log — ATS CV Generator

A step-by-step record of every resource created, exactly as configured, plus every real issue hit along the way.

## 📋 Quick Navigation

| Step | Section |
|---|---|
| 1 | [🔑 Key Pair](#step-1--key-pair) |
| 2 | [🌐 VPC](#step-2--vpc) |
| 3 | [🧩 Subnets](#step-3--subnets) |
| 4 | [🚪 Internet Gateway](#step-4--internet-gateway) |
| 5 | [🗺️ Route Table](#step-5--route-table) |
| 6 | [🛡️ Security Groups](#step-6--security-groups) |
| 7 | [🖥️ EC2 Instances](#step-7--ec2-instances) |
| 8 | [⚖️ Target Group & Load Balancer](#step-8--target-group--load-balancer) |
| 9 | [🪣 S3 Bucket](#step-9--s3-bucket) |
| 10 | [🗃️ DynamoDB Table](#step-10--dynamodb-table) |
| 11 | [🔐 IAM Role & Policy](#step-11--iam-role--policy) |
| 12 | [⚡ Lambda: CV Generator](#step-12--lambda-cv-generator) |
| 13 | [⚡ Lambda: JD Analyzer](#step-13--lambda-jd-analyzer) |
| 14 | [🔌 API Gateway](#step-14--api-gateway) |
| 15 | [🚀 Flask Deployment](#step-15--flask-deployment) |
| 16 | [✅ End-to-End Test](#step-16--end-to-end-test) |

---

## Step 1 — 🔑 Key Pair

| Setting | Value |
|---|---|
| Name | `ats-keypair` |
| Type | RSA |
| Format | .pem |

![Key pair](screenshots/01-keypair.png)

---

## Step 2 — 🌐 VPC

| Setting | Value |
|---|---|
| Name | `ats-cv-vpc` |
| IPv4 CIDR | `10.0.0.0/16` |

![VPC](screenshots/02-vpc.png)

---

## Step 3 — 🧩 Subnets

| Name | AZ | CIDR | Auto-assign public IPv4 |
|---|---|---|---|
| `ats-public-subnet-1` | us-east-1a | 10.0.1.0/24 | ✅ |
| `ats-public-subnet-2` | us-east-1b | 10.0.2.0/24 | ✅ |

![Subnets](screenshots/03-subnets.png)

---

## Step 4 — 🚪 Internet Gateway

| Setting | Value |
|---|---|
| Name | `ats-igw` |
| Attached VPC | `ats-cv-vpc` |
| State | Attached |

![Internet gateway](screenshots/04-igw.png)

---

## Step 5 — 🗺️ Route Table

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | `ats-igw` |

Associated with both public subnets.

![Route table](screenshots/05-route-table.png)
![Route table associations](screenshots/05-route-table-associations.png)

---

## Step 6 — 🛡️ Security Groups

**ALB security group** — `ats-alb-sg`

| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | 0.0.0.0/0 |

![ALB security group](screenshots/06-sg-alb.png)

**EC2 security group** — `ats-ec2-sg`

| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | `ats-alb-sg` |
| Inbound | SSH | 22 | Restricted — [see Step 15](#step-15--flask-deployment) |

![EC2 security group](screenshots/06-sg-ec2.png)

---

## Step 7 — 🖥️ EC2 Instances

| Setting | Instance 1 | Instance 2 |
|---|---|---|
| Name | ats-ec2-1 | ats-ec2-2 |
| Type | t3.micro | t3.micro |
| Subnet | ats-public-subnet-1 | ats-public-subnet-2 |
| Security group | ats-ec2-sg | ats-ec2-sg |
| Key pair | ats-keypair | ats-keypair |

User data (both instances):
```bash
#!/bin/bash
yum update -y
yum install -y python3 python3-pip
pip3 install flask requests
mkdir -p /home/ec2-user/ats-app/templates
```

> ⚠️ **Issue:** `t2.micro` is not Free Tier–eligible on new AWS accounts (post-July 2025 Free Plan). Used **t3.micro** instead.

![EC2 instances running](screenshots/07-ec2.png)

---

## Step 8 — ⚖️ Target Group & Load Balancer

| Setting | Value |
|---|---|
| Target group | `ats-target-group` — HTTP:80, health check `/health` |
| Load balancer | `ats-alb` — Internet-facing |
| Listener | HTTP:80 → `ats-target-group` |

![Load balancer](screenshots/08-alb.png)

---

## Step 9 — 🪣 S3 Bucket

| Setting | Value |
|---|---|
| Name | `ats-cv-storage-<account-id>` |
| Region | us-east-1 |
| Public access | Blocked |

![S3 bucket](screenshots/09-s3.png)

---

## Step 10 — 🗃️ DynamoDB Table

| Setting | Value |
|---|---|
| Name | `ats-cv-records` |
| Partition key | `cv_id` (String) |
| Capacity mode | On-demand |

![DynamoDB table](screenshots/10-dynamodb.png)
![DynamoDB empty items](screenshots/10-dynamodb-items.png)

---

## Step 11 — 🔐 IAM Role & Policy

| Setting | Value |
|---|---|
| Policy | `ats-lambda-policy` |
| Role | `ats-lambda-role` |
| Principle | Least privilege — no wildcard resources |

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": ["s3:PutObject", "s3:GetObject"],
            "Resource": "arn:aws:s3:::ats-cv-storage-*/*"
        },
        {
            "Effect": "Allow",
            "Action": ["dynamodb:PutItem", "dynamodb:GetItem", "dynamodb:UpdateItem"],
            "Resource": "arn:aws:dynamodb:*:*:table/ats-cv-records"
        }
    ]
}
```

![IAM policy](screenshots/11-iam-policy.png)
![IAM role](screenshots/11-iam-role.png)

---

## Step 12 — ⚡ Lambda: CV Generator

| Setting | Value |
|---|---|
| Name | `ats-cv-generator` |
| Runtime | Python 3.12 |
| Role | `ats-lambda-role` |
| Env vars | `BUCKET_NAME`, `TABLE_NAME` |
| Timeout / Memory | 30s / 256MB |

![Lambda generator config](screenshots/12-lambda-generator.png)

---

## Step 13 — ⚡ Lambda: JD Analyzer

| Setting | Value |
|---|---|
| Name | `ats-jd-analyzer` |
| Runtime | Python 3.12 |
| Role | `ats-lambda-role` |
| Env var | `TABLE_NAME` |

![Lambda analyzer code](screenshots/13-lambda-analyzer-code.png)
![Lambda analyzer config](screenshots/13-lambda-analyzer.png)

---

## Step 14 — 🔌 API Gateway

| Setting | Value |
|---|---|
| API name | `ats-cv-api` (REST, Regional) |
| Resources | `/generate` → `ats-cv-generator`, `/analyze` → `ats-jd-analyzer` |
| Integration | Lambda proxy + CORS enabled |
| Stage | `prod` |

![API Gateway resources](screenshots/14-apigateway-resources.png)
![API Gateway invoke URL](screenshots/14-apigateway-invoke.png)

---

## Step 15 — 🚀 Flask Deployment

Deployed to both EC2 instances via EC2 Instance Connect, running as a `systemd` service pointing to the API Gateway Invoke URL.

### 🐛 Issues Hit & Fixed

| Issue | Root Cause | Fix |
|---|---|---|
| SSH connection failed via EC2 Instance Connect despite correct "My IP" | Browser-based connection is proxied through AWS's own infrastructure — real source IP is AWS's `EC2_INSTANCE_CONNECT` service range, not the client's | Allowed `18.206.107.24/29` (us-east-1 service range) in `ats-ec2-sg` |
| Flask systemd service crash-looping with generic `exit-code` | `systemctl status` doesn't show the actual Python traceback | Ran `python3 app.py` directly to expose and fix the real error |

![Systemd service running](screenshots/15-systemd.png)

---

## Step 16 — ✅ End-to-End Test

1. Opened the ALB DNS name in the browser.
2. Filled in the form with sample details and generated a CV — got a success message with a download link.
3. Pasted a job description ("Cloud Security Operations") and checked the match score.

![CV generation form filled and submitted](screenshots/16-cv-form-filled.png)
![JD match result — score and missing keywords](screenshots/16-fulltest.png)

| Check | Result |
|---|---|
| Target group health | ✅ Healthy |
| ALB DNS reachable | ✅ |
| CV generation | ✅ Successful |
| JD match analysis | ✅ Returned score + missing keywords |

**🧹 Cleanup:** terminated both EC2 instances, deleted the ALB and target group immediately after testing to avoid ongoing charges. Lambda, S3, DynamoDB, and API Gateway left running (near-zero idle cost).

---

<div align="center">

**[⬆ Back to top](#-build-log--ats-cv-generator)**

</div>
