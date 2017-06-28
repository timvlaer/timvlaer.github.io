---
layout: post
title: "Embedded Redis as JUnit Rule"
description: "Use embedded Redis cluster for intergration testing"
date: 2017-06-28
tags: [redis, junit]
comments: false
share: true
---


I am very happy with the [EmbeddedRedis implementation of Krzysztof Styrc](https://github.com/kstyrc/embedded-redis) for integration testing.
To add some convenience, I created a JUnit ExternalResource from it, so I can add it as a ClassRule to my test. JUnit will then start and stop the cluster before and after the test suite.
  
I use a cluster setup of Redis, so it chooses a random free network port itself.  
  
```java
@ClassRule
public static final EmbeddedRedis REDIS = new EmbeddedRedis();
```

The implementation:

```java
import org.junit.rules.ExternalResource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import redis.embedded.RedisCluster;

public class EmbeddedRedis extends ExternalResource {
  private static final Logger logger = LoggerFactory.getLogger(EmbeddedRedis.class);

  private RedisCluster redisCluster;

  @Override
  protected void before() throws Throwable {
    redisCluster =  RedisCluster.builder().ephemeral().sentinelCount(1).quorumSize(1)
        .replicationGroup("master1", 1)
        .build();
    redisCluster.start();
  }

  @Override
  protected void after() {
   redisCluster.stop();
  }

  public String getHost() {
    return "localhost";
  }

  public int getPort() {
    return redisCluster.serverPorts().get(0);
  }
}
```