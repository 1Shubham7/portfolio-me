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

A Service with three endpoints hands each new connection to one of them, and with kube-proxy in iptables mode the pick is random. That is fine until the app keeps something in the pod: a login session in memory, a cache warmed for one customer. The client's second request lands on a pod that has never heard of it. Kubernetes has exactly one knob for this on the Service itself, `sessionAffinity: ClientIP`, and it is both simpler and narrower than the name suggests.

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

There are two values, `None`, the default, and `ClientIP`. [The API reference](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/) says `timeoutSeconds` must be greater than 0 and at most 86400 (one day), defaults to 10800 (three hours), and is only accepted with `ClientIP`. The setting is per Service, not per port, so every port in the list shares it. And nothing in it is specific to TCP; the match is on the source address, so UDP gets the same treatment.

## It pins new connections, nothing else

kube-proxy works at layer 4. It makes one decision per connection, on the first packet, and conntrack carries that decision for every packet after it. An established TCP connection already stays on one pod, with or without affinity. All `ClientIP` adds is a memory across connections: when the same source address opens a new one, send it where the last one went.

Two things follow. HTTP keep-alive and gRPC over HTTP/2 give you affinity by accident, because the client reuses one connection for many requests. That is also why gRPC balances badly behind a plain ClusterIP: every request rides the one connection to the one pod, and nothing on the Service changes that, because the Service never sees a second connection. gRPC needs client-side balancing or an L7 proxy, not affinity.

The demo has to respect this. A 3-replica Deployment that echoes its hostname looks sticky even with `sessionAffinity: None` if the client reuses its connection. A loop of separate `curl` runs opens a fresh connection each time; a keep-alive client needs `Connection: close` on every request. Then `None` prints three hostnames and `ClientIP` prints one.

## Under the hood

In iptables mode the implementation is the kernel's `recent` match. [kube-proxy's iptables proxier](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/iptables/proxier.go) puts a rule at the top of the service chain for each endpoint, `-m recent --name <endpoint chain> --rcheck --seconds <timeout> --reap`, that jumps to that endpoint's chain on a hit, and the endpoint chain itself carries `-m recent --name <endpoint chain> --set`, which records the source address before the DNAT. The list is per endpoint, not per Service. Two details of the module matter. `--reap` is housekeeping: a check may evict the single oldest entry in the list if it is past the window, so stale addresses drain out instead of piling up, but `--rcheck` already ignores an entry older than `--seconds` with or without it. The detail that defines the behaviour is `--set` on an address already in the list, which refreshes its timestamp: in [xt_recent.c](https://github.com/torvalds/linux/blob/master/net/netfilter/xt_recent.c) the `XT_RECENT_SET` path calls `recent_entry_update` whether the entry is new or not. The nat table only sees the first packet of a connection, so every new connection from a client resets its clock. The timeout is a sliding window from the client's last new connection, not three hours from its first.

Both halves are readable on the node:

```bash
iptables-save -t nat | grep -- '--rcheck'
cat /proc/net/xt_recent/KUBE-SEP-*
```

IPVS mode sets the persistent flag on the virtual server with the same timeout; `ipvsadm -Ln` shows it as `persistent 10800` next to the scheduler name. The nftables backend keeps a per-endpoint set of addresses with a timeout, checked in the service chain and updated in the endpoint chain. Cilium's kube-proxy replacement [honours the field too](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/), with the same three-hour default refreshed by each request.

## When the client IP is not the client

