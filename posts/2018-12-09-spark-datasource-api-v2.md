---
title: Spark DataSource API V2
tags:
  - spark
categories: Big Data
date: 2018-12-09 11:10:59
---


Spark 1.3 引入了第一版的数据源 API，我们可以使用它将常见的数据格式整合到 Spark SQL 中。但是，随着 Spark 的不断发展，这一 API 也体现出了其局限性，故而 Spark 团队不得不加入越来越多的专有代码来编写数据源，以获得更好的性能。Spark 2.3 中，新一版的数据源 API 初见雏形，它克服了上一版 API 的种种问题，原来的数据源代码也在逐步重写。本文将演示这两版 API 的使用方法，比较它们的不同之处，以及新版 API 的优势在哪里。

## DataSource V1 API

V1 API 由一系列的抽象类和接口组成，它们位于 [spark/sql/sources/interfaces.scala][1] 文件中。主要的内容有：

```scala
trait RelationProvider {
  def createRelation(sqlContext: SQLContext, parameters: Map[String, String]): BaseRelation
}

abstract class BaseRelation {
  def sqlContext: SQLContext
  def schema: StructType
}

trait TableScan {
  def buildScan(): RDD[Row]
}
```

通过实现 `RelationProvider` 接口，表明该类是一种新定义的数据源，可以供 Spark SQL 取数所用。传入 `createRelation` 方法的参数可以用来做初始化，如文件路径、权限信息等。`BaseRelation` 抽象类则用来定义数据源的表结构，它的来源可以是数据库、Parquet 文件等外部系统，也可以直接由用户指定。该类还必须实现某个 `Scan` 接口，Spark 会调用 `buildScan` 方法来获取数据源的 RDD，我们将在下文看到。

<!-- more -->

### JdbcSourceV1

下面我们来使用 V1 API 实现一个通过 JDBC 读取数据库的自定义数据源。为简单起见，表结构信息是直接写死在代码里的，我们先从整表扫描开始。完整的代码可以在 GitHub（[链接][2]）中找到，数据源表则可以在这个 [链接][3] 中查看。

```scala
class JdbcSourceV1 extends RelationProvider {
  override def createRelation(parameters: Map[String, String]): BaseRelation = {
    new JdbcRelationV1(parameters("url"))
  }
}

class JdbcRelationV1(url: String) extends BaseRelation with TableScan {
  override def schema: StructType = StructType(Seq(
    StructField("id", IntegerType),
    StructField("emp_name", StringType)
  ))

  override def buildScan(): RDD[Row] = new JdbcRDD(url)
}

class JdbcRDD(url: String) extends RDD[Row] {
  override def compute(): Iterator[Row] = {
    val conn = DriverManager.getConnection(url)
    val stmt = conn.prepareStatement("SELECT * FROM employee")
    val rs = stmt.executeQuery()

    new Iterator[Row] {
      def hasNext: Boolean = rs.next()
      def next: Row = Row(rs.getInt("id"), rs.getString("emp_name"))
    }
  }
}
```

`JdbcRDD#compute` 负责实际的读取操作，它从上游获取到数据库连接信息、选取的字段、以及过滤条件，拼装 SQL 后执行，并返回一个 `Row` 类型的迭代器对象，每一行数据的结构和请求的字段列表相符。定义好数据源后，我们就可以用 `DataFrame` 对象来直接操作了：

```scala
val df = spark.read
  .format("JdbcSourceV2")
  .option("url", "jdbc:mysql://localhost/spark")
  .load()

df.printSchema()
df.show()
```

上述代码输出的内容是：

```text
root
 |-- id: integer (nullable = true)
 |-- emp_name: string (nullable = true)
 |-- dep_name: string (nullable = true)
 |-- salary: decimal(7,2) (nullable = true)
 |-- age: decimal(3,0) (nullable = true)

+---+--------+----------+-------+---+
| id|emp_name|  dep_name| salary|age|
+---+--------+----------+-------+---+
|  1| Matthew|Management|4500.00| 55|
|  2|  Olivia|Management|4400.00| 61|
|  3|   Grace|Management|4000.00| 42|
|  4|     Jim|Production|3700.00| 35|
|  5|   Alice|Production|3500.00| 24|
+---+--------+----------+-------+---+
```

