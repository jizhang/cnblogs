---
title: 深入理解 Hive ACID 事务表
tags:
  - hive
  - hadoop
categories: Big Data
date: 2019-06-10 22:40:55
---

[Apache Hive][1] 0.13 版本引入了事务特性，能够在 Hive 表上实现 ACID 语义，包括 INSERT/UPDATE/DELETE/MERGE 语句、增量数据抽取等。Hive 3.0 又对该特性进行了优化，包括改进了底层的文件组织方式，减少了对表结构的限制，以及支持条件下推和向量化查询。Hive 事务表的介绍和使用方法可以参考 [Hive Wiki][2] 和 [各类教程][3]，本文将重点讲述 Hive 事务表是如何在 HDFS 上存储的，及其读写过程是怎样的。

## 目录结构

### 插入数据

```sql
CREATE TABLE employee (id int, name string, salary int)
STORED AS ORC TBLPROPERTIES ('transactional' = 'true');

INSERT INTO employee VALUES
(1, 'Jerry', 5000),
(2, 'Tom',   8000),
(3, 'Kate',  6000);
```

INSERT 语句会在一个事务中运行。它会创建名为 `delta` 的目录，存放事务的信息和表的数据。

```text
/user/hive/warehouse/employee/delta_0000001_0000001_0000
/user/hive/warehouse/employee/delta_0000001_0000001_0000/_orc_acid_version
/user/hive/warehouse/employee/delta_0000001_0000001_0000/bucket_00000
```

目录名称的格式为 `delta_minWID_maxWID_stmtID`，即 delta 前缀、写事务的 ID 范围、以及语句 ID。具体来说：

* 所有 INSERT 语句都会创建 `delta` 目录。UPDATE 语句也会创建 `delta` 目录，但会先创建一个 `delete` 目录，即先删除、后插入。`delete` 目录的前缀是 delete_delta；
* Hive 会为所有的事务生成一个全局唯一的 ID，包括读操作和写操作。针对写事务（INSERT、DELETE 等），Hive 还会创建一个写事务 ID（Write ID），该 ID 在表范围内唯一。写事务 ID 会编码到 `delta` 和 `delete` 目录的名称中；
* 语句 ID（Statement ID）则是当一个事务中有多条写入语句时使用的，用作唯一标识。

<!-- more -->

再看文件内容，`_orc_acid_version` 的内容是 2，即当前 ACID 版本号是 2。它和版本 1 的主要区别是 UPDATE 语句采用了 split-update 特性，即上文提到的先删除、后插入。这个特性能够使 ACID 表支持条件下推等功能，具体可以查看 [HIVE-14035][4]。`bucket_00000` 文件则是写入的数据内容。由于这张表没有分区和分桶，所以只有这一个文件。事务表都以 [ORC][5] 格式存储的，我们可以使用 [orc-tools][6] 来查看文件的内容：

```text
$ orc-tools data bucket_00000
{"operation":0,"originalTransaction":1,"bucket":536870912,"rowId":0,"currentTransaction":1,"row":{"id":1,"name":"Jerry","salary":5000}}
{"operation":0,"originalTransaction":1,"bucket":536870912,"rowId":1,"currentTransaction":1,"row":{"id":2,"name":"Tom","salary":8000}}
{"operation":0,"originalTransaction":1,"bucket":536870912,"rowId":2,"currentTransaction":1,"row":{"id":3,"name":"Kate","salary":6000}}
```

输出内容被格式化为了一行行的 JSON 字符串，我们可以看到具体数据是在 `row` 这个键中的，其它键则是 Hive 用来实现事务特性所使用的，具体含义为：

* `operation` 0 表示插入，1 表示更新，2 表示删除。由于使用了 split-update，UPDATE 是不会出现的；
* `originalTransaction` 是该条记录的原始写事务 ID。对于 INSERT 操作，该值和 `currentTransaction` 是一致的。对于 DELETE，则是该条记录第一次插入时的写事务 ID；
* `bucket` 是一个 32 位整型，由 `BucketCodec` 编码，各个二进制位的含义为：
    * 1-3 位：编码版本，当前是 `001`；
    * 4 位：保留；
    * 5-16 位：分桶 ID，由 0 开始。分桶 ID 是由 CLUSTERED BY 子句所指定的字段、以及分桶的数量决定的。该值和 `bucket_N` 中的 N 一致；
    * 17-20 位：保留；
    * 21-32 位：语句 ID；
    * 举例来说，整型 `536936448` 的二进制格式为 `00100000000000010000000000000000`，即它是按版本 1 的格式编码的，分桶 ID 为 1；
