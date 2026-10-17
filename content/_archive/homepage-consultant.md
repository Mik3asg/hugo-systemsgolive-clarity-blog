---
title: "Archived homepage: consultant positioning"
draft: true
build:
  list: never
  render: never
---

Snapshot of the homepage/About copy as it was at tag `consultant-positioning-2026-10`
(commit 39c0847), before the switch to "Senior Cloud & DevOps Engineer" positioning.
Only English existed at the time; the Portuguese language was disabled, so there is
no /pt/ copy. Strings below are verbatim.

## Hero tagline

File: `layouts/partials/about-content.html`

```html
<p class="abouttest-hero-tagline">{{ .Params.heroTagline | default "Independent Cloud & DevOps Consultant helping growing SaaS engineering teams fix cloud cost,<br>deployment and reliability problems as their infrastructure gets harder to run." | safeHTML }}</p>
```

## Intro paragraphs (About section)

File: `layouts/partials/about-content.html` (the third paragraph, the personal bio, was unchanged and is not archived here)

```html
<p class="abouttest-about-text">I'm an independent Cloud & DevOps Consultant with 14+ years of experience in the tech space, spanning Automotive, Energy, Telecom, Aerospace, and Social Care sectors. I diagnose infrastructure problems and deliver practical, production-ready improvements across cloud architecture, infrastructure automation and software delivery — I also write about this work on the <a href="/blog/">blog</a>.</p>
<p class="abouttest-about-text">My focus is on measurable outcomes: lower cloud costs, faster and safer deployments, more reliable infrastructure and less operational overhead for engineering teams. I work primarily on defined, fixed-outcome consulting projects rather than permanent placements. Currently taking on new consulting engagements — <a href="/consulting">see how I can help</a>.</p>
```

## Availability text

Contained in the second intro paragraph above:
"I work primarily on defined, fixed-outcome consulting projects rather than permanent placements. Currently taking on new consulting engagements"

## Homepage CTA

File: `layouts/partials/about-content.html`

```html
<section class="abouttest-consulting-cta">
      <h2 class="abouttest-section-title">Need help with your cloud infrastructure?</h2>
      <p class="abouttest-about-text">Book a free 15-minute intro call to see if I can help with your cloud cost, deployment or reliability challenges.</p>
      <a href="/consulting" class="button abouttest-cta-button">Book an intro call</a>
    </section>
```

## Sidebar author blurb

File: `config/_default/params.toml`

```toml
introDescription = "Independent Cloud & DevOps Consultant helping growing SaaS teams fix cloud cost, deployment and reliability problems."
```

## Homepage meta description

File: `content/_index.md`

```toml
description = "DevOps Engineer with 14+ years of experience — background, work history, and what I'm about"
```

## Navigation

File: `config/_default/menus/menu.en.toml`

```toml
[[main]]
  name = "Consulting"
  url = "consulting/"
  weight = -106
```
