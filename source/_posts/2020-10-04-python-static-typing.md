---
title: Python 类型检查实践
tags:
  - python
categories: Programming
date: 2020-10-04 19:11:18
---


Python 作为一门动态类型语言，代码灵活度和开发效率都是非常高的。但随着项目代码逐渐变多，函数之间的调用变得更复杂，经常会出现参数或返回值类型不正确等问题。并且这些问题只能在运行时被发现，甚至会产生线上 Bug。那么如何能让 Python 像 Java 或 Go 这样的语言一样，在编译期就进行类型检查呢？从 3.5 版本开始，Python 就能支持静态类型检查了。本文整理自团队的一次内部分享，介绍了 Python 的这一新特性。

<img src="/cnblogs/images/python-static-typing/python-static-typing.002.jpeg" width="60%" />

<!-- more -->

## Quick Start

首先我们来快速看一下，Python 中要如何使用静态类型检查。

<img src="/cnblogs/images/python-static-typing/python-static-typing.003.jpeg" width="60%" />

通过给函数的参数和返回值添加类型注解，配合使用 mypy 等工具，我们就可以对 Python 代码做静态类型检查了。上图中的 `greeting` 函数接收一个 `name` 参数，我们将其定义为 `str` 类型；函数的返回值也是 `str` 类型。使用 `pip install` 安装 mypy 后，就可以进行类型检查了。很显然，这个函数是能通过类型检查的，因为字符串之间可以进行 `+` 的操作。假设我们将 `name` 变量与一个整数相加，mypy 就会报告无法对 `str` 和 `int` 类型做 `+` 的操作；同样，如果调用 `greeting` 时传入的是整型，或者将 `greeting` 的返回值和整型相加，都会触发类型检查错误。

<img src="/cnblogs/images/python-static-typing/python-static-typing.004.jpeg" width="60%" />

除了对函数参数做类型注解，我们也可以对本地变量做注解，语法是类似的。上图中的 `number` 变量我们声明为 `int` 类型，在将字符串赋值给它的时候就会引发报错；同样，如果将 `number` 作为函数返回值，也会触发报错。

除了 `int` 和 `str` 这类基本类型，我们也可以对容器类进行注解。更进一步，我们可以去规定容器中可以存放什么类型的变量。图中的 `items` 变量被定义为了 `list` 类型，因此无法赋值一个整型。但是，`list` 类型没有规定容器中元素的类型，因此 `[1, 2, 3]` 和 `['a', 'b', 'c']` 都是可以赋值给 `items` 变量的。如果我们想定义一个只包含整型元素的 `list`，就需要使用 `typing` 模块提供的 `List` 类型。比如 `nums` 变量就是一个只包含整型元素的列表，如果我们想要将字符串加入这个列表中，就会引发类型检查错误；字典类型 `Dict` 也是类似的作用。需要注意的是，这些代码在执行时是不会报错的，因为 Python 仍然是动态类型语言，运行期不会进行类型检查，只有用 mypy 等工具去检查时才会有效。

<img src="/cnblogs/images/python-static-typing/python-static-typing.005.jpeg" width="60%" />

那是不是所有的变量都需要做类型注解呢？答案当然是否定的，因为 mypy 有类型推断机制。比如上图中的 `nums` 变量，虽然没有声明类型，但是因为赋值给它的是一个整型列表，所以该变量会被推断成 `List[int]` 类型，也就无法添加字符串类型的元素了。绕过这一机制的方式当然是显式地去指定类型，比如当指定为 `List[object]` 类型时，就可以写入字符串、整型等任意元素了。

最后我们看一下如何给类成员变量注解类型。两种方式：第一种是图中的 `suffix` 变量，在声明的时候进行注解；第二种是利用类型推断，比如构造函数中，`date` 参数传入的是 `str` 类型，并赋值给了 `self.date`，那么 `self.date` 就会被推断为 `str` 类型。这样一来，`run` 函数中的两条语句都会产生类型检查错误。

## 为什么要引入类型检查

我们可以看到，静态类型检查是会增加额外的工作量的，那么我们为什么要引入静态类型检查，它的优点有哪些呢？

<img src="/cnblogs/images/python-static-typing/python-static-typing.007.jpeg" width="60%" />

我之前在分享 *Learn Go in 10 Minutes* 的时候提到过，Go 语言的优点之一是其静态类型系统。静态类型可以在编译期就发现错误，而不用等到程序运行出错时才去修复；而且研究表明，静态类型系统是可以减少 Bug 的数量的，研究者使用的对象是 TypeScript 和 Flow，这两者都为 JavaScript 语言提供了类型检查机制；最后，静态类型系统对程序性能的提升也是有帮助的。

<img src="/cnblogs/images/python-static-typing/python-static-typing.008.jpeg" width="60%" />

