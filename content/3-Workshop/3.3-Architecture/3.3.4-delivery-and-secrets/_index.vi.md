---
title : "Phân phối, lưu trữ và secret"
date : 2026-07-28
weight : 4
chapter : false
pre : " <b> 3.3.4 </b> "
---

Ba quyết định nhỏ hơn, mỗi quyết định đều có phương án thay thế hợp lý.

## Phân phối nội dung tĩnh

#### Các phương án

| Phương án | Chi phí/tháng | Ghi chú |
|---|---|---|
| **S3 + CloudFront** | ~$0.50 | Cache tại edge toàn cầu; bucket vẫn private |
| **AWS Amplify Hosting** | ~$0.15 + phút build | Đơn giản nhất; gắn hosting với quy trình Git |
| **Phục vụ từ ALB** | Không tốn thêm | Dồn tải lên API; mất hoàn toàn khả năng cache |

#### Quyết định

**Amazon S3 kết hợp Amazon CloudFront.** Ứng dụng là một bundle single-page đã
biên dịch cộng với ảnh người dùng tải lên — các đối tượng tĩnh hưởng lợi từ việc
cache ở edge gần người dùng Đông Nam Á.

Phương án phục vụ từ ALB bị loại vì nó đẩy traffic tài nguyên tĩnh qua Fargate
task, tiêu tốn phần compute bạn đang trả tiền và mất hoàn toàn khả năng cache.

Amplify Hosting là lựa chọn sát sao hơn: rẻ hơn và lo luôn khâu build. Nó bị loại
vì che đi phần cấu hình S3 và CloudFront vốn là thứ workshop này tồn tại để trình
bày, và vì nó giả định một pipeline build gắn với Git mà ở đây không có.

#### Giữ bucket ở chế độ private

Cả hai bucket đều dùng `BlockPublicAccess.BLOCK_ALL`. CloudFront truy cập chúng qua
**Origin Access Control**, cơ chế ký request tới S3 bằng SigV4; bucket policy chỉ
tin tưởng đúng distribution đó.

Phương án phổ biến hơn — bật S3 static website hosting kèm policy cho phép đọc công
khai — khiến bucket ai đoán trúng tên cũng đọc được. OAC cho phép đọc công khai
*thông qua CloudFront* mà không cho đọc công khai *chính bucket*.

Dùng `PRICE_CLASS_200`: bao gồm Đông Nam Á và bỏ qua các edge location đắt nhất ở
Nam Mỹ và châu Đại Dương.

#### Một distribution, ba origin

Cùng distribution đó cũng đứng trước **API**, trên `/api/*`, `/health` và
`/api-docs*`. Đây không nằm trong quyết định phân phối ban đầu — nó bị ép ra về
sau bởi hai ràng buộc chỉ lộ diện khi frontend thật sự được deploy: một trang
HTTPS không thể gọi load balancer vốn chỉ có HTTP, và một cookie phiên
`sameSite=strict` không sống sót qua một API host khác site.

Điều này đáng nhắc ở đây vì nó đổi luôn *vai trò* của CloudFront trong kiến trúc:
không chỉ là một lớp cache, mà là thứ làm cho API trở thành same-origin và loại
bỏ nhu cầu CORS. [3.7](../../3.7-Delivery/) sẽ dựng phần đó.

## Lưu trữ secret

#### Các phương án

| Phương án | Chi phí | Tích hợp RDS | Xoay vòng |
|---|---|---|---|
| **AWS Secrets Manager** | $0.40/secret/tháng | **Sẵn có** — RDS sinh và lưu | Có sẵn |
| **SSM Parameter Store** (chuẩn) | Miễn phí | Thủ công | Thủ công |
| **Biến môi trường** | Miễn phí | Không | Không |

#### Quyết định

**AWS Secrets Manager**, chọn vì một lý do cụ thể: `rds.Credentials.fromGeneratedSecret()`
khiến mật khẩu do RDS sinh ra và ghi thẳng vào Secrets Manager. **Không người nào
từng nhìn thấy nó, và nó không bao giờ tồn tại bên ngoài AWS.**

