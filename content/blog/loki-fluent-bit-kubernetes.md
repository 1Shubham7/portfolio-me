---
title: "Ten Things to Get Right Before You Run Loki in Production"
description: "Deployment mode, object storage, label cardinality, multi-tenancy and auth, cross-cluster shipping, and the failures that are silent rather than loud. Written after moving a Loki install twice and debugging an object store that corrupted everything it was given."
dateString: August 2026
draft: false
tags: ["Kubernetes", "Loki", "Grafana", "Logging", "Observability", "DevOps", "SRE"]
weight: 2
cover:
    image: ""
---

Loki is easy to install and easy to get wrong. Most of the ways it goes wrong are quiet: queries return partial results with no error, objects are stored corrupted with a 200 response, a label filter matches nothing and looks like a broken pipeline. You find out weeks later, usually during an incident, which is exactly when you were relying on it.

This is what I wish I had known before starting. It is written in the order the decisions actually arrive, and every claim in it is something I checked against a running system rather than read in a doc.


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

For production the answer is `Distributed`, and it is worth being blunt about that because a lot of write-ups (including an earlier version of this one) present Monolithic as a sensible default you grow out of later.

The chart's own comments set the boundaries:

- **Monolithic**: "useful for small installs typically without HA, up to a few tens of GB/day"
- **SimpleScalable**: "up to about 1TB/day", and marked `deprecated, removed in Loki 4`
- **Distributed**: everything else

Read that again: the middle option is being removed. So the practical choice is now binary, and Monolithic explicitly says *without HA*. If you need more than one replica of anything, you are in Distributed.

Monolithic is genuinely fine for a lab, a single small cluster, or proving the pipeline works before you commit. It is one process instead of eighteen pods, which makes the first day much easier. Just do not plan to stay there, because the migration is not a values tweak: the workloads are different objects, and you get to rediscover every default in the next section.

I ran Monolithic first and moved to Distributed a day later. If I were starting again I would go straight to Distributed on any cluster whose logs someone will actually depend on, and use Monolithic only as a throwaway to validate credentials and connectivity.

## Object storage is a correctness requirement, not a capacity upgrade

This is the single most important thing in this post.

In Distributed mode with `replication_factor: 3`, every chunk is written to three ingesters, and each of them flushes to storage independently. A querier then has to read chunks flushed by *any* ingester.

- **On a PVC**: ingester-0 flushes to its own volume. No other querier can see it. Queries come back missing whatever the other ingesters wrote, and **nothing errors**. You get a plausible-looking graph with holes in it.
- **On object storage**: every ingester writes to the same bucket, every querier reads it.

That is why the chart refuses to start Distributed without object storage, and why Monolithic is allowed to use either.

The confusing part is that the ingesters still want PVCs, and people conclude they can therefore skip the bucket. Those PVCs are the **write ahead log**: the unflushed data a crashed ingester replays on restart. Deliberately node-local, small, and not a substitute for shared storage. The chart default is `persistence.enabled: false`, meaning emptyDir, with its own comment warning that all ingester data is lost on pod restart. Turn it on.

## Which collector, and why it matters less than you think



Loki doesn't collect anything. Something has to tail `/var/log/containers`, attach Kubernetes metadata, and push to Loki's API. There are three real candidates and one honourable mention.

**Fluent Bit.** Written in C, tiny footprint, CNCF project under the Fluentd umbrella, and completely vendor neutral. Its Loki output plugin is one of about a hundred outputs it supports, which is the point: if you ever want to fan the same logs out to S3 and Loki, or swap Loki for something else, the collector layer doesn't change. It's also already running in a lot of clusters, because it's what most managed Kubernetes log forwarding is built on.

**Grafana Alloy.** Grafana's distribution of the OpenTelemetry Collector, and the official successor to Promtail. If you're already all-in on Grafana (Loki, Mimir, Tempo, Pyroscope), Alloy is the obvious answer. It handles metrics, logs, traces and profiles in one agent with one config language, and Grafana ships a conversion tool that turns an old Promtail config into an Alloy one.

