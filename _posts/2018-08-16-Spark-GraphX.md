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

## Join 操作

在许多情况下，有必要将外部集合（RDD）中的数据与图形连接起来。
例如，我们可能有我们想要与现有图形合并的额外用户属性，或者我们可能希望将顶点属性从一个图形拉到另一个图形。
可以使用连接运算符完成这些任务。下面我们列出了键连接运算符：
```scala
class Graph[VD, ED] {
  def joinVertices[U](table: RDD[(VertexId, U)])(map: (VertexId, VD, U) => VD): Graph[VD, ED]
  def outerJoinVertices[U, VD2](table: RDD[(VertexId, U)])(map: (VertexId, VD, Option[U]) => VD2): Graph[VD2, ED]
}
```

### joinVertices

将顶点与输入RDD连接，并返回一个新图形，其中包含通过将用户定义的映射函数应用于连接顶点的结果而获得的顶点属性。
RDD中没有匹配值的顶点保留其原始值。
> 请注意，如果RDD包含给定顶点的多个值，则只使用一个。 因此，建议使用以下方法使输入RDD唯一，这也将对结果值进行预索引，以大大加速后续连接。
```scala
val nonUniqueCosts: RDD[(VertexId, Double)]
val uniqueCosts: VertexRDD[Double] =
  graph.vertices.aggregateUsingIndex(nonUnique, (a,b) => a + b)
val joinedGraph = graph.joinVertices(uniqueCosts)(
  (id, oldCost, extraCost) => oldCost + extraCost)
```

### outerJoinVertices

outerJoinVertices的行为通常来讲与joinVertices类似，不同之处在于用户定义的map函数应用于所有顶点并且可以更改顶点属性类型。 
因为并非所有顶点在输入RDD中都具有匹配值，所以map函数采用Option类型。 
例如，我们可以通过使用outDegree初始化顶点属性来为PageRank设置图形。
```scala
val outDegrees: VertexRDD[Int] = graph.outDegrees
val degreeGraph = graph.outerJoinVertices(outDegrees) { (id, oldAttr, outDegOpt) =>
  outDegOpt match {
    case Some(outDeg) => outDeg
    case None => 0 // No outDegree means zero outDegree
  }
}
```
> 您可能已经注意到在上面的示例中使用的多个参数列表（例如，f(a)(b)）curried函数模式。 
虽然我们可以将 f(a)(b) 写成 f(a, b)，但这意味着b上的类型推断不依赖于a。
因此，用户需要为用户定义的函数提供类型注释：
```scala
val joinedGraph = graph.joinVertices(uniqueCosts,
  (id: VertexId, oldCost: Double, extraCost: Double) => oldCost + extraCost)
```

## 邻域聚合 Neighborhood Aggregation

许多图形分析任务中的关键步骤是聚合关于每个顶点的邻域的信息。 
例如，我们可能想知道每个用户拥有的关注者数量或每个用户的关注者的平均年龄。 
许多迭代图算法（例如，PageRank，最短路径和连通分量）重复地聚合相邻顶点的属性（例如，当前PageRank值，到源的最短路径和最小可到达顶点id）。
> 为了提高性能，主聚合运算符从 graph.mapReduceTriplets 更改为新 graph.AggregateMessages。 
虽然API的变化相对较小，但我们在下面提供了过渡指南。

### 聚合信息 (aggregateMessages)

