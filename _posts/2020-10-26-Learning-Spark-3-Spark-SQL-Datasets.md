---

layout: post
title: Learning Spark (3) - Spark SQL 和 Datasets
date: 2020-10-26 18:11:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Learning Spark]
---

进入第六章：《Spark SQL 和 Datasets》。

<!-- more -->

---

* 目录
{:toc}
---

## Datasets 和 DataFrames的内存管理

Spark是一个密集的内存分布式大数据引擎，因此对其内存的有效使用对其执行速度至关重要。在其整个发行历史中，Spark的内存使用已发生了重大变化：

- Spark 1.0使用基于RDD的Java对象进行内存存储，序列化和反序列化，这在资源和速度方面都非常昂贵。 另外，存储空间是分配在Java堆上的，因此您只能依靠JVM的大型数据集的垃圾回收（GC）。
- Spark 1.x引入了Project Tungsten。 它的突出特点之一是基于一种新的基于行的内部格式，可使用偏移量和指针在堆内存中对数据集和数据帧进行布局。 Spark使用一种称为 Encodes 的有效机制在JVM及其内部Tungsten格式之间进行序列化和反序列化。 将内存分配到堆外意味着Spark较少受到GC的阻碍。
- Spark 2.x引入了第二代Tungsten引擎，具有完整的代码生成和基于列的矢量化内存布局的功能。 该新版本以现代编译器的思想和技术为基础，还利用现代CPU和缓存体系结构，以“单指令，多数据”（SIMD）方法进行快速并行数据访问。

## Dataset Encoder

未完待续。。。

