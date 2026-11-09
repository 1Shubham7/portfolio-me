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

Going down is the privileged direction. Anyone can make their own process nicer; making it less nice needs either `CAP_SYS_NICE` or a permissive `RLIMIT_NICE`. `getrlimit(2)` gives the rule: the limit is a ceiling on how far the nice value can be raised in priority, calculated as `20 - rlim_cur`, so the useful range runs from 1 (allowing nice 19) to 40 (allowing nice -20). `ulimit -e` shows yours. That is also the cleanest example of the ulimit versus cgroup split: `ulimit` is a set of per-process limits, and cgroups are accounting and limits for a group of processes together.

None of that touches memory. When a machine runs short, the order of events is reclaim first, dropping page cache and swapping out if there is swap, and the OOM killer last, and the OOM killer does not read `NI`. It reads `/proc/<pid>/oom_score`, which `proc(5)` describes as a score based on the amount of memory the process uses, where a higher score makes the process more likely to be selected. `/proc/<pid>/oom_score_adj` is the knob on top of it, from -1000 to 1000, added to the score before the choice is made; -1000 makes the score always 0, so that task is never picked. Kubernetes writes that file from the pod's QoS class, which is the one place QoS decides who dies, and [the node-eviction post](/node-eviction/) has the table.

The confusion has a history, which made me feel a little better. The same man page notes that before Linux 2.6.36 the OOM score did factor in the nice value, along with runtime and CPU use. That changed in 2010. It has been a pure CPU knob for a long time.

## A quota is a ceiling, nice is a weight

My next question was a sharper version of the same confusion. If a container's cgroup already says it gets half a core, what is nice even doing? The cgroup decided how much CPU the thing gets.

`cpu.max` holding `50000 100000` means 50,000 microseconds of CPU time in every 100,000 microsecond period: half a core, and once the 50 ms is spent the group's threads stop until the next period starts. That is a cap on consumption, and [the millicores post](/millicores/) walks through how a Kubernetes limit becomes exactly that file. nice has no opinion about the cap. A process at nice -20 in a group that has exhausted its quota is throttled with everything else in the group, and nothing about its nice value refills the bucket.

What nice decides is the share a runnable task gets when it is competing with other runnable tasks, and it decides it through weight. `kernel/sched/core.c` has the table, `sched_prio_to_weight`, with a comment saying nice levels are multiplicative and each step changes a task's CPU share by about 10%, implemented as a factor of 1.25 per level. Nice 0 is weight 1024. Nice -10 is 9548. Nice +10 is 110. So the gap between -10 and +10 is not twenty steps on a line; it is a ratio of about 87 to 1, and two CPU-bound tasks at those values split a core roughly 99 to 1.

The interview version of the question goes: process A has a 500m quota and nice -10, process B has a 500m quota and nice +10, both CPU-bound on one core. Which is scheduled more favourably, and can either exceed its quota? Lower nice wins, and no. But the setup hides a catch that `sched(7)` states plainly: under group scheduling, a thread's nice value affects scheduling decisions only relative to other threads in the same task group. Two containers are two cgroups. What splits a busy CPU between cgroups is `cpu.weight`, the thing a Kubernetes CPU request turns into. nice inside container A says nothing to container B; it only reorders A's own threads among themselves.

So three knobs, three jobs. `cpu.max` caps the group. `cpu.weight` divides a contended CPU between groups. nice divides a group's share between its own tasks. None of them lets anything past the first.

## The %Cpu(s) line is every CPU added up

This is the header from my machine, 32 GB and a day and a half of uptime:

```
top - 16:34:37 up 1 day, 11:43,  1 user,  load average: 2.00, 1.20, 0.84
Tasks: 717 total,   4 running, 713 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.7 us,  1.3 sy,  0.0 ni, 95.8 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :  31781.7 total,   3882.7 free,  11868.5 used,  16030.5 buff/cache
MiB Swap:   2048.0 total,   2046.7 free,      1.2 used.  17273.4 avail Mem
```
