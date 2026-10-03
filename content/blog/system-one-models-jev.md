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

## System 1 and System 2

The name is borrowed from Daniel Kahneman's *Thinking, Fast and Slow*. System 1 is the fast, intuitive mode: you recognise a face, you flinch, you know a sentence is angry before you have finished reading it. System 2 is the slow, deliberate one that you use to do long division or fill in a tax form. TypeSafe's docs make the reference themselves, and spell it "System One" where Kahneman writes "System 1". It is the same idea.

Mapped onto models, a chat LLM is the System 2 tool. It reasons in steps, writes, explains itself, and takes seconds to do it. A System One model makes quick judgment calls and does nothing else. TypeSafe's definition is "a class of AI models built to make fast, structured decisions that software can use directly", and the docs are just as direct about the other side: System One models "do not write replies, produce code, or generate explanations of their reasoning."

The psychology is a loose analogy and I would not lean on it. The useful test is plainer: does the answer fit in an enum, a number on a scale, or a yes/no? If it does, you are asking a System One question, whichever model you send it to.

## What Jev is, and how you call it

Jev is TypeSafe AI's first model. It went into early access on 15 September 2026, the same day [DCVC announced](https://www.dcvc.com/news-insights/typesafe-emerges-from-stealth-with-a-new-way-of-doing-ai/) it had led TypeSafe's $40 million seed round. The founder and CEO is Diogo Almeida, who came from OpenAI. The company's team page credits him with co-inventing RLHF and InstructGPT, the methods that led to ChatGPT. The name comes from William Stanley Jevons, of the Jevons paradox: more efficient steam engines made coal cheaper to use, and coal consumption went up. TypeSafe's [launch post](https://typesafe.ai/blog/introducing-system-one-models-and-jev) expects machine intelligence "to follow a similar path to coal".

Two things make it different from a small LLM with a JSON schema bolted on. It has no text output at all, so there is nothing to parse and nothing that can come back malformed. And it is not autoregressive: in the launch post's words, "Jev outputs all probabilities in parallel instead of autoregressively generating by token." Ten questions come back in the same pass as one. TypeSafe trains it with a method it calls Reinforcement Learning for Calibrated Decisions (RLCD), and the calibration part matters more than the name. I come back to it in the section on "can't hallucinate".

### State in, typed answers out

A request is a `state` and a set of `questions`. The state is whatever the decision is about: a string, a list of strings, or a JSON object. Each question has one of three types:

- **Choice** picks one option from a list you define, and returns a probability for every option.
- **Score** places the state on a scale whose levels you describe in words.
- **Noul** (TypeSafe's spelling, and not a typo) returns one number: the probability that a yes/no statement is true.

This is the example from TypeSafe's [quick start](https://docs.typesafe.ai/introduction/quickstart), unchanged, using the Python SDK (`pip install typesafe-sdk`; there is a JavaScript one too, `@typesafe-ai/sdk`):

```python
from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

client = TypeSafeClient()

ticket = "Hi, I've been trying to connect my Stripe account for 3 days and the integration keeps failing. I'm losing sales. Please help ASAP."

response = client.system_one(
    state=ticket,
    questions={
        "department": Choice(
            instructions="Which team should handle this",
            criteria={
                "billing": "Payment or subscription issues",
                "technical": "Bugs or integration problems",
                "sales": "Pricing or account questions",
            },
        ),
        "frustration": Score(
            instructions="How frustrated the customer appears",
            criteria=[
                "Calm, just stating facts",
                "Frustrated but civil",
                "Very angry, strong language",
            ],
        ),
        "is_urgent": Noul(
            instructions="The message conveys urgency or time-sensitivity",
        ),
    },
)

print(response.answers["department"].choice)  # "technical"
print(response.answers["frustration"].score)  # 1.0
print(response.answers["is_urgent"].noul)     # 1.0
```

The client reads `TYPESAFE_API_KEY` from the environment and calls `jev-latest`, which at the time of writing is an alias for `jev-1.13.0`. Underneath there is one endpoint, `POST https://api.typesafe.ai/v1/systemone`, and the response the docs show for that request looks like this (I have dropped the `legend` field from the score answer):

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "department": {
      "type": "choice",
      "choice": "technical",
      "confidence": 0.78,
      "probabilities": { "technical": 0.85, "sales": 0.0, "billing": 0.15 }
    },
    "frustration": {
      "type": "score",
      "score": 1.0,
      "confidence": 1.0,
      "probabilities": { "0": 0.0, "1": 1.0, "2": 0.0 }
    },
    "is_urgent": { "type": "noul", "noul": 1.0 }
  },
  "usage": { "input_tokens": 392, "output_tokens": 65 }
}
```

Every value in `answers` is something you defined: an option key, a level index, a probability. There is no free text anywhere in it.

### Speed, price and access

TypeSafe quotes 70 to 500 ms end to end. Input costs $0.042 per million tokens and output is free. The request above used 392 input tokens, so a million calls like it cost about $16.50.

Direct API access started as a waitlist; the launch post says TypeSafe is "bringing developers off the waitlist as quickly as we can." Two gateways carry the model in the meantime. [Vercel AI Gateway](https://vercel.com/docs/ai-gateway/sdks-and-apis/typesafe) lists it as `typesafe-ai/jev` and exposes a TypeSafe-compatible base URL, so the official SDK works with a changed `baseURL` and a gateway key. [Cloudflare Workers AI](https://developers.cloudflare.com/ai/models/typesafe/jev/) lists it as `typesafe/jev`, a third-party model you call with `env.AI.run`. Both authenticate with the gateway's own credentials. There are also several community Go clients on GitHub, none of them official, and I have not vetted any of them.

## Confidence, and the thresholds you put on it

Every answer arrives with probabilities, and what you do with them is your code's job. That is the design pattern the whole model is built around: the model reports how sure it is, and a threshold you chose decides whether software acts on its own.

Look at `department` in the response above. The top option has probability 0.85, but `confidence` says 0.78. They are different numbers on purpose. For a Choice with `n` options, the [confidence docs](https://docs.typesafe.ai/confidence) define confidence as how far the top probability sits above a uniform guess:

```text
confidence = (p_max - 1/n) / (1 - 1/n)
           = (0.85 - 1/3) / (1 - 1/3)
           = 0.775
