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

## Reaching the instance that has no address

A ClusterIP Service is reachable from inside the cluster and from anything with a route into it, and nothing else. In my case the route is NetBird, an open-source WireGuard mesh. A NetBird operator in the cluster runs a routing peer (a `NetworkRouter`) and publishes chosen Services as `NetworkResource` objects:

```yaml
apiVersion: netbird.io/v1alpha1
kind: NetworkResource
metadata:
  name: traefik-internal
  namespace: traefik
spec:
  networkRouterRef:
    name: my-cluster
    namespace: netbird
  serviceRef:
    name: traefik-internal
  groups:
    - name: cluster-admins
```

The operator publishes the Service under a mesh DNS zone, so for peers in the right group `argocd.example.com` resolves to the internal Traefik's ClusterIP and the request arrives on its `websecure` entrypoint with TLS. The resource names the Service, which is the reason for `fullnameOverride`: the comment in my values file says to keep the Service called exactly `traefik-internal` so the existing `NetworkResource` keeps resolving, because renaming it breaks mesh access to the cluster. The name is a contract with something outside the chart, so write that down next to the override. The resource also only accepts a ClusterIP Service, which is why the internal instance has no other Service type; point it at a LoadBalancer or NodePort Service and it goes Ready False with "Referenced Service is not of type ClusterIP." and nothing else complains.

Nothing here is specific to NetBird. Tailscale's operator does the same job, and so does a plain WireGuard peer with a route into the pod CIDR. On a cloud cluster you do not even need a mesh: keep `type: LoadBalancer` and ask the cloud for an internal one, `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` on Azure (on AWS the load balancer controller already defaults `aws-load-balancer-scheme` to `internal`), the Hetzner form in the three-tier section below. Your VPN into the provider's private network is then the boundary.

The three ways I run a public or internal instance, side by side:

| | Bare metal, public | Cloud, public | Internal |
| :-- | :-- | :-- | :-- |
| Workload | DaemonSet, hostPort 80/443 | Deployment, 2 replicas | Deployment, 2 replicas |
| Service | NodePort with `externalIPs` set to a failover IP | LoadBalancer, PROXY protocol on | ClusterIP, or an internal LoadBalancer |
| Client IP | Direct | From the PROXY header | Direct (mesh peer IP) |
| How traffic arrives | DNS CNAME to the failover IP | DNS to the LB IP | Mesh DNS to the ClusterIP |

## The cloud variant and PROXY protocol

On a cluster on Hetzner Cloud with the cloud controller manager, the public instance is a Deployment with two replicas behind a `LoadBalancer` Service. Without help Traefik sees the load balancer's IP as the client. The fix on both sides is PROXY protocol:

```yaml
additionalArguments:
  - "--entrypoints.web.proxyprotocol=true"
  - "--entrypoints.web.proxyprotocol.insecure=true"
  - "--entrypoints.websecure.proxyprotocol=true"
  - "--entrypoints.websecure.proxyprotocol.insecure=true"
service:
  annotations:
    load-balancer.hetzner.cloud/network-zone: eu-central
    load-balancer.hetzner.cloud/uses-proxyprotocol: "true"
    load-balancer.hetzner.cloud/use-private-ip: "true"
```

Both sides, because the Hetzner default for `uses-proxyprotocol` is `false`. Set the Traefik side only and every connection is rejected, since Traefik waits for a PROXY header the load balancer never sends; cert-manager's HTTP-01 challenges are connections too, so no certificate on that instance is issued or renewed until the annotation is fixed. The Traefik docs are blunt about the reverse case: configure PROXY protocol on both sides or you have opened a request forgery hole. They also prefer `proxyProtocol.trustedIPs` with the load balancer's addresses over `proxyProtocol.insecure`, which trusts every incoming connection. The values above use `insecure`; if your provider documents the addresses its load balancers connect from, `trustedIPs` is the tighter choice.

## TLS: one wildcard, served by every instance, from DNS-01

Let's Encrypt cannot complete an HTTP-01 challenge against a Traefik it cannot reach, and it cannot issue a wildcard over HTTP-01 at all; wildcards always need DNS-01. So every cluster gets one wildcard certificate from cert-manager through the DNS provider's API, and each instance serves it as its default:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-example-com
  namespace: traefik
spec:
  secretName: wildcard-example-com
  issuerRef:
    group: cert-manager.io
    kind: ClusterIssuer
    name: letsencrypt-prod
  dnsNames:
    - "*.example.com"
    - "example.com"
