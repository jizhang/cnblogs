---
layout: post
title: "Clojure 实战(4)：编写 Hadoop MapReduce 脚本"
date: 2013-02-09 16:43
comments: true
categories: [Big Data]
tags: [clojure, hadoop, tutorial]
---

Hadoop简介
----------

众所周知，我们已经进入了大数据时代，每天都有PB级的数据需要处理、分析，从中提取出有用的信息。Hadoop就是这一时代背景下的产物。它是Apache基金会下的开源项目，受[Google两篇论文](http://en.wikipedia.org/wiki/Apache_Hadoop#Papers)的启发，采用分布式的文件系统HDFS，以及通用的MapReduce解决方案，能够在数千台物理节点上进行分布式并行计算。

对于Hadoop的介绍这里不再赘述，读者可以[访问其官网](http://hadoop.apache.org/)，或阅读[Hadoop权威指南](http://product.dangdang.com/main/product.aspx?product_id=21127813)。

Hadoop项目是由Java语言编写的，运行在JVM之上，因此我们可以直接使用Clojure来编写MapReduce脚本，这也是本文的主题。Hadoop集群的搭建不在本文讨论范围内，而且运行MapReduce脚本也无需搭建测试环境。

<!-- more -->

clojure-hadoop类库
------------------

Hadoop提供的API是面向Java语言的，如果不想在Clojure中过多地操作Java对象，那就需要对API进行包装（wrapper），好在已经有人为我们写好了，它就是[clojure-hadoop](https://github.com/alexott/clojure-hadoop)。

从clojure-hadoop的项目介绍中可以看到，它提供了不同级别的包装，你可以选择完全规避对Hadoop类型和对象的操作，使用纯Clojure语言来编写脚本；也可以部分使用Hadoop对象，以提升性能（因为省去了类型转换过程）。这里我们选择前一种，即完全使用Clojure语言。

示例1：Wordcount
---------------

Wordcount，统计文本文件中每个单词出现的数量，可以说是数据处理领域的“Hello, world!”。这一节我们就通过它来学习如何编写MapReduce脚本。

### Leiningen 2

前几章我们使用的项目管理工具`lein`是1.7版的，而前不久Leiningen 2已经正式发布了，因此从本章开始我们的示例都会基于新版本。新版`lein`的安装过程也很简单：

```bash
$ cd ~/bin
$ wget https://raw.github.com/technomancy/leiningen/stable/bin/lein
$ chmod 755 lein
$ lein repl
user=>
```

其中，`lein repl`这一步会下载`lein`运行时需要的文件，包括Clojure 1.4。

### 新建项目

```bash
$ lein new cia-hadoop
```

编辑`project.clj`文件，添加依赖项`clojure-hadoop "1.4.1"`，尔后执行`lein deps`。

### Map和Reduce

MapReduce，简称mapred，是Hadoop的核心概念之一。可以将其理解为处理问题的一种方式，即将大问题拆分成多个小问题来分析和解决，最终合并成一个结果。其中拆分的过程就是Map，合并的过程就是Reduce。

以Wordcount为例，将一段文字划分成一个个单词的过程就是Map。这个过程是可以并行执行的，即将文章拆分成多个段落，每个段落分别在不同的节点上执行划分单词的操作。这个过程结束后，我们便可以统计各个单词出现的次数，这也就是Reduce的过程。同样，Reduce也是可以并发执行的。整个过程如下图所示：

![Wordcount](/images/cia-hadoop/wordcount.png)

中间Shuffle部分的功能是将Map输出的数据按键排序，交由Reduce处理。整个过程全部由Hadoop把控，开发者只需编写`Map`和`Reduce`函数，这也是Hadoop强大之处。

#### 编写Map函数

在本示例中，我们处理的原始数据是文本文件，Hadoop会逐行读取并调用Map函数。Map函数会接收到两个参数：`key`是一个长整型，表示该行在整个文件中的偏移量，很少使用；`value`则是该行的内容。以下是将一行文字拆分成单词的Map函数：

```clojure
;; src/cia_hadoop/wordcount.clj

(ns cia-hadoop.wordcount
  (:require [clojure-hadoop.wrap :as wrap]
            [clojure-hadoop.defjob :as defjob])
  (:import [java.util StringTokenizer])
  (:use clojure-hadoop.job))

(defn my-map [key value]
  (map (fn [token] [token 1])
       (enumeration-seq (StringTokenizer. value))))
```

可以看到，这是一个纯粹的Clojure函数，并没有调用Hadoop的API。函数体虽然只有两行，但还是包含了很多知识点的：

`(map f coll)`函数的作用是将函数`f`应用到序列`coll`的每个元素上，并返回一个新的序列。如`(map inc [1 2 3])`会对每个元素做加1操作（参考`(doc inc)`），返回`[2 3 4]`。值得一提的是，`map`函数返回的是一个惰性序列（lazy sequence），即序列元素不会一次性完全生成，而是在遍历过程中逐个生成，这在处理元素较多的序列时很有优势。

`map`函数接收的参数自然不会只限于Clojure内部函数，我们可以将自己定义的函数传递给它：

```clojure
(defn my-inc [x]
  (+ x 1))

(map my-inc [1 2 3]) ; -> [2 3 4]
```

我们更可以传递一个匿名函数给`map`。上一章提过，定义匿名函数的方式是使用`fn`，另外还可使用`#(...)`简写：

```clojure
(map (fn [x] (+ x 1)) [1 2 3])
(map #(+ % 1) [1 2 3])
```

对于含有多个参数的情况：

```clojure
((fn [x y] (+ x y)) 1 2) ; -> 3
(#(+ %1 %2) 1 2) ; -> 3
```

`my-map`中的`(fn [token] [token 1])`即表示接收参数`token`，返回一个向量`[token 1]`，其作用等价于`#(vector % 1)`。为何是`[token 1]`，是因为Hadoop的数据传输都是以键值对的形式进行的，如`["apple" 1]`即表示“apple”这个单词出现一次。

[StringTokenizer](http://docs.oracle.com/javase/6/docs/api/java/util/StringTokenizer.html)则是用来将一行文字按空格拆分成单词的。他的返回值是`Enumeration`类型，Clojure提供了`enumeration-seq`函数，可以将其转换成序列进行操作。

所以最终`my-map`函数的作用就是：将一行文字按空格拆分成单词，返回一个形如`[["apple" 1] ["orange" 1] ...]`的序列。

#### 编写Reduce函数

从上文的图表中可以看到，Map函数处理完成后，Hadoop会对结果按照键进行排序，并使用`key, [value1 value2 ...]`的形式调用Reduce函数。在clojure-hadoop中，Reduce函数的第二个参数是一个函数，其返回结果才是值的序列：

```clojure
(defn my-reduce [key values-fn]
  [[key (reduce + (values-fn))]])
```

和Map函数相同，Reduce函数的返回值也是一个序列，其元素是一个个`[key value]`。注意，函数体中的`(reduce f coll)`是Clojure的内置函数，其作用是：取`coll`序列的第1、2个元素作为参数执行函数`f`，将结果和`coll`序列的第3个元素作为参数执行函数`f`，依次类推。因此`(reduce + [1 2 3])`等价于`(+ (+ 1 2) 3)`。

#### 定义脚本

有了Map和Reduce函数，我们就可以定义一个完整的脚本了：

```clojure
(defjob/defjob job
  :map my-map
  :map-reader wrap/int-string-map-reader
  :reduce my-reduce
  :input-format :text
  :output-format :text
  :compress-output false
  :replace true
  :input "README.md"
  :output "out-wordcount")
```

简单说明一下这些配置参数：`:map`和`:reduce`分别指定Map和Reduce函数；`map-reader`表示读取数据文件时采用键为`int`、值为`string`的形式；`:input-format`至`compress-output`指定了输入输出的文件格式，这里采用非压缩的文本形式，方便阅览；`:replace`表示每次执行时覆盖上一次的结果；`:input`和`:output`则是输入的文件和输出的目录。

#### 执行脚本

我们可以采用Clojure的测试功能来执行脚本：

```clojure
;; test/cia_hadoop/wordcount_test.clj

(ns cia-hadoop.wordcount-test
  (:use clojure.test
        clojure-hadoop.job
        cia-hadoop.wordcount))

(deftest test-wordcount
  (is (run job)))
```

尔后执行：

```bash
$ lein test cia-hadoop.wordcount-test
...
13/02/14 00:25:52 INFO mapred.JobClient:  map 0% reduce 0%
..
13/02/14 00:25:58 INFO mapred.JobClient:  map 100% reduce 100%
...
$ cat out-wordcount/part-r-00000
...
"java"  1
"lein"	3
"locally"	2
"on"	1
...
```

如果想要将MapReduce脚本放到Hadoop集群中执行，可以采用以下命令：

```bash
$ lein uberjar
$ hadoop jar target/cia-hadoop-0.1.0-SNAPSHOT-standalone.jar clojure_hadoop.job -job cia-hadoop.wordcount/job
```

示例2：统计浏览器类型
--------------------

下面我们再来看一个更为实际的示例：从用户的访问日志中统计浏览器类型。

### 需求概述

用户访问网站时，页面中会有段JS请求，将用户的IP、User-Agent等信息发送回服务器，并记录成文本文件的形式：

```text
{"stamp": "1346376858286", "ip": "58.22.113.189", "agent": "Mozilla/5.0 (iPad; CPU OS 5_0_1 like Mac OS X) AppleWebKit/534.46 (KHTML, like Gecko) Version/5.1 Mobile/9A405 Safari/7534.48.3"}
{"stamp": "1346376858354", "ip": "116.233.51.2", "agent": "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; WOW64; Trident/5.0)"}
{"stamp": "1346376858365", "ip": "222.143.28.2", "agent": "Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.1; Trident/4.0)"}
{"stamp": "1346376858423", "ip": "123.151.144.40", "agent": "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/536.11 (KHTML, like Gecko) Chrome/20.0.1132.57 Safari/536.11"}
```

我们要做的是从User-Agent中统计用户使用的浏览器类型所占比例，包括IE、Firefox、Chrome、Opera、Safari、以及其它。

### User-Agent中的浏览器类型

由于一些[历史原因](http://webaim.org/blog/user-agent-string-history/)，User-Agent中的信息是比较凌乱的，浏览器厂商会随意添加信息，甚至仿造其它浏览器的内容。因此在过滤时，我们需要做些额外的处理。Mozilla的[这篇文章](https://developer.mozilla.org/en-US/docs/Browser_detection_using_the_user_agent)很好地概括了如何从User-Agent中获取浏览器类型，大致如下：

* IE: MSIE xyz
* Firefox: Firefox/xyz
* Chrome: Chrome/xyz
* Opera: Opera/xyz
* Safari: Safari/xyz, 且不包含 Chrome/xyz 和 Chromium/xyz

### 解析JSON字符串

Clojure除了内置函数之外，周边还有一个名为`clojure.contrib`的类库，其中囊括了各类常用功能，包括JSON处理。目前`clojure.contrib`中的各个组件已经分开发行，读者可以到 https://github.com/clojure 中浏览。

处理JSON字符串时，首先在项目声明文件中添加依赖项`[org.clojure/data.json "0.2.1"]`，然后就能使用了：

```clojure
user=> (require '[clojure.data.json :as json])
user=> (json/read-str "{\"a\":1,\"b\":2}")
{"a" 1, "b" 2}
user=> (json/write-str [1 2 3])
"[1,2,3]"
```

### 正则表达式

Clojure提供了一系列的内置函数来使用正则表达式，其实质上是对`java.util.regex`命名空间的包装。

```clojure
user=> (def ptrn #"[0-9]+") ; #"..."是定义正则表达式对象的简写形式
user=> (def ptrn (re-pattern "[0-9]+")) ; 和上式等价
user=> (re-matches ptrn "123") ; 完全匹配
"123"
user=> (re-find ptrn "a123") ; 返回第一个匹配项
"123"
user=> (re-seq ptrn "a123b456") ; 返回匹配项序列（惰性序列）
("123" "456")
user=> (re-find #"([a-z]+)/([0-9]+)" "a/1") ; 子模式
["a/1" "a" "1"]
user=> (def m (re-matcher #"([a-z]+)/([0-9]+)" "a/1 b/2")) ; 返回一个Matcher对象
user=> (re-find m) ; 返回第一个匹配
["a/1" "a" "1"]
user=> (re-groups m) ; 获取当前匹配
["a/1" "a" "1"]
user=> (re-find m) ; 返回下一个匹配，或nil
["b/2" "b" "2"]
```

### Map函数

```clojure
(defn json-decode [s]
  (try
    (json/read-str s)
    (catch Exception e)))

(def rule-set {"ie" (partial re-find #"(?i)MSIE [0-9]+")
               "chrome" (partial re-find #"(?i)Chrome/[0-9]+")
               "firefox" (partial re-find #"(?i)Firefox/[0-9]+")
               "opera" (partial re-find #"(?i)Opera/[0-9]+")
               "safari" #(and (re-find #"(?i)Safari/[0-9]+" %)
                              (not (re-find #"(?i)Chrom(e|ium)/[0-9]+" %)))
               })

(defn get-type [ua]
  (if-let [rule (first (filter #((second %) ua) rule-set))]
    (first rule)
    "other"))

(defn my-map [key value]
  (when-let [ua (get (json-decode value) "agent")]
    [[(get-type ua) 1]]))
```

`json-decode`函数是对`json/read-str`的包装，当JSON字符串无法正确解析时返回`nil`，而非异常终止。

`rule-set`是一个`map`类型，键是浏览器名称，值是一个函数，这里都是匿名函数。`partial`用于构造新的函数，`(partial + 1)`和`#(+ 1 %)`、`(fn [x] (+ 1 x))`是等价的，可以将其看做是为函数`+`的第一个参数定义了默认值。正则表达式中的`(?i)`表示匹配时不区分大小写。

`get-type`函数中，`(filter #((second %) ua) rule-set)`会用`rule-set`中的正则表达式逐一去和User-Agent字符串进行匹配，并返回第一个匹配项，也就是浏览器类型；没有匹配到的则返回`other`。

### 单元测试

我们可以编写一组单元测试来检验上述`my-map`函数是否正确：

```clojure
;; test/cia_hadoop/browser_test.clj

(ns cia-hadoop.browser-test
  (:use clojure.test
        clojure-hadoop.job
        cia-hadoop.browser))

(deftest test-my-map
  (is (= [["ie" 1]] (my-map 0 "{\"agent\":\"MSIE 6.0\"}")))
  (is (= [["chrome" 1]] (my-map 0 "{\"agent\":\"Chrome/20.0 Safari/6533.2\"}")))
  (is (= [["other" 1]] (my-map 0 "{\"agent\":\"abc\"}")))
  (is (nil? (my-map 0 "{"))))

(deftest test-browser
  (is (run job)))
```

其中`deftest`和`is`都是`clojure.test`命名空间下定义的。

```bash
$ lein test cia-hadoop.browser-test
```

小结
----

本章我们简单介绍了Hadoop这一用于大数据处理的开源项目，以及如何借助clojure-hadoop类库编写MapReduce脚本，并在本地和集群上运行。Hadoop已经将大数据处理背后的种种细节都包装了起来，用户只需编写Map和Reduce函数，而借助Clojure语言，这一步也变的更为轻松和高效。Apache Hadoop是一个生态圈，其周边有很多开源项目，像Hive、HBase等，这里再推荐一个使用Clojure语言在Hadoop上执行查询的工具：[cascalog](https://github.com/nathanmarz/cascalog)。它的作者是[Nathan Marz](http://nathanmarz.com/)，也是我们下一章的主题——Storm实时计算框架——的作者。

本文涉及到的源码可以到 https://github.com/jizhang/blog-demo/tree/master/cia-hadoop 中查看。
