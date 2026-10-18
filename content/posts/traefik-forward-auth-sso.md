---
title: "Adding SSO to Apps Without Built-in Login, with Keycloak and Traefik forwardAuth"
description: "How I put Keycloak login in front of Prometheus, Alertmanager, the Traefik dashboard and other UIs with no login of their own: what Traefik's forwardAuth sends and does with the answer, the Middleware and annotation that make up the integration, and the ClusterRoles of URL patterns that decide which group sees which hostname."
dateString: October 2026
draft: false
tags: ["Kubernetes", "Traefik", "Keycloak", "SSO", "OIDC", "Security", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/traefik-forward-auth-sso/traefik-forward-auth-sso.png"
---

[NEED: cover image at static/blog/traefik-forward-auth-sso/traefik-forward-auth-sso.png]

Prometheus has no login. Neither does Alertmanager, nor the Traefik dashboard, nor the whoami page I keep around to test routing. Graylog and OpenCost do have one, with their own user database that somebody has to keep in step with who actually works here. For a while the clusters I run had the usual mix in front of these: a basicAuth middleware with one shared password, an IP allow-list for the office, and in some cases nothing at all. The allow-list broke the first time someone worked from home. None of it records who opened what.

Since early 2023 the answer on two environments I run has been Traefik's forwardAuth middleware with a small OIDC service behind it that talks to Keycloak; today 13 Ingresses on one environment and 21 on the other sit behind it. The apps did not change. Traefik asks the auth service "is this request allowed?" before every request, the service sends unknown browsers to Keycloak, sets a signed cookie on the way back, and answers 200 for the rest of the session. One annotation on the Ingress is the whole integration. Teaching six apps OAuth themselves would have been six integrations and six slightly different logouts.

The first half of this post is that plumbing. The second half is the part most write-ups stop before: once everybody can log in, who is allowed to see what.

