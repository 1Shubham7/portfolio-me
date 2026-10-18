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

## One Traefik, two Ingress providers

Traefik has always had a Kubernetes Ingress provider for plain Ingress objects and the `traefik.ingress.kubernetes.io/*` annotations. The [Kubernetes Ingress NGINX provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress-nginx/) is a second one in the same process. It looks for IngressClasses whose `spec.controller` is `k8s.io/ingress-nginx` (the `controllerClass` option; `ingressClass` defaults to `nginx`), takes every Ingress that names one of them, and builds Traefik routers from them, translating the nginx annotations into Traefik's own middlewares as the configuration loads. The Ingress objects are read, not rewritten.

It arrived as an experimental feature in [Traefik Proxy v3.5](https://traefik.io/blog/traefik-proxy-v3-5) on 24 July 2025, with the announcement putting it plainly: "This isn't about building 100% compatibility with every NGINX annotation." v3.6.2 deprecated the experimental flag, and the current docs do not mark it experimental at all. The Helm chart added it in 37.4.0 as `providers.kubernetesIngressNginx` and renamed the key to `providers.kubernetesIngressNGINX` in 40.0.0, so older snippets, Traefik's own migration guide included, may carry the old spelling. The [chart's own example](https://github.com/traefik/traefik-helm-chart/blob/master/EXAMPLES.md) is the whole method in miniature: whoami behind one `ingressClassName: nginx` Ingress, curled first through nginx and then through a Traefik installed beside it with `providers.kubernetesIngressNGINX.enabled=true`, with the conclusion: "The same Ingress resource is now served by both NGINX and Traefik!"

Which annotations survive the translation decides whether this works for your cluster, and the [annotation reference](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress-nginx/) answers it row by row. The common ones are there: `ssl-redirect` and `force-ssl-redirect`, `rewrite-target` and `use-regex`, the permanent and temporal redirects, `auth-url`, `auth-type` with `auth-secret`, `auth-tls-secret`, `whitelist-source-range` and its newer spelling `allowlist-source-range`, `proxy-body-size` with the buffering and timeout family, `affinity` and `session-cookie-*`, `backend-protocol`, `enable-cors`, `limit-rps`, `limit-rpm` and `limit-connections`, `canary-*`, `custom-http-errors`, `default-backend`, `ssl-passthrough`, and the snippet annotations, each for a fixed list of directives.

Most rows carry a note, and the notes are where migrations go sideways. Rate limits return 429 where nginx returned 503. `affinity-mode: balanced` is not supported, only `persistent`. `backend-protocol` takes neither `FCGI` nor `AUTO_HTTP`. CORS is "partial support". `ssl-passthrough` has "some differences in SNI/default backend handling". `load-balance` is "not implemented; silently ignored". The snippets accept `add_header`, `proxy_set_header`, `more_set_headers`, `set`, `if`, `return`, `rewrite` and a few others, nothing else, and only with the provider's `allowSnippetAnnotations` on. Then the explicitly unsupported list: `denylist-source-range`, `limit-rate`, `global-rate-limit`, the mirroring annotations, the `proxy-redirect-*` and `proxy-cookie-*` rewrites, `satisfy`, `stream-snippet`, the tracing toggles and the per-Ingress ModSecurity annotations.

Regex paths deserve their own test. Two forum threads from January 2026, both on Traefik 3.6.6, report `use-regex` paths that routed under nginx and not under Traefik: [one](https://community.traefik.io/t/kubernetesingressnginx-extension-not-working-with-nginx-ingress-kubernetes-io-use-regex/29570) shows the router Traefik built from `/api/v\d{1}/users`, with the backslash doubled and a 404 where nginx had routed, [the other](https://community.traefik.io/t/nginx-ingress-compatibility-nginx-ingress-kubernetes-io-use-regex-annotation-not-working/29602) a `/test/[0-9]+` path that simply stopped matching. I have not reproduced either and 3.7 may have fixed them, but any host with `use-regex: "true"` goes first on the verification list.

The provider translates Ingress annotations. It does not translate the controller. Everything ingress-nginx did through its ConfigMap, body size, proxy timeouts, `use-proxy-protocol`, HSTS, ciphers, access log format, has to be set again on Traefik's side, some of it as `kubernetesIngressNGINX` options with nginx-flavoured names (`proxyBodySize`, `proxyReadTimeout`, `customHTTPErrors`), the rest as entrypoint, TLS option and access log settings. Traefik's [migration guide](https://doc.traefik.io/traefik/migrate/nginx-to-traefik/) has that mapping as its step zero. Two defaults differ in ways nothing in the cluster reports. ingress-nginx redirects HTTP to HTTPS with a 308 for any Ingress that has TLS; Traefik redirects nothing unless the entrypoint says so, and one [production write-up](https://dev.to/pavel-hostim/migrating-a-production-cluster-off-ingress-nginx-to-traefik-v3-19ij) from August 2026 found that out from monitoring after the cutover: "Every app in the cluster answered on port 80 in cleartext." The same post met the second: a hostname with no certificate of its own gets Traefik's generated self-signed certificate, and "Nothing in the control plane reports an error; the failure is a browser warning." Both are one values key each, in the values file below.

## The order that works

This assumes ingress-nginx installed by its Helm chart as release `ingress-nginx` in namespace `ingress-nginx`, exposed by a `LoadBalancer` Service, with cert-manager issuing certificates and ExternalDNS writing records. Adjust the names; the order does not change.

### Take inventory

Before installing anything, find out which annotations you actually have:

```bash
kubectl get ingress -A -o json | jq -r '
  [.items[].metadata.annotations // {} | keys[] | select(startswith("nginx.ingress.kubernetes.io/"))]
  | group_by(.) | map("\(length)\t\(.[0])") | .[]' | sort -rn
```

One line per distinct annotation with a count. Check each against the reference page, and dump the ingress-nginx ConfigMap (`kubectl -n ingress-nginx get configmap ingress-nginx-controller -o yaml`) for the global settings that need a home on the Traefik side.

### Install Traefik next to nginx

Traefik gets its own namespace, its own `LoadBalancer` Service and so its own public IP. This is the values file I run on cloud clusters, with the NGINX provider and the default certificate added:

```yaml
ports:
  web:
    redirections:
      entryPoint:
        to: websecure
        scheme: https
        permanent: true
  websecure:
    http:
      tls:
        enabled: true
providers:
  kubernetesCRD:
    enabled: true
    allowCrossNamespace: true
  kubernetesIngress:
    publishedService:
      enabled: true
  kubernetesIngressNGINX:
    enabled: true
    publishService:
      enabled: false
additionalArguments:
  - "--providers.kubernetesingress.ingressclass=traefik"
  - "--metrics.prometheus=true"
metrics:
  prometheus:
    serviceMonitor:
      enabled: true
accessLog:
  enabled: true
tlsStore:
  default:
    defaultCertificate:
      secretName: wildcard-example-com
deployment:
  replicas: 2
podDisruptionBudget:
  enabled: true
  maxUnavailable: 1
```

```bash
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n traefik --create-namespace --version 41.6.0 -f values.yaml
```

Three of those lines are the migration. `ports.web.redirections` is the HTTP to HTTPS redirect nginx did for you. `tlsStore.default.defaultCertificate` is what any hostname without a Secret of its own gets served instead of the self-signed one; it points at a wildcard from cert-manager, which has to be DNS-01, since Let's Encrypt will not issue wildcards over HTTP-01. And the `ingressclass` argument pins Traefik's standard Ingress provider to the `traefik` class, so the two providers in this one process never both claim an Ingress; the provider docs warn that otherwise the NGINX provider "may lead to duplicated routers if you are also using the standard Kubernetes Ingress provider". Two unpinned controllers fight over Ingress status on every reconcile, and a migration has two by design. From here, an Ingress with `ingressClassName: nginx` is served by nginx and by Traefik's NGINX provider, an Ingress with `ingressClassName: traefik` by Traefik's standard provider, and nothing twice by Traefik.

### Decide who writes the status

Every Ingress has a `status.loadBalancer.ingress` field, and both controllers want their Service's address in it: ingress-nginx through `--publish-service`, Traefik's NGINX provider through `publishService`. If both write, the migration guide says, "Each controller overwrites the other in a tight reconciliation loop, with no error reported in the logs", while "Routing itself is not affected." The damage is in whatever reads the field. ExternalDNS does: its [Ingress source](https://kubernetes-sigs.github.io/external-dns/latest/docs/sources/ingress/) walks `status.loadBalancer.ingress`, "adding each non-empty `ip` and `hostname`", so a status holding both addresses is two A records for every hostname, and a status that flaps is records that flap.

So `publishService.enabled: false` on the NGINX provider, the chart's default, spelled out in the values above anyway. nginx stays the only writer, ExternalDNS changes nothing, and Traefik is reachable only by its IP, which is what verification needs. The guide's alternative, a transitional IngressClass for nginx so the controllers never share an Ingress, means editing every Ingress, the thing this approach avoids.

The standard provider has the same switch under a different spelling, `publishedService`, and there it stays on. Without it the status of every traefik-class Ingress is empty, `kubectl get ingress` shows no ADDRESS, and Argo CD marks every application behind them Progressing forever, since its Ingress health check is that one field having an entry (argoproj/argo-cd issue 1704).

### Verify through Traefik's own address

The hostnames still resolve to nginx. Tell curl where to connect instead:

```bash
TRAEFIK_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' \
  --resolve app.example.com:443:$TRAEFIK_IP https://app.example.com/
```

Do not add `-k`. The Secrets named in each Ingress's `spec.tls` are read by both controllers, so Traefik should present the same Let's Encrypt certificate nginx does, and a verification failure means a host that fell through to the default certificate, which is a finding. Loop over every host in the inventory, then exercise the annotations with notes: an `auth-url` login, a `proxy-body-size` upload, a `whitelist-source-range` from outside it, every `use-regex` path, a websocket. HTTP-01 challenges do not go through Traefik in this phase; Let's Encrypt connects to whatever DNS says, which is still nginx, so cert-manager keeps renewing through nginx and Traefik serves the resulting Secrets.

Then open the dashboard. The chart does not expose it (`ingressRoute.dashboard.enabled` is `false`); set it to `true`, `kubectl -n traefik port-forward deploy/traefik 8080:8080`, open `http://localhost:8080/dashboard/`. The HTTP routers page lists every router Traefik built and the provider each came from. A host curl could not reach is either missing from that list, which points at the IngressClass, or present with a rule you did not expect, which points at the annotation reference. It is the first place I look when an Ingress does not route, before any log.

### Move DNS

Lower the TTLs a day before. Then point the records at Traefik's IP and watch the access logs on both sides until nginx goes quiet. The guide's advice is to "keep NGINX running for at least 24-48 hours before uninstalling", because some resolvers ignore TTLs, and a rollback in that window is the record pointed back.

How the records move depends on who writes them. Hand-managed: change them. Written by ExternalDNS from Ingress status: the guide suggests leaving Traefik's `publishService` off and pointing nginx's own publishing at Traefik's Service with `controller.publishService.pathOverride: traefik/traefik`, so nginx writes Traefik's address into every status and ExternalDNS follows; removing the override is the rollback. If your load balancer lets you move an IP between Services, that is the third way, and the one the August write-up used with a MetalLB address.

### Keep the `nginx` IngressClass, then remove nginx

This is the step that bites, and the chart's example says so in full: "When uninstalling the NGINX Ingress Controller helm chart, it removes the `nginx` IngressClass. Traefik needs this IngressClass to detect and serve Ingress resources that use `ingressClassName: nginx`. Before uninstalling NGINX, it's recommended to ensure that an IngressClass like this will stay:"

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```

The provider finds the Ingresses through the IngressClass's `spec.controller`, so with the object gone, every Ingress that says `ingressClassName: nginx` stops routing the moment `helm uninstall` returns. Under GitOps, put that manifest in the repository as a resource of its own. Under plain Helm, add `helm.sh/resource-policy: keep` to the IngressClass through `helm upgrade --reuse-values`; the guide is specific that `kubectl annotate` on the live object does not do it, because Helm consults its own release record, not the cluster.

Then, in this order:

```bash
kubectl delete validatingwebhookconfiguration ingress-nginx-admission
helm uninstall ingress-nginx -n ingress-nginx
kubectl get ingressclass nginx
```

The webhook goes first. ingress-nginx registers a validating admission webhook for Ingress objects with `failurePolicy: Fail`, so once its pods are gone the webhook rejects every Ingress create and update in the cluster. The August write-up puts it in one line: delete the namespace and orphan the webhook, and "every Ingress and certificate change in the cluster stops working." If the last command prints the class, the Ingresses are still served, now by Traefik alone. Set `publishService.enabled: true` on the NGINX provider at this point: ADDRESS fills in with Traefik's IP and ExternalDNS has one target per host.

### Move cert-manager off the `nginx` name

Nothing breaks yet, because the solver Ingresses cert-manager creates still say `ingressClassName: nginx` and Traefik serves them. That holds only while the `nginx` IngressClass exists, and the next phase ends by deleting it. cert-manager's [HTTP-01 solver](https://cert-manager.io/docs/configuration/acme/http01/) picks the controller with `solvers[].http01.ingress.ingressClassName`, the field the docs recommend over the older `class`. Change it on every Issuer and ClusterIssuer:

```yaml
solvers:
  - http01:
      ingress:
        ingressClassName: traefik
```

Leaving it unset is worse than leaving it on `nginx`. The docs say that with no class, "all ingress controllers installed in your cluster will serve traffic for the challenge solver", which during a migration is a solver Ingress with two owners. ExternalDNS needs a change only if it runs with `--ingress-class=nginx`.

## Convert at your own pace

The cluster now runs Traefik only, every Ingress is unchanged, and the pressure is off. What remains is moving nginx-class Ingresses to Traefik's own class and middlewares, and a variant of the inventory filter lists what is left:

```bash
kubectl get ingress -A -o json | jq -r '
  .items[]
  | select(any((.metadata.annotations // {}) | keys[]; startswith("nginx.ingress.kubernetes.io/")))
  | "\(.metadata.namespace)/\(.metadata.name)"'
```

Converting means setting `ingressClassName: traefik`, at which point the standard provider takes the Ingress and ignores the nginx annotations, and adding Traefik's equivalents. Middlewares are CRD objects, usually in the application's namespace, attached with the annotation `traefik.ingress.kubernetes.io/router.middlewares` whose value is `<namespace>-<name>@kubernetescrd`, comma-separated for several. Five translations cover most inventories.

**`ssl-redirect` and `force-ssl-redirect`** become nothing. The entrypoint redirection in the values file redirects every request on port 80, so the annotation is deleted. The reference notes the flip side, "Cannot opt-out per route if enabled globally"; a host that must answer plain HTTP needs an entrypoint without the redirect.

**`whitelist-source-range`** becomes an `ipAllowList` middleware, which Traefik v3 renamed from `IPWhiteList` with no other change:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: office
spec:
  ipAllowList:
    sourceRange:
      - 198.51.100.0/24
```

By default it checks the remote address of the connection, so behind a load balancer the source IP is right only with PROXY protocol on; otherwise add `ipStrategy.depth` to read `X-Forwarded-For`.

**`auth-type: basic` with `auth-secret`** becomes a `basicAuth` middleware pointing at the same Secret. The CRD provider accepts any generic Secret with exactly one key and reads it as `htpasswd` lines, so the `auth` key ingress-nginx asked for works as it is; a `kubernetes.io/basic-auth` Secret with `username` and `password` works too.

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: admin-users
spec:
  basicAuth:
    secret: basic-auth
```

**`proxy-body-size`** becomes a `buffering` middleware with `maxRequestBodyBytes`, in bytes, so `50m` is `52428800`, and a request over it gets a 413. The default is the trap. Traefik's own is 0, which the docs define as unlimited, while the NGINX provider carries `proxyBodySize: 1048576` to match nginx's 1m. So an Ingress on the nginx class is capped at 1 MiB whether it says so or not, and the same Ingress on the traefik class is uncapped until you add the middleware.

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: body-50m
spec:
  buffering:
    maxRequestBodyBytes: 52428800
```

**`rewrite-target`** becomes `replacePathRegex`. The nginx idiom of `path: /api(/|$)(.*)` with `rewrite-target: /$2` and `use-regex` is a plain `/api` prefix path plus:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: strip-api
spec:
  replacePathRegex:
    regex: "^/api/(.*)"
    replacement: "/${1}"
```

The docs prefer `${1}` to `$1` because `$1x` is read as a group named `1x`. The original path reaches the backend in an `X-Replaced-Path` header.

Snippets are the residue. Whatever a `configuration-snippet` did that is not in the provider's directive list has to be reproduced as a Traefik middleware (`headers` covers most `add_header` and `proxy_set_header` cases), as a plugin, or in the application. Two more things change when an Ingress leaves the nginx class. An `ExternalName` backend works under the NGINX provider, where `disableSvcExternalName` defaults to false, and stops under the standard one until `providers.kubernetesIngress.allowExternalNameServices: true`. Custom error pages come from `customHTTPErrors` and `defaultBackendService` on the NGINX provider, and from an `errors` middleware once converted.

When the filter prints nothing, set `kubernetesIngressNGINX.enabled: false`, delete the `nginx` IngressClass, and it is over. The August write-up left the provider on afterwards to keep rollback cheap, which costs nothing.

## What I would tell a first-time Traefik operator

Four more things from running Traefik that the procedure had no reason to mention.

**PROXY protocol on both sides or on neither.** Set `--entrypoints.websecure.proxyprotocol=true` on Traefik because you want client IPs behind a cloud load balancer, and the load balancer has to send the header. On Hetzner Cloud that is the Service annotation `load-balancer.hetzner.cloud/uses-proxyprotocol: "true"`, and its default is off. Traefik side only, and every connection is rejected while Traefik waits for a header that never comes, cert-manager's HTTP-01 challenges included.

**Learn the middleware reference format once.** With a middleware called `forward-auth` in namespace `traefik`, the reference is `traefik-forward-auth@kubernetescrd`, and the doubled prefix trips everyone the first time. Traefik v3.7.11 added a `safeNaming` provider option that changes the separator to an underscore, `traefik_forward-auth@kubernetescrd`; it is off by default and warns at startup until set either way, and a reference in the wrong form is not resolved, so the router is silently not created.

**Secrets behind TLS stores and TLS options hot-reload.** Rotating the default certificate, or adding a CA to a client-certificate option, needs no restart.

**On bare metal, the pods own the ports.** With no cloud load balancer I run Traefik as a DaemonSet with hostPort 80 and 443, and the chart's default rolling update, surge one and zero unavailable, cannot work there: the new pod cannot bind a port the old pod still holds. `updateStrategy.rollingUpdate.maxSurge: 0, maxUnavailable: 1` deletes the old pod first.

