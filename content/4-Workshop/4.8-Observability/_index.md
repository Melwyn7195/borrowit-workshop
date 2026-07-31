---
title : "Observability and operations"
date : 2026-07-28
weight : 8
chapter : false
pre : " <b> 4.8. </b> "
---

A system you cannot observe is a system you cannot claim works. This section adds
the three signals — logs, metrics and alarms — and the operational access needed
to investigate when one of them fires.

#### What is being watched, and why

| Question | Answered by | Section |
|---|---|---|
| What did the application do? | CloudWatch Logs | [3.8.1](4.8.1-logs/) |
| How is the system behaving over time? | CloudWatch Metrics + dashboard | [3.8.2](4.8.2-metrics-dashboard/) |
| Should someone be woken up? | CloudWatch Alarms + SNS | [3.8.3](4.8.3-alarms/) |
| What is actually happening inside right now? | ECS Exec | [3.8.4](4.8.4-debugging/) |

#### Why observability lives in `BorrowitApp`

A separate observability stack would have to **import** from `BorrowitApp` — and
nothing is allowed to do that, because the one-way dependency is what keeps
`cdk destroy BorrowitApp` working ([3.1](../4.1-Overview/)).

The cost of co-locating them is that destroying the application also removes its
alarms and dashboard. That is acceptable: with no service running, they have
nothing left to watch.

#### What was deliberately not used

Choosing what to leave out is part of the design:

| Service | Cost | Why not |
|---|---|---|
| **Container Insights** | ~$0.30/metric/month, dozens of metrics | Detailed per-container metrics; the ECS service metrics already answer the questions being asked |
| **AWS X-Ray** | $5 per million traces | Distributed tracing pays off across many services; this is one service and one database |
| **CloudWatch Synthetics** | ~$0.0012/canary run | A scheduled external probe would be genuinely useful — the unhealthy-target alarm covers the same failure more cheaply |
| **Third-party APM** | $$$ | Out of budget entirely |

<!-- TODO(prose): X-Ray is the most defensible omission to be challenged on. The
     honest answer is that with one API and one database, the CloudWatch metrics
     already localise a problem to the task or the database - tracing would add
     cost without adding an answer. If you disagree after building it, say so. -->

{{% notice note %}}
**Total cost of this section:** approximately **$4.80/month** — one dashboard at
~$3, seven standard alarms at ~$0.10 each, two custom metrics derived from the
logs at ~$0.30 each, log ingestion and storage at well under $1, and SNS email
delivery which is effectively free at this volume. This is the one place in the
project where monitoring was allowed to cost something.
{{% /notice %}}

#### Content

1. [Logs](4.8.1-logs/)
2. [Metrics and the dashboard](4.8.2-metrics-dashboard/)
3. [Alarms and notification](4.8.3-alarms/)
4. [Debugging a running task](4.8.4-debugging/)
