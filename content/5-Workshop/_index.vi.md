---
title: "Workshop"
date: "2026-11-27"
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# 5. Workshop Thực hành — Xây dựng & Triển khai MediMind Clinical AI trên AWS

Hướng dẫn thực hành theo từng bước (`Step-by-Step`) triển khai nền tảng trợ lý y tế lâm sàng **MediMind Clinical AI** trên kiến trúc **AWS VPC 3 tầng (`10.0.0.0/16`)** tại khu vực **`ap-southeast-1` (Singapore)**.

- **Ứng dụng đang chạy Live HTTPS:** https://d2typ9pphzk4as.cloudfront.net
- **Mã nguồn triển khai (`main`):** https://github.com/khiemdong2005/chatbot_medical

## Lộ trình thực hành (6 chương)

| Chương | Nội dung | Trạng thái |
| :--- | :--- | :---: |
| **5.1** | [Tổng quan về Workshop & Kiến trúc AWS](5.1-Workshop-overview/) | ✅ Hoàn thành |
| **5.2** | [Các bước chuẩn bị (Region, CloudShell, Mã nguồn)](5.2-Prerequisite/) | ✅ Hoàn thành |
| **5.3** | [Khởi tạo Hạ tầng AWS 3 Tầng Multi-AZ (9 bước: VPC → 2 NAT GW & S3 Endpoint)](5.3-AWS-Infrastructure/) | ✅ Hoàn thành |
| **5.4** | [Triển khai MediMind AI, EFS & Auto Scaling 2 EC2 (9 bước: ECR → 2/2 Healthy Multi-AZ)](5.4-Deploy-MediMind/) | ✅ Hoàn thành |
| **5.5** | [Vận hành CloudWatch, Kiểm soát Chi phí & Kiểm thử Live (5 bước)](5.5-Operations/) | ✅ Hoàn thành |
| **5.6** | [Dọn dẹp tài nguyên AWS (Kịch bản sẵn sàng, chờ chạy sau khi bảo vệ live)](5.6-Cleanup/) | ⏳ Chưa thực hiện |
