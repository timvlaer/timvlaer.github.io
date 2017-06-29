---
comments: false
date: 2017-06-22
layout: post
title: Combining the DSL and Processor API of Kafka Streams
description: "Get the most out of the Kafka Streams library"
tags: [kafka, kafka streams]
comments: false
share: true
---

The Kafka Streams library consists of two API's:
	
  1. The high level, yet powerful Domain Specific Language (DSL). There you'll find the KStreams, KTables, filter, map, flatMap etc. If you are familiar with the Java 8 Streams API you'll find it easy to reason about this Kafka Streams DSL.
  2. The low-level, rather complex but full armed Processor API (PAPI) which gives you all the power of Kafka Streams. Here you'll have to route your messages from a source (input topic) via processors to a sink (output topic). The DSL is built on top of this Processor API.


For most use cases I'd like to stick to the DSL. It is expressive and self-explanatory.


```java
KTable<String, Action> actionKTable = new KStreamBuilder()
    .stream("actions_topic")
    .mapValues(Actions::gunzipAndDeserialize)
    .groupBy((key, action) -> action.getId(), Serdes.String(), ThriftSerde.forClass(Action.class))
    .aggregate(..., "aggregate_store");
```


But there are scenario's that you cannot express with the DSL, e.g.

  * Fan out multiple messages based on one incoming message	
  * Producing messages based on timing of overall processing, e.g. every 100 messages, every minute, invalidate state after a while...	
  * Write access to the state store of a KTable


In these cases it might be interesting to attach one of these powerful processors to the KTable result stream. (Heads up! You can only attach Processors to KStreams, so you'll have to transform the KTable to a stream.)


```java
actionKTable
    .toStream()
    .transform(() -> new ExpireActionsTransformer(), "aggregate_store")
    .to(Serdes.String(), new ActionSerde(), "output_topic");
```


In the code example below, I show you how the PAPI Transformer can
  * access the state store
  * generate messages (fan out) via the _ProcessorContext_
  * schedule itself to be executed every once in a while. (_schedule_ method on the _ProcessorContext_). The _punctuate_ method is called every _x_ time units based on the message timestamps (Heads up! [This is event time, not wall-clock time](https://github.com/confluentinc/examples/issues/86).)

 
```java   
public class ExpireActionsTransformer implements Transformer<String, Action, KeyValue<String, Action>> {
  private ProcessorContext context;
  private KeyValueStore<String, Action> subscriptionsStateStore;

  @Override
  public void init(ProcessorContext context) {
    this.context = context;

    // schedule this transformer and call this processor's punctuate() method every 1000 time units.
    this.context.schedule(1000); 

    // full read/write access to the KTable state store
    this.subscriptionsStateStore = (KeyValueStore<String, Action>) context.getStateStore("aggregate_store");
  }

  @Override
  public KeyValue<String, Action> transform(String key, Action value) {
    // transform key and/or value or use context.forward to produce multiple messages
    return new KeyValue<>(key, value);    
  }

  @Override
  public KeyValue<String, Action> punctuate(long timestamp) {
    ...
    context.forward(key, expiredAction);
    return null; // method should always return null
  }

  @Override
  public void close() {  }
}
```
