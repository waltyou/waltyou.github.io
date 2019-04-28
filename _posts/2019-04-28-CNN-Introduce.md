---
layout: post
title: CNN （Convolution Neural Network）介绍
date: 2019-04-28 15:48:04
author: admin
comments: true
categories: [Machine Learning]
tags: [Machine Learning, Deep Learning]
---

神经网络

在神经网络

CNN，卷积神经网络，是属于深度学习范畴的一个算法框架，它在图片处理方面很有建树，来了解一下。

<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 前言

之前自己有一篇关于神经网络的介绍，里面只是介绍了最基本的神经网络，它是全连接的。

全连接的神经网络，每一层的每个神经元都接收了**所有**前一层的输入。这就带来了一个问题，就是我们需要训练好多好多的参数，那么势必要减慢我们的训练速度。在机器学习中，训练速度某种程度上也影响了我们的训练精度。因为如果训练速度很快，我们就可以进行更多的调参，更容易找到最优解。

CNN 的结构就巧妙地减少神经元数量。

# 简述

CNN，全称 convolution neural network，卷积神经网络。

使用范围：

- 图片或者视频中的物体识别（Object recognition）
- 自然语言处理
- 玩游戏，最著名的就是围棋的 阿尔法go
- 医疗创新，从药物发现到疾病预测

# 核心思想

为了更加形象地讲述，我们假设现在对一张鸟类图片进行 object detection。

从常识角度来讲，图片其实具有三个特点：

1. 识别出物体的某个特征时，我们并不需要整张图片，只需要一个区域即可。比如我们要找鸟嘴的这个特征，其实只需要图片中某一块区域就可以。
2. 不同的图片中，特征的位置可能不一样，但是特征的匹配是差不多的。还以鸟嘴为例子，可能一张图片的鸟嘴在左上角，另外一张的在右下角。这说明某些神经元是可以通用的。
3. 图片进行抽样后，并不影响整张图片意思的表达。比如我们去掉图片的偶数行、奇数列，虽然整张图片的大小缩小了一半，但是我们仍然可以看出图片里面有什么。

# 结构

1. 输入层 input layer
2. 隐含层
   1. 卷积层 （convolutional layer）
   2. 池化层（pooling layer）
   3. 全连接层（fully-connected layer）
3. 输出层

卷积过程：

[![](/images/posts/CNN-kernel-mv.gif)](/images/posts/CNN-kernel-mv.gif) 

# 参考

1. [Convolutional Neural Networks - Basics](https://mlnotebook.github.io/post/CNN1/)



## 未完待续。。。
