---
title : "Gỡ lỗi task đang chạy"
date : 2026-07-28
weight : 4
chapter : false
pre : " <b> 4.8.4 </b> "
---

Log, metric và alarm cho bạn biết *rằng* có gì đó sai. Trang này nói về việc tìm ra
*cái gì* sai.

#### Trình tự phân loại sự cố

Khi API hỏng, hãy đi từ load balancer ra ngoài. Mỗi bước loại trừ một tầng:

| Bước | Lệnh | Nếu thất bại |
|---|---|---|
| 1. Có task nào đang chạy không? | `aws ecs describe-services` | ECS không đặt hoặc không khởi động được task — sang bước 2 |
| 2. Vì sao task dừng? | `aws ecs describe-tasks` → `stoppedReason` | Lỗi pull image, phân giải secret, hoặc ứng dụng chết |
| 3. Ứng dụng nói gì? | `aws logs tail` | Lỗi ở mức ứng dụng |
| 4. Target có healthy không? | Tình trạng target group | Kiểm tra readiness thất bại — thường là do database |
| 5. Bên trong container đang ra sao? | `aws ecs execute-command` | Mọi khả năng khác đã bị loại trừ |

<!-- TODO(prose): nếu bạn thực sự đã lần theo một sự cố theo trình tự này, hãy viết
     lại thành một ví dụ ngắn. Một sự cố thật được kể từ đầu tới cuối là nội dung
     thuyết phục nhất mà mục này có thể có. -->

#### Đọc một task đã dừng

Trường thông tin hữu ích nhất trong ECS:

```powershell
aws ecs describe-tasks --cluster $env:CLUSTER --tasks $env:TASK `
  --query "tasks[0].{stopped:stoppedReason,exit:containers[0].exitCode,health:healthStatus}"
```

Các giá trị thường gặp và ý nghĩa:

| `stoppedReason` / mã thoát | Nguyên nhân |
|---|---|
| `CannotPullContainerError` | Thiếu tag image, hoặc không có đường tới ECR |
| `ResourceInitializationError: unable to pull secrets` | Execution role không đọc được secret |
| `exec format error` | Image build sai kiến trúc CPU ([3.6.1](../../4.6-Compute/4.6.1-image-and-registry/)) |
| Mã thoát **137** | Bị giết — hết bộ nhớ ([3.8.2](../4.8.2-metrics-dashboard/)) |
| Mã thoát **139** | Lỗi segmentation trong một native module |
| `Essential container in task exited` | Chính ứng dụng đã thoát |

{{% notice note %}}
Với `natGateways: 0`, lỗi `CannotPullContainerError` rất thường là triệu chứng về
mạng chứ không phải về image: task bắt buộc phải có public IP và nằm trong public
subnet. Hãy kiểm tra `assignPublicIp` trước khi kiểm tra tag của image.
{{% /notice %}}

#### Vào bên trong container

```powershell
aws ecs execute-command --cluster $env:CLUSTER --task $env:TASK `
  --container api --interactive --command "/bin/sh"
```

Khi đã vào trong, các phép kiểm tra hữu ích:

```sh
# Tiến trình có chạy không và đang dùng bao nhiêu tài nguyên?
ps aux

# Secret có thực sự được phân giải không? (chỉ tên - tuyệt đối không in giá trị)
env | cut -d= -f1 | sort

# Container có tới được database không?
node -e "require('net').connect(5432, process.env.DB_HOST)
  .on('connect', () => { console.log('reachable'); process.exit(0); })
  .on('error', e => { console.log('unreachable:', e.code); process.exit(1); })"

# Ứng dụng có tự trả lời không?
node -e "require('http').get('http://127.0.0.1:3456/health', r => console.log(r.statusCode))"
```

{{% notice warning %}}
`env` in ra đầy đủ `DB_PASSWORD` đã được inject. Hãy đưa qua `cut -d= -f1` như trên
để chỉ liệt kê tên, và **tuyệt đối không chụp màn hình kết quả thô**.
{{% /notice %}}

#### Phân biệt hai health check

Vì liveness và readiness khác nhau
([3.6.3](../../4.6-Compute/4.6.3-load-balancer/)), triệu chứng của chúng cũng khác
nhau — và phân biệt được là khoanh vùng lỗi ngay lập tức:

| Triệu chứng | Ý nghĩa |
|---|---|
| Task liên tục khởi động lại | **Liveness** thất bại — tiến trình đang chết |
| Task vẫn chạy nhưng target unhealthy | **Readiness** thất bại — tiến trình ổn, database không ổn |
| Cả hai healthy nhưng request trả 5xx | Lỗi logic ứng dụng — hãy xem log |

Dòng ở giữa chính là thiết kế đang hoạt động đúng ý đồ: một sự cố database *không*
làm container khởi động lại.

#### Truy vết ai đã làm gì

Các phiên ECS Exec được ghi trong **CloudTrail**, điều này quan trọng vì tính năng
này cấp quyền mở shell bên trong container production:

```powershell
aws cloudtrail lookup-events `
  --lookup-attributes AttributeKey=EventName,AttributeValue=ExecuteCommand `
  --region ap-southeast-1
```

<!-- TODO(prose): nên kết bằng góc nhìn an toàn. ECS Exec rất mạnh, và cách xử lý
     đúng là giới hạn bằng IAM kèm truy vết thay vì tắt hẳn — nhưng cần ghi rõ rằng
     ở đây nó được bật cho một hệ thống trình diễn, còn với production thì cần giới
     hạn chặt hơn. -->
