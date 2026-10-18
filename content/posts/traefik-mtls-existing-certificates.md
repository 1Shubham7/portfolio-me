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

## Traefik: one Secret and one TLSOption

Traefik reads the CA from a Secret in the same namespace as the TLSOption that references it. The CRD reference says the Secret "must contain a certificate under either a `tls.ca` or a `ca.crt` key", and the provider code checks `tls.ca` first, then `ca.crt`.

```bash
kubectl -n traefik create secret generic puppetca-cert --from-file=ca.crt=ca.pem
```

I keep that Secret in git as a SealedSecret, but there is nothing in it a plain Secret cannot hold; a CA certificate is public.

The TLSOption is what turns a CA into a handshake policy:

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: prometheus-client-cert
  namespace: traefik
spec:
  minVersion: VersionTLS12
  clientAuth:
    clientAuthType: RequireAndVerifyClientCert
    secretNames:
      - puppetca-cert
```

`clientAuthType` takes the five values of Go's `tls.ClientAuthType`, under Go's names. Two of them authenticate: `RequireAndVerifyClientCert` requires a certificate and verifies it against the CAs, `VerifyClientCertIfGiven` verifies one if it is sent and proceeds without one otherwise. `NoClientCert`, `RequestClientCert` and `RequireAnyClientCert` either ignore the certificate or accept it unverified, for applications that want to inspect it themselves.

I run two options from the same CA list, and the difference is that one word:

```yaml
tlsOptions:
  api-client-cert:
    minVersion: VersionTLS12
    maxVersion: VersionTLS13
    clientAuth:
      clientAuthType: VerifyClientCertIfGiven
      secretNames:
        - puppetca-cert
        - puppetca-cert-2025-12
        - customer-a-puppetca-cert
        - customer-b-puppetca-cert
  prometheus-client-cert:
    minVersion: VersionTLS12
    maxVersion: VersionTLS13
    clientAuth:
      clientAuthType: RequireAndVerifyClientCert
      secretNames:
        - puppetca-cert
        - puppetca-cert-2025-12
        - customer-a-puppetca-cert
        - customer-b-puppetca-cert
```

That is the `tlsOptions` value of the `traefik/traefik` Helm chart, 41.6.0 at the time of writing (Traefik v3.7.13). The chart renders one TLSOption per key, named after the key, in the release namespace, which is why the Secrets live in `traefik` too.

The Prometheus remote-write endpoint is only ever called by servers. A connection without a certificate from one of those CAs has no business there, and `RequireAndVerifyClientCert` refuses it during the handshake, before a single HTTP byte. The API also serves a browser UI on the same hostname, where people log in and get a JWT, and a browser has no Puppet certificate. So the API gets `VerifyClientCertIfGiven`: a server's certificate is verified, a browser's missing certificate is tolerated, and the application decides per route whether a verified certificate was required. Both options verify a certificate when one is presented. Only the first refuses a client that presents none.

The `secretNames` list is longer than one entry on purpose. Each customer environment has its own Puppet server and so its own CA, one Secret each. Two of the entries are my own CA twice over in time: in December 2025 a storage failure destroyed my Puppet server's CA volume, a new CA was generated, and a new Secret went in next to the old one so that, in the words of the commit, agents not yet re-issued keep working. Two months later the same thing happened on another cluster. The list is the union of every CA any live server might present. Removing an entry is a deliberate step once no client uses it, and nothing in Traefik will tell you when that is.

