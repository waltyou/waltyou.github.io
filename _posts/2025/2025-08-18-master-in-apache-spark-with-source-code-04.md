---
layout: post
title: 精通 Apache Spark 源码 | 问题 04・二、RDD与抽象 | Dataset/DataFrame 内存优化（Encoders 与 Tungsten 格式转换）
date: 2025-08-18 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第四个问题，属于第二大类"RDD与数据抽象"，问题描述为：

```markdown
**Dataset/DataFrame internals**: How do Encoders bridge JVM objects to Tungsten's memory format?  
   *Key classes:* `ExpressionEncoder`, `RowEncoder`
```

## 解答

### Tungsten

在 Apache Spark 中，Tungsten 项目是一个重要的优化项目，旨在大幅提升 Spark 应用程序的内存和 CPU 效率，从而使其性能更接近现代硬件的极限。简单来说，它就像一个引擎的升级，让 Spark 跑得更快、更高效。

Tungsten 项目的核心思想是，随着现代硬件的发展，I/O和网络传输不再是主要瓶颈，而CPU和内存成为了新的性能瓶颈。Tungsten通过一系列技术改进来解决这些问题。

#### Tungsten 的核心技术

Tungsten 项目主要通过以下几个关键技术实现其性能提升：

1. 内存管理和二进制处理（Off-Heap Memory Management and Binary Processing）

   * **消除 JVM 对象模型开销：** 在传统的 Spark 模式下，数据以 Java 对象的形式存储在堆内存中。这会带来额外的开销，比如对象头信息，并且会频繁触发垃圾回收（GC），这在处理海量数据时会成为一个严重的性能瓶颈。
   * **直接操作裸内存：** Tungsten 引入了一种新的内存管理机制，它使用 `sun.misc.Unsafe` 等技术，直接在堆外（off-heap）分配和操作内存。
   * **Tungsten Row Format（UnsafeRow）：** 数据不再是 Java 对象，而是以紧凑的、二进制格式（UnsafeRow）存储。这种格式非常节省空间，并且可以直接进行操作，无需频繁地序列化和反序列化，极大地减少了GC的压力和内存占用。
2. 缓存感知的计算（Cache-aware Computation）

   * **利用CPU缓存：** 现代CPU都有多级缓存（L1、L2、L3），访问缓存的速度远快于访问主内存。
   * **优化数据布局：** Tungsten 采用了一种对缓存友好的数据布局方式，使得相关数据在内存中更紧密地排列。这大大提高了缓存命中率，减少了CPU从主内存读取数据的时间，从而加快了计算速度。

3. 全程代码生成（Whole-Stage Code Generation）

   * **JIT编译：** Tungsten 将多个物理算子（如过滤、投影、聚合）融合为一个完整的代码块，并由JVM的JIT（即时编译）编译器进行编译。
   * **消除虚函数调用开销：** 传统的 Spark 执行模型中，每个算子都会有单独的函数调用，这会带来一定的虚函数调用开销。全程代码生成将多个步骤合并成一个单一的、高度优化的函数，消除了这种开销，使数据处理流程更加流畅。
   * **更快的执行：** 生成的 JVM 字节码是专门为特定查询优化的，可以更高效地执行，尤其是在复杂的查询中，性能提升非常显著。

#### Tungsten 的作用和好处

Tungsten 项目并非一个独立的组件，而是 Spark SQL 物理执行层的一个核心部分。它与 Catalyst 优化器紧密协作，共同提升 Spark 的性能。

* **Catalyst 优化器** 负责生成逻辑和物理查询计划，决定“如何做”。
* **Tungsten 引擎** 则负责高效地执行物理计划，决定“如何更高效地做”。

通过 Tungsten，Spark 能够：

* **大幅减少内存开销和GC压力**：通过堆外内存和紧凑的二进制格式，能够处理更大的数据集。
* **提高CPU效率**：利用缓存感知的计算和全程代码生成，充分利用现代硬件的性能。
* **在某些工作负载下实现数倍甚至十倍的性能提升**：尤其是在数据密集型的聚合、排序和连接操作中，效果非常明显。

总之，它通过在底层优化内存管理和CPU利用率，使得Spark能够更高效地处理大规模数据，为现代大数据应用提供了强大的动力。

### Encoders

Encoders 是 Spark SQL 里的组件，其核心任务是实现 JVM 对象和 Tungsten 格式之间的相互转换。它主要完成以下工作：

* 序列化：把 JVM 对象转成 Tungsten 的二进制格式。
* 反序列化：将 Tungsten 格式的数据还原为 JVM 对象。
* 模式映射：在转换过程中，建立起 JVM 类字段和 Spark SQL 数据类型之间的对应关系。

