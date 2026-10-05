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

## "But we run Prometheus in a cluster"

That was my first objection to "there is no clustering", and it comes from two different things sharing the word cluster.

Running Prometheus on a Kubernetes cluster means its pod can be scheduled on any node and rescheduled when a node dies. It does not make Prometheus a clustered database. Set `replicas: 2` on a Prometheus Operator resource and you get what the [Prometheus FAQ](https://prometheus.io/docs/introduction/faq/) recommends for high availability: "run identical Prometheus servers on two or more separate machines." The operator's [HA docs](https://prometheus-operator.dev/docs/platform/high-availability/) describe the pair as instances with the same configuration, apart from one external label that tells them apart (`prometheus_replica` by default), which "scrape the same targets and evaluate the same rules".

```text
               targets
              /       \
    Prometheus A     Prometheus B
         |                |
      TSDB A           TSDB B
```

### Isn't that what PostgreSQL and Redis replicas are?

It looks the same from a distance: a few pods, one of which can die. The difference is what flows between them.

```text
    writes
      |
      v
   primary  ==== WAL stream ====>  standby
```

A PostgreSQL standby does not take the application's writes and build its own state. In the [docs'](https://www.postgresql.org/docs/current/warm-standby.html) words, "In standby mode, the server continuously applies WAL received from the primary server." There is one database state, produced by the primary, and the standby is a copy of it. A standby that restarts or falls behind carries on from the last WAL record it has, and replication slots exist so that the primary "does not remove WAL segments until they have been received by all standbys". [Redis](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/) has the same shape: the master keeps a replica updated by sending it "a stream of commands", and a replica that loses the link reconnects and tries to "obtain the part of the stream of commands it missed during the disconnection".

Replication, then, means one node decides what the data is and the others receive it, including whatever they missed. An HA Prometheus pair has no stream. A and B each scrape, each write their own samples to their own TSDB, and neither knows the other exists. That is duplication. The Alertmanager deduplicates the identical alerts the pair sends, which is why paging still works. Nothing reconciles the data.

### How two replicas drift

Both replicas scrape a target every 15 seconds. At 12:05, A crashes and stays down for five minutes.

```text
A's TSDB:   12:00 ======= 12:05   [ gap ]   12:10 =======>
B's TSDB:   12:00 =======================================>
```

When A comes back, B does not send it the missing five minutes, because no mechanism exists that could. A carries a hole in every series until that data ages out of retention. Put both replicas behind one Service as a Grafana datasource, and the same panel has a gap or not depending on which pod answered.

The two differ even when nothing crashes. Each replica scrapes on its own schedule, and the operator docs warn that queries against each "may return slightly different results", recommending sticky sessions for dashboards.

So something above the pair has to choose one copy or merge the two. Thanos does it at query time: its Querier is told which label marks a replica (`--query.replica-label`) and deduplicates across it. Cortex does it on the way in, with an HA tracker that "deduplicates incoming samples from redundant Prometheus servers".

