---
layout: post
title: 在 IntelliJ IDEA 中运行、调试 PySpark Source Code
date: 2022-09-20 13:39:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, PySpark]
---

之前写过一篇文章来介绍如何构建 Spark 源码本地的 Debug 环境 （详情看[这里](https://waltyou.github.io/Spark-Source-Code-Build-And-Run-In-Idea-Intellij/)），但是对于 PySpark 的调试环境略有不同，再来一篇文章介绍一下。

<!-- more -->
---


* 目录
{:toc}
---

## 提交 Python SDK

下载源代码，用 IDEA 打开，首先添加 Python SDK:


![picture 1](../images/80eaa5525aee1eb0ed7af856ac13b43c95dbd3282b63e3c119e5a3d6aa4765a5.png)  


![picture 2](../images/6985394a3c9401d15189b686d9f9af6441fd25126c77b1b06d373a661207bd99.png)  


```shell
source venv/bin/activate
pip install -r dev/requirements.txt

./build/mvn -DskipTests clean package -Phive
cd python; python setup.py sdist
```

## Run PySpark

Run `pi.py`

![picture 3](../images/33649a50141e35be1aeae812ea778e241c78a24442cba9c316da4a2482c57743.png)  

![picture 4](../images/c5e2afde7eb98d5cf337742551486b4f14e2feb35482eb662025093ae5eae842.png)  

## Debug Python thread in PySpark

很简单，和debug 其他程序一样，只要设置 BreakPoints，然后点击debug 按钮就可debug：

![picture 5](../images/cbe155d757ce2628729981c276a5df2074aad98697fddec3efc85c95c6b7913a.png)  

## Debug Java thread in PySpark

> 在此之前，可以先去了解一下 pyspark基础的架构，比如[这一篇](https://www.mobvista.com/cn/blog/2019-12-27-2/)。

这一部分复杂一些，需要手动去设置一些属性，具体如下图所示：
![picture 7](../images/75d4bd14e95a352ff402821cba818168916871dc6d20f506eb6a5c0084255a91.png)  

- Main Class: org.apache.spark.deploy.SparkSubmit
- Program Arguments: --master local examples/src/main/python/pi.py
- Environment variables: PYSPARK_DRIVER_PYTHON=/Users/yonyou/git/spark/venv/bin/python;PYSPARK_PYTHON=/Users/yonyou/git/spark/venv/bin/python

最重要的是我们需要设置两个环境变量 `PYSPARK_DRIVER_PYTHON` 和 `PYSPARK_PYTHON` 告诉 pyspark 应该用哪个 python。因为我用 IDEA 默认创建了一个 Python venv, 而且在这个 venv 里 install 了所有 pyspark 的依赖，所以我用这个。当然如果你默认的python环境有所有pyspark的依赖，也可以不设置这两个环境变量。

最终效果：
![picture 8](../images/404ac6e0c9004f836ce57a73c2f94ad66be48e74298c683270a9c68db87a0f11.png)  
