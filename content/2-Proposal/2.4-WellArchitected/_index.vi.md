---
title : "Đánh giá Well-Architected"
date : 2026-07-28
weight : 4
chapter : false
pre : " <b> 2.4. </b> "
---

Thiết kế ở [2.3](../2.3-Design/) được soi lại ở đây theo **AWS Well-Architected
Framework** — sáu trụ cột, mỗi trụ cột đặt câu hỏi liệu kiến trúc đứng vững vì có
lý do hay chỉ vì tình cờ.

Việc đánh giá được làm **trước** khi dựng, và đó chính là lý do phải làm. Một bản
đánh giá Well-Architected chạy sau khi bàn giao chỉ ghi lại chuyện đã xảy ra; chạy
ở giai đoạn đề xuất thì nó thay đổi thứ sẽ được dựng.

{{% notice note %}}
Một workload chạy bằng $160 credit sẽ không đạt điểm cao trước một framework viết
cho hệ thống production, và bản đánh giá này không giả vờ ngược lại. Mỗi trụ cột
dưới đây ghi rõ **thiết kế làm được gì**, **chấp nhận đánh mất gì**, và **thứ gì
sẽ được sửa đầu tiên** nếu có ngân sách lớn hơn. Cột thứ ba mới là thứ chứng minh
framework thực sự đã được áp dụng.
{{% /notice %}}

#### Sáu trụ cột, nhìn tổng quan

| Trụ cột | Kết luận | Lỗ hổng lớn nhất |
|---|---|---|
| Operational excellence | **Tốt** | Không có CI/CD — mọi lần deploy đều thủ công |
| Security | **Chấp nhận được, kèm một ngoại lệ được nêu tên** | Task có public IP; traffic API đi HTTP |
| Reliability | **Yếu, và là chủ ý** | Một task, một region, `removalPolicy: DESTROY` |
| Performance efficiency | **Chấp nhận được** | Chưa load test tới tuần 10; không auto-scaling |
| Cost optimisation | **Tốt — và nó chi phối năm trụ cột còn lại** | ALB tốn hơn phần compute nó phục vụ |
| Sustainability | **Chấp nhận được** | Một workload demo chạy 24/7 |

---

#### 1. Operational excellence

*Vận hành, giám sát và liên tục cải tiến quy trình.*

| Trong thiết kế | Cơ chế |
|---|---|
| Hạ tầng dưới dạng mã | Bốn stack CDK; không thao tác console trong luồng dựng |
| Thay đổi nhỏ, đảo được | `cdk diff` trước mọi lần deploy; phụ thuộc stack một chiều |
| Sự cố quan sát được | Dashboard, 6 alarm, email SNS; log giữ 1 tuần trong CloudWatch |
| Deploy tự rollback | ECS circuit breaker tự quay lại khi deploy hỏng |
| Vận hành có tài liệu | Workshop ở phần 3 chính là runbook |

**Đánh mất:** không có pipeline CI/CD nên deploy thủ công và không lưu vết; không
có test tự động (backend vốn không có); không có quy trình xử lý sự cố chính thức.

**Cải tiến đầu tiên:** CodePipeline hoặc GitHub Actions deploy khi merge. Đây là cơ
chế còn thiếu rẻ nhất — chi phí pipeline gần như bằng không ở tần suất này, và nó
loại bỏ hoàn toàn kiểu lỗi "chạy được trên máy tôi".

#### 2. Security

*Bảo vệ dữ liệu, hệ thống và tài sản.*

| Trong thiết kế | Cơ chế |
|---|---|
| Danh tính | IAM task execution role và task role, chỉ giới hạn vào đúng một secret và một log group |
| Credential không bao giờ viết ra | RDS sinh mật khẩu vào Secrets Manager; nạp vào qua `valueFrom` |
| Cách ly mạng | Ba security group nối chuỗi — ALB 80, task 3456, database 5432 |
| Dữ liệu khi lưu trữ | RDS bật mã hoá; bucket S3 private, chỉ CloudFront OAC đọc được |
| Dữ liệu khi truyền | HTTPS từ trình duyệt tới CloudFront |

**Đánh mất, và nói thẳng:**

