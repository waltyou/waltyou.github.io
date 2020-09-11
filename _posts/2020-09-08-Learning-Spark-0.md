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





未完待续。。。









