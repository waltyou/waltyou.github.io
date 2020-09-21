---

layout: post
title: Learning Spark (0) - 总览及第三章结构化API
date: 2020-09-08 18:11:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Learning Spark]
---

Databricks 出了本《Learning Spark》的电子书，来学习一下。 

<!-- more -->

---

* 目录
{:toc}
---

## 总览

[![](/images/posts/LearningSpark.png)](/images/posts/LearningSpark.png)

可以看出这本书总共分为五大部分，分别为：

1. 基本介绍
2. Spark SQL 使用与调优
3. Streaming
4. MLlib
5. Spark 3.0 的新特性


电子书下载地址： https://databricks.com/p/ebook/learning-spark-from-oreilly



前两章非常基础，直接从第三章《Structured APIs》开始。



## RDD 到底是什么？

RDD 是 Spark 最基础的抽象。它分为三部分：

1. Dependencies 依赖
2. Partitions 分区
3. Compute funtion: Partitions => Iterator[T] 计算函数

第一部分：依赖。定义了这个RDD是从何处来的，也保证了必要时重新计算这个RDD。第二部分：partitions 分区。将数据分区存储，天然支持并行计算。第三部分计算函数可以将消费分区数据，然后得到我们想要的 Iterator[T].

整体架构简洁明了。但是也存在几个问题。其中一个就是compute function对Spark是不透明的。对于Spark来讲，compute function 只是一个 lambda expression，不管它实际上是join、filter、select。 另外一个问题是 Iterator[T]  的数据类型，对Python RDD来讲，也是透明的，Spark只知道它是一个python的 generic object。

而且，因为无法检查计算函数，那就无从谈起优化这个函数。另外因为不知道数据类型，也无法直接获取某种类型的某列，Spark 只能将它序列化为字节，而且不能使用任何压缩技术。

毫无疑问，这些不透明性，影响了Spark优化我们的计算，那么解决方案是什么呢？



## 结构化Spark

Spark 2.x 为结构化Spark提供了几个关键方案。

一个是在数据分析中找到的一些通用模式来表达计算。这些模式表示为high-level 处理，比如有filtering、selecting、counting、aggregating、averaging 和 grouping。这样子更清晰和简单。

通过在DSL中使用一组通用运算符，可以进一步缩小这种特异性。 通过DSL中的一组操作（可以在Spark受支持的语言（Java，Python，Spark，R和SQL）中以API的形式使用），这些运算符可以让您告诉Spark您希望对数据进行什么计算，结果， 它可以构建一个有效的执行查询计划。

最终的结构方案是允许我们以表格格式（如SQL表格或电子表格）排列数据，并使用受支持的结构化数据类型。
但是，这种结构有什么用呢？

### 主要优点和好处

- expressivity 表现力
- simplicity 简单性
- composability 可组合性
- uniformity 统一性

使用low-level RDD API：

```python
# In Python
# Create an RDD of tuples (name, age)
dataRDD = sc.parallelize([("Brooke", 20), ("Denny", 31), ("Jules", 30), ("TD", 35)， ("Brooke", 25)])
# Use map and reduceByKey transformations with their lambda # expressions to aggregate and then compute average
agesRDD = (dataRDD
           .map(lambda x: (x[0], (x[1], 1)))
           .reduceByKey(lambda x, y: (x[0] + y[0], x[1] + y[1]))
           .map(lambda x: (x[0], x[1][0]/x[1][1])))
```

这堆lambda 函数，神秘且难读懂。同时Spark也不了解这些代码的意图。而且相同意图下，用Scala写的代码看起来也和用Python写的有很大不同。

high-level API：

```python
# In Python
from pyspark.sql import SparkSession 
from pyspark.sql.functions import avg 
# Create a DataFrame using SparkSession 
spark = (SparkSession
         .builder
	       .appName("AuthorsAges")
         .getOrCreate())
# Create a DataFrame
data_df = spark.createDataFrame([("Brooke", 20), ("Denny", 31), ("Jules", 30), ("TD", 35), ("Brooke", 25)], ["name", "age"])
# Group the same names together, aggregate their ages, and compute an average 
avg_df = data_df.groupBy("name").agg(avg("age"))
# Show the results of the final execution
avg_df.show()

+------+--------+
|  name|avg(age)|
+------+--------+
|Brooke|    22.5|
| Jules|    30.0|
|    TD|    35.0|
| Denny|    31.0|
+------+--------+
```

