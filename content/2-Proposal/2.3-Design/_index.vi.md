---
title : "Thiết kế kiến trúc"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 2.3. </b> "
---

Thiết kế suy ra từ các yêu cầu ở [2.2](../2.2-Requirements/) và từ một ràng buộc
lấn át mọi thứ còn lại: số dư cố định **$160 credit**. Mỗi khối trong sơ đồ dưới
đây đều được chọn kèm theo một con số chi phí hàng tháng.

#### Kiến trúc mục tiêu

{{<mermaid align="center">}}
graph LR
  U["Browser clients"]
  CF["CloudFront distribution"]
  S3W["S3 - web client"]
  S3U["S3 - user uploads"]
  ECR["ECR - images"]
  SM["Secrets Manager"]
  CW["CloudWatch"]
  SNS["SNS - email"]

  subgraph Public subnets in two AZs
    ALB["Application Load Balancer"]
    T["ECS Fargate task - 0.25 vCPU"]
    VPE["Interface VPC endpoints - PrivateLink"]
  end

  subgraph Isolated subnets in two AZs
    DB["RDS PostgreSQL 16 - db.t4g.micro"]
  end

  U -->|HTTPS| CF
  CF -->|OAC| S3W
  CF -->|OAC| S3U
  U -->|HTTP port 80| ALB
  ALB -->|port 3456| T
  T -->|port 5432| DB
  T -. port 443 .-> VPE
  VPE -. image pull .-> ECR
  VPE -. credentials at start .-> SM
  VPE -. logs and metrics .-> CW
  CW -. alarm .-> SNS
{{</mermaid>}}

Mũi tên liền là đường đi của request; mũi tên đứt là đường điều khiển, telemetry
và triển khai. Cả hai nhóm subnet đều nằm trong cùng một VPC **không có NAT
Gateway** — đường bao mà sơ đồ phẳng ở trên không vẽ được, và cũng là lý do tồn
tại của bản draw.io bên dưới.

Hãy để ý vị trí của `VPE`: mọi lời gọi từ task tới dịch vụ AWS đều đi *xuyên qua*
nó, chứ không vòng quanh nó. Đó chính là ý nghĩa của interface endpoint — nó không
phải một khối treo bên rìa sơ đồ, mà là cánh cửa trên tường VPC để traffic đi ECR,
Secrets Manager và CloudWatch thoát ra. Năm endpoint được tạo (`ecr.api`,
`ecr.dkr`, `logs`, `secretsmanager`, `ssmmessages`) và chúng là một phần của kiến
trúc đã triển khai chứ không phải một tuỳ chọn, với chi phí ~$36/tháng ở một AZ.
Phép tính nằm ở [4.3.3](../../4-Workshop/4.3-Architecture/4.3.3-networking/).

#### Sơ đồ có thể chỉnh sửa

Sơ đồ ở trên sinh ra từ chính mã nguồn của trang, nên dễ giữ cho chính xác nhưng
chỉ dừng ở mức hộp và mũi tên. Bản **dùng để trình bày** — icon dịch vụ AWS chính
thức, các đường bao AWS Cloud / Region / VPC / subnet vẽ theo đúng phong cách sơ
đồ kiến trúc tham chiếu của AWS — nằm ở tệp draw.io có thể chỉnh sửa:

&emsp; 📐 **[borrowit-architecture.drawio](../../diagrams/borrowit-architecture.drawio)**

