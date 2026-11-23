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
