---
title : "Debugging a running task"
date : 2026-07-28
weight : 4
chapter : false
pre : " <b> 4.8.4 </b> "
---

Logs, metrics and alarms tell you *that* something is wrong. This page is about
finding out *what*.

#### A triage order

When the API is failing, work outward from the load balancer. Each step rules out
a layer:

| Step | Command | If it fails |
|---|---|---|
| 1. Is a task running? | `aws ecs describe-services` | ECS cannot place or start the task — check step 2 |
| 2. Why did the task stop? | `aws ecs describe-tasks` → `stoppedReason` | Image pull, secret resolution, or the app crashed |
| 3. What did the app say? | `aws logs tail` | Application-level error |
| 4. Is the target healthy? | Target group health | The readiness check is failing — usually the database |
| 5. What is true inside the container? | `aws ecs execute-command` | Everything else has been ruled out |

<!-- TODO(prose): if you actually worked through a failure in this order, write it
     up as a short worked example. A real incident narrated end to end is the most
     convincing content this section can contain. -->

#### Reading a stopped task

The single most useful field in ECS:

```powershell
aws ecs describe-tasks --cluster $env:CLUSTER --tasks $env:TASK `
  --query "tasks[0].{stopped:stoppedReason,exit:containers[0].exitCode,health:healthStatus}"
```

Common values and what they mean:

| `stoppedReason` / exit code | Cause |
|---|---|
| `CannotPullContainerError` | Image tag missing, or no route to ECR |
| `ResourceInitializationError: unable to pull secrets` | Execution role cannot read the secret |
| `exec format error` | Image built for the wrong CPU architecture ([3.6.1](../../4.6-Compute/4.6.1-image-and-registry/)) |
| Exit code **137** | Killed — out of memory ([3.8.2](../4.8.2-metrics-dashboard/)) |
| Exit code **139** | Segmentation fault in a native module |
| `Essential container in task exited` | The application itself exited |

{{% notice note %}}
With `natGateways: 0`, a `CannotPullContainerError` is very often a networking
symptom rather than an image problem: the task must have a public IP and sit in a
public subnet. Check `assignPublicIp` before you check the image tag.
{{% /notice %}}

#### Getting inside

```powershell
aws ecs execute-command --cluster $env:CLUSTER --task $env:TASK `
  --container api --interactive --command "/bin/sh"
```

Once inside, the useful checks:

```sh
# Is the process running and what is it using?
ps aux

# Did the secrets actually resolve? (names only - never print the values)
env | cut -d= -f1 | sort

# Can the container reach the database at all?
node -e "require('net').connect(5432, process.env.DB_HOST)
  .on('connect', () => { console.log('reachable'); process.exit(0); })
  .on('error', e => { console.log('unreachable:', e.code); process.exit(1); })"

# Does the app answer itself?
node -e "require('http').get('http://127.0.0.1:3456/health', r => console.log(r.statusCode))"
```

{{% notice warning %}}
`env` prints the injected `DB_PASSWORD` in full. Pipe through `cut -d= -f1` as
above to list names only, and **never screenshot the raw output**.
{{% /notice %}}

#### Distinguishing the two health checks

Because liveness and readiness differ ([3.6.3](../../4.6-Compute/4.6.3-load-balancer/)),
their symptoms differ too — and telling them apart localises the fault immediately:

| Symptom | Meaning |
|---|---|
| Task keeps restarting | **Liveness** failing — the process is dying |
| Task stays running but the target is unhealthy | **Readiness** failing — the process is fine, the database is not |
| Both healthy but requests 5xx | Application logic — go to the logs |

That middle row is the design working as intended: a database problem that does
*not* restart the container.

#### Auditing who did what

ECS Exec sessions are logged in **CloudTrail**, which matters because the feature
grants a shell inside a production container:

```powershell
aws cloudtrail lookup-events `
  --lookup-attributes AttributeKey=EventName,AttributeValue=ExecuteCommand `
  --region ap-southeast-1
```

<!-- TODO(prose): worth closing on the security posture. ECS Exec is powerful, and
     the right way to hold it is IAM-scoped and audited rather than disabled -
     but note that it is enabled here for a demonstration system and would deserve
     tighter scoping in production. -->
