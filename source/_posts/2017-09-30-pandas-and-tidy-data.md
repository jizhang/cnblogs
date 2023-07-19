---
title: Pandas 与数据整理
tags:
  - pandas
  - analytics
  - python
categories:
  - Big Data
date: 2017-09-30 14:37:56
---

在 [Tidy Data][1] 论文中，[Wickham 博士][2] 提出了这样一种“整洁”的数据结构：每个变量是一列，每次观测结果是一行，不同的观测类型存放在单独的表中。他认为这样的数据结构可以帮助分析师更简单高效地进行处理、建模、和可视化。他在论文中列举了 *五种* 不符合整洁数据的情况，并演示了如何通过 [R 语言][3] 对它们进行整理。本文中，我们将使用 Python 和 Pandas 来达到同样的目的。

文中的源代码和演示数据可以在 GitHub（[链接][4]）上找到。读者应该已经安装好 Python 开发环境，推荐各位使用 Anaconda 和 Spyder IDE。

## 列名称是数据值，而非变量名

```python
import pandas as pd
df = pd.read_csv('data/pew.csv')
df.head(10)
```

![宗教信仰与收入 - Pew 论坛](/images/tidy-data/pew.png)

表中的列“<$10k”、“$10-20k”其实是“收入”变量的具体值。*变量* 是指某一特性的观测值，如身高、体重，本例中则是收入、宗教信仰。表中的数值数据构成了另一个变量——人数。要做到 *每个变量是一列* ，我们需要进行以下变换：

```python
df = df.set_index('religion')
df = df.stack()
df.index = df.index.rename('income', level=1)
df.name = 'frequency'
df = df.reset_index()
df.head(10)
```

![宗教信仰与收入 - 整洁版](/images/tidy-data/pew-tidy.png)

<!-- more -->

这里我们使用了 Pandas 多级索引的 [stack / unstack][5] 特性。`stack()` 会将列名转置为新一级的索引，并将数据框（DataFrame）转换成序列（Series）。转置后，我们对行和列的名称做一些调整，再用 `reset_index()` 将数据框还原成普通的二维表。

除了使用多级索引，Pandas 还提供了另一种更为便捷的方法——[`melt()`][6]。该方法接收以下参数：

* `frame`: 需要处理的数据框；
* `id_vars`: 保持原样的数据列；
* `value_vars`: 需要被转换成变量值的数据列；
* `var_name`: 转换后变量的列名；
* `value_name`: 数值变量的列名。

```python
df = pd.read_csv('data/pew.csv')
df = pd.melt(df, id_vars=['religion'], value_vars=list(df.columns)[1:],
             var_name='income', value_name='frequency')
df = df.sort_values(by='religion')
df.to_csv('data/pew-tidy.csv', index=False)
df.head(10)
```

这段代码会输出相同的结果，下面的示例中我们都将使用 `melt()` 方法。我们再来看另外一个案例：

![Billboard 2000](/images/tidy-data/billboard.png)

在这个数据集中，每周的排名都被记录到了不同的数据列中。如果我们想要回答“Dancing Queen 这首歌在 2000年7月15日 的排名如何”，就需要结合 `date.entered` 字段做一些运算才行。下面我们来对这份数据进行整理：

```python
df = pd.read_csv('data/billboard.csv')
df = pd.melt(df, id_vars=list(df.columns)[:5], value_vars=list(df.columns)[5:],
             var_name='week', value_name='rank')
df['week'] = df['week'].str[2:].astype(int)
df['date.entered'] = pd.to_datetime(df['date.entered']) + pd.to_timedelta((df['week'] - 1) * 7, 'd')
df = df.rename(columns={'date.entered': 'date'})
df = df.sort_values(by=['track', 'date'])
df.to_csv('data/billboard-intermediate.csv', index=False)
df.head(10)
```

![Billboard 2000 - 中间版](/images/tidy-data/billboard-intermediate.png)

上述代码中，我们还将 `date.entered` 转换成了每一周的具体日期，`week` 字段也作为单独的数据列进行存储。但是，我们会在表中看到很多重复的信息，如歌手、曲名等，我们将在第四节解决这个问题。

## 一列包含多个变量

人们之所以会将变量值作为列名，一方面是这样的表示方法更为紧凑、可以在一页中显示更多信息，还有一点是这种格式便于做交叉验证等数据分析工作。下面的数据集更是将性别和年龄这两个变量都放入了列名中：

![结核病 (TB)](/images/tidy-data/tb.png)

`m` 表示男性（Male），`f` 表示女性（Female），`0-14`、`15-24` 则表示年龄段。进行数据整理时，我们先用 Pandas 的字符串处理功能截取 `sex` 字段，再对剩余表示年龄段的子串做映射处理。

