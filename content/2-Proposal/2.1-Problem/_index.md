---
title : "Problem and objectives"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 2.1. </b> "
---

#### The system today

**BorrowIt** is a peer-to-peer rental platform. Someone lists an item, someone
else borrows it, and the platform handles listings, images and the record of who
has what.

Three components, one hosting provider:

| Component | Technology | Where it runs today |
|---|---|---|
| Web client | React single-page application | Static hosting |
| API | Express (Node.js), REST, listens on port 3456 | Supabase-hosted container |
| Database | PostgreSQL, relational schema | Supabase managed Postgres |
| User uploads | Item images | Supabase Storage |

<!-- TODO(prose): confirm this table against the running system before submitting.
     If uploads were served straight from the API rather than from object storage,
     say so — it changes the migration story for section 4.7. -->

#### Why move it

The application works. That is worth stating plainly, because "it was broken" is
not the reason for this migration and pretending otherwise would be dishonest.

The reasons are:

+ **A single provider owns every layer.** Compute, database, storage and auth all
  come from one vendor with one bill and one failure domain. There is no way to
  scale, secure or observe any layer independently.
+ **The infrastructure is not written down.** It was clicked into existence. There
  is no artefact that can rebuild it, review it, or explain it to someone new.
+ **There is no operational visibility.** No metrics, no alarms, no logs anyone
  looks at. Whether the API is healthy is currently discovered by opening it.
+ **It is the assigned internship topic.** *Đề tài 3 — Application Development on
  AWS*. The migration is the deliverable, and the platform is the workload chosen
  to carry it.

{{% notice tip %}}
The fourth reason is the honest one and belongs in the document. A proposal that
invents business urgency for an academic project is easy to see through; one that
states the real driver and then does rigorous engineering anyway is not.
{{% /notice %}}

#### Objectives

In priority order — when two conflict, the higher one wins:

| # | Objective | Measured by |
|---|---|---|
| 1 | Run BorrowIt entirely on AWS, no Supabase dependency left | The app works with Supabase credentials revoked |
| 2 | Define all infrastructure as code | `cdk destroy` then `cdk deploy` reproduces the environment |
| 3 | Stay inside the credit budget | Cost Explorer, filtered on `Project=BorrowIt` |
| 4 | Make the system observable | Dashboard and alarms that fire before a user notices |
| 5 | Document it so someone else can rebuild it | The section 4 workshop, followed by a reader with an empty account |

Objective 3 is the one that constrains the design most, and
[2.6](../2.6-Budget/) shows why.

#### Constraints accepted up front

+ **Region `ap-southeast-1`.** Users are in Vietnam; Singapore is the nearest
  region with the full service set.
+ **$160 of Free Plan credits, expiring 28/01/2027.** The account post-dates the
  15 July 2025 change, so **there is no 12-month free tier** — RDS bills from the
  first hour.
+ **One engineer, part-time, twelve weeks.** This rules out anything that needs a
  team to operate.
+ **The application source is treated as fixed.** Changes to it are limited to
  what the migration requires — configuration, health check endpoints, the storage
  client. No feature work.

#### Non-objectives

Named here so that not doing them reads as a decision rather than an omission:

+ Zero-downtime cutover. A short maintenance window is acceptable.
+ Production-grade compliance posture. See the tradeoff in
  [4.3.3](../../4-Workshop/4.3-Architecture/4.3.3-networking/).
+ Cost optimisation beyond the credit budget. The target is "affordable and
  understood", not "minimal".
