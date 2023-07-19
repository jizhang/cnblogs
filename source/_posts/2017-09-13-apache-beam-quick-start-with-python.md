---
title: Apache Beam 快速入门（Python 版）
tags:
  - apache beam
  - python
  - mapreduce
  - stream processing
categories:
  - Big Data
date: 2017-09-13 12:39:03
---


[Apache Beam][1] 是一种大数据处理标准，由谷歌于 2016 年创建。它提供了一套统一的 DSL 用以处理离线和实时数据，并能在目前主流的大数据处理平台上使用，包括 Spark、Flink、以及谷歌自身的商业套件 Dataflow。Beam 的数据模型基于过去的几项研究成果：[FlumeJava][2]、[Millwheel][3]，适用场景包括 ETL、统计分析、实时计算等。目前，Beam 提供了两种语言的 SDK：Java、Python。本文将讲述如何使用 Python 编写 Beam 应用程序。

![Apache Beam Pipeline](/images/beam/arch.jpg)

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

## 输入与输出

目前，Beam Python SDK 对输入输出的支持十分有限。下表列出了现阶段支持的数据源（[资料来源][7]）：

| 语言 | 文件系统 | 消息队列 | 数据库 |
| --- | --- | --- | --- |
| Java | HDFS<br>TextIO<br>XML | AMQP<br>Kafka<br>JMS | Hive<br>Solr<br>JDBC |
| Python | textio<br>avroio<br>tfrecordio | - | Google Big Query<br>Google Cloud Datastore |

这段代码演示了如何使用 `textio` 对文本文件进行读写：

```python
lines = p | 'Read' >> beam.io.ReadFromText('/path/to/input-*.csv')
lines | 'Write' >> beam.io.WriteToText('/path/to/output', file_name_suffix='.csv')
```

通过使用通配符，`textio` 可以读取多个文件。我们还可以从不同的数据源中读取文件，并用 `Flatten` 方法将多个 `PCollection` 合并成一个。输出文件默认也会是多个，因为 Beam Pipeline 是并发执行的，不同的进程会写入独立的文件。

## 转换函数

Beam 中提供了基础和上层的转换函数。通常我们更偏向于使用上层函数，这样就可以将精力聚焦在实现业务逻辑上。下表列出了常用的上层转换函数：

| 转换函数 | 功能含义 |
| --- | --- |
| Create(value) | 基于内存中的集合数据生成一个 PCollection。|
| Filter(fn) | 使用 `fn` 函数过滤 PCollection 中的元素。 |
| Map(fn) | 使用 `fn` 函数做一对一的转换处理。 |
| FlatMap(fn) | 功能和 `Map` 类似，但是 `fn` 需要返回一个集合，里面包含零个或多个元素，最终 `FlatMap` 会将这些集合合并成一个 PCollection。 |
| Flatten() | 合并多个 PCollection。 |
| Partition(fn) | 将一个 PCollection 切分成多个分区。`fn` 可以是 `PartitionFn` 或一个普通函数，能够接受两个参数：`element`、`num_partitions`。 |
| GroupByKey() | 输入源必须是使用二元组表示的键值对，该方法会按键进行分组，并返回一个 `(key, iter<value>)` 的序列。 |
| CoGroupByKey() | 对多个二元组 PCollection 按相同键进行合并，如输入的是 `(k, v)` 和 `(k, w)`，则输出 `(k, (iter<v>, iter<w>))`。 |
| RemoveDuplicates() | 对 PCollection 的元素进行去重。 |
| CombinePerKey(fn) | 功能和 `GroupByKey` 类似，但会进一步使用 `fn` 对值列表进行合并。`fn` 可以是一个 `CombineFn`，或是一个普通函数，接收序列并返回结果，如 `sum`、`max` 函数等。 |
| CombineGlobally(fn) | 使用 `fn` 将整个 PCollection 合并计算成单个值。 |

### Callable, DoFn, ParDo

