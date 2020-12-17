---

layout: post
title: Learning Spark (5) - Spark 3.0
date: 2020-12-09 18:11:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Learning Spark]
---

Spark 3.0 已经release了，来看看有什么新东西。

<!-- more -->

---

* 目录
{:toc}
---

## Spark Core 和 Spark SQL

Spark Core和Spark SQL引擎中引入了许多更改，以帮助加快查询速度。 加快查询的一种方法是使用动态分区修剪（Dynamic Partition Pruning）来读取较少的数据。 另一个是在执行过程中调整和优化查询计划（Adaptive Query Execution）。

### Dynamic Partition Pruning

动态分区修剪（DPP）的目的是跳过查询结果中不需要的数据。 DPP最佳的典型方案是联接两个表：事实表（分区在多列上）和维度表（未分区），如下图所示。 通常，过滤器位于表的Nonpartitioned 边（在本例中为 Date）。 例如，考虑对两个表Sales和Date的常见查询：

```sql
-- In SQL
SELECT * FROM Sales JOIN ON Sales.date = Date.date
```

[![](/images/posts/spark-dynamic-filter.jpg)](/images/posts/spark-dynamic-filter.jpg)

DPP中的关键优化技术是从维度表中获取过滤器的结果，并将其注入到事实表中，作为扫描操作的一部分，以限制读取的数据。

考虑维度表小于事实表，并且我们要执行一个join。

[![](/images/posts/spark-DPP-2.jpg)](/images/posts/spark-DPP-2.jpg) 

在这种情况下，Spark很可能会进行BHJ。 在此 join 期间，Spark将执行以下步骤以最大程度地减少从较大的事实表中扫描的数据量：

1. 在 join的维度表一边，Spark 会从维度表中创建一个hash table，也被叫做build relation，也是filter query的一部分。
2. Spark会将查询结果插入哈希表，并将其分配给广播变量，该变量将分发给参与此联接操作的所有执行程序。
3. 在每个执行程序上，Spark都会探测广播的哈希表，以确定要从事实表读取哪些对应的行。
4. 最后，Spark将把此过滤器动态注入事实表的文件扫描操作中，并重用广播变量的结果。 这样，作为事实表上文件扫描操作的一部分，仅扫描与过滤器匹配的分区，并且仅读取所需的数据。

默认情况下已启用，因此您无需显式配置它，当在两个表之间执行联接时，所有这些都会动态发生。 通过DPP优化，Spark 3.0可以更好地处理 start-schema 查询。

### Adaptive Query Execution

Spark 3.0优化查询性能的另一种方法是在运行时调整其物理执行计划。 自适应查询执行（AQE）根据在查询执行过程中收集的运行时统计信息来重新优化和调整查询计划。 它尝试在运行时执行以下操作：

- 通过减少shuffle分区的数量，来减少reducer数量。
- 优化查询的物理执行计划，例如通过在适当的地方将`SortMergeJoin`转换为`BroadcastHashJoin`。
- 处理join过程中的数据倾斜

所有这些自适应措施都在计划执行期间发生。 要在Spark 3.0中使用AQE，请将配置`spark.sql.adaptive.enabled`设置为true。

[![](/images/posts/spark-AQE.jpg)](/images/posts/spark-AQE.jpg) 

#### AQE 架构

查询中的Spark操作是通过流水线方式并在并行进程中执行的，但是一个 shuffle或者 broadcast 交换会打破这个pipeline，因为需要将这一级的输出作为下一级的输入。 这些断点在查询阶段称为实现点（*materialization points*），它们提供了重新优化和重新检查查询的机会，如图所示：

[![](/images/posts/spark-AQE-framework.jpg)](/images/posts/spark-AQE-framework.jpg) 

如图所示，这是AQE框架迭代的概念步骤：

1. 执行每个阶段的所有叶节点，例如扫描操作。
2. 实现点完成执行后，将其标记为完成，并且在执行期间收集的所有相关统计信息都会在其逻辑计划中进行更新。
3. 基于这些统计信息，例如读取的分区数，读取的数据字节等，框架再次运行Catalyst优化器以了解它是否可以：
   1. 合并分区的数量以减少用于读取shuffle数据的reducer的数量。
   2. 根据读取的表的大小，用BHJ 替换 SMJ
   3. 尝试补救倾斜join
   4. 创建一个新的优化逻辑计划，然后创建一个新的优化物理计划。

重复以上过程，直到执行了查询计划的所有阶段。

简而言之，这种重新优化是动态完成的，其目标是动态合并shuffle分区，减少读取shuffle输出数据所需的reduce数量，并在适当时切换join策略，并纠正任何倾斜连接。