**Promtail.** EOL as of March 2026. Nobody starting today should pick it. The problem is that a large share of the Loki documentation that exists on the internet assumes it, so you'll keep finding Promtail examples and have to translate them.

**Vector** is worth a mention: fast, good transform language, popular with people who want to do real processing in the pipeline rather than at query time.

The honest version is that there isn't a winner, and this decision is far less consequential than the two above it. Fluent Bit if you want backend independence or already run it. Alloy if you're committed to the Grafana stack and want one agent for logs, metrics, traces and profiles. Both are correct. The wrong answer is Promtail in 2026, and the *really* wrong answer is installing a chart that picks Promtail for you without you noticing.

Whichever you pick, the collector is where **label design** happens, and that is consequential. That is the next section, and it applies identically to all of them.

One thing worth knowing if you are hoping to consolidate: none of these collect traces. Fluent Bit has OpenTelemetry input and output plugins, so it can forward OTLP, but it cannot produce spans, and Loki cannot store them. Traces need instrumented applications and a separate backend. A logging pipeline does not get you part of the way there.

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

## Multi-tenancy: decide on day one, not day thirty

Loki separates data by **tenant**, identified by an `X-Scope-OrgID` header on every request. Turn it on with:

```yaml
loki:
  auth_enabled: true
```

Two things about this are worth internalising before you decide.

**The tenant is a path prefix in object storage.** Not a filter applied at query time, a physical partition:

```
my-bucket/
  team-a/<chunks>
  team-b/<chunks>
```

Data written as one tenant is invisible to another. There is no cross-tenant query.

**With `auth_enabled: false` you are already using a tenant.** Loki hardcodes one called, literally, `fake`, and everything lands under `fake/`. So this is not "off versus on", it is "one tenant named fake versus tenants you chose". The moment you enable auth and define real names, everything under `fake/` becomes unreachable. It is not deleted, it just sits there until retention expires.

Which means the cost of adopting tenancy grows every single day. On day one it is free. After a month it is a migration or a gap in your history. Even if you only ever want one tenant, name it something real.

### The neat part: auth and tenancy from one mechanism

The obvious implementation is to configure the tenant header in every client. Do not. The chart's nginx gateway can derive it from basic auth instead:

```yaml
loki:
  loki:
    auth_enabled: true
    tenants:
      - name: cluster-a
      - name: cluster-b
  gateway:
    basicAuth:
      enabled: true
      existingSecret: loki-gateway-htpasswd
```

which produces, in the generated nginx config:

```nginx
proxy_set_header X-Scope-OrgID $remote_user;
```

The tenant comes from **the authenticated username**. Three consequences worth having:

- a client cannot write into another tenant regardless of what headers it sends, because it does not choose the header
- shippers need no tenant configuration at all, just `http_user` and `http_passwd`
- rotating a credential rotates auth and tenant routing together

Use `existingSecret` rather than `tenants[].passwordHash`. Both work, but the second puts a bcrypt hash in git, and there is no reason to treat it differently from any other credential. The tenant list still has to be present with names, because the nginx snippet above is gated on it being non-empty.

### Grafana needs one datasource per tenant

There is no merged view. Each datasource carries its own header:

```yaml
- name: logs-cluster-a
  type: loki
  url: http://loki-query-frontend:3100
  jsonData:
    httpHeaderName1: X-Scope-OrgID
  secureJsonData:
    httpHeaderValue1: cluster-a
```

Note the URL points at the **query frontend**, not the gateway. Routing Grafana through the gateway would mean giving it basic auth credentials, which then live wherever your datasource config lives. The gateway is the auth boundary for traffic arriving from outside the cluster; in-cluster reads can go straight to the read path.

Get this wrong and the symptom is a bare `401 Authorization Required` from nginx, which looks like Loki is broken rather than like a datasource pointing at the wrong port.

And if a request arrives with no tenant header at all, Loki answers **404 with `no org id`**. Not 401, not 403. A 404 sends people looking for a missing route rather than a missing header, and that misdirection is worth knowing in advance.

## Shipping from other clusters: the DNS trap