不过使用high-level、表达力强的DSL 操作符，还是会有一些争议（虽然这些操作符在数据分析中很常见或是重复率很高），那就是我们限制了开发人员的控制他们请求的能力。但是请放心，Spark支持你随时切换到low-level的API中，虽然几乎没有必要。

除了易于阅读之外，Spark的高级API的结构还引入了其组件和语言之间的统一性。 例如，此处显示的Scala代码与以前的Python代码具有相同的作用-并且API看起来几乎相同：

```scala
// In Scala
import org.apache.spark.sql.functions.avg 
import org.apache.spark.sql.SparkSession 
// Create a DataFrame using SparkSession 
val spark = SparkSession
      .builder
      .appName("AuthorsAges")
      .getOrCreate()
// Create a DataFrame of names and ages
val dataDF = spark.createDataFrame(Seq(("Brooke", 20), ("Brooke", 25), ("Denny", 31), ("Jules", 30), ("TD", 35))).toDF("name", "age")
// Group the same names together, aggregate their ages, and compute an average
val avgDF = dataDF.groupBy("name").agg(avg("age")) 
// Show the results of the final execution
avgDF.show()
```

正是因为Spark 构建了high-level结构化API的Spark SQL引擎，这种简单性和表达性才成为可能。由于有了此引擎，它是所有Spark组件的基础，所以我们获得了统一的API。 无论您是在结构化流式处理还是MLlib中对DataFrame表达查询，您始终都在将DataFrame转换为结构化数据并对其进行操作。 

## DataFrame API

受 pandas dataframe 的启发，Spark Dataframe 就像是一张分布在内存中的表格，它带有column name 和type。

### 基本数据类型

[![](/images/posts/spark_scala_dataTypes.jpg)](/images/posts/spark_scala_dataTypes.jpg)

[![](/images/posts/spark_python_dataTypes.jpg)](/images/posts/spark_python_dataTypes.jpg)



### 结构化的和复杂的数据类型

[![](/images/posts/spark_scala_complex_dataTypes.jpg)](/images/posts/spark_scala_complex_dataTypes.jpg)

[![](/images/posts/spark_python_complex_dataTypes.jpg)](/images/posts/spark_python_complex_dataTypes.jpg)



### Schemas 和create DataFrame

Spark中的schema定义了DataFrame的列名和关联的数据类型。 

相对于采用“读取schema”方法，预先定义schema具有三个好处：

- 您可以把 Spark 从推断数据类型的负担中释放出来。
- 您可以阻止Spark创建单独的作业，而只是为了读取文件的大部分内容以确定schema，这对于大型数据文件而言可能既昂贵又费时。
- 您可以及早发现错误，如果数据与架构不匹配。

#### 定义 schema

Using Spark DataFrame API:

```scala
// In Scala
import org.apache.spark.sql.types._
val schema = StructType(Array(StructField("author", StringType, false),StructField("title", StringType, false), StructField("pages", IntegerType, false)))
```

```python
# In Python
from pyspark.sql.types import *
schema = StructType([StructField("author", StringType(), False),
      StructField("title", StringType(), False),
      StructField("pages", IntegerType(), False)])
```

Using DDL:

```scala
// In Scala
val schema = "author STRING, title STRING, pages INT" 

# In Python
schema = "author STRING, title STRING, pages INT"

df = spark.createDataFrame(data, schema)
```

### 列和表达式

Spark 也支持在column上进行逻辑转换：expr("columnName * 5") or (expr("colum nName - 5") > col(anothercolumnName)) 。

