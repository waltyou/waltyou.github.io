---
layout: post
title: CNN （Convolution Neural Network）介绍
date: 2019-04-28 15:48:04
author: admin
comments: true
categories: [Machine Learning]
tags: [Machine Learning, Deep Learning]
---

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

CNN 的结构就巧妙地减少了需要训练的参数数量。

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

## 0. Overview

1. 输入层 input layer
2. 隐含层
   1. 卷积层 （convolutional layer）
   2. 池化层（pooling layer）
   3. Flatten 层
   4. 全连接层（fully-connected layer）
3. 输出层

输入输出层，没什么好说的。我们直接来看隐含层。

隐含层中的全连接层就相当于一个正常的全连接神经网络，不过它的输入是池化层的输出。而卷积层、池化层，这两层通常都是会重复出现的，如下图：

[![](/images/posts/CNN-Architecture.png)](/images/posts/CNN-Architecture.png) 

着重来看一下隐含层的前三部分。

## 1. 卷积层

卷积层的实现，其实就是实现了核心思想的前两条。

它主要是做一件事情：将输入通过多个卷积核（kernel）（或者叫做filter），生成 result maps。

比较形象的过程可以看下图：

[![](/images/posts/CNN-kernel-mv.gif)](/images/posts/CNN-kernel-mv.gif) 

最左边的就是一个输入，第二个的就是一个卷积核（其实就是一个matrix），最右边的就是卷积核的输出们。

卷积核每次只关心和它同样大小的输入，然后矩阵内积后，产生出一个输出。接着，移动卷积核观察的区域，依次产生所有输出。最后，我们就得到一个新的矩阵。

每个卷积核都会生成一个矩阵作为输出。一个卷积层可能有多个卷积核。

## 2. 池化层

池化层做的事情，对应了核心思想的第三条。

在尝试学习内核之前，它会将卷积图像中的像素区域合并在一起（缩小图像）。 

[![](/images/posts/CNN-poolfig.gif)](/images/posts/CNN-poolfig.gif) 


## 3. Flatten 层 

它做的事情比较简单，就是把一个矩阵拉平，变为一个高维向量，作为全连接层的输入。


# 参考

1. [Convolutional Neural Networks - Basics](https://mlnotebook.github.io/post/CNN1/)
2. [知乎：能否对卷积神经网络工作原理做一个直观的解释？](https://www.zhihu.com/question/39022858)


