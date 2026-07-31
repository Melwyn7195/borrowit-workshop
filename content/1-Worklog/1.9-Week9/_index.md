---
title: "Week 9 Worklog"
date: 2026-07-28
weight: 9
chapter: false
pre: " <b> 1.9. </b> "
---

{{% notice note %}}
**Planned.** Replace with what actually happened and fill in completion dates.
{{% /notice %}}

### Week 9 Objectives

* Add a CloudWatch dashboard and alarms.
* Verify the real bill against the week 3 estimate.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - CloudWatch: metrics, statistics, periods, alarm states, `treatMissingData` | 10/08/2026 | | |
| 3 | - Add six alarms; set `BREACHING` on unhealthy targets so a vanished task reads as an outage rather than as silence | 11/08/2026 | | |
| 4 | - Create the SNS topic and email subscription; build the five-widget dashboard | 12/08/2026 | | |
| 5 | - Trigger an alarm deliberately (`-c desiredCount=0`) and confirm the notification arrives | 13/08/2026 | | |
| 6 | - Cost Explorer, filtered on the `Project=BorrowIt` tag; compare actual against estimated | 14/08/2026 | | |

### Week 9 Planned Outcomes

* Six alarms covering the load balancer, the service and the database, publishing
  to SNS.
* A dashboard showing requests, errors, p50/p95 latency, task and database health.
* An alarm proven to fire, not merely configured.
* Actual monthly cost compared against the week 3 estimate, per service.

<!-- TODO(prose): `treatMissingData` is the detail worth understanding properly.
     A task that has disappeared publishes no data at all, not zero - so with the
     default setting the service could be completely down while the alarm stayed
     green. Explain why "no data" means the opposite thing for the 5xx alarm. -->
