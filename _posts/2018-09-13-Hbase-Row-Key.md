---
layout: post
title: Hbase RegionServer Size的调整
date: 2018-09-13 16:04:04
author: admin
comments: true
categories: [Hbase]
tags: [Big Data, Hbase]
---

针对具体数据，来选择合适的row key 设计，将会有助于提高读写效率。 
另外 hbase 其中也有一些高级功能可供选择，如 TTL、version、keep delete 等。


<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 关于列族的数量

HBase 目前不适用于两个或三个列族以上的任何内容，因此请保持模式中列族的数量较少。 
目前，flush 和 compaction 是按区域进行的，因此如果一个列族携带大量数据带来 flush，即使它们携带的数据量很小，相邻的族也将被 flush。 
当存在许多列族时，flush 和 compaction 交互可以产生一堆不必要的 I/O（通过改变 flush 和 compaction 以按列族工作来解决）。

如果您可以在 schema 中尝试使用一个列族。 
在数据访问通常是列作用域的情况下，仅引入第二和第三列族; 即您查询一个列族或另一个列族，但通常不是同时查询两个列族。

如果单个表中存在多个 ColumnFamilies，请注意基数（即行数）。 
如果 ColumnFamilyA 有100万行而 ColumnFamilyB 有10亿行，则ColumnFamilyA的数据可能会分布在许多区域（和RegionServers）中。 
这使得 ColumnFamilyA 的质量扫描效率降低。

---

# Rowkey 设计

## 1. 热点 Hotspotting

HBase 中的行按行按字典顺序排序。 
此设计优化了扫描（scan），允许您将相关的行或将要一起读取的行存储在彼此附近。 
但是，设计不良的行键是热点的常见来源。 
当大量客户端流量指向群集的一个节点或仅几个节点时，就会发生热点。 
此流量可能表示读取，写入或其他操作。 
流量超过负责托管该区域的单个机器，导致性能下降并可能导致区域不可用。 
这也可能对同一区域服务器托管的其他区域产生负面影响，因为该主机无法为请求的负载提供服务。 
设计数据访问模式非常重要，这样才能完全均匀地利用集群。

为了防止写入热点，设计行键使得真正需要位于同一区域的行是，但从更大的角度来看，数据被写入群集中的多个区域，而不是一次一个。 
下面描述了一些避免热点的常用技术，以及它们的一些优点和缺点。

### “加盐” Salting

在这种情况下，Salting 是指的是将随机数据添加到行键的开头。 
在这种情况下，salting 指的是向行键添加随机分配的前缀，以使其排序与其他方式不同。 
可能的前缀数量对应于您希望跨数据传播的区域数量。 
如果您有一些“热”行键模式在其他更均匀分布的行中反复出现，则Salting可能会有所帮助。 

请考虑以下示例，该示例显示salting可以在多个RegionServers之间传播写入负载，并说明了对读取的一些负面影响。

比如你设计了一张表，默认以 row key 的首字母分区，那么以下 row key，将被分在同一区中：

    foo0001
    foo0002
    foo0003
    foo0004

现在，想象一下，您希望将它们分布在四个不同的区域。 
您决定使用四种不同的随机数：a，b，c和d。 
在这种情况下，这些字母前缀中的每一个都将位于不同的区域。 
应用 salt 后，您将改为使用以下rowkeys。 
既然你现在可以写入四个不同的区域，理论上你在写入时的吞吐量是所有写入到同一区域时的吞吐量的四倍。

    a-foo0003
    b-foo0001
    c-foo0004
    d-foo0002

然后，如果添加另一行，将随机分配四个可能的盐值中的一个，并最终接近其中一个现有行。

    a-foo0003
    b-foo0001
    c-foo0003
    c-foo0004
    d-foo0002

由于此分配将是随机的，因此如果要按字典顺序检索行，则需要执行更多工作。 
通过这种方式，salting尝试增加写入的吞吐量，但在读取期间有成本。

### Hashing

