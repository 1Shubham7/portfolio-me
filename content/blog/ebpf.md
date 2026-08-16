---
title: "eBPF Explained: How Code Runs Inside the Linux Kernel, and Why Cilium Is Built on It"
description: "What eBPF actually is, why the verifier means it can't crash your kernel, hooks and maps in plain words, two runnable examples with bpftrace and libbpf, and how Cilium turns all of it into a CNI."
dateString: August 2026
draft: false
tags: ["eBPF", "Linux", "Kubernetes", "Cilium", "Networking", "DevOps"]
weight: 1
---

I'm an SRE, a Certified Kubernetes Administrator, and I help maintain [KubeAid](https://KubeAid.io). For the past few months I've been writing a lot of CiliumNetworkPolicies (enough to write a whole post on [the ways they silently break](/blog/cnp-antipatterns/)), and every time I read Cilium's docs to figure out why a packet got dropped, I hit the same phrase: "eBPF-based". Cilium is eBPF-based networking. Policy enforcement happens in eBPF. Hubble reads flows from eBPF.

I was nodding along at that phrase without actually knowing what it meant. Which eventually got annoying, so I sat down and learned what eBPF is, and this post is the write-up I wish I'd found on day one. If you know Linux basics and Kubernetes but have never touched kernel programming, this is for you.

## What eBPF actually is

eBPF lets you run your own small programs inside the Linux kernel. Not a kernel module, not a patched kernel, no reboot. You write a program in userspace, hand it to the kernel through a syscall, and the kernel runs it every time a specific event happens: a packet arrives, a syscall fires, a function gets called, a file gets opened.

That's the whole idea. Event-driven programs, loaded from userspace, executed by the kernel, in kernel context, at kernel speed.

The name is historical baggage. BPF, Berkeley Packet Filter, has been in the kernel since the 90s, and it did one narrow job: filter packets for tools like `tcpdump`. When you run `tcpdump port 443`, that filter expression gets compiled into BPF instructions and evaluated in the kernel so only matching packets are copied to userspace. Around kernel 3.18 the machinery was "extended": more registers, a real instruction set, the ability to call kernel helper functions, and attachment points far beyond packet filtering. Extended BPF, eBPF. Today the "packet filter" part of the name is mostly a lie, since eBPF programs trace syscalls, enforce security policy, and profile applications just as often as they touch packets.

## The problem it solves

Everything interesting on a Linux box goes through the kernel. Every packet, every syscall, every file open, every process start. If you want to observe or change that behavior, you have to be where it happens, and userspace isn't it.

Before eBPF you had two options, and both were bad in their own way:

1. **Write a kernel module.** Your code runs with full kernel privileges and zero safety net. A null pointer dereference doesn't crash your tool, it panics the box. Every kernel upgrade can break your module. Most ops teams, reasonably, refuse to run third-party modules in production at all.
2. **Get your change into the mainline kernel.** Now it's safe and maintained, but you're looking at years between "I need this" and "my distro ships it", assuming upstream wants the feature at all.

eBPF is the middle path: your code, running in the kernel, today, but sandboxed hard enough that the kernel can afford to trust it. The reason it can afford to is the verifier, which is the part worth actually understanding.

## From C code to running in the kernel

The pipeline looks like this:

1. You write a small restricted-C program and compile it with clang to **eBPF bytecode**, an instruction set for a little virtual machine defined by the kernel.
2. Your userspace loader hands the bytecode to the kernel via the `bpf()` syscall.
3. The kernel's **verifier** statically analyzes it, before a single instruction runs.
4. If the verifier accepts it, the bytecode is **JIT-compiled** to native machine instructions, so at runtime there's no interpreter overhead. It's real native code.
5. The program is **attached to a hook**, and from then on the kernel runs it whenever that event fires.

The verifier is the load-bearing step. In plain words, it refuses to load any program it can't prove is safe. It walks every possible execution path and checks that each one terminates, so no infinite loops (loops are allowed, but only when the verifier can prove they're bounded). It checks every memory access: you can't read a packet byte without first proving, with an explicit bounds check in your own code, that the byte is inside the packet. You can't dereference a pointer that might be null. Programs also have size and complexity limits, so "small program" is enforced, not a suggestion.

If any check fails, the `bpf()` syscall returns an error with a (famously grumpy) log explaining which instruction it didn't like, and nothing runs. This is why eBPF programs can't crash the kernel the way modules can: the entire class of "oops, bad pointer, kernel panic" bugs is rejected at load time. It's also why writing eBPF C feels strange at first. You end up writing bounds checks the compiler seems not to need, because the verifier is the real audience.

## Hooks: where your program runs

An eBPF program is useless until it's attached to an event source. The main ones:

