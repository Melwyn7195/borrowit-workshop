---
title : "Load balancer và health check"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 4.6.3 </b> "
---

ALB là thành phần tốn tiền nhất trên đường đi của request — khoảng **$17/tháng**,
tính theo giờ bất kể có request nào tới hay không. Chỉ có năm interface VPC
endpoint tốn hơn, mà chúng lại hoàn toàn không nằm trên đường đi của request. Đáng
để hiểu chính xác khoản $17 đó mua được gì.

#### ALB mang lại những gì

| Chức năng | Vì sao quan trọng ở đây |
|---|---|
| Tên DNS ổn định | Fargate task đổi IP mỗi lần thay mới; tên ALB không bao giờ đổi |
| Kiểm tra sức khoẻ | Tự động loại task hỏng khỏi vòng phục vụ |
| Phân phối liên AZ | Bắt buộc cho phần trình diễn Multi-AZ |
| Rút kết nối êm | Cho request đang dở hoàn tất trước khi task bị dừng |
| Metric | Số request, phân vị độ trễ, 4xx/5xx — nền tảng cho [3.8](../../4.8-Observability/) |

<!-- TODO(prose): Network Load Balancer rẻ hơn theo giờ và API Gateway HTTP API còn
     rẻ hơn nữa ở lưu lượng thấp. Nếu bạn đã cân nhắc thì hãy nói vì sao ALB thắng —
     định tuyến theo đường dẫn và bộ metric HTTP phong phú hơn là lý do thường gặp. -->

#### Cấu hình

```typescript
const alb = new elbv2.ApplicationLoadBalancer(this, 'LoadBalancer', {
  vpc: props.vpc,
  internetFacing: true,
  securityGroup: props.albSecurityGroup,
  vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC },
});

const listener = alb.addListener('Listener', { port: 80, open: false });

const targetGroup = listener.addTargets('AppTargets', {
  port: 3456,
  protocol: elbv2.ApplicationProtocol.HTTP,
  targets: [service],
  deregistrationDelay: cdk.Duration.seconds(15),
  healthCheck: {
    path: '/health',
    interval: cdk.Duration.seconds(30),
    timeout: cdk.Duration.seconds(5),
    healthyThresholdCount: 2,
    unhealthyThresholdCount: 3,
  },
});
```

`open: false` trên listener rất quan trọng: nếu thiếu, CDK sẽ tự thêm một ingress
rule vào security group, trùng lặp với rule đã định nghĩa ở [3.4](../../4.4-Network/)
và làm mờ đâu mới là nguồn chân lý.

#### Hai health check trả lời hai câu hỏi khác nhau

Đây là điểm thiết kế quan trọng nhất của tầng compute.

| | Liveness | Readiness |
|---|---|---|
| Thuộc về | ECS task definition | ALB target group |
| Đường dẫn | `/health/live` | `/health` |
| Có chạm database | **Không** | **Có** — chạy `SELECT 1` |
| Khi thất bại | ECS giết và thay task | ALB ngừng gửi traffic tới task |

Kiểm tra ở container hỏi **"tiến trình còn sống không?"**. Nếu thất bại, task bị
giết và thay bằng task mới.

Kiểm tra ở load balancer hỏi **"task này có nên nhận traffic không?"**. Nó chạy một
truy vấn thật, nên task đã mất kết nối database sẽ bị đưa ra khỏi vòng phục vụ mà
không bị huỷ.

**Vì sao việc tách bạch này quan trọng.** Nếu liveness check cũng truy vấn database,
một gián đoạn ngắn của RDS — một lần failover, một trục trặc mạng — sẽ làm liveness
thất bại trên mọi task cùng lúc. ECS giết hết, các task thay thế khởi động lại đúng
vào database đang không sẵn sàng, lại thất bại, và dịch vụ rơi vào vòng lặp khởi
động lại kéo dài hơn hẳn sự cố gốc.

Tách chúng ra nghĩa là sự cố database làm dịch vụ **suy giảm** thay vì **sụp đổ**.
Task vẫn sống, ngừng nhận traffic, và tự phục vụ trở lại khi database hồi phục.

<!-- TODO(prose): đây là điểm đáng giá nhất trong báo cáo để giải thích khi bị chất
     vấn. Hãy viết bằng lời của bạn. Nếu bạn đã đo được hành vi trong bài kiểm thử
     failover, hãy dẫn kết quả về đây. -->

#### Rút kết nối êm

```typescript
deregistrationDelay: cdk.Duration.seconds(15),
```

Giá trị này phải **lớn hơn** thời gian drain của ứng dụng (5 giây). Nếu ngắn hơn,
task sẽ đóng socket trong khi load balancer vẫn đang gửi request tới, tạo ra lỗi
502 ở mỗi lần deploy.

#### Triển khai và kiểm chứng

Lấy URL:

```powershell
$env:API = (aws cloudformation describe-stacks --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" --output text)
curl "$env:API/health"
```

#### Chứng minh mức độ cách ly

Thiết kế security group ở [4.3.3](../../4.3-Architecture/4.3.3-networking/)
khẳng định rằng task có public IP vẫn không truy cập được. Hãy chứng minh:

```powershell
$env:CLUSTER = (aws cloudformation describe-stacks --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='ClusterName'].OutputValue" --output text)
$env:TASK = (aws ecs list-tasks --cluster $env:CLUSTER --query "taskArns[0]" --output text)

$env:ENI = (aws ecs describe-tasks --cluster $env:CLUSTER --tasks $env:TASK `
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" --output text)
$env:TASK_IP = (aws ec2 describe-network-interfaces --network-interface-ids $env:ENI `
  --query "NetworkInterfaces[0].Association.PublicIp" --output text)

# Bắt buộc phải thất bại - security group chỉ chấp nhận ALB qua cổng 3456
curl "http://$($env:TASK_IP):3456/health" --max-time 10
```

{{% notice note %}}
**Chi phí:** ~$17/tháng cho ALB, cộng phí LCU không đáng kể ở mức lưu lượng này. Đây
là thành phần cần bỏ đầu tiên nếu phải làm hệ thống rẻ hơn — và bỏ nó nghĩa là bỏ
luôn điểm vào ổn định, nên không có phương án tiết kiệm một phần.
{{% /notice %}}
