---
title: "Week 1 Worklog"
date: 2026-07-28
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Week 1 Objectives

* Meet the First Cloud AI Journey team and understand the programme structure.
* Build a working mental model of AWS: regions, availability zones, the shared
  responsibility model, and how billing works.
* Get an AWS account and a working CLI.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - Introduction to the FCAJ programme and mentors <br> - Read the internship rules, reporting format and stamp requirements | 15/06/2026 | 15/06/2026 | |
| 3 | - AWS global infrastructure: regions, AZs, edge locations <br> - Why `ap-southeast-1` for a Vietnam-based project | 16/06/2026 | 16/06/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 4 | - Create the AWS account <br> - **Practice:** enable MFA on the root user, create an admin IAM user, set up a $10 budget alarm | 17/06/2026 | 17/06/2026 | |
| 5 | - Install and configure AWS CLI v2 <br> - **Practice:** `aws configure`, `aws sts get-caller-identity`, `aws freetier get-account-plan-state` | 18/06/2026 | 18/06/2026 | |
| 6 | - Understand the account's billing model: Free Plan credits, no 12-month free tier post-July-2025 <br> - Estimate a first budget for the project | 19/06/2026 | 19/06/2026 | <https://calculator.aws/> |

### Week 1 Achievements

<!-- TODO(prose): rewrite these in your own words. Suggested points:
       - What surprised you about the account's billing model. The absence of the
         12-month free tier changed the whole sizing approach later, so it is worth
         recording that you found it out in week 1 rather than week 8.
       - Whether the budget alarm was actually configured and tested.
       - Anything that did not work first time. -->

* Created and secured an AWS account: MFA on the root user, a separate IAM admin
  user for daily work, and a budget alarm before deploying anything.
* Confirmed the account runs on **AWS Free Plan credits** rather than the legacy
  12-month free tier, which set the cost constraints for the rest of the project.
* Installed and configured the AWS CLI, and verified access with
  `aws sts get-caller-identity`.
* Chose `ap-southeast-1` (Singapore) as the project region — closest region to
  the users, and the one every later resource would be pinned to.
