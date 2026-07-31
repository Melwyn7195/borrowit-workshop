---
title : "Khởi tạo cơ sở dữ liệu"
date : 2026-07-28
weight : 4
chapter : false
pre : " <b> 3.6.4 </b> "
---

Database đang chạy nhưng rỗng, nên kiểm tra readiness vẫn thất bại. Bước này tạo
schema — và phần thú vị là *làm sao tới được một database không có đường mạng nào
từ máy bạn*.

#### Vấn đề truy cập

RDS nằm trong isolated subnet với `publiclyAccessible: false`. Không có route nào
từ máy trạm tới nó. Đó là thiết kế đang hoạt động đúng, và cũng là điều bất tiện.

| Phương án | Chi phí/tháng | Kết luận |
|---|---|---|
| Bật public access cho RDS | $0 | Phá vỡ toàn bộ thiết kế cách ly. Không. |
| Bastion host trong public subnet | ~$8 (t4g.nano) | Chạy được — nguyên một EC2 phải vá lỗi và nhớ tắt, chỉ để chạy một lệnh |
| VPN hoặc Direct Connect | $$$ | Không tương xứng |
| **`aws ecs execute-command`** | **$0** | Shell ngay trong task vốn đã kết nối được tới database |

#### Vì sao ECS Exec thắng ở đây

`enableExecuteCommand: true` trên service mở ra phương án thứ ba. Nó hoạt động qua
**AWS Systems Manager Session Manager**: ECS agent mở kết nối chiều ra tới SSM, và
CLI của bạn kết nối qua cùng kênh đó. Không mở cổng chiều vào, không khoá SSH,
không thêm máy chủ nào.

CDK tự thêm các quyền SSM cần thiết vào task role.

Về mặt an toàn, phương án này tốt hơn bastion chứ không chỉ rẻ hơn:

+ Quyền truy cập do **IAM** kiểm soát, không phải do sở hữu khoá SSH.
+ Mọi phiên đều truy vết được trong **CloudTrail**.
+ Không có máy chủ dài hạn phải vá lỗi, không có gì để quên tắt.

<!-- TODO(prose): trình bày như một quyết định của bạn. Bastion là câu trả lời quen
     thuộc và dễ giải thích với người chưa biết Session Manager; ECS Exec không tốn
     gì và không để lại dấu vết nào. -->

#### Mở shell trong task đang chạy

```powershell
$env:CLUSTER = (aws cloudformation describe-stacks `
  --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='ClusterName'].OutputValue" `
  --output text)

$env:TASK = (aws ecs list-tasks --cluster $env:CLUSTER --query "taskArns[0]" --output text)

aws ecs execute-command `
  --cluster $env:CLUSTER `
  --task $env:TASK `
  --container api `
  --interactive `
  --command "/bin/sh"
```

{{% notice note %}}
Lỗi `TargetNotConnectedException` thường có nghĩa là chưa cài Session Manager
plugin, hoặc task đã khởi động trước khi bật `enableExecuteCommand`. Hãy ép deploy
lại rồi thử lần nữa.
{{% /notice %}}

#### Chạy schema

Bên trong shell của container:

```sh
node scripts/run-sql.js db/schema.sql
npm run seed
```

Các script kết nối bằng chính các biến `DB_*` mà ECS đã nạp từ Secrets Manager —
nên không phải gõ chuỗi kết nối ở bất kỳ đâu, và mật khẩu vẫn chưa từng có ai nhìn thấy.

<!-- TODO: file schema phải có sẵn trong image trước lệnh docker build ở 3.6.1. Nếu
     database nguồn nằm nơi khác, hãy export trước bằng:
       pg_dump --schema-only --no-owner --no-privileges <source-url> -f db/schema.sql
     --no-owner và --no-privileges loại bỏ các lệnh cấp quyền sẽ lỗi trên RDS.
     Hãy ghi lại những điểm không tương thích bạn thực sự gặp — một báo cáo migration
     không có chút trở ngại nào đọc như thể migration chưa từng diễn ra. -->

#### Target chuyển sang healthy

Sau khoảng một phút, kiểm tra readiness thành công và target chuyển sang **healthy**.

Kiểm chứng từ đầu tới cuối:

```powershell
curl "$env:API/health"
curl "$env:API/api/products"
```

{{% notice warning %}}
`enableExecuteCommand` cấp quyền mở shell bên trong container production đang chạy
cho bất kỳ ai có quyền IAM tương ứng. Nó cực kỳ hữu ích cho việc gỡ lỗi vận hành ở
[3.8.4](../../3.8-Observability/3.8.4-debugging/), và là một đặc quyền nên được giới
hạn có chủ đích thay vì để trong một policy quá rộng.
{{% /notice %}}
