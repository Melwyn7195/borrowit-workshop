---
title : "Prerequisites"
date : 2026-07-28
weight : 2
chapter : false
pre : " <b> 4.2. </b> "
---

#### Tools

| Tool | Version | Used for |
|---|---|---|
| AWS CLI | v2 | Authentication, ECR login, reading stack outputs |
| Node.js | 22 | The CDK application |
| Docker Desktop | recent | Building the API image — must be **running** |
| Session Manager plugin | latest | `aws ecs execute-command` in [3.6.4](../4.6-Compute/4.6.4-database-init/) and [3.8.4](../4.8-Observability/4.8.4-debugging/) |

The Session Manager plugin installs separately from the AWS CLI:
[installation guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html).

#### AWS account

Confirm the CLI is authenticated and pointed at the right account:

```powershell
aws sts get-caller-identity
```

Check the credit balance before deploying anything that bills:

```powershell
aws freetier get-account-plan-state
```

![Account plan state showing the remaining credit balance](/images/4-Workshop/4.2-Prerequisites/account-plan.png?width=100pc)

{{% notice warning %}}
**Set up a budget alarm before deploying.** Billing → Budgets → create a $10 budget
with an email notification. An Application Load Balancer bills approximately
$0.0225/hour whether or not any request reaches it, so a stack left running by
mistake is a real cost against a fixed credit balance.
{{% /notice %}}

#### IAM permissions

Deploying this workshop requires permissions across CloudFormation, EC2/VPC, ECS,
ECR, RDS, S3, CloudFront, Secrets Manager, CloudWatch, SNS, IAM and SSM.

<!-- TODO: if you used an administrator identity, say so plainly rather than
     implying a least-privilege policy you did not actually construct. If you did
     build a scoped policy, paste it here - that is a genuinely strong artefact. -->

#### Bootstrap CDK

CDK needs a small set of resources in the account before it can deploy anything —
an S3 bucket for assets, an ECR repository, and several IAM roles. This runs
**once per account and region**.

```powershell
cd infra
npm install

$env:ACCOUNT = (aws sts get-caller-identity --query Account --output text)
npx cdk bootstrap "aws://$env:ACCOUNT/ap-southeast-1"
```

Bootstrapping creates a CloudFormation stack named `CDKToolkit`, visible in the
[CloudFormation console](https://ap-southeast-1.console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks).

#### Inspect before deploying

`cdk synth` compiles the TypeScript into CloudFormation templates without creating
anything. It is the cheapest way to catch a mistake.

```powershell
npx cdk synth -c albDns=none
npx cdk list
```

`cdk list` should print the four stacks:

```
BorrowitFoundation
BorrowitData
BorrowitFrontend
BorrowitApp
```

{{% notice note %}}
`-c albDns=none` is required on a **first** synth or deploy. `BorrowitFrontend`
looks up the load balancer's DNS name from the SSM parameter `/borrowit/alb-dns`,
which `BorrowitApp` publishes — and on a fresh account that parameter does not
exist yet. The flag tells the frontend stack to skip its API routes for now.
Section [3.7](../4.7-Delivery/) explains the whole arrangement.
{{% /notice %}}

#### Deployment order

The stacks deploy in dependency order. `BorrowitApp` comes last: it imports from
all three of the others, and it will not deploy until an image exists in ECR.

```
3.4  BorrowitFoundation    VPC, security groups, ECR
3.5  BorrowitData          RDS, Secrets Manager        (~10 min)
3.7  BorrowitFrontend      S3, CloudFront              (~20 min)  -c albDns=none
3.6  BorrowitApp           ECS, Fargate, ALB, alarms   (~5 min)
3.7  npm run wire          point CloudFront at the ALB (~5 min)
```

Two things about that order are worth noticing before you start, because both
are easy to trip over:

+ **The section numbers are not sequential.** `BorrowitFrontend` (3.7) is
  deployed **before** `BorrowitApp` (3.6), because AppStack needs the uploads
  bucket and the distribution URL as inputs. You will still *read* 3.6 before
  3.7 — the compute layer is where the interesting decisions are — but the
  deploy in 3.6.2 will pull `BorrowitFrontend` up with it if you have not
  deployed it already.
+ **The dependency is circular in appearance only.** FrontendStack needs the ALB
  hostname; AppStack needs the distribution URL. It is resolved by splitting the
  frontend deploy in two: once without API routes, then again — via
  `npm run wire` — once the load balancer exists. Nothing imports from
  `BorrowitApp` at any point, which is what keeps `cdk destroy BorrowitApp`
  working.
