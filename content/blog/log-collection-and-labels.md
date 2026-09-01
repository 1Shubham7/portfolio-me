---
title: "How Log Collectors Collect and Label Your Logs"
description: "A container log line on disk contains no idea which pod wrote it. Everything you filter by later is constructed by the collector at collection time. What the file actually looks like, how collectors find it, where every label comes from, and why that design decides the health of your logging stack."
dateString: September 2026
draft: false
tags: ["Kubernetes", "Logging", "Observability", "DevOps", "SRE", "Containers"]
weight: 1
cover:
    image: "/blog/log-labels/cover.png"
---

Most people picture logging like this: an application emits a log line, the line carries context about where it came from, and a backend indexes that context so you can search it.

That is not what happens. The truth is stranger and more useful to know:

**A container log line, as written to disk, contains no information about which pod, namespace, or container produced it.**

Every field you filter by later is constructed after the fact by your log collector. Not read, not passed through. Constructed, using information that does not exist in the log line itself.

Once you hold that picture, a lot of otherwise baffling behaviour becomes predictable, and a set of design decisions that look like tuning knobs turn out to be the difference between a logging stack that stays healthy for years and one that quietly rots. This applies whether you run Fluent Bit, Alloy, Vector or Fluentd, and whether the logs land in Loki, Elasticsearch, OpenSearch, OpenObserve or anything else. The vocabulary differs. The architecture does not.

## Where a log actually comes from

Follow the path backwards from your query to the source.

Your application writes to stdout. It does not know it is in a container, let alone a pod. The container runtime captures that stream and writes it to a file on the node's filesystem. The collector, running as a DaemonSet, reads that file.

Nothing in that chain involves your application knowing anything about Kubernetes, which is exactly why the log line cannot carry Kubernetes context.

Under CRI runtimes, containerd and CRI-O, the file lives at a path shaped like this:

```
/var/log/containers/<pod>_<namespace>_<container>-<containerid>.log
```

For example:

```
/var/log/containers/checkout-api-7d9f8b6c4-x2mnp_payments_checkout-api-3f8a....log
```

That path is usually a symlink into `/var/log/pods/`, which is where the kubelet actually organises things. Both matter, because different collectors are configured against different ones.

A line inside is four fields:

```
2026-09-02T08:03:51.071571564Z stdout F {"level":"info","msg":"charge accepted"}
```

- an RFC3339Nano timestamp, written by the runtime, not the app
- the stream, `stdout` or `stderr`
- a flag, `F` for a full line or `P` for a partial one
- the message, exactly as the app wrote it

That is the whole format. If your runtime is older Docker, the format is a JSON object per line with `log`, `stream` and `time` keys instead, which is why collectors ship parsers for both.

Read those four fields again and notice what is missing. No pod. No namespace. No container name. No node, no image, no labels, no annotations.

**The only place that information exists is the filename.**

## Stage one: finding and reading the files

The first job is unglamorous and full of edge cases.

A tail input watches a directory glob, typically `/var/log/containers/*.log`, and follows every file it matches. Three problems come with that.

**Rotation.** The kubelet rotates container logs at a size limit, moving the current file aside and starting a new one. A naive tailer either loses the tail of the old file or starts the new one from the wrong offset. Collectors handle this by tracking inodes rather than paths, and by waiting a configurable grace period before letting go of a rotated file so late writes still get read.

**Restarts.** If the collector restarts, it must not re-ship everything it has already sent, and must not skip what arrived while it was down. So it keeps a position database on the node, mapping each file to a byte offset. This is a real file on disk, and if you delete it, you will get duplicates or gaps.

**Multi-line entries.** A Java stack trace is thirty lines in the file and one logical event. Worse, the runtime itself splits very long lines, which is what the `P` and `F` flag is for. Collectors reassemble these with a multiline parser, and getting it wrong is one of the more common reasons a stack trace shows up as thirty separate unhelpful entries.

None of this touches labelling yet. At the end of stage one, the collector has a record that looks roughly like:

