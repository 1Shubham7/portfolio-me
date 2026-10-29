---
title: "Your LLM Writes Too Much: How ASD-STE100 Makes It Speak a Controlled Language"
description: "ASD-STE100 is a forty-year-old aerospace standard for writing maintenance manuals that nobody can misread, the models already know it, and naming it in a prompt at about 80% strength is the cheapest editing pass there is."
dateString: October 2026
draft: false
tags: ["LLM", "Writing", "Technical Writing", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/asd-ste100/asd-ste100.png"
---

Ask a model what a liveness probe is and you get a paragraph. Somewhere in the middle of it is the answer, wrapped in "essentially", "in order to", "it's worth noting", and a closing sentence that restates the opening one. I had got used to reading past that. Everyone who uses these tools daily has, the same way you stop seeing the ads on a page.

Then at the start of this month Andrej Karpathy [posted on X](https://x.com/karpathy/status/2105819303471976479) that he asks his LLM to explain things in something called ASD-STE100, and that the output is a lot more readable. I had never heard of it. The name looked like a part number. So I went and read about it, and then started putting it in prompts. This post is what I found, in the order I found it. Karpathy's post goes on to diagrams, HTML pages and videos as richer formats; I am ignoring all of that and staying on the writing.

## What is it?

ASD-STE100 is Simplified Technical English, a controlled language. A controlled language is a subset of a natural language with fewer words and stricter grammar, built so that a reader who is not fluent, or who is halfway up a wing, cannot read a sentence two ways.

The standard has two parts.

The first is a dictionary of roughly 900 approved words. The thing that makes it different from a vocabulary list is that each approved word gets exactly one meaning and one part of speech. "Close" is approved as a verb: close a door, close a circuit breaker. The adjective "close", as in "close to the landing gear", is not approved; you write "near". "Fall" means to move down by gravity and nothing else, so pressure does not fall in STE, it decreases. "Test" is a noun only, so you do not test the pump, you do a test of the pump. Where English has five words for one idea, the dictionary keeps one: "start" is approved, and "begin", "commence" and "initiate" are listed as not approved with "start" as the replacement. On top of the dictionary you are allowed your own technical nouns and technical verbs, the names of the parts and actions in your field, which is how "kubelet", "cordon" and "request" (the HTTP kind) get into an STE sentence without being in a list written by the aerospace industry.

The second part is a set of 53 writing rules. The ones that do most of the work:

- Procedural sentences (instructions) have at most 20 words. Descriptive sentences have at most 25.
- One instruction per sentence.
- One topic per paragraph, and at most six sentences in it.
- Active voice. In descriptive text, passive is allowed only when nobody knows who the agent is.
- Only simple verb forms: infinitive, imperative, simple present, simple past, simple future, and the past participle as an adjective ("the installed unit"). No "is being restarted", no "has been configured".
- No "-ing" forms except inside technical nouns. "Logging" the noun is fine; "the kubelet is restarting the container" is not.
- Noun clusters of at most three words. "Kubernetes node drain runbook step" is five nouns in a row and has to be broken up.
- Do not leave out words to make a sentence shorter. Articles and verbs stay.

Hold those in your head while reading an average LLM answer and you can see every rule it breaks in the first sentence.

## Where did it come from?

The original users were aircraft mechanics. In the early 1980s the European airlines asked AECMA, the European aerospace industry association, to look at the English in maintenance manuals, because a large share of the people maintaining their fleets did not speak English as a first language and a misread step on an aircraft is not a documentation problem. In 1983 AECMA decided to write its own controlled English instead of adopting an existing one, and the first AECMA Simplified English guide came out in the middle of that decade. The ATA specification for maintenance manuals has required it since 1986, so if you have ever seen a page of an aircraft maintenance manual, you have seen STE.

AECMA merged into ASD (the Aerospace, Security and Defence Industries Association of Europe) in 2004, and the guide was renamed ASD-STE100 and turned into a specification. It was a paid document for a long time. Since Issue 6 in 2013 it has been free. The current issue is Issue 9, released on 15 January 2025, and with it ASD changed the subtitle from "specification" to "standard" and now describes it as an international standard for technical documentation. That is ASD's own label; it is not an ISO standard, and I would not call it one. Issue 9 added no new rules, which makes twelve years without one. It reworded 31 of the 53 and updated 555 dictionary entries.

Getting a copy is a form on [asd-ste100.org](https://asd-ste100.org/) that asks who you are and what you will use it for, and the PDF follows. ASD's distribution log for the previous issue showed that most of the people requesting it were outside aerospace and defence.

## Why does naming it in a prompt work?

Karpathy's reasoning, from the post:

> Writing. Something I've had success with: Ask your LLM to explain something in ASD-STE100, it's a controlled language specification originally developed for aerospace maintenance documentation. LLMs well-versed in this language and it comes with heavy constraints on clean writing style that I often find a lot more readable. Sometimes I've tried to soften it a bit e.g. ask for "80% of the way to ASD-STE100" because the spec is quite stringent.

Two things make this work and neither is magic.

The first is that the models have read the standard, the forum threads about the standard, and decades of manuals written in it. "ASD-STE100" is a name that points at a large, consistent body of text with a very recognisable shape. Saying it is the same kind of instruction as "write this as a man page": you are not explaining a style, you are naming one the model already has.

The second is what the rules forbid, because the list of things STE forbids is almost exactly the list of things LLM prose does by default:

- Hedges. "Essentially", "generally speaking", "it's worth noting", "may opt to". There is no approved word for most of them and the one-meaning rule kills the rest.
- Padding. "In order to" becomes "to". "In scenarios where it isn't" becomes "if it fails". The 20-word limit does this on its own.
- Synonyms for variety. Models are trained on prose that rotates "container", "workload" and "the running process" to avoid repetition. STE uses one word for one thing, every time, which is what you want in a runbook.
- Nominalisations. "Make a comparison of the two readings" becomes "compare the two readings". That one is the dictionary's own example under "comparison", which is not approved as a noun because "compare" is approved as a verb.
- Passive voice. "The container will be restarted by the kubelet" becomes "the kubelet restarts the container", and now the sentence says who does it.
- Multi-clause sentences. One instruction per sentence means the "and then, provided that, unless" chains get broken into steps, and steps are what a reader at 3 a.m. can follow.

Naming the standard gets all of that in ten characters of prompt. I have not found a cheaper editing pass.

## The same text three ways

Every example below is the same explanation written three times: the kind of paragraph a model produces with no style instruction, the same content in full STE, and the "80% of the way to ASD-STE100" setting Karpathy mentions. The first version in each set is illustrative. I wrote it to show the pattern, it is not a quote from a specific model run; if you have used these tools you will recognise it anyway. The STE versions are mine, written against the rules with the dictionary open, and I would still not call them certified compliant. More on that under limits.

### A liveness probe

The typical answer:

> A liveness probe is essentially a mechanism that Kubernetes leverages in order to ascertain whether a container is still functioning as expected, and in scenarios where it isn't, the kubelet may opt to restart it.

Full STE:

> A liveness probe makes sure that a container operates correctly. The kubelet sends the probe to the container at set intervals. If the probe fails, the kubelet restarts the container.

80% STE:

> A liveness probe checks whether a container is still working. The kubelet runs it on a fixed interval, and if it fails a set number of times in a row, the kubelet restarts the container. The pod stays where it is; only the container restarts.

One sentence of 35 words became three of 10, 11 and 9. Two dictionary calls are visible in the full version. "Makes sure" is there because "check" is approved only as a noun, and "operates" is there because "work" and "run" are not approved as verbs at all. "Fails" and "restarts" are my own technical verbs: the dictionary wants "if ... not" and "start ... again" instead, but Kubernetes documents a probe as succeeding or failing and a container as restarting, and I am not going to rename either. The 80% version gets back "checks" and "working", a compound sentence, and one clarifying detail that full STE would make you spell out as a separate instruction. That last sentence is the one I would actually want in a doc.

### A runbook step: drain a node

The typical answer:

> Before performing maintenance on a node, it is generally recommended to first cordon the node in order to prevent any new pods from being scheduled onto it, after which you can proceed to drain it, which will gracefully evict the existing workloads, ensuring they are rescheduled elsewhere in the cluster. Once the maintenance has been completed, don't forget to uncordon the node so that it can once again accept workloads.

Full STE:

> 1. Cordon the node. New pods cannot start on the node.
> 2. Drain the node. The pods on the node stop. New pods start on other nodes.
> 3. Wait until the drain is completed.
> 4. Do the maintenance.
> 5. Uncordon the node. New pods can start on the node again.

80% STE:

> 1. Cordon the node so nothing new is scheduled on it: `kubectl cordon <node>`.
> 2. Drain it: `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`. The pods are evicted and come back on other nodes. Wait for the command to return.
> 3. Do the maintenance.
> 4. Uncordon the node: `kubectl uncordon <node>`.

Full STE did something here that I did not expect: it forced "one instruction per sentence", so the explanation of what each step does became its own short sentence after the imperative, which is the exact shape of a good runbook. It also refused "gracefully", "ensuring", and "don't forget", none of which carry an instruction. The 80% version adds the commands and the flags, which are technical, and keeps the tone of someone who has run this before.

### An incident write-up paragraph

The typical answer:

> At approximately 14:02 UTC, a deployment was rolled out which inadvertently introduced a misconfiguration in the service's connection pool settings, leading to a situation where the number of upstream connections was significantly higher than anticipated, and as a result, the database became close to its connection limit, causing elevated error rates for end users. The issue was identified by the on-call engineer and a rollback was performed, after which error rates fell back to normal levels.

Full STE:

> At 14:02 UTC we deployed a new version of the API service. The new version had an incorrect connection pool configuration. The service made many more database connections than before. The database was near its connection limit. Some requests from users had errors. The on-call engineer found the cause. We did a rollback of the deployment at 14:19 UTC. The error rate decreased to its usual value.

80% STE:

> At 14:02 UTC we deployed a new version of the API service with a wrong connection pool setting. The service opened far more database connections than before, the database got close to its connection limit, and users started seeing errors. The on-call engineer traced it to the deployment and rolled it back at 14:19 UTC. Error rates returned to normal within a minute.

Three dictionary rules show up in the full STE version. "Close to its connection limit" is out, because "close" is only a verb; it has to be "near". "Error rates fell" is out, because "fall" only means move down by gravity; the approved word for a number going down is "decrease". And "performed a rollback" is a nominalisation with an unapproved verb in front of it, so it becomes "did a rollback" or just "rolled back", depending on whether you treat "roll back" as a technical verb of your own. The 80% version keeps "close" and "returned to normal" because nobody reading a postmortem will misread them, and that is the whole point of the dial: full STE is optimised for a reader who might, and your colleagues mostly will not.

The incident is made up and the times are there to give it a shape, nothing more.

### A config comment

The typical answer, as a comment a model will happily put above a probe:

```yaml
# This liveness probe is configured to periodically check the health endpoint
# in order to ensure that the container is still responsive and healthy.
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

Full STE:

```yaml
# The kubelet sends a GET request to /healthz at 10-second intervals. After 3 failures it restarts the container.
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

There is no 80% version because the STE one is already what a human would write. The comment went from restating the keys below it to saying the one thing the keys do not: what happens when it fails. ("Every 10 seconds" would have been the natural phrasing. "Every" is not approved; the dictionary sends you to "interval".)

## Where full STE fights you

The dictionary was written for aircraft, and some of its choices are strange in a terminal.

The version of the liveness example I started this post from was this: "A liveness probe checks if a container works. The kubelet runs the probe at a set interval. If the probe fails, the kubelet restarts the container." It reads like STE. It has the sentence lengths, the active voice, the one instruction per sentence. Look each verb up and all five are not approved: "check" is a noun only (you do a check of something), "work" is a noun only (you do work), "run" is replaced by "operate", "fail" is replaced by "if ... not", and "restart" is replaced by "start ... again". I did not see any of that until I opened the dictionary. [NEED: say where this sentence came from. If you wrote it yourself after reading the rules, say so here; if a model produced it and you kept it as STE, say that instead, it makes the same point] The corrected version above is what came out of that, with "fail" and "restart" kept as declared technical verbs, and it is also the first lesson about how much of STE is the dictionary rather than the rules.

"Test" as a noun only is the one I would argue with. "Test the fix" is out; "do a test of the fix" is in. That is correct STE and I will never write it. "Begin" and "commence" are gone in favour of "start", which I agree with. "Accessible" is not approved, the replacement is the noun "access", so "the endpoint is accessible from the VPN" has to become something like "you can get access to the endpoint from the VPN". The three-word limit on noun clusters hits almost every Kubernetes object name: "horizontal pod autoscaler target" is four, "cluster API server certificate rotation" is five, and the rule wants you to split them with a preposition even when the cluster is the actual name of the thing. You are allowed to declare these as technical nouns, which is the escape hatch, but then you are the one deciding what counts, and a model asked for full STE has to make that call on its own without knowing your field.

The verb rules are the other place. No "-ing" forms outside technical nouns means no "while the pod is terminating", no "after draining the node". No continuous or perfect tenses means "has been evicted" is out and "was evicted" is in, and sometimes the perfect was carrying meaning. A model under a length rule also has two ways to comply: find an approved way to say the clause, or delete the clause. Deleting is cheaper, so expect some of it, and the result is a quiet failure: the sentence is compliant and shorter and now missing a detail.

This is why Karpathy's 80% is the practical setting and not a joke. What I want "80% of the way to ASD-STE100" to mean is: keep the sentence limits, the active voice, the one-meaning habit and the hedge removal, and stop policing "test" and "close". I cannot tell you what the 20% a model gives back is in any precise sense, because the model cannot either, which is why the 80% prompt at the end of this post says what to keep in words instead of trusting the number. In practice the target reads as: STE rules, English dictionary.

## What it is bad at, and what "STE-style" is not

Three limits.

STE was built for procedures and descriptions. It has nothing to say about an argument, an opinion, or a tradeoff. Ask for a design review in full STE and you get a list of facts with the reasoning squeezed out, because "I think X because Y, although Z" is three clauses and a hedge. The standard itself is meant to be used beside a style guide and the conventions of your field, not in place of them. [NEED: confirm against your Issue 9 PDF introduction; quote the heading and the sentence after it with a page number, or cut this sentence] Fine for a runbook. Wrong for the paragraph explaining why the runbook exists.

It reads robotic. The short-sentence, one-word-one-meaning style has a sound, and the sound is a manual. For a runbook that is a feature. For a message to a human, two paragraphs of it and they will ask what is wrong with you. It belongs on the text you read, not the text you send.

And a model writing "in ASD-STE100" is approximating it. Compliance means every word in the text is either in the approved dictionary, or a technical noun or verb you have declared, used as the approved part of speech with the approved meaning. A model does not have the dictionary open; it has a strong statistical memory of what STE text looks like. Expect it to get "start" right, because "start" is what it would have written anyway, and expect it to miss "check", "run" and "test", because every other text it has read uses them as verbs. Nothing in the output marks the miss. The first draft of the first example in this post had exactly those mistakes in it, five unapproved verbs in three sentences, and they got past me. If you want to see it yourself, ask your model for the liveness probe explanation in full ASD-STE100 and look for "checks". The STE versions in this post are corrected against the dictionary, and I would still not put them through a checker and expect zero findings.

If you actually need compliant STE, because a contract or S1000D says so, there are real checkers: Boeing's Simplified English Checker, HyperSTE from Etteplan, Congree, and TechScribe's term checker, among others. ASD does not endorse or certify any of them, and the vendors are the first to say what a checker cannot do: TechScribe's documentation for its term checker states that it is not a grammar checker and that you should make sure the text is correct English before you run it. For the rest of us, "STE-style" is the right name for what we are getting, and it is still a big improvement.

## A prompt to start from

The full version, for runbooks, procedures and anything you will read while something is broken:

```
Write the answer in ASD-STE100 Simplified Technical English.
Sentences of at most 20 words. One instruction per sentence.
Active voice, simple present tense. One word for one thing: do not vary terms for style.
No hedges and no filler. Use my technical names (Kubernetes objects, commands, flags) as technical nouns.
Do not drop a fact to make a sentence shorter; split the sentence instead.
```

The 80% version, which is the one to default to:

```
Write about 80% of the way to ASD-STE100 Simplified Technical English.
Keep the short sentences, active voice, one meaning per word, and no hedges or filler.
Normal English vocabulary is fine; do not restrict yourself to the STE dictionary.
Do not drop a fact to make a sentence shorter; split the sentence instead.
```

[NEED: run both prompts at least once and edit them to what you actually use; if they are already what you use, retitle this section back to "The prompt I use"]

The last line is in both for the reason above: left to itself, a model under a length rule shortens by deleting, and the thing it deletes is the detail you asked for.

## Ending notes

The thing that stuck with me is that none of this is new. The question "how do you write so that nobody can misread you" was answered in the 1980s by people whose readers had a torque wrench in the other hand, and the answer was not a tone or a tip. It was a list of words and a sentence limit. The models have read that list. All the prompt does is point at it.

If you read LLM output all day, try the 80% line in your system prompt for a week and then take it out. The difference is in the second half of that experiment.

Thanks for your time :)
