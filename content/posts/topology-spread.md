---
title: "What's the Point of 3 Replicas If They're All on the Same Node? Enter Pod Topology Spread Constraints"
description: "A Deployment's replica count says nothing about where the replicas land, and three on one node is one failure from zero. Pod topology spread constraints give the count a shape: a topologyKey, the skew arithmetic the scheduler runs for every placement, a hard zone rule next to a soft hostname rule, matchLabelKeys so a rollout counts only its own ReplicaSet, and minDomains for zones that have no nodes yet. Plus the ways it quietly does nothing: a missing node label, no rebalancing after a drain or scale-down, and a zonal disk that pins the pod first."
dateString: November 2026
draft: false
tags: ["Kubernetes", "Scheduling", "High Availability", "Topology Spread", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/topology-spread/topology-spread.png"
---

`replicas: 3` is a count, not a placement. The Deployment controller makes sure three pods exist. Where they run is the scheduler's business, and a scheduler with no instructions scores nodes on resources and a few soft preferences and puts the pods wherever that comes out, which on a quiet cluster with one roomy node can be the same node three times. Lose that node and the replica count bought you nothing. One level up it is the same story: three nodes in one zone is one zone.

Pod topology spread constraints are how you tell the scheduler that the count has a shape, and this is the short version of them.

## Domains, skew, and one worked pod
