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

```Scala
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

```Scala
// In Scala/Python
spark.sql("CREATE DATABASE learn_spark_db")
spark.sql("USE learn_spark_db")
```

#### 1. 创建托管表

```Scala
// In Scala/Python
spark.sql("CREATE TABLE managed_us_delay_flights_tbl (date STRING, delay INT,
      distance INT, origin STRING, destination STRING)")
```

或者使用DataFrame API，像这个样子：

```Python
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



未完待续。。。。。
