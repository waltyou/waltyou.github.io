---
layout: post
title: Spark 三种部署模式：YARN，Mesos，Standalone 介绍
date: 2018-07-24 17:59:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark]
---

Spark 的 Cluster Manager 有三种类型： Spark Standalone cluster, YARN mode, and Spark Mesos。来看看都是什么。

<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# Cluster Manager

Spark 是个大数据处理工具，那么它必然能以分布式模式运行在集群之上。 集群通常由一个 master 和多个 worker 组成。

Cluster Manager 的作用就是负责在不同的应用之间调度和划分资源，同时也为集群分配任务。

Spark 支持可插拔的 Cluster Manager， 并且 spark 中的 Cluster Manager 负责初始化executor进程。

Spark 支持的有三种，如下。

---

# Standalone

Standalone 模式是最简单的一个 cluster manager。 它可以运行在各种操作系统上。

## 1. 工作原理

它具有提前配置好内存和CPU内核数量的 master 和 workers。

在这种模式下， Spark 通过 core 来分配资源。默认情况下，一个应用会获取集群中所有的core。

在 Standalone cluster manager 中，Zookeeper 仲裁使用 standby master 恢复 master。 
通过使用文件系统，我们可以实现 master 节点的手动恢复。 
Spark通过整个集群管理器的共享密钥来支持身份验证。
用户使用共享密钥配置每个节点。 对于通信协议，数据使用SSL加密。 但对于块传输，它使用数据SASL加密。

为了检查应用程序，每个Apache Spark应用程序都有一个Web用户界面。 
Web UI提供一个应用程序中必要的信息，比如 executors 状态，存储使用情况，运行任务情况。
在此集群管理器中，我们使用Web UI查看集群和作业统计信息。 它还为每个作业提供详细的日志输出。 
如果应用程序在其生命周期内记录了事件，则Spark Web UI将在应用程序退出后重新构建应用程序的UI。

---

# Apache Mesos

Mesos通过动态资源共享和隔离来处理分布式环境中的工作负载。 
它在大规模集群环境中部署和管理应用程序非常有用。 

Apache Mesos将集群中所有机器/节点的现有资源聚集在一起。 由此，多种工作负载策略都可以使用。
这是一种节点抽象，它减少了为不同工作负载分配特定机器的开销。 
它是Hadoop和大数据集群的资源管理平台。 Twitter，Xogito和Airbnb等公司使用Apache Mesos，因为它可以在Linux或Mac OSX上运行。

在某种程度上，Apache Mesos与虚拟化相反。 
这是因为在虚拟化中，一个物理资源划分为许多虚拟资源。 在Mesos中，许多物理资源都是一个虚拟资源。

## 1. 工作原理

Apache Mesos 有三个组件： Mesos masters, Mesos slave, Frameworks.

**Mesos Master** 是集群的一个实例。 一个集群拥有很多 Mesos Master， 以用来容错。 但是它们之中只有一个是领头的master。

**Mesos slave** 是集群中提供资源的那些实例。 Mesos master 会给 Mesos slave 安排任务。

**Mesos Framework** 允许应用可以从集群中请求资源。

Mesos的其他一些框架是Chronos，Marathon，Aurora，Hadoop，Spark，Jenkins等。

---

# Hadoop YARN

Yarn 是 Mapreduce 2.0 后出现的一个组件，它用来进行资源管理。详细参考[这里](/Hadoop-Yarn)

---

# 三者对比

来比较一下这三者，可以帮助我们更好地挑选合适的集群管理者。

## 1. 高可用性

**Standalone**： 

使用 ZooKeeper Quorum，它支持自动恢复主服务器。也可以使用文件系统实现手动恢复。尽管主服务器的恢复正在启用，但集群仍能容忍工作人员故障。

**Apache Mesos**： 

使用 Apache ZooKeeper，它支持自动恢复主服务器。在故障转移的情况下，当前正在执行的任务不会停止执行。

**Apache Hadoop YARN**： 

使用命令行实用程序，它支持手动恢复。并使用嵌入在ResourceManager中的基于Zookeeper的ActiveStandbyElector进行自动恢复。因此，无需运行单独的ZooKeeper故障转移控制器。

## 2. 安全

**Standalone**： 

要求用户使用共享密钥配置每个节点，使用SSL为通信协议加密数据，数据块传输支持SASL加密，其他选项也可用于加密数据。 可以通过访问控制列表控制对Web UI中的Spark应用程序的访问。

**Apache Mesos**： 

对于与集群交互的任何实体，Mesos提供身份验证。 这包括向主服务器注册的从服务器，提交给集群的框架以及使用端点（如HTTP端点）的操作者。 
这些实体可以启用是否使用身份验证。 
自定义模块可以取代Mesos的默认验证模块Cyrus SASL。
为了允许访问Mesos Access中的服务，它使用控制列表。 默认情况下，Mesos中模块之间的通信未加密。 可以启用SSL / TLS来加密通信。 Mesos WebUI支持HTTPS。

**Apache Hadoop YARN**： 

它包含身份验证，服务级别授权，Web控制台身份验证和数据机密性的安全性。
使用服务级别授权可确保使用Hadoop服务的客户端具有权限。 使用访问控制列表可以控制Hadoop服务。 
此外，通过使用SSL，客户端与服务之间的通信和数据也保证是加密的。 而且它使用HTTPS在Web控制台和客户端之间传输的数据。

## 3. 监控 

**Standalone**： 

它拥有可以查看群集和作业统计信息的Web UI。它还为每个作业提供详细的日志输出。如果应用程序在其生命周期内记录了事件，则Spark Web UI将在应用程序存在后重建应用程序的UI。

**Apache Mesos**： 

它支持每个容器网络监控和隔离。它通过URL来获取主节点和从节点的许多指标。这些指标包括分配的CPU的百分比和数量，内存使用情况等。

**Apache Hadoop YARN**： 

具有ResourceManager和NodeManager的Web UI。 ResourceManager UI提供群集的度量标准。 NodeManager为节点上运行的每个节点，应用程序和容器提供信息。

---

# 参考链接

1. [Apache Spark Cluster Managers – YARN, Mesos & Standalone](https://data-flair.training/blogs/apache-spark-cluster-managers-tutorial/)