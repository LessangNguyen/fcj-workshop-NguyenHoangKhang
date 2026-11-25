---
title: "Bản đề xuất"
date: 2026-09-14
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# MediMind Clinical AI  
## Nền tảng Trợ lý Y tế Lâm sàng, Đối soát Đơn thuốc & Chăm sóc Sức khỏe Thông minh trên AWS  

### 1. Tóm tắt điều hành  
MediMind Clinical AI được thiết kế nhằm hỗ trợ sàng lọc triệu chứng ban đầu, đối soát đơn thuốc/phiếu xét nghiệm và tư vấn dinh dưỡng cá nhân hóa cho người bệnh và bác sĩ tại Việt Nam. Hệ thống kết hợp kiến trúc **Hybrid RAG (BM25 + ChromaDB Vector + Cross-Encoder Reranker)** trên 364 tài liệu y khoa chuẩn hóa (*Bộ Y Tế, kcb.vn, MedlinePlus NIH*), cơ chế bảo vệ hai lớp (**Clinical Triage & Double Guardrails**) và **Pipeline OCR Đối soát 8 Bước** xử lý song song EasyOCR cùng Vision LLM (`nvidia/nemotron-3.5-super-vl:free` qua OpenRouter và Groq API). Nền tảng được đóng gói Docker và triển khai trên hạ tầng **Amazon Web Services (AWS)** khu vực `ap-southeast-1` (Singapore) với kiến trúc VPC 3 tầng mạng tách biệt, tuân thủ **AWS Well-Architected Framework** và chuẩn bảo mật dữ liệu y tế **HIPAA**.  

### 2. Tuyên bố vấn đề  
*Vấn đề hiện tại*  
Các bệnh viện tuyến đầu thường xuyên quá tải 150% – 200% công suất, khiến người bệnh chờ đợi 2 – 4 giờ và bác sĩ thiếu bản tóm tắt bệnh sử trước khám. Đồng thời, việc người dân tự tra cứu triệu chứng qua các công cụ AI không kiểm chứng dễ dẫn đến bỏ lỡ "giờ vàng" cấp cứu, kháng kháng sinh hoặc nhầm lẫn liều dùng từ đơn thuốc viết tay mờ. Các chatbot thông thường cũng dễ mắc bẫy phủ định (báo động giả khi người bệnh nhập *"không khó thở"*) hoặc tự bịa nguồn trích dẫn y khoa và thực đơn gây dị ứng.  

*Giải pháp*  
Nền tảng xây dựng quy trình chăm sóc sức khỏe khép kín gồm 5 phân hệ: (1) Sàng lọc triệu chứng đa lượt phân tầng 3 mức nguy cơ (**Xanh – Vàng – Đỏ**), xử lý tiếng Việt không dấu và bẫy phủ định để kích hoạt gọi **Cấp cứu 115** kịp thời; (2) Pipeline OCR 8 bước chạy song song EasyOCR và Vision LLM, đối chiếu chéo từng trường dữ liệu, gắn cờ `needs_review` khi có sai lệch và chuyển thành lịch nhắc uống thuốc trực quan (*Sáng – Trưa – Chiều – Tối; Trước/Sau ăn*) sau khi người dùng xác nhận; (3) Tra cứu tri thức có căn cứ từ 364 tài liệu y khoa chuẩn; (4) Tư vấn dinh dưỡng lâm sàng sàng lọc tất định (*Fail-Closed Allergy Filtering*) từ **2.462 công thức món Việt (ViFoodRec)**; và (5) Xuất hồ sơ tóm tắt tiền khám 30 giây kết nối đặt lịch tại **2.144 cơ sở y tế** (1.505 tại TP.HCM và 639 tại Hà Nội). Toàn bộ ứng dụng chạy trên **Amazon EC2 + EBS**, **Amazon RDS PostgreSQL**, bảo vệ biên bởi **AWS WAF**, **CloudFront** và **ALB**.  

*Lợi ích và hoàn vốn đầu tư (ROI)*  
Giải pháp giúp rút ngắn thời gian khai thác bệnh sử ban đầu xuống còn 30 giây, giảm sai sót dùng thuốc tại nhà và ngăn ngừa biến cố dị ứng/tương tác thuốc nhờ cơ chế xác nhận *Human-in-the-Loop*. Nhờ lưu trữ chỉ mục ChromaDB trực tiếp trên **Amazon EBS** thay cho các dịch vụ Managed Vector DB đắt đỏ và tận dụng các mô hình `:free` trên OpenRouter, chi phí vận hành giai đoạn thử nghiệm (Dev/PoC) chỉ ở mức **16,26 USD/tháng** (195,12 USD/12 tháng) và đạt **0,00418 USD/phiên tư vấn** (~105 VNĐ/lượt) ở quy mô Production 10.000 người dùng/tháng, tiết kiệm hơn 95% chi phí triển khai AI y tế ban đầu.  

