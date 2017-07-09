title: "2017 Top 15 数据科学 Python 类库；时间序列异常点检测"
categories: Digest
tags: [data science, analytics]
---

## 2017 Top 15 数据科学 Python 类库

![Google Trends](https://cdn-images-1.medium.com/max/800/0*9pG2IJUPXSFZh-J0.png)

* Core libraries
  * NumPy: operate n-arrays and matrices, vectorization of mathematical operations, high performance;
  * SciPy: linear algebra, optimization, integration, and statistics;
  * Pandas: work with labeled for relational data, perfect for data wrangling, quick data manipulation, aggregation, and visualization;
* Visualization
  * Matplotlib: generate simple and powerful visualizations with ease; low level, need to write more code;
  * Seaborn: focus on visualization on statistial models, built on matplotlib;
  * Bokeh: aim at interactive visualization in modern browsers;
  * Plotly: web-based toolbox for building visualization; needs API access;
* Machine Learning
  * SciKit-Learn: expose a concise and consistent interface to common machine learning algorithms; de-facto industry standard;
* Deep Learning
  * Theano: similar to numpy, but can run on all architectures (CPU, GPU); faster and more precise results;
  * Tensorflow: data flow graphs computation; multi-layered nodes system that enables quick training of artificial neural network on large datasets;
  * Keras: high level interface for building neural networks; fast and easy experimentations;
* Natural Language Processing
  * NLTK: text tagging, tokenizing, building corpus tree that reveals inter and intra-sentence dependencies;
  * Gensim: vector space modeling and topic modeling; examine recurring patterns in large texts;
* Data Mining
  * Scrapy: general purpose scrawler, e.g. contact info, URL, API result;
  * Statsmodels: various methods of estimation of statistical models;

原文：http://www.kdnuggets.com/2017/06/top-15-python-libraries-data-science.html

<!-- more -->

## 时间序列数据异常点检测算法

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
