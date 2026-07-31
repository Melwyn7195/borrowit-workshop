---
title : "Ngân sách và mô hình chi phí"
date : 2026-07-28
weight : 6
chapter : false
pre : " <b> 2.6. </b> "
---

#### Ngân sách

| | |
|---|---|
| Nguồn kinh phí | **Credit AWS Free Plan — $160** |
| Ngày tạo tài khoản | 28/07/2026 |
| Credit hết hạn | **28/01/2027** |
| Free tier 12 tháng | **Không có.** Tài khoản tạo sau thay đổi ngày 15/07/2025 |
| Kiểm tra số dư | `aws freetier get-account-plan-state` |

Dòng thứ hai và thứ ba là thứ biến đây thành ràng buộc thật chứ không phải hình
thức. Không có free tier để gánh phần cơ sở dữ liệu, nên **RDS tính tiền từ giờ
đầu tiên**, và credit bị tiêu dù hệ thống có ai dùng hay không.

#### Ước tính chi phí hàng tháng

Tính theo giá `ap-southeast-1`, với cấu hình đề xuất ở [2.3](../2.3-Design/):

| Dịch vụ | Cấu hình | Ước tính/tháng |
|---|---|---|
| Interface VPC endpoint | 5 endpoint × 1 AZ | ~$36 |
| Application Load Balancer | 1 ALB, LCU tối thiểu | ~$17 |
| Amazon RDS | `db.t4g.micro`, 20 GB gp2, single-AZ | ~$16 |
| ECS Fargate | 1 task, 0.25 vCPU / 0.5 GB, 24/7 | ~$9 |
| CloudWatch | 1 dashboard, 6 alarm, log | ~$3.60 |
| Public IPv4 | 1 địa chỉ trên task | ~$3.60 |
| S3, CloudFront, ECR, Secrets Manager | Lưu lượng thấp | ~$1 |
| **Tổng** | | **~$86** |

Dòng đầu tiên mới là dòng đáng tranh luận. Năm endpoint PrivateLink tốn hơn tổng
của load balancer, compute và telemetry cộng lại, mà lại không mang traffic ứng
dụng nào — chúng tồn tại để thông tin đăng nhập, image và log không bao giờ băng
qua internet công cộng. [4.3.3](../../4-Workshop/4.3-Architecture/4.3.3-networking/)
trình bày đầy đủ lập luận đó.

#### Có vừa ngân sách không?

| | |
|---|---|
| Toàn hệ thống, chạy liên tục | ~$86/tháng |
| Credit khả dụng | $160 |
| Thời gian chạy được | **~1,8 tháng**, tới khoảng cuối tháng 9/2026 |
| Cần dùng tới | 21/08/2026 — chưa đầy một tháng nữa |
| Chi phí để chạy mọi thứ tới hạn chót | **~$60** |

Vừa, nhưng không còn dư dả. Kết luận đáng chú ý vẫn nằm ở dòng áp chót: **hạn chót
đến trước khi credit cạn**, nên thiết kế không cần tối ưu cho một hệ thống chạy vô
thời hạn. Chỉ có điều bây giờ hạn chót phải giữ đúng. Nếu dự án buộc phải chạy sang
tháng 10, endpoint là dòng đầu tiên cần cân nhắc lại — chúng chiếm ~40% hoá đơn.

{{% notice tip %}}
Đó là lý do việc teardown giữa các buổi làm việc được xem là *tuỳ chọn*, không
phải *bắt buộc*. Xoá rồi dựng lại `BorrowitApp` mỗi tối tiết kiệm được vài đô và
đánh đổi bằng nguy cơ đến buổi demo mà không có gì đang chạy. Một hệ thống đang
sống đáng giá hơn khoản tiết kiệm đó.
{{% /notice %}}

#### Chỗ nào thiết kế đã đánh đổi chi phí lấy thứ khác

Ba quyết định ở [2.3](../2.3-Design/) tồn tại là vì trang này:

| Quyết định | Tiết kiệm | Cái giá |
|---|---|---|
| Không dùng NAT Gateway | ~$33/tháng | Phòng thủ nhiều lớp mỏng hơn; tốn ~$3.60/tháng phí public IPv4 |
| Dùng interface endpoint thay thế | — (chi thêm ~$36/tháng) | Quyết định duy nhất trên trang này đi theo hướng *ngược lại*: mua lại tính bí mật mà NAT lẽ ra đã mang tới, với giá cao hơn chính NAT |
| RDS single-AZ | ~$16/tháng | Không có standby; failover chỉ chứng minh khi cần, không sẵn sàng thường trực |
| Không dùng Container Insights, WAF, Config | ~$10–20/tháng | Ít telemetry hơn và không có phát hiện mối đe doạ được quản lý |

Đảo ngược bất kỳ quyết định nào cũng chỉ là sửa một dòng CDK. Mỗi cái đều dễ đảo
trên giấy nhưng đắt trong thực tế, và chính vì thế chúng được quyết ở đây chứ
không phải giữa lúc đang dựng.

#### Các cơ chế kiểm soát

| Cơ chế | Ở đâu | Ngăn được điều gì |
|---|---|---|
| Budget alarm ở mức $10 | Billing → Budgets | Chi phí vọt lên mà cả tháng không ai biết |
| Cost allocation tag `Project` / `ManagedBy` | Gắn ở cấp CDK app | Một hoá đơn không quy được về đâu |
| `maxAllocatedStorage` ghim bằng `allocatedStorage` | `BorrowitData` | Storage RDS tự mở rộng làm phình hoá đơn trong im lặng |
| Phụ thuộc stack một chiều | Cấu trúc stack | Tầng $31/tháng không xoá được |

{{% notice note %}}
Cost allocation tag phải được **kích hoạt** trong console Billing thì Cost Explorer
mới nhóm theo tag, và **việc kích hoạt không có tác dụng hồi tố**. Hãy làm ngay
tuần 1 — phát hiện ra ở tuần 10 nghĩa là mười tuần chi tiêu không bóc tách được.
{{% /notice %}}

#### Nếu chi phí vượt dự kiến

Theo thứ tự tiết kiệm trên mỗi đơn vị công sức:

1. **Xoá `BorrowitApp`** — bớt ~$31/tháng, dựng lại mất khoảng năm phút.
2. **Giảm thời gian giữ log** xuống dưới một tuần.
3. **Xoá dashboard** — ~$3/tháng, alarm vẫn hoạt động bình thường.
4. **Đừng** đặt `desiredCount` về 0 để tiết kiệm. ALB vẫn tính tiền, nên cách này
   chỉ bớt ~$9 và để lại ~$21.

#### Kiểm chứng ước tính

Một ước tính không bao giờ được đối chiếu thì chỉ là phỏng đoán. Ở tuần 10, mô
hình trên được so với Cost Explorer lọc theo tag dự án, và phần chênh lệch được
giải thích dựa trên các giả định đã nêu ở đây.

```powershell
aws ce get-cost-and-usage `
  --time-period Start=2026-08-01,End=2026-09-01 `
  --granularity MONTHLY `
  --metrics BlendedCost `
  --group-by Type=DIMENSION,Key=SERVICE `
  --filter '{"Tags":{"Key":"Project","Values":["BorrowIt"]}}'
```

<!-- TODO(prose): khi đã có số liệu thật, ghi ở đây ước tính có đúng không. Hai bất
     ngờ hay gặp nhất là lượng log CloudWatch nạp vào và phí public IPv4. Sai mà
     nói ra thì không sao; không kiểm chứng mới là vấn đề. -->