If several clusters ship into one Loki, the shippers have to reach it, and this is where an assumption cost me half a day.

Both of my clusters were on the same VPN. The internal hostname resolved fine from my laptop. It did not resolve from inside a pod:

```
grafana.other-cluster.example.com  ->  10.105.40.228   from a laptop on the VPN
                                       NXDOMAIN        from a pod in the other cluster
```

**The pods are not VPN peers.** My laptop was a mesh peer and could route that private address. The pods used cluster DNS, which forwards to public resolvers, and had no route to the private range even if they could resolve it. Making the internal path work would have meant forwarding cluster DNS to the mesh resolver *and* routing the pod network onto the mesh. Neither is a five minute change.

Two smaller things in the same area:

- **An Ingress does not publish DNS.** It tells the ingress controller how to route a request that already arrived. Creating one does not make the name resolvable, and I have watched more than one person assume it does.
- **Check for a wildcard before assuming one.** I assumed `*.cluster.example.com` existed. It did not, on either the public or the internal side, so every hostname needed its own record and changing the hostname changed nothing.

The workable answer was a public name pointing at the ingress, with TLS and per-tenant basic auth. Worth being explicit that this puts a log ingestion endpoint on the internet. Basic auth over TLS is defensible; an IP allowlist on the ingress is a cheap improvement; mTLS is better if you already have a CA. Decide deliberately rather than by default.

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

## The failures that don't announce themselves

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

### 5. An object store that corrupted everything, with a 200 response

This one took the longest and is the one I would most want someone else to be warned about.

Symptom: the index gateway in CrashLoopBackOff.

```
table-name=loki_index_20690 msg="sync failed" err="gzip: invalid header"
error initialising module: store
```

I pulled one of the `.tsdb.gz` index objects out of the bucket and looked at the bytes:

```
32 64 37 0d 0a 1f 8b 08 00 ...
"2  d  7  \r \n"   then 1f 8b, the real gzip magic
```

`2d7` is hex for 727, which was exactly the object's size. That is an **HTTP chunked transfer-encoding length header stored inside the object body**. The chunk files had it too.

I then reproduced it with ten bytes and no Loki involved at all:

```
aws s3api put-object  "ABCDEFGHIJ"   ->  read back as "a\r\nABCDEFGHIJ"
HeadObject ContentLength: 10          <- metadata correct, body framed
server: S3 Server
```

The object store accepted `aws-chunked` streaming uploads and stored the framing as content. No error, 200 responses throughout.

I got the fix wrong twice before getting it right, and the wrong turns are the useful part:

1. **`AWS_REQUEST_CHECKSUM_CALCULATION=when_required`.** This genuinely fixes the `aws` CLI, which is how I proved the server was at fault. It did nothing for Loki, because those are AWS SDK **v2** variables and Loki's chunk client does not read them. I confirmed by writing an object after the variable was live and finding the framing still there.
2. **Switching to a different client library entirely** (thanos objstore, minio-go). Still framed. It streams too.
3. **`send_content_md5: true`.** This worked. The client computes an MD5 up front and issues a single PUT with `Content-MD5` instead of a streaming signature, so there is no framing for the server to mishandle.

The lesson is not about any of those settings. It is that I spent two attempts assuming my client was misconfigured, and about ninety seconds proving the server was broken once I finally tested the server directly with the smallest possible payload. **When something is corrupt end to end, bisect the pipeline before tuning either end of it.**

Also worth noting: existing objects stay corrupt. Fixing the writer does not repair what is already there, so the crash continues until you purge them.

### 6. A dead pod that kept the new ones from starting

After migrating from Monolithic to Distributed, the ingesters ran but never became ready:

```
found an existing instance(s) with a problem in the ring, this instance cannot
become ready until this problem is resolved
err="instance 10.244.7.47:9095 past heartbeat timeout"
```

The old Monolithic pod had run with `-target=all`, which means it had registered itself in the ingester ring. Deleting the pod did not remove that registration; it persisted in the ring as UNHEALTHY, and Loki refuses to bring new ingesters ready while an unhealthy member is present.

