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

The first is a dictionary of roughly 900 approved words. The thing that makes it different from a vocabulary list is that each approved word gets exactly one meaning and one part of speech. "Close" is approved as a verb: close a door, close a circuit breaker. The adjective "close", as in "close to the landing gear", is not approved; you write "near". "Fall" means to move down by gravity and nothing else, so pressure does not fall in STE, it decreases. "Test" is a noun only, so you do not test the pump, you do a test of the pump. Where English has five words for one idea, the dictionary keeps one: "start" is approved, and "begin", "commence" and "initiate" are listed as not approved with "start" as the replacement. On top of the dictionary you are allowed your own technical nouns and technical verbs, the names of the parts and actions in your field, which is how "kubelet" and "cordon" get into an STE sentence without being in a list written by the aerospace industry.

The second part is a set of 53 writing rules. The ones that do most of the work:

- Procedural sentences (instructions) have at most 20 words. Descriptive sentences have at most 25.
- One instruction per sentence.
- One topic per paragraph, and at most six sentences in it.
- Active voice. In descriptive text, passive is allowed only when nobody knows who the agent is.
- Only simple verb forms: infinitive, imperative, simple present, simple past, simple future. No "is being restarted", no "has been configured".
- No "-ing" forms except inside technical nouns. "Logging" the noun is fine; "the kubelet is restarting the container" is not.
- Noun clusters of at most three words. "Kubernetes node drain runbook step" is five nouns in a row and has to be broken up.
- Do not leave out words to make a sentence shorter. Articles and verbs stay.

Hold those in your head while reading an average LLM answer and you can see every rule it breaks in the first sentence.

## Where did it come from?

The original users were aircraft mechanics. In the early 1980s the European airlines asked AECMA, the European aerospace industry association, to look at the English in maintenance manuals, because a large share of the people maintaining their fleets did not speak English as a first language and a misread step on an aircraft is not a documentation problem. In 1983 AECMA decided to write its own controlled English instead of adopting an existing one, and the first AECMA Simplified English guide came out in the middle of that decade. The ATA specification for maintenance manuals has required it since 1986, so if you have ever seen a page of an aircraft maintenance manual, you have seen STE.

AECMA merged into ASD (the Aerospace, Security and Defence Industries Association of Europe) in 2004, and the guide was renamed ASD-STE100 and turned into a specification. It was a paid document for a long time. Since Issue 6 in 2013 it has been free. The current issue is Issue 9, released on 15 January 2025, and with it ASD changed the subtitle from "specification" to "standard" and now describes it as an international standard for technical documentation. That is ASD's own label; it is not an ISO standard, and I would not call it one. Issue 9 added no new rules. It reworded 31 of the 53 and updated 555 dictionary entries, which tells you how settled the thing is after forty years.

Getting a copy is a form on [asd-ste100.org](https://asd-ste100.org/) that asks who you are and what you will use it for, and the PDF follows. ASD's distribution log for the previous issue showed that most of the people requesting it were outside aerospace and defence.

## Why does naming it in a prompt work?

Karpathy's reasoning, from the post:

> Writing. Something I've had success with: Ask your LLM to explain something in ASD-STE100, it's a controlled language specification originally developed for aerospace maintenance documentation. LLMs well-versed in this language and it comes with heavy constraints on clean writing style that I often find a lot more readable. Sometimes I've tried to soften it a bit e.g. ask for "80% of the way to ASD-STE100" because the spec is quite stringent.

Two things make this work and neither is magic.

The first is that the models have read the standard, the forum threads about the standard, and decades of manuals written in it. "ASD-STE100" is a name that points at a large, consistent body of text with a very recognisable shape. Saying it is the same kind of instruction as "write this as a man page": you are not explaining a style, you are naming one the model already has.
