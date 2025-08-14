---
layout: post
title: 精通 Apache Spark 源码 | 问题 04・二、RDD与抽象 | Dataset/DataFrame 内存优化（Encoders 与 Tungsten 格式转换）
date: 2025-05-19 13:39:04
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

### 相关背景知识

在 Apache Spark 中，Encoders 的主要功能是把 JVM 对象转化为 Tungsten 的内存格式，从而让数据能更高效地存储与处理。

Tungsten 是 Spark 引入的一种内存管理机制，它采用了列式存储和 off-heap 内存分配技术，极大地提升了内存使用效率和数据处理速度。与 JVM 对象那种占用空间大、访问效率低的存储方式不同，Tungsten 格式以紧凑的二进制形式存储数据，还能利用平台的 off-heap 内存功能。

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


工作流程示例

当执行Dataset[Person].write.parquet(...)操作时：

1. 类型解析：Spark 会分析Person类的结构，确定其对应的StructType（例如StructType(StructField("name", StringType), StructField("age", IntegerType))）。
2. 编码器生成：动态生成能将Person对象序列化为 Tungsten 格式的代码。
3. 数据转换：在运行过程中，利用生成的编码器把每个Person对象转换为二进制格式。
4. 存储优化：将二进制数据按列存储到 Parquet 文件中。

性能优势

* 减少 GC 压力：Tungsten 格式减少了 Java 对象的数量，从而降低了垃圾回收的频率。
* 高效的内存利用：通过紧凑的二进制表示，减少了内存占用。
* 快速数据处理：借助代码生成和直接内存访问，提升了处理速度。

### ExpressionEncoder