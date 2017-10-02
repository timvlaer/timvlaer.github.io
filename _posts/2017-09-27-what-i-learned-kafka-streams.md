---
layout: post
title: "What I learned from working with Kafka Streams"
description: ""
date: 2017-10-02
tags: [kafka, kafka streams]
comments: true
share: true
---

### ALWAYS implement `UncaughtExceptionHandler`
If an exception occurs, the StreamThread will be exited, but the streams application will keep running. 
An application without StreamThreads won't make much progress and is completely useless. If you run a managed service, better call streams.stop() here.

### Always provide a timeout for the `streams.close()` method
It dares to hang, so better be safe. Especially when you're trying to shut down your application.

### Try to avoid blocking the loop between `poll()`'s longer than `max.poll.interval.ms`
It will cause rebalancing with quite some overhead that you could easily avoid by keeping an eye on processing times.

Keep an eye on rebalancing times, e.g.
```java
streams.setStateListener(new KafkaStreams.StateListener() {
  private Timer.Context rebalancingTimer = null;

  @Override
  public void onChange(KafkaStreams.State newState, KafkaStreams.State oldState) {
    if (KafkaStreams.State.REBALANCING.equals(newState)) {
      rebalancingTimer = FirehoseMetricRegistry.get().timer("rebalancing").time();
    } else if (rebalancingTimer != null && KafkaStreams.State.REBALANCING.equals(oldState)) {
      rebalancingTimer.stop();
    }
  }
});
```

### Increase the replication factor of internal topics
Set the kafka stream configuration parameter `replication.factor` to the same value as your input topics.

Kafka Streams will assume that data written to intermediate topics is durable. 
If a broker crashes, you might loose data if it has only one replica.

### Lower the commit interval when you work with stateful applications
Updates to changelog topics will only flow through at every commit interval, which is 30 seconds by default.

Caching is important, but if you'd like to maintain a reasonable throughput on less frequently changing KTable entries, you better set it to 1 second or disable it at all. 

You'll have to lower the value for sure when writing tests.  

### Prefer using the ByteArray Serde and do the actual deserialization yourself
In the current versions of Kafka Streams (and the Kafka Java Consumer in general) to handle serialization exceptions yourself.
https://docs.confluent.io/current/streams/faq.html#streams-faq-failure-handling-deserialization-errors-serde


### Measure the throughput / processing time
* By measuring 
** kafka lag (difference between current offset and topic size), 
** offset rate (growth rate of the offset),
** topic size rate (growth rate of the topic)
* Exact time it took to process a message, you can add this transformer in the beginning of the flow:
```java
@Override
public KeyValue<byte[], Trigger> transform(byte[] key, Trigger value) {
  Timer.Context timerContext = processingTime.time();
  context.forward(key, value);
  timerContext.stop();
  return null;
}
```

### Kafka Streams is blocking
Parallelism is realized using Kafka partitions. This makes reasoning about your applications easy.
But if you need external resources, you'll see performance drop dramatically because messages are processed one at the time. 