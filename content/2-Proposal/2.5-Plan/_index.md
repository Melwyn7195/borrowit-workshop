---
title : "Implementation plan"
date : 2026-07-28
weight : 5
chapter : false
pre : " <b> 2.5. </b> "
---

The plan follows the **dependency order of the stacks**, not the order of the
architecture diagram. Nothing can be built before the thing it imports from
exists, and that single rule fixes the sequence.

#### Phases

| Phase | Weeks | Outcome | Gate before moving on |
|---|---|---|---|
| **0 — Foundations** | 1–3 | AWS fundamentals, account and budget set up, this proposal | Proposal reviewed and the architecture agreed |
| **1 — Network and data** | 4–5 | `BorrowitFoundation` and `BorrowitData` deployed | `psql` reaches the database from inside the VPC and nowhere else |
| **2 — Compute** | 6–7 | Image in ECR, `BorrowitApp` serving through the ALB | `GET /healthz` returns 200 through the load balancer |
| **3 — Delivery** | 8 | `BorrowitFrontend`, web client live on CloudFront | The app works end to end from a browser |
| **4 — Operations** | 9–10 | Dashboard, alarms, Multi-AZ failover test, cost review | An alarm email arrives from a deliberately broken system |
| **5 — Documentation** | 11–12 | The section 4 workshop, blogs, final report | A reader can rebuild the system from the workshop alone |

#### Week by week

| Week | Work | Deliverable | How it is verified |
|---|---|---|---|
| 1 | AWS fundamentals; account, MFA, budget alarm, cost tags | Account ready | `aws freetier get-account-plan-state` shows the credit balance |
| 2 | IAM, VPC, EC2, S3 by hand — before automating any of it | Notes and scratch resources | Scratch resources deleted; nothing left billing |
| 3 | Assess what BorrowIt depends on; design the target; write this proposal | **This document** | Architecture diagram and cost model reviewed |
| 4 | CDK bootstrap; `BorrowitFoundation` — VPC, security groups, ECR | Stack deployed | `cdk diff` clean; subnets and routes match the design |
| 5 | `BorrowitData` — RDS, Secrets Manager, schema and seed restore | Database live | Row counts match the Supabase export |
| 6 | Dockerise the API; add `/healthz` and `/readyz`; push to ECR | Image in ECR | Container runs locally against the RDS endpoint |
| 7 | `BorrowitApp` — cluster, task definition, service, ALB, target group | API reachable | 200 through the ALB; direct task IP times out |
| 8 | `BorrowitFrontend` — S3 buckets, CloudFront, OAC; point the client at the ALB | Full stack live | End-to-end: list an item, upload an image, reload |
| 9 | Log group and retention, metric filters, dashboard, six alarms, SNS | Observability in place | Kill the task; alarm email arrives |
| 10 | Enable Multi-AZ, force failover, measure; load test; review the real bill | Resilience evidence | Failover time recorded; Cost Explorer against the estimate |
| 11 | Write the workshop — sections 3.1 to 3.11 with screenshots | Workshop draft | A peer follows it from an empty account |
| 12 | Blogs, final report, handover, teardown decision | Submission | Everything in this table has evidence attached |

{{% notice note %}}
The internship runs to **04/09/2026**, but the assessment deadline is
**21/08/2026** — the end of week 10. Weeks 11 and 12 are documentation and buffer,
which means **the system has to work by the end of week 10**, not week 12. Every
gate above is placed with that earlier date in mind.
{{% /notice %}}

#### Build order, and why it cannot change

{{<mermaid align="center">}}
graph LR
  A["Foundation - VPC, SG, ECR"] --> B["Data - RDS, Secrets"]
  B --> C["Image - build, push to ECR"]
  C --> D["App - Fargate, ALB"]
  D --> E["Frontend - S3, CloudFront"]
  E --> F["Operations - alarms, failover, cost"]
{{</mermaid>}}

+ The **image must exist in ECR before the service is created** — an ECS service
  pointing at an empty repository fails to stabilise and the stack rolls back
  after roughly fifteen minutes of waiting.
+ The **database must exist before the task starts**, because the task reads its
  password from the secret RDS generated.
+ The **frontend needs the ALB DNS name** to point the client at the API, so it
  follows the app rather than preceding it.

<!-- TODO(prose): if you hit the empty-ECR rollback in practice, write the
     timestamp and the error in section 1 week 7. A plan that names a failure mode
     in advance and then survives it is the most useful thing in this document. -->

#### Deployment method

All four stacks deploy from a laptop with the CDK CLI:

```powershell
cd infra
npm install
npx cdk bootstrap aws://<account-id>/ap-southeast-1
npx cdk deploy BorrowitFoundation BorrowitData
npx cdk deploy BorrowitFrontend BorrowitApp
```

No CI/CD, by the scope decision in [2.2](../2.2-Requirements/). The tradeoff is
that every deploy is manual and unrecorded, which is acceptable at one engineer
and a handful of deploys per week.

#### Effort estimate

| Phase | Estimated | Confidence | Where it is likely to slip |
|---|---|---|---|
| 0 — Foundations | 3 weeks | High | — |
| 1 — Network and data | 2 weeks | Medium | Data export and restore fidelity |
| 2 — Compute | 2 weeks | **Low** | Container networking, health checks, IAM permissions |
| 3 — Delivery | 1 week | High | CORS between CloudFront and the ALB |
| 4 — Operations | 2 weeks | Medium | Alarm thresholds need tuning against real metrics |
| 5 — Documentation | 2 weeks | Medium | Screenshots need the system to still be running |

Phase 2 carries the schedule risk. It is the first time all three of container
build, IAM task roles and load-balancer health checks have to be correct
simultaneously, and each fails in a way that looks like the others — see
[2.7](../2.7-Risks/).
