---
report_key: "workshopChapters"
report_language: "en"
report_fields: {"shortTitle": "3-Tier AWS Infra", "customTitle": "5.3. Provisioning the 3-Tier AWS Infrastructure (VPC, SG, IAM, RDS, S3 & Secrets)", "summary": "Provision all 9 Multi-AZ security foundation layers sequentially via AWS CloudShell in ap-southeast-1: 6-subnet VPC, IGW, 2 symmetric NAT Gateways, 3 chained Security Groups, Least-Privilege IAM Role, RDS PostgreSQL 16.16, S3 Backup, 3 Secrets Manager groups, and S3 Gateway Endpoint.", "content": "Provisioned the complete 3-tier isolated AWS foundation in ap-southeast-1: medimind-vpc (10.0.0.0/16, 6 subnets across 2 AZs, IGW, NAT Gateway, 3 route tables), 3 chained Security Groups (ALB -> EC2 :8000 -> RDS :5432), Least-Privilege IAM Role medimind-ec2-role, RDS PostgreSQL 16.16, S3 backup lifecycle, and 3 Secrets Manager groups.", "notes": "", "steps": [{"num": "Step 5.3.1", "title": "Create VPC (10.0.0.0/16) & 6 Subnets across 2 Availability Zones", "summary": "Create `medimind-vpc` (DNS Support & Hostnames enabled) segmented into three isolated network tiers across `ap-southeast-1a` and `ap-southeast-1b`:", "table": {"headers": ["Tier", "Subnet AZ-1a", "Subnet AZ-1b", "Purpose"], "rows": [["Public Tier", "`10.0.1.0/24` (`medimind-public-a`)", "`10.0.2.0/24` (`medimind-public-b`)", "Hosts Application Load Balancer & NAT Gateway."], ["Private App Tier", "`10.0.11.0/24` (`medimind-private-app-a`)", "`10.0.12.0/24` (`medimind-private-app-b`)", "Hosts Private EC2 Docker runtime (no Public IP)."], ["Private DB Tier", "`10.0.21.0/24` (`medimind-private-db-a`)", "`10.0.22.0/24` (`medimind-private-db-b`)", "Hosts Amazon RDS PostgreSQL (zero Internet route)."]]}, "code": "export AWS_REGION=ap-southeast-1\nVPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region $AWS_REGION \\\n  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=medimind-vpc}]' --query 'Vpc.VpcId' --output text)\naws ec2 modify-vpc-attribute --vpc-id \"$VPC_ID\" --enable-dns-support '{\"Value\":true}'\naws ec2 modify-vpc-attribute --vpc-id \"$VPC_ID\" --enable-dns-hostnames '{\"Value\":true}'\n# Create all 6 subnets (public, private-app, private-db across 1a/1b) and save IDs to ~/medimind_ids.env", "expected": "VPC_ID=vpc-0173e16d7c27c35f8 (10.0.0.0/16) + 6 Subnets created across ap-southeast-1a & 1b", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03a-cli-vpc-subnets.png", "caption": "Figure 03a (CLI) — Provisioning medimind-vpc and 6 Subnets across 2 AZs in CloudShell"}]}, {"num": "Step 5.3.2", "title": "Attach Internet Gateway, NAT Gateway & 3 Isolated Route Tables", "summary": "Configure strict 3-tier routing: `rt-public` routes `0.0.0.0/0` to IGW, `rt-private-app` egresses via NAT Gateway in `public-a`, and `rt-private-db` stays strictly local.", "code": "IGW_ID=$(aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=medimind-igw}]' --query 'InternetGateway.InternetGatewayId' --output text)\naws ec2 attach-internet-gateway --vpc-id \"$VPC_ID\" --internet-gateway-id \"$IGW_ID\"\nEIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)\nNAT_ID=$(aws ec2 create-nat-gateway --subnet-id \"$PUB_A\" --allocation-id \"$EIP_ALLOC\" --query 'NatGateway.NatGatewayId' --output text)\naws ec2 wait nat-gateway-available --nat-gateway-ids \"$NAT_ID\"", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03b-cli-igw-nat.png", "caption": "Figure 03b (CLI) — Attaching Internet Gateway & provisioning NAT Gateway"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03c-cli-route-tables.png", "caption": "Figure 03c (CLI) — Creating & associating 3 isolated Route Tables"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/03-vpc-resource-map.png", "caption": "Figure 03 (Console) — medimind-vpc Resource Map verifying 6 Subnets, 3 Route Tables, IGW & NAT GW"}]}, {"num": "Step 5.3.3", "title": "Configure 3 Chained Security Groups (ALB → EC2 → RDS)", "summary": "Enforce defense-in-depth Security Group Chaining so neither EC2 nor RDS is reachable directly from the Internet:", "bullets": ["**`sg-medimind-alb` (`medimind-alb-sg`):** Allows inbound HTTP `80` and HTTPS `443`.", "**`sg-medimind-ec2` (`medimind-ec2-sg`):** Allows TCP `8000` exclusively from **`sg-medimind-alb`**. **No SSH port `22` open** (keyless SSM only).", "**`sg-medimind-rds` (`medimind-rds-sg`):** Allows TCP `5432` exclusively from **`sg-medimind-ec2`**."], "code": "SG_ALB=$(aws ec2 create-security-group --group-name medimind-alb-sg --description 'ALB SG' --vpc-id \"$VPC_ID\" --query 'GroupId' --output text)\nSG_EC2=$(aws ec2 create-security-group --group-name medimind-ec2-sg --description 'EC2 SG' --vpc-id \"$VPC_ID\" --query 'GroupId' --output text)\nSG_RDS=$(aws ec2 create-security-group --group-name medimind-rds-sg --description 'RDS SG' --vpc-id \"$VPC_ID\" --query 'GroupId' --output text)\naws ec2 authorize-security-group-ingress --group-id \"$SG_EC2\" --protocol tcp --port 8000 --source-group \"$SG_ALB\"\naws ec2 authorize-security-group-ingress --group-id \"$SG_RDS\" --protocol tcp --port 5432 --source-group \"$SG_EC2\"", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/04a-cli-sg.png", "caption": "Figure 04a (CLI) — Creating 3 Chained Security Groups on CloudShell"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/04-security-groups.png", "caption": "Figure 04 (Console) — All 3 Chained Security Groups inside medimind-vpc"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/04b-sg-ec2-inbound.png", "caption": "Figure 04b (Console) — sg-medimind-ec2 Inbound rule: TCP 8000 strictly from sg-medimind-alb (zero SSH 22)"}]}, {"num": "Step 5.3.4", "title": "Create Least-Privilege IAM Role (medimind-ec2-role)", "summary": "Eliminate static AWS keys on EC2 by attaching **`medimind-ec2-role`** with exactly three scoped policies:", "bullets": ["**`AmazonSSMManagedInstanceCore`**: Keyless administration via Systems Manager Session Manager.", "**`AmazonEC2ContainerRegistryReadOnly`**: Pull-only access to Amazon ECR images.", "**Inline `MediMindRuntimeAccess`**: Read `secret:medimind/*`, read/write `medimind-clinical-backup-*` S3 bucket, and write `/medimind/*` CloudWatch Logs."], "code": "aws iam create-role --role-name medimind-ec2-role --assume-role-policy-document file:///tmp/ec2-trust.json\naws iam attach-role-policy --role-name medimind-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore\naws iam attach-role-policy --role-name medimind-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly\naws iam put-role-policy --role-name medimind-ec2-role --policy-name MediMindRuntimeAccess --policy-document file:///tmp/runtime-policy.json", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/05a-cli-iam-role.png", "caption": "Figure 05a (CLI) — Creating medimind-ec2-role, attaching policies & Instance Profile"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/05-iam-role.png", "caption": "Figure 05 (Console) — IAM Role medimind-ec2-role with 3 Least-Privilege Policies"}]}, {"num": "Step 5.3.5", "title": "Provision Amazon RDS for PostgreSQL 16.16 (medimind-pg-db)", "summary": "Create a DB Subnet Group across both Private DB Subnets and launch PostgreSQL 16.16 (`db.t4g.micro`, `20 GiB gp3` KMS-encrypted, `Public access: No`) for MediMind's 7 ORM tables.", "code": "aws rds create-db-subnet-group --db-subnet-group-name medimind-db-subnet-group \\\n  --db-subnet-group-description 'Private DB Subnets' --subnet-ids \"$PRIV_DB_A\" \"$PRIV_DB_B\"\naws rds create-db-instance --db-instance-identifier medimind-pg-db \\\n  --db-instance-class db.t4g.micro --engine postgres --engine-version 16.16 \\\n  --master-username medimind_admin --master-user-password \"$DB_PASS\" --db-name medimind_prod \\\n  --allocated-storage 20 --storage-type gp3 --storage-encrypted \\\n  --vpc-security-group-ids \"$SG_RDS\" --db-subnet-group-name medimind-db-subnet-group --no-publicly-accessible\naws rds wait db-instance-available --db-instance-identifier medimind-pg-db", "expected": "Status: Available | Engine: PostgreSQL 16.16 | Publicly accessible: No\nEndpoint: medimind-pg-db.ctu0qcaw6hfa.ap-southeast-1.rds.amazonaws.com:5432", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/06a-cli-rds.png", "caption": "Figure 06a (CLI) — Provisioning RDS PostgreSQL 16.16 & waiting for Available state"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/06-rds-available.png", "caption": "Figure 06 (Console) — Amazon RDS medimind-pg-db Available in Private DB Subnet"}]}, {"num": "Step 5.3.6", "title": "Create Amazon S3 Backup Bucket with Versioning & Lifecycle Policy", "summary": "Create `medimind-clinical-backup-<AWS_ACCOUNT_ID>` with Block All Public Access, Versioning, and automated archival tiering (`30d STANDARD_IA` → `90d GLACIER`).", "code": "aws s3api create-bucket --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --region ap-southeast-1 \\\n  --create-bucket-configuration LocationConstraint=ap-southeast-1\naws s3api put-public-access-block --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> \\\n  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true\naws s3api put-bucket-versioning --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --versioning-configuration Status=Enabled\naws s3api put-bucket-lifecycle-configuration --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --lifecycle-configuration file:///tmp/s3-lifecycle.json", "expected": "Block Public Access: ON | Versioning: Enabled | Lifecycle: 30d STANDARD_IA -> 90d GLACIER", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/07a-cli-s3.png", "caption": "Figure 07a (CLI) — Creating S3 Backup Bucket with Public Access Block, Versioning & Lifecycle"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/07-s3-backup.png", "caption": "Figure 07 (Console) — S3 Backup Bucket with MediMindBackupArchiveTiering Lifecycle Rule"}]}, {"num": "Step 5.3.7", "title": "Create 3 AWS Secrets Manager Groups (Zero CLI Credential Exposure)", "summary": "Segment runtime secrets into three groups for clean key rotation and access governance:", "bullets": ["**`medimind/app-db`**: Auto-generated 64-hex `SECRET_KEY` and RDS PostgreSQL `DATABASE_URL`.", "**`medimind/auth-otp`**: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, Gmail SMTP, and SpeedSMS credentials.", "**`medimind/ai-providers`**: 4 Groq API Keys (4-Key Pool Failover for Chat, OCR, Nutrition & Backup) and `GROQ_MODEL=qwen/qwen3.8-27b`."], "code": "aws secretsmanager create-secret --name medimind/app-db --secret-string \"{\\\"SECRET_KEY\\\":\\\"$APP_SECRET\\\",\\\"DATABASE_URL\\\":\\\"postgresql+psycopg2://medimind_admin:$DB_PASS@$RDS_ENDPOINT:5432/medimind_prod\\\"}\"\n# Create safe placeholders for auth-otp & ai-providers, then fill real keys in AWS Console -> Edit", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/08a-cli-secret-app-db.png", "caption": "Figure 08a (CLI) — Auto-generating medimind/app-db and placeholder templates"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/08-secrets-manager.png", "caption": "Figure 08 (Console) — All 3 AWS Secrets Manager groups ready for EC2 runtime (values masked)"}]}, {"title": "Create VPC S3 Gateway Endpoint ($0 — Bypass NAT Gateway Data Charges)", "summary": "Create `medimind-s3-gateway-endpoint` (`com.amazonaws.ap-southeast-1.s3`, `Gateway` type) attached directly to `rt-private-app` (2 Private App Subnets) so all S3 backup/artifact traffic stays on the private AWS backbone with zero NAT Gateway data processing charges.", "code": "export AWS_REGION=ap-southeast-1\nVPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters \"Name=cidr,Values=10.0.0.0/16\" --query 'Vpcs[0].VpcId' --output text)\nPRIV_APP_SUBNETS=$(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24\" --query 'Subnets[*].SubnetId' --output text)\nRT_PRIV_APP=$(aws ec2 describe-route-tables --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=association.subnet-id,Values=$(echo $PRIV_APP_SUBNETS | tr ' ' ',')\" --query 'RouteTables[*].RouteTableId' --output text | tr '\\t' '\\n' | sort -u | tr '\\n' ' ')\n\nVPCE_ID=$(aws ec2 create-vpc-endpoint --region $AWS_REGION --vpc-id \"$VPC_ID\" --vpc-endpoint-type Gateway --service-name \"com.amazonaws.$AWS_REGION.s3\" --route-table-ids $RT_PRIV_APP --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=medimind-s3-gateway-endpoint}]' --query 'VpcEndpoint.VpcEndpointId' --output text)\naws ec2 describe-vpc-endpoints --region $AWS_REGION --vpc-endpoint-ids \"$VPCE_ID\" --query 'VpcEndpoints[0].{EndpointId:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,State:State,RouteTables:RouteTableIds}' --output table", "expected": "EndpointId: vpce-02f65f450b37931e5 | Service: com.amazonaws.ap-southeast-1.s3 | State: available | Type: Gateway | RouteTables: rtb-0494923986fe92b92", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/15a-cli-s3-vpce.png", "caption": "Figure 15a (CLI) — Creating VPC S3 Gateway Endpoint attached to rt-private-app on AWS CloudShell"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/15-s3-vpce.png", "caption": "Figure 15 (Console) — VPC Endpoint medimind-s3-gateway-endpoint Available across 2 Private App Subnets"}], "num": "Step 5.3.8"}, {"num": "Step 5.3.9", "title": "Deploy NAT Gateway 2 (ap-southeast-1b) & Symmetric Dual-AZ Route Tables", "summary": "Eliminate single-AZ outbound risk by deploying **NAT Gateway 2 (`medimind-nat-gw-b`)** in `Public Subnet B` (`10.0.2.0/24`), associating `Private App Subnet B` (`10.0.12.0/24`) with a dedicated `rt-private-app-b` route table pointing to NAT Gateway 2, and attaching the S3 Gateway Endpoint to both private route tables.", "code": "export AWS_REGION=ap-southeast-1\nVPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters \"Name=cidr,Values=10.0.0.0/16\" --query 'Vpcs[0].VpcId' --output text)\nPUB_B=$(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.2.0/24\" --query 'Subnets[0].SubnetId' --output text)\nPRIV_APP_B=$(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.12.0/24\" --query 'Subnets[0].SubnetId' --output text)\nVPCE_ID=$(aws ec2 describe-vpc-endpoints --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=vpc-endpoint-type,Values=Gateway\" --query 'VpcEndpoints[0].VpcEndpointId' --output text)\n\n# 1. Allocate Elastic IP & create NAT Gateway 2 in Public Subnet B (ap-southeast-1b)\nEIP_B=$(aws ec2 allocate-address --region $AWS_REGION --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=medimind-nat-eip-b}]' --query 'AllocationId' --output text)\nNAT_B=$(aws ec2 create-nat-gateway --region $AWS_REGION --subnet-id \"$PUB_B\" --allocation-id \"$EIP_B\" --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=medimind-nat-gw-b}]' --query 'NatGateway.NatGatewayId' --output text)\naws ec2 wait nat-gateway-available --region $AWS_REGION --nat-gateway-ids \"$NAT_B\"\n\n# 2. Create dedicated rt-private-app-b -> NAT Gateway 2 & attach S3 Gateway Endpoint\nRT_B=$(aws ec2 create-route-table --region $AWS_REGION --vpc-id \"$VPC_ID\" --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-private-app-b}]' --query 'RouteTable.RouteTableId' --output text)\naws ec2 create-route --region $AWS_REGION --route-table-id \"$RT_B\" --destination-cidr-block 0.0.0.0/0 --nat-gateway-id \"$NAT_B\" >/dev/null\nASSOC_B=$(aws ec2 describe-route-tables --region $AWS_REGION --filters \"Name=association.subnet-id,Values=$PRIV_APP_B\" --query \"RouteTables[0].Associations[?SubnetId=='$PRIV_APP_B'].RouteTableAssociationId\" --output text)\naws ec2 replace-route-table-association --region $AWS_REGION --association-id \"$ASSOC_B\" --route-table-id \"$RT_B\" >/dev/null\naws ec2 modify-vpc-endpoint --region $AWS_REGION --vpc-endpoint-id \"$VPCE_ID\" --add-route-table-ids \"$RT_B\" >/dev/null\n\n# 3. Verify both Dual-AZ NAT Gateways are Available\naws ec2 describe-nat-gateways --region $AWS_REGION --filter \"Name=vpc-id,Values=$VPC_ID\" \"Name=state,Values=available\" --query 'NatGateways[*].{NatId:NatGatewayId,Name:Tags[?Key==`Name`]|[0].Value,Subnet:SubnetId,PublicIP:NatGatewayAddresses[0].PublicIp,State:State}' --output table", "expected": "medimind-nat-gw   | nat-00781023e54fbbc54 | available | Public Subnet A (AZ-1a)\nmedimind-nat-gw-b | nat-07bbbfc1d6a05203e | available | Public Subnet B (AZ-1b)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/20a-cli-nat-gw-2.png", "caption": "Figure 20a (CLI) — Creating NAT Gateway 2, rt-private-app-b & verifying Dual-AZ NAT Gateways"}, {"url": "static/images/5-Workshop/5.5-Operations/20-nat-gw-2.png", "caption": "Figure 20 (Console) — VPC NAT Gateways confirming both medimind-nat-gw & medimind-nat-gw-b Available"}]}]}
report_index: 2
report_shared: {"code": "5.3"}
title: "3-Tier AWS Infra"
date: "2026-11-27"
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

