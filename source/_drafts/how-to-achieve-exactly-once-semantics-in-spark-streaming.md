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

// create Spark streaming context
val conf = new SparkConf().setAppName("ExactlyOnce").setIfMissing("spark.master", "local[2]")
val ssc = new StreamingContext(conf, Seconds(5))

// create Kafka DStream with Direct API
val messages = KafkaUtils.createDirectStream[String, String](ssc,
   LocationStrategies.PreferConsistent,
   ConsumerStrategies.Subscribe[String, String](Seq("alog"), kafkaParams))

messages.foreachRDD { rdd =>
  // do transformation
  val result = rdd.map(_.value)
    .flatMap(parseLog) // utility function to parse log line into case class
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

## Stream Processing Semantics

There're three semantics in stream processing, namely at-most-once, at-least-once, and exactly-once. In a typical Spark Streaming application, there're three processing phases: receive data, do transformation, and push outputs. Each phase takes different efforts to achieve different semantics.

For **receiving data**, it largely depends on the data source. For instance, reading files from a fault-tolerant file system like HDFS, gives us exactly-once semantics. For upstream queues that support acknowledgement, e.g. RabbitMQ, we can combine it with Spark's write ahead logs to achieve at-least-once semantics. For unreliable receivers like `socketTextStream`, there might be data loss due to worker/driver failure and gives us undefined semantics. Kafka, on the other hand, is offset based, and its direct API can give us exactly-once semantics.

When **transforming data** with Spark's RDD, we automatically get exactly-once semantics, for RDD is itself immutable, fault-tolerant and deterministically re-computable. As long as the source data is available, and there's no side effects during transformation, the result will always be the same.

**Output operation** by default has at-least-once semantics. The `foreachRDD` function will execute more than once if there's worker failure, thus writing same data to external storage multiple times. There're two approaches to solve this issue, idempotent updates, and transactional updates. `saveAsTextFile` is a typical idempotent update; messages with unique keys can be written to database without duplication. Transactional updates usually require a unique identifier, generated from batch time, partition id, or offsets, and then write the result along with the identifier into external storage within a single transaction. And this is the method we use to achieve end-to-end exactly-once semantics.

## Managing Kafka Offsets

* idempotent - map only
  * checkpoint
  * Kafka commit
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
