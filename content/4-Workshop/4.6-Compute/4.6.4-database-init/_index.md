---
title : "Initialising the database"
date : 2026-07-28
weight : 4
chapter : false
pre : " <b> 4.6.4 </b> "
---

The database is running but empty, so the readiness check is failing. This step
creates the schema — and the interesting part is *how you reach a database that
has no network path from your machine*.

#### The access problem

RDS sits in isolated subnets with `publiclyAccessible: false`. There is no route
from a workstation to it. That is the design working correctly, and it is also
inconvenient.

| Option | Cost/month | Verdict |
|---|---|---|
| Make RDS publicly accessible | $0 | Defeats the entire isolation design. No. |
| Bastion host in a public subnet | ~$8 (t4g.nano) | Works — a whole EC2 instance, to patch and remember, for one command |
| VPN or Direct Connect | $$$ | Disproportionate |
| **`aws ecs execute-command`** | **$0** | A shell inside a task that can already reach the database |

#### Why ECS Exec wins here

`enableExecuteCommand: true` on the service makes the third option available. It
works through **AWS Systems Manager Session Manager**: the ECS agent opens an
outbound connection to SSM, and your CLI connects through the same channel. No
inbound ports, no SSH keys, no additional host.

CDK adds the required SSM permissions to the task role automatically.

The security properties are better than a bastion, not merely cheaper:

+ Access is controlled by **IAM**, not by possession of an SSH key.
+ Every session is auditable in **CloudTrail**.
+ There is no long-lived host to patch, and nothing to forget to shut down.

<!-- TODO(prose): present this as a decision you made. A bastion is the
     conventional answer and easier to explain to someone who has not met Session
     Manager; ECS Exec costs nothing and leaves no residue. -->

#### Open a shell in the running task

```powershell
$env:CLUSTER = (aws cloudformation describe-stacks `
  --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='ClusterName'].OutputValue" `
  --output text)

$env:TASK = (aws ecs list-tasks --cluster $env:CLUSTER --query "taskArns[0]" --output text)

aws ecs execute-command `
  --cluster $env:CLUSTER `
  --task $env:TASK `
  --container api `
  --interactive `
  --command "/bin/sh"
```

{{% notice note %}}
`TargetNotConnectedException` usually means the Session Manager plugin is not
installed, or the task started before `enableExecuteCommand` was turned on. Force a
new deployment and retry.
{{% /notice %}}

#### Apply the schema

Inside the container shell:

```sh
node scripts/run-sql.js db/schema.sql
npm run seed
```

The scripts connect using the same `DB_*` variables ECS injected from Secrets
Manager — so no connection string is typed anywhere, and the password is still
never seen by a human.

<!-- TODO: the schema file has to exist in the image before the docker build in
     3.6.1. If the source database is elsewhere, export it first with:
       pg_dump --schema-only --no-owner --no-privileges <source-url> -f db/schema.sql
     --no-owner and --no-privileges strip role grants that will fail on RDS.
     Record any incompatibilities you actually hit - a migration write-up with no
     friction in it reads as though the migration was never performed. -->

#### The target goes healthy

Within about a minute the readiness check passes and the target moves to
**healthy**.

Confirm end to end:

```powershell
curl "$env:API/health"
curl "$env:API/api/products"
```

{{% notice warning %}}
`enableExecuteCommand` grants a shell inside a running production container to
anyone with the matching IAM permission. It is invaluable for the operational
debugging in [3.8.4](../../4.8-Observability/4.8.4-debugging/), and it is a
privilege worth scoping deliberately rather than leaving on an over-broad policy.
{{% /notice %}}
