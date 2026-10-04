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

## Why unit tests don't work for agents

A unit test rests on three assumptions, and an agent breaks all of them.

The first is that the same input produces the same output. A model samples its output, so the same prompt against the same cluster can open with `kubectl describe` on one run and `kubectl logs` on the next. One green run tells you the agent can do the task, not how often.

The second is that the thing under test returns a value you can inspect. An agent's result is twenty tool calls that changed a cluster, followed by a paragraph claiming success, and only the first of those is evidence. To test it you need something real for it to change, and a way to put that thing back before the next run.

The third is that there is one right answer to assert on. Give three competent engineers a crash-looping pod and they will fix it three ways, all acceptable. Agents are the same. An assertion on the exact sequence of tool calls fails most of the correct runs.

Unit tests still have a job. The code behind each tool is ordinary code and should have ordinary tests. What they cannot reach is the behaviour of the loop that decides which tool to call and when to stop.

