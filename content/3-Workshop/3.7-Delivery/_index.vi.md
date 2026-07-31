---
title : "Phân phối nội dung"
date : 2026-07-28
weight : 7
chapter : false
pre : " <b> 3.7. </b> "
---

`BorrowitFrontend` gồm hai S3 bucket và một CloudFront distribution. Ở mức lưu
lượng này chi phí **dưới $1/tháng**. Lập luận lựa chọn dịch vụ nằm ở
[3.3.4](../3.3-Architecture/3.3.4-delivery-and-secrets/).

Distribution này phục vụ cả ứng dụng đã biên dịch **lẫn API**. Chính quyết định
đó mới là nội dung thật sự của trang này.

#### Một origin, không phải hai

Thiết kế hiển nhiên là hai hostname: CloudFront cho bundle tĩnh, load balancer
cho API. Cách đó không dùng được ở đây, và cả hai lý do đều là lỗi âm thầm chứ
không phải lỗi nhìn thấy được:

+ **CloudFront phục vụ HTTPS; ALB chỉ lắng nghe HTTP.** Một trang tải qua HTTPS
  không thể gọi endpoint `http://` — trình duyệt chặn vì mixed content trước cả
  khi request được gửi đi. ACM không cấp chứng chỉ cho hostname
  `elb.amazonaws.com`, nên không có cách rẻ nào đặt TLS thẳng lên load balancer.
+ **`userController.js` đặt cookie phiên với `sameSite: 'strict'`.** Trỏ frontend
  sang một API host khác site thì trình duyệt âm thầm bỏ cookie đó. Đăng nhập trả
  về 200, và mọi request sau đó đều là ẩn danh.

Định tuyến API qua cùng distribution làm cho nó **same-origin**, khắc phục cả hai
cùng lúc — và loại bỏ hoàn toàn nhu cầu CORS trên đường đi của trình duyệt.

{{% notice note %}}
Hệ quả với bản build frontend: API được gọi bằng **đường dẫn tương đối**, và
`VITE_API_URL` cố ý để **rỗng**. Không có hostname API nào nằm trong bundle. Ở
môi trường dev, Vite proxy `/api` sang `localhost:3456`; ở production, CloudFront
định tuyến sang load balancer. Cùng một đoạn code cho cả hai môi trường.
{{% /notice %}}

#### Các behaviour

| Behaviour | Origin | Phục vụ |
|---|---|---|
| Mặc định (`*`) | Web bucket (OAC) | Bundle ứng dụng đã biên dịch |
| `/uploads/*` | Uploads bucket (OAC) | Ảnh người dùng tải lên |
| `/api/*` | Application Load Balancer | REST API |
| `/health` | Application Load Balancer | Readiness probe, gọi được từ edge |
| `/api-docs*` | Application Load Balancer | Swagger UI và tài nguyên của nó |

Tiền tố `/uploads/*` ánh xạ thẳng vào tiền tố object key của S3 — ứng dụng ghi
object dưới `uploads/`, nên không cần rule rewrite nào.

`/api-docs*` cố ý không có dấu gạch chéo trước ký tự đại diện: nó phải khớp cả
`/api-docs` lẫn các tài nguyên Swagger tải từ `/api-docs/`. Nó không chồng lấn
`/api/*` — pattern đó thì cần dấu gạch chéo.

#### Behaviour của API

```typescript
const apiOrigin = new origins.HttpOrigin(albDns, {
  protocolPolicy: cloudfront.OriginProtocolPolicy.HTTP_ONLY,
  httpPort: 80,
  readTimeout: cdk.Duration.seconds(30),
});

const apiBehavior: cloudfront.BehaviorOptions = {
  origin: apiOrigin,
  viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
  allowedMethods: cloudfront.AllowedMethods.ALLOW_ALL,
  cachePolicy: cloudfront.CachePolicy.CACHING_DISABLED,
  originRequestPolicy: cloudfront.OriginRequestPolicy.ALL_VIEWER,
};
```

