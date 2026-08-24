---
title: "Centralized Logging on Kubernetes with Grafana Loki: Dead Charts, Label Cardinality, and Four Silent Failures"
description: "Picking the right Loki chart in a maze of deprecated ones, designing labels so Loki doesn't fall over, the Fluent Bit output config that actually works, and the four things that broke quietly after cutover."
dateString: August 2026
draft: false
tags: ["Kubernetes", "Loki", "Grafana", "Logging", "Observability", "DevOps"]
weight: 2
cover:
    image: ""
---

The first three guides I found for running Loki on Kubernetes all told me to install a chart that hasn't been maintained in years, shipping a log collector that reached end of life in March 2026. None of them said so. They were all reasonably well written, they all worked well enough that you'd get logs into Grafana, and they'd all leave you running dead software on day one.

That was the first surprise. The rest of them came later, after everything was green and shipping and apparently fine.

This is a write-up of setting up centralized logging on a cluster from scratch: which chart, which collector, how to design labels so Loki doesn't fall over in three months, and the four things that broke silently enough that I only found them by going looking.

## The chart you find first is the one you shouldn't install

Search for "loki helm chart" and the top result is almost always `grafana/loki-stack`. It's the friendliest one. It bundles Loki, a collector, and optionally Grafana, so one `helm install` gets you a working pipeline. It's also deprecated. The chart's own README says so:

> This chart is deprecated and will no longer receive updates or support.

The collector it bundles by default is Promtail, which matters more than the chart being stale. Promtail went into long-term support in February 2025 and reached end of life on **March 2, 2026**. No more updates, no more security fixes. Installing `loki-stack` today means adopting an EOL log collector without ever making a decision about it.

The next thing people find is `loki-distributed`, usually recommended in an answer that says "use loki-distributed for production". Same story, same wording in its README, superseded by the official chart.

The one you want is the official `loki` chart, and it has a `deploymentMode` value that covers everything the two dead charts used to do separately:

- **`Monolithic`** (the old value `SingleBinary` is deprecated but still accepted): every Loki component in one process. Filesystem storage works here.
- **`SimpleScalable`**: read, write, and backend split into separate deployments, scaled independently. Requires object storage.
- **`Distributed`**: every microservice separately (distributor, ingester, querier, query-frontend, compactor, and friends).

There's one more thing that isn't in most blog posts yet, because it's recent. As of March 2026 the Loki Helm chart was forked out of the Loki repository and is now maintained by Grafana Community Champions in a separate org. The Grafana Labs repo keeps the Grafana Enterprise Logs charts; the open source Loki chart moved:

```bash
# the one most guides still tell you to add
helm repo add grafana https://grafana.github.io/helm-charts

# where the open source chart is maintained now
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
```

The old repo still publishes a `loki` chart, so this isn't a case of one being dead. It's that the two have drifted: at the time of writing the Grafana Labs one is on app version 3.6.12 and the community one is on 3.7.6. Check the `appVersion` you're actually getting rather than assuming the chart version tracks the Loki version, because the two repos don't even share a chart versioning scheme.

Also worth knowing before you commit to a topology: `SimpleScalable` is on the way out. The docs now say SSD mode is being deprecated and will be gone before Loki 4.0, with microservices mode recommended for production at scale. The bundled MinIO subchart is deprecated too, to the point that the chart refuses to install with `minio.enabled=true` unless you set `ignoreMinioDeprecation`, so if you were planning to lean on it for object storage, don't.

### Which mode to actually pick

For a single cluster shipping its own logs, `Monolithic` with filesystem storage on a PVC is fine, and I'd argue it's the correct default rather than a compromise. One process, no object storage dependency, no S3 credentials to rotate, one thing to debug when queries are slow. A cluster generating a few GB of logs a day does not need eight microservices.