可以看到，多数转换函数都会接收另一个函数（Callable）做为参数。在 Python 中，[Callable][8] 可以是一个函数、类方法、Lambda 表达式、或是任何包含 `__call__` 方法的对象实例。Beam 会将这些函数包装成一个 `DoFn` 类，所有转换函数最终都会调用最基础的 `ParDo` 函数，并将 `DoFn` 传递给它。

我们可以尝试将 `lambda x: x.split(' ')` 这个表达式转换成 `DoFn` 类：

```python
class SplitFn(beam.DoFn):
    def process(self, element):
        return element.split(' ')

lines | beam.ParDo(SplitFn())
```

`ParDo` 转换和 `FlatMap` 的功能类似，只是它的 `fn` 参数必须是一个 `DoFn`。除了使用 `return`，我们还可以用 `yield` 语句来返回结果：

```python
class SplitAndPairWithOneFn(beam.DoFn):
    def process(self, element):
        for word in element.split(' '):
            yield (word, 1)
```

### 合并函数

合并函数（`CombineFn`）用来将集合数据合并计算成单个值。我们既可以对整个 PCollection 做合并（`CombineGlobally`），也可以计算每个键的合并结果（`CombinePerKey`）。Beam 会将普通函数（Callable）包装成 `CombineFn`，这些函数需要接收一个集合，并返回单个结果。需要注意的是，Beam 会将计算过程分发到多台服务器上，合并函数会被多次调用来计算中间结果，因此需要满足[交换律][9]和[结合律][10]。`sum`、`min`、`max` 是符合这样的要求的。

Beam 提供了许多内置的合并函数，如计数、求平均值、排序等。以计数为例，下面两种写法都可以用来统计整个 PCollection 中元素的个数：

```python
lines | beam.combiners.Count.Globally()
lines | beam.CombineGlobally(beam.combiners.CountCombineFn())
```

其他合并函数可以参考 Python SDK 的官方文档（[链接][12]）。我们也可以自行实现合并函数，只需继承 `CombineFn`，并实现四个方法。我们以内置的 `Mean` 平均值合并函数的源码为例：

[`apache_beam/transforms/combiners.py`][11]

```python
class MeanCombineFn(core.CombineFn):
  def create_accumulator(self):
    """创建一个“本地”的中间结果，记录合计值和记录数。"""
    return (0, 0)

  def add_input(self, (sum_, count), element):
    """处理新接收到的值。"""
    return sum_ + element, count + 1

  def merge_accumulators(self, accumulators):
    """合并多个中间结果。"""
    sums, counts = zip(*accumulators)
    return sum(sums), sum(counts)

  def extract_output(self, (sum_, count)):
    """计算平均值。"""
    if count == 0:
      return float('NaN')
    return sum_ / float(count)
```

### 复合转换函数

我们简单看一下上文中使用到的 `beam.combiners.Count.Globally` 的源码（[链接][13]），它继承了 `PTransform` 类，并在 `expand` 方法中对 PCollection 应用了转换函数。这会形成一个小型的有向无环图，并合并到最终的 DAG 中。我们称其为复合转换函数，主要用于将相关的转换逻辑整合起来，便于理解和管理。

```python
class Count(object):
  class Globally(ptransform.PTransform):
    def expand(self, pcoll):
      return pcoll | core.CombineGlobally(CountCombineFn())
```

更多内置的复合转换函数如下表所示：

| 复合转换函数 | 功能含义 |
| --- | --- |
| Count.Globally() | 计算元素总数。 |
| Count.PerKey() | 计算每个键的元素数。 |
| Count.PerElement() | 计算每个元素出现的次数，类似 Wordcount。 |
| Mean.Globally() | 计算所有元素的平均值。 |
| Mean.PerKey() | 计算每个键的元素平均值。 |
| Top.Of(n, reverse) | 获取 PCollection 中最大或最小的 `n` 个元素，另有 Top.Largest(n), Top.Smallest(n). |
| Top.PerKey(n, reverse) | 获取每个键的值列表中最大或最小的 `n` 个元素，另有 Top.LargestPerKey(n), Top.SmallestPerKey(n) |
| Sample.FixedSizeGlobally(n) | 随机获取 `n` 个元素。 |
| Sample.FixedSizePerKey(n) | 随机获取每个键下的 `n` 个元素。 |
| ToList() | 将 PCollection 合并成一个列表。 |
| ToDict() | 将 PCollection 合并成一个哈希表，输入数据需要是二元组集合。 |

