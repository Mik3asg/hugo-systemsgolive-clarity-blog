---
title: "The Postmortem Mindset Why Engineers Who Write Them Stand Out"
date: 2026-03-08T11:39:31Z
draft: false
tags:
  - postmortem
  - incident-management
  - blameless
  - sre
  - devops
  - infrastructure
categories:
  - SRE
  - DevOps
  - Career
thumbnail: ""
summary: "Fixing incidents is table stakes. What separates good engineers from exceptional ones is what happens after — the postmortem. Learn why writing them well is one of the most underrated habits in infrastructure engineering, and how to do it properly."
---

Most engineers are great at firefighting. The alert fires, you jump in, you find the issue, you push the fix, you close the ticket. Job done. But here's the thing — that's just table stakes. That's the bare minimum. What separates good engineers from truly exceptional ones isn't just how fast they fix things. It's what they do after.

Writing a postmortem is one of those habits that will genuinely make you stand out — not just to your team, but to leadership, to hiring managers, and frankly to yourself.

---

## Wait, What Even Is a Postmortem?

A postmortem (sometimes called an incident review or post-incident report) is a structured written record of what went wrong, why it went wrong, and — most importantly — what you're going to do to make sure it never happens again.

Google's SRE team literally wrote the book on this. Their approach, outlined in the SRE Workbook, describes a blameless postmortem culture as one of the most powerful tools an engineering organisation can have. The goal isn't to find someone to blame. It's to understand the system well enough to make it more resilient.

> *"A truly blameless postmortem culture results in more reliable systems."*
> — Google SRE Workbook, Chapter 10

## It's Not Just Documentation. It's a Mindset.

Anyone can deploy infrastructure. Anyone can follow a runbook and fix an SSL cert renewal issue at 2am. What not everyone does is sit down the next morning and ask: "Why did this happen? Could I have seen it coming? What does this tell me about my system?"

That's the postmortem mindset — and it's a genuine differentiator. Here's why:

- **It shows you think beyond the fix.** You're not just patching the symptom; you're trying to understand the disease.
- **It demonstrates technical ownership.** You're not just a cog in the machine — you understand how the whole thing fits together.
- **It signals leadership potential.** Writing something that helps your entire team avoid the same mistake? That's a leadership contribution, full stop.
- **It creates institutional memory.** Future-you (or your successor) will be grateful.

I've worked in infrastructure long enough to know that the engineers who write these things — and write them well — are the ones who get trusted with the bigger problems.

## Being Blameless: The One Rule That Changes Everything

This is the bit most people get wrong. Postmortems are not about naming who broke prod. They're about understanding how a complex system allowed something to go wrong.

Even if a human made a mistake, the real question is: why was it possible for that mistake to cause this level of impact? Could a better alert have caught it earlier? Could a safeguard have stopped the blast radius? Was the documentation clear enough?

Blame kills learning. Blamelessness creates safety. And psychological safety is what makes people flag near-misses before they become outages.

## A Real-World Example

Let me paint a picture. Your team's web server stops serving HTTPS traffic at 07:30 on a Monday morning. After some frantic digging, you find the SSL certificate expired over the weekend. The renewal cron job failed silently because it was running as a user that had lost write access to the cert directory — a change that was made during a routine SELinux policy update three weeks ago.

You fix it in 45 minutes. Services restored. But did you just move on?

A good engineer writes the postmortem. They document the timeline, the root cause (SELinux context change broke file permissions for the renewal process), and — this is the key part — they add action items: a monitoring alert on certificate expiry dates, a test in the CI pipeline that validates the renewal process works end-to-end, and a review of other cron jobs that might be affected by the same SELinux change.

Now that 45-minute fix becomes a permanent improvement to your infrastructure. And the engineer who wrote that up? They just showed everyone they understand their system at a level beyond "fix and move on."

## The Postmortem Template

Here's a clean, practical template based on Google's SRE recommendations. Adapt it to your team's needs — the format matters less than the habit.

| Field | Description |
|---|---|
| Title | Short, descriptive summary of the incident |
| Date | When did the incident occur? |
| Authors | Who wrote this postmortem? |
| Status | Draft / In Review / Final |
| Severity | e.g. SEV1 (critical) to SEV3 (low impact) |
| Duration | Start time → End time |

### 1. Executive Summary

Two or three sentences. What happened, what was the impact, and how was it resolved? Think: if a VP reads only this section, do they understand the situation?

### 2. Impact

- Which systems / services were affected?
- How many users or requests were impacted?
- What was the duration of the degradation?
- Any SLA/SLO breaches?

### 3. Timeline

A chronological log of events. Keep it factual — no editorialising. Times are key.

| Time (UTC) | Event |
|---|---|
| 07:30 | First alert triggered: HTTPS requests returning 502 |
| 07:34 | On-call engineer paged and begins investigation |
| 07:52 | Root cause identified: expired SSL certificate |
| 08:15 | Certificate renewed manually, services restored |

### 4. Root Cause

Go deep here. The "5 Whys" technique works well: keep asking why until you reach something systemic, not human error.

> ❌ "The cert expired because nobody renewed it."
>
> ✅ "The cert expired because the automated renewal cron job failed silently after a SELinux policy change removed write access to the cert directory. There were no alerts for failed renewals or upcoming certificate expiry."

### 5. What Went Well

Always include this. It reinforces good practices and builds team morale.

- Monitoring detected the issue within minutes
- Incident comms were clear and stakeholders were informed promptly

### 6. What Could Have Gone Better

- No alerting on certificate expiry dates
- No automated test to verify the renewal process works
- SELinux policy change wasn't cross-referenced with dependent processes

### 7. Action Items

This is the most important section. Each item should be specific, assigned, and tracked.

| Action | Type | Priority | Owner |
|---|---|---|---|
| Add Prometheus alert for SSL certs expiring within 14 days | Detect | P1 | Mickael |
| Add CI test: dry-run cert renewal on deploy | Prevent | P1 | Team |
| Audit other cron jobs affected by SELinux context changes | Prevent | P2 | Mickael |
| Update runbook with SELinux troubleshooting steps | Mitigate | P3 | Team |

### 8. Lessons Learned

What does this incident teach you about your system? What assumptions were wrong? What architectural or process improvements could prevent a whole class of similar issues?

## Final Thoughts

If you're reading this thinking "I fix things and move on, I don't have time for this" — I get it. But I'd challenge you to try it just once after your next incident. A proper postmortem doesn't have to be 10 pages. Even a tight one-pager that captures the root cause and three action items is enormously valuable.

The engineers I respect most aren't the ones who never break things. They're the ones who, when things break, learn from it — and make sure the whole team learns too. That's the difference between someone who's good at their job, and someone who makes the entire system better.

That's the postmortem mindset. And it will make you stand out.

---

## Further Reading

[Google SRE Workbook – Chapter 10: Postmortem Culture](https://sre.google/workbook/postmortem-culture/)