```python
df = pd.read_csv('data/tb.csv')
df = pd.melt(df, id_vars=['country', 'year'], value_vars=list(df.columns)[2:],
             var_name='column', value_name='cases')
df = df[df['cases'] != '---']
df['cases'] = df['cases'].astype(int)
df['sex'] = df['column'].str[0]
df['age'] = df['column'].str[1:].map({
    '014': '0-14',
    '1524': '15-24',
    '2534': '25-34',
    '3544': '35-44',
    '4554': '45-54',
    '5564': '55-64',
    '65': '65+'
})
df = df[['country', 'year', 'sex', 'age', 'cases']]
df.to_csv('data/tb-tidy.csv', index=False)
df.head(10)
```

![结核病 (TB) - 整洁版](/images/tidy-data/tb-tidy.png)

## 变量存储在行和列中

下表是一个名为 MX17004 的气象站收集的温度数据。可以看到，日期被放置在列名中，我们可以用 `melt` 进行处理；`tmax` 和 `tmin` 则表示最高温度和最低温度，他们很显然是两个不同的变量，用来衡量单个观测对象的属性的，本例中的观测对象是“天”。因此，我们需要使用 `unstack` 将其拆分成两列。

![气象站](/images/tidy-data/weather.png)

```python
df = pd.read_csv('data/weather.csv')
df = pd.melt(df, id_vars=['id', 'year', 'month', 'element'],
             value_vars=list(df.columns)[4:],
             var_name='date', value_name='value')
df['date'] = df['date'].str[1:].astype('int')
df['date'] = df[['year', 'month', 'date']].apply(
    lambda row: '{:4d}-{:02d}-{:02d}'.format(*row),
    axis=1)
df = df.loc[df['value'] != '---', ['id', 'date', 'element', 'value']]
df = df.set_index(['id', 'date', 'element'])
df = df.unstack()
df.columns = list(df.columns.get_level_values('element'))
df = df.reset_index()
df.to_csv('data/weather-tidy.csv', index=False)
df
```

![气象站 - 整洁版](/images/tidy-data/weather-tidy.png)

## 同一表中包含多种观测类型

在处理 Billboard 数据集时，我们会看到冗余的曲目信息，这是因为该表实际记录的是两种不同的观测类型——歌曲曲目和周排名。整理时，我们需要先为每首歌曲生成一个唯一标识，即 `id`，然后拆分到单独的表中。

```python
df = pd.read_csv('data/billboard-intermediate.csv')
df_track = df[['artist', 'track', 'time']].drop_duplicates()
df_track.insert(0, 'id', range(1, len(df_track) + 1))
df = pd.merge(df, df_track, on=['artist', 'track', 'time'])
df = df[['id', 'date', 'rank']]
df_track.to_csv('data/billboard-track.csv', index=False)
df.to_csv('data/billboard-rank.csv', index=False)
print(df_track, '\n\n', df)
```

![Billboard 2000 - 歌曲](/images/tidy-data/billboard-track.png)

![Billboard 2000 - 排名](/images/tidy-data/billboard-rank.png)

## 同一观测类型分布在不同表中

原始的数据集可能会以两种方式进行了拆分，一种是按照某个变量拆分，如按年拆分为2000年、2001年，按地理位置拆分为中国、英国；另一种是按不同的属性拆分，如一份数据是收集温度的传感器记录的，另一份是湿度传感器，他们记录的都是每一天的观测值。对于第一种情况，我们可以编写一个读取数据的函数，遍历目录中的文件，并将文件名作为单独的列加入数据框，最后使用 [`pd.concat`][7] 进行合并；第二种情况则要求数据集中的记录有一个唯一标识，如日期、身份证号，并通过 [`pd.merge`][8] 将各个数据集联系起来。

## 参考资料

* https://tomaugspurger.github.io/modern-5-tidy.html
* https://hackernoon.com/reshaping-data-in-python-fa27dda2ff77
* http://www.jeannicholashould.com/tidy-data-in-python.html

[1]: https://www.jstatsoft.org/article/view/v059i10
[2]: https://en.wikipedia.org/wiki/Hadley_Wickham
[3]: https://github.com/hadley/tidy-data/
[4]: https://github.com/jizhang/blog-demo/tree/master/pandas-tidy-data
[5]: https://pandas.pydata.org/pandas-docs/stable/reshaping.html
[6]: https://pandas.pydata.org/pandas-docs/stable/generated/pandas.melt.html
[7]: https://pandas.pydata.org/pandas-docs/stable/generated/pandas.concat.html
[8]: https://pandas.pydata.org/pandas-docs/stable/generated/pandas.merge.html
