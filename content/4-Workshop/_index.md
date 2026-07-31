---
title: "Workshop"
date: 2026-07-28
weight: 4
chapter: false
pre: " <b> 4. </b> "
---

# Building and Operating a Web Platform on AWS

#### Overview

This workshop builds a complete production-shaped system on AWS and then operates
it: a private database, a containerised API behind a load balancer, a global
content delivery layer, and the monitoring needed to know whether any of it is
working.

The application it runs — **BorrowIt**, a peer-to-peer rental platform — is
deliberately not the subject. The application is a workload; what matters here is
which AWS services carry it, **why those services rather than the alternatives**,
how they are wired together, and how the result is observed and paid for.

By the end you will have built:

+ An **Amazon VPC** across two Availability Zones, with a documented decision to
  omit the NAT Gateway and to place isolation in security groups instead.
+ **Amazon RDS for PostgreSQL** in isolated subnets, with credentials generated
  into **AWS Secrets Manager** and injected at container start.
+ **Amazon ECS on AWS Fargate** behind an **Application Load Balancer**, with
  separate liveness and readiness checks and an automatic deployment rollback.
+ **Amazon S3** and **Amazon CloudFront** with Origin Access Control.
+ **Amazon CloudWatch** logs, metrics, a dashboard and seven alarms publishing to
  **Amazon SNS**.
+ A cost model that is verified against the real bill, not just estimated.

Everything is defined in the **AWS CDK** (TypeScript). There are no console
clicks in the build path — the console is used only to verify what the code
created.

{{% notice note %}}
Every resource lives in **`ap-southeast-1` (Singapore)**. The region appears in
console URLs and CLI commands throughout.
{{% /notice %}}

#### Architecture

![BorrowIt target architecture on AWS](/images/4-Workshop/4.1-Overview/architecture.png?width=100pc)

The architecture built here, and the editable source for the diagram above, are
set out in [2.3](../2-Proposal/2.3-Design/). This section builds it; the proposal
explains what was decided before any of it existed.

#### Content

1. [Introduction](4.1-Overview/)
2. [Choosing the AWS services](4.3-Architecture/)
3. [Prerequisites](4.2-Prerequisites/)
4. [Network foundation](4.4-Network/)
5. [Data layer](4.5-Data/)
6. [Compute layer](4.6-Compute/)
7. [Content delivery](4.7-Delivery/)
8. [Observability and operations](4.8-Observability/)
9. [Clean up](4.11-Cleanup/)
