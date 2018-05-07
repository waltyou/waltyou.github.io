---
layout: post
title: Hadoop MapReduce 工作流程
date: 2018-05-02 16:16:04
author: admin
comments: true
categories: [Hadoop]
tags: [Hadoop, Big Data, MapReduce]
---

Mapreduce 作为hadoop的计算框架层， 是hadoop的核心之一。

今天来从InputFormat角度来简单看一看它的工作流程是什么。

<!-- more -->
---
## 目录
{:.no_toc}

* 目录
{:toc}
---
## 流程图

[![](/images/posts/MapReduceWorkFlow.jpg)](/images/posts/MapReduceWorkFlow.jpg)

## 过程步骤

1. client启动jvm，创建一个job和JobClient
2. JobClient向JobTracker申请一个job ID，来标识这个job
3. JobClient在HDFS上创建一个以Job ID命名的目录，同时将job运行时所有需要的资源上传，如jar包、配置文件、InputSplit等
4. 在上传资源结束后，JobClient向JobTracker submit job
5. JobTracker收到job，并且初始化job
6. JobTracker向HDFS上获取这个job的split信息,生成一系列待处理的Map和Reduce任务
7. JobTracker向TaskTracker分配任务。
8. TaskTracker读取HDFS获取任务相关资源
9. TaskTracker启动一个child JVM
10. child jvm中运行MapTask或者ReduceTask


## 注意
- 数据划分是在JobClient上完成的，它适用InputFormat将输入数据做一次划分，形成若干split。
- 在第7步，JobTracker会根据TaskTracker的地址，再结合上一步读到的split中location信息，来选择一个location离TaskTracker最近的map或reduce任务分配给它
- 在第10步，MapTask会使用InputSplit.getRecordReader()返回的RecordReader对象，来读取Split中的每一条记录。