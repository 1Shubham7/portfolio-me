---
title: "Agent Evals: How to Know a Change Made Your Agent Worse"
description: "Normal code has tests; agents need evals. What an eval harness is, the vocabulary that goes with it, how I would build one in Go for an SRE agent working against Kubernetes, why pass^k matters more than pass@k for ops, the tools that exist (and the one OpenAI is shutting down), the ways a harness's numbers lie to you, and the habits that keep a suite useful."
dateString: October 2026
draft: false
tags: ["AI", "Agents", "Evals", "SRE", "Kubernetes", "Go"]
weight: 1
---

Say you tidy up a tool description. The `restart_deployment` tool had four sentences explaining when to use it, and you cut them to one. No code changed. The unit tests pass, and the PR is a few lines of deleted prose. Meanwhile the agent that used to read a pod's events before touching anything now sometimes restarts first and looks afterwards.

Nothing in an ordinary CI pipeline will tell you that. Normal code has tests; agents need evals. An eval harness is how you know a change made your agent worse.

The rest of this post designs one for an SRE agent working against Kubernetes. Designs is the right word: the task, the Go and the output format were written for this post, there are no scores of mine anywhere in it, and anything taken from published work links to where it came from.

