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
