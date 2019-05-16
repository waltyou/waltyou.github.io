---
layout: post
title: Mastering Apache Spark 总览
date: 2018-12-25 14:00:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Mastering Apache Spark]
---

在网上发现一篇gitbook，名为：《Mastering Apache Spark》，来学习一下。

<!-- more -->



* 目录
{:toc}

------

# 简介

这本书的地址是:  https://legacy.gitbook.com/book/jaceklaskowski/mastering-apache-spark/details

基于spark 2.3.2。

# 书籍目录

### SPARK CORE 

####  从集群中转移数据块

- ShuffleClient — 负责拉取shuffle的blocks
- NettyBlockTransferService — 基于Netty的 BlockTransferService
- BlockFetchingListener
- RetryingBlockFetcher

#### WEB UI：Spark 应用的web控制台

- Web UI
- JobsTab
- StagesTab — Stages for All Jobs 所有任务的 stages
- StorageTab
- EnvironmentTab
- ExecutorsTab
- SparkUI
- BlockStatusListener Spark Listener
- EnvironmentListener Spark Listener
- ExecutorsListener Spark Listener
- JobProgressListener Spark Listener
- StorageStatusListener Spark Listener
- StorageListener — 追踪RDD Blocks持久化状态的Listener
- RDDOperationGraphListener Spark Listener
- WebUI — Framework For Web UIs
- RDDStorageInfo
- RDDInfo
- LiveEntity
- UIUtils
- JettyUtils
- web UI Configuration Properties

####  METRICS

- Spark Metrics
- MetricsSystem
- MetricsConfig — Metrics System Configuration
- Source — Contract of Metrics Sources
- Sink — Contract of Metrics Sinks
- Metrics Configuration Properties

#### STATUS REST API

- Status REST API — Monitoring Spark Applications Using REST API
- ApiRootResource — /api/v1 URI Handler
- AbstractApplicationResource
- BaseAppResource
- ApiRequestContext
- UIRoot — Contract for Root Contrainers of Application UI Information

#### TOOLS

- Spark Shell — spark-shell shell script
- Spark Submit — spark-submit shell script
- spark-class shell script
- SparkLauncher — Launching Spark Applications Programmatically

#### ARCHITECTURE

- Spark Architecture
- Driver
- Executor
- Master
- Workers

#### RDD

- Anatomy of Spark Application
- SparkConf — Programmable Configuration for Spark Applications
- SparkContext
- RDD — Resilient Distributed Dataset
- Operators
- Caching and Persistence
- Partitions and Partitioning
- Shuffling
- Checkpointing
- RDD Dependencies
- Map/Reduce-side Aggregator
- AppStatusStore
- AppStatusPlugin
- AppStatusListener
- KVStore
- InterruptibleIterator — Iterator With Support For Task Cancellation

#### OPTIMIZATIONS

- Broadcast variables
- Accumulators

#### SERVICES

- SerializerManager
- MemoryManager — Memory Management
- SparkEnv — Spark Runtime Environment
- DAGScheduler — Stage-Oriented Scheduler
- TaskScheduler — Spark Scheduler
- SchedulerBackend — Pluggable Scheduler Backends
- ExecutorBackend — Pluggable Executor Backends
- BlockManager — Key-Value Store of Blocks of Data
- MapOutputTracker — Shuffle Map Output Registry
- ShuffleManager — Pluggable Shuffle Systems
- Serialization
- ExternalClusterManager — Pluggable Cluster Managers
- BroadcastManager
- ContextCleaner — Spark Application Garbage Collector
- Dynamic Allocation (of Executors)
- HTTP File Server
- Data Locality
- Cache Manager
- OutputCommitCoordinator
- RpcEnv — RPC Environment
- TransportConf — Transport Configuration
- Utils Helper Object

#### SECURITY

- Securing Web UI

#### SPARK DEPLOYMENT ENVIRONMENTS

- Deployment Environments — Run Modes
- Spark local (pseudo-cluster)
- Spark on cluster

#### Spark on YARN

- YarnShuffleService — ExternalShuffleService on YARN
- ExecutorRunnable
- Client
- YarnRMClient
- ApplicationMaster
- YarnClusterManager — ExternalClusterManager for YARN
- TaskSchedulers for YARN
- SchedulerBackends for YARN
- YarnAllocator
- Introduction to Hadoop YARN
- Setting up YARN Cluster
- Kerberos
- ClientDistributedCacheManager
- YarnSparkHadoopUtil
- Settings

#### SPARK STANDALONE

- Spark Standalone
- Standalone Master — Cluster Manager of Spark Standalone
- Standalone Worker
- web UI
- LocalSparkCluster — Single-JVM Spark Standalone Cluster
- Submission Gateways
- Management Scripts for Standalone Master
- Management Scripts for Standalone Workers
- Checking Status
- Example 2-workers-on-1-node Standalone Cluster (one executor per worker)
- StandaloneSchedulerBackend

#### SPARK ON MESOS

- Spark on Mesos
- MesosCoarseGrainedSchedulerBackend
- About Mesos

### EXECUTION MODEL

- Execution Model

### MONITORING, TUNING AND DEBUGGING

- Unified Memory Management
- Spark History Server
- Logging
- Performance Tuning
- SparkListener — Intercepting Events from Spark Scheduler
- JsonProtocol
- Debugging Spark

### VARIA 杂项

- Building Apache Spark from Sources
- Spark and Hadoop
- Spark and software in-memory file systems
- Spark and The Others
- Distributed Deep Learning on Spark
- Spark Packages

### INTERACTIVE NOTEBOOKS

- Interactive Notebooks
  - [Apache Zeppelin](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/interactive-notebooks/apache-zeppelin.html)
  - [Spark Notebook](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/interactive-notebooks/spark-notebook.html)

### SPARK TIPS AND TRICKS

- Spark Tips and Tricks
- Access private members in Scala in Spark shell
- SparkException: Task not serializable
- Running Spark Applications on Windows

### EXERCISES

- One-liners using PairRDDFunctions
- Learning Jobs and Partitions Using take Action
- Spark Standalone - Using ZooKeeper for High-Availability of Master
- Spark’s Hello World using Spark shell and Scala
- WordCount using Spark shell
- Your first complete Spark application (using Scala and sbt)
- Spark (notable) use cases
- Using Spark SQL to update data in Hive using ORC files
- Developing Custom SparkListener to monitor DAGScheduler in Scala
- Developing RPC Environment
- Developing Custom RDD
- Working with Datasets from JDBC Data Sources (and PostgreSQL)
- Causing Stage to Fail

### SPARK MLLIB

- Spark MLlib — Machine Learning in Spark
- ML Pipelines (spark.ml)
- ML Persistence — Saving and Loading Models and Pipelines
- Example — Text Classification
- Example — Linear Regression
- Logistic Regression
- Latent Dirichlet Allocation (LDA)
- Vector
- LabeledPoint
- Streaming MLlib
- GeneralizedLinearRegression
- Alternating Least Squares (ALS) Matrix Factorization
- Instrumentation
- MLUtils

### FURTHER LEARNING

- [Courses](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/spark-courses.html)
- [Books](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/spark-books.html)

### 另外的书籍

#### Spark SQL

#### Spark Structured Streaming

#### Spark Streaming

### SPARK GRAPHX （已过期）

- Spark GraphX — Distributed Graph Computations
- Graph Algorithms

# 结语

对照整个目录与自己现在掌握的情况，决定先着重看下 Spark Core 模块。
