---
layout: post
title: "HotSpot JVM 中的对象指针压缩"
date: 2015-06-25 17:41
comments: true
categories: [Programming]
tags: [java, jvm]
published: true
---

原文：https://wiki.openjdk.java.net/display/HotSpot/CompressedOops

## 什么是一般对象指针？

一般对象指针（oop, ordinary object pointer）是HotSpot虚拟机的一个术语，表示受托管的对象指针。它的大小通常和本地指针是一样的。Java应用程序和GC子系统会非常小心地跟踪这些受托管的指针，以便在销毁对象时回收内存空间，或是在对空间进行整理时移动（复制）对象。

在一些从Smalltalk和Self演变而来的虚拟机实现中都有一般对象指针这个术语，包括：

* [Self][1]：一门基于原型的语言，是Smalltalk的近亲
* [Strongtalk][2]：Smalltalk的一种实现
* [Hotspot][3]
* [V8][4]

部分系统中会使用小整型（smi, small integers）这个名称，表示一个指向30位整型的虚拟指针。这个术语在Smalltalk的V8实现中也可以看到。

## 为什么需要压缩？

在[LP64][5]系统中，指针需要使用64位来表示；[ILP32][5]系统中则只需要32位。在ILP32系统中，堆内存的大小只能支持到4Gb，这对很多应用程序来说是不够的。在LP64系统中，所有应用程序运行时占用的空间都会比ILP32大1.5倍左右，这是因为指针占用的空间增加了。虽然内存是比较廉价的，但网络带宽和缓存容量是紧张的。所以，为了解决4Gb的限制而增加堆内存的占用空间，就有些得不偿失了。

在x86芯片中，ILP32模式可用的寄存器数量是LP64模式的一半。SPARC没有此限制；RISC芯片本来就提供了很多寄存器，LP64模式下会提供更多。

压缩后的一般对象指针在使用时需要将32位整型按因数8进行扩展，并加到一个64位的基础地址上，从而找到所指向的对象。这种方法可以表示四十亿个对象，相当于32Gb的堆内存。同时，使用此法压缩数据结构也能达到和ILP32系统相近的效果。

我们使用*解码*来表示从32位对象指针转换成64位地址的过程，其反过程则称为*编码*。

<!-- more -->

## 什么情况下会进行压缩？

运行在ILP32模式下的Java虚拟机，或在运行时将`UseCompressedOops`标志位关闭，则所有的对象指针都不会被压缩。

如果`UseCompressedOops`是打开的，则以下对象的指针会被压缩：

* 所有对象的[klass][6]属性
* 所有[对象指针实例][7]的属性
* 所有对象指针数组的元素（objArray）

HotSpot VM中，用于表示Java类的数据结构是不会压缩的，这部分数据都存放在永久代（PermGen）中。

在解释器中，一般对象指针也是不压缩的，包括JVM本地变量和栈内元素、调用参数、返回值等。解释器会在读取堆内对象时解码对象指针，并在存入时进行编码。

同样，方法调用序列（method calling sequence），无论是解释执行还是编译执行，都不会使用对象指针压缩。

在编译后的代码中，对象指针是否压缩取决于不同的优化结果。优化后的代码可能会将压缩后的对象指针直接从一处搬往另一处，而不进行编解码操作。如果芯片（如x86）支持解码，那在使用对象指针时就不需要自行解码了。

所以，以下数据结构在编译后的代码中既可以是压缩后的对象指针，也可能是本地地址：

* 寄存器或溢出槽（spill slot）中的数据
* 对象指针映射表（GC映射表）
* 调试信息
* 嵌套在机器码中的对象指针（在非RISC芯片中支持，如x86）
* [nmethod][8]常量区（包括那些影响到机器码的重定位操作）

在HotSpot JVM的C++代码部分，对象指针压缩与否反映在C++的静态类型系统中。通常情况下，对象指针是不压缩的。具体来说，C++的成员函数在操作本地代码传递过来的指针时（如*this*），其执行过程不会有什么不同。JVM中的部分方法则提供了重载，能够处理压缩和不压缩的对象指针。

重要的C++数据不会被压缩：

* C++对象指针（*this*）
* 受托管指针的句柄（Handle类型等）
* JNI句柄（jobject类型）

C++在使用对象指针压缩时（加载和存储等），会以`narrowOop`作为标记。

## 使用压缩寻址

以下是使用对象指针压缩的x86指令示例：

