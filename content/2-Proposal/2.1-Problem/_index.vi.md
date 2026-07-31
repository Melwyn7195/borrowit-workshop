---
title : "Bài toán và mục tiêu"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 2.1. </b> "
---

#### Hệ thống hiện tại

**BorrowIt** là sàn cho thuê đồ giữa người dùng với nhau. Một người đăng món đồ,
người khác mượn, nền tảng lo phần đăng tin, hình ảnh và ghi nhận ai đang giữ cái
gì.

Ba thành phần, một nhà cung cấp:

| Thành phần | Công nghệ | Đang chạy ở đâu |
|---|---|---|
| Web client | React single-page application | Static hosting |
| API | Express (Node.js), REST, lắng nghe cổng 3456 | Container do Supabase host |
| Cơ sở dữ liệu | PostgreSQL, schema quan hệ | Supabase managed Postgres |
| Ảnh người dùng tải lên | Ảnh món đồ | Supabase Storage |

<!-- TODO(prose): đối chiếu bảng này với hệ thống đang chạy trước khi nộp. Nếu ảnh
     được phục vụ trực tiếp từ API thay vì từ object storage, hãy nói rõ — điều đó
     làm thay đổi câu chuyện migration ở mục 3.7. -->

#### Vì sao phải chuyển

Ứng dụng vẫn chạy tốt. Cần nói thẳng điều đó, vì "nó đang hỏng" không phải lý do
của lần migration này, và giả vờ ngược lại là không trung thực.

Lý do thật sự là:

+ **Một nhà cung cấp nắm mọi tầng.** Compute, cơ sở dữ liệu, lưu trữ và auth đều
  đến từ một vendor, một hoá đơn, một vùng lỗi chung. Không có cách nào scale, bảo
  vệ hay quan sát từng tầng một cách độc lập.
+ **Hạ tầng không được ghi lại ở đâu cả.** Nó được tạo ra bằng cách bấm chuột.
  Không có hiện vật nào dựng lại được, review được, hay giải thích được cho người
  mới.
+ **Không có khả năng quan sát vận hành.** Không metric, không alarm, không ai đọc
  log. Muốn biết API còn sống hay không thì phải mở ứng dụng lên xem.
+ **Đây là đề tài thực tập được giao.** *Đề tài 3 — Application Development on
  AWS*. Việc migration chính là sản phẩm cần nộp, còn nền tảng này là workload
  được chọn để chở nó.

{{% notice tip %}}
Lý do thứ tư mới là lý do thành thật và nó xứng đáng nằm trong tài liệu. Một bản
đề xuất bịa ra tính cấp bách kinh doanh cho một dự án học thuật rất dễ bị nhìn
thấu; một bản nói thẳng động cơ thật rồi vẫn làm kỹ thuật nghiêm túc thì không.
{{% /notice %}}

#### Mục tiêu

Theo thứ tự ưu tiên — khi hai mục tiêu xung đột, mục tiêu ở trên thắng:

| # | Mục tiêu | Đo bằng |
|---|---|---|
| 1 | BorrowIt chạy hoàn toàn trên AWS, không còn phụ thuộc Supabase | Ứng dụng vẫn chạy sau khi thu hồi credential Supabase |
| 2 | Toàn bộ hạ tầng định nghĩa bằng mã | `cdk destroy` rồi `cdk deploy` dựng lại được môi trường |
| 3 | Không vượt ngân sách credit | Cost Explorer, lọc theo `Project=BorrowIt` |
| 4 | Hệ thống quan sát được | Dashboard và alarm báo trước khi người dùng nhận ra |
| 5 | Tài liệu đủ để người khác dựng lại | Workshop ở phần 3, có người đọc làm theo từ tài khoản trống |

Mục tiêu 3 là mục tiêu ràng buộc thiết kế nhiều nhất, và
[2.6](../2.6-Budget/) cho thấy vì sao.

#### Ràng buộc chấp nhận từ đầu

+ **Region `ap-southeast-1`.** Người dùng ở Việt Nam; Singapore là region gần nhất
  có đủ bộ dịch vụ.
+ **$160 credit Free Plan, hết hạn 28/01/2027.** Tài khoản được tạo sau thay đổi
  ngày 15/07/2025 nên **không có free tier 12 tháng** — RDS tính tiền từ giờ đầu
  tiên.
+ **Một người làm, bán thời gian, mười hai tuần.** Điều này loại bỏ mọi phương án
  cần cả một đội để vận hành.
+ **Mã nguồn ứng dụng xem như cố định.** Chỉ sửa những gì migration bắt buộc —
  cấu hình, endpoint health check, client lưu trữ. Không làm thêm tính năng.

#### Không nằm trong mục tiêu

Nêu ra ở đây để việc không làm được hiểu là một quyết định, chứ không phải một
thiếu sót:

+ Cutover không downtime. Chấp nhận một khoảng bảo trì ngắn.
+ Mức độ tuân thủ đạt chuẩn production. Xem đánh đổi ở
  [3.3.3](../../3-Workshop/3.3-Architecture/3.3.3-networking/).
+ Tối ưu chi phí vượt ngoài phạm vi ngân sách credit. Mục tiêu là "vừa túi tiền và
  hiểu rõ", không phải "thấp nhất có thể".
