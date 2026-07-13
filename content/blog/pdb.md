---
title: "Pod Disruption Budgets Explained: How Kubernetes Protects You From Voluntary Disruptions"
description: "What a PodDisruptionBudget actually does, the difference between voluntary and involuntary disruptions, how the eviction API respects it, and the mistakes that quietly break node drains and cluster upgrades."
dateString: July 2026
draft: false
tags: ["Kubernetes", "DevOps", "Reliability"]
weight: 2
cover:
    image: "/blog/pdb/pdb.png"
---

If you've spent time debugging a `kubectl drain` that hangs forever, or a cluster autoscaler that refuses to scale a node down even though it should be safe to, there's a decent chance a PodDisruptionBudget (PDB) was involved. It's one of those Kubernetes objects that's simple on paper but easy to reason about incorrectly, because it only kicks in for a specific category of disruption, not all of them. Understanding that distinction is really the whole trick to using PDBs well.

## First, what counts as a "disruption"?

Kubernetes splits pod disruptions into two categories, and PDBs only care about one of them.

**Involuntary disruptions** are things that happen *to* your pod, outside your control: the node it's running on crashes, a kernel panic takes the machine down, the underlying VM is preempted by the cloud provider, there's a network partition. Kubernetes can't prevent these, and a PDB has no power over them either. This is what redundancy (multiple replicas, spread across nodes/zones) is for - not PDBs.

**Voluntary disruptions** are things that happen *because* someone or something deliberately asked for a pod to be evicted, as part of a normal, healthy cluster operation:

- Draining a node for maintenance (`kubectl drain`)
- The cluster autoscaler removing an underutilized node
- A rolling update to a Deployment or DaemonSet
- Someone manually deleting a pod
- Cordoning and evicting pods off a node before decommissioning it

This is the category PDBs govern. A PDB is essentially a promise you make to the cluster: "no matter what voluntary disruption you're trying to perform, don't let the number of healthy pods for this workload drop below X (or by more than Y) at any given moment."

## The actual object

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

Two fields do the real work:

- **`minAvailable`** - the minimum number (or percentage, e.g. `"50%"`) of matching pods that must remain healthy at all times.
- **`maxUnavailable`** - the maximum number (or percentage) of matching pods that are allowed to be unhealthy at once.

You set one or the other, not both - they're two ways of expressing the same constraint from opposite directions. If you're running 3 replicas and want at least 2 up at any time, `minAvailable: 2` and `maxUnavailable: 1` mean exactly the same thing here. Percentages are usually the better choice for workloads whose replica count changes over time (via an HPA, for instance), since a fixed integer stops making sense the moment you scale.

## How it's actually enforced

PDB doesn't do anything by itself. It's enforced through the **Eviction API** - the same API that `kubectl drain` and the cluster autoscaler use under the hood to remove pods gracefully. When something calls the eviction endpoint for a pod, the API server checks whether evicting that pod would violate any PDB matching it. If it would, the eviction request gets rejected with a 429, and the caller is expected to back off and retry later - usually after the workload has recovered elsewhere.

That "enforced through eviction" detail matters, because it means a PDB does **not** protect against:

- `kubectl delete pod` - a direct delete bypasses the eviction API entirely, so PDBs are silently ignored.
- A node dying outright - there's no eviction call involved, the pod is just gone.
- OOMKills, crash loops, liveness probe failures - none of these go through eviction.

A PDB only guards the door that voluntary, cooperative disruption tools walk through. It's not a general-purpose availability guarantee - it's specifically a contract with anything using the eviction API.

## Where this becomes visible in practice

**Node drains.** When you run `kubectl drain node-x`, the drain command tries to evict every pod on that node one at a time. If evicting a pod would violate its PDB, the eviction is refused, and `drain` will sit there retrying (or eventually time out, depending on your flags) until either the workload has enough healthy replicas elsewhere or you intervene manually. It stops you from accidentally draining a node in a way that takes your application below its minimum safe capacity. But it becomes a real problem when the PDB is too strict for the workload's actual replica count.

**Cluster autoscaler scale-downs.** The autoscaler will not remove a node if doing so would violate a PDB for any pod on it. If your PDBs are overly conservative across the board, you can end up with a cluster that simply never scales down, because there's always *some* pod somewhere blocking eviction.

**Rolling updates.** Ordinary Deployment rollouts are governed by the Deployment's own `maxSurge`/`maxUnavailable`, not by the PDB directly - but the PDB still applies underneath, since pod termination during a rollout also goes through normal deletion, and any voluntary eviction-based action layered on top (like a concurrent drain) still has to respect it.

## The classic footgun

The most common way people break their own cluster with PDBs is this:

```yaml
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: single-replica-app
```

...applied to a workload that only ever runs **one replica**. `minAvailable: 1` on a 1-replica app means: "you may never voluntarily evict this pod, under any circumstances, because doing so would drop availability to zero." Kubernetes takes that literally. Any node this pod lands on becomes un-drainable through normal means - `kubectl drain` will hang on it indefinitely, and the cluster autoscaler will refuse to remove that node, forever. You've accidentally pinned a node to the cluster.

A gentler version of the same mistake: setting `maxUnavailable: 0` on a workload with tight resource requests in a cluster with little slack capacity. The PDB says the new pod must be up before the old one goes anywhere, but if there's no room to schedule that extra pod, the drain stalls waiting for capacity that never arrives.

For most stateless, horizontally-scaled workloads, something like this is a reasonable starting point:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

This says "you can take down one pod at a time for maintenance," which lets drains and autoscaler scale-downs proceed steadily without ever leaving the app fully down, as long as you have at least 2 replicas. For workloads where replica count scales dynamically, prefer a percentage:

```yaml
spec:
  maxUnavailable: "25%"
```
