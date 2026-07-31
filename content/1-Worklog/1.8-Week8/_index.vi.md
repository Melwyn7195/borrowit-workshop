---
title: "Nhật ký tuần 8"
date: 2026-07-28
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

{{% notice note %}}
**Dự kiến.** Tuần này chưa diễn ra — nội dung bên dưới là kế hoạch. Hãy thay bằng
những gì thực sự đã làm và điền ngày hoàn thành.
{{% /notice %}}

### Mục tiêu tuần 8

* Triển khai `BorrowitFrontend` — các S3 bucket và một CloudFront distribution.
* Phục vụ bản build React và ảnh người dùng mà không để bucket nào ở chế độ public.
* Xử lý, hoặc ghi nhận rõ, vấn đề mixed content giữa HTTP và HTTPS.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - Khái niệm CloudFront: distribution, behaviour, cache policy, price class <br> - So sánh Origin Access Control với Origin Access Identity cũ | 03/08/2026 | | |
| 3 | - Viết `FrontendStack`: hai bucket private, một distribution, behaviour `/uploads/*` trỏ tới origin thứ hai | 04/08/2026 | | |
| 4 | - Xử lý routing cho SPA: ánh xạ 403 và 404 về `/index.html` với mã 200 để React Router phân giải deep link | 05/08/2026 | | |
| 5 | - Build ứng dụng React với `VITE_API_URL`, sync lên S3, invalidate cache | 06/08/2026 | | |
| 6 | - Khảo sát lỗi mixed content; cân nhắc dùng ACM trên ALB so với đưa API qua CloudFront | 07/08/2026 | | |

### Kết quả dự kiến tuần 8

* Cả hai bucket đều private, chỉ đọc được qua CloudFront nhờ Origin Access Control.
* Một distribution duy nhất phục vụ web app và ảnh người dùng từ hai origin riêng.
* Ghi nhận quyết định về HTTPS cho API, kèm lập luận.

<!-- TODO(prose): mixed content là vấn đề còn tồn đọng thật sự của dự án ở thời
     điểm hiện tại — CloudFront chạy HTTPS, ALB chạy HTTP, và trình duyệt chặn lời
     gọi. Dù bạn chọn cách nào (mua tên miền rồi dùng ACM, đưa API vào chung
     distribution, hay demo frontend ở local), hãy ghi lại quyết định và lý do ở
     đây thay vì để lỗ hổng không được giải thích. -->
