---
title: "Migrating from ingress-nginx to Traefik Without Rewriting Your Ingresses"
description: "How to move a cluster off the retired ingress-nginx by running Traefik next to it on the same Ingress objects: what the Kubernetes Ingress NGINX provider translates and what it does not, the parallel run and DNS cutover step by step, the IngressClass you must not let Helm delete, and the things I would tell anyone operating Traefik for the first time."
dateString: October 2026
draft: false
tags: ["Kubernetes", "Traefik", "ingress-nginx", "Ingress", "Migration", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/nginx-to-traefik/nginx-to-traefik.png"
---

[NEED: cover image at static/blog/nginx-to-traefik/nginx-to-traefik.png]

Ingress NGINX was retired in March 2026. The [Kubernetes blog post](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/) from November 2025 said it in two sentences: "Best-effort maintenance will continue until March 2026. Afterward, there will be no further releases, no bugfixes, and no updates to resolve any security vulnerabilities that may be discovered." Nothing stopped working; the post is explicit that existing deployments are not broken and the charts and images stay available. What stopped is anyone fixing it. Seven months later a lot of clusters still put every public hostname through a controller nobody patches, and the usual reason is the size of the job: a few hundred Ingresses, each with its own `nginx.ingress.kubernetes.io/*` annotations, and the only plan on the table is to rewrite all of them for the new controller in one go.

That plan is the wrong one. Three hundred Ingresses rewritten is three hundred routing changes in one pull request, none testable before the controller that serves them exists, with a rollback that reverts the lot while the site is down. `ssl-redirect` and `proxy-body-size` translate in a minute each; a `configuration-snippet` someone pasted in 2021 does not, and under a deadline it gets translated to a guess. Traefik added a feature for exactly this situation, a provider that reads the nginx Ingresses as they are, and it turns the migration from a rewrite into a parallel run: install Traefik next to nginx, point it at the same Ingress objects, prove it host by host on its own IP, move DNS, remove nginx, and only then convert annotations, at whatever pace suits you. The change that touches production is one DNS record, and the rollback is the same record pointed back.

To be clear about where I stand: I run Traefik on every cluster I operate and have for years, and I have set up ingress-nginx before, but I have not migrated a cluster from one to the other. The procedure below comes from Traefik's documentation, the Helm chart's own migration example and the write-ups of people who have done it, cited as I go. The operating advice is mine.

