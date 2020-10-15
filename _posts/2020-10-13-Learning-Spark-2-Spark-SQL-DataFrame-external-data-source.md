---

layout: post
title: Learning Spark (2) - Spark SQL 和 DataFrame：外部数据源
date: 2020-10-13 18:11:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Learning Spark]
---

进入第五章：《Spark SQL 和 DataFrame：对接外部数据源》。

<!-- more -->

---

* 目录
{:toc}
---

## Spark SQL 与 Hive

Spark SQL是Apache Spark的基础组件，该组件将关系处理与Spark的功能编程API集成在一起。 Spark SQL使Spark程序员可以利用更快的性能和关系编程（例如声明式查询和优化的存储）以及调用复杂的分析库（例如机器学习）的好处。 

###  自定义函数UDF

尽管Apache Spark具有大量内置函数，但Spark的灵活性允许数据工程师和数据科学家也定义自己的功能。 这些被称为用户定义函数（UDF）。

#### 1. Spark SQL UDFs

```scala
// In Scala
// Create cubed function 
val cubed = (s: Long) => {
	s*s*s
}
// Register UDF
spark.udf.register("cubed", cubed) 
// Create temporary view
spark.range(1, 9).createOrReplaceTempView("udf_test") 

```

```python
# In Python
from pyspark.sql.types import LongType
# Create cubed function
def cubed(s): returns*s*s
# Register UDF
spark.udf.register("cubed", cubed, LongType()) 
# Generate temporary view
spark.range(1, 9).createOrReplaceTempView("udf_test")
```

使用Spark SQL 调用 cubed 函数：

```python
// In Scala/Python
// Query the cubed UDF
spark.sql("SELECT id, cubed(id) AS id_cubed FROM udf_test").show()
```

#### 2. Spark SQL中的评估顺序和空检查

Spark SQL（包括SQL，DataFrame API和Dataset API）不保证子表达式的求值顺序。 例如，以下查询不能保证在`strlen(s) > 1`子句之前执行`s is NOT NULL`子句：

```scala
spark.sql("SELECT s FROM test1 WHERE s IS NOT NULL AND strlen(s) > 1")
```

因此，要执行正确的空检查，有如下的建议：

1. 使UDF本身了解Null，并在UDF内部进行Null检查。
2. 使用IF或CASE WHEN表达式进行空检查，并在条件分支中调用UDF。

#### 3. 使用Pandas UDF加速和分发PySpark UDF

之前使用PySpark UDF存在的主要问题之一是，它们的性能比Scala UDF慢。 这是因为PySpark UDF需要在JVM和Python之间进行数据移动，这非常昂贵。 为了解决此问题，Pandas UDF（也称为矢量化UDF）作为Apache Spark 2.3的一部分引入。 Pandas UDF使用Apache Arrow传输数据，使用Pandas处理数据。 您可以使用关键字`pandas_udf`作为装饰器来定义Pandas UDF，或者包装函数本身。 一旦数据以Apache Arrow格式存储，就不再需要对数据进行序列化/处理，因为它已经是Python进程可使用的格式。 您不是在逐行操作单个输入，而是在Pandas Series或DataFrame上进行操作（即矢量化执行）。

从具有Python 3.6及更高版本的Apache Spark 3.0起，Pandas UDF分为两类API：Pandas UDF和Pandas Function API。

*Pandas UDFs*

>使用Apache Spark 3.0，Pandas UDF从Pandas UDF中的Python类型提示（例如pandas.Series，pandas.DataFrame，Tuple和Iterator）推断Pandas UDF类型。 以前，您需要手动定义和指定每种Pandas UDF类型。 当前，Pandas UDF中支持的Python类型提示的情况包括：“序列到序列”，“序列到迭代器”，“序列到迭代器”，“多个序列”到“序列迭代器”以及“序列到标量”（单个值）。

*Pandas Function APIs*

> Pandas函数API允许您将本地Python函数直接应用于PySpark DataFrame，其中输入和输出均为Pandas实例。 对于Spark 3.0，受支持的Pandas Function API为 grouped map, map, cogrouped map.

以下是用于Spark 3.0的标量Pandas UDF的示例:

