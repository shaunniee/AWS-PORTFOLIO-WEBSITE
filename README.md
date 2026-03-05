<div align="center">

# ☁️ AWS Static Portfolio Website

**Production-grade static site on AWS with fully automated CI/CD**

[![Live Site](https://img.shields.io/badge/🌐_Live-shaunvividszportfolio.com-2563eb?style=for-the-badge)](https://www.shaunvividszportfolio.com)
[![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonwebservices&logoColor=white)](https://aws.amazon.com)
[![CI/CD](https://img.shields.io/badge/CI%2FCD-CodePipeline-16a34a?style=for-the-badge&logo=amazonaws&logoColor=white)](#-cicd-pipeline)

![S3](https://img.shields.io/badge/S3-569A31?style=flat-square&logo=amazons3&logoColor=white)
![CloudFront](https://img.shields.io/badge/CloudFront-8C4FFF?style=flat-square&logo=amazonaws&logoColor=white)
![Route 53](https://img.shields.io/badge/Route_53-8C4FFF?style=flat-square&logo=amazonroute53&logoColor=white)
![ACM](https://img.shields.io/badge/ACM_TLS-DD344C?style=flat-square&logo=letsencrypt&logoColor=white)
![CodeBuild](https://img.shields.io/badge/CodeBuild-2563EB?style=flat-square&logo=amazonaws&logoColor=white)
![CloudWatch](https://img.shields.io/badge/CloudWatch-FF4F8B?style=flat-square&logo=amazoncloudwatch&logoColor=white)
![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=flat-square&logo=html5&logoColor=white)
![CSS3](https://img.shields.io/badge/CSS3-1572B6?style=flat-square&logo=css3&logoColor=white)

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [AWS Resources](#-aws-resources)
- [CI/CD Pipeline](#-cicd-pipeline)
- [Security](#-security)
- [Observability](#-observability--monitoring)
- [Cost](#-cost-profile)
- [Deployment](#-deployment)
- [Repository Structure](#-repository-structure)
- [Lessons Learned](#-lessons-learned)

---

## 🔎 Overview

A personal cloud engineering portfolio hosted on AWS using a **production-style architecture** — private S3 origin, global CDN delivery via CloudFront, custom domain with TLS, fully automated deployments on every `git push`, and operational monitoring with alerting.

### 🎯 Goals

| Goal | Implementation |
|------|---------------|
| ⚡ **Global performance** | CloudFront CDN with edge caching |
| 🔒 **HTTPS everywhere** | ACM TLS certificate, HTTP → HTTPS redirect |
| 🚫 **No public S3** | Private bucket, Origin Access Control (OAC) |
| 🔄 **Zero-touch deploys** | `git push` → CodePipeline → CodeBuild → live |
| 📊 **Operational visibility** | CloudWatch alarms, CloudFront access logs |

### 🛠 Tech Stack

| Layer | Service | Purpose |
|-------|---------|---------|
| 🗄️ **Origin** | Amazon S3 | Private static file storage |
| 🌐 **CDN** | Amazon CloudFront | Global edge delivery + HTTPS termination |
| 📍 **DNS** | Amazon Route 53 | Custom domain resolution |
| 🔐 **TLS** | AWS Certificate Manager | Free public TLS certificate |
| 🔄 **CI/CD** | CodePipeline + CodeBuild | Automated build and deploy |
| 📊 **Monitoring** | CloudWatch + SNS | 5xx alerting and metrics |
| 📝 **Logging** | CloudFront Access Logs | Traffic analysis and debugging |

---

## 🏗 Architecture

<p align="center">
  <img width="100%" alt="Architecture Diagram — S3 + CloudFront + CodePipeline" src="https://github.com/user-attachments/assets/5a1c27b6-06bb-4148-af41-ba14100c5baf" />
</p>

### 📐 Request Flow

```
Developer                    AWS
────────                    ───
  │
  ├── git push main ──────► GitHub
  │                           │
  │                    ┌──────┴──────┐
  │                    │ CodePipeline │
  │                    └──────┬──────┘
  │                           │
  │                    ┌──────┴──────┐
  │                    │  CodeBuild  │
  │                    └──┬──────┬───┘
  │                       │      │
  │               s3 sync ▼      ▼ cache invalidation
  │                    ┌────┐  ┌────────────┐
  │                    │ S3 │  │ CloudFront │
  │                    └────┘  └─────┬──────┘
  │                                  │
User ── HTTPS ── Route 53 ──────────►│
                                     ▼
                              Edge location
                           (cached response)
```

---

## 📁 AWS Resources

### 🗄️ Amazon S3

<table>
<tr><th>Bucket</th><th>Region</th><th>Purpose</th><th>Configuration</th></tr>
<tr>
  <td><code>shaun-portfolio-content</code></td>
  <td><code>eu-west-1</code></td>
  <td>Static site files</td>
  <td>
    ✅ Block all public access<br>
    ✅ OAC-only bucket policy<br>
    ✅ Default encryption (SSE-S3)<br>
    ✅ Versioning enabled
  </td>
</tr>
<tr>
  <td><code>shaun-portfolio-cf-logs</code></td>
  <td><code>eu-west-1</code></td>
  <td>CloudFront access logs</td>
  <td>
    ✅ Block all public access<br>
    ✅ ACL for CloudFront logging
  </td>
</tr>
</table>

**Bucket Policy (content bucket):**
```json
{
  "Effect": "Allow",
  "Principal": { "Service": "cloudfront.amazonaws.com" },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::shaun-portfolio-content/*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT>:distribution/<DIST_ID>"
    }
  }
}
```

### 🌐 Amazon CloudFront

| Setting | Value |
|---------|-------|
| **Origin** | S3 via Origin Access Control (OAC) |
| **Default root object** | `index.html` |
| **Custom domain** | `www.shaunvividszportfolio.com` |
| **TLS certificate** | ACM public cert (`us-east-1`) |
| **Viewer protocol** | Redirect HTTP → HTTPS |
| **Cache policy** | Managed `CachingOptimized` |
| **Access logging** | Enabled → `shaun-portfolio-cf-logs` |

### 📍 Amazon Route 53

| Record | Type | Target |
|--------|------|--------|
| `www.shaunvividszportfolio.com` | A (Alias) | CloudFront distribution |

### 🔐 AWS Certificate Manager

- **Certificate type:** Public
- **Region:** `us-east-1` (required for CloudFront)
- **Domain:** `*.shaunvividszportfolio.com`
- **Validation:** DNS (CNAME record in Route 53)
- **Auto-renewal:** ✅ Enabled

---

## 🔄 CI/CD Pipeline

### Pipeline Architecture

```
┌─────────────────────────────────────────────────────┐
│                   CodePipeline                       │
├──────────────┬──────────────┬───────────────────────┤
│ 📥 Source     │ 🔨 Build     │ 📤 Deploy             │
│              │              │                       │
│ GitHub       │ CodeBuild    │ S3 sync               │
│ main branch  │ buildspec.yml│ CloudFront invalidate  │
└──────────────┴──────────────┴───────────────────────┘
```

### 🔨 buildspec.yml

```yaml
version: 0.2

env:
  parameter-store:
    S3_BUCKET: "/portfolio/static-site/s3-bucket"
    CLOUDFRONT_DISTRIBUTION_ID: "/portfolio/static-site/cf-distribution-id"

phases:
  install:
    commands:
      - echo "Install phase - nothing to install for basic static site"
  build:
    commands:
      - echo "Build phase - no framework build step for plain HTML"
  post_build:
    commands:
      - echo "Deploying to S3..."
      - aws s3 sync . s3://$S3_BUCKET --delete --exclude ".git/*" --exclude "buildspec.yml"
      - echo "Creating CloudFront invalidation..."
      - aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_DISTRIBUTION_ID --paths "/*"

artifacts:
  files:
    - '**/*'
```

> 💡 **Note:** Sensitive values (`S3_BUCKET`, `CLOUDFRONT_DISTRIBUTION_ID`) are pulled from **AWS Systems Manager Parameter Store** — no secrets hardcoded in the repo.

### ⚙️ CodeBuild IAM Permissions

The CodeBuild service role follows **least-privilege** principles:

| Permission | Resource | Purpose |
|------------|----------|---------|
| `s3:ListBucket` | Content bucket | List existing objects |
| `s3:GetObject` | Content bucket `/*` | Read files during sync |
| `s3:PutObject` | Content bucket `/*` | Upload new/changed files |
| `s3:DeleteObject` | Content bucket `/*` | Remove deleted files (`--delete`) |
| `cloudfront:CreateInvalidation` | Distribution ARN | Purge CDN cache |
| `logs:*` | CodeBuild log group | Write build logs |
| `ssm:GetParameters` | Parameter Store paths | Read deploy config |

---

## 🔐 Security

| Control | Implementation |
|---------|---------------|
| 🚫 **No public S3** | Block all public access enabled; bucket policy only allows CloudFront OAC |
| 🔒 **HTTPS enforced** | CloudFront viewer protocol redirects HTTP → HTTPS; ACM TLS on custom domain |
| 🔑 **Least-privilege IAM** | CodeBuild role scoped to specific bucket + distribution only |
| 🗄️ **Encryption** | S3 default encryption (SSE-S3); TLS in transit |
| 📝 **Audit trail** | CloudFront access logs + CodeBuild logs in CloudWatch |
| 🔗 **Origin isolation** | OAC replaces legacy OAI — S3 is never directly accessible |

---

## 📊 Observability & Monitoring

### 🚨 CloudWatch Alarm — 5xx Error Rate

| Setting | Value |
|---------|-------|
| **Region** | `us-east-1` (CloudFront metrics) |
| **Metric** | `5xxErrorRate` |
| **Period** | 5 minutes |
| **Threshold** | > 1% for 1 consecutive period |
| **Action** | SNS → email notification |

### 📝 CloudFront Access Logs

- **Destination:** `shaun-portfolio-cf-logs` (private S3 bucket)
- **Use cases:**
  - 🔍 Debug 4xx/5xx errors
  - 📈 Analyse traffic patterns
  - 📊 Future: feed into Athena/Glue for dashboards

### 🔨 CodeBuild Logs

- **Log group:** `/aws/codebuild/shaunvividsz-portfolio-build`
- **Region:** `eu-west-1`
- **Contents:** Full output of `s3 sync` and `cloudfront create-invalidation`

### 💰 Cost Monitoring

- AWS Budgets monthly limit with email alerts on actual + forecasted spend
- CloudWatch billing alarm on `EstimatedCharges` in `us-east-1`

---

## 💰 Cost Profile

This architecture is designed for **near-zero cost** for a personal site:

| Service | Cost Driver | Expected |
|---------|------------|----------|
| 🗄️ S3 | Storage + GET requests | ~ $0.01/mo |
| 🌐 CloudFront | Data transfer + requests | ~ $0.00 (free tier) |
| 📍 Route 53 | 1 hosted zone + queries | ~ $0.50/mo |
| 🔐 ACM | Public certificate | **Free** |
| 🔄 CodePipeline | 1 pipeline | **Free** (1 free) |
| 🔨 CodeBuild | Build minutes | ~ $0.00 (100 min/mo free) |
| 📊 CloudWatch | 1 alarm + logs | ~ $0.10/mo |
| | **Total** | **< $1/mo** |

> ✅ No EC2 instances, no RDS databases, no continuously running compute.

---

## 🚀 Deployment

### Quick Deploy

```bash
# 1. Make your changes
vim index.html

# 2. Commit and push
git add .
git commit -m "Update portfolio"
git push origin main

# 3. CodePipeline automatically:
#    📥 Pulls code from GitHub
#    🔨 Runs CodeBuild (buildspec.yml)
#    📤 Syncs to S3 + invalidates CloudFront cache
#
# 4. ✅ Live at https://www.shaunvividszportfolio.com
```

### Pipeline Stages

| Stage | Action | Duration |
|-------|--------|----------|
| 📥 **Source** | Pull from GitHub `main` | ~10s |
| 🔨 **Build** | CodeBuild runs `buildspec.yml` | ~30s |
| 📤 **Deploy** | `s3 sync` + CloudFront invalidation | ~20s |
| ✅ **Live** | Changes propagated to edge locations | ~60s |

---

## 📂 Repository Structure

```
AWS-PORTFOLIO-WEBSITE/
├── 📄 index.html          # Main portfolio page (HTML + CSS + Lucide icons)
├── 📋 buildspec.yml       # CodeBuild deploy specification
├── 📖 README.md           # This documentation
└── 📁 projects/           # Individual project READMEs
    ├── readme (2).md      # Serverless Blog Platform
    ├── README (3).md      # Three-Tier Blog App
    ├── README (4).md      # Serverless Ordering System
    └── README (5).md      # Static Portfolio Site (this project)
```

---

## 📝 Lessons Learned

| Challenge | Resolution |
|-----------|-----------|
| 🌍 ACM cert must be in `us-east-1` for CloudFront | Created cert in `us-east-1` with DNS validation via Route 53 |
| 🔗 OAI vs OAC for S3 origin | Used modern **OAC** — simpler bucket policy, supports SSE-KMS |
| 🔄 Cache stale after deploy | Added `cloudfront create-invalidation --paths "/*"` in buildspec |
| 🔐 CodeBuild IAM too broad initially | Scoped down to specific bucket ARN + distribution ARN only |
| 📊 No visibility into errors | Added CloudWatch 5xx alarm + SNS email notification |

---

## 🏗 Related Projects

| # | Project | Architecture | Key Services |
|---|---------|-------------|-------------|
| 01 | **[Portfolio Site](https://www.shaunvividszportfolio.com)** (this) | Static + Edge | S3, CloudFront, CodePipeline |
| 02 | **Three-Tier Blog App** | VPC + 3-Tier | VPC, ALB, EC2 ASG, RDS |
| 03 | **Serverless Blog Platform** | Serverless Full-Stack | Lambda, API GW, DynamoDB |
| 04 | **Serverless Ordering System** | Event-Driven Saga | Step Functions, SQS, DynamoDB |

---

<div align="center">

**Built by [Shaun](https://www.linkedin.com/in/shaun-vivian-dsouza-b12a73176)** · Deployed on AWS · Automated with CodePipeline

![AWS Solutions Architect](https://img.shields.io/badge/AWS-Solutions_Architect_Associate-FF9900?style=flat-square&logo=amazonwebservices&logoColor=white)
![AWS Developer](https://img.shields.io/badge/AWS-Developer_Associate-FF9900?style=flat-square&logo=amazonwebservices&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-Associate-844FBA?style=flat-square&logo=terraform&logoColor=white)

</div>