两种Spark SQL配置指示AQE如何减少reducer的数量：

- `spark.sql.adaptive.coalescePartitions.enabled` (set to true)
- `spark.sql.adaptive.skewJoin.enabled` (set to true)



### SQL Join Hints

在现有的`BROADCAST`联接提示中，Spark 3.0为所有Spark联接策略添加了联接提示。 此处为每种连接类型提供了示例。

#### 1. Shuffle sort merge join (SMJ)

如以下示例所示，您可以在/ * + ... * /注释块内的SELECT语句中添加一个或多个提示：

```sql
SELECT /*+ MERGE(a, b) */ id FROM a JOIN b ON a.key = b.key
SELECT /*+ MERGE(customers, orders) */ * FROM customers, orders WHERE
		orders.custId = customers.custId
```

#### 2. Broadcast hash join (BHJ)

```sql
SELECT /*+ BROADCAST(a) */ id FROM a JOIN b ON a.key = b.key
SELECT /*+ BROADCAST(customers) */ * FROM customers, orders WHERE
		orders.custId = customers.custId
```

#### 3. Shuffle hash join (SHJ)

```sql
SELECT /*+ SHUFFLE_HASH(a, b) */ id FROM a JOIN b ON a.key = b.key
SELECT /*+ SHUFFLE_HASH(customers, orders) */ * FROM customers, orders WHERE
		orders.custId = customers.custId
```

#### 4. Shuffle-and-replicate nested loop join (SNLJ)

```sql
SELECT /*+ SHUFFLE_REPLICATE_NL(a, b) */ id FROM a JOIN b
```



### Catalog Plugin API and DataSourceV2

Spark 3.0的实验性DataSourceV2 API不仅限于Hive元存储和catalog，还扩展了Spark生态系统并为开发人员提供了三个核心功能。 具体来说，它：

- 允许插入用于catalog和表管理的外部数据源
- 通过支持的文件格式，如ORC，Parquet，Kafka，Cassandra，Delta Lake和Apache Iceberg，支持将谓词下推到其他数据源。
- 提供统一的API，用于接收器和源的数据源的流式处理和批处理

针对希望扩展Spark使用外部源和接收器功能的开发人员，Catalog API提供了SQL和编程API来从指定的可插入目录中创建，更改，加载和删除表。 该目录提供了在不同级别执行的功能和操作的分层抽象:

[![](/images/posts/spark-Catalog-plugin.jpg)](/images/posts/spark-Catalog-plugin.jpg) 

Spark和特定连接器之间的初始交互是解决与其实际Table对象的关系。 Catalog 定义了如何在此连接器中查找表。 此外，Catalog 可以定义如何修改自己的元数据，从而启用诸如CREATE TABLE，ALTER TABLE等操作。

例如，在SQL中，您现在可以发出命令来为Catalog 创建 namespace。 要使用可插入的目录，请在`spark-defaults.conf`文件中启用以下配置：

```
spark.sql.catalog.ndb_catalog com.ndb.ConnectorImpl # connector implementation
spark.sql.catalog.ndb_catalog.option1  value1
spark.sql.catalog.ndb_catalog.option2  value2
```

在此，数据源目录的连接器具有两个选项：option1-> value1和option2-> value2。 定义完后，Spark或SQL中的应用程序用户可以使用DataFrameReader和DataFrameWriter API方法或带有这些已定义选项的Spark SQL命令作为数据源操作的方法。 例如：

```sql
-- In SQL
SHOW TABLES ndb_catalog;
CREATE TABLE ndb_catalog.table_1; 
SELECT * from ndb_catalog.table_1; 
ALTER TABLE ndb_catalog.table_1;
```

```scala
// In Scala
df.writeTo("ndb_catalog.table_1")
val dfNBD = spark.read.table("ndb_catalog.table_1")
      .option("option1", "value1")
      .option("option2", "value2")
```

这些目录插件API扩展了Spark的能力，可将外部数据源用作接收器和源，但它们仍处于试验阶段，不应在生产中使用。 

### 加速器感知调度程序 Accelerator-Aware Scheduler

Project Hydrogen 是一项将AI和大数据整合在一起的社区计划，其主要目标是三个：实现屏障执行模式, accelerator-aware scheduling 和 optimized data exchange。 Apache Spark 2.4.0中引入了屏障执行模式的基本实现。 在Spark 3.0中，已实现了基本的调度程序，以利用目标平台上的GPU（如独立模式，YARN或Kubernetes部署Spark的硬件加速器）的优势。





未完待续。。。