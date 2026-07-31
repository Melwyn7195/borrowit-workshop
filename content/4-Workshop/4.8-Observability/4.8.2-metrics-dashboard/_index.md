---
title : "Metrics and the dashboard"
date : 2026-07-28
weight : 2
chapter : false
pre : " <b> 4.8.2 </b> "
---

#### Where the metrics come from

None of these are published by the application. Every one is emitted automatically
by the AWS service itself, which is a large part of what you are paying these
services for.

| Source | Metrics used | Published by |
|---|---|---|
| ALB target group | `RequestCount`, `TargetResponseTime`, `HTTPCode_Target_5XX_Count`, `HealthyHostCount`, `UnHealthyHostCount` | Elastic Load Balancing |
| ALB | `HTTPCode_ELB_5XX_Count` | Elastic Load Balancing |
| ECS service | `CPUUtilization`, `MemoryUtilization` | Amazon ECS |
| RDS instance | `CPUUtilization`, `DatabaseConnections`, `FreeStorageSpace` | Amazon RDS |

All are **standard-resolution (1-minute)** metrics included at no extra charge. The
dashboard aggregates them at a 5-minute period.

#### Target 5xx versus ELB 5xx

The dashboard graphs both, and the distinction is diagnostic:

+ **`HTTPCode_Target_5XX_Count`** — the application returned an error. The task is
  running and responding; the fault is in the code or the database.
+ **`HTTPCode_ELB_5XX_Count`** — the *load balancer* generated the error, because
  it had no healthy target to route to, or the target timed out.

Seeing which of the two spikes tells you immediately whether to look at the
application or at the infrastructure. Graphing only a combined error count throws
that distinction away.

<!-- TODO(prose): if you saw both during a failover test, describe which
     appeared and when. That is the kind of observation that shows the dashboard
     was actually used rather than merely built. -->

#### p50 and p95, not average

```typescript
left: [
  targetGroup.metrics.targetResponseTime({ period, statistic: 'p50' }),
  targetGroup.metrics.targetResponseTime({ period, statistic: 'p95' }),
],
```

An average latency hides the requests that actually annoy users. If 95 requests
take 50 ms and 5 take 4 seconds, the average is a comfortable 250 ms and one user
in twenty is having a bad time.

**p50** shows the typical experience; **p95** shows the tail. The gap between them
is the signal — a widening gap means something is intermittently slow, which is
usually more informative than either number alone.

#### The dashboard

Five widgets across four rows:

| Widget | Left axis | Right axis | Answers |
|---|---|---|---|
| Requests and errors | `RequestCount` | Target 5xx, ELB 5xx | Is it being used, and is it failing? |
| Latency | p50, p95 response time | — | Is it fast, and for everyone? |
| Fargate task | CPU %, Memory % | — | Is the container sized correctly? |
| Database | RDS CPU % | Connections, free storage | Is the database the bottleneck? |
| Healthy targets | Healthy, unhealthy host count | — | Is anything actually serving? |

```powershell
aws cloudformation describe-stacks --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='DashboardUrl'].OutputValue" --output text
```

![The full CloudWatch dashboard with real traffic data in it](/images/4-Workshop/4.8-Observability/4.8.2-metrics-dashboard/dashboard.png?width=100pc)

#### Memory is the one to watch

512 MiB is not much for Node.js. The memory widget matters more than the CPU one
here: Fargate **kills a container that exceeds its memory limit** — there is no
swap and no graceful degradation, the task simply dies with exit code 137 and ECS
starts a replacement.

CPU exhaustion merely makes things slow. Memory exhaustion is fatal, which is why
the memory alarm in [3.8.3](../4.8.3-alarms/) is set at 85% rather than the 80%
used for CPU.

<!-- TODO(prose): report the steady-state memory figure you actually observe. If it
     sits near 400 MiB, the headroom is thin and worth naming as a risk. If it sits
     at 150 MiB, the task is oversized and could arguably be smaller. -->

#### Reading a metric from the CLI

Useful when you want a number rather than a picture:

```powershell
aws cloudwatch get-metric-statistics `
  --namespace AWS/ApplicationELB `
  --metric-name TargetResponseTime `
  --dimensions Name=LoadBalancer,Value=<lb-id> `
  --start-time (Get-Date).AddHours(-3).ToString("yyyy-MM-ddTHH:mm:ss") `
  --end-time (Get-Date).ToString("yyyy-MM-ddTHH:mm:ss") `
  --period 300 --statistics Average --region ap-southeast-1
```

{{% notice note %}}
**Cost:** a CloudWatch dashboard is **$3/month** after the first three, which is
the largest single line in the observability budget. Standard metrics from AWS
services are free; only custom metrics and additional dashboards cost extra.
{{% /notice %}}
