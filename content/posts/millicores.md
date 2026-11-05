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

