---
layout: post
title: 机器学习入门(二) -- 什么是机器学习 
date: 2018-10-23 18:00:04
author: admin
comments: true
categories: [Machine Learning]
tags: [Machine Learning]
---

既然要学习”机器学习“，那先来看看它到底是什么。


<!-- more -->

---



* 目录
{:toc}
---

# 一、 多样的定义

## 1. Arthur Samuel

在不直接针对问题进行编程的情况下，赋予计算机学习能力的一个研究领域。

## 2. Tom Mitchell

对于某类任务T和性能度量P，如果计算机程序在T上以P衡量的性能随着经验E而自我完善，那么就称这个计算机程序从经验E学习。

## 3. 李宏毅

机器学习相当于找一个函数(looking for a Function)，它可以接收一个输入，然后给出唯一输出。


简单来讲，机器学习就是经过一定**数据**的**训练**，得到合适的**模型**，从而获取相应的处理事件的能力。

---

# 二、 机器学习的发展史

## 1. 五十年代到七十年代初

### “推理期”：赋予机器逻辑推理的能力。

A.Newell和H.Simon的“逻辑理论家”程序证明了《数学原理》第38条原理（1952年），此后证明了所有52条原理（1963年）。

1950年，图灵曾经于关于图灵测试的文章提到了机器学习的可能

五十年代中后期，基于神经网络的“连接主义”（connectionism）学习开始出现，代表性工作有F.Rosenblatt的感知机（Perceptron），B.Widrow的Adaline等。

六七十年代，基于逻辑表示的“符号主义”（symbolism）学习技术开始发展

## 2. 七十年代中期
      
### “知识期”：让机器拥有知识。

E.A.Feigenbaum等人认为，机器必须拥有知识才能拥有智能，并且他主持研制了世界上第一个专家系统DENDRAL（1965）
      
## 3. 八十年代

### “学习期”：让机器去学习

“从样例中学习”：”从训练样例中归纳出学习结果。

“符号主义学习”：从样例中学习的一大主流，代表包括决策树（decision tree）和基于逻辑的学习。

“基于神经网络的连接主义学习“：到九十年中期的从样例中学习的另一大主流，五十年代的连接主义拘泥于符号表示，1983年J.J.Hopfield利用神经网络求解“流动推销员问题”这个NP问题取得重大进展；
1986年，D.E.Rumelhat等人发明了著名的BP算法。此刻的连接更多的是“黑箱”操作，更容易操作。但“参数”影响太大！！

## 4. 九十年代中期

“统计学习”：statistical learning，九十年代中期出现，迅速占领主流舞台，代表技术是支持向量机（Support Vector Machine，SVM）以及更一般的“核方法”（kernel methods）。

这方面的研究早于六七十年代已经开始，但九十年代在文本分类应用中才得以显现；另一方面连接注意学习的局限性，大家才意识到统计的好处来。

## 5. 二十一世纪初

“深度学习”：五十年代的连接主义又卷土重来了！名为“深度学习”，狭义的角度就是“很多层”的神经网络。

在涉及语音，图像等复杂对象的应用中，深度学习取得了非常优越的性能。
以往的机器学习对使用者的要求比较高；深度学习涉及的模型复杂度高，只要下功夫“调参”，性能往往就很好。

深度学习缺乏严格的理论基础，但显著降低了机器学习使用者的门槛！其实从另一个角度来看是机器处理速度的大幅度提升……

---

# 三、 机器学习分类

## 1. 学习方式

### 监督式学习

[![](/images/posts/supervised-learning.jpg)](/images/posts/supervised-learning.jpg)

在监督式学习下，输入数据被称为“训练数据”，每组训练数据有一个明确的标识或结果，如对防垃圾邮件系统中“垃圾邮件”“非垃圾邮件”，对手写数字识别中的“1“，”2“，”3“，”4“等。

在建立预测模型的时候，监督式学习建立一个学习过程，将预测结果与“训练数据”的实际结果进行比较，不断的调整预测模型，直到模型的预测结果达到一个预期的准确率。

监督式学习的常见应用场景如分类问题和回归问题。

常见算法有逻辑回归（Logistic Regression）和反向传递神经网络（Back Propagation Neural Network）。

### 非监督式学习

[![](/images/posts/unsupervised-learning.jpg)](/images/posts/unsupervised-learning.jpg)

在非监督式学习中，数据并不被特别标识，学习模型是为了推断出数据的一些内在结构。

