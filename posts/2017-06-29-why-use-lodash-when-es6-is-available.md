---
title: 为什么不用 ES6 完全替换 Lodash
tags:
  - frontend
  - javascript
  - es6
  - lodash
categories: Programming
date: 2017-06-29 14:49:14
---


[Lodash](https://lodash.com/) 是一款非常知名的 JavaScript 工具库，能够让开发者十分便捷地操纵数组和对象。我则是非常喜欢用它提供的函数式编程风格来操作集合类型，特别是链式调用和惰性求值。然而，随着 [ECMAScript 2015 Standard (ES6)][1] 得到越来越多主流浏览器的支持，以及像 [Babel](https://babeljs.io/) 这样，能够将 ES6 代码编译成 ES5 从而在旧浏览器上运行的工具日渐流行，人们会发现许多 Lodash 提供的功能已经可以用 ES6 来替换了。然而真的如此吗？我认为，Lodash 仍然会非常流行，因为它可以为程序员提供更多的便利，并且优化我们编程的方式。

## `_.map` 和 `Array#map` 是有区别的

在处理集合对象时，`_.map`、`_.reduce`、`_.filter`、以及 `_.forEach` 的使用频率很高，而今 ES6 已经能够原生支持这些操作：

```js
_.map([1, 2, 3], (i) => i + 1)
_.reduce([1, 2, 3], (sum, i) => sum + i, 0)
_.filter([1, 2, 3], (i) => i > 1)
_.forEach([1, 2, 3], (i) => { console.log(i) })

// 使用 ES6 改写
[1, 2, 3].map((i) => i + 1)
[1, 2, 3].reduce((sum, i) => sum + i, 0)
[1, 2, 3].filter((i) => i > 1)
[1, 2, 3].forEach((i) => { console.log(i) })
```

但是，Lodash 的 `_.map` 函数功能更强大，它能够操作对象类型，提供了遍历和过滤的快捷方式，能够惰性求值，对 `null` 值容错，并且有着更好的性能。

<!-- more -->

### 遍历对象类型

ES6 中有以下几种方式来遍历对象：

```js
for (let key in obj) { console.log(obj[key]) }
for (let key of Object.keys(obj)) { console.log(obj[key]) }
Object.keys(obj).forEach((key) => { console.log(obj[key]) })
```

Lodash 中则有一个统一的方法 `_.forEach`：

```js
_.forEach(obj, (value, key) => { console.log(value) })
```

虽然 ES6 的 `Map` 类型也[提供][2]了 `forEach` 方法，但我们需要先费些功夫将普通的对象类型转换为 `Map` 类型：

```js
// http://stackoverflow.com/a/36644532/1030720
const buildMap = o => Object.keys(o).reduce((m, k) => m.set(k, o[k]), new Map());
```

### 遍历和过滤的快捷方式

比如我们想从一组对象中摘取出某个属性的值：

```js
let arr = [{ n: 1 }, { n: 2 }]
// ES6
arr.map((obj) => obj.n)
// Lodash
_.map(arr, 'n')
```

当对象类型的嵌套层级很多时，Lodash 的快捷方式就更实用了：

```js
let arr = [
  { a: [ { n: 1 } ]},
  { b: [ { n: 1 } ]}
]
// ES6
arr.map((obj) => obj.a[0].n) // TypeError: 属性 'a' 在 arr[1] 中未定义
// Lodash
_.map(arr, 'a[0].n') // => [1, undefined]
```

可以看到，Lodash 的快捷方式还对 `null` 值做了容错处理。此外还有过滤快捷方式，以下是从 Lodash 官方文档中摘取的示例代码：

```js
let users = [
  { 'user': 'barney', 'age': 36, 'active': true },
  { 'user': 'fred',   'age': 40, 'active': false }
];
// ES6
users.filter((o) => o.active)
// Lodash
_.filter(users, 'active')
_.filter(users, ['active', true])
_.filter(users, {'active': true, 'age': 36})
```

### 链式调用和惰性求值

Lodash 的这一特性就非常有趣和实用了。现在很流行使用短小可测的函数，结合链式调用和惰性求值，来操作集合类型。大部分 Lodash 函数都可以进行链式调用，下面是一个典型的 **WordCount** 示例：

```js
let lines = `
an apple orange the grape
banana an apple melon
an orange banana apple
`.split('\n')

_.chain(lines)
  .flatMap(line => line.split(/\s+/))
  .filter(word => word.length > 3)
  .groupBy(_.identity)
  .mapValues(_.size)
  .forEach((count, word) => { console.log(word, count) })

// apple 3
// orange 2
// grape 1
// banana 2
// melon 1
```

## 解构赋值和箭头函数

ES6 引入了解构赋值、箭头函数等新的语言特性，可以用来替换 Lodash：

```js
// Lodash
_.head([1, 2, 3]) // => 1
_.tail([1, 2, 3]) // => [2, 3]
// ES6 解构赋值（destructuring syntax）
const [head, ...tail] = [1, 2, 3]

// Lodash
let say = _.rest((who, fruits) => who + ' likes ' + fruits.join(','))
say('Jerry', 'apple', 'grape')
// ES6 spread syntax
say = (who, ...fruits) => who + ' likes ' + fruits.join(',')
say('Mary', 'banana', 'orange')

// Lodash
_.constant(1)() // => 1
_.identity(2) // => 2
// ES6
(x => (() => x))(1)() // => 1
(x => x)(2) // => 2

// 偏应用（Partial application）
let add = (a, b) => a + b
// Lodash
let add1 = _.partial(add, 1)
// ES6
add1 = b => add(1, b)

// 柯里化（Curry）
// Lodash
let curriedAdd = _.curry(add)
let add1 = curriedAdd(1)
// ES6
curriedAdd = a => b => a + b
add1 = curriedAdd(1)
```

对于集合类操作，我更倾向于使用 Lodash 函数，因为它们的定义更准确，而且可以串联成链；对于那些可以用箭头函数来重写的用例，Lodash 同样显得更简单和清晰一些。此外，[参考资料](#参考资料)里的几篇文章中也提到，在函数式编程场景下，Lodash 提供了更为实用的柯里化、[运算符函数][3]、[FP 模式][4]等特性。

## 总结

Lodash 为 JavaScript 语言增添了诸多特性，程序员可以用它轻松写出语义精确、执行高效的代码。此外，Lodash 已经完全[模块化][5]了。虽然它的某些特性最终会被淘汰，但仍有许多功能是值得我们继续使用的。同时，Lodash 这样的类库也在不断推动 JavaScript 语言本身的发展。

## 参考资料

* [10 Lodash Features You Can Replace with ES6](https://www.sitepoint.com/lodash-features-replace-es6/)
* [Does ES6 Mean The End Of Underscore / Lodash?](https://derickbailey.com/2016/09/12/does-es6-mean-the-end-of-underscore-lodash/)
* [Why should I use lodash - reddit](https://www.reddit.com/r/javascript/comments/41fq2s/why_should_i_use_lodash_or_rather_what_lodash/)

[1]: http://www.ecma-international.org/ecma-262/6.0/
[2]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map
[3]: https://lodash.com/docs/#add
[4]: https://github.com/lodash/lodash/wiki/FP-Guide
[5]: https://lodash.com/custom-builds
