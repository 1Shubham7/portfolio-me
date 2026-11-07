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

The answer I would have given was "BestEffort, then Burstable, then Guaranteed". That is roughly what happens and not what the kubelet computes. The kubelet sorts by three keys, none of which is the QoS class. You cannot tell the kubelet who to pick. You can tell it when to start, and you can shape the inputs. This post is the short version of that.

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

A request is a number on paper. The scheduler places a pod on a node only if the requests already there plus this one fit in allocatable, and usage never enters that sum. At runtime the three requests do three different jobs. The CPU request becomes a cgroup weight, a share of the node when it is busy, which is what [the millicores post](/millicores/) covers. The memory request is enforced by nothing; the kernel does not know it exists. It is the number the kubelet compares usage against when it ranks pods for eviction, and the number behind the container's OOM score, and that is the whole reason it matters here. The ephemeral-storage request does two of those three jobs for disk: scheduling, and the line the kubelet ranks disk usage against under disk pressure.

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

Two gotchas follow from the derivation. Setting only limits, CPU and memory both, gives you Guaranteed, because the requests are copied from them. One sidecar with no limits drops the whole pod to Burstable.

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

A Burstable container requesting a tenth of the node scores about 900; one requesting half scores about 500. Bigger request, safer container. That formula is the docs' version. The code in `pkg/kubelet/qos/policy.go` floors it at 3 rather than 2, so a Burstable score cannot fall below what a Guaranteed container using the whole node reaches, and gives any container in a `system-node-critical` pod the Guaranteed -997 regardless of class. That score is read by the kernel OOM killer, not by the kubelet, and those are two separate events. Hold that thought until the ranking.

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

`value` is any 32-bit integer up to 1,000,000,000. Above that is reserved for the two built-ins, `system-cluster-critical` at 2000000000 and `system-node-critical` at 2000001000. A pod names its class with `spec.priorityClassName: high-priority`, and a pod that names nothing gets 0 unless one class in the cluster has `globalDefault: true`. `preemptionPolicy: Never` gives a pod its place in the scheduling queue without letting it push anyone out, which is the right setting for batch work that should wait rather than kill.

Priority is read by the scheduler and by the kubelet, and the two behave differently enough to keep apart:

| | Preemption | Node-pressure eviction |
| :-- | :-- | :-- |
| Who | kube-scheduler | kubelet |
| Trigger | a higher-priority pod is Pending and nothing fits | the node is low on memory, disk or PIDs |
| Respects PDBs | best effort | no |
| Grace period | the pod's own, 30 s by default | soft threshold: up to `evictionMaxPodGracePeriod`; hard threshold: 0 s |

## Quota and LimitRange, since they come up in the same breath

ResourceQuota is a namespace budget checked at admission. It rejects new objects and never touches a running pod. Once a quota names `requests.memory` or `limits.cpu`, every new pod in the namespace has to declare them or is refused, which is why quotas ship with a LimitRange that injects `default` and `defaultRequest`. The one quota feature that belongs in this post is the `PriorityClass` scope: `pods: "5"` scoped to `high-priority` caps how many pods a namespace may mark important, which is where the priority docs point when a tenant cannot be trusted not to mark everything.

## What the kubelet watches, and when it starts

CPU never triggers eviction. It is compressible: a contended node slows everyone down in proportion to their weights, and there is no CPU signal in the list. Memory, disk and PIDs are not compressible, and those have signals: `memory.available`, `nodefs.available`, `nodefs.inodesFree`, `imagefs.available`, `imagefs.inodesFree`, `containerfs.available`, `containerfs.inodesFree` and `pid.available`. The list is fixed.

Thresholds live in the KubeletConfiguration, per node. The `evictionHard` block below is the Linux default; the rest are example values for the knobs around it:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
  imagefs.inodesFree: "5%"
evictionSoft:
  memory.available: "500Mi"
evictionSoftGracePeriod:
  memory.available: "1m30s"
evictionMaxPodGracePeriod: 60
evictionMinimumReclaim:
  memory.available: "500Mi"
evictionPressureTransitionPeriod: 5m
kubeReserved:
  memory: 1Gi
systemReserved:
  memory: 1Gi
```

A hard threshold evicts the moment it is crossed, with a 0 s grace period. A soft one has to stay crossed for its grace period first, and then pods get up to `evictionMaxPodGracePeriod` to leave. `evictionMinimumReclaim` makes the kubelet keep going past the threshold so it is not back here ten seconds later. `evictionPressureTransitionPeriod` is how long the node must look healthy before the condition clears, 5m by default. The two reservations are what the daemons are promised, and they matter more than they look: Allocatable = Capacity - kubeReserved - systemReserved - evictionHard, and pods are scheduled against Allocatable, so the eviction headroom is carved out before any pod lands.

The kubelet checks the signals every housekeeping interval, 10 s by default. When a threshold is crossed the node gets a condition, and the control plane turns it into a taint:

```
$ kubectl describe node worker-1
...
Taints:             node.kubernetes.io/memory-pressure:NoSchedule
...
Conditions:
  Type             Status  ...  Reason                        Message
  MemoryPressure   True    ...  KubeletHasInsufficientMemory  kubelet has insufficient memory available
