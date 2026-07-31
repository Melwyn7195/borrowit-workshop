---
title : "Mạng — không dùng NAT Gateway"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 3.3.3 </b> "
---

Đây là quyết định dễ bị chất vấn nhất, nên được trình bày đầy đủ nhất.

#### Vấn đề

Một Fargate task cần truy cập internet **chiều ra** thì mới hoạt động được:

+ kéo image từ Amazon ECR,
+ đọc thông tin đăng nhập database từ AWS Secrets Manager,
+ đẩy log lên Amazon CloudWatch.

Nó **không cần** truy cập chiều vào từ internet — load balancer là thứ duy nhất nên
tới được nó. Câu hỏi là làm sao cấp vế đầu mà không kèm vế sau.

#### Các phương án

| Phương án | Chi phí/tháng | Mức phơi bày chiều vào | Ghi chú |
|---|---|---|---|
| **Private subnet + NAT Gateway** | ~$33 + $0.045/GB | Không — không tồn tại route | Câu trả lời sách vở |
| **Private subnet + VPC interface endpoint** | ~$7.30/endpoint/AZ | Không | Cần 5 endpoint ≈ $73/tháng trên hai AZ |
| **Public subnet + chỉ public IP** | **$0** | Chỉ security group | Rẻ nhất; mọi lời gọi AWS đi đường công cộng |
| **Public subnet + interface endpoint** | ~$36/tháng ở một AZ | Chỉ security group | **Phương án dự án này triển khai** |

#### Vì sao không dùng NAT Gateway

NAT Gateway tốn khoảng **$33/tháng** tại `ap-southeast-1` chưa tính phí xử lý dữ
liệu. Toàn bộ ngân sách compute của dự án khoảng $35/tháng. Thêm NAT sẽ làm chi phí
hệ thống tăng gần gấp đôi để giải quyết một vấn đề mà security group cũng giải
quyết được.

Trên số dư cố định $160 credit, riêng một NAT Gateway đã ngốn gần hết quãng đường
còn lại.

#### Interface endpoint — đã triển khai, và cái giá của nó

Câu trả lời cho bài toán chiều ra mà không cần NAT là **interface endpoint của AWS
PrivateLink** — traffic tới ECR, Secrets Manager, CloudWatch và SSM không bao giờ
rời mạng AWS, và không cần route ra internet cho phần đó.

Chúng được triển khai trong `BorrowitFoundation` như một phần của kiến trúc, không
nằm sau cờ bật/tắt. Đó là một quyết định tốn tiền, nên phép tính cần được nói rõ.
Interface endpoint tính phí **theo từng endpoint, từng Availability Zone, từng
giờ** — $0.01/giờ tại `ap-southeast-1`, tức **$7.30 mỗi endpoint mỗi AZ mỗi
tháng**. Workload này cần năm cái:

| Endpoint | Vì sao task cần nó |
|---|---|
| `ecr.api` | Cấp quyền kéo image |
| `ecr.dkr` | Trả về manifest của image |
| `logs` | Đẩy log container lên CloudWatch |
| `secretsmanager` | Đọc mật khẩu database lúc task khởi động |
| `ssmmessages` | Nền tảng cho `enableExecuteCommand` — `aws ecs execute-command` |

Năm endpoint trên hai AZ là **mười tổ hợp endpoint-AZ, ~$73/tháng** — còn tệ hơn
chính cái NAT Gateway mà nó định thay thế. **Ghim vào một AZ thì còn ~$36/tháng**,
và đó là cấu hình được deploy mặc định:

```powershell
npx cdk deploy BorrowitFoundation     # một AZ,  ~$36/tháng
npm run endpoints:ha                  # cả hai AZ, ~$73/tháng
```

Một AZ là mặc định vì private DNS trả về mọi ENI mà endpoint có, và một lời gọi
xuyên AZ bên trong VPC vẫn chạy — một AZ giảm nửa hoá đơn, đổi lại là phụ thuộc
vào AZ đó còn sống. `-c vpcEndpoints=ha` gỡ bỏ phụ thuộc đó khi phần trình diễn
cần tới.

{{<mermaid align="center">}}
graph LR
  T["Fargate task"]
  VPE["Interface endpoints - ENIs inside the VPC"]
  AWS["ECR, Secrets Manager, CloudWatch, SSM"]
  IGW["Internet gateway"]
  NET["Everything else"]

  T -->|443, private DNS| VPE
  VPE -->|stays on the AWS network| AWS
  T -. no endpoint, so still public .-> IGW
  IGW -.-> NET
{{</mermaid>}}

Nhánh nét đứt là phần mà endpoint **không** phủ. Task vẫn giữ public IP và route
mặc định của nó, nên mọi lời gọi tới thứ không có endpoint vẫn đi ra qua internet
gateway. Điều đã thay đổi là bốn dịch vụ trên đường khởi động và telemetry — những
dịch vụ giữ thông tin đăng nhập, image và log — thì không còn như vậy nữa.

Hai chi tiết quyết định đây là một biện pháp thật hay chỉ là khoản chi vô ích:

+ **`privateDnsEnabled: true`.** Không có nó, hostname vùng của dịch vụ vẫn phân
  giải ra địa chỉ công cộng và traffic vẫn đi qua internet gateway — tức là trả
  tiền cho những endpoint không ai dùng.
+ **Một security group riêng.** Nếu không, `addInterfaceEndpoint` sẽ tự sinh một
  group cho phép toàn bộ dải CIDR của VPC trên cổng 443. Endpoint ở đây chỉ nhận
  443 từ `ServiceSecurityGroup`, nên task là bên gọi duy nhất.

