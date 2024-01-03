---
title: Flume 源码解析：HDFS Sink
tags:
  - flume
  - hdfs
  - java
categories: Big Data
date: 2018-10-04 13:49:03
---


Apache Flume 数据流程的最后一部分是 Sink，它会将上游抽取并转换好的数据输送到外部存储中去，如本地文件、HDFS、ElasticSearch 等。本文将通过分析源码来展现 HDFS Sink 的工作流程。

## Sink 组件的生命周期

[在上一篇文章中][1], 我们了解到 Flume 组件都会实现 `LifecycleAware` 接口，并由 `LifecycleSupervisor` 实例管理和监控。不过，Sink 组件并不直接由它管理，而且被包装在了 `SinkRunner` 和 `SinkProcessor` 这两个类中。Flume 支持三种 [Sink 处理器][2]，该处理器会将 Channel 和 Sink 以不同的方式连接起来。这里我们只讨论 `DefaultSinkProcessor` 的情况，即一个 Channel 只会连接一个 Sink。同时，我们也将略过对 Sink 分组的讨论。

![Sink Component LifeCycle](/images/flume/sink-component-lifecycle.png)

<!-- more -->

## HDFS Sink 模块中的类

HDFS Sink 模块的源码在 `flume-hdfs-sink` 子目录中，主要由以下几个类组成：

![HDFS Sink Classes](/images/flume/hdfs-sink-classes.png)

`HDFSEventSink` 类实现了生命周期的各个方法，包括 `configure`、`start`、`process`、`stop` 等。它启动后会维护一组 `BucketWriter` 实例，每个实例对应一个 HDFS 输出文件路径，上游的消息会传递给它，并写入 HDFS。通过不同的 `HDFSWriter` 实现，它可以将数据写入文本文件、压缩文件、或是 `SequenceFile`。

## 配置与启动

Flume 配置文件加载时，会实例化各个组件，并调用它们的 `configure` 方法，其中就包括 Sink 组件。在 `HDFSEventSink#configure` 方法中，程序会读取配置文件中以 `hdfs.` 为开头的项目，为其提供默认值，并做基本的参数校验。如，`batchSize` 必须大于零，`fileType` 指定为 `CompressedStream` 时 `codeC` 参数也必须指定等等。同时，程序还会初始化一个 `SinkCounter`，用于统计运行过程中的各项指标。

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

`HDFSEventSink#start` 方法中会创建两个线程池：`callTimeoutPool` 线程池会在 `BucketWriter#callWithTimeout` 方法中使用，用来限定 HDFS 远程调用的请求时间，如 [`FileSystem#create`][3] 或 [`FSDataOutputStream#hflush`][4] 都有可能超时；`timedRollerPool` 则用于对文件进行滚动，前提是用户配置了 `rollInterval` 选项，我们将在下一节详细说明。

```java
public void start() {
  callTimeoutPool = Executors.newFixedThreadPool(threadsPoolSize,
      new ThreadFactoryBuilder().setNameFormat(timeoutName).build());
  timedRollerPool = Executors.newScheduledThreadPool(rollTimerPoolSize,
      new ThreadFactoryBuilder().setNameFormat(rollerName).build());
}
```

## 处理数据

`process` 方法包含了 HDFS Sink 的主要逻辑，也就是从上游的 Channel 中获取数据，并写入指定的 HDFS 文件，流程图如下：

![Process Method Flow Chart](/images/flume/process-method-flow-chart.png)

### Channel 事务

处理逻辑的外层是一个 Channel 事务，并提供了异常处理。以 Kafka Channel 为例：事务开始时，程序会从 Kafka 中读取数据，但不会立刻提交变动后的偏移量。只有当这些消息被成功写入 HDFS 文件之后，偏移量才会提交给 Kafka，下次循环将从新的偏移量开始消费。

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

### 查找或创建 `BucketWriter`

`BucketWriter` 实例和 HDFS 文件一一对应，文件路径是通过配置生成的，例如：

```
a1.sinks.access_log.hdfs.path = /user/flume/access_log/dt=%Y%m%d
a1.sinks.access_log.hdfs.filePrefix = events.%[localhost]
a1.sinks.access_log.hdfs.inUsePrefix = .
a1.sinks.access_log.hdfs.inUseSuffix = .tmp
a1.sinks.access_log.hdfs.rollInterval = 300
a1.sinks.access_log.hdfs.fileType = CompressedStream
a1.sinks.access_log.hdfs.codeC = lzop
```

以上配置生成的临时文件和目标文件路径为：

```
/user/flume/access_log/dt=20180925/.events.hostname1.1537848761307.lzo.tmp
/user/flume/access_log/dt=20180925/events.hostname1.1537848761307.lzo
```

配置中的占位符会由 [`BucketPath#escapeString`][5] 方法替换，Flume 支持三类占位符：

* `%{...}`：使用消息中的头信息进行替换；
* `%[...]`：目前仅支持 `%[localhost]`、`%[ip]`、以及 `%[fqdn]`；
* `%x`：日期占位符，通过头信息中的 `timestamp` 来生成，或者使用 `useLocalTimeStamp` 配置项。