```python
# In Python
# Import pandas 
import pandas as pd

# Import various pyspark SQL functions including pandas_udf
from pyspark.sql.functions import col, pandas_udf 
from pyspark.sql.types import LongType

# Declare the cubed function
def cubed(a: pd.Series) -> pd.Series: 
  return a*a*a

# Create the pandas UDF for the cubed function
cubed_udf = pandas_udf(cubed, returnType=LongType())
```

前面的代码片段声明了一个称为`cubes()`的函数，该函数执行立方体操作。 这是一个常规的Pandas函数，带有附加的 `cubed_udf = pandas_udf()`调用，以创建我们的Pandas UDF。

For pandas series:

```python
# Create a Pandas Series
x = pd.Series([1, 2, 3])
# The function for a pandas_udf executed with local Pandas data
print(cubed(x))
```

For Spark DataFrame:

```python
# Create a Spark DataFrame, 'spark' is an existing SparkSession
df = spark.range(1, 4)
# Execute function as a Spark vectorized UDF
df.select("id", cubed_udf(col("id"))).show()
```

与局部函数相反，使用向量化的UDF将导致执行Spark作业。 先前的本地函数是仅在Spark driver 上执行的Pandas函数。 当查看此pandas_udf函数的其中一个阶段的Spark UI时，这一点变得更加明显:

[![](/images/posts/Spark-UI-execution-pandas-udf.jpg)](/images/posts/Spark-UI-execution-pandas-udf.jpg)

像许多Spark作业一样，该 job 从`parallelize()`开始以将本地数据（Arrow binary batches）发送给exectutors，并调用`mapPartitions()`将Arrow binary batches转换为Spark的内部数据格式，该格式可以分发给Spark workers 。有很多 WholeStageCodegen的步骤，代表了性能上的根本提升（由于Project Tungsten的整个阶段代码生成，可以显着提高CPU效率和性能）。 但是，正是在 ArrowEvalPython 步骤执行了 Pandas UDF。



## 使用 Spark SQL Shell，Beeline，Tableau

有多种查询Apache Spark的机制，包括Spark SQL Shell，Beeline CLI实用程序以及诸如Tableau和Power BI之类的报告工具。

### 使用 Spark SQL Shell

一个执行Spark SQL查询的便捷工具是spark-sql CLI。 虽然此实用程序在本地模式下与Hive Metastore服务进行通信，但它不会与Thrift JDBC / ODBC服务器（也称为Spark Thrift Server或STS）通信。 STS允许JDBC / ODBC客户端在Apache Spark上通过JDBC和ODBC协议执行SQL查询。
要启动Spark SQL CLI，请在`$SPARK_HOME`文件夹中执行以下命令：

> ./bin/spark-**sql**

#### Create a table

> spark-**sql**> **CREATE TABLE** people (name STRING, age int);

#### Insert data into the table

> spark-sql> **INSERT INTO people VALUES ("Michael", NULL);**

#### Running a Spark SQL query

> spark-sql> **SHOW TABLES;**
>
> spark-sql> **SELECT \* FROM people WHERE age < 20;**



### 使用 Beeline

如果您使用过Apache Hive，那么您可能会熟悉命令行工具Beeline，这是用于针对HiveServer2运行HiveQL查询的通用实用程序。 Beeline是基于SQLLine CLI的JDBC客户端。 您可以使用同一实用程序对Spark Thrift服务器执行Spark SQL查询。 请注意，当前实现的Thrift JDBC / ODBC服务器与Hive 1.2.1中的HiveServer2相对应。 您可以使用Spark或Hive 1.2.1随附的以下Beeline脚本测试JDBC服务器。

#### 启动 Thrift server

在`$SPARK_HOME`文件夹中执行以下命令：

> ​    ./sbin/start-thriftserver.sh

#### 通过Beeline 连接 Thrift server

> ./bin/beeline
>
> !connect jdbc:hive2://localhost:10000

#### 执行Spark SQL 查询

> 0: jdbc:hive2://localhost:10000> **SHOW tables;**
>
> 0: jdbc:hive2://localhost:10000> **SELECT \* FROM people;**



#### 停止 Thrift Server

> ./sbin/stop-thriftserver.sh



## 外部数据源

### JDBC 和 SQL 数据库