```json
{
  "time":   "2026-09-02T08:03:51.071571564Z",
  "stream": "stdout",
  "log":    "{\"level\":\"info\",\"msg\":\"charge accepted\"}"
}
```

Still anonymous. What it also has is a **tag** derived from the file path, and that tag is the seed for everything that follows.

## Stage two: working out who this is

This is the stage that creates every label you will ever query on.

The collector parses the pod, namespace and container name out of the file path. It could stop there, and technically it would have three usable fields. Almost none of them do, because the path tells you the names and nothing else. It tells you nothing about labels, annotations, the owning workload, the image, or the node.

So the collector makes an HTTP call to the Kubernetes API server:

```
GET /api/v1/namespaces/payments/pods/checkout-api-7d9f8b6c4-x2mnp
```

and attaches what comes back to the record:

```json
{
  "time": "...", "stream": "stdout", "log": "...",
  "kubernetes": {
      "pod_name":        "checkout-api-7d9f8b6c4-x2mnp",
      "namespace_name":  "payments",
      "container_name":  "checkout-api",
      "container_image": "registry.example.com/checkout-api:1.4.2",
      "host":            "node-07",
      "labels":          { "app": "checkout-api", "tier": "backend" },
      "annotations":     { ... }
  }
}
```

There are four things worth understanding about this call, because between them they explain most of the ways logging pipelines fail.

**It is a network call, per pod, from every node.** On a large cluster with a lot of pod churn, the collector fleet is a meaningful source of API server load. This is why collectors cache aggressively, and why some offer the option of reading from the local kubelet instead of the API server, trading a little accuracy for a lot less central pressure.

**It is cached, and caches go stale.** Metadata is fetched on first sight of a pod and kept. If a pod is deleted and a new one takes its name, or labels change on a running pod, a collector with a long cache TTL keeps stapling the old answer onto new lines. Most of the time this does not matter. When you are debugging why a log line claims to belong to a workload that was renamed last week, it matters a lot.

**It needs permission.** The collector's ServiceAccount must be allowed to read pods, usually cluster-wide. That is a genuinely broad grant, because pod specs contain env vars, and env vars contain things people should not have put in env vars. It is worth knowing your log collector can read all of it.

**It can fail.** RBAC, network policy, an API server under pressure, a response larger than the client's buffer. Any of these means no metadata.

And what happens then is the part most people have never thought about.

## What collectors do when they cannot identify a pod

There are two defensible options, and essentially every collector picks the same one.

Option A is to drop the line. You never store anything you cannot attribute, and the failure is loud, because your log volume falls off a cliff.

Option B is to ship the line anyway, without metadata. You never lose data.

Everybody picks B, and it is the right call in isolation. Losing logs is worse than losing labels. But it has a consequence that is rarely spelled out:

**A log line that failed enrichment is still collected, still transmitted, still indexed, and still billed for. It is simply unfindable.**

It has no namespace, so no namespace filter matches it. It has no pod, so no pod filter matches it. It sits in whatever the backend's equivalent of an unlabelled bucket is, and every query you would naturally write skips straight past it.

Compare the two outcomes:

```
enriched:
  {cluster="prod", job="collector", namespace="payments",
   container="checkout-api", pod="checkout-api-7d9f8b6c4-x2mnp", stream="stdout"}

not enriched:
  {cluster="prod", job="collector", stream="stdout"}
```

Note which fields survive. The constants configured on the collector itself, and `stream`, because that came from the file format at stage one before enrichment ran. Everything sourced from the API server is gone. That gap is the signature of a failed lookup, and it is usually the only visible symptom.

Two more design choices compound the quiet. Output plugins **skip missing fields silently** rather than erroring, because you do not want one bad record to stall a pipeline. And label-indexed backends have **no schema**, so a record with two labels is exactly as valid as one with eight. Nothing anywhere in the chain considers this a problem.

## Stage three: deciding what becomes searchable

