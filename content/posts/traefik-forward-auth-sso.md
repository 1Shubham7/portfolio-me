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

## What Traefik does with a forwardAuth middleware

A forwardAuth middleware makes Traefik send a sub-request to an address of your choosing before it forwards the real request. The rule, per the docs: a 2XX from the service and the original request goes through; anything else and the service's response is returned to the client instead. That second half is the trick. When the auth service answers 302 with a `Location` pointing at Keycloak, Traefik hands that 302 to the browser. The app never sees the request.

The sub-request carries the original request's details as headers: `X-Forwarded-Method`, `X-Forwarded-Proto`, `X-Forwarded-Host`, `X-Forwarded-Uri` and `X-Forwarded-For`. By default it also carries every header the client sent, cookies included, which is how the auth service finds its own session cookie. It is a GET unless `preserveRequestMethod` is on, and it has no body unless `forwardBody` is on, which the docs warn breaks streaming.

```
browser      -> Traefik       GET https://prometheus.ops.example.com/graph
Traefik      -> auth service  GET http://traefik-forward-auth:4181/
                              X-Forwarded-Host: prometheus.ops.example.com
                              X-Forwarded-Uri: /graph
                              X-Forwarded-Proto: https
                              X-Forwarded-Method: GET
                              Cookie: (whatever the browser sent)

no valid session cookie:      auth service answers 302 to Keycloak
                              Traefik returns that 302 to the browser

valid session cookie:         auth service answers 200, X-Forwarded-User: someone@example.com
                              Traefik forwards the original request to Prometheus with that header
```

The last line is `authResponseHeaders` at work: headers named there are copied from the auth service's response onto the request that reaches the app, replacing any the client sent under the same name. The app learns who the user is without ever talking to Keycloak.

The middleware I run:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: traefik-forward-auth
  namespace: traefik
spec:
  forwardAuth:
    address: http://traefik-forward-auth:4181
    authResponseHeaders:
      - X-Forwarded-User
    trustForwardHeader: true
```

`address` is the auth service's ClusterIP Service, plain HTTP inside the cluster. `trustForwardHeader: true` keeps the incoming `X-Forwarded-*` headers on the sub-request, which the auth service needs to know which host and path were asked for and to build the URL Keycloak should send the browser back to. `authRequestHeaders` is the other direction, which of the client's headers reach the sub-request; empty means all, which the cookie needs.

That last field in the manifest has moved under me since I wrote it. The docs now mark `trustForwardHeader` deprecated, to be removed in the next major version. The recommended shape is `forwardedHeaders.trustedIPs` on the entrypoint, so Traefik strips `X-Forwarded-*` headers from anyone not on that list, and then `trustForwardHeader: true` on the middleware anyway. Unset, you get a startup warning and a legacy behaviour where some forwarded headers are removed and others, `X-Forwarded-Prefix` among them, go through untouched. Set it explicitly either way, and keep the auth service a ClusterIP Service with no Ingress of its own: the whole scheme rests on `X-Forwarded-Host` and `X-Forwarded-Uri` describing the real request, so nothing but Traefik should write them.

