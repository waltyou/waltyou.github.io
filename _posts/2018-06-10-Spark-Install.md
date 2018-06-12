---
layout: post
title: Spark 安装与环境配置
date: 2018-06-10 17:39:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark]
---

先在ubuntu下安装一下Spark。

<!-- more -->
---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 安装

## 1. 下载压缩包

第一步是选择适当版本的压缩包，可以到[官网](http://spark.apache.org/downloads.html)进行下载。

截至到2018年2月，Spark最新版本是2.3.0版本。

然后下载压缩包到自己的目录下就好。

## 2. 配置环境

下载压缩包的时候，来配置一下环境。

先看看[官方文档](http://spark.apache.org/docs/2.3.0/)，里面有句关于环境依赖的说明:

> Spark runs on Java 8+, Python 2.7+/3.4+ and R 3.1+. For the Scala API, Spark 2.3.0 uses Scala 2.11. You will need to use a compatible Scala version (2.11.x).
>
> Note that support for Java 7, Python 2.6 and old Hadoop versions before 2.6.5 were removed as of Spark 2.2.0. Support for Scala 2.10 was removed as of 2.3.0.

也就是说，Spark 2.3.0 的其他语言版本要求是： 2.11.x的Scala，Java 8+，Python 2.7+或者3.1+， R 要3.1+。

根据自己常用开发语言安装即可。

参考如下：
1. [使用IDEA安装Scala](https://docs.scala-lang.org/getting-started-intellij-track/getting-started-with-scala-in-intellij.html)
2. [使用sbt在命令行安装](https://docs.scala-lang.org/getting-started-sbt-track/getting-started-with-scala-and-sbt-on-the-command-line.html)

## 2. 解压

```shell
tar xzf spark-2.3.0-bin-hadoop2.7.tgz
```

## 3. 设置环境变量

Spark需要两个环境变量： JAVA_HOME 和 SPARK_HOME

在ubuntu下，直接编辑用户目录下的 .bashrc即可：
```
export JAVA_HOME=<path-to-the-root-of-your-Java-installation> (eg: /usr/lib/jvm/java-7-oracle/)
export SPARK_HOME=<path-to-the-root-of-your-spark-installation> (eg: /path/spark/in/spark-1.6.1-bin-hadoop2.6/)
```

## 4. 检验是否成功

进入到解压目录下，执行
```
./bin/spark-shell
```
即可进入spark命令行界面。

然后也可以打开 _http://localhost:4040_ ，通过web UI查看任务情况。


---

# 参考链接
1. [Apache Spark Installation On Ubuntu](https://data-flair.training/blogs/apache-spark-installation-on-ubuntu/)
