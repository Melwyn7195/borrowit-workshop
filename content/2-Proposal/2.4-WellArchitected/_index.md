---
title : "Well-Architected review"
date : 2026-07-28
weight : 4
chapter : false
pre : " <b> 2.4. </b> "
---

The design in [2.3](../2.3-Design/) is reviewed here against the **AWS
Well-Architected Framework** — six pillars, each asking whether the architecture
holds up for a reason rather than by accident.

The review is done **before** the build, which is the point of doing it at all. A
Well-Architected review run after delivery documents what happened; one run at
proposal time changes what gets built.

{{% notice note %}}
A workload on a $160 credit budget will not score well against a framework written
for production systems, and this review does not pretend otherwise. Each pillar
below records **what the design does**, **what it knowingly gives up**, and **what
would change first** with a larger budget. The third column is the one that shows
the framework was actually applied.
{{% /notice %}}

#### The six pillars at a glance

| Pillar | Verdict | The single biggest gap |
|---|---|---|
| Operational excellence | **Strong** | No CI/CD — every deploy is manual |
| Security | **Adequate, with a named exception** | Tasks hold public IPs; API traffic is HTTP |
| Reliability | **Weak by design** | One task, one region, `removalPolicy: DESTROY` |
| Performance efficiency | **Adequate** | No load test until week 10; no auto-scaling |
| Cost optimisation | **Strong — it drove every other pillar** | ALB costs more than the compute it fronts |
| Sustainability | **Adequate** | A demo workload running 24/7 |

---

#### 1. Operational excellence

*Run and monitor systems, and continuously improve processes.*

| In the design | Mechanism |
|---|---|
| Infrastructure as code | Four CDK stacks; no console clicks in the build path |
| Small, reversible changes | `cdk diff` before every deploy; one-way stack dependencies |
| Failure is observable | Dashboard, 6 alarms, SNS email; 1 week of logs in CloudWatch |
| Deployments roll back | ECS circuit breaker reverts a failed deployment automatically |
| Documented operations | The section 3 workshop is the runbook |

**Given up:** no CI/CD pipeline, so deploys are manual and unrecorded; no automated
tests (the backend has none); no formal incident process.

**First improvement:** a CodePipeline or GitHub Actions deploy on merge. It is the
cheapest missing mechanism — pipeline cost is near zero at this frequency, and it
removes the "worked on my laptop" failure mode entirely.

#### 2. Security

*Protect data, systems and assets.*

| In the design | Mechanism |
|---|---|
| Identity | IAM task execution role and task role, scoped to the one secret and one log group |
| Credentials never written down | RDS generates the password into Secrets Manager; injected as `valueFrom` |
| Network isolation | Three chained security groups — ALB 80, task 3456, database 5432 |
| Data at rest | RDS storage encryption; S3 buckets private, reachable only through CloudFront OAC |
| Data in transit | HTTPS from the browser to CloudFront |

**Given up, and stated plainly:**

+ **Tasks run in public subnets with public IPs.** One security-group mistake is
  the difference between isolated and exposed, where a private subnet would have
  no route at all. The reasoning is in
  [3.3.3](../../3-Workshop/3.3-Architecture/3.3.3-networking/) and the cost is in
  [2.6](../2.6-Budget/).
+ **The ALB serves HTTP, not HTTPS** — there is no domain to validate a
  certificate against, so API traffic is unencrypted between browser and ALB.
+ **No WAF, GuardDuty or Config**, each of which adds recurring cost.

**First improvement:** a domain plus an ACM certificate, which is the cheapest of
the three (a certificate is free; only the domain costs money) and closes the
largest hole.

#### 3. Reliability

*Recover from failure and meet demand.*

| In the design | Mechanism |
|---|---|
| Multi-AZ foundation | Subnets in two Availability Zones; the ALB is AZ-redundant by default |
| Health checks that mean something | `/healthz` (liveness, no DB) separate from `/readyz` (readiness, checks the DB) |
| Self-healing | ECS replaces a failed task automatically |
| Data durability | Automated RDS backups; Multi-AZ enabled on demand and the failover **measured**, not assumed |
| Change failure contained | Deployment rollback on failed health checks |

**Given up:** `desiredCount: 1`, so a task failure is 1–2 minutes of downtime with
nothing to fail over to. Single region. `removalPolicy: DESTROY` on the database —
defensible only because the data is seed data.

