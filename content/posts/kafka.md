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

The file name is the offset of the first record in that segment. An offset is just a number: the position of a record in its partition, starting at zero and going up by one with each append. It is the only identity a record has. The 2011 paper is explicit that this was a choice. There are no message IDs, so there is no seek-heavy index mapping IDs to disk locations, and the offset of the next message is the current offset plus the current message's length.

A topic is a named set of partitions. `num.partitions` defaults to 1, so a topic created without saying otherwise is a single log on a single broker. Give it twelve partitions and the records are spread across twelve independent logs, which can live on twelve brokers.

```bash
kafka-topics.sh --bootstrap-server broker:9092 --create \
  --topic orders --partitions 12 --replication-factor 3
```

Which partition a record lands in is decided by the producer, not the broker. If the record has a key, the producer hashes the key and takes it modulo the partition count, so every record with key `customer-42` goes to the same partition for as long as the partition count does not change. If there is no key the producer spreads records across partitions. That hash is the whole mechanism behind ordering, and I come back to it.

Replication is per partition too. Each partition has one leader and some followers, placed on different brokers; `--replication-factor 3` means one leader and two followers. The leader takes every write and, by default, serves every read. Followers fetch from the leader the way a consumer would and append to their own copy. The leader tracks which followers are caught up, and that set is the ISR, the in-sync replicas. A record is committed once every replica in the ISR has it, and the docs put the guarantee plainly: with f+1 replicas, a partition survives f failures without losing a committed record. The producer's `acks` setting decides whether it waits for that commit. `acks=all` with `min.insync.replicas=2` on a three-replica topic is the usual durable configuration, and it means a write is refused rather than quietly under-replicated when two of the three replicas are down.

A consumer group is how reads are shared out. Every consumer joins with a `group.id`. Within a group each partition is assigned to exactly one consumer, so a twelve-partition topic can be read by up to twelve consumers in parallel, and a thirteenth sits idle. Two groups with different IDs do not know about each other at all. Each gets every record.

## What happens on a write, and on a read?

A producer batches records per partition, compresses the batch if told to, and sends it to the leader of that partition. The leader appends the batch to the end of the active segment. It writes to the filesystem, which means the page cache; it does not `fsync` per record, and the docs are open about that: durability comes from replication, not from flushing. Followers fetch the new bytes, append them, and report their position. Once the ISR has caught up the leader advances the high watermark and the records become visible to consumers. If the producer asked for `acks=all`, that is when it hears back.

A read is a fetch request: partition, offset, maximum bytes. The broker finds the segment, looks up the byte position in the index, and ships bytes from there. If nothing new has arrived, the request can wait (`fetch.max.wait.ms`) rather than return empty, which is how a pull model avoids a tight polling loop. The consumer gets a batch, hands records to the application, and remembers the next offset to ask for.

Notice who holds state. The broker keeps the records and knows the end of each log. It does not know who has read what. The consumer's position is one integer per partition, and committing an offset means writing that integer somewhere it will survive a restart; modern Kafka stores it in an internal topic called `__consumer_offsets`, which is itself a log. A consumer that crashes picks up from its last committed offset. One that wants to reprocess sets its offset back and reads again. Nothing on the broker objects, because nothing on the broker was tracking it.

```
$ kafka-consumer-groups.sh --bootstrap-server broker:9092 --describe --group billing
GROUP    TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
billing  orders  0          184203          184211          8
billing  orders  1          183990          183990          0
```

That output (trimmed of the consumer ID and host columns) is the whole consumer-side model in two numbers. `LOG-END-OFFSET` is the broker's fact. `CURRENT-OFFSET` is the group's claim. Lag is the difference.
