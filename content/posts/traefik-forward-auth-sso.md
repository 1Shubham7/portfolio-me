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

## One annotation per Ingress

A Middleware from the Kubernetes CRD provider is referenced by Traefik's internal name for it: namespace and name joined with a hyphen, followed by `@kubernetescrd`. The Ingress annotation takes a comma-separated list of those. The real Prometheus Ingress, sanitised:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-k8s
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    traefik.ingress.kubernetes.io/router.middlewares: traefik-traefik-forward-auth@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: prometheus.ops.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-k8s
                port:
                  name: web
  tls:
    - hosts:
        - prometheus.ops.example.com
      secretName: prometheus-tls
```

The doubled `traefik-traefik-` is not a typo: namespace `traefik`, middleware `traefik-forward-auth`. The Ingress lives in `monitoring` and references a Middleware in `traefik`, and nothing has to be enabled for that; the namespace is simply part of the name the Ingress provider passes through.

The Traefik dashboard, exposed through a Service on port 9000, chains the auth middleware in front of a redirect to `/dashboard/`:

```yaml
metadata:
  name: traefik-dashboard
  namespace: traefik
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: traefik-traefik-forward-auth@kubernetescrd,traefik-traefik-dashboard-redirect@kubernetescrd
```

Middlewares run in the order the router declares them, so here the auth check comes first. An `ipAllowList` goes before it if blocked networks should never see a login page; a `stripPrefix` or a redirect goes after, so it only runs for requests that passed.

On an IngressRoute the same middleware is attached by name and namespace:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: whoami-auth
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`whoami-auth.ops.example.com`)
      kind: Rule
      middlewares:
        - name: traefik-forward-auth
          namespace: traefik
      services:
        - name: whoami
          port: 80
```

`namespace` can be omitted when the Middleware is in the IngressRoute's own namespace; otherwise the CRD provider's `allowCrossNamespace` option, default `false` and IngressRoute-only, has to be on.

For apps installed from a chart, Graylog and OpenCost in my case, the annotation goes into the chart's `ingress.annotations` and nothing else changes. The whoami page at `whoami-auth.ops.example.com` is the permanent canary on both environments: after a login it prints the request headers it received, so `X-Forwarded-User: someone@example.com` on that page proves the login, the cookie and the header pass-through in one look.

## The auth service