Spark SQL包含一个数据源API，可以使用以下命令从其他数据库读取数据
JDBC。 当它以Dataframe形式返回结果时，它简化了对这些数据源的查询，从而提供了Spark SQL的所有优势（包括性能和与其他数据源的连接能力）。
首先，您需要为JDBC数据源指定JDBC驱动程序，并且该驱动程序必须位于Spark类路径上。 在$SPARK_HOME文件夹中，您将发出如下命令：

> ./bin/spark-shell --driver-class-path $database.jar --jars $database.jar



#### 分区的重要性

在Spark SQL和JDBC外部源之间传输大量数据时，对数据源进行分区很重要。 您的所有数据都通过一个驱动程序连接进行，这可能导致饱和并显着降低提取性能，并可能使源系统的资源饱和。 尽管这些JDBC属性是可选的，但对于任何大规模操作，强烈建议使用表5-2中所示的属性。

[![](/images/posts/spark-partitioning-connection-properties.jpg)](/images/posts/spark-partitioning-connection-properties.jpg)

用一个例子来说明以上几个参数的作用。比如根据以下设置：

- numPartitions:10 
- lowerBound:1000 
- upperBound:10000

那么步长就是1000， 然后将会创建 10 个 partition。这等同于十个查询：

- SELECT * FROM table WHERE partitionColumn BETWEEN 1000 and 2000
- SELECT * FROM table WHERE partitionColumn BETWEEN 2000 and 3000 
-  ...
- SELECT * FROM table WHERE partitionColumn BETWEEN 9000 and 10000

虽然不包含所有内容，但在使用这些属性时，请牢记以下提示：

- numPartitions的最好是使用数量为Spark工作者数的倍数。 例如，如果有四个Spark工作节点，则可能从4个或8个分区开始。 但是，注意源系统可以很好地处理读取请求也很重要。 对于具有处理窗口（processing windows）的系统，您可以最大程度地增加对源系统的并发请求数。 对于缺少处理窗口的系统（例如OLTP系统连续处理数据），应减少并发请求的数量以防止源系统饱和。
- 最初，根据最小和最大partitionColumnColumn实际值计算lowerBound和upperBound。 例如，如果您选择`{numPartitions：10，lowerBound：1000，upperBound：10000}`，但是所有值都在2000到4000之间，则10个查询中只有2个（每个分区一个）将执行所有工作。 在这种情况下，更好的配置将是`{numPartitions：10，lowerBound：2000，upperBound：4000}`。
- 选择可以均匀分布的partitionColumn以避免数据倾斜。 例如，如果您的partitionColumn的大多数值为2500，且`{numPartitions：10，lowerBound：1000，upperBound：10000}`，大部分工作将由请求2000到3000之间的值的任务执行。 选择不同的partitionColumn，或者在可能的情况下生成一个新的（也许是多个列的哈希），以便更均匀地分配您的分区。

### 连接其他数据源

1. PostgreSQL
2. MySQL
3. Azure Cosmos DB
4. MS SQL Server
5. Apache Cassandra 
6. Snowflake
7. MongoDB



## DataFrames和Spark SQL中的高阶函数

由于复杂数据类型是简单数据类型的组合，因此很容易直接对其进行操作。 有两种用于处理复杂数据类型的典型解决方案：

- 将嵌套结构分解为单独的行，应用某些函数，然后重新创建嵌套结构
- 构建用户定义的函数

这些方法的好处是允许您以表格格式考虑问题。 它们通常涉及（但不限于）使用统一函数（utility functions），例如 get_json_object(), from_json(), to_json(), explode(), and selectExpr()。

### 选项1：分解和收集

在此嵌套的SQL语句中，我们首先 explode(values)，它为值内的每个元素（value）创建一个新行（带有id）：

```sql
-- In SQL
SELECT id, collect_list(value + 1) AS values 
FROM (SELECT id, EXPLODE(values) AS value
			FROM table) x 
GROUP BY id
```

尽管`collect_list()`返回具有重复项的对象列表，但是GROUP BY语句需要随机操作，这意味着重新收集的数组的顺序不一定与原始数组的顺序相同。 由于 values 可以是任意数量的维度（一个非常宽和/或非常长的数组），而且我们正在执行GROUP BY，所以这种方法可能会非常昂贵。

### 选择2： UDF

和上述例子完成同样的功能，可以创建一个UDF，它遍历values中所有的值并加一：

```python
spark.sql("SELECT id, plusOneInt(values) AS values FROM table").show()
```

