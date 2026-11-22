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

`replicas: 3` is a count, not a placement. The Deployment controller makes sure three pods exist. Where they run is the scheduler's business, and a scheduler with no instructions scores nodes on resources and a few soft preferences and puts the pods wherever that comes out, which on a quiet cluster with one roomy node can be the same node three times. Lose that node and the replica count bought you nothing. One level up it is the same story: three nodes in one zone is one zone. Pod topology spread constraints are how you tell the scheduler that the count has a shape.

## Domains, skew, and one worked pod

A constraint names a node label as its `topologyKey`. Every node sharing a value for that label is one domain: `topology.kubernetes.io/zone` gives a domain per zone, `kubernetes.io/hostname` one per node, and any label you set yourself, a rack label on bare metal say, works the same way. The `labelSelector` says which pods to count, normally the workload's own labels. For each candidate node the scheduler counts matching pods in that node's domain, finds the domain with the fewest, and the difference is the skew. `maxSkew` is the most skew a placement may create.

The exact check is in the [scheduler's filter](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/framework/plugins/podtopologyspread/filtering.go): matching pods already in the domain, plus one for the incoming pod, minus the global minimum, must not exceed `maxSkew`. Take three zones holding 2, 2 and 1 matching pods and a sixth replica arriving with `maxSkew: 1`. The global minimum is 1. Zone A would come to 2 + 1 - 1 = 2, over the limit, and zone B the same. Zone C comes to 1 + 1 - 1 = 1 and passes. The pod can go to zone C and nowhere else. If no node in zone C has room, `whenUnsatisfiable` decides what happens: `DoNotSchedule` leaves the pod Pending and `kubectl describe` says `node(s) didn't match pod topology spread constraints`; `ScheduleAnyway` turns the constraint into a score, so zone C is preferred and A or B get the pod if that is what is left.

Two details of that arithmetic come back later. The plus one exists only when the incoming pod's own labels match the `labelSelector`; the filter sets it to 1 on a match and 0 otherwise. Point the selector at a label the pods do not carry and nothing is counted at all, not the incoming pod and not its siblings: every zone computes 0 + 0 - 0 = 0, every zone passes, and the constraint does nothing while looking fine. The subtler version is a template label changed without the selector following it, so the old pods still count and the new one does not, and on the same 2/2/1 cluster 2 - 1 = 1 passes everywhere too. And the global minimum is taken over eligible domains only. `nodeAffinityPolicy` defaults to `Honor`, so nodes your `nodeAffinity` excludes are not domains; `nodeTaintsPolicy` defaults to `Ignore`, so nodes carrying taints you do not tolerate still are. A zone whose nodes all carry such a taint holds zero of your pods, becomes the global minimum, and caps every other zone at `maxSkew` pods. That is a Pending pod that makes no sense until you look at the taints.

## One Deployment, two constraints

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web
          matchLabelKeys:
            - pod-template-hash
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: web
      containers:
        - name: web
          image: nginx
```

Zone is the hard rule and hostname the soft one, and the mix is deliberate. Constraints are ANDed, so the scheduler needs one node that satisfies all of them, and the [docs](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) walk through two hard constraints on a small cluster where the zone rule permits only zone B, the hostname rule permits only a node in zone A, and the pod sits Pending. Hard zone plus soft hostname asks for what you actually mean: never let one zone get more than one pod ahead of the emptiest, and spread across nodes inside the zone when it fits. Six replicas on three zones land 2/2/2.

`matchLabelKeys: [pod-template-hash]` is the line that matters during a rollout. Without it the selector matches the old ReplicaSet's pods too, so the new pods are spread against a layout that is about to disappear, and once the old pods go the new ones can be lopsided. With it, each pod's own `pod-template-hash` is added to the selector and only its own revision is counted. The merge is not the scheduler's doing. The docs say the kube-apiserver reads the listed keys from the incoming pod's labels at creation and merges them into the `labelSelector`, explicitly so since v1.34, which is why a stored pod can show a selector you never wrote. A key may not appear in both fields, and `matchLabelKeys` needs a `labelSelector` to merge into. It has been beta and on by default since v1.27.

## Zones that have no nodes yet

The scheduler can only count domains it can see. If the autoscaler has taken one zone's node group to zero, that zone is not a domain, the minimum is taken over the other two, and the block above is satisfied by a 3/3/0 split it never knew was lopsided. `minDomains` is the fix. Set `minDomains: 3` and, while fewer than three eligible domains exist, the global minimum is treated as 0, so with `maxSkew: 1` each visible zone takes one pod and the third goes Pending. Pending is the point. The Cluster Autoscaler simulates scheduling against each node group, finds that a node in the empty zone's group would take the pod, and brings one up. The [KEP's](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/3022-min-domains-in-pod-topology-spread) motivation is exactly that: force a spread over a minimum number of domains and, when there are not enough, make the autoscaler provision them. Its own example counts nodes rather than zones, but the shape is the same. It needs a node group per zone to work, which the [HA EKS post](/ha-eks/) goes into. `minDomains` only works with `DoNotSchedule`, and has been GA since v1.30.

## Where it quietly does nothing

A node without the `topologyKey` label is bypassed for that constraint: pods already on it are not counted, and the incoming pod cannot be placed there, with `(missing required label)` appended to the Pending reason. Managed clouds set the zone label for you. On bare metal, and on Hetzner, where I have spent some of my time, whether it exists depends on what provisioned the node, and a hard zone constraint on a cluster where no node has the label leaves every replica Pending. `kubectl get nodes -L topology.kubernetes.io/zone` is the first thing to run; a blank column means label the nodes before writing the constraint.

The constraint is checked when the pod is scheduled and never again. Drain a node and its pods are rescheduled onto whatever satisfies the constraint at that moment; uncordon it and nothing moves back. Scale a Deployment down and the ReplicaSet controller picks victims without consulting your constraints, so 3/3/3 can become 3/3/0. The [descheduler's](https://github.com/kubernetes-sigs/descheduler) `RemovePodsViolatingTopologySpreadConstraint` plugin evicts the fewest pods needed to bring each domain back under `maxSkew`; by default it considers hard constraints only, and its `constraints` parameter has to list `ScheduleAnyway` before it touches soft ones.

A hard zone rule and a zonal disk can deadlock. With `volumeBindingMode: Immediate` the PVC is provisioned, in some zone, before the pod is scheduled; if the spread rule then wants the pod elsewhere, the pod is Pending for good. `WaitForFirstConsumer` on the StorageClass delays provisioning until the pod has a node, so the disk is created where the spread decided. For a StatefulSet that first placement is permanent, because the volume pins the pod to its zone on every restart after that.

Last, you already have a weak version of all this. The scheduler ships soft defaults, `maxSkew: 3` by hostname and `maxSkew: 5` by zone, both `ScheduleAnyway`, for pods that define no constraints and belong to a Service, ReplicaSet, StatefulSet or ReplicationController. That is a preference weighed against resource scores, not a rule, which is why the situation in the title is likely rather than impossible.

## Spread or anti-affinity

Pod anti-affinity is binary. `requiredDuringSchedulingIgnoredDuringExecution` on the zone key means one pod per zone, so three zones cap you at three replicas and the fourth is Pending forever; the `preferred` form stops guaranteeing anything. Spread constraints have a dial. Nine replicas across three zones with `maxSkew: 1` is 3/3/3, and losing a zone leaves two thirds running. Keep anti-affinity for the rule it is good at, which is "these two things never share a node". For keeping replicas of one thing apart, use spread.

## Ending notes

The block I would start from is the one above: a hard zone rule at `maxSkew: 1`, a soft hostname rule, `matchLabelKeys: [pod-template-hash]`, and a PodDisruptionBudget next to it. Spread decides where pods land. The PDB decides how many may be taken away at once by a drain or an upgrade, which spread knows nothing about; [the PDB post](/pdb/) covers that half. Either without the other leaves a gap.

To see what the spread actually is, count pods per zone:

```bash
kubectl get pods -l app=web -o json | jq -r '.items[].spec.nodeName' \
  | xargs -I{} kubectl get node {} -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}' \
  | sort | uniq -c
```

If the numbers are off by more than `maxSkew`, the constraint was never satisfied, a node is missing the label, or something moved after scheduling and nobody asked the scheduler again.

Thanks for your time :)
