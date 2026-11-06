---
title: "Who Dies First When a Kubernetes Node Runs Out of Memory? And How to Control That"
description: "The kubelet does not evict by QoS class. It sorts pods by usage over request, then priority, then how far over, and the only inputs you own are the pod's own requests and its priorityClassName. What requests and limits do, where QoS actually matters (the kernel OOM killer), what the eviction thresholds configure and what they cannot, and why Evicted and OOMKilled are two different programs killing for two different reasons."
dateString: November 2026
draft: false
tags: ["Kubernetes", "SRE", "DevOps", "Scheduling", "Resource Management", "Platform Engineering"]
weight: 2
cover:
    image: "/blog/node-eviction/node-eviction.png"
---

