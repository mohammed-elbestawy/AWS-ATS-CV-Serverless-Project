# 🎯 ATS CV Generator — Serverless AWS Project

A fully-serverless, cloud-native application that generates ATS-optimized CVs and analyzes their compatibility with job descriptions — built and deployed end-to-end on AWS.

![AWS](https://img.shields.io/badge/AWS-FF9900?style=flat&logo=amazonaws&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=flat&logo=flask&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen)

## Overview

A production-style AWS architecture where multiple managed services work together: users submit their professional details through a web form, receive a downloadable ATS-friendly CV, and instantly check how well it matches a given job description.

Every component below — networking, compute, security groups, IAM permissions — was designed, deployed, and verified by hand on a personal AWS account.

## Architecture

| Layer | Service | Purpose |
|---|---|---|
| Networking | VPC, Subnets, IGW, Route Table | Isolated network foundation across 2 Availability Zones |
| Compute | EC2 (x2, t3.micro) | Hosted the Flask frontend |
| Load Balancing | Application Load Balancer | Distributed traffic across both AZs |
| Serverless | Lambda (x2, Python 3.12) | CV generation + JD analysis logic |
| Storage | S3 | Stores generated CV files |
| Database | DynamoDB | Stores CV records (on-demand capacity) |
| API | API Gateway (REST) | Connects Flask → Lambda |
| Security | IAM Role (least privilege) | Lambda permissions scoped to exact resource ARNs |

Region: `us-east-1`

## Features

- **CV Generation** — fills out a form → generates an ATS-optimized text CV → stores it in S3 → returns a download link
- **JD Match Analysis** — pastes any job description → gets a match score, missing keywords, and improvement suggestions
- **High Availability** — two EC2 instances across two AZs behind a load balancer
- **Serverless Logic** — Lambda functions handle generation/analysis with zero server management
- **Least-Privilege Security** — no wildcard IAM permissions anywhere in the stack

## Live Test Result

Successfully generated a CV, stored it in S3, and ran the JD analyzer end-to-end — returned a real match score with missing-keyword suggestions against an actual job description.

## Real Issues Hit & Fixed

1. **EC2 instance type**: `t2.micro` isn't Free Tier–eligible on new AWS accounts (post-July 2025 Free Plan) — switched to `t3.micro`.
2. **SSH via EC2 Instance Connect kept failing**: allowing "My IP" in the security group wasn't enough — EC2 Instance Connect (browser-based) proxies the SSH session through AWS's own infrastructure, so the real source is AWS's `EC2_INSTANCE_CONNECT` service IP range, not the client's IP. Fixed by allowing `18.206.107.24/29` (the us-east-1 service range).
3. **Flask service crash-looping**: `systemctl status` only showed a generic `exit-code`. Ran the app directly (`python3 app.py`) to expose the real traceback and resolve it.

## Cost Management

EC2 instances and the Application Load Balancer were terminated immediately after validating the full flow, to avoid unnecessary charges on billable resources. Lambda, S3, DynamoDB, and API Gateway were left running since their idle cost is effectively zero.

## Project Structure

    AWS-ATS-CV-Serverless-Project/
    ├── README.md
    ├── STEPS.md
    ├── CONCEPTS.md
    ├── screenshots/
    ├── code/
    │   ├── lambda_generator/lambda_function.py
    │   ├── lambda_analyzer/lambda_function.py
    │   └── flask_app/app.py
    └── iam/ats-lambda-policy.json
