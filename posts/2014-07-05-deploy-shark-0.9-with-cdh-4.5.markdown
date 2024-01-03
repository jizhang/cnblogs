---
layout: post
title: "在 CDH 4.5 上安装 Shark 0.9"
date: 2014-07-05 17:16
comments: true
categories: [Big Data]
tags: [spark, cdh]
---

[Spark](http://spark.apache.org)是一个新兴的大数据计算平台，它的优势之一是内存型计算，因此对于需要多次迭代的算法尤为适用。同时，它又能够很好地融合到现有的[Hadoop](http://hadoop.apache.org)生态环境中，包括直接存取HDFS上的文件，以及运行于YARN之上。对于[Hive](http://hive.apache.org)，Spark也有相应的替代项目——[Shark](http://shark.cs.berkeley.edu/)，能做到 **drop-in replacement** ，直接构建在现有集群之上。本文就将简要阐述如何在CDH4.5上搭建Shark0.9集群。

## 准备工作

* 安装方式：Spark使用CDH提供的Parcel，以Standalone模式启动
* 软件版本
    * Cloudera Manager 4.8.2
    * CDH 4.5
    * Spark 0.9.0 Parcel
    * [Shark 0.9.1 Binary](http://cloudera.rst.im/shark/)
* 服务器基础配置
    * 可用的软件源（如[中科大的源](http://mirrors.ustc.edu.cn/)）
    * 配置主节点至子节点的root账户SSH无密码登录。
    * 在`/etc/hosts`中写死IP和主机名，或者DNS做好正反解析。

<!-- more -->

## 安装Spark

* 使用CM安装Parcel，不需要重启服务。
* 修改`/etc/spark/conf/spark-env.sh`：（其中one-843是主节点的域名）

```bash
STANDALONE_SPARK_MASTER_HOST=one-843
DEFAULT_HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop
export SPARK_CLASSPATH=$SPARK_CLASSPATH:/opt/cloudera/parcels/HADOOP_LZO/lib/hadoop/lib/*
export SPARK_LIBRARY_PATH=$SPARK_LIBRARY_PATH:/opt/cloudera/parcels/HADOOP_LZO/lib/hadoop/lib/native
```

* 修改`/etc/spark/conf/slaves`，添加各节点主机名。
* 将`/etc/spark/conf`目录同步至所有节点。
* 启动Spark服务（即Standalone模式）：

```bash
$ /opt/cloudera/parcels/SPARK/lib/spark/sbin/start-master.sh
$ /opt/cloudera/parcels/SPARK/lib/spark/sbin/start-slaves.sh
```

* 测试`spark-shell`是否可用：

```scala
sc.textFile("hdfs://one-843:8020/user/jizhang/zj_people.txt.lzo").count
```

## 安装Shark

* 安装Oracle JDK 1.7 Update 45至`/usr/lib/jvm/jdk1.7.0_45`。
* 下载别人编译好的二进制包：[shark-0.9.1-bin-2.0.0-mr1-cdh-4.6.0.tar.gz][1]
* 解压至`/opt`目录，修改`conf/shark-env.sh`：

```bash
export JAVA_HOME=/usr/lib/jvm/jdk1.7.0_45
export SCALA_HOME=/opt/cloudera/parcels/SPARK/lib/spark
export SHARK_HOME=/root/shark-0.9.1-bin-2.0.0-mr1-cdh-4.6.0

export HIVE_CONF_DIR=/etc/hive/conf

export HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop
export SPARK_HOME=/opt/cloudera/parcels/SPARK/lib/spark
export MASTER=spark://one-843:7077

export SPARK_CLASSPATH=$SPARK_CLASSPATH:/opt/cloudera/parcels/HADOOP_LZO/lib/hadoop/lib/*
export SPARK_LIBRARY_PATH=$SPARK_LIBRARY_PATH:/opt/cloudera/parcels/HADOOP_LZO/lib/hadoop/lib/native
```

* 开启SharkServer2，使用Supervisord管理：

```
[program:sharkserver2]
command = /opt/shark/bin/shark --service sharkserver2
autostart = true
autorestart = true
stdout_logfile = /var/log/sharkserver2.log
redirect_stderr = true
```

```bash
$ supervisorctl start sharkserver2
```

* 测试

```bash
$ /opt/shark/bin/beeline -u jdbc:hive2://one-843:10000 -n root
```

## 版本问题

### 背景

#### CDH

CDH是对Hadoop生态链各组件的打包，每个CDH版本都会对应一组Hadoop组件的版本，如[CDH4.5][2]的部分对应关系如下：

* Apache Hadoop: hadoop-2.0.0+1518
* Apache Hive: hive-0.10.0+214
* Hue: hue-2.5.0+182

可以看到，CDH4.5对应的Hive版本是0.10.0，因此它的[Metastore Server][3]使用的也是0.10.0版本的API。

#### Spark

Spark目前最高版本是0.9.1，CDH前不久推出了0.9.0的Parcel，使得安装过程变的简单得多。CDH5中对Spark做了深度集成，即可以用CM来直接控制Spark的服务，且支持Spark on YARN架构。

#### Shark

Shark是基于Spark的一款应用，可以简单地认为是将Hive的MapReduce引擎替换为了Spark。

Shark的一个特点的是需要使用特定的Hive版本——[AMPLab patched Hive][4]：

* Shark 0.8.x: AMPLab Hive 0.9.0
* Shark 0.9.x: AMPLab Hive 0.11.0

在0.9.0以前，我们需要手动下载AMPLab Hive的二进制包，并在Shark的环境变量中设置$HIVE_HOME。在0.9.1以后，AMPLab将该版本的Hive包上传至了Maven，可以直接打进Shark的二进制包中。但是，这个Jar是用JDK7编译的，因此运行Shark需要使用Oracle JDK7。CDH建议使用Update 45这个小版本。

#### Shark与Hive的并存

Shark的一个卖点是和Hive的[高度兼容](5)，也就是说它可以直接操作Hive的metastore db，或是和metastore server通信。当然，前提是两者的Hive版本需要一致，这也是目前遇到的最大问题。

### 目前发现的不兼容SQL

* DROP TABLE ...

```
FAILED: Error in metadata: org.apache.thrift.TApplicationException: Invalid method name: 'drop_table_with_environment_context'
```

* INSERT OVERWRITE TABLE ... PARTITION (...) SELECT ...
* LOAD DATA INPATH '...' OVERWRITE INTO TABLE ... PARTITION (...)

```
Failed with exception org.apache.thrift.TApplicationException: Invalid method name: 'partition_name_has_valid_characters'
```

也就是说上述两个方法名是0.11.0接口中定义的，在0.10.0的定义中并不存在，所以出现上述问题。

### 解决方案

#### 对存在问题的SQL使用Hive命令去调

因为Shark初期是想给分析师使用的，他们对分区表并不是很在意，而DROP TABLE可以在客户端做判断，转而使用Hive来执行。

这个方案的优点是可以在现有集群上立刻用起来，但缺点是需要做一些额外的开发，而且API不一致的问题可能还会有其他坑在里面。

#### 升级到CDH5

CDH5中Hive的版本是0.12.0，所以不排除同样存在API不兼容的问题。不过网上也有人尝试跳过AMPLab Hive，让Shark直接调用CDH中的Hive，其可行性还需要我们自己测试。

对于这个问题，我只在[Google Groups][6]上看到一篇相关的帖子，不过并没有给出解决方案。


目前我们实施的是 **第一种方案**，即在客户端和Shark之间添加一层，不支持的SQL语句直接降级用Hive执行，效果不错。


[1]: http://cloudera.rst.im/shark/shark-0.9.1-bin-2.0.0-mr1-cdh-4.6.0.tar.gz
[2]: http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH4/4.5.0/CDH-Version-and-Packaging-Information/cdhvd_topic_3.html
[3]: http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH4/4.5.0/CDH4-Installation-Guide/cdh4ig_hive_metastore_configure.html
[4]: https://github.com/amplab/hive
[5]: https://github.com/amplab/shark/wiki/Running-Shark-on-a-Cluster#dependencies
[6]: https://groups.google.com/forum/#!starred/shark-users/x_Dh5-3isIc
