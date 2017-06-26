---
layout: post
title: "Asynchronous API styles for microservices"
description: "Thinking about different communication approaches"
date: 2017-06-25
tags: [microservices, api, kafka]
comments: true
share: true
---

*(This is a draft)*


These days I work a lot with asynchronous microservices.
* For me, *microservices* are small, independently deployable applications with a very clear defined responsability (do one thing and do it well).
* With asynchronous, I mean these microservices get their input from a message queue (might be a Kafka topic) and produce their result to another queue. 
Due to this processing fashion, these services are easy to duplicate to increase the message throughput (horizontal scaling).


I currently see three different flavors of API design:
* request/response
* event sourced
* state upserts

### request/response
The request contains a correlation-id. The microservices responds with that same correlation-id. The consumer of the service listens on the output topic for its responses. 

### event-sourcing
The microservices communicates every internal state change with a business event. E.g. `SubscriptionCreated(id, userId, ...)`, `SubscriptionApproved(id, date)`, `SubscriptionRemoved(id)`

### state upserts
Every message is a full update of the object. It overrides the last message with the same key. 
* `1:{subscriptionId:1,userId:u1,approved:null,...,active:true}`
* `1:{subscriptionId:1,userId:u1,approved:1498425221681...,active:true}`
* `1:{subscriptionId:1,userId:u1,approved:1498425221681...,active:false}`