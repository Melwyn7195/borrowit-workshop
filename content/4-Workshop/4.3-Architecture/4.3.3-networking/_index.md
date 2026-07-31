---
title : "Networking — no NAT Gateway"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 4.3.3 </b> "
---

This is the decision most likely to be challenged, so it gets the fullest
treatment.

#### The problem

A Fargate task needs **outbound** internet access to function at all:

+ pull its image from Amazon ECR,
+ read the database credentials from AWS Secrets Manager,
+ ship logs to Amazon CloudWatch.

It needs **no inbound** access from the internet — the load balancer is the only
thing that should reach it. The question is how to give it the first without the
second.

#### The candidates

| Option | Monthly cost | Inbound exposure | Notes |
|---|---|---|---|
| **Private subnets + NAT Gateway** | ~$33 + $0.045/GB | None — no route exists | The textbook answer |
| **Private subnets + VPC interface endpoints** | ~$7.30/endpoint/AZ | None | Needs 5 endpoints ≈ $73/month across two AZs |
| **Public subnets + public IP only** | **$0** | Security group only | Cheapest; all AWS calls cross the public path |
| **Public subnets + interface endpoints** | ~$36/month in one AZ | Security group only | **What this project deploys** |

#### Why not a NAT Gateway

A NAT Gateway costs approximately **$33/month** in `ap-southeast-1` before data
processing charges. The entire compute budget for this project is around
$35/month. Adding a NAT would nearly double the cost of the system to solve a
problem that security groups also solve.

On a fixed $160 credit balance, a NAT Gateway alone would consume most of the
runway.

#### Interface endpoints — deployed, and what they cost

The NAT-free answer to the outbound problem is **AWS PrivateLink interface
endpoints** — traffic to ECR, Secrets Manager, CloudWatch and SSM never leaves the
AWS network, and no internet route is needed for it.

They are deployed in `BorrowitFoundation` as part of the architecture, not behind a
flag. That is a paid decision and the arithmetic is worth stating plainly.
Interface endpoints bill **per endpoint per Availability Zone per hour** —
$0.01/hour in `ap-southeast-1`, so **$7.30 per endpoint per AZ per month**. This
workload needs five:

| Endpoint | Why the task needs it |
|---|---|
| `ecr.api` | Authorises the image pull |
| `ecr.dkr` | Serves the image manifest |
| `logs` | Ships container logs to CloudWatch |
| `secretsmanager` | Reads the database password at task start |
| `ssmmessages` | Backs `enableExecuteCommand` — `aws ecs execute-command` |

Five endpoints across two AZs is **ten endpoint-AZ combinations, ~$73/month** —
worse than the NAT Gateway it was meant to avoid. **Pinned to a single AZ it is
~$36/month**, and that is what deploys by default:

```powershell
npx cdk deploy BorrowitFoundation     # one AZ,   ~$36/month
npm run endpoints:ha                  # both AZs, ~$73/month
```

Single AZ is the default because private DNS returns every ENI the endpoint has
and a cross-AZ call inside a VPC still works — one AZ halves the bill in exchange
for a dependency on that AZ staying up. `-c vpcEndpoints=ha` removes that
dependency when a demonstration needs it.

{{<mermaid align="center">}}
graph LR
  T["Fargate task"]
  VPE["Interface endpoints - ENIs inside the VPC"]
  AWS["ECR, Secrets Manager, CloudWatch, SSM"]
  IGW["Internet gateway"]
  NET["Everything else"]

  T -->|443, private DNS| VPE
  VPE -->|stays on the AWS network| AWS
  T -. no endpoint, so still public .-> IGW
  IGW -.-> NET
{{</mermaid>}}

The dashed branch is what the endpoints do **not** cover. The task keeps its public
IP and its default route, so any call to something without an endpoint still leaves
through the internet gateway. What changed is that the four services on the startup
and telemetry path — the ones holding credentials, images and logs — no longer do.

Two details make the difference between a real control and an expensive no-op:

+ **`privateDnsEnabled: true`.** Without it the regional service hostname still
  resolves to a public address and the traffic keeps using the internet gateway —
  you would be paying for endpoints nobody uses.
