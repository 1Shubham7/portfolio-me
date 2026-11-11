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

