---
title: "OpenBao Is a Secrets Service, Not a Secrets File: Where It Fits Next to Sealed Secrets, and Why It Forked From Vault"
description: "What OpenBao is, from the seal up: auth methods, secrets engines, policies, leases and audit, the three ways it gets into a pod, the gaps Sealed Secrets leaves and the cases where Sealed Secrets is still enough, where OpenBao has moved away from Vault, and a Postgres password stored twice, once as a static value and once as a credential that expires."
dateString: November 2026
draft: false
tags: ["OpenBao", "Vault", "Secrets Management", "Kubernetes", "Security", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/openbao/openbao.png"
---


The question I had when I started reading about OpenBao was not how it worked. It was where it went. I already had ConfigMaps for configuration and Sealed Secrets for anything sensitive, both in Git, both applied by the same GitOps pipeline as everything else. A third place to put a password needed a better reason than "Vault, but open source".

The reason, once I found it, fits in two sentences. Sealed Secrets is a secrets file: an encrypted static value in Git, decrypted once into a plain Kubernetes Secret. OpenBao is a secrets service: an application proves who it is at runtime, OpenBao decides whether to answer, writes the request to an audit log, and hands back something that expires. Everything else in this post hangs off that distinction, so I build up to it: what OpenBao is and why a fork exists at all, the concepts you need before the commands make sense, how it gets into a pod, where it has moved away from Vault, and a Postgres password stored both ways.

## What is it, and why does it exist?
