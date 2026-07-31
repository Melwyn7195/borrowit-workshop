---
title: "Week 3 Worklog"
date: 2026-07-28
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Week 3 Objectives

* Audit what BorrowIt actually depends on in Supabase.
* Choose a compute target and justify it.
* Produce a target architecture diagram and a cost estimate to work from.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - Read the BorrowIt codebase: Express routes, controllers, models, `db/index.js` <br> - List every Supabase-specific dependency | 29/06/2026 | 29/06/2026 | |
| 3 | - Map each dependency to an AWS service: Postgres → RDS, storage → S3, auth → existing JWT code retained | 30/06/2026 | 30/06/2026 | |
| 4 | - Compare compute options: EC2, ECS on EC2, **ECS Fargate**, Lambda <br> - Decide on Fargate and write down why | 01/07/2026 | 01/07/2026 | |
| 5 | - Draw the target architecture <br> - Decide the four-stack split by cost and lifecycle | 02/07/2026 | 02/07/2026 | |
| 6 | - Build the cost estimate in AWS Pricing Calculator | 03/07/2026 | 03/07/2026 | <https://calculator.aws/> |

### Week 3 Achievements

<!-- TODO(prose): the compute comparison is worth several sentences of your own.
     Lambda is cheaper at idle but the app is a long-lived Express server with a
     connection pool; EC2 is cheaper at steady load but you have to patch it.
     Fargate was the choice — say what you traded away to get it. -->

* Produced a complete inventory of Supabase dependencies and their AWS equivalents.
* Chose **ECS Fargate** as the compute target: no servers to patch, and the Express
  app runs unmodified in a container.
* Designed the four-stack split — `Foundation`, `Data`, `Frontend`, `App` —
  separated by **how much each costs**, so that the expensive layer could be
  destroyed independently.
* Produced a costed architecture: ~$50/month for the full stack, against a fixed
  credit balance.

![First architecture sketch](/images/1-Worklog/1.3-Week3/architecture-draft.png?width=100pc)
