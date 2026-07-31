---
title : "Log"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 3.8.1 </b> "
---

#### Output của container tới CloudWatch bằng cách nào

```typescript
const logGroup = new logs.LogGroup(this, 'ApiLogGroup', {
  logGroupName: '/borrowit/api',
  retention: logs.RetentionDays.ONE_WEEK,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});

logging: ecs.LogDrivers.awsLogs({
  streamPrefix: 'borrowit',
  logGroup,
}),
```

Log group được **khai báo tường minh** thay vì để log driver tự sinh ra, vì hai lý
do: các metric filter ở [3.8.3](../3.8.3-alarms/) và các saved query bên dưới cần
một chỗ để gắn vào, và một cái tên cố định giúp tìm được log group trong console
mà không phải tra hậu tố do CloudFormation sinh ra trước.

Driver `awslogs` gửi mọi thứ container ghi ra stdout và stderr thẳng lên CloudWatch
Logs. Ứng dụng không cần SDK ghi log, không cần agent, không cần xoay vòng file —
ghi ra stdout là toàn bộ phần tích hợp.

**Thời gian giữ log đặt là một tuần.** Mặc định của CloudWatch Logs là *không bao
giờ hết hạn*, và với một dự án chạy dài thì điều đó tích luỹ chi phí lưu trữ vô thời
hạn cho những log không ai đọc. Một tuần đủ dài để điều tra sự cố và đủ ngắn để chi
phí lưu trữ không đáng kể.

<!-- TODO(prose): đáng viết một câu về đánh đổi — một tuần nghĩa là sự cố phát hiện
     sau nửa tháng sẽ không điều tra được từ log. Với dự án này thì chấp nhận được;
     với hệ thống bị kiểm toán thì không. -->

#### Mỗi task một stream

Log group chứa mỗi task một stream, đặt tên `borrowit/api/<task-id>`. Khi ECS thay
task, task mới nhận stream mới — điều này hữu ích, vì ranh giới giữa task cũ và task
mới hiện ra ngay trong cấu trúc log.

#### Đọc log từ CLI

Tên log group là cố định, nên không phải tra cứu gì cả:

```powershell
aws logs tail /borrowit/api --follow --region ap-southeast-1
```

Stack cũng xuất ra tên đó, kèm luôn câu lệnh trong phần mô tả:

```powershell
aws cloudformation describe-stacks --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='LogGroupName'].OutputValue" --output text
```

#### Logs Insights

Với những việc phức tạp hơn theo dõi trực tiếp, **CloudWatch Logs Insights** truy
vấn thẳng log group. Vài truy vấn khởi đầu hữu ích:

```
fields @timestamp, @message
| filter @message like /error/
| sort @timestamp desc
| limit 50
```

Đếm lỗi theo thời gian:

```
fields @timestamp
| filter @message like /error/
| stats count() by bin(5m)
```

![Truy vấn CloudWatch Logs Insights kèm kết quả](/images/3-Workshop/3.8-Observability/3.8.1-logs/insights-query.png?width=100pc)

#### Saved query

Viết cú pháp truy vấn từ trí nhớ đúng lúc có sự cố là thời điểm tệ nhất để làm việc
đó, nên năm truy vấn được lưu sẵn cùng stack:

```typescript
const queryLines = [
  'fields @timestamp, path, message, stack',
  "filter level = 'error'",
  'sort @timestamp desc',
  'limit 50',
];

new logs.CfnQueryDefinition(this, 'Query0', {
  name: 'BorrowIt/Errors with stack traces',
  logGroupNames: [logGroup.logGroupName],
  queryString: queryLines.join('\n| '),
});
```

Chúng xuất hiện ở **Logs Insights → Saved queries → BorrowIt**, và chúng miễn phí:

| Truy vấn | Trả lời |
|---|---|
| Errors with stack traces | Cái gì hỏng, và hỏng ở đâu trong mã nguồn |
| Slowest requests | Những lệnh gọi cụ thể nào chậm |
| Status code breakdown | Phân bố các mã phản hồi |
| Traffic by route | Endpoint nào thực sự được dùng |
| Raw error lines | Các dòng `console.error` văn bản thuần mà truy vấn có cấu trúc bỏ sót |

{{% notice warning %}}
Chuỗi ký tự trong Insights phải đặt trong **nháy đơn**. Insights đọc một token
trong nháy kép như *tên field*, nên `type = "request"` so sánh field `type` với
field `request` — trả về không dòng nào mà không hề báo lỗi. Một filter nhìn thì
đúng nhưng âm thầm không khớp gì cả còn tệ hơn một lỗi cú pháp.
{{% /notice %}}

{{% notice note %}}
Logs Insights tính phí **theo GB quét** ($0.0057/GB tại `ap-southeast-1`). Ở mức log
này mỗi truy vấn tốn chưa tới một cent, nhưng thu hẹp khoảng thời gian trước khi
chạy truy vấn rộng là thói quen tốt — trên một log group lớn, đó là khác biệt giữa
miễn phí và đắt đỏ.
{{% /notice %}}

#### Nên ghi gì và không nên ghi gì

<!-- TODO(prose): đây là điểm vận hành quan trọng. Vì mật khẩu database được inject
     dưới dạng biến môi trường bên trong container, bất cứ thứ gì in ra toàn bộ môi
     trường lúc khởi động — một lệnh debug, một handler ngoại lệ serialise
     process.env — sẽ ghi mật khẩu vào CloudWatch Logs dưới dạng văn bản thuần, phá
     bỏ toàn bộ thiết kế Secrets Manager ở 3.3.4.

     Nếu bạn đã kiểm tra điều này thì hãy nói ra. Nếu bạn tìm thấy và đã xoá thì càng tốt. -->

{{% notice warning %}}
Mọi thứ in ra stdout đều được lưu trong CloudWatch Logs và đọc được bởi bất kỳ ai có
quyền `logs:GetLogEvents`. Hãy kiểm tra không có đoạn chẩn đoán lúc khởi động nào in
ra toàn bộ biến môi trường — `DB_PASSWORD` được inject nằm ở đó.
{{% /notice %}}

{{% notice note %}}
**Chi phí:** phí nhận log khoảng $0.63/GB và lưu trữ khoảng $0.03/GB-tháng tại
`ap-southeast-1`. Ở mức lưu lượng này, thấp hơn nhiều so với $1/tháng.
{{% /notice %}}