常见的应用场景包括关联规则的学习以及聚类等。

常见算法包括Apriori算法以及k-Means算法。

### 半监督式学习

[![](/images/posts/semi-supervised-learning.jpg)](/images/posts/semi-supervised-learning.jpg)

在此学习方式下，输入数据部分被标识，部分没有被标识，这种学习模型可以用来进行预测，但是模型首先需要学习数据的内在结构以便合理的组织数据来进行预测。
应用场景包括分类和回归，算法包括一些对常用监督式学习算法的延伸，这些算法首先试图对未标识数据进行建模，在此基础上再对标识的数据进行预测。

如图论推理算法（Graph Inference）或者拉普拉斯支持向量机（Laplacian SVM.）等。

### 强化学习

[![](/images/posts/reinforcement-learning.jpg)](/images/posts/reinforcement-learning.jpg)

在这种学习模式下，输入数据作为对模型的反馈，不像监督模型那样，输入数据仅仅是作为一个检查模型对错的方式，
在强化学习下，输入数据直接反馈到模型，模型必须对此立刻作出调整。

常见的应用场景包括动态系统以及机器人控制等。

常见算法包括Q-Learning以及时间差学习（Temporal difference learning）

## 2. 算法类似性

### 回归算法

[![](/images/posts/regression.jpg)](/images/posts/regression.jpg)

回归算法是试图采用对误差的衡量来探索变量之间的关系的一类算法。
回归算法是统计机器学习的利器。
在机器学习领域，人们说起回归，有时候是指一类问题，有时候是指一类算法，这一点常常会使初学者有所困惑。

常见的回归算法包括：最小二乘法（Ordinary Least Square），逻辑回归（Logistic Regression），逐步式回归（Stepwise Regression），
多元自适应回归样条（Multivariate Adaptive Regression Splines）以及本地散点平滑估计（Locally Estimated Scatterplot Smoothing）

线性回归详细介绍看[这里](../Linear-Regression/).

逻辑回归详细介绍看[这里](../Logistic-Regression/).

### 基于实例的算法

[![](/images/posts/knn.jpg)](/images/posts/knn.jpg)

基于实例的算法常常用来对决策问题建立模型，这样的模型常常先选取一批样本数据，然后根据某些近似性把新数据与样本数据进行比较。
通过这种方式来寻找最佳的匹配。因此，基于实例的算法常常也被称为“赢家通吃”学习或者“基于记忆的学习”。

常见的算法包括 k-Nearest Neighbor(KNN), 学习矢量量化（Learning Vector Quantization， LVQ），
以及自组织映射算法（Self-Organizing Map ， SOM）

knn 详细介绍看[这里](../KNN/).

### 正则化方法

[![](/images/posts/Ridge-Regression.jpg)](/images/posts/Ridge-Regression.jpg)

正则化方法是其他算法（通常是回归算法）的延伸，根据算法的复杂度对算法进行调整。
正则化方法通常对简单模型予以奖励而对复杂算法予以惩罚。

常见的算法包括：Ridge Regression， Least Absolute Shrinkage and Selection Operator（LASSO），以及弹性网络（Elastic Net）。

### 决策树学习

[![](/images/posts/decision-tree.jpg)](/images/posts/decision-tree.jpg)

决策树算法根据数据的属性采用树状结构建立决策模型， 决策树模型常常用来解决分类和回归问题。

常见的算法包括：分类及回归树（Classification And Regression Tree， CART）， ID3 (Iterative Dichotomiser 3)， C4.5， 
Chi-squared Automatic Interaction Detection(CHAID), Decision Stump, 随机森林（Random Forest）， 
多元自适应回归样条（MARS）以及梯度推进机（Gradient Boosting Machine， GBM）。

决策树详细介绍看[这里](../Decision-Tree/).

随机森林详细介绍看[这里](../Random-Forest/).

### 贝叶斯方法

[![](/images/posts/BBN.jpg)](/images/posts/BBN.jpg)

贝叶斯方法算法是基于贝叶斯定理的一类算法，主要用来解决分类和回归问题。

常见算法包括：朴素贝叶斯算法，平均单依赖估计（Averaged One-Dependence Estimators， AODE），以及Bayesian Belief Network（BBN）。


### 基于核的算法

[![](/images/posts/svm.jpg)](/images/posts/svm.jpg)

