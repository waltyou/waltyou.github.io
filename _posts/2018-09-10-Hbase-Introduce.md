---
layout: post
title: Hbase 介绍
date: 2018-09-10 16:04:04
author: admin
comments: true
categories: [Hbase]
tags: [Big Data, Hbase]
---

Hbase 是 Hadoop 生态中重要的一个组成部分，它作为一个 NoSql 数据库的角色存在，来解决大数据情景下的数据查询存储问题。

<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 简介

## HBase是什么?

HBase是建立在Hadoop文件系统之上的分布式面向列的数据库。
它利用了Hadoop的文件系统（HDFS）提供的容错能力。

## 特点

- Base线性可扩展。
- 它具有自动故障支持。
- 它提供了一致的读取和写入。
- 它集成了Hadoop，作为源和目的地。
- 客户端方便的Java API。
- 它提供了跨集群数据复制。

## 应用场景

- Apache HBase曾经是随机，实时的读/写访问大数据。
- 它承载在集群普通硬件的顶端是非常大的表。
- Apache HBase是此前谷歌Bigtable模拟非关系型数据库。 Bigtable对谷歌文件系统操作，同样类似Apache HBase工作在Hadoop HDFS的顶部。

---

# 架构

HBase有三个主要组成部分：client 客户端库，master server 主服务器和 Region Server 区域服务器。
区域服务器可以按要求添加或删除。

## 主服务器 Master Server

主服务器是:

- 分配区域给区域服务器并在Apache ZooKeeper的帮助下完成这个任务。
- 处理跨区域的服务器区域的负载均衡。它卸载繁忙的服务器和转移区域较少占用的服务器。
- 通过判定负载均衡以维护集群的状态。
- 负责模式变化和其他元数据操作，如创建表和列。

## 区域 Region

区域只不过是表被拆分，并分布在区域服务器。

## 区域服务器 Region Server

区域服务器拥有区域如下 -

- 与客户端进行通信并处理数据相关的操作。
- 句柄读写的所有Region的请求。
- 由区域大小的阈值决定的区域的大小。

## Zookeeper

- Zookeeper管理是一个开源项目，提供如维护配置信息，命名，提供分布式同步等服务
- Zookeeper代表不同区域的服务器短暂节点。主服务器使用这些节点来发现可用的服务器。
- 除了可用性，该节点也用于追踪服务器故障或网络分区。
- 客户端通过与zookeeper区域服务器进行通信。
- 在模拟和独立模式，HBase由zookeeper来管理。

---

# Data Model 及其操作

## 基本构成
 
### 表 Table 

包含了许多的row。

### 行 Row

HBase中的一行(row)由一个行键(row key)和一个或多个具有与之关联的值(value)的列(column)组成。

    row = row_key + [column(key, value)...]

行存储时，行按字母顺序排序。 因此，行键的设计非常重要。 目标是以相关行彼此靠近的方式存储数据。 

常见的行键模式是网站域。 如果您的行键是域，则应该反向存储它们（org.apache.www，org.apache.mail，org.apache.jira）。 这样，所有Apache域都在表中彼此靠近，而不是基于子域的第一个字母展开。

### 列 Column

