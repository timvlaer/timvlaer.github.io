---
layout: post
title: "What I learned from working with Kafka Streams"
description: ""
date: 2017-09-27
tags: [kafka, kafka streams]
comments: true
share: true
---

### Try to avoid blocking the loop between `poll()`'s longer than `max.poll.interval.ms`
It will mess up the rebalancing which might cause the whole consumer group to halt. 

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

### Prefer using the ByteArray Serde and do the actual deserialization in map function
Handle exceptions yourself



### Measure the throughput / processing time
* By measuring 
** kafka lag, 
** offset rate and,
** topic size rate
* By adding a transformer in the beginning of the flow
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