Nothing in any values file could have fixed it. It was leftover runtime state from the previous topology, and the fix was one POST to the distributor's `/ring` endpoint to forget the dead instance.

If you migrate between deployment modes, check `/ring` first when ingesters hang at Running but never Ready.

### 7. Helm replaces lists, and the failure is far from the cause

The ingester StatefulSet created no pods at all:

```
create Claim data-loki-ingester-0 failed:
PersistentVolumeClaim is invalid: spec.accessModes: Required value
```

I had overridden `ingester.persistence.claims` to set a storage class and size. `claims` is a **list**, and Helm replaces lists wholesale rather than merging them, so my entry silently dropped the `accessModes` the chart default supplied.

This is a general Helm trap rather than a Loki one, but it is worth internalising because the error surfaces in the StatefulSet controller, several layers from the values file you edited. Any time you override a list, you own every field in it.

The same class of thing bit me with a config block that was a *string*: overriding it replaced the whole block, so the parts I did not repeat vanished.

### 8. Defaults tuned for someone else's cluster

Three that were quietly wrong for a modest install:

- Every Distributed component defaults to `replicas: 0`. Setting `deploymentMode: Distributed` on its own deploys nothing.
- `chunksCache` defaults to an **8GB** memcached.
- `zoneAwareReplication` defaults to on, which renders `ingester-zone-a/b/c` and expects topology zone labels that plenty of clusters do not have.

And one that was wrong *because* I changed something else: the compactor's `delete_request_store` was set to `filesystem`, which had been fine when common storage was a filesystem path. Once it became object storage, the filesystem delete store had no directory configured and fell back to `/index` at the container root:

```
init compactor: failed to init delete store: rm /index: read-only file system
```

Changing your storage backend can invalidate settings that have nothing obviously to do with storage.

## The checklist

Condensed, in decision order:

1. **Deployment mode.** `Distributed` for anything production. Monolithic only as a throwaway, and know that moving off it is a rebuild, not a values change.
2. **Object storage from the start.** It is a correctness requirement in Distributed, not a capacity choice. Verify a round trip yourself, list, put, get and delete, before pointing Loki at it. Delete matters: the compactor needs it to enforce retention.
3. **Turn on multi-tenancy immediately**, even for one tenant. The tenant is a storage path prefix, so adopting it later strands everything already written.
4. **Derive the tenant from auth** rather than configuring headers in clients, and keep the credential in a secret rather than in values.
5. **Design labels before the first line ships.** Namespace, container, stream. Never pod, never node, never anything unbounded. Those go in structured metadata.
6. **Set every Distributed component's replica count.** They all default to zero.
7. **Enable ingester persistence.** The default is emptyDir and loses unflushed data on restart.
8. **Check what your object store does with streamed uploads** if it is not AWS S3. Ten bytes in, ten bytes out.
9. **If shippers are outside the cluster**, confirm a pod can actually resolve and reach the endpoint. Not your laptop, a pod.
10. **Decide what protects the write endpoint** before you expose it.

Then, once it is running, three checks that would have caught most of the above on the first afternoon rather than the third day:

- Query for a pod you know exists using `| pod="..."`, not `{pod="..."}`, and confirm you get lines back.
- Look at the raw content of one of those lines and confirm it is the application's output and not a JSON envelope wrapped around it.
- Pull one object out of the bucket and check the first two bytes are what you expect. For a `.gz`, that is `1f 8b`.

## Ending notes and links

- [Loki Helm chart docs](https://grafana.com/docs/loki/latest/setup/install/helm/), and the [migration guide to the community chart](https://grafana.com/docs/loki/latest/setup/upgrade/upgrade-to-community/) if you're already running the old one
- [Structured metadata](https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/), the page I wish I'd read first
- [Fluent Bit Loki output plugin](https://docs.fluentbit.io/manual/data-pipeline/outputs/loki.md) for the full option list
- [Fluent Bit Kubernetes filter](https://docs.fluentbit.io/manual/data-pipeline/filters/kubernetes.md), including `Buffer_Size`
- [Migrating from Promtail to Alloy](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/) if you inherited a Promtail setup

Thanks for your time.
