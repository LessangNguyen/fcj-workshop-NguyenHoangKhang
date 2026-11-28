---
report_key: "workshopChapters"
report_language: "vi"
report_fields: {"shortTitle": "Các bước chuẩn bị", "customTitle": "5.2. Các bước chuẩn bị (Prerequisites)", "summary": "Chọn đúng AWS Region ap-southeast-1 (Singapore), xác thực AWS CLI trên CloudShell và kiểm tra mã nguồn, khóa API trước khi khởi tạo hạ tầng.", "content": "Kiểm tra điều kiện tiên quyết trước khi dựng hạ tầng: chọn khu vực Asia Pacific (Singapore) ap-southeast-1 trên AWS Console, xác thực danh tính IAM qua AWS CloudShell (aws sts get-caller-identity) và rà soát kho mã nguồn nhánh main cùng các khóa API dịch vụ ngoài.", "notes": "", "steps": [{"num": "Bước 5.2.1", "title": "Chọn khu vực Asia Pacific (Singapore) ap-southeast-1 trên AWS Console", "summary": "Đăng nhập AWS Management Console và chuyển thanh chọn Region góc trên bên phải sang **Asia Pacific (Singapore) `ap-southeast-1`** để giảm độ trễ cho người dùng tại Việt Nam.", "bullets": ["Tài khoản AWS cần có quyền khởi tạo VPC, EC2, EBS, RDS, S3, ECR, CodeBuild, ALB, CloudFront, WAFv2, Secrets Manager và IAM.", "Mọi tài nguyên khu vực (VPC, RDS, EC2, ALB, S3, ECR, Secrets) đều nằm đồng bộ tại `ap-southeast-1`."], "images": [{"url": "static/images/5-Workshop/5.2-Prerequisite/01-region.png", "caption": "Ảnh 01 — Chọn Region Asia Pacific (Singapore) ap-southeast-1 trên AWS Management Console"}]}, {"num": "Bước 5.2.2", "title": "Mở AWS CloudShell & xác thực danh tính AWS CLI", "summary": "Mở **AWS CloudShell** ngay trên thanh công cụ AWS Console để chạy toàn bộ lệnh khởi tạo hạ tầng sạch sẽ, không cần cài đặt AWS CLI hay lưu access key trên máy cá nhân.", "code": "export AWS_REGION=ap-southeast-1\naws configure set region ap-southeast-1\naws sts get-caller-identity\naws configure get region", "expected": "{\n    \"UserId\": \"AIDA...\",\n    \"Account\": \"<AWS_ACCOUNT_ID>\",\n    \"Arn\": \"arn:aws:iam::<AWS_ACCOUNT_ID>:user/...\"\n}\nap-southeast-1", "images": [{"url": "static/images/5-Workshop/5.2-Prerequisite/02-aws-identity.png", "caption": "Ảnh 02 — Xác thực danh tính AWS CLI (aws sts get-caller-identity) & Region ap-southeast-1 trên CloudShell"}]}, {"num": "Bước 5.2.3", "title": "Chuẩn bị mã nguồn nhánh main & danh sách khóa tích hợp ngoài", "summary": "Đảm bảo kho mã nguồn `chatbot_medical` trên nhánh `main` đã tích hợp `scripts/docker_entrypoint.py` (tự động kiểm tra và nạp chỉ mục ChromaDB 364 tài liệu khi khởi động container lần đầu) và chuẩn bị sẵn các khóa API cá nhân để nạp vào AWS Secrets Manager ở Chương 5.3:", "table": {"headers": ["Nhóm cấu hình", "Thành phần cần chuẩn bị", "Mục đích sử dụng"], "rows": [["Mã nguồn & Container", "Nhánh `main` (`Dockerfile` non-root `appuser`, `scripts/docker_entrypoint.py`)", "AWS CodeBuild kéo trực tiếp để đóng gói image `v1.0.0` (~334 MB)."], ["AI & Vision OCR", "4 Groq API Keys (4-Key Pool Failover tách riêng Chat, OCR, Dinh dưỡng & dự phòng cho `qwen/qwen3.8-27b`)", "Tư vấn triệu chứng, đối soát ảnh đơn thuốc OCR & lập thực đơn dinh dưỡng."], ["Xác thực & OTP", "Google OAuth 2.0 Client ID/Secret, Gmail SMTP App Password, SpeedSMS Token", "Đăng nhập nhanh Google, gửi mã OTP 6 số & email xác nhận đặt lịch khám."]]}}]}
report_index: 1
report_shared: {"code": "5.2"}
title: "Các bước chuẩn bị"
date: "2026-11-27"
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

