# Blog App – Three-Tier AWS Architecture (CloudFront + API Gateway + EC2 + RDS) ✨

> Scope of this README: A production-style three-tier architecture (web / app / DB) on AWS, reflecting the actual code and infrastructure in this repository.

[![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-844FBA?logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Node.js](https://img.shields.io/badge/API-Node.js-43853D?logo=node.js&logoColor=white)](https://nodejs.org/)
[![Express](https://img.shields.io/badge/Framework-Express-000000?logo=express&logoColor=white)](https://expressjs.com/)
[![React](https://img.shields.io/badge/Frontend-React-61DAFB?logo=react&logoColor=061a23)](https://react.dev/)
[![PostgreSQL](https://img.shields.io/badge/DB-PostgreSQL-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Cognito](https://img.shields.io/badge/Auth-AWS%20Cognito-6B46C1?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/cognito/)
[![Region](https://img.shields.io/badge/Region-eu--west--1-232F3E?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](#license)

---

## 1. Project Overview

### What this project is

This is a production-style 3-tier web application on AWS. The stack (grounded in this repo’s Terraform and code):

- 🌐 Web tier – React SPA on Amazon S3, delivered via Amazon CloudFront (OAI)
- 🚪 API entry – Amazon API Gateway (HTTP API) in front of the backend ALB
- ⚙️ App tier – Node.js/Express on EC2 Auto Scaling behind an internal ALB
- 🗄️ Data tier – Amazon RDS PostgreSQL (private subnets, Multi-AZ)
- 🔐 Auth – AWS Cognito (JWT)
- 🪣 Media – Private S3 bucket + pre-signed URLs
- 🧰 Ops – CI/CD with AWS CodePipeline + CodeBuild; ECR for images
- 🛡️ Access – Bastion host for SSH and `psql` from inside the VPC

Endpoints include `/health`, `/posts`, and `/upload`. Focus areas: infrastructure, networking, observability, and robust deployment patterns.

### Why I built it

- 🏗️ Feels like “real prod” rather than a single EC2 box
- 🧭 Exercises VPC, subnets, route tables, NAT, IGW, SGs vs NACLs
- 🧪 Practices ALB + API Gateway integration, ASG rollouts, user data, ECR
- 🔄 Creates a baseline to compare costs/ops vs more serverless-heavy variants

---

## 2. High-Level Architecture

### Diagram

<img width="3924" height="3467" alt="3 Teir AWS" src="https://github.com/user-attachments/assets/4ce0db77-8ce2-47fb-bd69-d3f0a9daef4a" />


Highlights:
- Frontend is static and globally cached.
- API path is public via API Gateway, but backend remains private behind an internal ALB.
- Database is isolated in private subnets.

---

## 3. Naming & Inventory

### 3.1 Naming conventions

| Type               | Pattern                             | Example                                  |
|--------------------|-------------------------------------|------------------------------------------|
| VPC                | `blogapp-vpc-*`                     | `blogapp-vpc-main`                       |
| Subnets            | `blogapp-subnet-<tier>-<az>`        | `blogapp-subnet-app-a`                   |
| Route tables       | `blogapp-rt-*`                      | `blogapp-rt-public`                      |
| IGW / NAT          | `blogapp-igw-*`, `blogapp-nat-*`    | `blogapp-igw-main`, `blogapp-nat-a`      |
| Security groups    | `blogapp-sg-*`                      | `blogapp-sg-backend`, `blogapp-sg-rds`   |
| ALB                | `blogapp-alb-*`                     | `blogapp-alb-backend`                    |
| Target groups      | `blogapp-tg-*`                      | `blogapp-tg-backend`                     |
| Auto Scaling group | `blogapp-asg-*`                     | `blogapp-asg-backend`                    |
| Launch template    | `blogapp-backend-lt-*`              | `blogapp-backend-lt-ch`                  |
| Bastion            | `blogapp-ec2-bastion`               | `blogapp-ec2-bastion`                    |
| RDS DB             | `blogapp-rds-*`                     | `blogapp-rds-postgres`                   |
| ECR repo           | `blog-backend`                      | `blog-backend`                           |
| CloudFront         | `blogapp-cf-*`                      | `blogapp-cf-frontend`                    |
| S3 buckets         | `blogapp-frontend`, `blogapp-media` | `blogapp-frontend`, `blogapp-media`      |
| Cognito            | `blogapp-user-pool`                 | `blogapp-user-pool`                      |
| API Gateway        | `blogapp-http-api`                  | `blogapp-http-api`                       |

Tags:
- Environment: development
- Project: Three-Tier Blog App
- Region: eu-west-1
- Name prefix: blogapp

---

## 4. VPC & Networking

### 4.1 VPC and subnets

VPC: `blogapp-vpc-main` – `10.0.0.0/16` (eu-west-1, 2 AZs)

| Layer  | Name                       | AZ          | CIDR         | Public? | Purpose                          |
|--------|----------------------------|-------------|--------------|---------|----------------------------------|
| Public | blogapp-subnet-public-a    | eu-west-1a  | 10.0.1.0/24  | Yes     | Bastion, NAT GW                  |
| Public | blogapp-subnet-public-b    | eu-west-1b  | 10.0.2.0/24  | Yes     | HA for public resources          |
| App    | blogapp-subnet-app-a       | eu-west-1a  | 10.0.11.0/24 | No      | ALB + EC2 app nodes              |
| App    | blogapp-subnet-app-b       | eu-west-1b  | 10.0.12.0/24 | No      | ALB + EC2 app nodes              |
| DB     | blogapp-subnet-db-a        | eu-west-1a  | 10.0.21.0/24 | No      | RDS primary                      |
| DB     | blogapp-subnet-db-b        | eu-west-1b  | 10.0.22.0/24 | No      | RDS standby (Multi-AZ)           |

### 4.2 Route tables & gateways

| Route table        | Subnets attached                 | Routes                                                                 |
|--------------------|----------------------------------|------------------------------------------------------------------------|
| blogapp-rt-public  | public-a, public-b               | 10.0.0.0/16 → local; 0.0.0.0/0 → IGW                                  |
| blogapp-rt-app     | app-a, app-b                     | 10.0.0.0/16 → local; 0.0.0.0/0 → NAT GW                               |
| blogapp-rt-db      | db-a, db-b                       | 10.0.0.0/16 → local (no internet route)                               |

Notes:
- IGW: `blogapp-igw-main`
- NAT GW: `blogapp-nat-a` (in public-a)

### 4.3 NACLs

- Simplified/allow-all style per subnet for easier learning.
- Real network security enforced via Security Groups.

---

## 5. Security Groups & Access Model

### 5.1 Security group matrix

| SG                       | Attached to        | Inbound rules (key)                                            |
|--------------------------|--------------------|-----------------------------------------------------------------|
| blogapp-sg-bastion       | Bastion EC2        | 22 from trusted IP / CloudShell                                |
| blogapp-sg-backend-alb   | Backend ALB        | 80 from API Gateway integration (VPC/private)                  |
| blogapp-sg-backend       | Backend EC2 ASG    | 80 from ALB; 22 from bastion                                   |
| blogapp-sg-rds           | RDS PostgreSQL     | 5432 from blogapp-sg-backend and blogapp-sg-bastion            |

Outbound on all SGs: allow all (default).

### 5.2 Access principles

- Internet-facing: CloudFront and API Gateway only.
- Backend EC2: accessible only from ALB (80) and bastion (22).
- RDS: accessible from backend EC2 (5432) and bastion (5432 for `psql`).

Tip:
- Prefer SG-to-SG references over CIDRs for internal trust chains. 🔗

---

## 6. Ports & Traffic Hierarchy

### 6.1 End-to-end port flow

Frontend:

```text
Client browser
  → CloudFront (HTTPS 443)
    → S3 [OAI-secured]
```

API:

```text
Client browser
  → API Gateway (HTTPS 443)
    → Backend ALB (HTTP 80)
      → Backend EC2 (HTTP 80)
        → RDS Postgres (TCP 5432)
```

Admin/debug:

```text
CloudShell / Local IP
  → Bastion EC2 (SSH 22)
    → Backend EC2 (SSH 22)
    → RDS (TCP 5432)
```

### 6.2 Port table

| Layer       | Source       | Destination     | Port | Protocol | Purpose                        |
|------------|---------------|-----------------|------|----------|--------------------------------|
| Frontend   | Client        | CloudFront      | 443  | TCP      | SPA delivery                   |
| API entry  | Client        | API Gateway     | 443  | TCP      | Public API                     |
| App tier   | API Gateway   | Backend ALB     | 80   | TCP      | Internal ingress               |
| App tier   | Backend ALB   | Backend EC2     | 80   | TCP      | ALB → Node API                 |
| DB tier    | Backend EC2   | RDS             | 5432 | TCP      | App → Database                 |
| Admin SSH  | Bastion       | Backend EC2     | 22   | TCP      | Remote admin                   |
| Admin DB   | Bastion       | RDS             | 5432 | TCP      | Direct SQL                     |

---

## 7. Bastion & Database

### 7.1 Bastion host

- Instance: `blogapp-ec2-bastion` (public subnet)
- Use cases:
  - SSH into private EC2
  - `psql` into RDS for setup/debug

From CloudShell/local:

```bash
ssh -i blogapp-bastion-key.pem ec2-user@<bastion-public-ip>
```

From bastion:

```bash
psql "host=<rds-endpoint> port=5432 dbname=blog_db user=blog_user"
```

Troubleshooting:
- If `psql` times out: verify SG sources (SG-to-SG), route table associations, and NACL rules.

### 7.2 RDS PostgreSQL

- Engine: PostgreSQL, Multi-AZ
- Private subnets only (no public IP)
- Security via SG, not NACLs

Example schema (blog posts):

```sql
CREATE TABLE posts (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL,
  title       text NOT NULL,
  body        text,
  created_at  timestamptz NOT NULL DEFAULT now()
);
```

---

## 8. App Tier – Internal ALB + Node.js API

### 8.1 App ALB & target group

- ALB: internal, app subnets
- Listener: HTTP 80 → target group on instance port 80
- Health check: `HTTP /health` (200 OK)

### 8.2 App Auto Scaling Group

- Launch template: `blogapp-backend-lt-ch`
- Instance type: `t3.micro`
- Desired capacity: 2
- ECR repository: `blog-backend`
- Instance profile: allows pulling secrets/params and S3 access as needed

Observability:
- Send app/container logs to CloudWatch Logs for tailing and alarms.

### 8.3 Node.js API

Endpoints:
- `GET /health` → “OK” for health checks
- `GET /posts` → list posts
- `POST /upload` → presigned S3 upload flow

On an app instance:

```bash
sudo ss -lntp | grep ':80' || echo "nothing on 80"
curl -v http://localhost/health
curl -v http://localhost/posts
```

---

## 9. Web Tier – CloudFront + S3

### 9.1 CloudFront distribution

- Origin: S3 (OAI-secured)
- SPA-friendly behaviors
- Default root: `index.html`
- Consider compression, immutable asset caching, SEO headers

### 9.2 S3 frontend hosting

- Bucket for static assets (built via Vite)
- Uploads handled by CI/CD
- Block public access; rely on OAI

---

## 10. End-to-End Testing

From bastion → backend ALB:

```bash
BACKEND_ALB_DNS="internal-blogapp-alb-backend-XXXX.eu-west-1.elb.amazonaws.com"
curl -v http://$BACKEND_ALB_DNS/health
curl -v http://$BACKEND_ALB_DNS/posts
```

From internet → API Gateway:

```bash
API_GW_URL="https://abc123.execute-api.eu-west-1.amazonaws.com"
curl -v $API_GW_URL/health
curl -v $API_GW_URL/posts
```

Frontend:

```bash
CLOUDFRONT_URL="https://dxxxxxxxxxxxx.cloudfront.net"
curl -I $CLOUDFRONT_URL
open $CLOUDFRONT_URL
```

Upload flow (presigned URL):

```bash
# 1) Request presigned URL (authenticated)
curl -X POST "$API_GW_URL/upload" -H "Authorization: Bearer <JWT>" -d '{"filename":"img.png","contentType":"image/png"}'

# 2) PUT file to S3 with returned URL
curl -X PUT "<presignedUrl>" -H "Content-Type: image/png" --data-binary @img.png
```

---

## 11. Issues, Root Causes & Fixes

### 11.1 Summary table

| Symptom                               | Root cause                                        | Fix                                                        | Takeaway                                 |
|---------------------------------------|---------------------------------------------------|------------------------------------------------------------|------------------------------------------|
| `psql` from bastion times out         | NACL/SG mismatch                                  | Simplify NACLs, enforce via SG; verify SG-to-SG            | Check SG + NACL first                    |
| ALB targets unhealthy (timeout)       | App not listening (user data/container failure)   | Check logs; validate service binds on port 80              | Prove instance-local health before ALB   |
| API Gateway 502                       | Integration target/HC misconfigured               | Correct ALB integration, ensure `/health` 200              | 502 = gateway can’t reach backend        |
| 401/403 on API                        | Cognito pool/client mismatch                       | Align env/SSM params, region, domain                       | Auth drift is common                     |
| S3 upload fails                       | Wrong presign headers/bucket permissions          | Use exact Content-Type; confirm bucket policy & VPC endpoint| Signed URLs are strict                   |
| CI/CD fails to deploy                 | IAM perms or OAuth token missing                  | Update CodeBuild/CodePipeline roles; set GitHub token      | CI/CD needs correct roles/secrets        |

### 11.2 Troubleshooting flow

1) Instance-level
- `curl http://localhost/health`
- `sudo ss -lntp | grep ':80'`
- CloudWatch Logs for app/container

2) VPC-internal
- `curl http://<internal-alb-dns>/health` from bastion

3) Control plane
- Target group health reason codes
- API Gateway integration mappings

4) Network
- Route tables, NAT, NACL sanity

---

## 12. Cost Awareness

Relative costs (eu-west-1, low traffic):

- EC2 (t3.micro): ~ $7.5/instance/month
- Backend ASG (2x) + Bastion: ~ $22–25/month
- RDS (db.t3.micro): ~ $20–25/month (+ storage, Multi-AZ)
- ALB (1x): ~ $18–19/month base
- API Gateway (HTTP API): low dev usage → a few dollars
- CloudFront + S3: a few dollars (depends on traffic/assets)
- NAT Gateway: ~ $32–33/month base (+ data)
- Public IPv4/EIP: ~$3.6 each/month (bastion, NAT)

Total: roughly $120–150/month for always-on dev with minimal traffic.

Cost tips:
- Use t3.micro or spot for non-critical
- Scale to 1 desired in dev when idle
- Consider SSM Session Manager instead of bastion
- Explore VPC endpoints to reduce NAT data costs

---

## 13. CI/CD Pipeline

- Source: GitHub (triggers on push/PR)
- Build:
  - Backend → Docker image, push to ECR
  - Frontend → `vite build`, upload artifacts
- Deploy:
  - Backend → refresh instances/ASG rollout
  - Frontend → sync to S3 + CloudFront invalidation
- Artifacts: stored in S3

<img width="5138" height="4119" alt="3 Teir AWS (2)" src="https://github.com/user-attachments/assets/c7f1a86c-a698-42ad-bf29-c33ca90ff2ba" />



---

## 14. Configuration & Region

- Default region: `eu-west-1`
- Secrets & parameters: SSM / Secrets Manager
- Environment variables:
  - Database credentials/ARNs (Secrets Manager)
  - Cognito user pool/client IDs
  - S3 buckets (frontend, media)
  - Any third-party keys (kept out of source)

Operational guidance:
- Never commit secrets to Git
- Use IAM least privilege for roles and pipelines
- Prefer SG-to-SG links for internal trust

---

## 15. Cleaning Up AWS Resources

To avoid ongoing charges:

```bash
cd infrastructure
terraform destroy
```

Ensure:
- ALB access logs/CloudFront logs are rotated
- ECR images are pruned where appropriate
- S3 buckets are emptied if `force_destroy` is not set

---

## Quick Links

- Architecture: section 2
- Networking: section 4
- Security: section 5
- App/API: section 8
- Web (CloudFront + S3): section 9
- CI/CD: section 13
- Cost: section 12
- Cleanup: section 15

---

Made with ❤️ for learning, reliability, and clean AWS patterns.
