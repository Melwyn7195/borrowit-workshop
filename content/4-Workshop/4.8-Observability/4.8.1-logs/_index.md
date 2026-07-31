---
title : "Logs"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 4.8.1 </b> "
---

#### How container output reaches CloudWatch

```typescript
const logGroup = new logs.LogGroup(this, 'ApiLogGroup', {
  logGroupName: '/borrowit/api',
  retention: logs.RetentionDays.ONE_WEEK,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});

logging: ecs.LogDrivers.awsLogs({
  streamPrefix: 'borrowit',
  logGroup,
}),
```

The group is **declared explicitly** rather than letting the log driver generate
one, for two reasons: the metric filters in [3.8.3](../4.8.3-alarms/) and the
saved queries below need something to attach to, and a fixed name makes the group
findable in the console without first looking up a CloudFormation-generated
suffix.

The `awslogs` driver sends everything the container writes to stdout and stderr
straight to CloudWatch Logs. The application needs no logging SDK, no agent, and no
file rotation — writing to stdout is the entire integration.

**Retention is set to one week.** CloudWatch Logs defaults to *never expire*, which
on a long-running project accumulates storage charges indefinitely for logs nobody
will read. A week is long enough to investigate an incident and short enough that
storage stays negligible.

<!-- TODO(prose): worth a sentence on the tradeoff - one week means an incident
     discovered a fortnight later cannot be investigated from logs. For this
     project that is acceptable; for an audited system it would not be. -->

#### One stream per task

The log group contains one stream per task, named
`borrowit/api/<task-id>`. When ECS replaces a task, the new one gets a new stream —
which is useful, because it means the boundary between the old and new task is
visible in the log structure itself.

#### Reading logs from the CLI

The group name is fixed, so there is nothing to look up:

```powershell
aws logs tail /borrowit/api --follow --region ap-southeast-1
```

The stack outputs it too, with the command spelled out in the description:

```powershell
aws cloudformation describe-stacks --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='LogGroupName'].OutputValue" --output text
```

#### Logs Insights

For anything beyond tailing, **CloudWatch Logs Insights** queries the log group
directly. Useful starting queries:

```
fields @timestamp, @message
| filter @message like /error/
| sort @timestamp desc
| limit 50
```

Count errors over time:

```
fields @timestamp
| filter @message like /error/
| stats count() by bin(5m)
```

![CloudWatch Logs Insights query with its results](/images/4-Workshop/4.8-Observability/4.8.1-logs/insights-query.png?width=100pc)

#### Saved queries

Writing query syntax from memory at the moment something is broken is the worst
time to do it, so five queries are saved with the stack:

```typescript
const queryLines = [
  'fields @timestamp, path, message, stack',
  "filter level = 'error'",
  'sort @timestamp desc',
  'limit 50',
];

new logs.CfnQueryDefinition(this, 'Query0', {
  name: 'BorrowIt/Errors with stack traces',
  logGroupNames: [logGroup.logGroupName],
  queryString: queryLines.join('\n| '),
});
```

They appear under **Logs Insights → Saved queries → BorrowIt**, and they are
free:

| Query | Answers |
|---|---|
| Errors with stack traces | What broke, and where in the code |
| Slowest requests | Which individual calls were slow |
| Status code breakdown | How the responses are distributed |
| Traffic by route | Which endpoints are actually used |
| Raw error lines | The plain-text `console.error` output the structured queries miss |

{{% notice warning %}}
String literals in Insights must be in **single quotes**. Insights reads a
double-quoted token as a *field name*, so `type = "request"` compares the field
`type` against the field `request` — returning zero rows without erroring. A
filter that looks right and quietly matches nothing is worse than a syntax error.
{{% /notice %}}

{{% notice note %}}
Logs Insights bills **per GB scanned** ($0.0057/GB in `ap-southeast-1`). At this log
volume a query costs a fraction of a cent, but narrowing the time range before
running a broad query is a good habit — on a large log group it is the difference
between free and expensive.
{{% /notice %}}

#### What to log, and what not to

<!-- TODO(prose): the important operational point. Since the database password is
     injected as an environment variable inside the container, anything that dumps
     the environment on startup - a debug print, an unhandled exception handler
     that serialises process.env - writes the password into CloudWatch Logs in
     plain text, undoing the entire Secrets Manager design from 3.3.4.

     If you checked for this, say so. If you found one and removed it, that is
     even better material. -->

{{% notice warning %}}
Anything printed to stdout is stored in CloudWatch Logs and readable by anyone with
`logs:GetLogEvents`. Verify that no startup diagnostic dumps the process
environment — the injected `DB_PASSWORD` lives there.
{{% /notice %}}

{{% notice note %}}
**Cost:** ingestion is ~$0.63/GB and storage ~$0.03/GB-month in `ap-southeast-1`.
At this traffic level, well under $1/month.
{{% /notice %}}
