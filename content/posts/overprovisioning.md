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

## Pods scale in seconds, nodes in minutes

The chain is worth spelling out once, because every fix later in the post attacks one link of it. The HPA recomputes its replica count every loop, and its default scale-up behaviour adds up to four pods or 100% of the current count every 15 seconds, whichever is more. Those pods reach the scheduler, which finds no node whose free requests can hold them, and marks them unschedulable. Cluster Autoscaler notices, picks a node group that would fit them and asks the cloud for an instance; Karpenter does the same with an instance it sizes for the pending pods. Then the part no controller can shorten: the instance boots, the kubelet joins, the node turns `Ready`, and the image is pulled onto a disk that has never seen it. The chat I was learning from put that at two to five minutes; an [AWS walkthrough](https://aws.amazon.com/blogs/containers/eliminate-kubernetes-node-scaling-lag-with-pod-priority-and-over-provisioning) of the same pattern with Karpenter calls it approximately one to two. Either way, autoscaling is working as designed, and the design has a floor.
