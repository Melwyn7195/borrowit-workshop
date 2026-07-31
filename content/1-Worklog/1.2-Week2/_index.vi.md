---
title: "Nhật ký tuần 2"
date: 2026-07-28
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Mục tiêu tuần 2

* Học các dịch vụ cốt lõi mà quá trình migration sẽ cần: IAM, VPC, EC2, S3, RDS.
* Hiểu mạng VPC đủ sâu để về sau có thể ra quyết định có chủ đích về NAT Gateway.
* Thực hành từng dịch vụ bằng tay trên console trước khi tự động hoá.

### Công việc

| Thứ | Nội dung | Ngày bắt đầu | Ngày hoàn thành | Tài liệu |
| --- | -------- | ------------ | --------------- | -------- |
| 2 | - IAM: user, group, role, policy <br> - Nguyên tắc đặc quyền tối thiểu; vì sao role tốt hơn access key dài hạn | 22/06/2026 | 22/06/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 3 | - VPC: subnet, route table, internet gateway, NAT gateway <br> - So sánh security group và network ACL | 23/06/2026 | 23/06/2026 | |
| 4 | - **Thực hành:** tự dựng một VPC — public và private subnet, khởi tạo EC2, kết nối qua Session Manager | 24/06/2026 | 24/06/2026 | |
| 5 | - S3: bucket, object key, storage class, bucket policy, Block Public Access <br> - **Thực hành:** static site hosting, rồi đưa cùng site đó ra sau CloudFront | 25/06/2026 | 25/06/2026 | |
| 6 | - Kiến thức cơ bản RDS: engine, instance class, Multi-AZ, backup <br> - Tính giá NAT Gateway, ALB, Fargate và RDS đối chiếu với số credit | 26/06/2026 | 26/06/2026 | <https://calculator.aws/> |

### Kết quả tuần 2

<!-- TODO(prose): bài tính giá ở thứ 6 là phần đáng chú ý nhất. NAT Gateway
     ~$33/tháng so với toàn bộ ngân sách dự án chính là thứ dẫn tới thiết kế dùng
     public subnet ở tuần 4. Nếu đó là lúc bạn tính ra điều này thì hãy ghi rõ —
     nó biến một ràng buộc chi phí thành một quyết định thiết kế có căn cứ. -->

* Tự dựng một VPC và kết nối tới EC2 mà không mở cổng SSH ra internet, nhờ
  Session Manager.
* Hiểu được khác biệt giữa security group (có trạng thái, mức instance) và NACL
  (không trạng thái, mức subnet), cùng tình huống phù hợp cho từng loại.
* Tính giá kiến trúc dự kiến đối chiếu với số credit và xác định NAT Gateway là
  khoản chi lớn nhất có thể tránh được.
* Thực hành S3 static hosting và CloudFront, về sau trở thành thiết kế frontend
  ở tuần 8.
