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

## What an eval harness is

An eval harness runs the agent on tasks in resettable environments, records everything it does, grades the result automatically, and tracks the scores over time. Anthropic's [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents), published in January 2026, has the definition I would hand to someone new: "An evaluation harness is the infrastructure that runs evals end-to-end. It provides instructions and tools, runs tasks concurrently, records all the steps, grades outputs, and aggregates results."

The closest things an SRE already has are CI and a chaos-engineering game day, and an eval harness is the two put together. From CI it takes the trigger and the gate: it runs unattended on every change that matters, and it can block a merge. From the game day it takes the scenario: you break an environment on purpose and watch how the responder handles it. Here the responder is the agent, and the debrief is done by code.

One naming trap. I wrote a whole post on the [agent harness](/blog/agent-harness/): the loop, tools, permissions and context management around the model. That is the thing under test here. The eval harness is a separate program that starts the agent harness, points it at a broken environment and scores what it does. When someone says "the harness" with no qualifier, ask which one they mean.

## The vocabulary

Five terms carry most of the conversation. They come from the same Anthropic post and I use them its way from here on. Restated for an agent that works on a cluster:

- **Task**: one test, with fixed inputs and a definition of success. Here that is one broken environment and one definition of fixed.
- **Trial**: one attempt at a task. In practice, one fresh namespace and one run of the agent inside it. You run several per task, because a single attempt by something that does not repeat itself tells you very little.
- **Transcript**: the complete record of a trial, every model message, tool call and tool output. The post notes that people also say trace or trajectory, and they mean the same thing.
- **Grader**: the logic that scores one aspect of a trial. "Is the pod Ready" is a grader. So is "did it delete anything", and a task normally carries several.
- **Eval suite**: a collection of tasks that measure one capability or behaviour. On disk, a directory of task files.

The glossary also defines the **outcome**: the state of the environment when the trial ends. Keep it apart from the transcript, which only holds what the agent did and said. An agent can write "the deployment is healthy" in its last message while the pod is still in back-off.

## What a harness is made of

None of the parts is exotic, and most of the work is in the second one, the sandbox.

The task suite comes first. Each task is a setup, a prompt and success criteria. The setup puts the environment into its broken state. The prompt is what the agent is told, and it should read like something an alert or a colleague would send. The success criteria say what must be true at the end. Tasks live in the repo as files and get reviewed like code.

Every trial gets its own sandbox, fresh and isolated, which Anthropic's post treats as a requirement. Leftover state from one trial changes the next, and then failures are correlated for reasons that have nothing to do with the agent. For Kubernetes, a namespace per trial in a shared cluster is quick and cheap, but the trials share nodes and every cluster-scoped object. A throwaway cluster per trial, with something like kind, is clean and slower. Whichever you pick, the agent in each trial runs as a ServiceAccount created for that trial, with credentials scoped to the sandbox and nothing else.

The agent runner starts the real agent in that sandbox: the same system prompt, tool definitions and guardrails that ship. It enforces step, time and token budgets. A trial that hits a budget is a failure, and it is recorded as its own kind of failure, because "looped until the step limit" and "confidently did the wrong thing" need different fixes.

While the agent runs, the transcript recorder writes every model message, tool call, argument, tool output, timestamp and token count to disk as it happens, so that a crash still leaves a record. A Kubernetes agent leaves a second record that it does not get to write: the API server's [audit log](/blog/audit-logging/), which can hold an event for every request the agent's ServiceAccount made, including the ones RBAC refused. The harness keeps that next to the transcript, because it is the better evidence of what the agent tried to do to the cluster.

Graders come in three kinds in Anthropic's post: code-based, model-based and human. A code-based grader checks something like cluster state. The post's comparison table lists these as fast, cheap, objective and reproducible, and gives their weakness as being brittle to valid variations. A model-based grader, the LLM-as-judge, handles what code cannot express, such as whether the agent's incident summary names the real root cause. It is flexible, non-deterministic, and costs money on every call. Human review is expensive and slow, and inside a harness its main job is checking the other two. I would start with code and add a judge only where a criterion cannot be written as a check.

Last is the scorecard: pass rates, cost and step counts per task and per suite, printed next to the same numbers from a baseline, normally the last run on main, since the diff is what gets acted on.

