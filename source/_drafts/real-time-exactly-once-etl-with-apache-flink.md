---
title: Real-time Exactly-once ETL with Apache Flink
tags: [flink, kafka, hdfs, java, etl]
categories: Big Data
---

Apache Flink is another popular big data processing framework, which differs from Apache Spark in that Flink uses stream processing to mimic batch processing and provides sub-second latency along with exactly-once semantics. One of its use cases is to build a real-time data pipeline, move and transform data between different stores. This article will show you how to build such an application, and explain how Flink guarantees its correctness.

![Apache Flink](/images/flink/arch.png)

## Demo Application

Let us build a project that extracts data from Kafka and loads them into HDFS. The result files should be stored in bucketed directories according to event time. Source messages are encoded in JSON, and the event time is stored as timestamp. Samples are:

```
{"timestamp":1545184226.432,"event":"page_view","uuid":"ac0e50bf-944c-4e2f-bbf5-a34b22718e0c"}
{"timestamp":1545184602.640,"event":"adv_click","uuid":"9b220808-2193-44d1-a0e9-09b9743dec55"}
{"timestamp":1545184608.969,"event":"thumbs_up","uuid":"b44c3137-4c91-4f36-96fb-80f56561c914"}
```

The result directory structure should be:

```
/user/flink/event_log/dt=20181219/part-0-1
/user/flink/event_log/dt=20181220/part-1-9
```

<!-- more -->

### Create Project

Flink application requires Java 8, and we can create a project from Maven template.

```
mvn archetype:generate \
  -DarchetypeGroupId=org.apache.flink \
  -DarchetypeArtifactId=flink-quickstart-java \
  -DarchetypeVersion=1.7.0
```

Import it into your favorite IDE, and we can see a class named `StreamingJob`. We will start from there.

* demo
    * create project
    * source, partition number
    * streaming file sink with timestamp
    * hadoop 2.7, row vs bulk
    * deploy & monitoring
* how flink ensures exactly-once
    * kafka
    * checkpoint
    * sink
* misc
    * parallelism




## References