+ **Task chạy trong public subnet kèm public IP.** Một sai sót security group là
  ranh giới giữa cách ly và phơi ra ngoài, trong khi private subnet thì đơn giản là
  không có route nào. Lập luận nằm ở
  [3.3.3](../../3-Workshop/3.3-Architecture/3.3.3-networking/) và cái giá nằm ở
  [2.6](../2.6-Budget/).
+ **ALB phục vụ HTTP chứ không phải HTTPS** — không có domain để xác thực chứng
  chỉ, nên traffic API giữa trình duyệt và ALB không được mã hoá.
+ **Không có WAF, GuardDuty hay Config**, mỗi thứ đều cộng thêm chi phí định kỳ.

**Cải tiến đầu tiên:** một domain kèm chứng chỉ ACM — rẻ nhất trong ba thứ trên
(chứng chỉ miễn phí, chỉ domain tốn tiền) và bịt được lỗ hổng lớn nhất.

#### 3. Reliability

*Phục hồi sau sự cố và đáp ứng được nhu cầu.*

| Trong thiết kế | Cơ chế |
|---|---|
| Nền tảng trải hai AZ | Subnet ở hai Availability Zone; ALB vốn đã dự phòng theo AZ |
| Health check có ý nghĩa | `/healthz` (liveness, không chạm DB) tách khỏi `/readyz` (readiness, có kiểm tra DB) |
| Tự phục hồi | ECS tự thay thế task chết |
| Độ bền dữ liệu | RDS backup tự động; bật Multi-AZ khi cần và **đo** thời gian failover thay vì phỏng đoán |
| Khoanh vùng thay đổi hỏng | Rollback khi health check thất bại |

**Đánh mất:** `desiredCount: 1`, nên một task chết là 1–2 phút gián đoạn mà không
có gì để chuyển sang. Một region duy nhất. `removalPolicy: DESTROY` trên database —
chỉ biện minh được vì đó là dữ liệu mẫu.

**Cải tiến đầu tiên:** `desiredCount: 2`. Chỉ là một tham số, tốn khoảng $9/tháng,
và biến lỗ hổng lớn nhất về độ tin cậy thành chuyện không đáng kể.

#### 4. Performance efficiency

*Dùng tài nguyên tính toán hiệu quả khi nhu cầu thay đổi.*

| Trong thiết kế | Cơ chế |
|---|---|
| Compute vừa đủ | 0.25 vCPU / 0.5 GB — task Fargate nhỏ nhất, khớp với lưu lượng demo |
| Bộ xử lý hiện đại | `db.t4g.micro` là Graviton (ARM), giá/hiệu năng tốt hơn bản x86 tương đương |
| Nội dung gần người dùng | CloudFront cache tài nguyên tĩnh ở edge |
| Không có hạ tầng nằm không | Compute serverless; không cấp phát dư cho đỉnh tải rồi bỏ trống |
| Đo đạc thay vì phỏng đoán | Load test ở tuần 10 xác định trần thực tế |

**Đánh mất:** không có chính sách auto-scaling nên một đợt tăng tải do đúng một
container gánh; không có mốc đo hiệu năng cho tới tuần 10; không có tầng cache
trước cơ sở dữ liệu.

**Cải tiến đầu tiên:** chính sách ECS target-tracking theo CPU. Không tốn gì cho
tới khi nó thực sự kích hoạt.

#### 5. Cost optimisation

*Tránh những chi phí không cần thiết.*

Đây là trụ cột định hình năm trụ cột còn lại, nên bảng này đọc khác đi — các cơ
chế ở đây chính là lý do những trụ cột kia có lỗ hổng.

| Trong thiết kế | Cơ chế | Tiết kiệm |
|---|---|---|
| Không NAT Gateway | Public subnet, cách ly bằng security group | ~$33/tháng |
| Database mặc định single-AZ | Multi-AZ nằm sau context flag, chỉ dùng cho demo | ~$16/tháng |
| Instance Graviton | Dùng `t4g` thay vì `t3` | ~10–20% chi phí instance |
| Ghim storage autoscaling | `maxAllocatedStorage` bằng `allocatedStorage` | Chặn việc phình âm thầm |
| Quy trách nhiệm chi phí | Tag `Project` / `ManagedBy` gắn ở cấp CDK app | Khiến hoá đơn đọc được |
| Ý thức về chi tiêu | Budget alarm ở $10; ước tính đối chiếu Cost Explorer | — |
| Tầng đắt tiền có thể bỏ đi | Phụ thuộc stack một chiều khiến `cdk destroy BorrowitApp` luôn chạy được | ~$31/tháng khi rảnh |

