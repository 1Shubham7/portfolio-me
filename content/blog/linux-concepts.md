---
title: "Where Linux Meets Kubernetes: Six Mental Models for When top Isn't Enough"
description: "Processes and signals, namespaces and cgroups, memory, storage, networking, and identity: the mechanism behind each, the two or three ways it breaks under a pod, and the one check that confirms it. Written from on-call across 100+ production Kubernetes clusters."
dateString: September 2026
draft: false
tags: ["Linux", "Kubernetes", "SRE", "DevOps", "Containers", "Incident Response"]
weight: 1
cover:
    image: "/blog/linux-concepts/cover.png"
---

Most Linux material aimed at DevOps people is a list of commands. That holds up until a container is misbehaving, `top` shows nothing useful, and the command you need depends on a mechanism nobody taught you. The six models below are the Linux I actually use on call, where Kubernetes stops explaining things and the node has to.

## Processes and signals

### The model

`ps` prints process state in the STAT column, and two states matter here. D is uninterruptible sleep: the process is inside the kernel, almost always blocked on I/O, and nothing wakes it until that I/O returns, signals included. Some of those waits have been killable since 2.6.25, meaning a fatal signal does wake them; `ps` shows both kinds as D. Z is a zombie: already exited, waiting for a parent that has not called `wait()`. Nothing to kill; fix the parent, or kill it so the zombies are reparented to something that reaps.

A signal has two moments. Sending marks it pending; delivery happens when the target next returns from kernel space to user space. That gap is why `kill -9` does nothing to a process in a true uninterruptible sleep: SIGKILL cannot be caught or ignored, but it cannot be delivered either until the process comes back from the I/O it is stuck in, and only the I/O completing, the device giving up, or a reboot brings it back. Killable waits are the exception, the NFS client's RPC waits among them, so SIGKILL is worth one try before going after the storage. SIGTERM is a request the process handles; SIGKILL it never sees.

Inside a container the entrypoint is PID 1 of its PID namespace, and PID 1 has special rules. The kernel does not deliver a signal to a namespace's init unless init installed a handler for it; only SIGKILL and SIGSTOP from an ancestor namespace are forced through. Orphans are reparented to it. So a binary with no SIGTERM handler dies to SIGTERM on a normal host and ignores it as PID 1 in a container.

The usual way in is the shell form of `CMD`. `CMD node server.js` runs as `/bin/sh -c "node server.js"`: sh is PID 1, node is its child, and sh does not forward SIGTERM. Use the exec form, or `exec` the real process from the entrypoint script. If the process needs a parent, tini and dumb-init are a tiny PID 1 that forwards signals and reaps orphans; Docker's `--init` is tini.

The Kubernetes sequence is fixed. On delete, the pod gets a deletion timestamp and a grace period (30 seconds by default), and the kubelet starts shutdown: the `preStop` hook if there is one, then the image's stop signal (SIGTERM unless `STOPSIGNAL` says otherwise) to PID 1 in each container, then SIGKILL at expiry. The clock starts before `preStop` runs, not when SIGTERM lands, so the hook and the application's own shutdown share one budget; a hook still running at expiry gets a one-off two seconds. Endpoint removal happens in parallel, not before, hence the `preStop` sleep.

### What actually breaks

A pod sits in Terminating far past its grace period. `kubectl delete --force` makes the object disappear and changes nothing on the node: something in the container is in D state with SIGKILL already pending, so this is not a killable wait. `ps -eo pid,stat,wchan:32,cmd` on the node finds the D and what it is waiting in, and `dmesg` usually has a matching `task ... blocked for more than 120 seconds`. Nearly always a hung NFS mount or a failing disk, and the fix is the storage.

Every rollout takes the full grace period and clients see errors for the duration. PID 1 is not handling SIGTERM. `kubectl exec <pod> -- cat /proc/1/cmdline`: if it prints `sh`, that is the answer; if it prints the application, send `kill -TERM 1` inside and watch whether it exits. Exec form, `exec` in the script, or an init.