## 时间窗口

在处理事件数据时，如访问日志、用户点击流，每条数据都会有一个 *事件时间* 属性，而通常我们会按事件时间对数据进行分组统计，这些分组即时间窗口。在 Beam 中，我们可以定义不同的时间窗口类型，能够支持有界和无界数据。由于 Python SDK 暂时只支持有界数据，我们就以一个离线访问日志文件作为输入源，统计每个时间窗口的记录条数。对于无界数据，概念和处理流程也是类似的。

```
64.242.88.10 - - [07/Mar/2004:16:05:49 -0800] "GET /edit HTTP/1.1" 401 12846
64.242.88.10 - - [07/Mar/2004:16:06:51 -0800] "GET /rdiff HTTP/1.1" 200 4523
64.242.88.10 - - [07/Mar/2004:16:10:02 -0800] "GET /hsdivision HTTP/1.1" 200 6291
64.242.88.10 - - [07/Mar/2004:16:11:58 -0800] "GET /view HTTP/1.1" 200 7352
64.242.88.10 - - [07/Mar/2004:16:20:55 -0800] "GET /view HTTP/1.1" 200 5253
```

`logmining.py` 的完整源码可以在 GitHub（[链接][14]）中找到：

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

首先，我们需要为每一条记录附加上时间戳。自定义函数 `extract_timestamp` 用以将日志中的时间 `[07/Mar/2004:16:05:49 -0800]` 转换成 Unix 时间戳，`TimestampedValue` 则会将这个时间戳和对应记录关联起来。之后，我们定义了一个大小为 *10 分钟*，间隔为 *5 分钟* 的滑动窗口（Sliding Window）。从零点开始，第一个窗口的范围是 `[00:00, 00:10)`，第二个窗口的范围是 `[00:05, 00:15)`，以此类推。所有窗口的长度都是 *10 分钟*，相邻两个窗口之间相隔 *5 分钟*。滑动窗口和固定窗口（Fixed Window）不同，因为相同的元素可能会落入不同的窗口中参与计算。最后，我们使用一个合并函数计算每个窗口中的记录数。通过这个方法得到前五条记录的计算结果为：

```
[2004-03-08T00:00:00Z, 2004-03-08T00:10:00Z) @ 2
[2004-03-08T00:05:00Z, 2004-03-08T00:15:00Z) @ 4
[2004-03-08T00:10:00Z, 2004-03-08T00:20:00Z) @ 2
[2004-03-08T00:15:00Z, 2004-03-08T00:25:00Z) @ 1
[2004-03-08T00:20:00Z, 2004-03-08T00:30:00Z) @ 1
```

在无界数据的实时计算过程中，事件数据的接收顺序是不固定的，因此需要利用 Beam 的水位线和触发器机制来处理延迟数据（Late Data）。这个话题比较复杂，而且 Python SDK 尚未支持这些特性，感兴趣的读者可以参考 Stream [101][4] 和 [102][15] 这两篇文章。

## Pipeline 运行时

上文中提到，Apache Beam 是一个数据处理标准，只提供了 SDK 和 API，因而必须使用 Spark、Flink 这样的计算引擎来运行它。下表列出了当前支持 Beam Model 的引擎，以及他们的兼容程度：

![Beam 运行时能力矩阵](/images/beam/matrix.png)

[图片来源](https://beam.apache.org/documentation/runners/capability-matrix/)

## 参考资料

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
[14]: https://github.com/jizhang/blog-demo/blob/master/hello-beam/logmining.py
[15]: https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-102
