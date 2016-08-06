---
layout: post
title: "Generate Auto-increment Id in Map-reduce Job"
date: 2013-10-31 09:35
comments: true
categories: [Notes, Big Data]
published: true
---

In DBMS world, it's easy to generate a unique, auto-increment id, using MySQL's [AUTO_INCREMENT attribute][1] on a primary key or MongoDB's [Counters Collection][2] pattern. But when it comes to a distributed, parallel processing framework, like Hadoop Map-reduce, it is not that straight forward. The best solution to identify every record in such framework is to use UUID. But when an integer id is required, it'll take some steps.

Solution A: Single Reducer
--------------------------

This is the most obvious and simple one, just use the following code to specify reducer numbers to 1:

```java
job.setNumReduceTasks(1);
```

And also obvious, there are several demerits:

1. All mappers output will be copied to one task tracker.
2. Only one process is working on shuffel & sort.
3. When producing output, there's also only one process.

The above is not a problem for small data sets, or at least small mapper outputs. And it is also the approach that Pig and Hive use when they need to perform a total sort. But when hitting a certain threshold, the sort and copy phase will become very slow and unacceptable.

<!-- more -->

Solution B: Increment by Number of Tasks
----------------------------------------

Inspired by a [mailing list][3] that is quite hard to find, which is inspired by MySQL master-master setup (with auto\_increment\_increment and auto\_increment\_offset), there's a brilliant way to generate a globally unique integer id across mappers or reducers. Let's take mapper for example:

```java
public static class JobMapper extends Mapper<LongWritable, Text, LongWritable, Text> {

    private long id;
    private int increment;

    @Override
    protected void setup(Context context) throws IOException,
            InterruptedException {

        super.setup(context);

        id = context.getTaskAttemptID().getTaskID().getId();
        increment = context.getConfiguration().getInt("mapred.map.tasks", 0);
        if (increment == 0) {
            throw new IllegalArgumentException("mapred.map.tasks is zero");
        }
    }

    @Override
    protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {

        id += increment;
        context.write(new LongWritable(id),
                new Text(String.format("%d, %s", key.get(), value.toString())));
    }

}
```

The basic idea is simple:

1. Set the initial id to current tasks's id.
2. When mapping each row, increment the id by the number of tasks.

It's also applicable to reducers.

Solution C: Sorted Auto-increment Id
------------------------------------

Here's a real senario: we have several log files pulled from different machines, and we want to identify each row by an auto-increment id, and they should be in time sequence order.

We know Hadoop has a sort phase, so we can use timestamp as the mapper output key, and the framework will do the trick. But the sorting thing happends in one reducer (partition, in fact), so when using multiple reducer tasks, the result is not in total order. To achieve this, we can use the [TotalOrderPartitioner][4].

How about the incremental id? Even though the outputs are in total order, Solution B is not applicable here. So we take another approach: seperate the job in two phases, use the reducer to do sorting *and* counting, then use the second mapper to generate the id.

Here's what we gonna do:

1. Use TotalOrderPartitioner, and generate the partition file.
2. Parse logs in mapper A, use time as the output key.
3. Let the framework do partitioning and sorting.
4. Count records in reducer, write it with [MultipleOutput][5].
5. In mapper B, use count as offset, and increment by 1.

To simplify the situation, we assume to have the following inputs and outputs:

```text
 Input       Output
 
11:00 a     1 11:00 a
12:00 b     2 11:01 aa
13:00 c     3 11:02 aaa

11:01 aa    4 12:00 b
12:01 bb    5 12:01 bb
13:01 cc    6 12:02 bbb

11:02 aaa   7 13:00 c
12:02 bbb   8 13:01 cc
13:02 ccc   9 13:02 ccc
```

### Generate Partition File

To use TotalOrderpartitioner, we need a partition file (i.e. boundaries) to tell the partitioner how to partition the mapper outputs. Usually we'll use [InputSampler.RandomSampler][6] class, but this time let's use a manual partition file.

```java
SequenceFile.Writer writer = new SequenceFile.Writer(fs, getConf(), partition,
        Text.class, NullWritable.class);
Text key = new Text();
NullWritable value = NullWritable.get();
key.set("12:00");
writer.append(key, value);
key.set("13:00");
writer.append(key, value);
writer.close();
```

So basically, the partitioner will partition the mapper outputs into three parts, the first part will be less than "12:00", seceond part ["12:00", "13:00"), thrid ["13:00", ).

And then, indicate the job to use this partition file:

```java
job.setPartitionerClass(TotalOrderPartitioner.class);
otalOrderPartitioner.setPartitionFile(job.getConfiguration(), partition);

// The number of reducers should equal the number of partitions.
job.setNumReduceTasks(3);
```

### Use MutipleOutputs

In the reducer, we need to note down the row count of this partition, to do that, we'll need the MultipleOutputs class, which let use output multiple result files apart from the default "part-r-xxxxx". The reducer's code is as following:

