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

## The IngressClass is the whole mechanism

Traefik's Kubernetes Ingress provider has one option that makes this work: `--providers.kubernetesingress.ingressclass`. Each instance gets its own value, and from then on each one only sees its own Ingresses.

The provider docs say what happens when it is unset: "If empty, resources missing the annotation, having an empty value, or the value `traefik` are processed." The code (`shouldProcessIngress` in `kubernetes.go`, linked below) also accepts any Ingress whose `spec.ingressClassName` names a Traefik IngressClass, and with the flag set it accepts only the class and the annotation value that match it.

So two instances with the flag unset both claim every Ingress with no class, every Ingress annotated `traefik`, and every Ingress pointing at any Traefik IngressClass. Routing still works, because both build the same routers. What breaks is status. Each instance writes its own Service's address into `status.loadBalancer` on every Ingress it processes, skipping the write only when the existing status already matches, so two instances with two addresses overwrite each other on every reconcile. The note I wrote down after that is one sentence: running multiple Traefik instances requires binding each to its own ingress class with `--providers.kubernetesingress.ingressclass=<class>`, or they fight over Ingress status updates.

That status field matters more than it looks. Argo CD's health check for an Ingress is one condition: `status.loadBalancer.ingress` has an entry, Healthy; otherwise Progressing (argoproj/argo-cd issue 1704, `health_ingress.go` in gitops-engine). Traefik only publishes an address when `publishedService` points at its own Service (`providers.kubernetesIngress.publishedService.enabled: true` in the chart, the default), and I set it on every instance. What gets copied depends on the Service type: for `LoadBalancer`, the IPs in its `status.loadBalancer`; for `NodePort`, the nodes' external IPs; for `ClusterIP`, only its `externalIPs`. A plain ClusterIP Service has none, so the internal instance's Ingresses carry an empty status even with `publishedService` on, and by that health rule Argo CD shows them Progressing. If that matters to you, give the ClusterIP Service an `externalIPs` entry so there is something to copy, or override the Ingress health check in `argocd-cm`.

The deprecated `kubernetes.io/ingress.class` annotation still works through the same flag, so old and new Ingresses coexist: in the configs I read while writing this, 157 Ingresses use the annotation set to `traefik`, 102 use it set to `traefik-private` (the private tier of the three-instance cluster below), and a handful use `spec.ingressClassName`. All of them land on the right instance.

Everything above is about Ingress objects. IngressRoute CRDs have their own copy of the flag, `--providers.kubernetescrd.ingressclass` (`providers.kubernetesCRD.ingressClass` in the chart), with the same rule: empty means every IngressRoute with no class or with class `traefik`. Both of my values files leave it empty, so an IngressRoute with no class is loaded by both instances, and one of them has a public address. If you use IngressRoute for anything that must stay internal, set the flag on each instance and put `spec.ingressClassName` on the route; the field arrived in v3.7, and the `kubernetes.io/ingress.class` annotation now logs a deprecation warning. The chart adds the class to its own dashboard IngressRoute when the value is set. Middlewares and TLSStores are not filtered by class at all, which is what lets one `default` TLSStore and one `office` Middleware serve every instance further down.

