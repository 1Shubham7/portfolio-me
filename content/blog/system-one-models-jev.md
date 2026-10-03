---
title: "System One Models: Jev, the Open-Source Alternatives, and Where They Fit"
description: "Most LLM calls in a backend are structured decisions dressed up as chat. TypeSafe's Jev answers only that kind of question: typed answers with probabilities, in one pass, with no text generation. What it is, how its confidence numbers work, what 'can't hallucinate' does and does not mean, the open-source alternatives that appeared within weeks, and where an SRE would put one."
dateString: October 2026
draft: false
tags: ["AI", "LLM", "Jev", "System One", "SRE", "RAG"]
weight: 1
---

Most of your LLM calls are really decisions.

Look at what the prompts in a backend service ask. Is this ticket billing or technical? Is this email spam? Is this alert a duplicate of the incident that is already open? Should the agent be allowed to run this command? Each has a small set of answers that you knew before you asked. We send the question to a model built to write essays, ask it to reply in JSON, wait while it produces the answer one token at a time, parse the result, and retry when the parse fails. Then we take the string `"billing"` and feed it to an `if` statement.

That is a structured decision dressed up as chat. On 15 September 2026 a company called TypeSafe AI released a model named Jev that does only the decision. It cannot chat, write or code. You give it some state and a list of typed questions, and it returns typed answers with probabilities, in one pass, for $0.042 per million input tokens. TypeSafe calls the category System One models.

Here is how it works, what "can't hallucinate" is worth, and which of the open alternatives I would try first. One caveat before any of it: the category is under three weeks old as I write this, and none of the numbers below are mine. They come from TypeSafe's docs, from the projects' READMEs, and from whoever those READMEs are quoting, and I say whose number it is every time.
