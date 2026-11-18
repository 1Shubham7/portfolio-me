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

The answer turned out to be simpler than I expected when I started reading: buy the nodes before the sale, and keep them occupied with pods that do nothing until something real needs the space. The timeline below is illustrative, with a three-minute node for the sake of a number, but the shape is the whole argument.

| | Without a buffer | With a buffer |
| :-- | :-- | :-- |
| t+0 | traffic spikes | traffic spikes |
| t+15 s | HPA adds pods, they go `Pending` | HPA adds pods, they preempt the placeholders and are `Running` |
| t+15 s to t+3 min | a node is requested, boots, joins, pulls images; errors the whole time | the same node boot happens, for the evicted placeholders; nobody is waiting on it |
| t+3 min | pods schedule, errors stop | placeholders land on the new node, buffer restored |

## Pods scale in seconds, nodes in minutes

The chain is worth spelling out once, because every fix later in the post attacks one link of it. The HPA recomputes its replica count every loop, and its default scale-up behaviour adds up to four pods or 100% of the current count every 15 seconds, whichever is more. Those pods reach the scheduler, which finds no node whose free requests can hold them, and marks them unschedulable. Cluster Autoscaler notices, picks a node group that would fit them and asks the cloud for an instance; Karpenter does the same with an instance it sizes for the pending pods. Then the part no controller can shorten: the instance boots, the kubelet joins, the node turns `Ready`, and the image is pulled onto a disk that has never seen it. Call it two to five minutes; an [AWS walkthrough](https://aws.amazon.com/blogs/containers/eliminate-kubernetes-node-scaling-lag-with-pod-priority-and-over-provisioning) of the same pattern with Karpenter puts it at approximately one to two. Either way, autoscaling is working as designed, and the design has a floor.

## Pause pods as a reservation

A placeholder is a pod that requests real CPU and memory and runs `registry.k8s.io/pause`, the binary Kubernetes already uses as the sandbox of every pod. Its [source](https://github.com/kubernetes/kubernetes/blob/master/build/pause/linux/pause.c) is short: install handlers for SIGINT, SIGTERM and SIGCHLD, then call `pause()` forever. It uses nothing. But the scheduler places pods by requests, not usage, a point [the node-eviction post](/node-eviction/) spends some time on, so the request is held as firmly as a busy pod would hold it. Put the placeholders under a PriorityClass with a negative value and they rank below every pod that names no class, since those sit at 0.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: overprovisioning
---
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

Now the spike. The HPA creates app pods and the scheduler finds nowhere for them, so the [preemption logic](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) runs: find a node where removing pods of lower priority than the pending pod would make it fit, evict them, and record the node in the pending pod's `nominatedNodeName`. The placeholders rank below everything else in the cluster, so they are the first victims. The app pods start on nodes that are already warm and, if the app has run there before, already have the image. The placeholder Deployment replaces its evicted replicas, those go `Pending`, and the autoscaler scales up for them; the [Cluster Autoscaler FAQ](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md) describes the pattern in exactly those two steps. The node boots, the placeholders land on it, and the buffer is back for the next wave.

To watch it, scale the app Deployment past what the cluster holds and run `kubectl get pods -w -A`: placeholders go `Terminating`, app pods go `Running` within a few seconds, replacement placeholders appear `Pending`, and a minute or more later `kubectl get nodes -w` shows the node they were waiting for.

## The priority value that breaks it

Cluster Autoscaler has a flag, `--expendable-pods-priority-cutoff`, and the FAQ defines it in one sentence: pods with priority below the cutoff are expendable. Expendable pods do not trigger scale-ups, no node is added to run them, and they do not block scale-downs either. The default is -10, and the FAQ adds that nothing changes for pods with priority greater than or equal to the cutoff. So a placeholder at -100 is preempted on the first wave, goes `Pending`, and stays there. No node is requested for it, the buffer never comes back, and the second wave meets the cluster you had before you started.

The official examples do not agree with each other, which is why I wanted the flag's exact semantics. The Cluster Autoscaler FAQ's own overprovisioning example uses -10 and calls it the lowest priority that still triggers scaling. The [kubernetes.io task page](https://kubernetes.io/docs/tasks/administer-cluster/node-overprovisioning/) uses -1000 and is written for any node autoscaler. Pair that manifest with Cluster Autoscaler at default flags and your placeholders are expendable. The manifest above uses -1, which is also what the AWS walkthrough uses with Karpenter: safely above the cutoff, below every ordinary pod. Anything from -9 to -1 behaves the same; -10 works too, with no margin.

## What the manifest leaves out

The preemption docs say victims get their graceful termination period, that this opens a gap between the preemption and the pending pod actually being scheduled, and that the fix is to set the grace period of low-priority pods to zero or a small number. The pause binary handles SIGTERM and exits at once, so in practice the wait is short rather than the full 30 seconds [the 30-second pod post](/pod-30-seconds/) is about. `terminationGracePeriodSeconds: 0` takes it out of the path entirely, and the Cluster Autoscaler FAQ's example sets it.

The manifest sizes each placeholder at 1 CPU and 2Gi, and that number does more work than the replica count, because preemption is decided per node: the docs' test is whether the pending pod could be scheduled if every lower-priority pod on that node were removed. Five placeholders requesting 0.5 CPU each, spread across five nodes, free five half-cores, and a pod that requests 2 CPU fits in none of them. Size each placeholder like the largest pod the HPA will add, or larger, and prefer fewer big placeholders to many small ones.

Headroom in the wrong place is the same money spent for nothing. The placeholders need the same `nodeSelector`, affinity and tolerations as the workload they stand in for, since a buffer on the general pool does nothing for a checkout service pinned to a tainted pool. Zones are the same problem one level up: without a `topologySpreadConstraints` entry on `topology.kubernetes.io/zone`, all five placeholders can land in one AZ while the app spreads across three, and the block in [the HA EKS post](/ha-eks/) is the one to copy.

The kubernetes.io example also spreads them across nodes with a preferred anti-affinity on hostname, and the same page notes that Karpenter treats preferred affinity rules as hard rules when deciding whether to launch nodes, so under Karpenter the replica count becomes a minimum node count as well. I would leave that part out. Spread across zones with the constraint above, packed within a zone, is the arrangement that keeps the slots pod-shaped; the hostname anti-affinity pulls the other way and, on Karpenter, costs nodes to do it.

Their own namespace, which the manifest creates, is not cosmetic either. Their requests count against the ResourceQuota of whichever namespace they live in, and a team discovering that a pod of nothing has eaten their CPU budget is a conversation you can skip.

## How much to hold

The rule of thumb: the buffer should cover the ramp for as long as a node takes to arrive, so pods added per minute during the ramp, times node-ready time in minutes, in placeholder-shaped slots. If the HPA adds about 20 pods a minute and a node takes about 3 minutes to become `Ready`, that is roughly 60 pods of headroom. Less than that and pods go `Pending` partway through the ramp, just later than they would have. This is arithmetic, not a result from anywhere; treat it as a starting point. The HPA's `scaleUp` policies cap the first number and a load test tells you the real one. The second is on every node object: `metadata.creationTimestamp` against the `lastTransitionTime` of its `Ready` condition.

A fixed replica count drifts as the cluster grows, so the Cluster Autoscaler FAQ wires the placeholder Deployment to the [cluster-proportional-autoscaler](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler), which watches the number of schedulable nodes and cores and resizes a target. In linear mode `replicas = max(ceil(cores / coresPerReplica), ceil(nodes / nodesPerReplica))`, clamped by `min` and `max`. With 1 CPU placeholders, `coresPerReplica: 10` holds about 10% of the cluster's cores in reserve, and the number tracks the cluster instead of someone's memory of it.

## Sale days are scheduled, so schedule the buffer

Reactive scaling is for spikes you did not see coming. A sale has a start time. Bump the placeholder replicas before it and drop them after, with a CronJob that runs `kubectl scale`, or with the [KEDA cron scaler](https://keda.sh/docs/latest/scalers/cron/) on the placeholder Deployment: a `timezone`, a `start` and `end` cron expression, and a `desiredReplicas` to hold between them. When other triggers sit on the same ScaledObject, `desiredReplicas` acts as a floor, because the HPA takes the maximum across metrics. The same schedule applies to the app's own `minReplicas`: raise it for the window so the first wave is already absorbed.

Scaling back down afterwards is what keeps this affordable. The five placeholders in the manifest hold 5 CPU and 10Gi of nothing, on nodes billed whether or not anything runs on them; a sale-day buffer of 60 slots at the same shape holds 60 CPU and 120Gi. That is defensible for a few hours around a spike you can name, and not for the rest of the year, which is what the proportional sizing above is for.

## The buffer fixes one layer

The buffer covers the ramp, so anything that shortens the ramp shrinks the buffer. CPU utilisation is measured against requests and rises after the traffic is already there; requests per second, or queue depth through a KEDA external metric, moves earlier, so the HPA asks for pods earlier and there is less climb left for the placeholders to absorb. The HPA target is headroom too: 50% CPU instead of 80% is overprovisioning at the pod level, and so is a higher `minReplicas` in the sale window.

The image pull is the piece of node-ready time still in the path. With the buffer working, the first wave lands on nodes that have run the app before, but the second wave preempts placeholders on the node that just joined, and that node has never pulled the image. A node image with the app image baked in, or a DaemonSet that pulls it on every node as it joins, takes the pull off that path; on a cluster running [Spegel](/spegel/), the pull comes from a peer that already has it.

Everything below the cluster has the same problem one layer down, and no placeholder for it. If the HPA triples the checkout Deployment, there are three times as many connection pools open against a database whose `max_connections` did not move, three times the cache clients, three times the calls against a payment provider's rate limit, through a load balancer that was sized for last month. The cluster scales in seconds, the database does not, and the sale falls over on the layer that was never in the autoscaling chain. Those ceilings get raised by hand before the day, and the way to find which one is hit first is to load test the ramp rather than the plateau: replay the expected climb with the buffer in place and watch what goes red first.

## What to watch

Time spent `Pending` is the whole point, and the scheduler exposes it: `scheduler_pod_scheduling_sli_duration_seconds` is the end-to-end latency from a pod entering the scheduling queue, across however many attempts it took, and `scheduler_pending_pods{queue="unschedulable"}` is how many are stuck right now. With a working buffer the first stays flat through the spike and the second stays near zero. `scheduler_preemption_attempts_total` and `scheduler_preemption_victims` show the buffer being spent; a sale with no preemptions means the HPA never needed the space, which is its own finding. On the autoscaler side, `cluster_autoscaler_unschedulable_pods_count` should be your placeholders and nothing else, and `cluster_autoscaler_scaled_up_nodes_total` is counted when the node group size is updated, not when the node is `Ready`, so the end-to-end number still comes from the node objects. The cost is the placeholder requests summed over time; `kube_pod_container_resource_requests` from kube-state-metrics, filtered to the placeholder namespace, gives you the curve to argue about.

## Ending notes

The sentence I would give in an interview: overprovisioning reserves node capacity with low-priority pause pods, real pods preempt them the moment the HPA asks, the evicted placeholders go `Pending` and pull the next node up in the background, and for a scheduled sale the buffer itself is scaled up on a timer beforehand. Node provisioning time does not get shorter. It gets moved to where nobody is waiting on it.

That is the trade. You pay for idle capacity to buy seconds of latency, and the price is worth paying around spikes you can predict and mostly not the rest of the time. Two things to check on a cluster that already has this: the PriorityClass value against the autoscaler's cutoff, and the placeholder requests against the app's.

Thanks for your time :)
