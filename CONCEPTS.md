<a id="top"></a>

# 🧠 Design Concepts & Rationale

This file explains **why** each decision was made, not just what was built. These are the questions most likely to come up in an interview.

## 📋 Quick Navigation

| Topic | Section |
|---|---|
| 🌐 | [Networking Decisions](#networking) |
| 🛡️ | [Security Decisions](#security) |
| 🖥️ | [Compute & Availability](#compute) |
| ⚡ | [Serverless Layer](#serverless) |
| 💰 | [Cost Decisions](#cost) |

---

<a id="networking"></a>
## 🌐 Networking Decisions

| Question | Answer |
|---|---|
| Why a `/16` CIDR for the VPC? | Gives 65,536 IPs — far more than needed. Deliberately oversized to leave room for future subnets without re-architecting later. |
| Why two subnets across two AZs instead of one? | High availability. If `us-east-1a` fails, the ALB still has a healthy target in `us-east-1b`. A single-AZ setup means one outage takes down the whole app. |
| Why only one custom route (`0.0.0.0/0 → igw`)? | Both subnets are public — they only need a path to the internet. Private subnets (not used here) would route through a NAT Gateway instead. |

---

<a id="security"></a>
## 🛡️ Security Decisions

| Question | Answer |
|---|---|
| Why does EC2 only accept HTTP from the ALB's security group, not `0.0.0.0/0`? | **Security group chaining** — defense in depth. The ALB is the only allowed entry point from the internet; EC2 is unreachable directly even if someone finds its public IP. |
| Why restrict SSH instead of leaving it open? | Port 22 is a common attack target. Restricting the source keeps the attack surface to only the legitimate connection path instead of the entire internet. |
| Why exact actions (`s3:PutObject`, `s3:GetObject`) instead of `s3:*`? | Least privilege. The Lambdas only ever write/read CV files — never delete. `s3:*` would let a bug or leaked credential wipe the whole bucket. |
| Why no `DeleteItem` in the DynamoDB permissions? | The app never deletes a CV record, so the permission simply doesn't exist. A missing permission can't be misused. |

**Blast radius comparison:**

| Permission style | Risk if compromised |
|---|---|
| `s3:*` on `*` | Read/delete every object in every bucket in the account |
| `dynamodb:*` on `*` | Touch unrelated tables outside this project |
| Scoped actions on scoped ARNs *(what was actually used)* | Only the specific things this app needs, on the resources it owns |

---

<a id="compute"></a>
## 🖥️ Compute & Availability

| Question | Answer |
|---|---|
| Why two EC2 instances behind a load balancer instead of one? | No single point of failure in the compute layer. The ALB only routes to instances that pass the `/health` check, so a crashed instance is automatically taken out of rotation. |
| Why Application Load Balancer instead of Network Load Balancer? | ALB works at Layer 7 (HTTP) — it can route by path and health-check an actual HTTP endpoint (`/health`). NLB is Layer 4, built for raw TCP/UDP throughput — the wrong tool for a simple web app. |

---

<a id="serverless"></a>
## ⚡ Serverless Layer

| Question | Answer |
|---|---|
| Why API Gateway between Flask and Lambda instead of calling Lambda directly? | **Decoupling.** EC2 only needs one HTTP URL — no AWS SDK calls, no IAM permissions to invoke Lambda. The Lambdas could be rewritten entirely and Flask wouldn't need to change, as long as the API contract stays the same. |
| Why REST API instead of the cheaper/faster HTTP API? | For production, HTTP API would be the better choice. REST API was used deliberately here to demonstrate a broader feature set (usage plans, proxy integration, request/response handling) more commonly asked about in interviews. |
| Why does the CV Generator write to both S3 and DynamoDB, but the Analyzer only touches DynamoDB? | Separation by data type. S3 stores the actual CV *file* (unstructured text). DynamoDB stores *structured metadata* for fast lookups. The analyzer only needs structured fields to compute a score — it never touches the raw file, and its IAM permissions reflect that. |

---

<a id="cost"></a>
## 💰 Cost Decisions

| Question | Answer |
|---|---|
| Why terminate EC2 and the ALB right after testing? | The goal was proving the architecture works end-to-end, not serving real traffic. EC2 and ALB are the only billable-by-the-hour components — leaving them running 24/7 with zero users burns Free Tier credit for no benefit. |
| Why leave Lambda, S3, DynamoDB, and API Gateway running? | Their idle cost is effectively zero, and keeping them live means the project can be demoed again instantly without rebuilding anything. |

---

<div align="center">

**[⬆ Back to top](#top)**

</div>