| Hook | Fires when | Typical use |
|------|-----------|-------------|
| kprobes / uprobes | any kernel (or userspace) function is entered or returns | tracing, debugging, profiling |
| tracepoints | stable, named events the kernel maintainers placed by hand, e.g. every syscall entry | tracing that survives kernel upgrades |
| XDP | a packet arrives at the network driver, before the kernel has allocated any packet metadata | dropping DDoS traffic, load balancing at wire speed |
| tc (traffic control) | a packet passes the kernel's traffic control layer, ingress or egress, on any interface | container networking, policy, packet rewriting |
| socket and cgroup hooks | socket operations like `connect()` or `sendmsg()`, scoped to a cgroup | per-container policy, transparent redirection |

The ordering in the network path matters. XDP runs earliest, in the driver, which makes it the cheapest place to drop a packet you never wanted. tc runs a bit later but exists on virtual interfaces too, like the veth pair connecting a pod to its node, which is exactly why a certain CNI likes it. Hold that thought.

## Maps: how data gets in and out

Your program runs in the kernel and your tooling runs in userspace, so there has to be a bridge. That bridge is **eBPF maps**: data structures that live in the kernel and are accessible from both sides. The kernel side reads and writes them through helper functions, and userspace reads and writes them through the same `bpf()` syscall used for loading.

There are many map types, but three cover most real programs:

- **Hash maps** for key-value lookups, e.g. connection state keyed by 5-tuple.
- **Arrays** for fixed-size indexed data, e.g. counters or config flags.
- **Ring buffers** for streaming events out to userspace, e.g. "process X just called execve".

Maps go both directions, and that's the part that clicked late for me. Results flow out (your program counts things, userspace reads the counts), but configuration flows in: userspace writes entries into a map, and the kernel program looks them up on the next event. An eBPF firewall is exactly this, a program at a network hook doing a map lookup per packet, with userspace managing the map contents. No reload, no rule recompilation, just map updates.

## Hello world: bpftrace

You don't need to write C to start. `bpftrace` is a one-liner language that compiles to eBPF under the hood, and it's the fastest way to feel what "run my code on every kernel event" means. Install it from your distro's repos and run this as root:

```bash
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```

Read it left to right: attach to the tracepoint that fires on **every syscall entry**, and on each event, increment a counter keyed by the process name. `@[comm]` is a map, the same kernel-resident map from the previous section, just with friendlier syntax. Let it run a few seconds, hit Ctrl-C, and it prints the map:

```
Attaching 1 probe...
^C

@[systemd]: 41
@[sshd]: 194
@[containerd]: 2189
@[etcd]: 5091
@[kube-apiserver]: 12876
```

That's every syscall on the machine, counted per process, with a throwaway one-liner. No agent, no restart of anything.

One more, because this one feels like a superpower the first time. Print every program executed anywhere on the host:

```bash
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s -> %s\n", comm, str(args.filename)); }'
```

```
bash -> /usr/bin/kubectl
containerd-shim -> /usr/local/bin/runc
runc -> /proc/self/exe
```

Every `bpftrace` invocation goes through the full pipeline from earlier: compile to bytecode, `bpf()` syscall, verifier, JIT, attach. You just don't see it.

## A real one: counting packets with XDP and libbpf

For anything beyond one-liners you write two pieces: the kernel-side program in C, and a userspace loader. The modern way is libbpf. Here's a complete XDP program that counts packets per IP protocol:

```c
// xdp_counter.bpf.c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 256);
    __type(key, __u32);
    __type(value, __u64);
} proto_count SEC(".maps");

SEC("xdp")
int count_packets(struct xdp_md *ctx)
{
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)   // verifier demands this
        return XDP_PASS;

    if (eth->h_proto != bpf_htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)    // and this
        return XDP_PASS;

    __u32 key = ip->protocol;           // 6 = TCP, 17 = UDP, 1 = ICMP
    __u64 *count = bpf_map_lookup_elem(&proto_count, &key);
    if (count)
        __sync_fetch_and_add(count, 1);

    return XDP_PASS;
}

char LICENSE[] SEC("license") = "GPL";
```

A few things to notice, because they're conventions you'll see in every libbpf program:

- **`SEC(...)`** places code and data into named ELF sections. `SEC("xdp")` is how the loader knows this function is an XDP program, and `SEC(".maps")` is how it finds the map definition. The map itself is declared as a struct describing its type, size and key/value types; libbpf creates the actual kernel map from that description at load time.
- **The two bounds checks** are not optional style. Delete either one and the verifier rejects the program, because you'd be reading memory it can't prove is inside the packet.
- **The license declaration** is required. Some kernel helper functions are only available to GPL-compatible programs, and the kernel checks this string at load time.
- **`XDP_PASS`** means "I'm done, continue normal processing". Return `XDP_DROP` instead and you've written a firewall, which is genuinely how XDP-based DDoS mitigation works.

