---
layout: post
title: 机器学习入门(三) -- 线性回归
date: 2018-10-23 18:00:04
author: admin
comments: true
categories: [Machine Learning]
tags: [Machine Learning]
---

线性回归是机器学习中最基本的一个算法。但它“麻雀虽小，五脏俱全”，仔细理解后，将有助于我们更加了解机器学习。

<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 什么是线性回归

线性回归包括**一元**线性回归和**多元**线性回归。

一元的是只有一个x和一个y。 也就是以前数学里常见的： 
```
y = wx + b;
```

w 和 b 是参数。


多元的是指有多个x和一个y。
```
y = a1 * x^1 + a2 * x^2 + a3 * x^3 + ... + b
```

我们这次只看一元线性回归。

--- 

# 预测房价

## 提前准备

房价的高低，可变因素很多。但简单起见，我们知道房子越大，总价应该是越高的。所以这次我们只关系一个变量：房屋大小。

比如数据可以是这个样子的(我自己随便造的)：

| area | prices（w）|
| --- | --- |
| 100 | 100 |
| 200 | 130 |
| 300 | 170 |

如何在一个二维坐标系上表示这些数据呢？ 自变量定为 area，也就是 x 坐标，因变量定位 prices， 也就是 y 左边。
 
来把这些点画到坐标系上看一看。

[![](/images/posts/linear-regression-data1.jpg)](/images/posts/linear-regression-data1.jpg)


对于一元线性回归来讲，我们希望找到一个 w 和 b， 可以满足输入任意的 x ，我们都能得到一个 y， 这个 y 就是我们对于结果的预测。

在坐标系上表示，就是来找一条直线，这条直线能以最小的误差（Loss）来拟合数据。如下：

[![](/images/posts/linear-regression-data2.jpg)](/images/posts/linear-regression-data2.jpg)

## 怎么找最小误差

未完待续。。。。。

---

# 参考

1. [机器学习-线性回归预测房价模型demo](https://www.jianshu.com/p/928b95645757)
2. [Python 机器学习：线性回归](https://baijiahao.baidu.com/s?id=1602127602901158968&wfr=spider&for=pc)