* `rowId` 是一个自增的唯一 ID，在写事务和分桶的组合中唯一；
* `currentTransaction` 当前的写事务 ID；
* `row` 具体数据。对于 DELETE 语句，则为 `null`。

我们可以注意到，文件中的数据会按 (`originalTransaction`, `bucket`, `rowId`) 进行排序，这点对后面的读取操作非常关键。

这些信息还可以通过 `row__id` 这个虚拟列进行查看：

```sql
SELECT row__id, id, name, salary FROM employee;
```

输出结果为：

```text
{"writeid":1,"bucketid":536870912,"rowid":0}    1       Jerry   5000
{"writeid":1,"bucketid":536870912,"rowid":1}    2       Tom     8000
{"writeid":1,"bucketid":536870912,"rowid":2}    3       Kate    6000
```

#### 增量数据抽取 API V2

Hive 3.0 还改进了先前的 [增量抽取 API][7]，通过这个 API，用户或第三方工具（Flume 等）就可以利用 ACID 特性持续不断地向 Hive 表写入数据了。这一操作同样会生成 `delta` 目录，但更新和删除操作不再支持。

```java
StreamingConnection connection = HiveStreamingConnection.newBuilder().connect();
connection.beginTransaction();
connection.write("11,val11,Asia,China".getBytes());
connection.write("12,val12,Asia,India".getBytes());
connection.commitTransaction();
connection.close();
```

### 更新数据

```sql
UPDATE employee SET salary = 7000 WHERE id = 2;
```

这条语句会先查询出所有符合条件的记录，获取它们的 `row__id` 信息，然后分别创建 `delete` 和 `delta` 目录：

```text
/user/hive/warehouse/employee/delta_0000001_0000001_0000/bucket_00000
/user/hive/warehouse/employee/delete_delta_0000002_0000002_0000/bucket_00000
/user/hive/warehouse/employee/delta_0000002_0000002_0000/bucket_00000
```

`delete_delta_0000002_0000002_0000/bucket_00000` 包含了删除的记录：

```text
{"operation":2,"originalTransaction":1,"bucket":536870912,"rowId":1,"currentTransaction":2,"row":null}
```

`delta_0000002_0000002_0000/bucket_00000` 包含更新后的数据：

```text
{"operation":0,"originalTransaction":2,"bucket":536870912,"rowId":0,"currentTransaction":2,"row":{"id":2,"name":"Tom","salary":7000}}
```

DELETE 语句的工作方式类似，同样是先查询，后生成 `delete` 目录。

### 合并表

MERGE 语句和 MySQL 的 INSERT ON UPDATE 功能类似，它可以将来源表的数据合并到目标表中：

```sql
CREATE TABLE employee_update (id int, name string, salary int);
INSERT INTO employee_update VALUES
(2, 'Tom',  7000),
(4, 'Mary', 9000);

MERGE INTO employee AS a
USING employee_update AS b ON a.id = b.id
WHEN MATCHED THEN UPDATE SET salary = b.salary
WHEN NOT MATCHED THEN INSERT VALUES (b.id, b.name, b.salary);
```

这条语句会更新 Tom 的薪资字段，并插入一条 Mary 的新记录。多条 WHEN 子句会被视为不同的语句，有各自的语句 ID（Statement ID）。INSERT 子句会创建 `delta_0000002_0000002_0000` 文件，内容是 Mary 的数据；UPDATE 语句则会创建 `delete_delta_0000002_0000002_0001` 和 `delta_0000002_0000002_0001` 两个文件，删除并新增 Tom 的数据。

```text
/user/hive/warehouse/employee/delta_0000001_0000001_0000
/user/hive/warehouse/employee/delta_0000002_0000002_0000
/user/hive/warehouse/employee/delete_delta_0000002_0000002_0001
/user/hive/warehouse/employee/delta_0000002_0000002_0001
```

### 压缩

随着写操作的积累，表中的 `delta` 和 `delete` 文件会越来越多。事务表的读取过程中需要合并所有文件，数量一多势必会影响效率。此外，小文件对 HDFS 这样的文件系统也是不够友好的。因此，Hive 引入了压缩（Compaction）的概念，分为 Minor 和 Major 两类。

Minor Compaction 会将所有的 `delta` 文件压缩为一个文件，`delete` 也压缩为一个。压缩后的结果文件名中会包含写事务 ID 范围，同时省略掉语句 ID。压缩过程是在 Hive Metastore 中运行的，会根据一定阈值自动触发。我们也可以使用如下语句人工触发：

```sql
ALTER TABLE employee COMPACT 'minor';
```

以上文中的 MERGE 语句的结果举例，在运行了一次 Minor Compaction 后，文件目录结构将变为：

```text
/user/hive/warehouse/employee/delete_delta_0000001_0000002
/user/hive/warehouse/employee/delta_0000001_0000002
```

