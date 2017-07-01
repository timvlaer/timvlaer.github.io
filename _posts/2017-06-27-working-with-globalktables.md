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

KTables *eventually* contains every change published to the underlying topic.

Normal KTables only contain the data of the partitions consumed by the Kafka Streams application. If you run *N* instances of your application, a KTable will contain roughly `total entries / N` entries. Every instance consumes a different set of [Kafka partitions](https://kafka.apache.org/documentation.html#intro_topics) resulting in different KTable content. 

If you'd like to have the same state in every instance, you'll have to use a GlobalKTable. Its API is very similar to KTable but it contains the full state. So every instance of your application will read all partitions of the underlying topic and thus eventually have the same state. 

Scenario's to consider a GlobalTable over a normal one:
* to avoid [co-partitioning](http://docs.confluent.io/current/streams/developer-guide.html#streams-developer-guide-dsl-joins-co-partitioning) when you have to join a stream with multiple tables or two tables with each other. ([See this example](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=67633649#KIP-99:AddGlobalTablestoKafkaStreams-Example))
* the content of the table is limited or changes not very often
* the table key is not a good candidate for evenly distributed partitioning. When joining this table, some instances will have a much bigger workload than others. This makes it hard to scale the application.

Some remarks when you work with GlobalKTables:

* GlobalKTables are fully populated before actual data processing starts ([details](https://stackoverflow.com/questions/44827559/how-does-kafkastreams-determine-whether-a-globalktable-is-fully-populated-while/44829013#44829013)). If your underlying topic is big, this might take a while. It might make sense so tune retention period and log compaction on that topic. 
* As of release 0.11, [global tables checkpoints their offset](https://issues.apache.org/jira/browse/KAFKA-5241). This improves reboot time if you use a [durable state store](http://docs.confluent.io/current/streams/architecture.html#streams-architecture-state). This is the default, but doesn't make sense if you run your application inside a container without mount. 
* GlobalKTables are populated in a different thread (`client_id-GlobalStreamThread`) and a different consumer group per instance (`client_id-global`). ([See source code](https://github.com/apache/kafka/blob/trunk/streams/src/main/java/org/apache/kafka/streams/KafkaStreams.java#L364)) 
* When you join a KStream with a GlobalKTable, messages with `null` key or value are ignored and do not trigger a join. Make sure you input stream contains a key. If not, first map your stream and then join.
([Read more](http://docs.confluent.io/current/streams/developer-guide.html#kstream-globalktable-join))


```java
stream
    .map((k, v) -> new KeyValue(v.getId(), v))
    .leftJoin(...)
```