# 5.3. Provisioning the 3-Tier AWS Infrastructure (VPC, SG, IAM, RDS, S3 & Secrets)

Provision all 9 Multi-AZ security foundation layers sequentially via AWS CloudShell in ap-southeast-1: 6-subnet VPC, IGW, 2 symmetric NAT Gateways, 3 chained Security Groups, Least-Privilege IAM Role, RDS PostgreSQL 16.16, S3 Backup, 3 Secrets Manager groups, and S3 Gateway Endpoint.

## Step 5.3.1 — Create VPC (10.0.0.0/16) & 6 Subnets across 2 Availability Zones

Create `medimind-vpc` (DNS Support & Hostnames enabled) segmented into three isolated network tiers across `ap-southeast-1a` and `ap-southeast-1b`:

| Tier | Subnet AZ-1a | Subnet AZ-1b | Purpose |
| --- | --- | --- | --- |
| Public Tier | `10.0.1.0/24` (`medimind-public-a`) | `10.0.2.0/24` (`medimind-public-b`) | Hosts Application Load Balancer & NAT Gateway. |
| Private App Tier | `10.0.11.0/24` (`medimind-private-app-a`) | `10.0.12.0/24` (`medimind-private-app-b`) | Hosts Private EC2 Docker runtime (no Public IP). |
| Private DB Tier | `10.0.21.0/24` (`medimind-private-db-a`) | `10.0.22.0/24` (`medimind-private-db-b`) | Hosts Amazon RDS PostgreSQL (zero Internet route). |

