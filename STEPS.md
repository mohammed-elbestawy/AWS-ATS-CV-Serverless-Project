# 🛠️ Build Log — Step by Step

## Step 1 — Key Pair
- Created `ats-keypair` | Type: **RSA** | Format: **.pem**
- Saved securely for EC2 SSH access

![Key pair](screenshots/01-keypair.png)

## Step 2 — VPC
- Name: `ats-cv-vpc`
- IPv4 CIDR: `10.0.0.0/16`

![VPC](screenshots/02-vpc.png)

## Step 3 — Subnets

| Name | AZ | CIDR |
|---|---|---|
| `ats-public-subnet-1` | us-east-1a | 10.0.1.0/24 |
| `ats-public-subnet-2` | us-east-1b | 10.0.2.0/24 |

✅ Auto-assign public IPv4 enabled on both

![Subnets](screenshots/03-subnets.png)

## Step 4 — Internet Gateway
- Name: `ats-igw`
- Attached to `ats-cv-vpc`

![Internet gateway](screenshots/04-igw.png)

## Step 5 — Route Table
- Name: `ats-public-rt` | VPC: `ats-cv-vpc`
- Route: `0.0.0.0/0` → `ats-igw`
- Associated with both public subnets

![Route table](screenshots/05-route-table.png)
![Route table associations](screenshots/05-route-table-associations.png)

## Step 6 — Security Groups

**ALB Security Group (`ats-alb-sg`)**

| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | 0.0.0.0/0 |

![ALB security group](screenshots/06-sg-alb.png)

**EC2 Security Group (`ats-ec2-sg`)**

| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | ats-alb-sg |
| Inbound | SSH | 22 | Restricted (see Issues below) |

![EC2 security group](screenshots/06-sg-ec2.png)

## Step 7 — EC2 Instances

Launched two instances, one per AZ:

| Setting | Instance 1 | Instance 2 |
|---|---|---|
| Name | ats-ec2-1 | ats-ec2-2 |
| Type | t3.micro | t3.micro |
| Subnet | ats-public-subnet-1 | ats-public-subnet-2 |
| Security Group | ats-ec2-sg | ats-ec2-sg |
| Key Pair | ats-keypair | ats-keypair |

User Data:
```bash
#!/bin/bash
yum update -y
yum install -y python3 python3-pip
pip3 install flask requests
mkdir -p /home/ec2-user/ats-app/templates
```

> ⚠️ **Issue:** `t2.micro` is not Free Tier–eligible on new AWS accounts (post-July 2025 Free Plan). Used **t3.micro** instead.

![EC2 instances running](screenshots/07-ec2.png)

## Step 8 — Target Group & Load Balancer

**Target Group:** `ats-target-group` | Protocol: HTTP | Port: 80 | Health check: `/health`

**Load Balancer:** `ats-alb` | Scheme: Internet-facing | Listener: HTTP:80 → `ats-target-group`

![Load balancer](screenshots/08-alb.png)

## Step 9 — S3 Bucket
- Name: `ats-cv-storage-<account-id>`
- Region: us-east-1 | Default settings, public access blocked

![S3 bucket](screenshots/09-s3.png)

## Step 10 — DynamoDB Table
- Name: `ats-cv-records`
- Partition key: `cv_id` (String)
- Capacity mode: On-demand

![DynamoDB table](screenshots/10-dynamodb.png)
![DynamoDB empty items](screenshots/10-dynamodb-items.png)

## Step 11 — IAM Role

Policy: `ats-lambda-policy` — scoped to exact resource ARNs, no wildcards:
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

Role: `ats-lambda-role` — attached to `ats-lambda-policy`

![IAM policy](screenshots/11-iam-policy.png)
![IAM role](screenshots/11-iam-role.png)

## Step 12 — Lambda: CV Generator
- Name: `ats-cv-generator` | Runtime: Python 3.12 | Role: `ats-lambda-role`
- Env vars: `BUCKET_NAME`, `TABLE_NAME`
- Timeout: 30s | Memory: 256MB

![Lambda generator config](screenshots/12-lambda-generator.png)

## Step 13 — Lambda: JD Analyzer
- Name: `ats-jd-analyzer` | Runtime: Python 3.12 | Role: `ats-lambda-role`
- Env var: `TABLE_NAME`

![Lambda analyzer code](screenshots/13-lambda-analyzer-code.png)
![Lambda analyzer config](screenshots/13-lambda-analyzer.png)

## Step 14 — API Gateway
- API: `ats-cv-api` (REST, Regional)
- Resources: `/generate` → `ats-cv-generator`, `/analyze` → `ats-jd-analyzer`
- Proxy integration + CORS enabled on both
- Deployed to stage: `prod`

![API Gateway resources](screenshots/14-apigateway-resources.png)
![API Gateway invoke URL](screenshots/14-apigateway-invoke.png)

## Step 15 — Flask Deployment

Deployed to both EC2 instances via EC2 Instance Connect, running as a `systemd` service pointing to the API Gateway Invoke URL.

> ⚠️ **Issue:** EC2 Instance Connect (browser-based) kept failing with "Error establishing SSH connection" even with "My IP" correctly configured in the security group.
> **Root cause:** the browser-based connection is proxied through AWS's own infrastructure — the real source IP hitting port 22 is AWS's `EC2_INSTANCE_CONNECT` service range, not the client's IP.
> **Fix:** allowed `18.206.107.24/29` (us-east-1 service range) in `ats-ec2-sg`.

> ⚠️ **Issue:** Flask systemd service crash-looped, `systemctl status` only showed a generic `exit-code` with no visible error.
> **Fix:** ran `python3 app.py` directly to expose the real traceback and resolve it.

![Systemd service running](screenshots/15-systemd.png)

## Step 16 — ✅ End-to-End Test

1. Opened the ALB DNS name in the browser.
2. Filled in the form with sample details and generated a CV — got a success message with a download link.
3. Pasted a job description ("Cloud Security Operations") and checked the match score.

![JD match result — score and missing keywords](screenshots/16-fulltest.png)
![CV generation form filled and submitted](screenshots/16-cv-form-filled.png)


| Check | Result |
|---|---|
| Target group health | ✅ Healthy |
| ALB DNS reachable | ✅ |
| CV generation | ✅ Successful |
| JD match analysis | ✅ Returned score + missing keywords |

**🧹 Cleanup:** terminated both EC2 instances, deleted the ALB and target group immediately after testing to avoid ongoing charges. Lambda, S3, DynamoDB, and API Gateway left running (near-zero idle cost).
