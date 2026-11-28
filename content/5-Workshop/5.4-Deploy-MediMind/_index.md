---
report_key: "workshopChapters"
report_language: "en"
report_fields: {"shortTitle": "Deploy MediMind AI", "customTitle": "5.4. Containerization & Deploying MediMind AI (ECR, EC2/EBS, ALB, WAF & CloudFront)", "summary": "Build and push the Docker image via CodeBuild to Amazon ECR (333.87 MB), deploy onto Private EC2 with a 20 GB KMS-encrypted EBS volume and auto-RAG bootstrap, then expose HTTPS via ALB, AWS WAF, and Amazon CloudFront.", "content": "Built and pushed medimind-clinical-ai:v1.0.0 (333.87 MB) via CodeBuild to ECR, deployed onto Private EC2 t3.small + 20 GB gp3 EBS via SSM Session Manager (auto-ingesting 364 RAG documents, /health ok, /ready ready), and exposed live HTTPS via ALB (1 Healthy), AWS WAF (4 rules), and CloudFront at https://d2typ9pphzk4as.cloudfront.net.", "notes": "", "steps": [{"num": "Step 5.4.1", "title": "Create Amazon ECR & Build Docker Image via AWS CodeBuild (333.87 MB)", "summary": "Create private repository `medimind-clinical-ai` (`scanOnPush`, AES256 encryption) and trigger **AWS CodeBuild (`medimind-docker-build`)** to build directly from GitHub `main` (`python:3.11-slim`, non-root `appuser`) and push `v1.0.0` / `latest`.", "code": "aws ecr create-repository --repository-name medimind-clinical-ai \\\n  --image-scanning-configuration scanOnPush=true --encryption-configuration encryptionType=AES256\naws codebuild start-build --project-name medimind-docker-build --region ap-southeast-1", "expected": "Build status: SUCCEEDED\nECR Image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0 (333.87 MB < 500 MB Free Tier)", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/09a-cli-ecr.png", "caption": "Figure 09a (CLI) — Creating Amazon ECR repository medimind-clinical-ai"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/09b-cli-codebuild.png", "caption": "Figure 09b (CLI) — AWS CodeBuild building & pushing Docker image to ECR"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/09-ecr-scan.png", "caption": "Figure 09 (Console) — Amazon ECR showing v1.0.0 / latest image at 333.87 MB"}]}, {"num": "Step 5.4.2", "title": "Launch Private EC2 (t3.small) + 20 GB KMS-Encrypted EBS Volume", "summary": "Launch `medimind-app-server` (Ubuntu 22.04, `t3.small`) inside `Private App Subnet A` with no Public IP, attached `sg-medimind-ec2`, IAM profile `medimind-ec2-role`, and a dedicated `20 GB gp3` encrypted EBS volume at `/dev/xvdf`.", "code": "EC2_ID=$(aws ec2 run-instances --image-id \"$AMI_ID\" --instance-type t3.small \\\n  --subnet-id \"$PRIV_APP_A\" --security-group-ids \"$SG_EC2\" \\\n  --iam-instance-profile Name=medimind-ec2-role --no-associate-public-ip-address \\\n  --block-device-mappings '[{\"DeviceName\":\"/dev/sda1\",\"Ebs\":{\"VolumeSize\":10,\"VolumeType\":\"gp3\",\"Encrypted\":true}},{\"DeviceName\":\"/dev/xvdf\",\"Ebs\":{\"VolumeSize\":20,\"VolumeType\":\"gp3\",\"Encrypted\":true}}]' \\\n  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=medimind-app-server}]' --query 'Instances[0].InstanceId' --output text)\naws ec2 wait instance-status-ok --instance-ids \"$EC2_ID\"", "expected": "Instance ID: i-09e81d49793fb5300 | Private IP: 10.0.11.95 | Public IP: None | Status: 3/3 checks passed", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/10a-cli-ec2.png", "caption": "Figure 10a (CLI) — Launching Private EC2 + 20 GB gp3 EBS volume in CloudShell"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/10-ec2-ebs.png", "caption": "Figure 10 (Console) — Private EC2 medimind-app-server Running (3/3 checks passed, no Public IP)"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/10b-ssm-connect.png", "caption": "Figure 10b (Console) — Connecting keyless to Private EC2 via SSM Session Manager"}]}, {"num": "Step 5.4.3", "title": "Mount EBS, Load Secrets & Start Auto-RAG Container via SSM Session Manager", "summary": "Connect via **SSM Session Manager**, install Docker/jq/awscli, format & mount `/dev/nvme1n1` to `/mnt/medimind-data` (`chown 1000:1000`), pull the 3 Secrets Manager groups into `/etc/medimind.env` (`chmod 600`), and run the container on `:8000`. On first boot, `scripts/docker_entrypoint.py` automatically ingests all 364 RAG documents into ChromaDB.", "code": "bash\nsudo mkfs -t ext4 /dev/nvme1n1 && sudo mkdir -p /mnt/medimind-data\nsudo mount /dev/nvme1n1 /mnt/medimind-data\nsudo mkdir -p /mnt/medimind-data/medical_vectordb /mnt/medimind-data/uploads /mnt/medimind-data/logs\nsudo chown -R 1000:1000 /mnt/medimind-data\naws ecr get-login-password --region ap-southeast-1 | sudo docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com\nsudo docker run -d --name medimind-clinical-api --restart unless-stopped -p 8000:8000 \\\n  --env-file /etc/medimind.env \\\n  -v /mnt/medimind-data/medical_vectordb:/app/medical_vectordb \\\n  -v /mnt/medimind-data/uploads:/app/uploads \\\n  -v /mnt/medimind-data/logs:/app/logs \\\n  <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0\ncurl -s http://127.0.0.1:8000/health && echo '' && curl -s http://127.0.0.1:8000/ready", "expected": "{\"status\":\"ok\"}\n{\"status\":\"ready\",\"rag\":\"ready\",\"gemini\":\"ready\",\"circuit_breaker\":\"CLOSED\"}", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/11a-ssm-mount-ebs.png", "caption": "Figure 11a (SSM) — Installing Docker, formatting ext4 & mounting 20 GB EBS at /mnt/medimind-data"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/11b-ssm-secrets-env.png", "caption": "Figure 11b (SSM) — Fetching 3 Secrets Manager groups into /etc/medimind.env (chmod 600)"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/11-ssm-health.png", "caption": "Figure 11 (SSM) — Verified EBS mount, Docker container Up, auto-ingested active_collection.json, /health ok & /ready ready"}]}, {"num": "Step 5.4.4", "title": "Create Target Group (1 Healthy) & Application Load Balancer (medimind-alb)", "summary": "Create Target Group `medimind-tg` (HTTP `:8000`, `/health` check), register EC2 `i-09e81d49793fb5300`, and provision Internet-facing `medimind-alb` across both Public Subnets.", "code": "TG_ARN=$(aws elbv2 create-target-group --name medimind-tg --protocol HTTP --port 8000 \\\n  --vpc-id \"$VPC_ID\" --target-type instance --health-check-path /health --query 'TargetGroups[0].TargetGroupArn' --output text)\naws elbv2 register-targets --target-group-arn \"$TG_ARN\" --targets Id=$EC2_ID,Port=8000\nALB_ARN=$(aws elbv2 create-load-balancer --name medimind-alb --subnets \"$PUB_A\" \"$PUB_B\" \\\n  --security-groups \"$SG_ALB\" --scheme internet-facing --type application --query 'LoadBalancers[0].LoadBalancerArn' --output text)\naws elbv2 create-listener --load-balancer-arn \"$ALB_ARN\" --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=\"$TG_ARN\"", "expected": "ALB medimind-alb: Active | Target Group medimind-tg: 1 Healthy", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/12a-cli-alb.png", "caption": "Figure 12a (CLI) — Provisioning Target Group, ALB & awaiting Healthy target"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/12-alb-tg-healthy.png", "caption": "Figure 12 (Console) — Target Group medimind-tg (:8000) reporting 1 Healthy target"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/12c-alb-active.png", "caption": "Figure 12c (Console) — Application Load Balancer medimind-alb Active across 2 AZs"}]}, {"num": "Step 5.4.5", "title": "Configure Edge Firewall AWS WAF (medimind-edge-waf — 4 Rules)", "summary": "Create a Global `CLOUDFRONT` Web ACL in **`us-east-1`** with four Layer-7 defense rules:", "bullets": ["**Rule 1 — `AWSManagedRulesCommonRuleSet`**: Blocks OWASP Top 10 & XSS; **excludes `SizeRestrictions_BODY`** so OCR prescription images (> 8 KB) upload cleanly.", "**Rule 2 — `AWSManagedRulesSQLiRuleSet`**: Blocks SQL injection payloads.", "**Rule 3 — `AWSManagedRulesKnownBadInputsRuleSet`**: Blocks known exploit patterns.", "**Rule 4 — `MediMindRateLimit`**: Caps each IP at **1,000 requests / 5 minutes**."], "code": "WAF_ARN=$(aws wafv2 create-web-acl --name medimind-edge-waf --scope CLOUDFRONT --region us-east-1 \\\n  --default-action Allow={} --rules file:///tmp/waf-rules.json \\\n  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=MediMindEdgeWAF \\\n  --query 'Summary.ARN' --output text)", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/13a-cli-waf.png", "caption": "Figure 13a (CLI) — Creating Global AWS WAF medimind-edge-waf in us-east-1"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/13-waf-rules.png", "caption": "Figure 13 (Console) — AWS WAF medimind-edge-waf showing all 4 active rules"}]}, {"num": "Step 5.4.6", "title": "Deploy Amazon CloudFront HTTPS & Sync Google OAuth / CORS", "summary": "Create CloudFront Distribution **`E3FNAGSNUILKRW`** fronting `medimind-alb` with `medimind-edge-waf`, splitting `/static/*` (`CachingOptimized`) and `/api/*` (`CachingDisabled` + `AllViewer` + `OriginReadTimeout: 60s`), then update `CORS_ALLOWED_ORIGINS` & `GOOGLE_REDIRECT_URI` on EC2 and Google Cloud Console.", "code": "aws cloudfront create-distribution --distribution-config file:///tmp/cf-config.json\naws cloudfront wait distribution-deployed --id \"$CF_ID\"\n# On EC2 SSM: update /etc/medimind.env with the CloudFront domain and recreate container\nsudo docker rm -f medimind-clinical-api && sudo docker run -d --name medimind-clinical-api --restart unless-stopped -p 8000:8000 --env-file /etc/medimind.env -v /mnt/medimind-data/medical_vectordb:/app/medical_vectordb -v /mnt/medimind-data/uploads:/app/uploads -v /mnt/medimind-data/logs:/app/logs <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0", "expected": "CloudFront ID: E3FNAGSNUILKRW | Status: Deployed\nLive HTTPS URL: https://d2typ9pphzk4as.cloudfront.net", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/14a-cli-cloudfront.png", "caption": "Figure 14a (CLI) — Creating CloudFront Distribution & obtaining the live HTTPS domain"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/14-cloudfront-route53.png", "caption": "Figure 14 (Console) — CloudFront E3FNAGSNUILKRW (Enabled) live at https://d2typ9pphzk4as.cloudfront.net"}]}, {"title": "Provision Encrypted Amazon EFS (medimind-shared-efs) & Dual-AZ Mount Targets", "summary": "Create `medimind-efs-sg` restricting NFS `TCP 2049` strictly to `medimind-ec2-sg`, create KMS-encrypted `medimind-shared-efs`, and attach Mount Targets in both `ap-southeast-1a` (`10.0.11.254`) and `ap-southeast-1b` (`10.0.12.244`) for multi-node shared ChromaDB & `/uploads` storage.", "code": "export AWS_REGION=ap-southeast-1\nVPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters \"Name=cidr,Values=10.0.0.0/16\" --query 'Vpcs[0].VpcId' --output text)\nSG_EC2=$(aws ec2 describe-security-groups --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=group-name,Values=medimind-ec2-sg\" --query 'SecurityGroups[0].GroupId' --output text)\n\n# 1. Create NFS 2049 Security Group & KMS-encrypted Amazon EFS\nSG_EFS=$(aws ec2 create-security-group --region $AWS_REGION --group-name medimind-efs-sg --description \"Allow NFS 2049 only from medimind-ec2-sg\" --vpc-id \"$VPC_ID\" --query 'GroupId' --output text)\naws ec2 authorize-security-group-ingress --region $AWS_REGION --group-id \"$SG_EFS\" --protocol tcp --port 2049 --source-group \"$SG_EC2\"\nEFS_ID=$(aws efs create-file-system --region $AWS_REGION --performance-mode generalPurpose --throughput-mode bursting --encrypted --tags Key=Name,Value=medimind-shared-efs --query 'FileSystemId' --output text)\n\n# 2. Attach Dual-AZ Mount Targets in both Private App Subnets (10.0.11.0/24 & 10.0.12.0/24)\nfor SUBNET in $(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24\" --query 'Subnets[*].SubnetId' --output text); do\n  aws efs create-mount-target --region $AWS_REGION --file-system-id \"$EFS_ID\" --subnet-id \"$SUBNET\" --security-groups \"$SG_EFS\" --output text\ndone\naws efs describe-mount-targets --region $AWS_REGION --file-system-id \"$EFS_ID\" --query 'MountTargets[*].{MountTargetId:MountTargetId,AZ:AvailabilityZoneName,SubnetId:SubnetId,IP:IpAddress,State:LifeCycleState}' --output table", "expected": "FileSystemId: fs-0d9a2cb6275ca9926 (Encrypted: True, available)\nMountTarget AZ-1a: fsmt-0aee953335ac475d4 (10.0.11.254, Available)\nMountTarget AZ-1b: fsmt-0590fef6da5225039 (10.0.12.244, Available)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/18a-cli-efs-create.png", "caption": "Figure 18a (CLI) — Creating medimind-efs-sg (TCP 2049) & KMS-encrypted Amazon EFS"}, {"url": "static/images/5-Workshop/5.5-Operations/cli-steps/18b-cli-efs-mounts.png", "caption": "Figure 18b (CLI) — Creating Dual-AZ EFS Mount Targets across ap-southeast-1a & 1b"}, {"url": "static/images/5-Workshop/5.5-Operations/18-efs-network.png", "caption": "Figure 18 (Console) — EFS Network tab confirming both Mount Targets Available"}, {"url": "static/images/5-Workshop/5.5-Operations/18c-efs-overview.png", "caption": "Figure 18c (Console) — Amazon EFS medimind-shared-efs overview (Encrypted, Regional)"}], "num": "Step 5.4.7"}, {"title": "Build Golden AMI, Launch Template & Dual-AZ Auto Scaling Group (medimind-asg)", "summary": "Capture a live Golden AMI (`medimind-clinical-ami-v1`) with `--no-reboot` (zero downtime), create IMDSv2-enforced Launch Template `medimind-app-lt`, provision **Auto Scaling Group (`medimind-asg`)** across both `ap-southeast-1a` & `1b`, attach the running EC2 instance (`Min=1, Desired=1, Max=2`, ELB Health Check), and enable a **70% CPU Target Tracking Scaling Policy**.", "code": "export AWS_REGION=ap-southeast-1\nEC2_ID=$(aws ec2 describe-instances --region $AWS_REGION --filters \"Name=tag:Name,Values=medimind-app-server\" \"Name=instance-state-name,Values=running\" --query 'Reservations[0].Instances[0].InstanceId' --output text)\nSG_EC2=$(aws ec2 describe-instances --region $AWS_REGION --instance-ids \"$EC2_ID\" --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' --output text)\nPROFILE_ARN=$(aws ec2 describe-instances --region $AWS_REGION --instance-ids \"$EC2_ID\" --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn' --output text)\n\n# 1. Capture Golden AMI (--no-reboot, zero downtime) & IMDSv2 Launch Template\nAMI_ID=$(aws ec2 create-image --region $AWS_REGION --instance-id \"$EC2_ID\" --name \"medimind-clinical-ami-v1\" --description \"MediMind Clinical AI Golden Image\" --no-reboot --query 'ImageId' --output text)\naws ec2 create-launch-template --region $AWS_REGION --launch-template-name medimind-app-lt --version-description \"MediMind t3.small Private App Launch Template\" --launch-template-data \"{\\\"ImageId\\\":\\\"$AMI_ID\\\",\\\"InstanceType\\\":\\\"t3.small\\\",\\\"SecurityGroupIds\\\":[\\\"$SG_EC2\\\"],\\\"IamInstanceProfile\\\":{\\\"Arn\\\":\\\"$PROFILE_ARN\\\"},\\\"MetadataOptions\\\":{\\\"HttpTokens\\\":\\\"required\\\",\\\"HttpEndpoint\\\":\\\"enabled\\\"}}\"\n\n# 2. REQUIRED: Wait for AMI snapshot (pending -> available) before creating ASG\naws ec2 wait image-available --image-ids \"$AMI_ID\" --region $AWS_REGION\n\n# 3. Create Dual-AZ Auto Scaling Group, attach live EC2 ($0 extra) & 70% CPU Target Tracking\nVPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters \"Name=cidr,Values=10.0.0.0/16\" --query 'Vpcs[0].VpcId' --output text)\nPRIV_APP_CSV=$(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24\" --query 'Subnets[*].SubnetId' --output text | tr '\\t' ',')\nTG_ARN=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text)\n\naws autoscaling create-auto-scaling-group --region $AWS_REGION --auto-scaling-group-name medimind-asg --launch-template 'LaunchTemplateName=medimind-app-lt,Version=$Latest' --min-size 0 --max-size 2 --desired-capacity 0 --vpc-zone-identifier \"$PRIV_APP_CSV\" --target-group-arns \"$TG_ARN\" --health-check-type ELB --health-check-grace-period 300\naws autoscaling attach-instances --region $AWS_REGION --instance-ids \"$EC2_ID\" --auto-scaling-group-name medimind-asg\naws autoscaling update-auto-scaling-group --region $AWS_REGION --auto-scaling-group-name medimind-asg --min-size 1 --max-size 2 --desired-capacity 1\naws autoscaling put-scaling-policy --region $AWS_REGION --auto-scaling-group-name medimind-asg --policy-name medimind-cpu-target-70 --policy-type TargetTrackingScaling --target-tracking-configuration '{\"PredefinedMetricSpecification\":{\"PredefinedMetricType\":\"ASGAverageCPUUtilization\"},\"TargetValue\":70.0}' --output text\n\naws autoscaling describe-auto-scaling-groups --region $AWS_REGION --auto-scaling-group-names medimind-asg --query 'AutoScalingGroups[0].{Name:AutoScalingGroupName,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,HealthCheck:HealthCheckType,AZs:AvailabilityZones}' --output table", "expected": "LaunchTemplate: lt-02ed0bffddb2a5492 (medimind-app-lt v1, ami-0858b3aafe17cd29e)\nAutoScalingGroup: medimind-asg | Min: 1 | Desired: 1 | Max: 2 | HealthCheck: ELB | AZs: ap-southeast-1a, ap-southeast-1b", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/19a-cli-launch-template.png", "caption": "Figure 19a (CLI) — Capturing Golden AMI (--no-reboot) & creating Launch Template medimind-app-lt"}, {"url": "static/images/5-Workshop/5.5-Operations/cli-steps/19b-cli-asg.png", "caption": "Figure 19b (CLI) — Creating Dual-AZ medimind-asg, attaching live EC2 & enabling 70% CPU Target Tracking"}, {"url": "static/images/5-Workshop/5.5-Operations/19-asg-console.png", "caption": "Figure 19 (Console) — Auto Scaling Group medimind-asg At desired capacity (Scaling limits 1–2)"}], "num": "Step 5.4.8"}, {"num": "Step 5.4.9", "title": "Scale Auto Scaling Group to 2 Dual-AZ EC2 Nodes & Verify 2/2 ALB Healthy", "summary": "Increase `medimind-asg` `DesiredCapacity` from `1` → **`2`** so AWS Auto Scaling automatically launches the second `t3.small` EC2 instance in **`ap-southeast-1b`** (`10.0.12.18`) from Golden AMI `medimind-clinical-ami-v1`. Docker automatically starts `medimind-clinical-api` and passes ALB `/health`, reaching **`2/2 Healthy`** across both AZs.", "code": "export AWS_REGION=ap-southeast-1\naws autoscaling set-desired-capacity --region $AWS_REGION --auto-scaling-group-name medimind-asg --desired-capacity 2\n\n# Wait for the 2nd EC2 in ap-southeast-1b to boot from Golden AMI & pass ALB /health (~90s)\nsleep 90\n\naws ec2 describe-instances --region $AWS_REGION --filters \"Name=instance-state-name,Values=running,pending\" --query 'Reservations[*].Instances[*].{Id:InstanceId,AZ:Placement.AvailabilityZone,Type:InstanceType,PrivateIP:PrivateIpAddress,State:State.Name}' --output table\n\nTG_ARN=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text)\naws elbv2 describe-target-health --region $AWS_REGION --target-group-arn \"$TG_ARN\" --query 'TargetHealthDescriptions[*].{InstanceId:Target.Id,Port:Target.Port,State:TargetHealth.State}' --output table", "expected": "EC2 #1: i-09e81d49793fb5300 | ap-southeast-1a | 10.0.11.95 | running | healthy (:8000)\nEC2 #2: i-076aaacd95d5affaf | ap-southeast-1b | 10.0.12.18 | running | healthy (:8000)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/21a-cli-2ec2-healthy.png", "caption": "Figure 21a (CLI) — Scaling medimind-asg to 2 EC2 nodes (AZ-1a & AZ-1b) & verifying 2/2 healthy targets"}, {"url": "static/images/5-Workshop/5.5-Operations/21-alb-2healthy.png", "caption": "Figure 21 (Console) — Target Group medimind-tg showing 2/2 Healthy targets balanced across both AZs"}]}], "url": "https://d2typ9pphzk4as.cloudfront.net"}
report_index: 3
report_shared: {"code": "5.4", "url": "https://d2typ9pphzk4as.cloudfront.net"}
title: "Deploy MediMind AI"
date: "2026-11-27"
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

# 5.4. Containerization & Deploying MediMind AI (ECR, EC2/EBS, ALB, WAF & CloudFront)

Build and push the Docker image via CodeBuild to Amazon ECR (333.87 MB), deploy onto Private EC2 with a 20 GB KMS-encrypted EBS volume and auto-RAG bootstrap, then expose HTTPS via ALB, AWS WAF, and Amazon CloudFront.

## Step 5.4.1 — Create Amazon ECR & Build Docker Image via AWS CodeBuild (333.87 MB)

Create private repository `medimind-clinical-ai` (`scanOnPush`, AES256 encryption) and trigger **AWS CodeBuild (`medimind-docker-build`)** to build directly from GitHub `main` (`python:3.11-slim`, non-root `appuser`) and push `v1.0.0` / `latest`.

```bash
aws ecr create-repository --repository-name medimind-clinical-ai \
  --image-scanning-configuration scanOnPush=true --encryption-configuration encryptionType=AES256
aws codebuild start-build --project-name medimind-docker-build --region ap-southeast-1
```

**Expected Output:**
```text
Build status: SUCCEEDED
ECR Image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0 (333.87 MB < 500 MB Free Tier)
```

![Figure 09a (CLI) — Creating Amazon ECR repository medimind-clinical-ai](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/09a-cli-ecr.png)
*Figure 09a (CLI) — Creating Amazon ECR repository medimind-clinical-ai*

![Figure 09b (CLI) — AWS CodeBuild building & pushing Docker image to ECR](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/09b-cli-codebuild.png)
*Figure 09b (CLI) — AWS CodeBuild building & pushing Docker image to ECR*

![Figure 09 (Console) — Amazon ECR showing v1.0.0 / latest image at 333.87 MB](../../../static/images/5-Workshop/5.4-Deploy-MediMind/09-ecr-scan.png)
*Figure 09 (Console) — Amazon ECR showing v1.0.0 / latest image at 333.87 MB*

## Step 5.4.2 — Launch Private EC2 (t3.small) + 20 GB KMS-Encrypted EBS Volume

Launch `medimind-app-server` (Ubuntu 22.04, `t3.small`) inside `Private App Subnet A` with no Public IP, attached `sg-medimind-ec2`, IAM profile `medimind-ec2-role`, and a dedicated `20 GB gp3` encrypted EBS volume at `/dev/xvdf`.

```bash
EC2_ID=$(aws ec2 run-instances --image-id "$AMI_ID" --instance-type t3.small \
  --subnet-id "$PRIV_APP_A" --security-group-ids "$SG_EC2" \
  --iam-instance-profile Name=medimind-ec2-role --no-associate-public-ip-address \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":10,"VolumeType":"gp3","Encrypted":true}},{"DeviceName":"/dev/xvdf","Ebs":{"VolumeSize":20,"VolumeType":"gp3","Encrypted":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=medimind-app-server}]' --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-status-ok --instance-ids "$EC2_ID"
```

**Expected Output:**
```text
Instance ID: i-09e81d49793fb5300 | Private IP: 10.0.11.95 | Public IP: None | Status: 3/3 checks passed
```

![Figure 10a (CLI) — Launching Private EC2 + 20 GB gp3 EBS volume in CloudShell](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/10a-cli-ec2.png)
*Figure 10a (CLI) — Launching Private EC2 + 20 GB gp3 EBS volume in CloudShell*

![Figure 10 (Console) — Private EC2 medimind-app-server Running (3/3 checks passed, no Public IP)](../../../static/images/5-Workshop/5.4-Deploy-MediMind/10-ec2-ebs.png)
*Figure 10 (Console) — Private EC2 medimind-app-server Running (3/3 checks passed, no Public IP)*

![Figure 10b (Console) — Connecting keyless to Private EC2 via SSM Session Manager](../../../static/images/5-Workshop/5.4-Deploy-MediMind/10b-ssm-connect.png)
*Figure 10b (Console) — Connecting keyless to Private EC2 via SSM Session Manager*

## Step 5.4.3 — Mount EBS, Load Secrets & Start Auto-RAG Container via SSM Session Manager

Connect via **SSM Session Manager**, install Docker/jq/awscli, format & mount `/dev/nvme1n1` to `/mnt/medimind-data` (`chown 1000:1000`), pull the 3 Secrets Manager groups into `/etc/medimind.env` (`chmod 600`), and run the container on `:8000`. On first boot, `scripts/docker_entrypoint.py` automatically ingests all 364 RAG documents into ChromaDB.

```bash
bash
sudo mkfs -t ext4 /dev/nvme1n1 && sudo mkdir -p /mnt/medimind-data
sudo mount /dev/nvme1n1 /mnt/medimind-data
sudo mkdir -p /mnt/medimind-data/medical_vectordb /mnt/medimind-data/uploads /mnt/medimind-data/logs
sudo chown -R 1000:1000 /mnt/medimind-data
aws ecr get-login-password --region ap-southeast-1 | sudo docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com
sudo docker run -d --name medimind-clinical-api --restart unless-stopped -p 8000:8000 \
  --env-file /etc/medimind.env \
  -v /mnt/medimind-data/medical_vectordb:/app/medical_vectordb \
  -v /mnt/medimind-data/uploads:/app/uploads \
  -v /mnt/medimind-data/logs:/app/logs \
  <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0
curl -s http://127.0.0.1:8000/health && echo '' && curl -s http://127.0.0.1:8000/ready
```

**Expected Output:**
```text
{"status":"ok"}
{"status":"ready","rag":"ready","gemini":"ready","circuit_breaker":"CLOSED"}
```

![Figure 11a (SSM) — Installing Docker, formatting ext4 & mounting 20 GB EBS at /mnt/medimind-data](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/11a-ssm-mount-ebs.png)
*Figure 11a (SSM) — Installing Docker, formatting ext4 & mounting 20 GB EBS at /mnt/medimind-data*

![Figure 11b (SSM) — Fetching 3 Secrets Manager groups into /etc/medimind.env (chmod 600)](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/11b-ssm-secrets-env.png)
*Figure 11b (SSM) — Fetching 3 Secrets Manager groups into /etc/medimind.env (chmod 600)*

![Figure 11 (SSM) — Verified EBS mount, Docker container Up, auto-ingested active_collection.json, /health ok & /ready ready](../../../static/images/5-Workshop/5.4-Deploy-MediMind/11-ssm-health.png)
*Figure 11 (SSM) — Verified EBS mount, Docker container Up, auto-ingested active_collection.json, /health ok & /ready ready*

## Step 5.4.4 — Create Target Group (1 Healthy) & Application Load Balancer (medimind-alb)

Create Target Group `medimind-tg` (HTTP `:8000`, `/health` check), register EC2 `i-09e81d49793fb5300`, and provision Internet-facing `medimind-alb` across both Public Subnets.

```bash
TG_ARN=$(aws elbv2 create-target-group --name medimind-tg --protocol HTTP --port 8000 \
  --vpc-id "$VPC_ID" --target-type instance --health-check-path /health --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 register-targets --target-group-arn "$TG_ARN" --targets Id=$EC2_ID,Port=8000
ALB_ARN=$(aws elbv2 create-load-balancer --name medimind-alb --subnets "$PUB_A" "$PUB_B" \
  --security-groups "$SG_ALB" --scheme internet-facing --type application --query 'LoadBalancers[0].LoadBalancerArn' --output text)
aws elbv2 create-listener --load-balancer-arn "$ALB_ARN" --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn="$TG_ARN"
```

**Expected Output:**
```text
ALB medimind-alb: Active | Target Group medimind-tg: 1 Healthy
```

![Figure 12a (CLI) — Provisioning Target Group, ALB & awaiting Healthy target](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/12a-cli-alb.png)
*Figure 12a (CLI) — Provisioning Target Group, ALB & awaiting Healthy target*

![Figure 12 (Console) — Target Group medimind-tg (:8000) reporting 1 Healthy target](../../../static/images/5-Workshop/5.4-Deploy-MediMind/12-alb-tg-healthy.png)
*Figure 12 (Console) — Target Group medimind-tg (:8000) reporting 1 Healthy target*

![Figure 12c (Console) — Application Load Balancer medimind-alb Active across 2 AZs](../../../static/images/5-Workshop/5.4-Deploy-MediMind/12c-alb-active.png)
*Figure 12c (Console) — Application Load Balancer medimind-alb Active across 2 AZs*

## Step 5.4.5 — Configure Edge Firewall AWS WAF (medimind-edge-waf — 4 Rules)

Create a Global `CLOUDFRONT` Web ACL in **`us-east-1`** with four Layer-7 defense rules:

- **Rule 1 — `AWSManagedRulesCommonRuleSet`**: Blocks OWASP Top 10 & XSS; **excludes `SizeRestrictions_BODY`** so OCR prescription images (> 8 KB) upload cleanly.
- **Rule 2 — `AWSManagedRulesSQLiRuleSet`**: Blocks SQL injection payloads.
- **Rule 3 — `AWSManagedRulesKnownBadInputsRuleSet`**: Blocks known exploit patterns.
- **Rule 4 — `MediMindRateLimit`**: Caps each IP at **1,000 requests / 5 minutes**.

```bash
WAF_ARN=$(aws wafv2 create-web-acl --name medimind-edge-waf --scope CLOUDFRONT --region us-east-1 \
  --default-action Allow={} --rules file:///tmp/waf-rules.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=MediMindEdgeWAF \
  --query 'Summary.ARN' --output text)
```

![Figure 13a (CLI) — Creating Global AWS WAF medimind-edge-waf in us-east-1](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/13a-cli-waf.png)
*Figure 13a (CLI) — Creating Global AWS WAF medimind-edge-waf in us-east-1*

![Figure 13 (Console) — AWS WAF medimind-edge-waf showing all 4 active rules](../../../static/images/5-Workshop/5.4-Deploy-MediMind/13-waf-rules.png)
*Figure 13 (Console) — AWS WAF medimind-edge-waf showing all 4 active rules*

## Step 5.4.6 — Deploy Amazon CloudFront HTTPS & Sync Google OAuth / CORS

Create CloudFront Distribution **`E3FNAGSNUILKRW`** fronting `medimind-alb` with `medimind-edge-waf`, splitting `/static/*` (`CachingOptimized`) and `/api/*` (`CachingDisabled` + `AllViewer` + `OriginReadTimeout: 60s`), then update `CORS_ALLOWED_ORIGINS` & `GOOGLE_REDIRECT_URI` on EC2 and Google Cloud Console.

```bash
aws cloudfront create-distribution --distribution-config file:///tmp/cf-config.json
aws cloudfront wait distribution-deployed --id "$CF_ID"
# On EC2 SSM: update /etc/medimind.env with the CloudFront domain and recreate container
sudo docker rm -f medimind-clinical-api && sudo docker run -d --name medimind-clinical-api --restart unless-stopped -p 8000:8000 --env-file /etc/medimind.env -v /mnt/medimind-data/medical_vectordb:/app/medical_vectordb -v /mnt/medimind-data/uploads:/app/uploads -v /mnt/medimind-data/logs:/app/logs <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0
```

**Expected Output:**
```text
CloudFront ID: E3FNAGSNUILKRW | Status: Deployed
Live HTTPS URL: https://d2typ9pphzk4as.cloudfront.net
```

![Figure 14a (CLI) — Creating CloudFront Distribution & obtaining the live HTTPS domain](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/14a-cli-cloudfront.png)
*Figure 14a (CLI) — Creating CloudFront Distribution & obtaining the live HTTPS domain*

![Figure 14 (Console) — CloudFront E3FNAGSNUILKRW (Enabled) live at https://d2typ9pphzk4as.cloudfront.net](../../../static/images/5-Workshop/5.4-Deploy-MediMind/14-cloudfront-route53.png)
*Figure 14 (Console) — CloudFront E3FNAGSNUILKRW (Enabled) live at https://d2typ9pphzk4as.cloudfront.net*

## Step 5.4.7 — Provision Encrypted Amazon EFS (medimind-shared-efs) & Dual-AZ Mount Targets

Create `medimind-efs-sg` restricting NFS `TCP 2049` strictly to `medimind-ec2-sg`, create KMS-encrypted `medimind-shared-efs`, and attach Mount Targets in both `ap-southeast-1a` (`10.0.11.254`) and `ap-southeast-1b` (`10.0.12.244`) for multi-node shared ChromaDB & `/uploads` storage.

```bash
export AWS_REGION=ap-southeast-1
VPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters "Name=cidr,Values=10.0.0.0/16" --query 'Vpcs[0].VpcId' --output text)
SG_EC2=$(aws ec2 describe-security-groups --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=medimind-ec2-sg" --query 'SecurityGroups[0].GroupId' --output text)

# 1. Create NFS 2049 Security Group & KMS-encrypted Amazon EFS
SG_EFS=$(aws ec2 create-security-group --region $AWS_REGION --group-name medimind-efs-sg --description "Allow NFS 2049 only from medimind-ec2-sg" --vpc-id "$VPC_ID" --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --region $AWS_REGION --group-id "$SG_EFS" --protocol tcp --port 2049 --source-group "$SG_EC2"
EFS_ID=$(aws efs create-file-system --region $AWS_REGION --performance-mode generalPurpose --throughput-mode bursting --encrypted --tags Key=Name,Value=medimind-shared-efs --query 'FileSystemId' --output text)

# 2. Attach Dual-AZ Mount Targets in both Private App Subnets (10.0.11.0/24 & 10.0.12.0/24)
for SUBNET in $(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24" --query 'Subnets[*].SubnetId' --output text); do
  aws efs create-mount-target --region $AWS_REGION --file-system-id "$EFS_ID" --subnet-id "$SUBNET" --security-groups "$SG_EFS" --output text
done
aws efs describe-mount-targets --region $AWS_REGION --file-system-id "$EFS_ID" --query 'MountTargets[*].{MountTargetId:MountTargetId,AZ:AvailabilityZoneName,SubnetId:SubnetId,IP:IpAddress,State:LifeCycleState}' --output table
```

**Expected Output:**
```text
FileSystemId: fs-0d9a2cb6275ca9926 (Encrypted: True, available)
MountTarget AZ-1a: fsmt-0aee953335ac475d4 (10.0.11.254, Available)
MountTarget AZ-1b: fsmt-0590fef6da5225039 (10.0.12.244, Available)
```

![Figure 18a (CLI) — Creating medimind-efs-sg (TCP 2049) & KMS-encrypted Amazon EFS](../../../static/images/5-Workshop/5.5-Operations/cli-steps/18a-cli-efs-create.png)
*Figure 18a (CLI) — Creating medimind-efs-sg (TCP 2049) & KMS-encrypted Amazon EFS*

![Figure 18b (CLI) — Creating Dual-AZ EFS Mount Targets across ap-southeast-1a & 1b](../../../static/images/5-Workshop/5.5-Operations/cli-steps/18b-cli-efs-mounts.png)
*Figure 18b (CLI) — Creating Dual-AZ EFS Mount Targets across ap-southeast-1a & 1b*

![Figure 18 (Console) — EFS Network tab confirming both Mount Targets Available](../../../static/images/5-Workshop/5.5-Operations/18-efs-network.png)
*Figure 18 (Console) — EFS Network tab confirming both Mount Targets Available*

![Figure 18c (Console) — Amazon EFS medimind-shared-efs overview (Encrypted, Regional)](../../../static/images/5-Workshop/5.5-Operations/18c-efs-overview.png)
*Figure 18c (Console) — Amazon EFS medimind-shared-efs overview (Encrypted, Regional)*

## Step 5.4.8 — Build Golden AMI, Launch Template & Dual-AZ Auto Scaling Group (medimind-asg)

Capture a live Golden AMI (`medimind-clinical-ami-v1`) with `--no-reboot` (zero downtime), create IMDSv2-enforced Launch Template `medimind-app-lt`, provision **Auto Scaling Group (`medimind-asg`)** across both `ap-southeast-1a` & `1b`, attach the running EC2 instance (`Min=1, Desired=1, Max=2`, ELB Health Check), and enable a **70% CPU Target Tracking Scaling Policy**.

```bash
export AWS_REGION=ap-southeast-1
EC2_ID=$(aws ec2 describe-instances --region $AWS_REGION --filters "Name=tag:Name,Values=medimind-app-server" "Name=instance-state-name,Values=running" --query 'Reservations[0].Instances[0].InstanceId' --output text)
SG_EC2=$(aws ec2 describe-instances --region $AWS_REGION --instance-ids "$EC2_ID" --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' --output text)
PROFILE_ARN=$(aws ec2 describe-instances --region $AWS_REGION --instance-ids "$EC2_ID" --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn' --output text)

# 1. Capture Golden AMI (--no-reboot, zero downtime) & IMDSv2 Launch Template
AMI_ID=$(aws ec2 create-image --region $AWS_REGION --instance-id "$EC2_ID" --name "medimind-clinical-ami-v1" --description "MediMind Clinical AI Golden Image" --no-reboot --query 'ImageId' --output text)
aws ec2 create-launch-template --region $AWS_REGION --launch-template-name medimind-app-lt --version-description "MediMind t3.small Private App Launch Template" --launch-template-data "{\"ImageId\":\"$AMI_ID\",\"InstanceType\":\"t3.small\",\"SecurityGroupIds\":[\"$SG_EC2\"],\"IamInstanceProfile\":{\"Arn\":\"$PROFILE_ARN\"},\"MetadataOptions\":{\"HttpTokens\":\"required\",\"HttpEndpoint\":\"enabled\"}}"

# 2. REQUIRED: Wait for AMI snapshot (pending -> available) before creating ASG
aws ec2 wait image-available --image-ids "$AMI_ID" --region $AWS_REGION

# 3. Create Dual-AZ Auto Scaling Group, attach live EC2 ($0 extra) & 70% CPU Target Tracking
VPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters "Name=cidr,Values=10.0.0.0/16" --query 'Vpcs[0].VpcId' --output text)
PRIV_APP_CSV=$(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24" --query 'Subnets[*].SubnetId' --output text | tr '\t' ',')
TG_ARN=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text)

aws autoscaling create-auto-scaling-group --region $AWS_REGION --auto-scaling-group-name medimind-asg --launch-template 'LaunchTemplateName=medimind-app-lt,Version=$Latest' --min-size 0 --max-size 2 --desired-capacity 0 --vpc-zone-identifier "$PRIV_APP_CSV" --target-group-arns "$TG_ARN" --health-check-type ELB --health-check-grace-period 300
aws autoscaling attach-instances --region $AWS_REGION --instance-ids "$EC2_ID" --auto-scaling-group-name medimind-asg
aws autoscaling update-auto-scaling-group --region $AWS_REGION --auto-scaling-group-name medimind-asg --min-size 1 --max-size 2 --desired-capacity 1
aws autoscaling put-scaling-policy --region $AWS_REGION --auto-scaling-group-name medimind-asg --policy-name medimind-cpu-target-70 --policy-type TargetTrackingScaling --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":70.0}' --output text

aws autoscaling describe-auto-scaling-groups --region $AWS_REGION --auto-scaling-group-names medimind-asg --query 'AutoScalingGroups[0].{Name:AutoScalingGroupName,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,HealthCheck:HealthCheckType,AZs:AvailabilityZones}' --output table
```

**Expected Output:**
```text
LaunchTemplate: lt-02ed0bffddb2a5492 (medimind-app-lt v1, ami-0858b3aafe17cd29e)
AutoScalingGroup: medimind-asg | Min: 1 | Desired: 1 | Max: 2 | HealthCheck: ELB | AZs: ap-southeast-1a, ap-southeast-1b
```

![Figure 19a (CLI) — Capturing Golden AMI (--no-reboot) & creating Launch Template medimind-app-lt](../../../static/images/5-Workshop/5.5-Operations/cli-steps/19a-cli-launch-template.png)
*Figure 19a (CLI) — Capturing Golden AMI (--no-reboot) & creating Launch Template medimind-app-lt*

![Figure 19b (CLI) — Creating Dual-AZ medimind-asg, attaching live EC2 & enabling 70% CPU Target Tracking](../../../static/images/5-Workshop/5.5-Operations/cli-steps/19b-cli-asg.png)
*Figure 19b (CLI) — Creating Dual-AZ medimind-asg, attaching live EC2 & enabling 70% CPU Target Tracking*

![Figure 19 (Console) — Auto Scaling Group medimind-asg At desired capacity (Scaling limits 1–2)](../../../static/images/5-Workshop/5.5-Operations/19-asg-console.png)
*Figure 19 (Console) — Auto Scaling Group medimind-asg At desired capacity (Scaling limits 1–2)*

## Step 5.4.9 — Scale Auto Scaling Group to 2 Dual-AZ EC2 Nodes & Verify 2/2 ALB Healthy

Increase `medimind-asg` `DesiredCapacity` from `1` → **`2`** so AWS Auto Scaling automatically launches the second `t3.small` EC2 instance in **`ap-southeast-1b`** (`10.0.12.18`) from Golden AMI `medimind-clinical-ami-v1`. Docker automatically starts `medimind-clinical-api` and passes ALB `/health`, reaching **`2/2 Healthy`** across both AZs.

```bash
export AWS_REGION=ap-southeast-1
aws autoscaling set-desired-capacity --region $AWS_REGION --auto-scaling-group-name medimind-asg --desired-capacity 2

# Wait for the 2nd EC2 in ap-southeast-1b to boot from Golden AMI & pass ALB /health (~90s)
sleep 90

aws ec2 describe-instances --region $AWS_REGION --filters "Name=instance-state-name,Values=running,pending" --query 'Reservations[*].Instances[*].{Id:InstanceId,AZ:Placement.AvailabilityZone,Type:InstanceType,PrivateIP:PrivateIpAddress,State:State.Name}' --output table

TG_ARN=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 describe-target-health --region $AWS_REGION --target-group-arn "$TG_ARN" --query 'TargetHealthDescriptions[*].{InstanceId:Target.Id,Port:Target.Port,State:TargetHealth.State}' --output table
```

**Expected Output:**
```text
EC2 #1: i-09e81d49793fb5300 | ap-southeast-1a | 10.0.11.95 | running | healthy (:8000)
EC2 #2: i-076aaacd95d5affaf | ap-southeast-1b | 10.0.12.18 | running | healthy (:8000)
```

![Figure 21a (CLI) — Scaling medimind-asg to 2 EC2 nodes (AZ-1a & AZ-1b) & verifying 2/2 healthy targets](../../../static/images/5-Workshop/5.5-Operations/cli-steps/21a-cli-2ec2-healthy.png)
*Figure 21a (CLI) — Scaling medimind-asg to 2 EC2 nodes (AZ-1a & AZ-1b) & verifying 2/2 healthy targets*

![Figure 21 (Console) — Target Group medimind-tg showing 2/2 Healthy targets balanced across both AZs](../../../static/images/5-Workshop/5.5-Operations/21-alb-2healthy.png)
*Figure 21 (Console) — Target Group medimind-tg showing 2/2 Healthy targets balanced across both AZs*
