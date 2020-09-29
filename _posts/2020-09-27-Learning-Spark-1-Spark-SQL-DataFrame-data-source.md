---

layout: post
title: Learning Spark (1) - Spark SQL 和 DataFrame：内置数据源
date: 2020-09-27 18:11:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Learning Spark]
---

直接进入第四章：《Spark SQL 和 DataFrame：内置数据源第介绍》。

<!-- more -->

---

* 目录
{:toc}
---

## 在Spark 应用中使用 Spark SQL

`SparkSession` 提供了一个统一的入口来使用Spark中的结构化API。在SparkSession的实例中，可以使用 sql 函数来运行语句：`spark.sql("SELECT * FROM myTableName")`，这个语句会返回一个 DataFrame。

### 基本例子

```scala
spark.sql("""SELECT date, delay, origin, destination
    FROM us_delay_flights_tbl
    WHERE delay > 120 AND ORIGIN = 'SFO' AND DESTINATION = 'ORD'
    ORDER by delay DESC""").show(10)

spark.sql("""SELECT delay, origin, destination,
              CASE
                  WHEN delay > 360 THEN 'Very Long Delays'
                  WHEN delay > 120 AND delay < 360 THEN 'Long Delays'
                  WHEN delay > 60 AND delay < 120 THEN 'Short Delays'
                  WHEN delay > 0 and delay < 60  THEN  'Tolerable Delays'
                  WHEN delay = 0 THEN 'No Delays'
                  ELSE 'Early'
               END AS Flight_Delays
               FROM us_delay_flights_tbl
               ORDER BY origin, delay DESC""").show(10)
```



## SQL table 和 view

表用来存储数据。与spark中每张表相关的是它们的metadata，比如表结构、介绍、表名、数据库名字、列名、分区、物理地址等等。 这些信息都存在中心metastore。

默认情况下，Spark使用位于/ user / hive / warehouse的Apache Hive元存储来保留关于表的所有元数据，而不是为Spark表提供单独的元存储。 但是，您可以通过将Spark配置变量`spark.sql.warehouse.dir`设置为另一个位置来更改默认位置，该位置可以设置为本地或外部分布式存储。

### 托管与非托管表

Spark允许您创建两种类型的表：托管表和非托管表(managed and unmanaged)。 对于管理表，Spark会管理元数据和文件存储中的数据。 这可以是本地文件系统，HDFS或对象存储，例如Amazon S3或Azure Blob。 对于非托管表，Spark仅管理元数据，而您自己在外部数据源（如Cassandra）中管理数据。

对于托管表，由于Spark可以管理所有内容，因此SQL命令（例如DROP TABLE table_name）会删除元数据和数据。 对于非托管表，同一命令将仅删除元数据，而不删除实际数据。 在下一节中，我们将介绍一些有关如何创建托管表和非托管表的示例。

### 创建 SQL 数据库和表

表驻留在数据库中。 默认情况下，Spark在默认数据库下创建表。 要创建自己的数据库名称，可以从Spark应用程序或笔记本发出SQL命令。

```scala
// In Scala/Python
spark.sql("CREATE DATABASE learn_spark_db")
spark.sql("USE learn_spark_db")
```

#### 1. 创建托管表

```scala
// In Scala/Python
spark.sql("CREATE TABLE managed_us_delay_flights_tbl (date STRING, delay INT,
      distance INT, origin STRING, destination STRING)")
```

或者使用DataFrame API，像这个样子：

```python
# In Python
# Path to our US flight delays CSV file
csv_file = "/databricks-datasets/learning-spark-v2/flights/departuredelays.csv" 
# Schema as defined in the preceding example
schema="date STRING, delay INT, distance INT, origin STRING, destination STRING" 
flights_df = spark.read.csv(csv_file, schema=schema) flights_df.write.saveAsTable("managed_us_delay_flights_tbl")
```

#### 2. 创建非托管表