The service is [mesosphere/traefik-forward-auth](https://github.com/mesosphere/traefik-forward-auth), a partial rewrite of thomseddon's original for any OIDC provider with issuer discovery, plus Kubernetes-flavoured RBAC. I run image `v3.3.0` from the project's own Helm chart, `traefik-forward-auth` version 0.3.10 from `https://mesosphere.github.io/charts/staging`, in the `traefik` namespace, one replica, as a ClusterIP Service on port 4181. It is a tiny Go process: my requests are 5m to 10m CPU and 20Mi to 32Mi memory, with memory limits of 64Mi to 80Mi.

The project moves slowly, which is why the tag is pinned. The chart's last commit is from August 2023 and its default image tag is 3.1.0; the service released nothing between February 2022 and September 2024, then twice in 2026. Nothing from 3.1.0 to 3.3.1 adds or changes a flag: v3.3.0 is a race-condition fix in the user info cache plus dependency bumps, and v3.3.1 canonicalises a trailing dot in the forwarded Host.

The values, sanitised:

```yaml
image:
  tag: v3.3.0
traefikForwardAuth:
  enabled: true
  clientId: traefik-forward-auth
  clientSecret:
    valueFrom:
      secretKeyRef:
        name: traefik-forward-auth-keycloak
        key: client-secret
  oidcUri: https://keycloak.example.com/auth/realms/example
  enableRBAC: true
env:
  GROUP_CLAIM_PREFIX: ""
  ENABLE_V3_URL_PATTERN_MATCHING: "true"
replicaCount: 1
```

`clientId`, `oidcUri` and the user cookie name (`_forward_auth_name`, a display cookie separate from the session cookie below) go into a ConfigMap mounted as `/etc/traefik-forward-auth/config/config.ini`; `CLIENT_SECRET` is an env var read from the Secret you point at. The two secrets the service itself requires, `SECRET` for signing the session cookie and `ENCRYPTION_KEY` for encrypting it, are generated by pre-install hook Jobs into Secrets named `traefik-forward-auth-secret` and `traefik-forward-auth-session-key`. There is no pre-upgrade hook, so they are created once and survive `helm upgrade`; post-delete Jobs remove them on uninstall. Argo CD runs pre-install hooks as PreSync hooks on every sync, and on my clusters the signing secret has stayed stable through that too. So a restart or an image bump logs nobody out, and a reinstall logs everybody out. `enableRBAC: true` becomes the `--enable-rbac` argument, and anything under `env` is passed straight through.

`oidcUri` has to be the issuer URL, the one that serves `/.well-known/openid-configuration`; the service uses discovery. The session cookie is called `_forward_auth`, lives for 43200 seconds (12 hours) by default, and is marked Secure unless you set `INSECURE_COOKIE`.

## The Keycloak client and the callback

On the Keycloak side there is one client in the realm, Client ID `traefik-forward-auth`, with Client authentication on and Standard flow enabled. Keycloak's docs call that a confidential client; it gets a Credentials tab with a generated secret, which goes in the Kubernetes Secret:

```bash
kubectl -n traefik create secret generic traefik-forward-auth-keycloak \
  --from-literal=client-secret='<from the Credentials tab>'
```

In my repos that object is a SealedSecret, so the ciphertext sits in Git next to the chart values and Argo CD applies both. Any OIDC provider works in Keycloak's place; nothing in the service is Keycloak-specific.

The part that catches people is the redirect URI. I do not set `AUTH_HOST` and the auth service has no Ingress of its own, so the OIDC callback is served on every protected hostname under the service's callback path, `/_oauth` by default. A login that starts on `prometheus.ops.example.com` comes back to `https://prometheus.ops.example.com/_oauth`; Traefik runs the middleware on that request like any other, the auth service recognises its own path, exchanges the code for a token, sets the cookie and sends the browser back where it was going. So every protected hostname is a redirect URI Keycloak has to accept.

Keycloak compares redirect URIs with exact, case-sensitive string matching, and the only wildcard it understands is a trailing `*`. There is no `https://*.ops.example.com/_oauth`; a `*` in the host part is a literal character and matches nothing. So Valid redirect URIs is one entry per hostname, and adding an app to SSO is two commits: the annotation on the Ingress and the entry in Keycloak. If the list bothers you, `AUTH_HOST` is the alternative: every login returns to one hostname and one redirect URI, with `COOKIE_DOMAIN` set so the cookie covers the apps. I have not needed it.

For authorisation the token has to carry the user's groups: a protocol mapper on the client of type Group Membership, token claim name `groups`, Full group path off. Its help text: include the full path to the group, `/top/level1/level2`, or just the group name; it defaults to on. Off, a user in the `sre` group gets `"groups": ["sre"]` in the token, the bare name the ClusterRoleBindings below use.

## Who may see what

Once everybody with a Keycloak account can log in, the question is which of them should see the Traefik dashboard. With `--enable-rbac` the auth service watches Kubernetes ClusterRoles and ClusterRoleBindings and allows a request only when the user, or one of their groups, is bound to a role with a matching rule. The real objects, sanitised:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sso-rbac-developer
rules:
  - nonResourceURLs:
      - ~^https?://whoami-auth\.ops\.example\.com/
      - ~^https?://prometheus\.ops\.example\.com/
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sso-rbac-developer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: sso-rbac-developer
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: developers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sso-rbac-sre
rules:
  - nonResourceURLs:
      - ~^https?://whoami-auth\.ops\.example\.com/
      - ~^https?://traefik\.ops\.example\.com/
      - ~^https?://prometheus\.ops\.example\.com/
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sso-rbac-sre
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: sso-rbac-sre
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: sre
```

Developers see whoami and Prometheus. SREs also get the Traefik dashboard. A `kind: User` subject with `name: someone@example.com` grants one person the same way; the user's name on this side is the email from the token.

These are real Kubernetes objects, and the API server evaluates the bindings like any other. They grant nothing there, though: an API request path is `/api/v1/...` or `/healthz`, and Kubernetes matches a `nonResourceURLs` entry by exact string or a trailing `*`, so an entry that starts with `~^https?://` can never match one. That holds even if your API server authenticates against the same Keycloak and knows a group called `developers`. What the objects are is a typed, validated, GitOps-friendly place to keep URL grants next to the Ingress they protect.

How the service evaluates them, from `internal/authorization/rbac/rbac.go`: it keeps the ClusterRoleBindings whose subjects match, a `User` whose name equals the user's or a `Group` that is one of the user's groups, looks up each one's `roleRef.name` as a ClusterRole, and allows the request on the first rule where both the verb and a URL match. An entry in `nonResourceURLs` is interpreted by its first character. A `~` prefix means the rest is a regular expression matched against `scheme://host/path` with the query string dropped; it is not anchored for you, hence the `^` in mine, and the trailing `/` means any path on that host. An entry starting with `/` is a path-only wildcard match, kept for backward compatibility. Anything else is a full-URL wildcard match, and `ENABLE_V3_URL_PATTERN_MATCHING` governs those: on, a single `*` matches within one path segment and `**` across segments; off, only a trailing `*` means anything. The `~` form does not depend on the flag; I set it anyway.

The verb is compared case-insensitively against the request's method, and `*` matches any. The method is the real one, copied from `X-Forwarded-Method` before the check, so `verbs: [get]` covers GETs and nothing else, and an app that POSTs from the browser needs `post` in the list or a `*`. Mine get away with `get` because the RBAC middleware only fronts read-only UIs: the Prometheus UI, the whoami canary and, on one cluster, the Traefik dashboard. Graylog and OpenCost both POST from the browser and I have not tested either behind RBAC; they sit behind the authentication-only middleware from the two-instance setup below.

An authenticated user with no matching rule gets a 401 with the body `Not Authorized`, straight from `http.Error` in `internal/handlers/server.go`. I had it in my notes as a 404 for a long time; the code says 401. The reason is in the pod logs, but not at the default `warn` level, because the denial is logged at info:

```
user someone@example.com is not authorized to `GET` in https://traefik.ops.example.com/dashboard/
```

One level further down, at debug, the line before it prints the user and the group list the check ran with. `LOG_LEVEL` is the chart's `traefikForwardAuth.logLevel` value; I set it to `debug` while onboarding a new app and back to `warn` after.

That group list is where `GROUP_CLAIM_PREFIX` matters. Before the check, the service prefixes every group from the token with that value and adds `system:authenticated`. The default prefix is `oidc:`, so with Keycloak's bare group names the service looks for a binding to `oidc:sre`, your binding says `sre`, and every SRE is denied with a debug line showing `groups: [system:authenticated oidc:sre]`. An empty prefix makes both sides the same string; keeping the default and writing `oidc:sre` in the bindings works just as well.

### Rolling RBAC out one hostname at a time

Turning on RBAC for every Ingress at once closes every one of them until its role exists. On one environment I avoided that by running two instances of the service from the same chart and the same Keycloak client: a release `traefik-forward-auth` with `enableRBAC: false`, where any logged-in user gets in, and `traefik-forward-auth-rbac` with `enableRBAC: true`, each with its own Middleware pointing at its own Service. An Ingress chooses by the middleware name it carries, so moving a hostname is a one-line change and the hostnames still on the open middleware keep working. Both serve `/_oauth` on the protected host, so the redirect URIs do not change.

