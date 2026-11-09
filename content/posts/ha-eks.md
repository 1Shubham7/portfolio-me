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

The answer is that the volume does not die with its node, and it does die with its AZ, and the gap between those two sentences is the whole subject. "Highly available" is only ever a claim about failure domains: which things can disappear together, and what you have arranged to survive it. This post walks the EKS stack by failure domain, quickly for the parts that just reschedule and slowly for the storage, since that is where I was stuck.

## The control plane is AWS's problem

You do not design this part. The [EKS resilience page](https://docs.aws.amazon.com/eks/latest/userguide/disaster-recovery-resiliency.html) says the control plane is at least two API server instances and three `etcd` instances across three Availability Zones in the region, that EKS replaces unhealthy instances across those zones, and that this is what backs the SLA on the API endpoint. There is no setting that puts it in one zone. Everything below is the data plane, and the data plane is yours.

## Nodes: three AZs, and room for one to be missing
