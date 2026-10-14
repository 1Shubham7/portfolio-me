---
title: "ExternalDNS: Who Owns the Record"
description: "How ExternalDNS turns an Ingress host into a DNS record: where the hostname comes from, why the target decides between A and CNAME, what the ownership TXT record actually does, and what to check before you switch the policy from upsert-only to sync."
dateString: October 2026
draft: false
tags: ["Kubernetes", "DNS", "DevOps", "SRE", "GitOps"]
weight: 2
cover:
    image: "/blog/external-dns/external-dns.png"
---

Exposing a service from a cluster ends with the same manual step every time: someone opens the DNS provider's dashboard and points a name at the load balancer. The Ingress is in git, the certificate is in git, the record is in a browser tab. ExternalDNS is the kubernetes-sigs controller that moves that last step into the cluster. The setup is one Deployment. The part that deserves real attention is how it decides which records it is allowed to touch, because that is the only thing standing between you and a sync loop that empties a zone.

## What it watches and where the hostname comes from

ExternalDNS is not a DNS server. It reads Kubernetes resources, works out which DNS names they want, and calls a provider's API to make the zone match. The resources it reads are called sources, chosen with `--source`, and the list is long: `service`, `ingress`, the Gateway API routes (`gateway-httproute`, `gateway-grpcroute`, `gateway-tlsroute`, `gateway-tcproute`, `gateway-udproute`), `istio-gateway` and `istio-virtualservice`, `traefik-proxy` for Traefik's IngressRoute CRDs, `crd` for a DNSEndpoint object you write by hand, plus Contour, Gloo, Kong, OpenShift routes, and a few more.

For an Ingress you do not have to annotate anything. The source walks `spec.rules[].host` and `spec.tls[].hosts` and takes every non-empty name it finds.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: traefik
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
```

A Service of type LoadBalancer has no host field, so the name goes in an annotation:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mqtt
  annotations:
    external-dns.kubernetes.io/hostname: mqtt.example.com
    external-dns.kubernetes.io/ttl: "120"
spec:
  type: LoadBalancer
  selector:
    app: mqtt
  ports:
    - port: 1883
```

The same annotation works on an Ingress when you want names beyond the ones in the rules, and it takes a comma-separated list.

One thing to check before copying from an older tutorial: the prefix. Every release before v0.22.0 used `external-dns.alpha.kubernetes.io/`. From v0.22.0 the default is `external-dns.kubernetes.io/`, with no fallback, and the release notes say outright that this change can delete all your DNS records, since an instance running `--policy=sync` stops seeing the old annotations and concludes the records are no longer wanted. If you upgrade with the old prefix in place, pass `--enable-legacy-annotation-prefix` so both are read, or set `--annotation-prefix=external-dns.alpha.kubernetes.io/` to keep the old one, then migrate the manifests.

## The provider is a flag, not an annotation

I wanted to know whether the annotation picks the provider. It does not. The annotation carries the DNS name. The provider is set once, on the controller, and so is everything that scopes what it may touch:

```yaml
containers:
  - name: external-dns
    image: registry.k8s.io/external-dns/external-dns:v0.23.0
    args:
      - --provider=cloudflare
      - --source=ingress
      - --source=service
      - --domain-filter=example.com
      - --registry=txt
      - --txt-owner-id=hetzner-prod
      - --policy=upsert-only
    env:
      - name: CF_API_TOKEN
        valueFrom:
          secretKeyRef:
            name: cloudflare-api-token
            key: token
```

The Cloudflare tutorial's example token has Zone Read and DNS Edit on all zones. Grant those on the one zone in `--domain-filter` instead. `--domain-filter` is a suffix match and can be repeated. If you also pass `--zone-id-filter`, the provider is pinned to the zone IDs you list on top of that. Both flags stop ExternalDNS from looking at records in other zones; the token is what stops it being able to change them if a flag goes missing.

`--policy` has to be there. Before v0.22.0 it defaulted to `sync`, and that release made it required with no default, for the same reason the prefix change was dangerous. More on the two values further down.

Cloudflare, Route53, Google Cloud DNS, Azure DNS, RFC2136, PowerDNS and a dozen others are in-tree. Newer providers, Hetzner DNS included, are webhook providers: a separate container that speaks ExternalDNS's provider API on one side and the vendor's API on the other. The README says the maintainers have not reviewed those providers and you use them at your own risk. That is not a reason to avoid them, but it is a reason to read the webhook's issue tracker before putting it in front of production DNS.

