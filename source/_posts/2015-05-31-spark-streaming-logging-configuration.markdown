---
layout: post
title: "Spark Streaming Logging Configuration"
date: 2015-05-31 18:18
comments: true
categories: [Notes, Big Data]
published: false
---

Spark Streaming applications tend to run forever, so their logging files should be properly handled, to avoid exploding server hard drives. This article will give a brief introduction to log4j's rolling-file configuration, and how to use it in either Spark on YARN or standalone mode.

<!-- more -->
