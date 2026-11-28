---
report_key: "workshopChapters"
report_language: "vi"
report_fields: {"shortTitle": "Hạ tầng AWS 3 tầng", "customTitle": "5.3. Khởi tạo Hạ tầng AWS 3 Tầng (VPC, SG, IAM, RDS, S3 & Secrets)", "summary": "Khởi tạo tuần tự 9 lớp nền tảng bảo mật & Multi-AZ bằng AWS CloudShell tại ap-southeast-1: VPC 6 Subnets, IGW, 2 NAT Gateways đối xứng, 3 Security Groups lồng nhau, IAM Role đặc quyền tối thiểu, RDS PostgreSQL 16.16, S3 Backup, 3 nhóm Secrets Manager và S3 Gateway Endpoint.", "content": "Khởi tạo toàn bộ nền tảng mạng 3 tầng cô lập (medimind-vpc 10.0.0.0/16, 6 Subnets qua 2 AZ, IGW, NAT Gateway, 3 Route Tables), 3 Security Groups lồng nhau (ALB -> EC2 :8000 -> RDS :5432), IAM Role medimind-ec2-role, RDS PostgreSQL 16.16, S3 Backup Lifecycle và 3 nhóm AWS Secrets Manager tại ap-southeast-1.", "notes": "", "steps": [{"num": "Bước 5.3.1", "title": "Tạo VPC (10.0.0.0/16) & 6 Subnets trải đều 2 Availability Zones (1a / 1b)", "summary": "Thiết lập mạng riêng ảo `medimind-vpc` (bật DNS Support & DNS Hostnames) và chia làm 3 tầng mạng tách biệt trên 2 AZ (`ap-southeast-1a` & `ap-southeast-1b`):", "table": {"headers": ["Tầng mạng", "Subnet AZ-1a", "Subnet AZ-1b", "Mục đích"], "rows": [["Public Tier", "`10.0.1.0/24` (`medimind-public-a`)", "`10.0.2.0/24` (`medimind-public-b`)", "Đặt Application Load Balancer & NAT Gateway."], ["Private App Tier", "`10.0.11.0/24` (`medimind-private-app-a`)", "`10.0.12.0/24` (`medimind-private-app-b`)", "Đặt EC2 chạy Docker container MediMind (không Public IP)."], ["Private DB Tier", "`10.0.21.0/24` (`medimind-private-db-a`)", "`10.0.22.0/24` (`medimind-private-db-b`)", "Đặt Amazon RDS PostgreSQL (cô lập hoàn toàn khỏi Internet)."]]}, "code": "export AWS_REGION=ap-southeast-1\nVPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region $AWS_REGION \\\n  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=medimind-vpc}]' --query 'Vpc.VpcId' --output text)\naws ec2 modify-vpc-attribute --vpc-id \"$VPC_ID\" --enable-dns-support '{\"Value\":true}'\naws ec2 modify-vpc-attribute --vpc-id \"$VPC_ID\" --enable-dns-hostnames '{\"Value\":true}'\n# Tạo đủ 6 Subnets: public-a/b, private-app-a/b, private-db-a/b và lưu vào ~/medimind_ids.env", "expected": "VPC_ID=vpc-0173e16d7c27c35f8 (10.0.0.0/16) + 6 Subnets created across ap-southeast-1a & 1b", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03a-cli-vpc-subnets.png", "caption": "Ảnh 03a (CLI) — Khởi tạo medimind-vpc và 6 Subnets qua 2 AZ trên AWS CloudShell"}]}, {"num": "Bước 5.3.2", "title": "Gắn Internet Gateway, NAT Gateway & 3 Route Tables cô lập", "summary": "Định tuyến chuẩn 3 tầng: `rt-public` đi trực tiếp qua Internet Gateway (`medimind-igw`); `rt-private-app` đi chiều ra qua NAT Gateway (`medimind-nat-gw` đặt tại `public-a`); `rt-private-db` chỉ có tuyến nội bộ `10.0.0.0/16 local`.", "code": "IGW_ID=$(aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=medimind-igw}]' --query 'InternetGateway.InternetGatewayId' --output text)\naws ec2 attach-internet-gateway --vpc-id \"$VPC_ID\" --internet-gateway-id \"$IGW_ID\"\nEIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)\nNAT_ID=$(aws ec2 create-nat-gateway --subnet-id \"$PUB_A\" --allocation-id \"$EIP_ALLOC\" --query 'NatGateway.NatGatewayId' --output text)\naws ec2 wait nat-gateway-available --nat-gateway-ids \"$NAT_ID\"\n# Tạo 3 Route Tables: rt-public (0.0.0.0/0 -> IGW), rt-private-app (0.0.0.0/0 -> NAT), rt-private-db (local only)", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03b-cli-igw-nat.png", "caption": "Ảnh 03b (CLI) — Gắn Internet Gateway & khởi tạo NAT Gateway thành công"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03c-cli-route-tables.png", "caption": "Ảnh 03c (CLI) — Cấu hình 3 Route Tables tách biệt (Public / Private App / Private DB)"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/03-vpc-resource-map.png", "caption": "Ảnh 03 (Console) — VPC Resource Map của medimind-vpc xác nhận đủ 6 Subnets, 3 Route Tables, IGW & NAT GW"}]}, {"num": "Bước 5.3.3", "title": "Thiết lập 3 Security Groups lồng nhau (ALB → EC2 → RDS)", "summary": "Áp dụng mô hình tường lửa chuỗi (Security Group Chaining) để chặn tuyệt đối mọi truy cập trực tiếp từ Internet vào EC2 và RDS:", "bullets": ["**`sg-medimind-alb` (`medimind-alb-sg`):** Mở HTTP `80` và HTTPS `443` từ bên ngoài cho Load Balancer.", "**`sg-medimind-ec2` (`medimind-ec2-sg`):** Chỉ cho phép TCP `8000` từ nguồn **`sg-medimind-alb`**. **Không mở cổng SSH `22`** (dùng hoàn toàn SSM Session Manager).", "**`sg-medimind-rds` (`medimind-rds-sg`):** Chỉ cho phép TCP `5432` từ nguồn **`sg-medimind-ec2`**."], "code": "SG_ALB=$(aws ec2 create-security-group --group-name medimind-alb-sg --description 'ALB SG' --vpc-id \"$VPC_ID\" --query 'GroupId' --output text)\nSG_EC2=$(aws ec2 create-security-group --group-name medimind-ec2-sg --description 'EC2 SG' --vpc-id \"$VPC_ID\" --query 'GroupId' --output text)\nSG_RDS=$(aws ec2 create-security-group --group-name medimind-rds-sg --description 'RDS SG' --vpc-id \"$VPC_ID\" --query 'GroupId' --output text)\naws ec2 authorize-security-group-ingress --group-id \"$SG_EC2\" --protocol tcp --port 8000 --source-group \"$SG_ALB\"\naws ec2 authorize-security-group-ingress --group-id \"$SG_RDS\" --protocol tcp --port 5432 --source-group \"$SG_EC2\"", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/04a-cli-sg.png", "caption": "Ảnh 04a (CLI) — Khởi tạo 3 Security Groups lồng nhau trên CloudShell"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/04-security-groups.png", "caption": "Ảnh 04 (Console) — Danh sách 3 Security Groups trong medimind-vpc"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/04b-sg-ec2-inbound.png", "caption": "Ảnh 04b (Console) — Luật Inbound của sg-medimind-ec2: chỉ cho phép TCP 8000 từ sg-medimind-alb (không có SSH 22)"}]}, {"num": "Bước 5.3.4", "title": "Tạo IAM Role medimind-ec2-role (Least Privilege) & Instance Profile", "summary": "Thay vì lưu khóa AWS Access Key lên máy chủ EC2, gắn IAM Role **`medimind-ec2-role`** với đúng 3 nhóm quyền tối thiểu:", "bullets": ["**`AmazonSSMManagedInstanceCore`**: Kết nối quản trị không cần khóa SSH qua AWS Systems Manager Session Manager.", "**`AmazonEC2ContainerRegistryReadOnly`**: Chỉ được phép kéo Docker Image từ Amazon ECR.", "**Inline Policy `MediMindRuntimeAccess`**: Chỉ đọc 3 bí mật `secret:medimind/*`, đọc/ghi bucket sao lưu `medimind-clinical-backup-*` và đẩy log `/medimind/*`."], "code": "aws iam create-role --role-name medimind-ec2-role --assume-role-policy-document file:///tmp/ec2-trust.json\naws iam attach-role-policy --role-name medimind-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore\naws iam attach-role-policy --role-name medimind-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly\naws iam put-role-policy --role-name medimind-ec2-role --policy-name MediMindRuntimeAccess --policy-document file:///tmp/runtime-policy.json\naws iam create-instance-profile --instance-profile-name medimind-ec2-role\naws iam add-role-to-instance-profile --instance-profile-name medimind-ec2-role --role-name medimind-ec2-role", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/05a-cli-iam-role.png", "caption": "Ảnh 05a (CLI) — Tạo IAM Role medimind-ec2-role, gắn 3 Policies & Instance Profile"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/05-iam-role.png", "caption": "Ảnh 05 (Console) — Xác nhận IAM Role medimind-ec2-role có đủ 3 quyền đặc quyền tối thiểu"}]}, {"num": "Bước 5.3.5", "title": "Khởi tạo Amazon RDS for PostgreSQL 16.16 (medimind-pg-db)", "summary": "Tạo DB Subnet Group trải trên 2 Private DB Subnets (`10.0.21.0/24` & `10.0.22.0/24`), khởi tạo phiên bản PostgreSQL 16.16 (`db.t4g.micro`, ổ đĩa `20 GiB gp3` mã hóa KMS, tắt Public Access) để lưu trữ 7 bảng ORM của MediMind.", "code": "aws rds create-db-subnet-group --db-subnet-group-name medimind-db-subnet-group \\\n  --db-subnet-group-description 'Private DB Subnets' --subnet-ids \"$PRIV_DB_A\" \"$PRIV_DB_B\"\naws rds create-db-instance --db-instance-identifier medimind-pg-db \\\n  --db-instance-class db.t4g.micro --engine postgres --engine-version 16.16 \\\n  --master-username medimind_admin --master-user-password \"$DB_PASS\" --db-name medimind_prod \\\n  --allocated-storage 20 --storage-type gp3 --storage-encrypted \\\n  --vpc-security-group-ids \"$SG_RDS\" --db-subnet-group-name medimind-db-subnet-group --no-publicly-accessible\naws rds wait db-instance-available --db-instance-identifier medimind-pg-db", "expected": "Status: Available | Engine: PostgreSQL 16.16 | Publicly accessible: No\nEndpoint: medimind-pg-db.ctu0qcaw6hfa.ap-southeast-1.rds.amazonaws.com:5432", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/06a-cli-rds.png", "caption": "Ảnh 06a (CLI) — Khởi tạo RDS PostgreSQL 16.16 & chờ trạng thái Available"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/06-rds-available.png", "caption": "Ảnh 06 (Console) — Amazon RDS medimind-pg-db (db.t4g.micro, Private Subnet) đạt trạng thái Available"}]}, {"num": "Bước 5.3.6", "title": "Khởi tạo Amazon S3 Backup Bucket kèm Versioning & Lifecycle Policy", "summary": "Tạo kho sao lưu dài hạn `medimind-clinical-backup-<AWS_ACCOUNT_ID>` cho chỉ mục ChromaDB & bản sao lưu uploads, khóa kín toàn bộ truy cập công khai và thiết lập luật tự động chuyển tầng lưu trữ tiết kiệm chi phí.", "code": "aws s3api create-bucket --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --region ap-southeast-1 \\\n  --create-bucket-configuration LocationConstraint=ap-southeast-1\naws s3api put-public-access-block --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> \\\n  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true\naws s3api put-bucket-versioning --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --versioning-configuration Status=Enabled\naws s3api put-bucket-lifecycle-configuration --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --lifecycle-configuration file:///tmp/s3-lifecycle.json", "expected": "Block Public Access: ON | Versioning: Enabled | Lifecycle: 30d STANDARD_IA → 90d GLACIER", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/07a-cli-s3.png", "caption": "Ảnh 07a (CLI) — Tạo S3 Backup Bucket, bật Block Public Access, Versioning & Lifecycle"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/07-s3-backup.png", "caption": "Ảnh 07 (Console) — Xác nhận S3 Backup Bucket kèm Lifecycle Rule MediMindBackupArchiveTiering"}]}, {"num": "Bước 5.3.7", "title": "Khởi tạo 3 nhóm AWS Secrets Manager (Không lộ khóa vào CLI)", "summary": "Tách bí mật hệ thống thành 3 nhóm riêng biệt để dễ xoay vòng và quản trị quyền:", "bullets": ["**`medimind/app-db`**: Tự động sinh `SECRET_KEY` (64-hex) và chuỗi kết nối `DATABASE_URL` trỏ thẳng tới RDS PostgreSQL vừa tạo.", "**`medimind/auth-otp`**: Lưu `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `SPEEDSMS_ACCESS_TOKEN`.", "**`medimind/ai-providers`**: Lưu bộ khóa `GROQ_API_KEY` (4-Key Pool Failover cho Chat, OCR, Nutrition) và `GROQ_MODEL=qwen/qwen3.8-27b`."], "code": "aws secretsmanager create-secret --name medimind/app-db --secret-string \"{\\\"SECRET_KEY\\\":\\\"$APP_SECRET\\\",\\\"DATABASE_URL\\\":\\\"postgresql+psycopg2://medimind_admin:$DB_PASS@$RDS_ENDPOINT:5432/medimind_prod\\\"}\"\n# Tạo khung placeholder cho auth-otp & ai-providers, sau đó mở AWS Console -> Edit để điền khóa thật", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/08a-cli-secret-app-db.png", "caption": "Ảnh 08a (CLI) — Tạo tự động medimind/app-db & khung placeholder cho 2 nhóm bí mật còn lại"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/08-secrets-manager.png", "caption": "Ảnh 08 (Console) — Đủ 3 nhóm AWS Secrets Manager sẵn sàng nạp vào EC2 (đã ẩn giá trị mật)"}]}, {"title": "Tạo VPC S3 Gateway Endpoint (Miễn phí $0 — Tránh phí NAT Gateway)", "summary": "Khởi tạo `medimind-s3-gateway-endpoint` (`com.amazonaws.ap-southeast-1.s3`, loại `Gateway`) gắn trực tiếp vào Route Table `rt-private-app` (2 Private App Subnets) để mọi lưu lượng sao lưu/tải tệp S3 đi hoàn toàn trong mạng nội bộ AWS, vừa bảo mật vừa tiết kiệm 100% phí xử lý dữ liệu qua NAT Gateway.", "code": "export AWS_REGION=ap-southeast-1\nVPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters \"Name=cidr,Values=10.0.0.0/16\" --query 'Vpcs[0].VpcId' --output text)\nPRIV_APP_SUBNETS=$(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24\" --query 'Subnets[*].SubnetId' --output text)\nRT_PRIV_APP=$(aws ec2 describe-route-tables --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=association.subnet-id,Values=$(echo $PRIV_APP_SUBNETS | tr ' ' ',')\" --query 'RouteTables[*].RouteTableId' --output text | tr '\\t' '\\n' | sort -u | tr '\\n' ' ')\n\nVPCE_ID=$(aws ec2 create-vpc-endpoint --region $AWS_REGION --vpc-id \"$VPC_ID\" --vpc-endpoint-type Gateway --service-name \"com.amazonaws.$AWS_REGION.s3\" --route-table-ids $RT_PRIV_APP --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=medimind-s3-gateway-endpoint}]' --query 'VpcEndpoint.VpcEndpointId' --output text)\naws ec2 describe-vpc-endpoints --region $AWS_REGION --vpc-endpoint-ids \"$VPCE_ID\" --query 'VpcEndpoints[0].{EndpointId:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,State:State,RouteTables:RouteTableIds}' --output table", "expected": "EndpointId: vpce-02f65f450b37931e5 | Service: com.amazonaws.ap-southeast-1.s3 | State: available | Type: Gateway | RouteTables: rtb-0494923986fe92b92", "images": [{"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/15a-cli-s3-vpce.png", "caption": "Ảnh 15a (CLI) — Khởi tạo VPC S3 Gateway Endpoint gắn rt-private-app trên AWS CloudShell"}, {"url": "static/images/5-Workshop/5.3-AWS-Infrastructure/15-s3-vpce.png", "caption": "Ảnh 15 (Console) — VPC Endpoint medimind-s3-gateway-endpoint trạng thái Available gắn 2 Private App Subnets"}], "num": "Bước 5.3.8"}, {"num": "Bước 5.3.9", "title": "Triển khai NAT Gateway 2 (ap-southeast-1b) & Tách Route Table Multi-AZ đối xứng", "summary": "Hoàn thiện tính sẵn sàng cao (High Availability) cho lưu lượng outbound bằng cách khởi tạo **NAT Gateway 2 (`medimind-nat-gw-b`)** tại `Public Subnet B` (`10.0.2.0/24`), tách riêng Route Table `rt-private-app-b` cho `Private App Subnet B` (`10.0.12.0/24`) đi ra qua NAT Gateway 2 và gắn bổ sung `S3 Gateway Endpoint` để triệt tiêu điểm lỗi đơn (Single Point of Failure) giữa 2 AZ.", "code": "export AWS_REGION=ap-southeast-1\nVPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters \"Name=cidr,Values=10.0.0.0/16\" --query 'Vpcs[0].VpcId' --output text)\nPUB_B=$(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.2.0/24\" --query 'Subnets[0].SubnetId' --output text)\nPRIV_APP_B=$(aws ec2 describe-subnets --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=cidr-block,Values=10.0.12.0/24\" --query 'Subnets[0].SubnetId' --output text)\nVPCE_ID=$(aws ec2 describe-vpc-endpoints --region $AWS_REGION --filters \"Name=vpc-id,Values=$VPC_ID\" \"Name=vpc-endpoint-type,Values=Gateway\" --query 'VpcEndpoints[0].VpcEndpointId' --output text)\n\n# 1. Cấp Elastic IP & tạo NAT Gateway 2 tại Public Subnet B (ap-southeast-1b)\nEIP_B=$(aws ec2 allocate-address --region $AWS_REGION --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=medimind-nat-eip-b}]' --query 'AllocationId' --output text)\nNAT_B=$(aws ec2 create-nat-gateway --region $AWS_REGION --subnet-id \"$PUB_B\" --allocation-id \"$EIP_B\" --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=medimind-nat-gw-b}]' --query 'NatGateway.NatGatewayId' --output text)\naws ec2 wait nat-gateway-available --region $AWS_REGION --nat-gateway-ids \"$NAT_B\"\n\n# 2. Tạo Route Table riêng rt-private-app-b -> NAT Gateway 2 & gắn S3 Gateway Endpoint\nRT_B=$(aws ec2 create-route-table --region $AWS_REGION --vpc-id \"$VPC_ID\" --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-private-app-b}]' --query 'RouteTable.RouteTableId' --output text)\naws ec2 create-route --region $AWS_REGION --route-table-id \"$RT_B\" --destination-cidr-block 0.0.0.0/0 --nat-gateway-id \"$NAT_B\" >/dev/null\nASSOC_B=$(aws ec2 describe-route-tables --region $AWS_REGION --filters \"Name=association.subnet-id,Values=$PRIV_APP_B\" --query \"RouteTables[0].Associations[?SubnetId=='$PRIV_APP_B'].RouteTableAssociationId\" --output text)\naws ec2 replace-route-table-association --region $AWS_REGION --association-id \"$ASSOC_B\" --route-table-id \"$RT_B\" >/dev/null\naws ec2 modify-vpc-endpoint --region $AWS_REGION --vpc-endpoint-id \"$VPCE_ID\" --add-route-table-ids \"$RT_B\" >/dev/null\n\n# 3. Xác nhận đủ 2 NAT Gateways Available trên 2 AZ\naws ec2 describe-nat-gateways --region $AWS_REGION --filter \"Name=vpc-id,Values=$VPC_ID\" \"Name=state,Values=available\" --query 'NatGateways[*].{NatId:NatGatewayId,Name:Tags[?Key==`Name`]|[0].Value,Subnet:SubnetId,PublicIP:NatGatewayAddresses[0].PublicIp,State:State}' --output table", "expected": "medimind-nat-gw   | nat-00781023e54fbbc54 | available | Public Subnet A (AZ-1a)\nmedimind-nat-gw-b | nat-07bbbfc1d6a05203e | available | Public Subnet B (AZ-1b)", "images": [{"url": "static/images/5-Workshop/5.5-Operations/cli-steps/20a-cli-nat-gw-2.png", "caption": "Ảnh 20a (CLI) — Khởi tạo NAT Gateway 2, tách rt-private-app-b & xác nhận 2 NAT Gateways Available"}, {"url": "static/images/5-Workshop/5.5-Operations/20-nat-gw-2.png", "caption": "Ảnh 20 (Console) — AWS VPC NAT Gateways xác nhận đủ 2 NAT Gateways (AZ-1a & AZ-1b) trạng thái Available"}]}]}
report_index: 2
report_shared: {"code": "5.3"}
title: "Hạ tầng AWS 3 tầng"
date: "2026-11-27"
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

