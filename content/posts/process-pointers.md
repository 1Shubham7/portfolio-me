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


`nice` came up while I was preparing for an SRE interview, and I gave the answer I had been carrying around for years: it sets a process's priority, from -20 to 19, lower number wins, and that priority is also what decides which processes get thrown out when the node runs low on memory. The first half is right. The second half is a different subsystem entirely, and once I started pulling on that thread I found a few more things in `top` that I had been reading every day without ever checking what they meant. This post is those pointers, each kept short.

## nice is a CPU knob, and only a CPU knob

The range is -20 to 19, and `sched(7)` is specific about what it covers: the nice value affects the scheduling of `SCHED_OTHER` and `SCHED_BATCH` processes, which is to say ordinary tasks under the fair scheduler. Real-time classes ignore it. Lower is more favourable. `nice -n 10 ./job` starts a process at nice 10, `renice -n 5 -p 4211` changes one that is already running, and the `NI` column in `ps -l` or `top` shows the current value.
