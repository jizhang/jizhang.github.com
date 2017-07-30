---
title: How to Achieve Exactly-Once Semantics in Spark Streaming
tags: [spark, spark streaming, kafka, stream processing, scala]
categories: [Big Data]
---

Exactly-once semantics is one of the advanced topics of stream processing. To process every message once and only once, in spite of system or network failure, not only the stream processing framework needs to provide such functionality, but also the message delivery system, the output data store, as well as how we implement the processing procedure, altogether can we ensure the exactly-once semantics. In this article, I'll demonstrate how to use Spark Streaming, with Kafka as data source and MySQL the output storage, to achieve exactly-once stream processing.

![Spark Streaming](http://spark.apache.org/docs/latest/img/streaming-arch.png)

## An Introductory Example

First let's implement a simple yet complete stream processing application that receive access logs from Kafka, parse and count the errors, then write the errors per minute metric into MySQL database.

Sample access logs:

```text
2017-07-30 14:09:08 ERROR some message
2017-07-30 14:09:20 INFO  some message
2017-07-30 14:10:50 ERROR some message
```

Output table, where `log_time` should be truncated to minutes:

```sql
create table error_log (
  log_time datetime primary key,
  log_count int not null default 0
);
```

<!-- more -->

Scala projects are usually managed by `sbt` tool. Let's add the following dependencies into `build.sbt` file. We're using Spark 2.2 with Kafka 0.10. The choice of database library is ScalikeJDBC 3.0.

```scala
scalaVersion := "2.11.11"

libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-streaming" % "2.2.0" % "provided",
  "org.apache.spark" %% "spark-streaming-kafka-0-10" % "2.2.0",
  "org.scalikejdbc" %% "scalikejdbc" % "3.0.1",
  "mysql" % "mysql-connector-java" % "5.1.43"
)
```

The complete code can be found on GitHub ([link][1]), so here only shows the major parts of the application:

```scala
// initialize database connection
ConnectionPool.singleton("jdbc:mysql://localhost:3306/spark", "root", "")

// create context
val conf = new SparkConf().setAppName("ExactlyOnce").setIfMissing("spark.master", "local[2]")
val ssc = new StreamingContext(conf, Seconds(5))

// create Kafka DStream
val messages = KafkaUtils.createDirectStream[String, String](ssc,
   LocationStrategies.PreferConsistent,
   ConsumerStrategies.Subscribe[String, String](Seq("alog"), kafkaParams))

// do transformation
messages.foreachRDD { rdd =>
  val result = rdd.map(_.value)
    .flatMap(parseLog) // parse log line into case class
    .filter(_.level == "ERROR")
    .map(log => log.time.truncatedTo(ChronoUnit.MINUTES) -> 1)
    .reduceByKey(_ + _)
    .collect()

  // store result into database
  DB.autoCommit { implicit session =>
    result.foreach { case (time, count) =>
      sql"""
      insert into error_log (log_time, log_count)
      value (${time}, ${count})
      on duplicate key update log_count = log_count + values(log_count)
      """.update.apply()
    }
  }
}
```

## System Failures

* Data receivers
* Transformation
* Storing output

## Stream Processing Semantics

* At most once
* At least once
* Exactly once

## Managing Kafka Offsets

* indempotent - map only
  * checkpoint
  * kafka commit
* transactional - map & aggregation
  * with shuffle -> collect to driver
  * no shuffle or repartitino -> foreachPartition

### a side note on transactional message delivery

## References

* http://blog.cloudera.com/blog/2015/03/exactly-once-spark-streaming-from-apache-kafka/
* http://spark.apache.org/docs/latest/streaming-programming-guide.html
* http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html
* http://kafka.apache.org/documentation.html#semantics

[1]: https://github.com/jizhang/spark-sandbox/blob/master/src/main/scala/ExactlyOnce.scala
