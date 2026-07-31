---
title : "Dọn dẹp"
date : 2026-07-28
weight : 11
chapter : false
pre : " <b> 3.11. </b> "
---

Chúc mừng bạn đã hoàn thành workshop này.

#### Những gì đã dựng

Một hệ thống hoàn chỉnh trên AWS, định nghĩa hoàn toàn bằng code, với mọi lựa chọn
dịch vụ đều được lập luận đối chiếu với một phương án thay thế cụ thể:

+ **Compute** — ECS trên Fargate thay vì Lambda, vì workload là tiến trình chạy dài
  kèm connection pool, và RDS Proxy sẽ xoá sạch lợi thế giá của Lambda.
+ **Cơ sở dữ liệu** — RDS PostgreSQL thay vì Aurora Serverless v2 hay DynamoDB, vì
  schema đã tồn tại và chi phí cố định tốt hơn chi phí biến động trên một số dư
  credit cố định.
+ **Mạng** — public subnet không NAT Gateway, vì NAT tốn hơn toàn bộ ngân sách
  compute, với việc cách ly chuyển vào một chuỗi security group rồi được chứng minh
  bằng thực nghiệm.
+ **Phân phối** — S3 phía sau CloudFront với Origin Access Control, nên bucket không
  bao giờ ở chế độ public, và API được định tuyến qua **cùng distribution** để nó
  same-origin với trang gọi nó — đó chính là thứ làm cho một frontend HTTPS và một
  load balancer chỉ có HTTP hoạt động được cùng nhau.
+ **Secret** — Secrets Manager, giữ hai giá trị mà không người nào từng nhìn thấy:
  mật khẩu database do RDS sinh ra, và khoá ký JWT do CloudFormation sinh ra.
+ **Giám sát** — log, metric, dashboard và bảy alarm của CloudWatch, trong đó có một
  alarm cố ý coi việc thiếu dữ liệu là vi phạm.

#### Nên xoá gì và giữ gì

Cách chia bốn stack tồn tại để quyết định này trở nên đơn giản.

| Stack | Chi phí/tháng | Khuyến nghị |
|---|---|---|
| `BorrowitApp` | ~$35 | Xoá khi nghỉ nhiều tuần |
| `BorrowitData` | ~$16 | **Giữ** — xoá là mất dữ liệu |
| `BorrowitFrontend` | ~$1 | Giữ |
| `BorrowitFoundation` | ~$0 | Giữ |

Chỉ xoá tầng tốn tiền:

```powershell
cd infra
npm run down       # cdk destroy BorrowitApp - khoảng 2 phút
```

Dữ liệu và image đã push vẫn còn, vì RDS nằm trong `BorrowitData` và ECR repository
nằm trong `BorrowitFoundation` — cả hai đều là vị trí đặt có chủ đích ở
[3.4](../3.4-Network/) và [3.5](../3.5-Data/). Khôi phục bằng:

```powershell
npx cdk deploy --exclusively BorrowitApp -c imageTag=v1
npm run wire
```

`npm run up` nối đúng hai lệnh đó — nhưng nó **không** truyền `-c imageTag`, nên
nó deploy tag mặc định là `latest`. Nếu bạn làm theo
[3.6.1](../3.6-Compute/3.6.1-image-and-registry/) và đã push `v1` chứ không phải
`latest`, task khôi phục sẽ không khởi động được với lỗi
`CannotPullContainerError`. Hoặc truyền tag tường minh như trên, hoặc push thêm
một tag `latest`.

{{% notice note %}}
`npm run wire` cần thiết ở đây vì xoá `BorrowitApp` là xoá luôn load balancer, và
bản thay thế nhận một **DNS name mới**. Không đấu nối lại thì CloudFront vẫn định
tuyến `/api/*` tới một hostname không còn phân giải được — và vì fallback SPA chỉ
gắn vào behaviour mặc định, các lệnh gọi đó sẽ lỗi chứ không âm thầm trả về HTML.
Khôi phục stack đúng là trường hợp mà [3.7](../3.7-Delivery/) đã cảnh báo.
{{% /notice %}}

