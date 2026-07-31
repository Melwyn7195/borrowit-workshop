---
title: "Nhật ký tuần 7"
date: 2026-07-28
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Mục tiêu tuần 7

* Triển khai `BorrowitApp` — ECS cluster, Fargate service, Application Load Balancer.
* Đưa API lên phục vụ request qua internet.
* Nạp schema vào RDS mà không cần cấp endpoint công khai cho database.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - Khái niệm ECS: cluster, task definition, service, target group <br> - Viết task definition kèm inject từ Secrets Manager | 27/07/2026 | 27/07/2026 | |
| 3 | - Tách liveness (`/health/live`, không chạm DB) khỏi readiness (`/health`, chạy `SELECT 1`) <br> - Thêm ALB, listener và target group | 28/07/2026 | | |
| 4 | - Thêm `circuitBreaker` để tự rollback và `minHealthyPercent: 50` để mỗi lần deploy không chạy hai task <br> - Lần `cdk deploy BorrowitApp` thành công đầu tiên | 29/07/2026 | | |
| 5 | - Bật `enableExecuteCommand`, cài Session Manager plugin <br> - Nạp schema và dữ liệu mẫu từ bên trong task đang chạy | 30/07/2026 | | |
| 6 | - Kiểm chứng API từ đầu tới cuối; xác nhận public IP của task không truy cập được qua cổng 3456 <br> - Tinh chỉnh `deregistrationDelay` theo thời gian drain của ứng dụng | 31/07/2026 | | |

### Kết quả tuần 7

<!-- TODO(prose): tuần này là phần cốt lõi của cả dự án — hãy viết cho kỹ. Việc
     tách hai health check là quyết định thiết kế đáng bảo vệ nhất trong báo cáo:
     một liveness check phụ thuộc database sẽ biến một trục trặc ngắn thành vòng
     lặp khởi động lại vô tận. Hãy giải thích bằng lời của bạn.

     Đồng thời ghi lại cách bạn thực sự đưa được schema vào một database bị cách
     ly. Các lựa chọn khác là bastion host (~$8/tháng, thêm một máy phải vá lỗi)
     hoặc mở public cho RDS (phá vỡ thiết kế). ECS Exec không tốn gì và không để
     lại thứ gì. -->

* Đưa API Express chạy trên Fargate phía sau ALB, truy cập được qua internet.
* Thiết kế liveness và readiness để trả lời hai câu hỏi thực sự khác nhau, nhờ đó
  sự cố database làm dịch vụ suy giảm chứ không gây khởi động lại.
* Nạp schema vào database bị cách ly bằng `aws ecs execute-command`, không cần
  bastion host và không cần endpoint công khai.
* Chứng minh public IP của task không truy cập được qua cổng 3456 từ bên ngoài —
  chỉ load balancer mới tới được.

