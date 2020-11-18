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

或者你也可以通过Spark UI 里的 Environment 页来看。

在修改配置之前，可以使用 `spark.conf.isModifiable("*<config_name>*") ` 来查看该配置是否可以更改。

虽然有很多种方式来设置配置。但是它们也是有读取顺序的，spark-defaults.conf 首先被读出来，然后是 spark-submit 的命令行，最后是 SparkSession。所有这些属性将被合并，并且所有重复属性会按优先顺序进行去重。 比如，在命令行中提供的值将取代配置文件中的设置，前提是它们不会在应用程序本身中被覆盖。

调整或提供正确的配置有助于提高性能，正如您将在下一部分中看到的那样。 这里的建议来自社区中从业人员的观察，并且侧重于如何最大程度地利用Spark的群集资源以适应大规模工作负载。



### 扩展Spark以处理大工作量

大型Spark工作负载通常是批处理工作，有些是每晚运行的，有些则是每天定期执行的。 无论哪种情况，这些作业都可能处理数十TB字节的数据或更多。 为了避免由于资源匮乏或性能逐步下降而导致作业失败，可以启用或更改一些Spark配置。 这些配置影响三个Spark组件：Spark driver，executor 和在 executor 上运行的 shuffle 服务。

Spark driver 的职责是与集群管理器协调，以在集群中启动executor，并在其上安排Spark任务。 对于大工作量，您可能有数百个任务。 本节介绍了一些可以调整或启用的配置，以优化资源利用率、任务并行度，来避免出现大任务的瓶颈。 

#### 静态与动态资源分配

如前所述，当您将计算资源指定为`spark-submit`的命令行参数时，您便设置了上限。 这意味着，如果由于工作负载超出预期而导致以后在驱动程序中排队任务时需要更多资源，Spark将无法容纳或分配额外的资源。

相反，如果您使用Spark的动态资源分配配置( `dynamic resource allocation configuration`)，则Spark driver 会随着大型工作负载的需求不断增加和减少，可以请求更多或更少的计算资源。 在工作负载是动态的情况下（即，它们对计算能力的需求各不相同），使用动态分配有助于适应突然出现的峰值。

一种可能有用的用例是streaming，其中数据流量可能不均匀。 另一个是按需数据分析，在高峰时段您可能需要大量SQL查询。 启用动态资源分配可以使Spark更好地利用资源，在不使用执行器时释放它们，并在需要时获取新的执行器。

要启用和配置动态分配，可以使用如下设置。 注意这里的数字是任意的； 适当的设置将取决于您的工作负载的性质，应进行相应的调整。 其中一些配置无法在Spark REPL内设置，因此您必须以编程方式进行设置：

> spark.dynamicAllocation.enabled true
> spark.dynamicAllocation.minExecutors 2
> spark.dynamicAllocation.schedulerBacklogTimeout 1m
> spark.dynamicAllocation.maxExecutors 20
> spark.dynamicAllocation.executorIdleTimeout 2min

默认情况下，`spark.dynamicAllocation.enabled`设置为 false。 当使用此处显示的设置启用时，Spark驱动程序将要求集群管理器创建两个执行程序作为最低启动条件（`spark.dynamicAllocation.minExecutor`）。 随着任务队列积压的增加，每次超过积压超时（`spark.dynamicAllocation.schedulerBacklogTimeout`）都将请求新的执行者。 在这种情况下，每当有未计划的待处理任务超过1分钟时，驱动程序将请求启动新的执行程序以计划积压的任务，最多20个（`spark.dynamicAllocation.maxExecutor`）。 相比之下，如果执行者完成任务并空闲2分钟（`spark.dynamicAllocation.executorIdleTimeout`），Spark驱动程序将终止它。

#### 配置 Spark executors’ memory 和 shuffle service

仅启用动态资源分配是不够的。 您还必须了解Spark如何布置和使用执行程序内存，以便使执行程序不会因内存不足而受JVM垃圾回收的困扰。

每个执行器可用的内存量由`spark.executor.memory`控制。 如图所示，它分为三个部分：执行内存，存储内存和保留内存。 在保留300 MB的预留内存之后，默认内存划分为60％的执行内存和40％的存储内存，以防止OOM错误。 Spark文档建议此方法适用于大多数情况，但是您可以调整希望哪一部分用作基准的`spark.executor.memory`比例。 当不使用存储内存时，Spark可以获取它以供执行内存用于执行目的，反之亦然。

[![](/images/posts/spark-executor-memory-layout.jpg)](/images/posts/spark-executor-memory-layout.jpg)

执行内存用于Spark shuffles，joins, sorts, 和 aggregations。 由于不同的查询可能需要不同的内存量，因此专用于此的可用内存的fraction（默认情况下，`spark.memory.fraction`为0.6）可能很难最优，但是它很易调整。另外，存储内存主要用于缓存从DataFrame派生的用户数据结构和分区。

在 map 和 shuffle 操作期间，Spark会写入和读取本地磁盘的shuffle文件，因此 I/O 活动繁重。 这可能会导致瓶颈，因为对于大型Spark作业，默认配置并不理想。 在Spark作业的此阶段，知道要进行哪些配置调整可以减轻这种风险。

我们捕获了一些建议的配置进行调整，以使这些操作期间的map、spill、merge过程，不会因效率低下的I / O所困扰，并使这些操作能够在将最终的shuffle分区写入磁盘之前使用缓冲内存。 调整在每个executor上运行的shuffle服务还可以帮助提高大型Spark工作负载的整体性能。

| 配置                                    | 默认值、推荐值及介绍                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| spark.driver.memory                     | 默认值为1g（1 GB）。 这是分配给Spark驱动程序以从执行程序接收数据的内存量。 这通常在`spark-submit` 时使用`--driver-memory`进行更改。<br/>仅当您希望驱动程序从诸如`collect()`之类的操作中接收到大量数据，或者驱动程序内存用完时，才更改此设置。 |
| spark.shuffle.file.buffer               | 默认值为32 KB。 推荐为1 MB。 这允许Spark在将最终map结果写入磁盘之前做更多缓冲。 |
| spark.file.transferTo                   | 默认为true。 将其设置为false将强制Spark在最终写入磁盘之前使用文件缓冲区传输文件； 这将减少I / O活动。 |
| spark.shuffle.unsafe.file.output.buffer | 默认值为32 KB。 这可以控制在shuffle操作期间合并文件时可能的缓冲量。 通常，较大的值（例如1 MB）更适合较大的工作负载，而默认值可以适用于较小的工作负载。 |
| spark.io.compression.lz4.blockSize      | 默认值为32 KB。 增加到512 KB。 您可以通过增加块的压缩大小来减少shuffle文件的大小。 |
| spark.shuffle.service.index.cache.size  | 默认值为100m。 缓存条目限制为指定的内存占用空间（以字节为单位）。 |
| spark.shuffle.registration.timeout      | 默认值为5000毫秒。 增加到120000 ms。                         |
| spark.shuffle.registration.maxAttempts  | 默认值为3。如果需要，请增加到5。                             |



未完待续。。。