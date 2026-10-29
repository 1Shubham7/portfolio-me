---
title: "Your LLM Writes Too Much: How ASD-STE100 Makes It Speak a Controlled Language"
description: "ASD-STE100 is a forty-year-old aerospace standard for writing maintenance manuals that nobody can misread: one meaning per word, twenty words per sentence, one instruction per sentence. The models already know it. Naming it in a prompt is the cheapest editing pass there is, and asking for 80% of it is the setting that actually works."
dateString: October 2026
draft: false
tags: ["LLM", "Writing", "Technical Writing", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/asd-ste100/asd-ste100.png"
---

Ask a model what a liveness probe is and you get a paragraph. Somewhere in the middle of it is the answer, wrapped in "essentially", "in order to", "it's worth noting", and a closing sentence that restates the opening one. I had got used to reading past that. Everyone who uses these tools daily has, the same way you stop seeing the ads on a page.

Then at the start of this month Andrej Karpathy [posted on X](https://x.com/karpathy/status/2105819303471976479) that he asks his LLM to explain things in something called ASD-STE100, and that the output is a lot more readable. I had never heard of it. The name looked like a part number. So I went and read about it, and then started putting it in prompts. This post is what I found: what the thing is, where it came from, why a model responds to it, and a set of side-by-side examples from the kind of text I actually read at work. Karpathy's post goes on to diagrams, HTML pages and videos as richer formats; I am ignoring all of that and staying on the writing.

## What is it?

ASD-STE100 is Simplified Technical English, a controlled language. A controlled language is a subset of a natural language with fewer words and stricter grammar, built so that a reader who is not fluent, or who is tired, or who is halfway up a wing, cannot read a sentence two ways.

The standard has two parts.
