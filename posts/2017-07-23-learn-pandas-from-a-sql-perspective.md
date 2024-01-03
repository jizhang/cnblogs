---
title: 通过 SQL 查询学习 Pandas 数据处理
tags:
  - pandas
  - sql
  - analytics
  - python
categories:
  - Big Data
date: 2017-07-23 20:57:00
---


[Pandas](http://pandas.pydata.org/) 是一款广泛使用的数据处理工具。结合 NumPy 和 Matplotlib 类库，我们可以在内存中进行高性能的数据清洗、转换、分析及可视化工作。虽然 Python 本身是一门非常容易学习的语言，但要熟练掌握 Pandas 丰富的 API 接口及正确的使用方式，还是需要投入一定时间的。对于数据开发工程师或分析师而言，SQL 语言是标准的数据查询工具。本文提供了一系列的示例，如何将常见的 SQL 查询语句使用 Pandas 来实现。

Pandas 的安装和基本概念并不在本文讲述范围内，请读者到官网上阅读相关文档，或者阅读《[利用 Python 进行数据分析][1]》一书。我推荐大家使用 [Anaconda][2] Python 套件，其中集成了 [Spyder][3] 集成开发环境。在运行下文的代码之前，请先引入 Pandas 和 NumPy 包：

```python
import pandas as pd
import numpy as np
```

## `FROM` - 读取数据

首先，我们需要将数据加载到工作区（内存）。Pandas 原生支持非常多的数据格式，CSV 是较常见的一种。我们以航班延误时间数据集为例（[下载地址](/uploads/flights.csv)）：

```csv
date,delay,distance,origin,destination
02221605,3,358,BUR,SMF
01022100,-5,239,HOU,DAL
03210808,6,288,BWI,ALB
```

我们可以使用 `pd.read_csv` 函数加载它：

```python
df = pd.read_csv('flights.csv', dtype={'date': str})
df.head()
```

这条命令会将 `flights.csv` 文件读入内存，使用首行作为列名，并自动检测每一列的数据类型。其中，由于 `date` 一列的日期格式是 `%m%d%H%M`，自动转换成数字后会失去月份的前异零（02 月的 0），因此我们显式指定了该列的 `dtype`，告知 Pandas 保留原值。

<!-- more -->

`df.head` 用于查看数据集的前 N 行，功能类似于 `LIMIT N`。如果要实现 `LIMIT 10, 100`，可以使用 `df.iloc[10:100]`。此外，IPython 终端默认只显示 60 行数据，我们可以通过以下方法修改设置：

```python
pd.options.display.max_rows = 100
df.iloc[10:100]
```

另外一种常见的数据源是关系型数据库，Pandas 也提供了内置支持：

```python
conn = pymysql.connect(host='localhost', user='root')
df = pd.read_sql("""
select `date`, `delay`, `distance`, `origin`, `destination`
from flights limit 1000
""", conn)
```

如果要将 DataFrame 保存到文件或数据库中去，可以分别使用 `pd.to_csv` 和 `pd.to_sql` 函数。

## `SELECT` - 选择列

`SELECT` 语句在 SQL 中用于选择需要的列，并对数据做清洗和转换。

```python
df['date'] # SELECT `date`
df[['date', 'delay']] # SELECT `date`, `delay`
df.loc[10:100, ['date', 'delay']] # SELECT `date, `delay` LIMIT 10, 100
```

SQL 提供了诸多函数，大部分都可以用 Pandas 来实现，而且我们也很容易用 Python 编写自定义函数。下面我将列举一些常用的函数。

### 字符串函数

Pandas 的字符串函数可以通过 DateFrame 和 Series 的 `str` 属性来调用，如 `df['origin'].str.lower()`。

```python
# SELECT CONCAT(origin, ' to ', destination)
df['origin'].str.cat(df['destination'], sep=' to ')

df['origin'].str.strip() # TRIM(origin)
df['origin'].str.len() # LENGTH(origin)
df['origin'].str.replace('a', 'b') # REPLACE(origin, 'a', 'b')

# SELECT SUBSTRING(origin, 1, 1)
df['origin'].str[0:1] # 使用 Python 字符串索引

# SELECT SUBSTRING_INDEX(domain, '.', 2)
# www.example.com -> www.example
df['domain'].str.split('.').str[:2].str.join('.')
df['domain'].str.extract(r'^([^.]+\.[^.]+)')
```

Pandas 有一个名为广播的特性（broadcast），简单来说就是能够将低维数据（包括单个标量）和高维数据进行结合和处理。例如：

```python
df['full_date'] = '2001' + df['date'] # CONCAT('2001', `date`)
df['delay'] / 60
df['delay'].div(60) # 同上
```

Pandas 还内置了很多字符串函数，它们的用法和 SQL 有一定区别，但功能更强。完整列表可以参考文档 [Working with Text Data][4]。

### 日期函数

`pd.to_datetime` 用于将各种日期字符串转换成标准的 `datetime64` 类型。日期类型的 Series 都会有一个 `dt` 属性，从中可以获取到有关日期时间的信息，具体请参考文档 [Time Series / Date functionality][5]。

```python
# SELECT STR_TO_DATE(full_date, '%Y%m%d%H%i%s') AS `datetime`
df['datetime'] = pd.to_datetime(df['full_date'], format='%Y%m%d%H%M%S')

# SELECT DATE_FORMAT(`datetime`, '%Y-%m-%d')
df['datetime'].dt.strftime('%Y-%m-%d')

df['datetime'].dt.month # MONTH(`datetime`)
df['datetime'].dt.hour # HOUR(`datetime`)

# SELECT UNIX_TIMESTAMP(`datetime`)
df['datetime'].view('int64') // pd.Timedelta(1, unit='s').value

# SELECT FROM_UNIXTIME(`timestamp`)
pd.to_datetime(df['timestamp'], unit='s')

# SELECT `datetime` + INTERVAL 1 DAY
df['datetime'] + pd.Timedelta(1, unit='D')
```

## `WHERE` - 选择行

在 Pandas 中使用逻辑表达式后，会返回一个布尔型的 Series，通过它可以对数据集进行过滤：

```python
(df['delay'] > 0).head()
# 0  True
# 1 False
# 2  True
# dtype: bool

# WHERE delay > 0
df[df['delay'] > 0]
```

我们可以用位运算符来组合多个查询条件：

```python
# WHERE delay > 0 AND distance <= 500
df[(df['delay'] > 0) & (df['distance'] <= 500)]

# WHERE delay > 0 OR origin = 'BUR'
df[(df['delay'] > 0) | (df['origin'] == 'BUR')]

# WHERE NOT (delay > 0)
df[~(df['delay'] > 0)]
```

对于 `IS NULL` 和 `IS NOT NULL`，也提供了相应的内置函数：

```python
df[df['delay'].isnull()] # delay IS NULL
df[df['delay'].notnull()] # delay IS NOT NUL
```

此外，Pandas 还提供了 `df.query` 方法，可以使用字符串表达式来编写过滤条件：

```python
df.query('delay > 0 and distaince <= 500')
df.query('(delay > 0) | (origin == "BUR")')
```

其实，Pandas 提供了功能强大的数据选取工具，很多是无法用 SQL 表达出来的，建议详细阅读 [Indexing and Selecting Data][6] 文档，其中包含了丰富的示例。

## `GROUP BY` - 汇总

```python
# SELECT origin, COUNT(*) FROM flights GROUP BY origin
df.groupby('origin').size()
# origin
# ABQ    22
# ALB     4
# AMA     4
# dtype: int64
```

聚合运算包含了两个部分，一是分组字段，二是聚合函数。我们可以传递多个分组字段给 `df.groupby`，也能够指定多个聚合函数：

```python
# SELECT origin, destination, SUM(delay), AVG(distance)
# GROUP BY origin, destination
df.groupby(['origin', 'destination']).agg({
    'delay': np.sum,
    'distance': np.mean
})

# SELECT origin, MIN(delay), MAX(delay) GROUP BY origin
df.groupby('origin')['delay'].agg(['min', 'max'])
```

我们还可以将函数的运行结果作为分组条件。更多示例请见 [Group By: split-apply-combine][7]。

```python
# SELECT LENGTH(origin), COUNT(*) GROUP BY LENGTH(origin)
df.set_index('origin').groupby(len).size()
```

## `ORDER BY` - 排序

Pandas 中有两类排序，按索引和按数值。如果不了解 Pandas 的索引，还请自行查阅相关教程。

```python
# ORDER BY origin
df.set_index('origin').sort_index()
df.sort_values(by='origin')

# ORDER BY origin ASC, destination DESC
df.sort_values(by=['origin', 'destination'], ascending=[True, False])
```

## `JOIN` - 关联查询

```python
# FROM product a LEFT JOIN category b ON a.cid = b.id
pd.merge(df_product, df_category, left_on='cid', right_on='id', how='left')
```

如果联合查询的键是同名的，可以直接使用 `on=['k1', 'k2']`。默认的关联方式是 `INNER JOIN`（`how='inner'`），其它还有左外连接（`left`）、右外连接（`right`）、以及 `FULL OUTER JOIN`（`outer`)。

`pd.concat` 可用于实现 `UNION` 查询。 更多关联查询的示例请参考 [Merge, join, and concatenate][8]。

```python
# SELECT * FROM a UNION SELECT * FROM b
pd.concat([df_a, df_b]).drop_duplicates()
```

# 分组排名

最后，我们经常会需要在分组中按某种规则排序，并获得前几位的记录。MySQL 中需要通过变量来实现，Pandas 中则可以使用 `rank` 函数：

```python
rnk = df.groupby('origin')['delay'].rank(method='first', ascending=False)
df.assign(rnk=rnk).query('rnk <= 3').sort_values(['origin', 'rnk'])
```

## 参考资料

* https://pandas.pydata.org/pandas-docs/stable/comparison_with_sql.html
* http://www.gregreda.com/2013/01/23/translating-sql-to-pandas-part1/
* http://codingsight.com/pivot-tables-in-mysql/

[1]: https://book.douban.com/subject/25779298/
[2]: https://www.continuum.io/downloads
[3]: https://pythonhosted.org/spyder/
[4]: https://pandas.pydata.org/pandas-docs/stable/text.html
[5]: https://pandas.pydata.org/pandas-docs/stable/timeseries.html
[6]: https://pandas.pydata.org/pandas-docs/stable/indexing.html
[7]: https://pandas.pydata.org/pandas-docs/stable/groupby.html
[8]: https://pandas.pydata.org/pandas-docs/stable/merging.html
