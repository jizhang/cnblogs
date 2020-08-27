---
title: Python 类型检查实践
tags: [python]
categories: Programming
---

Python 作为一门动态类型语言，代码灵活度和开发效率都是非常高的。但随着项目代码逐渐变多，函数之间的调用变得更复杂，经常会出现参数或返回值类型不正确等问题。并且这些问题只能在运行时被发现，甚至会产生线上 Bug。那么如何能让 Python 像 Java 或 Go 这样的语言一样，在编译期就进行类型检查呢？从 3.5 版本开始，Python 就能支持静态类型检查了。下面我就将团队的一次内部分享整理给大家：

![目录](/cnblogs/images/python-static-typing/python-static-typing.002.jpeg)

<!-- more -->

首先我们来快速看一下，Python 中要如何使用静态类型检查。

![Quick Start I](/cnblogs/images/python-static-typing/python-static-typing.003.jpeg)

## 参考资料