# 5.3. Khởi tạo Hạ tầng AWS 3 Tầng (VPC, SG, IAM, RDS, S3 & Secrets)

Khởi tạo tuần tự 9 lớp nền tảng bảo mật & Multi-AZ bằng AWS CloudShell tại ap-southeast-1: VPC 6 Subnets, IGW, 2 NAT Gateways đối xứng, 3 Security Groups lồng nhau, IAM Role đặc quyền tối thiểu, RDS PostgreSQL 16.16, S3 Backup, 3 nhóm Secrets Manager và S3 Gateway Endpoint.

## Bước 5.3.1 — Tạo VPC (10.0.0.0/16) & 6 Subnets trải đều 2 Availability Zones (1a / 1b)

Thiết lập mạng riêng ảo `medimind-vpc` (bật DNS Support & DNS Hostnames) và chia làm 3 tầng mạng tách biệt trên 2 AZ (`ap-southeast-1a` & `ap-southeast-1b`):

| Tầng mạng | Subnet AZ-1a | Subnet AZ-1b | Mục đích |
| --- | --- | --- | --- |
| Public Tier | `10.0.1.0/24` (`medimind-public-a`) | `10.0.2.0/24` (`medimind-public-b`) | Đặt Application Load Balancer & NAT Gateway. |
| Private App Tier | `10.0.11.0/24` (`medimind-private-app-a`) | `10.0.12.0/24` (`medimind-private-app-b`) | Đặt EC2 chạy Docker container MediMind (không Public IP). |
| Private DB Tier | `10.0.21.0/24` (`medimind-private-db-a`) | `10.0.22.0/24` (`medimind-private-db-b`) | Đặt Amazon RDS PostgreSQL (cô lập hoàn toàn khỏi Internet). |