```scala
scala> import org.apache.spark.sql.functions._
scala> blogsDF.columns
res2: Array[String] = Array(Campaigns, First, Hits, Id, Last, Published, Url)
// Access a particular column with col and it returns a Column type
scala> blogsDF.col("Id")
res3: org.apache.spark.sql.Column = id
// Use an expression to compute a value
scala> blogsDF.select(expr("Hits * 2")).show(2) 
// or use col to compute value
scala> blogsDF.select(col("Hits") * 2).show(2)

// Use an expression to compute big hitters for blogs
// This adds a new column, Big Hitters, based on the conditional expression 
blogsDF.withColumn("Big Hitters", (expr("Hits > 10000"))).show()

// Concatenate three columns, create a new column, and show the 
// newly created concatenated column
blogsDF
.withColumn("AuthorsId", (concat(expr("First"), expr("Last"), expr("Id")))) .select(col("AuthorsId"))
.show(4)

// These statements return the same value, showing that 
// expr is the same as a col method call 
blogsDF.select(expr("Hits")).show(2)
blogsDF.select(col("Hits")).show(2) 
blogsDF.select("Hits").show(2)

// Sort by column "Id" in descending order 
blogsDF.sort(col("Id").desc).show() 
blogsDF.sort($"Id".desc).show()
```



### Rows 行

Create Row：

```Scala
// In Scala
import org.apache.spark.sql.Row
// Create a Row
val blogRow = Row(6, "Reynold", "Xin", "https://tinyurl.6", 255568, "3/2/2015",
Array("twitter", "LinkedIn"))
// Access using index for individual items blogRow(1)
res62: Any = Reynold
```

```Python
# In Python
from pyspark.sql import Row
blog_row = Row(6, "Reynold", "Xin", "https://tinyurl.6", 255568, "3/2/2015",
["twitter", "LinkedIn"])
# access using index for individual items 
blog_row[1]
'Reynold'
```

Create DataFrame by rows：

```Scala
// In Scala
val rows = Seq(("Matei Zaharia", "CA"), ("Reynold Xin", "CA")) 
val authorsDF = rows.toDF("Author", "State")
authorsDF.show()
```

```Python
# In Python
rows = [Row("Matei Zaharia", "CA"), Row("Reynold Xin", "CA")]
authors_df = spark.createDataFrame(rows, ["Authors", "State"])
authors_df.show()
```

### 通用的DataFrame操作

Spark 提供 DataFrameReader 来读取各式各样的数据源来生成DataFrame，比如 JSON, CSV, Parquet, Text, Avro, ORC, etc。 同时，也提供 DataFrameWriter 来把 DataFame 写回各类数据源。

#### 使用 DataFrameReader & DataFrameWriter

在Spark中读取和写入非常简单，因为社区提供了这些高级抽象和贡献，可以连接到各种数据源，包括常见的NoSQL存储，RDBMS，Apache Kafka和Kinesis等流引擎。

Without schema:

```Scala
// In Scala
val sampleDF = spark 
	.read
	.option("samplingRatio", 0.001)
	.option("header", true) 
	.csv("""/databricks-datasets/learning-spark-v2/sf-fire/sf-fire-calls.csv""")
```

Define schema firstly:

```Python
# In Python, define a schema
from pyspark.sql.types import *
# Programmatic way to define a schema
fire_schema = StructType([StructField('CallNumber', IntegerType(), True),
                          StructField('UnitID', StringType(), True),
                          StructField('IncidentNumber', IntegerType(), True),
                          StructField('CallType', StringType(), True),
                          StructField('CallDate', StringType(), True),
                          StructField('WatchDate', StringType(), True),
                          ...
                          ...
                          StructField('Delay', FloatType(), True)])
# Use the DataFrameReader interface to read a CSV file
sf_fire_file = "/databricks-datasets/learning-spark-v2/sf-fire/sf-fire-calls.csv"
fire_df = spark.read.csv(sf_fire_file, header=True, schema=fire_schema)
```

```Scala
// In Scala it would be similar
val fireSchema = StructType(Array(StructField("CallNumber", IntegerType, true),
                                  StructField("Location", StringType, true),
                                  ...
                                  ...
                                  StructField("Delay", FloatType, true)))
// Read the file using the CSV DataFrameReader
val sfFireFile="/databricks-datasets/learning-spark-v2/sf-fire/sf-fire-calls.csv" 
val fireDF = spark.read.schema(fireSchema)
      .option("header", "true")
      .csv(sfFireFile)
```

#### Saving a DataFrame as a Parquet file or SQL table

你可以使用 DataFrameWriter 保持 DataFrame 到各类数据源中。 Parquet，是默认的格式。这种格式使用snappy来压缩数据，并且会将schema信息保存到metadata中，这样子下次读取这个文件的时候，就不需要手动提供schema。

