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

## Two releases of one chart

Both instances come from `traefik/traefik` 41.6.0 (Traefik v3.7.13), both live in the `traefik` namespace, as two Helm releases named `traefik` and `traefik-internal`. I deploy them as two Argo CD Applications pointing at the same chart with different values files, but nothing below depends on Argo CD.

The internal instance is the simpler of the two, so it goes first:

```yaml
# Second, ClusterIP-only Traefik instance that serves the private admin ingresses
# (ingressClassName: traefik-internal). No hostPort / NodePort / LoadBalancer:
# reachable only in-cluster and over the mesh via the traefik-internal ClusterIP
# Service.
fullnameOverride: traefik-internal
instanceLabelOverride: traefik-internal
deployment:
  kind: Deployment
  replicas: 2
service:
  spec:
    type: ClusterIP
  additionalServices: {}
ingressClass:
  enabled: true
  name: traefik-internal
  isDefaultClass: false
additionalArguments:
  - '--providers.kubernetesingress.ingressclass=traefik-internal'
  - '--entrypoints.websecure.http.tls'
  - '--metrics.prometheus=true'
```

`fullnameOverride` pins the Deployment, Service and ServiceAccount to the name `traefik-internal`, and the ClusterRole and ClusterRoleBinding to `traefik-internal-traefik`, since the chart suffixes those two with the namespace. They would not collide without it, because every one of those names derives from the release name, so two releases in one namespace already get distinct Deployments, ClusterRoles and IngressClasses. The override is there because something outside the chart depends on the Service being called exactly that, which the next section gets to. `instanceLabelOverride` does the same for the `app.kubernetes.io/instance` label, which the chart would otherwise set to `traefik-internal-traefik`; the metrics carry that label, so it is the name I want to type in Prometheus.

`service.spec.type: ClusterIP` is the actual isolation. The chart's default is `LoadBalancer`; with ClusterIP there is no NodePort, hostPort or cloud load balancer, so the only way in is from inside the cluster or from something routed into the pod network.

`ingressClass.name` creates the IngressClass this instance answers to (unset, it falls back to the fullname), and `isDefaultClass: false` keeps an Ingress with no class from landing here by accident. The `ingressclass` flag ties the provider to that class; `providers.kubernetesIngress.ingressClass` renders the same flag, and I use `additionalArguments` only so the arguments that define the instance sit together.

The public instance on my bare-metal clusters has no cloud load balancer to hide behind, so it runs as a DaemonSet with hostPorts and a failover IP:

```yaml
instanceLabelOverride: traefik
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
# DaemonSet + hostPort: delete old pod before starting new.
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 0
    maxUnavailable: 1
deployment:
  kind: DaemonSet
ports:
  web:
    hostPort: 80
    expose:
      default: true
  websecure:
    hostPort: 443
    expose:
      default: true
tlsStore:
  default:
    defaultCertificate:
      secretName: wildcard-example-com
service:
  enabled: true
  spec:
    type: NodePort
    externalIPs:
      - 203.0.113.10
additionalArguments:
  - "--providers.kubernetesingress.ingressclass=traefik"
  - "--entrypoints.websecure.http.tls"
  - "--metrics.prometheus=true"
providers:
  kubernetesCRD:
    enabled: true
    allowCrossNamespace: true
  kubernetesIngress:
    publishedService:
      enabled: true
podDisruptionBudget:
  enabled: false
```

A single failover IP is moved between nodes by a small controller that talks to the hosting provider's API, and the cluster's wildcard DNS record is a CNAME to that IP's name. The IP can land on a control-plane node, hence the toleration. `externalIPs` on the NodePort Service makes kube-proxy accept traffic for that IP on every node, but it is not what ends up in Ingress status: for a NodePort Service, `publishedService` copies the nodes' external IPs, and nodes joined with kubeadm often have no `ExternalIP` in `status.addresses` at all. So on bare metal the `ADDRESS` column of a public Ingress can be empty while traffic flows fine, and the same two fixes as for the internal instance apply.

The chart's default rolling update is `maxSurge: 1, maxUnavailable: 0`. With hostPort that cannot work: the new pod cannot bind port 80 while the old pod still holds it, so the old one has to go first. A PodDisruptionBudget on a DaemonSet with one pod per node adds nothing, so it is off. On one cluster the same DaemonSet carries a third entrypoint for a git server's SSH, which is just another port:

```yaml
ports:
  gitea-ssh:
    port: 2222
    hostPort: 2222
    protocol: TCP
    expose:
      default: true
```

The dashboard is not exposed on either instance. My values leave `ingressRoute.dashboard.enabled` at its default of `false`, and with `api.insecure` unset the API is only served through a router, so there is nothing to port-forward to. If you want it, set `ingressRoute.dashboard.enabled: true` and port-forward the `traefik` entrypoint, which on chart 41.6.0 is 8080 (`ports.traefik.port`): `kubectl -n traefik port-forward <pod> 8080:8080`, then `http://localhost:8080/dashboard/`.

