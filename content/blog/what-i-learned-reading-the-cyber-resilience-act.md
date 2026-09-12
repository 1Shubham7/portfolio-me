---
title: "What I Learned Reading All 71 Articles of the Cyber Resilience Act"
description: "I read the whole of Regulation (EU) 2024/2847, every article and every annex, and mapped KubeAid and LinuxAid against it for my employer. These are the things that surprised me, the mental model I ended up with, and what I would tell the next engineer who has to do the same reading."
dateString: September 2026
draft: false
tags: ["Compliance", "Cyber Resilience Act", "Security", "Open Source", "SRE", "EU Regulation"]
weight: 1
---

## Why I read the whole thing

Earlier this year I was handed the job of working out what the Cyber Resilience Act means for Obmondo, where I work as an SRE. We maintain two open-source projects, KubeAid and LinuxAid, both AGPL, and we run an agent inside customer clusters as part of a managed service. Somebody had to decide which of those things the regulation touches, what we needed to have in place, and by when. I decided the only defensible way to answer that was to read the text. All of it: 71 articles, 8 annexes, and the recitals whenever an article did not make sense on its own.

I expected a long checklist for engineers. What I found was a law that is mostly addressed to governments and test labs, with a short core that applies to companies, and a handful of definitions that decide everything. Most of what follows is about those definitions, and about the mistakes I made or watched other people make on the way to understanding them.

Scope note up front: this is not a map of the regulation. This post is what does not fit in a map: what I think about the law now that I have read it, and what I would do differently if I started again. Nothing here is legal advice.

## Half of the law is not written to you

The first surprise was how little of the text is addressed to a company at all. Seventeen articles, 35 to 51, are about how a Member State accredits and supervises the labs that do third-party assessment. Nine more, 52 to 60, are about the powers of market surveillance authorities. Three, 66 to 68, amend other pieces of EU law. Add the articles on consultation, skills programmes and committee procedure, and you are at roughly half the regulation before you reach anything you could act on.

If I had known the shape in advance I would have read it in a different order: Article 3 for the definitions, Article 2 for scope, then work out your role, then read only the articles addressed to that role. For a manufacturer that is Articles 13 and 14 and Chapter III. For a steward it is Article 24 and the parts of 14 it pulls in.

Role first is the whole trick. The regulation has four roles for companies: manufacturer, importer, distributor, open-source software steward. Each gets its own article and they barely overlap. Reading Article 13, the 25-paragraph manufacturer article, when you turn out to be a steward is two hours you do not get back.

**Work out which role you are before you read a single obligation. Most of the text is not for you.**

## The word that decides everything is "commercial", not "paid"

This is the part I would put at the top of every open-source project's README. The regulation applies to products "made available on the market", and making available is defined as supplying a product "in the course of a commercial activity, whether in return for payment or free of charge".

So the question is never "do we charge for it". The question is "is it supplied as part of a commercial activity". Free software published by a hobbyist with no business behind it is out. The same software, published by a company that uses it to sell something, is in. A free tier, a paid support contract tied to the software, a "managed X" offering where X is your project: any of those can pull a free download into scope.

For KubeAid and LinuxAid the analysis came down to what we actually sell. We sell cluster and server management, and customers can buy that whether or not they used our tools to build the thing we manage. The tools are not the product. That is what keeps us in the steward role, and the internal note I wrote says plainly what would break it: a paid tier, a paid licence, or selling implementation services for those specific tools. If any of those happen, Article 13 lands on us in full. A copyleft licence does not protect you from that, and a company name on the repo does not condemn you to it.

**Free is not the same as out of scope. Commercial is the word, and it is about what you sell, not what you charge.**

## Product or service: three questions, and where it runs does not matter

The second classification I had to make was for the agent we deploy inside customer clusters. It runs on the customer's infrastructure, so the obvious reading is that it is a product we have given them. I spent a while on this one and ended up with a test I now use for everything of this shape.

Can the customer install or remove it themselves? Do they control when it updates? Do they keep it when the contract ends? For our agent the answers are no, no and no. We deploy it, we update it, and when a customer leaves we remove it. If it breaks, only we can fix it. The customer never takes ownership of running the thing, which means it was never placed on the market as a product. It is a delivery mechanism for a service.

The part I had to unlearn is that location is irrelevant. I kept thinking "it runs on their cluster, so it is theirs". The regulation does not care whose hardware the bits execute on. It cares whether a product was supplied to someone who then operates it. Hand it over and let them run it, and it is a product, even if it cost nothing. Operate it yourself, and it is a service, even though it lives on their nodes.

**If the customer cannot install it, update it, or keep it, you are running a service. Where it executes is not the question.**

