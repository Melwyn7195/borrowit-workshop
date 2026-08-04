---
title: "Sharing and Feedback"
date: 2026-07-28
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

Personal reflections on the First Cloud AI Journey programme, intended to help the
FCAJ team improve it for future cohorts.

<!-- TODO(prose): this is a DRAFT written from the project repository, not from
     your experience. Sections 1, 2, 5 and 6 in particular are guesses — rewrite
     them from what actually happened. Honest criticism is more useful to the team
     than praise, and it is not penalised. -->

### Overall Evaluation

**1. Working Environment**

The programme ran mostly remotely, with scheduled visits to the AWS Vietnam
office. That arrangement suited the work: building infrastructure means long
uninterrupted stretches waiting on deployments and reading documentation, and
that is easier at a desk of your own than in an open office. The office sessions
were more useful for the things remote work is bad at — asking a question that
would take three messages to write down, and seeing how other interns had
approached the same problem differently.

**2. Support from Mentor / Team Admin**

Guidance was given mostly as pointers rather than direct answers: a service to
look at, a documentation page, a question about why I had made a particular
choice. That was frustrating in the moment and clearly the right approach in
hindsight. The most valuable interventions were not solutions but challenges to
my assumptions — being asked what a component actually cost per month, or why a
resource needed to be public, forced me to justify designs I had copied from
tutorials without understanding.

Administrative support was generally timely. Documents and confirmations were
handled without much chasing.

**3. Relevance of Work to Academic Major**

My coursework in Computer Science covered the application layer well — data
structures, databases, web development, the Express and React work in BorrowIt
was familiar ground. What it did not cover at all was everything underneath:
VPCs and subnet routing, IAM policies, container orchestration, load balancers,
and above all the economics of running any of it.

**4. Learning & Skill Development Opportunities**

Concretely, things I can do now that I could not in June:

- Design a VPC — subnets, route tables, security groups, VPC endpoints — and
  justify the monthly cost of every component in it.
- Write a multi-stack AWS CDK application in TypeScript with dependencies
  pointing one direction, so that a single stack can be torn down and rebuilt
  without CloudFormation refusing over an exported value.
- Containerise an Express application, push it to ECR, and run it on Fargate
  behind an Application Load Balancer.
- Serve a SPA and its API from one CloudFront distribution, and explain why —
  mixed-content blocking and `SameSite=strict` cookies both quietly break the
  obvious alternative.
- Manage database credentials through Secrets Manager and injected task
  definition secrets, rather than environment files.
- Read a bill. Estimate a monthly cost before deploying, then check the estimate
  against Cost Explorer afterwards and understand the difference.

The most durable lesson is smaller than any of those: infrastructure failures are
almost never mysterious. They are recorded somewhere — in a CloudFormation event,
a CloudWatch log stream, a security group rule — and the skill is knowing where
to look and being patient enough to look properly.

---

### Additional Questions

**What did you find most satisfying during your internship?**

Watching the whole environment come up from nothing. Running `cdk deploy` against
an empty account and getting back a working URL — network, database, API, CDN,
all of it defined in code and reproducible by anyone who follows the workshop —
is a different feeling from getting a local development server to run. Everything
in that system exists because I chose it, and I can explain the reason and the
price of every piece.

The runner-up is narrower: the first time I diagnosed a production-shaped bug
from logs alone, without guessing, and the fix worked the first time.

---
