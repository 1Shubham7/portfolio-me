---
title: "Why Your DaemonSet Rolling Update Gets Stuck on hostPort Pods"
description: "A breakdown of why DaemonSets using hostPort can get stuck in Pending during rolling updates, and why the default maxSurge/maxUnavailable settings aren't always right for them."
dateString: July 2026
draft: false
tags: ["Kubernetes", "DevOps", "Networking"]
weight: 1
cover:
    image: "/blog/daemonset-hostport/cover.png"
---

A little while back I got pulled into debugging a stuck DaemonSet rollout. A new pod had been sitting in `Pending` for a while, the old pod on the same node was still running fine, and nothing about it looked obviously broken at first glance. Just a pod that refused to schedule. The real problem is one level up, in how the DaemonSet controller was told to roll out changes.

Once I dug in, the cause turned out to be a pretty common trap for anyone running DaemonSets that bind to host ports - think ingress controllers, CNI agents, log shippers, anything that needs to sit directly on a node's network. I figured it was worth writing up in a general way, since the failure mode is easy to hit and not obvious unless you've run into it before.

## A quick refresher on how DaemonSet rollouts work

DaemonSets get a pod on (almost) every node, and by default they update using the `RollingUpdate` strategy. That strategy is governed by two fields:

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 0
```

- **maxUnavailable** controls how many existing DaemonSet pods can be down at once during the rollout.
- **maxSurge** controls how many *extra* pods can be created on a node before the old one is removed.

For a typical Deployment, `maxSurge: 1` is what gives you zero-downtime updates. Say you're running 3 replicas. With `maxSurge: 1`, Kubernetes is allowed to briefly go up to 4 pods during a rollout: it creates 1 new pod, waits for it to become Ready, then kills 1 old pod. Repeat until all 3 are replaced. At no point do you drop below 3 healthy pods - the new one always comes up before the old one goes away. That's the "create first, remove second" behavior, and it's why `maxSurge` is generally treated as the safer, more availability-friendly setting.

Because that assumption holds so often, a lot of Helm charts carry it over into their DaemonSet templates too, defaulting to something like `maxSurge: 1, maxUnavailable: 0` even though DaemonSets don't have a replica count to surge above - each node only ever runs one instance of the pod.

For most workloads, that assumption holds. For anything binding to `hostPort`, it doesn't.

## Where it falls apart

If a DaemonSet pod uses `hostPort` - binding directly to a port on the node's network namespace, rather than going through a Service - only one pod on that node can hold that port at a time. It's a property of the host's network stack, not something Kubernetes can arbitrate around.

So picture a rollout with `maxSurge: 1, maxUnavailable: 0`:

1. The controller wants to update a node's pod, so it tries to schedule a **new** pod on that node first, per the surge setting.
2. The **old** pod is still running and still holding the host port.
3. The new pod can't bind the same port, so it never becomes Ready - it just sits there `Pending` or endlessly restarting, depending on the failure mode.
4. Because the new pod never becomes healthy, the controller never removes the old one. The rollout stalls, node by node.

The scheduler and kubelet aren't doing anything wrong here. They're following the rollout strategy exactly as configured. The strategy itself just doesn't fit the workload.

## The fix

Flip the priority: remove the old pod before adding the new one.

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 0
```

With `maxSurge: 0` and `maxUnavailable: 1`, the controller tears down the old pod on a node first, frees the host port, and only then schedules the replacement. You trade a brief moment of per-node unavailability for a rollout that actually completes - which, for something like an ingress controller running as a DaemonSet across multiple nodes, is a reasonable trade. Other nodes keep serving traffic while one node briefly has no pod at all.

The [Kubernetes documentation on rolling DaemonSet updates](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/) confirms this is intentional behavior, not a bug: `maxSurge` defaults to `0` upstream for exactly this reason, and charts or manifests that override it to enable surge need to account for host-networking constraints. If you're pulling in a community Helm chart for a hostPort-based DaemonSet, it's worth checking whether it silently sets a surge value - plenty do, since surge is a sensible default for the general case, and hostPort usage is more of an edge case.
