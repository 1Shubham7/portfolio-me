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

Picture where the secrets in a typical setup live. A `.env` on someone's laptop. A set of CI variables. A few base64 Secrets in Git that everyone agrees not to call encryption. A database password set in 2024 that has not changed since, because changing it means finding every place it was copied to. Nobody can say who has read it. Nobody can take it back.

OpenBao puts the secret in one place and wraps four things around the act of reading it. Authentication: the client proves who it is before it gets anything. Authorization: a policy says which paths that identity may touch, and policies are deny by default, so an identity with no policy can do nothing. Audit: every request and response is written to an audit device before the response goes out. And expiry: much of what OpenBao hands out comes with a lease, a point in time after which it stops being valid unless something renews it. The fourth is what makes it a different kind of tool rather than a nicer home for the same static password, and the Postgres example at the end is built around it.

## How does it work, from the seal up?

Seven pieces. The first two are about the server itself; the rest are about what clients do with it.

**Seal and unseal.** Everything OpenBao writes to storage is encrypted, and the key that protects it is not on the disk in usable form. A freshly started server is sealed: it knows where its storage is and cannot read any of it, and the only operations it accepts are "what is your seal status" and "here is a piece of the unseal key". The chain is short. Data is encrypted with keys in a keyring, the keyring is encrypted with a root key, and the root key is encrypted with an unseal key. By default the unseal key is split with Shamir's Secret Sharing into a set of shares with a threshold, say five shares with any three required, and `bao operator unseal` is run once per share, by different people if you want it that way. Every node in a cluster is unsealed separately, and a restart puts a node back into the sealed state. That is inconvenient on purpose. Most real deployments use auto-unseal instead, where a cloud KMS or an HSM holds the key that decrypts the root key and the server asks it at startup. Either way the property you get is the one that matters: a stolen disk or a copied storage snapshot is ciphertext with no key next to it.

**Storage.** The usual backend is the integrated Raft storage, so a three or five node cluster replicates its own data and needs no external database. If you have run etcd the shape is familiar: one leader, followers, a quorum. The old single-file backend was deprecated in 2.6, so Raft is the answer even for one node.

**Auth methods.** These are the ways a client proves who it is, each mounted at a path under `auth/`. The one workloads use is `kubernetes`: a pod sends its ServiceAccount token, OpenBao calls the Kubernetes TokenReview API to check that the token is valid, and a role on the OpenBao side says which ServiceAccount names in which namespaces may log in through it and which policies they get. (OpenBao needs permission to call TokenReview, normally the `system:auth-delegator` ClusterRole; when it runs inside the cluster it picks up the reviewer token and the CA certificate from its own ServiceAccount mount, so only the API address has to be configured.) For humans and CI there is `jwt`/`oidc`, which covers SSO through an identity provider and the OIDC tokens GitHub Actions issues. `approle` is for machines outside Kubernetes, `userpass` is for people when there is no SSO, and `cert` is TLS client certificates. Every successful login returns a token with the role's policies attached, and that token is what the client sends with every request afterwards.

**Secrets engines.** Also plugins mounted at paths, and the part that varies most. KV is a plain key/value store; v2 keeps versions of each secret and is what a dev server mounts at `secret/`. PKI makes OpenBao a certificate authority that issues short-lived TLS certificates on request, which is how people do internal mTLS without a spreadsheet of expiry dates. Transit is encryption as a service: the application sends plaintext and gets ciphertext back, and the key never leaves the server. Database is the dynamic one. Given an admin connection to Postgres or MySQL, it creates a fresh database user per request with a TTL and drops the user when the lease ends.

**Policies.** Path-based ACLs in HCL. Everything in OpenBao is a path, so a policy is a list of paths and the capabilities allowed on each: `create`, `read`, `update`, `delete`, `list`, plus `sudo` for root-protected paths and `deny`, which wins over everything else including `sudo`.

```hcl
path "secret/data/payments/*" {
  capabilities = ["read"]
}
```