### V1 API 的局限性

我们可以看到，V1 API 使用起来非常方便，因此能够满足 Spark SQL 初期的需求，但也不免存在很多局限性：

#### 依赖上层 API

`createRelation` 接收 `SQLContext` 作为参数；`buildScan` 方法返回的是 `RDD[Row]` 类型；而在实现写操作时，`insert` 方法会直接接收 `DataFrame` 类型的参数：

```scala
trait InsertableRelation {
  def insert(data: DataFrame, overwrite: Boolean): Unit
}
```

这些类型都属于较为上层的 Spark API，其中某些类已经发生了变化，如 `SQLContext` 已被 `SparkSession` 取代，而 `DataFrame` 也改为了 `Dataset[Row]` 类型的一个别称。这些改变不应该体现到底层的数据源 API 中。

#### 难以添加新的优化算子

除了上文中的 `TableScan` 接口，V1 API 还提供了 `PrunedScan` 接口，用来裁剪不需要的字段；`PrunedFilteredScan` 接口则可以将过滤条件下推到数据源中。在 `JdbcSourceV1` 示例中，这类下推优化会体现在 SQL 语句里：

```scala
class JdbcRelationV1 extends BaseRelation with PrunedFilteredScan {
  def buildScan(requiredColumns: Array[String], filters: Array[Filter]) = {
    new JdbcRDD(requiredColumns, filters)
  }
}

class JdbcRDD(columns: Array[String], filters: Array[Filter]) {
  def compute() = {
    val wheres = filters.flatMap {
      case EqualTo(attribute, value) => Some(s"$attribute = '$value'")
      case _ => None
    }
    val sql = s"SELECT ${columns.mkString(", ")} FROM employee WHERE ${wheres.mkString(" AND ")}"
  }
}
```

如果我们想添加新的优化算子（如 LIMIT 子句），就不免要引入一系列的 `Scan` 接口组合：

```scala
trait LimitedScan {
  def buildScan(limit: Int): RDD[Row]
}

trait PrunedLimitedScan {
  def buildScan(requiredColumns: Array[String], limit: Int): RDD[Row]
}

trait PrunedFilteredLimitedScan {
  def buildScan(requiredColumns: Array[String], filters: Array[Filter], limit: Int): RDD[Row]
}
```

#### 难以传递分区信息

对于支持数据分区的数据源，如 HDFS、Kafka 等，V1 API 没有提供原生的支持，因而也不能利用数据局部性（Data Locality）。我们需要自己继承 RDD 来实现，比如下面的代码就对 Kafka 数据源进行了分区，并告知 Spark 可以将数据读取操作放入 Kafka Broker 所在的服务器上执行，以提升效率：

```scala
case class KafkaPartition(partitionId: Int, leaderHost: String) extends Partition {
  def index: Int = partitionId
}

class KafkaRDD(sc: SparkContext) extends RDD[Row](sc, Nil) {
  def getPartitions: Array[Partition] = Array(
    // populate with Kafka PartitionInfo
    KafkaPartition(0, "broker_0"),
    KafkaPartition(1, "broker_1")
  )

  override def getPreferredLocations(split: Partition): Seq[String] = Seq(
    split.asInstanceOf[KafkaPartition].leaderHost
  )
}
```

此外，类似 Cassandra 这样的数据库，会按主键对数据进行分片。那么，如果查询语句中包含了按该主键进行分组的子句，Spark 就可以省去一次 Shuffle 操作。这在 V1 API 中也是不支持的，而 V2 API 则提供了 [`SupportsReportPartitioning`][6] 接口来支持。

#### 缺少事务性的写操作

Spark 任务是有可能失败的，使用 V1 API 时就会留下部分写入的数据。当然，对于 HDFS 这样的文件系统来说问题不大，因为可以用 `_SUCCESS` 来标记该次写操作是否执行成功。但这一逻辑也需要最终用户来实现，而 V2 API 则提供了明确的接口来支持事务性的写操作。

#### 缺少列存储和流式计算支持

