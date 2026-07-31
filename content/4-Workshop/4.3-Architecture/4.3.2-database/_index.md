---
title : "Database — RDS over the alternatives"
date : 2026-07-28
weight : 2
chapter : false
pre : " <b> 4.3.2 </b> "
---

#### The candidates

| Option | Model | Billing | Cost at this scale |
|---|---|---|---|
| **RDS PostgreSQL** (`db.t4g.micro`) | Managed relational instance | Per instance-hour + storage | ~$16/month |
| **Aurora Serverless v2** | Managed relational, auto-scaling capacity | Per ACU-hour | ~$44/month at 0.5 ACU minimum |
| **Amazon DynamoDB** | Managed key-value / document | Per request + storage | ~$0–1/month |
| **PostgreSQL on EC2** | Self-managed | Per instance-hour | ~$8/month |

#### Why not DynamoDB

DynamoDB is by far the cheapest, and at this traffic level would be nearly free.
It was rejected because the workload is relational in a way that is not
incidental.

BorrowIt's core queries join users, items, bookings and availability windows, and
ask questions like "which items are free between these dates, excluding ones with
an overlapping booking". In DynamoDB that becomes either a carefully denormalised
single-table design worked out in advance, or several round trips joined in
application code.

<!-- TODO(prose): name the specific query from the application that decided this.
     A concrete example - a booking overlap check, or a listing filtered by
     category and availability - makes the argument concrete rather than abstract. -->

The deeper problem is that the schema already exists. Migrating to DynamoDB means
rewriting every model and query, not moving data. That is a rewrite disguised as a
migration, and the timeline did not allow it.

**DynamoDB would be the right choice** for a greenfield design with known access
patterns. It is the wrong choice for lifting an existing relational schema.

#### Why not Aurora Serverless v2

Aurora Serverless v2 is the closest thing to "RDS that scales to your load", and
it is technically superior to `db.t4g.micro` in most respects — faster storage,
faster failover, finer-grained scaling.

It costs roughly **3.7×** as much. The minimum capacity is 0.5 ACU, billed
continuously, and unlike RDS there is no burstable micro tier to drop to. At
~$44/month it would consume most of the credit balance on the database alone.

Aurora is the right answer when load is spiky and unpredictable. This workload is
neither.

#### Why not PostgreSQL on EC2

Cheapest of the managed-ish options, and it hands back everything RDS does for
you: backups, patching, failover, storage management, and encryption at rest. On a
project with a deadline, running your own database is a liability, not a saving.

The ~$8/month difference does not pay for one evening spent recovering a database
you forgot to back up.

#### The decision

**Amazon RDS for PostgreSQL 16** on `db.t4g.micro` with 20 GB of gp2 storage.

| Setting | Value | Reason |
|---|---|---|
| Engine | PostgreSQL 16 | Matches the existing schema; no application changes |
| Instance class | `db.t4g.micro` | Graviton — cheapest class that runs Postgres 16 |
| Storage | 20 GB | Minimum RDS accepts |
| `maxAllocatedStorage` | **20 GB** | Pinned equal to allocated, so autoscaling cannot grow the bill |
| Subnets | `PRIVATE_ISOLATED` | No route to the internet in either direction |
| `publiclyAccessible` | `false` | Reachable only from inside the VPC |
| `storageEncrypted` | `true` | Free; no reason not to |
| Multi-AZ | `false` by default | Doubles cost — enabled on demand with `-c multiAz=true` |
| Backup retention | 1 day | Enough to recover a bad migration, minimal snapshot cost |

{{% notice note %}}
**Cost:** ~$16/month, billed against Free Plan credits from the first hour. This
account post-dates the 15 July 2025 free-tier change, so **the 12-month free RDS
allowance does not apply** — do not size this instance as though it did.
{{% /notice %}}

#### What this choice costs

+ **A single point of failure by default.** One instance in one AZ. Multi-AZ is
  one context flag away but doubles the cost, so it is off by default.
+ **Fixed capacity.** `t4g.micro` is burstable; sustained CPU above the baseline
  exhausts credits and throttles. The `db-cpu` alarm in
  [3.8.3](../../4.8-Observability/4.8.3-alarms/) exists to catch that.
+ **Storage cannot grow.** Pinning `maxAllocatedStorage` protects the budget and
  makes running out of disk a real failure mode — which is why one of the seven
  alarms watches free storage specifically.
