---
title: Python 类型检查实践
tags: [python]
categories: Programming
---

Python 作为一门动态类型语言，代码灵活度和开发效率都是非常高的。但随着项目代码逐渐变多，函数之间的调用变得更复杂，经常会出现参数或返回值类型不正确等问题。并且这些问题只能在运行时被发现，甚至会产生线上 Bug。那么如何能让 Python 像 Java 或 Go 这样的语言一样，在编译期就进行类型检查呢？从 3.5 版本开始，Python 就能支持静态类型检查了。本文整理自团队的一次内部分享，介绍了 Python 的这一新特性。

![目录](/cnblogs/images/python-static-typing/python-static-typing.002.jpeg)

<!-- more -->

## Quick Start

首先我们来快速看一下，Python 中要如何使用静态类型检查。

![Quick Start I](/cnblogs/images/python-static-typing/python-static-typing.003.jpeg)

通过给函数的参数和返回值添加类型注解，配合使用 mypy 等工具，我们就可以对 Python 代码做静态类型检查了。上图中的 `greeting` 函数接收一个 `name` 参数，我们将其定义为 `str` 类型；函数的返回值也是 `str` 类型。使用 `pip install` 安装 mypy 后，就可以进行类型检查了。很显然，这个函数是能通过类型检查的，因为字符串之间可以进行 `+` 的操作。假设我们将 `name` 变量与一个整数相加，mypy 就会报告无法对 `str` 和 `int` 类型做 `+` 的操作；同样，如果调用 `greeting` 时传入的是整型，或者将 `greeting` 的返回值和整型相加，都会触发类型检查错误。

![Quick Start II](/cnblogs/images/python-static-typing/python-static-typing.004.jpeg)

除了对函数参数做类型注解，我们也可以对本地变量做注解，语法是类似的。上图中的 `number` 变量我们声明为 `int` 类型，在将字符串赋值给它的时候就会引发报错；同样，如果将 `number` 作为函数返回值，也会触发报错。

除了 `int` 和 `str` 这类基本类型，我们也可以对容器类进行注解。更进一步，我们可以去规定容器中可以存放什么类型的变量。图中的 `items` 变量被定义为了 `list` 类型，因此无法赋值一个整型。但是，`list` 类型没有规定容器中元素的类型，因此 `[1, 2, 3]` 和 `['a', 'b', 'c']` 都是可以赋值给 `items` 变量的。如果我们想定义一个只包含整型元素的 `list`，就需要使用 `typing` 模块提供的 `List` 类型。比如 `nums` 变量就是一个只包含整型元素的列表，如果我们想要将字符串加入这个列表中，就会引发类型检查错误；字典类型 `Dict` 也是类似的作用。需要注意的是，这些代码在执行时是不会报错的，因为 Python 仍然是动态类型语言，运行期不会进行类型检查，只有用 `mypy` 等工具去检查时才会有效。

![Quick Start III](/cnblogs/images/python-static-typing/python-static-typing.005.jpeg)

那是不是所有的变量都需要做类型注解呢？答案当然是否定的，因为 mypy 有类型推断机制。比如上图中的 `nums` 变量，虽然没有声明类型，但是因为赋值给它的是一个整型列表，所以该变量会被推断成 `List[int]` 类型，也就无法添加字符串类型的元素了。绕过这一机制的方式当然是显式地去指定类型，比如当指定为 `List[object]` 类型时，就可以写入字符串、整型等任意元素了。

最后我们看一下如何给类成员变量注解类型。两种方式：第一种是图中的 `suffix` 变量，在声明的时候进行注解；第二种是利用类型推断，比如构造函数中，`date` 参数传入的是 `str` 类型，并赋值给了 `self.date`，那么 `self.date` 就会被推断为 `str` 类型。这样一来，`run` 函数中的两条语句都会产生类型检查错误。

## 为什么要引入类型检查

我们可以看到，静态类型检查是会增加额外的工作量的，那么我们为什么要引入静态类型检查，它的优点有哪些呢？

![Why Go?](/cnblogs/images/python-static-typing/python-static-typing.007.jpeg)

我之前在分享 *Learn Go in 10 Minutes* 的时候提到过，Go 语言的优点之一是其静态类型系统。静态类型可以在编译期就发现错误，而不用等到程序运行出错时才去修复；而且研究表明，静态类型系统是可以减少 Bug 的数量的，研究者使用的对象是 TypeScript 和 Flow，这两者都为 JavaScript 语言提供了类型检查机制；最后，静态类型系统对程序性能的提升也是有帮助的。

![概念梳理](/cnblogs/images/python-static-typing/python-static-typing.008.jpeg)

先来梳理几个概念，我们一直会听到静态类型、动态类型、强类型、弱类型等术语。其中，静态类型和动态类型比较容易区分：前者在编译期进行类型检查，后者在运行期进行检查。像 Java、C、Golang 等是比较常见的静态类型语言，而 JavaScript、PHP、Python 等则是动态类型的。强类型和弱类型则比较难以区分了，判断标准是：允许隐式类型转换的程度。比如 JavaScript 是最弱的一门弱类型语言，因为任何类型的变量都可以进行相加等操作，执行引擎会自动做隐式转换；PHP 也是弱类型语言，不过在转换时会报一些 Warning 信息；Python 则是强类型语言，因为当我们将字符串和整型做相加操作时会直接报错。

## 参考资料
