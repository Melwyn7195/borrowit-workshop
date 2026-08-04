---
title: "Chia sẻ và góp ý"
date: 2026-07-28
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

Những cảm nhận cá nhân về chương trình First Cloud AI Journey, nhằm giúp đội ngũ
FCAJ cải thiện chương trình cho các khoá sau.

<!-- TODO(prose): đây là BẢN NHÁP viết dựa trên repository dự án, không phải từ
     trải nghiệm thực tế của bạn. Đặc biệt các mục 1 và 2 chỉ là phỏng đoán — hãy
     viết lại theo đúng những gì đã diễn ra. Góp ý thẳng thắn hữu ích cho đội ngũ
     hơn lời khen, và không bị trừ điểm. -->

### Đánh giá chung

**1. Môi trường làm việc**

Chương trình chủ yếu diễn ra từ xa, kèm theo các buổi lên văn phòng AWS Việt Nam
theo lịch. Cách tổ chức này phù hợp với tính chất công việc: làm hạ tầng nghĩa là
những khoảng thời gian dài liên tục chờ deploy và đọc tài liệu, và điều đó dễ hơn
khi ngồi ở chỗ làm việc quen thuộc của mình. Các buổi lên văn phòng lại hữu ích
cho những thứ mà làm việc từ xa vốn kém — hỏi một câu mà viết ra thì mất ba tin
nhắn, và nhìn thấy các bạn thực tập khác tiếp cận cùng một vấn đề theo hướng khác.

**2. Sự hỗ trợ từ mentor / đội ngũ quản lý**

Hướng dẫn chủ yếu được đưa dưới dạng gợi ý thay vì đáp án trực tiếp: một dịch vụ
nên xem, một trang tài liệu, một câu hỏi về lý do tôi chọn phương án đó. Lúc đó
thì khó chịu, nhưng nhìn lại thì rõ ràng là cách đúng. Những can thiệp giá trị
nhất không phải là lời giải mà là các câu hỏi thách thức giả định của tôi — được
hỏi thành phần này tốn bao nhiêu một tháng, hay tại sao tài nguyên này cần
public, buộc tôi phải biện minh cho những thiết kế mà tôi chỉ chép lại từ tutorial
mà chưa thực sự hiểu.

Hỗ trợ hành chính nhìn chung kịp thời. Giấy tờ và xác nhận được xử lý mà không
phải hối thúc nhiều.

**3. Mức độ liên quan giữa công việc và chuyên ngành**

Chương trình học Khoa học Máy tính của tôi bao phủ tốt tầng ứng dụng — cấu trúc
dữ liệu, cơ sở dữ liệu, phát triển web; phần Express và React của BorrowIt là
lãnh địa quen thuộc. Thứ hoàn toàn không được dạy là mọi tầng bên dưới: VPC và
định tuyến subnet, chính sách IAM, điều phối container, load balancer, và trên hết
là bài toán kinh tế của việc vận hành tất cả những thứ đó.

**4. Cơ hội học hỏi và phát triển kỹ năng**

Cụ thể, những việc bây giờ tôi làm được mà tháng 6 chưa làm được:

- Thiết kế một VPC — subnet, route table, security group, VPC endpoint — và giải
  thích được chi phí hàng tháng của từng thành phần trong đó.
- Viết một ứng dụng AWS CDK nhiều stack bằng TypeScript với quan hệ phụ thuộc chỉ
  theo một chiều, để có thể xoá và dựng lại một stack riêng lẻ mà CloudFormation
  không từ chối vì một giá trị export.
- Đóng gói ứng dụng Express thành container, đẩy lên ECR và chạy trên Fargate phía
  sau Application Load Balancer.
- Phục vụ SPA và API từ cùng một CloudFront distribution, và giải thích được tại
  sao — mixed content bị trình duyệt chặn và cookie `SameSite=strict` đều âm thầm
  làm hỏng phương án hiển nhiên hơn.
- Quản lý thông tin đăng nhập cơ sở dữ liệu qua Secrets Manager và secret được
  inject vào task definition, thay vì file môi trường.
- Đọc được hoá đơn. Ước tính chi phí hàng tháng trước khi deploy, rồi đối chiếu
  với Cost Explorer sau đó và hiểu được chênh lệch từ đâu ra.

Bài học bền nhất lại nhỏ hơn tất cả những điều trên: lỗi hạ tầng gần như không bao
giờ bí ẩn. Chúng luôn được ghi lại ở đâu đó — trong một CloudFormation event, một
CloudWatch log stream, một security group rule — và kỹ năng ở đây là biết chỗ để
tìm và đủ kiên nhẫn để đọc cho tử tế.

---

### Câu hỏi bổ sung

**Điều gì khiến bạn hài lòng nhất trong kỳ thực tập?**

Nhìn toàn bộ hệ thống dựng lên từ con số không. Chạy `cdk deploy` trên một tài
khoản trống và nhận lại một URL hoạt động được — mạng, cơ sở dữ liệu, API, CDN,
tất cả đều được định nghĩa bằng code và bất kỳ ai làm theo workshop cũng dựng lại
được — là một cảm giác khác hẳn với việc chạy được server ở máy local. Mọi thứ
trong hệ thống đó tồn tại vì tôi đã chọn nó, và tôi giải thích được lý do lẫn cái
giá của từng phần.

Điều thứ hai hẹp hơn: lần đầu tiên tôi chẩn đoán được một lỗi mang dáng dấp
production chỉ từ log, không phải đoán, và bản sửa chạy đúng ngay lần đầu.

---
