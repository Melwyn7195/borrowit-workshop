---
title: "Nhật ký tuần 3"
date: 2026-07-28
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Mục tiêu tuần 3

* Rà soát xem BorrowIt thực sự phụ thuộc vào những gì trong Supabase.
* Chọn nền tảng compute và lập luận cho lựa chọn đó.
* Vẽ sơ đồ kiến trúc đích và lập bảng ước tính chi phí để làm căn cứ.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - Đọc mã nguồn BorrowIt: route Express, controller, model, `db/index.js` <br> - Liệt kê mọi phụ thuộc riêng của Supabase | 29/06/2026 | 29/06/2026 | |
| 3 | - Ánh xạ từng phụ thuộc sang dịch vụ AWS: Postgres → RDS, lưu trữ → S3, xác thực → giữ nguyên phần JWT sẵn có | 30/06/2026 | 30/06/2026 | |
| 4 | - So sánh các lựa chọn compute: EC2, ECS trên EC2, **ECS Fargate**, Lambda <br> - Chốt Fargate và ghi lại lý do | 01/07/2026 | 01/07/2026 | |
| 5 | - Vẽ kiến trúc đích <br> - Quyết định chia bốn stack theo chi phí và vòng đời | 02/07/2026 | 02/07/2026 | |
| 6 | - Lập bảng ước tính chi phí trên AWS Pricing Calculator | 03/07/2026 | 03/07/2026 | <https://calculator.aws/> |

### Kết quả tuần 3

<!-- TODO(prose): phần so sánh compute đáng viết vài câu bằng lời của bạn. Lambda
     rẻ hơn khi nhàn rỗi nhưng ứng dụng là một server Express chạy dài kèm
     connection pool; EC2 rẻ hơn khi tải ổn định nhưng bạn phải tự vá lỗi. Fargate
     là lựa chọn cuối cùng — hãy nói bạn đã đánh đổi điều gì để có nó. -->

* Lập được danh mục đầy đủ các phụ thuộc Supabase kèm dịch vụ AWS tương ứng.
* Chọn **ECS Fargate** làm nền tảng compute: không phải vá máy chủ, và ứng dụng
  Express chạy nguyên trạng trong container.
* Thiết kế cách chia bốn stack — `Foundation`, `Data`, `Frontend`, `App` — phân
  tách theo **mức chi phí của từng phần**, để tầng tốn tiền có thể xoá riêng.
* Hoàn thành kiến trúc kèm chi phí: ~$50/tháng cho toàn bộ hệ thống, đối chiếu với
  số credit cố định.

![Bản phác kiến trúc đầu tiên](/images/1-Worklog/1.3-Week3/architecture-draft.png?width=100pc)
