---
title: "Week 6 Worklog"
date: 2026-07-28
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Week 6 Objectives

* Containerise the Express API.
* Push a working image to ECR.
* Move file uploads from the old bucket to an S3 bucket the project owns.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - Docker fundamentals: layers, build cache, multi-stage builds <br> - Write the first `Dockerfile` for the API | 20/07/2026 | 20/07/2026 | |
| 3 | - Debug the `bcrypt` build failure on Alpine; switch the base image to `node:22-bookworm-slim` <br> - Split dependencies into their own stage for cache reuse | 21/07/2026 | 21/07/2026 | |
| 4 | - Add `USER node`, `HEALTHCHECK` and `.dockerignore` <br> - Test with `docker run` locally against the RDS instance | 22/07/2026 | 22/07/2026 | |
| 5 | - Authenticate Docker to ECR and push the first tagged image <br> - Review the `imageScanOnPush` findings | 23/07/2026 | 23/07/2026 | |
| 6 | - Rework `uploadController` to write to S3 via `multer-s3` using the task role, with no static access keys | 24/07/2026 | 24/07/2026 | |

### Week 6 Achievements

<!-- TODO(prose): the Alpine/bcrypt problem is a good concrete story if it is what
     actually happened — musl vs glibc, prebuilt binaries, and the choice to take a
     slightly larger Debian image rather than compile from source. Check your notes
     before writing it up as fact. -->

* Produced a working container image for the Express API, running as an
  unprivileged user.
* Chose a Debian base over Alpine for a specific, documented reason rather than by
  default.
* Pushed a tagged image to ECR and reviewed its vulnerability scan.
* Replaced static AWS credentials in the upload path with the Fargate task role.

