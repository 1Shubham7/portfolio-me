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
