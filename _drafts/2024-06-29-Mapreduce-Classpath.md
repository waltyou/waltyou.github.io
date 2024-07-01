---
layout: post
title: MapReduce Classpath
date: 2024-06-29 13:39:04
author: admin
comments: true
categories: [Hadoop]
tags: [MapReduce]
---

最近工作中遇到一个关于MapReduce ClassPath的问题，记录一下。

<!-- more -->

---

* Outline
{:toc}
---

## 背景

公司有一个时间久远的大数据集群，Hadoop 版本为 2.x.x，在上面有一组运行了接近10年的MapReduce jobs，给公司提供一大批非常基础的数据指标。
而我负责维护它的工作，大部分时间它都运行良好，但是偶尔会有一些奇怪的问题发生。比如最近的job 突然开始失败，下面是报错信息：

```log
java.lang.IllegalArgumentException: The datetime zone id 'America/Punta_Arenas' is not recognised
```


## Reference

- https://hadoop.apache.org/docs/r3.2.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/DistributedCacheDeploy.html