Two policies are built in. `default` is attached to most tokens. `root` can do anything, and the docs tell you to revoke root tokens before production. Policies are attached to a token at login and cannot be changed on that token afterwards, but a policy's contents are read at request time, so editing one takes effect on the next request.

**Tokens and leases.** A lease is OpenBao's promise about how long a thing stays valid. Every dynamic secret and every service token comes with one: a `lease_id`, a `lease_duration` in seconds, and a flag for whether it can be renewed. The client is expected to renew before the duration runs out. If it does not, OpenBao revokes the lease, and for a database credential that means the user is dropped. Revocation runs the other way too. A lease ID begins with the path the secret came from, so `bao lease revoke -prefix database/creds/myapp` kills every credential that role ever issued, and revoking a token revokes every lease created with it. That tree-shaped revocation is the incident-response feature: one command and a compromised application's entire footprint is gone.

**Audit devices.** A file, syslog or socket sink that receives a JSON line for every request and every response. String values are HMAC-SHA256'd with a salt before they are written, so the log shows that a secret was read and by which identity without containing the secret, and you can still check whether a given value appears by hashing it through `/sys/audit-hash`. One behaviour to know before you enable it: if no enabled audit device can write the record, OpenBao refuses the request. A full disk under your only audit device is an outage, so enable two.

## Why not ConfigMap plus Sealed Secrets?

Sealed Secrets answers one question well: how do I put a secret in Git without putting the secret in Git? You encrypt it with the cluster's public key, commit the SealedSecret, and the controller decrypts it into an ordinary Secret. At that point its job is finished. What you have is a plain, static Kubernetes Secret, and every gap below follows from that.

The secret never changes. The Postgres password sealed in 2024 is still the Postgres password today. Rotating it means generating a new one, changing it in the database, re-sealing, committing, waiting for the sync and restarting the pods, in that order, with a window in the middle where the database and the pods disagree. In practice it does not happen.

Anyone with `get secrets` in the namespace can read it. After unsealing it is base64 in etcd like any other Secret. There is no record of who read it and no way to take it back.

One secret, many copies. Sealing keys are per cluster. If a fleet of a hundred clusters needs the same cloud provider API token, that token is sealed a hundred times with a hundred keys, and rotating it is a hundred commits. (At Obmondo, where I work, a fleet is the shape I think about; the hundred is a round number for the example, not a count.)

It only works for Kubernetes. A CI job, a VM, an Ansible run and an engineer's laptop cannot consume a SealedSecret.

Git history is permanent. If a cluster's sealing key leaks, every SealedSecret ever committed for that cluster is readable, including the ones you rotated away from years ago.

OpenBao's answer to all five is the same answer: the application does not receive a copy of the secret at deploy time. It asks at runtime, proves who it is, and OpenBao decides, logs it and responds. Rotation happens in one place. Reads are attributed to an identity. Clients outside Kubernetes use `approle` or OIDC. Git holds a reference, not a ciphertext. On top of that are the things a static file cannot do at all: issue a TLS certificate on demand, encrypt a payload without handing the application the key, revoke a credential everywhere in one command.

They are not rivals, though, and you usually do not make applications call OpenBao directly. The common GitOps pattern puts External Secrets Operator in between. Git holds an ExternalSecret that names a path, `secret/payments/db` say, and nothing else. ESO logs into OpenBao, reads the path and writes a normal Kubernetes Secret, which the Deployment mounts exactly as it always did. Rotate the value in OpenBao and ESO refreshes the Secret on its next interval. Nothing in Git changes.

When you do not need it: a handful of static secrets on one or two clusters, with no auditor asking who read what, is a case Sealed Secrets handles fine, and adding OpenBao means operating a critical, highly available, unseal-managed service that every deploy depends on. The case starts when you need rotation, dynamic credentials, many clusters sharing a secret, consumers outside Kubernetes, or audit evidence. So the division of labour is: ConfigMap for plain configuration, Sealed Secrets for an encrypted static value in Git, OpenBao for secrets as a managed, audited, expiring service.

## How does it get into a pod?

Three patterns, and they are not exclusive.