Spark SQL 目前已支持列存储和流式计算，但两者都不是用 V1 API 实现的。`ParquetFileFormat` 和 `KafkaSource` 等类型都使用了专有代码和内部 API。这些特性也在 V2 API 中得到支持。

## DataSource V2 API

V2 API 首先使用了一个标记性的 `DataSourceV2` 接口，实现了该接口的类还必须实现 `ReadSupport` 或 `WriteSupport`，用来表示自身支持读或写操作。`ReadSupport` 接口中的方法会被用来创建 `DataSourceReader` 类，同时接收到初始化参数；该类继而创建 `DataReaderFactory` 和 `DataReader` 类，后者负责真正的读操作，接口中定义的方法和迭代器类似。此外，`DataSourceReader` 还可以实现各类 `Support` 接口，表明自己支持某些优化下推操作，如裁剪字段、过滤条件等。`WriteSupport` API 的层级结构与之类似。这些接口都是用 Java 语言编写，以获得更好的交互支持。

```java
public interface DataSourceV2 {}

public interface ReadSupport extends DataSourceV2 {
  DataSourceReader createReader(DataSourceOptions options);
}

public interface DataSourceReader {
  StructType readSchema();
  List<DataReaderFactory<Row>> createDataReaderFactories();
}

public interface SupportsPushDownRequiredColumns extends DataSourceReader {
  void pruneColumns(StructType requiredSchema);
}

public interface DataReaderFactory<T> {
  DataReader<T> createDataReader();
}

public interface DataReader<T> extends Closeable {
  boolean next();
  T get();
}
```

可能你会注意到，`DataSourceReader#createDataReaderFactories` 仍然捆绑了 `Row` 类型，这是因为目前 V2 API 只支持 `Row` 类型的返回值，且这套 API 仍处于进化状态（Evolving）。

### JdbcSourceV2

让我们使用 V2 API 来重写 JDBC 数据源。下面是一个整表扫描的示例，完整代码可以在 GitHub（[链接][4]）上查看。

```scala
class JdbcDataSourceReader extends DataSourceReader {
  def readSchema = StructType(Seq(
    StructField("id", IntegerType),
    StructField("emp_name", StringType)
  ))

  def createDataReaderFactories() = {
    Seq(new JdbcDataReaderFactory(url)).asJava
  }
}

class JdbcDataReader(url: String) extends DataReader[Row] {
  private var conn: Connection = null
  private var rs: ResultSet = null

  def next() = {
    if (rs == null) {
      conn = DriverManager.getConnection(url)
      val stmt = conn.prepareStatement("SELECT * FROM employee")
      rs = stmt.executeQuery()
    }
    rs.next()
  }

  def get() = Row(rs.getInt("id"), rs.getString("emp_name"))
}
```

#### 裁剪字段

通过实现 `SupportsPushDownRequiredColumns` 接口，Spark 会调用其 `pruneColumns` 方法，传入用户所指定的字段列表（`StructType`），`DataSourceReader` 可以将该信息传给 `DataReader` 使用。

```scala
class JdbcDataSourceReader with SupportsPushDownRequiredColumns {
  var requiredSchema = JdbcSourceV2.schema
  def pruneColumns(requiredSchema: StructType)  = {
    this.requiredSchema = requiredSchema
  }

  def createDataReaderFactories() = {
    val columns = requiredSchema.fields.map(_.name)
    Seq(new JdbcDataReaderFactory(columns)).asJava
  }
}
```

我们可以用 `df.explain(true)` 来验证执行计划。例如，`SELECT emp_name, age FROM employee` 语句的执行计划在优化前后是这样的：

```text
== Analyzed Logical Plan ==
emp_name: string, age: decimal(3,0)
Project [emp_name#1, age#4]
+- SubqueryAlias employee
   +- DataSourceV2Relation [id#0, emp_name#1, dep_name#2, salary#3, age#4], datasource.JdbcDataSourceReader@15ceeb42

== Optimized Logical Plan ==
Project [emp_name#1, age#4]
+- DataSourceV2Relation [emp_name#1, age#4], datasource.JdbcDataSourceReader@15ceeb42
```

可以看到，字段裁剪被反映到了数据源中。如果我们将实际执行的 SQL 语句打印出来，也能看到字段裁剪下推的结果。