### 3. Kiến trúc giải pháp  
Nền tảng áp dụng kiến trúc **AWS 3-Tier VPC (`10.0.0.0/16`)** tại khu vực `ap-southeast-1` nhằm cô lập hoàn toàn tầng tính toán và cơ sở dữ liệu khỏi truy cập Internet trực tiếp. Lưu lượng người dùng đi qua **Route 53**, **AWS WAF** và **Amazon CloudFront** tới **Application Load Balancer (ALB)** tại `Public Subnet`, sau đó định tuyến nội bộ qua cổng `8000` vào container trên **Amazon EC2** nằm trong `Private App Subnet`. Dữ liệu Vector **ChromaDB (`./medical_vectordb`)** và tệp tải lên (`./uploads`) được lưu bền vững trên ổ đĩa **Amazon EBS (`gp3`)** kèm sao lưu sang **Amazon S3**, trong khi 5 bảng dữ liệu giao dịch lưu tại **Amazon RDS PostgreSQL** (`SQL :5432`) trong `Private Database Subnet`. Mọi kết nối chiều ra tới **OpenRouter / Gemini AI**, **Google OAuth 2.0** và **Gmail SMTP / SpeedSMS** đều đi qua **NAT Gateway**.  

<img width="2560" height="1674" alt="architecture" src="https://github.com/user-attachments/assets/ab55faef-924c-40ff-8622-f7a663c190ec" />


*Dịch vụ AWS sử dụng*  
- *Amazon Route 53 & Amazon CloudFront*: Phân giải DNS, lưu đệm tài nguyên tĩnh (1 TB miễn phí/tháng) và ẩn địa chỉ gốc máy chủ.  
- *AWS WAF & AWS Certificate Manager (ACM)*: Tường lửa lớp 7 chống OWASP Top 10, Rate Limiting và cấp phát chứng chỉ SSL/TLS 1.3 miễn phí.  
- *Application Load Balancer (ALB) & NAT Gateway*: Thực hiện TLS Termination tại `Public Subnet` và cấp quyền truy cập chiều ra an toàn cho `Private App Subnet`.  
- *Amazon EC2 & Amazon EBS (`gp3`)*: Máy chủ chạy container ứng dụng (`PORT=8000`) gắn ổ đĩa khối lưu chỉ mục ChromaDB (364 tài liệu) và thư mục `/uploads`.  
- *Amazon RDS for PostgreSQL*: Cơ sở dữ liệu quan hệ cô lập trong `Private Database Subnet` (`SQL :5432`) quản lý người dùng, hồ sơ y tế và lịch hẹn.  
- *Amazon S3 & Amazon ECR*: Lưu trữ sao lưu (`Backup / Archive`) hồ sơ, snapshot RAG và quản lý Docker Image riêng tư.  
- *AWS Secrets Manager, CloudWatch & AWS IAM*: Quản lý khóa API tập trung, thu thập log đã ẩn danh PII và phân quyền tối thiểu qua `EC2 IAM Role`.  

*Thiết kế thành phần*  
- *Tầng Biên & Bảo mật*: Route 53, WAF, CloudFront và ALB chặn tấn công mạng và mã hóa toàn bộ lưu lượng HTTPS.  
- *Tầng Ứng dụng & AI (`Private App Subnet`)*: Backend Python (`app.py` / `core/main_fastapi.py`) xử lý Double Guardrails, Hybrid RAG (BM25 + ChromaDB + BGE Reranker), OCR 8 bước và Circuit Breaker dự phòng.  
- *Tầng Dữ liệu & Lưu trữ (`Private Database Subnet` & Storage)*: RDS PostgreSQL lưu 5 bảng nghiệp vụ (`users`, `medical_documents`, `clinical_consultations`, `intake_sessions`, `medical_appointments`); EBS lưu dữ liệu Vector tốc độ cao; S3 lưu bản sao lưu dài hạn.  
- *Tích hợp Dịch vụ Ngoài*: Xác thực qua Google OAuth 2.0, gửi OTP 6 số qua Gmail SMTP / SpeedSMS, suy luận AI đa khóa qua OpenRouter (`:free`) và hiển thị bản đồ 2.144 cơ sở y tế qua Google Maps Platform API.  

