---
title : "Image container và registry"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 4.6.1 </b> "
---

`BorrowitApp` sẽ không deploy được cho tới khi ECR có image, nên bước này làm trước.

Dockerfile của ứng dụng không phải trọng tâm ở đây — điều quan trọng là image tới
ECR bằng cách nào, ECS được cấp quyền pull ra sao, và cách đặt phiên bản ảnh hưởng
thế nào tới khả năng quay lui.

#### Xác thực Docker với ECR

ECR dùng **IAM** thay vì thông tin đăng nhập registry lưu sẵn. `get-login-password`
đổi định danh AWS của bạn lấy một token ngắn hạn:

```powershell
$env:ACCOUNT  = (aws sts get-caller-identity --query Account --output text)
$env:REGION   = "ap-southeast-1"
$env:REGISTRY = "$env:ACCOUNT.dkr.ecr.$env:REGION.amazonaws.com"

aws ecr get-login-password --region $env:REGION | docker login --username AWS --password-stdin $env:REGISTRY
```

Kết quả mong đợi: `Login Succeeded`.

Token hết hạn sau 12 giờ. Không có thứ gì dài hạn được lưu trên máy, đây là lý do
chính chọn ECR thay vì Docker Hub ở
[4.3.4](../../4.3-Architecture/4.3.4-delivery-and-secrets/).

#### Build đúng kiến trúc

```powershell
cd ..\Renting-Online-Backend-main
docker build --platform linux/amd64 -t borrowit-be:v1 .
```

{{% notice warning %}}
`--platform linux/amd64` là bắt buộc. Task definition của Fargate nhắm kiến trúc
X86_64. Image build trên máy ARM mà thiếu cờ này vẫn push thành công rồi chết lúc
khởi động task với lỗi `exec format error` — lỗi xuất hiện lúc chạy, không phải lúc
build hay lúc deploy.
{{% /notice %}}

<!-- TODO(prose): Fargate chạy được ARM64 gốc qua runtimePlatform, rẻ hơn X86_64.
     Nếu bạn đã cân nhắc và vẫn giữ X86_64 thì hãy nói lý do. -->

#### Gắn tag và push

```powershell
docker tag borrowit-be:v1 "$env:REGISTRY/borrowit-be:v1"
docker push "$env:REGISTRY/borrowit-be:v1"
```

{{% notice note %}}
Hãy dùng tag phiên bản thật thay vì `latest`. Với `latest` thì không có cách nào
quay lại image đã chạy tốt, vì tag không còn trỏ tới nó. Stack đọc tag từ context:
`npx cdk deploy BorrowitApp -c imageTag=v2`.
{{% /notice %}}

#### ECS được cấp quyền pull như thế nào

Có hai IAM role tham gia, và nhầm lẫn giữa chúng là nguyên nhân lỗi phổ biến:

| Role | Ai dùng | Cần quyền gì |
|---|---|---|
| **Task execution role** | ECS agent, trước khi container khởi động | Pull từ ECR, đọc Secrets Manager, ghi CloudWatch Logs |
| **Task role** | Mã ứng dụng, trong lúc chạy | Các API AWS mà ứng dụng gọi — ở đây là `s3:PutObject` lên uploads bucket |

```typescript
props.uploadsBucket.grantPut(taskDefinition.taskRole);
```

CDK tự tạo và cấp quyền cho execution role. Task role khởi đầu rỗng và chỉ được cấp
đúng một quyền — ứng dụng tải ảnh lên S3 và không làm gì khác với AWS.

<!-- TODO(prose): đáng viết một câu — điều này thay thế access key tĩnh trong ứng
     dụng bằng một role được assume lúc chạy, nên không có thông tin đăng nhập AWS
     nào nằm trong mã nguồn. -->

#### Kiểm chứng trên ECR

![Repository ECR liệt kê image đã push kèm tag, dung lượng và thời điểm push](/images/4-Workshop/4.6-Compute/4.6.1-image-and-registry/ecr-image.png?width=100pc)

`imageScanOnPush: true` tự động chạy một bản quét lỗ hổng cơ bản.

<!-- TODO(prose): báo cáo đúng những gì bản quét tìm ra. Nếu base image có lỗ hổng
     thì nói rõ và nói hướng xử lý — build lại trên base tag mới hơn thường là toàn
     bộ câu trả lời. Đừng khẳng định quét sạch khi thực tế không phải. -->
