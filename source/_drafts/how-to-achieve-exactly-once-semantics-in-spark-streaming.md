---
title: Spark Streaming 中如何实现 Exactly-Once 语义
tags: [spark, spark streaming, kafka, stream processing, scala]
categories: [Big Data]
---

Exactly-once 语义是实时计算的难点之一。要做到每一条记录只会被处理一次，即使服务器或网络发生故障时也能保证没有遗漏，这不仅需要实时计算框架本身的支持，还对上游的消息系统、下游的数据存储有所要求。此外，我们在编写计算流程时也需要遵循一定规范，才能真正实现 Exactly-once。本文将要讲述的是如何结合 Spark Streaming 框架、Kafka 消息系统、以及 MySQL 数据库来实现 Exactly-once 的实时计算流程。

![Spark Streaming](http://spark.apache.org/docs/latest/img/streaming-arch.png)

## 引例

首先让我们实现一个简单但完整的实时计算流程。我们从 Kafka 接收用户访问日志，解析并提取其中的时间和日志级别，并统计每分钟错误日志的数量，结果保存到 MySQL 中。

示例日志:

```text
2017-07-30 14:09:08 ERROR some message
2017-07-30 14:09:20 INFO  some message
2017-07-30 14:10:50 ERROR some message
```

结果表结构，其中 `log_time` 字段会截取到分钟级别：

```sql
create table error_log (
  log_time datetime primary key,
  log_count int not null default 0
);
```

<!-- more -->

Scala 项目通常使用 `sbt` 来管理。我们将下列依赖添加到 `build.sbt` 文件中。本例使用的是 Spark 2.2 和 Kafka 0.10，数据库操作类库使用了 ScalikeJDBC 3.0。

```scala
scalaVersion := "2.11.11"

libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-streaming" % "2.2.0" % "provided",
  "org.apache.spark" %% "spark-streaming-kafka-0-10" % "2.2.0",
  "org.scalikejdbc" %% "scalikejdbc" % "3.0.1",
  "mysql" % "mysql-connector-java" % "5.1.43"
)
```

完整的示例代码已上传至 GitHub（[链接][1]），下面我仅选取重要的部分加以说明：

```scala
// 初始化数据库连接
ConnectionPool.singleton("jdbc:mysql://localhost:3306/spark", "root", "")

// 创建 Spark Streaming 上下文
val conf = new SparkConf().setAppName("ExactlyOnce").setIfMissing("spark.master", "local[2]")
val ssc = new StreamingContext(conf, Seconds(5))

// 使用 Kafka Direct API 创建 DStream
val messages = KafkaUtils.createDirectStream[String, String](ssc,
   LocationStrategies.PreferConsistent,
   ConsumerStrategies.Subscribe[String, String](Seq("alog"), kafkaParams))

messages.foreachRDD { rdd =>
  // 日志处理
  val result = rdd.map(_.value)
    .flatMap(parseLog) // 日志解析函数
    .filter(_.level == "ERROR")
    .map(log => log.time.truncatedTo(ChronoUnit.MINUTES) -> 1)
    .reduceByKey(_ + _)
    .collect()

  // 结果保存至数据库
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

## 实时计算语义

实时计算有三种语义，分别是 At-most-once、At-least-once、以及 Exactly-once。一个典型的 Spark Streaming 应用程序会包含三个处理阶段：接收数据、处理汇总、保存结果。每个阶段都需要做不同的处理才能实现相应的语义。

对于 **接收数据**，主要取决于上游数据源的特性。例如，从 HDFS 这类支持容错的文件系统中读取文件，能够直接支持 Exactly-once 语义。如果上游消息系统支持 ACK（如RabbitMQ），我们就可以结合 Spark 的 Write Ahead Log 特性来实现 At-least-once 语义。对于非可靠的数据接收器（如 `socketTextStream`），当 Worker 或 Driver 节点发生故障时就会发生数据丢失，提供的语义也是未知的。而 Kafka 消息系统是基于偏移量（Offset）的，它的 Direct API 可以提供 Exactly-once 语义。

When **transforming data** with Spark's RDD, we automatically get exactly-once semantics, for RDD is itself immutable, fault-tolerant and deterministically re-computable. As long as the source data is available, and there's no side effects during transformation, the result will always be the same.

**Output operation** by default has at-least-once semantics. The `foreachRDD` function will execute more than once if there's worker failure, thus writing same data to external storage multiple times. There're two approaches to solve this issue, idempotent updates, and transactional updates. They are further discussed in the following sections.

## Exactly-once with Idempotent Writes

If multiple writes produce the same data, then this output operation is idempotent. `saveAsTextFile` is a typical idempotent update; messages with unique keys can be written to database without duplication. This approach will give us the equivalent exactly-once semantics. Note though it's usually for map-only procedures, and it requires some setup on Kafka DStream.

* Set `enable.auto.commit` to `false`. By default, Kafka DStream will commit the consumer offsets right after it receives the data. We want to postpone this action unitl the batch is fully processed.
* Turn on Spark Streaming's checkpointing to store Kafka offsets. But if the application code changes, checkpointed data is not reusable. This leads to a second option:
* Commit Kafka offsets after outputs. Kafka provides a `commitAsync` API, and the `HasOffsetRanges` class can be used to extract offsets from the initial RDD:

```scala
messages.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  rdd.foreachPartition { iter =>
    // output to database
  }
  messages.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
}
```

## Exactly-once with Transactional Writes

Transactional updates require a unique identifier. One can generate from batch time, partition id, or Kafka offsets, and then write the result along with the identifier into external storage within a single transaction. This atomic operation gives us exactly-once semantics, and can be applied to both map-only and aggregation procedures.

Usually writing to database should happen in `foreachPartition`, i.e. in worker nodes. It is true for map-only procedure, because Kafka RDD's partition is correspondent to Kafka partition, so we can extract each partition's offset like this:

```scala
messages.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  rdd.foreachPartition { iter =>
    val offsetRange = offsetRanges(TaskContext.get.partitionId)
  }
}
```

But for shuffled operations like the error log count example, we need to first collect the result back into driver and then perform the transaction.

```scala
messages.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  val result = processLogs(rdd).collect() // parse log and count error
  DB.localTx { implicit session =>
    result.foreach { case (time, count) =>
      // save to error_log table
    }
    offsetRanges.foreach { offsetRange =>
      val affectedRows = sql"""
      update kafka_offset set offset = ${offsetRange.untilOffset}
      where topic = ${topic} and `partition` = ${offsetRange.partition}
      and offset = ${offsetRange.fromOffset}
      """.update.apply()

      if (affectedRows != 1) {
        throw new Exception("fail to update offset")
      }
    }
  }
}
```

If the offsets fail to update, or there's a duplicate offset range detected by `offset != $fromOffset`, the whole transaction will rollback, which guarantees the exactly-once semantics.

## 总结

Exactly-once is a very strong semantics in stream processing, and will inevitably bring some overhead to your application and impact the throughput. It's also not applicable to [windowed][2] operations. So you need to decide whether it's necessary to spend such efforts, or weaker semantics even with few data loss will suffice. But surely knowing how to achieve exactly-once is a good chance of learning, and it's a great fun.

## 参考资料

* http://blog.cloudera.com/blog/2015/03/exactly-once-spark-streaming-from-apache-kafka/
* http://spark.apache.org/docs/latest/streaming-programming-guide.html
* http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html
* http://kafka.apache.org/documentation.html#semantics

[1]: https://github.com/jizhang/spark-sandbox/blob/master/src/main/scala/ExactlyOnce.scala
[2]: https://github.com/koeninger/kafka-exactly-once/blob/master/src/main/scala/example/Windowed.scala
