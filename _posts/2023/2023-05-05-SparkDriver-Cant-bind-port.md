---
layout: post
title: sparkDriver 无法绑定本地端口
date: 2023-05-05 13:39:04
author: admin
comments: true
categories: [Spark]
tags: [Spark]
---


<!-- more -->

---

* 目录
{:toc}
---

## 背景

在连接公司VPN的情况下，发现本地无法运行Spark example的code，会抛出以下错误：

```text
Service 'sparkDriver' could not bind on a random free port. You may check whether configuring an appropriate binding address.
............
java.net.BindException: Cannot assign requested address: Service 'sparkDriver' failed after 16 retries (on a random free port)! Consider explicitly setting the appropriate binding address for the service 'sparkDriver' (for example spark.driver.bindAddress for SparkDriver) to the correct binding address.
```

而不连接VPN的情况下，程序却可以正常运行。

## 解决方案

1. add `export SPARK_LOCAL_HOSTNAME=127.0.0.1` into .zshrc
2. source .zshrc
3. restart IDE

## 分析过程

经过调查，是因为方法 core/src/main/scala/org/apache/spark/util/Utils.scala#localCanonicalHostName 的返回值在连接VPN的时候，返回是一个公司内部的主机名，而不连接VPN的时候，返回的是一个ip地址。

```scala
  /**
   * Get the local machine's FQDN.
   */
  def localCanonicalHostName(): String = {
    customHostname.getOrElse(localIpAddress.getCanonicalHostName)
  }
```

`customHostname` 是从环境变量 `SPARK_LOCAL_HOSTNAME` 获取的，它默认应该是空值。localCanonicalHostName 的结果不同其实是 localIpAddress.getCanonicalHostName 导致的。

```scala
private var customHostname: Option[String] = sys.env.get("SPARK_LOCAL_HOSTNAME")
```

但是看了看源代码，发现并没有什么方式可以保持 localIpAddress.getCanonicalHostName 的返回值不变。

于是想到可以 export 环境变量 `SPARK_LOCAL_HOSTNAME`，这样子保证 `customHostname` 非空，就不会使用 localIpAddress.getCanonicalHostName 的返回值作为 `localCanonicalHostName` 的结果。
