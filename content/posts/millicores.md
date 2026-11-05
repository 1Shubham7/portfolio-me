---
title: "CPU Isn't a Thing, It's Time: Kubernetes Millicores Explained"
description: "What a Kubernetes CPU value actually measures: CPU-seconds per second on one logical CPU. How a 250m limit becomes cpu.max 25000 100000 and a 100 ms throttle window, why a request is a weight and not a cap, and why 1000m is a hyperthread, not a core."
dateString: November 2026
draft: false
tags: ["Kubernetes", "Linux", "cgroups", "Performance", "SRE", "DevOps"]
weight: 2
cover:
    image: "/blog/millicores/millicores.png"
---


I have written `cpu: 250m` into more Deployments than I can count, and until recently I could not have told you what it meant. Everyone says it is a quarter of a CPU. The docs talk about CPU time. A quarter of a CPU's time over what? A day? A second? Nobody writes down the denominator, so the number stays a thing you copy from the last manifest. This post is the explanation I wish I had read first: where the denominator is, and what the kubelet turns the number into.

## You cannot split a core, only its time

A CPU core runs one thread at any instant. Your laptop looks like it is running three hundred things at once because the kernel switches between them every few milliseconds, which is fast enough that nothing seems to stop. So there is no such thing as giving a process half a core. The silicon does not divide. What divides is the core's time: how many of its milliseconds each thread gets before the kernel moves on to the next one.

One cashier at a shop is the picture that made it click for me. You cannot hand a customer half a cashier. You can give them half of the cashier's minutes.

## The unit is CPU-seconds per second

The confusing word is "time", because the unit is actually a rate, like km/h. One Kubernetes CPU is one second of CPU time for every second of wall-clock time. That is the missing denominator: it is always per second, and it is always a fraction of one core's seconds.

| Value | Per 1 s of wall-clock time |
| :-- | :-- |
| `1` (`1000m`) | 1000 ms of CPU time, a whole core |
| `500m` | 500 ms of CPU time |
| `250m` | 250 ms of CPU time |
| `1m` | 1 ms of CPU time |
| `2` (`2000m`) | 2000 ms of CPU time, which needs two cores running in parallel |

So one millicore is one millisecond of CPU work per second. That is the whole definition. The docs add that `0.1` and `100m` are the same value, that nothing finer than `1m` is allowed, and that the amount is absolute: `500m` asks for the same thing on a 2-core node and a 64-core node.

The last row is why it has to be time and not a core. You cannot fit two seconds of work into one second on one core, but two cores can do it between them. CPU time adds up across cores, and the kernel is free to add it up on whichever cores it likes.

## What a limit turns into

The kernel does not check once a second. It checks in a smaller window called the CFS period, which the [CFS bandwidth control docs](https://docs.kernel.org/scheduler/sched-bwc.html) default to 100 ms. The kubelet uses the same number: in `pkg/kubelet/cm/helpers_linux.go` the period is a constant, `QuotaPeriod = 100000` microseconds, and the quota is `milliCPU * period / 1000`. It can be changed with `cpuCFSQuotaPeriod` behind the `CustomCPUCFSQuotaPeriod` feature gate, and almost nobody does.

So `limits.cpu: 250m` becomes `250 * 100000 / 1000 = 25000`, and on a cgroup v2 node you can read it back from inside the container:

```bash
$ kubectl exec -it api-7d4c9-x2k1p -- cat /sys/fs/cgroup/cpu.max
25000 100000
```

Quota first, period second, both in microseconds. On cgroup v1 the same two numbers are `cpu.cfs_quota_us` and `cpu.cfs_period_us`. A container with no limit gets the kernel default of `max 100000`: the same period, nothing to run out of.

The mechanism is a bucket. At the start of each 100 ms period the container gets 25 ms of CPU time. Its threads spend from that bucket as they run, and when it is empty they stop, all of them, until the next period starts and the bucket is refilled. The kernel docs say it plainly: throttled threads will not be able to run again until the next period when the quota is replenished. That is what a CPU limit is. Not a slower CPU, a CPU that is switched off for the rest of the window once you have used your share of it.

## "On any cores" is the part that bites

The 25 ms is not tied to a core. It can be spent as 25 ms on core 3, or 10 ms on core 0 and 15 ms on core 7, or four threads on four cores each running about 6.25 ms. The kernel hands quota out to each CPU's run queue in 5 ms slices (`kernel.sched_cfs_bandwidth_slice_us`), so the real picture is lumpier than that, but the total is still 25 ms per period however it is split.

That last case is the production gotcha. A Go service, a JVM with its thread pools, a Node app with workers: four busy threads burn through a 25 ms bucket in about 6.25 ms of wall time, and then the whole container is frozen for the remaining 93.75 ms. Averaged over a second it used a quarter of a core, exactly what the limit said. The CPU graph looks calm. Meanwhile every request that arrived during the freeze waited up to 94 ms for nothing, and p99 latency is terrible.

The counter that shows this is `container_cpu_cfs_throttled_periods_total` from cAdvisor, the number of periods in which the container was throttled, next to `container_cpu_cfs_periods_total` for the total. The same two numbers are `nr_throttled` and `nr_periods` in the container's `cpu.stat`. High ratio, low CPU graph: this is why. Whether you should set CPU limits at all is an argument for another post; this one is only about what a limit does when you set it.

## A request is a weight, not a cap

`requests.cpu: 250m` never throttles anything. It does two unrelated jobs.

The first is placement. The scheduler only puts the pod on a node where the sum of requests, including this one, fits inside the node's allocatable CPU. That is accounting on paper; nothing on the node enforces it.

The second is a share of the node when the node is busy. The kubelet turns the request into cgroup v1 shares at 1024 per CPU, so 250m is 256 shares. Cgroup v2 has no shares, only `cpu.weight` in the range 1 to 10000, and the kubelet and runc map one to the other with `1 + (shares - 2) * 9999 / 262142`. For 250m that is 10:

```bash
$ kubectl exec -it api-7d4c9-x2k1p -- cat /sys/fs/cgroup/cpu.weight
10
```

Weights only matter under contention. The cgroup v2 docs describe the controller as work-conserving: only children that can use the resource right now take part in the split. On an idle node a pod with a 250m request and no limit can use the whole machine. When two pods want the same core, the kernel divides its time in proportion to their weights, and that is all it does.

Pod A requests 250m and pod B requests 750m, both on a one-core node and both busy. The weights are 10 and 30, so A gets about a quarter of the core and B about three quarters. B goes idle, A gets the whole core. Nothing was promised to A except "at least this much when there is a fight", and nothing was taken from it when there was not.