```bash
export AWS_REGION=ap-southeast-1
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region $AWS_REGION \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=medimind-vpc}]' --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames '{"Value":true}'
# Create all 6 subnets (public, private-app, private-db across 1a/1b) and save IDs to ~/medimind_ids.env
```

**Expected Output:**
```text
VPC_ID=vpc-0173e16d7c27c35f8 (10.0.0.0/16) + 6 Subnets created across ap-southeast-1a & 1b
```

![Figure 03a (CLI) — Provisioning medimind-vpc and 6 Subnets across 2 AZs in CloudShell](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03a-cli-vpc-subnets.png)
*Figure 03a (CLI) — Provisioning medimind-vpc and 6 Subnets across 2 AZs in CloudShell*

## Step 5.3.2 — Attach Internet Gateway, NAT Gateway & 3 Isolated Route Tables

Configure strict 3-tier routing: `rt-public` routes `0.0.0.0/0` to IGW, `rt-private-app` egresses via NAT Gateway in `public-a`, and `rt-private-db` stays strictly local.

```bash
IGW_ID=$(aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=medimind-igw}]' --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id "$VPC_ID" --internet-gateway-id "$IGW_ID"
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id "$PUB_A" --allocation-id "$EIP_ALLOC" --query 'NatGateway.NatGatewayId' --output text)
aws ec2 wait nat-gateway-available --nat-gateway-ids "$NAT_ID"
```

