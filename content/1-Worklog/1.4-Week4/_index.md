---
title: "Week 4 Worklog"
date: 2026-07-28
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Week 4 Objectives

* Learn AWS CDK well enough to build the project in it rather than in the console.
* Deploy `BorrowitFoundation` — VPC, security groups, ECR.
* Commit to the no-NAT design and make the isolation come from security groups.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - CDK concepts: App, Stack, Construct, L1 vs L2 vs L3 <br> - How CDK synthesises to CloudFormation | 06/07/2026 | 06/07/2026 | <https://docs.aws.amazon.com/cdk/v2/guide/> |
| 3 | - **Practice:** `cdk init`, `cdk bootstrap`, `cdk synth`, `cdk deploy`, `cdk destroy` on a throwaway stack | 07/07/2026 | 07/07/2026 | |
| 4 | - Write `FoundationStack`: VPC with `natGateways: 0`, public + isolated subnets across 2 AZs | 08/07/2026 | 08/07/2026 | |
| 5 | - Add the three security groups, each referencing the previous one rather than a CIDR <br> - Add the free S3 gateway endpoint | 09/07/2026 | 09/07/2026 | |
| 6 | - Add the ECR repository with `imageScanOnPush` and a 10-image lifecycle rule <br> - Deploy and verify in the VPC console | 10/07/2026 | 10/07/2026 | |

### Week 4 Achievements

<!-- TODO(prose): the security group chain is the technical highlight of this
     week. Each group accepts traffic only from the group before it, so the rules
     stay correct when ECS replaces a task and the IP changes. Explain that in
     your own words — it is the answer to "why is a task with a public IP safe?" -->

* Deployed the first stack entirely from code — no console clicks.
* Built a VPC with **no NAT Gateway**, saving ~$33/month, and accepted the
  consequence that Fargate tasks would need public subnets.
* Implemented isolation through a chain of security groups (ALB → service →
  database) rather than through subnet placement.
* Put the ECR repository in the foundation layer rather than the app layer, so
  that destroying the app would not discard pushed images.