+ **A dedicated security group.** `addInterfaceEndpoint` otherwise generates one
  allowing the whole VPC CIDR on 443. The endpoints here accept 443 from
  `ServiceSecurityGroup` only, so the tasks are the sole callers.

{{% notice warning %}}
Having these does **not** let you remove `assignPublicIp: true`. Dropping the
public IP means every AWS call the task makes must have an endpoint, and a missing
one shows up as a task hanging at startup rather than failing with a clear error.
The security boundary is still the security group chain, not the endpoints.
{{% /notice %}}

{{% notice warning %}}
**This charge does not go away when you destroy the app.** The endpoints live in
`BorrowitFoundation`, which is never torn down, so `cdk destroy BorrowitApp` now
drops the run rate to ~$53/month rather than ~$17. That is the real cost of making
them mandatory.
{{% /notice %}}

{{% notice note %}}
The **S3 gateway endpoint** is a different thing entirely and is **always** on —
gateway endpoints are **free**. It is added in [3.4](../../4.4-Network/), and it is
load-bearing rather than decorative: ECR image layers are stored in S3, so it
carries part of every task startup.
{{% /notice %}}

#### The decision

Fargate tasks run in **public subnets with `assignPublicIp: true`**, inbound
isolation is enforced by security groups, and outbound AWS traffic goes over
**interface endpoints** so it never crosses the public internet.

```typescript
this.vpc = new ec2.Vpc(this, 'Vpc', {
  maxAzs: 2,
  natGateways: 0,
  subnetConfiguration: [
    { name: 'public', subnetType: ec2.SubnetType.PUBLIC, cidrMask: 24 },
    { name: 'isolated', subnetType: ec2.SubnetType.PRIVATE_ISOLATED, cidrMask: 24 },
  ],
});
```

The security groups form a chain, each accepting traffic only from the one before
it:

| Group | Accepts from | Port |
|---|---|---|
| `AlbSecurityGroup` | `0.0.0.0/0` | 80 |
| `ServiceSecurityGroup` | `AlbSecurityGroup` | 3456 |
| `DatabaseSecurityGroup` | `ServiceSecurityGroup` | 5432 |

Referencing a **security group** rather than a CIDR block is what makes this
durable: the rule remains correct when ECS replaces a task and the public IP
changes.

#### A public IP is not a public service

The task holds a routable address, but nothing on the internet can open a
connection to it — inbound packets are dropped unless they originate from the load
balancer's security group.

This is demonstrated rather than asserted in
[3.6.3](../../4.6-Compute/4.6.3-load-balancer/), by curling the task's public IP
directly and watching it time out.

#### What this choice costs

+ **Defence in depth is thinner.** One security group misconfiguration exposes the
  task directly to the internet. In the private-subnet design the same mistake
  exposes nothing, because no route exists. This is a real reduction in safety
  margin, not a cosmetic one.
+ **Each task consumes a public IPv4 address**, which AWS now charges for
  (~$3.60/month per address) and which matters at larger scale.
+ **It fails a compliance audit** that mandates no public IPs on workloads.

<!-- TODO(prose): state plainly what you would do with a larger budget. The honest
     answer is private subnets behind a NAT Gateway. A workshop that presents a
     budget workaround as a best practice is misleading; one that names the
     tradeoff is trustworthy. -->

#### When each design is right

| | No NAT (public subnets) | No NAT + interface endpoints | NAT Gateway (private subnets) |
|---|---|---|---|
| Cost | $0 | **~$36/month, one AZ** | ~$33/month + data processing |
| Isolation | Security group only | Security group; AWS traffic off the public path | Security group **and** no route |
| Public IPv4 charges | Per task | Per task — the IP is still required | None |
| Appropriate for | Demos, hard budget ceilings | Small projects that must keep AWS traffic private | Production, regulated workloads |

The middle column is what this project deploys. It is worth being precise about
what that buys and what it does not: the credentials, images and logs stop
traversing the public internet, which is a genuine control and a demonstrable one.
The task is still addressable from the internet at the routing layer, and only the
security group stops a packet reaching it. Endpoints improve confidentiality of the
outbound path; they do not restore the defence in depth that a private subnet gives
you on the inbound path.

For a workload that genuinely cannot have a public IP, the right answer is still
the third column, and no amount of endpoint spending substitutes for it.
