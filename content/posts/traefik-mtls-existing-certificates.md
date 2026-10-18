---
title: "Authenticating Servers with Their Existing Certificates via Traefik mTLS"
description: "Why the servers I manage authenticate to services in Kubernetes with the certificate their Puppet agent already holds: the Traefik TLSOption and CA Secret, RequireAndVerifyClientCert versus VerifyClientCertIfGiven, the passTLSClientCert header the app reads, and the seven months a stale CA bundle rejected every server without anyone noticing."
dateString: October 2026
draft: false
tags: ["Kubernetes", "Traefik", "mTLS", "Security", "Puppet", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/traefik-mtls-existing-certificates/traefik-mtls-existing-certificates.png"
---

Every Linux server I manage ships metrics to a Prometheus in a Kubernetes cluster and calls a Go API in the same cluster. Both endpoints sit behind Traefik on a public hostname. The usual answer to how a server proves who it is to endpoints like these is a token per host, written into a config file by configuration management, stored somewhere central, rotated when someone remembers. The token is a second identity for a machine that already has one.

It already has one because every one of those servers runs a Puppet (OpenVox) agent, and the Puppet CA signed a certificate for each host the day it enrolled. The subject is the host's certname. The key never leaves the machine. Issuing that certificate, putting it on the disk and keeping it current is already part of how the fleet is managed, for reasons that have nothing to do with my API. The hard parts of mTLS already belong to something else.

So the servers authenticate with that certificate. Traefik trusts the Puppet CA, the Ingresses that only servers should reach require a client certificate, and a middleware hands the certificate's CN to the backend as a header. Most of what follows is about the one decision inside that, which `clientAuthType` goes where, and about the way it failed on me: a CA bundle that was a hand-made snapshot, and a Puppet CA that moved on without it.

Puppet is the CA here because it is what I run. Anything that already issues a certificate per host fits the same shape: step-ca, Vault PKI, SPIFFE, Teleport host certificates, a corporate PKI. If a key pair and a CA-signed certificate are already on every machine for some other reason, this post applies.