### The one-liner

*A signal is delivered only when the process comes back from the kernel, and PID 1 only receives the ones it asked for.*

## Namespaces and cgroups

### The model

Namespaces control what a process can see, cgroups control what it can use, and the two are independent: a systemd service has a cgroup and, by default, no namespaces, while a container with no limits has namespaces and no real cgroup constraint. A container is an ordinary process with both applied, on a kernel it shares with every other container on the node. That shared kernel is why namespaces are not a security boundary: a kernel bug does not check which namespace called it. A real boundary is its own kernel: gVisor, Kata, or a VM.

A pod shares some namespaces and not others. The pause container holds the network namespace, so every container in the pod shares one IP, one `localhost` and one port space; it holds IPC and UTS too. PID namespaces are per container unless `shareProcessNamespace: true`, which makes pause PID 1 for everyone and turns `kill 1` inside a container into a signal to pause. Mount namespaces are always per container, so two containers in a pod talk over `localhost` and still cannot see each other's files.

cgroup v2 is one tree under `/sys/fs/cgroup`. The kubelet builds `kubepods`, then `burstable` and `besteffort` under it, while Guaranteed pods sit directly under `kubepods`; each pod gets a cgroup named by UID and each container a child of its pod.

`memory.max` is the hard limit. When reclaim cannot keep usage under it, the OOM killer runs scoped to that cgroup and kills its highest-scoring process: the biggest, not necessarily PID 1. On cgroup v1, or on v2 with `singleProcessOOMKill: true` (kubelet 1.32 and later), that one process dies, PID 1 carries on, and nothing restarts; the only trace is the `oom_kill` counter in `memory.events` and a `dmesg` line. On cgroup v2 with Kubernetes 1.28 or later, the kubelet sets `memory.oom.group` on the container cgroup, every process in it dies together, and the container exits 137 as `OOMKilled`. Know which your clusters do, because the first is invisible from `kubectl`.

`cpu.max` holds a quota and a period: `50000 100000` is half a CPU. Every thread in the cgroup draws on the quota, and it refills each period. Eight busy threads under that limit burn 50 ms of quota in about 6 ms of wall time, then all eight are frozen for the rest of the period. Averaged over a minute the graph says 30 or 40 percent utilisation; every request that arrived in the frozen window waited anyway. That is CFS throttling, and only `cpu.stat` shows it: `nr_throttled` against `nr_periods`, and `throttled_usec`.

### What actually breaks

Tail latency is bad and the CPU dashboard says a third of the limit. `cat /sys/fs/cgroup/cpu.stat` inside the container; if `nr_throttled` is a large fraction of `nr_periods`, the quota is the bottleneck. Raise the limit, drop it and keep the request, or size the thread pool to the quota (`GOMAXPROCS`, JVM flags).

Workers inside a container keep vanishing, memory sawtooths, and the pod has never restarted. Single-process OOM kills, usually a cgroup v1 node. Confirm with `oom_kill` in the container's `memory.events` and `Memory cgroup out of memory` in `dmesg`. "Never restarted" is not "never OOM killed".

### The one-liner

*Namespaces decide what a process sees and cgroups what it uses; the kernel underneath is shared either way.*

## Memory

### The model

Only touched pages cost RAM. Virtual size is reserved address space; resident set is what is in RAM right now, and what the OOM killer scores a process on. The cgroup charges more than RSS, though. The kernel keeps every file it has read in RAM until something needs the space, so `free` near zero on a healthy node is normal, and the number that matters is `available` (`MemAvailable` in `/proc/meminfo`): what a new allocation could get without swapping. Two kinds of cache are not droppable. tmpfs pages, which is what a memory-backed `emptyDir` and `/dev/shm` are, can only be swapped, never discarded, and count against the limit of the container that wrote them. Dirty pages must be written back before they can go.