在 `delta_0000001_0000002/bucket_00000` 文件中，数据会被排序和合并起来，因此文件中将包含两行 Tom 的数据。Minor Compaction 不会删除任何数据。

Major Compaction 则会将所有文件合并为一个文件，以 `base_N` 的形式命名，其中 N 表示最新的写事务 ID。已删除的数据将在这个过程中被剔除。`row__id` 则按原样保留。

```text
/user/hive/warehouse/employee/base_0000002
```

需要注意的是，在 Minor 或 Major Compaction 执行之后，原来的文件不会被立刻删除。这是因为删除的动作是在另一个名为 Cleaner 的线程中执行的。因此，表中可能同时存在不同事务 ID 的文件组合，这在读取过程中需要做特殊处理。

## 读取过程

Now we see three kinds of files in an ACID table, `base`, `delta`, and `delete`. Each contains data rows that can be identified by `row__id` and sorted by it, too. Reading data from an ACID table is a process of merging these files, and reflecting the result of the last transaction. This process is written in `OrcInputFormat` and `OrcRawRecordMerger` class, and it is basically a merge-sort algorithm.

Take the following files for an instance. This structure can be generated by: insert three rows, do a major compaction, then update two rows. `1-0-0-1` is short for `originalTransaction` - `bucketId` (not encoded) - `rowId` - `currentTransaction`.

```text
+----------+    +----------+    +----------+
| base_1   |    | delete_2 |    | delta_2  |
+----------+    +----------+    +----------+
| 1-0-0-1  |    | 1-0-1-2  |    | 2-0-0-2  |
| 1-0-1-1  |    | 1-0-2-2  |    | 2-0-1-2  |
| 1-0-2-1  |    +----------+    +----------+
+----------+
```

Merging process:

* Sort rows from all files by (`originalTransaction`, `bucketId`, `rowId`) ascendingly, (`currentTransaction`) descendingly. i.e.
    * `1-0-0-1`
    * `1-0-1-2`
    * `1-0-1-1`
    * ...
    * `2-0-1-2`
* Fetch the first record.
* If the `row__id` is the same as previous, skip.
* If the operation is DELETE, skip.
    * As a result, for `1-0-1-2` and `1-0-1-1`, this row will be skipped.
* Otherwise, emit the row.
* Repeat.

The merging is done in a streaming way. Hive will open all the files, read the first record, and construct a `ReaderKey` class, storing `originalTransaction`, `bucketId`, `rowId`, and `currentTransaction`. `ReaderKey` class implements the `Comparable` interface, so they can be sorted in an customized order.

```java
public class RecordIdentifier implements WritableComparable<RecordIdentifier> {
  private long writeId;
  private int bucketId;
  private long rowId;
  protected int compareToInternal(RecordIdentifier other) {
    if (other == null) {
      return -1;
    }
    if (writeId != other.writeId) {
      return writeId < other.writeId ? -1 : 1;
    }
    if (bucketId != other.bucketId) {
      return bucketId < other.bucketId ? - 1 : 1;
    }
    if (rowId != other.rowId) {
      return rowId < other.rowId ? -1 : 1;
    }
    return 0;
  }
}

public class ReaderKey extends RecordIdentifier {
  private long currentWriteId;
  private boolean isDeleteEvent = false;
  public int compareTo(RecordIdentifier other) {
    int sup = compareToInternal(other);
    if (sup == 0) {
      if (other.getClass() == ReaderKey.class) {
        ReaderKey oth = (ReaderKey) other;
        if (currentWriteId != oth.currentWriteId) {
          return currentWriteId < oth.currentWriteId ? +1 : -1;
        }
        if (isDeleteEvent != oth.isDeleteEvent) {
          return isDeleteEvent ? -1 : +1;
        }
      } else {
        return -1;
      }
    }
    return sup;
  }
}
```

Then, the `ReaderKey` and the file handler will be put into a `TreeMap`, so every time we poll for the first entry, we can get the desired file handler and read data.

```java
public class OrcRawRecordMerger {
  private TreeMap<ReaderKey, ReaderPair> readers = new TreeMap<>();
  public boolean next(RecordIdentifier recordIdentifier, OrcStruct prev) {
    Map.Entry<ReaderKey, ReaderPair> entry = readers.pollFirstEntry();
  }
}
```

### Select Files

Previously we pointed out that different transaction files may co-exist at the same time, so Hive needs to first select the files that are valid for the latest transaction. For instance, the following directory structure is the result of these operations: two inserts, one minor compact, one major compact, and one delete.

```text
delta_0000001_0000001_0000
delta_0000002_0000002_0000
delta_0000001_0000002
base_0000002
delete_delta_0000003_0000003_0000
```

