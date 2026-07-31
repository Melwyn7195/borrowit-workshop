---
title : "Blogs"
date : 2026-07-31
weight : 3
chapter : true
pre : " <b> 3. </b> "
---

# Blogs Posted

The FCAJ stamp requires **3 blog posts** published to the AWS Study Group over
the course of the internship, alongside the finished project, the finished
report, 3 months of duration and 10 office visits. This section lists the
three posts and what each one covers.

All three write up a decision actually made in [section 4](../4-Workshop/) of
this workshop — they are the same lessons, told to a wider audience before the
workshop itself was finished.

## Blog 1 — Infrastructure as code: lessons from a first time with AWS CDK

Writes up dropping console clicks in favour of 100% AWS CDK for BorrowIt's
infrastructure. Covers why reproducibility (not speed) is the real argument
for IaC on a limited budget, splitting the four stacks by **lifecycle and
cost** rather than by service type, why cross-stack dependencies have to flow
one way (`BorrowitApp` reading from the other three, never the reverse) so
that `cdk destroy BorrowitApp` doesn't fail on a CloudFormation export, and
how `removalPolicy` decides whether `cdk destroy` actually deletes data.

<!-- SCREENSHOT: /images/3-Blogs/blog-1-cdk.png
     The published post on AWS Study Group. -->

## Blog 2 — One CloudFront for both frontend and API: lessons on HTTPS, cookies and caching

Writes up the decision to serve the SPA and the API from the **same
CloudFront distribution** instead of pointing the frontend at the load
balancer directly. Covers two silent failures that rule out the usual
two-origin setup: an HTTPS page cannot call an `http://` load balancer
(mixed content, blocked with no visible error), and a `sameSite: 'strict'`
session cookie gets silently dropped across origins, so login returns 200 but
every request after it is anonymous. Putting both behind one domain fixes
both and removes the need for CORS entirely.

<!-- SCREENSHOT: /images/3-Blogs/blog-2-cloudfront.png
     The published post on AWS Study Group. -->

## Blog 3 — What I learned running the backend on AWS Fargate

Writes up why the backend runs on **Fargate** rather than a cheaper EC2
instance: no OS to patch, no instance to forget running, and deploys that
replace an image instead of touching a server — trading money for time on a
one-person, deadline-bound project. Covers the fixed CPU/RAM pairs Fargate
accepts (BorrowIt runs the smallest, 256 CPU / 512 MiB), and the detail that
surprised the author most: in the ALB + one Fargate task setup (~$31/month),
the **load balancer costs more than the task**, since the ALB bills hourly
regardless of traffic.

<!-- SCREENSHOT: /images/3-Blogs/blog-3-fargate.png
     The published post on AWS Study Group. -->
