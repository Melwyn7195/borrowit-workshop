---
title : "Blogs"
date : 2026-07-31
weight : 3
chapter : true
pre : " <b> 3. </b> "
---

# Các blog đã đăng

Để được đóng mộc thực tập FCAJ, cần **3 bài blog** đăng lên AWS Study Group,
cùng với dự án hoàn thành, báo cáo hoàn thành, 3 tháng thực tập và 10 buổi lên
văn phòng. Phần này liệt kê ba bài đã đăng và nội dung từng bài.

Cả ba bài đều viết lại một quyết định thực tế trong [phần 4](../4-Workshop/)
của workshop này — cùng những bài học đó, kể cho một nhóm độc giả rộng hơn,
trước khi workshop được hoàn thiện.

## Blog 1 — Viết hạ tầng bằng code: vài điều học được khi dùng AWS CDK lần đầu

Viết lại quyết định bỏ hẳn việc bấm console, chuyển sang 100% AWS CDK cho hạ
tầng BorrowIt. Nói về lý do khả năng dựng lại được (chứ không phải tốc độ) mới
là lý lẽ thật sự cho IaC khi ngân sách có hạn, cách chia bốn stack theo **tuổi
thọ và chi phí** thay vì theo loại service, vì sao phụ thuộc giữa các stack
phải đi một chiều (`BorrowitApp` đọc từ ba stack còn lại, không có chiều
ngược) để `cdk destroy BorrowitApp` không bị chặn bởi một CloudFormation
export, và `removalPolicy` quyết định `cdk destroy` có thật sự xoá dữ liệu hay
không.

<!-- SCREENSHOT: /images/3-Blogs/blog-1-cdk.png
     Bài đã đăng trên AWS Study Group. -->

## Blog 2 — Một CloudFront cho cả frontend và API: bài học về HTTPS, cookie và cache

Viết lại quyết định phục vụ SPA và API từ **cùng một CloudFront distribution**
thay vì cho frontend gọi thẳng tới load balancer. Nói về hai lỗi âm thầm khiến
cách làm hai origin thông thường không dùng được: một trang HTTPS không gọi
được tới load balancer `http://` (lỗi mixed content, bị chặn mà không báo gì
rõ ràng), và cookie phiên `sameSite: 'strict'` bị browser âm thầm bỏ khi khác
origin, khiến đăng nhập trả về 200 nhưng mọi request sau đó đều là người lạ.
Đưa cả hai về cùng một domain giải quyết được cả hai lỗi, đồng thời không cần
cấu hình CORS nữa.

<!-- SCREENSHOT: /images/3-Blogs/blog-2-cloudfront.png
     Bài đã đăng trên AWS Study Group. -->

## Blog 3 — Mình học được gì khi dùng AWS Fargate để chạy backend

Viết lại lý do backend chạy trên **Fargate** thay vì một EC2 instance rẻ hơn:
không có OS để vá, không có instance nào để quên đang chạy, và deploy là thay
image chứ không phải sửa server — đổi tiền lấy thời gian cho một dự án một
người, có deadline. Nói về các cặp CPU/RAM cố định mà Fargate chấp nhận
(BorrowIt dùng cặp nhỏ nhất, 256 CPU / 512 MiB), và chi tiết khiến tác giả bất
ngờ nhất: trong cụm ALB + 1 task Fargate (~$31/tháng), **load balancer mới là
phần tốn hơn**, vì ALB tính tiền theo giờ bất kể có request hay không.

<!-- SCREENSHOT: /images/3-Blogs/blog-3-fargate.png
     Bài đã đăng trên AWS Study Group. -->