```

## The ranking

Under memory pressure the kubelet sorts every pod on the node by three keys and evicts from the top. `pkg/kubelet/eviction/helpers.go` does it in one line, `orderedBy(exceedMemoryRequests(stats), priority, memory(stats))`:

1. Is usage above the request? Pods over their request go first.
2. Priority. Lower goes first.
3. How far over the request. Bigger overshoot goes first.

QoS is nowhere in that list, and it still comes out of it. A BestEffort pod has a request of zero, so any usage at all puts it in the first group. A Burstable pod over its request is in the same group. A Guaranteed pod cannot stay over its request, because request equals limit and the limit is enforced, so it sits in the safe group with every Burstable pod that is under its request. The docs promise that Guaranteed pods will never be evicted because of another pod's consumption, and the same holds for any pod under its request. They are only reached when the system daemons eat past `kubeReserved` and `systemReserved` and the node has nothing else left to take, and then it goes by priority.

The node is under memory pressure and has four pods:

| Pod | QoS | Priority | Request | Usage |
| :-- | :-- | :-- | :-- | :-- |
| A | BestEffort | 0 | 0 | 300Mi |
| B | Burstable | 1000 | 512Mi | 900Mi |
| C | Burstable | 0 | 512Mi | 400Mi |
| D | Guaranteed | 0 | 1Gi | 1Gi |

Key one splits them: A and B are over their requests, C and D are not. Key two orders the first group: A has priority 0 and B has 1000, so A goes, then B if that was not enough. C and D are untouched unless the daemons are the problem. Now give A `priorityClassName: high-priority` at 1000000 and leave B at 1000. B goes first, a Burstable pod before a BestEffort one. The rule of thumb survives only because so few pods ever get a priority.

Afterwards B looks like this in `kubectl describe pod`, with the message assembled from the format strings in that same file:

```
Status:   Failed
Reason:   Evicted
Message:  The node was low on resource: memory. Threshold quantity: 100Mi, available: 60Mi. Container app was using 900Mi, request is 512Mi, has larger consumption of memory.
```

Disk pressure is the same shape with a different first move. The kubelet first reclaims what it owns, garbage-collecting dead pods and containers and then deleting unused images, and only then ranks pods: over their ephemeral-storage request first, then priority, then disk usage. [The deleted-file post](/deleted-file-disk/) has the part where a deleted-but-open log counts fully against `nodefs.available`.

## Evicted is not OOMKilled

The kubelet looks every 10 s. A process that grabs a few gigabytes inside that window goes straight through the threshold, and the kernel OOM killer acts first. A container over its own memory limit meets the same killer by a different route, its own cgroup's `memory.max`, whatever the node is doing. Either way the kernel kills a process, which on a modern node takes the container with it, and the kubelet only finds out afterwards.

| | Evicted | OOMKilled |
| :-- | :-- | :-- |
| Who kills | kubelet | kernel |
| Why | the node crossed an eviction signal | node OOM before the kubelet moved, or the container went over its own limit |
| Picked by | the three-key sort above | `oom_score_adj`, so QoS and the size of the request |
| Result | pod `Failed`, reason `Evicted`, replaced elsewhere | container restarts in place, exit code 137 |

Neither respects a PodDisruptionBudget. A PDB guards the Eviction API, which `kubectl drain`, the cluster autoscaler and the Descheduler go through and node-pressure eviction does not; [the PDB post](/pdb/) is about that boundary.

## What you can and cannot configure

You configure when: thresholds, soft versus hard, grace periods, minimum reclaim, reservations, all per node. You do not configure who. The signal list is fixed, the ranking is hardcoded, and there is no setting for "evict pods with this label first" or "on disk pressure only take the log shippers". What you have is the inputs to the sort, and those are in the pod spec:

| Goal | Lever |
| :-- | :-- |
| Evicted last | high `priorityClassName`, memory request at real usage, ideally request = limit |
| Evicted first | no requests (BestEffort), priority 0 |
| Survive the kernel OOM killer | bigger memory request, or Guaranteed |
| Avoid disk-pressure eviction | `ephemeral-storage` request and limit, logs to stdout rather than `emptyDir` |

If the policy you want is not expressible that way, it lives outside the kubelet: the Descheduler for utilisation or pod-age policies, or a controller of your own on the Eviction API. Both go through the API, so both respect PDBs, which the kubelet never will.

## Ending notes

The kubelet does not read the QoS class when it evicts. It sorts by usage over request, then priority, then how far over, and all three come from the pod spec. The kernel OOM killer does read the class, through `oom_score_adj`, and it only gets involved when the kubelet is too slow or a container trips its own limit.

So the checklist for a service that should die last is short. Memory request equal to limit, so it is Guaranteed and never in the first group. A PriorityClass with a real value, so it loses the tiebreak to nobody. And `kubeReserved` and `systemReserved` set honestly on the nodes, so the daemons never make the kubelet reach into the safe group at all.

Thanks for your time :)
