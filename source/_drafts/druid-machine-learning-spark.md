---
title: "Hive+Druid实现快速查询；回归分析是机器学习吗；StructuredStreaming可用于生产环境"
tags: [hive, druid, machine learning, stream processing, spark]
categories: [Digest]
---

## 结合Apache Hive和Druid实现高速OLAP查询

![使用HiveQL预汇总数据并保存至Druid](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/05/Part1Image2.png)

* Druid is fast because it combines the best qualities of a column store and inverted indexing.
  * different from Hive because of inverted indexing
  * more dimensions are added, fewer rows are needed
* Druid limitation
  * larce-scale join is not a priority
  * limited sql implementation
* integration
  * olap tools/query -> hive -> druid
  * Apache Calcite divides work according to the engine's strengths
    * simple query answered directly by druid
      * topN, time series, simple aggregations grouped by defined dimensions
    * complex operations push work down to druid

原文：https://dzone.com/articles/ultra-fast-olap-analytics-with-apache-hive-and-dru

<!-- more -->
