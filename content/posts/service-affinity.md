---
title: "When Your App Needs the Same Pod Every Time: Kubernetes Service Affinity Explained"
description: "A Service picks a pod once per connection and forgets. sessionAffinity: ClientIP is the one knob that makes it remember, and it is narrower than the name suggests: it pins new connections only, by source address, in a per-node table with a sliding timeout. How kube-proxy builds it from the iptables recent match, IPVS persistence and nftables sets, why NAT and a non-sticky load balancer defeat it, why it does nothing behind Traefik, and what to use there instead."
dateString: November 2026
draft: false
tags: ["Kubernetes", "Networking", "kube-proxy", "Services", "Traefik", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/service-affinity/service-affinity.png"
---

A Service with three endpoints hands each new connection to one of them, and with kube-proxy in iptables mode the pick is random. That is fine until the app keeps something in the pod: a login session in memory, a cache warmed for one customer. The client's second request lands on a pod that has never heard of it. Kubernetes has exactly one knob for this on the Service itself, `sessionAffinity: ClientIP`, and it is both simpler and narrower than the name suggests. This post is about what it pins, how kube-proxy does it, and where it does nothing at all.

## The field

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

There are two values, `None`, the default, and `ClientIP`. [The API reference](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/) says `timeoutSeconds` must be greater than 0 and at most 86400, one day, defaults to 10800, three hours, and is only accepted with `ClientIP`. The setting is per Service, not per port, so every port in the list shares it. And nothing in it is specific to TCP; the match is on the source address, so UDP gets the same treatment.

## It pins new connections, nothing else

kube-proxy works at layer 4. It makes one decision per connection, on the first packet, and conntrack carries that decision for every packet after it. An established TCP connection already stays on one pod, with or without affinity. All `ClientIP` adds is a memory across connections: when the same source address opens a new one, send it where the last one went.

Two things follow. HTTP keep-alive and gRPC over HTTP/2 give you affinity by accident, because the client reuses one connection for many requests. That is also why gRPC balances badly behind a plain ClusterIP: every request rides the one connection to the one pod, and nothing on the Service changes that, because the Service never sees a second connection. gRPC needs client-side balancing or an L7 proxy, not affinity.

The demo has to respect this. A 3-replica Deployment that echoes its hostname looks sticky even with `sessionAffinity: None` if the client reuses its connection. A loop of separate `curl` runs opens a fresh connection each time; a keep-alive client needs `Connection: close` on every request. Then `None` prints three hostnames and `ClientIP` prints one.

## Under the hood

In iptables mode the implementation is the kernel's `recent` match. [kube-proxy's iptables proxier](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/iptables/proxier.go) puts a rule at the top of the service chain for each endpoint, `-m recent --name <endpoint chain> --rcheck --seconds <timeout> --reap`, that jumps to that endpoint's chain on a hit, and the endpoint chain itself carries `-m recent --name <endpoint chain> --set`, which records the source address before the DNAT. The list is per endpoint, not per Service. Two details of the module matter. `--reap` purges entries older than the window on every check. And `--set` on an address already in the list refreshes its timestamp: in [xt_recent.c](https://github.com/torvalds/linux/blob/master/net/netfilter/xt_recent.c) the `XT_RECENT_SET` path calls `recent_entry_update` whether the entry is new or not. The nat table only sees the first packet of a connection, so every new connection from a client resets its clock. The timeout is a sliding window from the client's last new connection, not three hours from its first.

Both halves are readable on the node:

```bash
iptables-save -t nat | grep -- '--rcheck'
cat /proc/net/xt_recent/KUBE-SEP-*
```
