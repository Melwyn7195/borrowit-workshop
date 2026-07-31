---
title : "Architecture design"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 2.3. </b> "
---

The design follows from the requirements in [2.2](../2.2-Requirements/) and from
one constraint that dominates all others: a fixed **$160 credit balance**. Every
box below was chosen with a monthly price attached to it.

#### The target architecture

{{<mermaid align="center">}}
graph LR
  U["Browser clients"]
  CF["CloudFront distribution"]
  S3W["S3 - web client"]
  S3U["S3 - user uploads"]
  ECR["ECR - images"]
  SM["Secrets Manager"]
  CW["CloudWatch"]
  SNS["SNS - email"]

  subgraph Public subnets in two AZs
    ALB["Application Load Balancer"]
    T["ECS Fargate task - 0.25 vCPU"]
    VPE["Interface VPC endpoints - PrivateLink"]
  end

  subgraph Isolated subnets in two AZs
    DB["RDS PostgreSQL 16 - db.t4g.micro"]
  end

  U -->|HTTPS| CF
  CF -->|OAC| S3W
  CF -->|OAC| S3U
  U -->|HTTP port 80| ALB
  ALB -->|port 3456| T
  T -->|port 5432| DB
  T -. port 443 .-> VPE
  VPE -. image pull .-> ECR
  VPE -. credentials at start .-> SM
  VPE -. logs and metrics .-> CW
  CW -. alarm .-> SNS
{{</mermaid>}}

Solid arrows are the request path; dashed arrows are control, telemetry and
deployment paths. Both subnet groups sit inside a single VPC with **no NAT
Gateway** — the boundary the flat diagram above cannot draw, and the reason the
draw.io version below exists.

Note where `VPE` sits: every call the task makes to an AWS service goes *through*
it, not around it. That is the whole point of an interface endpoint — it is not a
box hanging off the side of the diagram, it is the door in the VPC wall that the
ECR, Secrets Manager and CloudWatch traffic leaves by. Five endpoints are created
(`ecr.api`, `ecr.dkr`, `logs`, `secretsmanager`, `ssmmessages`) and they are part
of the deployed architecture rather than an option, at ~$36/month in one AZ. The
arithmetic is in [4.3.3](../../4-Workshop/4.3-Architecture/4.3.3-networking/).

#### The editable diagram

The diagram above is generated from the page source, which makes it easy to keep
correct but limits it to boxes and arrows. The **presentation version** — official
AWS service icons, the AWS Cloud / Region / VPC / subnet boundaries drawn in the
standard AWS reference-architecture style — lives as an editable draw.io file:

&emsp; 📐 **[borrowit-architecture.drawio](../../diagrams/borrowit-architecture.drawio)**