![Figure 03b (CLI) — Attaching Internet Gateway & provisioning NAT Gateway](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03b-cli-igw-nat.png)
*Figure 03b (CLI) — Attaching Internet Gateway & provisioning NAT Gateway*

![Figure 03c (CLI) — Creating & associating 3 isolated Route Tables](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03c-cli-route-tables.png)
*Figure 03c (CLI) — Creating & associating 3 isolated Route Tables*

![Figure 03 (Console) — medimind-vpc Resource Map verifying 6 Subnets, 3 Route Tables, IGW & NAT GW](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/03-vpc-resource-map.png)
*Figure 03 (Console) — medimind-vpc Resource Map verifying 6 Subnets, 3 Route Tables, IGW & NAT GW*

## Step 5.3.3 — Configure 3 Chained Security Groups (ALB → EC2 → RDS)

Enforce defense-in-depth Security Group Chaining so neither EC2 nor RDS is reachable directly from the Internet:

- **`sg-medimind-alb` (`medimind-alb-sg`):** Allows inbound HTTP `80` and HTTPS `443`.
- **`sg-medimind-ec2` (`medimind-ec2-sg`):** Allows TCP `8000` exclusively from **`sg-medimind-alb`**. **No SSH port `22` open** (keyless SSM only).
- **`sg-medimind-rds` (`medimind-rds-sg`):** Allows TCP `5432` exclusively from **`sg-medimind-ec2`**.