# 5.2. Các bước chuẩn bị (Prerequisites)

Chọn đúng AWS Region ap-southeast-1 (Singapore), xác thực AWS CLI trên CloudShell và kiểm tra mã nguồn, khóa API trước khi khởi tạo hạ tầng.

## Bước 5.2.1 — Chọn khu vực Asia Pacific (Singapore) ap-southeast-1 trên AWS Console

Đăng nhập AWS Management Console và chuyển thanh chọn Region góc trên bên phải sang **Asia Pacific (Singapore) `ap-southeast-1`** để giảm độ trễ cho người dùng tại Việt Nam.

- Tài khoản AWS cần có quyền khởi tạo VPC, EC2, EBS, RDS, S3, ECR, CodeBuild, ALB, CloudFront, WAFv2, Secrets Manager và IAM.
- Mọi tài nguyên khu vực (VPC, RDS, EC2, ALB, S3, ECR, Secrets) đều nằm đồng bộ tại `ap-southeast-1`.

![Ảnh 01 — Chọn Region Asia Pacific (Singapore) ap-southeast-1 trên AWS Management Console](../../../static/images/5-Workshop/5.2-Prerequisite/01-region.png)
*Ảnh 01 — Chọn Region Asia Pacific (Singapore) ap-southeast-1 trên AWS Management Console*

## Bước 5.2.2 — Mở AWS CloudShell & xác thực danh tính AWS CLI

Mở **AWS CloudShell** ngay trên thanh công cụ AWS Console để chạy toàn bộ lệnh khởi tạo hạ tầng sạch sẽ, không cần cài đặt AWS CLI hay lưu access key trên máy cá nhân.

```bash
export AWS_REGION=ap-southeast-1
aws configure set region ap-southeast-1
aws sts get-caller-identity
aws configure get region
```

**Kết quả kỳ vọng:**
```text
{
    "UserId": "AIDA...",
    "Account": "<AWS_ACCOUNT_ID>",
    "Arn": "arn:aws:iam::<AWS_ACCOUNT_ID>:user/..."
}
ap-southeast-1
```

![Ảnh 02 — Xác thực danh tính AWS CLI (aws sts get-caller-identity) & Region ap-southeast-1 trên CloudShell](../../../static/images/5-Workshop/5.2-Prerequisite/02-aws-identity.png)
*Ảnh 02 — Xác thực danh tính AWS CLI (aws sts get-caller-identity) & Region ap-southeast-1 trên CloudShell*

## Bước 5.2.3 — Chuẩn bị mã nguồn nhánh main & danh sách khóa tích hợp ngoài

Đảm bảo kho mã nguồn `chatbot_medical` trên nhánh `main` đã tích hợp `scripts/docker_entrypoint.py` (tự động kiểm tra và nạp chỉ mục ChromaDB 364 tài liệu khi khởi động container lần đầu) và chuẩn bị sẵn các khóa API cá nhân để nạp vào AWS Secrets Manager ở Chương 5.3:

| Nhóm cấu hình | Thành phần cần chuẩn bị | Mục đích sử dụng |
| --- | --- | --- |
| Mã nguồn & Container | Nhánh `main` (`Dockerfile` non-root `appuser`, `scripts/docker_entrypoint.py`) | AWS CodeBuild kéo trực tiếp để đóng gói image `v1.0.0` (~334 MB). |
| AI & Vision OCR | 4 Groq API Keys (4-Key Pool Failover tách riêng Chat, OCR, Dinh dưỡng & dự phòng cho `qwen/qwen3.8-27b`) | Tư vấn triệu chứng, đối soát ảnh đơn thuốc OCR & lập thực đơn dinh dưỡng. |
| Xác thực & OTP | Google OAuth 2.0 Client ID/Secret, Gmail SMTP App Password, SpeedSMS Token | Đăng nhập nhanh Google, gửi mã OTP 6 số & email xác nhận đặt lịch khám. |
