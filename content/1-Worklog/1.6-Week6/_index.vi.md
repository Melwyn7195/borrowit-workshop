---
title: "Nhật ký tuần 6"
date: 2026-07-28
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Mục tiêu tuần 6

* Đóng gói API Express thành container.
* Push một image hoạt động được lên ECR.
* Chuyển phần upload file từ bucket cũ sang S3 bucket thuộc quyền dự án.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - Kiến thức Docker: layer, build cache, multi-stage build <br> - Viết `Dockerfile` đầu tiên cho API | 20/07/2026 | 20/07/2026 | |
| 3 | - Gỡ lỗi build `bcrypt` trên Alpine; đổi base image sang `node:22-bookworm-slim` <br> - Tách dependency thành stage riêng để tái dùng cache | 21/07/2026 | 21/07/2026 | |
| 4 | - Thêm `USER node`, `HEALTHCHECK` và `.dockerignore` <br> - Kiểm thử bằng `docker run` ở local kết nối tới RDS | 22/07/2026 | 22/07/2026 | |
| 5 | - Xác thực Docker với ECR và push image có tag đầu tiên <br> - Xem kết quả quét `imageScanOnPush` | 23/07/2026 | 23/07/2026 | |
| 6 | - Sửa `uploadController` để ghi lên S3 qua `multer-s3` bằng task role, không dùng access key tĩnh | 24/07/2026 | 24/07/2026 | |

### Kết quả tuần 6

<!-- TODO(prose): vấn đề Alpine/bcrypt là một câu chuyện cụ thể rất tốt nếu đó đúng
     là điều đã xảy ra — musl với glibc, binary dựng sẵn, và lựa chọn chấp nhận
     image Debian lớn hơn một chút thay vì biên dịch từ mã nguồn. Hãy kiểm tra lại
     ghi chú của bạn trước khi viết ra như một sự thật. -->

* Tạo được image container hoạt động cho API Express, chạy bằng user không đặc quyền.
* Chọn base Debian thay vì Alpine vì một lý do cụ thể và có ghi chép, không phải
  theo mặc định.
* Push image có tag lên ECR và xem xét kết quả quét lỗ hổng.
* Thay thế access key tĩnh trong luồng upload bằng task role của Fargate.

