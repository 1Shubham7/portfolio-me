---
title: "Some Super Interesting Process Management Pointers"
description: "nice and renice set a CPU scheduling weight and nothing else: the OOM killer reads oom_score, not NI. Why a lower nice cannot lift a cgroup cpu.max quota and why -10 versus +10 is a weight ratio near 90 to 1 rather than 20 steps, what each field of top's %Cpu(s) line means and why it is every logical CPU summed, and why 717 tasks with 4 running means 713 of them are asleep."
dateString: November 2026
draft: false
tags: ["Linux", "SRE", "DevOps", "cgroups", "Performance", "Troubleshooting"]
weight: 2
cover:
    image: "/blog/process-pointers/process-pointers.png"
---

