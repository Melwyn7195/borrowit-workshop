---
title : "Metric và dashboard"
date : 2026-07-28
weight : 2
chapter : false
pre : " <b> 3.8.2 </b> "
---

#### Metric đến từ đâu

Không metric nào trong số này do ứng dụng phát ra. Tất cả đều được chính dịch vụ AWS
tự động sinh ra, và đó là một phần lớn giá trị bạn đang trả tiền cho các dịch vụ này.

| Nguồn | Metric sử dụng | Do ai phát ra |
|---|---|---|
| ALB target group | `RequestCount`, `TargetResponseTime`, `HTTPCode_Target_5XX_Count`, `HealthyHostCount`, `UnHealthyHostCount` | Elastic Load Balancing |
| ALB | `HTTPCode_ELB_5XX_Count` | Elastic Load Balancing |
| ECS service | `CPUUtilization`, `MemoryUtilization` | Amazon ECS |
| RDS instance | `CPUUtilization`, `DatabaseConnections`, `FreeStorageSpace` | Amazon RDS |

Tất cả đều là metric **độ phân giải chuẩn (1 phút)**, không tính thêm phí. Dashboard
tổng hợp chúng theo chu kỳ 5 phút.

#### Target 5xx so với ELB 5xx

Dashboard vẽ cả hai, và sự phân biệt này có giá trị chẩn đoán:

+ **`HTTPCode_Target_5XX_Count`** — ứng dụng trả về lỗi. Task vẫn chạy và vẫn phản
  hồi; lỗi nằm ở code hoặc ở database.
+ **`HTTPCode_ELB_5XX_Count`** — chính *load balancer* sinh ra lỗi, vì không có
  target healthy nào để chuyển tiếp, hoặc target hết thời gian chờ.

Nhìn xem cái nào tăng vọt là biết ngay nên xem phía ứng dụng hay phía hạ tầng. Chỉ
vẽ một con số lỗi gộp chung là vứt bỏ sự phân biệt đó.

<!-- TODO(prose): nếu bạn thấy cả hai trong bài kiểm thử failover, hãy mô tả
     cái nào xuất hiện và vào lúc nào. Đó là loại quan sát cho thấy dashboard thực
     sự được dùng chứ không chỉ được dựng lên. -->

#### p50 và p95, không phải giá trị trung bình

```typescript
left: [
  targetGroup.metrics.targetResponseTime({ period, statistic: 'p50' }),
  targetGroup.metrics.targetResponseTime({ period, statistic: 'p95' }),
],
```

Độ trễ trung bình che mất chính những request gây khó chịu cho người dùng. Nếu 95
request mất 50 ms và 5 request mất 4 giây, giá trị trung bình là 250 ms nghe rất ổn
trong khi cứ hai mươi người thì một người đang có trải nghiệm tệ.

**p50** cho thấy trải nghiệm điển hình; **p95** cho thấy phần đuôi. Khoảng cách giữa
hai đường mới là tín hiệu — khoảng cách nới rộng nghĩa là có thứ gì đó chậm không
đều, thường mang nhiều thông tin hơn từng con số riêng lẻ.

#### Dashboard

Năm widget trên bốn hàng:

| Widget | Trục trái | Trục phải | Trả lời câu hỏi |
|---|---|---|---|
| Request và lỗi | `RequestCount` | Target 5xx, ELB 5xx | Có ai dùng không, và có lỗi không? |
| Độ trễ | p50, p95 | — | Có nhanh không, và nhanh với mọi người chứ? |
| Fargate task | CPU %, bộ nhớ % | — | Container có được cấp đúng kích thước? |
| Database | CPU RDS % | Số kết nối, dung lượng trống | Database có phải nút thắt cổ chai? |
| Healthy target | Số host healthy, unhealthy | — | Có gì đang thực sự phục vụ không? |

```powershell
aws cloudformation describe-stacks --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='DashboardUrl'].OutputValue" --output text
```

<!-- SCREENSHOT: /images/3-Workshop/3.8-Observability/3.8.2-metrics-dashboard/dashboard.png
     Chụp toàn bộ dashboard khi đã có dữ liệu thật. Hãy tạo traffic trước — một
     dashboard rỗng không chứng minh được gì và còn tệ hơn là không chụp. -->

#### Bộ nhớ mới là thứ cần canh

512 MiB không nhiều với Node.js. Widget bộ nhớ ở đây quan trọng hơn widget CPU:
Fargate **giết container vượt quá giới hạn bộ nhớ** — không có swap, không có suy
giảm êm ái, task đơn giản là chết với mã thoát 137 và ECS khởi động bản thay thế.

Cạn CPU chỉ làm mọi thứ chậm đi. Cạn bộ nhớ thì chết hẳn, và đó là lý do alarm bộ
nhớ ở [3.8.3](../3.8.3-alarms/) đặt tại 85% thay vì 80% như CPU.

<!-- TODO(prose): báo cáo mức bộ nhớ ở trạng thái ổn định mà bạn thực sự quan sát
     được. Nếu nó quanh 400 MiB thì dư địa rất mỏng và đáng gọi tên như một rủi ro.
     Nếu chỉ 150 MiB thì task đang được cấp thừa và có thể nhỏ hơn. -->

#### Đọc metric từ CLI

Hữu ích khi bạn cần một con số thay vì một hình vẽ:

```powershell
aws cloudwatch get-metric-statistics `
  --namespace AWS/ApplicationELB `
  --metric-name TargetResponseTime `
  --dimensions Name=LoadBalancer,Value=<lb-id> `
  --start-time (Get-Date).AddHours(-3).ToString("yyyy-MM-ddTHH:mm:ss") `
  --end-time (Get-Date).ToString("yyyy-MM-ddTHH:mm:ss") `
  --period 300 --statistics Average --region ap-southeast-1
```

{{% notice note %}}
**Chi phí:** một CloudWatch dashboard tốn **$3/tháng** sau ba dashboard đầu tiên,
đây là khoản lớn nhất trong ngân sách giám sát. Metric tiêu chuẩn từ các dịch vụ AWS
đều miễn phí; chỉ metric tuỳ chỉnh và dashboard bổ sung mới tính thêm tiền.
{{% /notice %}}