## The steward role is real relief, and "no fines" is not "no enforcement"

Article 24 is short. I counted four obligations: put in place and document, "in a verifiable manner", a cybersecurity policy covering secure development and vulnerability handling; cooperate with market surveillance authorities on request; report actively exploited vulnerabilities in code you are involved in developing; and report severe incidents on the infrastructure you use to build and release the software. That is the list. There is no Annex I obligation, no risk assessment, no technical file, no conformity assessment, no CE marking, no minimum support period.

And Article 64(10) says administrative fines "shall not apply" to "any infringement of this Regulation by open-source software stewards". The paragraph is worded as a derogation from paragraphs 3 to 9 while the top fine tier lives in paragraph 2, a drafting wrinkle that made me reread it three times, but the recitals are clear that stewards are not fined.

Here is where I see people stop reading. No fines does not mean nobody can touch you. Article 52(3) says market surveillance authorities supervise the steward obligations, and where a steward is not complying, they require corrective action. The lever is an order rather than a fine. It is still a lever.

The other phrase to notice is "in a verifiable manner". Our process lived in an internal wiki and in the habits of the people who cut releases. Nobody outside the company could verify any of it. That one phrase turned an internal process into a public page, and it turned out to be most of our actual work.

**Stewards cannot be fined. They can be ordered. Read "verifiable" as "public".**

## The clock that is already running

Everything in the regulation applies from 11 December 2027, except one thing, and it is the thing with the shortest deadline. Article 14, the reporting obligation, has applied since 11 September 2026. This week, as I write this. And through Article 69(3) it applies to every in-scope product regardless of when it was placed on the market. A product you stopped thinking about in 2020 owes nothing on documentation and everything on reporting.

The shape of it: 24 hours from becoming aware of an actively exploited vulnerability to an early warning to your national CSIRT and ENISA, 72 hours to a fuller notification, 14 days from a fix being available to a final report. Severe incidents run on the same 24 and 72 hour clocks, with the final report one month after the notification.

When I first read "24 hours" I filed it as a legal problem. It is not. It is an on-call problem. Whoever is on call at 3am, looking at a GitHub issue that says "we are seeing this exploited", has to be able to decide, or wake someone who can, and the decision has to be right in the CRA's sense of the words. Our internal severity scale was Low, Medium, High. None of those map to "actively exploited" or "severe incident". A High that nobody is exploiting is not reportable. A Medium that someone is exploiting is. So the first artefact I wrote for Article 14 was not a template. It was a one-page definition of the two triggers, in our own words, that an on-call engineer can apply without a lawyer.

"Actively exploited" itself is narrower than most people assume. The definition in Article 3 is a vulnerability "for which there is reliable evidence that a malicious actor has exploited it in a system without permission of the system owner". Not a CVSS 9.8, and not a Trivy finding. A scanner report full of criticals is not a reporting event, and a boring-looking bug that a customer's incident response team caught being used is.

For a steward the scope is narrower again. The vulnerability report applies "to the extent that" you are involved in developing the affected code, and the incident report applies "to the extent that" the incident hits the systems you use to build and release. A break-in to a user's deployment of KubeAid is not my report to file. A break-in to the CI that builds KubeAid is.

**The 24-hour early warning is an on-call decision, not a legal one. Write the trigger definition before you write the template.**

## ISO 27001 got us most of the process and none of the artefacts

We are ISO 27001 certified, and I went into the mapping hoping that would cover most of the CRA. It did, for one half. Secure development is A.8.25 and A.8.29. Incident management is A.5.24 through A.5.28. Event reporting is A.6.8. When I mapped the CRA onto those controls, the process side was already there: we scan in CI, we review, we sign commits, we triage and patch, we have an escalation chain.

What ISO never made us produce is a list of things the CRA names specifically. A software bill of materials. A public coordinated vulnerability disclosure policy. A public contact address for reporting vulnerabilities. A stated support period. Public security advisories once a fix ships. A process for filing reports with a CSIRT. Not one of those exists because of an ISO control, because ISO is about your management system and the CRA is about your product.

So the gap, when I wrote it up, was not engineering. It was documents and one account: a public policy naming the projects, a CVD page with an acknowledgement timeline and a safe harbour clause, a `security.txt` in each repo, the Article 14 trigger definitions, six report templates, and a registration on ENISA's reporting platform. Nothing in that list took more than an afternoon. Nothing in that list existed before.

**ISO 27001 proves you have a process. The CRA wants artefacts a stranger can find. The overlap is smaller than it looks.**

## Annex I is the only part a manufacturer really needs to internalise

