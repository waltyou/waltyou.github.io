---
layout: post
title: 精通 Apache Spark 源码 | 问题 03・二、RDD与抽象 | RDD 血缘链实现（map/filter/join 转换的宽窄依赖对比）
date: 2025-06-25 13:39:04
author: admin
comments: true
categories: [Spark]
tags: [BigData, Spark]
---

<!-- more -->

---

* 目录
  {:toc}

---

## 前言

这是[本系列](../master-in-apache-spark-with-source-code-00)的第三个问题，属于第二大类"RDD与数据抽象"，问题描述为：

```markdown
**RDD lineage implementation**: How do transformations (map/filter/join) create lineage chains? Compare narrow vs wide dependencies in code.  
   *Key files:* `RDD.scala`, `Dependency.scala`
```

## 解答

在 Spark 中，诸如 map、filter、join 这样的转换操作（transformation）是不会立即执行计算的，而是返回一个新的 RDD。每个新的 RDD 都会记录它的父 RDD 以及应用的转换操作，这样就形成了一条“血缘链”（lineage chain）。每个 RDD 都有一个 dependencies 属性，描述其父 RDD 及依赖类型（如 OneToOneDependency、ShuffleDependency）。这些依赖串联起来就形成了完整的 lineage chain。

在 `Dependency.scala` 中定义了Dependency 的接口和一些基本实现。

```text
Dependency[T] # 抽象类
│
├── NarrowDependency[T]  # 抽象类，多了 getParents 方法
│     ├── OneToOneDependency[T] # 实现类，getParents 返回所有分区
│     └── RangeDependency[T] # 实现类，getParents 返回部分分区
│
└── ShuffleDependency[K, V, C] # 代表宽依赖
```

`ShuffleDependency` 是 Spark 中用于表示宽依赖（Shuffle 依赖）的类，其具体实现细节如下：

### 主要属性

- **`_rdd`**: 父 RDD，表示依赖的输入数据。
- **`partitioner`**: 分区器，用于定义 Shuffle 输出的分区规则。
- **`serializer`**: 序列化器，用于序列化数据。
- **`keyOrdering`**: 可选的键排序规则。
- **`aggregator`**: 可选的聚合器，用于 map/reduce 端的聚合操作。
- **`mapSideCombine`**: 是否启用 map 端的部分聚合。
- **`shuffleWriterProcessor`**: 控制 Shuffle 写行为的处理器。
- **`shuffleId`**: 当前 Shuffle 的唯一标识符。
- **`shuffleHandle`**: Shuffle 的句柄，用于注册 Shuffle。
- **`mergerLocs`**: 存储处理 Shuffle 合并请求的外部服务位置。
- **`shuffleMergeFinalized`**: 标记 Shuffle 合并是否已完成。

### 主要方法

- **`rdd`**: 返回父 RDD。
- **`setShuffleMergeAllowed`**: 设置是否允许 Shuffle 合并。
- **`shuffleMergeEnabled`**: 检查是否启用了 Shuffle 合并。
- **`setMergerLocs`**: 设置 Shuffle 合并服务的位置。
- **`getMergerLocs`**: 获取 Shuffle 合并服务的位置。
- **`markShuffleMergeFinalized`**: 标记 Shuffle 合并已完成。
- **`shuffleMergeFinalized`**: 检查 Shuffle 合并是否已完成。
- **`newShuffleMergeState`**: 重置 Shuffle 合并状态。
- **`incPushCompleted`**: 标记某个 map 任务的 block 推送完成。

### 关键逻辑

1. **构造函数**: 初始化属性，注册 Shuffle，并检查是否启用 Push-Based Shuffle。
2. **Push-Based Shuffle**: 通过 `shuffleMergeEnabled` 和相关方法支持推送式 Shuffle 合并。
3. **依赖关系**: 继承自 `Dependency`，但不实现 `getParents`，因为宽依赖的子分区可能依赖父 RDD 的所有分区。

