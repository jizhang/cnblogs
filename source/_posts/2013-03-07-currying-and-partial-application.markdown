---
layout: post
title: "柯里化与偏应用（JavaScript 描述）"
date: 2013-03-07 20:59
comments: true
categories: Programming
tags: [functional programming, javascript, translation]
---

原文：[http://raganwald.com/2013/03/07/currying-and-partial-application.html](http://raganwald.com/2013/03/07/currying-and-partial-application.html)

上周末我参加了[wroc_love.rb大会](http://wrocloverb.com/)，其间[Steve Klabnik](http://steveklabnik.com/)的一张PPT中提到了[偏应用（Partial Application）](https://en.wikipedia.org/wiki/Partial_application)和[柯里化（Currying）](https://en.wikipedia.org/wiki/Currying)，并说这两者之间的区别如今已经不重要了。但是我不这么认为。

在这周发布的博文中，我用五种方式对`this`和闭包做了解释，但只有三到四种提到了柯里化。所以这篇博文就重点来谈谈这个。

函数参数的个数
--------------

在讲解之前，我们先明确一些术语。函数定义时会写明它所接收的参数个数（Arity）。“一元函数”（Unary）接收一个参数，“多元函数”（Polyadic）接收多个参数。还有一些特殊的名称，如“二元函数”（Binary）接收两个参数，“三元函数”（Ternary）接收三个参数等。你可以对照希腊语或拉丁语词汇来创造这些特殊的名称。

有些函数能够接收不定数量的参数，我们称之为“可变参数函数”（Variadic）。不过这类函数、以及不接收参数的函数并不是本文讨论的重点。

<!-- more -->

偏应用
------

偏应用的概念很容易理解，我们可以使用加法函数来做简单的演示，但如果你不介意的话，我想引用[allong.es](http://allong.es/)这一JavaScript类库中的代码来做演示，而且它也是会在日常开发中用到的代码。

作为铺垫，我们首先实现一个`map`函数，用来将某个函数应用到数组的每个元素上：

```javascript
var __map = [].map;

function map (list, unaryFn) {
  return __map.call(list, unaryFn);
};

function square (n) {
  return n * n;
};

map([1, 2, 3], square);
  //=> [1, 4, 9]
```

显然，`map`是二元函数，`square`是一元函数。当我们使用`[1, 2, 3]`和`square`作为参数来调用`map`时，我们是将这两个参数 *应用（Apply）* 到`map`函数，并获得结果。

由于`map`函数接收两个参数，我们也提供了两个参数，所以说这是一次 *完整应用* 。那何谓偏应用（或部分应用）呢？其实就是提供少于指定数量的参数。如，仅提供一个参数来调用`map`。

如果我们只提供一个参数来调用`map`会怎么样？我们无法得到所要的结果，只能得到一个新的一元函数，通过调用这个函数并传递缺失的参数后，才能获得结果。

假设现在我们只提供一个参数给`map`，这个参数是`unaryFn`。我们从后往前来逐步实现，首先为`map`函数创建一个包装函数：

```javascript
function mapWrapper (list, unaryFn) {
  return map(list, unaryFn);
};
```

然后，我们将这个二元函数分割成两个嵌套的一元函数：

```javascript
function mapWrapper (unaryFn) {
  return function (list) {
    return map(list, unaryFn);
  };
};
```

这样一来，我们就能每次仅传递一个参数来进行调用了：

```javascript
mapWrapper(square)([1, 2, 3]);
  //=> [1, 4, 9]
```

和之前的`map`函数相较，新的函数`mapWrapper`是一元函数，它的返回值是另一个一元函数，需要再次调用它才能获得返回值。那么偏应用要从何体现？让我们从第二个一元函数着手：

```javascript
var squareAll = mapWrapper(square);
  //=> [function]

squareAll([1, 2, 3]);
  //=> [1, 4, 9]
squareAll([5, 7, 5]);
  //=> [25, 49, 25]
```

我们首先将`square`这个参数部分应用到了`map`函数，并获得一个一元函数`squareAll`，它能实现我们需要的功能。偏应用后的`map`函数十分便捷，而[allong.es](http://allong.es/)库中提供的`splat`函数做的也是相同的事情。

如果每次想要使用偏应用都需要手动编写这样一个包装函数，程序员显然会想到要自动化实现它。这就是下一节的内容：柯里化。

柯里化
------

首先，我们可以编写一个函数来返回包装器。我们仍然以二元函数为例：

```javascript
function wrapper (unaryFn) {
  return function (list) {
    return map(list, unaryFn);
  };
};
```

将函数`map`和参数名称替换掉：

```javascript
function wrapper (secondArg) {
  return function (firstArg) {
    return binaryFn(firstArg, secondArg);
  };
};
```

最后，我们再包装一层：

```javascript
function rightmostCurry (binaryFn) {
  return function (secondArg) {
    return function (firstArg) {
      return binaryFn(firstArg, secondArg);
    };
  };
};
```

这样一来，我们之前使用的“模式”就抽象出来了。这个函数的用法是：

```javascript
var rightmostCurriedMap = rightmostCurry(map);

var squareAll = rightmostCurriedMap(square);

squareAll([1, 4, 9]);
  //=> [1, 4, 9]
squareAll([5, 7, 5]);
  //=> [25, 49, 25]
```

将一个多元函数转换成一系列一元函数的嵌套调用，这种转换称之为 **柯里化** 。它的名称取自其发明者Haskell Curry，他也重新定义了由[Moses Schönfinkel](https://en.wikipedia.org/wiki/Moses_Sch%C3%B6nfinkel)提出的组合子逻辑（Combinatory Logic）。（[注1](#fn:birds)）

`rightmostCurry`函数可以将任意二元函数转换为一组一元函数，从传递第二个参数开始，因此才称其为“右起柯里化”。

和它相反的自然是“左起柯里化”，大多数逻辑学家使用“左起柯里化”，所以人们常说的柯里化指的也是左起柯里化：

```javascript
function curry (binaryFn) {
  return function (firstArg) {
    return function (secondArg) {
      return binaryFn(firstArg, secondArg);
    };
  };
};

var curriedMap = curry(map),
    double = function (n) { n + n; };

var oneToThreeEach = curriedMap([1, 2, 3]);

oneToThreeEach(square);
  //=> [1, 4, 9]
oneToThreeEach(double);
  //=> [2, 4, 6]
```

那这两种柯里化方式应该如何选择呢？这就要看你的用途了。在上述二元函数的示例中，我们模拟的是一种“主体-客体”（Subject-Object）的语法。第一个参数表示主体，第二个参数表示客体。

当我们使用“右起柯里化”的`map`函数时，我们即假定主体是那个将被调用多次的函数（unaryFn）。

看到`squareAll([1, 2, 3])`时，我们会理解为“将数组[1, 2, 3]中的每个元素做平方运算”。使用“右起柯里化”，我们使平方运算成为主体，数组成为客体。而当使用一般的柯里化时，则是让数组作为主体，平方运算作为客体。

另一种理解的方式是看你需要重用哪一部分。通过不同的柯里化方式，你可以选择重用函数还是重用列表。

再谈偏应用
----------

上文谈了那么多柯里化，那偏应用呢？事实上，当你有了柯里化，你就不需要偏应用了。同样地，当你使用了偏应用，也不会需要柯里化。所以当你需要为此撰写一篇文章时，最便捷的做法是先描述其中的一个，然后建立在其基础之上来描述另外一个。

首先让我们回顾一下右起柯里化：

```javascript
function rightmostCurry (binaryFn) {
  return function (secondArg) {
    return function (firstArg) {
      return binaryFn(firstArg, secondArg);
    };
  };
};
```

你会发现在实际使用过程中会一直出现以下代码：

```javascript
var squareAll = rightmostCurry(map)(square),
    doubleAll = rightmostCurry(map)(double);
```

这种创建了柯里化函数后立刻调用的情况很常见，因此好事的人们就为它起了一个名字，称之为 *map函数的右起一元偏应用* 。

名字很长，我们分解开来看：

1. 右起：从最右边的参数开始；
2. 一元：一个参数；
3. 偏应用：只应用部分函数；
4. map：即`map`函数。

所以我们实际上是想为`map`函数预先指定一个参数。它是一个二元函数，指定参数后便成了一元函数。在函数式编程语言或类库中，都提供了相应的方式来支持这种用法。

我们可以用柯里化来实现这样的功能：

```javascript
function rightmostUnaryPartialApplication (binaryFn, secondArg) {
  return rightmostCurry(binaryFn)(secondArg);
};
```

但更多时候我们会使用更为直接的方式：（[注2](#fn:caveat)）

```javascript
function rightmostUnaryPartialApplication (binaryFn, secondArg) {
  return function (firstArg) {
    return binaryFn(firstArg, secondArg);
  };
};
```

`rightmostUnaryPartialApplication`有些过长了，我们将其称为`applyLast`：

```javascript
var applyLast = rightmostUnaryPartialApplication;
```

这样，我们的`squareAll`和`doubleAll`函数就可以写为：

```javascript
var squareAll = applyLast(map, square),
    doubleAll = applyLast(map, double);
```

你同样可以实现一个`applyFirst`函数（我们就不提`leftmostUnaryPartialApplication`这种叫法了）：

```javascript
function applyFirst (binaryFn, firstArg) {
  return function (secondArg) {
    return binaryFn(firstArg, secondArg);
  };
};
```

和“左起/右起柯里化”一样，你应该在工具箱中保留这两种偏应用的方式，以便在实际使用过程中选择。

柯里化和偏应用的区别
--------------------

“柯里化是将一个多元函数分解为一系列嵌套调用的一元函数。分解后，你可以部分应用一个或多个参数（[注3](#fn:also)）。柯里化的过程不会向函数传递参数。”

“偏应用是为一个多元函数预先提供部分参数，从而在调用时可以省略这些参数。”

这就是全部吗？
--------------

是，但又不是。以下这些还请读者自行探索和实现：

1. 上文中，我们用柯里化实现了偏应用，那偏应用可以实现柯里化吗？为什么？（[注4](#fn:tao)）
2. 所有的示例都是将二元函数转换为一元函数，尝试写出一个更为通用的`applyFirst`和`applyLast`函数，能够为任意元的函数提供一个参数。如，假设有一个函数接收四个参数，那在使用了`applyFirst`后会返回一个接收三个参数的函数。
3. 第2步完成后，再实现一组`applyLeft`和`applyRight`函数，它能为任意元的函数预先指定任意数量的参数，如，假设向`applyLeft`传递了一个三元函数和两个参数，那就会返回一个一元函数。
4. 重写`curry`和`rightmostCurry`这两个函数，使其能够接收任意元的函数。一个三元函数柯里化后会产生三个嵌套调用的一元函数。
5. 阅读[allong.es](http://allong.es/)的代码，这是一个从[JavaScript Allongé](http://leanpub.com/javascript-allonge)中提取的函数式编程类库。重点阅读它的partial_application.js文件。

感谢你的阅读，如果你在代码中发现了Bug，请[克隆这个镜像](https://github.com/raganwald/raganwald.github.com)，提交合并申请，或者[在Github上提交一个事务](https://github.com/raganwald/raganwald.github.com/issues)。

PS：你可能会对另一篇文章也感兴趣：[Practical Applicaitons for Partial Application](http://raganwald.com/2013/01/05/practical-applications-of-partial-application.html)。

（[讨论](http://www.reddit.com/r/javascript/comments/19urej/whats_the_difference_between_currying_and_partial/)）

脚注
----

1. <a name="fn:birds"></a>当Raymond Smullyan为组合子逻辑撰写介绍时，他称之为“嘲鸟的模仿者”（To Mock a Mockingbird）。他通篇使用树林和小鸟来做比喻，以表达对Schönfinkel的敬意。Schön意为“美丽”，Fink则指德语中的Finch（燕雀），也指犹太语中的Finkl（火花）。所以他的名字可以理解为“美丽的燕雀”或“美丽的火花”。
2. <a name="fn:caveat"></a>本文的示例都异常简单。完整的实现应该能够接收任意元的函数，并依调用情况返回恰当的值。
3. <a name="fn:also"></a>柯里化还有很多其它应用，只是本文着重讲述的是柯里化和偏应用的区别，而不是组合子逻辑和函数式编程。
4. <a name="fn:tao"></a>一位道教人士向街边小贩购买一个素食热狗，并说道：“我要全套。”