保存为Parquet 文件：

```Scala
// In Scala to save as a Parquet file
val parquetPath = ... 
fireDF.write.format("parquet").save(parquetPath)
```

```Python
# In Python to save as a Parquet file
parquet_path = ...
fire_df.write.format("parquet").save(parquet_path)
```

保存为sql table，这个表会注册到Hive的metastore 中：

```Scala
// In Scala to save as a table
val parquetTable = ... // name of the table fireDF.write.format("parquet").saveAsTable(parquetTable)
```

```Python
# In Python
parquet_table = ... # name of the table fire_df.write.format("parquet").saveAsTable(parquet_table)
```

#### Projections and filters

关系表达式中的*projection*是一种通过使用过滤器 filters 仅返回与特定关系条件匹配的行的方法。 在Spark中，投影是使用 select() 方法完成的，而过滤器可以使用 filter() 或 where() 方法来表示。

```Python
# In Python
few_fire_df = (fire_df
               .select("IncidentNumber", "AvailableDtTm", "CallType")
               .where(col("CallType") != "Medical Incident"))
few_fire_df.show(5, truncate=False)
```

```Scala
// In Scala
val fewFireDF = fireDF
	.select("IncidentNumber", "AvailableDtTm", "CallType") 
	.where($"CallType" =!= 	"Medical Incident")
fewFireDF.show(5, false)
```

#### 重命名、添加、删除列

```Python
# In Python
fire_ts_df = (new_fire_df
              .withColumn("IncidentDate", to_timestamp(col("CallDate"), "MM/dd/yyyy"))
              .drop("CallDate")
              .withColumn("OnWatchDate", to_timestamp(col("WatchDate"), "MM/dd/yyyy"))
              .drop("WatchDate")
              .withColumn("AvailableDtTS", to_timestamp(col("AvailableDtTm"),
                                                        "MM/dd/yyyy hh:mm:ss a"))
              .drop("AvailableDtTm"))
# Select the converted columns
(fire_ts_df
 .select("IncidentDate", "OnWatchDate", "AvailableDtTS")
 .show(5, False))

```

```Scala
// In Scala
val fireTsDF = newFireDF
  .withColumn("IncidentDate", to_timestamp(col("CallDate"), "MM/dd/yyyy"))
  .drop("CallDate")
  .withColumn("OnWatchDate", to_timestamp(col("WatchDate"), "MM/dd/yyyy"))
  .drop("WatchDate")
  .withColumn("AvailableDtTS", to_timestamp(col("AvailableDtTm"), "MM/dd/yyyy hh:mm:ss a"))
  .drop("AvailableDtTm")
// Select the converted columns
fireTsDF
	.select("IncidentDate", "OnWatchDate", "AvailableDtTS") .show(5, false)
```



#### 聚合 Aggregations

DataFrame 有很多有用的 transforma 和 action，比如 groupBy(), orderBy(), and count()，他们提供了通过列名进行聚合然后得出数量的能力。

```Python
# In Python
(fire_ts_df
 .select("CallType")
 .where(col("CallType").isNotNull())
 .groupBy("CallType")
 .count()
 .orderBy("count", ascending=False)
 .show(n=10, truncate=False))
```

```Scala
// In Scala
fireTsDF
  .select("CallType") 
  .where(col("CallType").isNotNull) 
  .groupBy("CallType")
  .count()
  .orderBy(desc("count")) 
  .show(10, false)
```

#### 其他操作

除了上面看到的，DataFrame API 还提供了状态统计函数：min(), max(), sum(), and avg()。

```Python
# In Python
import pyspark.sql.functions as F 
(fire_ts_df
      .select(F.sum("NumAlarms"), F.avg("ResponseDelayedinMins"),
              F.min("ResponseDelayedinMins"), F.max("ResponseDelayedinMins"))
      .show())
```

```Scala
// In Scala
import org.apache.spark.sql.{functions => F} 
fireTsDF
      .select(F.sum("NumAlarms"), F.avg("ResponseDelayedinMins"),
      	F.min("ResponseDelayedinMins"), F.max("ResponseDelayedinMins"))
      .show()
```



## Datasets API









未完待续。。。









