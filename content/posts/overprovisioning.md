---
title: "How E-commerce Companies Overprovision Kubernetes to Survive Sale Days"
description: "The HPA adds pods in seconds and the cloud adds nodes in minutes, and a sale lands in the gap. The pattern that closes it: pause pods under a negative PriorityClass holding space on nodes you already pay for, so real pods preempt them at once and the evicted placeholders pull the next node up in the background. The priority value that quietly breaks it with Cluster Autoscaler, why placeholders need the shape and the node pool of the pods they stand in for, a rule of thumb for how much to hold, growing the buffer on a schedule before the sale, what else has to be warmed up, and the metrics that show it working."
dateString: November 2026
draft: false
tags: ["Kubernetes", "Autoscaling", "Cluster Autoscaler", "Karpenter", "SRE", "DevOps"]
weight: 2
cover:
    image: "/blog/overprovisioning/overprovisioning.png"
---


The sale opens and the traffic graph goes vertical. The HorizontalPodAutoscaler does its job inside its first loop; the [HPA docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) put the controller's sync period at 15 seconds by default, with no stabilisation window on the way up, so new replicas appear almost immediately. Then they sit in `Pending`. Every node is full, and the thing that fixes a full cluster is a cloud API call followed by a boot, a kubelet registering, an image pull and a `Ready` condition, which is minutes on a good day. Pods scale in seconds. Nodes scale in minutes. Customers shop in the difference.

I read up on this for an interview question, "how would you handle a flash sale on Kubernetes", and the answer turned out to be simpler than I expected: buy the nodes before the sale, and keep them occupied with pods that do nothing until something real needs the space. The timeline below is illustrative, with a three-minute node for the sake of a number, but the shape is the whole argument.

| | Without a buffer | With a buffer |
| :-- | :-- | :-- |
| t+0 | traffic spikes | traffic spikes |
| t+15 s | HPA adds pods, they go `Pending` | HPA adds pods, they preempt the placeholders and are `Running` |
| t+15 s to t+3 min | a node is requested, boots, joins, pulls images; errors the whole time | the same node boot happens, for the evicted placeholders; nobody is waiting on it |
| t+3 min | pods schedule, errors stop | placeholders land on the new node, buffer restored |