Parameter Store miễn phí và cũng dùng được, nhưng bạn phải tự sinh mật khẩu rồi đặt
nó ở đâu đó để truyền vào — đúng cái tình huống hỏng mà thiết kế này muốn loại bỏ.

$0.40/tháng là mức giá hợp lý để xoá bỏ nguyên một nhóm nguy cơ lộ thông tin đăng nhập.

#### Hai secret, sinh ra theo hai cách khác nhau

| Secret | Sinh bởi | Nằm ở | Sinh lại khi |
|---|---|---|---|
| Thông tin đăng nhập cơ sở dữ liệu | **RDS**, qua `fromGeneratedSecret()` | `BorrowitData` | `BorrowitData` được tạo lại |
| Khoá ký JWT | **CloudFormation**, qua `generateSecretString` | `BorrowitApp` | `BorrowitApp` được tạo lại |

API ký cookie phiên bằng một khoá mà không có dịch vụ nào sinh hộ theo kiểu RDS
sinh mật khẩu cơ sở dữ liệu — nên `BorrowitApp` tự tạo lấy một khoá
([3.6.2](../../3.6-Compute/3.6.2-task-and-service/)). Nguyên tắc thì vẫn như
nhau: **giá trị được sinh bên trong AWS và không người nào từng nhìn thấy nó.**
Cách còn lại — tự nghĩ ra một khoá ký rồi dán vào `cdk.json` — đặt nó vào lịch sử
git vĩnh viễn.

Vị trí đặt secret đi theo vòng đời. Khoá của cơ sở dữ liệu thuộc về cơ sở dữ
liệu; khoá ký thuộc về API, và việc mất nó khi `BorrowitApp` bị xoá không gây hậu
quả nào nặng hơn là mọi người phải đăng nhập lại.

#### Giá trị đi tới container bằng cách nào

Khác biệt giữa `environment` và `secrets` trong task definition của ECS chính là
mấu chốt:

+ Giá trị trong `environment` được lưu **nguyên văn** trong task definition. Bất kỳ
  ai có quyền `ecs:DescribeTaskDefinition` đều đọc được.
+ Giá trị trong `secrets` chỉ lưu **ARN**. ECS phân giải lúc task khởi động bằng
  execution role, nên giá trị thật không xuất hiện trong task definition, trên
  console, hay trong template CloudFormation.

```typescript
secrets: {
  DB_HOST: ecs.Secret.fromSecretsManager(dbSecret, 'host'),
  DB_PASSWORD: ecs.Secret.fromSecretsManager(dbSecret, 'password'),
  // ...
},
```

<!-- TODO(prose): xoay vòng mật khẩu tự động làm được bằng Lambda và ở đây không
     bật. Hãy nói rõ lý do — phạm vi và chi phí — thay vì bỏ qua. Người chấm rất có
     thể sẽ hỏi. -->

## Container registry

**Amazon ECR**, thay vì Docker Hub hay GitHub Container Registry, vì ba lý do:

+ **Xác thực bằng IAM** — không phải lưu thông tin đăng nhập registry ở đâu cả.
+ **Pull cùng region** — task kéo image từ `ap-southeast-1`, nên nhanh và không tốn
  phí truyền dữ liệu liên vùng.
+ **`imageScanOnPush`** — quét lỗ hổng cơ bản miễn phí ở mỗi lần push.

Repository nằm trong `BorrowitFoundation` chứ không phải `BorrowitApp`, để khi xoá
tầng ứng dụng thì không mất các image đã push. Một lifecycle rule giới hạn dung
lượng ở mười image.

{{% notice note %}}
**Tổng chi phí mọi thứ trong trang này:** khoảng **$1,50/tháng** — lưu trữ và
request S3, truyền dữ liệu CloudFront ở mức thấp, **hai** secret trong Secrets
Manager với giá $0,40 mỗi cái, và dung lượng ECR cho mười image nhỏ.
{{% /notice %}}
