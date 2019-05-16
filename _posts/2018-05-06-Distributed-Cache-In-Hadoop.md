---
layout: post
title: Hadoop中的分布式缓存
date: 2018-05-06 10:16:04
author: admin
comments: true
categories: [Hadoop]
tags: [Hadoop, Big Data, MapReduce]
---

Hadoop中的MapReduce有个一个很实用的机制，叫做分布式缓存（Distributed cache）。

那它是什么？怎么用？有什么特点和注意点？

<!-- more -->
---



* 目录
{:toc}
---

# 1. 什么是Hadoop中的分布式缓存（Distributed cache）

**Distributed cache** 是Hadoop中**MapReduce**计算框架中的一个部分。

它缓存了那些被application需要的文件，比如只读的文本文件、jar包、压缩包等。

一旦我们为我们的job缓存了一个文件，Hadoop就能使这个文件，在map/reduce任务运行时对所有datanode都可用。

因此，map和reduce任务从所有datanode上访问文件。


## 常见使用场景

1. 可以完成分布式文件共享
2. 在执行一些join操作时，将小表放入cache中，来提高连接效率。

## 几种共享模式的比较

1. 使用Configuration的set方法，只适合数据内容比较小的场景
2. 将共享文件放在HDFS上，每次都去读取，效率比较低
3. 将共享文件放在DistributedCache里，在setup初始化一次后，即可多次使用，缺点是不支持修改操作，仅能读取


---

# 2. 使用分布式缓存

## 1) 条件
首先，使用分布式缓存来缓存文件有两个条件：
- 文件必须可用
- 文件必须可以通过url访问，Url可以是https://或者http://。

满足上述条件后，当用户提议某个文件进行分布式缓存时，MapReduce job会在node上运行task前，将这些缓存文件复制到所有node上。

---

## 2）使用

1. 复制文件到HDFS：

    ```
    $ hdfs dfs-put/user/dataflair/lib/jar_file.jar
    ```
2. 设置application的jobConf：

    ```
    DistributedCache.addFileToClasspath(new Path("/user/dataflair/lib/jar-file.jar"),
                                                    conf)
    ```
    不过我发现在hadoop 2.x和3.x版本，DistributedCache已经显示弃用。

    弃用的原因是：

    > 在YARN中，所有文件都会创建一个符号链接。文件/目录的名称将是符号链接的名称。
     因为之前的DistributedCache，在同时缓存名字相同路径不同的文件时，如hdfs：//foo/bar.zip和hdfs：//bar/bar.zip， yarn只会显示一个warning，然后只下载其中一个文件。
     另外由于API的编写方式，映射程序代码可能不知道只有其中一个被下载，并且无法找到丢失的并毁灭它。
     这就是为什么我弃用它们而赞成让人们总是使用符号链接，这样行为总是一致的。

    所以可以按照下面代码使用：
    ```
    Job job = new Job();
    job.addCacheFile(new URI(fileName + "#symbolName"));
    ```

3. 在map或reduce中使用

    ```java
    File cacheFile = new File("symbolName");
    ```

## 3) 分布式缓存的大小

可以在文件**mapred-site.xml**中设置，默认为10GB。

---

# 3. 分布式缓存的优点

## 1）存储复杂的数据

它分发了简单、只读的文本文件和复杂类型的文件，如jar包、压缩包。这些压缩包将在各个slave节点解压。

## 2）数据一致性

Hadoop分布式缓存追踪了缓存文件的修改时间戳。

然后当job在执行时，它也会通知这些文件不能被修改。

使用hash 算法，缓存引擎可以始终确定特定键值对在哪个节点上。

所以，缓存cluster只有一个状态，它永远不会是不一致的。

## 3）单点失败

分布式缓存作为一个跨越多个节点独立运行的进程。

因此单个节点失败，不会导致整个缓存失败。

---

# 4. 分布式缓存的开销

MapReduce分布式缓存有一些开销，因此它要比进程内缓存慢上一些。

## 1）对象系列化

分布式缓存必须系列化对象。但是系列化机制有两个主要问题：
- 非常慢：序列化使用反射来检查运行时的信息类型。与预编译代码相比，反射是一个非常缓慢的过程。
- 非常笨重：序列化存储完整的类名，群集和程序集细节。它还在成员变量中存储对其他实例的引用。所有这些使得序列化非常庞大笨重。

---

# 5. 总结

Hadoop中的分布式缓存，它是Hadoop MapReduce框架支持的机制。

在Hadoop中使用分布式缓存，我们可以向所有工作节点广播小型或中等大小的文件（只读）。

作业成功运行后，分布式缓存文件将从工作节点中删除。

---

# 6. 参考链接

1. Distributed Cache in Hadoop: Most Comprehensive Guide: <https://data-flair.training/blogs/hadoop-distributed-cache/>
2. 如何使用Hadoop的DistributedCache: <http://qindongliang.iteye.com/blog/2038108>
3. MapReduce Tutorial: <https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html>
4. Distibuted Cache Compatability Issues: <https://issues.apache.org/jira/browse/MAPREDUCE-4493>
5. Example Hadoop Job that reads a cache file loaded from S3: <https://gist.github.com/twasink/8813628>