The agent sidecar injector is a mutating admission webhook. Annotate a pod with `openbao.org/agent-inject: "true"` and `openbao.org/role: myapp`, add one `openbao.org/agent-inject-secret-<name>` annotation per secret you want, and the webhook adds OpenBao Agent containers to the pod. The agent logs in with the pod's ServiceAccount token, fetches the secrets, renders them to files on a volume shared with your container, and keeps renewing the leases and re-rendering the files for as long as the pod lives. Nothing is stored outside OpenBao and nothing is written to a Kubernetes Secret. The cost is an extra container per pod and an application that reads its credentials from a file. It ships as the `openbao-k8s` project and is switched on with `injector.enabled` in the Helm chart.

The CSI provider goes through the Secrets Store CSI driver. A `SecretProviderClass` with `provider: openbao` lists the objects you want, each with a path in OpenBao, a key inside it and the filename to write, and the pod mounts it as a `csi` volume with the `secrets-store.csi.k8s.io` driver. The provider logs in as the pod's ServiceAccount and writes the files before the container starts, which means a pod is blocked from starting until OpenBao has answered. It needs the CSI driver installed separately and `csi.enabled` in the chart, and `provider: openbao` needs chart 0.18.0 or later.

ESO is the GitOps route from the previous section. There is no OpenBao-specific provider; [ESO's docs](https://external-secrets.io/latest/provider/openbao/) say to use the Vault provider, which they have tested against OpenBao, so the `SecretStore` points `provider.vault.server` at the OpenBao service. The secret ends up as a normal Secret, which is both the attraction (no application changes, works with every Helm chart you already have) and the limitation (it is a static Secret again, readable by anyone with RBAC in the namespace, until ESO's next refresh). For most teams this is where to start.

On the server side, the official Helm chart (`helm repo add openbao https://openbao.github.io/openbao-helm`) runs OpenBao in dev, standalone or HA Raft mode (`server.ha.raft.enabled`) and installs the injector and CSI provider alongside. There is also an OpenBao Operator, an independent project under the KubeBao name rather than part of the OpenBao organisation, whose stated goal is to install and operate OpenBao clusters on Kubernetes. I mention it so you know it exists when you go looking, not as a recommendation.

## How is it different from Vault?

Mostly in what it includes for free, and in a few places where the fork has gone its own way.

Namespaces are the big one. A namespace is a partition inside one server with its own auth methods, secrets engines and policies, so one OpenBao can serve many teams or many customers that cannot see each other. Vault has this only in the Enterprise tier. OpenBao shipped it in open source with the 2.3 beta in May 2025, and 2.6 added per-namespace sealing on top, so a tenant can seal its own slice without touching anyone else's. For an MSP that is the difference between one server and one per customer.

Underneath are two pieces of storage work that Vault's open-source edition never did: transactional storage, so a multi-step write either fully happens or does not (merged for the Raft and PostgreSQL backends in late 2024), and paginated list operations, which shipped in 2.0. Neither is something you notice from a `bao kv get`. They are the kind of plumbing you want in place before carving one server into many tenants.

CEL. OpenBao uses Common Expression Language, the same language Kubernetes validating admission policies use, in two engines: JWT/OIDC auth, where a CEL program inspects the token's claims and decides which policies to grant, and PKI, where a CEL program decides what certificate a request is allowed to receive. Vault has neither.

Kubernetes ergonomics. 2.6, released in July 2026, added `bao login -method=kubernetes`, so a pod or a person with a ServiceAccount token logs in with one command instead of hand-building a `bao write auth/kubernetes/login` call, and the container images now run as the `openbao` user rather than root by default.

