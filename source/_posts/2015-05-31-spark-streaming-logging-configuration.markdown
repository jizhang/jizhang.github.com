---
layout: post
title: "Spark Streaming Logging Configuration"
date: 2015-05-31 18:18
comments: true
categories: [Notes, Big Data]
published: false
---

Spark Streaming applications tend to run forever, so their log files should be properly handled, to avoid exploding server hard drives. This article will give some practical advices of dealing with these log files, on both Spark on YARN and standalone mode.

## Log4j's RollingFileAppender

Spark uses log4j as logging facility. The default configuraiton is to write all logs into standard error, which is fine for batch jobs. But for streaming jobs, we'd better use rolling-file appender, to cut log files by size and keep only several recent files. Here's an example:

```properties
log4j.rootLogger=INFO, rolling

log4j.appender.rolling=org.apache.log4j.RollingFileAppender
log4j.appender.rolling.layout=org.apache.log4j.PatternLayout
log4j.appender.rolling.layout.conversionPattern=[%d] %p %m (%c)%n
log4j.appender.rolling.maxFileSize=50MB
log4j.appender.rolling.maxBackupIndex=5
log4j.appender.rolling.file=/var/log/spark/${dm.logging.name}.log
log4j.appender.rolling.encoding=UTF-8

log4j.logger.org.apache.spark=WARN
log4j.logger.org.eclipse.jetty=WARN

log4j.logger.com.anjuke.dm=${dm.logging.level}
```

This means log4j will roll the log file by 50MB and keep only 5 recent files. These files are saved in `/var/log/spark` directory, with filename picked from system property `dm.logging.name`. We also set the logging level of our package `com.anjuke.dm` according to `dm.logging.level` property. Another thing to mention is that we set `org.apache.spark` to level `WARN`, so as to ignore verbose logs from spark.

<!-- more -->
