---
title: "I Ran Traefik in Production for a Year Before I Really Understood It"
description: "The questions I should have asked before copying my first Middleware: how Traefik differs from NGINX beyond the YAML, which traffic it actually sees, what happens to a request between the entrypoint and the pod, why some changes need a restart, what a TLS 1.3 handshake really does with the certificate, and why we run more than one instance per cluster."
dateString: October 2026
draft: false
tags: ["Traefik", "Kubernetes", "TLS", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/traefik/traefik.png"
---

For about a year, Traefik was the thing I copied. A new app needed a hostname, so I took the Ingress from the last app, changed the host and the Service name, and it worked. It needed an IP allow-list, so I found a Middleware that did that on another cluster and pasted it in. Across the customer clusters I run at Obmondo this got me surprisingly far. Traefik is forgiving of people who do not read the docs, and I took full advantage of that.

Then I had to explain our setup to someone outside the team, and I could not get past the YAML. I could say what each object was called. I could not say what happened to a request between the load balancer and the pod, why one change to Traefik needed a rollout and another did not, or what the certificate in the Secret was actually for during the handshake.

So I went back and asked the questions I should have asked at the start. This post is those questions, in the order I asked them. If you have ever copied a Middleware without reading what it does, this is for you.

