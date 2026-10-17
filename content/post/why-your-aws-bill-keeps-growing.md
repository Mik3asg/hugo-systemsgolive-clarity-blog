---
title: "Why Your SaaS Cloud Bill Keeps Growing; Where to Look First"
date: 2026-09-26
draft: false
description: "A quick diagnosis for early-stage SaaS companies whose AWS bill is growing faster than the business: where to look, what to spot, and what fixing it involves."
summary: "Most cloud waste in early-stage SaaS sits in the same few places. Here's where to look, what to spot, and why fixing it takes more care than it seems."
tags: ["aws", "cost-optimisation", "finops", "saas", "startups"]
categories: ["AWS", "Cloud Cost"]
thumbnail: "images/cloud-cost-increasing-logo.png"
---

A lot of early-stage SaaS companies end up in the same spot. Customers are growing, the product is moving, and the AWS bill is growing even faster. When someone finally asks why, nobody has a clear answer.

That's normal. At this stage the team is focused on shipping, and nobody really owns the bill. Engineering spends it, finance pays it, and it keeps creeping up month after month.

Most of the waste sits in the same few places. Below is a quick diagnosis you can run yourself, followed by what fixing it actually involves.

## Quick diagnosis: where to look

Start in Cost Explorer. Go back 6 to 12 months and group by service, then group again by usage type. Grouping by service alone hides a lot, because NAT gateway, EBS and data transfer costs all sit under one line called "EC2-Other".

| Area | Where to look | What to spot |
|---|---|---|
| Overall spend | Cost Explorer, grouped by service and by usage type | Unexplained jumps, large "EC2-Other" line |
| Non-prod environments | Cost Explorer by tag or account, EC2 and RDS consoles | Test or demo stacks running 24/7 |
| EC2 / ECS / Fargate | CloudWatch (EC2 memory needs the agent), Compute Optimizer | Low CPU at peak, not just average |
| RDS | CloudWatch, Performance Insights, RDS console | Oversized, Multi-AZ in non-prod, old engine versions |
| NAT gateway | Cost Explorer, usage type "NatGateway-Bytes" (inside EC2-Other) | High data processing charges |
| Public IPs and load balancers | VPC console (public IPv4), EC2 console (load balancers) | Unused public IPs, idle load balancers |
| Data transfer | Cost Explorer, usage types containing "DataTransfer" | Rising cross-AZ traffic costs |
| CloudWatch Logs | CloudWatch console, Cost Explorer usage types for CloudWatch | High ingestion volume, no retention set |
| EBS and snapshots | EC2 console, Volumes and Snapshots | Unattached volumes, old snapshots and AMIs |
| S3 | S3 Storage Lens | No lifecycle rules, old versions piling up |
| Pricing | Cost Explorer, Savings Plans and RI coverage reports | Steady workloads on on-demand pricing |
| Tagging | Tag Editor, Billing console (cost allocation tags) | Untagged resources, tags not activated for billing |

If you tick three or more of these, there's very likely money to recover.

## What fixing it involves

Spotting the problem is easy. Fixing it without breaking something is harder.

**Non-prod environments.** Some can go, some must stay. Knowing which is the hard part.

**Compute and databases.** Size down too far and you find out at your busiest hour.

**Database versions.** Engine versions past end of standard support are charged extra. Upgrading only helps if the new version is still supported, and it needs careful testing.

**NAT gateway.** Usually traffic taking an expensive route. Fixing it means network changes.

**Public IPs and load balancers.** Both are charged every hour, even with no traffic. But a customer's firewall rule or a DNS record may still point to them.

**Data transfer.** Often down to how services talk to each other. Changing it touches your architecture.

**Logs.** Setting retention is quick. Logging less without losing what you need takes thought.

**Volumes and snapshots.** It might be the only copy of something. Check before deleting.

**S3.** Wrong lifecycle rules can move or delete data too early.

**Pricing commitments.** Commit too early or too much and you pay for capacity you don't use.

**Tagging.** Without it, every cost conversation is guesswork.

## A sensible order to tackle it

Cost work goes wrong when it's rushed. This is the order I'd follow:

1. **Get the data first.** Read-only access to Cost Explorer and the main consoles, and 30 minutes with whoever knows the setup best. Most surprises come from things nobody remembers creating.
2. **Review against real usage.** Size from peak load, not averages, and check what each resource is actually for before touching it.
3. **List findings by saving, effort and risk.** Quick, safe wins first: log retention, unattached volumes, idle IPs. Leave rightsizing and network changes until they've been tested.
4. **Change everything through code and review.** A console change that saves money today is drift tomorrow, and it's the one nobody can explain in six months.
5. **Put guardrails in place.** Tagging, budgets, cost alerts and a short monthly review, so the bill doesn't creep back up.

## Final thoughts

A growing cloud bill is rarely one big mistake. It's a dozen small ones that nobody owns. The diagnosis is the easy part and takes an afternoon. The fixes need the same care as any other production change, because that's what they are.
