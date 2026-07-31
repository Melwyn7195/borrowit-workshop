---
title: "Nhật ký tuần 9"
date: 2026-07-28
weight: 9
chapter: false
pre: " <b> 1.9. </b> "
---

{{% notice note %}}
**Dự kiến.** Hãy thay bằng những gì thực sự đã làm và điền ngày hoàn thành.
{{% /notice %}}

### Mục tiêu tuần 9

* Bổ sung CloudWatch dashboard và các alarm.
* Đối chiếu hoá đơn thực tế với bảng ước tính ở tuần 3.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - CloudWatch: metric, statistic, period, trạng thái alarm, `treatMissingData` | 10/08/2026 | | |
| 3 | - Thêm sáu alarm; đặt `BREACHING` cho unhealthy target để một task biến mất được hiểu là sự cố chứ không phải im lặng | 11/08/2026 | | |
| 4 | - Tạo SNS topic và subscription email; dựng dashboard năm widget | 12/08/2026 | | |
| 5 | - Chủ động kích hoạt một alarm (`-c desiredCount=0`) và xác nhận nhận được thông báo | 13/08/2026 | | |
| 6 | - Cost Explorer lọc theo tag `Project=BorrowIt`; so sánh thực tế với ước tính | 14/08/2026 | | |

### Kết quả dự kiến tuần 9

* Sáu alarm bao phủ load balancer, service và database, gửi thông báo qua SNS.
* Dashboard hiển thị request, lỗi, độ trễ p50/p95, tình trạng task và database.
* Một alarm đã được chứng minh là kêu thật, không chỉ được cấu hình.
* Chi phí thực tế hằng tháng đối chiếu với ước tính tuần 3, theo từng dịch vụ.

<!-- TODO(prose): `treatMissingData` là chi tiết đáng hiểu cho thấu đáo. Một task
     đã biến mất thì không gửi dữ liệu nào cả, chứ không phải gửi giá trị 0 — nên
     với thiết lập mặc định, dịch vụ có thể sập hoàn toàn mà alarm vẫn xanh. Hãy
     giải thích vì sao "không có dữ liệu" lại mang ý nghĩa ngược lại với alarm 5xx. -->
