---
title: 实时计算工具库 stream-lib 使用指南
tags:
  - stream processing
  - java
  - algorithm
categories:
  - Big Data
date: 2017-08-27 13:47:16
---


进行大数据处理时，计算唯一值、95% 分位数等操作非常占用空间和时间。但有时我们只是想对数据集有一个概略的了解，数值的准确性并不那么重要。实时监控系统中也是如此，可以容忍一定的错误率。目前已经有许多算法可以通过牺牲准确性来减少计算所需的空间和时间，这些算法大多支持数据结构之间的合并，因此可以方便地用在实时计算中。[`stream-lib`][1] 就是一个集成了很多此类算法的实时计算工具库，是对现有研究成果的 Java 实现。本文就将介绍这一工具库的使用方法。

## 唯一值计算 `HyperLogLog`

独立访客（UV）是网站的重要指标之一。我们通常会为每一个用户生成一个 UUID，并在 HTTP Cookie 中记录和跟踪，或直接使用 IP 地址做近似计算。我们可以使用一个 `HashSet` 来计算 UV 的准确值，但无疑会占用大量的空间。`HyperLogLog` 则是一种近似算法，用于解决此类唯一值计算的问题。该算法[在对超过 10^9 个唯一值进行计算时可以做到 2% 的标准差，并只占用 1.5 kB 内存][2]。

```xml
<dependency>
    <groupId>com.clearspring.analytics</groupId>
    <artifactId>stream</artifactId>
    <version>2.9.5</version>
</dependency>
```

```java
ICardinality card = new HyperLogLog(10);
for (int i : new int[] { 1, 2, 3, 2, 4, 3 }) {
    card.offer(i);
}
System.out.println(card.cardinality()); // 4
```

<!-- more -->

`HyperLogLog` 会计算每一个成员二进制值首位有多少个零，如果零的最大个数是 `n`，则唯一值数量就是 `2^n`。算法中有两个关键点，首先，成员的值必须是服从正态分布的，这一点可以通过哈希函数实现。`stream-lib` 使用的是 [MurmurHash][3]，它简单、快速、且符合分布要求，应用于多种基于哈希查询的算法。其次，为了降低计算结果的方差，集合成员会先被拆分成多个子集合，最后的唯一值数量是各个子集合结果的调和平均数。上文代码中，我们传递给 `HyperLogLog` 构造函数的整型参数就表示会采用多少个二进制位来进行分桶。最后，准确性可以通过这个公式计算：`1.04/sqrt(2^log2m)`。

`HyperLogLog` 是对 `LogLog` 算法的扩展，而 `HyperLogLogPlus` 则包含了更多优化策略。比如，它使用了 64 位的哈希函数，以减少哈希碰撞；对于唯一值数较小的集合，会引入纠偏机制；此外，它还对子集合的存储方式做了改进，能够从稀疏型的数据结构逐渐扩展为密集型。这几种算法都已包含在 `stream-lib` 中。

## 集合成员测试 `BloomFilter`

![Bloom Filter](/images/stream-lib/bloom-filter.jpg)

`BloomFilter` 用于检测一个元素是否包含在集合中，是一种广泛应用的数据结构。它的特点是有一定几率误报（False Positive Probability, FPP），但绝不会漏报（False Negative）。举例来说，Chrome 在检测恶意 URL 时，会在本地维护一个布隆过滤器。当用户输入一个 URL 时，如果布隆过滤器说它不在恶意网址库里，则它一定不在；如果返回结果为真，则 Chrome 会进一步请求服务器以确认是否的确是恶意网址，并提示给用户。

```java
Filter filter = new BloomFilter(100, 0.01);
filter.add("google.com");
filter.add("twitter.com");
filter.add("facebook.com");
System.out.println(filter.isPresent("bing.com")); // false
```

布隆过滤器的构造过程比较简单：

* 创建一个包含 `n` 个元素的位数组，Java 中可以直接使用 [`BitSet`][6]；
* 使用 `k` 个哈希函数对新元素进行处理，结果更新到数组的对应位置中；
* 当需要测试一个元素是否在集合中时，同样进行 `k` 次哈希：
  * 若哈希结果的每一位都命中了，那这个元素就有可能会在集合中（False Positive）；
  * 如果不是所有的比特位都命中，则该元素一定不在集合中。

同样，这些哈希函数必须是服从正态分布的，且要做到两两之间相互独立。Murmur 哈希算法能够满足这一要求。FPP 的计算公式为 `(1-e^(-kn/m))^k`，这个页面（[链接][4]）提供了在线的布隆过滤器可视化过程。这一算法的其它应用场景有：邮件服务器中用来判别垃圾发件人；Cassandra、HBase 会用它来过滤不存在的记录行；Squid 则基于布隆过滤器实现了[缓存摘要][5]。

## Top K 排名 `CountMinSketch`

![Count Min Sketch](/images/stream-lib/count-min-sketch.png)

