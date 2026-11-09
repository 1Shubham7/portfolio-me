---
title: "Designing a Highly Available EKS Cluster: Why the Disk Is the Hard Part"
description: "What multi-AZ actually means on EKS, drawn out one failure domain at a time: the control plane AWS already spreads for you, node groups per AZ and why the Cluster Autoscaler needs them once pods use EBS, topology spread constraints and PodDisruptionBudgets, and the storage questions I got stuck on. Why an EBS volume survives its node but not its AZ, why deleting a pod does not delete the volume, what WaitForFirstConsumer is for, how a StatefulSet spans AZs on AZ-local disks, and when EFS is the answer and when it is not."
dateString: November 2026
draft: false
tags: ["Kubernetes", "AWS", "EKS", "Storage", "SRE", "DevOps"]
weight: 2
cover:
    image: "/blog/ha-eks/ha-eks.png"
---

