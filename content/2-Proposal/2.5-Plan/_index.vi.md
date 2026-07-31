---
title : "Kế hoạch triển khai"
date : 2026-07-28
weight : 5
chapter : false
pre : " <b> 2.5. </b> "
---

Kế hoạch đi theo **thứ tự phụ thuộc giữa các stack**, không theo thứ tự trong sơ
đồ kiến trúc. Không thứ gì dựng được trước cái mà nó import, và chỉ một quy tắc đó
đã cố định toàn bộ trình tự.

#### Các giai đoạn

| Giai đoạn | Tuần | Kết quả | Điều kiện để đi tiếp |
|---|---|---|---|
| **0 — Nền tảng** | 1–3 | Kiến thức AWS, tài khoản và ngân sách sẵn sàng, bản đề xuất này | Đề xuất được review và kiến trúc được thống nhất |
| **1 — Mạng và dữ liệu** | 4–5 | `BorrowitFoundation` và `BorrowitData` đã deploy | `psql` vào được DB từ trong VPC và không từ đâu khác |
| **2 — Compute** | 6–7 | Image nằm trong ECR, `BorrowitApp` phục vụ qua ALB | `GET /healthz` trả 200 qua load balancer |
| **3 — Phân phối nội dung** | 8 | `BorrowitFrontend`, web client chạy trên CloudFront | Ứng dụng chạy trọn vẹn từ trình duyệt |
| **4 — Vận hành** | 9–10 | Dashboard, alarm, diễn tập failover Multi-AZ, rà soát chi phí | Nhận được email alarm từ một hệ thống bị làm hỏng có chủ đích |
| **5 — Tài liệu** | 11–12 | Workshop ở phần 3, blog, báo cáo cuối | Người đọc dựng lại được hệ thống chỉ từ workshop |

#### Chi tiết theo tuần

| Tuần | Công việc | Sản phẩm | Kiểm chứng |
|---|---|---|---|
| 1 | Kiến thức AWS; tài khoản, MFA, budget alarm, cost tag | Tài khoản sẵn sàng | `aws freetier get-account-plan-state` hiện số dư credit |
| 2 | IAM, VPC, EC2, S3 làm tay — trước khi tự động hoá bất cứ thứ gì | Ghi chú và tài nguyên nháp | Xoá hết tài nguyên nháp; không còn gì tính tiền |
| 3 | Khảo sát BorrowIt phụ thuộc những gì; thiết kế đích; viết đề xuất | **Tài liệu này** | Sơ đồ kiến trúc và mô hình chi phí được review |
| 4 | CDK bootstrap; `BorrowitFoundation` — VPC, security group, ECR | Stack đã deploy | `cdk diff` sạch; subnet và route khớp thiết kế |
| 5 | `BorrowitData` — RDS, Secrets Manager, khôi phục schema và dữ liệu | DB chạy | Số dòng khớp bản export từ Supabase |
| 6 | Đóng gói API vào Docker; thêm `/healthz` và `/readyz`; push lên ECR | Image trong ECR | Container chạy ở máy cá nhân, kết nối tới endpoint RDS |
| 7 | `BorrowitApp` — cluster, task definition, service, ALB, target group | API tiếp cận được | 200 qua ALB; gọi thẳng IP của task thì timeout |
| 8 | `BorrowitFrontend` — bucket S3, CloudFront, OAC; trỏ client sang ALB | Toàn hệ thống chạy | Đầu-cuối: đăng tin, tải ảnh, tải lại trang |
| 9 | Log group và retention, metric filter, dashboard, sáu alarm, SNS | Có khả năng quan sát | Giết task; email alarm gửi đến |
| 10 | Bật Multi-AZ, ép failover, đo; load test; soát hoá đơn thật | Bằng chứng về khả năng phục hồi | Ghi lại thời gian failover; Cost Explorer đối chiếu ước tính |
| 11 | Viết workshop — mục 3.1 đến 3.11 kèm ảnh chụp | Bản nháp workshop | Một người khác làm theo từ tài khoản trống |
| 12 | Blog, báo cáo cuối, bàn giao, quyết định teardown | Bài nộp | Mọi dòng trong bảng này đều có bằng chứng kèm theo |

{{% notice note %}}
Kỳ thực tập kéo tới **04/09/2026**, nhưng hạn chấm là **21/08/2026** — cuối tuần
10. Tuần 11 và 12 là thời gian viết tài liệu và dự phòng, nghĩa là **hệ thống phải
chạy được từ cuối tuần 10**, không phải tuần 12. Mọi mốc kiểm tra ở trên đều đặt
theo cái hạn sớm hơn đó.
{{% /notice %}}

#### Thứ tự dựng, và vì sao không đổi được

{{<mermaid align="center">}}
graph LR
  A["Foundation - VPC, SG, ECR"] --> B["Data - RDS, Secrets"]
  B --> C["Image - build, push to ECR"]
  C --> D["App - Fargate, ALB"]
  D --> E["Frontend - S3, CloudFront"]
  E --> F["Operations - alarms, failover, cost"]
{{</mermaid>}}

+ **Image phải có trong ECR trước khi tạo service** — một ECS service trỏ vào
  repository rỗng sẽ không bao giờ ổn định và stack rollback sau khoảng mười lăm
  phút chờ đợi.
+ **Cơ sở dữ liệu phải tồn tại trước khi task khởi động**, vì task đọc mật khẩu từ
  secret do RDS sinh ra.
+ **Frontend cần DNS name của ALB** để trỏ client tới API, nên nó đi sau tầng app
  chứ không đi trước.

<!-- TODO(prose): nếu thực tế có gặp lỗi rollback do ECR rỗng, hãy ghi lại thời
     điểm và thông báo lỗi ở phần 1 tuần 7. Một kế hoạch gọi tên trước một kiểu lỗi
     rồi vượt qua được chính là thứ giá trị nhất trong tài liệu này. -->

#### Cách triển khai

Cả bốn stack deploy từ máy cá nhân bằng CDK CLI:

```powershell
cd infra
npm install
npx cdk bootstrap aws://<account-id>/ap-southeast-1
npx cdk deploy BorrowitFoundation BorrowitData
npx cdk deploy BorrowitFrontend BorrowitApp
```

Không có CI/CD, theo quyết định phạm vi ở [2.2](../2.2-Requirements/). Đánh đổi là
mọi lần deploy đều thủ công và không lưu vết, điều chấp nhận được với một người và
vài lần deploy mỗi tuần.

#### Ước lượng công sức

| Giai đoạn | Ước tính | Độ tin cậy | Nơi dễ trễ nhất |
|---|---|---|---|
| 0 — Nền tảng | 3 tuần | Cao | — |
| 1 — Mạng và dữ liệu | 2 tuần | Trung bình | Độ chính xác của việc export và restore dữ liệu |
| 2 — Compute | 2 tuần | **Thấp** | Mạng container, health check, quyền IAM |
| 3 — Phân phối nội dung | 1 tuần | Cao | CORS giữa CloudFront và ALB |
| 4 — Vận hành | 2 tuần | Trung bình | Ngưỡng alarm cần chỉnh theo metric thật |
| 5 — Tài liệu | 2 tuần | Trung bình | Chụp ảnh màn hình đòi hỏi hệ thống vẫn đang chạy |

Giai đoạn 2 gánh rủi ro tiến độ. Đây là lần đầu tiên cả ba thứ — build container,
IAM task role và health check của load balancer — phải cùng đúng một lúc, mà mỗi
thứ khi hỏng lại trông giống hai thứ kia; xem [2.7](../2.7-Risks/).
