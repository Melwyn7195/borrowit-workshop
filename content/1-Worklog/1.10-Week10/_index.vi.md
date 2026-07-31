---
title: "Nhật ký tuần 10"
date: 2026-07-28
weight: 10
chapter: false
pre: " <b> 1.10. </b> "
---

{{% notice warning %}}
**Hạn chót cứng trong tuần này.** Hồ sơ công ty (biểu mẫu D2–D5) phải nộp về Khoa
trước **16:00 ngày 21/08/2026**. Hãy coi phần kỹ thuật là phải hoàn tất trước mốc đó.
{{% /notice %}}

### Mục tiêu tuần 10

* Trình diễn tính sẵn sàng cao: failover Multi-AZ của RDS và autoscaling của Fargate.
* Đo lường hành vi dưới tải thay vì chỉ khẳng định.
* Nộp hồ sơ công ty.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - Bật `-c multiAz=true -c scaling=true` <br> - Hiểu bản sao dự phòng đồng bộ bảo vệ được gì và không bảo vệ được gì | 17/08/2026 | | |
| 3 | - Ép RDS failover; ghi lại API trả lỗi trong bao nhiêu giây và ECS có khởi động lại task hay không | 18/08/2026 | | |
| 4 | - Chạy kiểm thử tải; ghi lại số task tăng 1 → 2 → 3 và độ trễ hồi phục | 19/08/2026 | | |
| 5 | - Tắt cả hai cờ và xác nhận Multi-AZ hiển thị **No** trở lại <br> - Kiểm tra hoá đơn đã giảm về mức cũ | 20/08/2026 | | |
| 6 | - **Nộp biểu mẫu D2–D5 về Khoa trước 16:00** | 21/08/2026 | | |

### Kết quả dự kiến tuần 10

* Một lần failover có đo đạc: số liệu thật về khoảng thời gian gián đoạn, không phải ước lượng.
* Bằng chứng autoscaling phản ứng theo CPU, ghi nhận trên dashboard.
* Cả hai cờ làm tăng chi phí đã được tắt, kiểm chứng trên console.
* Hồ sơ nộp đúng hạn.

<!-- TODO(prose): phép đo failover chính là lúc thiết kế health check ở tuần 7 được
     kiểm chứng. Nếu liveness check không khởi động lại task trong lúc failover thì
     đó là thiết kế đã phát huy tác dụng, và điều đó đáng nói rõ. Nếu có khởi động
     lại thì cũng nói ra và giải thích bạn sẽ thay đổi điều gì. -->

{{% notice warning %}}
Multi-AZ nhân đôi chi phí RDS và autoscaling có thể nhân ba chi phí Fargate. Đừng
để bất kỳ cờ nào bật sau khi trình diễn xong.
{{% /notice %}}
