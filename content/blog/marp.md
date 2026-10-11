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