Encoder 会依据数据类型动态生成专门的编码器代码，以此优化序列化和反序列化的性能。以处理Person类为例：

```python
case class Person(name: String, age: Int)
```

```java
public void serialize(Person person, BinaryRow row) {
  row.setInt(0, person.age()); // 直接把int值写入二进制行
  row.setString(1, person.name()); // 高效地处理字符串
}
```

同时encoder 还做了以下优化：

1. 内存布局相关
   1. 列式存储：Tungsten 按列来组织数据，这样在进行分析操作时，能有效减少 I/O 和缓存缺失的情况。
   2. 消除对象头：Tungsten 格式不会存储 JVM 对象头，大大降低了空间开销。
   3. 固定字段偏移量：由于每个字段的偏移量是固定的，所以可以通过偏移量直接访问数据，无需进行方法调用。
2. 平台相关
   1. Unsafe API：借助sun.misc.Unsafe，可以直接进行内存操作，避免了 Java 堆内存管理带来的开销。
   2. 内存对齐：按照平台的要求对数据进行对齐，从而提高内存访问速度。

Encoder 的分类

* Primitive Encoders：用于处理基本数据类型，像Int、String等。
* Product Encoders：适用于处理case class这种结构化数据。
* Kryo Encoders：当没有预定义的编码器时，可以使用 Kryo 库来进行通用序列化。


工作流程示例, 当执行`Dataset[Person].write.parquet(...)`操作时：

1. 类型解析：Spark 会分析Person类的结构，确定其对应的StructType（例如`StructType(StructField("name", StringType), StructField("age", IntegerType))`）。
2. 编码器生成：动态生成能将Person对象序列化为 Tungsten 格式的代码。
3. 数据转换：在运行过程中，利用生成的编码器把每个Person对象转换为二进制格式。
4. 存储优化：将二进制数据按列存储到 Parquet 文件中。

性能优势:

* 减少 GC 压力：Tungsten 格式减少了 Java 对象的数量，从而降低了垃圾回收的频率。
* 高效的内存利用：通过紧凑的二进制表示，减少了内存占用。
* 快速数据处理：借助代码生成和直接内存访问，提升了处理速度。

### AgnosticEncoder 

AgnosticEncoder是 Spark SQL Catalyst 层的一个通用编码器抽象，主要作用是描述如何将任意类型的 JVM 对象与 Spark SQL 的内部二进制格式（如 InternalRow）进行互相转换，但本身不依赖具体实现。

核心点如下：

* AgnosticEncoder[T] 是一个特质（trait），继承自 Encoder[T]，定义了类型 T 的编码/解码能力。
* 它包含类型信息（如 dataType、clsTag），并描述了类型是否为原始类型（isPrimitive）、是否为结构体（isStruct）、是否支持宽松序列化（lenientSerialization）等。
* trait 只描述类型和结构，不直接实现序列化/反序列化逻辑，具体实现由 ExpressionEncoder 等类根据它生成 Catalyst 表达式。
* 文件中还定义了多种具体的 AgnosticEncoder 实现，如 Option、Array、Map、Product（case class/tuple）、Row、JavaBean、UDT、Enum 及各种基础类型的编码器。

这些编码器为 Spark SQL 支持复杂嵌套类型、集合类型、Java/Scala 类型互操作等提供了基础。

### ExpressionEncoder

ExpressionEncoder 是 Spark SQL Catalyst 中用于对象与内部行（InternalRow）之间序列化和反序列化的通用编码器。它通过 Catalyst 表达式（Expression）和代码生成机制，将 JVM 对象（如 case class、元组、JavaBean、基本类型等）转换为 Spark SQL 的内部二进制格式，或反向转换。

总结：
AgnosticEncoder 是一种通用的编码器抽象，定义了类型 T 如何与 Spark SQL 的内部数据类型（如 InternalRow）进行互转的元信息，但不包含具体的序列化/反序列化实现。它只描述类型结构、数据类型、是否为原始类型等。
ExpressionEncoder 则是具体的实现，基于 AgnosticEncoder 提供的信息，生成 Catalyst 表达式（如 objSerializer 和 objDeserializer），并通过代码生成实现 JVM 对象与 Spark 内部格式的高效转换。

两者关系如下：

* AgnosticEncoder 负责描述类型和结构，是“元数据”层。
* ExpressionEncoder 负责利用这些描述，生成实际的序列化/反序列化逻辑，是“实现”层。

在源码中，ExpressionEncoder.apply 方法会接收一个 AgnosticEncoder，并基于它构建出 ExpressionEncoder。简言之：AgnosticEncoder 提供类型描述，ExpressionEncoder 负责具体实现，两者配合实现 Spark SQL 的类型安全与高效序列化。