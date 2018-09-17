---
layout: post
title: Faiss 介绍
date: 2018-09-17 16:29:04
author: admin
comments: true
categories: [Faiss]
tags: [Image Search]
---

Faiss 是 FaceBook 开源的一个项目，主要为了图片相似性搜索。来看看。

<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 简介

## 是什么

图片相似度搜索框架。

## 开发语言

开发语言是 C++， 但是提供了 Python 接口。

## 优点

支持高维度、速度快、GPU 优化好。

---

# 上手

## 配置环境

坚持”能 docker ， 绝不 install“的原则，在 dockerHub 上搜索一下，找到自己想要的。

    docker pull docker pull illagrenan/faiss-python

## 官方 ”Hello World“ 例子

这里仅使用 python 版，如果需要了解 C++ 版，请参考[github wiki](https://github.com/facebookresearch/faiss/wiki/Getting-started).

Faiss 总体使用过程可以分为三步：

1. 构建训练数据（以矩阵形式表达）
2. 挑选合适的 Index （Faiss 的核心部件），将训练数据 add 进 Index 中。
3. Search，也就是搜索，得到最后结果

### 构建训练数据

```python
import numpy as np
d = 64                           # dimension
nb = 100000                      # database size
nq = 10000                       # nb of queries
np.random.seed(1234)             # make reproducible
xb = np.random.random((nb, d)).astype('float32')
xb[:, 0] += np.arange(nb) / 1000.
xq = np.random.random((nq, d)).astype('float32')
xq[:, 0] += np.arange(nq) / 1000.
```

上面的代码，生成了训练数据矩阵 xd ，以及查询数据矩阵 xq。

仅仅为了好玩，分别在 xd 和 xq 所有数据的第一个维度中添加了一个偏移量，并且偏移量随着 id 数字的增大而增大。

### 创建 Index 对象以及 add 训练数据

```python
import faiss                   # make faiss available
index = faiss.IndexFlatL2(d)   # build the index
print(index.is_trained)
index.add(xb)                  # add vectors to the index
print(index.ntotal)
```

Faiss 是围绕 Index 对象构建的。 
Faiss 也提供了许多种类的 Index， 这里简单起见，使用 IndexFlatL2： 一个蛮力L2距离搜索的索引。

所有索引都需要知道它们是何时构建的，它们运行的向量维数是多少（在我们的例子中是d）。 
然后，大多数索引还需要训练阶段（training phase），以分析向量的分布。 
对于IndexFlatL2，我们可以跳过此操作。

### Search

```python
k = 4                          # we want to see 4 nearest neighbors
D, I = index.search(xb[:5], k) # sanity check
print(I)
print(D)

D, I = index.search(xq, k)     # actual search
print(I[:5])                   # neighbors of the 5 first queries
print(I[-5:])                  # neighbors of the 5 last queries
```

可以对索引执行的基本搜索操作是k最近邻搜索，即：对于每个查询向量，在数据库中查找其k个最近邻居。
所以结果集应该是一个 size 为 nq * k 的矩阵。

上述代码，做了两次搜索。

第一次的查询数据，直接使用了训练数据的前5行，这样子更加有比较性。
I 和 D， 分别代表 Id 和 Distance， 也就是距离和邻居的id。结果分别如下：

    [[  0 393 363  78]
     [  1 555 277 364]
     [  2 304 101  13]
     [  3 173  18 182]
     [  4 288 370 531]]
    
    [[ 0.          7.17517328  7.2076292   7.25116253]
     [ 0.          6.32356453  6.6845808   6.79994535]
     [ 0.          5.79640865  6.39173603  7.28151226]
     [ 0.          7.27790546  7.52798653  7.66284657]
     [ 0.          6.76380348  7.29512024  7.36881447]]

I 直接显示了与搜索向量最相近的 4 个邻居的 ID。
D 显示了与搜索向量之间的距离。

第二次的搜索结果如下：

    [[ 381  207  210  477]
     [ 526  911  142   72]
     [ 838  527 1290  425]
     [ 196  184  164  359]
     [ 526  377  120  425]]
    
    [[ 9900 10500  9309  9831]
     [11055 10895 10812 11321]
     [11353 11103 10164  9787]
     [10571 10664 10632  9638]
     [ 9628  9554 10036  9582]]