尽管这比使用 `explode()`和`collect_list()`更好，因为不会出现任何排序问题，但是序列化和反序列化过程本身可能会很昂贵。 另外，还必须注意，`collect_list()`可能会使 executor遇到大型数据集的内存不足问题，而使用UDF可以缓解这些问题。

### 高阶函数

Spark SQL 还有一些将匿名lambda函数作为参数的高阶函数。 下面是一个高阶函数的示例：

```sql
-- In SQL
transform(values, value -> lambda expression)
```

`transform()` 函数将 array(values) 和匿名函数（lambda表达式）作为输入。 通过将匿名函数应用于每个元素，然后将结果分配给输出数组，该函数可以透明地创建一个新数组（类似于UDF方法，但效率更高）。

举个例子，先来创建一个数据集：

```python
# In Python
from pyspark.sql.types import *
schema = StructType([StructField("celsius", ArrayType(IntegerType()))])

t_list = [[35, 36, 32, 30, 40, 42, 38]], [[31, 32, 34, 55, 56]]
t_c = spark.createDataFrame(t_list, schema)
t_c.createOrReplaceTempView("tC")

# Show the DataFrame
t_c.show()
```

```scala
// In Scala
// Create DataFrame with two rows of two arrays (tempc1, tempc2) 
val t1 = Array(35, 36, 32, 30, 40, 42, 38)
val t2 = Array(31, 32, 34, 55, 56)
val tC = Seq(t1, t2).toDF("celsius") 
tC.createOrReplaceTempView("tC")
// Show the DataFrame
tC.show()
```

输出：

```
+--------------------+
|             celsius|
+--------------------+
|[35, 36, 32, 30, ...|
|[31, 32, 34, 55, 56]|
+--------------------+
```

#### transform()

```python
// In Scala/Python
// Calculate Fahrenheit from Celsius for an array of temperatures 
spark.sql("""
	SELECT celsius,
			transform(celsius, t -> ((t * 9) div 5) + 32) as fahrenheit
  FROM tC
""").show()
```

输出：

```
+--------------------+--------------------+ 
| 						celsius| 					fahrenheit| 
+--------------------+--------------------+ 
|[35, 36, 32, 30, ...|[95, 96, 89, 86, ...| 
|[31, 32, 34, 55, 56]|[87, 89, 93, 131,...| 
+--------------------+--------------------+
```

#### filter()

```python
// In Scala/Python
// Filter temperatures > 38C for array of temperatures 
spark.sql("""
	SELECT celsius,
     filter(celsius, t -> t > 38) as high
  FROM tC
""").show()
```

输出：

```
+--------------------+--------+ 
| 						celsius| 		high| 
+--------------------+--------+ 
|[35, 36, 32, 30, ...|[40, 42]| 
|[31, 32, 34, 55, 56]|[55, 56]| 
+--------------------+--------+
```

#### exists()

```python
// In Scala/Python
// Is there a temperature of 38C in the array of temperatures 
spark.sql("""
	SELECT celsius,
         exists(celsius, t -> t = 38) as threshold
  FROM tC
""").show()

```

输出：

```
+--------------------+---------+ 
| 						celsius|threshold| 
+--------------------+---------+ 
|[35, 36, 32, 30, ...| 		 true|
|[31, 32, 34, 55, 56]| 		false| 
+--------------------+---------+
```

#### reduce()

> ​    reduce(array<T>, B, function<B, T, B>, function<B, R>)

`reduce()`函数通过使用`function <B，T，B>`将元素合并到缓冲区B中，并在最终缓冲区B 上调用`function<B，R>`，来实现将数组的元素减少为单个值：

```python
// In Scala/Python
// Calculate average temperature and convert to F 
spark.sql("""
	SELECT celsius,
           reduce(
              celsius,
              0,
              (t, acc) -> t + acc,
              acc -> (acc div size(celsius) * 9 div 5) + 32
            ) as avgFahrenheit
   FROM tC
""").show()
```

输出：

```
+--------------------+-------------+ 
| 						celsius|avgFahrenheit| 
+--------------------+-------------+ 
|[35, 36, 32, 30, ...| 					 96| 
|[31, 32, 34, 55, 56]| 					105| 
+--------------------+-------------+
```





未完待续。。。。





