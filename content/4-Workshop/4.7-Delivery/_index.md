---
title : "Content delivery"
date : 2026-07-28
weight : 7
chapter : false
pre : " <b> 4.7. </b> "
---

`BorrowitFrontend` holds two S3 buckets and one CloudFront distribution. At this
volume it costs **under $1/month**. The service selection rationale is in
[4.3.4](../4.3-Architecture/4.3.4-delivery-and-secrets/).

The distribution serves the compiled application **and the API**. That single
decision is what this page is really about.

#### One origin, not two

The obvious design is two hostnames: CloudFront for the static bundle, the load
balancer for the API. It does not work here, and both reasons are silent
failures rather than errors you can see:

+ **CloudFront serves HTTPS; the ALB listens only on HTTP.** A page loaded over
  HTTPS cannot call an `http://` endpoint — the browser blocks it as mixed
  content before the request is sent. ACM will not issue a certificate for an
  `elb.amazonaws.com` hostname, so there is no cheap way to put TLS on the load
  balancer directly.
+ **`userController.js` sets the session cookie with `sameSite: 'strict'`.**
  Point the frontend at a cross-site API host and the browser silently drops
  that cookie. Login returns 200, and every request after it is anonymous.

Routing the API through the same distribution makes it **same-origin**, which
fixes both at once — and removes the need for CORS on the browser path
entirely.

{{% notice note %}}
The consequence for the frontend build: the API is called on **relative paths**,
and `VITE_API_URL` is deliberately **empty**. There is no API hostname anywhere
in the bundle. In development, Vite proxies `/api` to `localhost:3456`; in
production, CloudFront routes it to the load balancer. Same code, both
environments.
{{% /notice %}}

#### The behaviours

| Behaviour | Origin | Serves |
|---|---|---|
| Default (`*`) | Web bucket (OAC) | The compiled application bundle |
| `/uploads/*` | Uploads bucket (OAC) | User-uploaded images |
| `/api/*` | Application Load Balancer | The REST API |
| `/health` | Application Load Balancer | Readiness probe, reachable from the edge |
| `/api-docs*` | Application Load Balancer | Swagger UI and its assets |

The `/uploads/*` prefix maps directly onto the S3 object key prefix — the
application writes objects under `uploads/`, so no rewrite rule is needed.

`/api-docs*` has no slash before the wildcard on purpose: it has to match both
`/api-docs` and the assets Swagger loads from `/api-docs/`. It does not overlap
`/api/*`, which does require the slash.

#### The API behaviour

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

Every one of those four settings is load-bearing:

| Setting | Why |
|---|---|
| `HTTP_ONLY` | The ALB has no certificate. This hop runs inside AWS between the edge and the load balancer; the browser half is still HTTPS, which is what the mixed-content rule and the secure cookie flag care about |
| `ALLOW_ALL` | This is a write API. POST, PUT and DELETE all have to reach the origin — CloudFront's read-only default would reject them |
| `CACHING_DISABLED` | Never cache an authenticated response at a shared edge. Two users requesting `/api/users/me` must not see each other's answer |
| `ALL_VIEWER` | Forwards cookies, query strings and headers to the origin. The session cookie **is** the authentication mechanism; anything less logs every request out |

#### Finding the load balancer

`BorrowitFrontend` needs the ALB's DNS name, which only exists once
`BorrowitApp` is deployed. Importing it would reverse the dependency direction
and break `cdk destroy BorrowitApp` ([3.1](../4.1-Overview/)), so it is passed
out of band instead: AppStack **publishes** it to an SSM parameter, and
FrontendStack **looks it up**.

```typescript
// In BorrowitApp - standard parameters are free.
new ssm.StringParameter(this, 'AlbDnsParameter', {
  parameterName: '/borrowit/alb-dns',
  stringValue: alb.loadBalancerDnsName,
});

// In BorrowitFrontend - resolved at synth time.
const looked = ssm.StringParameter.valueFromLookup(this, '/borrowit/alb-dns');
const albDns = looked.startsWith('dummy-value-for-') ? undefined : looked;
```

{{% notice warning %}}
The lookup is **synth-time on purpose**. A deploy-time `{{resolve:ssm:...}}`
reference looks tidier and is wrong: it leaves the template unchanged, so when
the load balancer is replaced CloudFormation reports "no changes" and quietly
keeps routing to a load balancer that no longer exists. Resolving at synth bakes
the literal hostname into the template, so a replacement genuinely changes it.
{{% /notice %}}

Context still overrides the lookup, for two cases:

```powershell
# First deploy, before BorrowitApp exists - skip the API behaviours entirely.
npx cdk deploy BorrowitFrontend -c albDns=none

# Force a specific hostname, ignoring any cached lookup.
npx cdk deploy BorrowitFrontend -c albDns=borrowit-alb-123.ap-southeast-1.elb.amazonaws.com
```

This used to be a bare context flag with no SSM fallback, and it was a trap: any
deploy that forgot the flag silently dropped the `/api/*` behaviours, and the
symptom was every API call returning **HTTP 200 with the SPA shell** instead of
an error. Reading from Parameter Store means the routing survives a plain
`cdk deploy BorrowitFrontend`.

#### Serving a single-page application

A client-side route such as `/seller/dashboard` has no matching S3 object.
Because Origin Access Control grants `GetObject` but not `ListBucket`, S3 answers
**403**, not 404.

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

Attached to the **default behaviour only**, as a viewer-request function.

