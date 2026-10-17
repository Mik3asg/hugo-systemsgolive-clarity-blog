---
title: "How to restore the consultant positioning"
draft: true
build:
  list: never
  render: never
---

This folder keeps the site's previous "independent consultant" positioning so it can be
reactivated later. Nothing in `content/_archive/` (or `content/consulting-original.md`)
is built or published: every file has `draft: true` and `build: { list: never, render: never }`.

## What was archived

- **Git:** branch `archive/consultant-positioning` and tag `consultant-positioning-2026-10`
  both point at commit `39c0847`, the last commit with the consultant positioning live.
- **Consulting page:** `content/consulting-original.md` (front matter) plus
  `layouts/_default/consulting.html` (the page body and styles, left in place).
- **Homepage copy:** `content/_archive/homepage-consultant.md` holds the old hero tagline,
  intro paragraphs, availability text, CTA, sidebar blurb, meta description and nav item, verbatim.

## Option 1: restore everything from the tag

```bash
git checkout consultant-positioning-2026-10 -- \
  layouts/partials/about-content.html \
  config/_default/params.toml \
  config/_default/menus/menu.en.toml \
  content/_index.md \
  content/consulting.md
git rm content/services.md   # drop the Services page and its /consulting/ redirect
```

Check `git diff --staged`, run `hugo`, and commit.

To view the old site without changing `main`: `git switch archive/consultant-positioning`.

## Option 2: copy the archived text back by hand

1. Paste the strings from `homepage-consultant.md` into the files named in each section.
2. Re-enable the consulting page: in `content/consulting-original.md`, remove `draft = true`
   and the `[build]` block, then rename it to `content/consulting.md`.
3. Delete `content/services.md` (it holds the `/consulting/` alias, which would clash),
   and set the nav item in `config/_default/menus/menu.en.toml` back to
   `name = "Consulting"`, `url = "consulting/"`.
