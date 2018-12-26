---
layout: post
title: Mastering Apache Spark Core（一）：转移Spark集群中数据块
date: 2018-12-25 15:46:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Mastering Apache Spark]
---

这一篇主要介绍了一些参与到读取block中数据的模块。

<!-- more -->

------

## 目录

{:.no_toc}

[TOC]

------

## ShuffleClient 

定义了拉取shuffle的blocks的合约：`fetchBlocks`，可以通过 `appId`  进行初始化。

```java
package org.apache.spark.network.shuffle;

abstract class ShuffleClient implements Closeable {
  // only required methods that have no implementation
  // the others follow
  abstract void fetchBlocks(
      String host,
      int port,
      String execId,
      String[] blockIds,
      BlockFetchingListener listener,
      TempFileManager tempFileManager);
}
```

这个抽象方法定义另一个抽象方法：fetchBlocks，也就是这个方法约定了如何异步从远程块管理器节点获取一系列块。这个方法只会在 ShuffleBlockFetcherIterator 被请求 [sendRequest](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/spark-ShuffleBlockFetcherIterator.html#sendRequest) 时专门使用。

它有两个实现类：BlockTransferService 和 ExternalShuffleClient。

#### 请求与 shuffle相关的 metrics

```java
public MetricSet shuffleMetrics() {
    // Return an empty MetricSet by default.
    return () -> Collections.emptyMap();
  }
```

shuffleMetrics默认返回一个空的MetricSet。

`shuffleMetrics` 只在当一个与 shuffle 有关的 metrics source 请求BlockManager时使用（仅当为非本地/集群模式创建Executor时）。

### BlockTransferService 

BlockTransferService在ShuffleClients的基础上，可以同步或异步地获取和上传数据块。

```scala
package org.apache.spark.network

abstract class BlockTransferService extends ShuffleClient {
  // only required methods that have no implementation the others follow
  def init(blockDataManager: BlockDataManager): Unit
  def close(): Unit
  def port: Int
  def hostName: String
    
  def fetchBlocks(
    host: String,
    port: Int,
    execId: String,
    blockIds: Array[String],
    listener: BlockFetchingListener,
    tempFileManager: TempFileManager): Unit
    
  def uploadBlock(
    hostname: String,
    port: Int,
    execId: String,
    blockId: BlockId,
    blockData: ManagedBuffer,
    level: StorageLevel,
    classTag: ClassTag[_]): Future[Unit]
}
```



## NettyBlockTransferService 

NettyBlockTransferService是一个BlockTransferService，它使用Netty上传或获取数据块。

为 drive 和 executor 创建SparkEnv时，会创建NettyBlockTransferService（以创建BlockManager）。

[![spark NettyBlockTransferService.png](/images/posts/spark-NettyBlockTransferService.png)](/images/posts/spark-NettyBlockTransferService.png)

### NettyBlockRpcServer

NettyBlockRpcServer是一个RpcHandler，它处理NettyBlockTransferService的消息。



## BlockFetchingListener

BlockFetchingListener是EventListeners的合约，希望收到有关onBlockFetchSuccess和onBlockFetchFailure的通知。

在以下情况下使用BlockFetchingListener：

- 请求ShuffleClient，BlockTransferService，NettyBlockTransferService和ExternalShuffleClient获取一系列块
- 请求BlockFetchStarter createAndStart
- RetryingBlockFetcher 和 OneForOneBlockFetcher被创建

```
package org.apache.spark.network.shuffle;

interface BlockFetchingListener extends EventListener {
  void onBlockFetchSuccess(String blockId, ManagedBuffer data);
  void onBlockFetchFailure(String blockId, Throwable exception);
}
```



## RetryingBlockFetcher

创建RetryingBlockFetcher并在以下时间立即启动：

- NettyBlockTransferService被请求fetchBlocks（默认当maxIORetries大于0时）
- 请求ExternalShuffleClient fetchBlocks（默认当maxIORetries大于0时）

当请求启动时以及稍后的initiateRetry时，RetryingBlockFetcher使用BlockFetchStarter来createAndStart。

### 创建RetryingBlockFetcher实例

RetryingBlockFetcher在创建时采用以下内容：

- TransportConf
- BlockFetchStarter
- Block IDs to fetch
- BlockFetchingListener

### BlockFetchStarter

```java
void createAndStart(String[] blockIds, BlockFetchingListener listener)
   throws IOException, InterruptedException;
```

在以下情况下使用createAndStart：

- 请求ExternalShuffleClient fetchBlocks（当maxIORetries为0时）
- 请求NettyBlockTransferService fetchBlocks（当maxIORetries为0时）
- RetryingBlockFetcher被请求fetchAllOutstanding 