```python
spark.sql("""CREATE TABLE us_delay_flights_tbl(date STRING, delay INT,
      distance INT, origin STRING, destination STRING)
      USING csv OPTIONS (PATH
      '/databricks-datasets/learning-spark-v2/flights/departuredelays.csv')""")
```

或者使用DataFrame API:

```python
(flights_df
      .write
      .option("path", "/tmp/data/us_flights_delay")
      .saveAsTable("us_delay_flights_tbl"))
```

### 创建视图 view

视图和表之间的区别在于，视图实际上并不保存数据。 您的Spark应用程序终止后，表仍然存在，但是视图消失了。

create view

```sql
-- In SQL
CREATE OR REPLACE GLOBAL TEMP VIEW us_origin_airport_SFO_global_tmp_view AS SELECT date, delay, origin, destination from us_delay_flights_tbl WHERE origin = 'SFO';

CREATE OR REPLACE TEMP VIEW us_origin_airport_JFK_tmp_view AS
SELECT date, delay, origin, destination from us_delay_flights_tbl WHERE origin = 'JFK'
```

```python
# In Python
df_sfo = spark.sql("SELECT date, delay, origin, destination FROM us_delay_flights_tbl WHERE origin = 'SFO'") 
df_jfk = spark.sql("SELECT date, delay, origin, destination FROM us_delay_flights_tbl WHERE origin = 'JFK'")
# Create a temporary and global temporary view
df_sfo.createOrReplaceGlobalTempView("us_origin_airport_SFO_global_tmp_view")                                 df_jfk.createOrReplaceTempView("us_origin_airport_JFK_tmp_view")
```

drop a view

```sql
-- In SQL
DROP VIEW IF EXISTS us_origin_airport_SFO_global_tmp_view; 
DROP VIEW IF EXISTS us_origin_airport_JFK_tmp_view

```

```python
// In Scala/Python
spark.catalog.dropGlobalTempView("us_origin_airport_SFO_global_tmp_view")
spark.catalog.dropTempView("us_origin_airport_JFK_tmp_view")
```

#### 临时视图与全局临时视图

临时视图和全局临时视图之间的差异是细微的，这可能会使Spark新手开发人员产生轻微的混淆。 临时视图绑定到Spark应用程序中的单个SparkSession。 相反，全局临时视图在一个Spark应用程序中的多个SparkSession中可见。 是的，您可以在单个Spark应用程序中创建多个SparkSession。这很方便，例如，当您要访问（并合并）来自两个不共享相同Hive Metastore配置的不同SparkSession中的数据时。



### 查看原数据

如前所述，Spark管理与每个托管或非托管表关联的元数据。 这是在Catalog中捕获的，它是Spark SQL中用于存储元数据的高级抽象。 Catalog的功能在Spark 2.x中通过新的公共方法进行了扩展，使您能够检查与数据库，表和视图关联的元数据。 Spark 3.0将其扩展为使用外部目录。

```scala
// In Scala/Python
spark.catalog.listDatabases()
spark.catalog.listTables()
spark.catalog.listColumns("us_delay_flights_tbl")
```

### 缓存SQL表

您可以缓存和取消缓存SQL表和视图。 在Spark 3.0中，除了其他选项之外，您还可以将表指定为LAZY，这意味着只应在首次使用该表时对其进行缓存，而不是立即对其进行缓存：

```sql
-- In SQL
CACHE [LAZY] TABLE <table-name> 
UNCACHE TABLE <table-name>
```



### 将表读取进DataFrame 中

```scala
// In Scala
val usFlightsDF = spark.sql("SELECT * FROM us_delay_flights_tbl") 
val usFlightsDF2 = spark.table("us_delay_flights_tbl")
```

```python
# In Python
us_flights_df = spark.sql("SELECT * FROM us_delay_flights_tbl")
us_flights_df2 = spark.table("us_delay_flights_tbl")
```













未完待续。。。。。