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

## 未完待续......


---

# 参考链接

1. [GraphX Programming Guide](http://spark.apache.org/docs/latest/graphx-programming-guide.html)