{{% notice tip %}}
**Cách chỉnh sửa.** Mở [app.diagrams.net](https://app.diagrams.net) → *File → Open
From → Device*, hoặc cài extension **Draw.io Integration** cho VS Code rồi bấm đúp
vào tệp trong repository. Các shape AWS đã nhúng sẵn trong tệp, nhưng muốn thêm
shape mới thì bật *More Shapes → Networking → AWS 2025*. Xuất bằng *File → Export
as → PNG*, `zoom 200%`, tắt nền trong suốt, lưu vào
`static/images/2-Proposal/2.3-Design/architecture.png`.
{{% /notice %}}

![Kiến trúc mục tiêu](/images/2-Proposal/2.3-Design/architecture.png?width=100pc)

#### Danh mục thành phần

| Dịch vụ | Vai trò | Cấu hình đề xuất | Stack |
|---|---|---|---|
| **VPC** | Hai AZ, subnet public + isolated, không NAT | `/16`, bốn subnet `/24` | `BorrowitFoundation` |
| **ECR** | Registry chứa container image | Scan khi push, lifecycle rule | `BorrowitFoundation` |
| **Security group** | Ranh giới cách ly, thay cho private subnet | Ba nhóm, nối chuỗi | `BorrowitFoundation` |
| **VPC endpoint** | S3 gateway và năm interface endpoint, đều luôn bật | Gateway miễn phí; interface ~$36/tháng ở một AZ | `BorrowitFoundation` |
| **RDS PostgreSQL 16** | Kho dữ liệu quan hệ | `db.t4g.micro`, 20 GB, single-AZ | `BorrowitData` |
| **Secrets Manager** | Credential DB, sinh tự động chứ không viết tay | Một secret, do RDS quản lý | `BorrowitData` |
| **S3** | Bundle web client, ảnh người dùng | Hai bucket, đều private | `BorrowitFrontend` |
| **CloudFront** | HTTPS và cache toàn cầu trước S3 | Một distribution, dùng OAC | `BorrowitFrontend` |
| **ECS trên Fargate** | Tiến trình API | 1 task, 0.25 vCPU / 0.5 GB | `BorrowitApp` |
| **Application Load Balancer** | Điểm vào ổn định, health check | Internet-facing, cổng 80 | `BorrowitApp` |
| **CloudWatch** | Log, metric, dashboard, 6 alarm | Giữ log 1 tuần | `BorrowitApp` |
| **SNS** | Gửi cảnh báo | Một topic, subscription email | `BorrowitApp` |

#### Đường đi của request

Ba đường, và cách kiểm tra thiết kế dễ nhất là lần theo từng đường:

1. **Nội dung tĩnh.** Browser → CloudFront → S3. Bucket vẫn private; chỉ
   distribution đọc được, thông qua Origin Access Control. Không chạm tới VPC.
2. **Gọi API.** Browser → ALB cổng 80 → target group → Fargate task cổng 3456 →
   RDS cổng 5432. Mỗi chặng được cho phép bởi đúng một rule security group, và mỗi
   rule tham chiếu tới *security group phía trước* thay vì một dải CIDR, nên vẫn
   đúng khi task bị thay thế.
3. **Khởi động.** Task kéo image từ ECR, đọc mật khẩu từ Secrets Manager, rồi bắt
   đầu đẩy log lên CloudWatch — mỗi luồng đều đi qua một interface endpoint, nên
   không luồng nào rời khỏi VPC. Task vẫn giữ public IP, vì bỏ nó đi đồng nghĩa bất
   kỳ lời gọi AWS nào không có endpoint sẽ treo âm thầm lúc khởi động. Chính sự kết
   hợp đó — public IP nhưng traffic AWS đi riêng — là quyết định gây tranh cãi nhất
   của thiết kế, và được mổ xẻ đầy đủ ở
   [4.3.3](../../4-Workshop/4.3-Architecture/4.3.3-networking/).

#### Cấu trúc stack

Bốn stack, với phụ thuộc chỉ đi theo **một chiều**:

{{<mermaid align="center">}}
graph TD
  F["BorrowitFoundation - VPC, SG, ECR"]
  D["BorrowitData - RDS, Secrets"]
  W["BorrowitFrontend - S3, CloudFront"]
  A["BorrowitApp - ALB, Fargate, CloudWatch, SNS"]
  F --> D
  F --> A
  D --> A
  W --> A
{{</mermaid>}}

Không có gì import từ `BorrowitApp`. Đó là lựa chọn cấu trúc có chủ đích, không
phải ngẫu nhiên do thứ tự: nó khiến **`cdk destroy BorrowitApp` luôn thành công**,
vì CloudFormation không bao giờ phải từ chối do một giá trị export đang được stack
khác dùng. Nhờ vậy tầng đắt tiền nhất cũng chính là tầng bỏ đi được — xem
[2.6](../2.6-Budget/).

#### Tóm tắt các quyết định thiết kế

Mỗi dòng là một quyết định có phương án rẻ hơn hoặc an toàn hơn đã bị loại. Lập
luận đầy đủ, kèm cái giá của từng lựa chọn, nằm ở
[4.3](../../4-Workshop/4.3-Architecture/).

| Tầng | Đề xuất | Phương án thay thế chính | Yếu tố quyết định |
|---|---|---|---|
| Compute | ECS trên Fargate | Lambda, App Runner, EC2 | Tiến trình chạy dài kèm connection pool tới DB |
| Cơ sở dữ liệu | RDS PostgreSQL | Aurora Serverless v2, DynamoDB | Schema quan hệ có sẵn; chi phí cố định dễ đoán |
| Kết nối ra ngoài | Subnet public, không NAT, bật interface endpoint | NAT Gateway | NAT tốn ~$33/tháng và phục vụ *toàn bộ* egress; endpoint tốn ~$36 nhưng chỉ phủ đúng những lời gọi AWS cần thiết |
| Cách ly | Security group | Private subnet | Chi phí; chấp nhận phòng thủ nhiều lớp mỏng hơn |
| Phục vụ nội dung tĩnh | S3 + CloudFront | Amplify Hosting, phục vụ từ ALB | Chi phí, và giữ bucket ở chế độ private |
| Secret | Secrets Manager | SSM Parameter Store | RDS tự sinh và lưu mật khẩu |
| Giám sát | CloudWatch + SNS | Container Insights, X-Ray | Đủ tín hiệu với ~$3.60/tháng |

#### Điểm yếu đã biết của thiết kế

Nêu ngay từ lúc đề xuất, thay vì để lộ ra khi chấm:

+ **Một task là một điểm chết đơn lẻ.** ALB không có gì để chuyển sang. Chỉ giảm
  nhẹ được nhờ ECS khởi động lại task, mất 1–2 phút.
+ **ALB tốn hơn cả phần compute nó cân bằng** — ~$17 so với ~$9. Ở quy mô này, đây
  là dòng khó biện minh nhất trên hoá đơn, và [2.6](../2.6-Budget/) nói thẳng điều
  đó.
+ **Traffic API đi HTTP chứ không phải HTTPS**, vì không có domain để xác thực
  chứng chỉ. Chấp nhận được với bản demo; không chấp nhận được với người dùng thật.
+ **Task có public IP**, đánh đổi phòng thủ nhiều lớp lấy khoản tiết kiệm NAT, và
  tốn ~$3.60/tháng cho mỗi địa chỉ. Interface endpoint thu hẹp đáng kể điểm yếu
  này — traffic tới ECR, Secrets Manager, CloudWatch và SSM không bao giờ đi đường
  công cộng — nhưng **không** bỏ được public IP, thứ vẫn bắt buộc phải có.
+ **Endpoint là dòng lớn thứ hai trên hoá đơn**, ~$36/tháng trên nền $160 credit,
  và chúng nằm trong `BorrowitFoundation` — stack không bao giờ bị xoá. Khác với
  ALB, khoản này vẫn tiếp tục phát sinh ngay cả khi tầng ứng dụng đã được dỡ bỏ —
  xem [2.6](../2.6-Budget/).

<!-- TODO(prose): sau khi dựng xong, quay lại thêm một đoạn ngắn "đã thay đổi những
     gì". Một thiết kế sống sót qua mười hai tuần va chạm với thực tế mà không sứt
     mẻ gì thì nhiều khả năng là chưa bị thử thách đủ. -->