{{% notice warning %}}
Có endpoint **không** cho phép bỏ `assignPublicIp: true`. Bỏ public IP nghĩa
là mọi lời gọi AWS của task đều phải có endpoint tương ứng, và thiếu một cái sẽ
biểu hiện thành task treo lúc khởi động chứ không báo lỗi rõ ràng. Ranh giới bảo
mật vẫn là chuỗi security group, không phải các endpoint.
{{% /notice %}}

{{% notice warning %}}
**Khoản phí này không biến mất khi bạn xoá tầng ứng dụng.** Endpoint nằm trong
`BorrowitFoundation` — stack không bao giờ bị dỡ bỏ — nên `cdk destroy BorrowitApp`
giờ chỉ kéo mức chi hàng tháng xuống ~$53 thay vì ~$17. Đó là cái giá thật của việc
biến chúng thành bắt buộc.
{{% /notice %}}

{{% notice note %}}
**S3 gateway endpoint** là thứ hoàn toàn khác và **luôn** bật — gateway endpoint
**miễn phí**. Nó được thêm ở [3.4](../../3.4-Network/), và nó gánh việc thật chứ
không phải trang trí: layer image của ECR nằm trên S3, nên nó tham gia vào mọi lần
task khởi động.
{{% /notice %}}

#### Quyết định

Fargate task chạy trong **public subnet với `assignPublicIp: true`**, việc cách ly
chiều vào do security group đảm nhiệm, còn traffic AWS chiều ra đi qua **interface
endpoint** nên không bao giờ băng qua internet công cộng.

```typescript
this.vpc = new ec2.Vpc(this, 'Vpc', {
  maxAzs: 2,
  natGateways: 0,
  subnetConfiguration: [
    { name: 'public', subnetType: ec2.SubnetType.PUBLIC, cidrMask: 24 },
    { name: 'isolated', subnetType: ec2.SubnetType.PRIVATE_ISOLATED, cidrMask: 24 },
  ],
});
```

Các security group tạo thành một chuỗi, mỗi group chỉ nhận traffic từ group liền
trước:

| Group | Nhận traffic từ | Cổng |
|---|---|---|
| `AlbSecurityGroup` | `0.0.0.0/0` | 80 |
| `ServiceSecurityGroup` | `AlbSecurityGroup` | 3456 |
| `DatabaseSecurityGroup` | `ServiceSecurityGroup` | 5432 |

Tham chiếu tới **security group** thay vì dải CIDR là điều khiến thiết kế này bền
vững: rule vẫn đúng khi ECS thay task và public IP thay đổi.

#### Có public IP không đồng nghĩa với dịch vụ công khai

Task có một địa chỉ định tuyến được, nhưng không gì trên internet mở được kết nối
tới nó — gói tin chiều vào bị loại bỏ trừ khi xuất phát từ security group của load
balancer.

Điều này được chứng minh chứ không chỉ khẳng định ở
[3.6.3](../../3.6-Compute/3.6.3-load-balancer/), bằng cách curl thẳng vào public IP
của task và quan sát nó timeout.

#### Cái giá của lựa chọn này

+ **Phòng thủ nhiều lớp mỏng đi.** Một lần cấu hình sai security group là task bị
  phơi thẳng ra internet. Ở thiết kế private subnet, cùng sai lầm đó không phơi bày
  gì cả vì không tồn tại route. Đây là mức giảm an toàn thật sự, không phải hình thức.
+ **Mỗi task chiếm một địa chỉ IPv4 công cộng**, thứ mà AWS hiện đã tính phí
  (~$3.60/tháng mỗi địa chỉ) và trở nên đáng kể khi quy mô lớn hơn.
+ **Không qua được kiểm toán tuân thủ** nếu quy định cấm workload có public IP.

<!-- TODO(prose): nói thẳng bạn sẽ làm gì nếu ngân sách rộng hơn. Câu trả lời trung
     thực là private subnet phía sau NAT Gateway. Một workshop trình bày giải pháp
     né ngân sách như thể đó là best practice thì gây hiểu lầm; một workshop gọi
     tên đánh đổi thì đáng tin. -->

#### Khi nào nên dùng thiết kế nào

| | Không NAT (public subnet) | Không NAT + interface endpoint | Có NAT Gateway (private subnet) |
|---|---|---|---|
| Chi phí | $0 | **~$36/tháng, một AZ** | ~$33/tháng + phí xử lý dữ liệu |
| Cách ly | Chỉ security group | Security group; traffic AWS rời khỏi đường công cộng | Security group **và** không có route |
| Phí IPv4 công cộng | Theo từng task | Theo từng task — vẫn bắt buộc có IP | Không có |
| Phù hợp với | Demo, trần ngân sách cứng | Dự án nhỏ nhưng buộc phải giữ traffic AWS đi riêng | Production, hệ thống bị quản lý chặt |

Cột giữa là phương án dự án này triển khai. Cần nói chính xác nó mua được gì và
không mua được gì: thông tin đăng nhập, image và log thôi không băng qua internet
công cộng nữa — đó là một biện pháp thật và chứng minh được. Nhưng task vẫn có địa
chỉ tiếp cận được từ internet ở tầng định tuyến, và chỉ security group ngăn gói tin
tới được nó. Endpoint cải thiện tính bí mật của đường ra; chúng không khôi phục lớp
phòng thủ chiều vào mà private subnet mang lại.

Với workload thật sự không được phép có public IP, câu trả lời đúng vẫn là cột thứ
ba, và không khoản chi endpoint nào thay thế được điều đó.
