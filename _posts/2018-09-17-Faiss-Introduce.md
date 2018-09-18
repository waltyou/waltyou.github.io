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

## 1. 是什么

图片相似度搜索框架。

## 2. 开发语言

开发语言是 C++， 但是提供了 Python 接口。

## 3. 优点

支持高维度、速度快、GPU 优化好。

---

# 上手

## 1. 配置环境

坚持”能 docker ， 绝不 install“的原则，在 dockerHub 上搜索一下，找到自己想要的。

    docker pull docker pull illagrenan/faiss-python

## 2. 官方 ”Hello World“ 例子

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
Faiss 也提供了许多种类的 Index， 这里简单起见，使用 **_IndexFlatL2_**： 一个蛮力L2距离搜索的索引。

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
     
---

# 进一步

## 1. 如何更快

### 理论基础

为了加快搜索速度，我们可以按照一定规则或者顺序将数据集分段。 
我们可以在 d 维空间中定义 Voronoi 单元，并且每个数据库向量都会落在其中一个单元。 
在搜索时，查询向量 x， 可以经过计算，算出它会落在哪个单元格中。
然后我们只需要在这个单元格以及与它相邻的一些单元格中，进行与查询向量 x 的比较工作就可以了。

（这里可以类比 HashMap 的实现原理。训练就是生成 hashmap 的过程，查询就是 getByKey 的过程）。

上述工作已经通过 **_IndexIVFFlat_** 实现了。
这种类型的索引需要一个训练阶段，可以对与数据库矢量具有相同分布的任何矢量集合执行。
在这种情况下，我们只使用数据库向量本身。

IndexIVFFlat 同时需要另外一个 Index： quantizer， 来给 Voronoi 单元格分配向量。
每个单元格由质心（centroid）定义。
找某个向量落在哪个 Voronoi 单元格的任务，就是一个在质心集合中找这个向量最近邻居的任务。
这是另一个索引的任务，通常是 **_IndexFlatL2_**。

这里搜索方法有两个参数：nlist（单元格数量），nprobe （一次搜索可以访问的单元格数量，默认为1）。
搜索时间大致随着 nprobe 的值加上一些由于量化产生的常数，进行线性增长。

### 代码

```python
nlist = 100
k = 4
quantizer = faiss.IndexFlatL2(d)  # 另外一个 Index
index = faiss.IndexIVFFlat(quantizer, d, nlist, faiss.METRIC_L2)
       # 这里我们指定了 METRIC_L2, 默认它执行 inner-product 搜索。
assert not index.is_trained
index.train(xb)
assert index.is_trained

index.add(xb)                  # add may be a bit slower as well
D, I = index.search(xq, k)     # actual search
print(I[-5:])                  # neighbors of the 5 last queries
index.nprobe = 10              # default nprobe is 1, try a few more
D, I = index.search(xq, k)
print(I[-5:])                  # neighbors of the 5 last queries

```

### 运行结果

当 nprobe=1 的输出：

    [[ 9900 10500  9831 10808]
     [11055 10812 11321 10260]
     [11353 10164 10719 11013]
     [10571 10203 10793 10952]
     [ 9582 10304  9622  9229]]

结果与蛮力搜索类似，但不完全相同（见上文）。
这是因为一些结果不在完全相同的 Voronoi 单元格中。
因此，访问更多的单元格可能会有用。

把 nprobe 上升到10的结果：

    [[ 9900 10500  9309  9831]
     [11055 10895 10812 11321]
     [11353 11103 10164  9787]
     [10571 10664 10632  9638]
     [ 9628  9554 10036  9582]]
     
这个就是正确的结果。
请注意，在这种情况下获得完美结果仅仅是数据分布的工件，因为它在x轴上具有强大的组件，这使得它更容易处理。 
nprobe 参数始终是调整结果的速度和准确度之间权衡的一种方法。 
设置nprobe = nlist会产生与暴力搜索相同的结果（但速度较慢）。

## 2. 减少内存占用

### 理论基础

IndexFlatL2 和 IndexIVFFlat 都会存储所有的向量。
为了扩展到非常大的数据集，Faiss提供了一些变体，它们根据产品量化器（product quantizer）压缩存储的矢量并进行有损压缩。

矢量仍然存储在Voronoi单元中，但是它们的大小减小到可配置的字节数m（d必须是m的倍数）。