A slimmer core. [2.7](https://github.com/openbao/openbao/releases/tag/v2.7.0), released on 23 September 2026, took the cloud KMS seals (`awskms`, `azurekeyvault`, `gcpckms`, `ocikms`, plus `alicloudkms` and `pkcs11`) out of the binary; they are external plugins now, installed separately. LDAP, Kerberos and RADIUS auth, and the LDAP secrets engine, moved out to the `openbao-plugins` repository in the same release. If you auto-unseal with AWS KMS or log people in through LDAP, read the 2.7 notes before upgrading, because the seal is the first thing a server needs on startup.

What is missing. Vault Enterprise's performance replication and disaster-recovery replication, the cross-cluster kind, are not in OpenBao. Raft replicates within a cluster, so a lost node is fine. A lost cluster, or a bad upgrade, is your backup's problem, which makes a scheduled Raft snapshot and a restore you have actually rehearsed part of the install, not something to add later.

## What does storing a Postgres password look like?

Two ways, with a third in between. The first is Sealed Secrets with a server in the middle. The second is the thing Sealed Secrets cannot do. Everything below runs against a dev server, which is the right place to learn the flow and the wrong place for anything else: `bao server -dev` keeps all data in memory, is already unsealed, listens on `127.0.0.1:8200` without TLS, prints a root token at startup and forgets everything when it exits.

```bash
bao server -dev
export BAO_ADDR=http://127.0.0.1:8200
```

### A static password in KV

Store it. On a dev server KV v2 is already mounted at `secret/`, so the `enable` line is only for a real server.

```bash
bao secrets enable -path=secret kv-v2
bao kv put secret/myapp/db username=app password='S0meStr0ngPass'
```

Write a policy that can read only that path.

```hcl
path "secret/data/myapp/db" {
  capabilities = ["read"]
}
```

```bash
bao policy write myapp-db myapp-db.hcl
```

Notice the `data/`. KV v2 puts it in the real API path: `bao kv get secret/myapp/db` reads `secret/data/myapp/db` and hides that from you, but policies see the real path, and a policy written against `secret/myapp/db` grants nothing. The KV docs recommend `bao kv get -mount=secret myapp/db` for exactly this reason, and this is the gotcha that catches everyone the first time.

Let the application's ServiceAccount log in. The role binds a ServiceAccount name and namespace to the policy, so only pods running as `myapp` in `prod` can read that one path.

```bash
bao auth enable kubernetes
bao write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"

bao write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp \
  bound_service_account_namespaces=prod \
  policies=myapp-db \
  ttl=1h
```

Get it into the cluster with External Secrets Operator. Two manifests, both safe to commit. The `SecretStore` is how ESO authenticates, and the `ExternalSecret` is the reference. On older ESO releases the apiVersion is `external-secrets.io/v1beta1`.

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: openbao
  namespace: prod
spec:
  provider:
    vault:
      server: "http://openbao.openbao.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp"
          serviceAccountRef:
            name: myapp
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: myapp-db
  namespace: prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: openbao
    kind: SecretStore
  target:
    name: myapp-db
  data:
    - secretKey: password
      remoteRef:
        key: myapp/db
        property: password
```

ESO creates a Secret called `myapp-db` in `prod` with a `password` key, and the Deployment uses it like any other Secret. Git holds these two manifests and never the password. The `remoteRef.key` has no `data/` in it, because ESO adds the prefix itself for a v2 mount; the policy is the only place you write it by hand. To rotate, run `bao kv put` with the new value and ESO updates the Secret within `refreshInterval`. The pods still need a restart to pick it up, which is the same problem a changed ConfigMap has, and the same fixes apply.

This is already better than a SealedSecret. One copy, a policy per reader, an audit line per read, rotation in one place. But the password itself is still a long-lived string that every pod shares.

### The middle step: static roles

Before going fully dynamic, there is a version that keeps your existing `app` user and lets OpenBao own its password. The database engine's static roles map one OpenBao role to one existing database user and rotate that user's password on a period you choose.

```bash
bao secrets enable database

bao write database/config/app-pg \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@pg.prod.svc:5432/app?sslmode=require" \
  allowed_roles="myapp,app-static" \
  username="bao_admin" \
  password="InitialAdminPass"

bao write -force database/rotate-root/app-pg
```

The `rotate-root` line is the one to pause on. After it runs, only OpenBao knows `bao_admin`'s password, which is the point, and it is also why `bao_admin` should be a user created for OpenBao and not a shared admin account somebody else still logs in with. It needs `CREATEROLE` in Postgres, since it is going to create and alter roles on OpenBao's behalf.
