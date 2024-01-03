---
layout: post
title: "深入理解 Reduce-side Join"
date: 2015-01-13 14:20
comments: true
tags: [hadoop, mapreduce]
categories: Big Data
published: true
---

在《[MapReduce Design Patterns][1]》一书中，作者给出了Reduce-side Join的实现方法，大致步骤如下：

![](/images/reduce-side-join/reduce-side-join.png)

1. 使用[MultipleInputs][2]指定不同的来源表和相应的Mapper类；
2. Mapper输出的Key为Join的字段内容，Value为打了来源表标签的记录；
3. Reducer在接收到同一个Key的记录后，执行以下两步：
    1. 遍历Values，根据标签将来源表的记录分别放到两个List中；
    2. 遍历两个List，输出Join结果。

具体实现可以参考[这段代码][3]。但是这种实现方法有一个问题：如果同一个Key的记录数过多，存放在List中就会占用很多内存，严重的会造成内存溢出（Out of Memory, OOM）。这种方法在一对一的情况下没有问题，而一对多、多对多的情况就会有隐患。那么，Hive在做Reduce-side Join时是如何避免OOM的呢？两个关键点：

1. Reducer在遍历Values时，会将前面的表缓存在内存中，对于最后一张表则边扫描边输出；
2. 如果前面几张表内存中放不下，就写入磁盘。

<!-- more -->

按照我们的实现，Mapper输出的Key是`product_id`，Values是打了标签的产品表（Product）和订单表（Order）的记录。从数据量来看，应该缓存产品表，扫描订单表。这就要求两表记录到达Reducer时是有序的，产品表在前，边扫描边放入内存；订单表在后，边扫描边结合产品表的记录进行输出。要让Hadoop在Shuffle&Sort阶段先按`product_id`排序、再按表的标签排序，就需要用到二次排序。

二次排序的概念很简单，将Mapper输出的Key由单一的`product_id`修改为`product_id+tag`的复合Key就可以了，但需通过以下几步实现：

### 自定义Key类型

原来`product_id`是Text类型，我们的复合Key则要包含`product_id`和`tag`两个数据，并实现`WritableComparable`接口：

```java
public class TaggedKey implements WritableComparable<TaggedKey> {

    private Text joinKey = new Text();
    private IntWritable tag = new IntWritable();

    @Override
    public int compareTo(TaggedKey taggedKey) {
        int compareValue = joinKey.compareTo(taggedKey.getJoinKey());
        if (compareValue == 0) {
            compareValue = tag.compareTo(taggedKey.getTag());
        }
        return compareValue;
    }

    // 此处省略部分代码
}
```

可以看到，在比较两个TaggedKey时，会先比较joinKey（即`product_id`），再比较`tag`。

### 自定义分区方法

默认情况下，Hadoop会对Key进行哈希，以保证相同的Key会分配到同一个Reducer中。由于我们改变了Key的结构，因此需要重新编 写分区函数：

```java
public class TaggedJoiningPartitioner extends Partitioner<TaggedKey, Text> {

    @Override
    public int getPartition(TaggedKey taggedKey, Text text, int numPartitions) {
        return taggedKey.getJoinKey().hashCode() % numPartitions;
    }

}
```

### 自定义分组方法

同理，调用reduce函数需要传入同一个Key的所有记录，这就需要重新定义分组函数：

```java
public class TaggedJoiningGroupingComparator extends WritableComparator {

    public TaggedJoiningGroupingComparator() {
        super(TaggedKey.class, true);
    }

    @SuppressWarnings("rawtypes")
    @Override
    public int compare(WritableComparable a, WritableComparable b) {
        TaggedKey taggedKey1 = (TaggedKey) a;
        TaggedKey taggedKey2 = (TaggedKey) b;
        return taggedKey1.getJoinKey().compareTo(taggedKey2.getJoinKey());
    }

}
```

### 配置Job

```java
job.setMapOutputKeyClass(TaggedKey.class);
job.setMapOutputValueClass(Text.class);

job.setPartitionerClass(TaggedJoiningPartitioner.class);
job.setGroupingComparatorClass(TaggedJoiningGroupingComparator.class);
```

### MapReduce过程

最后，我们在Mapper阶段使用TaggedKey，在Reducer阶段按照tag进行不同的操作就可以了：

```java
@Override
protected void reduce(TaggedKey key, Iterable<Text> values, Context context)
        throws IOException, InterruptedException {

    List<String> products = new ArrayList<String>();

    for (Text value : values) {
        switch (key.getTag().get()) {
        case 1: // Product
            products.add(value.toString());
            break;

        case 2: // Order
            String[] order = value.toString().split(",");
            for (String productString : products) {
                String[] product = productString.split(",");
                List<String> output = new ArrayList<String>();
                output.add(order[0]);
                // ...
                context.write(NullWritable.get(), new Text(StringUtils.join(output, ",")));
            }
            break;

        default:
            assert false;
        }
    }
}
```

遍历values时，开始都是tag=1的记录，之后都是tag=2的记录。以上代码可以[在这里][4]查看。

对于第二个问题，超过缓存大小的记录（默认25000条）就会存入临时文件，由Hive的RowContainer类实现，具体可以看[这个链接][5]。

需要注意的是，Hive默认是按SQL中表的书写顺序来决定排序的，因此应该将大表放在最后。如果要人工改变顺序，可以使用STREAMTABLE配置：

```sql
SELECT /*+ STREAMTABLE(a) */ a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)
```

但不要将这点和Map-side Join混淆，在配置了`hive.auto.convert.join=true`后，是不需要注意表的顺序的，Hive会自动将小表缓存在Mapper的内存中。

## 参考资料

1. http://codingjunkie.net/mapreduce-reduce-joins/
2. https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Joins


[1]: http://www.amazon.com/MapReduce-Design-Patterns-Effective-Algorithms/dp/1449327176
[2]: https://hadoop.apache.org/docs/r1.0.4/api/org/apache/hadoop/mapred/lib/MultipleInputs.html
[3]: https://github.com/jizhang/java-sandbox/blob/master/mapred/src/main/java/mapred/join/InnerJoinJob.java
[4]: https://github.com/jizhang/java-sandbox/blob/master/mapred/src/main/java/mapred/join/ReduceSideJoinJob.java
[5]: http://grepcode.com/file/repository.cloudera.com/content/repositories/releases/org.apache.hive/hive-exec/0.10.0-cdh4.5.0/org/apache/hadoop/hive/ql/exec/persistence/RowContainer.java#RowContainer.add%28java.util.List%29
