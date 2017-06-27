---
layout: post
title: "Working with GlobalKTable"
description: "Kafka Streams GlobalKTable"
date: 2017-06-28
tags: [kafka streams]
comments: false
share: true
---

KTables are the way to go when working with state in Kafka Streams. Basically they are a materialized view on a topic where every message is an upsert of the last record with the same key. (Read [duality of streams and tables](http://docs.confluent.io/current/streams/concepts.html#duality-of-streams-and-tables)). 

KTables eventually contains every change published to the underlying topic.

Normal KTables only contain the data of the partitions consumed by the Kafka Streams application. If you run *N* instances of your application, a KTable will contain roughly `total entries / N` entries. Every instance consumes a different set of partitions and thus reads another part of the data resulting in different KTable content. In case you want to join two KTables, you have to make sure data of the two streams is on the right machine ([data co-partitioning](http://docs.confluent.io/current/streams/developer-guide.html#streams-developer-guide-dsl-joins-co-partitioning)). 

In case your data is not partitioned with the right KTable key, you have to repartition your data. This is something that will happen anyway when you create a KTable from a stream via a [KGroupedStream](https://kafka.apache.org/0102/javadoc/org/apache/kafka/streams/kstream/KGroupedStream.html).

If you'd like to have the same state in every application, you'll have to use a GlobalKTable. These are very similar to KTables but contains the full state. So every instance of your application will read the full underlying topic and eventually have the same state. 

Some import remarks when you work with GlobalKTables:

* GlobalKTables are fully populated before actual data processing starts. If your underlying topic is big, this might take a while. It might make sense so tune retention period and log compaction on that topic. 
* When you join a KStream with a GlobalKTable, messages with `null` keys are ignored. Make sure you input stream contains a key. If not, first map your stream and then join.

```
stream
    .map((k, v) -> new KeyValue(v.getId(), v))
    .join(...)
```