```text
! int R8; oop[] R9;  // R9是64位
! oop R10 = R9[R8];  // R10是32位
! 从原始基址指针加载压缩对象指针：
movl R10, [R9 + R8<<3 + 16]
! klassOop R11 = R10._klass;  // R11是32位
! void* const R12 = GetHeapBase();
! 从压缩基址指针加载klass指针：
movl R11, [R12 + R10<<3 + 8]
```

以下sparc指令用于解压对象指针（可为空）：

```text
! java.lang.Thread::getThreadGroup@1 (line 1072)
! L1 = L7.group
ld  [ %l7 + 0x44 ], %l1
! L3 = decode(L1)
cmp  %l1, 0
sllx  %l1, 3, %l3
brnz,a   %l3, .+8
add  %l3, %g6, %l3  ! %g6是常量堆基址
```

*输出中的注解来自[PrintAssembly插件][9]。*

## 空值处理

32位零值会被解压为64位空值，这就需要在解码逻辑中加入一段特殊的逻辑。或者说可以默认某些压缩对象指针肯定不会空（如klass的属性），这样就能使用简单一些的编解码逻辑了。

隐式空值检测对JVM的性能至关重要，包括解释执行和编译执行的字节码。对于一个偏移量较小的对象指针，如果基址指针为空，那很有可能造成系统崩溃，因为虚拟地址空间的前几页通常是没有映射的。

对于压缩对象指针，我们可以用一种类似的技巧来欺骗它：将堆内存前几页的映射去除，如果解压出的指针为空（相对于基址指针），仍可以用它来做加载和存储的操作，隐式空值检测也能照常运行。

## 对象头信息

对象头信息通常包含几个部分：固定长度的标志位；klass信息；如果对象是数组，则包含一个32位的信息，并可能追加一个32位的空隙进行对齐；零个或多个实例属性，数组元素，元信息等。（有趣的是，Klass的对象头信息包含了一个C++的[虚拟方法表][10]）

上述追加的32位空隙通常也可用于存储属性信息。

如果`UseCompressedOops`关闭，标志位和klass都是正常长度。对于数组，32位空隙在LP64系统中总是存在；而ILP32系统中，只有当数组元素是64位数据时才存在这个空隙。

如果`UseCompressedOops`打开，则klass是32位的。非数组对象在klass后会追加一个空隙，而数组对象则直接开始存储元素信息。

## 零基压缩技术

压缩对象指针（narrow-oop）是基于某个地址的偏移量，这个基础地址（narrow-oop-base）是由Java堆内存基址减去一个内存页的大小得来的，从而支持隐式空值检测。所以一个属性字段的地址可以这样得到：

```text
<narrow-oop-base> + (<narrow-oop> << 3) + <field-offset>.
```

如果基础地址可以是0（Java堆内存不一定要从0偏移量开始），那么公式就可以简化为：

```text
(<narrow-oop << 3) + <field-offset>
```

理论上说，这一步可以省去一次寄存器上的加和操作。而且使用零基压缩技术后，空值检测也就不需要了。

之前的解压代码是：

```text
if (<narrow-oop> == NULL)
    <wide_oop> = NULL
else
    <wide_oop> = <narrow-oop-base> + (<narrow-oop> << 3)
```

使用零基压缩后，只需使用移位操作：

```text
<wide_oop> = <narrow-oop> << 3
```

零基压缩技术会根据堆内存的大小以及平台特性来选择不同的策略：

1. 堆内存小于4Gb，直接使用压缩对象指针进行寻址，无需压缩和解压；
2. 堆内存大于4Gb，则尝试分配小于32Gb的堆内存，并使用零基压缩技术；
3. 如果仍然失败，则使用普通的对象指针压缩技术，即`narrow-oop-base`。


[1]: https://github.com/russellallen/self/blob/master/vm/src/any/objects/oop.hh
[2]: http://code.google.com/p/strongtalk/wiki/VMTypesForSmalltalkObjects
[3]: http://hg.openjdk.java.net/hsx/hotspot-main/hotspot/file/0/src/share/vm/oops/oop.hpp
[4]: http://code.google.com/p/v8/source/browse/trunk/src/objects.h
[5]: http://docs.oracle.com/cd/E19620-01/805-3024/lp64-1/index.html
[6]: http://stackoverflow.com/questions/16721021/what-is-klass-klassklass
[7]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/sun/jvm/hotspot/oops/Oop.java#Oop
[8]: http://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html#nmethod
[9]: https://wiki.openjdk.java.net/display/HotSpot/PrintAssembly
[10]: https://en.wikipedia.org/wiki/Virtual_method_table
