---
layout: post
title: Faiss 中的线程与异步调用
date: 2018-09-21 09:49:04
author: admin
comments: true
categories: [Faiss]
tags: [Image Search]
---

来了解一下 Faiss 中的线程与异步调用。

<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 线程安全

Faiss CPU索引对于并发搜索以及不更改索引的其他操作是线程安全的。
多线程使用改变索引的函数需要实现互斥。

即使对于只读函数，Faiss GPU索引也不是线程安全的。
这是因为GPU Faiss的 StandardGpuResources 不是线程安全的。
必须为主动运行GPU Faiss索引的每个线程创建一个StandardGpuResources对象。
多个GPU索引由单个CPU线程管理并共享相同的StandardGpuResources（实际这是应该的，因为它们可以使用相同的GPU内存临时区域）。
单个GpuResources对象可以支持多个设备，但只能支持单个调用CPU线程。
多GPU Faiss（通过index_cpu_to_gpu_multiple获得）在内部运行来自不同线程的不同GPU索引。

---

# 内部线程

Faiss 本身有几种不同的内部线程。
对于 CPU Faiss，索引（训练，添加，搜索）的三个基本操作是内部多线程的。
线程是通过 OpenMP 和多线程 BLAS 实现完成的。 
Faiss 没有设置线程数。
调用者可以通过环境变量 OMP_NUM_THREADS 调整此数字，也可以随时调用 omp_set_num_threads（10）。
这个函数在 Python 中通过 faiss 提供。

对于 add 和 search 函数，线程在矢量上。
这意味着查询或添加单个向量不是多线程的。

---

# search 的性能

当批量提交查询时，获得 QPS 方面的最佳性能。

如果逐个提交查询，那么它们将在调用线程中执行（目前所有索引都是这种情况）。
因此，让多个线程调用单个查询的搜索也是相对有效的。

但是，从多个线程调用批量查询是非常低效的，这将产生比核心更多的线程。

---

# 内部线程的性能（OpenML）

选择 OpenMP 线程的数量并不总是很明显。
有一些架构，其中设置的线程数少于核心数，从而显着提高了执行效率。
例如，在Intel E5-2680 v2上，将线程数设置为20而不是默认值40是很有用的。

使用 OpenBLAS BLAS 实现时，将线程策略设置为 PASSIVE 是很有用的：
    
    export OMP_WAIT_POLICY=PASSIVE

---

# 异步搜索

与其他一些计算并行执行索引搜索操作可能很有用，包括：
- 单线程计算
- I/O 等待
- GPU 计算

这样，程序就可以并行运行。 

对于Faiss CPU，与其他多线程计算（例如其他搜索）并行化是没有用的，因为这会产生太多线程并降低整体性能; 
在递交给Faiss之前，应该将来自可能不同的用户线程的多个传入搜索排队并由用户聚合/批处理。

在多个GPU上并行运行操作当然是可行且有用的，其中每个CPU线程专用于在不同GPU上的内核启动，这就是IndexProxy和IndexShards的实现方式。

如何生成搜索线程：
- C++ 中， 使用 pthread_create + pthread_join
- Python 中， thread.start_new_thread + 一个锁， 或者使用 multiprocessing.dummy.Pool。
    search、add、train 函数都会释放 GIL （Global Interpreter Lock）。



