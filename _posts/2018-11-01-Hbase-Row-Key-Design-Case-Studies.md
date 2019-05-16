---
layout: post
title: Hbase Row Key 设计实例学习
date: 2018-11-01 14:44:04
author: admin
comments: true
categories: [Hbase]
tags: [Big Data, Hbase]
---

这一篇主要学习一下Hbase官方文档上提到的几个 rowkey 设计的例子。

<!-- more -->

---



* 目录
{:toc}
---

# 总览

这次主要看五个例子。

- Log Data / Timeseries Data : 日志数据、时间序列数据
- Log Data / Timeseries on Steroids: 在Steroids上的日志数据、时间序列数据
- Customer/Order： 用户、订单
- Tall/Wide/Middle Schema Design：高、宽、中 schem 设计
- List Data：列表数据

---

# Log Data / Timeseries Data : 日志数据、时间序列数据

假设有以下数据将会被采集：
- Hostname
- Timestamp
- Log event
- Value/message

我们把它存储到表名为“LOG_DATA”的表中。那么该怎么设计rowkey呢？

## 1. Timestamp 放在 Rowkey 开头

如果 rowkey 为 [timestamp][hostname][log-event]， 那它就是一个单调递增的 rowkey。
这个设计可能会导致热点问题，详情可参考[这里](http://hbase.apache.org/book.html#timeseries).

改善这个情况的一种方法就是通过对时间戳执行mod操作。 
如果面向时间的扫描非常重要，那么这可能是一种有用的方法。 
必须注意桶的数量，因为这将需要相同数量的扫描来返回结果。
```java
long bucket = timestamp % numBuckets;
```
然后我们就可以把 rowkey 设计成为： [bucket][timestamp][hostname][log-event] 。

不过需要权衡的是，当要选择特定时间范围的数据，需要为每个存储桶执行扫描。 例如，100个存储桶将在密钥空间中提供广泛的分布，但是需要100个扫描才能获得时间戳的数据。

## 2. Host 放在 Rowkey 开头

即 rowkey 为 [hostname][timestamp][log-event]。

这种设计的使用要满足两个条件：
1. 必须是有大量不同 hostname
2. 按主机名扫描是优先事项

## 3. 反转 Timestamp ？

如果最重要的事情是找寻最近的事件，那么反转时间戳（timestamp = Long.MAX_VALUE – timestamp）将是有效的。只需轻松的 Scan 即可获取最近记录。

## 4. 可变长度或固定长度的Rowkeys？

记住，在HBase的每一列上都标记了rowkeys，这一点至关重要。

如果一个记录的 hostname 为 a ， event type 为 e1， 那 rowkey 的长度可以是很小的。
可是如果有另外一个记录的 hostname 为 myserver1.mycompany.com， event type 为 com.package1.subpackage2.subsubpackage3.ImportantService 呢? 

这时候就需要做一些转换。通常有两种做法：hash 和 数字化。

### hash

rowkey 可以是下列的组合：
- [MD5 hash of hostname] = 16 bytes
- [MD5 hash of event-type] = 16 bytes
- [timestamp] = 8 bytes

### 数字化

对于这种方法，除了LOG_DATA之外，还需要另一个查找表，称为LOG_TYPES。 LOG_TYPES的rowkey是：

- [type] (e.g., 表示 hostname 或 event-type 的字节)
- [bytes] 原始 hostname 、 event-type 的可变长字节.

rowkey 可以是下列的组合：

- [MD5 hash of hostname] = 16 bytes
- [MD5 hash of event-type] = 16 bytes
- [timestamp] = 8 bytes

---

# Log Data / Timeseries on Steroids: 在Steroids上的日志数据、时间序列数据

这实际上是OpenTSDB方法。

它将下列数据：

    [hostname][log-event][timestamp1]
    [hostname][log-event][timestamp2]
    [hostname][log-event][timestamp3]
    
将上述每个事件转换为以相对于开始时间范围的时间偏移存储的列（比如5分钟）：

    [hostname][log-event][timerange]
    
这显然是一种非常先进的处理技术，但HBase使这成为可能。

---

# Customer/Order： 用户、订单

假设HBase用于存储客户和订单信息。摄取了两种核心记录类型：客户记录类型和订单记录类型。

客户记录类型将包括您通常期望的所有内容： 
- Customer number
- Customer name
- Address (e.g., city, state, zip)
- Phone numbers, etc.

订单记录类型包括以下内容： 
- Customer number
- Order number
- Sales date
- A series of nested objects for shipping locations and line-items

假设客户编号和销售订单的组合唯一地标识订单，这两个属性将组成rowkey，特别是组合键，例如：
    
    [customer number][order number]

但是，还有更多的设计决策要做：原始值是rowkeys的最佳选择吗？

我们仍然要面对和日志数据用例中相同的设计问题：什么是客户编号的密钥空间，以及格式是什么（例如，数字？字母数字？）

因为在HBase中使用固定长度密钥是有利的，以及可以支持密钥空间中合理传播的密钥，类似的选择出现了：

- 带哈希的复合Rowkey： 

    [MD5 of customer number] = 16 bytes
    [MD5 of order number] = 16 bytes
    
- 复合数字/哈希组合Rowkey： 

    [substituted long for customer number] = 8 bytes
    [MD5 of order number] = 16 bytes

## 1. 单表？ 多表？

传统的设计方法将为CUSTOMER和SALES提供单独的表。
另一种选择是将多种记录类型打包到一个表中（例如，CUSTOMER ++）。

Customer Record Type Rowkey:
- [customer-id]
- [type] = type indicating `1' for customer record type

Order Record Type Rowkey:
- [customer-id]
- [type] = type indicating `2' for order record type
- [order]

这种特殊的CUSTOMER ++方法的优点是可以按客户ID组织许多不同的记录类型（例如，单次扫描可以获得有关该客户的所有信息）。 

缺点是扫描特定记录类型并不容易。

## 2. 订单对象设计

现在我们需要解决如何为Order对象建模。假设类结构如下：

Order：
    一个 order 有多个 ShippingLocations。
ShippingLocation：
    一个 ShippingLocation  有 LineItem。

### 完全标准化

使用这种方法，ORDER，SHIPPING_LOCATION 和 LINE_ITEM 会有单独的表。

ORDER表的rowkey如上所述。

- [order-rowkey]
- [ORDER record type]

SHIPPING_LOCATION的复合rowkey将是这样的：
- [order-rowkey]
- [shipping location number]

LINE_ITEM表的复合rowkey将是这样的：
- [order-rowkey]
- [shipping location number]
- [line item number]

这样的规范化模型可能是使用RDBMS的方法，但这不是您使用HBase的唯一选择。

这种方法的缺点是，要检索有关任何订单的信息，您将需要：
- 获取订单的ORDER表 
- 在SHIPPING_LOCATION表上扫描该订单以获取ShippingLocation实例 
- 在LINE_ITEM上扫描每个ShippingLocation

无论如何，这就是RDBMS所做的事情，但由于HBase中没有连接，你要更加注意到这一事实。

### 单表与记录类型

使用这种方法，将存在一个包含记录类型的单个表ORDER。

ORDER表的rowkey如上所述。
- [order-rowkey]
- [ORDER record type]

SHIPPING_LOCATION 的复合rowkey将是这样的：
- [order-rowkey]
- [SHIPPING record type]
- [shipping location number]

LINE_ITEM 的复合rowkey将是这样的：
- [order-rowkey]
- [LINE record type]
- [shipping location number] (e.g., 1st location, 2nd, etc.)
- [line item number]

### 反规范化

具有记录类型的单表方法的变体是对一些对象层次结构进行非规范化和展平，例如将ShippingLocation属性折叠到每个LineItem实例上。

LineItem复合rowkey将是这样的：
- [order-rowkey]
- [LINE record type]
- [line item number]（例如，第1个lineitem，第2个等，必须注意整个订单中是唯一的）

而 LineItem 列将是这样的：
- itemNumber
- quantity
- price
- shipToLine1 (denormalized from ShippingLocation)
- shipToLine2 (denormalized from ShippingLocation)
- shipToCity (denormalized from ShippingLocation)
- shipToState (denormalized from ShippingLocation)
- shipToZip (denormalized from ShippingLocation)

这种方法的优点包括一个不太复杂的对象层次结构，但其中一个缺点是，如果任何这些信息发生变化，更新会变得更加复杂。

### Object BLOB

使用这种方法，整个Order对象图以某种方式被视为BLOB。

例如，ORDER表的rowkey如上所述。
名为“order”的单个列将包含一个可以反序列化的对象，该对象包含容器Order，ShippingLocations和LineItems。

这里有很多选项：JSON，XML，Java Serialization，Avro，Hadoop Writables等。
所有这些都是相同方法的变体：将对象图编码为字节数组。

在对象模型发生变化的情况下，应该注意这种方法以确保向后兼容性，以便仍然可以从HBase中读回较旧的持久性结构。

专业人员能够以最少的 I/O 管理复杂的对象图（例如，在此示例中为单个HBase Get per Order），
但缺点包括前面提到的关于序列化的向后兼容性，序列化的语言依赖性（例如，Java序列化）的警告只适用于Java客户端），
事实上你必须反序列化整个对象以获取BLOB中的任何信息，并且难以获得像Hive这样的框架来处理像这样的自定义对象。

---

# Tall/Wide/Middle Schema Design：高、宽、中 schem 设计

本节将介绍其他架构设计问题，特别是有关高表和宽表的问题。

这些是一般准则，而不是法律 - 每个应用程序必须考虑自己的需求。

## 1. Rows vs. Versions

一个常见的问题是，应该选择 row 或HBase的  built-in-versioning 。 
行这种方式，需要在rowkey的某些部分中存储时间戳，以便它们不会在每次连续更新时覆盖。

偏好：行（一般来说）。

## 2. Rows vs. Columns

另一个常见问题是，是否应该更喜欢行或列。 

上下文通常在宽表的极端情况下，例如具有1行具有100万个属性，或者1百万行，每行1列。

偏好：行（一般来说）。 
需要明确的是，本指南在非常广泛的情况下，而不是在需要存储几十或一百列的标准用例中。 但是这两个选项之间还有一条中间路径，那就是“Rows as Columns”。

## 3. Rows as Columns

行与列之间的中间方法是打包数据，对于某些行，这些数据将成为列中的单独行。 

OpenTSDB是这种情况的最佳示例，其中单行表示定义的时间范围，然后将离散事件视为列。 

这种方法通常更复杂，可能需要重新编写数据的额外复杂性，但具有 I/O 效率的优势。

---

# List Data：列表数据

这个是一问一答的形式，最好看英文：[地址](http://hbase.apache.org/book.html#casestudies.schema.listdata)


---

# 参考

1. [Apache HBase ™ Reference Guide](http://hbase.apache.org/book.html) 

