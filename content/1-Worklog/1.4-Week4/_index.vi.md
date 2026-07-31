---
title: "Nhật ký tuần 4"
date: 2026-07-28
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Mục tiêu tuần 4

* Học AWS CDK đủ để xây dựng dự án bằng code thay vì bấm console.
* Triển khai `BorrowitFoundation` — VPC, security group, ECR.
* Chốt thiết kế không dùng NAT và để việc cách ly đến từ security group.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - Khái niệm CDK: App, Stack, Construct, phân biệt L1/L2/L3 <br> - Cách CDK synth ra CloudFormation | 06/07/2026 | 06/07/2026 | <https://docs.aws.amazon.com/cdk/v2/guide/> |
| 3 | - **Thực hành:** `cdk init`, `cdk bootstrap`, `cdk synth`, `cdk deploy`, `cdk destroy` trên một stack thử | 07/07/2026 | 07/07/2026 | |
| 4 | - Viết `FoundationStack`: VPC với `natGateways: 0`, public + isolated subnet trên 2 AZ | 08/07/2026 | 08/07/2026 | |
| 5 | - Thêm ba security group, mỗi group tham chiếu group trước đó thay vì dải CIDR <br> - Thêm S3 gateway endpoint miễn phí | 09/07/2026 | 09/07/2026 | |
| 6 | - Thêm ECR repository với `imageScanOnPush` và lifecycle rule giữ 10 image <br> - Deploy và kiểm chứng trên VPC console | 10/07/2026 | 10/07/2026 | |

### Kết quả tuần 4

<!-- TODO(prose): chuỗi security group là điểm kỹ thuật nổi bật của tuần này. Mỗi
     group chỉ nhận traffic từ group liền trước, nên quy tắc vẫn đúng khi ECS thay
     task và IP thay đổi. Hãy giải thích bằng lời của bạn — đây chính là câu trả
     lời cho "vì sao một task có public IP vẫn an toàn?" -->

* Triển khai stack đầu tiên hoàn toàn bằng code — không bấm console.
* Dựng VPC **không có NAT Gateway**, tiết kiệm ~$33/tháng, và chấp nhận hệ quả là
  Fargate task phải nằm ở public subnet.
* Thực hiện cách ly bằng chuỗi security group (ALB → service → database) thay vì
  dựa vào vị trí subnet.
* Đặt ECR repository ở tầng foundation thay vì tầng app, để khi xoá app không mất
  các image đã push.

