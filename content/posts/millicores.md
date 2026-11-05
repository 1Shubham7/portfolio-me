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