Now the collector holds a fully enriched record, and faces the decision that determines the long-term health of the whole system: which of those fields become searchable dimensions.

This is where terminology diverges but the concept does not.

- In label-indexed backends like Loki, indexed fields are **labels**, and every unique combination of label values is a distinct **stream** with its own chunks.
- In document-indexed backends like Elasticsearch, OpenSearch or OpenObserve, fields become part of the index mapping, and high-cardinality fields inflate the index and its memory footprint.

Different mechanics, same rule: **an indexed dimension costs you in proportion to how many distinct values it takes.**

So the fields split into two groups.

**Bounded values, safe to index.** Namespace, container name, stream, cluster, environment, the app label. You have a knowable number of namespaces. Container names are set at deploy time and rarely change. `stream` takes exactly two values. These make excellent primary dimensions because they carve your logs into a manageable number of big buckets.

**Unbounded values, do not index.** Pod name changes on every single deploy. Pod IP changes on every restart. Request ID, trace ID, user ID, session ID are effectively infinite. Indexing any of these multiplies your distinct dimension count without bound.

The second group still needs to be queryable, so collectors and backends offer a middle tier: fields that travel with the line and can be filtered on, but are not part of the index. Loki calls this structured metadata; document stores achieve the same effect with fields that are stored but not indexed. You end up querying in two steps:

```
select the bucket by indexed labels   ->   filter within it by metadata
```

which is the query pattern almost every log system converges on, once you know why.

The failure mode here deserves calling out, because it is the same shape as the enrichment failure: **it is silent, and it is delayed.** Put a pod name in the index and nothing breaks today. It breaks over months, as a gradual query slowdown and a growing index, and by then nobody connects it to a line in a config file that someone added at the start.

There is a decent rule of thumb. Before making a field an index dimension, ask how many distinct values it will have taken by the end of the year. If you cannot answer, or the answer scales with traffic, deploys or users, it is not an index dimension.

One last wrinkle: modern backends add dimensions of their own that you did not configure. Some derive a service name from the fields you did set, or infer a severity level by looking at the line content. Useful, and worth knowing about, because the set of labels you designed is not necessarily the set you end up with.

## Putting the whole path together

Four stages, and every one of them is a place where information is either created or lost:

```
[1] the runtime writes a line to a file on the node
        the line has a timestamp, a stream, and your message
        it has NO identity. the path is the only clue.

[2] the collector tails the file
        handles rotation, restarts, partial lines
        derives a tag from the path

[3] the collector asks the API server who this pod is
        attaches namespace, container, image, node, labels
        THIS is where every searchable field is born
        if it fails, the line ships anyway, anonymous

[4] the collector decides which fields become indexed
        bounded values -> index dimensions
        unbounded values -> non-indexed metadata
        this choice is permanent for every line already written
```

That last point is worth its own sentence. In most log backends, the dimensions attached to a line are fixed at ingestion and never recomputed. There is no re-labelling pass, no backfill, no reprocessing. Fix a broken collector and lines from that moment forward are correct; everything before it stays as it was until retention deletes it.

## What to take from this

Three things I would want someone to walk away with.

**The collector is not a pipe, it is an author.** It does not forward context, it manufactures it. Treat it as a component that can be wrong, not as plumbing that either works or does not.

**Ask what your collector does when the lookup fails.** For essentially every collector, the answer is "ships the line without metadata". That means "logs are arriving" and "logs are findable" are two different claims, and only the first one is easy to check. If you have a labelling scheme where every line should carry a namespace, then lines without one are a real signal worth alerting on, and it catches every cause at once rather than one cause at a time.

**Cardinality is a design decision, not a tuning knob.** The choice of which fields become index dimensions is made once, early, usually by whoever copied a config from a blog post, and it is expensive to revisit because it only applies going forward. Spend an hour on it before you have a year of logs written the wrong way.

The single sentence version: your logs do not know where they came from, something has to tell them, and everything downstream depends on that something being right.
