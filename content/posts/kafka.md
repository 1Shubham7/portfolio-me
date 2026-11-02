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