先来梳理几个概念，我们一直会听到静态类型、动态类型、强类型、弱类型等术语。其中，静态类型和动态类型比较容易区分：前者在编译期进行类型检查，后者在运行期进行检查。像 Java、C、Golang 等是比较常见的静态类型语言，而 JavaScript、PHP、Python 等则是动态类型的。强类型和弱类型则比较难以区分了，判断标准是：允许隐式类型转换的程度。比如 JavaScript 是最弱的一门弱类型语言，因为任何类型的变量都可以进行相加等操作，执行引擎会自动做隐式转换；PHP 也是弱类型语言，不过在转换时会报一些 Warning 信息；Python 则是强类型语言，因为当我们将字符串和整型做相加操作时会直接报错。

<img src="/cnblogs/images/python-static-typing/python-static-typing.009.jpeg" width="60%" />

这里列举了静态类型语言的优点，我挑选几条重点讲一下。首先是提升代码可读性，可以看到，函数的参数类型被清楚地标注出来了，如果传入了类型不正确的参数，在编译阶段就会报错。以前我们调用函数时要通过阅读文档或代码来确定参数的类型，有了标注后就可以省去这一步。

<img src="/cnblogs/images/python-static-typing/python-static-typing.010.jpeg" width="60%" />

类型标注也可以增强 IDE 的代码提示功能。比如将 `value` 标注为字符串类型，就可以 `.` 出对应的方法了。同样的，对于函数的调用提示，IDE 也可以展示参数类型信息。Python 的标准库都已经增加了类型注解。

<img src="/cnblogs/images/python-static-typing/python-static-typing.011.jpeg" width="60%" />

最后是性能提升。首先说明一下，Python 的类型注解在运行期是不会被执行的，目前 Python 的解释器也没有使用类型信息对代码进行优化，所以这里我用 Clojure 作为例子。可以看到，对变量 `x` 进行类型注解后，运行速度得到了很大提升，因为 Clojure 也是动态类型语言，原本需要通过反射来获取变量的类型，有了注解后这一步就可以省去了，从而提升了速度。

当然，类型注解也不是没有缺点的。比如需要编写和维护额外的代码，而且变量有了类型之后，Python 的灵活性也会打一些折扣。所以一般来说，小项目或者平时写的简单脚本，是不需要用类型注解的；只有当项目达到一定规模，才需要考虑引入这个工具。

<img src="/cnblogs/images/python-static-typing/python-static-typing.012.jpeg" width="60%" />

其他动态类型语言也在引入静态类型检查。比如 JavaScript 对应的 TypeScript，以及 PHP 对应的 Hack 语言。它们的方式也和 Python 类似，类型注解是可选的，没有注解的变量不做检查，从而让开发人员可以平滑地切换到静态类型语言。

## Python 类型检查功能详解

<img src="/cnblogs/images/python-static-typing/python-static-typing.014.jpeg" width="60%" />

Python 语言引入新特性时都会有相应的 PEP，这里列举了一些和静态类型检查相关且比较重要的 PEP。可以看到，Python 3.0 开始引入了函数参数和返回值的类型注解，但当时并不是专门为静态类型检查设计的。直到 Python 3.5，我们才能正式使用静态类型检查，`typing` 模块也是从这个版本开始引入的。Python 3.6 则是增加了本地变量的类型注解语法，让类型检查功能更加完整。我们目前使用的也是 3.6 版本。3.7 开始类型注解会被延迟处理，一是解决性能问题，因为类型注解虽然在运行期不进行检查，但解释器还是会去解析；二是解决类型注解过早引用的问题，这点我们在介绍如何对类方法进行注解时会说明。Python 3.8 则引入了更多的高级特性，比如支持 Duck typing、定义 Dict 键值列表、以及 `final` 关键字等，这些内容我会在最后一部分做个简单介绍。

<img src="/cnblogs/images/python-static-typing/python-static-typing.015.jpeg" width="60%" />

Python 的类型检查是可选的，mypy 只会检查有类型标注的代码，这种做法我们称之为渐进式类型检查。没有标注类型的变量会被定义为 `Any` 类型，我们可以将任意类型的数据赋值给它，它也能够被当作任意类型使用。`Any` 和 `object` 是有区别的，虽然 `object` 也可以满足第一条，但它无法作为任意类型使用。比如例子中的 `item` 变量，因为它是 `object` 类型，所以调用 `bar()` 方法时就会报错，`Any` 则不会。需要额外注意的是，如果你的函数没有参数也没有返回值，一定要标注返回值为 `None`，否则 mypy 会认为这个函数没有做类型注解，从而跳过检查。

<img src="/cnblogs/images/python-static-typing/python-static-typing.016.jpeg" width="60%" />

类型标注有三种方式，除了前面一直使用的注解外，还可以通过注释和接口文件来进行。注释的方式主要是为了兼容 Python 2.x，因为低版本的 Python 没有引入注解语法，所以用注释来代替。`typing` 模块也有 2.x 的版本，需要单独安装。注解和注释的方式都需要修改源代码，而接口文件的方式就可以在无法修改源码的时候对代码进行注解，这点在使用第三方库时很有用，其实 Python 的标准库也用了相同的做法，我们下面会讲。接口文件是以 `.pyi` 结尾的，内容只有类和函数的定义，方法体则是用 `...` 做占位。接口文件可以和源码放在同一个目录中，也可以放在独立的目录中，通过配置参数告知 mypy 该去哪里加载。

