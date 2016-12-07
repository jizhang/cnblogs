---
title: ES6普及后为何仍要使用Lodash
date: 2016-12-07 14:06
categories: Programming
tags: [frontend, javascript, es6, lodash]
---

* map, filter, reduce的功能还是不一样的：
  * lazy evaluation
  * iteratee / predicate shortcuts
  * works for object
  * null values (robust, browser difference)
  * performance
* 很多实用函数，而且可以模块化加载
* 推动语言发展
* fp模式（对比rambda）

<!-- more -->

## 参考资料

* [10 Lodash Features You Can Replace with ES6](https://www.sitepoint.com/lodash-features-replace-es6/) 文章评论中有Lodash作者的观点
* [Does ES6 Mean The End Of Underscore / Lodash?](https://derickbailey.com/2016/09/12/does-es6-mean-the-end-of-underscore-lodash/)
* [Why should I use lodash - reddit](https://www.reddit.com/r/javascript/comments/41fq2s/why_should_i_use_lodash_or_rather_what_lodash/)
