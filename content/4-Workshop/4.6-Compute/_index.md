---
title : "Compute layer"
date : 2026-07-28
weight : 6
chapter : false
pre : " <b> 4.6. </b> "
---

`BorrowitApp` costs roughly **$35/month** with one task and a load balancer
running continuously. It is the only *disposable* stack of the four — it imports
from the other three and nothing imports from it, which is what makes it safe to
destroy and rebuild on its own. (`BorrowitFoundation` costs slightly more, at
~$36, but it is permanent; see [3.4](../4.4-Network/).)

The reasoning for choosing Fargate is in
[4.3.1](../4.3-Architecture/4.3.1-compute/); this section builds it.

#### What gets created

| Resource | Purpose | Cost/month |
|---|---|---|
| ECS cluster | Logical grouping; no cost of its own | $0 |
| Fargate task definition + service | Runs the container | ~$9 |
| Application Load Balancer | Public entry point, health checking | ~$17 |
| Target group | Routes to healthy tasks | $0 |
| CloudWatch log group | Container stdout/stderr | ~$0.50 |
| Dashboard + 7 alarms + SNS topic | Covered in [3.8](../4.8-Observability/) | ~$4.80 |

#### Content

1. [Container image and registry](4.6.1-image-and-registry/)
2. [Task definition and service](4.6.2-task-and-service/)
3. [Load balancer and health checks](4.6.3-load-balancer/)
4. [Initialising the database](4.6.4-database-init/)
