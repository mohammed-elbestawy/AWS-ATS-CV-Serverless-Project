# 🧠 Design Concepts & Rationale

This file explains *why* each decision was made, not just *what* was built. Useful for interview prep — these are the questions likely to come up.

## Networking

**Why a /16 CIDR for the VPC?**
`10.0.0.0/16` gives 65,536 IP addresses — far more than this project needs. It's deliberately oversized to leave room for future subnets (private subnets, additional AZs) without re-architecting the network later.

**Why two subnets across two AZs instead of one?**
High availability. If `us-east-1a` has an outage, the ALB still has a healthy target in `us-east-1b`. A single-AZ setup would mean the whole app goes down if that one AZ fails.

**Why does the route table only have one custom route (`0.0.0.0/0 → igw`)?**
Both subnets are public — they only need one thing beyond the default local route: a path to the internet. Private subnets (not used in this project) would instead route through a NAT Gateway.

## Security

**Why does the EC2 security group only accept HTTP from the ALB security group, not from `0.0.0.0/0`?**
This is security group chaining — a form of defense in depth. The ALB is the only allowed entry point from the internet; EC2 instances are unreachable directly even if someone finds their public IP. If the ALB were ever bypassed or misconfigured, EC2 still wouldn't accept unsolicited traffic.

**Why is SSH restricted instead of left open?**
Port 22 is a common attack target. Restricting the source (even though it required troubleshooting the EC2 Instance Connect service range) keeps the attack surface to only the legitimate connection path, instead of `0.0.0.0/0`.

**Why does the IAM policy list exact actions (`s3:PutObject`, `s3:GetObject`) instead of `s3:*`?**
Least privilege. The Lambda functions only ever write and read CV files — they never delete anything. If the Lambda code were ever compromised or had a bug, `s3:*` would let an attacker (or a bad deploy) delete every file in the bucket. Scoping to the exact two actions on the exact bucket ARN limits the blast radius of any failure.

**Why is the DynamoDB action list `PutItem`, `GetItem`, `UpdateItem` — no `DeleteItem`?**
Same reasoning. Nothing in the application logic ever needs to delete a CV record, so the permission simply doesn't exist. A missing permission can't be misused.

| If we had used | Risk |
|---|---|
| `s3:*` on `*` | Any Lambda bug or leaked credential could read/delete every object in every bucket in the account |
| `dynamodb:*` on `*` | Same Lambda could touch unrelated tables, including ones outside this project |
| Scoped actions on scoped ARNs (what was actually used) | A compromised function can only do the specific things this app needs, on the specific resources it owns |

## Compute & Availability

**Why two EC2 instances behind a load balancer instead of one instance?**
Same high-availability reasoning as the subnets — no single point of failure in the compute layer. The ALB also spreads load and only routes to instances that pass the `/health` check, so a crashed instance is automatically taken out of rotation.

**Why Application Load Balancer instead of Network Load Balancer?**
ALB operates at Layer 7 (HTTP), which means it can route based on paths, do health checks against an actual HTTP endpoint (`/health`), and terminate/inspect HTTP traffic. NLB operates at Layer 4 and is meant for raw TCP/UDP performance — overkill and the wrong tool for a simple HTTP web app.

## Serverless Layer

**Why put API Gateway between Flask and Lambda instead of calling Lambda directly from EC2?**
Decoupling. EC2 only needs to know one HTTP URL — it has no AWS SDK calls, no IAM permissions to invoke Lambda, and no knowledge of Lambda internals. If the Lambda functions were rewritten, renamed, or replaced entirely, the Flask app wouldn't need to change at all as long as the API contract stays the same.

**Why REST API instead of the cheaper/faster HTTP API?**
For a real production app this size, HTTP API would be the better choice — lower cost, lower latency. REST API was used here deliberately because it demonstrates a broader feature set (usage plans, request/response transformation, older-style proxy integration) that's more commonly asked about in interviews and closer to what larger legacy systems still run.

**Why does the CV Generator Lambda write to both S3 and DynamoDB, but the JD Analyzer only touches DynamoDB?**
Separation of concerns by data type. S3 stores the actual CV *file* (unstructured text) because that's what object storage is for. DynamoDB stores *structured metadata* about each CV (name, skills, scores) because that's what a fast key-value lookup is for. The analyzer only ever needs the structured fields to compute a match score — it never needs the raw file — so it has no reason to touch S3, and its IAM permissions reflect that.

## Cost Decisions

**Why terminate EC2 and the ALB right after testing instead of leaving the project "live"?**
The project's purpose was to prove the architecture works end-to-end, not to serve real traffic indefinitely. EC2 and ALB are the only billable-by-the-hour components in this stack — leaving them running 24/7 with zero real users burns Free Tier credit for no benefit. Lambda, S3, DynamoDB, and API Gateway all have effectively-zero idle cost, so they were left running in case the project needs to be demoed again.