You outgrow it when one of these becomes true: you want retention long enough that a PVC gets uncomfortable, you want logs to survive the cluster, or your read load and write load have genuinely different shapes (a steady ingest with occasional expensive dashboard queries that shouldn't be able to starve ingestion). Then you go to object storage and split read from write. Given the SSD deprecation, if you're making that jump now, look at `Distributed` rather than `SimpleScalable`.

## Which collector

Loki doesn't collect anything. Something has to tail `/var/log/containers`, attach Kubernetes metadata, and push to Loki's API. There are three real candidates and one honourable mention.

**Fluent Bit.** Written in C, tiny footprint, CNCF project under the Fluentd umbrella, and completely vendor neutral. Its Loki output plugin is one of about a hundred outputs it supports, which is the point: if you ever want to fan the same logs out to S3 and Loki, or swap Loki for something else, the collector layer doesn't change. It's also already running in a lot of clusters, because it's what most managed Kubernetes log forwarding is built on.

**Grafana Alloy.** Grafana's distribution of the OpenTelemetry Collector, and the official successor to Promtail. If you're already all-in on Grafana (Loki, Mimir, Tempo, Pyroscope), Alloy is the obvious answer. It handles metrics, logs, traces and profiles in one agent with one config language, and Grafana ships a conversion tool that turns an old Promtail config into an Alloy one.

**Promtail.** EOL as of March 2026. Nobody starting today should pick it. The problem is that a large share of the Loki documentation that exists on the internet assumes it, so you'll keep finding Promtail examples and have to translate them.

**Vector** is worth a mention: fast, good transform language, popular with people who want to do real processing in the pipeline rather than at query time.

The honest version of the trade-off is that there isn't a winner. Fluent Bit if you want backend independence or already run it. Alloy if you're committed to the Grafana stack and want one agent for all four signals. Both are correct. The wrong answer is Promtail in 2026, and the *really* wrong answer is installing a chart that picks Promtail for you without you noticing.

I went with Fluent Bit, mostly for the backend independence and because it was already there.

## Labels are the whole game

This is the part that determines whether your Loki install works or quietly turns into a problem, and it's the part most tutorials get wrong by omission.

Loki does not index log content. It indexes labels, and for everything else it greps. That's the entire design. Every distinct combination of label values is a **stream**, and streams are the unit of cost: each one gets its own chunks, its own index entries, its own place in memory in the ingester.

So do the arithmetic before you write a config, not after.

`namespace` x `container` x `stream` (stdout/stderr) on a cluster with 30 namespaces and a few containers each lands somewhere in the low hundreds of streams. Completely fine.

Add `pod` and you've broken it. Not because of the pod count at any instant, but because pod names are ephemeral. Every rollout, every OOMKill, every node drain mints brand new pod names, and every one of those is a permanently new stream in the index. The count only goes up. A cluster with a hundred pods that restart a few times a day will produce tens of thousands of dead streams inside a month, and the index carries all of them forever.

The same applies to anything else that churns: `pod_ip`, `container_id`, `request_id`, `trace_id`, anything with a timestamp in it.

### Structured metadata is the answer, and it postdates most of the blog posts

The obvious objection to all of the above is that you genuinely want to know which pod a line came from. Older posts have no good answer to this and either tell you to label by pod anyway or tell you to live without it.

Loki 3.x has a proper answer: **structured metadata**. It's key/value data attached to each log line that is *not* indexed and *not* part of the stream identity, but is stored alongside the line and returned with every result. You can still filter on it. It just doesn't cost you a stream.

It needs schema version 13 or higher and the TSDB index, which the current chart gives you by default. It does not work with the deprecated `boltdb-shipper` store.

So the design is:

- **Labels**: `namespace`, `container`, `stream`, plus maybe a static `job` or `cluster`. Bounded, low cardinality, changes rarely.
- **Structured metadata**: `pod`, `node`, `container_id`. High cardinality, still there on every line.

Here's the Fluent Bit side of it. The Kubernetes filter nests everything under a `kubernetes` map, so a `nest` filter lifts it to the top level first, which makes the output block much easier to read:

```ini
[FILTER]
    Name          kubernetes
    Match         kube.*
    Kube_Tag_Prefix  kube.var.log.containers.
    Merge_Log     Off
    Keep_Log      On

[FILTER]
    Name          nest
    Match         kube.*
    Operation     lift
    Nested_under  kubernetes

[OUTPUT]
    Name                 loki
    Match                kube.*
    Host                 loki-gateway.logging.svc.cluster.local
    Port                 80
    Labels               job=fluent-bit, namespace=$namespace_name, container=$container_name
    Label_Keys           $stream
    Structured_Metadata  pod=$pod_name, node=$host, container_id=$container_hash
    Remove_Keys          namespace_name,container_name,pod_name,pod_id,pod_ip,host,container_hash,container_image,docker_id,labels,annotations,stream,_p,time
    Drop_Single_Key      raw
    Line_Format          json
```

Two things about that block that are easy to skim past.

`Labels` and `Label_Keys` do different jobs. `Labels` takes `key=value` pairs and lets you name the label whatever you want, with `$field` record accessors on the right-hand side, which is how `namespace_name` becomes the label `namespace`. `Label_Keys` takes bare record keys and uses the key's own name as the label name, so `$stream` becomes the label `stream`. Use `Labels` when you want to rename, `Label_Keys` when the field name is already what you want.

`Remove_Keys` lists every metadata field explicitly rather than relying on label extraction to consume them. That's deliberate, and the reason is the next section.

### The syntax trap that costs you an afternoon

Structured metadata is not a label, and it does not work in the stream selector. This query:

```logql
{pod="checkout-7d9f4b8c6-xk2mn"}
```

does not error. It doesn't warn. It returns nothing at all, forever, which reads exactly like "Loki is broken" or "the collector isn't shipping" rather than "you used the wrong syntax". Nothing in the UI tells you the difference.

The correct form selects a stream first, then filters:

```logql
{namespace="checkout"} | pod="checkout-7d9f4b8c6-xk2mn"
```

Curly braces are for labels only. The pipeline after them is where structured metadata lives. I lost real time to this before it clicked, and I don't think that's unusual, because every Loki example written before version 3 uses `{pod=...}` and works fine, since back then people *were* labelling by pod.

## LogQL, the parts you'll use daily

You can't grep the whole cluster. The stream selector in `{}` is mandatory and it's the first thing that runs, so the narrower it is the less data ever gets read off disk.

The single highest-leverage thing to know about LogQL is **filter ordering**. These two look equivalent:

```logql
{namespace="checkout"} | json | level="fatal"
{namespace="checkout"} |= "error" | json | level="fatal"
```

The second is dramatically faster. The line filter `|= "error"` is a substring match that runs before the JSON parser, so most lines are discarded without ever being parsed. In the first query, every single line in the namespace gets JSON-decoded before the level check happens, and JSON parsing is by far the most expensive stage in a typical pipeline. Put a cheap line filter first whenever the parsed condition implies a substring you can match on.

Related: label filters are index lookups, line filters are greps. Selecting `stream="stderr"` in the stream selector costs essentially nothing because it narrows which chunks get read at all. Grepping for `"error"` reads every chunk in the selector and scans it. If the thing you want is expressible as a label, express it as a label.

Parsers are `| json` and `| logfmt`, and after either one, every extracted field is available as a label filter for the rest of the pipeline.

Logs become metrics with the range aggregations, which is where Loki starts earning its keep:

```logql
# error rate per container over 5 minutes
sum by (container) (rate({namespace="checkout"} |= "error" [5m]))

# which namespace is generating the most log volume right now
topk(5, sum by (namespace) (rate({job="fluent-bit"}[5m])))

# restarts, crashes, whatever you can pattern match, as a countable series
count_over_time({namespace="checkout"} |= "panic" [1h])
```

Structured metadata works in aggregations too, which is the thing that makes the label design above actually livable:

```logql
sum by (pod) (rate({namespace="checkout"} |= "error" [5m]))
```

You get per-pod breakdown without per-pod streams. That's the whole trade, and it's a good one.

## Four things that went wrong quietly

Everything above is the design. This section is what actually happened.

### 1. One extra field silently disabled `drop_single_key`

The goal is bare log lines in Loki. Not JSON-wrapped ones, because if the application already emits JSON, wrapping it in another layer of JSON means every query needs two `| json` stages and the Grafana log view shows you an envelope instead of a message.

That's what `Drop_Single_Key raw` is for. The docs are precise about what it does: when, after extracting labels, **exactly one key remains**, the log line sent to Loki is the value of that key, unquoted.

Exactly one. If two keys survive, the setting does nothing and the plugin silently falls back to `Line_Format json`, wrapping the record. No warning, no error, no log line about it.

The Kubernetes metadata filter emits `pod_ip`. My `Remove_Keys` list didn't have it. So two keys survived, `log` and `pod_ip`, and every line arrived looking like this:

```json
{"log":"level=info msg=\"order placed\" order_id=88213","pod_ip":"10.42.3.117"}
```

instead of this:

```
level=info msg="order placed" order_id=88213
```

Adding `pod_ip` to `Remove_Keys` fixed it in one line. Finding it took considerably longer, and the reason why is the interesting part.

**It does not reproduce offline.** Before touching a cluster I'd tested the output block properly: a file of realistic CRI-format lines, a real Loki running in Docker, Fluent Bit pointed at both. That was the right call and it caught two other bugs before they went anywhere near a cluster. But with no API server to talk to, the Kubernetes filter has no pod to look up, so it never adds `pod_ip`, only one key survives, `drop_single_key` fires, and the bare log line looks perfect.

The offline test passed for a reason that only exists offline. I'd still do it, it's cheap and it caught real problems, but it has a known blind spot: any bug whose trigger is a field the Kubernetes filter only adds when it can reach a live API server is invisible to it. Test the output block offline, then check the actual bytes landing in Loki once it's on a cluster.

### 2. Wrong option names fail loudly, wrong field names fail quietly

Two mistakes from the same afternoon, with completely different failure modes.

I wrote `structured_metadata_keys`, reasoning from the existence of `label_keys`. It isn't a real option. Fluent Bit refuses to start:

```
[error] [config] loki: unknown configuration property 'structured_metadata_keys'
```

That's the *good* outcome. You find out in five seconds and the message tells you exactly what's wrong. (The real option, if you want to promote a whole map into structured metadata, is `structured_metadata_map_keys`.)

The other one was worse. I wrote `Remove_Keys` and pipeline references assuming the CRI parser emits `message` and `logtag`, because that's what the `cri` parser in the bundled `parsers.conf` emits. But the `cri` **multiline** parser, which is what you use in the tail input to handle partial lines correctly, emits different field names:

```
^(?<time>.+?) (?<stream>stdout|stderr) (?<_p>F|P) (?<log>.*)$
```

`log` and `_p`, not `message` and `logtag`. Two parsers, same name, same log format, different capture groups. A config built on the wrong assumption starts perfectly, ships logs perfectly, and does the wrong thing with the fields. Nothing fails.

The lesson generalises past Fluent Bit: an option name that doesn't exist gets rejected by a parser, and you find out immediately. A *field* name that doesn't exist just evaluates to nothing, and nothing complains. Check field names against actual parser output, not against docs and not against memory. Run the collector against one file with stdout output and look at the records.

### 3. The default ingestion rate limit bites you on cutover

Loki's default per-tenant ingestion limit is 4 MB/s, with a 6 MB burst. With `auth_enabled: false`, which is the default for a single-cluster install and completely reasonable, there is exactly one tenant called `fake`, and **every collector on every node shares that one budget**.

Repoint a whole fleet of collectors at a new Loki at the same time and they all replay their buffers at once. You get a wall of this:

```
Ingestion rate limit exceeded for user fake (limit: 4194304 bytes/sec) while attempting
to ingest '2500' lines totaling '1048576' bytes, reduce log volume or contact your Loki
administrator to see if the limit can be increased
```

paired with HTTP 429s in every collector's logs. It looks like a catastrophe and it isn't. Nothing is lost. The collector backs off, retries, and the chunks land. Within a few minutes the buffers drain and the 429s stop on their own.

The judgement call is telling that apart from a limit that's genuinely too low. The distinction is shape, not volume: a cutover burst decays and stops, a real limit produces a steady-state stream of 429s that never goes away. Watch it for ten minutes before you raise anything. If it's still going, raise `ingestion_rate_mb` and `ingestion_burst_size_mb` together, because raising the rate without the burst just moves where you hit the wall.

### 4. The Kubernetes filter's 32KB buffer

This one produced records with no Kubernetes metadata at all, from specific pods only, intermittently:

```
[error] [filter:kubernetes:kubernetes.0] cannot increase buffer: current=32000 requested=64768
```

The Kubernetes filter reads pod metadata from the API server into a fixed HTTP response buffer, and `Buffer_Size` defaults to `32k`. Pods with large annotations blow past it. Anything managed by a controller that stores state in annotations can easily exceed 32KB of pod object, and when that happens the lookup fails and the record ships with no labels attached, which means no `namespace` label, which means it lands in the wrong stream or gets dropped by your routing.

```ini
[FILTER]
    Name         kubernetes
    Match        kube.*
    Buffer_Size  1MB
```

Setting it to `0` removes the limit entirely and lets the buffer grow as needed, which the docs support, though I'd rather have a large ceiling than none at all.

## What I'd check on day one

Query for a pod you know exists, using `| pod="..."` and not `{pod="..."}`, and confirm you get lines back. Then look at the raw content of one of those lines and confirm it's the application's actual output and not a JSON envelope wrapped around it. Then run `sum by (namespace) (rate({job="fluent-bit"}[5m]))` and see whether the volume distribution matches your intuition about which apps are chatty, because if one namespace is producing 80% of your logs, you want to know that before you size retention rather than after.

Those three checks would have caught two of the four problems above on the first afternoon instead of the third.

## Ending notes and links

- [Loki Helm chart docs](https://grafana.com/docs/loki/latest/setup/install/helm/), and the [migration guide to the community chart](https://grafana.com/docs/loki/latest/setup/upgrade/upgrade-to-community/) if you're already running the old one
- [Structured metadata](https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/), the page I wish I'd read first
- [Fluent Bit Loki output plugin](https://docs.fluentbit.io/manual/data-pipeline/outputs/loki.md) for the full option list
- [Fluent Bit Kubernetes filter](https://docs.fluentbit.io/manual/data-pipeline/filters/kubernetes.md), including `Buffer_Size`
- [Migrating from Promtail to Alloy](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/) if you inherited a Promtail setup

Thanks for your time.