If you are a manufacturer rather than a steward, the regulation is longer for you, and it is easy to mistake the length for complexity. Almost the whole substance is in one annex.

Annex I Part I is the product: one umbrella requirement to be secure in proportion to the risk, then thirteen specific properties, from no known exploitable vulnerabilities at release and secure defaults down to logging and secure data removal. Part II is the process: eight vulnerability-handling requirements, starting with an SBOM and running through timely fixes, regular testing, advisories, a CVD policy, a public contact, secure update delivery and free security updates.

That is 21 things. Everything else a manufacturer has to do is paperwork about proving those 21 things. The risk assessment says which of the thirteen apply and why. The technical documentation records how you met them. The conformity assessment checks the documentation, the declaration signs it, the CE mark advertises it. If your product meets Annex I and you can show it, the rest of the regulation is a filing exercise. If it does not, no amount of documentation will help.

**Thirteen properties and eight processes in Annex I are the law. Everything else is the proof.**

## For the Kubernetes crowd: your stack is "important"

Annex III lists the "important" products, split into two classes. Hypervisors and container runtimes are Class II. So are firewalls and intrusion detection and prevention systems. Class II means mandatory third-party assessment for whoever places them on the market, no matter what standards exist. Operating systems and network management systems are Class I, alongside identity and access management, password managers and browsers. Class I can self-assess, but only by fully applying a harmonised standard that covers the requirements, and as of today those standards do not exist. Until they do, Class I in practice means the same third-party routes as Class II.

Annex IV, the "critical" list, is three categories of hardware: devices with security boxes, smart meter gateways, and smartcards and secure elements. There is no software in that list, whatever some summaries imply.

If your company sells a Kubernetes distribution, you are shipping an operating system, a container runtime and probably a network policy engine, and each of those has a classification. Whether a CNI that inspects application traffic counts as an "intrusion detection system" is a question I could not answer from the text.

**If you sell a container runtime or a hypervisor, a third party assesses it. Nobody gets to self-assess Class I until the standards exist.**

## The regulation thinks in decades

Three numbers stuck with me. The support period, during which a manufacturer has to handle vulnerabilities, is at least five years, unless the product is genuinely expected to be used for less, and you have to write down why. Every security update issued during that period has to stay available for at least ten years after it was issued. Technical documentation and the declaration of conformity are kept for ten years after placing on the market, or the support period, whichever is longer.

I work in a world where a Kubernetes minor version gets patches for about fourteen months. The CRA asks a manufacturer to name, at the time of sale, the month and year after which security updates stop, and to make that date at least five years out. It asks for a security update from 2028 to still be downloadable in 2038. The security requirements in Annex I are things a good team mostly does already. Committing to a support window in writing, and keeping a decade of updates reachable, is something almost nobody does.

**Support for five years, updates reachable for ten, documents kept for ten. Sprints do not survive contact with that.**

## Read the text, not the summaries

I read perhaps a dozen summaries before I read the regulation, and every one of them was wrong in a way that mattered. Not wrong on the headlines: the fines, the dates, the existence of a steward role. Wrong in the qualifiers. Summaries drop "to the extent that". They drop paragraph numbers. They say "old products are exempt" without the paragraph that carves reporting back in. They say "open source is exempt" without the word "commercial".

The scope of this regulation lives in its qualifiers. A summary is a regulation with the qualifiers removed, and what remains is a rougher and usually scarier thing than the law.

The text is not hard to read. It is long and it references itself constantly, but the sentences are plain and the structure is consistent. Keep a list of paragraph numbers as you go and you can trace every obligation back to a specific sentence, which is what you will need when someone in a meeting says "I thought we had to".

**Every summary drops the qualifiers. The qualifiers are where the scope is. Keep the paragraph numbers.**

## What I would do first if I started over

Reporting. The deadline is already here, it applies to things you shipped years ago, and it is the one obligation where the cost of being unprepared is measured in hours. Write the trigger definitions so that on-call can apply them. Register on the reporting platform. Draft the templates.

Then the public documents, because they are cheap. A CVD policy, a security contact, a `security.txt`, a page that says how you develop and how you fix. If you are a steward this is nearly the whole job.

Then, and only then, the per-product paperwork: the risk assessment, the technical file, the support period decision, the SBOM in CI. This is the bulk of the work for a manufacturer and it does not need to exist until December 2027.

I went in expecting a compliance project and came out with a much clearer idea of what my own company is, in the law's terms: a steward of two projects, an operator of one service, a manufacturer of nothing. Getting to that sentence took reading the whole thing. Everything after it was a short list.

[GitHub](https://github.com/1Shubham7) | [LinkedIn](https://linkedin.com/in/1shubham7)