The userspace side compiles, loads, attaches, and then just reads the map in a loop:

```c
// xdp_counter.c (loader, abbreviated)
#include <stdio.h>
#include <unistd.h>
#include <net/if.h>
#include "xdp_counter.skel.h"   // generated by: bpftool gen skeleton

int main(int argc, char **argv)
{
    struct xdp_counter_bpf *skel = xdp_counter_bpf__open_and_load();
    bpf_program__attach_xdp(skel->progs.count_packets,
                            if_nametoindex(argv[1]));

    int fd = bpf_map__fd(skel->maps.proto_count);
    for (;;) {
        __u64 tcp = 0, udp = 0, icmp = 0;
        __u32 k;
        k = 6;  bpf_map_lookup_elem(fd, &k, &tcp);
        k = 17; bpf_map_lookup_elem(fd, &k, &udp);
        k = 1;  bpf_map_lookup_elem(fd, &k, &icmp);
        printf("TCP: %llu  UDP: %llu  ICMP: %llu\n", tcp, udp, icmp);
        sleep(2);
    }
}
```

The kernel part is compiled with `clang -O2 -g -target bpf`, then `bpftool gen skeleton` generates the `.skel.h` header that gives the loader typed access to the programs and maps. Rather than hand-rolling the build, clone [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap), which is the reference scaffold with the Makefile already sorted, and drop these two files in. Attach it to a busy interface and watch the counters move: two files, under sixty lines, and you're running verified native code on every packet the NIC receives.

## And now Cilium makes sense

This is where I started, so let's close the loop. Everything Cilium does is these same three primitives, hooks, maps, and the verifier, deployed at cluster scale.

**The datapath.** Cilium is a CNI that replaces the iptables chains and kube-proxy with eBPF programs attached at the tc hooks on every pod's veth pair, plus XDP and tc on the host NICs. When a pod sends a packet to a Service ClusterIP, the eBPF program at its veth looks the Service up in a BPF hash map, picks a backend, and rewrites the destination right there. One map lookup, instead of the kernel walking a chain of iptables rules that grows with every Service you add. For the programmers reading: O(1) per packet instead of O(n), with n being your rule count. And when a Service changes, Cilium's userspace agent just updates map entries, which is the "config flows in through maps" pattern from earlier, instead of resyncing rule chains.

**Policy.** Cilium derives a security identity from each pod's labels and stores IP-to-identity mappings in eBPF maps. When a packet crosses a veth hook, the program resolves source and destination identities from those maps, then does one more lookup in the policy map keyed on (source identity, destination identity, port, protocol). Allow or drop is the result of a hash lookup, not a rule scan. When I wrote about [CiliumNetworkPolicy anti-patterns](/blog/cnp-antipatterns/), nearly every trap on the list came down to what ends up in those maps versus what you assumed ends up in them.

**Observability.** Hubble, Cilium's flow observability layer, is almost a free lunch mechanically. The eBPF programs already sit on every packet path making forwarding and policy decisions, so exporting what they saw is a matter of pushing events through a ring buffer to userspace. No sidecar, no packet capture, no extra hop in the datapath.

And underneath all of it sits the verifier. A CNI that injects code into the packet path of every node in your cluster sounds terrifying until you remember that every one of those programs had to be proven safe before the kernel would load it. That guarantee is the reason "eBPF-based" is a feature and not a warning label.

## Ending notes and links

eBPF stopped being an intimidating phrase for me the moment it became three concrete ideas: small programs the kernel proves safe and then runs on events, hooks that decide which events, and maps that move data and config across the kernel boundary. Everything marketed as "eBPF-powered", Cilium included, is some arrangement of those three. Start with `bpftrace` one-liners on any Linux box you have, graduate to libbpf-bootstrap when a one-liner isn't enough, and the Cilium docs read very differently afterwards.

- [ebpf.io](https://ebpf.io/) - the best starting point, including "What is eBPF?"
- [github.com/bpftrace/bpftrace](https://github.com/bpftrace/bpftrace) - the one-liner tutorial in the repo is excellent
- [github.com/libbpf/libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) - scaffold for real libbpf programs
- [docs.cilium.io](https://docs.cilium.io/) - see the eBPF datapath section
- [Brendan Gregg's BPF page](https://www.brendangregg.com/bpf-performance-tools-book.html) - the deep end, for tracing and performance work
- [My CiliumNetworkPolicy anti-patterns post](/blog/cnp-antipatterns/) - where this rabbit hole started

Thanks for your time.
