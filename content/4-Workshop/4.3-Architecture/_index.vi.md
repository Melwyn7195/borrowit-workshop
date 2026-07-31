---
title : "Lựa chọn dịch vụ AWS"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 4.3. </b> "
---

AWS thường có ba tới bốn dịch vụ về mặt kỹ thuật đều làm được việc. Phần kỹ thuật
nằm ở chỗ chọn giữa chúng và giải thích được vì sao.

Phần này ghi lại từng quyết định đúng như lúc nó được đưa ra: các phương án đã cân
nhắc, tiêu chí áp dụng, lựa chọn cuối cùng, và — phần mà hầu hết báo cáo bỏ qua —
**cái giá của lựa chọn đó**. Mọi quyết định ở đây đều dễ đảo ngược trên giấy nhưng
tốn kém trên thực tế, nên chúng được chốt trước khi viết dòng code nào ở
[3.4](../4.4-Network/) trở đi.

#### Tổng hợp quyết định

| Tầng | Lựa chọn | Phương án chính bị loại | Yếu tố quyết định |
|---|---|---|---|
| Compute | **ECS trên Fargate** | Lambda, EC2, ECS trên EC2, App Runner | Tiến trình chạy dài kèm connection pool; không phải vá máy chủ |
| Cơ sở dữ liệu | **RDS PostgreSQL** | Aurora Serverless v2, DynamoDB, Postgres trên EC2 | Đã có schema quan hệ; chi phí cố định dễ dự đoán |
| Mạng chiều ra | **Public subnet, không NAT** | NAT Gateway, dùng VPC endpoint cho mọi thứ | NAT tốn hơn toàn bộ ngân sách compute |
| Phân phối tĩnh | **S3 + CloudFront** | Amplify Hosting, phục vụ từ ALB | Chi phí, và giữ bucket private nhờ OAC |
| Secret | **Secrets Manager** | SSM Parameter Store, biến môi trường thường | Tích hợp sẵn với RDS để sinh và lưu mật khẩu |
| Registry | **ECR** | Docker Hub, GitHub Container Registry | Xác thực bằng IAM, pull cùng region, quét khi push |
| Giám sát | **CloudWatch + SNS** | Container Insights, X-Ray, dịch vụ bên thứ ba | Đủ tín hiệu với ~$4,80/tháng |

#### Về ràng buộc

Mọi quyết định bên dưới đều được đưa ra dưới số dư cố định **$160 credit AWS Free
Plan**, trên một tài khoản tạo sau thay đổi free tier ngày 15/07/2025 — nên
**không có free tier 12 tháng**, và RDS bị tính phí ngay từ giờ đầu tiên.

Ràng buộc này được nêu ngay từ đầu vì nó là yếu tố quyết định ở ba trong bảy dòng
phía trên. Một ngân sách khác sẽ dẫn tới một kiến trúc khác, và nói rõ điều đó
trung thực hơn là trình bày các lựa chọn này như thể luôn luôn đúng.

#### Nội dung

1. [Compute — vì sao chọn Fargate thay vì Lambda](4.3.1-compute/)
2. [Cơ sở dữ liệu — vì sao chọn RDS thay vì Aurora Serverless hay DynamoDB](4.3.2-database/)
3. [Mạng — vì sao không dùng NAT Gateway](4.3.3-networking/)
4. [Phân phối, lưu trữ và secret](4.3.4-delivery-and-secrets/)
