---
title: Apache Spark 2.2 新特性；
tags:
  - spark
categories: Digest
---

## Apache Spark 2.2 有哪些新特性？

![Spark 2.2](/cnblogs/images/spark-2.2.png)

今年7月 Apache Spark 2.2.0 版本正式发布，该版本最大特点是将 Structured Streaming 标记为生产环境可用，其他亮点包括：进一步支持更多的 SQL 语法；新增 R 语言版的分布式机器学习算法；新增 MLlib 和 GraphX 算法。

Structured Streaming 还增加了如下特性：

* Kafka 输出源：以往将处理完的数据写入 Kafka 需要自己实现，现在有了官方支持；
* 新增状态 API：使用 `[flat]MapGroupsWithState` 来管理复杂的状态和超时时间；

SQL 部分新增的功能有：

* 对创建 Hive 表提供了更好的支持；
* 能够使用 `BROADCAST`、`BROADCASTJOIN`、`MAPJOIN` 等查询提示（Query Hint）；
* 基于成本的优化规则，针对多表 JOIN 提供更优的执行计划；

此外，Spark 2.2 移除了对 Hadoop 2.5 及 Java 7 以下版本的支持。PySpark 也做了较大更新，用户可以通过 `pip install` 安装最新版本。SparkR 则增加了 ALS、保序回归、多层感知器分类器、随机森林、LDA 等一系列机器学习算法，同时补充了 R 语言版的 Structured Streaming API。

原文：https://dzone.com/articles/whats-new-in-apache-spark-22
