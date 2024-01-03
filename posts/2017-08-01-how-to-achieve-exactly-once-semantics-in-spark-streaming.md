---
title: Spark Streaming 中如何实现 Exactly-Once 语义
tags:
  - spark
  - spark streaming
  - kafka
  - stream processing
  - scala
categories:
  - Big Data
date: 2017-08-01 12:54:47
---


Exactly-once 语义是实时计算的难点之一。要做到每一条记录只会被处理一次，即使服务器或网络发生故障时也能保证没有遗漏，这不仅需要实时计算框架本身的支持，还对上游的消息系统、下游的数据存储有所要求。此外，我们在编写计算流程时也需要遵循一定规范，才能真正实现 Exactly-once。本文将讲述如何结合 Spark Streaming 框架、Kafka 消息系统、以及 MySQL 数据库来实现 Exactly-once 的实时计算流程。

![Spark Streaming](http://spark.apache.org/docs/latest/img/streaming-arch.png)

## 引例

首先让我们实现一个简单而完整的实时计算流程。我们从 Kafka 接收用户访问日志，解析并提取其中的时间和日志级别，并统计每分钟错误日志的数量，结果保存到 MySQL 中。

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

实时计算有三种语义，分别是 At-most-once、At-least-once、以及 Exactly-once。一个典型的 Spark Streaming 应用程序会包含三个处理阶段：接收数据、处理汇总、输出结果。每个阶段都需要做不同的处理才能实现相应的语义。

对于 **接收数据**，主要取决于上游数据源的特性。例如，从 HDFS 这类支持容错的文件系统中读取文件，能够直接支持 Exactly-once 语义。如果上游消息系统支持 ACK（如RabbitMQ），我们就可以结合 Spark 的 Write Ahead Log 特性来实现 At-least-once 语义。对于非可靠的数据接收器（如 `socketTextStream`），当 Worker 或 Driver 节点发生故障时就会产生数据丢失，提供的语义也是未知的。而 Kafka 消息系统是基于偏移量（Offset）的，它的 Direct API 可以提供 Exactly-once 语义。

在使用 Spark RDD 对数据进行 **转换或汇总** 时，我们可以天然获得 Exactly-once 语义，因为 RDD 本身就是一种具备容错性、不变性、以及计算确定性的数据结构。只要数据来源是可用的，且处理过程中没有副作用（Side effect），我们就能一直得到相同的计算结果。

**结果输出** 默认符合 At-least-once 语义，因为 `foreachRDD` 方法可能会因为 Worker 节点失效而执行多次，从而重复写入外部存储。我们有两种方式解决这一问题，幂等更新和事务更新。下面我们将深入探讨这两种方式。

## 使用幂等写入实现 Exactly-once

如果多次写入会产生相同的结果数据，我们可以认为这类写入操作是幂等的。`saveAsTextFile` 就是一种典型的幂等写入。如果消息中包含唯一主键，那么多次写入相同的数据也不会在数据库中产生重复记录。这种方式也就能等价于 Exactly-once 语义了。但需要注意的是，幂等写入只适用于 Map-only 型的计算流程，即没有 Shuffle、Reduce、Repartition 等操作。此外，我们还需对 Kafka DStream 做一些额外设置：

* 将 `enable.auto.commit` 设置为 `false`。默认情况下，Kafka DStream 会在接收到数据后立刻更新自己的偏移量，我们需要将这个动作推迟到计算完成之后。
* 打开 Spark Streaming 的 Checkpoint 特性，用于存放 Kafka 偏移量。但若应用程序代码发生变化，Checkpoint 数据也将无法使用，这就需要改用下面的操作：
* 在数据输出之后手动提交 Kafka 偏移量。`HasOffsetRanges` 类，以及 `commitAsync` API 可以做到这一点：

```scala
messages.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  rdd.foreachPartition { iter =>
    // output to database
  }
  messages.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
}
```

## 使用事务写入实现 Exactly-once

在使用事务型写入时，我们需要生成一个唯一 ID，这个 ID 可以使用当前批次的时间、分区号、或是 Kafka 偏移量来生成。之后，我们需要在一个事务中将处理结果和这个唯一 ID 一同写入数据库。这一原子性的操作将带给我们 Exactly-once 语义，而且该方法可以同时适用于 Map-only 以及包含汇聚操作的计算流程。

我们通常会在 `foreachPartition` 方法中来执行数据库写入操作。对于 Map-only 流程来说是适用的，因为这种流程下 Kafka 分区和 RDD 分区是一一对应的，我们可以用以下方式获取各分区的偏移量：

```scala
messages.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  rdd.foreachPartition { iter =>
    val offsetRange = offsetRanges(TaskContext.get.partitionId)
  }
}
```

但对于包含 Shuffle 的计算流程（如上文的错误日志统计），我们需要先将处理结果拉取到 Driver 进程中，然后才能执行事务操作：

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

如果偏移量写入失败，或者重复处理了某一部分数据（`offset != $fromOffset` 判断条件不通过），该事务就会回滚，从而做到 Exactly-once。

## 总结

实时计算中的 Exactly-once 是比较强的一种语义，因而会给你的应用程序引入额外的开销。此外，它尚不能很好地支持[窗口型][2]操作。因此，是否要在代码中使用这一语义就需要开发者自行判断了。很多情况下，数据丢失或重复处理并不那么重要。不过，了解 Exactly-once 的开发流程还是有必要的，对学习 Spark Streaming 也会有所助益。

## 参考资料

* http://blog.cloudera.com/blog/2015/03/exactly-once-spark-streaming-from-apache-kafka/
* http://spark.apache.org/docs/latest/streaming-programming-guide.html
* http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html
* http://kafka.apache.org/documentation.html#semantics

[1]: https://github.com/jizhang/spark-sandbox/blob/master/src/main/scala/ExactlyOnce.scala
[2]: https://github.com/koeninger/kafka-exactly-once/blob/master/src/main/scala/example/Windowed.scala
