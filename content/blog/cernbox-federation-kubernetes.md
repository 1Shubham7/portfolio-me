---
title: "Deploying CERNBox's federation stack on Kubernetes"
description: "A build log: running CERN's file sync-and-share middleware on Kubernetes, two independent instances federating over OCM, and proof that the shared file never leaves its origin."
dateString: August 2026
draft: false
tags: ["Kubernetes", "CERN", "Reva", "OCM", "Helm", "CERNBox", "Federation"]
weight: 11
cover:
    image: "/blog/cernbox-federation/cover.png"
---

CERN is working on making CERNBox deployable outside CERN, as part of the
EOSC Federation work. That makes a concrete question interesting: what does
it actually take to run this stack on Kubernetes, with federation between
two institutions working end to end?

I spent a week finding out. The end state: two independent Reva instances in
separate namespaces of one cluster, with separate volumes, separate users
and deliberately different secrets. A user on instance 1 shares a file with
a user on instance 2, who reads it through her own instance. And the proof
that this is federation rather than a fancy copy:

```sh
kubectl -n reva1 exec deploy/revad -- find /var/lib/revad/storage -name "my-note*"
# /var/lib/revad/storage/data/einstein/my-note.txt

kubectl -n reva2 exec deploy/revad -- find /var/lib/revad -name "my-note*"
# (nothing)
```

Marie just read that file. Her instance does not have it and never will.
What it holds is a pointer and a credential.

This post is the build log: what the pieces are, how the daemon is put
together, how I got it running on Kubernetes, and what the protocol looks
like on the wire. I hit problems along the way and reported them upstream;
they appear where they happened, but the story is the build.

## What CERNBox and Reva are

CERNBox is CERN's file sync-and-share service: the Dropbox-shaped tool that
tens of thousands of physicists use for their files. The distinctive thing
about it is not the sync client. It is what the service sits on: EOS, the
same storage system the physics compute reads from.

