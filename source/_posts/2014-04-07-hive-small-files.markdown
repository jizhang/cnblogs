---
layout: post
title: "Hive 小文件问题的处理"
date: 2014-04-07 17:09
comments: true
categories: [Big Data]
tags: [hive]
published: true
---

Hive的后端存储是HDFS，它对大文件的处理是非常高效的，如果合理配置文件系统的块大小，NameNode可以支持很大的数据量。但是在数据仓库中，越是上层的表其汇总程度就越高，数据量也就越小。而且这些表通常会按日期进行分区，随着时间的推移，HDFS的文件数目就会逐渐增加。

## 小文件带来的问题

关于这个问题的阐述可以读一读Cloudera的[这篇文章](http://blog.cloudera.com/blog/2009/02/the-small-files-problem/)。简单来说，HDFS的文件元信息，包括位置、大小、分块信息等，都是保存在NameNode的内存中的。每个对象大约占用150个字节，因此一千万个文件及分块就会占用约3G的内存空间，一旦接近这个量级，NameNode的性能就会开始下降了。

此外，HDFS读写小文件时也会更加耗时，因为每次都需要从NameNode获取元信息，并与对应的DataNode建立连接。对于MapReduce程序来说，小文件还会增加Mapper的个数，每个脚本只处理很少的数据，浪费了大量的调度时间。当然，这个问题可以通过使用CombinedInputFile和JVM重用来解决。

<!-- more -->

## Hive小文件产生的原因

前面已经提到，汇总后的数据量通常比源数据要少得多。而为了提升运算速度，我们会增加Reducer的数量，Hive本身也会做类似优化——Reducer数量等于源数据的量除以hive.exec.reducers.bytes.per.reducer所配置的量（默认1G）。Reducer数量的增加也即意味着结果文件的增加，从而产生小文件的问题。

## 配置Hive结果合并

我们可以通过一些配置项来使Hive在执行结束后对结果文件进行合并：

* `hive.merge.mapfiles` 在map-only job后合并文件，默认`true`
* `hive.merge.mapredfiles` 在map-reduce job后合并文件，默认`false`
* `hive.merge.size.per.task` 合并后每个文件的大小，默认`256000000`
* `hive.merge.smallfiles.avgsize` 平均文件大小，是决定是否执行合并操作的阈值，默认`16000000`

Hive在对结果文件进行合并时会执行一个额外的map-only脚本，mapper的数量是文件总大小除以size.per.task参数所得的值，触发合并的条件是：

1. 根据查询类型不同，相应的mapfiles/mapredfiles参数需要打开；
2. 结果文件的平均大小需要大于avgsize参数的值。

示例：

```sql
-- map-red job，5个reducer，产生5个60K的文件。
create table dw_stage.zj_small as
select paid, count(*)
from dw_db.dw_soj_imp_dtl
where log_dt = '2014-04-14'
group by paid;

-- 执行额外的map-only job，一个mapper，产生一个300K的文件。
set hive.merge.mapredfiles=true;
create table dw_stage.zj_small as
select paid, count(*)
from dw_db.dw_soj_imp_dtl
where log_dt = '2014-04-14'
group by paid;

-- map-only job，45个mapper，产生45个25M左右的文件。
create table dw_stage.zj_small as
select *
from dw_db.dw_soj_imp_dtl
where log_dt = '2014-04-14'
and paid like '%baidu%';

-- 执行额外的map-only job，4个mapper，产生4个250M左右的文件。
set hive.merge.smallfiles.avgsize=100000000;
create table dw_stage.zj_small as
select *
from dw_db.dw_soj_imp_dtl
where log_dt = '2014-04-14'
and paid like '%baidu%';
```

### 压缩文件的处理

如果结果表使用了压缩格式，则必须配合SequenceFile来存储，否则无法进行合并，以下是示例：

```sql
set mapred.output.compression.type=BLOCK;
set hive.exec.compress.output=true;
set mapred.output.compression.codec=org.apache.hadoop.io.compress.LzoCodec;
set hive.merge.smallfiles.avgsize=100000000;

drop table if exists dw_stage.zj_small;
create table dw_stage.zj_small
STORED AS SEQUENCEFILE
as select *
from dw_db.dw_soj_imp_dtl
where log_dt = '2014-04-14'
and paid like '%baidu%';
```

## 使用HAR归档文件

Hadoop的[归档文件](http://hadoop.apache.org/docs/stable1/hadoop_archives.html)格式也是解决小文件问题的方式之一。而且Hive提供了[原生支持](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Archiving)：

```
set hive.archive.enabled=true;
set hive.archive.har.parentdir.settable=true;
set har.partfile.size=1099511627776;

ALTER TABLE srcpart ARCHIVE PARTITION(ds='2008-04-08', hr='12');

ALTER TABLE srcpart UNARCHIVE PARTITION(ds='2008-04-08', hr='12');
```

如果使用的不是分区表，则可创建成外部表，并使用`har://`协议来指定路径。

## HDFS Federation

Hadoop V2引入了HDFS Federation的概念：

![](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/images/federation.gif)

实则是将NameNode做了拆分，从而增强了它的扩展性，小文件的问题也能够得到缓解。

## 其他工具

对于通常的应用，使用Hive结果合并就能达到很好的效果。如果不想因此增加运行时间，可以自行编写一些脚本，在系统空闲时对分区内的文件进行合并，也能达到目的。