#### 过滤条件

类似的，实现 `SupportsPushDownFilters` 接口可以将过滤条件下推到数据源中：

```scala
class JdbcDataSourceReader with SupportsPushDownFilters {
  var filters = Array.empty[Filter]
  var wheres = Array.empty[String]

  def pushFilters(filters: Array[Filter]) = {
    val supported = ListBuffer.empty[Filter]
    val unsupported = ListBuffer.empty[Filter]
    val wheres = ListBuffer.empty[String]

    filters.foreach {
      case filter: EqualTo => {
        supported += filter
        wheres += s"${filter.attribute} = '${filter.value}'"
      }
      case filter => unsupported += filter
    }

    this.filters = supported.toArray
    this.wheres = wheres.toArray
    unsupported.toArray
  }

  def pushedFilters = filters

  def createDataReaderFactories() = {
    Seq(new JdbcDataReaderFactory(wheres)).asJava
  }
}
```

#### 多分区支持

`createDataReaderFactories` 返回的是列表类型，每个读取器都会产生一个 RDD 分区。如果我们想开启多个读取任务，就可以生成多个读取器工厂，并为每个读取器限定主键范围：

```scala
def createDataReaderFactories() = {
  Seq((1, 6), (7, 11)).map { case (minId, maxId) =>
    val partition = s"id BETWEEN $minId AND $maxId"
    new JdbcDataReaderFactory(partition)
  }.asJava
}
```

### 事务性的写操作

V2 API 提供了两组 `commit` / `abort` 方法，用来实现事务性的写操作：

```java
public interface DataSourceWriter {
  void commit(WriterCommitMessage[] messages);
  void abort(WriterCommitMessage[] messages);
}

public interface DataWriter<T> {
  void write(T record) throws IOException;
  WriterCommitMessage commit() throws IOException;
  void abort() throws IOException;
}
```

`DataSourceWriter` 在 Spark Driver 中执行，`DataWriter` 则运行在其他节点的 Spark Executor 上。当 `DataWriter` 成功执行了写操作，就会将提交信息传递给 Driver；当 `DataSourceWriter` 收集到了所有写任务的提交信息，就会执行最终的提交操作。如果某个写任务失败了，它的 `abort` 方法会得到执行；如果经过多轮重试后仍然失败，则所有写任务的 `abort` 方法都会被调用，进行数据清理操作。

### 列存储与流式计算支持

这两个特性仍处于实验性阶段，在 Spark 中还没有得到使用。简单来说，`DataSourceReader` 类可以实现 `SupportsScanColumnarBatch` 接口来声明自己会返回 `ColumnarBatch` 对象，这个对象是 Spark 内部用来存放列式数据的。对于流式数据，则有 `MicroBatchReader` 和 `ContinuousReader` 这两个接口，感兴趣的读者可以到 Spark [单元测试][5] 代码中查看。

## 参考资料

* http://blog.madhukaraphatak.com/spark-datasource-v2-part-1/
* https://databricks.com/session/apache-spark-data-source-v2
* https://databricks.com/blog/2015/01/09/spark-sql-data-sources-api-unified-data-access-for-the-spark-platform.html
* https://developer.ibm.com/code/2018/04/16/introducing-apache-spark-data-sources-api-v2/

[1]: https://github.com/apache/spark/blob/v2.3.2/sql/core/src/main/scala/org/apache/spark/sql/sources/interfaces.scala
[2]: https://github.com/jizhang/spark-sandbox/blob/master/src/main/scala/datasource/JdbcExampleV1.scala
[3]: https://github.com/jizhang/spark-sandbox/blob/master/data/employee.sql
[4]: https://github.com/jizhang/spark-sandbox/blob/master/src/main/scala/datasource/JdbcExampleV2.scala
[5]: https://github.com/apache/spark/blob/v2.3.2/sql/core/src/test/scala/org/apache/spark/sql/sources/v2/DataSourceV2Suite.scala
[6]: https://github.com/apache/spark/blob/v2.3.2/sql/core/src/main/java/org/apache/spark/sql/sources/v2/reader/SupportsReportPartitioning.java