Swap is only for anonymous memory. A heap page has no file to re-read it from, so under pressure it goes to swap or its process meets the OOM killer. Swap in use is not thrashing; thrashing is `vmstat` showing continuous `si` and `so`. The kubelet still refuses to start with swap enabled by default (`failSwapOn: true`). Support went GA in 1.34, opt-in per node with `swapBehavior: LimitedSwap` and only for Burstable pods, because swap makes a pod's speed depend on its neighbours.

Three different things get called OOM. A container OOM is the kernel enforcing `memory.max`: `OOMKilled`, exit code 137, restarted with backoff. A node OOM is the global killer firing because the whole node ran out. It scores every process by RSS, swap and page tables, shifted by `oom_score_adj`, which the kubelet sets per QoS class: Guaranteed -997, BestEffort 1000, Burstable between 2 and 999, falling as the memory request grows. A pod well under its own limit can die, BestEffort first, and `dmesg` says `Out of memory: Killed process` rather than `Memory cgroup out of memory`. A kubelet eviction is not the kernel at all: the kubelet sees `memory.available` under its threshold (100Mi by default), ranks pods by whether usage exceeds requests, then priority, then how far over requests they are, and terminates one. BestEffort goes first because it has no requests to be under, not because the kubelet looks at QoS. The pod ends `Failed` with reason `Evicted`, and the fix is requests that reflect reality.

Leak or cache is answered by `memory.stat` in the container's cgroup, which splits usage into `anon` and `file`. `anon` climbing for hours with no plateau is a leak; raising the limit changes the date of the next kill and nothing else. `file` climbing is page cache and reclaimable, unless `shmem` is climbing with it: that is a tmpfs `emptyDir` or `/dev/shm`, and it counts like anon. `anon` flattening above the limit is a working set bigger than the limit, and the limit is what is wrong.

### What actually breaks

OOMKilled, limit doubled, OOMKilled again a day later. Leak. Sample `anon` from `memory.stat` over the container's life and look for the plateau that never comes. Raising the limit is a scheduling decision, not a fix.

OOMKilled while the dashboard shows the pod nowhere near its limit. Either a node OOM, which one `dmesg` line on the node settles, or a dashboard plotting RSS while the cgroup was also charging a memory-backed `emptyDir` or dirty pages. `memory.current` and `memory.stat` on the node show what was actually counted.

### The one-liner

*Page cache is free until it isn't, and a container OOM, a node OOM and an eviction are three events with three fixes.*

## Storage and filesystems

### The model

A filename is a directory entry pointing at an inode, so a filesystem can be full in two ways: out of blocks, or out of inodes. Both return `ENOSPC`, which applications print as "No space left on device", and only the first shows in `df -h`. `df -i` shows the other; millions of small files exhaust inodes with space to spare.

Deleting a file removes the name, and the blocks are freed only when the last name and the last open descriptor are both gone. A log removed with `rm` while still open keeps every block, and `df` does not move. `lsof +L1` lists open files with a link count of zero. Truncating through the descriptor, `: > /proc/<pid>/fd/<n>`, frees the space without a restart. For active logs, `truncate -s 0` or logrotate's `copytruncate`, never `rm`.

Three things fill a node's disk. Image layers, which the kubelet garbage-collects only once disk usage passes 85 percent. Applications writing inside the container: the image layers are read-only overlayfs lower directories, every container gets one writable upper directory on the node's disk, the first write to an image file copies the whole file up, and all of it vanishes with the container. And stdout/stderr, written under `/var/log/pods` and rotated by the kubelet at 10Mi and five files per container by default, so a container that logs more than that between reads loses the oldest of it. The writable layer, the logs and any disk-backed `emptyDir` count toward the pod's `ephemeral-storage` limit, and exceeding it gets the pod evicted; the node's filesystem dropping under `nodefs.available` (10 percent by default) raises `DiskPressure` and the kubelet evicts regardless of limits.

