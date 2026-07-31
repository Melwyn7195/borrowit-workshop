---
title : "Tổng quan"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 3.1. </b> "
---

#### Workload

Hệ thống là một sàn cho thuê đồ: một REST API trên nền cơ sở dữ liệu quan hệ, kèm
client chạy trên trình duyệt và ảnh do người dùng tải lên. Diễn giải sang ngôn ngữ
AWS thì có bốn yêu cầu, và toàn bộ kiến trúc suy ra từ đó:

| Yêu cầu | Cần gì từ AWS |
|---|---|
| Phục vụ request HTTP liên tục | Compute kèm điểm vào ổn định |
| Lưu dữ liệu quan hệ bền vững | Cơ sở dữ liệu được quản lý, truy cập riêng tư |
| Lưu và phục vụ ảnh người dùng | Lưu trữ đối tượng kèm tầng cache |
| Giữ thông tin đăng nhập ngoài mã nguồn | Kho secret tích hợp với tầng compute |

<!-- TODO(prose): một đoạn ngắn về việc ứng dụng làm gì. Giữ cho ngắn gọn — phần
     còn lại của workshop nói về AWS, người đọc không cần mô hình nghiệp vụ để
     theo dõi. -->

#### Sẽ dựng những gì

Mười một dịch vụ AWS, định nghĩa hoàn toàn bằng CDK TypeScript và deploy từ máy cá
nhân:

**VPC** · **ECS trên Fargate** · **Application Load Balancer** · **RDS PostgreSQL** ·
**Secrets Manager** · **ECR** · **S3** · **CloudFront** · **CloudWatch** ·
**SNS** · **IAM**

Hệ thống hoàn chỉnh tốn khoảng **$88/tháng**, và workshop chỉ rõ từng đồng đi về
đâu cũng như phần nào có thể tắt đi. Riêng năm interface VPC endpoint chiếm ~$36
trong số đó — chúng giữ traffic API của AWS ngoài internet công cộng, và vẫn tính
tiền kể cả khi ứng dụng đã bị gỡ.

#### Workshop này nói về điều gì

Trọng tâm là **vì sao chọn từng dịch vụ và vận hành kết quả ra sao** — không phải
mã nguồn ứng dụng. Đọc xong, người đọc phải trả lời được câu "vì sao Fargate mà
không phải Lambda?" kèm con số chi phí và một đánh đổi cụ thể, đồng thời biết cách
xác định hệ thống có khoẻ hay không mà không cần mở ứng dụng.

| Phần | Trả lời câu hỏi |
|---|---|
| [3.2 Chuẩn bị](../3.2-Prerequisites/) | Cần có sẵn những gì trước khi deploy |
| [3.3 Mô tả kiến trúc](../3.3-Architecture/) | Hệ thống trông ra sao, và vì sao chọn các dịch vụ này |
| 3.4 – 3.8 | Dựng từng tầng thế nào, theo đúng thứ tự deploy |
| [3.11 Dọn dẹp](../3.11-Cleanup/) | Xoá gì, theo thứ tự nào, và giữ lại những gì |

#### Không bao gồm

+ Mã nguồn ứng dụng. API và web client được xem như hiện vật cần triển khai.
+ CI/CD. Việc deploy chạy từ máy cá nhân; tự động hoá là hướng phát triển sau.
+ Tên miền riêng và TLS **trên load balancer**. Trình duyệt luôn nói HTTPS với
  CloudFront; chặng từ edge tới ALB thì không, và ALB vẫn gọi thẳng được qua HTTP
  — xem [3.7](../3.7-Delivery/) để biết cái giá của điều đó và cách khắc phục.
