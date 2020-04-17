---
layout: post
title: MapReduce 中的二级排序 Secondary Sort
date: 2020-04-17 18:16:04
author: admin
comments: true
categories: [Hadoop]
tags: [Hadoop, Big Data, MapReduce]
---

<!-- more -->
---


* 目录
{:toc}
---

## 前言

在MapReduce中，Reduce接收到的是<key， list[value]>，其中的 list[value] 是无序的。但有时候，我们希望它是有序的，此时应该怎么办呢？

## 两种方法

### 1. 在reduce端自行排序

优点：简单明了
缺点：当reduce端接收的 list[value] 过大的时候，自己进行排序，可能会 run out of memory（java.lang.OutOfMemoryError or GC limit）。

### 2. Secondary Sort

这种方法主要是借助 MapReduce 框架对Reducer 的值进行排序，可以很好的扩展，不会产生内存不足错误。

具体流程可以概括为：将想要排序的值与key组合在一起，组成符合键，然后让框架帮我们排序。

##实例

### 1. 样例数据

气温数据：年、月、日、气温

```
2012, 01, 01, 5
2012, 01, 02, 45
2012, 01, 03, 35
2012, 01, 04, 10
...
2001, 11, 01, 46
2001, 11, 02, 47
2001, 11, 03, 48
2001, 11, 04, 40
...
2005, 08, 20, 50
2005, 08, 21, 52
2005, 08, 22, 38
2005, 08, 23, 70
```

我们希望mapreduce得到如下结果：

```
2012-01:  5, 10, 35, 45, ...
2001-11: 40, 46, 47, 48, ...
2005-08: 38, 50, 52, 70, ...
```

其实就是拿到每个月的所有气温，而且气温有低到高排序。

### 2. 创建组合Key： DateTemperaturePair

```java
 1 import org.apache.hadoop.io.Writable;
 2 import org.apache.hadoop.io.WritableComparable;
 3 ...
 4 public class DateTemperaturePair
 5    implements Writable, WritableComparable<DateTemperaturePair> {
 6
 7     private Text yearMonth = new Text();                 // natural key
 8     private Text day = new Text();
 9     private IntWritable temperature = new IntWritable(); // secondary key
10
11     ...
12
13     @Override
14     /**
15      * This comparator controls the sort order of the keys.
16      */
17     public int compareTo(DateTemperaturePair pair) {
18         int compareValue = this.yearMonth.compareTo(pair.getYearMonth());
19         if (compareValue == 0) {
20             compareValue = temperature.compareTo(pair.getTemperature());
21         }
22         //return compareValue;    // sort ascending
23         return -1*compareValue;   // sort descending
24     }
25     ...
26 }
```

### 3. 自定义 partitioner

partitioner负责分配map端的输出到各个reducer端。所以我们需要针对DateTemperaturePair重写一个partitioner。

```java
 1 import org.apache.hadoop.io.Text;
 2 import org.apache.hadoop.mapreduce.Partitioner;
 3
 4 public class DateTemperaturePartitioner
 5    extends Partitioner<DateTemperaturePair, Text> {
 6
 7     @Override
 8     public int getPartition(DateTemperaturePair pair,
 9                             Text text,
10                             int numberOfPartitions) {
11         // make sure that partitions are non-negative
12         return Math.abs(pair.getYearMonth().hashCode() % numberOfPartitions);
13      }
14 }
```

### 4. 自定义 Grouping comparator

这个类控制在一次对Reducer.reduce() 函数的调用中将哪些key分组在一起。

```java
 1 import org.apache.hadoop.io.WritableComparable;
 2 import org.apache.hadoop.io.WritableComparator;
 3
 4 public class DateTemperatureGroupingComparator
 5    extends WritableComparator {
 6
 7     public DateTemperatureGroupingComparator() {
 8         super(DateTemperaturePair.class, true);
 9     }
10
11     @Override
12     /**
13      * This comparator controls which keys are grouped
14      * together into a single call to the reduce() method
15      */
16     public int compare(WritableComparable wc1, WritableComparable wc2) {
17         DateTemperaturePair pair = (DateTemperaturePair) wc1;
18         DateTemperaturePair pair2 = (DateTemperaturePair) wc2;
19         return pair.getYearMonth().compareTo(pair2.getYearMonth());
20     }
21 }
```

### 5. 修改 MapReduce Job

```java
import org.apache.hadoop.mapreduce.Job;
...
Job job = ...;
...
job.setPartitionerClass(TemperaturePartitioner.class);
job.setGroupingComparatorClass(YearMonthGroupingComparator.class);
```



## 参考

[Secondary Sort](https://www.oreilly.com/library/view/data-algorithms/9781491906170/ch01.html)
