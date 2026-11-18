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

I read up on this for an interview question, "how would you handle a flash sale on Kubernetes", and the answer turned out to be simpler than I expected: buy the nodes before the sale, and keep them occupied with pods that do nothing until something real needs the space. The timeline below is illustrative, with a three-minute node for the sake of a number, but the shape is the whole argument.

| | Without a buffer | With a buffer |
| :-- | :-- | :-- |
| t+0 | traffic spikes | traffic spikes |
| t+15 s | HPA adds pods, they go `Pending` | HPA adds pods, they preempt the placeholders and are `Running` |
| t+15 s to t+3 min | a node is requested, boots, joins, pulls images; errors the whole time | the same node boot happens, for the evicted placeholders; nobody is waiting on it |
| t+3 min | pods schedule, errors stop | placeholders land on the new node, buffer restored |

## Pods scale in seconds, nodes in minutes

The chain is worth spelling out once, because every fix later in the post attacks one link of it. The HPA recomputes its replica count every loop, and its default scale-up behaviour adds up to four pods or 100% of the current count every 15 seconds, whichever is more. Those pods reach the scheduler, which finds no node whose free requests can hold them, and marks them unschedulable. Cluster Autoscaler notices, picks a node group that would fit them and asks the cloud for an instance; Karpenter does the same with an instance it sizes for the pending pods. Then the part no controller can shorten: the instance boots, the kubelet joins, the node turns `Ready`, and the image is pulled onto a disk that has never seen it. The chat I was learning from put that at two to five minutes; an [AWS walkthrough](https://aws.amazon.com/blogs/containers/eliminate-kubernetes-node-scaling-lag-with-pod-priority-and-over-provisioning) of the same pattern with Karpenter calls it approximately one to two. Either way, autoscaling is working as designed, and the design has a floor.

## Pause pods as a reservation

A placeholder is a pod that requests real CPU and memory and runs `registry.k8s.io/pause`, the binary Kubernetes already uses as the sandbox of every pod. Its [source](https://github.com/kubernetes/kubernetes/blob/master/build/pause/linux/pause.c) is short: install handlers for SIGTERM and SIGCHLD, then call `pause()` forever. It uses nothing. But the scheduler places pods by requests, not usage, a point [the node-eviction post](/node-eviction/) spends some time on, so the request is held as firmly as a busy pod would hold it. Put the placeholders under a PriorityClass with a negative value and they rank below every pod that names no class, since those sit at 0.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -1
globalDefault: false
description: "Placeholder capacity, preempted first"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
  namespace: overprovisioning
spec:
  replicas: 5
  selector:
    matchLabels:
      app: overprovisioning
  template:
    metadata:
      labels:
        app: overprovisioning
    spec:
      priorityClassName: overprovisioning
      terminationGracePeriodSeconds: 0
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.10
        resources:
          requests:
            cpu: "1"
            memory: 2Gi
```

Now the spike. The HPA creates app pods and the scheduler finds nowhere for them, so the [preemption logic](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) runs: find a node where removing pods of lower priority than the pending pod would make it fit, evict them, and record the node in the pending pod's `nominatedNodeName`. The placeholders are the lowest-priority pods in the cluster, so they are always the victims. The app pods start on nodes that are already warm and, if the app has run there before, already have the image. The placeholder Deployment replaces its evicted replicas, those go `Pending`, and the autoscaler scales up for them; the [Cluster Autoscaler FAQ](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md) describes the pattern in exactly those two steps. The node boots, the placeholders land on it, and the buffer is back for the next wave. The boot still took its minutes. It happened behind pods nobody was waiting for.

To watch it, scale the app Deployment past what the cluster holds and run `kubectl get pods -w` across both namespaces: placeholders go `Terminating`, app pods go `Running` in the same second, replacement placeholders appear `Pending`, and a minute or more later `kubectl get nodes -w` shows the node they were waiting for.

## The priority value that breaks it

Cluster Autoscaler has a flag, `--expendable-pods-priority-cutoff`, and the FAQ defines it in one sentence: pods with priority below the cutoff are expendable. Expendable pods do not trigger scale-ups, no node is added to run them, and they do not block scale-downs either. The default is -10, and the FAQ adds that nothing changes for pods with priority greater than or equal to the cutoff. So a placeholder at -100 is preempted on the first wave, goes `Pending`, and stays there. No node is requested for it, the buffer never comes back, and the second wave meets the cluster you had before you started.

The official examples do not agree with each other, which is why I wanted the flag's exact semantics. The Cluster Autoscaler FAQ's own overprovisioning example uses -10 and calls it the lowest priority that still triggers scaling. The [kubernetes.io task page](https://kubernetes.io/docs/tasks/administer-cluster/node-overprovisioning/) uses -1000 and is written for any node autoscaler. Pair that manifest with Cluster Autoscaler at default flags and your placeholders are expendable. The manifest above uses -1, which is also what the AWS walkthrough uses with Karpenter: safely above the cutoff, below every ordinary pod. Anything from -9 to -1 behaves the same; -10 works too, with no margin.

## Grace period, shape, pool, zone, namespace

The preemption docs say victims get their graceful termination period, that this opens a gap between the preemption and the pending pod actually being scheduled, and that the fix is to set the grace period of low-priority pods to zero or a small number. The pause binary handles SIGTERM and exits at once, so in practice the wait is short rather than the full 30 seconds [the 30-second pod post](/pod-30-seconds/) is about. `terminationGracePeriodSeconds: 0` takes it out of the path entirely, and the Cluster Autoscaler FAQ's example sets it.

Shape matters more than count. Preemption is decided per node: the docs' test is whether the pending pod could be scheduled if every lower-priority pod on that node were removed. Five placeholders requesting 0.5 CPU each, spread across five nodes, free five half-cores, and a pod that requests 2 CPU fits in none of them. Size each placeholder like the largest pod the HPA will add, or larger, and prefer fewer big placeholders to many small ones. The kubernetes.io example spreads them with a preferred anti-affinity on hostname, and the same page notes that Karpenter treats preferred affinity rules as hard rules when deciding whether to launch nodes, so under Karpenter the replica count also becomes a minimum node count. Decide whether you want the placeholders spread or packed, rather than inheriting it.

Pool matters as much. The placeholders need the same `nodeSelector`, affinity and tolerations as the workload they stand in for. Headroom on the general pool does nothing for a checkout service pinned to a tainted pool, and the money is spent either way.

Zone matters for the reason it does for the app. Without a `topologySpreadConstraints` entry on `topology.kubernetes.io/zone`, all five placeholders can land in one AZ while the app spreads across three; the block in [the HA EKS post](/ha-eks/) is the one to copy.

And give them their own namespace. Their requests count against the ResourceQuota of whichever namespace they live in, and a team discovering that a pod of nothing has eaten their CPU budget is a conversation you can skip.

## How much to hold

The rule of thumb: the buffer should cover the ramp for as long as a node takes to arrive, so pods added per minute during the ramp, times node-ready time in minutes, in placeholder-shaped slots. If the HPA adds about 20 pods a minute and a node takes about 3 minutes to become `Ready`, that is roughly 60 pods of headroom. Less than that and pods go `Pending` partway through the ramp, just later than they would have. This is arithmetic, not a result from anywhere; treat it as a starting point. The HPA's `scaleUp` policies cap the first number and a load test tells you the real one. The second is on every node object: `metadata.creationTimestamp` against the `lastTransitionTime` of its `Ready` condition.
