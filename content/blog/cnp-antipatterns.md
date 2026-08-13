---
title: "CiliumNetworkPolicy Anti-Patterns: 8 Ways Correct-Looking YAML Silently Drops Traffic"
description: "Lessons from writing CiliumNetworkPolicies for the KubeAid-addons chart - toServices port traps, labels Cilium ignores, toFQDNs without a DNS rule, default-deny surprises, and why every namespace needs exactly one default-deny."
dateString: August 2026
draft: false
tags: ["Kubernetes", "Cilium", "Networking", "Security", "DevOps", "SRE"]
weight: 1
cover:
    image: "/blog/cnp/cnp-cover.png"
---

I'm a Certified Kubernetes Administrator, I contribute to CNCF projects, and I help maintain [KubeAid](https://KubeAid.io) and [KubeAid CLI](https://github.com/obmondo/KubeAid-cli).

The goal behind KubeAid-addons was simple to state: only the traffic that should be allowed is allowed, and not just on the clusters we run ourselves but for every open source user too. So we built it as a centralized, opinionated Helm chart that firewalls applications behind a toggle, following GitOps principles. You set `global.netpol.enabled`, flip the per-app switch, commit, and Argo CD applies it. The application-level firewalls are open source and live [here](https://github.com/Obmondo/KubeAid/tree/master/argocd-helm-charts/KubeAid-addons).

Getting there meant writing CiliumNetworkPolicies for a lot of very different applications: databases managed by operators, apps calling external APIs, apps sitting behind an ingress controller, apps with liveness probes, apps sharing a namespace with other apps nobody was firewalling yet.

That's where this post comes from. Every anti-pattern below is something that broke on us, usually silently, usually with YAML that looked completely correct.

## Cilium in one breath

If you already know Cilium, skip ahead. If you don't, here's the short version.

Cilium is an eBPF-based networking, observability and security layer for Kubernetes. It replaces the traditional iptables-based CNI plus kube-proxy setup with an eBPF datapath.

Why that matters is mechanical. In an iptables world, every Service and every policy becomes iptables rules, and the kernel walks those rules linearly, top to bottom, for every packet. With 5,000 Services and 10,000 Pods, kube-proxy can generate hundreds of thousands of rules, and every packet potentially traverses a very long chain (for the programmers reading: that's O(n)). Updates are worse, since a single Service change can force a resync of a large chunk of the ruleset.

Cilium compiles the same information into eBPF programs and BPF maps living in the kernel. A BPF map is a kernel-resident hash table, so instead of walking 10,000 rules the kernel does one hash lookup (hi again programmers, that's O(1) instead of O(n)). That's the performance story.

The part that actually matters for policy isn't performance, though. Cilium didn't just port iptables logic to eBPF, it changed the security model. Policy is not evaluated against IP addresses. Cilium derives a **security identity** from a pod's labels, every pod with the same relevant labels shares one identity, and the policy map is keyed on identity pairs: (source identity -> destination identity, port, protocol).

Cilium also ships Hubble, which exports flow data as logs, metrics and a UI, since the eBPF programs are inspecting every packet anyway. We'll need it later.

## Why CiliumNetworkPolicy and not the Kubernetes-native one

Native `NetworkPolicy` stops at L3/L4: IPs and ports. CiliumNetworkPolicy goes up to L7, so you can allow specific HTTP methods and paths, gRPC services, or Kafka topics.

There's also `CiliumClusterwideNetworkPolicy` for baseline rules across the whole cluster. Native netpols are namespace-scoped, and so is the regular `CiliumNetworkPolicy`, but the clusterwide CRD gives you cluster-wide enforcement.

Native netpols have no notion of hostnames, only CIDR. CNP can allow egress by domain name, resolved dynamically. That's `toFQDNs`, and it has a trap of its own, which we'll get to.

Native netpols can only allow. CNP adds explicit deny, and deny always wins over allow when both match:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: db-access
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: database
  ingress:
    - fromEndpoints:
        - matchLabels:
            role: backend
  ingressDeny:
    - fromEndpoints:
        - matchLabels:
            role: untrusted-client
```

And where native netpols can only reference pods and namespaces via selectors, or raw CIDR blocks, CNP gives you predefined entities: `world`, `host`, `cluster`, `kube-apiserver`, `remote-node`. These come up constantly once you write real policies.

## The anti-patterns

### 1. `toServices` with the Service port instead of the pod's targetPort

You write a rule allowing egress to a database Service. The Service exposes 5432. You write 5432. The traffic gets dropped.

```yaml
egress:
  - toServices:
      - k8sService:
          serviceName: my-database
          namespace: my-namespace
    toPorts:
      - ports:
          - port: "5432"
```

The database here sits behind pgbouncer, which listens on 6432. The Service exposes 5432 and targets 6432. Follow the packet:

1. Pod A sends a request to the Service's ClusterIP on port 5432, because that's the only address it knows. It has no idea what Pod B's IP or port are.
2. eBPF intercepts this at the host side of Pod A's veth. The program looks the Service up in the service map, picks Pod B as the backend, and rewrites the packet. Destination IP becomes Pod B's real IP. Destination port becomes **6432**.
3. eBPF resolves identities: Pod A's IP maps to Pod A's identity, Pod B's IP maps to Pod B's identity.
4. It does the policy map lookup on the post-DNAT tuple: (Pod A identity -> Pod B identity, port 6432, TCP).

Your rule says 5432. The lookup is for 6432. No match, packet dropped, and nothing in the policy hints at why. The DNAT has already happened by the time the policy engine sees the packet, so `toPorts` under a `toServices` rule must be the backend pod's real `targetPort`.

```yaml
egress:
  - toServices:
      - k8sService:
          serviceName: my-database
          namespace: my-namespace
    toPorts:
      - ports:
          - port: "6432"
```

If Pod B also has an ingress policy, that gets checked separately on Pod B's side, against the same post-DNAT tuple.

### 2. Using `toServices` at all when a stable label exists

The port issue is fixable once you know about it. The bigger problem with `toServices` is that it pins your policy to whatever the Service happens to be named right now.

Rename the release, or reinstall under a different name, and the Service this rule points at no longer exists. Cilium doesn't complain. The rule just stops matching anything, and you find out when an app starts timing out.

Operator-set labels don't move. CloudNativePG puts `cnpg.io/cluster` on every pod in a cluster, and its value is the cluster name, not the release name. Selecting on that label with `toEndpoints` sidesteps both problems at once: the label is stable across releases, and with no Service or DNAT layer involved there's no port ambiguity to get wrong. `toEndpoints` also covers direct pod-to-pod connections that never go through the Service at all.

We now default to `toEndpoints` everywhere in KubeAid. `toServices` is the exception in a few places.

### 3. Selecting on labels Cilium throws away

This one cost me an afternoon.

```yaml
endpointSelector:
  matchLabels:
    statefulset.kubernetes.io/pod-name: oncall-pgsql-1
```

The label exists on the pod. `kubectl get pod --show-labels` shows it. The selector matches nothing.

Cilium keeps a list of label patterns excluded from identity calculation, and `statefulset.kubernetes.io/pod-name` is on it, along with `pod-template-hash`, `controller-revision-hash`, `apps.kubernetes.io/pod-index`, `batch.kubernetes.io/controller-uid` and a handful of others. You can check the list [here](https://github.com/cilium/cilium/blob/30a6b8c5/Documentation/operations/performance/scalability/identity-relevant-labels.rst#L13-L19).

The reason the list exists is worth understanding, because it tells you which of your own labels are risky. Identity-based policy works by grouping pods. If Cilium computed identity from every label, a Deployment with two replicas would produce two different identities, because `pod-template-hash` and per-pod names differ. Now write a `toEndpoints` rule against that Deployment: it would match one replica and not the other. That defeats the whole point of grouping by identity, and it also means every rollout churns identities across the cluster. So Cilium filters those patterns out before computing identity.

The consequence: a selector written against an excluded label doesn't error, doesn't warn, and silently matches nothing.

```yaml
# CORRECT
endpointSelector:
  matchLabels:
    cnpg.io/cluster: oncall-pgsql
    k8s:io.kubernetes.pod.namespace: my-namespace
```

One label shared by all replicas, one stable identity.

The exclusion list is configurable per cluster, so you can append your own patterns if you have labels that churn.

### 4. `toFQDNs` without an L7 DNS rule

You want a pod to reach `example.com`. You write:

```yaml
# WRONG
egress:
  - toFQDNs:
      - matchName: example.com
    toPorts:
      - ports:
          - port: "443"
```

That's a complete, valid `toFQDNs` rule, and on its own it will never work.

Before the pod can connect to `example.com`, it has to resolve it. That DNS lookup is itself just network traffic, a UDP request on port 53 heading to CoreDNS.

Before that query leaves the pod's veth, eBPF checks one thing: does a policy selecting this pod have a `rules.dns` block covering this traffic? If yes, the query is redirected into Cilium's DNS proxy instead of going straight to CoreDNS untouched. The proxy forwards the query to CoreDNS, gets the answer back, and because it sits in the middle of the conversation, it sees that answer. It records the mapping, this IP belongs to `example.com`, and writes it into the FQDN cache.

The pod receives the DNS answer normally and connects to the IP. eBPF now checks the `toFQDNs` rule, looks up whether that IP is associated with `example.com`, finds the entry the proxy just wrote, and allows the connection.

Without a `rules.dns` block anywhere, that middle step never happens. Cilium never learns what the IP is, so the `toFQDNs` lookup has nothing to match against.

You get one of two symptoms, and the second is the confusing one:

- **Nothing allows DNS egress at all.** The pod can't resolve. `curl` exits 6, "could not resolve host". Annoying, but it points straight at DNS.
- **Something allows port 53 as plain L4, but no `rules.dns`.** DNS resolves perfectly. The pod gets a good IP. Then the connection to that IP is dropped by policy. `curl` exits 7. Everything about DNS looks healthy, which is exactly why this one eats an afternoon.

```yaml
# CORRECT
egress:
  - toEndpoints:
      - matchLabels:
          k8s-app: kube-dns
          io.kubernetes.pod.namespace: kube-system
    toPorts:
      - ports:
          - port: "53"
            protocol: ANY
        rules:
          dns:
            - matchPattern: "*"
  - toFQDNs:
      - matchName: example.com
    toPorts:
      - ports:
          - port: "443"
            protocol: TCP
```

The `rules.dns` block doesn't have to live in the same policy object as `toFQDNs`. Policies selecting the same pod compose, so a namespace-wide default-deny carrying the DNS rule works fine, and the per-app policy can hold nothing but `toFQDNs`. What matters is that *some* policy selecting this pod has the DNS rule.

I'd still put both in the same policy as a habit. Splitting them means your FQDN rule quietly depends on the default-deny actually being rendered for that namespace, and on the pod not being carved out of it. Those are real conditions that nothing validates, and when one of them fails, the failure shows up in the app, not in the policy you'd think to look at.

To check whether interception is actually on for a pod, look for its endpoint in the proxy redirect list:

```bash
kubectl exec -n kube-system <cilium-pod> -c cilium-agent -- \
  cilium-dbg status -o json | jq -r '.proxy.redirects[] | select(.proxy|test("dns")) | .name'
```

No entry for your endpoint ID means the proxy was never turned on for it, and `toFQDNs` cannot work no matter how the rule is written.

### 5. Default-deny surprises

Apply any CiliumNetworkPolicy that selects a pod, and that pod flips from "enforcement: never" to default-deny. The moment a pod is selected by a policy for a direction, everything not explicitly allowed in that direction is denied.

This happens per direction, independently. A policy with only an `ingress` section flips ingress to default-deny and leaves egress alone.

In practice it looks like this: DNS stops resolving, because you never allowed egress to kube-dns. Kubelet health probes get dropped, because probes originate from the node, which carries the `host` identity rather than a pod identity, and your ingress rules only mention other apps. The container fails its liveness probe, gets killed, and you're staring at CrashLoopBackOff with an application that's actually running fine.

So don't write app rules first. This is the order that has always worked for me:

1. **Write a default-deny policy for the namespace, and allow DNS egress in that same policy.** If the namespace has other apps you're not firewalling yet, use `NotIn` in the endpoint selector to carve them out.
2. **If the app has probes, allow ingress from the kubelet.** That's the `host` entity.
3. **If the app talks to the Kubernetes API at runtime, allow egress to `kube-apiserver`.** Most apps don't. If your pod only consumes ConfigMaps and Secrets through mounts or env vars, kubelet does that on the pod's behalf: it watches the API server, pulls the data, and writes it to the node filesystem or injects it at pod start. The pod process never opens a TCP connection to the API server, so it needs no rule. You need this rule only when something inside the pod is programmatically calling the API with client-go or an SDK to GET, WATCH or PATCH resources. When you do need it, use `toEntities: [kube-apiserver]`.
4. **If the workload is managed by an operator, check whether the operator talks to the workload pod directly**, not just to the API server about it. If it does, allow ingress from the operator. CloudNativePG does this, and the operator lives in its own namespace, so the selector needs the namespace label too.
5. **Roll out in audit mode first.** Policy gets evaluated, nothing gets dropped, Hubble shows you what would have been dropped.
6. **Read Hubble.** It will show you every flow the app actually makes, including the ones nobody documented.
7. **Then write the real app rules.**

Here's the default-deny template from the KubeAid-addons chart:

```yaml
{{- range $ns, $config := .Values.defaultDeny.namespaces }}
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny
  namespace: {{ $ns }}
spec:
  endpointSelector:
  {{- if $config.excludedPods }}
    matchExpressions:
    {{- range $config.excludedPods }}
    {{- range $key, $value := . }}
    - key: {{ $key }}
      operator: NotIn
      values:
      - {{ $value }}
    {{- end }}
    {{- end }}
  {{- else }}
    {}
  {{- end }}

  enableDefaultDeny:
    egress: true
    ingress: true

  # Allow only DNS egress to kube-dns
  egress:
  - toEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: kube-system
        k8s-app: kube-dns
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
      - port: "53"
        protocol: TCP
      rules:
        dns:
        - matchPattern: "*"
{{- end }}
```

Two things to notice. It ranges over a map of namespaces, so it renders exactly one `default-deny` per namespace, and there's no way for a second one to sneak into the same namespace. And an empty `endpointSelector: {}` selects every pod in that namespace, with `excludedPods` as the escape hatch for namespaces where we're only firewalling part of what's running.

### 6. Assuming LoadBalancer traffic is pre-authorized

Your pod is internet-facing behind a `type: LoadBalancer` Service. You write ingress rules for the in-cluster callers and ship it. External traffic gets dropped.

```yaml
ingress:
  - fromEndpoints:
      - matchLabels:
          app: frontend
```

The Service being internet-facing grants nothing. Service DNAT and policy enforcement are two separate, sequential steps.

The policy engine sits on the host side of the receiving pod's veth. By the time the packet reaches it, the Service is out of the picture entirely. All policy sees is a source identity and a destination identity, and for anything originating outside the cluster that source identity is `world`. Your rule allows `app: frontend`. `world` isn't that. Dropped, right after a Service load-balancing step that succeeded perfectly.

```yaml
ingress:
  - fromEndpoints:
      - matchLabels:
          app: frontend
  - fromEntities:
      - world
```

Policy doesn't know or care how traffic arrived. It only knows what identity the traffic carries once it gets there.

### 7. Relabeling doesn't affect established connections

Changing a pod's labels does recompute its Cilium identity, and that does trigger the endpoint to regenerate its policy. But already-established connections, the ones tracked in conntrack, aren't necessarily re-evaluated against the new policy instantly. Enforcement is largely applied at connection setup time.

I want to be careful here: the exact behavior is version and configuration dependent. Verify it on your own cluster and version rather than assuming.

### 8. Death by a thousand netpols

This is the one I feel most strongly about, because it isn't a mechanism bug. It's an organizational one, and you can't fix it with better YAML.

Real namespaces aren't one app per namespace. A shared namespace like `checkout` might run multiple services. Every service owner bolts on their own CiliumNetworkPolicy whenever they need a new path open. No shared owner, no central review, and nothing forces them to coordinate, because policies only ever compose additively. Nobody's rule technically conflicts with anyone else's. They just keep piling up.

Then something breaks, and debugging stops being "read the policy". It becomes "grep every CNP that selects this namespace and hope you find the wrong one". You can't reason about a pod's effective policy from any single file, because the effective policy is the union of every rule from every policy that happens to select it.

The fix is centralizing policy authoring under one roof. That's one of the reasons KubeAid-addons is a single centralized Helm chart for all our network policies (we've centralized our operator configs there as well). Network policies for every addon live in one chart, so they're easy to read and get rendered conditionally per namespace, one targeted policy per app.

Naive centralization has its own trap, though. If you generate a policy per app and each one includes its own default-deny, you end up with several default-denies in a namespace, each written from the perspective of one app, each unaware of the others. That causes a different mess.

![KubeAid Addons Architecture](/blog/cnp/kubeaid-addons.png)

So the rule we follow is: **one targeted policy per app, and exactly one default-deny per namespace, never one per app.** The default-deny is a namespace-level concern. The app policies only ever add allows on top of it. That's what KubeAid-addons does. If you're firewalling your applications and want an open source way to keep your Cilium network policies in one place, check out [KubeAid](https://github.com/Obmondo/KubeAid) and [KubeAid-addons](https://github.com/Obmondo/KubeAid/tree/master/argocd-helm-charts/KubeAid-addons), and if you do, please leave feedback in the GitHub issues. And if you're doing this in your own clusters and hit something that isn't on this list, I'd like to hear about that too.

## Ending notes and links

- [docs.cilium.io/en/stable/security/policy](https://docs.cilium.io/en/stable/security/policy/)
- [editor.networkpolicy.io](https://editor.networkpolicy.io/) for visualizing a policy before you apply it
- [KubeAid.io](https://KubeAid.io)
- [github.com/Obmondo/KubeAid](https://github.com/Obmondo/KubeAid) (do give it a star)
- [The KubeAid-addons chart](https://github.com/Obmondo/KubeAid/tree/master/argocd-helm-charts/KubeAid-addons)

Thanks for your time.
