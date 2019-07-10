---
layout: post
title: HFile 底层存储格式
date: 2019-07-10 18:04:04
author: admin
comments: true
categories: [Hbase]
tags: [Big Data, Hbase]
---

HFile 的底层存储格式，一共有三个版本，按先后顺序来学习一下。

<!-- more -->

---

* 目录
{:toc}
---


# V1

由下而上来看，主要分为三层，分别为 data block，index，trailer。当然还有 FileInfo 层，Mete block 层，这些不怎么重要。

[![](/images/posts/hfile_v1.jpg)](/images/posts/hfile_v1.jpg) 

## 1. Data block

真正用来存储数据的一层，k-v结构。

k 为 rowkey + CF1:Col + version，v 为 value。

顺序存储。

## 2. Index 层

分为两类，data index：用来索引data block位置；Meta index：用来记录 meta 信息。

## 3. trailer 层

Trailer纪录了FileInfo、Data Index、Meta Index块的起始位置，Data Index和Meta Index索引的数量等。


# V2

## v1 的问题

启动时，需要将 index 层全部 load 进入内存，花费很多时间及内存。

## 四层

[![](/images/posts/hfile_v2.png)](/images/posts/hfile_v2.png) 

还是由下往上看。

### 1. Scanne block section

和 v1 的 data block 很像，存储实际的数据。

但是不一样的是，v2 的这一层，也会存储 data index，这里叫做 leaf index block。

这也是 v2 主要优化的点，它会将多个 data index block 用一个 inde block 来表示，这里叫做 root data index。因为一个 data index block 记录的其实是 startPosition、length、startkey，那么合并起来也简单，找到最靠前的 key 以及 startPosition，然后把 length 加起来就好。

### 2. Non-scanned block section

这一层是当 load-on-open 层放不下时，才会被启用。

### 3. Load-on-open section

这一层就存放着刚刚在第一层中提到的 root data index。
还有 meta index、file info、bloom filter metadata。

### 4. Trailer

记录 root data index 的位置。

# V3

V3版本基本和V2版本相比，并没有太大的改变，它在KeyValue(Cell)层面上添加了Tag数组的支持；并在FileInfo结构中添加了和Tag相关的两个字段。


# 参考

https://blog.csdn.net/zhaojianting/article/details/78480329
