---
title: "Đề xuất giải pháp"
date: 2026-07-28
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Chuyển BorrowIt lên AWS — Đề xuất giải pháp

Phần này là bản kế hoạch mà dự án được duyệt dựa trên đó, viết vào cuối **tuần
3**, trước khi có bất kỳ dòng mã hạ tầng nào. Nó nêu bài toán, các yêu cầu mà lời
giải phải đáp ứng, kiến trúc mục tiêu, tiến độ, ngân sách và rủi ro.

Tài liệu được giữ nguyên ở dạng **nhìn về phía trước**. Chỗ nào quá trình triển
khai sau đó đi lệch khỏi kế hoạch thì ghi nhận lại thay vì sửa cho khớp — một bản
đề xuất trùng khít với kết quả thường là bản đã được viết lại sau khi xong việc.

{{% notice note %}}
Phần 2 là **kế hoạch**. [Phần 4](../4-Workshop/) là **biên bản** — quá trình dựng
lại được, kèm đầy đủ lập luận cho từng lựa chọn dịch vụ. Nếu hai phần mâu thuẫn,
phần 4 mới là điều đã thực sự xảy ra.
{{% /notice %}}

#### Đề xuất này cam kết những gì

| | Cam kết |
|---|---|
| **Kết quả** | BorrowIt chạy trọn vẹn trên AWS, không còn phụ thuộc Supabase |
| **Cách làm** | 100% hạ tầng dưới dạng mã bằng AWS CDK — không thao tác tay trên console trong luồng dựng |
| **Region** | `ap-southeast-1` (Singapore) |
| **Ngân sách** | Nằm trong $160 credit AWS Free Plan, hết hạn 28/01/2027 |
| **Hạn chót** | Hệ thống chạy được và workshop hoàn chỉnh trước **21/08/2026** |
| **Bằng chứng** | Mô hình chi phí đối chiếu với hoá đơn thật, và bài kiểm thử failover được chạy chứ không chỉ nói suông |

#### Cách đọc

| Mục | Trả lời câu hỏi |
|---|---|
| [2.1 Bài toán và mục tiêu](2.1-Problem/) | Đang chuyển cái gì, và vì sao phải chuyển |
| [2.2 Phạm vi và yêu cầu](2.2-Requirements/) | Lời giải phải làm được gì, và cái gì nằm ngoài phạm vi |
| [2.3 Thiết kế kiến trúc](2.3-Design/) | Sẽ dựng cái gì — kèm sơ đồ có thể chỉnh sửa |
| [2.4 Đánh giá Well-Architected](2.4-WellArchitected/) | Thiết kế đứng vững tới đâu trước sáu trụ cột của AWS |
| [2.5 Kế hoạch triển khai](2.5-Plan/) | Theo thứ tự nào, đến khi nào, và kiểm chứng mỗi bước ra sao |
| [2.6 Ngân sách và mô hình chi phí](2.6-Budget/) | Tốn bao nhiêu, và chuyện gì xảy ra khi hết credit |
| [2.7 Rủi ro và tiêu chí thành công](2.7-Risks/) | Cái gì có thể hỏng, và căn cứ nào để nói là "xong" |

<!-- TODO(prose): nếu đề xuất này thực sự được mentor review, hãy nói rõ ở đây và
     ghi lại họ đã phản biện điều gì. Một bản đề xuất có lịch sử review thuyết
     phục hơn nhiều so với bản trông như chưa ai chất vấn. -->

#### Nội dung

1. [Bài toán và mục tiêu](2.1-Problem/)
2. [Phạm vi và yêu cầu](2.2-Requirements/)
3. [Thiết kế kiến trúc](2.3-Design/)
4. [Đánh giá Well-Architected](2.4-WellArchitected/)
5. [Kế hoạch triển khai](2.5-Plan/)
6. [Ngân sách và mô hình chi phí](2.6-Budget/)
7. [Rủi ro và tiêu chí thành công](2.7-Risks/)