压缩基于[Product Quantizer](https://hal.inria.fr/file/index/docid/514462/filename/paper_hal.pdf)，
其可以被视为额外的量化水平，其应用于要编码的矢量的子矢量。

在这种情况下，由于矢量未精确存储，因此搜索方法返回的距离也是近似值。

### 代码

```python
nlist = 100
m = 8                             # number of bytes per vector
k = 4
quantizer = faiss.IndexFlatL2(d)  # this remains the same
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8)
                                    # 8 specifies that each sub-vector is encoded as 8 bits
index.train(xb)
index.add(xb)
D, I = index.search(xb[:5], k) # sanity check
print(I)
print(D)
index.nprobe = 10              # make comparable with experiment above
D, I = index.search(xq, k)     # search
print(I[-5:])

```

### 结果

结果看起来时这个样子的：

    [[   0  608  220  228]
     [   1 1063  277  617]
     [   2   46  114  304]
     [   3  791  527  316]
     [   4  159  288  393]]
    
    [[ 1.40704751  6.19361687  6.34912491  6.35771513]
     [ 1.49901485  5.66632462  5.94188499  6.29570007]
     [ 1.63260388  6.04126883  6.18447495  6.26815748]
     [ 1.5356375   6.33165455  6.64519501  6.86594009]
     [ 1.46203303  6.5022912   6.62621975  6.63154221]]

我们可以观察到我们正确找到了最近邻居（它是矢量ID本身），但是矢量与其自身的估计距离不是0，尽管它明显低于到其他邻居的距离。 
这是由于有损压缩造成的。

这里我们将64个32位浮点数压缩为8个字节，因此压缩因子为32。

搜索真实查询时，结果如下所示：

    [[ 9432  9649  9900 10287]
     [10229 10403  9829  9740]
     [10847 10824  9787 10089]
     [11268 10935 10260 10571]
     [ 9582 10304  9616  9850]]

它们可以与上面的IVFFlat结果进行比较。
对于这种情况，大多数结果都是错误的，但它们位于空间的正确区域，如10000左右的ID所示。
实际数据的情况更好，因为：
- 统一数据很难索引，因为没有可用于聚类或降低维度的规律性
- 对于自然数据，语义最近邻居通常比不相关的结果更接近。

### 简化索引构建

由于构建索引可能变得复杂，因此有一个工厂函数在给定字符串的情况下构造它们。
上述索引可以通过以下简写获得：

```python
index = faiss.index_factory(d, "IVF100,PQ8")
```

把"PQ8"替换为“Flat”，就可以得到一个 IndexFlat。

当需要预处理（PCA）应用于输入向量时，工厂就特别有用。
例如，要使用PCA投影将矢量减少到32D的预处理时，工厂字符串应该时："PCA32,IVF100,Flat"。

## 3. 使用 GPU

### 获取单个 GPU 资源

```python
res = faiss.StandardGpuResources()  # use a single GPU
```

### 使用GPU资源构建GPU索引

```python
# build a flat (CPU) index
index_flat = faiss.IndexFlatL2(d)
# make it into a gpu index
gpu_index_flat = faiss.index_cpu_to_gpu(res, 0, index_flat)
```

多个索引可以使用单个GPU资源对象，只要它们不发出并发查询即可。

### 使用

获得的GPU索引可以与CPU索引完全相同的方式使用：

```python
gpu_index_flat.add(xb)         # add vectors to the index
print(gpu_index_flat.ntotal)

k = 4                          # we want to see 4 nearest neighbors
D, I = gpu_index_flat.search(xq, k)  # actual search
print(I[:5])                   # neighbors of the 5 first queries
print(I[-5:])                  # neighbors of the 5 last queries
```

### 使用多个 GPU

使用多个GPU主要是声明几个GPU资源。
在python中，这可以使用index_cpu_to_all_gpus帮助程序隐式完成。

```python
ngpus = faiss.get_num_gpus()

print("number of GPUs:", ngpus)

cpu_index = faiss.IndexFlatL2(d)

gpu_index = faiss.index_cpu_to_all_gpus(  # build the index
    cpu_index
)

gpu_index.add(xb)              # add vectors to the index
print(gpu_index.ntotal)

k = 4                          # we want to see 4 nearest neighbors
D, I = gpu_index.search(xq, k) # actual search
print(I[:5])                   # neighbors of the 5 first queries
print(I[-5:])                  # neighbors of the 5 last queries
```

---

# 参考链接

1. [faiss wiki](https://github.com/facebookresearch/faiss/wiki)