文件的前后缀则是在 `BucketWriter#open` 方法中追加的。代码中的 `counter` 是当前文件的创建时间戳，`lzo` 则是当前压缩格式的默认文件后缀。

```java
String fullFileName = fileName + "." + counter;
fullFileName += fileSuffix;
fullFileName += codeC.getDefaultExtension();
bucketPath = filePath + "/" + inUsePrefix + fullFileName + inUseSuffix;
targetPath = filePath + "/" + fullFileName;
```

如果指定路径没有对应的 `BucketWriter` 实例，程序会创建一个，并根据 `fileType` 配置项来生成对应的 `HDFSWriter` 实例。Flume 支持的三种类型是：`HDFSSequenceFile`、`HDFSDataStream`、以及 `HDFSCompressedDataStream`，写入 HDFS 的动作是由这些类中的代码完成的。

```java
bucketWriter = sfWriters.get(lookupPath);
if (bucketWriter == null) {
  hdfsWriter = writerFactory.getWriter(fileType);
  bucketWriter = new BucketWriter(hdfsWriter);
  sfWriters.put(lookupPath, bucketWriter);
}
```

### 写入数据并刷新

在写入数据之前，`BucketWriter` 首先会检查文件是否已经打开，如未打开则会命关联的 `HDFSWriter` 类开启新的文件，以 `HDFSCompressedDataStream` 为例：

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

Flume 默认的 `serializerType` 配置是 `TEXT`，即使用 [BodyTextEventSerializer][6] 来序列化数据，不做加工，直接写进输出流：

```java
public void write(Event e) throws IOException {
  out.write(e.getBody());
  if (appendNewline) {
    out.write('\n');
  }
}
```

当 `BucketWriter` 需要关闭或重开时会调用 `HDFSWriter#sync` 方法，进而执行序列化实例和输出流实例上的 `flush` 方法：

```java
public void sync() throws IOException {
  serializer.flush();
  compOut.finish();
  fsOut.flush();
  hflushOrSync(fsOut);
}
```

从 Hadoop 0.21.0 开始，[`Syncable#sync`][7] 拆分成了 `hflush` 和 `hsync` 两个方法，前者只是将数据从客户端的缓存中刷新出去，后者则会保证数据已被写入 HDFS 本地磁盘。为了兼容新老 API，Flume 会通过 Java 反射机制来确定 `hflush` 是否存在，不存在则调用 `sync` 方法。上述代码中的 `flushOrSync` 正是做了这样的判断。

### 文件滚动

HDFS Sink 支持三种滚动方式：按文件大小、按消息数量、以及按时间间隔。按大小和按数量的滚动是在 `BucketWriter#shouldRotate` 方法中判断的，每次 `append` 时都会调用：

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

按时间滚动则是使用了上文提到的 `timedRollerPool` 线程池，通过启动一个定时线程来实现：

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

## 关闭与停止

当 `HDFSEventSink#close` 被触发时，它会遍历所有的 `BucketWriter` 实例，调用它们的 `close` 方法，进而关闭下属的 `HDFSWriter`。这个过程和 `flush` 类似，只是还会做一些额外操作，如关闭后的 `BucketWriter` 会将自身从 `sfWriters` 哈希表中移除：

```java
public synchronized void close(boolean callCloseCallback) {
  writer.close();
  timedRollFuture.cancel(false);
  onCloseCallback.run(onCloseCallbackPath);
}
```

`onCloseCallback` 回调函数是在 `HDFSEventSink` 初始化 `BucketWriter` 时传入的：

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

最后，`HDFSEventSink` 会关闭 `callTimeoutPool` 和 `timedRollerPool` 线程池，整个组件随即停止。

```java
ExecutorService[] toShutdown = { callTimeoutPool, timedRollerPool };
for (ExecutorService execService : toShutdown) {
  execService.shutdown();
}
```

## 参考资料

* https://flume.apache.org/FlumeUserGuide.html#hdfs-sink
* https://github.com/apache/flume
* https://data-flair.training/blogs/flume-sink-processors/
* http://hadoop-hbase.blogspot.com/2012/05/hbase-hdfs-and-durable-sync.html


[1]: http://shzhangji.com/cnblogs/2017/10/24/flume-source-code-component-lifecycle/
[2]: https://flume.apache.org/FlumeUserGuide.html#flume-sink-processors
[3]: http://hadoop.apache.org/docs/r2.4.1/api/org/apache/hadoop/fs/FileSystem.html
[4]: https://hadoop.apache.org/docs/r2.4.1/api/org/apache/hadoop/fs/FSDataOutputStream.html
[5]: https://flume.apache.org/releases/content/1.4.0/apidocs/org/apache/flume/formatter/output/BucketPath.html
[6]: https://flume.apache.org/releases/content/1.4.0/apidocs/org/apache/flume/serialization/BodyTextEventSerializer.html
[7]: https://hadoop.apache.org/docs/r2.4.1/api/org/apache/hadoop/fs/Syncable.html