Cả bốn thiết lập đều mang tính quyết định:

| Thiết lập | Vì sao |
|---|---|
| `HTTP_ONLY` | ALB không có chứng chỉ. Chặng này chạy bên trong AWS giữa edge và load balancer; nửa phía trình duyệt vẫn là HTTPS, và đó mới là điều quy tắc mixed content cùng cờ secure cookie quan tâm |
| `ALLOW_ALL` | Đây là API có ghi. POST, PUT và DELETE đều phải tới được origin — mặc định chỉ-đọc của CloudFront sẽ từ chối chúng |
| `CACHING_DISABLED` | Không bao giờ cache một phản hồi đã xác thực ở edge dùng chung. Hai người gọi `/api/users/me` không được thấy câu trả lời của nhau |
| `ALL_VIEWER` | Chuyển tiếp cookie, query string và header tới origin. Cookie phiên **chính là** cơ chế xác thực; bất kỳ mức nào thấp hơn cũng làm mọi request bị đăng xuất |

#### Tìm load balancer

`BorrowitFrontend` cần DNS name của ALB, thứ chỉ tồn tại khi `BorrowitApp` đã
được deploy. Import nó sẽ đảo chiều phụ thuộc và làm hỏng
`cdk destroy BorrowitApp` ([3.1](../3.1-Overview/)), nên giá trị này được truyền
ngoài luồng: AppStack **công bố** nó vào một tham số SSM, còn FrontendStack
**tra cứu** nó.

```typescript
// Trong BorrowitApp - tham số standard là miễn phí.
new ssm.StringParameter(this, 'AlbDnsParameter', {
  parameterName: '/borrowit/alb-dns',
  stringValue: alb.loadBalancerDnsName,
});

// Trong BorrowitFrontend - phân giải ở thời điểm synth.
const looked = ssm.StringParameter.valueFromLookup(this, '/borrowit/alb-dns');
const albDns = looked.startsWith('dummy-value-for-') ? undefined : looked;
```

{{% notice warning %}}
Việc tra cứu diễn ra ở **thời điểm synth, một cách có chủ đích**. Tham chiếu
`{{resolve:ssm:...}}` ở thời điểm deploy trông gọn hơn nhưng sai: nó giữ nguyên
template, nên khi load balancer bị thay thế, CloudFormation báo "no changes" và
âm thầm tiếp tục trỏ tới một load balancer không còn tồn tại. Phân giải lúc synth
ghi thẳng hostname vào template, nên một lần thay thế sẽ thật sự làm template đổi.
{{% /notice %}}

Context vẫn ghi đè được kết quả tra cứu, cho hai tình huống:

```powershell
# Lần deploy đầu, khi BorrowitApp chưa tồn tại - bỏ qua hẳn các behaviour API.
npx cdk deploy BorrowitFrontend -c albDns=none

# Ép một hostname cụ thể, bỏ qua mọi giá trị đã cache.
npx cdk deploy BorrowitFrontend -c albDns=borrowit-alb-123.ap-southeast-1.elb.amazonaws.com
```

Trước đây đây chỉ là một context flag trần không có SSM dự phòng, và đó là một
cái bẫy: bất kỳ lần deploy nào quên flag cũng âm thầm bỏ mất các behaviour
`/api/*`, với triệu chứng là mọi lệnh gọi API trả về **HTTP 200 kèm khung SPA**
thay vì một lỗi. Đọc từ Parameter Store giúp định tuyến sống sót qua một lệnh
`cdk deploy BorrowitFrontend` thông thường.

#### Phục vụ ứng dụng single-page

Một route phía client như `/seller/dashboard` không có object S3 tương ứng. Vì
Origin Access Control chỉ cấp `GetObject` mà không cấp `ListBucket`, S3 trả về
**403** chứ không phải 404.

