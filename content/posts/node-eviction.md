---
title: "Who Dies First When a Kubernetes Node Runs Out of Memory? And How to Control That"
description: "The kubelet does not evict by QoS class. It sorts pods by usage over request, then priority, then how far over, and the only inputs you own are the pod's own requests and its priorityClassName. What requests and limits do, where QoS actually matters (the kernel OOM killer), what the eviction thresholds configure and what they cannot, and why Evicted and OOMKilled are two different programs killing for two different reasons."
dateString: November 2026
draft: false
tags: ["Kubernetes", "SRE", "DevOps", "Scheduling", "Resource Management", "Platform Engineering"]
weight: 2
cover:
    image: "/blog/node-eviction/node-eviction.png"
---


`kubectl get pods` has two words for a pod that died over memory, and for a long time I read them as the same thing. `Evicted` and `OOMKilled` come from different programs, for different reasons, and leave different wreckage: one moves the pod to another node, the other restarts a container where it stands. The question I actually wanted answered sits underneath both. When a node runs out of memory, which pod goes first, and what in my manifests decides it?

The answer I carried around was "BestEffort, then Burstable, then Guaranteed". That is roughly what happens and not what the kubelet computes. The kubelet sorts by three keys, none of which is the QoS class, and the only inputs you own are the pod's `requests` and its `priorityClassName`. You cannot tell the kubelet who to pick. You can tell it when to start, and you can shape the inputs. This post is the short version of that.

## Everything starts with the resources block

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
    ephemeral-storage: 1Gi
  limits:
    cpu: "1"
    memory: 512Mi
    ephemeral-storage: 2Gi
```

A request is a number on paper. The scheduler places a pod on a node only if the requests already there plus this one fit in allocatable, and usage never enters that sum. At runtime the three requests do three different jobs. The CPU request becomes a cgroup weight, a share of the node when it is busy, which is what [the millicores post](/millicores/) covers. The memory request is enforced by nothing; the kernel does not know it exists. It is the number the kubelet compares usage against when it ranks pods for eviction, and the number behind the container's OOM score, and that is the whole reason it matters here. The ephemeral-storage request only affects scheduling.

A limit is a ceiling, and the kernel enforces the three of them differently. CPU over its limit is throttled by CFS quota: slowed down, never killed. Memory over its limit is `memory.max` on the container's cgroup (`memory.limit_in_bytes` on v1), and the kernel kills the container: `Reason: OOMKilled`, exit code 137, a restart in place, however much memory the node has free. Ephemeral storage over its limit (writable layer plus logs plus `emptyDir`) gets the whole pod evicted by the kubelet.

Three rules at the edges. A limit without a request copies the limit into the request. A request larger than its limit is rejected by the API server. And a node can be overcommitted on limits but never on requests, so memory pressure is what happens when the pods already there use more than they asked for, between them. A pod's effective request is the larger of the sum of its app containers and its biggest init container. Pod-level `spec.resources` (beta since 1.34) and in-place resize (stable since 1.35) exist too, if the cluster is new enough.

The unit foot-gun is memory with a lowercase `m`. The docs are blunt about it: `400m` of memory is a request for 0.4 bytes. `Mi` and `Gi` are what you meant.

The defaults that make everything below simplest: memory request equal to limit, so the pod never overshoots its request and the OOM point is a known number; a realistic CPU request with no limit or a generous one; and both numbers taken from observed usage, not from the last manifest you copied.

## QoS is derived, and it is not the eviction order

You cannot write `qosClass` in a manifest. Kubernetes derives it from the requests and limits of every container, init containers included, and shows the result in `status.qosClass`.

| Class | Rule |
| :-- | :-- |
| Guaranteed | every container has CPU and memory requests and limits, and each request equals its limit |
| Burstable | not Guaranteed, and at least one container has any CPU or memory request or limit |
| BestEffort | no CPU or memory request or limit on any container |

Two gotchas follow from the derivation. Setting only limits gives you Guaranteed, because the requests are copied from them. One sidecar with no limits drops the whole pod to Burstable.

```bash
$ kubectl get pod api-7d4c9-x2k1p -o jsonpath='{.status.qosClass}'
Burstable
```

The class changes three things. Guaranteed pods with whole-number CPU requests can get exclusive cores under the CPU Manager's static policy. BestEffort pods are not scheduled onto a node carrying the `MemoryPressure` condition, because the control plane adds a toleration for that taint to every pod that is not BestEffort. And the kubelet writes `oom_score_adj` on every container from the class, which is the one place QoS decides who dies:

| QoS | `oom_score_adj` |
| :-- | :-- |
| Guaranteed | -997 |
| Burstable | min(max(2, 1000 - 1000 × memoryRequest / nodeMemoryCapacity), 999) |
| BestEffort | 1000 |

A Burstable container requesting a tenth of the node scores about 900; one requesting half scores about 500. Bigger request, safer container. The code in `pkg/kubelet/qos/policy.go` also keeps every Burstable score above anything a Guaranteed container can reach, and gives any container in a `system-node-critical` pod the Guaranteed -997 regardless of class. That score is read by the kernel OOM killer, not by the kubelet, and those are two separate events. Hold that thought until the ranking.

## PriorityClass is the tiebreaker, in two places

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Payment and checkout services"
```
