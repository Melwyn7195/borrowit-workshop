---
title : "Compute — Fargate over Lambda"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 4.3.1 </b> "
---

Five AWS services could run this API. This is the reasoning that selected one.

#### The candidates

| Option | What you manage | Billing model | Cost at this scale |
|---|---|---|---|
| **AWS Lambda** | Function code only | Per request + GB-second | ~$0 at low traffic |
| **ECS on Fargate** | Container image | Per vCPU-second + GB-second, while running | ~$9/month (0.25 vCPU, 0.5 GB) |
| **ECS on EC2** | Container image + the instances | Per instance-hour | ~$8/month (t4g.small) |
| **EC2 directly** | OS, runtime, deployment, patching | Per instance-hour | ~$8/month |
| **App Runner** | Container image | Per vCPU/GB with a provisioned floor | ~$25/month minimum |

On price alone, Lambda wins outright. It was still rejected.

#### Why not Lambda

Lambda is the reflexive answer for a low-traffic API, and for a greenfield
event-driven design it would likely be right. Four properties of *this* workload
argued against it:

**1. The database connection model.** The API is an Express server using a
PostgreSQL connection pool. Lambda's execution model gives each concurrent
invocation its own container, and each one opens its own connections. A
`db.t4g.micro` supports roughly 80–100 connections; a modest concurrency spike
exhausts that. The standard fix is **RDS Proxy**, which costs about
**$11/month per vCPU of the database instance** — turning the cheapest option into
one comparable to Fargate, purely to solve a problem Fargate does not have.

**2. It is a long-lived server, not a set of handlers.** Porting Express to Lambda
means either adapting every route to a handler signature, or wrapping the whole
app in an adapter so a full HTTP server boots on each cold start. The first is a
rewrite the timeline did not allow; the second keeps Lambda's costs while
discarding most of its benefits.

**3. Cold starts on a user-facing path.** A cold Node.js container with a fresh
database connection adds noticeable latency to the first request. Provisioned
concurrency removes it — and reintroduces a fixed hourly charge, again converging
on Fargate's price.

**4. Predictability of the bill.** Lambda's cost scales with traffic. On a fixed,
non-replenishing credit balance, a runaway loop or a crawler is a budget incident.
Fargate at `desiredCount: 1` costs the same every month regardless of what
happens, which on this budget is worth more than a lower expected value.

<!-- TODO(prose): if you actually prototyped a Lambda version before deciding,
     say so and give the number you measured. A decision backed by a measurement
     is worth far more than one backed by reasoning alone. -->

#### Why not EC2 or ECS on EC2

Both are marginally cheaper than Fargate and both hand back work: OS patching,
AMI lifecycle, capacity planning, and — for ECS on EC2 — running the container
agent and sizing instances to fit tasks.

The saving is roughly **$1–2/month**. The cost is every hour spent on
undifferentiated maintenance, on a project with a hard deadline. Fargate's premium
buys the removal of an entire operational category.

#### Why not App Runner

App Runner is genuinely the simplest option — push an image, receive an HTTPS URL
with a valid certificate, which would also have solved the TLS problem described
in [3.7](../../4.7-Delivery/). It was rejected on two grounds: its provisioned
floor makes it the **most expensive** option here, and it abstracts away the VPC,
load balancer and target group configuration that this workshop exists to
demonstrate.

#### The decision

**ECS on Fargate**, 0.25 vCPU / 0.5 GB, one task.

| Criterion | Weight | Why Fargate wins |
|---|---|---|
| Runs the existing app unmodified | High | Container, no code changes |
| Connection pooling to RDS | High | One long-lived process, one pool |
| No servers to patch | High | AWS manages the host |
| Predictable monthly cost | High | Fixed while running |
| Absolute lowest cost | Medium | Loses to Lambda and EC2 |
| Simplest possible operation | Low | Loses to App Runner |

#### What this choice costs

Being explicit about the downside matters more than defending the decision:

+ **You pay while idle.** At 3 a.m. with no traffic, Fargate bills the same as at
  peak. Lambda would not.
+ **Scaling is coarse.** Adding a task adds a whole 0.25 vCPU; Lambda scales per
  request.
+ **Startup is slow.** A new task takes 60–90 seconds to become healthy, which is
  why the autoscaling threshold is set to 65% CPU
  rather than 80% — the headroom exists to absorb load arriving while a task boots.

{{% notice note %}}
**Cost:** one Fargate task at 0.25 vCPU / 0.5 GB running continuously is
approximately **$9/month** in `ap-southeast-1`. The load balancer in front of it
costs roughly twice that — see [3.6.3](../../4.6-Compute/4.6.3-load-balancer/).
{{% /notice %}}
