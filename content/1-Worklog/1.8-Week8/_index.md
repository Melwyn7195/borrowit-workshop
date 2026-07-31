---
title: "Week 8 Worklog"
date: 2026-07-28
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

{{% notice note %}}
**Planned.** This week has not happened yet — the plan below is what is scheduled.
Replace it with what actually occurred, and fill in the completion dates.
{{% /notice %}}

### Week 8 Objectives

* Deploy `BorrowitFrontend` — S3 buckets and a CloudFront distribution.
* Serve the React build and user uploads without making either bucket public.
* Resolve, or document, the HTTP/HTTPS mixed-content problem.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - CloudFront concepts: distributions, behaviours, cache policies, price classes <br> - Origin Access Control vs the legacy Origin Access Identity | 03/08/2026 | | |
| 3 | - Write `FrontendStack`: two private buckets, one distribution, a `/uploads/*` behaviour pointing at the second origin | 04/08/2026 | | |
| 4 | - Handle SPA routing: map 403 and 404 to `/index.html` with a 200 so React Router resolves deep links | 05/08/2026 | | |
| 5 | - Build the React app with `VITE_API_URL`, sync to S3, invalidate the cache | 06/08/2026 | | |
| 6 | - Investigate the mixed-content failure; evaluate ACM on the ALB against routing the API through CloudFront | 07/08/2026 | | |

### Week 8 Planned Outcomes

* Both buckets private, readable only through CloudFront via Origin Access Control.
* A single distribution serving the web app and user uploads from separate origins.
* A decision recorded on HTTPS for the API, with reasoning.

<!-- TODO(prose): the mixed-content problem is a real unresolved issue in the
     project as it stands - CloudFront is HTTPS, the ALB is HTTP, and browsers
     block the call. Whatever you decide (buy a domain and use ACM, put the API
     behind the same distribution, or demo the frontend locally), record the
     decision and the reasoning here rather than leaving the gap unexplained. -->
