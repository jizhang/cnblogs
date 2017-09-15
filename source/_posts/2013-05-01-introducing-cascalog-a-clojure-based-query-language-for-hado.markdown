---
layout: post
title: "Cascalog：基于 Clojure 的 Hadoop 查询语言"
date: 2013-05-01 18:01
comments: true
categories: [Big Data]
tags: [clojure, translation]
published: true
---

原文：[http://nathanmarz.com/blog/introducing-cascalog-a-clojure-based-query-language-for-hado.html](http://nathanmarz.com/blog/introducing-cascalog-a-clojure-based-query-language-for-hado.html)

我非常兴奋地告诉大家，[Cascalog](http://github.com/nathanmarz/cascalog)开源了！Cascalog受[Datalog](http://en.wikipedia.org/wiki/Datalog)启发，是一种基于Clojure、运行于Hadoop平台上的查询语言。

## 特点

* **简单** - 使用相同的语法编写函数、过滤规则、聚合运算；数据联合（join）变得简单而自然。
* **表达能力强** - 强大的逻辑组合条件，你可以在查询语句中任意编写Clojure函数。
* **交互性** - 可以在Clojure REPL中执行查询语句。
* **可扩展** - Cascalog的查询语句是一组MapReduce脚本。
* **任意数据源** - HDFS、数据库、本地数据、以及任何能够使用Cascading的`Tap`读取的数据。
* **正确处理空值** - 空值往往让事情变得棘手。Cascalog提供了内置的“非空变量”来自动过滤空值。
* **与Cascading结合** - 使用Cascalog定义的流程可以在Cascading中直接使用，反之亦然。
* **与Clojure结合** - 能够使用普通的Clojure函数来编写操作流程、过滤规则，又因为Cascalog是一种Clojure DSL，因此也能在其他Clojure代码中使用。

<!--more-->

好，下面就让我们开始Cascalog的学习之旅！我会用一系列的示例来介绍Cascalog。这些示例会使用到项目本身提供的“试验场”数据集。我建议你立刻下载Cascalog，一边阅读本文一边在REPL中操作。（安装启动过程只有几分钟，README中有步骤）

## 基本查询

首先让我们启动REPL，并加载“试验场”数据集：

```clojure
lein repl
user=> (use 'cascalog.playground) (bootstrap)
```

以上语句会加载本文用到的所有模块和数据。你可以阅读项目中的`playground.clj`文件来查看这些数据。下面让我们执行第一个查询语句，找出年龄为25岁的人：

```clojure
user=> (?<- (stdout) [?person] (age ?person 25))
```

这条查询语句可以这样阅读：找出所有`age`等于25的`?person`。执行过程中你可以看到Hadoop输出的日志信息，几秒钟后就能看到查询结果。

好，让我们尝试稍复杂的例子。我们来做一个范围查询，找出年龄小于30的人：

```clojure
user=> (?<- (stdout) [?person] (age ?person ?age) (< ?age 30))
```

看起来也不复杂。这条语句中，我们将人的年龄绑定到了`?age`变量中，并对该变量做出了“小于30”的限定。

我们重新执行这条语句，只是这次会将人的年龄也输出出来：

```clojure
user=> (?<- (stdout) [?person ?age] (age ?person ?age)
            (< ?age 30))
```

我们要做的仅仅是将`?age`添加到向量中去。

让我们执行另一条查询，找出艾米丽关注的所有男性：

```clojure
user=> (?<- (stdout) [?person] (follows "emily" ?person)
            (gender ?person "m"))
```

可能你没有注意到，这条语句使用了联合查询。各个数据集中的`?person`值都必须对应，而`follows`和`gender`分属于不同的数据集，Cascalog便会使用联合查询。

## 查询语句的结构

让我们分析一下查询语句的结构，以下面这条语句为例：

```clojure
user=> (?<- [stdout] [?person ?a2] (age ?person ?age)
            (< ?age 30) (* 2 ?age :> ?a2))
```

`?<-`操作符出现的频率很高，它能同时定义并执行一条查询。`?<-`实际上是对`<-`和`?-`的包装。我们之后会看到如何使用这些操作符编写更为复杂的查询语句。

首先，我们指定了查询结果的输出目的地，就是这里的`(stdout)`。`(stdout)`会创建一个Cascading的`tap`组件，它会在查询结束后将结果打印到标准输出中。我们可以使用任意一种Cascading的`tap`组件，也就是说输出结果的格式可以是序列文件（Sequence file）、文本文件等等；也可以输出到任何地方，如本地磁盘、HDFS、数据库等。

在定义了输出目的地后，我们使用Clojure的向量结构来定义输出结果所包含的内容。本例中，我们定义的是`?person`和`?a2`。

接下来，我们定义了一系列的约束条件。Cascalog有三种约束条件：

1. 生成器（Generator）：表示一个数据源，可以是以下两种类型：
    * Cascading Tap：如HDFS上某个路径中的文件；
    * 一个已经使用`<-`定义的查询。
2. 操作器（Operation）：引入预定义的变量，将其绑定至新的变量，或是设定一个过滤条件。
3. 集合器（Aggregator）：计数、求和、最小值、最大值等等。

约束条件由名称、一组输入变量、以及一组输出变量构成。上述查询中的约束条件有：

* (age ?person ?age)
* (< ?age 30)
* (* 2 ?age :> ?a2)

其中，`:>`关键字用于将输入变量和输出变量隔开。如果没有这个关键字，那么该变量在操作器中就会被识别为输入变量，在生成器和集合器中会被认为是输出变量。

`age`约束指向`playground.clj`中定义的一个`tap`，所以它是一个生成器，会输出`?person`和`?age`这两个数据。

`<`约束是一个Clojure函数，因为没有指定输出变量，所以这条约束会构成一个过滤器，将`?age`小于30的记录筛选出来。如果我们这样写：

```clojure
(< ?age 30 :> ?young)
```

那么`<`约束会将“年龄是否小于30”作为一个布尔值传递给`?young`变量。

约束之间的顺序不重要，因为Cascalog是声明式语言。

## 变量替换为常量

变量是以`?`或`!`起始的标识。有时你不在意变量的值，可以直接用`_`代替。其他的变量则会在解析时替换成常量。我们已经在很多示例中用到这一特性了。下面这个示例中，我们将输出变量作为一种过滤条件：

```clojure
(* 4 ?v2 :> 100)
```

这里使用了两个常量：4和100。4是一个输入变量，100则是作为一个过滤条件，只有满足`?v2`乘以4等于100的记录才会被筛选出来。字符串、数字、以及其他基本类型和对象类型，只要在Hadoop有对应的序列化操作，都可以被作为常量使用。

让我们回到示例中。找出所有关注了比自己年龄小的用户的列表：

```clojure
user=> (?<- (stdout) [?person1 ?person2]
            (age ?person1 ?age1) (follows ?person1 ?person2)
            (age ?person2 ?age2) (< ?age2 ?age1))
```

同时，我们将年龄差异也输出出来：

```clojure
user=> (?<- (stdout) [?person1 ?person2 ?delta]
            (age ?person1 ?age1) (follows ?person1 ?person2)
            (age ?person2 ?age2) (- ?age2 ?age1 :> ?delta)
            (< ?delta 0))
```

## 聚合

下面让我们看看聚合查询的使用方法。统计所有年龄小于30的用户人数：

```clojure
user=> (?<- (stdout) [?count] (age _ ?a) (< ?a 30)
            (c/count ?count))
```

这条查询会统计所有的记录。我们也可以只聚合部分记录。比如，让我们找出每个人所关注的用户的数量：

```clojure
user=> (?<- (stdout) [?person ?count] (follows ?person _)
            (c/count ?count))
```

因为我们在输出结果中指定了`?person`这个变量，所以Cascalog会将数据记录按照用户来分组，然后使用`c/count`进行聚合运算。

你可以在单个查询中使用多个聚合条件，它们的分组方式是一致的。例如，我们可以计算每个国家的用户的平均年龄，使用计数和求和这两种聚合方式：

```clojure
user=> (?<- (stdout) [?country ?avg]
            (location ?person ?country _ _) (age ?person ?age)
            (c/count ?count) (c/sum ?age :> ?sum)
            (div ?sum ?count :> ?avg))
```

可以看到，我们对`?sum`和`?count`这两个聚合结果执行了`div`操作，该操作会在聚合过程结束后进行。

## 自定义操作

下面我们来编写一个查询，统计几句话中每个单词的出现次数。首先，我们编写一个自定义操作：

```clojure
user=> (defmapcatop split [sentence]
         (seq (.split sentence "\\s+")))
user=> (?<- (stdout) [?word ?count] (sentence ?s)
            (split ?s :> ?word) (c/count ?count))
```

`defmapcatop split`定义了一个方法，这个方法接收一个参数`sentence`，并会输出0个或多个元组（tuple）。`deffilterop`可以用来定义一个返回布尔型的方法，用来筛选记录；`defmapop`定义的函数会返回一个元组；`defaggregateop`定义一个聚合函数。这些函数都能在Cascalog工作流API中使用，我会在另一篇博客中叙述。

在上述查询中，如果单词字母大小写不一致，会被分别统计。我们用以下方法来修复这个问题：

```clojure
user=> (defn lowercase [w] (.toLowerCase w))
user=> (?<- (stdout) [?word ?count]
            (sentence ?s) (split ?s :> ?word1)
            (lowercase ?word1 :> ?word) (c/count ?count))
```

可以看到，这里直接使用了纯Clojure编写的函数。当这个函数不包含输出变量时，会被作为过滤条件来执行；当包含一个返回值时，则会作为`defmapop`来解析。而对于返回0个或多个元组的函数，则必须使用`defmapcatop`来定义。

下面这个查询会按照性别和年龄范围来统计用户数量：

```clojure
user=> (defn agebucket [age]
         (find-first (partial <= age) [17 25 35 45 55 65 100 200]))
user=> (?<- (stdout) [?bucket ?gender ?count]
            (age ?person ?age) (gender ?person ?gender)
            (agebucket ?age :> ?bucket) (c/count ?count))
```

## 非空变量

Cascalog提供了“非空变量”这样的机制来帮助用户处理空值的情况。其实我们每个示例中都在使用这一特性。以`?`开头的变量都是非空变量，而以`!`开头的则是可空变量。Cascalog会在执行过程中将空值排除在外。

为了体验非空变量的效果，让我们对比下面这两条查询语句：

```clojure
user=> (?<- (stdout) [?person ?city] (location ?person _ _ ?city)
user=> (?<- (stdout) [?person !city] (location ?person _ _ !city)
```

第二组查询结果中会包含空值。

## 子查询

最后，我们来看看更为复杂的查询，我们会用到子查询这一特性。让我们找出关注了两人以上的用户列表，并找出这些用户之间的关注关系：

```clojure
user=> (let [many-follows (<- [?person] (follows ?person _)
                              (c/count ?c) (> ?c 2))]
            (?<- (stdout) [?person1 ?person2] (many-follows ?person1)
                 (many-follows ?person2) (follows ?person1 ?person2)))
```

这里，我们使用`let`来定义了一个子查询`many-follows`。这个子查询是用`<-`定义的。之后，我们便可以在后续查询中使用这个子查询了。

我们还可以在一个查询中指定多个输出目的地。比如我们想要同时得到`many-follows`的查询结果：

```clojure
user=> (let [many-follows (<- [?person] (follows ?person _)
                              (c/count ?c) (> ?c 2))
             active-follows (<- [?p1 ?p2] (many-follows ?p1)
                                (many-follows ?p2) (follows ?p1 ?p2))]
            (?- (stdout) many-follows (stdout) active-follows))
```

这里我们分别定义了两个查询，没有立刻执行它们，而是在后续的`?-`中将两个查询分别绑定到了两个`tap`上，并同时执行。

## 小结

Cascalog目前在还不断的改进中，未来会增加更多查询特性，以及对查询过程的优化。

我非常希望能够得到你对Cascalog的反馈，如果你有任何评论、问题、或是顾虑，请留言，或者在[Twitter](http://twitter.com/nathanmarz)上联系我，给我发送邮件[nathan.marz@gmail.com](nathan.marz@gmail.com)，或是在freenode的#cascading频道和我聊天。

[下一篇博客](http://nathanmarz.com/blog/new-cascalog-features)会介绍Cascalog的外联合、排序、组合等特性。
