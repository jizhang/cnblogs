---
title: Apache Beam 快速入门（Python 版）
tags:
  - apache beam
  - python
  - mapreduce
  - stream processing
categories:
  - Big Data
---

[Apache Beam][1] 是一种大数据处理标准，由谷歌于 2016 年创建。它提供了一套统一的 DSL 用以处理离线和实时数据，并能在目前主流的大数据处理平台上使用，包括 Spark、Flink、以及谷歌自身的商业套件 Dataflow。Beam 的数据模型基于过去的几项研究成果：[FlumeJava][2]、[Millwheel][3]，适用场景包括 ETL、统计分析、实时计算等。目前，Beam 提供了两种语言的 SDK：Java、Python。本文将讲述如何使用 Python 编写 Beam 应用程序。

![Apache Beam Pipeline](/cnblogs/images/beam/arch.jpg)

## 安装 Apache Beam

Apache Beam Python SDK 必须使用 Python 2.7.x 版本，你可以安装 [pyenv][5] 来管理不同版本的 Python，或者直接从[源代码][6]编译安装（需要支持 SSL）。之后，你便可以在 Python 虚拟环境中安装 Beam SDK 了：

```
$ virtualenv venv --distribute
$ source venv/bin/activate
(venv) $ pip install apache-beam
```

<!-- more -->

## Wordcount 示例

Wordcount 是大数据领域的 Hello World，我们来看如何使用 Beam 实现：

```python
from __future__ import print_function
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
with beam.Pipeline(options=PipelineOptions()) as p:
    lines = p | 'Create' >> beam.Create(['cat dog', 'snake cat', 'dog'])
    counts = (
        lines
        | 'Split' >> (beam.FlatMap(lambda x: x.split(' '))
                      .with_output_types(unicode))
        | 'PairWithOne' >> beam.Map(lambda x: (x, 1))
        | 'GroupAndSum' >> beam.CombinePerKey(sum)
    )
    counts | 'Print' >> beam.ParDo(lambda (w, c): print('%s: %s' % (w, c)))
```

运行脚本，我们便可得到每个单词出现的次数：

```
(venv) $ python wordcount.py
cat: 2
snake: 1
dog: 2
```

Apache Beam 有三个重要的基本概念：Pipeline、PCollection、以及 Transform。

* **Pipeline** （管道）用以构建数据集和处理过程的 DAG（有向无环图）。我们可以将它看成 MapReduce 中的 `Job` 或是 Storm 的 `Topology`。
* **PCollection** 是一种数据结构，我们可以对其进行各类转换操作，如解析、过滤、聚合等。它和 Spark 中的 `RDD` 概念类似。
* **Transform** （转换）则用于编写业务逻辑。通过它，我们可以将一个 PCollection 转换成另一个 PCollection。Beam 提供了许多内置的转换函数，我们将在下文讨论。

在本例中，`Pipeline` 和 `PipelineOptions` 用来创建一个管道。通过 `with` 关键字，上下文管理器会自动调用 `Pipeline.run` 和 `wait_until_finish` 方法。

```
[Output PCollection] = [Input PCollection] | [Label] >> [Transform]
```

`|` 是 Beam 引入的新操作符，用来添加一个转换。每次转换都可以定义一个唯一的标签，默认由 Beam 自动生成。转换能够串联，我们可以构建出不同形态的转换流程，它们在运行时会表示为一个 DAG。

`beam.Create` 用来从内存数据创建出一个 PCollection，主要用于测试和演示。Beam 提供了多种内置的输入源（Source）和输出目标（Sink），可以接收和写入有界（Bounded）或无界（Unbounded）的数据，并且能进行自定义。

`beam.Map` 是一种 *一对一* 的转换，本例中我们将一个个单词转换成形如 `(word, 1)` 的元组。`beam.FlatMap` 则是 `Map` 和 `Flatten` 的结合体，通过它，我们将包含多个单词的数组合并成一个一维的数组。

`CombinePerKey` 的输入源是一系列的二元组（2-element tuple）。这个操作会将元素的第一个元素作为键进行分组，并将相同键的值（第二个元素）组成一个列表。最后，我们使用 `beam.ParDo` 输出统计结果。这个转换函数比较底层，我们会在下文详述。

## Input and Output

Currently, Beam's Python SDK has very limited supports for IO. This table ([source][7]) gives an overview of the available built-in transforms:

| Language | File-based | Messaging | Database |
| --- | --- | --- | --- |
| Java | HDFS<br>TextIO<br>XML | AMQP<br>Kafka<br>JMS | Hive<br>Solr<br>JDBC |
| Python | textio<br>avroio<br>tfrecordio | - | Google Big Query<br>Google Cloud Datastore |

The following snippet demonstrates the usage of `textio`:

```python
lines = p | 'Read' >> beam.io.ReadFromText('/path/to/input-*.csv')
lines | 'Write' >> beam.io.WriteToText('/path/to/output', file_name_suffix='.csv')
```

