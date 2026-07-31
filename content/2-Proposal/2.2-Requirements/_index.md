---
title : "Scope and requirements"
date : 2026-07-28
weight : 2
chapter : false
pre : " <b> 2.2. </b> "
---

Requirements are written so that each one can be **checked**, not argued about.
Anything that cannot be demonstrated at the end of the project does not belong on
this page.

#### Functional requirements

| ID | Requirement | Verified by |
|---|---|---|
| F1 | The web client loads over HTTPS from a global endpoint | Open the CloudFront URL |
| F2 | The API answers the same REST contract as it does on Supabase | Existing client works unmodified against the new base URL |
| F3 | The API persists to a managed PostgreSQL database | Create a listing, confirm the row in `psql` |
| F4 | Item images upload and are served back to the client | Upload through the app, fetch the object URL |
| F5 | The existing schema and seed data are restored intact | `db/seed_data.sql` applied, row counts match |
| F6 | The API exposes a liveness and a readiness endpoint | `GET /healthz` and `GET /readyz` return 200 |

F6 is new work on the application. The health checks did not exist before and are
required by the load balancer — the one application change this migration forces.

#### Non-functional requirements

| ID | Requirement | Target | Verified by |
|---|---|---|---|
| N1 | The database is unreachable from the internet | No public route at all | Attempt to connect from a laptop; it must fail |
| N2 | The API container is unreachable except through the load balancer | Security-group ingress only from the ALB | `curl` the task's public IP directly; must time out |
| N3 | No credential appears in source, `.env`, or a task definition | Zero | `grep` the repo; read the rendered task definition |
| N4 | A failed deployment rolls back automatically | No manual intervention | Deploy a deliberately broken image and watch it revert |
| N5 | Logs from the API are queryable centrally | Retained 1 week | A Logs Insights query returning application errors |
| N6 | An unhealthy system raises an alarm | Email within ~5 minutes | Stop the task, wait for the SNS mail |
| N7 | The database survives the loss of an Availability Zone | Recovery without data loss | Multi-AZ enabled, failover forced, measured |
| N8 | AWS API traffic from the workload does not cross the public internet | Image pulls, secret reads and log delivery over PrivateLink | `describe-vpc-endpoints` shows five interface endpoints with private DNS |
| N9 | Monthly run cost stays inside the credit budget | ≤ $90/month, and credits outlast the deadline | Cost Explorer against the estimate in [2.6](../2.6-Budget/) |

{{% notice note %}}
**N7 and N9 are in direct tension.** Multi-AZ doubles the RDS instance cost. The
resolution proposed here is to run single-AZ and enable Multi-AZ on demand for the
failover demonstration, then turn it back off — which satisfies N7 as a
*demonstrated capability* rather than a *standing configuration*. That is a real
weakening of the requirement and is called out rather than hidden.
{{% /notice %}}

{{% notice note %}}
**N8 and N9 are the more expensive tension**, and unlike N7 it was resolved the
other way. Satisfying N8 costs ~$36/month in interface VPC endpoints — more than
the load balancer — and it is what moved the N9 ceiling from $50 to $90. The
alternative was to leave the endpoints as an optional flag and treat N8 as
demonstrated-not-standing, exactly as N7 is. That was rejected: a security control
that is off during normal running is not a control, and the credits still cover the
deadline. See [2.6](../2.6-Budget/) for the arithmetic that makes it fit.
{{% /notice %}}

#### Out of scope

| Not doing | Why | Cost of not doing it |
|---|---|---|
| CI/CD pipeline | Deploys are infrequent and run by one person | Every deploy is manual and unlogged |
| Custom domain and TLS on the ALB | No domain budget; ACM needs a domain to validate | API traffic is HTTP; only the CloudFront leg is HTTPS |
| Multi-region | Doubles everything for a single-region audience | A region-wide outage takes the system down |
| Auto-scaling policies | One task is enough for demo traffic | A traffic spike is served by one container |
| WAF, GuardDuty, Config | Each adds recurring cost against a fixed credit balance | The security posture is security groups and IAM only |
| Application feature work | The app is a workload, not the subject | — |

The third column is the part most scope tables leave out. Listing what each
exclusion costs is what makes it a scope decision instead of a gap.

#### Assumptions

+ Traffic is demo-scale: tens of requests per minute, not thousands.
+ A single maintenance window for the data cutover is acceptable.
+ The container image builds and runs locally before any AWS work starts.
+ AWS credit balance and pricing hold for the project duration.

<!-- TODO(prose): if any of these assumptions broke during the project, that is
     worth a sentence here and a fuller note in section 4. An assumption that
     failed and was handled is stronger evidence than one that happened to hold. -->
