---
title : "Phạm vi và yêu cầu"
date : 2026-07-28
weight : 2
chapter : false
pre : " <b> 2.2. </b> "
---

Yêu cầu được viết sao cho từng cái đều **kiểm chứng được**, không phải để tranh
luận. Thứ gì không chứng minh được vào cuối dự án thì không thuộc về trang này.

#### Yêu cầu chức năng

| ID | Yêu cầu | Kiểm chứng bằng |
|---|---|---|
| F1 | Web client tải qua HTTPS từ một endpoint toàn cầu | Mở URL CloudFront |
| F2 | API giữ nguyên hợp đồng REST như khi chạy trên Supabase | Client cũ chạy được với base URL mới mà không sửa gì |
| F3 | API lưu dữ liệu vào PostgreSQL được quản lý | Tạo một tin đăng, kiểm tra bản ghi bằng `psql` |
| F4 | Ảnh món đồ tải lên được và phục vụ lại cho client | Tải ảnh qua ứng dụng, mở URL của object |
| F5 | Schema và dữ liệu mẫu khôi phục nguyên vẹn | Chạy `db/seed_data.sql`, số dòng khớp |
| F6 | API có endpoint liveness và readiness | `GET /healthz` và `GET /readyz` trả về 200 |

F6 là phần việc mới trên ứng dụng. Health check trước đây không tồn tại và là thứ
load balancer bắt buộc phải có — thay đổi duy nhất mà migration này ép lên mã
nguồn.

#### Yêu cầu phi chức năng

| ID | Yêu cầu | Mục tiêu | Kiểm chứng bằng |
|---|---|---|---|
| N1 | Cơ sở dữ liệu không thể truy cập từ internet | Không có route công khai | Thử kết nối từ laptop; phải thất bại |
| N2 | Container API chỉ tiếp cận được qua load balancer | Security group chỉ nhận traffic từ ALB | `curl` thẳng vào public IP của task; phải timeout |
| N3 | Không credential nào nằm trong mã nguồn, `.env` hay task definition | Bằng không | `grep` toàn repo; đọc task definition đã render |
| N4 | Deploy hỏng tự động rollback | Không cần can thiệp tay | Deploy một image cố tình hỏng và xem nó tự quay lại |
| N5 | Log của API truy vấn được tập trung | Giữ 1 tuần | Một truy vấn Logs Insights trả về lỗi ứng dụng |
| N6 | Hệ thống không khoẻ thì phát cảnh báo | Email trong khoảng 5 phút | Dừng task, chờ email SNS |
| N7 | Cơ sở dữ liệu sống sót khi mất một Availability Zone | Khôi phục không mất dữ liệu | Bật Multi-AZ, ép failover, đo thời gian |
| N8 | Traffic API AWS của workload không băng qua internet công cộng | Kéo image, đọc secret và đẩy log đi qua PrivateLink | `describe-vpc-endpoints` cho thấy năm interface endpoint có private DNS |
| N9 | Chi phí hàng tháng nằm trong ngân sách credit | ≤ $90/tháng, và credit trụ qua hạn chót | Cost Explorer đối chiếu ước tính ở [2.6](../2.6-Budget/) |

{{% notice note %}}
**N7 và N9 mâu thuẫn trực tiếp với nhau.** Multi-AZ nhân đôi chi phí instance RDS.
Cách xử lý đề xuất ở đây là chạy single-AZ và chỉ bật Multi-AZ khi cần diễn tập
failover rồi tắt đi — tức là thoả mãn N7 như một **năng lực đã chứng minh** chứ
không phải một **cấu hình thường trực**. Đó là sự nới lỏng thật sự của yêu cầu, và
nó được nói ra chứ không giấu đi.
{{% /notice %}}

{{% notice note %}}
**N8 và N9 mới là mâu thuẫn tốn kém hơn**, và khác với N7, nó được xử lý theo hướng
ngược lại. Thoả mãn N8 tốn ~$36/tháng cho interface VPC endpoint — hơn cả load
balancer — và đó là lý do trần của N9 dời từ $50 lên $90. Phương án còn lại là để
endpoint sau một cờ tuỳ chọn và coi N8 là "đã chứng minh chứ không thường trực",
đúng như cách xử lý N7. Phương án đó bị loại: một biện pháp bảo mật tắt trong lúc
vận hành bình thường thì không còn là biện pháp, và credit vẫn đủ tới hạn chót. Xem
[2.6](../2.6-Budget/) để biết phép tính khiến nó vừa vặn.
{{% /notice %}}

#### Ngoài phạm vi

| Không làm | Vì sao | Cái giá của việc không làm |
|---|---|---|
| Pipeline CI/CD | Deploy thưa và do một người thực hiện | Mọi lần deploy đều thủ công và không có dấu vết |
| Tên miền riêng và TLS trên ALB | Không có ngân sách tên miền; ACM cần domain để xác thực | Traffic API đi HTTP; chỉ chặng CloudFront là HTTPS |
| Multi-region | Nhân đôi mọi thứ cho một tệp người dùng một vùng | Sự cố toàn region làm sập hệ thống |
| Chính sách auto-scaling | Một task đủ cho lưu lượng demo | Một đợt tăng tải sẽ do đúng một container gánh |
| WAF, GuardDuty, Config | Mỗi thứ đều cộng chi phí định kỳ vào ngân sách cố định | An toàn chỉ dựa vào security group và IAM |
| Phát triển tính năng ứng dụng | Ứng dụng là workload, không phải chủ đề | — |

Cột thứ ba là phần mà hầu hết bảng phạm vi bỏ qua. Nêu rõ cái giá của từng mục bị
loại chính là thứ biến nó thành một quyết định phạm vi thay vì một lỗ hổng.

#### Giả định

+ Lưu lượng ở mức demo: vài chục request mỗi phút, không phải hàng nghìn.
+ Chấp nhận một khoảng bảo trì duy nhất khi chuyển dữ liệu.
+ Container image build và chạy được ở máy cá nhân trước khi động tới AWS.
+ Số dư credit và bảng giá AWS giữ nguyên trong suốt dự án.

<!-- TODO(prose): nếu có giả định nào đổ vỡ trong quá trình làm, nó đáng một câu ở
     đây và một ghi chú đầy đủ hơn ở phần 4. Một giả định sai nhưng đã được xử lý
     là bằng chứng mạnh hơn một giả định tình cờ đúng. -->
