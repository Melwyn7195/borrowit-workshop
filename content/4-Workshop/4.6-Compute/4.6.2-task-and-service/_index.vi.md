---
title : "Task definition và service"
date : 2026-07-28
weight : 2
chapter : false
pre : " <b> 4.6.2 </b> "
---

Task definition mô tả *chạy cái gì*; service mô tả *bao nhiêu bản* và *ở đâu*, đồng
thời duy trì điều đó theo thời gian.

#### Task definition

```typescript
const taskDefinition = new ecs.FargateTaskDefinition(this, 'TaskDefinition', {
  cpu: 256,
  memoryLimitMiB: 512,
});
```

0.25 vCPU và 512 MiB — tổ hợp Fargate nhỏ nhất hợp lệ, khoảng **$9/tháng** khi chạy
liên tục. Fargate chỉ chấp nhận một số cặp CPU/bộ nhớ nhất định; 256 CPU unit cho
phép 512, 1024 hoặc 2048 MiB, không có lựa chọn nào khác.

#### Cấu hình vào một chỗ, secret vào chỗ khác

```typescript
environment: {
  PORT: '3456',
  NODE_ENV: 'production',
  AWS_REGION: this.region,
  AWS_S3_BUCKET: props.uploadsBucket.bucketName,
  ASSET_BASE_URL: props.distributionUrl,
  FRONTEND_URL: props.distributionUrl,
},
secrets: {
  DB_HOST: ecs.Secret.fromSecretsManager(dbSecret, 'host'),
  DB_PORT: ecs.Secret.fromSecretsManager(dbSecret, 'port'),
  DB_USER: ecs.Secret.fromSecretsManager(dbSecret, 'username'),
  DB_PASSWORD: ecs.Secret.fromSecretsManager(dbSecret, 'password'),
  DB_NAME: ecs.Secret.fromSecretsManager(dbSecret, 'dbname'),
  JWT_SECRET: ecs.Secret.fromSecretsManager(jwtSecret, 'jwtSecret'),
},
```

Cấu hình không nhạy cảm đặt trong `environment`; thứ gì bí mật đặt trong `secrets`,
nơi chỉ lưu ARN. Xem [4.3.4](../../4.3-Architecture/4.3.4-delivery-and-secrets/)
để hiểu vì sao khác biệt đó quan trọng.

Ba giá trị đáng chú ý:

+ **`AWS_REGION` phải set tường minh.** Khác với EC2, Fargate task không tự cấp
  region cho SDK, và luồng upload đọc trực tiếp biến này.
+ **`ASSET_BASE_URL` và `FRONTEND_URL` đến từ frontend stack.** Đây chính là phần
  import khiến `BorrowitApp` phụ thuộc `BorrowitFrontend` — và là lý do không gì
  được phép phụ thuộc ngược lại vào `BorrowitApp`.
+ **`JWT_SECRET` là secret duy nhất do chính stack này sinh ra.** Mật khẩu cơ sở
  dữ liệu đến từ RDS; giá trị này không có dịch vụ nào sinh hộ, nên AppStack tự
  tạo nó.

#### Secret thứ hai