```bash
export AWS_REGION=ap-southeast-1
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region $AWS_REGION \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=medimind-vpc}]' --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames '{"Value":true}'
# Tạo đủ 6 Subnets: public-a/b, private-app-a/b, private-db-a/b và lưu vào ~/medimind_ids.env
```

**Kết quả kỳ vọng:**
```text
VPC_ID=vpc-0173e16d7c27c35f8 (10.0.0.0/16) + 6 Subnets created across ap-southeast-1a & 1b
```

![Ảnh 03a (CLI) — Khởi tạo medimind-vpc và 6 Subnets qua 2 AZ trên AWS CloudShell](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03a-cli-vpc-subnets.png)
*Ảnh 03a (CLI) — Khởi tạo medimind-vpc và 6 Subnets qua 2 AZ trên AWS CloudShell*

## Bước 5.3.2 — Gắn Internet Gateway, NAT Gateway & 3 Route Tables cô lập

Định tuyến chuẩn 3 tầng: `rt-public` đi trực tiếp qua Internet Gateway (`medimind-igw`); `rt-private-app` đi chiều ra qua NAT Gateway (`medimind-nat-gw` đặt tại `public-a`); `rt-private-db` chỉ có tuyến nội bộ `10.0.0.0/16 local`.

```bash
IGW_ID=$(aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=medimind-igw}]' --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id "$VPC_ID" --internet-gateway-id "$IGW_ID"
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id "$PUB_A" --allocation-id "$EIP_ALLOC" --query 'NatGateway.NatGatewayId' --output text)
aws ec2 wait nat-gateway-available --nat-gateway-ids "$NAT_ID"
# Tạo 3 Route Tables: rt-public (0.0.0.0/0 -> IGW), rt-private-app (0.0.0.0/0 -> NAT), rt-private-db (local only)
```

