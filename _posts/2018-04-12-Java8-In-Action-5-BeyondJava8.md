---
layout: post
title: 《Java 8 in Action》学习日志（五）：超越Java 8
date: 2018-4-12 10:30:04
author: admin
comments: true
categories: [Java]
tags: [Java,Java8]
---


<!-- more -->

学习资料主要参考： 《Java 8 In Action》、《Java 8实战》，以及其源码：[Java8 In Action](https://github.com/java8/Java8InAction)

---
# 目录
{:.no_toc}

* 目录
{:toc}
---


# 函数式编程的思考

## 1. 实现和维护系统

实现和维护系统时，大多数程序员最关心是什么？

是代码遭遇一些无法预期的值就有可能发生崩溃，换句话说是我们无法预知的变量修改问题。

而这些问题都源于**共享的数据结构**被你所维护的代码中的多个方法读取和更新。

### 1）共享的可变数据

正是由于使用了可变的共享数据结构，我们很难追踪你程序的各个组成部分所发生的变化。

如果一个方法既不修改它内嵌类的状态，也不修改其他对象的状态，使用return返回所有的计算结果，那么我们称其为**纯粹的**或者**无副作用**的。

#### 哪些因素会造成副作用呢？
- 除了构造器内的初始化操作，对类中数据结构的任何修改，包括字段的赋值操作（一个典型的例子是setter方法）
- 抛出一个异常
- 进行输入/输出操作，比如向一个文件写数据

#### “无副作用”的好处

1. 如果构成系统的各个组件都能遵守这一原则，该系统就能在完全无锁的情况下，使用多核的并发机制，因为任何一个方法都不会对其他的方法造成干扰
2. 这还是一个让你了解你的程序中哪些部分是相互独立的非常棒的机会

### 2）函数式编程的基石:声明式编程 

一般通过编程实现一个系统，有两种思考方式：
1. 专注于如何实现
2. 更加关注要做什么

Stream API的思考方式正是后一种。它把实现的细节留给了函数库。
```java
//找出最大值
Optional<Transaction> mostExpensive = 
    transactions.stream() 
        .max(comparing(Transaction::getValue));
```
我们把这种思想称之为内部迭代。

采用这种“要做什么”风格的编程通常被称*为声明式编程*。

### 3）为什么要采用函数式编程

1. 声明式编程
2. 无副作用计算

## 2. 什么是函数式编程

简而言之：它是一种使用**函数**进行编程的方式

那什么又是“函数”呢？

在函数式编程的上下文中，一个“函数”对应于一个数学函数：它接受零个或多个参数，生成一个或多个结果，并且不会有任何副作用。你可以把它看成一个黑盒，它接收输入并产生一些输出。

### 1）函数式Java编程

被称为“函数式”的函数或方法应具备以下特点：
1. 都只能修改本地变量
2. 它引用的对象都应该是不可修改的对象
3. 函数或者方法不应该抛出任何异常（使用Optional<T>类型）

### 2）引用透明性

“没有可感知的副作用”（不改变对调用者可见的变量、不进行I/O、不抛出异常）的这些限制都隐含着**引用透明性**。

如果一个函数只要传递同样的参数值，总是返回同样的结果，那这个函数就是引用透明的。

### 3）面向对象的编程和函数式编程的对比

Java 8认为函数式编程其实只是面向对象的一个极端。

实际操作中，Java程序员经常混用这两种风格。

### 4）函数式编程实战

#### 需求：

给定一个列表List<value>，比如{1,4,9}，构造一个List<List<Integer>>，它的成员都是类表{1, 4, 9}的子集

#### 初步实现
递归实现
```java
static List<List<Integer>> subsets(List<Integer> list) { 
    if (list.isEmpty()) { 
        List<List<Integer>> ans = new ArrayList<>(); 
        ans.add(Collections.emptyList()); 
        return ans; 
    } 
    // 将list分为两部分，第一个元素与subList
    Integer first = list.get(0); 
    List<Integer> rest = list.subList(1, list.size()); 
    // 获取 subList的所有子集
    List<List<Integer>> subans = subsets(rest); 
    // 将第一个元素加入subList的所有子集中，生成新的list
    List<List<Integer>> subans2 = insertAll(first, subans); 
    // 合并两个list
    return concat(subans, subans2);
}
```
#### 如何定义insertAll方法

这个地方要小心，不能在产生subans2时，修改到subans

```java
static List<List<Integer>> insertAll(Integer first, 
                                    List<List<Integer>> lists) { 
    List<List<Integer>> result = new ArrayList<>(); 
    for (List<Integer> list : lists) { 
        // 创建一个新的list，而不是直接修改
        List<Integer> copyList = new ArrayList<>(); 
        copyList.add(first); 
        copyList.addAll(list); 
        result.add(copyList); 
    } 
    return result; 
}
```
#### 定义concat方法

简单的实现，但是希望你不要这样使用，因为它修改了参数
```java
static List<List<Integer>> concat(List<List<Integer>> a, 
                                List<List<Integer>> b) { 
    a.addAll(b); 
    return a; 
} 
```
纯粹的函数式
```java
static List<List<Integer>> concat(List<List<Integer>> a, 
                                List<List<Integer>> b) { 
    List<List<Integer>> r = new ArrayList<>(a); 
    r.addAll(b); 
    return r; 
}
```

## 3. 递归和迭代

来实现一个阶乘计算

### 1）迭代式

```java
static int factorialIterative(int n) {
    int r = 1;
    for (int i = 1; i <= n; i++) {
        r *= i;
    }
    return r;
}
```
### 2）递归式

```java
static long factorialRecursive(long n) {
    return n == 1 ? 1 : n * factorialRecursive(n-1);
}
```

### 3）基于 Stream

```java
static long factorialStreams(long n){
return LongStream.rangeClosed(1, n)
    .reduce(1, (long a, long b) -> a * b);
}
```
### 4）基于“尾-递”的阶乘

```java
static long factorialTailRecursive(long n) {
    return factorialHelper(1, n);
}

static long factorialHelper(long acc, long n) {
    return n == 1 ? acc : factorialHelper(acc * n, n-1);
}
```

第4个递归和第2个递归相比，采用了**尾-调优化(tail-call optimization)**。

尾-调优化的基本的思想是你可以编写阶乘的一个迭代定义,不过迭代调用发生在函数的最后(所以我们说调用发生在尾部)。它避免了过多的在不同的栈帧上保存每次递归计算的中间值，编译器能够自行决定复用某个栈帧进行计算，从一定程度上避免了StackOverflowError 异常。


# 未完待续。。。。。。