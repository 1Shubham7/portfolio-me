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