您可以使用单向散列而不是随机赋值，这种散列可以让给定行始终使用相同的前缀“加盐”，从而将负载分散到RegionServers，同时也允许在读取期间进行预测。 
使用确定性散列允许客户端重建完整的rowkey并使用Get操作正常检索该行。

### 反转 key

防止热点的第三个常用技巧是反转固定宽度或数字行键，以便最常更改（最低有效数字）的部分是第一个。 
这有效地使行键随机化，但牺牲行排序属性。

## 2. 单调递增行键/时间序列数据

通常最好应该避免使用时间戳或序列（例如，1,2,3） 作为行键。

如果您确实需要将时间序列数据上传到HBase，您应该将 [OpenTSDB](http://opentsdb.net/) 作为一个成功的例子来学习。 
它有一个描述它在HBase中使用的 [schema](http://opentsdb.net/docs/build/html/user_guide/backends/hbase.html)的页面。 
OpenTSDB中 key 的格式实际上是 [metric_type] + [event_timestamp]，乍一看似乎与先前关于不使用时间戳作为关键字的建议相矛盾。 
然而，不同之处在于时间戳不在 key 的前导位置，并且设计假设是存在数十或数百（或更多）不同的 metric_type。 
因此，即使具有混合度量类型的连续输入数据流，Put 动作也分布在表中的各个区域点上。

## 3. 尽量减少行和列的大小

在HBase中，值总是用它们的坐标运算; 当一个单元格值通过系统时，它将伴随着它的行，列名和时间戳 - 总是如此。 
如果您的行和列名称很大，特别是与单元格值的大小相比，那么您可能会遇到一些有趣的场景。 
保留在HBase存储文件（StoreFile（HFile））上以便于随机访问的索引，可能最终占用HBase分配的RAM的大块，因为单元值坐标很大。 
上面引用的注释中的标记建议增加块大小，以便存储文件索引中的条目以更大的间隔发生，或者修改表模式，以便使较小的行和列名称。 
压缩也将使更大的指数。

### 使用字节模式

long 是 8 个字节。 
这8个字节中可以存储最多18,446,744,073,709,551,615个无符号数。 
如果将此数字存储为字符串 - 假设每个字符有一个字节 - 则需要将近3倍的字节。

```java
// long
long l = 1234567890L;
byte[] lb = Bytes.toBytes(l);
System.out.println("long bytes length: " + lb.length);   // returns 8

String s = String.valueOf(l);
byte[] sb = Bytes.toBytes(s);
System.out.println("long as string length: " + sb.length);    // returns 10

// hash
MessageDigest md = MessageDigest.getInstance("MD5");
byte[] digest = md.digest(Bytes.toBytes(s));
System.out.println("md5 digest bytes length: " + digest.length);    // returns 16

String sDigest = new String(digest);
byte[] sbDigest = Bytes.toBytes(sDigest);
System.out.println("md5 digest as string length: " + sbDigest.length);    // returns 26

```

不幸的是，使用字节模式，导致数据的可读性太差。

## 4. 反转时间戳

数据库处理中的一个常见问题是快速查找最新版本的值。 
使用反向时间戳作为 key 的一部分，可以极大地帮助解决此问题的特殊情况。
即：[key][reverse_timestamp]， 其中的 reverse_timestamp = Long.MAX_VALUE - timestamp。

这样子，只要通过执行Scan [key]并获取第一条记录，就可以找到表中[key]的最新值。

将使用此技术而不是使用“Number of Versions”，是为了“永久地”（或很长时间）保留所有版本，同时通过使用相同的扫描技术快速获得对任何其他版本的访问。

## 5. Rowkeys 和 ColumnFamilies

Rowkeys 的作用域为 ColumnFamilies。 
因此，在同一张表中，可以在每个 ColumnFamily 中存在相同的rowkey。

## 6. Rowkeys 的不变性

Rowkeys无法更改。 
它们可以在表中“更改”的唯一方法是删除行然后重新插入。 
因此首先要确保（和/或在插入大量数据之前），rowkeys正确是值得的。

## 7. RowKeys 与 Region Splits的关系

### 注意点一

预分割表通常是最佳实践，但您需要以这样的方式对它们进行预分割，即可以在键空间中访问所有区域。
注意不要预分割了十个区域，但是数据却只能落在其中两个上，产生热点问题。
一定要了解自己的数据。

### 注意点二

虽然通常不可取，但只要在 key 空间中可以访问所有创建的区域，使用十六进制键（以及更一般地说，可显示的数据）仍然可以使用预拆分表。

---

# Version 的数量

## 1. 最大版本数

最大版本数可以通过 HColumnDescriptor 来设置，默认为1。

不建议将最大版本的数量设置为非常高的级别（例如，数百或更多），除非这些旧值非常珍贵，因为这会大大增加StoreFile的大小。

## 2. 最小版本数

和最大版本数一样，也可以通过 HColumnDescriptor 来设置，默认为0，表示该功能已禁用。

最小行版本参数与生存时间参数一起使用，并且可以与行版本参数的数量组合以允许配置，例如“保留最后T分钟的数据，最多N个版本，但是 保持至少M个版本“（其中M是最小行数版本的值，M <N）。 
仅当为列族启用生存时间且必须小于行版本数时，才应设置此参数。

---

# 支持的数据类型

HBase通过Put和Result支持“bytes-in/bytes-out”接口，因此任何可以转换为字节数组的都可以存储为值。 
输入可以是字符串，数字，复杂对象，甚至是图像，只要它们可以呈现为字节。

## 1. Counters

值得特别提及的一种受支持的数据类型是“计数器”（即，能够进行数字的原子增量）。 

细节见 Table 中的 [Increment](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#increment%28org.apache.hadoop.hbase.client.Increment%29)。

计数器上的同步在RegionServer上完成，而不是在客户端中完成。

---

# Joins

如果您有多个表，请不要忘记考虑将Joins加入模式设计的可能性。

---

# 存活时间(TTL)

ColumnFamilies可以设置TTL长度（以秒为单位），HBase将在到达到期时间后自动删除行。 
这适用于行的所有版本。 
在HBase中为行编码的TTL时间以UTC指定。

在轻微压缩时删除仅包含过期行的存储文件。 
将 hbase.store.delete.expired.storefile 设置为false将禁用此功能。 
将最小版本数设置为0以外也会禁用此功能。

最近版本的 HBase 还支持基于每个单元格设置生存时间。 
有关更多信息，请参阅 [HBASE-10560](https://issues.apache.org/jira/browse/HBASE-10560)。 
使用 Mutation＃setTTL 将单元格的TTL作为突变请求（Appends，Increments，Puts等）的属性提交。 
如果设置了TTL属性，它将应用于操作在服务器上更新的所有单元格。 

单元格TTL处理和ColumnFamily TTL之间存在两个显着差异：
- Cell TTL 以毫秒而不是秒为单位表示。
- Cell TTL 不能延长 cell 的有效寿命超过ColumnFamily水平TTL设置。

---

# 保持删除的单元格

默认情况下，删除标记会延伸回到开头的时间。 
因此，即使 Get 或 Scan 操作是在删除标记之前执行的，它们也不会看到已删除的单元格（行或列）。

ColumnFamilies 可以选择保留已删除的单元格。 
在这种情况下，仍然可以检索已删除的单元格，只要这些操作执行的时间是在删除单元格的时间戳之前。 
这个特性使得即使存在删除，也允许进行时间点（point-in-time）查询。

删除的单元格仍然受TTL限制，并且永远不会有超过“最大版本数量”的已删除单元格。 
新的“原始” scan 选项将返回所有已删除的行和删除标记。

可以使用 HBase Shell 来更改 KEEP_DELETED_CELLS：

    hbase> hbase> alter ‘t1′, NAME => ‘f1′, KEEP_DELETED_CELLS => true

或者使用 API 来更改：
    
    HColumnDescriptor.setKeepDeletedCells(true);

---

# 参考

1. [Apache HBase ™ Reference Guide](http://hbase.apache.org/book.html) 

