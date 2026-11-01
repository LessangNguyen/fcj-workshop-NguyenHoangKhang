---
title: "Worklog Tuần 5"
date: 2026-10-12
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---
{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}


### Mục tiêu tuần 5:

- Tìm hiểu và thực hành tự động hóa triển khai hạ tầng dưới dạng mã với **AWS CloudFormation**.
- Thiết lập quản lý danh tính tập trung và Đăng nhập một lần với **AWS IAM Identity Center (SSO)** cho tổ chức.
- Tăng cường bảo mật và kiểm soát quyền hạn IAM thông qua **Permission Boundaries** (Ranh giới quyền).
- Kiểm soát chặt chẽ việc chuyển giao vai trò IAM bằng các điều kiện nâng cao (Role Assumption Conditions).
- Giám sát, đánh giá và kiểm tra các tiêu chuẩn tuân thủ bảo mật hệ thống bằng **AWS Security Hub**.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Tìm hiểu khái niệm và thực hành khởi tạo cơ sở hạ tầng dưới dạng mã (Infrastructure as Code) sử dụng **AWS CloudFormation** (viết template, tạo stack, quản lý vòng đời tài nguyên). | 12/10/2026   | 12/10/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 3   | - Thiết lập quản lý truy cập và Đăng nhập một lần (**AWS IAM Identity Center / AWS SSO**) cho Tổ chức/Nhiều tài khoản. | 13/10/2026   | 13/10/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 4   | - Tìm hiểu và thực hành giới hạn quyền tối đa của người dùng/vai trò IAM bằng **Permission Boundaries** (Ranh giới quyền). | 14/10/2026   | 14/10/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 5   | - Cấu hình và hạn chế việc chuyển giao vai trò (AssumeRole) dựa trên các điều kiện ngữ cảnh bảo mật (`aws:PrincipalTag`, điều kiện nguồn, v.v.). | 15/10/2026   | 15/10/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 6   | - Khám phá và kiểm tra các tiêu chuẩn tuân thủ bảo mật, quét lỗ hổng cấu hình hệ thống với **AWS Security Hub**. | 16/10/2026   | 16/10/2026      | <https://cloudjourney.awsstudygroup.com/> |


### Kết quả đạt được trong tuần 5:

- Nắm vững cách viết template và tự động hóa toàn bộ quy trình khởi tạo, cập nhật, xóa tài nguyên hạ tầng bằng **AWS CloudFormation**.
- Cấu hình thành công **AWS IAM Identity Center (SSO)** để quản lý danh tính tập trung và phân quyền truy cập an toàn cho người dùng trong môi trường AWS đa tài khoản.
- Áp dụng thành công **IAM Permission Boundaries** để thiết lập ranh giới quyền lực tối đa, ngăn chặn việc leo thang đặc quyền khi phân quyền cho developer hoặc người dùng khác.
- Thiết lập các chính sách kiểm soát chuyển giao vai trò khắt khe dựa trên **điều kiện nâng cao (Conditions)**, tăng cường tính bảo mật cho mô hình giả định quyền (AssumeRole).
- Triển khai và làm quen với **AWS Security Hub**, tổng hợp các cảnh báo bảo mật và kiểm tra mức độ tuân thủ theo các tiêu chuẩn bảo mật ngành.
- Dọn dẹp tài nguyên sau các bài thực hành để đảm bảo an toàn và tối ưu chi phí.
