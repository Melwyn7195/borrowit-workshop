---
title : "Tầng dữ liệu"
date : 2026-07-28
weight : 5
chapter : false
pre : " <b> 3.5. </b> "
---

`BorrowitData` tạo RDS instance và, như một hệ quả kèm theo, secret trong Secrets
Manager chứa thông tin đăng nhập của nó. Lập luận về cấu hình nằm ở
[3.3.2](../3.3-Architecture/3.3.2-database/).

Đây là stack chứa trạng thái, nên cũng là stack không được xoá tuỳ tiện.

#### Instance

```typescript
this.database = new rds.DatabaseInstance(this, 'Database', {
  engine: rds.DatabaseInstanceEngine.postgres({
    version: rds.PostgresEngineVersion.VER_16,
  }),
  vpc: props.vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
  securityGroups: [props.databaseSecurityGroup],
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.BURSTABLE4_GRAVITON,
    ec2.InstanceSize.MICRO,
  ),
  allocatedStorage: 20,
  maxAllocatedStorage: 20,
  storageEncrypted: true,
  multiAz,
  publiclyAccessible: false,
  databaseName: 'borrowit',
  credentials: rds.Credentials.fromGeneratedSecret('borrowit'),
  backupRetention: cdk.Duration.days(1),
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

#### Cách ly, và cái giá phải trả

Subnet `PRIVATE_ISOLATED` cộng với `publiclyAccessible: false` nghĩa là không tồn
tại đường mạng nào từ bên ngoài VPC tới database này. Không gì kết nối vào được, và
bản thân nó cũng không kết nối ra được.

Hệ quả thực tế xuất hiện ngay: **bạn không chạy được `psql` từ máy cá nhân.** Đó là
lý do schema được nạp từ bên trong container đang chạy ở
[3.6.4](../3.6-Compute/3.6.4-database-init/) thay vì từ máy trạm.

<!-- TODO(prose): trình bày như một đánh đổi bạn chấp nhận có ý thức, không phải
     một bất ngờ vấp phải. Các phương án khác — bastion host ~$8/tháng, hoặc mở
     public cho RDS — đều đã được cân nhắc và loại bỏ. -->

#### Thông tin đăng nhập

```typescript
credentials: rds.Credentials.fromGeneratedSecret('borrowit'),
```

RDS sinh mật khẩu ngẫu nhiên lúc tạo và ghi vào AWS Secrets Manager dưới dạng một
tài liệu JSON:

```json
{
  "username": "borrowit",
  "password": "...",
  "engine": "postgres",
  "host": "borrowitdata-....ap-southeast-1.rds.amazonaws.com",
  "port": 5432,
  "dbname": "borrowit"
}
```

CDK không bao giờ in nó ra, không ghi vào `cdk.json`, và nó không xuất hiện trong
template đã synth. Hãy kiểm chứng thay vì tin tưởng:

```powershell
npx cdk synth BorrowitData | Select-String "password"
```

Nếu bạn cần tự xem thông tin kết nối:

```powershell
$env:SECRET_ARN = (aws cloudformation describe-stacks `
  --stack-name BorrowitData `
  --query "Stacks[0].Outputs[?OutputKey=='DbSecretArn'].OutputValue" `
  --output text)

aws secretsmanager get-secret-value --secret-id $env:SECRET_ARN --query SecretString --output text
```

{{% notice warning %}}
Lệnh đó in mật khẩu ra terminal. **Không chụp màn hình kết quả.** Nếu cần ảnh cho
báo cáo, hãy chụp trong console Secrets Manager khi giá trị vẫn đang bị ẩn.
{{% /notice %}}

#### Hành vi khi xoá

```typescript
removalPolicy: cdk.RemovalPolicy.DESTROY,
deleteAutomatedBackups: true,
```

`DESTROY` là có chủ đích và **không** phải lựa chọn an toàn cho production.

`RETAIN` sẽ để lại một instance mồ côi vẫn tính tiền sau khi `cdk destroy` — trên
số dư credit cố định, đó là kết cục tệ hơn việc mất dữ liệu mẫu vốn có script dựng
lại được. Với hệ thống có dữ liệu người dùng thật, `RETAIN` mới đúng và phần chi
phí đó phải chấp nhận.

<!-- TODO(prose): viết lại bằng lời của bạn. Đây là ví dụ hay về một quyết định
     đúng trong bối cảnh này và sai trong bối cảnh chung, đúng loại sắc thái đáng
     thể hiện. -->

#### Triển khai

RDS mất **5–10 phút** để tạo xong.

```powershell
npx cdk deploy BorrowitData
```

#### Kiểm chứng

![Instance RDS: trạng thái Available, class db.t4g.micro, Multi-AZ No](/images/3-Workshop/3.5-Data/rds-instance.png?width=100pc)

![Tab Connectivity & security: Publicly accessible No, DatabaseSecurityGroup đã gắn, subnet group liệt kê các isolated subnet](/images/3-Workshop/3.5-Data/rds-connectivity.png?width=100pc)

{{% notice note %}}
**Chi phí:** ~$16/tháng — `db.t4g.micro` cộng 20 GB dung lượng gp2, trừ vào credit
Free Plan ngay từ giờ đầu tiên. Tài khoản này không có free tier 12 tháng.
{{% /notice %}}
