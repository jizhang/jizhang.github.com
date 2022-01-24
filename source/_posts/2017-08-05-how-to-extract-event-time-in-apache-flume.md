---
title: How to Extract Event Time in Apache Flume
tags:
  - flume
  - etl
  - java
categories:
  - Big Data
date: 2017-08-05 15:10:47
---


Extracting data from upstream message queues is a common task in ETL. In a Hadoop based data warehouse, we usually use Flume to import event logs from Kafka into HDFS, and then run MapReduce jobs agaist it, or create Hive external tables partitioned by time. One of the keys of this process is to extract the event time from the logs, since real-time data can have time lags, or your system is temporarily offline and need to perform a catch-up. Flume provides various facilities to help us do this job easily.

![Apache Flume](/images/flume.png)

## HDFS Sink and Timestamp Header

Here is a simple HDFS Sink config:

```properties
a1.sinks = k1
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = /user/flume/ds_alog/dt=%Y%m%d
```

`%Y%m%d` is the placeholders supported by this sink. It will use the milliseconds in `timestamp` header to replace them. Also, HDFS Sink provides `hdfs.useLocalTimeStamp` option so that it'll use the local time to replace these placeholders, but this is not what we intend.

Another sink we could use is the Hive Sink, which directly communicates with Hive metastore and loads data into HDFS as Hive table. It supports both delimited text and JSON serializers, and also requires a `timestamp` header. But we don't choose it for the following reasons:

* It doesn't support regular expression serializer, so we cannot extract columns from arbitrary data format like access logs;
* The columns to be extracted are defined in Hive metastore. Say the upstream events add some new keys in JSON, they will be dropped until Hive table definition is updated. As in data warehouse, it's better to preserve the original source data for a period of time.

<!-- more -->

## Regex Extractor Interceptor

Flume has a mechanism called Interceptor, i.e. some optionally chained operations appended to Source, so as to perform various yet primitive transformation. For instance, the `TimestampInterceptor` is to add current local timestamp to the event header. In this section, I'll demonstrate how to extract event time from access logs and JSON serialized logs with the help of interceptors.

```text
0.123 [2017-06-27 09:08:00] GET /
0.234 [2017-06-27 09:08:01] GET /
```

[`RegexExtractorInterceptor`][1] can be used to extract values based on regular expressions. Here's the config:

```properties
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = regex_extractor
a1.sources.r1.interceptors.i1.regex = \\[(.*?)\\]
a1.sources.r1.interceptors.i1.serializers = s1
a1.sources.r1.interceptors.i1.serializers.s1.type = org.apache.flume.interceptor.RegexExtractorInterceptorMillisSerializer
a1.sources.r1.interceptors.i1.serializers.s1.name = timestamp
a1.sources.r1.interceptors.i1.serializers.s1.pattern = yyyy-MM-dd HH:mm:ss
```

It searches the string with pattern `\[(.*?)\]`, capture the first sub-pattern as `s1`, then parse it as a datetime string, and finally store it into headers with the name `timestamp`.

### Search And Replace Interceptor

For JSON strings:

```json
{"actionTime":1498525680.023,"actionType":"pv"}
{"actionTime":1498525681.349,"actionType":"pv"}
```

We can also extract `actionTime` with a regular expression, but note that HDFS Sink requires the timestamp in milliseconds, so we have to first convert the timestamp with `SearchAndReplaceInterceptor`.

```properties
a1.sources.r1.interceptors = i1 i2
a1.sources.r1.interceptors.i1.type = search_replace
a1.sources.r1.interceptors.i1.searchPattern = \"actionTime\":(\\d+)\\.(\\d+)
a1.sources.r1.interceptors.i1.replaceString = \"actionTime\":$1$2
a1.sources.r1.interceptors.i2.type = regex_extractor
a1.sources.r1.interceptors.i2.regex = \"actionTime\":(\\d+)
a1.sources.r1.interceptors.i2.serializers = s1
a1.sources.r1.interceptors.i2.serializers.s1.name = timestamp
```

There're two chained interceptors, first one replaces `1498525680.023` with `1498525680023` and second extracts `actionTime` right into headers.

### Custom Interceptor

It's also possible to write your own interceptor, thus do the extraction and conversion in one step. Your interceptor should implements `org.apache.flume.interceptor.Interceptor` and then do the job in `intercept` method. The source code and unit test can be found on GitHub ([link][3]). Please add `flume-ng-core` to your project dependencies.

```java
public class ActionTimeInterceptor implements Interceptor {
    private final static ObjectMapper mapper = new ObjectMapper();
    @Override
    public Event intercept(Event event) {
        try {
            JsonNode node = mapper.readTree(new ByteArrayInputStream(event.getBody()));
            long timestamp = (long) (node.get("actionTime").getDoubleValue() * 1000);
            event.getHeaders().put("timestamp", Long.toString(timestamp));
        } catch (Exception e) {
            // no-op
        }
        return event;
    }
}
```

## Use Kafka Channel Directly

When the upstream is Kafka, and you have control of the message format, you can further eliminate the Source and directly pass data from Kafka to HDFS. The trick is to write messages in `AvroFlumeEvent` format, so that [Kafka Channel][4] can deserialize them and use the `timestamp` header within. Otherwise, Kafka channel will parse messages as plain text with no headers, and HDFS sink will complain missing `timestamp`.

```java
// construct an AvroFlumeEvent, this class can be found in flume-ng-sdk artifact
Map<CharSequence, CharSequence> headers = new HashMap<>();
headers.put("timestamp", "1498525680023");
String body = "some message";
AvroFlumeEvent event = new AvroFlumeEvent(headers, ByteBuffer.wrap(body.getBytes()));

// serialize event with Avro encoder
ByteArrayOutputStream out = new ByteArrayOutputStream();
BinaryEncoder encoder = EncoderFactory.get().directBinaryEncoder(out, null);
SpecificDatumWriter<AvroFlumeEvent> writer = new SpecificDatumWriter<>(AvroFlumeEvent.class);
writer.write(event, encoder);
encoder.flush();

// send bytes to Kafka
producer.send(new ProducerRecord<String, byte[]>("alog", out.toByteArray()));
```

## References

* http://flume.apache.org/FlumeUserGuide.html
* https://github.com/apache/flume

[1]: http://flume.apache.org/FlumeUserGuide.html#regex-extractor-interceptor
[2]: http://flume.apache.org/FlumeUserGuide.html#search-and-replace-interceptor
[3]: https://github.com/jizhang/java-blog-demo/blob/blog-flume/flume/src/main/java/com/shzhangji/demo/flume/ActionTimeInterceptor.java
[4]: http://flume.apache.org/FlumeUserGuide.html#kafka-channel