![Ảnh 03b (CLI) — Gắn Internet Gateway & khởi tạo NAT Gateway thành công](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03b-cli-igw-nat.png)
*Ảnh 03b (CLI) — Gắn Internet Gateway & khởi tạo NAT Gateway thành công*

![Ảnh 03c (CLI) — Cấu hình 3 Route Tables tách biệt (Public / Private App / Private DB)](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/03c-cli-route-tables.png)
*Ảnh 03c (CLI) — Cấu hình 3 Route Tables tách biệt (Public / Private App / Private DB)*

![Ảnh 03 (Console) — VPC Resource Map của medimind-vpc xác nhận đủ 6 Subnets, 3 Route Tables, IGW & NAT GW](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/03-vpc-resource-map.png)
*Ảnh 03 (Console) — VPC Resource Map của medimind-vpc xác nhận đủ 6 Subnets, 3 Route Tables, IGW & NAT GW*

## Bước 5.3.3 — Thiết lập 3 Security Groups lồng nhau (ALB → EC2 → RDS)

Áp dụng mô hình tường lửa chuỗi (Security Group Chaining) để chặn tuyệt đối mọi truy cập trực tiếp từ Internet vào EC2 và RDS:

- **`sg-medimind-alb` (`medimind-alb-sg`):** Mở HTTP `80` và HTTPS `443` từ bên ngoài cho Load Balancer.
- **`sg-medimind-ec2` (`medimind-ec2-sg`):** Chỉ cho phép TCP `8000` từ nguồn **`sg-medimind-alb`**. **Không mở cổng SSH `22`** (dùng hoàn toàn SSM Session Manager).
- **`sg-medimind-rds` (`medimind-rds-sg`):** Chỉ cho phép TCP `5432` từ nguồn **`sg-medimind-ec2`**.

