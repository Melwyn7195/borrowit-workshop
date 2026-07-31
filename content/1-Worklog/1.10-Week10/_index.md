---
title: "Week 10 Worklog"
date: 2026-07-28
weight: 10
chapter: false
pre: " <b> 1.10. </b> "
---

{{% notice warning %}}
**Hard deadline this week.** Company paperwork (forms D2–D5) is due to the Faculty
before **16:00 on 21/08/2026**. Treat the technical work as needing to be finished
by then.
{{% /notice %}}

### Week 10 Objectives

* Demonstrate high availability: RDS Multi-AZ failover and Fargate autoscaling.
* Measure behaviour under load rather than asserting it.
* Submit the company paperwork.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - Enable `-c multiAz=true -c scaling=true` <br> - Understand what a synchronous standby does and does not protect against | 17/08/2026 | | |
| 3 | - Force an RDS failover; record how many seconds of errors the API returned and whether ECS restarted the task | 18/08/2026 | | |
| 4 | - Run a load test; capture the task count moving 1 → 2 → 3 and the latency recovering | 19/08/2026 | | |
| 5 | - Revert both flags and confirm Multi-AZ reads **No** again <br> - Verify the bill has come back down | 20/08/2026 | | |
| 6 | - **Submit forms D2–D5 to the Faculty before 16:00** | 21/08/2026 | | |

### Week 10 Planned Outcomes

* A measured failover: real numbers for the outage window, not an estimate.
* Evidence of autoscaling responding to CPU, captured on the dashboard.
* Both cost-increasing flags reverted, verified in the console.
* Paperwork submitted on time.

<!-- TODO(prose): the failover measurement is where the week 7 health-check design
     is tested. If the liveness check did not restart the task during the failover,
     that is the design working, and it is worth saying so explicitly. If it did
     restart, say that too and explain what you would change. -->

{{% notice warning %}}
Multi-AZ doubles the RDS instance cost and autoscaling can triple the Fargate cost.
Do not leave either flag on after the demonstration.
{{% /notice %}}
