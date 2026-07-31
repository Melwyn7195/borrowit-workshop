---
title: "Workshop"
date: 2026-07-28
weight: 4
chapter: false
pre: " <b> 4. </b> "
---

# Xây dựng và vận hành một nền tảng web trên AWS

#### Tổng quan

Workshop này xây dựng một hệ thống hoàn chỉnh mang dáng dấp production trên AWS
rồi vận hành nó: một cơ sở dữ liệu riêng tư, một API dạng container phía sau load
balancer, một tầng phân phối nội dung toàn cầu, và phần giám sát cần thiết để biết
liệu tất cả có đang hoạt động hay không.

Ứng dụng chạy trên đó — **BorrowIt**, nền tảng cho thuê đồ giữa người dùng — cố ý
không phải là trọng tâm. Ứng dụng chỉ là một workload; điều quan trọng ở đây là
dịch vụ AWS nào gánh nó, **vì sao chọn dịch vụ đó thay vì các lựa chọn khác**,
chúng được ghép nối ra sao, và kết quả được giám sát cũng như trả phí thế nào.

Sau khi hoàn thành, bạn sẽ dựng được:

+ Một **Amazon VPC** trải trên hai Availability Zone, kèm quyết định có ghi chép về
  việc bỏ NAT Gateway và đặt việc cách ly vào security group.
+ **Amazon RDS for PostgreSQL** trong isolated subnet, thông tin đăng nhập được
  sinh vào **AWS Secrets Manager** và inject lúc container khởi động.
+ **Amazon ECS trên AWS Fargate** phía sau **Application Load Balancer**, với
  liveness và readiness tách biệt cùng cơ chế tự rollback khi deploy.
+ **Amazon S3** và **Amazon CloudFront** dùng Origin Access Control.
+ Log, metric, dashboard và bảy alarm của **Amazon CloudWatch** gửi thông báo qua
  **Amazon SNS**.
+ Một mô hình chi phí được kiểm chứng bằng hoá đơn thật, không chỉ dừng ở ước tính.

Toàn bộ được định nghĩa bằng **AWS CDK** (TypeScript). Không có bước bấm console
nào trong quá trình dựng — console chỉ dùng để kiểm chứng những gì code đã tạo ra.

{{% notice note %}}
Mọi tài nguyên đều nằm ở **`ap-southeast-1` (Singapore)**. Region xuất hiện trong
cả URL console lẫn câu lệnh CLI xuyên suốt bài.
{{% /notice %}}

#### Kiến trúc

![Kiến trúc mục tiêu của BorrowIt trên AWS](/images/4-Workshop/4.1-Overview/architecture.png?width=100pc)

Kiến trúc được dựng ở đây, cùng tệp nguồn có thể chỉnh sửa của sơ đồ trên, nằm ở
[2.3](../2-Proposal/2.3-Design/). Phần này dựng nó; bản đề xuất giải thích những
gì đã được quyết trước khi bất cứ thứ gì tồn tại.

#### Nội dung

1. [Giới thiệu](4.1-Overview/)
2. [Lựa chọn dịch vụ AWS](4.3-Architecture/)
3. [Chuẩn bị](4.2-Prerequisites/)
4. [Nền tảng mạng](4.4-Network/)
5. [Tầng dữ liệu](4.5-Data/)
6. [Tầng compute](4.6-Compute/)
7. [Phân phối nội dung](4.7-Delivery/)
8. [Giám sát và vận hành](4.8-Observability/)
9. [Dọn dẹp](4.11-Cleanup/)