{{% notice tip %}}
**Editing it.** Open [app.diagrams.net](https://app.diagrams.net) → *File → Open
From → Device*, or install the **Draw.io Integration** extension for VS Code and
double-click the file in the repository. The AWS shapes are already embedded in
the file, but to add new ones enable *More Shapes → Networking → AWS 2025*.
Export with *File → Export as → PNG*, `zoom 200%`, transparent background off, to
`static/images/2-Proposal/2.3-Design/architecture.png`.
{{% /notice %}}

![Target architecture](/images/2-Proposal/2.3-Design/architecture.png?width=100pc)

#### Component inventory

| Service | Role | Sizing proposed | Stack |
|---|---|---|---|
| **VPC** | Two AZs, public + isolated subnets, no NAT | `/16`, four `/24` subnets | `BorrowitFoundation` |
| **ECR** | Container image registry | Scan on push, lifecycle rule | `BorrowitFoundation` |
| **Security groups** | The isolation boundary, in place of private subnets | Three, chained | `BorrowitFoundation` |
| **VPC endpoints** | S3 gateway plus five interface endpoints, both always on | Gateway free; interface ~$36/mo in one AZ | `BorrowitFoundation` |
| **RDS PostgreSQL 16** | Relational store | `db.t4g.micro`, 20 GB, single-AZ | `BorrowitData` |
| **Secrets Manager** | DB credentials, generated not written | One secret, RDS-managed | `BorrowitData` |
| **S3** | Web client bundle, user uploads | Two buckets, both private | `BorrowitFrontend` |
| **CloudFront** | HTTPS and global cache in front of S3 | One distribution, OAC | `BorrowitFrontend` |
| **ECS on Fargate** | The API process | 1 task, 0.25 vCPU / 0.5 GB | `BorrowitApp` |
| **Application Load Balancer** | Stable entry point, health checks | Internet-facing, port 80 | `BorrowitApp` |
| **CloudWatch** | Logs, metrics, dashboard, 6 alarms | 1 week retention | `BorrowitApp` |
| **SNS** | Alarm delivery | One topic, email subscription | `BorrowitApp` |

#### Request paths

Three paths, and the design is easiest to check by tracing each one:

1. **Static content.** Browser → CloudFront → S3. The bucket stays private; only
   the distribution can read it, through Origin Access Control. Nothing here
   touches the VPC.
2. **API call.** Browser → ALB on port 80 → target group → Fargate task on port
   3456 → RDS on 5432. Each hop is permitted by exactly one security group rule,
   and each rule references the *previous security group* rather than a CIDR
   block, so it survives a task being replaced.
3. **Startup.** The task pulls its image from ECR, reads the database password
   from Secrets Manager, and starts streaming to CloudWatch — each over an
   interface endpoint, so none of it leaves the VPC. The task still holds a public
   IP, because dropping it would mean any AWS call without an endpoint hangs
   silently at startup. That combination — public IP, private AWS traffic — is the
   design's most contested decision and gets its full treatment in
   [4.3.3](../../4-Workshop/4.3-Architecture/4.3.3-networking/).

#### Stack topology

Four stacks, with dependencies pointing **one way only**:

{{<mermaid align="center">}}
graph TD
  F["BorrowitFoundation - VPC, security groups, ECR"]
  D["BorrowitData - RDS, Secrets Manager"]
  W["BorrowitFrontend - S3, CloudFront"]
  A["BorrowitApp - ALB, Fargate, CloudWatch, SNS"]
  F --> D
  F --> A
  D --> A
  W --> A
{{</mermaid>}}

Nothing imports from `BorrowitApp`. That is a deliberate structural choice, not an
accident of ordering: it means **`cdk destroy BorrowitApp` always succeeds**,
because CloudFormation never has to refuse over an exported value another stack is
consuming. The expensive layer is therefore the disposable one — see
[2.6](../2.6-Budget/).

#### Design decisions, summarised

Each row is a decision with a cheaper or safer alternative that was rejected. The
full reasoning, including what each choice costs, is in
[4.3](../../4-Workshop/4.3-Architecture/).

| Layer | Proposed | Main alternative | Deciding factor |
|---|---|---|---|
| Compute | ECS on Fargate | Lambda, App Runner, EC2 | Long-lived process with a DB connection pool |
| Database | RDS PostgreSQL | Aurora Serverless v2, DynamoDB | Existing relational schema; predictable fixed cost |
| Outbound networking | Public subnets, no NAT, interface endpoints on | NAT Gateway | NAT is ~$33/month and serves *all* egress; endpoints cost ~$36 and cover only the AWS calls that matter |
| Isolation | Security groups | Private subnets | Cost; accepted as thinner defence in depth |
| Static delivery | S3 + CloudFront | Amplify Hosting, serving from the ALB | Cost, and keeping buckets private |
| Secrets | Secrets Manager | SSM Parameter Store | RDS generates and stores the password natively |
| Monitoring | CloudWatch + SNS | Container Insights, X-Ray | Enough signal at ~$3.60/month |

#### Known weaknesses of this design

Stated now, at proposal time, rather than discovered in the assessment:

+ **One task is a single point of failure.** The ALB has nothing to fail over to.
  Mitigated only by ECS restarting the task, which takes 1–2 minutes.
+ **The ALB costs more than the compute behind it** — ~$17 against ~$9. At this
  scale it is the least defensible line item on the bill, and
  [2.6](../2.6-Budget/) says so.
+ **API traffic is HTTP, not HTTPS**, because there is no domain to validate a
  certificate against. Acceptable for a demo; unacceptable for real users.
+ **Public IPs on tasks** trade defence in depth for the NAT saving, and cost
  ~$3.60/month per address. The interface endpoints narrow this considerably — the
  ECR, Secrets Manager, CloudWatch and SSM traffic never uses the public path —
  but they do **not** remove the public IP, which is still required.
+ **The endpoints are the second-largest line on the bill**, at ~$36/month against
  a $160 credit balance, and they sit in `BorrowitFoundation`, which is never
  destroyed. Unlike the ALB, this cost keeps accruing while the application layer
  is torn down — see [2.6](../2.6-Budget/).

<!-- TODO(prose): after the build, come back and add a short "what changed" note
     here. Any design that survives twelve weeks of contact with reality untouched
     was probably not tested hard enough. -->