<img src="/cnblogs/images/python-static-typing/python-static-typing.017.jpeg" width="60%" />

Python 官方维护了一个名为 typeshed 的项目，里面包含了 Python 标准库的接口文件，以及其他一些常用的三方库的类型注解，比如 Flask, requests 等。安装 mypy 时会自动安装 typeshed 项目。

<img src="/cnblogs/images/python-static-typing/python-static-typing.018.jpeg" width="60%" />

Python 中常用的类型包括 `int`、`str` 等基础类型，`List`、`Dict` 等容器类型，以及用来标注函数参数和返回值的 `Callback` 回调类型。当然，类本身也是一种类型，我们下一节会讲。此外还有一些特殊类型，比如 `Union[int, str]` 可以用来标注变量可以是整型或字符串类型；`Optional` 表示该变量可能为 `None`。这个类型在标注函数返回值时非常有用：用户必须写代码去判断函数的返回值是否为空，否则 mypy 会提示错误。

<img src="/cnblogs/images/python-static-typing/python-static-typing.019.jpeg" width="60%" />

类本身也是一种类型，mypy 能够识别类的继承关系，类也能在容器中使用，比如上图中的 `cats` 变量就是一个元素为 `Cat` 的数组。如果我们觉得类型定义过长，可以给它起一个别名，`CatList` 和 `List[Cat]` 是等价的。对于一些静态方法，比如工厂模式需要返回类本身，定义时类本身还没有被定义完全，因此会报类不存在的错误。这就是之前提到的过早引用问题，解决方法是将类型注解表达成字符串。最后一个例子是 Python 3.8 引入的 `TypedDict`，变量本身是一个普通的 `dict`，但通过注解就可以对键值列表进行类型检查了。

以上就是 Python 类型检查最基础的知识了，有了这些模块和工具，我们就可以在项目中立刻用起来了，也不用担心会对已有代码产生破坏。

## 进阶知识

最后我们来看一些类型检查的高级特性，这些特性在官方文档中有详细说明，这里我只是简单介绍一下，而且实际项目中也不太会用到。

<img src="/cnblogs/images/python-static-typing/python-static-typing.021.jpeg" width="60%" />

当我们自己定义一个容器类的时候，就需要用到泛型，因为我们不关心容器里装的是什么。比如示例中的 `LoggedVar` 类被定义为可以接收任意类型的变量，在变量被修改时打出一行日志。此外，泛型是可以设置下界的，意思是传入的类型必须是某个类型的子类。比如示例中，`longer` 函数要求传入的变量必须是 `Sized` 的子类型，也就是必须包含 `__len__()` 方法。比如列表和集合都可以获取到长度，因此可以被同时传入 `longer`。字符串虽然也有长度，但类型推断会将其判断为 `object`，解决方法是显式声明字符串为 `Sequence` 类型，这样就能和 `List` 一起传给 `longer` 函数了。

<img src="/cnblogs/images/python-static-typing/python-static-typing.022.jpeg" width="60%" />

Duck typing 是比较常见的一个概念，当你走路像鸭子、叫声像鸭子，你就是一只鸭子，也就是说我们检查的是类的行为。Python 预定义了一些常用的类型，比如前面提到的 `Sized`，它并不是一个真正的类，而且是表示类含有 `__len__()` 方法。这些类型定义在 `collections.abc` 模块中，`typing` 模块中定义的则是相应的泛型版本。

<img src="/cnblogs/images/python-static-typing/python-static-typing.023.jpeg" width="60%" />

先看左边的例子，上方的代码是显式指定了 `Bucket` 类所继承的类型，这是过去常用的做法；而下方则只是定义了这两个类型对应的方法 `__len__` 和 `__iter__`，当我们将 `Bucket` 实例赋值给 `Iterator[int]` 类型的变量时是可以通过类型检查的，因为 `Bucket` 实例包含了 `Iterator[int]` 所需要的 `__iter__` 方法。这样一来，类的定义就会更加简洁。右边的例子则是用户自定义类型，同样的，`Resource` 不需要声明自己继承自 `Closeable` 类型，只需要提供合乎规定的 `close` 方法即可。

<img src="/cnblogs/images/python-static-typing/python-static-typing.024.jpeg" width="60%" />

Python 的静态类型检查时在编译期发生的，如果想在运行期去检查就得自己写代码实现了，比如使用 `isinstance` 函数。当然，借助第三方库，我们可以将这个检查自动化，比如 typeguard 可以在函数被调用时检查传入的参数类型；pydantic 则能够检查类成员变量是否符合类型注解。

## 参考资料

* https://www.python.org/dev/peps/pep-0483/
* http://wphomes.soic.indiana.edu/jsiek/what-is-gradual-typing/
* https://www.bernat.tech/the-state-of-type-hints-in-python/
* https://blog.zulip.com/2016/10/13/static-types-in-python-oh-mypy/
* https://mypy.readthedocs.io/en/latest/index.html#overview-type-system-reference
