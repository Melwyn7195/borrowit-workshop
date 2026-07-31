---
title : "Choosing the AWS services"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 4.3. </b> "
---

AWS usually offers three or four services that will technically do the job. The
engineering is in choosing between them and being able to say why.

This section records each decision as it was actually made: the candidates
considered, the criteria applied, the choice, and — the part most write-ups skip —
**what the choice costs you**. Every decision here is reversible on paper and
expensive in practice, which is why they are made before any code in
[3.4](../4.4-Network/) onwards.

#### Decision summary

| Layer | Chosen | Main alternatives rejected | Deciding factor |
|---|---|---|---|
| Compute | **ECS on Fargate** | Lambda, EC2, ECS on EC2, App Runner | Long-lived process with a connection pool; no servers to patch |
| Database | **RDS PostgreSQL** | Aurora Serverless v2, DynamoDB, Postgres on EC2 | Existing relational schema; predictable fixed cost |
| Outbound networking | **Public subnets, no NAT** | NAT Gateway, VPC endpoints for everything | NAT costs more than the entire compute budget |
| Static delivery | **S3 + CloudFront** | Amplify Hosting, serving from the ALB | Cost, and keeping buckets private via OAC |
| Secrets | **Secrets Manager** | SSM Parameter Store, plain env vars | Native RDS integration generates and stores the password |
| Registry | **ECR** | Docker Hub, GitHub Container Registry | IAM-native auth, same-region pulls, scan on push |
| Monitoring | **CloudWatch + SNS** | Container Insights, X-Ray, third-party | Sufficient signal at ~$4.80/month |

#### A note on the constraint

Every decision below was made against a fixed balance of **$160 in AWS Free Plan
credits**, on an account created after the 15 July 2025 free-tier change — so
there is **no 12-month free tier**, and RDS is billed from the first hour.

That constraint is stated up front because it is the deciding factor in three of
the seven rows above. A different budget would have produced a different
architecture, and saying so is more honest than presenting these as universally
correct choices.

#### Content

1. [Compute — why Fargate rather than Lambda](4.3.1-compute/)
2. [Database — why RDS rather than Aurora Serverless or DynamoDB](4.3.2-database/)
3. [Networking — why no NAT Gateway](4.3.3-networking/)
4. [Delivery, storage and secrets](4.3.4-delivery-and-secrets/)