**First improvement:** `desiredCount: 2`. It is a one-parameter change costing
about $9/month and it converts the largest reliability gap into a non-event.

#### 4. Performance efficiency

*Use computing resources efficiently as demand changes.*

| In the design | Mechanism |
|---|---|
| Right-sized compute | 0.25 vCPU / 0.5 GB — the smallest Fargate task, matched to demo traffic |
| Modern processor | `db.t4g.micro` is Graviton (ARM), better price-performance than the x86 equivalent |
| Content close to users | CloudFront caches static assets at the edge |
| No idle infrastructure | Serverless compute; nothing is provisioned for peak that is not used |
| Measurement, not assumption | Load test in week 10 establishes the actual ceiling |

**Given up:** no auto-scaling policy, so a spike is served by one container; no
performance baseline until week 10; no caching layer in front of the database.

**First improvement:** an ECS target-tracking scaling policy on CPU. It costs
nothing until it triggers.

#### 5. Cost optimisation

*Avoid unnecessary costs.*

This is the pillar that shaped the other five, so the table reads differently — the
mechanisms are the reasons the other pillars have gaps.

| In the design | Mechanism | Saves |
|---|---|---|
| No NAT Gateway | Public subnets, isolation by security group | ~$33/month |
| Single-AZ database by default | Multi-AZ behind a context flag, for the demo only | ~$16/month |
| Graviton instance | `t4g` rather than `t3` | ~10–20% on the instance |
| Storage autoscaling pinned | `maxAllocatedStorage` equals `allocatedStorage` | Prevents silent growth |
| Cost attribution | `Project` / `ManagedBy` tags applied at the CDK app level | Makes the bill readable |
| Expenditure awareness | Budget alarm at $10; estimate checked against Cost Explorer | — |
| Disposable expensive layer | One-way stack dependencies make `cdk destroy BorrowitApp` always work | ~$31/month when idle |

**Given up:** nothing was sacrificed *to* cost optimisation that is not already
named under the other pillars — which is the honest way to read this table.

**The uncomfortable line:** the ALB costs ~$17/month against ~$9 for the compute
behind it. At one task it is the least efficient item on the bill, and the
strongest argument for App Runner or an API Gateway HTTP API at this scale.

#### 6. Sustainability

*Minimise the environmental impact of running the workload.*

| In the design | Mechanism |
|---|---|
| Efficient hardware | Graviton (ARM) for the database — more work per watt than x86 |
| Right-sized, not over-provisioned | Smallest available Fargate task; storage capped at 20 GB |
| Region matched to users | `ap-southeast-1` is the nearest region to the user base; no cross-region traffic |
| Managed services | Fargate and RDS share underlying capacity more efficiently than dedicated instances would |
| Data lifecycle | Log retention capped at one week; ECR lifecycle rule removes old images |

**Given up:** a demo workload runs 24/7 for the convenience of always having
something to show. Destroying `BorrowitApp` when idle is the sustainability answer
as well as the cost answer.

---

#### Applying the framework properly

This page is a self-assessment. AWS provides the **Well-Architected Tool** in the
console (Console → Well-Architected Tool → Define workload), which walks the same
six pillars as a structured questionnaire and produces an improvement plan.

It is **free**, so it costs nothing against the credit balance.

<!-- TODO: run the Well-Architected Tool against this workload in week 9 or 10 and
     screenshot the generated improvement plan. A real tool output beside this
     hand-written review is far stronger evidence than the review alone - and any
     high-risk item the tool raises that this page missed is worth writing up
     honestly rather than quietly fixing. -->

#### The improvement plan, in priority order

Every item below is deliberately *not* being done now. This is the backlog the
review produced, ordered by benefit per dollar:

| # | Improvement | Pillar | Recurring cost |
|---|---|---|---|
| 1 | `desiredCount: 2` | Reliability | ~$9/month |
| 2 | CI/CD deploy on merge | Operational excellence | ~$0 |
| 3 | ECS target-tracking auto-scaling | Performance | $0 until it triggers |
| 4 | Domain + ACM certificate, HTTPS on the ALB | Security | Domain only |
| 5 | Private subnets + NAT Gateway | Security | ~$33/month |
| 6 | Multi-AZ RDS as the standing configuration | Reliability | ~$16/month |

Items 5 and 6 are the two that the budget, not the engineering, decided against.
If this workload were funded rather than credited, they would be items 1 and 2.