```

So 0 means the model spread its probability evenly and 1 means it put everything on one option, regardless of how many options there were. Score has its own formula that penalises probability on distant levels more than on adjacent ones. Noul returns no `confidence` field at all, only the probability `p`; if you want a comparable number, the docs suggest `|2p - 1|`.

The pattern on top of this is short. This is the branching from the example on the same page, with the comments shortened: a Choice named `action` decides whether the user wants to check a balance, approve a transfer or get support.

```python
action = response.answers["action"]

if action.confidence < 0.5:
    route_to_human(user_message)           # unsure: a human, or a full LLM
elif action.choice == "check_balance":
    show_balance(account_id)               # low stakes, a wrong screen is recoverable
elif action.choice == "approve_transfer":
    if action.confidence > 0.9:
        confirm_then_execute(account_id)   # high stakes, high confidence
    else:
        ask_user_to_confirm(account_id)    # high stakes, moderate confidence
```

In TypeSafe's version 0.5 is the floor below which you do not guess, and 0.9 is the bar for the high-stakes action. Even above it, the function is called `confirm_then_execute`. The numbers are examples. The same docs use 0.6 and 0.85 in the confidence-gated routing pattern and 0.35, 0.70 and 0.85 in the guardrails cookbook, and the confidence page says outright that "the correct threshold values depend on your domain and the performance of the model for your use case."

Two things follow from that. Thresholds belong to the action: in the snippet, reading an account balance and approving a transfer hang off the same Choice with different bars. And for each gate you have to decide which way it fails. Low confidence on "is this spam?" can fail towards the inbox. Low confidence on "should this page someone?" must not fail towards silence.

## "Can't hallucinate", honestly

TypeSafe's home page says "Zero Hallucinations" and the launch post says Jev "*can't* hallucinate". That is true in a narrow sense and worth getting exactly right.

Jev cannot fabricate content. The output is always one of the answers you defined, so it cannot invent a team that does not exist or return a field your code does not expect. A whole class of failure is removed by construction.

It can still be wrong. `"billing"` is a perfectly typed answer to a ticket that was about a bug. TypeSafe's own docs say this in one sentence: "Calibration is measured across groups of predictions; it does not guarantee that an individual answer is correct." There are three ways it goes wrong that matter in practice.

The confidently wrong answer is the hardest to catch: clean, valid, incorrect, with 0.95 next to it. An LLM hallucination often gives itself away: the URL 404s, the function does not exist, the JSON does not parse. A wrong enum value looks exactly like a right one, and nothing downstream will raise an error.

A Choice is also a forced choice. If none of your options fit, it still returns one of them, and the probabilities still sum to 1. The fix is in the docs for Choice: "Add an `other` or `none of the above` option when the list might not cover every input, so the model can say none of the others fit." Do that on every Choice whose inputs you do not control.

Then there is calibration. Thresholds only work if the probabilities are honest: of all the answers given at 0.9, about nine in ten should be right. TypeSafe trains for that, on its data. Whether it holds on yours is an empirical question, and the outside numbers so far do not agree with each other. The usual measure is expected calibration error (ECE), the average gap between stated confidence and observed accuracy, where 0 is perfect. open-alternative-jev's README puts Jev at 0.144 on the typed-decisions benchmark, and says that figure is the benchmark authors' own run through TypeSafe's API on 18 September. Laya's README puts Jev at 0.246 without naming the dataset, and says its Jev figures are third-party numbers that Laya's authors never measured themselves. Neither figure is TypeSafe's, and both reach you through the README of a project that competes with Jev. Treat them as a reason to measure and nothing more. The check is cheap: label a few hundred of your own cases, bucket the answers by confidence, and compare each bucket's confidence to its accuracy.

One more thing about the framing. The absence of fabrication is a property of bounded, typed output. It is not a property of "System 1" as a concept. Human System 1 is the part of us that answers "ten cents" to the bat-and-ball question, quickly and with total assurance. Fast intuition makes confident errors; that is most of what Kahneman's book is about.

Jev can't make things up, but it can make wrong decisions, sometimes confidently.

## The open-source alternatives

Jev is closed, hosted and waitlisted, which is a reliable recipe for open reimplementations. [Pinggy's roundup](https://pinggy.io/blog/best_open_source_jev_alternatives_self_hosted_decision_models/) says the first ones appeared within about 24 hours of the launch, and by the time it was published on 23 September the list it cites was tracking more than a dozen. On 1 October Cloudflare [released its own](https://developers.cloudflare.com/changelog/post/2026-10-01-clef-workers-ai/) open-weight decision models, Clef and Clef-flash.

The ones worth knowing about fall into three groups: encoder models with decision heads (Laya, Von), LLM backbones adapted for typed decisions (Kev, JevK5, Clef), and libraries that train nothing and read option probabilities straight off a frozen model's logits (SemIf, open-alternative-jev). jevos I cannot place: it is small enough for a laptop CPU, and its README does not say what it is built on. Every number in this table comes from the project's own README or announcement, and where that README is quoting someone else's measurement the cell says so.

| Project | Runs where | License | Speaks Jev's API? | Size and hardware | Latency (their number) | Accuracy claim, and who ran it |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| Jev | TypeSafe's cloud, plus Vercel and Cloudflare gateways | Proprietary, closed weights | It is the API | None of yours | 70 to 500 ms (TypeSafe) | See the rows below: everyone benchmarks against it |
| [Clef, Clef-flash](https://developers.cloudflare.com/changelog/post/2026-10-01-clef-workers-ai/) | Workers AI, or self-hosted from Hugging Face | Apache 2.0 | Yes: "follows the System One API" | 27B and 9B parameter models | 209.3 ms and 38.8 ms median, against 524.1 ms for Jev (Cloudflare, 43 runs) | A Clef model scores highest on 7 of 10 decision benchmarks (Cloudflare) |
| [Laya](https://github.com/NandhaKishorM/laya) | Self-hosted; also on Vercel AI Gateway | Apache 2.0 | Yes: `laya-serve` exposes `POST /v1/systemone` | CPU or GPU; 421M parameters | 39.5 ms for one question on a Tesla T4 with the English checkpoint, 32.8 ms with the smaller multilingual one | 0.766 against Jev's 0.727 on typed-decisions with the fine-tuned `laya-typed-decisions` checkpoint; 0.362 with the base English one (authors; the Jev figure is third-party and Laya did not run it) |
| [Kev](https://github.com/jaredpalmer/kev) | Self-hosted | Apache 2.0 | Yes: the TypeSafe Python SDK can point at it | 0.8B to 27B; the 27B needs an 80 GB GPU | 18.1 ms of model time for six questions, Kev-4B on an H100 | Kev-27B 0.851 against Jev's 0.857 on new sources (authors) |
| [Von](https://github.com/wfzyx/von) | Self-hosted | Apache 2.0 | Yes: "a drop-in server and client" for the wire protocol | CPU, CUDA, ROCm or Apple MPS; 395M parameters | Raw p50 0.096 s on a 4-vCPU Xeon (OpenVINO), 0.023 s on an A10G; 0.34 s on CPU in the JevBench v1.4 table | Composite 27.5 against Jev's 63.3 on JevBench v1.4, and 0.279 against 0.367 on its sealed set (figures as given in Von's README) |
| [JevK5](https://github.com/allebee/jevk5) | Self-hosted | Apache 2.0, code and weights | Accepts the `/v1/systemone` request shape | About 9 GB of GPU memory for the 4B model | p50 13.2 ms on an H100 | 33.1% against Jev's 36.7% on the 308 sealed decisions of JevBench v1.4, which the README describes as independent |
| [jevos](https://github.com/feder-cr/jev) | Self-hosted | MIT | Yes for yes/no questions, which is all the README claims; the server also accepts Choice and Score and expands them into yes/no questions | Laptop CPU, about 1 GB of memory | 25 to 110 ms | 0.810 against Jev's 0.927 on 2,000 yes/no questions from unseen policies (jevos's README; the figure is for jevos-v2, and the current model is v3) |
| [SemIf](https://github.com/TheoLeeCJ/SemIf-OpenJev) | Self-hosted | MIT | Not stated in the README | RTX 3090; CPU and Apple Silicon backends exist | 1.023 s median for 21 yes/no criteria | 0.845 agreement against Jev's 0.883 on 102 rows (authors, with the Jev figure read from TypeSafe's published records) |
| [open-alternative-jev](https://github.com/ikermoel/open-alternative-jev) | In-process Python library | Apache 2.0 | No, and the README says so | CPU for small models, a GPU for large ones; Hugging Face Transformers or vLLM | 582 ms per case with a 27B model | 73.7% (author) against Jev's 72.7% (the benchmark authors' run) on the 400 cases of typed-decisions |

Some notes the table cannot carry.

The API compatibility column is the one I would look at first. Laya, Kev, Von, jevos and Clef all say they serve TypeSafe's own `/v1/systemone` contract (jevos promises unchanged client code only for yes/no questions), and the official SDK takes a base URL (a `base_url` argument, or `TYPESAFE_BASE_URL` in the environment). So a client written for Jev can be pointed at a pod in your own cluster by changing that URL; Cloudflare says Clef needs the model name changed as well. That makes the choice reversible in both directions: start on the hosted model and move in-house later, or prototype locally while you wait for an invite.

Calibration is the column I left out, because the figures do not line up into a column. Most projects publish something:

- Laya: an ECE of 0.081, after temperature scaling.
- Kev: a fitted temperature shipped with every checkpoint, which the README says takes Kev-9B's calibration error from 0.103 to 0.041 on new sources.
- SemIf: a temperature fitted per workload, with 0.208 falling to 0.069 on WANLI.
- JevK5: ECE on JevBench's hard tier, between 0.054 and 0.126 depending on the version.
- Von: a JevBench calibration score of 75.7 against Jev's 76.3, and a `von calibrate` command for refitting on your own labels.
- open-alternative-jev: a raw ECE of 0.020 for its 27B run on typed-decisions, next to the 0.144 it quotes for Jev. The same README warns that the library is "not calibrated out of the box", and that fitting a temperature raised its hard-label ECE from 0.020 to 0.135.

I found nothing on calibration in jevos's README or in Cloudflare's Clef announcement. These are different metrics on different data, and several are post-temperature figures, with the temperature fitted on the project's own data or on data the README does not describe. None of them tells you how the probabilities will behave on your tickets or your alerts.

Read the accuracy column sideways. On typed-decisions, which open-alternative-jev's README calls "the benchmark the community uses", two open projects come out ahead of Jev: open-alternative-jev by a point and Laya by four. (I am assuming Laya's row is the same benchmark; it has the same name, the same 2,000 decisions and the same 0.727 for Jev.) Laya's lead comes with a caveat. It belongs to a fine-tuned checkpoint called `laya-typed-decisions`, and the same README puts Laya's base English checkpoint at 0.362 on the same decisions, half of Jev's score. jevos's chart has Laya at 0.489 on its yes/no set.

On Kev's own suite Jev is ahead of Kev-27B by less than a point, of the 4B and 9B by about four, and of the 0.8B by twenty-one. On JevBench's sealed set it leads JevK5 by 3.6 points and Von by nearly nine, and Von's composite score is less than half of Jev's. On the yes/no set in jevos's README Jev is ahead of jevos by almost twelve. That is four of the benchmarks in the table, and for two of them (Kev's suite and jevos's yes/no set) the project being measured is the only source I have. Across them the open model lands anywhere from four points ahead to a long way behind, and one model, Laya, moves from 0.362 to 0.766 on the same decisions depending on which checkpoint you load.

So I would not pick from this table. I would label a few hundred of my own cases and run them through Jev and two of the open models. The first is Kev, because its 27B model comes within a point of Jev on its own suite and the TypeSafe SDK can point at it. That figure needs an 80 GB GPU; the 4B and 9B are about four points back. The second is Laya, with the numbers above in mind: it is an encoder that runs on a CPU and serves `/v1/systemone`, so it is cheap to stand up, and what I would want to know is how far its base checkpoint gets on my labels before any fine-tuning. The same client and the same questions work against all three, so on the client side the comparison is a change of base URL.

There is also [NanoJev](https://github.com/TianyuCodings/NanoJev), a 0.6B Qwen backbone with decision heads trained on four game environments (two ViZDoom tasks, a maze, Snake) and aimed at control loops. It is a different use from everything else in this post, and I mention it because it shows where the idea goes when a decision is one step in a loop and not one call in a request handler.

### The older ways to get a typed answer

None of this started in September. Two older approaches solve an overlapping problem.

Structured-output libraries such as BAML, Instructor and Outlines, and the structured output modes that OpenAI and Gemini provide, make an LLM return JSON that matches a schema. That gives you type safety at parse time. It does not give you a decision engine: the model still generates token by token, and any confidence figure you extract comes from the underlying LLM's token probabilities, which nobody trained to be calibrated for your question.

A trained classifier is the other old answer: fine-tune a small encoder on labelled examples, by hand or with something like Hugging Face AutoTrain. For a fixed label set with plenty of data this is still the cheapest and fastest option, and it is entirely yours. What it lacks is the ability to change the question at runtime. Add a team or reword a category and you are retraining. Laya and Von are, mechanically, this same kind of encoder with the labels moved into the request.

The trade-off between all of these and Jev is plain. Jev is hosted, so your state goes to TypeSafe, or to a gateway and then to TypeSafe. The self-hosted options keep the data inside your network, usually at some cost in accuracy and always at the cost of running a model server.

## Where it fits: general use cases, and three for SREs

The general list is whatever you currently do with a prompt that ends in "reply with one of the following": support ticket routing, refund and intent detection, spam and email triage, content moderation. The newer one is guardrails on AI agents, where a fast model approves or blocks each tool call before it runs. LangChain [shipped middleware](https://www.langchain.com/blog/building-a-harness-with-jev) for this two days after the launch, marked experimental:

```python
from langchain.agents import create_agent
from langchain_typesafe.experimental.middleware import AutoModeMiddleware

