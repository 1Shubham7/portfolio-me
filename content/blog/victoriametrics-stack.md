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

## What Prometheus leaves out, on purpose

A Prometheus server is one binary with four jobs: it scrapes targets, stores the samples in a local TSDB, evaluates recording and alerting rules, and answers PromQL queries. For one cluster that is a good design. There is one thing to deploy, and it depends on nothing else being up.

The limits come from the same design.

**One node, one local disk.** There is no clustering. When a server runs out of room, the built-in answer is more servers with the scrape targets divided between them. The Prometheus Operator [exposes this as `shards`](https://prometheus-operator.dev/docs/platform/high-availability/), and every shard is a separate Prometheus with its own data.

**No global view of the detail.** Each server answers queries about its own data. [Federation](https://prometheus.io/docs/prometheus/latest/federation/) lets one server scrape selected series from another, and the docs' example is a set of global servers that "collect and store only aggregated data" from the local ones. That is a global view of the aggregates you picked in advance. It is not a query layer over many servers, so a question nobody planned for, such as which pods in forty clusters were throttled last night, has nowhere to run.

**Retention bounded by the disk.** The default retention is 15 days. You can raise it with `--storage.tsdb.retention.time`, and then a year of history lives on one volume behind one pod, in a store the [docs](https://prometheus.io/docs/prometheus/latest/storage/) call "not arbitrarily scalable or durable in the face of drive or node outages".

**Memory that follows cardinality.** The newest samples of every active series are held in memory before they are cut into blocks on disk. Pod churn and labels with many values multiply the series count, and RAM goes up with it.

Prometheus's answer to the middle two is `remote_write`: stream every sample to another system, and let that system keep the history and answer the queries that span servers. It does nothing for the first or the fourth. The scraping server is still one node holding every active series in memory, and the [remote write tuning page](https://prometheus.io/docs/practices/remote_write/) says shipping them adds to the bill: "Most users report ~25% increased memory usage".

Three well-known projects were built to be that other system, and each is a distributed system of some size. Thanos gets the data out of Prometheus two ways, a Sidecar that uploads the TSDB blocks to a bucket or a Receiver that takes `remote_write`, and its [tutorial](https://thanos.io/tip/thanos/quick-tutorial.md/) lists seven components (Sidecar, Store Gateway, Compactor, Receiver, Ruler, Querier, Query Frontend) around that bucket. [Cortex](https://cortexmetrics.io/docs/architecture/) has distributors, ingesters, queriers, a compactor and a store gateway, with object storage, a key-value store for its hash ring and optional caches behind them. Mimir, Grafana's fork of Cortex, [lists seven required components](https://grafana.com/docs/mimir/latest/references/architecture/components/), needs an object store too, and since version 3.0 [prefers an architecture](https://grafana.com/docs/mimir/latest/get-started/about-grafana-mimir-architecture/) with Kafka in the middle.

That complexity buys something real: long-term data in a bucket that never needs resizing. It is also a lot to operate if what you wanted was Prometheus with a longer memory.