```

`tlsStore.default.defaultCertificate.secretName` in the public values above renders a `TLSStore` named `default` in the `traefik` namespace, holding the certificate Traefik serves when no Ingress certificate matches the SNI. Only the store named `default` is used, and the docs are specific that you cannot have two of them in different namespaces. So the store is one object, rendered by the public release, and the internal instance reads it through its own CRD provider, which the chart enables by default; that is why the internal values file can leave `tlsStore` out. If both releases set `tlsStore.default`, the second `helm install` fails, because Helm refuses to take over an object that belongs to another release. Argo CD's chart values that put it on the internal instance look like this; since the hostname is under the wildcard, the issuer annotation and the `tls` block can go:

```yaml
# Layout of the argo-cd chart I run. Current charts use a single
# server.ingress.hostname and a boolean tls; check your values.yaml.
server:
  ingress:
    enabled: true
    ingressClassName: traefik-internal
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt
    hosts:
      - argocd.example.com
    tls:
      - secretName: argocd-ssl
        hosts:
          - argocd.example.com
```

## Three tiers: public, protected, private

One environment I run has needed a middle tier since July: hostnames that must be reachable from the office network, but from nowhere else. That cluster runs three instances, `traefik-public`, `traefik-protected` and `traefik-private`, from the same chart, three values files, all in `traefik`. Each has `nameOverride` and `instanceLabelOverride` set to its own name, two replicas, access logs on, PROXY protocol on both entrypoints, its own `providers.kubernetesIngress.ingressClass`, `publishedService.enabled: true` and `allowExternalNameServices: true`. All three also carry the same `tlsStore.default`, which only works because Argo CD renders the chart and applies the objects itself; nothing checks release ownership, and the Applications just pick up a `SharedResourceWarning` condition naming the other owner. With plain `helm install` the second release would fail, so set it in one file.

What differs is the Service and one Middleware. Public is a `LoadBalancer` with a public IP (`load-balancer.hetzner.cloud/disable-public-network: "false"`), class `traefik-public`, requests of 33m CPU and 333Mi memory, limits of 100m and 1000Mi. Protected is the same shape with class `traefik-protected`, plus an IP allow-list on its routes. Private is a `LoadBalancer` with the public network disabled and `use-private-ip: "true"`, class `traefik-private`, and the largest resources of the three (requests 111m and 700Mi, limits 333m and 2.4Gi), because it carries most of the hostnames. Private is also the one with `ingressClass.isDefaultClass: true`: most workloads on that cluster are internal, so an Ingress that forgets its class should land on the private network, not the internet. That default should exist on exactly one instance. Kubernetes lets you create a second, but from then on the admission controller rejects every new Ingress without an `ingressClassName`, so the first symptom is an `apply` failing on an Ingress that has nothing to do with Traefik.

Private also publishes a second Service, because its main Service is a `LoadBalancer` and the mesh resource needs a ClusterIP:

```yaml
# Ports are opt-in per Service via expose.<name>: expose web and websecure
# on the internal ClusterIP Service too, keep default for the LoadBalancer.
service:
  additionalServices:
    internal:
      enabled: true
      spec:
        type: ClusterIP
ports:
  web:
    expose:
      default: true
      internal: true
  websecure:
    expose:
      default: true
      internal: true
```

The extra Service is named `<fullname>-internal`. The allow-list is a Middleware in the `traefik` namespace. Traefik v3 renamed `IPWhiteList` to `IPAllowList` with no change to the configuration, so use the new name:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: office
  namespace: traefik
spec:
  ipAllowList:
    sourceRange:
      - 198.51.100.0/24
```

Behind a load balancer with PROXY protocol, the source IP Traefik checks is the real client, which is why the PROXY protocol setup above has to be right before this middleware means anything.

There are two ways to attach it. The way I run in production is per Ingress, with the annotation `traefik.ingress.kubernetes.io/router.middlewares: traefik-office@kubernetescrd`; CRD middlewares are referenced as `<namespace>-<name>@kubernetescrd`. The other way, which the entrypoints docs describe and I have not run in production, is `ports.websecure.http.middlewares: ["traefik-office@kubernetescrd"]` in the protected instance's values, which renders the equivalent of `--entrypoints.websecure.http.middlewares=traefik-office@kubernetescrd` and prepends the middleware to every router on that entrypoint. For a protected tier that is the better shape: it turns the allow-list from a policy back into something closer to a boundary, since no Ingress on that instance can forget it. Set it on `web` too if the protected instance answers plain HTTP.

Is a third tier worth it? Only when there is a real population of hostnames that must be reachable from a fixed set of networks but not by everyone. Two instances cover most clusters I run. The protected tier earns its place on one.

## Set it up yourself

The minimum is a namespace, the chart repo, and two installs.

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
kubectl create namespace traefik

