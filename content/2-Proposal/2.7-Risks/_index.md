---
title : "Risks and success criteria"
date : 2026-07-28
weight : 7
chapter : false
pre : " <b> 2.7. </b> "
---

#### Risk register

Likelihood and impact are judged against the **21/08/2026** deadline — a risk that
costs two days matters far more in week 9 than in week 4.

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Fargate task cannot pull its image or read its secret, and the service never stabilises | High | High | Deploy the foundation and data stacks first; verify the task role and the ECR repository before creating the service. Read the **stopped-task reason**, not the service events |
| R2 | Health check misconfigured — the ALB kills a healthy container | High | Medium | Separate `/healthz` (liveness, no DB) from `/readyz` (readiness, checks the DB); generous `startPeriod` |
| R3 | Data export from Supabase does not restore cleanly | Medium | High | Do the restore in week 5, not at cutover; keep `db/seed_data.sql` as a rebuild path |
| R4 | Credits exhausted before the deadline | Low | High | ~$86/month against $160 with under a month to run; budget alarm at $10; `BorrowitApp` is disposable, and the ~$36 of VPC endpoints can be dropped if the deadline slips |
| R5 | A security group change accidentally exposes the task or database | Low | High | Ingress rules reference security groups, never CIDRs; verify with a direct connection attempt after every network change |
| R6 | The single task fails and there is nothing to fail over to | Medium | Medium | Accepted. ECS restarts it in 1–2 minutes; the alarm reports it. Scaling to two tasks is one parameter if it becomes a problem |
| R7 | Multi-AZ left switched on after the failover demo, doubling RDS cost | Medium | Low | It is enabled by an explicit context flag, `-c multiAz=true`, so it is off by default and reverting is a redeploy |
| R8 | Screenshots and evidence not captured before teardown | Medium | Medium | Capture evidence in the same session as the work; the workshop marks every image slot with a `SCREENSHOT` comment |
| R9 | Time lost to CDK or CloudFormation errors that are hard to read | High | Medium | Two-week buffer in weeks 11–12; `cdk diff` before every deploy |

R1 and R2 are the two rated *High/High* and *High/Medium*, and both land in the
same week. That is why phase 2 in [2.5](../2.5-Plan/) is given two weeks and
flagged as the schedule risk.

{{% notice warning %}}
**R7 is the one that quietly costs money.** Multi-AZ doubles the RDS instance
charge and nothing in the console shouts about it. Whoever runs the failover
demonstration turns it back off the same day.
{{% /notice %}}

#### Risks that were accepted, not mitigated

Being explicit about these is the point of the section:

+ **No NAT Gateway** — a security-group mistake exposes the task directly. Accepted
  because the alternative costs more than the entire compute budget. Full argument
  in [4.3.3](../../4-Workshop/4.3-Architecture/4.3.3-networking/).
+ **No HTTPS on the API** — there is no domain to validate a certificate against.
  Accepted for a demo; it would not be acceptable with real users.
+ **`removalPolicy: DESTROY` on the database** — destroying `BorrowitData` drops
  the data. Accepted because the data is seed data and `db/seed_data.sql` rebuilds
  it; it would be indefensible with real user data.
+ **No CI/CD** — every deploy is manual and unlogged. Accepted at one engineer.

#### Success criteria

The project is finished when every row below has evidence attached — a screenshot,
a command output, or a section of the workshop.

| # | Criterion | Evidence |
|---|---|---|
| S1 | The application works end to end on AWS with no Supabase dependency | Browser walkthrough: list an item, upload an image, reload |
| S2 | The whole environment is reproducible from code | `cdk destroy BorrowitApp` then `cdk deploy BorrowitApp`, working again |
| S3 | The database is unreachable from the internet | A connection attempt from outside that times out |
| S4 | The API is unreachable except through the load balancer | `curl` against the task's public IP, timing out |
| S5 | No credential exists outside Secrets Manager | The rendered task definition, showing `valueFrom` rather than a value |
| S6 | Failure is detected without looking at the app | An alarm email triggered by a stopped task |
| S7 | The database survives an AZ failure | A forced Multi-AZ failover, with the recovery time recorded |
| S8 | Spend is inside budget and understood | Cost Explorer beside the estimate in [2.6](../2.6-Budget/) |
| S9 | Someone else can rebuild it | A peer completing the section 4 workshop from an empty account |

<!-- TODO(prose): at the end of the project, come back and mark each row honestly.
     A criterion that was not met, with an explanation, reads far better than nine
     unqualified ticks - and section 5 is where the marker looks for exactly that
     kind of self-assessment. -->

#### Approval

| | |
|---|---|
| Prepared by | *[your name]* — FCAJ intern |
| Date | 28/07/2026 |
| Reviewed by | *[mentor name]* |
| Decision | *[approved / approved with changes / rejected]* |

<!-- TODO: fill in the table above. If this proposal was never formally reviewed,
     say so rather than inventing a signature - and note who you did discuss the
     architecture with. -->
