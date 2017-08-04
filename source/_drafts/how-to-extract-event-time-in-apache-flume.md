---
title: How to Extract Event Time in Apache Flume
tags: [flume, etl, java]
categories: [Big Data]
---

<!-- more -->

why? hdfs; why not current time? time lag.
reg extract + time interpreter, e.g. access log
reg replace (second -> millisecond), e.g. json
custom interceptor, e.g. json
already in header?
kafka to hdfs -> timestamp
