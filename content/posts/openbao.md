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

OpenBao is a secrets manager: a server that stores passwords, API keys, TLS certificates and encryption keys and hands them to clients that can prove who they are. Its own docs call it an identity-based secrets and encryption management system, and the identity part is the point. A secret is not stored under a filename. It is stored under a path that a policy can be written against, and every request for it arrives attached to an identity the server has already checked.

If you know HashiCorp Vault you know most of OpenBao, because it is Vault. On 10 August 2023 HashiCorp moved its products from the Mozilla Public License 2.0 to the Business Source License, which is source-available but not open source. The same move produced OpenTofu from Terraform, and it produced OpenBao from Vault. A group inside the Linux Foundation's LF Edge, organised by people from the Open Horizon project with IBM engineers doing the early work, forked the last MPL-licensed release branch, 1.14.x, and kept it under MPL 2.0. The project still lives under the Linux Foundation and lists itself as an OpenSSF sandbox project. The CLI is `bao` instead of `vault`, the default port is still 8200, and the HTTP API is compatible enough that most Vault client libraries and tooling work against it with little or no change.

## What problem does it solve?
