---
title : "Introduction"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 3.1. </b> "
---

#### The workload

The system is a rental marketplace: a REST API over a relational database, with a
browser client and user-uploaded images. In AWS terms that means four
requirements, and the whole architecture follows from them:

| Requirement | What it needs from AWS |
|---|---|
| Serve HTTP requests continuously | Compute with a stable entry point |
| Store relational data durably | A managed database, privately reachable |
| Store and serve user images | Object storage with a caching layer |
| Keep credentials out of code | A secret store integrated with the compute layer |

<!-- TODO(prose): one short paragraph on what the application does. Keep it brief -
     the rest of the workshop is about AWS, and a reader does not need the domain
     model to follow it. -->

#### The four stacks

The infrastructure is split into four CDK stacks, divided by **cost and lifecycle**
rather than by function:

| Stack | AWS services | Cost/month | Lifecycle |
|---|---|---|---|
| `BorrowitFoundation` | VPC, security groups, ECR, S3 gateway + 5 interface endpoints | ~$36 | Permanent |
| `BorrowitApp` | ECS, Fargate, ALB, CloudWatch, SNS, Secrets Manager | ~$35 | Disposable |
| `BorrowitData` | RDS, Secrets Manager | ~$16 | Permanent — holds state |
| `BorrowitFrontend` | S3 ×2, CloudFront | ~$1 | Permanent |

Roughly **$88/month** in total. Note that the most expensive stack is a permanent one:
the interface VPC endpoints keep AWS API traffic off the public internet, and
they bill whether or not the application is deployed.

Dependencies point **one way**: `BorrowitApp` imports from the other three, and
nothing imports from `BorrowitApp`.

That constraint is what makes the expensive layer disposable. CloudFormation
refuses to delete a stack whose outputs another stack is importing, so if
observability or the load balancer were referenced by a fifth stack, `cdk destroy
BorrowitApp` would fail. Keeping the dependency graph acyclic and one-directional
is a deliberate design property, not an accident of ordering.

#### Architecture

![BorrowIt target architecture on AWS](/images/3-Workshop/3.1-Overview/architecture.png?width=100pc)

#### Request path

Following one request end to end is the fastest way to see how the services
connect:

1. The browser requests the page. **CloudFront** serves the static bundle from the
   **S3** web bucket through Origin Access Control — the bucket itself is private.
2. The bundle calls the API on a **relative path** — `/api/products`, not an
   absolute hostname. That request goes back to **CloudFront**, which matches the
   `/api/*` behaviour and forwards it to the **Application Load Balancer**. The
   API is same-origin with the page, which is what keeps the session cookie alive
   ([3.7](../3.7-Delivery/)).
3. The ALB checks its target group and forwards to a healthy **Fargate** task on
   port 3456.
4. The task queries **RDS** over port 5432, inside the VPC. It authenticates with
   credentials it read from **Secrets Manager** at startup.
5. Product images referenced in the response are fetched by the browser from
   **CloudFront** again, this time from the uploads bucket behind the
   `/uploads/*` behaviour.
6. Throughout, the task writes to **CloudWatch Logs**, and the ALB and ECS publish
   metrics that **CloudWatch alarms** evaluate.

<!-- TODO(prose): write this out in your own words as well. Being able to trace a
     request across six services is the clearest demonstration that you understand
     the architecture rather than having copied it. -->

#### What this workshop does not cover

+ Application code. The API and the web client are treated as artefacts to deploy.
+ CI/CD. Deployments are run from a workstation; automating them is future work.
+ Custom domains and TLS **on the load balancer**. The browser always speaks
  HTTPS to CloudFront; the edge-to-ALB hop does not, and the ALB stays reachable
  directly over HTTP — see [3.7](../3.7-Delivery/) for what that costs and how it
  would be closed.
