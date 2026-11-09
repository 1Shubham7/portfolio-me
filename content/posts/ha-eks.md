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


Ask me a month ago what a highly available EKS cluster looked like and I would have said: nodes in three Availability Zones, replicas spread across them, a load balancer in front. That is the right shape, and it is the easy half. I was working through the topic for an interview when the hard half arrived as a question I could not answer: if a node dies, why does its EBS volume go down with it? EBS is network storage. It is not inside the instance. So why does every discussion of HA treat a stateful pod as something that dies with its zone?
