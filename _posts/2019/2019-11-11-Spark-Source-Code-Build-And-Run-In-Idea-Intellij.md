---
layout: post
title: Spark 源码 Build 及在 IntelliJ IDEA 中运行、调试Source Code
date: 2019-11-11 13:39:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark]
---

为了方便的阅读、理解 Spark 源码，debug 是个好方式。来介绍一下Spark 在 IntelliJ IDEA 中 Debug 环境构建。

<!-- more -->
---


* 目录
{:toc}
---

## 源码获取

这个可以从 [github](https://github.com/apache/spark) 上找到。下载自己感兴趣的版本即可。

```shell
git clone https://github.com/apache/spark.git
```

## 构建项目

以下步骤主要参考[官方文档](https://spark.apache.org/developer-tools.html)。打开页面，搜索 “IntelliJ IDEA”关键词。

1. 下载 IntelliJ IDEA， *Preferences > Plugins*，搜索 [Scala Plugin](https://www.jetbrains.com/help/idea/discover-intellij-idea-for-scala.html) 并安装.
2. *File -> Import Project*, 到达代码位置，并选择 “Maven Project”。[![](/images/posts/spark-debug-in-idea-1.png)](/images/posts/spark-debug-in-idea-1.png)
3. 在 Import 过程中，选中 “Import Maven projects automatically”，其他选项不变。[![](/images/posts/spark-debug-in-idea-2.png)](/images/posts/spark-debug-in-idea-2.png)
4. 接下来要参考另外一个官方文档：[Building Spark](https://spark.apache.org/docs/latest/building-spark.html#apache-maven) 。
5. 在项目根目录打开终端
6. Run `export MAVEN_OPTS="-Xmx2g -XX:ReservedCodeCacheSize=512m"`
7. Run `./build/mvn -DskipTests clean package` 

第7步会花费一些时间。等到第七步完成，整个项目就build 成功了。



## 运行 example

举个例子，比如运行 SparkPi。只需修改 Run Configuration 两处地方就好。
[![](/images/posts/spark-debug-in-idea-3.png)](/images/posts/spark-debug-in-idea-3.png)

接下来就可以debug 运行了。
















