---
title: Apache Flume 如何解析消息中的事件时间
tags:
  - flume
  - etl
  - java
categories:
  - Big Data
date: 2017-08-06 09:09:06
---


数据开发工作中，从上游消息队列抽取数据是一项常规的 ETL 流程。在基于 Hadoop 构建的数据仓库体系中，我们通常会使用 Flume 将事件日志从 Kafka 抽取到 HDFS，然后针对其开发 MapReduce 脚本，或直接创建以时间分区的 Hive 外部表。这项流程中的关键一环是提取日志中的事件时间，因为实时数据通常会包含延迟，且在系统临时宕机的情况下，我们需要追回遗漏的数据，因而使用的时间戳必须是事件产生的时间。Flume 提供的诸多工具能帮助我们非常便捷地实现这一点。

![Apache Flume](/images/flume.png)

## HDFS Sink 和时间戳头信息

以下是一个基本的 HDFS Sink 配置：

```properties
a1.sinks = k1
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = /user/flume/ds_alog/dt=%Y%m%d
```

`%Y%m%d` 是该 Sink 支持的时间占位符，它会使用头信息中 `timestamp` 的值来替换这些占位符。HDFS Sink 还提供了 `hdfs.useLocalTimeStamp` 选项，直接使用当前系统时间来替换时间占位符，但这并不是我们想要达到的目的。

我们还可以使用 Hive Sink 直接将事件日志导入成 Hive 表，它能直接和 Hive 元数据库通信，自动创建表分区，并支持分隔符分隔和 JSON 两种序列化形式。当然，它同样需要一个 `timestamp` 头信息。不过，我们没有选择 Hive Sink，主要出于以下原因：

* 它不支持正则表达式，因此我们无法从类似访问日志这样的数据格式中提取字段列表；
* 它所提取的字段列表是根据 Hive 表信息产生的。假设上游数据源在 JSON 日志中加入了新的键值，直至我们主动更新 Hive 元信息，这些新增字段将被直接丢弃。对于数据仓库来说，完整保存原始数据是很有必要的。

<!-- more -->

## 正则表达式拦截器

Flume 提供了拦截器机制，我们可以在 Source 之后接上一系列操作，对数据进行基础的转换。例如，`TimestampInterceptor` 拦截器可以在消息头信息中增加当前时间。在本节中，我将演示如何借助拦截器来提取访问日志型和 JSON 型消息中的事件时间。

```text
0.123 [2017-06-27 09:08:00] GET /
0.234 [2017-06-27 09:08:01] GET /
```

[`RegexExtractorInterceptor`][1] 可以基于正则表达式来提取字符串，配置如下：

```properties
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = regex_extractor
a1.sources.r1.interceptors.i1.regex = \\[(.*?)\\]
a1.sources.r1.interceptors.i1.serializers = s1
a1.sources.r1.interceptors.i1.serializers.s1.type = org.apache.flume.interceptor.RegexExtractorInterceptorMillisSerializer
a1.sources.r1.interceptors.i1.serializers.s1.name = timestamp
a1.sources.r1.interceptors.i1.serializers.s1.pattern = yyyy-MM-dd HH:mm:ss
```

它会搜索字符串中满足 `\[(.*?)\]` 模式的子串，将第一个子模式即 `s1` 作为日期字符串进行解析，并将其转化成毫秒级时间戳，存入头信息 `timestamp`。

### 搜索与替换拦截器

对于 JSON 数据:

```json
{"actionTime":1498525680.023,"actionType":"pv"}
{"actionTime":1498525681.349,"actionType":"pv"}
```

我们同样可以用正则拦截器将 `actionTime` 提取出来，但要注意的是该字段的单位是秒，而 HDFS Sink 要求的是毫秒，这就需要我们在提取之间对其进行转换，如直接将 `.` 去掉。`SearchAndReplaceInterceptor` 可以做到这一点：

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

这里我们串联了两个拦截器。首先使用正则替换将 `1498525680.023` 转换成 `1498525680023`，再用正则提取出 `actionTime` 并存入头信息。

### 自定义拦截器

我们还可以编写自定义的拦截器，从而一次性完成提取、转换和更新操作。我们只需实现 `org.apache.flume.interceptor.Interceptor` 接口的 `intercept` 方法即可，源代码可以在 GitHub（[链接][3]）中找到，记得需要添加 `flume-ng-core` 依赖：

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

## 使用 Kafka Channel 直接导入数据

当上游消息系统是 Kafka，并且你能够完全控制消息的数据格式，那就可以省去 Source 一环，直接用 Kafka Channel 将数据导入至 HDFS。其中的关键在于要使用 `AvroFlumeEvent` 格式来存储消息，这样 [Kafka Channel][4] 才能从消息体中解析出 `timestamp` 头信息。如果消息内容是纯文本，那下游的 HDFS Sink 就会报时间戳找不到的错误了。

```java
// 构建 AvroFlumeEvent 消息，该类可以在 flume-ng-sdk 依赖中找到
Map<CharSequence, CharSequence> headers = new HashMap<>();
headers.put("timestamp", "1498525680023");
String body = "some message";
AvroFlumeEvent event = new AvroFlumeEvent(headers, ByteBuffer.wrap(body.getBytes()));

// 使用 Avro 编码器对消息进行序列化
ByteArrayOutputStream out = new ByteArrayOutputStream();
BinaryEncoder encoder = EncoderFactory.get().directBinaryEncoder(out, null);
SpecificDatumWriter<AvroFlumeEvent> writer = new SpecificDatumWriter<>(AvroFlumeEvent.class);
writer.write(event, encoder);
encoder.flush();

// 发送字节码至 Kafka
producer.send(new ProducerRecord<String, byte[]>("alog", out.toByteArray()));
```

## 参考资料

* http://flume.apache.org/FlumeUserGuide.html
* https://github.com/apache/flume

[1]: http://flume.apache.org/FlumeUserGuide.html#regex-extractor-interceptor
[2]: http://flume.apache.org/FlumeUserGuide.html#search-and-replace-interceptor
[3]: https://github.com/jizhang/blog-demo/blob/master/flume/src/main/java/com/shzhangji/demo/flume/ActionTimeInterceptor.java
[4]: http://flume.apache.org/FlumeUserGuide.html#kafka-channel