guardrail = AutoModeMiddleware(tools=["bash"])
agent = create_agent("openai:gpt-5.6-luna", middleware=[guardrail])
```

What follows is how the same model looks from where I sit, running Kubernetes with Prometheus and Loki. These are designs, not reports. I have not put any of them in front of a pager.

### Alert triage

An alert fires. Before anyone is woken up, three questions need answers: whose is it, how bad is it, and is it the same thing as the incident that is already open? That is a Choice, a Score and a Noul, and they fit in one request:

```python
alert = {
    "alertname": "KubePodCrashLooping",
    "namespace": "payments",
    "pod": "ledger-api-7c9f8d6b5-x2x9q",
    "summary": "Pod payments/ledger-api-7c9f8d6b5-x2x9q is restarting repeatedly",
    "open_incidents": ["INC-2291: ledger-api OOMKilled after the 14:05 deploy"],
}

resp = client.system_one(
    state=alert,
    questions={
        "team": Choice(
            instructions="Which team owns this alert",
            criteria={
                "payments": "Ledger, checkout and billing services",
                "platform": "Cluster, nodes, ingress, DNS, CI",
                "data": "Kafka, Postgres, pipelines",
                "unclear": "None of the above, or not enough information",
            },
        ),
        "urgency": Score(
            instructions="How urgently a human needs to look at this",
            criteria=[
                "Noise, no action needed",
                "Look at it during working hours",
                "Page the on-call now",
            ],
        ),
        "duplicate": Noul(
            instructions="This alert is caused by one of the incidents in open_incidents",
        ),
    },
)

