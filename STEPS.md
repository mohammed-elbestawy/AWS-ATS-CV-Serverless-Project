<a id="top"></a>

# 🛠️ Build Log — ATS CV Generator

A step-by-step record of every resource created, exactly as configured, plus every real issue hit along the way.

## 📋 Quick Navigation

| Step | Section |
|---|---|
| 1 | [🔑 Key Pair](#step-1) |
| 2 | [🌐 VPC](#step-2) |
| 3 | [🧩 Subnets](#step-3) |
| 4 | [🚪 Internet Gateway](#step-4) |
| 5 | [🗺️ Route Table](#step-5) |
| 6 | [🛡️ Security Groups](#step-6) |
| 7 | [🖥️ EC2 Instances](#step-7) |
| 8 | [⚖️ Target Group & Load Balancer](#step-8) |
| 9 | [🪣 S3 Bucket](#step-9) |
| 10 | [🗃️ DynamoDB Table](#step-10) |
| 11 | [🔐 IAM Role & Policy](#step-11) |
| 12 | [⚡ Lambda: CV Generator](#step-12) |
| 13 | [⚡ Lambda: JD Analyzer](#step-13) |
| 14 | [🔌 API Gateway](#step-14) |
| 15 | [🚀 Flask Deployment](#step-15) |
| 16 | [✅ End-to-End Test](#step-16) |

---

<a id="step-1"></a>
## Step 1 — 🔑 Key Pair

This key is the only way to log into the EC2 servers securely, instead of a password.

| Setting | Value |
|---|---|
| Name | `ats-keypair` |
| Type | RSA |
| Format | .pem |

![Key pair](screenshots/01-keypair.png)

---

<a id="step-2"></a>
## Step 2 — 🌐 VPC

The VPC is the private network that every resource in this project lives inside, isolated from the rest of AWS.

| Setting | Value |
|---|---|
| Name | `ats-cv-vpc` |
| IPv4 CIDR | `10.0.0.0/16` |

![VPC](screenshots/02-vpc.png)

---

<a id="step-3"></a>
## Step 3 — 🧩 Subnets

Split the network into two zones in two different physical locations (AZs), so if one location has a problem, the other keeps working.

| Name | AZ | CIDR | Auto-assign public IPv4 |
|---|---|---|---|
| `ats-public-subnet-1` | us-east-1a | 10.0.1.0/24 | ✅ |
| `ats-public-subnet-2` | us-east-1b | 10.0.2.0/24 | ✅ |

![Subnets](screenshots/03-subnets.png)

---

<a id="step-4"></a>
## Step 4 — 🚪 Internet Gateway

This is the door that connects the network to the internet — without it, nothing outside could reach the servers.

| Setting | Value |
|---|---|
| Name | `ats-igw` |
| Attached VPC | `ats-cv-vpc` |
| State | Attached |

![Internet gateway](screenshots/04-igw.png)

---

<a id="step-5"></a>
## Step 5 — 🗺️ Route Table

This tells traffic leaving the network to go through the Internet Gateway to reach the internet.

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | `ats-igw` |

Associated with both public subnets.

![Route table](screenshots/05-route-table.png)
![Route table associations](screenshots/05-route-table-associations.png)

---

<a id="step-6"></a>
## Step 6 — 🛡️ Security Groups

These are firewall rules — configured so the EC2 servers only accept traffic from the Load Balancer, not directly from the internet.

**ALB security group** — `ats-alb-sg`

| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | 0.0.0.0/0 |

![ALB security group](screenshots/06-sg-alb.png)

**EC2 security group** — `ats-ec2-sg`

| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | `ats-alb-sg` |
| Inbound | SSH | 22 | Restricted — [see Step 15](#step-15) |

![EC2 security group](screenshots/06-sg-ec2.png)

---

<a id="step-7"></a>
## Step 7 — 🖥️ EC2 Instances

These are the servers running the website (Flask app) — two of them, in two different zones, for availability.

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

<a id="step-8"></a>
## Step 8 — ⚖️ Target Group & Load Balancer

This distributes visitors across both servers, and makes sure each server is healthy before sending it traffic.

| Setting | Value |
|---|---|
| Target group | `ats-target-group` — HTTP:80, health check `/health` |
| Load balancer | `ats-alb` — Internet-facing |
| Listener | HTTP:80 → `ats-target-group` |

![Load balancer](screenshots/08-alb.png)

---

<a id="step-9"></a>
## Step 9 — 🪣 S3 Bucket

This is where the generated CV files are stored.

| Setting | Value |
|---|---|
| Name | `ats-cv-storage-<account-id>` |
| Region | us-east-1 |
| Public access | Blocked |

![S3 bucket](screenshots/09-s3.png)

---

<a id="step-10"></a>
## Step 10 — 🗃️ DynamoDB Table

This is the database that stores each CV's data so it can be looked up later during analysis.

| Setting | Value |
|---|---|
| Name | `ats-cv-records` |
| Partition key | `cv_id` (String) |
| Capacity mode | On-demand |

![DynamoDB table](screenshots/10-dynamodb.png)
![DynamoDB empty items](screenshots/10-dynamodb-items.png)

---

<a id="step-11"></a>
## Step 11 — 🔐 IAM Role & Policy

These are the exact permissions the Lambda functions need — nothing more than what they actually use.

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

<a id="step-12"></a>
## Step 12 — ⚡ Lambda: CV Generator

This function takes the form data, builds a CV file out of it, and saves it to S3.

| Setting | Value |
|---|---|
| Name | `ats-cv-generator` |
| Runtime | Python 3.12 |
| Role | `ats-lambda-role` |
| Env vars | `BUCKET_NAME`, `TABLE_NAME` |
| Timeout / Memory | 30s / 256MB |

![Lambda generator config](screenshots/12-lambda-generator.png)

---

<a id="step-13"></a>
## Step 13 — ⚡ Lambda: JD Analyzer

This function compares the CV against a job description and returns a match score with missing keywords.

| Setting | Value |
|---|---|
| Name | `ats-jd-analyzer` |
| Runtime | Python 3.12 |
| Role | `ats-lambda-role` |
| Env var | `TABLE_NAME` |

![Lambda analyzer code](screenshots/13-lambda-analyzer-code.png)
![Lambda analyzer config](screenshots/13-lambda-analyzer.png)

---

<a id="step-14"></a>
## Step 14 — 🔌 API Gateway

This is the link that connects the website (Flask) to the Lambda functions.

| Setting | Value |
|---|---|
| API name | `ats-cv-api` (REST, Regional) |
| Resources | `/generate` → `ats-cv-generator`, `/analyze` → `ats-jd-analyzer` |
| Integration | Lambda proxy + CORS enabled |
| Stage | `prod` |

![API Gateway resources](screenshots/14-apigateway-resources.png)
![API Gateway invoke URL](screenshots/14-apigateway-invoke.png)

---

<a id="step-15"></a>
## Step 15 — 🚀 Flask Deployment

This is where the actual website code was uploaded to the servers and set to keep running continuously.

### 🐛 Issues Hit & Fixed

| Issue | Root Cause | Fix |
|---|---|---|
| SSH connection failed via EC2 Instance Connect despite correct "My IP" | Browser-based connection is proxied through AWS's own infrastructure — real source IP is AWS's `EC2_INSTANCE_CONNECT` service range, not the client's | Allowed `18.206.107.24/29` (us-east-1 service range) in `ats-ec2-sg` |
| Flask systemd service crash-looping with generic `exit-code` | `systemctl status` doesn't show the actual Python traceback | Ran `python3 app.py` directly to expose and fix the real error |

![Systemd service running](screenshots/15-systemd.png)

---

<a id="step-16"></a>
## Step 16 — ✅ End-to-End Test

A final check to confirm everything works together, from start to finish.

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

**[⬆ Back to top](#top)**

</div>
