---
title: "Nhật ký tuần 5"
date: 2026-07-28
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Mục tiêu tuần 5

* Triển khai `BorrowitData` — RDS PostgreSQL 16 trong isolated subnet.
* Loại bỏ vĩnh viễn mật khẩu database khỏi các file `.env`.
* Chuyển schema và dữ liệu ra khỏi Supabase.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - Chọn cấu hình RDS: instance class, dung lượng, Multi-AZ, thời gian giữ backup <br> - Chốt `db.t4g.micro` 20 GB và lập luận theo số credit | 13/07/2026 | 13/07/2026 | |
| 3 | - Viết `DataStack`; ghim `maxAllocatedStorage` bằng `allocatedStorage` để autoscaling không làm tăng hoá đơn <br> - Deploy (~10 phút) | 14/07/2026 | 14/07/2026 | |
| 4 | - Secrets Manager: `rds.Credentials.fromGeneratedSecret` <br> - Kiểm chứng mật khẩu không xuất hiện trong template đã synth | 15/07/2026 | 15/07/2026 | |
| 5 | - Xuất schema từ Supabase bằng `pg_dump --schema-only --no-owner --no-privileges` <br> - Xử lý các lệnh cấp quyền riêng của Supabase bị lỗi trên RDS | 16/07/2026 | 16/07/2026 | |
| 6 | - Chỉnh `db/index.js` để ghép `DATABASE_URL` từ các biến `DB_*` được inject <br> - Kiểm thử kết nối ở máy local | 17/07/2026 | 17/07/2026 | |

### Kết quả tuần 5

<!-- TODO(prose): việc export từ Supabase gần như chắc chắn không suôn sẻ ngay lần
     đầu — role grant, extension, hoặc policy RLS mà RDS từ chối. Gặp vấn đề gì thì
     ghi lại vấn đề đó. Một báo cáo migration không có chút trở ngại nào đọc như
     thể quá trình migration chưa từng thực sự diễn ra. -->

* Triển khai PostgreSQL 16 vào isolated subnet, không có endpoint công khai.
* Loại bỏ mật khẩu database khỏi mã nguồn: RDS sinh mật khẩu vào Secrets Manager,
  ECS inject vào task lúc khởi động.
* Tắt storage autoscaling để hoá đơn không thể âm thầm tăng trên số credit cố định.
* Chuyển schema ra khỏi Supabase, xử lý các điểm không tương thích phát sinh.

