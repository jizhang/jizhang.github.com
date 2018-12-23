---
title: Real-time Exactly-once ETL with Apache Flink
tags: [flink, kafka, hdfs, java, etl]
categories: Big Data
---

Apache Flink is another popular big data processing framework, which differs from Apache Spark in that Flink uses stream processing to mimic batch processing and provides sub-second latency along with exactly-once semantics. One of its use cases is to build a real-time data pipeline, move and transform data between different stores. This article will show you how to build such an application, and explain how Flink guarantees its correctness.

![Apache Flink](/images/flink/arch.png)

## Demo ETL Application

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

### Kafka Consumer Source

Flink provides [native support][1] for consuming messages from Kafka. Choose the right version and add to dependencies:

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.10_${scala.binary.version}</artifactId>
  <version>${flink.version}</version>
</dependency>
```

We need some Kafka server to bootstrap this source. For testing, one can follow the Kafka [official document][2] to setup a local broker. Create the source and pass the host and topic name.

```java
Properties props = new Properties();
props.setProperty("bootstrap.servers", "localhost:9092");
FlinkKafkaConsumer010<String> consumer = new FlinkKafkaConsumer010<>(
    "flink_test", new SimpleStringSchema(), props);
DataStream<String> stream = env.addSource(consumer);
```

Flink will read data from a local Kafka broker, with topic `flink_test`, and transform it into simple strings, indicated by `SimpleStringSchema`. There are other built-in deserialization schema like JSON and Avro, or you can create a custom one.

### Streaming File Sink

`StreamingFileSink` is replacing the previous `BucketingSink` to store data into HDFS in different directories. The key concept here is bucket assigner, which defaults to `DateTimeBucketAssigner`, that divides messages into timed buckets according to processing time, i.e. the time when messages arrive the operator. But what we want is to divide messages by event time, so we have to write one on our own.

```java
public class EventTimeBucketAssigner implements BucketAssigner<String, String> {
  @Override
  public String getBucketId(String element, Context context) {
    JsonNode node = mapper.readTree(element);
    long date = (long) (node.path("timestamp").floatValue() * 1000);
    String partitionValue = new SimpleDateFormat("yyyyMMdd").format(new Date(date));
    return "dt=" + partitionValue;
  }
}
```

We create a bucket assigner that ingests a string, decodes with Jackson, extracts the timestamp, and returns the bucket name of this message. Then the sink will know where to put them. Full code can be found on GitHub ([link][3]).

```java
StreamingFileSink<String> sink = StreamingFileSink
    .forRowFormat(new Path("/tmp/kafka-loader"), new SimpleStringEncoder<String>())
    .withBucketAssigner(new EventTimeBucketAssigner())
    .build();
stream.addSink(sink);
```

There is also a `forBulkFormat`, if you prefer storing data in a more compact way like Parquet.

A note on `StreamingFileSink` though, it only works with Hadoop 2.7 and above, because it requires the file system supporting `truncate`, which helps recovering the writing process from the last checkpoint.

### Enable Checkpointing

So far, the application can be put into work by invoking `env.execute()`, but it only guarantees at-least-once semantics. To achieve exactly-once, we simply turn on Flink's checkpointing:

```java
env.enableCheckpointing(60_000);
env.setStateBackend((StateBackend) new FsStateBackend("/tmp/flink/checkpoints"));
env.getCheckpointConfig().enableExternalizedCheckpoints(
    ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION);
```

Checkpoint is Flink's solution to fault tolerance, which we will cover later. Here we switch the state backend from default `MemoryStateBackend` to `FsStateBackend`, that stores state into filesystem like HDFS, instead of in memory, to help surviving job manager failure. Flink also recommends using `RocksDBStateBackend`, when job state is very large and requires incremental checkpointing.

### Submit and Manage Jobs

Flink application can be directly run in IDE, or you can setup a local [standalone cluster][4] and submit jobs with Flink CLI:

```
bin/flink run -c com.shzhangji.flinksandbox.kafka.KafkaLoader target/flink-sandbox-0.1.0.jar
```

We can check out the job information in Flink dashboard:

![Flink Dashboard](/images/flink/dashboard.png)

#### Cancel and Resume Job with Savepoint

To cancel or restart the job, say we want to upgrade the code logic, we need to create a savepoint. A savepoint is like a checkpoint, storing state of the running tasks. But savepoint is usually manually created, for planned backup or upgrade, while checkpoint is managed by Flink to provide fault tolerance. The `cancel` sub-command accepts `-s` option to write savepoint into some directory.

```
$ bin/flink cancel -s /tmp/flink/savepoints 1253cc85e5c702dbe963dd7d8d279038
Cancelled job 1253cc85e5c702dbe963dd7d8d279038. Savepoint stored in file:/tmp/flink/savepoints/savepoint-1253cc-0df030f4f2ee.
```

For our ETL application, savepoint will include current Kafka offsets, in-progress output file names, etc. To resume from a savepoint, pass `-s` to `run` sub-command. The application will start from the savepoint, e.g. consume messages right after the saved offsets, without losing or duplicating data.

```
flink run -s /tmp/flink/savepoints/savepoint-1253cc-0df030f4f2ee -c com.shzhangji.flinksandbox.kafka.KafkaLoader target/flink-sandbox-0.1.0.jar
```

#### YARN Support

Running Flink jobs on YARN also uses `flink run`. Replace the file paths with HDFS prefix, re-package and run the following command:

```
$ export HADOOP_CONF_DIR=/path/to/hadoop/conf
$ bin/flink run -m yarn-cluster -c com.shzhangji.flinksandbox.kafka.KafkaLoader target/flink-sandbox-0.1.0.jar
Submitted application application_1545534487726_0001
```

Flink dashboard will run in YARN application master. The returned application ID can be used to manage the jobs through Flink CLI:

```
bin/flink cancel -s hdfs://localhost:9000/tmp/flink/savepoints -yid application_1545534487726_0001 84de00a5e193f26c937f72a9dc97f386
```

## Exactly-once Guarantees

* how flink ensures exactly-once
    * kafka
    * checkpoint
    * sink
* misc
    * parallelism


## References

*

[1]: https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/connectors/kafka.html
[2]: https://kafka.apache.org/quickstart
[3]: https://github.com/jizhang/flink-sandbox/blob/master/src/main/java/com/shzhangji/flinksandbox/kafka/EventTimeBucketAssigner.java
[4]: https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/deployment/cluster_setup.html
