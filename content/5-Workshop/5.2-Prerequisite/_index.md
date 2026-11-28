---
report_key: "workshopChapters"
report_language: "en"
report_fields: {"shortTitle": "Prerequisites", "customTitle": "5.2. Prerequisites & Environment Verification", "summary": "Select the ap-southeast-1 (Singapore) region, verify AWS CLI identity on CloudShell, and prepare the main repository branch and external API credentials.", "content": "Prepare all prerequisites before provisioning: switch AWS Console to Asia Pacific (Singapore) ap-southeast-1, verify AWS CLI caller identity in CloudShell, and confirm the main branch codebase and external API credentials.", "notes": "", "steps": [{"num": "Step 5.2.1", "title": "Select Asia Pacific (Singapore) ap-southeast-1 in AWS Console", "summary": "Sign in to the AWS Management Console and select **Asia Pacific (Singapore) `ap-southeast-1`** in the top-right region selector for low latency to Vietnam.", "bullets": ["Ensure the IAM identity has permissions for VPC, EC2, EBS, RDS, S3, ECR, CodeBuild, ALB, CloudFront, WAFv2, Secrets Manager, and IAM.", "All regional resources (VPC, RDS, EC2, ALB, S3, ECR, Secrets) are co-located in `ap-southeast-1`."], "images": [{"url": "static/images/5-Workshop/5.2-Prerequisite/01-region.png", "caption": "Figure 01 — Selecting Asia Pacific (Singapore) ap-southeast-1 in AWS Console"}]}, {"num": "Step 5.2.2", "title": "Launch AWS CloudShell & Verify AWS CLI Identity", "summary": "Open **AWS CloudShell** from the Console toolbar to run provisioning scripts cleanly without storing long-lived credentials on a workstation.", "code": "export AWS_REGION=ap-southeast-1\naws configure set region ap-southeast-1\naws sts get-caller-identity\naws configure get region", "expected": "{\n    \"UserId\": \"AIDA...\",\n    \"Account\": \"<AWS_ACCOUNT_ID>\",\n    \"Arn\": \"arn:aws:iam::<AWS_ACCOUNT_ID>:user/...\"\n}\nap-southeast-1", "images": [{"url": "static/images/5-Workshop/5.2-Prerequisite/02-aws-identity.png", "caption": "Figure 02 — Verifying AWS CLI identity (aws sts get-caller-identity) and ap-southeast-1 on CloudShell"}]}, {"num": "Step 5.2.3", "title": "Prepare the main Branch & External Integration Credentials", "summary": "Verify that `chatbot_medical` (`main`) contains `scripts/docker_entrypoint.py` (which auto-ingests the 364 RAG documents on first boot) and gather external credentials to store in AWS Secrets Manager in Chapter 5.3:", "table": {"headers": ["Category", "Items Required", "Role"], "rows": [["Code & Container", "`main` branch (`Dockerfile` non-root `appuser`, `scripts/docker_entrypoint.py`)", "Built by AWS CodeBuild into a ~334 MB ECR image (`v1.0.0`)."], ["AI & Vision OCR", "4 Groq API Keys (4-Key Pool Failover separated for Chat, OCR, Nutrition & backup for `qwen/qwen3.8-27b`)", "Clinical chat triage, prescription OCR reconciliation, and nutrition planning."], ["Auth & OTP", "Google OAuth 2.0 Client ID/Secret, Gmail SMTP App Password, SpeedSMS Token", "Google Sign-In, 6-digit OTP verification, and appointment confirmation emails."]]}}]}
report_index: 1
report_shared: {"code": "5.2"}
title: "Prerequisites"
date: "2026-11-27"
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

# 5.2. Prerequisites & Environment Verification

Select the ap-southeast-1 (Singapore) region, verify AWS CLI identity on CloudShell, and prepare the main repository branch and external API credentials.

## Step 5.2.1 — Select Asia Pacific (Singapore) ap-southeast-1 in AWS Console

Sign in to the AWS Management Console and select **Asia Pacific (Singapore) `ap-southeast-1`** in the top-right region selector for low latency to Vietnam.

- Ensure the IAM identity has permissions for VPC, EC2, EBS, RDS, S3, ECR, CodeBuild, ALB, CloudFront, WAFv2, Secrets Manager, and IAM.
- All regional resources (VPC, RDS, EC2, ALB, S3, ECR, Secrets) are co-located in `ap-southeast-1`.

![Figure 01 — Selecting Asia Pacific (Singapore) ap-southeast-1 in AWS Console](../../../static/images/5-Workshop/5.2-Prerequisite/01-region.png)
*Figure 01 — Selecting Asia Pacific (Singapore) ap-southeast-1 in AWS Console*

## Step 5.2.2 — Launch AWS CloudShell & Verify AWS CLI Identity

Open **AWS CloudShell** from the Console toolbar to run provisioning scripts cleanly without storing long-lived credentials on a workstation.

```bash
export AWS_REGION=ap-southeast-1
aws configure set region ap-southeast-1
aws sts get-caller-identity
aws configure get region
```

**Expected Output:**
```text
{
    "UserId": "AIDA...",
    "Account": "<AWS_ACCOUNT_ID>",
    "Arn": "arn:aws:iam::<AWS_ACCOUNT_ID>:user/..."
}
ap-southeast-1
```

![Figure 02 — Verifying AWS CLI identity (aws sts get-caller-identity) and ap-southeast-1 on CloudShell](../../../static/images/5-Workshop/5.2-Prerequisite/02-aws-identity.png)
*Figure 02 — Verifying AWS CLI identity (aws sts get-caller-identity) and ap-southeast-1 on CloudShell*

## Step 5.2.3 — Prepare the main Branch & External Integration Credentials

Verify that `chatbot_medical` (`main`) contains `scripts/docker_entrypoint.py` (which auto-ingests the 364 RAG documents on first boot) and gather external credentials to store in AWS Secrets Manager in Chapter 5.3:

| Category | Items Required | Role |
| --- | --- | --- |
| Code & Container | `main` branch (`Dockerfile` non-root `appuser`, `scripts/docker_entrypoint.py`) | Built by AWS CodeBuild into a ~334 MB ECR image (`v1.0.0`). |
| AI & Vision OCR | 4 Groq API Keys (4-Key Pool Failover separated for Chat, OCR, Nutrition & backup for `qwen/qwen3.8-27b`) | Clinical chat triage, prescription OCR reconciliation, and nutrition planning. |
| Auth & OTP | Google OAuth 2.0 Client ID/Secret, Gmail SMTP App Password, SpeedSMS Token | Google Sign-In, 6-digit OTP verification, and appointment confirmation emails. |
