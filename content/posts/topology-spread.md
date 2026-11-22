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

A constraint names a node label as its `topologyKey`. Every node sharing a value for that label is one domain: `topology.kubernetes.io/zone` gives a domain per zone, `kubernetes.io/hostname` one per node, and any label you set yourself works the same way, a rack label on bare metal or `karpenter.sh/capacity-type` so that not every replica sits on spot. The `labelSelector` says which pods to count, normally the workload's own labels. For each candidate node the scheduler counts matching pods in that node's domain, finds the domain with the fewest, and the difference is the skew. `maxSkew` is the most skew a placement may create.

The exact check is in the [scheduler's filter](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/framework/plugins/podtopologyspread/filtering.go): matching pods already in the domain, plus one for the incoming pod, minus the global minimum, must not exceed `maxSkew`. Take three zones holding 2, 2 and 1 matching pods and a sixth replica arriving with `maxSkew: 1`. The global minimum is 1. Zone A would come to 2 + 1 - 1 = 2, over the limit, and zone B the same. Zone C comes to 1 + 1 - 1 = 1 and passes. The pod can go to zone C and nowhere else. If no node in zone C has room, `whenUnsatisfiable` decides what happens: `DoNotSchedule` leaves the pod Pending and `kubectl describe` says `didn't match pod topology spread constraints`; `ScheduleAnyway` turns the constraint into a score, so zone C is preferred and A or B get the pod if that is what is left.
