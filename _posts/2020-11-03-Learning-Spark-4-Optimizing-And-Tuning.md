---

layout: post
title: Learning Spark (4) - Spark 应用的优化与调整
date: 2020-11-03 18:11:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Learning Spark]
---

进入第七章：《Spark 应用的优化与调整》。

<!-- more -->

---

* 目录
{:toc}
---

## 优化和调整Spark以提高效率

### 查看和设置 Spark 的配置

通常有三种方式来读取和设置Spark 属性。

第一种是通过配置文件。 在 `$SPARK_HOME` 目录下，有许多配置文件：*conf/spark-defaults.conf.template*, *conf/ log4j.properties.template*, 和 *conf/spark-env.sh.template*。 修改其中某些值，并保存为不带 template 后缀的新文件。

第二种是直接在使用 `spark-submit` 命令行提交应用时，使用 `--conf` 指：

```bash
spark-submit \
--conf spark.sql.shuffle.partitions=5 \
--conf "spark.executor.memory=2g" \
--class main.scala.chapter7.SparkConfig_7_1 jars/main-scala-chapter7_2.12-1.0.jar
```

或者在代码里：

```scala
// In Scala
import org.apache.spark.sql.SparkSession

def printConfigs(session: SparkSession) = { 
  // Get conf
  val mconf = session.conf.getAll
  // Print them
  for (k <- mconf.keySet) { 
    println(s"${k} -> ${mconf(k)}\n") 
  }
}

def main(args: Array[String]) { 
  // Create a session
  val spark = SparkSession.builder
       .config("spark.sql.shuffle.partitions", 5)
       .config("spark.executor.memory", "2g")
       .master("local[*]")
       .appName("SparkConfig")
       .getOrCreate()
  printConfigs(spark)
  spark.conf.set("spark.sql.shuffle.partitions",
                 spark.sparkContext.defaultParallelism)
  println(" ****** Setting Shuffle Partitions to Default Parallelism")
  printConfigs(spark)
}

spark.driver.host -> 10.8.154.34 
spark.driver.port -> 55243 
spark.app.name -> SparkConfig 
spark.executor.id -> driver 
spark.master -> local[*] 
spark.executor.memory -> 2g 
spark.app.id -> local-1580162894307 
spark.sql.shuffle.partitions -> 5
```

第三种是通过Spark shell 的编程接口。与Spark中的所有其他内容一样，API是交互的主要方法。 通过`SparkSession`对象，您可以访问大多数Spark配置设置。

```shell
scala> val mconf = spark.conf.getAll
...
scala> for (k <- mconf.keySet) { println(s"${k} -> ${mconf(k)}\n") }

spark.driver.host -> 10.13.200.101
spark.driver.port -> 65204
spark.repl.class.uri -> spark://10.13.200.101:65204/classes
...
```

或者Spark SQL：

```scala
// In Scala
spark.sql("SET -v").select("key", "value").show(5, false)
```

```python
# In Python
spark.sql("SET -v").select("key", "value").show(n=5, truncate=False)
```





未完待续。。。