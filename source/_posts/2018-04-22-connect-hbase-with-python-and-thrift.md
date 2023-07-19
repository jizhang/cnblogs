---
title: 使用 Python 和 Thrift 连接 HBase
tags:
  - python
  - hbase
  - thrift
categories: Big Data
date: 2018-04-22 20:36:08
---


[Apache HBase][1] 是 Hadoop 生态环境中的键值存储系统（Key-value Store）。它构建在 HDFS 之上，可以对大型数据进行高速的读写操作。HBase 的开发语言是 Java，因此提供了原生的 Java 语言客户端。不过，借助于 Thrift 和其丰富的语言扩展，我们可以十分便捷地在任何地方调用 HBase 服务。文本将讲述的就是如何使用 Thrift 和 Python 来读写 HBase。

![](/images/hbase.png)

## 生成 Thrift 类定义

如果你对 [Apache Thrift][2] 并不熟悉，它提供了一套 IDL（接口描述语言），用于定义远程服务的方法签名和数据类型，并能将其转换成所需要的目标语言。举例来说，以下是用该 IDL 定义的一个数据结构：

```thrift
struct TColumn {
  1: required binary family,
  2: optional binary qualifier,
  3: optional i64 timestamp
}
```

转换后的 Python 代码是：

```python
class TColumn(object):
    def __init__(self, family=None, qualifier=None, timestamp=None,):
        self.family = family
        self.qualifier = qualifier
        self.timestamp = timestamp

    def read(self, iprot):
        iprot.readStructBegin()
        while True:
            (fname, ftype, fid) = iprot.readFieldBegin()
            # ...

    def write(self, oprot):
        oprot.writeStructBegin('TColumn')
        # ...
```

<!-- more -->

### HBase Thrift vs Thrift2

HBase 提供了 [两个版本][3] 的 IDL 文件，它们有以下两个不同点：

首先，`thrift2` 模仿了 HBase Java API 的数据类型和方法定义，调用方式更人性化一些。比如，构建一个 `Get` 操作的 Java 代码是：

```java
Get get = new Get(Bytes.toBytes("rowkey"));
get.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("col1"));
get.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("col2"));
```

在 `thrift2` 中有对应的 `TGet` 类型：

```python
tget = TGet(
    row='rowkey',
    columns=[
        TColumn(family='cf', qualifier='col1'),
        TColumn(family='cf', qualifier='col2'),
    ]
)
```

如果使用旧版的 `thrift`，我们就需要直接调用其众多的 `get` 方法之一了：

```python
client.getRowWithColumns(
    tableName='tbl',
    row='rowkey',
    columns=['cf:col1', 'cf:col2'],
    attributes=None
)
```

第二个不同点在于 `thrift2` 目前尚缺少 HBase 管理相关的接口，如 `createTable`、`majorCompact` 等。这些 API 仍在开发过程中，因此如果你需要通过 Thrift 来建表或维护 HBase，就只能使用旧版的 `thrift` 了。

决定了使用哪个版本的描述文件后，我们就可以将 `hbase.thrift` 下载到本地，通过它来生成 Python 代码。对于 Apache Thrift 本身的版本这里还要强调一点：由于我们使用的是 Python 3.x，而 Thrift 从 0.10 版本才开始支持，因此请确认自己安装了正确的版本。执行以下命令，我们就可以得到一组 Python 文件：

```bash
$ thrift -gen py hbase.thrift
$ find gen-py
gen-py/hbase/__init__.py
gen-py/hbase/constants.py
gen-py/hbase/THBaseService.py
gen-py/hbase/ttypes.py
```

## 在单机模式下运行 HBase

如果你手边没有可供测试的 HBase 服务，可以根据官网上的快速开始指引（[链接][4]），下载 HBase 二进制包，做一下简单的配合，并执行下列命令来启动 HBase 服务及 Thrift2 Server。

```bash
bin/start-hbase.sh
bin/hbase-daemon.sh start thrift2
bin/hbase shell
```

进入 HBase 命令行后，我们可以创建一个测试表，并尝试读写数据：

```ruby
> create "tsdata", NAME => "cf"
> put "tsdata", "sys.cpu.user:20180421:192.168.1.1", "cf:1015", "0.28"
> get "tsdata", "sys.cpu.user:20180421:192.168.1.1"
COLUMN                                        CELL
 cf:1015                                      timestamp=1524277135973, value=0.28
1 row(s) in 0.0330 seconds
```

## 通过 Thrift2 Server 连接 HBase

以下是创建 Thrift 连接的样板代码。需要注意的是，Thrift 客户端并不是线程安全的，因此无法在多个线程间共享。而且，它也没有提供类似连接池的特性。通常我们会选择每次查询都创建新的连接，当然你也可以引入自己的连接池机制。