Filtering process:

* Consult the Hive Metastore to find out the valid write ID list.
* Extract transaction information from files names, including file type, write ID range, and statement ID.
* Select the `base` file with the maximum valid write ID.
* Sort `delta` and `delete` files by write ID range:
    * Smaller `minWID` orders first;
    * If `minWID` is the same, larger `maxWID` orders first;
    * Otherwise, sort by `stmtID`; files w/o `stmtID` orders first.
* Use the `base` file's write ID as the current write ID, then iterate and filter `delta` files:
    * If `maxWID` is larger than the current write ID, keep it, and update the current write ID;
    * If write ID range is the same as previous, keep the file, too.

There are some special cases in this process, e.g. no `base` file, multiple statements, contains original data files, even ACID version 1 files. More details can be found in `AcidUtils#getAcidState`.

### Parallel Execution

When executing in parallel environment, such as multiple Hadoop mappers, `delta` files need to be re-organized. In short, `base` and `delta` files can be divided into different splits, while all `delete` files have to be available to all splits. This ensures deleted records will not be emitted.

![Parallel Execution](/cnblogs/images/hive-acid/parallel-execution.png)

### Vectorized Query

For [vectoried query][8], Hive will first try to load all `delete` files into memory and construct an optimized data structure that can be used to filter out deleted rows when processing row batches. If the `delete` files are too large, it falls back to sort-merge algorithm.

```java
public class VectorizedOrcAcidRowBatchReader {
  private final DeleteEventRegistry deleteEventRegistry;

  protected static interface DeleteEventRegistry {
    public void findDeletedRecords(ColumnVector[] cols, int size, BitSet selectedBitSet);
  }
  static class ColumnizedDeleteEventRegistry implements DeleteEventRegistry {}
  static class SortMergedDeleteEventRegistry implements DeleteEventRegistry {}

  public boolean next(NullWritable key, VectorizedRowBatch value) {
    BitSet selectedBitSet = new BitSet(vectorizedRowBatchBase.size);
    this.deleteEventRegistry.findDeletedRecords(innerRecordIdColumnVector,
        vectorizedRowBatchBase.size, selectedBitSet);
    for (int setBitIndex = selectedBitSet.nextSetBit(0), selectedItr = 0;
        setBitIndex >= 0;
        setBitIndex = selectedBitSet.nextSetBit(setBitIndex+1), ++selectedItr) {
      value.selected[selectedItr] = setBitIndex;
    }
  }
}
```

## Transaction Management

Hive introduced a new lock manager to support transactional tables. `DbTxnManager` will detect the ACID operations in query plan and contact the Hive Metastore to open and commit new transactions. It also implements the read-write lock mechanism to support normal locking requirements.

![Transaction Management](/cnblogs/images/hive-acid/transaction-management.png)

The Hive Metastore is responsible for allocating new transaction IDs. This is done in a database transaction so that multiple Metastore instances will not conflict with each other.

```java
abstract class TxnHandler {
  private List<Long> openTxns(Connection dbConn, Statement stmt, OpenTxnRequest rqst) {
    String s = sqlGenerator.addForUpdateClause("select ntxn_next from NEXT_TXN_ID");
    s = "update NEXT_TXN_ID set ntxn_next = " + (first + numTxns);
    for (long i = first; i < first + numTxns; i++) {
      txnIds.add(i);
      rows.add(i + "," + quoteChar(TXN_OPEN) + "," + now + "," + now + ","
          + quoteString(rqst.getUser()) + "," + quoteString(rqst.getHostname()) + "," + txnType.getValue());
    }
    List<String> queries = sqlGenerator.createInsertValuesStmt(
        "TXNS (txn_id, txn_state, txn_started, txn_last_heartbeat, txn_user, txn_host, txn_type)", rows);
  }
}
```

## 参考资料

* [Hive Transactions][2]
* [Transactional Operations in Apache Hive](https://www.slideshare.net/Hadoop_Summit/transactional-operations-in-apache-hive-present-and-future-102803358)
* [ORCFile ACID Support](https://orc.apache.org/docs/acid.html)

[1]: http://hive.apache.org/
[2]: https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions
[3]: https://hortonworks.com/tutorial/using-hive-acid-transactions-to-insert-update-and-delete-data/
[4]: https://jira.apache.org/jira/browse/HIVE-14035
[5]: https://orc.apache.org/
[6]: https://orc.apache.org/docs/java-tools.html
[7]: https://cwiki.apache.org/confluence/display/Hive/Streaming+Data+Ingest+V2
[8]: https://cwiki.apache.org/confluence/display/Hive/Vectorized+Query+Execution
