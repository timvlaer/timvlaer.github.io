---
author: eskimologie
comments: true
date: 2017-04-15 21:37:35+00:00
excerpt: In Apache Kafka, the difference between the consumer offset and the end of
  the topic is called the lag. If the lag is increasing, the consumer is down or cannot
  keep up the producer.
layout: post
link: https://timswritings.wordpress.com/2017/04/15/monitor-kafka-consumer-lag/
slug: monitor-kafka-consumer-lag
title: Monitor Kafka Consumer lag
wordpress_id: 4
---

In [Apache Kafka](https://kafka.apache.org/), the difference between the consumer offset and the end of the topic is called the **lag**. If the lag is increasing, the consumer is down or cannot keep up the producer.

In Kafka producing and consuming messages are [completely disconnected](https://kafka.apache.org/documentation/#intro_topics). While producers keep appending messages to a topic, consumers can read these messages at their own pace (offset). In most streaming applications you want to make sure a consumer doesn't get too much behind.

The command line tool _kafka-consumer-groups_ lists consumer groups with their topics and its lag for every partition.

    
    > ./kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
    consumer-group-1
    > ./kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group consumer-group-1
    Group Topic Pid Offset logSize Lag Owner
     consumer-group-1 my-topic 0 123 126 3 test_c
     consumer-group-1 my-topic 1 155 155 0 test_c


At [Sentiance](http://www.sentiance.com/), we gather this information every minute, sum it and push it to [AWS Cloudwatch](https://aws.amazon.com/cloudwatch/). In these figure you see three consumer groups with their lag over time.

[caption id="attachment_22" align="alignnone" width="657"]![Schermafbeelding 2017-05-03 om 20.59.58](https://timswritings.files.wordpress.com/2017/04/schermafbeelding-2017-05-03-om-20-59-58.png) No issues here, a spike with a lag of 60 messages but the consumer rapidly catches up[/caption]