`textio` is able to read multiple input files by using wildcard or you can flatten PCollections created from difference sources. The outputs are also split into several files due to pipeline's parallel processing nature.

## Transforms

There're basic transforms and higher-level built-ins. In general, we prefer to use the later so that we can focus on the application logic. The following table lists some commonly used higher-level transforms:

| Transform | Meaning |
| --- | --- |
| Create(value) | Creates a PCollection from an iterable. |
| Filter(fn) | Use callable `fn` to filter out elements. |
| Map(fn) | Use callable `fn` to do a one-to-one transformation. |
| FlatMap(fn) | Similar to `Map`, but `fn` needs to return an iterable of zero or more elements, and these iterables will be flattened into one PCollection. |
| Flatten() | Merge several PCollections into a single one. |
| Partition(fn) | Split a PCollection into several partitions. `fn` is a `PartitionFn` or a callable that accepts two arguments - `element`, `num_partitions`. |
| GroupByKey() | Works on a PCollection of key/value pairs (two-element tuples), groups by common key, and returns `(key, iter<value>)` pairs. |
| CoGroupByKey() | Groups results across several PCollections by key. e.g. input `(k, v)` and `(k, w)`, output `(k, (iter<v>, iter<w>))`. |
| RemoveDuplicates() | Get distint values in PCollection. |
| CombinePerKey(fn) | Similar to `GroupByKey`, but combines the values by a `CombineFn` or a callable that takes an iterable, such as `sum`, `max`. |
| CombineGlobally(fn) | Reduces a PCollection to a single value by applying `fn`. |

### Callable, DoFn, and ParDo

Most transforms accepts a callable as argument. In Python, [callable][8] can be a function, method, lambda expression, or class instance that has `__call__` method. Under the hood, Beam will wrap the callable as a `DoFn`, and all these transforms will invoke `ParDo`, the lower-level transform, with the `DoFn`.

Let's replace the expression `lambda x: x.split(' ')` with a `DoFn` class:

```python
class SplitFn(beam.DoFn):
    def process(self, element):
        return element.split(' ')

lines | beam.ParDo(SplitFn())
```

The `ParDo` transform works like `FlatMap`, except that it only accepts `DoFn`. In addition to `return`, we can `yield` element from `process` method:

```python
class SplitAndPairWithOneFn(beam.DoFn):
    def process(self, element):
        for word in element.split(' '):
            yield (word, 1)
```

### Combiner Functions

Combiner functions, or `CombineFn`, are used to reduce a collection of elements into a single value. You can either perform on the entire PCollection (`CombineGlobally`), or combine the values for each key (`CombinePerKey`). Beam is capable of wrapping callables into `CombinFn`. The callable should take an iterable and returns a single value. Since Beam distributes computation to multiple nodes, the combiner function will be invoked multiple times to get partial results, so they ought to be [commutative][9] and [associative][10]. `sum`, `min`, `max` are good examples.

Beam provides some built-in combiners like count, mean, top. Take count for instance, the following two lines are equivalent, they return the total count of lines.

```python
lines | beam.combiners.Count.Globally()
lines | beam.CombineGlobally(beam.combiners.CountCombineFn())
```

Other combiners can be found in Beam Python SDK Documentation ([link][12]). For more complex combiners, we need to subclass the `CombinFn` and implement four methods. Take the built-in `Mean` for an example:

[`apache_beam/transforms/combiners.py`][11]

```python
class MeanCombineFn(core.CombineFn):
  def create_accumulator(self):
    """Create a "local" accumulator to track sum and count."""
    return (0, 0)

  def add_input(self, (sum_, count), element):
    """Process the incoming value."""
    return sum_ + element, count + 1

  def merge_accumulators(self, accumulators):
    """Merge several accumulators into a single one."""
    sums, counts = zip(*accumulators)
    return sum(sums), sum(counts)

  def extract_output(self, (sum_, count)):
    """Compute the mean average."""
    if count == 0:
      return float('NaN')
    return sum_ / float(count)
```

### Composite Transform

Take a look at the [source code][13] of `beam.combiners.Count.Globally` we used before. It subclasses `PTransform` and applies some transforms to the PCollection. This forms a sub-graph of DAG, and we call it composite transform. Composite transforms are used to gather relative codes into logical modules, making them easy to understand and maintain.

```python
class Count(object):
  class Globally(ptransform.PTransform):
    def expand(self, pcoll):
      return pcoll | core.CombineGlobally(CountCombineFn())
```

More built-in transforms are listed below:

| Transform | Meaning |
| --- | --- |
| Count.Globally() | Count the total number of elements. |
| Count.PerKey() | Count number elements of each unique key. |
| Count.PerElement() | Count the occurrences of each element. |
| Mean.Globally() | Compute the average of all elements. |
| Mean.PerKey() | Compute the averages for each key. |
| Top.Of(n, reverse) | Get the top `n` elements from the PCollection. See also Top.Largest(n), Top.Smallest(n). |
| Top.PerKey(n, reverse) | Get top `n` elements for each key. See also Top.LargestPerKey(n), Top.SmallestPerKey(n) |
| Sample.FixedSizeGlobally(n) | Get a sample of `n` elements. |
| Sample.FixedSizePerKey(n) | Get samples from each key. |
| ToList() | Combine to a single list. |
| ToDict() | Combine to a single dict. Works on 2-element tuples. |

