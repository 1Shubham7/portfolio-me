---
title: "VictoriaMetrics, VictoriaLogs and VictoriaTraces: What Prometheus Left Out, and the Stack Built Around It"
description: "VictoriaMetrics started as a remote storage backend for Prometheus and grew into a full monitoring stack. What Prometheus deliberately does not do, why two HA Prometheus replicas are not a cluster, how VictoriaMetrics splits Prometheus's jobs into components and keeps long-term data without object storage, how VictoriaLogs compares with Loki and VictoriaTraces with Tempo and Jaeger, what else the project ships, and when it is the right choice."
dateString: October 2026
draft: false
tags: ["VictoriaMetrics", "VictoriaLogs", "VictoriaTraces", "Prometheus", "Observability", "Kubernetes", "SRE"]
weight: 1
---

The Prometheus storage docs have two sentences that explain a whole category of software:

> Prometheus's local storage is limited to a single node's scalability and durability. Instead of trying to solve clustered storage in Prometheus itself, Prometheus offers a set of interfaces that allow integrating with remote storage systems.

The limit is deliberate. Prometheus keeps what fits on one machine and leaves the rest to whoever wants to build it. Thanos, Cortex and Mimir were built on the other side of that interface. So was VictoriaMetrics, which began as a storage backend you point `remote_write` at and has since grown into a full stack: a metrics database, a scraping agent, a rule evaluator, an auth proxy, a log database and, since last year, a trace database.

VictoriaMetrics is best understood by looking at what Prometheus deliberately does not do. This is an introduction for someone who runs Prometheus on Kubernetes, probably with Loki beside it, has heard the name VictoriaMetrics, and could not say what `vminsert` does.

I am writing from the Prometheus and Loki side, which is what [my other posts](/blog/loki-production-checklist/) are about. What follows comes from the Victoria projects' docs and repositories as they stood in the first week of October 2026, not from running them, and every performance figure in it is the project's own claim.