The key is an address, not a user. Everything behind one NAT, one corporate proxy or one load balancer that proxies rather than forwards lands on one pod, which is a hot spot rather than a spread. Inside the cluster the address is honest: [the source IP tutorial](https://kubernetes.io/docs/tutorials/services/source-ip/) says packets to a ClusterIP from in-cluster clients are never source NAT'd in iptables mode. NodePort and LoadBalancer traffic is SNAT'd by default under `externalTrafficPolicy: Cluster`, which is why the pod sees a node IP, but the masquerade happens in POSTROUTING, after the service chain has already run its `recent` check in PREROUTING, so kube-proxy's own table on the receiving node is still keyed on the real address. What `Cluster` costs you is different.

There is no cluster-wide table. The `recent` list lives in one node's kernel, so each node pins independently, and an external load balancer that is not sticky itself can send a client to node A, then node B, and get a different pod each time. `externalTrafficPolicy: Local` does not change that; it only narrows each node's choice to its own pods. For affinity to hold end to end, whatever sits in front has to be sticky as well.

## Behind Traefik it does nothing

The ingress case is the one that catches people. [The Traefik post](/traefik/) went through which traffic Traefik actually sees; the part that matters here is where it sends it, and that is not the ClusterIP. Its Kubernetes providers default `nativeLBByDefault` to false, and [the annotation reference](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress/) describes that switch as choosing whether the load balancer's children are the pod IPs directly or the Service's clusterIP. By default they are the pod IPs, Traefik balances across them itself, and kube-proxy's rules for that Service are never consulted. Even if you flip it, the source address kube-proxy sees is the Traefik pod, so every external user is one client. NGINX ingress works the same way. For HTTP behind an ingress, stickiness is the controller's job: in Traefik, `sticky.cookie` on the service inside an IngressRoute, or the `traefik.ingress.kubernetes.io/service.sticky.cookie` annotation on an Ingress. Traefik sets a cookie on the first response naming the server that handled it and routes on it afterwards; if that server is gone the request goes to a new one and the cookie follows. A per-user key instead of a per-address one, which fixes the NAT problem as well.

Headless Services (`clusterIP: None`) are skipped entirely; the docs say kube-proxy does not handle them and no proxying is done, DNS hands out pod IPs and the client picks. ExternalName is a CNAME. A service mesh sidecar chooses the endpoint itself, so Istio wants a DestinationRule with `consistentHash` on a cookie, a header or `useSourceIp` instead. Gateway API has session persistence on the way, but [GEP-1619](https://gateway-api.sigs.k8s.io/geps/gep-1619/) is still marked Experimental. Envoy-based proxies offer consistent hashing, ring hash or Maglev, on a header, cookie or source IP, which survives scaling better than a pinned table because only a fraction of keys move when a pod is added or removed.

## When the pinned pod goes away

The pod's endpoint leaves the EndpointSlice, kube-proxy deletes that endpoint's chain and its `recent` list with it, and the client's next new connection falls through to the random pick and is pinned wherever it lands. Connections that are already open are conntrack's business, not the affinity table's, and keep flowing until they close or the pod stops answering, which is why graceful shutdown and `preStop` still matter; [the 30-second pod post](/pod-30-seconds/) covers that side. Affinity makes a client likely to return to the same pod. It never promises it.

## Not the other affinities

The scheduler once had a `ServiceAffinity` plugin that placed pods on nodes sharing a label; [KEP-785](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/785-scheduler-component-config-api) removed it from the `v1beta2` config API in 1.22 in favour of `NodeAffinity`, and [topology spread](/topology-spread/) is what you reach for now. `trafficDistribution: PreferSameZone` (the older `PreferClose` is a deprecated alias) and `internalTrafficPolicy: Local` steer traffic to endpoints in the same zone or on the same node as the caller. Those are affinities to a place. `sessionAffinity` is the only one keyed on who is calling.

## Ending notes

Use it for the cases it was built for: a legacy app that holds sessions in memory and cannot be changed soon, WebSocket-style traffic that benefits from client locality, a per-client cache that is expensive to warm. Set the timeout to the session length, not the maximum. And before any of that, check whether the traffic reaches the ClusterIP at all; behind Traefik it does not, and the cookie is the answer there.

The better fix is still to take the state out of the pod, into Redis or a database, and let the Service balance however it likes. `ClientIP` affinity is a per-node, per-address, best-effort memory with a sliding window. It is good at exactly that and nothing more.

Thanks for your time :)