基于核的算法中最著名的莫过于支持向量机（SVM）了。 
基于核的算法把输入数据映射到一个高阶的向量空间， 在这些高阶向量空间里， 有些分类或者回归问题能够更容易的解决。 

常见的基于核的算法包括：支持向量机（Support Vector Machine， SVM）， 径向基函数（Radial Basis Function ，RBF)， 
以及线性判别分析（Linear Discriminate Analysis ，LDA)等。

SVM 详细介绍看[这里](../SVM/).

### 聚类算法

[![](/images/posts/k-means.jpg)](/images/posts/k-means.jpg)

聚类，就像回归一样，有时候人们描述的是一类问题，有时候描述的是一类算法。
聚类算法通常按照中心点或者分层的方式对输入数据进行归并。
所以的聚类算法都试图找到数据的内在结构，以便按照最大的共同点将数据进行归类。

常见的聚类算法包括 k-Means算法以及期望最大化算法（Expectation Maximization， EM）。

k-Means 详细介绍看[这里](../K-Means/).

### 关联规则学习

[![](/images/posts/Association-rule-learning.jpg)](/images/posts/Association-rule-learning.jpg)

关联规则学习通过寻找最能够解释数据变量之间关系的规则，来找出大量多元数据集中有用的关联规则。

常见算法包括 Apriori算法和Eclat算法等。

### 人工神经网络

[![](/images/posts/neural-network.jpg)](/images/posts/neural-network.jpg)

人工神经网络算法模拟生物神经网络，是一类模式匹配算法。
通常用于解决分类和回归问题。
人工神经网络是机器学习的一个庞大的分支，有几百种不同的算法。（其中深度学习就是其中的一类算法，我们会单独讨论）。

重要的人工神经网络算法包括：感知器神经网络（Perceptron Neural Network）, 反向传递（Back Propagation）， 
Hopfield网络，自组织映射（Self-Organizing Map, SOM）, 学习矢量量化（Learning Vector Quantization， LVQ）。

神经网络详细介绍看[这里](../Neural-Network/).

### 深度学习

[![](/images/posts/deep-learning.jpg)](/images/posts/deep-learning.jpg)

深度学习算法是对人工神经网络的发展。
在计算能力变得日益廉价的今天，深度学习试图建立大得多也复杂得多的神经网络。
很多深度学习的算法是半监督式学习算法，用来处理存在少量未标识数据的大数据集。

常见的深度学习算法包括：受限波尔兹曼机（Restricted Boltzmann Machine， RBN）， Deep Belief Networks（DBN），
卷积网络（Convolutional Network）, 堆栈式自动编码器（Stacked Auto-encoders）。

### 降低维度算法

[![](/images/posts/PCA.jpg)](/images/posts/PCA.jpg)

像聚类算法一样，降低维度算法试图分析数据的内在结构，不过降低维度算法是以非监督学习的方式试图利用较少的信息来归纳或者解释数据。

这类算法可以用于高维数据的可视化或者用来简化数据以便监督式学习使用。

常见的算法包括：主成份分析（Principle Component Analysis， PCA），偏最小二乘回归（Partial Least Square Regression，PLS）， 
Sammon映射，多维尺度（Multi-Dimensional Scaling, MDS）,  投影追踪（Projection Pursuit）等。

### 集成算法

[![](/images/posts/ensemble-learning.png)](/images/posts/ensemble-learning.png)

集成算法用一些相对较弱的学习模型独立地就同样的样本进行训练，然后把结果整合起来进行整体预测。

集成算法的主要难点在于究竟集成哪些独立的较弱的学习模型以及如何把学习结果整合起来。
这是一类非常强大的算法，同时也非常流行。

常见的算法包括：Boosting， Bootstrapped Aggregation（Bagging）， AdaBoost，堆叠泛化（Stacked Generalization， Blending），
梯度推进机（Gradient Boosting Machine, GBM），随机森林（Random Forest）。

---

# 四、 总结

这篇文章简单介绍了机器学习的基本定义、发展过程以及常用算法分类。

这些算法大部分在 SKlearn 中都有实现，可以参考下图。

[![](/images/posts/machine_learning_map.png)](/images/posts/machine_learning_map.png)


---

# 五、参考

1. [Andrew Ng Coursera 课程](https://www.coursera.org/learn/machine-learning)
2. [机器学习发展历程](https://blog.csdn.net/zmdsjtu/article/details/52690839)
3. [常见机器学习分类](https://blog.csdn.net/sinat_31337047/article/details/78247609)