### 示例代码

以下是 `ShuffleDependency` 的核心实现：

```scala
class ShuffleDependency[K: ClassTag, V: ClassTag, C: ClassTag](
    @transient private val _rdd: RDD[_ <: Product2[K, V]],
    val partitioner: Partitioner,
    val serializer: Serializer = SparkEnv.get.serializer,
    val keyOrdering: Option[Ordering[K]] = None,
    val aggregator: Option[Aggregator[K, V, C]] = None,
    val mapSideCombine: Boolean = false,
    val shuffleWriterProcessor: ShuffleWriteProcessor = new ShuffleWriteProcessor)
  extends Dependency[Product2[K, V]] with Logging {

  override def rdd: RDD[Product2[K, V]] = _rdd.asInstanceOf[RDD[Product2[K, V]]]

  val shuffleId: Int = _rdd.context.newShuffleId()
  val shuffleHandle: ShuffleHandle = _rdd.context.env.shuffleManager.registerShuffle(
    shuffleId, this)

  private[this] val numPartitions = rdd.partitions.length
  private[this] var _shuffleMergeAllowed = canShuffleMergeBeEnabled()

  def shuffleMergeEnabled: Boolean = shuffleMergeAllowed && mergerLocs.nonEmpty
  def shuffleMergeAllowed: Boolean = _shuffleMergeAllowed

  private def canShuffleMergeBeEnabled(): Boolean = {
    val isPushShuffleEnabled = Utils.isPushBasedShuffleEnabled(rdd.sparkContext.getConf, isDriver = true)
    isPushShuffleEnabled && numPartitions > 0 && !rdd.isBarrier()
  }
}
```

`ShuffleDependency` 的实现主要用于管理 Shuffle 的元数据和行为，支持 Spark 的宽依赖计算模型。



再来看窄依赖的 map/filter， 它们定义在 org.apache.spark.rdd.RDD 类中。它们都是返回了一个 MapPartitionsRDD，

```scala
  /**
   * Return a new RDD by applying a function to all elements of this RDD.
   */
  def map[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (_, _, iter) => iter.map(cleanF))
  }

    /**
   * Return a new RDD containing only the elements that satisfy a predicate.
   */
  def filter(f: T => Boolean): RDD[T] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[T, T](
      this,
      (_, _, iter) => iter.filter(cleanF),
      preservesPartitioning = true)
  }
```

宽依赖 join 定义在 org.apache.spark.rdd.PairRDDFunctions 类中。

```scala
  /**
   * Return an RDD containing all pairs of elements with matching keys in `this` and `other`. Each
   * pair of elements will be returned as a (k, (v1, v2)) tuple, where (k, v1) is in `this` and
   * (k, v2) is in `other`. Uses the given Partitioner to partition the output RDD.
   */
  def join[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (V, W))] = self.withScope {
    this.cogroup(other, partitioner).flatMapValues( pair =>
      for (v <- pair._1.iterator; w <- pair._2.iterator) yield (v, w)
    )
  }

  def cogroup[W](other: RDD[(K, W)], partitioner: Partitioner)
      : RDD[(K, (Iterable[V], Iterable[W]))] = self.withScope {
    if (partitioner.isInstanceOf[HashPartitioner] && keyClass.isArray) {
      throw SparkCoreErrors.hashPartitionerCannotPartitionArrayKeyError()
    }
    val cg = new CoGroupedRDD[K](Seq(self, other), partitioner)
    cg.mapValues { case Array(vs, w1s) =>
      (vs.asInstanceOf[Iterable[V]], w1s.asInstanceOf[Iterable[W]])
    }
  }
```

它返回了一个 CoGroupedRDD ， CoGroupedRDD中的 `getDependencies` 方法会返回一个 ShuffleDependency，这样子 DAGScheduler 就知道它是个宽依赖了。