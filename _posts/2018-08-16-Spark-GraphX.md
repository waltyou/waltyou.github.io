---
layout: post
title: Spark 图计算包：Graphx
date: 2018-08-16 15:04:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark]
---

针对图计算，Spark 下有一个单独的包，叫做： GraphX。来看看它是什么。

<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 总览

GraphX 是 Spark 一个新的组件，专门用来表示图以及进行图的并行计算。
从一个高的角度来看，GraphX 通过重新定义了图的抽象概念来拓展了 RDD： 定向多图，其属性附加到每个顶点和边。

为了支持图计算， GraphX 公开了一系列的基本运算符（比如：subgraph, joinVertices, aggregateMessages）以及优化后的 Pregel API 变种。

此外，GraphX包含越来越多的图算法和构建器，以简化图形分析任务。

---

# 属性图 The Property Graph

属性图是一个有向多图，其中用户定义的对象附加到每个顶点和边。 
有向多图是个有向图，它可能有多个平行边共享相同的源和目标顶点
支持平行边的能力简化了在相同顶点之间可存在多个关系（例如，同事和朋友）的建模场景。

每个顶点由唯一的64位长标识符编码（VertexId）。 GraphX不对顶点标识符施加任何排序约束。
类似地，边具有对应的源和目标顶点标识符。

属性图在顶点（VD）和边缘（ED）类型上进行参数化。这些是分别与每个顶点和边相关联的对象的类型。
> 当GraphX是原始数据类型（例如，int，double等等）时，GraphX 会优化顶点和边缘类型的表示，也就是将它们存储在专用数组中来减少内存占用。

在某些情况下，可能需要在同一图表中具有不同属性类型的顶点。这可以通过继承来完成。
例如，要将用户和产品建模为二分图，我们可能会执行以下操作：
```scala
class VertexProperty()
case class UserProperty(val name: String) extends VertexProperty
case class ProductProperty(val name: String, val price: Double) extends VertexProperty
// The graph might then have the type:
var graph: Graph[VertexProperty, String] = null
```

与RDD一样，属性图是不可变的，分布式的和容错的。
对图的值或结构的更改，都是通过生成具有所需更改的新图完成的。
注意，原始图的大部分（即，未受影响的结构，属性和索引）在新图中被重用，从而降低了这种固有功能数据结构的成本。
使用一系列**顶点分区启发法**将图形划分给执行程序。
与RDD一样，图形的每个分区都可以在发生故障时在不同的机器上重新创建。

逻辑上，属性图对应于一对编码每个顶点和边的属性的类型集合（RDD）。
因此，图类包含访问图的顶点和边的成员：
```scala
class Graph[VD, ED] {
  val vertices: VertexRDD[VD]
  val edges: EdgeRDD[ED]
}
```

_VertexRDD[VD]_ 和 _EdgeRDD[ED]_ 类扩展并分别是 _RDD[(VertexId, VD)]_ 和 _RDD[Edge [ED]]_ 的优化版本。
VertexRDD[VD] 和 EdgeRDD[ED] 都提供了围绕图形计算构建的附加功能，并利用了内部优化。
我们在顶点和边缘RDD部分中更详细地讨论了 VertexRDD 和 EdgeRDD API，但是现在它们可以被认为是 RDD[(VertexId，VD)]和 RDD[Edge[ED]] 形式的简单RDD。

---

# 属性图的例子

假设我们要构建一个由GraphX项目上的各种协作者组成的属性图。
vertex属性可能包含用户名和职业。我们可以使用描述协作者之间关系的字符串来注释edges：

[![](/images/posts/Spark-GraphX-property_graph.png)](/images/posts/Spark-GraphX-property_graph.png)

那么构成的结果图将具有以下的类型签名:
```scala
val userGraph: Graph[(String, String), String]
```

