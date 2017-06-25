---
date: 2017-06-21
layout: post
title: Testing services using Kafka (Streams)
description: "Writing integration tests with embedded Kafka cluster."
tags: [kafka, kafka streams]
comments: false
share: true
---

While developing services based on [Kafka Streams](https://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple/), you like to test the full topology. Either you use a locally installed Kafka cluster or you use an **EmbeddedKafkaCluster**. In this article I show how to use the latter.

The EmbeddedKafkaCluster is included in the Kafka Streams Test project. It is not really documented, but you can find its usage in the test classes of the [Kafka Streams source code](https://github.com/apache/kafka/blob/ca8915d2efc225dbc0a4c138a2a34cf34d07e347/streams/src/test/java/org/apache/kafka/streams/KafkaStreamsTest.java).

Setup a test cluster like this:


```java
@ClassRule //make sure you use a recent junit version!
public static final EmbeddedKafkaCluster CLUSTER = new EmbeddedKafkaCluster(1);
private final MockTime mockTime = CLUSTER.time; // used to generate timestamps when producing test messages
````


You can pass extra [broker configuration](https://kafka.apache.org/documentation/#brokerconfigs) as a second argument to the EmbeddedKafkaCluster constructor.

```java
@BeforeClass
public static void createKafkaTopics() throws Exception {
  CLUSTER.createTopic("input_topic");
  CLUSTER.createTopic("result_topic");
}
```


That's it for the brokers! An actual test can look like this:

```java
@Test
public void testKafkaFlow() throws Exception {
  Properties producerConfig = new Properties();
  producerConfig.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, CLUSTER.bootstrapServers());
  producerConfig.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
  producerConfig.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
  IntegrationTestUtils.produceValuesSynchronously(ACTIONS_TOPIC, inputMessage, producerConfig, mockTime);

  Properties consumerConfig = new Properties();
  consumerConfig.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, CLUSTER.bootstrapServers());
  consumerConfig.put(ConsumerConfig.GROUP_ID_CONFIG, "junit-check-consumer");
  consumerConfig.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
  consumerConfig.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
  consumerConfig.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
  int expectedNumRecords = 1;
  int waitTime = 40000;
  List<KeyValue<String, String>> resultRecords = IntegrationTestUtils.waitUntilMinKeyValueRecordsReceived(consumerConfig, "result_topic", expectedNumRecords, waitTime);
  
  assertThat(resultRecords).hasSize(1);
  assertThat(resultRecords.get(0).key).isEqualTo(subscriptionId);
}
```



## My tests are slow!


Your application is stateful when you join, window or aggregate messages. These operators are executed on a state store (cache) and messages are not immediately pushed downstream. Messages will be forwarded either when the cache is full or when the commit interval is reached. See '[record caches in the DSL](http://docs.confluent.io/current/streams/developer-guide.html#record-caches-in-the-dsl)' in the documentation.

The commit interval defaults to 30 seconds. In case of integration testing, this is long and I suggest to decrease the commit interval. By doing so, KTables will faster propagate their changes.


```java    
streamsConfiguration.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
```


You can also disable the cache, but this might change the behaviour of your application.

```java    
// Disable record cache
streamsConfiguration.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 0);
````



## Dependencies


Not very well documented by the Kafka project itself, but these are the Maven dependencies you'll need. Notice the <**classifier**>.

```xml   
<dependency>
 <groupId>org.apache.kafka</groupId>
 <artifactId>kafka_2.11</artifactId>
 <version>${kafka.version}</version>
 <classifier>test</classifier>
 <scope>test</scope>
</dependency>
<dependency>
 <groupId>org.apache.kafka</groupId>
 <artifactId>kafka-clients</artifactId>
 <version>${kafka.version}</version>
 <classifier>test</classifier>
 <scope>test</scope>
</dependency>
<dependency>
 <groupId>org.apache.kafka</groupId>
 <artifactId>kafka-streams</artifactId>
 <version>${kafka.version}</version>
 <classifier>test</classifier>
 <scope>test</scope>
</dependency>
<!-- Scala library needed for EmbeddedKafkaCluster -->
<dependency>
 <groupId>org.scala-lang</groupId>
 <artifactId>scala-library</artifactId>
 <version>2.11.11</version>
 <scope>test</scope>
</dependency>
```
