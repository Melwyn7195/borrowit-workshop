---
title: "Week 2 Worklog"
date: 2026-07-28
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Week 2 Objectives

* Learn the core services the migration would need: IAM, VPC, EC2, S3, RDS.
* Understand VPC networking well enough to make a deliberate choice about NAT
  Gateways later.
* Practise each service by hand in the console before automating it.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - IAM: users, groups, roles, policies <br> - Principle of least privilege; why roles beat long-lived access keys | 22/06/2026 | 22/06/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 3 | - VPC: subnets, route tables, internet gateway, NAT gateway <br> - Security groups vs network ACLs | 23/06/2026 | 23/06/2026 | |
| 4 | - **Practice:** build a VPC by hand — public and private subnets, launch an EC2 instance, connect via Session Manager | 24/06/2026 | 24/06/2026 | |
| 5 | - S3: buckets, object keys, storage classes, bucket policies, Block Public Access <br> - **Practice:** static site hosting, then the same site behind CloudFront | 25/06/2026 | 25/06/2026 | |
| 6 | - RDS basics: engines, instance classes, Multi-AZ, backups <br> - Price out NAT Gateway, ALB, Fargate and RDS against the credit balance | 26/06/2026 | 26/06/2026 | <https://calculator.aws/> |

### Week 2 Achievements

<!-- TODO(prose): the pricing exercise on day 6 is the interesting one. A NAT
     Gateway at ~$33/month against the whole project budget is what forced the
     public-subnet design in week 4. If that is when you worked it out, say so —
     it turns a cost constraint into a documented design decision. -->

* Built a VPC by hand and connected to an EC2 instance without opening SSH to the
  internet, using Session Manager.
* Understood the difference between security groups (stateful, instance-level) and
  NACLs (stateless, subnet-level), and when each is the right tool.
* Priced the candidate architecture against the credit balance and identified the
  NAT Gateway as the single largest avoidable cost.
* Practised S3 static hosting and CloudFront, which became the frontend design in
  week 8.