**Đánh mất:** không có thứ gì bị hy sinh *cho* việc tối ưu chi phí mà chưa được
nêu tên ở các trụ cột khác — và đó là cách đọc trung thực bảng này.

**Dòng khó chịu nhất:** ALB tốn ~$17/tháng so với ~$9 của phần compute phía sau
nó. Ở mức một task, đây là khoản kém hiệu quả nhất trên hoá đơn, và là lý lẽ mạnh
nhất cho App Runner hoặc API Gateway HTTP API ở quy mô này.

#### 6. Sustainability

*Giảm thiểu tác động môi trường của việc vận hành workload.*

| Trong thiết kế | Cơ chế |
|---|---|
| Phần cứng hiệu quả | Graviton (ARM) cho database — làm được nhiều việc hơn trên mỗi watt so với x86 |
| Vừa đủ, không cấp phát dư | Task Fargate nhỏ nhất; storage chặn ở 20 GB |
| Region khớp với người dùng | `ap-southeast-1` là region gần nhất; không có traffic liên vùng |
| Dịch vụ được quản lý | Fargate và RDS chia sẻ hạ tầng bên dưới hiệu quả hơn instance riêng |
| Vòng đời dữ liệu | Log chỉ giữ một tuần; ECR lifecycle rule xoá image cũ |

**Đánh mất:** một workload demo chạy 24/7 chỉ để lúc nào cũng có cái để trình bày.
Xoá `BorrowitApp` khi không dùng vừa là câu trả lời cho chi phí, vừa là câu trả lời
cho tính bền vững.

---

#### Áp dụng framework cho đúng cách

Trang này là bản tự đánh giá. AWS có sẵn **Well-Architected Tool** trong console
(Console → Well-Architected Tool → Define workload), đi qua đúng sáu trụ cột dưới
dạng bộ câu hỏi có cấu trúc và sinh ra một kế hoạch cải tiến.

Công cụ này **miễn phí**, nên không tốn gì vào số dư credit.

<!-- TODO: chạy Well-Architected Tool trên workload này ở tuần 9 hoặc 10 rồi chụp
     lại kế hoạch cải tiến do nó sinh ra. Kết quả từ công cụ thật đặt cạnh bản đánh
     giá viết tay này là bằng chứng mạnh hơn nhiều so với chỉ có bản viết tay — và
     bất kỳ mục rủi ro cao nào công cụ nêu ra mà trang này bỏ sót đều đáng được ghi
     lại một cách trung thực thay vì lặng lẽ sửa đi. -->

#### Kế hoạch cải tiến, theo thứ tự ưu tiên

Mọi mục dưới đây đều là thứ *có chủ ý* chưa làm ngay. Đây là backlog mà bản đánh
giá sinh ra, xếp theo lợi ích trên mỗi đồng bỏ ra:

| # | Cải tiến | Trụ cột | Chi phí định kỳ |
|---|---|---|---|
| 1 | `desiredCount: 2` | Reliability | ~$9/tháng |
| 2 | CI/CD deploy khi merge | Operational excellence | ~$0 |
| 3 | ECS auto-scaling target-tracking | Performance | $0 cho tới khi kích hoạt |
| 4 | Domain + chứng chỉ ACM, HTTPS trên ALB | Security | Chỉ tốn tiền domain |
| 5 | Private subnet + NAT Gateway | Security | ~$33/tháng |
| 6 | RDS Multi-AZ thành cấu hình thường trực | Reliability | ~$16/tháng |

Mục 5 và 6 là hai thứ bị ngân sách chứ không phải kỹ thuật gạt đi. Nếu workload
này được cấp tiền thật thay vì credit, chúng sẽ là mục 1 và 2.