That changes what the tool is. A physicist drops a script into her CERNBox
folder, opens a notebook on SWAN (CERN's hosted Jupyter), and the file is
just there. No upload step, no download step, no second copy. The batch farm
can read the same file through the same namespace. Dropbox structurally
cannot do this: it is a silo with its own storage, and everything entering
or leaving it is a transfer. When your storage is measured in exabytes, you
do not copy it into a product. You put the product on top of the storage.
That is why CERN built instead of bought.

EOS itself, in one paragraph: a distributed storage system built at CERN for
physics data. An MGM node holds the namespace (the metadata: directory
tree, permissions, file locations), FST nodes hold the actual disks, and the
metadata path is separate from the data path, so opening a file means asking
the MGM where it lives and then streaming bytes directly from an FST. It is
built for large files, streaming reads, and exabyte scale.

Between the clients and the storage sits Reva, the middleware this post is
about. Reva is a Go daemon (`revad`) that implements the CS3 APIs, a
protobuf/gRPC contract between the sync-and-share layer and storage.
Storage backends are pluggable drivers behind one interface. CERN runs the
EOS driver; I ran the exact same daemon against a local filesystem driver.
Nothing above the driver knows the difference.

If you know Kubernetes, the analogy is CSI. Kubernetes does not know how to
talk to any particular storage vendor; it defines an interface and lets
backends implement it out of tree. CS3 does the same for sync-and-share:
Reva is the reference implementation, and EOS, CephFS or a plain directory
are drivers.

Why does the middle layer exist at all? Because storage systems store.
They do not do "share this folder with three people, read-only, expiring in
14 days", they do not know what an external collaborator is, and they have
no opinion about identity at another institution. Sharing, identity and
federation are collaboration semantics, not storage semantics. Reva is
where those semantics live.

## What OCM is

The problem federation solves: a CERN physicist wants to share a dataset
folder with a collaborator at SURF, who runs Nextcloud. Every option
without a protocol is bad. Give the collaborator a CERN account, and now
CERN manages identity for the world. Use a public link, and there is no
identity at all. Email the files, and there are now three diverging copies.
Or both sides give up and move to Dropbox, which is how institutions lose
control of their data.

OCM, Open Cloud Mesh, applies the email model to file sharing. Your account
lives at your provider, mine lives at mine, neither of us registered with
the other's system, and it works anyway because there is a protocol between
the providers. Nobody considers it strange that Gmail can mail Outlook.
OCM makes sharing work the same way between Nextcloud, ownCloud, OpenCloud,
Seafile and CERNBox.

The key property, and the design decision everything else follows from: the
file does not move. A share is a pointer to the resource on the owner's
server plus a credential to access it. There is one authoritative copy, and
the recipient's institution gets remote access across the organisational
boundary. No sync, no replicas, no reconciliation.

The protocol is currently an IETF working group draft
(draft-ietf-ocm-open-cloud-mesh-04), with the API spec at version 1.4.0 in
cs3org/OCM-API. Concretely it consists of four flows:

1. Discovery: each server publishes a capability document at
   `/.well-known/ocm`. This is how servers learn how to talk to each other.
2. The invite handshake: sharing requires prior consent. A user generates
   an invite token, hands it to the collaborator out of band, and the
   acceptance registers each user with the other's server.
3. Share creation: a server-to-server POST carrying the pointer and the
   credential.
4. Access: the recipient's server reads the bytes from the owner's server
   over WebDAV, using that credential.

I will show each of these on the wire further down.

## Prior art

CERN has packaged this for Kubernetes before. ScienceBox is their umbrella
Helm chart bundling EOS, CERNBox and SWAN, with published papers behind it.
It depends on the revad chart in cs3org/charts. Both repos have been
dormant for two to three years, and the published chart pins
`appVersion: v1.24.0` while Reva released v3.12.2 during the week I worked
on this.

So none of this is untrodden ground, and I am not claiming otherwise. This
is picking up a stalled effort and finding out how far it still gets you.
The software turned out to be alive and well. The packaging around it is
what had decayed.

## How revad is actually structured

This is the section that makes everything else make sense, and it is the
most interesting design in the codebase.

`revad` is one binary that, on its own, does nothing. The TOML config lists
services, and revad instantiates exactly those. A gateway, an auth
provider, a storage provider, a WebDAV frontend: each is a service you
either configure or do not have. Deployment topology is a config decision,
not a code decision. CERN runs fleets of revads with a handful of services
each; I ran every service in one process. Same binary, same code paths. If
you know kube-controller-manager's `--controllers` flag, it is the same
idea: genuinely different components that happen to ship in one executable.

A revad process has two kinds of listeners. gRPC carries the internal
control plane: the CS3 APIs, service to service. HTTP faces the outside
world: WebDAV for clients, the OCM endpoints for peer servers.

The service that ties it together is the gateway. Every other service is
configured with only the gateway's address, and the gateway knows where
everyone is. On the HTTP side it routes by longest URL prefix; on the
storage side it routes by a mount table that maps namespace paths to
storage providers. It is the cluster's router and front door in one.

One constraint shapes the config: you cannot register the same gRPC service
definition twice on one server. So when you need multiple instances of the
same service type, each needs its own listener. My config gives every
singleton service one shared port and hands out dedicated loopback ports to
the arrays:

```
19000        gRPC: gateway + all singleton services
19010-19012  gRPC: three authproviders, one listener each
19020-19022  gRPC: three storageproviders
19001        HTTP: ocdav + ocm + wellknown + datagateway, longest-prefix routed
19002-19004  HTTP: three dataproviders
```

Why three authproviders? Because federation needs three kinds of
authentication: `json` checks passwords for local users, `machine` lets the
system impersonate users for internal operations, and `ocmshares`
authenticates a remote server presenting a share token. Why three storage
mounts? `/home` holds local files, `/ocm` is the outgoing window through
which peers read what you shared, and `/sciencemesh` is the incoming window
where shares you received appear. That pair of windows is the clearest
picture of how federation composes: your outgoing mount is someone else's
incoming one.

The last structural choice matters most for Kubernetes: byte transfer is
separate from metadata. The storage driver interface has no read or write
call. Instead, an upload or download request returns a URL plus a signed
transfer token, and the bytes flow over HTTP directly against a
dataprovider. The datagateway service exists to be the public face of that:
clients get one public URL, the datagateway validates the token and proxies
to the right internal dataprovider. Keep this in mind; it comes back as a
failure below.

## Getting it running on Kubernetes

The first milestone was one instance in one pod: ConfigMap for the TOML,
Secret, PVC, Service, Ingress. Three Kubernetes-level decisions worth
recording:

`strategy: Recreate`, because the PVC is ReadWriteOnce and the state on it
is single-writer. A RollingUpdate briefly runs old and new pods against the
same files.

Readiness probes by `httpGet` on `/status.php`, which reva serves without
authentication. Liveness by `tcpSocket` on the gRPC port, never `httpGet`:
a gRPC listener does not answer plain HTTP, so an HTTP probe fails forever
and Kubernetes restarts a perfectly healthy pod, indefinitely.

Secrets took an initContainer. revad cannot expand environment variables in
its TOML, so you cannot reference a Kubernetes Secret from the config. I
verified this in the config loader: its template engine resolves references
within the config document and nothing else. So the ConfigMap holds the
config with placeholders, and an initContainer renders the real values from
Secret-backed env vars into an emptyDir that the main container reads.
Every Kubernetes deployment of this software has to invent some version of
this step.

Before any of that ran, the shipped example had to boot, and it did not:

```
error creating reva runtime: rgrpc: grpc service ocmshareprovider could not be started: webapp_endpoint is a required field
```

Reva's curated example configs live in a separate repo, cs3org/reva-configs,
and its `ocm/` example predates a rename in cs3org/reva#5664, where
`webapp_template` became `webapp_endpoint` for OCM spec alignment. The
configs repo is not automatically synced with reva, which the maintainer
confirmed when I filed it. The same stale key sat in three other config
sets in that repo; one PR fixed all four.

Then the second instance. Separate namespace, separate hostname via nip.io,
and a DNS trap good enough to write down: `reva2.127.0.0.1.nip.io` resolves
to 127.0.0.1, and inside a pod 127.0.0.1 is the pod itself. Instance 1
would have federated with itself. The fix is a CoreDNS rewrite pointing
both hostnames at the ingress controller's Service. Both names resolving to
one IP still routes correctly because DNS only picks the TCP destination;
nginx routes on the Host header, which each request still carries.

The two instances deliberately do not share secrets. Within one instance
every service must hold the same `jwt_secret`, because tokens minted at
login are validated everywhere. Between instances there must be no shared
secret at all, because CERN and SURF would never agree on one. Giving my
two instances different secrets is what made the federation test honest.

With OCM services enabled on both sides, the first federated download
failed:

```
Downloading from: http://localhost:19004/data/simple/4fbb804b-d2be-4893-abe3-7ab2524d8199
Get "http://localhost:19004/data/simple/...": dial tcp [::1]:19004: connect: connection refused
```

This is the byte-transfer design from the previous section biting. The
example config sets `expose_data_server = true` with a localhost
`data_server_url`, which hands clients the pod-internal dataprovider
address. That works exactly when the client shares a host with revad,
which on a laptop it does. The fix is `expose_data_server = false` and the
gateway's `datagateway` set to the public URL: clients then receive a
public address carrying a signed transfer token, and the datagateway
proxies to the internal dataprovider. Never hand a client an address only
the server can reach.

The last of the big three: reva's OCM services persist their state in
`ocm-shares.json` and `ocm-invites.json`, and both default to `/var/tmp`,
which is container-local. Cycle a pod and both files are gone. What makes
this worse than an ordinary data loss is what the state is: not a cache
but a trust relationship. The invite handshake with the partner
institution, gone. The shares their server holds now point at a server
that no longer recognises them, and nothing on either side explains why.
One line per file points them at the PVC, and my kill test passes: delete
both pods, everything survives, the federated read still works.

Two smaller notes from the same stretch. Reva's OCM clients hardcode https
toward any non-localhost peer, and the insecure override for a self-signed
lab is spelled three different ways across the services that embed the
client: `ocm_client_insecure`, `client_insecure`, `ocm_insecure`. You find
each spelling through a separate failure. And the `cs3org/reva` CLI image
is a 13 MB binary-only image with no shell, so running it as a helper pod
with `sleep infinity` crashloops; I extracted the binary with
`docker create` + `docker cp` into an alpine-based pod instead.

## The federation flow, end to end

Everything in this section is a real capture from the running setup.

Discovery first. Each instance publishes its business card at
`/.well-known/ocm`:

```json
{
   "enabled": true,
   "apiVersion": "1.3.0",
   "endPoint": "http://reva1.127.0.0.1.nip.io/ocm",
   "provider": "reva1",
   "resourceTypes": [{
      "name": "file",
      "shareTypes": ["user"],
      "protocols": {
         "webdav": "/remote.php/dav/ocm",
         "webdav-receive": { "uri": "absolute" }
      }
   }],
   "capabilities": ["invites", "protocol-object", "invite-wayf"]
}
```

The `webdav` path is the namespace remote peers will read shared bytes
from. Peers fetch this document before talking to each other.

Next the invite handshake. OCM sharing is consent-first: you cannot share
at a stranger. Einstein generates an invite token on instance 1, hands it
to Marie out of band (in real life, an email or a chat message), and Marie
accepts it on instance 2. The moment she does, her revad calls his. From
the ingress log:

```
10.244.0.17 - - "POST /ocm/invite-accepted HTTP/1.1" 200 106 "-" "Go-http-client/1.1"
```

The User-Agent is the point: `Go-http-client/1.1`. No browser, no CLI.
This is one revad calling another. Users only ever talk to their own
instance; the instances talk to each other. After the exchange, each side
lists the other's user as a known federated identity.

Share creation is another server-to-server POST, this time `/ocm/shares`
from instance 1 to instance 2, answered 201. What lands on Marie's
instance is worth staring at (from its state file, trimmed):

```json
{
  "name": "federation-proof.txt",
  "owner":   { "idp": "reva1.127.0.0.1.nip.io", "opaqueId": "4c510ada-..." },
  "protocols": [{
    "webdavOptions": {
      "sharedSecret": "e3drmUykghliCYxd5ggD3XuPFpIRGsep",
      "permissions": { "permissions": { "initiateFileDownload": true, "listContainer": true, "stat": true } },
      "uri": "http://reva1.127.0.0.1.nip.io/remote.php/dav/ocm/29b6ec61-5039-437c-a52a-869dc4afe74e"
    }
  }]
}
```

That object is the share. A URI on the owner's server, a secret, a
permission set. Nothing else crossed the wire.

When Marie reads, the path is:

```
Marie's client
  -> instance 2 gateway
    -> ocmreceived storage driver (the /sciencemesh incoming window)
      -> WebDAV, with the sharedSecret:
         GET http://reva1.../remote.php/dav/ocm/29b6ec61-...
        -> instance 1 ocdav -> instance 1's PVC
```

On instance 1's ingress, her read looks like this:

```
"PROPFIND /remote.php/dav/ocm/29b6ec61-.../ HTTP/1.1" 207 754 "-" "Go-http-client/1.1"
"GET      /remote.php/dav/ocm/29b6ec61-.../ HTTP/1.1" 200 88  "-" "Go-http-client/1.1"
```

Those 88 bytes are the file, streamed from instance 1's volume every time
she reads. And that is why the negative test at the top of this post works:

```sh
kubectl -n reva2 exec deploy/revad -- find /var/lib/revad -name "federation-proof*"
# (nothing)
```

There is no copy, no cache, no sync artifact. If instance 2's volume were
destroyed, Einstein's file would not notice.

The model makes testable predictions, so I tested them.

Revocation should be instant, because there is no copy to chase down.
Einstein removes the share; Marie's next read, the same command that
worked seconds earlier:

```
error: code=CODE_INTERNAL msg="error statting: path:\"/sciencemesh/280d083a-...\""
```

Access died with the share record on instance 1. No propagation delay, no
cleanup job. One rough edge: Marie's received-shares list still shows the
revoked entry. Instance 1 stopped honoring it, but instance 2 was never
told; the spec defines a SHARE_UNSHARED notification for this, and this
version does not send it.

Permissions should be enforced by the owner, since only the owner has the
file. Marie attempts a write to a read-only share and gets a refusal that
originates on instance 1, whose log reads:

```
access token is invalid error="error: permission denied: access to resource not allowed within the assigned scope"
```

The sharedSecret is a scoped token: it encodes what the share permits, and
the owner's server enforces that scope on every request. A recipient
instance cannot grant itself more than the share carries. The contrast
test: a share created with `-rol editor` accepts Marie's write, and
Einstein reads her bytes back on instance 1. Data crossed the federation
in the reverse direction, and there is still no copy on instance 2.

The operational question: what does Marie see when Einstein's server is
down? I scaled instance 1 to zero. Her received-shares list keeps working,
because it is local state. A download fails fast rather than hanging, with
a 503 from instance 1's ingress underneath. When instance 1 comes back,
the same command succeeds with the same bytes and nothing needs repair on
either side. The one thing I would flag: the user-facing error during an
outage is byte-for-byte identical to the error after a revocation. Marie
cannot tell "their server is down, retry later" from "access was removed"
without someone reading server logs.

Two more pushes to make sure the demo was not doing the work. A 10 MB file
of `/dev/urandom` crossed the federation in about half a second with an
identical sha256 on both ends; the ingress annotation
`proxy-body-size: "0"` quietly matters here, since nginx's default 1 MiB
body cap would have cut the upload at the first hop. And a shared
directory with a nested subdirectory traverses like a mounted filesystem
from Marie's side: list the tree, list the subdirectory, download a nested
file, every operation a live WebDAV call against instance 1.

## The Helm chart

I finished by folding the working setup into a Helm chart, so that the two
instances above are literally two `helm install` commands with different
values.

The chart parameterises what the manifests did by hand: instance identity
(the provider domain doubles as the OCM identity), the peer allowlist
rendered into providers.json from values, users from values. The findings
above are encoded as defaults rather than documented as advice: every OCM
state path lives on the PVC, `expose_data_server` is false with the
datagateway templated to the public URL, and per-instance secrets are
generated at install and preserved across upgrades via `lookup`, because a
naive `randAlphaNum` silently rotates them on every upgrade.

One value is deliberately capped: `replicaCount` is 1, and the chart fails
the render if you raise it with a persistent volume attached. The ceiling
comes from the state drivers, not the packaging: the storage metadata is
SQLite and the share stores are JSON files, all single-writer. Real
horizontal scale means moving state to SQL-backed drivers and a shared
storage backend, after which revad is stateless and the cap dissolves.
Moving to SQL is a prerequisite for HA, not an optimisation.

The chart, manifests and a runbook README are at
[github.com/1Shubham7/reva-on-kubernetes](https://github.com/1Shubham7/reva-on-kubernetes).

## Upstream

Everything that looked like a bug got filed while it was fresh: three
issues on cs3org/reva-configs, one on cs3org/charts, and PRs behind them.
The boot failure was confirmed within hours by Giuseppe Lo Presti,
co-author of the IETF draft, who explained the rename and merged the fix
the same day.

On security posture: he also confirmed that RFC 9421 HTTP message
signatures are expected to be implemented on the OCM routes and eventually
enforced as the draft progresses, discovery excepted. Today v3.12.2 relies
on single-use invite tokens plus the providers.json allowlist. I read that
as spec-versus-implementation maturity, the normal state of a protocol in
mid-standardization, not as a finding.

## What is not covered

The EOS driver is untested in this work; everything ran against the local
filesystem driver, which is the point of the driver abstraction but means
nothing here validates the EOS path. The TLS-insecure flags are lab-only:
real federation needs certificates both sides trust. And revad ran as one
pod per instance; the multi-pod split CERN actually operates, where the
shared jwt_secret constraint stops being theoretical, is future work.

## Close

Two `helm install` commands now produce two independent Reva instances
that federate over OCM: invite, share, read, revoke, all across an
organisational boundary, with the file provably never leaving its origin.
The chart deploys the demo drivers and says so; it does not pretend to be
HA, and every non-obvious default in it exists because of something in
this post. The software was never the problem. It was packaging all along.
