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

## A boundary, not a policy

An auth middleware is a policy. It has to be attached to every route that needs it, and forgetting is silent: the Ingress works, the certificate is issued, nothing in the cluster complains. A Traefik that is not reachable from the internet is a network boundary. An Ingress on that class cannot be exposed by accident, because there is no address for the public to reach it at. You still put authentication on Argo CD and Grafana. You just stop depending on it as the only thing between them and port scanners.

There are smaller wins I did not plan for. A bad rollout of the public Traefik no longer takes down Argo CD, the tool I would use to roll it back. The instances are sized separately: on the three-instance cluster further down, the one carrying the internal hostnames has about three times the CPU request of the public one, because that is where most of the hostnames live. Access logs and metrics are per instance, so the public access log holds only public traffic.

