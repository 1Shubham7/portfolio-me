---
title: "Kubernetes Audit Logging: Stages, Levels, and Policy Design"
description: "A breakdown of how kube-apiserver audit events actually work - request stages, audit levels, writing policies, and how SREs tune them differently for prod vs dev clusters."
dateString: June 2026
draft: false
tags: ["Kubernetes", "DevOps", "Security", "Compliance", "kube-apiserver", "SRE"]
weight: 1
cover:
    image: "/blog/audit-logging/k8s-audit.png"
---

## Architecture

![K8s Audit Logging Architecture](/blog/audit-logging/architecture.png)

Every single request that hits `kube-apiserver` (eg. `kubectl get pods`) generates an audit event. There's no separate "audit service" sitting beside the apiserver; auditing is a feature built directly into the apiserver's request handling pipeline.

As that request moves through its lifecycle, it passes through up to four **stages**, and the apiserver can emit an audit event at each one.

---

## The four stages

#### 1. RequestReceived

Generated the moment the audit handler receives the request, before it's been processed or authorized.

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "stage": "RequestReceived",
  "requestURI": "/api/v1/namespaces/default/pods",
  "verb": "create",
  "user": { "username": "deploy-bot", "groups": ["system:serviceaccounts"] },
  "requestReceivedTimestamp": "2026-06-23T10:15:30.100000Z"
}
```

This stage exists, but not very usefull in most cases cause it roughly doubles event volume while telling you nothing you don't already get.

### ResponseStarted

Fired once the response headers go out, but before the response body is sent. This only applies to long-running requests - `watch` connections are the main case. A `kubectl get pods -w` or a controller's informer watch will hit this stage, a normal `kubectl get pods` won't.

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "stage": "ResponseStarted",
  "requestURI": "/api/v1/namespaces/default/pods?watch=true",
  "verb": "watch",
  "user": { "username": "system:kube-controller-manager" }
}
```

### ResponseComplete

The response body has been fully sent - nothing more is coming. This is the stage that matters most for almost every use case (security review, compliance evidence, "who deleted this?", etc.), because it carries the complete picture: what was asked, who asked, and what the server actually did about it.

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Request",
  "auditID": "a1b2c3d4-e5f6-7890-abcd-ef0123456789",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/production/pods/my-app-pod",
  "verb": "delete",
  "user": {
    "username": "shubham@company.com",
    "groups": ["developers", "system:authenticated"]
  },
  "sourceIPs": ["10.0.0.50"],
  "objectRef": {
    "resource": "pods",
    "namespace": "production",
    "name": "my-app-pod",
    "apiVersion": "v1"
  },
  "responseStatus": { "code": 200 },
  "requestReceivedTimestamp": "2026-06-23T10:15:30.123456Z",
  "stageTimestamp": "2026-06-23T10:15:30.234567Z"
}
```

### Panic

Generated only when the apiserver hits an internal panic while handling the request. Rare, but useful as a signal of something seriously wrong in the request path - worth never filtering out, regardless of how aggressive your policy is elsewhere.

---

## Audit levels - how much detail gets captured

### None

Nothing is logged. The request is matched by a rule and dropped entirely - no event written at all.

```yaml
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
```

### Metadata

Logs the request metadata - who, what verb, what resource, timestamp, source IP, response code - but never the request or response body.

```json
{
  "kind": "Event",
  "level": "Metadata",
  "stage": "ResponseComplete",
  "verb": "get",
  "user": { "username": "jane.doe", "groups": ["developers"] },
  "objectRef": { "resource": "secrets", "namespace": "production", "name": "db-creds" },
  "responseStatus": { "code": 403, "reason": "Forbidden" }
}
```

For anything that touches secrets - Metadata is the only level you should use - otherwise it will show you the request body or the response body which has the base64 encoded secret which anyone can easily decode. Metadata doesnt show those details.

### Request

Metadata plus the full request body. No response body.

```json
{
  "kind": "Event",
  "level": "Request",
  "stage": "ResponseComplete",
  "verb": "create",
  "user": { "username": "ci-bot" },
  "objectRef": { "resource": "configmaps", "namespace": "kube-system", "name": "feature-flags" },
  "requestObject": {
    "kind": "ConfigMap",
    "apiVersion": "v1",
    "metadata": { "name": "feature-flags", "namespace": "kube-system" },
    "data": { "new-checkout-flow": "true" }
  }
}
```

You see what was *sent*, but not what the server returned.

### RequestResponse

The most verbose - metadata, request body, and response body. Useful for high-value resources where you want to know exactly what changed and exactly what the server's final state ended up being, but expensive in storage if we apply it broadly.

```json
{
  "kind": "Event",
  "level": "RequestResponse",
  "stage": "ResponseComplete",
  "verb": "create",
  "user": { "username": "kubernetes-admin", "groups": ["system:masters"] },
  "objectRef": { "resource": "rolebindings", "namespace": "production", "name": "ci-deploy-binding" },
  "requestObject": {
    "kind": "RoleBinding",
    "roleRef": { "kind": "Role", "name": "deploy-role" },
    "subjects": [{ "kind": "ServiceAccount", "name": "ci-bot", "namespace": "production" }]
  },
  "responseObject": {
    "kind": "RoleBinding",
    "metadata": { "uid": "rb-uid-456", "resourceVersion": "9821" }
  }
}
```

This is the level you want for RBAC changes, NetworkPolicy changes, and anything where "what did it end up looking like after" matters as much as "what was requested."

---

## Writing an audit policy

A policy is just a list of rules evaluated **top to bottom** - the first matching rule decides the level for that request, and evaluation stops there. Order matters a lot here; a broad rule placed too early will swallow requests you meant to catch with a more specific rule below it.

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  # Drop noisy, low-value system traffic first
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]

  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get", "list", "watch"]

  - level: None
    nonResourceURLs:
      - "/healthz*"
      - "/readyz*"
      - "/metrics"

  # Never write secret bodies, no matter the verb
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]

  # Full detail for the resources that actually matter for security review
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
      - group: "networking.k8s.io"
        resources: ["networkpolicies"]
      - group: ""
        resources: ["pods/exec", "pods/attach"]

  # Request body for general mutations elsewhere
  - level: Request
    verbs: ["create", "update", "patch", "delete"]

  # Catch-all
  - level: Metadata
```

