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

I went looking for "ArgoCD rollouts" and the first thing I learned is that there is no such thing. The project is Argo Rollouts, a sibling of Argo CD in the same family, and the two do different jobs. Argo CD syncs what is in Git into the cluster and decides *what* runs. Argo Rollouts decides *how* a new version replaces the old one, and it does not care where the manifest came from: Argo CD, Flux, Helm, or `kubectl apply` all work.
