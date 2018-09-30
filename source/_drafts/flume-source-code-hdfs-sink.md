---
title: "Flume Source Code: HDFS Sink"
tags: [flume, hdfs, java]
categories: Big Data
---

Sink is the last component of Apache Flume data flow, and it is used to output data into storages like local files, HDFS, ElasticSearch, etc. In this article, I will illustrate how Flume's HDFS sink works, by analyzing its source code with diagrams.

## Sink Component Lifecycle

In the [previous article][1], we learnt that every Flume component implements `LifecycleAware` interface, and is started and monitored by `LifecycleSupervisor`. Sink component is not directly invoked by this supervisor, but wrapped in `SinkRunner` and `SinkProcessor` classes. Flume supports three different [sink processors][2], to connect channel and sinks in different semantics. But here we only consider the `DefaultSinkProcessor`, that accepts only one sink, and we will skip the concept of sink group as well.

a sequence diagram - ![Sink Component LifeCycle](/images/flume/sink-component-lifecycle.png)

something to illustrate?

<!-- more -->

## HDFS Sink Classes

HDFS sink's source code locates in `flume-hdfs-sink` sub-module, and is composed of the following classes:

class diagram - ![HDFS Sink Classes](/images/flume/hdfs-sink-classes.png)

`HDFSEventSink` class implements the lifecycle methods, including `configure`, `start`, `process`, and `stop`. It maintains a list of `BucketWriter`, according to the output file paths, and delegates received events to them. With different implementations of `HDFSWriter`, `BucketWriter` can append data to either text file, compressed file, or sequence file.

## Configure and Start

When Flume configuration file is loaded, `configure` method is called on every sink component. In `HDFSEventSink#configure`, it reads properties that are prefixed with `hdfs.` from the context, provides default values, and does some sanity checks. For instance, `batchSize` must be greater than 0, `codeC` must be provided when `fileType` is `CompressedStream`, etc. It also initializes a `SinkCounter` to provide various metrics for monitoring.

```java
public void configure(Context context) {
  filePath = Preconditions.checkNotNull(
      context.getString("hdfs.path"), "hdfs.path is required");
  rollInterval = context.getLong("hdfs.rollInterval", defaultRollInterval);

  if (sinkCounter == null) {
    sinkCounter = new SinkCounter(getName());
  }
}
```

`SinkProcessor` will invoke the `HDFSEventSink#start` method, in which two thread pools are created. `callTimeoutPool` is used by `BucketWriter#callWithTimeout` to limit the time that HDFS calls may take, such as [`FileSystem#create`][3], or [`FSDataOutputStream#hflush`][4]. `timedRollerPool` is used to schedule a periodic task to do time-based file rolling, if `rollInterval` property is provided. More details will be covered in the next section.

```java
public void start() {
  callTimeoutPool = Executors.newFixedThreadPool(threadsPoolSize,
      new ThreadFactoryBuilder().setNameFormat(timeoutName).build());
  timedRollerPool = Executors.newScheduledThreadPool(rollTimerPoolSize,
      new ThreadFactoryBuilder().setNameFormat(rollerName).build());
}
```

## Process Events

The `process` method contains the main logic, i.e. pull events from upstream channel and send them to HDFS. Here is the flow chart of this method.

![Process Method Flow Chart](/images/flume/process-method-flow-chart.png)

### Channel Transaction

Codes are wrapped in a channel transaction, with exception handling. Take Kafka channel for instance, when transaction begins, we take events without committing the offset. Only after we successfully write these events into HDFS, the consumed offset will be sent to Kafka. And in the next transaction, we can consume messages from the new offset.

```java
Channel channel = getChannel();
Transaction transaction = channel.getTransaction();
transaction.begin()
try {
  event = channel.take();
  bucketWriter.append(event);
  transaction.commit()
} catch (Throwable th) {
  transaction.rollback();
  throw new EventDeliveryException(th);
} finally {
  transaction.close();
}
```

### Find or Create BucketWriter

`BucketWriter` corresponds to an HDFS file, and the file path is generated from configuration. For example:

```
a1.sinks.access_log.hdfs.path = /user/flume/access_log/dt=%Y%m%d
a1.sinks.access_log.hdfs.filePrefix = events.%[localhost]
a1.sinks.access_log.hdfs.inUsePrefix = .
a1.sinks.access_log.hdfs.inUseSuffix = .tmp
a1.sinks.access_log.hdfs.rollInterval = 300
a1.sinks.access_log.hdfs.fileType = CompressedStream
a1.sinks.access_log.hdfs.codeC = lzop
```

The generated file paths, temporary and final, will be:

```
/user/flume/access_log/dt=20180925/.events.hostname1.1537848761307.lzo.tmp
/user/flume/access_log/dt=20180925/events.hostname1.1537848761307.lzo
```

Placeholders are replaced in [`BucketPath#escapeString`][5]. It supports three kinds of placeholders:

* `%{...}`: replace with arbitrary header values;
* `%[...]`: currently only supports `%[localhost]`, `%[ip]`, and `%[fqdn]`;
* `%x`: date time patterns, which requires a `timestamp` entry in headers, or `useLocalTimeStamp` is enabled.

And the prefix and suffix is added in `BucketWriter#open`. `counter` is the timestamp when this bucket is opened or re-opened, and `lzo` is the default extension of the configured compression codec.

