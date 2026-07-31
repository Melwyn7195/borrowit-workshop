---
title : "Alarm và thông báo"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 4.8.3 </b> "
---

Dashboard cho bạn biết có vấn đề khi bạn nhìn vào nó. Alarm cho bạn biết mà không
cần ai hỏi.

#### Đường đi của thông báo

```typescript
const alarmTopic = new sns.Topic(this, 'AlarmTopic', {
  displayName: 'BorrowIt alarms',
});

if (alarmEmail) {
  alarmTopic.addSubscription(new subscriptions.EmailSubscription(String(alarmEmail)));
}
```

SNS topic luôn được tạo để alarm luôn có nơi gửi — một topic không có subscription
thì không tốn phí. Địa chỉ email truyền qua context thay vì viết cứng, để email cá
nhân không nằm trong repository:

```powershell
npx cdk deploy BorrowitApp -c alarmEmail=you@example.com
```

{{% notice warning %}}
AWS gửi một email xác nhận và **subscription không gửi gì cho tới khi bạn bấm vào
liên kết**. Subscription chưa xác nhận là lý do phổ biến nhất khiến một alarm bị cho
là "không kêu" trong khi thực tế nó có kêu.
{{% /notice %}}

#### Bảy alarm

| Alarm | Metric | Điều kiện | Số chu kỳ | Khi thiếu dữ liệu |
|---|---|---|---|---|
| `unhealthy-targets` | `UnHealthyHostCount` | ≥ 1 | 1 × 5 phút | **BREACHING** |
| `api-5xx` | Số lỗi 5xx của target | > 5 | 1 × 5 phút | NOT_BREACHING |
| `service-cpu` | CPU % của ECS | > 80 | 3 × 5 phút | NOT_BREACHING |
| `service-memory` | Bộ nhớ % của ECS | > 85 | 3 × 5 phút | NOT_BREACHING |
| `db-cpu` | CPU % của RDS | > 80 | 3 × 5 phút | NOT_BREACHING |
| `db-storage` | Dung lượng trống RDS | < 2 GiB | 1 × 5 phút | NOT_BREACHING |
| `error-logs` | `BorrowIt/ApiErrorLogCount` | > 20 | 1 × 5 phút | NOT_BREACHING |

Sáu alarm đến từ các metric hạ tầng do AWS tự công bố. Alarm thứ bảy đến từ
**log** — xem phần bên dưới.

#### `treatMissingData` là mấu chốt của tất cả

Đây là khái niệm quan trọng nhất phần này, và rất dễ đặt sai theo cách âm thầm vô
hiệu hoá alarm.

```typescript
new cloudwatch.Alarm(this, 'UnhealthyTargetsAlarm', {
  metric: targetGroup.metrics.unhealthyHostCount({ period }),
  threshold: 1,
  evaluationPeriods: 1,
  treatMissingData: cloudwatch.TreatMissingData.BREACHING,
}),
```

**Một task đã biến mất hoàn toàn thì không gửi dữ liệu nào cả — chứ không phải gửi
giá trị 0.** Với hành vi mặc định `MISSING` của CloudWatch, alarm sẽ giữ nguyên
trạng thái cuối, còn với thiết lập phổ biến `NOT_BREACHING` thì nó bị hiểu là khoẻ
mạnh. Dịch vụ có thể sập hoàn toàn trong khi alarm vẫn xanh.

`BREACHING` nghĩa là "không có dữ liệu chính là tình huống khẩn cấp", và với metric
này thì đó đúng là điều cần.

Alarm 5xx thì ngược lại hoàn toàn. **Không có request nào trong khung 5 phút không
phải là lỗi** — đó là 3 giờ sáng trên một hệ thống trình diễn. Đặt thiếu dữ liệu là
vi phạm ở đó sẽ báo động cho bạn mỗi đêm. Nên `NOT_BREACHING` mới đúng.

Cùng một thiết lập, giá trị ngược nhau, vì hai metric mang ý nghĩa khác nhau khi im lặng.

<!-- TODO(prose): đây là điểm đáng giá nhất trang này để giải thích khi bị chất vấn.
     Hãy viết hai ba câu bằng lời của bạn về việc vì sao "không có dữ liệu" mang ý
     nghĩa trái ngược ở hai metric này. -->

#### Vì sao số chu kỳ đánh giá khác nhau

+ **`unhealthy-targets` và `db-storage` kêu sau một chu kỳ.** Cả hai đều là tình
  trạng đã hỏng rồi — không có gì để chờ thêm.
+ **Các alarm CPU và bộ nhớ cần ba chu kỳ liên tiếp (15 phút).** Một lần vọt lên
  trên 80% CPU trong 5 phút là bình thường khi deploy hoặc khi có đợt tải ngắn. Báo
  động vì điều đó chỉ tạo nhiễu, và một alarm mà bạn học cách phớt lờ còn tệ hơn là
  không có alarm.

