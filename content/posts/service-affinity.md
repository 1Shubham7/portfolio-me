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
