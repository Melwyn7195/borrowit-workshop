---
title : "Cơ sở dữ liệu — chọn RDS"
date : 2026-07-28
weight : 2
chapter : false
pre : " <b> 3.3.2 </b> "
---

#### Các phương án

| Phương án | Mô hình | Cách tính phí | Chi phí ở quy mô này |
|---|---|---|---|
| **RDS PostgreSQL** (`db.t4g.micro`) | Instance quan hệ được quản lý | Theo giờ instance + dung lượng | ~$16/tháng |
| **Aurora Serverless v2** | Quan hệ được quản lý, tự co giãn | Theo giờ ACU | ~$44/tháng ở mức tối thiểu 0.5 ACU |
| **Amazon DynamoDB** | Key-value / document được quản lý | Theo request + dung lượng | ~$0–1/tháng |
| **PostgreSQL trên EC2** | Tự quản lý | Theo giờ instance | ~$8/tháng |

#### Vì sao không chọn DynamoDB

DynamoDB rẻ hơn hẳn, và ở mức lưu lượng này gần như miễn phí. Nó bị loại vì
workload mang tính quan hệ theo cách không hề ngẫu nhiên.

Các truy vấn cốt lõi của BorrowIt join giữa người dùng, món đồ, lượt đặt thuê và
khoảng thời gian còn trống, và đặt những câu hỏi kiểu "món nào còn rảnh trong
khoảng ngày này, loại trừ những món có lượt đặt chồng lấn". Trong DynamoDB, điều đó
trở thành hoặc một thiết kế single-table phi chuẩn hoá phải tính trước rất kỹ, hoặc
nhiều lượt gọi rồi tự join trong mã ứng dụng.

<!-- TODO(prose): nêu đúng truy vấn cụ thể trong ứng dụng đã quyết định điều này.
     Một ví dụ cụ thể — kiểm tra chồng lấn lịch đặt, hay lọc danh sách theo danh mục
     và tình trạng còn trống — làm lập luận trở nên cụ thể thay vì trừu tượng. -->

Vấn đề sâu hơn là schema đã tồn tại rồi. Chuyển sang DynamoDB nghĩa là viết lại mọi
model và truy vấn, chứ không phải chuyển dữ liệu. Đó là viết lại được nguỵ trang
thành migration, và tiến độ không cho phép.

**DynamoDB sẽ là lựa chọn đúng** cho một thiết kế mới hoàn toàn với các mẫu truy
cập đã biết trước. Nó là lựa chọn sai để bê nguyên một schema quan hệ sẵn có.

#### Vì sao không chọn Aurora Serverless v2

Aurora Serverless v2 là thứ gần nhất với khái niệm "RDS tự co giãn theo tải", và về
mặt kỹ thuật vượt trội `db.t4g.micro` ở hầu hết khía cạnh — lưu trữ nhanh hơn,
failover nhanh hơn, co giãn mịn hơn.

Nó tốn khoảng **2,7 lần**. Mức tối thiểu là 0.5 ACU, tính phí liên tục, và khác với
RDS thì không có bậc micro burstable để hạ xuống. Ở mức ~$44/tháng, riêng cơ sở dữ
liệu đã ngốn gần hết số credit.

Aurora là câu trả lời đúng khi tải lên xuống thất thường và khó dự đoán. Workload
này không thuộc cả hai.

#### Vì sao không chọn PostgreSQL trên EC2

Rẻ nhất trong nhóm "gần như được quản lý", và nó trả lại cho bạn mọi thứ RDS đang
làm hộ: backup, vá lỗi, failover, quản lý dung lượng, và mã hoá khi lưu trữ. Với
một dự án có hạn chót, tự vận hành cơ sở dữ liệu là rủi ro chứ không phải tiết kiệm.

Khoản chênh ~$8/tháng không đủ trả cho một buổi tối ngồi khôi phục cơ sở dữ liệu mà
bạn quên sao lưu.

#### Quyết định

**Amazon RDS for PostgreSQL 16** trên `db.t4g.micro` với 20 GB dung lượng gp2.

| Thiết lập | Giá trị | Lý do |
|---|---|---|
| Engine | PostgreSQL 16 | Khớp schema sẵn có; không phải sửa ứng dụng |
| Instance class | `db.t4g.micro` | Graviton — bậc rẻ nhất chạy được Postgres 16 |
| Dung lượng | 20 GB | Mức tối thiểu RDS chấp nhận |
| `maxAllocatedStorage` | **20 GB** | Ghim bằng mức cấp phát để autoscaling không làm tăng hoá đơn |
| Subnet | `PRIVATE_ISOLATED` | Không có đường ra internet theo cả hai chiều |
| `publiclyAccessible` | `false` | Chỉ truy cập được từ trong VPC |
| `storageEncrypted` | `true` | Miễn phí, không có lý do để tắt |
| Multi-AZ | mặc định `false` | Nhân đôi chi phí — bật theo nhu cầu bằng `-c multiAz=true` |
| Thời gian giữ backup | 1 ngày | Đủ khôi phục sau migration hỏng, tốn ít snapshot |

{{% notice note %}}
**Chi phí:** ~$16/tháng, trừ vào credit Free Plan ngay từ giờ đầu tiên. Tài khoản
này tạo sau thay đổi free tier ngày 15/07/2025, nên **không áp dụng ưu đãi RDS miễn
phí 12 tháng** — đừng chọn cấu hình như thể vẫn còn ưu đãi đó.
{{% /notice %}}

#### Cái giá của lựa chọn này

+ **Mặc định là điểm hỏng đơn lẻ.** Một instance trong một AZ. Multi-AZ chỉ cách
  một cờ context nhưng nhân đôi chi phí, nên mặc định nó tắt.
+ **Năng lực cố định.** `t4g.micro` là loại burstable; CPU duy trì trên mức nền sẽ
  cạn credit và bị bóp. Alarm `db-cpu` ở
  [3.8.3](../../3.8-Observability/3.8.3-alarms/) tồn tại để bắt tình huống đó.
+ **Dung lượng không tự lớn.** Ghim `maxAllocatedStorage` bảo vệ ngân sách và khiến
  hết đĩa trở thành tình huống hỏng thật sự — đó là lý do một trong bảy alarm theo
  dõi riêng dung lượng trống.
