---
report_key: "workshopChapters"
report_language: "vi"
report_fields: {"shortTitle": "Triển khai MediMind AI", "customTitle": "5.4. Đóng gói Container & Triển khai MediMind AI (ECR, EC2/EBS, ALB, WAF & CloudFront)", "summary": "Đóng gói Docker Image tự động bằng CodeBuild lên Amazon ECR (333.87 MB), triển khai lên Private EC2 gắn EBS 20 GB qua SSM Session Manager kèm Auto-RAG, sau đó công bố HTTPS an toàn qua ALB, AWS WAF và Amazon CloudFront.", "content": "Đóng gói ứng dụng MediMind từ nhánh main qua CodeBuild lên ECR (v1.0.0 — 333.87 MB), khởi chạy container cổng 8000 trên Private EC2 t3.small + EBS 20 GB gp3 (tự nạp 364 tài liệu RAG qua docker_entrypoint.py, /health ok, /ready ready), gắn Target Group 1 Healthy + ALB Active + AWS WAF 4 luật + CloudFront HTTPS live tại https://d2typ9pphzk4as.cloudfront.net.", "notes": "", "steps": [{"num": "Bước 5.4.1", "title": "Tạo Amazon ECR & Build Docker Image tự động qua AWS CodeBuild (333.87 MB)", "summary": "Tạo kho chứa riêng tư `medimind-clinical-ai` (bật `scanOnPush` và mã hóa AES256), sau đó dùng **AWS CodeBuild (`medimind-docker-build`)** kéo trực tiếp nhánh `main` từ GitHub, build `Dockerfile` (`python:3.11-slim`, non-root `appuser` UID 1000) và đẩy tag `v1.0.0` / `latest` lên ECR.", "code": "aws ecr create-repository --repository-name medimind-clinical-ai \\\n  --image-scanning-configuration scanOnPush=true --encryption-configuration encryptionType=AES256\n# Khởi chạy dự án CodeBuild medimind-docker-build (kéo nhánh main -> docker build -> docker push ECR)\naws codebuild start-build --project-name medimind-docker-build --region ap-southeast-1", "expected": "Build status: SUCCEEDED\nECR Image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0 (333.87 MB < 500 MB Free Tier)", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/09a-cli-ecr.png", "caption": "Ảnh 09a (CLI) — Khởi tạo kho chứa Amazon ECR medimind-clinical-ai"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/09b-cli-codebuild.png", "caption": "Ảnh 09b (CLI) — AWS CodeBuild tự động build Docker Image & push lên ECR thành công"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/09-ecr-scan.png", "caption": "Ảnh 09 (Console) — Amazon ECR xác nhận image v1.0.0 / latest dung lượng tối ưu 333.87 MB"}]}, {"num": "Bước 5.4.2", "title": "Khởi tạo EC2 Private (t3.small) + Ổ dữ liệu EBS 20 GB gp3 mã hóa KMS", "summary": "Khởi tạo máy chủ `medimind-app-server` (Ubuntu 22.04, `t3.small`, đặt trong `Private App Subnet A`, tắt Public IP, gắn `sg-medimind-ec2` và IAM Instance Profile `medimind-ec2-role`) kèm ổ dữ liệu chuyên biệt `20 GB gp3` mã hóa KMS tại `/dev/xvdf` cho ChromaDB & uploads.", "code": "EC2_ID=$(aws ec2 run-instances --image-id \"$AMI_ID\" --instance-type t3.small \\\n  --subnet-id \"$PRIV_APP_A\" --security-group-ids \"$SG_EC2\" \\\n  --iam-instance-profile Name=medimind-ec2-role --no-associate-public-ip-address \\\n  --block-device-mappings '[{\"DeviceName\":\"/dev/sda1\",\"Ebs\":{\"VolumeSize\":10,\"VolumeType\":\"gp3\",\"Encrypted\":true}},{\"DeviceName\":\"/dev/xvdf\",\"Ebs\":{\"VolumeSize\":20,\"VolumeType\":\"gp3\",\"Encrypted\":true}}]' \\\n  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=medimind-app-server}]' --query 'Instances[0].InstanceId' --output text)\naws ec2 wait instance-status-ok --instance-ids \"$EC2_ID\"", "expected": "Instance ID: i-09e81d49793fb5300 | Private IP: 10.0.11.95 | Public IP: None | Status: 3/3 checks passed", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/10a-cli-ec2.png", "caption": "Ảnh 10a (CLI) — Khởi tạo EC2 Private + ổ EBS 20 GB gp3 trên CloudShell"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/10-ec2-ebs.png", "caption": "Ảnh 10 (Console) — EC2 medimind-app-server Running (3/3 checks passed) trong Private Subnet, không Public IP"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/10b-ssm-connect.png", "caption": "Ảnh 10b (Console) — Kết nối trực tiếp vào EC2 Private qua SSM Session Manager không cần khóa SSH"}]}, {"num": "Bước 5.4.3", "title": "Gắn EBS, nạp Secrets & khởi chạy Container Auto-RAG trên SSM Session Manager", "summary": "Vào EC2 qua **SSM Session Manager**, cài Docker/jq/awscli, định dạng và mount ổ EBS 20 GB vào `/mnt/medimind-data` (gán quyền `1000:1000` cho non-root `appuser`), kéo 3 nhóm Secrets về `/etc/medimind.env` (`chmod 600`), rồi khởi chạy container cổng `8000`. Khi khởi động lần đầu, `scripts/docker_entrypoint.py` tự động nạp 364 tài liệu y khoa vào ChromaDB.", "code": "bash\n# 1. Cài Docker, định dạng & mount ổ EBS 20 GB vào /mnt/medimind-data\nsudo mkfs -t ext4 /dev/nvme1n1 && sudo mkdir -p /mnt/medimind-data\nsudo mount /dev/nvme1n1 /mnt/medimind-data\nsudo mkdir -p /mnt/medimind-data/medical_vectordb /mnt/medimind-data/uploads /mnt/medimind-data/logs\nsudo chown -R 1000:1000 /mnt/medimind-data\n# 2. Nạp 3 nhóm Secrets vào /etc/medimind.env (chmod 600) & kéo image ECR\naws ecr get-login-password --region ap-southeast-1 | sudo docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com\nsudo docker run -d --name medimind-clinical-api --restart unless-stopped -p 8000:8000 \\\n  --env-file /etc/medimind.env \\\n  -v /mnt/medimind-data/medical_vectordb:/app/medical_vectordb \\\n  -v /mnt/medimind-data/uploads:/app/uploads \\\n  -v /mnt/medimind-data/logs:/app/logs \\\n  <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0\ncurl -s http://127.0.0.1:8000/health && echo '' && curl -s http://127.0.0.1:8000/ready", "expected": "{\"status\":\"ok\"}\n{\"status\":\"ready\",\"rag\":\"ready\",\"gemini\":\"ready\",\"circuit_breaker\":\"CLOSED\"}", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/11a-ssm-mount-ebs.png", "caption": "Ảnh 11a (SSM) — Cài Docker, định dạng ext4 & gắn ổ đĩa EBS 20 GB vào /mnt/medimind-data"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/11b-ssm-secrets-env.png", "caption": "Ảnh 11b (SSM) — Nạp tự động 3 nhóm AWS Secrets Manager về /etc/medimind.env (chmod 600)"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/11-ssm-health.png", "caption": "Ảnh 11 (SSM) — Xác nhận EBS mounted, Docker Up, ChromaDB đã nạp active_collection.json, /health ok & /ready ready"}]}, {"num": "Bước 5.4.4", "title": "Khởi tạo Target Group (1 Healthy) & Application Load Balancer (medimind-alb)", "summary": "Tạo Target Group `medimind-tg` (HTTP `:8000`, kiểm tra sức khỏe `/health` mỗi 15 giây), đăng ký EC2 `i-09e81d49793fb5300` và tạo Internet-facing ALB `medimind-alb` đặt trên 2 Public Subnets.", "code": "TG_ARN=$(aws elbv2 create-target-group --name medimind-tg --protocol HTTP --port 8000 \\\n  --vpc-id \"$VPC_ID\" --target-type instance --health-check-path /health --query 'TargetGroups[0].TargetGroupArn' --output text)\naws elbv2 register-targets --target-group-arn \"$TG_ARN\" --targets Id=$EC2_ID,Port=8000\nALB_ARN=$(aws elbv2 create-load-balancer --name medimind-alb --subnets \"$PUB_A\" \"$PUB_B\" \\\n  --security-groups \"$SG_ALB\" --scheme internet-facing --type application --query 'LoadBalancers[0].LoadBalancerArn' --output text)\naws elbv2 create-listener --load-balancer-arn \"$ALB_ARN\" --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=\"$TG_ARN\"", "expected": "ALB medimind-alb: Active | Target Group medimind-tg: 1 Healthy", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/12a-cli-alb.png", "caption": "Ảnh 12a (CLI) — Khởi tạo Target Group, ALB & đợi Target đạt trạng thái Healthy"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/12-alb-tg-healthy.png", "caption": "Ảnh 12 (Console) — Target Group medimind-tg (:8000) xác nhận EC2 đạt 1 Healthy"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/12c-alb-active.png", "caption": "Ảnh 12c (Console) — Application Load Balancer medimind-alb trạng thái Active trên 2 AZ"}]}, {"num": "Bước 5.4.5", "title": "Thiết lập Tường lửa biên AWS WAF (medimind-edge-waf — 4 luật bảo vệ)", "summary": "Khởi tạo Web ACL phạm vi `CLOUDFRONT` tại vùng bắt buộc **`us-east-1`** gồm 4 lớp phòng thủ Layer-7:", "bullets": ["**Rule 1 — `AWSManagedRulesCommonRuleSet`**: Chặn OWASP Top 10 & XSS; **loại trừ `SizeRestrictions_BODY`** để ảnh chụp đơn thuốc OCR (> 8 KB) tải lên mượt mà không bị lỗi 403.", "**Rule 2 — `AWSManagedRulesSQLiRuleSet`**: Chặn tấn công SQL Injection.", "**Rule 3 — `AWSManagedRulesKnownBadInputsRuleSet`**: Chặn các payload khai thác lỗ hổng đã biết.", "**Rule 4 — `MediMindRateLimit`**: Giới hạn tối đa **1.000 request / 5 phút / IP** chống DDoS / spam API AI."], "code": "WAF_ARN=$(aws wafv2 create-web-acl --name medimind-edge-waf --scope CLOUDFRONT --region us-east-1 \\\n  --default-action Allow={} --rules file:///tmp/waf-rules.json \\\n  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=MediMindEdgeWAF \\\n  --query 'Summary.ARN' --output text)", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/13a-cli-waf.png", "caption": "Ảnh 13a (CLI) — Khởi tạo AWS WAF medimind-edge-waf (Global / us-east-1) với 4 luật"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/13-waf-rules.png", "caption": "Ảnh 13 (Console) — AWS WAF medimind-edge-waf hiển thị đủ 4 luật bảo vệ lớp biên"}]}, {"num": "Bước 5.4.6", "title": "Khởi tạo Amazon CloudFront HTTPS & Đồng bộ Google OAuth / CORS", "summary": "Tạo CloudFront Distribution **`E3FNAGSNUILKRW`** trỏ Origin về `medimind-alb`, gắn `medimind-edge-waf`, tách 2 Cache Behavior: `/static/*` (`CachingOptimized`) và `/api/*` (`CachingDisabled` + `AllViewer` + `OriginReadTimeout: 60s`). Sau khi có tên miền HTTPS live, cập nhật `CORS_ALLOWED_ORIGINS` & `GOOGLE_REDIRECT_URI` trong `/etc/medimind.env` và Google Cloud Console.", "code": "aws cloudfront create-distribution --distribution-config file:///tmp/cf-config.json\naws cloudfront wait distribution-deployed --id \"$CF_ID\"\n# Trên EC2 SSM: cập nhật /etc/medimind.env với domain CloudFront và tạo lại container để nạp biến môi trường mới\nsudo docker rm -f medimind-clinical-api && sudo docker run -d --name medimind-clinical-api --restart unless-stopped -p 8000:8000 --env-file /etc/medimind.env -v /mnt/medimind-data/medical_vectordb:/app/medical_vectordb -v /mnt/medimind-data/uploads:/app/uploads -v /mnt/medimind-data/logs:/app/logs <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0", "expected": "CloudFront ID: E3FNAGSNUILKRW | Status: Deployed\nLive HTTPS URL: https://d2typ9pphzk4as.cloudfront.net", "images": [{"url": "static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/14a-cli-cloudfront.png", "caption": "Ảnh 14a (CLI) — Khởi tạo CloudFront Distribution & nhận link HTTPS Production live"}, {"url": "static/images/5-Workshop/5.4-Deploy-MediMind/14-cloudfront-route53.png", "caption": "Ảnh 14 (Console) — CloudFront E3FNAGSNUILKRW (Enabled) phục vụ HTTPS tại https://d2typ9pphzk4as.cloudfront.net"}]}, {"title": "Khởi tạo Amazon EFS mã hóa (medimind-shared-efs) & 2 Mount Targets qua 2 AZ", "summary": "Tạo Security Group `medimind-efs-sg` chỉ mở cổng NFS `TCP 2049` từ nguồn `medimind-ec2-sg`, khởi tạo hệ thống tệp dùng chung `medimind-shared-efs` (mã hóa KMS) và gắn đủ 2 Mount Targets trên `ap-southeast-1a` (`10.0.11.254`) & `ap-southeast-1b` (`10.0.12.244`) để sẵn sàng chia sẻ ChromaDB & `/uploads` khi mở rộng đa nút EC2.", "code": "export AWS_REGION=ap-southeast-1\nVPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters \"Name=cidr,Values=10.0.0.0/16\" --query 'Vpcs[0].VpcId' --output text)\nSG_EC2=$(aws ec2 describe-security-groups --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=group-name,Values=medimind-ec2-sg\" --query 'SecurityGroups[0].GroupId' --output text)\n\n# 1. Tạo Security Group NFS 2049 & Amazon EFS mã hóa KMS\nSG_EFS=$(aws ec2 create-security-group --region $AWS_REGION --group-name medimind-efs-sg --description \"Allow NFS 2049 only from medimind-ec2-sg\" --vpc-id \"$VPC_ID\" --query 'GroupId' --output text)\naws ec2 authorize-security-group-ingress --region $AWS_REGION --group-id \"$SG_EFS\" --protocol tcp --port 2049 --source-group \"$SG_EC2\"\nEFS_ID=$(aws efs create-file-system --region $AWS_REGION --performance-mode generalPurpose --throughput-mode bursting --encrypted --tags Key=Name,Value=medimind-shared-efs --query 'FileSystemId' --output text)\n\n# 2. Gắn 2 Mount Targets trên 2 Private App Subnets (10.0.11.0/24 & 10.0.12.0/24)\nfor SUBNET in $(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24\" --query 'Subnets[*].SubnetId' --output text); do\n  aws efs create-mount-target --region $AWS_REGION --file-system-id \"$EFS_ID\" --subnet-id \"$SUBNET\" --security-groups \"$SG_EFS\" --output text\ndone\naws efs describe-mount-targets --region $AWS_REGION --file-system-id \"$EFS_ID\" --query 'MountTargets[*].{MountTargetId:MountTargetId,AZ:AvailabilityZoneName,SubnetId:SubnetId,IP:IpAddress,State:LifeCycleState}' --output table", "expected": "FileSystemId: fs-0d9a2cb6275ca9926 (Encrypted: True, available)\nMountTarget AZ-1a: fsmt-0aee953335ac475d4 (10.0.11.254, Available)\nMountTarget AZ-1b: fsmt-0590fef6da5225039 (10.0.12.244, Available)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/18a-cli-efs-create.png", "caption": "Ảnh 18a (CLI) — Khởi tạo medimind-efs-sg (TCP 2049) & Amazon EFS mã hóa KMS"}, {"url": "static/images/5-Workshop/5.5-Operations/cli-steps/18b-cli-efs-mounts.png", "caption": "Ảnh 18b (CLI) — Tạo 2 EFS Mount Targets trên ap-southeast-1a & ap-southeast-1b"}, {"url": "static/images/5-Workshop/5.5-Operations/18-efs-network.png", "caption": "Ảnh 18 (Console) — Tab Network của medimind-shared-efs xác nhận đủ 2 Mount Targets trạng thái Available"}, {"url": "static/images/5-Workshop/5.5-Operations/18c-efs-overview.png", "caption": "Ảnh 18c (Console) — Tổng quan Amazon EFS medimind-shared-efs (Encrypted, Regional)"}], "num": "Bước 5.4.7"}, {"title": "Đóng gói Golden AMI, Launch Template & Auto Scaling Group (medimind-asg — 2 AZ)", "summary": "Đóng gói Golden AMI (`medimind-clinical-ami-v1`) bằng cờ `--no-reboot` không gián đoạn web live, tạo Launch Template `medimind-app-lt` (bắt buộc IMDSv2 `HttpTokens=required`), sau đó tạo **Auto Scaling Group (`medimind-asg`)** trải đều 2 AZ (`ap-southeast-1a` & `1b`), gắn trực tiếp máy chủ EC2 đang chạy vào ASG (`Min=1, Desired=1, Max=2`, ELB Health Check) và bật luật **Target Tracking tự scale khi CPU > 70%**.", "code": "export AWS_REGION=ap-southeast-1\nEC2_ID=$(aws ec2 describe-instances --region $AWS_REGION --filters \"Name=tag:Name,Values=medimind-app-server\" \"Name=instance-state-name,Values=running\" --query 'Reservations[0].Instances[0].InstanceId' --output text)\nSG_EC2=$(aws ec2 describe-instances --region $AWS_REGION --instance-ids \"$EC2_ID\" --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' --output text)\nPROFILE_ARN=$(aws ec2 describe-instances --region $AWS_REGION --instance-ids \"$EC2_ID\" --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn' --output text)\n\n# 1. Tạo Golden AMI (--no-reboot tránh downtime) & Launch Template (bắt buộc IMDSv2)\nAMI_ID=$(aws ec2 create-image --region $AWS_REGION --instance-id \"$EC2_ID\" --name \"medimind-clinical-ami-v1\" --description \"MediMind Clinical AI Golden Image\" --no-reboot --query 'ImageId' --output text)\naws ec2 create-launch-template --region $AWS_REGION --launch-template-name medimind-app-lt --version-description \"MediMind t3.small Private App Launch Template\" --launch-template-data \"{\\\"ImageId\\\":\\\"$AMI_ID\\\",\\\"InstanceType\\\":\\\"t3.small\\\",\\\"SecurityGroupIds\\\":[\\\"$SG_EC2\\\"],\\\"IamInstanceProfile\\\":{\\\"Arn\\\":\\\"$PROFILE_ARN\\\"},\\\"MetadataOptions\\\":{\\\"HttpTokens\\\":\\\"required\\\",\\\"HttpEndpoint\\\":\\\"enabled\\\"}}\"\n\n# 2. BẮT BUỘC đợi AMI hoàn tất snapshot (pending -> available) trước khi tạo ASG\naws ec2 wait image-available --image-ids \"$AMI_ID\" --region $AWS_REGION\n\n# 3. Tạo Auto Scaling Group 2 AZ, gắn EC2 live ($0 phí thêm) & luật Target Tracking CPU 70%\nVPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters \"Name=cidr,Values=10.0.0.0/16\" --query 'Vpcs[0].VpcId' --output text)\nPRIV_APP_CSV=$(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24\" --query 'Subnets[*].SubnetId' --output text | tr '\\t' ',')\nTG_ARN=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text)\n\naws autoscaling create-auto-scaling-group --region $AWS_REGION --auto-scaling-group-name medimind-asg --launch-template 'LaunchTemplateName=medimind-app-lt,Version=$Latest' --min-size 0 --max-size 2 --desired-capacity 0 --vpc-zone-identifier \"$PRIV_APP_CSV\" --target-group-arns \"$TG_ARN\" --health-check-type ELB --health-check-grace-period 300\naws autoscaling attach-instances --region $AWS_REGION --instance-ids \"$EC2_ID\" --auto-scaling-group-name medimind-asg\naws autoscaling update-auto-scaling-group --region $AWS_REGION --auto-scaling-group-name medimind-asg --min-size 1 --max-size 2 --desired-capacity 1\naws autoscaling put-scaling-policy --region $AWS_REGION --auto-scaling-group-name medimind-asg --policy-name medimind-cpu-target-70 --policy-type TargetTrackingScaling --target-tracking-configuration '{\"PredefinedMetricSpecification\":{\"PredefinedMetricType\":\"ASGAverageCPUUtilization\"},\"TargetValue\":70.0}' --output text\n\naws autoscaling describe-auto-scaling-groups --region $AWS_REGION --auto-scaling-group-names medimind-asg --query 'AutoScalingGroups[0].{Name:AutoScalingGroupName,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,HealthCheck:HealthCheckType,AZs:AvailabilityZones}' --output table", "expected": "LaunchTemplate: lt-02ed0bffddb2a5492 (medimind-app-lt v1, ami-0858b3aafe17cd29e)\nAutoScalingGroup: medimind-asg | Min: 1 | Desired: 1 | Max: 2 | HealthCheck: ELB | AZs: ap-southeast-1a, ap-southeast-1b", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/19a-cli-launch-template.png", "caption": "Ảnh 19a (CLI) — Đóng gói Golden AMI (--no-reboot) & tạo Launch Template medimind-app-lt"}, {"url": "static/images/5-Workshop/5.5-Operations/cli-steps/19b-cli-asg.png", "caption": "Ảnh 19b (CLI) — Khởi tạo medimind-asg 2 AZ, gắn EC2 live & bật luật Target Tracking CPU 70%"}, {"url": "static/images/5-Workshop/5.5-Operations/19-asg-console.png", "caption": "Ảnh 19 (Console) — Auto Scaling Group medimind-asg trạng thái At desired capacity (1–2 nodes, Launch Template lt-02ed0bffddb2a5492)"}], "num": "Bước 5.4.8"}, {"num": "Bước 5.4.9", "title": "Mở rộng Auto Scaling Group lên 2 EC2 Multi-AZ & Xác nhận 2/2 Healthy trên ALB", "summary": "Nâng `DesiredCapacity` của `medimind-asg` từ `1` → **`2`** để AWS Auto Scaling tự động khởi chạy máy chủ EC2 `t3.small` thứ hai tại **`ap-southeast-1b`** (`10.0.12.18`) từ Golden AMI `medimind-clinical-ami-v1`. Container `medimind-clinical-api` tự động khởi chạy và vượt qua kiểm tra sức khỏe `/health`, nâng tổng số đích hoạt động trên `medimind-tg` lên **`2/2 Healthy`** trải đều 2 AZ.", "code": "export AWS_REGION=ap-southeast-1\naws autoscaling set-desired-capacity --region $AWS_REGION --auto-scaling-group-name medimind-asg --desired-capacity 2\n\n# Chờ EC2 số 2 tại ap-southeast-1b khởi động từ Golden AMI & vượt qua ALB /health (~90s)\nsleep 90\n\naws ec2 describe-instances --region $AWS_REGION --filters \"Name=instance-state-name,Values=running,pending\" --query 'Reservations[*].Instances[*].{Id:InstanceId,AZ:Placement.AvailabilityZone,Type:InstanceType,PrivateIP:PrivateIpAddress,State:State.Name}' --output table\n\nTG_ARN=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text)\naws elbv2 describe-target-health --region $AWS_REGION --target-group-arn \"$TG_ARN\" --query 'TargetHealthDescriptions[*].{InstanceId:Target.Id,Port:Target.Port,State:TargetHealth.State}' --output table", "expected": "EC2 #1: i-09e81d49793fb5300 | ap-southeast-1a | 10.0.11.95 | running | healthy (:8000)\nEC2 #2: i-076aaacd95d5affaf | ap-southeast-1b | 10.0.12.18 | running | healthy (:8000)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/21a-cli-2ec2-healthy.png", "caption": "Ảnh 21a (CLI) — Scale medimind-asg lên 2 EC2 (AZ-1a & AZ-1b) & xác nhận 2/2 targets healthy trên CloudShell"}, {"url": "static/images/5-Workshop/5.5-Operations/21-alb-2healthy.png", "caption": "Ảnh 21 (Console) — Target Group medimind-tg hiển thị 2/2 Healthy targets cân bằng tải qua ap-southeast-1a & 1b"}]}], "url": "https://d2typ9pphzk4as.cloudfront.net"}
report_index: 3
report_shared: {"code": "5.4", "url": "https://d2typ9pphzk4as.cloudfront.net"}
title: "Triển khai MediMind AI"
date: "2026-11-27"
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

# 5.4. Đóng gói Container & Triển khai MediMind AI (ECR, EC2/EBS, ALB, WAF & CloudFront)

Đóng gói Docker Image tự động bằng CodeBuild lên Amazon ECR (333.87 MB), triển khai lên Private EC2 gắn EBS 20 GB qua SSM Session Manager kèm Auto-RAG, sau đó công bố HTTPS an toàn qua ALB, AWS WAF và Amazon CloudFront.

## Bước 5.4.1 — Tạo Amazon ECR & Build Docker Image tự động qua AWS CodeBuild (333.87 MB)

Tạo kho chứa riêng tư `medimind-clinical-ai` (bật `scanOnPush` và mã hóa AES256), sau đó dùng **AWS CodeBuild (`medimind-docker-build`)** kéo trực tiếp nhánh `main` từ GitHub, build `Dockerfile` (`python:3.11-slim`, non-root `appuser` UID 1000) và đẩy tag `v1.0.0` / `latest` lên ECR.

```bash
aws ecr create-repository --repository-name medimind-clinical-ai \
  --image-scanning-configuration scanOnPush=true --encryption-configuration encryptionType=AES256
# Khởi chạy dự án CodeBuild medimind-docker-build (kéo nhánh main -> docker build -> docker push ECR)
aws codebuild start-build --project-name medimind-docker-build --region ap-southeast-1
```

**Kết quả kỳ vọng:**
```text
Build status: SUCCEEDED
ECR Image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0 (333.87 MB < 500 MB Free Tier)
```

![Ảnh 09a (CLI) — Khởi tạo kho chứa Amazon ECR medimind-clinical-ai](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/09a-cli-ecr.png)
*Ảnh 09a (CLI) — Khởi tạo kho chứa Amazon ECR medimind-clinical-ai*

![Ảnh 09b (CLI) — AWS CodeBuild tự động build Docker Image & push lên ECR thành công](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/09b-cli-codebuild.png)
*Ảnh 09b (CLI) — AWS CodeBuild tự động build Docker Image & push lên ECR thành công*

![Ảnh 09 (Console) — Amazon ECR xác nhận image v1.0.0 / latest dung lượng tối ưu 333.87 MB](../../../static/images/5-Workshop/5.4-Deploy-MediMind/09-ecr-scan.png)
*Ảnh 09 (Console) — Amazon ECR xác nhận image v1.0.0 / latest dung lượng tối ưu 333.87 MB*

## Bước 5.4.2 — Khởi tạo EC2 Private (t3.small) + Ổ dữ liệu EBS 20 GB gp3 mã hóa KMS

Khởi tạo máy chủ `medimind-app-server` (Ubuntu 22.04, `t3.small`, đặt trong `Private App Subnet A`, tắt Public IP, gắn `sg-medimind-ec2` và IAM Instance Profile `medimind-ec2-role`) kèm ổ dữ liệu chuyên biệt `20 GB gp3` mã hóa KMS tại `/dev/xvdf` cho ChromaDB & uploads.

```bash
EC2_ID=$(aws ec2 run-instances --image-id "$AMI_ID" --instance-type t3.small \
  --subnet-id "$PRIV_APP_A" --security-group-ids "$SG_EC2" \
  --iam-instance-profile Name=medimind-ec2-role --no-associate-public-ip-address \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":10,"VolumeType":"gp3","Encrypted":true}},{"DeviceName":"/dev/xvdf","Ebs":{"VolumeSize":20,"VolumeType":"gp3","Encrypted":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=medimind-app-server}]' --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-status-ok --instance-ids "$EC2_ID"
```

**Kết quả kỳ vọng:**
```text
Instance ID: i-09e81d49793fb5300 | Private IP: 10.0.11.95 | Public IP: None | Status: 3/3 checks passed
```

![Ảnh 10a (CLI) — Khởi tạo EC2 Private + ổ EBS 20 GB gp3 trên CloudShell](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/10a-cli-ec2.png)
*Ảnh 10a (CLI) — Khởi tạo EC2 Private + ổ EBS 20 GB gp3 trên CloudShell*

![Ảnh 10 (Console) — EC2 medimind-app-server Running (3/3 checks passed) trong Private Subnet, không Public IP](../../../static/images/5-Workshop/5.4-Deploy-MediMind/10-ec2-ebs.png)
*Ảnh 10 (Console) — EC2 medimind-app-server Running (3/3 checks passed) trong Private Subnet, không Public IP*

![Ảnh 10b (Console) — Kết nối trực tiếp vào EC2 Private qua SSM Session Manager không cần khóa SSH](../../../static/images/5-Workshop/5.4-Deploy-MediMind/10b-ssm-connect.png)
*Ảnh 10b (Console) — Kết nối trực tiếp vào EC2 Private qua SSM Session Manager không cần khóa SSH*

## Bước 5.4.3 — Gắn EBS, nạp Secrets & khởi chạy Container Auto-RAG trên SSM Session Manager

Vào EC2 qua **SSM Session Manager**, cài Docker/jq/awscli, định dạng và mount ổ EBS 20 GB vào `/mnt/medimind-data` (gán quyền `1000:1000` cho non-root `appuser`), kéo 3 nhóm Secrets về `/etc/medimind.env` (`chmod 600`), rồi khởi chạy container cổng `8000`. Khi khởi động lần đầu, `scripts/docker_entrypoint.py` tự động nạp 364 tài liệu y khoa vào ChromaDB.

```bash
bash
# 1. Cài Docker, định dạng & mount ổ EBS 20 GB vào /mnt/medimind-data
sudo mkfs -t ext4 /dev/nvme1n1 && sudo mkdir -p /mnt/medimind-data
sudo mount /dev/nvme1n1 /mnt/medimind-data
sudo mkdir -p /mnt/medimind-data/medical_vectordb /mnt/medimind-data/uploads /mnt/medimind-data/logs
sudo chown -R 1000:1000 /mnt/medimind-data
# 2. Nạp 3 nhóm Secrets vào /etc/medimind.env (chmod 600) & kéo image ECR
aws ecr get-login-password --region ap-southeast-1 | sudo docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com
sudo docker run -d --name medimind-clinical-api --restart unless-stopped -p 8000:8000 \
  --env-file /etc/medimind.env \
  -v /mnt/medimind-data/medical_vectordb:/app/medical_vectordb \
  -v /mnt/medimind-data/uploads:/app/uploads \
  -v /mnt/medimind-data/logs:/app/logs \
  <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0
curl -s http://127.0.0.1:8000/health && echo '' && curl -s http://127.0.0.1:8000/ready
```

**Kết quả kỳ vọng:**
```text
{"status":"ok"}
{"status":"ready","rag":"ready","gemini":"ready","circuit_breaker":"CLOSED"}
```

![Ảnh 11a (SSM) — Cài Docker, định dạng ext4 & gắn ổ đĩa EBS 20 GB vào /mnt/medimind-data](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/11a-ssm-mount-ebs.png)
*Ảnh 11a (SSM) — Cài Docker, định dạng ext4 & gắn ổ đĩa EBS 20 GB vào /mnt/medimind-data*

![Ảnh 11b (SSM) — Nạp tự động 3 nhóm AWS Secrets Manager về /etc/medimind.env (chmod 600)](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/11b-ssm-secrets-env.png)
*Ảnh 11b (SSM) — Nạp tự động 3 nhóm AWS Secrets Manager về /etc/medimind.env (chmod 600)*

![Ảnh 11 (SSM) — Xác nhận EBS mounted, Docker Up, ChromaDB đã nạp active_collection.json, /health ok & /ready ready](../../../static/images/5-Workshop/5.4-Deploy-MediMind/11-ssm-health.png)
*Ảnh 11 (SSM) — Xác nhận EBS mounted, Docker Up, ChromaDB đã nạp active_collection.json, /health ok & /ready ready*

## Bước 5.4.4 — Khởi tạo Target Group (1 Healthy) & Application Load Balancer (medimind-alb)

Tạo Target Group `medimind-tg` (HTTP `:8000`, kiểm tra sức khỏe `/health` mỗi 15 giây), đăng ký EC2 `i-09e81d49793fb5300` và tạo Internet-facing ALB `medimind-alb` đặt trên 2 Public Subnets.

```bash
TG_ARN=$(aws elbv2 create-target-group --name medimind-tg --protocol HTTP --port 8000 \
  --vpc-id "$VPC_ID" --target-type instance --health-check-path /health --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 register-targets --target-group-arn "$TG_ARN" --targets Id=$EC2_ID,Port=8000
ALB_ARN=$(aws elbv2 create-load-balancer --name medimind-alb --subnets "$PUB_A" "$PUB_B" \
  --security-groups "$SG_ALB" --scheme internet-facing --type application --query 'LoadBalancers[0].LoadBalancerArn' --output text)
aws elbv2 create-listener --load-balancer-arn "$ALB_ARN" --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn="$TG_ARN"
```

**Kết quả kỳ vọng:**
```text
ALB medimind-alb: Active | Target Group medimind-tg: 1 Healthy
```

![Ảnh 12a (CLI) — Khởi tạo Target Group, ALB & đợi Target đạt trạng thái Healthy](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/12a-cli-alb.png)
*Ảnh 12a (CLI) — Khởi tạo Target Group, ALB & đợi Target đạt trạng thái Healthy*

![Ảnh 12 (Console) — Target Group medimind-tg (:8000) xác nhận EC2 đạt 1 Healthy](../../../static/images/5-Workshop/5.4-Deploy-MediMind/12-alb-tg-healthy.png)
*Ảnh 12 (Console) — Target Group medimind-tg (:8000) xác nhận EC2 đạt 1 Healthy*

![Ảnh 12c (Console) — Application Load Balancer medimind-alb trạng thái Active trên 2 AZ](../../../static/images/5-Workshop/5.4-Deploy-MediMind/12c-alb-active.png)
*Ảnh 12c (Console) — Application Load Balancer medimind-alb trạng thái Active trên 2 AZ*

## Bước 5.4.5 — Thiết lập Tường lửa biên AWS WAF (medimind-edge-waf — 4 luật bảo vệ)

Khởi tạo Web ACL phạm vi `CLOUDFRONT` tại vùng bắt buộc **`us-east-1`** gồm 4 lớp phòng thủ Layer-7:

- **Rule 1 — `AWSManagedRulesCommonRuleSet`**: Chặn OWASP Top 10 & XSS; **loại trừ `SizeRestrictions_BODY`** để ảnh chụp đơn thuốc OCR (> 8 KB) tải lên mượt mà không bị lỗi 403.
- **Rule 2 — `AWSManagedRulesSQLiRuleSet`**: Chặn tấn công SQL Injection.
- **Rule 3 — `AWSManagedRulesKnownBadInputsRuleSet`**: Chặn các payload khai thác lỗ hổng đã biết.
- **Rule 4 — `MediMindRateLimit`**: Giới hạn tối đa **1.000 request / 5 phút / IP** chống DDoS / spam API AI.

```bash
WAF_ARN=$(aws wafv2 create-web-acl --name medimind-edge-waf --scope CLOUDFRONT --region us-east-1 \
  --default-action Allow={} --rules file:///tmp/waf-rules.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=MediMindEdgeWAF \
  --query 'Summary.ARN' --output text)
```

![Ảnh 13a (CLI) — Khởi tạo AWS WAF medimind-edge-waf (Global / us-east-1) với 4 luật](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/13a-cli-waf.png)
*Ảnh 13a (CLI) — Khởi tạo AWS WAF medimind-edge-waf (Global / us-east-1) với 4 luật*

![Ảnh 13 (Console) — AWS WAF medimind-edge-waf hiển thị đủ 4 luật bảo vệ lớp biên](../../../static/images/5-Workshop/5.4-Deploy-MediMind/13-waf-rules.png)
*Ảnh 13 (Console) — AWS WAF medimind-edge-waf hiển thị đủ 4 luật bảo vệ lớp biên*

## Bước 5.4.6 — Khởi tạo Amazon CloudFront HTTPS & Đồng bộ Google OAuth / CORS

Tạo CloudFront Distribution **`E3FNAGSNUILKRW`** trỏ Origin về `medimind-alb`, gắn `medimind-edge-waf`, tách 2 Cache Behavior: `/static/*` (`CachingOptimized`) và `/api/*` (`CachingDisabled` + `AllViewer` + `OriginReadTimeout: 60s`). Sau khi có tên miền HTTPS live, cập nhật `CORS_ALLOWED_ORIGINS` & `GOOGLE_REDIRECT_URI` trong `/etc/medimind.env` và Google Cloud Console.

```bash
aws cloudfront create-distribution --distribution-config file:///tmp/cf-config.json
aws cloudfront wait distribution-deployed --id "$CF_ID"
# Trên EC2 SSM: cập nhật /etc/medimind.env với domain CloudFront và tạo lại container để nạp biến môi trường mới
sudo docker rm -f medimind-clinical-api && sudo docker run -d --name medimind-clinical-api --restart unless-stopped -p 8000:8000 --env-file /etc/medimind.env -v /mnt/medimind-data/medical_vectordb:/app/medical_vectordb -v /mnt/medimind-data/uploads:/app/uploads -v /mnt/medimind-data/logs:/app/logs <AWS_ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/medimind-clinical-ai:v1.0.0
```

**Kết quả kỳ vọng:**
```text
CloudFront ID: E3FNAGSNUILKRW | Status: Deployed
Live HTTPS URL: https://d2typ9pphzk4as.cloudfront.net
```

![Ảnh 14a (CLI) — Khởi tạo CloudFront Distribution & nhận link HTTPS Production live](../../../static/images/5-Workshop/5.4-Deploy-MediMind/cli-steps/14a-cli-cloudfront.png)
*Ảnh 14a (CLI) — Khởi tạo CloudFront Distribution & nhận link HTTPS Production live*

![Ảnh 14 (Console) — CloudFront E3FNAGSNUILKRW (Enabled) phục vụ HTTPS tại https://d2typ9pphzk4as.cloudfront.net](../../../static/images/5-Workshop/5.4-Deploy-MediMind/14-cloudfront-route53.png)
*Ảnh 14 (Console) — CloudFront E3FNAGSNUILKRW (Enabled) phục vụ HTTPS tại https://d2typ9pphzk4as.cloudfront.net*

## Bước 5.4.7 — Khởi tạo Amazon EFS mã hóa (medimind-shared-efs) & 2 Mount Targets qua 2 AZ

Tạo Security Group `medimind-efs-sg` chỉ mở cổng NFS `TCP 2049` từ nguồn `medimind-ec2-sg`, khởi tạo hệ thống tệp dùng chung `medimind-shared-efs` (mã hóa KMS) và gắn đủ 2 Mount Targets trên `ap-southeast-1a` (`10.0.11.254`) & `ap-southeast-1b` (`10.0.12.244`) để sẵn sàng chia sẻ ChromaDB & `/uploads` khi mở rộng đa nút EC2.

```bash
export AWS_REGION=ap-southeast-1
VPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters "Name=cidr,Values=10.0.0.0/16" --query 'Vpcs[0].VpcId' --output text)
SG_EC2=$(aws ec2 describe-security-groups --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=medimind-ec2-sg" --query 'SecurityGroups[0].GroupId' --output text)

# 1. Tạo Security Group NFS 2049 & Amazon EFS mã hóa KMS
SG_EFS=$(aws ec2 create-security-group --region $AWS_REGION --group-name medimind-efs-sg --description "Allow NFS 2049 only from medimind-ec2-sg" --vpc-id "$VPC_ID" --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --region $AWS_REGION --group-id "$SG_EFS" --protocol tcp --port 2049 --source-group "$SG_EC2"
EFS_ID=$(aws efs create-file-system --region $AWS_REGION --performance-mode generalPurpose --throughput-mode bursting --encrypted --tags Key=Name,Value=medimind-shared-efs --query 'FileSystemId' --output text)

# 2. Gắn 2 Mount Targets trên 2 Private App Subnets (10.0.11.0/24 & 10.0.12.0/24)
for SUBNET in $(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24" --query 'Subnets[*].SubnetId' --output text); do
  aws efs create-mount-target --region $AWS_REGION --file-system-id "$EFS_ID" --subnet-id "$SUBNET" --security-groups "$SG_EFS" --output text
done
aws efs describe-mount-targets --region $AWS_REGION --file-system-id "$EFS_ID" --query 'MountTargets[*].{MountTargetId:MountTargetId,AZ:AvailabilityZoneName,SubnetId:SubnetId,IP:IpAddress,State:LifeCycleState}' --output table
```

**Kết quả kỳ vọng:**
```text
FileSystemId: fs-0d9a2cb6275ca9926 (Encrypted: True, available)
MountTarget AZ-1a: fsmt-0aee953335ac475d4 (10.0.11.254, Available)
MountTarget AZ-1b: fsmt-0590fef6da5225039 (10.0.12.244, Available)
```

![Ảnh 18a (CLI) — Khởi tạo medimind-efs-sg (TCP 2049) & Amazon EFS mã hóa KMS](../../../static/images/5-Workshop/5.5-Operations/cli-steps/18a-cli-efs-create.png)
*Ảnh 18a (CLI) — Khởi tạo medimind-efs-sg (TCP 2049) & Amazon EFS mã hóa KMS*

![Ảnh 18b (CLI) — Tạo 2 EFS Mount Targets trên ap-southeast-1a & ap-southeast-1b](../../../static/images/5-Workshop/5.5-Operations/cli-steps/18b-cli-efs-mounts.png)
*Ảnh 18b (CLI) — Tạo 2 EFS Mount Targets trên ap-southeast-1a & ap-southeast-1b*

![Ảnh 18 (Console) — Tab Network của medimind-shared-efs xác nhận đủ 2 Mount Targets trạng thái Available](../../../static/images/5-Workshop/5.5-Operations/18-efs-network.png)
*Ảnh 18 (Console) — Tab Network của medimind-shared-efs xác nhận đủ 2 Mount Targets trạng thái Available*

![Ảnh 18c (Console) — Tổng quan Amazon EFS medimind-shared-efs (Encrypted, Regional)](../../../static/images/5-Workshop/5.5-Operations/18c-efs-overview.png)
*Ảnh 18c (Console) — Tổng quan Amazon EFS medimind-shared-efs (Encrypted, Regional)*

## Bước 5.4.8 — Đóng gói Golden AMI, Launch Template & Auto Scaling Group (medimind-asg — 2 AZ)

Đóng gói Golden AMI (`medimind-clinical-ami-v1`) bằng cờ `--no-reboot` không gián đoạn web live, tạo Launch Template `medimind-app-lt` (bắt buộc IMDSv2 `HttpTokens=required`), sau đó tạo **Auto Scaling Group (`medimind-asg`)** trải đều 2 AZ (`ap-southeast-1a` & `1b`), gắn trực tiếp máy chủ EC2 đang chạy vào ASG (`Min=1, Desired=1, Max=2`, ELB Health Check) và bật luật **Target Tracking tự scale khi CPU > 70%**.

```bash
export AWS_REGION=ap-southeast-1
EC2_ID=$(aws ec2 describe-instances --region $AWS_REGION --filters "Name=tag:Name,Values=medimind-app-server" "Name=instance-state-name,Values=running" --query 'Reservations[0].Instances[0].InstanceId' --output text)
SG_EC2=$(aws ec2 describe-instances --region $AWS_REGION --instance-ids "$EC2_ID" --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' --output text)
PROFILE_ARN=$(aws ec2 describe-instances --region $AWS_REGION --instance-ids "$EC2_ID" --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn' --output text)

# 1. Tạo Golden AMI (--no-reboot tránh downtime) & Launch Template (bắt buộc IMDSv2)
AMI_ID=$(aws ec2 create-image --region $AWS_REGION --instance-id "$EC2_ID" --name "medimind-clinical-ami-v1" --description "MediMind Clinical AI Golden Image" --no-reboot --query 'ImageId' --output text)
aws ec2 create-launch-template --region $AWS_REGION --launch-template-name medimind-app-lt --version-description "MediMind t3.small Private App Launch Template" --launch-template-data "{\"ImageId\":\"$AMI_ID\",\"InstanceType\":\"t3.small\",\"SecurityGroupIds\":[\"$SG_EC2\"],\"IamInstanceProfile\":{\"Arn\":\"$PROFILE_ARN\"},\"MetadataOptions\":{\"HttpTokens\":\"required\",\"HttpEndpoint\":\"enabled\"}}"

# 2. BẮT BUỘC đợi AMI hoàn tất snapshot (pending -> available) trước khi tạo ASG
aws ec2 wait image-available --image-ids "$AMI_ID" --region $AWS_REGION

# 3. Tạo Auto Scaling Group 2 AZ, gắn EC2 live ($0 phí thêm) & luật Target Tracking CPU 70%
VPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters "Name=cidr,Values=10.0.0.0/16" --query 'Vpcs[0].VpcId' --output text)
PRIV_APP_CSV=$(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24" --query 'Subnets[*].SubnetId' --output text | tr '\t' ',')
TG_ARN=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text)

aws autoscaling create-auto-scaling-group --region $AWS_REGION --auto-scaling-group-name medimind-asg --launch-template 'LaunchTemplateName=medimind-app-lt,Version=$Latest' --min-size 0 --max-size 2 --desired-capacity 0 --vpc-zone-identifier "$PRIV_APP_CSV" --target-group-arns "$TG_ARN" --health-check-type ELB --health-check-grace-period 300
aws autoscaling attach-instances --region $AWS_REGION --instance-ids "$EC2_ID" --auto-scaling-group-name medimind-asg
aws autoscaling update-auto-scaling-group --region $AWS_REGION --auto-scaling-group-name medimind-asg --min-size 1 --max-size 2 --desired-capacity 1
aws autoscaling put-scaling-policy --region $AWS_REGION --auto-scaling-group-name medimind-asg --policy-name medimind-cpu-target-70 --policy-type TargetTrackingScaling --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":70.0}' --output text

aws autoscaling describe-auto-scaling-groups --region $AWS_REGION --auto-scaling-group-names medimind-asg --query 'AutoScalingGroups[0].{Name:AutoScalingGroupName,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,HealthCheck:HealthCheckType,AZs:AvailabilityZones}' --output table
```

**Kết quả kỳ vọng:**
```text
LaunchTemplate: lt-02ed0bffddb2a5492 (medimind-app-lt v1, ami-0858b3aafe17cd29e)
AutoScalingGroup: medimind-asg | Min: 1 | Desired: 1 | Max: 2 | HealthCheck: ELB | AZs: ap-southeast-1a, ap-southeast-1b
```

![Ảnh 19a (CLI) — Đóng gói Golden AMI (--no-reboot) & tạo Launch Template medimind-app-lt](../../../static/images/5-Workshop/5.5-Operations/cli-steps/19a-cli-launch-template.png)
*Ảnh 19a (CLI) — Đóng gói Golden AMI (--no-reboot) & tạo Launch Template medimind-app-lt*

![Ảnh 19b (CLI) — Khởi tạo medimind-asg 2 AZ, gắn EC2 live & bật luật Target Tracking CPU 70%](../../../static/images/5-Workshop/5.5-Operations/cli-steps/19b-cli-asg.png)
*Ảnh 19b (CLI) — Khởi tạo medimind-asg 2 AZ, gắn EC2 live & bật luật Target Tracking CPU 70%*

![Ảnh 19 (Console) — Auto Scaling Group medimind-asg trạng thái At desired capacity (1–2 nodes, Launch Template lt-02ed0bffddb2a5492)](../../../static/images/5-Workshop/5.5-Operations/19-asg-console.png)
*Ảnh 19 (Console) — Auto Scaling Group medimind-asg trạng thái At desired capacity (1–2 nodes, Launch Template lt-02ed0bffddb2a5492)*

## Bước 5.4.9 — Mở rộng Auto Scaling Group lên 2 EC2 Multi-AZ & Xác nhận 2/2 Healthy trên ALB

Nâng `DesiredCapacity` của `medimind-asg` từ `1` → **`2`** để AWS Auto Scaling tự động khởi chạy máy chủ EC2 `t3.small` thứ hai tại **`ap-southeast-1b`** (`10.0.12.18`) từ Golden AMI `medimind-clinical-ami-v1`. Container `medimind-clinical-api` tự động khởi chạy và vượt qua kiểm tra sức khỏe `/health`, nâng tổng số đích hoạt động trên `medimind-tg` lên **`2/2 Healthy`** trải đều 2 AZ.

```bash
export AWS_REGION=ap-southeast-1
aws autoscaling set-desired-capacity --region $AWS_REGION --auto-scaling-group-name medimind-asg --desired-capacity 2

# Chờ EC2 số 2 tại ap-southeast-1b khởi động từ Golden AMI & vượt qua ALB /health (~90s)
sleep 90

aws ec2 describe-instances --region $AWS_REGION --filters "Name=instance-state-name,Values=running,pending" --query 'Reservations[*].Instances[*].{Id:InstanceId,AZ:Placement.AvailabilityZone,Type:InstanceType,PrivateIP:PrivateIpAddress,State:State.Name}' --output table

TG_ARN=$(aws elbv2 describe-target-groups --names medimind-tg --region $AWS_REGION --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 describe-target-health --region $AWS_REGION --target-group-arn "$TG_ARN" --query 'TargetHealthDescriptions[*].{InstanceId:Target.Id,Port:Target.Port,State:TargetHealth.State}' --output table
```

**Kết quả kỳ vọng:**
```text
EC2 #1: i-09e81d49793fb5300 | ap-southeast-1a | 10.0.11.95 | running | healthy (:8000)
EC2 #2: i-076aaacd95d5affaf | ap-southeast-1b | 10.0.12.18 | running | healthy (:8000)
```

![Ảnh 21a (CLI) — Scale medimind-asg lên 2 EC2 (AZ-1a & AZ-1b) & xác nhận 2/2 targets healthy trên CloudShell](../../../static/images/5-Workshop/5.5-Operations/cli-steps/21a-cli-2ec2-healthy.png)
*Ảnh 21a (CLI) — Scale medimind-asg lên 2 EC2 (AZ-1a & AZ-1b) & xác nhận 2/2 targets healthy trên CloudShell*

![Ảnh 21 (Console) — Target Group medimind-tg hiển thị 2/2 Healthy targets cân bằng tải qua ap-southeast-1a & 1b](../../../static/images/5-Workshop/5.5-Operations/21-alb-2healthy.png)
*Ảnh 21 (Console) — Target Group medimind-tg hiển thị 2/2 Healthy targets cân bằng tải qua ap-southeast-1a & 1b*
