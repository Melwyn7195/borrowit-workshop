---
title: "Solution Proposal"
date: 2026-07-28
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Migrating BorrowIt to AWS — Solution Proposal

This section is the plan the project was approved on, written at the end of
**week 3** and before any infrastructure code existed. It states the problem, the
requirements the solution has to meet, the target architecture, the schedule, the
budget and the risks.

It is deliberately kept as a **forward-looking document**. Where the build later
diverged from it, the divergence is recorded rather than edited away — a proposal
that matches the outcome perfectly is usually one that was rewritten afterwards.

{{% notice note %}}
Section 2 is the **plan**. [Section 3](../3-Workshop/) is the **record** — the
reproducible build, with the full rationale behind each service choice. If the two
disagree, section 3 is what actually happened.
{{% /notice %}}

#### What this proposal commits to

| | Commitment |
|---|---|
| **Outcome** | BorrowIt running end to end on AWS, no remaining Supabase dependency |
| **Method** | 100% infrastructure-as-code in AWS CDK — no console clicks in the build path |
| **Region** | `ap-southeast-1` (Singapore) |
| **Budget** | Inside $160 of AWS Free Plan credits, expiring 28/01/2027 |
| **Deadline** | Working system and written workshop by **21/08/2026** |
| **Evidence** | A cost model checked against the real bill, and a failover test that is run, not asserted |

#### How to read it

| Sub-section | Question it answers |
|---|---|
| [2.1 Problem and objectives](2.1-Problem/) | What is being migrated, and why bother |
| [2.2 Scope and requirements](2.2-Requirements/) | What the solution must do, and what is explicitly out of scope |
| [2.3 Architecture design](2.3-Design/) | What is being built — with the editable diagram |
| [2.4 Well-Architected review](2.4-WellArchitected/) | Whether the design holds up against the six AWS pillars |
| [2.5 Implementation plan](2.5-Plan/) | In what order, by when, and how each step is verified |
| [2.6 Budget and cost model](2.6-Budget/) | What it costs, and what happens when the credits run out |
| [2.7 Risks and success criteria](2.7-Risks/) | What could go wrong, and how "done" is judged |

<!-- TODO(prose): if this proposal was actually reviewed by a mentor, say so here
     and note what they pushed back on. A proposal with a review history is far
     more convincing than one that appears to have been written unopposed. -->

#### Content

1. [Problem and objectives](2.1-Problem/)
2. [Scope and requirements](2.2-Requirements/)
3. [Architecture design](2.3-Design/)
4. [Well-Architected review](2.4-WellArchitected/)
5. [Implementation plan](2.5-Plan/)
6. [Budget and cost model](2.6-Budget/)
7. [Risks and success criteria](2.7-Risks/)