{{% notice warning %}}
**Không** chạy `cdk destroy --all` trừ khi bạn đã hoàn toàn kết thúc dự án.
`BorrowitData` dùng `removalPolicy: DESTROY`, nên xoá nó là xoá luôn database **và**
các bản backup tự động. Dựng lại đồng nghĩa phải chạy lại schema và seed ở
[3.6.4](../3.6-Compute/3.6.4-database-init/).
{{% /notice %}}

#### Xoá toàn bộ

Chỉ làm khi dự án đã thực sự kết thúc:

```powershell
npx cdk destroy BorrowitApp
npx cdk destroy BorrowitFrontend
npx cdk destroy BorrowitData
npx cdk destroy BorrowitFoundation
```

**Thứ tự quan trọng.** `BorrowitApp` phải xoá trước — nó là stack import giá trị từ
ba stack còn lại, và CloudFormation sẽ từ chối xoá một stack có output vẫn đang được
import. Đây là quan hệ phụ thuộc một chiều ở [3.1](../3.1-Overview/) xuất hiện
lần cuối.

Stack bootstrap `CDKToolkit` được giữ lại. Nó gần như không tốn phí và sẽ cần lại ở
lần deploy sau.

#### Xác nhận không còn gì tính tiền

```powershell
aws elbv2 describe-load-balancers --region ap-southeast-1 --query "LoadBalancers[].LoadBalancerName"
aws rds describe-db-instances --region ap-southeast-1 --query "DBInstances[].DBInstanceIdentifier"
aws ecs list-clusters --region ap-southeast-1
aws ec2 describe-addresses --region ap-southeast-1 --query "Addresses[].PublicIp"
```

Lệnh cuối quan trọng: một Elastic IP không gắn với tài nguyên nào vẫn bị tính tiền,
và rất dễ bị bỏ quên.

Sau đó kiểm tra số credit còn lại:

```powershell
aws freetier get-account-plan-state
```

#### Những hạn chế đã biết

Gọi tên những gì còn dang dở hữu ích hơn là ngụ ý rằng mọi thứ đã xong:

| Hạn chế | Ảnh hưởng | Hướng khắc phục |
|---|---|---|
| **ALB gọi trực tiếp được qua HTTP** | Ai biết hostname đều có thể bỏ qua edge và gọi API không mã hoá | Giới hạn security group của ALB theo prefix list `origin-facing` của CloudFront ($0), hoặc tên miền + chứng chỉ ACM ([3.7](../3.7-Delivery/)) |
| **CORS đặt `origin: true`** | API phản chiếu mọi origin kèm cho phép credential. Không còn ý nghĩa với đường đi của trình duyệt vì nay đã same-origin, nhưng nó mở rộng thêm phần rủi ro gọi thẳng vào ALB ở trên | Giới hạn về đúng tên miền CloudFront |
| **Chưa có CI/CD** | Deploy chạy từ máy cá nhân | GitHub Actions để build, push và `cdk deploy` |
| **Mặc định single-AZ** | Database là điểm hỏng đơn lẻ | Multi-AZ, với chi phí gấp đôi |
| **Chưa có kiểm thử tự động** | Circuit breaker khi deploy là lưới an toàn duy nhất | Bổ sung bộ kiểm thử trước khi thêm CI/CD |
| **Secret không được xoay vòng** | Một mật khẩu database bị lộ vẫn còn hiệu lực vô thời hạn | Xoay vòng bằng Lambda của Secrets Manager — không bật ở đây vì lý do phạm vi và chi phí |

<!-- TODO(prose): kết lại bằng việc bạn sẽ làm gì tiếp nếu có thêm thời gian và ngân
     sách, xếp theo thứ tự ưu tiên. Một workshop kết thúc bằng việc gọi tên chính
     những lỗ hổng của mình đọc đáng tin hơn nhiều so với một workshop kết thúc bằng
     lời khẳng định đã hoàn hảo. -->