team = resp.answers["team"]
urgency = resp.answers["urgency"]

known_owner = team.choice != "unclear" and team.confidence >= 0.5
owner = team.choice if known_owner else "platform"   # the catch-all rotation
needs_page = urgency.score >= 1.5 or urgency.confidence < 0.5

if resp.answers["duplicate"].noul >= 0.9:
    attach_to_incident(alert)             # a human already owns that incident
elif needs_page:
    page(owner, alert)
elif known_owner:
    open_ticket(owner, alert)
else:
    send_to_triage_queue(alert, resp)
```

Note the `unclear` option, and note the order of the checks. Urgency is tested before ownership, so an urgent alert the model cannot attribute still pages someone (the platform rotation in this sketch), and an alert whose urgency the model is unsure about pages too. Only the low-urgency, no-owner case goes to a queue. Nothing is thrown away, either: an alert the model scores as noise still becomes a ticket or a queue entry, so the only noise this sketch removes is duplicates. That is the price of never failing towards silence. The one branch that can swallow a page is the duplicate one, which is why it sits behind 0.9 and attaches the alert to an incident somebody is already working. A triage layer that drops a real page is worse than no triage layer.

TypeSafe's page of known weak spots for Jev 1.13 shapes what goes in the state. "Jev is not a calculator", so do not ask it whether an error rate crossed a threshold; PromQL already answered that when the alert fired. And accuracy "falls as the state grows with content unrelated to the decision", so send the handful of labels and annotations that matter, not the whole Alertmanager payload with two hundred log lines attached.

### A guardrail in front of a Kubernetes agent

If an AI agent has `kubectl`, something has to stand between its proposed command and the cluster. The commands you never want, deleting a namespace or a PVC in production, belong in RBAC and admission policy, where the answer has no probability attached. A decision model is for the grey area: a scale to zero, a `helm rollback`, a `kubectl drain`.

```python
resp = client.system_one(
    state={"command": cmd, "cluster": cluster, "namespace": namespace},
    questions={
        "destructive": Noul(
            instructions="Running this command deletes data or removes serving capacity in a way that is hard to undo",
        ),
    },
)