HBase中的列由列族（Column Family）和列限定符（Column Qualifier）组成，它们由:(冒号）字符分隔。

### 列族 Column Family

出于性能考虑，列族通常物理地放置了一组列及其值。 
每个列族都有一组存储属性，例如是否应将其值缓存在内存中，如何压缩其数据或对其行键进行编码等。
表中的每一行都具有相同的列族，但给定的行可能不会在给定的列族中存储任何内容。

### 列限定符 Column Qualifier

将列限定符添加到列族中，是为了给特定的数据提供索引。
给定列族 content，列限定符可能是 content：html， 另一个可能是 content：pdf。 

虽然列族在创建表时是固定的，但列限定符是可变的，并且行之间可能有很大差异。

### Cell

Cell 是行，列族和列限定符的组合，并包含值和时间戳，表示值的版本。

### Timestamp 

时间戳与每个值一起写入，并且是给定版本的值的标识符。
默认情况下，timestamp表示数据写入RegionServer上的时间，但是当您将数据放入 cell 时，可以指定不同的时间戳值。

## 操

### Get

获取特定 row 的属性， 可以通过 [Table.get](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#get-org.apache.hadoop.hbase.client.Get-) 执行。

### Put

当 key 存在时，更新原有值，当 key 不存在的时，新增一个值。
可以通过 [Table.put](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#put-org.apache.hadoop.hbase.client.Put-)
或 [Table.batch](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#batch-java.util.List-java.lang.Object:A-) 执行。

### Scans

Scan 允许迭代多行以获取指定的属性。

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...

Table table = ...      // instantiate a Table instance

Scan scan = new Scan();
scan.addColumn(CF, ATTR);
scan.setRowPrefixFilter(Bytes.toBytes("row"));
ResultScanner rs = table.getScanner(scan);
try {
  for (Result r = rs.next(); r != null; r = rs.next()) {
    // process result...
  }
} finally {
  rs.close();  // always close the ResultScanner!
}
```

### Delete

删除特定的行。
可以通过 [Table.delete](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#delete-org.apache.hadoop.hbase.client.Delete-) 执行。


---

# HBase Shell

## 通用命令

- status: 提供HBase的状态，例如，服务器的数量。
- version: 提供正在使用HBase版本。
- table_help: 表引用命令提供帮助。
- whoami: 提供有关用户的信息。

## 数据定义语言 DDL

- create: 创建一个表。
- list: 列出HBase的所有表。
- disable: 禁用表。
- is_disabled: 验证表是否被禁用。
- enable: 启用一个表。
- is_enabled: 验证表是否已启用。
- describe: 提供了一个表的描述。
- alter: 改变一个表。
- exists: 验证表是否存在。
- drop: 从HBase中删除表。
- drop_all: 丢弃在命令中给出匹配“regex”的表。
- Java Admin API: 在此之前所有的上述命令，Java提供了一个通过API编程来管理实现DDL功能。 在这个org.apache.hadoop.hbase.client包中有HBaseAdmin和HTableDescriptor 这两个重要的类提供DDL功能。

## 数据操纵语言 DML

- put: 把指定列在指定的行中单元格的值在一个特定的表。
- get: 取行或单元格的内容。
- delete: 删除表中的单元格值。
- deleteall: 删除给定行的所有单元格。
- scan: 扫描并返回表数据。
- count: 计数并返回表中的行的数目。
- truncate: 禁用，删除和重新创建一个指定的表。
- Java client API: 在此之前所有上述命令，Java提供了一个客户端API来实现DML功能，CRUD（创建检索更新删除）操作更多的是通过编程，在org.apache.hadoop.hbase.client包下。 在此包HTable 的 Put和Get是重要的类。

---

# HBase Admin API

HBase是用Java编写的，因此它提供Java API和HBase通信。 Java API是与HBase通信的最快方法。下面给出的是引用Java API管理，涵盖用于管理表的任务。

细节可以查看 [官网Doc](https://hbase.apache.org/2.0/devapidocs/index.html)。

## Admin

HBaseAdmin是一个表示管理的类。 这个类属于 org.apache.hadoop.hbase.client 包。 使用这个类，可以执行管理员任务。 使用 Connection.getAdmin() 方法来获取管理员的实例。

使用它可以创建表、删除表、删除表的某个CF等。

## Descriptor

这个类包含一个HBase表的详细信息：

 - 所有列族的描述
 - 表是否为目录表 
 - 表是否为只读的 
 - 存储的最大尺寸 
 - 是否发生了区域分割 
 - 与之相关联的协同处理器等

---
 
# 表的设计

## 准备
 
首先要回答一下几个问题：
1. 行键结构应该是什么以及它应该包含什么？
2. 表中应该有多少列族？
3. 哪些数据属于哪个列族？
4. 每个列族中有多少列？
5. 列名应该是什么？ 虽然列名不需要在表创建时定义，您需要在编写或读取数据时了解它们。
6. 哪些信息应该进入 Cell？
7. 每个单元应存储多少个版本？

## 注意点

在HBase表中定义的最重要的是行键结构。
为了有效地定义，有必要预先定义访问模式（读取和写入）。
要定义模式，必须考虑有关HBase表的几个属性：

1. 索引仅基于 Key 完成。
2. 基于行键对表进行排序。表中的每个区域对对行键空间（由开始和结束行键标识）的一部分责任。该区域包含从开始键到结束键的行的排序列表。
3. HBase表中的所有内容都存储为 byte[]。 没有类型。
4. 仅在行级保证原子性。没有原子性保证跨行，这意味着没有多行事务。
5. 必须在创建表时预先定义列族。
6. 列限定符是动态的，也可以在写入时定义。 它们被当作 byte[] 存储起来， 所以你甚至也可以将数据放入其中。

## 总结

- 行键是HBase表设计中最重要的一个方面，它决定了应用程序与HBase表的交互方式。它们还会影响您从HBase中提取的性能。
- HBase表是灵活的，您可以以byte []的形式存储任何内容。
- 将具有类似访问模式的所有内容存储在同一列族中。
- 仅对 key 进行索引。使用它对您有利。
- 高的表可能会让您更快，更简单的操作，但您可以权衡原子性。宽表，每行有很多列，允许行级的原子性。
- 考虑如何在单个API调用中完成访问模式，而不是多个API调用。HBase没有跨行事务，您希望避免在客户端代码中构建该逻辑。
- 散列允许固定长度的键和更好的分布，但是消除了使用字符串作为键所暗示的顺序。
- 列限定符可用于存储数据，就像单元格本身一样。
- 列限定符的长度会影响存储空间，因为您可以将数据放入其中。访问数据时，长度也会影响磁盘和网络I / O成本。要简明扼要。
- 列系列名称的长度会影响通过线路发送到客户端的数据大小（在KeyValue对象中）。要简明扼要。


# 参考

1. [HBase教程](https://www.yiibai.com/hbase/)
2. [Apache HBase ™ Reference Guide](http://hbase.apache.org/book.html) 
3. [Introduction to Basic Schema Design](http://0b4af6cdc2f0c5998459-c0245c5c937c5dedcca3f1764ecc9b2f.r43.cf2.rackcdn.com/9353-login1210_khurana.pdf)

 
