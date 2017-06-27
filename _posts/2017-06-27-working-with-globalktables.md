---
layout: post
title: "Working with GlobalKTable"
description: "Kafka Streams GlobalKTable"
date: 2017-06-28
tags: [kafka streams]
comments: false
share: true
---

* GlobalKTable are fully populated before actual data processing starts.
* messages with `null` keys are ignored, so make sure you input stream contains a key. If not, first map your stream and then join.