```bash
SG_ALB=$(aws ec2 create-security-group --group-name medimind-alb-sg --description 'ALB SG' --vpc-id "$VPC_ID" --query 'GroupId' --output text)
SG_EC2=$(aws ec2 create-security-group --group-name medimind-ec2-sg --description 'EC2 SG' --vpc-id "$VPC_ID" --query 'GroupId' --output text)
SG_RDS=$(aws ec2 create-security-group --group-name medimind-rds-sg --description 'RDS SG' --vpc-id "$VPC_ID" --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id "$SG_EC2" --protocol tcp --port 8000 --source-group "$SG_ALB"
aws ec2 authorize-security-group-ingress --group-id "$SG_RDS" --protocol tcp --port 5432 --source-group "$SG_EC2"
```

![Ảnh 04a (CLI) — Khởi tạo 3 Security Groups lồng nhau trên CloudShell](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/04a-cli-sg.png)
*Ảnh 04a (CLI) — Khởi tạo 3 Security Groups lồng nhau trên CloudShell*

![Ảnh 04 (Console) — Danh sách 3 Security Groups trong medimind-vpc](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/04-security-groups.png)
*Ảnh 04 (Console) — Danh sách 3 Security Groups trong medimind-vpc*

![Ảnh 04b (Console) — Luật Inbound của sg-medimind-ec2: chỉ cho phép TCP 8000 từ sg-medimind-alb (không có SSH 22)](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/04b-sg-ec2-inbound.png)
*Ảnh 04b (Console) — Luật Inbound của sg-medimind-ec2: chỉ cho phép TCP 8000 từ sg-medimind-alb (không có SSH 22)*

