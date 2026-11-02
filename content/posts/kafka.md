---
title: "Kafka Is a Log, Not a Queue: What That Changes, and Why LinkedIn Built It"
description: "What Kafka actually is, from the partition file up: offsets, consumer groups, replication and retention, how it differs from a RabbitMQ-style queue and where a queue is still the better tool, and the LinkedIn problem that made a replicated log the right answer in 2010."
dateString: November 2026
draft: false
tags: ["Kafka", "Distributed Systems", "Messaging", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/kafka/kafka.png"
---

Kafka is usually introduced as a message queue that goes faster. That is the description I carried around for a long time, and it explains almost nothing. It does not explain why a consumer can read last Tuesday's records again, why two unrelated teams can read the same topic without stepping on each other, why ordering is only promised inside a partition, or why there is no "delete this message" call anywhere in the API.

All of those follow from one decision. Kafka is not a queue. It is a log: an append-only file, split across machines and copied for safety, that consumers read at their own pace while remembering their own position. Hold that one idea and most of the design falls out of it. This post is the model from the file up, a comparison with a real queue that tries to be fair to the queue, what people use it for, and the LinkedIn problem that made a log the right answer in the first place.

## What is it, actually?

Start with a single partition, because that is the unit everything else is built on. On disk a partition is a directory of segment files. Each segment is a plain file of records appended in order, plus an index that maps offsets to byte positions. When a segment reaches a size limit (`log.segment.bytes`, 1 GiB by default) the broker rolls a new one and leaves the old one alone.

```
$ ls /var/lib/kafka/data/orders-0/
00000000000000000000.index
00000000000000000000.log
00000000000000000000.timeindex
00000000000001048576.index
00000000000001048576.log
00000000000001048576.timeindex
```