```bash
SG_ALB=$(aws ec2 create-security-group --group-name medimind-alb-sg --description 'ALB SG' --vpc-id "$VPC_ID" --query 'GroupId' --output text)
SG_EC2=$(aws ec2 create-security-group --group-name medimind-ec2-sg --description 'EC2 SG' --vpc-id "$VPC_ID" --query 'GroupId' --output text)
SG_RDS=$(aws ec2 create-security-group --group-name medimind-rds-sg --description 'RDS SG' --vpc-id "$VPC_ID" --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id "$SG_EC2" --protocol tcp --port 8000 --source-group "$SG_ALB"
aws ec2 authorize-security-group-ingress --group-id "$SG_RDS" --protocol tcp --port 5432 --source-group "$SG_EC2"
```

![Figure 04a (CLI) — Creating 3 Chained Security Groups on CloudShell](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/04a-cli-sg.png)
*Figure 04a (CLI) — Creating 3 Chained Security Groups on CloudShell*

![Figure 04 (Console) — All 3 Chained Security Groups inside medimind-vpc](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/04-security-groups.png)
*Figure 04 (Console) — All 3 Chained Security Groups inside medimind-vpc*

![Figure 04b (Console) — sg-medimind-ec2 Inbound rule: TCP 8000 strictly from sg-medimind-alb (zero SSH 22)](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/04b-sg-ec2-inbound.png)
*Figure 04b (Console) — sg-medimind-ec2 Inbound rule: TCP 8000 strictly from sg-medimind-alb (zero SSH 22)*