```typescript
const jwtSecret = new secretsmanager.Secret(this, 'JwtSecret', {
  description: 'BorrowIt API token signing key',
  generateSecretString: {
    secretStringTemplate: JSON.stringify({}),
    generateStringKey: 'jwtSecret',
    passwordLength: 64,
    excludePunctuation: true,
  },
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

Giá trị này ký cookie phiên. Thiếu nó thì mọi lần đăng nhập trả về **500** kèm
`secretOrPrivateKey must have a value` — `jwt.sign` từ chối một khoá undefined
chứ không tự thay thế bằng gì cả.

Nó do CloudFormation sinh ra, vì đúng lý do mà mật khẩu cơ sở dữ liệu do RDS sinh
ra: một giá trị ghi vào `cdk.json` hay `.env` là một giá trị nằm trong lịch sử
git. `excludePunctuation` giữ nó ở các ký tự không bao giờ cần escape ở bất kỳ
khâu nào, vì nó được đọc lại dưới dạng biến môi trường.

{{% notice note %}}
Xoá rồi deploy lại `BorrowitApp` sẽ sinh lại secret này, làm vô hiệu mọi token đã
cấp. Trên thực tế điều đó nghĩa là mọi người phải đăng nhập lại — nên biết trước
khi bạn xoá stack giữa lúc đang trình diễn.
{{% /notice %}}

`JWT_REFRESH_SECRET` cố ý **không** được set: `userController.js` vốn đã quay về
dùng `JWT_SECRET` cho refresh token, và thêm một secret nữa là thêm $0,40/tháng.
Tách riêng chúng là một bước siết bảo mật, không phải một bản vá lỗi.

#### Health check của container

```typescript
healthCheck: {
  command: ['CMD-SHELL', "node -e \"...http://127.0.0.1:3456/health/live...\""],
  interval: cdk.Duration.seconds(30),
  timeout: cdk.Duration.seconds(5),
  retries: 3,
  startPeriod: cdk.Duration.seconds(15),
},
```

Đây là kiểm tra **liveness**: tiến trình Node còn sống không? Nó cố ý **không** chạm
tới database. Kiểm tra readiness có chạm database nằm ở load balancer — xem
[3.6.3](../4.6.3-load-balancer/).

{{% notice note %}}
ECS **bỏ qua** chỉ thị `HEALTHCHECK` trong Dockerfile. Nó phải được khai báo lại
trong task definition. Bản trong Dockerfile chỉ giữ để `docker run` ở local hành xử
giống hệt.
{{% /notice %}}

#### Service

```typescript
const service = new ecs.FargateService(this, 'Service', {
  cluster,
  taskDefinition,
  desiredCount,
  assignPublicIp: true,
  vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC },
  securityGroups: [props.serviceSecurityGroup],
  circuitBreaker: { rollback: true },
  enableExecuteCommand: true,
  minHealthyPercent: 50,
  maxHealthyPercent: 200,
});
```

| Thiết lập | Tác dụng |
|---|---|
| `assignPublicIp` + subnet `PUBLIC` | Truy cập chiều ra mà không cần NAT — [4.3.3](../../4.3-Architecture/4.3.3-networking/) |
| `securityGroups` | Chiều vào chỉ nhận từ load balancer qua cổng 3456 |
| `circuitBreaker: { rollback: true }` | Bản deploy không qua health check sẽ tự rollback |
| `enableExecuteCommand` | Mở shell trong task đang chạy — [3.6.4](../4.6.4-database-init/), [3.8.4](../../4.8-Observability/4.8.4-debugging/) |
| `minHealthyPercent: 50` | Với một task, ECS dừng task cũ trước khi khởi động task mới |

#### An toàn khi deploy mà không phải trả thêm tiền

`minHealthyPercent: 50` cùng `desiredCount: 1` cho phép ECS hạ về không task healthy
trong lúc deploy — nhờ đó mỗi lần deploy không bao giờ chạy hai task cùng lúc và
không bao giờ tốn gấp đôi.

Đánh đổi là **vài giây gián đoạn mỗi lần deploy**. Với `desiredCount: 2`, bạn sẽ đặt
`minHealthyPercent: 100` để deploy cuốn chiếu không gián đoạn và trả tiền cho task
thứ hai vĩnh viễn.

<!-- TODO(prose): nói rõ bạn chọn cách nào và vì sao. Với ngân sách này, vài giây
     gián đoạn mỗi lần deploy là đổi chác dễ chấp nhận để giảm nửa hoá đơn compute —
     nhưng đó vẫn là một đánh đổi, và gọi tên nó cho thấy bạn hiểu thiết lập chứ
     không phải chép lại. -->

Circuit breaker là lưới an toàn khiến điều này chấp nhận được: nếu task mới không
bao giờ qua được health check, ECS dừng đợt triển khai và tự khôi phục task
definition trước đó, thay vì để dịch vụ nằm chết.

#### Triển khai

```powershell
cd ..\infra
npx cdk deploy BorrowitApp -c imageTag=v1 -c alarmEmail=you@example.com
```

Mất khoảng **5 phút**.

`-c alarmEmail` ở đây là tuỳ chọn và được trình bày kỹ ở
[3.8.3](../../4.8-Observability/4.8.3-alarms/) — truyền ngay bây giờ thì đỡ được
một lần deploy năm phút nữa. `-c imageTag` thì không tuỳ chọn: mặc định của nó là
`latest`, còn 3.6.1 đã push `v1`.

{{% notice note %}}
Lệnh deploy này sẽ kéo theo `BorrowitFrontend` nếu bạn chưa deploy stack đó, vì
task definition đọc tên uploads bucket và distribution URL từ nó. Khi chạy xong,
hãy chạy `npm run wire` để CloudFront biết hostname của load balancer mới —
[3.7](../../4.7-Delivery/).
{{% /notice %}}

![Console ECS: cluster, service, số task đang chạy và trạng thái triển khai](/images/4-Workshop/4.6-Compute/4.6.2-task-and-service/ecs-service.png?width=100pc)
![JSON của task definition cho thấy DB_PASSWORD ánh xạ tới ARN của secret chứ không phải giá trị thật](/images/4-Workshop/4.6-Compute/4.6.2-task-and-service/task-definition-secrets.png?width=100pc)

{{% notice note %}}
Ở bước này target sẽ báo **unhealthy**, và đó là điều bình thường — kiểm tra
readiness chạy `SELECT 1` trên một database chưa có bảng nào. Vấn đề được xử lý ở
[3.6.4](../4.6.4-database-init/).
{{% /notice %}}