if resp.answers["destructive"].noul <= 0.1:
    run(cmd)
else:
    hold_for_human(cmd, resp)
```

The gate reads "run only when the model is at least 0.9 sure this is safe", which for a Noul means a probability of 0.1 or lower that the statement is true. Everything else waits for a person. One warning from the same weak-spots page applies with full force here: "State is data, and `jev-1.13` does not treat it as hostile by default." A command assembled by an agent that has just read a poisoned web page is hostile data.

### Log classification

Tagging error lines by category (timeout, auth failure, OOM, bad config, dependency down) is the volume case. A full LLM per line is too slow and costs too much. A decision model at $0.042 per million tokens is priced for it.

Two practical limits. The hosted API's documented rate limits are 80 requests and 100K tokens per second (the same page says they are "adjusting dynamically"), so classify log patterns after deduplication, not raw lines. And logs are where the hosted-versus-local trade-off bites hardest, because they contain whatever your customers typed. This is the use case where I would look at a self-hosted encoder first.

### Where it does not fit

Anything that needs an explanation, a summary, generated text or multi-step reasoning. "Why is this pod crash-looping" is not a System One question. Neither is a postmortem.

## Jev in front, an LLM and RAG behind

A decision model does not replace your LLM. The architecture that makes sense is a pipeline, with the cheap model as a front-line filter and the expensive one behind it for the cases that need thought.

```text
alert fires
  -> decision model: team, urgency, duplicate?
       duplicate        -> attach to the open incident, stop
       low urgency      -> ticket, stop
       serious, or unsure
         -> retrieve similar past incidents and runbooks
         -> LLM drafts likely cause and next steps from what was retrieved
         -> on-call engineer reads it with the page
