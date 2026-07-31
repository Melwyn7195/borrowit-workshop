---
title : "Alarms and notification"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 3.8.3 </b> "
---

A dashboard tells you something is wrong once you look at it. An alarm tells you
without being asked.

#### Notification path

```typescript
const alarmTopic = new sns.Topic(this, 'AlarmTopic', {
  displayName: 'BorrowIt alarms',
});

if (alarmEmail) {
  alarmTopic.addSubscription(new subscriptions.EmailSubscription(String(alarmEmail)));
}
```

The SNS topic is created unconditionally so alarms always have somewhere to
publish — a topic with no subscription costs nothing. The email address is passed
as context rather than hardcoded, keeping a personal address out of the repository:

```powershell
npx cdk deploy BorrowitApp -c alarmEmail=you@example.com
```

{{% notice warning %}}
AWS sends a confirmation email and **the subscription delivers nothing until you
click the link**. An unconfirmed subscription is the most common reason an alarm
"did not fire" when it actually did.
{{% /notice %}}

#### The seven alarms

| Alarm | Metric | Condition | Periods | Missing data |
|---|---|---|---|---|
| `unhealthy-targets` | `UnHealthyHostCount` | ≥ 1 | 1 × 5 min | **BREACHING** |
| `api-5xx` | Target 5xx count | > 5 | 1 × 5 min | NOT_BREACHING |
| `service-cpu` | ECS CPU % | > 80 | 3 × 5 min | NOT_BREACHING |
| `service-memory` | ECS memory % | > 85 | 3 × 5 min | NOT_BREACHING |
| `db-cpu` | RDS CPU % | > 80 | 3 × 5 min | NOT_BREACHING |
| `db-storage` | RDS free storage | < 2 GiB | 1 × 5 min | NOT_BREACHING |
| `error-logs` | `BorrowIt/ApiErrorLogCount` | > 20 | 1 × 5 min | NOT_BREACHING |

Six come from infrastructure metrics AWS publishes on its own. The seventh comes
from the **logs** — see below.

#### `treatMissingData` is the whole game

This is the most important concept in the section, and it is easy to get wrong in
a way that silently disables an alarm.

```typescript
new cloudwatch.Alarm(this, 'UnhealthyTargetsAlarm', {
  metric: targetGroup.metrics.unhealthyHostCount({ period }),
  threshold: 1,
  evaluationPeriods: 1,
  treatMissingData: cloudwatch.TreatMissingData.BREACHING,
}),
```

**A task that has disappeared entirely publishes no data at all — not a value of
zero.** With CloudWatch's default `MISSING` behaviour, the alarm would hold its
last state, and with the common `NOT_BREACHING` setting it would read as healthy.
The service could be completely down while the alarm sat green.

`BREACHING` means "no data is itself the emergency", which for this metric is
exactly right.

The 5xx alarm is the mirror image. **No requests in a five-minute window is not an
error** — it is 3 a.m. on a demonstration system. Marking missing data as breaching
there would page you every night. So `NOT_BREACHING` is correct.

Same setting, opposite values, because the metrics mean different things when
silent.

<!-- TODO(prose): this is the single best thing on this page to be able to explain
     under questioning. Write two or three sentences in your own words on why "no
     data" means opposite things for these two metrics. -->

#### Why the evaluation periods differ

+ **`unhealthy-targets` and `db-storage` fire after one period.** Both are
  already-broken conditions — there is nothing to wait out.
+ **The CPU and memory alarms require three consecutive periods (15 minutes).** A
  single 5-minute spike above 80% CPU is normal during a deployment or a burst.
  Alarming on it produces noise, and an alarm you learn to ignore is worse than no
  alarm.

#### Why `db-storage` exists at all

Storage autoscaling is deliberately pinned off
([3.3.2](../../3.3-Architecture/3.3.2-database/)) so RDS cannot silently grow
the bill. The consequence is that **running out of disk is a real failure mode**
rather than something RDS quietly fixes — so it needs an alarm. This is a good
example of one cost decision creating an operational requirement.

#### The alarm that comes from logs

The other six watch metrics AWS publishes for you. `error-logs` watches something
the application wrote, which requires turning log lines into a metric first:

```typescript
const errorLogMetric = logGroup
  .addMetricFilter('ErrorLogFilter', {
    filterPattern: logs.FilterPattern.anyTerm('ERROR', 'Error:', 'error:'),
    metricNamespace: 'BorrowIt',
    metricName: 'ApiErrorLogCount',
    metricValue: '1',
    defaultValue: 0,
  })
  .metric({ period, statistic: 'Sum' });
```

It exists because it catches failures the ALB metrics **cannot see by
definition** — a bad query caught and returned as a 400, an email that will not
send. Those are not 5xx, so `api-5xx` stays silent while the feature is broken.

Two details are deliberate:

+ **The filter matches plain text, not JSON.** `index.js` emits structured
  errors, but the `console.error` calls scattered through the controllers
  ("Login error:", "Auth middleware error:") are plain strings — and they are
  still the majority of what actually gets logged when something breaks. A JSON
  filter would silently miss all of them.
+ **`defaultValue: 0`** makes the filter emit an explicit zero when logs arrive
  with no error in them, so the alarm can tell "quiet" apart from "no data".

Metric filters themselves are free — they run on logs already being ingested.
The custom metric they produce costs ~$0.30/month.

{{% notice note %}}
The threshold of 20 is a **guess, not a measurement**. A handful of failed logins
an hour is normal; twenty errors in five minutes is not. Watch it for a few days
and move it rather than leaving it noisy.
{{% /notice %}}

<!-- TODO(prose): if you moved the threshold after watching real traffic, say what
     you moved it to and why. An alarm tuned against observed behaviour is worth
     far more than one set from a guess and never revisited. -->

<!-- SCREENSHOT: /images/3-Workshop/3.8-Observability/3.8.3-alarms/alarms-list.png
     All seven alarms in the CloudWatch console with their current states. -->

#### Prove one fires

An alarm you have never seen fire is an alarm you do not know works.

```powershell
# Scale to zero - unhealthy-targets should reach ALARM within ~5 minutes
npx cdk deploy BorrowitApp -c desiredCount=0

# ...confirm the alarm reaches ALARM and the email arrives, then restore
npx cdk deploy BorrowitApp
```

{{% notice warning %}}
`desiredCount=0` stops the tasks but **the load balancer keeps billing**. This is a
test, not a cost-saving measure — restore the service afterwards.
{{% /notice %}}

<!-- TODO(prose): record how long the alarm actually took to fire. The expected
     answer is 5-10 minutes, because the metric period is 5 minutes and CloudWatch
     evaluates after the period closes. If it took longer than you expected, that
     delay is worth knowing about before you rely on these alarms. -->

{{% notice note %}}
**Cost:** standard alarms are **$0.10/month each** — $0.70 for seven — plus
~$0.30/month for the custom metric behind `error-logs`. SNS email delivery is free
for the first 1,000 notifications per month.
{{% /notice %}}
