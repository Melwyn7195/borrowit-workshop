---
title: "Báo cáo thực tập"
date: 2026-07-28
weight: 1
chapter: false
---

# Báo cáo thực tập

### Thông tin sinh viên

<!-- TODO: điền đầy đủ các mục bên dưới. -->

&emsp; **Họ và tên:** [họ tên của bạn]

&emsp; **Số điện thoại:** [số điện thoại của bạn]

&emsp; **Email:** duc097872917@gmail.com

&emsp; **Trường:** Trường Đại học Bách khoa — ĐHQG TP.HCM

&emsp; **Khoa:** Khoa học và Kỹ thuật Máy tính

&emsp; **Ngành:** Khoa học Máy tính

&emsp; **Lớp:** [mã lớp FCAJ của bạn, ví dụ AWS062026]

&emsp; **Công ty thực tập:** Công ty TNHH Dịch vụ Web Amazon Việt Nam

&emsp; **Vị trí thực tập:** Workforce Bootcamp — First Cloud AI Journey

&emsp; **Thời gian thực tập:** Từ ngày 15/06/2026 đến ngày 15/09/2026

&emsp; **Đề tài đăng ký:** Đề tài 3 — Application Development on AWS

![Ảnh đại diện](/images/avatar.png)

<!-- TODO: thay static/images/avatar.png bằng ảnh của bạn. -->

### Tóm tắt dự án

**BorrowIt** là một nền tảng cho thuê đồ giữa người dùng với nhau — gồm frontend
React, API Express và cơ sở dữ liệu PostgreSQL, ban đầu được host trên Supabase.
Trong kỳ thực tập này, toàn bộ hệ thống đã được chuyển lên AWS, định nghĩa hoàn
toàn bằng infrastructure-as-code với AWS CDK: Amazon RDS cho cơ sở dữ liệu,
AWS Fargate phía sau Application Load Balancer cho API, và Amazon S3 kết hợp
CloudFront cho web client cùng ảnh người dùng tải lên.

Kế hoạch và thiết kế dẫn tới kiến trúc đó được trình bày ở
[phần 2](2-Proposal/), còn toàn bộ quá trình migration được ghi lại thành một
workshop có thể tái lập ở [phần 4](4-Workshop/), viết sao cho người đọc bắt đầu
từ một tài khoản AWS trống vẫn dựng lại được toàn bộ môi trường.

### Nội dung báo cáo

1. [Nhật ký công việc](1-Worklog/)
2. [Đề xuất giải pháp](2-Proposal/)
3. [Blogs](3-Blogs/)
4. [Workshop](4-Workshop/)
5. [Tự đánh giá](5-Self-evaluation/)
6. [Chia sẻ và góp ý](6-Feedback/)
