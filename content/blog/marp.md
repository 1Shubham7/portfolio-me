---
title: "Marp: Slides That Live in Git Next to the Code They Talk About"
description: "What Marp is, where it came from, how Marpit turns a Markdown file into a deck, rendering to PDF and PPTX with the CLI, and how to keep a theme shared across every deck instead of copy-pasting CSS."
dateString: October 2026
draft: false
tags: ["Markdown", "Presentations", "DevOps", "Tooling"]
weight: 3
cover:
    image: "/blog/marp/marp.png"
---

Marp is a converter that turns a Markdown file into a slide deck. A `---` starts a new slide, a few YAML lines at the top pick a theme, and the output is HTML, PDF, or PowerPoint. That's the whole pitch, and it's enough for my slides to live in git next to the runbooks and the code they talk about.

## Why slides in Markdown at all

A deck in Keynote or Google Slides is a binary blob that nobody can diff. For a talk you give once that's fine. For the decks I keep coming back to, an onboarding walkthrough, an incident review template, the same architecture overview updated every quarter, it isn't.

A Marp deck is one Markdown file where every `---` is a slide boundary, so a pull request diff reads as "slide seven changed" and `git blame` on a bullet tells you who added it and when. Code on a slide is a fenced block rather than a screenshot, so it's the snippet from the repo and it goes through the same Markdown linting as the README. You write it in whatever editor you already write everything else in.

The usual objection is that you give up layout control. You give up some. What you get back is a theme that's plain CSS, which turns out to be a better deal than it sounds, and most of this post is about that trade.

## Where Marp came from

The original Marp was an Electron desktop app by Yuki Hattori, published from the `yhatt/marp` repo with the first tags landing in May 2016: a live-preview editor with PDF export. Maintenance stopped in 2017, and the repo was archived in September 2019 with a blunt closing note: a security report had shown that opening a malicious Markdown file could execute arbitrary code, so stop using it and move to "Marp Next".

Marp Next is what everyone means by Marp today, and it's a rewrite into layers rather than a single app:

- **Marpit** is the base framework, with a first changelog entry in March 2018. It's deliberately small: Markdown plus a CSS theme in, HTML and CSS out, with a plugin system and nothing else. It doesn't even ship a theme.
- **Marp Core** sits on Marpit and adds the practical bits: three built-in themes, emoji, math, auto-scaling, and safer HTML handling. v1.0.0 came in January 2020. Every official Marp tool uses Marp Core as its engine.
- **Marp CLI** is the converter, with tags going back to August 2018.
- **Marp for VS Code** started in February 2019. It's a preview and export extension, and it's what replaced the desktop app for most people.

The split matters because it explains a lot of behaviour. When a feature says "only works in inline SVG mode" or "the app must support this attribute", that's Marpit talking: it produces markup and leaves presentation behaviour to whatever consumes it.

## A complete deck in one file

Here's a small but complete deck.

```markdown
---
marp: true
theme: default
paginate: true
header: "Platform team"
footer: "Internal, do not distribute"
---

<!-- _paginate: false -->
<!-- _class: invert -->

# On-call handover

What changed this week, and what's still on fire

---

## Things that paged

- Disk pressure on the metrics nodes
- Cert renewal failed for the staging ingress

---

<!-- _backgroundColor: #fff8e1 -->

## Still open

* The disk alert has no runbook yet
* Nobody owns the staging cert

---

![bg right:40%](./img/dashboard.png)

## Next week

Add the runbook, assign the cert, close this out.
```

The front-matter is YAML. `marp: true` tells the VS Code extension to treat this file as a deck instead of a normal Markdown document; the CLI converts whatever you hand it and doesn't need it. `---` on its own line starts a new slide. Everything else is a directive or an image with a special alt text.