{{% notice warning %}}
The conventional fix — a distribution-level `errorResponses` entry mapping
403/404 to `/index.html` with status 200 — **cannot be used here.** Custom error
responses apply to every behaviour, not per-behaviour. `GET /api/products/999`
would come back as the HTML app shell with status 200, and the client would read
that as success with a null body rather than throwing. Every 403 from the auth
middleware would do the same.
{{% /notice %}}

Rewriting the path at the edge keeps the fallback on the SPA behaviour where it
belongs. A path containing a `.` is treated as a real asset and left alone, so a
missing bundle still fails visibly instead of returning the app shell. CloudFront
Functions cost about **$0.10 per million requests**.

#### Deploy

CloudFront takes up to **20 minutes** to propagate on first creation.

**First time**, before `BorrowitApp` exists — the SSM parameter is not there yet:

```powershell
cd infra
npx cdk deploy BorrowitFrontend -c albDns=none
```

The distribution comes up SPA-only. That is expected: `BorrowitApp` in
[3.6](../4.6-Compute/) needs the bucket and distribution URL from this stack,
so it has to exist first.

**After `BorrowitApp` is deployed**, wire the API routing:

```powershell
npm run wire
```

`npm run wire` clears the cached SSM lookup from `cdk.context.json` and
redeploys `BorrowitFrontend`. The cache is the reason it exists: without
clearing it, CDK would faithfully rebuild the distribution around the **old**
hostname.

{{% notice note %}}
Wiring is only needed when the load balancer has been **replaced**, which gives
it a new DNS name — a full `cdk destroy BorrowitApp` / redeploy does that. A
routine backend deploy does not, and the cached value stays correct.
`npm run up` chains both steps.
{{% /notice %}}

Check the `ApiOrigin` output to confirm:

```powershell
aws cloudformation describe-stacks --stack-name BorrowitFrontend `
  --query "Stacks[0].Outputs[?OutputKey=='ApiOrigin'].OutputValue" --output text
```

If it reads `(not wired - deploy BorrowitApp, then npm run wire)`, the
distribution has no API route and every `/api` call will 403 out of S3.

#### Publish the application bundle

```powershell
cd infra
npm run deploy:web
```

One command, four steps — it is a release process, not an infrastructure change,
which is why it is a script rather than a CDK `BucketDeployment`:

1. **Builds** the frontend with `npm run build`.
2. **Syncs** the hashed assets with `--cache-control public,max-age=31536000,immutable`
   and `--delete`. Hashed filenames can be cached forever; `--delete` removes
   files from previous builds.
3. **Uploads `index.html` separately** with `no-cache`. It is excluded from the
   sync above because it is the one file that must never be served stale — it is
   what points at the current asset hashes.
4. **Invalidates** the distribution. The first 1,000 invalidation paths per month
   are free.

{{% notice warning %}}
Leave `VITE_API_URL` **empty**. Setting it bakes an absolute API hostname into
the bundle, which makes the API cross-site again and brings back both failures
from the top of this page — the mixed-content block and the dropped session
cookie.
{{% /notice %}}

#### Verify

Every check below goes through the CloudFront domain — that is the point.

```powershell
$env:CF = (aws cloudformation describe-stacks --stack-name BorrowitFrontend `
  --query "Stacks[0].Outputs[?OutputKey=='DistributionUrl'].OutputValue" --output text)

curl "$env:CF/health"            # through the edge to the ALB, not S3
curl "$env:CF/api/products"      # JSON from the seeded database
curl -o /dev/null -w "%{http_code}" "$env:CF/api/products/999999"
```

That last one is the regression test for the `errorResponses` trap: it must
return **404**, not 200 with HTML.

<!-- SCREENSHOT: /images/4-Workshop/4.7-Delivery/site-loaded.png
     The application on the CloudFront URL, with images visible and a logged-in
     session - the session cookie surviving is the proof that same-origin
     routing works. -->

<!-- SCREENSHOT: /images/4-Workshop/4.7-Delivery/deep-link.png
     A client-side route like /seller/dashboard loaded directly by URL, showing
     the CloudFront Function rewrite working. -->

<!-- SCREENSHOT: /images/4-Workshop/4.7-Delivery/s3-direct-blocked.png
     An S3 object URL accessed directly, returning AccessDenied - proof that OAC
     is doing its job. -->

#### What this design still does not fix

{{% notice warning %}}
The load balancer remains **publicly reachable on `http://`**. Same-origin
routing solves the browser's problems; it does not stop anyone who knows the ALB
hostname from calling the API directly, unencrypted and bypassing the edge.
{{% /notice %}}

Two remedies, neither adopted here:

| Remedy | Cost | Effect |
|---|---|---|
| Restrict the ALB security group to the `com.amazonaws.global.cloudfront.origin-facing` managed prefix list | $0 | Only CloudFront can reach the ALB. Direct access stops |
| Register a domain, request an ACM certificate, add an HTTPS listener | ~$12/year | Encrypts the edge-to-origin hop as well |

<!-- TODO(prose): the prefix-list option is free and takes about four lines of
     CDK. If you did not do it, say why - the honest answer is usually scope and
     the deadline. Naming a limitation with a costed remedy reads far better than
     leaving the gap unexplained. -->

{{% notice note %}}
**Cost:** under $1/month. S3 storage and requests are pennies at this volume;
CloudFront data transfer at `PRICE_CLASS_200` is negligible for a demonstration
workload, and the SPA routing function adds ~$0.10 per million requests.
{{% /notice %}}
