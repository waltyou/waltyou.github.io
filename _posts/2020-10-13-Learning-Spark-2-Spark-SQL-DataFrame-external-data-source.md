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





未完待续。。。。







