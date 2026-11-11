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