### 4. Triển khai kỹ thuật  
*Các giai đoạn triển khai*  
Dự án được xây dựng đồng bộ từ lõi lâm sàng đến hạ tầng AWS qua 4 giai đoạn chính:  
1. *Khảo sát & Thiết kế Kiến trúc*: Thu thập dữ liệu y khoa chuẩn hóa, thiết kế sơ đồ AWS 3-Subnet VPC, lược đồ 5 bảng PostgreSQL và đặc tả API (Tuần 1–2).  
2. *Phát triển Lõi Lâm sàng & RAG*: Xây dựng Double Guardrails chống bẫy phủ định, chỉ mục Hybrid RAG 364 tài liệu và Pipeline OCR đối soát 8 bước (Tuần 3–6).  
3. *Mở rộng Nghiệp vụ & Giao diện SPA*: Tích hợp 2.462 công thức dinh dưỡng ViFoodRec, danh mục 2.144 cơ sở y tế, đặt lịch khám, xác thực OAuth/OTP và giao diện Web SPA (Tuần 7–8).  
4. *Đóng gói Container, Triển khai AWS & Kiểm định*: Đóng gói Docker lên Amazon ECR, cấu hình EC2 + EBS + RDS + ALB + WAF + Secrets Manager, chạy 78 bộ kiểm thử tự động và đánh giá benchmark (Tuần 9–12).  

*Yêu cầu kỹ thuật*  
- *Backend & AI Pipeline*: Python (`Flask` / `FastAPI` gateway, `SQLAlchemy`), `ChromaDB`, `rank-bm25`, `BAAI/bge-reranker-v2-m3`, `EasyOCR`, tích hợp đa khóa OpenRouter (`OPENROUTER_CHAT_API_KEY`, `OPENROUTER_OCR_API_KEY`, `OPENROUTER_NUTRITION_API_KEY`) kèm Circuit Breaker và Retry Policy.  
- *An toàn Tệp & Dữ liệu*: Kiểm tra Magic Bytes chống giả mạo định dạng, chặn Decompression/Zip Bomb (ảnh $\le 25\text{ MP}$, PDF $\le 30\text{ trang}$, tỷ lệ nén DOCX $< 50\text{x}$), bảo vệ CSRF và ẩn danh PII tự động trong log.  
- *Hạ tầng AWS*: VPC `10.0.0.0/16` tại `ap-southeast-1`, Security Groups phân tách chặt chẽ giữa ALB (`443`), EC2 (`8000`) và RDS (`5432`), nạp biến môi trường động từ AWS Secrets Manager thông qua EC2 IAM Role.  

### 5. Lộ trình & Mốc triển khai  
- *Trước thực tập (Tháng 0)*: Khảo sát bài toán quá tải y tế tuyến đầu, thu thập 364 tài liệu phác đồ (*Bộ Y Tế, MedlinePlus NIH*), 2.462 công thức món ăn Việt (*ViFoodRec*) và danh mục 2.144 cơ sở y tế.  
- *Thực tập (Tháng 1–3 / 12 Tuần)*:  
  - Tháng 1 (Tuần 1–4): Thiết kế kiến trúc AWS 3-Subnet VPC, phát triển lõi Clinical Triage, Double Guardrails và cơ sở dữ liệu.  
  - Tháng 2 (Tuần 5–8): Hoàn thiện Hybrid RAG, Pipeline OCR 8 bước, Clinical Nutrition Studio, đặt lịch khám và giao diện Web SPA.  
  - Tháng 3 (Tuần 9–12): Đóng gói Docker lên ECR, triển khai trên EC2 + EBS + RDS + ALB + WAF, chạy 78 tệp kiểm thử tự động (`pytest`) và chuẩn bị Live Demo Workshop.  
- *Sau triển khai*: Mở rộng tích hợp chuẩn liên thông dữ liệu y tế HL7 FHIR và triển khai thử nghiệm tại các phòng khám đa khoa vệ tinh.  

### 6. Ước tính ngân sách  
Chi phí được tính toán trên **AWS Pricing Calculator** tại khu vực **Singapore (`ap-southeast-1`)** cho hai giai đoạn: **Dev/PoC Workshop** (tận dụng Free Tier và bật ALB/NAT theo giờ demo) và **Production** (10.000 người dùng/tháng chạy 24/7).  

