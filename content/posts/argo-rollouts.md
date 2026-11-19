---
title: "Canary, Blue-Green, and Analysis: A Practical Intro to Argo Rollouts"
description: "A Deployment can roll pods forward or recreate them, and that is the whole menu. Argo Rollouts is a controller and a Rollout CRD that take the same pod template and add a strategy: shift a weighted slice of traffic to the new version through a TraefikService, pause for a timer or a human, run a Prometheus query against the canary and abort on its own when it fails, or stage a full blue-green copy behind a preview Service and flip the active one. The gotchas that come with it: Argo CD drift on the weight field, an abort that does not touch Git, a stable ReplicaSet that stays at full size, and the 30 seconds that keep old pods alive after promotion."
dateString: November 2026
draft: false
tags: ["Argo Rollouts", "Kubernetes", "Progressive Delivery", "GitOps", "Traefik", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/argo-rollouts/argo-rollouts.png"
---

I went looking for "ArgoCD rollouts" and the first thing I learned is that there is no such thing. The project is Argo Rollouts, a sibling of Argo CD, and the two do different jobs. Argo CD syncs what is in Git into the cluster and decides *what* runs. Argo Rollouts decides *how* a new version replaces the old one, and it does not care where the manifest came from: Argo CD, Flux, Helm, or `kubectl apply` all work.

The gap it fills is the one a Deployment leaves. A Deployment has two strategies, `RollingUpdate` and `Recreate`, which [an older post here](/k8s-depl-stratigies/) covers, and neither can send 5% of traffic to a new version, wait for a human or a metric before going further, abort by itself when the error rate climbs, or hold two full ReplicaSets side by side for a blue-green switch. [Argo Rollouts](https://argo-rollouts.readthedocs.io/en/stable/) is a controller plus a set of CRDs that add exactly those.

## A Deployment with a strategy block

A Rollout is a Deployment with `spec.strategy` replaced. The pod template, selector and replica count are the same, and the controller manages ReplicaSets the way a Deployment would: when `spec.template` changes it creates a new ReplicaSet and works through the steps you gave it, one at a time.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api
spec:
  replicas: 10
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: registry.example.com/api:1.4.0
  strategy:
    canary:
      canaryService: api-canary
      stableService: api-stable
      trafficRouting:
        traefik:
          weightedTraefikServiceName: api-weighted
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50
      - pause: {}
```

Read the steps top to bottom. Send 10% of traffic to the new ReplicaSet, hold for five minutes, run the `success-rate` analysis and stop here if it fails, go to 50%, then wait. A `pause` with no duration waits until someone runs `kubectl argo rollouts promote api`, or clicks the matching action in the Argo CD UI, so the last step is a manual gate. After it the new ReplicaSet becomes stable and gets 100%.

Blue-green is the other strategy and it is shorter to describe. `strategy.blueGreen` names an `activeService` and, optionally, a `previewService`. The controller brings the new ReplicaSet up to full size, points the preview Service at it so you can test it on a URL nobody else uses, and then switches the active Service's selector in one move. With `autoPromotionEnabled: false` that switch waits for a promote, and a `prePromotionAnalysis` block runs an analysis before it and refuses the switch if the analysis fails. Canary is for shifting a percentage in steps; blue-green is for a full copy and one flip.

## Replicas or weights

The `trafficRouting` block in the example is what makes `setWeight` mean what it says. Without it, the controller has only replica counts to work with, and the [canary docs](https://argo-rollouts.readthedocs.io/en/stable/features/canary/) describe the result as a best-effort attempt at the percentage in the last `setWeight`: a 41% step on 10 replicas gives four new pods and six old ones. On three replicas a step is one pod, two or all three, there is no 10% to be had, and a Service spreads traffic across those pods however kube-proxy feels like it.

With a traffic router the controller sets the weight on the router and the split is exact. On a Traefik cluster that router is a TraefikService using weighted round robin, which the [Traefik integration page](https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/traefik/) shows as a `weighted.services` list naming the stable and canary Services on their port. The controller rewrites the weights on that object as the steps advance, and rewrites the selectors on the two Services so each one matches only the pod-template hash of its ReplicaSet; your IngressRoute sends traffic to the TraefikService instead of to a plain Service. Other routers on the [traffic management page](https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/): NGINX, ALB, Istio, SMI, and Gateway API as a plugin.

## Analysis is what makes it progressive

Without analysis the steps above are a slow rolling update with a human watching Grafana. An AnalysisTemplate is a metric query plus a condition; a step or a background block names the template, and the controller creates an AnalysisRun that polls on an interval and compares each measurement against the condition.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 60s
    count: 5
    successCondition: result[0] >= 0.95
    failureLimit: 1
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc:9090
        query: |
          sum(rate(http_requests_total{app="api",role="canary",code!~"5.."}[2m]))
          /
          sum(rate(http_requests_total{app="api",role="canary"}[2m]))
```

Five measurements a minute apart, and more than one below 95% fails the run. Prometheus is the common provider; the [analysis docs](https://argo-rollouts.readthedocs.io/en/stable/features/analysis/) also cover Datadog, New Relic, Kayenta, a plain web endpoint and a Kubernetes Job. The `role="canary"` label in the query is not something your pods have by default. `canaryMetadata` and `stableMetadata` under the canary strategy add labels to the canary and stable pods, and your scrape config has to copy that pod label onto the series (`podTargetLabels` on a PodMonitor, or a relabel from `__meta_kubernetes_pod_label_role`) before the query above returns anything. Key it on a label like `role` and not on `rollouts-pod-template-hash`, which changes every revision; the [ephemeral metadata page](https://argo-rollouts.readthedocs.io/en/stable/features/ephemeral-metadata/) notes the canary's pods are relabelled in place to `stable` on promotion, so the label follows the role. Without it the analysis scores a blend of both versions, and a broken canary at 10% hides inside stable's numbers.

An analysis step blocks the rollout until the run finishes. A `canary.analysis` block runs in the background instead, from the step named in `startingStep` onward, so the check keeps going while a `pause` counts down. Either way a failed run aborts the rollout: the docs say the canary weight is set back to zero, and the Rollout is marked Degraded. Inconclusive runs pause the rollout at its step rather than aborting it. A metric listed under `dryRun` is evaluated and reported in the AnalysisRun but cannot change the outcome, which is how to try a new query for a few deploys before you let it abort anything.

## Where it bites

The weight field is the GitOps problem everyone hits. The Traefik page is explicit that its TraefikService example leaves `weight` out, because the controller sets it, and if Git also declares a value Argo CD sees the controller's change as drift and shows the resource OutOfSync on every step. Leave the field out of the manifest, or add an `ignoreDifferences` entry for it. Istio VirtualServices have the same issue.

An abort is not a rollback in Git. Traffic goes back to stable and the old pods keep serving, but `spec.template` still holds the new image; the [abort command docs](https://argo-rollouts.readthedocs.io/en/stable/generated/kubectl-argo-rollouts/kubectl-argo-rollouts_abort/) say the Rollout will try the new version again if it leaves the aborted state, and the FAQ says Argo CD shows it as degraded until then. Rollouts never writes to Git. You revert the commit, or fix forward with a new image, and the Rollout is healthy again when the template matches something that works.

During a traffic-routed canary the stable ReplicaSet stays at full size. The traffic management docs state it directly: stable remains fully scaled while the canary progresses, since the controller has to assume stable can take 100% back at any moment. At the last step you are paying for roughly two copies of the app. `dynamicStableScale: true` scales stable down as the canary weight goes up. The docs say only that the default exists so traffic can shift back immediately on an abort, so the unstated price has to be a slower abort: stable scales back up before it can take the traffic.

`scaleDownDelaySeconds` is the gap between promotion and the old pods going away, 30 seconds by default for both strategies. For canary with traffic routing the [spec reference](https://argo-rollouts.readthedocs.io/en/stable/features/specification/) says the delay gives traffic providers time to re-target the new pods after the stable Service selector moves; for blue-green it recommends at least 30 seconds for iptables propagation across nodes. Set it lower and a proxy that still holds the old endpoints is sending requests to pods that are gone: a burst of 5xx right after a promotion that looked clean. Setting it higher costs nothing but the old pods' resources, and what those pods do when they are finally told to stop is [the 30-second pod post](/pod-30-seconds/).

Weights apply when a request is routed. A websocket or a streaming gRPC call opened against the canary stays there as the weight moves, so for a service that is mostly long-lived connections the percentage on a step is the share of new connections, not of traffic. An abort sets the weight to zero but does not close those connections. `abortScaleDownDelaySeconds`, 30 seconds by default, is how long the canary pods live after an abort before the controller scales them away and cuts whatever is still attached.

And two versions of the app serve real requests at the same time, for as long as the steps take. Every schema change and every API contract in between has to work in both directions, and nothing in the Rollout can check that for you.

## Adopting it without a rewrite

`workloadRef` points a Rollout at an existing Deployment and uses its pod template, so you keep editing the Deployment and the Rollout runs the strategy. The [migration page](https://argo-rollouts.readthedocs.io/en/stable/migrating/) has the Rollout scale the Deployment down for you with `scaleDown: onsuccess` or `progressively`, and warns that you run twice the pods until that happens. The HPA keeps working with `scaleTargetRef.kind: Rollout` and `apiVersion: argoproj.io/v1alpha1`, because the Rollout exposes the same `/scale` subresource a Deployment does. `rollbackWindow.revisions: 3` is worth setting from the start: without it, re-applying an older manifest is just another spec change and runs every step and every analysis, and with it a return to one of the last three ReplicaSets skips the steps entirely.

The `kubectl argo rollouts` plugin is the day-to-day tool. `get rollout api --watch` draws the ReplicaSets, the current step and any AnalysisRuns and updates as they change, `promote` moves past a pause, `abort` sends traffic back to stable, `undo` goes to a previous revision, and `dashboard` starts the UI locally. Argo CD ships resume, abort and promote-full as resource actions on a Rollout, so a team that lives in that UI rarely needs the plugin.

If you have looked at Flagger, the structural difference is that Flagger keeps your Deployment and creates a `<name>-primary` copy that takes the traffic, scaling the original to zero, while Rollouts replaces the Deployment with its own resource. Both do canary and analysis; which one fits depends on whether you would rather own a new CRD or a shadow Deployment.

## Ending notes

A Rollout is a Deployment whose strategy can do arithmetic on traffic and ask Prometheus a question. The canary steps, the TraefikService weights and the AnalysisRun are the three pieces, and the controller's job is to move between them without a person at the keyboard for the whole of it.

The three things I would check on an existing setup: that the TraefikService in Git has no `weight` field, that the analysis query can actually tell canary pods from stable ones, and that `scaleDownDelaySeconds` was never lowered to make deploys look faster.

Thanks for your time :)