A filesystem that turns read-only is the kernel protecting data on a device it no longer trusts. ext4 is mounted `errors=remount-ro` on most distributions, so the first metadata or I/O error flips it. "Read-only file system" in application logs is a storage problem, not a permissions problem: `dmesg` has the `EXT4-fs error` and the I/O errors under it. Do not chmod anything.

### What actually breaks

Pods on one node are evicted for `ephemeral-storage` or the node shows `DiskPressure`, and nobody deployed anything. `du -xsh /var/lib/containerd /var/log/pods` on the node separates images and writable layers from logs. An application writing inside its container is the usual one, and the fix is a volume.

Unrelated pods on one node start failing with "Read-only file system" in the same minute. The kernel remounted; `dmesg` has the device. Cordon, drain, replace the disk or the node; every pod scheduled there afterwards inherits the same broken disk.

### The one-liner

*Space is freed when the last name and the last open descriptor are both gone, and a read-only filesystem is the kernel telling you the disk is broken.*

## Networking

### The model

Every packet the kernel handles passes a fixed set of netfilter hooks (PREROUTING, INPUT or FORWARD, OUTPUT, POSTROUTING), and conntrack runs at those hooks ahead of NAT. That is why kube-proxy and most CNI plugins live here: a Service IP is a DNAT rule at PREROUTING and OUTPUT that rewrites the destination to one pod IP, and the reply is rewritten back because conntrack remembered the translation. kube-proxy's `iptables`, `ipvs` and `nftables` modes store and match those rules differently; all three need conntrack on the reply.

Conntrack is a hash table of flows with a fixed maximum, `nf_conntrack_max`, which kube-proxy sets at startup from its `maxPerCore` and `min` settings. When the table is full, the kernel drops any packet that would need a new entry while existing flows keep working. That asymmetry is the tell: new connections time out, everything already open is fine, and `dmesg` says `nf_conntrack: table full, dropping packet`. The usual cause is TIME_WAIT churn from a client opening a fresh connection per request; each entry lingers two minutes. `sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max` shows how close you are and `conntrack -S` shows drops.

A pod's network namespace reaches the node through a veth pair, bridged or routed depending on the CNI, and traffic to another node usually leaves the host encapsulated. Encapsulation costs bytes: VXLAN adds 50 to every IPv4 packet, IP-in-IP 20, WireGuard 60, and the pod interface's MTU has to drop by that much, 1450 for VXLAN on a 1500-byte network. Otherwise a full-size pod packet is too big once wrapped. TCP sets do-not-fragment, so it is dropped and the sender is supposed to learn the right size from an ICMP "fragmentation needed" reply, which firewalls and cloud security groups routinely drop, so the sender never learns. The signature: small requests succeed, anything that fills a packet hangs, and only between nodes. `ip link` in the pod shows the MTU the CNI set; `ping -M do -s <bytes> <remote>` from a pod finds the real boundary, with 28 added for headers, so `-s 1422` tests a 1450 path.

DNS from a pod is expensive by default. The kubelet writes three `search` domains and `options ndots:5` into `resolv.conf`, so a name with fewer than five dots is tried with every search domain before it is tried as written. `api.example.com` has two dots: three cluster-suffixed lookups return NXDOMAIN before the real one succeeds, each doubled for A and AAAA. Eight queries through CoreDNS for one external hostname, which `tcpdump -ni eth0 port 53` in the pod shows as an NXDOMAIN storm. A trailing dot (`api.example.com.`) skips the search list, a lower `ndots` in `dnsConfig` fixes it for the pod, and NodeLocal DNSCache hides it.

### What actually breaks

Connections to a Service time out intermittently while established ones work, on one node or a few. Conntrack full. `dmesg` for the `table full` line, then count against max. Raise the limit through kube-proxy's conntrack settings and find the client churning connections, or the table fills again at the new size.

Small requests work, large ones hang, only across nodes. MTU. `ip link` in the pod against the encapsulation's overhead, then `ping -M do -s` upward from 1400 until it fails. Fix the CNI's MTU setting; nothing on the application side is wrong.

### The one-liner

