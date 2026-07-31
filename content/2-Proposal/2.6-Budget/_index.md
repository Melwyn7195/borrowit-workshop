---
title : "Budget and cost model"
date : 2026-07-28
weight : 6
chapter : false
pre : " <b> 2.6. </b> "
---

#### The budget

| | |
|---|---|
| Funding | **AWS Free Plan credits — $160** |
| Account created | 28/07/2026 |
| Credits expire | **28/01/2027** |
| 12-month free tier | **None.** The account post-dates the 15 July 2025 change |
| Check the balance | `aws freetier get-account-plan-state` |

The second and third rows are what make this a real constraint rather than a
formality. There is no free tier to absorb the database, so **RDS bills from the
first hour**, and the credits are spent whether the system is used or not.

#### Estimated monthly cost

Priced for `ap-southeast-1`, at the sizing proposed in [2.3](../2.3-Design/):

| Service | Configuration | Estimate/month |
|---|---|---|
| Interface VPC endpoints | 5 endpoints × 1 AZ | ~$36 |
| Application Load Balancer | 1 ALB, minimal LCU | ~$17 |
| Amazon RDS | `db.t4g.micro`, 20 GB gp2, single-AZ | ~$16 |
| ECS Fargate | 1 task, 0.25 vCPU / 0.5 GB, 24/7 | ~$9 |
| CloudWatch | 1 dashboard, 6 alarms, logs | ~$3.60 |
| Public IPv4 | 1 address on the task | ~$3.60 |
| S3, CloudFront, ECR, Secrets Manager | Low volume | ~$1 |
| **Total** | | **~$86** |

The first row is the one to argue about. Five PrivateLink endpoints cost more than
the load balancer, the compute and the telemetry combined, and they carry no
application traffic — they exist so that credentials, images and logs never cross
the public internet. [3.3.3](../../3-Workshop/3.3-Architecture/3.3.3-networking/)
makes that case in full.

#### Does it fit?

| | |
|---|---|
| Full stack, running continuously | ~$86/month |
| Credits available | $160 |
| Runway | **~1.8 months**, to roughly late September 2026 |
| Needed until | 21/08/2026 — under one month away |
| Cost of running everything to the deadline | **~$60** |

It fits, but not with room to spare. The conclusion that matters is still the
second-to-last row: **the deadline arrives before the credits run out**, so the
design does not need to be optimised for a system that runs indefinitely. It does
now need the deadline to hold. If the project has to stay up into October, the
endpoints are the first line to reconsider — they are ~40% of the bill.

{{% notice tip %}}
This is why teardown between work sessions is treated as *available*, not
*required*. Destroying and rebuilding `BorrowitApp` every evening saves a few
dollars and risks arriving at a demo with nothing deployed. A running system is
worth more than the saving.
{{% /notice %}}

#### Where the design gave up cost for something else

Three decisions in [2.3](../2.3-Design/) exist because of this page:

| Decision | Saves | Costs |
|---|---|---|
| No NAT Gateway | ~$33/month | Thinner defence in depth; ~$3.60/month in public IPv4 charges |
| Interface endpoints instead | — (spends ~$36/month) | The one decision on this page that went the *other* way: bought back the confidentiality the NAT would have provided, for more than the NAT cost |
| Single-AZ RDS | ~$16/month | No standby; failover is demonstrated on demand, not always available |
| No Container Insights, WAF, Config | ~$10–20/month | Less telemetry and no managed threat detection |

Reversing any of these is a one-line CDK change. Each is reversible on paper and
expensive in practice, which is exactly why they are decided here rather than
during the build.

#### Controls

| Control | Where | What it prevents |
|---|---|---|
| Budget alarm at $10 | Billing → Budgets | A runaway cost going unnoticed for a month |
| Cost allocation tags `Project` / `ManagedBy` | Applied at the CDK app level | An unattributable bill |
| `maxAllocatedStorage` pinned to `allocatedStorage` | `BorrowitData` | RDS storage autoscaling growing the bill silently |
| One-way stack dependencies | Stack topology | The $31/month layer being undestroyable |

{{% notice note %}}
Cost allocation tags must be **activated** in the Billing console before Cost
Explorer will group by them, and **activation is not retroactive**. Do this in
week 1 — discovering it in week 10 means ten weeks of spend that cannot be broken
down.
{{% /notice %}}

#### If it runs hot

In order of saving per unit of effort:

1. **Destroy `BorrowitApp`** — removes ~$31/month, rebuilt in about five minutes.
2. **Cut log retention** below one week.
3. **Delete the dashboard** — ~$3/month, and the alarms keep working without it.
4. Do **not** set `desiredCount` to 0 as a saving. The ALB bills regardless, so
   this removes ~$9 and leaves ~$21 in place.

#### Verifying the estimate

An estimate that is never checked is a guess. In week 10 the model above is
compared against Cost Explorer, filtered on the project tag, and any difference
is explained against the assumptions listed here.

```powershell
aws ce get-cost-and-usage `
  --time-period Start=2026-08-01,End=2026-09-01 `
  --granularity MONTHLY `
  --metrics BlendedCost `
  --group-by Type=DIMENSION,Key=SERVICE `
  --filter '{"Tags":{"Key":"Project","Values":["BorrowIt"]}}'
```

<!-- TODO(prose): once you have the actual figures, note here whether the estimate
     held. The most common surprises are CloudWatch Logs ingestion and the public
     IPv4 charge. Being wrong and saying so is fine; not checking is not. -->
