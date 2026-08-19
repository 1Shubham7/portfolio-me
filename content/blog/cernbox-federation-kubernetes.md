---
title: "Deploying CERNBox's federation stack on Kubernetes"
description: "Two Reva instances, one federated file share, zero copies. What actually breaks when you deploy CERN's file sync-and-share middleware on Kubernetes, and what the OCM protocol looks like on the wire."
dateString: August 2026
draft: false
tags: ["Kubernetes", "CERN", "Reva", "OCM", "Helm", "CERNBox", "Federation"]
weight: 11
cover:
    image: "/blog/cernbox-federation/cover.png"
---

## The payoff first

Two Reva instances run in separate namespaces of one kind cluster. They are genuinely independent: separate volumes, separate users, deliberately different secrets. Einstein lives on instance 1, Marie on instance 2.

Einstein creates a file and shares it with Marie:

```sh
echo "my new shared file" | curl -X PUT -u einstein:relativity \
  --data-binary @- http://reva1.127.0.0.1.nip.io/remote.php/webdav/my-note.txt
# 201

kubectl -n reva1 exec reva-cli -- reva -host revad:19000 -insecure \
  ocm-share-create -grantee f7fbf8c8-139b-4376-b307-cf0a8c2d0d9c \
  -idp reva2.127.0.0.1.nip.io -webdav /home/my-note.txt
```

Marie, on her own instance, sees a received share and reads it:

```sh
kubectl -n reva2 exec reva-cli -- reva -host revad:19000 -insecure \
  ocm-share-list-received          # note the id in the first column
kubectl -n reva2 exec reva-cli -- reva -host revad:19000 -insecure \
  download /sciencemesh/<received-id> /tmp/my-note.txt
kubectl -n reva2 exec reva-cli -- cat /tmp/my-note.txt
# my new shared file
```

Now the part that matters. Look for the file on each instance's volume:

```sh
kubectl -n reva1 exec deploy/revad -c revad -- find /var/lib/revad/storage -name "my-note*"
# /var/lib/revad/storage/data/einstein/my-note.txt

kubectl -n reva2 exec deploy/revad -c revad -- find /var/lib/revad -name "my-note*"
# (nothing)
```

Marie just read the file, and her instance does not have it. It never will. What her instance holds is a pointer and a credential. That is the entire idea of this protocol, and that empty `find` output is the most convincing thing in this post.

Getting there took three failures in sequence, and each one only appeared after the previous one was fixed. The software is actively maintained. The packaging around it is not. That gap is what this post is about.

## What this stack is

CERNBox is CERN's file sync-and-share service. The unusual part is what it sits on: EOS, the same multi-hundred-petabyte storage that physics compute reads from. Your synced folder and the batch farm see the same bytes. That is why CERN built instead of buying: no commercial product mounts your existing exabyte-scale storage as its backend.

Reva is the middleware between clients and storage. It is a single Go binary (`revad`) whose config decides which services run: a gateway that routes everything, auth and user providers, storage providers, a WebDAV frontend. Storage backends are pluggable drivers behind one interface, which is why I could run the whole thing against a local filesystem driver instead of EOS. The interface it implements is the CS3 APIs.

A comparison that works if you know Kubernetes: CS3 is to sync-and-share what CSI is to storage. Define the interface, let backends implement it. Reva is the reference implementation.

OCM, Open Cloud Mesh, is the federation protocol on top. Institution to institution, no shared accounts, no central broker. A user at CERN shares with a user at another lab; each side only ever authenticates against their own home server. The protocol is now an IETF working group draft (draft-ietf-ocm-open-cloud-mesh-04) and is implemented by Nextcloud, ownCloud, OpenCloud, Seafile, and CERNBox.

The key design decision: the file does not move. A share is a pointer to the owner's server plus a credential to access it. There is no sync, no replica, no eventual consistency. The recipient reads the owner's bytes over WebDAV, live, every time.

## Prior art

CERN has packaged this for Kubernetes before. ScienceBox is their umbrella Helm chart bundling EOS, CERNBox and SWAN, with published papers behind it. It depends on the revad chart in cs3org/charts. Both repos have been dormant for two to three years. The published chart pins `appVersion: v1.24.0`; Reva released v3.12.2 while I was working on this, two days before I checked.