```typescript
const spaRouting = new cloudfront.Function(this, 'SpaRouting', {
  code: cloudfront.FunctionCode.fromInline(`
function handler(event) {
  var request = event.request;
  if (request.uri.indexOf('.') === -1) {
    request.uri = '/index.html';
  }
  return request;
}
  `),
});
```

Gắn vào **riêng behaviour mặc định**, dưới dạng hàm viewer-request.

{{% notice warning %}}
Cách khắc phục thông thường — một mục `errorResponses` ở cấp distribution ánh xạ
403/404 sang `/index.html` với mã 200 — **không dùng được ở đây.** Custom error
response áp dụng cho mọi behaviour chứ không theo từng behaviour.
`GET /api/products/999` sẽ trả về khung HTML với mã 200, và client sẽ hiểu đó là
thành công với body rỗng thay vì ném lỗi. Mọi mã 403 từ middleware xác thực cũng
vậy.
{{% /notice %}}

Viết lại đường dẫn ngay tại edge giữ phần fallback nằm đúng chỗ của nó — trên
behaviour SPA. Đường dẫn có chứa dấu `.` được coi là tài nguyên thật và giữ
nguyên, nên một bundle thiếu vẫn lỗi rõ ràng thay vì trả về khung ứng dụng.
CloudFront Functions tốn khoảng **$0,10 cho mỗi triệu request**.

#### Triển khai

Lần tạo đầu tiên, CloudFront mất tới **20 phút** để lan truyền cấu hình.

**Lần đầu tiên**, khi `BorrowitApp` chưa tồn tại — tham số SSM chưa có ở đó:

```powershell
cd infra
npx cdk deploy BorrowitFrontend -c albDns=none
```

Distribution dựng lên ở chế độ chỉ-SPA. Điều đó là bình thường: `BorrowitApp` ở
[3.6](../3.6-Compute/) cần bucket và distribution URL từ stack này, nên stack này
phải tồn tại trước.

**Sau khi `BorrowitApp` đã deploy**, đấu nối định tuyến API:

```powershell
npm run wire
```

`npm run wire` xoá giá trị SSM đã cache trong `cdk.context.json` rồi deploy lại
`BorrowitFrontend`. Chính bộ nhớ cache là lý do script này tồn tại: không xoá
trước thì CDK sẽ trung thành dựng lại distribution quanh hostname **cũ**.

{{% notice note %}}
Chỉ cần đấu nối lại khi load balancer bị **thay thế**, tức là có DNS name mới —
một lần `cdk destroy BorrowitApp` rồi deploy lại sẽ gây ra điều đó. Một lần deploy
backend thông thường thì không, và giá trị đã cache vẫn đúng. `npm run up` nối cả
hai bước.
{{% /notice %}}

Kiểm tra output `ApiOrigin` để xác nhận:

```powershell
aws cloudformation describe-stacks --stack-name BorrowitFrontend `
  --query "Stacks[0].Outputs[?OutputKey=='ApiOrigin'].OutputValue" --output text