## Step 5.3.4 — Create Least-Privilege IAM Role (medimind-ec2-role)

Eliminate static AWS keys on EC2 by attaching **`medimind-ec2-role`** with exactly three scoped policies:

- **`AmazonSSMManagedInstanceCore`**: Keyless administration via Systems Manager Session Manager.
- **`AmazonEC2ContainerRegistryReadOnly`**: Pull-only access to Amazon ECR images.
- **Inline `MediMindRuntimeAccess`**: Read `secret:medimind/*`, read/write `medimind-clinical-backup-*` S3 bucket, and write `/medimind/*` CloudWatch Logs.

```bash
aws iam create-role --role-name medimind-ec2-role --assume-role-policy-document file:///tmp/ec2-trust.json
aws iam attach-role-policy --role-name medimind-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam attach-role-policy --role-name medimind-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam put-role-policy --role-name medimind-ec2-role --policy-name MediMindRuntimeAccess --policy-document file:///tmp/runtime-policy.json
```

![Figure 05a (CLI) — Creating medimind-ec2-role, attaching policies & Instance Profile](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/05a-cli-iam-role.png)
*Figure 05a (CLI) — Creating medimind-ec2-role, attaching policies & Instance Profile*

![Figure 05 (Console) — IAM Role medimind-ec2-role with 3 Least-Privilege Policies](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/05-iam-role.png)
*Figure 05 (Console) — IAM Role medimind-ec2-role with 3 Least-Privilege Policies*

## Step 5.3.5 — Provision Amazon RDS for PostgreSQL 16.16 (medimind-pg-db)

Create a DB Subnet Group across both Private DB Subnets and launch PostgreSQL 16.16 (`db.t4g.micro`, `20 GiB gp3` KMS-encrypted, `Public access: No`) for MediMind's 7 ORM tables.

```bash
aws rds create-db-subnet-group --db-subnet-group-name medimind-db-subnet-group \
  --db-subnet-group-description 'Private DB Subnets' --subnet-ids "$PRIV_DB_A" "$PRIV_DB_B"
aws rds create-db-instance --db-instance-identifier medimind-pg-db \
  --db-instance-class db.t4g.micro --engine postgres --engine-version 16.16 \
  --master-username medimind_admin --master-user-password "$DB_PASS" --db-name medimind_prod \
  --allocated-storage 20 --storage-type gp3 --storage-encrypted \
  --vpc-security-group-ids "$SG_RDS" --db-subnet-group-name medimind-db-subnet-group --no-publicly-accessible
aws rds wait db-instance-available --db-instance-identifier medimind-pg-db
```

**Expected Output:**
```text
Status: Available | Engine: PostgreSQL 16.16 | Publicly accessible: No
Endpoint: medimind-pg-db.ctu0qcaw6hfa.ap-southeast-1.rds.amazonaws.com:5432
```

![Figure 06a (CLI) — Provisioning RDS PostgreSQL 16.16 & waiting for Available state](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/06a-cli-rds.png)
*Figure 06a (CLI) — Provisioning RDS PostgreSQL 16.16 & waiting for Available state*

![Figure 06 (Console) — Amazon RDS medimind-pg-db Available in Private DB Subnet](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/06-rds-available.png)
*Figure 06 (Console) — Amazon RDS medimind-pg-db Available in Private DB Subnet*

