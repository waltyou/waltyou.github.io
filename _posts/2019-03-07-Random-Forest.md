---
layout: post
title: 机器学习入门(六) -- 随机森林
date: 2019-03-07 18:45:04
author: admin
comments: true
categories: [Machine Learning]
tags: [Machine Learning]
---




<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

## 定义

**随机森林**指的是利用多棵树对样本进行训练并预测的一种分类器，并且其输出的类别是由个别树输出的类别的众数而定。

## 背景知识

随机森林属于集成学习（Ensemble Learning）中的bagging算法。

### Bagging（套袋法）

bagging的算法过程如下：

- 从原始样本集中使用Bootstraping方法随机抽取n个训练样本，共进行k轮抽取，得到k个训练集。（k个训练集之间相互独立，元素可以有重复）
- 对于k个训练集，我们训练k个模型（这k个模型可以根据具体问题而定，比如决策树，knn等）
- 对于分类问题：由投票表决产生分类结果；对于回归问题：由k个模型预测结果的均值作为最后预测结果。（所有模型的重要性相同）

## 决策树

要说随机森林，必须先讲决策树，因为所谓“森林”就是很多的“树”组成的。

决策树是一种基本的分类器，一般是将特征分为两类。

构建好的决策树呈树形结构，可以认为是if-then规则的集合，主要优点是模型具有可读性，分类速度快。

## 构建随机森林

随机森林顾名思义，是用随机的方式建立一个森林，森林里面有很多的决策树组成，随机森林的每一棵决策树之间是没有关联的。在得到森林之后，当有一个新的输 入样本进入的时候，就让森林中的每一棵决策树分别进行一下判断，看看这个样本应该属于哪一类（对于分类算法），然后看看哪一类被选择最多，就预测这个样本 为那一类。

那随机森林具体如何构建呢？有两个方面：**数据的随机性选取**，以及**待选特征的随机选取**。

### 1. 数据的随机选取

首先，从原始的数据集中采取有放回的抽样，构造子数据集，子数据集的数据量是和原始数据集相同的。不同子数据集的元素可以重复，同一个子数据集中的元素也可以重复。

第二，利用子数据集来构建子决策树，将这个数据放到每个子决策树中，每个子决策树输出一个结果。

最后，如果有了新的数据需要通过随机森林得到分类结果，就可以通过对子决策树的判断结果的投票，得到随机森林的输出结果了。

### 2. 待选特征的随机选取

与数据集的随机选取类似，随机森林中的子树的每一个分裂过程并未用到所有的待选特征，而是从所有的待选特征中随机选取一定的特征，之后再在随机选取的特征中选取最优的特征。

这样能够使得随机森林中的决策树都能够彼此不同，提升系统的多样性，从而提升分类性能。

## 优缺点

### 优点

- 在数据集上表现良好
- 在当前的很多数据集上，相对其他算法有着很大的优势
- 它能够处理很高维度的数据，并且不用做特征选择
- 在训练完后，它能够给出哪些feature比较重要
- 在创建随机森林的时候，对generlization error使用的是无偏估计
- 训练速度快
- 在训练过程中，能够检测到feature间的互相影响
- 容易做成并行化方法
- 实现比较简单

### 缺点

- 当随机森林中的决策树个数很多时，训练时需要的空间和时间会较大
- 随机森林模型还有许多不好解释的地方，有点算个黑盒模型

## 例子

```python
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
import pandas as pd
import numpy as np

iris = load_iris()
df = pd.DataFrame(iris.data, columns=iris.feature_names)
df['is_train'] = np.random.uniform(0, 1, len(df)) <= .75
df['species'] = pd.Categorical.from_codes(iris.target, iris.target_names)
df.head()

train, test = df[df['is_train']==True], df[df['is_train']==False]

features = df.columns[:4]
clf = RandomForestClassifier(n_jobs=2)y, _ = pd.factorize(train['species'])
clf.fit(train[features], y)

preds = iris.target_names[clf.predict(test[features])]

pd.crosstab(test['species'], preds, rownames=['actual'], colnames=['preds'])
```



## 参考

1. https://baike.baidu.com/item/%E9%9A%8F%E6%9C%BA%E6%A3%AE%E6%9E%97
2. https://blog.csdn.net/qq547276542/article/details/78304454
3. https://segmentfault.com/a/1190000007463203