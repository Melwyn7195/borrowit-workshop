---
title: "Week 7 Worklog"
date: 2026-07-28
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Week 7 Objectives

* Deploy `BorrowitApp` — ECS cluster, Fargate service, Application Load Balancer.
* Get the API answering requests over the internet.
* Load the schema into RDS without giving the database a public endpoint.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - ECS concepts: cluster, task definition, service, target group <br> - Write the task definition with Secrets Manager injection | 27/07/2026 | 27/07/2026 | |
| 3 | - Separate liveness (`/health/live`, no DB) from readiness (`/health`, runs `SELECT 1`) <br> - Add the ALB, listener and target group | 28/07/2026 | | |
| 4 | - Add `circuitBreaker` for automatic rollback and `minHealthyPercent: 50` so a deploy never runs two tasks <br> - First successful `cdk deploy BorrowitApp` | 29/07/2026 | | |
| 5 | - Enable `enableExecuteCommand`, install the Session Manager plugin <br> - Load schema and seed data from inside a running task | 30/07/2026 | | |
| 6 | - Verify the API end to end; confirm the task's public IP is unreachable on 3456 <br> - Tune `deregistrationDelay` against the app's drain delay | 31/07/2026 | | |

### Week 7 Achievements

<!-- TODO(prose): this week is the core of the whole project — write it up
     properly. The two-health-check split is the single most defensible design
     decision in the report: a DB-dependent liveness check would turn a brief
     database blip into an endless restart loop. Explain it in your own words.

     Also record how you actually got the schema into an isolated database. The
     alternatives were a bastion host (~$8/month, another machine to patch) or
     making RDS public (defeats the design). ECS Exec cost nothing and left
     nothing behind. -->

* Got the Express API running on Fargate behind an ALB, reachable over the
  internet.
* Designed liveness and readiness checks to answer genuinely different questions,
  so a database problem degrades the service instead of restarting it.
* Loaded the schema into an isolated database using `aws ecs execute-command`,
  with no bastion host and no public endpoint.
* Demonstrated that the task's public IP is not reachable on port 3456 from
  outside — only the load balancer can reach it.