## Step 5.3.6 — Create Amazon S3 Backup Bucket with Versioning & Lifecycle Policy

Create `medimind-clinical-backup-<AWS_ACCOUNT_ID>` with Block All Public Access, Versioning, and automated archival tiering (`30d STANDARD_IA` → `90d GLACIER`).

```bash
aws s3api create-bucket --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1
aws s3api put-public-access-block --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-versioning --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --versioning-configuration Status=Enabled
aws s3api put-bucket-lifecycle-configuration --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --lifecycle-configuration file:///tmp/s3-lifecycle.json
```

**Expected Output:**
```text
Block Public Access: ON | Versioning: Enabled | Lifecycle: 30d STANDARD_IA -> 90d GLACIER
```

![Figure 07a (CLI) — Creating S3 Backup Bucket with Public Access Block, Versioning & Lifecycle](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/07a-cli-s3.png)
*Figure 07a (CLI) — Creating S3 Backup Bucket with Public Access Block, Versioning & Lifecycle*

![Figure 07 (Console) — S3 Backup Bucket with MediMindBackupArchiveTiering Lifecycle Rule](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/07-s3-backup.png)
*Figure 07 (Console) — S3 Backup Bucket with MediMindBackupArchiveTiering Lifecycle Rule*

## Step 5.3.7 — Create 3 AWS Secrets Manager Groups (Zero CLI Credential Exposure)

Segment runtime secrets into three groups for clean key rotation and access governance:

- **`medimind/app-db`**: Auto-generated 64-hex `SECRET_KEY` and RDS PostgreSQL `DATABASE_URL`.
- **`medimind/auth-otp`**: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, Gmail SMTP, and SpeedSMS credentials.
- **`medimind/ai-providers`**: 4 Groq API Keys (4-Key Pool Failover for Chat, OCR, Nutrition & Backup) and `GROQ_MODEL=qwen/qwen3.8-27b`.

```bash
aws secretsmanager create-secret --name medimind/app-db --secret-string "{\"SECRET_KEY\":\"$APP_SECRET\",\"DATABASE_URL\":\"postgresql+psycopg2://medimind_admin:$DB_PASS@$RDS_ENDPOINT:5432/medimind_prod\"}"
# Create safe placeholders for auth-otp & ai-providers, then fill real keys in AWS Console -> Edit
```

![Figure 08a (CLI) — Auto-generating medimind/app-db and placeholder templates](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/08a-cli-secret-app-db.png)
*Figure 08a (CLI) — Auto-generating medimind/app-db and placeholder templates*

![Figure 08 (Console) — All 3 AWS Secrets Manager groups ready for EC2 runtime (values masked)](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/08-secrets-manager.png)
*Figure 08 (Console) — All 3 AWS Secrets Manager groups ready for EC2 runtime (values masked)*

## Step 5.3.8 — Create VPC S3 Gateway Endpoint ($0 — Bypass NAT Gateway Data Charges)

Create `medimind-s3-gateway-endpoint` (`com.amazonaws.ap-southeast-1.s3`, `Gateway` type) attached directly to `rt-private-app` (2 Private App Subnets) so all S3 backup/artifact traffic stays on the private AWS backbone with zero NAT Gateway data processing charges.

```bash
export AWS_REGION=ap-southeast-1
VPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters "Name=cidr,Values=10.0.0.0/16" --query 'Vpcs[0].VpcId' --output text)
PRIV_APP_SUBNETS=$(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24" --query 'Subnets[*].SubnetId' --output text)
RT_PRIV_APP=$(aws ec2 describe-route-tables --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.subnet-id,Values=$(echo $PRIV_APP_SUBNETS | tr ' ' ',')" --query 'RouteTables[*].RouteTableId' --output text | tr '\t' '\n' | sort -u | tr '\n' ' ')

VPCE_ID=$(aws ec2 create-vpc-endpoint --region $AWS_REGION --vpc-id "$VPC_ID" --vpc-endpoint-type Gateway --service-name "com.amazonaws.$AWS_REGION.s3" --route-table-ids $RT_PRIV_APP --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=medimind-s3-gateway-endpoint}]' --query 'VpcEndpoint.VpcEndpointId' --output text)
aws ec2 describe-vpc-endpoints --region $AWS_REGION --vpc-endpoint-ids "$VPCE_ID" --query 'VpcEndpoints[0].{EndpointId:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,State:State,RouteTables:RouteTableIds}' --output table
```

**Expected Output:**
```text
EndpointId: vpce-02f65f450b37931e5 | Service: com.amazonaws.ap-southeast-1.s3 | State: available | Type: Gateway | RouteTables: rtb-0494923986fe92b92
```

