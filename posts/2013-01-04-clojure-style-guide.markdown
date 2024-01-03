---
layout: post
title: "Clojure 代码规范"
date: 2013-01-04 20:49
comments: true
categories: Programming
tags: [clojure, translation]
published: true
---

原文地址：https://github.com/bbatsov/clojure-style-guide

这份Clojure代码规范旨在提供一系列的最佳实践，让现实工作中的Clojure程序员能够写出易于维护的代码，并能与他人协作和共享。一份反应真实需求的代码规范才能被人接收，而那些理想化的、甚至部分观点遭到程序员拒绝的代码规范注定不会长久——无论它有多出色。

这份规范由多个章节组成，每个章节包含一组相关的规则。我会尝试去描述每条规则背后的理念（过于明显的理念我就省略了）。

这些规则并不是我凭空想象的，它们出自于我作为一个专业软件开发工程师长久以来的工作积累，以及Clojure社区成员们的反馈和建议，还有各种广为流传的Clojure编程学习资源，如[《Clojure Programming》](http://www.clojurebook.com/)、[《The Joy of Clojure》](http://joyofclojure.com/)等。

这份规范还处于编写阶段，部分章节有所缺失，内容并不完整；部分规则没有示例，或者示例还不能完全将其描述清楚。未来这些问题都会得到改进，只是请你了解这一情况。

<!-- more -->

你可以使用[Transmuter](https://github.com/TechnoGate/transmuter)生成一份本规范的PDF或HTML格式的文档。

## 目录

* [源代码的布局和组织结构](#source-code-layout--organization)
* [语法](#syntax)
* [命名](#naming)
* [注释](#comments)
    * [注释中的标识](#comment-annotations)
* [异常](#exceptions)
* [集合](#collections)
* [可变量](#mutation)
* [字符串](#strings)
* [正则表达式](#regular-expressions)
* [宏](#macros)
* [惯用法](#existential)

## <a name="source-code-layout--organization"></a>源代码的布局和组织结构

> 几乎所有人都认为任何代码风格都是丑陋且难以阅读的，除了自己的之外。把这句话中的“除了自己之外”去掉，那差不多就能成立了。
> —— Jerry Coffin 关于代码缩进的评论

* 使用两个 **空格** 进行缩进，不使用制表符。

```clojure
    ;; 正确
    (when something
      (something-else))

    ;; 错误 - 四个空格
    (when something
        (something-else))
```

* 纵向对齐函数参数。

```clojure
    ;; 正确
    (filter even?
            (range 1 10))

    ;; 错误
    (filter even?
      (range 1 10))
```

* 对齐let绑定，以及map类型中的关键字。

```clojure
    ;; 正确
    (let [thing1 "some stuff"
          thing2 "other stuff"]
      {:thing1 thing1
       :thing2 thing2})

    ;; 错误
    (let [thing1 "some stuff"
      thing2 "other stuff"]
      {:thing1 thing1
      :thing2 thing2})
```

* 当`defn`没有文档字符串时，可以选择省略函数名和参数列表之间的空行。

```clojure
    ;; 正确
    (defn foo
      [x]
      (bar x))

    ;; 正确
    (defn foo [x]
      (bar x))

    ;; 错误
    (defn foo
      [x] (bar x))
```

* 当函数体较简短时，可以选择忽略参数列表和函数体之间的空行。

```clojure
    ;; 正确
    (defn foo [x]
      (bar x))

    ;; 适合简单的函数
    (defn goo [x] (bar x))

    ;; 适合包含多种参数列表的函数
    (defn foo
      ([x] (bar x))
      ([x y]
        (if (predicate? x)
          (bar x)
          (baz x))))

    ;; 错误
    (defn foo
      [x] (if (predicate? x)
            (bar x)
            (baz x)))
```

* 跨行的文档说明字符串每行都要缩进。

```clojure
    ;; 正确
    (defn foo
      "Hello there. This is
      a multi-line docstring."
      []
      (bar))

    ;; 错误
    (defn foo
      "Hello there. This is
    a multi-line docstring."
      []
      (bar))
```
* 使用Unix风格的换行符（\*BSD、Solaris、Linux、OSX用户无需设置，Windows用户则需要格外注意了）
    * 如果你使用Git，为了防止项目中意外引入Windows风格的换行符，不妨添加如下设置：

```bash
        $ git config --global core.autocrlf true
```

* 在括号`(`、`{`、`[`、`]`、`}`、`)`的外部添加空格，括号内部不要添加。

```clojure
    ;; 正确
    (foo (bar baz) quux)

    ;; 错误
    (foo(bar baz)quux)
    (foo ( bar baz ) quux)
```

* 避免在集合中使用逗号分隔符。

```clojure
    ;; 正确
    [1 2 3]
    (1 2 3)

    ;; 错误
    [1, 2, 3]
    (1, 2, 3)
```

* 可以考虑在map中适当使用逗号和换行以增强可读性。

```clojure
    ;; 正确
    {:name "Bruce Wayne" :alter-ego "Batman"}

    ;; 正确，且会增强可读性
    {:name "Bruce Wayne"
     :alter-ego "Batman"}

    ;; 正确，且较为紧凑
    {:name "Bruce Wayne", :alter-ego "Batman"}
```

* 将所有的反括号放在一行中。

```clojure
    ;; 正确
    (when something
      (something-else))

    ;; 错误
    (when something
      (something-else)
    )
```

* 顶层函数之间空出一行。

```clojure
    ;; 正确
    (def x ...)

    (defn foo ...)

    ;; 错误
    (def x ...)
    (defn foo ...)
```

* 函数或宏的定义体中不要添加空行。
* 每行尽量不超过80个字符。
* 避免在行末输入多余的空格。
* 为每个命名空间创建单独的文件。
* 使用一个完整的`ns`指令来声明命名空间，其包含`import`、`require`、`refer`、以及`use`。

```clojure
    (ns examples.ns
      (:refer-clojure :exclude [next replace remove])
      (:require (clojure [string :as string]
                         [set :as set])
                [clojure.java.shell :as sh])
      (:use (clojure zip xml))
      (:import java.util.Date
               java.text.SimpleDateFormat
               (java.util.concurrent Executors
                                     LinkedBlockingQueue)))
```

* 避免使用只有一个元素的命名空间名。

```clojure
    ;; 正确
    (ns example.ns)

    ;; 错误
    (ns example)
```

* 避免使用过长的命名空间（不超过五个元素）。

* 一个函数不应超过10行代码。事实上，大多数函数应保持在5行代码以内。

* 函数的参数个数不应超过三到四个。

## <a name="syntax"></a>语法

* 避免使用`require`、`refer`等改变命名空间的函数，它们只应在REPL中使用。
* 使用`declare`实现引用传递。
* 优先使用`map`这类高阶函数，而非`loop/recur`。

* 优先使用前置、后置条件来检测函数参数和返回值：

```clojure
    ;; 正确
    (defn foo [x]
      {:pre [(pos? x)]}
      (bar x))

    ;; 错误
    (defn foo [x]
      (if (pos? x)
        (bar x)
        (throw (IllegalArgumentException "x must be a positive number!")))
```

* 不要在函数中定义变量：

```clojure
    ;; 非常糟糕
    (defn foo []
      (def x 5)
      ...)
```

* 本地变量名不应覆盖`clojure.core`中定义的函数：

```clojure
    ;; 错误——这样一来函数中调用`map`时就需要指定完整的命名空间了。
    (defn foo [map]
      ...)
```

* 使用`seq`来判断一个序列是否为空（空序列等价于nil）。

```clojure
    ;; 正确
    (defn print-seq [s]
      (when (seq s)
        (prn (first s))
        (recur (rest s))))

    ;; 错误
    (defn print-seq [s]
      (when-not (empty? s)
        (prn (first s))
        (recur (rest s))))
```

* 使用`when`替代`(if ... (do ...)`。

```clojure
    ;; 正确
    (when pred
      (foo)
      (bar))

    ;; 错误
    (if pred
      (do
        (foo)
        (bar)))
```

* 使用`if-let`替代`let` + `if`。

```clojure
    ;; 正确
    (if-let [result :foo]
      (something-with result)
      (something-else))

    ;; 错误
    (let [result :foo]
      (if result
        (something-with result)
        (something-else)))
```

* 使用`when-let`替代`let` + `when`。

```clojure
    ;; 正确
    (when-let [result :foo]
      (do-something-with result)
      (do-something-more-with result))

    ;; 错误
    (let [result :foo]
      (when result
        (do-something-with result)
        (do-something-more-with result)))
```

* 使用`if-not`替代`(if (not ...) ...)`。

```clojure
    ;; 正确
    (if-not (pred)
      (foo))

    ;; 错误
    (if (not pred)
      (foo))
```

* 使用`when-not`替代`(when (not ...) ...)`。

```clojure
    ;; 正确
    (when-not pred
      (foo)
      (bar))

    ;; 错误
    (when (not pred)
      (foo)
      (bar))
```

* 使用`not=`替代`(not (= ...))`。

```clojure
    ;; 正确
    (not= foo bar)

    ;; 错误
    (not (= foo bar))
```

* 当匿名函数只有一个参数时，优先使用`%`，而非`%1`。

```clojure
    ;; 正确
    #(Math/round %)

    ;; 错误
    #(Math/round %1)
```

* 当匿名函数有多个参数时，优先使用`%1`，而非`%`。

```clojure
    ;; 正确
    #(Math/pow %1 %2)

    ;; 错误
    #(Math/pow % %2)
```

* 只有在必要的时候才使用匿名函数。

```clojure
    ;; 正确
    (filter even? (range 1 10))

    ;; 错误
    (filter #(even? %) (range 1 10))
```

* 当匿名函数包含多行语句时，使用`fn`来定义，而非`#(do ...)`。

```clojure
    ;; 正确
    (fn [x]
      (println x)
      (* x 2))

    ;; 错误（你不得不使用`do`）
    #(do (println %)
         (* % 2))
```

* 在特定情况下优先使用`complement`，而非匿名函数。

```clojure
    ;; 正确
    (filter (complement some-pred?) coll)

    ;; 错误
    (filter #(not (some-pred? %)) coll)
```

当函数已存在对应的求反函数时，则应使用该求反函数（如`even?`和`odd?`）。

* 某些情况下可以用`comp`使代码更简洁。

```clojure
    ;; 正确
    (map #(capitalize (trim %)) ["top " " test "])

    ;; 更好
    (map (comp capitalize trim) ["top " " test "])
```

* 某些情况下可以用`partial`使代码更简洁。

```clojure
    ;; 正确
    (map #(+ 5 %) (range 1 10))

    ;; 或许更好
    (map (partial + 5) (range 1 10))
```

* 当遇到嵌套调用时，建议使用`->`宏和`->>`宏。

```clojure
    ;; 正确
    (-> [1 2 3]
        reverse
        (conj 4)
        prn)

    ;; 不够好
    (prn (conj (reverse [1 2 3])
               4))

    ;; 正确
    (->> (range 1 10)
         (filter even?)
         (map (partial * 2)))

    ;; 不够好
    (map (partial * 2)
         (filter even? (range 1 10)))
```

* 当需要连续调用Java类的方法时，优先使用`..`，而非`->`。

```clojure
    ;; 正确
    (-> (System/getProperties) (.get "os.name"))

    ;; 更好
    (.. System getProperties (get "os.name"))
```

* 在`cond`和`condp`中，使用`:else`来处理不满足条件的情况。

```clojure
    ;; 正确
    (cond
      (< n 0) "negative"
      (> n 0) "positive"
      :else "zero"))

    ;; 错误
    (cond
      (< n 0) "negative"
      (> n 0) "positive"
      true "zero"))
```

* 当比较的变量和方式相同时，优先使用`condp`，而非`cond`。

```clojure
    ;; 正确
    (cond
      (= x 10) :ten
      (= x 20) :twenty
      (= x 30) :forty
      :else :dunno)

    ;; 更好
    (condp = x
      10 :ten
      20 :twenty
      30 :forty
      :dunno)
```

* 当条件是常量时，优先使用`case`，而非`cond`或`condp`。

```clojure
    ;; 正确
    (cond
      (= x 10) :ten
      (= x 20) :twenty
      (= x 30) :forty
      :else :dunno)

    ;; 更好
    (condp = x
      10 :ten
      20 :twenty
      30 :forty
      :dunno)

    ;; 最佳
    (case x
      10 :ten
      20 :twenty
      30 :forty
      :dunno)
```

* 某些情况下，使用`set`作为判断条件。

```clojure
    ;; 错误
    (remove #(= % 0) [0 1 2 3 4 5])

    ;; 正确
    (remove #{0} [0 1 2 3 4 5])

    ;; 错误
    (count (filter #(or (= % \a)
                        (= % \e)
                        (= % \i)
                        (= % \o)
                        (= % \u))
                   "mary had a little lamb"))

    ;; 正确
    (count (filter #{\a \e \i \o \u} "mary had a little lamb"))
```

* 使用`(inc x)`和`(dec x)`替代`(+ x 1)`和`(- x 1)`。

* 使用`(pos? x)`、`(neg? x)`、以及`(zero? x)`替代`(> x 0)`、`(< x 0)`、和`(= x 0)`。

* 进行Java操作时，优先使用Clojure提供的语法糖。

```clojure
    ;;; 创建对象
    ;; 正确
    (java.util.ArrayList. 100)

    ;; 错误
    (new java.util.ArrayList 100)

    ;;; 调用静态方法
    ;; 正确
    (Math/pow 2 10)

    ;; 错误
    (. Math pow 2 10)

    ;;; 调用实例方法
    ;; 正确
    (.substring "hello" 1 3)

    ;; 错误
    (. "hello" substring 1 3)

    ;;; 访问静态属性
    ;; 正确
    Integer/MAX_VALUE

    ;; 错误
    (. Integer MAX_VALUE)

    ;;; 访问实例属性
    ;; 正确
    (.someField some-object)

    ;; 错误
    (. some-object some-field)
```

## <a name="naming"></a>命名

> 编程中真正的难点只有两个：验证缓存的有效性；命名。<br/>
> —— Phil Karlton

* 命名空间建议使用以下两种方式：
    * `项目名称.模块名称`
    * `组织名称.项目名称.模块名称`
* 对于命名空间中较长的元素，使用`lisp-case`格式，如`bruce.project-euler`。
* 使用`lisp-case`格式来命名函数和变量。
* 使用`CamelCase`来命名接口（protocol）、记录（record）、结构和类型（struct & type）。对于HTTP、RFC、XML等缩写，仍保留其大写格式。
* 对于返回布尔值的函数名称，使用问号结尾，如`even?`。
* 当方法或宏不能在STM中安全使用时，须以感叹号结尾，如`reset!`。
* 命名类型转换函数时使用`->`，而非`to`。

```clojure
    ;; 正确
    (defn f->c ...)

    ;; 不够好
    (defn f-to-c ...)
```

* 对于可供重绑定的变量（即动态变量），使用星号括起，如`*earmuffs*`。
* 无需对常量名进行特殊的标识，因为所有的变量都应该是常量，除非有特别说明。
* 对于解构过程中或参数列表中忽略的元素，使用`_`来表示。
* 参考`clojure.core`中的命名规范，如`pred`、`coll`：
    * 函数：
        * `f`、`g`、`h`：参数内容是一个函数
        * `n`：整数，通常是一个表示大小的值
        * `index`：整数索引
        * `x`、`y`：数值
        * `s`：字符串
        * `coll`：集合
        * `pred`：断言型的闭包
        * `& more`：可变参数
    * 宏：
        * `expr`：表达式
        * `body`：语句
        * `binding`：一个向量，包含宏的绑定

## <a name="collections"></a>集合

> 用100种函数去操作同一种数据结构，要好过用10种函数操作10种数据结构。<br/>
> —— Alan J. Perlis

* 避免使用列表（list）来存储数据（除非它真的就是你想要的）。
* 优先使用关键字（keyword），而非普通的哈希键：

```clojure
    ;; 正确
    {:name "Bruce" :age 30}

    ;; 错误
    {"name" "Bruce" "age" 30}
```

* 编写集合时，优先使用内置的语法形式，而非构造函数。但是，在定义唯一值集合（set）时，只有当元素都是常量时才可使用内置语法，否则应使用构造函数，如下所示：

```clojure
    ;; 正确
    [1 2 3]
    #{1 2 3}
    (hash-set (func1) (func2)) ; 元素在运行时确定

    ;; bad
    (vector 1 2 3)
    (hash-set 1 2 3)
    #{(func1) (func2)} ; 若(func1)和(func2)的值相等，则会抛出运行时异常。
```

* 避免使用数值索引来访问集合元素。

* 优先使用关键字来获取哈希表（map）中的值。

```clojure
    (def m {:name "Bruce" :age 30})

    ;; 正确
    (:name m)

    ;; 错误——太过啰嗦
    (get m :name)

    ;; 错误——可能抛出空指针异常
    (m :name)
```

* 集合可以被用作函数：

```clojure
    ;; 正确
    (filter #{\a \e \o \i \u} "this is a test")

    ;; 缺点——不够美观
```

* 关键字可以被用作函数：

```clojure
    ((juxt :a :b) {:a "ala" :b "bala"})
```

* 只有在非常强调性能的情况下才可使用瞬时集合（transient collection）。

* 避免使用Java集合。

* 避免使用Java数组，除非遇到需要和Java类进行交互，或需要高性能地处理基本类型时才可使用。

## <a name="mutation"></a>可变量

### 引用（Refs）

* 建议所有的IO操作都使用`io!`宏进行包装，以免不小心在事务中调用了这些代码。
* 避免使用`ref-set`。
* 控制事务的大小，即事务所执行的工作越少越好。
* 避免出现短期事务和长期事务访问同一个引用（Ref）的情形。

### 代理（Agents）

* `send`仅使用于计算密集型、不会因IO等因素阻塞的线程。
* `send-off`则用于会阻塞、休眠的线程。

### 原子（Atoms）

* 避免在事务中更新原子。
* 避免使用`reset!`。

## <a name="strings"></a>字符串

* 优先使用`clojure.string`中提供的字符串操作函数，而不是Java中提供的或是自己编写的函数。

```clojure
    ;; 正确
    (clojure.string/upper-case "bruce")

    ;; 错误
    (.toUpperCase "bruce")
```

## <a name="exceptions"></a>异常

* 复用已有的异常类型，如：
    * `java.lang.IllegalArgumentException`
    * `java.lang.UnsupportedOperationException`
    * `java.lang.IllegalStateException`
    * `java.io.IOException`
* 优先使用`with-open`，而非`finally`。

## <a name="macros"></a>宏

* 如果可以用函数实现相同功能，不要编写一个宏。
* 首先编写一个宏的用例，尔后再编写宏本身。
* 尽可能将一个复杂的宏拆解为多个小型的函数。
* 宏只应用于简化语法，其核心应该是一个普通的函数。
* 使用语法转义（syntax-quote，即反引号），而非手动构造`list`。

## <a name="comments"></a>注释

> 好的代码本身就是文档。因此在添加注释之前，先想想自己该如何改进代码，让它更容易理解。做到这一点后，再通过注释让代码更清晰。<br/>
> ——Steve McConnel

* 学会编写容易理解的代码，然后忽略下文的内容。真的！

* 对于标题型的注释，使用至少四个分号起始。

* 对于顶层注释，使用三个分号起始。

* 为某段代码添加注释时，使用两个分号起始，且应与该段代码对齐。

* 对于行尾注释，使用一个分号起始即可。

* 分号后面要有一个空格。

```clojure
    ;;;; Frob Grovel

    ;;; 这段代码有以下前提：
    ;;;   1. Foo.
    ;;;   2. Bar.
    ;;;   3. Baz.

    (defn fnord [zarquon]
      ;; If zob, then veeblefitz.
      (quux zot
            mumble             ; Zibblefrotz.
            frotz))
```

* 对于成句的注释，句首字母应该大写，[句与句之间用一个空格分隔](http://en.wikipedia.org/wiki/Sentence_spacing)。
* 避免冗余的注释：

```clojure
    ;; 错误
    (inc counter) ; counter变量的值加1
```

* 注释要和代码同步更新。过期的注释还不如没有注释。
* 有时，使用`#_`宏要优于普通的注释：

```clojure
    ;; 正确
    (+ foo #_(bar x) delta)

    ;; 错误
    (+ foo
       ;; (bar x)
       delta)
```

> 好的代码和好的笑话一样，不需要额外的解释。
> ——Russ Olsen

* 避免使用注释去描述一段写得很糟糕的代码。重构它，让它更为可读。（做或者不做，没有尝试这一说。——Yoda）

### <a name="comment-annotations"></a>注释中的标识

* 标识应该写在对应代码的上一行。
* 标识后面是一个冒号和一个空格，以及一段描述文字。
* 如果标识的描述文字超过一行，则第二行需要进行缩进。
* 将自己姓名的首字母以及当前日期附加到标识描述文字中：

```clojure
    (defn some-fun
      []
      ;; FIXME: 这段代码在v1.2.3之后偶尔会崩溃，
      ;;        这可能和升级BarBazUtil有关。（xz 13-1-31）
      (baz))
```

* 对于功能非常明显，实在无需添加注释的情况，可以在行尾添加一个标识：

```clojure
    (defn bar
      []
      (sleep 100)) ; OPTIMIZE
```

* 使用`TODO`来表示需要后期添加的功能或特性。
* 使用`FIXME`来表示需要修复的问题。
* 使用`OPTIMIZE`来表示会引起性能问题的代码，并需要修复。
* 使用`HACK`来表示这段代码并不正规，需要在后期进行重构。
* 使用`REVIEW`来表示需要进一步审查这段代码，如：`REVIEW: 你确定客户会正确地操作X吗？`
* 可以使用其它你认为合适的标识关键字，但记得一定要在项目的`README`文件中描述这些自定义的标识。

## <a name="existential"></a>惯用法

* 使用函数式风格进行编程，避免改变变量的值。
* 保持编码风格。
* 用正常人的思维来思考。

# 贡献

本文中的所有内容都还没有最后定型，我很希望能够和所有对Clojure代码规范感兴趣的同仁一起编写此文，从而形成一份对社区有益的文档。

你可以随时创建讨论话题，或发送合并申请。我在这里提前表示感谢。

# 宣传

一份由社区驱动的代码规范如果得不到社区本身的支持和认同，那它就毫无意义了。发送一条推特，向朋友和同事介绍此文。任何评论、建议、以及意见都能够让我们向前迈进一小步。请让我们共同努力吧！
