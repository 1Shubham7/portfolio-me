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