## Bước 5.3.4 — Tạo IAM Role medimind-ec2-role (Least Privilege) & Instance Profile

Thay vì lưu khóa AWS Access Key lên máy chủ EC2, gắn IAM Role **`medimind-ec2-role`** với đúng 3 nhóm quyền tối thiểu:

- **`AmazonSSMManagedInstanceCore`**: Kết nối quản trị không cần khóa SSH qua AWS Systems Manager Session Manager.
- **`AmazonEC2ContainerRegistryReadOnly`**: Chỉ được phép kéo Docker Image từ Amazon ECR.
- **Inline Policy `MediMindRuntimeAccess`**: Chỉ đọc 3 bí mật `secret:medimind/*`, đọc/ghi bucket sao lưu `medimind-clinical-backup-*` và đẩy log `/medimind/*`.

```bash
aws iam create-role --role-name medimind-ec2-role --assume-role-policy-document file:///tmp/ec2-trust.json
aws iam attach-role-policy --role-name medimind-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam attach-role-policy --role-name medimind-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam put-role-policy --role-name medimind-ec2-role --policy-name MediMindRuntimeAccess --policy-document file:///tmp/runtime-policy.json
aws iam create-instance-profile --instance-profile-name medimind-ec2-role
aws iam add-role-to-instance-profile --instance-profile-name medimind-ec2-role --role-name medimind-ec2-role
```

![Ảnh 05a (CLI) — Tạo IAM Role medimind-ec2-role, gắn 3 Policies & Instance Profile](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/05a-cli-iam-role.png)
*Ảnh 05a (CLI) — Tạo IAM Role medimind-ec2-role, gắn 3 Policies & Instance Profile*

![Ảnh 05 (Console) — Xác nhận IAM Role medimind-ec2-role có đủ 3 quyền đặc quyền tối thiểu](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/05-iam-role.png)
*Ảnh 05 (Console) — Xác nhận IAM Role medimind-ec2-role có đủ 3 quyền đặc quyền tối thiểu*

## Bước 5.3.5 — Khởi tạo Amazon RDS for PostgreSQL 16.16 (medimind-pg-db)

Tạo DB Subnet Group trải trên 2 Private DB Subnets (`10.0.21.0/24` & `10.0.22.0/24`), khởi tạo phiên bản PostgreSQL 16.16 (`db.t4g.micro`, ổ đĩa `20 GiB gp3` mã hóa KMS, tắt Public Access) để lưu trữ 7 bảng ORM của MediMind.

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

**Kết quả kỳ vọng:**
```text
Status: Available | Engine: PostgreSQL 16.16 | Publicly accessible: No
Endpoint: medimind-pg-db.ctu0qcaw6hfa.ap-southeast-1.rds.amazonaws.com:5432
```

![Ảnh 06a (CLI) — Khởi tạo RDS PostgreSQL 16.16 & chờ trạng thái Available](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/06a-cli-rds.png)
*Ảnh 06a (CLI) — Khởi tạo RDS PostgreSQL 16.16 & chờ trạng thái Available*

![Ảnh 06 (Console) — Amazon RDS medimind-pg-db (db.t4g.micro, Private Subnet) đạt trạng thái Available](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/06-rds-available.png)
*Ảnh 06 (Console) — Amazon RDS medimind-pg-db (db.t4g.micro, Private Subnet) đạt trạng thái Available*

## Bước 5.3.6 — Khởi tạo Amazon S3 Backup Bucket kèm Versioning & Lifecycle Policy

