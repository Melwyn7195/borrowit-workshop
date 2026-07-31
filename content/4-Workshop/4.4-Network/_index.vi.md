---
title : "Nền tảng mạng"
date : 2026-07-28
weight : 4
chapter : false
pre : " <b> 4.4. </b> "
---

`BorrowitFoundation` tạo VPC, bốn security group, một S3 gateway endpoint, năm
interface VPC endpoint và ECR repository. Mọi thứ trong stack này đều miễn phí hoặc
chỉ vài cent **trừ các interface endpoint**, vốn tốn ~$36/tháng và khiến đây thành
stack đắt thứ hai của dự án. Application stack sẽ được xoá đi dựng lại nhiều lần
trên nền của nó, nên stack này giữ nguyên lâu dài — và khoản phí kia cũng vậy.

Lập luận cho thiết kế không NAT nằm ở
[4.3.3](../4.3-Architecture/4.3.3-networking/); trang này thực thi nó.

#### VPC

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

Bốn subnet trên hai Availability Zone:

| Loại subnet | Chứa gì | Đường ra internet |
|---|---|---|
| `PUBLIC` ×2 | ALB, Fargate task | Qua internet gateway |
| `PRIVATE_ISOLATED` ×2 | RDS instance | **Không có, cả hai chiều** |

`maxAzs: 2` là mức tối thiểu ALB chấp nhận — nó đòi subnet ở ít nhất hai AZ. Đây
cũng là điều giúp phần trình diễn Multi-AZ thực hiện
được mà không phải đổi mạng.

{{% notice warning %}}
Đừng đổi subnet của Fargate sang `PRIVATE_WITH_EGRESS`. Loại subnet đó âm thầm tạo
NAT Gateway, cộng thêm ~$33/tháng mà không có cảnh báo nào lúc synth.
{{% /notice %}}

#### Chuỗi security group

Ba group, mỗi group chỉ nhận traffic chiều vào từ đúng một nguồn:

```typescript
this.albSecurityGroup.addIngressRule(
  ec2.Peer.anyIpv4(), ec2.Port.tcp(80), 'HTTP from the internet');

this.serviceSecurityGroup.addIngressRule(
  this.albSecurityGroup, ec2.Port.tcp(3456), 'Load balancer to app');

this.databaseSecurityGroup.addIngressRule(
  this.serviceSecurityGroup, ec2.Port.tcp(5432), 'App to Postgres');
```

Chuỗi này có nghĩa một gói tin từ internet chỉ tới được database nếu đi qua ALB
trước, rồi tới task. Không có rule nào cho phép bỏ qua một chặng.

Hai chi tiết quan trọng:

+ **Rule tham chiếu security group, không phải dải CIDR.** Khi ECS thay task và
  public IP đổi, rule vẫn đúng. Rule dựa trên CIDR sẽ phải cập nhật mỗi lần thay task.
+ **`allowAllOutbound: false` trên group của database.** Database không có lý do gì
  để chủ động mở kết nối ra ngoài, nên nó không thể.

Một group thứ tư, `EndpointSecurityGroup`, nằm ngoài chuỗi này và bảo vệ các ENI
của interface endpoint. Nó được nói tới ở phần dưới.

#### S3 gateway endpoint

```typescript
this.vpc.addGatewayEndpoint('S3Endpoint', {
  service: ec2.GatewayVpcEndpointAwsService.S3,
});
```

Gateway endpoint **miễn phí** — khác với interface endpoint đã được tính giá ở
[4.3.3](../4.3-Architecture/4.3.3-networking/). Nó thêm một route vào route
table để traffic S3 đi trong mạng AWS thay vì vòng ra internet gateway. Độ trễ thấp
hơn, không tốn phí truyền dữ liệu, không tốn tiền.

Nó cũng gánh việc thật chứ không phải trang trí: ECR lưu layer image trên S3, nên
endpoint này tham gia vào mọi lần task khởi động, không chỉ phục vụ ảnh người dùng.

#### Interface endpoint

Cũng stack này đưa ECR, Secrets Manager, CloudWatch Logs và SSM ra sau PrivateLink,
để traffic đó không bao giờ rời VPC. Chúng được deploy **mọi lần** — không có cờ
nào để bật. Khoản ~$36/tháng ở một AZ được biện luận ở
[4.3.3](../4.3-Architecture/4.3.3-networking/).

```typescript
const endpointSecurityGroup = new ec2.SecurityGroup(this, 'EndpointSecurityGroup', {
  vpc: this.vpc,
  description: 'BorrowIt interface VPC endpoints',
  allowAllOutbound: false,
});
endpointSecurityGroup.addIngressRule(
  this.serviceSecurityGroup, ec2.Port.tcp(443), 'Fargate tasks to AWS service endpoints');

this.vpc.addInterfaceEndpoint('SecretsManagerEndpoint', {
  service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
  subnets,
  securityGroups: [endpointSecurityGroup],
  privateDnsEnabled: true,
});
```

Năm endpoint được tạo theo cách này: `ecr.api`, `ecr.dkr`, `logs`,
`secretsmanager` và `ssmmessages`. Hai lựa chọn đáng học theo:

+ **Chỉ định security group rõ ràng.** Nếu để mặc, `addInterfaceEndpoint` sẽ sinh
  ra một group cho phép toàn bộ dải CIDR của VPC trên cổng 443. Group ở đây chỉ
  nhận 443 từ `ServiceSecurityGroup`.
+ **`privateDnsEnabled: true`.** Đây là thứ khiến hostname vùng của dịch vụ phân
  giải về ENI của endpoint. Không có nó, traffic vẫn đi qua internet gateway và
  tiền endpoint coi như đổ sông.

`subnets` ghim chúng vào một AZ. `npm run endpoints:ha` deploy lại trên cả hai AZ
với giá ~$73/tháng — đáng làm cho phần trình diễn khả năng chịu lỗi, và
không đáng để nguyên như vậy.

{{% notice warning %}}
Đây là dòng duy nhất trong `BorrowitFoundation` tốn tiền thật, và nó vẫn tiếp tục
tốn ngay cả khi `BorrowitApp` đã bị xoá — stack này không bao giờ bị dỡ bỏ. Nếu số
dư credit trở nên eo hẹp, gỡ khối endpoint là khoản tiết kiệm lớn nhất còn lại
ngoài `BorrowitApp`.
{{% /notice %}}

#### ECR repository

```typescript
this.repository = new ecr.Repository(this, 'Repository', {
  repositoryName: 'borrowit-be',
  imageScanOnPush: true,
  lifecycleRules: [{ maxImageCount: 10, description: 'Keep the last 10 images' }],
});
```

Đặt ở stack này chứ không phải `BorrowitApp` là có chủ đích: xoá tầng ứng dụng
không được làm mất những image đã build và push. Lifecycle rule giới hạn dung lượng
ở mười image để các bản build cũ không tích tụ.

#### Triển khai

```powershell
cd infra
npx cdk deploy BorrowitFoundation
```

#### Kiểm chứng

Mở [VPC console](https://ap-southeast-1.console.aws.amazon.com/vpcconsole/home?region=ap-southeast-1#vpcs:)
và xem resource map: bốn subnet, hai AZ, một internet gateway, và **không có NAT
Gateway**.

![Sơ đồ tài nguyên VPC: bốn subnet trên hai AZ, một internet gateway, không có NAT Gateway](/images/4-Workshop/4.4-Network/vpc-resource-map.png?width=100pc)

![Ba security group với luật inbound mở rộng, mỗi nhóm tham chiếu nhóm trước đó thay vì một CIDR](/images/4-Workshop/4.4-Network/security-groups.png?width=100pc)

Xác nhận isolated subnet không có route tới `0.0.0.0/0`:

```powershell
aws ec2 describe-route-tables --region ap-southeast-1 `
  --filters "Name=vpc-id,Values=<VpcId lấy từ output của stack>" `
  --query "RouteTables[].{Name:Tags[?Key=='Name']|[0].Value,Routes:Routes[].DestinationCidrBlock|join(', ',@)}" `
  --output table
```

Chỉ hai route table của public subnet mới có default route:

```
BorrowitFoundation/Vpc/isolatedSubnet1    10.0.0.0/16
BorrowitFoundation/Vpc/isolatedSubnet2    10.0.0.0/16
BorrowitFoundation/Vpc/publicSubnet1      10.0.0.0/16, 0.0.0.0/0
BorrowitFoundation/Vpc/publicSubnet2      10.0.0.0/16, 0.0.0.0/0
```

Isolated subnet chỉ định tuyến trong nội bộ VPC. Không có đường từ database ra
internet, cũng không có đường ngược lại — đó là lý do việc không dùng NAT Gateway
là một quyết định thiết kế chứ không phải một thiếu sót.

Liệt kê các endpoint:

```powershell
aws ec2 describe-vpc-endpoints --region ap-southeast-1 `
  --query "VpcEndpoints[].{Service:ServiceName,Type:VpcEndpointType,DNS:PrivateDnsEnabled}" `
  --output table
```

Kết quả mong đợi là sáu dòng: S3 gateway endpoint cộng năm interface endpoint.
**Năm interface endpoint** phải có `PrivateDnsEnabled` bằng true — nếu cột đó là
`false` thì endpoint đang bị tính tiền mà không ai đi qua.

Riêng S3 gateway endpoint hiển thị `False`, và đó là đúng chứ không phải lỗi.
Gateway endpoint không hoạt động dựa trên DNS: nó thêm một route theo prefix list
vào route table của subnet, nên không có hostname riêng nào để bật. Chỉ interface
endpoint mới có `PrivateDnsEnabled` đáng để đọc.

<!-- SCREENSHOT: /images/4-Workshop/4.4-Network/vpc-endpoints.png
     Danh sách endpoint với private DNS đã bật. -->

{{% notice note %}}
**Chi phí:** khoảng **$36/tháng**, gần như toàn bộ là năm interface endpoint. VPC,
subnet, route table, security group và gateway endpoint đều miễn phí, dung lượng
ECR cho mười image nhỏ chỉ vài cent. Điều đó khiến đây là stack đắt thứ hai trong
dự án.
{{% /notice %}}