helm install traefik traefik/traefik -n traefik --version 41.6.0 -f values-public.yaml
helm install traefik-internal traefik/traefik -n traefik --version 41.6.0 -f values-internal.yaml
```

`values-internal.yaml` is the internal file above, plus `ingressRoute.dashboard.enabled: true` if you want the dashboard check at the end. For `values-public.yaml` on a cloud cluster, the minimum is the class and the published service:

```yaml
instanceLabelOverride: traefik
deployment:
  replicas: 2
ingressClass:
  enabled: true
  name: traefik
  isDefaultClass: true
providers:
  kubernetesIngress:
    ingressClass: traefik
    publishedService:
      enabled: true
```

Then a test workload and one Ingress per class:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  selector:
    app: whoami
  ports:
    - port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-public
spec:
  ingressClassName: traefik
  rules:
    - host: whoami.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-internal
spec:
  ingressClassName: traefik-internal
  rules:
    - host: whoami.internal.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
```

`kubectl get ingress` shows which instance picked each one up. `CLASS` is what you asked for; `ADDRESS` is what `publishedService` copied from that instance's Service: the load balancer IP on the public one, empty on the internal one. Empty on the public one means it did not claim the Ingress or has no published Service. If both show the same address, or the public one flickers between the load balancer IP and empty, both instances are claiming both Ingresses and one is missing its `ingressclass` flag. From a mesh peer, `curl -k https://whoami.internal.example.com` returns the whoami page; from anywhere else the name should not resolve. The `-k` is because neither walkthrough file sets `tlsStore` yet, so you get Traefik's generated certificate. The dashboard on the 8080 port-forward lists the routers that instance loaded, which for the Ingresses is only its own.

## Things that bite

**The chart defaults `isDefaultClass` to true, so the second release you install is a second default unless you turn it off.** The API server then refuses every new Ingress without an `ingressClassName`, cluster-wide. Run `kubectl get ingressclass -o yaml` and look for `ingressclass.kubernetes.io/is-default-class: "true"` on more than one object.

**If you run this under Argo CD, expect drift that is not yours.** My Applications carry `ignoreDifferences` for the Service's `/spec/clusterIP` and `/spec/clusterIPs`, the Deployment's checksum annotations, the ClusterRole's `/rules`, the ClusterRoleBinding's `/subjects`, the dashboard IngressRoute's `/spec` and the PodDisruptionBudget's `/spec`. None of it is a problem; all of it shows OutOfSync until you tell Argo CD to ignore it.

## Ending notes and links

The whole pattern is one flag and one Service type. `--providers.kubernetesingress.ingressclass` gives each instance its own set of Ingresses, and `service.spec.type: ClusterIP` gives one of them no way in from the internet. Everything else here is plumbing around those two lines. If you have admin UIs behind an auth middleware on a public Traefik today, the second instance is an afternoon's work, and the Ingresses move over one `ingressClassName` at a time.

Links:

- [Traefik Kubernetes Ingress provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/): `ingressClass`, `publishedService`
- [Traefik Kubernetes CRD provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-crd/): its own `ingressClass`
- [EntryPoints](https://doc.traefik.io/traefik/reference/install-configuration/entrypoints/): `http.middlewares`, PROXY protocol
- [Ingress annotations](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress/)
- [IPAllowList middleware](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/ipallowlist/) and the [v2 to v3 migration notes](https://doc.traefik.io/traefik/migrate/v2-to-v3-details/)
- [TLSStore CRD](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/tls/tlsstore/)
- [traefik-helm-chart v41.6.0](https://github.com/traefik/traefik-helm-chart/tree/v41.6.0)
- `shouldProcessIngress` in the [Ingress provider](https://github.com/traefik/traefik/blob/v3.7/pkg/provider/kubernetes/ingress/kubernetes.go) and the [CRD provider](https://github.com/traefik/traefik/blob/v3.7/pkg/provider/kubernetes/crd/kubernetes.go)
- [Kubernetes Ingress concepts](https://kubernetes.io/docs/concepts/services-networking/ingress/#default-ingress-class): the default IngressClass rule
- [argoproj/argo-cd issue 1704](https://github.com/argoproj/argo-cd/issues/1704) and [`health_ingress.go` in gitops-engine](https://github.com/argoproj/gitops-engine/blob/master/pkg/health/health_ingress.go)
- [Hetzner cloud controller manager annotations](https://github.com/hetznercloud/hcloud-cloud-controller-manager/blob/main/internal/annotation/load_balancer.go)
- [NetBird Kubernetes operator](https://github.com/netbirdio/kubernetes-operator) and its [API reference](https://github.com/netbirdio/kubernetes-operator/blob/main/docs/api-reference.md)

Thanks for your time.