[图片来源](https://stackoverflow.com/a/35356116/1030720)

[`CountMinSketch`][9] 是一种“速写”算法，能够使用较小的空间勾勒出数据集内各类事件的频次。比如，我们可以统计出当前最热门的推特内容，或是计算网站访问量最大的页面。当然，这一算法同样会牺牲一定的准确性。

下面这段代码演示的是如何使用 `stream-lib` 来统计数据量最多的记录：

```java
List<String> animals;
IFrequency freq = new CountMinSketch(10, 5, 0);
Map<String, Long> top = Collections.emptyMap();
for (String animal : animals) {
    freq.add(animal, 1);
    top = Stream.concat(top.keySet().stream(), Stream.of(animal)).distinct()
              .map(a -> new SimpleEntry<String, Long>(a, freq.estimateCount(a)))
              .sorted(Comparator.comparing(SimpleEntry<String, Long>::getValue).reversed())
              .limit(3)
              .collect(Collectors.toMap(SimpleEntry::getKey, SimpleEntry::getValue));
}

System.out.println(top); // {rabbit=25, bird=45, spider=35}
```

`CountMinSketch#estimateCount` 方法又称为 *点查询* ，用来读取“速写”中某一事件的频次。由于数据结构中无法记录具体的值，我们需要在另行编写代码来实现。

`CountMinSketch` 的数据结构和布隆过滤器类似，只不过它会使用 `d` 个 `w` 位的数组，从而组成一个 `d x w` 的矩阵。加入新值时，该算法会对其应用 `d` 个哈希函数，并更新到矩阵的相应位置。这些哈希函数只需做到[两两独立][7]即可，因此 `stream-lib` 使用了一种简单而快速的算法：`(a*x+b) mod p`。在进行 *点查询* 时，同样计算该值的哈希结果，找到矩阵中最小的值，即是它的频次。

这一算法的误差是 `ε = e / w`，误差概率是 `δ = 1 / e ^ d`。因此，我们可以通过增加 `w` 或 `d` 来提升计算精度。算法论文可以查看这个[链接][8]。

## 分位数计算  `T-Digest`

![T-Digest](/images/stream-lib/t-digest.png)

[图片来源](https://dataorigami.net/blogs/napkin-folding/19055451-percentile-and-quantile-estimation-of-big-data-the-t-digest)

中位数、95% 分位数，这类计算在描述性统计中很常见。相较于平均数，中位数不会受到异常值的影响，但它的计算过程比较复杂，需要保留所有具体值，排序后取得中间位置的数作为结果。`T-Digest` 算法则通过一定计算，将数据集的分布情况粗略地记录下来，从而估计出指定的分位数值。

```java
Random rand = new Random();
List<Double> data = new ArrayList<>();
TDigest digest = new TDigest(100);
for (int i = 0; i < 1000000; ++i) {
    double d = rand.nextDouble();
    data.add(d);
    digest.add(d);
}
Collections.sort(data);
for (double q : new double[] { 0.1, 0.5, 0.9 }) {
    System.out.println(String.format("quantile=%.1f digest=%.4f exact=%.4f",
            q, digest.quantile(q), data.get((int) (data.size() * q))));
}
// quantile=0.1 digest=0.0998 exact=0.1003
// quantile=0.5 digest=0.5009 exact=0.5000
// quantile=0.9 digest=0.8994 exact=0.8998
```

`T-Digest` 的论文可以在这个[链接][10]中找到。简单来说，该算法使用了类似一维 k-means 聚类的过程，将真实的分布情况用若干中心点（Centroid）来描述。此外，不同的 `T-Digest` 实例之间可以进行合并，得到一个体积略大、但准确性更高的实例，这一点非常适用于并行计算。

## 总结

我们可以看到，本文提到的大部分算法都是通过牺牲准确性来提升时间与空间的利用效率的。通过对数据集进行“速写”，抓住其中的“特征”，我们就能给出不错的估计结果。`stream-lib` 以及其它开源项目能够让我们非常便捷地将这类算法应用到实际问题中。

## 参考资料

* https://www.javadoc.io/doc/com.clearspring.analytics/stream/2.9.5
* http://www.addthis.com/blog/2011/03/29/new-open-source-stream-summarizing-java-library/
* https://www.mapr.com/blog/some-important-streaming-algorithms-you-should-know-about

[1]: https://github.com/addthis/stream-lib
[2]: https://en.wikipedia.org/wiki/HyperLogLog
[3]: https://en.wikipedia.org/wiki/MurmurHash
[4]: https://llimllib.github.io/bloomfilter-tutorial/
[5]: https://wiki.squid-cache.org/SquidFaq/CacheDigests
[6]: https://docs.oracle.com/javase/8/docs/api/java/util/BitSet.html
[7]: https://en.wikipedia.org/wiki/Pairwise_independence
[8]: https://web.archive.org/web/20060907232042/http://www.eecs.harvard.edu/~michaelm/CS222/countmin.pdf
[9]: https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch
[10]: https://raw.githubusercontent.com/tdunning/t-digest/master/docs/t-digest-paper/histo.pdf