```

Every alert pays for the first step, which costs a fraction of a cent. The LLM only runs on the ones a human was going to look at anyway, and by then a few seconds of generation is not what anyone is waiting on.

The retrieval step is RAG. In one sentence: runbooks, postmortems and past incident channels are split into chunks, embedded and stored in a vector database, and at query time the chunks nearest the question go into the prompt for the LLM to answer from. Three refinements matter for ops data in particular. Hybrid search combines vectors with keyword search such as BM25, because embeddings are bad at exact strings like `OOMKilled`, an error code or a pod name, and those are what you search incidents by. Reranking takes the first few dozen hits and reorders them with a more careful model; TypeSafe's docs have a cookbook for using Jev itself there. Metadata filtering restricts retrieval by customer, cluster or service before similarity is considered, so one tenant's incident never lands in another's prompt.

The escalation half is already a product feature in at least one place. Vercel's AI Gateway has "evaluation fallbacks": a condition such as `confidenceBelow: 0.6` on a question reruns the request against a full LLM. Its docs note that a triggered request bills both stages.

There is a parallel between the two halves. RAG reduces hallucination and does not eliminate it: the model can still misread a retrieved runbook, or be handed the wrong one. A decision model eliminates fabrication and does not eliminate wrong answers. Neither is "always correct", and a pipeline built from both still needs the human at the end of it.

## Limitations, and whose numbers these are

The speed and cost figures are TypeSafe's own. The launch post claims 40x to 200x faster than frontier LLMs on System One shaped queries, and the home page says 193.6x faster and 444.6x cheaper. Those two headline numbers come from workflows TypeSafe wrote, and the post itself says "we expect that these are on the higher end of real world gains."

Latency depends on where you are. From the same post: "our published evals are generally run from our laptops on the West Coast (this is where our service is currently based)." From India or Europe, add a round trip across an ocean to every call. Other people's numbers already differ from the brochure: Cloudflare's Clef announcement puts Jev's median at 524.1 ms across its benchmark runs, and Laya's README cites third-party measurements of 236 to 276 ms.

TypeSafe publishes a list of [known weak spots for Jev 1.13](https://docs.typesafe.ai/model-jaggedness/jev-1.13), to its credit. Beyond the ones already mentioned: it reads instructions literally, it handles dates as text and not as ordered quantities, double negatives hurt it, and "the order of a Choice's options can affect the answer, and jev-1.13 leans toward the option that comes first." Shuffle your options and see whether the answer moves.

It is text only, English first (other languages "are handled but not equally well"), limited to 64k tokens per request with 32k for the state plus the longest question, and a Choice takes at most 255 options.

It is a closed model behind an alias. Your thresholds are tuned to one version's calibration, and `jev-latest` will move. Pin `jev-1.13.0`, and re-run your calibration check before you adopt the next one.

Several of the alternatives' benchmarks are run by the project's own authors, and some of the Jev figures in those READMEs are copied from third parties and were never measured by the project quoting them. That applies to the results where Jev loses and to the ones where it wins.

All of this is a snapshot from the first days of October 2026. Version numbers, pricing, access routes and project status in this space change weekly. Check each one before you depend on it.
