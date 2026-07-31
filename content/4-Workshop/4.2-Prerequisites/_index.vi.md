---
title : "Chuẩn bị"
date : 2026-07-28
weight : 2
chapter : false
pre : " <b> 4.2. </b> "
---

#### Công cụ

| Công cụ | Phiên bản | Dùng để làm gì |
|---|---|---|
| AWS CLI | v2 | Xác thực, đăng nhập ECR, đọc output của stack |
| Node.js | 22 | Ứng dụng CDK |
| Docker Desktop | bản mới | Build image cho API — bắt buộc phải **đang chạy** |
| Session Manager plugin | mới nhất | `aws ecs execute-command` ở [3.6.4](../4.6-Compute/4.6.4-database-init/) và [3.8.4](../4.8-Observability/4.8.4-debugging/) |

Session Manager plugin cài riêng, không đi kèm AWS CLI:
[hướng dẫn cài đặt](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html).

#### Tài khoản AWS

Kiểm tra CLI đã xác thực và trỏ đúng tài khoản:

```powershell
aws sts get-caller-identity
```

Kiểm tra số credit trước khi triển khai thứ gì có tính phí:

```powershell
aws freetier get-account-plan-state
```

![Trạng thái account plan kèm số dư tín dụng còn lại](/images/4-Workshop/4.2-Prerequisites/account-plan.png?width=100pc)

{{% notice warning %}}
**Đặt cảnh báo ngân sách trước khi deploy.** Billing → Budgets → tạo budget $10 kèm
thông báo email. Application Load Balancer tính phí khoảng $0.0225/giờ bất kể có
request nào tới hay không, nên một stack quên xoá là chi phí thật trừ vào số credit
cố định.
{{% /notice %}}

#### Quyền IAM

Triển khai workshop này cần quyền trên CloudFormation, EC2/VPC, ECS, ECR, RDS, S3,
CloudFront, Secrets Manager, CloudWatch, SNS, IAM và SSM.

<!-- TODO: nếu bạn dùng định danh administrator thì hãy nói thẳng, đừng ngụ ý một
     policy đặc quyền tối thiểu mà bạn không thực sự xây. Nếu bạn có xây policy thu
     hẹp thật thì hãy dán vào đây — đó là một hiện vật rất giá trị. -->

#### Bootstrap CDK

CDK cần một số tài nguyên có sẵn trong tài khoản trước khi deploy được — một S3
bucket chứa asset, một ECR repository và vài IAM role. Bước này chạy **một lần cho
mỗi tài khoản và mỗi region**.

```powershell
cd infra
npm install

$env:ACCOUNT = (aws sts get-caller-identity --query Account --output text)
npx cdk bootstrap "aws://$env:ACCOUNT/ap-southeast-1"
```

Bootstrap tạo ra một CloudFormation stack tên `CDKToolkit`, xem được trong
[CloudFormation console](https://ap-southeast-1.console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks).

#### Kiểm tra trước khi triển khai

`cdk synth` biên dịch TypeScript thành template CloudFormation mà không tạo gì cả.
Đây là cách rẻ nhất để phát hiện lỗi.

```powershell
npx cdk synth -c albDns=none
npx cdk list
```

`cdk list` sẽ in ra bốn stack:

```
BorrowitFoundation
BorrowitData
BorrowitFrontend
BorrowitApp
```

{{% notice note %}}
`-c albDns=none` là bắt buộc ở lần synth hoặc deploy **đầu tiên**.
`BorrowitFrontend` tra cứu DNS name của load balancer từ tham số SSM
`/borrowit/alb-dns` do `BorrowitApp` công bố — và trên một tài khoản mới, tham số
đó chưa tồn tại. Flag này bảo frontend stack tạm bỏ qua các route API. Mục
[3.7](../4.7-Delivery/) giải thích toàn bộ cách sắp xếp này.
{{% /notice %}}

#### Thứ tự triển khai

Các stack deploy theo thứ tự phụ thuộc. `BorrowitApp` đi sau cùng: nó import từ
cả ba stack còn lại, và sẽ không deploy được cho tới khi ECR có image.

```
3.4  BorrowitFoundation    VPC, security group, ECR
3.5  BorrowitData          RDS, Secrets Manager        (~10 phút)
3.7  BorrowitFrontend      S3, CloudFront              (~20 phút)  -c albDns=none
3.6  BorrowitApp           ECS, Fargate, ALB, alarm    (~5 phút)
3.7  npm run wire          trỏ CloudFront sang ALB     (~5 phút)
```

Có hai điểm trong thứ tự đó đáng để ý trước khi bắt đầu, vì cả hai đều dễ vấp:

+ **Số hiệu mục không chạy tuần tự.** `BorrowitFrontend` (3.7) được deploy
  **trước** `BorrowitApp` (3.6), vì AppStack cần uploads bucket và distribution
  URL làm đầu vào. Bạn vẫn *đọc* 3.6 trước 3.7 — tầng compute là nơi có những
  quyết định thú vị — nhưng lệnh deploy ở 3.6.2 sẽ kéo `BorrowitFrontend` theo
  nếu bạn chưa deploy nó.
+ **Phụ thuộc chỉ vòng tròn về mặt hình thức.** FrontendStack cần hostname của
  ALB; AppStack cần distribution URL. Nút thắt được gỡ bằng cách tách lần deploy
  frontend làm hai: một lần không có route API, rồi một lần nữa — qua
  `npm run wire` — khi load balancer đã tồn tại. Không có gì import từ
  `BorrowitApp` ở bất kỳ thời điểm nào, và đó là điều giữ cho
  `cdk destroy BorrowitApp` chạy được.
