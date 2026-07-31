---
title: "Nhật ký tuần 1"
date: 2026-07-28
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Mục tiêu tuần 1

* Làm quen với đội ngũ First Cloud AI Journey và nắm cấu trúc chương trình.
* Xây dựng hình dung tổng thể về AWS: region, availability zone, mô hình trách
  nhiệm chung, và cách tính phí.
* Tạo được tài khoản AWS và cấu hình CLI hoạt động.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - Giới thiệu chương trình FCAJ và mentor <br> - Đọc quy định thực tập, quy cách báo cáo và điều kiện đóng mộc | 15/06/2026 | 15/06/2026 | |
| 3 | - Hạ tầng toàn cầu của AWS: region, AZ, edge location <br> - Vì sao chọn `ap-southeast-1` cho dự án tại Việt Nam | 16/06/2026 | 16/06/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 4 | - Tạo tài khoản AWS <br> - **Thực hành:** bật MFA cho root user, tạo IAM user quản trị, đặt budget alarm $10 | 17/06/2026 | 17/06/2026 | |
| 5 | - Cài và cấu hình AWS CLI v2 <br> - **Thực hành:** `aws configure`, `aws sts get-caller-identity`, `aws freetier get-account-plan-state` | 18/06/2026 | 18/06/2026 | |
| 6 | - Tìm hiểu mô hình tính phí của tài khoản: credit Free Plan, không còn free tier 12 tháng sau 07/2025 <br> - Ước tính ngân sách ban đầu cho dự án | 19/06/2026 | 19/06/2026 | <https://calculator.aws/> |

### Kết quả tuần 1

<!-- TODO(prose): viết lại bằng lời của bạn. Gợi ý:
       - Điều gì khiến bạn bất ngờ về mô hình tính phí. Việc không còn free tier
         12 tháng đã thay đổi toàn bộ cách chọn kích thước tài nguyên sau này, nên
         đáng ghi nhận rằng bạn phát hiện điều đó từ tuần 1 chứ không phải tuần 8.
       - Budget alarm đã thực sự được cấu hình và kiểm thử hay chưa.
       - Những gì không chạy đúng ngay lần đầu. -->

* Tạo và bảo vệ tài khoản AWS: bật MFA cho root user, tách riêng IAM user quản trị
  để dùng hằng ngày, và đặt budget alarm trước khi triển khai bất cứ thứ gì.
* Xác nhận tài khoản chạy trên **credit AWS Free Plan** thay vì free tier 12 tháng
  cũ, điều này định hình ràng buộc chi phí cho toàn bộ phần còn lại của dự án.
* Cài đặt và cấu hình AWS CLI, kiểm chứng quyền truy cập bằng
  `aws sts get-caller-identity`.
* Chọn `ap-southeast-1` (Singapore) làm region của dự án — region gần người dùng
  nhất, và là nơi mọi tài nguyên về sau đều được cố định vào.
