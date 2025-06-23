---
layout: post
title: Apache Pig 入门学习
date: 2019-07-07 09:43:04
author: admin
comments: true
categories: [Pig]
tags: [Big Data, Pig]
---

工作中需要用到 Pig，来学习一下。

<!-- more -->

* 目录
{:toc}
---

# 入门三问

## 1. 是什么

官方文档定义：

> **Apache Pig** is a platform for analyzing large data sets that consists of a high-level language for expressing data analysis programs, coupled with infrastructure for evaluating these programs. The salient property of Pig programs is that their structure is amenable to substantial parallelization, which in turns enables them to handle very large data sets.

是一个使用高层次语言来分析大数据的平台。它显著的一个特点就是它的结构可以直接地进行并行计算，这个特点对处理大数据很有帮助。

它允许用户使用类-Sql 语言（Pig Latin）来进行数据处理，这个语言也提供了强大方便的 UDF 功能，所以它也可以集成其他语言的实现的功能。

它最终会将处理逻辑转换为 MapReduce Job进行运行。

## 2. 为什么要有它

1. 减小非开发人员处理大数据的学习成本。
2. 平时除了需要处理结构化数据（hive），非结构化数据也很常见。
3. 需要处理多种来源的数据

## 3. 用它做什么

用类-Sql的语言 Pig Latin，方便非技术人员，将计算逻辑转换为 Mapreduce 任务，进行大规模数据处理。



# Pig Latin基本概念

## 1. 读取数据构建Relation

类似 Hive 中表的概念。

它可以从hdfs文件中构建：

```
truck_events = LOAD '/data/truck_event_text_partition.csv' USING PigStorage(',')
	AS (driverId:int, truckId:int, eventTime:chararray,
	eventType:chararray, longitude:double, latitude:double,
	eventKey:chararray, correlationId:long, driverName:chararray,
	routeId:long,routeName:chararray,eventDate:chararray);

DESCRIBE truck_events;
```

也可以从另外一个 relation 中构建：

```
truck_events_subset = LIMIT truck_events 100;
DESCRIBE truck_events_subset;
```

## 2. 处理数据

### DUMP 

展示所有数据：

```
DUMP truck_events_subset;
```

### 选择某些列

```
specific_columns = FOREACH truck_events_subset GENERATE driverId, eventTime, eventType;
DESCRIBE specific_columns;
```

### JOIN 操作

```
truck_events = LOAD '/user/maria_dev/truck_event_text_partition.csv' USING PigStorage(',')
  AS (driverId:int, truckId:int, eventTime:chararray,
  eventType:chararray, longitude:double, latitude:double,
  eventKey:chararray, correlationId:long, driverName:chararray,
  routeId:long,routeName:chararray,eventDate:chararray);
  
drivers =  LOAD '/user/maria_dev/drivers.csv' USING PigStorage(',')
  AS (driverId:int, name:chararray, ssn:chararray,
  location:chararray, certified:chararray, wage_plan:chararray);
  
join_data = JOIN  truck_events BY (driverId), drivers BY (driverId);
DESCRIBE join_data;
```

### ORDER BY

对数据排序：

```
drivers =  LOAD '/user/maria_dev/drivers.csv' USING PigStorage(',')
  AS (driverId:int, name:chararray, ssn:chararray,
  location:chararray, certified:chararray, wage_plan:chararray);
ordered_data = ORDER drivers BY name asc;
DUMP ordered_data;
```

### FILTER 和 GROUP BY

```
truck_events = LOAD '/user/maria_dev/truck_event_text_partition.csv' USING PigStorage(',')
  AS (driverId:int, truckId:int, eventTime:chararray,
  eventType:chararray, longitude:double, latitude:double,
  eventKey:chararray, correlationId:long, driverName:chararray,
  routeId:long,routeName:chararray,eventDate:chararray);
  
filtered_events = FILTER truck_events BY NOT (eventType MATCHES 'Normal');

grouped_events = GROUP filtered_events BY driverId;
DESCRIBE grouped_events;
DUMP grouped_events;
```



## 3. 落地数据

使用 STORE 命令：

```
STORE specific_columns INTO 'output/specific_columns' USING PigStorage(',');
```



更多的命令可以查看[这里](https://pig.apache.org/docs/latest/basic.html)。

# 参考

1. [Beginners Guide to Apache Pig](https://hortonworks.com/tutorial/beginners-guide-to-apache-pig/)