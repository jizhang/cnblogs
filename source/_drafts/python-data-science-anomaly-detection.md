---
title: "2017 Top 15 Python 数据科学类库；时间序列异常点检测；如何加入开源项目"
categories: [Digest]
tags: [data science, analytics, opensource]
---

## 2017 Top 15 Python 数据科学类库

![Google Trends](/cnblogs/images/digest/google-trends.png)

近年来，Python 在数据科学领域得到了越来越多的关注，本文整理归类了使用率最高的数据科学类库，供大家参考。

NumPy、SciPy、Pandas 是 Python 数据科学的核心类库。NumPy 提供了 N 维数组、矩阵、向量等数据结构，能够进行高性能的数学运算；SciPy 包含了线性代数、拟合优化、统计学习的通用方法；Pandas 则一般用于数据清洗、探索型分析等工作。

可视化方面，Matplotlib 是最早流行的类库，提供了丰富的图形化接口，但 API 的使用方式偏底层，需要编写较多代码；Seaborn 构建在 Matplotlib 之上，重新定义了图表样式，更适合在报告、演示文档中使用，并且它还预置了诸多探索型分析函数，可以快速地对数据进行描述性可视化；Bokeh 主打交互性，它运行在浏览器中，让使用者可以方便地调节可视化参数；Plotly 也是一款基于页面的可视化工具，但因为是商业软件，需要授权后才能使用。

SciKit-Learn 是公认的 Python 机器学习标准类库，它提供了准确、统一的接口，可以方便地使用各种机器学习算法；深度学习领域，Theano 是比较老牌的类库之一，特点是能够运行于不同的系统架构之上（CPU、GPU）；Tensorflow 则是最近较火的基础类库，使用它提供的各种算子和数据流工具，我们可以构建出多层神经网络，在集群上对大数据进行运算；Keras 则是一款较上层的工具库，底层使用 Theano 或 Tensorflow 作为引擎，可以通过快速构建实验来验证模型。

自然语言处理领域中，NLTK 提供了文本标记、分词、构建语料树等功能，用以揭示句中或句间的依赖关系；Gensim 则擅长构建向量空间模型、话题建模、挖掘大量文本中重复出现的模式，其算法都属于非监督学习，因此只需提供语料库就能得到结果。

原文：http://www.kdnuggets.com/2017/06/top-15-python-libraries-data-science.html

<!-- more -->

## 时间序列数据异常点检测算法

![](/cnblogs/images/digest/time-series-anomaly-detection.png)

* Types of anomalies
  * *Find outlier data points relative to some standard or usual signal.*
  * additive outlier: unexpected growth in a short period of time, a spike
  * temporal changes: zero or really low numbers
  * level shift: conversion rate drop in a funnel
  * two approaches: label a data point anomaly/not anomaly; forecast confidence interval
* STL decomposition
  * seasonal trend loess decomposition
  * split time series signals into three parts: seasonal, trend, residue
  * median absolute deviation threshold for residue, Generalized ESD test, [Twitter's library](https://github.com/twitter/AnomalyDetection)
  * pros: simple; good for additive outlier; use rolling average for level change;
  * cons: less tweaking options; not good for signals that change dramatically
* Classification and Regression Trees
  * teach trees to classify anomaly and non-anomaly data points; requires labeled datasets;
  * teach CART to predict the next data point, and check if the actual data actual lies inside the confidence interval with Generalized ESD or Grubbs' test; [xgboost library](https://github.com/dmlc/xgboost)
  * pros: introduce many feature parameters
  * cons: growing features damage performance
* ARIMA
  * use several points from the past to generate a forecast of the next point;
  * Box Jenkins method
  * cons: signals shouldn't depend on time
  * [tsoutliers](https://cran.r-project.org/web/packages/tsoutliers/tsoutliers.pdf) R package
* Exponential Smoothing
  * similar to ARIMA
  * Holt-Winters seasonal method
* Neural Networks
  * LSTM
  * multiple time series coupled with each other
* To Keep in Mind
  * try simplest model and algorithm that fit your problem the best
  * general solution is not always the best

原文：https://blog.statsbot.co/time-series-anomaly-detection-algorithms-1cef5519aef2

## 如何加入开源项目

![opensource](/cnblogs/images/digest/opensource.jpg)

已经蠢蠢欲动了吗？不如马上开始！找一个你正在使用的开源项目，Fork 源码仓库。先从修复 Bug 开始，或者为现有代码编写单元测试。过程中你需要学会读懂别人的代码，遵循代码规范。提交补丁后，你一定会受到猛烈的抨击，千万不要因此胆怯。熟悉代码之后，可以寻求一些更重要的职责，比如提出一个新的特性，或者认领新特性的开发工作。最后，你也可以开启自己的开源项目。

如何寻找自己想做贡献的开源项目？首先你可以在邮件列表、论坛、Bug 跟踪系统中找到这样的项目，问问自己是否喜欢这里的社区氛围。一开始不要在一棵树上吊死，多观察几个开源项目，找到自己对味的。从小处着手，比如重现 Bug、提交测试代码、改进文档等等。中途不要放弃！

最直接的加入方式其实是当正在使用的某个项目出现 Bug，或者你有用的不爽的地方，对它加以改进。此外，加入一个对开源项目友好的公司也是不错的选择。

原文：https://arstechnica.com/information-technology/2012/03/ask-stack-how-can-i-find-a-good-open-source-project-to-join/
