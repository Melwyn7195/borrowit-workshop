---
title : "Giám sát và vận hành"
date : 2026-07-28
weight : 8
chapter : false
pre : " <b> 3.8. </b> "
---

Một hệ thống không quan sát được là hệ thống không thể khẳng định là đang chạy tốt.
Phần này bổ sung ba loại tín hiệu — log, metric và alarm — cùng quyền truy cập vận
hành cần thiết để điều tra khi một trong số đó kêu.

#### Theo dõi cái gì, và vì sao

| Câu hỏi | Được trả lời bởi | Mục |
|---|---|---|
| Ứng dụng đã làm gì? | CloudWatch Logs | [3.8.1](3.8.1-logs/) |
| Hệ thống vận hành ra sao theo thời gian? | CloudWatch Metrics + dashboard | [3.8.2](3.8.2-metrics-dashboard/) |
| Có cần đánh thức ai không? | CloudWatch Alarms + SNS | [3.8.3](3.8.3-alarms/) |
| Ngay lúc này bên trong đang xảy ra chuyện gì? | ECS Exec | [3.8.4](3.8.4-debugging/) |

#### Vì sao phần giám sát nằm trong `BorrowitApp`

Một stack giám sát riêng sẽ phải **import** từ `BorrowitApp` — và điều đó không được
phép, vì quan hệ phụ thuộc một chiều chính là thứ giữ cho `cdk destroy BorrowitApp`
hoạt động ([3.1](../3.1-Overview/)).

Cái giá của việc đặt chung là khi xoá ứng dụng thì alarm và dashboard cũng mất theo.
Điều đó chấp nhận được: khi không còn dịch vụ nào chạy, chúng cũng chẳng còn gì để
theo dõi.

#### Những gì cố ý không dùng

Chọn bỏ cái gì cũng là một phần của thiết kế:

| Dịch vụ | Chi phí | Vì sao không dùng |
|---|---|---|
| **Container Insights** | ~$0.30/metric/tháng, hàng chục metric | Metric chi tiết theo container; metric sẵn có của ECS service đã trả lời đủ các câu hỏi đang đặt ra |
| **AWS X-Ray** | $5 mỗi triệu trace | Tracing phân tán phát huy giá trị khi có nhiều dịch vụ; ở đây chỉ một dịch vụ và một database |
| **CloudWatch Synthetics** | ~$0.0012 mỗi lần chạy canary | Một phép thăm dò từ bên ngoài theo lịch sẽ thực sự hữu ích — nhưng alarm unhealthy-target bắt được cùng loại sự cố với chi phí thấp hơn |
| **APM bên thứ ba** | $$$ | Hoàn toàn vượt ngân sách |

<!-- TODO(prose): X-Ray là phần bị lược bỏ dễ bị chất vấn nhất. Câu trả lời trung
     thực là với một API và một database, các metric CloudWatch đã đủ khoanh vùng
     vấn đề về phía task hay phía database — tracing sẽ thêm chi phí mà không thêm
     câu trả lời. Nếu sau khi làm xong bạn thấy khác, hãy nói ra. -->

{{% notice note %}}
**Tổng chi phí phần này:** khoảng **$4,80/tháng** — một dashboard ~$3, bảy alarm
tiêu chuẩn ~$0,10 mỗi cái, hai custom metric suy ra từ log ~$0,30 mỗi cái, thu
thập và lưu trữ log dưới $1, và gửi email qua SNS gần như miễn phí ở mức này. Đây là chỗ duy nhất trong dự án mà việc giám sát được phép tốn tiền.
{{% /notice %}}

#### Nội dung

1. [Log](3.8.1-logs/)
2. [Metric và dashboard](3.8.2-metrics-dashboard/)
3. [Alarm và thông báo](3.8.3-alarms/)
4. [Gỡ lỗi task đang chạy](3.8.4-debugging/)