```java
String fullFileName = fileName + "." + counter;
fullFileName += fileSuffix;
fullFileName += codeC.getDefaultExtension();
bucketPath = filePath + "/" + inUsePrefix + fullFileName + inUseSuffix;
targetPath = filePath + "/" + fullFileName;
```

If no `BucketWriter` is associated with the file path, a new one will be created. First, it creates an `HDFSWriter` corresponding to the `fileType` config. Flume supports three kinds of writers: `HDFSSequenceFile`, `HDFSDataStream`, and `HDFSCompressedDataStream`. They handle the actual writing to HDFS files, and will be assigned to a new `BucketWriter`.

```java
bucketWriter = sfWriters.get(lookupPath);
if (bucketWriter == null) {
  hdfsWriter = writerFactory.getWriter(fileType);
  bucketWriter = new BucketWriter(hdfsWriter);
  sfWriters.put(lookupPath, bucketWriter);
}
```

### Append Data and Flush

Before appending data, `BucketWriter` will first self-check whether it is opened. If not, it will call its underlying `HDFSWriter` to open a new file on HDFS filesystem. Take `HDFSCompressedDataStream` for instance:

```java
public void open(String filePath, CompressionCodec codec) {
  FileSystem hdfs = dstPath.getFileSystem(conf);
  fsOut = hdfs.append(dstPath)
  compressor = CodedPool.getCompressor(codec, conf);
  cmpOut = codec.createOutputStream(fsOut, compressor);
  serializer = EventSerializerFactory.getInstance(serializerType, cmpOut);
}

public void append(Event e) throws IO Exception {
  serializer.write(event);
}
```

Flume's default `serializerType` is `TEXT`, i.e. [BodyTextEventSerializer][6] that simply writes the event content to the output stream.

```java
public void write(Event e) throws IOException {
  out.write(e.getBody());
  if (appendNewline) {
    out.write('\n');
  }
}
```

When `BucketWriter` is about to close or re-open, it calls `sync` on `HDFSWrtier`, which in turn calls `flush` on serializer and underlying output stream.

```java
public void sync() throws IOException {
  serializer.flush();
  compOut.finish();
  fsOut.flush();
  hflushOrSync(fsOut);
}
```

From Hadoop 0.21.0, the [`Syncable#sync`] method is divided into `hflush` and `hsync` methods. Former just flushes data out of client's buffer, while latter guarantees data is synced to disk device. In order to handle both old and new API, Flume will use Java reflection to determine whether `hflush` exists, or fall back to `sync`. The `flushOrSync` method will invoke the right method.

### File Rotation

In HDFS sink, files can be rotated by file size, event count, or time interval. `BucketWriter#shouldRotate` is called in every `append`:

```java
private boolean shouldRotate() {
  boolean doRotate = false;
  if ((rollCount > 0) && (rollCount <= eventCounter)) {
    doRotate = true;
  }
  if ((rollSize > 0) && (rollSize <= processSize)) {
    doRotate = true;
  }
  return doRotate;
}
```

Time-based rolling, on the other hand, is scheduled in the previously mentioned `timedRollerPool`:

```java
private void open() throws IOException, InterruptedException {
  if (rollInterval > 0) {
    Callable<Void> action = new Callable<Void>() {
      public Void call() throws Exception {
        close(true);
      }
    };
    timedRollFuture = timedRollerPool.schedule(action, rollInterval);
  }
}
```

## Close and Stop

In `HDFSEventSink#close`, it iterates every `BucketWriter` and calls its `close` method, which in turns calls its underlying `HDFSWriter`'s `close` method. What it does is mostly like `flush` method, but also closes the output stream and invokes some callback functions, like removing current `BucketWriter` from the `sfWriters` hash map.

```java
public synchronized void close(boolean callCloseCallback) {
  writer.close();
  timedRollFuture.cancel(false);
  onCloseCallback.run(onCloseCallbackPath);
}
```

The `onCloseCallback` is passed from `HDFSEventSink` when initializing the `BucketWriter`:

```java
WriterCallback closeCallback = new WriterCallback() {
  public void run(String bucketPath) {
      synchronized (sfWritersLock) {
        sfWriters.remove(bucketPath);
      }
  }
}
bucketWriter = new BucketWriter(lookPath, closeCallback);
```

After all `BucketWriter`s are closed, `HDFSEventSink` then shutdown the `callTimeoutPool` and `timedRollerPool` executer services.

## References

* https://flume.apache.org/FlumeUserGuide.html#hdfs-sink
* https://github.com/apache/flume
* https://data-flair.training/blogs/flume-sink-processors/
* http://hadoop-hbase.blogspot.com/2012/05/hbase-hdfs-and-durable-sync.html


[1]: http://shzhangji.com/blog/2017/10/23/flume-source-code-component-lifecycle/
[2]: https://flume.apache.org/FlumeUserGuide.html#flume-sink-processors
[3]: http://hadoop.apache.org/docs/r2.4.1/api/org/apache/hadoop/fs/FileSystem.html
[4]: https://hadoop.apache.org/docs/r2.4.1/api/org/apache/hadoop/fs/FSDataOutputStream.html
[5]: https://flume.apache.org/releases/content/1.4.0/apidocs/org/apache/flume/formatter/output/BucketPath.html
[6]: https://flume.apache.org/releases/content/1.4.0/apidocs/org/apache/flume/serialization/BodyTextEventSerializer.html
