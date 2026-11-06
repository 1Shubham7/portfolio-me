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


`kubectl get pods` has two words for a pod that died over memory, and for a long time I read them as the same thing. `Evicted` and `OOMKilled` come from different programs, for different reasons, and leave different wreckage: one moves the pod to another node, the other restarts a container where it stands. The question I actually wanted answered sits underneath both. When a node runs out of memory, which pod goes first, and what in my manifests decides it?

The answer I carried around was "BestEffort, then Burstable, then Guaranteed". That is roughly what happens and not what the kubelet computes. The kubelet sorts by three keys, none of which is the QoS class, and the only inputs you own are the pod's `requests` and its `priorityClassName`. You cannot tell the kubelet who to pick. You can tell it when to start, and you can shape the inputs. This post is the short version of that.
