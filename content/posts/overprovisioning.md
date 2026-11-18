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