Tạo kho sao lưu dài hạn `medimind-clinical-backup-<AWS_ACCOUNT_ID>` cho chỉ mục ChromaDB & bản sao lưu uploads, khóa kín toàn bộ truy cập công khai và thiết lập luật tự động chuyển tầng lưu trữ tiết kiệm chi phí.

```bash
aws s3api create-bucket --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1
aws s3api put-public-access-block --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-versioning --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --versioning-configuration Status=Enabled
aws s3api put-bucket-lifecycle-configuration --bucket medimind-clinical-backup-<AWS_ACCOUNT_ID> --lifecycle-configuration file:///tmp/s3-lifecycle.json
```

**Kết quả kỳ vọng:**
```text
Block Public Access: ON | Versioning: Enabled | Lifecycle: 30d STANDARD_IA → 90d GLACIER
```

![Ảnh 07a (CLI) — Tạo S3 Backup Bucket, bật Block Public Access, Versioning & Lifecycle](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/07a-cli-s3.png)
*Ảnh 07a (CLI) — Tạo S3 Backup Bucket, bật Block Public Access, Versioning & Lifecycle*

![Ảnh 07 (Console) — Xác nhận S3 Backup Bucket kèm Lifecycle Rule MediMindBackupArchiveTiering](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/07-s3-backup.png)
*Ảnh 07 (Console) — Xác nhận S3 Backup Bucket kèm Lifecycle Rule MediMindBackupArchiveTiering*

## Bước 5.3.7 — Khởi tạo 3 nhóm AWS Secrets Manager (Không lộ khóa vào CLI)

Tách bí mật hệ thống thành 3 nhóm riêng biệt để dễ xoay vòng và quản trị quyền:

