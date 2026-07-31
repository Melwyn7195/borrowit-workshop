---
title : "Compute — chọn Fargate thay vì Lambda"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 4.3.1 </b> "
---

Có năm dịch vụ AWS có thể chạy được API này. Đây là lập luận dẫn tới lựa chọn cuối cùng.

#### Các phương án

| Phương án | Bạn phải quản lý gì | Cách tính phí | Chi phí ở quy mô này |
|---|---|---|---|
| **AWS Lambda** | Chỉ mã hàm | Theo request + GB-giây | ~$0 khi tải thấp |
| **ECS trên Fargate** | Image container | Theo vCPU-giây + GB-giây khi đang chạy | ~$9/tháng (0.25 vCPU, 0.5 GB) |
| **ECS trên EC2** | Image container + các instance | Theo giờ instance | ~$8/tháng (t4g.small) |
| **EC2 trực tiếp** | Hệ điều hành, runtime, deploy, vá lỗi | Theo giờ instance | ~$8/tháng |
| **App Runner** | Image container | Theo vCPU/GB kèm mức sàn | Tối thiểu ~$25/tháng |

Xét riêng giá thì Lambda thắng tuyệt đối. Nó vẫn bị loại.

#### Vì sao không chọn Lambda

Lambda là câu trả lời theo phản xạ cho một API lưu lượng thấp, và với thiết kế mới
hoàn toàn theo hướng event-driven thì có lẽ nó đúng. Bốn đặc điểm của *workload
này* phản đối lựa chọn đó:

**1. Mô hình kết nối cơ sở dữ liệu.** API là một server Express dùng connection
pool tới PostgreSQL. Mô hình thực thi của Lambda cấp cho mỗi lời gọi đồng thời một
container riêng, và mỗi container mở kết nối riêng. Một `db.t4g.micro` chịu được
khoảng 80–100 kết nối; chỉ cần một đợt tăng đồng thời vừa phải là cạn. Cách khắc
phục tiêu chuẩn là **RDS Proxy**, tốn khoảng **$11/tháng cho mỗi vCPU của instance
database** — biến phương án rẻ nhất thành ngang giá Fargate, chỉ để giải quyết một
vấn đề mà Fargate vốn không có.

**2. Đây là server chạy dài, không phải tập hợp handler.** Chuyển Express sang
Lambda nghĩa là hoặc sửa mọi route theo chữ ký handler, hoặc bọc cả ứng dụng trong
một adapter để mỗi lần cold start lại khởi động một HTTP server đầy đủ. Cách thứ
nhất là viết lại, thời gian không cho phép; cách thứ hai giữ nguyên chi phí Lambda
mà vứt bỏ gần hết lợi ích của nó.

**3. Cold start nằm trên đường đi của người dùng.** Một container Node.js nguội với
kết nối database mới toanh làm request đầu tiên chậm thấy rõ. Provisioned
concurrency loại bỏ điều đó — và mang trở lại một khoản phí cố định theo giờ, lại
tiệm cận giá Fargate.

**4. Tính dự đoán được của hoá đơn.** Chi phí Lambda tăng theo lưu lượng. Trên một
số dư credit cố định không được nạp thêm, một vòng lặp lỗi hay một con crawler là
một sự cố ngân sách. Fargate với `desiredCount: 1` tốn như nhau mỗi tháng bất kể
chuyện gì xảy ra, và với ngân sách này thì điều đó đáng giá hơn một kỳ vọng chi phí
thấp hơn.

<!-- TODO(prose): nếu bạn thực sự đã dựng thử bản Lambda trước khi quyết định thì
     hãy nói ra và đưa con số bạn đo được. Một quyết định có số liệu chống lưng giá
     trị hơn nhiều so với quyết định chỉ dựa trên lập luận. -->

#### Vì sao không chọn EC2 hay ECS trên EC2

Cả hai rẻ hơn Fargate một chút và cả hai đều trả việc lại cho bạn: vá hệ điều hành,
quản lý vòng đời AMI, tính toán năng lực, và — với ECS trên EC2 — chạy container
agent cùng việc chọn kích thước instance sao cho vừa task.

Khoản tiết kiệm khoảng **$1–2/tháng**. Cái giá là từng giờ bỏ ra cho công việc bảo
trì không tạo khác biệt, trên một dự án có hạn chót cứng. Phần chênh giá của Fargate
mua lại việc xoá bỏ nguyên một nhóm công việc vận hành.

#### Vì sao không chọn App Runner

App Runner thực sự là phương án đơn giản nhất — push image, nhận về một URL HTTPS
kèm chứng chỉ hợp lệ, điều này còn giải quyết luôn vấn đề TLS mô tả ở
[3.7](../../4.7-Delivery/). Nó bị loại vì hai lý do: mức sàn của nó khiến đây là
phương án **đắt nhất** ở quy mô này, và nó che đi phần cấu hình VPC, load balancer
và target group vốn là thứ workshop này tồn tại để trình bày.

#### Quyết định

**ECS trên Fargate**, 0.25 vCPU / 0.5 GB, một task.

| Tiêu chí | Trọng số | Vì sao Fargate thắng |
|---|---|---|
| Chạy ứng dụng sẵn có mà không sửa | Cao | Đóng container, không đổi code |
| Connection pool tới RDS | Cao | Một tiến trình chạy dài, một pool |
| Không phải vá máy chủ | Cao | AWS quản lý phần host |
| Chi phí hằng tháng dự đoán được | Cao | Cố định khi đang chạy |
| Chi phí tuyệt đối thấp nhất | Trung bình | Thua Lambda và EC2 |
| Vận hành đơn giản nhất có thể | Thấp | Thua App Runner |

#### Cái giá của lựa chọn này

Nói rõ mặt trái quan trọng hơn là bảo vệ quyết định:

+ **Bạn trả tiền cả khi nhàn rỗi.** 3 giờ sáng không có traffic, Fargate vẫn tính
  tiền như lúc cao điểm. Lambda thì không.
+ **Mở rộng theo bước thô.** Thêm một task là thêm nguyên 0.25 vCPU; Lambda mở rộng
  theo từng request.
+ **Khởi động chậm.** Một task mới mất 60–90 giây mới healthy, và đó là lý do ngưỡng
  autoscaling đặt tại 65% CPU thay vì 80% — phần dư
  địa đó để hấp thụ lượng tải ập tới trong lúc task đang khởi động.

{{% notice note %}}
**Chi phí:** một Fargate task 0.25 vCPU / 0.5 GB chạy liên tục tốn khoảng
**$9/tháng** tại `ap-southeast-1`. Load balancer đứng trước nó tốn gấp khoảng hai
lần — xem [3.6.3](../../4.6-Compute/4.6.3-load-balancer/).
{{% /notice %}}
