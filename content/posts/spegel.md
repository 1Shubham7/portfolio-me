---
title: "Why Is Every Node Pulling the Same Image? Meet Spegel"
description: "Node A has already pulled nginx, Node B has not, and a pod just landed on B. What Spegel does in that moment, and why it is safe when the answer is nothing: a DaemonSet that turns containerd's own content store into a cluster-wide mirror, finds peers through a Kademlia DHT, falls back to the real registry on a miss, and stores nothing of its own. Plus the containerd settings that make it work, the one that silently makes it do nothing, and a kind cluster to watch it on."
dateString: November 2026
draft: false
tags: ["Spegel", "Kubernetes", "containerd", "Containers", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/spegel/spegel.png"
---


I had read the one-line description of Spegel, a stateless cluster-local OCI registry mirror for Kubernetes, and nodded along, and then got stuck on the most basic question about it. Node A has already pulled `nginx:1.27`. Node B has not, and a pod for it has just been scheduled there. Node B is about to go to Docker Hub. What exactly does Spegel do in that moment? The whole tool is in the answer, so this post starts there and fills in the rest around it.

## Two nodes, one image

Without Spegel the answer is nothing, because nothing is there to do it. containerd on Node B resolves `docker.io/library/nginx:1.27`, fetches the manifest from Docker Hub and downloads every layer over the internet. The identical bytes are sitting on Node A's disk one network hop away, and Node B has no way of knowing that. Multiply by however many nodes a Deployment lands on and you have the list from Spegel's README: Docker Hub rate limits, a registry outage that blocks new pods even though the image is already in the cluster, slow starts for large images, and egress paid once per node.

With Spegel, every node runs a Spegel pod, and containerd on every node has been told, through its registry mirror config, to try a mirror on localhost before the upstream registry. So Node B's containerd asks the Spegel pod on Node B for the manifest, then for each blob, by digest. Node B's Spegel does not have them, so it asks the cluster who does. Node A's Spegel advertised those digests when Node A pulled the image, so the lookup comes back with Node A. Node B's Spegel fetches the blob from Node A's Spegel, which reads it straight off Node A's disk, and streams it back to containerd. The image crosses the cluster network instead of the internet, and Docker Hub is never contacted.

If nobody has it, Node B's Spegel answers 404 and containerd moves on to Docker Hub, which is what it would have done anyway. [containerd's hosts docs](https://github.com/containerd/containerd/blob/main/docs/hosts.md) describe the rule: mirror hosts are tried in order, and the upstream `server` is used as a fallback once they are exhausted. The peer lookup gives up after 20 ms by default, so a miss costs a few milliseconds. The project's own FAQ calls it a best-effort cache, and the worst case is the status quo. It cannot make a pull worse.

## How the nodes find each other

Spegel, mirror in Swedish, was first developed at Xenit AB. Its creator, Philip Laine, was a core maintainer of Flux before it, according to [his FOSDEM 2025 speaker page](https://archive.fosdem.org/2025/schedule/speaker/philip_laine/). It ships as a Helm chart that installs a DaemonSet, one pod per node, and each pod plays two roles: a registry that other nodes can pull from, and the mirror that its own node's containerd talks to.

The mirror half rides on containerd's hosts configuration. `config_path` points containerd at a directory, by convention `/etc/containerd/certs.d`, and inside it one `hosts.toml` per registry host says which mirrors to try before the real one. Spegel writes those files itself when it starts, for every registry unless you list specific ones in `mirroredRegistries`. It can do that because containerd reads the directory on the fly; the hosts docs say updates under it do not require restarting the daemon. Nothing in a workload changes. Pods keep saying `nginx:1.27`.

The discovery half is libp2p and its Kademlia DHT, both direct dependencies in the project's `go.mod`. Each Spegel pod advertises the digests present on its node into the DHT, with a TTL of 10 minutes per the FAQ, and a lookup is by digest. That is why it is safe to take a layer from whichever node answers: layers are content-addressed, and containerd verifies the hash of what it receives, so a peer cannot hand it the wrong bytes without being caught.

## Where the bytes actually live

Spegel has no storage of its own. When Node A pulled nginx, containerd wrote the compressed layers into its content store, which keeps each blob as a file under `blobs/<algorithm>/<digest>`. The Helm chart's `containerdContentPath` defaults to the store root, so a layer on Node A is at:

```
/var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/<digest>
```

That file is there with or without Spegel. The Spegel pod mounts the containerd socket (`/run/containerd/containerd.sock` by default) and that content directory, uses the socket to learn which blobs the node has so it can advertise them, and reads the file when a peer asks. It is a window into containerd's cache, not a copy of it. There are no PVCs, and a Spegel pod restart loses nothing because the data was never Spegel's.

The flip side is that an image is shareable only for as long as containerd keeps it on some node. Deleting the pod that used it changes nothing; containerd keeps the image. What removes it is whatever removes it from containerd. The kubelet's [image garbage collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) runs every five minutes, and once disk usage passes `HighThresholdPercent` it deletes images oldest-last-used first until usage is back under `LowThresholdPercent`. A replaced node takes its disk with it, and `crictl rmi` is the manual version. Spegel has no garbage collection of its own and relies on the kubelet's, which the FAQ warns is disabled by default on some clusters, with full node disks as the result.

Rebuild the whole cluster and the cache is empty, so the first pull of every image goes to the real registry again. This is why Spegel does not replace a persistent pull-through cache like Harbor or Zot, which have their own storage and survive a rebuild. The FAQ shows Zot as a secondary mirror via `additionalMirrorTargets`, so a miss in the cluster goes there before it goes to the internet. Dragonfly and Kraken are the heavier peer-to-peer alternatives, with their own components to run. Spegel covers a smaller problem with fewer moving parts.

## Before you install it

Spegel only works with containerd. CRI-O and Docker Engine are out.

`config_path` must be set in containerd's config, and setting it needs a containerd restart that Spegel cannot perform for you, so on self-managed nodes it belongs in the node image or bootstrap config. Spegel exits on startup if containerd is not configured correctly, which at least makes this one loud.

`discard_unpacked_layers` must be `false`. [containerd's CRI config](https://github.com/containerd/containerd/blob/main/docs/cri/config.md) describes it as allowing GC to remove layers from the content store after successfully unpacking them to the snapshotter. Upstream it defaults to `false`, but Spegel's docs say some distributions enable it, and kind does. With it on, the compressed blobs Spegel would serve are gone, the unpacked snapshot is useless to a peer, and Spegel installs fine and does nothing. This is the quiet failure, and the first thing to check.

A private image becomes a cluster-wide image. Once one node has pulled it with credentials, every other node can get its layers from Spegel without any, because credentials are not sent to a registry mirror. The FAQ says there is no good solution yet and that strict image-isolation requirements may rule Spegel out. On a multi-tenant cluster, decide this before installing.

Tags are resolved through peers by default (`resolveTags: true`). Once `latest` has resolved to a digest somewhere in the cluster it keeps resolving to that digest while the image is around, which can block an update you pushed upstream. [The tag resolution page](https://spegel.dev/docs/usage/resolving-tags/) gives the options: turn resolution off, or exclude `:latest$` with `registryFilters` and let the registry answer for that tag. Digests are never a problem.

Check your distribution first. K3s and RKE2 embed Spegel as their [embedded registry mirror](https://docs.k3s.io/installation/registry-mirror), switched on with `--embedded-registry` on servers or the matching config-file key, and K3s does not share `latest` between nodes by default. The compatibility section of [Spegel's getting started guide](https://spegel.dev/docs/getting-started/) rates each flavour and lists the extra configuration it needs.

For observability there are Prometheus metrics with an optional ServiceMonitor, and a debug web page in the pod.

## Try it on kind

kind needs both containerd settings patched in at creation. Newer kind images already set `config_path`; the discard setting is the one that matters here.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry]
    config_path = "/etc/containerd/certs.d"
  [plugins."io.containerd.grpc.v1.cri".containerd]
    discard_unpacked_layers = false
nodes:
- role: control-plane
- role: worker
- role: worker
```

Create the cluster, install the chart, pin a first pull to one worker, then scale a Deployment and watch the Spegel pods while it comes up.

```bash
kind create cluster --config kind.yaml
helm upgrade --create-namespace --namespace spegel --install spegel \
  oci://ghcr.io/spegel-org/helm-charts/spegel
kubectl run first --image=nginx:1.27 \
  --overrides='{"spec":{"nodeName":"kind-worker"}}'
kubectl create deployment web --image=nginx:1.27 --replicas=3
kubectl logs -n spegel -l app.kubernetes.io/name=spegel -f
```