So this is not new ground, and I am not claiming it is. It is picking up a stalled effort and finding out how far it still gets you. The answer: the chart works as a demo on its pinned 2023 image, and crashloops on every current one.

## Failure 1: the shipped example does not boot

Reva's example configs live in a separate repo, cs3org/reva-configs, and it has an `ocm/` directory with two pre-wired server configs for exactly the federation setup I wanted. Running one against the current image:

```
error creating reva runtime: rgrpc: grpc service ocmshareprovider could not be started: webapp_endpoint is a required field
```

Neither shipped config sets `webapp_endpoint`. The cause is a rename in cs3org/reva#5664: `webapp_template` became `webapp_endpoint` to align with the OCM spec, and the field is validated as required. The example predates the rename by about three months, and the configs repo is not automatically synced with reva, which the maintainer confirmed when I filed it.

The fix is two lines. The interesting part is the value semantics: the old `webapp_template` was a fill-in-the-blanks Go template that reva rendered with the share token. The new code deleted the template machinery entirely and just appends the share id to a base URL. The spec now forbids embedding the token in any URI, so carrying the old value forward under the new key would silently advertise garbage links to remote servers. Rename the key, and drop the old template suffix from the value.

Grepping for the old key found three more config sets in the repo with the same latent failure. One PR later, all four boot.

Two smaller traps in the same repo, for anyone following along: revad never creates its state directory, so a fresh container crashes with `open /var/tmp/reva/invites_server_1.json: no such file or directory` until something runs `mkdir -p /var/tmp/reva`. And the example's discovery endpoint lands on a random port, because no global HTTP address is set and the `wellknown` service does not declare one. My configs set a global `[http]` address so discovery shares the main port.

## Failure 2: clients outside the pod cannot fetch bytes

With both instances up, discovery working and the invite flow complete, the first federated download failed like this:

```
Downloading from: http://localhost:19004/data/simple/4fbb804b-d2be-4893-abe3-7ab2524d8199
Get "http://localhost:19004/data/simple/...": dial tcp [::1]:19004: connect: connection refused
```

The gateway handed the client a localhost URL. The example config sets `expose_data_server = true` with `data_server_url` pointing at localhost, which works exactly when the client runs on the same host as revad. It was written for a laptop. In a cluster, the client is in a different pod, and localhost is the client's own network namespace.

The fix taught me why Reva's `datagateway` service exists. Set `expose_data_server = false` and point the gateway's `datagateway` at the public URL. Now clients receive a public datagateway URL carrying a signed transfer token. The datagateway validates the token and proxies to the pod-internal dataprovider, which can keep its localhost address. The public door is one service; the internal addresses never leak.

The rule generalizes to any system that hands URLs to clients: never give a client an address only the server can reach.

## Failure 3: federation state dies with the pod

The OCM services persist their state in two JSON files, and both default to `/var/tmp`:

- `ocm-shares.json`: shares sent and received, including their access secrets
- `ocm-invites.json`: invite tokens and the accepted remote users

`/var/tmp` is container-local scratch. Deploy the defaults, federate with a partner institution, cycle a pod, and both files are gone.

This is the failure that matters most, because the lost state is not a cache. It is a trust relationship with another organisation's server. The invite handshake you did with them is gone. The shares they hold now point at a server that no longer recognises them. Nothing on their side errors in a way that explains it, and nothing on your side warned you.

It gets worse on bare metal: the upstream systemd-tmpfiles default ages idle `/var/tmp` entries out after 30 days, and the invites file is written once per accepted peer and then possibly never again. A quiet federation on a RHEL-family server loses its invites file after a month of calm.

The fix is one line per file: point them at a persistent volume. In my manifests everything lives under the PVC mount, and the kill test passes: delete both pods, wait for reschedule, the invite relationship, the shares and the federated read all survive.

A sidebar rather than a fourth failure: Reva's OCM clients hardcode https for any non-localhost peer, so a lab setup behind a self-signed ingress needs TLS verification switched off. The override is spelled three different ways depending on which service embeds the client: `ocm_client_insecure`, `client_insecure`, and `ocm_insecure`. A share crossing two instances traverses all three clients, so you discover each name through a separate failure. Lab-only flags; real federation needs real certificates.

