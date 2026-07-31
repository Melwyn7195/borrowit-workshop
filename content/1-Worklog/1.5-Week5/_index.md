---
title: "Week 5 Worklog"
date: 2026-07-28
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Week 5 Objectives

* Deploy `BorrowitData` — RDS PostgreSQL 16 in isolated subnets.
* Get the database password out of `.env` files permanently.
* Move the schema and data off Supabase.

### Tasks

| Day | Task | Start Date | Completion Date | Reference |
| --- | ---- | ---------- | --------------- | --------- |
| 2 | - RDS sizing: instance class, storage, Multi-AZ, backup retention <br> - Choose `db.t4g.micro` on 20 GB and justify against the credit balance | 13/07/2026 | 13/07/2026 | |
| 3 | - Write `DataStack`; pin `maxAllocatedStorage` equal to `allocatedStorage` so autoscaling cannot grow the bill <br> - Deploy (~10 min) | 14/07/2026 | 14/07/2026 | |
| 4 | - Secrets Manager: `rds.Credentials.fromGeneratedSecret` <br> - Verify the password never appears in the synthesised template | 15/07/2026 | 15/07/2026 | |
| 5 | - Export the schema from Supabase with `pg_dump --schema-only --no-owner --no-privileges` <br> - Resolve Supabase-specific role grants that fail on RDS | 16/07/2026 | 16/07/2026 | |
| 6 | - Adapt `db/index.js` to assemble `DATABASE_URL` from the injected `DB_*` variables <br> - Test the connection locally | 17/07/2026 | 17/07/2026 | |

### Week 5 Achievements

<!-- TODO(prose): the Supabase export almost certainly did not work on the first
     attempt - role grants, extensions, or RLS policies that RDS rejects. Whatever
     you hit, write it down. A migration report with no friction in it reads as
     though the migration was never actually performed. -->

* Deployed PostgreSQL 16 into isolated subnets with no public endpoint.
* Eliminated the database password from source control: RDS generates it into
  Secrets Manager, and ECS injects it into the task at start.
* Pinned storage autoscaling off so the bill cannot grow silently on a fixed
  credit balance.
* Migrated the schema off Supabase, resolving the incompatibilities that surfaced.