```python
from thrift.transport import TSocket
from thrift.protocol import TBinaryProtocol
from thrift.transport import TTransport
from hbase import THBaseService

transport = TTransport.TBufferedTransport(TSocket.TSocket('127.0.0.1', 9090))
protocol = TBinaryProtocol.TBinaryProtocolAccelerated(transport)
client = THBaseService.Client(protocol)
transport.open()
# 使用 client 实例进行操作
transport.close()
```

我们来尝试编写几个基本的读写操作：

```python
from hbase.ttypes import TPut, TColumnValue, TGet
tput = TPut(
    row='sys.cpu.user:20180421:192.168.1.1',
    columnValues=[
        TColumnValue(family='cf', qualifier='1015', value='0.28'),
    ]
)
client.put('tsdata', tput)

tget = TGet(row='sys.cpu.user:20180421:192.168.1.1')
tresult = client.get('tsdata', tget)
for col in tresult.columnValues:
    print(col.qualifier, '=', col.value)
```

## Thrift2 数据类型和方法一览

完整的方法列表可以直接查阅 `hbase.thrift` 和 `hbase/THBaseService.py` 这两个文件。下面是对常用方法的总结：

### 数据类型

| 类名 | 描述 | 示例 |
| --- | --- | --- |
| TColumn | 表示一个列族或单个列。 | TColumn(family='cf', qualifier='gender') |
| TColumnValue | 列名及其包含的值。 | TColumnValue(family='cf', qualifier='gender', value='male') |
| TResult | 查询结果（一行）。若 `row` 属性的值为 `None`，则表示查无结果。 | TResult(row='employee_001', columnValues=[TColumnValue]) |
| TGet | 查询单行。 | TGet(row='employee_001', columns=[TColumn]) |
| TPut | 修改一行数据. | TPut(row='employee_001', columnValues=[TColumnValue]) |
| TDelete | 删除整行或部分列。 | TDelete(row='employee_001', columns=[TColumn]) |
| TScan | 扫描多行数据。 | 见下文 |

### THBaseService 类方法

| 方法签名 | 描述 |
| --- | --- |
| get(table: str, tget: TGet) -> TResult | 查询单行。 |
| getMultiple(table: str, tgets: List[TGet]) -> List[TResult] | 查询多行。 |
| put(table: str, tput: TPut) -> None | 修改单行。 |
| putMultiple(table: str, tputs: List[TPut]) -> None | 修改多行。 |
| deleteSingle(table: str, tdelete: TDelete) -> None | 删除单行。 |
| deleteMultiple(table: str, tdeletes: List[TDelete]) -> None | 删除多行。 |
| openScanner(table: str, tscan: TScan) -> int | 打开一个扫描器，返回其唯一标识。 |
| getScannerRows(scannerId: int, numRows: int) -> List[TResult] | 返回扫描结果。 |
| closeScanner(scannerId: int) -> None | 关闭扫描器。 |
| getScannerResults(table: str, tscan: TScan, numRows: int) -> List[TResult] | 直接获取扫描结果的快捷方法。 |

### Scan 操作示例

我在 GitHub（[链接][5]）上放置了一些样例代码，以下是 `Scan` 操作的样例：

```python
scanner_id = client.openScanner(
    table='tsdata',
    tscan=TScan(
        startRow='sys.cpu.user:20180421',
        stopRow='sys.cpu.user:20180422',
        columns=[TColumn('cf', '1015')]
    )
)
try:
    num_rows = 10
    while True:
        tresults = client.getScannerRows(scanner_id, num_rows)
        for tresult in tresults:
            print(tresult)
        if len(tresults) < num_rows:
            break
finally:
    client.closeScanner(scanner_id)
```

## Thrift Server 高可用

Thrift Server 的单点问题有几种解决方案：

1. 在客户端中配置多个 Thrift Server 地址，发送请求时随机选择一个，并做好错误重试；
2. 搭建代理，对 TCP 连接做负载均衡；
3. 在客户端服务器上配置独立的 Thrift Server，每个客户端直接创建本地连接。

通常我们会选择第二种方案，这就需要和运维工程师一起配合搭建了。

![](/images/hbase-thrift-ha.png)

## 参考资料

* https://blog.cloudera.com/blog/2013/09/how-to-use-the-hbase-thrift-interface-part-1/
* https://thrift.apache.org/tutorial/py
* https://yq.aliyun.com/articles/88299
* http://opentsdb.net/docs/build/html/user_guide/backends/hbase.html


[1]: https://hbase.apache.org/
[2]: https://thrift.apache.org/
[3]: https://github.com/apache/hbase/tree/master/hbase-thrift/src/main/resources/org/apache/hadoop/hbase
[4]: https://hbase.apache.org/book.html#quickstart
[5]: https://github.com/jizhang/blog-demo/tree/master/python-hbase
