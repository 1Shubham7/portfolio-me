---
title: "Loop Engineering: How to Create Loops in Claude Code"
description: "What loop engineering actually means, where the term came from, and how /loop and /goal turn Claude Code into something that babysits deployments and works until the tests pass."
dateString: August 2026
draft: false
tags: ["Claude Code", "AI", "Agents", "DevOps"]
weight: 1
cover:
    image: "/blog/claude-loops/cover.png"
---

```
/loop 5m check if the deployment finished and tell me what happened
```

That one line turns Claude Code from a thing you prompt into a thing that keeps checking so you don't have to. This post is about that: what "loop engineering" means, where the term came from, and how `/loop` and `/goal` actually work.

## Two loops, not one

When people say "loop" around agents, they mean one of two things, and it helps to keep them apart.

The **inner loop** is the agentic loop itself: the model calls a tool, reads the result, decides the next step, and repeats until the task is done. This is what an agent *is*. Anthropic's [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) post from late 2024 essentially defined agents this way: an LLM using tools in a loop, directing its own process based on feedback from the environment. Every Claude Code session is already running this loop. You don't have to build it.

The **outer loop** is re-invoking the agent on a schedule or a condition. Check the deploy every five minutes. Look at the PR again when CI finishes. Run the maintenance pass every night. The inner loop finishes a task; the outer loop decides when to start the next one.

Loop engineering is designing that outer loop: what triggers each iteration, what state carries between them, and when the whole thing should stop.

## Where the term comes from

The lineage is more traceable than most AI jargon. First everyone did **prompt engineering**. Then in 2025 the focus shifted to **context engineering**, a term Tobi Lütke pushed as the better name for the actual skill: deciding what fills the context window, not how the question is phrased. **Loop engineering** is the next step up the stack. As far as I can verify, Addy Osmani coined it in June 2026, reacting to a line from Boris Cherny, who leads Claude Code at Anthropic: "I don't prompt Claude anymore. I write loops that prompt Claude."

The practice predates the name, though. Geoffrey Huntley's "Ralph" technique, named after Ralph Wiggum, is a bash `while true` that keeps re-launching a coding agent against a prompt file until the work is done. Each iteration starts a fresh process with a clean context; progress lives in files, not in the model's memory. It's crude and it works, well enough that Anthropic now ships a [ralph-wiggum plugin](https://github.com/anthropics/claude-code/blob/main/plugins/ralph-wiggum/README.md) in the Claude Code repo. Ralph is what made outer-loop agent running a normal thing to do.

## /loop, the practical part

Claude Code bundles this as the [`/loop` skill](https://code.claude.com/docs/en/scheduled-tasks). Three ways to call it:

```
/loop 5m check if the deployment finished
/loop check whether CI passed and address any review comments
/loop
```

**With an interval**, the prompt runs on a fixed schedule. Units are `s`/`m`/`h`/`d`, and `every 2 hours` works too. Cron granularity is one minute, so seconds round up, and odd intervals like `7m` get rounded to a clean step.

**Without an interval**, Claude paces itself. After each iteration it picks the next delay, between one minute and one hour, based on what it just saw: short waits while a build is moving, long waits once the PR goes quiet. In practice this burns fewer tokens than fixed polling, because it doesn't wake up every five minutes to report that nothing changed.

**Bare `/loop`** runs a built-in maintenance prompt: finish pending work, tend the current branch's PR (review comments, red CI, merge conflicts), then look for cleanup. Drop a `loop.md` in `.claude/` to replace that prompt with your own.

You can also loop a slash command, which is where this gets genuinely useful:

```
/loop 20m /review-pr 1234
```

And one-shot reminders don't need `/loop` at all. "Remind me at 3pm to push the release branch" just works.

Under the hood, a fixed interval becomes a 5-field cron expression, and a scheduler enqueues due tasks between your turns, never mid-response. Self-paced mode uses an internal `ScheduleWakeup` tool where Claude picks its own next delay. Recurring tasks get deterministic jitter (up to 30 minutes late, or half the interval for sub-hourly jobs) so a thousand sessions don't all fire at :00.

Stopping: `Esc` kills a self-paced loop while it waits, and Claude can end one itself when the job is done. For fixed loops, ask "what scheduled tasks do I have?" then "cancel the deploy check job".

## /goal, the other kind of loop

`/loop` re-runs on a clock. [`/goal`](https://code.claude.com/docs/en/goal) re-runs on a condition:

```
/goal all tests in test/auth pass and lint is clean
```

That sets a completion condition for the session, and Claude keeps taking turns until it holds. Under the hood it is a Stop hook: every time Claude tries to end its turn, the condition and the conversation so far go to a small fast model (Haiku by default), which rules met, not yet, or impossible. "Not yet" comes back with a reason, and that reason becomes the steer for the next turn. This is the Ralph pattern built in, no bash `while true` required.

Two things matter when writing the condition. It has to be provable from the transcript: "all tests pass" works because Claude runs the tests and the output is right there in the conversation, while "the server is running" is weak because the evaluator can't check anything itself. And there is no built-in timeout, so a condition that can never be met loops forever. Bound it yourself:

```
/goal migrate the auth module or stop after 20 turns
```

`/goal` on its own shows status, `/goal clear` stops it early. Full disclosure: this post was written under one.

## Which loop when

`/loop` when you're watching something on a clock: a deploy, a PR, a long build. `/goal` when there's a definition of done and you want Claude to keep going until it holds. Both are session-scoped, so they stop when the terminal closes. When a loop should outlive your laptop, `/schedule` creates a cloud routine that runs 24/7 with no session attached, at a minimum interval of one hour. That's the whole decision tree: `/loop` for the clock, `/goal` for the condition, `/schedule` for the standing job.
