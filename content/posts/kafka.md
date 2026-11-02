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

## Why is ordering per partition, not per topic?

Because a partition is one file appended by one leader, and a topic is several of them with nothing coordinating between them. Records in partition 0 are in the order they were appended to partition 0. Records in partition 0 and partition 7 have no defined order relative to each other, and Kafka does not pretend otherwise. The paper says it in one sentence: messages from a single partition are delivered in order, with no guarantee across partitions.

This is why the key matters. If every event about `customer-42` carries that key, every event about that customer is in one partition, in order, read by one consumer in the group. That is all the ordering most systems need. If you need a global order you need one partition, which means one leader and one consumer, and you have given up the parallelism that was the point.

Two consequences bite people. Adding partitions to a topic changes the modulo, so keys start hashing to different partitions, and the "all events for one key in one place" property no longer holds across the change. And a hot key is a hot partition: if one customer produces a tenth of all events, one partition carries a tenth of the topic and the consumer assigned to it does a tenth of the work alone.

## What happens to old data?

Nothing, until retention says so. By default a broker keeps a partition's segments for 168 hours (`log.retention.hours`) and has no size cap (`log.retention.bytes=-1`). Retention is enforced per segment: when a segment ages past the limit, the whole file is deleted. Whether anyone has read it is not part of the decision.

The alternative to deleting by age is compacting by key. With `cleanup.policy=compact` on a topic, a background cleaner rewrites old segments and keeps only the latest record for each key, so the topic becomes a changelog that always holds at least the last known value for every key. A record with a key and a null value is a tombstone: it says "delete this key" and is itself removed after `delete.retention.ms`. Compacted topics are how a consumer can rebuild a full table from Kafka without reading every update since the beginning of time, and they are what Kafka Streams uses to recover its own state after a crash. The two policies can be combined on one topic, in which case segments older than the retention limit are still deleted even though the topic is compacted.

## How is it different from a message queue?

I will use RabbitMQ as the queue, since it is the one most people have run, with SQS as the hosted cousin. Most of what follows applies to ActiveMQ and the rest of the JMS family too.

A queue is a broker that owns the message. A producer publishes; the broker routes it to a queue; when a consumer is ready the broker pushes it (`basic.consume`); the consumer acknowledges; the acknowledgement tells the broker the message can be discarded. If the consumer rejects it or dies, the broker requeues it or dead-letters it. Delivery state lives on the broker and the message's life ends at the ack. SQS is the same shape with different words: a received message is hidden for a visibility timeout and deleted when the consumer says so. Every detail of that design is about one message going to one consumer once.

Kafka inverts each of those. The broker does not push; the consumer pulls from an offset. The broker does not track delivery; the consumer commits its own position. Nothing is discarded on read; records stay until retention removes them. And the same record is read by every group that subscribes, not by whichever consumer got it first.

The differences people notice in practice follow from those.

**Replay.** With a queue, a message you have acknowledged is gone. If the code that handled it had a bug, you fix the bug and the data is not there to run through it again. With Kafka you reset the group's offset to a timestamp or a number and read forward. The paper called this out in 2011 as a violation of the common contract of a queue and an essential feature anyway, and the examples it gave are the same ones people give today: reloading a warehouse after a fix, and recovering a consumer that only checkpoints occasionally.

**Fan-out.** Getting a RabbitMQ message to several consumers means several queues bound to one exchange, each with its own copy, and a fourth consumer means a fourth binding. In Kafka a new consumer is a new `group.id`. The producer, the topic, and the existing readers are not touched. The data is written once and read N times, mostly out of the page cache.

**Ordering.** A single RabbitMQ queue is ordered until there are competing consumers and redeliveries, at which point it is not. Kafka's ordering is weaker on paper and stronger in practice: only per partition, but per partition for as long as the data exists, with the key as the lever.

**Throughput.** Kafka is fast for boring reasons. It writes sequentially and reads sequentially, and the docs make the case with a number: linear writes on a commodity disk run at hundreds of megabytes a second while random writes run at about a hundred kilobytes, a gap the docs put at over six thousand times. It keeps no in-process cache and relies on the kernel page cache, so a consumer that is keeping up reads from memory and a broker restart does not lose the cache. Producers and consumers move batches, not records, so one round trip carries thousands of records. And on the read path the broker uses `sendfile` to move bytes from the page cache to the socket without passing them through the JVM; the docs count four copies and two system calls on the ordinary path and a single copy to the NIC buffer on this one. (That optimisation is lost when clients talk TLS, because the broker has to encrypt the bytes first.) The 2011 paper's producer test is a fair summary of the result. With one broker and one producer, Kafka reached about 50,000 messages a second unbatched and about 400,000 with batches of 50, which the authors describe as orders of magnitude above ActiveMQ and at least twice RabbitMQ, and they note that the Kafka producer in that test did not wait for acknowledgements. They also say outright that the point was not that the other systems were inferior, since both had more features, but that a specialised system could be much faster.

**Where a queue is the better tool.** A queue routes per message: topic exchanges, header matching, a message that lands in three queues because of its routing key. Kafka routes per topic, and the producer picks the partition. A queue has priorities; Kafka has an append-only file, and the only order is arrival order. A queue tells you a specific message was handled and lets you drop a specific message; Kafka has a position in a log and no per-record delete. A queue does request/reply with a reply-to header and a correlation ID; in Kafka you build that yourself, and it is awkward. A queue gives per-message retry with a delay and a dead-letter destination; Kafka gives you an offset you can refuse to advance, which blocks the partition behind it, so dead-lettering is something you write. And a queue is small. RabbitMQ on one node with a few queues is an afternoon; a Kafka cluster is three brokers at minimum for replication worth having, and before 4.0 it needed a ZooKeeper ensemble beside it. If the job is "give each task to one worker and tell me when it is done", a queue is the right shape and Kafka is the wrong one.

## What do people use it for?

The short answer is anything where more than one system wants the same stream of facts. A few shapes come up constantly.