![Figure 15a (CLI) — Creating VPC S3 Gateway Endpoint attached to rt-private-app on AWS CloudShell](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/15a-cli-s3-vpce.png)
*Figure 15a (CLI) — Creating VPC S3 Gateway Endpoint attached to rt-private-app on AWS CloudShell*

![Figure 15 (Console) — VPC Endpoint medimind-s3-gateway-endpoint Available across 2 Private App Subnets](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/15-s3-vpce.png)
*Figure 15 (Console) — VPC Endpoint medimind-s3-gateway-endpoint Available across 2 Private App Subnets*

## Step 5.3.9 — Deploy NAT Gateway 2 (ap-southeast-1b) & Symmetric Dual-AZ Route Tables

Eliminate single-AZ outbound risk by deploying **NAT Gateway 2 (`medimind-nat-gw-b`)** in `Public Subnet B` (`10.0.2.0/24`), associating `Private App Subnet B` (`10.0.12.0/24`) with a dedicated `rt-private-app-b` route table pointing to NAT Gateway 2, and attaching the S3 Gateway Endpoint to both private route tables.

```bash
export AWS_REGION=ap-southeast-1
VPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters "Name=cidr,Values=10.0.0.0/16" --query 'Vpcs[0].VpcId' --output text)
PUB_B=$(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.2.0/24" --query 'Subnets[0].SubnetId' --output text)
PRIV_APP_B=$(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.12.0/24" --query 'Subnets[0].SubnetId' --output text)
VPCE_ID=$(aws ec2 describe-vpc-endpoints --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=vpc-endpoint-type,Values=Gateway" --query 'VpcEndpoints[0].VpcEndpointId' --output text)

# 1. Allocate Elastic IP & create NAT Gateway 2 in Public Subnet B (ap-southeast-1b)
EIP_B=$(aws ec2 allocate-address --region $AWS_REGION --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=medimind-nat-eip-b}]' --query 'AllocationId' --output text)
NAT_B=$(aws ec2 create-nat-gateway --region $AWS_REGION --subnet-id "$PUB_B" --allocation-id "$EIP_B" --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=medimind-nat-gw-b}]' --query 'NatGateway.NatGatewayId' --output text)
aws ec2 wait nat-gateway-available --region $AWS_REGION --nat-gateway-ids "$NAT_B"

# 2. Create dedicated rt-private-app-b -> NAT Gateway 2 & attach S3 Gateway Endpoint
RT_B=$(aws ec2 create-route-table --region $AWS_REGION --vpc-id "$VPC_ID" --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-private-app-b}]' --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --region $AWS_REGION --route-table-id "$RT_B" --destination-cidr-block 0.0.0.0/0 --nat-gateway-id "$NAT_B" >/dev/null
ASSOC_B=$(aws ec2 describe-route-tables --region $AWS_REGION --filters "Name=association.subnet-id,Values=$PRIV_APP_B" --query "RouteTables[0].Associations[?SubnetId=='$PRIV_APP_B'].RouteTableAssociationId" --output text)
aws ec2 replace-route-table-association --region $AWS_REGION --association-id "$ASSOC_B" --route-table-id "$RT_B" >/dev/null
aws ec2 modify-vpc-endpoint --region $AWS_REGION --vpc-endpoint-id "$VPCE_ID" --add-route-table-ids "$RT_B" >/dev/null

# 3. Verify both Dual-AZ NAT Gateways are Available
aws ec2 describe-nat-gateways --region $AWS_REGION --filter "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" --query 'NatGateways[*].{NatId:NatGatewayId,Name:Tags[?Key==`Name`]|[0].Value,Subnet:SubnetId,PublicIP:NatGatewayAddresses[0].PublicIp,State:State}' --output table
```

**Expected Output:**
```text
medimind-nat-gw   | nat-00781023e54fbbc54 | available | Public Subnet A (AZ-1a)
medimind-nat-gw-b | nat-07bbbfc1d6a05203e | available | Public Subnet B (AZ-1b)
```

![Figure 20a (CLI) — Creating NAT Gateway 2, rt-private-app-b & verifying Dual-AZ NAT Gateways](../../../static/images/5-Workshop/5.5-Operations/cli-steps/20a-cli-nat-gw-2.png)
*Figure 20a (CLI) — Creating NAT Gateway 2, rt-private-app-b & verifying Dual-AZ NAT Gateways*

![Figure 20 (Console) — VPC NAT Gateways confirming both medimind-nat-gw & medimind-nat-gw-b Available](../../../static/images/5-Workshop/5.5-Operations/20-nat-gw-2.png)
*Figure 20 (Console) — VPC NAT Gateways confirming both medimind-nat-gw & medimind-nat-gw-b Available*
