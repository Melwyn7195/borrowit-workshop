---
title : "Tầng compute"
date : 2026-07-28
weight : 6
chapter : false
pre : " <b> 3.6. </b> "
---

`BorrowitApp` tốn khoảng **$35/tháng** với một task và một load balancer chạy liên
tục. Đây là stack *xoá được* duy nhất trong bốn stack — nó import từ ba stack còn
lại và không stack nào import từ nó, nhờ vậy có thể xoá đi dựng lại riêng một cách
an toàn. (`BorrowitFoundation` còn tốn hơn một chút, ~$36, nhưng là stack vĩnh
viễn; xem [3.4](../3.4-Network/).)

Lập luận cho việc chọn Fargate nằm ở
[3.3.1](../3.3-Architecture/3.3.1-compute/); phần này thực thi nó.

#### Những gì được tạo ra

| Tài nguyên | Vai trò | Chi phí/tháng |
|---|---|---|
| ECS cluster | Nhóm logic; bản thân không tốn phí | $0 |
| Fargate task definition + service | Chạy container | ~$9 |
| Application Load Balancer | Điểm vào công khai, kiểm tra sức khoẻ | ~$17 |
| Target group | Định tuyến tới task healthy | $0 |
| CloudWatch log group | stdout/stderr của container | ~$0.50 |
| Dashboard + 7 alarm + SNS topic | Trình bày ở [3.8](../3.8-Observability/) | ~$4,80 |

#### Nội dung

1. [Image container và registry](3.6.1-image-and-registry/)
2. [Task definition và service](3.6.2-task-and-service/)
3. [Load balancer và health check](3.6.3-load-balancer/)
4. [Khởi tạo cơ sở dữ liệu](3.6.4-database-init/)
