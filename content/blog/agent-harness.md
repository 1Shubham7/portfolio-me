---
title: "Agent Harness: Everything That Isn't the Model"
description: "What an agent harness actually is, where the word comes from, and the dozen or so primitives hiding inside the term. A companion to the loop engineering post."
dateString: August 2026
draft: false
tags: ["Claude Code", "AI", "Agents", "DevOps"]
weight: 1
cover:
    image: "/blog/agent-harness/cover.png"
---

When you hit enter in Claude Code, the model doesn't run anything. It emits a request: run this command, edit this file. Something else executes it, captures the output, decides how much of it the model gets to see, checks whether it's allowed at all, and starts the next turn. That something else is the agent harness, and it's most of what you're actually using when you use an agent.

"Harness" has become one of those words everyone in the agent space uses and almost nobody defines. This post is the definition I wish someone had handed me: what the term means, where it came from, and what's actually inside it when you take it apart.

## Everything that isn't the model

The cleanest definition I've found is LangChain's, from Vivek Trivedy's [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness): a harness is "every piece of code, configuration, and execution logic that isn't the model itself." He follows it with the line that makes the split memorable: "The model contains the intelligence and the harness is the system that makes that intelligence useful."

Anthropic's version, from Lance Martin's [Agent Harness Design](https://claude.com/blog/harnessing-claudes-intelligence) post, lists the parts: "An agent harness is the software scaffolding around a model: the loop, tools, context management, and guardrails that turn raw intelligence into a working agent."

And if you want the precise version, a [May 2026 arXiv paper](https://arxiv.org/html/2605.23950v1) arguing that benchmark results are meaningless without harness disclosure defines it as "the software layer between the model and the task that constructs the context the model sees, mediates its tool calls, validates its outputs, and decides when to retry, escalate, or stop."

Three sources, one shape. The model proposes; the harness is everything that turns proposals into effects. [Simon Willison's definition of an agent](https://simonwillison.net/2025/Sep/18/agents/), "an LLM agent runs tools in a loop to achieve a goal", is really a definition of a model plus a harness, because the model on its own can't run anything.

Claude Code is the concrete example most people have touched. Anthropic describes the Claude Agent SDK as [the agent harness that powers Claude Code](https://claude.com/blog/building-agents-with-the-claude-agent-sdk). The chat interface is the thin part. The harness underneath is the thick part, and it's what the rest of this post is about.

## Where the word comes from

The lineage is older than agents, and it's worth tracing because the metaphor carried over intact.

In classic software engineering, a [test harness](https://en.wikipedia.org/wiki/Test_harness) is the stubs and drivers you build around a component so you can exercise it without its real environment. The harness holds the thing in place and drives it. That's the picture: not decoration, but a rig that makes something otherwise inert do useful work under controlled conditions.

LLM tooling borrowed the word early (EleutherAI's eval framework has carried the name [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness) since 2020), so "harness" was already floating around as "the rig you put a model into" before agents existed.

When it attached itself to agents specifically is fuzzier, and I'll be honest about that rather than manufacture an origin story. As far as I can verify, nobody coined "agent harness". It drifted in during 2025. Anthropic's [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) from December 2024, the post that defined the agent-as-loop framing, doesn't contain the word at all. By September 2025 Simon Willison was writing "its harness" as a term the reader is assumed to know, and at the end of that month Anthropic's Agent SDK launch used "agent harness" in the announcement itself. By November 2025 there was a whole Anthropic engineering post on [effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents). Somewhere between those two Decembers, it went from not-a-word to assumed vocabulary.

Then the word grew a discipline. In February 2026 Mitchell Hashimoto described his workflow as [harness engineering](https://mitchellh.com/writing/my-ai-adoption-journey): "anytime you find an agent makes a mistake, you take the time to engineer a solution such that the agent never makes that mistake again." OpenAI published a piece titled "Harness engineering: leveraging Codex in an agent-first world" six days later, and Birgitta Böckeler's [martinfowler.com memo](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering-memo.html) credits Hashimoto while allowing that OpenAI may have landed on the phrase independently. By April, Addy Osmani's [Agent Harness Engineering](https://addyosmani.com/blog/agent-harness-engineering/) post was passing around a slogan he attributes to Viv Trivedy: "Agent = Model + Harness. If you're not the model, you're the harness."

## The parts list

"Everything that isn't the model" is honest but not actionable. Here is what that everything decomposes into. This is the consensus list, the primitives that Anthropic's SDK docs, LangChain, Osmani, and [codecentric](https://www.codecentric.de/en/knowledge-hub/blog/loop-harness-context-engineering-explained) all name independently.

- **The agent loop.** Gather context, take action, verify, repeat. This is the skeleton everything else hangs off, and it's what my [previous post](/blog/claude-code-loops) was about: loop engineering sits on top of harness engineering, because the loop is one primitive among many.
- **The system prompt.** The standing instructions that shape every turn. In a product like Claude Code this is thousands of words you never see.
- **Tool definitions and execution.** Not just describing tools to the model, but actually running them, handling their failures, and deciding how much of a 40,000-line command output gets fed back.
- **Context management.** The window fills up. Compaction summarizes the conversation so far and continues in a fresh window, and how well that works determines whether long tasks survive.
- **Permissions and sandboxing.** Which tools run automatically, which need approval, what the blast radius is when the model does something dumb. This is the guardrail layer.
- **Subagents.** Spawning focused workers for subtasks, which buys parallelism and, more importantly, context isolation: the subagent's 200 tool calls don't pollute the parent's window.
- **Hooks.** Deterministic code that runs at lifecycle points, no model judgement involved. A formatter after every edit, a validator before every commit.
- **MCP servers.** The standardized plug for external systems, so the harness doesn't need bespoke code for every database and ticket tracker.
- **Skills.** Capabilities disclosed progressively, so the model discovers a detailed procedure when relevant instead of carrying every instruction in every prompt.
- **Memory and sessions.** State that persists across invocations, because the model itself remembers nothing between calls.
- **Observability.** Logs, traces, token metering. Boring until an agent misbehaves in production and you need to know what it saw.

Three more ideas are worth naming even though fewer sources push them. Anthropic's long-running-agents post recommends an initializer agent that sets up progress artifacts (init scripts, progress files, frequent git commits) so work survives context windows and crashes; the environment becomes the memory. Hashimoto treats AGENTS.md as harness: every entry exists because an agent once made a specific mistake, and the entry fixes it permanently. And codecentric splits the concept usefully into a Tool Harness (what the product ships) and a User Harness (your repo's rules, tests, and context docs), with the tidy conclusion that "it's not the strongest model that wins, it's the best-equipped one." I like that split, because it makes clear that your CLAUDE.md and your test suite are part of the harness whether you think of them that way or not.

## Does it actually matter that much

More than feels reasonable. Swap the harness under a model and you get a different agent: different context, different tools, different failure handling, different results. There's [published research](https://arxiv.org/html/2605.23950v1) showing the same model swinging by double digits on coding tasks depending on nothing but the harness it ran under, which is why that paper's title is a demand: stop comparing agents without disclosing the harness.

The more useful way to hold this, day to day, is that the harness is the part you control. You can't retrain the model. You can add a hook, tighten a permission, split a subagent out, write the AGENTS.md entry that makes a recurring mistake impossible. Almost all practical agent work is harness work.

There's a tension in that, and Lance Martin names it directly: harnesses "encode assumptions about what Claude can't do on its own, but those assumptions grow stale as Claude gets more capable." Every guardrail is a bet that the model still needs it. Some of those bets expire quietly, and a harness full of expired bets is just overhead.

So that's the word. A rig borrowed from test engineering, now naming the layer where most of the practical engineering around agents actually happens. The model is the part you can't change. The harness is the part you can.

Thanks for your time.