- **`medimind/app-db`**: Tự động sinh `SECRET_KEY` (64-hex) và chuỗi kết nối `DATABASE_URL` trỏ thẳng tới RDS PostgreSQL vừa tạo.
- **`medimind/auth-otp`**: Lưu `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `SPEEDSMS_ACCESS_TOKEN`.
- **`medimind/ai-providers`**: Lưu bộ khóa `GROQ_API_KEY` (4-Key Pool Failover cho Chat, OCR, Nutrition) và `GROQ_MODEL=qwen/qwen3.8-27b`.

```bash
aws secretsmanager create-secret --name medimind/app-db --secret-string "{\"SECRET_KEY\":\"$APP_SECRET\",\"DATABASE_URL\":\"postgresql+psycopg2://medimind_admin:$DB_PASS@$RDS_ENDPOINT:5432/medimind_prod\"}"
# Tạo khung placeholder cho auth-otp & ai-providers, sau đó mở AWS Console -> Edit để điền khóa thật
```

![Ảnh 08a (CLI) — Tạo tự động medimind/app-db & khung placeholder cho 2 nhóm bí mật còn lại](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/08a-cli-secret-app-db.png)
*Ảnh 08a (CLI) — Tạo tự động medimind/app-db & khung placeholder cho 2 nhóm bí mật còn lại*

![Ảnh 08 (Console) — Đủ 3 nhóm AWS Secrets Manager sẵn sàng nạp vào EC2 (đã ẩn giá trị mật)](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/08-secrets-manager.png)
*Ảnh 08 (Console) — Đủ 3 nhóm AWS Secrets Manager sẵn sàng nạp vào EC2 (đã ẩn giá trị mật)*

## Bước 5.3.8 — Tạo VPC S3 Gateway Endpoint (Miễn phí $0 — Tránh phí NAT Gateway)

Khởi tạo `medimind-s3-gateway-endpoint` (`com.amazonaws.ap-southeast-1.s3`, loại `Gateway`) gắn trực tiếp vào Route Table `rt-private-app` (2 Private App Subnets) để mọi lưu lượng sao lưu/tải tệp S3 đi hoàn toàn trong mạng nội bộ AWS, vừa bảo mật vừa tiết kiệm 100% phí xử lý dữ liệu qua NAT Gateway.

```bash
export AWS_REGION=ap-southeast-1
VPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters "Name=cidr,Values=10.0.0.0/16" --query 'Vpcs[0].VpcId' --output text)
PRIV_APP_SUBNETS=$(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.11.0/24,10.0.12.0/24" --query 'Subnets[*].SubnetId' --output text)
RT_PRIV_APP=$(aws ec2 describe-route-tables --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.subnet-id,Values=$(echo $PRIV_APP_SUBNETS | tr ' ' ',')" --query 'RouteTables[*].RouteTableId' --output text | tr '\t' '\n' | sort -u | tr '\n' ' ')

VPCE_ID=$(aws ec2 create-vpc-endpoint --region $AWS_REGION --vpc-id "$VPC_ID" --vpc-endpoint-type Gateway --service-name "com.amazonaws.$AWS_REGION.s3" --route-table-ids $RT_PRIV_APP --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=medimind-s3-gateway-endpoint}]' --query 'VpcEndpoint.VpcEndpointId' --output text)
aws ec2 describe-vpc-endpoints --region $AWS_REGION --vpc-endpoint-ids "$VPCE_ID" --query 'VpcEndpoints[0].{EndpointId:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,State:State,RouteTables:RouteTableIds}' --output table
```

**Kết quả kỳ vọng:**
```text
EndpointId: vpce-02f65f450b37931e5 | Service: com.amazonaws.ap-southeast-1.s3 | State: available | Type: Gateway | RouteTables: rtb-0494923986fe92b92
```

![Ảnh 15a (CLI) — Khởi tạo VPC S3 Gateway Endpoint gắn rt-private-app trên AWS CloudShell](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/cli-steps/15a-cli-s3-vpce.png)
*Ảnh 15a (CLI) — Khởi tạo VPC S3 Gateway Endpoint gắn rt-private-app trên AWS CloudShell*

![Ảnh 15 (Console) — VPC Endpoint medimind-s3-gateway-endpoint trạng thái Available gắn 2 Private App Subnets](../../../static/images/5-Workshop/5.3-AWS-Infrastructure/15-s3-vpce.png)
*Ảnh 15 (Console) — VPC Endpoint medimind-s3-gateway-endpoint trạng thái Available gắn 2 Private App Subnets*

## Bước 5.3.9 — Triển khai NAT Gateway 2 (ap-southeast-1b) & Tách Route Table Multi-AZ đối xứng

Hoàn thiện tính sẵn sàng cao (High Availability) cho lưu lượng outbound bằng cách khởi tạo **NAT Gateway 2 (`medimind-nat-gw-b`)** tại `Public Subnet B` (`10.0.2.0/24`), tách riêng Route Table `rt-private-app-b` cho `Private App Subnet B` (`10.0.12.0/24`) đi ra qua NAT Gateway 2 và gắn bổ sung `S3 Gateway Endpoint` để triệt tiêu điểm lỗi đơn (Single Point of Failure) giữa 2 AZ.

```bash
export AWS_REGION=ap-southeast-1
VPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION --filters "Name=cidr,Values=10.0.0.0/16" --query 'Vpcs[0].VpcId' --output text)
PUB_B=$(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.2.0/24" --query 'Subnets[0].SubnetId' --output text)
PRIV_APP_B=$(aws ec2 describe-subnets --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=10.0.12.0/24" --query 'Subnets[0].SubnetId' --output text)
VPCE_ID=$(aws ec2 describe-vpc-endpoints --region $AWS_REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=vpc-endpoint-type,Values=Gateway" --query 'VpcEndpoints[0].VpcEndpointId' --output text)

# 1. Cấp Elastic IP & tạo NAT Gateway 2 tại Public Subnet B (ap-southeast-1b)
EIP_B=$(aws ec2 allocate-address --region $AWS_REGION --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=medimind-nat-eip-b}]' --query 'AllocationId' --output text)
NAT_B=$(aws ec2 create-nat-gateway --region $AWS_REGION --subnet-id "$PUB_B" --allocation-id "$EIP_B" --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=medimind-nat-gw-b}]' --query 'NatGateway.NatGatewayId' --output text)
aws ec2 wait nat-gateway-available --region $AWS_REGION --nat-gateway-ids "$NAT_B"

# 2. Tạo Route Table riêng rt-private-app-b -> NAT Gateway 2 & gắn S3 Gateway Endpoint
RT_B=$(aws ec2 create-route-table --region $AWS_REGION --vpc-id "$VPC_ID" --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-private-app-b}]' --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --region $AWS_REGION --route-table-id "$RT_B" --destination-cidr-block 0.0.0.0/0 --nat-gateway-id "$NAT_B" >/dev/null
ASSOC_B=$(aws ec2 describe-route-tables --region $AWS_REGION --filters "Name=association.subnet-id,Values=$PRIV_APP_B" --query "RouteTables[0].Associations[?SubnetId=='$PRIV_APP_B'].RouteTableAssociationId" --output text)
aws ec2 replace-route-table-association --region $AWS_REGION --association-id "$ASSOC_B" --route-table-id "$RT_B" >/dev/null
aws ec2 modify-vpc-endpoint --region $AWS_REGION --vpc-endpoint-id "$VPCE_ID" --add-route-table-ids "$RT_B" >/dev/null

# 3. Xác nhận đủ 2 NAT Gateways Available trên 2 AZ
aws ec2 describe-nat-gateways --region $AWS_REGION --filter "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" --query 'NatGateways[*].{NatId:NatGatewayId,Name:Tags[?Key==`Name`]|[0].Value,Subnet:SubnetId,PublicIP:NatGatewayAddresses[0].PublicIp,State:State}' --output table
```

**Kết quả kỳ vọng:**
```text
medimind-nat-gw   | nat-00781023e54fbbc54 | available | Public Subnet A (AZ-1a)
medimind-nat-gw-b | nat-07bbbfc1d6a05203e | available | Public Subnet B (AZ-1b)
```

![Ảnh 20a (CLI) — Khởi tạo NAT Gateway 2, tách rt-private-app-b & xác nhận 2 NAT Gateways Available](../../../static/images/5-Workshop/5.5-Operations/cli-steps/20a-cli-nat-gw-2.png)
*Ảnh 20a (CLI) — Khởi tạo NAT Gateway 2, tách rt-private-app-b & xác nhận 2 NAT Gateways Available*

![Ảnh 20 (Console) — AWS VPC NAT Gateways xác nhận đủ 2 NAT Gateways (AZ-1a & AZ-1b) trạng thái Available](../../../static/images/5-Workshop/5.5-Operations/20-nat-gw-2.png)
*Ảnh 20 (Console) — AWS VPC NAT Gateways xác nhận đủ 2 NAT Gateways (AZ-1a & AZ-1b) trạng thái Available*