*A Service is a DNAT rule that only works while conntrack remembers the flow, and every overlay takes bytes off the MTU that PMTUD usually fails to give back.*

## Permissions and identity

### The model

The kernel knows numbers; a username is whatever the nearest `/etc/passwd` says. uid 1000 in a container is uid 1000 on the node and on every volume it mounts, with no translation, because pods run with `hostUsers: true` unless told otherwise; user namespaces (`hostUsers: false`) are opt-in per pod. Root in a container is uid 0 on the kernel; only capabilities and seccomp stop it acting like the node's root.

Capabilities split root's power into about forty flags, and the runtime starts a container with a default set of roughly a dozen. `privileged: true` grants all of them plus device access and disables seccomp and AppArmor, which makes the container the node. The pattern that holds, and the one the Restricted pod security standard requires, is `drop: ["ALL"]` and add back the one you need: `NET_BIND_SERVICE` for ports below 1024, `NET_ADMIN` for something that genuinely configures the network.

`fsGroup` is how a non-root process gets at a volume. At mount time the kubelet changes the volume's group to that gid with the setgid bit set, and adds the gid to the supplementary groups of every process in the pod. It only works on volume types that support it: secrets, configMaps and `emptyDir` always, a PVC if the CSI driver's `fsGroupPolicy` allows it, `hostPath` never. `fsGroupChangePolicy` decides how much work the mount does: `Always`, the default, walks and chowns the whole volume on every mount, which on a PVC with millions of files is minutes of `ContainerCreating` on every restart; `OnRootMismatch` skips the walk when the root directory already matches and makes it a one-time cost.

`runAsNonRoot: true` makes the kubelet refuse to start a container whose uid is 0. An image with no `USER` fails with "image will run as root"; one with a named `USER` fails with "cannot verify user is non-root", because the kubelet cannot read the image's `/etc/passwd`. Once the process is a non-zero uid, every file it touches must be readable by that uid or one of its groups. Secrets mount `0644` by default, readable by anyone; `defaultMode: 0400`, which the Kubernetes docs' own example sets, leaves the file root-owned and readable by nobody else.

The gotcha is how applications report it. A process that cannot read its key file almost never says `EACCES`; it says "authentication failed" or "unable to load private key", because the permission error was swallowed and re-thrown as the thing the code was trying to do. A permission problem on a mount surfaces as a credential problem, and the credential gets rotated before anyone looks at the file. I check the file first: `kubectl exec <pod> -- id` for the uid and groups the process has, then `ls -ln` on the mount, `-n` because names inside the container mean nothing.

### What actually breaks

An application reports an auth error the moment it moves to `runAsNonRoot` or a new uid, with no change to the credential. The process uid does not own the file and is not in its group. `id` and `ls -ln` inside the container settle it. `fsGroup`, a looser `defaultMode` on the secret, or a `chown` in the image.

Something works with `privileged: true` and fails without it, with `EPERM` in the logs. A missing capability. `grep Cap /proc/1/status` inside the container gives the effective set in hex and `capsh --decode=<hex>` names it. Add that one capability and remove `privileged`.

### The one-liner

*The kernel sees numbers, not names; a credential error that appears when a uid changes is a file permission, so `ls -ln` the mount before you rotate anything.*

## Links

- [pid_namespaces(7)](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html), [ps(1)](https://man7.org/linux/man-pages/man1/ps.1.html), [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html), the [cgroup v2 admin guide](https://docs.kernel.org/admin-guide/cgroup-v2.html), and [overlayfs](https://docs.kernel.org/filesystems/overlayfs.html)
- [Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/), [container lifecycle hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/), [tini](https://github.com/krallin/tini) and [dumb-init](https://github.com/Yelp/dumb-init)
- [Node-pressure eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/), [swap memory management](https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/), and the [kubelet configuration reference](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [Virtual IPs and service proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/), [Calico's MTU page](https://docs.tigera.io/calico/latest/networking/configuring/mtu), and [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Configure a security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) and [user namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/)