有许多方法可以从原始文件，RDD甚至合成生成器（synthetic generators）构建属性图，这些在[图构建器](#图构建器)一节中有更详细的讨论。
可能最常用的方法就是使用[Graph对象](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.graphx.Graph$)。
如下的代码，显示了如何从一组 RDD 中构建一个图：

```scala
// 假设 SparkContext 已经被构建
val sc: SparkContext
// 为 vertices 创建一个 RDD
val users: RDD[(VertexId, (String, String))] =
  sc.parallelize(Array((3L, ("rxin", "student")), (7L, ("jgonzal", "postdoc")),
                       (5L, ("franklin", "prof")), (2L, ("istoica", "prof"))))
// 为 edges 创建一个 RDD
val relationships: RDD[Edge[String]] =
  sc.parallelize(Array(Edge(3L, 7L, "collab"),    Edge(5L, 3L, "advisor"),
                       Edge(2L, 5L, "colleague"), Edge(5L, 7L, "pi")))
// 定义一个默认的用户，来防止一个关系指向了不存在的用户
val defaultUser = ("John Doe", "Missing")
// 构建初步的 Graph
val graph = Graph(users, relationships, defaultUser)
```

在上面的例子中，我们使用了Edge case类。 Edges 有一个 srcId 和一个 dstId 来对应源顶点和目标顶点的id。
同时 Edges 有一个 attr 成员来存储边的属性。

我们可以分别使用 graph.vertices 和 graph.edges 成员将图解构为相应的顶点和边视图。

```scala
val graph: Graph[(String, String), String] // 上面构造的graph
// 计算所有是postdocs的用户数量
graph.vertices.filter { case (id, (name, pos)) => pos == "postdoc" }.count
// 计算所有 src > dst 的边的数量
graph.edges.filter(e => e.srcId > e.dstId).count
```

> 注意的是 graph.vertices 返回了一个继承 RDD[(VertexId, (String, String))] 的 VertexRDD[(String, String)]。
所以我们可以使用 scala 的 _case_ 表达式来结构这个元组（tuple）。
另一方面来讲，graph.edges 返回了一个包含 Edge[String] 的 EdgeRDD。
我们也可以使用case类类型构造函数，如下所示：
```scala
graph.edges.filter { case Edge(src, dst, prop) => src > dst }.count
```

除了属性图的顶点和边视图外，GraphX还公开了三元组视图（triplet view）。
三元组视图在逻辑上连接顶点和边缘属性，并产生一个包含 EdgeTriplet 类实例的 RDD[EdgeTriplet[VD, ED]]。

此连接可以用以下SQL表达式表示：
```sql
SELECT src.id, dst.id, src.attr, e.attr, dst.attr
FROM edges AS e LEFT JOIN vertices AS src, vertices AS dst
ON e.srcId = src.Id AND e.dstId = dst.Id
```
或者用图形表示：
[![](/images/posts/Spark-GraphX-triplet.png)](/images/posts/Spark-GraphX-triplet.png)

[EdgeTriplet](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.graphx.EdgeTriplet) 继承了 Edges，
但是添加了 srcAttr 和 dstAttr 两个成员变量来分别表示源顶点和目标顶点的属性。
我们使用图的三元组视图可以得到一个能够呈现描述用户之间关系的字符串集合。
```scala
val graph: Graph[(String, String), String] 
// Use the triplets view to create an RDD of facts.
val facts: RDD[String] =
  graph.triplets.map(triplet =>
    triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.dstAttr._1)
facts.collect.foreach(println(_))
```

---

# 图运算符（Graph Operators）

就像 RDD 拥有 map、filter 和 reduceByKey 等基本操作一样，属性图也拥有一系列基本操作，来获取用户定义的函数并生成具有已转换属性和结构的新图形。
具有优化实现的核心运算符在[Graph](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.graphx.Graph)中定义，
并且表示为核心运算符的组合的便捷运算符在[GraphOps](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.graphx.GraphOps)中定义。

但是，得益于Scala 的 implicits，GraphOps中的运算符自动作为Graph的成员提供。例如，我们可以通过以下方式计算每个顶点的 in-degree（在GraphOps中定义）：
```scala
val graph: Graph[(String, String), String]
// Use the implicit GraphOps.inDegrees operator
val inDegrees: VertexRDD[Int] = graph.inDegrees
```

区分核心图操作和GraphOps的原因是为了能够在将来支持不同的图表表示。每个图形表示必须提供核心操作的实现，并重用GraphOps中定义的许多有用操作。

## 操作的摘要列表

以下是 Graph 和 GraphOps 中定义的功能的快速摘要，但为了简单起见，作为Graph的成员提供。
请注意，某些功能签名已经简化（例如，删除了默认参数和类型约束），并且删除了一些更高级的功能，因此请参阅API文档以获取正式的操作列表。

```scala
/** Summary of the functionality in the property graph */
class Graph[VD, ED] {
  // Graph 的信息 ===================================================================
  val numEdges: Long
  val numVertices: Long
  val inDegrees: VertexRDD[Int]
  val outDegrees: VertexRDD[Int]
  val degrees: VertexRDD[Int]
  // graph 的集合视角 =============================================================
  val vertices: VertexRDD[VD]
  val edges: EdgeRDD[ED]
  val triplets: RDD[EdgeTriplet[VD, ED]]
  // 缓存 graphs 的函数 ==================================================================
  def persist(newLevel: StorageLevel = StorageLevel.MEMORY_ONLY): Graph[VD, ED]
  def cache(): Graph[VD, ED]
  def unpersistVertices(blocking: Boolean = true): Graph[VD, ED]
  // 更改分区启发式  ============================================================
  def partitionBy(partitionStrategy: PartitionStrategy): Graph[VD, ED]
  // 变换顶点和边缘属性 ==========================================================
  def mapVertices[VD2](map: (VertexId, VD) => VD2): Graph[VD2, ED]
  def mapEdges[ED2](map: Edge[ED] => ED2): Graph[VD, ED2]
  def mapEdges[ED2](map: (PartitionID, Iterator[Edge[ED]]) => Iterator[ED2]): Graph[VD, ED2]
  def mapTriplets[ED2](map: EdgeTriplet[VD, ED] => ED2): Graph[VD, ED2]
  def mapTriplets[ED2](map: (PartitionID, Iterator[EdgeTriplet[VD, ED]]) => Iterator[ED2]): Graph[VD, ED2]
  // 修改 graph 结构 ====================================================================
  def reverse: Graph[VD, ED]
  def subgraph(
      epred: EdgeTriplet[VD,ED] => Boolean = (x => true),
      vpred: (VertexId, VD) => Boolean = ((v, d) => true))
    : Graph[VD, ED]
  def mask[VD2, ED2](other: Graph[VD2, ED2]): Graph[VD, ED]
  def groupEdges(merge: (ED, ED) => ED): Graph[VD, ED]
  // join RDDs 和 graph ======================================================================
  def joinVertices[U](table: RDD[(VertexId, U)])(mapFunc: (VertexId, VD, U) => VD): Graph[VD, ED]
  def outerJoinVertices[U, VD2](other: RDD[(VertexId, U)])
      (mapFunc: (VertexId, VD, Option[U]) => VD2)
    : Graph[VD2, ED]
  // 有关相邻三元组的汇总信息 =================================================
  def collectNeighborIds(edgeDirection: EdgeDirection): VertexRDD[Array[VertexId]]
  def collectNeighbors(edgeDirection: EdgeDirection): VertexRDD[Array[(VertexId, VD)]]
  def aggregateMessages[Msg: ClassTag](
      sendMsg: EdgeContext[VD, ED, Msg] => Unit,
      mergeMsg: (Msg, Msg) => Msg,
      tripletFields: TripletFields = TripletFields.All)
    : VertexRDD[A]
  // 迭代图并行计算 ==========================================================
  def pregel[A](initialMsg: A, maxIterations: Int, activeDirection: EdgeDirection)(
      vprog: (VertexId, VD, A) => VD,
      sendMsg: EdgeTriplet[VD, ED] => Iterator[(VertexId,A)],
      mergeMsg: (A, A) => A)
    : Graph[VD, ED]
  // 图基本算法 ========================================================================
  def pageRank(tol: Double, resetProb: Double = 0.15): Graph[Double, Double]
  def connectedComponents(): Graph[VertexId, ED]
  def triangleCount(): Graph[Int, ED]
  def stronglyConnectedComponents(numIter: Int): Graph[VertexId, ED]
}
```

## 属性操作

类似 RDD 中的 map 操作， Graph 中也包含如下操作：

```scala
class Graph[VD, ED] {
  def mapVertices[VD2](map: (VertexId, VD) => VD2): Graph[VD2, ED]
  def mapEdges[ED2](map: Edge[ED] => ED2): Graph[VD, ED2]
  def mapTriplets[ED2](map: EdgeTriplet[VD, ED] => ED2): Graph[VD, ED2]
}
```

每个操作都会产生一个新的 graph， 这个 graph 的 vertex 和 edge 的属性会更具用户输入的 map 函数发生更改。

> 需要注意的是在每种情况下， graph 的结构都是不变的。这是这些操作中的一个核心特点，它允许产生的图重用原始图的结构索引。
> 以下两个代码片段在逻辑上是等效的，但第一个片段不保留结构索引，并且不会受益于GraphX系统优化
```scala
// 第一个片段
val newVertices = graph.vertices.map { case (id, attr) => (id, mapUdf(id, attr)) }
val newGraph = Graph(newVertices, graph.edges)
// 第二个片段： 使用 mapVertices 
val newGraph = graph.mapVertices((id, attr) => mapUdf(id, attr))
```

这些运算通常用于初始化一些用于特定计算或者项目的图形，让它们远离不必要的属性。
例如，给出一个以 out degree 为顶点属性的图形（我们稍后将描述如何构建这样的图形），我们初始化它用于 PageRank：
```scala
// 创建一个顶点属性为 out degree 的图
val inputGraph: Graph[Int, String] =
  graph.outerJoinVertices(graph.outDegrees)((vid, _, degOpt) => degOpt.getOrElse(0))
// 构建一个每条边都包含权重的图
// 图的每个顶点是初始化的 PageRank
val outputGraph: Graph[Double, Double] =
  inputGraph.mapTriplets(triplet => 1.0 / triplet.srcAttr).mapVertices((id, _) => 1.0)
``` 

## 结构操作

目前GraphX仅支持一组简单的常用结构运算符：
```scala
class Graph[VD, ED] {
  def reverse: Graph[VD, ED]
  def subgraph(epred: EdgeTriplet[VD,ED] => Boolean,
               vpred: (VertexId, VD) => Boolean): Graph[VD, ED]
  def mask[VD2, ED2](other: Graph[VD2, ED2]): Graph[VD, ED]
  def groupEdges(merge: (ED, ED) => ED): Graph[VD,ED]
}
```

### reverse 

返回一个新图形，其中所有边方向都反转。 
例如，当尝试计算反向PageRank时，这可能很有用。 
由于反向操作不会修改顶点或边缘属性或更改边数，因此可以有效地实现，而无需数据移动或复制。

### subgraph 

采用顶点和边谓词，并返回仅包含满足顶点谓词（计算为true）的顶点和满足边谓词的边的图，并连接满足顶点谓词的顶点。 
可以在多种情况下使用子图操作符将图形限制为感兴趣的顶点和边缘或消除断开的链接。 
例如，在以下代码中，我们删除了断开的链接：
```scala
// 创建一个顶点的 RDD
val users: RDD[(VertexId, (String, String))] =
  sc.parallelize(Array((3L, ("rxin", "student")), (7L, ("jgonzal", "postdoc")),
                       (5L, ("franklin", "prof")), (2L, ("istoica", "prof")),
                       (4L, ("peter", "student"))))
// 创建一个边的 RDD
val relationships: RDD[Edge[String]] =
  sc.parallelize(Array(Edge(3L, 7L, "collab"),    Edge(5L, 3L, "advisor"),
                       Edge(2L, 5L, "colleague"), Edge(5L, 7L, "pi"),
                       Edge(4L, 0L, "student"),   Edge(5L, 0L, "colleague")))
// 定义一个默认 user 防止关系连到不存在的 user 
val defaultUser = ("John Doe", "Missing")
// 构建图
val graph = Graph(users, relationships, defaultUser)
// 注意在边的初始中，存在一个 user0 （缺失信息）
graph.triplets.map(
  triplet => triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.dstAttr._1
).collect.foreach(println(_))
// 删除丢失的顶点，以及连接到它们的顶点
// 请注意，这里仅提供了顶点谓词。如果未提供顶点或边谓词，则子图运算符默认为true
val validGraph = graph.subgraph(vpred = (id, attr) => attr._2 != "Missing")
// subgraph 中移除了 user 0 后， user 4 和 5 将不再相连
validGraph.vertices.collect.foreach(println(_))
validGraph.triplets.map(
  triplet => triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.dstAttr._1
).collect.foreach(println(_))
```

### mask 

构建了一个子图，这个子图包含了那些能在输入图中找到的顶点和边。 
这可以与子图运算符结合使用，以基于另一个相关图中的属性来限制图。 
例如，我们可以使用缺少顶点的图运行连接的组件，然后将答案限制为有效的子图。
```scala
// 运行 Connected Components
val ccGraph = graph.connectedComponents() // No longer contains missing field
// 删除无效的顶点，及连向它们的边
val validGraph = graph.subgraph(vpred = (id, attr) => attr._2 != "Missing")
// 限制答案都必须在 validGraph 中
val validCCGraph = ccGraph.mask(validGraph)
```

### groupEdges 

合并多图中的平行边（即，顶点对之间的重复边）。 
在许多数字应用中，可以将平行边缘（它们的权重组合）添加到单个边缘中，从而减小图形的尺寸。


---


# 未完待续......


---

# 参考链接

1. [GraphX Programming Guide](http://spark.apache.org/docs/latest/graphx-programming-guide.html)