```java
public static class JobReducer extends Reducer<Text, Text, NullWritable, Text> {

    private MultipleOutputs<NullWritable, Text> mos;
    private long count;

    @Override
    protected void setup(Context context)
            throws IOException, InterruptedException {

        super.setup(context);
        mos = new MultipleOutputs<NullWritable, Text>(context);
        count = 0;
    }

    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context)
            throws IOException, InterruptedException {

        for (Text value : values) {
            context.write(NullWritable.get(), value);
            ++count;
        }
    }

    @Override
    protected void cleanup(Context context)
            throws IOException, InterruptedException {

        super.cleanup(context);
        mos.write("count", NullWritable.get(), new LongWritable(count));
        mos.close();
    }

}
```

There're several things to pay attention to:

1. MultipleOutputs is declared as class member, defined in Reducer#setup method, and must be closed at Reducer#cleanup (otherwise the file will be empty).
2. When instantiating MultipleOutputs class, the generic type needs to be the same as reducer's output key/value class.
3. In order to use a different output key/value class, additional setup needs to be done at job definition:

```java
Job job = new Job(getConf());
MultipleOutputs.addNamedOutput(job, "count", SequenceFileOutputFormat.class,
    NullWritable.class, LongWritable.class);
```

For example, if the output folder is "/tmp/total-sort/", there'll be the following files when job is done:

```text
/tmp/total-sort/count-r-00001
/tmp/total-sort/count-r-00002
/tmp/total-sort/count-r-00003
/tmp/total-sort/part-r-00001
/tmp/total-sort/part-r-00002
/tmp/total-sort/part-r-00003
```

### Pass Start Ids to Mapper

When the second mapper processes the inputs, we want them to know the initial id of its partition, which can be calculated from the "count-*" files we produce before. To pass this information, we can use the job's Configuration object.

```java
// Read and calculate the start id from those row-count files.
Map<String, Long> startIds = new HashMap<String, Long>();
long startId = 1;
FileSystem fs = FileSystem.get(getConf());
for (FileStatus file : fs.listStatus(countPath)) {

    Path path = file.getPath();
    String name = path.getName();
    if (!name.startsWith("count-")) {
        continue;
    }

    startIds.put(name.substring(name.length() - 5), startId);

    SequenceFile.Reader reader = new SequenceFile.Reader(fs, path, getConf());
    NullWritable key = NullWritable.get();
    LongWritable value = new LongWritable();
    if (!reader.next(key, value)) {
        continue;
    }
    startId += value.get();
    reader.close();
}

// Serialize the map and pass it to Configuration.
job.getConfiguration().set("startIds", Base64.encodeBase64String(
        SerializationUtils.serialize((Serializable) startIds)));
        
// Recieve it in Mapper#setup
public static class JobMapperB extends Mapper<NullWritable, Text, LongWritable, Text> {

    private Map<String, Long> startIds;
    private long startId;

    @SuppressWarnings("unchecked")
    @Override
    protected void setup(Context context)
            throws IOException, InterruptedException {

        super.setup(context);
        startIds = (Map<String, Long>) SerializationUtils.deserialize(
                Base64.decodeBase64(context.getConfiguration().get("startIds")));
        String name = ((FileSplit) context.getInputSplit()).getPath().getName();
        startId = startIds.get(name.substring(name.length() - 5));
    }

    @Override
    protected void map(NullWritable key, Text value, Context context)
            throws IOException, InterruptedException {

        context.write(new LongWritable(startId++), value);
    }

}
```

### Set the Input Non-splitable

When the file is bigger than a block or so (depending on some configuration entries), Hadoop will split it, which is not good for us. So let's define a new InputFormat class to disable the splitting behaviour:

```java
public static class NonSplitableSequence extends SequenceFileInputFormat<NullWritable, Text> {

    @Override
    protected boolean isSplitable(JobContext context, Path filename) {
        return false;
    }

}

// use it
job.setInputFormatClass(NonSplitableSequence.class);
```

And that's it, we are able to generate a unique, auto-increment id for a sorted collection, with Hadoop's parallel computing capability. The process is rather complicated, which requires several techniques about Hadoop. It's worthwhile to dig.

A workable example can be found in my [Github repository][7]. If you have some more straight-forward approach, please do let me know.

[1]: http://dev.mysql.com/doc/refman/5.1/en/example-auto-increment.html
[2]: http://docs.mongodb.org/manual/tutorial/create-an-auto-incrementing-field/
[3]: http://mail-archives.apache.org/mod_mbox/hadoop-common-user/200904.mbox/%3C49E13557.7090504@domaintools.com%3E
[4]: http://hadoop.apache.org/docs/r1.0.4/api/org/apache/hadoop/mapred/lib/TotalOrderPartitioner.html
[5]: http://hadoop.apache.org/docs/r1.0.4/api/org/apache/hadoop/mapreduce/lib/output/MultipleOutputs.html
[6]: https://hadoop.apache.org/docs/r1.0.4/api/org/apache/hadoop/mapreduce/lib/partition/InputSampler.RandomSampler.html
[7]: https://github.com/jizhang/mapred-sandbox/blob/master/src/main/java/com/shzhangji/mapred_sandbox/AutoIncrementId2Job.java