GraphX中的核心聚合操作是 [aggregateMessages](https://github.com/apache/spark/blob/v2.3.1/graphx/src/main/scala/org/apache/spark/graphx/Graph.scala)。
此运算符将用户定义的 sendMsg 函数应用于图中的每个边三元组，然后使用 mergeMsg 函数在其目标顶点聚合这些消息。 
```scala
class Graph[VD, ED] {
  def aggregateMessages[Msg: ClassTag](
      sendMsg: EdgeContext[VD, ED, Msg] => Unit,
      mergeMsg: (Msg, Msg) => Msg,
      tripletFields: TripletFields = TripletFields.All)
    : VertexRDD[Msg]
}
```
用户定义的 sendMsg 函数接收 EdgeContext， 它公开源和目标属性以及edge属性和函数（sendToSrc和sendToDst）来将消息发送到源和目标属性。
可以将 sendMsg 当作 Map-Reduce 中的 Map 函数。
用户定义的 mergeMsg 函数接收两个发往同一顶点的消息，并生成一条消息。可以将 mergeMsg 当作 Map-Reduce 中的 Reduce 函数。
aggregateMessages运算符返回一个 VertexRDD[Msg] ，其中包含发往每个顶点的聚合消息（类型为Msg）。
未收到消息的顶点不包含在返回的 VertexRDD 中。

此外，aggregateMessages 采用可选的 tripletsFields，它指示在 EdgeContext 中访问哪些数据（比如只要源顶点属性，但不是目标顶点属性）。
tripletsFields 的可能选项在 [TripletFields](http://spark.apache.org/docs/latest/api/java/org/apache/spark/graphx/TripletFields.html) 中定义，默认值为 TripletFields.All，表示用户定义的 sendMsg 函数可以访问 EdgeContext 中的任何字段。
tripletFields 参数可用于通知 GraphX 只需要部分 EdgeContext，从而允许 GraphX 选择优化的连接策略。
例如，如果我们计算每个用户的关注者的平均年龄，我们只需要源字段，因此我们将使用 TripletFields.Src 来指示我们只需要源字段。
> 在早期版本的GraphX中，我们使用字节码检查来推断TripletField，但是我们发现字节码检查稍微不可靠，而是选择更明确的用户控制。

在下面的示例中，我们使用aggregateMessages运算符来计算每个用户的更高级关注者的平均年龄。
```scala
import org.apache.spark.graphx.{Graph, VertexRDD}
import org.apache.spark.graphx.util.GraphGenerators

// Create a graph with "age" as the vertex property.
// Here we use a random graph for simplicity.
val graph: Graph[Double, Int] =
  GraphGenerators.logNormalGraph(sc, numVertices = 100).mapVertices( (id, _) => id.toDouble )
// Compute the number of older followers and their total age
val olderFollowers: VertexRDD[(Int, Double)] = graph.aggregateMessages[(Int, Double)](
  triplet => { // Map Function
    if (triplet.srcAttr > triplet.dstAttr) {
      // Send message to destination vertex containing counter and age
      triplet.sendToDst((1, triplet.srcAttr))
    }
  },
  // Add counter and age
  (a, b) => (a._1 + b._1, a._2 + b._2) // Reduce Function
)
// Divide total age by number of older followers to get average age of older followers
val avgAgeOfOlderFollowers: VertexRDD[Double] =
  olderFollowers.mapValues( (id, value) =>
    value match { case (count, totalAge) => totalAge / count } )
// Display the results
avgAgeOfOlderFollowers.collect.foreach(println(_))
```
> 当消息们（和消息们总和）的大小恒定时（例如，浮动和添加而不是列表和连接），aggregateMessages操作最佳地执行。

### 计算度信息

常见的聚合任务是计算每个顶点的度数：每个顶点相邻的边数。
在有向图的上下文中，通常需要知道每个顶点的 in-degree，out-degree 和 total-degree。
GraphOps类包含一组运算符，用于计算每个顶点的度数。
例如，在下面我们计算最大的 in，out 和 total degree：
```scala
// Define a reduce operation to compute the highest degree vertex
def max(a: (VertexId, Int), b: (VertexId, Int)): (VertexId, Int) = {
  if (a._2 > b._2) a else b
}
// Compute the max degrees
val maxInDegree: (VertexId, Int)  = graph.inDegrees.reduce(max)
val maxOutDegree: (VertexId, Int) = graph.outDegrees.reduce(max)
val maxDegrees: (VertexId, Int)   = graph.degrees.reduce(max)
```

### 收集邻域信息

在某些情况下，通过在每个顶点收集相邻顶点及其属性来表达计算可能更容易。
这可以使用 collectNeighborIds 和 collectNeighbors 运算符轻松完成。
```scala
class GraphOps[VD, ED] {
  def collectNeighborIds(edgeDirection: EdgeDirection): VertexRDD[Array[VertexId]]
  def collectNeighbors(edgeDirection: EdgeDirection): VertexRDD[ Array[(VertexId, VD)] ]
}
```
> 这些运算符可能非常昂贵，因为它们复制信息并需要大量通信。如果可能，尝试直接使用aggregateMessages运算符表示相同的计算。

## 缓存与非缓存

在Spark中，默认情况下RDD不会保留在内存中。
为避免重新计算，必须多次使用它们时显式缓存它们（请参阅[Spark编程指南](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)）。
GraphX中的图表行为相同。多次使用图形时，请务必先在其上调用 _Graph.cache()_。

在迭代计算中，为了获得最佳性能，还可能需要非缓存。
默认情况下，缓存的RDD和图形将保留在内存中，直到内存压力迫使它们按LRU顺序逐出。
对于迭代计算，来自先前迭代的中间结果将填满缓存。
虽然它们最终会被逐出，但存储在内存中的不必要数据会减慢垃圾收集速度。
一旦不再需要中间结果，就 uncache 它们，这样子会使程序更有效率。
这涉及在每次迭代中实体化（caching 和 forcing）图形或RDD，uncache 所有其他数据集，并且仅在将来的迭代中使用实体化数据集。
但是，由于图形由多个RDD组成，因此很难正确地非持久化它们。
对于迭代计算，我们建议使用 Pregel API，它正确地解决了中间结果。

---

# Pregel API

图是本质上递归的数据结构，因为顶点的属性取决于它们的邻居的属性，而邻居的属性又取决于它们的邻居的属性。
因此，许多重要的图算法迭代地重新计算每个顶点的属性，直到达到定点条件。
一系列图形并行抽象已经被提出，用来表达这些迭代算法。
GraphX公开了Pregel API的变体。

在高层次上，GraphX中的Pregel运算符是一种大规模同步并行消息传递抽象，它受到图形拓扑的约束。
Pregel 运算符在一系列超级步骤（super steps）中执行，在超级步骤中，顶点从前一个超级步骤接收其入站消息的总和，计算顶点属性的新值，然后在下一个超级步骤中将消息发送到相邻顶点。
与 Pregel 不同，消息是作为边缘三元组的函数并行计算的，并且消息计算可以访问源和目标顶点属性。
在超级步骤中跳过未接收消息的顶点。
Pregel运算符终止迭代并在没有剩余消息时返回最终图。

> 注意，与更标准的 Pregel 实现不同，GraphX 中的顶点只能向邻近顶点发送消息，并且消息构造是使用用户定义的消息传递函数并行完成的。
这些约束允许在GraphX中进行额外的优化。

以下是Pregel运算符的类型签名以及其实现的草图
（注意：为了避免由于长谱系链而导致的 stackOverflowError，
pregel通过将*spark.graphx.pregel.checkpointInterval*设置为正数来定期支持检查点图和消息，例如10。
并使用 *SparkContext.setCheckpointDir(directory: String)*设置检查点目录）：

```scala
class GraphOps[VD, ED] {
  def pregel[A]
      (initialMsg: A,
       maxIter: Int = Int.MaxValue,
       activeDir: EdgeDirection = EdgeDirection.Out)
      (vprog: (VertexId, VD, A) => VD,
       sendMsg: EdgeTriplet[VD, ED] => Iterator[(VertexId, A)],
       mergeMsg: (A, A) => A)
    : Graph[VD, ED] = {
    // Receive the initial message at each vertex
    var g = mapVertices( (vid, vdata) => vprog(vid, vdata, initialMsg) ).cache()

    // compute the messages
    var messages = g.mapReduceTriplets(sendMsg, mergeMsg)
    var activeMessages = messages.count()
    // Loop until no messages remain or maxIterations is achieved
    var i = 0
    while (activeMessages > 0 && i < maxIterations) {
      // Receive the messages and update the vertices.
      g = g.joinVertices(messages)(vprog).cache()
      val oldMessages = messages
      // Send new messages, skipping edges where neither side received a message. We must cache
      // messages so it can be materialized on the next line, allowing us to uncache the previous
      // iteration.
      messages = GraphXUtils.mapReduceTriplets(
        g, sendMsg, mergeMsg, Some((oldMessages, activeDirection))).cache()
      activeMessages = messages.count()
      i += 1
    }
    g
  }
}
```
请注意，Pregel有两个参数列表： _graph.pregel(list1)(list2)_ 。
第一个参数列表包含配置参数，包括初始消息，最大迭代次数以及发送消息的边缘方向（默认情况下沿边缘）。
第二个参数列表包含用户定义的函数，用于接收消息（顶点程序vprog），计算消息（sendMsg）和组合消息mergeMsg。

我们可以使用 Pregel 运算符来表示以下示例中的单源最短路径等计算。

```scala
import org.apache.spark.graphx.{Graph, VertexId}
import org.apache.spark.graphx.util.GraphGenerators

// A graph with edge attributes containing distances
val graph: Graph[Long, Double] =
  GraphGenerators.logNormalGraph(sc, numVertices = 100).mapEdges(e => e.attr.toDouble)
val sourceId: VertexId = 42 // The ultimate source
// Initialize the graph such that all vertices except the root have distance infinity.
val initialGraph = graph.mapVertices((id, _) =>
    if (id == sourceId) 0.0 else Double.PositiveInfinity)
val sssp = initialGraph.pregel(Double.PositiveInfinity)(
  (id, dist, newDist) => math.min(dist, newDist), // Vertex Program
  triplet => {  // Send Message
    if (triplet.srcAttr + triplet.attr < triplet.dstAttr) {
      Iterator((triplet.dstId, triplet.srcAttr + triplet.attr))
    } else {
      Iterator.empty
    }
  },
  (a, b) => math.min(a, b) // Merge Message
)
println(sssp.vertices.collect.mkString("\n"))
```

---

# Graph 构造器

GraphX提供了几种从RDD或磁盘上的顶点和边集合构建图形的方法。

默认情况下，没有任何图形构建器会对图形的边重新分区; 相反，边缘保留在其默认分区中（例如HDFS中的原始块）。

*Graph.groupEdges* 要求对图表进行重新分区，因为它假定相同的边缘将在同一个分区上共存，因此在调用 groupEdges 之前必须调用 _Graph.partitionBy_。

```scala
object GraphLoader {
  def edgeListFile(
      sc: SparkContext,
      path: String,
      canonicalOrientation: Boolean = false,
      minEdgePartitions: Int = 1)
    : Graph[Int, Int]
}
```
GraphLoader.edgeListFile提供了一种从磁盘上的边缘列表加载图形的方法。它解析以下形式的（源顶点ID，目标顶点ID）对的邻接列表，跳过以＃开头的注释行：
```text
# This is a comment
2 1
4 1
1 2
```
它从指定的边创建一个Graph，自动创建边提到的任何顶点。
所有顶点和边缘属性都默认为1。

canonicalOrientation参数允许重定向正方向上的边（srcId < dstId），这是连接组件算法所需的。

minEdgePartitions参数指定要生成的最小边缘分区数;例如，如果HDFS文件具有更多块，则可能存在比指定更多的边缘分区。

```scala
object Graph {
  def apply[VD, ED](
      vertices: RDD[(VertexId, VD)],
      edges: RDD[Edge[ED]],
      defaultVertexAttr: VD = null)
    : Graph[VD, ED]

  def fromEdges[VD, ED](
      edges: RDD[Edge[ED]],
      defaultValue: VD): Graph[VD, ED]

  def fromEdgeTuples[VD](
      rawEdges: RDD[(VertexId, VertexId)],
      defaultValue: VD,
      uniqueEdges: Option[PartitionStrategy] = None): Graph[VD, Int]

}
```
_Graph.apply_ 允许从顶点和边的RDD创建图形。
任意被拾取的重复顶点们，以及在边中 RDD 存在但是在顶点 RDD 中不存在的顶点，都会被赋予默认属性。

Graph.fromEdges 允许仅从边缘的RDD创建图形，自动创建边缘提到的任何顶点并为其指定默认值。

Graph.fromEdgeTuples 允许仅从边缘元组的RDD创建图形，为边缘指定值1，并自动创建边缘提到的任何顶点并为其指定默认值。
它还支持重复删除边缘; 为了删除重复，可以将 PartitionStrategy 当作 uniqueEdges 参数传入（比如 _uniqueEdges = Some(PartitionStrategy.RandomVertexCut)_）。
一个分区策略是必须的，来在同一个分区上放置相同的边缘，以便对它们进行重复数据删除。


---

# 顶点和边的 RDDs

GraphX公开了图中存储的顶点和边的RDD视图。
但是，由于 GraphX 是在优化的数据结构中维护顶点和边缘的，并且为这些数据结构提供了额外的功能，因此顶点和边缘分别作为 VertexRDD 和 EdgeRDD 返回。
在本节中，我们将回顾这些类型中的一些其他有用功能。请注意，这只是一个不完整的列表，请参阅API文档以获取正式的操作列表。

## VertexRDDs

VertexRDD[A] 扩展 RDD[(VertexId，A)] 并添加保证每个 VertexId 仅出现一次的附加约束。
此外，VertexRDD[A] 表示一组顶点，每个顶点具有类型 A 的属性。
在内部，这是通过将顶点属性存储在可重用的 hash-map 数据结构中来实现的。
因此，如果两个 VertexRDD 来自相同的基础 VertexRDD（例如，通过filter或mapValues），则它们可以在恒定时间内 join 而无需散列评估（hash evaluations）。
为了利用这种索引数据结构，VertexRDD公开了以下附加功能：

```scala
class VertexRDD[VD] extends RDD[(VertexId, VD)] {
  // 过滤顶点集但保留内部索引
  def filter(pred: Tuple2[VertexId, VD] => Boolean): VertexRDD[VD]
  // 转换值而不更改id（保留内部索引）
  def mapValues[VD2](map: VD => VD2): VertexRDD[VD2]
  def mapValues[VD2](map: (VertexId, VD) => VD2): VertexRDD[VD2]
  // 根据VertexId显示此集合唯一的顶点
  def minus(other: RDD[(VertexId, VD)])
  // 从该集中删除出现在另一个集合中的顶点
  def diff(other: VertexRDD[VD]): VertexRDD[VD]
  // 加入利用内部索引来加速 join（实质上）
  def leftJoin[VD2, VD3](other: RDD[(VertexId, VD2)])(f: (VertexId, VD, Option[VD2]) => VD3): VertexRDD[VD3]
  def innerJoin[U, VD2](other: RDD[(VertexId, U)])(f: (VertexId, VD, U) => VD2): VertexRDD[VD2]
  // 使用此RDD上的索引来加速输入RDD上的`reduceByKey`操作。
  def aggregateUsingIndex[VD2](other: RDD[(VertexId, VD2)], reduceFunc: (VD2, VD2) => VD2): VertexRDD[VD2]
}
```

请注意，例如，过滤器运算符如何返回 VertexRDD。
过滤器实际上是使用BitSet实现的，因此重用索引并保留与其他VertexRDD快速连接的能力。
同样，mapValues 运算符不允许 map 函数更改 VertexId，从而允许重用相同的 HashMap 数据结构。
leftJoin 和 innerJoin 都能够识别何时连接从同一 HashMap 派生的两个 VertexRDD，并通过线性扫描实现连接，而不是昂贵的点查找。

aggregateUsingIndex 运算符可用于从 RDD[(VertexId，A)] 高效构造新的VertexRDD。
从概念上讲，如果我在一组顶点上构造了一个 VertexRDD[B]，这是一些 RDD[(VertexId，A)] 中顶点的超集，那么我可以重复使用该索引进行聚合，然后对 RDD[(VertexId，A)] 进行索引。
举个例子：

```scala
val setA: VertexRDD[Int] = VertexRDD(sc.parallelize(0L until 100L).map(id => (id, 1)))
val rddB: RDD[(VertexId, Double)] = sc.parallelize(0L until 100L).flatMap(id => List((id, 1.0), (id, 2.0)))
// rddB 应该有 200 个元素
rddB.count
val setB: VertexRDD[Double] = setA.aggregateUsingIndex(rddB, _ + _)
// setB 应该有 100 个元素
setB.count
// Joining A 和 B 现在会很快！
val setC: VertexRDD[Double] = setA.innerJoin(setB)((id, a, b) => a + b)
```

## EdgeRDDs

EdgeRDD[ED] 扩展了 RDD[Edge [ED]]，使用 [PartitionStrategy](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.graphx.PartitionStrategy) 中定义的各种分区策略之一来组织分区块中的边缘。
在每个分区内，边缘属性和邻接结构分别存储，以便在更改属性值时最大程度地重用。

EdgeRDD公开的三个附加功能是:
```scala
// 在保留结构的同时转换边缘属性
def mapValues[ED2](f: Edge[ED] => ED2): EdgeRDD[ED2]
// 反转边缘重用属性和结构
def reverse: EdgeRDD[ED]
// join 两个使用相同分区策略分区的EdgeRDD。
def innerJoin[ED2, ED3](other: EdgeRDD[ED2])(f: (VertexId, VertexId, ED, ED2) => ED3): EdgeRDD[ED3]
```

在大多数应用程序中，我们发现 EdgeRDD 上的操作是通过图形运算符完成的，或者依赖于基本RDD类中定义的操作。

---

# 优化表示

虽然分布式图形的GraphX表示中使用的优化的详细描述超出了本指南的范围，但一些高级别的理解可能有助于可伸缩算法的设计以及API的最佳使用。
GraphX采用顶点切割方法进行分布式图分区：

[![](/images/posts/edge_cut_vs_vertex_cut.png)](/images/posts/edge_cut_vs_vertex_cut.png)

GraphX不是沿着边缘分割图形，而是沿着顶点划分图形，这可以减少通信和存储开销。
从逻辑上讲，这对应于为机器分配边缘并允许顶点跨越多台机器。
分配边缘的确切方法取决于 PartitionStrategy ，并且对各种启发式方法有几种权衡。
用户可以通过使用 Graph.partitionBy 运算符重新分区图表来选择不同的策略。
默认分区策略是使用图形构造中提供的边的初始分区。
但是，用户可以轻松切换到GraphX中包含的2D分区或其他启发式方法。

[![](/images/posts/vertex_routing_edge_tables.png)](/images/posts/vertex_routing_edge_tables.png)

一旦边缘被分割，有效图形并行计算的关键挑战就是有效地将顶点属性与边缘连接起来。 
因为真实世界的图形通常具有比顶点更多的边缘，所以我们将顶点属性移动到边缘。 
因为并非所有分区都包含与所有顶点相邻的边，所以我们在内部维护一个路由表，该路由表标识在实现 triplets 和 aggregateMessages 等操作所需的连接时广播顶点的位置。

---

# 图算法

GraphX包含一组图算法，可简化分析任务。
算法包含在org.apache.spark.graphx.lib包中，可以通过GraphOps直接作为Graph上的方法访问。
本节介绍算法及其使用方法。

## PageRank

PageRank测量图中每个顶点的重要性，假设从u到v的边表示u对v的重要性的认可。
例如，如果Twitter用户跟随许多其他用户，则用户将被排名很高。

GraphX带有PageRank的静态和动态实现作为PageRank对象的方法。 
静态PageRank运行固定的迭代次数，而动态PageRank运行直到排名收敛（即停止更改超过指定的容差）。
GraphOps允许直接将这些算法作为Graph上的方法调用。

GraphX还包括一个我们可以运行PageRank的示例社交网络数据集。 
data/graphx/users.txt 中给出了一组用户，data/graphx/followers.txt 中给出了用户之间的一组关系。
我们按如下方式计算每个用户的 PageRank：

```scala
import org.apache.spark.graphx.GraphLoader

// 加载图的边
val graph = GraphLoader.edgeListFile(sc, "data/graphx/followers.txt")
// 运行 PageRank
val ranks = graph.pageRank(0.0001).vertices
// 使用 usernames 进行 Join 排名  
val users = sc.textFile("data/graphx/users.txt").map { line =>
  val fields = line.split(",")
  (fields(0).toLong, fields(1))
}
val ranksByUsername = users.join(ranks).map {
  case (id, (username, rank)) => (username, rank)
}
// 打印结果
println(ranksByUsername.collect().mkString("\n"))
```

## 连通器（Connected Components）

连通器算法使用其编号最小的顶点的ID标记图的每个连通分量。 
例如，在社交网络中，连接的组件可以近似聚类。 
GraphX包含ConnectedComponents对象中的算法实现，我们从PageRank部分计算示例社交网络数据集的连接组件，如下所示：
```scala
import org.apache.spark.graphx.GraphLoader

// Load the graph as in the PageRank example
val graph = GraphLoader.edgeListFile(sc, "data/graphx/followers.txt")
// Find the connected components
val cc = graph.connectedComponents().vertices
// Join the connected components with the usernames
val users = sc.textFile("data/graphx/users.txt").map { line =>
  val fields = line.split(",")
  (fields(0).toLong, fields(1))
}
val ccByUsername = users.join(cc).map {
  case (id, (username, cc)) => (username, cc)
}
// Print the result
println(ccByUsername.collect().mkString("\n"))
```

## 三角计数（Triangle Counting）

当顶点具有两个相邻顶点并且在它们之间具有边缘时，顶点是三角形的一部分。
GraphX 在 TriangleCount 对象中实现了一个三角形计数算法，该算法确定了通过每个顶点的三角形数量，从而提供了一种聚类度量。
我们从 PageRank 部分计算社交网络数据集的三角形计数。
请注意，TriangleCount要求边缘采用规范方向（srcId <dstId），并使用Graph.partitionBy对图形进行分区。
```scala
import org.apache.spark.graphx.{GraphLoader, PartitionStrategy}

// 按规范顺序加载边，并将图分区为三角形计数
val graph = GraphLoader.edgeListFile(sc, "data/graphx/followers.txt", true)
  .partitionBy(PartitionStrategy.RandomVertexCut)
// 找到每个顶点的三角形计数
val triCounts = graph.triangleCount().vertices
// 使用用户名加入三角形计数
val users = sc.textFile("data/graphx/users.txt").map { line =>
  val fields = line.split(",")
  (fields(0).toLong, fields(1))
}
val triCountByUsername = users.join(triCounts).map { case (id, (username, tc)) =>
  (username, tc)
}
// 打印结果
println(triCountByUsername.collect().mkString("\n"))
```

---

# 参考链接

1. [GraphX Programming Guide](http://spark.apache.org/docs/latest/graphx-programming-guide.html)