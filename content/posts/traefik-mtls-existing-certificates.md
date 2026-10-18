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

## The certificate is already on the disk

A Puppet agent keeps its credentials under `/etc/puppetlabs/puppet/ssl`. Three settings name the files, and `puppet config print` resolves them:

```
$ puppet config print hostcert hostprivkey localcacert
hostcert = /etc/puppetlabs/puppet/ssl/certs/web01.acme.pem
hostprivkey = /etc/puppetlabs/puppet/ssl/private_keys/web01.acme.pem
localcacert = /etc/puppetlabs/puppet/ssl/certs/ca.pem
```

The certificate itself:

```
$ openssl x509 -in /etc/puppetlabs/puppet/ssl/certs/web01.acme.pem -noout -subject -issuer -dates
subject=CN = web01.acme
issuer=CN = "Puppet CA generated on puppet at 2025-12-11 12:31:33 +0000"
notBefore=Dec 23 06:47:41 2025 GMT
notAfter=Dec 23 06:47:41 2030 GMT
```

Two things in that output carry the whole design. The CN is the certname, and in my environments the certname is deliberately two labels, `<hostname>.<customer-id>`, so the certificate says not just which machine is calling but which tenant it belongs to. And the issuer is a CA I control, whose CN is a timestamp of when it was generated. That timestamp matters later.

The certificates are five-year. In one environment the CA is a two-certificate chain, a signing CA under a self-signed `Puppet Root CA: <id>`. Custom facts `hostcert`, `hostprivkey` and `localcacert` return the corresponding `Puppet[:setting]`, so a manifest that configures a client never hard-codes a path.

