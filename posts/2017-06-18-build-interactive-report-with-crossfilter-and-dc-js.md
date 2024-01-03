---
title: 使用 Crossfilter 和 dc.js 构建交互式报表
tags:
  - crossfilter
  - dc.js
  - analytics
categories:
  - Big Data
date: 2017-06-18 18:59:53
---


在对多维数据集进行图表分析时，我们希望在图表之间建立联系，选择图表中的一部分数据后，其他图表也会相应变动。这项工作可以通过开发完成，即在服务端对数据进行过滤，并更新所有图表。此外，我们还可以借助 Crossfilter 和 dc.js 这两个工具，直接在浏览器中对数据进行操作。

## 航班延误统计

这是 Crossfilter 官方网站提供的示例，基于 [ASA Data Expo](http://stat-computing.org/dataexpo/2009/) 数据集的航班延误统计。下面我们将介绍如何用 dc.js 来实现这份交互式报表。项目源码可以在 [JSFiddle](https://jsfiddle.net/zjerryj/gjao9sws/) 中浏览，演示的数据量减少到 1000 条。

![](/images/airline-ontime-performance.png)

<!-- more -->

## 背景知识

[Crossfilter](http://crossfilter.github.io/crossfilter/) 是一个 JavaScript 类库，能够在浏览器端对大量数据进行多维分析。它的特点是可以在不同的 Group By 查询之间实现“交叉过滤”，自动连接和更新查询结果。结合 [dc.js](https://dc-js.github.io/dc.js/) 图表类库，我们就可以构建出高性能、交互式的分析报表了。

## 数据集、维度、度量

Crossfilter 中有维度、度量等概念。如果你对数据仓库或统计分析有所了解，这些术语和 OLAP 立方体中的定义是相似的。

* 数据集：即一张二维表，包含行和列，在 JavaScript 中通常是由对象组成的数组；
* 维度：用于进行 Group By 操作的字段，它通常是可枚举的值，如日期、性别，也可以是数值范围，如年龄范围等；
* 度量：可以进行合计、计算标准差等操作，通常是数值型的，如收入、子女人数；记录数也是一种度量；

```js
let flights = d3.csv.parse(flightsCsv)
let flight = crossfilter(flights)
let hour = flight.dimension((d) => d.date.getHours() + d.date.getMinutes() / 60)
let hours = hour.group(Math.floor)
```

这段代码首先创建了一个 crossfilter 对象，数据来源是 CSV 格式的文本。之后，我们定义了一个“小时”的维度，它是从 `date` 字段计算得到的，并近似到了整数，用作 Group By 条件。定义完这个查询后，我们就可以获取航班延误次数最多的三个小时的数据了：

```js
hours.top(3)
// 输出结果
[
  { key: 13, value: 72 },
  { key: 20, value: 72 },
  { key:  8, value: 71 },
]
```

## 可视化

我们将 24 个小时的延误次数用柱状图展现出来：

```js
let hourChart = dc.barChart('#hour-chart')
hourChart
  .width(350)
  .height(150)
  .dimension(hour)
  .group(hours)
  .x(d3.scale.linear()
    .domain([0, 24])
    .rangeRound([0, 10 * 24]))
  .controlsUseVisibility(true)
```

对应的 HTML 代码是：

```html
<div id="hour-chart">
  <div class="title">Time of Day
    <a class="reset" href="javascript:;" style="visibility: hidden;">reset</a>
  </div>
</div>
```

我们可以看到，dc.js 和 crossfilter 结合的非常好，只需将维度信息传入 dc.js ，并对坐标轴进行适当配置，就可以完成可视化。在这个例子中，X 轴是 0 到 24 个小时（维度），Y 轴则是航班延误次数（度量）。

注意这里 `class="reset"` 的用法，它和 `controlUseVisibility` 一起使用，会在指定位置产生一个 `reset` 按钮。你可以试着在图表上进行拖拽，对数据进行筛选后就能看到 `reset` 按钮了。

## 交叉过滤

接下来我们可以创建更多图表，如航班延误时间的直方图，源代码可以在 JSFiddle 中找到。当你对其中一个图表进行过滤时，其他图表就会自动过滤和更新，这样我们就可以查看某些限定条件下数据的分布情况了。

dc.js 还提供了许多可视化类型，如饼图、表格、自定义HTML等。当然，要完全掌握这些，还需要学习 d3.js ，市面上很多绘图类库都是构建在它之上的。

## 参考资料

* [Crossfilter - Fast Multidimensional Filtering for Coordinated Views](http://crossfilter.github.io/crossfilter/)
* [dc.js - Dimensional Charting Javascript Library](https://dc-js.github.io/dc.js/)
* [Crossfiler Tutorial](http://blog.rusty.io/2012/09/17/crossfilter-tutorial/)