## What the wire actually looks like

Everything in this section is a real capture from the running setup.

Discovery is a GET to `/.well-known/ocm`. Instance 1 answers:

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

That `webdav` path is the namespace remote peers will read shared bytes from.

The invite handshake: Einstein generates a token, hands it to Marie out of band, Marie accepts on her instance. Her revad then calls his. The ingress log shows the moment one server talks to the other:

```
10.244.0.17 - - "POST /ocm/invite-accepted HTTP/1.1" 200 106 "-" "Go-http-client/1.1"
```

User-Agent `Go-http-client/1.1`. No browser, no CLI. This is server-to-server; the user only ever talks to their own instance. Also absent: any Authorization header. The single-use invite token inside the body, plus a mutual allowlist file, is the entire authentication at this layer.

Share creation is another server-to-server POST, `/ocm/shares`, 816 bytes, answered 201. What lands on Marie's instance is this (from its state file, verbatim except trimming):

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

That object is the share. A URI on the owner's server, a secret, and a permission set. Nothing else.

The read path when Marie downloads: her client asks her gateway, which routes to the `ocmreceived` storage driver, which makes a WebDAV request against that URI with that secret. On instance 1's ingress:

```
"PROPFIND /remote.php/dav/ocm/29b6ec61-.../ HTTP/1.1" 207 754 "-" "Go-http-client/1.1"
"GET      /remote.php/dav/ocm/29b6ec61-.../ HTTP/1.1" 200 88  "-" "Go-http-client/1.1"
```

Those 88 bytes are the file, streamed from instance 1's volume through instance 2's datagateway to Marie. Every read repeats this. There is no copy to fall out of date.

## Revoking a share [UNVERIFIED]

The pointer-plus-credential model makes a testable prediction: revocation should be instant, because there is no copy to chase down. Einstein removes the share:

```sh
kubectl -n reva1 exec reva-cli -- reva -host revad:19000 -insecure \
  ocm-share-remove d7e47b2b-32e2-4c72-994e-ad18b53c6909
# OK
```

Marie's next read, same command that worked seconds earlier:

```
error: code=CODE_INTERNAL msg="error statting: path:\"/sciencemesh/280d083a-e94b-4a1e-93e4-d38afc97cbab\""
```

Access died with the share record on instance 1. No propagation delay, no cleanup job.

One rough edge: Marie's `ocm-share-list-received` still lists the revoked share. Instance 1 stopped honoring it, but instance 2 was never told. The OCM spec defines a `/notifications` endpoint with a SHARE_UNSHARED type for exactly this; the version deployed here does not send it, so recipients accumulate dangling entries that fail only when used.

## Permissions are enforced by the owner [UNVERIFIED]

The shares so far were viewer-role. Marie attempts a write to one:

```
upload: PUT request returned 500 Internal Server Error
```

The refusal happens on instance 1, not hers. Its log:

```
access token is invalid error="error: permission denied: access to resource not allowed within the assigned scope"
```

The shared secret is a scoped token: it encodes what the share permits, and the owner's server enforces that scope on every request. A recipient instance cannot grant itself more access than the share carries. Surfacing the denial as a 500 rather than a 403 is not ideal, but the enforcement itself is exactly where you want it, at the data.

The contrast test: Einstein creates a share with `-rol editor`, Marie writes to it, and the write lands on instance 1:

```sh
kubectl -n reva2 exec reva-cli -- reva -host revad:19000 -insecure \
  upload /tmp/m.txt /sciencemesh/2aa5fbfd-0313-4876-9966-81db6e4f8470
# File uploaded: 33 bytes

curl -s -u einstein:relativity http://reva1.127.0.0.1.nip.io/remote.php/webdav/editable.txt
# written by marie from instance 2
```

Bytes crossed the federation in the reverse direction, and still no copy exists on instance 2.

## What Marie sees when Einstein's server is down [UNVERIFIED]

The operational question for anyone running this: shares reach into another organisation's infrastructure, so what happens during their outage? I scaled instance 1 to zero replicas and looked at the world from Marie's side.

