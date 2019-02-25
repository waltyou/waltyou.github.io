---
layout: post
title: 《HeadFirst设计模式》学习日志（三）:装饰者模式
date: 2019-02-25 09:53:00
author: admin
comments: true
categories: [Java]
tags: [Java,Design Patterns]

---



<!-- more -->

------

学习资料主要参考： 《Head First 设计模式》

------

## 目录
{:.no_toc}

* 目录
{:toc}

------

### 简述

装饰者模式“给喜欢用继承的人，一个全新的设计眼界”。一旦熟悉了装饰的技巧，就可以在不修改任何底层代码的情况下，给你的（或别人的）对象赋予新的职责。

简单流程：

1. 获取一个基本对象
2. 拿另外一个对象装饰它
3. 调用它的方法，并依赖委托获取装饰者信息

## 开发-关闭原则

类应该对扩展开放，对修改关闭。

这样的设计，既保证了类的弹性以面对改变，又能避免修改原有的健康代码。

但是我们没必要让每处地方都遵循这个原则，否则代码就会变得复杂且难以理解。

## 特点

- 装饰者和被装饰者具有相同的超类型
- 可以用一个或者多个装饰者来装饰同一个对象
- 在需要原始对象的场合，可以用装饰过的对象替代
- 装饰者可以在所委托被装饰者的行为之前/之后，加上自己的行为，以达到特定目的
- 对象可以在任何时候被装饰，所以可以在运行时动态地、不限量地使用各种装饰者来装饰对象

**装饰者模式：动态地将责任附加到对象上。若要拓展功能，装饰者提供了比继承更有弹性的替代方案。**

```java
public abstract Component{
    void methodA();
    void methodB();
}

public ConcreteComponent extends Component{
    void methodA();
    void methodB();
}

public Decorator extends Component{
    void methodA();
    void methodB();
}

public ConcreteDecoratorA extends Decorator{
    Component wrappedObj;
    void methodA();
    void methodB();
    void newBehavior();
}
```

## Java 中的装饰者：I/O

比如 BufferedInputStream 以及 Line NumberInputStream 都拓展自 FilterInputStream，而 FilterInputStream 是一个抽象的装饰类。

装饰 java.io 类：

[![](/images/posts/java-io-decorator.png)](/images/posts/java-io-decorator.png)

## 缺点

- 在代码中引入大量的小类
- 客户端若是依赖特定的类，但是却忽然导入装饰者，可能会引起错误
- 实例化组件过于复杂
