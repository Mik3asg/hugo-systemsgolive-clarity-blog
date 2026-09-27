---
title: "Why Your SaaS Cloud Bill Keeps Growing; Where to Look First"
date: 2026-09-26
draft: false
description: "A quick diagnosis for early-stage SaaS companies whose AWS bill is growing faster than the business: where to look, what to spot, and what fixing it involves."
summary: "Most cloud waste in early-stage SaaS sits in the same few places. Here's where to look, what to spot, and why fixing it takes more care than it seems."
tags: ["aws", "cost-optimisation", "finops", "saas", "startups", "consulting"]
categories: ["Consulting"]
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

## How I work

I'm an independent DevOps and cloud engineer. A cost review with me typically runs over two to three weeks:

1. Read-only access to your AWS account and a short call with whoever knows the setup best.
2. A full review of your spend and infrastructure, based on real usage data.
3. A short report listing each finding with its estimated saving, effort and risk, in priority order.
4. Implementation, either by your team with my support or by me working alongside them. All changes go through code and review.
5. Guardrails before I step away: tagging, budgets, alerts and a simple monthly routine, so the bill stays under control.

## What to expect

For an account that has never been properly reviewed, a 30 to 50% reduction is a realistic range. If your setup is already in good shape, I'll tell you, and the review stays short.

You also come out of it knowing where your money goes. That matters when you're managing runway or preparing for a fundraise.

## Is this for you?

It's probably worth a conversation if:

- your cloud bill is somewhere between £2k and £20k a month and rising
- you don't have a dedicated DevOps or platform engineer
- nobody can explain last month's bill with confidence

## Get in touch

I've spent over 14 years in infrastructure, including regulated environments where cost, security and reliability all need to hold up together. I offer a fixed-price cloud cost review for early-stage and growing SaaS companies. No long contracts, no lock-in.

If your cloud bill is growing faster than your business, get in touch via [systemsgolive.com](https://systemsgolive.com).
