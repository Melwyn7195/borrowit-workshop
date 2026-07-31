---
title : "Rủi ro và tiêu chí thành công"
date : 2026-07-28
weight : 7
chapter : false
pre : " <b> 2.7. </b> "
---

#### Danh mục rủi ro

Khả năng xảy ra và mức tác động được đánh giá theo hạn chót **21/08/2026** — một
rủi ro làm mất hai ngày ở tuần 9 nghiêm trọng hơn nhiều so với ở tuần 4.

| # | Rủi ro | Khả năng | Tác động | Cách giảm thiểu |
|---|---|---|---|---|
| R1 | Task Fargate không kéo được image hoặc không đọc được secret, service không bao giờ ổn định | Cao | Cao | Deploy stack foundation và data trước; kiểm tra task role và ECR repository trước khi tạo service. Đọc **lý do task bị dừng**, không phải service event |
| R2 | Health check cấu hình sai — ALB giết một container vẫn khoẻ | Cao | Trung bình | Tách `/healthz` (liveness, không chạm DB) khỏi `/readyz` (readiness, có kiểm tra DB); đặt `startPeriod` rộng rãi |
| R3 | Dữ liệu export từ Supabase không khôi phục sạch | Trung bình | Cao | Làm phần restore ở tuần 5, không để tới lúc cutover; giữ `db/seed_data.sql` như đường dựng lại |
| R4 | Hết credit trước hạn chót | Thấp | Cao | ~$86/tháng so với $160 mà chỉ còn chưa đầy một tháng; budget alarm ở $10; `BorrowitApp` xoá được, và ~$36 tiền VPC endpoint có thể bỏ đi nếu hạn chót bị lùi |
| R5 | Một thay đổi security group vô tình phơi task hoặc database ra internet | Thấp | Cao | Rule ingress luôn tham chiếu security group, không bao giờ dùng CIDR; thử kết nối trực tiếp sau mỗi lần đổi mạng |
| R6 | Task duy nhất chết và không có gì để chuyển sang | Trung bình | Trung bình | Chấp nhận. ECS khởi động lại trong 1–2 phút; alarm sẽ báo. Nâng lên hai task chỉ là một tham số nếu điều này thành vấn đề |
| R7 | Quên tắt Multi-AZ sau khi diễn tập failover, chi phí RDS nhân đôi | Trung bình | Thấp | Chỉ bật bằng context flag tường minh `-c multiAz=true`, nên mặc định là tắt và trả về cũ chỉ cần deploy lại |
| R8 | Không kịp chụp ảnh và thu bằng chứng trước khi teardown | Trung bình | Trung bình | Thu bằng chứng ngay trong buổi làm việc đó; workshop đánh dấu mọi chỗ cần ảnh bằng chú thích `SCREENSHOT` |
| R9 | Mất thời gian vì lỗi CDK hoặc CloudFormation khó đọc | Cao | Trung bình | Hai tuần dự phòng ở tuần 11–12; chạy `cdk diff` trước mọi lần deploy |

R1 và R2 là hai rủi ro được xếp *Cao/Cao* và *Cao/Trung bình*, và cả hai đều rơi
vào cùng một tuần. Đó là lý do giai đoạn 2 ở [2.5](../2.5-Plan/) được cấp hai tuần
và bị đánh dấu là rủi ro tiến độ.

{{% notice warning %}}
**R7 là rủi ro âm thầm tốn tiền.** Multi-AZ nhân đôi phí instance RDS và không có
gì trong console nhắc bạn về điều đó. Ai chạy diễn tập failover thì tắt nó đi ngay
trong ngày.
{{% /notice %}}

#### Rủi ro được chấp nhận thay vì giảm thiểu

Nói rõ những điều này chính là mục đích của phần này:

+ **Không có NAT Gateway** — một sai sót security group sẽ phơi task ra ngoài. Chấp
  nhận vì phương án thay thế tốn hơn cả ngân sách compute. Lập luận đầy đủ ở
  [4.3.3](../../4-Workshop/4.3-Architecture/4.3.3-networking/).
+ **API không có HTTPS** — không có domain để xác thực chứng chỉ. Chấp nhận với bản
  demo; sẽ không chấp nhận được nếu có người dùng thật.
+ **`removalPolicy: DESTROY` trên database** — xoá `BorrowitData` là mất dữ liệu.
  Chấp nhận vì đây là dữ liệu mẫu và `db/seed_data.sql` dựng lại được; sẽ không thể
  biện minh nếu là dữ liệu người dùng thật.
+ **Không có CI/CD** — mọi lần deploy đều thủ công và không lưu vết. Chấp nhận ở
  quy mô một người.

#### Tiêu chí thành công

Dự án hoàn thành khi mọi dòng dưới đây đều có bằng chứng đi kèm — một ảnh chụp,
một kết quả lệnh, hoặc một mục trong workshop.

| # | Tiêu chí | Bằng chứng |
|---|---|---|
| S1 | Ứng dụng chạy trọn vẹn trên AWS, không còn phụ thuộc Supabase | Thao tác trên trình duyệt: đăng tin, tải ảnh, tải lại trang |
| S2 | Toàn bộ môi trường dựng lại được từ mã | `cdk destroy BorrowitApp` rồi `cdk deploy BorrowitApp`, chạy lại bình thường |
| S3 | Cơ sở dữ liệu không tiếp cận được từ internet | Một lần thử kết nối từ bên ngoài bị timeout |
| S4 | API không tiếp cận được ngoài đường qua load balancer | `curl` vào public IP của task, bị timeout |
| S5 | Không credential nào tồn tại ngoài Secrets Manager | Task definition đã render, hiện `valueFrom` chứ không phải giá trị |
| S6 | Phát hiện sự cố mà không cần mở ứng dụng | Email alarm do task bị dừng kích hoạt |
| S7 | Cơ sở dữ liệu sống sót khi mất một AZ | Một lần ép failover Multi-AZ, có ghi thời gian khôi phục |
| S8 | Chi tiêu nằm trong ngân sách và được hiểu rõ | Cost Explorer đặt cạnh ước tính ở [2.6](../2.6-Budget/) |
| S9 | Người khác dựng lại được | Một bạn cùng khoá hoàn thành workshop phần 4 từ tài khoản trống |

<!-- TODO(prose): cuối dự án, quay lại đánh dấu từng dòng một cách trung thực. Một
     tiêu chí không đạt kèm lời giải thích đọc hay hơn nhiều so với chín dấu tích
     không điều kiện — và phần 5 chính là chỗ người chấm tìm đúng kiểu tự đánh giá
     đó. -->

#### Phê duyệt

| | |
|---|---|
| Người lập | *[họ tên của bạn]* — thực tập sinh FCAJ |
| Ngày | 28/07/2026 |
| Người review | *[tên mentor]* |
| Kết luận | *[duyệt / duyệt kèm chỉnh sửa / không duyệt]* |

<!-- TODO: điền bảng trên. Nếu đề xuất này chưa từng được review chính thức, hãy
     nói vậy thay vì bịa ra một chữ ký — và ghi lại bạn đã trao đổi kiến trúc với
     những ai. -->
