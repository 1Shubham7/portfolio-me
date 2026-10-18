---
title: "Separating Public and Internal Traffic with Multiple Traefik Instances"
description: "Why I run two Traefik instances per cluster instead of one Traefik with an auth middleware: how the IngressClass flag keeps them apart, the Helm values for each, how the internal one is reached over a mesh VPN, and the three-tier variant with an IP allow-list in the middle."
dateString: October 2026
draft: false
tags: ["Kubernetes", "Traefik", "Ingress", "Networking", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/multiple-traefik-instances/multiple-traefik-instances.png"
---

Until July this year, Argo CD on the clusters I run sat behind the same Traefik as everything the clusters serve to the public. Same pods, same IP, same wildcard DNS record. What kept it off the internet was an auth middleware in front of the route. That works right up to the day someone copies an Ingress, forgets the annotation, and the login page for the thing that deploys everything is on a public hostname.

The fix was not a better middleware. It was a second Traefik, from the same Helm chart, with a ClusterIP Service and no public path at all. An Ingress picks which one serves it by naming an IngressClass. That is the whole design, and this post is how it fits together: the one flag that keeps the instances from fighting, the values for each, how the internal one is reached, and the version with three tiers.