#### Vì sao cần `db-storage`

Storage autoscaling bị tắt có chủ đích
([4.3.2](../../4.3-Architecture/4.3.2-database/)) để RDS không âm thầm làm tăng
hoá đơn. Hệ quả là **hết dung lượng đĩa trở thành tình huống hỏng thật sự** chứ
không phải thứ RDS lặng lẽ tự xử lý — nên nó cần một alarm. Đây là ví dụ hay về việc
một quyết định chi phí tạo ra một yêu cầu vận hành.

#### Alarm đến từ log

Sáu alarm còn lại theo dõi những metric mà AWS tự công bố sẵn. `error-logs` theo
dõi thứ do chính ứng dụng ghi ra, và điều đó đòi hỏi phải biến các dòng log thành
metric trước đã:

```typescript
const errorLogMetric = logGroup
  .addMetricFilter('ErrorLogFilter', {
    filterPattern: logs.FilterPattern.anyTerm('ERROR', 'Error:', 'error:'),
    metricNamespace: 'BorrowIt',
    metricName: 'ApiErrorLogCount',
    metricValue: '1',
    defaultValue: 0,
  })
  .metric({ period, statistic: 'Sum' });
```

Nó tồn tại vì nó bắt được những lỗi mà metric của ALB **về bản chất không thể
thấy** — một truy vấn hỏng bị bắt lại và trả về 400, một email không gửi được.
Những thứ đó không phải 5xx, nên `api-5xx` vẫn im lặng trong khi tính năng đã
hỏng.

Hai chi tiết là có chủ đích:

+ **Filter khớp văn bản thuần, không phải JSON.** `index.js` phát ra lỗi có cấu
  trúc, nhưng các lệnh `console.error` rải rác trong controller ("Login error:",
  "Auth middleware error:") là chuỗi thuần — và chúng vẫn chiếm phần lớn những gì
  thực sự được ghi lại khi có sự cố. Một filter JSON sẽ âm thầm bỏ sót tất cả.
+ **`defaultValue: 0`** khiến filter phát ra số 0 tường minh khi log về mà không
  có lỗi nào, để alarm phân biệt được "yên tĩnh" với "không có dữ liệu".

Bản thân metric filter là miễn phí — chúng chạy trên log vốn đã được thu thập.
Custom metric mà chúng sinh ra tốn ~$0,30/tháng.

{{% notice note %}}
Ngưỡng 20 là một **phỏng đoán, không phải một phép đo**. Vài lần đăng nhập thất
bại mỗi giờ là bình thường; hai mươi lỗi trong năm phút thì không. Hãy theo dõi
vài ngày rồi chỉnh lại, thay vì để nó ồn ào.
{{% /notice %}}

<!-- TODO(prose): nếu bạn đã dịch chuyển ngưỡng sau khi quan sát lưu lượng thật,
     hãy nói bạn chuyển sang mức nào và vì sao. Một alarm được tinh chỉnh theo hành
     vi quan sát được đáng giá hơn nhiều so với một alarm đặt theo phỏng đoán rồi
     không bao giờ xem lại. -->

<!-- SCREENSHOT: /images/4-Workshop/4.8-Observability/4.8.3-alarms/alarms-list.png
     Cả bảy alarm trong console CloudWatch kèm trạng thái hiện tại. -->

#### Chứng minh một alarm có kêu

Một alarm chưa từng thấy nó kêu là một alarm bạn không biết có hoạt động hay không.

```powershell
# Hạ về 0 task - unhealthy-targets sẽ chuyển ALARM sau khoảng 5 phút
npx cdk deploy BorrowitApp -c desiredCount=0

# ...xác nhận alarm chuyển sang ALARM và email đã tới, rồi khôi phục
npx cdk deploy BorrowitApp
```

{{% notice warning %}}
`desiredCount=0` dừng task nhưng **load balancer vẫn tính tiền**. Đây là một phép
thử, không phải cách tiết kiệm chi phí — hãy khôi phục dịch vụ sau khi xong.
{{% /notice %}}

<!-- TODO(prose): ghi lại alarm thực sự mất bao lâu mới kêu. Đáp án kỳ vọng là 5-10
     phút, vì chu kỳ metric là 5 phút và CloudWatch đánh giá sau khi chu kỳ đóng.
     Nếu lâu hơn bạn nghĩ, độ trễ đó đáng biết trước khi bạn tin vào các alarm này. -->

{{% notice note %}}
**Chi phí:** alarm tiêu chuẩn giá **$0,10/tháng mỗi cái** — $0,70 cho bảy cái —
cộng thêm ~$0,30/tháng cho custom metric phía sau `error-logs`. Gửi email qua SNS
miễn phí cho 1.000 thông báo đầu tiên mỗi tháng.
{{% /notice %}}
