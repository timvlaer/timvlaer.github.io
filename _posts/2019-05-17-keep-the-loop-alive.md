---
layout: post
title: "How to keep the kafka consumer alive"
description: "Insights in the behaviour of the kafka consumer poll loop"
date: 2019-05-17
tags: [kafka]
comments: true
share: true
---

The kafka consumer has a very convenient api. You configure it with a couple of connection properties, subscribe to one or more topics 
and start polling for messages.

Under the hood, quite a few things happen to detect if the consumer is still active. There are basically two mechanisms, 
one to check if the consumer is not dead (heartbeat) and one to check if the consumer is actually making progress (poll interval).

A background thread is sending heartbeats every 3 seconds (`heartbeat.interval.ms`). If the group coordinator (one of the brokers) doesn't hear a heartbeat 
for more than 10 seconds (`session.timeout.ms`), the consumer is considered dead and kicked out of the group. 

On the other hand, the consumer itself validates if the client code is regularly calling the `poll` method. If the consumer takes more 
than 5 minutes (`max.poll.interval.ms`)  between two `poll` calls, the consumer will proactively leave the group 
and the partitions will be assigned to another consumer in the group. Be aware that with the next `poll`, the consumer actually 
rejoins the group causing another rebalance round. 
So if processing time is too high, you risk a lot of rebalances which can stop processing almost completely at worst.  
 
By default, the consumer will process 500 records per poll (`max.poll.records`). If the processing can take at most 5 minutes, 
your consumer can take up to 600ms processing time per record. 
See [KafkaConsumer#failure-detection](https://kafka.apache.org/22/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html#failuredetection)

Apart from fetching mesages, a `poll` request also checks the group metadata. 
This metadata contains the partition assignment and rebalance information. 
As long as the consumer doesn't poll, it will not be aware of any changes to the consumer group. 
With other words, `max.poll.interval.ms` is also the maximum amount of time a rebalance can take, 
since every consumer in the group needs at most that amount of time to check the consumer group metadata.
 
Usually a group rebalance takes less than 5 minutes, as consumers usually call the `poll` method often. 
In case record processing time is high, I'd suggest decreasing `max.poll.records` first to keep rebalances within a reasonable amount of time. 
If processing time is still too high, I suggest to decouple polling from processing. 

You can decouple polling by executing the actual processing in another thread and keep the main thread alive to execute the `poll` method.
Of course, in this case you want to disable new records coming in which you can do by pausing the consumer. 
See [KafkaConsumer#pause](https://kafka.apache.org/22/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html#pause-java.util.Collection-) 
If you want to play around with a asynchronous processing thread, I can recommend the 
[Akka Streams Kafka](https://doc.akka.io/docs/alpakka-kafka/current/home.html) library, it does this out of the box. 