A rule can filter on any combination of `users`, `userGroups`, `verbs`, `resources` (with `group` / `resources` / `resourceNames`), `namespaces`, and `nonResourceURLs`. The catch all thing at the end is for any other event that doesnt fall into the above criterias.

---

## How to tune this in practice

There isn't one universal policy - what gets logged, and at what level, depends heavily on the cluster's role.

**It depends on environment (prod vs. dev/staging):**
- Production clusters generally run `RequestResponse` on RBAC, NetworkPolicy, and exec/attach - these are the things you need full forensic detail on if something goes wrong.
- Dev and staging clusters usually drop to `Metadata`-only almost everywhere, or even disable auditing for low-value namespaces, since the security stakes are lower and the volume isn't worth the storage cost.

**It depends on compliance requirements:**
- If you're working toward ISO 27001, SOC 2, or NIS2 (relevant to anything classified as an Important/Essential Entity), auditors generally want to see who changed access control and network policy objects, with enough detail to reconstruct the change - which pushes RBAC and NetworkPolicy rules toward `RequestResponse` regardless of environment.
- Without a compliance driver, most teams are comfortable with `Metadata` as the default and only escalate specific resource types.

**It depends on log volume and retention budget:**
- A high-traffic cluster with thousands of requests per second at `RequestResponse` level can generate enormous log volume fast. Teams with tighter storage or shipping budgets push more rules toward `Metadata` and reserve `Request`/`RequestResponse` for a less no. of resource types.

- This is also why `omitStages: ["RequestReceived"]` is used by most of the SRE teams - it's like a "50% volume cut with no loss of information" in most cases, since `ResponseComplete` already contains the request_received timestamp.

**It depends on what's already noisy in your cluster:**
- Health checks (`system:kube-probe`), kube-proxy watches, kubelet node status updates, and reconciler loops (ArgoCD, controller-manager, your GitOps tooling) account for a large fraction of total apiserver traffic. Filtering these to `None` early in the rule list is a good practice - without it, signals will get buried in noise.

So what you should do is:
1. `None` for known system/health noise, placed first
2. `Metadata`- only, forced, for `secrets`.
3. `RequestResponse` for RBAC objects, netpols, and `pods/exec`/`pods/attach`
4. `Request` for general mutating verbs (`create`/`update`/`patch`/`delete`) on everything else
5. `Metadata` catch all at the bottom

---