*Chi phí hạ tầng (Giai đoạn Dev / PoC Workshop)*  
- Amazon EC2 (`t3.micro` / `t3.small`, cổng 8000): 0,00 – 3,17 USD/tháng (miễn phí 750 giờ `t3.micro` Free Tier).  
- Amazon EBS (`gp3` 20 GB lưu ChromaDB & `/uploads`): 0,00 USD/tháng (nằm trong 30 GB Free Tier).  
- Amazon RDS PostgreSQL (`db.t4g.micro`, cổng 5432): 0,00 USD/tháng (miễn phí 750 giờ + 20 GB Free Tier).  
- Application Load Balancer & NAT Gateway (~40 giờ kiểm thử/demo): 3,37 USD/tháng.  
- Amazon S3 (`Backup / Archive` ~2 GB) & Amazon ECR (~450 MB): 0,00 USD/tháng (trong hạn mức Free Tier).  
- AWS Secrets Manager (3 nhóm Secrets): 1,20 USD/tháng.  
- Amazon CloudFront (15 GB), ACM (TLS 1.3), CloudWatch & AWS IAM: 0,00 USD/tháng (Always-Free / Free Tier).  
- Amazon Route 53 (1 Hosted Zone + 50.000 queries): 0,52 USD/tháng.  
- AWS WAF (1 Web ACL + 3 Managed Rules): 8,00 USD/tháng.  
- Dịch vụ AI & Xác thực ngoài (OpenRouter `:free`, Gmail SMTP, Google OAuth): 0,00 USD/tháng.  

*Tổng*: **16,26 USD/tháng** (~410.000 VNĐ/tháng), **195,12 USD/12 tháng**  
- *Chi phí mở rộng Production (10.000 người dùng / 25.000 phiên tư vấn / 8.000 lượt OCR mỗi tháng, chạy 24/7)*: **104,60 USD/tháng** (tương đương **0,00418 USD/phiên**).  

### 7. Đánh giá rủi ro  
*Ma trận rủi ro*  
- Bỏ sót dấu hiệu cấp cứu hoặc mắc bẫy phủ định (*"không khó thở"*): Ảnh hưởng cao, xác suất trung bình.  
- Sai lệch nhận diện OCR trên đơn thuốc viết tay mờ/lóa: Ảnh hưởng cao, xác suất trung bình.  
- Tải lên tệp độc hại (Decompression/Zip Bomb) hoặc truy cập trái phép hồ sơ (IDOR): Ảnh hưởng cao, xác suất thấp.  
- Gián đoạn API từ nhà cung cấp AI bên ngoài (OpenRouter/Gemini) hoặc vượt ngân sách: Ảnh hưởng trung bình, xác suất thấp.  

*Chiến lược giảm thiểu*  
- Lâm sàng & Phủ định: Áp dụng Double Guardrails (`core/triage.py`) đạt 100% Emergency Recall và 100% Negation Accuracy trên 100 ca kiểm định, chặn tuyệt đối việc tự kê đơn kháng sinh.  
- OCR Đơn thuốc: Chạy song song EasyOCR & Vision LLM, tự động gắn cờ `needs_review=True` khi lệch trường và bắt buộc người dùng xác nhận (*Human-in-the-Loop*) trước khi tạo lịch uống thuốc.  
- Bảo mật & Hạ tầng: Cô lập EC2 và RDS trong Private Subnets, kiểm tra Magic Bytes & giới hạn kích thước tệp (`core/storage_service.py`), ẩn danh PII trong log và cài đặt cảnh báo **AWS Budgets** ở ngưỡng 20 USD.  

*Kế hoạch dự phòng*  
- Kích hoạt **Circuit Breaker** (`core/circuit_breaker.py`) tự động ngắt mạch sau 5 lỗi API liên tiếp để chuyển sang **Fallback Clinical Engine** và bộ lọc dinh dưỡng tất định mà không làm treo hệ thống.  
- Khôi phục nhanh chỉ mục ChromaDB và cơ sở dữ liệu từ bản sao lưu định kỳ trên **Amazon S3** và Automated Snapshots của **Amazon RDS**.  

### 8. Kết quả kỳ vọng  
*Cải tiến kỹ thuật*  
Hoàn thiện nền tảng trợ lý y tế đa phương thức vượt qua **78 tệp kiểm thử tự động** và bộ 4 benchmark (308 mẫu kiểm định), đạt **100% Emergency Recall**, **100% Negation Accuracy**, **0,0% tỷ lệ lời khuyên nguy hiểm**, **Holdout Recall@5 $\ge$ 85%** và thời gian phản hồi trung bình **$< 2,5\text{ giây}$**.  

*Giá trị dài hạn*  
Cung cấp kiến trúc tham chiếu chuẩn AWS bảo mật cao và chi phí thấp (**~105 VNĐ/phiên tư vấn** ở quy mô 10.000 người dùng), có thể tái sử dụng và mở rộng cho các phòng khám, bệnh viện tuyến cơ sở nhằm giảm tải khâu sàng lọc tiếp đón và nâng cao an toàn sử dụng thuốc cho cộng đồng.