Her received-shares list keeps working. It is local state, four shares listed, no errors. Browsing works until the moment bytes or metadata are needed from the remote.

A download fails fast, no hang:

```
error: code=CODE_INTERNAL msg="error statting: path:\"/sciencemesh/4fbb804b-...\""
```

Underneath, instance 2's log shows what actually happened: `error statting ... error="PROPFIND /: 503"`. The 503 is instance 1's ingress with no backend to route to.

Two observations. First, recovery is automatic: scale instance 1 back up, the same command succeeds with the same bytes, nothing to repair on either side. Second, the user-facing error for an outage is byte-for-byte identical to the error for a revoked share. Marie cannot tell "their server is down, retry later" from "access was removed, stop trying" without reading server logs. That distinction matters operationally and currently is not surfaced.

## Ten megabytes, checksummed [UNVERIFIED]

An 88-byte file proves the protocol, not the data path. Same flow with 10 MB of `/dev/urandom`:

```sh
sha256sum big.bin
# a0967accc81cb5fa5246d7a38fe855282f525367e97982dc8688b435ccec5d7d
curl -X PUT -u einstein:relativity --data-binary @big.bin \
  http://reva1.127.0.0.1.nip.io/remote.php/webdav/big.bin
# 201
```

Share it, download as Marie, hash on her side:

```
10.00 MiB / 10.00 MiB  100.00%
a0967accc81cb5fa5246d7a38fe855282f525367e97982dc8688b435ccec5d7d  /tmp/big.bin
```

Identical hash, about half a second over the ingress path. The `proxy-body-size: "0"` annotation on the ingress is doing quiet work here; nginx's default 1 MiB body cap would have cut the upload off at the first hop.

## Sharing a directory [UNVERIFIED]

Single files are the demo case. Real sharing is a project tree:

```sh
curl -X MKCOL -u einstein:relativity .../remote.php/webdav/project        # 201
curl -X MKCOL -u einstein:relativity .../remote.php/webdav/project/sub    # 201
# PUT project/a.txt, PUT project/sub/nested.txt, share /home/project
```

Marie traverses it like a mounted filesystem, nested paths and all:

```sh
reva ls /sciencemesh/<received-id>          # a.txt  sub
reva ls /sciencemesh/<received-id>/sub      # nested.txt
reva download /sciencemesh/<received-id>/sub/nested.txt /tmp/n.txt
# nested file content
```

Every one of those operations is a live WebDAV call against instance 1 under the hood. After all of these experiments, the `find` on instance 2's volume still returns nothing: no file, no directory, no cached copy.

## Upstream

Everything above that looked like a bug got filed while it was fresh: three issues on cs3org/reva-configs (the boot failure, the ephemeral state, the three insecure spellings), one on cs3org/charts about the chart being pinned two major versions behind, and PRs behind them, including the chart modernization. The boot-failure fix was merged the same day by Giuseppe Lo Presti, co-author of the IETF draft, who confirmed the rename and that the configs repo is not yet synced with reva.

On the security posture: he also confirmed that RFC 9421 HTTP message signatures are expected to be implemented on the OCM routes and eventually enforced as the draft progresses, discovery excepted. The deployed version relies on single-use tokens plus a mutual allowlist. I read that as spec-versus-implementation maturity, the normal state of a protocol mid-standardization, not as a finding.

## Limitations

The chart caps `replicaCount` at 1 and fails the render if you raise it with a persistent volume attached. That ceiling comes from the state drivers, not the packaging: the storage metadata lives in SQLite and the share stores are JSON files, all single-writer. Scaling out means moving state to SQL-backed drivers and a shared storage backend, at which point revad becomes stateless and the cap dissolves. HA here is a driver change, not a replica change.

The EOS driver is untested in this work; everything ran on the local filesystem driver. And the insecure TLS flags are for the lab: production federation needs certificates trusted by both sides.

## Close

The chart and manifests are at [github.com/1Shubham7/reva-on-kubernetes](https://github.com/1Shubham7/reva-on-kubernetes). Two `helm install` commands give you two federated Reva instances; the README is a runbook from empty cluster to the empty `find`. It deploys the demo drivers, it does not pretend to be HA, and every non-obvious default in it exists because one of the failures above put it there.
