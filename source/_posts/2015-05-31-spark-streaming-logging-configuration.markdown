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

## Standalone Mode

In standalone mode, Spark Streaming driver is running on the machine where you submit the job, and each Spark worker node will run an executor for this job. So you need to setup log4j for both driver and executor. 

For driver, since it's a long-running application, we tend to use some process management tools like [supervisor](http://supervisord.org/) to monitor it. And supervisor itself provides the facility of rolling log files, so we can safely write all logs into standard output when setting up driver's log4j.

For executor, there're two approaches. One is using `spark.executor.logs.rolling.strategy` provided by Spark 1.1 and above. It has both time-based and size-based rolling methods. These log files are stored in Spark's work directory. You can find more details in the [documentation](https://spark.apache.org/docs/1.1.0/configuration.html). 

The other approach is to setup log4j manually, when you're using a legacy version, or want to gain more control on the logging process. Here are the steps:

1. Make sure the logging directory exists on all worker nodes. You can use some provisioning tools like [ansbile](https://github.com/ansible/ansible) to create them.
2. Create driver's and executors' log4j configuration files, and distribute the executor's to all worker nodes.
3. Use the above two files in `spark-submit` command:

```
spark-submit
  --driver-java-options "-Dlog4j.configuration=file:/path/to/log4j-driver.properties -Ddm.logging.level=DEBUG"
  --conf "spark.executor.extraJavaOptions=-Dlog4j.configuration=file:/path/to/log4j-executor.properties -Ddm.logging.name=myapp -Ddm.logging.level=DEBUG"
  ...
```