## Windowing

When processing event data, such as access log or click stream, there's an *event time* property attached to every item, and it's common to perform aggregation on a per-time-window basis. With Beam, we can define different kinds of windows to divide event data into groups. Windowing can be used in both bounded and unbounded data source. Since current Python SDK only supports bounded source, the following example will work on an offline access log file, but the process can be applied to unbounded source as is.

```
64.242.88.10 - - [07/Mar/2004:16:05:49 -0800] "GET /edit HTTP/1.1" 401 12846
64.242.88.10 - - [07/Mar/2004:16:06:51 -0800] "GET /rdiff HTTP/1.1" 200 4523
64.242.88.10 - - [07/Mar/2004:16:10:02 -0800] "GET /hsdivision HTTP/1.1" 200 6291
64.242.88.10 - - [07/Mar/2004:16:11:58 -0800] "GET /view HTTP/1.1" 200 7352
64.242.88.10 - - [07/Mar/2004:16:20:55 -0800] "GET /view HTTP/1.1" 200 5253
```

`logmining.py`, full source code can be found on GitHub ([link][14]).

```python
lines = p | 'Create' >> beam.io.ReadFromText('access.log')
windowed_counts = (
    lines
    | 'Timestamp' >> beam.Map(lambda x: beam.window.TimestampedValue(
                              x, extract_timestamp(x)))
    | 'Window' >> beam.WindowInto(beam.window.SlidingWindows(600, 300))
    | 'Count' >> (beam.CombineGlobally(beam.combiners.CountCombineFn())
                  .without_defaults())
)
windowed_counts =  windowed_counts | beam.ParDo(PrintWindowFn())
```

First of all, we need to add a timestamp to each record. `extract_timestamp` is a custom function to parse `[07/Mar/2004:16:05:49 -0800]` as a unix timestamp. `TimestampedValue` links this timestamp to the record. Then we define a sliding window with the size *10 minutes* and period *5 minutes*, which means the first window is `[00:00, 00:10)`, second window is `[00:05, 00:15)`, and so forth. All windows have a *10 minutes* duration, and adjacent windows have a *5 minutes* shift. Sliding window is different from fixed window, in that the same elements could appear in different windows. The combiner function is a simple count, so the pipeline result of the first five logs will be:

```
[2004-03-08T00:00:00Z, 2004-03-08T00:10:00Z) @ 2
[2004-03-08T00:05:00Z, 2004-03-08T00:15:00Z) @ 4
[2004-03-08T00:10:00Z, 2004-03-08T00:20:00Z) @ 2
[2004-03-08T00:15:00Z, 2004-03-08T00:25:00Z) @ 1
[2004-03-08T00:20:00Z, 2004-03-08T00:30:00Z) @ 1
```

In stream processing for unbounded source, event data will arrive in different order, so we need to deal with late data with Beam's watermark and trigger facility. This is a rather advanced topic, and the Python SDK has not yet implemented this feature. If you're interested, please refer to Stream [101][4] and [102][15] articles.

## Pipeline Runner

As metioned above, Apache Beam is just a standard that provides SDK and APIs. It's the pipeline runner that is reponsible to execute the workflow graph. The following matrix lists all available runners and their capabilities compared to Beam Model.

![Beam Capability Matrix](/cnblogs/images/beam/matrix.png)

[Source](https://beam.apache.org/documentation/runners/capability-matrix/)

## References

* https://beam.apache.org/documentation/programming-guide/
* https://beam.apache.org/documentation/sdks/pydoc/2.1.0/
* https://sookocheff.com/post/dataflow/get-to-know-dataflow/

[1]: https://beam.apache.org/get-started/beam-overview/
[2]: https://web.archive.org/web/20160923141630/https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/35650.pdf
[3]: https://web.archive.org/web/20160201091359/http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/41378.pdf
[4]: https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101
[5]: https://github.com/pyenv/pyenv
[6]: https://www.python.org/downloads/source/
[7]: https://beam.apache.org/documentation/io/built-in/
[8]: https://docs.python.org/2/library/functions.html#callable
[9]: https://en.wikipedia.org/wiki/Commutative_property
[10]: https://en.wikipedia.org/wiki/Associative_property
[11]: https://github.com/apache/beam/blob/v2.1.0/sdks/python/apache_beam/transforms/combiners.py#L75
[12]: https://beam.apache.org/documentation/sdks/pydoc/2.1.0/apache_beam.transforms.html#module-apache_beam.transforms.combiners
[13]: https://github.com/apache/beam/blob/v2.1.0/sdks/python/apache_beam/transforms/combiners.py#L101
[14]: https://github.com/jizhang/hello-beam/blob/master/logmining.py
[15]: https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-102