```

Nếu nó hiện `(not wired - deploy BorrowitApp, then npm run wire)` thì distribution
chưa có route API, và mọi lệnh gọi `/api` sẽ nhận 403 từ S3.

#### Xuất bản bundle ứng dụng

```powershell
cd infra
npm run deploy:web
```

Một lệnh, bốn bước — đây là quy trình phát hành chứ không phải thay đổi hạ tầng,
nên nó là một script thay vì `BucketDeployment` của CDK:

1. **Build** frontend bằng `npm run build`.
2. **Đồng bộ** các tài nguyên có hash với `--cache-control public,max-age=31536000,immutable`
   và `--delete`. Tên file có hash thì cache vĩnh viễn được; `--delete` xoá các
   file từ bản build trước.
3. **Tải `index.html` riêng** với `no-cache`. File này bị loại khỏi bước đồng bộ
   ở trên vì nó là file duy nhất không bao giờ được phục vụ ở bản cũ — nó chính
   là thứ trỏ tới các hash tài nguyên hiện hành.
4. **Invalidate** distribution. 1.000 đường dẫn invalidation đầu tiên mỗi tháng
   là miễn phí.

{{% notice warning %}}
Hãy để `VITE_API_URL` **rỗng**. Đặt giá trị cho nó sẽ nhúng một hostname API
tuyệt đối vào bundle, khiến API trở lại thành khác site và kéo về cả hai lỗi ở
đầu trang này — chặn mixed content và mất cookie phiên.
{{% /notice %}}

#### Kiểm chứng

Mọi kiểm tra dưới đây đều đi qua domain CloudFront — đó chính là điểm mấu chốt.

```powershell
$env:CF = (aws cloudformation describe-stacks --stack-name BorrowitFrontend `
  --query "Stacks[0].Outputs[?OutputKey=='DistributionUrl'].OutputValue" --output text)

curl "$env:CF/health"            # qua edge tới ALB, không phải S3
curl "$env:CF/api/products"      # JSON từ cơ sở dữ liệu đã seed
curl -o /dev/null -w "%{http_code}" "$env:CF/api/products/999999"
```

Lệnh cuối cùng là bài kiểm tra hồi quy cho cái bẫy `errorResponses`: nó phải trả
về **404**, không phải 200 kèm HTML.

<!-- SCREENSHOT: /images/3-Workshop/3.7-Delivery/site-loaded.png
     Ứng dụng trên URL CloudFront, hiển thị ảnh và một phiên đã đăng nhập — cookie
     phiên còn sống chính là bằng chứng định tuyến same-origin hoạt động. -->

<!-- SCREENSHOT: /images/3-Workshop/3.7-Delivery/deep-link.png
     Một route phía client như /seller/dashboard mở trực tiếp bằng URL, cho thấy
     phần rewrite của CloudFront Function hoạt động. -->

<!-- SCREENSHOT: /images/3-Workshop/3.7-Delivery/s3-direct-blocked.png
     Một URL object S3 truy cập trực tiếp, trả về AccessDenied - bằng chứng OAC
     đang làm đúng việc của nó. -->

#### Thiết kế này vẫn chưa khắc phục điều gì

{{% notice warning %}}
Load balancer vẫn **truy cập công khai được qua `http://`**. Định tuyến
same-origin giải quyết các vấn đề của trình duyệt; nó không ngăn được ai đó biết
hostname của ALB gọi thẳng vào API, không mã hoá và bỏ qua edge.
{{% /notice %}}

Hai phương án khắc phục, không phương án nào được áp dụng ở đây:

| Phương án | Chi phí | Tác dụng |
|---|---|---|
| Giới hạn security group của ALB theo managed prefix list `com.amazonaws.global.cloudfront.origin-facing` | $0 | Chỉ CloudFront tới được ALB. Truy cập trực tiếp bị chặn |
| Mua tên miền, xin chứng chỉ ACM, thêm HTTPS listener | ~$12/năm | Mã hoá luôn cả chặng edge tới origin |

<!-- TODO(prose): phương án prefix list miễn phí và chỉ tốn khoảng bốn dòng CDK.
     Nếu bạn không làm, hãy nói vì sao - câu trả lời trung thực thường là phạm vi
     và thời hạn. Gọi tên một hạn chế kèm phương án khắc phục có tính giá đọc hay
     hơn nhiều so với để lỗ hổng không giải thích. -->

{{% notice note %}}
**Chi phí:** dưới $1/tháng. Lưu trữ và request S3 chỉ vài cent ở mức này; phí
truyền dữ liệu CloudFront với `PRICE_CLASS_200` không đáng kể cho một workload
trình diễn, và hàm định tuyến SPA thêm khoảng $0,10 cho mỗi triệu request.
{{% /notice %}}
